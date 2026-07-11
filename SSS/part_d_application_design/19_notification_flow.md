# 19 Notification Flow

## Purpose
Chương này đặc tả chi tiết cơ chế và luồng thông báo (Notification Flow) trong hệ thống FixTrack. Tài liệu định nghĩa các sự kiện kích hoạt, đối tượng nhận tin, nội dung mẫu thông báo và cấu trúc kỹ thuật của dữ liệu thông báo (Payload JSON) nhằm đảm bảo sự phối hợp thời gian thực (Real-time Collaboration) nhịp nhàng giữa các bộ phận: Lái xe (DRIVER), Đội cơ giới (MECHANIC), Đội kỹ thuật (TECHNICIAN) và Kho vật tư (INVENTORY).

Tài liệu này là cơ sở để:
- **Backend Design** (Chapter 21, 24) triển khai dịch vụ thông báo đẩy (Push Notification Service) và tích hợp các hàng chờ chạy nền (Background Notification Queue).
- **Frontend/Mobile Implementation** xử lý việc đăng ký Token thiết bị (Device Token) và hiển thị thông báo.

## Questions Answered
- Những sự kiện nghiệp vụ nào sẽ kích hoạt việc gửi thông báo?
- Bộ phận nào sẽ nhận được thông báo trong từng kịch bản cụ thể?
- Các cấp độ ưu tiên (Priority Levels) và quy trình leo thang cảnh báo (Escalation Policy) khi bị bỏ qua?
- Vòng đời và trạng thái của một thông báo (Read / Acknowledge / Expired) hoạt động như thế nào?
- Cấu trúc dữ liệu kỹ thuật gửi kèm thông báo là gì để client tự động điều hướng (Deep Linking)?
- Kênh truyền thông báo và chính sách lưu trữ (Retention Policy) được cấu hình ra sao?

## Inputs
- [05 User Roles](../part_a_business_foundation/05_user_roles.md)
- [06 Workflow Analysis](../part_b_requirement_analysis/06_workflow_analysis.md)
- [10 Use Cases](../part_b_requirement_analysis/10_use_cases.md)
- [17 Screen Specs](./17_screen_specs.md)
- [18 State Machine](./18_state_machine.md)

---

## Content

### Level 1 — Kênh truyền tải & Nguyên tắc thông báo

Hệ thống FixTrack sử dụng hai kênh thông báo chính để tối ưu hóa khả năng tiếp cận:
1. **Push Notification (Thông báo đẩy trên thiết bị di động/PWA)**:
   - Dành cho các vai trò thường xuyên di chuyển và tác nghiệp thực tế: **Lái xe (DRIVER)**, **Đội cơ giới (MECHANIC)**, **Kỹ thuật viên (TECHNICIAN)**.
   - Sử dụng Firebase Cloud Messaging (FCM) hoặc Web Push API để gửi thông báo trực tiếp đến thiết bị di động/tablet.
2. **Web Portal Notification (Thông báo dạng pop-up & chuông báo trên trình duyệt Web)**:
   - Dành cho các vai trò làm việc tại bàn trạm cố định: **Kho vật tư (INVENTORY)**, **Quản lý (MANAGER)**.
   - Hiển thị danh sách thông báo chưa đọc tại biểu tượng Chuông báo góc phải Header.

#### 1.1 Nguyên tắc xác định đối tượng nhận (Recipient Resolution Rules)
Để tránh spam thông báo tới toàn bộ nhân viên (ví dụ: gửi tin cho toàn bộ 50 lái xe hoặc 20 KTV), hệ thống áp dụng các quy tắc lọc đối tượng nhận đích danh (Targeted Recipients):
- **Assigned User (Nhận đích danh)**: Gửi trực tiếp cho Lái xe đang được gán vận hành xe đó (`active_assignment`), hoặc KTV được phân công phụ trách phiếu sửa chữa (`assigned_technician`).
- **Role Group on Shift (Nhận theo ca trực)**: Đối với các vai trò dùng chung như MECHANIC hoặc INVENTORY, hệ thống chỉ gửi push cho những tài khoản có trạng thái "Đang trong ca trực" (Active Shift) tại khu vực/kho xảy ra sự cố.
- **Manager Fallback (Nhận leo thang)**: Gửi tới cấp quản lý của bộ phận tương ứng nếu có sự cố bị chậm trễ xử lý.

