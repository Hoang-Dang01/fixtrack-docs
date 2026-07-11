# 36 Runbook

**Trạng thái: Hoàn thành (Finalized - Version 1.0)**

---

## Purpose

Chương này đặc tả chi tiết quy trình vận hành và ứng phó sự cố (Operations & Incident Runbook) của hệ thống FixTrack. Tài liệu cung cấp các hướng dẫn chi tiết theo hướng thực hành (command-driven) cho kỹ sư trực vận hành (on-call) nhằm đảm bảo tính sẵn sàng của hệ thống, xử lý thảm họa khẩn cấp, triển khai di cư dữ liệu an toàn, và khôi phục hoạt động sau sự cố.

---

## Inputs

*   [05 User Roles](../part_a_business_foundation/05_user_roles.md)
*   [30 Infrastructure](./30_infrastructure.md)
*   [31 Deployment](./31_deployment.md)
*   [33 Monitoring & Logging](../part_g_devops/33_monitoring_logging.md)

---

# Level 0 — Incident Classification & Escalation (Phân loại & Báo động Sự cố)

Quy trình quản lý sự cố của FixTrack phân định rõ 4 cấp độ nghiêm trọng để tối ưu hóa nguồn lực xử lý và đảm bảo tốc độ phản ứng:

## 0.1 Ma trận Cấp độ Sự cố (Incident Severity Matrix)

| Severity | Ý nghĩa sự cố | Target SLA (Phản ứng / Sửa lỗi) | Ví dụ điển hình |
| :--- | :--- | :---: | :--- |
| **SEV-1** | Hệ thống ngưng hoạt động hoàn toàn (Full Outage). | **< 15 phút** / **< 2 giờ** | Sập CSDL chính, sập Redis B (mất hàng chờ/phiên), VPS sập nguồn. |
| **SEV-2** | Hệ thống suy giảm hiệu năng nghiêm trọng (Major Degradation) hoặc tính năng cốt lõi bị hỏng. | **< 30 phút** / **< 4 giờ** | Lỗi auth không thể đăng nhập, BullMQ bị nghẽn chậm > 15 phút, đĩa cứng VPS trống < 15%. |
| **SEV-3** | Sự cố nhỏ (Minor Degradation) ảnh hưởng tới tính năng phụ, trải nghiệm người dùng chậm nhẹ. | **< 2 giờ** / **< 24 giờ** | Redis A (Cache) sập (sử dụng database fallback chậm hơn), Bull Board không hiển thị. |
| **SEV-4** | Lỗi hiển thị, lỗi thẩm mỹ hoặc tài liệu chưa cập nhật (Cosmetic). | **< 24 giờ** / **1 tuần** | Lỗi chính tả UI, sai lệch nhẹ trong tài liệu API. |

## 0.2 Quy trình Giao tiếp & Ứng phó (Incident Communication Protocol)

Khi phát hiện sự cố **SEV-1** hoặc **SEV-2** từ hệ thống giám sát Prometheus/Alertmanager hoặc Sentry:

1. **Khởi tạo Sự cố (Incident Declaration)**:
   * Lập tức tạo một kênh Slack khẩn cấp với tên dạng `#incident-[YYYYMMDD]-[tên_sự_cố]` (ví dụ: `#incident-20260630-postgres-down`).
   * Gửi thông báo toàn đội và chỉ định **Incident Commander (Chỉ huy Sự cố)** - mặc định là Tech Lead hoặc DevOps trực ban.
2. **Đóng băng Triển khai (Deployment Freeze)**:
   * Áp dụng lệnh đóng băng triển khai tức thời trên toàn hệ thống. Không thực hiện deploy bất kỳ tính năng mới nào trong lúc đang khắc phục sự cố trừ bản vá lỗi khẩn cấp (hotfix).
3. **Ghi nhật ký dòng thời gian (Timeline Logging)**:
   * Incident Commander chịu trách nhiệm cập nhật liên tục các mốc thời gian, hành động khắc phục và trạng thái phản hồi vào kênh sự cố.
