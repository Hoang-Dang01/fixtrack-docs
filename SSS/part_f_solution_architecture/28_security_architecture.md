# 28 Security Architecture

**Trạng thái: Hoàn thành (Finalized - Version 1.0)**

---

## Purpose

Chương này đặc tả chi tiết kiến trúc an ninh (Security Architecture) của hệ thống FixTrack. Tài liệu định nghĩa mô hình đe dọa (Threat Modeling), các giải pháp bảo mật định danh, phân quyền RBAC/ABAC, phòng vệ API, an ninh hạ tầng triển khai trên máy chủ VPS/Docker, mã hóa dữ liệu, và quy trình phản ứng nhanh khi xảy ra sự cố an ninh (Incident Response).

---

## Inputs

*   [05 User Roles](../part_a_business_foundation/05_user_roles.md)
*   [21 API Design](../part_e_backend_design/21_api_design.md)
*   [22 Backend Modules](../part_e_backend_design/22_backend_modules.md)
*   [24 Background Jobs](../part_e_backend_design/24_background_jobs.md)
*   [25 Cache Strategy](../part_e_backend_design/25_cache_strategy.md)
*   [26 System Architecture](./26_system_architecture.md)
*   [27 Component Architecture](./27_component_architecture.md)

---

# Level 1 — Threat Modeling (STRIDE-lite)

Hệ thống FixTrack áp dụng mô hình phân tích mối đe dọa STRIDE để xác định rủi ro và thiết lập các chốt chặn an ninh tương ứng:

| Loại đe dọa (STRIDE) | Mô tả nguy cơ đối với FixTrack (Threat) | Cơ chế kiểm soát an ninh tương ứng (Mitigation) |
| :--- | :--- | :--- |
| **Spoofing (Giả mạo)** | Kẻ xấu giả danh kỹ thuật viên hoặc tài xế để thao tác hệ thống. | Bắt buộc sử dụng chữ ký JWT được ký bằng khóa bí mật mạnh. Xác thực đa yếu tố (MFA) được khuyến nghị áp dụng riêng cho các vai trò đặc quyền (MANAGER, INVENTORY) nhằm tối ưu chi phí triển khai. |
| **Tampering (Tráo đổi)** | Thay đổi thông tin báo hỏng xe hoặc trạng thái xuất kho phụ tùng. | Kiểm tra tính toàn vẹn dữ liệu, phân quyền ABAC và mã hóa truyền tải TLS 1.3. |
| **Repudiation (Chối bỏ)** | Người dùng chối bỏ việc đã duyệt hoặc nhận bàn giao vật tư sửa chữa. | Ghi nhật ký kiểm toán đồng bộ (Critical Sync Audit) ghi nhận đầy đủ chữ ký người dùng và IP. |
| **Information Disclosure** | Rò rỉ thông tin khách hàng, số điện thoại hoặc sơ đồ kho vật tư. | Cấm trả trực tiếp Entity về API; mã hóa dữ liệu nhạy cảm ở dạng Rest (AES-256). |
| **Denial of Service (DoS)** | Gửi dồn dập request API để làm nghẽn Database hoặc spam API xác thực. | Giới hạn tần suất (Rate Limiting/Throttler) cấu hình tại NestJS và lưu trữ trạng thái đếm trên Redis. |
| **Elevation of Privilege** | KTV hoặc tài xế gọi API nội bộ của Quản lý để duyệt vật tư. | Sử dụng NestJS Guards kết hợp kiểm duyệt RBAC và phân quyền theo ngữ cảnh (ABAC). |

---

# Level 2 — Identity & Authentication Security