#### 1.2 Trạng thái của một thông báo (Notification State Machine)
Mỗi bản ghi thông báo trong ứng dụng trải qua các trạng thái sau:
- **`unread`**: Thông báo mới được tạo, chưa được người dùng click mở.
- **`read`**: Người dùng đã bấm xem chi tiết thông báo (đánh dấu đã đọc).
- **`acknowledged`**: Dành cho các thông báo khẩn cấp (HIGH/CRITICAL) yêu cầu người nhận bấm nút xác nhận "Tôi đã nhận thông tin và đang xử lý" (như MECHANIC nhận yêu cầu nghiệm thu).
- **`expired`**: Thông báo đã quá hạn lưu trữ hoặc thông báo cũ của các phiên làm việc đã đóng.

---

### Level 2 — Ma trận sự kiện, Ưu tiên và Nội dung thông báo (Notification Matrix)

Hệ thống chia thông báo làm 4 cấp độ ưu tiên:
- **`LOW`**: Thông tin (Informational) — chỉ ghi chuông báo Web, không rung/push khẩn cấp.
- **`MEDIUM`**: Cần hành động sớm (Action soon) — push bình thường.
- **`HIGH`**: Cần xử lý ngay (Immediate action) — push kèm rung/âm thanh đặc biệt.
- **`CRITICAL`**: Nghiêm trọng (Escalate instantly) — push lặp lại, gọi điện tự động / SMS fallback nếu không phản hồi.

| Mã sự kiện | Sự kiện kích hoạt | Đối tượng nhận | Cấp ưu tiên | Kênh truyền | Tiêu đề (Title) | Nội dung mẫu (Body) | Deep Link / Route |
|:---|:---|:---|:---|:---|:---|:---|:---|
| **`NT-TICKET-01`** | Lái xe gửi báo hỏng | `MECHANIC` (on-shift) | `MEDIUM` | Push | Phiếu báo hỏng mới | Xe {ma_xe} vừa được báo hỏng tại khu vực {khu_vuc}. Vui lòng tiếp nhận. | `/repair-requests/{id}` |
| **`NT-TICKET-02`** | Cơ giới tiếp nhận kiểm tra | `DRIVER` (assigned) | `LOW` | Push | Đang kiểm tra xe | Phiếu báo hỏng {ma_phieu} của xe {ma_xe} đã được Cơ giới tiếp nhận kiểm tra. | `/repair-requests/{id}` |
| **`NT-TICKET-03`** | Cơ giới từ chối phiếu | `DRIVER` (assigned) | `HIGH` | Push | Phiếu báo hỏng bị từ chối | Phiếu {ma_phieu} bị từ chối. Lý do: {reason}. Xe trở lại hoạt động. | `/repair-requests/{id}` |
| **`NT-TICKET-04`** | Đẩy xe vào hàng chờ xưởng | `DRIVER` (assigned) | `LOW` | Push | Xe vào hàng chờ | Xe {ma_xe} đã được duyệt chuyển xưởng và đang ở vị trí #{vi_tri} trong hàng chờ. | `/repair-requests/{id}` |
| **`NT-MAT-01`** | KTV gửi yêu cầu vật tư | `INVENTORY` (on-shift)| `MEDIUM` | Web / Chuông | Yêu cầu cấp vật tư mới | KTV {ten_ktv} yêu cầu cấp phụ tùng cho phiếu {ma_phieu}. | `/materials/requests?id={mr_id}` |
| **`NT-MAT-02`** | Thủ kho xuất kho | `DRIVER` (assigned) | `HIGH` | Push | Vật tư đã xuất kho | Vật tư cho xe {ma_xe} đã xuất kho. Vui lòng đến kho nhận phụ tùng. | `/repair-requests/{id}` |
| **`NT-MAT-03`** | Thủ kho từ chối yêu cầu | `TECHNICIAN` (assigned)| `HIGH` | Push / Web | Yêu cầu vật tư bị từ chối | Yêu cầu phụ tùng cho phiếu {ma_phieu} bị từ chối. Lý do: {reason}. | `/repair-requests/{id}` |
| **`NT-MAT-04`** | Lái xe xác nhận nhận hàng | `TECHNICIAN` (assigned)| `MEDIUM` | Push | Lái xe đã nhận vật tư | Lái xe {ten_lai_xe} đã nhận vật tư từ kho. Vui lòng đợi nhận bàn giao tại xưởng. | `/repair-requests/{id}` |
| **`NT-REP-01`** | KTV bắt đầu sửa xe | `DRIVER` (assigned) | `LOW` | Push | Bắt đầu sửa chữa | KTV đã nhận đủ vật tư và tiến hành sửa chữa xe {ma_xe}. | `/repair-requests/{id}` |
| **`NT-REP-02`** | KTV báo sửa chữa xong | `MECHANIC` (on-shift) | `HIGH` | Push | Sửa chữa hoàn tất | Xe {ma_xe} đã được sửa xong. Vui lòng kiểm tra và nghiệm thu kỹ thuật. | `/repair-requests/{id}` |
| **`NT-REP-03`** | Nghiệm thu không đạt | `TECHNICIAN` (assigned)| `HIGH` | Push | Nghiệm thu không đạt | Xe {ma_xe} nghiệm thu chưa đạt. Lý do: {reason}. Yêu cầu kiểm tra lại. | `/repair-requests/{id}` |

