# SECURITY

Status: **Phase 9 complete** (tokenization, transient encryption, and
central-app admin JWT auth are real and tested — see below for file/line
references) and extended further in Phase 22 (dedicated security testing
pass). This file documents the *actual* implementation, not aspirational
design.

## Phase 7 addition: outbox delivery authentication (interim, not the final design)

Each loan platform's outbox delivery worker (`src/outbox/deliveryWorker.js`,
see `API.md`'s "Outbox mechanism" section) sends an `X-API-Key` header
(`CENTRAL_FRAUD_API_KEY`) and an `X-Idempotency-Key` header (the event's
`eventId`, a UUID) on every delivery POST to the central application. This
is a real, working mechanism today — a shared-secret header plus a
replay-safe idempotency key — but it is explicitly an interim measure:

- It authenticates the *platform* to the central app (a single shared key
  today, not yet a per-platform key with rotation — that refinement is
  deferred to Phase 9 alongside the rest of the key-management design).
- It says nothing about the *payload*. The outbox event body currently
  carries raw, non-tokenized applicant/loan fields (see `DATA_FLOW.md`,
  "Steps 1-2: real, as of Phase 7") because Phase 7's job was the
  delivery mechanics (retry, backoff, idempotency), not tokenization.
  Replacing those raw fields with HMAC-SHA256 tokens per the table below
  is still entirely Phase 9's job, unchanged by this addition.
- TLS termination for this traffic is still whatever Docker Compose's
  local network provides (not TLS-terminated, called out below) until a
  real deployment target is chosen — this header does not imply transport
  security beyond what's already documented.

Verified for real in each platform's `tests/outbox.test.js`: the header is
actually sent on every delivery attempt, against a real local HTTP server
(not asserted only against a mock).

## Phase 8 addition: the central app now actually checks that key

Phase 7 only verified that platforms *send* `X-API-Key`/
`X-Idempotency-Key`; nothing existed yet to receive and check them. As of
Phase 8, `central-fraud-intelligence`'s `POST /api/v1/applications/process`
(`src/security/auth.py`) really does: reject a missing/unrecognized
`X-API-Key` (401, `resolve_platform_from_api_key` returns `None`); reject a
key that authenticates a *different* platform than the event body's
`platformId` claims (403) - a real check against the configured
`CENTRAL_ACCEPTS_API_KEY_{A,B,C,D}` values, using `hmac.compare_digest` for
constant-time comparison rather than `==`; and reject an
`X-Idempotency-Key` that doesn't match the body's `eventId` (400). All
three are covered by real tests in
`central-fraud-intelligence/tests/test_ingestion.py`.

This remains the same interim mechanism described above, not the final
design: still one shared secret per platform (not rotated, not per-request
signed). As of Phase 9 (below), the payload itself is no longer stored as
received — the outbox event body is tokenized/encrypted before it is ever
sent.

## Phase 9 addition: tokenization, transient encryption, and admin JWT auth

