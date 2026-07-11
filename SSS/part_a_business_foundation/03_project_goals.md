# 03 Project Goals

Hệ thống Quản Lý Sửa Chữa Phương Tiện (Vehicle Repair Management System) được xây dựng nhằm giải quyết các vấn đề hiện tại trong quy trình sửa chữa và hướng tới việc nâng cao hiệu quả vận hành tổng thể.

Các mục tiêu của dự án được chia thành ba nhóm chính: mục tiêu nghiệp vụ, mục tiêu vận hành và mục tiêu chiến lược.

## 3.1 Mục tiêu nghiệp vụ (Business Goals)

### 1. Số hóa quy trình báo hư hỏng
Chuyển đổi hoàn toàn quy trình báo lỗi thủ công sang nền tảng số thông qua ứng dụng di động (Mobile Application). Hệ thống cần cho phép tài xế:
- Tạo phiếu sửa chữa nhanh chóng
- Báo nhiều lỗi trong cùng một phiếu
- Cung cấp thông tin lỗi chính xác và đầy đủ

Việc số hóa giúp giảm phụ thuộc vào giấy tờ, gọi điện thoại hoặc tin nhắn rời rạc.

### 2. Chuẩn hóa luồng xử lý sửa chữa
Thiết lập một luồng xử lý chuẩn (Standardized Workflow) giữa các bộ phận liên quan: Tài xế, Đội cơ giới, Đội kỹ thuật, Kho vật tư và Quản lý. Mục tiêu là đảm bảo mọi phiếu sửa chữa đều được xử lý theo quy trình thống nhất, rõ ràng và có thể kiểm tra.

### 3. Tăng tính minh bạch và khả năng truy vết
Mọi thao tác trong hệ thống cần được ghi nhận đầy đủ (Ai tạo phiếu? Ai tiếp nhận? Ai đang xử lý? Ai nhận vật tư? Ai bàn giao vật tư?). Điều này giúp xây dựng dấu vết xử lý (Audit Trail) rõ ràng cho toàn bộ vòng đời của phiếu sửa chữa.

## 3.2 Mục tiêu vận hành (Operational Goals)

### 4. Theo dõi tiến độ theo thời gian thực
Hệ thống phải cho phép theo dõi trạng thái của từng phiếu sửa chữa (Repair Ticket) theo thời gian thực (ví dụ: Đã báo lỗi, Đang kiểm tra, Đang chờ sửa, Đang chờ vật tư, Đang sửa chữa, Đã hoàn thành). Điều này giúp các bộ phận phối hợp hiệu quả hơn trong quá trình xử lý.

### 5. Quản lý hàng chờ sửa chữa
Hệ thống cần hỗ trợ quản lý hàng chờ sửa chữa (Repair Queue) để theo dõi thứ tự sửa chữa, xác định mức độ ưu tiên, giảm ùn tắc tại xưởng và tối ưu phân bổ nguồn lực kỹ thuật nhằm rút ngắn thời gian chờ sửa chữa.

### 6. Quản lý cấp phát vật tư
Hệ thống cần số hóa toàn bộ quy trình cấp phát vật tư (yêu cầu vật tư, xác nhận cấp phát, tài xế nhận vật tư, bàn giao vật tư cho đội kỹ thuật, ghi nhận vật tư thực tế sử dụng) nhằm đảm bảo cấp đúng, dễ truy vết và giảm thất thoát.

### 7. Tối ưu hiệu suất sửa chữa
Hệ thống cần hỗ trợ rút ngắn thời gian chờ sửa, thời gian sửa chữa thực tế và thời gian ngừng hoạt động của phương tiện (Downtime). Mục tiêu cuối cùng là tăng khả năng sẵn sàng vận hành của đội xe.

## 3.3 Mục tiêu chiến lược (Strategic Goals)

### 8. Lưu trữ lịch sử sửa chữa tập trung
Xây dựng cơ sở dữ liệu tập trung lưu trữ toàn bộ lịch sử sửa chữa của phương tiện (số lần sửa, các loại lỗi xảy ra, vật tư đã sử dụng, thời gian xử lý và người tham gia xử lý). Điều này giúp doanh nghiệp có góc nhìn dài hạn về độ bền và tình trạng kỹ thuật của từng phương tiện.

### 9. Hỗ trợ thống kê và báo cáo quản trị
Hệ thống cần cung cấp dữ liệu cho bảng điều khiển (Dashboard) và báo cáo vận hành. Các chỉ số quan trọng bao gồm: số phiếu sửa chữa theo ngày/tuần/tháng, thời gian sửa chữa trung bình, loại lỗi phổ biến nhất, phương tiện gặp sự cố nhiều nhất, và mức độ sử dụng vật tư.

### 10. Tạo nền tảng mở rộng cho tương lai
Hệ thống cần được thiết kế đủ linh hoạt để hỗ trợ các tính năng mở rộng trong tương lai như: trang quản trị web (Web Admin Dashboard), theo dõi chi phí sửa chữa chi tiết (Repair Cost Tracking), phân tích dự đoán bảo trì (Predictive Maintenance Analytics), hoặc tích hợp hệ thống ERP hiện có của doanh nghiệp.

## 3.4 Tiêu chí thành công (Success Criteria)
Dự án được xem là thành công khi đạt được các tiêu chí sau:
- 100% phiếu sửa chữa được tạo và theo dõi trên hệ thống.
- 100% trạng thái và vị trí vật tư được cập nhật theo thời gian thực.
- Giảm thiểu đáng kể thời gian dừng xe (Downtime) do tối ưu hóa hàng chờ và quy trình cấp phát vật tư.
- Xác định rõ ràng trách nhiệm của từng cá nhân/bộ phận trong mỗi bước xử lý.
