# 30 Infrastructure

**Trạng thái: Hoàn thành (Finalized - Version 1.0)**

---

## Purpose

Chương này đặc tả chi tiết hạ tầng vật lý và máy chủ ảo (Infrastructure Specification) của hệ thống FixTrack. Tài liệu định nghĩa chi tiết phân bổ tài nguyên phần cứng (Capacity & Sizing), cấu trúc mạng ảo nội bộ và tường lửa (Network & Firewall), cơ chế phân bổ ổ đĩa lưu trữ bền vững (Persistent Storage Volumes), kế hoạch sao lưu khôi phục dữ liệu phòng ngừa thảm họa (Disaster Recovery Plan), quy trình phát hành mã nguồn (Deployment Strategy), hệ thống giám sát tài nguyên (Monitoring & Alerting), và lộ trình mở rộng tính sẵn sàng cao (Scaling Roadmap).

---

## Inputs

*   [26 System Architecture](./26_system_architecture.md)
*   [28 Security Architecture](./28_security_architecture.md)
*   [29 Integration Architecture](./29_integration_architecture.md)

---

# Level 1 — Environment Topologies (Môi trường Hạ tầng)

FixTrack được vận hành đồng bộ trên 3 môi trường hạ tầng độc lập:

```
                  ┌──────────────────────────────────────────────┐
                  │          Cloudflare (Full Strict)            │
                  └──────────────────────┬───────────────────────┘
                                         │ HTTPS (443)
                                         ▼
┌────────────────────────────────────────────────────────────────────────────────┐
│ VPS Host System (Ubuntu Server - Single Node SPOF in Phase 1)                 │
│                                                                                │
│   ┌────────────────────────────────────────────────────────────────────────┐   │
│   │ Docker Daemon (Virtual Bridge Network: fixtrack-net)                    │   │
│   │                                                                        │   │
│   │   ┌───────────────┐     ┌───────────────┐      ┌──────────────────┐    │   │
│   │   │  NestJS API   ├────►│  Redis Cache  ├─────►│ PostgreSQL Master│    │   │
│   │   └───────┬───────┘     └───────────────┘      └────────┬─────────┘    │   │
│   │           │                                             │              │   │
│   │           ▼                                             ▼              │   │
│   │   ┌───────────────┐                             ┌───────────────┐      │   │
│   │   │ BullMQ Worker │                             │  WAL Archiver  │      │   │
│   │   └───────┬───────┘                             └───────┬───────┘      │   │
│   │           │                                             │              │   │
│   └───────────┼─────────────────────────────────────────────┼──────────────┘   │
│               │                                             │                  │
└───────────────┼─────────────────────────────────────────────┼──────────────────┘
                │ Presigned Upload                            │ WAL Log Stream
                ▼                                             ▼
┌────────────────────────────────────────────────────────────────────────────────┐
│ External Services (S3 / Cloudflare R2 Cloud)                                   │
│                                                                                │
│       [ S3 Attachments Bucket ]                      [ S3 Backups Bucket ]     │
└────────────────────────────────────────────────────────────────────────────────┘
```

1.  **Local Development (Môi trường phát triển nội bộ)**:
    *   *Triển khai*: Chạy cục bộ trên máy trạm cá nhân của lập trình viên thông qua Docker Desktop.
    *   *Mục tiêu*: Phục vụ viết code, chạy kiểm thử đơn vị (Unit Test). Chạy phiên bản rút gọn của Database và Redis không cấu hình HA.
2.  **Staging / UAT (Môi trường kiểm thử tích hợp & nghiệm thu)**:
    *   *Triển khai*: Chạy trên 1 máy chủ VPS riêng biệt (Single VPS) thông qua Docker Compose.
    *   *Mục tiêu*: Đồng bộ mã nguồn mới nhất từ nhánh `develop` để Tester kiểm thử tích hợp (Integration Test) và Người dùng nghiệm thu (UAT).
    *   *Bảo mật*: Giới hạn truy cập bằng lớp bảo vệ HTTP Basic Auth phía trước Gateway và chặn hoàn toàn truy cập trực tiếp từ Internet vào CSDL.
