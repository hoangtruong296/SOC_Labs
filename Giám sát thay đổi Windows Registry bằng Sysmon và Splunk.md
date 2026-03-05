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

<img width="1431" height="697" alt="image" src="https://github.com/user-attachments/assets/9a7ffacf-9255-4f29-81dc-05eea1fd8498" />

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

## Bước 4: Mô phỏng các thay đổi độc hại với Windows Registry

Sử dụng powershell để thực hiện các mô phỏng.

1. **Thêm một mục persistence giả**

```powershell
New-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" -Name "MalwareTest" -Value "C:\malwaretest.exe"
```

<img width="1183" height="161" alt="image" src="https://github.com/user-attachments/assets/8fb96b5c-c8f7-4018-aee9-4a834f77c9fe" />

- **Ý nghĩa:**
    - **Tạo một giá trị (registry value)** có tên là `MalwareTest` tại key `Run` trong nhánh `HKCU` (người dùng hiện tại).
    - Gán giá trị là `"C:\malwaretest.exe"`.
- **Tác dụng:**
    - Đây là **một kỹ thuật persistence**: mỗi khi người dùng **log in**, Windows sẽ **tự động chạy file `C:\malwaretest.exe`**.
    - `HKCU:\Software\Microsoft\Windows\CurrentVersion\Run` là một **Run Key phổ biến** dùng để auto-start chương trình khi đăng nhập.
2. **Sửa đổi một cài đặt bảo mật quan trọng**

```powershell
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters" -Name "DisableIPSourceRouting" -Value 1
```

<img width="1171" height="40" alt="image" src="https://github.com/user-attachments/assets/8abb745b-53b9-451a-b6a7-ce3e4ef464c0" />

- **Ý nghĩa:**
    - Thay đổi giá trị `DisableIPSourceRouting` thành `1` trong key `Tcpip Parameters` của `HKLM` (toàn bộ hệ thống).
- **Tác dụng:**
    - **Vô hiệu hóa IP source routing**, một kỹ thuật có thể bị lợi dụng để thực hiện **spoofing attack** hoặc **network reconnaissance**.
    - Đây là **một thay đổi cấu hình an ninh mạng ở cấp hệ thống**.
3. **Xóa một khóa registry**

```
Remove-Item -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run\MalwareSimulation"
```

<img width="927" height="35" alt="image" src="https://github.com/user-attachments/assets/4d516db2-d957-4ad8-8879-64904716a1e2" />

- **Ý nghĩa:**
    - Xóa giá trị `MalwareSimulation` trong `Run Key` của người dùng hiện tại.
- **Tác dụng:**
    - Nếu có một malware đã tạo key này để chạy mỗi khi user đăng nhập, thì lệnh này **xóa bỏ cơ chế persistence đó**.
    - **Defender hoặc analyst** để **gỡ mã độc**
    - Hoặc ngược lại, **malware tự xóa dấu vết** để tránh phát hiện (rare)

## Bước 5: Phân tích log trên Splunk

1. **Thêm persistence**

```
03/05/2026 01:56:19.917 PM
LogName=Microsoft-Windows-Sysmon/Operational
EventCode=13
EventType=4
ComputerName=truonghuyhoang-b22dcat130-VPNClient
User=NOT_TRANSLATED
Sid=S-1-5-18
SidType=0
SourceName=Microsoft-Windows-Sysmon
Type=Information
RecordNumber=42641
Keywords=None
TaskCategory=Registry value set (rule: RegistryEvent)
OpCode=Info
Message=Registry value set:
RuleName: technique_id=T1547.001,technique_name=Registry Run Keys / Start Folder
EventType: SetValue
UtcTime: 2026-03-05 06:56:19.908
ProcessGuid: {3478df39-281d-69a9-9505-000000003800}
ProcessId: 6408
Image: C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
TargetObject: HKU\S-1-5-21-3253799767-270482447-3232193585-1001\SOFTWARE\Microsoft\Windows\CurrentVersion\Run\MalwareTest
Details: C:\malwaretest.exe
User: TRUONGHUYHOANG-\Hoang

    host = truonghuyhoang-b22dcat130-VPNClient
    source = WinEventLog:Microsoft-Windows-Sysmon/Operational
    sourcetype = XmlWinEventLog:Sysmon
```

**Sự kiện thiết lập Registry Run Key – Dấu hiệu tạo Persistence**

📌 Thời gian (UTC): 2026-03-05 06:56:19

📌 Thời gian (giờ địa phương – UTC+7): ~13:56:19

📌 Hostname: truonghuyhoang-b22dcat130-VPNClient

📌 User thực hiện: TRUONGHUYHOANG-\Hoang
(SID: S-1-5-21-3253799767-270482447-3232193585-1001)

