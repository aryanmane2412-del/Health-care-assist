# Health Assist — Project Status

_Last updated: 2026-10-25 (Phase 8: final testing and handover)_

| Phase | Scope | Status |
|---|---|---|
| 1 | Flutter frontend (all six roles, demo data) | ✅ Complete |
| 2 | Go backend foundation | ✅ Complete |
| 3 | PostgreSQL database and migrations | ✅ Complete |
| 4 | Connect Flutter ↔ Go ↔ PostgreSQL | ✅ Complete (verified end to end on a throwaway local PostgreSQL 18.4 cluster, see below) |
| 5 | AI Assistant (Google Gemini) | ✅ Complete — tested with mocked AI responses; **no real Gemini call has been made** (no API key available) |
| 6 | Smart mapping | ⏳ Not started (skipped for now; Phase 7 does not depend on it) |
| 7 | Optional external services: payments, notifications, file storage | ✅ Complete. Tested with a local fake of the Razorpay API and the development simulator; **no real Razorpay call or real email has been made** (no credentials available) |
| 8 | Final testing and development handover | ✅ Complete. All automated suites pass; development build, **not production-ready** (see Phase 8) |

---

## Phase 8 — Final testing and development handover (complete)

**Verdict:** this is a working **development build**. All automated suites pass, and a full local run (PostgreSQL → Go API → Flutter API client) was carried out end to end. It is **not production-ready** (see "Outstanding issues"), and three external services were never exercised against the real provider because no credentials were available.

**Environment used:** Windows 11, Go 1.26.5, Flutter 3.44.7, PostgreSQL 18.4, Windows PowerShell 5.1. All database tests ran on a **throwaway PostgreSQL cluster** (`initdb` in a temporary folder, port 55433, non-superuser app role); your own PostgreSQL service was not touched. The API under test ran on ports 18080/18081.

### Test results (2026-10-25)

Legend: ✅ passed · 🔧 failed first, fixed, re-run and passed · ⚠️ passed with a caveat · ⛔ could not run

**Frontend**

| Check | Result | Evidence |
|---|---|---|
| Dependency resolution (`flutter pub get`) | ✅ | "Got dependencies!" (18 packages have newer major versions available; not upgraded) |
| Static analysis (`flutter analyze`) | ✅ | No issues |
| Widget and unit tests (`flutter test`) | ✅ | **43 passed**, 3 skipped (the live tests, run separately below) |
| Web release build (`flutter build web --release`) | ✅ | Built `build/web`, including the Razorpay JS interop; bundle contains no secrets (Phase 7 scan) |
| Navigation: every route for all six roles at 390 / 900 / 1440 px | ✅ | `routes_smoke_test.dart` |
| Role guard: signed-out users go to welcome; a patient cannot open the doctor, admin, lab, pharmacy or delivery workspaces | ✅ | New: `forms_and_states_test.dart` |
| Form validation: login (required fields, email format, server 401 message shown); registration (email, phone, password policy, matching passwords, terms) with no API call while invalid | ✅ | New: `forms_and_states_test.dart` |
| API states: loading, retryable error (500) → retry → data, 403 "no access", empty lists | ✅ | New: `forms_and_states_test.dart`; Phase 7 screens |
| Manual click-through in a real browser | ⛔ | Not done; widget tests render every screen, but not inside Chrome |

**Backend** (`gofmt -l .` → no output; `go vet ./...` → clean; `go build ./...` → ok; `go test -p 1 ./...` with `TEST_DATABASE_URL`: **107 passed, 0 failed, 0 skipped**, of which 62 are unit tests and 45 are HTTP/PostgreSQL integration tests)

