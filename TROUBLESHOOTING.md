# TROUBLESHOOTING

Status: living document — real issues actually hit during this build get
logged here as they happen, with the fix applied. This is not a generic
FAQ; every entry below either already happened in this repository or is a
known risk flagged in advance because of a concrete technical reason.

## Known risks flagged in advance (not yet hit)

- **Local Postgres does not survive a sandbox session reset.** Hit for
  real during Phase 20: mid-phase, this sandbox's environment reset (a
  new working-directory announcement appeared with no user action), and
  the local Postgres 16 service that had been running (`service
  postgresql status` → `online`) came back down (`connection to server
  at "127.0.0.1", port 5432 failed: Connection refused`, causing 143
  central test errors on the next `pytest` run). Fix is one command:
  `service postgresql start` - the on-disk data (dev + test databases)
  is unaffected, only the running service needs restarting. On any
  `CONTINUE FROM CHECKPOINT` or otherwise-unexpected central test
  failure wall of `psycopg.OperationalError: connection refused`, check
  `service postgresql status` before assuming a real regression.

- **Mongo healthcheck on first boot.** `mongo:7`'s `mongosh` may not be
  ready within the default healthcheck `start_period`; if `docker compose
  up` reports a platform container stuck waiting on its Mongo dependency,
  re-run — this is a cold-start timing issue, not a config error. If it
  recurs, increase `start_period` in `docker-compose.yml`.
- **PyTorch not yet installed.** `central-fraud-intelligence/requirements.txt`
  intentionally excludes `torch` until Phase 13, so any import of `src.gnn`
  before then will fail — expected, not a bug, until that phase lands.
- **Frontend build inside Docker vs. local dev.** The loan platform
  Dockerfiles build the React frontend and copy `dist/` into the backend's
  `public/` folder; running the backend locally without ever running
  `npm run build` in `frontend/` means `/` will show the fallback plain-HTML
  message instead of the real UI. This is expected in local dev — build the
  frontend first, or use `docker compose up` which always builds it.
- **`central-fraud-intelligence/README.md`'s endpoint-by-endpoint detail
  is stale as of Phase 22.** Noticed (not caused) this phase: that file's
  "Current status" header and file-by-file test breakdown were last
  fully written at Phase 19 and were never re-itemized for Phase 20/21's
  own new endpoints/files, even though the header line was accurate at
  the time. Phase 22 fixed the header and the top-level test count
  (now 258) and added an explicit note pointing at `PROGRESS.md`/
  `TESTING.md`/`SECURITY.md` as the current source of truth, but did NOT
  attempt a full line-by-line re-verification of that file's ~900 lines
  of endpoint detail - that's a real documentation-refresh task, flagged
  here rather than either silently left stale or risked with a rushed,
  unverified rewrite. No fixed phase assigned yet.

## Log (chronological, updated every phase)

### Phase 1 — Docker Hub is blocked in this build sandbox

`docker compose up --build` cannot complete in *this specific cloud build
environment*: the sandbox's egress policy returns `403 Forbidden` for
`registry-1.docker.io` (confirmed via the proxy status endpoint — this is an
explicit policy denial, not a transient network error), so none of
`mongo:7`, `postgres:16-alpine`, `node:22-slim` or `python:3.11-slim` can be
pulled here. `docker compose config` (which only parses/resolves the compose
file, no network needed) passes cleanly, confirming the compose file itself
is syntactically and structurally correct.

**What was verified instead, for real, in this environment:**
- `central-fraud-intelligence`: installed `requirements.txt` with pip, ran
  `uvicorn src.main:app`, hit `/health`, `/api/v1/health`, and `/` — all
  returned correct JSON. All `src/*` module packages import cleanly.
- `loan-platform-a`: `npm install` + `node src/server.js`, hit `/health` and
  `/api/v1/health` — correct JSON. Built the frontend with
  `vite build`, copied `dist/` into `backend/public/`, restarted the
  backend, and confirmed `GET /` now serves the real built QuickFund page
  (not the fallback message) — proving the exact two-stage sequence the
  Dockerfile encodes.
- `loan-platform-b/c/d`: `node --check` on `server.js` (valid syntax) and
  confirmed each `App.jsx` carries its own platform's branding
  (TrustLend/SecureCredit/FinBridge) rather than copy-paste of A's.

**What this means for you:** the application code and the Docker/Compose
definitions are real and independently verified to work; only the actual
`docker compose up --build` run itself is blocked *in this sandbox*. Please
run `docker compose up --build` and `docker compose ps` yourself (locally,
or in a CI/cloud environment with normal internet access) at your
convenience, and tell me the output — I'll fix anything that surfaces. Until
then, Docker startup is recorded as **NOT VERIFIED** (not as passing), per
this project's own no-fake-success rule.

### Phase 8 — unlike MongoDB, PostgreSQL IS reachable in this build sandbox (useful to know for future phases)

