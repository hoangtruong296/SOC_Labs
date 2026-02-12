# Giám sát thay đổi trái phép trong thư mục nhạy cảm bằng Auditd và Splunk 
## 1. Mục tiêu
- Cấu hình hệ thống để **giám sát thời gian thực** các thay đổi trái phép (sửa, xóa, thay đổi thuộc tính) trong thư mục quan trọng như `/etc/`.
- Sử dụng **Auditd** để ghi nhận sự kiện hệ thống liên quan đến thay đổi file.
- Dùng **Splunk** để thu thập, phân tích và trực quan hóa log từ Auditd.
- Mô phỏng các hành vi tấn công (ví dụ: sửa file `/etc/passwd`, xóa file) và từ đó rèn luyện kỹ năng điều tra, phản ứng sự cố và bảo vệ hệ thống.

## 2. Kiến thức cơ bản

**Auditd** (Linux Auditing System) là một dịch vụ giúp ghi nhận các hành vi xảy ra trên hệ thống:

- **Ghi log hành động**: Mọi thao tác như mở file, sửa đổi, xóa… sẽ được ghi lại.
- **Tùy biến luật kiểm soát**: Cho phép cấu hình để giám sát những file hoặc hoạt động cụ thể.
- **Hỗ trợ chuẩn an toàn**: Giúp đạt được các yêu cầu tuân thủ bảo mật (compliance).
- **Tích hợp với các công cụ SIEM**: Như Splunk để phân tích và cảnh báo.

**Splunk** và **Splunk Universal Forwarder**:

- **Splunk** là nền tảng phân tích log tập trung.
- **Splunk Universal Forwarder**: Gửi log từ server tới Splunk Server.
- Cho phép viết truy vấn, tạo dashboard, và cảnh báo khi có dấu hiệu bất thường.

## 3. Sơ đồ mạng

<img width="710" height="407" alt="auditd+splunk lab drawio" src="https://github.com/user-attachments/assets/7f7f8340-d5cc-455e-b43b-b1d484d8fc21" />

## 4. Các bước thực hiện

### Bước 1: Chuẩn bị môi trường
- Tải và cài đặt máy ảo Ubuntu Server 22.04 và Kali Linux 2025.1a qua VMware Player.
    - Ubuntu Server: https://releases.ubuntu.com/jammy/ubuntu-22.04.5-live-server-amd64.iso
    - Kali Linux: https://cdimage.kali.org/kali-2025.1a/kali-linux-2025.1a-vmware-amd64.7z
