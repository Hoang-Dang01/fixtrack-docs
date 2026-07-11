# Backend Remediation Stage 4 — State Machine Execution Plan

**Status:** Approved for Stage 4 implementation  
**Version:** 2.0  
**Date:** 2026-07-11  
**Prerequisite:** Stages 2 and 3 implemented; SQLite suite currently passing.

## 1. Goal and boundaries

Centralize repair-ticket transitions without conflating repair state with dispatch queue state. Stage 4 must preserve FastAPI/SQLAlchemy/Alembic conventions and must not add Redis, outbox workers, attachment infrastructure or rewrite the legacy SSS.

Stage 4.0 proved that queue metadata and blocking-material intent cannot be represented safely by the current models. The migrations described by Gates A and B are now authorized.

## 2. Responsibility split

### RepairStateService

- Validates and applies `RepairRequest.trang_thai` transitions.
- Checks source state, action, actor role, branch, assignment, material/issue preconditions, required reason and OCC version.
- Updates ticket version, status log, audit, timestamps and vehicle state in the caller-owned transaction.
- Calls QueueService only for queue side effects tied to a successful ticket transition.
- Never commits.

### QueueService

- Owns QueueEntry lifecycle, metadata, active-entry constraints, priority/order and promotion/cancellation.
- Queue status is dispatch metadata, not `RepairRequest.trang_thai`.
- `BLOCKED → READY` does not automatically transition the ticket.
- Never commits.

Transaction ownership belongs to the request-scoped unit of work or outer caller. `get_db` already rolls back on exceptions.

## 3. Stage 4.0 — Mandatory read-only baseline

Before code or migration changes:

1. Inventory every direct assignment to repair status.
2. Trace every caller of `change_repair_status`.
3. Inventory QueueEntry fields, creation paths, update paths and cardinality.
4. Determine whether queue entries are current state, history, or both.
5. Check active-entry uniqueness and existing duplicate data.
6. Inventory WorkAssignment lifecycle and all `ngay_hoan_tat` mutations.
7. Inventory vehicle-status synchronization and open-ticket checks.
8. Identify the real technical-handover data currently available.
9. Confirm quantity sources of truth:
   - Approved: `MaterialRequestItem.so_luong_cap`.
   - Issued: sum of item-linked `OUTFLOW`.
   - Driver received: sum of item-linked `DRIVER_RECEIPT`.
   - Technician received: sum of item-linked `TECH_HANDOVER`.
   - Returned: sum of item-linked `RETURN`, scoped by warehouse when applicable.
10. Confirm event implementation is the existing in-session DomainEventBus, not a durable outbox.

Output: `STAGE_4_STATE_MACHINE_BASELINE.md` containing file/function, current behavior, canonical mismatch, severity and migration need.

## 4. Decision gates after baseline

### Gate A — Queue schema

Only propose fields that do not duplicate existing semantics. Candidate fields:

- `queue_stage`: `inspection` or `repair`.
- `queue_reason`: canonical reason string.
- `status`: reuse existing VARCHAR with canonical lifecycle values.
- `override_reason`: only if no existing audit field can preserve it.

New values follow project convention: Python constants/Enum with VARCHAR columns, not native PostgreSQL ENUM.

Legacy `queue_stage` and `queue_reason` start nullable. Deterministic rows are backfilled; ambiguous rows remain NULL and appear in a reconciliation report. Unreconciled rows cannot be promoted.

### Gate B — Blocking materials — DECIDED

Add `MaterialRequest.is_blocking` as a non-null boolean with application/database default `true`.

- `true`: the request represents materials required to continue the current repair scope and may drive `inspecting/repairing → waiting_parts`.
- `false`: supplementary, optional or reserve materials; creating the request does not stop repair.
- Legacy material requests are backfilled to `true`, matching historical behavior.
- The create API may explicitly set the flag; omission means `true`.
- Read schemas expose the flag.
- Changing the flag after approval/issue is not part of Stage 4.

### Gate C — Technical handover evidence

Use an existing structured field if available. If no suitable field exists, propose the smallest MVP field such as non-empty technical handover notes. Do not build attachment/document infrastructure in Stage 4.

## 5. Canonical queue lifecycle

```text
WAITING → BLOCKED | READY
BLOCKED → READY
READY → PROMOTED
WAITING | BLOCKED | READY → CANCELLED
```

- `PROMOTED` and `CANCELLED` are terminal for one queue-entry lifecycle.
- Active QueueEntry statuses are `WAITING`, `BLOCKED` and `READY`.
- Re-entry/rework creates a new queue entry; it does not reopen a promoted entry.
- One ticket may have history across stages but at most one active entry per `queue_stage`.
- A ticket cannot have active inspection and repair entries simultaneously unless a later explicit rule authorizes it.
- Leaving the corresponding queue step promotes or cancels the active entry in the same transaction.
- Inspection entries become `PROMOTED` only when the ticket successfully enters `inspecting`; repair entries become `PROMOTED` only when it successfully enters `repairing`.

## 6. Canonical repair transitions

