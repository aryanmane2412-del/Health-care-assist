# Health Assist

*Your Health. Your Family. Your AI Companion.*

A healthcare platform for six roles: patient, doctor, laboratory, pharmacy, delivery partner and administrator.

- **Flutter** web app in `lib/`
- **Go** REST API in [`backend/`](backend/README.md)
- **PostgreSQL** database, migrated and seeded by the backend
- Optional services, all called **only by the API**:
  - AI Assistant (Google Gemini)
  - online payments (Razorpay, plus a clearly labelled development simulator)
  - email notifications (SMTP)

The Flutter app talks **only** to the Go API. It never connects to PostgreSQL and contains no database credentials or server-side keys. The server enforces every permission at three layers: route role rules, ownership checks and PostgreSQL row-level security.

**Status:** development build, verified by automated tests (see [PROJECT_STATUS.md](PROJECT_STATUS.md)). It is **not production-ready**; see "Outstanding issues" there.

| Document | What it covers |
|---|---|
| [PROJECT_STATUS.md](PROJECT_STATUS.md) | Features, test results, outstanding issues |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | Components, request flow, security layers, key flows |
| [docs/DATABASE_SETUP.md](docs/DATABASE_SETUP.md) | PostgreSQL on Windows, migrations, seed, DB tests |
| [docs/API_DOCUMENTATION.md](docs/API_DOCUMENTATION.md) | Every endpoint, errors, auth rules |
| [docs/API_KEYS_AND_SETUP.md](docs/API_KEYS_AND_SETUP.md) | Gemini, Razorpay, SMTP credentials and setup |
| [docs/FILE_STORAGE_AND_RETENTION.md](docs/FILE_STORAGE_AND_RETENTION.md) | Uploads, file access, retention, deletion |
| [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) | Publishing: Firebase Hosting for the frontend; what the API and database need |

---

## Run it locally on Windows (PowerShell)

