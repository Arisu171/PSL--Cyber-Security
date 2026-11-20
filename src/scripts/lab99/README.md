# TÀI LIỆU CHI TIẾT QUY TRÌNH THỰC NGHIỆM VÀ ĐÁNH GIÁ NĂNG LỰC PHÒNG THỦ – LAB 99

## Đề tài: Nghiên cứu, Thiết kế và Đánh giá Năng lực Hệ thống Phòng thủ theo Chiều sâu (Defense-in-Depth) kết hợp Gia cố Hệ điều hành Linux

---

## I. TỔNG QUAN KIẾN TRÚC MÔ HÌNH THỰC NGHIỆM

Hệ thống thực nghiệm Lab 99 được thiết kế theo mô hình **Đánh giá Độc lập (Red Team / Blue Team Verification)** hoạt động trên một vùng mạng ảo hóa cô lập hoàn toàn (`10.2.99.0/24 - lab99-net`), bao gồm 2 thực thể nút mạng chuyên biệt:

1. **Máy chủ Mục tiêu (Target Node - `lab99_target` / `10.2.99.10`):**
   * Hệ điều hành: Ubuntu 22.04 LTS.
   * Dịch vụ: OpenSSH Server, ứng dụng Web nghiệp vụ (`lab99_app.py` trên cổng 8080/tcp), cơ sở dữ liệu SQLite.
   * Vai trò: Đối tượng thụ hưởng toàn bộ quy trình gia cố an ninh đa tầng (OS Hardening) và chịu sự kiểm tra, thẩm định khắt khe từ trạm Assessor.
2. **Nút Thẩm định An ninh Độc lập (Assessor Node - `lab99_assessor` / `10.2.99.20`):**
   * Hệ điều hành: Ubuntu 22.04 LTS trang bị bộ công cụ bảo mật chuyên sâu (`nmap`, `hping3`, `arping`, `curl`, `netcat`, `tcpdump`, `python3`).
   * Vai trò: Máy kiểm thử xâm nhập độc lập trong cùng dải mạng, phát động các kịch bản tấn công thực nghiệm từ Layer 2 đến Layer 7 để đo kiểm và nghiệm thu năng lực tự vệ của máy chủ Target.

---

## II. CHI TIẾT CÁC BƯỚC TRIỂN KHAI, MỤC ĐÍCH VÀ KẾT QUẢ ĐO KIỂM

Toàn bộ quy trình thực nghiệm Lab 99 gồm **16 bước tuần tự**, được chia thành **5 giai đoạn logic chặt chẽ**:

```
Giai đoạn 1: Khởi tạo & Đánh giá đường cơ sở (00 -> 02)
Giai đoạn 2: Gia cố hệ thống & Tái đo lường (03 -> 07)
Giai đoạn 3: Kiểm chứng phòng thủ chủ động & Giám sát số (08 -> 09)
Giai đoạn 4: Kiểm chứng tự vệ trước các vector tấn công đa tầng (10 -> 13)
Giai đoạn 5: Tổng hợp ma trận nghiệm thu & Giải phóng lab (14 -> 99)
```

---

### GIAI ĐOẠN 1: KHỞI TẠO & ĐÁNH GIÁ ĐƯỜNG CƠ SỞ (BASELINE)

---

### Bước 00: Khởi tạo hệ sinh thái Lab 99
* **Script thực thi:** `.lab/00_init.sh`
* **Lệnh chạy:**
  ```bash
  bash .lab/00_init.sh
  ```
* **1. Mục đích kỹ thuật:**
  * Dựng môi trường container độc lập bằng Docker Compose với cấu hình cấp phát tài nguyên đặc quyền (`privileged`, `cap_add: ALL` cho Target để can thiệp kernel/sysctl/iptables/auditd, và `NET_RAW`, `NET_ADMIN` cho Assessor để phát tán gói tin thô).
  * Kiểm tra tính sẵn sàng của kết nối mạng nội bộ giữa Target và Assessor.
  * Đảm bảo ứng dụng Web nghiệp vụ (`lab99_app.py`) trên cổng 8080 và dịch vụ OpenSSH baseline trên cổng 22 của Target đã sẵn sàng tiếp nhận kiểm thử.
* **2. Quy trình triển khai chi tiết:**
  1. Kiểm tra sự tồn tại của file `docker-compose.yml`.
  2. Tạo thư mục lưu trữ báo cáo `volumes/reports/` và nhật ký `volumes/logs/`.
  3. Thực thi `docker compose build` và `docker compose up -d --force-recreate`.
  4. Chờ 5 giây khởi động (`WAIT_BOOT=5`), kiểm tra trạng thái hoạt động của 2 container bằng `docker ps`.
  5. Assessor gửi 2 gói tin ICMP ping tới Target (`10.2.99.10`), kiểm tra tỷ lệ mất gói `0% packet loss`.
  6. Kiểm tra tiến trình SSH trên Target lắng nghe cổng 22 (`ss -lnt`).
  7. Kiểm tra endpoint kiểm tra sức khỏe Web App qua HTTP `curl -s http://127.0.0.1:8080/health`.
* **3. Kết quả đo kiểm & Tiêu chuẩn nghiệm thu:**
  * Cả 2 container `lab99_target` và `lab99_assessor` đều ở trạng thái `Up`.
  * Ping giữa 2 máy thành công 100%, độ trễ dưới 1ms.
  * Dịch vụ SSH và Web App phản hồi trạng thái `healthy`.
  * **Trạng thái:** `KET QUA KHOI TAO LAB99: PASS` (Exit code: 0).

---

### Bước 01: Kiểm toán hiện trạng hệ thống ban đầu (Baseline Before Hardening)
* **Script thực thi:** `.lab/01_baseline_audit.sh`
* **Lệnh chạy:**
  ```bash
  bash .lab/01_baseline_audit.sh
  ```
