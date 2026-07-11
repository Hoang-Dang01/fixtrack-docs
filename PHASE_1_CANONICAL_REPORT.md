# Giai đoạn 1 — Canonical Documentation Report

**Trạng thái:** Hoàn thành  
**Ngày:** 2026-07-11

Tên “Giai đoạn 1” trong báo cáo này có nghĩa là **Backend Remediation Stage 1 — Canonical Baseline**, không phải Product Phase 1 trong SSS cũ.

## Thứ tự ưu tiên tài liệu

1. `CANONICAL_BUSINESS_RULES.md` quyết định điều kiện nghiệp vụ.
2. `CANONICAL_STATE_MACHINE.md` quyết định transition và actor.
3. `CANONICAL_ROLES.md` quyết định role và phạm vi quyền.
4. `TECHNICAL_BASELINE.md` quyết định giới hạn kỹ thuật.
5. SSS cũ chỉ dùng để tham khảo khi không xung đột.

Tài liệu canonical mô tả quyết định cần triển khai, không tuyên bố backend đã hỗ trợ sẵn.

## Kết quả

Đã tạo bốn tài liệu canonical tạm thời: technical baseline, roles, business rules và state machine. Chưa sửa backend, database, migration hoặc toàn bộ SSS.

## Hiệu chỉnh so với đầu vào

- Đổi role canonical từ `kho_vat_tu` thành `vat_tu` vì đây là enum thật trong `backend/app/enums.py`.
- Giữ audit hash chain trong phạm vi regression vì implementation, migration và test đã tồn tại.
- Không bắt buộc outbox/worker trong P0 vì runtime chưa có.
- Xác định queue stage/reason cần schema proposal; model hiện tại chỉ có request, position, priority, status và timestamps/version.
- Xác định inventory ledger hiện tham chiếu material request qua chuỗi `reference_id` nhưng không có foreign key đến material-request item. Ledger hiện tại chưa đủ chắc chắn cho nhiều lần giao/trả nếu một phiếu có dòng trùng material.

## Migration required

**Có, nhưng chưa thực hiện.** Dự kiến tối thiểu:

- Queue: `queue_stage`, `queue_reason` và dispatch status phù hợp.
- Inventory transaction: foreign key/reference đến material request item hoặc operation-line model để hỗ trợ partial/multiple delivery/return chính xác.
- Có thể cần bảng/field lưu quyết định vật tư bị từ chối và technical handover; sẽ xác nhận trong Change Set B/C.

Không thay enum role hoặc repair status.

## Change Set A — Security and Ownership

- Fail-closed branch policy.
- Material-request ABAC list/detail.
- Validate assignment-ticket relationship and active technician.
- Enforce technician ownership for create/handover.
- Move reusable authorization into service layer.
- Migration: không dự kiến.
- Rollback: revert service/policy changes; no data transformation.

## Change Set B — Inventory Integrity

- Validate item membership and approval limits.
- Introduce per-operation idempotency.
- Support reliable cumulative delivery/return and original warehouse validation.
- Preserve row locking and atomic transaction boundaries.
- Migration: dự kiến có do ledger thiếu item-level reference.
- Rollback: downgrade schema after confirming no new item-level transactions require preservation; backup required before production migration.

## Change Set C — State Machine Consistency

- Route all repair status mutations through the shared service.
- Add canonical queue/material/rework transitions.
- Apply branch, assignment, reason, OCC, status log and audit consistently.
- Add queue stage/reason support.
- Migration: dự kiến có cho queue metadata; repair enum remains unchanged.
- Rollback: downgrade queue columns only after mapping affected entries to their prior generic status.

## Tests to add

- Branch missing/same/cross and audited admin override.
- Material list/detail ABAC and direct service authorization.
- Assignment wrong ticket, inactive assignment and unassigned technician.
- Unknown approval item and approved quantity over requested.
- Partial/multiple issue and return, over-return, return before issue, wrong warehouse and idempotent retry.
- Concurrent stock issue and atomic rollback.
- Every canonical transition, role failure, missing reason and OCC conflict.
- Audit chain continuity after successful and rolled-back P0 operations.

## Open technical questions before schema changes

- Whether to store cumulative delivered/returned quantities on each item or derive them exclusively from a normalized item-linked ledger. Preferred proposal: normalized immutable ledger plus query aggregation.
- Whether material exception decisions belong on each item or in a separate decision-history table. Preferred proposal: history table for auditability.
- Whether existing queue rows can be reliably backfilled to inspection/repair stages from ticket status/history. A data audit is required before migration.

## Verification status

Document consistency was checked against the repository. Automated tests were not run because the current shell cannot resolve `python` or `pytest`. Docker-based testing must be restored before a code change set is considered complete.
