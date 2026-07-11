# 31 Deployment

**Trạng thái: Hoàn thành (Finalized - Version 1.0)**

---

## Purpose

Chương này đặc tả chi tiết cấu hình và quy trình triển khai (Deployment Specification) của hệ thống FixTrack. Tài liệu định nghĩa chiến lược container hóa, mã nguồn Dockerfile multi-stage, tệp cấu hình chạy Production `docker-compose.yml`, sơ đồ phân bổ biến môi trường theo nhóm, chiến lược di cư cơ sở dữ liệu (Database Migration), quy trình tích hợp và phát hành tự động (CI/CD Pipeline), quản lý nhật ký vận hành (Runtime Logging), và các kịch bản shell script hỗ trợ triển khai/khôi phục lỗi (Deployment & Rollback Scripts).

---

## Inputs

*   [22 Backend Modules](../part_e_backend_design/22_backend_modules.md)
*   [24 Background Jobs](../part_e_backend_design/24_background_jobs.md)
*   [26 System Architecture](../part_f_solution_architecture/26_system_architecture.md)
*   [28 Security Architecture](../part_f_solution_architecture/28_security_architecture.md)
*   [30 Infrastructure](./30_infrastructure.md)

---

# Level 1 — Containerization Strategy (Chiến lược Container hóa)

Hệ thống FixTrack đóng gói toàn bộ mã nguồn ứng dụng và các dịch vụ phụ trợ dưới dạng các Docker Container để đảm bảo tính đồng nhất giữa môi trường phát triển (Local) và vận hành chính thức (Staging/Production).

## 1.1 Quy tắc Đặt thẻ Hình ảnh (Image Tagging Rules)

*   **Môi trường Dev/Staging**: Sử dụng tag ngắn theo tên nhánh Git hoặc mã hash commit (ví dụ: `fixtrack-api:dev`, `fixtrack-api:sha-a1b2c3d`).
*   **Môi trường Production**: 
    *   **Nghiêm cấm** sử dụng tag `:latest` trên môi trường Production để tránh việc tự động cập nhật ngoài kiểm soát khi container khởi động lại.
    *   Mọi image deploy bắt buộc phải gắn thẻ theo định dạng **Semantic Versioning kết hợp Git Hash ngắn**:
        $$\text{Tag} = \text{Version} + \text{"-"} + \text{GitCommitHash}$$
        *Ví dụ: `fixtrack-api:v1.0.3-a1b2c3d`*

## 1.2 Chiến lược "Một Image, Hai Vai trò" (One-Image-Two-Roles Pattern)

Để đơn giản hóa quy trình build và đảm bảo tính nhất quán của mã nguồn, cả hai dịch vụ **NestJS API Gateway** và **BullMQ Worker** đều được build chung ra một Docker Image duy nhất (`fixtrack-app`). Việc phân chia vai trò khi chạy được thực thi bằng cách ghi đè lệnh khởi chạy (`CMD`) trong file Docker Compose:

*   **NestJS API Instance**: Khởi chạy API server lắng nghe HTTP requests.
    ```bash
    CMD ["node", "dist/main.js"]
    ```
*   **BullMQ Worker Instance**: Khởi chạy tiến trình xử lý hàng chờ chạy nền.
    ```bash
    CMD ["node", "dist/worker.js"]
    ```

---

# Level 2 — Multi-stage Dockerfile Specification (Đặc tả Dockerfile)

Dockerfile của dự án FixTrack áp dụng mẫu thiết kế **Multi-stage Build** gồm 3 giai đoạn để tối ưu hóa dung lượng ảnh cuối và loại bỏ hoàn toàn các thư viện phát triển (Development Dependencies) khỏi môi trường chạy chính thức nhằm giảm thiểu bề mặt tấn công an ninh (CVEs).

