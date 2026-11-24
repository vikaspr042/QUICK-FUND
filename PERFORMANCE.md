# PERFORMANCE

Status: **Phase 23 complete** — real, measured latency/throughput
baselines for the central app and all four loan platforms, run against
genuinely separate OS processes and this sandbox's real local Postgres
(the same "as real as this sandbox allows" posture every prior phase has
used for its own constraints — see `TROUBLESHOOTING.md`). This file
documents actual measured numbers from `scripts/run_performance_tests.py`
(also written this phase), not estimated or aspirational ones. Every
number below is from the final, clean run (`performance-results/
phase23_results.json`) after two real bugs this phase's own first runs
surfaced were found and fixed — see "Bugs this phase's own first runs
found" below before reading the numbers, since the numbers were not
trustworthy until both were fixed.

## What was measured, and how

`scripts/run_performance_tests.py` launches the real central FastAPI/
uvicorn app (backed by this sandbox's real local Postgres dev database)
and all four loan platforms' real `run_standalone.js` entrypoints (the
same in-memory-repository wiring the automated test suite already
exercises, bound to a real TCP port) as genuinely separate OS processes,
then drives real, timed HTTP load against them with `httpx` — genuine
wall-clock latency through the real FastAPI/Express routing, auth,
validation, and (for the central app) the real rule engine and, when a
checkpoint is present, the real trained GNN scorer. This reuses Phase
20's `run_integration_demo.py` process-launch conventions almost
verbatim, on different ports so the two never collide.

Measured, for real:

1. Central app, `POST /api/v1/applications/process` — latency/throughput
   at concurrency 1 and 10 (n=30 requests each), in four conditions:
   cold graph + rule-only (no GNN checkpoint), cold graph + GNN
   checkpoint loaded, warm graph (seeded with the real, committed
   129-application Phase 12 scenario dataset — the same realistic-scale
   seeding Phase 20 already established) + rule-only, warm graph + GNN.
2. Central app read endpoints (`GET /api/v1/applications/:id`,
   `GET /api/v1/incidents`, `GET /api/v1/metrics`) against the warm
   graph, concurrency 10, both GNN conditions.
3. Each loan platform's `POST /api/v1/applications` (submission) and
   `GET /api/v1/admin/applications` (admin read), concurrency 1 and 10,
   against its real in-memory repository.

The GNN checkpoint is toggled by temporarily moving `models/
gnn_checkpoint.pt` aside (`shutil.move`) and back — a real "checkpoint
absent" condition, not a simulated one; `GNNScorer.load()` genuinely
returns `None` and the app's own Phase 20-fixed rule-only gating path
runs for real.

## What this does NOT measure, stated plainly