* **1. Mục đích kỹ thuật:**
  * Thu thập toàn bộ các thông số cấu hình an ninh của hệ điều hành Target ở trạng thái cài đặt mặc định (chưa gia cố).
  * Đo lường chỉ số an toàn theo tiêu chuẩn kiểm toán Lynis (Hardening Index) và 12 tiêu chí bảo mật nội bộ nhằm thiết lập đường cơ sở (Baseline Metrics) dùng để so sánh đối chứng định lượng sau này.
* **2. Quy trình triển khai chi tiết:**
  1. Gọi script kiểm toán nội bộ `collect_metrics.sh before` bên trong container `lab99_target`.
  2. Đọc cấu hình SSH qua `sshd -T` (cổng dịch vụ, quyền root, xác thực mật khẩu, danh sách người dùng).
  3. Kiểm tra trạng thái tường lửa `ufw status`, dịch vụ phòng thủ chủ động `fail2ban-client status`, và giám sát nhân `auditd`.
  4. Thu thập các tham số nhân qua `sysctl`: kiểm tra đường dẫn ngược (`rp_filter`), chống tràn hàng đợi kết nối (`tcp_syncookies`), ngẫu nhiên hóa bộ nhớ (`randomize_va_space`).
  5. Rà quét danh sách cổng TCP đang mở bằng lệnh `ss -lnt`.
  6. Thực thi phiên kiểm toán Lynis tự động (`lynis audit system --quick --quiet --no-colors`).
  7. Chấm điểm tự động trên thang điểm 12 và xuất dữ liệu ra định dạng JSON và Text.
* **3. Kết quả đo kiểm & Tiêu chuẩn nghiệm thu:**
  * File xuất: `volumes/reports/metrics_before.json` và `metrics_before.txt`.
  * Hệ thống mặc định chỉ đạt **5 / 12 điểm** (41.6%).
  * Ghi nhận 5 rủi ro nghiêm trọng: Cho phép login root (`PermitRootLogin yes`), cho phép xác thực mật khẩu (`PasswordAuthentication yes`), tường lửa tắt (`ufw inactive`), Fail2ban không hoạt động (`fail2ban_sshd=inactive`), tính năng chống giả mạo IP bị tắt (`rp_filter=0`).
  * **Trạng thái:** `KET QUA BASELINE: PASS` (Exit code: 0).

---

### Bước 02: Thăm dò bề mặt tấn công mạng ban đầu
* **Script thực thi:** `.lab/02_assess_surface.sh before`
* **Lệnh chạy:**
  ```bash
  bash .lab/02_assess_surface.sh before
  ```
* **1. Mục đích kỹ thuật:**
  * Đóng vai trò đối tượng trinh sát từ bên ngoài (Assessor), sử dụng các công cụ thăm dò mạng tiêu chuẩn (Nmap, Netcat) để khảo sát bề mặt phơi nhiễm công khai của Target trước khi áp dụng các biện pháp phòng vệ.
* **2. Quy trình triển khai chi tiết:**
  1. Assessor thực thi script `/audit/assess_surface.sh before`.
  2. Đo độ trễ và kiểm tra kết nối bằng ICMP ping.
  3. Thực hiện rà quét Nmap danh mục 50 cổng phổ biến nhất (`nmap -Pn -sT -sV --version-intensity 2 --top-ports 50`).
  4. Thăm dò dịch vụ SSH cổng tiêu chuẩn 22 bằng Netcat và gửi yêu cầu banner handshake.
  5. Thăm dò các phương thức xác thực khả dụng trên cổng 22 (`PreferredAuthentications=none`).
  6. Kiểm tra xem cổng tùy chỉnh 2222 đã tồn tại hay chưa.
* **3. Kết quả đo kiểm & Tiêu chuẩn nghiệm thu:**
  * File xuất: `volumes/reports/surface_before.txt`.
  * Nmap ghi nhận 2 cổng mở công khai: Cổng 22/tcp (`OpenSSH 8.9p1 Ubuntu`) và cổng 8080/tcp (`BaseHTTPServer 0.6 / Python 3.10`).
  * SSH Handshake tiết lộ danh sách phương thức xác thực cho phép: `publickey,password` $\rightarrow$ Cảnh báo nguy cơ cao bị tấn công dò quét từ điển (Brute-force).
  * **Trạng thái:** `KET QUA ASSESS SURFACE (before): PASS` (Exit code: 0).

---

### GIAI ĐOẠN 2: GIA CỐ HỆ THỐNG & TÁI ĐO LƯỜNG (HARDENING & RE-ASSESSMENT)

---

### Bước 03: Triển khai quy trình gia cố hệ thống đa tầng
* **Script thực thi:** `.lab/03_harden.sh`
* **Lệnh chạy:**
  ```bash
  bash .lab/03_harden.sh
  ```
* **1. Mục đích kỹ thuật:**
  * Khắc phục toàn bộ các lỗ hổng và điểm yếu đã phát hiện ở giai đoạn Baseline.
  * Triển khai kiến trúc phòng thủ theo chiều sâu (Defense-in-Depth) bao phủ: Tầng quản trị từ xa (SSH), Tầng lọc gói (UFW), Tầng điều khiển lưu lượng (Rate Limiting), Tầng bảo vệ nhân Linux (Sysctl), Tầng mạng cục bộ (Static ARP), Tầng xác thực (PAM pwquality), và Tầng cập nhật an ninh tự động.
