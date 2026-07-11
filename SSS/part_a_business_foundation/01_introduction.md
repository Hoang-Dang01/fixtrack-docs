# 01 Introduction

## 1.1 Mục đích tài liệu (Purpose)
Tài liệu này được xây dựng nhằm phân tích, thiết kế và mô tả tổng thể hệ thống Quản Lý Sửa Chữa Phương Tiện (Vehicle Repair Management System).
Mục đích của hệ thống là số hóa toàn bộ quy trình quản lý sửa chữa phương tiện, bắt đầu từ giai đoạn phát hiện hư hỏng, báo lỗi, kiểm tra, sửa chữa, cấp phát vật tư cho đến khi phương tiện được đưa trở lại hoạt động bình thường.

Hệ thống phục vụ cho:
- Phương tiện nội bộ của doanh nghiệp
- Phương tiện bên ngoài (xe khách, xe đối tác, xe nhà cung cấp) khi phát sinh nhu cầu sửa chữa

Mặc dù nguồn gốc phương tiện có thể khác nhau, luồng xử lý nghiệp vụ sửa chữa về cơ bản vẫn giống nhau.
Tài liệu này đóng vai trò là nền tảng cho các giai đoạn sau của dự án, bao gồm:
- Phân tích nghiệp vụ (Business Analysis)
- Thiết kế hệ thống (System Design)
- Thiết kế cơ sở dữ liệu (Database Design)
- Thiết kế giao diện người dùng (UI/UX Design)
- Thiết kế dịch vụ API phía máy chủ (Backend API Design)
- Lập kế hoạch triển khai và vận hành (Deployment & Operations Planning)

Toàn bộ quá trình phát triển hệ thống sẽ dựa trên tài liệu này để đảm bảo tính nhất quán giữa yêu cầu nghiệp vụ và giải pháp kỹ thuật.

## 1.2 Bài toán hiện tại (Problem Statement)
Hiện tại, quy trình quản lý sửa chữa phương tiện phần lớn đang được thực hiện thủ công thông qua:
- Trao đổi trực tiếp giữa các bộ phận
- Gọi điện thoại
- Nhắn tin qua các ứng dụng chat
- Ghi chú trên giấy hoặc file rời rạc

Quy trình này áp dụng cho cả:
- Phương tiện nội bộ
- Phương tiện bên ngoài

Cách vận hành thủ công gây ra nhiều vấn đề trong quản lý:

### 1. Khó theo dõi trạng thái sửa chữa
Khi một phương tiện gặp sự cố, rất khó xác định chính xác:
- Phương tiện đang ở đâu?
- Đang chờ xử lý hay đang được sửa chữa?
- Khi nào hoàn thành?

Việc thiếu khả năng theo dõi theo thời gian thực (real-time) làm giảm hiệu quả phối hợp giữa các bộ phận.

### 2. Thiếu minh bạch trong quy trình xử lý
Do thông tin bị phân tán ở nhiều nơi, việc xác định trách nhiệm gặp khó khăn:
- Ai đã tiếp nhận phiếu?
- Ai đang xử lý?
- Bộ phận nào đang giữ phiếu sửa chữa?
- Tại sao phiếu bị chậm xử lý?

Điều này gây khó khăn trong việc kiểm soát tiến độ và đánh giá hiệu suất.

### 3. Quản lý hàng chờ sửa chữa chưa hiệu quả
Khi có nhiều phương tiện cùng cần sửa chữa:
- Khó xác định thứ tự ưu tiên
- Dễ xảy ra ùn tắc tại xưởng
- Khó phân bổ nguồn lực kỹ thuật viên

Việc thiếu cơ chế quản lý hàng chờ (Queue) rõ ràng có thể làm tăng thời gian chờ sửa chữa.

### 4. Quản lý vật tư chưa tối ưu
Việc theo dõi vật tư phục vụ sửa chữa còn thủ công:
- Khó kiểm tra tồn kho
- Khó xác định vật tư đã sử dụng cho phiếu nào
- Dễ xảy ra thất thoát hoặc sai lệch số liệu

Điều này ảnh hưởng trực tiếp đến chi phí vận hành.

