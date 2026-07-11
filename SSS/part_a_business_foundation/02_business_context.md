# 02 Business Context

Hiện tại, quy trình sửa chữa phương tiện trong doanh nghiệp chủ yếu đang được vận hành theo phương thức thủ công kết hợp bán tự động (một số bộ phận đã có phần mềm quản lý riêng lẻ) và phụ thuộc nhiều vào trao đổi thủ công giữa con người với nhau.

Các phương thức giao tiếp hiện tại bao gồm:
- Giao tiếp trực tiếp giữa các bộ phận
- Gọi điện thoại
- Nhắn tin qua các ứng dụng chat
- Ghi chú trên giấy hoặc tài liệu nội bộ

Quy trình này được áp dụng cho cả:
- Phương tiện nội bộ của doanh nghiệp
- Phương tiện bên ngoài (xe khách, xe đối tác, xe nhà cung cấp)

Mặc dù quy trình hiện tại vẫn có thể đáp ứng nhu cầu vận hành cơ bản, việc thiếu một hệ thống quản lý tập trung đang tạo ra nhiều hạn chế trong quá trình theo dõi, điều phối và kiểm soát hoạt động sửa chữa.

## 2.1 Mô hình vận hành hiện tại (Current Operational Model)
Quy trình nghiệp vụ hiện tại diễn ra theo luồng tổng quát như sau:
1. Phương tiện phát sinh hư hỏng hoặc sự cố.
2. Tài xế đang chịu trách nhiệm vận hành phương tiện phát hiện lỗi và báo lỗi.
3. Đội cơ giới tiếp nhận và kiểm tra sơ bộ.
4. Nếu cần sửa chữa, phương tiện được chuyển sang đội kỹ thuật.
5. Đội kỹ thuật đánh giá tình trạng và xác định nhu cầu vật tư.
6. Kho vật tư kiểm tra, xác nhận và cấp phát vật tư cho tài xế.
7. Tài xế nhận vật tư và bàn giao lại cho đội kỹ thuật.
8. Đội kỹ thuật tiến hành sửa chữa.
9. Phương tiện được bàn giao lại để tiếp tục vận hành.

Toàn bộ luồng xử lý trên hiện chưa được quản lý tập trung trên một nền tảng số thống nhất. Mặc dù bộ phận quản lý xưởng và quản lý vật tư đều đã có phần mềm quản lý riêng của từng bộ phận, nhưng các hệ thống này hoạt động hoàn toàn độc lập, không có tính đồng bộ dữ liệu với nhau, dẫn đến việc thông tin bị phân tán và cán bộ quản lý cấp cao không có được cái nhìn tổng thể để giám sát.

## 2.2 Các vấn đề hiện tại (Current Pain Points)

### 1. Khó theo dõi trạng thái sửa chữa
Sau khi phương tiện được báo lỗi, việc theo dõi trạng thái xử lý gặp nhiều khó khăn. Các câu hỏi thường xuyên phát sinh:
- Phương tiện hiện đang ở đâu?
- Đang chờ kiểm tra hay đang sửa chữa?
- Đang được xử lý bởi bộ phận nào?
- Dự kiến khi nào hoàn thành?

Việc thiếu khả năng theo dõi theo thời gian thực (Real-time Tracking) làm giảm hiệu quả phối hợp giữa các bộ phận.

### 2. Thiếu minh bạch trong hàng chờ sửa chữa
Khi số lượng phương tiện cần sửa chữa tăng lên, xưởng thường xuất hiện hàng chờ sửa chữa (Repair Queue). Hiện tại chưa có cơ chế rõ ràng để quản lý:
- Thứ tự ưu tiên sửa chữa
- Phương tiện nào đến trước / đến sau
- Phương tiện nào đang chờ lâu
- Trường hợp nào cần xử lý khẩn cấp

Điều này có thể dẫn đến chậm tiến độ, ùn tắc tại xưởng và khó phân bổ nguồn lực kỹ thuật.

### 3. Khó quản lý vật tư sửa chữa
Mặc dù bộ phận kho vật tư đã có phần mềm quản lý riêng, việc thiếu kết nối và đồng bộ với quy trình sửa chữa thực tế vẫn gây ra nhiều bất cập:
- Khó đối chiếu tức thời nhu cầu vật tư thực tế của KTV với tồn kho khả dụng trên hệ thống.
- Khó xác định chính xác vật tư đã xuất kho được sử dụng cụ thể cho phiếu sửa chữa (ticket) nào.
- Khó xác định vật tư hiện đang được giữ bởi ai trong luồng bàn giao (Thủ kho -> Tài xế -> Kỹ thuật viên xưởng).
- Khó kiểm soát rủi ro thất thoát hoặc nhầm lẫn phụ tùng trong quá trình trung chuyển.
- Khó tự động hóa việc thống kê lịch sử hao phí vật tư gắn với từng đầu xe.

