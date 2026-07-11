# Stage 4.0 — State Machine Baseline

**Status:** Read-only baseline complete  
**Date:** 2026-07-11  
**Code changes:** None  
**Migration created:** None

## 1. Direct RepairRequest state mutation

| File/function | Current behavior | Bypasses state service | Severity |
|---|---|---:|---|
| `app/services/repair.py:change_repair_status` | Controlled transition, version/log/timestamp/vehicle/event | No | Canonical core |
| `app/services/material.py:approve_material_request` | Directly changes ticket `inspecting → waiting_parts` after legacy approve-and-issue | Yes | P0 High |
| `app/api/routes/repair_requests.py:create_request` | Initializes new ticket as `reported` | Initialization only | Allowed |
| Tests/seed/migrations | Construct fixtures or defaults | Not runtime transition | Allowed with review |

The legacy `approve_material_request` function remains present even though Stage 3 routes use newer approval/issue functions. It can still be called directly and must be removed or routed through RepairStateService during implementation.

## 2. Callers of `change_repair_status`

| Caller | Actor | Branch check | Assignment check | OCC input | Transaction owner |
|---|---|---|---|---|---|
| Generic `/repair-requests/{id}/status` | Current user | Yes | No | Payload version | Route commit; `get_db` rollback |
| Inspect/reject/queue/start/complete/accept/rework routes | Current user | Yes | Mostly role only | Endpoint payload where defined | Route commit |
| `work_assignments.start_assignment` | Current user | Yes | Actor equals assigned tech unless admin | None | Route commit |
| `work_assignments.complete_assignment` | Current user | Yes | Actor equals assigned tech unless admin | None | Route commit |

Findings:

- OCC is optional at service level and absent from work-assignment calls.
- Assignment validation is implemented in work-assignment routes, not centrally in the transition service.
- Admin bypasses role in the service but assignment context is not recorded in audit payload.
- `change_repair_status` does not commit. Caller owns commit; `get_db` explicitly rolls back exceptions.

## 3. QueueEntry baseline

Current columns: `id`, `request_id`, `position`, `priority`, `entered_at`, `status`, `version`.

Current observed status values: `WAITING` and `PROMOTED`. No canonical constants or DB check constraint exist.

Creation/update paths:

- Repair queue route creates an entry after a ticket transition.
- Acceptance route marks a matching `WAITING` entry `PROMOTED` even though acceptance is not entry into inspection/repair.
- Queue promote endpoint changes priority only; its name does not promote lifecycle status.
- GET queue recalculates and mutates ORM `position` values for display without committing.

Current semantics are mixed current-state/history:

- Old `PROMOTED` rows are retained as history.
- `WAITING` rows represent active queue state.
- There is no `BLOCKED`, `READY` or `CANCELLED` lifecycle.

Indexes/constraints:

- Index on `request_id` only.
- No uniqueness preventing multiple active entries for one ticket.
- No stage metadata, reason, override reason, promoted/cancelled timestamp or active partial unique index.

Equivalent-field result:

- `priority` already exists.
- `status` exists and should be reused/expanded.
- No equivalent for `queue_stage`, `queue_reason` or `override_reason`.

Actual production duplicate-active data cannot be proven from repository files alone; a PostgreSQL data audit is required. The checked-in backup databases are not an authoritative deployment source.

## 4. WorkAssignment lifecycle

Active currently means `ngay_hoan_tat IS NULL`; no cancelled/deleted flag exists.

Mutation paths:

- Generic assignment patch can set `ngay_hoan_tat`.
- `complete_assignment` sets it before potentially transitioning the ticket to `completed`.

Canonical mismatches:

- Current completion closes assignments at `repairing → completed`, but canonical policy requires keeping them active for possible rework until `closed`.
- `completed → repairing` does not reactivate a completed assignment.
- `closed` transition does not centrally close active assignments.
- `rejected` transition does not close anomalous active assignments.
- `waiting_parts` currently has no centralized assignment lifecycle side effect.

Migration is not required for the canonical lifecycle; service/refactor changes are sufficient.

## 5. Vehicle synchronization

Mutation paths:

- Ticket creation directly sets vehicle to `hong`.
- Repair state service maps `reported → hong`, `repairing → dang_sua`, and `rejected/closed → hoat_dong`.
- Vehicle admin CRUD can also update `tinh_trang` independently.

There is no query that recalculates vehicle state from all open repair requests. Closing or rejecting one ticket can therefore set `hoat_dong` while another open ticket still exists. A centralized recalculation helper is required; no schema migration is needed.

## 6. Technical handover evidence

Existing evidence:

- `WorkAssignment.ket_qua_xu_ly` is free-text repair-result data.
- `InspectionRecord` stores inspection type, findings and decision.
- Generic `Attachment` can reference repair entities but no route-level completion requirement uses it.
- `TicketIssue` has no completion/resolution field.

Gate C conclusion: enough evidence for an MVP without new attachment infrastructure. The smallest existing completion precondition is non-empty `ket_qua_xu_ly` on relevant active assignment(s). Issue-level completion cannot be proven with the current TicketIssue model and would require a later schema decision if mandatory.

## 7. Material quantity sources

Actual transaction names in current code:

- Approval: `MaterialRequestItem.so_luong_cap`; approval is not a ledger transaction.
- Issue: `OUTFLOW` with `reference_type=MATERIAL_ISSUE` in the Stage 3 repeatable operation.
- Driver receipt: `DRIVER_RECEIPT` with `reference_type=MATERIAL_DRIVER_RECEIPT`.
- Technician handover: `TECH_HANDOVER` with `reference_type=MATERIAL_TECH_HANDOVER`.
- Return: `RETURN`.

All new operations link to `material_request_item_id`; stock issue/return is warehouse-scoped. The readiness formula currently provable is:

```text
required_qty = approved blocking quantity
technician_received_qty = SUM(TECH_HANDOVER.quantity by item)
ready when technician_received_qty >= required_qty for every blocking item
```

This formula cannot be activated until Gate B defines which requests/items are blocking. Returned quantities affect custody/usage reporting but do not by themselves undo evidence that a technician received material; a later business rule may need to distinguish returned-unused before repair starts.

## 8. DomainEventBus

Implementation is synchronous in-memory publish/listen using the same SQLAlchemy session.

Handlers:

- Audit listener appends AuditLog to the same session.
- Notification listener appends an in-database Notification row to the same session.

No handler performs an external network request or commits a separate session. Handler exceptions are caught and logged instead of aborting the business transaction; this can allow a transition to commit without its audit/notification side effect and is a P0 transaction-consistency concern. Stage 4 should let audit failures propagate or invoke audit directly as a required side effect.

## 9. Decision gates

### Gate A — Queue schema

**Evidence sufficient; migration candidate required.** Existing model lacks stage/reason/lifecycle and active-entry uniqueness. Proposed migration must keep legacy stage/reason nullable and use VARCHAR convention. No migration has been created.

### Gate B — Blocking materials

**Decision completed after baseline.** Add request-level `MaterialRequest.is_blocking`, non-null and default `true`. Legacy rows backfill to `true`. Only blocking requests may cause `waiting_parts`.

### Gate C — Technical handover

**Evidence sufficient for MVP.** Use non-empty `WorkAssignment.ket_qua_xu_ly` as the minimum technical completion record. No migration required. If per-issue resolution becomes mandatory, TicketIssue needs future fields/table.

## 10. Migration need

- Queue metadata/active constraint: candidate and likely required after approval.
- Blocking-material flag: depends on Gate B choice.
- Technical completion notes: none for MVP.
- Assignment/vehicle/state-service refactor: none.

## 11. Baseline test evidence

Last verified command before this baseline:

```text
<bundled-python> -m pytest -q -p no:cacheprovider
```

Result: `20 passed in 1.20s`, executed 2026-07-11. No tests were reported skipped. Baseline 4.0 did not modify or rerun tests because it is read-only.

## 12. Authorized next step

No code changed. No migration created.

All gates are resolved. Stage 4 migration and implementation are authorized according to `STAGE_4_EXECUTION_PLAN.md`; completion still requires migration review and the full verification gates.
