
# Smart Door User Application (Đa luồng & Auto-boot)

Chương trình này là lớp Ứng dụng (User Space Application), đóng vai trò "bộ não" liên kết tất cả 4 Driver (Keypad, Button, LCD, LED) lại với nhau. Nó sử dụng cơ chế **Đa luồng (Pthread)** và **Khóa đồng bộ (Mutex)** để xử lý song song các tác vụ, đồng thời tích hợp kịch bản tự động chạy khi bo mạch vừa cấp nguồn.

---

### 1. Tệp cấu hình: `Config.in`
> **Mô tả chức năng:** Thêm ứng dụng `app_test` vào menu của Buildroot để hệ thống biết và đem đi biên dịch chéo ra mã máy của chip ARM.

```kconfig
config BR2_PACKAGE_APP_TEST
    bool "app_test"
    depends on BR2_LINUX_KERNEL
    help
      Sequential App to test Keypad and LCD with password masking (Manual Run).

```

---

### 2. Tệp cấu hình: `app_test.mk`

> **Mô tả chức năng:** Kịch bản Buildroot cho Ứng dụng. Điểm đặc biệt nhất nằm ở biến `APP_TEST_INSTALL_INIT_SYSV`. Khi Buildroot thấy lệnh này, nó sẽ tự động lấy file script `S99app_test` (viết ở phần 4) copy vào thư mục `/etc/init.d/` của bo mạch. Nhờ đó, app sẽ **tự động chạy ngay khi bật nguồn** mà không cần cắm cáp kết nối máy tính để gõ lệnh thủ công.

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

---

### 3. Mã nguồn Chương trình chính: `src/app_test.c`

> **Mô tả luồng hoạt động chi tiết:** > Hệ thống chạy 2 luồng (Thread) song song.
> * **Luồng 1 (Keypad Thread)**: Chuyên túc trực chờ người dùng bấm số. Khi có số, nó giấu số đó thành dấu sao `*` in lên màn hình LCD và lưu vào mảng bộ nhớ.
> * **Luồng 2 (Button Thread)**: Chuyên túc trực chờ người dùng bấm nút Xác nhận (Enter). Khi bấm, nó sẽ đem mảng bộ nhớ kia đi so sánh với mật khẩu gốc `"1234"`. Nếu đúng, nó ra lệnh cho Driver LCD in chữ "Correct!" và ra lệnh cho Driver GPIO bật sáng đèn LED trong 3 giây.
> * **Khóa Mutex**: Đảm bảo an toàn dữ liệu. Ngăn chặn lỗi "xung đột" khi người dùng vừa bấm phím số, lại vừa bấm phím Enter cùng một lúc khiến hệ thống bị treo.
> 
> 

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <signal.h>
#include <pthread.h>

int fd_keypad = -1, fd_lcd = -1, fd_btn = -1, fd_led = -1;
char input[5] = {0}; // Mảng lưu trữ tối đa 4 ký tự mật khẩu
int input_count = 0; // Biến đếm số lượng ký tự đã nhập
pthread_mutex_t lock; // Biến khóa bảo vệ tài nguyên dùng chung

/* * KHỐI 1: BẮT SỰ KIỆN TẮT (Ctrl+C hoặc lệnh kill)
 * Nếu không có hàm này, khi ứng dụng bị tắt đột ngột, đèn LED có thể vẫn sáng
 * và màn hình LCD vẫn hiện chữ cũ. Hàm này đảm bảo khi thoát app, màn hình 
 * sẽ được xóa sạch (\f) và đèn LED bị tắt đi, các file /dev/ được đóng an toàn.
 */
void sigint_handler(int sig_num) {
    printf("\n[Hệ thống] Đang dọn dẹp và thoát...\n");
    if (fd_lcd >= 0) { dprintf(fd_lcd, "\f"); close(fd_lcd); }
    if (fd_led >= 0) { write(fd_led, "0", 1); close(fd_led); }
    if (fd_keypad >= 0) close(fd_keypad);
    if (fd_btn >= 0) close(fd_btn);
    exit(0);
}

// Hàm hỗ trợ vẽ lại giao diện chờ nhập: Hiển thị chữ "Password:" và in ra các dấu * tương ứng số phím đã bấm.
void update_lcd_waiting(void) {
    char stars[5] = {0};
    int i;
    for (i = 0; i < input_count; i++) stars[i] = '*';
    dprintf(fd_lcd, "\fPassword:\n%s", stars);
}

/* * KHỐI 2: LUỒNG BÀN PHÍM (Nhiệm vụ: Ghi nhận số)
 * Hàm read() sẽ nằm "ngủ" hoàn toàn (không tốn % CPU nào) cho đến khi 
 * Driver Keypad gửi tín hiệu đánh thức (wake_up).
 * Khi tỉnh dậy, nó khóa Mutex -> Lưu phím vừa bấm vào mảng -> Cập nhật LCD -> Mở Mutex.
 */
