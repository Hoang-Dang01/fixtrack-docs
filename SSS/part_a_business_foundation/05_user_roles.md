# 05 User Roles

Hệ thống dự kiến hỗ trợ nhiều nhóm người dùng với vai trò và quyền hạn khác nhau nhằm đảm bảo quy trình sửa chữa được vận hành rõ ràng, minh bạch và có kiểm soát.

Mỗi vai trò (Role) sẽ có phạm vi truy cập dữ liệu và chức năng riêng theo cơ chế phân quyền theo vai trò (Role-Based Access Control - RBAC).

## 5.1 Tổng quan vai trò
Các vai trò chính trong hệ thống bao gồm:
- **Tài xế (Driver)**: Người vận hành phương tiện, báo hỏng và nhận bàn giao vật tư.
- **Đội cơ giới (Mechanic Team)**: Bộ phận quản lý đội xe và kiểm tra sơ bộ lỗi phương tiện.
- **Đội kỹ thuật (Technical Team)**: Kỹ thuật viên phụ trách chẩn đoán chuyên sâu và sửa chữa.
- **Kho vật tư (Inventory Staff)**: Thủ kho phụ trách quản lý tồn kho và xuất kho phụ tùng.
- **Quản lý / Giám sát (Manager / Supervisor)**: Giám sát hoạt động tổng thể và báo cáo hiệu suất.

## 5.2 Tài xế (Driver)
### Vai trò nghiệp vụ
Tài xế là người trực tiếp vận hành phương tiện và là người đầu tiên phát hiện sự cố. Trong hệ thống này, tài xế là một actor trung tâm vì không chỉ báo lỗi mà còn tham gia vào luồng bàn giao vật tư.
Tài xế chịu trách nhiệm:
- Theo dõi tình trạng phương tiện đang được phân công.
- Phát hiện và báo lỗi kịp thời.
- Nhận vật tư từ kho khi được phê duyệt.
- Bàn giao vật tư cho đội kỹ thuật tại xưởng.
- Theo dõi tiến độ sửa chữa và nhận lại xe khi hoàn thành.

### Quyền trên hệ thống (System Permissions)
Tài xế được phép:
- Đăng nhập hệ thống bằng tài khoản cá nhân.
- Tạo phiếu sửa chữa cho phương tiện đang được phân công lái.
- Nhập mô tả lỗi, chọn mức độ nghiêm trọng cảm nhận và đính kèm hình ảnh/video hiện trường.
- Theo dõi trạng thái của phiếu sửa chữa do mình tạo.
- Xác nhận đã nhận vật tư từ kho và xác nhận đã bàn giao vật tư cho đội kỹ thuật.
- Xác nhận nhận lại phương tiện sau sửa chữa.

### Hạn chế (Restrictions)
Tài xế không được phép:
- Chỉnh sửa trạng thái của phiếu sửa chữa (ví dụ: chuyển sang REPAIRING, COMPLETED).
- Cập nhật số lượng tồn kho vật tư.
- Tạo yêu cầu vật tư thay thế.
- Đóng phiếu sửa chữa (đóng ticket).

## 5.3 Đội cơ giới (Mechanic Team)
### Vai trò nghiệp vụ
Đội cơ giới chịu trách nhiệm điều phối vận hành phương tiện và xử lý sơ bộ các yêu cầu sửa chữa. Trong thực tế, đội cơ giới có thể bao gồm nhiều vị trí nghiệp vụ khác nhau:
- **Điều hành cơ giới (Fleet Dispatcher / Mechanic Dispatcher)**: Phân công phương tiện cho tài xế, theo dõi tài xế nào đang vận hành phương tiện nào, và quản lý lịch sử bàn giao phương tiện. Điều này rất quan trọng vì chỉ tài xế đang được phân công phương tiện tại thời điểm hiện tại mới được phép tạo phiếu sửa chữa cho phương tiện đó.
- **Nhân viên cơ giới (Mechanic Staff / Inspector)**: Tiếp nhận phiếu sửa chữa từ tài xế, thực hiện kiểm tra sơ bộ ban đầu, đánh giá mức độ hư hỏng ban đầu, phân loại lỗi và chuyển tiếp xử lý.

