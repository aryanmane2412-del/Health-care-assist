# Health Assist — Go Backend

REST API for the Health Assist Flutter app.

**Phase 2: API foundation**

- HTTP server with configuration, graceful shutdown, request logging and panic recovery
- Consistent JSON envelope and error handling
- Request validation
- JWT authentication with bcrypt password hashing
- Role-based authorization middleware
- Route groups for all six roles and all feature areas

**Phase 3: PostgreSQL database**
- 10 migrations (42 tables) with constraints, indexes, double-booking prevention and row-level security on health data
- pgx connection pool and a `/ready` check that verifies connectivity and the schema version
- `cmd/migrate` and `cmd/seed` (fictional development data)
- PostgreSQL user repository plus integration tests

**Phase 4: connected to PostgreSQL and the Flutter app**
- Accounts, profiles and addresses; the doctor directory and published availability; appointments; prescriptions; medical records with patient-controlled sharing; lab bookings and report upload; pharmacy inventory and medicine orders; delivery assignment and status tracking; admin overview and partner verification. All of these run against PostgreSQL.
- Every request runs inside a transaction with the caller set for row-level security. Role rules apply per route, ownership checks run in the services, and RLS is enforced in the database.
- Uploaded files are stored on local disk (`STORAGE_DIR`) and served through short-lived signed links, with each issued link audited.

**Phase 5: AI Assistant** (Google Gemini, optional)
- Health-information chat, medical report reader, prescription reader, food scanner, diet planner and saved history, all under `/api/v1/ai` (patients only).
- Provider-neutral `internal/ai` package with the Gemini client, safety prompts, emergency detection, personal-data redaction and per-patient rate limits.
- Without `AI_API_KEY` the server starts normally and the AI endpoints answer `AI_NOT_CONFIGURED`; emergency guidance still works. Setup: [../docs/API_KEYS_AND_SETUP.md](../docs/API_KEYS_AND_SETUP.md).

**Phase 7: optional external services**
- **Payments** (`internal/payments`): provider interface, Razorpay (Orders API, Checkout signature, webhook HMAC with secret rotation) and a development-only simulator. Amounts always come from the database. Idempotency keys, one paid payment per item, webhook deduplication, and a payment is marked paid only after server-side verification. Off unless `PAYMENT_PROVIDER` is set.
- **Notifications** (`internal/notify`): in-app notifications written in the same transaction as each event; optional email through an outbox and a background worker (SMTP over TLS, or a development log mailer), so email failures never affect bookings.
- **Files:** every download re-checks access for the user the link was issued to; patients can delete their own uploads. Retention: [../docs/FILE_STORAGE_AND_RETENTION.md](../docs/FILE_STORAGE_AND_RETENTION.md).
- Setup for every provider: [../docs/API_KEYS_AND_SETUP.md](../docs/API_KEYS_AND_SETUP.md). Without credentials, everything runs locally.

Still `501 NOT_IMPLEMENTED`: symptom check, settlements/earnings, refunds, nearby search/mapping, password reset/logout and some admin tools. See [../docs/API_DOCUMENTATION.md](../docs/API_DOCUMENTATION.md).

Docs:
- [../docs/API_DOCUMENTATION.md](../docs/API_DOCUMENTATION.md): endpoints
- [../docs/ARCHITECTURE.md](../docs/ARCHITECTURE.md): components and request flow
- [../docs/DATABASE_SETUP.md](../docs/DATABASE_SETUP.md): PostgreSQL setup on Windows
- [docs/DATABASE.md](docs/DATABASE.md): schema, ER diagram, security, production settings
- [docs/MIGRATIONS_WINDOWS.md](docs/MIGRATIONS_WINDOWS.md): PowerShell setup, migrations, backup and restore

## Requirements

- Go 1.26+ (developed with go1.26.5)
- PostgreSQL 13+ (verified on 18.4). PostGIS is optional.

## Setup

```bash
cd backend
cp .env.example .env
# Set JWT_SECRET in .env (at least 32 bytes):
openssl rand -base64 48
go mod download
```

The server reads real environment variables first. `.env` is optional and only fills in values that aren't set. **The server refuses to start without `JWT_SECRET`**; there is no built-in default.

PowerShell, without a `.env` file:

