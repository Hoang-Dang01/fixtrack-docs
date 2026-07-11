# 37 Release Strategy

**Trạng thái: Hoàn thành (Finalized - Version 1.0)**

---

## Purpose

Chương này đặc tả chi tiết chiến lược phát hành phần mềm (Release Strategy Specification) của hệ thống FixTrack. Tài liệu định nghĩa mô hình quản lý nhánh Git kết hợp Semantic Versioning, quy chế bàn giao và nâng cấp artifact (Environment Promotion), kế hoạch di cư cơ sở dữ liệu an toàn (Expand-Contract), kỹ nghệ phát hành ngầm bằng cờ tính năng (Feature Flagging), và các tiêu chuẩn chốt chặn nghiệm thu (Release Acceptance & Rollback Gates) đi kèm khung thời gian theo dõi sau phát hành.

---

## Inputs

*   [09 Business Rules](../part_b_requirement_analysis/09_business_rules.md)
*   [30 Infrastructure](./30_infrastructure.md)
*   [31 Deployment](./31_deployment.md)
*   [32 CI/CD](./32_ci_cd.md)

---

# Level 1 — Branching, Versioning & Release Types (Chiến lược Phát hành)

Hệ thống FixTrack áp dụng mô hình Gitflow rút gọn (Gitflow-lite) kết hợp tiêu chuẩn định danh phiên bản SemVer 2.0.0 để kiểm soát chất lượng mã nguồn.

## 1.1 Chiến lược Quản lý Nhánh Git (Gitflow-lite)

*   `main`: Nhánh chứa mã nguồn chạy chính thức (Production). Chỉ được cập nhật thông qua Pull Request từ `release/*` hoặc `hotfix/*` sau khi đã nghiệm thu kỹ càng.
*   `develop`: Nhánh tích hợp chính (Staging/UAT). Toàn bộ tính năng mới được merge vào đây để chạy thử nghiệm.
*   `release/*`: Nhánh đóng băng tính năng, ổn định hóa phiên bản và chạy các bài kiểm thử khép kín cuối cùng (Release Candidates - RC) trước khi sáp nhập vào `main`.
*   `feature/*`: Nhánh phát triển tính năng mới của lập trình viên, tách ra từ `develop`.
*   `hotfix/*`: Nhánh sửa lỗi khẩn cấp trực tiếp cho Production, tách ra từ `main` và sau đó merge ngược lại vào cả `main` và `develop`.

```
           ┌─── feature/ticket-auth ───┐ (Merge)
           │                           ▼
develop ───┴───────────────────────────★───► (Auto deploy to Staging)
                                       │
                                       ▼ (Release branch created)
release/v1.1.0 ────────────────────────★───► (Final Testing)
                                       │
                                       ├────────────────────────┐
                                       ▼ (Deploy to Prod)       ▼ (Merge back)
main ──────────────────────────────────★────────────────────────┴──►
                                    (Tag: v1.1.0)
```

## 1.2 Quy tắc Đánh số Phiên bản (Semantic Versioning 2.0.0)

Mọi bản phát hành chính thức bắt buộc phải gắn thẻ Git Tag theo chuẩn SemVer:
$$\text{Phiên bản} = \text{v} + \text{Major} + \text{"."} + \text{Minor} + \text{"."} + \text{Patch}$$

Ví dụ: `v1.2.4`

*   **Major (Chính)**: Tăng khi có thay đổi lớn phá vỡ tính tương thích ngược (e.g., v1.0.0 -> v2.0.0).
*   **Minor (Phụ)**: Tăng khi thêm tính năng mới nhưng vẫn tương thích ngược (e.g., v1.2.0 -> v1.3.0).
*   **Patch (Sửa lỗi)**: Tăng khi có bản sửa lỗi nhỏ tương thích ngược (e.g., v1.2.3 -> v1.2.4).

## 1.3 Phân loại Bản Phát hành (Release Types)

| Loại phát hành (Type) | Quy chuẩn phiên bản | Tần suất phát hành | Phương thức kích hoạt |
| :--- | :--- | :--- | :--- |
| **Major Release** | Thay đổi số `Major` (v2.0.0) | 6 tháng - 1 năm | Cần họp Hội đồng Nghiệm thu & DevOps Lead duyệt thủ công. |
| **Minor Release** | Thay đổi số `Minor` (v1.4.0) | 2 - 4 tuần (Theo Sprint) | Tự động chạy CI/CD khi tạo Release branch, yêu cầu Tech Lead ký duyệt. |
| **Patch / Hotfix** | Thay đổi số `Patch` (v1.4.3) | Bất kỳ lúc nào (Khẩn cấp) | DevOps / Tech Lead kích hoạt trực tiếp từ nhánh hotfix khi có sự cố SEV-1/SEV-2. |

---

# Level 2 — Release Artifact & Environment Promotion (Quản lý Artifact & Bàn giao)

