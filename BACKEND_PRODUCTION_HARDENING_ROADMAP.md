# FixTrack Backend — Production Hardening Roadmap

**Trạng thái:** Approved  
**Phạm vi:** Backend only  
**Phạm vi thực thi ngay:** Giai đoạn 1–3  
**Ngoài phạm vi:** Frontend, thay đổi stack, mở rộng nghiệp vụ và refresh-token.

## 1. Mục tiêu và nguyên tắc trạng thái

Roadmap đưa backend từ trạng thái `integration/staging ready` tới khả năng triển khai production-like an toàn hơn. Hoàn thành Giai đoạn 1–3 không đồng nghĩa production-ready hoàn toàn.

Trạng thái được phép công bố sau Giai đoạn 1–3:

> **Operationally hardened for staging and production-like deployment, but not yet fully production-approved.**

Final production approval chỉ được xem xét sau khi hoàn thành security, observability, PostgreSQL CI, backup/restore và release gate ở Giai đoạn 4–8.

## 2. Thứ tự thực hiện

1. Configuration, runtime, secret và CI nền tảng.
2. Migration/seed/startup separation.
3. Health/readiness/shutdown.
4. HTTP, CORS và upload security.
5. Authentication protection.
6. Logging, metrics và audit vận hành.
7. CI đầy đủ, PostgreSQL và concurrency verification.
8. Backup, restore, rollback và production release gate.

Giai đoạn 4 và 5 có thể phát triển song song nhưng phải có acceptance evidence và báo cáo riêng.

## 3. Giai đoạn 1 — Configuration, runtime, secret và CI nền tảng

### Phạm vi

- Định nghĩa cố định `APP_ENV`: `development`, `test`, `staging`, `production`.
- Validation tập trung cho `JWT_SECRET`, `DATABASE_URL`, `CORS_ORIGINS`, token expiry, log level, trusted hosts và cấu hình upload/storage.
- Staging/production từ chối khởi động khi JWT secret thiếu, dưới độ dài quy định hoặc trùng placeholder/default như `secret`, `changeme`, `your-secret-key`, `supersecret`.
- Development/test được phép dùng cấu hình local riêng nhưng phải cảnh báo rõ và không phụ thuộc `.env` của developer trong test.
- Image production không có `--reload`.
- Worker count, keep-alive, proxy headers và graceful timeout cấu hình qua environment; không hard-code tùy ý.
- `.env.example` chỉ chứa placeholder, không chứa credential thật.
- Thiết lập CI nền tảng: compile, regression test và Docker build.

### Acceptance evidence

- Production thiếu secret: startup fail.
- Production dùng placeholder/default: startup fail.
- Staging dùng secret yếu: startup fail.
- Development chạy với cấu hình local được cho phép.
- Test environment độc lập `.env` máy developer.
- Docker runtime không có reload.
- CI nền tảng chạy thành công trên commit phát hành.

## 4. Giai đoạn 2 — Migration, seed và startup separation

### Phạm vi

- Backend startup chỉ chạy API, không gọi Alembic và không gọi seed.
- Migration là deployment task riêng và trả exit code khác 0 khi thất bại.
- Tách `seed-core`, `seed-demo` và fixture test.
- Production không được chạy demo seed.
- Seed core chạy trong transaction, idempotent, không ghi đè dữ liệu operator đã chỉnh, không reset password hoặc thay đổi canonical ID/code ngoài chủ đích.
- Không dùng `Base.metadata.create_all()` trong production startup.
- Không sửa migration đã áp dụng trên staging/production.
- Migration phá vỡ tương thích phải dùng triển khai nhiều bước; data migration lớn phải đánh giá lock time.
- Mỗi migration được phân loại: `reversible`, `conditionally reversible`, `forward-only` hoặc `requires backup restore`.

### Deployment sequence

```text
pre-deployment checks
→ backup
→ verify backup
→ stop/drain traffic nếu cần
→ run migration
→ roll out API
→ readiness verification
→ smoke test
→ monitor
```

### Acceptance evidence

- Restart API không tự thay đổi schema hoặc dữ liệu.
- Có thể migrate mà không khởi động API.
- Có thể deploy production mà không seed.
- Migration đạt trên PostgreSQL sạch và database ở revision ngay trước.
- Seed failure không làm API tự động tiếp tục deploy.
- Rollback procedure chỉ rõ khi dùng downgrade, rollback image/config hay restore backup.

## 5. Giai đoạn 3 — Liveness, readiness, database pool và shutdown

### Phạm vi

- `/health` là liveness, không phụ thuộc PostgreSQL.
- `/ready` chạy truy vấn ngắn `SELECT 1`, có timeout và trả 503 khi database unavailable.
- Response readiness không lộ DB host, username, password, connection string, SQL error hoặc stack trace.
- Docker healthcheck dùng `/ready` để xác nhận API có thể nhận traffic nghiệp vụ.
- Không cấu hình restart policy gây restart loop chỉ vì database tạm thời unavailable.
- Cấu hình pool gồm `pool_size`, `max_overflow`, `pool_timeout` và `pool_recycle` khi phù hợp.
- Không tạo engine theo request; session luôn được đóng.
- Graceful shutdown đóng engine/pool và thống nhất với Docker stop grace period.

