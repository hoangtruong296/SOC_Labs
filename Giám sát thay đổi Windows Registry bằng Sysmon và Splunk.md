# Giám sát thay đổi Windows Registry bằng Sysmon và Splunk

# 1. Mục tiêu

- Mục tiêu của lab này là giám sát và phát hiện các thay đổi trái phép hoặc đáng ngờ đối với Windows Registry, đặc biệt là các hành vi liên quan đến cài đặt malware, cơ chế duy trì quyền truy cập (persistence), hoặc thay đổi cấu hình bảo mật. Sử dụng công cụ **Sysmon** và **Splunk** để giám sát các thay đổi này.

# 2. Kiến thức nền

- **Sysmon:** Là một công cụ thuộc bộ Sysinternals của Microsoft, hoạt động ở cấp độ kernel như một trình điều khiển và dịch vụ nền. Nó ghi lại các sự kiện chi tiết về hệ thống như tạo tiến trình, hoạt động mạng, thay đổi registry, và tải DLL — những hành vi thường bị phần mềm độc hại lợi dụng. Với file cấu hình phù hợp, Sysmon giúp các nhà phân tích bảo mật dễ dàng phát hiện các hành vi bất thường và theo dõi toàn bộ hành trình tấn công (attack chain).
- **Windows Registry:** Là một cơ sở dữ liệu phân cấp lưu trữ thông tin cấu hình và tùy chỉnh cho hệ điều hành Windows, thiết bị phần cứng, dịch vụ, tài khoản người dùng, và phần mềm cài đặt. Nhiều kỹ thuật tấn công và persistence (duy trì truy cập) lợi dụng registry để thiết lập backdoor hoặc tự động chạy mã độc. Việc theo dõi các thay đổi đáng ngờ trong registry là rất quan trọng trong giám sát an ninh hệ thống.
- **Splunk Universal Forwarder:** Splunk Universal Forwarder là một lightweight agent được cài trên máy Windows (hoặc các hệ điều hành khác) để thu thập log theo thời gian thực và gửi về máy chủ Splunk. Trong môi trường giám sát bảo mật, nó thường được cấu hình để gửi log của Windows Event Log, Sysmon hoặc các file log tùy chỉnh về Splunk nhằm phục vụ việc phân tích tập trung.
- **Splunk:** Là một nền tảng SIEM mạnh mẽ hỗ trợ thu thập, tìm kiếm, phân tích, cảnh báo và trực quan hóa dữ liệu log từ nhiều nguồn. Với khả năng truy vấn linh hoạt bằng SPL (Search Processing Language), Splunk giúp các analyst nhanh chóng phát hiện các hành vi đáng ngờ, như việc tạo Scheduled Task bất thường hoặc hoạt động lateral movement, từ các log được gửi về.

# 3. Sơ đồ mạng

<img width="749" height="399" alt="Windows Registry Sysmon+Splunk" src="https://github.com/user-attachments/assets/09c23db3-b6f2-49d2-aa0b-d18dacd45c90" />

# 4. Các bước tiến hành

## Bước 1: Chuẩn bị môi trường

- Tải và cài đặt máy ảo Ubuntu Server 22.04 VMware Player.
    - Ubuntu Server: https://releases.ubuntu.com/jammy/ubuntu-22.04.5-live-server-amd64.iso
- Thực hiện cài đặt, cấu hình Splunk Server trên Ubuntu Server và Splunk Universal Forwarder trên Windows theo hướng dẫn tại: [https://github.com/0xrajneesh/90-days-security-challenge/blob/main/Challenge%231/Lab Set up.md](https://github.com/0xrajneesh/90-days-security-challenge/blob/main/Challenge%232/Lab%20Set%20up%20.md)

## Bước 2: Cài đặt Sysmon trên máy Windows

1. **Tải Sysmon**
- Truy cập: https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon
- Tải file `.zip` về và giải nén (thường sẽ có `Sysmon.exe` và `Sysmon64.exe`)
2. **Tải file cấu hình Sysmon**
- Dùng mẫu cấu hình từ SwiftOnSecurity (rất phổ biến): https://github.com/SwiftOnSecurity/sysmon-config
    
    -> Tải file `sysmonconfig-export.xml`.
    
3. **Cài đặt Sysmon**
- Mở **Command Prompt với quyền Administrator**, điều hướng đến thư mục chứa Sysmon, rồi chạy:

```
sysmon64.exe -i sysmonconfig-export.xml
```

- Nếu thành công, Sysmon sẽ bắt đầu ghi lại các sự kiện vào Windows Event Log, mục **"Microsoft-Windows-Sysmon/Operational"**.
- **Kiểm tra:** Mở Event Viewer > Applications and Services Logs > Microsoft > Windows > Sysmon > Operational.

<img width="1498" height="448" alt="image" src="https://github.com/user-attachments/assets/945f8347-c78c-4f82-bb19-f54fbaf9839e" />

## Bước 3: Cấu hình Splunk Forwarder theo dõi và gửi log của Sysmon

1. **Mở file inputs.conf của Splunk Forwarder**

```
C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf
```

2. **Thêm nội dung sau**

```
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = false
index = sysmon_logs
sourcetype = XmlWinEventLog:Sysmon
renderXml = false
```

<img width="420" height="410" alt="image" src="https://github.com/user-attachments/assets/bf72b302-034b-49cc-be4e-1779f879f72d" />

3. **Khởi động lại Splunk Forwarder**

```
C:\WINDOWS\system32>"C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" restart
```

4. **Kiểm tra xem Splunk nhận được log của Sysmon**



- **Lưu ý:** Nếu Splunk không nhận được log của **Sysmon** mà xem thủ công log của **Sysmon** trên máy Windows vẫn có thì khả năng là do Splunk Forwarder **không chạy bằng Local System Account** nên không đủ quyền đọc log Event Viewer ở `Microsoft-Windows-Sysmon/Operational`
- **Cách khắc phục:**
    - Kiểm tra lại account đang chạy Splunk Forwarder bằng powershell:
    
    ```powershell
    Get-WmiObject win32_service | Where-Object { $_.Name -eq "SplunkForwarder" } | Select-Object StartName
    
    ```
    
    - Nếu không thấy `LocalSystem`, bạn cần sửa:
        - Nhấn `Win + R`, nhập: `services.msc`
        - Tìm dịch vụ **"SplunkForwarder"**
        - Chuột phải → `Properties`
        - Chuyển sang tab **Log On**
        - Chọn: Local System account
        - Khởi động lại Splunk Forwarder