| Area | Result | Tests |
|---|---|---|
| Authentication: hashing, JWT (expiry, wrong secret, `alg: none`, token type), refresh, login | ✅ | `internal/auth`, `TestAuthFlowWithTestRepository`, `TestAuthServiceWithPostgres` |
| Role-based authorization: every planned and implemented route, wrong roles → 403, unauthenticated → 401 | ✅ | `routes_test.go`, the e2e tests |
| Request validation and error responses: malformed JSON, unknown fields, oversized bodies, field errors, envelope with request id, 404/405, panics → JSON 500 without leaking details | ✅ | `TestMalformedBodies`, `TestRegisterValidation`, `foundation_test.go` |
| Appointment conflicts: same slot twice; **overlapping doctor availability; end before start; the same patient double-booked across two doctors**; consultation type not offered | ✅ | `TestAppointmentDoubleBookingRejected`; new `TestSchedulingConflicts` |
| Order and delivery transitions: **every invalid pharmacy step (pack before accept, accept twice, cancel after accept, reject after packing); every invalid delivery step (transit before pickup, delivered before transit, unknown status, failure without a reason, wrong role); a failed delivery returns the order to the pharmacy, which re-assigns it; delivered is final** | ✅ | `TestEndToEndWorkflows`; new `TestOrderAndDeliveryStateMachines` |
| Database connectivity: readiness `ok`/`503`, server starts without a DB and answers 503 | ✅ | `TestDatabaseHealthCheck`, `TestFeatureRoutesWithoutDatabase`, live `/ready` |
| Sensitive record access: row-level security, owner / shared doctor / lab / pharmacy, admin and delivery blocked, revoked links die | ✅ | `TestRowLevelSecurityOnHealthRecords`, `TestFileAuthorizationOnEveryRequest` |
| File handling: content sniffing, size limits, random keys, path traversal, deletion rules | ✅ | `internal/storage`, `TestFileAuthorizationOnEveryRequest` |
| File permissions on disk | ⚠️ | The storage folder is created with mode `0700`. **On Windows, Go mode bits do not set ACLs**: the folder inherits its parent's ACL (observed: SYSTEM, Administrators, the user). Files are never served statically. In production, place `STORAGE_DIR` on a volume that only the service account can read. |
| AI integration failures: missing key, 401/403/429/500, blocked, empty, malformed, timeout, the key never in errors, rate limits | ✅ | `internal/ai`, `TestAIWithoutConfiguration`, `TestAIAssistantWithMockedProvider`, `TestAIRateLimit` |
| Payment webhook security: forged/unsigned → 400 and not stored, raw-body HMAC, secret rotation, dedupe by event id, amount mismatch, no downgrade, card data never stored | ✅ | `internal/payments`, `TestRazorpayCheckoutVerificationAndWebhooks`, `TestFailedPaymentKeepsBookingAndLateCaptureIsFlagged` |
| Logs contain no secrets | ✅ | `TestLogsDoNotContainSecrets`; live log scan (0 tokens, passwords or full email addresses) |

**Database**

| Check | Result | Evidence |
|---|---|---|
| Migrations up on an empty database | ✅ | 13 applied, 43 tables |
| **Full rollback** (`down -steps 13`), then up again | ✅ | 43 → 0 → 43 tables, not dirty |
| Roll back and re-apply the latest migration on a **seeded** database | ✅ | 12 ↔ 13 |
| Destructive commands need `-yes` | ✅ | `down` without `-yes` refused |
| Constraints and relationships: exclusion constraints, FK restrict, `CHECK`s, plain-text password rejection, one paid payment per item, append-only webhooks | ✅ | `TestIntegrityConstraints`, `TestPostgresRejectsPlainTextPassword`, Phase 7 tests |
| Seed: loads, and a second run changes nothing | ✅ | CLI run twice; `TestSeedIsIdempotent` |
| `scripts/db/create-local-db.sql` (used by `setup-local.ps1`), run twice in PowerShell | ✅ | Non-superuser `health_assist_user` and the databases created; the second run is a no-op |
| `setup-local.ps1` interactive prompts | ⛔ | Not run: it asks for passwords with `Read-Host`, which this non-interactive session cannot answer. The SQL it runs was verified (row above). |
| Repository operations and transaction rollback | ✅ | `TestPostgresUserRepository*`, `TestWithTxRollsBackOnError`, `TestPostgresUserRepositoryCreateIsAtomic` |
| Record ownership restrictions (RLS, as each role) | ✅ | `TestRowLevelSecurityOnHealthRecords`, `TestNotificationsAreOwnOnly` |

**End-to-end workflows.** The live run used the real Go API on a freshly migrated and seeded database, driven by the Flutter `ApiClient`/`HealthApi` (`test/integration/api_e2e_test.dart`: **3 of 3 passed**), plus the Go e2e tests.

| # | Workflow | Result |
|---|---|---|
| 1 | Patient registration and login | ✅ |
| 2 | Appointment booking and doctor acceptance (plus prescription and completion) | ✅ |
| 3 | Lab booking and report publication (and patient download) | ✅ |
| 4 | Medicine ordering and pharmacy processing (Rx attachment, stock) | ✅ |
| 5 | Delivery assignment and completion (and failed delivery → re-assign) | ✅ |
| 6 | Access to authorised medical records (sharing, revocation, deletion) | ✅ |
| 7 | AI with configured credentials | ⛔ **Could not run: no Gemini API key.** The "not configured" path and emergency guidance were verified live, and the full AI flow with a mocked provider. |
| 8 | Nearby-provider search | ⛔ **Not implemented** (Phase 6 was never built). `GET /nearby/providers` returns `501 NOT_IMPLEMENTED` (verified live), and the app shows no fake nearby results. |
| 9 | Payment workflow | ⚠️ Verified with the **development simulator** (live) and a **local fake of the Razorpay API** (tests). **No real Razorpay payment**: no keys. |

**Documentation runbook.** The README's PowerShell steps 2, 4, 5 and 7 and the API-docs login snippet were executed in Windows PowerShell 5.1 (in a scratch copy of `backend/`, against the throwaway cluster). The steps were: `.env` from the example and a generated JWT secret, `migrate up`/`status`, `seed`, `go run ./cmd/api`, `/health` → ok, `/ready` → database ok, login → `/auth/me`, and the 403 error envelope. The troubleshooting messages for "port in use" and "missing JWT_SECRET" were reproduced word for word. Not executed by me: the `winget` installs (package ids were confirmed with `winget search`) and `flutter run -d chrome` (interactive). The web build itself passed.

