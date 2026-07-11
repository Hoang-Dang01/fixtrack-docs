# FixTrack — Technical Baseline

**Status:** Approved for P0 implementation  
**Version:** 1.0  
**Effective date:** 2026-07-11
**Supersedes:** Conflicting legacy SSS rules for the covered scope  
**Implementation baseline:** Current FastAPI backend

## Current stack

- Backend: FastAPI 0.115.6.
- ORM: SQLAlchemy 2.0.36.
- Migration: Alembic 1.14.0.
- Database: PostgreSQL 15.
- Validation: Pydantic 2.
- Authentication: JWT access token with Passlib/Bcrypt.
- Frontend: Next.js 14.2.15, React 18 and TypeScript.
- Backend testing: Pytest and HTTPX.

The current FastAPI implementation is the technical baseline. It must not be migrated to another framework as part of P0.

## Non-baseline technology

NestJS, Prisma, TypeORM, BullMQ, Redis workers, S3/R2, FCM, ClamAV and React Native are Planned, Future Architecture or Reference Architecture. Their presence in the legacy SSS does not authorize adding them to the MVP.

## Implementation rules

- Do not rename persisted roles or statuses without an explicit data migration.
- Keep business state, OCC version, audit/status logs and inventory ledger changes in the same database transaction when they belong to one operation.
- Do not perform external network notifications inside a database transaction.
- OpenAPI is canonical for HTTP endpoints and schemas only. Authorization, business rules, transactions and idempotency remain manually documented.
- Audit SHA-256 hash chaining already exists and must be preserved, not rebuilt.
- An outbox/worker is not required for P0 because it is not part of the current runtime.
- Backend API/OpenAPI is the contract for this remediation stream. Frontend implementation is owned by a separate developer and is explicitly out of scope; the frontend maps to the finalized backend contract.
