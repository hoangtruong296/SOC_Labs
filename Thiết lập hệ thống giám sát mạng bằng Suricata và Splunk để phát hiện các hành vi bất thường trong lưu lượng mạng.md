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