> **Already set up on this computer?** Start everything with:
> ```powershell
> cd "<path to>\Health care assist"
> Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
> .\scripts\start-dev.ps1          # starts the Health Assist database (port 5433) and the API (port 8080)
> ```
> Then, in a second terminal: `flutter run -d chrome`. See [docs/DATABASE_SETUP.md](docs/DATABASE_SETUP.md#option-b-private-development-database-no-postgres-password-needed) for the private development database it uses.

These steps take about 20–30 minutes the first time. Run each block in **Windows PowerShell** or PowerShell 7. Replace `<…>` placeholders with your own values.

### 1. Install prerequisites

```powershell
winget install --id Git.Git -e
winget install --id GoLang.Go -e                 # Go 1.26 or newer (go.mod requires 1.26.5)
winget install --id PostgreSQL.PostgreSQL.18 -e  # remember the "postgres" superuser password you set
winget install --id Google.Chrome -e             # Flutter Web runs in Chrome
```

**Flutter** has no official winget package. Install the Flutter SDK (stable channel) by following <https://docs.flutter.dev/get-started/install/windows/web>:

1. Download the ZIP.
2. Extract it to a folder without spaces (for example `C:\src\flutter`).
3. Add `C:\src\flutter\bin` to your user `PATH`.

The app was tested with Flutter 3.44.7 (Dart SDK `^3.12.2` is required).

Open a **new** PowerShell window, so the updated `PATH` is loaded, and check:

```powershell
git --version
go version                        # go1.26.x or newer
flutter --version                 # Flutter 3.44+ (Dart 3.12+)
flutter doctor                    # "Chrome - develop for the web" must show a check mark
& "C:\Program Files\PostgreSQL\18\bin\psql.exe" --version
```

### 2. Create the local environment file

```powershell
cd "<path to>\Health care assist\backend"
Copy-Item .env.example .env

# Generate a JWT secret (48 random bytes, Base64) and copy the output:
$b = New-Object byte[] 48; [Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($b); [Convert]::ToBase64String($b)

notepad .env
```

In `.env`, set these values and leave the rest at their defaults:

```env
JWT_SECRET=<paste the generated secret>
DATABASE_PASSWORD=<choose a password of 12+ characters; you will type the same one in step 3>
SEED_USER_PASSWORD=<a development login password, 8+ characters with a letter and a digit>
```

Notes:

- The Flutter app needs **no** environment file. Its only setting is the API URL, passed with `--dart-define`.
- Optional services (AI, payments, email) stay **off** until you set them; see [docs/API_KEYS_AND_SETUP.md](docs/API_KEYS_AND_SETUP.md). To try online payments without a Razorpay account, add `PAYMENT_PROVIDER=simulator` (development only; no money moves).
- `.env` is ignored by Git. Never commit it.

### 3. Configure PostgreSQL

```powershell
cd "<path to>\Health care assist\backend"
Get-Service postgresql*                                     # should be Running
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass  # this window only
.\scripts\db\setup-local.ps1
```

The script asks you to **choose** the `health_assist_user` password. Use the same value as `DATABASE_PASSWORD`. Then `psql` asks for the **`postgres` superuser** password. The script creates the role and the `health_assist` and `health_assist_test` databases, and never drops anything. Details and options: [docs/DATABASE_SETUP.md](docs/DATABASE_SETUP.md).

### 4. Run migrations (and load demo data)

```powershell
cd "<path to>\Health care assist\backend"
go run ./cmd/migrate up
go run ./cmd/migrate status      # expect: applied: 13   latest: 13   pending: 0   dirty: false
go run ./cmd/seed                # optional: fictional demo accounts (uses SEED_USER_PASSWORD)
```

### 5. Start the Go backend

```powershell
cd "<path to>\Health care assist\backend"
go run ./cmd/api
```

Leave this window open. The log shows `server starting addr=:8080`, plus one line per optional service saying whether it is on or off. Stop the server with **Ctrl+C**.

### 6. Start Flutter Web (second PowerShell window)

```powershell
cd "<path to>\Health care assist"
flutter pub get
flutter run -d chrome --dart-define=API_BASE_URL=http://localhost:8080/api/v1
```

Chrome opens the app. `API_BASE_URL` defaults to `http://localhost:8080/api/v1`, so the `--dart-define` is needed only if you changed `PORT` or use a remote API. Press `q` in this window to stop.

### 7. Test the health endpoint (third window, or before step 6)

```powershell
Invoke-RestMethod http://localhost:8080/health
# success: True, data.status = "ok"

(Invoke-RestMethod http://localhost:8080/api/v1/ready).data
# status = "ready", checks.database.status = "ok"  → the API can reach PostgreSQL
```

`/health` works without a database. If `/ready` fails with a `503` error, the API cannot reach PostgreSQL; see Troubleshooting.

### 8. Sign in

Register a new patient, or use a seeded account with your `SEED_USER_PASSWORD`. Debug builds show a "Use seeded account" button that fills in the email.

| Role | Seeded email |
|---|---|
| Patient | `asha.verma@example.com`, `rohan.iyer@example.com` |
| Doctor (verified) | `dr.meera.kapoor@example.com` |
| Doctor (pending verification) | `dr.arjun.nair@example.com` |
| Laboratory | `carepath.lab@example.com` |
| Pharmacy | `greenleaf.pharmacy@example.com` |
| Delivery partner | `vikram.singh@example.com` |
| Administrator | `ops.admin@example.com` |

All seeded people and organisations are fictional. Tokens are kept in memory only, so reloading the page signs you out.

### Troubleshooting

| Symptom | Fix |
|---|---|
| `go`, `flutter` or `psql` "is not recognized" | Open a new PowerShell window after installing. For `psql` in the current window: `$env:Path += ";C:\Program Files\PostgreSQL\18\bin"`. |
| `health-assist-api: invalid configuration: JWT_SECRET is required …` | Run `go run ./cmd/api` from the `backend` folder (that is where `.env` is read), and set `JWT_SECRET` in `.env`. The message lists every problem found. |
| `JWT_SECRET must be at least 32 bytes` | Generate one with the command in step 2. |
| `listen tcp :8080: bind: Only one usage of each socket address …` | Port 8080 is taken. Find the process with `Get-NetTCPConnection -LocalPort 8080 \| Select-Object OwningProcess`, or set `PORT=8081` in `.env` and run Flutter with `--dart-define=API_BASE_URL=http://localhost:8081/api/v1`. |
| `/api/v1/ready` → 503, and the API log shows `readiness check failing check=database` (for example `password authentication failed` or `connection refused`) | PostgreSQL is not running (`Get-Service postgresql*`), or `DATABASE_*` in `.env` does not match what `setup-local.ps1` created. See [docs/DATABASE_SETUP.md](docs/DATABASE_SETUP.md#troubleshooting). |
| `setup-local.ps1 cannot be loaded because running scripts is disabled` | `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass` (current window only). |
| `migrate: Dirty database version N` | A migration failed halfway. Fix the reported error, then `go run ./cmd/migrate force -version <N-1> -yes` and `go run ./cmd/migrate up`. |
| `seed: SEED_USER_PASSWORD is required` | Set it in `backend\.env` or in the window: `$env:SEED_USER_PASSWORD = "<password>"`. |
| App shows "Cannot reach the Health Assist server…" | The API is not running, or `API_BASE_URL` points elsewhere. Check `Invoke-RestMethod http://localhost:8080/health`. |
| Browser console shows a CORS error | Outside production, `http://localhost:*` and `http://127.0.0.1:*` are allowed by default. If you open the app from another host, add it to `CORS_ALLOWED_ORIGINS` in `.env` and restart the API. |
| Login works, then you are signed out after a reload | Expected: tokens are kept in memory only, never in browser storage. |
| AI screens say "AI Assistant is not set up yet" | Expected without `AI_API_KEY`. See [docs/API_KEYS_AND_SETUP.md](docs/API_KEYS_AND_SETUP.md). Emergency guidance still works. |
| No "Pay online" button | Expected unless `PAYMENT_PROVIDER` is set. Items are paid offline. |
| `flutter doctor` shows no Chrome | Install Chrome, or set `CHROME_EXECUTABLE` to the path of a Chromium-based browser. |

---

## Try the full workflow

1. **Doctor:** Appointments → *My availability* → publish slots.
2. **Patient:** Find a doctor → pick a slot → request the appointment.
3. **Doctor:** accept it → *Prescribe* → mark complete.
4. **Patient:** Lab tests → book. Home collection needs a saved address; add one in the dialog or under Profile.
5. **Lab:** accept → mark sample collected → *Upload report* (PDF/JPEG/PNG).
6. **Patient:** Lab tests → My bookings → *Download report*. The report is also listed under Medical records, where it can be shared with a doctor.
7. **Patient:** Medicines → add items → attach a prescription for Rx items → place the order.
8. **Pharmacy:** Orders → accept → mark packed. An available delivery partner is assigned automatically.
9. **Delivery partner:** Assigned deliveries → confirm pickup → start delivery → mark delivered.
10. **Payments (optional):** with `PAYMENT_PROVIDER=simulator`, use *Pay … online* on a booking or order and choose the simulated outcome.
11. **Notifications:** every user's Notifications page lists the updates from the steps above.
12. **AI (needs `AI_API_KEY` on the server):** AI Assistant → chat, read a report or prescription, scan a meal, plan a diet.

## Tests

```powershell
# Flutter (from the repository root)
flutter pub get
flutter analyze
flutter test                                   # widget tests against a fake API client
flutter build web --release

# Go (from backend\)
gofmt -l .                                     # prints nothing when formatted
go vet ./...
go build ./...
go test ./...                                  # PostgreSQL tests are skipped without TEST_DATABASE_URL
$env:TEST_DATABASE_URL = "postgres://health_assist_user:<password>@localhost:5432/health_assist_test?sslmode=disable"
go test -p 1 ./...                             # includes the PostgreSQL integration tests (they ERASE the _test database)

# Live end-to-end: the real Flutter API client → running Go API → PostgreSQL (development data only)
$env:API_E2E_BASE_URL = "http://localhost:8080/api/v1"
$env:API_E2E_SEED_PASSWORD = "<SEED_USER_PASSWORD>"
flutter test test/integration/api_e2e_test.dart
```

## Project layout

```
lib/        Flutter app: core (config, router), models, services (ApiClient, HealthApi),
            providers (Riverpod), features (screens per role), widgets
test/       widget tests, fake API client, live end-to-end test
backend/    Go API: cmd (api, migrate, seed), internal packages, migrations, tests, docs
docs/       setup, API, database, architecture, file retention
```

More detail: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).
