# FixTrack — Current System Baseline

**Trạng thái:** Hoàn thành kiểm kê — Giai đoạn 0  
**Ngày kiểm kê:** 2026-07-11  
**Phạm vi:** Chỉ đọc và đối chiếu repository; chưa thay đổi logic backend.

## 1. Baseline kỹ thuật thực tế

| Thành phần | Công nghệ đang dùng |
|---|---|
| Backend | FastAPI 0.115.6, Python 3.11+ |
| ORM | SQLAlchemy 2.0.36 |
| Migration | Alembic 1.14.0 |
| Database production | PostgreSQL 15 (Docker image `postgres:15-alpine`) |
| Database test/dev | SQLite được hỗ trợ trong cấu hình và test |
| Validation | Pydantic 2.10.4 |
| Authentication | JWT access token, Passlib/Bcrypt |
| Frontend | Next.js 14.2.15, React 18, TypeScript, Tailwind CSS |
| Test backend | Pytest 8.2.2, HTTPX 0.27.0 |
| Runtime services trong Compose | PostgreSQL, FastAPI backend, Next.js frontend |

NestJS, Prisma, TypeORM, BullMQ, Redis worker, S3/R2 và ClamAV không thuộc runtime hiện tại. Chúng chỉ được xem là kiến trúc dự kiến cho đến khi có implementation và quyết định triển khai riêng.

## 2. Enum canonical đang tồn tại trong code

### Vai trò

| Enum | Ý nghĩa tạm thời |
|---|---|
| `lai_xe` | Driver |
| `co_gioi` | Fleet Dispatcher / Vehicle Coordinator; tiếp nhận và nghiệm thu |
| `ky_thuat` | Repair Technician |
| `vat_tu` | Inventory Staff |
| `admin` | Role kỹ thuật hiện đang gộp Manager và System Administrator |

Lưu ý: `kho_vat_tu` xuất hiện trong một số tài liệu/UI nhưng không phải enum backend. Không đổi enum khi chưa có migration và kế hoạch chuyển đổi dữ liệu.

### Trạng thái phiếu sửa chữa

`reported`, `inspecting`, `waiting_queue`, `waiting_parts`, `repairing`, `completed`, `closed`, `rejected`.

### Trạng thái yêu cầu vật tư

`cho_duyet`, `da_xuat`, `da_nhan_tai_xe`, `da_ban_giao`, `tu_choi`.

## 3. State machine đang được backend thực thi

Các transition hiện có trong `app/services/repair.py`:

| Từ | Hành động | Đến | Role thông thường |
|---|---|---|---|
| `reported` | inspect | `inspecting` | `co_gioi` |
| `reported` | reject | `rejected` | `co_gioi` |
| `inspecting` | reject | `rejected` | `co_gioi` |
| `inspecting` | route_queue | `waiting_queue` | `co_gioi` |
| `inspecting` | start_repair | `repairing` | `ky_thuat` |
| `waiting_queue` | inspect | `inspecting` | `co_gioi` |
| `waiting_parts` | start_repair | `repairing` | `ky_thuat` |
| `repairing` | complete_repair | `completed` | `ky_thuat` |
| `completed` | accept_repair | `closed` | `co_gioi` |
| `completed` | reject_repair | `repairing` | `co_gioi` |

`admin` hiện được phép override các role trong service này.

### Điểm không nhất quán trong implementation

- Material service tự gán `inspecting → waiting_parts`, không đi qua state-machine service.
- Transition trên không tăng `RepairRequest.version` và không áp dụng đầy đủ side effect chung.
- Queue hiện đưa ticket `waiting_queue → inspecting`, trong khi SSS có nơi mô tả `waiting_queue → repairing`.
- Chưa có `repairing → waiting_parts` khi phát sinh thêm nhu cầu vật tư.
- Route và service khác vẫn có các vị trí thay đổi trạng thái trực tiếp.

## 4. Mô hình dữ liệu liên quan

- `RepairRequest` có trạng thái, mức ưu tiên, OCC `version`, status logs, issues, work assignments và material requests.
- `MaterialRequestItem` chỉ có `so_luong_yeu_cau` và `so_luong_cap`.
- Chưa có cột `delivered_qty` hoặc `returned_qty`; lượng trả hiện phải suy ra từ inventory ledger.
- `InventoryTransaction` có unique `idempotency_key`, kho, số lượng và `balance_after`.
- `WorkAssignment` liên kết ticket với technician nhưng chưa có cột trạng thái active riêng; thời điểm hoàn tất được biểu diễn bằng `ngay_hoan_tat`.
- `VehicleAssignment` có partial unique indexes trên PostgreSQL để bảo đảm một xe/một tài xế chỉ có một assignment active.
- `AuditLog` có SHA-256 hash chain và trace ID; migration và test kiểm tra tampering đã tồn tại.
- Chưa có outbox table hoặc asynchronous worker trong runtime hiện tại.

## 5. Các rủi ro P0 đã xác nhận

### Security và ownership

