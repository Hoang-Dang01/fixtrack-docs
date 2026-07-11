# 04 System Scope

Phạm vi hệ thống xác định các chức năng sẽ được xây dựng trong phiên bản hiện tại của hệ thống, đồng thời làm rõ những chức năng chưa thuộc phạm vi triển khai để tránh mở rộng ngoài kiểm soát.

Hệ thống được định hướng là một ứng dụng di động (Mobile Application) phục vụ việc quản lý toàn bộ quy trình sửa chữa phương tiện từ lúc phát sinh lỗi cho đến khi hoàn tất sửa chữa.

## 4.1 Trong phạm vi triển khai (In Scope)
Phiên bản hiện tại của hệ thống sẽ bao gồm các phân hệ chức năng chính sau:

### 4.1.1 Quản lý người dùng (User Management)
Phân hệ này chịu trách nhiệm quản lý thông tin người dùng và quyền truy cập hệ thống.
- Đăng nhập hệ thống nội bộ.
- Xác thực người dùng (Authentication).
- Phân quyền theo vai trò (Role-Based Access Control - RBAC).
- Quản lý thông tin tài khoản.
- Các vai trò dự kiến: Tài xế (Driver), Đội cơ giới (Mechanic Team), Đội kỹ thuật (Technical Team), Kho vật tư (Inventory Team), Quản lý / Giám sát (Manager / Supervisor).

### 4.1.2 Quản lý phương tiện (Vehicle Management)
Phân hệ này quản lý thông tin phương tiện phục vụ sửa chữa.
- Quản lý danh sách phương tiện (mã phương tiện, biển số, loại phương tiện, trạng thái hoạt động).
- Phân loại phương tiện nội bộ và phương tiện bên ngoài.
- Theo dõi trạng thái hiện tại của phương tiện.
- Xem lịch sử sửa chữa của phương tiện.

### 4.1.3 Quản lý phân công phương tiện (Vehicle Assignment Management)
Phân hệ này quản lý việc phân công phương tiện cho tài xế bởi điều hành cơ giới (Fleet Dispatcher / Mechanic Dispatcher). Đây là phân hệ quan trọng vì nó ảnh hưởng trực tiếp đến quyền tạo phiếu sửa chữa.
- Phân công phương tiện cho tài xế.
- Thu hồi / kết thúc phân công.
- Theo dõi tài xế hiện tại của từng phương tiện.
- Lưu lịch sử phân công phương tiện.
- **Business Rule quan trọng**: Chỉ tài xế đang được phân công vận hành phương tiện tại thời điểm hiện tại mới được phép tạo phiếu sửa chữa cho phương tiện đó.

### 4.1.4 Quản lý phiếu sửa chữa (Repair Ticket Management)
Đây là phân hệ trung tâm của toàn bộ hệ thống.
- Tài xế tạo phiếu báo lỗi.
- Hỗ trợ ghi nhận nhiều lỗi trong cùng một phiếu (ví dụ: hỏng phanh, rò dầu, hỏng đèn).
- Đính kèm mô tả chi tiết và hình ảnh lỗi ngoài hiện trường.
- Theo dõi trạng thái phiếu sửa chữa theo thời gian thực.
- Lưu lịch sử xử lý phiếu.

### 4.1.5 Quản lý kiểm tra sơ bộ (Inspection Workflow)
Phân hệ này hỗ trợ quy trình tiếp nhận và kiểm tra ban đầu bởi đội cơ giới.
- Tiếp nhận phiếu sửa chữa từ tài xế.
- Kiểm tra sơ bộ tình trạng phương tiện.
- Phân loại mức độ lỗi.
- Quyết định chuyển tiếp sang đội kỹ thuật (lỗi nặng) hoặc kết thúc xử lý/trả xe chạy tiếp (lỗi nhẹ).

### 4.1.6 Quản lý hàng chờ sửa chữa (Repair Queue Management)
Phân hệ này hỗ trợ quản lý hàng chờ sửa chữa (Repair Queue) tại xưởng.
- Đưa phương tiện vào hàng chờ khi xưởng quá tải.
- Theo dõi thứ tự sửa chữa.
- Hiển thị trạng thái chờ cho tài xế và các bộ phận khác.
- Hỗ trợ điều phối công việc trong xưởng.
- *Lưu ý*: Quy tắc ưu tiên hàng chờ hiện chưa được chốt hoàn toàn và sẽ được phân tích chi tiết ở giai đoạn sau.

### 4.1.7 Quản lý vật tư (Material Management)
Phân hệ này quản lý toàn bộ luồng vật tư phục vụ sửa chữa.
- Quản lý danh sách vật tư và theo dõi tồn kho.
- Tạo yêu cầu vật tư đi kèm phiếu sửa chữa.
- Xác nhận cấp phát vật tư từ kho.
- Theo dõi luồng bàn giao vật tư (Kho -> Tài xế nhận -> Tài xế bàn giao cho Đội kỹ thuật -> Đội kỹ thuật sử dụng).

### 4.1.8 Quản lý sửa chữa (Repair Execution)
Phân hệ này hỗ trợ đội kỹ thuật trong quá trình sửa chữa thực tế.
- Ghi nhận thời điểm bắt đầu sửa chữa.
- Cập nhật tiến độ sửa chữa.
- Ghi nhận công việc và vật tư thực tế đã sử dụng.
- Xác nhận hoàn tất sửa chữa và đóng phiếu sửa chữa.

### 4.1.9 Báo cáo và thống kê (Reporting)
Phân hệ này cung cấp khả năng giám sát và hỗ trợ ra quyết định.
- Hiển thị bảng điều khiển (Dashboard) vận hành.
- Thống kê lịch sử sửa chữa của toàn bộ đội xe.
- Theo dõi các chỉ số hiệu suất (KPI) như thời gian sửa chữa trung bình, thời gian dừng xe (Downtime), tần suất lỗi, và mức sử dụng vật tư.

## 4.2 Ngoài phạm vi triển khai hiện tại (Out of Scope - Initial Version)
Các chức năng sau chưa nằm trong phạm vi triển khai của phiên bản đầu tiên (MVP):
- **Thanh toán / Tính phí (Payment / Billing)**: Chưa hỗ trợ báo giá sửa chữa, xuất hóa đơn tài chính hoặc tích hợp cổng thanh toán.
- **Mua sắm vật tư / Nhà cung cấp (Supplier & Procurement)**: Chưa bao gồm quản lý nhà cung cấp, đặt mua vật tư và luồng nhập kho từ nhà cung cấp bên ngoài.
- **Cổng khách hàng bên ngoài (External Customer Portal)**: Chưa hỗ trợ giao diện riêng cho các chủ xe, đối tác ngoài. Mọi thao tác đều thông qua nhân viên nội bộ của doanh nghiệp.
- **Phân tích nâng cao và AI (Advanced Analytics / AI Prediction)**: Chưa hỗ trợ dự đoán lỗi phương tiện, bảo trì dự đoán (predictive maintenance) hoặc đề xuất tự động.

## 4.3 Phạm vi mở rộng tương lai (Future Scope)
Các tính năng có thể mở rộng trong các phase tiếp theo bao gồm:
- Trang quản trị web nâng cao (Web Admin Dashboard) cho Quản lý.
- Theo dõi chi phí sửa chữa chi tiết (Repair Cost Tracking) gồm chi phí nhân công và vật tư.
- Tích hợp với hệ thống ERP / Kho tổng của doanh nghiệp.
- Tích hợp thiết bị định vị/cảm biến IoT trên phương tiện để tự động phát hiện lỗi.
