# 21 API Design

**Trạng thái: Hoàn thành (Finalized - Version 1.0)**

---

## Purpose

Chương này đặc tả thiết kế API của hệ thống FixTrack theo chuẩn RESTful API. Tài liệu định nghĩa endpoint contract giữa Frontend/Mobile Client và Backend Server, bao gồm request schema, response schema, authentication mechanism, authorization rules, error handling, versioning strategy và API conventions.

Tài liệu là nền tảng cho:
*   **Backend implementation** (NestJS Controllers / DTOs / Guards / Pipes).
*   **Frontend/Mobile integration** (React / React Native API clients / Hooks / State Management).
*   **QA testing** (Postman / Integration tests / Automated API Testing).
*   **API documentation generation** (OpenAPI / Swagger specs).

---

## Questions Answered

*   Hệ thống expose những API nào cho các ứng dụng Client?
*   Cơ chế định danh, URL structure và API naming convention hoạt động ra sao?
*   Cơ chế xác thực (Authentication) và phân quyền (Authorization) bằng JWT được kiểm soát thế nào?
*   Cấu trúc chuẩn của phản hồi thành công (Success Response) và phản hồi lỗi (Error Response) là gì?
*   Các giải pháp kỹ thuật nâng cao: Idempotency, Optimistic Locking, Bulk API Partial Success và Real-time WebSockets được thiết kế ra sao?

---

## Inputs

*   [05 User Roles](../part_a_business_foundation/05_user_roles.md)
*   [07 Functional Requirements](../part_b_requirement_analysis/07_functional_requirements.md)
*   [09 Business Rules](../part_b_requirement_analysis/09_business_rules.md)
*   [10 Use Cases](../part_b_requirement_analysis/10_use_cases.md)
*   [12 Logical DB Design & ERD](../part_c_data_design/12_erd.md)
*   [14 CRUD Matrix](../part_c_data_design/14_crud_matrix.md)
*   [15 Database Schema](../part_c_data_design/15_database_schema.md)
*   [18 State Machine](../part_d_application_design/18_state_machine.md)
*   [19 Notification Flow](../part_d_application_design/19_notification_flow.md)
*   [20 Attachment Design](../part_d_application_design/20_attachment_design.md)

---

# Level 1 — API Architecture Overview

## 1.1 API Style & Base URL
Hệ thống FixTrack sử dụng kiến trúc RESTful API cho các giao tiếp Request-Response đồng bộ và WebSocket cho các luồng sự kiện Real-time bất đồng bộ.
*   **Protocol**: HTTPS (REST API) & WSS (WebSocket).
*   **Encoding**: UTF-8.
*   **Data Format**: JSON (JavaScript Object Notation).
*   **Base URL**: `/api/v1`

Ví dụ:
```http
POST https://api.fixtrack.com/api/v1/auth/login
GET https://api.fixtrack.com/api/v1/repair-tickets
```

## 1.2 API Naming Convention
Để đảm bảo tính nhất quán trên toàn hệ thống, các nhà phát triển Backend và Frontend bắt buộc phải tuân theo quy chuẩn đặt tên sau:
*   **Resource Paths**: Sử dụng danh từ số nhiều (Plural Nouns). Không dùng động từ trong đường dẫn tĩnh.
    *   *Good*: `GET /api/v1/repair-tickets`
    *   *Bad*: `GET /api/v1/get-repair-tickets`, `GET /api/v1/repair-ticket`
*   **URL Paths**: Sử dụng định dạng `kebab-case` cho các đường dẫn tĩnh và `camelCase` hoặc `uuid` cho các tham số động.
    *   *Good*: `PATCH /api/v1/material-requests/:id/approve`
    *   *Bad*: `PATCH /api/v1/material_requests/:id/Approve`
*   **JSON Payloads**: Sử dụng định dạng `snake_case` cho toàn bộ thuộc tính (Request body, Response body, Query Parameters).
    *   *Good*: `{"vehicle_id": "...", "assigned_at": "..."}`
    *   *Bad*: `{"vehicleId": "...", "AssignedAt": "..."}`
*   **Enum Values**: Sử dụng định dạng viết hoa cách nhau bằng dấu gạch dưới (`UPPER_SNAKE_CASE`).
    *   *Good*: `REPORTED`, `WAITING_QUEUE`, `HIGH`, `DRIVER`
    *   *Bad*: `reported`, `WaitingQueue`, `high`, `Driver`

## 1.3 Authentication & Token Lifecycle
Hệ thống áp dụng cơ chế xác thực **Bearer JWT (JSON Web Token)** thông qua Header của Request:
```http
Authorization: Bearer <access_token>
```

### Token Lifecycle & Configuration
Để cân bằng giữa trải nghiệm người dùng và tính bảo mật, vòng đời của token được cấu hình như sau:

| Token Type | Storage Location | TTL (Time-To-Live) | Storage Backend / Management |
| :--- | :--- | :--- | :--- |
| **Access Token** | Client Memory (React State) / HTTP Header | **15 phút** | Không lưu trữ trạng thái ở DB (Stateless) |
| **Refresh Token** | Secure HttpOnly Cookie (Web) / Secure Storage (Mobile) | **7 ngày** | Lưu ở **Redis** (Stateful) để hỗ trợ Revocation |

### Token Refresh Flow & Revocation Logic
1.  **Refresh Flow**: Khi Access Token hết hạn (nhận mã lỗi `401 Unauthorized` kèm code `TOKEN_EXPIRED`), Client sẽ gọi API `POST /api/v1/auth/refresh` gửi kèm Refresh Token nằm trong HttpOnly cookie hoặc request body. Backend xác thực Refresh Token trong Redis, nếu hợp lệ sẽ cấp phát cặp Access/Refresh Token mới và ghi đè lên Redis.
2.  **Revocation Logic (Đăng xuất / Thu hồi)**: Khi người dùng bấm Đăng xuất (`POST /api/v1/auth/logout`) hoặc khi Quản trị viên khóa tài khoản, Refresh Token tương ứng sẽ bị xóa lập tức khỏi Redis. Mọi request refresh tiếp theo sử dụng token đó sẽ bị từ chối.

## 1.4 Response Envelope Standard
Tất cả các phản hồi REST API từ hệ thống FixTrack đều được bọc trong một "Envelope" chuẩn hóa để thuận tiện cho việc xử lý chung tại Client (Interceptors).

