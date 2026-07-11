# Backend Remediation Stage 2 — Security and Ownership Report

**Status:** Verified  
**Date:** 2026-07-11  
**Scope:** Change Set A only

## Outcome

Implemented P0 security and ownership controls without changing database schema, inventory partial flows or repair-state transitions.

## Files changed

- `backend/app/services/branch_policy.py`: non-admin access now fails closed when user or vehicle branch is missing; admin remains explicit global access.
- `backend/app/services/material_access.py`: new service-layer ABAC, assignment-ticket validation and handover authorization.
- `backend/app/api/routes/material_requests.py`: list/detail scope, creation validation and handover authorization now call the service layer; material request creation is limited to technician/admin roles.
- `backend/tests/test_security_ownership.py`: new security integration/service tests.
- Canonical documents: added approval/version metadata, precedence rules and explicit global-admin branch policy.

## Canonical rules applied

- Role kho remains `vat_tu`.
- Non-admin branch access is fail-closed.
- Admin has explicit global branch visibility but cannot bypass ticket-assignment relationships.
- Material-request list/detail is no longer visible to every active account.
- A material request requires an active assignment belonging to the same repair ticket.
- A non-admin creator must be the technician on that active assignment.
- Handover confirmation requires the active assigned technician; authorization exists in a service and cannot be bypassed by directly calling the domain operation through the route.

## ABAC behavior

- `admin`: global list/detail access.
- `lai_xe`: only requests for tickets reported by that driver, within a determinable matching branch.
- `ky_thuat`: only tickets with an active assignment to that technician, within the matching branch.
- `co_gioi` and `vat_tu`: branch-scoped list/detail access.
- Missing user or vehicle branch denies non-admin access.

## Database migration

**No.** Change Set A only changes policy, query scoping and service validation.

## Transaction/session boundary

Authorization and mutations use the request-scoped SQLAlchemy session injected by `get_db`. The new authorization service does not open or commit a separate session. Existing route commits remain unchanged in this change set.

## Tests added

- Same-branch access.
- Missing user branch denied.
- Missing vehicle branch denied.
- Cross-branch list returns no records.
- Cross-branch detail returns 403.
- Assignment belonging to another ticket is rejected.
- Unassigned technician cannot confirm handover.
- Direct service call cannot bypass handover authorization.
- Admin cross-branch access succeeds.
- Former/inactive technician cannot confirm handover.

## Test execution

Bundled Python was configured from `backend/requirements.txt`. Final regression command:

```text
<bundled-python> -m pytest -q -p no:cacheprovider
```

Final result:

```text
19 passed in 1.16s
```

The first runtime run exposed fixtures that relied on PostgreSQL-generated ticket codes and omitted branch data. The fixtures were corrected with explicit `ma_phieu` values and branch assignments. Security tests and the complete existing suite now pass under the SQLite integration-test runtime.

## Risks remaining

- Warehouse branch ownership is not yet modeled in material-request access beyond the ticket/vehicle branch. This needs Inventory Change Set B review.
- Existing users/vehicles with NULL branch will now be denied as intended; production data must be audited before deployment.
- Admin override reason/audit for exceptional handover is not implemented because the existing endpoint has no override-reason payload. This remains a follow-up requiring an API contract decision.
- Repair/work-assignment routes outside material-request scope retain their existing policies and will be addressed only where required by Change Set C.

## Rollback

Revert the policy/service/route changes. No schema or data rollback is required.

## Scope confirmation

No inventory quantity logic, state-machine transition, queue schema, ledger schema or legacy SSS chapter was changed.
