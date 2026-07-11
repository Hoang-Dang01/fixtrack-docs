# 07 Functional Requirements

## Purpose
Chương này đặc tả chi tiết các yêu cầu chức năng (Functional Requirements - FR) của hệ thống Quản Lý Sửa Chữa Phương Tiện (VRMS), phân rã từ luồng nghiệp vụ 7 bước và ma trận phân quyền người dùng. Đây là cơ sở để đội ngũ Phát triển thiết kế giao diện (UI) và lập trình dịch vụ Backend (API).

## Questions Answered
- Hệ thống cần cung cấp các chức năng cụ thể nào cho từng vai trò người dùng (Lái xe, Đội cơ giới, Đội kỹ thuật, Kho vật tư, Quản lý)?
- Các Module chức năng chính trong hệ thống hoạt động như thế nào và có những ràng buộc nghiệp vụ gì?
- Các yêu cầu chức năng được chuẩn hóa dưới mã nhận diện (FR-ID) nào để phục vụ việc kiểm thử (Test Case mapping) sau này?

## Inputs
- [05 User Roles](file:///c:/Users/Legion/Desktop/IT/SNP/.agents/skills/system_engineering_copilot/deliverables/current/sss/part_a_business_foundation/05_user_roles.md) (Quyền hạn và ma trận phân quyền).
- [06 Workflow Analysis](file:///c:/Users/Legion/Desktop/IT/SNP/.agents/skills/system_engineering_copilot/deliverables/current/sss/part_b_requirement_analysis/06_workflow_analysis.md) (7 bước nghiệp vụ số hóa).

---

## Content

### Level 1 — Summary
Hệ thống VRMS cung cấp các tính năng được nhóm thành 9 Module chức năng chính trên ứng dụng di động:
1. **Module Xác thực (Auth)**: Đảm bảo đăng nhập an toàn và tự động nhận diện vai trò người dùng để điều hướng giao diện phù hợp.
2. **Module Phương tiện (Vehicle)**: Quản lý danh mục phương tiện nội bộ và bên ngoài cùng trạng thái kỹ thuật của chúng.
3. **Module Phân công phương tiện (Vehicle Assignment)**: Hỗ trợ Đội cơ giới điều phối gán phương tiện cho Lái xe.
4. **Module Phiếu báo hỏng (Repair Ticket)**: Cho phép Lái xe tạo phiếu báo hỏng gồm nhiều lỗi và đính kèm hình ảnh.
5. **Module Tiếp nhận & Kiểm tra sơ bộ (Inspection)**: Công cụ để Đội cơ giới sàng lọc và định tuyến xe.
6. **Module Hàng chờ xưởng (Queue)**: Tự động điều hành hàng xe chờ sửa tại xưởng.
7. **Module Yêu cầu & Cấp phát vật tư (Material Request)**: Quản lý khép kín luồng yêu cầu vật tư từ KTV, Kho xuất, Lái xe nhận và bàn giao.
8. **Module Thực hiện sửa chữa (Repair Execution)**: Ghi nhận thời gian, tiến độ và công việc sửa chữa thực tế của KTV.
9. **Module Báo cáo & Giám sát (Dashboard & Reporting)**: Bảng điều khiển theo dõi KPI dành cho Quản lý.

---

### Level 2 — Breakdown
Từng Module chức năng lớn trong ứng dụng được phân rã thành các chức năng con độc lập (Atomic Functional Requirements) để thuận tiện cho việc phát triển và kiểm thử. Dưới đây là danh sách các yêu cầu chức năng:

- **Module Xác thực & Phân quyền (Auth & RBAC)**:
  - `FR-AUTH-01`: Đăng nhập bằng số điện thoại/tài khoản nội bộ.
  - `FR-AUTH-02`: Đăng xuất và xóa phiên làm việc local.
- **Module Phương tiện (Vehicle)**:
  - `FR-VEH-01`: Xem thông tin xe và lịch sử sửa chữa trong quá khứ.
- **Module Phân công phương tiện (Vehicle Assignment)**:
  - `FR-ASSIGN-01`: Phân công xe cho tài xế theo ca.
  - `FR-ASSIGN-02`: Thu hồi phân công xe sau khi kết thúc ca.
- **Module Phiếu báo hỏng (Repair Ticket)**:
  - `FR-TICKET-01`: Tạo phiếu báo sự cố cho phương tiện đang lái.
  - `FR-TICKET-02`: Khai báo nhiều lỗi trong một phiếu báo sự cố.
  - `FR-TICKET-03`: Tải hình ảnh/video hiện trường lỗi (tối đa 3 file).
  - `FR-TICKET-04`: Theo dõi trạng thái của phiếu sửa chữa thời gian thực.
- **Module Tiếp nhận & Kiểm tra sơ bộ (Inspection/Triage)**:
  - `FR-INSP-01`: Tiếp nhận phiếu báo sự cố của tài xế.
  - `FR-INSP-02`: Kiểm tra sơ bộ ban đầu và phân loại mức độ lỗi.
  - `FR-INSP-03`: Phê duyệt chuyển xưởng sửa chữa hoặc từ chối phiếu.
- **Module Hàng chờ xưởng (Queue)**:
  - `FR-QUEUE-01`: Tự động xếp xe vào hàng chờ sửa chữa (FIFO).
  - `FR-QUEUE-02`: Xem số thứ tự chờ sửa chữa thực tế của xe.
  - `FR-QUEUE-03`: Thay đổi thứ tự ưu tiên trong hàng chờ (Chỉ dành cho Quản lý).
- **Module Yêu cầu & Cấp phát vật tư (Material/Inventory)**:
  - `FR-MAT-01`: Tìm kiếm vật tư trong kho theo SKU/Tên và xem số lượng tồn kho khả dụng.
  - `FR-MAT-02`: Kỹ thuật viên (KTV) lập yêu cầu vật tư thay thế cho xe.
  - `FR-MAT-03`: Thủ kho duyệt yêu cầu và xuất phụ tùng vật lý khỏi kho.
  - `FR-MAT-04`: Lái xe xác nhận đã nhận đủ vật tư từ thủ kho.
  - `FR-MAT-05`: KTV xác nhận đã nhận bàn giao vật tư từ lái xe tại xưởng.
- **Module Thực hiện sửa chữa (Repair Execution)**:
  - `FR-REP-01`: Ghi nhận thời điểm bắt đầu sửa xe (Hệ thống đếm giờ).
  - `FR-REP-02`: Cập nhật tiến độ sửa chữa và ghi chú kỹ thuật.
  - `FR-REP-03`: Ghi nhận hoàn tất sửa chữa và chuyển sang chờ nghiệm thu.
  - `FR-REP-04`: Nghiệm thu kết quả sửa chữa (Mechanic kiểm tra đạt/không đạt).
- **Module Báo cáo & Giám sát (Dashboard & Reporting)**:
  - `FR-DASH-01`: Xem số liệu xe đang hoạt động, xe hỏng và xe đang sửa thời gian thực.
  - `FR-DASH-02`: Xem báo cáo thống kê downtime, hiệu suất KTV và linh kiện hao phí.

---

### Level 3 — Technical Detail (Đặc tả chi tiết các yêu cầu chức năng Atomic)

#### FR-AUTH-01 — Đăng nhập hệ thống
- **Người dùng**:
  - Tất cả vai trò.
- **Điều kiện**:
  - Tài khoản đã được tạo trên hệ thống nội bộ.
- **Thao tác**:
  - Nhập số điện thoại/tài khoản và mật khẩu.
  - Bấm "Đăng nhập".
- **Kết quả**:
  - Hệ thống xác thực thành công.
  - Backend trả về JWT Token chứa thông tin vai trò (`DRIVER`, `MECHANIC`, `TECHNICIAN`, `INVENTORY`, `MANAGER`).
  - Giao diện tự động điều hướng sang màn hình chức năng phù hợp của vai trò đó.
- **Trường hợp lỗi**:
  - Nhập sai mật khẩu hoặc tài khoản: Hệ thống hiển thị "Tài khoản hoặc mật khẩu không chính xác". Khóa tài khoản tạm thời nếu nhập sai quá 5 lần liên tiếp.
  - Tài khoản bị vô hiệu hóa: Hệ thống thông báo "Tài khoản của bạn đã bị khóa. Vui lòng liên hệ Admin".
  - Mất kết nối internet: Hệ thống báo lỗi kết nối mạng.

#### FR-ASSIGN-01 — Phân công xe cho tài xế
- **Người dùng**:
  - Đội cơ giới (Điều hành).
- **Điều kiện**:
  - Đã đăng nhập.
  - Phương tiện được chọn đang ở trạng thái rảnh (`ACTIVE`).
  - Tài xế được chọn đang ở trạng thái rảnh (chưa được gán xe nào khác).
- **Thao tác**:
  - Chọn xe từ danh sách xe rảnh.
  - Chọn tài xế từ danh sách tài xế rảnh.
  - Nhấn "Xác nhận gán".
- **Kết quả**:
  - Ghi nhận liên kết xe - tài xế trong cơ sở dữ liệu (`vehicle_assignments`).
  - Gửi thông báo push tới tài xế: "Bạn đã được gán xe [Biển số]".
- **Trường hợp lỗi**:
  - Xe đang hỏng hoặc đang sửa: Ẩn nút gán xe.
  - Tài xế đã được gán xe khác: Hệ thống báo lỗi "Tài xế đang vận hành xe khác".

#### FR-TICKET-01 — Tạo phiếu báo sự cố hư hỏng
- **Người dùng**:
  - Lái xe.
- **Điều kiện**:
  - Đã đăng nhập.
  - Đang được phân công lái xe hợp lệ và đang hoạt động (`BR-01`).
- **Thao tác**:
  - Hệ thống tự điền xe đang gán, lái xe chọn "Báo lỗi".
  - Chọn danh mục lỗi, mức độ nghiêm trọng và nhập mô tả chi tiết lỗi.
  - Chụp ảnh hoặc quay video hiện trường sự cố.
  - Nhấn "Gửi báo cáo".
- **Kết quả**:
  - Tạo mới phiếu sửa chữa (Status = `REPORTED`).
  - Cập nhật trạng thái xe thành `BROKEN`.
  - Gửi thông báo push real-time cho Đội cơ giới.
- **Trường hợp lỗi**:
  - Không có xe được gán: Hệ thống chặn không cho vào giao diện tạo phiếu.
  - Dung lượng file đính kèm quá lớn (>10MB/file): Hệ thống báo lỗi dung lượng tệp tin.
  - Lỗi kết nối mạng: Lưu nháp phiếu cục bộ trên bộ nhớ máy (Local Storage) và tự động đồng bộ (offline sync) khi có mạng.

#### FR-TICKET-02 — Khai báo nhiều lỗi trong một phiếu báo hỏng
- **Người dùng**:
  - Lái xe.
- **Điều kiện**:
  - Đang ở giao diện tạo phiếu sửa chữa (FR-TICKET-01).
- **Thao tác**:
  - Nhập thông tin lỗi thứ nhất.
  - Bấm nút "+ Thêm lỗi khác" (`BR-03`).
  - Nhập thông tin lỗi thứ hai (mô tả, mức độ và hình ảnh riêng).
  - Bấm gửi sau khi hoàn thành.
- **Kết quả**:
  - Hệ thống lưu 1 ticket cha (`repair_tickets`) và nhiều bản ghi lỗi con (`ticket_issues`).
- **Trường hợp lỗi**:
  - Khai báo quá 5 lỗi cùng lúc: Hệ thống hiển thị cảnh báo "Tối đa chỉ báo 5 lỗi trên cùng một phiếu".

#### FR-INSP-01 — Kiểm tra sơ bộ và phân loại lỗi
- **Người dùng**:
  - Đội cơ giới.
- **Điều kiện**:
  - Đã đăng nhập.
  - Ticket sửa chữa đang ở trạng thái `REPORTED`.
- **Thao tác**:
  - Bấm tiếp nhận ticket (chuyển sang `INSPECTING`).
  - Thực hiện kiểm tra nhanh xe thực tế.
  - Chọn kết quả: "Từ chối" (Reject) hoặc "Chuyển xưởng" (Accept to Garage).
- **Kết quả**:
  - *Nếu Từ chối*: Ticket đóng (`REJECTED`), xe trở lại trạng thái `ACTIVE`.
  - *Nếu Chuyển xưởng*: Hệ thống kiểm tra slot sửa chữa tại xưởng.
- **Trường hợp lỗi**:
  - Không nhập lý do khi Từ chối: Hệ thống yêu cầu bắt buộc nhập lý do từ chối.

#### FR-QUEUE-01 — Tự động xếp hàng xe chờ sửa chữa
- **Người dùng**:
  - Hệ thống (Tự động) / Quản lý (Override).
- **Điều kiện**:
  - Cơ giới duyệt chuyển xưởng nhưng toàn bộ slot sửa chữa trong xưởng đã đầy.
- **Thao tác**:
  - Hệ thống tự động đẩy xe vào hàng chờ.
  - Quản lý có thể chọn xe và bấm "Ưu tiên sửa trước".
- **Kết quả**:
  - Trạng thái ticket chuyển sang `WAITING_QUEUE`.
  - Khi xưởng có xe sửa xong và ra xưởng, xe đứng đầu hàng chờ (FIFO) tự động được gọi vào.
- **Trường hợp lỗi**:
  - Hệ thống mất kết nối DB: Hàng chờ tạm dừng cập nhật vị trí thời gian thực.

#### FR-MAT-02 — Lập yêu cầu cấp phát vật tư phụ tùng
- **Người dùng**:
  - Kỹ thuật viên.
- **Điều kiện**:
  - Đã tiếp nhận xe sửa trong xưởng (Status ticket = `INSPECTING` tại xưởng).
- **Thao tác**:
  - Vào chi tiết xe, chọn "Yêu cầu vật tư".
  - Tìm kiếm và chọn loại phụ tùng, nhập số lượng cần dùng.
  - Nhấn "Gửi yêu cầu".
- **Kết quả**:
  - Tạo yêu cầu vật tư (`material_requests`) ở trạng thái `PENDING`.
  - Gửi thông báo yêu cầu cấp phát tới thủ kho.
- **Trường hợp lỗi**:
  - Số lượng yêu cầu lớn hơn tồn kho hiện tại: Hệ thống báo lỗi đỏ và chặn không cho bấm gửi.

#### FR-MAT-04 — Bàn giao vật tư khép kín 3 bên
- **Người dùng**:
  - Thủ kho, Lái xe, Kỹ thuật viên (KTV).
- **Điều kiện**:
  - Yêu cầu vật tư đang ở trạng thái `PENDING` hoặc `DISBURSED` hoặc `RECEIVED_BY_DRIVER`.
- **Thao tác**:
  - Thủ kho xuất hàng vật lý và bấm "Xác nhận xuất kho" trên app.
  - Lái xe nhận hàng tại kho, bấm "Xác nhận đã nhận vật tư từ kho" trên app.
  - Lái xe mang hàng về xưởng giao cho KTV, KTV bấm "Xác nhận đã nhận vật tư từ lái xe" trên app.
- **Kết quả**:
  - Lần lượt ghi nhận vết bàn giao đầy đủ của 3 người dùng thực tế trên hệ thống (bám sát `BR-02`).
  - Sau khi KTV xác nhận nhận đủ, trạng thái yêu cầu chuyển thành `HANDED_OVER`.
- **Trường hợp lỗi**:
  - KTV cố tình bấm "Bắt đầu sửa chữa" trước khi xác nhận nhận vật tư từ lái xe: Hệ thống chặn hành động và cảnh báo "Vui lòng xác nhận nhận bàn giao vật tư từ lái xe trước".

#### FR-REP-01 — Ghi nhận tiến độ sửa chữa xe
- **Người dùng**:
  - Kỹ thuật viên.
- **Điều kiện**:
  - Trạng thái yêu cầu vật tư liên quan đã được xác nhận bàn giao (`HANDED_OVER`).
- **Thao tác**:
  - Bấm nút "Bắt đầu sửa chữa".
  - Thực hiện sửa chữa và cập nhật tiến độ.
  - Bấm nút "Xác nhận sửa xong".
- **Kết quả**:
  - Bấm bắt đầu -> Ticket chuyển sang `REPAIRING`, xe chuyển sang `IN_REPAIR`. Ghi nhận timestamp `started_at`.
  - Bấm sửa xong -> Ticket chuyển sang `COMPLETED`. Ghi nhận timestamp `ended_at`. Gửi thông báo nghiệm thu cho Cơ giới.
- **Trường hợp lỗi**:
  - Mất kết nối internet khi cập nhật: Lưu dữ liệu local và tự động đồng bộ khi có kết nối trở lại.

#### FR-REP-04 — Nghiệm thu sửa chữa và đóng phiếu
- **Người dùng**:
  - Đội cơ giới.
- **Điều kiện**:
  - Ticket sửa chữa đang ở trạng thái `COMPLETED`.
- **Thao tác**:
  - Thực hiện chạy thử và kiểm tra chất lượng xe thực tế.
  - Bấm chọn "Nghiệm thu đạt" hoặc "Nghiệm thu không đạt" trên ứng dụng.
- **Kết quả**:
  - *Nếu Đạt*: Ticket chuyển sang `CLOSED` (đóng phiếu), xe trở lại trạng thái `ACTIVE`.
  - *Nếu Không đạt*: Ticket quay lại `REPAIRING`, gửi thông báo yêu cầu KTV sửa lại.
- **Trường hợp lỗi**:
  - Bấm "Không đạt" nhưng bỏ trống ô nhập lý do: Hệ thống yêu cầu bắt buộc nhập lý do chưa đạt.

#### FR-DASH-01 — Dashboard thống kê và giám sát
- **Người dùng**:
  - Quản lý.
- **Điều kiện**:
  - Đăng nhập tài khoản có quyền `MANAGER`.
- **Thao tác**:
  - Truy cập màn hình Dashboard trên ứng dụng.
- **Kết quả**:
  - Biểu đồ thời gian thực hiển thị: Tần suất hỏng hóc, tỉ lệ downtime xe, hàng chờ xưởng, hiệu suất làm việc của KTV và danh mục tiêu hao linh kiện.
- **Trường hợp lỗi**:
  - Kết nối máy chủ bị nghẽn (quá 5 giây): Hiển thị trạng thái Loading và sử dụng dữ liệu Cache lưu gần nhất để hiển thị tạm thời.

---

## Outputs
- Tệp đặc tả yêu cầu chức năng hoàn chỉnh: [07_functional_requirements.md](file:///c:/Users/Legion/Desktop/IT/SNP/.agents/skills/system_engineering_copilot/deliverables/current/sss/part_b_requirement_analysis/07_functional_requirements.md)
