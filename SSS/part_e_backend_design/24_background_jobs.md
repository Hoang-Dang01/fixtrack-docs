# 24 Background Jobs

**Trạng thái: Hoàn thành (Finalized - Version 1.0)**

---

## Purpose

Chương này đặc tả chi tiết thiết kế hệ thống tác vụ nền (Background Jobs) của FixTrack. Hệ thống sử dụng mô hình hàng chờ thông điệp (Message Queue) để xử lý các tác vụ tốn thời gian, chạy định kỳ hoặc không đồng bộ nhằm tối ưu hóa thời gian phản hồi (Response Time) của APIs, đảm bảo tính ổn định và khả năng tự phục hồi của hệ thống trước các lỗi tạm thời (Transient Failures).

---

## Inputs

*   [09 Business Rules](../part_b_requirement_analysis/09_business_rules.md)
*   [18 State Machine](../part_d_application_design/18_state_machine.md)
*   [19 Notification Flow](../part_d_application_design/19_notification_flow.md)
*   [20 Attachment Design](../part_d_application_design/20_attachment_design.md)
*   [21 API Design](./21_api_design.md)
*   [22 Backend Modules](./22_backend_modules.md)
*   [23 Service Contracts](./23_service_contracts.md)

---

# Level 1 — Background Job Architecture

Hệ thống FixTrack áp dụng giải pháp hàng chờ **BullMQ** chạy trên nền tảng **Redis** và được tích hợp vào **NestJS** thông qua thư viện `@nestjs/bullmq`.

```
┌─────────────────┐       Produce Job      ┌────────────────┐
│   HTTP Request  ├───────────────────────►│  Redis Queue   │
└────────┬────────┘                        └───────┬────────┘
         │ (Sync Transactions)                     │ (Async Fetch)
         ▼                                         ▼
┌─────────────────┐                        ┌────────────────┐
│ PostgreSQL (DB) │                        │ BullMQ Workers │
└─────────────────┘                        └────────────────┘
```

## 1.1 Nguyên tắc thiết kế (Core Design Principles)
1.  **Tách biệt tiến trình (Decoupling execution)**: Việc gửi thông báo đẩy, quét virus tệp tin, dọn dẹp bộ nhớ đệm hoặc ghi nhật ký kiểm toán không quan trọng sẽ được chuyển giao hoàn toàn cho các tiến trình Worker chạy nền, giải phóng tức thì luồng xử lý API chính (HTTP thread).
2.  **Đảm bảo giao dịch nhất quán (Sync Transaction for Critical Audits)**: Tránh việc đẩy tất cả nhật ký kiểm toán vào hàng chờ không đồng bộ. Thay vào đó, chia thành 2 chế độ:
    *   **Critical Audit (Đồng bộ)**: Các hành động như thay đổi trạng thái phiếu (`status_changed`), cấp phát vật tư, duyệt xuất kho, và hành động ghi đè của quản lý (`manager_override`) **bắt buộc ghi đồng bộ** cùng một Database Transaction với tác vụ chính. Nếu ghi log lỗi, transaction sẽ bị rollback.
    *   **Non-critical Audit (Bất đồng bộ)**: Các hành động như đăng nhập (`login`), xem màn hình (`read_access`), truy vấn dashboard sẽ được chuyển sang hàng chờ `audit-queue` xử lý không đồng bộ.
3.  **Tự động thử lại & Hàng chờ lỗi (Retry & DLQ)**: Mọi tác vụ lỗi tạm thời (ví dụ: FCM Gateway mất kết nối tạm thời) phải tự động thử lại với độ trễ lũy thừa (Exponential Backoff). Các tác vụ lỗi vĩnh viễn sẽ bị đẩy sang hàng chờ lỗi (Dead Letter Queue - DLQ) để kiểm tra thủ công.

---

# Level 2 — Queue Configuration Standard

Mỗi hàng chờ (Queue) trong hệ thống FixTrack được cấu hình riêng biệt dựa trên tần suất, mức độ quan trọng và khả năng xử lý song song:

| Tên Hàng chờ (Queue Name) | Module phát hành (Producer) | Worker xử lý (Consumer) | Mức ưu tiên (Priority) | Xử lý song song (Concurrency) | Số lần thử lại (Max Retries) | Chiến lược hoãn (Backoff Policy) | Khóa khử trùng (Dedupe Key) | DLQ Policy |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`notification-queue`** | `NotificationModule` | `NotificationWorker` | `HIGH` | 20 | 5 | Exponential (10s) | `event_id` + `user_id` + `channel` | Có (DLQ sau 5 lần lỗi) |
| **`attachment-scan-queue`**| `AttachmentModule` | `VirusScanWorker` | `HIGH` | 5 | 3 | Linear (30s) | `attachment_id` | Có (Chuyển cách ly & Khóa tệp) |
| **`attachment-gc-queue`** | `AttachmentModule` | `OrphanGCWorker` | `LOW` (Cron) | 1 | 1 | None | `attachment_id` | Không (Log & Alert) |
| **`ticket-timeout-monitor`**| `JobQueueModule` | `TimeoutMonitorWorker`| `MEDIUM` (Cron) | 1 | 2 | Linear (60s) | `ticket_id` + `status` + `timestamp` | Không (Log & Alert) |
| **`audit-queue`** | `AuditModule` | `AuditWorker` | `LOW` | 10 | 3 | Exponential (5s) | `trace_id` + `action_name` + `entity_id` | Có (Lưu file log khẩn cấp) |

---

# Level 3 — Detailed Queue Specifications

## 3.1 `notification-queue` (Gửi Thông báo đẩy & Web)
*   **Mục đích**: Gửi các cảnh báo đẩy thời gian thực (Push Notifications) qua Firebase Cloud Messaging (FCM) và qua Socket.IO Gateway tới người dùng.
*   **Cấu hình kỹ thuật**:
    ```typescript
    const queueOptions = {
      defaultJobOptions: {
        attempts: 5,
        backoff: {
          type: 'exponential',
          delay: 10000, // 10 giây ban đầu
        },
        removeOnComplete: true, // Xóa job thành công để tiết kiệm RAM Redis
        removeOnFail: false,   // Giữ lại job lỗi để đẩy sang DLQ
      }
    };
    ```

## 3.2 `attachment-scan-queue` (Quét Virus tệp đính kèm)
*   **Mục đích**: Quét các tệp đính kèm mới tải lên để ngăn chặn mã độc (virus, malware) xâm nhập máy chủ lưu trữ. Tác vụ này chạy ngay lập tức khi client gửi API xác nhận đã tải file lên S3 (`confirm-upload`).
*   **Cấu hình kỹ thuật**:
    *   **Concurrency**: 5 (giới hạn để tránh nghẽn CPU của server quét virus).
    *   **Timeout**: 60 giây.
*   **Quarantine Workflow (Quy trình cách ly khi nhiễm độc)**:
    Nếu phát hiện tệp tin bị nhiễm mã độc (infected):
    1.  **Cách ly vật lý**: Gọi S3 API sao chép tệp tin vật lý từ S3 bucket chính sang một **S3 Quarantine Bucket** (xóa tệp tin gốc ở bucket chính ngay lập tức để cô lập hoàn toàn).
    2.  **Khóa hiển thị**: Cập nhật trạng thái bản ghi đính kèm thành `INFECTED` trong CSDL nhằm chặn hoàn toàn khả năng đọc/xuất URL truy cập của tệp tin từ phía Client.
    3.  **Ghi nhật ký bảo mật**: Tạo một bản ghi Security Audit Log đặc biệt với mức độ ưu tiên `CRITICAL` ghi nhận hành vi đăng tải tệp tin độc hại.
    4.  **Cảnh báo khẩn**: Gửi thông báo đẩy tức thời cho Quản lý và Quản trị viên hệ thống để thực thi biện pháp rà soát an ninh.

## 3.3 `attachment-gc-queue` (Dọn dẹp file mồ côi - Orphaned Files GC)
*   **Mục đích**: Tìm kiếm và dọn dẹp các tệp đính kèm đã sinh Presigned URL nhưng Client không hoàn tất upload hoặc không gửi API xác nhận hoàn thành sau 24 giờ.
*   **Cấu hình kỹ thuật**:
    *   Chạy định kỳ (Cron Job) vào **02:00 sáng hàng ngày** (`0 2 * * *`).
    *   Thực hiện truy vấn lấy các bản ghi có trạng thái `PENDING` và quá 24 giờ, sau đó gọi S3 Client để xóa file vật lý và xóa bản ghi DB.