3.  **Production (Môi trường vận hành chính thức - Giai đoạn 1)**:
    *   *Triển khai*: Sử dụng 1 máy chủ Enterprise VPS cấu hình cao, chạy Docker Compose tối ưu hóa hiệu năng container (Phase 1).
    *   *Rủi ro vận hành*: Chấp nhận rủi ro lỗi máy chủ đơn lẻ (Single-node SPOF) cho giai đoạn đầu của dự án. Khả năng sẵn sàng cao (High Availability) được dời lại vào Phase 2.
    *   *Bảo mật*: Tên miền được bảo vệ bởi **Cloudflare Proxy** chạy chế độ **Full (Strict)**. Lưu ý: Cloudflare đóng vai trò làm DNS Proxy và TLS Edge bảo vệ cổng (chuyển tiếp traffic HTTPS an toàn về IP công cộng của VPS Host), không đóng vai trò làm Application Load Balancer nội bộ. VPS Host kích hoạt tường lửa UFW nghiêm ngặt chỉ mở cổng 80/443 (từ dải IP Cloudflare) và cổng SSH tùy biến.

---

# Level 2 — Capacity & Sizing (Định lượng Tài nguyên & Hiệu năng)

## 2.1 Giả định Tải thiết kế (Throughput Assumptions)

Thông số phần cứng được định cỡ dựa trên dự phóng quy mô sử dụng trong năm đầu tiên vận hành:
*   **Daily Active Users (DAU)**: ~200 người dùng hoạt động hàng ngày (Tài xế, kỹ thuật viên, quản lý kho).
*   **Peak Concurrency (Số kết nối đồng thời)**: ~50 người dùng thao tác cùng lúc tại thời điểm cao điểm (giao nhận ca xe, cập nhật phụ tùng).
*   **Peak Request Rate (Tần suất yêu cầu)**: ~20 requests/second (req/sec).
*   **Storage Growth (Mức tăng dung lượng)**: Ước tính ~5 GB dữ liệu tệp đính kèm (ảnh chụp báo hỏng, chứng từ) tăng thêm mỗi tháng.

## 2.2 Đặc tả Tài nguyên Phần cứng tối thiểu (Hardware Specifications)

| Phân lớp / Thành phần | Môi trường Dev | Môi trường Staging (UAT) | Môi trường Production (Phase 1) | Ghi chú & Giới hạn tài nguyên (Docker Limits) |
| :--- | :--- | :--- | :--- | :--- |
| **Cấu hình VPS Host** | Máy trạm cục bộ | **2 vCPUs** (Shared AMD/Intel)<br>**4 GB RAM**<br>40 GB SSD Storage | **4 vCPUs** (Dedicated CPU)<br>**16 GB RAM**<br>80 GB NVMe SSD | Khuyến nghị sử dụng các nhà cung cấp uy tín. Chọn mức 16GB RAM cho Production để tránh rủi ro OOM (Out Of Memory) khi cộng dồn các giới hạn bộ nhớ của CSDL, Redis, API và Worker. |
| **NestJS API Container** | Không giới hạn | Lên tới 1.0 vCPU<br>1 GB RAM | Lên tới 2.0 vCPUs<br>2 GB RAM | Thiết lập trong file compose: `cpus: 2.0`, `memory: 2G`. Ở Phase 1 MVP, chạy 1 instance độc lập ánh xạ trực tiếp ra cổng Host 80/443. Lên Phase 2 sẽ nâng lên nhiều instances kết hợp Load Balancer nội bộ. |
| **BullMQ Worker Container** | Không giới hạn | Lên tới 0.5 vCPU<br>512 MB RAM | Lên tới 1.0 vCPU<br>1 GB RAM | Xử lý tác vụ nền bất đồng bộ (Push notifications, quét virus, dọn tệp rác). |
| **PostgreSQL Container** | Không giới hạn | Lên tới 1.0 vCPU<br>1.5 GB RAM | Lên tới 2.0 vCPUs<br>3 GB RAM | Cấu hình `shared_buffers = 2GB`, `work_mem = 64MB` để tối ưu hóa bộ nhớ đệm cho ghi nhận giao dịch. |
| **Redis A (Cache Container)** | Không giới hạn | Lên tới 0.25 vCPU<br>512 MB RAM | Lên tới 0.5 vCPU<br>1 GB RAM | Cấu hình cờ giới hạn bộ nhớ: `maxmemory 1gb`, cơ chế thu hồi khóa tự động `maxmemory-policy allkeys-lru`. |
| **Redis B (Queue/Sec Container)** | Không giới hạn | Lên tới 0.25 vCPU<br>512 MB RAM | Lên tới 0.5 vCPU<br>1 GB RAM | Cấu hình bộ nhớ `maxmemory 1gb` nhưng cấm tự động xóa khi đầy: `maxmemory-policy noeviction`. |
| **Object Storage (S3/R2)** | MinIO Local | 10 GB Storage | 100 GB NVMe Storage | Sử dụng AWS S3 hoặc Cloudflare R2 để tối ưu hóa chi phí. Tốc độ tăng trưởng dự kiến: ~5GB/tháng. |

