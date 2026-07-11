# 25 Cache Strategy

**Trạng thái: Hoàn thành (Finalized - Version 1.0)**

---

## Purpose

Chương này đặc tả chi tiết chiến lược bộ nhớ đệm (Cache Strategy) của hệ thống FixTrack. Tài liệu định nghĩa phân loại cache, danh sách các thực thể áp dụng cache (Cache Registry), thời gian sống (TTL), cơ chế xóa cache (Eviction), giải pháp bảo mật và các cơ chế phòng chống lỗi bộ đệm kinh điển nhằm đảm bảo tốc độ phản hồi tối ưu và bảo vệ cơ sở dữ liệu PostgreSQL khỏi quá tải.

---

## Inputs

*   [15 Database Schema](../part_c_data_design/15_database_schema.md)
*   [18 State Machine](../part_d_application_design/18_state_machine.md)
*   [21 API Design](./21_api_design.md)
*   [22 Backend Modules](./22_backend_modules.md)
*   [24 Background Jobs](./24_background_jobs.md)

---

# Level 1 — Caching Architecture & Categories

Hệ thống FixTrack sử dụng **Redis** làm cơ sở dữ liệu lưu trữ bộ nhớ đệm phân tán (Distributed Cache) tập trung kết hợp với **NestJS Cache Manager** để phục vụ 3 mục đích nghiệp vụ chuyên biệt:

```
                          ┌───────────────────────────┐
                          │   FixTrack Backend (API)  │
                          └─────────────┬─────────────┘
          ┌─────────────────────────────┼─────────────────────────────┐
          ▼                             ▼                             ▼
┌───────────────────┐         ┌───────────────────┐         ┌───────────────────┐
│    Data Cache     │         │   Security Cache  │         │ Coordination Cache│
│ - materials       │         │ - session_store   │         │ - distributed     │
│ - warehouses      │         │ - token_blacklist │         │   mutex locks     │
│ - dashboard_stats │         │ - rate_limit      │         │                   │
└───────────────────┘         └───────────────────┘         └───────────────────┘
```

## 1.1 Cache Categories (Phân loại Bộ nhớ đệm)
1.  **Data Cache (Bộ nhớ đệm Dữ liệu)**:
    *   *Mục tiêu*: Rút ngắn thời gian phản hồi của API đọc, giảm tải truy vấn (CPU/Read IOPS) cho PostgreSQL.
    *   *Mẫu thiết kế*: Áp dụng mẫu **Cache-Aside (Read-Through)**. Ứng dụng đọc từ Redis trước, nếu trống (Cache Miss) sẽ truy vấn DB, lưu lại Redis và trả về kết quả.
2.  **Security Cache (Bộ nhớ đệm Bảo mật)**:
    *   *Mục tiêu*: Lưu trữ phiên làm việc, giới hạn tần suất gọi API, và quản lý các token bị thu hồi.
    *   *Mẫu thiết kế*: Lưu trực tiếp vào Redis bằng cấu trúc dữ liệu tối ưu với thời gian sống (TTL) tự động thu hồi.
3.  **Coordination Cache (Bộ nhớ đệm Phối hợp)**:
    *   *Mục tiêu*: Đồng bộ tiến trình xử lý song song giữa các instances của Backend (ví dụ: lock tài nguyên xe hoặc cầu sửa chữa).
    *   *Mẫu thiết kế*: Khóa phân tán (Distributed Mutex Locks) sử dụng thuộc tính nguyên tử `SET NX PX` của Redis.

---

# Level 2 — Cache Registry Matrix

Dưới đây là bảng đăng ký cấu hình các khóa bộ nhớ đệm (Cache Keys) trên Redis của FixTrack:

| Phân nhóm | Khóa Redis (Key Pattern) | Cấu trúc dữ liệu Redis | Thời gian sống (TTL) | Cơ chế xóa cache (Eviction Strategy) | Module quản lý | Mục đích nghiệp vụ |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Data** | `materials:catalog` | `String` (JSON) | 30 phút $\pm$ Jitter | Xóa ngay khi cập nhật/xóa danh mục vật tư | `MaterialModule` | Cache danh mục phụ tùng |
| **Data** | `warehouses:list` | `String` (JSON) | 60 phút $\pm$ Jitter | Xóa ngay khi thay đổi danh sách kho | `MaterialModule` | Cache danh sách kho vật tư |
| **Data** | `workshop_slots:list` | `String` (JSON) | 30 giây | Tự động hết hạn (không sử dụng cơ chế xóa thủ công) | `RepairModule` | Cache danh sách cầu sửa chữa |
| **Data** | `dashboard_stats:realtime` | `Hash` | 60 giây | Tự động hết hạn | `RepairModule` / `TicketModule` | Cache số liệu động (xe chờ, xe đang sửa) |
| **Data** | `dashboard_stats:analytics`| `String` (JSON) | 15 phút $\pm$ Jitter | Xóa khi có ticket đổi trạng thái cuối (`closed`) | `TicketModule` | Cache báo cáo nặng (KPI, hiệu suất, MTTR) |
| **Data** | `notif:unread:{user_id}` | `String` (Integer) | 2 phút | Xóa khi user đọc thông báo hoặc nhận tin mới | `NotificationModule`| Cache số lượng thông báo chưa đọc |
| **Data** | `queue:active_snapshot` | `String` (JSON) | 30 giây | Xóa khi có thay đổi vị trí trong hàng chờ | `WorkshopQueueModule`| Snapshot hiển thị bảng chờ tại xưởng |
| **Security** | `session:{user_id}:{session_id}`| `Hash` | 7 ngày (trùng Refresh Token) | Xóa lập tức khi người dùng đăng xuất (Logout) | `AuthModule` | Lưu trữ phiên làm việc, quản lý đa thiết bị |
| **Security** | `blacklist:{jti}` | `String` | TTL của Access Token (tối đa 15p) | Tự động hết hạn | `AuthModule` | Chứa danh sách đen các JWT bị thu hồi |
| **Security** | `rate_limit:{ip}:{endpoint}`| `String` (Integer) | 1 phút | Tự động hết hạn | `SharedKernel` | Bộ đếm giới hạn tần suất gọi API |
| **Coord** | `lock:{resource_name}` | `String` (Mutex) | Tối đa 10 giây | Xóa chủ động bằng lệnh DEL sau khi xong tác vụ | `SharedKernel` | Khóa phân tán chống trùng lặp ghi |

---

# Level 3 — Cache Invalidation & Inconsistency Prevention

Để ngăn chặn việc dữ liệu trong Cache bị lệch pha (Stale Data) so với Cơ sở dữ liệu PostgreSQL:

## 3.1 Chiến lược Xóa Cache chủ động (Active Cache Invalidation / Write-Invalidate)
Các API làm thay đổi dữ liệu (`POST`/`PATCH`/`DELETE`) sau khi thực thi thành công Database Transaction phải phát hành lệnh xóa (evict/invalidate) các cache tương ứng trong Redis trước khi trả về HTTP Response:
```
Client ──► API Route ──► DB Transaction Success ──► Redis EVICT Key ──► Client Response Success
```
*   Khi sửa đổi vật tư $\rightarrow$ Evict khóa `materials:catalog`.
*   Khi có thay đổi hàng chờ (xếp nốt, promote) $\rightarrow$ Evict khóa `queue:active_snapshot`.
*   Khi người dùng click đọc thông báo $\rightarrow$ Evict khóa `notif:unread:{user_id}`.

## 3.2 Tách biệt Bộ số liệu Dashboard (Dashboard Cache Split)
Để tối ưu hóa hiệu năng và tránh việc tính toán lại báo cáo quá nhiều lần:
1.  **Fast-changing metrics (`dashboard_stats:realtime`)**:
    *   Các số liệu về số lượng xe trong hàng chờ, số xe đang nằm cầu sửa chữa thay đổi rất nhanh.
    *   *Giải pháp*: Cấu hình TTL siêu ngắn (60 giây), không thực hiện xóa chủ động khi ticket thay đổi trạng thái. Chấp nhận dữ liệu trễ tối đa 1 phút.
2.  **Slow-changing metrics (`dashboard_stats:analytics`)**:
    *   Các số liệu về tỷ lệ xe hoàn thành sửa chữa trong tuần, thời gian sửa chữa trung bình (MTTR), hiệu suất của kỹ thuật viên cần tính toán nặng (nhiều lệnh JOIN phức tạp).
    *   *Giải pháp*: Cấu hình TTL dài (15 phút), chỉ thực hiện xóa chủ động khi phiếu sửa chữa được nghiệm thu đóng hoàn toàn (`closed`).