### Success Response Standard
```json
{
  "success": true,
  "data": {},
  "meta": {
    "timestamp": "2026-06-28T10:28:13Z"
  }
}
```
*Đối với API phân trang (Pagination):*
```json
{
  "success": true,
  "data": [],
  "meta": {
    "page": 1,
    "limit": 20,
    "total_records": 150,
    "total_pages": 8,
    "timestamp": "2026-06-28T10:28:13Z"
  }
}
```

### Error Response Standard
```json
{
  "success": false,
  "error": {
    "code": "TICKET_NOT_FOUND",
    "message": "Phiếu sửa chữa không tồn tại trên hệ thống hoặc đã bị xóa.",
    "details": [
      {
        "field": "ticket_id",
        "issue": "Must be a valid UUID format"
      }
    ]
  }
}
```

## 1.5 Security Standards
*   **HTTPS Only**: Toàn bộ lưu lượng truyền tải bắt buộc phải mã hóa TLS 1.3.
*   **CORS Whitelist**: Cấu hình CORS nghiêm ngặt, chỉ cho phép các domain được định nghĩa trước (ví dụ: `*.fixtrack.com`) truy cập API.
*   **Request Validation**: Áp dụng NestJS `ValidationPipe` kết hợp với `class-validator` để validate dữ liệu đầu vào nghiêm ngặt trước khi đi vào logic nghiệp vụ.
*   **Payload Sanitization**: Xóa bỏ các ký tự độc hại (XSS prevention) và chặn SQL Injection thông qua PostgreSQL Parameterized Queries (TypeORM/Prisma).
*   **Rate Limiting**: Giới hạn tần suất gọi API để bảo vệ hệ thống khỏi các cuộc tấn công DDoS hoặc brute-force.
*   **Audit Logging**: Mọi thao tác làm thay đổi dữ liệu hoặc trạng thái quan trọng bắt buộc phải được ghi vết tự động vào bảng `audit_logs`.

## 1.6 Versioning Policy
Để quản lý sự tiến hóa của hệ thống API và tránh gây ảnh hưởng đến các ứng dụng Client đang vận hành (Mobile App, Web Portal):
*   **Non-breaking Changes** (Thêm endpoint mới, thêm trường tùy chọn trong response): Sẽ không làm tăng phiên bản API. Hệ thống vẫn giữ nguyên ở `/api/v1`.
*   **Breaking Changes** (Xóa trường, đổi kiểu dữ liệu, thay đổi cấu trúc URL tĩnh, sửa đổi logic nghiệp vụ cốt lõi): Bắt buộc phải nâng cấp phiên bản API lên `/api/v2`, `/api/v3`.
*   **Deprecation Policy**: Khi một phiên bản cũ bị thay thế bởi phiên bản mới, hệ thống sẽ duy trì phiên bản cũ song song trong tối thiểu **90 ngày** trước khi ngừng hỗ trợ hoàn toàn. Khi đó, header phản hồi sẽ đính kèm thông báo cảnh báo deprecation:
    ```http
    Deprecation: @1782782945
    Link: <https://api.fixtrack.com/docs/v1-deprecation>; rel="deprecation"
    ```

## 1.7 Observability & Distributed Tracing (Request ID)
Nhằm phục vụ giám sát vận hành (Observability), gỡ lỗi và truy vết log hệ thống (Tracing):
*   Mỗi request gửi lên hệ thống được khuyến khích đính kèm Header `X-Request-ID` với giá trị là một UUID v4 duy nhất được sinh từ phía Client. Nếu Client không gửi, API Gateway/Backend Server sẽ tự động sinh và đính kèm `X-Request-ID` cho request đó.
*   Header `X-Request-ID` sẽ được trả về trong mọi phản hồi của API:
    ```http
    X-Request-ID: req_a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d
    ```
*   Giá trị `X-Request-ID` này sẽ được tự động ghi lại trong tất cả các dòng log hệ thống (application logs) và được ghi vào trường tương ứng của bảng `audit_logs` để dễ dàng tra cứu toàn bộ luồng xử lý của một request duy nhất khi xảy ra sự cố.

---

# Level 2 — API Module Breakdown

Hệ thống FixTrack được phân rã thành **9 Module API chức năng** dưới đây:

1.  **Authentication APIs** (4 endpoints)
2.  **User APIs** (3 endpoints)
3.  **Vehicle APIs** (4 endpoints)
4.  **Repair Ticket APIs** (9 endpoints)
5.  **Queue APIs** (3 endpoints)
6.  **Material APIs** (10 endpoints)
7.  **Repair Execution APIs** (3 endpoints)
8.  **Notification APIs** (3 endpoints)
9.  **Attachment APIs** (3 endpoints)

## 2.1 API Matrix Summary
Bảng tổng hợp nhanh toàn bộ Endpoint của hệ thống giúp QA và Dev dễ dàng tra cứu nhanh:

| API ID | Method | Endpoint Path | Module | Roles Allowed | Purpose / Action |
| :--- | :---: | :--- | :---: | :--- | :--- |
| **API-AUTH-01** | POST | `/auth/login` | Auth | Public | Đăng nhập hệ thống bằng Mã nhân viên và Mật khẩu |
| **API-AUTH-02** | POST | `/auth/logout` | Auth | All Roles | Đăng xuất, hủy phiên hoạt động |
| **API-AUTH-03** | POST | `/auth/refresh` | Auth | All Roles | Làm mới Access Token bằng Refresh Token |
| **API-AUTH-04** | GET | `/auth/me` | Auth | All Roles | Lấy thông tin tài khoản đang đăng nhập |
| **API-USER-01** | GET | `/users` | User | MANAGER | Lấy danh sách nhân viên phân trang & lọc |
| **API-USER-02** | GET | `/users/:id` | User | MANAGER, Self | Lấy chi tiết thông tin một nhân viên |
| **API-USER-03** | PATCH | `/users/:id` | User | MANAGER, Self | Cập nhật thông tin nhân viên |
| **API-VEH-01** | GET | `/vehicles` | Vehicle | DRIVER, MECHANIC, TECH, INVENTORY, MANAGER | Lấy danh sách phương tiện |
| **API-VEH-02** | GET | `/vehicles/:id` | Vehicle | DRIVER, MECHANIC, TECH, INVENTORY, MANAGER | Chi tiết phương tiện |
| **API-VEH-03** | POST | `/vehicles` | Vehicle | MANAGER | Đăng ký phương tiện mới vào hệ thống |
| **API-VEH-04** | PATCH | `/vehicles/:id` | Vehicle | MANAGER | Cập nhật thông tin phương tiện |
| **API-TICKET-01**| POST | `/repair-tickets` | Ticket | DRIVER, MANAGER | Tạo phiếu báo hỏng xe (Lái xe báo) |
| **API-TICKET-02**| GET | `/repair-tickets` | Ticket | All Roles (ABAC Scope) | Lọc & Lấy danh sách phiếu sửa chữa |
| **API-TICKET-03**| GET | `/repair-tickets/:id` | Ticket | All Roles (ABAC Scope) | Lấy chi tiết phiếu sửa chữa |
| **API-TICKET-04**| PATCH | `/repair-tickets/:id` | Ticket | MECHANIC, TECH, MANAGER | Cập nhật thông tin phiếu (Lost Update protection) |
| **API-TICKET-05**| POST | `/repair-tickets/:id/accept` | Ticket | MECHANIC | Tiếp nhận kiểm tra sơ bộ |
| **API-TICKET-06**| POST | `/repair-tickets/:id/reject` | Ticket | MECHANIC | Từ chối báo hỏng (Lỗi quá nhẹ) |
| **API-TICKET-07**| POST | `/repair-tickets/:id/route-queue`| Ticket | MECHANIC | Đẩy xe vào hàng chờ sửa chữa của xưởng |
| **API-TICKET-08**| POST | `/repair-tickets/:id/approve-inspection`| Ticket | MECHANIC | Nghiệm thu ĐẠT và đóng phiếu |
| **API-TICKET-09**| POST | `/repair-tickets/:id/reject-inspection`| Ticket | MECHANIC | Nghiệm thu KHÔNG ĐẠT, yêu cầu sửa lại |
| **API-QUEUE-01** | GET | `/queue` | Queue | All Roles | Xem danh sách hàng chờ hiện tại |
| **API-QUEUE-02** | POST | `/queue/promote` | Queue | MANAGER | Điều phối thứ tự ưu tiên trong hàng chờ |
| **API-QUEUE-03** | PATCH | `/queue/:id` | Queue | System Worker (Non-human actor), MANAGER | Cập nhật trạng thái dòng hàng chờ |
| **API-MAT-01** | GET | `/materials` | Material | TECH, INVENTORY, MANAGER | Lấy danh sách danh mục vật tư |
| **API-MAT-02** | GET | `/inventory` | Material | TECH, INVENTORY, MANAGER | Xem tồn kho khả dụng tại các kho |
| **API-MAT-03** | POST | `/material-requests` | Material | TECH | Yêu cầu cấp vật tư phục vụ sửa chữa |
| **API-MAT-04** | GET | `/material-requests` | Material | TECH, INVENTORY, MANAGER | Danh sách yêu cầu cấp vật tư |
| **API-MAT-05** | GET | `/material-requests/:id` | Material | TECH, INVENTORY, MANAGER | Chi tiết yêu cầu cấp vật tư |
| **API-MAT-06** | PATCH | `/material-requests/:id/approve`| Material | INVENTORY | Duyệt xuất kho (một phần hoặc toàn bộ) |
| **API-MAT-07** | PATCH | `/material-requests/:id/reject`| Material | INVENTORY | Từ chối yêu cầu cấp vật tư |
| **API-MAT-08** | POST | `/material-requests/bulk-approve`| Material | INVENTORY | Duyệt hàng loạt yêu cầu vật tư |
| **API-MAT-09** | POST | `/material-requests/:id/confirm-delivery`| Material | DRIVER | Lái xe xác nhận nhận phụ tùng tại kho |
| **API-MAT-10** | POST | `/material-requests/:id/confirm-handover`| Material | TECH | KTV xác nhận nhận bàn giao tại xưởng |
| **API-EXEC-01** | POST | `/repair-jobs/:id/start` | Repair | TECH | Bấm bắt đầu thực hiện sửa chữa |
| **API-EXEC-02** | POST | `/repair-jobs/:id/complete`| Repair | TECH | Bấm hoàn thành sửa chữa |
| **API-EXEC-03** | GET | `/repair-jobs` | Repair | TECH, MANAGER | Lấy danh sách lịch sử sửa chữa |
| **API-NOTIF-01** | GET | `/notifications` | Notification| All Roles | Lấy hộp thư thông báo cá nhân |
| **API-NOTIF-02** | PATCH | `/notifications/:id/read`| Notification| All Roles | Đánh dấu thông báo đã đọc |
| **API-NOTIF-03** | POST | `/devices/register-token`| Notification| All Roles | Đăng ký Device Token nhận Push Notification |
| **API-ATTACH-01**| POST | `/attachments/presigned-url`| Attachment | All Roles | Yêu cầu cấp presigned URL để upload file |
| **API-ATTACH-02**| GET | `/attachments/:id/access-url`| Attachment | All Roles | Lấy link tạm thời để hiển thị/tải file |
| **API-ATTACH-03**| POST | `/attachments/confirm-upload`| Attachment | All Roles | Xác nhận client đã upload thành công lên S3 |

### Chi tiết Phân quyền Dữ liệu Dựa trên Thuộc tính (ABAC Scope)
Đối với các endpoint danh sách hoặc chi tiết của Phiếu sửa chữa (`API-TICKET-02`, `API-TICKET-03`), hệ thống áp dụng kiểm soát truy cập như sau:
*   **DRIVER**: Chỉ được phép đọc các phiếu do chính mình tạo ra (`driver_id = current_user_id`).
*   **MECHANIC**: Được phép đọc toàn bộ các phiếu trong trạng thái điều phối (`REPORTED`, `INSPECTING`) hoặc các phiếu mình trực tiếp nghiệm thu chạy thử.
*   **TECH**: Chỉ được phép đọc các phiếu sửa chữa mà mình được phân công thực hiện các công việc sửa chữa (`repair_jobs.technician_id = current_user_id`).
*   **INVENTORY**: Chỉ được phép đọc thông tin phiếu sửa chữa có gắn các yêu cầu cấp phát vật tư liên quan tới kho mà mình quản lý.
*   **MANAGER**: Có quyền đọc toàn bộ dữ liệu phiếu sửa chữa trên toàn hệ thống (Global Scope).

---

# Level 3 — Detailed Endpoint Specifications

Phần này đặc tả chi tiết 10 Endpoint cốt lõi và phức tạp nhất của hệ thống FixTrack.

---

## 3.1 API-AUTH-01: Đăng nhập hệ thống

