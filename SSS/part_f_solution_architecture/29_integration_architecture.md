# 29 Integration Architecture

**Trạng thái: Hoàn thành (Finalized - Version 1.0)**

---

## Purpose

Chương này đặc tả chi tiết kiến trúc tích hợp (Integration Architecture) của hệ thống FixTrack với các hệ thống và dịch vụ bên thứ ba (outbound communication boundaries). Tài liệu định nghĩa các nguyên tắc tích hợp, ma trận xử lý lỗi, lớp trừu tượng (Adapter Layer), chi tiết kỹ thuật của từng dịch vụ tích hợp (S3, FCM, SMS, Email, Webhooks), cơ chế giám sát tập trung, và các mẫu thiết kế tăng cường khả năng chịu lỗi (Resilience Patterns) để đảm bảo hệ thống vận hành bền bỉ và an toàn.

---

## Inputs

*   [19 Notification Flow](../part_d_application_design/19_notification_flow.md)
*   [20 Attachment Design](../part_d_application_design/20_attachment_design.md)
*   [21 API Design](../part_e_backend_design/21_api_design.md)
*   [22 Backend Modules](../part_e_backend_design/22_backend_modules.md)
*   [24 Background Jobs](../part_e_backend_design/24_background_jobs.md)
*   [26 System Architecture](./26_system_architecture.md)
*   [28 Security Architecture](./28_security_architecture.md)

---

# Level 0 — Integration Principles (Nguyên tắc Tích hợp)

Để tránh hệ thống bị phụ thuộc chặt chẽ và gián đoạn vận hành do sự cố từ các dịch vụ bên ngoài, FixTrack tuân thủ 4 nguyên tắc cốt lõi sau:

1.  **Optional Dependencies (Phụ thuộc tùy chọn)**: Dịch vụ bên thứ ba chỉ đóng vai trò hỗ trợ tăng trải nghiệm. Sự cố từ bên thứ ba (ví dụ: Firebase bị nghẽn, SMS Gateway mất kết nối) tuyệt đối **không được làm hỏng (break) giao dịch nghiệp vụ chính** lưu trữ tại CSDL (ví dụ: tài xế vẫn phải tạo được ticket báo hỏng xe thành công ngay cả khi thông báo push hoặc email gửi đi bị lỗi).
2.  **Adapter Pattern & Anti-Corruption Layer (Lớp chống xói mòn nghiệp vụ)**: Mã nguồn nghiệp vụ (Core Domain/Application Use Cases) không được tương tác trực tiếp với SDK hoặc API thô của bên thứ ba. Tất cả phải giao tiếp qua các cổng trừu tượng (Interfaces). Các thay đổi về nhà cung cấp (ví dụ: chuyển từ AWS S3 sang Cloudflare R2, chuyển từ eSMS sang Twilio) chỉ yêu cầu viết thêm Adapter mới mà không cần chỉnh sửa logic nghiệp vụ cốt lõi.
3.  **Fail-safe by Default**: Mọi cuộc gọi mạng (Network calls) ra ngoài hệ thống phải được giới hạn thời gian (Timeout tối đa 3-5 giây) và được bao bọc bởi các cơ chế tự động ngắt mạch (Circuit Breaker) để ngăn ngừa hiện tượng thắt nút cổ chai tài nguyên hệ thống (như cạn kiệt thread pool hoặc database connection pool).
4.  **Observability & Traceability (Khả năng giám sát hành trình)**: Mọi yêu cầu tích hợp phải mang theo mã định danh duy nhất (`trace_id` hoặc W3C `traceparent`). Tất cả thông tin đầu vào/đầu ra và thời gian xử lý của API ngoại vi phải được ghi nhận vào hệ thống Log/Trace phục vụ công tác đối soát và gỡ lỗi (Debugging).

---

# Level 1 — Integration Registry & Failure Matrix (Bản đăng ký Tích hợp & Ma trận Xử lý Lỗi)

## 1.1 Sơ đồ Tích hợp Tổng quan (Integration Topology)

Sơ đồ dưới đây mô tả các điểm tích hợp ngoại vi của FixTrack và vai trò thực thi giữa Backend (NestJS) và Frontend (React Native & Next.js):

