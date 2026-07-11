# 06 Workflow Analysis

## Purpose
Chương này phân tích chi tiết quy trình nghiệp vụ (Business Workflow) số hóa từ lúc phát hiện sự cố phương tiện cho tới khi phương tiện được nghiệm thu và hoạt động bình thường trở lại. Tài liệu làm rõ sự phối hợp tác nghiệp giữa 4 bộ phận chính: Lái xe, Đội cơ giới, Đội kỹ thuật và Kho vật tư trên ứng dụng di động (Mobile App).

## Questions Answered
- Quy trình sửa chữa phương tiện trải qua các bước cụ thể nào?
- Sự phối hợp và trách nhiệm bàn giao vật tư giữa Kho, Lái xe và Kỹ thuật viên (KTV) được kiểm soát như thế nào để tránh thất thoát?
- Hệ thống xử lý các tình huống đặc biệt như xưởng hết chỗ (full slot) hoặc thiếu vật tư như thế nào?

## Inputs
- Sơ đồ quy trình nghiệp vụ gốc từ tệp `quy_trinh_sua_chua_phuong_tien_v2.pdf`.
- Quy tắc nghiệp vụ cốt lõi: `BR-01` (Quyền tạo phiếu của tài xế được phân công), `BR-02` (Luồng bàn giao vật tư khép kín), `BR-03` (Báo nhiều lỗi trong một phiếu).
- Sơ đồ Business Flow và Sequence Diagram đã được kết xuất tại thư mục `docs/images/`.

---

## Content

### Level 1 — Summary
Quy trình sửa chữa phương tiện được số hóa hoàn toàn nhằm thay thế các phương thức trao đổi thủ công (giấy tờ, chat, gọi điện) bằng luồng xử lý thời gian thực trên Mobile App. Quy trình bắt đầu khi Lái xe phát hiện sự cố và tạo phiếu báo lỗi. Đội cơ giới tiếp nhận và kiểm tra sơ bộ nhằm lọc nhanh các lỗi nhỏ có thể tự xử lý để trả xe chạy tiếp. Nếu cần can thiệp chuyên sâu, xe được chuyển sang Đội kỹ thuật. 

Tại xưởng, KTV chẩn đoán chi tiết và lập đơn yêu cầu vật tư gửi cho Kho. Kho duyệt và xuất vật tư cho Lái xe. Lái xe nhận vật tư vật lý mang về xưởng bàn giao cho KTV thực hiện sửa chữa. Sau khi sửa xong, Đội cơ giới tiến hành nghiệm thu thực tế và xác nhận đóng phiếu trên hệ thống để đưa xe trở lại trạng thái hoạt động bình thường.

### Level 2 — Breakdown
Quy trình nghiệp vụ chi tiết bao gồm 7 bước chính sau:

#### Bước 1: Lái xe báo hư hỏng (Driver)
- **Hành động**: Lái xe phát hiện sự cố (trong quá trình chạy hoặc lúc kiểm tra đầu ca). Lái xe đăng nhập app, chọn phương tiện đang được phân công cho mình (bắt buộc theo `BR-01`).
- **Nội dung khai báo**: Nhập mô tả lỗi (hỗ trợ khai báo nhiều hạng mục lỗi đồng thời theo `BR-03`), đính kèm hình ảnh/video thực tế tại hiện trường, chọn mức độ nghiêm trọng và gửi yêu cầu.
- **Hệ thống**: Tạo phiếu sửa chữa với trạng thái ban đầu là `REPORTED` (Đã báo lỗi) và gửi thông báo (Push Notification) tới Đội cơ giới.

#### Bước 2: Đội cơ giới kiểm tra sơ bộ (Mechanic)
- **Hành động**: Nhân viên Cơ giới tiếp nhận xe tại bãi, tiến hành kiểm tra nhanh ban đầu.
- **Quyết định**:
  - *Nếu là lỗi rất nhẹ hoặc báo sai*: Cơ giới từ chối phiếu (Status: `REJECTED`), cập nhật lý do và trả xe tiếp tục hoạt động.
  - *Nếu cần sửa chữa chuyên sâu*: Cơ giới xác nhận chuyển xưởng (Status: `INSPECTING`).

#### Bước 3: Đội kỹ thuật khám xe & Đưa ra phán quyết (Technical)
- **Hành động**: Xe được di chuyển vào khu vực xưởng. KTV chẩn đoán chuyên sâu để xác định chính xác các lỗi cần sửa.
- **Xử lý hàng chờ (Queue)**: Nếu toàn bộ slot sửa chữa trong xưởng đang bị đầy (Full), hệ thống tự động đưa xe vào hàng chờ (Status: `WAITING_QUEUE`). Khi có slot trống (KTV bấm hoàn tất xe trước), hệ thống thông báo điều phối xe tiếp theo vào vị trí để KTV bắt đầu xử lý.
- **Phán quyết**: KTV xác định phương án sửa chữa và danh sách linh kiện, phụ tùng cần thay thế.