📌 Process thực thi

Image: `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe`

Process ID: 6408

Process GUID: {3478df39-281d-69a9-9505-000000003800}

📌 Hành vi

Tiến trình PowerShell đã thực hiện thao tác SetValue trên Registry tại vị trí:

```
HKU\S-1-5-21-3253799767-270482447-3232193585-1001\SOFTWARE\Microsoft\Windows\CurrentVersion\Run\MalwareTest
```

Đây là hành vi thiết lập một Registry Run Key, thường được sử dụng để duy trì persistence.

📌 Giá trị thiết lập

```
C:\malwaretest.exe
```

Điều này đồng nghĩa với việc file malwaretest.exe sẽ tự động được khởi chạy mỗi khi người dùng đăng nhập.

📌 Phân tích kỹ thuật

EventCode: 13

TaskCategory: Registry value set

EventType: SetValue

Theo cơ chế của Windows: `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`

Tương đương với: `HKU\<SID>\Software\Microsoft\Windows\CurrentVersion\Run`

Do đó, persistence này áp dụng cho user cụ thể (Hoang), không phải toàn hệ thống.

Sysmon rule đã gắn nhãn:

```
technique_id=T1547.001
technique_name=Registry Run Keys / Start Folder
```

📌 Mapping MITRE ATT&CK

- Technique ID: T1547.001
- Technique Name: Boot or Logon Autostart Execution – Registry Run Keys / Startup Folder
- Tactic: Persistence

📌 Mức độ nghi vấn: Trung bình đến Cao, phụ thuộc ngữ cảnh:

PowerShell chỉnh sửa Run Key là hành vi thường thấy trong:

- Malware
- Script persistence
- Red team simulation

📌 Đề xuất điều tra (theo góc nhìn SOC thực tế)

- Kiểm tra sự tồn tại thực tế của file: `C:\malwaretest.exe`
- Kiểm tra hash file (SHA256)
- Kiểm tra parent process của PowerShell
- Xác định có command-line bất thường hay không
- Kiểm tra có hành vi:
    - Network connection
    - Process injection
    - Additional registry changes
Nếu không nằm trong change management hợp lệ → đáng điều tra

2. **Chỉnh sửa Registry**
```
03/05/2026 01:57:57.848 PM
LogName=Microsoft-Windows-Sysmon/Operational
EventCode=13
EventType=4
ComputerName=truonghuyhoang-b22dcat130-VPNClient
User=NOT_TRANSLATED
Sid=S-1-5-18
SidType=0
SourceName=Microsoft-Windows-Sysmon
Type=Information
RecordNumber=42645
Keywords=None
TaskCategory=Registry value set (rule: RegistryEvent)
OpCode=Info
Message=Registry value set:
RuleName: -
EventType: SetValue
UtcTime: 2026-03-05 06:57:57.846
ProcessGuid: {3478df39-281d-69a9-9505-000000003800}
ProcessId: 6408
Image: C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
TargetObject: HKLM\System\CurrentControlSet\Services\Tcpip\Parameters\DisableIPSourceRouting
Details: DWORD (0x00000001)
User: TRUONGHUYHOANG-\Hoang

    host = truonghuyhoang-b22dcat130-VPNClient
    source = WinEventLog:Microsoft-Windows-Sysmon/Operational
    sourcetype = XmlWinEventLog:Sysmon
```

3. **Xóa một khóa registry**

```
03/05/2026 02:16:51.611 PM
LogName=Microsoft-Windows-Sysmon/Operational
EventCode=12
EventType=4
ComputerName=truonghuyhoang-b22dcat130-VPNClient
User=NOT_TRANSLATED
Sid=S-1-5-18
SidType=0
SourceName=Microsoft-Windows-Sysmon
Type=Information
RecordNumber=42667
Keywords=None
TaskCategory=Registry object added or deleted (rule: RegistryEvent)
OpCode=Info
Message=Registry object added or deleted:
RuleName: technique_id=T1547.001,technique_name=Registry Run Keys / Start Folder
EventType: DeleteKey
UtcTime: 2026-03-05 07:16:51.610
ProcessGuid: {3478df39-281d-69a9-9505-000000003800}
ProcessId: 6408
Image: C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
TargetObject: HKU\S-1-5-21-3253799767-270482447-3232193585-1001\SOFTWARE\Microsoft\Windows\CurrentVersion\Run\MalwareSimulation
User: TRUONGHUYHOANG-\Hoang

    host = truonghuyhoang-b22dcat130-VPNClient
    source = WinEventLog:Microsoft-Windows-Sysmon/Operational
    sourcetype = XmlWinEventLog:Sysmon
```