When Phase 8 needed a real database for `central-fraud-intelligence`'s
ingestion endpoint, `postgresql-16` turned out to already be installed in
this sandbox (`apt-cache policy postgresql` resolves normally — unlike
MongoDB, PostgreSQL is a normal open-source Ubuntu package, never dropped
from apt). `service postgresql start` brought a real local Postgres 16
cluster up on port 5432 with no egress/network dependency at all (this is
local package installation + a local service, not a Docker Hub pull or an
external binary download — the two things actually blocked, per the Phase
1 entry above).

**What this means for future phases**: `central-fraud-intelligence`'s
Postgres-backed tests (Phase 8 on) run against a real local Postgres
database, not an in-memory double — a stronger verification story than the
loan platforms get for MongoDB. A dev database (`central_fraud_intelligence`)
and a separate test database (`central_fraud_intelligence_test`) were
created under a `quickfundx` role (matching `docker-compose.yml`'s
`CENTRAL_DB_USER` default) — see `central-fraud-intelligence/README.md`'s
"Local dev database" section for the exact commands. This local Postgres
instance is specific to this build sandbox/session and is NOT the same as
the `central-db` Docker container `docker compose up` will start in
production — both need to work, and only the Docker one has been left
**NOT VERIFIED** (same Docker Hub blocker as everything else in the Phase 1
entry above).

### Phase 13 — download.pytorch.org is also blocked in this build sandbox (same class of issue as Phase 1's Docker Hub block)

GNN training (Phase 13) needs `torch`. The official way to install a
small, CPU-only build —
`pip install torch --index-url https://download.pytorch.org/whl/cpu` —
fails in this sandbox: the egress policy returns `403 Forbidden` (via the
proxy) for `download.pytorch.org`, the same kind of explicit policy denial
as the Phase 1 Docker Hub block, not a transient network error.

Falling back to a plain `pip install torch --break-system-packages` (the
default PyPI index, no explicit `--index-url`) *does* succeed — PyPI is
not blocked — but PyPI's current default wheel for `torch` bundles the
full CUDA 13 runtime: `torch==2.14.0+cu130` plus roughly 20 `nvidia-*`
packages (cuBLAS, cuDNN, cuFFT, cuRAND, cuSOLVER, cuSPARSE, NCCL, NVSHMEM,
nvJitLink, CUPTI, NVRTC, etc.), plus `triton`. This is ~1.2GB for `torch`
itself and several GB installed in total, versus a low-hundreds-of-MB
CPU-only wheel.

**What was verified instead, for real, in this environment:** the CUDA
build imports and runs correctly —
`python3 -c "import torch; torch.cuda.is_available()"` returns `False`
(this sandbox has no GPU, as expected) but every tensor operation used by
`src/gnn/` and `training/` runs on CPU without error, and the full GNN
test suite (see `TESTING.md`) passes against this install. So this is a
disk-space/install-size inefficiency, not a functional defect — training
in this sandbox genuinely ran, on CPU, for real, using this build.

**What this means for you:** `requirements.txt` pins `torch==2.14.0` (the
version-only pin resolves correctly against either wheel). In a real
deployment or CI environment with unrestricted egress, installing the
CPU-only wheel from `download.pytorch.org/whl/cpu` instead is recommended
— same functional result, a small fraction of the disk footprint. Nothing
about the trained model, checkpoint format, or inference code depends on
which build produced it.

**Phase 14 update — this now also affects the live request path, not just
offline training.** `src/gnn/inference.py`'s `GNNScorer` (loaded once at
FastAPI startup, see `src/main.py`'s lifespan) imports `torch` the same
way `training/` does, so every real deployment's container now carries
this same CUDA-bundled wheel unless it's rebuilt with the CPU-only
`--index-url`. That makes the recommendation above more worth acting on
than it was in Phase 13 (training was one offline run in this sandbox;
inference runs in every long-lived server process, so the several-GB image
size and the CUDA runtime's larger cold-start footprint matter more in a
real deployment) — but the underlying blocker in *this* sandbox is
unchanged (`download.pytorch.org` is still 403'd via the proxy), so this
remains **NOT VERIFIED against the CPU-only wheel** here, same as Phase
13. Functionally nothing changed: `GNNScorer.load()`/`score_subgraph()`
were exercised for real in this sandbox (see `tests/test_inference.py`)
against the same CUDA-bundled CPU-mode install, and passed.

### Phase 20 — the Docker daemon itself cannot even start in this build sandbox (stronger finding than Phase 1's registry-only block)

Phase 20 needed to actually run all five services together, so before
falling back to a non-Docker approach, `service docker start` was tried
again in case a locally-cached image or partial functionality could help.
It cannot: the Docker **daemon itself** fails to start in this sandbox —
`ulimit: error setting limit (Operation not permitted)` and `System has
not been booted with systemd as init system (PID 1). Can't operate.` —
and `docker info` reports `Cannot connect to the Docker daemon at
unix:///var/run/docker.sock`. This is a more fundamental blocker than
Phase 1's entry (which only documented `registry-1.docker.io` returning
403 through the egress proxy): even with a fully cached image and zero
network calls, no container could be started here, because there is no
running daemon to start it. Also re-confirmed this session: `curl` directly
against `registry-1.docker.io:443` and `fastdl.mongodb.org:443` both
return `connect_rejected` (organization policy) — consistent with, and
reconfirming, this entry and the Phase 13 one below it were already
correct; `apt-get install -y mongodb` still fails with "Package 'mongodb'
has no installation candidate" (no apt package exists for it on this
Debian/Ubuntu base, same finding as every prior phase that touched this).