Trong phạm vi hệ thống hiện tại, các vị trí này được gom thành một vai trò duy nhất là **Đội cơ giới (Mechanic Team)**.

### Quyền trên hệ thống (System Permissions)
Đội cơ giới được phép:
- Xem danh sách phương tiện và trạng thái hoạt động của chúng.
- Phân công phương tiện cho tài xế và cập nhật trạng thái bàn giao xe.
- Xem lịch sử phân công phương tiện.
- Xem danh sách phiếu sửa chữa mới được báo.
- Tiếp nhận phiếu sửa chữa (ticket) để bắt đầu kiểm tra.
- Cập nhật kết quả kiểm tra sơ bộ.
- Phân loại mức độ lỗi và quyết định chuyển tiếp ticket sang đội kỹ thuật (lỗi nặng) hoặc từ chối/kết thúc ticket và trả xe hoạt động (lỗi nhẹ).

### Hạn chế (Restrictions)
Đội cơ giới không được phép:
- Phê duyệt và cấp phát vật tư từ kho.
- Cập nhật số lượng tồn kho phụ tùng.
- Tạo yêu cầu vật tư thay thế.

## 5.4 Đội kỹ thuật (Technical Team)
### Vai trò nghiệp vụ
Đội kỹ thuật là bộ phận chịu trách nhiệm sửa chữa chính tại xưởng. Đây là bộ phận xử lý phần lớn khối lượng công việc kỹ thuật của hệ thống.
Đội kỹ thuật chịu trách nhiệm:
- Chẩn đoán lỗi chi tiết (khám xe chuyên sâu).
- Xác định phương án sửa chữa.
- Tạo yêu cầu vật tư linh kiện thay thế.
- Thực hiện sửa chữa xe thực tế.
- Cập nhật tiến độ sửa chữa.
- Xác nhận hoàn tất sửa chữa.

### Quyền trên hệ thống (System Permissions)
Đội kỹ thuật được phép:
- Xem danh sách ticket được chuyển đến từ đội cơ giới.
- Cập nhật kết quả chẩn đoán chi tiết và phương án sửa chữa.
- Tạo yêu cầu vật tư thay thế đi kèm phiếu sửa chữa.
- Xác nhận đã nhận vật tư từ tài xế để bắt đầu sửa chữa.
- Cập nhật trạng thái bắt đầu sửa chữa, ghi nhận tiến độ sửa chữa thực tế.
- Đánh dấu hoàn thành sửa chữa phương tiện.
- Đóng phiếu sửa chữa sau khi nghiệm thu kỹ thuật thành công.

### Hạn chế (Restrictions)
Đội kỹ thuật không được phép:
- Tạo phiếu báo sự cố mới (chỉ tài xế mới được tạo).
- Cập nhật số lượng tồn kho vật tư trực tiếp hoặc phê duyệt cấp phát vật tư mà không qua thủ kho.
- Quản lý tài khoản người dùng khác.

## 5.5 Kho vật tư (Inventory Staff)
### Vai trò nghiệp vụ
Kho vật tư chịu trách nhiệm quản lý toàn bộ vật tư, phụ tùng phục vụ sửa chữa trong doanh nghiệp.
Kho vật tư chịu trách nhiệm:
- Quản lý danh mục vật tư và cập nhật số lượng tồn kho.
- Kiểm tra tính khả dụng của vật tư theo yêu cầu sửa chữa.
- Phê duyệt và thực hiện cấp phát vật tư cho tài xế.
- Theo dõi lịch sử xuất nhập kho.

### Quyền trên hệ thống (System Permissions)
Kho vật tư được phép:
- Xem danh sách vật tư và số lượng tồn kho hiện tại.
- Tạo mới hoặc cập nhật thông tin vật tư.
- Tiếp nhận và xem chi tiết yêu cầu vật tư đi kèm phiếu sửa chữa.
- Phê duyệt yêu cầu vật tư và cập nhật trừ kho sau khi xuất kho.
- Theo dõi lịch sử cấp phát vật tư.

