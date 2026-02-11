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
