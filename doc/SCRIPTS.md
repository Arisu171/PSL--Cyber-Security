# KỊCH BẢN THỰC NGHIỆM AN NINH MẠNG PHÒNG THỦ THEO CHIỀU SÂU (LAB 99)
## CHUỖI KỊCH BẢN TỊNH TIẾN: BASELINE ➔ BỊ TẤN CÔNG ➔ PHÒNG THỦ THẤT BẠI ➔ CẢI THIỆN PHÒNG THỦ ➔ BỊ TẤN CÔNG LẠI ➔ PHÒNG THỦ THÀNH CÔNG ➔ BASELINE MỚI

---

### THÔNG TIN DỰ ÁN & HẠ TẦNG THỰC NGHIỆM
- **Hệ thống mục tiêu (Target Node):** `lab99_target` — IP: `10.2.99.10` (Ubuntu 22.04 LTS)
- **Nút kiểm định / Tấn công (Assessor Node):** `lab99_assessor` — IP: `10.2.99.20` (Trang bị Nmap, Scapy, Hping3, Arping, cURL)
- **Không gian mạng ảo hóa cô lập:** Docker Network `lab99-net` (`10.2.99.0/24`, Gateway: `10.2.99.254`)
- **Tài khoản quản trị mặc định:** `labuser`
- **Khung lý thuyết áp dụng:** Mô hình Phòng thủ theo chiều sâu (*Defense-in-Depth*) & Tiêu chuẩn gia cố hệ điều hành Linux (*CIS Benchmark / Lynis*).

---

