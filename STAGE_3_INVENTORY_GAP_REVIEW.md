# Stage 3 — Inventory Gap Review

**Status:** Remediated; SQLite verification complete  
**Date:** 2026-07-11

## 1. Current implementation findings

The current `/material-requests/{id}/approve` endpoint is not approval-only. It performs `approve_and_issue`: it selects a warehouse, locks stock, subtracts stock and writes OUTFLOW ledger rows. It supports partial quantity selection once, then moves the material request to `da_xuat`.

Current support matrix:

| Capability | Status |
|---|---|
| Partial approve-and-issue once | Implemented |
| Item-level OUTFLOW ledger | Implemented |
| Multiple cumulative return | Implemented statically |
| Per-warehouse return limit | Implemented statically |
| Multiple issue operations | Not implemented |
| Multiple delivery/handover quantities | Not implemented |
| Separate approved/issued/delivered quantities | Not implemented |

## 2. Canonical mismatch

Canonical P0 requires partial approval, multiple issue/delivery and multiple return. Moving multi-issue to P1 in the prior report was inconsistent. The canonical scope is not reduced by this review.

Decision: keep multi-issue/multi-delivery in P0. Supplemental Change Set B2 now separates approval from repeatable item-level issue, driver receipt and technician handover operations.

## 3. Corrections applied during review

- Idempotency now distinguishes same-key/same-payload from same-key/different-payload.
- Reusing a key across actor, request, operation type or warehouse returns conflict.
- Duplicate item IDs in approve/return payloads are rejected by schema validation.
- Returnable quantity is calculated by exact item and warehouse.
- Legacy ambiguous NULL ledger rows return `LEGACY_LEDGER_RECONCILIATION_REQUIRED`.
- Request-scoped DB dependency explicitly calls rollback on exceptions.
- Stage 3 report now states that downgrade loses new item-level metadata.

## 4. Idempotency scope

The database key remains globally unique. Application keys are expanded per line as `<operation-key>:<item-id>`. Existing rows are compared against request ID, reference type, actor, warehouse, item IDs and quantities before returning an idempotent result. A mismatch returns HTTP 409 through `BusinessError`.

## 5. Return allocation limitation

Validation is now per item and warehouse:

```text
returnable(item, warehouse)
= issued(item, warehouse) - returned(item, warehouse)
```

However, one RETURN ledger row still references one source OUTFLOW. A return spanning several issue rows is validated cumulatively but not allocated across all sources. Supplemental B2 should either create allocation rows or split a return into FIFO source-specific ledger rows.

## 6. Legacy reconciliation query

Rows requiring manual review can be listed with:

```sql
SELECT id, material_id, warehouse_id, reference_id, quantity, created_at
FROM inventory_transactions
WHERE transaction_type = 'OUTFLOW'
  AND reference_type = 'MATERIAL_REQUEST'
  AND material_request_item_id IS NULL
ORDER BY created_at, id;
```

The API does not guess a material-request item for these rows.

## 7. Transaction boundary

- One SQLAlchemy session per request.
- Route owns commit; inventory service does not commit internally.
- `get_db` now explicitly rolls back on exception.
- Row locks remain held until route commit/rollback.
- Audit/domain records use the same session.
- No network notification was added inside the transaction.

## 8. Required supplemental Change Set B2

- Separate approval from physical issue, or formally expose an `approve_and_issue` initial operation plus repeatable `issue` endpoint.
- Represent approved, issued and delivered quantities independently through immutable item-level operations.
- Support multiple issue operations with per-operation idempotency.
- Support quantity-aware driver receipt and technician handover instead of one status flip for the entire request.
- Persist exact FIFO return-to-issue allocation or split return ledger rows.
- Add PostgreSQL integration tests for concurrent issue and migration upgrade/downgrade.

This supplemental work may require another migration. No state-machine work should start before its design and implementation are complete.

## 9. Verification

`compileall` and `git diff --check` pass. Pytest/PostgreSQL runtime verification remains unavailable because pytest is not installed in the bundled Python and Docker daemon is not running.

## 10. Final status

**Verified in the SQLite integration suite: 20 tests passed.** PostgreSQL-specific row-lock and migration upgrade/downgrade verification remains required before production deployment.