### Acceptance evidence

- `/health` trả 200 khi database unavailable.
- `/ready` trả 200 khi PostgreSQL sẵn sàng.
- `/ready` trả 503 an toàn khi PostgreSQL unavailable.
- Readiness timeout hoạt động đúng.
- Docker healthcheck sử dụng `/ready`.
- Startup/shutdown và pool lifecycle có test hoặc bằng chứng runtime.

## 6. Giai đoạn 4 — HTTP, CORS và upload security

- CORS allowlist được parse/validate; không phản chiếu Origin tùy ý và có test preflight.
- Production không dùng wildcard origin cùng credentials.
- Security headers, trusted hosts, proxy headers, request timeout và giới hạn body.
- HSTS chỉ bật khi HTTPS được enforce.
- Chính sách exposure `/docs` và `/redoc` theo môi trường.
- Upload có size limit, MIME allowlist, extension/signature validation, UUID filename, path safety và không thực thi file.
- Response nhạy cảm dùng cache policy phù hợp; error không lộ stack trace.

## 7. Giai đoạn 5 — Authentication protection

- Rate limit kết hợp IP, normalized username và tổng endpoint; cân nhắc NAT và chống lockout abuse.
- Chỉ tin proxy headers từ trusted proxy.
- Response đăng nhập sai thống nhất, không tiết lộ account tồn tại/inactive.
- Audit login success, repeated failure và rate-limit trigger có sampling/aggregation.
- Không log password, raw JWT hoặc refresh token.
- Refresh-token chỉ được tài liệu hóa hướng thiết kế, không triển khai khi chưa có yêu cầu nghiệp vụ.

## 8. Giai đoạn 6 — Logging, metrics và audit

- Structured JSON logging và `X-Request-ID`; chưa bắt buộc full distributed tracing.
- Redaction secret/dữ liệu nhạy cảm.
- Metrics theo route template: request count, latency, status class, error, readiness và auth rate-limit.
- Không dùng raw URL làm metric label.
- Phân biệt application log, security event và business audit.
- Audit có actor, action, target, old/new value, reason, request ID và timestamp.

## 9. Giai đoạn 7 — CI đầy đủ, PostgreSQL và concurrency

- PostgreSQL integration, migration từ clean DB và previous revision, multiple-head check, OpenAPI generation và Docker build.
- Alembic model-drift check; kiểm tra migration phát hành không bị sửa.
- Concurrency tests cho queue uniqueness, inventory issue/return, OCC và idempotent mutation.
- Dependency/container vulnerability scanning.
- GitHub ruleset/branch protection yêu cầu PR, review và required status checks.

## 10. Giai đoạn 8 — Backup, restore, rollback và release gate

- Backup phải được restore thử ở môi trường tách biệt.
- Ghi thời gian restore, kiểm tra dữ liệu, Alembic revision và khả năng ứng dụng kết nối/đọc dữ liệu.
- Xác định RPO/RTO sơ bộ.
- Rollback bao gồm image, config, database downgrade, backup restore hoặc forward hotfix tùy phân loại migration.
- Release artifact/image immutable và có version/tag cụ thể.

Final production acceptance yêu cầu:

- Không còn P0.
- Không còn security critical/high chưa được chấp nhận rủi ro.
- Mọi P1 còn lại có owner, risk assessment, workaround và kế hoạch xử lý.
- Migration staging, smoke test, metrics/logging và backup/restore evidence đều đạt.
- Rollback owner hiện diện trong release window.

## 11. Quy trình bắt buộc sau mỗi giai đoạn

1. Xác nhận phạm vi và baseline commit.
2. Thực hiện thay đổi đúng phạm vi.
3. Chạy regression test.
4. Chạy test mới của giai đoạn.
5. Kiểm tra PostgreSQL/Docker nếu liên quan.
6. Kiểm tra git diff để loại thay đổi ngoài phạm vi.
7. Viết báo cáo bằng chứng vào `fixtrack-docs`.
8. Commit backend và docs bằng revision có thể truy vết.
9. Push và xác nhận CI pass.
10. Chỉ khóa giai đoạn khi toàn bộ acceptance criteria có bằng chứng.

Báo cáo phải ghi rõ file thay đổi, lệnh/test đã chạy, kết quả, migration revision, commit hash, hạn chế còn lại và nội dung chưa thể xác minh.

## 12. Quyết định phê duyệt

> **Approved as the Production Hardening Roadmap for FixTrack Backend. Giai đoạn 1–3 are the immediate approved scope. Completion of these stages improves production deployment safety but does not constitute final production acceptance; final approval requires completion of security, observability, PostgreSQL CI, backup/restore, and release-gate requirements in Giai đoạn 4–8.**