```powershell
$env:JWT_SECRET = "<paste output of: openssl rand -base64 48>"
go run ./cmd/api
```

## Database

Full guide: [docs/MIGRATIONS_WINDOWS.md](docs/MIGRATIONS_WINDOWS.md). Short version (PowerShell):

```powershell
.\scripts\db\setup-local.ps1          # creates role + databases (asks for passwords)
# set DATABASE_PASSWORD in .env
go run ./cmd/migrate up
$env:SEED_USER_PASSWORD = "<dev password>"; go run ./cmd/seed   # optional fictional data
```

If `DATABASE_*` is unset, the API still starts and `/ready` reports the database as `not_configured`. When it is set, the database is a required readiness check: `/ready` returns 503 if PostgreSQL is unreachable or the schema isn't fully migrated.

## Run

```bash
go run ./cmd/api
# → listening on :8080
curl http://localhost:8080/health
curl http://localhost:8080/api/v1/ready
```

Run the Flutter Web app against it (from the repository root):

```bash
flutter run -d chrome --dart-define=API_BASE_URL=http://localhost:8080/api/v1
```

Sign in with a seeded account (debug builds offer a "Use seeded account" button that fills in the email) and the `SEED_USER_PASSWORD` you chose.

CORS allows `http://localhost:*` and `http://127.0.0.1:*` by default outside production, because Flutter Web picks a random port. In production, set `CORS_ALLOWED_ORIGINS` to exact origins. Wildcards are rejected there.

## Test, format, build

```bash
gofmt -l .            # prints nothing when formatted
go vet ./...
go test ./...          # DB integration and end-to-end workflow tests skip unless TEST_DATABASE_URL is set
go build -o bin/health-assist-api ./cmd/api
# with a version stamp:
go build -ldflags "-X main.version=0.2.0" -o bin/health-assist-api ./cmd/api
```

## Configuration

| Variable | Default | Notes |
|---|---|---|
| `APP_ENV` | `development` | `development`, `test`, `production` |
| `PORT` | `8080` | |
| `LOG_LEVEL` | `info` | `debug` also prints Gin's route table |
| `JWT_SECRET` | — (**required**) | ≥ 32 bytes; never logged |
| `JWT_ISSUER` / `JWT_AUDIENCE` | `health-assist-api` / `health-assist-app` | Verified on every token |
| `JWT_ACCESS_TTL` / `JWT_REFRESH_TTL` | `15m` / `168h` | Refresh must be longer than access |
| `BCRYPT_COST` | `12` | 10–31 (4+ only when `APP_ENV=test`) |
| `CORS_ALLOWED_ORIGINS` | dev: `http://localhost:*,http://127.0.0.1:*`; prod: none | Comma-separated |
| `HTTP_READ_TIMEOUT` / `HTTP_WRITE_TIMEOUT` / `HTTP_IDLE_TIMEOUT` / `HTTP_SHUTDOWN_TIMEOUT` | `15s` / `30s` / `60s` / `15s` | |
| `HTTP_MAX_BODY_BYTES` | `1048576` | Larger bodies get `413` |
| `DATABASE_HOST` / `PORT` / `NAME` / `USER` / `PASSWORD` | — / `5432` / — / — / — | Optional in dev; required in production |
| `DATABASE_SSLMODE` | dev `disable`, prod `verify-full` | Production accepts only `verify-full` / `verify-ca` |
| `DATABASE_MAX_CONNS` / `MIN_CONNS` | `10` / `0` | Pool size |
| `DATABASE_CONNECT_TIMEOUT` / `STATEMENT_TIMEOUT` | `5s` / `30s` | |
| `STORAGE_DIR` | `storage` | Where uploaded records and lab reports are kept (created with mode 0700). Development / single server only; use encrypted object storage in production. Never commit it. |
| `STORAGE_MAX_UPLOAD_BYTES` | `10485760` | Upload limit (1 byte – 25 MiB) |
| `FILE_LINK_TTL` | `5m` | Lifetime of signed download links (10s–1h) |
| `AI_PROVIDER` / `AI_API_KEY` / `AI_MODEL` | — | Optional AI Assistant (Gemini). The key is a secret; never logged. See docs/API_KEYS_AND_SETUP.md |
| `AI_TIMEOUT` | `45s` | |
| `AI_RATE_LIMIT_PER_MINUTE` / `AI_RATE_LIMIT_PER_DAY` | `6` / `100` | Per patient, per instance |
| `AI_MAX_UPLOAD_BYTES` | `8388608` | Largest image/PDF for AI analysis |
| `SEED_USER_PASSWORD` | — | Only for `cmd/seed` |
| `TEST_DATABASE_URL` | — | Only for tests; the database name must end in `_test` |

