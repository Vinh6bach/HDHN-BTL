
# LED Driver (GPIO 60 - Memory Mapping)

Trình điều khiển LED (GPIO 60) sử dụng kỹ thuật ánh xạ bộ nhớ vật lý `ioremap` trên AM335x, kèm theo một ứng dụng User Space để kiểm tra chớp tắt LED.

---

### 1. Tệp cấu hình: `Config.in`
> **Mô tả chức năng:** Khai báo gói phần mềm `my_gpio_driver` vào menu cấu hình của Buildroot, cho phép người dùng tick chọn để biên dịch Driver này cùng với hệ điều hành.

```kconfig
config BR2_PACKAGE_MY_GPIO_DRIVER
    bool "my_gpio_driver"
    depends on BR2_LINUX_KERNEL
    help
      GPIO driver + user app for LED control.

```

---

### 2. Tệp cấu hình: `my_gpio_driver.mk`

> **Mô tả chức năng:** Kịch bản Buildroot (Makefile) thực hiện biên dịch chéo cho cả Kernel Module (Driver) và ứng dụng C (User App). Sau đó nó sẽ copy file module `.ko` vào `/lib/modules/` và ném file thực thi `my_gpio_app` vào thư mục `/usr/bin/` của mạch nhúng.

```make
MY_GPIO_DRIVER_VERSION = 1.0
MY_GPIO_DRIVER_SITE = $(TOPDIR)/package/my_gpio_driver/src
MY_GPIO_DRIVER_SITE_METHOD = local

define MY_GPIO_DRIVER_BUILD_CMDS
	$(MAKE) $(LINUX_MAKE_FLAGS) -C $(@D) \
		LINUX_DIR=$(LINUX_DIR) \
		PWD=$(@D) \
		CC="$(TARGET_CC)" \
		all
endef

define MY_GPIO_DRIVER_INSTALL_TARGET_CMDS
	$(INSTALL) -D -m 0755 $(@D)/my_gpio_app $(TARGET_DIR)/usr/bin/my_gpio_app
	mkdir -p $(TARGET_DIR)/lib/modules/$(LINUX_VERSION)
	cp $(@D)/my_gpio_driver.ko $(TARGET_DIR)/lib/modules/$(LINUX_VERSION)/
endef

$(eval $(kernel-module))
$(eval $(generic-package))

```

---

### 3. Mã nguồn Driver: `src/my_gpio_driver.c`

> **Mô tả chức năng tổng quan:** Driver này thao tác trực tiếp với thanh ghi phần cứng. Nó sử dụng lệnh `ioremap` để ánh xạ địa chỉ vật lý của bộ điều khiển GPIO1 (`0x4804C000`) vào không gian nhớ ảo của Kernel. Các hàm read/write sẽ can thiệp trực tiếp vào thanh ghi DATAIN, SETDATAOUT, CLEARDATAOUT để điều khiển GPIO 60 (Tương ứng bit 28).