## 3.4 `ticket-timeout-monitor` (Giám sát cảnh báo trễ hạn - SLA Monitor)
*   **Mục đích**: Thực hiện quét định kỳ CSDL để phát hiện các phiếu sửa chữa (Tickets) bị tắc nghẽn ở các trạng thái quá lâu theo SLA (4h cho `inspecting`, 24h cho `waiting_queue`, 7d cho `waiting_parts`, 48h cho `repairing`) để gửi thông báo cảnh báo leo thang.
*   **Cấu hình kỹ thuật**:
    *   Chạy định kỳ **mỗi 30 phút một lần** (`*/30 * * * *`).
    *   Sử dụng cơ chế gom lô (Batched Queries) để tối ưu hiệu năng truy vấn DB.

## 3.5 `audit-queue` (Ghi nhận log kiểm toán bất đồng bộ - Low-value Audits)
*   **Mục đích**: Lưu trữ các log audit có mức độ ưu tiên thấp (như lịch sử xem màn hình, đọc dữ liệu, thống kê).
*   **Cấu hình kỹ thuật**:
    *   Sử dụng cơ chế gom lô ghi dữ liệu (Bulk Insert) để lưu tối thiểu 50 log cùng lúc hoặc lưu mỗi 5 giây một lần để giảm số lượng câu lệnh INSERT tới PostgreSQL.

---

# Level 4 — Idempotency & Deduplication (Quy tắc khử trùng lặp)

Để ngăn ngừa việc một tác vụ nền bị chạy lặp lại nhiều lần do lỗi mạng, worker khởi động lại đột ngột hoặc cơ chế gửi lại tin nhắn (At-least-once delivery):

```
                       Check Redis Dedupe Key
Job Received ─────────► [Key Exists?] ─────────► (Yes) ──► Skip Job (Success)
                             │
                             ▼ (No)
                       Save Dedupe Key to Redis
                             │
                             ▼
                       Execute Job Logic ──────► Delete Key on Fail
```

## 4.1 Cơ chế Khử trùng lặp (Job Deduplication Logic)
Trước khi đưa bất kỳ tác vụ nào vào hàng chờ, producer phải sinh một thuộc tính **`jobId`** đại diện cho khóa khử trùng (Dedupe Key) theo cấu trúc dưới đây:

1.  **Notification Jobs**:
    *   `jobId = notif_idempotency:<event_id>:<user_id>:<channel>`
    *   *Mục đích*: Đảm bảo mỗi sự kiện nghiệp vụ chỉ gửi đúng 1 thông báo duy nhất tới 1 tài khoản người dùng ứng với từng kênh truyền tải (ví dụ: một sự kiện có thể gửi song song qua cả Web Push và Socket IO nhưng không gửi trùng 2 tin cùng kênh).
2.  **Virus Scan Jobs**:
    *   `jobId = scan_idempotency:<attachment_id>`
    *   *Mục đích*: Chặn việc quét lặp lại một file nhiều lần.
3.  **State Timeout Alert Jobs**:
    *   `jobId = timeout_idempotency:<ticket_id>:<status>:<sla_milestone>`
    *   *Mục đích*: Ngăn việc bắn trùng lặp các thông báo cảnh báo quá hạn của cùng một mốc thời gian (ví dụ: cảnh báo trễ hạn 24h chỉ được gửi đúng một lần).
4.  **Audit Logs Jobs**:
    *   `jobId = audit_idempotency:<trace_id>:<action_name>:<entity_id>`
    *   *Mục đích*: Loại bỏ trùng lặp nhật ký kiểm toán không quan trọng khi một hành động gọi API bị Client retry nhiều lần trong cùng một phiên giao dịch.

## 4.2 Triển khai Idempotent Consumer (Mẫu thiết kế kiểm tra trước khi ghi)
Mỗi Worker khi nhận Job bắt buộc phải thực thi mẫu kiểm tra an toàn (Check-then-Act) trong database trước khi thực hiện logic nghiệp vụ:
```typescript
async process(job: Job<NotificationPayload>) {
  const { event_id, user_id } = job.data;
  
  // 1. Kiểm tra xem thông báo cho sự kiện này đã được ghi nhận trong DB chưa
  const exists = await this.notificationRepo.findOne({ where: { eventId: event_id, userId: user_id } });
  if (exists) {
    return; // Đã xử lý thành công trước đó, bỏ qua để tránh trùng lặp.
  }
  
  // 2. Thực hiện gửi thông báo và ghi nhận vào DB
  await this.sendPushAndSave(job.data);
}
```

---

# Level 5 — Dead Letter Queue (DLQ) & Failure Recovery