### Hạn chế (Restrictions)
Kho vật tư không được phép:
- Tạo phiếu sửa chữa phương tiện.
- Cập nhật trạng thái sửa chữa của phương tiện (ngoài trạng thái cấp phát vật tư).
- Đóng phiếu sửa chữa.

## 5.6 Quản lý / Giám sát (Manager / Supervisor)
### Vai trò nghiệp vụ
Quản lý là người giám sát hoạt động tổng thể của hệ thống. Vai trò này tập trung vào giám sát (Monitoring), kiểm soát (Control) và báo cáo (Reporting) nhằm tối ưu hóa hiệu suất và giải quyết các điểm nghẽn (Bottlenecks) trong quy trình vận hành.

### Quyền trên hệ thống (System Permissions)
Quản lý được phép:
- Xem toàn bộ dữ liệu hoạt động trong hệ thống.
- Truy cập bảng điều khiển (Dashboard) và kết xuất các báo cáo tổng hợp.
- Theo dõi trạng thái hàng chờ sửa chữa (Queue) và can thiệp điều chỉnh thứ tự ưu tiên nếu cần.
- Theo dõi hiệu suất làm việc của các bộ phận (tài xế, cơ giới, kỹ thuật, kho).
- Xem lịch sử log hệ thống (Audit trail).
- Override (ghi đè) một số trạng thái đặc biệt trong trường hợp khẩn cấp hoặc có tranh chấp giữa các bộ phận.

### Hạn chế (Restrictions)
Thông thường, quản lý không trực tiếp thực hiện các thao tác tác nghiệp như tạo ticket, kiểm tra xe, sửa chữa hay xuất kho vật tư, trừ các trường hợp override đặc biệt.

## 5.7 Ma trận phân quyền tổng quan (Role Permission Matrix)

| Chức năng / Tính năng | Driver | Đội Cơ Giới | Đội Kỹ Thuật | Kho Vật Tư | Quản Lý |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Đăng nhập & Quản lý profile** | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Tạo phiếu báo hỏng (Ticket)** | ✓ | ✗ | ✗ | ✗ | Override |
| **Phân công phương tiện cho tài xế** | ✗ | ✓ | ✗ | ✗ | Override |
| **Kiểm tra sơ bộ & Phân loại lỗi** | ✗ | ✓ | ✗ | ✗ | Override |
| **Chẩn đoán chi tiết & Sửa chữa** | ✗ | ✗ | ✓ | ✗ | ✗ |
| **Tạo yêu cầu vật tư** | ✗ | ✗ | ✓ | ✗ | ✗ |
| **Duyệt và xuất kho vật tư** | ✗ | ✗ | ✗ | ✓ | Override |
| **Xác nhận nhận & bàn giao vật tư** | ✓ | ✗ | ✓ | ✗ | ✗ |
| **Cập nhật tiến độ & hoàn thành sửa**| ✗ | ✗ | ✓ | ✗ | Override |
| **Đóng phiếu sửa chữa** | ✗ | ✗ | ✓ | ✗ | Override |
| **Xem Dashboard & Báo cáo quản trị**| ✗ | ✗ | ✗ | ✗ | ✓ |
| **Xem lịch sử log hệ thống** | ✗ | ✗ | ✗ | ✗ | ✓ |

*Ghi chú: Quyền **Override** chỉ áp dụng cho tài khoản Quản lý cấp cao trong các tình huống khẩn cấp.*

## 5.8 Kết luận
Mô hình phân quyền trên được thiết kế nhằm tối đa hóa tính minh bạch và kiểm soát chặt chẽ quy trình sửa chữa. Bằng việc phân chia vai trò rõ ràng, hệ thống giảm thiểu tối đa rủi ro thao tác sai quyền, thất thoát vật tư và tăng tính chịu trách nhiệm của mỗi bộ phận trong quy trình số hóa này.
