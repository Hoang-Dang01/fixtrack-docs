# 22 Backend Modules

**Trạng thái: Hoàn thành (Finalized - Version 1.0)**

---

## Purpose

Chương này đặc tả chi tiết kiến trúc phân rã module backend (Backend Module Breakdown) của hệ thống FixTrack. Tài liệu định nghĩa cách tổ chức mã nguồn theo hướng **Modular Monolith (Modulith)** trên nền tảng **NestJS**, quy chuẩn phân lớp (Clean Architecture / Layered Architecture), sơ đồ phụ thuộc giữa các module (Dependency Graph), cấu trúc thư mục chuẩn, cơ chế phân định Infrastructure, và vai trò của từng thành phần.

Tài liệu là nền tảng cho:
*   **Backend Developers**: Khởi tạo cấu trúc thư mục, triển khai các NestJS Modules, Controllers, Services và Providers.
*   **Architects**: Giám sát tính cô lập (encapsulation), quản lý phụ thuộc và tránh các lỗi phụ thuộc vòng (circular dependencies).
*   **QA Team**: Xác định ranh giới kiểm thử (Testing Boundaries) để viết unit, integration và contract tests.

---

## Questions Answered

*   Hệ thống backend được tổ chức phân lớp theo triết lý kiến trúc nào?
*   Mối quan hệ và ranh giới phụ thuộc giữa các Module Core, Infrastructure và Feature được quy định ra sao?
*   Cơ chế Event-Driven giúp khử phụ thuộc vòng (Anti-Circular Dependency) hoạt động như thế nào?
*   Cấu trúc thư mục chuẩn và AppModule Composition Root được khai báo ra sao?
*   Các sự kiện nghiệp vụ (Event Registry) có payload contract và naming convention thế nào?
*   Chiến lược testing boundaries và lộ trình tiến hóa lên Microservices được hoạch định ra sao?
ex
---

## Inputs

*   [05 User Roles](../part_a_business_foundation/05_user_roles.md)
*   [07 Functional Requirements](../part_b_requirement_analysis/07_functional_requirements.md)
*   [09 Business Rules](../part_b_requirement_analysis/09_business_rules.md)
*   [12 Logical DB Design & ERD](../part_c_data_design/12_erd.md)
*   [15 Database Schema](../part_c_data_design/15_database_schema.md)
*   [18 State Machine](../part_d_application_design/18_state_machine.md)
*   [21 API Design](./21_api_design.md)

---

# Level 1 — Architecture Philosophy

## 1.1 Modular Monolith (Modulith) Rationale
Hệ thống FixTrack lựa chọn mô hình kiến trúc **Modular Monolith** làm giải pháp triển khai ban đầu:
*   **Đơn giản hóa vận hành**: Một tiến trình chạy vật lý giúp đơn giản hóa CI/CD, phân bổ tài nguyên và giảm thiểu độ trễ mạng so với Microservices.
*   **Domain Boundary rõ ràng**: Mã nguồn được chia thành các module tương ứng với từng miền nghiệp vụ riêng biệt. Tính đóng gói (encapsulation) được bảo vệ bằng cách chỉ export các public interface cần thiết.
*   **Transaction nhất quán**: Cho phép thực thi ACID transaction trực tiếp trên PostgreSQL khi luồng nghiệp vụ đi xuyên qua nhiều bảng dữ liệu.