```mermaid
graph TD
    subgraph ClientLayer[Frontend Application]
        Mobile[Mobile App: React Native]
        Web[Web Admin: Next.js]
    end

    subgraph AppLayer[FixTrack Backend]
        NestAPI[NestJS Core API]
        BullWorker[BullMQ Workers]
    end

    subgraph StorageSvc[Object Storage]
        S3Main[S3 Main Bucket]
        S3Quar[S3 Quarantine Bucket]
    end

    subgraph AlertSvc[Messaging & Notifications]
        FCM[Firebase Cloud Messaging]
        SMS[SMS / Zalo OA Gateway]
        SMTP[SMTP / Amazon SES]
        Slack[Slack/Teams Incident Webhook]
    end

    subgraph MonitorSvc[Diagnostics]
        Sentry[Sentry Exception Tracker]
        Prom[Prometheus Metrics]
    end

    %% Client Integrations
    Mobile -->|1. PUT Upload trực tiếp| S3Main
    Mobile -->|2. Đăng ký nhận Push| FCM
    Mobile & Web -->|3. Báo cáo Crash| Sentry

    %% Backend Integrations
    NestAPI -->|Gen Presigned URL| S3Main
    NestAPI -->|Expose Metrics| Prom
    NestAPI -->|Log Server Crash| Sentry
    
    %% Background Job Integrations
    BullWorker -->|4. Quét mã độc & Cách ly| S3Quar
    BullWorker -->|5. Gửi Push Notification| FCM
    BullWorker -->|6. Fallback SMS khẩn cấp| SMS
    BullWorker -->|7. Gửi email thông báo| SMTP
    BullWorker -->|8. Cảnh báo an ninh| Slack
```

## 1.2 Phân kỳ Tích hợp theo Giai đoạn (Phasing Specification)

Để tối ưu hóa chi phí và tốc độ phát triển dự án, các đầu mục tích hợp được phân chia cụ thể như sau:

*   **Phase 1 (MVP - Hiện tại)**: Tích hợp Object Storage (S3/MinIO), Firebase Cloud Messaging (FCM) và giám sát lỗi cơ bản (Sentry).
*   **Phase 2 (Mở rộng nghiệp vụ)**: Tích hợp Email thông báo (SMTP/Amazon SES) và ChatOps an ninh (Slack/Teams Webhook).
*   **Phase 3 (Enterprise - Vận hành quy mô lớn)**: Tích hợp tin nhắn viễn thông (SMS & Zalo Cloud ZNS) làm kênh fallback khẩn cấp.

## 1.3 Ma trận Xử lý Lỗi Tích hợp (Failure Matrix)

| Dịch vụ tích hợp | Vai trò chính | Giao thức & Cổng | Giai đoạn | Tác động khi lỗi (Failure Impact) | Chiến lược xử lý & Phục hồi lỗi (Failure Recovery Strategy) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Object Storage (S3)** | Lưu trữ ảnh/video báo hỏng xe và chứng từ vật tư. | HTTPS (Port 443) | Phase 1 | **Medium**: Người dùng không thể xem hoặc tải tệp đính kèm. Nghiệp vụ tạo ticket vẫn chạy bình thường nhưng thiếu ảnh minh họa. | - Client tự động thử lại (Retry) 3 lần.<br>- Sử dụng GC Worker quét dọn dữ liệu rác trên DB nếu tệp vật lý chưa được tải lên thành công. |
| **Firebase Messaging (FCM)** | Đẩy thông báo tức thời (Push notifications) tới KTV, Tài xế. | HTTPS (Port 443) | Phase 1 | **Low**: Người dùng không nhận được cảnh báo realtime tức thời. Họ vẫn thấy danh sách thông báo khi mở hộp thư (In-app Inbox) trong ứng dụng. | - BullMQ tự động retry lũy thừa (Exponential Backoff) tối đa 5 lần.<br>- Fallback sang gửi SMS/Zalo ở Phase 3 đối với cảnh báo khẩn cấp (`CRITICAL`). |
| **Sentry Exception Tracker** | Ghi nhận lỗi crash trên thiết bị người dùng và server. | HTTPS (Port 443) | Phase 1 | **None**: Không ảnh hưởng tới người dùng cuối. Mất khả năng giám sát ngoại lệ tạm thời. | - Sentry Client SDK tự lưu cache lỗi cục bộ (Local Storage/Memory) và gửi lại khi mạng phục hồi. |
| **Email Service (SMTP/SES)** | Gửi email báo cáo hàng tuần và khôi phục mật khẩu. | SMTP/S (Port 465/587) | Phase 2 | **Low**: Khách hàng hoặc quản lý nhận báo cáo chậm hơn bình thường. | - Gửi bất đồng bộ thông qua BullMQ, thiết lập retry tối đa 5 lần. |
| **Slack/Teams Webhook** | Gửi cảnh báo bảo mật khẩn cấp (Incident alerts) cho IT Admin. | HTTPS (Port 443) | Phase 2 | **Low**: Admin phát hiện sự cố an ninh chậm hơn qua ChatOps, vẫn có thể giám sát qua file log. | - Ghi nhận log lỗi mức `CRITICAL` cục bộ.<br>- Không retry dồn dập tránh spam và làm nghẽn thread. |
| **SMS & Zalo OA Gateway** | Kênh liên lạc dự phòng tối khẩn (Fallback) khi FCM thất bại. | HTTPS (Port 443) | Phase 3 | **Low**: Người dùng ngoại tuyến không nhận được tin nhắn SMS thay thế. | - Ghi nhận trạng thái gửi `FAILED` vào CSDL.<br>- Đưa thông tin vào hàng chờ lỗi (DLQ) để admin thao tác gửi lại thủ công. |