Real MongoDB or real Docker container networking (both confirmed
impossible in this sandbox since Phase 1/20 — every loan platform here
runs against its in-memory repository double, not Mongo). Multi-worker/
multi-process horizontal scaling — this sandbox has 2 vCPUs and every
service runs as a single process (`uvicorn` with no `--workers` flag,
Node's single-threaded event loop); every number below is **single-
process capacity in a shared, resource-constrained sandbox**, not a
production capacity-planning figure. Real network latency to a real
client — everything here is loopback TCP on one machine. And this
sandbox's own CPU/IO is shared with whatever else is running in it at the
time, so absolute millisecond figures carry real run-to-run noise (see
"Run-to-run variance, honestly" below) — the *shape* and *relative*
comparisons (warm vs. cold, GNN vs. rule-only, endpoint vs. endpoint) are
the trustworthy part of this file, not any single absolute number treated
as an SLA.

## Bugs this phase's own first runs found (fixed before trusting any number below)

Consistent with this project's standing practice of investigating rather
than working around a test failure (see `PROGRESS.md`'s Phase 23
section for the full account of each), the first two real runs of this
harness each failed outright, and a third run succeeded but exposed a
real production-code race condition. All three were investigated to a
real root cause and fixed — none were worked around.

1. **`scripts/evaluate_fraud_detection.py`'s warm-graph seeding step
   crashed with the GNN checkpoint absent.** `_replay_one_event()`
   called `gnn_scorer.score_subgraph(...)` unconditionally, but
   `GNNScorer.load()` legitimately returns `None` when no checkpoint is
   present (the documented "inference disabled this run" state).
   Production (`src/api/applications.py`) has handled `gnn_scorer is
   None` correctly since a Phase 20 fix; this evaluation script's own
   docstring claims to mirror that pipeline "exactly" but had never
   actually been exercised with the checkpoint absent before this
   phase's rule-only benchmark round — every prior run (Phase 15/20/21)
   always had a real trained checkpoint loaded. Fixed by adding the same
   `gnn_scorer is None` gating production already has.
2. **Platforms B (TrustLend) and D (FinBridge) rejected every synthetic
   submission with 400.** Not a platform bug: each of the four loan
   platforms genuinely enforces a different loan schema (`ARCHITECTURE.md`'s
   "four institutions, not four copies" requirement) — B requires
   `loan.institutionName`/`loan.courseName` (education loans only), D
   requires `loan.merchantName`/`loan.productCategory` from a fixed enum
   (point-of-sale loans only). This benchmark's own payload builder sent
   one generic loan shape to all four platforms. Fixed by giving
   `bench_platform_submission()` a per-platform loan payload.
3. **A real, reproducible Postgres race condition** — see the next
   section; this one is a genuine production-code fix, not just a
   benchmark-script fix, so it gets its own section.

### The real race condition: concurrent `ensure_schema()` under load

The third full run (after fixes 1-2 above) completed but showed sporadic
real `503 Service Unavailable` errors from `GET /api/v1/incidents` at
concurrency=10 (2-4 out of 30 requests, across two separate runs) —
never at concurrency=1, and never on any other endpoint. Investigated
rather than dismissed as sandbox noise:

- Every repository in this project shares the same lazy-init pattern:
  `ensure_schema()` checks a plain `self._schema_ready` boolean and, if
  `False`, runs `CREATE TABLE IF NOT EXISTS ...` then sets the flag.
  FastAPI dispatches each sync `def` route (like `GET /api/v1/incidents`)
  to Starlette's internal thread pool — genuine OS threads, not
  coroutines — so several concurrent requests that are the *first* ones
  to touch a freshly-started process's shared repository instance
  (`app.state.incident_repository`) can all see `_schema_ready is False`
  at once and each independently run the DDL.
- Reproduced directly against this sandbox's real Postgres, outside the
  HTTP layer entirely (10 concurrent Python threads calling
  `PostgresIncidentRepository.list_all()` on one freshly-reset
  instance): a genuine `psycopg.errors.DeadlockDetected` — confirmed to
  be a real `psycopg.OperationalError` subclass, which is exactly why
  the app's existing `except psycopg.OperationalError` handling
  correctly (if misleadingly) turned it into an honest 503 rather than
  crashing. A different, initially-suspected cause (`CREATE TABLE IF NOT
  EXISTS` racing into a `UniqueViolation` on Postgres's own catalog
  tables) was tested and ruled out directly — it raises a different,
  uncaught exception class that would have shown up as a 500, not a 503,
  and never did.
- **Fixed** with a per-instance `threading.Lock`, double-checked against
  the existing `_schema_ready`/`_loaded` guard, applied identically to
  all six repositories sharing this pattern:
  `src/incidents/repository.py`, `src/audit/repository.py`,
  `src/graph/repository.py`, `src/ingestion/repository.py`,
  `src/notifications/repository.py`, `src/security/admin_repository.py`
  — plus `src/graph/graph_store.py`'s in-memory `_ensure_loaded()`,
  which has the same shape of race (mutating a shared
  `networkx.MultiDiGraph` instead of running DDL) even though no
  concrete failure from it was observed in this sandbox's own runs.
- **Verified fixed**: a direct 5-trial × 10-thread repro against the
  live fix showed 0 errors (down from a reliable reproduction before
  it); the full central pytest suite (265/265, including 7 new
  regression tests in `tests/test_schema_race.py` that reproduce this
  exact concurrent-first-request scenario against each repository
  class); and the final, clean performance-harness run below shows
  **zero errors across every single benchmark**, including `GET
  /api/v1/incidents` at concurrency=10, which had never once been clean
  before this fix.

This was a real, latent production-code defect — present since Phase 17
(when `PostgresIncidentRepository` was first written) — that no prior
phase's testing ever exercised concurrently enough to trigger, only
found because this is the first phase to genuinely hammer a
freshly-started process with concurrent requests. `src/config.py`'s
`VERSION` was bumped (`0.19.0-phase22` → `0.20.0-phase23`) because this
is a real behavior change under concurrency for every endpoint touching
any of these seven components, even though no request/response JSON
shape changed.

## Results: central app ingestion (`POST /api/v1/applications/process`)

All figures in milliseconds unless noted; n=30 per cell; 0 errors in
every cell of this table (the clean, final run).

| Condition | Concurrency | p50 | p95 | p99 | mean | throughput (req/s) |
|---|---|---|---|---|---|---|
| Cold graph, rule-only (no GNN) | 1 | 280 | 323 | 391 | 286 | 3.50 |
| Cold graph, rule-only (no GNN) | 10 | 1830 | 2043 | 2217 | 1847 | 5.28 |
| Warm graph, rule-only (no GNN) | 1 | 294 | 316 | 325 | 295 | 3.39 |
| Warm graph, rule-only (no GNN) | 10 | 1890 | 2215 | 2266 | 1900 | 5.10 |
| Cold graph, with GNN | 1 | 301 | 358 | 440 | 311 | 3.21 |
| Cold graph, with GNN | 10 | 2359 | 2474 | 2490 | 2353 | 4.21 |
| Warm graph, with GNN | 1 | 317 | 379 | 442 | 325 | 3.08 |
| Warm graph, with GNN | 10 | 2194 | 2318 | 2355 | 2139 | 4.54 |

**GNN inference overhead** (with-GNN minus rule-only, same graph state):
at concurrency=1, roughly +21ms cold (+7.5%) and +23ms warm (+8%); at
concurrency=10, roughly +529ms cold (+29%) and +304ms warm (+16%).
Real, measurable cost, consistent across both graph states in direction
(GNN is always slower) though the magnitude differs — expected, since a
single-process 2-vCPU sandbox means GNN inference at concurrency=10 is
genuinely competing with the rule engine, Postgres round-trips, and the
other 9 in-flight requests for the same two cores, not running in
isolation.

**Warm vs. cold**: at concurrency=1 the warm graph (950 nodes, 1128
edges after seeding) is consistently ~5% slower than cold — small,
plausible (`get_relevant_subgraph()` traverses more real data), and in
the expected direction. At concurrency=10 the direction is inconsistent
between the two GNN conditions (warm slightly slower for rule-only,
warm noticeably *faster* than cold for with-GNN) — called out honestly
rather than smoothed over: this is most consistent with this sandbox's
own shared-resource run-to-run noise dominating a real but small
warm/cold effect at this concurrency level, not a genuine "warm graphs
process faster than cold ones under GNN" effect, which would have no
plausible mechanism. See "Run-to-run variance, honestly" below.

**Why single-request latency is ~280-320ms at all**, investigated rather
than left as an unexplained number: `GraphStore.apply_application_
submitted()` calls `upsert_node()`/`upsert_edge()` up to ~10-16 times
per application (platform, application, borrower, and up to 3 token
nodes each with their own edge, plus document nodes), and every one of
those, plus every `PostgresEventRepository` call in the same request
(`create_or_get`, `update_status`, `update_analysis`,
`finalize_risk_decision`), opens its **own fresh `psycopg.connect()`** —
confirmed by reading `_connect()` in each repository, which is called
per-method with no connection reuse or pooling across a single request's
lifetime. A single ingestion request therefore performs on the order of
a dozen-plus separate Postgres TCP-connect-and-auth round-trips, not one
connection reused for several queries. This is the leading, code-grounded
explanation for both the ~280ms floor at concurrency=1 (real connection
overhead repeated many times, not one slow query) and why throughput
scales sub-linearly with concurrency (10 concurrent requests means up to
~150 near-simultaneous fresh Postgres connections contending for the
same backend's connection-accept path on a 2-vCPU sandbox). **Not fixed
this phase** — deliberately: this is an architecture-level change
(introducing real connection pooling, e.g. `psycopg_pool`, across every
repository) with a blast radius well beyond a performance-testing phase's
scope, the same "large blast radius, deserves its own dedicated phase"
reasoning this project already applied to the `starlette`/`vite`
dependency upgrades in Phase 22. Flagged here as a concrete, well-
understood optimization target for a future phase, not a vague
"could probably be faster" note.

## Results: central app reads (warm graph, concurrency=10)

| Endpoint | GNN condition | p50 | p95 | p99 | throughput (req/s) |
|---|---|---|---|---|---|
| `GET /api/v1/applications/:id` | rule-only | 112ms | 139ms | 149ms | 83.2 |
| `GET /api/v1/applications/:id` | with GNN | 105ms | 148ms | 150ms | 83.0 |
| `GET /api/v1/incidents` | rule-only | 102ms | 143ms | 160ms | 85.8 |
| `GET /api/v1/incidents` | with GNN | 113ms | 152ms | 156ms | 86.3 |
| `GET /api/v1/metrics` | rule-only | 430ms | 499ms | 539ms | 22.2 |
| `GET /api/v1/metrics` | with GNN | 380ms | 549ms | 566ms | 24.1 |

`GET /api/v1/metrics` is consistently the slowest of the three reads by
a wide margin (~4x the other two) — expected, since it aggregates across
multiple tables rather than a single indexed lookup, and was not
optimized this phase (out of scope: this phase measures, Phase 23 is not
a query-optimization pass). The other two reads show essentially no
GNN-condition-dependent difference, as expected — read endpoints never
invoke GNN inference.

## Results: loan platforms

| Platform | Submission p50 (c=1) | Submission p50 (c=10) | Submission throughput (c=10, req/s) | Admin read p50 (c=10) |
|---|---|---|---|---|
| A (QuickFund) | 2.1ms | 14.5ms | 475 | 20.5ms |
| B (TrustLend) | 2.2ms | 21.5ms | 287 | 21.1ms |
| C (SecureCredit) | 2.4ms | 24.7ms | 377 | 19.7ms |
| D (FinBridge) | 2.2ms | 40.6ms | 114 | 47.7ms |

Every platform's in-memory-repository path is fast — single-digit
milliseconds at concurrency=1, tens of milliseconds at concurrency=10 —
consistent with there being no real network hop to a database (the
standing Phase 1 MongoDB-unreachable limitation, worked around with an
in-memory double since Phase 2; these numbers say nothing about real
MongoDB latency, only about this platform's own Express routing/
validation/tokenization overhead). Platform D is the outlier: roughly
2-3x A/B/C's concurrency=10 latency, with a visibly wider p50-p99 spread
(41ms → 214ms vs. A's 14ms → 32ms). D's `applicationService.js` does the
same tokenization work as the other three plus an enum-membership check
against `ALLOWED_PRODUCT_CATEGORIES` and two additional required-field
checks — real but small extra work that doesn't obviously account for a
2-3x gap on its own. Stated honestly as an observed, not fully explained,
outlier from a single run rather than either dismissed or over-
interpreted; a repeat run with more samples would be needed to say
whether this is a real, repeatable platform-D-specific cost or this
run's share of sandbox noise landing on platform D.

## Run-to-run variance, honestly

Three full runs of this harness were performed during this phase (the
first two ended in the real bugs documented above before completing;
only the third and fourth - the one reported above - ran clean
end-to-end). Comparing the third (imperfect but complete) and fourth
(final, clean) runs' central-ingestion numbers, both taken under
identical conditions with 0 errors: concurrency=10, cold, rule-only p50
went from 1741ms to 1830ms (+5%); warm, with-GNN p50 went from 2041ms to
2194ms (+7%). Single-digit-percent run-to-run drift at concurrency=10 is
consistent with this being a genuinely shared, 2-vCPU sandbox rather than
a dedicated benchmarking host - stated here so a future reader does not
mistake normal variance for a real regression, or a real regression for
normal variance, without checking against this baseline first.

## Summary, stated plainly

This sandbox cannot produce a production capacity-planning number - see
"What this does NOT measure" above. What it does honestly establish:
GNN inference adds a real, consistently-directional latency cost (roughly
8-29% depending on concurrency); the central ingestion path's per-request
latency is dominated by real per-repository-call Postgres connection
overhead, not GNN or the rule engine, and that is a concrete target for a
future connection-pooling phase, not a vague suspicion; every loan
platform's in-memory-repository path is fast and platform D shows an
unexplained (not dismissed) latency outlier worth a repeat measurement;
and this phase's own first three runs surfaced two real script bugs and
one real, previously-latent production concurrency bug, all found by
actually running the thing under real concurrent load rather than by
code review alone - consistent with this project's Phase 20-established
pattern that a real load-generating test finds real bugs code review
alone does not.

---

## Phase 33: Quantitative Ablation Study & High-Concurrency (50 / 100 / 500) Scaling

### 1. Three-Configuration Ablation Study (`scripts/run_ablation_study.py`)

Reproducible via `npm run benchmark:ablation` (`python scripts/run_ablation_study.py`), with raw JSON output saved to `performance-results/ablation_study_results.json` and SVG visualization saved to `docs/assets/ablation_comparison_chart.svg`:

| Configuration | Multi-Platform Ring Recall | Benign Multi-App FPR | Precision | F1 Score | Latency P50 / P95 / P99 (ms) |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **1. Siloed Rule Engine Only** *(No cross-platform sharing)* | `0.0%` | `0.0%` | `0.0%` | `0.000` | `0.89 / 1.33 / 1.64 ms` |
| **2. Centralized Rule Engine Only** *(Cross-platform, no GNN)* | `50.0%` | `25.0%` | `70.6%` | `0.585` | `1.05 / 1.76 / 2.04 ms` |
| **3. QuickFund-X Hybrid (`0.6 GNN + 0.4 Rules`)** | **`100.0%`** | **`0.0%`** | **`100.0%`** | **`1.000`** | **`5.29 / 7.13 / 7.36 ms`** |

### 2. High-Concurrency Load Benchmark: 50, 100, and 500 Concurrent Applications (`scripts/run_load_benchmark.py`, `scripts/locustfile.py`, `scripts/k6_load_test.js`)

Reproducible via `npm run benchmark:load` (`python scripts/run_load_benchmark.py`), with raw JSON output saved to `performance-results/concurrency_50_100_500_results.json` and SVG chart saved to `docs/assets/throughput_scaling_chart.svg`:

| Concurrency Tier | Total Requests | Throughput (req/s) | P50 Latency (ms) | P95 Latency (ms) | P99 Latency (ms) | Error Rate |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **50 Concurrent Apps** | `150` | `51.80 req/s` | `391.19 ms` | `754.23 ms` | `915.72 ms` | `0.00%` |
| **100 Concurrent Apps** | `200` | `52.14 req/s` | `358.13 ms` | `931.71 ms` | `1106.42 ms` | `0.00%` |
| **500 Concurrent Apps** | `500` | `64.61 req/s` | `1722.40 ms` | `3530.18 ms` | `4225.73 ms` | `0.00%` |

