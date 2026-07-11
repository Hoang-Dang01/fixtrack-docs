# 33 Monitoring, Logging & Observability

**Trạng thái: Hoàn thành (Finalized - Version 1.0)**

---

## Purpose

Chương này đặc tả chi tiết kiến trúc giám sát, nhật ký vận hành và khả năng quan trắc (Monitoring, Logging & Observability Specification) của hệ thống FixTrack. Tài liệu định nghĩa sơ đồ 4 luồng dữ liệu quan trắc (Metrics, Logs, Traces, Exceptions), quy chuẩn ghi nhật ký có cấu trúc và mặt nạ bảo mật (PII Masking), các chỉ số hiệu năng và cam kết dịch vụ (SLI/SLO), cấu hình thiết kế bảng điều khiển Grafana, quy tắc cảnh báo Prometheus Alertmanager chia cấp độ nghiêm trọng, và quy trình quản lý sự cố lỗi phát sinh qua Sentry.

---

## Inputs

*   [26 System Architecture](../part_f_solution_architecture/26_system_architecture.md)
*   [28 Security Architecture](../part_f_solution_architecture/28_security_architecture.md)
*   [29 Integration Architecture](../part_f_solution_architecture/29_integration_architecture.md)
*   [30 Infrastructure](./30_infrastructure.md)
*   [31 Deployment](./31_deployment.md)

---

# Level 1 — Observability Pipeline (Đường dẫn Quan trắc Tổng thể)

Kiến trúc quan trắc của FixTrack được tổ chức chặt chẽ thành 4 đường dẫn dữ liệu biệt lập nhằm trả lời toàn diện trạng thái hoạt động của hệ thống:

```mermaid
graph TD
    subgraph HostSystem[FixTrack Servers]
        App[NestJS App Container]
        Worker[BullMQ Worker Container]
        DB[Postgres Container]
        Redis[Redis Container]
    end

    %% 1. Metrics Pipeline
    subgraph MetricsFlow[Metrics Pipeline]
        DB_Exp[Postgres Exporter]
        Red_Exp[Redis Exporter]
        Node_Exp[Node Exporter]
        cAdv[cAdvisor Container]
        Prom[Prometheus Server]
    end
    DB --> DB_Exp
    Redis --> Red_Exp
    HostSystem --> Node_Exp & cAdv
    DB_Exp & Red_Exp & Node_Exp & cAdv -->|Scrape Metrics| Prom

    %% 2. Logging Pipeline
    subgraph LoggingFlow[Logging Pipeline]
        Docker_Log[Docker JSON Logs]
        Tail[Promtail Collector]
        Loki[Grafana Loki]
    end
    HostSystem -->|stdout/stderr| Docker_Log
    Docker_Log --> Tail
    Tail -->|Push Logs| Loki

    %% 3. Tracing Pipeline
    subgraph TracingFlow[Tracing Pipeline]
        OTel[OpenTelemetry SDK]
        Jaeger[Jaeger / Grafana Tempo (Future)]
    end
    App & Worker -->|Propagate traceparent| OTel
    OTel -->|Export Spans| Jaeger

    %% 4. Exception Pipeline
    subgraph ExceptionFlow[Exception Pipeline]
        SentrySDK[Sentry SDK (BE & FE)]
        SentryCloud[Sentry Cloud Gateway]
    end
    App & Worker -->|Catch Uncaught Exception| SentrySDK
    SentrySDK -->|Push Crash Logs| SentryCloud

    %% Visualization & Alerts
    Prom & Loki & Jaeger --> Grafana[Grafana Dashboards]
    Prom --> Alertmanager[Prometheus Alertmanager]
    Alertmanager & SentryCloud -->|ChatOps Alerts| Slack[Slack / SMS Alert Channels]
```

> [!NOTE]
> **Lưu ý về Tracing Visualization:**
> Jaeger có thể chạy giao diện UI riêng biệt hoặc tích hợp trực tiếp làm nguồn dữ liệu (Data Source) bên trong Grafana (hoặc kết nối qua Grafana Tempo ở Phase 2) để phân tích trực quan hóa biểu đồ cuộc gọi phân tán (traces diagram).

