# Backend Hardening Stage 1 Report

**Stage:** Configuration, runtime, secret and foundational CI

**Baseline backend commit:** `a1eee46`
**Status:** Backend implementation pushed; GitHub CI verification pending.

**Backend implementation commit:** `88c0307`

## Scope completed

- Added strict `APP_ENV`: `development`, `test`, `staging`, `production`.
- Added centralized validation for JWT secret, database URL, CORS list, token expiry, upload limit and worker count.
- Staging/production fail fast for weak/default JWT secret and debug mode.
- Added runtime settings for log level, trusted hosts, worker count, keep-alive and graceful shutdown timeout.
- Removed Uvicorn `--reload` from the container runtime.
- Kept migration and seed in startup temporarily; their separation belongs to Stage 2.
- Added foundational GitHub Actions jobs for compile, regression tests and Docker image build.
- Added dedicated settings hardening tests independent of developer `.env`.
- Split `pytest` and `httpx` into `requirements-dev.txt`; the production image installs runtime dependencies only.

## Files changed locally

Backend repository:

- `app/core/config.py`
- `Dockerfile`
- `docker-compose.yml`
- `.env.example`
- `tests/test_settings_hardening.py`
- `requirements.txt`
- `requirements-dev.txt`
- `.github/workflows/backend-ci.yml`

Documentation repository, local only:

- `BACKEND_HARDENING_STAGE_1_REPORT.md`

## Verification evidence

- Regression and Stage 1 tests after review expansion: `60 passed in 1.82s`.
- Python compile: passed.
- Production with default JWT secret: process exited with code `1` and a validation error.
- Docker Compose configuration validation: passed.
- Docker image `fixtrack-backend:stage1-local`: built successfully.
- Image runtime command inspection confirms no `--reload`.
- Runtime process inspection confirmed `--workers 2`, `--timeout-keep-alive 17` and `--timeout-graceful-shutdown 41` from environment overrides.
- Production image import inspection confirmed `pytest` and `httpx` are absent.
- `.env.example` contains development placeholders only; the JWT placeholder is explicitly rejected in staging/production.
- Docker server used for verification: `29.6.1`.

## Acceptance status

| Criterion | Result |
|---|---|
| Strict environment names | Pass |
| Production/staging reject placeholder secret | Pass |
| Staging rejects short secret | Pass |
| Production rejects debug mode | Pass |
| Development/test remain usable | Pass |
| Tests do not require developer `.env` | Pass |
| Production image has no reload | Pass |
| Missing APP_ENV defaults explicitly to development | Pass |
| Invalid APP_ENV/case/whitespace is rejected | Pass |
| Empty/whitespace/common placeholder secrets are rejected in secure environments | Pass |
| Staging and production reject debug mode | Pass |
| Secure environments require PostgreSQL URL | Pass |
| CORS origins are trimmed and validated | Pass |
| Token/upload/worker/keep-alive/graceful boundaries | Pass |
| `.env.example` contains no real secret | Pass |
| Production image excludes test dependencies | Pass |
| Runtime worker and timeout overrides are applied | Pass |
| Foundational CI workflow exists | Pass locally |
| GitHub CI run is green | Pending push |
| Application startup still runs migration/seed | Deferred to Stage 2 |

## Known limits and next stage

- Container startup still invokes Alembic and seed. This is intentionally deferred to Stage 2 to preserve stage boundaries.
- Trusted host middleware and HTTP enforcement belong to Stage 4; Stage 1 only validates/configures the setting.
- GitHub Actions was triggered by backend commit `88c0307`; final closure awaits the run result.
- Documentation commit is recorded after this report is pushed.

## Validation rules implemented

- Token expiry: 1–1440 minutes.
- Upload limit: 1–100 MB.
- Worker count: 1–32.
- Keep-alive timeout: 1–120 seconds.
- Graceful shutdown timeout: 1–300 seconds.
- CORS: comma-separated HTTP/HTTPS origins; whitespace is trimmed and path-bearing/invalid origins are rejected.
- Staging/production: PostgreSQL URL required, debug forbidden and JWT secret must be at least 32 characters and not a known placeholder.
- Missing `APP_ENV` intentionally defaults to `development`; invalid aliases, mixed case and surrounding whitespace are rejected.

Runtime configuration is hardened and the backend implementation has been pushed. Deployment lifecycle remains non-production-safe until Stage 2 removes automatic migration and seed from application startup. Formal Stage 1 closure requires a successful GitHub CI run and documentation revision evidence.