## 1.2 Clean Architecture Lite (Upgrade Path)
Để chuẩn bị cho khả năng mở rộng trong tương lai khi mã nguồn phình to, hệ thống áp dụng triết lý phân lớp theo hướng **Clean Architecture Lite** bên trong từng module Feature:
*   **Presentation Layer (Controllers / Gateways)**: Tiếp nhận request, chịu trách nhiệm parse header, validate DTO, kiểm tra quyền hạn (Guards).
*   **Application Layer (Use Cases / Application Services)**: Điều phối luồng nghiệp vụ. Nơi khởi tạo transactions, lấy dữ liệu từ repositories, thực thi các Use Case nghiệp vụ cụ thể.
*   **Domain Layer (Domain Services / Entities / State Machines)**: Chứa core business logic tinh khiết (ví dụ: máy trạng thái chuyển đổi phiếu, kiểm tra nghiệp vụ của xe). Lớp này hoàn toàn độc lập với các thư viện ngoài hoặc framework.
*   **Infrastructure Layer (Database Repositories / External Clients)**: Thực thi kỹ thuật cụ thể (ORM queries, S3 client, FCM client).

```
┌─────────────────────────────────────────────────────────┐
│ Presentation (Controllers, Socket Gateways)             │
│   └─► Application (Use Cases, Transaction Coordinators) │
│         └─► Domain (Pure State Machine, Enums, Rules)   │
│               └─► Infrastructure (PostgreSQL, Redis)    │
└─────────────────────────────────────────────────────────┘
```

## 1.3 Future Migration Path to Microservices
Khi hệ thống đạt tải lớn hoặc có sự phân hóa về tần suất sử dụng giữa các tính năng (ví dụ: luồng Notification và Realtime chịu tải cao hơn nhiều so với Material Management), lộ trình tách nhỏ hệ thống được hoạch định như sau:

```mermaid
chronology
    title Lộ trình tiến hóa hệ thống FixTrack
    Phase 1 (Modulith) : Triển khai 1 Database duy nhất & 1 Process NestJS chứa toàn bộ Module
    Phase 2 (Async Decoupling) : Tách Notification & Realtime thành các worker chạy process riêng sử dụng BullMQ chung
    Phase 3 (Full Microservices) : Tách Ticket, Material và Notification thành các dịch vụ độc lập với database riêng, giao tiếp qua RabbitMQ / Kafka
```

---

# Level 2 — Dependency Rules & Module Boundaries

Để tránh kiến trúc bị thoái hóa thành "Big Ball of Mud" (Bóng bùn lớn) sau một thời gian phát triển, hệ thống áp dụng các luật phụ thuộc cứng (Dependency Rules) dưới đây:

## 2.1 Allowed Dependencies (Phụ thuộc cho phép)
*   **Feature Modules** được phép import trực tiếp các module thuộc lớp **Core** và **Shared Kernel** (ví dụ: `ConfigModule`, `DatabaseModule`, `SharedKernelModule`).
*   **Feature Modules** được phép gọi các helper hoặc module thuộc lớp **Infrastructure** thông qua các Service Provider đăng ký toàn cục.
*   **Feature Modules** được phép phụ thuộc vào **EventBus** (`EventEmitter2`) để giao tiếp bất đồng bộ.

## 2.2 Forbidden Dependencies (Cấm phụ thuộc vòng)
*   **CẤM TUYỆT ĐỐI** việc import chéo (Bidirectional/Circular Imports) trực tiếp giữa các Feature Modules.
    *   *Ví dụ*: `TicketModule` import `RepairModule` và ngược lại `RepairModule` lại import `TicketModule` là hành vi bị chặn ở mức biên dịch (NestJS circular dependency warning).
*   **Giải pháp xử lý giao tiếp**:
    *   Sử dụng **Event-Driven Decoupling**: Khi một trạng thái thay đổi, Module phát ra sự kiện và kết thúc luồng xử lý. Module khác tự lắng nghe và thực hiện side-effect.
    *   Sử dụng **Shared Interfaces**: Định nghĩa các Interface chung tại `SharedKernel` để tiêm phụ thuộc (Dependency Injection) lỏng lẻo thông qua Tokens.