**What this means for you:** Docker Compose remains genuinely impossible
to run in this specific build sandbox, for two independent reasons now
(registry block + daemon cannot start), not one. Please run `docker
compose up --build` and `docker compose ps` yourself, as every earlier
phase has asked — nothing here changes that recommendation, it only
explains more precisely why this sandbox itself can't do it.

**What Phase 20 did instead, and why it's still a real test, not a
simulation:** all five services were launched as genuinely separate OS
processes on real loopback TCP ports (loan platforms 4001-4004, central
on 5000) — see `scripts/run_integration_demo.py` and each platform's new
`backend/scripts/run_standalone.js`. The four loan platforms use the
exact same `createApp()` + in-memory-repository wiring this project's own
test suite has used since Phase 2 (a real MongoDB has never been
reachable in this sandbox — see the Phase 1-13 entries below), the one
difference being `app.listen()` on a real port instead of supertest
driving requests in-process, so a genuinely separate process (the central
app) reaches each platform over real HTTP. The central app is unchanged
production code, running against this sandbox's real local Postgres dev
database (see the Phase 8 entry below). This proves the real outbox
delivery workers, the real notification delivery worker, the real HTTP
contracts between all five services, and the real multi-step pipeline
end-to-end across genuine process boundaries — everything except real
MongoDB driver behavior and real Docker networking, both unchanged,
already-documented gaps. See `PROGRESS.md`'s Phase 20 section for the
real output this produced.

### Phase 20 — a real, significant pipeline gap found by this genuine end-to-end run: rule-only decisioning was unreachable when no GNN checkpoint is loaded

While running the integration demo above with the GNN checkpoint
temporarily set aside (to isolate one variable at a time), every
submitted application got stuck at `GRAPH_UPDATED` forever — no
`CLEARED`/`FRAUD_DETECTED` decision was ever made, even for a request
that shared phone+email with another application already in the graph.