4. **Hậu kiểm & Khắc phục Dài hạn (Postmortem)**:
   * Trong vòng **48 giờ** sau khi khắc phục xong sự cố SEV-1/SEV-2, bắt buộc phải tổ chức cuộc họp Postmortem để làm rõ Nguyên nhân gốc rễ (Root Cause), Hành động khắc phục triệt để và cập nhật lại tài liệu Runbook này nếu phát hiện điểm thiếu sót.

---

# Level 1 — System Startup & Shutdown (Khởi động & Dừng hệ thống)

Quy trình quản lý trạng thái các container Docker Compose trên máy chủ VPS Production.

## 1.1 Khởi động Hệ thống theo Thứ tự (Graceful Startup)

Các dịch vụ phải được khởi động tuần tự để đảm bảo các phụ thuộc CSDL và Cache đã sẵn sàng trước khi ứng dụng chính (`api-service`) khởi chạy.

```bash
# 1. Di chuyển vào thư mục dự án trên VPS
cd /var/lib/fixtrack

# 2. Khởi động PostgreSQL và hai cụm Redis trước
docker compose up -d postgres-db redis-cache redis-queue clamav-scanner

# 3. Kiểm tra trạng thái sức khoẻ của các dịch vụ nền (Vòng lặp tự động đợi trạng thái "healthy")
until [ "$(docker inspect --format='{{.State.Health.Status}}' fixtrack-postgres)" = "healthy" ]; do
    echo "Đang đợi Postgres database khởi động và báo healthy..."
    sleep 2
done

# 4. Khi DB và các dịch vụ nền đã healthy, khởi động API và Worker
docker compose up -d api-service worker-service
```

## 1.2 Kiểm tra Trạng thái Runtime (System Status Check)

```bash
# Xem trạng thái tổng quát và sức khoẻ của các container
docker compose ps

# Kiểm tra tài nguyên tiêu thụ của các container thời gian thực
docker stats

# Kiểm tra log của API instance (100 dòng cuối và theo dõi tiếp)
docker compose logs -f --tail=100 api-service

# Kiểm tra log của Worker instance
docker compose logs -f --tail=100 worker-service
```

## 1.3 Dừng Hệ thống An toàn (Graceful Shutdown)

```bash
# Dừng API và Worker trước để ngắt kết nối nhận requests/jobs mới
docker compose stop api-service worker-service

# Chờ 10 giây để các kết nối hiện tại hoàn tất (graceful drain)
sleep 10

# Dừng toàn bộ các container còn lại và giải phóng tài nguyên mạng nội bộ
docker compose down
```

---

# Level 2 — Database Backup & Restore (Sao lưu & Khôi phục CSDL)

## 2.1 Môi trường Hiện tại (Phase 1 — Single VPS)

### Quy trình Sao lưu Thủ công (Manual pg_dump)
Thực hiện sao lưu khẩn cấp trước khi can thiệp hệ thống hoặc nâng cấp lớn:

```bash
# 1. Tạo thư mục chứa backup nếu chưa có
mkdir -p /var/lib/fixtrack/backups

# 2. Thực thi lệnh pg_dump nén gzip trực tiếp từ container (Sử dụng tài khoản chuyên biệt app_db_user thay vì superuser postgres để đảm bảo an toàn bảo mật)
docker exec -t fixtrack-postgres pg_dump -U app_db_user -d fixtrack_prod | gzip > /var/lib/fixtrack/backups/manual_backup_$(date +%F_%H%M%S).sql.gz

# 3. Xác thực dung lượng file backup hợp lệ (> 0 bytes)
ls -lh /var/lib/fixtrack/backups/
```

### Quy trình Khôi phục Thủ công (Manual Restore)
Quy trình khôi phục dữ liệu từ tệp tin backup nén `.sql.gz`:

```bash
# 1. Đưa ứng dụng chính về trạng thái dừng để ngắt toàn bộ kết nối ghi vào DB
docker compose stop api-service worker-service

# 2. Xoá sạch database cũ và khởi tạo database trống (Sử dụng tài khoản app_db_user để thực thi)
docker exec -i fixtrack-postgres dropdb -U app_db_user --if-exists fixtrack_prod
docker exec -i fixtrack-postgres createdb -U app_db_user fixtrack_prod

# 3. Giải nén và nạp dữ liệu từ file backup vào database mới
gunzip -c /var/lib/fixtrack/backups/pre_deploy_backup.sql.gz | docker exec -i fixtrack-postgres psql -U app_db_user -d fixtrack_prod

# 4. Khởi động lại các container ứng dụng
docker compose start api-service worker-service
```