*   **Endpoint**: `POST /api/v1/auth/login`
*   **Purpose**: Người dùng đăng nhập hệ thống bằng mã số nhân viên và mật khẩu.
*   **Roles**: `Public`
*   **Request Headers**:
    ```http
    Content-Type: application/json
    ```
*   **Request Body**:
    ```json
    {
      "employee_code": "EMP-001",
      "password": "PasswordSecure123"
    }
    ```
*   **Response (200 OK)**:
    ```json
    {
      "success": true,
      "data": {
        "access_token": "eyJhbGciOiJIUzI1NiIsIn...",
        "user": {
          "id": "e3b0c442-98fc-11ee-b9d1-0242ac120002",
          "employee_code": "EMP-001",
          "full_name": "Nguyen Van A",
          "phone": "0987654321",
          "role": "DRIVER",
          "status": "ACTIVE"
        }
      },
      "meta": {
        "timestamp": "2026-06-28T10:28:13Z"
      }
    }
    ```
*   **Errors**:

    | HTTP Status | Error Code | Meaning / Reason |
    | :--- | :--- | :--- |
    | `400 Bad Request` | `AUTH_VALIDATION_FAILED` | Định dạng mã nhân viên hoặc mật khẩu không hợp lệ |
    | `401 Unauthorized` | `AUTH_INVALID_CREDENTIALS` | Sai mã nhân viên hoặc mật khẩu |
    | `403 Forbidden` | `AUTH_ACCOUNT_INACTIVE` | Tài khoản đang bị khóa (INACTIVE) |

*   **Business Rules**:
    *   Xác thực định dạng mã nhân viên (ví dụ: tiền tố `EMP-` đi kèm chuỗi số).
    *   Tự động kiểm tra trạng thái hoạt động của nhân sự trước khi cho phép đăng nhập.

---

## 3.2 API-TICKET-01: Tạo phiếu báo hỏng xe

*   **Endpoint**: `POST /api/v1/repair-tickets`
*   **Purpose**: Lái xe (hoặc Manager) tạo phiếu báo hỏng phương tiện.
*   **Roles**: `DRIVER`, `MANAGER`
*   **Request Headers**:
    ```http
    Authorization: Bearer <access_token>
    Content-Type: application/json
    Idempotency-Key: <unique_uuid_key>
    ```
*   **Request Body**:
    ```json
    {
      "vehicle_id": "8c59f0de-98fc-11ee-b9d1-0242ac120002",
      "priority": "HIGH",
      "issues": [
        {
          "issue_type": "Hệ thống phanh",
          "description": "Phanh chân không ăn khi đạp sâu, hành trình phanh dài.",
          "severity": "HIGH",
          "attachments": [
            "f2a1b3c4-98fc-11ee-b9d1-0242ac120002"
          ]
        }
      ]
    }
    ```
*   **Response (201 Created / 200 OK if Idempotency hit)**:
    ```json
    {
      "success": true,
      "data": {
        "ticket_id": "d1c2b3a4-98fc-11ee-b9d1-0242ac120002",
        "ticket_no": "TKT-20260628-0001",
        "status": "REPORTED",
        "version": 1
      },
      "meta": {
        "timestamp": "2026-06-28T10:28:13Z"
      }
    }
    ```
*   **Errors**:

    | HTTP Status | Error Code | Meaning / Reason |
    | :--- | :--- | :--- |
    | `400 Bad Request` | `TICKET_TOO_MANY_ISSUES` | Số lượng lỗi khai báo vượt quá cấu hình tối đa (BR-TICKET-04) |
    | `400 Bad Request` | `TICKET_INVALID_ATTACHMENTS` | Tệp đính kèm không tồn tại hoặc sai định dạng (Chapter 20) |
    | `403 Forbidden` | `TICKET_VEHICLE_NOT_ASSIGNED` | Tài xế không được gán vận hành xe này (BR-ASSIGN-03) |
    | `409 Conflict` | `TICKET_ACTIVE_EXISTS` | Xe đang có một phiếu sửa chữa chưa đóng (BR-TICKET-03) |

*   **Business Rules**:
    *   **BR-ASSIGN-03**: Lái xe chỉ được báo hỏng xe họ đang được gán hoạt động.
    *   **BR-TICKET-03**: Một phương tiện tại một thời điểm chỉ có tối đa 1 phiếu sửa chữa hoạt động.
    *   **BR-TICKET-04**: Số lượng lỗi tối đa mặc định là 5 lỗi/phiếu.

---

## 3.3 API-TICKET-03: Chi tiết phiếu sửa chữa

*   **Endpoint**: `GET /api/v1/repair-tickets/:id`
*   **Purpose**: Xem thông tin chi tiết một phiếu sửa chữa bao gồm danh mục lỗi, biên bản kiểm định, vật tư yêu cầu, lịch trình sửa chữa và tệp đính kèm.
*   **Roles**: `All Roles (ABAC Scope)`
*   **Request Headers**:
    ```http
    Authorization: Bearer <access_token>
    ```
*   **Response (200 OK)**:
    ```json
    {
      "success": true,
      "data": {
        "id": "d1c2b3a4-98fc-11ee-b9d1-0242ac120002",
        "ticket_no": "TKT-20260628-0001",
        "vehicle": {
          "id": "8c59f0de-98fc-11ee-b9d1-0242ac120002",
          "plate_number": "29C-12345",
          "vehicle_code": "XE-01"
        },
        "driver": {
          "id": "e3b0c442-98fc-11ee-b9d1-0242ac120002",
          "full_name": "Nguyen Van A"
        },
        "status": "INSPECTING",
        "priority": "HIGH",
        "version": 3,
        "issues": [
          {
            "id": "b1a2c3d4-98fc-11ee-b9d1-0242ac120002",
            "issue_type": "Hệ thống phanh",
            "description": "Phanh chân không ăn khi đạp sâu, hành trình phanh dài.",
            "severity": "HIGH",
            "attachments": [
              {
                "id": "f2a1b3c4-98fc-11ee-b9d1-0242ac120002",
                "file_name": "brake_issue.jpg",
                "file_url": "https://cdn.fixtrack.com/attachments/tickets/d1c2b3a4/issues/b1a2c3d4/uuid.webp?token=temp_signed_get_url"
              }
            ]
          }
        ],
        "created_at": "2026-06-28T09:00:00Z"
      },
      "meta": {
        "timestamp": "2026-06-28T10:28:13Z"
      }
    }
    ```