The root cause: `src/api/applications.py`'s pipeline only ever advanced an
event's status to `ANALYZED` (the status the rule-engine step is gated on)
*after a GNN score was successfully computed*. When `gnn_scorer` is
`None` (no checkpoint loaded — a fully supported, documented state, see
`src/gnn/inference.py`), that block never ran, so the rule engine below
it never ran either, even though `src/fraud_detection/scoring.py`'s
`combine_signals()` has always explicitly documented handling
`gnn_score=None` gracefully ("the rule score alone determines the
outcome... RULE_WEIGHT is implicitly renormalized to 1.0") — that branch
was real, tested-in-isolation code (`test_fraud_detection.py`) that was
simply never reachable from the live endpoint. No test caught this
before Phase 20 because every existing test either forces `gnn_scorer` to
`None` on an *isolated* application (so a stuck `GRAPH_UPDATED` status
looked identical to a correct `CLEARED` one, since neither test asserted
anything different) or provides a real/stub scorer. `tests/
test_ingestion.py` even had an explicit, passing assertion from Phase 10
(`assert body["status"] == "GRAPH_UPDATED"`) that predates the rule
engine (Phase 15) and had simply never been revisited once the rule
engine made that assertion describe a bug, not a feature.

**Fixed** in `src/api/applications.py`: when `gnn_scorer` is `None`, the
event now advances straight to `ANALYZED` with `fraud_score` left `None`,
so the rule engine genuinely runs and `combine_signals()`'s
already-documented `None`-handling is actually exercised. Four tests
updated to match the corrected behavior (`tests/test_ingestion.py`,
`tests/test_graph.py` ×3, `tests/test_inference.py` — the last one's
`test_post_leaves_status_at_graph_updated_and_fraud_score_null_when_no_scorer_is_loaded`
renamed and rewritten, since its old name and docstring described the bug
as the intended behavior). Full central suite re-run afterward: 238/238
passing both with the checkpoint present and with it absent (no
regression in the `gnn_scorer is not None` path, which was untouched).
See `PROGRESS.md`'s Phase 20 section for the real demo output before and
after this fix.

### Phase 20 — a real, unresolved gap: the trained GNN checkpoint collapses to a near-zero score for genuinely live-submitted applications, regardless of input

With the gating bug above fixed and the real checkpoint back in place,
Phase 20's live demo still could not produce an incident through the GNN
path alone: every freshly-submitted application's real inference score
came back `0.0` (sigmoid of a logit around **-121 to -125**), including
applications the rule engine correctly identified as sharing phone+email
with another application already in the graph (`rule_score=0.6`).
Because `combine_signals()` uses `GNN_WEIGHT=0.6`/`RULE_WEIGHT=0.4`, a
`gnn_score` of `0.0` caps `final_risk_score` at `0.4 * rule_score`
(maximum `0.4` even at the highest possible rule score), which can never
reach the `0.5` `FRAUD_DETECTED_THRESHOLD` — so with the real checkpoint
active, live incident creation cannot fire from rule signals alone, no
matter how strong they are.

This is real and reproducible, not a fluke: it was checked directly
(bypassing the API) at two very different local-subgraph sizes for the
same application — a near-empty 21-node subgraph (cold start) and a
211-node subgraph (after seeding the same live database with the real,
already-committed 129-application Phase 12 scenario dataset via
`scripts/evaluate_fraud_detection.py`'s own `_score_interleaved()`
replay, reused directly) — both produced the same deeply negative logit.
That rules out "the graph is too small overall" as the explanation; the
collapse is specific to how a genuinely live, freshly-submitted
application's own local subgraph is built and scored, something no test
suite had exercised before, because every existing test either forces
`gnn_scorer` to `None` or substitutes a fixed-value `_StubScorer` (see
`tests/conftest.py`, `tests/test_incidents.py`) — this is the first time
in this project the real Phase 13 checkpoint has scored a real live graph
through the real `src/api/applications.py` code path end to end.

By contrast, `scripts/evaluate_fraud_detection.py`'s own offline
"interleaved" replay of the same 129-application dataset (the real
evaluation Phase 15 already ran and reported in `ML_MODEL.md`) gets 73.6%
accuracy and correctly flags 82/129 applications as `FRAUD_DETECTED` in
this same sandbox with this same checkpoint — so the model clearly *can*
separate signal from noise in that replay's own conditions. The gap
between "replaying the training-adjacent dataset offline" and "scoring
one genuinely new live request" is real, current, and unresolved.

**Not fixed in Phase 20** — deliberately: diagnosing and correcting a
trained model's behavior is Phase 13-15's domain (retraining, feature
re-engineering, or a distribution-shift fix all carry real risk of
silently changing already-evaluated, already-documented behavior), and
doing that carefully is out of scope for an integration-testing phase.
Flagged here as a high-priority open item for whichever future phase
picks up GNN work again — see `ML_MODEL.md`'s new Phase 20 section for
the full repro details and hypotheses considered. Phase 20's own
"genuine 3-platform incident" verification (see `PROGRESS.md`) was
completed with the checkpoint deliberately set aside for that one run,
exercising the now-fixed rule-only path instead — clearly labeled as
such, not presented as GNN-driven detection.

### Phase 21 — a real, unresolved gap: false-positive rate on legitimate hard negatives is much worse than Phase 15's small sample suggested, with an evidenced platform-bias hypothesis

Phase 15's original false-positive measurement rested on only 12 GENUINE
applications (9 hard negatives) - too small a sample to trust. Phase 21
re-ran the same kind of evaluation at 5x the volume (690 applications,
270 scenarios, `seed-data/scenarios/scenarios_fpr.json`) plus one new
adversarial pattern (21: the same borrower legitimately re-applying a
*third* time). The results are worse than Phase 15 suggested, not
better:

- Overall accuracy dropped to 68.12% (interleaved) / 65.94% (isolated),
  down from Phase 15's reported 73.6%.
- Hard-negative accuracy is 55.56% (75/135) - barely better than chance
  on data that is unambiguously legitimate by construction.
- The new pattern 21 (three-application legitimate repeat) is 0/45 -
  every single instance misclassified, categorically worse than pattern
  17's already-poor ~50% (a third legitimate repeat application makes
  the existing false-positive class strictly worse, not better).
- `normal_genuine_baseline` - the simplest, least ambiguous GENUINE case
  in the entire catalog - is only 33.3% accurate (5/15), and every one of
  its 10 misclassifications landed on platform B or C specifically (4/4
  and 6/6 respectively), while every platform A/D instance (2/2, 3/3) was
  correctly cleared, even though that pattern's platform assignment is
  genuinely randomized in the generator (`rng.choice(PLATFORMS)`) - so
  this isn't a generator artifact, it's evidence the trained model's
  behavior really does vary by platform.

**Well-evidenced hypothesis, not a proven root cause:** two of the
generator's hard-negative patterns (17 and the new 21) hardcode their
platform (`PLATFORMS[1]`/"b" and `PLATFORMS[2]`/"c" respectively - never
randomized, confirmed by grep). Combined with Phase 13's already-
documented severe overfitting (`finalTrainLoss: 3.86e-5`), the plausible
explanation is that the model picked up a spurious platform-based
shortcut during training rather than a genuine identity/graph-structure
signal. This is stated as a hypothesis because no ablation (retraining
with platform assignment randomized, or held out as a feature) was run
to confirm it - see `ML_MODEL.md`'s Phase 21 section for the full
evidence and reasoning, including how this connects to Phase 20's
separate live-inference-collapse finding above (both read as symptoms of
the same underlying overfitting problem, manifesting differently
depending on the input).