---

# Level 2 — Logging Strategy & PII Masking (Chiến lược Nhật ký & Bảo mật)

Hệ thống ghi nhận log thống nhất để hỗ trợ truy vết lỗi nhanh chóng và bảo vệ thông tin cá nhân của người dùng.

## 2.1 Định dạng Log JSON có cấu trúc (Structured Logging)

Trên môi trường Production, ứng dụng NestJS bắt buộc phải ghi log ra luồng xuất chuẩn (`stdout`/`stderr`) dưới định dạng JSON một dòng. Cấu trúc chuẩn hóa của một bản ghi log bao gồm:

```json
{
  "timestamp": "2026-06-30T08:50:12.345Z",
  "level": "error",
  "service_name": "fixtrack-api",
  "trace_id": "abc123xyz456",
  "user_id": "U-0032",
  "ip_address": "115.79.22.18",
  "method": "POST",
  "path": "/api/v1/repair-jobs/assign",
  "message": "Không thể cập nhật trạng thái phân công: Thợ sửa chữa đang bận.",
  "stack_trace": "Error: Technician is busy\n    at RepairJobsService.assign (dist/repair-jobs.service.js:45:12)..."
}
```

## 2.2 Chính sách Cấp độ Log (Log Level Policy)

Hệ thống phân định 5 cấp độ log chuẩn để cấu hình lọc thông tin trên các môi trường:
*   **DEBUG**: Log chi tiết phát triển (các câu truy vấn DB thô, dữ liệu payload API). **Tắt hoàn toàn** trên Production để tránh phình dung lượng log và lộ secrets.
*   **INFO**: Ghi nhận các sự kiện nghiệp vụ thông thường (khởi động app, người dùng đăng nhập, tạo job thành công).
*   **WARN**: Cảnh báo lỗi không chí mạng (FCM push fail đã được retry, cache miss).
*   **ERROR**: Các ngoại lệ phát sinh làm gián đoạn request của người dùng nhưng app vẫn hoạt động được (Lỗi validate đầu vào, lỗi nghiệp vụ chặn lại).
*   **FATAL / CRITICAL**: Lỗi nghiêm trọng đe dọa sự sống còn của hệ thống (sập kết nối DB, Redis, tràn ổ đĩa cứng, lỗi bảo mật tệp đính kèm nhiễm virus).

## 2.3 Che giấu Dữ liệu Nhạy cảm (PII Masking & Redaction)

Để bảo vệ thông tin định danh cá nhân (PII) và an toàn mã khóa theo Chapter 28, lớp Logging Interceptor của NestJS bắt buộc phải lọc và thay thế các trường nhạy cảm trước khi ghi ra stdout:

*   **Mã xác thực (JWT / Bearer Token / Password)**:
    *   *Input Header*: `Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...`
    *   *Log output*: `Authorization: Bearer eyJhbGc...***[REDACTED]***`
*   **Thông tin người dùng (Phone number / Email)**:
    *   *Input*: `"phone": "0908123456"`
    *   *Log output*: `"phone": "090****456"`
*   *Quy tắc*: Tuyệt đối không log thông tin raw của mật khẩu người dùng (`password`, `oldPassword`, `newPassword`).

## 2.4 Chính sách Lưu trữ Nhật ký (Log Retention Policy)

| Loại Nhật ký (Log Type) | Vùng lưu trữ (Storage) | Chu kỳ Lưu trữ (Retention) | Mục tiêu / Lý do |
| :--- | :--- | :---: | :--- |
| **Application Logs** | Grafana Loki | **30 ngày** | Hỗ trợ điều tra lỗi phát sinh và tối ưu chi phí dung lượng. |
| **Security Audit Logs** | PostgreSQL (Schema chuyên biệt) | **180 ngày** | Phục vụ hậu kiểm (Forensics) các sự kiện truy cập bất thường. |
| **System Audit Logs** | CSDL lưu trữ ngoài (S3/Cold Storage) | **3 năm** | Đáp ứng tiêu chuẩn tuân thủ và kiểm toán doanh nghiệp. |

---

# Level 3 — Metrics Specification & SLI/SLO (Đặc tả Metrics & Cam kết dịch vụ)

