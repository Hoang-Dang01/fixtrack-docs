# 27 Component Architecture

**Trạng thái: Hoàn thành (Finalized - Version 1.0)**

---

## Purpose

Chương này đặc tả cấu trúc thành phần phần mềm (Component Architecture) bên trong mã nguồn NestJS của FixTrack. Tài liệu định nghĩa phân loại các cấu phần logic (Component Taxonomy), cấu trúc thư mục phân lớp lai (Hybrid Layered Directory Layout), ranh giới nghiệp vụ (Aggregate Boundaries), luồng quản lý giao dịch (Transaction Boundaries) và chiến lược kiểm thử nhằm giúp đội ngũ lập trình viên dễ dàng triển khai code chuẩn xác và nhất quán.

---

## Inputs

*   [15 Database Schema](../part_c_data_design/15_database_schema.md)
*   [18 State Machine](../part_d_application_design/18_state_machine.md)
*   [21 API Design](../part_e_backend_design/21_api_design.md)
*   [22 Backend Modules](../part_e_backend_design/22_backend_modules.md)
*   [23 Service Contracts](../part_e_backend_design/23_service_contracts.md)

---

# Level 1 — Component Taxonomy & Aggregate Boundaries

## 1.1 Phân loại Cấu phần Logic (Component Taxonomy)
Mã nguồn của FixTrack được chia tách thành các cấu phần phần mềm có trách nhiệm rõ ràng sau:

| Loại Cấu phần (Component Type) | Trách nhiệm chính (Responsibility) | Thuộc phân lớp (Layer) |
| :--- | :--- | :--- |
| **Controller** | Tiếp nhận HTTP Request, giải nén tham số và gọi xuống lớp Application. | Infrastructure |
| **Guard** | Xác thực JWT và phân quyền vai trò người dùng (Roles authorization). | Infrastructure |
| **Use Case** | Điều phối một ca nghiệp vụ cụ thể (chứa Transaction boundary). | Application |
| **Domain Service** | Chứa các quy tắc nghiệp vụ phức tạp, liên quan đến nhiều entities hoặc cần kiểm tra invariant. | Domain |
| **Repository** | Định nghĩa cổng lưu trữ (Interface) ở Domain và triển khai ghi DB (Adapter) ở Infra. | Domain (Port) / Infra (Adapter) |
| **Mapper** | Ánh xạ hai chiều giữa Domain Entity và DTO (Data Transfer Object). | Application / Infrastructure |
| **Event Listener** | Lắng nghe và xử lý các sự kiện bất đồng bộ phát hành từ Event Bus. | Infrastructure |

## 1.2 Thành phần dùng chung (Shared Components)
Hệ thống sử dụng bộ cấu phần chia sẻ xuyên suốt để nhất quán hành vi:
*   `JwtAuthGuard`: Xác thực Access Token trong JWT.
*   `RolesGuard`: Phân quyền dựa trên vai trò người dùng (`DRIVER`, `MECHANIC`, `TECH`, `INVENTORY`, `MANAGER`) đồng bộ với Chapter 21.
*   `RequestContextInterceptor`: Đính kèm `trace_id` (Request ID) vào luồng thực thi để theo dõi vết log (Ch. 26).
*   `GlobalExceptionFilter`: Bắt toàn bộ lỗi, tự động định dạng mã lỗi nghiệp vụ tùy chỉnh (Custom Business Error Code) về Response Envelope tiêu chuẩn.
*   `CacheService`: Adapter tương tác với cụm Redis Caching.
*   `EventBusAdapter`: Wrapper quanh `EventEmitter2` để phát hành và lắng nghe sự kiện bất đồng bộ.

## 1.3 Ví dụ Biên giới Aggregate (Aggregate Boundary Example)
Áp dụng mẫu thiết kế DDD, chúng ta gom nhóm các thực thể phụ thuộc vào một **Aggregate Root** duy nhất để quản lý các quy tắc ràng buộc (Invariants) toàn vẹn dữ liệu:

```
┌────────────────────────────────────────────────────────┐
│ RepairTicket Aggregate (Aggregate Root: RepairTicket)   │
│                                                        │
│  ┌────────────────┐    1..*   ┌───────────────┐        │
│  │  RepairTicket  ├──────────►│  TicketIssue  │        │
│  └───────┬────────┘           └───────────────┘        │
│          │                                             │
│          │ 0..1               0..*                     │
│          ▼                    ┌─────────────────────┐  │
│  ┌───────────────────┐        │ AttachmentReference │  │
│  │  InspectionRecord │        └─────────────────────┘  │
│  └───────────────────┘                                 │
└────────────────────────────────────────────────────────┘
```
*   **Aggregate Root**: `RepairTicket` (Phiếu sửa chữa). Mọi đột biến dữ liệu bên ngoài muốn tác động lên các thực thể con bên trong đều phải gọi thông qua method của `RepairTicket`.
*   **Thực thể phụ thuộc (Children Entities)**:
    *   `TicketIssue`: Các hạng mục hỏng hóc do tài xế báo cáo.
    *   `InspectionRecord`: Bản ghi chẩn đoán lỗi của KTV (đội kỹ thuật).
    *   `AttachmentReference`: Liên kết tới tệp đính kèm (hình ảnh/video sự cố).
*   **Quy tắc toàn vẹn (Invariants) được thực thi tại Aggregate Root**:
    *   Một phiếu sửa chữa đã ở trạng thái đóng (`closed`) hoặc đã hủy (`cancelled`) thì **không được phép** sửa đổi hay thêm mới bất kỳ hạng mục lỗi hoặc tệp đính kèm nào khác.
    *   Một phiếu sửa chữa khi khởi tạo bắt buộc phải đi kèm tối thiểu một lỗi chi tiết (`TicketIssue`).

---

# Level 2 — Hybrid Module Layout (Mô hình Module Lai)

Để tối ưu hóa thời gian triển khai, tránh sinh thư mục dư thừa đối với các nghiệp vụ đơn giản nhưng vẫn đảm bảo tính cô lập và khả năng bảo trì cho các nghiệp vụ lõi phức tạp, FixTrack áp dụng mô hình thiết kế cấu trúc thư mục lai:

## 2.1 Cấu trúc Module Đơn giản (Simple Module Structure)
*   *Áp dụng*: `NotificationModule`, `AttachmentModule`, `VehicleModule`.
*   *Mô tả*: Cấu trúc thư mục phẳng chuẩn của NestJS:
```text
src/modules/vehicle/
├── controllers/      # VehicleController
├── services/         # VehicleService
├── entities/         # Vehicle (TypeORM Entity)
├── dtos/             # CreateVehicleDto, VehicleResponseDto
└── vehicle.module.ts
```

## 2.2 Cấu trúc Module Phức tạp (Complex Module Structure)
*   *Áp dụng*: `TicketModule`, `RepairModule`, `MaterialModule`.
*   *Mô tả*: Phân rã theo Clean Architecture kết hợp DDD:
```text
src/modules/ticket/
├── domain/                      # Chỉ chứa logic nghiệp vụ thuần, không phụ thuộc framework
│   ├── entities/                # RepairTicket (Aggregate Root), TicketIssue
│   ├── value-objects/           # TicketPriority, TicketStatus
│   └── repositories/            # ITicketRepository (Ports)
├── application/                 # Điều phối luồng nghiệp vụ
│   ├── use-cases/               # CreateTicketUseCase, ApproveTicketUseCase
│   ├── dtos/                    # Request/Response DTOs
│   └── interfaces/              # IAttachmentService (giao tiếp module chéo)
├── infrastructure/              # Chi tiết triển khai công nghệ và frameworks
│   ├── persistence/             # TypeOrmTicketRepository (Adapters), Schema mappings
│   ├── controllers/             # TicketController
│   └── listeners/               # TicketEventListener (lắng nghe Event Bus)
└── ticket.module.ts
```

---

# Level 3 — Dependency Rules & Inter-Component Communication