* **2. Quy trình triển khai chi tiết:**
  Script kích hoạt `/hardening/harden.sh` trên Target với 7 phân hệ chính:
  1. **Gia cố OpenSSH:** Tạo drop-in config `/etc/ssh/sshd_config.d/99-lab99-hardening.conf`. Di chuyển SSH sang cổng 2222, tắt đăng nhập tài khoản root (`PermitRootLogin no`), tắt chứng thực mật khẩu (`PasswordAuthentication no`), bật chứng thực bằng khóa mã hóa phi đối xứng (`PubkeyAuthentication yes`), giới hạn danh sách người dùng (`AllowUsers labuser`). Tự động sinh cặp khóa ED25519 (`/volumes/lab99_id_ed25519`) và phân quyền an toàn `chmod 600`.
  2. **Thiết lập Tường lửa UFW & Rate Limiting:** Thiết lập chính sách mặc định `default deny incoming` và `default allow outgoing`. Chỉ mở 2 cổng cần thiết: 2222/tcp (SSH) và 8080/tcp (Web). Bật quy tắc giới hạn tốc độ (Rate Limiting) trên iptables: tối đa 10 gói TCP SYN/giây (burst 20) và 5 gói ICMP/giây (burst 10).
  3. **Kích hoạt Phòng thủ chủ động Fail2ban:** Thiết lập `/etc/fail2ban/jail.local`, kích hoạt jail `sshd` giám sát log `/var/log/auth.log` qua cơ chế polling, ngưỡng vi phạm `maxretry = 3`, thời gian cấm `bantime = 10m`.
  4. **Tối ưu hóa tham số an ninh nhân Linux (Sysctl):** Nạp file `/etc/sysctl.d/99-lab99-hardening.conf`:
     * `net.ipv4.conf.all.rp_filter = 1` (Reverse Path Filtering - chống IP Spoofing).
     * `net.ipv4.tcp_syncookies = 1` (Chống tràn hàng đợi TCP SYN Flood).
     * `net.ipv4.conf.all.arp_ignore = 1` và `arp_announce = 2` (Kiểm soát gói tin ARP).
     * `kernel.randomize_va_space = 2` (ASLR toàn phần chống khai thác tràn bộ đệm).
  5. **Cơ chế phòng thủ ARP tĩnh (Static ARP Defense):** Khóa cứng ánh xạ IP-MAC của Gateway (`10.2.99.254`) và Assessor (`10.2.99.20`) vào bảng ARP với cờ `nud permanent`, triệt tiêu khả năng bị đầu độc bảng MAC.
  6. **Chuẩn hóa độ phức tạp mật khẩu PAM:** Áp dụng `/etc/security/pwquality.conf` yêu cầu mật khẩu tối thiểu 12 ký tự, đủ chữ hoa, chữ thường, số và ký hiệu đặc biệt.
  7. **Cơ chế cập nhật bản vá tự động:** Bật dịch vụ `unattended-upgrades` với lịch trình tải bản vá an ninh định kỳ từ Ubuntu Security.
* **3. Kết quả đo kiểm & Tiêu chuẩn nghiệm thu:**
  * Toàn bộ 7/7 module gia cố đều nạp thành công mà không có lỗi cú pháp.
  * Dịch vụ SSH chuyển sang lắng nghe thành công trên cổng 2222.
  * Tường lửa UFW ở trạng thái `active`.
  * Khóa mã hóa `lab99_id_ed25519` được tạo thành công trong thư mục `volumes/`.
  * **Trạng thái:** `KET QUA HARDEN: PASS` (Exit code: 0).

---

### Bước 04: Xác minh hiệu lực các cấu hình gia cố trên Target
* **Script thực thi:** `.lab/04_verify_harden.sh`
* **Lệnh chạy:**
  ```bash
  bash .lab/04_verify_harden.sh
  ```
* **1. Mục đích kỹ thuật:**
  * Thực hiện kiểm tra đối soát trực tiếp từ bên trong hệ thống Target để chứng minh các chính sách cấu hình drop-in đã thực sự có hiệu lực trên tiến trình thực thi của nhân Linux và các daemon dịch vụ.
* **2. Quy trình triển khai chi tiết:**
  1. Kiểm tra tiến trình OpenSSH daemon qua lệnh biên dịch cấu hình thực tế `sshd -T`:
     * Kiểm tra `port 2222`.
     * Kiểm tra `permitrootlogin no`.
     * Kiểm tra `passwordauthentication no`.
  2. Kiểm tra trạng thái tường lửa qua `ufw status`.
  3. Kiểm tra tiến trình giám sát jail `sshd` của Fail2ban qua `fail2ban-client status sshd`.
  4. Truy vấn các biến nhân Linux đang hoạt động bằng `sysctl`: `rp_filter`, `tcp_syncookies`, `randomize_va_space`.
* **3. Kết quả đo kiểm & Tiêu chuẩn nghiệm thu:**
  * `sshd -T` trả về đúng các giá trị gia cố nghiêm ngặt.
  * Tường lửa UFW hiển thị `Status: active` và danh sách quy tắc cho phép cổng 2222/tcp.
  * Jail `sshd` của Fail2ban hiển thị trạng thái hoạt động bình thường (`Currently failed: 0`, `Currently banned: 0`).
  * Các tham số `rp_filter=1`, `tcp_syncookies=1`, `aslr=2` được xác nhận trên nhân kernel.
  * **Trạng thái:** `KET QUA VERIFY HARDEN: PASS` (Exit code: 0).

---

### Bước 05: Tái kiểm toán chỉ số hệ thống sau gia cố (After Hardening)
* **Script thực thi:** `.lab/05_reassess.sh`
* **Lệnh chạy:**
  ```bash
  bash .lab/05_reassess.sh
  ```
* **1. Mục đích kỹ thuật:**
  * Thu thập lại toàn bộ bộ chỉ số bảo mật của Target sau gia cố, đồng thời cập nhật bản đồ bề mặt tấn công mạng mới để chuẩn bị cho bước so sánh định lượng.
* **2. Quy trình triển khai chi tiết:**
  1. Chạy `collect_metrics.sh after` bên trong Target để tính toán lại điểm an toàn.
  2. Chạy `assess_surface.sh after` từ trạm Assessor để ghi nhận bề mặt phơi nhiễm mới qua mạng.
