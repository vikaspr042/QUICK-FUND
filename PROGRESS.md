# PROGRESS

> Read this file, `BUILD_STATE.json` and `TODO.md`, in that order, at the
> start of every session — before writing or changing any code. This is the
> resume system required because this project is built incrementally across
> many Claude Free chat turns.

## Current status: PHASE 27 COMPLETE

> Note: this heading previously read "PHASE 19 COMPLETE" - stale since
> Phase 19 and never updated through Phases 20-26, even though the actual
> history through Phase 26 ("all 27 planned phases complete") was always
> correctly recorded further down in this file and in the Session log.
> Corrected here rather than left misleading. See the new "Phase 27"
> section at the end of this file for the most recent work (a GNN bugfix
> plus a UI/feature addition beyond the original 27-phase plan).

### What exists right now (cumulative)

- Phase 0/1 deliverables (architecture, full skeleton, docker-compose,
  root docs) — unchanged, see prior session log entries below.
- **loan-platform-a (QuickFund) backend is real**, not a skeleton:
  - `src/models/{User,Application,Repayment}.js` — Mongoose schemas.
  - `src/repositories/{user,application,repayment}Repository.js` — each
    exports a Mongo-backed implementation (production) and an in-memory
    implementation (tests), behind the same interface.
  - `src/services/{auth,application,repayment,admin}Service.js` — real
    business logic: registration/login validation, bcrypt hashing, JWT
    issuance, application validation + document hashing (never stores raw
    document content), status-transition rules
    (`SUBMITTED → UNDER_REVIEW → APPROVED|REJECTED`, invalid transitions
    rejected with 409), automatic repayment-schedule generation on
    approval, repayment ownership checks.
  - `src/middleware/auth.js` — JWT verification + role-based authorization.
  - `src/routes/{auth,applications,repayments,admin}.routes.js` — wired to
    the services above.
  - `src/app.js` — builds the Express app from injected repositories
    (production wiring in `src/server.js`, test wiring in
    `tests/helpers/buildTestApp.js`).
  - `src/server.js` — now starts the HTTP server immediately and connects
    to MongoDB in the background with retry/backoff, so `/health` responds
    even while MongoDB is still coming up (fixed after discovering the
    original await-then-listen order made health checks block for up to
    MongoDB's connection timeout).
  - `scripts/createAdmin.js` — CLI to provision an admin account (no public
    self-registration-as-admin endpoint, by design).
- **loan-platform-a frontend is real**, not a skeleton: `AuthContext.jsx`
  (JWT + user persisted to localStorage), `api.js` (typed fetch wrapper for
  every endpoint), and pages `Login`, `Register`, `ApplyLoan`,
  `ApplicationStatus`, `LoanHistory`, `Repayments`, `AdminLogin`,
  `AdminDashboard` — every button calls a real backend endpoint (register,
  login, logout, submit application, check status, view history, pay an
  installment, admin approve/reject/request-review, admin view repayments).
- `API.md` updated: loan-platform-a's endpoints moved from "planned" to
  "live", with a short description of each.

### What was actually tested (real commands, real output)

- `loan-platform-a/backend`: `npm test` → **34/34 tests passing** across 6
  suites (`auth`, `applications`, `admin`, `repayments`, `authz` edge cases,
  and one full `e2e_flow` covering register → two applications submitted →
  admin approves one/rejects the other → user sees both updated statuses →
  repayment schedule generated on approval → user pays an installment).
  Tests run against the in-memory repository implementations (see
  `TESTING.md` for exactly why, given MongoDB is unreachable in this build
  sandbox) but exercise the real app/services/routes/middleware over real
  HTTP via supertest.
- Verified the fixed `server.js` boot behavior directly: with an
  intentionally unreachable `MONGO_URI`, `/health` and `/api/v1/health`
  respond immediately (not blocked on the DB connection), while a DB-backed
  endpoint (`register`) correctly fails after MongoDB's own timeout instead
  of hanging the process forever.
- `loan-platform-a/frontend`: `npm run build` succeeds (`vite build`,
  no errors); built output copied into `backend/public/` and served
  correctly by the running backend (confirmed `<title>QuickFund</title>`
  present in the served HTML).
- All test/build artifacts (`node_modules`, `dist`, `backend/public`)
  deleted after verification; not committed (see `.gitignore`).

### Phase 3 additions

- **loan-platform-b (TrustLend) backend and frontend are real**, built from
  loan-platform-a's architecture (same models/repositories/services/
  routes/app.js/server.js pattern, same dual Mongo+in-memory repository
  approach) rather than a byte-for-byte copy: the `loan` object on an
  application now requires `institutionName` and `courseName`, enforced at
  both the Mongoose schema level and in `applicationService.submit()`
  (400 if missing) — a real, education-loan-specific schema difference from
  QuickFund's generic personal-loan application, not just a branding swap.
  All platform identifiers (`loan-platform-b`, `TrustLend`, `mongo-loan-b`,
  applicationId prefix `B`, brand color `#059669`) were substituted
  throughout.
- Frontend: `ApplyLoan` now collects institution/course; `ApplicationStatus`
  and `AdminDashboard` display them.
- `loan-platform-b/backend`: `npm test` → **35/35 tests passing** (the same
  6 suites as loan-platform-a, plus one new test asserting the
  institutionName/courseName validation).
- `loan-platform-b/frontend`: `vite build` succeeds; built output verified
  served correctly by the backend (`<title>TrustLend</title>` confirmed in
  the response).
- Updated `API.md` (loan-platform-b moved to "live" with its schema
  difference noted), `TESTING.md`, `loan-platform-b/README.md`.
- All test/build artifacts deleted after verification; not committed.

### Phase 4 additions

- **loan-platform-c (SecureCredit) backend and frontend are real**, built
  from loan-platform-a's architecture (same models/repositories/services/
  routes/app.js/server.js pattern, same dual Mongo+in-memory repository
  approach): `loan.amount` is capped at 50000 (INR) and `loan.tenureMonths`
  at 12, enforced in `applicationService.submit()` (400 naming the ceiling
  exceeded) and mirrored as `max` validators on the Mongoose schema as
  defense in depth — a real, micro-loan-specific validation difference from
  QuickFund's uncapped personal loans, not just a branding swap. All
  platform identifiers (`loan-platform-c`, `SecureCredit`, `mongo-loan-c`,
  applicationId prefix `C`, brand color `#7c3aed`) were substituted
  throughout.
- Frontend: `ApplyLoan` now shows the micro-loan ceilings inline and caps
  the amount/tenure inputs' `max` attributes accordingly; page title reads
  "Apply for a micro-loan".
- **Two bugs found and fixed while finishing this phase** (both via direct
  inspection, not user-reported):
  1. `frontend/src/brand.js` was missing entirely from the copy of
     loan-platform-a used to derive loan-platform-c (`App.jsx` and
     `components/ui.jsx` both import it, so the frontend build would have
     failed the first time anyone actually ran it) — recreated with
     SecureCredit's branding.
  2. The copy+sed process that builds each platform from loan-platform-a's
     footer text turned "QuickFund-X fraud intelligence project" into
     "<PlatformName>-X fraud intelligence project" instead of leaving the
     actual project name (`QuickFund-X`) alone — this had already slipped
     into **loan-platform-b** in Phase 3 undetected ("TrustLend-X") in
     addition to loan-platform-c ("SecureCredit-X"). Fixed in both.
- `loan-platform-c/backend`: `npm test` → **37/37 tests passing** (the same
  6 suites as loan-platform-a, plus three new tests: amount over the
  ceiling rejected, tenure over the ceiling rejected, exactly-at-ceiling
  accepted).
- `loan-platform-b/backend`: re-ran `npm test` after the footer-text fix to
  confirm no regression → still **35/35 tests passing**.
- `loan-platform-c/frontend`: `vite build` succeeds (no errors).
- Updated `API.md` (loan-platform-c moved to "live" with its ceiling rule
  noted; loan-platform-d split out separately), `TESTING.md`,
  `loan-platform-c/README.md`.
- All test/build artifacts (`node_modules`, `dist`) deleted after
  verification; not committed.

### Phase 5 additions