FixTrack quản lý chất lượng vận hành bằng cách định nghĩa các chỉ số đo lường dịch vụ (SLI) hướng tới mục tiêu cam kết độ ổn định (SLO) cho người dùng cuối.

## 3.1 Bảng chỉ tiêu SLI / SLO & Error Budget

Hạ tầng hệ thống được thiết kế để duy trì các chỉ số chất lượng dịch vụ sau:

| Chỉ số dịch vụ (SLI) | Phương pháp đo lường | Mục tiêu cam kết (SLO) | Ngân sách Lỗi (Error Budget) |
| :--- | :--- | :---: | :---: |
| **API Availability (Tính sẵn sàng)** | Số lượng HTTP request thành công (Status code `2xx`, `3xx`, `4xx`*) trên tổng số request gửi lên API Gateway. | **99.5%** | **0.5%** (Tối đa ~3.6 giờ downtime hoặc lỗi hệ thống mỗi tháng) |
| **API Latency (Độ trễ phản hồi)** | Thời gian xử lý phản hồi API đo tại đầu vào NestJS HTTP Request. | **p95 < 500ms** | **5.0%** tổng số request được phép vượt quá 500ms lúc cao điểm. |
| **Job Processing SLA (Hàng chờ)** | Khoảng thời gian từ lúc Job gửi thông báo (BullMQ) được tạo đến khi Worker xử lý thành công. | **p90 < 10 giây** | **10.0%** số Job chạy nền được phép chậm trễ do nghẽn tải. |

> [!NOTE]
> **Ghi chú về mã lỗi 4xx trong SLO:**
> Các mã lỗi 4xx thông thường phát sinh do phía khách hàng (ví dụ: 401 Unauthorized do hết hạn token, 403 Forbidden do ABAC chặn, 404 do nhập sai URL) được tính là phản hồi xử lý thành công của hệ thống. Tuy nhiên, các đợt tăng vọt (spikes) bất thường của mã lỗi 4xx vẫn được hệ thống theo dõi và cảnh báo độc lập để phát hiện lỗi logic giao diện (FE bugs) hoặc các dấu hiệu dò quét bảo mật trái phép.

## 3.2 Phân loại Chỉ số Giám sát (Metrics Registry)

### Chỉ số Hạ tầng & Hệ thống (System Metrics)
*   `node_cpu_seconds_total`: Tỷ lệ sử dụng CPU của VPS Host.
*   `node_memory_Active_bytes`: Bộ nhớ RAM đang sử dụng thực tế của VPS.
*   `container_memory_usage_bytes`: Bộ nhớ tiêu hao cho từng container riêng biệt (phát hiện rò rỉ RAM API).
*   `pg_stat_database_numbackends`: Số lượng kết nối đang hoạt động tới PostgreSQL.
*   `redis_connected_clients`: Số lượng client đang kết nối tới các cụm Redis.

### Chỉ số Nghiệp vụ Doanh nghiệp (Business KPIs)
Hệ thống thu thập số liệu nghiệp vụ từ cơ sở dữ liệu để cảnh báo các sự cố quy trình:
*   `fixtrack_active_repair_tickets`: Tổng số phiếu báo hỏng xe đang ở trạng thái chưa xử lý (`reported` hoặc `assigned`).
*   `fixtrack_overdue_repair_tickets`: Số lượng phiếu sửa chữa bị trễ hạn hoàn thành so với cam kết SLA sửa chữa.
*   `fixtrack_waiting_material_approvals`: Số lượng yêu cầu duyệt xuất kho phụ tùng đang bị nghẽn chờ Quản lý xưởng ký duyệt.
*   `fixtrack_material_approval_latency_seconds`: Thời gian nghẽn trung bình của một phiếu duyệt vật tư (nếu vượt quá 2 giờ, hệ thống sẽ cảnh báo leo thang).

---

# Level 4 — Grafana Dashboard Design (Thiết kế Bảng Điều khiển)

Bảng điều khiển Grafana tập trung được tổ chức thành 4 hàng (Row) trực quan để hiển thị nhanh trạng thái vận hành:

```
┌───────────────────────────────────────────────────────────────────────────┐
│ FIXTRACK CENTRAL OBSERVABILITY DASHBOARD                                  │
├───────────────────────────────────────────────────────────────────────────┤
│ [Row 1: Business KPIs & Operations]                                       │
│ ┌─────────────────────────┐ ┌─────────────────────────┐ ┌───────────────┐ │
│ │ Active Tickets: 18      │ │ Overdue Tickets: 2      │ │ Pending Appr:4│ │
│ └─────────────────────────┘ └─────────────────────────┘ └───────────────┘ │
├───────────────────────────────────────────────────────────────────────────┤
│ [Row 2: Application Health (NestJS API & BullMQ)]                         │
│ ┌─────────────────────────┐ ┌─────────────────────────┐ ┌───────────────┐ │
│ │ Request Rate: 12 req/s  │ │ Latency (p95): 180ms    │ │ Queue size: 0 │ │
│ └─────────────────────────┘ └─────────────────────────┘ └───────────────┘ │
├───────────────────────────────────────────────────────────────────────────┤
│ [Row 3: Databases & Cache (Postgres & Redis)]                             │
│ ┌─────────────────────────┐ ┌─────────────────────────┐ ┌───────────────┐ │
│ │ DB Conn: 15 / 100       │ │ Cache Hit Rate: 94.2%   │ │ Redis Mem:512M│ │
│ └─────────────────────────┘ └─────────────────────────┘ └───────────────┘ │
├───────────────────────────────────────────────────────────────────────────┤
│ [Row 4: Host Infrastructure (VPS Server)]                                 │
│ ┌─────────────────────────┐ ┌─────────────────────────┐ ┌───────────────┐ │
│ │ Host CPU: 24% (Green)   │ │ Host RAM: 4.8G / 16G    │ │ Disk Free: 62%│ │
│ └─────────────────────────┘ └─────────────────────────┘ └───────────────┘ │
└───────────────────────────────────────────────────────────────────────────┘
```

---

# Level 5 — Prometheus Alerting Rules (Cấu hình Cảnh báo)

## 5.1 Ma trận Cấp độ Nghiêm trọng Cảnh báo (Severity Matrix)

| Cấp độ (Severity) | Định nghĩa sự cố | Kênh Cảnh báo (Channel) | Hành động khắc phục tự động / Thủ công |
| :--- | :--- | :--- | :--- |
| **Warning** | Các chỉ số chạm ngưỡng cảnh báo sớm, chưa ảnh hưởng đến người dùng cuối ngay (ví dụ: RAM > 80%, CPU > 75%, nghẽn hàng chờ nhẹ). | Slack Channel (`#fixtrack-alerts`) | Tự động ghi nhận log, DevOps Team theo dõi trong giờ làm việc. |
| **Critical** | Sự cố ảnh hưởng trực tiếp đến người dùng hoặc an toàn dữ liệu (ví dụ: API Error Rate > 5%, Disk Free < 15%, phát hiện virus file đính kèm). | Slack + SMS Alert | DevOps trực ban nhận cuộc gọi/SMS và tiến hành xử lý khẩn cấp. Kích hoạt scripts kiểm tra bộ nhớ. |
| **Fatal** | Hệ thống mất hoàn toàn khả năng phục vụ dịch vụ (CSDL sập, sập nguồn VPS Host). | Slack + SMS + Gọi điện tự động (Pager Duty) | DevOps/Sysadmin trực ban xử lý tức thời 24/7. Kích hoạt quy trình DR Runbook (Ch. 30). |

## 5.2 Mẫu File Cấu hình Luật Cảnh báo (`alerts.yml`)

