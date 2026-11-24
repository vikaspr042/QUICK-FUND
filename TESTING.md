# TESTING

Status: living document, updated every phase that adds tests. This file is
the single source of truth for "what is actually tested and passing right
now" — never aspirational. Anything not yet covered is listed under Not Yet
Tested, not silently omitted.

## How to run what exists so far

```
cd loan-platform-a/backend
npm install
npm test
```

As of Phase 2 (loan-platform-a / QuickFund): 34/34 core tests passing (6
suites — auth, application validation/ownership, admin status transitions +
repayment generation, repayment payment, JWT/authorization edge cases, and
one full end-to-end register→apply→review→repay flow). As of Phase 6, with
`scripts/seed.js` and its verification test added: 35/35 tests passing (7
suites). As of Phase 7, with the outbox event/delivery-worker suite added:
41/41 tests passing (8 suites). As of Phase 9, with the tokenization/
encryption suite added: 54/54 tests passing (9 suites). As of Phase 18,
with a new `tests/fraudAlerts.test.js` suite (14 tests) covering
`POST /api/v1/fraud-notifications/receive` (X-API-Key auth, required
X-Idempotency-Key, required body fields, redelivery idempotency,
standalone-mode with no repository wired) and `GET /api/v1/admin/
fraud-alerts` (admin auth, listing what's been received): 68/68 tests
passing (10 suites). As of Phase 19, with 5 new tests in the same file
covering `POST /api/v1/admin/fraud-alerts/:alertId/review` (auth, 404,
successful acknowledgement with who/when/note persisted, a second
attempt rejected with 409, a fresh alert starting unacknowledged):
**73/73 tests passing** (10 suites).

```
cd loan-platform-b/backend
npm install
npm test
```

As of Phase 3 (loan-platform-b / TrustLend): 35/35 core tests passing — the
same 6 suites as loan-platform-a plus one additional test validating the
education-loan-specific `institutionName`/`courseName` requirement. As of
Phase 6, with the seed script added: 36/36 tests passing (7 suites). As of
Phase 7, with the outbox suite added: 42/42 tests passing (8 suites). As of
Phase 9, with the tokenization/encryption suite added: 55/55 tests
passing (9 suites). As of Phase 18, with the same `fraudAlerts.test.js`
suite as loan-platform-a (14 tests): 69/69 tests passing (10 suites). As
of Phase 19, with the same 5 acknowledgement tests as loan-platform-a:
**74/74 tests passing** (10 suites).

```
cd loan-platform-c/backend
npm install
npm test
```

As of Phase 4 (loan-platform-c / SecureCredit): 37/37 core tests passing —
the same 6 suites as loan-platform-a plus three additional tests validating
the micro-loan ceilings (`loan.amount` > 50000 rejected, `loan.tenureMonths`
> 12 rejected, exactly-at-ceiling accepted). As of Phase 6, with the seed
script added: 38/38 tests passing (7 suites). As of Phase 7, with the
outbox suite added: 44/44 tests passing (8 suites). As of Phase 9, with the
tokenization/encryption suite added: 57/57 tests passing (9 suites). As
of Phase 18, with the same `fraudAlerts.test.js` suite (14 tests):
71/71 tests passing (10 suites). As of Phase 19, with the same 5
acknowledgement tests as loan-platform-a: **76/76 tests passing** (10
suites).
`npm run build` for `loan-platform-c/frontend` was also run and succeeds
(Vite production build, no errors).

```
cd loan-platform-d/backend
npm install
npm test
```

As of Phase 5 (loan-platform-d / FinBridge): 36/36 core tests passing — the
same 6 suites as loan-platform-a plus two additional tests validating the
point-of-sale fields (`loan.merchantName`/`loan.productCategory` missing
rejected, `loan.productCategory` outside the allowed list rejected). As of
Phase 6, with the seed script added: 37/37 tests passing (7 suites). As of
Phase 7, with the outbox suite added: 43/43 tests passing (8 suites). As of
Phase 9, with the tokenization/encryption suite added: 56/56 tests
passing (9 suites). As of Phase 18, with the same `fraudAlerts.test.js`
suite (14 tests): 70/70 tests passing (10 suites). As of Phase 19, with
the same 5 acknowledgement tests as loan-platform-a: **75/75 tests
passing** (10 suites). `npm run build`
for `loan-platform-d/frontend` was also run and succeeds (Vite
production build, no errors; served `<title>FinBridge</title>`
confirmed).

See "Why tests don't need a live MongoDB" below for how this runs without a
MongoDB instance in this build sandbox, and each platform's own README for
the same explanation in context.

```
cd central-fraud-intelligence
pip install -r requirements-dev.txt
pytest tests/ -v
```

As of Phase 8: 10/10 tests passing (`tests/test_ingestion.py`) — see
"Phase 8: the central app's own tests, against a real local Postgres, not
a double" below. As of Phase 9, with `tests/test_admin_auth.py` (10 tests)
and `tests/test_tokenization.py` (16 tests, including real Node↔Python
cross-language parity checks) added, and `test_ingestion.py` extended by 2
more (auth-required / invalid-token cases on the now-protected GET
endpoint): 38/38 tests passing (3 files). As of Phase 10, with
`tests/test_graph.py` (5 tests) added and `test_ingestion.py`'s one
status-assertion updated (`RECEIVED` → `GRAPH_UPDATED`, reflecting the
graph step now completing in the same request): 43/43 tests passing
(4 files). As of Phase 11, with `tests/test_subgraph.py` (7 tests) added
for `GraphStore.get_relevant_subgraph()`: 50/50 tests passing (5 files).
As of Phase 12, with `tests/test_scenario_generator.py` (14 tests) added
for the synthetic fraud scenario generator: 64/64 tests passing (6
files) — see "Phase 12: scenario generator tests" below. As of Phase 13,
with 2 tests added to `tests/test_graph.py` (`export_all()`, see below)
and a new `tests/test_gnn.py` (21 tests) covering feature engineering,
the model, dataset construction, the non-graph baseline, and a real
end-to-end training run: 87/87 tests passing (7 files) — see "Phase
13: GNN training tests" below. As of Phase 14, with a new
`tests/test_inference.py` (14 tests) covering `GNNScorer.load()`'s
failure modes, `score_subgraph()` against a real extracted subgraph, the
live wiring into the POST/GET endpoints (success, failure/fallback,
redelivery-retry, already-`ANALYZED`-not-rescored), and a real-app
boot-order test proving a corrupted checkpoint doesn't crash startup:
101/101 tests passing (8 files) — see "Phase 14: GNN inference
tests" below. As of Phase 15, with a new `tests/test_fraud_detection.py`
(15 tests) covering the rule engine (`evaluate_rules()` against real graph
fixtures — isolated/shared-token/hub/document-reuse/hard-negative cases)
and `combine_signals()` (weighted-average combination, the `None`-GNN
fallback, threshold boundary, clamping), plus `tests/test_inference.py`
updated for the fact that a successfully-scored event now progresses
automatically past `ANALYZED` to a real terminal status within the same
request (previously it stopped at `ANALYZED`, since Phase 14 had nothing
downstream to consume `fraudScore`) and gaining one new dedicated
rule-engine-failure retry-safety test (net: 14 → 15 tests in that file):
117/117 tests passing (9 files) — see "Phase 15: fraud detection
tests" below. As of Phase 16, with a new `tests/test_explainability.py`
(15 tests) covering `generate_explanation()` directly (every status/
signal combination — in-progress, CLEARED with/without weak signals,
FRAUD_DETECTED driven by rules+GNN/GNN-only/rules-only, document-hash
phrasing, multiple triggered rules, an unrecognized status, and
malformed/missing input never raising) plus two live-wiring tests
proving a real `explanation` key is present on both the POST and GET
responses: 132/132 tests passing (10 files) — see "Phase 16:
explainability tests" below. As of Phase 17, with a new
`tests/test_incidents.py` (17 tests) covering the pure incident-assembly
functions in `src/incidents/incidents.py` directly (deriving affected
applications/platforms from a hand-built subgraph, deduplication,
deterministic `incident_id`, defensive fallbacks) plus real live wiring:
a GNN-only `FRAUD_DETECTED` decision creates a real, persisted incident;
a `CLEARED` decision creates none; a genuinely multi-platform incident
(two applications on two different platforms sharing a phone token,
minutes apart) where the second event's incident lists both applications
while the first event's own incident - created before the second
application existed - lists only itself; a redelivered `FRAUD_DETECTED`
event never creates a duplicate incident (verified against the real
table row count, not just the response); `incident` appears correctly
(present/`null`) in `GET /api/v1/applications/:id`'s per-event output;
and the two new endpoints, `GET /api/v1/incidents` /
`GET /api/v1/incidents/:id`, real - admin-auth-gated (401 without a
token), 404 for an unknown id, and returning what was actually
persisted: 149/149 tests passing (11 files) — see "Phase 17: fraud
incident tests" below. As of Phase 18, with a new
`tests/test_notifications.py` (30 tests) covering: the pure
`build_notification_payload()` assembly function directly; real
`PostgresNotificationRepository` behavior against the test database
(idempotent `create_or_get`, `list_deliverable`'s PENDING-and-due
filter, `record_failed_attempt`'s capped-backoff/FAILED-after-max-
attempts logic); `NotificationDeliveryWorker.deliver_one()`/
`deliver_pending()`/`start()`/`stop()` against a real `httpx.AsyncClient`
built on `httpx.MockTransport` (a real client, fake transport - no real
socket, but the actual request/response code path is exercised),
covering SENT/RETRY/FAILED outcomes, missing-platform-config, and a
missing-incident defensive case; live wiring through
`POST /api/v1/applications/process` proving `notifications` appear in
the response for both a single- and a multi-platform incident, one row
per affected platform, empty when CLEARED, and never duplicated on
redelivery; `notifications` folded into `GET /api/v1/applications/:id`'s
per-event output; and the two new endpoints,
`GET /api/v1/fraud-notifications` / `GET /api/v1/fraud-notifications/:id`,
admin-auth-gated, 404 for an unknown id, returning what was actually
persisted: 179/179 tests passing (12 files) — see "Phase 18: fraud
notification tests" below. As of Phase 19, with five new test files
(`test_graph_api.py`, 13 tests; `test_incidents_review.py`, 30 tests;
`test_notifications_review.py`, 9 tests; `test_dashboard.py`, 18 tests;
`test_metrics.py`, 6 tests): **238/238 tests passing** (17 files) — see
"Phase 19: review workflow, graph explorer, metrics, and dashboard
tests" below.

All four loan platforms have real, passing test suites: 298 tests total
(73 + 74 + 76 + 75), each including a `seed.test.js` suite (Phase 6), an
`outbox.test.js` suite (Phase 7), a `tokenization.test.js` suite
(Phase 9), and a `fraudAlerts.test.js` suite (Phase 18's 14 receiving/
listing tests + Phase 19's 5 acknowledgement tests, 19 total each). With
the central app's 238, **536 tests total across this project.**

## Why tests don't need a live MongoDB (this sandbox specifically)

A real MongoDB instance is not reachable in the sandbox this project is
being built in — Docker Hub and MongoDB's own binary-download hosts both
return a hard `403` under this session's egress policy (see
`TROUBLESHOOTING.md`). Rather than skip DB-backed tests, each loan
platform's data-access layer is written as a repository interface
(`src/repositories/*.js`) with two implementations: a Mongoose-backed one
used in production/Docker, and an in-memory one used only by the test
suite. Every test in `loan-platform-a/backend/tests/` drives the real
Express app (`src/app.js`) — same routes, same services, same auth
middleware, same validation and status-transition rules — over real HTTP
via `supertest`; only the storage layer is swapped. This is why these
results can be reported as genuinely passing rather than "not verified":
the thing being tested (business logic + API contract) is real, even
though the specific MongoDB driver code path (`repositories/*Mongo*`) is
exercised only once `docker compose up` runs against a real `mongo-loan-a`
container (Phase 24, pending the user's environment).

## Phase 6: seed data is verified the same way

Each platform's `scripts/seed.js` is written the same way as its
`server.js`/`createAdmin.js`: its core logic (`seed()`) is exported
separately from its `main()` CLI entrypoint, which is the only place that
wires the Mongo-backed repositories and connects to a real MongoDB. Each
platform's `tests/seed.test.js` calls `seed()` directly against the
in-memory repositories - real execution, not a syntax check - and asserts
both what `seed()` reports back *and* what's actually now in the
repositories (an admin with `role: "admin"`, every seeded status
represented, a non-empty repayment schedule for every approved
application). This is the same "real logic, swapped storage layer"
approach as every other test in this project, applied to the seed script
specifically because it is what a demo of this project actually depends
on working.

## Phase 7: the outbox mechanism is verified against a real local server, not a mock

Each platform's `tests/outbox.test.js` covers two things separately:

1. **Event creation** - submitting an application, or an admin approving/
   rejecting/requesting review, produces exactly the right
   `application_outbox` row (right `eventType`, right `applicationId`,
   right payload contents), verified by reading it back out of the
   in-memory `outboxRepository` after the HTTP call. A third test confirms
   the platform still works with no `outboxRepository` at all (standalone
   mode - nothing crashes, no event is created).
2. **Delivery** - `src/outbox/deliveryWorker.js` is tested against a real
   `http.createServer()` instance spun up on an ephemeral local port for
   the duration of each test (not a mocked `fetch`), so the worker's
   actual request (method, body, `X-Idempotency-Key`/`X-API-Key` headers)
   and its actual handling of the response are genuinely exercised over a
   real socket. Three scenarios are covered: a 200 response marks the
   event `SENT` and a second poll does not re-deliver it (idempotency);
   repeated 500 responses are retried with backoff and the event is marked
   `FAILED` only after `maxAttempts` real attempts, each recorded with its
   error; and `start()`/`stop()` actually run `deliverPending()` on an
   interval without leaking a dangling timer past `stop()`.

The one thing genuinely NOT verified in this sandbox as of Phase 7 was
delivery against the real central application, because that application's
ingestion endpoint didn't exist yet — see "Phase 8" below for how that
changed.

## Phase 8: the central app's own tests, against a real local Postgres, not a double

Unlike the four loan platforms — MongoDB is unreachable in this build
sandbox (Docker Hub / MongoDB binary hosts both `403`, see
`TROUBLESHOOTING.md`), forcing the Mongo-backed/in-memory dual-repository
workaround described above — a real local **PostgreSQL 16** instance IS
reachable in this same sandbox (`postgresql-16` is a normal Ubuntu apt
package, unlike MongoDB, which was dropped from apt years ago over SSPL
licensing). So `central-fraud-intelligence/tests/` needed no in-memory
double at all: every test in `tests/test_ingestion.py` runs against a real,
dedicated `central_fraud_intelligence_test` Postgres database (truncated
between tests), driving the real FastAPI app end-to-end via `TestClient`
(a real ASGI request/response cycle - routing, dependency injection, header
parsing, JSON (de)serialization). This is a stronger guarantee than the
loan platforms get: the actual `psycopg` driver code path — not just the
business logic above it — is exercised by every test.

Covered: health endpoints unaffected by the new router; missing/invalid
`X-API-Key` rejected (401); a valid key that authenticates a *different*
platform than the event's `platformId` claims is rejected (403); an
`X-Idempotency-Key` that doesn't match the body's `eventId` is rejected
(400); a malformed event (missing a required field) is rejected (422); a
valid event is accepted, persisted, and retrievable via `GET
/api/v1/applications/:id`; a second, different event for the same
application appears as a second row; the *same* event delivered twice
(matching `eventId`) is persisted exactly once — verified via a real
Postgres `ON CONFLICT (event_id) DO NOTHING` constraint, not an
application-level check-then-write, so this is genuinely race-safe, not
just race-unlikely; an unknown `applicationId` returns 404.

**Also verified as a true end-to-end smoke test** (manual, one-off, not
part of the automated suite — recorded in `PROGRESS.md`'s Phase 8 session
log since it spans two separate services/processes rather than fitting
either one's own test suite): loan-platform-a's real
`src/outbox/deliveryWorker.js`, given an in-memory outbox event and pointed
at a locally-running `uvicorn` instance of `central-fraud-intelligence`,
successfully delivered the event over a real HTTP request and the worker
correctly marked it `SENT` — confirmed independently via the central app's
own `GET /api/v1/applications/:id`. This is the first time in this project
that an event has traveled all the way from a loan platform's real outbox
mechanism into the real central application and back with a real ack —
previously (Phase 7) every real delivery attempt genuinely failed because
the target endpoint didn't exist.

**What's still genuinely NOT verified**: everything past `status:
"RECEIVED"` in the pipeline (graph, GNN, scoring, incidents, notifications
- Phases 10-18). The payload decrypt/tokenize item listed here as of Phase
8 is resolved as of Phase 9 below — not by this endpoint decrypting/
tokenizing anything (it never needed to), but by tokenization happening
upstream on the loan platform before the event is ever sent (see
`DATA_FLOW.md`'s Step 3 update).

## Phase 9: tokenization and admin auth tests

**Tokenization/encryption (`tokenization.test.js` on all four loan
platforms, `test_tokenization.py` on the central app)**: determinism (same
input -> same token, always), normalization equivalence (`"John Doe"` and
`"  john   doe  "` tokenize identically), different inputs producing
different tokens, the four token purposes (name/phone/email/address) not
being correlatable with each other even though they're derived from the
same master key, a different master key producing entirely different
tokens, missing/empty values returning `null` rather than tokenizing an
empty string, phone normalization stripping non-digit characters before
hashing, AES-256-GCM round-tripping correctly, two encryptions of the same
plaintext producing different ciphertext (random IV verified, not just
documented), decryption failing closed on the wrong key, and decryption
failing closed on a tampered ciphertext (the GCM auth tag genuinely doing
its job, not a no-op). Each platform's `outbox.test.js` was also extended
to assert that a real submitted-application's outbox payload, serialized to
JSON, never contains the raw email/phone/name strings, that
`payload.applicant` is `undefined`, and that the token values present
match what calling `tokenize()` directly on the same raw input produces (so
the test would catch a wrong-but-still-PII-free token, not just "PII
absent").

**Real cross-language parity, not just "both compile"**: the central app's
`test_tokenization.py::TestCrossLanguageParity` class `subprocess.run()`s
the real `node` binary to execute the actual
`loan-platform-a/backend/src/tokenization/{tokenize,encrypt}.js` files (the
same files loan-platform-a's own Node test suite exercises) and asserts
byte-for-byte agreement with the Python implementation's output for the
same inputs — including a full round trip where Node encrypts and Python
decrypts, and the reverse. This is a genuine behavioral check that the two
independently-written implementations (Node's built-in `crypto.hkdfSync`
vs. Python's `cryptography` library HKDF) actually agree, not merely that
both exist. (Automatically skipped, not silently passed, if `node` isn't on
`PATH` — see the module's `NODE_AVAILABLE` guard.)

**Central-app admin JWT auth (`test_admin_auth.py`)**: login succeeds with
correct credentials and returns a real 3-part JWT; login is case-insensitive
on email; wrong password and unknown email are both rejected with an
identical 401 (anti-enumeration); a token obtained through the real
`/api/v1/admin/login` HTTP endpoint (not hand-constructed) actually grants
access to the protected `GET /api/v1/applications/:id` endpoint (asserted
by getting a 404 for a nonexistent application rather than a 401 — proof
the request reached the real business logic); a token issued with a
negative `expires_in_seconds` is rejected as expired; a token with one
character flipped near the signature is rejected as tampered; a token
signed with the wrong secret is rejected; and the `jwt_auth` module itself
is round-tripped and its wrong-secret-rejection checked directly,
independent of the HTTP layer. `test_ingestion.py`'s GET-endpoint tests
were updated to pass a real admin JWT (via a new `admin_token` fixture that
creates a real admin and logs in through the real endpoint), and two new
tests confirm the endpoint now genuinely 401s with no token or an invalid
one — this is what actually verifies the auth gap `SECURITY.md` flagged as
open as of Phase 8 is now closed, not merely documented as closed.

All 38 central-app tests (`test_ingestion.py` + `test_admin_auth.py` +
`test_tokenization.py`) were run together as one `pytest tests/ -v`
invocation (not just each file individually) to confirm no fixture
ordering/interaction issue between the new admin-auth fixtures in
`conftest.py` and the existing ingestion tests — all 38 passed.

## Phase 10: persistent graph tests

`tests/test_graph.py` (5 tests), against the real
`central_fraud_intelligence_test` Postgres database (a new `graph_store`
fixture in `conftest.py`, truncated between tests same as every other
fixture - see `GraphStore.truncate_all()`):

- **Node/edge creation from a real event** - POSTs a real
  `APPLICATION_SUBMITTED` payload (full shape: `applicantTokens`,
  `documents`, `loan`) through the real HTTP endpoint, then asserts every
  expected node (application, platform, borrower, 4 token nodes, 1
  document) and edge exists in the graph, with the right properties
  (loan amount/status as Application node properties, per
  `ARCHITECTURE.md` section 6) and exact node/edge counts (8 nodes, 7
  edges for this fixture) - not just "some nodes got created somewhere".
- **A status-changed event updates the existing node in place** - posts
  SUBMITTED then STATUS_CHANGED for the same application, asserts the
  node count is unchanged (no second Application node), the `status`
  property reflects the new status, and `loanAmount` (set by the earlier
  event) survived the merge - proves the shallow-merge upsert semantics
  described in `ARCHITECTURE.md` section 6, not a naive overwrite.
- **Entity resolution - the core reason this phase exists**: two
  applications, submitted from two *different* platforms with two
  *different* local `userId`s, whose applicants share one phone token.
  Asserts there is exactly one `phone_token` node (not two), that both
  distinct Borrower nodes have an edge into it, and that the two
  Application/Borrower node pairs otherwise stay distinct - the real
  cross-platform-linkage structure a later fraud signal (Phase 15+) would
  need to exist.
- **Idempotent replay does not duplicate graph state** - the identical
  event POSTed twice through the real HTTP endpoint (mirroring a real
  delivery-worker retry) leaves node/edge counts unchanged on the second
  call (`duplicate: true`, `status` still `GRAPH_UPDATED`); a third,
  direct `GraphStore.apply_event()` call with the same payload (bypassing
  the HTTP idempotency-key layer entirely) is a lower-level proof that the
  upserts themselves - not just the HTTP-layer skip - are what make this
  safe.
- **NetworkX materialization matches Postgres across `GraphStore`
  instances** - builds a small graph with one `GraphStore` instance, then
  constructs a *second*, independent instance pointed at the same
  database that has never had `upsert_node`/`upsert_edge` called on it,
  and asserts its node/edge counts, node properties, and edge-derived
  neighbor relationships are identical - the concrete meaning of
  "materialized from Postgres" per `graph_store.py`'s docstring, given
  materialization is lazy rather than an eager blocking startup call (see
  that docstring, and `ARCHITECTURE.md` section 6, for why).

Also verified directly, mirroring the exact same boot-order check Phase 8
did for `PostgresEventRepository`: pointed `CENTRAL_DB_URI` at an
intentionally unreachable host and confirmed `/health`/`/api/v1/health`
still return 200 (the FastAPI startup event's best-effort graph warm-up
catches the connection failure and logs a warning rather than crashing the
process - see `src/main.py`'s `_lifespan`), while
`POST /api/v1/applications/process` correctly returns 503. This was a
one-off manual verification (not part of the automated suite, since it
requires deliberately misconfiguring the app's own DSN rather than the
test database), recorded here rather than as a counted test.

## Phase 11: subgraph extraction tests

`tests/test_subgraph.py` (7 tests), against the real
`central_fraud_intelligence_test` Postgres database via the same
`graph_store` fixture as `test_graph.py`. These build graph state directly
with `upsert_node`/`upsert_edge` (not the HTTP layer) since this phase is
about reading data back out, not re-testing ingestion:

- **Nonexistent node returns `None`** - the base case, checked first
  since everything else assumes a root that exists.
- **An isolated node returns itself with no edges** - `nodeCount == 1`,
  `edgeCount == 0`, `truncated == False`.
- **Default hop count (4) reaches a sibling application across a shared
  token, but not one hop further** - the core case this phase exists for:
  builds the minimal two-application/shared-phone-token "ring" and asserts
  the *other* application is present in the default-hop subgraph, while
  that other application's own platform node (one hop further) is
  correctly absent - proving the default bound is exactly what
  `graph_store.py`'s docstring claims it is, not an arbitrary number that
  happens to work in this one test.
- **`hops` parameter measurably bounds the traversal** - the same ring
  with `hops=1` returns only the root's immediate neighbors (3 nodes, 2
  edges), proving the parameter isn't ignored.
- **`max_nodes` truncates and reports `truncated: true`** - the same ring
  with `max_nodes=4` stops mid-traversal (right after the shared token,
  before crossing into the other application's side) and the result
  says so.
- **Result is deterministic across repeated calls** - two calls with
  identical arguments return byte-identical results, verifying the
  sorted-neighbor-order traversal genuinely removes the set-iteration
  non-determinism a naive BFS would have.
- **Subgraph is a true induced subgraph, not just a BFS tree** - two
  borrowers sharing *both* a phone and an email token: both shared-token
  edges appear in the result even though BFS only needed one of the two
  to discover each borrower, proving `subgraph_view.edges()` (all edges
  among visited nodes) is what's returned, not just the edges BFS
  happened to traverse first.

## Phase 12: scenario generator tests

`tests/test_scenario_generator.py` (14 tests). Two layers, mirroring the
`test_graph.py`/`test_subgraph.py` split:

- **Generator-logic tests, no database needed** (fast, checks
  `generate_catalog()`'s own output directly): generation is
  deterministic given a seed; different seeds produce genuinely different
  data; every `scenario_id`/`applicationId`/`eventId` across the whole
  catalog is unique; the catalog covers exactly 17 pattern numbers and
  the 3 deliberately deferred ones are exactly `{6, 7, 16}`, so an
  accidental drift in either set fails loudly; every scenario carries a
  valid `expected` label (`SUSPICIOUS`/`GENUINE`) and a boolean
  `hard_negative` flag; targeted structural assertions per pattern family
  - `same_phone_different_identities` shares only the phone token and
    nothing else;
  - `same_borrower_2_platforms` shares every token and is SUSPICIOUS;
  - `legitimate_cross_platform_applicant` shares the *identical* tokens as
    `same_borrower_2_platforms` (same graph shape) but is GENUINE/
    hard-negative, and its `submittedAt` values are genuinely months apart
    (year-month prefixes differ) where `same_borrower_2_platforms`'s are
    minutes apart (same year-month) - the concrete proof of the
    topology-vs-timing distinction described in `scenario_generator.py`'s
    module docstring;
  - `legitimate_repeated_application` reuses the same `platformId`+
    `userId` (same Borrower) across two different `applicationId`s;
  - `shared_document_fingerprint` has identical `sha256` but different
    applicant tokens;
  - `multiple_identities_one_controller` has one shared phone token
    across 5 otherwise-distinct identities;
  - `collusive_graph_pattern` has no single token type shared by all four
    identities (only adjacent pairs);
  - `mixed_fraud_genuine_neighborhood`'s bridge application's phone token
    genuinely matches a real SUSPICIOUS scenario and its address token
    genuinely matches a real GENUINE scenario.
- **A real-database integration test**: every one of the (currently 51
  scenarios, 129 applications) generated events is fed through the exact
  same `GraphStore.apply_event()` the real ingestion endpoint calls,
  against the real `central_fraud_intelligence_test` Postgres database -
  zero errors required, not assumed. A representative structural
  spot-check follows against the real, persisted graph (not the in-memory
  Python objects): the hub scenario's shared phone token has exactly 5
  `BORROWER_HAS_PHONE` edges into it, read back via Phase 11's
  `get_relevant_subgraph()`.

Also verified directly (part of `scripts/generate_scenarios.py`'s own
run, not a counted pytest test, recorded here and in `PROGRESS.md`): the
actual committed `seed-data/scenarios/scenarios.json` - 51 scenarios, 129
applications - applied to a real `GraphStore` with zero errors before
being committed.

## Phase 13: GNN training tests

`tests/test_graph.py` gained 2 tests for the new
`GraphStore.export_all()` method (the training-time whole-graph
counterpart to Phase 11's bounded `get_relevant_subgraph()`): a real
integration test proving two entirely disconnected components (no shared
node/edge at all) both come back in a single `export_all()` call - the
specific property a bounded BFS from one root node structurally cannot
provide, demonstrated by directly comparing against
`get_relevant_subgraph()` on the same data in the same test; and a
trivial empty-graph case.

`tests/test_gnn.py` (21 tests, all against real code paths - no mocked
torch/sklearn) covers four layers:

- **`src/gnn/features.py`** (no database needed): the node-type
  vocabulary is exactly the 8 real node types; `ALL_RELATIONS` is exactly
  the 7 real edge types plus their 7 reverse counterparts;
  `build_node_index()` is deterministic regardless of input order;
  `FEATURE_DIM == 16`; a non-Application node gets only its type one-hot
  set (everything else zero); an Application node gets the correct scaled
  loan amount / platform one-hot / scaled timestamp; a missing or
  unparseable timestamp defaults to `0.0` rather than crashing; **the
  topology-vs-timing distinction is provably visible only in the
  timestamp feature dimension** - two applications built with identical
  node-type/platform/loan-amount but timestamps minutes vs. months apart
  differ in exactly one feature dimension, the concrete proof that this
  feature is where DEMO_SCENARIOS.md patterns 1-3 vs. 19 (see
  `datasets/scenario_generator.py`) become distinguishable at all;
  `add_structural_features()` sets the correct log-scaled degree values;
  `TIMESTAMP_EPOCH` matches `scenario_generator.py`'s `_BASE_DATETIME`
  exactly (a cross-module consistency check, not just an internal one).
- **`src/gnn/model.py`** (no database needed, synthetic tiny graphs): a
  forward pass produces exactly one logit per node; a graph with zero
  edges at all still produces a finite logit (the self-transform path
  alone); gradients actually flow into a relation's weight matrix when
  that relation has at least one edge; `--num-layers`'s default equals
  `graph_store.DEFAULT_SUBGRAPH_HOPS` (pins the cross-phase design claim
  from `model.py`'s docstring so it can't silently drift); and a cheap
  "loss decreases over 30 steps on a tiny linearly-separable toy graph"
  smoke test proving the training loop itself (forward → loss → backward
  → optimizer step) is wired correctly, independent of how well the real
  model learns the real data.
- **`training/dataset.py`** (real database, real committed
  `scenarios.json`): `scenarios.json` loads and has the expected 51
  scenarios / 129 applications; the scenario-stratified split is
  deterministic across repeated calls, covers every scenario exactly
  once, and every pattern with ≥2 instances appears in both train and
  eval; and a full real-database integration test building the whole
  dataset - proving `export_all()` + feature/label/split construction
  together produce a leak-free result (no application in both splits,
  every application accounted for), that label propagation from scenario
  → every one of its applications is exactly correct (spot-checked
  against every one of the 129 real applications, not sampled), and that
  the adjacency tensors are genuinely non-empty (catches a key-mismatch
  bug between `export_all()`'s edge shape and this module's parsing of
  it, which a shape-only check would miss).
- **`training/baseline.py` and `training/train.py`** (real database, real
  committed `scenarios.json`): the non-graph baseline runs end-to-end and
  returns a structurally valid metrics dict (correct dataset counts,
  metrics in `[0, 1]`, a confusion matrix that sums to the eval count);
  and a real (few-epoch, for test speed) end-to-end run of `train()`
  itself, proving the whole pipeline - dataset build, model, training
  loop, evaluation, checkpoint + meta save - runs without error and that
  the saved checkpoint is a genuinely reloadable state dict, not just a
  file that happens to exist.

None of these tests assert a specific trained accuracy number - that
would be asserting the outcome of a stochastic training run rather than
code correctness. The real training run's real metrics are reported
honestly (not as a test assertion) in `models/gnn_checkpoint.meta.json`
and `ML_MODEL.md`'s Phase 13 section instead, per this project's "NO FAKE
SUCCESS" discipline.

## Phase 14: GNN inference tests

`tests/test_inference.py` (15 tests as of Phase 15 - was 14 at the Phase
14 checkpoint, see "Phase 15: fraud detection tests" below for what
changed and why, all against real code paths) covers three layers:

- **`GNNScorer.load()`'s failure modes** (no database needed): a missing
  checkpoint file, a malformed `meta.json`, a `meta.json` missing a
  required key, and a checkpoint file that isn't a real torch file at all
  - every case returns `None`, none of them raise. Separately, a real
    test loads the actual committed
  `models/gnn_checkpoint.meta.json` + `models/gnn_checkpoint.pt` (this
  build sandbox's real Phase 13 training output) and asserts a real
  `GNNScorer` comes back - **skipped, not silently passed, if that
  git-ignored `.pt` file isn't present** in the environment running the
  suite (mirrors `test_tokenization.py`'s `shutil.which`-based skip
  pattern for an unavailable external binary).
- **`score_subgraph()`** (same real-checkpoint skip guard): run against a
  real subgraph extracted via `GraphStore.get_relevant_subgraph()` from a
  real committed scenario, proving the whole feature-build →
  tensor-build → forward-pass → sigmoid chain produces a genuine
  probability in `[0, 1]`; and a case proving `None` comes back (not an
  exception) when the requested root node isn't actually present in the
  subgraph it's given.
- **Live wiring into `POST /api/v1/applications/process` and
  `GET /api/v1/applications/:id`** (real database, `client`/`admin_token`
  fixtures, `app.dependency_overrides[get_gnn_scorer]` swapped per test -
  never the real checkpoint, so these are deterministic regardless of
  whether `models/gnn_checkpoint.pt` exists in the environment running the
  suite): no scorer loaded leaves `status: GRAPH_UPDATED` and
  `fraudScore: null`; a stub scorer that returns a score advances
  `status` to `ANALYZED` with that real score echoed back; a stub scorer
  that raises leaves the event at `GRAPH_UPDATED` with no score recorded,
  and the request still returns 200 (proves the best-effort
  `try/except` in `src/api/applications.py` actually works, not just that
  it's present in the source); a redelivery of an event that previously
  failed scoring retries inference and can succeed the second time
  (mirrors the exact retry-safety property `test_graph.py` already
  proved for the graph-update step); an event already at a real terminal
  status (`CLEARED`/`FRAUD_DETECTED`, as of Phase 15 - previously
  `ANALYZED`, before anything downstream existed to consume `fraudScore`)
  is **not** rescored on redelivery (the regression this phase's
  status-gating fix - `if status == "GRAPH_UPDATED"` instead of the old
  `if status != "GRAPH_UPDATED"` - specifically guards against, see
  `PROGRESS.md`'s Phase 14 session log); and the GET endpoint returns
  `fraudScore` per event.
- **Boot-order guarantee**: a dedicated test uses the *real*
  `src/main.py` lifespan (not a dependency override) with
  `GNNScorer.load` monkeypatched to run its real implementation against a
  deliberately corrupt checkpoint file and a deliberately malformed
  `meta.json`, then asserts `/health` and `/api/v1/health` still return
  200 and `app.state.gnn_scorer is None` - the same boot-survives-a-
  broken-dependency guarantee Phase 8/10 already proved for an
  unreachable Postgres, now proved for a broken GNN checkpoint too.

Because the real trained checkpoint genuinely exists in this build
sandbox (it was produced here, in Phase 13, and never deleted), every
skip-guarded test above ran for real, not skipped, when this suite was
last run in this environment - confirmed by reading the actual `pytest
-v` output, not assumed from the code.

## Phase 15: fraud detection tests

`tests/test_fraud_detection.py` (15 tests, all against real graph
fixtures built the same way `test_graph.py`/`test_subgraph.py` build
theirs - real `GraphStore.upsert_node`/`upsert_edge` calls against a real
Postgres test database, never a mock) covers `evaluate_rules()` and
`combine_signals()` directly:

- An isolated application with zero shared signals scores `0.0`; a
  borrower re-submitting their own application (no other identity
  involved at all - the exact shape of pattern 17, the
  `legitimate_repeated_application` hard negative) also scores `0.0`,
  proving the root-borrower-exclusion logic actually works and isn't
  just documented.
- A shared phone number minutes apart scores near the full `0.30`
  weight; the same shared phone five months apart scores near the
  `0.05`-floor weight instead - both from the *same* time-proximity
  multiplier function, proving the "velocity is folded into every
  shared-token check, not a separate rule" design decision (see
  `rules.py`'s module docstring and `ML_MODEL.md`'s Phase 15 section)
  actually produces the two different real numbers it claims to.
- A shared address alone, two weeks apart (pattern 15's hard negative)
  scores low; the same four-token full identity match five months apart
  (pattern 19's hard negative - deliberately the same graph topology as
  the SUSPICIOUS pattern 1) also scores low despite matching every
  token type at once - both hard negatives the GNN alone was already
  checked against in Phase 13 now checked against the rule engine too.
- Address-plus-phone scores higher than address alone (corroborating
  signal escalation, `ARCHITECTURE.md` section 8's explicit example); a
  hub of three or more distinct identities sharing one token gets the
  hub bonus on top of the base weight; a shared document hash across
  distinct applications triggers the (highest-weighted) document signal;
  a malformed/incomplete subgraph degrades to `rule_score: 0.0` rather
  than raising.
- `combine_signals()`: the weighted average when both scores are
  present; using `rule_score` directly (unscaled) when `gnn_score` is
  `None`; the exact `FRAUD_DETECTED`/`CLEARED` threshold boundary; score
  clamping to `[0, 1]`; and the "no signals at all" case landing at
  `CLEARED`.

`tests/test_inference.py` gained one new test this phase specifically
for the rule engine's OWN best-effort/retry-safety guarantee (the same
property Phase 14 proved for GNN inference, now proved for this step
too): `evaluate_rules` monkeypatched to raise leaves the event parked at
`ANALYZED` with `ruleScore`/`triggeredRules`/`finalRiskScore` all `null`
and the request still returning 200; removing the monkeypatch and
redelivering the same event reaches a real `FRAUD_DETECTED` the second
time.

**Real evaluation against the full committed dataset** (not a pytest
test - a standalone script, `scripts/evaluate_fraud_detection.py`, run
directly in this environment against the real trained checkpoint and all
129 real applications in `seed-data/scenarios/scenarios.json`): real
measured accuracy of 73.64% (realistic interleaved-traffic replay) /
71.32% (isolated per-scenario replay, run specifically to test - and
ultimately disprove - a cross-scenario-interleaving hypothesis for the
gap from Phase 13's 97.67% offline GNN-only number). The real,
empirically-confirmed explanation, and a second genuine unresolved GNN
false-positive class this evaluation surfaced, are both written up
honestly in `ML_MODEL.md`'s Phase 15 section, with full detail in
`models/hybrid_scoring_eval.json` - reported as real findings, not
smoothed into a single flattering headline number, per this project's
"NO FAKE SUCCESS" discipline.

## Phase 16: explainability tests

`tests/test_explainability.py` (15 tests) covers `generate_explanation()`
(`src/explainability/explain.py`) directly - no HTTP or database
involved for most of them, since it is a pure function - plus two live-
wiring tests:

- Every in-progress status (`RECEIVED`/`GRAPH_UPDATED`/`ANALYZED`) and a
  missing/empty status all produce an honest "not decided yet" summary,
  never a guess at a decision that hasn't been made.
- `CLEARED` with zero signals; `CLEARED` with a weak signal present but
  correctly explained as insufficient to cross the threshold (mentions
  the signal by name, states it wasn't enough).
- `FRAUD_DETECTED` with both a rule signal and a GNN score contributing -
  the summary names the specific signal (with its real minute/hour
  timing derived from `triggeredRules`' `closestHoursApart` detail) and
  the model's independent probability; the technical breakdown's
  `weightedContribution` values are cross-checked against
  `scoring.py`'s real `GNN_WEIGHT`/`RULE_WEIGHT` constants (imported, not
  hand-copied, so the test can't silently drift from the real weights).
- `FRAUD_DETECTED` driven solely by the GNN (`triggeredRules: []`) -
  proves the specific caveat naming Phase 15's known legitimate-repeat-
  applicant false-positive limitation and recommending manual review is
  present, and proves it is absent when a real rule signal did fire.
- `FRAUD_DETECTED` driven solely by rules (`fraudScore: None`, no
  checkpoint loaded) - proves the technical breakdown reflects
  `combine_signals()`'s real documented fallback behavior (rules decide
  alone, `effectiveWeight: 1.0`, not scaled by the static `RULE_WEIGHT`
  constant) rather than just echoing the configured weights.
- The document-hash signal gets its own natural-language phrasing,
  distinct from a shared-token signal; multiple simultaneously-triggered
  rules are all mentioned in one summary; an unrecognized status (the
  documented-but-never-actually-set `FAILED` value) falls back to a
  generic, honest message rather than guessing at wording for a case no
  real code path has ever produced; and a completely empty/malformed
  input never raises.
- Live wiring: a real `POST /api/v1/applications/process` response (stub
  GNN scorer, real rule engine, real isolated application) includes a
  real `explanation` with the GNN-only caveat present; a real
  `GET /api/v1/applications/:id` response includes a real `explanation`
  per event, reflecting a real `CLEARED` decision.

## Phase 17: fraud incident tests

`tests/test_incidents.py` (17 tests), three layers - pure assembly, live
wiring, and the two new read endpoints:

- Pure functions (`src/incidents/incidents.py`), no HTTP/database: every
  Application node in a hand-built subgraph is returned, sorted,
  deduplicated (`derive_affected_applications`); a missing root falls
  back defensively without raising; `affected_platform_ids` dedupes,
  sorts, and ignores a missing platform id; `build_incident_record`
  produces the exact right shape, a deterministic `incident_id`
  (`INC-<eventId>`), `review_status: "OPEN"`, and always includes the
  triggering platform in `affected_platforms` even in a defensive
  subgraph-is-`None` fallback case.
- Live wiring, real HTTP against real Postgres/graph, a stubbed GNN
  scorer (same pattern as `test_explainability.py`): a GNN-only
  `FRAUD_DETECTED` decision on an isolated application produces a real,
  persisted incident with the manual-review caveat carried into its
  explanation snapshot; a `CLEARED` decision produces `incident: null`;
  **a genuinely multi-platform incident** - two applications, platforms
  `a` and `b`, sharing one phone token three minutes apart - where the
  *second* event's incident correctly lists both applications and both
  platforms while the *first* event's own incident (created before the
  second application existed in the graph) correctly lists only itself,
  proving an incident is a real point-in-time snapshot, not silently
  backfilled later; a redelivered `FRAUD_DETECTED` event returns the
  identical `incidentId` both times and the real `fraud_incidents` table
  has exactly one matching row (checked directly against the repository,
  not just the response); `GET /api/v1/applications/:id` carries the
  right `incident` (present for `FRAUD_DETECTED`, `null` for `CLEARED`)
  per event.
- `GET /api/v1/incidents` / `GET /api/v1/incidents/:id`: both 401
  without an admin token; `GET .../:id` 404s for an id that was never
  created; both return what a real POST actually persisted (list
  contains the right `incidentId`; detail matches field-for-field).

## Phase 18: fraud notification tests

`tests/test_notifications.py` (30 tests), four layers - pure payload
assembly, the repository, the delivery worker, and live wiring/read
endpoints:

- Pure function (`src/notifications/notifications.py`), no HTTP/
  database: `build_notification_payload()` shapes every field from a
  hand-built incident row; null `affected_applications`/
  `affected_platforms` default to `[]`; a multi-platform incident's full
  ring (every application/platform, not just one) is carried through
  unchanged.
- `PostgresNotificationRepository`, real Postgres, no HTTP: `create_or_get`
  produces the deterministic `NOTIF-<incidentId>-<platformId>` id and is
  idempotent (a second call for the same incident/platform returns the
  existing row, `created: False`); a separate row is created per platform
  for the same incident; `list_deliverable` returns PENDING-and-due rows,
  excludes SENT ones; `mark_sent` sets `status`/`sent_at`;
  `record_failed_attempt` applies the capped-exponential-backoff formula
  correctly (not immediately due again after a failure) and moves to
  FAILED once `attempts >= max_attempts`, and returns `None` for an
  unknown `notification_id`.
- `NotificationDeliveryWorker`, a real `httpx.AsyncClient` built on
  `httpx.MockTransport` (a real client, fake transport - no real socket,
  but the actual request/response parsing code path is exercised, not a
  hand-rolled double), against real incident/notification rows: a 2xx
  response marks the row SENT and the correct `X-API-Key`/
  `X-Idempotency-Key` headers and payload were sent; a non-2xx response
  retries with a recorded error; repeated failures move the row to
  FAILED after `max_attempts`; a platform missing from
  `platform_base_urls`/`platform_api_keys` is a retryable failure that
  never attempts an HTTP call; a notification referencing a nonexistent
  incident is a retryable failure, never a crash; `deliver_pending()`
  delivers every due row in one pass; `start()`/`stop()` run the
  background loop on an interval and tear it down cleanly (the HTTP
  client is closed, the task is cancelled and awaited).
- Live wiring, real HTTP against real Postgres/graph, a stubbed GNN
  scorer (same pattern as `test_incidents.py`): a GNN-only
  `FRAUD_DETECTED` decision produces exactly one notification, `PENDING`,
  `attempts: 0`, matching the incident it belongs to; a `CLEARED`
  decision produces `notifications: []`; the same genuinely
  multi-platform scenario `test_incidents.py` uses (two applications,
  platforms `a` and `b`, sharing a phone token) produces one notification
  per platform on the incident-creating event; a redelivered
  `FRAUD_DETECTED` event never creates a duplicate notification row
  (checked directly against the repository, not just the response);
  `GET /api/v1/applications/:id` carries the right `notifications` per
  event (populated for `FRAUD_DETECTED`, `[]` for `CLEARED`).
- `GET /api/v1/fraud-notifications` / `GET /api/v1/fraud-notifications/:id`:
  both 401 without an admin token; `GET .../:id` 404s for an id that was
  never created; both return what a real POST actually persisted (list
  contains the right `notificationId`; detail matches field-for-field).

Each loan platform's `tests/fraudAlerts.test.js` (14 tests as of Phase
18, real HTTP via supertest against the real app, in-memory repository -
same pattern every other suite in that codebase uses):
`POST /api/v1/fraud-notifications/receive` rejects a missing/wrong
`X-API-Key` (401), rejects a missing `X-Idempotency-Key` or a missing
required body field or a non-numeric `finalRiskScore` (400), accepts a
valid alert authenticated with the platform's own `CENTRAL_FRAUD_API_KEY`
and persists every field correctly, never creates a duplicate alert on
redelivery of the same `X-Idempotency-Key`, and still acknowledges
receipt (storing nothing) when no `fraudAlertRepository` is wired
(standalone mode, mirroring `outbox.test.js`'s own "works with no
outboxRepository at all" case); `GET /api/v1/admin/fraud-alerts`
requires admin auth, lists what's been received newest-first, and
returns `[]` when nothing has been received yet.

## Phase 19: review workflow, graph explorer, metrics, and dashboard tests

Five new central-app test files (`central-fraud-intelligence/tests/`),
plus 5 new tests appended to each loan platform's existing
`fraudAlerts.test.js`:

- `test_graph_api.py` (13 tests): `GET /api/v1/graph/application/:id` /
  `GET /api/v1/graph/entity/:id`, seeded directly via `graph_store`'s
  `upsert_node`/`upsert_edge` (same approach `test_subgraph.py` already
  uses - this is about the HTTP layer, not re-testing traversal). Both
  401 without an admin token; `.../application/:id` 404s for an unknown
  application; the default neighborhood correctly reaches an application
  across a shared token, `?hops=1` correctly does NOT; `?maxNodes=1`
  correctly truncates (`truncated: true`); an out-of-range `?hops=9999`
  is silently clamped rather than erroring or returning something
  unbounded; `.../entity/:id` 400s for an invalid `nodeType`, 422s
  (FastAPI's own validation) when `nodeType` is omitted entirely, and
  works for every one of the eight documented node types.
- `test_incidents_review.py` (30 tests): pure-logic tests for
  `is_valid_review_transition()`/`build_review_event()` (every status
  has a transition entry, `OPEN` reaches all four others, `UNDER_REVIEW`
  reaches every terminal state but not back to `OPEN`, every terminal
  state accepts no further transition, `build_review_event()`'s shape
  and default `note: None`) plus live HTTP tests against
  `POST /api/v1/incidents/:id/review`: 401 without an admin token, 404
  for an unknown incident, 400 for a garbage `newStatus`, 409 for an
  invalid transition (verified with a real terminal-state incident, not
  just the pure predicate), and a full real transition sequence
  (`OPEN -> UNDER_REVIEW -> CONFIRMED`) that checks the persisted
  `reviewHistory` grows correctly (action/by/note/at) at each step and
  that `GET /api/v1/incidents/:id` reflects the same history back.
- `test_notifications_review.py` (9 tests): repository-level
  (`acknowledge()` sets every field correctly, a second call returns
  `already_acknowledged`, an unknown `notification_id` returns
  `not_found`, acknowledging never touches delivery `status`) and live
  HTTP (401, 404, a real acknowledgement persisting who/when/note and
  visible on a subsequent GET, a second attempt returning 409).
- `test_dashboard.py` (18 tests, real HTML via `TestClient` - actual
  requested pages, actual assertions on their content, not mocked):
  unauthenticated `/dashboard/` and every protected page redirect (303)
  to `/dashboard/login`; the login page renders a real `<form>`; wrong
  credentials show an error and set no cookie; correct credentials set
  the `central_admin_token` cookie and redirect to `/dashboard/incidents`;
  logout clears the cookie; the incidents list and incident detail pages
  render real seeded data (including the review `<form>`'s offered
  transitions changing correctly after a real transition); an unknown
  incident detail page 404s with a message, not a crash; the review
  `<form>` POST actually transitions the incident and persists a note;
  the notifications page lists and can acknowledge a real row; the graph
  pages render a real subgraph's nodes/edges as tables, and handle an
  unknown node without erroring.
- `test_metrics.py` (6 tests): 401 without an admin token; an empty
  database returns all-zero counts in the right shape; events/incidents/
  notifications seeded directly into their repositories are counted
  correctly by status (including the acknowledged/unacknowledged split,
  independent of delivery status); graph node/edge counts match real
  `upsert_node`/`upsert_edge` calls.

Each loan platform's `fraudAlerts.test.js` gained 5 tests for
`POST /api/v1/admin/fraud-alerts/:alertId/review`: 401 without an admin
token, 404 for an unknown `alertId`, a real acknowledgement persisting
who/when/note (verified via a follow-up `GET /api/v1/admin/fraud-alerts`),
a second attempt returning 409, and a freshly received alert starting
`acknowledged: false`.

## Phase 20: real multi-process integration testing (not a unit/route-test phase)

Everything above this section is route/unit-level testing (FastAPI
`TestClient`, Jest + supertest) against in-process or real-local-database
test doubles. Phase 20 is different in kind: it runs all five real
services as five genuinely separate OS processes, talking real HTTP over
real loopback TCP ports, and drives a real multi-step scenario against
them - the same "as real as this sandbox allows" posture this project has
followed since MongoDB and Docker were both confirmed unreachable here
(see `TROUBLESHOOTING.md`'s Phase 1/8/13 entries), extended one more
level: this sandbox's Docker daemon cannot even start at all (a stronger,
newly-confirmed finding this phase - see `TROUBLESHOOTING.md`), so five
real OS processes on real ports is the honest substitute for
`docker compose up`, not a downgrade chosen for convenience.

**What's new, concretely:**
- Each loan platform gained `backend/scripts/run_standalone.js` - a
  Phase-20-only entrypoint (NOT the production `src/server.js`, which is
  unchanged and still requires a real MongoDB) that builds the exact same
  `createApp()` + in-memory-repository wiring `tests/helpers/
  buildTestApp.js` already uses, but calls `app.listen()` on a real port
  instead of driving requests in-process via supertest, and starts a real
  `deliveryWorker` pointed at a real URL. Verified structurally identical
  and platform-agnostic across all four platforms (config.js differs only
  in `PLATFORM_ID`/`PLATFORM_NAME`, read dynamically) before copying it
  to b/c/d unchanged, then smoke-tested standalone (real `/health`, real
  login) before use.
- The root-level `scripts/run_integration_demo.py` orchestrates all five
  processes: truncates the real local Postgres dev database
  (`central_fraud_intelligence`, not `_test`), optionally seeds it with a
  real replay of the committed Phase 12 scenario dataset (reusing
  `scripts/evaluate_fraud_detection.py`'s own `_score_interleaved()`
  directly, not reimplemented), starts all five processes with the real
  env-var wiring `docker-compose.yml`/`.env.example` would otherwise
  provide (matching shared secrets, `localhost` URLs instead of Docker
  DNS names), waits for every `/health`, then drives real HTTP calls:
  real user registration/login on each platform, real admin login on
  each platform and on central, real application submissions, and real
  polling of `GET /api/v1/incidents` and each platform's
  `GET /api/v1/admin/fraud-alerts` for the async outbox/notification
  pipeline to complete. Every process's stdout/stderr is captured to its
  own log file for inspection. See `PROGRESS.md`'s Phase 20 section for
  the real output this produced, including two genuine bugs/gaps it
  found (documented in detail in `TROUBLESHOOTING.md` and, for the GNN
  one, `ML_MODEL.md`) - this is precisely the kind of finding route-level
  testing with test doubles cannot surface, since every prior test either
  drove one service in isolation or used a stub/None GNN scorer.

**Regression coverage added from what this found:** the pipeline-gating
bug (rule engine unreachable with no GNN checkpoint loaded) is now also
covered at the route level, not just by the live demo - `tests/
test_inference.py`'s renamed
`test_post_reaches_a_real_decision_with_fraud_score_null_when_no_scorer_is_loaded`
asserts the corrected behavior directly, and `tests/test_ingestion.py`
and `tests/test_graph.py` (×3 assertions) were updated from their old,
now-incorrect expectations to match. Full central suite re-verified at
238/238 both with and without the real checkpoint present on disk (no
regression in the checkpoint-present path, which the fix didn't touch).
All four loan platforms' suites re-verified unchanged (298 total,
73/74/76/75) since no loan-platform code changed this phase.

**What this run does NOT prove**, stated plainly: real MongoDB driver
behavior (still unreachable in this sandbox - unchanged since Phase 1)
and real Docker container networking (still blocked, and now confirmed
blocked at the daemon level too - see `TROUBLESHOOTING.md`). It also does
not fix or claim to fix the GNN live-inference collapse finding
(`ML_MODEL.md`'s Phase 20 section) - the demo's genuine 3-platform
incident (see `PROGRESS.md`) was captured with the checkpoint
deliberately set aside for that one run, exercising the real, now-fixed
rule-only decision path, clearly labeled as such rather than presented as
GNN-driven detection.

## Phase 21: false-positive-rate evaluation at a larger sample size

Phase 21 adds no new route/unit tests to either the central app's pytest
suite or the loan platforms' Jest suites beyond one thing: a new
structural regression test pair in
`central-fraud-intelligence/tests/test_scenario_generator.py` —
`test_pattern_21_is_excluded_by_default_and_included_when_requested` and
`test_legitimate_repeated_application_three_times_is_the_same_borrower_three_times`
— covering the new opt-in `include_pattern_21` generator flag and the new
pattern 21 scenario shape, in the same style as the existing pattern-by-
pattern structural tests in that file. These run as part of the normal
pytest suite; no new test infrastructure was needed.

The bulk of Phase 21's work is **offline evaluation tooling**, not
route-level testing: `central-fraud-intelligence/scripts/
generate_fpr_dataset.py` (new) generates a 690-application dataset
(`seed-data/scenarios/scenarios_fpr.json`) at 5x the committed dataset's
volume, verified for real against a real Postgres `GraphStore` the same
way `scripts/generate_scenarios.py` verifies the committed dataset. That
dataset is then run through the existing
`scripts/evaluate_fraud_detection.py` (unchanged this phase - it already
supported both interleaved and isolated modes from an earlier phase),
producing `central-fraud-intelligence/models/hybrid_scoring_eval_fpr.json`.
This evaluation calls the real production pipeline functions
(`evaluate_rules()`, `combine_signals()`, and, when a checkpoint is
present, `GNNScorer.score_subgraph()`) directly against real generated
data - not a mock or a hand-computed expected value - so its results are
real measurements, not simulated ones. The findings themselves (accuracy,
confusion matrices, per-pattern breakdown, the platform-bias hypothesis)
are documented in `ML_MODEL.md`'s Phase 21 section, not here - this
section is about the testing/evaluation methodology and infrastructure
only.

**Full regression check after this phase's changes:** the entire central
pytest suite was re-run after these changes to confirm no regression:
**240 passed** (the pre-existing 238 plus the 2 new pattern-21 tests
above), 0 failed. One of the two new tests initially failed on its first
real run (an incorrect assumption in the test itself about which
calendar months three `_minutes_after_base` timestamps 30 real days
apart would land in - not a bug in the generator) and was corrected
before this final count - see `PROGRESS.md`'s Phase 21 entry for the
honest account of that. No loan-platform code changed this phase, so
their suites were not re-run.

## Phase 22: security testing

Two kinds of new coverage this phase, plus a real dependency audit.

**New automated tests** (all run as part of each service's normal
suite - no new test infrastructure needed):
- `central-fraud-intelligence/tests/test_audit_log.py` (10 tests): the
  new `src/audit/repository.py`'s `sanitize_detail()` denylist logic
  directly, plus real HTTP-level proof that admin login (success and
  failure), incident review, and notification review all record a real
  audit event with the right actor/detail and never leak a password -
  including sending a real password over a real login request and
  asserting it's nowhere in the stored row, not merely that a "password"
  key is absent.
- `central-fraud-intelligence/tests/test_security_hardening.py` (8
  tests): a forged "alg: none" JWT (and a variant with a plausible-
  looking fake signature) rejected; malformed/wrong-segment-count tokens
  rejected; the unknown-email-vs-wrong-password admin-login responses
  proven genuinely indistinguishable; two SQL-injection-style probe
  strings (`applicationId`, `eventId`) sent through the real ingestion
  endpoint and confirmed stored/returned as inert literal text with the
  events table still intact afterward.
- `loan-platform-{a,b,c,d}/backend/tests/security.test.js` (7 tests
  each, 28 total): the two real findings this phase's review turned up
  (see `SECURITY.md`'s and `TROUBLESHOOTING.md`'s Phase 22 sections) -
  `register`/`login` now reject a non-string `email`/`password`, and
  `GET /api/v1/admin/applications?status=...` now rejects a
  MongoDB-operator-shaped `status` filter (`?status[$ne]=REJECTED`)
  instead of passing it through - plus a control test confirming an
  ordinary, valid request still works normally in each case.

Full suite counts after this phase: central
**258 passed** (240 Phase 21 baseline + 18 new), loan platforms
**326 total** (298 Phase 20 baseline + 28 new: 80/81/83/82 for
a/b/c/d respectively). All four platforms' new tests were adapted to
each platform's own `validPayload` conventions (SecureCredit's loan
ceilings, FinBridge's merchant fields, etc.) rather than copied
verbatim, the same care `admin.test.js`/`applications.test.js` already
take per platform.

**Dependency audit** (run for the first time this project has run
either tool):
- `pip-audit -r central-fraud-intelligence/requirements.txt` - found 29
  vulnerabilities across 3 packages. `cryptography` and
  `python-multipart` were bumped (46.0.7→50.0.1, 0.0.26→0.0.32) and the
  full central suite re-verified green afterward; `starlette` (a
  transitive dependency of the pinned `fastapi==0.115.6`, 14 advisories)
  was deliberately left as-is - see `SECURITY.md`'s Phase 22 section for
  why.
- `npm audit` re-run on all 8 Node packages (4 backends, 4 frontends):
  every backend is clean (0 vulnerabilities); every frontend reports the
  same pre-existing `esbuild`/`vite` dev-dependency finding first
  documented at Phase 1, re-confirmed and still deliberately deferred
  (see `TROUBLESHOOTING.md`'s Phase 22 entries for both).

**Not tested this phase, stated plainly**: rate limiting and CSRF
protection remain unimplemented (see `SECURITY.md`'s Planned-controls
list) - there is no control to test yet for either, only the gap itself,
which this phase re-confirmed still exists rather than silently letting
go stale. No timing-attack testing was attempted against the
constant-time comparisons already in place (`hmac.compare_digest` in
`src/security/auth.py` and `src/security/jwt_auth.py`) - a real timing
measurement needs tooling and statistical care this phase's scope didn't
include; the constant-time-comparison *code* itself was confirmed
present by reading it, which is a weaker claim than measuring it, and
this file says so rather than blurring the two.

## Phase 23: performance testing (real load, one real production concurrency bug found and fixed)

New root-level `scripts/run_performance_tests.py` - launches the real
central FastAPI app and all four loan platforms' real `run_standalone.js`
entrypoints as genuinely separate OS processes (reusing Phase 20's
process-launch conventions), then drives real, timed, bounded-concurrency
HTTP load with `httpx`+`asyncio`. See root `PERFORMANCE.md` for the full
methodology, every measured number, and the honest analysis - not
duplicated here.

**A real, pre-existing flaky test found and fixed before this phase's own
baseline was trustworthy**: `test_decrypt_tampered_ciphertext_fails`
(central) and its identical JS counterpart across all four platforms
tampered with ciphertext by hardcoding its first byte to `0xff`, which
GCM's effectively-uniform-random ciphertext bytes made a ~1-in-256
accidental no-op (confirmed empirically: 86/20000 trials had a real first
byte of `0xff`, matching the expected ~78.1). This was never a real
cryptographic weakness - only fixed by XOR-flipping the observed byte
instead of hardcoding a literal, which is mathematically guaranteed to
differ regardless of the original value (verified with a 5000-trial loop,
0 unexpected successes, across all 5 affected files).

**A real, previously-latent production concurrency bug found by this
phase's own load, not by code review**: every repository sharing this
project's lazy `ensure_schema()`/`_ensure_loaded()` pattern (six Postgres
repositories plus `GraphStore`) guarded its one-time setup with a plain,
non-thread-safe boolean. Concurrent requests that are the first to touch
a freshly-started process's shared repository instance could race into
the guard at once - reproduced directly against this sandbox's real
Postgres as a genuine `psycopg.errors.DeadlockDetected` (concurrent
`CREATE TABLE IF NOT EXISTS` statements), observed live as sporadic real
503s from `GET /api/v1/incidents` at concurrency=10. Fixed with a
per-instance `threading.Lock`, identically across all seven components.
New `tests/test_schema_race.py` (7 tests: one parametrized across the six
Postgres repository classes, one for `GraphStore`) reproduces this exact
concurrent-first-request scenario directly against the real test
database and asserts zero exceptions - a genuine regression test, not a
mock. See `PERFORMANCE.md`'s "Bugs this phase's own first runs found"
section and `PROGRESS.md`'s Phase 23 section for the full investigation
(including a ruled-out alternative hypothesis, tested and rejected, not
just assumed).

Central suite after this phase: **265 passed** (258 Phase 22 baseline + 7
new, all in `test_schema_race.py`). Loan platforms: **326 total**,
unchanged from Phase 22 (the tokenization test fix changed test *content*,
not test *count* - one existing test per platform was rewritten, none
added or removed). `VERSION` in `central-fraud-intelligence/src/config.py`
**was bumped** (0.19.0-phase22 → 0.20.0-phase23, see `config.py`'s own
updated docstring) - the `threading.Lock` fix is a real behavior change
under concurrency for every endpoint touching any of the seven fixed
components, even though no request/response JSON shape changed.

## Phase 24: Docker end-to-end verification (daemon works this session; registry access is the real, remaining blocker)

New `scripts/verify_compose.sh` - re-investigated Phase 20's "Docker
Compose is completely impossible in this sandbox" finding from scratch
rather than assuming it still applied unchanged, and found a real, more
precise picture. See `TROUBLESHOOTING.md`'s Phase 24 entry for the full
investigation; summarized here:

- Running `dockerd` directly (bypassing the `service`/systemd wrapper
  Phase 20 used, which fails for a real, unrelated reason - no systemd
  PID 1 in this container) succeeded in this session: `docker info`,
  `docker build`, and `docker run` all genuinely work. Verified with a
  real `FROM scratch` image built from a freshly `gcc -static`-compiled
  probe binary and run to completion - zero external registry involved,
  isolating "does this sandbox's Docker engine work at all" from "can
  this sandbox reach a container registry".
- Every container registry this project's Dockerfiles actually need was
  tested directly: `registry-1.docker.io` (Docker Hub), `ghcr.io`, and
  `mirror.gcr.io` all returned `403 Forbidden` through the egress proxy.
  The proxy's own diagnostic endpoint
  (`http://127.0.0.1:37829/__agentproxy/status`) logs this explicitly as
  `connect_rejected` / `"gateway answered 403 to CONNECT (policy
  denial...)"` for `registry-1.docker.io:443` - a real, confirmed
  organization-level egress policy, not a transient failure, and (per
  `/root/.ccr/README.md`) not something to route around.

**`scripts/verify_compose.sh`** runs every check this sandbox actually
can perform without needing a container registry, and reports honestly
what it skipped and why:
1. `docker compose config` - structural/YAML validity of
   `docker-compose.yml` (passes).
2. Every Dockerfile's literal `COPY` source paths actually exist in the
   repo (passes, all 5 Dockerfiles) - a real bug in this checker script
   itself was found and fixed on its own first run (it mis-parsed
   `COPY --from=<stage> /app/... ./public` as a host filesystem path
   instead of skipping the whole multi-stage-reference line, producing 4
   false positives - fixed before trusting the check).
3. Published host ports across all 10 containers are unique (passes, 10
   ports, no collisions).
4. Per-platform database network isolation, parsed directly from the
   resolved compose config: each loan platform's backend is only ever on
   its own `net-{a,b,c,d}-db` network plus the shared `net-central-bus`,
   never another platform's db network or the central app's own
   `net-central-db` (passes for real, for all four platforms - this is a
   genuine, structural, Docker-network-level enforcement of
   `ARCHITECTURE.md`'s database-isolation requirement, not merely an
   application-level convention).
5. The registry-free daemon capability probe described above (passes).
6. A real `docker compose build` - attempted only if
   `registry-1.docker.io` actually answers; in this session it does not,
   so this check is honestly reported as **SKIPPED**, not silently
   passed or silently omitted.

Separately (outside the script, since these steps need real network
access this session's proxy doesn't grant to npm/pypi's *registries* vs.
container registries - npm's and pip's package registries ARE reachable
here, unlike any container registry): every `RUN` step inside each
Dockerfile that doesn't require a container-registry pull was verified
directly - `npm install` and `npm run build` for a loan platform's
frontend (real `vite build` output produced), and `npm install
--omit=dev` for a backend (clean, 0 vulnerabilities in production
dependencies) - both exercised in a scratch location, not committed.
`pip install -r requirements.txt` for the central app has already been
exercised, extensively, by every prior phase's real pytest runs in this
same sandbox.

**Still not verified, stated plainly**: a full `docker compose up
--build` bringing up all 9 containers together and exercising the real
Mongo/Postgres-backed system end-to-end. This needs an environment with
real container-registry access, which this sandbox's organization egress
policy does not grant (confirmed, not assumed) - the honest ask, same as
every prior phase's Docker section, is for a human to run `docker compose
up --build` (or `./scripts/verify_compose.sh` itself, which will run
Check 6 for real wherever `registry-1.docker.io` is reachable) somewhere
with normal internet access.

## Phase 25: cloud deployment preparation (IaC/pipeline syntax verified; no cloud execution)

Phase 25 added no application code, so this phase adds no `pytest`/`jest`
test counts and does not change any prior phase's test totals. What was
added and how it was actually checked in this sandbox:

- `deploy/aws/terraform/*.tf` (10 files, real AWS resources for ECS
  Fargate, RDS, DocumentDB, ALB, Secrets Manager, ECR, IAM, VPC/security
  groups) - the Terraform CLI could not be installed here
  (`releases.hashicorp.com` blocked, re-confirmed this phase - see
  `TROUBLESHOOTING.md`'s Phase 25 entry), so `terraform validate` was
  never run. Instead, `pip install python-hcl2` (PyPI is reachable) and
  every `.tf` file was parsed with it: all 10 parsed as syntactically
  valid HCL2. This is real, executed evidence that the files are
  well-formed - explicitly NOT equivalent to `terraform validate` (no
  AWS-provider schema checking), stated as such in both
  `TROUBLESHOOTING.md` and `deploy/aws/terraform/README.md`.
- `.github/workflows/deploy.yml` - parsed with PyYAML (`yaml.safe_load`)
  in this sandbox and confirmed to be syntactically valid YAML. Never
  executed by GitHub Actions (no GitHub repository exists for this
  sandbox to run it against).
- `central-fraud-intelligence/Dockerfile`'s torch CPU-wheel fix - the
  existing Python test suite (265 tests as of Phase 23, unchanged - see
  below) still passes locally against the CUDA-bundled torch wheel this
  sandbox has always had (the fix changes image *build* behavior, not
  anything `import torch`-reachable at the Python level, so this sandbox's
  own test run cannot exercise the fix itself - `download.pytorch.org` is
  still blocked, confirmed by direct re-test this phase, so the fix
  remains NOT VERIFIED end-to-end, same honest status as the underlying
  issue since Phase 13).

**Regression check**: `central-fraud-intelligence`'s full suite (265
tests) and all four loan platforms' suites (326 tests total) were re-run
after this phase's non-application-code changes (`Dockerfile`,
`requirements.txt` comments, docs) to confirm nothing was accidentally
broken - all still passing, same counts as Phase 23/24 (see `PROGRESS.md`'s
Phase 25 entry for the exact command output).

## Planned test surfaces (by phase)

| Area | Phase | Location (once it exists) |
|---|---|---|
| Container health / compose startup | 1, 24 (done, real - see Phase 24 section; full `docker compose up` still needs an environment with real registry access) | `scripts/verify_compose.sh` |
| Auth (register/login/logout, JWT) | 2-5 | `loan-platform-*/backend/tests` |
| Authorization (user vs admin) | 2-5 | `loan-platform-*/backend/tests` |
| Loan application flow | 2-5 | `loan-platform-*/backend/tests` |
| Admin processing | 2-5 | `loan-platform-*/backend/tests` |
| Database isolation (A cannot reach B's DB, etc.) | 29 | `tests/test_db_isolation.*` |
| Encryption / token generation | 9 (done) | `loan-platform-*/backend/tests`, `central-fraud-intelligence/tests` |
| Event generation / outbox / retry | 7 | `loan-platform-*/backend/tests` |
| Central ingestion / idempotency | 8 (done); admin auth on GET is 9 (done); graph update wired in is 10 (done) | `central-fraud-intelligence/tests` |
| Graph creation / incremental update / entity resolution | 10 (done) | `central-fraud-intelligence/tests/test_graph.py` |
| Bounded subgraph extraction | 11 (done) | `central-fraud-intelligence/tests/test_subgraph.py` |
| Synthetic fraud scenario generation | 12 (done) | `central-fraud-intelligence/tests/test_scenario_generator.py` |
| GNN training (features, model, dataset, baseline, checkpointing) | 13 (done) | `central-fraud-intelligence/tests/test_gnn.py` |
| GNN inference (wired into the live request path) | 14 (done) | `central-fraud-intelligence/tests/test_inference.py` |
| Fraud detection rule engine + hybrid scoring (unit-level) | 15 (done) | `central-fraud-intelligence/tests/test_fraud_detection.py`, `tests/test_inference.py` |
| Fraud detection end-to-end accuracy against the full committed dataset | 15 (done, real numbers) + 21 (broader e2e) | `central-fraud-intelligence/scripts/evaluate_fraud_detection.py` + `seed-data/scenarios` (dataset real as of 12); real measured numbers in `models/hybrid_scoring_eval.json` and `ML_MODEL.md`'s Phase 15 section |
| Explainability (human-readable + technical) | 16 (done) | `central-fraud-intelligence/tests/test_explainability.py` |
| Fraud incident creation, persistence, and admin read/review endpoints | 17 (done); review workflow 19 (done) | `central-fraud-intelligence/tests/test_incidents.py`, `tests/test_incidents_review.py` |
| Notification creation, delivery worker, receiving endpoint, admin read/acknowledgement endpoints | 18 (done); acknowledgement 19 (done) | `central-fraud-intelligence/tests/test_notifications.py`, `tests/test_notifications_review.py`, `loan-platform-*/backend/tests/fraudAlerts.test.js` |
| Graph explorer read endpoints | 19 (done) | `central-fraud-intelligence/tests/test_graph_api.py` |
| Operational metrics endpoint | 19 (done) | `central-fraud-intelligence/tests/test_metrics.py` |
| Admin dashboard (server-rendered) | 19 (done) | `central-fraud-intelligence/tests/test_dashboard.py` |
| End-to-end scenarios | 20 (done, real multi-process run - see this file's Phase 20 section and `PROGRESS.md`), 30 (broader/repeatable pass) | `scripts/run_integration_demo.py` (Phase 20); `tests/test_e2e_*` not yet created - Phase 20 found real, high-value issues from a one-off run and judged a permanent root-level suite not yet worth the added maintenance surface until Phase 30 decides the final shape (see `PROGRESS.md`'s Phase 20 section for the reasoning) |
| Security (invalid JWT, replay, tampering, injection, rate limiting, secret leakage) | 22 | `tests/test_security_*` |
| Performance (processing time, no full-history reload) | 23 (done, real numbers) | `scripts/run_performance_tests.py`; real measured numbers in `performance-results/phase23_results.json` and root `PERFORMANCE.md` |
| Health checks | 1, 24 | every service's `/health` |

## Reporting convention

Every phase's checkpoint update to `PROGRESS.md` / `BUILD_STATE.json`
records `tests_passed` and `tests_failed` explicitly. A feature is never
described as "done" in `TODO.md` while its corresponding test is failing or
missing — it stays `[~]` (in progress) instead.
