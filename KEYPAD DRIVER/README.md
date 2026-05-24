
# Keypad 1x4 Platform Driver

Trình điều khiển bàn phím Ma trận nút nhấn (1x4) sử dụng cấu trúc `platform_driver` và nạp thuộc tính phần cứng thông qua Device Tree (`of_match_table`).

---

### 1. Tệp cấu hình: `Config.in`
> **Mô tả chức năng:** Khai báo module Keypad vào danh sách các gói phần mềm của Buildroot, cho phép chọn biên dịch cùng Kernel.

```kconfig
config BR2_PACKAGE_KEYPAD2
    bool "keypad2"
    depends on BR2_LINUX_KERNEL
    help
      Kernel module for 1x4 Keypad using Interrupts.

```

---

### 2. Tệp cấu hình: `keypad2.mk`

> **Mô tả chức năng:** Kịch bản Buildroot biên dịch module bàn phím. Sử dụng biến `$(LINUX_VERSION_PROMOTED)/extra/` để sắp xếp module sau khi cài đặt vào đúng thư mục của hệ thống nhúng.

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

---

### 3. Mã nguồn Driver: `src/keypad2.c`

> **Mô tả chức năng tổng quan:** Trình điều khiển thiết bị nền tảng (Platform Driver). Nó tự động đọc các chân GPIO được định nghĩa trong Device Tree (DTS), cấu hình cả 4 chân này thành Input Interrupt. Sử dụng cơ chế khóa `mutex` và `wait_queue` để xử lý đồng bộ dữ liệu một cách an toàn khi có người dùng bấm phím.

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

// Cấu trúc quản lý trạng thái của toàn bộ Keypad
struct keypad_dev {
    struct keypad_btn btns[BTN_COUNT];
    dev_t dev_num;
    struct cdev cdev;
    struct class *class;
    struct device *device;
    unsigned long last_jiffies;
    char key_pressed;
    bool has_data;
    wait_queue_head_t wq; // Hàng đợi chờ phím
    struct mutex lock;    // Khóa Mutex chống xung đột luồng
};

static struct keypad_dev *kdev;

/* * KHỐI 1: XỬ LÝ NGẮT TỪ 4 NÚT NHẤN (ISR)
 * Hàm này dùng chung cho cả 4 nút. Tham số dev_id mang theo thông tin nút nào bị nhấn.
 * Lọc nhiễu (Debounce) 200ms, ghi nhận giá trị phím (1, 2, 3, hoặc 4) 
 * và đánh thức tiến trình đang ngủ chờ trong hàng đợi.
 */
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

/* * KHỐI 2: ĐỌC GIÁ TRỊ PHÍM TỪ USER SPACE
 * Khi ứng dụng gọi read(), nó sẽ bị ép ngủ nếu chưa có nút nào được nhấn.
 * Sử dụng mutex_lock để đảm bảo biến kdev->key_pressed không bị thay đổi bởi ngắt 
 * trong lúc Kernel đang copy dữ liệu lên User Space.
 */
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

/* * KHỐI 3: PLATFORM PROBE (LIÊN KẾT DEVICE TREE)
 * Khi Kernel phân tích file DTS và thấy nhãn "my,keypad-1x4", nó sẽ tự động gọi hàm probe.
 * Hàm này trích xuất cấu hình GPIO từ thuộc tính "keypad-gpios" trong Device Tree.
 * Vòng lặp duyệt qua 4 GPIO, đăng ký cấp phát ngắt tương ứng cho từng chân (Sườn xuống).
 */
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

/* * KHỐI 4: HÀM HỦY THIẾT BỊ (REMOVE)
 * Thu hồi ngắt, giải phóng GPIO và xóa các Device Node khi gỡ module.
 */
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

// Bảng Match Table: Định danh phần cứng tương ứng trong Device Tree
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

---

### 4. Tệp: `src/Makefile`

> **Mô tả chức năng:** Kbuild Makefile sử dụng biến `obj-m` để báo cho trình biên dịch biết mã nguồn C nào sẽ được build thành Kernel Module.

```make
obj-m += keypad2.o

all:
	$(MAKE) -C $(LINUX_DIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(LINUX_DIR) M=$(PWD) clean

```

```


```