---

# Level 4 — Advanced Cache Robustness & Reliability

Hệ thống FixTrack thiết lập các giải pháp phòng vệ nghiêm ngặt chống lại các nguy cơ lỗi hệ thống Cache kinh điển:

```
                            Cache Request
                                  │
                         [Cache Key Exists?]
                       (Yes) /         \ (No)
                            ▼           ▼
                      Return Data   [Lock Acquired via Mutex?]
                                     (Yes) /           \ (No)
                                          ▼             ▼
                                     Query DB &     Wait & Retry
                                    Update Cache     Fetch Cache
```

## 4.1 Chống Lở bộ đệm (Cache Avalanche Protection)
*   **Nguy cơ**: Khi hệ thống khởi động lại hoặc khi một lượng lớn cache có cùng TTL hết hạn đồng thời, hàng loạt truy vấn sẽ dội thẳng vào Database PostgreSQL gây quá tải.
*   **Giải pháp**: Áp dụng cơ chế lệch thời gian sống ngẫu nhiên (TTL Jitter). Thời gian sống thực tế của khóa được tính toán bằng công thức:
    $$\text{TTL}_{\text{actual}} = \text{TTL}_{\text{base}} \pm \text{Random}(0, \text{Jitter})$$
    *Ví dụ*: Với `materials:catalog` có $\text{TTL}_{\text{base}} = 30$ phút, $\text{Jitter} = 5$ phút $\rightarrow$ TTL thực tế sẽ dao động ngẫu nhiên từ 25 đến 35 phút.

## 4.2 Chống Thống kê dồn dập (Cache Stampede Protection - Mutex Lock)
*   **Nguy cơ**: Khi một khóa cache phổ biến (Hot Key) hết hạn, hàng trăm request đồng thời truy cập sẽ thấy Cache Miss và cùng lúc gửi truy vấn nặng tới Database.
*   **Giải pháp**: Sử dụng cơ chế khóa phân tán Mutex đơn giản dựa trên thuộc tính nguyên tử `SET NX PX` của Redis. Để tránh nguy cơ tràn ngăn xếp (Stack Overflow) khi đệ quy sâu, hệ thống sử dụng thuật toán vòng lặp thử lại có giới hạn (Loop-based Retry với Timeout):
    ```typescript
    async getOrSet<T>(key: string, fetchFn: () => Promise<T>, ttl: number): Promise<T> {
      const maxRetries = 20;
      const backoffMs = 100; // Đợi 100ms mỗi lần thử lại
      
      for (let attempt = 0; attempt < maxRetries; attempt++) {
        let data = await this.redis.get(key);
        if (data) {
          if (data === 'EMPTY_VALUE') return null as any;
          return JSON.parse(data);
        }
        
        const lockKey = `lock:${key}`;
        const acquired = await this.redis.set(lockKey, '1', 'NX', 'PX', 10000); // Lock tối đa 10s
        
        if (acquired) {
          try {
            // Chỉ 1 instance duy nhất được quyền truy vấn DB
            const result = await fetchFn();
            if (result === null || result === undefined) {
              await this.redis.set(key, 'EMPTY_VALUE', 'EX', 60); // Null Caching (Ch. 4.3)
            } else {
              await this.redis.set(key, JSON.stringify(result), 'EX', ttl);
            }
            return result;
          } finally {
            await this.redis.del(lockKey); // Giải phóng lock chủ động
          }
        }
        
        // Chờ đợi trước khi thử lại ở vòng lặp kế tiếp
        await this.sleep(backoffMs);
      }
      
      throw new CacheTimeoutException('Vượt quá thời gian chờ khóa phân tán Redis (Timeout: 2s)');
    }
    ```
    *Lưu ý*: Thiết kế khóa mutex đơn giản trên 1 node Redis này phù hợp với quy mô hiện tại. Nếu tương lai nâng cấp lên cụm Redis Multi-node cluster, hệ thống sẽ chuyển sang dùng giải pháp Redlock.