### 4. Khó thống kê lịch sử sửa chữa
Dữ liệu lịch sử sửa chữa chưa được tập trung thành một nguồn dữ liệu thống nhất. Do đó doanh nghiệp khó thống kê:
- Một phương tiện đã sửa bao nhiêu lần
- Những lỗi nào xảy ra thường xuyên nhất
- Loại phương tiện nào hay gặp sự cố
- Thời gian sửa chữa trung bình
- Loại vật tư nào được sử dụng nhiều nhất

Việc thiếu dữ liệu lịch sử khiến doanh nghiệp khó đưa ra quyết định tối ưu vận hành.

### 5. Khó kiểm tra trách nhiệm giữa các bộ phận
Khi xảy ra chậm trễ hoặc sai sót, việc xác định nguyên nhân gặp nhiều khó khăn. Ví dụ:
- Tài xế đã báo lỗi chưa?
- Đội cơ giới đã tiếp nhận chưa?
- Đội kỹ thuật đang chờ gì?
- Chậm do thiếu vật tư hay do xưởng quá tải?
- Vật tư đang ở kho, ở tài xế hay đã giao cho xưởng?

Do chưa có dấu vết xử lý (Audit Trail) rõ ràng, doanh nghiệp khó đánh giá hiệu suất và trách nhiệm của từng bộ phận.

## 2.3 Nhu cầu chuyển đổi số (Digital Transformation Need)
Để giải quyết các vấn đề trên, doanh nghiệp cần xây dựng một hệ thống quản lý sửa chữa phương tiện tập trung trên nền tảng số.
Hệ thống mới cần cho phép:
- Chuẩn hóa toàn bộ quy trình sửa chữa
- Số hóa luồng xử lý giữa các bộ phận
- Theo dõi trạng thái sửa chữa theo thời gian thực
- Quản lý hàng chờ sửa chữa minh bạch
- Kiểm soát vật tư chặt chẽ
- Lưu trữ lịch sử sửa chữa đầy đủ
- Hỗ trợ thống kê và báo cáo quản trị

Ngoài việc số hóa quy trình sửa chữa, hệ thống còn cần hỗ trợ quản lý trách nhiệm sử dụng phương tiện và luồng bàn giao vật tư giữa các bên.
Hệ thống phải cho phép xác định rõ:
- Tài xế nào đang chịu trách nhiệm cho phương tiện tại thời điểm phát sinh sự cố
- Vật tư đang được giữ bởi ai
- Thời điểm vật tư được cấp phát và bàn giao
- Bộ phận nào đang chịu trách nhiệm xử lý ticket

## 2.4 Định hướng giải pháp (Solution Direction)
Giải pháp được đề xuất là xây dựng ứng dụng di động (Mobile Application) kết hợp với hệ thống quản lý dữ liệu tập trung.
Hệ thống sẽ hoạt động như một nền tảng trung tâm kết nối giữa:
- Tài xế (Driver)
- Đội cơ giới (Mechanic Team)
- Đội kỹ thuật (Technical Team)
- Kho vật tư (Inventory Team)
- Quản lý / Giám sát (Manager / Supervisor)

Thông qua hệ thống này, toàn bộ dữ liệu sửa chữa sẽ được quản lý tập trung, giúp tăng tính minh bạch, khả năng kiểm soát, hiệu suất vận hành và chất lượng ra quyết định quản lý.

## 2.5 Các quy tắc nghiệp vụ quan trọng (Key Business Rules)
- **BR-01: Quyền tạo phiếu sửa chữa**: Chỉ tài xế đang chịu trách nhiệm vận hành phương tiện tại thời điểm hiện tại mới được phép tạo phiếu sửa chữa (Repair Ticket) cho phương tiện đó. Điều này nhằm đảm bảo xác định đúng người chịu trách nhiệm ban đầu, tránh tạo phiếu sai phương tiện và đảm bảo tính minh bạch trong vận hành. Tại một thời điểm, một phương tiện chỉ nên có một tài xế chịu trách nhiệm chính. Tuy nhiên theo thời gian, một phương tiện có thể được bàn giao cho nhiều tài xế khác nhau, và một tài xế cũng có thể vận hành nhiều phương tiện khác nhau.
- **BR-02: Luồng cấp phát vật tư**: Vật tư sửa chữa không được cấp trực tiếp cho đội kỹ thuật. Quy trình cấp phát vật tư gồm các bước: Đội kỹ thuật xác định nhu cầu vật tư -> Kho vật tư xác nhận và cấp phát -> Tài xế nhận vật tư từ kho -> Tài xế bàn giao vật tư cho đội kỹ thuật -> Đội kỹ thuật sử dụng vật tư để sửa chữa. Quy trình này nhằm đảm bảo vật tư được cấp phát có kiểm soát, dễ truy vết lịch sử sử dụng và xác định rõ trách nhiệm giữa các bên.
- **BR-03: Một phiếu có thể chứa nhiều lỗi**: Một phiếu sửa chữa (Repair Ticket) có thể chứa nhiều lỗi của cùng một phương tiện (Ví dụ: hỏng phanh, rò dầu, hỏng đèn). Điều này giúp giảm số lượng phiếu phát sinh và phản ánh chính xác tình trạng thực tế của phương tiện tại thời điểm báo lỗi.
