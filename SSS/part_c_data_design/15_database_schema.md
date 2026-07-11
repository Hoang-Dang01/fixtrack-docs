# Chapter 15 — Database Schema

**Trạng thái: Hoàn thành (Finalized - Version 2.0)**

---

## Purpose
Chương này đặc tả **Kịch bản cơ sở dữ liệu vật lý (Physical DDL SQL Schema Script)** để thiết lập cấu trúc cơ sở dữ liệu cho hệ thống VRMS trên hệ quản trị cơ sở dữ liệu quan hệ PostgreSQL (phiên bản khuyến nghị: 16+). Tài liệu này cung cấp mã nguồn SQL DDL sạch, được tối ưu hóa cho các chỉ mục (indexes), các ràng buộc kiểm tra toàn vẹn (check constraints), và cơ chế phân vùng bảng (partitioning), sẵn sàng để sử dụng cho các công cụ Migration (như Prisma, TypeORM) hoặc chạy trực tiếp trong PostgreSQL CLI.

---

## Questions Answered
- Các câu lệnh SQL cụ thể nào được sử dụng để tạo kiểu dữ liệu ENUM tùy chỉnh cho hệ thống?
- Cú pháp DDL chi tiết để tạo 18 bảng vật lý với đầy đủ Primary Keys, Foreign Keys và CHECK Constraints là gì?
- Chỉ mục (Indexes), Chỉ mục một phần (Partial Indexes) và chỉ mục phức hợp nào được thiết lập để tối ưu hóa hiệu năng truy vấn và bảo vệ business rules?
- Cấu trúc phân vùng (Partitioning by Range) cho bảng `audit_logs` được khai báo như thế nào trong môi trường production?
- Các trigger tự động (như tự động cập nhật thời gian `updated_at`) được viết như thế nào bằng PL/pgSQL?
- Chiến lược quản lý tiến trình thay đổi cấu trúc cơ sở dữ liệu (Schema Evolution & Migration Strategy) được quy định ra sao?

---