---

# Level 2 — Integration Adapter Layer (Lớp Adapter Tích hợp)

Để cách ly mã nguồn core nghiệp vụ khỏi sự thay đổi của các SDK bên thứ ba, FixTrack thiết lập cấu trúc thư mục phân lớp tích hợp chặt chẽ.

## 2.1 Cấu trúc Thư mục Tích hợp (Directory Layout)

Toàn bộ logic kết nối ngoại vi được gom nhóm trong thư mục `src/integrations/`:

```text
src/
├── integrations/
│   ├── storage/
│   │   ├── storage.interface.ts          # Cổng giao tiếp chung của Object Storage
│   │   ├── s3.adapter.ts                 # Triển khai kết nối AWS S3 / Cloudflare R2
│   │   └── minio.adapter.ts              # Triển khai kết nối MinIO (Local Dev)
│   ├── notification/
│   │   ├── push-provider.interface.ts    # Cổng giao tiếp gửi tin nhắn đẩy
│   │   └── fcm.adapter.ts                # Triển khai kết nối Firebase Admin SDK
│   └── messaging/
│       ├── message-sender.interface.ts   # Cổng giao tiếp gửi tin viễn thông
│       ├── sms.adapter.ts                # Triển khai tích hợp eSMS / Twilio Gateway
│       └── zalo.adapter.ts               # Triển khai tích hợp Zalo Cloud (ZNS)
```

## 2.2 Quy định Thiết kế Code (Coding Standard)

Các mô-đun nghiệp vụ tuyệt đối không import trực tiếp SDK bên thứ ba. 

*   **Sai (Bad)**:
    ```typescript
    import * as admin from 'firebase-admin'; // Vi phạm nguyên tắc cách ly

    @Injectable()
    export class NotificationService {
      async send(token: string, msg: string) {
        await admin.messaging().send({ token, notification: { body: msg } });
      }
    }
    ```
*   **Đúng (Good)**:
    Sử dụng NestJS Dependency Injection để inject interface token. Tại runtime, hệ thống tự chọn Adapter phù hợp thông qua cấu hình môi trường.
    ```typescript
    // src/integrations/notification/push-provider.interface.ts
    export interface IPushProvider {
      sendPush(token: string, title: string, body: string, data?: Record<string, string>): Promise<void>;
    }

    // src/integrations/notification/fcm.adapter.ts
    @Injectable()
    export class FcmAdapter implements IPushProvider {
      constructor(@Inject('FCM_ADMIN') private readonly fcm: admin.messaging.Messaging) {}

      async sendPush(token: string, title: string, body: string, data?: Record<string, string>): Promise<void> {
        try {
          await this.fcm.send({ token, notification: { title, body }, data });
        } catch (error) {
          throw new ExternalIntegrationException('FCM_SEND_FAILED', error.message);
        }
      }
    }
    ```

