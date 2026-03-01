# Thiết lập hệ thống giám sát mạng bằng Suricata và Splunk để phát hiện các hành vi bất thường trong lưu lượng mạng

## 1. Mục tiêu

- Thiết lập một hệ thống giám sát mạng để phát hiện các mẫu lưu lượng bất thường, bao gồm cả việc xâm nhập dữ liệu và giao tiếp command-and-control (C2), bằng cách sử dụng Suricata với các quy tắc Emerging Threats (ET) và phân tích nhật ký bằng Splunk.
- Mô phỏng một cuộc tấn công dựa trên mạng và phản ứng với các cảnh báo được tạo bởi các quy tắc ET.

## 2. Kiến thức nền

- **Suricata:** Là một công cụ giám sát mạng và phát hiện xâm nhập (IDS) mạnh mẽ.  Nó phân tích lưu lượng mạng theo thời gian thực và có thể phát hiện các hoạt động độc hại.
- **Emerging Threats (ET) Rules:** Một bộ quy tắc nguồn mở được sử dụng với Suricata để phát hiện nhiều loại mối đe dọa, bao gồm phần mềm độc hại, giao tiếp C2 và xâm nhập dữ liệu.
- **Splunk Universal Forwarder:** Một thành phần của Splunk, được cài đặt trên các máy chủ để thu thập và chuyển tiếp nhật ký đến một Splunk Server trung tâm để phân tích.
- **Splunk:** Một nền tảng phần mềm mạnh mẽ để tìm kiếm, phân tích và trực quan hóa dữ liệu nhật ký.  Nó cho phép các nhà phân tích bảo mật xác định các xu hướng, điều tra sự cố và giám sát hoạt động của mạng.

## 3. Sơ đồ mạng

<img width="750" height="392" alt="Suricata+splunk" src="https://github.com/user-attachments/assets/93aff2d7-7bc9-4e42-bc34-aa9933492374" />

## 4. Các bước tiến hành

### Bước 1: Chuẩn bị môi trường

- Tải và cài đặt máy ảo Ubuntu Server 22.04 và Kali Linux 2025.1a qua VMware Player.
    - Ubuntu Server: https://releases.ubuntu.com/jammy/ubuntu-22.04.5-live-server-amd64.iso
    - Kali Linux: https://cdimage.kali.org/kali-2025.1a/kali-linux-2025.1a-vmware-amd64.7z
- Thực hiện cài đặt, cấu hình Splunk Server trên Windows và Splunk Universal Forwarder trên Ubuntu Server theo hướng dẫn tại: [https://github.com/0xrajneesh/90-days-security-challenge/blob/main/Challenge%231/Lab Set up.md](https://github.com/0xrajneesh/90-days-security-challenge/blob/main/Challenge%231/Lab%20Set%20up.md)

### Bước 2: Cài đặt và cấu hình Suricata

1. **Cài đặt Suricata**

```
sudo add-apt-repository ppa:oisf/suricata-stable
sudo apt-get update
sudo apt-get install suricata -y
```

<img width="1143" height="472" alt="image" src="https://github.com/user-attachments/assets/3b75c933-4b20-470c-9635-686da8579863" />

2. **Tải các quy tắc Emerging threats**

```
cd /tmp/ 
curl -LO https://rules.emergingthreats.net/open/suricata-7.0.3/emerging.rules.tar.gz
sudo tar -xvzf emerging.rules.tar.gz 
sudo mkdir /etc/suricata/rules 
sudo mv rules/*.rules /etc/suricata/rules/
sudo chmod 640 /etc/suricata/rules/*.rules
```

<img width="971" height="418" alt="image" src="https://github.com/user-attachments/assets/f73cbbdd-80e4-4453-bd4a-5c18d49c8124" />

3. **Cập nhật cấu hình Suricata**

- Mở tệp cấu hình Suricata

```
sudo nano /etc/suricata/suricata.yaml
```

- Cập nhật HOME_NET, EXTERNAL_NET, defaut-rule-path, rule-files

<img width="651" height="361" alt="image" src="https://github.com/user-attachments/assets/c75d35fd-90ab-4a76-b8f4-b16d66b0edf4" />

<img width="612" height="295" alt="image" src="https://github.com/user-attachments/assets/62aa1950-99c8-4073-8c8e-19c6abf4ca78" />


4. **Khởi động lại dịch vụ Suricata**

```
sudo systemctl restart suricata
sudo systemctl start suricata
sudo systemctl enable suricata
sudo systemctl status suricata
```

<img width="944" height="287" alt="image" src="https://github.com/user-attachments/assets/9a764aae-4494-43fc-ba8c-e40cd613e3a5" />

## Bước 3: Cấu hình Splunk Forwarder theo dõi log của Suricata

1. **Chỉnh sửa tệp cấu hình đầu vào của Splunk**

- Mở file cấu hình

```
sudo nano /opt/splunkforwarder/etc/system/local/inputs.conf
```

- Thêm nội dung sau để giám sát log của Suricata

```
[monitor:///var/log/suricata/eve.json]
sourcetype = suricata
index = network_security_logs
```

2. **Lưu và khởi động lại Splunk Forwarder**

```
sudo /opt/splunkforwarder/bin/splunk restart
```

3. **Xác minh chuyển tiếp log thành công**

- Ở giao diện quản lý Splunk, tạo index mới để nhận log từ Suricata

<img width="809" height="570" alt="image" src="https://github.com/user-attachments/assets/6e42928d-259c-4e64-8aca-2546b5e2620c" />

