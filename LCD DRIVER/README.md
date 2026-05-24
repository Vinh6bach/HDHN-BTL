
# LCD I2C Driver

Trình điều khiển màn hình LCD 16x2 thông qua Module I2C chuyển đổi (Địa chỉ mặc định 0x27 trên Bus I2C-2).

---

### 1. Tệp cấu hình: `Config.in`
> **Mô tả chức năng:** Khai báo gói phần mềm `lcd_driver` vào menu cấu hình của Buildroot, cho phép chọn biên dịch Driver này cùng với Linux Kernel.

```kconfig
config BR2_PACKAGE_LCD_DRIVER
    bool "lcd_driver"
    depends on BR2_LINUX_KERNEL
    help
      Kernel module for LCD I2C display.

```

---

### 2. Tệp cấu hình: `lcd_driver.mk`

> **Mô tả chức năng:** Makefile của Buildroot dùng để chỉ đạo quá trình biên dịch chéo (Cross-compile). Nó truyền các biến như môi trường kiến trúc (`ARCH`) và trình biên dịch chéo (`CROSS_COMPILE`) vào Makefile của Kernel, sau đó copy file `.ko` vào thư mục `/lib/modules/` của bo mạch.

```make
LCD_DRIVER_VERSION = 1.0
LCD_DRIVER_SITE = $(TOPDIR)/package/lcd_driver/src
LCD_DRIVER_SITE_METHOD = local

define LCD_DRIVER_BUILD_CMDS
	$(MAKE) -C $(LINUX_DIR) \
		ARCH=$(KERNEL_ARCH) \
		CROSS_COMPILE=$(TARGET_CROSS) \
		M=$(@D) modules
endef

define LCD_DRIVER_INSTALL_TARGET_CMDS
	$(INSTALL) -D -m 0755 $(@D)/lcd_driver.ko $(TARGET_DIR)/lib/modules/lcd_driver.ko
endef

$(eval $(kernel-module))
$(eval $(generic-package))

```

---

### 3. Mã nguồn Driver: `src/lcd_driver.c`

> **Mô tả chức năng tổng quan:** Trình điều khiển I2C Client kết hợp Character Device. Nó tự động dò tìm địa chỉ 0x27 trên I2C bus 2, sau đó tạo file `/dev/lcd_dev`. Khi User Space ghi chuỗi vào file này, Driver sẽ chuyển đổi chuỗi thành các tín hiệu I2C điều khiển module mở rộng PCF8574.