## 2.3 Giao tiếp đồng bộ và bất đồng bộ
*   **Giao tiếp đồng bộ (Synchronous)**: Dùng Direct Injection khi một Module cần đọc dữ liệu read-only từ module khác một cách nhanh chóng (ví dụ: `TicketService` inject `VehiclesService` để kiểm tra xe tồn tại).
*   **Giao tiếp bất đồng bộ (Asynchronous)**: Sử dụng Event Bus khi thay đổi trạng thái gây ra các hiệu ứng phụ (ví dụ: `MaterialRequestsService` duyệt cấp phát phát ra sự kiện `material.request.approved` để `NotificationModule` gửi push và `AuditModule` ghi log).

> [!NOTE]
> **Định hướng CQRS / Command Bus trong tương lai**:
> Khi các luồng nghiệp vụ đồng bộ đi xuyên qua nhiều Module trở nên phức tạp, hệ thống có thể tích hợp thêm **Command Bus** (ví dụ: `@nestjs/cqrs`). Việc này giúp giảm bớt coupling của việc inject trực tiếp Service giữa các module.
> *Lưu ý*: Command Bus là tùy chọn (optional) và chỉ được đưa vào khi việc điều phối lệnh đồng bộ giữa các module thực sự phức tạp để tránh over-engineering và chi phí overhead không đáng có của lớp trừu tượng.

## 2.4 Aggregate Ownership Rules (Quy tắc sở hữu Aggregate)
Để đảm bảo tính độc lập và toàn vẹn của mô hình Domain Domain-Driven Design (DDD) trong Modular Monolith, hệ thống quy định quy tắc phân định quyền sở hữu và thay đổi dữ liệu (mutation) như sau:

| Aggregate (Thực thể lõi) | Module sở hữu (Owner Module) |
| :--- | :--- |
| **RepairTicket** (Phiếu sửa chữa) | `TicketModule` |
| **Vehicle** (Phương tiện) | `VehicleModule` |
| **MaterialRequest** (Yêu cầu vật tư) | `MaterialModule` |
| **User** & **VehicleAssignment** | `UserModule` |
| **RepairJob** (Công việc sửa chữa) | `RepairModule` |
| **QueueEntry** (Dòng hàng chờ xưởng) | `WorkshopQueueModule` |
| **Notification** (Thông báo) | `NotificationModule` |
| **Attachment** (Tệp đính kèm) | `AttachmentModule` |

### Quy tắc thay đổi dữ liệu Aggregate:
*   **Chỉ module sở hữu (Owner Module)** mới được phép trực tiếp thay đổi (mutate) trạng thái hoặc dữ liệu của Aggregate đó (qua database write hoặc hàm Domain mutation trực tiếp).
*   **Yêu cầu thay đổi đồng bộ (Sync Mutation Request - Command)**: Nếu Module A muốn thay đổi dữ liệu Aggregate thuộc sở hữu của Module B và mong đợi phản hồi đồng bộ ngay lập tức, Module A **phải gọi qua Interface Service/Command** được cung cấp bởi Module B. Nghiêm cấm việc Module A tự ý ghi trực tiếp vào Database của Module B hoặc gọi thẳng hàm Repository của Module B.
*   **Tác vụ phụ bất đồng bộ (Async Side Effect - Event)**: Nếu thay đổi trạng thái ở Module A chỉ tạo ra phản ứng phụ (side effect) ở Module B mà không cần đợi kết quả phản hồi ngay lập tức, Module A **phải phát ra (Emit) một Domain Event** qua Event Bus. Module B sẽ tự lắng nghe (Subscribe) và thực thi logic thay đổi trên Aggregate của mình (Fire-and-forget).

---

# Level 3 — Module Dependency Graph

Sơ đồ thể hiện luồng phụ thuộc kiến trúc của hệ thống FixTrack, làm nổi bật vai trò trung gian giải vây phụ thuộc của **EventBus**:

```mermaid
graph TD
    %% Core & Shared Kernels
    DatabaseModule[Database Module]
    AuthModule[Auth Module]
    JobQueueModule[Job Queue Module]
    RealtimeModule[Realtime Module]
    AuditModule[Audit Module]
    ConfigModule[Config Module]
    SharedKernelModule[Shared Kernel Module]

    %% Event Bus (Decoupler)
    EventBus[EventEmitter2 EventBus]

    %% Business Features
    UserModule[User Module]
    VehicleModule[Vehicle Module]
    TicketModule[Ticket Module]
    WorkshopQueueModule[Workshop Queue Module]
    MaterialModule[Material Module]
    RepairModule[Repair Module]
    NotificationModule[Notification Module]
    AttachmentModule[Attachment Module]

    %% Flow of dependency
    UserModule --> AuthModule
    VehicleModule --> UserModule
    TicketModule --> DatabaseModule
    TicketModule --> AttachmentModule
    TicketModule --> SharedKernelModule
    
    MaterialModule --> DatabaseModule
    MaterialModule --> SharedKernelModule
    
    RepairModule --> DatabaseModule
    RepairModule --> SharedKernelModule

    %% Decoupling via Event Bus
    TicketModule -- Emits --> EventBus
    MaterialModule -- Emits --> EventBus
    RepairModule -- Emits --> EventBus

    EventBus -- Triggers --> NotificationModule
    EventBus -- Triggers --> AuditModule
    EventBus -- Triggers --> RealtimeModule
    EventBus -- Triggers --> WorkshopQueueModule

    %% Notification & Audit dependencies
    NotificationModule --> JobQueueModule
    NotificationModule --> RealtimeModule
    AuditModule --> DatabaseModule
```

---

# Level 4 — Directory & Project Structure

Mã nguồn được cấu trúc theo 4 phân vùng chính để phân định ranh giới giữa logic nghiệp vụ thuần túy và mã hạ tầng kỹ thuật:

```text
src/
├── core/                                 # Cấu hình lõi & Khởi tạo framework
│   ├── config/                           # ConfigModule & Validation
│   ├── database/                         # DatabaseModule (ORM setup)
│   └── auth/                             # AuthModule (JWT & Security)
│
├── infrastructure/                       # Các tích hợp kỹ thuật bên ngoài
│   ├── cache/                            # Redis Client setup (CacheModule)
│   ├── storage/                          # S3 / MinIO Storage Client
│   ├── messaging/                        # FCM (Firebase Cloud Messaging) Client
│   ├── gateway/                          # Socket.IO Gateway & Adapters (RealtimeModule)
│   └── queue/                            # BullMQ Queue & Workers (JobQueueModule)
│
├── shared-kernel/                        # Nhân dùng chung (Kernel / Shared Utilities)
│   ├── decorators/                       # Custom decorators (e.g., GetUser)
│   ├── filters/                          # Global Exception Filters
│   ├── interceptors/                     # Request/Response Enveloping
│   ├── pipes/                            # Validation Pipes
│   └── utils/                            # Date, String, Crypto helpers
│
├── modules/                              # Business Feature Modules (B Boundaries)
│   ├── user/
│   ├── vehicle/
│   ├── ticket/
│   ├── workshop-queue/                   # Quản lý hàng chờ sửa chữa tại xưởng
│   ├── material/
│   ├── repair/
│   ├── notification/
│   └── attachment/
│
├── app.module.ts                         # Root AppModule (Composition Root)
└── main.ts                               # Entry Point của ứng dụng

> [!TIP]
> **Quy tắc phân chia cấu trúc theo độ phức tạp (Use Case Architecture):**
> *   **Simple Modules** (như `AttachmentModule`, `VehicleModule`): Sử dụng cấu trúc NestJS phẳng truyền thống (`controllers/`, `services/`, `entities/`, `dto/`).
> *   **Complex Modules** (như `TicketModule`, `MaterialModule`): **Nên** phân tách nhỏ Service bằng cấu trúc thư mục Use Case ở tầng Application để tránh các file Service phình to quá mức (ví dụ: trên 1500 dòng):
>     ```text
>     modules/ticket/
>     ├── presentation/                  # Controllers, DTOs
>     ├── application/
>     │   ├── use-cases/                # Mỗi file là một luồng nghiệp vụ riêng biệt
>     │   │   ├── create-ticket.usecase.ts
>     │   │   ├── accept-ticket.usecase.ts
>     │   │   └── reject-ticket.usecase.ts
>     │   └── services/                 # Cung cấp facade/coordinator
>     └── domain/                       # Pure business entities & state guards
>     ```
```

