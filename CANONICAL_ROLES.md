# FixTrack — Canonical Roles for MVP

**Status:** Approved for P0 implementation  
**Version:** 1.0  
**Effective date:** 2026-07-11
**Supersedes:** Conflicting legacy SSS rules for the covered scope  
**Implementation baseline:** Current FastAPI backend

| Code role | Business name | Core responsibility |
|---|---|---|
| `lai_xe` | Driver | Operates an assigned vehicle and reports incidents. |
| `co_gioi` | Fleet Dispatcher / Vehicle Coordinator | Assigns vehicles, receives tickets, coordinates queues and performs acceptance. |
| `ky_thuat` | Repair Technician | Diagnoses and repairs vehicles and receives issued materials. |
| `vat_tu` | Inventory Staff | Manages stock, issues materials and confirms returns. |
| `admin` | Manager / System Administrator | Temporarily combines business-management and system-administration authority. |

`kho_vat_tu` is a legacy documentation/UI label, not the persisted backend enum. The MVP uses `vat_tu`.

## Authorization principles

- A role grants permission to attempt an action; it does not bypass source state, assignment, branch, quantity, OCC or transaction rules.
- A technician-scoped action requires an active work assignment (`ngay_hoan_tat IS NULL`) for that ticket.
- Branch access is fail-closed when either relevant branch cannot be determined.
- `admin` may substitute for an allowed role only where explicitly supported.
- Administrative queue or exceptional business overrides require a reason and audit log.
- `admin` does not automatically bypass stock limits, assignment-ticket integrity, OCC or transaction integrity.
- In MVP, `admin` has explicit global cross-branch access for administration and supervision. This does not bypass entity relationships or other business preconditions.
