# FixTrack — Backend API Handoff

**Audience:** Frontend/client developers  
**Source of truth:** FastAPI OpenAPI at `/docs` and `/openapi.json`

## Authentication

Use Bearer JWT from `POST /auth/login`. Current user: `GET /auth/me`.

## Important contracts

### Repair transitions

- Every transition requires the current ticket `version`/expected version in its payload where exposed.
- A `409` means stale OCC version or invalid business transition.
- Prefer action-specific endpoints (`inspect`, `reject`, `route-queue`, `accept-repair`, `reject-repair`). Generic `/status` is compatibility-only.

### Material workflow

1. Create material request with `assignment_id`, item quantities and `is_blocking`.
2. Approve quantities at `/material-requests/{id}/approve` with an idempotency key.
3. Issue one or more times at `/issue` using `item_id`, quantity, warehouse and operation key.
4. Driver confirms item quantities at `/receive-driver-items`.
5. Assigned technician confirms item quantities at `/handover-items`.
6. Inventory returns unused quantities at `/return` using item ID, warehouse and operation key.

Retries reuse the same key and identical payload. Reusing a key with a different payload returns `409`.

### Queue override

`POST /queue/promote` requires:

```json
{
  "request_id": 1,
  "priority": 10,
  "reason": "Operational reason",
  "expected_version": 1
}
```

Blank reason is invalid. Stale version returns `409 OCC_CONFLICT`.

### Notifications

- `GET /notifications?unread_only=true&limit=50&offset=0`.
- `POST /notifications/{id}/read` is idempotent.
- Cross-owner access returns 404.

### Pagination

Repair requests, material requests and queue support bounded `limit` and non-negative `offset`. Queue also supports `stage`. Responses remain arrays and do not currently include `total` or `has_more`.

### Errors

Business errors retain compatibility:

```json
{
  "detail": "Human-readable message",
  "error": {
    "code": "BUSINESS_ERROR",
    "message": "Human-readable message"
  }
}
```

Some errors have specialized codes such as `OCC_CONFLICT`. FastAPI validation errors (`422`) retain the standard FastAPI schema.

## Deprecated/compatibility endpoints

- `/repair-requests/{id}/accept`
- `/repair-requests/{id}/approve-inspection`
- `/repair-requests/{id}/reject-inspection`
- Generic `/repair-requests/{id}/status`
- Whole-request material receive/handover endpoints are compatibility paths; item-level endpoints are canonical.

Client code should be generated or checked against the live OpenAPI after the final backend migration release.