## 2.1 Mã hóa Mật khẩu (Password Hashing)
*   **Giải pháp ưu tiên (Preferred)**: Sử dụng thuật toán **Argon2id** (thư viện `argon2` npm) - đây là chuẩn mã hóa chống tấn công brute-force phần cứng (GPU/ASIC) và tấn công kênh kề tối ưu nhất hiện nay.
    *   *Cấu hình khuyến nghị (Argon2id parameters)*:
        *   `memoryCost`: 65536 (64MB RAM)
        *   `timeCost`: 3 iterations
        *   `parallelism`: 2 threads
*   **Giải pháp dự phòng (Fallback)**: Thuật toán `bcrypt` với hệ số tải (work factor/cost) tối thiểu là 12.

## 2.2 JWT Lifecycle & Session Management
Hệ thống sử dụng cơ chế Token kép (Dual-token pattern) để đảm bảo trải nghiệm người dùng không bị gián đoạn nhưng vẫn bảo mật:
1.  **Access Token**:
    *   *Thời gian sống (TTL)*: 15 phút.
    *   *Truyền tải*: Đính kèm ở header `Authorization: Bearer <token>`.
2.  **Refresh Token**:
    *   *Thời gian sống (TTL)*: 7 ngày.
    *   *Truyền tải*: Lưu trữ trong cookie phía Client với các cờ cấu hình nghiêm ngặt: **`HttpOnly`** (chống XSS đánh cắp cookie), **`Secure`** (chỉ truyền qua HTTPS), và **`SameSite=Strict`** (ngăn chặn tấn công CSRF).

## 2.3 Refresh Token Rotation & Reuse Detection
Để chống lại việc đánh cắp Refresh Token:
*   Mỗi lần Client gọi API `/auth/refresh` để lấy Access Token mới, hệ thống sẽ **hủy bỏ Refresh Token cũ** và cấp một cặp Access/Refresh Token hoàn toàn mới (Rotation).
*   **Reuse Detection (Phát hiện dùng lại)**: Nếu hệ thống phát hiện một Refresh Token cũ (đã bị thu hồi) được gửi lại lần nữa:
    1.  Hệ thống coi đây là dấu hiệu của hành vi trộm cắp token.
    2.  Lập tức **thu hồi toàn bộ cây gia phả (Token Family)** của phiên làm việc đó (xóa Session Key trên Redis).
    3.  Yêu cầu người dùng thật đăng nhập lại và ghi nhận một cảnh báo bảo mật đặc biệt mức `CRITICAL`.

```
User Refresh Request ──► [Token Used Before?] ─(Yes)─► Revoke Token Family ──► Force Logout & Alert
                               │
                             (No)
                               ▼
                        Rotate Token Setup
```

## 2.4 Emergency Access Token Revocation (Thu hồi Access Token Khẩn cấp)
Mặc định Access Token là stateless (tự xác thực qua chữ ký) và không thể bị thu hồi giữa vòng đời. Tuy nhiên, đối với các sự cố bảo mật mức độ `HIGH`/`CRITICAL` (ví dụ: khóa tài khoản, hạ cấp vai trò, hoặc lộ khóa bí mật), hệ thống áp dụng một trong hai chiến lược xử lý:
1.  **Token Versioning (Giải pháp ưu tiên)**:
    *   Thêm trường `token_version` (Integer) vào bảng `users`.
    *   Đính kèm `token_version` vào JWT payload của Access Token.
    *   Khi có sự kiện thu hồi khẩn cấp $\rightarrow$ Tăng giá trị `token_version` trong CSDL thêm 1 đơn vị. Các request gọi lên sau đó sẽ bị Guard kiểm tra lệch phiên bản với DB/Redis cache và chặn lại lập tức.
2.  **Redis Blacklist (Giải pháp dự phòng)**:
    *   Đưa mã nhận diện `jti` của Access Token bị hủy vào danh sách đen `blacklist:{jti}` trên Redis B (với TTL bằng thời gian sống còn lại của Token đó). Guard sẽ quét blacklist trước khi cho phép đi qua.

---

# Level 3 — Authorization Model

Hệ thống kết hợp hai tầng phân quyền:

## 3.1 Phân quyền Vai trò (RBAC - Role-Based Access Control)
*   Được thực thi thông qua `RolesGuard` toàn cục của NestJS.
*   Xác định quyền truy cập các endpoints dựa trên Enums vai trò đính kèm trong JWT Payload: `DRIVER`, `MECHANIC`, `TECH`, `INVENTORY`, `MANAGER`.

## 3.2 Phân quyền theo Ngữ cảnh (ABAC - Attribute-Based Access Control)
Phân quyền vai trò là chưa đủ. Hệ thống cần áp dụng ABAC để kiểm tra mối liên kết giữa tài nguyên và người dùng thao tác:
*   *Quy tắc TECH*: Kỹ thuật viên (TECH) chỉ được cập nhật tiến độ sửa chữa của các phiếu (`RepairJob`) mà chính họ được phân công nhiệm vụ (`repair_jobs.technician_id === current_user_id`).
*   *Quy tắc DRIVER*: Lái xe (DRIVER) chỉ được xem chi tiết các phiếu sửa chữa của xe do họ làm chủ quản hoặc do họ báo lỗi.

---

# Level 4 — Application & API Hardening

## 4.1 HTTP Security Headers (Helmet Middleware)
Ứng dụng NestJS tích hợp thư viện `helmet` để thiết lập các HTTP headers tiêu chuẩn bảo vệ trình duyệt:
*   `HSTS (HTTP Strict Transport Security)`: Ép trình duyệt chỉ tương tác qua HTTPS.
*   `X-Frame-Options: DENY`: Chống tấn công Clickjacking (ngăn nhúng app vào iframe).
*   `X-Content-Type-Options: nosniff`: Chống MIME-sniffing.
*   `Content-Security-Policy (CSP)`: Hạn chế nguồn tải script/style để chống XSS.

## 4.2 Giới hạn CSRF theo mục tiêu (Targeted CSRF Protection)
*   *Chính sách*: Nếu Bearer Access Token chỉ được lưu trong bộ nhớ Javascript (JS Memory) hoặc bộ nhớ cục bộ (Local Storage) và Client chủ động đính kèm vào header `Authorization`, trình duyệt sẽ không tự động gửi token này khi có các request từ site khác (Cross-Site Requests), do đó rủi ro bị tấn công CSRF là cực kỳ thấp.
*   *Phạm vi bảo vệ*: CSRF chỉ đe dọa các API sử dụng Cookie để tự động truyền tải thông tin định danh. Do đó, hệ thống tập trung triển khai giải pháp phòng vệ CSRF (bằng cờ `SameSite=Strict` kết hợp cơ chế Double Submit Cookie) đặc hiệu tại hai API `/auth/refresh` và `/auth/logout` (nơi sử dụng Cookie chứa Refresh Token).

## 4.3 Giới hạn tần suất gọi API (Rate Limiting & Anti-Bruteforce)
Sử dụng cụm **Redis B** để đếm lượt truy cập thông qua `@nestjs/throttler` theo hai cơ chế song song:
1.  **IP-based Rate Limiting (Giới hạn theo địa chỉ IP)**:
    *   *Auth APIs (Login/Register)*: Tối đa 5 requests / 1 phút.
    *   *Attachment Upload APIs*: Tối đa 10 requests / 1 phút.
    *   *Normal Business APIs*: Tối đa 100 requests / 1 phút.
2.  **Identity-based Rate Limiting (Anti-Bruteforce theo tài khoản)**:
    *   Ngăn chặn kẻ tấn công xoay vòng IP để thử mật khẩu của một tài khoản nhất định.
    *   *Chính sách*: Khóa tạm thời quyền đăng nhập của một tài khoản (`username`/`email`) trong 15 phút nếu phát hiện đăng nhập thất bại quá 10 lần liên tiếp (10 failed attempts / 15 mins).

---