- **loan-platform-d (FinBridge) backend and frontend are real**, built from
  loan-platform-a's architecture (same models/repositories/services/
  routes/app.js/server.js pattern, same dual Mongo+in-memory repository
  approach): the `loan` object now requires `merchantName` and
  `productCategory` (`productCategory` restricted to a fixed list —
  electronics/appliances/furniture/fashion/education-tech/other), enforced
  both at the Mongoose schema level (`enum`) and in
  `applicationService.submit()` (400 if missing or not in the list) — a
  real, point-of-sale-specific schema difference from the other three
  platforms (an enum/categorical check, distinct in kind from B's
  required-field check and C's numeric-ceiling check), not just a branding
  swap. All platform identifiers (`loan-platform-d`, `FinBridge`,
  `mongo-loan-d`, applicationId prefix `D`, brand color `#ea580c`) were
  substituted throughout. With this phase, all four loan platforms have
  real, independently-differentiated, fully tested functionality.
- Frontend: `ApplyLoan` now collects merchant name (text) and product
  category (select, restricted to the same allowed list as the backend);
  `ApplicationStatus`, `AdminDashboard` and `LoanHistory` display them.
- **Found and fixed two of the same class of copy+sed bugs already known
  from Phase 4** (checked for proactively this time, since Phase 4 proved
  the copy+sed process reliably introduces them): the footer's project-name
  substitution again over-matched ("FinBridge-X" instead of "QuickFund-X"),
  and three comments referencing `mongo-loan-a`/`loan_platform_a`/
  `localhost:27018` (a bare `-a` sed didn't touch, same root cause as
  Phase 4's `ids.js` bug) were left pointing at platform A. All fixed
  before running any tests.
- `loan-platform-d/backend`: `npm test` → **36/36 tests passing** (the same
  6 suites as loan-platform-a, plus two new tests: missing merchantName/
  productCategory rejected, productCategory outside the allowed list
  rejected).
- `loan-platform-d/frontend`: `vite build` succeeds; served
  `<title>FinBridge</title>` confirmed in `index.html`/`dist/index.html`.
- Updated `API.md` (loan-platform-d moved to "live" with its schema
  difference noted), `TESTING.md`, `loan-platform-d/README.md`.
- All test/build artifacts (`node_modules`, `dist`) deleted after
  verification; not committed.

### Phase 6 additions

- **Every loan platform now has a real seed-data generator**,
  `<platform>/backend/scripts/seed.js`, following the same
  production/test split as `server.js` and `createAdmin.js`: its core
  `seed({ userRepository, applicationRepository, repaymentRepository })`
  function is exported separately from its `main()` CLI entrypoint (which
  is the only place that connects to a real MongoDB). Deliberately built to
  exercise the real `authService`/`applicationService`/`adminService` -
  the same business logic routes and tests use - rather than writing
  Mongo documents directly, so a seeded application/repayment schedule is
  indistinguishable from one a real user and a real admin produced, and an
  invalid seed payload would fail the same way a real bad request would.
- Each generator creates: 1 admin account, 5 demo users (realistic Indian
  names/addresses), and 8 loan applications spread across all four
  statuses (2 SUBMITTED, 2 UNDER_REVIEW, 2 APPROVED with a real generated
  repayment schedule, 2 REJECTED) - using platform-appropriate loan data
  (A: personal-loan amounts/purposes; B: real institution/course pairs;
  C: amounts/tenure kept within the 50000/12 micro-loan ceilings; D: real
  merchant names and allowed product categories). All demo accounts share
  the password `DemoPass123!` (seed/demo data only).
- Seed data stays **independent per platform** on purpose - no shared
  identifiers between platforms yet. Cross-platform overlap for fraud
  scenarios is explicitly Phase 12's job (`seed-data/scenarios/`), not
  Phase 6's.
- **Verified for real**: each platform gained `tests/seed.test.js`, which
  calls the exported `seed()` function directly against that platform's
  in-memory repositories (the same MongoDB-unreachable workaround every
  other test uses) and asserts both what `seed()` reports back and what
  the repositories actually now contain (admin role, every status
  represented, non-empty PENDING repayment schedules for approved
  applications). This is real, passing execution of the seed logic, not a
  syntax check - only the Mongo driver path itself (`main()`) is
  unexercised until Docker runs.
- **Found and fixed a third instance of the same copy+sed-derived bug
  class** while writing the seed scripts: `loan-platform-b` and
  `loan-platform-c`'s `scripts/createAdmin.js` usage-comment docstrings
  still referenced platform A's Mongo port/database name
  (`localhost:27018/loan_platform_a`) and the `admin@quickfund.example`
  example email, left over from Phase 1's skeleton generation and never
  corrected in Phases 3-4 because they're comment text, not executed code.
  Fixed all three platforms' docstrings (including D's example email,
  which had the same issue) to reference their own port/database/admin
  email.
- Test counts after adding `seed.test.js` to each platform: loan-platform-a
  **35/35**, loan-platform-b **36/36**, loan-platform-c **38/38**,
  loan-platform-d **37/37** (142 core tests unchanged + 4 new seed tests =
  146 total).
- Rewrote `seed-data/seed-platform-{a,b,c,d}/README.md` (previously "not
  yet generated" placeholders) to document where each generator actually
  lives, how to run it, and exactly what it creates - and each platform's
  own `README.md` gained a "Seeding demo data" section.
- Updated `TESTING.md` with the new counts and a "Phase 6: seed data is
  verified the same way" explanation section.
- All test/build artifacts (`node_modules`) deleted after verification;
  not committed.

### Phase 7 additions

- **Every loan platform now has a real secure event/outbox mechanism**,
  built first on loan-platform-a as the template and then applied to B/C/D:
  - `src/models/OutboxEvent.js` — Mongoose schema for `application_outbox`:
    `eventId` (unique), `eventType` (`APPLICATION_SUBMITTED` |
    `APPLICATION_STATUS_CHANGED`), `applicationId`, `payload` (Mixed),
    `status` (`PENDING`/`SENT`/`FAILED`, default `PENDING`), `attempts`,
    `lastError`, `nextAttemptAt`, `sentAt`.
  - `src/repositories/outboxRepository.js` — same dual Mongo-backed/
    in-memory pattern as every other repository in this project: `create`,
    `listDeliverable(now)`, `markSent`, `recordFailedAttempt` (capped
    exponential backoff: `min(backoffCapMs, backoffBaseMs * 2^(attempts-1))`,
    moves to `FAILED` once `attempts >= maxAttempts`), `listAll`.
  - `src/outbox/deliveryWorker.js` — polls `listDeliverable()`, POSTs each
    event (with `X-Idempotency-Key: eventId` and `X-API-Key`) to
    `{CENTRAL_FRAUD_API_URL}/api/v1/applications/process`, marks it `SENT`
    on a 2xx response or records a failed attempt otherwise; `start()`/
    `stop()` wrap it in a `setInterval` (unref'd, so it never blocks process
    exit) for production use.
  - `applicationService.submit()` and every `adminService` status
    transition (approve/reject/request-review) now write one outbox event
    each, with `outboxRepository` as an optional dependency (a platform
    still works standalone, with no outbox at all, if it's omitted —
    verified by a dedicated test).
  - `server.js` now also builds the Mongo-backed outbox repository, wires
    it into `createApp()`, and starts the delivery worker (5s poll
    interval) after the HTTP server comes up.
- **Verified for real, not mocked**: each platform's new
  `tests/outbox.test.js` spins up an actual `http.createServer()` on an
  ephemeral local port (not a mocked `fetch`) and exercises the delivery
  worker's real HTTP behavior end to end — a 200 response marks the event
  `SENT` and a repeat poll does not re-deliver it (idempotency); repeated
  500s are retried with backoff and the event is marked `FAILED` only after
  `maxAttempts` real attempts, each with its error recorded; `start()`/
  `stop()` actually run `deliverPending()` on an interval and leave no
  dangling timer. Event-creation tests confirm submitting an application or
  an admin approving one produces exactly the right outbox row (event type,
  applicationId, payload contents), read back from the real (in-memory)
  `outboxRepository` after a real HTTP call through the app.
- **The one thing genuinely NOT verified this phase**: delivery against the
  real central application, because its ingestion endpoint doesn't exist
  until Phase 8 — every real delivery attempt today genuinely fails and
  retries, which is documented as the correct, honest, currently-true
  behavior in `DATA_FLOW.md` and `API.md`, not hidden or worked around.
- **Found and fixed a fourth instance of the recurring copy+sed
  identifier-leak bug class**: `loan-platform-b/backend/src/server.js` and
  `loan-platform-c/backend/src/server.js` both still contained the comment
  `"...this platform's own MongoDB (mongo-loan-a - never any other
  platform's..."` — present since Phase 2, never caught before because it's
  prose inside a comment, not executed code, and the bare `-a` suffix in
  `mongo-loan-a` doesn't match a `loan-platform-a` → `loan-platform-b` sed
  substitution. Found via direct diff/grep before copying A's new
  `server.js` into B/C for this phase, fixed in both.
- Test counts after adding `outbox.test.js` to each platform:
  loan-platform-a **41/41**, loan-platform-b **42/42**, loan-platform-c
  **44/44**, loan-platform-d **43/43** — **170 tests total**.
- Updated `API.md` (new "Outbox mechanism" section; the planned
  `POST /api/v1/applications/process` row annotated as already targeted by
  every platform's delivery worker), `DATA_FLOW.md` (status upgraded from
  "stub" to "partial"; new "Steps 1-2: real, as of Phase 7" section),
  `TESTING.md` (new counts + a "Phase 7: verified against a real local
  server, not a mock" explanation section), `SECURITY.md` (new section on
  the `X-API-Key`/`X-Idempotency-Key` mechanism, explicitly scoped as
  interim — payload tokenization is still Phase 9's job), and all four
  platforms' `README.md` (status line, test counts, a new "Outbox / event
  delivery" section, updated testing-approach paragraph).
- All test/build artifacts (`node_modules`) deleted after verification;
  not committed.

### Phase 8 additions

- **`central-fraud-intelligence` now has a real application-event ingestion
  API**, the first endpoints that actually exist beyond `/health`:
  - `POST /api/v1/applications/process` — the endpoint every loan
    platform's outbox delivery worker has targeted since Phase 7. Real
    steps performed: authenticate the caller via `X-API-Key`
    (`src/security/auth.py`'s `resolve_platform_from_api_key`, constant-time
    comparison against `CENTRAL_ACCEPTS_API_KEY_{A,B,C,D}`), reject a key
    that authenticates a different platform than the event's `platformId`
    claims (403), validate the event's routing/dedup fields via a Pydantic
    model with `extra="allow"` (`src/ingestion/models.py`), and persist it
    idempotently to a real Postgres table (`src/ingestion/repository.py`,
    `INSERT ... ON CONFLICT (event_id) DO NOTHING RETURNING *` — a real
    atomic idempotency check, not an app-level race).
  - `GET /api/v1/applications/:applicationId` — every event received so
    far for one application; 404 if none.
  - Explicitly, honestly out of scope this phase (stated in the response
    body, the README, and DATA_FLOW.md, not silently skipped): payload
    decryption/tokenization (Phase 9), graph update, GNN inference, fraud
    scoring, incident creation, notifications (Phases 10-18). An accepted
    event is persisted with `status: "RECEIVED"` and stops there.
- **A real local PostgreSQL 16 instance turned out to be available in this
  build sandbox** (unlike MongoDB, which is fully blocked — see
  TROUBLESHOOTING.md's new Phase 8 entry). `postgresql-16` was already
  installed as a normal Ubuntu package; `service postgresql start` brought
  up a real cluster on port 5432 with zero network dependency. Created a
  `quickfundx` role (matching `docker-compose.yml`'s `CENTRAL_DB_USER`
  default) and two databases: `central_fraud_intelligence` (dev) and
  `central_fraud_intelligence_test` (tests, truncated between runs).
- **Proactively avoided the Phase-2 boot-order bug class this time**,
  rather than finding it after the fact: `PostgresEventRepository`'s
  constructor never opens a connection — it only stores the DSN — so
  `/health` and `/api/v1/health` (which never touch the repository)
  respond immediately even when Postgres is unreachable; a DB-dependent
  endpoint returns a real 503 instead. Verified directly (not just by
  design): pointed `app.state.event_repository` at an intentionally
  unreachable host and confirmed `/health` still returned 200 while
  `POST /api/v1/applications/process` correctly returned 503.
- **Verified for real, against a real Postgres database, not a mock or an
  in-memory double**: `central-fraud-intelligence/tests/test_ingestion.py`
  (10 tests) runs the real FastAPI app end-to-end via `TestClient` against
  the dedicated `central_fraud_intelligence_test` database — the actual
  `psycopg` driver code path is exercised by every test, a stronger
  guarantee than any loan platform's MongoDB-avoidance workaround gives.
  Covered: health endpoints unaffected; missing/invalid API key (401);
  platform/key mismatch (403); idempotency-key/eventId mismatch (400);
  malformed payload (422); valid event accepted + retrievable; a second
  event for the same application is a second row; the same event
  redelivered twice is persisted exactly once (`duplicate: true` on
  replay, verified via the real Postgres unique constraint); unknown
  applicationId returns 404.
- **A true end-to-end smoke test — the first real cross-service delivery in
  this project**: started a real `uvicorn` instance of
  `central-fraud-intelligence` locally, then used loan-platform-a's real
  `src/outbox/deliveryWorker.js` (with a real in-memory outbox event) to
  actually POST to it. The worker correctly marked the event `SENT`, and a
  follow-up `GET /api/v1/applications/A9999` on the central app confirmed
  the event was genuinely received and persisted with the right payload.
  This was a one-off manual verification (not part of either service's
  automated test suite, since it spans two separate processes/services),
  recorded here rather than in `TESTING.md`'s per-suite counts. Test
  artifacts (the smoke-test script, `loan-platform-a`'s temporarily
  installed `node_modules`, the inserted Postgres row) were all cleaned up
  afterward; nothing from this manual step was committed.
- Test counts: `central-fraud-intelligence` **10/10** (new); the four loan
  platforms unchanged at 170 total (41+42+44+43) since this phase touched
  no loan-platform code. **180 tests total across the project.**
- Updated `API.md` (both endpoints moved from "Planned" to "Live"),
  `DATA_FLOW.md` (new "Step 3: real, as of Phase 8" section; pipeline
  diagram split into real vs. still-design halves), `TESTING.md` (new
  counts; new "Phase 8: the central app's own tests, against a real local
  Postgres, not a double" section), `SECURITY.md` (new "Phase 8 addition:
  the central app now actually checks that key" section),
  `TROUBLESHOOTING.md` (new entry recording that Postgres, unlike MongoDB,
  is genuinely reachable in this sandbox — useful for every phase from
  here on that touches the central app's database), and
  `central-fraud-intelligence/README.md` (status, scope boundaries, local
  dev database setup, testing approach).
- All test/build artifacts (`__pycache__`, `.pytest_cache`, the temporary
  smoke-test script and `loan-platform-a/backend/node_modules`) deleted or
  left properly gitignored after verification; not committed.

### Phase 9 additions

- **Every loan platform now tokenizes and encrypts applicant fields before
  they ever leave the platform**, built first on loan-platform-a as the
  template and then applied unchanged (only built-in Node `crypto`, no new
  dependency) to B/C/D:
  - `src/tokenization/tokenize.js` — `deriveSubKey(masterKey, purpose)` via
    `crypto.hkdfSync('sha256', ikm, emptySalt, 'quickfund-x-token-v1:'+purpose, 32)`,
    `normalize()` (phone strips non-digits, email lowercases, address/name
    lowercase+collapse-whitespace), `tokenize()` (HMAC-SHA256 hex, or
    `null` for a missing/empty value), `tokenizeApplicant()` (returns
    `{nameToken, emailToken, phoneToken, addressToken}`).
  - `src/tokenization/encrypt.js` — AES-256-GCM `encryptField`/
    `decryptField`, random 12-byte IV per call, hex-encoded
    `{alg, iv, ciphertext, authTag}`.
  - `src/config.js` — added `CENTRAL_HMAC_MASTER_KEY`/`CENTRAL_AES_KEY`,
    same default value on all four platforms (deliberately **not**
    per-platform-suffixed — this is the exact key-distribution design
    `SECURITY.md` had left as "finalized in Phase 9": the same real-world
    value must tokenize identically no matter which platform computes it,
    or cross-platform entity resolution, Phase 10+, silently breaks).
  - `applicationService.js`'s outbox-event construction now calls
    `tokenizeApplicant()` and `encryptField()` *before* building the
    payload — the raw `applicant` object is never placed on the outbox
    event at all (not encrypted-then-included; simply absent).
- **Verified for real, not just "PII looks absent"**: each platform's new
  `tests/tokenization.test.js` (13 tests) covers determinism, normalization
  equivalence, different-inputs-differ, non-correlation across the four
  token purposes, different-master-key-differs, missing/empty→null,
  `tokenizeApplicant()`'s shape, phone normalization, AES round-trip,
  different-IV-per-call, wrong-key-fails, tampered-ciphertext-fails. Each
  platform's `tests/outbox.test.js` was extended to assert, against a real
  submitted application's real outbox row: no raw PII string appears
  anywhere in `JSON.stringify(payload)`, `payload.applicant` is
  `undefined`, and the token values present equal what calling
  `tokenize()` directly on the same raw input produces (so a wrong-but-
  still-PII-free token would also be caught).
- **`central-fraud-intelligence` gained a Python mirror of the same
  primitives** (`src/tokenization/tokenize.py`/`encrypt.py`, using the
  `cryptography` library's HKDF/AESGCM) plus **real cross-language parity
  tests** (`tests/test_tokenization.py::TestCrossLanguageParity`, 4 tests)
  that `subprocess.run(["node", ...])` the actual
  `loan-platform-a/backend/src/tokenization/*.js` files and assert
  byte-for-byte agreement with the Python output — a genuine behavioral
  check, not "both compile", specifically chosen because silent divergence
  here would invisibly break cross-platform fraud detection. All 16
  Python-side tokenization tests passed, including all 4 parity tests.
- **`central-fraud-intelligence` also gained its own admin JWT auth**,
  closing the gap Phase 8 explicitly left open:
  - `src/security/jwt_auth.py` — hand-rolled HS256 JWT using only stdlib
    (`hmac`/`hashlib`/`base64`/`json`/`time`) — deliberately no
    `python-jose`/`PyJWT` dependency, same "avoid a heavy dependency for
    something this small" reasoning planned for the hand-rolled GraphSAGE
    GNN later. `encode()`/`decode()`, `InvalidTokenError` on
    malformed/wrong-signature/expired tokens, `hmac.compare_digest` for
    the signature check.
  - `src/security/admin_repository.py` — `PostgresAdminRepository`,
    lazy-connecting like `PostgresEventRepository` (constructor only
    stores the DSN — proactively avoided the Phase-2/8 boot-order bug
    class again rather than rediscovering it).
  - `src/api/admin.py` — `POST /api/v1/admin/login` (bcrypt-checks a real
    `admin_users` row, issues a 24h token; identical 401 for unknown-email
    and wrong-password, anti-enumeration) and the `require_admin`
    dependency, now applied to `GET /api/v1/applications/:id`.
  - `scripts/create_admin.py` — CLI to provision an admin account (no
    public register endpoint, mirrors every loan platform's own
    `createAdmin.js`).
  - **Bug found and fixed while building this**: `src/api/admin.py`
    referenced `psycopg.OperationalError` without importing `psycopg` —
    caught immediately on first test run, fixed by adding the import.
- **Verified for real**: `tests/test_admin_auth.py` (10 tests) — correct-
  credentials login, case-insensitive email, wrong-password/unknown-email
  both 401, a token obtained through the *real* HTTP login endpoint (not
  hand-constructed) actually reaching the protected GET endpoint (proven
  via 404-not-401), an expired token rejected, a tampered token rejected, a
  wrong-secret-signed token rejected, plus two tests of the `jwt_auth`
  module directly. `tests/test_ingestion.py` updated to pass a real admin
  token (new `admin_token` fixture — creates a real admin, logs in through
  the real endpoint) on every GET call, plus two new tests confirming the
  endpoint genuinely 401s with no token or an invalid one. All three test
  files (`test_ingestion.py` + `test_admin_auth.py` + `test_tokenization.py`)
  were run together as one `pytest tests/ -v` to rule out any fixture
  ordering/interaction issue between the new admin-auth fixtures and the
  existing ingestion tests.
- Test counts: loan-platform-a **54/54** (+13), loan-platform-b **55/55**
  (+13), loan-platform-c **57/57** (+13), loan-platform-d **56/56** (+13);
  central-fraud-intelligence **38/38** (16 tokenization + 10 admin-auth + 12
  ingestion, +28 net over Phase 8's 10). **260 tests total across the
  project.**
- Added `cryptography`/`bcrypt` to `central-fraud-intelligence/requirements.txt`
  (both were already in use; the requirements file just hadn't caught up).
  Wired `CENTRAL_HMAC_MASTER_KEY`/`CENTRAL_AES_KEY` into all four
  `loan-platform-*` services' environment blocks in `docker-compose.yml`
  (previously only `central-fraud-intelligence` had them, even though
  `config.js` on every platform already read them with safe dev-fallback
  defaults — a real Docker/production wiring gap, now closed, not just a
  latent default). Expanded `.env.example`'s comments for these three
  variables to state explicitly which services consume each one.
- Updated `API.md` (new `POST /api/v1/admin/login` row; GET endpoint's auth
  column updated from "none yet" to real JWT), `DATA_FLOW.md` (Step 3
  rewritten: tokenization happens upstream on the platform, not centrally,
  so the central app never needed a decrypt/tokenize step of its own),
  `SECURITY.md` (design table filled in with real file/line references and
  per-row status; "Planned controls" section updated line-by-line to
  done/not-done; new "Phase 9 addition" section), `TESTING.md` (new counts;
  new "Phase 9: tokenization and admin auth tests" section), all five
  services' `README.md` (status line, test counts, new sections), and
  `ARCHITECTURE.md`'s stale "PHASE 0" status line (never updated since the
  file was first written — corrected here, out of general documentation
  hygiene rather than being strictly Phase 9 scope).
- All test/build artifacts (`node_modules`, `__pycache__`, `.pytest_cache`)
  deleted or left properly gitignored after verification; not committed.

### Phase 10 additions

- **`central-fraud-intelligence` now has a real, persistent graph**:
  - `src/graph/repository.py` — `PostgresGraphRepository`, two ordinary
    Postgres tables (`graph_nodes`: `node_type`/`node_key`/`properties`
    JSONB, unique on `(node_type, node_key)`; `graph_edges`:
    `edge_type`/`source_node_id`/`target_node_id`/`properties`, unique on
    `(edge_type, source_node_id, target_node_id)`, FKs into `graph_nodes`)
    — no dedicated graph database, matching the brief's own Docker Compose
    service count and `ARCHITECTURE.md`'s stated design. Every upsert is
    `INSERT ... ON CONFLICT ... DO UPDATE` with a shallow JSONB `||` merge
    of `properties`, so re-applying the same event never duplicates a
    node/edge and never clobbers properties a different call already set.
  - `src/graph/graph_store.py` — `GraphStore`, materializing those rows
    into an in-memory `networkx.MultiDiGraph`. Node identity in the
    in-memory graph is the composite string `"{node_type}:{node_key}"`;
    real identity is always the underlying `(node_type, node_key)` pair.
    Materialization is **lazy** (first real graph operation after a
    process boot triggers it) rather than an eager blocking call at
    startup — a deliberate, documented deviation from `ARCHITECTURE.md`'s
    original "on startup" wording, because an eager blocking Postgres call
    during FastAPI startup would violate this project's own boot-order
    lesson (`/health` must never depend on database reachability, learned
    the hard way in Phase 2 and applied proactively in Phase 8/9). A
    best-effort, never-fatal warm-up attempt still runs at startup via a
    `lifespan` context manager (switched from the deprecated
    `@app.on_event("startup")` while at it) so a normally-running process
    still pays that cost at boot rather than on its first event; a failure
    there is caught and logged, and the lazy guard means the very next
    real graph operation retries automatically.
  - `apply_application_submitted()`/`apply_application_status_changed()` —
    the actual entity-resolution logic: an `APPLICATION_SUBMITTED` event
    creates/updates Borrower (keyed `platformId:userId` — a platform-local
    identity, deliberately not collapsed across platforms, since the
    shared token nodes below are what's *supposed* to reveal cross-platform
    overlap), Application (keyed by the globally-unique `applicationId`,
    carrying loan amount/purpose/tenure/status as properties — no separate
    Loan node; see the design-scoping note below), Platform, and per-token
    nodes (name/email/phone/address — device/bank tokens skipped, no
    platform collects that data yet) plus the edges between them, all from
    `event["applicantTokens"]` — never a raw applicant field (there isn't
    one on the payload since Phase 9). An `APPLICATION_STATUS_CHANGED`
    event just updates the existing Application node's `status` property.
  - `src/api/applications.py`'s `POST /api/v1/applications/process` now
    calls `graph_store.apply_event()` right after persisting the event,
    whenever the stored event's status isn't already `GRAPH_UPDATED` (not
    only on first delivery — this also makes a genuine redelivery of a
    previously graph-failed event self-heal, since `GraphStore`'s upserts
    are idempotent and safe to retry). `src/ingestion/repository.py`
    gained `update_status()` to advance the event to `GRAPH_UPDATED` on
    success.
  - Deliberate scoping decisions, all documented in `ARCHITECTURE.md`
    section 6 rather than left implicit: no separate Loan node (the data
    model has no independent loan identity — one loan per application,
    never referenced elsewhere — so it's Application node properties
    instead, avoiding "unnecessary infrastructure" this project's own
    stated philosophy argues against); no `BORROWER_HAS_MULTIPLE_APPLICATIONS`
    edge (derived from existing `BORROWER_SUBMITTED_APPLICATION` out-edges
    via a graph query, not new information to double-store); a new
    NameToken node type + `BORROWER_HAS_NAME` edge, extending
    `ARCHITECTURE.md`'s original node-type list the same way Phase 9's
    `tokenize.js` already tokenizes `name` beyond `SECURITY.md`'s original
    table.
- **Verified for real**: `tests/test_graph.py` (5 tests) — node/edge
  creation from a real HTTP-level submitted-application event with exact
  node/edge counts and property assertions; a status-changed event
  updating the existing Application node in place (node count unchanged,
  `loanAmount` from the earlier event survives the merge); **entity
  resolution** — two applications from two different platforms whose
  applicants share one phone token resolve to exactly one shared
  `phone_token` node with two separate edges into it, not two unrelated
  nodes (the concrete proof this phase's whole reason for existing
  actually works); idempotent replay (HTTP-level redelivery, and a direct
  `GraphStore.apply_event()` call bypassing the idempotency-key layer
  entirely) not duplicating node/edge counts; and a **second, independent
  `GraphStore` instance** against the same test database, which has never
  had `upsert_node`/`upsert_edge` called on it, reconstructing an
  identical graph (same counts, properties, and edge-derived neighbor
  relationships) purely by rehydrating from Postgres — the concrete
  meaning of "materialized from Postgres" given lazy rather than
  eager-at-startup loading.
- **Also verified directly** (mirroring Phase 8's own boot-order check):
  pointed `CENTRAL_DB_URI` at an intentionally unreachable host and
  confirmed `/health`/`/api/v1/health` stayed 200 (the lifespan warm-up
  caught the connection failure and logged a warning, not fatal) while
  `POST /api/v1/applications/process` correctly returned 503 — the same
  boot-order guarantee every other database dependency in this app already
  has, now proactively extended to the graph store too rather than
  discovered as a bug later.
- **One existing test updated to match the new real behavior, not a
  regression**: `test_ingestion.py::test_accepts_valid_event_and_is_retrievable`
  previously asserted `status == "RECEIVED"` (accurate as of Phase 8, when
  nothing advanced it further); now asserts `status == "GRAPH_UPDATED"`,
  since the same POST request now completes the graph step too.
- Test counts: central-fraud-intelligence **43/43** (38 prior + 5 new
  `test_graph.py`; loan platforms unchanged at 222 since this phase
  touched only the central app). **265 tests total across the project.**
- Added `networkx==3.6.1` to `requirements.txt` (was already installed in
  this sandbox and listed in `ARCHITECTURE.md` as the chosen library;
  requirements file just hadn't caught up — same situation Phase 9 found
  for `cryptography`/`bcrypt`).
- Updated `ARCHITECTURE.md` (section 6 rewritten with real implementation
  detail and honest scoping notes for every originally-listed node/edge
  type; section 7's pipeline status updated; top status line), `API.md`
  (POST endpoint description, response `status` value, 503 case),
  `DATA_FLOW.md` (new Step 4 update section; pipeline diagram; "What still
  goes here" renumbered to Phases 14-18), `SECURITY.md` (the "never
  written into the central graph" privacy rule is now a checkable claim,
  not just a stated intent — noted with a pointer to the verifying test),
  `TESTING.md` (new counts; new "Phase 10: persistent graph tests"
  section; planned-test-surfaces table updated), and
  `central-fraud-intelligence/README.md` (status, graph section, layout
  table, testing section, boot-order verification note).
- All test/build artifacts (`__pycache__`, `.pytest_cache`) deleted or
  left properly gitignored after verification; not committed.

### Phase 11 additions

- **This session's sandbox had reset since the Phase 10 checkpoint** —
  worth recording since it changed what "CONTINUE FROM CHECKPOINT" had to
  do this time: the working directory environment reported a reset,
  PostgreSQL 16 was installed but not running (`pg_lsclusters` showed
  `down`), and none of the four loan platforms' `node_modules` were
  present. All state on disk (git history, committed source, the
  Postgres data directory itself) was intact — only running services and
  reinstallable packages were gone. Fixed by `pg_ctlcluster 16 main start`
  (confirmed the `quickfundx` role, both `central_fraud_intelligence`/
  `central_fraud_intelligence_test` databases, and all prior tables
  survived) and `npm install` in each of the four `loan-platform-*/backend`
  directories, then every test suite re-run for real (not assumed from
  the prior session's checkpoint report) before any new work began — the
  "NO FAKE SUCCESS" rule applied to inherited state, not just new claims.
- **`GraphStore.get_relevant_subgraph()` is real** — the first thing in
  this project that reads the persistent graph back out; Phase 10 only
  ever wrote to it. Bounded, direction-agnostic BFS around one node
  (`node_type`, `node_key`), both successors and predecessors followed at
  each step since fraud-relevant paths run in both directions (e.g.
  Application ← Borrower → shared token ← a different Borrower →
  different Application). Returns a plain, JSON-serializable dict —
  `{"root", "hops", "nodeCount", "edgeCount", "truncated", "nodes",
  "edges"}` — not a networkx object, so nothing outside `src/graph/` needs
  to depend on networkx's API.
  - `hops` defaults to 4 — the minimum hop count that can reach a
    *different* application through one shared token (Application →
    Borrower → token → different Borrower → that Borrower's Application);
    fewer hops would silently miss the exact cross-application linkage
    signal this whole graph exists to surface.
  - `max_nodes` defaults to 500 — bounds how much of the graph a single
    unusually high-degree "hub" token (shared by many borrowers) can pull
    into one call; the traversal stops expanding once the visited set
    hits this size and reports `truncated: true`, without discarding
    already-visited nodes or the edges among them.
  - Neighbors are visited in sorted node-id order at each step, so a
    truncated result is deterministic across runs rather than dependent
    on Python set-iteration order.
  - The result is a genuine **induced** subgraph: every edge between two
    visited nodes is included, not only the edges BFS actually traversed
    to discover them — verified directly by a test where two borrowers
    share both a phone and an email token, and both shared-token edges
    appear even though BFS only needed one of the two to reach both
    borrowers.
- **Two scope decisions made and documented, not silently assumed** (in
  `graph_store.py`'s Phase 11 docstring, `ARCHITECTURE.md` section 6,
  `SECURITY.md`'s new Phase 11 note, and `central-fraud-intelligence/
  README.md`):
  1. Entity resolution stays exact-token-match, exactly as Phase 10 built
     it — **not** extended to fuzzy/probabilistic linking. This is a
     direct consequence of Phase 9's tokenization choice, not a shortcut:
     HMAC-SHA256 has the avalanche property, so two inputs differing by
     even one character produce completely uncorrelated digests, making
     any similarity comparison on the token values themselves
     cryptographically impossible without a fundamentally different (and
     weaker-privacy) scheme such as locality-sensitive hashing.
     `tokenize.js`/`tokenize.py`'s `normalize()` already absorbs the
     fuzziness that's safe to absorb *before* hashing (phone: strip
     non-digits; email: lowercase; address/name: lowercase + collapse
     whitespace) — genuine fuzzy matching beyond that would require
     redesigning Phase 9's tokenization scheme itself, which is out of
     scope here. Revisit only if Phase 21's false-negative testing shows
     this genuinely costs recall.
  2. No graph retention/pruning policy is implemented. This project's
     graph is expected to stay small — a bounded synthetic demo dataset
     across four platforms, not a production-scale, years-deep
     transaction graph — so Postgres and an in-memory NetworkX graph both
     handle it without difficulty. Pruning old data would also actively
     work against the point of the system: an old shared phone/email
     token is exactly the kind of long-lived signal a fraud ring depends
     on, so discarding it early would destroy evidence, not just save
     space. A real production deployment would likely want time-windowed
     retention or cold-storage archiving, but building that now would be
     unneeded infrastructure this project never grows large enough to
     need — a stated decision, not an oversight.
- **`get_relevant_subgraph()` is deliberately not wired into any HTTP
  route or the live `POST /api/v1/applications/process` request path** —
  there is no real consumer for its output until Phase 14's GNN inference
  exists to call it per incoming event; calling it eagerly on every event
  now would only be discarded work. `src/main.py`'s module docstring and
  root `/` endpoint message, and `ARCHITECTURE.md` section 7, are explicit
  about this so it doesn't read as an oversight later.
- **Verified for real**: `tests/test_subgraph.py` (7 tests, against the
  real `central_fraud_intelligence_test` Postgres database via the
  `graph_store` fixture, building graph state directly with `upsert_node`/
  `upsert_edge` rather than the HTTP layer since this phase is about
  reading, not re-testing ingestion) — nonexistent node returns `None`;
  an isolated node returns itself with zero edges; the default hop count
  provably reaches a sibling application across a shared token while that
  application's own platform node (one hop further) correctly stays out
  — hand-traced the exact BFS expansion order before writing the
  assertions, then ran the test to confirm the trace was right rather
  than adjusting the code to match a guess; `hops=1` measurably limits the
  result to the root's immediate neighbors; `max_nodes=4` measurably
  truncates mid-traversal and reports `truncated: true`; two identical
  calls return byte-identical results; and the induced-subgraph proof
  described above.
- Test counts: central-fraud-intelligence **50/50** (43 prior + 7 new
  `test_subgraph.py`; loan platforms unchanged at 222, reinstalled and
  reverified this session since `node_modules` had to be restored).
  **272 tests total across the project.**
- Boot-order guarantee reverified directly (same check as every prior
  phase): `CENTRAL_DB_URI` pointed at an unreachable host still leaves
  `/health`/`/api/v1/health` at 200 (now reporting `version:
  "0.11.0-phase11"`) while `POST /api/v1/applications/process` correctly
  returns 503 — `get_relevant_subgraph()` sitting unused in the request
  path changes nothing here, confirmed rather than assumed.
- `src/config.py` `VERSION` bumped `0.10.0-phase10` → `0.11.0-phase11`; no
  new environment variables — the hop/node-count bounds are call-time
  parameters with in-code defaults, not env-driven, since nothing yet
  calls this with anything other than the defaults.
- Updated `ARCHITECTURE.md` (top status line; section 6's new "Reading the
  graph back out" subsection plus the two scope-decision paragraph;
  section 7's pipeline-step status), `DATA_FLOW.md` (status line; new
  "Step 4 update — Phase 11" subsection; "What still goes here" list;
  pipeline ASCII diagram), `SECURITY.md` (new paragraph on the HMAC
  avalanche property's direct consequence for entity-resolution scope),
  `TESTING.md` (new counts; new "Phase 11: subgraph extraction tests"
  section; planned-test-surfaces table row split into Phase 10/Phase 11),
  and `central-fraud-intelligence/README.md` (status header; new subgraph-
  extraction section; "What this phase does NOT do" list; layout table's
  `entity_resolution/`/`graph/` rows; testing section and counts).
- All test/build artifacts (`__pycache__`, `.pytest_cache`) deleted or
  left properly gitignored after verification; not committed.

### Phase 12 additions

- **`central-fraud-intelligence/datasets/scenario_generator.py`
  (`generate_catalog(seed, instances_per_pattern)`) is real** - a
  deterministic synthetic fraud scenario generator covering 17 of
  `DEMO_SCENARIOS.md`'s 20 catalog patterns. Design decisions made and
  stated up front, not discovered mid-build:
  - All applicant PII is obviously synthetic (`Synthetic Applicant
    00042`, `identity00042@synthetic.quickfund-x.test`, phone numbers
    built from a `9000000000 +` offset, `Synthetic Lane`/`Test City`
    addresses) - no resemblance to any real person's data.
  - Every applicant is tokenized via the real
    `src/tokenization/tokenize.py::tokenize_applicant()` (the same
    function every loan platform's `tokenize.js` mirrors), not
    hand-written token hex strings - "two identities share a phone
    number" and "two identities' phone tokens collide" are the same fact
    here as for a real applicant.
  - Every scenario's `applications` list is a list of real
    `APPLICATION_SUBMITTED` event payloads - the exact shape
    `GraphStore.apply_event()` (and, in principle,
    `POST /api/v1/applications/process`) already consumes. No second,
    parallel "training example" format was invented for the GNN to need
    its own loader for - a decision made explicitly before writing any
    generator code, per the standing instruction that data should
    "exercise the real graph... not a separate parallel data format".
  - **3 of the 20 catalog patterns are deliberately NOT generated**: #6
    (same device), #7 (same bank token), #16 (legitimate family shared
    device) - all three need a DeviceToken/BankToken graph node, and
    Phase 10 already established neither exists because no loan platform
    collects that data yet. `DEFERRED_PATTERNS` documents this the same
    way Phase 9/10 documented device/bank tokenization as "implemented,
    not yet collected". Generating fake device-token values the graph has
    nowhere to put would have been exactly the kind of fake coverage this
    project's checkpoint discipline exists to prevent.
  - **Fraud vs. genuine is not purely a graph-topology decision**:
    patterns 1-3 (same identity across N platforms) and pattern 19 (a
    legitimate cross-platform applicant) deliberately produce the
    *identical* graph shape - full token overlap across N Borrower
    nodes. What distinguishes them in the generated data is
    `submittedAt` timing (minutes apart vs. months apart - real
    `datetime`/`timedelta` arithmetic, not hand-rolled date-string
    offsets, which would have silently produced invalid dates past 31
    days), mirroring `ARCHITECTURE.md` section 8's requirement that a
    single signal (shape alone) must not by itself determine an
    incident.
  - A core parameterized building block (`_identity_cluster_scenario`)
    covers most of the catalog (patterns 1-5, 8-10, 12, 15, 19) by
    varying which fields are shared, across how many platforms, and how
    far apart in time - plus five dedicated functions for shapes that
    building block can't express: document reuse (11), a
    hub-with-one-shared-signal (12's variant), a three-node chain and a
    four-node ring where adjacent pairs share *different* signal types
    (13/14 - deliberately exercising the multi-hop traversal Phase 11's
    `get_relevant_subgraph` exists for), repeated application from the
    same Borrower (17), an isolated baseline (18), and a composite
    "bridge" scenario (20) that links one already-generated SUSPICIOUS
    scenario and one already-generated GENUINE scenario into one mixed
    neighborhood by reusing their real token values, not fabricating new
    ones.
- **`scripts/generate_scenarios.py`** - the CLI entrypoint. Writes
  `seed-data/scenarios/scenarios.json` and, by default, also applies
  every generated event to a real `GraphStore` against a real Postgres
  database to prove the dataset is genuinely ingestable before being
  trusted, not just structurally plausible. Defaults to the **test**
  database (`CENTRAL_DB_TEST_URI`, with the identical literal fallback
  `tests/conftest.py` uses) specifically so a plain no-flags run can
  never truncate the dev database by accident.
- **The committed dataset, real counts** (seed `12345`,
  `instances_per_pattern=3`): 51 scenarios, 129 synthetic applications,
  39 labeled SUSPICIOUS, 12 labeled GENUINE (9 of those hard negatives),
  17 patterns at exactly 3 instances each. Verified for real: every one
  of the 129 events applied to a real `GraphStore` against the real
  `central_fraud_intelligence_test` database with zero errors before this
  file was written or the dataset committed.
- **Verified for real**: `tests/test_scenario_generator.py` (14 tests) -
  determinism given a seed; different seeds differ; every
  scenario/application/event id is unique across the whole catalog; the
  catalog covers exactly pattern numbers `{1..20} - {6,7,16}`; every
  scenario has a valid label; targeted structural checks per pattern
  family (shared-single-token, identical-topology-different-timing,
  repeated-application, document-reuse, hub, ring, the bridge composite);
  and an integration test that applies the entire generated catalog to a
  real `GraphStore` against the real test database with zero errors, then
  spot-checks the *real, persisted* graph (not the in-memory Python
  objects) - the hub scenario's shared phone token has exactly 5
  `BORROWER_HAS_PHONE` edges into it, read back via Phase 11's
  `get_relevant_subgraph()`.
- Test counts: central-fraud-intelligence **64/64** (50 prior + 14 new
  `test_scenario_generator.py`; loan platforms unchanged at 222).
  **286 tests total across the project.**
- `src/config.py` `VERSION` deliberately left at `0.11.0-phase11` - this
  phase added zero request-handling code (`datasets/`/`scripts/` are
  offline tooling only), so bumping the running app's version would
  misrepresent what changed; documented directly in `config.py`'s
  docstring rather than left as an unexplained inconsistency.
- Added `datasets/__init__.py` (matching every other `src/` subpackage's
  convention, for the `from datasets.scenario_generator import ...`
  import in tests to be an ordinary package import rather than relying on
  implicit namespace packages).
- Updated `ML_MODEL.md` (new "Phase 12: real training data composition"
  section with the actual measured table, replacing the placeholder item
  4 in "What will go here"), `DEMO_SCENARIOS.md` (per-pattern
  generated/deferred status in the catalog list, real scenario JSON shape
  replacing the original sketch, real coverage statement with actual
  numbers), `TESTING.md` (new counts; new "Phase 12: scenario generator
  tests" section; planned-test-surfaces table row), `central-fraud-
  intelligence/README.md` (status header; new Phase 12 section; "What
  this phase does NOT do" list; layout table's `datasets/`/`scripts/`
  rows; testing section and counts; new "Generating the synthetic
  scenario dataset" usage section), `central-fraud-intelligence/
  requirements.txt` (phase-comment block; no new dependency), and
  `seed-data/scenarios/README.md` (real dataset structure and counts,
  replacing the "not yet generated" stub).
- All test/build artifacts (`__pycache__`, `.pytest_cache`) deleted or
  left properly gitignored after verification; not committed.

### Phase 13 additions

- **`central-fraud-intelligence/src/graph/graph_store.py` gained
  `export_all()`** - the whole-graph counterpart to Phase 11's bounded
  `get_relevant_subgraph()`, needed because training needs every one of
  the dataset's 51 disconnected scenario components, not a hop-bounded
  neighborhood around one root node. Returns the same plain
  nodes/edges JSON shape as `get_relevant_subgraph()` (nodeType/nodeKey/
  properties, edgeType/source/target/properties), minus the
  root/hops/truncated fields that don't apply to a whole-graph export.
  Proven, not just asserted, by a dedicated test: two entirely
  disconnected components both come back from one `export_all()` call,
  while a `get_relevant_subgraph()` call rooted in one of them provably
  cannot reach the other.
- **`central-fraud-intelligence/src/gnn/features.py` is real** - per-node
  16-dim feature engineering from `export_all()`'s plain node list: 8-dim
  node-type one-hot, 2 structural degree features (log-scaled combined
  and out-only), and 6 Application-specific fields (scaled loan amount,
  4-dim platform one-hot, scaled `submittedAt` timestamp). The timestamp
  feature is not cosmetic - it's the only feature dimension that differs
  between Phase 12's deliberately-identical-topology patterns 1-3 (same
  identity, minutes apart, SUSPICIOUS) and pattern 19 (same identity,
  months apart, GENUINE); proven directly by a test that shows every
  other dimension is identical between the two cases.
- **`central-fraud-intelligence/src/gnn/model.py` is real** -
  `HeteroSAGELayer`/`HeteroGraphSAGE`: a lightweight heterogeneous
  GraphSAGE-style model, pure PyTorch, no PyG/DGL (confirming, not
  revising, the working assumption `ML_MODEL.md` carried since Phase 10 -
  see that file's Phase 13 section for the full reasoning now that real
  data exists to check it against). Every one of the graph's 7 real edge
  types gets an explicit learned reverse relation (`<TYPE>_rev`), so
  message passing is bidirectional - directly mirroring Phase 11's
  direction-agnostic BFS design, for the same reason (fraud signals cross
  a directed graph both ways). `num_layers=4` is deliberately set equal
  to `graph_store.DEFAULT_SUBGRAPH_HOPS` (also 4) so the model's
  receptive field matches exactly the neighborhood Phase 11 established
  as sufficient - a cross-phase consistency a dedicated test pins so it
  can't silently drift.
- **`central-fraud-intelligence/training/dataset.py` is real** - builds a
  full-graph, scenario-stratified train/eval dataset from
  `seed-data/scenarios/scenarios.json`: truncates a target database, real-
  applies every scenario's events via the real `GraphStore.apply_event()`
  (the same code path production ingestion uses), calls `export_all()`,
  builds features/adjacency, and propagates each scenario's single
  `expected` label onto every one of its Application nodes. The
  train/eval split is scenario-stratified (never application-random, to
  avoid splitting one scenario's applications across both sides) and
  fully deterministic - grouped by pattern, sorted by `scenario_id`,
  `round(count * 2/3)` (minimum 1) to train, the rest to eval - proven by
  tests to be leak-free (no application in both splits) and to cover
  every pattern with ≥2 instances in both splits.
- **`central-fraud-intelligence/training/train.py` and
  `training/baseline.py` are real**, and a real training run happened:
  200 epochs, seed `12345`, `hidden_dim=32`, `num_layers=4`, against the
  real committed 51-scenario/129-application dataset (590 total graph
  nodes). `BCEWithLogitsLoss` with `pos_weight` from the real train-split
  class counts (72 SUSPICIOUS / 14 GENUINE), Adam. Real measured eval
  results on the held-out 43-application split (36 SUSPICIOUS / 7
  GENUINE): **accuracy 0.9767, precision 0.9730, recall 1.0000, F1
  0.9863**, confusion matrix `[[6,1],[0,36]]`. A non-graph
  `LogisticRegression` baseline (`training/baseline.py`), trained on the
  IDENTICAL per-node features and split but with zero access to the
  graph's edges, scored accuracy 0.9070 / F1 0.9474 / `[[3,4],[0,36]]` -
  the GNN's entire measured advantage is on the GENUINE side (6/7 correct
  vs. 3/7), the specific comparison `ML_MODEL.md` item 7 exists to make.
  **Honestly flagged, not hidden**: the one GNN false positive was
  `scenario_0047_normal_genuine_baseline` (not one of the deliberately
  hard `hard_negative` cases, which were all classified correctly),
  predicted SUSPICIOUS at probability 1.0; and final train loss reaching
  `3.86e-05` after 200 epochs on only 86 training applications is real
  evidence of overfitting to this specific small dataset, with a concrete
  documented improvement plan (more `--instances-per-pattern`, weight
  decay/more dropout, and eventually the 3 deferred device/bank patterns)
  rather than the number being quietly accepted at face value. Full
  writeup, including the "this is not a claim of real-world performance"
  explicit non-claim, in `ML_MODEL.md`'s Phase 13 section.
  `models/gnn_checkpoint.pt` (git-ignored) +
  `models/gnn_checkpoint.meta.json` + `models/baseline_metrics.json`
  (both committed - real architecture/args/loss-curve/eval-metrics, not a
  hand-written summary of them) are the artifacts of this run.
- **Environment limitation found and documented**: this build sandbox's
  egress policy blocks `download.pytorch.org` (403, same class of issue
  as the standing Phase 1 Docker Hub block) - the official CPU-only torch
  wheel install command fails here. Falling back to plain `pip install
  torch` (PyPI's default index, not blocked) succeeds but pulls PyPI's
  current default wheel, which bundles the full CUDA 13 runtime (~20
  `nvidia-*` packages, ~1.2GB for torch alone) even though this sandbox
  has no GPU (`torch.cuda.is_available()` confirmed `False`). Verified
  this still trains/runs correctly on CPU - a disk-space/install-size
  inefficiency, not a functional defect. Documented in
  `TROUBLESHOOTING.md` (new Phase 13 entry, mirroring the Phase 1 entry's
  structure) and in `requirements.txt`'s comment block; `requirements.txt`
  pins version-only `torch==2.14.0`/`numpy==2.4.4`/`scikit-learn==1.8.0`
  (resolves against either wheel), and the Dockerfile gained a comment
  noting a real deployment build should add
  `--extra-index-url https://download.pytorch.org/whl/cpu` for a much
  smaller image - left as a note, not guessed-at code, since Phase 24 is
  where that Dockerfile actually gets built/exercised.
- **Verified for real**: `tests/test_gnn.py` (21 new tests) + 2 new tests
  in `tests/test_graph.py` for `export_all()` - see `TESTING.md`'s "Phase
  13: GNN training tests" section for the full breakdown. None of these
  tests assert a specific trained accuracy number (that would test a
  stochastic outcome, not code correctness) - shapes, determinism, no
  data leakage, and a cheap loss-decreases-on-a-toy-graph smoke test are
  what's actually asserted; the real training run's real numbers are
  reported honestly above and in `ML_MODEL.md`/`models/*.json` instead.
- Test counts: central-fraud-intelligence **87/87** (64 prior + 21 new
  `test_gnn.py` + 2 new in `test_graph.py`; loan platforms unchanged at
  222). **309 tests total across the project.**
- Confirmed (grep, not assumed) that nothing under `src/gnn/` or
  `training/` is imported by `src/main.py` or any `src/api/*` router -
  this phase touched zero request-handling code, so the
  `/health`-never-depends-on-Postgres boot-order guarantee is untouched
  by construction. `src/config.py` `VERSION` deliberately left at
  `0.11.0-phase11` for the same reason Phase 12 gave (documented directly
  in `config.py`'s docstring, now with a Phase 13 paragraph too) - Phase
  14 (wiring inference into the live request path) is what will actually
  move it.
- Updated `ML_MODEL.md` (major: items 1-3/5-7 filled in with what was
  actually built/measured, full honest Phase 13 results writeup, a new
  explicit non-claim about eval-accuracy-vs-real-world-generalization),
  `ARCHITECTURE.md` (GNN row now says "implemented and trained as of
  Phase 13"), `TESTING.md` (new counts; new "Phase 13: GNN training
  tests" section; planned-test-surfaces table split into "GNN training
  (done)"/"GNN inference"), `central-fraud-intelligence/README.md`
  (status header; new Phase 13 section; "What this phase does NOT do"
  list; layout table's `gnn/`/`training/`/`models/` rows; testing section
  and counts; new "Training the GNN" usage section), `central-fraud-
  intelligence/requirements.txt` (real torch/numpy/scikit-learn pins +
  detailed environment-limitation comment), `central-fraud-intelligence/
  Dockerfile` (comment note re: CPU-only wheel for production),
  `TROUBLESHOOTING.md` (new Phase 13 entry), and added
  `central-fraud-intelligence/models/README.md` (new - explains which
  files are git-ignored vs. committed and why).
- All test/build artifacts (`__pycache__`, `.pytest_cache`) deleted or
  left properly gitignored after verification; not committed.

### Phase 14 additions

- **`central-fraud-intelligence/src/gnn/graph_tensors.py` is new** -
  extracted the tensor-construction logic (node index / per-node features
  / relation-keyed adjacency, from a plain nodes/edges dict) that used to
  live inline inside `training/dataset.py`'s `build_dataset()`, into one
  shared function, `build_graph_tensors()`. `training/dataset.py` now
  calls it too (refactor, verified behavior-identical by the existing
  `test_gnn.py` suite still passing unchanged). This exists specifically
  so offline training and live inference (below) can never silently
  diverge into two different tensor-construction implementations, while
  preserving the Phase 13 layering rule that `src/` never imports from
  `training/` (this module lives under `src/gnn/`, so `training/dataset.py`
  importing *it* doesn't violate that - confirmed by grep, same as every
  prior phase's layering claim).
- **`central-fraud-intelligence/src/gnn/inference.py` is new** -
  `GNNScorer`: `load()` (classmethod) reconstructs the exact trained
  `HeteroGraphSAGE` from `models/gnn_checkpoint.meta.json`'s recorded
  architecture and loads `models/gnn_checkpoint.pt`'s state dict into it;
  wraps everything in try/except and returns `None` on any failure
  (missing files, malformed JSON, a meta.json missing a required key, a
  checkpoint that isn't a valid torch file) rather than ever raising -
  same best-effort posture as every other optional dependency in this
  app. `score_subgraph()` builds tensors via `build_graph_tensors()`,
  runs one forward pass under `torch.no_grad()`, and returns
  `sigmoid(logit)` for the requested root node - a real probability in
  `[0, 1]`, or `None` if that root isn't actually present in the given
  subgraph.
- **Wired into the live request path** -
  `central-fraud-intelligence/src/main.py`'s lifespan now calls
  `GNNScorer.load()` once at startup (`app.state.gnn_scorer`, `None` if
  unavailable, logged either way, never fatal to boot).
  `src/api/applications.py`'s `POST /api/v1/applications/process` now
  runs inference whenever an event's status is `GRAPH_UPDATED` and a
  scorer was loaded: calls `GraphStore.get_relevant_subgraph()` (Phase
  11's first real caller), scores it, and on success persists a real
  score (new `fraud_score` column on `ingested_events`,
  `repository.update_analysis()`) and advances `status` to `ANALYZED`.
  Wrapped in a broad try/except - a missing checkpoint or any scoring
  failure leaves the event at `GRAPH_UPDATED` with no score, the request
  still returns 200, and a later redelivery retries it, exactly like the
  Phase 10 graph-update step. Both `POST .../process` and
  `GET /api/v1/applications/:id` now return `fraudScore` per event.
- **Bug caught and fixed before it shipped, not found by a failing test**:
  the graph-update step's original gate (`if status != "GRAPH_UPDATED"`)
  was correct back when only two statuses existed, but would have
  wrongly re-run the graph update *and* regressed an already-`ANALYZED`
  event's status back to `GRAPH_UPDATED` on a plain redelivery, now that
  a third status exists. Replaced with explicit `if status == "RECEIVED"`
  (graph step) / `if status == "GRAPH_UPDATED" and gnn_scorer is not None`
  (inference step) - both correct for all three statuses with no
  special-casing. Directly covered by a dedicated regression test,
  `test_already_analyzed_event_is_not_rescored_on_redelivery`.
- **Test-determinism issue found and fixed**: because the real Phase 13
  checkpoint genuinely exists on disk in this build sandbox, the shared
  `client` test fixture's real FastAPI lifespan was loading a real
  `GNNScorer` and actually running inference inside pre-existing tests
  that only expected status `GRAPH_UPDATED` - 3 tests failed
  (`test_graph.py` x2, `test_ingestion.py` x1) not because of a bug in
  the new code, but because those tests' outcomes now silently depended
  on whether a git-ignored binary file happened to be present. Fixed by
  making the `client` fixture force `get_gnn_scorer` to `None` by
  default (mirroring the existing override pattern for every other
  production dependency in that fixture) rather than editing the 3
  tests' assertions, which would then have been wrong in a fresh
  environment without a trained checkpoint. Re-ran those 3 tests after
  the fix: passed unchanged, zero edits to their own code.
- **Verified for real**: new `tests/test_inference.py` (14 tests) -
  `GNNScorer.load()`'s failure modes; `score_subgraph()` against a real
  extracted subgraph (the real committed checkpoint genuinely loads and
  scores in this sandbox, not skipped); the live POST/GET wiring via
  stub-scorer dependency overrides (success, scoring-raises-and-falls-
  back, redelivery-retry, already-`ANALYZED`-not-rescored,
  fraudScore-on-GET); and a real-app boot-order test (real
  `src/main.py` lifespan, `GNNScorer.load` monkeypatched to run its real
  implementation against a deliberately corrupt checkpoint) proving
  `/health`/`/api/v1/health` still return 200 and
  `app.state.gnn_scorer is None` when the checkpoint is broken - the same
  guarantee Phase 8/10 already proved for an unreachable Postgres, now
  proved for a broken GNN checkpoint too. One test-writing bug caught and
  fixed before the suite was reported passing: the boot-order test's
  monkeypatch lambda initially called `GNNScorer.load(...)` from inside
  itself, which re-resolved through the patched class at call time and
  recursed into itself with a signature mismatch - fixed by capturing a
  reference to the real (pre-patch) classmethod first.
- Test counts: central-fraud-intelligence **101/101** (87 prior + 14 new
  `test_inference.py`; loan platforms re-run and unchanged at 222 -
  54+55+57+56, verified for real this phase, not assumed). **323 tests
  total across the project.**
- `src/config.py` `VERSION` bumped `0.11.0-phase11` → `0.12.0-phase14` -
  the first phase since 11 to actually change request-handling behavior
  (Phase 12/13 deliberately did not bump it, per their own docstrings).
- **`.dockerignore`/`Dockerfile` fixed** - `.dockerignore` no longer
  excludes `models/` wholesale (it was doing so since Phase 1, back when
  nothing read from that directory); `Dockerfile` gained
  `COPY models ./models`, with an honest comment that
  `gnn_checkpoint.pt` is git-ignored and won't be present in a fresh
  `git clone` unless `training/train.py` is run (or a pre-trained
  checkpoint is placed there) before `docker build` - the container still
  boots and serves traffic fine without it, `fraudScore` just stays
  `null`. Not yet verified against a real Docker build (standing Docker
  Hub egress block, same as every prior phase).
- Updated `ARCHITECTURE.md` (section 6/7 - `get_relevant_subgraph`'s real
  caller, the full pipeline now real through `run_gnn_inference`, the
  `ANALYZED` status is real), `API.md` (both applications endpoints'
  real request/response shapes updated with `fraudScore`/`ANALYZED`),
  `DATA_FLOW.md` (new "Step 4 update — Phase 14" section, pipeline
  diagram, "What still goes here" list), `SECURITY.md` (new Phase 14
  consequence note - inference never sees raw PII, no new secret/key
  surface), `TESTING.md` (new counts, new "Phase 14: GNN inference
  tests" section, planned-surfaces table row marked done), `ML_MODEL.md`
  (status header, new Phase 14 section with the full inference design
  writeup), `central-fraud-intelligence/README.md` (status header, new
  Phase 14 section, "What this phase does NOT do" list, layout table,
  testing section/counts, new "GNN inference (Phase 14)" usage section,
  Docker section caveat), `TROUBLESHOOTING.md` (Phase 14 addendum to the
  existing torch/CUDA entry - inference now carries the same install-size
  tradeoff every long-lived server process, not just one offline training
  run).
- All test/build artifacts deleted or left properly gitignored after
  verification; not committed.

### Phase 15 additions

- **`central-fraud-intelligence/src/fraud_detection/rules.py` is new** -
  `evaluate_rules(subgraph)`: a deterministic rule engine over the same
  bounded subgraph `GraphStore.get_relevant_subgraph()` already provides
  the GNN. Walks the root borrower's shared phone/email/name/address
  token edges to *distinct other* borrowers (root borrower's own repeat
  applications explicitly excluded - required for the pattern 17 hard
  negative to score zero) and shared `APPLICATION_HAS_DOCUMENT` reuse.
  Each signal is `base_weight * time_proximity_multiplier` - "application
  velocity" is deliberately folded into every shared-token check rather
  than built as a separate rule, since several SUSPICIOUS/GENUINE
  hard-negative pairs in the committed dataset share identical graph
  topology and differ only in timing. A hub bonus applies when 3+ distinct
  identities share one token. Device/bank-token signals remain
  unimplemented (no data source, same documented gap since Phase 6).
- **`central-fraud-intelligence/src/fraud_detection/scoring.py` is new** -
  `combine_signals(gnn_score, rule_result)`: weighted average
  `0.6 * gnn_score + 0.4 * rule_score`; when `gnn_score` is `None`
  (no checkpoint loaded), `final_risk_score` is `rule_score` directly,
  unscaled - a deliberate decision so a missing model can never
  structurally cap the achievable risk score below the `FRAUD_DETECTED`
  threshold. `FRAUD_DETECTED` at `final_risk_score >= 0.5`, else
  `CLEARED` - the first real values these two statuses have ever taken.
- **Wired into the live request path** -
  `src/ingestion/repository.py` gained 3 new `ingested_events` columns
  (`rule_score`, `triggered_rules`, `final_risk_score`) and
  `finalize_risk_decision()`. `src/api/applications.py`'s
  `POST /api/v1/applications/process` now runs the rule engine + hybrid
  combination whenever an event's status is `ANALYZED`, reusing the
  subgraph already fetched for GNN inference in the common case (or
  refetching it, on a redelivery / prior rule-engine failure). Wrapped in
  a broad try/except - a rule-engine or combination failure leaves the
  event parked at `ANALYZED` with the new fields `null`, retried by a
  later redelivery, exactly like every prior best-effort step. Both
  `POST .../process` and `GET /api/v1/applications/:id` now return
  `ruleScore`, `triggeredRules`, and `finalRiskScore`.
- **Test regressions found and fixed, correctly diagnosed as expected new
  behavior, not a bug**: wiring the rule engine in made a
  successfully-scored event progress automatically past `ANALYZED` to a
  real terminal status within the same request for the first time - broke
  3 pre-existing `test_inference.py` assertions that expected the old
  Phase 14 stopping point. Fixed by updating those assertions to the new
  real terminal status (computed by hand from the same 0.6/0.4 weights
  the code uses) rather than changing the code to match the old tests.
- **Verified for real**: new `tests/test_fraud_detection.py` (15 tests) -
  `evaluate_rules()`/`combine_signals()` against real graph fixtures
  built the same way `test_graph.py`/`test_subgraph.py` build theirs:
  isolated/same-borrower-repeat both score `0.0`; shared-phone
  minutes-apart vs. five-months-apart from the same time-multiplier
  function produce the two different real numbers the design claims;
  pattern 15/19 hard negatives both score low; address+phone escalates
  beyond address alone; hub bonus; document-hash signal; malformed
  subgraph degrades to `0.0` rather than raising; `combine_signals()`'s
  weighted average, `None`-GNN fallback, threshold boundary, clamping.
  `tests/test_inference.py` gained one new dedicated test for the rule
  engine's own best-effort/retry-safety guarantee (`evaluate_rules`
  monkeypatched to raise: parks at `ANALYZED`, then redelivery reaches
  `FRAUD_DETECTED` once the failure is removed). Three float-precision/
  threshold-math test bugs (not implementation bugs) caught and fixed
  during this work by actually running the suite: two exact-equality
  weight-sum assertions replaced with `pytest.approx()` (Python float
  representation, e.g. `0.30 + 0.16 != 0.46` exactly), and one threshold-
  boundary test redesigned after realizing it hadn't accounted for
  `combine_signals()` scaling `gnn_score` by 0.6 before comparing to the
  threshold.
- Test counts: central-fraud-intelligence **117/117** (101 prior + 1 new
  in `test_inference.py` + 15 new `test_fraud_detection.py`; loan
  platforms unchanged at 222). **339 tests total across the project.**
- **Real evaluation against the full committed dataset, run for real in
  this environment, both findings investigated with real code/data, not
  guessed**: new `scripts/evaluate_fraud_detection.py` replays all 129
  real applications from `seed-data/scenarios/scenarios.json` through the
  exact production pipeline functions and the real trained checkpoint, in
  two modes (`interleaved`: one shared graph, true global chronological
  order; `isolated`: each of the 51 scenarios truncated and replayed
  alone). Real measured accuracy: **73.64% interleaved / 71.32%
  isolated** (vs. Phase 13's 97.67% offline GNN-only number) - full
  output in `models/hybrid_scoring_eval.json` (committed). Two real
  findings, reported honestly rather than smoothed over:
  1. The `isolated` mode was built specifically to test a hypothesis
     written into `rules.py`'s own docstring *before* evaluation ran -
     that unrelated scenarios sharing one common base epoch causes
     cross-scenario timestamp collisions. It disproved that hypothesis
     (`isolated` scored no better, in fact slightly worse). Cross-
     referencing every false negative in both modes against each
     scenario's own internal submission order found the real cause:
     **100% of false negatives in both modes (30/30 interleaved, 33/33
     isolated) are exactly the first-submitted application within their
     own scenario** - an unavoidable structural property of real-time
     single-pass scoring (whichever ring member applies first has zero
     corroborating siblings at the moment *it* is scored), not a dataset-
     interleaving artifact. `rules.py`'s docstring was corrected to state
     the verified explanation instead of the disproven original
     hypothesis.
  2. A second, distinct, genuine GNN limitation surfaced: 3
     `legitimate_repeated_application` (pattern 17, same borrower only,
     hard negative) scenarios scored `fraudScore ≈ 0.9999` from the GNN
     itself in both modes (`ruleScore` correctly stayed `0.0` - the rule
     engine's same-borrower exclusion is working as designed and is not
     the source of this false positive), pushing `finalRiskScore` past
     the threshold to a false `FRAUD_DETECTED`. Reproduces identically in
     `isolated` mode, confirming it is a real, repeatable property of the
     trained checkpoint on returning applicants, related to but distinct
     from Phase 13's one known offline false positive
     (`normal_genuine_baseline`, also scored 1.0). Reported as an open,
     unresolved limitation - not explained further or fixed in this
     phase (that's Phase 16's explainability scope / a future retraining
     candidate, not this checkpoint's).
- `src/config.py` `VERSION` bumped `0.12.0-phase14` → `0.13.0-phase15`.
- Updated `ARCHITECTURE.md` (sections 7/8 - pipeline now real through
  `combine_signals`, real hybrid-scoring design writeup, real evaluation
  numbers and both honest limitations), `API.md` (both applications
  endpoints' real request/response shapes updated with
  `ruleScore`/`triggeredRules`/`finalRiskScore`/`CLEARED`/
  `FRAUD_DETECTED`), `DATA_FLOW.md` (new "Step 4 update — Phase 15"
  section, pipeline diagram, "What still goes here" list now Phase 16+),
  `SECURITY.md` (new Phase 15 consequence note - rule engine/combination
  never see raw PII either, no new secret/key surface), `TESTING.md`
  (new counts, new "Phase 15: fraud detection tests" section,
  planned-surfaces table rows marked done), `ML_MODEL.md` (status header,
  full new Phase 15 section with the rule engine/combination design and
  both real evaluation findings written up honestly), `central-fraud-
  intelligence/README.md` (status header, new Phase 15 section, "What
  this phase does NOT do" list, layout table, testing section/counts, new
  "Rule engine + hybrid scoring (Phase 15)" usage section). No new
  `TROUBLESHOOTING.md` entry - this phase hit no new environment/egress
  blocker.
- All test/build artifacts deleted or left properly gitignored after
  verification; not committed.

### Phase 16 additions

- **`central-fraud-intelligence/src/explainability/explain.py` is new** -
  `generate_explanation(event)`: a pure, stateless function over the four
  score fields Phase 15 already computes (`status`, `fraudScore`,
  `ruleScore`, `triggeredRules`, `finalRiskScore`) - never touches
  Postgres, the graph, or the GNN checkpoint, never raises. Design
  decision made and documented in the module's own docstring: computed
  at READ TIME, called fresh from both POST and GET response building,
  never persisted as a new column - three reasons: (1) a pure function
  has nothing to fail against that would need retry-on-redelivery
  treatment; (2) computing it fresh means historical events benefit
  immediately if wording/logic improves later, instead of being frozen
  with whatever text existed at ingestion time; (3) it avoids persisting
  a second, derived copy of information already fully stored. Unlike
  `run_rule_engine`/`combine_signals` (Phase 15), it does not add a new
  status value and is not gated on `status` at all - it describes
  whatever stage an event is at, including "not decided yet" for an
  in-progress one.
- Returns `{"summary": <human-readable str>, "technical": {...}}`.
  `summary` names every `triggered_rules` signal in plain language (with
  real minute/hour timing derived from each signal's `closestHoursApart`
  detail) plus the GNN's independent score. `technical` reports each
  component's *actual* weighted contribution to `finalRiskScore` -
  honoring `combine_signals()`'s real documented `None`-GNN fallback
  (`effectiveWeight: 1.0` for rules when no GNN score exists, not the
  static `RULE_WEIGHT` constant) rather than just echoing configured
  weights - plus a `primaryDriver` classification (`gnn_only` /
  `rules_only` / `both`) and a `caveats` list.
- **Both of Phase 15's open findings addressed directly, within this
  phase's narrower "explain honestly, not fix" scope** (decided and
  documented as a deliberate scope boundary, not silently ignored - see
  the TODO checklist item this resolves): every `CLEARED` explanation
  carries a caveat stating the decision reflects only information
  available at processing time and is not automatically revisited if
  related applications arrive later (finding 1 - the first-arrival blind
  spot); every `FRAUD_DETECTED` decision with an empty `triggeredRules`
  (driven entirely by the GNN, zero corroborating rule signal - the exact
  shape of finding 2) gets an explicit caveat naming the known
  legitimate-repeat-applicant false-positive limitation and recommending
  manual review, rather than being presented with the same unqualified
  confidence as a rule-corroborated decision. Neither finding is
  remediated in this phase (no re-scan/re-scoring trigger was built,
  and the GNN false-positive class was not investigated further) -
  explanation, not remediation, was this phase's scope, and that
  boundary is stated explicitly rather than left ambiguous.
- **Wired into the live request path** -
  `src/api/applications.py`'s `POST /api/v1/applications/process` and
  `GET /api/v1/applications/:id` both now also return `explanation`,
  computed right before the response is returned. Wrapped in a
  try/except purely as defense against an unexpected formatting
  exception (consistent with every other step in this module), even
  though this function has no real external dependency that can fail.
- **Verified for real**: new `tests/test_explainability.py` (15 tests) -
  `generate_explanation()` tested directly (no HTTP/database needed for
  most of them): every in-progress status and a missing/empty status
  report "not decided yet"; `CLEARED` with/without a weak signal;
  `FRAUD_DETECTED` with both rules and GNN contributing (weighted
  contributions cross-checked against `scoring.py`'s real `GNN_WEIGHT`/
  `RULE_WEIGHT` constants, imported not hand-copied); `FRAUD_DETECTED`
  driven solely by the GNN (known-limitation caveat present) vs. solely
  by rules (technical breakdown reflects the real `None`-GNN fallback);
  document-hash phrasing; multiple triggered rules; an unrecognized
  status (the documented-but-never-actually-set `FAILED` value, confirmed
  by grep to never be set by any code in this project) falls back to a
  generic, honest message; never raises on malformed/empty input; plus
  two live-wiring tests proving a real `explanation` is present on both
  the POST and GET responses. All 15 tests passed on the first real run.
- Test counts: central-fraud-intelligence **132/132** (117 prior + 15 new
  `test_explainability.py`; loan platforms unchanged at 222). **354 tests
  total across the project.**
- `src/config.py` `VERSION` bumped `0.13.0-phase15` → `0.14.0-phase16` -
  both response endpoints genuinely changed shape (new `explanation`
  field), the same "bump only when request-handling behavior actually
  changes" discipline as every prior bump.
- Updated `ARCHITECTURE.md` (section 7 - `generate_explanation` now real
  but architecturally different from every prior pipeline step, a new
  "Explainability" subsection under section 8 covering the design and how
  both Phase 15 findings are addressed), `API.md` (both applications
  endpoints' real request/response shapes updated with `explanation`),
  `DATA_FLOW.md` (new "Step 4 update — Phase 16" section explaining the
  read-time-not-persisted distinction, pipeline diagram, "What still goes
  here" list now Phase 17+), `SECURITY.md` (new Phase 16 consequence note
  - no new PII exposure, no new storage/key surface), `TESTING.md` (new
  counts, new "Phase 16: explainability tests" section, planned-surfaces
  table row marked done), `ML_MODEL.md` (status header, new short Phase 16
  section explaining there's nothing new to measure and how both Phase 15
  findings are referenced concretely), `central-fraud-intelligence/
  README.md` (status header, new Phase 16 section, "What this phase does
  NOT do" list trimmed of the now-closed explainability gap, layout table,
  testing section/counts, new "Explanations (Phase 16)" usage section with
  a real example response). No new `TROUBLESHOOTING.md` entry - this phase
  hit no new environment/egress blocker.
- All test/build artifacts deleted or left properly gitignored after
  verification; not committed.

### Phase 17 additions

- **`central-fraud-intelligence/src/incidents/incidents.py`
  (`build_incident_record()`, `derive_affected_applications()`,
  `affected_platform_ids()` - pure, no I/O) and `src/incidents/
  repository.py` (`PostgresIncidentRepository`, a new `fraud_incidents`
  Postgres table, same lazy-connect discipline as every other repository
  here) are real and wired into the live request path: once an event's
  `status` reaches `FRAUD_DETECTED` (right after Phase 15's rule engine
  step), `src/api/applications.py`'s POST handler assembles and persists
  a real incident record. Two new admin-only read endpoints exist too:
  `GET /api/v1/incidents`, `GET /api/v1/incidents/:incidentId`
  (`src/api/incidents.py`) - both already named in `API.md`'s Planned
  table since Phase 0, now real.
- **What creates an incident, decided and stated explicitly**: only
  `status == "FRAUD_DETECTED"` - a CLEARED decision just under the 0.5
  threshold was considered and deliberately rejected (no principled
  secondary threshold this project's own evaluation data would justify;
  Phase 16's explanation already surfaces a borderline CLEARED decision
  honestly without a persisted record implying more certainty than the
  score supports).
- **Incidents can genuinely span multiple applications/platforms** - the
  concrete implementation of `ARCHITECTURE.md` section 9's "affected
  platforms {A, C}" framing, written back in Phase 0 before any of this
  existed. `derive_affected_applications()` reads every Application node
  already present in the same bounded subgraph the rule engine/GNN step
  already scored for this event (`GraphStore.get_relevant_subgraph()`,
  Phase 11) - no new traversal, and the identical hop/max_nodes horizon
  as the score that triggered the incident. This makes an incident a
  genuine point-in-time snapshot, proven directly (not just asserted) by
  a dedicated test: two applications on two different platforms sharing
  one phone token minutes apart - the *first* application's own incident
  (created before the second existed in the graph) lists only itself;
  the *second* application's incident correctly lists both.
- **Decided the interaction with Phase 15's finding 2 explicitly**: a
  `FRAUD_DETECTED` decision driven entirely by the GNN (zero triggered
  rules - finding 2's exact shape, legitimate repeat applicants scored
  near-1.0) still creates a full incident, not a suppressed or
  downgraded one. Silently declining to persist an incident for that
  decision class would hide a known limitation rather than surface it -
  the opposite of Phase 16's own discipline. The incident carries
  forward the same `primaryDriver: "gnn_only"` classification and
  manual-review caveat Phase 16 already generates (a snapshot of
  `generate_explanation()`'s output at creation time), so a future
  reviewer can triage `gnn_only` incidents more closely without this
  project inventing a second, lighter-weight incident type.
- **Idempotent the same way event ingestion is**: `incident_id` is
  deterministic (`INC-<triggering event's eventId>`), and
  `triggering_event_id` carries a real Postgres UNIQUE constraint with
  the same `ON CONFLICT ... DO NOTHING RETURNING *` pattern
  `src/ingestion/repository.py`'s `create_or_get()` uses for events - a
  redelivered `FRAUD_DETECTED` event never creates a second incident,
  verified directly against the real table's row count, not just the
  response.
- **Scope decision, stated not left ambiguous**: `review_status`
  defaults to `"OPEN"`, and the lifecycle this package's own
  `__init__.py` docstring already named before any of it was real
  (`UNDER_REVIEW`/`CONFIRMED`/`DISMISSED`/`ESCALATED`) is recorded as the
  intended state machine, but no endpoint transitions an incident
  between those states this phase - left as a reasonable candidate for
  the Phase 19 admin dashboard.
- **Best-effort, retry-safe, same posture as every prior pipeline step**:
  gated purely on stored `status == "FRAUD_DETECTED"`, wrapped in a
  broad try/except - a failure (e.g. Postgres briefly unreachable) never
  unmakes the already-final `FRAUD_DETECTED` decision, it just leaves
  that event without an incident yet, retried by the next redelivery.
- **Verified for real**: new `tests/test_incidents.py` (17 tests) - pure
  assembly functions tested directly with hand-built subgraphs (every
  Application node returned sorted/deduplicated, a defensive fallback
  when the root is missing, deterministic `incident_id`); real live
  wiring against real Postgres/graph with a stubbed GNN scorer (same
  pattern `test_explainability.py` established) - a GNN-only
  `FRAUD_DETECTED` decision creates a real incident with the manual-
  review caveat; `CLEARED` creates none; the genuine multi-platform
  case described above; redelivery idempotency checked against the real
  table; `incident` correctly appears/is `null` in
  `GET /api/v1/applications/:id`; both new endpoints 401 without an
  admin token, 404 for an unknown id, and return exactly what a real
  POST persisted. All 17 tests passed on the first real run.
- Test counts: central-fraud-intelligence **149/149** (132 prior + 17 new
  `test_incidents.py`; loan platforms re-verified for real this session,
  unchanged at 222: a 54/54, b 55/55, c 57/57, d 56/56). **371 tests
  total across the project.**
- `src/config.py` `VERSION` bumped `0.14.0-phase16` → `0.15.0-phase17` -
  both applications endpoints genuinely gained a new `incident` field and
  two new endpoints exist, the same "bump only when request-handling
  behavior actually changes" discipline as every prior bump.
- Updated `ARCHITECTURE.md` (section 7 - `create_incident_if_required`
  now real; new "Fraud incidents" subsection under section 8 covering
  every design decision above; section 9 updated - the "affected
  platforms {A, C}" framing is no longer hypothetical), `API.md` (both
  applications endpoints' real shapes updated with `incident`; the two
  incidents rows moved from Planned to Live with full real shapes),
  `DATA_FLOW.md` (new "Step 4 update — Phase 17" section, "What still
  goes here" list now Phase 18+, pipeline diagram updated), `SECURITY.md`
  (new Phase 17 consequence note - a genuinely new storage surface, but
  no new PII exposure, same admin-auth gating as the existing GET
  endpoint), `TESTING.md` (new counts, new "Phase 17: fraud incident
  tests" section, planned-surfaces table row marked done), `ML_MODEL.md`
  (status header, new short Phase 17 section on the finding-2 handling
  decision), `central-fraud-intelligence/README.md` (status header, new
  Phase 17 section with a real example response, "What this phase does
  NOT do" list updated, layout table, testing section/counts, new "Fraud
  incidents (Phase 17)" usage section). No new `TROUBLESHOOTING.md`
  entry - this phase hit no new environment/egress blocker.
- All test/build artifacts deleted or left properly gitignored after
  verification; not committed.

### Phase 18 additions

- **`central-fraud-intelligence/src/notifications/notifications.py`**
  (`build_notification_payload()` - pure, no I/O) and **`src/
  notifications/repository.py`** (`PostgresNotificationRepository`, a new
  `fraud_notifications` Postgres table, same lazy-connect discipline as
  every other repository here) are real and wired into the live request
  path: whenever an incident was just created (right after Phase 17's
  incident-creation step), `src/api/applications.py`'s POST handler
  creates one real, persisted notification row per platform in that
  incident's own `affected_platforms` - one row per (incident, platform)
  pair, not one row per incident with an embedded list, because delivery
  status is genuinely per-recipient (the same reasoning `OutboxEvent`
  already applies at the event level, one level up).
- **Delivery is a new, independent background process** -
  **`src/notifications/worker.py`**'s `NotificationDeliveryWorker`,
  started at app startup and stopped at shutdown (`src/main.py`'s
  lifespan, the first genuinely recurring background task any phase has
  added here - every prior startup step was a one-shot best-effort
  warm-up). Direct Python/asyncio port of every loan platform's own
  `deliveryWorker.js` (Phase 7): same `deliver_one`/`deliver_pending`/
  `start`/`stop` shape, same capped-exponential-backoff formula
  (`min(30s, 1s * 2^(attempts-1))`, FAILED after 8 attempts), same
  `X-Idempotency-Key` header (here, the notification's own deterministic
  id), just running in the opposite direction (central → platform
  instead of platform → central) and using an injectable
  `httpx.AsyncClient` where the JS worker injects `fetchImpl`.
- **Authentication reuses each platform's existing shared secret,
  deliberately** - `CENTRAL_ACCEPTS_API_KEY_{A,B,C,D}` (central-side
  name)/`CENTRAL_FRAUD_API_KEY` (platform-side name), the exact value
  Phase 7/8 already established for the opposite direction, now also
  checked in reverse. A second key-management surface was considered
  and rejected as unnecessary provisioning overhead for a connection
  that already has a working shared secret between the same two
  parties.
- **The receiving side, built on loan-platform-a first (this project's
  established "build fully on one platform, then propagate" pattern),
  then propagated to b/c/d with identifier substitution and full
  re-verification**: a new `POST /api/v1/fraud-notifications/receive`
  route (`routes/fraudNotifications.routes.js`) checks `X-API-Key`
  against the platform's own `CENTRAL_FRAUD_API_KEY` (a single equality
  check - exactly one legitimate sender, unlike central's own four-
  platform `resolve_platform_from_api_key()`), takes `alertId` directly
  from the `X-Idempotency-Key` header (the payload carries no
  notification id of its own), and persists into a new `fraud_alerts`
  Mongo collection (`models/FraudAlert.js`) via a new dual Mongo-backed/
  in-memory `fraudAlertRepository.js`, idempotent on `alertId` (upsert
  semantics - a redelivery after a lost 200 response never creates a
  duplicate). A new admin-only `GET /api/v1/admin/fraud-alerts` route
  lists what's been received, reusing the existing `requireAuth`/
  `requireRole("admin")` middleware. Deliberately **no** reviewed/
  acknowledged field on `FraudAlert` - no review lifecycle was named for
  platform-side alerts, so adding one now would be premature
  scaffolding for an undesigned feature.
- **Two new central-side admin-only read endpoints**: `GET /api/v1/
  fraud-notifications`, `GET /api/v1/fraud-notifications/:id`
  (`src/api/notifications.py`) - already named in `API.md`'s Planned
  table since Phase 0, now real. Deliberately **not** built this phase,
  the same "creation and read only, no review workflow yet" scope
  boundary Phase 17 drew for incidents: `POST /api/v1/fraud-
  notifications` (redundant - creation is automatic) and
  `POST /api/v1/fraud-notifications/:id/review` (naturally pairs with
  the incident review workflow already deferred to Phase 19).
- **Best-effort, retry-safe, same posture as every prior pipeline step**:
  notification *creation* is gated purely on `incident_row is not None`,
  wrapped in a broad try/except per platform (one platform's creation
  failing never blocks another's) - a failure never unmakes the
  already-final `FRAUD_DETECTED` decision or the incident it produced,
  it just leaves that platform without a queued notification yet,
  retried by the next redelivery. Delivery failures are handled the
  same way the JS outbox worker always has - recorded, backed off,
  eventually FAILED, never crashing the poller.
- **Verified for real**: new `tests/test_notifications.py` (30 tests) -
  pure payload assembly; real `PostgresNotificationRepository` behavior
  against the test database (idempotent creation, `list_deliverable`'s
  due-filter, backoff/FAILED-after-max-attempts); `NotificationDelivery
  Worker` against a real `httpx.AsyncClient` built on
  `httpx.MockTransport` (a real client, fake transport, not a hand-
  rolled double) covering SENT/RETRY/FAILED outcomes, missing-platform-
  config, and a missing-incident defensive case; live wiring proving
  `notifications` appear one-per-affected-platform for both single- and
  multi-platform incidents, empty when CLEARED, never duplicated on
  redelivery; both new endpoints' auth/404 behavior. All 30 tests passed
  on the first real run, alongside the full existing 149-test suite (no
  regressions). Each loan platform's new `tests/fraudAlerts.test.js` (14
  tests, identical across all four after propagation) covering
  X-API-Key/X-Idempotency-Key/required-field validation, successful
  persistence, redelivery idempotency, standalone-mode (no repository
  wired), and the new admin read route - verified for real on all four
  platforms independently, not assumed from loan-platform-a alone.
- Test counts: central-fraud-intelligence **179/179** (149 prior + 30 new
  `test_notifications.py`); all four loan platforms **278** total (a
  68/68, b 69/69, c 71/71, d 70/70 - each gaining the same 14-test
  `fraudAlerts.test.js`, all re-run for real this session, not assumed
  unchanged). **457 tests total across the project.**
- `src/config.py` `VERSION` bumped `0.15.0-phase17` → `0.16.0-phase18` -
  both applications endpoints genuinely gained a new `notifications`
  field and two new endpoints exist, the same "bump only when request-
  handling behavior actually changes" discipline as every prior bump.
- `docker-compose.yml` and root `.env.example` both gained
  `PLATFORM_{A,B,C,D}_BASE_URL` (optional - already match `src/
  config.py`'s own defaults, the in-network DNS names on
  `net-central-bus`) for the central service; each loan platform's own
  `backend/.env.example` gained a comment documenting `CENTRAL_FRAUD_
  API_KEY`'s new bidirectional reuse. No new secrets were introduced
  anywhere.
- Updated `ARCHITECTURE.md` (section 7 - `identify_affected_platforms`/
  `send_notifications` now real, event status progression clarified -
  no `NOTIFICATION_SENT` event status, delivery state lives on each
  notification row instead; new "Fraud notifications" subsection under
  section 8; section 9 fully resolved from hypothetical to real),
  `API.md` (both applications endpoints' real shapes updated with
  `notifications`; the loan-platform receive/list rows and the two
  central read rows moved from Planned to Live with full real shapes;
  the two deliberately-not-built rows documented with rationale instead
  of silently dropped), `DATA_FLOW.md` (new "Step 4 update — Phase 18"
  section covering both the central creation/delivery side and the
  platform receiving side, "What still goes here" list now Phase 19+,
  pipeline diagram extended with the background worker loop and the
  platform-side receive flow), `SECURITY.md` (new Phase 18 consequence
  note - a new storage surface on both sides and a new outbound network
  hop, but no new secret or PII-exposure surface, full reasoning for the
  bidirectional key reuse), `TESTING.md` (new counts, new "Phase 18:
  fraud notification tests" section, planned-surfaces table row marked
  done), `central-fraud-intelligence/README.md` (status header, new
  Phase 18 section with a real example response, "What this phase does
  NOT do" list updated, layout table, testing section/counts), and all
  four loan platforms' own `README.md` (status header, test counts, new
  "Fraud notifications (Phase 18)" usage section, API summary). No
  `ML_MODEL.md` changes needed - notifications don't touch scoring/
  evaluation. No new `TROUBLESHOOTING.md` entry - this phase hit no new
  environment/egress blocker.
- All test/build artifacts deleted or left properly gitignored after
  verification; not committed.

### Phase 19 additions

- **Incident review workflow, real**: `POST /api/v1/incidents/:id/review`
  (`src/api/incidents.py`) transitions `review_status`, backed by a new
  `src/incidents/repository.py` method `update_review_status()` (three-
  valued return - `(row, None)` / `(None, "not_found")` /
  `(None, "invalid_status")` / `(None, "invalid_transition")` - the API
  layer maps each to 404/400/409). Valid transitions
  (`ALLOWED_REVIEW_TRANSITIONS` in `src/incidents/incidents.py`): `OPEN`
  -> any of `UNDER_REVIEW`/`CONFIRMED`/`DISMISSED`/`ESCALATED`;
  `UNDER_REVIEW` -> `CONFIRMED`/`DISMISSED`/`ESCALATED` (not back to
  `OPEN`); the three terminal states accept nothing further. A new
  `review_history JSONB` column (`ALTER TABLE ... ADD COLUMN IF NOT
  EXISTS`, the same schema-evolution pattern `src/ingestion/repository.py`
  established in Phase 9) accumulates `{action, by, note, at}` events per
  transition - this exact shape mirrors `loan-platform-a/backend/src/
  models/Application.js`'s `ReviewEventSchema` field-for-field, found by
  reading that file for precedent rather than inventing a new shape.
- **Notification acknowledgement, real, deliberately simpler than
  incident review**: `POST /api/v1/fraud-notifications/:id/review`
  (`src/api/notifications.py`) - a one-way not-acknowledged ->
  acknowledged boolean (new `acknowledged`/`acknowledged_by`/
  `acknowledged_note`/`acknowledged_at` columns on `fraud_notifications`,
  same `ALTER TABLE ADD COLUMN IF NOT EXISTS` pattern), 409 on a second
  attempt, orthogonal to delivery `status`. The identical endpoint now
  exists on each loan platform's own `FraudAlert`
  (`POST /api/v1/admin/fraud-alerts/:alertId/review`) - built fully on
  loan-platform-a first (this project's established pattern), then
  propagated to b/c/d with full independent re-verification, not
  assumed. Both central's `fraud_notifications.acknowledged` and each
  platform's own `fraud_alerts.acknowledged` deliberately do NOT reuse
  the incident's `review_history` shape - a notification doesn't have a
  lifecycle to steer, only one fact to record, a genuinely different
  design decision, not an inconsistency (see `src/notifications/
  notifications.py`'s module docstring point 4).
- **Graph explorer, no new traversal logic**: `GET /api/v1/graph/
  application/:id` and `GET /api/v1/graph/entity/:id` (new `src/api/
  graph.py`) are thin, admin-authenticated wrappers over
  `GraphStore.get_relevant_subgraph()` - real and already exercised
  since Phase 11, just not previously reachable over HTTP.
  `?hops=`/`?maxNodes=` are clamped at the API layer (`[1,10]`/
  `[1,2000]`) so one request can't request an unbounded neighborhood;
  `GraphStore`'s own `max_nodes` already bounds the traversal itself.
- **Metrics, operational counts only**: `GET /api/v1/metrics` (new
  `src/api/metrics.py`) - explicitly NOT performance/latency data
  (stays Phase 23's concern, stated in that module's own docstring to
  resolve the ambiguity `API.md`'s Planned table left open since Phase
  0). New repository methods: `PostgresEventRepository.count_by_status()`,
  `PostgresIncidentRepository.count_by_review_status()`,
  `PostgresNotificationRepository.count_by_status()`/
  `count_acknowledged()` - each one real `GROUP BY`/`COUNT` query, no
  caching, no pre-aggregation (this project's data scale doesn't need
  it, the same reasoning `src/graph/graph_store.py` already gives for
  never needing pruning).
- **The central app's first-ever UI**: a server-rendered Jinja2 admin
  dashboard (new `src/web/` package, everything under `/dashboard/*`) -
  decided over a React/Vite SPA for three stated reasons: this service
  has been 100% Python/FastAPI through Phase 18 (no Node build stage in
  the Dockerfile at all); a Jinja2 page is testable directly with
  FastAPI's own `TestClient` (real HTML in, real HTML out), the same
  "test everything for real" discipline every other phase here has kept;
  and `requirements.txt` had already flagged this exact fork as the open
  Phase 19 question. Auth is cookie-based (`central_admin_token`,
  HttpOnly, SameSite=Lax) but not a second login system - it carries the
  identical JWT `POST /api/v1/admin/login` already issues. Pages: login,
  incidents list/detail (with a review form offering only the valid next
  transitions), notifications list (with an acknowledge button),
  graph explorer (node/edge tables, deliberately not a canvas/SVG
  visualization - a stated scope limit). Zero client-side JavaScript -
  every action is a plain HTML `<form>` POST, and every protected route
  redirects (303) to `/dashboard/login` rather than 401ing, the right
  behavior for a browsed page.
- **Verified for real**: five new central test files - `test_graph_api.py`
  (13 tests), `test_incidents_review.py` (30 tests, pure-logic +
  live HTTP including a full real `OPEN -> UNDER_REVIEW -> CONFIRMED`
  sequence with accumulating `reviewHistory`), `test_notifications_
  review.py` (9 tests), `test_dashboard.py` (18 tests, real HTML via
  `TestClient` - actual requested pages, actual assertions on their
  content), `test_metrics.py` (6 tests) - all passed on the first real
  run except one intentional fix-before-commit typo (event repository
  test keys corrected from snake_case to the real camelCase
  `create_or_get()` contract, caught by the test itself failing, not
  discovered later). Each loan platform's `fraudAlerts.test.js` gained 5
  acknowledgement tests, verified independently on all four platforms
  (not assumed from loan-platform-a alone) - one pre-existing, unrelated
  `tokenization.test.js` flaky test observed on loan-platform-b (a
  probabilistic tamper-test edge case, confirmed flaky and unrelated to
  this phase by an immediate clean re-run of the full suite, 74/74).
- Test counts: central-fraud-intelligence **238/238** (179 prior + 76 new
  across 5 files); all four loan platforms **298** total (a 73/73, b
  74/74, c 76/76, d 75/75 - each gaining the same 5-test acknowledgement
  addition to `fraudAlerts.test.js`, all re-run for real this session).
  **536 tests total across the project.**
- `src/config.py` `VERSION` bumped `0.16.0-phase18` → `0.17.0-phase19` -
  two new write endpoints and three new read endpoints exist, plus an
  entirely new HTML surface, the same "bump only when request-handling
  behavior actually changes" discipline as every prior bump.
- `requirements.txt` gained `jinja2==3.1.6` and `python-multipart==0.0.26`
  (the latter required at runtime for FastAPI's `Form()` parsing, used by
  the dashboard's login form) - both already present/importable in this
  sandbox, no install friction. No new environment variables anywhere -
  the dashboard reuses `CENTRAL_JWT_SECRET`.
- Updated `ARCHITECTURE.md` (incident review-transition rules and the
  audit-trail design fully written into section 8's "Fraud incidents"
  subsection; the "Fraud notifications" subsection's acknowledgement
  design; a new "Graph explorer, metrics, and the admin dashboard"
  subsection; section 9's closing paragraph resolved from hypothetical
  to real), `API.md` (four new Live rows for central, one new Live row
  per loan platform, a new "Dashboard" section documenting every
  `/dashboard/*` route, the old "Planned" table removed since nothing
  remains in it), `DATA_FLOW.md` (new "Step 4 update — Phase 19" section,
  "What still goes here" renamed to Phase 20+), `SECURITY.md` (new Phase
  19 consequence note - cookie-auth transport reusing the same JWT/
  secret, no new PII storage, the stated CSRF gap added to Planned
  controls), `TESTING.md` (new counts, new "Phase 19" test section,
  Planned-surfaces table rows marked done), `central-fraud-
  intelligence/README.md` (status header, layout table, test counts, new
  "Review workflow, graph explorer, metrics, and admin dashboard"
  section, stale "not built" bullet struck through), and all four loan
  platforms' own `README.md` (status header, test counts, acknowledgement
  usage section, API summary line). No `ML_MODEL.md` changes needed -
  this phase doesn't touch scoring/evaluation.
- All test/build artifacts deleted or left properly gitignored after
  verification; not committed.

### Still open from Phase 1 (unchanged)

Docker Hub / MongoDB binary downloads remain blocked in this build sandbox
(egress policy `403`, confirmed, not transient) — `docker compose up
--build` itself still needs to be run by the user in an unrestricted
environment. See `TROUBLESHOOTING.md`.

### Known issues from Phase 15, still open (explained honestly in Phase 16, not yet remediated)

Real-time single-pass scoring has a structural blind spot: a fraud ring's
first-arriving member cannot be detected at the moment it's scored (see
Phase 15 additions above, finding 1) - correcting this needs a re-scan/
re-scoring trigger once later ring members arrive, which does not exist
yet. A second, distinct, unresolved GNN limitation also opened in Phase
15: legitimate repeat applicants are sometimes scored near-1.0 by the GNN
itself (finding 2 above) - not yet explained further or mitigated at the
model level. Phase 16 addressed both **honestly, in the explanation
layer** (every `CLEARED` decision's caveats name finding 1; every
GNN-only `FRAUD_DETECTED` decision's caveats name finding 2 and recommend
manual review) but did not remediate either - no re-scan trigger was
built, and the GNN was not retrained or investigated further. Phase 17
made a deliberate decision about how this interacts with incident
creation (a GNN-only `FRAUD_DETECTED` decision still opens a full
incident, carrying the same caveat forward - see Phase 17 additions
above) but still did not remediate either underlying limitation. Both
remain real, open, reported honestly, not hidden - see `ML_MODEL.md`'s
Phase 15, Phase 16, and Phase 17 sections. Phase 18 (notifications) and
Phase 19 (review workflow/dashboard) did not touch scoring/detection at
all and do not change this.

## Next action

Begin **PHASE 20**, per `TODO.md`'s Phase 20 checklist. Only once the
user explicitly sends `CONTINUE FROM CHECKPOINT`.

## Session log

- 2026-09-17 — Phase 0: repo inspected (empty), reference PDF reviewed,
  architecture + tech stack decided and documented, checkpoint files
  created, git repo initialized. Stopped per instruction, awaited
  `CONTINUE FROM CHECKPOINT`.
- 2026-09-17 — Phase 1: full project skeleton, docker-compose.yml (10
  services, isolated networks), .env.example, root documentation set, all
  four loan-platform skeletons (backend+frontend+Dockerfile) generated via
  a reusable script, central-fraud-intelligence FastAPI skeleton with all
  13 module packages stubbed. Verified real code paths directly (pip/npm
  install + run + curl); Docker Hub pulls are blocked by this sandbox's
  egress policy, so `docker compose up --build` itself is recorded as
  NOT VERIFIED pending the user running it. Committed, checkpoint updated.
- 2026-09-17 — Phase 2: loan-platform-a (QuickFund) built out fully — real
  MongoDB models, dual Mongo/in-memory repositories, services, JWT+bcrypt
  auth, admin status-transition rules, auto-generated repayment schedules,
  and a real React frontend wired to every endpoint. 34/34 tests passing.
  Found and fixed a boot-order bug (health endpoint blocked on MongoDB
  connection) via direct testing. Updated API.md, TESTING.md, SECURITY.md,
  loan-platform-a/README.md. Committed, checkpoint updated.
- 2026-09-17 — Phase 3: loan-platform-b (TrustLend) built from
  loan-platform-a's architecture, differentiated with required
  institutionName/courseName fields (education-loan-specific validation,
  not just branding). 35/35 tests passing. Updated API.md, TESTING.md,
  loan-platform-b/README.md. Committed, checkpoint updated.
- 2026-09-17 — Phase 4: loan-platform-c (SecureCredit) built from
  loan-platform-a's architecture, differentiated with micro-loan amount/
  tenure ceilings (400 if exceeded, not just branding). Found and fixed two
  bugs from the copy+sed process: a missing `brand.js` (loan-platform-c
  only) and a mis-substituted project name in the footer text
  ("TrustLend-X"/"SecureCredit-X" instead of "QuickFund-X", present in both
  loan-platform-b and loan-platform-c — fixed in both, loan-platform-b
  re-tested at 35/35 to confirm no regression). 37/37 tests passing for
  loan-platform-c. Updated API.md, TESTING.md, loan-platform-c/README.md.
  Committed, checkpoint updated.
- 2026-09-17 — Phase 5: loan-platform-d (FinBridge) built from
  loan-platform-a's architecture, differentiated with required
  merchantName/productCategory fields (productCategory restricted to a
  fixed enum — a categorical validation, not just branding). Found and
  fixed the same class of copy+sed bugs already known from Phase 4 (footer
  project-name over-match, stray platform-A references in comments) before
  running any tests. 36/36 tests passing. All four loan platforms now have
  real, differentiated, fully tested functionality. Updated API.md,
  TESTING.md, loan-platform-d/README.md. Committed, checkpoint updated.
- 2026-09-17 — Phase 6: independent seed-data generator (`scripts/seed.js`)
  added to all four loan platforms, exercising the real auth/application/
  admin services (not direct DB writes) to create 1 admin + 5 users + 8
  applications across all statuses per platform, with platform-appropriate
  loan data. Verified for real via a new `seed.test.js` in each platform
  (run against the in-memory repositories, asserting both seed()'s return
  value and the repositories' actual contents). Found and fixed a third
  instance of the copy+sed identifier-leak bug class, this time in
  loan-platform-b/c's `createAdmin.js` usage docstrings (wrong Mongo
  port/database/example email left over from platform A). Test counts:
  a 35/35, b 36/36, c 38/38, d 37/37. Rewrote seed-data/seed-platform-*
  READMEs and each platform's own README with seeding instructions.
  Updated TESTING.md. Committed, checkpoint updated.
- 2026-09-17 — Phase 7: secure event/outbox mechanism added to all four
  loan platforms — `OutboxEvent` model, dual Mongo/in-memory
  `outboxRepository`, and a real polling `deliveryWorker` (retry with
  capped exponential backoff, idempotency key, API key), wired into
  `applicationService.submit()` and every `adminService` status transition.
  Verified for real against a local `http.createServer()` stub (not a
  mocked fetch) in a new `outbox.test.js` per platform: event creation,
  delivery + idempotent no-redelivery, retry/backoff to FAILED, and
  start()/stop() interval behavior. Found and fixed a fourth instance of
  the copy+sed identifier-leak bug class (`mongo-loan-a` left in B/C's
  `server.js` comments). Test counts: a 41/41, b 42/42, c 44/44, d 43/43
  (170 total). Updated API.md, DATA_FLOW.md, TESTING.md, SECURITY.md, and
  all four platforms' README.md. Committed, checkpoint updated.
- 2026-09-17 — Phase 8: central-fraud-intelligence gained a real
  application-event ingestion API (POST /api/v1/applications/process, GET
  /api/v1/applications/:id) - X-API-Key platform authentication, Pydantic
  event-shape validation, and idempotent persistence to a real PostgreSQL
  table (INSERT ... ON CONFLICT DO NOTHING). Discovered a real local
  Postgres 16 instance is available in this sandbox (unlike MongoDB),
  created dev + test databases, and ran the central app's 10-test suite
  against the real test database - no in-memory double needed, a stronger
  guarantee than any loan platform gets. Proactively avoided the Phase-2
  boot-order bug class (lazy DB connection; verified /health stays up with
  Postgres intentionally unreachable). Ran a true end-to-end smoke test:
  loan-platform-a's real deliveryWorker.js delivered a real event to a real
  running central-fraud-intelligence instance and it was marked SENT - the
  first real cross-service delivery in this project. Test counts:
  central-fraud-intelligence 10/10 (new), loan platforms unchanged at 170
  (180 total). Updated API.md, DATA_FLOW.md, TESTING.md, SECURITY.md,
  TROUBLESHOOTING.md, central-fraud-intelligence/README.md. Committed,
  checkpoint updated.
- 2026-09-17 — Phase 9: tokenization (HMAC-SHA256 + HKDF, per-purpose sub-
  keys) and AES-256-GCM transient encryption added to all four loan
  platforms' outbox payload construction — raw applicant PII no longer
  leaves a platform at all, verified via extended outbox.test.js
  assertions plus a new 13-test tokenization.test.js per platform (54
  Node-side crypto tests total across all four). Central app gained a
  Python mirror of the same primitives plus real cross-language parity
  tests (shells out to the actual Node implementation, 16/16 passing,
  including 4 genuine Node↔Python round-trip checks) and its own hand-
  rolled HS256 JWT admin auth (login endpoint, require_admin dependency
  now protecting GET /api/v1/applications/:id, scripts/create_admin.py;
  10/10 new tests). Found and fixed a missing `import psycopg` in
  src/api/admin.py. Full combined central-app suite run once
  (test_ingestion + test_admin_auth + test_tokenization) to rule out
  fixture interaction issues: 38/38 passing. Closed a real Docker/
  production wiring gap: CENTRAL_HMAC_MASTER_KEY/CENTRAL_AES_KEY were only
  in central-fraud-intelligence's docker-compose.yml environment block,
  not the four loan platforms' (added to all four; central app's own
  requirements.txt also updated to list cryptography/bcrypt, previously
  installed but undeclared). Test counts: a 54/54, b 55/55, c 57/57,
  d 56/56, central 38/38 (260 total). Updated API.md, DATA_FLOW.md,
  SECURITY.md, TESTING.md, all five services' README.md, .env.example, and
  corrected ARCHITECTURE.md's stale "PHASE 0" status line. Committed,
  checkpoint updated.
- 2026-09-17 — Phase 10: persistent graph added to central-fraud-intelligence
  - real Postgres-backed graph_nodes/graph_edges tables
  (PostgresGraphRepository, MERGE-style ON CONFLICT DO UPDATE upserts) and
  a GraphStore that materializes them into an in-memory NetworkX
  MultiDiGraph, lazily rather than eagerly-at-startup (documented,
  deliberate deviation from ARCHITECTURE.md's original wording, to keep
  the established boot-order guarantee that /health never depends on
  Postgres). POST /api/v1/applications/process now applies each event to
  the graph (Borrower/Application/Platform/token nodes, entity resolution
  via shared tokens) and advances event status to GRAPH_UPDATED; retried
  on a genuine redelivery if the graph step previously failed, safely,
  since GraphStore's upserts are idempotent. Switched main.py off the
  deprecated @app.on_event("startup") to a lifespan context manager while
  wiring the best-effort graph warm-up. Verified for real: 5 new
  test_graph.py tests including a genuine entity-resolution proof (two
  applications, two platforms, one shared phone token -> one shared node)
  and a fresh-GraphStore-instance rehydration-matches-Postgres test; a
  direct boot-order check (unreachable DB, /health still 200, POST
  correctly 503) mirroring Phase 8's own verification discipline. Updated
  one existing test's status assertion (RECEIVED -> GRAPH_UPDATED) to
  match the new real behavior. Deliberately scoped out (documented, not
  silently skipped): a separate Loan node, BORROWER_HAS_MULTIPLE_APPLICATIONS
  as a stored edge, and DeviceToken/BankToken/IPToken nodes (no data
  source yet) - all explained in ARCHITECTURE.md section 6. Test counts:
  central 43/43 (38 prior + 5 new), loan platforms unchanged (222) - 265
  total. Added networkx to requirements.txt. Updated ARCHITECTURE.md,
  API.md, DATA_FLOW.md, SECURITY.md, TESTING.md,
  central-fraud-intelligence/README.md. Committed, checkpoint updated.
- 2026-09-17 — Phase 11: bounded k-hop subgraph extraction added to
  central-fraud-intelligence - GraphStore.get_relevant_subgraph(node_type,
  node_key, hops=4, max_nodes=500), the first thing in this project that
  reads the persistent graph back out (Phase 10 only wrote to it). Direction-
  agnostic BFS returning a plain JSON-serializable dict, not a networkx
  object; default hops=4 is the minimum that reaches a different
  application through one shared token; max_nodes=500 bounds a single
  high-degree hub token's blast radius, with deterministic sorted-order
  traversal so truncation is reproducible. Result is a true induced
  subgraph (all edges among visited nodes included, not just BFS-tree
  edges) - proven by a dedicated test. Two scope decisions made and
  documented rather than silently assumed: entity resolution stays exact-
  token-match (a direct, documented consequence of HMAC-SHA256's avalanche
  property from Phase 9's tokenization design, not a shortcut); no graph
  retention/pruning policy (this project's graph is expected to stay small,
  and pruning old shared identifiers would destroy the long-lived signal
  fraud-ring detection depends on). Not wired into any HTTP route or the
  live ingestion request path - no real consumer exists until Phase 14's
  GNN inference. This session's sandbox had reset since the Phase 10
  checkpoint (Postgres down, loan-platform node_modules missing) - both
  restored and every suite re-run for real before trusting any prior
  number. Test counts: central 50/50 (43 prior + 7 new test_subgraph.py),
  loan platforms unchanged and reverified (222) - 272 total. Boot-order
  guarantee reverified directly. Updated ARCHITECTURE.md, DATA_FLOW.md,
  SECURITY.md, TESTING.md, central-fraud-intelligence/README.md.
  Committed, checkpoint updated.
- 2026-09-17 — Phase 12: synthetic fraud scenario generator added -
  central-fraud-intelligence/datasets/scenario_generator.py
  (generate_catalog) + scripts/generate_scenarios.py (CLI entrypoint),
  producing seed-data/scenarios/scenarios.json: 51 scenarios, 129
  synthetic applications, covering 17 of DEMO_SCENARIOS.md's 20 catalog
  patterns. 3 patterns (#6 device, #7 bank token, #16 family shared
  device) deliberately not generated - no DeviceToken/BankToken graph
  node exists yet, same reason Phase 10 gave for not implementing those
  node types; generating fake device-token values would have been fake
  coverage. Every applicant tokenized via the real tokenize_applicant()
  (not hand-written token strings); every scenario's applications are
  real APPLICATION_SUBMITTED event payloads, no parallel training-data
  format invented. Patterns 1-3 (same identity, N platforms) and pattern
  19 (legitimate cross-platform applicant) deliberately share identical
  graph topology, differing only in submittedAt timing (fixed a bug here:
  an initial hand-rolled date-offset formatter produced invalid dates
  past 31 days and silently kept the same year-month string for a
  5-month offset - replaced with real datetime/timedelta arithmetic
  before this was caught by test assertions, not after). The full
  129-application dataset applied to a real GraphStore against a real
  Postgres database with zero errors before being committed. VERSION
  deliberately not bumped - no request-handling code changed. Test
  counts: central 64/64 (50 prior + 14 new test_scenario_generator.py),
  loan platforms unchanged (222) - 286 total. Updated ML_MODEL.md,
  DEMO_SCENARIOS.md, TESTING.md, central-fraud-intelligence/README.md,
  central-fraud-intelligence/requirements.txt, seed-data/scenarios/
  README.md. Committed, checkpoint updated.
- 2026-09-17 — Phase 13: GNN training. Added GraphStore.export_all()
  (whole-graph export, the training-time counterpart to Phase 11's
  bounded get_relevant_subgraph()), src/gnn/features.py (16-dim per-node
  feature engineering, including the submittedAt-derived timestamp
  feature that is the only way patterns 1-3 vs. 19 become distinguishable
  given their deliberately identical topology), src/gnn/model.py
  (HeteroGraphSAGE - pure PyTorch, no PyG/DGL, bidirectional relations
  via a learned reverse edge type for each of the 7 real ones,
  num_layers=4 tied to Phase 11's DEFAULT_SUBGRAPH_HOPS), and
  training/dataset.py + training/train.py + training/baseline.py.
  Environment fight: download.pytorch.org is blocked by this sandbox's
  egress policy (403, same class as the Phase 1 Docker Hub block) -
  fell back to plain `pip install torch`, which pulled PyPI's default
  CUDA-bundled wheel (torch==2.14.0+cu130, ~20 nvidia-* packages, no GPU
  present so torch.cuda.is_available() is False but it runs fine on CPU)
  - documented honestly in TROUBLESHOOTING.md and requirements.txt rather
  than silently accepted or hidden. Ran real training for real: 200
  epochs, seed 12345, against the real 51-scenario/129-application/
  590-node committed dataset (86 train / 43 eval, scenario-stratified
  split). Real measured eval results: accuracy 0.9767, precision 0.9730,
  recall 1.0, F1 0.9863, confusion matrix [[6,1],[0,36]]. Built a
  non-graph LogisticRegression baseline on the identical features/split
  (training/baseline.py) to isolate what the graph structure specifically
  buys: 0.9070 accuracy / 0.9474 F1 / [[3,4],[0,36]] - the GNN's entire
  advantage was on the GENUINE side (6/7 vs 3/7 correct). Inspected the
  one GNN false positive directly rather than just counting it: a
  "normal_genuine_baseline" scenario, not one of the deliberately hard
  hard_negative cases (all of which were classified correctly) - flagged
  honestly, along with near-zero final train loss (3.86e-05) as a real
  overfitting risk given only 86 training applications, with a concrete
  documented improvement plan rather than the number being taken at face
  value. VERSION deliberately not bumped - confirmed by grep that nothing
  under src/gnn/ or training/ is imported by src/main.py or any
  src/api/* router, so this phase touched zero request-handling code and
  the boot-order guarantee is untouched by construction. Test counts:
  central 87/87 (64 prior + 21 new test_gnn.py + 2 new in test_graph.py
  for export_all()), loan platforms unchanged (222) - 309 total. Updated
  ML_MODEL.md (major - items 1-3/5-7 filled in with real results),
  ARCHITECTURE.md, TESTING.md, central-fraud-intelligence/README.md,
  central-fraud-intelligence/requirements.txt, central-fraud-
  intelligence/Dockerfile, TROUBLESHOOTING.md, and added central-fraud-
  intelligence/models/README.md. Committed, checkpoint updated.
- 2026-09-17 — Phase 14: wired the Phase 13 trained GNN checkpoint into
  the live request path. New src/gnn/graph_tensors.py (shared tensor-
  construction, extracted from training/dataset.py so offline training
  and live inference can't silently diverge) and src/gnn/inference.py
  (GNNScorer.load()/score_subgraph(), never raises, best-effort). Loaded
  once at FastAPI startup (src/main.py); POST /api/v1/applications/process
  now runs real inference once an event reaches GRAPH_UPDATED, persisting
  a real fraudScore (new fraud_score column) and advancing status to
  ANALYZED - best-effort and retry-safe, same posture as the Phase 10
  graph-update step. Both applications endpoints now return fraudScore.
  Caught and fixed a status-regression bug in the graph/inference gating
  logic before it shipped (reasoning through the state machine, not via a
  failing test) and a test-determinism issue (client fixture now forces
  gnn_scorer to None by default, since the real checkpoint's presence on
  disk is environment-dependent). New tests/test_inference.py (14 tests,
  including a real-checkpoint-scores-a-real-subgraph test and a real
  boot-order test proving a corrupt checkpoint can't crash startup) - one
  test-writing bug (a monkeypatch lambda recursing into itself) caught
  and fixed by actually running the suite, not assumed correct. Fixed
  .dockerignore/Dockerfile to actually ship models/ into the Docker
  build context (honest caveat: the .pt file is git-ignored, so a fresh
  clone needs training/train.py run before a real build - not yet
  verified against Docker itself, still blocked). VERSION bumped
  0.11.0-phase11 → 0.12.0-phase14. Test counts: central 101/101 (87 prior
  + 14 new test_inference.py), loan platforms re-verified unchanged
  (222) - 323 total across the project. Updated ARCHITECTURE.md, API.md,
  DATA_FLOW.md, SECURITY.md, TESTING.md, ML_MODEL.md, central-fraud-
  intelligence/README.md, TROUBLESHOOTING.md. Committed, checkpoint
  updated.
- 2026-09-17 — Phase 15: built the deterministic rule engine
  (src/fraud_detection/rules.py's evaluate_rules() - shared phone/email/
  name/address tokens to distinct other identities + shared document
  reuse, application velocity folded into every signal as a time-
  proximity multiplier rather than a separate rule) and the hybrid
  combination function (src/fraud_detection/scoring.py's
  combine_signals() - 0.6*gnn_score + 0.4*rule_score, or rule_score alone
  when no GNN is loaded). Wired both into POST /api/v1/applications/
  process, gated on status ANALYZED, best-effort/retry-safe like every
  prior step; both endpoints now return ruleScore/triggeredRules/
  finalRiskScore. CLEARED/FRAUD_DETECTED are real status values for the
  first time since Phase 8. New tests/test_fraud_detection.py (15 tests,
  real graph fixtures) plus one new retry-safety test in
  test_inference.py; fixed 3 pre-existing test_inference.py assertions
  that broke because a scored event now progresses automatically past
  ANALYZED (expected new behavior, not a bug) and 3 float-precision/
  threshold-math test-writing bugs caught by actually running the suite.
  117/117 passing (339 total across the project). Ran a real two-mode
  evaluation (scripts/evaluate_fraud_detection.py) against all 129
  applications in the committed dataset with the real trained checkpoint:
  73.64% accuracy interleaved / 71.32% isolated (models/
  hybrid_scoring_eval.json, committed) - materially below Phase 13's
  97.67% offline number, investigated for real rather than hidden. Built
  the isolated-replay mode specifically to test, and disprove, a
  cross-scenario-timestamp-collision hypothesis; found instead that 100%
  of false negatives in both modes are exactly the first-submitted
  application within their own scenario - a genuine structural property
  of real-time single-pass scoring, not a dataset artifact - and
  corrected rules.py's own docstring to state the verified explanation.
  Also surfaced a second, distinct, unresolved GNN limitation (legitimate
  repeat applicants scored ~0.9999 by the GNN itself in 3 scenarios,
  reproduced identically in isolated mode) - reported as an open finding,
  not explained away. VERSION bumped 0.12.0-phase14 → 0.13.0-phase15.
  Updated ARCHITECTURE.md, API.md, DATA_FLOW.md, SECURITY.md, TESTING.md,
  ML_MODEL.md, central-fraud-intelligence/README.md. Committed, checkpoint
  updated.
- 2026-09-18 — Phase 16: built explanation generation
  (src/explainability/explain.py's generate_explanation() - a pure,
  stateless function over Phase 15's already-computed status/fraudScore/
  ruleScore/triggeredRules/finalRiskScore fields, never touching
  Postgres/the graph/the GNN checkpoint, never raising). Design decision
  made and documented: computed at READ TIME from both POST and GET
  responses, never persisted as a new column - a pure function has
  nothing to retry against, historical events benefit immediately from
  later wording improvements, and no redundant derived state is stored.
  Returns a human-readable summary (names every triggered signal in
  plain language with real timing) plus a technical breakdown (each
  component's actual weighted contribution, honoring combine_signals()'s
  real None-GNN fallback rather than just the static weights, plus a
  primaryDriver classification). Both of Phase 15's open findings are
  addressed directly, within an explicitly-scoped "explain honestly, not
  fix" boundary: every CLEARED decision's caveats name the first-arrival
  blind spot; every GNN-only FRAUD_DETECTED decision's caveats name the
  legitimate-repeat-applicant limitation and recommend manual review -
  neither finding was remediated this phase (no re-scan trigger built,
  GNN not retrained), which is stated explicitly rather than left
  ambiguous. Wired into both applications endpoints, wrapped in a
  defensive try/except despite having no real external dependency to
  fail. New tests/test_explainability.py (15 tests, all passed on the
  first real run) covering every status/signal combination plus two live-
  wiring tests. 132/132 passing (354 total across the project). VERSION
  bumped 0.13.0-phase15 → 0.14.0-phase16 (both response shapes genuinely
  changed). Updated ARCHITECTURE.md, API.md, DATA_FLOW.md, SECURITY.md,
  TESTING.md, ML_MODEL.md, central-fraud-intelligence/README.md.
  Committed, checkpoint updated.
- 2026-09-18 — Phase 17: fraud incidents. New src/incidents/incidents.py
  (pure record assembly - build_incident_record, derive_affected_
  applications, affected_platform_ids) and src/incidents/repository.py
  (PostgresIncidentRepository, a new fraud_incidents table, idempotent on
  triggering_event_id via a deterministic INC-<eventId> incident_id, same
  ON CONFLICT DO NOTHING pattern as event ingestion). Wired into
  POST /api/v1/applications/process right after the rule-engine step,
  gated on status == FRAUD_DETECTED only (a CLEARED near-miss was
  considered and explicitly rejected - no principled secondary threshold
  this project's own data would justify), best-effort/retry-safe like
  every prior pipeline step. Decided and documented that an incident can
  genuinely span multiple applications/platforms, derived from the same
  bounded subgraph the rule engine/GNN already scored - the concrete
  implementation of ARCHITECTURE.md section 9's "affected platforms
  {A, C}" framing from Phase 0 - proven for real by a test where a
  second, linked application's incident lists both applications while
  the first application's own earlier incident lists only itself.
  Decided that a GNN-only zero-rule-signal FRAUD_DETECTED decision (Phase
  15's finding 2) still creates a full incident rather than a suppressed
  one, carrying forward Phase 16's manual-review caveat, so the known
  limitation stays visible on the record rather than disappearing.
  review_status defaults OPEN; no transition endpoint built this phase
  (explicit Phase 19 scope decision). Two new admin-only read endpoints,
  GET /api/v1/incidents and GET /api/v1/incidents/:id (already named in
  API.md's Planned table since Phase 0, now real). New
  tests/test_incidents.py (17 tests, all passed on the first real run)
  covering pure assembly, real multi-platform live wiring, redelivery
  idempotency against the real table, and both new endpoints' auth/404
  behavior. 149/149 passing centrally; all four loan platforms re-run for
  real this session and confirmed unchanged (222) - 371 total across the
  project. VERSION bumped 0.14.0-phase16 → 0.15.0-phase17 (both
  applications endpoints gained a new field, two new endpoints exist).
  Updated ARCHITECTURE.md, API.md, DATA_FLOW.md, SECURITY.md, TESTING.md,
  ML_MODEL.md, central-fraud-intelligence/README.md. Committed, checkpoint
  updated.
- 2026-09-18 — Phase 18: fraud notifications. New
  src/notifications/notifications.py (pure build_notification_payload())
  and src/notifications/repository.py (PostgresNotificationRepository, a
  new fraud_notifications table - one row per (incident, platform) pair,
  deliberately not one row per incident with an embedded list, since
  delivery status is genuinely per-recipient; idempotent on
  (incident_id, platform_id) via a deterministic NOTIF-<incidentId>-
  <platformId> id). Notification *creation* wired into
  POST /api/v1/applications/process right after Phase 17's incident
  creation, gated on incident_row is not None, best-effort/retry-safe
  like every prior step. *Delivery* is deliberately NOT synchronous in
  that request - a new background asyncio poller,
  src/notifications/worker.py's NotificationDeliveryWorker (started/
  stopped in main.py's lifespan, the first genuinely recurring
  background task any phase here has added), reuses Phase 7's proven
  outbox pattern (same capped-exponential-backoff formula, same
  X-Idempotency-Key discipline) running in the opposite direction, and
  authenticates itself to each platform by reusing that platform's own
  existing shared secret (CENTRAL_ACCEPTS_API_KEY_{A,B,C,D} / each
  platform's CENTRAL_FRAUD_API_KEY) rather than provisioning a second
  key. Two new central admin-only read endpoints, GET /api/v1/
  fraud-notifications and GET /api/v1/fraud-notifications/:id (already
  named in API.md's Planned table since Phase 0, now real); deliberately
  did NOT build POST /api/v1/fraud-notifications (redundant) or
  POST /api/v1/fraud-notifications/:id/review (deferred to Phase 19,
  same as incident review). Built the receiving side fully on
  loan-platform-a first (this project's established build-one-then-
  propagate pattern) - new POST /api/v1/fraud-notifications/receive
  route (X-API-Key checked against the platform's own
  CENTRAL_FRAUD_API_KEY; alertId taken directly from the
  X-Idempotency-Key header, since the payload carries no notification id
  of its own), a new fraud_alerts Mongo collection + dual Mongo-backed/
  in-memory repository (idempotent upsert on alertId), and a new admin-
  only GET /api/v1/admin/fraud-alerts route - then propagated to
  loan-platform-b/c/d with identifier substitution and full independent
  re-verification, not assumed unchanged. New
  tests/test_notifications.py (30 tests, all passed on the first real
  run) covering pure payload assembly, the repository directly, the
  delivery worker against a real httpx.AsyncClient built on
  httpx.MockTransport (SENT/RETRY/FAILED outcomes, missing-config and
  missing-incident defensive cases), live wiring (single- and
  multi-platform incidents, redelivery idempotency), and both new
  endpoints. New tests/fraudAlerts.test.js (14 tests, identical across
  all four platforms after propagation) covering auth, validation,
  redelivery idempotency, standalone mode, and the new admin read route.
  179/179 passing centrally (149 prior + 30 new); all four loan platforms
  re-run for real this session (a 68/68, b 69/69, c 71/71, d 70/70) - 457
  total across the project. VERSION bumped 0.15.0-phase17 →
  0.16.0-phase18 (both applications endpoints gained a new field, two
  new endpoints exist). docker-compose.yml and root .env.example gained
  optional PLATFORM_{A,B,C,D}_BASE_URL vars (already match src/config.py's
  own defaults); no new secrets introduced anywhere. Updated
  ARCHITECTURE.md, API.md, DATA_FLOW.md, SECURITY.md, TESTING.md,
  central-fraud-intelligence/README.md, and all four loan platforms' own
  README.md. No ML_MODEL.md changes needed (notifications don't touch
  scoring/evaluation). Committed, checkpoint updated.
- 2026-09-18 — Phase 19: review workflow, graph explorer, metrics, and
  the central app's first-ever admin dashboard. Incident review real:
  POST /api/v1/incidents/:id/review transitions review_status (OPEN ->
  UNDER_REVIEW -> CONFIRMED/DISMISSED/ESCALATED, terminal states 409),
  appending {action,by,note,at} to a new review_history JSONB column
  (ALTER TABLE ADD COLUMN IF NOT EXISTS, the same schema-evolution
  pattern established in Phase 9) - the event shape mirrors
  loan-platform-a's own ReviewEventSchema field-for-field, found by
  reading that file for precedent rather than inventing a new one.
  Notification acknowledgement real, deliberately simpler than incident
  review: POST /api/v1/fraud-notifications/:id/review centrally - a
  one-way not-acknowledged -> acknowledged boolean (new
  acknowledged/acknowledged_by/acknowledged_note/acknowledged_at
  columns), 409 on a second attempt, orthogonal to delivery status - and
  the identical POST /api/v1/admin/fraud-alerts/:alertId/review on each
  loan platform's own FraudAlert, built fully on loan-platform-a first
  then propagated to b/c/d with independent re-verification, this
  project's established pattern. Graph explorer real with zero new
  traversal logic: GET /api/v1/graph/application/:id and
  GET /api/v1/graph/entity/:id are thin, admin-gated wrappers over
  GraphStore.get_relevant_subgraph() (real since Phase 11), hops/
  maxNodes clamped at the API layer. GET /api/v1/metrics real -
  operational counts only (events/incidents/notifications by status,
  graph node/edge counts), explicitly not performance data (stays Phase
  23), resolving the "19/23" ambiguity API.md's Planned table had left
  open since Phase 0. Built the central app's first-ever UI: a
  server-rendered Jinja2 admin dashboard (src/web/, /dashboard/*),
  chosen over a React/Vite SPA because this service has been 100%
  Python/FastAPI through Phase 18 (no Node build stage in the Dockerfile
  at all) and a Jinja2 page is testable directly with FastAPI's own
  TestClient - the same "test everything for real" discipline every
  other phase here has kept. Cookie-based auth (central_admin_token,
  HttpOnly, SameSite=Lax) reuses the identical JWT the JSON API's own
  admin login already issues, not a second login system. Zero
  client-side JavaScript - every mutating action is a plain HTML <form>
  POST, and every protected page redirects (303) to /dashboard/login
  rather than 401ing. New tests/test_graph_api.py (13 tests),
  tests/test_incidents_review.py (30 tests - pure-logic transition
  predicate tests plus live HTTP including a full real
  OPEN -> UNDER_REVIEW -> CONFIRMED sequence with accumulating
  reviewHistory), tests/test_notifications_review.py (9 tests),
  tests/test_dashboard.py (18 tests, real HTML via TestClient - actual
  requested pages, actual assertions on their content), and
  tests/test_metrics.py (6 tests) - all passed on the first real run
  except one caught-by-the-test-itself fix (test_metrics.py's event-
  repository seed keys corrected from snake_case to the real camelCase
  create_or_get() contract before the suite was declared passing, not
  discovered later). Each loan platform's fraudAlerts.test.js gained 5
  acknowledgement tests, verified independently on all four platforms;
  one pre-existing, unrelated tokenization.test.js flaky test observed
  on loan-platform-b (a probabilistic tamper-test edge case), confirmed
  flaky and unrelated to this phase by an immediate clean re-run of the
  full suite (74/74). 238/238 passing centrally (179 prior + 59 new
  across five files); all four loan platforms re-run for real this
  session (a 73/73, b 74/74, c 76/76, d 75/75) - 536 total across the
  project. VERSION bumped 0.16.0-phase18 → 0.17.0-phase19. requirements.
  txt gained jinja2==3.1.6 and python-multipart==0.0.26 (the latter
  required at runtime for FastAPI's Form() parsing, used by the
  dashboard's login form); no new environment variables anywhere - the
  dashboard reuses CENTRAL_JWT_SECRET. Updated ARCHITECTURE.md, API.md,
  DATA_FLOW.md, SECURITY.md, TESTING.md, central-fraud-intelligence/
  README.md, and all four loan platforms' own README.md. No
  ML_MODEL.md changes needed (this phase doesn't touch scoring/
  evaluation). Committed, checkpoint updated.

## Phase 20 — full four-platform integration (real multi-process run, two real bugs found and one fixed)

Goal per ARCHITECTURE.md section 11 and DEMO_SCENARIOS.md's Phase 0-era
end-to-end script: prove the whole pipeline running across all four
platforms *simultaneously, for real*, not just the pairwise scenarios
the automated test suite already covers. Confirmed at the start of this
session (re-checked, not assumed) that Docker Compose remains impossible
in this sandbox for a stronger reason than previously documented: not
just the Phase 1 registry block, but the Docker *daemon itself* cannot
start here at all (`ulimit: error setting limit (Operation not
permitted)`; `System has not been booted with systemd as init system`).
Also re-confirmed MongoDB remains unreachable (no apt package;
`fastdl.mongodb.org` rejected by the egress proxy) - see
TROUBLESHOOTING.md's new Phase 20 entries for the full detail.

**What was built.** Each loan platform gained `backend/scripts/
run_standalone.js` - a Phase-20-only entrypoint (never the production
`src/server.js`) that reuses the exact `createApp()` + in-memory-
repository wiring the test suite already uses, but calls `app.listen()`
on a real TCP port instead of driving requests in-process via supertest.
Verified structurally identical/platform-agnostic across a/b/c/d (only
`config.js`'s `PLATFORM_ID`/`PLATFORM_NAME` differ, read dynamically)
before copying unchanged, then smoke-tested standalone for real (`curl
/health`, a real login) before wider use. A new root-level `scripts/
run_integration_demo.py` orchestrates all five processes (loan platforms
on 4001-4004, central on 5000, real env vars overriding the Docker-DNS
defaults), resets and optionally seeds the real local Postgres dev
database, waits for every `/health`, then drives the actual 5(+1)-step
DEMO_SCENARIOS.md scenario over real HTTP: real registration/login on
each platform, real admin logins, real application submissions, real
polling of `GET /api/v1/incidents` and each platform's
`GET /api/v1/admin/fraud-alerts`.

**First real run: 0 incidents, not the expected 1.** The very first full
run (real applications submitted, real outbox delivery, real central
ingestion all confirmed via each process's own log) produced zero
incidents even for a textbook two-platform case (Platform A + Platform C
sharing phone+email). Investigated for real rather than assumed: queried
the live Postgres graph directly and found tokenization/entity
resolution had worked exactly right (A1001 and C1001's borrower nodes
correctly shared the same phone_token/email_token node keys, confirming
cross-platform entity resolution works for real, live, for the first
time outside a test fixture) and the rule engine had correctly computed
`rule_score=0.6` (max phone+email signal) for both - but `fraud_score`
was `0.0` (not `None`) for every single application, capping
`final_risk_score` at `GNN_WEIGHT(0.6) * 0 + RULE_WEIGHT(0.4) * 0.6 =
0.24`, well under the `0.5` threshold.

**Investigated the GNN score directly** (bypassing the API, calling
`GNNScorer.score_subgraph()` the same way `src/api/applications.py`
does): the real logit for a genuinely live-submitted, well-corroborated
application was approximately **-121 to -125** - an extremely confident
"not suspicious" prediction, sigmoid-collapsing to 0.0. Checked whether
this was a graph-size/cold-start artifact by seeding the same live
database with a real replay of the full, already-committed 129-
application Phase 12 scenario dataset (reusing `scripts/
evaluate_fraud_detection.py`'s own `_score_interleaved()` directly, which
itself reports 73.6% real accuracy on this same dataset/checkpoint
offline) - the same live application's subgraph grew from 21 to 211
nodes, and the logit was essentially unchanged (~-125 still). This rules
out overall graph size as the explanation; the real root cause is not
confirmed (see ML_MODEL.md's new Phase 20 section for the honest
non-claim and the hypotheses considered), and fixing a trained model's
behavior was judged out of scope for an integration-testing phase - this
is documented as a real, high-priority, unresolved open item rather than
worked around silently.

**Second, independent real bug found and fixed.** To isolate whether the
rule engine alone could still produce a correct decision (a fully
supported, documented code path - `combine_signals()`'s own docstring
promises rule-only decisioning when `gnn_score` is `None`), the real
checkpoint was temporarily moved aside for one run. Result: *every*
application got stuck at `GRAPH_UPDATED` forever, never reaching a
decision at all - worse than the GNN-collapse case, not better. Root
cause, found by reading `src/api/applications.py`: the rule-engine block
was only ever reached after a *successful* GNN score, so with
`gnn_scorer is None` it was structurally unreachable, contradicting
`combine_signals()`'s own long-standing documented contract. No test
caught this before now because every existing test either used an
isolated application (making "stuck at GRAPH_UPDATED" and "correctly
CLEARED" look identical) or provided a real/stub scorer - and
`tests/test_ingestion.py` had an explicit, passing Phase 10 assertion
(predating the Phase 15 rule engine) that had simply never been revisited
once it started describing a bug instead of a feature.

**Fixed** in `src/api/applications.py`: when `gnn_scorer is None`, the
event now advances straight to `ANALYZED` with `fraud_score` left `None`,
letting the rule engine's already-correct `None`-handling actually run.
Updated four now-stale test assertions to match the corrected behavior
(`test_ingestion.py`, `test_graph.py` x3, and `test_inference.py`'s
renamed+rewritten
`test_post_reaches_a_real_decision_with_fraud_score_null_when_no_scorer_is_loaded`).
Re-ran the full central suite twice - once with the checkpoint absent
(235 passed, 3 skipped - the 3 checkpoint-dependent tests correctly
self-skip), once with it restored (238/238, confirming zero regression in
the untouched `gnn_scorer is not None` path). All four loan platforms
re-run unchanged (73/74/76/75 = 298 - no loan-platform code touched this
phase). 536 total, same as Phase 19's count (this phase added test
*content*, not new test *count*, since it corrected 4 existing
assertions rather than adding new test files).

**Final verification run**, checkpoint deliberately set aside to exercise
the now-working rule-only path (clearly labeled as such, not presented as
GNN-driven detection - the checkpoint was restored to its normal on-disk
location immediately after): the real 5-step DEMO_SCENARIOS.md script
plus one additional step (added specifically to satisfy this phase's
"genuinely 3+ platform incident" requirement) all ran correctly against
five real, separate OS processes:
- Step 3 (Platform C shares phone+email with Platform A's A1001):
  **2 real incidents** created (one per triggering event -
  `INC-...` for A1001, `INC-...` for C1001), both with
  `affectedApplications: [A1001, C1001]` / `affectedPlatforms: ["a",
  "c"]` - correct and verified by reading the real Postgres rows, not
  just the API response. Platform A and Platform C each received exactly
  2 real notifications (`POST /api/v1/fraud-notifications/receive`,
  status `SENT` in each platform's own `fraud_alerts` collection);
  Platform B and Platform D received zero.
- Step 5 (Platform B shares only phone, not email, with the cluster):
  correctly stayed `CLEARED` (`rule_score=0.3`, below the `0.5`
  threshold) - this was this session's own test-scripting expectation
  being wrong, not a system defect: a single shared-token signal
  correctly isn't enough alone, by rules.py's own documented design.
- Step 6 (new - Platform D shares phone+email with the cluster too): a
  real, genuine **3-platform-plus incident** - `affectedApplications`
  actually spans all four platforms (`a`, `b`, `c`, `d`; Platform B's
  B1002 from step 5 is correctly included too, since it does share a
  real phone-token edge with this cluster even though that alone hadn't
  been enough to flag it on its own). Platform D received exactly 1 real
  notification.
- Database isolation re-checked for real after all six steps: Platform
  B's own `GET /api/v1/applications/history` (its own borrower's real
  JWT) does not contain Platform A's `A1001` application id.

Decided NOT to create a permanent `tests/test_e2e_*` suite this phase:
this one-off run already found two real, valuable issues that route-
level tests with stubs/None scorers structurally cannot find (see
TESTING.md's Phase 20 section for the full reasoning) - a permanent
suite's ongoing maintenance cost is deferred to Phase 30, which owns the
project's final end-to-end verification shape, rather than committing to
one now.

Updated TROUBLESHOOTING.md (three new entries: the stronger Docker-daemon
finding, the pipeline-gating bug and its fix, the unresolved GNN
live-inference collapse), ML_MODEL.md (the GNN finding, in full, with
the hypotheses considered and explicitly not claiming a root cause it
hasn't verified), TESTING.md (the new Phase 20 section describing the
real multi-process approach and what it does/doesn't prove), and
DEMO_SCENARIOS.md (the end-to-end script marked as actually exercised,
with real per-step output, correcting the original phone/device/bank
wording to phone/email - the two signals this project actually
implements - and correcting step 5's original expectation). No
ARCHITECTURE.md/API.md/DATA_FLOW.md/SECURITY.md changes needed (no new
endpoints, contracts, or security surface this phase - only a bug fix to
already-documented behavior and a new local-only demo script). Committed,
checkpoint updated.

## Phase 21 — false-positive testing at statistically meaningful scale

Goal per TODO.md's Phase 21 entry: Phase 15's real false-positive
measurement was real, but statistically thin - only 12 GENUINE
applications (9 hard negatives) out of 129 total, too small a sample to
trust in either direction. Phase 21's job was to re-measure the same
thing on a sample an order of magnitude larger, using the existing,
already-reviewed generator rather than building new evaluation
infrastructure from scratch.

**What was built.** `datasets/scenario_generator.py` gained a new opt-in
parameter, `include_pattern_21: bool = False`, and a new pattern 21
(`_legitimate_repeated_application_three_times`) - the same borrower
legitimately re-applying to the same platform a *third* time (extending
pattern 17's two-application repeat), added specifically because Phase
15 had already measured pattern 17 scoring ~0.9999 fraud probability (a
real false positive) and this project had no data yet on whether a
longer genuine repeat-borrower history makes that worse, the same, or
better. The flag defaults off specifically so the committed
`scenarios.json` (what Phase 13's checkpoint actually trained on) stays
byte-for-byte unchanged on a plain re-run of `scripts/
generate_scenarios.py` - verified directly by a new test,
`test_pattern_21_is_excluded_by_default_and_included_when_requested`.

A new script, `scripts/generate_fpr_dataset.py`, reuses
`generate_catalog()` at `instances_per_pattern=15` (5x the committed
dataset's 3) with `include_pattern_21=True`, producing a separate,
evaluation-only dataset: `seed-data/scenarios/scenarios_fpr.json` - 270
scenarios, 690 applications. Verified for real against a real
`GraphStore` on the test Postgres database before trusting it further:
`Applied 690/690 events. Graph now has 3050 nodes, 3990 edges.` - every
event genuinely ingestable, not just structurally plausible Python
objects. This dataset was never used for training and does not replace
`scenarios.json`.

That dataset was then run through the existing (unchanged this phase)
`scripts/evaluate_fraud_detection.py`, which calls the real production
pipeline functions directly (rule engine, hybrid combination, and the
real trained GNN checkpoint), in both its interleaved and isolated
modes, producing `central-fraud-intelligence/models/
hybrid_scoring_eval_fpr.json`.

**What it found - real, and more concerning than Phase 15's small sample
suggested:**
- Interleaved: accuracy 68.12%, precision 84.78%, recall 72.22%, F1
  0.78 (TP 390 / FP 70 / TN 80 / FN 150). Isolated: accuracy 65.94%,
  precision 84.27%, recall 69.44%, F1 0.7614 (TP 375 / FP 70 / TN 80 /
  FN 165). Both well below Phase 15's reported 73.6% on the small
  sample.
- Hard-negative accuracy: 55.56% (75/135) in both modes - barely better
  than a coin flip on data that is unambiguously legitimate by
  construction.
- The new pattern 21 is 0/45 - every single instance misclassified. The
  adversarial hypothesis behind adding it was confirmed, and more
  strongly than expected: a third legitimate repeat application doesn't
  fix pattern 17's ~50% false-positive rate, it makes it categorically
  worse.
- `normal_genuine_baseline` (pattern 18, the simplest possible GENUINE
  case - one applicant, one application, no shared signals with
  anything) is only 33.3% accurate (5/15) - the single most concerning
  number this phase produced, since it means the unreliability isn't
  confined to intentionally tricky hard negatives.
- A real, evidenced platform-bias finding: `normal_genuine_baseline`'s
  platform assignment is genuinely randomized in the generator
  (`rng.choice(PLATFORMS)`), yet all 10 of its misclassifications landed
  on platform B (4/4) or C (6/6), while every platform A (2/2) and D
  (3/3) instance was correctly cleared. Example real scores:
  `SYN-B-00223` (fraudScore 0.9999969, ruleScore 0.0, finalRiskScore
  0.5999981), `SYN-C-00133` (fraudScore 1.0, ruleScore 0.0,
  finalRiskScore 0.6). Because the platform assignment here is
  genuinely random, this rules out "the generator always puts these
  cases on the same platform" as the explanation - the model's behavior
  really does correlate with platform.
- A well-evidenced, explicitly-not-proven hypothesis for why: patterns
  17 and 21 both hardcode their own platform (`PLATFORMS[1]`/"b" and
  `PLATFORMS[2]`/"c" respectively, confirmed by grep, never randomized).
  Combined with Phase 13's already-documented severe overfitting
  (`finalTrainLoss: 3.86e-5`), a plausible explanation is a spurious
  platform-based shortcut learned during training rather than a genuine
  identity/graph-structure signal. No ablation was run to confirm this
  (that's future GNN-work scope, not this phase's) - stated as a
  hypothesis, not a finding, deliberately.
- Synthesized with Phase 20's separate live-inference-collapse finding
  (near-zero scores for genuinely novel live requests): both read as
  symptoms of the same underlying cause - severe overfitting to a small,
  low-diversity training set - producing unreliable output outside the
  narrow slice of input space the model actually trained on, with the
  specific failure mode (over-confident SUSPICIOUS vs. under-confident
  GENUINE) depending on which part of that space a given input lands in.

**Honest process note, in keeping with this project's no-fake-success
discipline:** the first version of the new
`test_legitimate_repeated_application_three_times_is_the_same_borrower_three_times`
test actually failed on its first real run - it asserted the three
generated applications would land in three different calendar months,
but 30-real-days-apart timestamps starting mid-month don't reliably do
that (two of the three landed in the same month in the real generated
data: `{'2026-01', '2026-03'}` for i=0,1,2 with a 30-day step from a
January base). This was a wrong assumption in the test, not a bug in the
generator - fixed by asserting the three `submittedAt` timestamps are
distinct and strictly increasing instead, which is what the pattern
actually needs to prove. Full central pytest suite re-run after the fix:
**240 passed, 0 failed** (238 pre-existing + 2 new).

Updated ML_MODEL.md (new "Phase 21: false-positive testing at
statistically meaningful scale" section, in full, with every number
above plus the per-pattern table and the platform-bias evidence),
DEMO_SCENARIOS.md (pattern 21 added to the catalog list, a new "Phase 21
addendum" section describing `scenarios_fpr.json` without altering the
original Phase 12 coverage statement, which still accurately describes
the unchanged `scenarios.json`), TESTING.md (new Phase 21 section on the
evaluation methodology and the honest test-fix note), and
TROUBLESHOOTING.md (new Phase 21 entry, structured to match Phase 20's
GNN-collapse entry, flagging this as a second open item for future
GNN-focused work). `src/config.py`'s `VERSION` was deliberately **not**
bumped - Phase 21 adds offline evaluation tooling only (a new generator
pattern behind an opt-in flag, a new dataset-generation script, no
changes to any FastAPI route or request-handling code), the same
reasoning Phase 12/13 already established for offline-tooling-only
phases. Committed, checkpoint updated.

## Phase 22 — security testing (two real application-code findings fixed, audit logging built, a full dependency audit run for real)

Goal per TODO.md's Phase 22 entry: actually test the security properties
this project's own SECURITY.md has claimed since Phase 9, rather than
re-asserting them, plus two carried-forward items explicitly deferred to
this phase (the Phase 1 npm-audit finding, and a full dependency audit
across all five services) and the long-standing `src/audit/` stub, whose
own docstring has said "Phase 9/22" since Phase 9.

**Confirmed correct, for real, not just by reading the code**: every SQL
query across `central-fraud-intelligence/src/*/repository.py` is
genuinely parameterized (confirmed by grep across every `.execute(` call
- the handful of f-strings found interpolate only a hardcoded column-name
constant, never request data) - proven with a real SQL-injection-style
probe string (`'; DROP TABLE ingested_events; --`) sent through the live
ingestion endpoint and confirmed stored as inert text, table still
intact. The hand-rolled JWT (`src/security/jwt_auth.py`) rejects a forged
"alg: none" token - it has no code path that branches on the header's own
claimed algorithm, so this was never really an attack surface, now proven
rather than assumed. Admin login's anti-enumeration design (same error
for unknown email vs. wrong password, a real Phase 9 property) is
genuinely indistinguishable at the HTTP layer, verified directly.

**Two real application-code findings, both fixed this phase** (identical
across all four loan platforms - `authService.js`/`adminService.js` are
byte-for-byte the same file on each, confirmed by diff before and after):
1. `authService.js`'s `register()`/`login()` didn't reject a non-string
   `email`/`password`. A JSON body like `{"email": {"$ne": null},
   "password": "x"}` would previously reach `userRepository.findByEmail()`,
   which calls `.toLowerCase()` directly - against the real Mongo-backed
   repository this is a genuine operator-injection vector; the fact that
   `.toLowerCase()` throws on a non-string was accidental luck (a 500, not
   a clean validation error), not a real defense. Fixed by rejecting any
   non-string name/email/password/phone in the service layer, before
   anything touches a repository - closes the vector by construction.
   Real MongoDB is unreachable in this sandbox (the standing Phase 1
   limitation), so this was found by code review and fixed/tested against
   the in-memory repository double every other test in this project
   already trusts, not a live exploit.
2. `GET /api/v1/admin/applications?status=...` passed `req.query.status`
   unvalidated into `applicationRepository.listAll({ status })`, which the
   real Mongo repository turns into `Application.find({ status })`.
   Express's default query parser turns `?status[$ne]=REJECTED` into
   `req.query.status = { $ne: "REJECTED" }` - a real operator-injection
   shape, gated behind admin auth (lower severity) but still a real
   defect. Fixed by only accepting one of the platform's own known status
   values (reusing `VALID_TRANSITIONS`'s keys as the single source of
   truth) - anything else is now rejected with a 400.
Both fixes covered by a new `tests/security.test.js` per platform (7
tests each, adapted to each platform's own `validPayload` conventions,
not copy-pasted) - the operator-injection test genuinely reproduces the
`?status[$ne]=...` shape and confirms it's now rejected, not just
asserts the fix exists.

**Audit logging built for real**, closing the `src/audit/` stub Phase 9
left with a "Phase 9/22" docstring: `src/audit/repository.py`'s
`PostgresAuditRepository`, with a field-name denylist
(`DENIED_DETAIL_KEYS`) enforced inside the repository itself at write
time, not left to call sites to remember. Wired into admin login
(success and failure), incident review, and notification review - each a
best-effort write (a database hiccup while recording an audit event must
never block the real action it's recording, the same graceful-degradation
posture this project has used since Phase 11). `tests/test_audit_log.py`
(10 tests) verifies the denylist directly and, more importantly, sends a
real password through a real login attempt and confirms it's nowhere in
the stored row - the actual verification this gap has been waiting for
since Phase 9, not a repeat of the same assertion.

**Full dependency audit, run for the first time this project has run
either tool**: `pip-audit` against `central-fraud-intelligence/
requirements.txt` found 29 vulnerabilities across 3 packages.
`cryptography` (46.0.7→50.0.1, 7 CVEs, all in X.509/PKCS7 code paths this
app never calls) and `python-multipart` (0.0.26→0.0.32, 8 CVEs, genuinely
relevant since `src/web/dashboard.py`'s forms use it) were both bumped
and the full central suite re-verified green afterward (258/258).
`starlette` (14 CVEs, a transitive dependency of the pinned
`fastapi==0.115.6`, fix versions requiring a `starlette`/`fastapi`
major-version bump) was deliberately left alone this phase - reviewed for
actual relevance (confirmed by grep that this app uses none of
`StaticFiles`/`HTTPEndpoint`/`FileResponse` - three of the seven distinct
CVE categories don't even apply here), and a major-version bump judged too
large a blast radius for this already-large phase, the same reasoning
this project has applied to the vite/esbuild finding since Phase 1. `npm
audit` re-run on all 8 Node packages (4 backends, 4 frontends): every
backend clean, every frontend showing the same pre-existing esbuild/vite
dev-dependency finding (now 2 advisories, one bundling 3 CVE
descriptions - advisory data growth, not a new issue), still requiring a
breaking `vite@8.3.0` bump to fully resolve, still deliberately deferred
for the same reasoning.

**Deliberately not done this phase, stated plainly rather than left
stale**: rate limiting (still no middleware/custom code anywhere,
confirmed) and CSRF protection for the dashboard (still relies only on
`SameSite=Lax`) both remain unimplemented - re-reviewed, not silently
ignored, and both judged to need their own dedicated design pass rather
than a hasty addition squeezed into this phase. No timing-attack testing
was attempted against the existing constant-time comparisons
(`hmac.compare_digest` in `src/security/auth.py`/`jwt_auth.py`) - their
presence in the code was confirmed by reading it, which this file states
as a weaker claim than an actual timing measurement, not blurred into one.

Test counts after this phase: central **258/258** (240 Phase 21 baseline
+ 18 new: 10 audit-log tests, 8 hardening tests). Loan platforms **326
total** (298 Phase 20 baseline + 28 new: 7 security tests × 4 platforms) -
80/81/83/82 for a/b/c/d respectively. `VERSION` in
`central-fraud-intelligence/src/config.py` **was bumped** this phase
(0.18.0-phase20 → 0.19.0-phase22, see config.py's own updated docstring)
- unlike Phase 21's offline-tooling-only phase, this one genuinely
changes request-handling behavior: admin login and the two review
endpoints now each write a real audit-log row as a side effect of a
successful (or, for login, even a failed) call, even though none of
their request/response JSON shapes changed. Updated SECURITY.md (the largest single
addition this phase - full "Phase 22" section plus updated
Planned-controls bullets), TROUBLESHOOTING.md (Phase 1 npm-audit entry
revisited in place, new Phase 22 entry), TESTING.md (new Phase 22
section), TODO.md, BUILD_STATE.json. No ARCHITECTURE.md/API.md/
DATA_FLOW.md changes needed - no new endpoint shapes or data-flow steps,
only hardening of existing ones and a new internal-only audit trail no
API response exposes. Committed, checkpoint updated.

## Phase 23 — performance testing (real load, one real production concurrency bug found and fixed)

Goal per TODO.md's Phase 23 entry: establish real, honestly-measured
latency/throughput baselines for the central app and all four loan
platforms - this project had no dedicated performance-testing tooling or
numbers of any kind before this phase.

**Baseline verification surfaced a real, pre-existing flaky test before
any new work started** (this project's standing protocol: always
re-verify the suite before starting a new phase). A full central pytest
run failed once, non-deterministically: `test_decrypt_tampered_
ciphertext_fails` forced the ciphertext's first byte to the literal
`0xff`, assuming that always differs from the real (random-IV-derived)
byte - AES-GCM's keystream makes that byte effectively uniformly random
per call, so roughly 1-in-256 runs the real byte already happened to be
`0xff`, making the "tampered" ciphertext byte-for-byte identical to the
original and correctly decryptable (no tampering had actually occurred).
Confirmed empirically (a 20000-trial loop: 86 had first byte `ff`,
matching the expected ~78.1) before fixing, not just asserted. Never a
real cryptographic weakness - AES-GCM's authentication tag genuinely
detects real tampering, which is exactly what this test exists to prove;
only its own tampering method was occasionally a no-op. Fixed by
XOR-flipping the observed byte instead of hardcoding a literal
(mathematically guaranteed to differ from the original regardless of its
value), identically across `central-fraud-intelligence/tests/
test_tokenization.py` and all four platforms' `tokenization.test.js`
(confirmed byte-for-byte identical before the fix, via diff). Verified
with a 5000-trial loop (0 unexpected successes) and by running each
affected file individually (Python 16/16, each platform's JS 13/13)
before proceeding.

**`scripts/run_performance_tests.py` written** (new, ~450 lines, root
level): reuses Phase 20's `run_integration_demo.py` process-launch
conventions (genuinely separate OS processes: the real central
FastAPI/uvicorn app against this sandbox's real local Postgres, all four
loan platforms' real `run_standalone.js` entrypoints) on different ports
so the two scripts never collide, then drives real, timed,
bounded-concurrency HTTP load via `httpx`+`asyncio.Semaphore`+
`asyncio.gather`, with real per-request timing (`time.perf_counter()`)
and p50/p95/p99/mean/throughput summarization. Toggles the GNN checkpoint
by moving `models/gnn_checkpoint.pt` aside and back
(`shutil.move`) to force two real measurement rounds - `GNNScorer.load()`
genuinely returns `None` with it absent, exercising the app's real
rule-only gating path, not a simulated one. Measures: central ingestion
(cold/warm graph × rule-only/with-GNN × concurrency 1/10), central reads
(`applications/:id`, `incidents`, `metrics`), and each loan platform's
submission and admin-read endpoints.

**This phase's own first three runs found three real bugs, none worked
around**, consistent with this project's standing practice of
investigating a real failure to its actual root cause:

1. `scripts/evaluate_fraud_detection.py`'s warm-graph seeding step (reused
   from Phase 20's demo) crashed with `AttributeError: 'NoneType' object
   has no attribute 'score_subgraph'` the first time it was ever run with
   the GNN checkpoint deliberately absent. Root cause: `_replay_one_event()`
   called `gnn_scorer.score_subgraph(...)` unconditionally, but this
   function's own docstring has always claimed to mirror
   `src/api/applications.py`'s pipeline "exactly" - and production has
   correctly handled `gnn_scorer is None` since a real Phase 20 fix (see
   that phase's own PROGRESS.md entry). This evaluation script never got
   the equivalent fix because every prior run (Phase 15/20/21) always had
   a real trained checkpoint loaded; Phase 23's rule-only benchmark round
   is the first caller ever to invoke it without one. Fixed by adding the
   identical `gnn_scorer is None` gating production already has.
2. Platforms B (TrustLend) and D (FinBridge) rejected every synthetic
   submission with 400 - not a platform bug. Each of the four loan
   platforms genuinely enforces a different loan schema (`ARCHITECTURE.md`'s
   "four institutions, not four copies" requirement, confirmed by reading
   each platform's `applicationService.js`): B requires
   `loan.institutionName`/`loan.courseName` (education loans only), D
   requires `loan.merchantName`/`loan.productCategory` from a fixed enum
   (point-of-sale loans only) - a real, intentional, pre-existing design
   difference this benchmark script's own payload builder hadn't yet
   accounted for. Fixed by giving `bench_platform_submission()` a
   per-platform loan payload.
3. **A real, previously-latent production concurrency bug** - the
   headline finding of this phase. After fixes 1-2, the third full run
   completed but showed sporadic real `503 Service Unavailable` from
   `GET /api/v1/incidents` at concurrency=10 (2-4 of 30 requests, across
   two separate runs), never at concurrency=1, never on any other
   endpoint. Investigated rather than dismissed as noise: every
   repository in this project shares a lazy `ensure_schema()` pattern -
   check a plain `self._schema_ready` boolean, and if unset, run `CREATE
   TABLE IF NOT EXISTS ...` then set it. FastAPI dispatches each sync
   `def` route to Starlette's real thread pool, so several concurrent
   requests that are the first to touch a freshly-started process's
   shared repository instance can all see the guard as unset at once and
   each independently run the DDL. Reproduced directly against this
   sandbox's real Postgres, outside the HTTP layer entirely (10 threads
   calling `PostgresIncidentRepository.list_all()` on one freshly-reset
   instance): a genuine `psycopg.errors.DeadlockDetected`, confirmed by
   `__mro__` inspection to be a real `psycopg.OperationalError` subclass
   - exactly why the existing `except psycopg.OperationalError` handling
   correctly (if misleadingly, from the caller's perspective) turned it
   into an honest 503 rather than crashing or hanging. A tempting
   alternative hypothesis - `CREATE TABLE IF NOT EXISTS` racing into a
   `UniqueViolation` against Postgres's own catalog tables - was tested
   directly (10 concurrent threads against a real dropped-and-recreated
   table) and ruled out: it reproduces, but as a `psycopg.errors.
   UniqueViolation`, a sibling of `OperationalError` in the DB-API
   hierarchy, not a subclass - it would surface as an unhandled 500, not
   the 503 actually observed, so this was not the real mechanism, and
   this file says so rather than presenting the first plausible-sounding
   theory as the answer.

   **Fixed** with a per-instance `threading.Lock`, double-checked against
   the existing guard, applied identically to all six repositories
   sharing this exact pattern (`src/incidents/repository.py`,
   `src/audit/repository.py`, `src/graph/repository.py`,
   `src/ingestion/repository.py`, `src/notifications/repository.py`,
   `src/security/admin_repository.py`) plus `src/graph/graph_store.py`'s
   `_ensure_loaded()`, which has the identical unguarded check-then-act
   shape (mutating a shared `networkx.MultiDiGraph`/`dict` instead of
   running DDL) even though no concrete failure from that one was
   observed in this sandbox's own runs - fixed as a precaution rather
   than left asymmetric with the six DB repositories. Verified fixed
   three independent ways: a direct 5-trial × 10-thread repro against the
   live fix (0 errors, down from a reliable reproduction before it); a
   new `tests/test_schema_race.py` (7 tests - one parametrized across all
   six Postgres repository classes, one for `GraphStore` - folded into
   the normal suite as permanent regression coverage, not a one-off
   script); and a final, clean full run of the performance harness
   showing **zero errors across every single benchmark**, including `GET
   /api/v1/incidents` at concurrency=10, which had never once been clean
   in this phase's first three runs.

**Real measured results, full methodology, and honest analysis**: see
new root `PERFORMANCE.md` in full - not duplicated here. Headline,
stated plainly: GNN inference adds a real, consistently-directional
latency cost (roughly +8% at concurrency=1, +16-29% at concurrency=10,
depending on graph warmth); central ingestion's ~280-320ms
single-request latency floor is best explained, from actually reading
the code, by this project's "no connection pooling - a brand-new
`psycopg.connect()` per repository method call" design (confirmed:
`GraphStore.apply_application_submitted()` alone makes up to ~10-16 such
calls per request) rather than by the rule engine or GNN inference
themselves - flagged as a concrete, well-understood optimization target
for a future phase, not fixed this phase (large blast radius, same
"deserves its own dedicated phase" reasoning Phase 22 already applied to
the `starlette`/`vite` dependency upgrades); every loan platform's
in-memory-repository path is fast (single-digit-to-tens of milliseconds)
with platform D showing an unexplained, honestly-flagged-as-unexplained
latency outlier worth a repeat measurement rather than either a
dismissal or an over-confident explanation. Run-to-run variance between
this phase's third (bug-fixed but not yet clean) and fourth (final,
clean) runs was single-digit-percent at concurrency=10 - stated
explicitly so a future reader has a real baseline for "is this normal
sandbox noise or a real regression" rather than having to guess.

Test counts after this phase: central **265/265** (258 Phase 22 baseline
+ 7 new, all in `test_schema_race.py`). Loan platforms **326 total**,
unchanged from Phase 22 - the tokenization-test fix changed test
*content* (one existing test per platform rewritten), not test *count*.
`VERSION` in `central-fraud-intelligence/src/config.py` **was bumped**
this phase (0.19.0-phase22 → 0.20.0-phase23, see config.py's own updated
docstring) - unlike Phase 21's offline-tooling-only phase, the
`threading.Lock` fix is a genuine behavior change under concurrency for
every endpoint touching any of the seven fixed components, even though
no request/response JSON shape changed. Updated TESTING.md (new Phase 23
section), TROUBLESHOOTING.md (two new Phase 23 entries: the concurrency
bug, marked fixed rather than left open, and the connection-pooling
finding, marked deliberately not fixed), new root PERFORMANCE.md (the
largest single addition this phase), TODO.md, BUILD_STATE.json. No
ARCHITECTURE.md/API.md/DATA_FLOW.md/SECURITY.md changes needed - no new
endpoint shapes, data-flow steps, or security-relevant behavior; this
phase's one production-code fix is a concurrency-correctness fix, not a
new control or a new attack surface. Committed, checkpoint updated.

## Phase 24 — Docker end-to-end testing (daemon genuinely works this session; registry access is the real, confirmed blocker)

Goal per TODO.md's Phase 24 entry: re-attempt Docker end-to-end
verification for real rather than re-asserting Phase 20's "completely
impossible" finding unchanged for a fourth phase, and verify whatever
this sandbox *can* verify about the Docker artifacts if the daemon still
cannot run.

**Re-investigated the daemon question from scratch.** Phase 20's own
entry documented `service docker start` failing with `ulimit: error
setting limit (Operation not permitted)` and "System has not been booted
with systemd as init system (PID 1)". This phase tried running `dockerd`
directly instead of through that `service`/systemd wrapper - and it
started successfully: `docker info`, `docker build`, and `docker run` all
genuinely work in this session. Verified concretely, not just by
`docker info` reporting healthy: built and ran a real `FROM scratch`
image containing a freshly `gcc -static`-compiled probe binary, with
zero container-registry calls involved, proving the daemon/BuildKit/
containerd/runc stack itself is functional here, independent of any
network question. Stated honestly as a **this-session** finding, not a
blanket correction of Phase 20 - sandbox capabilities evidently vary
between sessions (this phase's own result differs from Phase 20's under
what is nominally the same kind of build sandbox), so a future phase
should re-verify fresh rather than trusting either entry blindly.

**The real, remaining, confirmed blocker is container-registry access,
not the daemon.** Tested three separate registries directly, not just
re-tried Docker Hub: `registry-1.docker.io` (needed for every base image
this project's five Dockerfiles pin - `node:22-slim`, `python:3.11-slim`,
`postgres:16-alpine`, `mongo:7`), `ghcr.io`, and Google's Docker Hub
mirror `mirror.gcr.io` - all three returned `403 Forbidden` through this
session's egress proxy. Confirmed from the proxy's own diagnostic log
(`http://127.0.0.1:37829/__agentproxy/status`), not inferred from the
failures alone: a `recentRelayFailures` entry explicitly records
`registry-1.docker.io:443` as `connect_rejected` with detail `"gateway
answered 403 to CONNECT (policy denial or upstream failure)"` - a named,
real, organization-level egress policy, and per `/root/.ccr/README.md`'s
own instruction ("do not retry or route around it"), not something this
phase attempted to work around by rewriting the project's Dockerfiles to
avoid Docker Hub base images - that would be an unrequested architecture
change made solely to dodge a sandbox limitation, the same restraint
every prior phase has applied to its own sandbox-specific blocker
(MongoDB, PyTorch's CUDA wheel, `starlette`'s major-version bump, and
now this).

**New `scripts/verify_compose.sh`** runs every check this sandbox can
actually perform without needing a container registry, and reports
honestly which checks ran versus which were skipped and why, rather than
silently passing or silently omitting the registry-dependent ones:
`docker compose config` structural validity (passes); every one of the
five Dockerfiles' literal `COPY` source paths actually exist in the repo
(passes for all five - a real bug in this checker script itself was
found and fixed on its own first run, before trusting its output: it
mis-parsed `COPY --from=<stage> /app/... ./public` multi-stage
references as host filesystem paths, producing 4 false positives, fixed
by skipping any COPY line containing `--from=` entirely rather than only
skipping that one token); published host ports across all 10 containers
are unique (passes, 10 ports); per-platform database network isolation,
parsed directly from the resolved compose config rather than assumed
from reading the YAML by eye (passes for real, for all four platforms -
each loan platform's backend is only ever on its own `net-{a,b,c,d}-db`
network plus the shared `net-central-bus`, confirmed to have zero
network path to any other platform's Mongo or to the central app's own
`net-central-db` - a genuine, structural, Docker-network-level
enforcement of `ARCHITECTURE.md`'s database-isolation requirement, not
merely an application-level convention); the registry-free daemon
capability probe described above (passes); and a real `docker compose
build`, honestly reported as **skipped** (not passed) since
`registry-1.docker.io` is unreachable this session.

Separately, outside that script (since npm's and pip's *package*
registries are reachable here even though every *container* registry is
blocked - a real, different distinction, not a contradiction): verified
directly that every `RUN` step inside each Dockerfile that doesn't
itself need a container-registry pull actually succeeds - `npm install`
+ `npm run build` for a loan platform's frontend (a real `vite build`
completed, real `dist/` output produced) and `npm install --omit=dev`
for a backend (clean, 0 vulnerabilities in production dependencies,
confirming the earlier `npm audit` findings from Phase 1/22 are
dev-dependency-only, as those phases already concluded) - both run in a
scratch location and cleaned up afterward, not committed. `pip install
-r requirements.txt` for the central app needed no fresh verification -
it has already been exercised, extensively and repeatedly, by every
phase's real pytest runs in this same sandbox since Phase 8.

**Still not verified, stated plainly, same honest ask every Docker
section since Phase 1 has made**: a full `docker compose up --build`
actually bringing up all 10 containers together (4 loan platforms + their
4 Mongo instances + the central app + its Postgres) and exercising the
real Mongo-backed system end-to-end - this needs an environment with
real container-registry access, which this sandbox's egress policy does
not grant, confirmed rather than assumed. `VERSION` in
`central-fraud-intelligence/src/config.py` was **not bumped** this
phase - no `src/` changes at all, only a new verification script and
documentation, the same "offline-tooling-only" reasoning Phase 12/13/21
already established. Updated TESTING.md (new Phase 24 section, plus the
"Container health" planned-surfaces row marked done-with-caveat),
TROUBLESHOOTING.md (new Phase 24 entry, explicitly reconciling itself
against Phase 20's differing finding rather than silently overwriting
it), TODO.md, BUILD_STATE.json. No ARCHITECTURE.md/API.md/DATA_FLOW.md/
SECURITY.md/PERFORMANCE.md changes needed - this phase verified existing
infrastructure, it didn't change any endpoint, data flow, or security
control. Committed, checkpoint updated.

## Phase 25 — cloud deployment preparation (AWS, real IaC/pipeline written and syntax-checked; no cloud account exists to deploy to, so none was)

Scope: per the original 27-phase plan, Phase 25 is cloud deployment
preparation. This project had never named a cloud provider before this
phase - confirmed by grepping every `.md` file in the repo for
`aws|gcp|azure|kubernetes|k8s|ecs|fargate|cloud run|heroku|render.com|
railway|digitalocean` before choosing one; zero prior matches anywhere.
Per this project's standing rule, restated at the top of `DEPLOYMENT.md`
since Phase 1: cloud deployment is never claimed unless it was actually
performed in this environment. It was not performed this phase either -
this sandbox has no AWS account or credentials, same as every prior
phase. What Phase 25 delivers instead is real, concrete, reviewed
deployment *preparation*, labeled exactly that throughout.

**Target chosen: AWS.** Reasoning (full detail in
`deploy/aws/terraform/README.md`): ECS Fargate matches this project's
existing "each service independently buildable/deployable, no shared
build-time dependency" design (`ARCHITECTURE.md`) at exactly the right
granularity - one Fargate service per `docker-compose.yml` application
container, no Kubernetes control plane needed for 5 containers. RDS
Postgres and DocumentDB are managed, same-engine replacements for
`central-db` and `mongo-loan-{a,b,c,d}` respectively (no database
migration implied). A single-provider design was chosen deliberately over
mixing vendors, to keep IAM/networking/secrets in one place.

**What was built**, all in `deploy/aws/terraform/` (10 `.tf` files) and
`.github/workflows/deploy.yml`:
- VPC with public/app/data subnet tiers across 2 AZs, NAT gateway.
- **11 security groups reproducing `docker-compose.yml`'s 6-network
  isolation topology exactly** - the load-bearing part of this design,
  since per-platform database isolation is a stated project requirement
  (`ARCHITECTURE.md` section 3), not incidental. Each platform's
  DocumentDB cluster accepts inbound traffic only from that platform's
  own ECS security group - verified by re-reading `docker-compose.yml`'s
  actual `networks:` blocks line by line while writing the mapping, not
  assumed from memory of the general shape (`network.tf`'s header has the
  full net-\* -> security-group table).
- ECS cluster, 5 task definitions (torch-loading central app gets more
  CPU/memory than the 4 Node loan platforms), 5 services, least-privilege
  IAM (execution role's Secrets Manager grant is scoped to exactly this
  project's own secrets, not `*`).
- RDS PostgreSQL 16 (multi-AZ, encrypted, 7-day backups) for the central
  app; 4 separate DocumentDB clusters (one per loan platform, not 4
  databases sharing one cluster - the strongest AWS-native isolation
  available) with a compatibility caveat stated plainly in `docdb.tf`
  (DocumentDB is Mongo wire-protocol compatible, not a 1:1 feature match -
  never tested against this project's actual data-access code, since no
  AWS account exists in any sandbox this project has used).
- A public ALB terminating TLS 1.2+ only (HTTP:80 redirects to HTTPS:443,
  no plaintext listener), host-based routing to all 5 services, health
  checks against the same `/health` path every Dockerfile already probes.
- Secrets Manager: one secret per sensitive `.env.example` value, wired
  into each task's `secrets:` block - Terraform creates the containers
  with a random placeholder only, real values are set out-of-band, never
  through Terraform state.
- `.github/workflows/deploy.yml`: build+push all 5 images to ECR via AWS
  OIDC role assumption (no static AWS keys in GitHub), then
  `terraform plan`. `terraform apply` is a separate, `if: false`-gated
  job requiring a manual environment-approval gate - deliberately not
  wired to run unattended for a system provisioning real databases and
  secrets.

**Two items from Phase 24's own next-step note, both addressed
honestly:**
1. The Phase 13/14 torch/CUDA image-bloat issue -
   `central-fraud-intelligence/Dockerfile` now installs torch's CPU-only
   wheel via `--extra-index-url https://download.pytorch.org/whl/cpu`
   *before* the rest of `requirements.txt`, so the multi-GB CUDA-bundled
   default PyPI wheel is never pulled. Re-tested this phase:
   `download.pytorch.org` is still blocked by this sandbox's egress
   policy (same `connect_rejected`/403 signature as Phase 13's original
   finding) - the fix is written correctly per PyTorch's own documented
   install instructions but has **never been executed in any sandbox this
   project has used**. NOT VERIFIED, stated plainly rather than silently
   carried forward unfixed for a fourth phase running.
2. `docker compose up --build` end-to-end - still not run, same
   registry-access blocker as Phase 24 (re-confirmed, not re-investigated
   from scratch again this phase, since Phase 24 already did that
   investigation thoroughly this session-family).

**What could not be exercised, and the real substitute check that was
actually run instead**: the Terraform CLI itself could not be installed
(`releases.hashicorp.com` also blocked - tested directly, same
`connect_rejected` signature), so `terraform validate`/`plan` have never
run against this code. Real substitute, actually executed: `pip install
python-hcl2` (PyPI is reachable) and every `.tf` file parsed successfully
as syntactically valid HCL2. `.github/workflows/deploy.yml` was parsed
with `yaml.safe_load` and confirmed valid YAML. Both are real, weaker-
than-ideal checks, reported as exactly that (syntax verified, AWS-
provider semantics NOT verified) in `TROUBLESHOOTING.md`,
`deploy/aws/terraform/README.md`, and `TESTING.md`'s new Phase 25
section - not inflated into "the infrastructure is verified."

**Regression check, run for real this phase** (Phase 25 touched
`central-fraud-intelligence/Dockerfile` and
`requirements.txt`'s comments, not `src/`, but the full suites were
re-run anyway rather than assumed unaffected): this session's Postgres
had to be started fresh (`service postgresql start` - a known, previously
documented per-session characteristic, not a new finding), after which
`central-fraud-intelligence`'s full suite ran clean: **265 passed**, same
count as Phase 23/24. All four loan platforms' Jest suites also re-run:
**a=80, b=81, c=83, d=82, 326 total passed** - identical counts to
Phase 23/24, confirming this phase's documentation/IaC/Dockerfile changes
introduced no regression.

**`VERSION` in `central-fraud-intelligence/src/config.py` was NOT
bumped** this phase - the only change to that service was to its
`Dockerfile` and a `requirements.txt` comment block, no `src/` code
changed, same "offline-tooling/non-application-code" reasoning Phase 24
already established for itself.

Updated: `DEPLOYMENT.md` (full "Cloud: AWS" section replacing the Phase 1
stub's placeholder list), `SECURITY.md` (Phase 25 addition: TLS
termination and Secrets Manager key storage now concretely designed,
resolving two previously-open bullets - rotation *procedure* remains
explicitly deferred), `TESTING.md` (new Phase 25 section), `TROUBLESHOOTING.md`
(new Phase 25 entry, re-confirming two more egress blocks and documenting
the real hcl2/yaml syntax-check workaround), `TODO.md`, `BUILD_STATE.json`.
No `ARCHITECTURE.md`/`API.md`/`DATA_FLOW.md`/`PERFORMANCE.md` changes
needed - no endpoint, data flow, or runtime behavior changed this phase.
Committed, checkpoint updated.

## Phase 26 — final end-to-end verification (closing phase - all 27 planned phases now complete)

Scope: per the phase list, Phase 26 is final end-to-end verification -
the closing phase of this project's original 27-phase plan (Phase 0-26).
No application code was changed this phase, deliberately: this is a
verification and documentation-consistency phase, not a catch-all bugfix
phase for every item `KNOWN_ISSUES.md` (new this phase) lists.

**Final fresh test run, executed for real one more time in this
session** (not a restatement of Phase 25's numbers, an actual re-run):
`central-fraud-intelligence` 265/265 passing; loan platforms
a=80/b=81/c=83/d=82, 326/326 total. **Project total: 591/591 automated
tests passing.** This is the final, closing verification number for this
project's 27-phase build.

**Full consistency pass across every doc**, checking they agree with
each other and with the actual code now that 26 phases of incremental
edits have accumulated. Real, minor, documentation-only drift was found
and fixed - genuine findings from this pass, not assumed clean:
- A repeated arithmetic slip: "9 containers" appeared three times
  (`TESTING.md`, `PROGRESS.md`'s Phase 24 section twice, and
  `BUILD_STATE.json`'s own `known_errors` entry) where the real,
  consistently-stated topology everywhere else is 10 services (4 loan
  platforms + 4 Mongo + 1 central app + 1 central-db) - fixed in all four
  places.
- `README.md`'s "Build status: incremental, checkpointed build in
  progress" banner and "Run it (once later phases have added real
  functionality)" heading, both accurate at Phase 1 but stale by Phase
  26 - rewritten to state all 27 phases are complete, while adding an
  explicit "complete describes planned scope, not zero remaining issues"
  caveat pointing at the new `KNOWN_ISSUES.md`, so "complete" is never
  read as "flawless."
- `ARCHITECTURE.md`'s section 10 ("Documentation set") and section 11
  ("Phase plan") still phrased in future/present tense as if phases were
  still being written - updated to past tense reflecting actual
  completion, and to list `PERFORMANCE.md`/`KNOWN_ISSUES.md` among the
  docs produced.
- `DEPLOYMENT.md`'s Docker Compose section still said "Expect, once every
  phase through Phase 24 is complete..." and "As of Phase 1, all 10
  services build and start... but carry no business logic yet" - both
  literally true only through Phase 1/24 and misleading by Phase 26 (26
  phases of real business logic now exist in every service) - rewritten
  to state the real, current, tested functionality plainly, while
  keeping the honest "docker compose up --build itself has still never
  been run to completion" caveat intact and undiminished.
- A broken cross-reference: `DEPLOYMENT.md` cited "`ARCHITECTURE.md`
  section 3/31" - `ARCHITECTURE.md` has 11 sections total, no section 31
  exists anywhere; fixed to "section 3" (the actually-correct reference).
- `BUILD_STATE.json`'s own `environment_verified.docker` field still read
  Phase 20's original "daemon cannot start" finding as the current
  status, even though Phase 24 (and this session, again, directly -
  `dockerd` is genuinely running right now) established the daemon does
  start and registry access is the real, remaining blocker - fixed to
  the current, accurate status while explicitly keeping Phase 20's
  original finding on record as real for that session, not overwritten
  as if it were wrong.

No other doc (`API.md`, `DATA_FLOW.md`, `ML_MODEL.md`, `SECURITY.md`,
`DEMO_SCENARIOS.md`, `PERFORMANCE.md`) needed a correction - checked, not
assumed clean.

**Created `KNOWN_ISSUES.md`** - the final, consolidated, honestly-labeled
list of every real open item accumulated across 26 phases, pulled from
`TROUBLESHOOTING.md`, `SECURITY.md`, `ML_MODEL.md`, `PERFORMANCE.md`, and
`BUILD_STATE.json`'s own `known_errors`, organized into four categories
(fraud-detection accuracy, security hardening, performance, infrastructure/
deployment) rather than left scattered phase-by-phase. Historical findings
that a later phase's re-investigation superseded (Phase 20's "Docker
daemon cannot start" vs. Phase 24's "daemon works, registry access is the
real blocker") are kept and explicitly labeled as historical, not deleted
or silently merged into the current finding as if no discrepancy ever
existed. Closes with an honest "what a team picking this up for real
production use needs first" summary, naming the actual root cause behind
every remaining infrastructure item: no sandbox this project has been
built in has ever had real internet/registry/cloud access.

`VERSION` in `central-fraud-intelligence/src/config.py` **not bumped** -
no `src/` changes this phase, same reasoning as Phase 12/13/21/24/25.
Updated `README.md`, `ARCHITECTURE.md`, `DEPLOYMENT.md`, `TESTING.md`
(no new section needed - Phase 25's section already covers this session's
fresh re-run methodology; the numbers above are this phase's own,
separately re-executed confirmation of the same counts), `TODO.md`,
`BUILD_STATE.json`. Committed, checkpoint updated.

**Project status: all 27 planned phases (Phase 0 - Phase 26) are now
complete.** This concludes the build plan this project has followed since
Phase 0. `KNOWN_ISSUES.md` is the honest, final word on what remains open
for whoever continues this project next.

## Phase 27 — GNN live-inference collapse fixed for real, plus a requested feature set beyond the original 27-phase plan (admin fraud-report detail view, responsive UI redesign, real document upload)

### Scope and why this phase exists

Phase 26 closed the original 27-phase plan; this phase is new work the
user asked for afterward, in two parts:

1. Fix the Phase 20 GNN live-inference collapse (`KNOWN_ISSUES.md` item
   A.1), which Phase 20 had explicitly left open as out of scope.
2. A UI/product request: make all four loan platforms' frontends
   responsive, show fraud status/risk level/explanation, accept real
   document uploads, and — specifically — make clicking any application
   in the admin dashboard open a detail view showing the full
   application alongside its fraud report, risk level and explanation.
   The user supplied a reference Angular UI (a separate, unrelated loan
   management system zip) as a visual/UX reference only; the actual
   implementation is a from-scratch responsive React redesign of the
   existing per-platform frontends, not a port of that reference app.

### Part 1 — GNN fix (central-fraud-intelligence)

Root-caused and fixed the collapse documented in `ML_MODEL.md`'s Phase 20
section and `TROUBLESHOOTING.md`. Cause: `src/gnn/features.py` encoded a
node's `submittedAt` timestamp as an offset from a fixed absolute
calendar epoch; every Phase 12 synthetic training example's offset fell
inside a bounded range by construction, so a live request scored at real
wall-clock "now" produced an offset far outside that range - unbounded
out-of-distribution extrapolation, not a subgraph-shape or graph-size
effect as originally hypothesized. Fixed via `_reference_epoch()` (a
graph-relative reference timestamp computed from the application nodes
actually present, not a fixed calendar date) feeding a rewritten
`_scaled_timestamp()`, with a defensive clamp to the empirically-measured
training range (`MAX_TRAINED_SCALED_TIMESTAMP = 5.0`). No retraining -
`models/gnn_checkpoint.pt` is byte-identical to the Phase 13 checkpoint.

A first-attempt fix (clamp the old absolute-epoch feature to its measured
training max, without making it relative) was tried and genuinely failed
- live score stayed `0.000` even clamped - because the model had
specifically learned to read large absolute offsets as "a genuine
reapplicant returning months later," not as generically out-of-range.
This negative result is recorded, not hidden, in `TROUBLESHOOTING.md`'s
Phase 27 section, along with the final fix and full verification detail.

**Verified, real commands, this session:** `pytest tests/test_gnn.py` ->
21/21 passing; full `pytest tests/` -> 265/265 passing; offline evaluation
accuracy unchanged at 73.64% (confirms the fix is a no-op for
`evaluate_fraud_detection.py`'s whole-graph replay method, which never hit
this bug); direct reproduction of the exact Phase 20 scenario now scores
`1.000` (was `0.000`); a rapid-vs-spaced-apart live timing check confirms
the model still correctly distinguishes timing patterns under real live
conditions (`1.000` vs `0.144`), i.e. the fix didn't just make everything
score `1.0`. Documented in `TROUBLESHOOTING.md` (new Phase 27 section),
`KNOWN_ISSUES.md` (item A.1 rewritten from "unresolved" to "fixed"), and
`ML_MODEL.md` (an update appended directly under its Phase 20 section).

### Part 2 — backend: per-application fraud report endpoint (all 4 platforms)

Added `GET /api/v1/admin/applications/:applicationId` to every platform,
identically: `adminService.getApplication()` (404 via `AdminError` if
unknown) joined with `fraudAlertService.findForApplication()` (matches
this platform's own received `FraudAlert` records by
`triggeringApplicationId` or by `affectedApplications[].applicationId` -
note `affectedApplications` is an array of `{applicationId, platformId}`
objects, not plain ID strings, confirmed against `incidents.py`'s
`derive_affected_applications()` and the existing `fraudAlerts.test.js`
fixture before shipping) into a new pure function `buildFraudReport()`
(risk level via `riskLevelForScore()`: >=0.85 CRITICAL, >=0.65 HIGH, else
MEDIUM; unflagged applications get a default `{flagged:false,
riskLevel:"LOW", explanationSummary:"No cross-platform fraud signals..."}`).
Byte-for-byte identical logic across platforms a/b/c/d (only each
platform's own field/validation differences elsewhere are preserved).

**Verified:** `npm test` per platform, all passing and all net +3 tests
(new `describe("GET /admin/applications/:applicationId")` block: 404
unknown, unflagged report shape, flagged report with CRITICAL risk/score/
explanation/incidents array) - platform a 83/83, b 84/84, c 86/86, d
85/85, **338/338 total**, re-confirmed again at the very end of this
phase after the frontend work below, with no regressions.

### Part 3 — frontend: responsive redesign, fraud-aware admin detail view, real document upload (all 4 platforms)

Built once on platform A (QuickFund), verified there, then replicated to
B/C/D exactly the way the backend was replicated in earlier phases: diff
platform A against B/C/D first to confirm which files are brand-neutral
(copied byte-for-byte) versus genuinely platform-specific (edited with
the platform's own fields preserved), never a blind overwrite.

New shared design system, `frontend/src/styles.css` (new file, identical
across all 4 platforms - brand color comes from CSS custom properties
`--brand`/`--brand-dark`, set once from each platform's own existing
`brand.js` in `App.jsx`, so one stylesheet serves 4 different accent
colors): responsive app shell/header/nav, cards, buttons, form fields,
status/risk badges, a stat-card grid, a filter toolbar, a fully
responsive applications table (`.app-table-wrap`/`.app-row`, grid on
desktop, stacked single-column cards with `data-label`-driven field
labels on screens <=760px), a fraud-report card with a risk meter, a
document dropzone, and detail-page layout helpers - all collapsing
correctly at the two breakpoints (760px, 600px) it defines.

Rewrote `components/ui.jsx` (identical across all 4 platforms) to add:
`RiskBadge`, `StatCard`, `KVGrid`/`KV`, `SectionHeading`, `Spinner`,
`BackLink`, `EmptyState`, and the two centerpiece components -
`FraudReportCard` (renders `riskLevel`/`finalRiskScore`/
`explanationSummary`/`primaryDriver` plus a per-incident breakdown with
affected platforms/applications/acknowledged state, fed directly by the
new backend endpoint's `fraudReport` object) and `FileUpload` (real
multi-file drag/drop upload - reads actual file bytes via `FileReader`,
base64-encodes them, and calls back with `[{name, content, size}]`,
exactly the shape `POST /api/v1/applications` already accepted, so **no
backend change was needed** to move from a fake filename-only "upload" to
genuine file content being hashed - consistent with this project's
existing data-minimization design: only the SHA-256 hash is ever
persisted server-side, raw bytes never are, and `FileUpload` says so in
its own hint text).

New page `pages/AdminApplicationDetail.jsx` (identical across all 4
platforms) - the specific feature requested: clicking an application now
navigates here and shows applicant/loan/document detail, approve/reject/
request-review actions (moved here from the old dashboard), repayment and
review-history sections, and the `FraudReportCard` fed by the new
`GET /admin/applications/:id` endpoint - risk level, score, explanation
and primary driver, per application, exactly as asked.

Rewrote `pages/AdminDashboard.jsx` (identical across all 4 platforms) -
now a stat-card summary (total/awaiting-action/approved/rejected) plus a
responsive, clickable applications table; all per-application action
buttons moved to the new detail page, since actions now happen after
seeing the fraud report rather than blind from a flat list.

Rewrote `pages/ApplyLoan.jsx` per platform (kept each platform's own
extra loan fields - B's `institutionName`/`courseName`, C's
`BRAND.maxAmount`/`maxTenureMonths` ceiling copy, D's `merchantName`/
`productCategory` - and swapped only the old single-text-field fake
document field for the new `FileUpload` component and its real
`documents` array state/payload wiring).

Rewrote `App.jsx` (identical across all 4 platforms) for the new
class-based `styles.css` shell, added a `useBrandTheme()` hook (sets the
CSS custom properties + `document.title` from each platform's existing
`brand.js` - no changes needed to any `brand.js`), and added the new
`admin-detail` view/route passing `{applicationId}` through to the new
page. Minor centering/`narrow` tweaks applied consistently across all 4
platforms' `Login`/`Register`/`AdminLogin`/`ApplicationStatus`/
`LoanHistory`/`Repayments` pages (no logic changes).

**Verified, real commands, this session:** `npm run build` (vite) run
for real on all 4 platforms' frontends after `npm install` - all 4 build
with zero errors (`✓ 44 modules transformed` each, ~169KB JS / ~9KB CSS
gzipped output each). Diffed every platform's `frontend/src/` tree
against platform A's afterward to confirm the only remaining differences
are the intentional per-platform ones (`brand.js`, `api.js`'s one
cosmetic doc-comment line naming the platform, `AdminLogin.jsx`'s
`PageTitle` text, `ApplyLoan.jsx`'s extra fields, `ApplicationStatus.jsx`/
`LoanHistory.jsx`'s extra display fields for B/D) - not a stray, unnoticed
divergence. Backend test suites re-run after all frontend changes to
confirm zero backend regression: 338/338 passing across all 4 platforms
(same counts as Part 2, confirming the frontend work touched nothing
backend-side).

**What was NOT verified this phase, honestly:** no live `docker compose
up --build` was run in this sandbox (same long-standing sandbox
constraint recorded since Phase 1/24/25/26 - this environment cannot
reach the container registries the compose stack's base images require).
So the actual served-by-Express-from-`dist/`-in-a-running-container
experience, and a real browser click-through of the new admin detail flow
against a running stack, have not been re-verified after this phase's
changes - only `npm run build` (compilation) and the backend/GNN
automated test suites (338 + 265 = 603 tests) were actually executed.
The user needs to rebuild (`docker compose up --build`) their own running
stack to see this phase's changes live, exactly as after every prior
phase's code changes.

`VERSION` in `central-fraud-intelligence/src/config.py` not bumped for
the GNN fix - it's a bugfix to feature encoding, not a new model version;
the existing `models/gnn_checkpoint.pt.meta.json` remains accurate since
the checkpoint itself is unchanged. Updated `TROUBLESHOOTING.md`,
`KNOWN_ISSUES.md`, `ML_MODEL.md`, `PROGRESS.md` (this section),
`BUILD_STATE.json`.

## Phase 28 — distinct per-platform seed data, and a real central-fraud-intelligence admin dashboard (overview stats + a live GNN/rule scoring visualization)

### Scope

Two more user-requested items, found after Phase 27's Docker rebuild was
verified working end to end on the user's own machine:

1. All four loan platforms' `scripts/seed.js` seeded the exact same 5
   applicant identities (same names/emails/phones/addresses), making
   every platform's admin dashboard look identical apart from loan
   purpose text.
2. `central-fraud-intelligence` had no real landing UI - `GET /` still
   returned the raw Phase-1 JSON message, and while a real Jinja2 admin
   dashboard already existed from Phase 19 (`/dashboard/*`, login +
   incidents + notifications + a plain-table graph explorer), it had no
   overview/stats page and nowhere that showed how a single application's
   GNN and rule-engine scores actually combined into its decision - the
   user explicitly asked for "a beautiful UI... dashboard showing the
   number of applications processed and the other details and it should
   also show the working of GNN and central fraud intelligence."

### Part 1 — distinct seed data (all 4 loan platforms)

Root cause, confirmed by diffing all four `seed.js` files: every
platform's `USERS` constant was a byte-for-byte copy of platform A's
(same 5 names, emails, phone numbers, addresses), directly contradicting
that file's own docstring ("Phase 6 seed data is deliberately independent
per platform - no shared identifiers across platforms"). Only the `loan`
object per application actually differed (purpose/amount and each
platform's own required fields - `institutionName`/`courseName` for B,
the micro-loan ceiling for C, `merchantName`/`productCategory` for D).

Fixed by giving each platform its own distinct 5-person applicant pool
and its own phone-number prefix range, so no seeded identity can
accidentally collide across platforms: A keeps `9876500xxx` (Bengaluru/
Hyderabad/Chandigarh, unchanged), B gets `9876510xxx` (Pune/Delhi/
Chennai/Kolkata/Ahmedabad, matching its education-loan focus), C gets
`9876520xxx` (Mumbai/Jaipur/Lucknow/Indore/Nagpur, matching its
micro-loan focus), D gets `9876530xxx` (Kochi/Surat/Bhopal/Patna/
Coimbatore, matching its point-of-sale-financing focus). The deliberate
cross-platform identity overlaps this project's fraud-detection demo
depends on were never this file's job in the first place - they come
from `seed-data/scenarios/scenarios.json` replayed through the central
ingestion API, or from a live manual demo (same person applying on two
platform UIs) - both untouched by this fix.

**Verified:** `npm test` re-run on all 4 platforms after the change -
still 338/338 passing (no test hardcoded a seeded name/email, confirmed
by grep before editing). Cross-checked the new data itself: all 20
seeded identities (5 per platform × 4) have distinct emails and phone
numbers, no accidental overlap.

### Part 2 — central-fraud-intelligence: overview stats + GNN/rule scoring visualization

Investigated before building anything (this project's own "research
first" discipline): `src/web/dashboard.py`'s existing Phase 19 dashboard
was real and working, just incomplete for what the user was now asking -
`GET /dashboard/` jumped straight to the incidents list with no
aggregate view, and there was no page showing one application's actual
GNN-score/rule-score/final-score breakdown (only the graph explorer's
plain node/edge tables existed at that level of detail). `GET /api/v1/
metrics` (Phase 19) already computed every aggregate count needed; the
GNN/rule breakdown needed was already exactly what `src/explainability/
explain.py`'s `generate_explanation()` returns (`technical.gnn.*`/
`technical.rules.*` with configured/effective weight and weighted
contribution per signal) - both reused directly, no re-derived scoring
logic anywhere.

Built two new routes:
- `GET /dashboard/overview` - the new landing page (`GET /dashboard/`,
  login, and login-submit all now redirect here instead of
  `/dashboard/incidents`). Stat cards (applications processed, fraud
  detected, cleared, still-processing, notifications sent, graph size),
  a "how central fraud intelligence decides" explainer showing the real
  `GNN_WEIGHT`/`RULE_WEIGHT`/`FRAUD_DETECTED_THRESHOLD` constants and
  whether a checkpoint is actually loaded this run, and a "recently
  processed applications" table (new `PostgresEventRepository.
  list_recent()` in `src/ingestion/repository.py`) linking into the
  route below.
- `GET /dashboard/applications/:applicationId` - "the working of the
  GNN" for one real application: every event received for it, each
  rendered as a scoring-pipeline visualization (a GNN-score bar, a
  rule-score bar with its triggered rules as chips, a combined
  final-risk-score bar with a marker at the real fraud-detected
  threshold) plus the same human-readable explanation and caveats
  `GET /api/v1/applications/:id` already returns.

Also gave the whole dashboard (all pages, not just the two new ones) a
real visual redesign - `src/web/templates/base.html`'s inline CSS
rewritten with a proper color system, card shadows, a sticky gradient
header with active-nav-link highlighting, styled score/status badges,
and a responsive stat-card grid - replacing the flat, unstyled Phase 19
version, without touching any existing template's Jinja logic (only
`incidents.html`'s/`notifications.html`'s context dicts gained an
`active_nav` key for nav highlighting).

**Verified, real commands, this session:** added 6 new tests to
`tests/test_dashboard.py` (auth-required checks, real stat/score
assertions against rows seeded directly into the real Postgres test
database via the repository layer, an empty-database case, and an
unknown-application case) and updated the 2 existing tests whose
asserted redirect target deliberately changed
(`/dashboard/incidents` → `/dashboard/overview`). Full
`central-fraud-intelligence` suite re-run: **271/271 passing** (was
265 after Phase 27 - the 6 new tests, zero regressions). All 4 loan
platforms' backend suites re-run unaffected: still 338/338. Rendered
both new pages through a real `TestClient` request with seeded data and
screenshotted them with a headless Chromium (desktop 1280px and mobile
390px viewports) to visually confirm the redesign actually renders
correctly - not just that template strings are present - and caught and
fixed one real issue this way (the new applications table had no
`min-width`, so `overflow-x: auto` never actually engaged and columns
squished illegibly narrow on a phone-width viewport; fixed with a
`640px` table `min-width` so the card scrolls horizontally instead).

**What was NOT verified this phase:** the same standing sandbox
constraint as every prior phase - no live `docker compose up --build`
was run here, so the actual rendered dashboard has not been re-checked
inside a running Docker container in this sandbox, only via FastAPI's
`TestClient` and static HTML screenshots. The user needs to rebuild
(`docker compose up --build`) to see this phase's changes live.

Updated `API.md`'s Dashboard section and `central-fraud-intelligence/
README.md`'s dashboard paragraph and file-tree comment. `VERSION` in
`central-fraud-intelligence/src/config.py` not bumped - presentation/
aggregation only, no scoring or decision logic changed. `PROGRESS.md`
(this section), `BUILD_STATE.json` updated.

## Phase 29 — central-fraud-intelligence: `/` now redirects to the dashboard instead of returning raw JSON

**User report**: after Phase 28 shipped the overview/application-detail
dashboard pages, the user reported still seeing the raw JSON discovery
payload when they checked the service - pasting the exact `GET /`
response. Root cause: Phase 28 added a real dashboard landing page at
`/dashboard/overview`, but never touched `GET /` itself (the bare
`http://localhost:5000/` URL a person actually types into a browser),
which still returned the same Phase 1 JSON blob it always had. The
dashboard existed; the one URL a human would try first just didn't point
at it.

**Fix** (`central-fraud-intelligence/src/main.py`):
- `GET /` now returns `RedirectResponse("/dashboard", status_code=303)`
  instead of the JSON payload. `/dashboard` (`dashboard_root()` in
  `src/web/dashboard.py`, unchanged) redirects again to `/dashboard/login`
  or `/dashboard/overview` depending on whether the request already
  carries a valid `central_admin_token` cookie - so a fresh browser visit
  to the bare service URL now lands on either the login form or the real
  stats dashboard, never JSON.
- The JSON discovery payload itself was not deleted - it's byte-identical,
  just moved to a new `GET /api` route, for any machine caller that
  actually wants it.
- `API.md`'s Central Fraud Intelligence table updated: `GET /` and
  `GET /api` rows added/changed.

**Tests**: new `tests/test_main.py` (4 tests) - `GET /` redirects (303) to
`/dashboard`; following that redirect chain with a fresh (unauthenticated)
client lands on the real login form HTML; `GET /api` returns the same JSON
shape `GET /` used to; `/health` and `/api/v1/health` unaffected. No
existing test asserted on `GET /`'s old JSON body (grepped first to
confirm), so nothing needed updating for the behavior change itself. Full
suite re-run: **275/275 passed** (271 before this phase + 4 new), local
Postgres 16 (`central_fraud_intelligence_test`).

**Honest scope note**: this phase is routing only - no new dashboard
page, no new business logic, no changes to the loan platforms. It fixes
exactly the gap the user's second report pointed at (the bare service URL
still being JSON), and is step 1 of the larger set of changes requested
in the same message (loan-platform profile/application separation +
repayment gateway, a less trigger-happy rule engine for coincidental
cross-platform matches, and retraining the GNN checkpoint on a richer
synthetic dataset) - those are tracked as separate, not-yet-started
phases, confirmed with the user to be done staged rather than all at
once. `VERSION` in `central-fraud-intelligence/src/config.py` not bumped
(routing change only). `BUILD_STATE.json` updated.

## Phase 30 — all 4 loan platforms: separate Profile from loan application, surface the (already-existing) repayment gateway

**User report** (same message that also asked for the Phase 29 fix, tighter
fraud rules, and a retrained GNN): "the profile must be shown seperately
and loan application should be seperated from user profile and add
repayment gateway."

**Investigated before writing code**: read every platform's `ApplyLoan.jsx`
and confirmed the complaint - name/email/phone/address were plain form
fields re-typed on every application, no `Profile` page existed anywhere,
and the `User` model had no `address` field at all (address lived only on
each `Application.applicant` snapshot). Separately, `Repayments.jsx` +
`GET /api/v1/repayments` + `POST /api/v1/repayments/:id/pay` already
existed on all four platforms, already linked in the nav, already labeled
"(demo)" - a real simulated repayment feature, not a gap. So this phase
is genuinely two different kinds of fix: build the missing profile
separation, and confirm + lightly enhance the repayment feature that
already existed but the user apparently hadn't noticed.

**Full detail (endpoints, files touched, test counts, honest scope
note) is in `API.md`'s new "Profile / loan-application separation +
repayment gateway visibility" section** - not duplicated here. Summary:
new `GET`/`PATCH /api/v1/profile` on all 4 backends, a new `Profile.jsx`
page on all 4 frontends, `ApplyLoan.jsx` rewritten on all 4 to read the
profile instead of asking for it again (and gate submission on having one
completed), `seed.js` fixed on all 4 to actually persist the seeded
address (a real regression this phase would otherwise have introduced -
caught before shipping, via a new seed-data regression test), and
`Repayments.jsx` gained a summary strip (outstanding balance, paid so
far, next due date) on all 4.

**Tests**: 9 new backend tests + 1 seed regression test per platform.
Full suite: A 92/92, B 93/93, C 95/95, D 94/94. All 4 frontend `vite
build`s succeed. Not verified against a live browser/Docker/MongoDB in
this sandbox (same honest limitation as every other loan-platform phase
here - see `TESTING.md`).

**Still pending** (confirmed with the user as staged, separate work):
tightening `central-fraud-intelligence/src/fraud_detection/rules.py` so
a single coincidental cross-platform match doesn't alone decide
FRAUD_DETECTED, and retraining the GNN checkpoint on a richer synthetic
dataset. `BUILD_STATE.json` updated.

## Phase 31 — rule engine tightened, GNN retrained on a larger/debiased dataset, real before/after measurement

**User report**: "continue with stage 3 and 4" - stage 3 is the rules.py
tightening deferred from Phase 30, stage 4 is the GNN retrain. Full
technical detail, methodology, and every real number below is in
`ML_MODEL.md`'s new Phase 31 section - this entry is the summary.

**Stage 3 (`src/fraud_detection/rules.py`)**: added
`TWO_PARTY_TOKEN_SCORE_CAP = 0.35` - when every token-edge signal
(phone/email/name/address) fired for an application traces back to the
same single other borrower, their combined contribution is capped
rather than summed without limit, since multiple matching fields
between the same two applications are correlated evidence for "one
real person, two applications," not independent fraud signals. Does
not apply when signals point to different other borrowers (a real
multi-party pattern), and doesn't touch the document-hash or hub-bonus
signals. `explain.py` now adds an explicit caveat whenever the cap
fired. 18 new/updated test assertions; verified this stage alone had
**zero measured effect** on `scripts/evaluate_fraud_detection.py`'s
output against the old checkpoint (diagnosed why: the old GNN score
was saturated near 0/1 and, at `GNN_WEIGHT=0.6`, dominated the
decision regardless of the rule term) - the honest finding that made
Stage 4 necessary rather than optional.

**Stage 4 (GNN retrain)**: fixed a real platform-hardcoding training
bias Phase 21 had evidenced but not confirmed (`_legitimate_repeated_
application[_three_times]` now randomize their platform instead of
hardcoding `PLATFORMS[1]`/`PLATFORMS[2]`); regenerated the scenario
dataset at `--instances-per-pattern 6` (51→102 scenarios, 129→258
applications, re-verified against real Postgres); added an L2
`--weight-decay` flag to `training/train.py` and retrained with
`--weight-decay 1e-4 --dropout 0.2`. Caught and fixed a real
evaluation-methodology bug mid-phase: `generate_scenarios.py`'s
verification step leaves the graph database non-truncated on exit,
which had silently inflated the first post-retrain evaluation run to a
suspicious 100% - fixed by truncating before every real eval run
that follows.

**Real, measured result** (both old and new numbers are in
`ML_MODEL.md`, from `scripts/evaluate_fraud_detection.py` runs on a
freshly truncated database): on the large-scale 690-application FPR
dataset (Phase 21's own stress test), hard-negative accuracy went from
55.6% to **100%**, and the specific pattern Phase 21 flagged as the
platform-bias victim (`legitimate_repeated_application_three_times`)
went from 0/45 correct to 45/45. On the main dataset, the interleaved/
isolated accuracy gap (the documented "early-arrival blind spot")
closed entirely and precision improved (95.1%→98.0%), at the cost of a
small recall drop (72.2%→69.4%) traced to first-arrival two-party
SUSPICIOUS applications scored before their matching pair exists in
the graph - a pre-existing structural limitation, not a new
regression. A residual GNN overconfidence issue remains on a minority
of isolated genuine `normal_genuine_baseline` applications
(33.3%→80.0%, not 100%) - reported as unresolved, not rounded up.

**Tests**: full backend suite 277/277 passed (updated 4 hardcoded
dataset-size assertions in `tests/test_gnn.py` to match the
regenerated dataset's real counts). `models/gnn_checkpoint.pt/.meta.
json`, `models/baseline_metrics.json`,
`models/hybrid_scoring_eval.json`, and
`models/hybrid_scoring_eval_fpr.json` all regenerated for real this
phase.

## Phase 32 — `docker compose up --build` actually run for real by a user for the first time; two real Dockerfile bugs found and fixed

Every prior phase's Docker section carried the same honest caveat: this
sandbox's container registries are blocked, so `docker compose up
--build` had never been run to completion by this project, in any
sandbox, ever. Phase 31's delivered zip changed that — the user
downloaded it, ran it on their own Windows machine with normal registry
access, and it genuinely built and started all 5 images/10 containers
for the first time. In the process, they hit and reported two real bugs
this project's own untested Dockerfile had been carrying, both fixed
this phase (full detail in `KNOWN_ISSUES.md` item 14):

1. `central-fraud-intelligence/Dockerfile` copied `src`, `models`, and
   `scripts` into the image but never `datasets` or `training` -
   `scripts/generate_scenarios.py` (and, unnoticed until now, `training/
   train.py`, `training/baseline.py`, `scripts/evaluate_fraud_
   detection.py` too) failed inside the running container with
   `ModuleNotFoundError: No module named 'datasets'`. Fixed by adding
   `COPY datasets ./datasets` and `COPY training ./training`.
2. `.dockerignore` separately excluded `datasets` wholesale, which would
   have silently defeated fix #1 even after adding the `COPY` line (the
   directory would never have reached the build context at all). Fixed
   by removing that exclusion.

Also documented, not fixed (a real design limitation rather than a
one-line bug): those dev-tooling scripts compute their file paths
assuming the full multi-service repo layout on disk, which doesn't
match the deployed container's layout (only `central-fraud-
intelligence`'s own contents exist under `/app`, and `seed-data/` was
never intended to ship inside that image). They were always meant to be
run during development, before `docker build` - not inside the running
container. Guided the user to the actually-correct way to seed demo
data and see real cross-platform fraud detection instead: each
platform's own `node scripts/seed.js`, `createAdmin.js` for admin
accounts on all 5 services, then applying for loans with an overlapping
phone/email across two platforms through the real running frontends and
checking the central dashboard.

An updated project archive with both fixes was delivered directly in
chat. No automated tests re-run this phase (Dockerfile/`.dockerignore`
changes only, outside the Python/Node test suites' scope; verified by
inspection - `git diff` of both files - not by rebuilding the image in
this sandbox, since the registry block here is unchanged).
