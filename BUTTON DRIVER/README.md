

# Button Driver (GPIO 48 - P9_15)

Trình điều khiển nút nhấn đơn (Single Button) sử dụng cơ chế ngắt (Interrupt) trên bo mạch BeagleBone Black.

---

### 1. Tệp cấu hình: `Config.in`
> **Mô tả chức năng:** File này dùng để tạo một menu tùy chọn trong giao diện `make menuconfig` của Buildroot. Nó giúp hệ thống nhận diện driver này phụ thuộc vào Linux Kernel và cho phép người dùng tick chọn để biên dịch.

```kconfig
config BR2_PACKAGE_MY_BUTTON
    bool "my_button"
    depends on BR2_LINUX_KERNEL
    help
      Kernel module for single Button using Interrupt on P9_15 (GPIO 48).

```

---

### 2. Tệp cấu hình: `my_button.mk`

> **Mô tả chức năng:** Đây là file Makefile dành riêng cho Buildroot. Nó chỉ định thư mục chứa mã nguồn (`src`), định nghĩa các lệnh để biên dịch module bằng công cụ của Kernel (`MY_BUTTON_BUILD_CMDS`) và lệnh copy file `.ko` đã biên dịch xong vào thư mục gốc của mạch nhúng (`MY_BUTTON_INSTALL_TARGET_CMDS`).

```make
MY_BUTTON_VERSION = 1.0
MY_BUTTON_SITE = $(TOPDIR)/package/my_button/src
MY_BUTTON_SITE_METHOD = local

define MY_BUTTON_BUILD_CMDS
	$(MAKE) -C $(LINUX_DIR) $(LINUX_MAKE_FLAGS) M=$(@D) modules
endef

define MY_BUTTON_INSTALL_TARGET_CMDS
	$(INSTALL) -D -m 0644 $(@D)/my_button.ko $(TARGET_DIR)/lib/modules/my_button.ko
endef

$(eval $(kernel-module))
$(eval $(generic-package))

```

---

### 3. Mã nguồn Driver: `src/my_button.c`

> **Mô tả chức năng tổng quan:** Trình điều khiển thiết bị dạng Character (Character Device Driver) kết hợp với GPIO Interrupt. Mã nguồn này đăng ký một thiết bị `/dev/my_button_dev`, lắng nghe ngắt từ chân GPIO 48 và gửi trạng thái lên ứng dụng User Space một cách an toàn (không tốn CPU chờ đợi nhờ cơ chế Wait Queue).

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/uaccess.h>
#include <linux/gpio.h>
#include <linux/interrupt.h>
#include <linux/device.h>
#include <linux/jiffies.h>
#include <linux/wait.h>

#define DEVICE_NAME "my_button_dev"
#define CLASS_NAME  "my_button_class"
#define BUTTON_GPIO 48 

// Các biến toàn cục quản lý Device Node và trạng thái
static int majorNumber;
static struct class* buttonClass = NULL;
static struct device* buttonDevice = NULL;
static int button_irq;
static char button_state = '0';
static bool data_ready = false;
static DECLARE_WAIT_QUEUE_HEAD(wait_queue); // Hàng đợi giúp App "ngủ" khi không có người nhấn
static unsigned long last_jiffies = 0;      // Biến lưu thời gian để chống dội phím (Debounce)

/* * KHỐI 1: HÀM XỬ LÝ NGẮT (ISR)
 * Được gọi tự động bởi Kernel mỗi khi trạng thái điện áp tại GPIO 48 thay đổi.
 * Sử dụng `time_after` để bỏ qua các ngắt nhiễu xảy ra trong vòng 200ms.
 * Nếu là ngắt hợp lệ, cập nhật trạng thái và "đánh thức" hàng đợi (wake_up).
 */
static irqreturn_t button_irq_handler(int irq, void *dev_id) {
    int state;
    if (time_after(jiffies, last_jiffies + msecs_to_jiffies(200))) {
        state = gpio_get_value(BUTTON_GPIO);
        button_state = state ? '1' : '0';
        data_ready = true;
        last_jiffies = jiffies;
        wake_up_interruptible(&wait_queue);
    }
    return IRQ_HANDLED;
}

/* * KHỐI 2: HÀM ĐỌC DỮ LIỆU (READ)
 * Ứng dụng User gọi hàm read() sẽ vào đây.
 * Hàm sử dụng wait_event_interruptible để đưa tiến trình vào trạng thái ngủ (tiết kiệm CPU) 
 * cho đến khi biến data_ready = true (do ISR đánh thức).
 * Sau đó dùng copy_to_user để copy dữ liệu trạng thái nút nhấn từ Kernel lên User Space.
 */
