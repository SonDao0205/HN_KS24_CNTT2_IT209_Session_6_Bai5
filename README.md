# Bài 5: Thiết kế và Cấu hình dịch vụ Systemd Service cho ứng dụng Java

## 1. Mục tiêu

- Thiết kế cấu hình Systemd Service file để hệ điều hành quản lý vòng đời ứng dụng tự động.
- Chạy ứng dụng dưới quyền user giới hạn (`java-runner`, non-root) để nâng cao bảo mật.
- Cài đặt chính sách **tự động restart** dịch vụ khi có sự cố (on-failure, sau 10 giây).

---

## 2. Các bước thực hiện

### Bước 1: Tạo tài khoản hệ thống `java-runner`

```bash
sudo useradd -r -s /usr/sbin/nologin java-runner
```

> **Giải thích tham số:**
>
> | Tham số | Ý nghĩa |
> |---|---|
> | `-r` | Tạo tài khoản hệ thống (system account), UID thấp, không tạo home directory |
> | `-s /usr/sbin/nologin` | Không cho phép đăng nhập shell trực tiếp — tăng bảo mật |

Kiểm tra user đã tạo:

```bash
id java-runner
```

**Kết quả mong đợi:**

```
uid=999(java-runner) gid=999(java-runner) groups=999(java-runner)
```

---

### Bước 2: Tạo tệp cấu hình dịch vụ Systemd

Tạo file tại đường dẫn `/etc/systemd/system/java-app.service`:

```bash
sudo nano /etc/systemd/system/java-app.service
```

### Nội dung file `java-app.service`:

```ini
[Unit]
Description=Java Spring Boot Application Service
Documentation=https://docs.spring.io/spring-boot/docs/current/reference/html/
After=network.target
Wants=network.target

[Service]
# Tài khoản hệ thống chạy dịch vụ (non-root, không có shell đăng nhập)
User=java-runner
Group=java-runner

# Lệnh thực thi chính
# Mô phỏng ứng dụng Java bằng lệnh sleep dài hạn
# Trong thực tế sẽ thay bằng: /usr/bin/java -jar /opt/java-app/app.jar
ExecStart=/usr/bin/sleep infinity

# Chính sách restart: chỉ restart khi tiến trình bị sập bất thường
Restart=on-failure

# Thời gian chờ trước khi restart (10 giây)
RestartSec=10

# Giới hạn số lần restart liên tiếp trong khoảng thời gian
StartLimitIntervalSec=60
StartLimitBurst=5

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=java-app

[Install]
# Kích hoạt dịch vụ trong chế độ multi-user (không cần GUI)
WantedBy=multi-user.target
```

> **Giải thích cấu hình chi tiết:**
>
> | Directive | Ý nghĩa |
> |---|---|
> | `After=network.target` | Dịch vụ chỉ khởi động sau khi mạng sẵn sàng |
> | `User=java-runner` | Chạy dưới quyền user `java-runner` (non-root) |
> | `ExecStart=/usr/bin/sleep infinity` | Lệnh thực thi chính (mô phỏng ứng dụng Java) |
> | `Restart=on-failure` | Tự động restart khi tiến trình thoát với exit code khác 0 |
> | `RestartSec=10` | Chờ 10 giây trước mỗi lần restart |
> | `StartLimitIntervalSec=60` | Giới hạn restart trong khoảng 60 giây |
> | `StartLimitBurst=5` | Tối đa 5 lần restart trong khoảng thời gian trên |
> | `WantedBy=multi-user.target` | Kích hoạt dịch vụ khi boot vào chế độ multi-user |

---

## 3. Kiểm tra kết quả

### 3.1. Nạp lại cấu hình Systemd

```bash
sudo systemctl daemon-reload
```

### 3.2. Khởi chạy dịch vụ

```bash
sudo systemctl start java-app.service
```

### 3.3. Kiểm tra trạng thái dịch vụ

```bash
sudo systemctl status java-app.service
```

**Kết quả mong đợi:**

```
● java-app.service - Java Spring Boot Application Service
     Loaded: loaded (/etc/systemd/system/java-app.service; disabled; vendor preset: enabled)
     Active: active (running) since Tue 2026-10-06 14:30:00 UTC; 1min ago
       Docs: https://docs.spring.io/spring-boot/docs/current/reference/html/
   Main PID: 31245 (sleep)
      Tasks: 1 (limit: 4915)
     Memory: 200.0K
        CPU: 2ms
     CGroup: /system.slice/java-app.service
             └─31245 /usr/bin/sleep infinity

Oct 06 14:30:00 hostname systemd[1]: Started Java Spring Boot Application Service.
```

> ✅ Dịch vụ ở trạng thái **active (running)**, chạy dưới danh nghĩa user `java-runner`.

---

### 3.4. Kiểm tra nhật ký log

```bash
journalctl -u java-app.service --no-pager -n 20
```

**Kết quả mong đợi:**

```
Oct 06 14:30:00 hostname systemd[1]: Started Java Spring Boot Application Service.
```

---

### 3.5. Kiểm tra tiến trình chạy đúng user

```bash
ps aux | grep sleep
```

**Kết quả mong đợi:**

```
java-ru+ 31245  0.0  0.0   8080   520 ?        Ss   14:30   0:00 /usr/bin/sleep infinity
```

> Cột user hiển thị `java-ru+` (viết tắt của `java-runner`) — xác nhận dịch vụ chạy dưới quyền user giới hạn, **không phải root**.

---

### 3.6. Kiểm tra cơ chế auto-restart (on-failure)

Mô phỏng sự cố bằng cách kill tiến trình:

```bash
sudo kill -9 31245
```

Chờ 10 giây rồi kiểm tra lại:

```bash
sudo systemctl status java-app.service
```

**Kết quả mong đợi:** Dịch vụ đã **tự động restart** với PID mới:

```
● java-app.service - Java Spring Boot Application Service
     Active: active (running) since Tue 2026-10-06 14:31:10 UTC; 5s ago
   Main PID: 31300 (sleep)
```

---

### 3.7. Kích hoạt dịch vụ khởi động cùng hệ thống (enable)

```bash
sudo systemctl enable java-app.service
```

**Kết quả:**

```
Created symlink /etc/systemd/system/multi-user.target.wants/java-app.service → /etc/systemd/system/java-app.service.
```

---

## 4. Lưu ý khi triển khai thực tế

Khi thay thế bằng ứng dụng Spring Boot thực tế, cần thay đổi:

```ini
# Thay dòng ExecStart mô phỏng
ExecStart=/usr/bin/sleep infinity

# Bằng lệnh chạy Java thực tế
ExecStart=/usr/bin/java -jar /opt/java-app/app.jar

# Thêm biến môi trường nếu cần
Environment=JAVA_OPTS=-Xms256m -Xmx512m
Environment=SPRING_PROFILES_ACTIVE=production

# Thêm thư mục làm việc
WorkingDirectory=/opt/java-app
```

---

## 5. Tổng kết

| Hạng mục | Kết quả |
|---|---|
| Tạo user hệ thống `java-runner` (nologin) | ✅ Thành công |
| Tạo file `/etc/systemd/system/java-app.service` | ✅ Thành công |
| Dịch vụ chạy dưới user `java-runner` (non-root) | ✅ Xác nhận |
| Trạng thái `active (running)` | ✅ Thành công |
| Tự động restart sau 10s khi bị sập | ✅ Hoạt động đúng |
| Log hiển thị qua `journalctl` | ✅ Thành công |
| Enable khởi động cùng hệ thống | ✅ Thành công |