---

# Level 3 — Network & Firewall (Mạng & Tường lửa)

Để bảo vệ các dịch vụ nhạy cảm khỏi bị quét cổng và tấn công từ bên ngoài Internet, FixTrack chia hạ tầng mạng làm hai phân vùng logic.

## 3.1 Ma trận Cổng Dịch vụ (Service Port Matrix)

| Tên Dịch vụ (Service) | Cổng Nội bộ (Internal Port) | Cổng Vật lý (Host Port) | Truy cập Công cộng (Public Access) | Phương thức bảo vệ |
| :--- | :---: | :---: | :---: | :--- |
| **NestJS API Gateway** | 3000 | 80 / 443 | **Yes** | Chỉ nhận request được chuyển tiếp từ IP dải của Cloudflare Proxy. |
| **PostgreSQL Database** | 5432 | None | **No** | Chỉ cho phép truy cập nội bộ từ mạng ảo Docker `fixtrack-net`. |
| **Redis A (Cache)** | 6379 | None | **No** | Chỉ cho phép truy cập nội bộ từ mạng ảo Docker `fixtrack-net`. |
| **Redis B (Queue/Sec)** | 6379 | None | **No** | Chỉ cho phép truy cập nội bộ từ mạng ảo Docker `fixtrack-net`. |
| **ClamAV Engine** | 3310 | None | **No** | Chỉ nhận request socket nội bộ để quét virus. |
| **Bull Board Dashboard** | 3000 | None | **No** | Là trang dashboard được tích hợp trực tiếp bên trong NestJS API (đường dẫn `/api/v1/admin/queues`), bảo vệ bằng Admin Auth và IP restriction. |

## 3.2 Quy tắc Tường lửa UFW (Firewall Rules)

Tường lửa mềm UFW (Uncomplicated Firewall) trên VPS Host chạy hệ điều hành Ubuntu được cấu hình nghiêm ngặt theo các tập lệnh dưới đây:

```bash
# Reset về trạng thái mặc định: Chặn toàn bộ chiều vào, cho phép chiều ra
ufw default deny incoming
ufw default allow outgoing

# Cho phép SSH truy cập từ xa (Đổi sang cổng tùy chọn và chỉ định IP admin cố định nếu có)
ufw allow from 115.79.x.x to any port 22022 proto tcp comment 'Allow Admin SSH'

# Chỉ cho phép các kết nối Web công cộng đi qua cổng 80 và 443 từ dải IP Cloudflare
for ip in $(curl -s https://www.cloudflare.com/ips-v4); do ufw allow from $ip to any port 80 proto tcp; done
for ip in $(curl -s https://www.cloudflare.com/ips-v4); do ufw allow from $ip to any port 443 proto tcp; done

# Nghiêm cấm hoàn toàn truy cập trực tiếp từ ngoài vào các cổng dịch vụ nội bộ
ufw deny 5432/tcp comment 'Block Postgres Public Access'
ufw deny 6379/tcp comment 'Block Redis Public Access'
ufw deny 3000/tcp comment 'Block NestJS Internal Port'

# Kích hoạt tường lửa
ufw enable
```

