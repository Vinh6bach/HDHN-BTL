


# Keypad 1x4 Platform Driver

Trình điều khiển bàn phím Ma trận nút nhấn (1x4) sử dụng cấu trúc `platform_driver` và nạp thuộc tính phần cứng thông qua Device Tree (`of_match_table`).


### 1. Tệp cấu hình: `Config.in`

```kconfig
config BR2_PACKAGE_KEYPAD2
    bool "keypad2"
    depends on BR2_LINUX_KERNEL
    help
      Kernel module for 1x4 Keypad using Interrupts.

```

### 2. Tệp cấu hình: `keypad2.mk`

```make
KEYPAD2_VERSION = 1.0
KEYPAD2_SITE = $(TOPDIR)/package/keypad2/src
KEYPAD2_SITE_METHOD = local
KEYPAD2_LICENSE = GPL-2.0

define KEYPAD2_BUILD_CMDS
	$(MAKE) -C $(LINUX_DIR) $(LINUX_MAKE_FLAGS) M=$(@D) modules
endef

define KEYPAD2_INSTALL_TARGET_CMDS
	$(INSTALL) -D -m 0644 $(@D)/keypad2.ko $(TARGET_DIR)/lib/modules/$(LINUX_VERSION_PROMOTED)/extra/keypad2.ko
endef

$(eval $(kernel-module))
$(eval $(generic-package))

```

### 3. Mã nguồn Driver: `src/keypad2.c`

```c
#include <linux/module.h>
#include <linux/gpio.h>
#include <linux/interrupt.h>
#include <linux/of_gpio.h>
#include <linux/platform_device.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/jiffies.h>
#include <linux/uaccess.h>
#include <linux/wait.h>
#include <linux/slab.h>
#include <linux/mutex.h>

#define DEVICE_NAME "keypad2"
#define BTN_COUNT 4

struct keypad_btn {
    int gpio;
    int irq;
    char val;
};

struct keypad_dev {
    struct keypad_btn btns[BTN_COUNT];
    dev_t dev_num;
    struct cdev cdev;
    struct class *class;
    struct device *device;
    unsigned long last_jiffies;
    char key_pressed;
    bool has_data;
    wait_queue_head_t wq;
    struct mutex lock;
};

static struct keypad_dev *kdev;

static irqreturn_t keypad_handler(int irq, void *dev_id) {
    struct keypad_btn *b = (struct keypad_btn *)dev_id;
    if (time_after(jiffies, kdev->last_jiffies + msecs_to_jiffies(200))) {
        kdev->key_pressed = b->val;
        kdev->has_data = true;
        kdev->last_jiffies = jiffies;
        wake_up_interruptible(&kdev->wq);
    }
    return IRQ_HANDLED;
}

static ssize_t keypad_read(struct file *f, char __user *buf, size_t len, loff_t *off) {
    char out[2];
    if (wait_event_interruptible(kdev->wq, kdev->has_data)) return -ERESTARTSYS;
    
    mutex_lock(&kdev->lock);
    out[0] = kdev->key_pressed;
    out[1] = '\n';
    kdev->has_data = false;
    mutex_unlock(&kdev->lock);
    
    if (copy_to_user(buf, out, 2)) return -EFAULT;
    return 2;
}

static struct file_operations fops = {
    .owner = THIS_MODULE,
    .read = keypad_read,
};

static int keypad_probe(struct platform_device *pdev) {
    struct device_node *np = pdev->dev.of_node;
    int i, ret;
    char names[4] = {'1', '2', '3', '4'};
    
    kdev = kzalloc(sizeof(*kdev), GFP_KERNEL);
    if (!kdev) return -ENOMEM;
    
    init_waitqueue_head(&kdev->wq);
    mutex_init(&kdev->lock);
    
    ret = alloc_chrdev_region(&kdev->dev_num, 0, 1, DEVICE_NAME);
    if (ret < 0) return ret;
    
    cdev_init(&kdev->cdev, &fops);
    kdev->cdev.owner = THIS_MODULE;
    cdev_add(&kdev->cdev, kdev->dev_num, 1);
    
    kdev->class = class_create(THIS_MODULE, DEVICE_NAME);
    if (IS_ERR(kdev->class)) {
        ret = PTR_ERR(kdev->class);
        goto err_class;
    }
    kdev->device = device_create(kdev->class, NULL, kdev->dev_num, NULL, DEVICE_NAME);
    
    for (i = 0; i < BTN_COUNT; i++) {
        kdev->btns[i].gpio = of_get_named_gpio(np, "keypad-gpios", i);
        kdev->btns[i].val = names[i];
        gpio_request(kdev->btns[i].gpio, "kp-btn");
        gpio_direction_input(kdev->btns[i].gpio);
        kdev->btns[i].irq = gpio_to_irq(kdev->btns[i].gpio);
        request_irq(kdev->btns[i].irq, keypad_handler, IRQF_TRIGGER_FALLING, "kp-irq", &kdev->btns[i]);
    }
    return 0;

err_class:
    unregister_chrdev_region(kdev->dev_num, 1);
    return ret;
}

static void keypad_remove(struct platform_device *pdev) {
    int i;
    for (i = 0; i < BTN_COUNT; i++) {
        free_irq(kdev->btns[i].irq, &kdev->btns[i]);
        gpio_free(kdev->btns[i].gpio);
    }
    device_destroy(kdev->class, kdev->dev_num);
    class_destroy(kdev->class);
    cdev_del(&kdev->cdev);
    unregister_chrdev_region(kdev->dev_num, 1);
    kfree(kdev);
}

static const struct of_device_id kp_match[] = {
    { .compatible = "my,keypad-1x4", },
    { }
};
MODULE_DEVICE_TABLE(of, kp_match);

static struct platform_driver kp_driver = {
    .probe = keypad_probe,
    .remove = keypad_remove,
    .driver = { .name = "keypad2", .of_match_table = kp_match, },
};

module_platform_driver(kp_driver);
MODULE_LICENSE("GPL");

```

### 4. Tệp: `src/Makefile`

```make
obj-m += keypad2.o

all:
	$(MAKE) -C $(LINUX_DIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(LINUX_DIR) M=$(PWD) clean

```

```

Sau khi paste xong, bạn sang tab `Preview` ngó xem nó đã chia thành 4 ô vuông vức chưa nhé! Nhắn "xong" để lấy file App Test cuối cùng nha.

```
