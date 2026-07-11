# FixTrack SSS v2 — Backend Canonical Specification

**Status:** Implemented and locally verified  
**Version:** 2.0  
**Effective date:** 2026-07-11  
**Implementation:** FastAPI backend

## Document authority

For implemented backend behavior, precedence is:

1. This SSS v2 backend specification.
2. `CANONICAL_BUSINESS_RULES.md`.
3. `CANONICAL_STATE_MACHINE.md`.
4. `CANONICAL_ROLES.md`.
5. `TECHNICAL_BASELINE.md`.
6. Generated FastAPI OpenAPI for endpoint/schema details.
7. Legacy 38-chapter SSS as reference only.

Legacy NestJS, Prisma, TypeORM, BullMQ, Redis worker, S3, FCM, ClamAV and React Native descriptions are Future/Reference Architecture and do not describe the current implementation.

## Technical baseline

- FastAPI 0.115.6, Pydantic 2 and SQLAlchemy 2.
- Alembic migrations.
- PostgreSQL 15 production database.
- JWT access-token authentication.
- Pytest/HTTPX integration suite; SQLite is used only as a fast local test double.

## Roles

- `lai_xe`: Driver.
- `co_gioi`: Fleet Dispatcher / Vehicle Coordinator.
- `ky_thuat`: Repair Technician.
- `vat_tu`: Inventory Staff.
- `admin`: global Manager/System Administrator role for MVP.

Admin has cross-branch visibility but cannot bypass source state, entity relationships, quantity constraints, OCC, transaction integrity or audit.

## Repair state machine

Canonical states:

```text
reported, inspecting, waiting_queue, waiting_parts,
repairing, completed, closed, rejected
```

All runtime mutation of `RepairRequest.trang_thai` goes through the repair state service. OCC expected version is mandatory. `closed` and `rejected` are terminal.

Queue dispatch lifecycle is independent:

```text
WAITING → BLOCKED | READY
BLOCKED → READY
READY → PROMOTED
WAITING | BLOCKED | READY → CANCELLED
```

Queue entries distinguish `inspection` and `repair` stages.

## Assignment and vehicle rules

- Technician actions require an active assignment belonging to the ticket.
- Waiting for parts, completed and rework retain assignment.
- Closed/rejected complete active assignments.
- Completion requires non-empty technical result notes.
- Vehicle state is recalculated from every open ticket instead of one closing ticket.

## Materials and inventory

- Material request has request-level `is_blocking`, default true.
- Only admin may explicitly create a non-blocking request.
- Approval is separate from repeatable physical issue.
- Issue, driver receipt, technician handover and return are immutable item-level operations.
- Each operation requires a unique idempotency key.
- Cumulative quantities cannot exceed approved/issued/received limits.
- Blocking readiness uses net technician custody: `TECH_HANDOVER - RETURN`.
- Stock changes use warehouse-scoped ledger entries and row locking.

## Security

- Non-admin branch access fails closed when branch data is missing.
- Material-request list/detail uses SQL-level ABAC.
- Assignment-ticket relationship and active technician ownership are enforced in service logic.
- Notification records are owner-scoped.

## Audit and transactions

- Ticket state, OCC version, status log, audit, queue, assignment and vehicle side effects share one caller-owned transaction.
- Inventory stock, ledger, request state/version and audit share one transaction.
- Audit uses an SHA-256 hash chain.
- Required listener failures propagate and roll back the outer transaction.

## P1 backend capabilities

- Queue priority override with reason, expected version and audit before/after values.
- SQL pagination/filter parameters with stable ordering.
- Internal notification inbox and idempotent mark-read.
- Backward-compatible structured business-error envelope.

## Deferred architecture

- Entity-bound attachment metadata contract.
- S3/presigned upload, virus scanning and background workers.
- External push/email delivery.
- Full error-code taxonomy and paginated response envelope.
- Separate Manager/System Administrator roles.
- Item-level blocking classification.

## Verification status

Local integration suite passes. Final PostgreSQL migration, partial-index, locking and concurrency verification is intentionally consolidated into the deployment stage.

