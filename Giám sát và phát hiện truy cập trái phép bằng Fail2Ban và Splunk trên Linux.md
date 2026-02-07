## 1. Mục tiêu
- Thiết lập môi trường theo dõi và ghi nhận các lần thử truy cập trái phép vào dịch vụ SSH.
- Cấu hình **Fail2Ban** để phát hiện và chặn các địa chỉ IP độc hại.
- Sử dụng **Splunk** để thu thập và phân tích log từ Fail2Ban nhằm phát hiện hành vi tấn công.

## 2. Kiến thức cơ bản

- **Splunk**: Nền tảng mạnh mẽ dùng để thu thập, phân tích và trực quan hóa log hệ thống. Splunk giúp phát hiện hành vi bất thường như thực thi tiến trình lạ, kết nối ngược (reverse shell), hay các cuộc tấn công mạng thông qua phân tích log theo thời gian thực.
- **Splunk Universal Forwarder**: Là agent nhẹ được cài đặt trên máy **Ubuntu Server** (nạn nhân), có nhiệm vụ chuyển tiếp log từ hệ thống (ví dụ: log tiến trình, log SSH) về **Splunk Server** (máy phân tích, thường là Windows hoặc Linux).
- **Fail2Ban**: Công cụ bảo mật trên Linux dùng để giám sát log và tự động **chặn các địa chỉ IP** có dấu hiệu tấn công (như đăng nhập sai nhiều lần qua SSH). Fail2Ban giúp giảm thiểu rủi ro từ các cuộc tấn công brute-force.
- **SSH (Secure Shell)**: Giao thức phổ biến để truy cập máy chủ từ xa. Do tính phổ biến và quyền truy cập cao, SSH thường là mục tiêu của các **cuộc tấn công brute-force**, nơi kẻ tấn công thử nhiều tổ hợp tên đăng nhập và mật khẩu để xâm nhập hệ thống.
- **Hydra**: Là công cụ tấn công brute-force nổi tiếng, hỗ trợ nhiều giao thức như SSH, FTP, HTTP, Telnet,… Hydra cho phép kẻ tấn công tự động thử hàng loạt mật khẩu trên một tài khoản nhằm chiếm quyền truy cập.
- **Tấn công Brute-force**: Là hình thức tấn công dò mật khẩu bằng cách thử tất cả các kết hợp có thể cho đến khi tìm ra thông tin đúng. Đây là kỹ thuật đơn giản nhưng vẫn rất hiệu quả nếu hệ thống không có cơ chế khóa tài khoản hoặc giám sát truy cập.

## 3. Sơ đồ mạng

<img width="720" height="392" alt="Fail2Ban-Splunk drawio" src="https://github.com/user-attachments/assets/10d0e6bd-ef16-4ff8-bd5b-3bfd93b3397d" />

## 4. Các bước thực hiện

### Bước 1: Chuẩn bị môi trường
- Tải và cài đặt máy ảo Ubuntu và Kali Linux tại đây:

  https://ubuntu.com/download/desktop

  https://www.kali.org/get-kali/#kali-virtual-machines
- Cài đặt và cấu hình Splunk server và Splunk Forwarder theo hướng dẫn dưới đây:

  https://github.com/0xrajneesh/90-days-security-challenge/blob/main/Challenge%231/Lab%20Set%20up.md
### Bước 2: Cài đặt và cấu hình Fail2Ban trên máy Ubuntu server