static ssize_t dev_read(struct file *filep, char __user *buffer, size_t len, loff_t *offset) {
    char out_state;
    if (wait_event_interruptible(wait_queue, data_ready))
        return -ERESTARTSYS;

    out_state = button_state;
    data_ready = false;

    if (copy_to_user(buffer, &out_state, 1) != 0)
        return -EFAULT;

    return 1;
}

static struct file_operations fops = {
    .owner = THIS_MODULE,
    .read = dev_read,
};

/* * KHỐI 3: HÀM KHỞI TẠO MODULE (INIT)
 * Chạy một lần khi nạp module (insmod).
 * - Cấp phát Major Number, tạo file thiết bị tự động trong /dev/.
 * - Xin phép hệ điều hành cấp quyền sử dụng chân GPIO 48.
 * - Chuyển GPIO sang chế độ Input và ánh xạ nó sang một số ngắt (IRQ).
 * - Đăng ký hàm ngắt kích hoạt ở cả sườn lên (RISING) và xuống (FALLING).
 */
static int __init button_init(void) {
    int result;
    majorNumber = register_chrdev(0, DEVICE_NAME, &fops);
    if (majorNumber < 0) return majorNumber;

    buttonClass = class_create(THIS_MODULE, CLASS_NAME);
    if (IS_ERR(buttonClass)) {
        unregister_chrdev(majorNumber, DEVICE_NAME);
        return PTR_ERR(buttonClass);
    }

    buttonDevice = device_create(buttonClass, NULL, MKDEV(majorNumber, 0), NULL, DEVICE_NAME);
    if (IS_ERR(buttonDevice)) {
        class_destroy(buttonClass);
        unregister_chrdev(majorNumber, DEVICE_NAME);
        return PTR_ERR(buttonDevice);
    }

    if (!gpio_is_valid(BUTTON_GPIO)) {
        printk(KERN_ERR "Invalid GPIO\n");
        return -ENODEV;
    }
    gpio_request(BUTTON_GPIO, "sysfs");
    gpio_direction_input(BUTTON_GPIO);

    button_irq = gpio_to_irq(BUTTON_GPIO);
    result = request_irq(button_irq, button_irq_handler, IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING, "my_button_irq", NULL);

    printk(KERN_INFO "Button Driver Loaded on P9_15 (GPIO 48)\n");
    return 0;
}

/* * KHỐI 4: HÀM HỦY MODULE (EXIT)
 * Chạy khi gỡ module (rmmod). 
 * Giải phóng ngắt, giải phóng chân GPIO và hủy các file /dev/ tương ứng để tránh rò rỉ bộ nhớ.
 */
static void __exit button_exit(void) {
    free_irq(button_irq, NULL);
    gpio_free(BUTTON_GPIO);
    device_destroy(buttonClass, MKDEV(majorNumber, 0));
    class_destroy(buttonClass);
    unregister_chrdev(majorNumber, DEVICE_NAME);
    printk(KERN_INFO "Button Driver Removed\n");
}

module_init(button_init);
module_exit(button_exit);

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Interrupt-driven Button Driver on P9_15");

```

---

### 4. Tệp: `src/Makefile`

> **Mô tả chức năng:** Makefile của Kernel (Kbuild). Biến `obj-m` báo cho trình biên dịch biết mã nguồn C nào sẽ được build thành Kernel Module (`.ko`). Các lệnh `all` và `clean` trỏ đường dẫn tới thư viện của Linux Kernel để lấy các header cần thiết.

```make
obj-m += my_button.o

all:
	$(MAKE) -C $(LINUX_DIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(LINUX_DIR) M=$(PWD) clean

```

---

### 5. Cấu hình Pinmux Device Tree (`am335x-boneblack.dts`)

> **Mô tả chức năng:** File Device Tree (DTS) cấu hình phần cứng mức thấp. Nó can thiệp vào thanh ghi Pinmux của chip AM335x để ép chân P9_15 (Địa chỉ `0x0840`) hoạt động ở Mode 7 (chức năng GPIO), đồng thời bật điện trở kéo xuống (PULLDOWN) để tín hiệu mặc định luôn ở mức 0V (LOW) khi nút chưa được nhấn.

```dts
&am33xx_pinmux {
    button_pins: pinmux_button_pins {
        pinctrl-single,pins = <
            AM33XX_IOPAD(0x0840, PIN_INPUT_PULLDOWN | MUX_MODE7)
        >;
    };
};

/ {
    my_button_hw {
        compatible = "simple-bus";
        pinctrl-names = "default";
        pinctrl-0 = <&button_pins>;
    };
};

```

```

```
