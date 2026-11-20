# ĐÁNH GIÁ NĂNG LỰC PHÒNG VỆ VÀ DANH MỤC CÁC THỦ THUẬT TẤN CÔNG BỊ TRIỆT TIÊU – LAB 99

## Đề tài: Nghiên cứu, Thiết kế và Đánh giá Năng lực Hệ thống Phòng thủ theo Chiều sâu (Defense-in-Depth) kết hợp Gia cố Hệ điều hành Linux

---

## Tổng quan năng lực phòng thủ

Sau khi triển khai toàn diện quy trình gia cố an ninh đa tầng (**Defense-in-Depth**) trong Lab 99, hệ thống máy chủ mục tiêu (**Target Node** `10.2.99.10`) sở hữu năng lực tự bảo vệ, cô lập và vô hiệu hóa trực tiếp **7 nhóm thủ thuật tấn công mạng và dịch vụ** phổ biến, bao phủ từ tầng vật lý/liên kết dữ liệu đến tầng ứng dụng và nhân hệ điều hành.

---

## Chi tiết 7 nhóm thủ thuật tấn công bị ngăn chặn

### 1. Tấn công đầu độc bảng ánh xạ mạng nội bộ (Layer 2 - ARP Spoofing / MITM)

* **Kỹ thuật tấn công bị chặn:**
  * Kẻ tấn công trong cùng mạng LAN (`10.2.99.0/24`) phát tán các gói tin ARP Reply giả mạo không yêu cầu (`arping -A`, `arpspoof`, Ettercap) nhằm ghi đè bảng ARP Cache của Target.
  * Mục tiêu của kẻ tấn công là hướng toàn bộ lưu lượng mạng nội bộ đi qua máy kiểm thử (Assessor) để nghe lén gói tin (Sniffing) hoặc sửa đổi dữ liệu truyền thông (Man-In-The-Middle - MITM).
* **Cơ chế phòng thủ kích hoạt:**
  * **Static ARP Mapping:** Cố định cứng ánh xạ địa chỉ IP-MAC của Gateway (`10.2.99.254`) và trạm kiểm thử Assessor (`10.2.99.20`) vào bảng ARP với cờ `nud permanent`.
  * **Reverse Path Filtering:** Kích hoạt tham số nhân `net.ipv4.conf.all.rp_filter = 1` để kiểm tra tính hợp lệ của đường định tuyến ngược và loại bỏ các gói tin có địa chỉ IP nguồn bất thường.
  * **ARP Flux Controls:** Thiết lập `net.ipv4.conf.all.arp_ignore = 1` và `arp_announce = 2` để kiểm soát chặt chẽ phản hồi ARP.
* **Kết quả đo kiểm thực tế:**
  * Bảng ARP trên Target giữ nguyên cờ `PERMANENT`. Mọi gói tin ARP Reply giả mạo đều bị kernel bỏ qua hoàn toàn, loại bỏ triệt để nguy cơ đánh cắp dữ liệu đường truyền.
  * **Minh chứng nghiệm thu:** `volumes/reports/lab02_verify.log` (`LAB02_VERIFY=PASS`).

---

### 2. Tấn công giả mạo nguồn và đầu độc hạ tầng DNS (Layer 3/4 - DNS Poisoning & Reflector)

* **Kỹ thuật tấn công bị chặn:**
  * **Kaminsky DNS Cache Poisoning:** Kẻ tấn công liên tục gửi các truy vấn DNS kèm phản hồi giả mạo với tốc độ cao, cố gắng đoán trúng số hiệu cổng nguồn (Source Port) và mã giao dịch (Transaction ID 16-bit) để đầu độc bản ghi phân giải tên miền trong bộ nhớ đệm (Cache).
  * **DNS Amplification DDoS (Phản xạ khuếch đại):** Khai thác máy chủ mở cổng 53/udp công khai để gửi các truy vấn phân giải (như `ANY .`) với địa chỉ IP nguồn giả mạo nhằm khuếch đại băng thông tấn công từ chối dịch vụ hướng vào nạn nhân khác.
* **Cơ chế phòng thủ kích hoạt:**
  * **Source Port Randomization (SPR):** Mở rộng dải cổng nguồn ngẫu nhiên `net.ipv4.ip_local_port_range = 32768 60999` (tổng cộng 28,231 cổng). Khi kết hợp không gian cổng này với Transaction ID 16-bit, không gian entropy mở rộng lên xấp xỉ $2^{31}$ giá trị khả dĩ, triệt tiêu tính khả thi về mặt xác suất của việc đoán cổng.
  * **Tắt dịch vụ DNS Server công khai:** Target hoàn toàn không mở cổng 53/udp ra mạng ngoài, chỉ sử dụng DNS Resolver nội bộ cô lập (`127.0.0.11`).