## 4.1 AppModule Composition Root
Tất cả các module thành phần được lắp ghép tại file gốc `AppModule`. Đây là điểm duy nhất lắp ráp toàn bộ hệ thống:

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { EventEmitterModule } from '@nestjs/event-emitter';
import { DatabaseModule } from './core/database/database.module';
import { AuthModule } from './core/auth/auth.module';
import { SharedKernelModule } from './shared-kernel/shared-kernel.module';

// Business Modules
import { UserModule } from './modules/user/user.module';
import { VehicleModule } from './modules/vehicle/vehicle.module';
import { TicketModule } from './modules/ticket/ticket.module';
import { WorkshopQueueModule } from './modules/workshop-queue/workshop-queue.module';
import { MaterialModule } from './modules/material/material.module';
import { RepairModule } from './modules/repair/repair.module';
import { NotificationModule } from './modules/notification/notification.module';
import { AttachmentModule } from './modules/attachment/attachment.module';

@Module({
  imports: [
    // Core Configurations
    ConfigModule.forRoot({ isGlobal: true }),
    EventEmitterModule.forRoot({ wildcard: true }),
    
    // Core System Modules
    DatabaseModule,
    AuthModule,
    SharedKernelModule,

    // Business Domains (Modulith Modules)
    UserModule,
    VehicleModule,
    TicketModule,
    WorkshopQueueModule,
    MaterialModule,
    RepairModule,
    NotificationModule,
    AttachmentModule,
  ],
})
export class AppModule {}
```

---

# Level 5 — Core Modules Specs

Đặc tả chi tiết các module hệ thống dùng chung làm bệ đỡ cho các domain nghiệp vụ.

## 5.1 ConfigModule (Environment Management)
*   **Purpose**: Đọc, validate và cung cấp cấu hình môi trường hệ thống.
*   **Environment Variables Managed**:
    *   `PORT`: Port chạy ứng dụng (mặc định: `3000`).
    *   `DATABASE_URL`: Đường dẫn kết nối CSDL PostgreSQL (`postgresql://user:pass@host:5432/db`).
    *   `REDIS_URL`: Cấu hình kết nối Redis server phục vụ cache và queue (`redis://host:6379`).
    *   `JWT_SECRET` / `JWT_REFRESH_SECRET`: Khóa bí mật ký mã token.
    *   `S3_ENDPOINT` / `S3_BUCKET`: Cấu hình Object Storage lưu attachments.
    *   `FCM_SERVER_KEY`: Khóa API của Firebase Cloud Messaging để gửi Push.

## 5.2 DatabaseModule
*   **Purpose**: Quản lý kết nối, cấu hình transaction và pooling tới CSDL PostgreSQL.
*   **Ràng buộc kỹ thuật**: Pool size mặc định là 20 connections. Áp dụng cơ chế auto-migration khi deploy phiên bản mới.

## 5.3 AuthModule
*   **Purpose**: Cung cấp cơ chế JWT token, mã hóa bcrypt, phân quyền RBAC Guard và ABAC Policy.
*   **Providers**: `AuthService`, `JwtStrategy`, `LocalStrategy`.

## 5.4 JobQueueModule
*   **Purpose**: Nền tảng điều phối tác vụ nền BullMQ tích hợp Redis phục vụ background jobs.
*   **Jobs Handled**: Quét virus bất đồng bộ, dọn dẹp file rác định kỳ (GC), gửi email, gửi push notification leo thang.