---

# Level 3 — Object Storage Integration (Tích hợp Lưu trữ Đối tượng S3)

Để tránh hiện tượng Backend NestJS quá tải băng thông khi nhiều người dùng cùng lúc tải tệp đính kèm (ảnh chụp xe hỏng, video mô tả lỗi), hệ thống áp dụng cơ chế **Direct-to-S3 Upload** kết hợp quét virus bất đồng bộ.

## 3.1 Luồng tuần tự Tải lên & Quét virus (Upload & Quarantine Flow)

Quy trình tải lên tệp tin vật lý diễn ra theo sơ đồ tuần tự dưới đây:

```mermaid
sequenceDiagram
    autonumber
    participant Client as Client (FE App)
    participant Nest as NestJS Backend (BE)
    participant S3Main as S3 Main Bucket
    participant S3Quar as S3 Quarantine Bucket
    participant Worker as BullMQ Worker
    participant ClamAV as ClamAV Container

    Client->>Nest: 1. Yêu cầu cấp Presigned URL (độ dài tệp, MIME)
    Note over Nest: Kiểm tra định dạng tệp & kích thước cho phép
    Nest-->>Client: 2. Trả về Presigned URL + Storage Key độc bản
    Client->>S3Main: 3. PUT File trực tiếp lên S3 (bằng Presigned URL)
    S3Main-->>Client: 4. HTTP 200 OK (Upload vật lý hoàn tất)
    Client->>Nest: 5. Xác nhận upload thành công (/attachments/confirm-upload)
    Note over Nest: Tạo bản ghi DB ở trạng thái PENDING
    Nest->>Worker: 6. Đẩy tác vụ quét virus vào BullMQ
    Nest-->>Client: 7. Phản hồi xác nhận thành công
    
    Note over Worker: Worker nhận job quét virus
    Worker->>ClamAV: 8. Truyền dữ liệu file (stream) qua TCP socket để quét
    ClamAV-->>Worker: 9. Trả về kết quả (CLEAN / INFECTED)
    
    alt Kết quả: CLEAN (Tệp sạch)
        Worker->>Nest: Cập nhật trạng thái DB thành ACTIVE
    else Kết quả: INFECTED (Có virus / mã độc)
        Worker->>S3Quar: 10. Sao chép tệp tin vật lý sang Quarantine Bucket
        Worker->>S3Main: 11. Xóa tệp tin gốc khỏi Main Bucket lập tức
        Worker->>Nest: 12. Cập nhật trạng thái DB thành QUARANTINED
        Worker->>Slack: 13. Gửi cảnh báo bảo mật khẩn cấp (Security Alert)
    end
```

## 3.2 Cấu hình Môi trường Tích hợp (S3 Configuration)

Mã nguồn sử dụng SDK `@aws-sdk/client-s3` và `@aws-sdk/s3-request-presigner`. Biến môi trường cấu hình tại CSDL/Docker Compose:

```properties
S3_ENDPOINT=https://s3.ap-southeast-1.amazonaws.com # Hoặc http://localhost:9000 đối với MinIO Dev
S3_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
S3_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
S3_MAIN_BUCKET=fixtrack-attachments-prod
S3_QUARANTINE_BUCKET=fixtrack-quarantine-prod
S3_PRESIGNED_TTL_SECONDS=900 # Presigned URL tồn tại trong 15 phút
```

---

# Level 4 — Push Notification Integration (Firebase Cloud Messaging)

FixTrack sử dụng dịch vụ thông báo đẩy (Push notification) thông qua Firebase Cloud Messaging để thông tin lập tức hiển thị trên thanh trạng thái di động của Tài xế và KTV.

## 4.1 Quản lý Vòng đời Thiết bị & FCM Token (Token Lifecycle Management)

Để tránh lãng phí tài nguyên gửi tin nhắn và đảm bảo tính chính xác của đối tượng nhận tin:

*   **Đăng ký Token**: Khi người dùng đăng nhập thành công trên ứng dụng di động, React Native App lấy FCM Token từ SDK và gửi lên Backend qua API `POST /users/fcm-token` để lưu trữ vào bảng `user_fcm_tokens`.
*   **Tự động dọn dẹp Token hết hạn (Auto Cleanup)**:
    Khi Backend gọi FCM Gateway gửi tin, nếu nhận được mã lỗi phản hồi báo hiệu Token không còn tồn tại hoặc hết hạn (`messaging/registration-token-not-registered` hoặc `messaging/invalid-argument`):
    1.  FCM Adapter phát hiện mã lỗi đặc trưng.
    2.  Lập tức gọi `usersService.removeFcmToken(token)` để xóa bỏ token này ra khỏi CSDL.
    3.  Ngăn chặn việc tiếp tục gửi tin lỗi vào các chu kỳ sau.

## 4.2 Xử lý Trạng thái ứng dụng trên Frontend (FCM Client-side)

Trên ứng dụng di động React Native (sử dụng thư viện `@react-native-firebase/messaging`), mã nguồn tích hợp xử lý đầy đủ 3 trạng thái hoạt động:

```javascript
import messaging from '@react-native-firebase/messaging';

// 1. Trạng thái Foreground (Ứng dụng đang mở và đang sử dụng)
messaging().onMessage(async remoteMessage => {
  // Không hiển thị banner hệ thống mặc định.
  // Hiển thị Banner Custom nội bộ (Toast Notification) để tăng trải nghiệm.
  showCustomToast(remoteMessage.notification.title, remoteMessage.notification.body);
});

// 2. Trạng thái Background (Ứng dụng đang thu nhỏ, chạy ngầm)
messaging().setBackgroundMessageHandler(async remoteMessage => {
  // Hệ điều hành tự động hiển thị Notification Banner tiêu chuẩn.
  console.log('Nhận thông báo trong nền:', remoteMessage);
});

// 3. Trạng thái Killed / Terminated (Ứng dụng đã bị tắt hoàn toàn)
messaging().getInitialNotification().then(remoteMessage => {
  if (remoteMessage) {
    // Người dùng nhấn vào thông báo để mở app
    // Điều hướng (Deep Link) trực tiếp đến màn hình chi tiết phiếu sửa chữa
    navigateToTicketDetail(remoteMessage.data.ticketId);
  }
});
```

---

# Level 5 — SMS / Email / Webhooks (Tích hợp Kênh bổ trợ)

## 5.1 SMS & Zalo Cloud (ZNS) Gateway (Giai đoạn Phase 3)

Tích hợp thông qua REST API Client gửi yêu cầu HTTP POST tới cổng thanh toán tin nhắn viễn thông (ví dụ: eSMS). Giao tiếp bảo mật bằng mã khóa Bearer Token trong tiêu đề (Header Authorization).

*   *Cấu trúc JSON Payload gửi tin nhắn SMS fallback khẩn cấp*:
    ```json
    {
      "to": "84908123456",
      "provider": "ESMS",
      "message_type": "OTP_OR_URGENT",
      "content": "[FixTrack] YEU CAU KHAN CAP: Xe tai 29C-123.45 gap su co nghiem trong tai Quoc lo 1A. Vui long kiem tra va xac nhan phan cong sua chua."
    }
    ```

## 5.2 SMTP / AWS SES Email Service (Giai đoạn Phase 2)

Sử dụng thư viện `nodemailer` trên NestJS tích hợp dịch vụ Amazon Simple Email Service (SES) để gửi mail giao dịch:

*   *JWT Secret Rotation & Password Reset*: Gửi email chứa link token đổi mật khẩu (TTL 10 phút).
*   *Báo cáo hiệu năng vận hành*: Định kỳ gửi file PDF báo cáo chỉ số sửa chữa xe cho Ban Giám đốc vào mỗi sáng thứ Hai.

## 5.3 Bảo mật Webhooks (Webhook Security Rules)

Để tránh các cuộc tấn công giả mạo yêu cầu (Request Spoofing) hoặc tấn công phát lại (Replay Attacks) vào các cổng Webhook tiếp nhận dữ liệu (ví dụ: webhook nhận thông báo từ bên dịch vụ viễn thông cập nhật trạng thái tin nhắn SMS đã gửi):

Hệ thống bắt buộc triển khai 3 tầng kiểm soát tại Webhook Guard:

