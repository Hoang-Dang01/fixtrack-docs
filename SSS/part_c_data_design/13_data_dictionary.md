# 13 Data Dictionary

**Trạng thái: Hoàn thành (Finalized - Version 3.0)**

---

## Purpose
Chương này định nghĩa ý nghĩa nghiệp vụ, quy tắc validation, format dữ liệu và ràng buộc nhập liệu cho toàn bộ các trường dữ liệu quan trọng trong hệ thống VRMS. Data Dictionary đóng vai trò cầu nối giữa Business Rules, Database Schema, API Contracts và UI Forms, đảm bảo tất cả các bên liên quan (BA, Backend, Frontend, QA, Data Engineer) có chung một hiểu biết thống nhất về mặt ngữ nghĩa (semantics) và cấu trúc dữ liệu.

---

## Questions Answered
- Mỗi trường (field) trong cơ sở dữ liệu mang ý nghĩa nghiệp vụ gì?
- Quy tắc kiểm định dữ liệu (validation rules), định dạng bắt buộc (regex) và tầm vực giá trị của từng trường là gì?
- Trường dữ liệu nào do người dùng cung cấp (User Source), trường nào do hệ thống tự động khởi tạo (System Source)?
- Các trường dữ liệu dùng chung (Common Fields) tuân thủ quy ước thế nào để tối ưu hóa thiết kế?

---

