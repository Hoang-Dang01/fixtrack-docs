# Backend Remediation Stage 4 — State Machine Report

**Status:** Implemented and locally verified — PostgreSQL deployment verification pending  
**Date:** 2026-07-11

## Outcome

Stage 4 centralized runtime repair-state mutation in `app/services/repair.py`, separated queue lifecycle into QueueService, added queue metadata and blocking-material classification, and aligned assignment/vehicle side effects with canonical rules.

## Files changed

- `backend/app/services/repair.py`
- `backend/app/services/queue_service.py` (new)
- `backend/app/services/event_bus.py`
- `backend/app/services/material.py`
- `backend/app/api/routes/repair_requests.py`
- `backend/app/api/routes/material_requests.py`
- `backend/app/api/routes/work_assignments.py`
- `backend/app/api/routes/queue.py`
- `backend/app/models/material_request.py`
- `backend/app/models/queue_entry.py`
- `backend/app/schemas/material_request.py`
- `backend/app/schemas/queue_entry.py`
- `backend/alembic/versions/20260711_0002_state_queue_metadata.py` (new)
- `backend/tests/test_state_machine_stage4.py` (new)

## Migration

Revision `20260711_0002` adds:

- `material_requests.is_blocking BOOLEAN NOT NULL DEFAULT true`.
- Nullable `queue_stage`, `queue_reason`, `override_reason`.
- PostgreSQL partial unique index for one active entry per request/stage.

Legacy queue metadata remains NULL for reconciliation and cannot be promoted through QueueService.

## State service

Added canonical actions for inspection queue and blocking materials, plus missing queue/material transitions. Transition validation now includes branch and active technician assignment for technician actions. Start from `waiting_parts` uses item-linked `TECH_HANDOVER` totals against approved blocking quantities.

OCC expected version is mandatory for every runtime transition, including work-assignment and blocking-material callers. Admin cannot bypass OCC.

Material readiness uses net technician custody: cumulative `TECH_HANDOVER` minus cumulative valid `RETURN`, compared with approved blocking quantity.

Direct runtime ticket-state assignment search now finds only the controlled assignment inside `change_repair_status`; initialization/defaults remain separate.

## Queue lifecycle

QueueService owns active lookup, enqueue, lifecycle validation and promotion. Ticket transitions create READY entries and promote the matching inspection/repair entry only when the business transition succeeds. Queue priority changes remain independent of repair state.

## Assignment and vehicle

- `waiting_parts`, `completed` and rework preserve assignment.
- `closed` and `rejected` complete active assignments.
- Completion requires non-empty `ket_qua_xu_ly` for active assignments.
- Vehicle state is recalculated from all open tickets; closing/rejecting one ticket no longer blindly activates the vehicle.

## Transaction behavior

State/queue/assignment/vehicle/audit/notification records share the caller-owned SQLAlchemy transaction. State and queue services do not commit. DomainEventBus listener errors now propagate so the outer unit of work rolls back instead of silently committing without required audit.

## Verification

```text
<bundled-python> -m pytest -q -p no:cacheprovider
25 passed in 1.81s
```

`compileall` and `git diff --check` passed. Repository search found no direct runtime repair-state assignment outside `app/services/repair.py`.

Additional hardening coverage includes mandatory OCC and forced audit-listener failure rollback. The rollback test confirms ticket state/version and vehicle state remain unchanged.

## Blocking-material policy

- Request-level classification is intentional for MVP; one material request cannot mix blocking and non-blocking items.
- Legacy and new requests default to `is_blocking=true`.
- Only admin may explicitly create `is_blocking=false`; technicians cannot use the flag to bypass `waiting_parts`.
- Item-level classification remains a future enhancement.

## Generic status endpoint

The compatibility endpoint still accepts a target status but maps it to a canonical `RepairAction` before calling the same state service. It cannot assign status directly or bypass state/role/assignment/OCC validation. Replacing its public contract with action-based input remains a P1 API cleanup.

## Remaining deployment gates

- Run Alembic upgrade/downgrade/upgrade against PostgreSQL 15.
- Audit existing queue duplicates before creating the partial unique index.
- Reconcile legacy queue rows with NULL stage/reason.
- Verify PostgreSQL concurrency and partial-index behavior.
- Expand tests for every canonical transition and forced mid-transaction audit failure before production release.

## Scope confirmation

No Redis, worker, outbox, external notification, attachment workflow or full legacy SSS rewrite was introduced.