```c
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/io.h>
#include <linux/device.h>
#include <linux/uaccess.h>

#define DEVICE_NAME "my_gpio_dev"
#define CLASS_NAME  "my_gpio_class"

// Khai báo địa chỉ Base và các thanh ghi Offset của GPIO1 trên AM335x
#define GPIO1_BASE         0x4804C000
#define GPIO1_SIZE         0x2000
#define GPIO_OE            0x134
#define GPIO_SETDATAOUT    0x194
#define GPIO_CLEARDATAOUT  0x190
#define GPIO_DATAIN        0x138
#define GPIO_60_BIT        (1 << 28) // GPIO 60 là chân số 28 thuộc Port GPIO1

static int majorNumber;
static struct class* gpioClass  = NULL;
static struct device* gpioDevice = NULL;
static void __iomem *gpio1_base_addr; // Con trỏ lưu địa chỉ ảo sau khi ánh xạ

static int dev_open(struct inode *inodep, struct file *filep) {
    return 0;
}

/* * KHỐI 1: HÀM ĐỌC TRẠNG THÁI (READ)
 * Trích xuất giá trị từ thanh ghi DATAIN.
 * Sử dụng phép toán AND bit (reg_val & GPIO_60_BIT) để kiểm tra xem 
 * chân GPIO 60 đang ở mức CAO (1) hay THẤP (0), sau đó gửi về User App.
 */
static ssize_t dev_read(struct file *filep, char *buffer, size_t len, loff_t *offset) {
    uint32_t reg_val;
    char state;
    if (*offset > 0) return 0;
    
    reg_val = ioread32(gpio1_base_addr + GPIO_DATAIN);
    state = (reg_val & GPIO_60_BIT) ? '1' : '0';
    
    if (copy_to_user(buffer, &state, 1) != 0) return -EFAULT;
    *offset = 1;
    return 1;
}

/* * KHỐI 2: HÀM GHI TRẠNG THÁI (WRITE)
 * Khi User App ghi '1', Driver sẽ ghi một bit 1 vào thanh ghi SETDATAOUT để bật LED.
 * Khi User App ghi '0', Driver sẽ ghi một bit 1 vào thanh ghi CLEARDATAOUT để tắt LED.
 */
static ssize_t dev_write(struct file *filep, const char *buffer, size_t len, loff_t *offset) {
    char k_buf;
    if (copy_from_user(&k_buf, buffer, 1) != 0) return -EFAULT;
    
    if (k_buf == '1') {
        iowrite32(GPIO_60_BIT, gpio1_base_addr + GPIO_SETDATAOUT);
    } else if (k_buf == '0') {
        iowrite32(GPIO_60_BIT, gpio1_base_addr + GPIO_CLEARDATAOUT);
    }
    return len;
}

static struct file_operations fops = {
    .open = dev_open,
    .read = dev_read,
    .write = dev_write,
};

/* * KHỐI 3: HÀM KHỞI TẠO (INIT)
 * Đăng ký Device Node /dev/my_gpio_dev.
 * Chạy ioremap để lấy địa chỉ ảo của vùng nhớ phần cứng.
 * Cấu hình chân GPIO 60 thành OUTPUT bằng cách xóa (clear) bit 28 trong thanh ghi OE (Output Enable).
 */
static int __init gpio_driver_init(void) {
    majorNumber = register_chrdev(0, DEVICE_NAME, &fops);
    gpioClass = class_create(THIS_MODULE, CLASS_NAME);
    gpioDevice = device_create(gpioClass, NULL, MKDEV(majorNumber, 0), NULL, DEVICE_NAME);
    
    // Ánh xạ bộ nhớ vật lý sang bộ nhớ ảo
    gpio1_base_addr = ioremap(GPIO1_BASE, GPIO1_SIZE);
    
    // Cấu hình chân thành chế độ Output
    uint32_t oe_val = ioread32(gpio1_base_addr + GPIO_OE);
    oe_val &= ~GPIO_60_BIT;
    iowrite32(oe_val, gpio1_base_addr + GPIO_OE);
    
    printk(KERN_INFO "GPIO Driver: Loaded. GPIO 60 ready.\n");
    return 0;
}

/* * KHỐI 4: HÀM HỦY THIẾT BỊ (EXIT)
 * Hủy ánh xạ bộ nhớ (iounmap) và xóa các file thiết bị.
 */
static void __exit gpio_driver_exit(void) {
    iounmap(gpio1_base_addr);
    device_destroy(gpioClass, MKDEV(majorNumber, 0));
    class_destroy(gpioClass);
    unregister_chrdev(majorNumber, DEVICE_NAME);
}

module_init(gpio_driver_init);
module_exit(gpio_driver_exit);
MODULE_LICENSE("GPL");

```

---

### 4. Ứng dụng User App phụ trợ: `src/my_gpio_app.c`

> **Mô tả chức năng:** Một chương trình C nhỏ chạy trên User Space. Nó mở file `/dev/my_gpio_dev` và liên tục ghi ký tự '1' rồi '0' xuống để làm LED chớp tắt liên tục. Người dùng có thể truyền tham số tần số nháy qua dòng lệnh (ví dụ: `my_gpio_app 100`).

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdlib.h>

int main(int argc, char *argv[]) {
    int fd = open("/dev/my_gpio_dev", O_RDWR);
    if (fd < 0) return 1;
    
    int freq = (argc > 1) ? atoi(argv[1]) : 500;
    printf("Blinking GPIO 60 moi %d ms...\n", freq);
    
    while(1) {
        write(fd, "1", 1);
        usleep(freq * 1000);
        write(fd, "0", 1);
        usleep(freq * 1000);
    }
    
    close(fd);
    return 0;
}

```

---

### 5. Tệp: `src/Makefile`

> **Mô tả chức năng:** Kbuild Makefile đặc biệt, chỉ đạo Buildroot biên dịch đồng thời cả Kernel Module (`my_gpio_driver.ko`) thông qua $(MAKE) của Kernel và chương trình User Space (`my_gpio_app`) thông qua trình biên dịch C $(CC) thông thường.

```make
obj-m += my_gpio_driver.o

all:
	$(MAKE) -C $(LINUX_DIR) M=$(PWD) modules
	$(CC) my_gpio_app.c -o my_gpio_app

clean:
	$(MAKE) -C $(LINUX_DIR) M=$(PWD) clean
	rm -f my_gpio_app

```

```



```