**Tokenization** (`loan-platform-{a,b,c,d}/backend/src/tokenization/tokenize.js`,
mirrored in Python at `central-fraud-intelligence/src/tokenization/tokenize.py`):
each loan platform's `applicationService.js` now calls `tokenizeApplicant()`
*before* constructing the outbox event — the raw `applicant` object is never
placed in the outbox payload at all (verified in every platform's
`tests/outbox.test.js`, which asserts `payload.applicant` is `undefined` and
that raw PII strings never appear anywhere in `JSON.stringify(payload)`).
`tokenize(masterKey, purpose, rawValue)` is `HMAC-SHA256(subKey, normalize(purpose, rawValue))`
where `subKey = HKDF-SHA256(masterKey, salt=empty, info="quickfund-x-token-v1:"+purpose)`
— a distinct sub-key per purpose (`name`/`phone`/`email`/`address`/`device`/`bank`),
so a compromised sub-key for one purpose cannot be used to correlate tokens
of a different purpose. `masterKey` is `CENTRAL_HMAC_MASTER_KEY` — the exact
same value on all four loan platforms AND the central app (see
`docker-compose.yml`; **not** per-platform-suffixed), which is the design
this section previously left as "finalized in Phase 9": it must be identical
everywhere so that the same real-world phone number, computed on any of the
four independent platforms, hashes to the identical token — this is what
makes cross-platform entity resolution (Phase 10+) possible at all. `name`
tokenization was added beyond this table's original scope (person names are
also a correlatable identifier across platforms); `device`/`bank` purposes
are implemented and tested but currently unused (no such fields are
collected by any platform's forms yet).

**Transient encryption** (`.../tokenization/encrypt.js`, mirrored at
`.../tokenization/encrypt.py`): `encryptField(keySource, plaintext)` is
AES-256-GCM with a random 12-byte IV per call, returning
`{alg, iv, ciphertext, authTag}` (hex-encoded). Each loan platform encrypts
`{email, phone}` into `encryptedContact` on the outbox payload using
`CENTRAL_AES_KEY` (also shared identically across all five services). Only
the central app is positioned to ever decrypt it. Not yet wired into any
actual decrypt/investigate workflow (that's Phase 17+) — built and tested as
a verified primitive: round-trip, wrong-key-fails, and tampered-ciphertext-
fails (GCM auth tag) are all covered by real tests.

**Real cross-language parity, not just "both compile"**:
`central-fraud-intelligence/tests/test_tokenization.py`'s
`TestCrossLanguageParity` class shells out to the actual Node binary to run
the real `.js` tokenize/encrypt modules and asserts byte-for-byte agreement
with the Python implementation's output (skipped only if `node` truly isn't
on PATH). This is the check that would have caught a silent HKDF-parameter
or normalization mismatch between the two languages — the class of bug that
would otherwise break cross-platform fraud detection invisibly.

**Central-app admin JWT auth** (`central-fraud-intelligence/src/security/jwt_auth.py`,
`src/security/admin_repository.py`, `src/api/admin.py`): hand-rolled HS256
JWT using only stdlib `hmac`/`hashlib`/`base64`/`json`/`time` (no
`python-jose`/`PyJWT` dependency — same "avoid a heavy dependency for
something this small" reasoning the project applies to the planned
hand-rolled GraphSAGE GNN). `POST /api/v1/admin/login` verifies a bcrypt
password hash against the `admin_users` Postgres table and issues a 24-hour
token; `GET /api/v1/applications/:id` now requires a valid
`Authorization: Bearer <token>` (`require_admin` FastAPI dependency),
closing the auth gap Phase 8 explicitly left open. Admin accounts are
provisioned out-of-band via `scripts/create_admin.py` (no public register
endpoint — mirrors every loan platform's own `createAdmin.js` pattern).
Covered by `tests/test_admin_auth.py` (10 tests: correct/wrong/unknown
credentials, case-insensitive email, a genuinely-issued token reaching a
protected endpoint, expired/tampered/wrong-secret token rejection, plus two
tests of the `jwt_auth` module directly).

Test counts as of Phase 9: loan-platform-a 54/54, b 55/55, c 57/57, d 56/56
(each +13 tokenization tests over its Phase 8 count);
central-fraud-intelligence 38/38 (16 tokenization + 10 admin-auth + 12
carried-forward/updated ingestion tests, all against real local
Postgres/Node — see `TESTING.md`).

## Privacy rule (binding for the whole project)

The central application receives only the minimum information required for
fraud intelligence. Raw PII (Aadhaar, PAN, phone, email, bank account,
street address) **never leaves the loan platform that collected it** and is
**never written into the central graph or central database**. What crosses
the boundary is: HMAC-derived pseudonymous tokens (per identifier type),
non-identifying metadata (platform id, timestamps, loan-amount bucket,
document hash), and — only where strictly necessary and only transiently —
AES-256-GCM-encrypted fields that the central app validates and then
discards the plaintext of.

The "never written into the central graph" half of this rule is now a real,
checkable claim rather than only a stated intent: as of Phase 10, the
central graph (`central-fraud-intelligence/src/graph/`) exists, and
`GraphStore.apply_application_submitted()`/`apply_application_status_changed()`
build every node/edge exclusively from `event["applicantTokens"]` and
other already-tokenized/non-identifying fields — there is no raw applicant
field on the event payload for graph code to read even if it wanted to
(tokenization happens upstream on the loan platform, Phase 9). Verified
directly: `central-fraud-intelligence/tests/test_graph.py`'s fixtures are
built entirely from token strings, never raw PII.

Verified against real code as of Phase 9 (file references above); rows
marked "not yet collected" are implemented and tested but have no live
producer yet, since no platform form collects that field today:

| Data element | Where it lives | What crosses to central | How | Status |
|---|---|---|---|---|
| Password | platform DB only (bcrypt hash) | never | n/a | done |
| Aadhaar / PAN | platform DB only | never | n/a | not yet collected |
| Name | platform DB (raw) | HMAC-SHA256 token only | keyed hash, per-purpose key via HKDF | done (Phase 9 addition beyond original scope) |
| Phone number | platform DB (raw) | HMAC-SHA256 token only | keyed hash, per-purpose key via HKDF | done |
| Email | platform DB (raw) | HMAC-SHA256 token only | keyed hash, per-purpose key via HKDF | done |
| Device fingerprint | platform DB (raw) | HMAC-SHA256 token only | keyed hash, per-purpose key via HKDF | implemented, not yet collected |
| Bank account fingerprint | platform DB (raw) | HMAC-SHA256 token only | keyed hash, per-purpose key via HKDF | implemented, not yet collected |
| Street address | platform DB (raw) | HMAC-SHA256 token only (coarse, e.g. locality-level) | keyed hash, per-purpose key via HKDF | implemented, not yet collected |
| Email + phone (transient) | platform DB (raw) | AES-256-GCM ciphertext (`encryptedContact`), decryptable only by central app | symmetric encryption, random IV per event | done, not yet wired to a decrypt workflow (Phase 17+) |
| Document | platform DB (raw file) | SHA-256 content hash only | one-way hash, no reversible token needed | done (Phase 7) |
| Loan amount / product / timestamps | platform DB | as-is (not identifying on their own) | plain, over TLS (TLS itself still pending a real deployment target) | done |

## Why HMAC-SHA256, not plain SHA-256, for tokenization

A 10-digit phone number space (or similarly small identifier spaces) is
brute-forceable by hashing every possible value and comparing. Plain
`SHA256(phone)` is therefore reversible in practice. `HMAC-SHA256(secret_key,
normalized_value)` requires the attacker to also know `secret_key`, which is
never transmitted and is held only by the parties doing the hashing (every
loan platform and the central app, each holding the identical
`CENTRAL_HMAC_MASTER_KEY` value — see the Phase 9 section above for why it
must be identical rather than per-platform-derived). Different token
*purposes* (name vs. phone vs. email vs. device vs. bank vs. address) use
independently HKDF-derived sub-keys, so tokens cannot be correlated across
purposes even if one sub-key were ever compromised.

**Phase 11 consequence, worth stating explicitly:** the same avalanche
property that makes HMAC-SHA256 safe here also means entity resolution in
the graph (`ARCHITECTURE.md` section 6) can only ever be exact-token-match
— two inputs differing by even one character produce completely
uncorrelated digests, so no similarity/fuzzy comparison is possible on the
token values themselves without a different, weaker-privacy scheme (e.g.
locality-sensitive hashing, which trades away some of the brute-force
resistance this section relies on). This was evaluated and deliberately
not built in Phase 11 (`central-fraud-intelligence/src/graph/graph_store.py`'s
Phase 11 docstring) — `normalize()` in `tokenize.js`/`tokenize.py` already
absorbs the fuzziness that's safe to absorb *before* hashing (phone: strip
non-digits; email: lowercase; address/name: lowercase + collapse
whitespace); anything beyond that is out of scope for this project.

**Phase 14 consequence, checked and worth stating explicitly (not a new
control needed):** GNN inference (`src/gnn/inference.py`) never sees raw
PII. Its inputs are the already-tokenized graph (Phase 10, built from the
same HMAC tokens this section describes) and features derived purely from
graph structure and node type (`src/gnn/features.py` — degree counts,
node-type one-hot, platform id; nothing recomputed from a raw applicant
field). The trained checkpoint (`models/gnn_checkpoint.pt`, Phase 13) is
therefore not itself a store of raw PII, and adds no new secret or
key-management surface: no new API key, no new encryption key, nothing
beyond the file-system permissions already protecting `models/` (same as
any other app file). The one file-handling nuance worth noting: the
checkpoint's `.pt` weights are git-ignored (a binary training artifact,
not source — see `models/README.md`) while `gnn_checkpoint.meta.json` is
committed; the metadata file contains architecture/training-run details,
not weights or applicant data, so committing it does not leak anything
sensitive.