## 5.5 RealtimeModule
*   **Purpose**: Socket.IO Gateway quản lý kết nối thời gian thực.
*   **Security**: Đọc token JWT gửi lên từ trường `auth` của kết nối để định danh thiết bị.

## 5.6 AuditModule
*   **Purpose**: Ghi nhận toàn bộ thay đổi dữ liệu nghiệp vụ quan trọng.
*   **Logic xử lý**:
    1.  Nhận payload sự kiện thay đổi dữ liệu từ Event Bus.
    2.  Tiến hành so sánh giá trị cũ và giá trị mới (old/new diff generation).
    3.  Lấy thông tin `X-Request-ID` và ID tài khoản thực hiện tác vụ (Actor).
    4.  Lưu trữ bất đồng bộ vào bảng `audit_logs` thông qua hàng chờ để tránh làm ảnh hưởng hiệu năng của API chính.

## 5.7 SharedKernelModule
*   **Purpose**: Chứa các utility, global exception filter, custom decorator, shared validator dùng chung. Cấm chứa bất kỳ logic nghiệp vụ cụ thể nào của các module.

## 5.8 CacheModule
*   **Purpose**: Quản lý bộ nhớ đệm (cấu hình Redis Client) toàn cục, giúp tối ưu hiệu năng đọc cho hệ thống.
*   **Supported Cache Types**:
    *   *Read-through cache*: Cache danh mục vật tư (`materials`), danh sách kho (`warehouses`), danh sách cầu sửa chữa (`workshop_slots`).
    *   *Write-through invalidation*: Tự động xóa hoặc làm mới cache khi có API thay đổi dữ liệu (`POST`/`PATCH`/`DELETE` liên quan).
    *   *TTL cache*: Lưu trữ các số liệu báo cáo/thống kê động trên dashboard với thời gian sống ngắn (30 phút).
    *   *Distributed cache / Blacklist*: Quản lý danh sách đen Token JWT bị thu hồi khi đăng xuất và rate limiting.

---

# Level 6 — Feature Modules Specs

Đặc tả các module quản trị và điều hành nghiệp vụ của hệ thống FixTrack.

## 6.1 UserModule
*   **Entities**: `users`, `vehicle_assignments`.
*   **Controllers**: `UsersController` (REST API).
*   **Services**: `UsersService` (CRUD người dùng, gán xe cho tài xế).
*   **Event Emitted**:
    *   `user.vehicle.assigned` (khi gán xe cho tài xế).

## 6.2 VehicleModule
*   **Entities**: `vehicles`.
*   **Controllers**: `VehiclesController`.
*   **Services**: `VehiclesService`.
*   **Logic**: Quản lý trạng thái vật lý của xe (`ACTIVE`, `BROKEN`, `REPAIRING`).

## 6.3 TicketModule
*   **Entities**: `repair_tickets`, `ticket_issues`, `inspection_records`.
*   **Controllers**: `TicketsController` (Tạo phiếu, tiếp nhận, nghiệm thu).
*   **Services**: `TicketsService` (Thực thi các nghiệp vụ phiếu và biên bản).
*   **Event Emitted**:
    *   `ticket.created` (khi lái xe tạo báo hỏng).
    *   `ticket.status_changed` (khi đổi trạng thái phiếu).

## 6.4 WorkshopQueueModule
*   **Entities**: `queue_entries`.
*   **Controllers**: `WorkshopQueueController` (Xem hàng chờ, điều phối vị trí).
*   **Services**: `WorkshopQueueService` (Logic FIFO, promote, giải phóng xe khi có cầu trống).
*   **Event Emitted**:
    *   `queue.updated` (khi hàng chờ thay đổi thứ tự).