## 3.3 Quản lý Mã khóa và Cấu hình (Secrets Management)

*   **Nguyên tắc tuyệt đối**: Cấm hardcode mọi thông tin nhạy cảm (CSDL, JWT Secret, Firebase API Key) vào mã nguồn Git và không được nướng mã khóa (Never bake secrets) vào trong ảnh của Container Image.
*   **Môi trường Dev**: Sử dụng tệp `.env` cục bộ (đã khai báo trong `.gitignore`).
*   **Môi trường Production (Phase 1)**: Cấu hình biến môi trường bằng tệp `.env.production` đặt trên ổ cứng máy chủ VPS Host, được Docker Compose nạp trực tiếp vào container tại thời điểm chạy (`runtime`). File này được phân quyền đọc độc quyền cho root (`chmod 600 .env.production`).
*   **Lộ trình tương lai**: Dịch chuyển sang hệ thống quản lý khóa tập trung có mã hóa nâng cao như **HashiCorp Vault** hoặc **AWS Secrets Manager** kết hợp mã hóa trực tiếp bằng Cloud KMS.

---

# Level 4 — Storage & Persistence (Lưu trữ & Dữ liệu Bền vững)

Để đảm bảo dữ liệu không bị mất mát khi các container bị tắt hoặc khởi động lại, hệ thống thiết lập các thư mục lưu trữ bền vững (Docker Volumes) gắn trực tiếp với ổ cứng của máy chủ VPS Host.

```yaml
# Docker Compose Volume Mapping Specification
volumes:
  # 1. Lưu trữ dữ liệu PostgreSQL vật lý
  postgres_data:
    driver: local
    driver_opts:
      type: 'none'
      o: 'bind'
      device: '/var/lib/fixtrack/postgres_data'

  # 2. Lưu trữ tệp tin nhật ký WAL phục vụ khôi phục dữ liệu
  postgres_wal:
    driver: local
    driver_opts:
      type: 'none'
      o: 'bind'
      device: '/var/lib/fixtrack/postgres_wal'

  # 3. Lưu dữ liệu hàng chờ và session Redis B
  redis_b_data:
    driver: local
    driver_opts:
      type: 'none'
      o: 'bind'
      device: '/var/lib/fixtrack/redis_b_data'
```

## 4.1 Lưu trữ PostgreSQL Volume
*   Dữ liệu của CSDL PostgreSQL được gắn từ container `/var/lib/postgresql/data` vào thư mục vật lý `/var/lib/fixtrack/postgres_data` trên VPS Host. Thư mục này được định cấu hình loại trừ hoàn toàn khỏi quyền truy cập của các user thường để đảm bảo an toàn.

## 4.2 Chiến lược Ghi dữ liệu bền vững của Redis (Redis Persistence Policy)

FixTrack chạy hai cụm Redis phục vụ hai mục đích khác nhau, do đó cấu hình ghi xuống đĩa cũng được tùy biến tối ưu:

1.  **Cụm Redis A (Cache & Throttler)**:
    *   *Chính sách*: **RDB (Redis Database) Snapshotting** làm dự phòng.
    *   *Chi tiết*: Định kỳ chụp ảnh snapshot dữ liệu bộ nhớ lưu xuống tệp tin `dump.rdb` mỗi 15 phút một lần nếu có ít nhất 1 thay đổi. Trong trường hợp sập nguồn đột ngột, việc mất một vài phút dữ liệu cache không ảnh hưởng tới tính đúng đắn của hệ thống.