## Inputs
- [12 Logical Database Design & ERD](file:///c:/Users/Legion/Desktop/IT/SNP/.agents/skills/system_engineering_copilot/deliverables/current/sss/part_c_data_design/12_erd.md)
- [09 Business Rules](file:///c:/Users/Legion/Desktop/IT/SNP/.agents/skills/system_engineering_copilot/deliverables/current/sss/part_b_requirement_analysis/09_business_rules.md)
- [07 Functional Requirements](file:///c:/Users/Legion/Desktop/IT/SNP/.agents/skills/system_engineering_copilot/deliverables/current/sss/part_b_requirement_analysis/07_functional_requirements.md)

---

## Common Fields Convention

Để tối ưu hóa không gian hiển thị và loại bỏ việc lặp lại thông tin không cần thiết ở cả 18 bảng, toàn bộ các bảng trong hệ thống VRMS mặc định tuân thủ quy ước đặt tên và kiểu dữ liệu cho các trường dùng chung dưới đây:

| Field | Business Name | Type | Required | Source | Validation | Description |
| :--- | :--- | :--- | :---: | :--- | :--- | :--- |
| `id` | ID bản ghi | UUID hoặc SERIAL/BIGSERIAL | Yes | System | Primary Key | Khóa chính duy nhất. UUID áp dụng cho Exposed Entities; SERIAL/BIGSERIAL áp dụng cho Internal Tables. |
| `created_at` | Thời điểm tạo | TIMESTAMP | Yes | System | Default: `CURRENT_TIMESTAMP` | Thời điểm bản ghi được ghi nhận vào CSDL. |
| `updated_at` | Thời điểm cập nhật | TIMESTAMP | Yes | System | Default: `CURRENT_TIMESTAMP` | Thời điểm bản ghi được cập nhật thông tin gần nhất. |

*Lưu ý: Các bảng dữ liệu dưới đây sẽ lược bỏ các cột dùng chung này và chỉ tập trung mô tả các trường nghiệp vụ đặc thù.*

---

## Level 1 — Custom ENUM Definitions

Hệ thống sử dụng **14 kiểu ENUM tùy chỉnh** để quản lý các giá trị trạng thái và vai trò nghiệp vụ cố định:

1.  **`user_role`**: Vai trò nghiệp vụ của tài khoản nhân sự.
    - Giá trị: `DRIVER`, `MECHANIC`, `TECH`, `INVENTORY`, `MANAGER`.
2.  **`user_status`**: Trạng thái hoạt động của tài khoản người dùng.
    - Giá trị: `ACTIVE`, `INACTIVE` (Khóa/Soft Delete).
3.  **`vehicle_type`**: Phân loại chủng loại phương tiện.
    - Giá trị: `INTERNAL_TRUCK` (Xe tải công ty), `INTERNAL_VAN` (Xe bán tải công ty), `EXTERNAL_CLIENT` (Xe khách ngoài vãng lai).
4.  **`vehicle_status`**: Trạng thái kỹ thuật/vận hành của phương tiện.
    - Giá trị: `ACTIVE` (Sẵn sàng chạy), `BROKEN` (Báo hỏng chờ xử lý), `REPAIRING` (Đang nằm xưởng sửa chữa).
5.  **`ticket_status`**: Vòng đời trạng thái của phiếu báo hỏng/sửa chữa.
    - Giá trị: `REPORTED`, `INSPECTING`, `WAITING_QUEUE`, `WAITING_PARTS`, `REPAIRING`, `COMPLETED`, `CLOSED`, `REJECTED`.
6.  **`ticket_priority`**: Độ ưu tiên xử lý sửa chữa xe.
    - Giá trị: `LOW`, `MEDIUM`, `HIGH`.
7.  **`inspection_type`**: Phân loại đợt kiểm tra kỹ thuật phương tiện.
    - Giá trị: `INITIAL_INSPECTION`, `TECHNICAL_DIAGNOSIS`, `FINAL_ACCEPTANCE`.
8.  **`inspection_decision`**: Quyết định của Cơ giới/KTV sau kiểm tra.
    - Giá trị: `APPROVE`, `REJECT`, `QUEUE`, `REPAIR_AGAIN`.
9.  **`queue_entry_status`**: Trạng thái hàng chờ xe đỗ xưởng.
    - Giá trị: `WAITING`, `PROMOTED`, `CANCELLED`.
10. **`warehouse_status`**: Trạng thái hoạt động nghiệp vụ của kho hàng.
    - Giá trị: `ACTIVE`, `INACTIVE` (Soft Delete).
11. **`material_request_status`**: Tiến trình phê duyệt cấp linh kiện thay thế.
    - Giá trị: `PENDING`, `APPROVED`, `PARTIAL`, `ISSUED`, `RECEIVED`, `REJECTED`.
12. **`repair_job_status`**: Trạng thái thực thi một đầu việc sửa chữa của KTV.
    - Giá trị: `REPAIRING`, `COMPLETED`.
13. **`notification_type`**: Phân loại thông điệp gửi ứng dụng di động.
    - Giá trị: `TICKET_UPDATED`, `QUEUE_PROMOTED`, `PARTS_READY`, `SYSTEM_ALERT`.
14. **`slot_status`**: Trạng thái vật lý của cầu sửa xe tại xưởng kỹ thuật.
    - Giá trị: `VACANT` (Đang trống), `OCCUPIED` (Đang có xe sửa), `MAINTENANCE` (Đang bảo trì cầu).
15. **`delivery_status`**: Trạng thái chuyển giao thông báo đẩy (Push/SMS notification).
    - Giá trị: `PENDING` (Chờ gửi), `SENT` (Đã gửi thành công), `FAILED` (Gửi lỗi).

---

## Level 2 — Field Dictionary

Tài liệu từ điển dữ liệu được phân chia thành **6 Business Modules** nghiệp vụ khép kín:

### Module 1 — User Data Dictionary

#### **Bảng `users` (Tài khoản nhân sự)**
- **Mô tả:** Quản lý thông tin định danh và phân quyền của toàn bộ nhân viên tham gia hệ thống.
- **Loại khóa chính (`id`):** UUID (Exposed Entity).

| Field | Business Name | Type | Required | Source | Validation | Description |
| :--- | :--- | :--- | :---: | :--- | :--- | :--- |
| `employee_code` | Mã nhân viên | VARCHAR(20) | Yes | User | UNIQUE, Regex: `^EMP-[0-9]{5}$` | Mã số nhân viên duy nhất dùng để đăng nhập. |
| `full_name` | Họ và tên | VARCHAR(100) | Yes | User | Độ dài: 2 - 100 ký tự | Họ và tên đầy đủ của nhân sự. |
| `phone` | Số điện thoại | VARCHAR(20) | Yes | User | UNIQUE, Regex: `^\+?[0-9]{10,15}$` | Số điện thoại di động liên hệ chính thức. |
| `role` | Vai trò hệ thống | ENUM | Yes | User | ENUM `user_role` | Quyền hạn nghiệp vụ để điều phối màn hình và API. |
| `status` | Trạng thái | ENUM | Yes | System | ENUM `user_status` | Trạng thái hoạt động tài khoản (Soft Delete nếu INACTIVE). |

---

### Module 2 — Vehicle Data Dictionary

#### **Bảng `vehicles` (Danh mục phương tiện)**
- **Mô tả:** Quản lý danh mục toàn bộ xe tải công ty và xe ngoài vãng lai vào sửa chữa.
- **Loại khóa chính (`id`):** UUID (Exposed Entity).

| Field | Business Name | Type | Required | Source | Validation | Description |
| :--- | :--- | :--- | :---: | :--- | :--- | :--- |
| `vehicle_code` | Mã quản lý xe | VARCHAR(50) | Yes | System/User | UNIQUE, Regex: `^VEH-[0-9]{5}$` | Mã định danh nội bộ do công ty quản lý và cấp phát. |
| `plate_number` | Biển số xe | VARCHAR(20) | Yes | User | UNIQUE, Regex: `^[0-9]{2}[A-Z]-[0-9]{4,5}$` | Biển kiểm soát thực tế đăng ký của phương tiện. |
| `type` | Chủng loại xe | ENUM | Yes | User | ENUM `vehicle_type` | Phân loại xe để định tuyến làn/cầu sửa chữa phù hợp. |
| `status` | Trạng thái xe | ENUM | Yes | System/Mech | ENUM `vehicle_status` | Tình trạng sẵn sàng hoạt động kỹ thuật của xe (Soft Delete bằng cách chuyển trạng thái ngừng hoạt động). |
| `is_external` | Phương tiện ngoài | BOOLEAN | Yes | User | DEFAULT `FALSE` | Cờ phân biệt xe khách vãng lai (TRUE) vs xe công ty (FALSE). |

#### **Bảng `vehicle_assignments` (Phân công xe cho tài xế)**
- **Mô tả:** Theo dõi lịch sử gán xe cho tài xế theo ca chạy, giải quyết bài toán 1 tài xế lái nhiều xe và ngược lại.
- **Loại khóa chính (`id`):** SERIAL (Internal Table).

| Field | Business Name | Type | Required | Source | Validation | Description |
| :--- | :--- | :--- | :---: | :--- | :--- | :--- |
| `vehicle_id` | ID phương tiện | UUID | Yes | User | FK `vehicles(id)`, ON DELETE CASCADE | ID xe được phân công lái. |
| `driver_id` | ID tài xế | UUID | Yes | User | FK `users(id)`, ON DELETE CASCADE | ID tài xế nhận xe hoạt động. |
| `assigned_at` | Thời gian nhận xe | TIMESTAMP | Yes | System | Phải trước `released_at` | Thời điểm bắt đầu bàn giao xe thực tế. |
| `released_at` | Thời gian trả xe | TIMESTAMP | No | System | Phải sau `assigned_at` | Thời điểm tài xế trả xe lại bãi (NULL khi đang chạy). |
| `is_active` | Trạng thái ca | BOOLEAN | Yes | System | DEFAULT `TRUE` | Ca phân công còn hiệu lực chạy xe hay không. |

---

### Module 3 — Ticket Data Dictionary

#### **Bảng `repair_tickets` (Phiếu báo sửa chữa)**
- **Mô tả:** Quản lý vòng đời phiếu báo sự cố kỹ thuật xe từ lúc khởi tạo đến khi nghiệm thu đóng phiếu.
- **Loại khóa chính (`id`):** UUID (Exposed Entity).

| Field | Business Name | Type | Required | Source | Validation | Description |
| :--- | :--- | :--- | :---: | :--- | :--- | :--- |
| `ticket_no` | Mã phiếu | VARCHAR(50) | Yes | System | UNIQUE, Regex: `^RT-\d{8}-\d{4}$` | Mã số định danh duy nhất của phiếu hiển thị UI. |
| `vehicle_id` | ID phương tiện | UUID | Yes | User | FK `vehicles(id)`, ON DELETE RESTRICT | ID phương tiện gặp sự cố kỹ thuật. |
| `driver_id` | ID tài xế báo | UUID | No | User/System | FK `users(id)`, ON DELETE SET NULL | ID tài xế báo lỗi (bắt buộc nếu xe nội bộ). Ràng buộc xe nội bộ thì driver_id NOT NULL được kiểm tra tại Application Service Layer. |
| `guest_owner_name`| Tên chủ xe ngoài | VARCHAR(100)| No | User | Độ dài: 2 - 100 ký tự | Tên chủ xe vãng lai (bắt buộc nếu xe ngoài). |
| `guest_owner_phone`| SĐT chủ xe ngoài | VARCHAR(20)| No | User | Regex: `^\+?[0-9]{10,15}$` | Số điện thoại chủ xe ngoài (bắt buộc nếu xe ngoài). |
| `status` | Trạng thái phiếu | ENUM | Yes | System | ENUM `ticket_status` | Trạng thái tiến độ xử lý phiếu sửa chữa. |
| `priority` | Độ ưu tiên sửa | ENUM | Yes | User/Mech | ENUM `ticket_priority` | Mức ưu tiên kỹ thuật để sắp xếp sửa chữa. |

> **Ràng buộc kiểm tra nghiệp vụ (CHECK constraint):**
> `CHECK ((driver_id IS NOT NULL AND guest_owner_name IS NULL AND guest_owner_phone IS NULL) OR (driver_id IS NULL AND guest_owner_name IS NOT NULL AND guest_owner_phone IS NOT NULL))`

#### **Bảng `ticket_issues` (Danh mục lỗi chi tiết)**
- **Mô tả:** Ghi nhận các lỗi hư hỏng chi tiết trên một ticket báo sửa chữa (Hỗ trợ báo đa lỗi).
- **Loại khóa chính (`id`):** UUID (Exposed Entity).

| Field | Business Name | Type | Required | Source | Validation | Description |
| :--- | :--- | :--- | :---: | :--- | :--- | :--- |
| `ticket_id` | ID phiếu cha | UUID | Yes | System | FK `repair_tickets(id)`, ON DELETE CASCADE | ID phiếu báo hỏng chứa hạng mục lỗi này. |
| `issue_type` | Danh mục lỗi | VARCHAR(50) | Yes | User | Độ dài: 2 - 50 ký tự | Nhóm hạng mục lỗi kỹ thuật (Lốp, Phanh, Động cơ...). |
| `description` | Mô tả chi tiết | TEXT | Yes | User | Không được để trống | Mô tả chi tiết hiện trạng hỏng hóc thực tế. |
| `severity` | Mức độ lỗi | ENUM | Yes | User | ENUM `ticket_priority` | Độ nghiêm trọng của lỗi đơn lẻ này. |

#### **Bảng `attachments` (Metadata tệp tài liệu, ảnh đính kèm)**
- **Mô tả:** Lưu trữ metadata liên kết các file hình ảnh/video sự cố kỹ thuật tải lên Cloud Storage.
- **Loại khóa chính (`id`):** UUID (Exposed Entity).

| Field | Business Name | Type | Required | Source | Validation | Description |
| :--- | :--- | :--- | :---: | :--- | :--- | :--- |
| `file_name` | Tên tệp gốc | VARCHAR(255) | Yes | User | Không trống, tối đa 255 ký tự | Tên file do người dùng chọn tải lên. |
| `mime_type` | Định dạng file | VARCHAR(100) | Yes | System | E.g., 'image/png', 'video/mp4' | Định dạng tệp tin. |
| `size_bytes` | Dung lượng tệp | BIGINT | Yes | System | CHECK: `size_bytes > 0` | Dung lượng tệp tin (byte). |
| `storage_key` | Khóa lưu kho | VARCHAR(255) | Yes | System | UNIQUE | Khóa định danh file trên S3/MinIO. |
| `file_url` | Đường dẫn CDN | VARCHAR(512) | Yes | System | Định dạng URL | Link CDN để ứng dụng di động truy xuất trực tiếp file. |
| `entity_type` | Thực thể liên kết | VARCHAR(50) | Yes | System | 'TICKET', 'INSPECTION', 'REPAIR' | Phân loại liên kết (quan hệ đa hình). Kiểm soát referential integrity tại Application Service Layer. |
| `entity_id` | ID thực thể | UUID | Yes | System | Định dạng UUIDv4 | ID bản ghi của thực thể cha liên quan. |
| `uploaded_by` | Người tải file | UUID | No | System | FK `users(id)`, ON DELETE SET NULL | ID tài khoản thực hiện upload file. |
| `uploaded_at` | Thời điểm tải | TIMESTAMP | Yes | System | Default: `CURRENT_TIMESTAMP` | Thời gian upload tệp thành công. |

---

### Module 4 — Inventory Data Dictionary

#### **Bảng `warehouses` (Danh mục kho phụ tùng)**
- **Mô tả:** Danh sách các kho chứa phụ tùng, linh kiện sửa chữa xe.
- **Loại khóa chính (`id`):** UUID (Exposed Entity).

| Field | Business Name | Type | Required | Source | Validation | Description |
| :--- | :--- | :--- | :---: | :--- | :--- | :--- |
| `warehouse_name` | Tên kho | VARCHAR(100) | Yes | User | UNIQUE, Độ dài: 2 - 100 ký tự | Tên gọi phân biệt kho hàng. |
| `location` | Địa chỉ kho | VARCHAR(255) | Yes | User | Không được để trống | Địa chỉ vật lý chính xác của kho. |
| `status` | Trạng thái hoạt động | ENUM | Yes | Manager | ENUM `warehouse_status` | Trạng thái mở/đóng kho hàng. |
| `is_active` | Đang vận hành | BOOLEAN | Yes | System | DEFAULT `TRUE` | Cờ hỗ trợ Soft Delete kho hàng. |

#### **Bảng `materials` (Danh mục vật tư phụ tùng)**
- **Mô tả:** Danh mục tổng quản lý toàn bộ linh kiện phụ tùng thay thế.
- **Loại khóa chính (`id`):** UUID (Exposed Entity).

| Field | Business Name | Type | Required | Source | Validation | Description |
| :--- | :--- | :--- | :---: | :--- | :--- | :--- |
| `sku` | Mã SKU | VARCHAR(50) | Yes | User | UNIQUE, định dạng mã vật tư | Mã số quản lý kho độc nhất của linh kiện. |
| `name` | Tên linh kiện | VARCHAR(100) | Yes | User | Độ dài: 2 - 100 ký tự | Tên gọi chi tiết của phụ tùng. |
| `category` | Phân nhóm vật tư | VARCHAR(50) | Yes | User | Độ dài: 2 - 50 ký tự | Nhóm danh mục (Dầu, Lốp, Lọc gió...). |
| `unit` | Đơn vị tính | VARCHAR(20) | Yes | User | Kiểm định theo list: `['Cái', 'Lít', 'Bộ', 'Cặp', 'Hộp']` | Đơn vị đo lường tính toán phụ tùng. |
| `reorder_level` | Ngưỡng báo động | INT | Yes | User | CHECK: `reorder_level >= 0` | Mức tồn tối thiểu để hệ thống cảnh báo nhập thêm. |
| `is_active` | Đang vận hành | BOOLEAN | Yes | System | DEFAULT `TRUE` | Cờ hỗ trợ Soft Delete linh kiện. |

#### **Bảng `inventory_stock` (Quản lý số lượng tồn kho thực tế)**
- **Mô tả:** Quản lý tồn kho thực tế đang có sẵn và số lượng đã đặt trước giữ chỗ tại từng kho hàng.
- **Loại khóa chính (`material_id`, `warehouse_id`):** Composite Key (2 UUIDs).

| Field | Business Name | Type | Required | Source | Validation | Description |
| :--- | :--- | :--- | :---: | :--- | :--- | :--- |
| `material_id` | ID phụ tùng | UUID | Yes | System | FK `materials(id)`, ON DELETE CASCADE | ID vật tư phụ tùng. |
| `warehouse_id` | ID kho chứa | UUID | Yes | System | FK `warehouses(id)`, ON DELETE CASCADE | ID kho lưu trữ. |
| `quantity_on_hand` | Tồn thực tế | INT | Yes | System/User | CHECK: `quantity_on_hand >= 0` | Số lượng tồn kho thực tế vật lý đang có trong kho. |
| `quantity_reserved` | Tồn đặt trước | INT | Yes | System | CHECK: `quantity_reserved >= 0` | Số lượng đã giữ chỗ cho xe đang chờ sửa. |

> **Ràng buộc CHECK ở mức CSDL:**
> `CHECK (quantity_on_hand >= quantity_reserved)`
> Lượng hàng khả dụng thực tế để tạo mới yêu cầu (available) được tính động tại Application layer bằng: `quantity_on_hand - quantity_reserved`.

#### **Bảng `material_requests` (Yêu cầu phụ tùng)**
- **Mô tả:** Đơn yêu cầu cấp linh kiện do KTV lập để sửa xe.
- **Loại khóa chính (`id`):** UUID (Exposed Entity).

| Field | Business Name | Type | Required | Source | Validation | Description |
| :--- | :--- | :--- | :---: | :--- | :--- | :--- |
| `ticket_id` | ID phiếu xe | UUID | Yes | User | FK `repair_tickets(id)`, ON DELETE RESTRICT | ID phiếu sửa xe cần nhận phụ tùng. |
| `requester_id` | ID KTV yêu cầu | UUID | Yes | System | FK `users(id)`, ON DELETE RESTRICT | ID KTV lập đơn yêu cầu. |
| `status` | Trạng thái đơn | ENUM | Yes | System/WH | ENUM `material_request_status` | Trạng thái duyệt cấp vật tư từ thủ kho. |

#### **Bảng `material_request_items` (Chi tiết linh kiện đơn yêu cầu)**
- **Mô tả:** Dòng chi tiết ghi nhận số lượng đề xuất, số lượng duyệt và thực xuất cho từng linh kiện.
- **Loại khóa chính (`id`):** BIGSERIAL (Internal Table).

| Field | Business Name | Type | Required | Source | Validation | Description |
| :--- | :--- | :--- | :---: | :--- | :--- | :--- |
| `request_id` | ID đơn yêu cầu cha | UUID | Yes | System | FK `material_requests(id)`, ON DELETE CASCADE | ID đơn yêu cầu cha. |
| `material_id` | ID linh kiện | UUID | Yes | User | FK `materials(id)`, ON DELETE RESTRICT | ID phụ tùng cần cấp phát. |
| `quantity_requested`| Số lượng yêu cầu | INT | Yes | User | CHECK: `quantity_requested > 0` | Số lượng KTV đăng ký muốn nhận. |
| `quantity_approved` | INT | | Yes | WH | CHECK: `quantity_approved >= 0` | Số lượng được thủ kho duyệt cấp. |
| `quantity_issued` | INT | | Yes | WH | CHECK: `quantity_issued >= 0` | Số lượng thực tế thủ kho bàn giao cho xe. |

> **Ràng buộc CHECK ở mức CSDL:**
> `CHECK (quantity_issued <= quantity_approved)`

---

### Module 5 — Repair Data Dictionary

#### **Bảng `queue_entries` (Dòng xếp hàng chờ chi tiết)**
- **Mô tả:** Quản lý số thứ tự xếp hàng chờ vào cầu của xe khi xưởng hết chỗ.
- **Loại khóa chính (`id`):** BIGSERIAL (Internal Table).

| Field | Business Name | Type | Required | Source | Validation | Description |
| :--- | :--- | :--- | :---: | :--- | :--- | :--- |
| `ticket_id` | ID phiếu xe | UUID | Yes | System | FK `repair_tickets(id)`, ON DELETE CASCADE | ID phiếu sửa chữa đang xếp hàng chờ. |
| `position` | Vị trí hàng chờ | INT | Yes | System | CHECK: `position > 0` | Số thứ tự xếp hàng tự động theo FIFO. |
| `priority` | Độ ưu tiên queue | SMALLINT | Yes | System/Mgr | CHECK: `priority >= 0` | Mức ưu tiên hàng chờ (0: Thường, 10: Gấp, 100: Khẩn). |
| `entered_at` | Thời điểm vào | TIMESTAMP | Yes | System | Default: `CURRENT_TIMESTAMP` | Thời điểm đi vào hàng chờ. |
| `status` | Trạng thái xếp hàng | ENUM | Yes | System | ENUM `queue_entry_status` | Trạng thái xếp hàng (đang chờ/được gọi/hủy). |

#### **Bảng `inspection_records` (Biên bản kiểm định)**
- **Mô tả:** Ghi nhận đánh giá kỹ thuật của 3 đợt kiểm tra phương tiện.
- **Loại khóa chính (`id`):** UUID (Exposed Entity).

| Field | Business Name | Type | Required | Source | Validation | Description |
| :--- | :--- | :--- | :---: | :--- | :--- | :--- |
| `ticket_id` | ID phiếu xe | UUID | Yes | System | FK `repair_tickets(id)`, ON DELETE CASCADE | ID phiếu xe được kiểm tra kỹ thuật. |
| `inspection_type` | Đợt kiểm định | ENUM | Yes | System | ENUM `inspection_type` | Loại kiểm định (Sơ bộ bãi, Chẩn đoán, Nghiệm thu). |
| `inspected_by` | Kỹ thuật kiểm tra | UUID | Yes | System | FK `users(id)`, ON DELETE RESTRICT | ID Cơ giới hoặc KTV thực hiện kiểm định. |
| `findings` | Ghi nhận kỹ thuật | TEXT | Yes | User | Không được để trống | Đánh giá chi tiết tình trạng xe. |
| `decision` | Quyết định xử lý | ENUM | Yes | User | ENUM `inspection_decision` | Kết luận sau kiểm định (Duyệt/Sửa lại/Xếp hàng). |
| `inspected_at` | Thời điểm kiểm | TIMESTAMP | Yes | System | Default: `CURRENT_TIMESTAMP` | Thời gian thực hiện kiểm tra xe. |

#### **Bảng `workshop_slots` (Cầu sửa chữa vật lý)**
- **Mô tả:** Danh sách các cầu nâng/làn sửa xe vật lý tại xưởng kỹ thuật.
- **Loại khóa chính (`id`):** UUID (Exposed Entity).

| Field | Business Name | Type | Required | Source | Validation | Description |
| :--- | :--- | :--- | :---: | :--- | :--- | :--- |
| `slot_name` | Ký hiệu cầu | VARCHAR(50) | Yes | User | UNIQUE, Độ dài: 2 - 50 ký tự | Tên gọi phân biệt làn cầu sửa (e.g., 'Cầu nâng số 1'). |
| `status` | Trạng thái cầu | ENUM | Yes | System | ENUM `slot_status` | Trạng thái hiện tại của cầu (VACANT / OCCUPIED / MAINTENANCE). |

#### **Bảng `workshop_slot_assignments` (Lịch sử đỗ cầu sửa xe)**
- **Mô tả:** Lưu vết đỗ xe tại cầu kỹ thuật phục vụ đo lường năng suất xưởng.
- **Loại khóa chính (`id`):** SERIAL (Internal Table).

| Field | Business Name | Type | Required | Source | Validation | Description |
| :--- | :--- | :--- | :---: | :--- | :--- | :--- |
| `slot_id` | ID cầu nâng | UUID | Yes | System | FK `workshop_slots(id)`, ON DELETE CASCADE | ID cầu sửa xe. |
| `ticket_id` | ID phiếu xe | UUID | Yes | System | FK `repair_tickets(id)`, ON DELETE CASCADE | ID phiếu sửa xe đỗ tại cầu. |
| `assigned_at` | Thời điểm lên cầu | TIMESTAMP | Yes | System | Phải trước `released_at` | Thời gian xe bắt đầu lên cầu nâng sửa. |
| `released_at` | Thời điểm rời cầu | TIMESTAMP | No | System | Phải sau `assigned_at` | Thời gian xe rời cầu sau sửa xong (NULL khi đang đỗ). |

> **Ràng buộc chỉ mục độc nhất một phần (Partial Unique Indexes):**
> 1. `unique_active_slot_assignment` trên `(slot_id) WHERE (released_at IS NULL)` (Khóa chặn 1 cầu sửa chỉ chứa tối đa 1 xe hoạt động).
> 2. `unique_active_ticket_assignment` trên `(ticket_id) WHERE (released_at IS NULL)` (Khóa chặn 1 xe chỉ đỗ ở duy nhất 1 cầu sửa tại một thời điểm).

#### **Bảng `repair_jobs` (Thực thi tác vụ sửa chữa)**
- **Mô tả:** Quá trình thi công thực tế của KTV trên một phiếu sửa chữa (Hỗ trợ phân công nhiều KTV làm các hạng mục khác nhau).
- **Loại khóa chính (`id`):** UUID (Exposed Entity).

| Field | Business Name | Type | Required | Source | Validation | Description |
| :--- | :--- | :--- | :---: | :--- | :--- | :--- |
| `ticket_id` | ID phiếu xe | UUID | Yes | System | FK `repair_tickets(id)`, ON DELETE CASCADE | ID phiếu sửa xe liên quan. |
| `technician_id` | ID KTV thi công | UUID | Yes | System | FK `users(id)`, ON DELETE RESTRICT | ID KTV được giao sửa hạng mục này. |
| `started_at` | Bắt đầu sửa | TIMESTAMP | Yes | System | Phải trước `completed_at` | Thời điểm KTV bắt đầu sửa thực tế. |
| `completed_at` | Hoàn thành sửa | TIMESTAMP | No | System | Phải sau `started_at` | Thời điểm KTV bấm báo sửa xong (NULL khi đang làm). |
| `notes` | Ghi chú thi công | TEXT | No | User | Độ dài tối đa 500 ký tự | Ghi chép kỹ thuật của KTV trong ca làm. |
| `status` | Trạng thái tác vụ | ENUM | Yes | System | ENUM `repair_job_status` | Trạng thái sửa (REPAIRING / COMPLETED). |

---

### Module 6 — Audit Data Dictionary

#### **Bảng `notifications` (Hộp thư thông báo trong ứng dụng)**
- **Mô tả:** Hộp thư lưu trữ thông báo đẩy cho người dùng xem lại khi online trở lại.
- **Loại khóa chính (`id`):** UUID (Exposed Entity).

| Field | Business Name | Type | Required | Source | Validation | Description |
| :--- | :--- | :--- | :---: | :--- | :--- | :--- |
| `user_id` | ID người nhận | UUID | Yes | System | FK `users(id)`, ON DELETE CASCADE | ID người dùng nhận thông báo. |
| `notification_type` | Nhóm thông báo | ENUM | Yes | System | ENUM `notification_type` | Nhóm loại thông báo gửi ứng dụng. |
| `title` | Tiêu đề | VARCHAR(150) | Yes | System | Không trống, tối đa 150 ký tự | Tiêu đề thông báo ngắn hiển thị trên UI. |
| `body` | Nội dung chi tiết | TEXT | Yes | System | Không được để trống | Nội dung thông điệp chi tiết. |
| `is_read` | Đã đọc | BOOLEAN | Yes | System | DEFAULT `FALSE` | Trạng thái người dùng đã bấm xem thông báo. |
| `delivery_status` | Trạng thái gửi | ENUM | Yes | System | ENUM `delivery_status` | Cảnh báo trạng thái chuyển giao notification đẩy. |
| `sent_at` | Thời điểm gửi | TIMESTAMP | No | System | NULL nếu chưa gửi / gửi lỗi | Thời điểm gửi thành công thông báo đến thiết bị. |

#### **Bảng `audit_logs` (Nhật ký kiểm toán hệ thống)**
- **Mô tả:** Lưu trữ lịch sử toàn bộ các thao tác sửa đổi dữ liệu nhạy cảm của nhân sự (Bảng phân vùng theo tháng).
- **Loại khóa chính (`id`, `created_at`):** Composite Key (BIGINT + TIMESTAMP).

| Field | Business Name | Type | Required | Source | Validation | Description |
| :--- | :--- | :--- | :---: | :--- | :--- | :--- |
| `actor_id` | ID người thao tác | UUID | Yes | System | FK `users(id)`, ON DELETE SET NULL | ID tài khoản thực hiện thao tác. |
| `action` | Loại hành động | VARCHAR(100) | Yes | System | Độ dài: 2 - 100 ký tự | Tên loại thao tác (e.g., 'QUEUE_OVERRIDE'). |
| `entity_type` | Bảng tác động | VARCHAR(50) | Yes | System | Tên bảng dữ liệu CSDL | Tên bảng dữ liệu chịu tác động thay đổi. |
| `entity_id` | ID dòng dữ liệu | UUID | Yes | System | Định dạng UUIDv4 | ID khóa chính của đối tượng bị thay đổi. |
| `old_values` | Trạng thái cũ | JSONB | No | System | Định dạng JSON | Dữ liệu cũ trước khi thao tác sửa đổi. |
| `new_values` | Trạng thái mới | JSONB | No | System | Định dạng JSON | Dữ liệu mới sau khi thao tác sửa đổi. |

---

## Level 3 — Validation & Business Semantics

#### 1. Định dạng Mã phiếu sửa chữa (`ticket_no` format)
- **Quy tắc:** Tạo tự động bởi hệ thống dưới định dạng `RT-YYYYMMDD-XXXX`.
  - `RT`: Hậu tố cố định (Repair Ticket).
  - `YYYYMMDD`: Năm, tháng, ngày khởi tạo phiếu.
  - `XXXX`: Số thứ tự tăng dần trong ngày gồm 4 chữ số, tự động reset về `0001` vào lúc 00:00:00 hàng ngày.
- **Biểu thức chính quy (Regex):**
  ```regex
  ^RT-\d{8}-\d{4}$
  ```
- **Ví dụ:** `RT-20260626-0001`

#### 2. Định dạng Số điện thoại di động (`phone` format)
- **Quy tắc:** Chỉ chấp nhận chữ số, độ dài từ 10 đến 15 ký tự, hỗ trợ tiền tố mã quốc gia dấu cộng (`+`).
- **Biểu thức chính quy (Regex):**
  ```regex
  ^\+?[0-9]{10,15}$
  ```
- **Ví dụ:** `0912345678`, `+84912345678`

#### 3. Định dạng Biển số xe đăng ký (`plate_number` format)
- **Quy tắc:** Ký tự viết hoa, loại bỏ toàn bộ dấu cách trống (trim), định dạng chuẩn Việt Nam gồm phân vùng tỉnh thành, chữ cái seri và dải số đăng ký (4 hoặc 5 số).
- **Biểu thức chính quy (Regex):**
  ```regex
  ^[0-9]{2}[A-Z]-[0-9]{4,5}$
  ```
- **Ví dụ:** `51A-12345` (TP.HCM), `29H-6789` (Hà Nội)

#### 4. Định dạng Mã số nhân viên (`employee_code` format)
- **Quy tắc:** Định dạng cố định của hệ thống: `EMP-` kết hợp 5 chữ số thứ tự.
- **Biểu thức chính quy (Regex):**
  ```regex
  ^EMP-[0-9]{5}$
  ```
- **Ví dụ:** `EMP-00042`

#### 5. Định dạng Mã phương tiện nội bộ (`vehicle_code` format)
- **Quy tắc:** Định dạng cố định của hệ thống: `VEH-` kết hợp 5 chữ số thứ tự.
- **Biểu thức chính quy (Regex):**
  ```regex
  ^VEH-[0-9]{5}$
  ```
- **Ví dụ:** `VEH-00012`