## 3.1 Quy tắc Phụ thuộc một chiều (Dependency Rule)
Trong các module phức tạp, sự phụ thuộc mã nguồn chỉ được phép đi từ ngoài vào trong:
$$\text{Infrastructure} \longrightarrow \text{Application} \longrightarrow \text{Domain}$$
*   **Domain Layer** là trung tâm, tuyệt đối không được Import hoặc sử dụng bất kỳ thư viện nào của NestJS, TypeORM hay các adapters bên ngoài.
*   **Application Layer** chỉ phụ thuộc vào Domain Layer. Giao tiếp với hạ tầng thông qua các cổng giao tiếp (Ports / Interfaces).
*   **Infrastructure Layer** chứa TypeORM Entities, Controllers và Configs, phụ thuộc vào hai lớp bên trong.

## 3.2 Giao tiếp liên Module (Inter-Module Communication)
*   **Đồng bộ (Synchronous)**: Khi `RepairModule` cần kiểm tra trạng thái xe trong `VehicleModule`, nó bắt buộc phải gọi thông qua Service Interface công khai được khai báo tại Chapter 23 (`IVehicleService`). Nghiêm cấm việc `RepairModule` tự ý Import `VehicleRepository` để truy vấn ghi DB chéo Aggregate.
*   **Bất đồng bộ (Asynchronous)**: Khi thực thi xong một hành động nghiệp vụ làm thay đổi trạng thái Aggregate, module sở hữu phải phát đi Domain Event thông qua Event Bus theo quy chuẩn tên sự kiện dấu chấm (Dot Notation) đã chốt tại Chapter 22 & 23:
    *   Ví dụ: `ticket.created`, `ticket.status_changed`, `repair.started`, `material.request.approved`.

---

# Level 4 — Data Flow & DTO Mapping Policies

Quy trình xử lý dữ liệu đầu vào và đầu ra được kiểm soát chặt chẽ qua sơ đồ luồng dưới đây:

```
[HTTP Request] 
      │
      ▼
┌──────────────┐      NestJS ValidationPipe
│  Input DTO   ├──────────────────────────────────────────┐
└──────┬───────┘                                          │
       │ (Validate OK)                                    ▼
       ▼                                         [HTTP 400 Bad Request]
┌──────────────┐      Mapper: Input DTO -> Entity
│ Domain Entity│
└──────┬───────┘
       │ (Execute Business Logic & Persist DB)
       ▼
┌──────────────┐      Mapper: Entity -> Output DTO
│  Output DTO  │
└──────┬───────┘
       │
       ▼
[HTTP Response]
```

## 4.1 Chính sách Ánh xạ (Mapping Policies)
1.  **Request Input Validation**: Sử dụng `ValidationPipe` toàn cục của NestJS kết hợp với `class-validator` tại Input DTO để chặn đứng các request sai định dạng trước khi chạm tới Controller logic.
2.  **Khử lộ lọt Cơ sở dữ liệu (Database Leak Protection)**:
    *   > [!IMPORTANT]
        > **Cấm trả trực tiếp Entity về Client**: Domain Entity chứa các trường vật lý của DB, thông tin nhạy cảm, hoặc trường audit (`deleted_at`, `version`). 
        > Lập trình viên bắt buộc phải sử dụng các lớp **Mapper** để ánh xạ Domain Entity sang một Output DTO sạch trước khi gửi phản hồi HTTP về Client.

---

# Level 5 — Transaction Boundaries (Biên giới giao dịch)

Để đảm bảo tính nhất quán dữ liệu ACID khi một ca nghiệp vụ cập nhật nhiều bảng hoặc tương tác giữa các thực thể con trong Aggregate:

*   **Transaction Boundary thuộc phân lớp Application (Use Case Level)**: Việc mở giao dịch (Start Transaction), cam kết (Commit Transaction) và hoàn trả (Rollback Transaction) phải được quản lý ở lớp **Application Use Case** (hoặc Unit of Work pattern).
*   **Quy trình thực thi chuẩn (Execution Sequence)**:
    ```typescript
    // Ví dụ cấu trúc điều phối giao dịch nghiệp vụ trong Use Case
    async execute(command: CreateTicketCommand): Promise<TicketResponseDto> {
      return this.unitOfWork.runInTransaction(async (transactionEntityManager) => {
        // 1. Khởi tạo Aggregate Root từ dữ liệu đầu vào thông qua Factory/Constructor
        const ticket = RepairTicket.create(command.description, command.issues);
        
        // 2. Load các phụ thuộc cần thiết bằng Repository tương ứng trong Transaction
        const vehicle = await this.vehicleRepo.findById(command.vehicleId, transactionEntityManager);
        
        // 3. Thực thi nghiệp vụ thay đổi trạng thái Aggregate Root
        ticket.assignVehicle(vehicle);
        
        // 4. Lưu lại Aggregate Root xuống database (TypeORM ghi nhận lưu cả các con phụ thuộc)
        const savedTicket = await this.ticketRepo.save(ticket, transactionEntityManager);
        
        // 5. Ánh xạ kết quả sang DTO
        const responseDto = TicketMapper.toResponseDto(savedTicket);
        
        // 6. Đăng ký Domain Event để tự động phát đi SAU KHI Transaction đã COMMIT thành công
        this.eventBus.publishAfterCommit(new TicketCreatedEvent(savedTicket.id));
        
        return responseDto;
      });
    }
    ```

---

# Level 6 — Exception & Error Handling

Hệ thống quản lý lỗi tập trung thông qua lớp Exception và Global Exception Filter:

1.  **Business Exception (Lỗi Nghiệp vụ)**:
    *   Tất cả các lỗi vi phạm quy tắc nghiệp vụ (ví dụ: kho không đủ hàng, xe đang chạy không được sửa) phải ném ra lớp kế thừa từ `BusinessException`.
    *   Mỗi ngoại lệ phải đi kèm một mã lỗi nghiệp vụ tùy chỉnh (Custom Business Error Code) đã được chuẩn hóa tại Chapter 21 (ví dụ: `MAT_INSUFFICIENT_STOCK`).
2.  **Global Exception Filter (Bộ lọc lỗi toàn cầu)**:
    *   Interceptor/Filter sẽ bắt các ngoại lệ này và định dạng về cấu trúc phản hồi JSON chuẩn (Response Envelope):
        ```json
        {
          "success": false,
          "error": {
            "code": "MAT_INSUFFICIENT_STOCK",
            "message": "Số lượng vật tư trong kho không đủ để cấp phát.",
            "traceId": "trace_123456"
          }
        }
        ```

---

# Level 7 — Component Testing Strategy

Để đảm bảo chất lượng phần mềm và tính ổn định khi nâng cấp, mỗi loại cấu phần trong Component Architecture của FixTrack được áp dụng một phương pháp kiểm thử tương ứng:

| Cấu phần (Component) | Phương pháp kiểm thử (Test Type) | Thư viện sử dụng | Trọng tâm kiểm tra (Testing Focus) |
| :--- | :--- | :--- | :--- |
| **Domain Layer** (Entities, Domain Services) | **Unit Test** (Kiểm thử đơn vị thuần) | Jest | Kiểm tra tính toán logic nghiệp vụ, trạng thái máy, các ràng buộc invariants nghiệp vụ (không dùng Mock DB). |
| **Application Layer** (Use Cases) | **Unit Test với Mocking** | Jest | Kiểm tra luồng điều phối nghiệp vụ, hành động đóng/mở transaction, đảm bảo Event được phát đi đúng điều kiện. |
| **Infrastructure Layer** (Repositories DB) | **Integration Test** (Kiểm thử tích hợp) | Jest + Testcontainers (PostgreSQL) | Kiểm tra câu lệnh SQL thực tế, TypeORM mapper, tính đúng đắn khi ghi đè hoặc quan hệ bảng. |
| **Controllers / APIs** | **E2E Test** (Kiểm thử đầu cuối) | Jest + Supertest | Kiểm tra phân quyền truy cập (Guards), validate dữ liệu đầu vào (Pipes), mã lỗi HTTP trả về. |
