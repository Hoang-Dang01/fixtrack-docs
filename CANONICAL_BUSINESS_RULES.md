# FixTrack — Canonical Business Rules for MVP

**Status:** Approved for P0 implementation  
**Version:** 1.0  
**Effective date:** 2026-07-11
**Supersedes:** Conflicting legacy SSS business rules for the covered scope  
**Implementation baseline:** Current FastAPI backend

## Security and ownership

- Branch authorization is fail-closed when user or vehicle branch is missing.
- Material-request list/detail access is restricted by role, branch, ownership, repair request and active assignment.
- An assignment supplied to a material request must belong to the same repair ticket.
- Only the actively assigned technician may request or receive materials, except for an explicit audited admin override.
- Authorization must be enforced in service logic as well as route dependencies.
- `admin` has global cross-branch access in MVP, but remains subject to entity relationships, source state, quantity, OCC, transaction and audit rules.

## Queue

- Default order: priority descending, then entry creation time ascending.
- Admin override requires a reason and audit.
- `waiting_queue` supports inspection and repair stages; these must not be inferred only from ticket history.
- A material-blocked queue entry is not dispatchable until marked ready.

## Materials and inventory

For every item:

```text
approved_qty <= requested_qty
total_delivered_qty <= approved_qty
total_returned_qty <= total_delivered_qty
remaining_returnable = total_delivered_qty - total_returned_qty
```

- Unknown item IDs in an approval/issue/return payload are rejected.
- Stock mutation is locked or atomic and cannot produce a negative balance.
- Issue and return use the warehouse associated with the original stock movement unless a separately approved transfer exists.
- Each operation has its own idempotency key. Reusing a key repeats the prior result without another stock mutation; a new key permits another valid partial operation.
- Stock, ledger, material state, ticket state/version and audit/event records for one operation commit or roll back together.
- The current ledger references a material request but not a material-request item. This is insufficient to distinguish repeated lines for the same material and requires a schema proposal before full multi-delivery/multi-return support.

## Rejected or unavailable materials

A rejected request does not automatically unblock repair. The technician assesses whether the item remains required, has an approved alternative, is no longer required or cancels repair scope. In MVP, `admin` makes the final exception decision and records decision, reason, actor and timestamp.

`waiting_parts → repairing` is allowed only when no required item remains unresolved or blocked, required delivered quantities are satisfied, or an authorized alternative/no-longer-required decision exists. The acting technician must be actively assigned unless an audited admin override applies.

## Blocking-material classification

- Every material request has `is_blocking`.
- The default is `true` for backward compatibility and safety.
- Only a blocking request may transition a repair ticket from `inspecting` or `repairing` to `waiting_parts`.
- A non-blocking request may be approved, issued, delivered and returned without changing the repair-ticket state.
- For a blocking request, readiness is evaluated from approved item quantity and cumulative `TECH_HANDOVER` quantity in the item-linked ledger.

## Repair completion

- A ticket may enter `completed` only after required repair issues are completed and the technical handover record is supplied.
- `completed` means awaiting fleet acceptance, not business closure.
- Acceptance success closes the ticket.
- Acceptance failure returns it to `repairing`, requires a reason and retains the technician assignment by default.
- `closed` and `rejected` are terminal in MVP.