* **3. Kết quả đo kiểm & Tiêu chuẩn nghiệm thu:**
  * File xuất: `volumes/reports/metrics_after.json`, `metrics_after.txt`, và `surface_after.txt`.
  * Điểm an toàn nội bộ tăng vọt đạt mức tối đa **12 / 12 điểm** (100%).
  * Bề mặt mạng ghi nhận cổng 22/tcp cũ đã bị chặn hoàn toàn (`timed out / filtered`), dịch vụ quản trị chỉ phản hồi trên cổng 2222/tcp.
  * **Trạng thái:** `KET QUA REASSESS: PASS` (Exit code: 0).

---

### Bước 06: Kiểm chứng bề mặt công khai từ Assessor
* **Script thực thi:** `.lab/06_validate_defense.sh after`
* **Lệnh chạy:**
  ```bash
  bash .lab/06_validate_defense.sh after
  ```
* **1. Mục đích kỹ thuật:**
  * Máy kiểm định Assessor đóng vai trò kiểm toán viên độc lập (Third-party Auditor), phát động bộ kiểm tra tự động nhằm xác minh xem các cơ chế phòng thủ có thực sự ngăn chặn được các hành vi rà quét và kết nối trái phép từ bên ngoài mạng hay không.
* **2. Quy trình triển khai chi tiết:**
  Script thực hiện 6 bài kiểm tra độc lập (T1 đến T6):
  * **T1 (Nmap Inventory):** Quét top 100 cổng TCP bằng Nmap (`-Pn -sT -T4`).
  * **T2 (Cổng 22 vs 2222):** Dùng `nc -z` kiểm tra cổng 22 (kỳ vọng: đóng/lọc) và cổng 2222 (kỳ vọng: mở).
  * **T3 (Thăm dò phương thức xác thực SSH):** Kết nối thử cổng 2222 bằng chế độ `BatchMode` không gửi key để kiểm tra danh sách gợi ý xác thực. Kỳ vọng: chỉ chấp nhận `publickey`, từ chối `password`.
  * **T4 (Thử nghiệm đăng nhập tài khoản root):** Kết nối thử với tài khoản `root@10.2.99.10:2222`. Kỳ vọng: bị từ chối ngay lập tức (`Permission denied`).
  * **T5 (Thử nghiệm đăng nhập bằng SSH Key hợp lệ):** Sử dụng khóa ED25519 đã tạo để đăng nhập `labuser@10.2.99.10:2222` và thực thi lệnh kiểm tra `echo KEY_LOGIN_OK`. Kỳ vọng: đăng nhập thành công.
  * **T6 (Rà soát dịch vụ thừa):** Quét kiểm tra các cổng dịch vụ nhạy cảm phổ biến (21-FTP, 80-HTTP, 443-HTTPS, 3306-MySQL, 5432-PostgreSQL). Kỳ vọng: toàn bộ các cổng này đều đóng.
* **3. Kết quả đo kiểm & Tiêu chuẩn nghiệm thu:**
  * File xuất: `volumes/reports/validate_after.txt` và `nmap_after.txt`.
  * Toàn bộ 6 bài kiểm tra đều đạt chuẩn `[PASS]`, không có bài kiểm tra nào bị `[FAIL]`.
  * Xác nhận quản trị viên đăng nhập bằng SSH Key cổng 2222 thành công (`KEY_LOGIN_OK`), trong khi mọi nỗ lực đăng nhập bằng mật khẩu hoặc tài khoản root đều bị chặn đứng.
  * **Trạng thái:** `KET QUA VALIDATE DEFENSE (after): PASS` (Exit code: 0).

---

### Bước 07: So sánh định lượng Before vs After
* **Script thực thi:** `.lab/07_compare_report.sh`
* **Lệnh chạy:**
  ```bash
  bash .lab/07_compare_report.sh
  ```
* **1. Mục đích kỹ thuật:**
  * Đối chiếu tự động giữa tệp chỉ số trước gia cố (`metrics_before.json`) và sau gia cố (`metrics_after.json`) để đưa ra bảng tổng hợp khoa học chứng minh mức độ thu hẹp bề mặt tấn công và cải thiện chỉ số an toàn.
* **2. Quy trình triển khai chi tiết:**
  1. Kiểm tra sự tồn tại của 2 file JSON chỉ số.
  2. Kích hoạt script Python `compare_report.py` để phân tích độ lệch (`delta`) của từng tiêu chí.
  3. Xuất bảng so sánh chi tiết ra định dạng Markdown tại `volumes/reports/compare_report.md`.
* **3. Kết quả đo kiểm & Tiêu chuẩn nghiệm thu:**
  * File xuất: `volumes/reports/compare_report.md`.
  * Điểm số bảo mật Lab 99 tăng từ **5/12** lên **12/12 điểm** (`SCORE_DELTA = +7`).
  * Bảng đối chiếu ghi nhận 100% các tiêu chí rủi ro cao ban đầu đều được chuyển đổi sang trạng thái an toàn tuyệt đối.
  * **Trạng thái:** `KET QUA COMPARE: PASS` (Exit code: 0).

---

### GIAI ĐOẠN 3: CÁC KỊCH BẢN KIỂM CHỨNG TỰ TẠO (CUSTOM DEFENSES)

---

### Bước 08: Kiểm chứng phòng thủ chủ động bằng Fail2ban
* **Script thực thi:** `.lab/08_apply_fail2ban.sh`
* **Lệnh chạy:**
  ```bash
  bash .lab/08_apply_fail2ban.sh
  ```
* **1. Mục đích kỹ thuật:**
  * Chứng minh hệ thống không chỉ chặn lọc tĩnh mà còn sở hữu năng lực **Phản ứng Thích ứng Thời gian thực (Adaptive Active Defense)** trước các cuộc tấn công dò quét mật khẩu (SSH Brute-force / Dictionary Spraying).