## 2.2 Môi trường Tương lai (Phase 2 — Patroni & WAL Archiving)

> [!NOTE]
> **Định hướng Kế hoạch Tương lai:**
> Lộ trình Phase 2 sẽ kích hoạt cơ chế tự động đồng bộ file WAL lên lưu trữ đám mây S3/Cloudflare R2 liên tục. Khi đó quy trình khôi phục sẽ cho phép chỉ định chính xác mốc thời gian muốn phục hồi (Point-in-Time Recovery - PITR) thông qua công cụ **pgBackRest** hoặc **WAL-G** thay vì khôi phục toàn phần từ bản backup tĩnh hàng ngày.

---

# Level 3 — Deployment Playbook & Checklists (Quy trình Triển khai)

Quy trình phát hành và kiểm soát lỗi khi cập nhật phiên bản ứng dụng trên môi trường Production.

## 3.1 Bảng Kiểm tra Trước khi Triển khai (Pre-deployment Checklist)

Trước khi thực thi script `./deploy.sh [phiên_bản]`, kỹ sư vận hành bắt buộc phải kiểm tra và xác thực các điều kiện sau:

```bash
# 1. Kiểm tra dung lượng đĩa cứng trống của host (Bắt buộc dung lượng trống > 20%)
df -h /

# 2. Kiểm tra bộ nhớ RAM trống của máy chủ (Bắt buộc RAM trống > 1.5 GB)
free -m

# 3. Xác thực CSDL PostgreSQL đang kết nối bình thường
docker exec -it fixtrack-postgres pg_isready -U postgres -d fixtrack_prod

# 4. Xác nhận tiến trình sao lưu tự động gần nhất thành công
ls -lh /var/lib/fixtrack/backups/ | tail -n 1
```

## 3.2 Kích hoạt Di cư CSDL trước khi Nâng cấp (Migration Execution)

Để tránh hiện tượng race condition hoặc crash hệ thống do lệch cấu trúc, lệnh migration được chạy độc lập thông qua một container chạy-xong-xóa (run-once) trước khi rollout container chính:

```bash
# Chạy migration khô (dry-run/check log) trước nếu ORM hỗ trợ, hoặc thực thi trực tiếp:
docker compose run --rm --entrypoint "npm run migration:run" api-service
```

## 3.3 Tiêu chí Kích hoạt Khôi phục Phiên bản cũ (Rollback Triggers)

Sau khi deploy, kỹ sư trực vận hành phải theo dõi các chỉ số trong vòng **15 phút**. Kích hoạt `./rollback.sh` khẩn cấp ngay lập tức nếu xảy ra một trong các điều kiện sau:

1. **Health Check Fail**: Container `fixtrack-api` ở trạng thái `unhealthy` liên tục quá 2 phút sau khi khởi chạy.
2. **Lỗi Nghiệp vụ Core (Auth/Login Broken)**: Kiểm tra log phát hiện lỗi authenticate/xác thực người dùng trả về mã lỗi 500 diện rộng.
3. **Tỷ lệ Lỗi HTTP tăng vọt (Http Error Rate High)**: Tỷ lệ phản hồi HTTP 5xx vượt quá **10%** tổng lưu lượng truy cập trong 3 phút liên tục.
4. **Lỗi Cú pháp Migration (Migration Error)**: CSDL báo lỗi thiếu trường/cột vật lý mà mã nguồn mới yêu cầu do lệnh migration chạy thất bại hoặc bị rollback giữa chừng.

---

# Level 4 — Redis A vs. Redis B Outage Recovery (Khắc phục Sự cố Redis)

Hạ tầng FixTrack tách biệt hai cụm Redis chuyên biệt, quy trình xử lý khi có sự cố sập nguồn của từng cụm được thực hiện như sau:

## 4.1 Redis A (Cache & Throttler) - Cấp độ Sự cố: SEV-3