---

### Level 3 — Quy trình leo thang cảnh báo (Escalation Policy)

Nếu các thông báo cấp độ `MEDIUM` hoặc `HIGH` bị bỏ qua quá lâu không có tương tác hoặc xác nhận (Acknowledge) từ đối tượng nhận trực tiếp, hệ thống tự động leo thang (escalate) cảnh báo lên cấp quản lý để tránh tắc nghẽn quy trình:

| Sự kiện bị tắc nghẽn | Thời gian chờ tối đa | Hành động leo thang (Escalation Action) |
|:---|:---|:---|
| **Yêu cầu cấp vật tư (`NT-MAT-01`)** | 15 phút | Gửi cảnh báo SMS / Zalo OAs cho Trưởng Kho vật tư duyệt thay. |
| **Xe chờ nghiệm thu (`NT-REP-02`)** | 30 phút | Gửi cảnh báo Push / Web cho Trưởng Đội Cơ giới thúc giục nhân viên nghiệm thu. |
| **Xe nằm trong hàng chờ (`NT-TICKET-04`)** | 1 giờ | Gửi thông báo đến Quản lý hệ thống (MANAGER) điều phối lại hàng chờ. |

---

### Level 4 — Cấu trúc dữ liệu thông báo kỹ thuật (Technical Payload Specs)

Mảng dữ liệu gửi đi bắt buộc phải chứa đầy đủ thông tin định danh và cơ chế khử trùng lặp (de-duplication) để client xử lý chính xác:

```json
{
  "to": "fcm_token_device_abcdef123456",
  "notification": {
    "title": "Sửa chữa hoàn tất",
    "body": "Xe XE-01 đã được sửa xong. Vui lòng kiểm tra và nghiệm thu kỹ thuật.",
    "sound": "default"
  },
  "data": {
    "notification_id": "notif_uuid_789101112",
    "event_code": "NT-REP-02",
    "priority": "HIGH",
    "created_at": "2026-06-27T15:59:20Z",
    "expires_at": "2026-06-27T16:59:20Z",
    "dedup_key": "NT-REP-02_ticket-45_status-completed",
    "click_action": "FLUTTER_NOTIFICATION_CLICK",
    "route": "/repair-requests/45",
    "metadata": {
      "request_id": 45,
      "vehicle_id": 12,
      "ma_phuong_tien": "XE-01"
    }
  }
}
```

#### Ràng buộc kỹ thuật các trường mới:
- **`dedup_key`**: Khóa khử trùng lặp. Client dựa vào đây để bỏ qua (ignore) các thông báo trùng lặp được gửi liên tiếp do lỗi retry của server.
- **`expires_at`**: Thời điểm hết hạn của thông báo. Sau thời gian này, client tự động ẩn/xóa thông báo khỏi khay hệ thống mà không cần người dùng thao tác.