* **2. Quy trình triển khai chi tiết:**
  1. Kiểm tra trạng thái jail `sshd` trước thử nghiệm (`Currently banned: 0`).
  2. Assessor đóng vai kẻ tấn công, sử dụng vòng lặp gửi liên tục 5 yêu cầu đăng nhập SSH với mật khẩu sai hướng vào cổng 2222 của Target.
  3. Cơ chế ghi log ghi nhận các sự kiện xác thực thất bại vào `/var/log/auth.log`.
  4. Tiến trình Fail2ban phân tích log qua bộ lọc regex, phát hiện số lần thất bại vượt quá ngưỡng `maxretry = 3`.
  5. Fail2ban kích hoạt cơ chế hành động, chèn quy tắc `iptables -I f2b-sshd 1 -s 10.2.99.20 -j REJECT/DROP`.
  6. Script truy vấn `fail2ban-client status sshd` để xác minh địa chỉ IP của Assessor đã bị đưa vào danh sách đen (`Banned IP list`).
  7. Tự động thực thi lệnh `unban 10.2.99.20` sau khi có bằng chứng để không làm gián đoạn các bài test tiếp theo.
* **3. Kết quả đo kiểm & Tiêu chuẩn nghiệm thu:**
  * File xuất: `volumes/reports/fail2ban_verify.log` và `fail2ban_verify_summary.txt`.
  * Nhật ký Fail2ban xác nhận: `Currently banned: 1`, `Banned IP list: 10.2.99.20`.
  * Chứng minh hệ thống tự động khóa IP tấn công ngay tại tầng mạng mà không cần sự can thiệp thủ công của quản trị viên.
  * **Trạng thái:** `FAIL2BAN_VERIFY=PASS` (Exit code: 0).

---

### Bước 09: Kiểm chứng giám sát truy vết số bằng Linux Kernel Auditd (HIDS)
* **Script thực thi:** `.lab/09_apply_auditd.sh`
* **Lệnh chạy:**
  ```bash
  bash .lab/09_apply_auditd.sh
  ```
* **1. Mục đích kỹ thuật:**
  * Kiểm chứng năng lực phát hiện xâm nhập cấp máy chủ (Host-based IDS) và bảo toàn **Dấu vết Pháp lý Số (Digital Forensics)** khi kẻ tấn công đã vượt qua các lớp ngoài và cố tình can thiệp vào các tệp tin hệ thống nhạy cảm.
* **2. Quy trình triển khai chi tiết:**
  1. Kích hoạt subsystem kiểm toán nhân Linux bằng `auditctl -e 1`.
  2. Nạp quy tắc giám sát ghi và thay đổi thuộc tính (`-p wa`) đối với tệp tin định danh người dùng `/etc/passwd` với nhãn định danh `identity_tampering`, và `/etc/ssh/sshd_config` với nhãn `ssh_config_tampering`.
  3. Kiểm tra danh sách quy tắc đang nạp vào kernel qua `auditctl -l`.
  4. Giả lập hành vi của kẻ tấn công: thực thi thao tác can thiệp thay đổi thuộc tính tệp tin bằng lệnh `touch /etc/passwd`.
  5. Sử dụng công cụ trích xuất chứng cứ số `ausearch -k identity_tampering` truy vấn trực tiếp từ nhật ký `/var/log/audit/audit.log`.
* **3. Kết quả đo kiểm & Tiêu chuẩn nghiệm thu:**
  * File xuất: `volumes/reports/auditd_verify.log` và `auditd_verify_summary.txt`.
  * Nhật ký kiểm toán ghi lại một bản ghi SYSCALL hoàn chỉnh:
    * Mã gọi hệ thống: `syscall=257` (`openat`).
    * Tiến trình thực thi: `comm="touch"`, `exe="/usr/bin/touch"`.
    * Định danh người dùng thực hiện: `UID="root"`.
    * Trạng thái và nhãn thời gian: `success=yes`, kèm timestamp và nhãn `key="identity_tampering"`.
  * Tạo lập bằng chứng số không thể chối bỏ phục vụ công tác điều tra sự cố.
  * **Trạng thái:** `AUDITD_VERIFY=PASS` (Exit code: 0).

---

### GIAI ĐOẠN 4: KIỂM CHỨNG TỰ VỆ TRƯỚC CÁC VECTOR TẤN CÔNG ĐA TẦNG

---

### Bước 10: Kiểm chứng phòng thủ tầng Liên kết dữ liệu - ARP Spoofing
* **Script thực thi:** `.lab/10_apply_lab02.sh`
* **Lệnh chạy:**
  ```bash
  bash .lab/10_apply_lab02.sh
  ```
* **1. Mục đích kỹ thuật:**
  * Đánh giá khả năng bảo vệ tính toàn vẹn của bảng ánh xạ địa chỉ IP-MAC trước kỹ thuật tấn công đầu độc ARP Cache Poisoning (Man-In-The-Middle sniffing) trong mạng cục bộ.
* **2. Quy trình triển khai chi tiết:**
  1. Thu thập địa chỉ MAC thực tế của Target và Assessor qua `/sys/class/net/eth0/address`.
  2. Assessor sử dụng công cụ `arping -A -I eth0 -s 10.2.99.99 10.2.99.10` liên tục gửi các gói tin ARP Reply không yêu cầu (Unsolicited ARP Reply) giả mạo địa chỉ IP của Gateway/nút mạng khác mang địa chỉ MAC của Assessor.
  3. Target tiếp nhận các gói tin và xử lý thông qua cơ chế lọc ngược `rp_filter=1` kết hợp tham số `arp_ignore=1`.
  4. Kiểm tra bảng phân giải địa chỉ lân cận trên Target bằng lệnh `ip neigh show`.
* **3. Kết quả đo kiểm & Tiêu chuẩn nghiệm thu:**
  * File xuất: `volumes/reports/lab02_verify.log` và `lab02_verify_summary.txt`.
  * Bảng ánh xạ MAC trên Target vẫn giữ nguyên bản ghi của Assessor với cờ `PERMANENT`.
  * Các gói tin ARP Reply giả mạo hoàn toàn không thể làm biến đổi bảng định tuyến MAC, loại bỏ triệt để nguy cơ bị chuyển hướng lưu lượng nghe lén.
  * **Trạng thái:** `LAB02_VERIFY=PASS` (Exit code: 0).

---