void *keypad_thread(void *arg) {
    char key_buf[2];
    while (1) {
        if (read(fd_keypad, key_buf, 2) > 0) {
            pthread_mutex_lock(&lock);
            if (input_count < 4) { // Chỉ cho phép nhập tối đa 4 số
                input[input_count++] = key_buf[0];
                update_lcd_waiting();
            }
            pthread_mutex_unlock(&lock);
        }
    }
    return NULL;
}

/* * KHỐI 3: LUỒNG NÚT NHẤN (Nhiệm vụ: Xác nhận & Điều khiển)
 * Tương tự luồng phím, hàm read() nằm "ngủ" chờ ngắt từ Driver Button.
 * Khi nút được bấm (b_state == '1'), nó khóa Mutex lại để kiểm tra:
 * - Nếu nhập chưa đủ 4 số: Bỏ qua không làm gì.
 * - Nếu đúng 4 số: Dùng hàm strcmp() đối chiếu với "1234".
 * + Đúng: In "Correct!", ghi '1' vào fd_led để bật LED báo mở cửa, chờ 3 giây rồi tắt LED.
 * + Sai: In "Incorrect!", chờ 3 giây.
 * Sau khi xử lý xong, reset toàn bộ mảng mật khẩu về 0 và vẽ lại màn hình chờ.
 */
void *button_thread(void *arg) {
    char b_state;
    while (1) {
        if (read(fd_btn, &b_state, 1) > 0) {
            if (b_state == '1') {
                pthread_mutex_lock(&lock);
                if (input_count == 4) {
                    if (strcmp(input, "1234") == 0) {
                        dprintf(fd_lcd, "\fCorrect!");
                        write(fd_led, "1", 1); // Bật LED
                        sleep(3);
                        write(fd_led, "0", 1); // Tắt LED
                    } else {
                        dprintf(fd_lcd, "\fIncorrect!");
                        sleep(3);
                    }
                    memset(input, 0, sizeof(input)); // Xóa trắng mật khẩu đã nhập
                    input_count = 0;
                    update_lcd_waiting();
                }
                pthread_mutex_unlock(&lock);
            }
        }
    }
    return NULL;
}

/* * KHỐI 4: HÀM MAIN VÀ QUẢN LÝ LUỒNG
 * Mở đồng loạt 4 file thiết bị trong /dev/ (được tạo ra bởi 4 Driver trước đó).
 * Tạo 2 luồng (t_keypad và t_button) chạy song song với nhau.
 * Hàm pthread_join() ép chương trình chính chờ đợi, giúp App không bị văng ra ngoài terminal.
 */
int main(void) {
    signal(SIGINT, sigint_handler);
    pthread_mutex_init(&lock, NULL);
    
    // Mở liên kết với 4 Driver phần cứng
    fd_keypad = open("/dev/keypad2", O_RDONLY);
    fd_lcd = open("/dev/lcd_dev", O_WRONLY);
    fd_btn = open("/dev/my_button_dev", O_RDONLY);
    fd_led = open("/dev/my_gpio_dev", O_WRONLY);
    
    if (fd_keypad < 0 || fd_lcd < 0 || fd_btn < 0 || fd_led < 0) {
        printf("Lỗi: Không thể mở đủ 4 thiết bị trong /dev/ (Chưa load module?)\n");
        return -1;
    }
    printf("Smart Door App Test (Multi-threaded) Started...\n");

    pthread_mutex_lock(&lock);
    update_lcd_waiting();
    pthread_mutex_unlock(&lock);

    // Kích hoạt đa luồng
    pthread_t t_keypad, t_button;
    pthread_create(&t_keypad, NULL, keypad_thread, NULL);
    pthread_create(&t_button, NULL, button_thread, NULL);

    pthread_join(t_keypad, NULL);
    pthread_join(t_button, NULL);
    return 0;
}

```

---

### 4. Script Khởi chạy cùng hệ thống: `src/S99app_test`

> **Mô tả chức năng:** Kịch bản Shell tuân thủ chuẩn SysV Init của Linux nhúng. Khi bo mạch bật nguồn, hệ điều hành sẽ quét thư mục `/etc/init.d/` và chạy các script bắt đầu bằng chữ "S" (Start).
> Kịch bản này làm 3 việc liên tiếp:
> 1. Dùng lệnh `modprobe` để gọi 4 Kernel Module thức dậy (Bắt buộc phải có vì module tạo ra các file `/dev/`).
> 2. Ngủ 2 giây (`sleep 2`) để chờ hệ thống Linux cấp phát xong file `/dev/` ổn định.
> 3. Kích hoạt App chạy ngầm (Dấu `&` ở cuối lệnh `/usr/bin/app_test &`).
> 
> 

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

---

### 5. Tệp: `src/Makefile`

> **Mô tả chức năng:** Lệnh build ứng dụng. Đặc biệt chú ý cờ `-static` (nhúng toàn bộ thư viện tĩnh vào file thực thi để chống lỗi thiếu thư viện `.so` trên bo mạch) và cờ `-lpthread` (liên kết thư viện Đa luồng POSIX Threads của Linux).

```make
all:
	$(CC) app_test.c -o app_test -static -lpthread

clean:
	rm -f app_test

```

```


```