*   **Triệu chứng**: Giao diện Alertmanager báo mất kết nối tới `fixtrack-redis-cache`.
*   **Ảnh hưởng**: Người dùng cảm nhận hệ thống chậm hơn nhẹ do API phải truy vấn trực tiếp xuống DB (Cache Fallback). Ứng dụng vẫn hoạt động bình thường, không gây mất mát dữ liệu.
*   **Quy trình Khắc phục**:
    ```bash
    # 1. Kiểm tra trạng thái container Redis A
    docker compose ps redis-cache

    # 2. Khởi động lại container nếu bị dừng đột ngột
    docker compose restart redis-cache

    # 3. Theo dõi log xem có bị đầy bộ nhớ (OOM) hay không
    docker compose logs --tail=100 redis-cache
    ```

## 4.2 Redis B (BullMQ Queues & Sessions) - Cấp độ Sự cố: SEV-1

*   **Triệu chứng**: Cảnh báo lỗi nghiêm trọng từ Sentry, tiến trình API báo lỗi không thể đẩy Job gửi thông báo hoặc xử lý hàng chờ.
*   **Ảnh hưởng**: Hàng chờ BullMQ ngừng hoạt động hoàn toàn. Các sự kiện gửi thông báo (Notifications), trừ kho vật tư, phân công xe bị đóng băng. Người dùng không nhận được thông báo thời gian thực.
*   **Quy trình Khắc phục**:
    ```bash
    # 1. Kiểm tra container Redis B
    docker compose ps redis-queue

    # 2. Kiểm tra tính toàn vẹn của tệp ghi nhận AOF (Append-Only File)
    # Nếu Redis B sập do crash tệp ghi đĩa AOF, tiến hành sửa lỗi tệp AOF trước khi bật lại
    docker run --rm --volumes-from fixtrack-redis-queue redis:7-alpine redis-check-aof --fix /data/appendonly.aof

    # 3. Khởi động lại Redis B
    docker compose restart redis-queue

    # 4. Khởi động lại BullMQ Worker để tái thiết lập kết nối và xử lý tiếp các jobs tồn đọng
    docker compose restart worker-service
    ```

---

# Level 5 — Infrastructure Alert Responses (Xử lý Cảnh báo Hạ tầng)

## 5.1 Xử lý CPU / RAM của VPS Host Tăng vọt (> 85%)

```bash
# 1. Kiểm tra tiến trình hệ thống đang ngốn nhiều CPU/RAM nhất
top -b -n 1 | head -n 20

# 2. Kiểm tra container tiêu thụ tài nguyên lớn nhất
docker stats --no-stream

# 3. Nếu container api-service bị rò rỉ bộ nhớ (Memory Leak), thực hiện restart graceful
docker compose restart api-service
```

## 5.2 Xử lý Ổ đĩa VPS Báo sắp đầy (Disk Space < 15% trống)

```bash
# 1. Kiểm tra phân bổ dung lượng đĩa
df -h

# 2. Quét dọn các Docker layers, images rác không sử dụng (Dangling Images)
docker image prune -af

# 3. Quét dọn các Docker volumes mồ côi
docker volume prune -f

# 4. Thực hiện thu hẹp hoặc xoá bớt các tệp tin log Docker cũ (LƯU Ý: Đây là biện pháp khẩn cấp cuối cùng vì nó sẽ làm mất vĩnh viễn logs dùng cho phân tích lỗi forensic. Khuyến nghị cấu hình log-rotation trong daemon.json làm giải pháp chính thống).
truncate -s 0 /var/lib/docker/containers/*/*-json.log
```

## 5.3 Xử lý Hàng chờ BullMQ bị nghẽn (BullMQ Queue Congested)

Cảnh báo kích hoạt khi số lượng job chờ xử lý vượt quá 50 jobs trong 30 phút.

```bash
# 1. Kiểm tra log của worker xem tiến trình có bị nghẽn kết nối hoặc API bên thứ ba (như FCM, S3) bị chậm không
docker compose logs --tail=200 worker-service

# 2. Tăng số lượng luồng xử lý bằng cách tạm thời scale worker lên 2 instances (CHỈ thực hiện nếu VPS Host còn dư tài nguyên CPU/RAM. Nếu nguyên nhân nghẽn do database lock hoặc API bên thứ ba bị sập, việc scale worker sẽ làm trầm trọng thêm tình trạng nghẽn tải).
docker compose up -d --scale worker-service=2 --no-recreate worker-service

# 3. Sau khi số lượng queue giảm về ngưỡng an toàn, hạ scale về 1 instance để tiết kiệm tài nguyên
docker compose up -d --scale worker-service=1 worker-service
```

