# QuickFund-X — Architecture

Status: PHASE 11 complete (bounded k-hop subgraph extraction —
`GraphStore.get_relevant_subgraph()`, the first thing that reads the
persistent graph back out rather than only writing to it) — see root
`PROGRESS.md`/`TODO.md` for the phase-by-phase build log;
this file describes the design/architecture and is updated when the design
itself changes, not on every phase checkpoint, so its own "Status" line can
lag a phase or two behind `PROGRESS.md` — treat `PROGRESS.md` as the source
of truth for exactly what phase the build is on.
Last updated: 2026-09-17

## 1. What this system is

QuickFund-X is a privacy-preserving, cross-institution loan fraud intelligence
platform. Four independent, demo loan-lending platforms (each with its own
frontend, backend and database) send *privacy-minimized* application events to
ONE central Fraud Intelligence + GNN application. The central application
maintains a persistent, incrementally-growing heterogeneous graph of
pseudonymous entities (borrowers, phones, emails, devices, addresses, bank
accounts, documents) and applications, runs a Graph Neural Network plus
deterministic rule signals over that graph, produces an explainable risk
score, opens fraud incidents when warranted, and notifies only the loan
platforms actually affected.

This repository is being built incrementally across many chat sessions
(Claude Free constraints). See `PROGRESS.md`, `BUILD_STATE.json` and
`TODO.md` for the checkpoint system — **read those three files first in any
new session**, before this one.

## 2. Repository assessment (Phase 0)

The repository was empty at the start of this project (no prior code, no git
history). This is a from-scratch build. A reference design conversation
(`new.pdf`, supplied by the user) was reviewed; it is **background rationale
for the architecture**, not additional binding requirements beyond the
project brief. Key decisions it confirmed and that this document adopts:

- Do not claim a GNN operates directly on encrypted ciphertext. Raw PII stays
  inside each loan platform; only pseudonymous, HMAC-derived tokens and
  minimized metadata cross the boundary to the central application.
- Keyed hashing (HMAC-SHA256, not plain SHA-256) for deterministic
  cross-platform token matching, to resist dictionary/rainbow attacks on a
  small identifier space (e.g. 10-digit phone numbers).
- The four platforms should be differentiated demo *institutions*
  (QuickFund / TrustLend / SecureCredit / FinBridge), not four copies of one
  app, so the graph has believable inter-institution structure.
- Hybrid scoring (GNN + deterministic rule signals + temporal/cross-platform
  signals), not GNN-only, is required to avoid naive "same phone = fraud"
  false positives and to give the system a defensible explanation.
- Explainability must show *why* (contributing relationships/signals), not
  just a fraud/not-fraud label.
- Track hard negatives explicitly (shared family address/device, legitimate
  repeat applicants, common employer, shared public Wi-Fi) as first-class
  test scenarios, not an afterthought.

## 3. Final architecture

```
 loan-platform-a (QuickFund)   --\
 loan-platform-b (TrustLend)   ---\    HTTPS (TLS) + signed/HMAC-authenticated events
 loan-platform-c (SecureCredit)---->  central-fraud-intelligence  (ONE app)
 loan-platform-d (FinBridge)   --/        - secure ingestion
                                            - auth/authz
                                            - encryption/tokenization
                                            - entity resolution
                                            - persistent graph
                                            - GNN inference
                                            - fraud rules/signals
                                            - risk scoring
                                            - explainability
                                            - incident management
                                            - notification dispatch
                                            - audit log
                                            - central dashboard + graph explorer
                                            - model training/evaluation (offline)
                     <-----------------------/   notifications back to affected
                                                   platform admins only
```

Each loan platform is independently deployable (own frontend, backend,
database, Dockerfile). The central application is a single deployable unit
internally organized into modules (`src/api`, `src/security`,
`src/ingestion`, `src/tokenization`, `src/entity_resolution`, `src/graph`,
`src/gnn`, `src/fraud_detection`, `src/explainability`, `src/incidents`,
`src/notifications`, `src/audit`) — never split into separate services.

## 4. Technology choices (and why)