1.  **Shared Secret Signature Verification**: Payload gửi đi bắt buộc phải được băm kèm khóa bí mật dùng chung (Shared Secret Key). Cổng nhận Webhook của FixTrack sẽ tính toán lại chữ ký HMAC-SHA256 và so khớp với header `X-FixTrack-Signature`.
2.  **Timestamp Validation (Chống Replay Attack)**: Header gửi kèm bắt buộc có `X-FixTrack-Timestamp`. Webhook Guard sẽ so sánh thời gian nhận được với thời gian gửi trong header. Nếu lệch quá **300 giây (5 phút)**, request sẽ bị từ chối lập tức.
3.  **Nonce Tracking**: Đính kèm một mã ngẫu nhiên dùng một lần `X-FixTrack-Nonce` lưu trong Redis (với TTL bằng 5 phút) để phát hiện và ngăn chặn việc gửi lại nguyên văn một gói tin cũ đã chặn trước đó.

```typescript
// Sơ đồ tính toán Chữ ký Webhook
Signature = HMAC_SHA256(secret, timestamp + "." + nonce + "." + JSON_Stringify(payload))
```

---

# Level 6 — Monitoring & Diagnostics (Giám sát & Chẩn đoán Tích hợp)

Để duy trì khả năng giám sát chặt chẽ hoạt động tích hợp, hệ thống áp dụng kiến trúc quan trắc 3 trụ cột:

```
                            Monitoring & Diagnostics
                                       │
         ┌─────────────────────────────┼─────────────────────────────┐
         ▼                             ▼                             ▼
   Error Tracking              Performance Metrics           Distributed Tracing
 (Sentry BE & FE)             (Prometheus Exporter)        (OpenTelemetry & Span)
```

## 6.1 Giám sát Lỗi Hệ thống (Sentry Integration)

*   **Sentry Backend**: NestJS khai báo một `SentryExceptionFilter` toàn cục để lọc và gửi các lỗi Server (HTTP Status Code `5xx`) lên Sentry Cloud kèm theo context của request:
    *   `trace_id` (Request ID để liên kết logs).
    *   `user_id` (Định danh người dùng gặp lỗi).
    *   `route_path` & `method`.
*   **Sentry Frontend (React Native)**: Tích hợp SDK bắt lỗi crash ứng dụng ở luồng Runtime JS (JS Crash). Ghi nhận các thông số thiết bị như dung lượng pin, kết nối mạng, phiên bản hệ điều hành Android/iOS khi xảy ra crash.

## 6.2 Thu thập Metrics hiệu năng (Prometheus Exporter)

*   Sử dụng thư viện `prom-client` expose một endpoint bảo mật `/metrics` (chỉ cho phép IP của máy chủ Prometheus cào dữ liệu).
*   Ghi nhận số lượng cuộc gọi tới dịch vụ ngoài (External Request Counter), thời gian phản hồi (External API Latency Histogram) của S3, FCM và SMS Gateway để phát hiện sớm các điểm nghẽn mạng hoặc dịch vụ bên ngoài bị chậm.

## 6.3 Truy vết Giao dịch Phân tán (Distributed Tracing với OpenTelemetry)

Khi một yêu cầu đi qua nhiều phân lớp dịch vụ và hàng chờ bất đồng bộ:

*   **Trace Propagation**: Sử dụng tiêu chuẩn **W3C Trace Context** đính kèm tiêu đề `traceparent` vào mọi yêu cầu kết nối HTTP ra bên ngoài hoặc đưa vào Job Payload khi đẩy vào hàng chờ BullMQ.
*   **BullMQ Trace Propagation**: 
    Khi NestJS API tạo một Job gửi thông báo (Push Job) trong Queue, nó sẽ truyền `trace_id` hiện tại vào metadata của Job đó. Khi BullMQ Worker nhấc Job lên để xử lý, nó sẽ kế thừa `trace_id` này để ghi nhận logs. Nhờ đó, kỹ sư vận hành có thể truy vết toàn bộ vòng đời của yêu cầu:
    ```text
    [HTTP POST /api/tickets] (trace_abc123) 
      ──► [DB Insert Ticket] (trace_abc123) 
      ──► [Push BullMQ Job] (trace_abc123) 
      ──► [Worker Process Job] (trace_abc123) 
      ──► [FCM API Call] (trace_abc123)
    ```