**Not fixed in Phase 21** — deliberately, for the same reason Phase 20's
GNN finding above wasn't fixed there: this is a measurement phase, and
retraining or otherwise altering the model's learned behavior is Phase
13-15's domain, not integration/evaluation tooling's. Flagged here as a
second high-priority open item (alongside Phase 20's) for whichever
future phase picks up GNN work again.

### Phase 1 — known, accepted npm audit finding (frontend dev dependency)

`npm audit` on each platform's `frontend/` reports 2 vulnerabilities (1
moderate, 1 high) in `esbuild`/`vite` (`GHSA-67mh-4wv8-2f99`): the Vite
*development* server can be made to relay requests/responses to an
attacker-controlled site. This affects `npm run dev` only, not the
production `vite build` output actually shipped in the Docker image, and is
a well-known, still-unpatched-without-a-breaking-upgrade issue across the
current Vite 5.x line as of this writing. Accepted for now; revisited for
real during Phase 22 (security testing), alongside a full dependency audit
of every service, not just this one.

**Phase 22 revisit**: re-ran `npm audit` for real on all four platforms'
`frontend/` (identical result on each - all four pin the same
`vite: ^5.4.11` range). Now 2 advisories (1 moderate `esbuild`, 1 high
`vite`, the high one bundling 3 separate CVE descriptions - advisory
data has grown since Phase 1 first pinned this range, not a new issue
this project introduced). `npm audit fix --force` would resolve it by
installing `vite@8.3.0` - a breaking major-version change. Deliberately
not attempted this phase: still accepted, still dev-server-only, and a
frontend build-tool major-version bump is exactly the kind of
large-blast-radius change this project treats as its own dedicated,
carefully-tested effort rather than folding into an already-large
security-testing phase - see `SECURITY.md`'s Phase 22 section for the
same reasoning applied to a parallel Python finding (`starlette`).

### Phase 22 — full dependency audit found and fixed two real Python CVEs, found and deliberately deferred a third (and two real application-code injection findings, both fixed)

`pip-audit`, run against `central-fraud-intelligence/requirements.txt`
for the first time this project has run it, found 29 known
vulnerabilities across 3 packages: `cryptography` 46.0.7 (7 CVEs, all in
X.509/PKCS7 code paths this app never calls), `python-multipart` 0.0.26
(8 CVEs, in multipart/urlencoded form parsing - genuinely relevant, since
`src/web/dashboard.py`'s login/review forms are real `Form()`-parsed
HTML posts), and `starlette` 0.41.3 (14 CVEs, a transitive dependency of
the pinned `fastapi==0.115.6`). `cryptography` and `python-multipart` are
both direct pins with narrow, well-understood usage in this codebase, so
both were bumped (50.0.1 and 0.0.32 respectively) and the full 258-test
central suite re-verified green before accepting the change.
`starlette`'s fix versions require a `starlette` major-version bump (and
likely a `fastapi` bump alongside it, since `fastapi==0.115.6` pins a
specific `starlette` range) - deliberately not attempted this phase, for
the same "large blast radius, deserves its own dedicated pass" reasoning
as the vite/esbuild finding above. See `SECURITY.md`'s Phase 22 section
for the full breakdown, including which of `starlette`'s CVE categories
are actually relevant to this app's real code paths (confirmed by grep,
not assumed) versus which touch Starlette features (`StaticFiles`,
`HTTPEndpoint`, `FileResponse`/`Range`) this app never uses at all.

Separately, real code-review-driven security testing of the four loan
platforms (MongoDB itself is unreachable in this sandbox, so these were
found by inspection and fixed defensively, then verified against the
in-memory repository double every other test in this project already
trusts - not a live exploit against a real MongoDB, consistent with this
project's standing MongoDB limitation) found two real defects, both
fixed and both covered by new tests (`tests/security.test.js`, identical
across all four platforms): `authService.js`'s `register()`/`login()`
didn't reject a non-string `email`/`password` (an object-shaped payload
like `{"email": {"$ne": null}}` would previously reach the repository
layer, where `.toLowerCase()` throwing on it was accidental luck, not a
real defense), and `adminService.js`'s `listApplications()` passed an
unvalidated `status` query parameter straight into a Mongo query filter
(`?status[$ne]=REJECTED` - Express's default query parser turns this
into a real MongoDB operator-injection shape). See `SECURITY.md`'s Phase
22 section for the full detail on both.

### Phase 23 — a real production concurrency bug, found only by real concurrent load (fixed)