```dockerfile
# ==============================================================================
# STAGE 1: Build & Compile (Builder)
# ==============================================================================
FROM node:20-alpine AS builder

WORKDIR /usr/src/app

# Cài đặt các công cụ build hệ thống nếu cần (như python, make, g++)
RUN apk add --no-cache python3 make g++

COPY package*.json ./
COPY tsconfig*.json ./

# Cài đặt toàn bộ dependencies bao gồm cả devDependencies
RUN npm ci

COPY src/ ./src

# Biên dịch mã nguồn sang Javascript (nằm trong thư mục dist/)
RUN npm run build

# ==============================================================================
# STAGE 2: Prune Dependencies (Production Deps)
# ==============================================================================
FROM node:20-alpine AS dist-clean

WORKDIR /usr/src/app

COPY package*.json ./

# Chỉ cài đặt các gói production (dependencies), loại bỏ devDependencies
RUN npm ci --omit=dev && npm cache clean --force

# ==============================================================================
# STAGE 3: Runtime Runner (Final Minimal Image)
# ==============================================================================
FROM node:20-alpine AS runner

WORKDIR /usr/src/app

ENV NODE_ENV=production

# Thiết lập múi giờ Việt Nam
RUN apk add --no-cache tzdata && \
    cp /usr/share/zoneinfo/Asia/Ho_Chi_Minh /etc/localtime && \
    echo "Asia/Ho_Chi_Minh" > /etc/timezone

# Copy kết quả biên dịch từ stage builder
COPY --from=builder /usr/src/app/dist ./dist
# Copy gói node_modules đã lược bỏ devDependencies từ stage dist-clean
COPY --from=dist-clean /usr/src/app/node_modules ./node_modules
COPY package*.json ./

# Tăng cường bảo mật: Chạy container bằng user thường 'node' (non-root)
RUN chown -R node:node /usr/src/app
USER node

EXPOSE 3000

# Lệnh khởi chạy mặc định (sẽ được ghi đè ở docker-compose đối với Worker)
CMD ["node", "dist/main.js"]
```

> [!IMPORTANT]
> **Quy tắc Kiểm tra An ninh Ảnh chứa (Image Security Scan):**
> Trong quy trình build tự động, ảnh chứa sau khi đóng gói bắt buộc phải được quét bằng công cụ **Trivy** hoặc **Grype**:
> `trivy image --severity HIGH,CRITICAL --exit-code 1 fixtrack-app:v1.0.3`
> Nếu phát hiện lỗi bảo mật ở mức `HIGH` hoặc `CRITICAL`, tiến trình CI sẽ tự động dừng hoạt động và hủy bỏ lệnh push lên registry.

---

# Level 3 — Production docker-compose.yml Template (Cấu hình Compose chính thức)