Nhằm đảm bảo lỗi không phát sinh do việc biên dịch lại mã nguồn trên các môi trường khác nhau, FixTrack áp đặt nguyên tắc: **Chỉ build một lần, chuyển dịch một artifact duy nhất qua các môi trường**.

```
[ Code Commit ] ──► [ Build & Test ] ──► [ Package Artifact: Image Tag v1.2.0 ]
                                                      │
                                                      ▼ (Deploy & Validate)
                                             [ Staging Environment ]
                                                      │
                                                      ▼ (Promote same Image without rebuilding)
                                            [ Production Environment ]
```

## 2.1 Artifact Bất biến (Immutable Artifacts)
*   **Định nghĩa**: Release Unit (Đơn vị phát hành) của FixTrack là một **Docker Image** được đóng gói duy nhất, định danh bằng mã băm commit và số phiên bản (`fixtrack-app:v1.3.2-a1b2c3d`).
*   **Chính sách bàn giao (Promotion Policy)**: Image sau khi vượt qua các vòng kiểm thử CI và deploy lên môi trường Staging thành công sẽ được **giữ nguyên dạng** để promote (bàn giao) lên Production.
*   **Nghiêm cấm** hành vi rebuild lại image từ mã nguồn Git trên server Production để loại bỏ hoàn toàn rủi ro lệch phiên bản thư viện bên thứ ba (Dependencies mismatch).

---

# Level 3 — Rollout Deployment Strategies (Chiến lược Triển khai)

## 3.1 Giai đoạn Phase 1 — Recreate Deployment (Hiện tại)
*   **Phương pháp**: Tắt container API cũ, cập nhật và khởi chạy container API mới.
*   **Đánh đổi**: Chấp nhận thời gian ngưng dịch vụ cực ngắn (Downtime 3 - 5 giây). Đây là lựa chọn tối ưu chi phí phần cứng và quản lý vận hành trong giai đoạn MVP (Single VPS).

## 3.2 Giai đoạn Phase 2 — HA Scale / Blue-Green & Rolling (Tương lai)
*   Để đạt mục tiêu Zero-Downtime khi hệ thống mở rộng, FixTrack thiết lập lộ trình nâng cấp hạ tầng mạng:
    *   **Công cụ Load Balancing**: Sử dụng **Caddy**, **Traefik**, **HAProxy** hoặc **Cloud Load Balancer** của nhà cung cấp hạ tầng để phân luồng.
    *   **Rolling Update**: Cập nhật cuốn chiếu từng phần các container API đằng sau Proxy.
    *   **Blue-Green Deployment**: Duy trì 2 môi trường song song (Blue chạy code cũ, Green chạy code mới). Khi Green vượt qua smoke test, Load Balancer đảo cấu hình trỏ toàn bộ traffic sang Green trong 1 giây.

## 3.3 Ma trận So sánh các Chiến lược Rollout

| Tiêu chí so sánh | Recreate (Phase 1) | Blue-Green (Phase 2) | Rolling Update (Phase 2) |
| :--- | :---: | :---: | :---: |
| **Downtime** | 3 - 5 giây | **0 giây (Không downtime)** | **0 giây (Không downtime)** |
| **Chi phí hạ tầng** | Thấp (1 VPS) | Cao (Yêu cầu $2\times$ resources) | Trung bình (Yêu cầu dung lượng đệm) |
| **Mức độ phức tạp** | Rất đơn giản | Trung bình - Cao | Trung bình |
| **Tốc độ Rollback** | Nhanh (Restart tag cũ) | Cực nhanh (Đổi trỏ Proxy) | Chậm (Rollback từng instance) |

---

# Level 4 — Safe Database Release Pattern (Mô hình CSDL Tương thích ngược)

Để đảm bảo database luôn tương thích ngược, hệ thống áp dụng mẫu thiết kế **Expand-Contract (Mở rộng trước - Co hẹp sau)**.

## 4.1 Quy trình Expand-Contract 3 bước khi đổi tên cột (Rename column)
Giả định cần đổi tên cột `dien_thoai` thành `so_dien_thoai` trong bảng `users`:

```
Trạng thái ban đầu: Code v1 tương tác với cột [dien_thoai]

Bước 1: Expand (Deploy Bản phát hành A)
  - Chạy migration thêm cột mới [so_dien_thoai] (để Nullable).
  - Code v2 được deploy: Đọc từ [so_dien_thoai] (nếu có), nếu không fallback về [dien_thoai]. 
    Ghi dữ liệu mới đồng thời vào CẢ HAI cột.
  - Kết quả: Code v1 (nếu rollback) và Code v2 đều chạy ổn định không lỗi.

Bước 2: Data Sync & Cutover (Deploy Bản phát hành B)
  - Chạy script chạy ngầm đồng bộ toàn bộ dữ liệu từ cột cũ sang cột mới.
  - Deploy code v3: Đọc và ghi duy nhất trên cột mới [so_dien_thoai]. 
    Cột cũ [dien_thoai] không còn được tương tác.

Bước 3: Contract (Deploy Bản phát hành C)
  - Chạy migration drop cột cũ [dien_thoai] sau khi hệ thống đã chạy ổn định 1 tuần.
```