| Concern | Choice | Rationale |
|---|---|---|
| Loan platform backend | Node.js + Express + TypeScript-free JS (kept simple) | Fast to stand up 4x, good JWT/bcrypt ecosystem, easy outbox worker as a background interval |
| Loan platform frontend | React (Vite), 4 separate builds with distinct branding/theme | Real working UI per platform, independent deploy artifact per platform |
| Loan platform database | MongoDB, one container per platform (`mongo-loan-a` .. `mongo-loan-d`) | Matches the brief's own example naming; documents fit loan-application shape well; trivial to prove DB isolation (different connection strings/containers/networks) |
| Central application | Python 3.11 + FastAPI | GNN/ML ecosystem is Python-first (PyTorch); FastAPI gives typed APIs, async I/O, OpenAPI docs, and can also serve the dashboard's static assets from the same container/process |
| Central database | PostgreSQL, one container (`central-db`) | The brief's own Docker Compose list (section 31) enumerates exactly 10 services — 4 platforms + 4 DBs + central app + `central-db` — with **no** separate graph database container. So the persistent graph is stored as relational tables (`graph_nodes`, `graph_edges`) in Postgres and materialized into an in-memory graph object for GNN inference/entity-neighborhood queries. This keeps the footprint honest to the brief and avoids an unnecessary Neo4j container ("only add infrastructure that is genuinely necessary", section 31). Postgres also holds applications-received, events, incidents, notifications, audit log and model metrics. |
| In-memory graph representation | NetworkX (already available) as the working graph object, materialized from Postgres and updated incrementally per event | No extra service; sufficient scale for a demo/academic dataset; swappable later. As implemented in Phase 10, materialization is lazy (triggered by the first real graph operation after a process boot, with a best-effort eager attempt at FastAPI startup that never blocks/crashes if Postgres isn't up yet) rather than a blocking startup step — see `central-fraud-intelligence/src/graph/graph_store.py`'s docstring for why: an eager, blocking rehydrate at startup would violate the boot-order lesson this project learned the hard way in Phase 2 and has applied to every database dependency since (`/health` must never depend on database reachability). |
| GNN | PyTorch (CPU) with a hand-rolled lightweight heterogeneous GraphSAGE-style message-passing model (no PyTorch-Geometric/DGL native-extension dependency) — implemented and trained as of Phase 13 (`central-fraud-intelligence/src/gnn/`, `training/`) | Keeps Docker builds portable (PyG/DGL C++ extensions are a common source of build breakage across platforms); architecture confirmed (not revised) against the real graph schema/dataset and full justification vs. GAT/HGT documented in `ML_MODEL.md`'s Phase 13 section, including real measured eval results vs. a non-graph baseline |
| Event delivery (loan platform → central) | Outbox pattern: `application_outbox` collection per platform DB + an in-process polling delivery worker that POSTs to the central ingestion API with exponential backoff retry | Satisfies the outbox/retry/idempotency requirements (sections 11–12) without requiring a message-broker container, which the brief's compose list also does not include |
| Cross-platform tokenization | HMAC-SHA256 with a per-purpose key derived via HKDF from one master secret (`CENTRAL_HMAC_MASTER_KEY`), producing separate token namespaces for phone/email/device/bank/address so tokens are not correlatable across purposes by design | Matches sections 9–10, 15–16 exactly (keyed hashing, not raw SHA-256; HKDF where appropriate) |
| Payload confidentiality | TLS in transit (documented for prod; compose uses plain HTTP between containers on an isolated Docker network for local dev, TLS termination covered in `DEPLOYMENT.md`) + AES-256-GCM field-level encryption for any sensitive payload fields the central app must transiently hold before deriving tokens | Section 9 |
| Auth | JWT (HS256, per-service and per-user secrets from env) for user/admin sessions and for service-to-service platform→central auth; bcrypt for password hashing | Sections 5, 9 |
| Containerization | Docker + Docker Compose, one container per service, exactly the 10 services enumerated in the brief plus what's strictly required (see `docker-compose.yml` once Phase 1 lands) | Section 31 |

## 5. Database isolation

Each loan platform backend holds exactly one DB connection string, to its own
Mongo container, injected via its own `.env`. No platform backend is ever
given credentials or network reachability to another platform's database —
enforced both by Docker Compose network/service boundaries and by an
automated test (`tests/test_db_isolation.*`, Phase/Section 29) that attempts
a cross-platform connection and asserts it fails. The central application
never connects directly to any of the four platform databases; the only path
in is the authenticated event-ingestion API.

## 6. Persistent, incremental graph — real as of Phase 10

The graph is never rebuilt from scratch. `graph_nodes`/`graph_edges` rows
live in `central-db` (two ordinary Postgres tables, real as of Phase 10 -
`central-fraud-intelligence/src/graph/repository.py`) and are materialized
into an in-memory NetworkX `MultiDiGraph`
(`central-fraud-intelligence/src/graph/graph_store.py`) lazily - the first
real graph operation after a process boot triggers the load, with a
best-effort, never-blocking attempt also made at FastAPI startup (see the
technology-choices table above for why this deviates from "on startup" as
originally written here). Every processed `APPLICATION_SUBMITTED`/
`APPLICATION_STATUS_CHANGED` event performs `MERGE`-style upserts
(`INSERT ... ON CONFLICT ... DO UPDATE`, shallow-merging `properties`) of
nodes/edges keyed by stable token/application identifiers, so re-processing
(idempotency, section 12) is safe and prior relationships are always
preserved (section 7) - verified directly by
`central-fraud-intelligence/tests/test_graph.py`, including a dedicated
entity-resolution test (two applications from different platforms sharing
one phone token resolve to one shared PhoneToken node, not two).

Node types, real as of Phase 10: Borrower (keyed by `platformId:userId` -
a platform-local identity; cross-platform identity is what the shared
token nodes below reveal, not something collapsed into one Borrower node),
Application (keyed by the globally-unique `applicationId`; carries loan
amount/purpose/tenureMonths/status as properties directly - see "Loan",
below), Platform (keyed by `platformId`), PhoneToken, EmailToken,
AddressToken, NameToken (added beyond this section's original list, the
same way Phase 9's `tokenize.js` already tokenizes `name` beyond
`SECURITY.md`'s original table), DocumentHash.

Node types listed here originally but NOT implemented, because no loan
platform collects the underlying data yet (same "implemented, not yet
collected" status `SECURITY.md` already gives the device/bank tokenization
purposes - revisit once/if a platform actually gathers this): DeviceToken,
BankToken, IPToken. NOT implemented, and not planned to be a separate
node: Loan - the current data model is strictly one loan per application
with no independent identity of its own (no loan ID, never referenced by
anything else); manufacturing a synthetic 1:1 node/edge pair for it would
be exactly the "unnecessary infrastructure" this document's own philosophy
(section 4's central-database rationale) argues against, so loan fields
are Application node properties instead.

Relationship types, real as of Phase 10: BORROWER_SUBMITTED_APPLICATION,
APPLICATION_SUBMITTED_TO_PLATFORM, BORROWER_HAS_PHONE, BORROWER_HAS_EMAIL,
BORROWER_HAS_ADDRESS, BORROWER_HAS_NAME (new, mirrors the NameToken
addition above), APPLICATION_HAS_DOCUMENT.

Relationship types listed here originally but NOT implemented, for the
same "no data source yet" reason as the node types above: BORROWER_USES_DEVICE,
APPLICATION_USES_DEVICE, APPLICATION_USES_IP, BORROWER_HAS_BANK. NOT
implemented, and not planned as a stored edge: BORROWER_HAS_MULTIPLE_APPLICATIONS
- this is a derived signal (a borrower node with more than one outgoing
BORROWER_SUBMITTED_APPLICATION edge already encodes exactly this fact via
a graph query, not new information), better computed than double-stored;
revisit if the rule engine (Phase 15) needs it materialized differently
for performance.

Raw PII (Aadhaar/PAN/phone/email/bank/address) is never written into the
graph — only the HMAC tokens plus non-identifying metadata (platform id,
timestamps, loan amount, etc). Verified directly: `test_graph.py`'s
fixtures build events from tokens only, and nothing in
`GraphStore.apply_application_submitted`/`apply_application_status_changed`
ever reads a raw applicant field (there isn't one on the payload to read -
see `DATA_FLOW.md`'s Step 3 update from Phase 9).

**Reading the graph back out — real as of Phase 11.** Phase 10 only wrote
to the graph; nothing consumed it. `GraphStore.get_relevant_subgraph()`
(`central-fraud-intelligence/src/graph/graph_store.py`) is a bounded,
direction-agnostic breadth-first extraction of the neighborhood around one
node (normally the Application node for a just-ingested event), returned
as a plain JSON-serializable dict of nodes/edges rather than a networkx
object. Default bounds: 4 hops (the minimum needed to reach a *different*
application through one shared token — Application → Borrower → shared
token → a different Borrower → that Borrower's Application) and 500 nodes
(a safety cap so one unusually high-degree token can't pull an unbounded
slice of the graph into a single request). The result is an *induced*
subgraph — every edge between two visited nodes is included, not only the
edges BFS happened to traverse to discover them. Verified directly by
`central-fraud-intelligence/tests/test_subgraph.py`: a nonexistent node
returns `None`; an isolated node returns itself with no edges; the default
hop count provably reaches a sibling application across a shared token
while the shared token's *other* metadata one hop further does not; the
`hops` parameter measurably bounds the traversal; `max_nodes` measurably
truncates it and reports `truncated: true`; two calls with identical
arguments return identical results; and a two-shared-token case proves the
subgraph is genuinely induced, not just a BFS tree.

Two scope decisions made and documented here, not silently assumed (see
`graph_store.py`'s "Phase 11" docstring section for the full reasoning):
entity resolution stays exact-token-match — HMAC-SHA256's avalanche
property makes fuzzy/probabilistic linking on the token values themselves
cryptographically impossible without a different (weaker-privacy) token
scheme, so this isn't a shortcut, it's a direct consequence of Phase 9's
tokenization design, and is out of scope here; and no graph retention/
pruning policy is implemented, because this project's graph is expected to
stay small (a bounded synthetic demo dataset, not a production-scale
transaction graph) and pruning old shared identifiers would actively
destroy the long-lived signal fraud-ring detection depends on.
`get_relevant_subgraph()` is not yet exposed over HTTP directly — its
first real caller, as of Phase 14, is the GNN inference step described in
section 7 below (`src/api/applications.py`'s POST handler calls it, then
feeds its output to `GNNScorer.score_subgraph()`); Phase 19's
graph-explorer endpoint is still the first place an admin-facing *read* of
raw graph data is planned.

## 7. Central processing pipeline

`process_new_application(event)` — executed for every event, exactly once
logically (idempotent on `event_id`):

```
validate_event -> authenticate_source -> decrypt_permitted_payload ->
normalize_data -> generate_tokens -> resolve_entities -> update_graph ->
get_relevant_subgraph -> run_gnn_inference -> run_rule_engine ->
combine_signals -> calculate_risk -> generate_explanation ->
create_incident_if_required -> identify_affected_platforms ->
send_notifications -> persist_processing_result -> return_result
```

Real, in the live request path, through `combine_signals` as of Phase 15
(`decrypt_permitted_payload`/`normalize_data`/`generate_tokens` turned out
to be moot for this service - tokenization happens upstream on the loan
platform, Phase 9, so the event this pipeline receives already carries
tokens; `resolve_entities` is what the graph's MERGE-style upserts already
do by keying nodes on those tokens, not a separate step - see section 6).
`get_relevant_subgraph` (`GraphStore.get_relevant_subgraph()`, Phase 11)
is genuinely invoked from the live request path: once an event's graph
update has completed (status `GRAPH_UPDATED`), `src/api/applications.py`'s
POST handler calls it for the event's `application` node, then hands the
result to `src/gnn/inference.py`'s `GNNScorer.score_subgraph()` - a
trained `HeteroGraphSAGE` checkpoint (Phase 13), loaded once at startup,
run against real per-node features (`src/gnn/features.py`) and the real
bounded adjacency (`src/gnn/graph_tensors.py`, shared with offline
training so the two never silently diverge) - producing a real
fraud-probability score in [0, 1]. This whole step is deliberately
best-effort end to end: `GNNScorer.load()` never raises (a missing/broken
checkpoint just leaves `app.state.gnn_scorer` as `None`), and the
inference call itself is wrapped in a broad try/except in the request
handler, so a missing model or a scoring failure never fails an otherwise-
successful ingestion request - it just leaves the event at `GRAPH_UPDATED`
with no score, retried by the next redelivery exactly like the graph step
is. As of Phase 15, `run_rule_engine` and `combine_signals` are real too:
once an event reaches `ANALYZED` (GNN inference succeeded, or was
unavailable and left `fraudScore: null`), the same handler reuses (or, on
a later redelivery, refetches) that event's bounded subgraph, runs
`src/fraud_detection/rules.py`'s `evaluate_rules()` deterministic rule
engine against it, and combines the two scores via `src/fraud_detection/
scoring.py`'s `combine_signals()` - see section 8. This step is
best-effort the same way: a rule-engine or combination failure is caught
and logged, leaving the event parked at `ANALYZED` with the new fields
null, retried by the next redelivery. As of Phase 16, `generate_explanation`
is real too, but it is architecturally different from every step before
it: it is a pure, stateless function over the four score fields already
computed above (`src/explainability/explain.py`'s `generate_explanation()`),
not a new database/graph/model-consuming pipeline stage, and it runs at
READ TIME - called fresh on every POST/GET response, never persisted as
its own column, and not gated on `status` the way the graph/GNN/rule
steps are (it has nothing to fail against that would need a retry). See
section 8's explainability paragraph and `src/explainability/explain.py`'s
own module docstring for the full "why read time, not persisted"
reasoning. As of Phase 17, `create_incident_if_required` is real too:
whenever `status` reaches `FRAUD_DETECTED`, the same handler assembles
and persists a real incident record (`src/incidents/`) - unlike
`generate_explanation`, this IS gated on status (only `FRAUD_DETECTED`,
never `CLEARED`) and IS a new persisted table, but it follows the exact
same best-effort/retry-safe posture as the graph/GNN/rule steps: a
failure here (e.g. Postgres briefly unreachable) never unmakes the
already-final `FRAUD_DETECTED` decision, it just leaves that event
without an incident yet, retried by the next redelivery. See section 8's
new "Fraud incidents" subsection and `src/incidents/incidents.py`'s
module docstring for the full set of design decisions this makes
explicit. As of Phase 18, `identify_affected_platforms` and
`send_notifications` are real too: for every incident just created, the
same handler creates one persisted notification row per platform in that
incident's own `affected_platforms` list (`src/notifications/`) -
creation only, gated on `incident_row is not None`, following the exact
same best-effort/idempotent posture as every step before it. Delivery
itself is deliberately NOT part of this request - a background poller
(`src/notifications/worker.py`'s `NotificationDeliveryWorker`, the same
shape as every loan platform's own outbox `deliveryWorker.js` since Phase
7) picks up PENDING rows out of band and POSTs each one to the owning
platform's `POST /api/v1/fraud-notifications/receive`, so a slow or
briefly-unreachable platform can never add latency to (or fail) an
otherwise-successful ingestion request. See section 8's new "Fraud
notifications" subsection and `src/notifications/notifications.py`'s /
`src/notifications/worker.py`'s module docstrings for the full set of
design decisions this makes explicit.

Event status is persisted through: RECEIVED → VALIDATED → PROCESSING →
GRAPH_UPDATED → ANALYZED → CLEARED | FRAUD_DETECTED, with FAILED
reachable from any step. Phase 18's notification creation/delivery does
NOT add a `NOTIFICATION_SENT` event status, despite that value having
been named in this progression back when it was still entirely
speculative (pre-Phase 17) - delivery state is genuinely per-recipient
(platform A's endpoint might be up while platform C's is down), so it
lives on each `fraud_notifications` row's own `status` column
(PENDING/SENT/FAILED, mirroring `OutboxEvent`'s), never folded back onto
the single triggering event, which stays `FRAUD_DETECTED` regardless of
how many of its notifications have actually been delivered yet. As
implemented (Phase 8/10), the ingestion
endpoint persists an event straight to RECEIVED (VALIDATED/PROCESSING
aren't separately observable states given how little happens between
"authenticated + shape-valid" and "graph update attempted" - the endpoint
does both essentially in one request), then to GRAPH_UPDATED once the
graph step succeeds (`src/api/applications.py`). As of Phase 14, a third
real status value exists: ANALYZED, set (along with a real `fraud_score`
column value) once GNN inference succeeds for that event - the first
status in this progression that actually reflects a model output rather
than just "a pipeline step ran". As of Phase 15, CLEARED/FRAUD_DETECTED
are real, final status values too, set by `combine_signals()` once the
rule engine has run against that event's subgraph - see section 8 for how
that decision is actually made and `ML_MODEL.md`'s Phase 15 section for
real measured accuracy against the committed dataset (73.64% interleaved
/ 71.32% isolated - reported honestly, including the two real limitations
that explain the gap from Phase 13's 97.67% offline GNN-only number).
Phase 16's explanation generation does NOT add a new status value - it
presents whatever status already exists (including an in-progress one),
it never advances or decides one. Phase 17's incident creation also does
NOT add a new event status value - `FRAUD_DETECTED` was already real as
of Phase 15; Phase 17 adds a new *table* (`fraud_incidents`) and its own
independent `review_status` column (`OPEN` by default - see section 8's
"Fraud incidents" subsection), not a new value in this progression.

## 8. Hybrid fraud scoring

Final risk = a weighted average of the GNN fraud-probability score (Phase
14) and the deterministic rule engine's score (Phase 15):
`final_risk_score = 0.6 * gnn_score + 0.4 * rule_score`, or `rule_score`
directly, unscaled, when no GNN checkpoint is loaded (so a missing model
never structurally caps the achievable risk score below the
`FRAUD_DETECTED` threshold). `FRAUD_DETECTED` is decided at
`final_risk_score >= 0.5`, else `CLEARED`
(`src/fraud_detection/scoring.py`'s `combine_signals()`).

The rule engine (`src/fraud_detection/rules.py`'s `evaluate_rules()`)
covers every signal this project actually has data for as of Phase 15:
shared phone/email/name/address tokens across DISTINCT identities
(device/bank tokens remain undetected - no platform collects that data
yet, same gap documented since Phase 6), and shared document-hash reuse.
Application velocity is deliberately not a separate rule - it is folded
into every shared-token signal as a time-proximity multiplier (full
weight within 24h, decaying to a 0.05 floor by 30 days), because several
of this project's SUSPICIOUS patterns and GENUINE hard negatives share
identical graph topology and differ only in timing (see `ML_MODEL.md`'s
Phase 15 section). A single weak signal (e.g. shared address alone) must
not, by itself, cross the incident-creation threshold — this is enforced
by rule design (address is the lowest-weighted signal, 0.12) and verified
by hard-negative tests (`tests/test_fraud_detection.py`, and section 18).

Two honestly-reported, real limitations, both empirically confirmed (not
just asserted) in `ML_MODEL.md`'s Phase 15 evaluation section: (1) the
bounded subgraph both the rule engine and the GNN see reaches only one
"identity link" hop from the root application, so a ring connected
through a chain of two or more different shared tokens is not fully
visible from every member's perspective; (2) real-time single-pass
scoring means whichever ring member applies FIRST is, structurally,
indistinguishable from an isolated genuine applicant at the moment it is
scored - confirmed to be the sole explanation for 100% of the false
negatives in a real evaluation against the full committed dataset, in
both a realistic interleaved-traffic replay and an isolated
single-scenario replay. Neither limitation is a defect unique to this
implementation; both are documented rather than hidden, and a re-scan/
re-scoring trigger that could correct case (2) once a ring's later
members arrive is noted as a reasonable future candidate - still not
built as of Phase 16 either, whose scope was explaining decisions
honestly, not remediating this structural gap (see the explainability
paragraph below for how Phase 16 addresses both limitations within that
narrower scope).

### Explainability (Phase 16)

Every risk decision gets both a human-readable summary and a technical
breakdown (`src/explainability/explain.py`'s `generate_explanation()`),
satisfying section 2's requirement that explainability "show *why*
... not just a fraud/not-fraud label": the summary names every
`triggered_rules` signal in plain language (e.g. "its phone number
matches 2 other application(s), submitted 3 minute(s) apart") and states
the model's independent score; the technical breakdown reports each
component's actual weighted contribution to `final_risk_score` (not just
the static configured weights - the weight `combine_signals()` actually
applied, honoring its documented `None`-GNN fallback) plus a
`primaryDriver` classification (`gnn_only` / `rules_only` / `both`).

This is a pure, stateless, read-time function over the four score fields
`combine_signals()` already computes - not a new pipeline stage with its
own graph/model access or its own persisted state - see that module's
docstring for the full "why read time, not persisted" design decision.

Both of section 8's two real limitations are addressed directly within
this narrower scope (explaining honestly, not fixing): a CLEARED decision
always carries a caveat that it reflects only information available at
processing time and is not automatically revisited (limitation 2); a
FRAUD_DETECTED decision with zero triggered rules - i.e. driven entirely
by the GNN, the exact shape of Phase 15's second unresolved finding
(legitimate repeat applicants scored near-1.0) - gets an explicit caveat
naming that known limitation and recommending manual review, rather than
being presented with the same unqualified confidence as a
rule-corroborated decision.

### Fraud incidents (Phase 17)

A `FRAUD_DETECTED` decision (`status == "FRAUD_DETECTED"` exactly - see
below) creates a real, persisted incident (`src/incidents/`, a new
`fraud_incidents` Postgres table): the triggering event/application/
platform, all four score fields, the explanation snapshot (Phase 16's
`generate_explanation()` output at the moment the incident was created),
a `primaryDriver` classification carried through from that snapshot, the
list of affected applications/platforms, and a `review_status` defaulted
to `OPEN`.

What creates an incident, decided explicitly rather than left ambiguous:
only `FRAUD_DETECTED` does. A CLEARED decision just under the 0.5
threshold was considered and deliberately rejected - this project's own
evaluation data (`ML_MODEL.md`'s Phase 15 section) gives no principled
secondary threshold to pick, and a borderline CLEARED decision is already
surfaced honestly by Phase 16's explanation (every weak signal named,
the real score next to the real threshold) without a persisted incident
record implying more certainty than the score supports.

Affected applications/platforms - the concrete answer to this section's
own "can one incident span multiple applications/platforms" question,
asked back in Phase 0 (see section 9 below) - are derived from the same
bounded subgraph the rule engine/GNN step already scored for this event
(`GraphStore.get_relevant_subgraph()`, Phase 11), not a new traversal:
every Application node visible in that neighborhood, at the exact moment
this event was processed. This means an incident is a genuine point-in-
time snapshot - an application's own incident, created before a linked
application existed in the graph, lists only itself; a later linked
application's incident lists both (verified directly, not just asserted,
by `tests/test_incidents.py`'s multi-platform test).

A FRAUD_DETECTED decision driven entirely by the GNN (Phase 15's finding
2 - zero triggered rules) still creates a full incident, deliberately
not a suppressed or downgraded one: silently declining to record a real
FRAUD_DETECTED decision would hide a known limitation rather than
surface it, the opposite of Phase 16's own discipline. The incident's
`primaryDriver`/caveats (from the explanation snapshot) let a future
reviewer triage `gnn_only` incidents more closely without this project
inventing a second, lighter-weight incident type.

Idempotency mirrors event ingestion: `incident_id` is deterministic
(`INC-<triggering event's eventId>`), and `triggering_event_id` carries
a real Postgres UNIQUE constraint with the same `ON CONFLICT ... DO
NOTHING RETURNING *` pattern `src/ingestion/repository.py` already uses
- a redelivered FRAUD_DETECTED event never creates a second incident.

As of Phase 19, `review_status`'s intended lifecycle (`OPEN` /
`UNDER_REVIEW` / `CONFIRMED` / `DISMISSED` / `ESCALATED`, recorded since
Phase 17) is real: `POST /api/v1/incidents/:id/review`
(`src/api/incidents.py`) transitions it, backed by
`src/incidents/repository.py`'s `update_review_status()`. The valid
transitions mirror each loan platform's own application status-
transition validation (Phase 2's `SUBMITTED -> UNDER_REVIEW -> APPROVED
| REJECTED`, one-directional, terminal states stay terminal): `OPEN` can
reach any of the other four; `UNDER_REVIEW` can reach `CONFIRMED`/
`DISMISSED`/`ESCALATED` but not back to `OPEN` (once a human starts
reviewing, there's no legitimate "un-review", only a verdict);
`CONFIRMED`/`DISMISSED`/`ESCALATED` are terminal (409 on any further
transition attempt). Every transition appends a `{action, by, note, at}`
event to a new `review_history` JSONB column (added via the same
`ALTER TABLE ... ADD COLUMN IF NOT EXISTS` pattern `src/ingestion/
repository.py` has used since Phase 9) - this shape is not invented
fresh for incidents, it mirrors `loan-platform-a/backend/src/models/
Application.js`'s `ReviewEventSchema` field-for-field, the same proven
answer to a near-identical problem (recording who changed a
review-gated record's status, when, with what note) this project
already has.

### Fraud notifications (Phase 18)

For every incident just created, one real, persisted notification row is
created per platform in that incident's own `affected_platforms` list
(`src/notifications/`, a new `fraud_notifications` Postgres table) -
never one row per incident with an embedded platform list, because
delivery status is genuinely per-recipient (platform A's endpoint might
be up while platform C's is down), the same reasoning `OutboxEvent`
(Phase 7) already applies at the event level, one level up. Each row
carries `notification_id` (deterministic: `NOTIF-<incident_id>-
<platform_id>`), `incident_id`, `platform_id`, and the same delivery-
bookkeeping columns as `OutboxEvent` - `status` (PENDING/SENT/FAILED),
`attempts`, `last_error`, `next_attempt_at`, `sent_at`.

Creation happens synchronously inside `POST /api/v1/applications/process`
(right after incident creation, same best-effort/idempotent posture);
delivery does not. A background poller
(`src/notifications/worker.py`'s `NotificationDeliveryWorker`) picks up
PENDING rows on an interval and POSTs each one's payload
(`build_notification_payload()` - the incident's own already-computed
fields: score, `primaryDriver`, the Phase 16 explanation snapshot, and
critically the FULL `affected_applications`/`affected_platforms` lists,
so a receiving platform sees the whole ring's shape, not just its own
slice) to that platform's `POST /api/v1/fraud-notifications/receive`,
authenticated with `X-API-Key` and idempotency-keyed with
`X-Idempotency-Key: <notification_id>` - the exact same headers, and the
exact same capped-exponential-backoff retry formula
(`min(cap, base * 2^(attempts-1))`), every loan platform's own outbox
`deliveryWorker.js` has used since Phase 7, just running in the opposite
direction (central → platform instead of platform → central) and ported
to Python/httpx/asyncio. Authentication reuses each platform's own
existing shared secret (`CENTRAL_ACCEPTS_API_KEY_{A,B,C,D}` on the
central side, the same value as that platform's own
`CENTRAL_FRAUD_API_KEY`) bidirectionally, rather than provisioning a
second key-management surface for the reverse direction.

On the receiving side, each loan platform's `POST /api/v1/fraud-
notifications/receive` route (`backend/src/routes/
fraudNotifications.routes.js`) checks `X-API-Key` against its own
`CENTRAL_FRAUD_API_KEY` (a single equality check - there is exactly one
legitimate sender, unlike central's own four-platform
`resolve_platform_from_api_key()`), then persists the alert into a local
`fraud_alerts` Mongo collection (`backend/src/models/FraudAlert.js`),
keyed on `alertId` taken directly from the `X-Idempotency-Key` header
(central's own `notification_id` - the request body carries no
notification id of its own, only the incident's fields, so the header
IS this alert's identity). Creation is idempotent (upsert on `alertId`),
so a redelivery - central's worker retries until it gets a 2xx - never
creates a duplicate alert. `GET /api/v1/admin/fraud-alerts` (admin-auth
protected, reusing the existing `requireAuth`/`requireRole("admin")`
middleware) lists what this platform has received. As of Phase 19,
`POST /api/v1/admin/fraud-alerts/:alertId/review` acknowledges one -
deliberately a one-way not-acknowledged -> acknowledged boolean, not a
multi-state workflow like incident review above: an alert doesn't have a
lifecycle an admin steers, only one fact worth recording (has this
platform's own admin seen it). 409 on a second acknowledgement attempt,
the same "already final" semantics a terminal incident-review state
gets.

The central app itself exposes `GET /api/v1/fraud-notifications` and
`GET /api/v1/fraud-notifications/:id` (admin-auth protected, same shape
as `GET /api/v1/incidents(/:id)`), and, as of Phase 19,
`POST /api/v1/fraud-notifications/:id/review` - central's own
acknowledgement endpoint, symmetric to the platform-side one above and
built on the identical one-way-boolean reasoning
(`src/notifications/notifications.py`'s module docstring point 4).

### Graph explorer, metrics, and the admin dashboard (Phase 19)

`GET /api/v1/graph/application/:id` and `GET /api/v1/graph/entity/:id`
(`src/api/graph.py`) are thin, admin-authenticated read endpoints over
`GraphStore.get_relevant_subgraph()` - real and fully exercised since
Phase 11 (it's the exact function the scoring pipeline itself calls),
so this phase adds no new traversal logic, only an HTTP-layer clamp on
`hops`/`maxNodes` so one request can't pull an unbounded neighborhood.

`GET /api/v1/metrics` (`src/api/metrics.py`) is operational counts only
- event/incident/notification counts by status, graph node/edge counts
- explicitly not performance/latency data, which stays Phase 23's
concern. Every count is a live `GROUP BY`/`COUNT` query against the real
repositories, not a cached or pre-aggregated figure - appropriate at
this project's data scale, the same reasoning section 6 already gives
for never needing graph pruning/archiving.

The central app's first-ever user-facing surface is a server-rendered
Jinja2 admin dashboard (`src/web/`, everything under `/dashboard/*`) -
not a React/Vite SPA. That framework choice rests on three things
already true about this codebase, not a fresh preference: this service
has been 100% Python/FastAPI through Phase 18 (its Dockerfile has no
Node build stage at all); a Jinja2 page can be asserted against with
FastAPI's own `TestClient` (real HTML in, real HTML out), the same
"test everything for real" discipline every other phase here has been
held to, which a browser-JS test runner would break; and
`requirements.txt` had already flagged this exact fork as the open
Phase 19 question before this phase began. Auth is cookie-based but not
a second login system - the dashboard's `central_admin_token` cookie
carries the identical JWT `POST /api/v1/admin/login` already issues,
so one admin account works for both surfaces. The dashboard is a thin
presentation layer with no new business logic: incident review and
notification acknowledgement are plain HTML `<form>` POSTs to the exact
persistence paths the JSON endpoints above use, and its graph page
renders `get_relevant_subgraph()`'s own output as node/edge tables -
deliberately not a canvas/SVG visualization, a stated scope limit, not
an oversight.

## 9. Notifications

When an incident's affected applications span platforms {A, C}, only Platform
A's and Platform C's admins are notified — never platforms uninvolved in that
incident. As of Phase 17, "an incident's affected applications span
platforms {A, C}" is no longer a hypothetical from this section's
original Phase-0 wording - it's real, computed data (section 8's "Fraud
incidents" subsection). As of Phase 18, the rest of this section is real
too: section 8's new "Fraud notifications" subsection above is exactly
this - each platform exposes a receiving endpoint
(`POST /api/v1/fraud-notifications/receive`), and the central app
exposes the read side of the notification API
(`GET /api/v1/fraud-notifications(/:id)`). As of Phase 19, the last
piece of this section's original wording is real too: a notification
review/acknowledgement workflow now exists on both sides (central's
`POST /api/v1/fraud-notifications/:id/review` against
`fraud_notifications.acknowledged`, and each platform's own
`POST /api/v1/admin/fraud-alerts/:alertId/review` against its own
`fraud_alerts.acknowledged`) - see section 8's "Fraud notifications"
subsection above for the one-way-boolean design.

## 10. Documentation set

`README.md`, `ARCHITECTURE.md` (this file), `DATA_FLOW.md`, `SECURITY.md`,
`ML_MODEL.md`, `DEPLOYMENT.md`, `TESTING.md`, `DEMO_SCENARIOS.md`,
`TROUBLESHOOTING.md`, `API.md`, `PERFORMANCE.md` (added Phase 23), and
`KNOWN_ISSUES.md` (added Phase 26, consolidating every open item into one
place) were each written/expanded alongside the phase that produced their
subject matter, so they stay accurate rather than aspirational, rather
than being drafted speculatively ahead of the real work.

## 11. Phase plan

Adopted the 27-phase plan (0–26) exactly as specified in the project
brief. See `TODO.md` for the full checklist (all 27 phases complete as of
Phase 26) and `BUILD_STATE.json` for machine-readable state. Phase 0
(this document + checkpoint files) was the first completed.
