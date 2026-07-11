# 38 Future Enhancements

**Trạng thái: Hoàn thành (Finalized - Version 1.0)**

---

## Purpose

Chương này đặc tả định hướng và lộ trình phát triển dài hạn (Future Evolution & Scaling Roadmap) của hệ thống FixTrack. Tài liệu đóng vai trò làm khung định hướng cho đội ngũ phát triển và vận hành khi quy mô doanh nghiệp mở rộng, định nghĩa các điều kiện chuyển đổi kiến trúc (Architecture Decision Gates), quy hoạch phân bổ tài nguyên theo giai đoạn, và các chiến lược nâng cấp bảo mật, cơ sở dữ liệu cùng khả năng quan trắc.

---

## Inputs

*   [22 Backend Modules](../part_e_backend_design/22_backend_modules.md)
*   [26 System Architecture](../part_f_solution_architecture/26_system_architecture.md)
*   [28 Security Architecture](../part_f_solution_architecture/28_security_architecture.md)
*   [30 Infrastructure](./30_infrastructure.md)

---

# Level 1 — Business Roadmap & Integrations (Lộ trình Nghiệp vụ)

## 1.1 Tích hợp Hệ thống Quản trị Doanh nghiệp (ERP / Accounting Integration)
*   **Mục tiêu**: Đồng bộ chi phí sửa chữa và danh mục vật tư phụ tùng tự động.
*   **Giải pháp**: Tích hợp FixTrack với hệ thống ERP trung tâm (SAP, Odoo hoặc Oracle) qua cổng API Webhook. Mỗi khi một phiếu yêu cầu vật tư (`material_request`) chuyển sang trạng thái hoàn tất bàn giao, hệ thống sẽ gửi lệnh ghi nhận chi phí (Cost Center Ledger) tương ứng vào phân hệ Kế toán của ERP để tự động khấu hao tài sản.

## 1.2 Bảo trì Dự báo thông qua Trí tuệ Nhân tạo (Predictive Maintenance)
*   **Mục tiêu**: Dự đoán hư hỏng của phương tiện trước khi xảy ra sự cố (chuyển đổi từ bảo trì phản ứng sang bảo trì chủ động).
*   **Điều kiện Tiền đề về Dữ liệu (Prerequisites)**:
    Mô hình Machine Learning (ML) chỉ được kích hoạt phát triển khi hệ thống tích lũy đủ:
    *   Tối thiểu **12 - 24 tháng** lịch sử vận hành và báo hỏng của đội xe.
    *   Tổng số lượng phiếu sửa chữa đã đóng đạt **> 10,000 tickets**.
    *   Dữ liệu được gán nhãn lỗi đầy đủ (Failure Labels) và có ghi nhận kết quả bảo trì rõ ràng.

---

# Level 2 — Architectural Evolution to Microservices (Tiến hoá Kiến trúc)

Mô hình Modular Monolith (NestJS) hiện tại sẽ được phân rã thành các dịch vụ độc lập (Distributed Microservices) để cô lập lỗi và giãn nở tài nguyên độc lập.

```
Modular Monolith (Phase 1)
┌────────────────────────────────────────────────────────┐
│ [Auth Module]  [Ticket Module]  [Notification Module]  │
└────────────────────────────────────────────────────────┘
                           │ (Migration)
                           ▼
Microservices Architecture (Phase 3)
┌──────────────┐     ┌──────────────┐     ┌─────────────────────┐
│  Auth Micro  │     │ Ticket Micro │     │ Notification Micro  │
└──────┬───────┘     └──────┬───────┘     └──────────┬──────────┘
       │                    │                        │
       └──────────────► [ Apache Kafka ] ◄───────────┘
```

## 2.1 Tiêu chí Dịch chuyển Kiến trúc (Microservices Migration Criteria)
Hệ thống **chỉ** được phép phân rã lên Microservices khi chạm các ngưỡng giới hạn vận hành sau:
1.  **Quy mô Người dùng**: Số lượng người dùng hoạt động hàng ngày **DAU > 5,000**.
2.  **Thông lượng API**: Tần suất truy cập API Gateway duy trì ở mức **> 200 req/sec** trong giờ cao điểm.
3.  **Quy mô Đội ngũ**: Đội ngũ kỹ sư phát triển backend vượt quá **8 người** dẫn đến xung đột code và bottleneck về tốc độ release.
4.  **Bottleneck về Deploy**: Nhu cầu deploy độc lập các tính năng (như cổng Notification thời gian thực) bị cản trở bởi quy trình deploy chung của Monolith.

