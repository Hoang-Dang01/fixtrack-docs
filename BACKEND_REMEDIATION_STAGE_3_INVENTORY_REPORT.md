# Backend Remediation Stage 3 — Inventory Integrity Report

**Status:** Verified in SQLite integration suite; PostgreSQL concurrency/migration verification pending  
**Date:** 2026-07-11  
**Scope:** Change Set B

## Outcome

Inventory approval and return flows now use item-level ledger references, per-operation idempotency and cumulative return validation. A reversible Alembic migration was added. Repair-state transitions were not redesigned in this stage.

## Files changed

- `backend/app/models/inventory_transaction.py`
- `backend/app/schemas/material_request.py`
- `backend/app/services/material.py`
- `backend/app/api/routes/material_requests.py`
- `backend/alembic/versions/20260711_0001_inventory_item_ledger.py`
- `backend/tests/test_sss_compliance.py`

## Migration

Revision `20260711_0001` adds nullable:

- `inventory_transactions.material_request_item_id`.
- `inventory_transactions.source_transaction_id`.
- Foreign keys and indexes for both fields.

Legacy OUTFLOW rows are backfilled only when a referenced request contains exactly one matching material-request item. Ambiguous legacy rows remain NULL and require data review; the migration does not guess.

Downgrade removes the indexes, foreign keys and columns. A production backup is required before upgrade/downgrade.

## Rules implemented

- Approval requires a client operation `idempotency_key`.
- Retrying an approval key does not create another stock mutation.
- Every approval payload item must belong to the material request.
- Approved quantity cannot exceed requested quantity.
- OUTFLOW ledger rows reference the exact material-request item.
- Return requests identify `item_id`, not only `material_id`.
- Return requires a unique client operation key.
- Retrying a return key is idempotent.
- Return is rejected when no valid item-linked OUTFLOW exists.
- Return warehouse must match an issue warehouse.
- Cumulative returned quantity cannot exceed cumulative issued quantity.
- RETURN rows reference the item and source issue transaction.
- Stock mutation continues to use row locking and the request SQLAlchemy transaction.

## API contract changes

Approval payload now requires:

```json
{"idempotency_key":"unique-operation-key","warehouse_id":1,"items":[]}
```

Return payload now requires item-level identity:

```json
{"idempotency_key":"unique-return-key","warehouse_id":1,"items":[{"item_id":10,"so_luong_tra":2}]}
```

This is intentionally stricter and requires corresponding frontend/client updates before deployment.

## Verification

- Python syntax compilation passed for `app`, `tests` and `alembic` using the bundled Python runtime.
- Pytest could not run because that bundled runtime does not include the `pytest` module.
- Docker test execution remains unavailable because Docker daemon is not running.

Commands attempted:

```text
<bundled-python> -m pytest -q
<bundled-python> -m compileall -q app tests alembic
```

The second command passed.

## Remaining limitations and P1 work

- Approval and physical issue are now separate operations. Repeatable item-level issue, driver receipt and technician handover endpoints are available.
- `so_luong_cap` is still a single approved quantity. The immutable item-linked ledger is now the source for cumulative issue/return quantities.
- Legacy ambiguous ledger rows cannot safely participate in item-level returns until manually reconciled.
- Return `source_transaction_id` records a source issue, while cumulative validity is checked across all issue rows. Exact allocation of one return across multiple source issues is future normalization.
- Warehouse branch authorization and warehouse-transfer approval remain future work.
- Full concurrency behavior must be verified against PostgreSQL; SQLite cannot validate `SELECT FOR UPDATE` semantics.

## Transaction and rollback

Stock, ledger, material-request version, audit event and existing ticket side effects use one request-scoped session and commit together. No external network call was introduced. `get_db` now explicitly rolls back on exceptions before closing the session.

The schema downgrade is structurally reversible but loses item/source linkage metadata created after upgrade; it is not fully data-reversible.

## Scope confirmation

No role enum, repair-status enum, queue schema, Redis/worker infrastructure or legacy SSS chapter was changed.

## Final B2 verification

Added `da_duyet`, approval-only quantity handling and repeatable endpoints:

- `POST /material-requests/{id}/issue`
- `POST /material-requests/{id}/receive-driver-items`
- `POST /material-requests/{id}/handover-items`

Final test result:

```text
20 passed in 1.20s
```