---

### Level 5 — Cấu trúc dữ liệu lưu trữ (Database Entities)

Để hỗ trợ ghi nhận và truy vấn lịch sử thông báo, hệ thống bổ sung 3 thực thể dữ liệu sau:

```sql
-- 1. Bảng lưu trữ thông báo in-app gửi cho người dùng
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id INT REFERENCES users(id),
    title VARCHAR(255) NOT NULL,
    body TEXT NOT NULL,
    event_code VARCHAR(50) NOT NULL,
    priority VARCHAR(20) NOT NULL, -- LOW, MEDIUM, HIGH, CRITICAL
    status VARCHAR(20) NOT NULL DEFAULT 'unread', -- unread, read, acknowledged, expired
    route VARCHAR(255),
    metadata JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP WITH TIME ZONE
);

-- 2. Bảng lưu thiết bị đăng ký nhận push notification (FCM tokens)
CREATE TABLE user_devices (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id),
    device_token TEXT NOT NULL UNIQUE,
    platform VARCHAR(50) NOT NULL, -- android, ios, web
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 3. Bảng hàng chờ gửi thông báo ngầm
CREATE TABLE notification_jobs (
    id BIGSERIAL PRIMARY KEY,
    payload JSONB NOT NULL,
    retry_count INT DEFAULT 0,
    status VARCHAR(20) DEFAULT 'pending', -- pending, processing, completed, failed
    error_message TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

---

### Level 6 — Chính sách lưu trữ dữ liệu (Retention Policy)

Để tránh phình to cơ sở dữ liệu và đảm bảo tốc độ truy vấn cao, hệ thống áp dụng chính sách tự động dọn dẹp (auto-purge) thông qua Background Worker chạy định kỳ hàng tuần:

| Bảng dữ liệu | Chính sách lưu trữ (Retention) | Hành động khi quá hạn |
|:---|:---|:---|
| `notifications` | **90 ngày** từ ngày tạo | Tự động xóa (hard delete) |
| `notification_jobs` | **30 ngày** kể từ khi hoàn tất/thất bại | Tự động xóa hoặc di chuyển sang Archive |
| `user_devices` (inactive) | **180 ngày** không hoạt động | Xóa Token thiết bị cũ |

---

### Level 7 — Cơ chế Đảm bảo gửi & Xử lý lỗi (Delivery Guarantees & Fallback)

1. **Khử trùng lặp (Idempotency / Dedup)**: Server sinh ra `dedup_key` duy nhất dựa trên `{event_code}_{entity_id}_{status}`. Nếu worker gửi thử lại do timeout, client dựa vào key này để không hiển thị thông báo trùng lặp cho người dùng.
2. **Cơ chế Fallback thất bại (Failure Fallback)**:
   - Đối với các thông báo cấp độ **`HIGH`** hoặc **`CRITICAL`**, nếu sau 3 lần retry qua FCM vẫn thất bại (do thiết bị offline hoặc mất mạng), hệ thống tự động fallback qua **kênh SMS hoặc Zalo OAs** gửi trực tiếp số điện thoại tài khoản người nhận.
3. **Cài đặt thông báo người dùng (User Notification Preferences)**:
   - Hệ thống cung cấp bảng thiết lập cho phép từng người dùng tùy chỉnh bật/tắt (mute) thông báo theo từng loại sự cố (chỉ áp dụng cho mức độ `LOW` và `MEDIUM`). Cảnh báo mức `HIGH` và `CRITICAL` bắt buộc phải nhận, không được tắt.

---

## Outputs
- Đặc tả Luồng thông báo hoàn chỉnh: [19_notification_flow.md](./19_notification_flow.md)
- Tham chiếu sang: [17_screen_specs.md](./17_screen_specs.md) (luồng màn hình nhận điều hướng từ notification)
- Tham chiếu sang: [24_background_jobs.md](../part_e_backend_design/24_background_jobs.md) (đặc tả kỹ thuật của background worker)
