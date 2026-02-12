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
    
```
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
- Tạo 1 file mới và xóa file đó trong thư mục `/etc/`

<img width="424" height="52" alt="image" src="https://github.com/user-attachments/assets/d6b80413-f059-40b7-8a81-7849ccc76bea" />