Dưới đây là tệp cấu hình [alerts.yml](file:///c:/Users/Legion/Desktop/IT/FixTrack/prometheus/alerts.yml) mẫu của Prometheus Alertmanager:

```yaml
groups:
  - name: fixtrack-infrastructure-alerts
    rules:
      # 1. Cảnh báo sập máy chủ VPS Host
      - alert: HostDown
        expr: up == 0
        for: 1m
        labels:
          severity: fatal
        annotations:
          summary: "VPS Host sập nguồn!"
          description: "Không thể kết nối đến node exporter trên máy chủ VPS. Hệ thống đã dừng hoạt động quá 1 phút."

      # 2. Cảnh báo sắp tràn đĩa cứng vật lý
      - alert: HostDiskFillingUp
        expr: (node_filesystem_free_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 15
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Ổ cứng VPS sắp đầy (< 15% dung lượng trống)"
          description: "Dung lượng đĩa trống hiện tại là {{ $value | printf \"%.2f\" }}%. Vui lòng chạy dọn dẹp log hoặc mở rộng ổ đĩa."

      # 3. Cảnh báo cạn kiệt kết nối PostgreSQL
      - alert: PostgresConnectionsExceeded
        expr: pg_stat_database_numbackends > 80
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "CSDL PostgreSQL quá tải kết nối (> 80% Connection Limit)"
          description: "Số lượng kết nối active hiện tại là {{ $value }}. Cần kiểm tra rò rỉ kết nối từ phía API instance."

      # 4. Cảnh báo tỷ lệ lỗi HTTP tăng cao đột biến
      - alert: HttpErrorRateHigh
        expr: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100 > 5
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Tỷ lệ lỗi hệ thống HTTP 5xx vượt ngưỡng cho phép (> 5%)"
          description: "Tỷ lệ lỗi hiện tại là {{ $value | printf \"%.2f\" }}% trên tổng số request trong 5 phút qua."

      # 5. Cảnh báo nghẽn luồng nghiệp vụ (Business KPI Alert)
      - alert: MaterialApprovalQueueCongested
        expr: fixtrack_waiting_material_approvals > 50
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "Yêu cầu duyệt xuất kho phụ tùng bị nghẽn (> 50 phiếu chờ)"
          description: "Số lượng yêu cầu đang chờ ký duyệt là {{ $value }}. Cần gửi cảnh báo nhắc nhở tới Quản lý xưởng."
```

---

# Level 6 — Sentry Issue Management & Release Health (Quản lý Lỗi & Sức khỏe Bản phát hành)

FixTrack tận dụng dịch vụ Sentry Cloud để theo dõi thời gian thực các sự kiện crash ứng dụng từ cả phía Server (NestJS) và Client di động (React Native).

## 6.1 Gắn Thẻ Truy vết Bản phát hành (Release Tracking Integration)

Để xác định chính xác nguyên nhân lỗi xuất hiện từ lần triển khai nào, Sentry SDK được nạp các thẻ định danh động cấu hình từ CI/CD:

```typescript
// NestJS Sentry Initialization Config
Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV, // production / staging
  release: `fixtrack-app@${process.env.APP_VERSION}`, // Liên kết trực tiếp phiên bản, ví dụ: fixtrack-app@v1.0.3
  integrations: [
    new Sentry.Integrations.Http({ tracing: true }),
  ],
  // Tỷ lệ lấy mẫu vết (tracesSampleRate) được cấu hình theo môi trường:
  // - Staging: 1.0 (Lấy mẫu 100% để debug toàn bộ lỗi phát sinh)
  // - Production: 0.1 (Lấy mẫu 10% để tối ưu hóa hiệu năng và chi phí truyền tải)
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
});
```

## 6.2 Theo dõi Sức khỏe Bản phát hành (Release Health Metrics)

Đội ngũ vận hành theo dõi 2 chỉ số an toàn chính trên Dashboard của Sentry:
*   **Crash-Free Sessions (Tỷ lệ phiên làm việc không bị sập)**: 
    *   *Mục tiêu*: Luôn duy trì mức **> 99.0%** đối với ứng dụng di động React Native và **> 99.9%** đối với dịch vụ NestJS API.
*   **Regression Detection (Cảnh báo tái xuất hiện lỗi)**:
    Sentry tự động kích hoạt cảnh báo mức độ `CRITICAL` nếu phát hiện một mã lỗi (Issue) đã được đánh dấu là "Đã giải quyết" (Resolved) ở phiên bản trước bất ngờ xuất hiện lại ở phiên bản mới. Điều này giúp ngăn chặn các lỗi logic cũ bị lọt vào production do lỗi gộp nhánh Git (Git merge issue).