This project's first genuine concurrent-load test (`scripts/
run_performance_tests.py`, Phase 23) found a real, previously-latent race
condition present since Phase 17: every repository's lazy
`ensure_schema()`/`GraphStore._ensure_loaded()` guarded its one-time
setup with a plain, non-thread-safe boolean. FastAPI dispatches sync
`def` routes to Starlette's real thread pool, so several concurrent
requests that are the first to touch a freshly-started process's shared
repository instance could all see the guard as unset at once and each
independently run `CREATE TABLE IF NOT EXISTS ...`. Reproduced directly
against this sandbox's real Postgres as a genuine
`psycopg.errors.DeadlockDetected`; observed live as sporadic real 503s
from `GET /api/v1/incidents` under concurrency=10 right after a central
restart, in 2 of this phase's first 2 full runs. An alternative
hypothesis (a `CREATE TABLE IF NOT EXISTS` race producing a
`UniqueViolation` against Postgres's own catalog) was tested directly and
ruled out - it raises a different, uncaught exception class that would
show up as an unhandled 500, not the 503 actually observed.

**Fixed** this phase with a per-instance `threading.Lock`, double-checked
against the existing guard, applied identically to all six Postgres
repositories (`src/incidents/repository.py`, `src/audit/repository.py`,
`src/graph/repository.py`, `src/ingestion/repository.py`,
`src/notifications/repository.py`, `src/security/admin_repository.py`)
plus `src/graph/graph_store.py`'s in-memory equivalent (fixed as a
precaution - no concrete failure from that one was actually observed in
this sandbox's runs, but it shares the identical unguarded
check-then-act shape). Verified fixed three ways: a direct 5-trial ×
10-thread repro against the live fix (0 errors, down from a reliable
reproduction before it); a new `tests/test_schema_race.py` (7 tests,
folded into the normal suite so this can never silently regress again);
and the final clean performance-harness run showing zero errors across
every single benchmark, including the endpoint that had never once been
clean before. See root `PERFORMANCE.md`'s "Bugs this phase's own first
runs found" section and `PROGRESS.md`'s Phase 23 section for the full
investigation.

This is flagged as a "known risk, now fixed" entry (not an open item)
specifically because it's a useful example of this project's own stated
limitation from Phase 20's TROUBLESHOOTING entry generalizing further: a
one-off integration run or a normal pytest suite (which calls
`ensure_schema()` once, sequentially, per fixture setup) will never
exercise this class of race - only genuine concurrent load does. Any
future repository added to this project that reuses this lazy-init
pattern should copy the lock, not just the boolean guard.

### Phase 23 — connection-per-repository-call is a real, identified (not yet fixed) performance characteristic

While investigating the race condition above, confirmed by reading every
repository's `_connect()` method that this project opens a brand-new
`psycopg.connect()` per repository method call, with no connection reuse
or pooling across a single request's lifetime - a single `POST
/api/v1/applications/process` call performs on the order of a
dozen-plus separate Postgres TCP-connect-and-auth round-trips (one per
`upsert_node`/`upsert_edge` call in `GraphStore.apply_application_
submitted()`, plus one per `PostgresEventRepository` call in the same
request). This is the leading, code-grounded explanation for this
phase's measured ~280-320ms single-request central-ingestion latency
floor (see `PERFORMANCE.md`) and for why throughput scales sub-linearly
with concurrency. **Deliberately not fixed this phase** - introducing
real connection pooling (e.g. `psycopg_pool`) across every repository is
an architecture-level change with a blast radius well beyond a
performance-*testing* phase's scope, the same "large blast radius,
deserves its own dedicated phase" reasoning already applied to the
`starlette`/`vite` dependency upgrades in Phase 22. Flagged here as a
concrete, well-understood optimization target for a future phase.

### Phase 24 — Docker daemon works this session; the real, confirmed blocker is a registry-access policy, not the daemon

Phase 20's entry above found the Docker **daemon itself** could not
start in that session (`service docker start` failed with `ulimit:
error setting limit (Operation not permitted)` and "System has not been
booted with systemd as init system"). Phase 24 re-investigated rather
than assuming that finding still holds unchanged, and found a real,
different result **in this session**: running `dockerd` directly (not
through the `service`/systemd wrapper Phase 20 used) starts a genuinely
working daemon - `docker info`, `docker build`, and `docker run` all
succeed for real. Verified concretely with a `FROM scratch` image built
from a freshly `gcc -static`-compiled probe binary and run to
completion, with zero container-registry involvement, so this proves
the daemon/BuildKit/containerd/runc stack itself works here, independent
of any network question.

The real, confirmed, remaining blocker is registry access, not the
daemon: `registry-1.docker.io` (Docker Hub, needed for `node:22-slim`,
`python:3.11-slim`, `postgres:16-alpine`, `mongo:7` - every base image
this project's Dockerfiles use), `ghcr.io`, and `mirror.gcr.io` (Google's
Docker Hub mirror - tried as a real alternative, not just Docker Hub
again) were all tested directly and all returned `403 Forbidden` through
this session's egress proxy. The proxy's own diagnostic endpoint
(`http://127.0.0.1:37829/__agentproxy/status`) logs the Docker Hub
attempt explicitly: `"kind": "connect_rejected", "detail": "gateway
answered 403 to CONNECT (policy denial or upstream failure)", "host":
"registry-1.docker.io:443"` - a real, named, organization-level egress
policy decision, confirmed from the proxy's own records, not inferred
from the failure alone. Per `/root/.ccr/README.md`'s own instructions
("do not retry or route around it — report the blocked host"), this was
not worked around (e.g. by rewriting the project's Dockerfiles to avoid
Docker Hub base images) - that would be an unrequested architecture
change to dodge a sandbox limitation, the same restraint this project
has applied to every other sandbox-specific blocker since Phase 1.