---

# Level 6 — Dual-Key Secret Rotation (Xoay vòng Khóa Bảo mật)

Quy trình xoay vòng các khóa nhạy cảm không gây gián đoạn dịch vụ hoặc buộc toàn bộ người dùng phải đăng nhập lại.

## 6.1 Xoay vòng JWT Secret (Zero-Downtime JWT Rotation)

Để xoay vòng JWT khóa bí mật mà không làm logout tất cả người dùng đang hoạt động, hệ thống NestJS được thiết kế để hỗ trợ danh sách các khóa chấp nhận song song.

```
Bước 1: Cấu hình mới (Deploy 1)
  - Khóa hoạt động để KÝ mới (Active Key): Key_V1
  - Khóa được CHẤP NHẬN để giải mã (Accept Keys): [Key_V1, Key_V2] (Key_V2 là khóa mới chuẩn bị)

Bước 2: Chuyển đổi vai trò (Deploy 2)
  - Khóa hoạt động để KÝ mới (Active Key): Key_V2
  - Khóa được CHẤP NHẬN để giải mã (Accept Keys): [Key_V1, Key_V2]
  - Giải thích: Toàn bộ JWT Token mới phát hành sẽ ký bằng Key_V2. 
    Các Token cũ ký bằng Key_V1 vẫn giải mã hợp lệ.

Bước 3: Thu hồi hoàn toàn (Deploy 3 - Sau 8 giờ/Vòng đời token hết hạn)
  - Khóa hoạt động để KÝ mới (Active Key): Key_V2
  - Khóa được CHẤP NHẬN để giải mã (Accept Keys): [Key_V2]
  - Giải thích: Key_V1 chính thức bị loại bỏ hoàn toàn khỏi hệ thống.
```

### Hướng dẫn Cấu hình trên `.env.production`:

```env
# Giai đoạn Bước 2:
JWT_SECRET=Key_V2_Mới_Của_Hệ_Thống
JWT_SECONDARY_SECRET=Key_V1_Cũ_Đang_Thu_Hồi
```

## 6.2 Xoay vòng AWS S3 / Cloudflare R2 Credentials

```env
# 1. Tạo một Access Key mới trên trang quản trị S3/R2 (đặt tên Key_B)
# 2. Cập nhật file .env.production cấu hình đồng thời khoá chính và khoá phụ (nếu app hỗ trợ fallback)
# Hoặc thay thế trực tiếp và khởi động lại API để nhận khoá mới:
S3_ACCESS_KEY_ID=Key_B_ID
S3_SECRET_ACCESS_KEY=Key_B_Secret

# 3. Khởi động lại container để áp dụng cấu hình mới
docker compose up -d --no-deps api-service worker-service

# 4. Khi xác thực app ghi ảnh lên S3 ổn định bằng Key_B, tiến hành xoá bỏ Key_A cũ trên trang quản trị S3/R2
```

---

# Level 7 — Procedure Ownership Matrix (Ma trận Trách nhiệm Vận hành)

Để phân định trách nhiệm rõ ràng khi hệ thống gặp sự cố cần leo thang xử lý:

| Quy trình vận hành | Người thực thi chính (Primary Owner) | Người giám sát / Hỗ trợ (Escalation) |
| :--- | :--- | :--- |
| **Dừng / Khởi động hệ thống** | DevOps Engineer | Tech Lead |
| **Khôi phục dữ liệu CSDL** | Senior Backend Engineer / DevOps | Tech Lead |
| **Triển khai di cư dữ liệu (Migration)** | DevOps / CI Pipeline Automation | Lead Developer |
| **Ứng phó sự cố SEV-1 / SEV-2** | On-call Engineer (DevOps/SRE) | Incident Commander / Tech Lead |
| **Xoay vòng khóa bí mật (Secrets)** | Security Administrator | DevOps Lead / Tech Lead |
