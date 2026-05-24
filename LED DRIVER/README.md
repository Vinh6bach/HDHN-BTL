Trình điều khiển LED (GPIO 60) sử dụng kỹ thuật ánh xạ bộ nhớ vật lý `ioremap` trên AM335x.
### 1. Tệp cấu hình: `Config.in`

```kconfig
config BR2_PACKAGE_MY_GPIO_DRIVER
    bool "my_gpio_driver"
    depends on BR2_LINUX_KERNEL
    help
      GPIO driver + user app for LED control.

```

### 2. Tệp cấu hình: `my_gpio_driver.mk`

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

### 3. Mã nguồn Driver: `src/my_gpio_driver.c`

```c
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/io.h>
#include <linux/device.h>
#include <linux/uaccess.h>

#define DEVICE_NAME "my_gpio_dev"
#define CLASS_NAME  "my_gpio_class"

#define GPIO1_BASE         0x4804C000
#define GPIO1_SIZE         0x2000
#define GPIO_OE            0x134
#define GPIO_SETDATAOUT    0x194
#define GPIO_CLEARDATAOUT  0x190
#define GPIO_DATAIN        0x138
#define GPIO_60_BIT        (1 << 28)

static int majorNumber;
static struct class* gpioClass  = NULL;
static struct device* gpioDevice = NULL;
static void __iomem *gpio1_base_addr;

static int dev_open(struct inode *inodep, struct file *filep) {
    return 0;
}

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

static int __init gpio_driver_init(void) {
    majorNumber = register_chrdev(0, DEVICE_NAME, &fops);
    gpioClass = class_create(THIS_MODULE, CLASS_NAME);
    gpioDevice = device_create(gpioClass, NULL, MKDEV(majorNumber, 0), NULL, DEVICE_NAME);
    
    gpio1_base_addr = ioremap(GPIO1_BASE, GPIO1_SIZE);
    
    uint32_t oe_val = ioread32(gpio1_base_addr + GPIO_OE);
    oe_val &= ~GPIO_60_BIT;
    iowrite32(oe_val, gpio1_base_addr + GPIO_OE);
    
    printk(KERN_INFO "GPIO Driver: Loaded. GPIO 60 ready.\n");
    return 0;
}

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

### 4. Ứng dụng User App: `src/my_gpio_app.c`

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

### 5. Tệp: `src/Makefile`

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