Hệ thống FixTrack cấu hình cơ chế tự động cô lập lỗi nghiệp vụ thông qua Dead Letter Queue (DLQ):

```
                      Worker Process Job
Job Executing ───────► [Has Error?] ───► (No) ──► Complete (Success)
                             │
                             ▼ (Yes)
                     [Max Retries Reached?]
                       (No) │   (Yes)
                            ▼     ▼
                     Exponential  Route to DLQ (Queue: failed-jobs)
                       Backoff    & Alert Manager (Slack/Email)
```

## 5.1 Retry Strategy với Exponential Backoff
Đối với các lỗi kết nối hoặc hạ tầng tạm thời (Transient Errors):
*   Tự động tính toán khoảng hoãn giữa các lần thử lại dựa trên lũy thừa:
    $$\text{delay} = \text{initial\_delay} \times 2^{(\text{attempt} - 1)}$$
*   Ví dụ cấu hình `notification-queue` (Attempts: 5, Delay: 10s):
    *   Lần 1: Lỗi $\rightarrow$ Đợi 10 giây.
    *   Lần 2: Lỗi $\rightarrow$ Đợi 20 giây.
    *   Lần 3: Lỗi $\rightarrow$ Đợi 40 giây.
    *   Lần 4: Lỗi $\rightarrow$ Đợi 80 giây.
    *   Lần 5: Vẫn lỗi $\rightarrow$ Chuyển sang **DLQ** (Trạng thái `failed` trong BullMQ).

## 5.2 Cơ chế Cảnh báo & Vận hành DLQ
*   **Tên Queue Lỗi**: Các job bị lỗi vượt quá số lần cấu hình sẽ được chuyển về trạng thái `failed` (BullMQ mặc định lưu trữ các bản ghi này riêng biệt trong Redis Hash).
*   **Hành động phụ (Side Effect)**: Một listener toàn cục lắng nghe sự kiện `JobFailedEvent` sẽ kích hoạt:
    1.  Ghi nhận lỗi chi tiết (Stack Trace) cùng mã `trace_id` vào Log hệ thống.
    2.  Nếu mức độ ưu tiên của Queue là `HIGH`, hệ thống gửi thông báo cảnh báo khẩn cấp tới kênh Slack/Microsoft Teams của đội vận hành.
*   **Cơ chế khắc phục (Manual Retry)**: Cung cấp giao diện quản trị Admin để quản trị viên có thể bấm "Retry" hàng loạt các job trong DLQ sau khi lỗi hạ tầng đã được khắc phục.

---

# Level 6 — Observability & Monitoring

## 6.1 Bull Board UI Setup
*   **Purpose**: Cung cấp giao diện đồ họa quản trị để theo dõi sức khỏe của hàng chờ theo thời gian thực.
*   **URL Endpoint**: `/admin/queues`
*   **Production Access Policy (Chính sách truy cập môi trường thực tế)**:
    *   *Môi trường Dev/Staging*: Có thể kích hoạt sử dụng bảo mật bằng Basic Auth để DEV/QA dễ dàng kiểm tra.
    *   *Môi trường Production*: **Nghiêm cấm** mở công khai ra Internet (dù có Basic Auth). Cổng truy cập `/admin/queues` phải bị block bởi API Gateway hoặc chỉ được phép truy cập từ mạng nội bộ doanh nghiệp thông qua **VPN** và được bảo vệ thêm bằng cơ chế phân quyền tài khoản quản trị cao cấp (**Admin RBAC**).
*   **Giao diện cho phép**:
    *   Theo dõi số lượng job trong các trạng thái: `active`, `waiting`, `completed`, `failed`, `delayed`.
    *   Đọc thông tin log lỗi chi tiết của từng Job bị hỏng.
    *   Dừng hoặc tiếp tục chạy các Queue khẩn cấp.

## 6.2 Distributed Tracing & Logging
*   **Trace ID Propagation**: Khi tạo một job nền, `trace_id` (Request ID) bắt buộc phải được truyền từ HTTP Thread ban đầu vào trường Metadata của Job Payload.
*   **Log correlation**: Toàn bộ các dòng log được ghi bởi Worker trong quá trình chạy tác vụ phải được gắn kèm tiền tố chứa `trace_id` và `job_id` đó:
    ```text
    [2026-06-30 07:30:12] [trace_12345] [job_notification_779] INFO: Gửi FCM thành công cho người dùng U-001.
    ```
