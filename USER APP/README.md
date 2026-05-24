
# Smart Door User Application (Đa luồng & Auto-boot)

Ứng dụng User Space xử lý logic đa luồng sử dụng `POSIX Threads` (Pthread) kết hợp đồng bộ hóa bằng `Mutex` bảo vệ tài nguyên dùng chung. App đồng thời tích hợp Script cấu hình Init tự khởi động cùng hệ thống Buildroot khi vừa bật bo mạch.


### 1. Tệp cấu hình: `Config.in`

```kconfig
config BR2_PACKAGE_APP_TEST
    bool "app_test"
    depends on BR2_LINUX_KERNEL
    help
      Sequential App to test Keypad and LCD with password masking (Manual Run).

```

### 2. Tệp cấu hình: `app_test.mk`

```make
APP_TEST_VERSION = 1.0
APP_TEST_SITE = $(TOPDIR)/package/app_test/src
APP_TEST_SITE_METHOD = local

define APP_TEST_BUILD_CMDS
	$(MAKE) CC="$(TARGET_CC)" -C $(@D) all
endef

define APP_TEST_INSTALL_TARGET_CMDS
	$(INSTALL) -D -m 0755 $(@D)/app_test $(TARGET_DIR)/usr/bin/app_test
endef

define APP_TEST_INSTALL_INIT_SYSV
	$(INSTALL) -D -m 0755 $(@D)/S99app_test $(TARGET_DIR)/etc/init.d/S99app_test
endef

$(eval $(generic-package))

```

### 3. Mã nguồn Chương trình chính: `src/app_test.c`

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <signal.h>
#include <pthread.h>

int fd_keypad = -1, fd_lcd = -1, fd_btn = -1, fd_led = -1;
char input[5] = {0};
int input_count = 0;
pthread_mutex_t lock;

void sigint_handler(int sig_num) {
    printf("\n[Hệ thống] Đang dọn dẹp và thoát...\n");
    if (fd_lcd >= 0) { dprintf(fd_lcd, "\f"); close(fd_lcd); }
    if (fd_led >= 0) { write(fd_led, "0", 1); close(fd_led); }
    if (fd_keypad >= 0) close(fd_keypad);
    if (fd_btn >= 0) close(fd_btn);
    exit(0);
}

void update_lcd_waiting(void) {
    char stars[5] = {0};
    int i;
    for (i = 0; i < input_count; i++) stars[i] = '*';
    dprintf(fd_lcd, "\fPassword:\n%s", stars);
}

void *keypad_thread(void *arg) {
    char key_buf[2];
    while (1) {
        if (read(fd_keypad, key_buf, 2) > 0) {
            pthread_mutex_lock(&lock);
            if (input_count < 4) {
                input[input_count++] = key_buf[0];
                update_lcd_waiting();
            }
            pthread_mutex_unlock(&lock);
        }
    }
    return NULL;
}

void *button_thread(void *arg) {
    char b_state;
    while (1) {
        if (read(fd_btn, &b_state, 1) > 0) {
            if (b_state == '1') {
                pthread_mutex_lock(&lock);
                if (input_count == 4) {
                    if (strcmp(input, "1234") == 0) {
                        dprintf(fd_lcd, "\fCorrect!");
                        write(fd_led, "1", 1);
                        sleep(3);
                        write(fd_led, "0", 1);
                    } else {
                        dprintf(fd_lcd, "\fIncorrect!");
                        sleep(3);
                    }
                    memset(input, 0, sizeof(input));
                    input_count = 0;
                    update_lcd_waiting();
                }
                pthread_mutex_unlock(&lock);
            }
        }
    }
    return NULL;
}

int main(void) {
    signal(SIGINT, sigint_handler);
    pthread_mutex_init(&lock, NULL);
    
    fd_keypad = open("/dev/keypad2", O_RDONLY);
    fd_lcd = open("/dev/lcd_dev", O_WRONLY);
    fd_btn = open("/dev/my_button_dev", O_RDONLY);
    fd_led = open("/dev/my_gpio_dev", O_WRONLY);
    
    if (fd_keypad < 0 || fd_lcd < 0 || fd_btn < 0 || fd_led < 0) {
        printf("Lỗi: Không thể mở đủ 4 thiết bị trong /dev/\n");
        return -1;
    }
    printf("Smart Door App Test (Multi-threaded) Started...\n");

    pthread_mutex_lock(&lock);
    update_lcd_waiting();
    pthread_mutex_unlock(&lock);

    pthread_t t_keypad, t_button;
    pthread_create(&t_keypad, NULL, keypad_thread, NULL);
    pthread_create(&t_button, NULL, button_thread, NULL);

    pthread_join(t_keypad, NULL);
    pthread_join(t_button, NULL);
    return 0;
}

```

### 4. Script Khởi chạy cùng hệ thống: `src/S99app_test`

```sh
#!/bin/sh
case "$1" in
  start)
    echo "[Hệ thống] Đang tự động nạp các Kernel Module..."
    modprobe my_gpio_driver
    modprobe my_button
    modprobe lcd_driver
    modprobe keypad2

    echo "[Hệ thống] Đang chờ khởi tạo file thiết bị..."
    sleep 2

    echo "[Hệ thống] Khởi chạy ứng dụng App Test..."
    /usr/bin/app_test &
    ;;
  stop)
    echo "[Hệ thống] Đang dừng ứng dụng App Test..."
    killall app_test
    ;;
  restart)
    $0 stop
    sleep 1
    $0 start
    ;;
  *)
    echo "Sử dụng: $0 {start|stop|restart}"
    exit 1
    ;;
esac
exit 0

```

### 5. Tệp: `src/Makefile`

```make
all:
	$(CC) app_test.c -o app_test -static -lpthread

clean:
	rm -f app_test

```