## 2.2 Công nghệ Tương tác Bất đồng bộ
*   Sử dụng **Apache Kafka** hoặc **RabbitMQ** làm trục trung chuyển sự kiện (Event Bus) giữa các Microservices để đảm bảo tính nhất quán dữ liệu cuối (Eventual Consistency).

---

# Level 3 — Enterprise Security Hardening (Bảo mật Đặc quyền)

## 3.1 Động hoá Khoá Bí mật (Dynamic Secrets Management)
*   Dịch chuyển từ việc lưu trữ khóa tĩnh trong tệp `.env.production` sang quản lý tập trung bằng **HashiCorp Vault** hoặc **AWS Secrets Manager**. Kích hoạt cơ chế tự động xoay vòng khóa (Auto Key Rotation) sau mỗi 30 ngày đối với cơ sở dữ liệu và các API Keys bên thứ ba.

## 3.2 Xác thực Đa yếu tố (MFA / 2FA Enforcement)
*   Bắt buộc kích hoạt xác thực 2 lớp sử dụng giao thức **TOTP** (qua Google Authenticator hoặc Microsoft Authenticator) đối với tất cả tài khoản thuộc nhóm vai trò có đặc quyền lớn (`MANAGER`, `INVENTORY`).

## 3.3 Bảo mật Mạng Nội bộ (Mutual TLS - mTLS)
*   Thiết lập mTLS cho toàn bộ giao tiếp giữa các container nội bộ (ví dụ: giữa API Gateway, Worker và Database) để ngăn chặn hoàn toàn nguy cơ nghe trộm dữ liệu (Man-in-the-Middle) trong mạng ảo Docker/Kubernetes.

---

# Level 4 — Database Slicing & High Availability (Mở rộng CSDL)

## 4.1 Cấu hình Tính Sẵn sàng cao (Patroni & Consul Failover)
*   Thiết lập cụm CSDL PostgreSQL Master-Replica.
*   Sử dụng **Patroni** kết hợp **Consul** để liên tục giám sát sức khỏe nút Master. Nếu Master gặp sự cố, hệ thống tự động thăng chức (failover) nút Replica lên làm Master mới dưới 30 giây mà không cần can thiệp thủ công.

## 4.2 Phân mảnh Bảng Dữ liệu lớn (Table Partitioning)
*   Áp dụng kỹ thuật phân mảnh bảng vật lý theo thời gian (Range Partitioning) đối với bảng dữ liệu tăng trưởng nhanh như `audit_logs` và `repair_tickets`.
*   CSDL sẽ chia nhỏ dữ liệu thành các phân vùng theo Năm (ví dụ: `audit_logs_2026`, `audit_logs_2027`) để giữ kích thước các index đủ nhỏ, giữ chi phí truy vấn gần như hằng số trong thực tế sau khi partition pruning (near-constant practical lookup time after partition pruning) khi dữ liệu đạt hàng chục triệu bản ghi.

---

# Level 5 — Distributed Observability & Auto-Scaling (Quan trắc & Giãn nở)

## 5.1 Giám sát Cuộc gọi Phân tán (Distributed Tracing)
*   Tích hợp **OpenTelemetry (OTel) SDK** vào mã nguồn NestJS kết hợp **Grafana Tempo** để truy vết luồng đi của một request xuyên suốt qua các Microservices, hàng chờ BullMQ và Database, giúp cô lập chính xác vị trí gây trễ (bottleneck latency).

## 5.2 Tự động Giãn nở Hạ tầng (Kubernetes HPA)
*   Dịch chuyển toàn bộ hệ thống sang vận hành trên cụm **Kubernetes (AWS EKS)**.
*   Cấu hình **Horizontal Pod Autoscaler (HPA)** để tự động tăng/giảm số lượng Pod ứng dụng chạy thực tế dựa trên các chỉ số tài nguyên:
    *   Tự động scale up khi: CPU Usage > 75% hoặc Memory Usage > 80% liên tục trong 2 phút.
    *   Hạ scale về mức tối thiểu khi tải giảm để tối ưu hóa chi phí đám mây.

---

# Level 6 — Offline Mobile Sync Strategy (Đồng bộ Ngoại tuyến)

Để đảm bảo các tài xế và kỹ thuật viên vận hành ổn định tại các vùng nhà kho hoặc bãi đỗ xe không có sóng mạng/Internet di động:

```
[ Offline Mobile Client ] ──► [ Write to Local DB: WatermelonDB ]
                                            │
                                    (Internet Restored)
                                            ▼
[ State Sync Session ]    ──► [ Conflict Resolution Rule Engine ]
                                            │
                                            ├─► Phase 2: Last-Write-Wins (LWW)
                                            └─► Phase 3: CRDT Data Models
```

## 6.1 Cơ sở dữ liệu Cục bộ (Local DB)
*   Ứng dụng di động (React Native) tích hợp **WatermelonDB** hoặc **SQLite** để lưu trữ trạng thái cục bộ của các phiếu sửa chữa và danh mục phụ tùng được phân công. Người dùng thao tác cập nhật bình thường ở chế độ offline.

## 6.2 Chiến lược Giải quyết Xung đột Dữ liệu (Conflict Resolution)
Khi thiết bị kết nối Internet trở lại, tiến trình đồng bộ (Sync Session) được kích hoạt theo lộ trình phát triển:
*   **Phase 2 (Hiện tại - Đơn giản)**: Áp dụng quy tắc **Last-Write-Wins (LWW)** dựa trên trường `updated_at`. Bản ghi nào có mốc thời gian cập nhật mới nhất từ client sẽ được ghi đè lên database trung tâm.
*   **Phase 3 (Tương lai - Nâng cao)**: Chuyển dịch sang mô hình dữ liệu **CRDT (Conflict-free Replicated Data Type)** đối với các tài nguyên có tính chia sẻ cao (như tồn kho phụ tùng) để tự động hòa trộn thay đổi mà không gây mất mát dữ liệu của các bên.

---

# Level 7 — Operations Roadmap & Decision Gates (Lộ trình Vận hành)

## 7.1 Lộ trình Phân kỳ Tính năng (Phase Roadmap Table)

| Tính năng / Nâng cấp | Phase 1 (MVP) | Phase 2 (HA Scale) | Phase 3 (Enterprise) |
| :--- | :---: | :---: | :---: |
| **Hạ tầng chạy Docker Compose** | ✓ | — | — |
| **Sao lưu CSDL tĩnh cơ bản (pg_dump)** | ✓ | ✓ | ✓ |
| **Xoay vòng khóa thủ công** | ✓ | — | — |
| **Mở rộng Postgres Master-Replica** | — | ✓ | ✓ |
| **Xoay vòng khóa động (Vault)** | — | ✓ | ✓ |
| **Đồng bộ Offline (LWW)** | — | ✓ | ✓ |
| **Phân rã Microservices + Kafka** | — | — | ✓ |
| **Triển khai Kubernetes (EKS)** | — | — | ✓ |
| **Đồng bộ Offline nâng cao (CRDT)** | — | — | ✓ |

## 7.2 Chốt chặn Quyết định Thay đổi Kiến trúc (Architecture Decision Gates)

| Quyết định nâng cấp | Trọng tâm kích hoạt (Trigger Condition) | Lợi ích đạt được |
| :--- | :--- | :--- |
| **Nâng cấp HA Database** | Yêu cầu nghiệp vụ cam kết Downtime CSDL < 1 phút. | Loại bỏ điểm lỗi đơn lẻ (SPOF) ở tầng lưu trữ. |
| **Chuyển dịch Kubernetes** | Số lượng container API chạy thực tế vượt quá 5 instances. | Tự động hóa giám sát, tự phục hồi pod hỏng và tự động giãn nở. |
| **Triển khai Kafka Bus** | Thông lượng sự kiện nội bộ vượt quá 500 events/sec. | Xử lý bất đồng bộ tải cực lớn, chống mất mát tin nhắn sự kiện. |

## 7.3 Ước lượng Tăng trưởng Chi phí (Cost Scaling Considerations)

Hệ thống định lượng chi phí vận hành tăng dần theo quy mô hạ tầng để phục vụ lập kế hoạch tài chính:

*   **Phase 1 (Low Cost)**: Ước tính **~$50 - $100 / tháng**. Chạy trên 1 VPS duy nhất. Rủi ro sập hệ thống cục bộ ở mức chấp nhận được.
*   **Phase 2 (Medium Cost)**: Ước tính **~$300 - $500 / tháng**. Tách biệt CSDL Master-Replica, chạy cụm Redis dự phòng độc lập, chi phí lưu trữ S3 backups.
*   **Phase 3 (High Cost)**: Ước tính **>$1,500 / tháng**. Vận hành cụm Kubernetes chuyên biệt, tích hợp hệ thống Enterprise APM (Datadog/Dynatrace), trục truyền tin Kafka hiệu năng cao, và hạ tầng quản lý Vault HA.
