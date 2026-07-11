# 09 Business Rules

**Trạng thái: Draft**

---

## Purpose
Chương này định nghĩa toàn bộ các quy tắc nghiệp vụ cốt lõi (Business Rules - BR) và ràng buộc hệ thống mà ứng dụng VRMS phải tuân thủ. Các quy tắc này đóng vai trò là "luật chơi" của hệ thống, ràng buộc hành vi người dùng, kiểm soát tiến trình trạng thái và bảo đảm dữ liệu nhất quán xuyên suốt vòng đời phiếu sửa chữa.

---

## Questions Answered
- Ai được phép thực hiện từng thao tác nghiệp vụ cụ thể?
- Những điều kiện bắt buộc nào phải thỏa mãn để cho phép hoặc ngăn chặn một thao tác?
- Quy trình kiểm soát vật tư và quản lý hàng chờ phải tuân theo các luật nào?
- Những vấn đề nghiệp vụ nào chưa được chốt phương án xử lý cuối cùng (TBD)?

---

## Inputs
- [05 User Roles](file:///c:/Users/Legion/Desktop/IT/SNP/.agents/skills/system_engineering_copilot/deliverables/current/sss/part_a_business_foundation/05_user_roles.md) (Quyền và ma trận phân quyền).
- [06 Workflow Analysis](file:///c:/Users/Legion/Desktop/IT/SNP/.agents/skills/system_engineering_copilot/deliverables/current/sss/part_b_requirement_analysis/06_workflow_analysis.md) (7 bước nghiệp vụ số hóa).
- [07 Functional Requirements](file:///c:/Users/Legion/Desktop/IT/SNP/.agents/skills/system_engineering_copilot/deliverables/current/sss/part_b_requirement_analysis/07_functional_requirements.md) (Danh sách chức năng hệ thống).

---

## Content

### Level 1 — Summary
Khác với Workflow (Chapter 06 - Công việc chạy theo luồng nào) và Functional Requirements (Chapter 07 - Hệ thống hỗ trợ chức năng gì), Business Rules là các luật và điều kiện bắt buộc mà tất cả các API, luồng xử lý và giao diện người dùng phải tuân thủ.

Hệ thống VRMS áp dụng 26 quy tắc nghiệp vụ cốt lõi chia thành 8 nhóm chức năng nghiệp vụ, đóng vai trò làm nền tảng để lập trình logic xác thực (Validation Logic), kiểm soát phân quyền (Authorization) và ràng buộc dữ liệu (Data Constraints). Ngoài ra, chương này cũng tổng hợp 3 điểm nghiệp vụ chưa thống nhất phương án (To Be Decided - TBD) để tiếp tục làm việc với các bên liên quan.

---

### Level 2 — Breakdown

#### 1. Assignment Rules (Quy tắc phân công phương tiện)

##### **BR-ASSIGN-01: Quyền phân công phương tiện**
*   **Mô tả:** Chỉ người dùng có vai trò Đội cơ giới (Fleet Dispatcher) mới được phép phân công phương tiện cho tài xế lái.
*   **Lý do nghiệp vụ:** Đảm bảo tính tập trung trong quản lý đội xe và tránh việc gán xe chồng chéo.
*   **Điều kiện áp dụng:** Khi thực hiện thao tác phân công xe trên màn hình điều hành.
*   **Hành động hệ thống:**
    *   *Nếu Pass:* Tạo bản ghi trong lịch sử gán xe và kích hoạt thông báo cho tài xế.
    *   *Nếu Fail:* Chặn thao tác và báo lỗi "Quyền hạn không hợp lệ".
*   **Ngoại lệ:** Quản lý cấp cao (`MANAGER`) có quyền ghi đè (Override) phân công trong trường hợp khẩn cấp.
*   **Related FR:** `FR-ASSIGN-01`

##### **BR-ASSIGN-02: Ràng buộc tài xế active**
*   **Mô tả:** Tại một thời điểm vận hành thực tế, một tài xế chỉ được hoạt động (Active) trên một phương tiện duy nhất.
*   **Lý do nghiệp vụ:** Xác định chính xác một người chịu trách nhiệm duy nhất cho phương tiện khi xảy ra sự cố trên đường.
*   **Điều kiện áp dụng:** Khi gán xe cho tài xế.
*   **Hành động hệ thống:**
    *   *Nếu Pass:* Thực hiện gán xe.
    *   *Nếu Fail:* Chặn thao tác, báo lỗi "Tài xế hiện đang vận hành xe khác hoặc xe đã được gán cho tài xế khác".
*   **Ngoại lệ:** Không có.
*   **Related FR:** `FR-ASSIGN-01`

##### **BR-ASSIGN-03: Quyền tạo Ticket sửa chữa**
*   **Mô tả:** Chỉ tài xế đang được gán xe (Active Driver) mới được phép tạo phiếu báo hỏng xe đó (BR-01 cũ).
*   **Lý do nghiệp vụ:** Đảm bảo người báo hỏng là người trực tiếp vận hành phương tiện, tránh tình trạng báo ảo hoặc báo nhầm xe.
*   **Điều kiện áp dụng:** Khi tài xế nhấn tạo phiếu sự cố.
*   **Hành động hệ thống:**
    *   *Nếu Pass:* Cho phép truy cập giao diện tạo ticket.
    *   *Nếu Fail:* Chặn không cho vào giao diện và hiển thị thông báo lỗi.
*   **Ngoại lệ:** Không có.
*   **Related FR:** `FR-TICKET-01`

##### **BR-ASSIGN-04: Điều hướng xử lý theo địa điểm**
*   **Mô tả:** Phương tiện đăng ký hoạt động tại chi nhánh nào (ví dụ: MTY1, MT2, TV, NT) thì chỉ có Đội Cơ giới và Đội Kỹ thuật tại chi nhánh đó mới được phép tiếp nhận, kiểm tra sơ bộ, chẩn đoán và sửa chữa phương tiện.
*   **Lý do nghiệp vụ:** Đảm bảo phân chia công việc theo địa giới hoạt động thực tế, tránh điều phối sai vùng địa lý của xưởng sửa chữa.
*   **Điều kiện áp dụng:** Khi Đội Cơ giới hoặc KTV thực hiện tiếp nhận, khám xe hoặc sửa xe.
*   **Hành động hệ thống:**
    *   *Nếu Pass:* Cho phép thao tác tiếp nhận và cập nhật trạng thái phiếu.
    *   *Nếu Fail:* Chặn hành động và báo lỗi "Phương tiện không thuộc chi nhánh của bạn phụ trách".
*   **Ngoại lệ:** Quản lý (`MANAGER`) có quyền xem và can thiệp điều phối liên chi nhánh.
*   **Related FR:** `FR-INSP-01`, `FR-REP-01`

---

#### 2. Ticket Rules (Quy tắc phiếu sửa chữa)

##### **BR-TICKET-01: Quan hệ Ticket - Xe**
*   **Mô tả:** Một phiếu sửa chữa (Repair Ticket) chỉ được liên kết với duy nhất một phương tiện.
*   **Lý do nghiệp vụ:** Nhất quán dữ liệu và dễ dàng đối chiếu lịch sử bảo trì của từng đầu xe.
*   **Related FR:** `FR-TICKET-01`

##### **BR-TICKET-02: Báo hỏng đa lỗi**
*   **Mô tả:** Một phiếu sửa chữa được phép chứa nhiều lỗi hư hỏng khác nhau của phương tiện để xử lý gộp (BR-03 cũ).
*   **Lý do nghiệp vụ:** Giảm thiểu số lượng phiếu rác và phản ánh chính xác tình trạng thực tế của xe tại thời điểm báo sự cố.
*   **Related FR:** `FR-TICKET-02`

##### **BR-TICKET-03: Chặn tạo phiếu sửa chữa trùng lặp**
*   **Mô tả:** Không cho phép tạo phiếu sửa chữa mới nếu phương tiện đang có phiếu sửa chữa khác ở các trạng thái mở (`REPORTED`, `INITIAL_INSPECTION`, `TECHNICAL_DIAGNOSIS`, `WAITING_QUEUE`, `REPAIRING`).
*   **Lý do nghiệp vụ:** Tránh tạo các phiếu yêu cầu sửa chữa trùng lặp gây lãng phí nguồn lực kiểm tra.
*   **Điều kiện áp dụng:** Khi nhấn nút "Gửi báo cáo" tạo phiếu mới.
*   **Hành động hệ thống:**
    *   *Nếu Pass:* Tạo mới ticket.
    *   *Nếu Fail:* Chặn lưu và báo lỗi "Phương tiện này hiện đang có phiếu sửa chữa chưa đóng".
*   **Ngoại lệ:** Không có.
*   **Related FR:** `FR-TICKET-01`

##### **BR-TICKET-04: Số lượng lỗi cấu hình trên phiếu**
*   **Mô tả:** Số lượng lỗi tối đa được khai báo trên một phiếu sửa chữa là giá trị cấu hình hệ thống (System Configurable), mặc định ban đầu là 5 lỗi. Giá trị này được quản lý bởi cấu hình hệ thống và có thể thay đổi mà không cần sửa mã nguồn.
*   **Lý do nghiệp vụ:** Tránh việc dồn quá nhiều lỗi nhỏ không liên quan vào cùng một phiếu làm kéo dài thời gian xử lý và gây ùn tắc xưởng sửa chữa.
*   **Related FR:** `FR-TICKET-02`

---

#### 3. Inspection Rules (Quy tắc kiểm tra & tiếp nhận)

##### **BR-INSP-01: Quyền kiểm tra sơ bộ**
*   **Mô tả:** Chỉ nhân viên thuộc Đội cơ giới (Mechanic Team) mới được phép thực hiện kiểm tra sơ bộ phương tiện sau khi tài xế báo lỗi.
*   **Lý do nghiệp vụ:** Đội cơ giới chịu trách nhiệm điều phối hoạt động xe, cần đánh giá sơ bộ trước khi gán việc cho xưởng kỹ thuật.
*   **Related FR:** `FR-INSP-01`

##### **BR-INSP-02: Phán quyết sau kiểm tra sơ bộ**
*   **Mô tả:** Phiếu sửa chữa bắt buộc phải được chuyển sang trạng thái từ chối (`REJECTED`) hoặc chuyển sang chẩn đoán kỹ thuật (`TECHNICAL_DIAGNOSIS`) sau bước kiểm tra sơ bộ (`INITIAL_INSPECTION`).
*   **Lý do nghiệp vụ:** Làm rõ định hướng xử lý xe, tránh tình trạng ticket bị treo lơ lửng không rõ hướng giải quyết.
*   **Related FR:** `FR-INSP-03`

##### **BR-INSP-03: Yêu cầu lý do từ chối**
*   **Mô tả:** Khi chọn kết quả kiểm tra là từ chối phiếu (Reject), Cơ giới bắt buộc phải nhập lý do cụ thể trên ứng dụng.
*   **Lý do nghiệp vụ:** Đảm bảo tài xế hiểu rõ lý do tại sao xe của họ được đánh giá là hoạt động tốt hoặc lỗi không cần sửa.
*   **Related FR:** `FR-INSP-03`

---

#### 4. Queue Rules (Quy tắc hàng chờ tại xưởng)

##### **BR-QUEUE-01: Tự động xếp hàng xe**
*   **Mô tả:** Khi Cơ giới duyệt chuyển xưởng nhưng toàn bộ các slot sửa chữa trong xưởng kỹ thuật bị đầy, xe tự động được đưa vào hàng chờ (`WAITING_QUEUE`).
*   **Lý do nghiệp vụ:** Tự động điều tiết công việc tại xưởng sửa chữa.
*   **Related FR:** `FR-QUEUE-01`

##### **BR-QUEUE-02: Ràng buộc duy nhất trong queue**
*   **Mô tả:** Một phương tiện chỉ được tồn tại tối đa một lần trong hàng chờ sửa chữa tại xưởng.
*   **Lý do nghiệp vụ:** Ngăn ngừa lỗi trùng lặp dữ liệu điều hành hàng chờ.
*   **Related FR:** `FR-QUEUE-01`

##### **BR-QUEUE-03: Thứ tự ưu tiên mặc định**
*   **Mô tả:** Hàng chờ sửa chữa hoạt động theo nguyên tắc FIFO (Vào trước sửa trước), trừ trường hợp được Quản lý (`MANAGER`) ghi đè (Override) quyền ưu tiên sửa chữa trước.
*   **Lý do nghiệp vụ:** Đảm bảo tính công bằng và có cơ chế xử lý linh hoạt khi phát sinh xe khẩn cấp.
*   **Related FR:** `FR-QUEUE-03`

##### **BR-QUEUE-04: Cảnh báo SLA hàng chờ**
*   **Mô tả:** Nếu một xe nằm trong hàng chờ (`WAITING_QUEUE`) vượt quá ngưỡng thời gian quy định trong cấu hình SLA (giá trị cấu hình SLA, mặc định ban đầu: 2 giờ) mà chưa được sửa, hệ thống sẽ tự động gửi cảnh báo (Alert) đến Quản lý.
*   **Lý do nghiệp vụ:** Giám sát thời gian chờ và ngăn chặn xe bị bỏ quên trong hàng đợi quá lâu (Downtime cao).
*   **Related FR:** `FR-QUEUE-01`

---

#### 5. Material Rules (Quy tắc cấp phát vật tư)

##### **BR-MAT-01: Quyền yêu cầu vật tư**
*   **Mô tả:** Chỉ Kỹ thuật viên (KTV) được phân công đảm nhận phiếu sửa chữa mới có quyền tạo đơn yêu cầu vật tư cho xe đó.
*   **Lý do nghiệp vụ:** KTV là người trực tiếp chẩn đoán chuyên sâu và chịu trách nhiệm về kỹ thuật sửa chữa.
*   **Related FR:** `FR-MAT-02`

##### **BR-MAT-02: Quyền xuất kho vật tư**
*   **Mô tả:** Chỉ thủ kho vật tư mới được quyền phê duyệt và xác nhận xuất kho linh kiện phụ tùng trên ứng dụng.
*   **Lý do nghiệp vụ:** Đảm bảo tính bảo mật và đúng trách nhiệm quản lý kho bãi vật lý.
*   **Related FR:** `FR-MAT-03`

##### **BR-MAT-03: Luồng cấp phát 3 bên**
*   **Mô tả:** Vật tư bắt buộc phải được bàn giao theo chuỗi khép kín: *Thủ kho duyệt xuất -> Tài xế nhận tại kho -> Tài xế mang giao cho KTV tại xưởng* (BR-02 cũ).
*   **Lý do nghiệp vụ:** Kiểm soát chặt chẽ phụ tùng bàn giao, chống thất thoát vật tư bằng cách gán trách nhiệm giữ hàng vật lý cho tài xế trước khi bàn giao cho xưởng.
*   **Related FR:** `FR-MAT-04`

##### **BR-MAT-04: Ràng buộc bắt đầu sửa chữa**
*   **Mô tả:** KTV không được phép bấm nút bắt đầu sửa chữa trên ứng dụng nếu yêu cầu vật tư tương ứng của phiếu chưa hoàn thành bước xác nhận nhận đủ vật tư từ lái xe.
*   **Lý do nghiệp vụ:** Tránh trường hợp sửa xe khống khi chưa thực sự có vật tư, đảm bảo tính chân thực của dữ liệu sửa chữa.
*   **Related FR:** `FR-REP-01`

##### **BR-MAT-05: Quyền hủy yêu cầu vật tư**
*   **Mô tả:** Chỉ KTV đã tạo yêu cầu hoặc Quản lý (`MANAGER`) mới được phép hủy (Cancel) yêu cầu vật tư khi vật tư đó chưa được thủ kho bấm xuất kho (trạng thái trước `ISSUED`).
*   **Lý do nghiệp vụ:** Cho phép sửa đổi phương án sửa chữa nếu chẩn đoán ban đầu thay đổi, đồng thời ngăn chặn thay đổi khi hàng vật lý đã rời kho.
*   **Related FR:** `FR-MAT-02`

##### **BR-MAT-06: Quyền nhập kho vật tư**
*   **Mô tả:** Chỉ nhân sự thuộc bộ phận Kho vật tư (`INVENTORY`) mới có quyền thực hiện nghiệp vụ Nhập kho thủ công để cộng thêm số lượng tồn kho khả dụng của linh kiện phụ tùng trên hệ thống.
*   **Lý do nghiệp vụ:** Kiểm soát chặt chẽ đầu vào vật tư, tránh việc tăng ảo số lượng tồn kho bởi các bộ phận khác.
*   **Related FR:** `FR-MAT-01`

##### **BR-MAT-07: Quy trình trả vật tư dư thừa**
*   **Mô tả:** Trong quá trình sửa chữa, nếu phát hiện phụ tùng đã cấp phát không dùng hết hoặc còn nguyên vẹn, KTV thực hiện tạo yêu cầu trả hàng trên ứng dụng. Hệ thống chỉ cộng lại số lượng vào tồn kho khả dụng (`on_hand`) sau khi Thủ kho kiểm tra thực tế và bấm xác nhận đã nhận lại hàng vật lý tại kho.
*   **Lý do nghiệp vụ:** Chống thất thoát vật tư dư thừa từ xưởng, đảm bảo số liệu tồn kho luôn khớp với thực tế kho vật lý.
*   **Related FR:** `FR-MAT-04`

---

#### 6. Repair Rules (Quy tắc thực thi sửa chữa)

##### **BR-REP-01: Quyền sửa xe**
*   **Mô tả:** Chỉ KTV mới được cập nhật tiến độ và bấm bắt đầu/kết thúc sửa chữa thực tế xe tại xưởng.
*   **Lý do nghiệp vụ:** Tránh các bộ phận khác cập nhật láo hoặc sai lệch thời gian sửa xe thực tế.
*   **Related FR:** `FR-REP-01`

##### **BR-REP-02: Điều kiện bắt đầu sửa**
*   **Mô tả:** Phiếu sửa chữa chỉ được chuyển sang trạng thái đang sửa (`REPAIRING`) khi có slot sửa chữa trống trong xưởng và toàn bộ vật tư yêu cầu đã được KTV xác nhận nhận bàn giao (đối với xe cần vật tư). Nếu tại bước phê duyệt cấp phát vật tư, Thủ kho xác nhận kho không đủ hàng để cấp, hệ thống tự động giải phóng slot xưởng của xe này, đưa xe về lại hàng chờ `WAITING_QUEUE` với nhãn cảnh báo `LACKING_MATERIALS` để nhường slot xưởng cho xe tiếp theo đã đủ vật tư.
*   **Lý do nghiệp vụ:** Đảm bảo KTV thực sự có đủ công cụ và phụ tùng trước khi bắt tay vào sửa, tránh việc nghẽn slot xưởng do chờ vật tư.
*   **Related FR:** `FR-REP-01`

##### **BR-REP-03: Mối liên kết vật tư**
*   **Mô tả:** Tất cả vật tư xuất kho phục vụ sửa chữa bắt buộc phải được gắn mã liên kết với một phiếu sửa chữa cụ thể.
*   **Lý do nghiệp vụ:** Phục vụ truy vết chi phí, phân tích độ hao mòn xe và thực hiện kiểm toán vật tư định kỳ.
*   **Related FR:** `FR-MAT-02`

---

#### 7. Closure Rules (Quy tắc đóng phiếu)

##### **BR-CLOSE-01: Nghiệm thu đóng phiếu**
*   **Mô tả:** Phiếu sửa chữa được chuyển sang trạng thái đóng (`CLOSED`) trực tiếp bởi nhân viên Đội Cơ giới sau khi chạy thử nghiệm thu thực tế đạt yêu cầu. Kỹ thuật viên xưởng không có quyền tự đóng phiếu sửa chữa trên ứng dụng. Xe sau khi đóng phiếu sẽ tự động trở lại trạng thái hoạt động bình thường (`ACTIVE`).
*   **Lý do nghiệp vụ:** Tạo chốt chặn kiểm soát chất lượng chéo (Cơ giới kiểm tra kết quả sửa chữa của Đội kỹ thuật), đảm bảo an toàn tuyệt đối cho phương tiện trước khi lưu thông.
*   **Related FR:** `FR-REP-04`

##### **BR-CLOSE-02: Quay vòng sửa lại**
*   **Mô tả:** Nếu kết quả nghiệm thu kỹ thuật không đạt, phiếu sửa chữa bắt buộc phải được Cơ giới đẩy ngược về trạng thái đang sửa (`REPAIRING`) để KTV xử lý lại.
*   **Lý do nghiệp vụ:** Đảm bảo xe hỏng không được đưa ra ngoài hoạt động và bắt buộc xưởng phải khắc phục triệt để lỗi.
*   **Related FR:** `FR-REP-04`

##### **BR-CLOSE-03: Quy tắc mở lại phiếu (Reopen Ticket)**
*   **Mô tả:** Phiếu sửa chữa đã đóng (`CLOSED`) không được phép chỉnh sửa dữ liệu trực tiếp. Nếu lỗi kỹ thuật cũ tái diễn, hệ thống yêu cầu tài xế tạo phiếu sửa chữa mới, hoặc chỉ cho phép Quản lý (`MANAGER`) thực hiện mở lại (Reopen) phiếu đã đóng trong khoảng thời gian gia hạn quy định (Grace Period).
*   **Lý do nghiệp vụ:** Đảm bảo tính nhất quán của dữ liệu lịch sử bảo dưỡng đã nghiệm thu và ngăn chặn sửa đổi log bất hợp pháp, đồng thời cung cấp cơ chế xử lý khẩn cấp khi xe tái hỏng lập tức.
*   **Related FR:** `FR-REP-04`

---

#### 8. Security & Audit Rules (Quy tắc bảo mật và ghi vết)

##### **BR-AUDIT-01: Ghi log trạng thái**
*   **Mô tả:** Mọi sự kiện chuyển dịch trạng thái của phiếu sửa chữa bắt buộc phải tự động ghi vào nhật ký hệ thống (Audit Log) gồm: Mã người dùng, vai trò, trạng thái cũ, trạng thái mới và timestamp.
*   **Lý do nghiệp vụ:** Đảm bảo tính minh bạch, hỗ trợ truy vết lỗi khi có tranh chấp và đo đạc chính xác thời gian xử lý của từng khâu (SLA tracking).
*   **Related FR:** `FR-DASH-02`

##### **BR-AUDIT-02: Truy vết vật tư**
*   **Mô tả:** Mọi giao dịch phụ tùng trên ứng dụng phải ghi vết rõ ràng 3 thời điểm: thời điểm Thủ kho xuất, thời điểm Tài xế nhận và thời điểm KTV nhận.
*   **Lý do nghiệp vụ:** Ngăn chặn tuyệt đối tình trạng thất thoát phụ tùng trên đường vận chuyển từ kho đến xưởng kỹ thuật.
*   **Related FR:** `FR-DASH-02`

---

#### 9. Quy tắc nghiệp vụ chưa chốt phương án (To Be Decided - TBD)

> [!WARNING]
> **TBD-01: Phương án xử lý khi Kho thiếu vật tư**
> Khi KTV yêu cầu phụ tùng nhưng kho không đủ số lượng tồn kho khả dụng:
> *   *Phương án 1 (Reject):* Từ chối toàn bộ đơn, yêu cầu KTV sửa lại đơn khác.
> *   *Phương án 2 (Partial Issue):* Cho phép xuất kho một phần trước, phần thiếu ghi nhận nợ (Backorder) cấp phát sau.
> *   *Phương án 3 (Purchase Trigger):* Tự động tạo phiếu yêu cầu mua sắm vật tư gửi phòng mua hàng.

> [!WARNING]
> **TBD-02: Quy tắc ưu tiên hàng chờ sửa chữa (Queue Priority)**
> Ngoài quy tắc mặc định FIFO, hệ thống có tự động nâng quyền ưu tiên đối với các trường hợp đặc biệt hay không:
> *   *Phương án 1:* FIFO hoàn toàn, chỉ cho phép Manager điều chỉnh tay trên App.
> *   *Phương án 2:* Tự động ưu tiên trước đối với các lỗi thuộc nhóm nguy hiểm/ảnh hưởng an toàn vận hành nghiêm trọng (Safety Critical).
> *   *Phương án 3:* Ưu tiên theo loại xe (xe khách chạy tuyến ưu tiên trước xe tải nội bộ).

> [!WARNING]
> **TBD-03: Xử lý đóng phiếu sửa chữa đa lỗi bán phần (Multi-issue Partial Close)**
> Khi 1 phiếu báo hỏng chứa 5 lỗi, KTV sửa xong 4 lỗi, còn 1 lỗi chưa sửa được do thiếu phụ tùng thay thế:
> *   *Phương án 1 (Chặn đóng phiếu):* Không cho phép nghiệm thu đóng phiếu. Bắt buộc phải đợi phụ tùng về sửa xong lỗi thứ 5 mới được đóng.
> *   *Phương án 2 (Tách phiếu - Recommend):* Cho phép Cơ giới nghiệm thu đạt 4 lỗi, lỗi thứ 5 tự động được hệ thống tách thành một phiếu sửa chữa mới ở trạng thái chờ phụ tùng để giải phóng xe chạy tiếp.
> *   *Phương án 3 (Đóng kèm ghi chú):* Cho phép đóng phiếu và ghi nhận lỗi thứ 5 là "lỗi tồn đọng chờ bảo trì định kỳ sau".

---

## Level 3 — Technical Detail

### 1. Danh mục Quy tắc Nghiệp vụ (Rule Catalog)
Dưới đây là danh mục toàn bộ 26 quy tắc nghiệp vụ áp dụng cho hệ thống:

| Mã Rule (ID) | Tên quy tắc (Rule Name) | Nhóm quy tắc | Mô tả tóm tắt (Description) |
| :--- | :--- | :--- | :--- |
| **BR-ASSIGN-01** | Quyền gán phương tiện | Assignment | Chỉ Cơ giới (Dispatcher) được gán xe cho tài xế. |
| **BR-ASSIGN-02** | Ràng buộc tài xế active | Assignment | Tại một thời điểm vận hành, tài xế chỉ active trên 1 xe. |
| **BR-ASSIGN-03** | Quyền tạo Ticket | Assignment | Chỉ tài xế active của xe mới được tạo phiếu báo hỏng. |
| **BR-ASSIGN-04** | Điều hướng theo địa điểm | Assignment | Xe ở chi nhánh nào chỉ do Cơ giới & KTV chi nhánh đó xử lý. |
| **BR-TICKET-01** | Liên kết Ticket - Xe | Ticket | Một phiếu sửa chữa chỉ liên kết với duy nhất một xe. |
| **BR-TICKET-02** | Báo hỏng đa lỗi | Ticket | Một ticket cho phép báo nhiều lỗi đồng thời. |
| **BR-TICKET-03** | Chặn tạo phiếu trùng | Ticket | Không tạo phiếu mới nếu xe đang có phiếu chưa đóng. |
| **BR-TICKET-04** | Giới hạn số lỗi cấu hình | Ticket | Số lỗi tối đa trên ticket là giá trị cấu hình, mặc định = 5. |
| **BR-INSP-01** | Quyền kiểm tra sơ bộ | Inspection | Chỉ Đội cơ giới được kiểm tra sơ bộ xe báo lỗi. |
| **BR-INSP-02** | Định tuyến sau kiểm tra | Inspection | Bắt buộc chuyển TECHNICAL_DIAGNOSIS hoặc REJECTED sau kiểm tra. |
| **BR-INSP-03** | Yêu cầu lý do từ chối | Inspection | Bắt buộc nhập lý do khi Cơ giới từ chối duyệt ticket. |
| **BR-QUEUE-01** | Tự động xếp hàng | Queue | Đẩy xe vào hàng chờ khi slot sửa chữa trong xưởng bị đầy. |
| **BR-QUEUE-02** | Hàng chờ duy nhất | Queue | Một xe chỉ được xuất hiện tối đa một lần trong hàng chờ xưởng. |
| **BR-QUEUE-03** | Thứ tự ưu tiên FIFO | Queue | Hàng chờ mặc định theo cơ chế FIFO, trừ khi Manager override. |
| **BR-QUEUE-04** | Cảnh báo SLA hàng chờ | Queue | Cảnh báo Manager nếu xe nằm trong hàng chờ vượt quá 2 giờ. |
| **BR-MAT-01** | Quyền yêu cầu vật tư | Material | Chỉ KTV được gán phiếu mới được yêu cầu cấp vật tư. |
| **BR-MAT-02** | Quyền xuất kho vật tư | Material | Chỉ thủ kho vật tư mới được quyền duyệt xuất kho phụ tùng. |
| **BR-MAT-03** | Luồng cấp phát 3 bên | Material | Chuỗi bàn giao vật lý bắt buộc: Kho → Tài xế → KTV. |
| **BR-MAT-04** | Chặn sửa khi thiếu vật tư| Material | KTV chỉ được bắt đầu sửa khi đã nhận đủ vật tư từ lái xe. |
| **BR-MAT-05** | Quyền hủy yêu cầu vật tư | Material | KTV hoặc Manager được hủy yêu cầu vật tư khi chưa xuất kho. |
| **BR-MAT-06** | Quyền nhập kho vật tư | Material | Chỉ nhân sự Kho (Inventory) mới được quyền nhập kho tăng tồn. |
| **BR-MAT-07** | Quy trình trả vật tư thừa | Material | Tái cộng tồn kho sau khi Thủ kho xác nhận nhận lại phụ tùng thừa. |
| **BR-REP-01** | Quyền sửa xe | Repair | Chỉ KTV mới được bấm bắt đầu/kết thúc sửa xe. |
| **BR-REP-02** | Điều kiện sửa chữa | Repair | Chuyển REPAIRING khi có slot trống và vật tư đã nhận đủ; nhường slot nếu kho thiếu hàng. |
| **BR-REP-03** | Liên kết vật tư ticket | Repair | Tất cả phụ tùng xuất kho bắt buộc gắn với một ticket. |
| **BR-CLOSE-01** | Đóng phiếu sau nghiệm thu| Closure | Chỉ Cơ giới có quyền đóng phiếu sau nghiệm thu chạy thử đạt yêu cầu. |
| **BR-CLOSE-02** | Sửa lại khi nghiệm thu fail | Closure | Nghiệm thu không đạt thì chuyển ngược ticket về trạng thái đang sửa. |
| **BR-CLOSE-03** | Quy tắc mở lại phiếu | Closure | Chỉ đóng/mở lại phiếu trong khoảng thời gian gia hạn bởi Manager. |
| **BR-AUDIT-01** | Ghi vết trạng thái | Security & Audit | Tự động ghi nhật ký lịch sử thay đổi trạng thái của ticket. |
| **BR-AUDIT-02** | Ghi vết vật tư | Security & Audit | Lưu vết chi tiết người giao, nhận, timestamp của các khâu vật tư. |

---

### 2. Mô tả Logic Xác thực (Validation Pseudo-Logic)
Ví dụ mô tả thuật toán kiểm tra nghiệp vụ cho quy tắc `BR-TICKET-03` (Chặn tạo trùng lặp ticket):

```text
Validation Logic BR-TICKET-03 (Chặn tạo trùng lặp ticket):
=========================================================
INPUT: vehicle_id
CHECK:
    Query all repair tickets where:
        ticket.vehicle_id == vehicle_id AND
        ticket.status IN ["REPORTED", "INITIAL_INSPECTION", "TECHNICAL_DIAGNOSIS", "WAITING_QUEUE", "REPAIRING"]
    IF any ticket is found:
        Raise ValidationError: "Không thể tạo phiếu mới. Phương tiện hiện đang có phiếu sửa chữa #ID chưa đóng."
        Block the operation.
    ELSE:
        Allow the operation to proceed.
```