## 4.3 Chống Thủng bộ đệm (Cache Penetration Protection)
*   **Nguy cơ**: Client liên tục truy vấn các thực thể với ID không tồn tại (do bị lỗi phần mềm hoặc bị tấn công Brute-force). Hệ thống liên tục gặp Cache Miss và dội truy vấn xuống DB để check.
*   **Giải pháp**: Áp dụng **Null Caching**. Khi truy vấn DB không tìm thấy bản ghi (trả về `null`), hệ thống vẫn ghi nhận giá trị rỗng đó vào Redis với TTL siêu ngắn (1 phút):
    ```typescript
    if (!result) {
      await this.redis.set(key, 'EMPTY_VALUE', 'EX', 60); // Cache rỗng trong 60 giây
    }
    ```

## 4.4 Khởi động và Nạp trước Cache (Cache Warmup / Preload Strategy)
*   **Nguy cơ**: Khi hệ thống khởi động lại (Cold Start) hoặc Redis bị crash sạch dữ liệu, các request đầu tiên dội vào sẽ làm sụt giảm hiệu năng nghiêm trọng do toàn bộ cache bị trống và DB bị nghẽn lệnh.
*   **Giải pháp (Cache Preload)**: Đăng ký một trình khởi chạy ở sự kiện On Application Startup để tự động nạp trước (warmup) các tập dữ liệu nóng và nặng:
    *   *Warmup targets*: Danh mục vật tư (`materials:catalog`), danh sách kho (`warehouses:list`), và báo cáo phân tích tĩnh (`dashboard_stats:analytics`).
    *   *Triển khai*: Tích hợp interface `OnApplicationBootstrap` của NestJS để thực thi luồng tải dữ liệu từ PostgreSQL rồi lưu thẳng vào Redis trước khi mở cổng nhận request HTTP từ Client.

---

# Level 5 — Cache Observability & Monitoring

Để đảm bảo hiệu quả vận hành của bộ nhớ đệm, hệ thống thu thập các chỉ số giám sát sau:

1.  **Cache Hit Ratio (Tỷ lệ trúng bộ đệm)**:
    *   $$\text{Hit Ratio} = \frac{\text{Cache Hits}}{\text{Cache Hits} + \text{Cache Misses}} \times 100\%$$
    *   *Chỉ số KPI*: Tỷ lệ trúng bộ đệm toàn hệ thống phải đạt **tối thiểu 80%** trong điều kiện vận hành bình thường.
2.  **Redis Memory Usage (Sức chứa bộ nhớ Redis) & Eviction Policy**:
    *   Giám sát dung lượng RAM tiêu thụ trên Redis. 
    *   *Eviction Policy (Chính sách giải phóng bộ nhớ)*: Cấu hình chính sách **`volatile-lru`** hoặc **`volatile-ttl`** (chỉ xóa các key có cấu hình thời gian sống TTL hết hạn khi bộ nhớ đầy).
    *   > [!CAUTION]
        > **Bảo vệ khóa nghiệp vụ cốt lõi (Eviction Protection)**:
        > Do Redis được dùng chung cho cả Data Cache (có TTL) và Security/Coordination Cache (phiên làm việc `session`, token bị thu hồi `blacklist`, khóa `lock` nghiệp vụ), cấu hình **`allkeys-lru`** có nguy cơ xóa nhầm các khóa bảo mật quan trọng khiến token đã thu hồi hoạt động trở lại. 
        > *Giải pháp an toàn*: Chỉ dùng `volatile-lru`/`volatile-ttl`. Trong môi trường Production, **khuyến nghị tách riêng thành 2 thực thể Redis chạy vật lý riêng biệt**:
        > 1. *Redis Cache Instance*: Sử dụng `allkeys-lru` để tối ưu hóa bộ nhớ cho dữ liệu cache đọc.
        > 2. *Redis Security & Queue Instance*: Sử dụng **`noeviction`** để bảo vệ tuyệt đối dữ liệu phiên làm việc, hàng chờ BullMQ, khóa và token blacklist, không bao giờ tự ý xóa dữ liệu khi đầy bộ nhớ (thay vào đó ném ra lỗi tràn bộ nhớ để hệ thống ops cảnh báo).
3.  **Latency p95 (Độ trễ phản hồi)**:
    *   Độ trễ đọc ghi trên Redis phải nằm dưới mức **5ms** đối với p95.