| From | Action | To | Normal actor |
|---|---|---|---|
| `reported` | queue for inspection | `waiting_queue [queue_stage=inspection]` | `co_gioi` |
| `reported` | inspect | `inspecting` | `co_gioi` |
| `reported` | reject | `rejected` | `co_gioi` |
| `waiting_queue [queue_stage=inspection]` | inspect | `inspecting` | `co_gioi` |
| `inspecting` | reject | `rejected` | `co_gioi` |
| `inspecting` | route to repair queue | `waiting_queue [queue_stage=repair]` | `co_gioi` |
| `inspecting` | request blocking materials | `waiting_parts` | assigned `ky_thuat` |
| `inspecting` | start repair | `repairing` | assigned `ky_thuat` |
| `waiting_queue [queue_stage=repair]` | start repair | `repairing` | assigned `ky_thuat` |
| `repairing` | request additional blocking materials | `waiting_parts` | assigned `ky_thuat` |
| `waiting_parts` | route to repair queue | `waiting_queue [queue_stage=repair]` | `co_gioi` or audited admin override |
| `waiting_parts` | start repair | `repairing` | assigned `ky_thuat` |
| `repairing` | complete repair | `completed` | assigned `ky_thuat` |
| `completed` | accept repair | `closed` | `co_gioi` |
| `completed` | reject repair | `repairing` | `co_gioi` |

`closed` and `rejected` are terminal.

## 7. Transition validation

Every ticket transition validates:

- Exact source state and action.
- Actor role and global-admin override scope.
- Same-branch access for non-admin actors.
- Active assignment belongs to the ticket.
- Admin acting for a technician does not become the assigned technician; audit records both actual actor and assigned technician context.
- Required reason/override reason.
- Queue stage and READY status when starting from a queue.
- Quantity-based blocking-material readiness using Stage 3 ledger data.
- Required repair-issue completion and technical-handover evidence.
- Client OCC version before any side effect.

Admin may substitute actor authorization but cannot bypass source state, assignment existence/relationship, quantities, OCC, transaction integrity or audit.

## 8. Assignment lifecycle

- `waiting_parts`: preserve active assignment.
- `repairing → completed`: preserve active assignment for possible rework.
- `completed → repairing`: preserve active assignment.
- `completed → closed`: complete all active assignments with `ngay_hoan_tat`.
- `reported/inspecting → rejected`: close any anomalous active assignments while preserving history.
- Inspection queue may have no technician assignment.
- Repair queue preserves an existing assignment; if none exists, start-repair requires assignment first.

## 9. Vehicle-state recalculation

Do not hard-code `hoat_dong` merely because one ticket is rejected or closed. Add one helper that recalculates from all non-terminal repair requests for the vehicle:

- Any `repairing` ticket → `dang_sua`.
- Otherwise any open repair ticket → `hong`.
- No open repair ticket → `hoat_dong`, unless an independent vehicle rule keeps it stopped.

The helper must execute in the same transaction as the ticket transition.

## 10. Atomic side effects

Within one caller-owned transaction:

- Validate OCC before mutation.
- Update repair state and increment version.
- Append RepairStatusLog and AuditLog.
- Update timestamps.
- Invoke QueueService for transition-related queue changes.
- Apply assignment lifecycle.
- Recalculate vehicle state.
- Publish only the existing in-session DomainEventBus event.

No outbox, worker or external network call is added.

## 11. Eliminate direct status mutation

After the transition service is ready, replace direct ticket-state assignment in material, repair, work-assignment and queue flows. Queue-only updates such as priority, `BLOCKED → READY` and display ordering remain in QueueService and do not trigger ticket transitions automatically.

## 12. Tests

### State and OCC

- Every valid canonical transition.
- Wrong source, role, branch, assignment and missing reason.
- Terminal-state protection.
- Same client version race; loser leaves no duplicate logs/audit.
- Rollback restores ticket, version, queue, vehicle, assignment and audit state.

### Queue

- Queue lifecycle transitions.
- NULL legacy stage cannot promote.
- At most one active entry per ticket/stage.
- `BLOCKED → READY` leaves ticket state unchanged.
- Successful start promotes the correct entry atomically.
- Rework creates a new entry when queued again.

### Materials

- Blocking request moves inspection/repair to `waiting_parts` only under the decision from Gate B.
- Issued quantity alone is insufficient; required technician-handover quantity must be satisfied.
- Rejected required materials do not automatically unblock.
- `waiting_parts → repairing` uses item-level cumulative quantities.

### Assignment and vehicle

- Close completes assignment; completed/rework/waiting-parts preserve it.
- Admin override retains the real technician owner.
- Rejecting a duplicate ticket does not activate a vehicle with another open ticket.
- Closing the last open ticket recalculates the vehicle correctly.

### Migration

- Empty DB upgrade/downgrade.
- Deterministic backfill.
- Ambiguous row remains NULL.
- Unreconciled row cannot promote.
- Downgrade metadata-loss warning documented.

## 13. Verification gates

1. `compileall` and `git diff --check` pass.
2. Entire SQLite integration suite passes.
3. PostgreSQL migration upgrade → downgrade → upgrade passes.
4. PostgreSQL queue uniqueness and `SELECT FOR UPDATE` behavior pass.
5. Repository search finds no direct repair-state assignment outside the controlled state service/model initialization/seed/migration.

## 14. Reports

- `STAGE_4_STATE_MACHINE_BASELINE.md` after read-only Stage 4.0.
- Migration proposal and decision record after Gates A–C.
- `BACKEND_REMEDIATION_STAGE_4_STATE_MACHINE_REPORT.md` after implementation and verification.

All three decision gates are now resolved. Stage 4 implementation may proceed in the documented order, with migration and tests reported separately.