**Phase 15 consequence, same reasoning, also not a new control needed:**
the deterministic rule engine (`src/fraud_detection/rules.py`) and the
hybrid combination function (`src/fraud_detection/scoring.py`) operate on
the exact same already-tokenized bounded subgraph the Phase 14 GNN
consumes — never on raw PII, never on a decrypted `encryptedContact`
blob. `triggeredRules`, the new array returned by
`POST /api/v1/applications/process` and `GET /api/v1/applications/:id`,
contains only token values (already-HMAC'd, per this section), counts,
and time deltas — no raw applicant field is added to any response by
this phase. No new secret, key, or storage surface was introduced.

**Phase 16 consequence, also not a new control needed:** explanation
generation (`src/explainability/explain.py`) formats only fields already
present in the same response — `status`/`fraudScore`/`ruleScore`/
`triggeredRules`/`finalRiskScore` — so it introduces no new PII exposure
beyond what Phase 14/15's consequence notes above already cover; it adds
no new field to `GET /api/v1/applications/:id` that a caller couldn't
already derive from the fields that endpoint already returned. It is
also not persisted (see `ARCHITECTURE.md` section 8's explainability
subsection), so there is no new storage surface, git-ignore concern, or
key-management surface to note here either.

**Phase 17 consequence - unlike Phase 15/16, this genuinely IS a new
storage surface, but not a new PII-exposure one:** the new
`fraud_incidents` table (`src/incidents/repository.py`) persists the
same already-tokenized fields every response since Phase 9 has carried -
`applicationId`/`platformId` (internal routing identifiers, never raw
PII, per this section's scope since Phase 1), the four score fields,
`triggeredRules` (token values/counts/time deltas, per the Phase 15 note
above), and a snapshot of Phase 16's explanation (which itself only
formats those same fields). No raw applicant field, decrypted
`encryptedContact` blob, or new secret/key is ever written to it. Access
is gated the same way `GET /api/v1/applications/:id` already is - both
new endpoints (`GET /api/v1/incidents`, `GET /api/v1/incidents/:id`) sit
behind `require_admin`, not a new or weaker auth mechanism.

**Phase 18 consequence - a new storage surface on BOTH sides, and a new
outbound network hop, but no new secret or PII-exposure surface:** the
new `fraud_notifications` table (central,
`src/notifications/repository.py`) and each loan platform's new
`fraud_alerts` Mongo collection (`backend/src/models/FraudAlert.js`)
both persist the same already-tokenized/already-derived fields the
`fraud_incidents` row (Phase 17 note above) already carries -
`incidentId`, `applicationId`/`platformId` values, the four score
fields, `primaryDriver`, and the explanation summary snapshot. No raw
applicant field, decrypted `encryptedContact` blob, or new applicant PII
crosses the platform boundary that Phase 9's tokenization scheme didn't
already establish crosses it (the outbox has always carried tokens
outbound to central; this phase is the first time something flows back
from central to a platform, but it's the same already-tokenized shape,
not a new raw-PII path in either direction).

Authentication for this new outbound hop (central's `NotificationDeliveryWorker`
POSTing to `POST /api/v1/fraud-notifications/receive`) deliberately
introduces NO new secret: it reuses each platform's own
`CENTRAL_ACCEPTS_API_KEY_{A,B,C,D}` (central-side name) /
`CENTRAL_FRAUD_API_KEY` (platform-side name) - the exact same shared-key
value Phase 7/8 already established for the opposite direction
(platform → central), now also checked in reverse (central → platform).
This was a deliberate design decision, not an oversight - see
`ARCHITECTURE.md` section 8's "Fraud notifications" subsection - because
provisioning a second key per platform would double the key-management
surface (rotation, storage, `.env.example` documentation) for a
connection that already has a working, already-shared secret between
exactly the same two parties. The receiving check is a single equality
comparison (`providedKey === apiKey`, not `hmac.compare_digest` - unlike
central's own multi-platform `resolve_platform_from_api_key()`, a loan
platform has exactly one legitimate sender, so there's no set of
candidate keys to iterate and no timing side-channel between multiple
candidates to worry about; a constant-time comparison against the single
correct value would be a marginal hardening step, not yet done, tracked
alongside the other "planned controls" below rather than blocking this
phase). `GET /api/v1/admin/fraud-alerts` is gated the same way every
other admin route on that platform already is - `requireAuth` +
`requireRole("admin")`, not a new or weaker auth mechanism.

**Phase 19 consequence - a new cookie-based auth transport and a new
HTML surface, but the same underlying identity/secret, and no new PII
storage:** the dashboard (`src/web/dashboard.py`) does not introduce a
second authentication system. `/dashboard/login` checks credentials
against the exact same `admin_users` table and bcrypt comparison as
`POST /api/v1/admin/login`, and issues the exact same HS256 JWT (same
`CENTRAL_JWT_SECRET`, same `jwt_auth.encode`) - the only difference is
that it's set as an `HttpOnly`/`SameSite=Lax` cookie (`central_admin_token`)
instead of returned in a JSON body for a caller to carry in an
`Authorization` header itself. `HttpOnly` means client-side JavaScript
cannot read the cookie (irrelevant defense-in-depth here anyway, since
the dashboard deliberately ships zero client-side JS - see
`ARCHITECTURE.md` section 8), and `SameSite=Lax` limits the cookie being
sent on cross-site requests, a real (if modest, for an internal admin
tool with no other origin talking to it) CSRF-surface reduction. No CSRF
token is implemented beyond that - a stated, honest gap: the dashboard's
mutating routes (`POST /dashboard/incidents/:id/review`,
`POST /dashboard/notifications/:id/review`) rely on `SameSite=Lax`
alone, tracked below alongside this project's other planned-not-done
controls, not silently omitted.

The new `review_history` (incidents) and `acknowledged_by`/
`acknowledged_note` (notifications, and the mirrored `fraud_alerts`
fields on each loan platform) columns store the reviewing admin's own
email and a free-text note THEY wrote - not applicant PII, not a new
token/key, and not data that crosses the platform boundary at all (an
incident's `review_history` never leaves the central database; a
platform's own `fraud_alerts.acknowledgedBy` never leaves that
platform). This is the same class of addition as Phase 2's
`adminService.approve()`/`reject()` already recording `by`/`note` on a
loan application - an established, already-reviewed pattern in this
codebase, not a new privacy surface.

The graph-explorer endpoints (`GET /api/v1/graph/application/:id`,
`GET /api/v1/graph/entity/:id`) expose the SAME graph data
`GraphStore.get_relevant_subgraph()` has computed and stored since Phase
10/11 - node/edge data keyed on already-tokenized identifiers (the graph
never stores a raw applicant field, per that phase's own design), now
reachable over HTTP for the first time rather than only consumed
internally by the scoring pipeline. Access is gated by the identical
`require_admin` dependency every other admin-facing read endpoint uses -
no new or weaker auth mechanism, and no new PII exposure, since nothing
in the response was ever un-tokenized to begin with.

## Planned controls (status as of Phase 9, verified further in Phase 22)

- TLS-ready transport (documented for production in `DEPLOYMENT.md`; the
  local Docker Compose network is isolated but not TLS-terminated — this is
  called out explicitly, not glossed over) — **concretely designed as of
  Phase 25** (public ALB terminates TLS 1.2+ only, `ELBSecurityPolicy-
  TLS13-1-2-2021-06`, HTTP:80 redirects to HTTPS:443 with no plaintext
  listener — see `deploy/aws/terraform/alb.tf`) but **still pending an
  actual deployment target** — this sandbox has no AWS account, domain, or
  ACM certificate to exercise it against, so this remains NOT VERIFIED
  beyond the design itself
- JWT (HS256) authentication: each loan platform's own user/admin auth
  (Phase 2) — **done**; central app's own admin auth (Phase 9,
  `src/security/jwt_auth.py`) — **done**; platform→central service-to-service
  calls still use the Phase 7/8 shared-API-key mechanism above, not a JWT —
  **not planned to change**, the shared-key + idempotency-key design is the
  intended final mechanism for that specific hop. Phase 22 added a real
  test (`tests/test_security_hardening.py`) proving the hand-rolled
  `jwt_auth.decode()` also rejects a classic "alg: none" forged token -
  it has no code path that branches on the token's own claimed algorithm,
  so there was never really an "alg" attack surface here, but this phase
  is what first proved that rather than only asserting it.
- Role-based authorization (user vs. admin) enforced on every state-changing
  endpoint — **done** on each loan platform (Phase 2); central app has only
  one role (admin) so there is nothing further to distinguish there
- bcrypt password hashing — loan platforms via `bcryptjs` (Phase 2, a
  pure-JS implementation, same algorithm/cost-factor properties as native
  `bcrypt`, chosen to avoid a native-addon build step in the Docker image);
  central app via native `bcrypt` (Phase 9, `src/security/admin_repository.py`
  — a native build step is acceptable there since it's Python, not a
  frontend-adjacent Node image) — **done**
- AES-256-GCM for any sensitive payload field the central app must
  transiently decrypt/validate — **done** (Phase 9, see above; not yet
  wired into an actual decrypt workflow — Phase 17+)
- HMAC-SHA256 + HKDF for all cross-platform-matchable tokens — **done**
  (Phase 9, see above)
- Secrets sourced only from environment variables (`.env`, never committed;
  `.env.example` documents every variable with real comments) — **done**;
  as of Phase 25, production key *storage* is concretely designed (AWS
  Secrets Manager, one secret per sensitive `.env.example` value, read into
  each ECS task at runtime via `secrets:` — never baked into an image, see
  `deploy/aws/terraform/secrets.tf` and this file's Phase 25 addition
  below). Rotation *procedure* (who rotates what, how often, how a
  rotation propagates to running tasks without downtime) is still **not
  yet documented** — Secrets Manager supports automatic rotation via a
  Lambda function, but writing and testing that Lambda against a real
  secret is out of scope for a phase with no real AWS account to test it
  in; deferred, no fixed phase, same honest status as before, now with a
  concrete storage mechanism underneath it instead of an unspecified one
- Request validation (schema-level) on every ingress endpoint — **done**
  (Pydantic on the central app since Phase 8, per-platform validation
  middleware since Phase 2). Phase 22 adds real proof, not just
  inspection, that this validation is backed by genuinely parameterized
  queries all the way down: `tests/test_security_hardening.py` sends
  classic SQL-injection-style strings (`'; DROP TABLE ...; --`) as
  `applicationId`/`eventId` through the real ingestion endpoint and
  confirms they're stored and returned as inert, literal text - a real
  demonstration, not an assumption from reading the code (which Phase 22
  also did - see this file's grep-verified note below).
- Rate limiting on public-facing endpoints (login, application submission,
  central ingestion) — **not yet implemented**, no fixed phase assigned.
  Phase 22 confirmed this gap still exists (no `express-rate-limit`,
  `helmet`, or equivalent middleware anywhere in any loan platform's
  `package.json`, and no custom rate-limiting code) and made a deliberate
  decision not to add it this phase - see this section's Phase 22 note
  below for why.
- Audit logging with an explicit denylist (passwords, raw Aadhaar/PAN/bank
  account numbers, encryption keys, JWT secrets are never logged) —
  **done as of Phase 22** (`src/audit/repository.py`'s
  `PostgresAuditRepository`, wired into admin login success/failure,
  incident review, and notification review). The denylist is enforced
  inside the repository itself at write time (`DENIED_DETAIL_KEYS`,
  case/separator-insensitive), not left to call sites to remember, and
  `tests/test_audit_log.py` verifies it by actually sending a real
  password through a real login attempt and asserting the stored row
  never contains it - the verification this bullet has asked for since
  Phase 9's stub. Scope, stated plainly: this is an audit trail of
  specific admin *actions*, not a general request logger - see
  `src/audit/repository.py`'s module docstring for exactly what is and
  isn't covered.
- Data minimization: the central app's Postgres schema (Phase 8) has no
  columns capable of holding raw PII — **done and structurally enforced**,
  not only a runtime discipline (the `application_events.payload` JSONB
  column now only ever receives tokenized/encrypted payloads as of Phase 9)
- CSRF protection for the Phase 19 dashboard's mutating routes
  (`POST /dashboard/incidents/:id/review`,
  `POST /dashboard/notifications/:id/review`) — **not yet implemented**
  beyond the cookie's own `SameSite=Lax` attribute (see the Phase 19
  consequence note above); no fixed phase assigned, since this project's
  original 27-phase plan never named a dedicated CSRF-hardening phase for
  a surface that didn't exist before Phase 19. Phase 22 re-reviewed this
  gap and made a deliberate decision not to close it this phase either -
  see the Phase 22 note below.

## Phase 22: security testing - findings, fixes, and deliberate non-fixes

Phase 22's job was to actually test the security properties this file has
claimed since Phase 9, not just re-assert them, plus a full dependency
audit across all five services (the Phase 1 npm-audit finding below was
explicitly deferred to this phase). What follows is the honest record of
what that testing found, in three categories: things confirmed already
correct, things found broken and fixed, and things found imperfect and
deliberately left alone with a stated reason.

**Confirmed correct by direct testing (not just code review), all
verified in `central-fraud-intelligence/tests/test_security_hardening.py`
and `tests/test_admin_auth.py`:**
- Every SQL query in `central-fraud-intelligence/src/*/repository.py` uses
  parameterized placeholders (`%(name)s` / `%s`) for every value that can
  ever come from a request; the handful of f-strings in that code
  (`src/incidents/repository.py`, `src/notifications/repository.py`,
  `src/ingestion/repository.py`) interpolate only a fixed, hardcoded
  column-name constant, never request data - confirmed by grep across
  every `.execute(` call in `src/`, not assumed. A real SQL-injection-
  style probe string sent as `applicationId`/`eventId` is stored and
  returned as inert text (see the Planned-controls bullet above).
- The hand-rolled JWT (`src/security/jwt_auth.py`) rejects a forged
  "alg: none" token, an empty/malformed token, and a token with the
  wrong number of `.`-separated segments - `decode()` always computes a
  real HMAC-SHA256 signature and never branches on the header's own
  claimed algorithm, so there was no "alg confusion" surface to begin
  with, now proven rather than assumed.
- Admin login's "same error for unknown email and wrong password"
  property (Phase 9's anti-enumeration design) is genuinely
  indistinguishable at the HTTP layer - same status code, same response
  body, verified directly rather than by reading the comment that says
  so.
- The `X-API-Key` platform-authentication check
  (`src/security/auth.py`) already used `hmac.compare_digest` for a
  constant-time comparison - confirmed by reading the code; not
  re-tested for timing directly (timing-attack tests are inherently
  noisy and out of scope for this phase's tooling), but the
  constant-time-comparison discipline itself was verified present.

**Found broken and fixed this phase** (see each fix's own in-code
comment for the full reasoning - `loan-platform-{a,b,c,d}/backend/src/
services/authService.js` and `adminService.js`, identical across all
four platforms):
1. **NoSQL type-confusion in registration/login.** `userRepository.js`'s
   `findByEmail()` calls `email.toLowerCase()` directly and (in the real,
   Mongo-backed production repository) passes the result straight into
   `User.findOne({ email: ... })`. A JSON body like
   `{"email": {"$ne": null}, "password": "x"}` would reach that call with
   `email` as an object, not a string. `.toLowerCase()` throwing on a
   non-string happened to stop the malformed query from ever reaching
   Mongo (a lucky accident of the existing code - it would have surfaced
   as an unhandled-looking 500, not a clean validation error), but that's
   not a real defense. Fixed by rejecting any non-string
   name/email/password/phone outright, in the service layer, before
   anything touches a repository - closes the vector by construction,
   independent of what any repository implementation does with its
   inputs. Real Mongo is unreachable in this sandbox (the same standing
   limitation TROUBLESHOOTING.md has documented since Phase 1), so this
   was found by code review, not a live exploit against the real
   database - but the fix and its test (`tests/security.test.js`, all
   four platforms) are real and verified against the in-memory
   repository double every other test in this project already trusts.
2. **MongoDB query-operator injection in the admin applications list.**
   `GET /api/v1/admin/applications?status=...` passed `req.query.status`
   straight into `applicationRepository.listAll({ status })`, which the
   real Mongo repository turns into `Application.find({ status })`.
   Express's default query parser accepts nested query keys
   (`?status[$ne]=REJECTED` becomes `req.query.status = { $ne:
   "REJECTED" }`), so an unvalidated `status` was a real operator-
   injection vector - gated behind admin auth already (lower severity
   than an unauthenticated one), but still a real defect that could
   silently override an admin's own filter intent. Fixed by only
   accepting one of the platform's own known status values (reusing
   `VALID_TRANSITIONS`'s keys as the single source of truth, rather than
   a second, separately-maintained allowlist) - anything else is rejected
   with a 400 before it ever reaches a query. `tests/security.test.js`
   sends exactly the `?status[$ne]=REJECTED` shape described above and
   confirms it's now rejected, not silently accepted.
3. **The audit-logging gap** named in the Planned-controls bullet above -
   see that bullet for the fix.
4. **Two outdated Python dependencies with real, fixable CVEs**:
   `cryptography` 46.0.7 → 50.0.1 (7 distinct CVEs at the installed
   version, all in X.509 certificate-chain/PKCS7 code paths this
   application never calls - it only uses `cryptography` for AES-256-GCM
   and HKDF, per `src/tokenization/` - fixed anyway since the bump was
   free and zero-risk for this app's actual usage) and `python-multipart`
   0.0.26 → 0.0.32 (4 distinct CVEs, all in multipart/urlencoded form
   parsing - genuinely relevant here, since `src/web/dashboard.py`'s
   login and review forms are real `Form()`-parsed HTML `<form>` POSTs).
   Both are direct pins in `requirements.txt`; both bumps were verified
   against the full 258-test central suite (240 pre-existing + 18 new
   this phase) with zero regressions before being accepted, not just
   `pip-audit` reporting 0 afterward.

**Found and deliberately NOT fixed this phase, with a stated reason for
each:**
1. **`starlette` (a transitive dependency of the pinned `fastapi==0.115.6`)
   has 7 distinct CVEs at the installed 0.41.3** (`pip-audit`, run for
   the first time this phase - see `TESTING.md`'s Phase 22 section for
   the full command and raw finding). Fix versions range from 0.47.2 up
   to a `starlette` 1.x major version - meaning a real fix here isn't a
   patch bump, it's very likely a `fastapi` major-version bump too, which
   this project has never attempted and has no reason to believe is a
   drop-in replacement for every endpoint's behavior. Reviewed for actual
   relevance, not just counted: this app uses none of `StaticFiles`,
   `HTTPEndpoint`, or `FileResponse`/`Range` handling anywhere (confirmed
   by grep) - three of the CVE categories concern code paths this app
   never invokes. The two that ARE relevant (`request.form()` resource-
   limit bypass and large-multipart-file parsing, both touching
   `src/web/dashboard.py`'s Form() usage) are real, accepted risk for
   now. A `fastapi`/`starlette` major-version upgrade is exactly the kind
   of "large blast radius, every endpoint affected" change this project's
   established discipline (see `TROUBLESHOOTING.md`'s Phase 1 vite/esbuild
   entry for the same reasoning applied there) treats as its own
   dedicated, carefully-tested effort, not something to fold into an
   already-large security-testing phase. Flagged here, not silently
   left out - a future phase's explicit job, no fixed phase number
   assigned yet.
2. **Rate limiting** - confirmed still absent (no middleware, no custom
   code, on any of the four loan platforms or the central app). Not
   added this phase: every admin-facing endpoint already requires a real
   JWT, and the two genuinely public endpoints most worth rate-limiting
   (`POST /api/v1/auth/login` on each platform, `POST /api/v1/admin/
   login`) would need a real design decision (per-IP? per-account?
   what window, what response?) this phase didn't have a mandate to make
   unilaterally - adding a naive implementation just to have one would
   risk a worse outcome (a DoS vector of its own, or a false sense of
   protection) than stating the gap plainly, which is what this file has
   always done for exactly this kind of tradeoff.
3. **CSRF protection for the dashboard's mutating routes** - re-reviewed,
   still relies only on the login cookie's `SameSite=Lax` attribute (see
   the Phase 19 consequence note earlier in this file for why that's a
   real, if modest, mitigation for a single-origin internal admin tool).
   Not implemented this phase for the same reason as Phase 19's original
   deferral: this project's 27-phase plan never named a dedicated
   CSRF-hardening phase, and a real CSRF-token implementation touches the
   dashboard's session/template layer in a way that deserves its own
   focused pass rather than being squeezed into this already-large
   phase's scope.
4. **The `esbuild`/`vite` frontend dev-dependency finding** (originally
   documented at Phase 1, explicitly deferred to Phase 22) - re-run for
   real this phase across all four platforms' frontends (identical
   result on each, since all four `package.json`s pin the same
   `vite: ^5.4.11` range): now 2 advisories (1 moderate on `esbuild`,
   1 high on `vite`, the high one itself bundling 3 separate CVE
   descriptions - `npm audit`'s advisory data has grown since the range
   was first pinned, not a new vulnerability introduced by this project).
   `npm audit fix --force` would resolve it by installing `vite@8.3.0`,
   a breaking major-version change to the frontend build tool - not
   attempted this phase for the same "large blast radius, deserves its
   own dedicated pass" reasoning as the `starlette` finding above. Still
   accepted as before: this affects `npm run dev` only, not the
   production `vite build` output actually shipped.

Every finding above that resulted in a code change was verified against
the real, full automated test suite for its service, not merely inspected
by eye - see `TESTING.md`'s Phase 22 section for the exact commands and
counts.

## Phase 25 addition: production key management and TLS termination (design, not exercised)

Phase 25's scope was cloud deployment preparation (`DEPLOYMENT.md`), not a
new security-testing pass, so nothing above was re-run this phase. Two
items this file previously left open ("pending a real deployment target" /
"deferred, no fixed phase") now have a concrete design, referenced from
their bullets above:

- **Key storage**: AWS Secrets Manager, one secret per sensitive value in
  `.env.example` — `CENTRAL_HMAC_MASTER_KEY`, `CENTRAL_AES_KEY`,
  `CENTRAL_JWT_SECRET`, `CENTRAL_DB_PASSWORD`, and each loan platform's own
  `JWT_SECRET` / `CENTRAL_FRAUD_API_KEY` / DocumentDB password
  (`deploy/aws/terraform/secrets.tf`). Values are injected into each ECS
  task's environment at container start via the task definition's
  `secrets:` block (`deploy/aws/terraform/ecs.tf`) — never written into a
  Docker image, never present in Terraform state as anything other than a
  random placeholder (see that file's header comment for exactly how the
  placeholder-vs-real-value split works and why it's safe against
  `terraform apply` overwriting a real value later).
- **TLS termination**: a public ALB, TLS 1.2+ only
  (`ELBSecurityPolicy-TLS13-1-2-2021-06`), HTTP:80 redirects to HTTPS:443
  with no plaintext listener (`deploy/aws/terraform/alb.tf`).
- **IAM least privilege**: the ECS execution role's Secrets Manager grant
  is scoped to exactly the secrets this project's own containers read —
  not `secretsmanager:*` on `*` (`deploy/aws/terraform/ecs.tf`'s
  `read_secrets` policy document).

**Not claimed**: neither of these has been exercised against a real AWS
account (none exists in this sandbox — see `DEPLOYMENT.md`'s Cloud: AWS
section for the full, honest status). Secrets Manager's automatic-rotation
capability (a Lambda that rotates a secret on a schedule) was deliberately
NOT implemented this phase — writing one untested against a real secret
would be exactly the kind of unverified claim this file has avoided since
Phase 9; it remains a real, named, deferred item rather than a silently
assumed one.

## What will NOT be claimed

Per the project's own rule: this file will never claim "zero false
positives," "100% fraud detection," or unconditional security guarantees.
Any such claim, if it ever appears, will be scoped exactly to the specific
synthetic test suite that produced it (see `TESTING.md` / `ML_MODEL.md`).
