# 08 Non-Functional Requirements

## Purpose
Chương này đặc tả các yêu cầu phi chức năng (Non-Functional Requirements - NFR) của hệ thống VRMS. Đây là các tiêu chuẩn kỹ thuật về hiệu năng, bảo mật, tính khả dụng ngoại tuyến (offline), khả năng mở rộng, độ tin cậy và trải nghiệm người dùng di động, giúp định hình kiến trúc hạ tầng và các quyết định thiết kế hệ thống.

## Questions Answered
- Hệ thống cần đạt những chỉ số định lượng nào về hiệu năng (thời gian phản hồi API, tải dữ liệu)?
- Làm thế nào để đảm bảo ứng dụng di động vẫn hoạt động được khi tài xế hoặc KTV di chuyển vào vùng sóng yếu trong bãi xe (Offline-first)?
- Các tiêu chuẩn bảo mật dữ liệu, phân quyền, sao lưu (backup) và khôi phục sau sự cố được thiết kế như thế nào?
- Các yêu cầu phi chức năng này ảnh hưởng trực tiếp đến kiến trúc hệ thống (Architecture Impact) như thế nào?

## Inputs
- [02 Business Context](file:///c:/Users/Legion/Desktop/IT/SNP/.agents/skills/system_engineering_copilot/deliverables/current/sss/part_a_business_foundation/02_business_context.md) (Ràng buộc bối cảnh vận hành bãi xe).
- [07 Functional Requirements](file:///c:/Users/Legion/Desktop/IT/SNP/.agents/skills/system_engineering_copilot/deliverables/current/sss/part_b_requirement_analysis/07_functional_requirements.md) (Các luồng chức năng cần tối ưu hiệu năng).

---

## Content

### Level 1 — Summary
Do VRMS là hệ thống mobile-first hoạt động chủ yếu trong môi trường nhà xưởng, bãi đỗ xe và kho vật tư — nơi có thể phát sinh các khu vực sóng di động (3G/4G/Wifi) yếu hoặc chập chờn — hệ thống được thiết kế ưu tiên các tiêu chuẩn chất lượng sau:
1. **Khả năng ngoại tuyến (Offline-first)**: Ứng dụng di động lưu trữ dữ liệu tạm thời cục bộ và cho phép thực hiện các thao tác cơ bản (báo lỗi, nhận vật tư) ngay cả khi mất mạng. Dữ liệu khi đồng bộ lại sẽ được kiểm tra xung đột nghiêm ngặt ở Server.
2. **Hiệu năng cao & Tiết kiệm tài nguyên**: API phản hồi nhanh, tự động nén dung lượng hình ảnh/video hiện trường trước khi upload để tiết kiệm băng thông. Giao diện mượt mà, không gây hao pin/nóng máy.
3. **Bảo mật nghiêm ngặt**: Dữ liệu truyền tải giữa Mobile Client và Server được mã hóa hoàn toàn. Phân quyền dựa trên vai trò (RBAC) được kiểm tra cả ở phía Client (ẩn/hiện nút) và phía Server (chặn API trái phép).
4. **Khả năng mở rộng & Bảo trì**: Thiết kế Backend theo dạng Kiến trúc nguyên khối phân module (Modular Monolith) sẵn sàng tách dịch vụ khi mở rộng quy mô xưởng, đi kèm tài liệu API tự động và tiêu chuẩn kiểm thử tự động cao.
5. **Trải nghiệm thân thiện với môi trường xưởng**: Giao diện nút to, dễ tương tác kể cả khi đeo găng tay bảo hộ. Hỗ trợ chế độ màn hình tối (Dark Mode) cho các ca trực sửa chữa ban đêm.

---

### Level 2 — Breakdown (Phân nhóm yêu cầu phi chức năng)

#### 1. Yêu cầu về Hiệu năng (Performance - PERF)
- **Thời gian phản hồi API**: Các API truy vấn thông thường (danh sách xe, danh sách ticket) phải có thời gian phản hồi dưới `1.5 giây` trong điều kiện mạng ổn định. Các tác vụ ghi dữ liệu (tạo ticket, gán xe) phải phản hồi dưới `2.0 giây`.
- **Tải tệp đa phương tiện**: Hệ thống tự động nén ảnh chụp hiện trường xuống dưới `500KB` và video dưới `5MB` trước khi tải lên máy chủ để đảm bảo thời gian tải lên dưới `5 giây` trên mạng di động thông thường.
- **Tiêu thụ tài nguyên thiết bị**: Ứng dụng di động không chiếm dụng quá `150MB RAM` và không tiêu hao quá `5% dung lượng pin` sau 1 giờ hoạt động liên tục.

#### 2. Yêu cầu về Bảo mật & Kiểm toán (Security & Audit - SEC)
- **Mã hóa đường truyền**: Toàn bộ dữ liệu trao đổi qua API bắt buộc sử dụng giao thức bảo mật `HTTPS/TLS 1.3`.
- **Bảo mật phiên làm việc (Session Security)**: 
  - Token truy cập (Access Token JWT) hết hạn trong vòng `1 giờ` để hạn chế rủi ro lộ lọt.
  - Token làm mới (Refresh Token) hỗ trợ duy trì phiên làm việc tối đa `7 ngày` trên thiết bị tin cậy. Lưu trữ trong vùng nhớ an toàn của thiết bị (iOS Keychain / Android EncryptedSharedPreferences).
- **Phân quyền API (Server-side Enforcement)**: Backend bắt buộc kiểm tra vai trò (Role) của người gửi request cho mỗi API endpoint. Cấm hoàn toàn việc dựa vào giao diện Client để chặn quyền truy cập.
- **Nhật ký kiểm toán (Audit Trail)**: Mọi thao tác làm thay đổi trạng thái xe, trạng thái ticket và xuất kho vật tư phải được ghi log vĩnh viễn vào DB (chứa thông tin: user_id, action, old_value, new_value, timestamp).

#### 3. Yêu cầu về Tính sẵn sàng & Độ tin cậy (Availability & Reliability - REL)
- **Độ sẵn sàng hệ thống**: Dịch vụ API Backend phải đạt tỉ lệ uptime tối thiểu `99.5%` trong một tháng đối với giai đoạn thử nghiệm MVP (tương đương thời gian gián đoạn tối đa ~3.6 giờ/tháng). Định hướng nâng cấp lên `99.9%` cho phiên bản chính thức (Production).
- **Cơ chế tự động thử lại (Retry Mechanism)**: Khi gửi yêu cầu hoặc thông báo push thất bại do lỗi mạng tạm thời, hệ thống di động tự động thử lại tối đa 3 lần với khoảng cách tăng dần (exponential backoff).

#### 4. Yêu cầu về Hoạt động ngoại tuyến & Đồng bộ (Offline & Sync - OFFLINE)
- **Cơ sở dữ liệu cục bộ**: Mobile App sử dụng cơ sở dữ liệu nhúng (ví dụ: SQLite hoặc Realm) để lưu trữ cấu hình tĩnh (danh sách vật tư, danh sách xe) và các ticket nháp.
- **Tạo phiếu offline**: Lái xe vẫn tạo được phiếu báo lỗi khi mất mạng. Phiếu được xếp vào hàng đợi đồng bộ cục bộ (Local Sync Queue).
- **Quy tắc giải quyết xung đột (Conflict Resolution)**:
  - *Dữ liệu tham chiếu tĩnh (Static Reference Data)*: Áp dụng quy tắc ghi đè tự động (Last Write Wins).
  - *Dữ liệu giao dịch (Transactional Data)*: Bắt buộc thực hiện kiểm tra nghiệp vụ và trạng thái phía máy chủ (Server-side conflict validation). Ví dụ: Không cho phép thiết bị offline đẩy trạng thái ticket về `WAITING_PARTS` nếu trên Server trạng thái thực tế của xe đã được KTV cập nhật sang `REPAIRING`.

#### 5. Yêu cầu về Khả năng mở rộng (Scalability - SCALE)
- **Hỗ trợ người dùng đồng thời**: Hệ thống đáp ứng tối thiểu `500 người dùng hoạt động đồng thời` (TBD - có thể điều chỉnh tùy theo quy mô triển khai: 1 xưởng chỉ cần 50-100 concurrent, chuỗi 20+ xưởng cần 500+ concurrent).
- **Dung lượng lưu trữ lịch sử**: Cơ sở dữ liệu đáp ứng lưu trữ và truy vấn mượt mà lịch sử của ít nhất `100.000 phiếu sửa chữa` (repair tickets history).

#### 6. Yêu cầu về Khả năng bảo trì (Maintainability - MAIN)
- **Kiến trúc mã nguồn**: Mã nguồn phía Backend bắt buộc tổ chức theo dạng Kiến trúc nguyên khối phân module (Modular Monolith) để sẵn sàng tách thành microservices khi cần.
- **Độ bao phủ kiểm thử (Test Coverage)**: Mã nguồn hệ thống phải đạt tỉ lệ bao phủ kiểm thử tự động (Unit Test / Integration Test) tối thiểu `80%`.
- **Tài liệu API**: Hệ thống tự động sinh tài liệu API (Swagger / OpenAPI docs) từ mã nguồn để các nhà phát triển dễ dàng tích hợp và nâng cấp.

#### 7. Yêu cầu về Trải nghiệm & Tính khả dụng (Usability - USA)
- **Tạo phiếu nhanh chóng**: Đảm bảo `95% người dùng` có thể hoàn thành việc tạo phiếu báo lỗi trên ứng dụng trong vòng `dưới 2 phút` mà không cần qua đào tạo hoặc hướng dẫn sử dụng.
- **Tương tác trong nhà xưởng**: Cỡ chữ tối thiểu `14sp`, các nút bấm tương tác chính (Báo lỗi, Xác nhận nhận hàng, Bắt đầu sửa) phải có kích thước tối thiểu `48dp x 48dp` để dễ bấm ngay cả khi đeo găng tay bảo hộ.
- **Chế độ ca đêm (Dark Mode)**: Hỗ trợ giao diện tối để giảm mỏi mắt cho KTV và Lái xe làm việc ban đêm dưới ánh sáng yếu của bãi xe.

#### 8. Yêu cầu về Sao lưu & Phục hồi thảm họa (Backup & Recovery - REC)
- **Tần suất sao lưu**: Cơ sở dữ liệu sản xuất (Production Database) phải được sao lưu tự động định kỳ `6 tiếng một lần`.
- **Mất mát dữ liệu tối đa cho phép (RPO - Recovery Point Objective)**: Đảm bảo dữ liệu mất mát khi xảy ra sự cố sập nguồn/hỏng ổ đĩa không vượt quá `30 phút` (RPO <= 30 minutes).

#### 9. Yêu cầu về Lưu trữ dữ liệu & Tuân thủ (Data Retention & Compliance - DATA)
- **Thời gian lưu trữ log kiểm toán**: Nhật ký thay đổi dữ liệu (`audit_logs`) bắt buộc lưu trữ tối thiểu `7 năm`.
- **Thời gian lưu trữ lịch sử sửa chữa**: Lịch sử các phiếu sửa chữa và danh mục vật tư đã thay thế (`repair_tickets`, `material_requests`) lưu trữ tối thiểu `5 năm`.
- **Thời gian lưu trữ tệp tin media**: Các hình ảnh/video đính kèm hiện trường lỗi được lưu trữ trực tuyến tối thiểu `2 năm`. Sau thời gian này sẽ được nén lưu trữ lạnh (Cold Storage) hoặc xóa tùy theo cấu hình dung lượng máy chủ.

---

### Level 3 — Technical Detail (Chỉ số kỹ thuật định lượng)

#### Bảng danh mục yêu cầu phi chức năng (NFR Catalog)

| Mã NFR | Phân Nhóm | Tiêu Chí Kỹ Thuật | Chỉ Số Mục Tiêu (Target Metric) | Phương Pháp Kiểm Chứng |
| :--- | :--- | :--- | :--- | :--- |
| **NFR-PERF-01** | Hiệu năng | Thời gian phản hồi API đọc | 95% request dưới `1.2 giây` (tải danh sách xe). | Kiểm thử tải (Load Test) bằng JMeter. |
| **NFR-PERF-02** | Hiệu năng | Thời gian phản hồi API ghi | 95% request dưới `2.0 giây` (tạo ticket). | Đo thời gian xử lý API trên APM. |
| **NFR-SEC-01**  | Bảo mật | Mã hóa dữ liệu lưu local | Mã hóa thông tin đăng nhập và dữ liệu nháp bằng AES-256. | Rà soát mã nguồn & Pentest thiết bị. |
| **NFR-SEC-02**  | Bảo mật | Phòng chống tấn công brute-force| Khóa tài khoản đăng nhập `15 phút` sau 5 lần sai. | Unit test luồng đăng nhập. |
| **NFR-SCALE-01**| Mở rộng | Khả năng tải đồng thời | Phục vụ >= `500` active connections đồng thời (TBD). | Giả lập tải bằng k6 / Locust. |
| **NFR-MAIN-01** | Bảo trì | Độ bao phủ kiểm thử tự động | Coverage của Unit Test >= `80%`. | Chạy công cụ đo coverage (Istanbul/Jest). |
| **NFR-OFF-01**  | Ngoại tuyến| Khả năng lưu trữ cục bộ | Lưu trữ tối thiểu `100 ticket nháp` kèm ảnh offline. | Test thủ công bằng cách ngắt kết nối. |
| **NFR-OFF-02**  | Ngoại tuyến| Đồng bộ & Xử lý xung đột | Trả lỗi và chặn ghi đè nếu trạng thái ticket trên client cũ hơn server. | Giả lập conflict state qua Postman. |
| **NFR-USA-01**  | Khả dụng | Tỉ lệ thành công lần đầu | >= `95%` người dùng hoàn thành tạo ticket dưới 2 phút. | Quan sát thử nghiệm người dùng (UAT). |
| **NFR-REC-01**  | Sao lưu | Điểm phục hồi tối đa (RPO) | RPO <= `30 phút`. | Kiểm tra cấu hình WAL archiving của Postgres. |
| **NFR-DATA-01** | Lưu trữ | Lưu trữ log kiểm toán | Lưu trữ dữ liệu log kiểm toán >= `7 năm`. | Rà soát cấu hình database retention job. |

---

## Architecture Impact
Các yêu cầu phi chức năng trên trực tiếp dẫn tới các quyết định thiết kế kiến trúc hệ thống sau đây:
- **Offline-first (NFR-OFF-01)**: Bắt buộc sử dụng cơ sở dữ liệu cục bộ (Local DB như SQLite/WatermelonDB) trên thiết bị di động kết hợp với cơ chế xếp hàng đồng bộ (Sync Queue Client-side).
- **Uptime 99.5% (NFR-REL-01)**: Triển khai hạ tầng dưới dạng High Availability (HA) - Tính sẵn sàng cao, chạy nhiều bản ghi API phía sau Bộ cân bằng tải (Load Balancer).
- **Thời gian phản hồi nhanh (NFR-PERF-01)**: Áp dụng Redis Caching cho các dữ liệu tĩnh ít thay đổi (danh mục xe, danh mục vật tư phụ tùng) để giảm tải cho database.
- **Traceability & Audit trail (NFR-SEC-03)**: Thiết kế bảng ghi nhật ký bất biến (`audit_logs`) riêng biệt, sử dụng trigger DB hoặc Middleware chặn ghi đè trực tiếp lịch sử.
- **RPO <= 30 phút (NFR-REC-01)**: Sử dụng Cơ chế ghi trước (Write-Ahead Logging - WAL) của PostgreSQL và đẩy file lưu trữ WAL liên tục lên cloud storage (ví dụ: AWS S3).
- **Data Retention (NFR-DATA-01)**: Thiết kế các Batch Jobs chạy ngầm định kỳ hàng tuần để tự động nén, tối ưu hóa database, di chuyển hình ảnh cũ hơn 2 năm sang các phân vùng lưu trữ lạnh (Cold Storage) có chi phí rẻ hơn.

---

## Outputs
- Tệp đặc tả yêu cầu phi chức năng hoàn chỉnh: [08_non_functional_requirements.md](file:///c:/Users/Legion/Desktop/IT/SNP/.agents/skills/system_engineering_copilot/deliverables/current/sss/part_b_requirement_analysis/08_non_functional_requirements.md)