```c
#include <linux/module.h>
#include <linux/i2c.h>
#include <linux/fs.h>
#include <linux/uaccess.h>
#include <linux/delay.h>
#include <linux/device.h>
#include <linux/cdev.h>

#define DEVICE_NAME "lcd_dev"
#define CLASS_NAME "lcd_class"
#define LCD_BACKLIGHT 0x08
#define LCD_ENABLE 0x04
#define LCD_RS 0x01

static struct i2c_client *lcd_client;
static dev_t dev_num;
static struct cdev lcd_cdev;
static struct class *lcd_class;
static struct device *lcd_device;
static struct i2c_client *auto_client;

/* * KHỐI 1: GIAO TIẾP I2C (MỨC THẤP)
 * Module I2C PCF8574 giao tiếp với LCD theo chế độ 4-bit.
 * Hàm này tách 1 byte dữ liệu thành 2 nửa (nibble cao và thấp),
 * kết hợp với các bit điều khiển (Đèn nền, Enable) rồi gửi qua I2C.
 */
static void lcd_send_byte(uint8_t val, uint8_t mode) {
    uint8_t buf[4];
    uint8_t hi = (val & 0xf0) | mode | LCD_BACKLIGHT;
    uint8_t lo = ((val << 4) & 0xf0) | mode | LCD_BACKLIGHT;
    
    buf[0] = hi | LCD_ENABLE;
    buf[1] = hi;
    buf[2] = lo | LCD_ENABLE;
    buf[3] = lo;
    i2c_master_send(lcd_client, buf, 4);
}

/* * KHỐI 2: KHỞI TẠO MÀN HÌNH LCD
 * Gửi chuỗi lệnh khởi tạo chuẩn của HD44780 để cấu hình LCD 
 * hoạt động ở chế độ 4-bit, 2 dòng, bật hiển thị và xóa màn hình.
 */
static void lcd_init(void) {
    mdelay(50);
    lcd_send_byte(0x33, 0);
    mdelay(5);
    lcd_send_byte(0x32, 0);
    mdelay(5);
    lcd_send_byte(0x28, 0);
    lcd_send_byte(0x0C, 0);
    lcd_send_byte(0x06, 0);
    lcd_send_byte(0x01, 0);
    mdelay(5);
}

/* * KHỐI 3: HÀM GHI DỮ LIỆU (WRITE) TỪ USER SPACE
 * Nhận chuỗi ký tự từ hàm write() của ứng dụng User.
 * Xử lý đặc biệt các ký tự điều khiển:
 * - '\n': Chuyển con trỏ xuống dòng thứ 2 (Lệnh 0xC0).
 * - '\f' hoặc '\r': Xóa toàn bộ màn hình (Lệnh 0x01).
 * Các ký tự khác được in ra bình thường.
 */
static ssize_t lcd_write(struct file *f, const char __user *buf, size_t len, loff_t *off) {
    char kbuf[64];
    int i;
    size_t count = (len > sizeof(kbuf)) ? sizeof(kbuf) : len;
    
    if (copy_from_user(kbuf, buf, count))
        return -EFAULT;
        
    for (i = 0; i < count; i++) {
        if (kbuf[i] == '\n') {
            lcd_send_byte(0xC0, 0);
        } else if (kbuf[i] == '\f' || kbuf[i] == '\r') {
            lcd_send_byte(0x01, 0);
            mdelay(2);
        } else {
            lcd_send_byte(kbuf[i], LCD_RS);
        }
    }
    return len;
}

static struct file_operations fops = {
    .owner = THIS_MODULE,
    .write = lcd_write,
};

/* * KHỐI 4: PROBE & INIT (CẤU HÌNH THIẾT BỊ)
 * Hàm probe được Kernel gọi khi phát hiện thiết bị I2C khớp địa chỉ.
 * Nó đăng ký Character Device, khởi tạo class sysfs và chạy lệnh lcd_init().
 * Hàm my_lcd_init ép Kernel tìm kiếm thiết bị ở bus i2c-2 với địa chỉ 0x27.
 */
static int lcd_probe(struct i2c_client *client) {
    int ret;
    lcd_client = client;
    ret = alloc_chrdev_region(&dev_num, 0, 1, DEVICE_NAME);
    if (ret < 0) return ret;
    
    cdev_init(&lcd_cdev, &fops);
    ret = cdev_add(&lcd_cdev, dev_num, 1);
    if (ret < 0) {
        unregister_chrdev_region(dev_num, 1);
        return ret;
    }
    
    lcd_class = class_create(THIS_MODULE, CLASS_NAME);
    if (IS_ERR(lcd_class)) {
        cdev_del(&lcd_cdev);
        unregister_chrdev_region(dev_num, 1);
        return PTR_ERR(lcd_class);
    }
    
    lcd_device = device_create(lcd_class, NULL, dev_num, NULL, DEVICE_NAME);
    if (IS_ERR(lcd_device)) {
        class_destroy(lcd_class);
        cdev_del(&lcd_cdev);
        unregister_chrdev_region(dev_num, 1);
        return PTR_ERR(lcd_device);
    }
    
    lcd_init();
    printk(KERN_INFO "LCD I2C Driver Probed. Major: %d, Minor: %d\n", MAJOR(dev_num), MINOR(dev_num));
    return 0;
}

static void lcd_remove(struct i2c_client *client) {
    device_destroy(lcd_class, dev_num);
    class_destroy(lcd_class);
    cdev_del(&lcd_cdev);
    unregister_chrdev_region(dev_num, 1);
    printk(KERN_INFO "LCD I2C Driver Removed\n");
}

static const struct i2c_device_id lcd_id[] = {
    {"lcd_i2c", 0},
    {}
};
MODULE_DEVICE_TABLE(i2c, lcd_id);

static struct i2c_driver lcd_i2c_driver = {
    .driver = { .name = "lcd_i2c" },
    .probe = lcd_probe,
    .remove = lcd_remove,
    .id_table = lcd_id,
};

static int __init my_lcd_init(void) {
    int ret;
    struct i2c_adapter *adapter;
    struct i2c_board_info board_info = {
        I2C_BOARD_INFO("lcd_i2c", 0x27),
    };
    
    ret = i2c_add_driver(&lcd_i2c_driver);
    if (ret) return ret;
    
    adapter = i2c_get_adapter(2);
    if (adapter) {
        auto_client = i2c_new_client_device(adapter, &board_info);
        i2c_put_adapter(adapter);
    } else {
        printk(KERN_ERR "LCD: Không tìm thấy bus i2c-2\n");
    }
    return 0;
}

static void __exit my_lcd_exit(void) {
    if (auto_client)
        i2c_unregister_device(auto_client);
    i2c_del_driver(&lcd_i2c_driver);
}

module_init(my_lcd_init);
module_exit(my_lcd_exit);
MODULE_LICENSE("GPL");

```

---

### 4. Tệp: `src/Makefile`

> **Mô tả chức năng:** Kbuild Makefile biên dịch file `lcd_driver.c` thành module `lcd_driver.ko`.

```make
obj-m += lcd_driver.o

all:
	$(MAKE) -C $(LINUX_DIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(LINUX_DIR) M=$(PWD) clean

```

```



```