## 6.5 MaterialModule
*   **Entities**: `materials`, `inventory_stock`, `material_requests`, `material_request_items`, `warehouses`.
*   **Controllers**: `MaterialsController`, `MaterialRequestsController`.
*   **Services**: `MaterialRequestsService` (Logic tạo yêu cầu, duyệt xuất kho, bàn giao vật tư).
*   **Event Emitted**:
    *   `material.request.created` (KTV yêu cầu).
    *   `material.request.approved` (Thủ kho duyệt xuất).
    *   `material.request.delivered` (Lái xe nhận).
    *   `material.request.completed` (KTV nhận đủ vật tư).

## 6.6 RepairModule
*   **Entities**: `repair_jobs`, `workshop_slots`, `workshop_slot_assignments`.
*   **Controllers**: `RepairJobsController`.
*   **Services**: `RepairJobsService` (Logic bắt đầu sửa chữa tại cầu, kết thúc sửa chữa).
*   **Event Emitted**:
    *   `repair.job.started` (KTV bấm bắt đầu sửa).
    *   `repair.job.completed` (KTV báo xong).

## 6.7 NotificationModule
*   **Entities**: `notifications`.
*   **Controllers**: `NotificationsController` (Đọc thông báo, đăng ký token).
*   **Services**: `NotificationsService` (Lắng nghe toàn bộ sự kiện nghiệp vụ từ EventBus để gửi Push FCM/Web Notification).
*   **Event Emitted**: Không có.

## 6.8 AttachmentModule
*   **Entities**: `attachments`.
*   **Controllers**: `AttachmentsController` (Yêu cầu link upload, xác nhận upload).
*   **Services**: `AttachmentsService` (Sinh presigned URL, dọn dẹp tệp tin mồ côi GC).
*   **Event Emitted**: Không có.

---

# Level 7 — Event Contract Registry

Định nghĩa chuẩn hóa toàn bộ sự kiện truyền tải bất đồng bộ trong hệ thống để tránh xung đột dữ liệu.

## 7.1 Quy tắc đặt tên sự kiện (Event Naming Convention)
Tất cả các tên sự kiện bắt buộc tuân theo cấu trúc phân cấp dấu chấm (Dot Notation) ngắn gọn và đồng nhất:
```text
<domain>.<entity>.<action>
```
hoặc `<domain>.<action>` nếu tên domain trùng tên thực thể chính của nghiệp vụ.
*Lưu ý*:
*   Không được lặp tên thực thể dư thừa (ví dụ: dùng `ticket.created` thay vì `ticket.repair_ticket.created`).
*   Không dùng kết hợp lai tạp giữa dấu chấm và dấu gạch dưới trong phân cấp thực thể (chọn `material.request.created` thay vì `material.request_created`).

## 7.2 Event Payload Contract Schemas
Đặc tả kiểu dữ liệu TypeScript (Interfaces) bắt buộc của các sự kiện cốt lõi:

### 1. Sự kiện `ticket.created` (Lái xe báo hỏng)
```typescript
interface TicketCreatedEvent {
  event_id: string;          // UUID v4 định danh sự kiện
  occurred_at: string;        // ISO-8601 Timestamp
  trace_id: string;           // X-Request-ID phục vụ observability
  actor_id: string;           // ID lái xe tạo phiếu
  ticket_id: string;          // ID phiếu sửa chữa vừa tạo
  vehicle_id: string;         // ID xe bị hỏng
  priority: 'LOW' | 'MEDIUM' | 'HIGH';
}
```

### 2. Sự kiện `ticket.status_changed` (Đổi trạng thái phiếu)
```typescript
interface TicketStatusChangedEvent {
  event_id: string;
  occurred_at: string;
  trace_id: string;
  actor_id: string;
  ticket_id: string;
  old_status: string;         // Enum ticket_status cũ
  new_status: string;         // Enum ticket_status mới
  version: number;            // Optimistic Lock version
}
```