### Bugs found in Phase 8

No application bugs were found in this phase. Two new backend tests and one new Flutter test failed on their first run. In each case the **test expectation** was wrong and the application behaved correctly:

- The seeded second doctor offers no video consults, so the server correctly answered 422 before the overlap check.
- `PUT /delivery/availability` returns 200, as documented.
- Registration requires the terms checkbox.

A Riverpod-internal API used in a new test was replaced to keep `flutter analyze` clean. (Phase 7's final verification had already found and fixed 8 bugs; see below.)

### Documentation updated

`README.md` (complete Windows PowerShell runbook and troubleshooting), `PROJECT_STATUS.md`, `docs/API_KEYS_AND_SETUP.md` (credential status, applying keys on Windows), and these new files:

- `docs/DATABASE_SETUP.md`
- `docs/API_DOCUMENTATION.md`: the API reference moved here from `backend/docs/API.md`, which now points to it.
- `docs/ARCHITECTURE.md`

### Handover checklist

**Completed features**

- [x] Six role portals (patient, doctor, lab, pharmacy, delivery, admin), responsive from phone to desktop
- [x] Registration, login, token refresh, role-based routing and server-side RBAC
- [x] Doctor directory, availability publishing, appointment booking / accept / reject / complete, prescriptions
- [x] Medical records: upload, patient-controlled sharing, re-authorised downloads, deletion of own uploads
- [x] Lab bookings, sample collection, report upload and replacement, patient report access
- [x] Pharmacy inventory, server-priced medicine orders, Rx verification, packing
- [x] Delivery auto-assignment, pickup → transit → delivered, failure → re-assignment, status history
- [x] Admin overview (counts only) and partner verification
- [x] In-app notifications; optional email through an outbox worker
- [x] Optional online payments (Razorpay with server verification and webhooks; development simulator)
- [x] AI Assistant (Gemini) with safety rules, emergency guidance, redaction and rate limits (requires a key)
- [x] PostgreSQL schema with 13 migrations, row-level security, seed data and integration tests

**Outstanding issues (not production-ready)**

- [ ] **Nearby-provider search and mapping (Phase 6) not built**: `/nearby/*` returns 501.
- [ ] No real Gemini, Razorpay or SMTP call has been made (no credentials).
- [ ] No manual browser click-through; no load or penetration testing; no accessibility audit.
- [ ] Not deployed: HTTPS/reverse proxy, managed PostgreSQL with `verify-full`, secret manager, backups and monitoring still to set up.
- [ ] No rate limiting on authentication endpoints; no token revocation, logout, password reset or change (those endpoints return 501).
- [ ] Account deletion and data export not implemented (501); use an admin procedure.
- [ ] Refunds are manual; settlements and earnings, admin user management, complaints and audit-log viewer still show **demo data** (labelled DEMO).
- [ ] The SOS "alert my contacts" action is simulated (labelled as such).
- [ ] Symptom checker endpoint `/ai/symptom-check` returns 501 (the app uses fixed rules).
- [ ] Doctors, labs and pharmacies cannot see whether an item was paid online.
- [ ] Local-disk file storage; object storage not implemented. On Windows, folder ACLs must be set by the operator.
- [ ] The AI rate limiter and simulated payment orders are in memory (per instance, reset on restart).
- [ ] Your Git repository root is your user folder (`C:\Users\DELL`) with no commits. Create a repository in the project folder before committing.
- [ ] 18 Flutter packages have newer major versions (not upgraded, to avoid unverified changes).

**External credentials still needed**

| Credential | For | Where |
|---|---|---|
| `AI_API_KEY` (Google AI Studio) | AI Assistant | `backend\.env` |
| `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, `RAZORPAY_WEBHOOK_SECRET` (test keys first) | Online payments | `backend\.env` |
| `SMTP_HOST`, `SMTP_USERNAME`, `SMTP_PASSWORD`, `EMAIL_FROM` | Email notifications | `backend\.env` |
| (you create) `JWT_SECRET`, `DATABASE_PASSWORD`, `SEED_USER_PASSWORD` | Required locally | `backend\.env` |

Setup steps: [docs/API_KEYS_AND_SETUP.md](docs/API_KEYS_AND_SETUP.md).

**Exact commands to run the app** (after the one-time setup in README steps 1–4)

```powershell
# Window 1: API
cd "<path to>\Health care assist\backend"
go run ./cmd/api

# Window 2: Flutter Web
cd "<path to>\Health care assist"
flutter run -d chrome --dart-define=API_BASE_URL=http://localhost:8080/api/v1

# Window 3: health check
Invoke-RestMethod http://localhost:8080/health
(Invoke-RestMethod http://localhost:8080/api/v1/ready).data
```

---

## Phase 7 — Optional external services (complete)

Provider setup: [docs/API_KEYS_AND_SETUP.md](docs/API_KEYS_AND_SETUP.md). File storage and retention: [docs/FILE_STORAGE_AND_RETENTION.md](docs/FILE_STORAGE_AND_RETENTION.md).

When I started, part of this phase's backend already existed from an earlier, interrupted session: the payment provider interface, the Razorpay client and simulator, the notification outbox and worker, migration `000013`, and the repositories. None of it was reachable: there were no routes for it and nothing wired it into `main.go`. I reviewed that code, fixed the bugs listed below and finished the phase.

### What runs, by configuration

| Setting | Empty (default) | Development value | Real value |
|---|---|---|---|
| `PAYMENT_PROVIDER` | Offline payment only; the app says online payment is not enabled | `simulator`: no money moves, every simulated payment is labelled SIMULATOR or [SIMULATED], and the server refuses this value in production | `razorpay` with `RAZORPAY_MODE=test` or `live`. Live mode needs `APP_ENV=production`, and the key prefix must match the mode. |
| `EMAIL_PROVIDER` | In-app notifications only | `log`: logs "email NOT sent" with a masked address; refused in production | `smtp` (TLS required) |
| File storage | Private local disk (`STORAGE_DIR`) | Same | Same, on an encrypted volume. Object storage is not implemented because no provider is configured. |

Local development needs no credentials.

### A. Payments

| Area | What it does |
|---|---|
| Abstraction | `internal/payments.Provider`, implemented by `Razorpay` and `Simulator` |
| Orders | `POST /payments/intents` takes the amount from the database (appointment fee, lab price or order total), never from the app. The server creates the Razorpay order. |
| Verification | Checkout callback: the server checks the HMAC signature over the order id it stored plus the payment id. It then fetches the payment from Razorpay, which must be captured, for the same order and for the exact amount. Webhooks: the server checks the HMAC of the raw body before parsing or storing anything. The previous secret is still accepted while a secret is being rotated. |
| Idempotency | An idempotency key from the client (unique per payer), a per-item advisory lock and reuse of an open payment. A unique index allows only one paid payment per item. Webhook events are deduplicated by `x-razorpay-event-id`, and a paid payment is never downgraded. |
| Stored in PostgreSQL | `payment_orders`: status, mode (`test`, `live` or `simulated`), provider order id, failure reason and `paid_at`. `payment_transactions`: provider payment id and method category. `payment_webhook_events`: sanitised and append-only. Row-level security lets payers see only their own payments, and status changes run only as the system after verification. |
| Never stored | Card number, CVV, expiry, card holder name, UPI ID, or the payer's email or phone from webhooks. Card details are entered in Razorpay's own frame. |
| Failures | A failed payment is recorded with its reason and the patient is notified. The booking or order stays unchanged and can be paid again. A provider outage returns `503` and leaves the booking intact. A payment captured after the item was cancelled, or a duplicate capture, is kept and flagged as `payment.refund_required` in `audit_logs`. |
| Flutter | A *Pay ₹… online* button on patient appointments, lab bookings and medicine orders, shown only when online payment is enabled and the item can be paid. **TEST MODE** and **SIMULATOR** tags. A simulator dialog that says no money moves. Razorpay Checkout via `checkout.js` (JS interop, loaded on demand). A **Paid online** chip. Booking screens explain how the patient can pay. |

### B. Notifications

- **In-app:** each notification is written in the same transaction as its event: bookings, status changes, prescriptions, record sharing, verification and payments. New endpoints: `GET /users/me/notifications`, `POST …/:id/read` and `POST …/read-all`. Row-level security limits each user to their own notifications.
- **Email (optional):** each email is an `email_outbox` row written in the same transaction, and a background worker in the API sends it.
  - Failed sends are retried with backoff, and the worker gives up after `EMAIL_MAX_ATTEMPTS`.
  - Running several API instances is safe (`SKIP LOCKED` plus a lease).
  - Emails contain only the title and a link to the app, with no health details.
  - The worker stops cleanly on shutdown.
- **Flutter:** the Notifications screen now shows real data; before this phase it showed demo data. The unread badge refreshes every 60 seconds while signed in. `lib/data/mock/mock_activity.dart` was removed.

### C. File storage

Phase 4 already provided random 128-bit storage names, file types detected from content and size limits. New in this phase:

- **Every download is re-authorised.** Signed links now name the user and the resource, and `/files/download` repeats the row-level security check as that user. Un-sharing a record, replacing a lab report or deleting a record invalidates links that were already issued.
- **Deletion:** `DELETE /patient/records/:id` deletes one of the patient's own uploads together with its shares and its file, and writes an audit entry. Lab reports and records attached to medicine orders are kept (`409`). In the Flutter app, the record sheet has *Delete record* with a confirmation step.
- Retention and deletion behaviour is documented in `docs/FILE_STORAGE_AND_RETENTION.md`.

### Bugs found and fixed

| Bug | Effect | Fix |
|---|---|---|
| The `notifications` insert used `RETURNING id` under row-level security. | Once notifications were turned on, **every booking or order that notified another user failed with 500**. `RETURNING` needs the SELECT policy, and a patient cannot read a lab's notifications. | The UUID is now generated in Go, and the insert no longer uses `RETURNING`. |
| Checkout verification turned an amount or order mismatch into `500`. | A mismatched payment looked like a server crash. | It now returns `409` with an explanation. |
| The Checkout description contained the lab test name or the doctor's name. | Health-related details were sent to Razorpay. | Descriptions are now generic. |
| Two concurrent payment requests could create two provider orders. | Possible double charge. | A per-item advisory lock is taken before the idempotency lookup. |
| Duplicate or late captures were only logged. | Refunds could be missed. | They now create a `payment.refund_required` audit entry, and the patient is notified. |
| Record deletion used `SELECT … FOR UPDATE`. | Row-level security hides rows that fail the UPDATE policy, so lab-filed records returned `404` instead of `409`. | The lock was removed, and the delete is now conditional. |
| The code expected SQLSTATE 23503 when deleting a record attached to an order. | `ON DELETE RESTRICT` raises **23001**, so the request returned `500`. | Both codes are handled. |
| `TestWithTxRollsBackOnError` counted notifications without setting an actor. | Under the new row-level security, the count would always be zero, so the test would pass even if the rollback were broken. | The test now inserts and counts as the system actor. |

Also in this phase: I removed the unused `services.PaymentGateway` stub, marked the seeded fictional payments explicitly as `simulated`, and extended `tests/routes_test.go` to cover the payment routes.

### Verification (2026-10-25)

**No Razorpay keys or SMTP credentials were available.** I tested Razorpay against a local fake of its REST API, using Razorpay's documented signature formulas. For the live runs, payments used the development simulator and email used the development log mailer. **No real payment, Razorpay API call or email has been made.**

As in earlier phases, the database tests ran on a **throwaway PostgreSQL 18.4 cluster**, created with `initdb` in a temporary folder and run on port 55433 with a non-superuser app role. Your own PostgreSQL service was not used.

| Check | Result |
|---|---|
| **Unit tests.** Razorpay signatures: valid, tampered, other order, other secret, rotated webhook secret and modified body. Card and contact data stripped from webhook payloads. The Orders and Payments APIs against a fake server, and the simulator. Config: test and live keys kept separate, simulators refused in production, and `.env.example` loads. SMTP header injection and retry backoff. User and resource bound into signed links. | Pass |
| **`TestRazorpayCheckoutVerificationAndWebhooks`.** Server-side pricing, the idempotency key and reuse of an open order. Reusing a key for another item returns 409, another patient gets 404 and a doctor gets 403. Forged and replayed signatures return 400 and are audited, an amount mismatch returns 409, and a valid verification marks the payment paid. A forged or unsigned webhook returns 400 and is not stored. The same event delivered three times is processed once, with one notification. A late failure does not downgrade a paid payment, an unknown order is acknowledged, and no card or contact data is stored. | Pass |
| **`TestFailedPaymentKeepsBookingAndLateCaptureIsFlagged`.** The failure is recorded with its reason, the booking stays pending and the patient gets a "Payment failed" notification. A retry reuses the order. A capture after cancellation is marked paid and flagged `refund_required`. | Pass |
| `TestSimulatorIsLabelledAndUsesRealVerification` and `TestPaymentsDisabledByDefault` | Pass |
| **`TestNotificationsAreOwnOnly`.** Users see only their own notifications, another user's id returns 404, and mark-read and read-all work. | Pass |
| **`TestEmailFailuresNeverLoseTheBooking`.** After an SMTP failure, the booking and the in-app notification are kept and a retry is scheduled with backoff. The email is marked failed after the maximum attempts. With a working mailer it is sent, and the email does not contain the test name. | Pass |
| **`TestFileAuthorizationOnEveryRequest`.** A wrong file type returns 422 and an oversized file 413, and storage keys are random. The owner can download; another patient or an unshared doctor gets 404, and bad tokens get 403. **A doctor's link stops working after the record is un-shared.** Only the owner can delete, which removes the row, shares and file and writes an audit entry. The link to a replaced lab report stops working. Deleting a lab report returns 409, and so does deleting an order's prescription. | Pass |
| `go test ./...` with `TEST_DATABASE_URL`: **43 tests in `tests/`, 0 skipped**; `go vet` and `gofmt` | Pass / clean |
| `flutter analyze`, and `flutter test` with **37 tests**. The 7 new tests cover live notifications, no Pay button when payments are off, the simulator dialog and labels, **a Checkout success that is not trusted without server verification**, a verified payment showing as paid, a dismissed Checkout not calling verify, and the record delete rules. | Clean / pass |
| `flutter build web --release`, which compiles the Razorpay JS interop. The bundle was scanned for keys and secrets. | Builds; no secrets found |
| **Live run.** The real Go API (`PAYMENT_PROVIDER=simulator`, `EMAIL_PROVIDER=log`) and PostgreSQL, driven by the Flutter `ApiClient`. All 3 tests in `test/integration/api_e2e_test.dart` pass, including the new Phase 7 test: notifications, a simulated failure then success, a forged signature returning 400, a link returning 403 after the record is shared and then revoked, and a link returning 403 after deletion. | Pass |
| **Live HTTP.** A forged webhook returns 400, the wrong provider 404 and an unauthenticated payment request 401. | Pass |
| **API log after the live run.** 36 "email NOT sent" lines, all with masked addresses. **No tokens, passwords, signatures or full email addresses.** | Pass |

**Not verified:**

- A real Razorpay test-mode payment in a browser. This needs your test keys, and webhooks also need a public HTTPS URL.
- Real SMTP delivery.
- A manual click-through in Chrome.

### To use real services

1. **Razorpay test mode:** generate test keys, turn on auto-capture, and add a webhook with a secret (see the setup guide). Then set `PAYMENT_PROVIDER=razorpay`, `RAZORPAY_MODE=test`, `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET` and `RAZORPAY_WEBHOOK_SECRET`. Run `flutter run -d chrome` and pay with a Razorpay test card.
2. **Email:** set `EMAIL_PROVIDER=smtp` and the `SMTP_*` values from your provider.

### Known limitations

- Refunds are made manually in the Razorpay Dashboard. Settlements and earnings still show demo data.
- Doctors, labs and pharmacies cannot see in the app whether an item was paid online, and delivery partners cannot see whether an order is prepaid or cash on delivery.
- Simulated orders are kept in server memory, so after a restart the patient has to start a new payment.
- In a short window, two requests with different keys for the same item can each create a Razorpay order. Only one can be marked paid, and any extra capture is flagged for refund.
- No push notifications and no per-user email opt-out yet.
- Account deletion and data export still return 501. Handle such requests with an admin procedure (see the retention doc).

---

## Phase 5 — AI Assistant (complete)

Setup and key handling: [docs/API_KEYS_AND_SETUP.md](docs/API_KEYS_AND_SETUP.md).

### Implemented

| Feature | Flutter | API | Stored |
|---|---|---|---|
| Health-information chat (multi-turn, last 20 turns as context) | AI Health Assistant | `POST /ai/chat` | `ai_conversations`, `ai_messages` |
| Medical Report Reader (upload, or pick one of my records) | Medical Report Reader | `POST /ai/report-reader` | `ai_analysis_jobs` (result only) |
| Prescription Reader (exact transcription) | Prescription Reader | `POST /ai/prescription-reader` | ″ |
| Food Scanner (calorie range, macros, confidence, assumptions) | Food Scanner | `POST /ai/food-scanner` | ″ |
| Diet Planner (7-day plan) | Diet Planner | `POST /ai/diet-plan` | ″ |
| Chat history (reopen, view, delete) | Chat history, AI hub "recent" | `/ai/conversations*`, `/ai/analyses*` | — |
| Status / configuration message | Every AI screen | `GET /ai/status` | — |

**Backend**
- `internal/ai`: a provider interface and the Gemini client (REST `generateContent`, key in the `x-goog-api-key` header, timeout, error mapping, nothing logged). It also holds the safety prompts and JSON schemas, emergency detection, redaction, and an in-memory per-patient rate limiter.
- **Provider choice:** Google Gemini. The model ID comes from `AI_MODEL`; Google's models page currently recommends `gemini-3.5-flash-lite` or `gemini-3.8-flash` for new projects. I could not confirm on Google's pages that these models accept image/PDF input before a documentation-fetch rate limit stopped me, so the setup guide asks you to check that.
- Config: `AI_PROVIDER`, `AI_API_KEY`, `AI_MODEL`, `AI_TIMEOUT`, `AI_RATE_LIMIT_PER_MINUTE`/`_PER_DAY`, `AI_MAX_UPLOAD_BYTES`. All are optional; the server starts without them.
- Migration `000012`: `ai_messages.emergency`; `ai_analysis_jobs.model` and `input_label`.
- Uploads are validated by sniffing their content (`storage.ReadValidated`), analysed in memory, and **not stored**.

**Safety** (server-side, not just UI)
- Prompts forbid diagnoses, inventing or changing doses, and invented values. They require stated uncertainty and referral to professionals.
- Red-flag messages (chest pain, breathing difficulty, stroke signs, seizure, severe bleeding, poisoning, self-harm…) get fixed emergency guidance (112 / 108 / Tele-MANAS 14416) without calling the AI. This works without a key and is never rate limited. If the model flags an emergency itself (`EMERGENCY:`), the app shows the emergency banner too.
- **Prescription reader:** illegible fields are left blank and listed, and the server always appends "confirm with your pharmacist or doctor".
- **Food scanner:** calories are a range, with a confidence level and the assumptions made.
- **Diet planner:** professional review is flagged for conditions or extreme targets.
- Every result carries a fixed server-written disclaimer.
- **Data minimisation:** emails, phone numbers and ID numbers are redacted before anything is sent to Google; names, account ids and other records are never sent; titles are redacted too.
- When AI is unavailable the app says so. All mock AI output (the Phase 1 `MockAiService` and fake chat history) has been **removed**.

**Other changes**
- The doctor "Clinical AI Assistant" menu item was removed: it was a canned chatbot, and the AI here is a patient-information tool. Doctor-facing AI would need its own clinical safety design.
- AI requests can take up to 45 s, so the Flutter client's receive timeout was raised from 30 s to 60 s.
- Bug found by the tests and fixed: a question and its reply saved in one transaction got identical timestamps and could display in the wrong order. They now use `clock_timestamp()`.

### Verification (2026-10-24)

**No Gemini API key was available, so no real AI call was made.** All AI behaviour was tested with mocked responses:

| Check | Result |
|---|---|
| `internal/ai` unit tests: Gemini client against a local fake HTTP server (key in header and not in URL, request shape, inline data base64, JSON config, 429/403/500/blocked/empty/malformed mapping, timeout, errors never contain the key); emergency detection; redaction; rate limiter | Pass |
| `internal/config` AI tests (optional, validation, key never echoed) | Pass |
| PostgreSQL integration tests with a scripted fake provider: `TestAIWithoutConfiguration`, `TestAIAssistantWithMockedProvider`, `TestAIRateLimit`. They cover persistence, ownership (another patient → 404, doctor → 403), redaction of what is sent, history context, both emergency paths, report/prescription/food/diet plus "not a report", bad JSON, wrong file types, record-by-id, deletion, files not stored and rate limits | Pass |
| Full `go test ./...` with `TEST_DATABASE_URL`: 36 tests in `tests/`, 0 skipped; `go vet`, `gofmt` | Pass / clean |
| `flutter analyze`; `flutter test`: 30 tests, including 5 new AI screen tests and every route at three widths | Clean / pass |
| Live run: Flutter `ApiClient` → Go API (no AI key) → PostgreSQL. Status is reported as not configured with a helpful message; emergency guidance is returned and saved; ordinary chat and analysis return 503; doctor → 403; delete → 404 | Pass |
| API log after the live run | No chat content, passwords or tokens |

**Separate real-integration result:** not run (no credentials). To run it, add a key as described in the setup guide, restart the API and run `flutter test test/integration/api_e2e_test.dart`. With a key, the AI test makes one real chat call and prints the reply.

### Known limitations

- The rate limiter is in memory (per API instance, resets on restart).
- The symptom checker (`/ai/symptom-check`) is still planned (501). The app's symptom form uses fixed rules to suggest a specialty.
- Free-tier Gemini usage may be used by Google to improve its products. Review Google's terms and use a paid tier before handling real patient data.

---

## Phase 4 — Integration (complete)

### What is genuinely connected (Flutter → Go API → PostgreSQL)

| Area | Flutter screens | API |
|---|---|---|
| Registration, login, session refresh, sign-out on expiry | Login, Register | `/auth/*` |
| Profile (name, phone, city, organisation) and saved addresses | Profile; address picker in lab booking and cart | `/users/me`, `/users/me/addresses` |
| Role dashboards | All six dashboards, built from the role's live lists (admin: live counts) | role list endpoints, `/admin/overview` |
| Doctor directory, specialties, doctor detail and real bookable slots | Find a doctor, Doctor detail, Search, Symptom check suggestions | `/doctors*`, `/specialties` |
| Doctor availability publishing / removal | Doctor → Appointments → *My availability* (new) | `/doctor/availability` |
| Appointments: book, cancel, accept, decline, complete | Patient and doctor appointment screens | `/patient/appointments*`, `/doctor/appointments*` |
| Prescriptions: write (tied to an appointment), view | Doctor prescriptions; patient Medical records → Prescriptions (new) | `/doctor/prescriptions`, `/patient/prescriptions` |
| Patient summary for the treating doctor | Doctor → Patients → detail sheet | `/doctor/patients/:id` |
| Medical records: upload, download, share/unshare with doctors | Patient Medical records; doctor Shared reports | `/patient/records*`, `/doctor/shared-records`, `/records/:id/download-link` |
| Lab tests and bookings: book, cancel, accept/reject, sample collected, report upload/replace, report download | Patient Lab tests; lab Dashboard, Bookings, Reports | `/lab-tests`, `/patient/lab-bookings*`, `/lab/bookings*` |
| Pharmacies, live inventory, cart and orders (server-priced, stock reserved), Rx attachment | Patient Medicines, Search | `/pharmacies*`, `/patient/medicine-orders*` |
| Pharmacy processing: accept (approves Rx), reject with reason, pack, assign delivery, stock updates, Rx viewing | Pharmacy Dashboard, Orders, Prescriptions, Inventory | `/pharmacy/*` |
| Delivery: availability toggle, assignments, pickup → transit → delivered, failure with reason | Delivery Dashboard, Assigned, History | `/delivery/*` |
| Partner verification: approve, reject with reason | Admin Dashboard, Verification | `/admin/verifications*` |
| Platform activity (counts only) | Admin Platform activity (rebuilt) | `/admin/overview` |

Every screen above has loading, error (with retry) and empty states. Actions show the server's error message, buttons show progress while a request runs, and lists update from the record the server returned.

### What still uses demo data (labelled DEMO in the app)

The backend endpoints for these still return `501`:

- ~~AI Assistant~~: connected in Phase 5.
- **Earnings and settlements** charts and screens; admin payouts.
- ~~Notifications list~~: connected in Phase 7.
- **Admin:** user management, complaints, audit-log viewer. The server does write real audit events; there is just no read API yet.
- **Password reset / change, logout revocation, data export:** the app says these are unavailable instead of pretending.
- ~~Online payments~~: added in Phase 7 (optional; offline payment remains the default).

Removed: the Phase 1 mock personas and in-memory catalogues/activity, the "any password works" login, and fake uploads and downloads.

### Security

- Flutter holds only the API base URL. The release web bundle was scanned and contains no database password, JWT secret, seed password or connection string.
- Tokens are kept in memory only (not in browser storage). Expired access tokens are refreshed once; if the refresh fails, the user is returned to sign-in.
- Authorization is enforced by the server for every protected operation: route role rules, service ownership checks, and PostgreSQL row-level security on health data. Cross-user and cross-role attempts were tested and get `403`/`404` (see below).
- Admins see counts and verification data only, never clinical content. The admin "Orders & appointments" screen that listed patients' transactions was replaced by aggregate counts.
- Patients can share records only with doctors they have booked; the server also requires the doctor to be verified.
- Files are served through 5-minute signed links issued after an access check; each issued link is written to `audit_logs`.
- Client-side validation mirrors the server (e.g. password policy), but the server is authoritative.

### Bugs and gaps fixed while integrating

- The Flutter project did not compile (22 analyzer errors left by a partial model refactor). It is now clean.
- `file_picker` 13 API migration (`FilePicker.pickFile`, size check before reading bytes).
- Doctors had no way to publish availability, so patients could not have booked anything. Added *My availability*.
- Patients had nowhere to see prescriptions their doctor wrote. Added a Prescriptions tab under Medical records.
- Riverpod 3 retried every failed provider; retries are now limited to network failures.
- Phone-width layout overflows (dropdowns, button rows, card headers), found by the widget tests and fixed.

### Verification (2026-10-24)

**Your PostgreSQL server was not used.** As in Phase 3, all database testing used a throwaway PostgreSQL 18.4 cluster created with `initdb` in a temporary folder, on port 55433 with a non-superuser app role. Your existing PostgreSQL 18 service and its data were not touched.

| Check | Result |
|---|---|
| `flutter analyze` | No issues |
| `flutter test` (25 widget tests: every route for every role at 390 / 900 / 1440 px, fed API-shaped JSON; login, logout, session expiry) | All pass |
| **`flutter test test/integration/api_e2e_test.dart`**: the app's real `ApiClient`/`HealthApi` over HTTP → Go API (port 18080) → PostgreSQL | **Pass** |
| `flutter build web --release` | Builds; bundle contains no secrets |
| `gofmt -l .`, `go vet ./...` | Clean |
| `go test ./...` with `TEST_DATABASE_URL` (33 tests in `tests/`, 0 skipped, including `TestEndToEndWorkflows` and `TestPartnerVerificationAndOrderCancellation`) | All pass |
| Direct SQL check after the Flutter E2E run | Rows present for the new user, appointment (`upcoming`), prescription, lab booking (`report_ready`), 2 medical records, order (`delivered`), 4 delivery history rows; files on disk; download audit events |
| API log check | No passwords, bearer tokens or signed-link tokens logged |

The Flutter E2E run covered all nine required workflows, and confirmed these denials from the app client:

- a patient calling a doctor/lab/pharmacy action → `403`
- another patient downloading the report → `404`
- the doctor downloading it before it was shared → `404`, and after sharing → `200`
- a pharmacy opening an unrelated record → `404`
- an Rx order without a prescription → `422`

**Not verified:** no manual click-through in a real browser was done (the widget tests render every screen, but not inside Chrome). The signed-link download opens a new browser tab; its HTTP behaviour is tested, but not the tab itself.

### Your next steps to run it locally

1. `backend\scripts\db\setup-local.ps1` (asks for your `postgres` password), then set `JWT_SECRET` and `DATABASE_PASSWORD` in `backend\.env`.
2. `go run ./cmd/migrate up`, then `go run ./cmd/seed` with `SEED_USER_PASSWORD` set.
3. `go run ./cmd/api`, then `flutter run -d chrome`.

### Known limitations

- Uploaded files are on local disk (`STORAGE_DIR`); production needs encrypted object storage.
- No rate limiting, token revocation or password reset yet.
- PostGIS is not installed here; nearby search isn't built yet anyway.

---

## Phase 3 — PostgreSQL database (complete)

11 migrations (10 from Phase 3; `000011` adds file names, free-text durations and the pharmacy prescription-read policy for Phase 4), 42 tables, forced row-level security on health data, `cmd/migrate`, `cmd/seed`, backup scripts. Details: [backend/docs/DATABASE.md](backend/docs/DATABASE.md), [backend/docs/MIGRATIONS_WINDOWS.md](backend/docs/MIGRATIONS_WINDOWS.md).

## Phase 2 — Go backend foundation (complete)

Gin server, env config, JWT + bcrypt auth, RBAC middleware, envelope/validation/errors, CORS, security headers, request logging without bodies or headers. Details: [backend/README.md](backend/README.md), [docs/API_DOCUMENTATION.md](docs/API_DOCUMENTATION.md).

## Phase 1 — Flutter frontend (complete)

Six role portals with Riverpod and GoRouter, responsive from phone to desktop.