#### Bước 4: Lên danh sách vật tư yêu cầu cấp phát (Technical)
- **Hành động**: KTV tạo Đơn yêu cầu vật tư (Material Request) trực tiếp trên app bám theo mã phiếu sửa chữa, chỉ định rõ tên phụ tùng, mã SKU và số lượng cần thiết. Gửi yêu cầu phê duyệt tới Kho.

#### Bước 5: Kho phê duyệt và bàn giao vật tư khép kín (Inventory & Driver & Technical)
Để kiểm soát chặt chẽ vật tư, hệ thống áp dụng luồng bàn giao 3 bên theo quy tắc `BR-02`:
1. **Thủ kho xuất kho**: Thủ kho tiếp nhận yêu cầu trên app, kiểm tra số lượng tồn kho. Nếu đủ, thủ kho bấm duyệt, chuẩn bị vật tư vật lý và xác nhận "Đã xuất kho" (Status: `WAITING_PARTS`).
2. **Lái xe nhận vật tư**: Lái xe di chuyển đến kho để nhận phụ tùng vật lý. Thủ kho giao hàng, Lái xe bấm xác nhận "Đã nhận vật tư từ kho" trên app.
3. **Bàn giao cho KTV**: Lái xe mang vật tư về xưởng bàn giao cho KTV. KTV kiểm tra đúng chủng loại/số lượng và bấm xác nhận "Kỹ thuật đã nhận đủ vật tư từ lái xe". Hệ thống tự động chuyển trạng thái phiếu sang sẵn sàng sửa chữa.

#### Bước 6: Tiến hành sửa chữa (Technical)
- **Hành động**: KTV bấm "Bắt đầu sửa chữa" trên app (Status: `REPAIRING`). Hệ thống bắt đầu ghi nhận thời gian sửa chữa thực tế. KTV tiến hành thay thế phụ tùng, khắc phục lỗi.
- **Sửa xong**: KTV bấm xác nhận "Đã sửa xong" trên ứng dụng (Status: `COMPLETED`).

#### Bước 7: Nghiệm thu và bàn giao phương tiện (Mechanic & Technical)
- **Hành động**: Nhân viên Cơ giới tiến hành chạy thử và nghiệm thu kỹ thuật thực tế của phương tiện.
- **Quyết định**:
  - *Nghiệm thu đạt*: Cơ giới bấm "Xác nhận nghiệm thu thành công" trên app. Phiếu sửa chữa chuyển sang trạng thái cuối cùng là đóng (`CLOSED`). Phương tiện chuyển về trạng thái hoạt động bình thường (`ACTIVE`).
  - *Nghiệm thu không đạt*: Cơ giới cập nhật lý do chưa đạt, yêu cầu xưởng khắc phục lại. Phiếu quay ngược về trạng thái sửa chữa (`REPAIRING`).

---

### Level 3 — Technical Detail

#### 1. Sơ đồ trạng thái của Phiếu sửa chữa (Ticket State Transitions)
Luồng chuyển đổi trạng thái của `Repair Ticket` được kiểm soát chặt chẽ thông qua các API tương tác của người dùng:

```mermaid
stateDiagram-v2
    [*] --> REPORTED : Driver báo lỗi
    REPORTED --> INSPECTING : Cơ giới tiếp nhận kiểm tra
    INSPECTING --> REJECTED : Cơ giới từ chối (Lỗi quá nhẹ)
    INSPECTING --> WAITING_QUEUE : Cần sửa nhưng xưởng full slot
    WAITING_QUEUE --> INSPECTING : Có slot trống trong xưởng
    INSPECTING --> WAITING_PARTS : KTV yêu cầu vật tư & Kho đã duyệt xuất
    WAITING_PARTS --> REPAIRING : KTV xác nhận đã nhận đủ vật tư từ lái xe
    REPAIRING --> COMPLETED : KTV xác nhận sửa xong
    COMPLETED --> CLOSED : Cơ giới nghiệm thu đạt & đóng phiếu
    COMPLETED --> REPAIRING : Nghiệm thu không đạt (yêu cầu sửa lại)
    REJECTED --> [*]
    CLOSED --> [*]
```

#### 2. Biểu đồ nghiệp vụ tổng thể (Business Flow Diagram)
Sơ đồ dưới đây biểu diễn luồng hoạt động nghiệp vụ tổng quát được số hóa:

```mermaid
graph TD
    classDef actor fill:#e8f5e9,stroke:#2e7d32,stroke-width:1px;
    classDef process fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,rx:5px,ry:5px;
    classDef decision fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,rx:5px,ry:5px;
    classDef doc fill:#f3e5f5,stroke:#6a1b9a,stroke-width:1px;

    Step1["🚗 1. Phát hiện & Báo hỏng<br>(Lái xe)"]:::process
    Doc1["📄 Phiếu báo hỏng (REPORTED)"]:::doc
    Step1 --> Doc1

    Doc1 --> Step2["🔍 2. Kiểm tra sơ bộ<br>(Cơ giới)"]:::process
    Dec2{"Lỗi cần vào xưởng?"}:::decision
    Step2 --> Dec2

    Dec2 -- Không --> EndNormal["Trả xe hoạt động<br>(CLOSED / REJECTED)"]:::process
    Dec2 -- Có --> Step3["🛠️ 3. Phân tích & Phán quyết<br>(Kỹ thuật)"]:::process

    Dec3{"Xưởng còn slot?"}:::decision
    Step3 --> Dec3

    Dec3 -- Không --> StepQueue["⏳ Hàng chờ xưởng<br>(WAITING_QUEUE)"]:::process
    StepQueue -->|Có slot trống| Step3

    Dec3 -- Có --> Step4["📝 4. Lập đơn yêu cầu vật tư<br>(Kỹ thuật)"]:::process
    Doc4["📄 Đơn yêu cầu vật tư (PENDING)"]:::doc
    Step4 --> Doc4

    Doc4 --> Step5["📦 5. Cấp phát vật tư<br>(Kho)"]:::process
    Dec5{"Kho đủ hàng?"}:::decision
    Step5 --> Dec5

    Dec5 -- Không --> StepQueueLack["⏳ Nhường slot, vào hàng chờ<br>(WAITING_QUEUE - Lacking Materials)"]:::process
    StepQueueLack -->|Nhập kho bổ sung| Step5

    Dec5 -- Có --> Step6["🔧 6. Bàn giao & Sửa chữa<br>(Kỹ thuật & Lái xe)"]:::process
    Doc6["📝 Nhật ký sửa chữa (REPAIRING)"]:::doc
    Step6 --> Doc6

    Step6 -->|Dư/Thiếu vật tư| StepPartAdjust["🔄 Trả vật tư thừa / Yêu cầu thêm"]:::process
    StepPartAdjust --> Step6

    Doc6 --> Step7["✅ 7. Nghiệm thu & Bàn giao<br>(Cơ giới)"]:::process
    Dec7{"Nghiệm thu đạt?"}:::decision
    Step7 --> Dec7

    Dec7 -- Không --> Step6
    Dec7 -- Có --> EndActive["🚗 Xe hoạt động bình thường<br>(CLOSED)"]:::process

    %% Notifications side track
    Step1 -.->|Firebase FCM| Notify["🔔 8. Thông báo & Theo dõi<br>(Tự động)"]:::actor
    Step2 -.->|Firebase FCM| Notify
    Step4 -.->|Firebase FCM| Notify
    Step5 -.->|Firebase FCM| Notify
    Step7 -.->|Firebase FCM| Notify
```

#### 3. Luồng tuần tự tạo phiếu sửa chữa (Sequence Flow)
Tương tác API giữa Mobile App (Client), Backend và Database khi khởi tạo luồng báo lỗi:

```mermaid
sequenceDiagram
    autonumber
    actor Driver as Lái xe (Driver)
    participant App as Mobile App (Client)
    participant API as Backend Service (API)
    participant DB as PostgreSQL DB
    participant Queue as Redis (BullMQ Queue)
    participant Worker as Background Worker
    participant FCM as Firebase FCM Service
    actor Fleet as Đội Cơ giới (Fleet Staff)

    Driver->>App: Mở chức năng Báo hỏng, chụp ảnh & gửi
    App->>API: POST /api/v1/repair-tickets (Payload: vehicle_id, description, severity, photos)
    
    rect rgb(240, 248, 255)
        note over API, DB: Kiểm tra nghiệp vụ (Sync Validation)
        API->>DB: Query active vehicle assignment for Driver
        DB-->>API: Active assignment info (Check: Driver owns vehicle)
        API->>DB: Query active tickets for Vehicle
        DB-->>API: Count of active tickets (Check: BR-TICKET-03 - No duplicate ticket)
    end

    alt Kiểm tra lỗi (Validation fails)
        API-->>App: Return 400 Bad Request (Error: MAT_DUPLICATE_TICKET)
        App-->>Driver: Hiển thị thông báo lỗi
    else Kiểm tra đạt (Validation passes)
        API->>DB: Insert repair_tickets (status = 'REPORTED', generated ma_phieu)
        DB-->>API: Created Ticket details
        API->>DB: Insert status_logs & audit_logs (transactional sync write)
        DB-->>API: Success
        API-->>App: Return 201 Created (Ticket Info)
        App-->>Driver: Hiển thị báo lỗi thành công, trạng thái xe: BROKEN
        
        rect rgb(255, 245, 238)
            note over API, Fleet: Tác vụ chạy nền không đồng bộ (Async Notifications)
            API->>Queue: Enqueue Notification Job (ticket_created, user_id = Fleet)
            Queue-->>API: Acknowledge (Job ID)
            Queue->>Worker: Dequeue Notification Job
            Worker->>FCM: Request Push Notification API
            FCM-->>Worker: Status 200 OK
            Worker->>Fleet: Push Notification: "Có xe báo hỏng mới cần kiểm tra!"
        end
    end
```


---

## Outputs
- Tệp đặc tả hoàn chỉnh: [06_workflow_analysis.md](file:///c:/Users/Legion/Desktop/IT/SNP/.agents/skills/system_engineering_copilot/deliverables/current/sss/part_b_requirement_analysis/06_workflow_analysis.md)