### 3. Sự kiện `material.request.approved` (Thủ kho duyệt cấp phát)
```typescript
interface MaterialRequestApprovedEvent {
  event_id: string;
  occurred_at: string;
  trace_id: string;
  actor_id: string;           // ID thủ kho duyệt
  request_id: string;         // ID của material_requests
  ticket_id: string;          // ID ticket liên đới
  warehouse_id: string;       // ID kho xuất hàng
  is_partial: boolean;        // Cấp phát một phần hay toàn bộ
  approved_items: Array<{
    material_id: string;
    quantity_approved: number;
  }>;
}
```

---

# Level 8 — Cross-Cutting Technical Concerns

## 8.1 Transaction Management (Giao dịch CSDL)
Để bảo vệ tính toàn vẹn dữ liệu (ACID) khi thực hiện nhiều thao tác ghi cùng lúc:
*   Mọi Use Case liên quan đến thay đổi nhiều bảng dữ liệu bắt buộc phải dùng TypeORM `QueryRunner` để kiểm soát giao dịch thủ công:
    ```typescript
    const queryRunner = this.dataSource.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();
    try {
      // Logic 1: Lưu status_logs
      // Logic 2: Cập nhật vehicles.tinh_trang
      await queryRunner.manager.save(statusLog);
      await queryRunner.commitTransaction();
    } catch (err) {
      await queryRunner.rollbackTransaction();
      throw err;
    } finally {
      await queryRunner.release();
    }
    ```

## 8.2 Global Exception Filter
*   Toàn bộ lỗi phát sinh (HTTP Exceptions, Database errors) đều đi qua lớp lọc lỗi tập trung `GlobalExceptionFilter`.
*   Lớp này bọc lỗi về định dạng chuẩn **Error Response Standard** quy định tại Chapter 21, ghi log lỗi chi tiết kèm mã `trace_id` để tiện kiểm tra.

## 8.3 Observability & Request Tracing
*   Mọi luồng xử lý nhận request từ Controller sẽ truyền giá trị `X-Request-ID` (Trace ID) xuống tầng Service, Repository và Audit Service.
*   Khi phát sự kiện ra EventBus, `trace_id` bắt buộc phải là một thuộc tính nằm trong payload sự kiện để đảm bảo log ghi nhận từ event listener vẫn liên kết được với HTTP request ban đầu.

## 8.4 Testing Boundaries (Ranh giới kiểm thử)
Hệ thống chia việc kiểm thử thành 3 mức phân tách rõ ràng để tối ưu hóa thời gian chạy test:

```
┌────────────────────────────────────────────────────────┐
│ Presentation Layer (Contract / API Testing)            │
│   - Check API payload, HTTP status codes, headers      │
├────────────────────────────────────────────────────────┤
│ Application & Domain Layer (Unit Testing)              │
│   - Check State transitions, Guards, Business rules    │
├────────────────────────────────────────────────────────┤
│ Infrastructure Layer (Integration Testing)             │
│   - Check database transactions, S3 / Redis clients    │
└────────────────────────────────────────────────────────┘
```
*   **Unit Tests**: Tập trung kiểm thử logic nghiệp vụ tinh khiết của Domain Service và máy trạng thái (không kết nối database, mock toàn bộ Repositories).
*   **Integration Tests**: Kiểm thử sự tương tác giữa code và PostgreSQL/Redis thực tế (chạy trên container Docker dùng Testcontainers). Tập trung kiểm tra Transaction Rollback và Query logic.
*   **Contract/API Tests**: Kiểm thử NestJS End-to-End (`supertest`) để đảm bảo schema JSON đầu ra khớp 100% với đặc tả tại Chapter 21.

## 8.5 Caching Strategy (Chiến lược bộ nhớ đệm)
*   **Read-heavy data**: Danh mục phụ tùng (`materials`), danh sách cầu sửa chữa (`workshop_slots`) được lưu cache tại Redis với thời gian TTL là 30 phút.
*   **Cache Eviction**: Bất kỳ thay đổi dữ liệu nào từ APIs (`POST`/`PATCH`/`DELETE` liên quan) sẽ lập tức kích hoạt xóa cache tương ứng (Evict Cache) trong Redis.