### Bước 11: Kiểm chứng phòng thủ hạ tầng DNS & Source Port Randomization
* **Script thực thi:** `.lab/11_apply_lab03.sh`
* **Lệnh chạy:**
  ```bash
  bash .lab/11_apply_lab03.sh
  ```
* **1. Mục đích kỹ thuật:**
  * Thẩm định năng lực tự vệ của hạ tầng phân giải tên miền: triệt tiêu nguy cơ bị lợi dụng làm trạm phản xạ khuếch đại (DNS Amplification Reflector) và loại bỏ nguy cơ bị đầu độc bộ nhớ đệm phân giải (Kaminsky DNS Cache Poisoning).
* **2. Quy trình triển khai chi tiết:**
  1. Kiểm tra cấu hình bộ giải tên miền `/etc/resolv.conf` trên Target (sử dụng resolver nội bộ cô lập `127.0.0.11`).
  2. Kiểm tra dải cổng nguồn khả dụng thông qua biến nhân `net.ipv4.ip_local_port_range`.
  3. Tính toán không gian tìm kiếm (Entropy) của dải cổng nguồn ephemeral ports.
  4. Assessor quét thăm dò cổng 53/udp trên Target bằng Netcat để kiểm tra xem Target có mở dịch vụ DNS Server ra ngoài mạng hay không.
  5. Target thực hiện một truy vấn phân giải tên miền outbound thực tế (`example.com`) thông qua Python socket để kiểm tra tính thông suốt và toàn vẹn của kết quả trả về.
* **3. Kết quả đo kiểm & Tiêu chuẩn nghiệm thu:**
  * File xuất: `volumes/reports/lab03_verify.log` và `lab03_verify_summary.txt`.
  * Dải cổng nguồn ngẫu nhiên đạt **28,231 cổng** (từ 32768 đến 60999). Khi kết hợp dải cổng này với trường Transaction ID 16-bit của giao thức DNS, không gian tìm kiếm mở rộng lên xấp xỉ $2^{31}$ khả năng $\rightarrow$ Kẻ tấn công không thể dự đoán trúng số hiệu cổng để tiêm phản hồi giả.
  * Target hoàn toàn đóng cổng 53/udp ra mạng ngoài (miễn nhiễm với DNS Amplification).
  * Phân giải tên miền outbound trả về đúng IP chính thức.
  * **Trạng thái:** `LAB03_VERIFY=PASS` (Exit code: 0).

---

### Bước 12: Kiểm chứng độ sẵn sàng tầng Giao vận - DoS SYN Flood & ICMP Flood
* **Script thực thi:** `.lab/12_apply_lab04.sh`
* **Lệnh chạy:**
  ```bash
  bash .lab/12_apply_lab04.sh
  ```
* **1. Mục đích kỹ thuật:**
  * Kiểm chứng tính sẵn sàng cao (High Availability) của máy chủ khi phải chịu luồng tải tấn công từ chối dịch vụ cực lớn nhằm làm cạn kiệt hàng đợi kết nối nửa mở (SYN Backlog Queue Exhaustion).
* **2. Quy trình triển khai chi tiết:**
  1. Kiểm tra cấu hình nhân kernel trên Target: `net.ipv4.tcp_syncookies = 1`, `net.ipv4.tcp_max_syn_backlog = 2048`, và `net.ipv4.icmp_echo_ignore_broadcasts = 1`.
  2. Đo kết nối baseline tới cổng dịch vụ của Target trước khi phát động tấn công.
  3. Assessor sử dụng công cụ `hping3` phát động một đợt bão gói tin TCP SYN Flood tốc độ cao với địa chỉ IP nguồn ngẫu nhiên (`hping3 -S -p 2222 --flood --rand-source 10.2.99.10`) duy trì liên tục trong 4 giây.
  4. **Trong khi đợt flood đang diễn ra ở cường độ tối đa**, Assessor đồng thời phát đi 3 kết nối hợp lệ độc lập bằng Netcat tới Target để kiểm tra khả năng phục vụ kết nối của máy chủ.
  5. Tắt tiến trình `hping3`, tiếp tục phát động đợt bão ICMP Ping Flood (`hping3 --icmp --flood`) và kiểm tra tính đáp ứng phản hồi ICMP sau flood.
* **3. Kết quả đo kiểm & Tiêu chuẩn nghiệm thu:**
  * File xuất: `volumes/reports/lab04_verify.log` và `lab04_verify_summary.txt`.
  * Kết quả đo lường: **3 / 3 lần kết nối hợp lệ đều thành công thông suốt** trong tâm bão SYN Flood.
  * Cơ chế `TCP SYN Cookies` mã hóa thông tin trạng thái trực tiếp vào Sequence Number của gói SYN-ACK mà không cần cấp phát bộ nhớ hàng đợi, kết hợp tường lửa Rate Limiting giúp máy chủ duy trì khả năng đáp ứng dịch vụ 100%.
  * **Trạng thái:** `LAB04_VERIFY=PASS` (Exit code: 0).

---

### Bước 13: Kiểm chứng an toàn tầng Ứng dụng - Web AppSec (SQLi & RCE)
* **Script thực thi:** `.lab/13_apply_lab05.sh`
* **Lệnh chạy:**
  ```bash
  bash .lab/13_apply_lab05.sh
  ```
* **1. Mục đích kỹ thuật:**
  * Thực nghiệm đối chứng trực tiếp giữa phương pháp lập trình truyền thống (dễ bị tổn thương) và kỹ thuật lập trình an toàn (Secure Coding) trước 2 lỗ hổng nghiêm trọng nhất thuộc OWASP Top 10: Tiêm mã truy vấn SQL (SQL Injection) và Thực thi lệnh hệ điều hành từ xa (OS Command Injection).