*   **Errors**:

    | HTTP Status | Error Code | Meaning / Reason |
    | :--- | :--- | :--- |
    | `404 Not Found` | `TICKET_NOT_FOUND` | Phiếu sửa chữa không tồn tại |

*   **Business Rules**:
    *   Tự động sinh Pre-signed GET URL (TTL: 1 giờ) cho toàn bộ tệp đính kèm khi Client truy xuất thông tin chi tiết (Chapter 20).

---

## 3.4 API-TICKET-05: Tiếp nhận kiểm tra sơ bộ

*   **Endpoint**: `POST /api/v1/repair-tickets/:id/accept`
*   **Purpose**: Đội cơ giới (Mechanic) tiếp nhận kiểm tra sơ bộ phương tiện sau khi lái xe báo hỏng.
*   **Roles**: `MECHANIC`
*   **Request Headers**:
    ```http
    Authorization: Bearer <access_token>
    Content-Type: application/json
    ```
*   **Request Body**:
    ```json
    {
      "version": 1
    }
    ```
*   **Response (200 OK)**:
    ```json
    {
      "success": true,
      "data": {
        "ticket_id": "d1c2b3a4-98fc-11ee-b9d1-0242ac120002",
        "status": "INSPECTING",
        "inspected_at": "2026-06-28T10:28:13Z",
        "version": 2
      },
      "meta": {
        "timestamp": "2026-06-28T10:28:13Z"
      }
    }
    ```
*   **Errors**:

    | HTTP Status | Error Code | Meaning / Reason |
    | :--- | :--- | :--- |
    | `400 Bad Request` | `TICKET_INVALID_STATE_TRANSITION` | Trạng thái phiếu hiện tại không phải `REPORTED` (Chapter 18) |
    | `409 Conflict` | `TICKET_RESOURCE_VERSION_CONFLICT` | Phiếu đã bị thay đổi bởi người khác (Optimistic Locking) |

*   **Business Rules**:
    *   **BR-INSP-01**: Chỉ nhân viên cơ giới mới được kiểm tra sơ bộ.
    *   Chuyển trạng thái Ticket từ `REPORTED` sang `INSPECTING`.
    *   Ghi log thời điểm bắt đầu kiểm tra (`inspected_at`).

---

## 3.5 API-TICKET-07: Đẩy xe vào hàng chờ sửa chữa

*   **Endpoint**: `POST /api/v1/repair-tickets/:id/route-queue`
*   **Purpose**: Đội cơ giới (Mechanic) xác định xe cần sửa nhưng xưởng hết slot trống, tự động đưa xe vào hàng chờ sửa chữa.
*   **Roles**: `MECHANIC`
*   **Request Headers**:
    ```http
    Authorization: Bearer <access_token>
    Content-Type: application/json
    ```
*   **Request Body**:
    ```json
    {
      "version": 2,
      "findings": "Xác nhận phanh mòn. Cần đưa vào cầu sửa chữa thay má phanh.",
      "priority_level": 0
    }
    ```
*   **Response (200 OK)**:
    ```json
    {
      "success": true,
      "data": {
        "ticket_id": "d1c2b3a4-98fc-11ee-b9d1-0242ac120002",
        "status": "WAITING_QUEUE",
        "queue_position": 4,
        "version": 3
      },
      "meta": {
        "timestamp": "2026-06-28T10:28:13Z"
      }
    }
    ```
*   **Errors**:

    | HTTP Status | Error Code | Meaning / Reason |
    | :--- | :--- | :--- |
    | `400 Bad Request` | `TICKET_WORKSHOP_SLOTS_AVAILABLE`| Xưởng vẫn còn slot sửa chữa trống, không thể cho xếp hàng |
    | `409 Conflict` | `TICKET_ALREADY_IN_QUEUE` | Xe đã tồn tại trong hàng chờ (BR-QUEUE-02) |
    | `409 Conflict` | `TICKET_RESOURCE_VERSION_CONFLICT` | Phiếu đã bị thay đổi bởi người khác |

*   **Business Rules**:
    *   **BR-QUEUE-01**: Tự động xếp hàng xe nếu xưởng hết slot.
    *   **BR-QUEUE-02**: Một phương tiện chỉ tồn tại tối đa một lần trong hàng chờ.
    *   **BR-QUEUE-03**: Xếp hàng theo cơ chế FIFO (vào trước sửa trước).

---

## 3.6 API-TICKET-08: Nghiệm thu ĐẠT và đóng phiếu

*   **Endpoint**: `POST /api/v1/repair-tickets/:id/approve-inspection`
*   **Purpose**: Đội cơ giới nghiệm thu chạy thử xe đạt yêu cầu, thực hiện khép luồng phiếu sửa chữa, chuyển xe về trạng thái hoạt động bình thường.
*   **Roles**: `MECHANIC`
*   **Request Headers**:
    ```http
    Authorization: Bearer <access_token>
    Content-Type: application/json
    ```
*   **Request Body**:
    ```json
    {
      "version": 5,
      "notes": "Chạy thử 3km, phanh ăn tốt, không phát sinh âm thanh lạ."
    }
    ```
*   **Response (200 OK)**:
    ```json
    {
      "success": true,
      "data": {
        "ticket_id": "d1c2b3a4-98fc-11ee-b9d1-0242ac120002",
        "status": "CLOSED",
        "vehicle_status": "ACTIVE",
        "version": 6
      },
      "meta": {
        "timestamp": "2026-06-28T10:28:13Z"
      }
    }
    ```
*   **Errors**:

    | HTTP Status | Error Code | Meaning / Reason |
    | :--- | :--- | :--- |
    | `400 Bad Request` | `TICKET_INVALID_STATE_TRANSITION` | Phiếu chưa ở trạng thái `COMPLETED` để có thể nghiệm thu |
    | `409 Conflict` | `TICKET_RESOURCE_VERSION_CONFLICT` | Phiếu bị thay đổi bởi luồng khác |

*   **Business Rules**:
    *   **BR-CLOSE-01**: Nghiệm thu đóng phiếu và trả xe về trạng thái `ACTIVE` (hoat_dong).
    *   Giải phóng xe và tài xế khỏi ca gán hiện tại (nếu hết ca).

---

## 3.7 API-MAT-03: Yêu cầu cấp vật tư

*   **Endpoint**: `POST /api/v1/material-requests`
*   **Purpose**: Kỹ thuật viên (Tech) gửi yêu cầu cấp linh kiện phụ tùng phục vụ cho phiếu sửa chữa hiện tại.
*   **Roles**: `TECH`
*   **Request Headers**:
    ```http
    Authorization: Bearer <access_token>
    Content-Type: application/json
    ```