- Thực hiện cài đặt, cấu hình Splunk Server trên Windows và Splunk Universal Forwarder trên Ubuntu Server theo hướng dẫn tại: [https://github.com/0xrajneesh/90-days-security-challenge/blob/main/Challenge%231/Lab Set up.md](https://github.com/0xrajneesh/90-days-security-challenge/blob/main/Challenge%231/Lab%20Set%20up.md)

### Bước 2: Cài đặt và cấu hình Auditd

1. Tải auditd và khởi động auditd
    
```bash
sudo apt update
sudo apt install auditd -y
systemctl start auditd
sudo systemctl enable auditd
```

2. Kiểm tra trạng thái auditd
    
```
sudo systemctl status auditd
```

<img width="852" height="263" alt="image" src="https://github.com/user-attachments/assets/503ba0fb-c79d-4c40-9fae-e8423d9ab19c" />

### Bước 3: Cấu hình luật giám sát cho Auditd

1. Thêm luật cho Auditd để giám sát thư mục nhạy cảm
   Mở file cấu hình luật của Auditd:
   
```
sudo nano /etc/audit/rules.d/audit.rules
```

    Thêm luật sau để giám sát thư mục /etc/ 
    
```
-w /etc/ -p wa -k file_integrity
```

- Ý nghĩa:
    - `-w`: theo dõi thư mục `/etc/`
    - `-p wa`: giám sát quyền ghi (`w`) và thay đổi thuộc tính (`a`)
    - `-k file_integrity`: keyword để gắn tag cho log

<img width="618" height="417" alt="image" src="https://github.com/user-attachments/assets/82ada69a-7971-49ba-863c-c2f966abb865" />

2. Khởi động lại dịch vụ auditd để áp dụng luật

```
sudo service auditd restart
sudo auditctl -l
```

<img width="513" height="73" alt="image" src="https://github.com/user-attachments/assets/e8b024fb-fe75-4685-ad27-6babf38a2177" />

### Bước 4: Cấu hình Splunk Universal Forwarder

1. Sửa file cấu hình `inputs.conf` để Splunk Forwarder theo dõi log của auditd và gửi tới Splunk Server

- Mở file `inputs.conf`
  
```
sudo nano /opt/splunkforwarder/etc/system/local/inputs.conf
```

- Thêm các dòng sau

```
[monitor:///var/log/audit/audit.log]
sourcetype = auditd
index = linux_file_integrity
```

<img width="717" height="472" alt="image" src="https://github.com/user-attachments/assets/796e11b6-7149-47e0-970f-b86063ac042f" />

- Khởi động lại Splunk Forwarder

```
sudo /opt/splunkforwarder/bin/splunk restart
```

<img width="855" height="578" alt="image" src="https://github.com/user-attachments/assets/8b3839b0-65f6-4a96-8c46-b4799087a884" />

2. Kiểm tra xem log đã được đẩy lên Splunk Server chưa

<img width="1831" height="672" alt="image" src="https://github.com/user-attachments/assets/a8d4eb38-6619-476d-9ecd-7920e8af291c" />

### Bước 5: Mô phỏng thay đổi trái phép trong thư mục `/etc/`

1. Chỉnh sửa file
- Chỉnh sửa file `/etc/passwd` để mô phỏng việc thay đổi trái phép

<img width="810" height="302" alt="image" src="https://github.com/user-attachments/assets/081966f3-c6c1-43aa-883b-d9904a55282c" />

2. Xóa file
- Thực hiện 1 file mới và xóa file đó trong thư mục `/etc/`

<img width="424" height="52" alt="image" src="https://github.com/user-attachments/assets/d6b80413-f059-40b7-8a81-7849ccc76bea" />

3. Truy xuất log với auditd
- Dùng `ausearch` dể xem logs cho những hành động thay đổi trái phép trên

```
sudo ausearch -k file_integrity
```

<img width="816" height="508" alt="image" src="https://github.com/user-attachments/assets/4c6b4e3e-ebbb-4694-96e5-a1067736f7a1" />


### Bước 6: Phân tích logs trên Splunk
1. Truy vấn tìm sự kiện integrity

```
index="linux_file_integrity" key="file_integrity"
```

<img width="1828" height="662" alt="image" src="https://github.com/user-attachments/assets/3954ea78-8c6c-4d15-8468-34b3545bb55e" />

2. Phân tích 1 sự kiện cụ thể
- Sự kiện vào thời gian (2/12/26 10:26:00.943 PM)

<img width="1828" height="590" alt="image" src="https://github.com/user-attachments/assets/16da0f7e-cd48-4814-8a1e-b299044a8dda" />

```
type=SYSCALL msg=audit(1770909960.943:457): arch=c000003e syscall=87 success=yes exit=0 a0=6401057ab700 a1=80100010 a2=1 a3=0 items=2 ppid=7421 pid=7422 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts1 ses=4 comm="nano" exe="/usr/bin/nano" subj=unconfined key="file_integrity"ARCH=x86_64 SYSCALL=unlink AUID="hoang" UID="root" GID="root" EUID="root" SUID="root" FSUID="root" EGID="root" SGID="root" FSGID="root"
```

```
type=CWD msg=audit(1770909960.943:457): cwd="/home/hoang"
type=PATH msg=audit(1770909960.943:457): item=0 name="/etc/" inode=917505 dev=08:02 mode=040755 ouid=0 ogid=0 rdev=00:00 nametype=PARENT cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0OUID="root" OGID="root"
type=PATH msg=audit(1770909960.943:457): item=1 name="/etc/.passwd.swp" inode=918363 dev=08:02 mode=0100644 ouid=0 ogid=0 rdev=00:00 nametype=DELETE cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0OUID="root" OGID="root"
```

```
type=PROCTITLE msg=audit(1770909960.943:457): proctitle=6E616E6F002F6574632F706173737764
```

#### Hành động chỉnh sửa file
**Thời gian sự kiện:**  `1770909960.943:457` tức `2/12/26 10:26:00.943 PM`
1. Tóm tắt sự kiện

   Tại thời điểm ghi nhận, một tiến trình do người dùng `hoang` khởi tạo đã thực thi trình soạn thảo `nano` để chỉnh sửa file hệ thống `/etc/passwd` với quyền root. Trong quá trình này, tiến trình đã xóa file swap tạm thời `/etc/.passwd.swp` bằng system call `unlink`. Hành động được thực hiện thành công (`success=yes`, `exit=0`).

3. Chi tiết kỹ thuật
- Lệnh thực thi: `nano /etc/passwd`
- Tên tiến trình: `nano`
- Đường dẫn thực thi: `/usr/bin/nano`
- PID: `7422` (cha: `7421`)
- Thư mục làm việc (CWD): `/home/hoang`
- File bị xóa: `/etc/.passwd.swp`
- Thư mục cha: `/etc/`
- syscall: `unlink` (`syscall=87`)
- Kiến trúc: x86_64
- Người dùng thật (AUID): `hoang` (auid=1000)
- Người thực thi: `root` (uid=0, euid=0, fsuid=0)
- TTY: `pts1`
- Audit rule key: `file_integrity`

3. Phân tích và đánh giá
- Mặc dù tiến trình chạy với quyền root, audit log cho thấy người khởi tạo phiên ban đầu là `hoang`, cho thấy nhiều khả năng hành động được thực hiện thông qua `sudo`.
- File bị xóa (`/etc/.passwd.swp`) là file swap tạm thời do nano tạo ra khi chỉnh sửa file `/etc/passwd`, không phải file cấu hình chính.
- Việc chỉnh sửa `/etc/passwd` là hành vi nhạy cảm, vì file này liên quan trực tiếp đến quản lý tài khoản người dùng và có thể bị lợi dụng để:
    - Tạo user trái phép
    - Thay đổi UID/GID
    - Cấp shell hoặc đặc quyền cao
- Không phát hiện lỗi hệ thống hoặc hành vi bị từ chối truy cập. Toàn bộ chuỗi hành động hoàn tất thành công.

4. Kiến nghị

- Xác minh nội dung thay đổi của /etc/passwd, đối chiếu với bản sao lưu hoặc baseline integrity nếu có.
- Kiểm tra log sudo và xác thực người dùng, ví dụ:
    - `/var/log/auth.log`
    - `ausearch -au hoang`
- Nếu không có thay đổi hợp lệ được phê duyệt:
    → Xem xét lại quyền sudo của user `hoang`
    → Áp dụng nguyên tắc least privilege
- Tăng cường giám sát auditd với các rule theo dõi:
    - `execve` khi truy cập `/etc/passwd`
    - `unlink`, `open`, `chmod`, `chown` tác động đến `/etc/*`
- Tích hợp log auditd vào SIEM (Wazuh/Splunk) để tạo cảnh báo real-time khi có chỉnh sửa file hệ thống quan trọng.

5. Kết luận

    Sự kiện ghi nhận cho thấy người dùng `hoang` đã sử dụng đặc quyền `root` để chỉnh sửa file hệ thống `/etc/passwd`, trong đó `nano` đã xóa file swap tạm thời `/etc/.passwd.swp` sau khi hoàn tất thao tác. Mặc dù     không có dấu hiệu tấn công rõ ràng, đây là hành vi có mức độ rủi ro cao do liên quan trực tiếp đến quản lý người dùng hệ thống. Khuyến nghị tiếp tục giám sát và xác minh tính hợp lệ của thay đổi.