1. Cài đặt Fail2Ban
```
sudo apt update
sudo apt install fail2ban -y
```
2. Cấu hình Fail2Ban cho dịch vụ SSH
```
sudo nano /etc/fail2ban/jail.conf
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

3. Add monitor để Forwarder theo dõi file log của Fail2Ban và khởi động lại Fail2Ban
```
/opt/splunkforwarder/bin/splunk add monitor /var/log/fail2ban.log
sudo systemctl restart fail2ban
```
4. Kiểm tra trạng thái hoạt động của Fail2Ban
```
sudo fail2ban-client status
```

<img width="441" height="95" alt="image" src="https://github.com/user-attachments/assets/689c1b7c-b49a-4bb1-b2dd-57f1400c655e" />

### Bước 3: Cấu hình Splunk Server và Splunk Forwarder

1. **Trên Splunk Server (Ubuntu)**
- Tạo index mới: `fail2ban_logs` (Settings → Indexes → New Index).

<img width="795" height="648" alt="image" src="https://github.com/user-attachments/assets/124ce464-6d1b-495f-af12-9ce84d4ca6be" />

2. **Trên Splunk Universal Forwarder (Ubuntu Server)**
- Thiết lập Splunk Forwarder giám sát log của Fail2Ban:
```bash
/opt/splunkforwarder/bin/splunk add monitor /var/log/fail2ban.log
sudo systemctl restart fail2ban
```
- Cấu hình Splunk Forwarder gửi log của Fail2Ban đến Splunk Server:
  Mở file inputs.conf:
  
```bash
sudo nano /opt/splunkforwarder/etc/system/local/inputs.conf
```
  Thêm các dòng sau:
  
```bash
[monitor:///var/log/fail2ban.log]
sourcetype = fail2ban_logs
index = fail2ban_logs
```

<img width="772" height="377" alt="image" src="https://github.com/user-attachments/assets/46dfcad6-8ca5-49c7-a813-873dc4eb368f" />

  Sau đó khởi động lại Splunk Forwarder:
  
```
sudo /opt/splunkforwarder/bin/splunk restart
```

## **Bước 4: Mô phỏng tấn công brute-force SSH từ Kali Linux**

1. **Kiểm tra thử hệ thống log hoạt động ổn định:**
- Sau khi cấu hình Splunk Server, Splunk Forwarder thì trên máy Ubuntu Server thử đăng nhập SSH với mật khẩu sai để tạo log xem mọi thứ đã hoạt động ổn chưa

  <img width="922" height="393" alt="image" src="https://github.com/user-attachments/assets/c4868c78-4a2c-4f29-8c8c-b6359a6bb66b" />

  <img width="1457" height="653" alt="image" src="https://github.com/user-attachments/assets/cfd6f5cd-2bd4-4ec2-842f-fc5dd51f38ce" />

- Hình trên đã cho thấy rằng Splunk Forwarder đã hoạt động và tạo log, Splunk Forwarder cũng đã theo dõi log của Fail2Ban từ file fail2ban.log sau đó gửi log đến Splunk Server
- Tiếp theo, sử dụng máy Kali Linux để mô phỏng tấn công brute-force

2. **Tạo danh sách mật khẩu brute-force (hoặc sử dụng wordlist có sẵn)**

  <img width="286" height="214" alt="image" src="https://github.com/user-attachments/assets/adafdc0d-e335-4af0-a48c-0c2b9245fcfe" />

3. **Chạy Hydra brute-force SSH**

```bash
hydra -l hoang -P passwd.txt 192.168.0.20 ssh
```

<img width="1029" height="220" alt="image" src="https://github.com/user-attachments/assets/af84ade0-f793-4ddc-859d-47cc5fece8af" />

4. **Kiểm tra Fail2Ban trên máy Ubuntu Server đã block IP tấn công chưa**

- File fail2ban.log đã ghi lại nỗ lực truy cập thất bại nhiều lần từ Kali(attacker) với địa chỉ IP 192.168.0.30
  
  <img width="927" height="242" alt="image" src="https://github.com/user-attachments/assets/315e70d7-8b14-4542-abe8-91263143b7b8" />

- Kiểm tra Fail2Ban trên Ubuntu Server, ta thấy rằng IP của Kali đã bị block

  <img width="857" height="218" alt="image" src="https://github.com/user-attachments/assets/0a4e7633-66c8-4d07-90ad-33bf5a63b536" />

- Splunk Server cũng đã nhận được log và hiển thị log về các sự kiện trên

  <img width="1244" height="623" alt="image" src="https://github.com/user-attachments/assets/066afb07-ca9a-49ae-adfb-4ef42573e5c7" />

## **Bước 5: Phân tích log trên Splunk**

- Truy cập giao diện Splunk Server qua trình duyệt: http://127.0.0.1:8000 và Search → chọn index fail2ban_logs và thực hiện truy vấn log

- Có thể thấy Splunk Server cũng đã ghi nhận và hiển thị log về các sự kiện trên

  <img width="1244" height="623" alt="image" src="https://github.com/user-attachments/assets/066afb07-ca9a-49ae-adfb-4ef42573e5c7" />

  ### 1. Thông tin các sự kiện

- **Thời gian:** 2026-02-07, khoảng 15:44:47
- **Dịch vụ bị tấn công:** SSH (`sshd`)
- **Địa chỉ IP tấn công:** `192.168.0.30`
- **Hành động:**
    - Fail2ban phát hiện nhiều lần truy cập thất bại từ IP trên.
    - Đã thực hiện chặn (`ban`) IP `192.168.0.30` vào lúc 15:44:47.
- **Cơ chế:** Fail2ban theo dõi file log `/var/log/auth.log` để phát hiện các hành vi đăng nhập sai và tự động chặn IP tấn công.

---

### 2. Đánh giá

- IP `192.168.0.30` thực hiện tấn công brute force vào dịch vụ SSH.
- Hệ thống đã phản ứng kịp thời bằng cách khóa IP, giảm nguy cơ bị xâm nhập trái phép.
- Fail2ban hoạt động hiệu quả, giúp bảo vệ dịch vụ SSH trước các tấn công phổ biến.

---

### 3. Khuyến nghị

- Tiếp tục giám sát log fail2ban và hệ thống để phát hiện các IP tấn công mới.
- Áp dụng các biện pháp bổ sung:
    - Sử dụng xác thực đa yếu tố (MFA) cho SSH.
    - Giới hạn truy cập SSH chỉ từ các IP tin cậy.
    - Thường xuyên cập nhật phần mềm và kiểm tra cấu hình bảo mật SSH.

## Để unban IP, sử dụng lệnh:

```bash
sudo fail2ban-client set sshd unbanip <banned-ip>
```