*   **Request Body**:
    ```json
    {
      "ticket_id": "d1c2b3a4-98fc-11ee-b9d1-0242ac120002",
      "items": [
        {
          "material_id": "00a1b2c3-98fc-11ee-b9d1-0242ac120002",
          "quantity_requested": 2
        }
      ]
    }
    ```
*   **Response (201 Created)**:
    ```json
    {
      "success": true,
      "data": {
        "request_id": "a9b8c7d6-98fc-11ee-b9d1-0242ac120002",
        "status": "PENDING"
      },
      "meta": {
        "timestamp": "2026-06-28T10:28:13Z"
      }
    }
    ```
*   **Errors**:

    | HTTP Status | Error Code | Meaning / Reason |
    | :--- | :--- | :--- |
    | `403 Forbidden` | `MAT_NOT_ASSIGNED_TECHNICIAN` | KTV không được gán phụ trách phiếu sửa chữa này (BR-MAT-01) |
    | `400 Bad Request` | `MAT_INVALID_TICKET_STATE` | Trạng thái phiếu không phải `INSPECTING` hoặc `WAITING_PARTS` |

*   **Business Rules**:
    *   **BR-MAT-01**: Chỉ KTV phụ trách mới được tạo yêu cầu vật tư.
    *   Chuyển trạng thái Ticket sang `WAITING_PARTS` tự động khi có yêu cầu vật tư đầu tiên (Side effect - Chapter 18).

---

## 3.8 API-MAT-08: Duyệt hàng loạt yêu cầu vật tư (Bulk API)

*   **Endpoint**: `POST /api/v1/material-requests/bulk-approve`
*   **Purpose**: Thủ kho duyệt cấp phát hàng loạt đơn yêu cầu vật tư của nhiều KTV. Áp dụng chiến lược Phản hồi Thành công Một phần (Partial Success).
*   **Roles**: `INVENTORY`
*   **Request Headers**:
    ```http
    Authorization: Bearer <access_token>
    Content-Type: application/json
    ```
*   **Request Body**:
    ```json
    {
      "approvals": [
        {
          "request_id": "a9b8c7d6-98fc-11ee-b9d1-0242ac120002",
          "warehouse_id": "7d6e5f4c-98fc-11ee-b9d1-0242ac120002"
        },
        {
          "request_id": "b8c7d6e5-98fc-11ee-b9d1-0242ac120002",
          "warehouse_id": "7d6e5f4c-98fc-11ee-b9d1-0242ac120002"
        }
      ]
    }
    ```
*   **Response (200 OK - Partial Success Envelope)**:
    ```json
    {
      "success": true,
      "data": {
        "success_count": 1,
        "failed_count": 1,
        "results": [
          {
            "request_id": "a9b8c7d6-98fc-11ee-b9d1-0242ac120002",
            "status": "ISSUED",
            "error": null
          },
          {
            "request_id": "b8c7d6e5-98fc-11ee-b9d1-0242ac120002",
            "status": "FAILED",
            "error": {
              "code": "MAT_OUT_OF_STOCK",
              "message": "Vật tư mã SKU-889 không đủ tồn kho khả dụng tại kho được chọn."
            }
          }
        ]
      },
      "meta": {
        "timestamp": "2026-06-28T10:28:13Z"
      }
    }
    ```
*   **Errors**: Toàn bộ mảng request bị từ chối chỉ khi có lỗi hệ thống hoặc phân quyền.

    | HTTP Status | Error Code | Meaning / Reason |
    | :--- | :--- | :--- |
    | `403 Forbidden` | `AUTH_INSUFFICIENT_PERMISSIONS` | Người thực hiện không phải Thủ kho |

*   **Business Rules**:
    *   Mỗi phần tử trong mảng được xử lý trong một Database Transaction độc lập. Lỗi của một bản ghi không gây rollback các bản ghi thành công khác.
    *   Trừ tồn kho khả dụng khi duyệt thành công và gửi thông báo nhận phụ tùng cho tài xế (`NT-MAT-02`).

---

## 3.9 API-EXEC-01: Bắt đầu sửa chữa

*   **Endpoint**: `POST /api/v1/repair-jobs/:id/start`
*   **Purpose**: Kỹ thuật viên (Tech) bấm xác nhận bắt đầu thực hiện sửa chữa hạng mục công việc.
*   **Roles**: `TECH`
*   **Request Headers**:
    ```http
    Authorization: Bearer <access_token>
    Content-Type: application/json
    ```
*   **Request Body**:
    ```json
    {
      "version": 1
    }
    ```
*   **Response (200 OK)**:
    ```json
    {
      "success": true,
      "data": {
        "job_id": "c7b6a5d4-98fc-11ee-b9d1-0242ac120002",
        "status": "REPAIRING",
        "started_at": "2026-06-28T10:28:13Z"
      },
      "meta": {
        "timestamp": "2026-06-28T10:28:13Z"
      }
    }
    ```
*   **Errors**:

    | HTTP Status | Error Code | Meaning / Reason |
    | :--- | :--- | :--- |
    | `400 Bad Request` | `REP_PARTS_NOT_DELIVERED` | Vật tư liên quan chưa hoàn thành bàn giao (BR-MAT-04) |
    | `403 Forbidden` | `REP_NOT_ASSIGNED_JOB` | KTV không phải người được phân công công việc này (BR-REP-01) |
    | `409 Conflict` | `TICKET_RESOURCE_VERSION_CONFLICT` | Phiếu đã bị sửa đổi trước đó |

*   **Business Rules**:
    *   **BR-MAT-04**: Không cho phép bắt đầu sửa chữa khi chưa xác nhận nhận đủ vật tư từ lái xe (với xe có yêu cầu vật tư).
    *   Chuyển trạng thái xe sang `REPAIRING` (dang_sua) và ghi nhận timestamp `started_at` cho Job sửa chữa.

---

## 3.10 API-ATTACH-01: Lấy Pre-signed PUT URL

*   **Endpoint**: `POST /api/v1/attachments/presigned-url`
*   **Purpose**: Client yêu cầu cấp Pre-signed URL để tải tệp trực tiếp lên Object Storage (S3/R2/MinIO) mà không đi qua Backend.
*   **Roles**: `All Roles`
*   **Request Headers**:
    ```http
    Authorization: Bearer <access_token>
    Content-Type: application/json
    ```