## 4.2 Schema Version Tracking (Giám sát phiên bản Schema)
*   Hệ thống sử dụng bảng **`_prisma_migrations`** (hoặc bảng `typeorm_metadata` / `migrations` tương đương tùy cấu hình ORM) để lưu trữ lịch sử và trạng thái chạy các bản migrations:
    ```sql
    SELECT id, checksum, applied_steps_count FROM _prisma_migrations WHERE rolled_back_at IS NULL;
    ```
*   Mỗi khi triển khai, tiến trình CI/CD sẽ đối soát checksum và ID của phiên bản migration hiện hành trong CSDL với danh sách tệp di cư trong thư mục `prisma/migrations` để xác minh trạng thái đồng bộ cấu trúc trước khi cho phép rollout.

---

# Level 5 — Feature Flagging & Dark Launching (Cờ Tính năng & Triển khai ngầm)

Tách biệt quá trình triển khai mã nguồn (Deployment) khỏi thời điểm phát hành tính năng tới người dùng (Release) bằng kỹ thuật Feature Flags.

```
[ Deploy Code ] ──► [ Feature Flags: DISABLED ] ──► (Tính năng ngầm, không lộ ra UI)
                               │
                       (Duyệt QA thành công)
                               ▼
                    [ Feature Flags: ENABLED ]  ──► (Kích hoạt cho người dùng cuối)
```

## 5.1 Kiến trúc Feature Flag trong Ứng dụng
Hệ thống lưu cấu hình cờ tính năng trong database hoặc service cấu hình động. Lớp logic nghiệp vụ sẽ kiểm tra trạng thái cờ trước khi trả dữ liệu:

```typescript
// Ví dụ logic kiểm tra cờ tính năng (Feature Gate Interceptor / Guard)
if (this.featureFlags.isEnabled('new-repair-scheduler')) {
    return this.newScheduler.execute();
} else {
    return this.legacyScheduler.execute();
}
```

## 5.2 Phân quyền và Trách nhiệm quản trị Cờ (Feature Flag Ownership)

| Thao tác (Action) | Người thực hiện (Owner) | Môi trường (Environment) |
| :--- | :--- | :--- |
| **Khai báo & Tạo mới cờ** | Lập trình viên (Developer) | Local / Dev branch |
| **Bật cờ thử nghiệm** | Kỹ sư kiểm thử (QA / QC) | Staging (UAT) |
| **Bật cờ sử dụng chính thức** | Tech Lead / Product Manager | Production (Phát hành cuốn chiếu) |
| **Dọn dẹp cờ (Cleanup / Hardcode)** | Lập trình viên (Developer) | Sprint kế tiếp sau khi tính năng chạy ổn định |

---

# Level 6 — Acceptance Criteria & Rollback Gates (Tiêu chuẩn Nghiệm thu & Quay lui)

## 6.1 Khung thời gian Theo dõi sau Phát hành (Post-release Observation Window)

Kỹ sư trực vận hành bắt buộc phải duy trì chế độ giám sát chặt chẽ theo các khung thời gian tiêu chuẩn:

```
[ DEPLOY SUCCESS ]
       │
       ├──► 15 phút đầu: Critical Monitoring (Theo dõi chí mạng)
       │    - Giám sát CPU/RAM, Tỷ lệ lỗi HTTP 5xx, Logs lỗi crash runtime.
       │
       ├──► 1 giờ tiếp theo: Heightened Monitoring (Theo dõi nâng cao)
       │    - Giám sát DB Connections, Tốc độ hàng chờ BullMQ, Cache hit rate.
       │
       └──► 24 giờ tiếp theo: Regression Watch (Theo dõi lỗi tái phát)
            - Theo dõi cảnh báo hồi quy (Regression alerts) trên Sentry.
```

## 6.2 Tiêu chuẩn Chốt chặn Quay lui (Rollback Decision Gates)

Lập tức thực thi lệnh quay lui phiên bản (`rollback.sh`) nếu hệ thống vi phạm bất kỳ chỉ số an toàn nào trong khung thời gian 15 phút đầu tiên:
*   **Health Check Failure**: API container báo trạng thái `unhealthy` quá 2 phút.
*   **API Latency Spike**: p95 Latency của toàn hệ thống tăng đột biến vượt quá **1.5 giây** trong 5 phút liên tục.
*   **High Error Rate**: Tỷ lệ lỗi HTTP 5xx vượt quá **10%** tổng lượng request.
*   **Database Lock**: Xuất hiện hiện tượng nghẽn luồng truy vấn (connection locks) kéo dài làm cạn kiệt connection pool.
*   **Business KPI Degradation (Lỗi Nghiệp vụ)**: Tỷ lệ tạo phiếu báo hỏng xe thành công trượt dưới **95%** hoặc tỷ lệ đăng nhập thành công giảm quá **20%** trong vòng 10 phút liên tục (báo động lỗi logic validation hoặc auth flow).
