# FixTrack — Canonical Repair State Machine

**Status:** Approved for P0 implementation  
**Version:** 1.0  
**Effective date:** 2026-07-11
**Supersedes:** Conflicting legacy SSS transition rules for the covered scope  
**Implementation baseline:** Current FastAPI backend

## State meanings

| State | Meaning |
|---|---|
| `reported` | Driver submitted an incident. |
| `inspecting` | Vehicle is being inspected or diagnosed. |
| `waiting_queue` | Vehicle is waiting for inspection or repair dispatch. Queue stage distinguishes the purpose. |
| `waiting_parts` | Repair work is blocked by required materials. |
| `repairing` | Assigned technician is repairing the vehicle. |
| `completed` | Technical work is complete and awaiting fleet acceptance. |
| `closed` | Fleet accepted the repair and closed the ticket. Terminal. |
| `rejected` | Ticket was rejected with a reason. Terminal. |

## Canonical transitions

| From | Action | To | Normal role |
|---|---|---|---|
| `reported` | `queue_for_inspection` | `waiting_queue` with stage `inspection` | `co_gioi` |
| `reported` | `inspect` | `inspecting` | `co_gioi` |
| `reported` | `reject` | `rejected` | `co_gioi` |
| inspection queue | `inspect` | `inspecting` | `co_gioi` |
| `inspecting` | `reject` | `rejected` | `co_gioi` |
| `inspecting` | `route_queue` | repair queue | `co_gioi` |
| `inspecting` | `request_required_materials` | `waiting_parts` | `ky_thuat` |
| `inspecting` | `start_repair` | `repairing` | assigned `ky_thuat` |
| repair queue | `start_repair` | `repairing` | assigned `ky_thuat` |
| `repairing` | `request_additional_materials` | `waiting_parts` | assigned `ky_thuat` |
| `waiting_parts` | `route_queue` | repair queue | `co_gioi` or audited `admin` override |
| `waiting_parts` | `start_repair` | `repairing` | assigned `ky_thuat` |
| `repairing` | `complete_repair` | `completed` | assigned `ky_thuat` |
| `completed` | `accept_repair` | `closed` | `co_gioi` |
| `completed` | `reject_repair` | `repairing` | `co_gioi` |

## Queue metadata

One `waiting_queue` state supports two purposes and therefore requires persisted metadata:

- `queue_stage`: `inspection` or `repair`.
- `queue_reason`: `no_inspection_slot`, `no_repair_slot`, `lacking_materials`, `manual_dispatch` or `other`.
- Dispatch state must distinguish at least `blocked` and `ready` if material-blocked tickets are represented in the queue.

The current queue model does not contain this metadata. A schema proposal is required before implementing these transitions.

## Common transition guarantees

Every transition must verify source state, action, role, branch, assignment, business/material preconditions, required reason and client OCC version. A successful transition must consistently update state and version, append status and audit records, update relevant timestamps and record the existing domain event in the same transaction.

No route or unrelated service may directly assign `RepairRequest.trang_thai`.

## Special rules

- Direct rejection from `reported` is allowed but requires a reason and audit.
- `repairing → waiting_parts` preserves the active technician assignment.
- `completed → repairing` means failed acceptance/rework, requires a reason and preserves the assignment by default.
- `waiting_parts` does not automatically reserve or release a physical workshop slot in MVP.