If configuration is invalid, every problem is reported at once on startup, and secret values are never echoed.

## Layout

```
backend/
  cmd/api/main.go            wiring, server start, graceful shutdown
  cmd/migrate/               migration CLI (status/up/down/force, with guards)
  cmd/seed/                  fictional development data loader
  migrations/                embedded SQL migrations (golang-migrate format)
  scripts/db/                setup-local.ps1, create-local-db.sql, backup.ps1, enable-postgis.sql
  internal/database/         pgx pool, WithTx/WithActor (RLS context), health check, migrator
  internal/seed/             seed data (fictional)
  internal/config/           env loading + validation
  internal/routes/           router, /api/v1 prefix, implemented routes, planned-endpoint catalogue
  internal/handlers/         HTTP handlers (health, auth, platform features, planned → 501)
  internal/storage/          private local file store, content-type sniffing, signed (user-bound) download links
  internal/payments/         payment provider interface, Razorpay client, development simulator
  internal/notify/           in-app notifier, email outbox worker, SMTP and development log mailers
  internal/middleware/       request ID, logging, recovery, security headers, CORS, body limit, authn, RBAC
  internal/auth/             bcrypt hasher, JWT token manager, request principal
  internal/services/         business logic (auth, care, commerce, admin, AI assistant, payments, notifications)
  internal/ai/               AI provider interface, Gemini client, safety prompts, redaction, rate limiter
  internal/repositories/     persistence interfaces + UnconfiguredUserRepository (503 without a DB)
  internal/repositories/postgres/  PostgreSQL queries (users, profiles, directory, care, labs, orders, admin, payments, notifications)
  internal/models/           domain types and request/response DTOs
  internal/validators/       custom validation rules (role, phone, password) + field errors
  internal/apperrors/        typed errors → HTTP status/code
  internal/response/         JSON envelope writer
  internal/testutil/         TEST DOUBLES ONLY (in-memory fake repo for tests)
  tests/                     HTTP-level tests, PostgreSQL integration tests (db_*_test.go), end-to-end workflows (e2e_workflows_test.go)
  docs/                      API.md, DATABASE.md, MIGRATIONS_WINDOWS.md
```

### Request flow

`RequestID → Logger → Recovery → SecurityHeaders → CORS → BodyLimit → [Authenticate → RequireRoles] → handler → service (ownership checks) → transaction with RLS actor → repository`

## Security notes

- Passwords: bcrypt (cost 12 by default). The plain text is never stored or logged, and passwords over 72 bytes are rejected rather than silently truncated.
- JWT: HS256 only. Algorithm, issuer, audience, expiry and token type are all verified. Refresh tokens cannot be used as access tokens, and `alg=none` is rejected.
- Login does not reveal whether an email exists. Unknown emails still run a bcrypt comparison, and a role mismatch gets the same error as a wrong password.
- Logs record method, path (without the query string), status, duration and request ID. Headers and bodies are never logged.
- 5xx responses return a generic message. The underlying cause goes only to the server log.
- Unknown JSON fields are rejected, and request bodies are size-limited.
- `X-Forwarded-For` is not trusted until a known proxy is configured.

- Database: the app role is not a superuser; plain-text passwords are rejected by a `CHECK` constraint; health data is protected by forced row-level security; no card data is stored. See docs/DATABASE.md.

## Known limitations

- No token revocation or server-side logout yet; the Flutter app keeps tokens in memory only.
- No rate limiting on auth endpoints yet. Add it before production.
- Uploaded files are on local disk; production needs an encrypted volume (or private object storage, not implemented) and backups.
- Refunds are manual (Razorpay Dashboard); `payment.refund_required` audit entries flag cases that need one. Doctors, labs and pharmacies cannot yet see in the app whether an item was paid online.
- Simulated payments live in server memory: after a restart, start a new payment.
- Endpoints marked 🕓 in docs/API_DOCUMENTATION.md still return 501.