* **Kết quả đo kiểm thực tế:**
  * Cổng 53/udp đóng hoàn toàn đối với các máy trạm ngoài mạng. Các truy vấn outbound từ Target duy trì độ ngẫu nhiên cổng nguồn cao và kết nối phân giải an toàn.
  * **Minh chứng nghiệm thu:** `volumes/reports/lab03_verify.log` (`LAB03_VERIFY=PASS`).

---

### 3. Tấn công từ chối dịch vụ tầng Giao vận (Layer 4 - TCP SYN Flood & ICMP Flood)

* **Kỹ thuật tấn công bị chặn:**
  * **TCP SYN Flood:** Kẻ tấn công phát động bão gói tin TCP SYN tốc độ cao (`hping3 --flood --rand-source`) với địa chỉ IP nguồn giả lập ngẫu nhiên nhằm làm cạn kiệt hàng đợi kết nối nửa mở (SYN Backlog Queue Exhaustion), khiến máy chủ treo cứng và từ chối các kết nối hợp lệ.
  * **ICMP Ping Flood / Smurf Attack:** Bão gói ICMP Echo Request hoặc ping broadcast nhằm gây nghẽn băng thông đường truyền và chiếm dụng tài nguyên ngắt của CPU.
* **Cơ chế phòng thủ kích hoạt:**
  * **TCP SYN Cookies:** Kích hoạt `net.ipv4.tcp_syncookies = 1` và mở rộng `net.ipv4.tcp_max_syn_backlog = 2048`. Kernel mã hóa trạng thái kết nối trực tiếp vào trường `Sequence Number` của gói `SYN-ACK`, cho phép tiếp nhận kết nối mà không cần phân bổ ô nhớ lưu trữ trước khi hoàn tất bắt tay 3 bước.
  * **Tường lửa Rate Limiting (UFW / iptables):** Áp dụng quy tắc kiểm soát lưu lượng nghiêm ngặt: giới hạn tối đa 10 gói TCP SYN/giây (burst 20) và 5 gói ICMP/giây (burst 10).
  * **Ignore Broadcast:** Kích hoạt `net.ipv4.icmp_echo_ignore_broadcasts = 1` chặn đứng hoàn toàn nguy cơ Smurf Attack.
* **Kết quả đo kiểm thực tế:**
  * Trong suốt thời gian diễn ra đợt bão gói SYN Flood cường độ cao, **3/3 kết nối kiểm tra hợp lệ** từ trạm Assessor đều thành công thông suốt, dịch vụ duy trì tính sẵn sàng 100%.
  * **Minh chứng nghiệm thu:** `volumes/reports/lab04_verify.log` (`LAB04_VERIFY=PASS`).

---

### 4. Tấn công dò quét và bẻ khóa dịch vụ quản trị từ xa (Host - SSH Reconnaissance & Brute-Force)

* **Kỹ thuật tấn công bị chặn:**
  * **Tự động quét cổng (Port Reconnaissance):** Botnet và công cụ rà quét cổng tiêu chuẩn 22/tcp để nhận diện phiên bản OpenSSH và tìm kiếm CVE đã biết.
  * **SSH Password Brute-force / Dictionary Spraying:** Sử dụng Hydra, Medusa hoặc script tự động thử hàng nghìn tổ hợp tài khoản và mật khẩu thông dụng.
  * **Leo thang chiếm quyền Root trực tiếp:** Nỗ lực đăng nhập thẳng vào tài khoản quản trị tối cao `root`.
* **Cơ chế phòng thủ kích hoạt:**
  * **Thay đổi cổng dịch vụ & Tường lửa lọc cổng:** Chuyển cổng dịch vụ SSH sang **2222/tcp**. Tường lửa UFW thiết lập chính sách `default deny incoming`, chặn lọc triệt để cổng 22 cũ (`filtered / timed out`).
  * **Xác thực khóa mã hóa phi đối xứng (Key-only Authentication):** Thiết lập `PasswordAuthentication no` và `PubkeyAuthentication yes`. Hệ thống chỉ cho phép xác thực bằng cặp khóa ED25519 được phân quyền an toàn `chmod 600`.
  * **Cấm Root & Danh sách trắng người dùng:** Cấu hình `PermitRootLogin no` và giới hạn tài khoản truy cập `AllowUsers labuser`.
  * **Phòng thủ chủ động Fail2ban:** Giám sát thời gian thực nhật ký xác thực `/var/log/auth.log`. Khi phát hiện quá 3 lần xác thực thất bại (`maxretry = 3`), tự động kích hoạt quy tắc iptables khóa cứng địa chỉ IP của kẻ tấn công trong 10 phút (`bantime = 10m`).