* **2. Quy trình triển khai chi tiết:**
  Kiểm thử được chia làm 2 phần đối chứng trên ứng dụng `lab99_app.py`:
  * **Phần 1: Kiểm thử SQL Injection:**
    1. Assessor gửi payload logic hằng đúng Tautology Bypass (`admin' OR '1'='1`) tới endpoint chưa bảo vệ `/sqli_vuln` (nối chuỗi trực tiếp).
    2. Assessor gửi cùng payload đó tới endpoint đã bảo vệ `/sqli_safe` (sử dụng truy vấn tham số hóa Parameterized Prepared Statements: `SELECT ... WHERE username = ?`).
  * **Phần 2: Kiểm thử OS Command Injection (RCE):**
    3. Assessor chèn ký tự điều khiển lệnh shell (`127.0.0.1; whoami`) tới endpoint chưa bảo vệ `/cmd_vuln` (sử dụng `os.popen`).
    4. Assessor gửi payload đó tới endpoint đã bảo vệ `/cmd_safe` (sử dụng biểu thức chính quy kiểm tra danh sách trắng `Whitelist Regex: ^[0-9a-zA-Z.-]+$` kết hợp gọi tiến trình con với `shell=False`).
* **3. Kết quả đo kiểm & Tiêu chuẩn nghiệm thu:**
  * File xuất: `volumes/reports/lab05_verify.log` và `lab05_verify_summary.txt`.
  * **Tại `/sqli_vuln`:** Khai thác thành công, trích xuất toàn bộ dữ liệu 3 tài khoản trong cơ sở dữ liệu.
  * **Tại `/sqli_safe`:** Prepared Statements biên dịch trước cú pháp SQL; chuỗi payload được nhân SQLite xử lý thuần túy như một giá trị văn bản $\rightarrow$ Trả về `count: 0`, bảo vệ dữ liệu an toàn tuyệt đối.
  * **Tại `/cmd_vuln`:** Khai thác RCE thành công, máy chủ thực thi lệnh và trả về kết quả `root`.
  * **Tại `/cmd_safe`:** Biểu thức chính quy phát hiện ký tự chấm phẩy độc hại, lập tức từ chối và phản hồi HTTP `400 Bad Request` (`blocked`), bảo vệ máy chủ an toàn.
  * **Trạng thái:** `LAB05_VERIFY=PASS` (Exit code: 0).

---

### GIAI ĐOẠN 5: TỔNG HỢP TOÀN DIỆN & GIẢI PHÓNG MÔI TRƯỜNG

---

### Bước 14: Tổng hợp ma trận nghiệm thu toàn diện dự án
* **Script thực thi:** `.lab/14_summary_project.sh`
* **Lệnh chạy:**
  ```bash
  bash .lab/14_summary_project.sh
  ```
* **1. Mục đích kỹ thuật:**
  * Tự động quét toàn bộ các tệp tóm tắt kết quả (`*_summary.txt`) và báo cáo so sánh định lượng được sinh ra từ các bước kiểm chứng độc lập trước đó, tổng hợp thành một Ma trận Nghiệm thu Khoa học (Defense Matrix) duy nhất lưu tại `project_summary.md`.
* **2. Quy trình triển khai chi tiết:**
  1. Trích xuất giá trị kết quả từ 6 tệp tóm tắt: `fail2ban_verify_summary.txt`, `auditd_verify_summary.txt`, `lab02_verify_summary.txt`, `lab03_verify_summary.txt`, `lab04_verify_summary.txt`, `lab05_verify_summary.txt`.
  2. Kiểm tra sự tồn tại của `compare_report.md` và `metrics_after.json`.
  3. Khởi tạo tài liệu Markdown tổng kết đề tài gồm 3 phần: Thông tin hạ tầng kiểm thử, Ma trận kiểm chứng 7 phân lớp phòng thủ, và Danh mục file bằng chứng kỹ thuật.
  4. Duyệt vòng lặp kiểm tra tính toàn vẹn: nếu tất cả các phân lớp đều đạt giá trị `PASS`, xác nhận toàn bộ dự án hoàn thành xuất sắc.
* **3. Kết quả đo kiểm & Tiêu chuẩn nghiệm thu:**
  * File xuất: `volumes/reports/project_summary.md`.
  * Ma trận tổng kết ghi nhận **100% các tầng phòng thủ đạt trạng thái PASS**:
    * Host Defense (Fail2ban): `PASS`
    * Host Auditing (Auditd HIDS): `PASS`
    * Layer 2/3 (ARP Spoofing Mitigation): `PASS`
    * Layer 3/4 (DNS SPR Mitigation): `PASS`
    * Layer 4 (Transport DoS Mitigation): `PASS`
    * Layer 7 (AppSec SQLi & RCE Mitigation): `PASS`
    * OS Hardening (Lynis / CIS Benchmark): `CO_COMPARE_REPORT` (12/12 điểm)
  * **Trạng thái:** `KET QUA TOAN BO DU AN LAB 99: PASS (100% cac tang phong thu deu duoc xac minh)` (Exit code: 0).

---

### Bước 99: Đóng và giải phóng môi trường Lab 99
* **Script thực thi:** `.lab/99_down.sh`
* **Lệnh chạy:**
  ```bash
  bash .lab/99_down.sh
  ```
* **1. Mục đích kỹ thuật:**
  * Dọn dẹp toàn bộ các tiến trình container, giải phóng dải mạng ảo và bộ nhớ hệ thống sau khi đã thu thập đầy đủ minh chứng thực nghiệm, đảm bảo tính sạch sẽ và không gây xung đột tài nguyên cho các phiên làm việc tiếp theo.
* **2. Quy trình triển khai chi tiết:**
  1. Thực thi `docker compose down`.
  2. Cưỡng chế gỡ bỏ các container liên quan (`lab99_target`, `lab99_assessor`).
  3. Xóa mạng ảo `lab99-net`.
  4. Kiểm tra lại danh sách container đang chạy và danh sách network Docker để đảm bảo tài nguyên đã được giải phóng hoàn toàn.