# Level 5 — Infrastructure & Deployment Security (VPS Hardening)

Ở giai đoạn đầu, FixTrack được triển khai trên môi trường **Single VPS chạy Docker Compose**, chưa áp dụng cụm proxy Nginx/Caddy chuyên biệt bên ngoài. Hạ tầng được bảo vệ bằng các giải pháp sau:

```
               Firewall (UFW Rules on VPS Host)
Public Internet ───────► [Port 80/443 Allow] ───────► NestJS Docker Container
                       │
                       ├─► [Port 22 SSH Restricted] ─► SSH Access
                       │
                       └─► [Port 5432 / 6379 Deny]  ─► Blocked & Dropped
```

## 5.1 Cấu hình Firewall trên VPS Host (UFW Rules)
Cấu hình tường lửa mềm (UFW) trên VPS chạy hệ điều hành Ubuntu để chặn đứng các truy cập từ Internet vào các cổng dữ liệu nhạy cảm:
```bash
# Thiết lập mặc định chặn tất cả chiều vào
ufw default deny incoming
ufw default allow outgoing

# Chỉ cho phép các cổng Web công khai và SSH bảo mật
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 22/tcp

# Nghiêm cấm public cổng DB, Redis, và Bull Board ra ngoài Internet
ufw deny 5432/tcp
ufw deny 6379/tcp
ufw deny 3000/tcp # (Cổng HTTP nội bộ của NestJS)

ufw enable
```

## 5.2 Cô lập Mạng Container (Docker Network Isolation)
*   NestJS, PostgreSQL, Redis, và ClamAV được chạy trong cùng một mạng ảo nội bộ (**Docker Bridge Network**).
*   Chỉ expose cổng `80/443` của container NestJS/Web ra ngoài host VPS để định hướng tên miền và TLS.
*   Các container dữ liệu như `postgres` và `redis` tuyệt đối không cấu hình thuộc tính `ports` ánh xạ ra ngoài host VPS (không cấu hình `5432:5432` mà chỉ giao tiếp nội bộ trong mạng ảo Docker).