**Important caveat, stated honestly rather than overclaimed**: this
finding is specific to *this* session's sandbox instance and its
particular container privileges - it is not a claim that Phase 20's
finding was wrong for *that* session, or that every future session of
this same build sandbox will behave identically (sandbox capabilities
can and evidently do vary between sessions, as this phase's own result
demonstrates against Phase 20's). Whoever picks up Docker work in a
future phase should re-verify `dockerd` reachability fresh in that
session rather than assuming either this entry or Phase 20's.

**What this means for you**: `docker compose up --build` still cannot be
run to completion in this specific sandbox session - not because the
tooling is broken, but because this session's organization egress policy
blocks every container registry tested. `scripts/verify_compose.sh`
(new this phase) runs and passes every check this sandbox *can* perform
without a registry (compose-file structural validity, every Dockerfile's
referenced source paths, port-collision freedom, real per-platform
database-network isolation parsed from the resolved compose config, and
the registry-free daemon capability probe above), and honestly reports
Check 6 (`docker compose build`) as **skipped**, not passed, when
`registry-1.docker.io` is unreachable. Please run `docker compose up
--build` (or this script) yourself in an environment with real registry
access, same ask as every earlier phase - this phase narrows *why* more
precisely than before, it doesn't remove the need to do it somewhere
with normal internet access.

### Phase 25 — two more egress blocks re-confirmed (not assumed) this session; a real, if partial, syntax-check workaround found for one of them

Phase 25 (cloud deployment preparation - see `DEPLOYMENT.md`'s Cloud: AWS
section, `deploy/aws/terraform/`) needed two external resources this
session, and re-tested both directly rather than assuming a prior phase's
finding (about different hosts) still applies here:

1. **`download.pytorch.org`** (needed for the CPU-only torch wheel fix to
   `central-fraud-intelligence/Dockerfile`) - re-tested with a direct
   `curl` against `https://download.pytorch.org/whl/cpu/torch/` this
   session: still `403` via the proxy CONNECT tunnel, same
   `connect_rejected` / policy-denial signature as Phase 13's original
   finding against the same host. The Dockerfile fix (see this file's
   Phase 13 entry above and `central-fraud-intelligence/Dockerfile`'s
   Phase 25 comment) is therefore written correctly but **has never been
   executed in any sandbox this project has used** - NOT VERIFIED, not
   silently carried forward as an unfixed note for a fourth phase.
2. **`releases.hashicorp.com`** (needed to install the Terraform CLI, to
   actually run `terraform validate`/`plan` against the new IaC in
   `deploy/aws/terraform/`) - tested directly this session: also `403`,
   same `connect_rejected` signature. Per the same standing instruction as
   every prior block in this file (`/root/.ccr/README.md`: "do not retry
   or route around it"), this was reported, not worked around by e.g.
   trying to fetch a Terraform binary from an alternate mirror.

**What was actually done instead, for real** (a partial, honestly-scoped
substitute for `terraform validate` - not a replacement for it): `pip
install python-hcl2` succeeded (PyPI itself is reachable in this session,
unlike `releases.hashicorp.com`), and every `.tf` file in
`deploy/aws/terraform/` was parsed with it and confirmed syntactically
valid HCL2. This is real, executed verification - genuinely better than
zero checking - but it is explicitly **not** equivalent to `terraform
validate` (no AWS-provider schema checking: a resource block with a
misspelled or nonexistent attribute, a wrong type, or a broken
cross-resource reference would parse as valid HCL2 and still fail a real
`terraform validate` or `plan`). Stated exactly this way in
`deploy/aws/terraform/README.md` as well, not just here.

**What this means for you**: run `terraform init && terraform validate &&
terraform plan` yourself, in an environment with real internet access and
real (even placeholder/sandbox) AWS credentials, before trusting this IaC
beyond "it parses." Same category of ask as every other registry/download
block in this file - this sandbox's policy is a real, tested constraint on
this specific session, not a defect in the IaC itself.

## Phase 27 — the Phase 20 GNN live-inference collapse, root-caused and fixed for real

This session had something no earlier phase had: torch, a working
Postgres, and the already-committed `models/gnn_checkpoint.pt` all
actually working together in one sandbox, so this phase did what Phase
20/21 explicitly deferred — diagnosed and fixed the live-inference
collapse rather than only documenting it.

**Repro, run directly against the real checkpoint via `GraphStore` +
`GNNScorer` (bypassing the API, same discipline as Phase 20's own repro):**
the identical two-application graph (`REPRO-A1`/`REPRO-C1`, sharing a
phone+email token pair - the same signal shape as the project's own
documented end-to-end demo) was scored three times, changing only the
`submittedAt` on both applications:

| submittedAt | scaled timestamp feature | gnn_score |
|---|---|---|
| training epoch itself (2026-01-01) | 0.0 | 1.000 |
| one month later (in-range) | 1.03 | 0.329 |
| real live "now" (this session's real date) | 8.73 | 0.000 |

**Root cause:** `src/gnn/features.py`'s `_scaled_timestamp()` measured
minutes since a *fixed* `TIMESTAMP_EPOCH` (2026-01-01) - an
absolute-calendar-position feature. The committed training dataset
(`seed-data/scenarios/scenarios.json`) never produces a value above 5.0
for this feature (confirmed by direct measurement over every application
in the file). A genuinely live application's `submittedAt` is real
wall-clock time (`loan-platform-*/backend/src/services/
applicationService.js`'s `submittedAt: application.createdAt`), so every
day that passes after the fixed epoch pushes this feature further outside
anything the model was ever trained on, with no ceiling - this is
unbounded silent extrapolation, not a graph-size or structural problem
(consistent with Phase 20's own finding that both a 21-node and a
211-node live subgraph collapsed identically).

**First attempt (tried, measured, and found insufficient - reported
honestly rather than silently discarded):** clamping the existing
absolute feature to its measured training maximum (5.0) did NOT fix live
scoring (still 0.000 at a live timestamp clamped to 5.0). Reason: the
model had also learned to associate a *large absolute offset* specifically
with the dataset's "genuine reapplication months later" pattern (which
uses large absolute offsets by construction), so even a clamped-but-still-
large value read as "this looks like that genuine pattern," regardless of
how close together the two live applications actually were in real time.

**Actual fix:** `_scaled_timestamp()` now takes a `reference_epoch`
parameter, and `build_node_features()` computes it once per call as the
*earliest* `submittedAt` among the Application nodes actually present in
whichever node list is being scored (the whole exported graph at training
time, or one bounded live subgraph at inference time - both already go
through the one shared `build_graph_tensors()` implementation
`graph_tensors.py`'s own docstring exists specifically to keep training
and inference from silently diverging). This turns the feature into a
genuinely relative signal - "how much later than the earliest related
application was this one submitted" - which is what the feature's own
original docstring already said it needed to measure, and which stays
small and bounded regardless of the real calendar date, because it is
never compared against a fixed point in the past. Falls back to the
original fixed-epoch behavior only when fewer than two dated Application
nodes are present (nothing to compare against - the isolated-application
case, where the collapse was never the reported problem: an isolated
application scoring low is correct, not a bug).

**Verification, all real, this session:**
- `tests/test_gnn.py`'s existing 21 tests, including the ones asserting
  exact absolute-epoch feature values on single/two-node inputs, pass
  unchanged - the relative-reference design was built specifically so the
  single-node case (nothing to compare against) falls back to the exact
  original behavior those tests check.
- Full central suite re-run: 265/265 passing, no regressions anywhere
  else in the codebase.
- `scripts/evaluate_fraud_detection.py` re-run against the *same*
  unmodified checkpoint after the fix: interleaved accuracy 73.64%,
  identical to the pre-fix baseline - confirming the fix is a true no-op
  on the offline evaluation this project already trusted, not a
  regression traded for a different one.
- Live-timestamp repro re-run after the fix: the same rapid phone+email
  cluster now scores 1.000 (correctly high) at a real live timestamp
  instead of collapsing to 0.000; a second repro with the two
  applications ~5 months apart, both still dated at real live "now",
  scores 0.144 (correctly lower) - confirming the model still correctly
  distinguishes rapid-vs-spaced timing under real wall-clock conditions,
  not just always returning 1.0 regardless of input.

No retraining was needed and none was done - `models/gnn_checkpoint.pt`
and `models/gnn_checkpoint.meta.json` are byte-for-byte unchanged from
Phase 13. Only `src/gnn/features.py` changed. This directly unblocks the
one concrete gap Phase 20 flagged as blocking real incident creation from
rule signals: with this fix, `combine_signals()`'s `GNN_WEIGHT=0.6` no
longer caps every live-scored `final_risk_score` at ~0.4 regardless of
rule strength, because the GNN itself can now legitimately reach a high
score for a genuinely live, correlated cluster.

## Phase 27 (addendum) — Dockerfile missing `COPY scripts ./scripts`, fixed

A real, separate bug surfaced live during the user's own seeding session
this project window (not in this build sandbox, which cannot run
`docker compose up --build` at all - see the Docker Hub entries above):
`central-fraud-intelligence`'s admin account could not be created because
`scripts/create_admin.py` did not exist inside the running container -
`python3: can't open file '/app/scripts/create_admin.py': [Errno 2] No
such file or directory`. Root cause: `central-fraud-intelligence/Dockerfile`
copied `src/` and `models/` into the image but never `scripts/`, even
though `scripts/create_admin.py` is this service's documented,
intentional way to provision an admin account (mirroring every loan
platform's own `createAdmin.js` - no public self-registration-as-admin
endpoint, by design). Worked around live at the time with `docker cp
central-fraud-intelligence/scripts central-fraud-intelligence:/app/scripts`
(into the already-running container). Fixed for real this phase by adding
`COPY scripts ./scripts` to the Dockerfile, so a fresh `docker compose up
--build` no longer needs that workaround.