* **3. Kết quả đo kiểm & Tiêu chuẩn nghiệm thu:**
  * Lệnh `docker ps` xác nhận không còn container nào của Lab 99 đang chạy.
  * Lệnh `docker network inspect lab99-net` trả về lỗi không tồn tại mạng.
  * Toàn bộ tệp nhật ký minh chứng trong `volumes/reports/` vẫn được bảo toàn nguyên vẹn trên máy host phục vụ việc viết báo cáo và làm minh chứng.
  * **Trạng thái:** `KET QUA DONG LAB99: PASS` (Exit code: 0).

---

## III. BẢNG TỔNG HỢP MA TRẬN 16 BƯỚC THỰC NGHIỆM

| STT | Script thực thi | Phân đoạn kiến trúc | Mục đích chính | Kết quả nghiệm thu |
|:---:|:---|:---|:---|:---:|
| **00** | `00_init.sh` | Hạ tầng & Mạng | Dựng 2 container trên subnet 10.2.99.0/24 | Container UP, Ping OK |
| **01** | `01_baseline_audit.sh` | Kiểm toán Baseline | Đo lường chỉ số an toàn ban đầu trước gia cố | 5/12 tiêu chuẩn, phát hiện rủi ro |
| **02** | `02_assess_surface.sh before` | Trinh sát mạng | Assessor quét Nmap thăm dò cổng và dịch vụ ban đầu | Cổng 22, 8080 mở công khai |
| **03** | `03_harden.sh` | Gia cố đa tầng | Nạp cấu hình SSH 2222, UFW, Sysctl, Static ARP, PAM | 7/7 modules PASS |
| **04** | `04_verify_harden.sh` | Xác minh cấu hình | Kiểm tra trực tiếp trên Target tính hiệu lực các drop-in | sshd -T, ufw, sysctl chuẩn xác |
| **05** | `05_reassess.sh` | Tái đo lường | Thu thập lại chỉ số an toàn sau khi đã gia cố | Đạt 12/12 điểm tối đa (100%) |
| **06** | `06_validate_defense.sh after`| Kiểm chứng bề mặt | Assessor quét kiểm tra tính chặn lọc của tường lửa | Cổng 22 bị lọc, SSH key-only OK |
| **07** | `07_compare_report.sh` | Đối chiếu định lượng | So sánh tự động Before vs After (compare_report.md) | Điểm tăng +7 (5 $\rightarrow$ 12/12) |
| **08** | `08_apply_fail2ban.sh` | Phòng thủ chủ động | Giả lập SSH Brute-force, kiểm tra tự động Ban IP | IP Assessor bị đưa vào Banned list |
| **09** | `09_apply_auditd.sh` | Giám sát truy vết | Can thiệp /etc/passwd, kiểm tra trích xuất bằng chứng số | Auditd ghi nhận SYSCALL & UID root |
| **10** | `10_apply_lab02.sh` | Phòng thủ Layer 2/3 | Tấn công ARP Reply Spoofing, kiểm tra bảng ARP | Bảng ARP giữ nguyên cờ PERMANENT |
| **11** | `11_apply_lab03.sh` | Phòng thủ DNS | Kiểm tra dải cổng ephemeral SPR và cổng 53/udp | Dải cổng 28,231 ports, cổng 53 đóng |
| **12** | `12_apply_lab04.sh` | Sẵn sàng dịch vụ DoS | Bão gói SYN Flood (hping3), kiểm tra khả năng phục vụ | 3/3 kết nối hợp lệ thành công |
| **13** | `13_apply_lab05.sh` | An toàn Web AppSec | Tấn công đối chứng SQLi và OS Command Injection | Prepared Statements & Regex chặn đứng |
| **14** | `14_summary_project.sh` | Nghiệm thu toàn diện | Tổng hợp số liệu từ các bài test vào ma trận hoàn chỉnh | 100% PASS trên tất cả các phân lớp |
| **99** | `99_down.sh` | Dọn dẹp tài nguyên | Dừng và giải phóng container, network ảo | Môi trường sạch sẽ, lưu giữ logs |

---

## IV. DANH MỤC CÁC TỆP MINH CHỨNG ARTIFACTS (`volumes/reports/`)

Sau khi hoàn tất kịch bản, toàn bộ bằng chứng số và tệp tin báo cáo được lưu trữ tập trung tại thư mục `volumes/reports/`:

1. `compare_report.md`: Báo cáo đối chiếu định lượng Before vs After chứng minh điểm số tăng từ 5/12 lên 12/12 điểm.
2. `surface_before.txt` & `surface_after.txt`: Minh chứng sự thu hẹp bề mặt tấn công mạng từ Assessor.
3. `validate_after.txt` & `nmap_after.txt`: Nhật ký 6 bài kiểm tra độc lập chứng minh tường lửa và SSH key-only hoạt động hiệu quả.
4. `fail2ban_verify.log` & `fail2ban_verify_summary.txt`: Minh chứng Fail2ban tự động phát hiện và khóa IP vi phạm.
5. `auditd_verify.log` & `auditd_verify_summary.txt`: Bằng chứng số chi tiết từ Linux Kernel Auditd ghi nhận hành vi can thiệp tệp tin mật.
6. `lab02_verify.log` & `lab02_verify_summary.txt`: Bằng chứng cơ chế Static ARP bảo vệ bảng ánh xạ trước ARP Spoofing.
7. `lab03_verify.log` & `lab03_verify_summary.txt`: Bằng chứng dải cổng nguồn SPR triệt tiêu nguy cơ tấn công Kaminsky DNS.
8. `lab04_verify.log` & `lab04_verify_summary.txt`: Bằng chứng dịch vụ duy trì phản hồi 100% trong tâm bão TCP SYN Flood.
9. `lab05_verify.log` & `lab05_verify_summary.txt`: Bằng chứng Prepared Statements và Whitelist Regex vô hiệu hóa SQLi và RCE.
10. `project_summary.md`: Ma trận tổng kết nghiệm thu toàn bộ dự án đạt tỷ lệ thành công PASS tuyệt đối trên tất cả các tầng.