---

# Level 7 — Resilience Patterns (Các mẫu thiết kế chống chịu lỗi)

Dịch vụ bên thứ ba mặc định là không đáng tin cậy. FixTrack áp dụng các mẫu thiết kế phục hồi để bảo vệ tính toàn vẹn hệ thống:

```
Request ──► [Circuit Breaker (Closed?)] ──► [Timeout Guard] ──► [External Call]
                  │                                                    │
               (Open)                                               (Failed)
                  ▼                                                    ▼
            Fail-fast Response                                 [Retry & Backoff]
                                                                       │
                                                                 (Max Retries?)
                                                                       ▼
                                                             [Dead Letter Queue]
```

## 7.1 Giới hạn Thời gian gọi API (Timeout Guard)

Mọi yêu cầu HTTP gửi tới bên thứ ba thông qua `HttpService` (Axios wrapper) bắt buộc phải xác định cờ `timeout` cụ thể:
*   FCM Gateway: `timeout: 3000` (3 giây).
*   SMS Gateway: `timeout: 4000` (4 giây).
*   ClamAV Service: `timeout: 5000` (5 giây).

## 7.2 Thử lại thông minh (Exponential Backoff Retry)

Đối với các tác vụ chạy ngầm trên BullMQ (gửi Push, gửi Email), khi gặp sự cố mạng tạm thời, hệ thống sử dụng cơ chế thử lại tăng dần độ trễ để tránh làm nghẽn hoặc bị chặn (Rate Limited) bởi bên thứ ba:

*   *Cấu hình thử lại trên BullMQ*:
    *   `attempts`: 5 (Thử lại tối đa 5 lần).
    *   `backoff`: `{ type: 'exponential', delay: 1000 }` (Độ trễ thử lại lần lượt là: 1s, 2s, 4s, 8s, 16s).

## 7.3 Mẫu thiết kế Ngắt mạch (Circuit Breaker Pattern)

Khi dịch vụ bên thứ ba bị sập hoàn toàn (ví dụ: FCM Gateway phản hồi lỗi `503` liên tục), việc tiếp tục gửi hàng nghìn yêu cầu sẽ làm kẹt luồng xử lý và cạn kiệt bộ nhớ hệ thống. FixTrack sử dụng thư viện `opossum` để thiết lập Circuit Breaker tại các đầu kết nối chính:

*   **Thông số ngắt mạch cấu hình**:
    *   `errorThresholdPercentage`: `50` (Tự động chuyển sang trạng thái ngắt mạch **Open** nếu tỷ lệ lỗi kết nối vượt quá 50% trong chu kỳ theo dõi).
    *   `rollingCountTimeout`: `60000` (Chu kỳ theo dõi rolling window dài 1 phút).
    *   `resetTimeout`: `300000` (Giữ mạch ở trạng thái Open trong vòng 5 phút. Trong thời gian này, mọi yêu cầu gửi đến dịch vụ ngoại vi sẽ bị chặn lại ngay lập tức tại cổng và trả về lỗi ngắt mạch nhanh - Fail-fast).
    *   Sau 5 phút, mạch chuyển sang trạng thái **Half-Open** để thử gửi một lượng nhỏ request dò đường. Nếu thành công, mạch đóng lại (**Closed**) quay về trạng thái vận hành bình thường.

## 7.4 Hàng chờ lỗi (Dead Letter Queue - DLQ)

Nếu một tác vụ gửi thông báo khẩn cấp hoặc xử lý nghiệp vụ thất bại hoàn toàn sau 5 lần thử lại trên BullMQ:
1.  Hệ thống chuyển trạng thái của Job thành `FAILED` và tự động đẩy Job này vào hàng chờ lỗi riêng biệt (`notification-dlq`).
2.  Gửi tin nhắn cảnh báo khẩn cấp (Security/System Incident Alert) tới kênh Slack hỗ trợ kỹ thuật của đội ngũ vận hành.
3.  Job được lưu giữ vĩnh viễn trên cụm Redis để kỹ sư hệ thống có thể truy cập, phân tích nguyên nhân lỗi và kích hoạt chạy lại bằng tay (Manual Trigger) sau khi đã xử lý xong sự cố bên thứ ba.
