# TODO — phase checklist

Legend: [x] done  [ ] not started  [~] in progress

- [x] **Phase 0** — Repository inspection and architecture
- [x] **Phase 1** — Project skeleton and Docker infrastructure
- [x] **Phase 2** — Loan Platform A (QuickFund) + Database A
- [x] **Phase 3** — Loan Platform B (TrustLend) + Database B
- [x] **Phase 4** — Loan Platform C (SecureCredit) + Database C
- [x] **Phase 5** — Loan Platform D (FinBridge) + Database D
- [x] **Phase 6** — Independent seed data (4 platforms)
- [x] **Phase 7** — Secure event/outbox mechanism
- [x] **Phase 8** — Central Fraud Intelligence + GNN application skeleton
- [x] **Phase 9** — Encryption/tokenization/security
- [x] **Phase 10** — Persistent graph
- [x] **Phase 11** — Incremental graph update pipeline
- [x] **Phase 12** — Synthetic fraud scenario generator
- [x] **Phase 13** — GNN training
- [x] **Phase 14** — GNN inference
- [x] **Phase 15** — Fraud rules + hybrid scoring
- [x] **Phase 16** — Explainability
- [x] **Phase 17** — Fraud incidents
- [x] **Phase 18** — Notifications
- [x] **Phase 19** — Central dashboard + graph explorer
- [x] **Phase 20** — Full four-platform integration
- [x] **Phase 21** — False-positive testing
- [x] **Phase 22** — Security testing
- [x] **Phase 23** — Performance testing
- [x] **Phase 24** — Docker end-to-end testing
- [x] **Phase 25** — Cloud deployment preparation
- [x] **Phase 26** — Final end-to-end verification

## Standing open item (since Phase 1, not blocking)

**Please run `docker compose up --build` and `docker compose ps` yourself**
(this sandbox's Docker Hub / MongoDB-binary access is blocked by egress
policy — see TROUBLESHOOTING.md) and report back what happens so any real
issue can be fixed. Everything built so far has instead been verified by
running the actual code directly (see PROGRESS.md for exact commands/
output each phase).

## Phase 6 — done, see PROGRESS.md for exactly what was verified and how

Every platform now has `backend/scripts/seed.js`, which uses the real
auth/application/admin services (not direct DB writes) to create 1 admin +
5 users + 8 applications spread across all four statuses, with
platform-appropriate loan data. Verified for real via a new
`tests/seed.test.js` per platform, run against the in-memory repositories.
Test counts: a 35/35, b 36/36, c 38/38, d 37/37. Also fixed a third
instance of the copy+sed identifier-leak bug class in loan-platform-b/c's
`createAdmin.js` usage docstrings.

## Phase 7 — done, see PROGRESS.md for exactly what was verified and how