Dưới đây là tệp cấu hình [docker-compose.yml](file:///c:/Users/Legion/Desktop/IT/FixTrack/docker-compose.yml) hoàn chỉnh chạy trên máy chủ Production. Cấu hình này áp đặt các giới hạn về tài nguyên, cấu hình tự động xoay vòng log, cô lập mạng nội bộ, và chỉ định thứ tự khởi động dựa trên sức khỏe dịch vụ.

```yaml
networks:
  fixtrack-net:
    driver: bridge

volumes:
  postgres_data:
    external: true
  postgres_wal:
    external: true
  redis_b_data:
    external: true

services:
  # ============================================================================
  # 1. DATABASE SERVICE (PostgreSQL)
  # ============================================================================
  postgres-db:
    image: postgres:16-alpine
    container_name: fixtrack-postgres
    restart: unless-stopped
    networks:
      - fixtrack-net
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: fixtrack_prod
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - postgres_wal:/var/lib/postgresql/pg_wal
    # Áp đặt giới hạn phần cứng trực tiếp (Hỗ trợ tốt từ Docker Compose V2)
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 3G
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d fixtrack_prod"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 15s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"

  # ============================================================================
  # 2. CACHE SERVICE (Redis A)
  # ============================================================================
  redis-cache:
    image: redis:7-alpine
    container_name: fixtrack-redis-cache
    restart: unless-stopped
    networks:
      - fixtrack-net
    command: redis-server --maxmemory 1gb --maxmemory-policy allkeys-lru
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 1G
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"

  # ============================================================================
  # 3. QUEUE & SESSION SERVICE (Redis B)
  # ============================================================================
  redis-queue:
    image: redis:7-alpine
    container_name: fixtrack-redis-queue
    restart: unless-stopped
    networks:
      - fixtrack-net
    command: redis-server --appendonly yes --appendfsync everysec --maxmemory 1gb --maxmemory-policy noeviction
    volumes:
      - redis_b_data:/data
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 1G
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"

  # ============================================================================
  # 4. CLAMAV VIRUS SCAN ENGINE
  # ============================================================================
  clamav-scanner:
    image: clamav/clamav:latest
    container_name: fixtrack-clamav
    restart: unless-stopped
    networks:
      - fixtrack-net
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1.5G
    healthcheck:
      test: ["CMD-SHELL", "echo PING | nc -w 2 localhost 3310 | grep -q PONG"]
      interval: 20s
      timeout: 10s
      retries: 3
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # ============================================================================
  # 5. NESTJS API APPLICATION
  # ============================================================================
  api-service:
    image: ${APP_IMAGE_REGISTRY}/fixtrack-app:${APP_IMAGE_TAG}
    container_name: fixtrack-api
    restart: unless-stopped
    ports:
      - "80:3000" # Map HTTP cổng host vào container (Phase 1 Single VPS)
    networks:
      - fixtrack-net
    env_file:
      - .env.production
    depends_on:
      postgres-db:
        condition: service_healthy
      redis-cache:
        condition: service_healthy
      redis-queue:
        condition: service_healthy
      clamav-scanner:
        condition: service_healthy
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/api/v1/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 20s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"

  # ============================================================================
  # 6. BULLMQ BACKGROUND WORKER
  # ============================================================================
  worker-service:
    image: ${APP_IMAGE_REGISTRY}/fixtrack-app:${APP_IMAGE_TAG}
    container_name: fixtrack-worker
    restart: unless-stopped
    networks:
      - fixtrack-net
    entrypoint: ["node", "dist/worker.js"] # Ghi đè tệp thực thi chạy nền
    env_file:
      - .env.production
    depends_on:
      postgres-db:
        condition: service_healthy
      redis-queue:
        condition: service_healthy
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
    healthcheck:
      test: ["CMD-SHELL", "pgrep -f dist/worker.js || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 15s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"
```

> [!NOTE]
> **Lưu ý về Cấu hình Giới hạn Tài nguyên (Docker Resource Caveat):**
> Trong file Docker Compose trên, thẻ `deploy.resources.limits` được sử dụng để giới hạn CPU và RAM.
> * Với Docker Compose V2 trở lên (chạy bằng lệnh `docker compose`), cơ chế này được áp dụng trực tiếp ở chế độ chạy độc lập (non-swarm mode) mà không cần thêm tham số.
> * Nếu sử dụng phiên bản Docker Compose V1 cũ (lệnh `docker-compose` có gạch nối), cấu hình này sẽ bị bỏ qua trừ khi chạy kèm cờ `--compatibility`. Do đó, khuyến nghị bắt buộc nâng cấp Docker Engine trên VPS Host lên tối thiểu phiên bản 24.0.0 (đi kèm Compose V2).

---

# Level 4 — Environment Variables (Quản lý Biến Môi trường)

Để phân nhóm trực quan và tránh rò rỉ thông tin nhạy cảm, toàn bộ biến môi trường lưu trong `.env.production` được quy hoạch thành 5 nhóm logic:

## 4.1 Bảng Phân loại Biến Môi trường

| Nhóm biến (Group) | Tên Biến Môi trường (Key) | Giá trị Mẫu (Example Value) | Mô tả Vai trò |
| :--- | :--- | :--- | :--- |
| **Group A: Application** | `NODE_ENV`<br>`PORT`<br>`APP_VERSION`<br>`API_PREFIX` | `production`<br>`3000`<br>`1.0.3`<br>`/api/v1` | Cấu hình cổng chạy NestJS, phân lớp logic và phiên bản ứng dụng chạy thực tế. |
| **Group B: Database** | `DATABASE_URL`<br>`POSTGRES_USER`<br>`POSTGRES_PASSWORD` | `postgresql://app_user:pass@postgres-db:5432/fixtrack_prod`<br>`app_db_user`<br>`Mật_Khẩu_Mã_Hóa_Siêu_Mạnh` | Thông tin kết nối CSDL PostgreSQL trong mạng Docker nội bộ. |
| **Group C: Redis** | `REDIS_CACHE_URL`<br>`REDIS_QUEUE_URL` | `redis://redis-cache:6379`<br>`redis://redis-queue:6379` | Đường dẫn kết nối tới hai cụm Redis A (Cache) và Redis B (Queue/Session) riêng biệt. |
| **Group D: Security** | `JWT_SECRET`<br>`JWT_REFRESH_SECRET`<br>`ARGON2_MEMORY_COST`<br>`WEBHOOK_SHARED_SECRET` | `Khóa_Bí_Mật_JWT_90_Ngày`<br>`Khóa_Bí_Mật_Refresh_7_Ngày`<br>`65536`<br>`Khóa_Băm_Webhook_HMAC` | Chứa khóa ký số Token và các thông số băm mật khẩu bảo mật (Argon2id). |
| **Group E: External Service** | `S3_ENDPOINT`<br>`S3_ACCESS_KEY_ID`<br>`S3_SECRET_ACCESS_KEY`<br>`FCM_SERVER_KEY` | `https://s3.ap-southeast-1.amazonaws.com`<br>`AKIAIOSFODNN7EXAMPLE`<br>`wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY`<br>`AIzaSyA1B2C3...` | Mã định danh kết nối đám mây S3/R2 và Firebase Cloud Messaging Gateway. |

---

# Level 5 — Deployment & Rollback Scripts (Kịch bản Triển khai & Khôi phục)

Các tệp lệnh shell (Bash script) dưới đây được cài đặt trên VPS Host để tự động hóa và đảm bảo an toàn cho quá trình cập nhật phiên bản.

## 5.1 Kịch bản Shell Triển khai tự động (`deploy.sh`)

Mã nguồn của file [deploy.sh](file:///c:/Users/Legion/Desktop/IT/FixTrack/deploy.sh) chạy trên máy chủ:

```bash
#!/bin/bash
set -e

# Khai báo cấu hình
export APP_IMAGE_REGISTRY="<private-registry>"
export APP_IMAGE_TAG=$1

if [ -z "$APP_IMAGE_TAG" ]; then
    echo "ERROR: Vui lòng cung cấp tag phiên bản muốn deploy (ví dụ: ./deploy.sh v1.0.3-sha123)"
    exit 1
fi

echo "========================================="
echo "BẮT ĐẦU QUY TRÌNH DEPLOY PHIÊN BẢN: $APP_IMAGE_TAG"
echo "========================================="

# 1. Đăng nhập vào Registry (nếu dùng private registry)
# docker login -u <username> -p <password> $APP_IMAGE_REGISTRY

echo "-> Đang tải image mới về..."
docker compose pull api-service worker-service

# 2. Tạo bản sao lưu CSDL khẩn cấp trước khi Migration (Pre-deploy Backup Hook)
echo "-> Thực hiện sao lưu dữ liệu trước khi di cư cấu trúc..."
mkdir -p /var/lib/fixtrack/backups
docker exec -t fixtrack-postgres pg_dump -U app_db_user -d fixtrack_prod | gzip > /var/lib/fixtrack/backups/pre_deploy_backup.sql.gz

# 3. Kích hoạt chạy Migrations trước khi cập nhật Container chính
echo "-> Kích hoạt chạy Database Migration..."
./migrate.sh

# 4. Cập nhật Container
echo "-> Khởi động lại các container ứng dụng..."
docker compose up -d --no-deps api-service worker-service

# 5. Kiểm tra Health Check của cả API và Worker
echo "-> Đang kiểm tra trạng thái sức khỏe API & Worker..."
sleep 15
API_HEALTH=$(docker inspect --format='{{.State.Health.Status}}' fixtrack-api)
WORKER_HEALTH=$(docker inspect --format='{{.State.Health.Status}}' fixtrack-worker)

if [ "$API_HEALTH" == "healthy" ] && [ "$WORKER_HEALTH" == "healthy" ]; then
    echo "SUCCESS: Deploy thành công phiên bản $APP_IMAGE_TAG!"
    # Sao lưu tag phiên bản ổn định
    echo "$APP_IMAGE_TAG" > .env.stable_version
    # Gửi notify Slack thành công
else
    echo "CRITICAL ERROR: Một hoặc nhiều dịch vụ gặp sự cố!"
    echo "- API Status: $API_HEALTH"
    echo "- Worker Status: $WORKER_HEALTH"
    echo "Kích hoạt tự động Rollback..."
    ./rollback.sh
    exit 1
fi
```

## 5.2 Kịch bản Shell Khôi phục nhanh (`rollback.sh`)

Mã nguồn của file [rollback.sh](file:///c:/Users/Legion/Desktop/IT/FixTrack/rollback.sh) chạy khi gặp sự cố:

```bash
#!/bin/bash
echo "========================================="
echo "BẮT ĐẦU QUY TRÌNH ROLLBACK KHẨN CẤP"
echo "========================================="

# 1. Lấy tag phiên bản ổn định gần nhất
if [ -f .env.stable_version ]; then
    export APP_IMAGE_TAG=$(cat .env.stable_version)
    export APP_IMAGE_REGISTRY="<private-registry>"
else
    echo "ERROR: Không tìm thấy tệp ghi nhận phiên bản ổn định trước đó (.env.stable_version)!"
    exit 2
fi

echo "-> Đang rollback về phiên bản cũ: $APP_IMAGE_TAG..."

# 2. Khôi phục dữ liệu CSDL về thời điểm trước deploy (nếu lỗi do migration gây ra)
if [ -f /var/lib/fixtrack/backups/pre_deploy_backup.sql.gz ]; then
    echo "-> Phát hiện bản sao lưu pre-deploy. Tiến hành khôi phục dữ liệu..."
    gunzip -c /var/lib/fixtrack/backups/pre_deploy_backup.sql.gz | docker exec -i fixtrack-postgres psql -U app_db_user -d fixtrack_prod
fi

# 3. Khởi chạy lại container ở phiên bản cũ
docker compose up -d --no-deps api-service worker-service

echo "SUCCESS: Đã khôi phục trạng thái ổn định ở phiên bản $APP_IMAGE_TAG!"
```

---

# Level 5.2 — Database Migration Strategy (Chiến lược Di cư CSDL)

Để tránh hiện tượng lệch cấu trúc bảng (Schema Mismatch) giữa mã nguồn API mới và CSDL cũ dẫn tới lỗi ứng dụng nghiêm trọng tại runtime, quy trình thực thi di cư dữ liệu được quy định cụ thể:

1.  **Tuyệt đối cấm** việc để container API tự động chạy migrations khi khởi động (`auto-run migrations on startup` thông qua ORM config). Việc này làm tăng nguy cơ race condition nếu chạy nhiều instances API song song, đồng thời làm chậm thời gian khởi động của container dẫn tới fail health check.
2.  **Quy trình Thực thi Di cư Tuần tự**:
    *   **Bước 1**: Pull image mới chứa các file migration mới nhất về VPS Host.
    *   **Bước 2**: Khởi chạy một container độc lập dạng chạy-xong-xóa (run-once) để thực thi lệnh migration trên DB hiện hữu:
        ```bash
        docker compose run --rm --entrypoint "npm run migration:run" api-service
        ```
    *   **Bước 3**: Chỉ khi lệnh trên trả về mã thoát thành công (`exit code 0`), quy trình deploy mới được đi tiếp đến bước khởi chạy container API chính thức. Nếu thất bại, toàn bộ tiến trình deploy dừng lại lập tức để cô lập lỗi.

## 5.3 Tương thích Ngược khi Di cư CSDL (Migration Backward Compatibility - Expand-Contract Pattern)

Để đảm bảo nếu quá trình cập nhật mã nguồn gặp lỗi và hệ thống phải kích hoạt `rollback.sh` về phiên bản code cũ, CSDL không bị crash do lệch cấu trúc. FixTrack áp dụng quy tắc thiết kế di cư **Expand-Contract (Mở rộng trước - Co hẹp sau)**:

*   **Không thực hiện đổi tên (Rename) hoặc xóa (Drop) cột/bảng trực tiếp trong bản cập nhật nghiệp vụ chính**: Bản di cư CSDL đi kèm mã nguồn mới chỉ được phép là **Additive Changes (Thay đổi mở rộng)**.
    *   *Ví dụ*: Nếu cần đổi tên cột `dien_thoai` thành `so_dien_thoai`:
        1.  **Bước 1 (Expand)**: Tạo cột mới `so_dien_thoai` (cho phép chứa giá trị Null hoặc có giá trị Default), copy dữ liệu cũ sang và duy trì cập nhật song song ở cả hai cột. Phiên bản code cũ (v1) vẫn ghi dữ liệu vào cột `dien_thoai` cũ mà không lỗi.
        2.  **Bước 2 (Deploy Code)**: Rollout code mới (v2) chuyển sang đọc/ghi hoàn toàn trên `so_dien_thoai`.
        3.  **Bước 3 (Contract)**: Sau một khoảng thời gian chạy ổn định trên Production (ví dụ: 1 tuần), tiến hành chạy bản migration dọn dẹp để drop cột `dien_thoai` cũ.
*   **Nguyên tắc cho phép Null (Nullable Constraints)**: Mọi cột mới được thêm vào trong các đợt phát hành bắt buộc phải định nghĩa là `NULL` hoặc có thuộc tính `DEFAULT`. Nghiêm cấm tạo cột mới có thuộc tính `NOT NULL` mà không có giá trị mặc định, vì code cũ (v1) khi ghi nhận bản ghi mới sẽ bị DB báo lỗi thiếu trường dữ liệu.

---

# Level 5.5 — CI/CD Pipeline (GitHub Actions Workflow)

Hệ thống FixTrack cấu hình quy trình CI/CD tự động bằng GitHub Actions:

```
[ Developer Git Push ] 
         │
         ├──► [ Step 1: Lint & Unit Tests ]
         │         │ (Pass)
         │         ▼
         ├──► [ Step 2: Build Multi-stage Image ]
         │         │
         │         ▼
         ├──► [ Step 3: Trivy Security Scan ] ──(CVE Found?)──► [ FAIL & STOP ]
         │         │ (Clean)
         │         ▼
         ├──► [ Step 4: Push to Private Registry ]
         │         │
         │         ▼
         └──► [ Step 5: SSH Deploy / Database Migration / Container Rollout ]
```

*   **Quy định Nhánh (Branch Policy)**:
    *   Mọi commit đẩy lên nhánh `develop` sẽ tự động trigger quy trình build và deploy lên môi trường **Staging (UAT)**.
    *   Môi trường **Production** chỉ được trigger khi tạo một **GitHub Release Tag** (định dạng `v*.*.*`) và bắt buộc có sự phê duyệt thủ công (Manual Gate Approval) của Tech Lead/DevOps Lead trên giao diện GitHub.
*   **Quy định Thất bại (Security Gate Fail Rule)**:
    *   Nếu bước quét bảo mật hình ảnh (`trivy image`) phát hiện bất kỳ lỗ hổng nào thuộc mức `CRITICAL` hoặc mức `HIGH` chưa được vá (unpatched), pipeline sẽ bị đánh dấu thất bại lập tức và chặn việc phát hành.

---

# Level 6 — Runtime Logging & Operations (Nhật ký Vận hành)

Để tránh máy chủ VPS Host bị tràn ổ đĩa cứng (Disk Full) do tệp nhật ký ghi đè liên tục trong thời gian dài, FixTrack triển khai chiến lược xoay vòng log (Log Rotation) ở cấp độ Docker Daemon kết hợp:

1.  **Xoay vòng Tệp tin Log (Docker Log Rotation)**:
    Mọi container khai báo trong `docker-compose.yml` bắt buộc phải cấu hình driver logging kiểu `json-file` giới hạn dung lượng:
    *   `max-size: "10m"`: Mỗi file log của container chỉ được phép đạt dung lượng tối đa 10 Megabytes.
    *   `max-file: "5"`: Giữ tối đa 5 file log lịch sử. File cũ nhất sẽ bị xóa đi khi xuất hiện file log mới. Dung lượng log tối đa cho mỗi container luôn được khống chế dưới 50MB.
2.  **Đầu ra Nhật ký chuẩn (Standard Streams)**:
    Ứng dụng NestJS cấu hình ghi log trực tiếp ra cổng đầu ra tiêu chuẩn `stdout` (dành cho log thường) và `stderr` (dành cho log lỗi). Không thực hiện ghi log trực tiếp ra các tệp tin text bên trong container để tránh phình dung lượng bộ nhớ tạm (container writable layer).
3.  **Lộ trình tương lai**:
    Tích hợp các công cụ thu thập log tập trung như **Promtail / Loki** hoặc **Filebeat / ElasticSearch** để đẩy logs ra khỏi ổ cứng VPS về cụm lưu trữ log chuyên biệt.