*   **Request Body**:
    ```json
    {
      "file_name": "damaged_tire.png",
      "mime_type": "image/png",
      "size_bytes": 1048576,
      "entity_type": "TICKET"
    }
    ```
*   **Response (200 OK)**:
    ```json
    {
      "success": true,
      "data": {
        "attachment_id": "f2a1b3c4-98fc-11ee-b9d1-0242ac120002",
        "upload_url": "https://s3.ap-southeast-1.amazonaws.com/fixtrack-attachments/attachments/tickets/temp/f2a1b3c4.png?AWSAccessKeyId=...&Expires=1782782900&Signature=...",
        "storage_key": "attachments/tickets/temp/f2a1b3c4.png",
        "expires_in_seconds": 300
      },
      "meta": {
        "timestamp": "2026-06-28T10:28:13Z"
      }
    }
    ```
*   **Errors**:

    | HTTP Status | Error Code | Meaning / Reason |
    | :--- | :--- | :--- |
    | `400 Bad Request` | `ATTACH_FILE_SIZE_LIMIT_EXCEEDED` | Dung lượng file vượt quá giới hạn (Max: 10MB ảnh, 20MB video) |
    | `400 Bad Request` | `ATTACH_UNSUPPORTED_FILE_TYPE` | Định dạng file không được phép (Chỉ nhận JPG/PNG/WEBP/MP4/MOV) |

*   **Business Rules**:
    *   Tự động sinh TTL cho upload URL là 5 phút (300 giây).
    *   Bản ghi `attachments` được tạo sẵn ở DB dưới trạng thái `requested_upload` phục vụ cơ chế GC quét dọn rác (Chapter 20).

---

# Level 4 — Cross-Cutting Technical Standards

## 4.1 Pagination, Filtering & Sorting

### Pagination Standard
Tất cả các API dạng danh sách (List APIs) đều phải áp dụng phân trang thông qua tham số truy vấn (Query Parameters):
*   `page`: Số trang cần lấy, kiểu số nguyên, mặc định là `1` (1-indexed).
*   `limit`: Số lượng bản ghi trên một trang, mặc định là `20`, tối đa là `100`.

### Filtering Standard
Lọc dữ liệu được truyền dưới dạng query parameters trùng khớp với tên trường cơ sở dữ liệu:
`GET /api/v1/repair-tickets?status=REPAIRING&priority=HIGH`

### Sorting Standard
*   Cú pháp: `?sort=field_name:direction`
*   `direction` nhận một trong hai giá trị: `asc` (tăng dần) hoặc `desc` (giảm dần).
*   Hỗ trợ sắp xếp đa trường bằng dấu phẩy: `?sort=priority:desc,created_at:desc`

---

## 4.2 Rate Limiting
Để đảm bảo tính khả dụng cao của hệ thống, Backend thực hiện giới hạn tần suất gọi API như sau:
*   **Auth APIs (Login/Reset)**: Tối đa 5 requests / 1 phút cho 1 IP.
*   **General APIs (Read/Write)**: Tối đa 100 requests / 1 phút cho 1 Tài khoản/IP.
*   *Headers trả về từ Server:*
    ```http
    X-RateLimit-Limit: 100
    X-RateLimit-Remaining: 98
    X-RateLimit-Reset: 1782782945
    ```

---

## 4.3 Idempotency
Để phòng tránh việc Mobile Client gửi lặp lại các tác vụ thay đổi dữ liệu do mạng chập chờn (như bấm Tạo phiếu 2 lần):
*   Áp dụng Header `Idempotency-Key` (bắt buộc phải là dạng UUID v4).
*   **Flow xử lý**:
    1. Khi nhận request có `Idempotency-Key`, Backend kiểm tra xem key này đã tồn tại trong Redis chưa.
    2. Nếu đã tồn tại (đang xử lý hoặc đã xử lý xong): Trả lại ngay lập tức response đã lưu trong Redis mà không chạy lại logic nghiệp vụ (tránh trùng lặp).
    3. Nếu chưa tồn tại: Khởi tạo lock trong Redis, xử lý request nghiệp vụ, lưu lại kết quả response vào Redis (TTL: 24 giờ) trước khi trả về cho Client.

---

## 4.4 Optimistic Locking (Lost Update Prevention)
Để loại bỏ nguy cơ ghi đè mất mát dữ liệu (Lost Update) khi hai người dùng (ví dụ: KTV và Manager) mở cùng một màn hình sửa đổi phiếu cùng lúc:
*   Tất cả các bảng nghiệp vụ cốt lõi (`repair_tickets`, `material_requests`) đều chứa cột `version` kiểu số nguyên (`INTEGER DEFAULT 1`).
*   **Quy trình xác thực**:
    1. Khi Client GET thông tin chi tiết một phiếu sửa chữa, API sẽ trả về trường `version` hiện tại (ví dụ: `version: 3`).
    2. Khi Client gửi yêu cầu cập nhật thông tin (`PATCH`) hoặc thay đổi trạng thái, Client bắt buộc phải đính kèm trường `version: 3` trong Request Body.
    3. **Backend check**:
       ```sql
       UPDATE repair_tickets 
       SET status = 'REPAIRING', version = version + 1 
       WHERE id = :ticket_id AND version = :version_client;
       ```
    4. Nếu số dòng được update thành công là `0` (nghĩa là DB đã lên version 4 bởi người khác), Backend lập tức rollback transaction và trả về lỗi `409 Conflict` kèm code `RESOURCE_VERSION_CONFLICT`.

---

## 4.5 Error Code Standard
Mọi mã lỗi nghiệp vụ trả về trong trường `error.code` phải tuân theo quy chuẩn tiền tố của module tương ứng:

| Mã lỗi chuẩn | HTTP Status | Ý nghĩa nghiệp vụ |
| :--- | :---: | :--- |
| `AUTH_INVALID_CREDENTIALS` | 401 | Mã nhân viên hoặc mật khẩu không chính xác |
| `AUTH_TOKEN_EXPIRED` | 401 | Access Token đã hết hạn |
| `AUTH_INSUFFICIENT_PERMISSIONS` | 403 | Không đủ quyền hạn thực hiện tác vụ (RBAC/ABAC) |
| `TICKET_NOT_FOUND` | 404 | Không tìm thấy mã phiếu sửa chữa |
| `TICKET_ACTIVE_EXISTS` | 409 | Xe đang có phiếu sửa chữa hoạt động, không được tạo mới |
| `TICKET_RESOURCE_VERSION_CONFLICT` | 409 | Phiếu bị cập nhật bởi người khác (Optimistic Locking) |
| `QUEUE_SLOT_FULL` | 400 | Hàng chờ đầy hoặc không còn slot trống |
| `MAT_OUT_OF_STOCK` | 400 | Phụ tùng yêu cầu không đủ số lượng tồn kho khả dụng |
| `REP_PARTS_NOT_DELIVERED` | 400 | Chưa hoàn thành nhận bàn giao vật tư, chưa được phép sửa |
| `ATTACH_FILE_SIZE_LIMIT_EXCEEDED` | 400 | Tệp đính kèm vượt quá dung lượng cho phép |