## Inputs
- [12 Logical Database Design & ERD](file:///c:/Users/Legion/Desktop/IT/SNP/.agents/skills/system_engineering_copilot/deliverables/current/sss/part_c_data_design/12_erd.md)
- [13 Data Dictionary](file:///c:/Users/Legion/Desktop/IT/SNP/.agents/skills/system_engineering_copilot/deliverables/current/sss/part_c_data_design/13_data_dictionary.md)
- [14 CRUD Matrix](file:///c:/Users/Legion/Desktop/IT/SNP/.agents/skills/system_engineering_copilot/deliverables/current/sss/part_c_data_design/14_crud_matrix.md)

---

## Content

### Level 1 — Summary
Kịch bản cơ sở dữ liệu vật lý của VRMS được thiết kế tối ưu cho PostgreSQL 16. Để đảm bảo tính độc lập và toàn vẹn tham chiếu, kịch bản khởi tạo được chia nhỏ và thực thi theo thứ tự tuyến tính nghiêm ngặt gồm 6 phần:
1. **Extensions & Types**: Khởi tạo extension `pgcrypto` để sinh mã khóa UUIDv4 tự động bằng `gen_random_uuid()` và định nghĩa 15 kiểu ENUM tùy chỉnh.
2. **Master Tables**: Khởi tạo các bảng danh mục độc lập không chứa khóa ngoại phức tạp (users, vehicles, warehouses, materials, workshop_slots).
3. **Transactional & Junction Tables**: Khởi tạo các bảng chứa khóa ngoại tham chiếu chéo phục vụ lưu trữ giao dịch và liên kết thực thể.
4. **Indexes**: Khai báo rõ ràng các chỉ mục chuẩn, chỉ mục khóa ngoại, chỉ mục phức hợp và các chỉ mục một phần (Partial Unique Indexes) để bảo vệ toàn vẹn nghiệp vụ.
5. **Triggers**: Tạo hàm PL/pgSQL và gán các trigger tự động cập nhật thời gian sửa đổi gần nhất (`updated_at`).
6. **Migration Strategy**: Định hướng chiến lược tiến hóa lược đồ dữ liệu và rollback khi triển khai.

---

### Level 2 — Breakdown (Cơ cấu & Thứ tự thực thi SQL DDL)

Mã nguồn SQL DDL trong chương này được tổ chức thành 6 phần chính để đảm bảo khả năng bảo trì và triển khai an toàn:

#### 1. Extensions & Custom Types (ENUMs)
Tạo extension phục vụ sinh khóa UUIDv4 (`gen_random_uuid()`) và khởi tạo 15 kiểu dữ liệu ENUM tùy chỉnh định nghĩa các trạng thái nghiệp vụ.

#### 2. Master Data Tables (Các bảng danh mục)
Khai báo cấu trúc các bảng không phụ thuộc vào khóa ngoại của bảng khác, bao gồm:
- Bảng `users` (Tài khoản người dùng)
- Bảng `vehicles` (Danh mục xe)
- Bảng `warehouses` (Danh mục kho)
- Bảng `materials` (Danh mục linh kiện)
- Bảng `workshop_slots` (Danh mục cầu nâng)

#### 3. Transactional & Junction Tables (Các bảng giao dịch và trung gian)
Khai báo cấu trúc các bảng chứa khóa ngoại tham chiếu đến các bảng danh mục, bao gồm:
- Bảng `vehicle_assignments` (Phân công lái xe)
- Bảng `repair_tickets` (Phiếu báo hỏng/sửa chữa)
- Bảng `ticket_issues` (Chi tiết lỗi hỏng)
- Bảng `attachments` (Tệp tài liệu/ảnh đính kèm đa hình)
- Bảng `inventory_stock` (Quản lý số tồn kho tại mỗi kho)
- Bảng `material_requests` (Yêu cầu cấp phát phụ tùng)
- Bảng `material_request_items` (Chi tiết phụ tùng yêu cầu)
- Bảng `queue_entries` (Hàng chờ đỗ xưởng)
- Bảng `inspection_records` (Biên bản kiểm định kỹ thuật)
- Bảng `workshop_slot_assignments` (Lịch sử xe đỗ cầu nâng)
- Bảng `repair_jobs` (Phân công tác vụ cho KTV)
- Bảng `notifications` (Thông báo ứng dụng)

#### 4. Partitioned Audit Tables (Bảng kiểm toán phân vùng)
Khai báo bảng `audit_logs` sử dụng tính năng phân vùng theo dải thời gian của PostgreSQL (`PARTITION BY RANGE (created_at)`), kèm theo kịch bản tạo các phân vùng mẫu theo tháng.

#### 5. Database Triggers & Functions
Khai báo hàm PL/pgSQL chung để cập nhật trường `updated_at` và gắn trigger vào các bảng liên quan để tự động hóa hoạt động này ở mức CSDL.

---

### Level 3 — Technical Detail (Mã nguồn SQL DDL Chi tiết)

Dưới đây là mã nguồn SQL DDL hoàn chỉnh cho toàn bộ hệ thống VRMS:

```sql
-- =========================================================================
-- PHẦN 1: KHỞI TẠO EXTENSIONS & CUSTOM ENUMS
-- =========================================================================

-- Kích hoạt extension pgcrypto để tự động sinh UUIDv4 bằng gen_random_uuid()
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- 1. ENUM vai trò người dùng
CREATE TYPE user_role AS ENUM ('DRIVER', 'MECHANIC', 'TECH', 'INVENTORY', 'MANAGER');

-- 2. ENUM trạng thái hoạt động tài khoản
CREATE TYPE user_status AS ENUM ('ACTIVE', 'INACTIVE');

-- 3. ENUM chủng loại phương tiện
CREATE TYPE vehicle_type AS ENUM ('INTERNAL_TRUCK', 'INTERNAL_VAN', 'EXTERNAL_CLIENT');

-- 4. ENUM trạng thái kỹ thuật phương tiện
CREATE TYPE vehicle_status AS ENUM ('ACTIVE', 'BROKEN', 'REPAIRING');

-- 5. ENUM vòng đời phiếu sửa chữa
CREATE TYPE ticket_status AS ENUM ('REPORTED', 'INSPECTING', 'WAITING_QUEUE', 'WAITING_PARTS', 'REPAIRING', 'COMPLETED', 'CLOSED', 'REJECTED');

-- 6. ENUM mức độ ưu tiên
CREATE TYPE ticket_priority AS ENUM ('LOW', 'MEDIUM', 'HIGH');

-- 7. ENUM phân loại đợt kiểm định
CREATE TYPE inspection_type AS ENUM ('INITIAL_INSPECTION', 'TECHNICAL_DIAGNOSIS', 'FINAL_ACCEPTANCE');

-- 8. ENUM quyết định xử lý kiểm định
CREATE TYPE inspection_decision AS ENUM ('APPROVE', 'REJECT', 'QUEUE', 'REPAIR_AGAIN');

-- 9. ENUM trạng thái hàng chờ
CREATE TYPE queue_entry_status AS ENUM ('WAITING', 'PROMOTED', 'CANCELLED');

-- 10. ENUM trạng thái kho hàng
CREATE TYPE warehouse_status AS ENUM ('ACTIVE', 'INACTIVE');

-- 11. ENUM trạng thái phê duyệt vật tư
CREATE TYPE material_request_status AS ENUM ('PENDING', 'APPROVED', 'PARTIAL', 'ISSUED', 'RECEIVED', 'REJECTED');

-- 12. ENUM trạng thái thi công của KTV
CREATE TYPE repair_job_status AS ENUM ('REPAIRING', 'COMPLETED');

-- 13. ENUM phân loại thông báo
CREATE TYPE notification_type AS ENUM ('TICKET_UPDATED', 'QUEUE_PROMOTED', 'PARTS_READY', 'SYSTEM_ALERT');

-- 14. ENUM trạng thái hoạt động cầu sửa xe
CREATE TYPE slot_status AS ENUM ('VACANT', 'OCCUPIED', 'MAINTENANCE');

-- 15. ENUM trạng thái gửi thông báo đẩy
CREATE TYPE delivery_status AS ENUM ('PENDING', 'SENT', 'FAILED');


-- =========================================================================
-- PHẦN 2: KHỞI TẠO CÁC BẢNG DANH MỤC (MASTER DATA TABLES)
-- =========================================================================

-- Bảng 1: users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_code VARCHAR(20) NOT NULL UNIQUE,
    full_name VARCHAR(100) NOT NULL,
    phone VARCHAR(20) NOT NULL UNIQUE,
    role user_role NOT NULL,
    status user_status NOT NULL DEFAULT 'ACTIVE',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_employee_code CHECK (employee_code ~ '^EMP-[0-9]{5}$'),
    CONSTRAINT chk_user_phone CHECK (phone ~ '^\+?[0-9]{10,15}$')
);

-- Bảng 2: vehicles
CREATE TABLE vehicles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_code VARCHAR(50) NOT NULL UNIQUE,
    plate_number VARCHAR(20) NOT NULL UNIQUE,
    type vehicle_type NOT NULL,
    status vehicle_status NOT NULL DEFAULT 'ACTIVE',
    is_external BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_vehicle_code CHECK (vehicle_code ~ '^VEH-[0-9]{5}$'),
    CONSTRAINT chk_plate_number CHECK (plate_number ~ '^[0-9]{2}[A-Z]-[0-9]{4,5}$')
);

-- Bảng 3: warehouses
CREATE TABLE warehouses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_name VARCHAR(100) NOT NULL UNIQUE,
    location VARCHAR(255) NOT NULL,
    status warehouse_status NOT NULL DEFAULT 'ACTIVE',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Bảng 4: materials
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sku VARCHAR(50) NOT NULL UNIQUE,
    name VARCHAR(100) NOT NULL,
    category VARCHAR(50) NOT NULL,
    unit VARCHAR(20) NOT NULL,
    reorder_level INT NOT NULL DEFAULT 10,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_reorder_level CHECK (reorder_level >= 0),
    CONSTRAINT chk_material_unit CHECK (unit IN ('Cái', 'Lít', 'Bộ', 'Cặp', 'Hộp'))
);

-- Bảng 5: workshop_slots
CREATE TABLE workshop_slots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slot_name VARCHAR(50) NOT NULL UNIQUE,
    status slot_status NOT NULL DEFAULT 'VACANT',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);


-- =========================================================================
-- PHẦN 3: KHỞI TẠO CÁC BẢNG GIAO DỊCH (TRANSACTIONAL & JUNCTION TABLES)
-- =========================================================================

-- Bảng 6: vehicle_assignments
CREATE TABLE vehicle_assignments (
    id SERIAL PRIMARY KEY,
    vehicle_id UUID NOT NULL REFERENCES vehicles(id) ON DELETE CASCADE,
    driver_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    released_at TIMESTAMPTZ,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_assignment_dates CHECK (released_at IS NULL OR released_at >= assigned_at)
);

-- Bảng 7: repair_tickets
CREATE TABLE repair_tickets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_no VARCHAR(50) NOT NULL UNIQUE,
    vehicle_id UUID NOT NULL REFERENCES vehicles(id) ON DELETE RESTRICT,
    driver_id UUID REFERENCES users(id) ON DELETE SET NULL,
    guest_owner_name VARCHAR(100),
    guest_owner_phone VARCHAR(20),
    status ticket_status NOT NULL DEFAULT 'REPORTED',
    priority ticket_priority NOT NULL DEFAULT 'MEDIUM',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_ticket_owner CHECK (
        (driver_id IS NOT NULL AND guest_owner_name IS NULL AND guest_owner_phone IS NULL)
        OR
        (driver_id IS NULL AND guest_owner_name IS NOT NULL AND guest_owner_phone IS NOT NULL)
    ),
    CONSTRAINT chk_ticket_no CHECK (ticket_no ~ '^RT-\d{8}-\d{4}$'),
    CONSTRAINT chk_guest_phone CHECK (guest_owner_phone IS NULL OR guest_owner_phone ~ '^\+?[0-9]{10,15}$')
);

-- Bảng 8: ticket_issues
CREATE TABLE ticket_issues (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_id UUID NOT NULL REFERENCES repair_tickets(id) ON DELETE CASCADE,
    issue_type VARCHAR(50) NOT NULL,
    description TEXT NOT NULL,
    severity ticket_priority NOT NULL DEFAULT 'MEDIUM',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Bảng 9: attachments
-- Sử dụng quan hệ đa hình (polymorphic relationship) liên kết với TICKET, INSPECTION, hoặc REPAIR.
-- Referential Integrity được kiểm soát và xử lý ở tầng Application Service Layer.
CREATE TABLE attachments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_name VARCHAR(255) NOT NULL,
    mime_type VARCHAR(100) NOT NULL,
    size_bytes BIGINT NOT NULL,
    storage_key VARCHAR(255) NOT NULL UNIQUE,
    file_url VARCHAR(512) NOT NULL,
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    uploaded_by UUID REFERENCES users(id) ON DELETE SET NULL,
    uploaded_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_file_size CHECK (size_bytes > 0),
    CONSTRAINT chk_entity_type CHECK (entity_type IN ('TICKET', 'INSPECTION', 'REPAIR'))
);

-- Bảng 10: inventory_stock (Junction table giữa materials và warehouses)
CREATE TABLE inventory_stock (
    material_id UUID NOT NULL REFERENCES materials(id) ON DELETE CASCADE,
    warehouse_id UUID NOT NULL REFERENCES warehouses(id) ON DELETE CASCADE,
    quantity_on_hand INT NOT NULL DEFAULT 0,
    quantity_reserved INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (material_id, warehouse_id),
    CONSTRAINT chk_stock_quantities CHECK (quantity_on_hand >= quantity_reserved),
    CONSTRAINT chk_on_hand_positive CHECK (quantity_on_hand >= 0),
    CONSTRAINT chk_reserved_positive CHECK (quantity_reserved >= 0)
);

-- Bảng 11: material_requests
CREATE TABLE material_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_id UUID NOT NULL REFERENCES repair_tickets(id) ON DELETE RESTRICT,
    requester_id UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    status material_request_status NOT NULL DEFAULT 'PENDING',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Bảng 12: material_request_items
CREATE TABLE material_request_items (
    id BIGSERIAL PRIMARY KEY,
    request_id UUID NOT NULL REFERENCES material_requests(id) ON DELETE CASCADE,
    material_id UUID NOT NULL REFERENCES materials(id) ON DELETE RESTRICT,
    quantity_requested INT NOT NULL,
    quantity_approved INT NOT NULL DEFAULT 0,
    quantity_issued INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_req_qty CHECK (quantity_requested > 0),
    CONSTRAINT chk_app_qty CHECK (quantity_approved >= 0),
    CONSTRAINT chk_iss_qty CHECK (quantity_issued >= 0),
    CONSTRAINT chk_issued_limit CHECK (quantity_issued <= quantity_approved)
);

-- Bảng 13: queue_entries
CREATE TABLE queue_entries (
    id BIGSERIAL PRIMARY KEY,
    ticket_id UUID NOT NULL REFERENCES repair_tickets(id) ON DELETE CASCADE,
    position INT NOT NULL,
    priority SMALLINT NOT NULL DEFAULT 0,
    entered_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    status queue_entry_status NOT NULL DEFAULT 'WAITING',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_queue_position CHECK (position > 0),
    CONSTRAINT chk_queue_priority CHECK (priority >= 0)
);

-- Bảng 14: inspection_records
CREATE TABLE inspection_records (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_id UUID NOT NULL REFERENCES repair_tickets(id) ON DELETE CASCADE,
    inspection_type inspection_type NOT NULL,
    inspected_by UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    findings TEXT NOT NULL,
    decision inspection_decision NOT NULL,
    inspected_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Bảng 15: workshop_slot_assignments
CREATE TABLE workshop_slot_assignments (
    id SERIAL PRIMARY KEY,
    slot_id UUID NOT NULL REFERENCES workshop_slots(id) ON DELETE CASCADE,
    ticket_id UUID NOT NULL REFERENCES repair_tickets(id) ON DELETE CASCADE,
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    released_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_slot_dates CHECK (released_at IS NULL OR released_at >= assigned_at)
);

-- Bảng 16: repair_jobs
CREATE TABLE repair_jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_id UUID NOT NULL REFERENCES repair_tickets(id) ON DELETE CASCADE,
    technician_id UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ,
    notes TEXT,
    status repair_job_status NOT NULL DEFAULT 'REPAIRING',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_job_dates CHECK (completed_at IS NULL OR completed_at >= started_at)
);

-- Bảng 17: notifications
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    notification_type notification_type NOT NULL,
    title VARCHAR(150) NOT NULL,
    body TEXT NOT NULL,
    is_read BOOLEAN NOT NULL DEFAULT FALSE,
    delivery_status delivery_status NOT NULL DEFAULT 'PENDING',
    sent_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);


-- =========================================================================
-- PHẦN 4: PHÂN VÙNG BẢNG KIỂM TOÁN (PARTITIONED AUDIT LOGS)
-- =========================================================================

-- Bảng 18: audit_logs (Phân vùng theo tháng trên trường created_at)
CREATE TABLE audit_logs (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY,
    actor_id UUID REFERENCES users(id) ON DELETE SET NULL,
    action VARCHAR(100) NOT NULL,
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    old_values JSONB,
    new_values JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- Thiết lập các bảng phân vùng vật lý mẫu theo tháng trong năm 2026
CREATE TABLE audit_logs_y2026m06 PARTITION OF audit_logs
    FOR VALUES FROM ('2026-06-01 00:00:00+00') TO ('2026-07-01 00:00:00+00');

CREATE TABLE audit_logs_y2026m07 PARTITION OF audit_logs
    FOR VALUES FROM ('2026-07-01 00:00:00+00') TO ('2026-08-01 00:00:00+00');

CREATE TABLE audit_logs_y2026m08 PARTITION OF audit_logs
    FOR VALUES FROM ('2026-08-01 00:00:00+00') TO ('2026-09-01 00:00:00+00');


-- =========================================================================
-- PHẦN 5: KHỞI TẠO CÁC CHỈ MỤC (INDEXES & PARTIAL INDEXES)
-- =========================================================================

-- 1. Chỉ mục cho các trường định danh thường xuyên tìm kiếm (Master Keys)
CREATE INDEX idx_users_employee_code ON users(employee_code);
CREATE INDEX idx_vehicles_plate_number ON vehicles(plate_number);
CREATE INDEX idx_materials_sku ON materials(sku);

-- 2. Chỉ mục Khóa ngoại (Foreign Key Indexes) để tối ưu hóa truy vấn JOIN
CREATE INDEX idx_vehicle_assignments_fk ON vehicle_assignments(vehicle_id, driver_id);
CREATE INDEX idx_repair_tickets_vehicle ON repair_tickets(vehicle_id);
CREATE INDEX idx_repair_tickets_driver ON repair_tickets(driver_id);
CREATE INDEX idx_ticket_issues_ticket ON ticket_issues(ticket_id);
CREATE INDEX idx_attachments_polymorphic ON attachments(entity_type, entity_id);
CREATE INDEX idx_material_requests_ticket ON material_requests(ticket_id);
CREATE INDEX idx_material_requests_requester ON material_requests(requester_id);
CREATE INDEX idx_material_request_items_request ON material_request_items(request_id);
CREATE INDEX idx_queue_entries_ticket ON queue_entries(ticket_id);
CREATE INDEX idx_inspection_records_ticket ON inspection_records(ticket_id);
CREATE INDEX idx_repair_jobs_ticket ON repair_jobs(ticket_id);
CREATE INDEX idx_repair_jobs_tech ON repair_jobs(technician_id);
CREATE INDEX idx_notifications_user ON notifications(user_id);

-- 3. Chỉ mục một phần độc nhất (Partial Unique Indexes) bảo vệ logic nghiệp vụ
-- Ràng buộc 1 cầu nâng tại một thời điểm chỉ chứa tối đa 1 xe active đỗ
CREATE UNIQUE INDEX unique_active_slot_assignment 
ON workshop_slot_assignments(slot_id) 
WHERE (released_at IS NULL);

-- Ràng buộc 1 xe tại một thời điểm chỉ được đỗ ở tối đa 1 cầu nâng active
CREATE UNIQUE INDEX unique_active_ticket_assignment 
ON workshop_slot_assignments(ticket_id) 
WHERE (released_at IS NULL);

-- Ràng buộc một phiếu sửa chữa tại một thời điểm chỉ được xếp hàng chờ 1 lần
CREATE UNIQUE INDEX unique_active_queue_ticket 
ON queue_entries(ticket_id) 
WHERE (status = 'WAITING');


-- =========================================================================
-- PHẦN 6: PL/PGSQL TRIGGERS TỰ ĐỘNG CẬP NHẬT UPDATED_AT
-- =========================================================================

-- Hàm PL/pgSQL thực thi cập nhật thời gian
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Gán trigger cho các bảng có trường updated_at
CREATE TRIGGER trg_update_users_modtime BEFORE UPDATE ON users 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_update_vehicles_modtime BEFORE UPDATE ON vehicles 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_update_warehouses_modtime BEFORE UPDATE ON warehouses 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_update_materials_modtime BEFORE UPDATE ON materials 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_update_workshop_slots_modtime BEFORE UPDATE ON workshop_slots 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_update_vehicle_assignments_modtime BEFORE UPDATE ON vehicle_assignments 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_update_repair_tickets_modtime BEFORE UPDATE ON repair_tickets 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_update_ticket_issues_modtime BEFORE UPDATE ON ticket_issues 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_update_attachments_modtime BEFORE UPDATE ON attachments 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_update_inventory_stock_modtime BEFORE UPDATE ON inventory_stock 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_update_material_requests_modtime BEFORE UPDATE ON material_requests 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_update_material_request_items_modtime BEFORE UPDATE ON material_request_items 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_update_queue_entries_modtime BEFORE UPDATE ON queue_entries 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_update_inspection_records_modtime BEFORE UPDATE ON inspection_records 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_update_workshop_slot_assignments_modtime BEFORE UPDATE ON workshop_slot_assignments 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_update_repair_jobs_modtime BEFORE UPDATE ON repair_jobs 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_update_notifications_modtime BEFORE UPDATE ON notifications 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

---

## 15.6 Schema Evolution & Migration Strategy

Để quản lý và triển khai lược đồ cơ sở dữ liệu trên môi trường phát triển và vận hành (Production), hệ thống quy định chiến lược quản lý phiên bản (Migration Strategy) như sau:

### 1. Công cụ quản lý di chuyển (Migration Tooling)
- **Prisma ORM**: Được sử dụng làm công cụ chính trong vòng đời phát triển của Backend (NestJS). Tập tin `schema.prisma` đóng vai trò là "Single Source of Truth".
- **Lệnh tạo migration**:
  ```bash
  npx prisma migrate dev --name init_vrms_schema
  ```
- **Triển khai Production**:
  ```bash
  npx prisma migrate deploy
  ```

### 2. Chiến lược thay đổi và Rollback (Evolution & Rollback Policy)
- **Zero Downtime**: Mọi thay đổi cấu trúc bảng trên môi trường chạy thực tế (Production) phải tuân thủ nguyên tắc mở rộng trước, thu hẹp sau (Expand and Contract pattern). Không đổi tên cột hay xóa cột trực tiếp; thay vào đó, chèn thêm cột mới, di chuyển dữ liệu (data backfill) và tiến hành loại bỏ cột cũ ở phiên bản tiếp theo.
- **Rollback SQL**: Với mỗi tệp SQL Migration sinh ra (ví dụ: `migration.sql`), lập trình viên chịu trách nhiệm viết kèm tệp rollback thủ công (ví dụ: `rollback.sql`) để đưa cấu trúc dữ liệu trở về phiên bản trước đó trong trường hợp xảy ra sự cố triển khai.
- **Quản lý phân vùng tự động (Partition Management)**: Hệ thống sử dụng một tác nhân dịch vụ chạy ngầm (`pg_partman` hoặc scheduler tự động chạy hàng tháng của hệ thống) để tạo trước các phân vùng `audit_logs` của các tháng tiếp theo, tránh lỗi chèn dữ liệu khi bước sang tháng mới.

---

## Outputs
- File kịch bản DDL SQL hoàn chỉnh: [15_database_schema.md](file:///c:/Users/Legion/Desktop/IT/SNP/.agents/skills/system_engineering_copilot/deliverables/current/sss/part_c_data_design/15_database_schema.md) đã được hoàn thiện cấu trúc, kiểu dữ liệu, các ràng buộc và chiến lược di chuyển.
