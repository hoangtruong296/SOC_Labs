## 1. Mục tiêu
- Thiết lập môi trường theo dõi và ghi nhận các lần thử truy cập trái phép vào dịch vụ SSH.
- Cấu hình **Fail2Ban** để phát hiện và chặn các địa chỉ IP độc hại.
- Sử dụng **Splunk** để thu thập và phân tích log từ Fail2Ban nhằm phát hiện hành vi tấn công.

## 2. Kiến thức nền tảng

- **Splunk**: Nền tảng mạnh mẽ dùng để thu thập, phân tích và trực quan hóa log hệ thống. Splunk giúp phát hiện hành vi bất thường như thực thi tiến trình lạ, kết nối ngược (reverse shell), hay các cuộc tấn công mạng thông qua phân tích log theo thời gian thực.
- **Splunk Universal Forwarder**: Là agent nhẹ được cài đặt trên máy **Ubuntu Server** (nạn nhân), có nhiệm vụ chuyển tiếp log từ hệ thống (ví dụ: log tiến trình, log SSH) về **Splunk Server** (máy phân tích, thường là Windows hoặc Linux).
- **Fail2Ban**: Công cụ bảo mật trên Linux dùng để giám sát log và tự động **chặn các địa chỉ IP** có dấu hiệu tấn công (như đăng nhập sai nhiều lần qua SSH). Fail2Ban giúp giảm thiểu rủi ro từ các cuộc tấn công brute-force.
- **SSH (Secure Shell)**: Giao thức phổ biến để truy cập máy chủ từ xa. Do tính phổ biến và quyền truy cập cao, SSH thường là mục tiêu của các **cuộc tấn công brute-force**, nơi kẻ tấn công thử nhiều tổ hợp tên đăng nhập và mật khẩu để xâm nhập hệ thống.
- **Hydra**: Là công cụ tấn công brute-force nổi tiếng, hỗ trợ nhiều giao thức như SSH, FTP, HTTP, Telnet,… Hydra cho phép kẻ tấn công tự động thử hàng loạt mật khẩu trên một tài khoản nhằm chiếm quyền truy cập.
- **Tấn công Brute-force**: Là hình thức tấn công dò mật khẩu bằng cách thử tất cả các kết hợp có thể cho đến khi tìm ra thông tin đúng. Đây là kỹ thuật đơn giản nhưng vẫn rất hiệu quả nếu hệ thống không có cơ chế khóa tài khoản hoặc giám sát truy cập.

## 3. Sơ đồ mạng

<img width="720" height="392" alt="Fail2Ban-Splunk drawio" src="https://github.com/user-attachments/assets/549be161-184b-41e8-9f17-882d32877ef4" />

## 4. Các bước thực hiện

### Bước 1: Chuẩn bị môi trường
- Tải và cài đặt máy ảo Ubuntu và Kali Linux tại đây:
  https://ubuntu.com/download/desktop
  https://www.kali.org/get-kali/#kali-virtual-machines
- Cài đặt và cấu hình Splunk server và Splunk forwarder theo hướng dẫn dưới đây:
  https://github.com/0xrajneesh/90-days-security-challenge/blob/main/Challenge%231/Lab%20Set%20up.md
### Bước 2: Cài đặt và cấu hình Fail2Ban trên máy Ubuntu server

1. Cài đặt Fail2Ban
```
sudo apt update
sudo apt install fail2ban -y
```
2. Cấu hình Fail2Ban cho dịch vụ SSH
```
sudo nano /etc/fail2ban/jail.local
```
  Thêm các dòng dưới để cấu hình bảo vệ dịch vụ SSH:
```
[sshd]
enabled = true
port = ssh
logpath = /var/log/auth.log
maxretry = 5
bantime = 600
findtime = 600
```

3. Add monitor để forwarder theo dõi file log và restart Fail2Ban
```
/opt/splunkforwarder/bin/splunk add monitor /var/log/fail2ban.log
sudo systemctl restart fail2ban
```
4. Kiểm tra trạng thái hoạt động của Fail2Ban
```
sudo fail2ban-client status
```
### Bước 3: Cấu hình Splunk Server và Splunk Forwarder

1. **Trên Splunk Server (Windows)**
- Tạo index mới: `fail2ban_logs` (Settings → Indexes → New Index).
2. **Trên Splunk Universal Forwarder (Ubuntu Server)**
- Thiết lập Splunk Forwarder giám sát log của Fail2Ban:
```bash
/opt/splunkforwarder/bin/splunk add monitor /var/log/fail2ban.log
sudo systemctl restart fail2ban
```