2.  **Cụm Redis B (BullMQ Queues, Sessions, Blacklist Tokens)**:
    *   *Chính sách*: **AOF (Append Only File) kết hợp RDB** để đảm bảo an toàn dữ liệu tuyệt đối.
    *   *Chi tiết*: Mọi thao tác thay đổi trạng thái hàng chờ và phiên làm việc được ghi nhận tuần tự vào tệp tin nhật ký AOF.
    *   *Cấu hình khuyến nghị trong tệp `redis.conf`*:
        ```properties
        appendonly yes
        appendfsync everysec # Ghi đồng bộ dữ liệu từ bộ đệm OS xuống ổ đĩa vật lý mỗi giây một lần
        no-appendfsync-on-rewrite yes
        auto-aof-rewrite-percentage 100
        auto-aof-rewrite-min-size 64mb
        ```

---

# Level 5 — Backup & Disaster Recovery (Sao lưu & Khôi phục Thảm họa)

Hệ thống FixTrack thiết lập quy trình sao lưu tự động bất đồng bộ để đáp ứng các chỉ tiêu phi chức năng cốt lõi:

*   **RPO (Recovery Point Objective)**: **< 15 phút** (Thời gian mất mát dữ liệu tối đa).
*   **RTO (Recovery Time Objective)**: **< 2 giờ** (Thời gian khôi phục hệ thống tối đa).

```
[ PostgreSQL Master ]
         │
         ├── (Mỗi 15 phút) ──► [ WAL Archiving ] ─────────► [ S3 Backups Bucket ]
         │                                                         ▲
         └── (Mỗi 24 giờ)  ──► [ pg_dump (Compressed) ] ───────────┘
```

## 5.1 Kịch bản Sao lưu Dữ liệu (Backup Plan)

1.  **Sao lưu CSDL Toàn phần (Full Database Backup)**:
    *   Một tiến trình Cron Job chạy trên VPS Host vào lúc **02:00 sáng hàng ngày** để xuất bản sao lưu logic nén của CSDL PostgreSQL dành riêng cho ứng dụng:
        ```bash
        docker exec -t fixtrack-postgres pg_dump -U app_db_user -d fixtrack_prod | gzip > /var/lib/fixtrack/backups/db_full_$(date +%F).sql.gz
        ```
    *   Tệp tin sau khi nén được mã hóa bằng thuật toán **AES-256** trước khi sử dụng S3 CLI đẩy lên `S3 Backups Bucket` nằm ngoài hạ tầng VPS Host.
2.  **Sao lưu Điểm Thời gian (Point-in-Time Recovery - PITR thông qua WAL Archiving)**:
    *   CSDL PostgreSQL được cấu hình ở chế độ `archive_mode = on`.
    *   Mỗi khi tệp tin ghi trước WAL (Write-Ahead Log) đạt kích thước 16MB hoặc sau mỗi 15 phút, tiến trình WAL Archiver sẽ tự động đẩy tệp WAL này lên vùng lưu trữ S3 bảo mật. 

## 5.2 Chính sách Lưu trữ bản sao lưu (Retention Policy)

*   **Bản sao lưu hàng ngày (Daily Backups)**: Lưu giữ trong **30 ngày** gần nhất, tự động xóa sau 30 ngày.
*   **Bản sao lưu hàng tuần (Weekly Backups)**: Lưu giữ trong **12 tuần** gần nhất.
*   **Bản sao lưu hàng tháng (Monthly Backups)**: Lưu giữ trong **12 tháng** gần nhất.

## 5.3 Quy trình Xác minh Khôi phục (Backup Verification & Restore Drill)

Bản sao lưu vô giá trị cho đến khi nó được xác thực có khả năng khôi phục thành công:
*   **Weekly Automated Restore Test (Kiểm thử khôi phục tự động hàng tuần)**:
    Hệ thống triển khai một container tạm vào mỗi tối Chủ nhật. Container này sẽ kéo bản sao lưu mới nhất từ S3 về, tiến hành giải nén và khôi phục vào một cơ sở dữ liệu nháp (Scratch DB) để kiểm tra tính toàn vẹn cấu trúc và dữ liệu. Nếu việc khôi phục gặp lỗi, lập tức gửi cảnh báo `CRITICAL` đến hệ thống ChatOps.