* **Kết quả đo kiểm thực tế:**
  * Cổng 22 bị chặn hoàn toàn. Cổng 2222 từ chối ngay lập tức mọi nỗ lực đăng nhập bằng mật khẩu hoặc tài khoản root.
  * Trạm Assessor sau khi thử nghiệm 5 lần gửi mật khẩu sai đã bị Fail2ban đưa vào danh sách đen (`Banned IP list: 10.2.99.20`).
  * **Minh chứng nghiệm thu:** `volumes/reports/validate_after.txt` và `volumes/reports/fail2ban_verify.log` (`FAIL2BAN_VERIFY=PASS`).

---

### 5. Tấn công khai thác tầng Ứng dụng Web (Layer 7 - OWASP Top 10 Injection)

* **Kỹ thuật tấn công bị chặn:**
  * **SQL Injection (SQLi):** Chèn payload logic hằng đúng Tautology (`admin' OR '1'='1`) hoặc mệnh đề `UNION SELECT` vào tham số người dùng nhằm vượt qua kiểm tra đăng nhập hoặc trích xuất toàn bộ dữ liệu nhạy cảm trong cơ sở dữ liệu.
  * **OS Command Injection (RCE):** Chèn các ký tự điều khiển shell (`;`, `&&`, `|`, `` ` ``) vào tham số hệ thống (chẳng hạn tính năng kiểm tra mạng ping) để ép máy chủ thực thi các lệnh Linux độc hại như `whoami`, `cat /etc/shadow` hoặc kích hoạt Reverse Shell.
* **Cơ chế phòng thủ kích hoạt:**
  * **Parameterized Prepared Statements:** Sử dụng cơ chế truy vấn tham số hóa trong SQLite/Python. Cú pháp câu lệnh SQL được biên dịch trước; mọi chuỗi ký tự do người dùng nhập vào đều được xử lý thuần túy dưới dạng biến dữ liệu (literal value), triệt tiêu hoàn toàn khả năng can thiệp vào cấu trúc cú pháp câu lệnh.
  * **Whitelist Regex Validation & Safe Subprocess:** Áp dụng biểu thức chính quy danh sách trắng nghiêm ngặt (`^[0-9a-zA-Z.-]+$`) từ chối ngay lập tức mọi ký tự siêu lệnh shell; thực thi lệnh hệ thống qua hàm API gọi tiến trình trực tiếp với cờ `shell=False`.
* **Kết quả đo kiểm thực tế:**
  * Thử nghiệm tấn công SQLi đối chứng: Endpoint chưa phòng thủ `/sqli_vuln` bị lộ toàn bộ dữ liệu, trong khi endpoint phòng thủ `/sqli_safe` trả về `count: 0` an toàn tuyệt đối.
  * Thử nghiệm tấn công Command Injection đối chứng: Endpoint `/cmd_vuln` bị thực thi mã từ xa, trong khi endpoint `/cmd_safe` phát hiện ký tự đặc biệt và trả về mã lỗi HTTP `400 Bad Request` (`blocked`).
  * **Minh chứng nghiệm thu:** `volumes/reports/lab05_verify.log` (`LAB05_VERIFY=PASS`).

---

### 6. Hành vi can thiệp âm thầm tệp tin hệ thống (Host Tampering & Forensic Evasion)

* **Kỹ thuật tấn công bị chặn & bắt quả tang:** Kẻ tấn công sau khi xâm nhập vào máy chủ tìm cách can thiệp tệp tin nhạy cảm (như chỉnh sửa `/etc/passwd` để tạo tài khoản ẩn, hoặc sửa `/etc/ssh/sshd_config` để mở lại quyền root) và cố tình xóa nhật ký để phi tang dấu vết.
* **Cơ chế phòng thủ kích hoạt:**
  * **Linux Kernel Auditd (HIDS):** Subsystem kiểm toán nhân Linux nạp các quy tắc giám sát mức gọi hệ thống (System Calls) đối với các tệp tin trọng yếu:
    * `auditctl -w /etc/passwd -p wa -k identity_tampering`
    * `auditctl -w /etc/ssh/sshd_config -p wa -k ssh_config_tampering`
* **Kết quả đo kiểm thực tế:**
  * Bất kỳ hành vi mở tệp tin để ghi (`openat` syscall 257) hoặc thay đổi thuộc tính (`chmod`, `chown`) đều bị nhân Linux ghi lại lập tức vào `/var/log/audit/audit.log`.
  * Truy vấn bằng lệnh `ausearch -k identity_tampering` trích xuất đầy đủ hồ sơ pháp lý số không thể chối cãi: tên tiến trình (`touch`), mã tiến trình, thời gian chính xác, và định danh người dùng thực hiện (`UID="root"`).
  * **Minh chứng nghiệm thu:** `volumes/reports/auditd_verify.log` (`AUDITD_VERIFY=PASS`).

---

### 7. Tấn công khai thác bộ nhớ nhị phân (Binary Exploitation - Buffer Overflow / ROP)

* **Kỹ thuật tấn công bị chặn:** Các cuộc tấn công khai thác lỗi tràn bộ đệm (Buffer Overflow), tiêm Shellcode hoặc kỹ thuật xâu chuỗi tiện ích trả về (Return-Oriented Programming - ROP) dựa vào các địa chỉ ô nhớ tĩnh của các hàm thư viện chuẩn (như `system()`, `/bin/sh` trong `libc.so`).
* **Cơ chế phòng thủ kích hoạt:**
  * **Full ASLR (Address Space Layout Randomization):** Kích hoạt chế độ ngẫu nhiên hóa không gian địa chỉ tối đa `kernel.randomize_va_space = 2`. Nhân Linux tự động xáo trộn ngẫu nhiên vị trí vùng nhớ Stack, Heap, bộ nhớ chia sẻ và các thư viện liên kết động tại mỗi lần tiến trình được khởi chạy.
* **Kết quả đo kiểm thực tế:**
  * Địa chỉ các hàm hệ thống thay đổi liên tục qua từng phiên thực thi, triệt tiêu hoàn toàn tính khả thi của các mã khai thác sử dụng địa chỉ nhớ cố định (Hardcoded Offsets).

---

## 📊 Bảng đối chiếu tổng hợp: Thủ thuật tấn công vs Cơ chế phòng thủ

|     STT     | Phân tầng kiểm soát           | Thủ thuật tấn công bị triệt tiêu        | Cơ chế phòng ngự cốt lõi                       |          Hiệu quả nghiệm thu thực tế          |
| :---------: | :-------------------------------- | :--------------------------------------------- | :--------------------------------------------------- | :-------------------------------------------------: |
| **1** | **Layer 2 (Data Link)**     | ARP Cache Poisoning, MITM Sniffing             | Static ARP (`nud permanent`), `rp_filter=1`      | **Chặn đứng 100% (Bảng ARP cố định)** |
| **2** | **Layer 3/4 (Network/DNS)** | IP Spoofing, DNS Kaminsky, Amplification       | Reverse Path Filter, SPR (28k ports), đóng port 53 |  **Chặn đứng 100% (Entropy $2^{31}$)**  |
| **3** | **Layer 4 (Transport)**     | DoS TCP SYN Flood, ICMP Ping Flood             | `tcp_syncookies=1`, UFW SYN/ICMP Rate Limit        | **Dịch vụ thông suốt (3/3 kết nối OK)** |
| **4** | **Host Remote Access**      | SSH Port Scanning, Password Brute-force        | Port 2222, Pubkey ED25519 key-only, Fail2ban         |  **Tự động cấm IP vi phạm sau 3 lần**  |
| **5** | **Layer 7 (Application)**   | SQL Injection (SQLi), Command Injection (RCE)  | Parameterized Prepared Statements, Whitelist Regex   |    **Vô hiệu hóa hoàn toàn payload**    |
| **6** | **Host Integrity (HIDS)**   | Can thiệp tệp `/etc/passwd`, `/etc/ssh/` | Linux Kernel Auditd (`auditctl -w -p wa`)          |  **Trích xuất bằng chứng số chi tiết**  |
| **7** | **Kernel Memory**           | Buffer Overflow, Return-to-libc, ROP chains    | ASLR toàn phần (`randomize_va_space=2`)          | **Triệt tiêu offset tĩnh trong bộ nhớ** |

---

## Danh mục các tệp nhật ký minh chứng (`volumes/reports/`)

Toàn bộ kết quả đối chứng và kiểm thử thực tế được ghi nhận độc lập tại các tệp sau:

1. `lab02_verify.log` & `lab02_verify_summary.txt`: Minh chứng Static ARP kháng đầu độc ARP Spoofing.
2. `lab03_verify.log` & `lab03_verify_summary.txt`: Minh chứng Source Port Randomization kháng tấn công DNS Kaminsky.
3. `lab04_verify.log` & `lab04_verify_summary.txt`: Minh chứng TCP SYN Cookies và Rate Limiting kháng DoS SYN Flood.
4. `lab05_verify.log` & `lab05_verify_summary.txt`: Minh chứng Prepared Statements và Whitelist Regex kháng SQLi & RCE.
5. `fail2ban_verify.log` & `fail2ban_verify_summary.txt`: Minh chứng Fail2ban tự động khóa IP tấn công dò quét SSH.
6. `auditd_verify.log` & `auditd_verify_summary.txt`: Bằng chứng số trích xuất từ Kernel Auditd khi có hành vi sửa file mật.
7. `validate_after.txt`: Minh chứng 6 bài kiểm tra độc lập xác nhận tường lửa và SSH key-only hoạt động hiệu quả.
8. `project_summary.md`: Ma trận nghiệm thu toàn diện đạt 100% PASS trên tất cả các phân lớp phòng thủ.