## 5.3 Giải pháp Tên miền & TLS (Domain & TLS Termination)
*   **Giai đoạn đầu**: Tích hợp dịch vụ **Cloudflare Proxy** làm lá chắn trước VPS. TLS/SSL sẽ được terminate tại Cloudflare bằng chế độ **Cloudflare Full (Strict)** (yêu cầu chứng chỉ SSL của Origin Server phải hợp lệ để tăng tính bảo mật). VPS chỉ nhận request đã được mã hóa được Cloudflare chuyển tiếp về.
*   **Lộ trình mở rộng tương lai**:
    *   *Phase 1*: Domain $\rightarrow$ VPS Host Firewall $\rightarrow$ NestJS API.
    *   *Phase 2*: Domain $\rightarrow$ Reverse Proxy (Caddy / Traefik tự động gia hạn Let's Encrypt) $\rightarrow$ NestJS Cluster.
    *   *Phase 3*: Domain $\rightarrow$ Cloud Load Balancer $\rightarrow$ Multiple VPS nodes.

## 5.4 Quản lý mã khóa (Secrets Management)
*   *Môi trường Dev*: Lưu trữ trong file `.env` cục bộ (có trong `.gitignore`).
*   *Môi trường Production*: Lưu trữ bằng **Docker Secrets** hoặc công cụ quản lý biến môi trường bảo mật của VPS provider, cấm tuyệt đối việc hardcode thông tin kết nối DB hoặc JWT Secret vào mã nguồn Git.

## 5.5 Chính sách Xoay vòng Khóa bí mật (Secret Rotation Policy)
Để hạn chế thiệt hại trong trường hợp rò rỉ mã khóa hoặc thông tin cấu hình, hệ thống quy định chu kỳ xoay vòng khóa định kỳ như sau:
*   **JWT Secret Key**: Định kỳ xoay vòng mỗi 90 ngày.
*   **Database Password (PostgreSQL)**: Định kỳ thay đổi mật khẩu truy cập mỗi 180 ngày.
*   **Redis Password**: Định kỳ thay đổi mật khẩu truy cập mỗi 180 ngày.
*   **Xoay vòng khẩn cấp (Emergency Rotation)**: Thực hiện ngay lập tức bất cứ lúc nào nếu có nghi ngờ hoặc bằng chứng về việc rò rỉ khóa bí mật, thông tin tài khoản quản trị hoặc máy chủ bị xâm nhập.

---

# Level 6 — Data & File Security

## 6.1 Mã hóa dữ liệu (Encryption)
*   **Dữ liệu truyền tải (In-Transit)**: Ép buộc sử dụng giao thức HTTPS và WSS (Secure WebSocket) hỗ trợ tối thiểu TLS 1.2 và tối ưu TLS 1.3.
*   **Dữ liệu lưu trữ (At-Rest)**: Các thông tin nhạy cảm của người dùng (như mật khẩu) bắt buộc băm bằng Argon2id. Các tài liệu kỹ thuật bảo mật hoặc chứng từ hóa đơn sửa chữa nhạy cảm được mã hóa trước khi ghi vào S3 bằng cơ chế mã hóa phía máy chủ (S3 Server-Side Encryption AES-256).

## 6.2 Bảo mật tệp tải lên (File Upload Hardening)
*   **Định dạng cho phép (Whitelisting)**: Chỉ cho phép tải lên hình ảnh (`.jpg`, `.jpeg`, `.png`) và video (`.mp4`, `.mov`). Chặn tuyệt đối tệp tin thực thi (`.exe`, `.sh`, `.bat`, `.js`).
*   **Quy trình xác thực tệp nghiêm ngặt**:
    Để ngăn chặn việc thay đổi phần mở rộng tệp giả mạo (ví dụ: đổi đuôi `.exe` thành `.jpg`), hệ thống thực hiện kiểm tra 3 tầng:
    1.  *Extension check*: Kiểm tra phần mở rộng của tên tệp tin.
    2.  *MIME type verification*: Kiểm tra định dạng dữ liệu truyền tải trong header.
    3.  *Magic byte inspection*: Đọc các byte đầu tiên của tệp tin vật lý để xác thực chính xác cấu trúc tệp tin gốc.
*   **Cô lập mã độc (ClamAV Scan & Quarantine)**: Xem chi tiết quy trình quét virus bất đồng bộ và chuyển sang Quarantine Bucket tại [26 System Architecture](./26_system_architecture.md#42-lu%C3%B4ng-qu%C3%A9t-virus-v%C3%A0-c%C3%A1ch-ly-t%E1%BB%87p-%C4%91%C3%ADnh-k%C3%A8m-file-scan--quarantine-flow).

## 6.3 Bảo mật Cơ sở dữ liệu (Database Security)
Hệ thống dữ liệu của FixTrack được thiết lập các cơ chế phòng vệ nhằm đảm bảo tính toàn vẹn và ngăn chặn khai thác đặc quyền:
*   **Tách biệt Tài khoản Quản trị**: Tài khoản kết nối từ ứng dụng NestJS đến CSDL (`app DB user`) phải là tài khoản có quyền hạn hạn chế, tuyệt đối không dùng tài khoản siêu quản trị (`postgres` superuser).
*   **Quyền hạn Tối thiểu (Least Privilege)**: Tài khoản kết nối chỉ được cấp các quyền cần thiết để đọc/ghi dữ liệu nghiệp vụ (SELECT, INSERT, UPDATE, DELETE) trên schema chỉ định. Quyền thay đổi cấu trúc bảng (DDL) phải được chạy bằng tiến trình Migration được kiểm soát riêng biệt, không cấp cho ứng dụng chạy runtime.
*   **Mã hóa Bản sao lưu (Encrypted Backups)**: Toàn bộ tệp tin sao lưu (Backup files) của CSDL PostgreSQL khi lưu trữ tĩnh (At-Rest) trên S3 hoặc dịch vụ lưu trữ ngoài phải được mã hóa bằng chuẩn AES-256.
*   **Khôi phục Điểm Thời gian (PITR - Point-in-Time Recovery)**: Cấu hình lưu trữ tệp tin nhật ký WAL (Write-Ahead Logging) để hỗ trợ khôi phục CSDL về bất kỳ thời điểm cụ thể nào (lộ trình tích hợp ở các giai đoạn mở rộng quy mô tiếp theo).

---

# Level 7 — Auditing & Incident Response

## 7.1 Phân luồng Nhật ký Kiểm toán & Chống tráo đổi (Audit Logging & Tamper Resistance)
Để tối ưu hóa hiệu năng và bảo vệ dữ liệu kiểm toán, hệ thống áp dụng cơ chế phân luồng và bảo mật mã hóa:
1.  **Critical Sync Audit (Đồng bộ)**: Các hành động gây rủi ro bảo mật hệ thống cao phải được ghi đồng bộ trong DB transaction của request chính:
    *   Đăng nhập thất bại (Brute-force signal).
    *   Thay đổi quyền hạn tài khoản hoặc thay đổi vai trò người dùng (Roles).
    *   Yêu cầu đổi mật khẩu hoặc phát hiện dùng lại Refresh Token cũ.
2.  **Normal Async Audit (Bất đồng bộ)**: Các hành động CRUD nghiệp vụ thông thường (tạo ticket, đổi trạng thái) được ghi bất đồng bộ qua `audit-queue` để giảm tải DB.
3.  **Chính sách chống sửa đổi (Tamper-Proof Policy)**:
    *   Dữ liệu audit logs được thiết lập thuộc tính **Append-only** (chỉ cho phép ghi thêm, cấm lệnh UPDATE và DELETE ở mức DB user).
    *   *Mã hóa chuỗi Hash (Hash-Chaining)*: Để phát hiện nếu quản trị viên hoặc kẻ tấn công can thiệp sửa đổi các dòng log cũ, mỗi dòng log kiểm toán quan trọng khi ghi lại sẽ được băm kết hợp với dòng log trước đó:
        $$\text{Hash}_n = \text{SHA256}(\text{LogContent}_n + \text{Hash}_{n-1})$$
        Nếu bất kỳ dòng log nào bị sửa, toàn bộ chuỗi hash phía sau sẽ bị lệch, giúp phát hiện tráo đổi lập tức trong quá trình đối soát (Forensics).

## 7.2 Quy trình ứng phó Sự cố An ninh (Security Incident Response)
Khi phát hiện vi phạm bảo mật nghiêm trọng (như phát hiện virus, tấn công Brute-force đăng nhập, hoặc trùng lặp Refresh Token):
```
Detection ──► Alert Manager ──► Lock Account ──► Revoke Sessions ──► Preserve Audit Trail
```
1.  **Cảnh báo tức thời (Alert)**: Gửi thông báo khẩn cấp mức `HIGH`/`CRITICAL` tới Slack/Teams của đội kỹ thuật và email quản trị viên.
2.  **Khóa tài khoản (Lock Account)**: Tự động cập nhật trạng thái người dùng thành `LOCKED` nếu sai mật khẩu liên tiếp 10 lần.
3.  **Thu hồi phiên làm việc (Revoke Sessions)**: Xóa toàn bộ Session Keys của tài khoản bị nghi ngờ trên cụm Redis B để buộc đăng xuất trên mọi thiết bị.
4.  **Bảo toàn chứng cứ log (Preserve Trail)**: Đóng băng dữ liệu audit log liên quan để phục vụ công tác rà soát lỗi và điều tra an ninh.