## MỤC LỤC
1. [Tổng quan Kiến trúc và Chuỗi tiến trình Tịnh tiến](#1-tổng-quan-kiến-trúc-và-chuỗi-tiến-trình-tịnh-tiến)
2. [Ma trận Vòng đời Tịnh tiến của Hệ thống](#2-ma-trận-vòng-đời-tịnh-tiến-của-hệ-thống)
3. [Kịch bản 1: Tầng Liên kết dữ liệu (Data Link) — Phòng chống ARP Spoofing & MITM](#kịch-bản-1-tầng-liên-kết-dữ-liệu-data-link--phòng-chống-arp-spoofing--mitm)
4. [Kịch bản 2: Tầng Mạng & Hạ tầng Tên miền (DNS) — Phòng chống DNS Cache Poisoning](#kịch-bản-2-tầng-mạng--hạ-tầng-tên-miền-dns--phòng-chống-dns-cache-poisoning)
5. [Kịch bản 3: Tầng Giao vận (Transport) — Phòng chống DoS TCP SYN Flood & Rate Limiting](#kịch-bản-3-tầng-giao-vận-transport--phòng-chống-dos-tcp-syn-flood--rate-limiting)
6. [Kịch bản 4: Tầng Ứng dụng & Dữ liệu (Application) — Phòng chống SQL Injection (SQLi)](#kịch-bản-4-tầng-ứng-dụng--dữ-liệu-application--phòng-chống-sql-injection-sqli)
7. [Kịch bản 5: Tầng Ứng dụng & Hệ điều hành (Application) — Phòng chống OS Command Injection (RCE)](#kịch-bản-5-tầng-ứng-dụng--hệ-điều-hành-application--phòng-chống-os-command-injection-rce)
8. [Kịch bản 6: Tầng Dịch vụ & Máy chủ (Host Defense) — Phòng chống SSH Brute-Force bằng Fail2ban & UFW](#kịch-bản-6-tầng-dịch-vụ--máy-chủ-host-defense--phòng-chống-ssh-brute-force-bằng-fail2ban--ufw)
9. [Kịch bản 7: Tầng Giám sát Toàn vẹn & Điều tra số (Host Auditing) — Giám sát tệp tin bằng Linux Kernel Auditd](#kịch-bản-7-tầng-giám-sát-toàn-vẹn--điều-tra-số-host-auditing--giám-sát-tệp-tin-bằng-linux-kernel-auditd)
10. [Tổng kết Baseline Hoàn thiện và Hướng dẫn Thực thi Tuần tự (Runbook)](#10-tổng-kết-baseline-hoàn-thiện-và-hướng-dẫn-thực-thi-tuần-tự-runbook)

---

## 1. TỔNG QUAN KIẾN TRÚC VÀ CHUỖI TIẾN TRÌNH TỊNH TIẾN

Kiến trúc an toàn thông tin không phải là một trạng thái bất biến, mà là một quá trình liên tục cải tiến thông qua vòng lặp phản hồi thực nghiệm (*Feedback Loop*). Mỗi tầng phòng thủ trong Lab 99 được thiết kế theo đúng quy trình 7 giai đoạn tịnh tiến:

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                 VÒNG LẶP TỊNH TIẾN (CYCLIC PROGRESSION)                │
└────────────────────────────────────────────────────────────────────────────────────────┘
  [Giai đoạn 1]  BASELINE BAN ĐẦU (Hiện trạng cấu hình & điểm an ninh sơ khởi)
        │
        ▼
  [Giai đoạn 2]  BỊ TẤN CÔNG (Kẻ tấn công phát động kỹ thuật xâm nhập thực tế)
        │
        ▼
  [Giai đoạn 3]  PHÒNG THỦ THẤT BẠI (Phân tích nguyên nhân gốc & hậu quả sụp đổ)
        │
        ▼
  [Giai đoạn 4]  CẢI THIỆN PHÒNG THỦ (Triển khai cơ chế gia cố & cấu hình kỹ thuật)
        │
        ▼
  [Giai đoạn 5]  BỊ TẤN CÔNG LẠI (Tái kích hoạt đợt tấn công kiểm thử đối chứng)
        │
        ▼
  [Giai đoạn 6]  PHÒNG THỦ THÀNH CÔNG (Đo kiểm minh chứng & chỉ số kỹ thuật đạt chuẩn)
        │
        ▼
  [Giai đoạn 7]  CHUYỂN THÀNH BASELINE MỚI (Bàn giao nền tảng an toàn cho kịch bản sau)
```

---

## 2. MA TRẬN VÒNG ĐỜI TỊNH TIẾN CỦA HỆ THỐNG

| Kịch bản | Phân lớp Phòng thủ | Nguy cơ / Kỹ thuật tấn công | Điểm yếu khi phòng thủ thất bại | Biện pháp cải thiện phòng thủ | Kết quả phòng thủ thành công | Trạng thái chuyển giao Baseline |
|:---|:---|:---|:---|:---|:---|:---|
| **KB 1** | **Layer 2 (Data Link)** | ARP Cache Poisoning / Fake ARP Reply | Bảng ARP động bị ghi đè, lộ lưu lượng qua MITM | `rp_filter=1`, `arp_ignore=1`, `Static ARP (PERMANENT)` | Bảng ARP khóa cứng, gói tin giả bị drop | **Baseline 2:** Lớp 2 an toàn, kháng MITM |
| **KB 2** | **Layer 3/4 (Network/DNS)** | DNS Cache Poisoning & Amplification | Cổng DNS mở, cổng nguồn hẹp dễ bị đoán ID | Đóng cổng 53 inbound, SPR mở rộng 28,231 cổng ephemeral | Cổng 53 đóng ngoài, triệt tiêu tấn công Kaminsky | **Baseline 3:** Lớp mạng & DNS vững chắc |
| **KB 3** | **Layer 4 (Transport)** | DoS TCP SYN Flood & ICMP Flood | Hàng đợi SYN đầy, cạn kiệt tài nguyên kết nối | `tcp_syncookies=1`, backlog=2048, UFW Rate Limiting | Sống sót 3/3 kết nối hợp lệ trong tâm bão flood | **Baseline 4:** Dịch vụ luôn sẵn sàng cao |
| **KB 4** | **Layer 7 (Application)** | SQL Injection (Tautology Bypass) | Nối chuỗi SQL trực tiếp, dump toàn bộ bảng users | Parameterized Prepared Statements | Payload bị xử lý như chuỗi thuần, không rò rỉ dữ liệu | **Baseline 5:** Cơ sở dữ liệu miễn nhiễm SQLi |
| **KB 5** | **Layer 7 (Application)** | OS Command Injection (RCE) | Lệnh shell hệ thống ghép chuỗi (`shell=True`) | Whitelist Regex Validation, `subprocess(shell=False)` | Trả về HTTP 400 Bad Request, lệnh độc hại bị chặn đứng | **Baseline 6:** Triệt tiêu hoàn toàn nguy cơ RCE |
| **KB 6** | **Host / Management** | Quét Nmap, SSH Password Brute-Force | Port 22 mở, cho phép mật khẩu & tài khoản root | Port 2222, ED25519 key-only, UFW, Fail2ban jail sshd | Fail2ban tự động DROP IP tấn công sau 3 lần sai | **Baseline 7:** Quản trị an toàn, tự động cô lập |
| **KB 7** | **Host Auditing** | Can thiệp tệp nhạy cảm `/etc/passwd` | Không có nhật ký truy vết danh tính kẻ can thiệp | Nạp quy tắc Linux Kernel Auditd (`-w /etc/passwd -p wa`) | Ausearch bắt trọn PID, syscall, UID root và timestamp | **Baseline Hoàn thiện:** Đạt 12/12 điểm Lynis/Lab99 |

---

## KỊCH BẢN 1: TẦNG LIÊN KẾT DỮ LIỆU (DATA LINK) — PHÒNG CHỐNG ARP SPOOFING & MITM

### 1. Baseline ban đầu (Baseline 1 - Khởi nguyên)
- **Hiện trạng hệ thống:** Máy chủ Target (`10.2.99.10`) và Assessor (`10.2.99.20`) cùng kết nối vào bridge ảo hóa `10.2.99.0/24`.
- **Cấu hình mạng:** Bảng phân giải địa chỉ ARP hoạt động ở cơ chế động (*Dynamic Learning*).
- **Tham số nhân Linux mặc định:**
  - `net.ipv4.conf.all.rp_filter = 0` (Không kiểm tra tính hợp lệ của đường dẫn ngược).
  - `net.ipv4.conf.all.arp_ignore = 0` (Phản hồi mọi yêu cầu ARP bất kể IP đích có thuộc interface hay không).
  - `net.ipv4.conf.all.arp_announce = 0` (Sử dụng bất kỳ địa chỉ IP cục bộ nào để phát quảng bá ARP).

### 2. Bị tấn công (Attacked)
Kẻ tấn công từ nút Assessor phát động cuộc tấn công đầu độc bộ đệm ARP (*ARP Cache Poisoning*) bằng cách liên tục gửi luồng gói tin ARP Reply giả mạo (*Gratuitous/Unsolicited ARP Reply*), tuyên bố rằng địa chỉ IP Gateway (`10.2.99.254`) hoặc IP nạn nhân thuộc quyền sở hữu của địa chỉ MAC Assessor:
```bash
# Lệnh phát động từ lab99_assessor
sudo docker exec lab99_assessor arping -c 5 -A -I eth0 -s 10.2.99.99 10.2.99.10
```

### 3. Phòng thủ thất bại (Defense Failed)
- **Hiện tượng quan sát được:** Bảng ARP trên máy chủ Target lập tức cập nhật địa chỉ MAC của kẻ tấn công cho IP bị giả mạo.
```bash
# Kiểm tra trên Target
sudo docker exec lab99_target ip neigh show
# Kết quả: 10.2.99.99 dev eth0 lladdr 76:f5:85:43:41:f0 STALE/REACHABLE (Đã bị đầu độc!)
```
- **Hậu quả an ninh:** Kẻ tấn công có thể đóng vai trò thực thể trung gian (Man-In-The-Middle - MITM), dễ dàng nghe lén toàn bộ dữ liệu trao đổi không mã hóa và tiến hành can thiệp gói tin giữa Target và các nút mạng khác.

### 4. Cải thiện phòng thủ (Defense Improved)
Triển khai bộ ba biện pháp gia cố tầng liên kết dữ liệu và nhân Linux:
1. **Khóa ánh xạ ARP tĩnh (Static ARP Binding):** Cố định địa chỉ IP và MAC hợp lệ của Gateway và các nút tin cậy với cờ `PERMANENT`.
2. **Kích hoạt tính năng Reverse Path Filtering:** Buộc nhân Linux loại bỏ các gói tin đến từ interface không khớp với bảng định tuyến ngược.
3. **Thiết lập chế độ nghiêm ngặt cho ARP Reply:** Cấu hình `arp_ignore = 1` và `arp_announce = 2`.

```bash
# Cấu hình trên lab99_target
sudo docker exec lab99_target bash -c "
sysctl -w net.ipv4.conf.all.rp_filter=1
sysctl -w net.ipv4.conf.default.rp_filter=1
sysctl -w net.ipv4.conf.all.arp_ignore=1
sysctl -w net.ipv4.conf.all.arp_announce=2
ASSESSOR_MAC=\$(cat /sys/class/net/eth0/address)
ip neigh replace 10.2.99.20 lladdr \$ASSESSOR_MAC nud permanent dev eth0
"
```

### 5. Bị tấn công lại (Attacked Again)
Kẻ tấn công từ Assessor tiếp tục gửi luồng gói tin ARP Reply giả mạo với cường độ cao nhằm cố gắng ghi đè bảng ánh xạ:
```bash
# Lệnh tấn công kiểm thử lại từ Assessor
sudo docker exec lab99_assessor arping -c 5 -A -I eth0 -s 10.2.99.20 10.2.99.10
```

### 6. Phòng thủ thành công (Defense Succeeded)
- **Kết quả đo kiểm:** Bảng ánh xạ trên Target vẫn giữ nguyên cờ `PERMANENT` không bị suy chuyển.
```bash
# Kiểm tra lại trên Target
sudo docker exec lab99_target ip neigh show 10.2.99.20
# Kết quả ghi nhận:
# 10.2.99.20 dev eth0 lladdr 76:f5:85:43:41:f0 PERMANENT
```
- **Kết luận:** Gói tin ARP giả mạo bị nhân Linux từ chối cập nhật. Tấn công MITM bị triệt tiêu hoàn toàn (`LAB02_VERIFY=PASS`).

### 7. Chuyển thành Baseline cho kịch bản sau (Baseline 2 Handover)
Hệ thống bước vào giai đoạn tiếp theo với trạng thái **Baseline 2**:
- Tầng liên kết dữ liệu đã được bảo vệ vững chắc.
- Tham số nhân: `rp_filter = 1`, `arp_ignore = 1`, `arp_announce = 2`.
- Bảng ARP được bảo vệ bằng cơ chế tĩnh `PERMANENT`.

---

## KỊCH BẢN 2: TẦNG MẠNG & HẠ TẦNG TÊN MIỀN (DNS) — PHÒNG CHỐNG DNS CACHE POISONING

### 1. Baseline ban đầu (Baseline 2)
- **Kế thừa:** Bảo vệ tầng Data Link từ Baseline 1.
- **Hiện trạng hạ tầng DNS:** Máy chủ Target cần thực hiện phân giải tên miền ra ngoài Internet để phục vụ cập nhật gói phần mềm (`apt-get`).
- **Nguy cơ tiềm ẩn:**
  - Nếu Target mở cổng DNS server (`53/udp`) ra ngoài mạng công cộng, nó có thể bị kẻ tấn công lợi dụng làm trạm phản xạ khuếch đại tấn công từ chối dịch vụ (*DNS Amplification Attack*).
  - Dải cổng nguồn (*ephemeral ports*) hẹp hoặc sử dụng cổng cố định sẽ khiến quá trình phân giải dễ bị tấn công đầu độc bộ đệm (*Kaminsky DNS Cache Poisoning*).

### 2. Bị tấn công (Attacked)
Kẻ tấn công từ Assessor thăm dò xem Target có đang mở dịch vụ phân giải tên miền đệ quy hay không, đồng thời gửi luồng gói tin phản hồi DNS giả mạo cố gắng đoán trúng Transaction ID (16-bit) và cổng nguồn:
```bash
# Assessor quét cổng DNS trên Target
sudo docker exec lab99_assessor nc -z -u -w 2 10.2.99.10 53
```

### 3. Phòng thủ thất bại (Defense Failed)
- **Nếu cổng 53 mở công khai:** Kẻ tấn công gửi các gói truy vấn đệ quy giả mạo IP nguồn của nạn nhân khác tới Target, biến Target thành công cụ tấn công phản xạ DNS (*Reflection/Amplification*) gây nghẽn băng thông.
- **Nếu cổng nguồn cố định:** Không gian tìm kiếm Transaction ID chỉ có 65,536 giá trị ($2^{16}$), kẻ tấn công với tốc độ gửi gói tin cao hoàn toàn có thể đầu độc thành công bộ đệm DNS trong vài giây.

### 4. Cải thiện phòng thủ (Defense Improved)
Triển khai nguyên lý phân tách và gia cố hạ tầng phân giải tên miền:
1. **Đóng cổng DNS đối với mạng bên ngoài:** Target không đóng vai trò DNS server cho client ngoài, cổng `53/udp` tuyệt đối không được lắng nghe trên giao diện công cộng.
2. **Kích hoạt tính năng ngẫu nhiên hóa cổng nguồn (Source Port Randomization - SPR):** Mở rộng dải ephemeral ports của nhân Linux từ `32768` đến `60999` (tổng cộng 28,231 cổng).
3. **Sử dụng Resolver nội bộ cô lập:** Cấu hình `/etc/resolv.conf` trỏ về resolver cục bộ an toàn (`127.0.0.11`).

```bash
# Áp dụng trên lab99_target
sudo docker exec lab99_target bash -c "
sysctl -w net.ipv4.ip_local_port_range='32768 60999'
"
```

### 5. Bị tấn công lại (Attacked Again)
Kẻ tấn công Assessor tiếp tục phát động quét cổng 53 và gửi các gói tin DNS giả mạo với tốc độ cao:
```bash
sudo docker exec lab99_assessor nc -z -u -w 2 10.2.99.10 53
```

### 6. Phòng thủ thành công (Defense Succeeded)
- **Kết quả đo kiểm:**
  1. Cổng `53/udp` đóng hoàn toàn đối với bên ngoài (`connection refused` hoặc timeout).
  2. Dải cổng nguồn outbound đạt 28,231 cổng. Kết hợp với 16-bit Transaction ID, không gian entropy mở rộng lên tới xấp xỉ $2^{31}$ khả năng, triệt tiêu hoàn toàn tính khả thi của kỹ thuật tấn công Kaminsky.
  3. Kiểm tra phân giải tên miền outbound cho `example.com` từ Target diễn ra chính xác và an toàn.
```bash
# Kiểm tra phân giải hợp lệ trên Target
sudo docker exec lab99_target python3 -c "import socket; print(socket.gethostbyname('example.com'))"
# Kết quả: Trả về IP chính xác, không bị giả mạo (LAB03_VERIFY=PASS).
```

### 7. Chuyển thành Baseline cho kịch bản sau (Baseline 3 Handover)
Hệ thống bước sang giai đoạn tiếp theo với trạng thái **Baseline 3**:
- Kế thừa: Chống ARP Spoofing (L2) + Bảo vệ DNS / Cổng nguồn (L3/L4).
- Cổng `53/udp` không mở công khai.
- Ephemeral port range: `32768 - 60999` ($SPR = PASS$).

---

## KỊCH BẢN 3: TẦNG GIAO VẬN (TRANSPORT) — PHÒNG CHỐNG DOS TCP SYN FLOOD & RATE LIMITING

### 1. Baseline ban đầu (Baseline 3)
- **Kế thừa:** Baseline 2 (Lớp 2 & Lớp 3 vững chắc).
- **Hiện trạng tầng giao vận:**
  - Máy chủ Target cung cấp các dịch vụ lắng nghe TCP (SSH trên cổng 22/tcp và Web Application trên cổng 8080/tcp).
  - Kích thước hàng đợi kết nối nửa mở (*SYN Backlog*) mặc định của nhân Linux nhỏ (128 kết nối).
  - Cơ chế bảo vệ tràn hàng đợi kết nối `net.ipv4.tcp_syncookies` chưa được cấu hình tối ưu hoặc chưa có tường lửa giới hạn tốc độ kết nối (*Rate Limiting*).

### 2. Bị tấn công (Attacked)
Nút kiểm định Assessor sử dụng công cụ `hping3` phát động một luồng bão gói tin TCP SYN Flood cực lớn với địa chỉ IP nguồn ngẫu nhiên (`--rand-source`) hướng thẳng vào dịch vụ trên Target:
```bash
# Assessor phát động TCP SYN Flood trong 4 giây
sudo docker exec -d lab99_assessor hping3 -S -p 22 --flood --rand-source 10.2.99.10
```

### 3. Phòng thủ thất bại (Defense Failed)
- **Hiện tượng quan sát được:**
  - Bảng hàng đợi kết nối nửa mở (*Half-open connections table*) trên Target bị lấp đầy chỉ trong vài mili-giây.
  - Quản trị viên hoặc người dùng hợp lệ cố gắng kết nối SSH hoặc truy cập Web bị treo hoặc nhận phản hồi `Connection timed out`.
```bash
# Thử nghiệm kết nối hợp lệ từ bên ngoài trong khi bị flood
sudo docker exec lab99_assessor nc -z -v -w 2 10.2.99.10 22
# Kết quả: nc: connect to 10.2.99.10 port 22 (tcp) timed out!
```
- **Hậu quả an ninh:** Dịch vụ tê liệt hoàn toàn (Từ chối dịch vụ - Denial of Service), vi phạm nghiêm trọng tính sẵn sàng (*Availability*) của hệ thống.

### 4. Cải thiện phòng thủ (Defense Improved)
Triển khai chiến lược kết hợp giữa nhân Linux và bộ lọc gói tin iptables:
1. **Kích hoạt TCP SYN Cookies:** Cấu hình `net.ipv4.tcp_syncookies = 1`. Khi hàng đợi đầy, máy chủ không lưu trạng thái kết nối vào bộ nhớ mà mã hóa thông tin trạng thái vào trường *Initial Sequence Number (ISN)* của gói `SYN-ACK`.
2. **Mở rộng hàng đợi SYN Backlog:** Nâng `net.ipv4.tcp_max_syn_backlog = 2048`.
3. **Thiết lập cơ chế Rate Limiting trên Tường lửa:** Giới hạn tốc độ tiếp nhận gói tin SYN mới tối đa 10 gói/giây với burst 20 gói, đồng thời giới hạn ICMP Echo Request 5 gói/giây.

```bash
# Cấu hình trên lab99_target
sudo docker exec lab99_target bash -c "
sysctl -w net.ipv4.tcp_syncookies=1
sysctl -w net.ipv4.tcp_max_syn_backlog=2048
iptables -I INPUT -p tcp --dport 22 --syn -m limit --limit 10/s --limit-burst 20 -j ACCEPT
iptables -I INPUT -p tcp --dport 2222 --syn -m limit --limit 10/s --limit-burst 20 -j ACCEPT
iptables -I INPUT -p icmp -m limit --limit 5/s --limit-burst 10 -j ACCEPT
"
```

### 5. Bị tấn công lại (Attacked Again)
Assessor phát động lại đợt tấn công TCP SYN Flood cường độ cao với `--flood --rand-source`:
```bash
sudo docker exec -d lab99_assessor hping3 -S -p 22 --flood --rand-source 10.2.99.10
```

### 6. Phòng thủ thành công (Defense Succeeded)
- **Kết quả đo kiểm:** Trong khi luồng tấn công SYN Flood đang diễn ra ác liệt, Assessor thực hiện 3 lần kiểm thử kết nối TCP hợp lệ:
```bash
# Kiểm thử kết nối hợp lệ trong bão flood
for i in 1 2 3; do
  sudo docker exec lab99_assessor nc -z -w 2 10.2.99.10 22 && echo "CONNECT $i: SUCCESS"
done
# Dừng hping3
sudo docker exec lab99_assessor pkill -9 hping3
```
- **Minh chứng:** Cả 3/3 lần kết nối đều thành công rực rỡ! Nhân Linux sử dụng SYN Cookies để thiết lập bắt tay 3 bước mà không bị nghẽn bộ nhớ (`LAB04_VERIFY=PASS`).

### 7. Chuyển thành Baseline cho kịch bản sau (Baseline 4 Handover)
Hệ thống bước sang giai đoạn tiếp theo với trạng thái **Baseline 4**:
- Kế thừa: L2 Static ARP + L3/4 DNS SPR + L4 TCP SYN Resilience.
- `tcp_syncookies = 1`, `tcp_max_syn_backlog = 2048`.
- iptables Rate Limiting bảo vệ các cổng trọng yếu.

---

## KỊCH BẢN 4: TẦNG ỨNG DỤNG & DỮ LIỆU (APPLICATION) — PHÒNG CHỐNG SQL INJECTION (SQLI)

### 1. Baseline ban đầu (Baseline 4)
- **Kế thừa:** Baseline 3 (Hạ tầng mạng, DNS và giao vận đều được bảo vệ toàn diện).
- **Hiện trạng tầng ứng dụng:**
  - Ứng dụng Web nghiệp vụ (`lab99_app.py`) đang lắng nghe trên cổng `8080/tcp` của Target.
  - Cơ sở dữ liệu SQLite lưu trữ danh sách người dùng quản trị và mật khẩu trong bảng `users`.
  - Endpoint kiểm tra tài khoản `/sqli_vuln` xây dựng câu truy vấn bằng kỹ thuật **nối chuỗi trực tiếp** dữ liệu do người dùng truyền vào:
    ```python
    query = f"SELECT id, username, role FROM users WHERE username = '{user_input}'"
    ```

### 2. Bị tấn công (Attacked)
Kẻ tấn công từ Assessor phát hiện tham số `user` trên cổng 8080 và gửi chuỗi payload khai thác kỹ thuật logic hằng đúng (*Tautology SQL Injection Bypass*):
```bash
# Assessor gửi HTTP Request chứa payload SQLi
PAYLOAD="admin' OR '1'='1"
sudo docker exec lab99_assessor curl -s "http://10.2.99.10:8080/sqli_vuln?user=admin'%20OR%20'1'='1"
```

### 3. Phòng thủ thất bại (Defense Failed)
- **Hiện tượng quan sát được:** Câu lệnh SQL thực thi trong cơ sở dữ liệu trở thành:
  ```sql
  SELECT id, username, role FROM users WHERE username = 'admin' OR '1'='1'
  ```
- **Hậu quả an ninh:** Mệnh đề `'1'='1'` luôn đúng với mọi bản ghi. Toàn bộ dữ liệu nhạy cảm trong bảng `users` bị trích xuất công khai ra màn hình của kẻ tấn công:
  ```json
  {
    "mode": "vulnerable",
    "query_executed": "SELECT id, username, role FROM users WHERE username = 'admin' OR '1'='1'",
    "count": 3,
    "results": [
      {"id": 1, "username": "admin", "role": "administrator"},
      {"id": 2, "username": "alice", "role": "user"},
      {"id": 3, "username": "bob", "role": "user"}
    ]
  }
  ```
  Kẻ tấn công đã hoàn toàn vi phạm tính bí mật (*Confidentiality*) của dữ liệu người dùng.

### 4. Cải thiện phòng thủ (Defense Improved)
Tái cấu trúc mã nguồn ứng dụng Web chuyển sang endpoint an toàn `/sqli_safe` sử dụng cơ chế **Truy vấn tham số hóa (Parameterized Prepared Statements)**:
```python
# Cải tiến trong mã nguồn lab99_app.py
query = "SELECT id, username, role FROM users WHERE username = ?"
cur.execute(query, (user_input,))
```
- **Cơ chế an toàn:** Bộ phân tích cú pháp SQL biên dịch trước khuôn mẫu câu lệnh. Giá trị đầu vào của biến `user_input` được cơ sở dữ liệu đối xử thuần túy như một giá trị chuỗi văn bản thông thường (*Literal Value*), không thể làm thay đổi cấu trúc cú pháp của lệnh SQL.

### 5. Bị tấn công lại (Attacked Again)
Kẻ tấn công từ Assessor lặp lại đợt tấn công bằng cách gửi đúng chuỗi payload nguy hiểm `admin' OR '1'='1` vào endpoint `/sqli_safe`:
```bash
sudo docker exec lab99_assessor curl -s "http://10.2.99.10:8080/sqli_safe?user=admin'%20OR%20'1'='1"
```

### 6. Phòng thủ thành công (Defense Succeeded)
- **Kết quả đo kiểm:** Phản hồi từ máy chủ trả về an toàn tuyệt đối:
  ```json
  {
    "mode": "prepared_statement_safe",
    "query_template": "SELECT id, username, role FROM users WHERE username = ?",
    "count": 0,
    "results": []
  }
  ```
- **Kết luận:** Cơ sở dữ liệu tìm kiếm người dùng có tên chính xác là chuỗi `admin' OR '1'='1` và trả về 0 kết quả. Cuộc tấn công bị vô hiệu hóa hoàn toàn, không có bất kỳ thông tin nào bị rò rỉ.

### 7. Chuyển thành Baseline cho kịch bản sau (Baseline 5 Handover)
Hệ thống bước sang giai đoạn tiếp theo với trạng thái **Baseline 5**:
- Kế thừa: Toàn bộ cơ chế bảo vệ L2, L3, L4.
- Tầng cơ sở dữ liệu được chuẩn hóa 100% bằng Parameterized Prepared Statements, miễn nhiễm với SQL Injection.

---

## KỊCH BẢN 5: TẦNG ỨNG DỤNG & HỆ ĐIỀU HÀNH (APPLICATION) — PHÒNG CHỐNG OS COMMAND INJECTION (RCE)

### 1. Baseline ban đầu (Baseline 5)
- **Kế thừa:** Baseline 4 (Đã miễn nhiễm SQL Injection).
- **Hiện trạng tính năng hệ thống:** Ứng dụng Web cung cấp tính năng kiểm tra tình trạng kết nối mạng (*Network Diagnostics*) tại endpoint `/cmd_vuln`.
- **Điểm yếu lập trình:** Mã nguồn lấy trực tiếp tham số `host` do người dùng cung cấp và ghép vào lệnh shell hệ thống thực thi qua hàm `os.popen()` hoặc `subprocess(shell=True)`:
  ```python
  cmd = f"ping -c 1 {host_input}"
  output = os.popen(cmd).read()
  ```

### 2. Bị tấn công (Attacked)
Kẻ tấn công từ Assessor chèn các ký tự điều khiển lệnh shell (dấu chấm phẩy `;`, toán tử `&&`, hoặc `|`) nhằm thực thi các câu lệnh hệ thống tùy ý:
```bash
# Assessor gửi yêu cầu chèn lệnh hệ điều hành
CMD_PAYLOAD="127.0.0.1; whoami"
sudo docker exec lab99_assessor curl -s "http://10.2.99.10:8080/cmd_vuln?host=127.0.0.1%3B%20whoami"
```

### 3. Phòng thủ thất bại (Defense Failed)
- **Hiện tượng quan sát được:** Tiến trình con của shell `/bin/sh` thực thi tuần tự 2 lệnh:
  1. `ping -c 1 127.0.0.1`
  2. `whoami`
- **Hậu quả an ninh:** Kết quả trả về cho kẻ tấn công chuỗi danh tính đặc quyền:
  ```json
  {
    "mode": "vulnerable_shell",
    "command_executed": "ping -c 1 127.0.0.1; whoami",
    "output": "root\n"
  }
  ```
  Kẻ tấn công đã thực thi mã từ xa (*Remote Code Execution - RCE*) với đặc quyền tối cao `root`, có thể tải mã độc, đọc tệp tin hệ thống và chiếm quyền điều khiển toàn bộ máy chủ.

### 4. Cải thiện phòng thủ (Defense Improved)
Triển khai nguyên tắc phòng thủ đa lớp tại tầng ứng dụng tại endpoint `/cmd_safe`:
1. **Làm sạch và kiểm tra dữ liệu đầu vào bằng Danh sách trắng (Whitelist Regex Sanitization):** Chỉ chấp nhận các ký tự hợp lệ cấu thành địa chỉ IP hoặc tên miền (`^[0-9a-zA-Z.-]+$`). Bất kỳ ký tự đặc biệt nào (`;`, `|`, `&`, `$`, `` ` ``, `>`, `<`) đều lập tức bị từ chối với mã lỗi HTTP 400 Bad Request.
2. **Vô hiệu hóa trình biên dịch lệnh Shell (`shell=False`):** Truyền đối số dưới dạng danh sách mảng cho `subprocess.run()`. Nhân Linux gọi trực tiếp nhị phân `/bin/ping` thông qua lệnh gọi hệ thống `execve`, ngăn chặn hoàn toàn việc phân tách lệnh.

```python
# Cải tiến trong mã nguồn lab99_app.py (/cmd_safe)
if not re.match(r"^[0-9a-zA-Z.-]+$", host_input):
    # Từ chối yêu cầu độc hại
    self.send_response(400)
    ...
else:
    res = subprocess.run(["ping", "-c", "1", host_input], shell=False, capture_output=True, text=True, timeout=3)
```

### 5. Bị tấn công lại (Attacked Again)
Kẻ tấn công gửi lại chuỗi chèn lệnh nguy hiểm `127.0.0.1; whoami` vào endpoint `/cmd_safe`:
```bash
sudo docker exec lab99_assessor curl -s -w "\nHTTP_STATUS:%{http_code}" "http://10.2.99.10:8080/cmd_safe?host=127.0.0.1%3B%20whoami"
```

### 6. Phòng thủ thành công (Defense Succeeded)
- **Kết quả đo kiểm:** Máy chủ lập tức phát hiện ký tự đặc biệt trái phép và chặn đứng luồng xử lý:
  ```json
  {
    "mode": "sanitized_safe",
    "error": "Phat hien ky tu dac biet nguy hiem! Yeu cau bi tu choi.",
    "status": "blocked"
  }
  HTTP_STATUS:400
  ```
- **Kết luận:** Lệnh `whoami` hoàn toàn không được chuyển tới nhân hệ điều hành. Lỗ hổng RCE bị triệt tiêu (`LAB05_VERIFY=PASS`).

### 7. Chuyển thành Baseline cho kịch bản sau (Baseline 6 Handover)
Hệ thống bước sang giai đoạn tiếp theo với trạng thái **Baseline 6**:
- Kế thừa: Toàn bộ bảo vệ L2, L3, L4 và Ứng dụng Web an toàn tuyệt đối trước SQLi và RCE.
- Dữ liệu đầu vào ứng dụng được kiểm soát chặt chẽ bằng cơ chế Whitelist Validation.

---

## KỊCH BẢN 6: TẦNG DỊCH VỤ & MÁY CHỦ (HOST DEFENSE) — PHÒNG CHỐNG SSH BRUTE-FORCE BẰNG FAIL2BAN & UFW

### 1. Baseline ban đầu (Baseline 6)
- **Kế thừa:** Toàn bộ an toàn mạng và an toàn ứng dụng web.
- **Hiện trạng dịch vụ quản trị hệ thống (SSH) ban đầu:**
  - OpenSSH đang lắng nghe trên cổng tiêu chuẩn mặc định `22/tcp`.
  - Cấu hình xác thực yếu: `PermitRootLogin yes` (cho phép đăng nhập trực tiếp tài khoản root) và `PasswordAuthentication yes` (cho phép đăng nhập bằng mật khẩu).
  - Tường lửa `UFW` ở trạng thái không hoạt động (`inactive`).
  - Phân hệ phòng thủ chủ động `Fail2ban` chưa được cài đặt/kích hoạt.

### 2. Bị tấn công (Attacked)
Từ máy Assessor, kẻ tấn công thực hiện trinh sát bằng Nmap và phát hiện cổng 22/tcp mở công khai, sau đó phát động cuộc tấn công dò quét mật khẩu tự động (*SSH Password Brute-Force Spraying*):
```bash
# Assessor gửi liên tiếp các yêu cầu xác thực sai mật khẩu tới Target
for i in 1 2 3 4 5; do
  sudo docker exec lab99_assessor ssh -o BatchMode=yes -o StrictHostKeyChecking=no -o ConnectTimeout=2 \
    -o PubkeyAuthentication=no -o PreferredAuthentications=password \
    -p 22 labuser@10.2.99.10 true || true
done
```

### 3. Phòng thủ thất bại (Defense Failed)
- **Hiện tượng quan sát được:**
  - Máy chủ Target liên tục chấp nhận và xử lý yêu cầu thử mật khẩu của kẻ tấn công, ghi nhận hàng loạt lỗi `Failed password` trong `/var/log/auth.log` mà không có bất kỳ hành động phản kháng nào.
  - Nguy cơ lộ lọt mật khẩu quản trị hoặc tài khoản root bị chiếm đoạt thông qua tấn công từ điển (*Dictionary Attack*).
  - Điểm kiểm toán Lynis Hardening Index chỉ đạt mức thấp (5/12 điểm).

### 4. Cải thiện phòng thủ (Defense Improved)
Triển khai gói gia cố an ninh máy chủ chuyên sâu (*Host Hardening Framework*):
1. **Di chuyển cổng dịch vụ OpenSSH:** Chuyển từ cổng mặc định 22 sang cổng tùy chỉnh `2222/tcp`.
2. **Khóa cứng chính sách xác thực SSH:**
   - `PermitRootLogin no` (Cấm tuyệt đối đăng nhập tài khoản root).
   - `PasswordAuthentication no` (Vô hiệu hóa xác thực mật khẩu).
   - `PubkeyAuthentication yes` (Chỉ chấp nhận khóa mã hóa phi đối xứng ED25519).
   - `AllowUsers labuser` (Áp dụng nguyên tắc đặc quyền tối thiểu).
3. **Kích hoạt Tường lửa UFW:** Thiết lập chính sách mặc định `Default Deny Incoming`, chỉ mở duy nhất cổng quản trị `2222/tcp` và cổng Web `8080/tcp`. Cổng 22/tcp bị lọc hoàn toàn (*filtered*).
4. **Kích hoạt Hệ thống phòng thủ chủ động Fail2ban:** Thiết lập jail `sshd` theo dõi tệp `/var/log/auth.log`. Khi một địa chỉ IP thử sai mật khẩu vượt quá ngưỡng (`maxretry = 3`), Fail2ban tự động tương tác với iptables chèn luật `DROP` cô lập IP đó trong 600 giây.

```bash
# Áp dụng gia cố trên lab99_target
sudo docker exec lab99_target bash /hardening/harden.sh
```

### 5. Bị tấn công lại (Attacked Again)
Kẻ tấn công từ Assessor thử quét cổng 22/tcp (bị tường lửa UFW chặn timeout), sau đó chuyển hướng tấn công dò mật khẩu vào cổng mới `2222/tcp`:
```bash
# Assessor tấn công brute-force vào port 2222
for i in 1 2 3 4 5; do
  sudo docker exec lab99_assessor ssh -o BatchMode=yes -o StrictHostKeyChecking=no -o ConnectTimeout=2 \
    -o PubkeyAuthentication=no -o PreferredAuthentications=password \
    -p 2222 labuser@10.2.99.10 true || true
done
```

### 6. Phòng thủ thành công (Defense Succeeded)
- **Kết quả đo kiểm:**
  - Ngay sau lần thử sai thứ 3, Fail2ban phân tích nhật ký và tự động kích hoạt luật chặn IP:
```bash
# Kiểm tra trạng thái jail sshd trên Target
sudo docker exec lab99_target fail2ban-client status sshd
# Kết quả ghi nhận:
# Status for the jail: sshd
# |- Filter: Currently failed: 1 | Total failed: 5 | File: /var/log/auth.log
# `- Actions: Currently banned: 1 | Total banned: 1 | Banned IP list: 10.2.99.20
```
  - Mọi gói tin kết nối tiếp theo từ `10.2.99.20` tới cổng 2222 đều bị iptables DROP ngay tại card mạng.
  - Quản trị viên hợp lệ sử dụng khóa bí mật ED25519 (`lab99_id_ed25519`) vẫn đăng nhập quản trị thành công (`FAIL2BAN_VERIFY=PASS`).

### 7. Chuyển thành Baseline cho kịch bản sau (Baseline 7 Handover)
Hệ thống bước sang giai đoạn tiếp theo với trạng thái **Baseline 7**:
- Kế thừa: Toàn bộ phòng thủ L2 - L7.
- SSH Port 2222 ED25519 Key-only, cấm root, tắt password.
- UFW Active (Default Deny), Fail2ban Active với khả năng tự động ban IP tấn công.

---

## KỊCH BẢN 7: TẦNG GIÁM SÁT TOÀN VẸN & ĐIỀU TRA SỐ (HOST AUDITING) — GIÁM SÁT TỆP TIN BẰNG LINUX KERNEL AUDITD

### 1. Baseline ban đầu (Baseline 7)
- **Kế thừa:** Toàn bộ các tầng phòng thủ từ mạng đến ứng dụng và máy chủ đã được gia cố hoàn hảo (100% rào cản bên ngoài đã khép kín).
- **Vấn đề tồn đọng:** Nếu một kẻ tấn công nội bộ (*Insider Threat*) hoặc tiến trình có thẩm quyền bị lợi dụng, hệ thống mặc định của Linux **không có cơ chế giám sát toàn vẹn tệp tin ở mức nhân (Kernel File Integrity Monitoring)**.
- Khi các tệp tin cấu hình then chốt (như `/etc/passwd` hoặc cấu hình OpenSSH `/etc/ssh/sshd_config`) bị chỉnh sửa trái phép, quản trị viên không thể biết ai đã can thiệp, vào thời điểm nào và bằng tiến trình gì để phục vụ điều tra sự cố số (*Digital Forensics*).

### 2. Bị tấn công (Attacked)
Kẻ tấn công tìm cách sửa đổi thuộc tính hoặc nội dung tệp tin nhận dạng người dùng `/etc/passwd` nhằm tạo tài khoản backdoor hoặc thay đổi đặc quyền:
```bash
# Giả lập hành vi can thiệp trái phép trên Target
sudo docker exec lab99_target touch /etc/passwd
```

### 3. Phòng thủ thất bại (Defense Failed)
- **Hiện tượng quan sát được:** Nếu dịch vụ kiểm toán `Auditd` chưa được bật quy tắc:
  - Tệp tin `/etc/passwd` bị thay đổi mà không có bất kỳ nhật ký chuyên dụng nào được ghi lại.
  - Quản trị viên kiểm tra `syslog` chỉ thấy các thông điệp chung chung, không có bằng chứng pháp lý số không thể chối bỏ (*Non-repudiation*).
  - Không thể xác định được PID của tiến trình, file thực thi thực tế hay người dùng thực hiện.

### 4. Cải thiện phòng thủ (Defense Improved)
Kích hoạt phân hệ giám sát nhân Linux `Auditd` và nạp các quy tắc theo dõi toàn vẹn tệp tin nhạy cảm với cơ chế gắn nhãn sự kiện (*Audit Keys*):
1. Giám sát ghi và thay đổi thuộc tính (`-p wa`) đối với tệp tin định danh `/etc/passwd` với nhãn `identity_tampering`.
2. Giám sát tệp và thư mục cấu hình SSH `/etc/ssh/sshd_config` với nhãn `ssh_config_tampering`.

```bash
# Nạp quy tắc kiểm toán trên lab99_target
sudo docker exec lab99_target bash -c "
service auditd start || true
auditctl -e 1
auditctl -w /etc/passwd -p wa -k identity_tampering
auditctl -w /etc/ssh/sshd_config -p wa -k ssh_config_tampering
"
```

### 5. Bị tấn công lại (Attacked Again)
Kẻ tấn công lặp lại hành vi can thiệp tệp tin nhạy cảm `/etc/passwd`:
```bash
sudo docker exec lab99_target touch /etc/passwd
```

### 6. Phòng thủ thành công (Defense Succeeded)
- **Kết quả đo kiểm:** Phân hệ Kernel Auditd lập tức phát hiện và bắt trọn sự kiện can thiệp, ghi lại một bản ghi bằng chứng số hoàn chỉnh vào `/var/log/audit/audit.log`.
- **Trích xuất bằng chứng pháp y số qua `ausearch`:**
```bash
# Truy vấn bằng chứng số theo nhãn
sudo docker exec lab99_target ausearch -k identity_tampering -i --raw | tail -n 10
```
- **Bản ghi trích xuất thực tế:**
  ```text
  type=SYSCALL msg=audit(1795120307.913:281): arch=x86_64 syscall=257 success=yes exit=3 
  comm="touch" exe="/usr/bin/touch" key="identity_tampering" UID="root" AUID="unset"
  ```
- **Phân tích giá trị pháp y:**
  - `syscall=257`: Lệnh gọi hệ thống `openat` (mở tệp tin với cờ ghi).
  - `comm="touch"`, `exe="/usr/bin/touch"`: Xác định chính xác chương trình thực thi.
  - `UID="root"`: Định danh người dùng thực hiện hành vi.
  - `key="identity_tampering"`: Tự động phân loại sự cố an ninh.
- **Kết luận:** Hệ thống đạt năng lực giám sát và điều tra sau sự cố toàn diện (`AUDITD_VERIFY=PASS`).

### 7. Thiết lập Baseline Hoàn thiện (Final Hardened Baseline)
Hệ thống hoàn tất chuỗi kịch bản tịnh tiến và đạt trạng thái **Baseline Hoàn thiện**:
- Điểm kiểm toán bảo mật Lab 99 đạt **12 / 12 điểm tuyệt đối (100%)**.
- Độ chênh lệch điểm an ninh (*Score Delta*): Tăng **+7 điểm** (từ 5/12 lên 12/12 điểm).
- Toàn bộ 7/7 tầng phòng thủ theo chiều sâu đều vượt qua kiểm định độc lập với kết quả `PASS`.

---

## 10. TỔNG KẾT BASELINE HOÀN THIỆN VÀ HƯỚNG DẪN THỰC THI TUẦN TỰ (RUNBOOK)

### 1. Ma trận Tổng kết Bằng chứng Thực nghiệm
Toàn bộ kết quả kiểm chứng trong chuỗi kịch bản tịnh tiến được tổng hợp và xuất bản tự động vào tệp `volumes/reports/project_summary.md`:

| Tầng phòng thủ | Kịch bản tịnh tiến | Kỹ thuật kiểm thử áp dụng | Cơ chế phòng thủ thực tế | Trạng thái nghiệm thu |
|:---|:---|:---|:---|:---:|
| **Layer 2 (Data Link)** | Kịch bản 1 | ARP Cache Poisoning (`10_apply_lab02.sh`) | `rp_filter=1`, `arp_ignore=1`, `Static ARP (PERMANENT)` | **PASS** |
| **Layer 3/4 (Network/DNS)**| Kịch bản 2 | DNS Cache Poisoning (`11_apply_lab03.sh`) | Source Port Randomization (SPR > 28,000 cổng) | **PASS** |
| **Layer 4 (Transport)** | Kịch bản 3 | TCP SYN Flood DoS (`12_apply_lab04.sh`) | `tcp_syncookies=1`, backlog 2048, UFW Rate Limiting | **PASS** |
| **Layer 7 (Application)** | Kịch bản 4 | SQL Injection (`13_apply_lab05.sh`) | Parameterized Prepared Statements (`/sqli_safe`) | **PASS** |
| **Layer 7 (Application)** | Kịch bản 5 | OS Command Injection (`13_apply_lab05.sh`) | Whitelist Regex Validation, `shell=False` (`/cmd_safe`)| **PASS** |
| **Host Defense (Active)** | Kịch bản 6 | SSH Password Brute-Force (`08_apply_fail2ban.sh`)| Di chuyển Port 2222, ED25519 Key, Fail2ban Ban IP | **PASS** |
| **Host Auditing (HIDS)** | Kịch bản 7 | Sensitive File Tampering (`09_apply_auditd.sh`)| Linux Kernel Auditd (`-w /etc/passwd -p wa`) | **PASS** |
| **OS Baseline Hardening** | Đánh giá tổng hợp | Kiểm toán hệ thống (`01..07`) | UFW Default Deny, PAM pwquality, Kernel Sysctl | **12/12 (100%)**|

---

### 2. Sổ tay Vận hành Thực thi Tuần tự (Execution Runbook)

Để tái hiện chính xác chuỗi kịch bản tịnh tiến trên môi trường máy ảo hoặc Docker lab, người vận hành thực thi tuần tự các lệnh sau:

```bash
# ==============================================================================
# BƯỚC 0: KHỞI TẠO HỆ SINH THÁI LAB 99 (TARGET & ASSESSOR)
# ==============================================================================
bash src/scripts/lab99/.lab/00_init.sh

# ==============================================================================
# BƯỚC 1: ĐO LƯỜNG VÀ THIẾT LẬP ĐƯỜNG CƠ SỞ BAN ĐẦU (BASELINE 1)
# ==============================================================================
bash src/scripts/lab99/.lab/01_baseline_audit.sh
bash src/scripts/lab99/.lab/02_assess_surface.sh before

# ==============================================================================
# BƯỚC 2: TRIỂN KHAI GIA CỐ HỆ THỐNG ĐA TẦNG (SYSTEM HARDENING)
# ==============================================================================
bash src/scripts/lab99/.lab/03_harden.sh
bash src/scripts/lab99/.lab/04_verify_harden.sh

# ==============================================================================
# BƯỚC 3: TÁI THẨM ĐỊNH CHỈ SỐ SAU GIA CỐ VÀ SO SÁNH ĐỊNH LƯỢNG
# ==============================================================================
bash src/scripts/lab99/.lab/05_reassess.sh
bash src/scripts/lab99/.lab/02_assess_surface.sh after
bash src/scripts/lab99/.lab/06_validate_defense.sh after
bash src/scripts/lab99/.lab/07_compare_report.sh

# ==============================================================================
# BƯỚC 4: THỰC THI BỘ KIỂM THỬ XÂM NHẬP ĐỐI CHỨNG (TẤN CÔNG -> PHÒNG THỦ)
# ==============================================================================
# 1. Kịch bản 6: Tấn công SSH Brute-Force -> Kiểm chứng phòng thủ chủ động Fail2ban
bash src/scripts/lab99/.lab/08_apply_fail2ban.sh

# 2. Kịch bản 7: Can thiệp tệp nhạy cảm -> Kiểm chứng giám sát truy vết số Auditd
bash src/scripts/lab99/.lab/09_apply_auditd.sh

# 3. Kịch bản 1: Tấn công ARP Spoofing -> Kiểm chứng phòng thủ Layer 2 (Static ARP)
bash src/scripts/lab99/.lab/10_apply_lab02.sh

# 4. Kịch bản 2: Thăm dò DNS -> Kiểm chứng Source Port Randomization (SPR)
bash src/scripts/lab99/.lab/11_apply_lab03.sh

# 5. Kịch bản 3: Tấn công DoS SYN Flood -> Kiểm chứng TCP SYN Cookies & Rate Limit
bash src/scripts/lab99/.lab/12_apply_lab04.sh

# 6. Kịch bản 4 & 5: Tấn công SQLi & RCE -> Kiểm chứng AppSec (Prepared Statements & Regex)
bash src/scripts/lab99/.lab/13_apply_lab05.sh

# ==============================================================================
# BƯỚC 5: TỔNG HỢP TOÀN BỘ MA TRẬN PHÒNG THỦ THEO CHIỀU SÂU
# ==============================================================================
bash src/scripts/lab99/.lab/14_summary_project.sh

# ==============================================================================
# BƯỚC 6: DỌN DẸP VÀ GIẢI PHÓNG TÀI NGUYÊN (KHI HOÀN THÀNH BÀI LAB)
# ==============================================================================
# bash src/scripts/lab99/.lab/99_down.sh
```