- Branch policy đang fail-open khi user hoặc vehicle thiếu branch.
- Material request list/detail chưa lọc theo owner, assignment hoặc branch.
- Tạo material request chưa xác nhận assignment thuộc đúng ticket.
- Xác nhận handover chưa xác nhận KTV là người được phân công.
- Một số authorization chỉ nằm ở route dependency, chưa được bảo vệ lại ở service layer.

### Inventory integrity

- Có thể cấp vượt `so_luong_yeu_cau`.
- Payload duyệt có thể chứa item ID không thuộc phiếu mà không bị từ chối rõ ràng.
- Hoàn trả chưa chặn vượt số đã cấp trừ số đã trả.
- Hoàn trả chưa buộc đúng trạng thái xuất/bàn giao và đúng kho xuất.
- Idempotency hoàn trả dùng prefix theo toàn phiếu, làm chặn các lần trả từng phần hợp lệ tiếp theo.

### State consistency

- Không phải mọi thay đổi trạng thái đều qua state-machine service.
- OCC version, audit/status log và event chưa đồng nhất ở mọi transition.
- Ý nghĩa nghiệp vụ của `waiting_queue` chưa thống nhất giữa code và SSS.

## 6. Hiện trạng test

Repository có test cho:

- Nhập kho và ledger.
- Hoàn trả vật tư cơ bản.
- Queue priority.
- Transition không hợp lệ.
- Cross-branch trên một route sửa chữa.
- OCC conflict.
- Idempotent inventory receipt.
- Audit authorization và audit hash-chain tampering.

Thiếu test bắt buộc cho:

- Branch fail-closed khi thiếu branch.
- Material request ABAC list/detail.
- KTV không được phân công và assignment sai ticket.
- Cấp vượt yêu cầu.
- Trả vượt, trả nhiều lần và trả sai kho.
- Concurrent stock issue.
- Atomic rollback của stock, audit và ticket state.
- Mọi transition canonical theo role/action matrix.

Test chưa chạy được trong phiên kiểm kê vì máy chủ lệnh hiện không tìm thấy executable `python` hoặc `pytest`, dù dependency đã được khai báo trong `requirements.txt`. Cần dùng Docker hoặc cấu hình lại Python runtime trước Giai đoạn 2.

## 7. Chênh lệch chính giữa implementation và SSS

- SSS phần lớn mô tả NestJS/Prisma/TypeORM/BullMQ, khác FastAPI implementation.
- SSS có nhiều state machine và cách đặt tên role/status khác nhau.
- Business Rules vẫn là Draft và còn các TBD nhưng master dossier tuyên bố Finalized/Production-ready.
- PostgreSQL version trong SSS là 16+, runtime hiện tại là 15.
- Nhiều liên kết tài liệu còn trỏ tới dự án cũ `SNP` hoặc đường dẫn tuyệt đối.
- Redis, S3, FCM, ClamAV, React Native và worker chưa có trong runtime hiện tại.

## 8. Quyết định cần chốt ở Giai đoạn 1

1. `waiting_queue` là chờ kiểm tra hay chờ slot sửa sau khi đã kiểm tra?
2. Có giữ transition `reported → rejected` hay bắt buộc qua `inspecting`?
3. Có cho `repairing → waiting_parts` khi phát sinh vật tư không?
4. Khi vật tư bị từ chối, ai được xác nhận vật tư không còn bắt buộc để tiếp tục sửa?
5. Partial approval/delivery/return thuộc MVP hay P1?
6. `admin` tiếp tục gộp Manager và System Administrator trong phiên bản hiện tại hay phải tách role?
7. `waiting_parts` có tác động tự động tới workshop slot trong MVP hay chỉ là trạng thái nghiệp vụ?

## 9. Phạm vi file dự kiến bị ảnh hưởng

### P0 Security

- `backend/app/services/branch_policy.py`
- `backend/app/api/routes/material_requests.py`
- `backend/app/services/material.py`
- `backend/app/api/routes/work_assignments.py`
- Các schema và test tương ứng

### P0 Inventory

- `backend/app/services/material.py`
- `backend/app/api/routes/material_requests.py`
- `backend/app/models/material_request.py`
- `backend/app/models/material_request_item.py`
- `backend/app/models/inventory_transaction.py`
- Alembic migration nếu cần lưu kho xuất/số lượng giao/trả
- Test inventory integration

### P0 State Machine

- `backend/app/services/repair.py`
- `backend/app/services/material.py`
- `backend/app/api/routes/repair_requests.py`
- `backend/app/api/routes/work_assignments.py`
- `backend/app/api/routes/queue.py`
- Test state transition và transaction

## 10. Kết luận Giai đoạn 0

Backend hiện tại là baseline FastAPI có đủ nền tảng model, migration, state machine, inventory ledger, OCC và audit hash chain. Tuy nhiên, chưa nên sửa logic trước khi chốt bảy quyết định nghiệp vụ tại mục 8. Không có thay đổi code hoặc database nào được thực hiện trong Giai đoạn 0.