---

# Level 5 — Advanced FixTrack Extensions

## 5.1 Open Design Decisions
Các quyết định về kiến trúc hệ thống dưới đây được chốt phương án và áp dụng đồng bộ trong thiết kế:
*   **Refresh Token Storage**: Lưu trữ tại **Redis** để hỗ trợ cơ chế thu hồi (Revoke/Signout) thời gian thực và tự động hủy sau 7 ngày (TTL).
*   **Real-time Protocol**: Chọn **WebSocket** làm giao thức truyền tin hai chiều thời gian thực (hỗ trợ live queue, live dashboard, live notifications).
*   **Optimistic Locking Strategy**: Chọn **Integer Versioning** (cột `version` tự tăng khi có sửa đổi) thay vì timestamp `updated_at` để loại bỏ sai số thời gian của server (server clock drift).
*   **Bulk API Transaction Policy**: Chọn **Partial Success (Thành công một phần)**. Cho phép xử lý độc lập từng dòng trong danh sách. Kết quả trả về chứa thống kê số lượng thành công/thất bại kèm lỗi chi tiết cho từng dòng.

---

## 5.2 Real-time Event Specification (WebSocket)
Hệ thống FixTrack sử dụng giao thức **WebSocket (WSS)** thông qua thư viện Socket.IO để cập nhật giao diện thời gian thực trên PWA Portal và Mobile App mà không cần Client phải thực hiện load lại trang (Polling).

*   **Base Socket Host**: `https://api.fixtrack.com`
*   **Default Socket.IO Path**: `/socket.io`
*   **Authentication**: Để đảm bảo tính bảo mật và tránh rò rỉ token qua URL logs, Client bắt buộc phải gửi Access Token trong trường `auth` payload khi khởi tạo kết nối:
    ```javascript
    const socket = io("wss://api.fixtrack.com", {
      auth: {
        token: "Bearer <access_token>"
      }
    });
    ```
    Phía Backend Server thực hiện xác thực token thông qua thuộc tính: `socket.handshake.auth.token` tại middleware xác thực kết nối.

### Core Real-time Events Payload List
Khi có sự thay đổi trạng thái hệ thống, Backend phát đi các sự kiện sau tới các Client đăng ký phòng (Rooms) tương ứng:

#### 1. Sự kiện `QUEUE_UPDATED` (Cập nhật hàng chờ xưởng)
*   **Đối tượng nhận**: Toàn bộ Client đang mở màn hình Hàng chờ sửa chữa (Room: `workshop_queue`).
*   **Payload**:
    ```json
    {
      "event": "QUEUE_UPDATED",
      "payload": {
        "action": "PROMOTE",
        "ticket_id": "d1c2b3a4-98fc-11ee-b9d1-0242ac120002",
        "new_position": 1,
        "previous_position": 4,
        "updated_by": "Manager B"
      }
    }
    ```

#### 2. Sự kiện `TICKET_STATUS_CHANGED` (Đổi trạng thái phiếu)
*   **Đối tượng nhận**: Tài xế phụ trách xe, KTV được gán, và Đội cơ giới (Room: `ticket_{ticket_id}`).
*   **Payload**:
    ```json
    {
      "event": "TICKET_STATUS_CHANGED",
      "payload": {
        "ticket_id": "d1c2b3a4-98fc-11ee-b9d1-0242ac120002",
        "old_status": "WAITING_PARTS",
        "new_status": "REPAIRING",
        "updated_at": "2026-06-28T10:28:13Z"
      }
    }
    ```

#### 3. Sự kiện `NOTIFICATION_RECEIVED` (Có thông báo mới)
*   **Đối tượng nhận**: Người dùng cụ thể (Room: `user_{user_id}`).
*   **Payload**:
    ```json
    {
      "event": "NOTIFICATION_RECEIVED",
      "payload": {
        "id": "e4d3c2b1-98fc-11ee-b9d1-0242ac120002",
        "title": "Vật tư đã xuất kho",
        "body": "Vật tư cho xe XE-01 đã xuất kho. Vui lòng đến kho nhận phụ tùng.",
        "deep_link": "/repair-requests/d1c2b3a4-98fc-11ee-b9d1-0242ac120002"
      }
    }
    ```

---

## 5.3 Verification Plan

Để bảo đảm tính toàn vẹn của thiết kế API trước khi chuyển giao sang giai đoạn lập trình, QA Team thực hiện quy trình đối chiếu tự động và thủ công theo bảng kiểm (Checklist) dưới đây:

### API Contract Validation Checklist
- `[ ]` **Mapping Use Case**: Đảm bảo 100% Use Cases trong [10 Use Cases](../part_b_requirement_analysis/10_use_cases.md) đều có ít nhất 1 API tương ứng thực thi hành động.
- `[ ]` **State Machine Alignment**: Mọi API đổi trạng thái phiếu/vật tư phải trùng khớp hoàn toàn với sơ đồ trạng thái và điều kiện Guards quy định tại [18 State Machine](../part_d_application_design/18_state_machine.md).
- `[ ]` **CRUD & Access Control Validation**: Ma trận quyền (`Roles Allowed`) trong API Matrix phải tương thích hoàn toàn với cấu trúc quyền của [14 CRUD Matrix](../part_c_data_design/14_crud_matrix.md) và [05 User Roles](../part_a_business_foundation/05_user_roles.md).
- `[ ]` **DB Schema Match**: Tên các thuộc tính trong JSON Request/Response (snake_case) phải ánh xạ trực tiếp và trùng khớp kiểu dữ liệu với cấu trúc các trường trong [15 Database Schema](../part_c_data_design/15_database_schema.md).
- `[ ]` **Attachment Constraint Verification**: Ràng buộc về kích thước (10MB/20MB) và định dạng file trong `API-ATTACH-01` phải khớp với đặc tả của [20 Attachment Design](../part_d_application_design/20_attachment_design.md).