### 5. Khó thống kê và báo cáo
Do dữ liệu chưa được tập trung, doanh nghiệp gặp khó khăn trong việc thống kê:
- Số lần sửa chữa
- Loại lỗi phổ biến
- Thời gian sửa chữa trung bình
- Thời gian ngừng hoạt động của phương tiện (Downtime)

Điều này làm giảm khả năng phân tích và tối ưu vận hành.

## 1.3 Mục tiêu hệ thống (Objectives)
Hệ thống mới được xây dựng nhằm giải quyết các vấn đề trên với các mục tiêu sau:

### 1. Số hóa quy trình sửa chữa
Chuyển đổi toàn bộ quy trình thủ công thành quy trình số hóa trên ứng dụng di động (Mobile Application).

### 2. Tăng khả năng theo dõi theo thời gian thực
Cho phép các bộ phận liên quan theo dõi trạng thái sửa chữa của từng phương tiện theo thời gian thực.

### 3. Chuẩn hóa luồng xử lý giữa các bộ phận
Thiết lập luồng xử lý (Workflow) thống nhất giữa:
- Người báo lỗi / tài xế
- Đội cơ giới
- Đội kỹ thuật
- Kho vật tư
- Quản lý

### 4. Tối ưu quản lý vật tư
Kiểm soát hiệu quả:
- Yêu cầu vật tư
- Cấp phát vật tư
- Tồn kho
- Lịch sử sử dụng vật tư

### 5. Hỗ trợ báo cáo và phân tích dữ liệu
Xây dựng nền tảng dữ liệu phục vụ:
- Bảng điều khiển (Dashboard)
- Chỉ số hiệu suất (KPI - Key Performance Indicator)
- Phân tích lỗi lặp lại
- Tối ưu hiệu suất sửa chữa

## 1.4 Các bên liên quan (Stakeholders)
Các bên liên quan chính trong hệ thống bao gồm:

### Người báo lỗi / Tài xế (Driver / Reporter)
Là người trực tiếp phát hiện sự cố của phương tiện.
Có thể là:
- Tài xế nội bộ
- Nhân viên vận hành
- Đại diện phương tiện bên ngoài

Vai trò:
- Báo lỗi
- Theo dõi trạng thái xử lý

### Đội cơ giới (Mechanic Team)
Bộ phận tiếp nhận và kiểm tra sơ bộ tình trạng phương tiện.
Vai trò:
- Xác minh lỗi
- Phân loại mức độ hư hỏng
- Chuyển bước xử lý tiếp theo

### Đội kỹ thuật (Technical Team)
Bộ phận chịu trách nhiệm sửa chữa.
Vai trò:
- Chẩn đoán lỗi chi tiết
- Sửa chữa
- Xác nhận hoàn thành

### Kho vật tư (Inventory Team)
Bộ phận quản lý vật tư phục vụ sửa chữa.
Vai trò:
- Theo dõi tồn kho
- Cấp phát vật tư
- Quản lý lịch sử xuất kho

### Quản lý / Giám sát (Manager / Supervisor)
Người giám sát hoạt động tổng thể.
Vai trò:
- Theo dõi chỉ số hiệu suất (KPI)
- Giám sát hàng chờ sửa chữa
- Điều phối vận hành

## 1.5 Giả định và ràng buộc (Assumptions & Constraints)

### Giả định (Assumptions)
- Hệ thống ưu tiên thiết bị di động (Mobile-first)
- Quy trình sửa chữa cho xe nội bộ và xe bên ngoài là tương tự
- Một phiếu sửa chữa có thể chứa nhiều lỗi
- Dữ liệu được đồng bộ tập trung qua dịch vụ API phía máy chủ (Backend API)

### Ràng buộc (Constraints)
- Luồng phê duyệt (Approval Workflow) chưa được xác định rõ
- Quy tắc ưu tiên hàng chờ (Queue Priority Rule) chưa được chốt
- Trang quản trị web (Web Admin Dashboard) chưa bắt buộc trong phiên bản đầu (MVP - Minimum Viable Product)
- Theo dõi chi phí sửa chữa (Cost Tracking) chưa được xác định đầy đủ