*   **Monthly DR Drill (Diễn tập khôi phục hàng tháng)**:
    Đội ngũ DevOps thực hiện diễn tập khôi phục thủ công hệ thống trên một VPS trắng để kiểm thử tốc độ đáp ứng RTO của quy trình vận hành và kiểm tra tài liệu hướng dẫn khôi phục thảm họa (Disaster Recovery Runbook).

---

# Level 6 — Deployment & Release Strategy (Chiến lược Triển khai & Phát hành)

FixTrack triển khai chiến lược phát hành tự động qua CI/CD và quy trình cập nhật không gián đoạn dịch vụ trên máy chủ VPS Host.

## 6.1 Quy trình Phát hành Mã nguồn (Deployment Flow)

```
[ Developer Commit ] ──► [ GitHub Action CI ] ──► [ Build & Push Docker Image ]
                                                         │
                                                         ▼
[ Docker Container Run ] ◄── [ Pull Image & Compose Up ] ◄── [ SSH Deploy Trigger ]
```

1.  **CI Build**: GitHub Actions chạy kiểm thử tự động, build Docker Image cho ứng dụng NestJS API và BullMQ Worker, sau đó push image kèm tag định danh phiên bản (`fixtrack-api:v1.0.3`) lên Private Container Registry.
2.  **Deployment Trigger**: GitHub Actions SSH kết nối an toàn vào VPS Host, tạo cấu hình môi trường mới và chạy script cập nhật.
3.  **Graceful Rollout (Quy trình cập nhật container)**:
    *   **Phase 1 (MVP - Single Instance)**: Do chạy trực tiếp 1 instance API map cứng cổng host (`80/443`) nên việc cập nhật container sẽ gây ra **thời gian downtime ngắn (dưới 5 giây)** khi chạy lệnh:
        ```bash
        docker compose pull api-service
        docker compose up -d --no-deps api-service
        ```
        Sự đánh đổi này được chấp nhận ở giai đoạn khởi điểm để đơn giản hóa hạ tầng.
    *   **Phase 2 (Load Balanced - Zero Downtime)**: Tích hợp thêm một container Reverse Proxy (Caddy hoặc Traefik) làm gateway điều hướng. Khi đó, lệnh cập nhật sẽ được scale tạm thời lên 2 instances để cập nhật cuốn chiếu (rolling restart), đạt trạng thái không downtime:
        ```bash
        docker compose up -d --no-deps --scale api-service=2 --no-recreate api-service
        # (Chờ instance mới healthcheck pass, tắt instance cũ và đưa scale về 1 nếu cần)
        ```

## 6.2 Cập nhật & Kiểm tra Sức khỏe (Health Checks & Rollback)

*   **Health Check Endpoint**: Cấu hình thuộc tính kiểm tra trạng thái sức khỏe trong file `docker-compose.yml`:
    ```yaml
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/api/v1/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 20s
    ```
*   **Chiến lược Khôi phục nhanh (Rollback)**: Nếu container mới khởi động không vượt qua được Health Check sau 3 lần thử, hệ thống CI/CD sẽ tự động chạy lệnh rollback về tag image ổn định trước đó (ví dụ: `v1.0.2`) để khôi phục dịch vụ lập tức, đảm bảo tính sẵn sàng.

---

# Level 7 — Infrastructure Monitoring & Alerting (Giám sát & Cảnh báo)

Kiến trúc quan trắc hạ tầng của FixTrack được thiết lập để theo dõi toàn bộ trạng thái tài nguyên phần cứng vật lý và ảo hóa:

```
[ Prometheus Server ]
         ├── (Cào metrics) ──► [ Node Exporter (Host CPU/RAM/Disk) ]
         ├── (Cào metrics) ──► [ cAdvisor (Container Resource Usage) ]
         ├── (Cào metrics) ──► [ Postgres & Redis Exporter ]
         └── (Cảnh báo)     ──► [ Alertmanager ] ──► [ Slack Security Channel ]
```

## 7.1 Thu thập Metrics Hạ tầng (Metrics Collection)

*   **Node Exporter**: Thu thập thông tin trực tiếp từ VPS Host (CPU usage, RAM allocated, Disk IOPS, Network throughput).
*   **cAdvisor**: Thu thập thông tin sử dụng tài nguyên của từng container cụ thể (đo lường rò rỉ RAM hoặc vòng lặp khởi động lại container - container restart loop).
*   **Database & Cache Exporters**: Theo dõi số lượng kết nối đang hoạt động (Active Connections) tới PostgreSQL, dung lượng lưu trữ thực tế của Redis.

## 7.2 Ngưỡng kích hoạt Cảnh báo (Alerting Thresholds)

Alertmanager được cấu hình để gửi thông báo khẩn cấp tới kênh Slack kỹ thuật khi đạt tới các ngưỡng rủi ro:

*   **CPU / RAM Usage**: Vượt quá **85%** liên tục trong vòng 5 phút (Cảnh báo mức `HIGH`).
*   **Disk Free Space**: Dung lượng trống của ổ đĩa dưới **15%** (Cảnh báo mức `CRITICAL` - cần dọn dẹp hoặc mở rộng ổ đĩa ngay).
*   **Container Restart**: Bất kỳ container nào tự động khởi động lại nhiều hơn **3 lần / 10 phút** (Cảnh báo mức `HIGH` - lỗi sập nguồn bộ nhớ).
*   **Database Connections**: Số lượng kết nối vượt quá **80%** giới hạn cho phép (Cảnh báo mức `HIGH` - nguy cơ nghẽn DB).

---

# Level 8 — Future HA / Scaling Roadmap (Lộ trình Nâng cấp Hạ tầng)

Để sẵn sàng đáp ứng khi quy mô người dùng tăng trưởng vượt mức giả định ban đầu (vượt quá 1,000 người dùng đồng thời), lộ trình nâng cấp hạ tầng được thiết lập theo các giai đoạn rõ ràng:

```mermaid
chronology
    title Lộ trình mở rộng hạ tầng FixTrack
    Phase 1 (Single Node) : Triển khai 1 Database PostgreSQL Master duy nhất cho cả Đọc và Ghi
    Phase 2 (HA Database & Cache) : Triển khai Postgres Master-Replica có Patroni + Consul để tự động Failover. Nâng cấp cụm Redis Sentinel.
    Phase 3 (K8s Application Cluster) : Chuyển dịch ứng dụng sang Kubernetes Cluster (AWS EKS hoặc tự dựng). Tích hợp Cloud Load Balancer (ALB).
```

*   **Giai đoạn Phase 2 (HA Database & Caching)**:
    *   *CSDL*: Tách biệt cơ sở dữ liệu thành cụm Master-Replica. Cấu hình **Patroni** kết hợp **Consul** để tự động thăng chức Replica lên làm Master trong vòng dưới 30 giây khi Master chính gặp sự cố phần cứng.
    *   *Cache*: Sử dụng mô hình **Redis Sentinel** hoặc Redis Cluster để đảm bảo bộ nhớ đệm và dữ liệu hàng chờ BullMQ được đồng bộ dự phòng liên tục.
*   **Giai đoạn Phase 3 (Kubernetes Cluster)**:
    *   Chuyển toàn bộ các container NestJS API và BullMQ Worker sang quản lý bằng **Kubernetes (K8s)**.
    *   Tự động mở rộng số lượng Pod (Horizontal Pod Autoscaler - HPA) dựa trên thông số tải CPU thực tế.
    *   Tích hợp dịch vụ cân bằng tải vật lý đám mây (Cloud Load Balancer) để phân phối lưu lượng truy cập ổn định.