Every platform now has a real `application_outbox` mechanism: an
`OutboxEvent` Mongoose model, a dual Mongo/in-memory `outboxRepository`,
and a real polling `deliveryWorker` (retry with capped exponential backoff,
idempotency key, API key) wired into `applicationService.submit()` and
every `adminService` status transition. Verified for real against a local
`http.createServer()` stub (not a mocked fetch): event creation, delivery +
idempotent no-redelivery, retry/backoff to FAILED, start()/stop() interval
behavior. Test counts: a 41/41, b 42/42, c 44/44, d 43/43 (170 total).
Delivery against the real central application is explicitly NOT verified
yet — that endpoint doesn't exist until Phase 8, so real delivery attempts
currently fail and retry, which is documented as correct/expected. Also
fixed a fourth instance of the copy+sed identifier-leak bug class
(`mongo-loan-a` left in loan-platform-b/c's `server.js` comments).

## Phase 8 — done, see PROGRESS.md for exactly what was verified and how

`central-fraud-intelligence` now has a real `POST /api/v1/applications/process`
(X-API-Key platform auth, Pydantic event-shape validation, idempotent
Postgres persistence via `ON CONFLICT (event_id) DO NOTHING`) and
`GET /api/v1/applications/:applicationId`. A real local Postgres 16 turned
out to be available in this sandbox (unlike MongoDB) - the 10-test suite
runs against a real `central_fraud_intelligence_test` database, no
in-memory double needed. Proactively avoided the Phase-2 boot-order bug
class (lazy DB connection). Ran a true end-to-end smoke test:
loan-platform-a's real `deliveryWorker.js` delivered a real event to a
real running central app and it was marked `SENT` - the first real
cross-service delivery in this project. Test counts: central app 10/10
(new), loan platforms unchanged (170) - 180 total. Explicitly out of
scope this phase (see central-fraud-intelligence/README.md): payload
tokenization, graph/GNN, fraud scoring, admin auth on the GET endpoint -
all still ahead.

## Phase 9 — done, see PROGRESS.md for exactly what was verified and how

Every loan platform now tokenizes (HMAC-SHA256 + HKDF, per-purpose sub-keys
from one shared `CENTRAL_HMAC_MASTER_KEY`) and encrypts (AES-256-GCM via
`CENTRAL_AES_KEY`) applicant fields *before* constructing the outbox event
- the decision from the prior checklist ("where does tokenization happen")
is resolved: upstream, on the loan platform, not on the central app on
ingestion, so raw PII never crosses the wire at all. The central app
gained a Python mirror of the same primitives plus real cross-language
parity tests (shells out to the actual Node implementation - 16/16
passing) and its own hand-rolled HS256 JWT admin auth (closing the
`GET /api/v1/applications/:id` auth gap Phase 8 left open). The interim
`X-API-Key` platform-authentication mechanism (Phase 7/8) was deliberately
left as-is - `SECURITY.md` now documents this as the intended final
mechanism for that specific hop, not something Phase 9 needed to replace.
Test counts: a 54/54, b 55/55, c 57/57, d 56/56, central 38/38 (260
total). Found and fixed a missing `import psycopg` in `src/api/admin.py`.
Closed a real Docker/production wiring gap (`CENTRAL_HMAC_MASTER_KEY`/
`CENTRAL_AES_KEY` now injected into all four loan-platform services'
environment blocks, not just the central app's).

## Phase 10 — done, see PROGRESS.md for exactly what was verified and how

`graph_nodes`/`graph_edges` relational tables (Postgres, MERGE-style
upserts) plus a NetworkX-backed `GraphStore` are real
(`central-fraud-intelligence/src/graph/`), wired into
`POST /api/v1/applications/process`: every event updates the graph and the
event's status advances to `GRAPH_UPDATED`. Entity resolution across
platforms works via shared token nodes (same phone/email/address/name
token -> same node, regardless of which platform's event created it) -
proven by a dedicated test, not just asserted. A redelivered event whose
graph step previously failed retries safely (status-gated, not
delivery-count-gated), because every upsert is idempotent. Materialization
is lazy (first real graph operation after boot), not eager-at-startup - a
deliberate, documented deviation from the original design note, to
preserve the "/health never depends on Postgres" guarantee; a best-effort
non-blocking warm-up still runs via a `lifespan` handler (which also
replaced the deprecated `@app.on_event("startup")`). Deliberately scoped
out for now, all documented in `ARCHITECTURE.md` section 6: a separate
Loan node, a stored `BORROWER_HAS_MULTIPLE_APPLICATIONS` edge, and
DeviceToken/BankToken/IPToken nodes (no platform collects that data yet).
Test counts: central-fraud-intelligence 43/43 (+5), loan platforms
unchanged (222) - 265 total.

## Phase 11 — done, see PROGRESS.md for exactly what was verified and how

`GraphStore.get_relevant_subgraph(node_type, node_key, hops=4,
max_nodes=500)` is real
(`central-fraud-intelligence/src/graph/graph_store.py`) - a bounded,
direction-agnostic BFS extraction of the neighborhood around one node,
returned as a plain JSON-serializable dict (nodes/edges), not a networkx
object. Default hops=4 is sized to be the minimum that reaches a
*different* application through one shared token (Application -> Borrower
-> shared token -> a different Borrower -> that Borrower's Application);
max_nodes=500 bounds how much a single high-degree hub token can pull in.
The result is a true induced subgraph (every edge between two visited
nodes is included, not only the ones BFS traversed to discover them) -
proven, not just asserted, by a dedicated test. Two scope decisions made
and documented rather than silently assumed: (1) entity resolution stays
exact-token-match - HMAC-SHA256's avalanche property makes fuzzy/
probabilistic linking on the token values themselves cryptographically
impossible without a weaker-privacy token scheme, so this is a direct,
documented consequence of Phase 9's tokenization design, not a shortcut
(see `SECURITY.md`'s Phase 11 note); (2) no graph retention/pruning policy
is implemented - this project's graph is expected to stay small (bounded
synthetic demo data), and pruning old shared identifiers would destroy
exactly the long-lived signal fraud-ring detection depends on. Not yet
wired to any HTTP route or called from the live ingestion request path -
there is no real consumer for its output until Phase 14's GNN inference
exists. Test counts: central-fraud-intelligence 50/50 (+7, `test_subgraph.py`),
loan platforms unchanged (222) - 272 total. This session's sandbox had
reset since the Phase 10 checkpoint (Postgres was down, loan-platform
`node_modules` were missing) - both were restored and every suite re-run
for real before trusting any prior-session number.

## Phase 12 — done, see PROGRESS.md for exactly what was verified and how

`central-fraud-intelligence/datasets/scenario_generator.py` (`generate_catalog`)
+ `scripts/generate_scenarios.py` (CLI entrypoint) are real, producing
`seed-data/scenarios/scenarios.json`: 51 scenarios, 129 synthetic
applications, covering 17 of `DEMO_SCENARIOS.md`'s 20 catalog patterns (3
deliberately deferred - #6/#7/#16 all need a DeviceToken/BankToken node
that doesn't exist yet, same reason Phase 10 gave for not implementing
those node types). Every applicant is tokenized via the real
`tokenize_applicant()` (not hand-written token strings) and every
scenario's `applications` are real `APPLICATION_SUBMITTED` event
payloads - no parallel training-data format invented. Patterns 1-3 (same
identity, N platforms) and pattern 19 (legitimate cross-platform
applicant) deliberately share identical graph topology, differing only in
`submittedAt` timing - the dataset itself demonstrates that shape alone
can't be the fraud signal. The full 129-application dataset was applied
to a real `GraphStore` against a real Postgres database with zero errors
before being committed. Test counts: central-fraud-intelligence 64/64
(+14, `test_scenario_generator.py`), loan platforms unchanged (222) - 286
total. `VERSION` deliberately not bumped (no request-handling code
changed). Updated `ML_MODEL.md` (real training-data-composition table),
`DEMO_SCENARIOS.md` (per-pattern generated/deferred status, real coverage
statement), `TESTING.md`, `central-fraud-intelligence/README.md`,
`seed-data/scenarios/README.md`.

## Phase 13 — done, see PROGRESS.md for exactly what was verified and how

`central-fraud-intelligence/src/gnn/features.py` (16-dim per-node feature
engineering, including the timestamp feature that makes Phase 12's
deliberately-identical-topology patterns 1-3 vs. 19 distinguishable at
all) + `src/gnn/model.py` (`HeteroGraphSAGE` - pure PyTorch, no PyG/DGL,
bidirectional relations, `num_layers=4` tied to Phase 11's
`DEFAULT_SUBGRAPH_HOPS`) + `training/dataset.py` (scenario-stratified
train/eval split, built via the new `GraphStore.export_all()` - the
whole-graph counterpart to Phase 11's bounded `get_relevant_subgraph()`)
+ `training/train.py` / `training/baseline.py` are all real. One real
training run (200 epochs, seed 12345) achieved 97.67% eval accuracy /
98.63% F1 on the held-out split, vs. a non-graph logistic-regression
baseline on identical features/split at 90.70% / 94.74% - real numbers,
including the one real false positive and an honest overfitting caveat,
all in `ML_MODEL.md`'s Phase 13 section (never adjusted to look better).
`torch`/`numpy`/`scikit-learn` added to `requirements.txt`, with the
`download.pytorch.org` egress block (this sandbox pulled the CUDA-bundled
PyPI default instead - functionally fine on CPU, just larger than
necessary) documented in `TROUBLESHOOTING.md`, mirroring the Phase 1
Docker Hub entry. Test counts: central-fraud-intelligence 87/87 (+21,
`test_gnn.py`, +2 in `test_graph.py` for `export_all()`), loan platforms
unchanged (222) - 309 total. `VERSION` deliberately not bumped - nothing
under `src/gnn/` is imported by `src/main.py` or any `src/api/*` router
yet (confirmed by grep), so every existing endpoint is byte-for-byte
unchanged; Phase 14 (GNN inference) is what will actually move it.

## Phase 14 — done, see PROGRESS.md for exactly what was verified and how

`central-fraud-intelligence/src/gnn/inference.py` (`GNNScorer.load()`/
`score_subgraph()`, never raises, best-effort) + new
`src/gnn/graph_tensors.py` (tensor-construction shared with
`training/dataset.py`, extracted so the two can't silently diverge) are
real and wired into the live request path: loaded once at FastAPI startup
(`src/main.py`), and `POST /api/v1/applications/process` now runs real
inference once an event reaches `GRAPH_UPDATED`, persisting a real
`fraudScore` (new `fraud_score` column) and advancing `status` to
`ANALYZED` - best-effort and retry-safe, same posture as the Phase 10
graph-update step (a missing checkpoint or scoring failure never fails
ingestion). Both `POST .../process` and `GET /api/v1/applications/:id`
now return `fraudScore` per event. Caught and fixed a status-regression
bug in the gating logic before it shipped, and a test-determinism issue
(the `client` fixture now forces `gnn_scorer` to `None` by default, since
the real checkpoint's presence on disk is environment-dependent).
`.dockerignore`/`Dockerfile` fixed to actually ship `models/` into the
Docker build context (honest caveat: `gnn_checkpoint.pt` is git-ignored,
so a fresh clone needs `training/train.py` run first - not yet verified
against a real Docker build, still blocked in this sandbox). `VERSION`
bumped `0.11.0-phase11` → `0.12.0-phase14` - the first phase since 11 to
actually change request-handling behavior. Test counts: central
101/101 (+14, `test_inference.py`), loan platforms re-verified unchanged
(222) - 323 total. Updated `ARCHITECTURE.md`, `API.md`, `DATA_FLOW.md`,
`SECURITY.md`, `TESTING.md`, `ML_MODEL.md`, `central-fraud-intelligence/
README.md`, `TROUBLESHOOTING.md`.

## Phase 15 — done, see PROGRESS.md for exactly what was verified and how

`central-fraud-intelligence/src/fraud_detection/rules.py` (`evaluate_rules()`
- deterministic, shared phone/email/name/address tokens to distinct other
identities + shared document reuse, velocity folded into every signal as
a time-proximity multiplier, hub bonus for 3+ identities; device/bank
tokens still deferred, no data source) and `src/fraud_detection/
scoring.py` (`combine_signals()` - `0.6*gnn_score + 0.4*rule_score`, or
`rule_score` alone when no GNN loaded, `FRAUD_DETECTED` at
`final_risk_score >= 0.5` else `CLEARED`) are real and wired into the live
request path, right after the Phase 14 inference step, gated on status
`ANALYZED`, best-effort/retry-safe like every prior step. Both
`POST .../process` and `GET /api/v1/applications/:id` now return
`ruleScore`/`triggeredRules`/`finalRiskScore`. `CLEARED`/`FRAUD_DETECTED`
are real status values for the first time since Phase 8. Test counts:
central 117/117 (+1 `test_inference.py`, +15 new `test_fraud_detection.py`),
loan platforms unchanged (222) - 339 total. `VERSION` bumped
`0.12.0-phase14` → `0.13.0-phase15`. Ran a real two-mode evaluation
(`scripts/evaluate_fraud_detection.py`) against all 129 applications in
the committed dataset with the real trained checkpoint: **73.64%**
accuracy interleaved / **71.32%** isolated (`models/
hybrid_scoring_eval.json`, committed) - below Phase 13's 97.67% offline
number, investigated for real: 100% of false negatives in both modes are
the first-submitted application within their own scenario (a genuine
real-time single-pass-scoring structural property, not a dataset
artifact - disproved an initial cross-scenario-timestamp hypothesis along
the way and corrected `rules.py`'s docstring accordingly), and a second,
distinct, unresolved GNN limitation surfaced (legitimate repeat
applicants scored ~0.9999 by the GNN itself in 3 scenarios) - both
reported honestly as open findings, not hidden or explained away. Updated
`ARCHITECTURE.md`, `API.md`, `DATA_FLOW.md`, `SECURITY.md`, `TESTING.md`,
`ML_MODEL.md`, `central-fraud-intelligence/README.md`.

## Phase 16 — done, see PROGRESS.md for exactly what was verified and how

`central-fraud-intelligence/src/explainability/explain.py`
(`generate_explanation()` - pure, stateless, over Phase 15's already-
computed `status`/`fraudScore`/`ruleScore`/`triggeredRules`/
`finalRiskScore`, never touches Postgres/graph/GNN, never raises) is real
and wired into both `POST /api/v1/applications/process` and
`GET /api/v1/applications/:id`. Design decision (documented in that
module's docstring): computed at READ TIME, never persisted as a new
column, and not gated on `status` at all (describes an in-progress event
honestly too) - unlike Phase 15's rule engine/combination, this is not a
new stage of the persisted state machine. Returns a human-readable
`summary` (names every triggered signal in plain language with real
timing, plus the GNN's independent score) and a `technical` breakdown
(real weighted contributions honoring `combine_signals()`'s documented
`None`-GNN fallback, a `primaryDriver` classification, a `caveats` list).
Both of Phase 15's open findings are addressed within an explicit
"explain honestly, not fix" scope: every `CLEARED` decision's caveats
name the first-arrival blind spot; every GNN-only `FRAUD_DETECTED`
decision's caveats name the legitimate-repeat-applicant limitation and
recommend manual review - neither is remediated this phase, stated
explicitly rather than left ambiguous. New `tests/test_explainability.py`
(15 tests, all passed on the first real run) covering every status/
signal combination plus live POST/GET wiring. Test counts: central
132/132 (+15), loan platforms unchanged (222) - 354 total. `VERSION`
bumped `0.13.0-phase15` → `0.14.0-phase16` (both response shapes
genuinely changed). Updated `ARCHITECTURE.md`, `API.md`, `DATA_FLOW.md`,
`SECURITY.md`, `TESTING.md`, `ML_MODEL.md`, `central-fraud-intelligence/
README.md`.

## Phase 17 — done, see PROGRESS.md for exactly what was verified and how

`src/incidents/incidents.py` (pure record assembly) + `src/incidents/
repository.py` (`PostgresIncidentRepository`, new `fraud_incidents`
table) are real, wired into `POST /api/v1/applications/process` right
after the rule-engine step: only `status == FRAUD_DETECTED` creates an
incident (a CLEARED near-miss was considered and explicitly rejected -
no principled secondary threshold this project's data would justify).
An incident genuinely can span multiple applications/platforms -
`derive_affected_applications()` reads every Application node already in
the same bounded subgraph the rule engine/GNN scored for this event, so
"affected platforms" is real, not hardcoded - proven by a test where a
linked second application's incident lists both while the first
application's own earlier incident lists only itself. A GNN-only
zero-rule-signal `FRAUD_DETECTED` decision (Phase 15's finding 2) still
opens a full incident rather than a suppressed one, carrying forward
Phase 16's manual-review caveat via an explanation snapshot. Idempotent
on `triggering_event_id` (deterministic `INC-<eventId>` incident_id,
same `ON CONFLICT DO NOTHING` pattern as event ingestion). Two new
admin-only read endpoints, `GET /api/v1/incidents` /
`GET /api/v1/incidents/:id`. `review_status` defaults `OPEN`; no
transition endpoint built this phase (explicit Phase 19 scope decision).
Test counts: central 149/149 (+17, `test_incidents.py`, all passed on
the first real run), loan platforms re-verified unchanged (222) - 371
total. `VERSION` bumped `0.14.0-phase16` → `0.15.0-phase17`.

## Phase 18 — done, see PROGRESS.md for exactly what was verified and how

`src/notifications/notifications.py` (pure `build_notification_payload()`)
+ `src/notifications/repository.py` (`PostgresNotificationRepository`, new
`fraud_notifications` table - one row per (incident, affected platform)
pair, since delivery status is genuinely per-recipient) are real, with
notification *creation* wired into `POST /api/v1/applications/process`
right after Phase 17's incident-creation step, gated on `incident_row is
not None`. *Delivery* is a genuinely new kind of thing for this project's
central app: a real background asyncio poller,
`src/notifications/worker.py`'s `NotificationDeliveryWorker` (started/
stopped in `main.py`'s lifespan), reusing Phase 7's proven outbox pattern
(capped exponential backoff, `X-Idempotency-Key`) running in the opposite
direction, authenticating itself to each platform via that platform's own
existing shared secret (no new key). Each loan platform (built fully on
loan-platform-a first, then propagated to b/c/d with identifier
substitution and independent re-verification) gained a real receiving
endpoint, `POST /api/v1/fraud-notifications/receive` (`X-API-Key` checked
against the platform's own `CENTRAL_FRAUD_API_KEY`; `alertId` taken from
`X-Idempotency-Key`, since the payload carries no id of its own), a new
`fraud_alerts` Mongo collection + repository (idempotent upsert), and a
new admin-only `GET /api/v1/admin/fraud-alerts` route. Two new central
admin-only read endpoints, `GET /api/v1/fraud-notifications` /
`GET /api/v1/fraud-notifications/:id`; deliberately did NOT build
`POST /api/v1/fraud-notifications` (redundant) or
`POST /api/v1/fraud-notifications/:id/review` (deferred to Phase 19, same
as incident review). Test counts: central 179/179 (+30,
`test_notifications.py`, all passed on the first real run), all four loan
platforms re-verified for real (+14 each, `fraudAlerts.test.js`) - 457
total. `VERSION` bumped `0.15.0-phase17` → `0.16.0-phase18`. No new
secrets introduced anywhere - both new network hops reuse existing shared
keys.

## Phase 19 — done, see PROGRESS.md for exactly what was verified and how

Incident review workflow real: `POST /api/v1/incidents/:id/review`
transitions `review_status` (`OPEN` → `UNDER_REVIEW` →
`CONFIRMED`/`DISMISSED`/`ESCALATED`, terminal states 409 on further
transition), appending `{action, by, note, at}` to a new `review_history`
JSONB column - the shape mirrors `loan-platform-a`'s own
`ReviewEventSchema`, found by reading that file for precedent.
Notification acknowledgement real, deliberately simpler (a one-way
boolean, not a state machine): `POST /api/v1/fraud-notifications/:id/
review` centrally, and the identical `POST /api/v1/admin/fraud-alerts/
:alertId/review` on each loan platform's own `FraudAlert` (built on
loan-platform-a first, propagated to b/c/d with full re-verification).
Graph explorer real, no new traversal logic: `GET /api/v1/graph/
application/:id` / `GET /api/v1/graph/entity/:id`, thin admin-gated
wrappers over `GraphStore.get_relevant_subgraph()` (real since Phase 11).
`GET /api/v1/metrics` real - operational counts only (events/incidents/
notifications by status, graph node/edge counts), explicitly not
performance data (stays Phase 23). The central app's first-ever UI: a
server-rendered Jinja2 admin dashboard (`src/web/`, `/dashboard/*`),
chosen over a React/Vite SPA because this service has been 100%
Python/FastAPI through Phase 18 and a Jinja2 page is testable directly
with FastAPI's own `TestClient` - cookie-based auth reusing the exact
same JWT the JSON API already issues, zero client-side JavaScript.
Test counts: central 238/238 (+59 across 5 new files -
`test_graph_api.py`, `test_incidents_review.py`, `test_notifications_
review.py`, `test_dashboard.py`, `test_metrics.py`), all four loan
platforms re-verified for real (+5 each, `fraudAlerts.test.js`'s
acknowledgement tests) - 536 total. `VERSION` bumped `0.16.0-phase18` →
`0.17.0-phase19`. New dependencies `jinja2`/`python-multipart`, no new
environment variables.

## Phase 20 — done, see PROGRESS.md for exactly what was verified and how

Full four-platform integration, run for real as five genuinely separate
OS processes (Docker Compose remains impossible in this sandbox - now
confirmed for a stronger reason: the daemon itself cannot start, not just
the Phase 1 registry block - see TROUBLESHOOTING.md). New `backend/
scripts/run_standalone.js` on every loan platform (real port, same
in-memory wiring the test suite uses) plus a new root-level `scripts/
run_integration_demo.py` orchestrator that drives the real
DEMO_SCENARIOS.md scenario (plus one added step) over real HTTP against
all five processes. This real run found and fixed a genuine pipeline bug
(rule-only decisioning was unreachable when no GNN checkpoint was loaded,
contradicting `combine_signals()`'s own long-standing documented
contract - fixed in `src/api/applications.py`, 4 stale test assertions
corrected to match) and surfaced a second, real, unresolved one (the real
GNN checkpoint collapses to a near-zero score for genuinely live-
submitted applications regardless of input - reproduced at two very
different graph sizes, root cause not confirmed, explicitly flagged for
a future ML phase rather than worked around silently - see ML_MODEL.md).
With the fix in place and the checkpoint deliberately set aside for one
labeled verification run, produced a real 2-platform incident (A+C,
correct `affectedApplications`/`affectedPlatforms`, correct 2-vs-0
notification fan-out) and a real 3-platform-plus incident (A+C+D, plus
B correctly included via a genuine shared-token edge), and confirmed
database isolation holds for real (Platform B's own history endpoint
does not contain Platform A's application id). Central suite re-verified
238/238 (both with and without the checkpoint present - no regression);
all four loan platforms re-verified unchanged at 298 total (536 overall,
same total as Phase 19 - this phase corrected existing test content, not
test count). Decided not to create a permanent `tests/test_e2e_*` suite
yet - deferred to Phase 30 (see TESTING.md's Phase 20 section for the
reasoning). Updated TROUBLESHOOTING.md (3 new entries), ML_MODEL.md,
TESTING.md, DEMO_SCENARIOS.md, PROGRESS.md.

## Phase 21 — done, see PROGRESS.md for exactly what was verified and how

Re-measured the false-positive rate Phase 15 first reported, at a sample
size large enough to trust (Phase 15's original measurement rested on
only 12 GENUINE applications - too thin to draw conclusions from in
either direction). `datasets/scenario_generator.py` gained an opt-in
`include_pattern_21` flag (default off, so the committed `scenarios.json`
Phase 13 trained on stays byte-for-byte unchanged) and one new
adversarial pattern - 21, a legitimate borrower re-applying to the same
platform a *third* time, extending pattern 17 specifically to stress-test
its already-known false-positive behavior. A new script, `scripts/
generate_fpr_dataset.py`, generated a separate 690-application dataset
(`seed-data/scenarios/scenarios_fpr.json`, 5x the committed dataset's
volume) verified for real against a real Postgres `GraphStore`
(690/690 events applied). Running this through the existing
`scripts/evaluate_fraud_detection.py` (real production pipeline
functions, both interleaved and isolated modes) found the false-positive
problem is real and worse than Phase 15's small sample suggested:
overall accuracy 68.12%/65.94%, hard-negative accuracy only 55.56%
(75/135), the new pattern 21 misclassified 0/45 (100% wrong - worse than
pattern 17's already-poor ~50%), and the simplest possible genuine case
(`normal_genuine_baseline`) only 33.3% accurate (5/15). A real,
evidenced (not proven) platform-bias finding: baseline's 10
misclassifications landed exclusively on platform B (4/4) or C (6/6)
even though that pattern's own platform assignment is genuinely
randomized in the generator - a well-evidenced hypothesis (not
confirmed by an ablation) is that patterns 17/21's hardcoded platform
assignment, combined with Phase 13's already-documented severe
overfitting, taught the model a spurious platform-based shortcut.
Synthesized with Phase 20's separate live-inference-collapse finding as
likely the same underlying overfitting problem manifesting differently
depending on input. One new regression test pair added (`tests/
test_scenario_generator.py`) covering the new opt-in flag and pattern
shape - the first version of one test had a wrong assumption about
calendar-month boundaries and was corrected after a real failed run (see
PROGRESS.md for the honest account). Full central suite re-verified at
240/240 (238 pre-existing + 2 new). `VERSION` deliberately not bumped -
offline evaluation tooling only, no request-handling code changed.
Updated ML_MODEL.md (full new Phase 21 section), DEMO_SCENARIOS.md
(pattern 21 added to the catalog, new addendum section, original Phase
12 coverage statement left untouched since it still accurately describes
the unchanged `scenarios.json`), TESTING.md, TROUBLESHOOTING.md (new
open item, second one alongside Phase 20's GNN finding), PROGRESS.md.

## Phase 22 — done, see PROGRESS.md for exactly what was verified and how

Real security testing, not a re-assertion of what SECURITY.md already
claimed. Confirmed correct by direct testing (not just code review):
every SQL query in the central app is genuinely parameterized (proven
with a real injection-style probe string, not just grepped), the
hand-rolled JWT rejects a forged "alg: none" token, and admin login's
anti-enumeration property is genuinely indistinguishable at the HTTP
layer. Found and fixed two real application-code defects, identical
across all four loan platforms: `authService.js` didn't reject a
non-string email/password (a real NoSQL type-confusion/operator-
injection vector against the production Mongo path, found by code
review since real MongoDB is unreachable in this sandbox); and
`GET /api/v1/admin/applications?status=...` passed an unvalidated status
filter straight into a Mongo query (`?status[$ne]=REJECTED` - a real
operator-injection shape, admin-gated so lower severity, but still a
real defect). Both fixed and covered by a new `tests/security.test.js`
per platform (7 tests × 4 = 28 new). Built real audit logging for the
first time, closing the `src/audit/` stub Phase 9 left open
("Phase 9/22" in its own docstring): `PostgresAuditRepository` with a
field-name denylist enforced at write time, wired into admin login and
both review endpoints, verified by sending a real password through a
real login and confirming it's nowhere in the stored row (10 new tests,
`tests/test_audit_log.py`). Ran a full dependency audit for the first
time this project has run either tool: `pip-audit` found 29
vulnerabilities across 3 Python packages - bumped `cryptography`
(46.0.7→50.0.1) and `python-multipart` (0.0.26→0.0.32), both verified
against the full central suite; deliberately left `starlette` alone (14
advisories, but a transitive dependency of `fastapi==0.115.6` whose fix
needs a major-version bump - reviewed for real relevance via grep, not
just counted, and judged too large a blast radius for this phase).
Re-ran `npm audit` on all 8 Node packages: every backend clean, every
frontend showing the same pre-existing esbuild/vite dev-dependency
finding first documented at Phase 1, still deliberately deferred for the
same large-blast-radius reasoning. Also deliberately left alone, stated
plainly rather than silently ignored: rate limiting and CSRF protection
for the dashboard, both re-confirmed still absent. `VERSION` bumped
(0.18.0-phase20 → 0.19.0-phase22) - admin login and the two review
endpoints now genuinely change behavior (a real audit-log write) even
though no response shape changed. Central suite: 258/258 (240 + 18 new).
Loan platforms: 326 total (298 + 28 new, 80/81/83/82 for a/b/c/d).
Updated SECURITY.md (the largest addition), TROUBLESHOOTING.md,
TESTING.md, PROGRESS.md.

## Phase 23 — done, see PROGRESS.md for exactly what was verified and how

Real performance testing, not asserted characteristics. New root-level
`scripts/run_performance_tests.py` launches the real central FastAPI app
and all four loan platforms as genuinely separate OS processes and drives
real, timed, bounded-concurrency HTTP load. Baseline verification found
and fixed a real, pre-existing flaky test (`test_decrypt_tampered_
ciphertext_fails` - a ~1-in-256 accidental tampering no-op, confirmed
empirically, never a real crypto weakness) across all 5 affected files.
This phase's own first three runs found and fixed three real bugs: a
GNN-checkpoint-absent crash in `evaluate_fraud_detection.py` (missing the
same `gnn_scorer is None` gating production has had since Phase 20); the
benchmark's own generic loan payload failing platforms B/D's real,
intentional per-platform loan-schema validation; and, most significantly,
a real, previously-latent production concurrency bug - every repository's
lazy `ensure_schema()`/`GraphStore._ensure_loaded()` guarded its one-time
setup with a non-thread-safe boolean, reproduced as a genuine Postgres
`DeadlockDetected` under concurrent first-requests to a freshly-started
process, observed live as sporadic real 503s from `GET /api/v1/incidents`
at concurrency=10. Fixed with a per-instance lock across all seven
affected components, verified by a direct repro, a new permanent
regression test (`tests/test_schema_race.py`, 7 tests), and a final clean
run with zero errors across every benchmark. Full real measured results
and honest analysis (GNN inference overhead, cold/warm graph comparison,
a code-grounded explanation for central ingestion's latency floor -
per-repository-call connection overhead, not GNN or the rule engine - and
an honestly-unexplained per-platform outlier) in new root PERFORMANCE.md.
`VERSION` bumped (0.19.0-phase22 → 0.20.0-phase23) - the concurrency fix
is a real behavior change under load. Central suite: 265/265 (258 + 7
new). Loan platforms: 326 total, unchanged (test content changed, not
count). Updated TESTING.md, TROUBLESHOOTING.md (two new entries), new
PERFORMANCE.md, PROGRESS.md.

## Phase 24 — done, see PROGRESS.md for exactly what was verified and how

Re-investigated Docker end-to-end testing for real rather than
re-asserting Phase 20's "completely impossible" finding unchanged for a
fourth phase. Found a real, more precise picture: running `dockerd`
directly (not through the `service`/systemd wrapper Phase 20 used, which
fails for a real, unrelated reason - no systemd PID 1 in this container)
starts a genuinely working daemon in this session - `docker build` and
`docker run` both verified for real with a registry-free `FROM scratch`
probe image. The real, confirmed, remaining blocker is container-registry
access, not the daemon: `registry-1.docker.io`, `ghcr.io`, and
`mirror.gcr.io` were all tested directly and all returned 403, confirmed
from the proxy's own diagnostic log as a named organization egress-policy
denial - not worked around, per that proxy's own "do not route around it"
instruction. New `scripts/verify_compose.sh` runs every check this
sandbox can perform without a registry (compose-file structural validity,
every Dockerfile's COPY source paths, port-collision freedom, real
per-platform database-network isolation parsed from the resolved compose
config - a genuine Docker-network-level enforcement of
`ARCHITECTURE.md`'s isolation requirement, not just an application-level
convention - and the registry-free daemon probe) and honestly reports the
one check that needs a registry as skipped, not passed. A real bug in
this checker script itself (a `COPY --from=<stage>` multi-stage reference
mis-parsed as a host path) was found and fixed on the script's own first
run before trusting its output. Every Dockerfile `RUN` step that doesn't
need a container-registry pull was separately verified to actually
succeed (`npm install`/`npm run build` for a frontend, `npm install
--omit=dev` for a backend). `VERSION` not bumped - no `src/` changes, only
a new verification script and documentation. Updated TESTING.md,
TROUBLESHOOTING.md (explicitly reconciled against Phase 20's differing
finding, not silently overwritten), PROGRESS.md.

## Phase 25 — done

Chose AWS (fresh decision - no provider was ever named anywhere in this
project before this phase, confirmed by a full-repo grep). Wrote real,
reviewed Terraform IaC (`deploy/aws/terraform/`, 10 files: VPC, 11
security groups reproducing `docker-compose.yml`'s 6-network isolation
topology exactly, ECS Fargate x5, RDS Postgres, 4 DocumentDB clusters,
ALB with TLS termination, Secrets Manager, ECR, least-privilege IAM) and
a CI/CD pipeline (`.github/workflows/deploy.yml` - build/push to ECR +
`terraform plan` on every push, `apply` gated behind a disabled/
manually-approved job). None of it was applied or even `terraform
validate`d - no AWS account exists in this sandbox, and
`releases.hashicorp.com` is blocked (confirmed, not assumed) so the
Terraform CLI couldn't even be installed; real substitute check actually
run instead: every `.tf` file parsed as syntactically valid HCL2 via
`python-hcl2`, the workflow YAML parsed via PyYAML - both real, both
explicitly weaker than full validation, reported as such everywhere
(`DEPLOYMENT.md`, `TROUBLESHOOTING.md`, `TESTING.md`,
`deploy/aws/terraform/README.md`). Fixed the Phase 13/14 torch/CUDA
image-bloat issue for real in `central-fraud-intelligence/Dockerfile`
(CPU-only wheel via `--extra-index-url .../whl/cpu`, installed before the
rest of `requirements.txt`) - re-tested `download.pytorch.org` this
phase, still blocked, so this fix is correct-per-documentation but NOT
VERIFIED end-to-end, stated plainly rather than silently repeated
unfixed. `DEPLOYMENT.md`'s Phase 1 stub fully resolved (container
platform, secrets management, CORS, health checks - all now concrete, not
placeholder prose). `SECURITY.md`'s two previously-open "pending a real
deployment target" / "deferred" bullets (TLS, key storage) now have a
concrete design, with rotation *procedure* still honestly left open.
`VERSION` not bumped - no `src/` changes. Full regression re-run for
real this phase (a fresh session needed `service postgresql start` first,
a known characteristic, not a new finding): central 265/265, loan
platforms 80+81+83+82=326/326, identical to Phase 23/24 - no regression
from this phase's Dockerfile/doc/IaC changes. This phase inherits, and
does not resolve, every real open item prior phases already carried
forward: the GNN live-inference collapse (Phase 20), the false-positive/
platform-bias finding (Phase 21), deferred `starlette`/`vite` upgrades
and still-absent rate-limiting/CSRF (Phase 22), no-connection-pooling
(Phase 23), and the still-never-executed full `docker compose up --build`
(Phase 24) - none of these were in Phase 25's scope (cloud deployment
prep), and none are silently dropped; see below for Phase 26.

## Phase 26 — done (all 27 planned phases now complete)

Final end-to-end verification, the closing phase of this project's
original 27-phase plan (Phase 0-26). Re-ran every automated test suite
fresh, one more time, in this session: central-fraud-intelligence
265/265, loan platforms a=80/b=81/c=83/d=82 = 326/326 - **591/591 total
across the project**, the actual final verification this phase's name
promises, recorded in `PROGRESS.md`'s closing entry.

Ran a full consistency pass across every doc and found (real, minor,
documentation-only) drift: a repeated "9 containers" vs the real
10-service `docker-compose.yml` topology in `TESTING.md`/`PROGRESS.md`/
`BUILD_STATE.json`'s own `known_errors`; `README.md`'s and
`ARCHITECTURE.md`'s Phase-1-era "in progress"/"once later phases..."
language, now updated to reflect all 27 phases being complete (while
still stating plainly that "complete" means planned scope, not zero
issues); `DEPLOYMENT.md`'s Docker Compose section still saying "once
every phase through Phase 24 is complete" and "as of Phase 1... carry no
business logic yet"; a broken `ARCHITECTURE.md section 3/31` citation (no
section 31 exists). All fixed. `BUILD_STATE.json`'s stale
`environment_verified.docker` field (still reflecting Phase 20's
superseded finding) also fixed to the current, re-verified state.

Created `KNOWN_ISSUES.md` - the final, consolidated, honestly-labeled
list of every real open item from 26 phases (GNN live-inference collapse,
false-positive/platform-bias, deferred security hardening, no connection
pooling, never-executed `docker compose up --build`/cloud deployment,
MongoDB never reachable in any sandbox), organized by category, with
historical findings explicitly distinguished from currently-open ones.

No application code was changed this phase, deliberately - Phase 26 is
verification and documentation consistency, not a catch-all bugfix phase;
every item in `KNOWN_ISSUES.md` remains exactly as open as before this
phase, now honestly consolidated rather than fixed or hidden. `VERSION`
not bumped (no `src/` changes).

**Project status: all 27 planned phases (0-26) complete.** This build
plan has no further phase defined. See `KNOWN_ISSUES.md` for what a team
continuing this project for real should pick up next, and its closing
section for what a team taking this to real production use needs first.
