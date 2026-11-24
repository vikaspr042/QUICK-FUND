# ML_MODEL

Status: **training, inference, the hybrid rule+GNN decision, explanation
generation, and incident persistence are all real** (Phase 13, Phase 14,
Phase 15, Phase 16, and Phase 17, below) — items 1-3 and 5-7 are no
longer a plan, they are what was actually built and actually measured.
Item 4 (training data composition) was completed in Phase 12. Inference
(loading the checkpoint into the live request path) was completed in
Phase 14. The deterministic rule engine, the GNN+rule combination
function, and the real `CLEARED`/`FRAUD_DETECTED` decision were
completed in Phase 15 — see that section below, including two findings
that came out lower/stranger than hoped and are reported here exactly as
measured, not smoothed over. Phase 16 turned those same decisions into
human-readable and technical explanations, directly referencing both
Phase 15 findings rather than presenting every decision with equal,
unqualified confidence. Phase 17 persists a `FRAUD_DETECTED` decision as
a real incident record - no new model output, but a deliberate choice
about how finding 2 (the GNN-only false-positive class) is handled at
that persistence step, recorded in that section below. Nothing in this
file is a result until it has actually been produced by running code in
this environment.

## Phase 12: real training data composition

Produced by `central-fraud-intelligence/scripts/generate_scenarios.py`
(seed `12345`, `instances_per_pattern=3` — see
`central-fraud-intelligence/datasets/scenario_generator.py` for the full
generation logic and every design decision, and `seed-data/scenarios/
scenarios.json` for the committed dataset itself), then verified for real
by applying every generated event to a real `GraphStore` against a real
Postgres test database, and separately by
`central-fraud-intelligence/tests/test_scenario_generator.py` (14 tests,
part of the normal automated suite) — these are the actual measured
counts, not a planning estimate:

| | Count |
|---|---|
| Total scenarios | 51 |
| Total applications (synthetic loan applications) | 129 |
| Labeled SUSPICIOUS | 39 |
| Labeled GENUINE | 12 |
| Of which hard negatives (GENUINE, structurally resembles a fraud pattern) | 9 |
| Distinct patterns generated | 17 of `DEMO_SCENARIOS.md`'s 20 |
| Instances per pattern | 3 (evenly split — every one of the 17 patterns has exactly 3 instances) |

3 of the 20 catalog patterns (#6 same device, #7 same bank token, #16
legitimate family shared device) are **deliberately not generated** — all
three need a DeviceToken/BankToken graph node, and Phase 10 already
established that neither exists because no loan platform collects that
data yet. Generating "device-sharing" examples anyway would mean
inventing token values the graph has nowhere to put — exactly the kind of
fake coverage this project's checkpoint discipline exists to prevent.
Revisit if a platform ever actually collects device/bank data.

These counts (and the dataset itself) regenerate deterministically from
the same seed — `test_scenario_generator.py::test_generation_is_deterministic`
checks this directly. Rerunning `generate_scenarios.py` with different
`--seed`/`--instances-per-pattern` values produces a different, but
equally real, dataset; whatever values actually produce the committed
`scenarios.json` are recorded in that file's own `generatedWith` field, not
just here.

## Phase 13: GNN training — real, and here is what was actually built and measured

### 1. Graph schema recap

As actually implemented by Phase 10/11 (`central-fraud-intelligence/src/graph/graph_store.py`)
and consumed directly by `src/gnn/features.py`/`src/gnn/model.py` (Phase 13)
via `GraphStore.export_all()`:

- **8 node types**: `platform`, `application`, `borrower`, `name_token`,
  `email_token`, `phone_token`, `address_token`, `document_hash`.
- **7 edge types**: `APPLICATION_SUBMITTED_TO_PLATFORM`,
  `BORROWER_SUBMITTED_APPLICATION`, `BORROWER_HAS_NAME`,
  `BORROWER_HAS_EMAIL`, `BORROWER_HAS_PHONE`, `BORROWER_HAS_ADDRESS`,
  `APPLICATION_HAS_DOCUMENT`.

No new node/edge type was added in Phase 13 — the GNN trains directly on
what Phase 10 already persists, which is the whole point of building the
graph store before the model.

### 2. Architecture decision — confirmed

The Phase 10-12 working assumption (a lightweight heterogeneous
GraphSAGE-style model, plain PyTorch, no PyG/DGL) is **confirmed, not
revised**, now that the real graph schema (above) and real dataset (Phase
12's 51 scenarios / 129 applications / ~590 total nodes) both actually
exist. See `central-fraud-intelligence/src/gnn/model.py`'s module
docstring for the full reasoning; summarized:

- **Scale**: a few hundred nodes, not millions — a full graph library's
  mini-batch samplers and GPU kernels solve a problem this project
  doesn't have.
- **Interpretability**: Phase 16 (explainability) needs to attribute a
  prediction to specific edges — a ~150-line hand-rolled implementation
  has no framework internals to reverse-engineer.
- **Environment**: this build sandbox already fought hard enough just to
  install plain `torch` (see `TROUBLESHOOTING.md`'s Phase 13 entry) —
  adding `torch-geometric`'s own compiled-extension wheels would multiply
  that risk for no benefit at this scale.

Concretely: `HeteroSAGELayer` gives every relation its own learned linear
transform and mean-aggregates messages into each target node; every one
of the 7 real edge types gets an explicit reverse counterpart
(`<TYPE>_rev`) so message passing is bidirectional — mirroring Phase 11's
direction-agnostic BFS (`get_relevant_subgraph`) for the same reason:
fraud signals need to flow both ways across a directed graph.
`num_layers=4` is not a tuned hyperparameter — it is **deliberately set
equal to** `graph_store.DEFAULT_SUBGRAPH_HOPS` (also 4, see Phase 11),
so the GNN's receptive field matches exactly the neighborhood Phase 11
already established as sufficient to reach a different application
through one shared token. `tests/test_gnn.py::test_num_layers_matches_default_subgraph_hops`
pins this cross-phase consistency so it can't silently drift.

### 3. Training vs. inference split

Training is fully offline: `central-fraud-intelligence/training/` (not
under `src/`, not imported by the running API — confirmed by grep, and by
the fact this phase touched zero HTTP-facing code, so the
`/health`-never-depends-on-Postgres boot-order guarantee from Phase 1/10
is untouched by construction, not just by luck) builds a dataset,
trains, evaluates, and writes a checkpoint under `models/`. As of Phase
14, a checkpoint really does get loaded once at central-app startup and
run on `get_relevant_subgraph()`'s bounded output per incoming event —
see the Phase 14 section below; the whole graph is never retrained or
reprocessed per event, per `ARCHITECTURE.md`.

### 4. ~~Training data composition~~ — done in Phase 12 (see above).

### 5. CLI commands (from `central-fraud-intelligence/`)

```
python3 training/train.py                     # train + evaluate + save checkpoint (defaults below)
python3 training/train.py --epochs 300 --lr 0.01 --hidden-dim 32 --num-layers 4
python3 training/baseline.py                   # non-graph baseline (see item 7)
```

`train.py` builds the dataset (truncating and re-populating a Postgres
test database from `seed-data/scenarios/scenarios.json` — see
`training/dataset.py`), trains full-batch for `--epochs` steps
(`BCEWithLogitsLoss` with `pos_weight` computed from the real train-split
class counts, Adam), evaluates on the held-out scenario-stratified eval
split, and writes `models/gnn_checkpoint.pt` (weights, gitignored — see
`.gitignore`'s existing `*.pt`/`*.pth` rule) plus
`models/gnn_checkpoint.meta.json` (architecture, training args, the full
loss curve, and eval metrics — committed, since this is the actual
evidence of what running `train.py` really produced). There is no
separate `evaluate`/`save-model`/`load-model`/`infer` CLI — with a
dataset this small, train+evaluate+save is one command, and `infer` is
explicitly Phase 14's job (inference doesn't exist as a concept yet:
there is no live consumer of the checkpoint).

### 6. Real evaluation results

One real run, seed `12345`, 200 epochs, `hidden_dim=32`, `num_layers=4`,
against the committed `seed-data/scenarios/scenarios.json` (51 scenarios
→ 129 applications → 590 total graph nodes after ingestion). Split:
scenario-stratified 2/3 train — 86 applications (72 SUSPICIOUS / 14
GENUINE) train, 43 applications (36 SUSPICIOUS / 7 GENUINE) eval (see
`training/dataset.py`'s `_scenario_split` for the exact deterministic
rule — every pattern with ≥2 instances appears in both splits). Full
`lossCurve` (all 200 values) is in `models/gnn_checkpoint.meta.json`,
not reproduced here; final train loss `3.86e-05`.

| Metric (held-out eval split, 43 applications) | GNN (this phase) | Non-graph baseline (item 7) |
|---|---|---|
| Accuracy | 0.9767 | 0.9070 |
| Precision | 0.9730 | 0.9000 |
| Recall | 1.0000 | 1.0000 |
| F1 | 0.9863 | 0.9474 |
| Confusion matrix (rows=true, cols=pred, order GENUINE/SUSPICIOUS) | `[[6,1],[0,36]]` | `[[3,4],[0,36]]` |

**Honest reading of this result, not just the numbers:**

- Recall is 1.0 for both models on this eval set — every truly SUSPICIOUS
  application was caught. That is a real, measured result, not a
  fabricated one, but it is also the *easier* class here (SUSPICIOUS
  outnumbers GENUINE roughly 5:1 in this dataset — see Phase 12's class
  counts), so a recall of 1.0 alone should not be over-read as "the model
  is excellent" without also looking at the smaller class.
- The GNN's entire advantage over the baseline on this run is on the
  GENUINE side: 6/7 correct vs. the baseline's 3/7 — i.e. the baseline
  (which sees each Application node's own 16-dim feature vector but has
  no access to the graph's edges at all) false-flagged 4 of 7 genuine
  applicants, while the graph-aware model false-flagged only 1. This is
  exactly the comparison item 7 exists to make, and it points the same
  direction the project's design predicted: cross-application graph
  structure (shared tokens, multi-platform velocity) is where the signal
  the flat baseline can't see actually lives.
- The one GNN false positive, inspected directly (not just counted): it
  was `scenario_0047_normal_genuine_baseline` (application
  `SYN-C-00124`), predicted SUSPICIOUS with probability 1.0. It was
  **not** one of the dataset's deliberately-constructed `hard_negative`
  scenarios (see Phase 12) — those were all classified correctly. A
  confidently-wrong call on the "easy", non-adversarial baseline case is
  a genuine weak spot worth flagging, not explaining away.
- Final train loss essentially reaching zero (`3.86e-05`) after 200
  epochs on only 86 training applications is a real sign of overfitting
  to this specific 51-scenario dataset, not evidence the model has
  learned a generally robust notion of "fraud". With only 3 instances per
  pattern (Phase 12), the model has very little variation within a
  pattern to generalize across. **The plan for improving this** (per this
  section's own mandate below): increase `--instances-per-pattern` when
  regenerating `scenarios.json` (more within-pattern variation), add
  L2 weight decay or increase `--dropout` (currently 0.1) to fight the
  near-zero training loss directly, and — the structurally important one
  — bring in the 3 deferred patterns (#6/#7/#16) once a platform actually
  collects device/bank data (Phase 10/12's open item), since a synthetic
  catalog capped at 17 patterns cannot represent fraud strategies outside
  that catalog no matter how well it's fit.

### 7. Comparison against a non-graph baseline

`central-fraud-intelligence/training/baseline.py` trains a plain
scikit-learn `LogisticRegression(class_weight="balanced")` directly on
the exact same per-node 16-dim feature vectors (`src/gnn/features.py`)
and the exact same scenario-stratified train/eval split
(`training/dataset.py`) as the GNN — the ONLY difference is that this
baseline never sees a single edge. This isolates specifically what the
GNN's message passing contributes, rather than comparing two systems that
differ in several ways at once (different features, different data,
different split). Results are in the table above; full output in
`models/baseline_metrics.json`. A rule-based-matching-only comparison
(the other half of this item's original scope) is deferred to Phase 15,
where the actual rule engine this project will compare against gets
built — comparing against a rule engine that doesn't exist yet would mean
fabricating one just for this table, which is exactly what this
project's checkpoint discipline exists to avoid.

## Phase 14: inference — real, wired into the live request path

`src/gnn/inference.py`'s `GNNScorer` class loads the Phase 13 checkpoint
once, at FastAPI startup (`src/main.py`'s lifespan), by reconstructing the
exact `HeteroGraphSAGE` architecture from `models/gnn_checkpoint.meta.json`
(feature dim, hidden dim, num layers, relation list, dropout - not
hand-copied constants that could drift from what was actually trained)
and loading the saved state dict into it, then setting the model to
`eval()` mode. Loading is entirely best-effort - any failure (missing
file, malformed JSON, a meta.json missing a required key, a checkpoint
that isn't a valid torch file, an architecture/state-dict mismatch) is
caught and logged, and `load()` returns `None` rather than raising, so a
missing or broken checkpoint degrades central-app startup, never crashes
it (see `tests/test_inference.py`'s real boot-order test for direct
proof, not just an assertion about the code).

On each incoming event, once the graph-update step has completed
(`status: GRAPH_UPDATED`), `src/api/applications.py`'s POST handler calls
`GraphStore.get_relevant_subgraph("application", applicationId)` (Phase
11's bounded k-hop extraction, `DEFAULT_SUBGRAPH_HOPS` matching the
model's `num_layers` per the cross-phase pin `test_gnn.py` already
checks) and passes the result to `GNNScorer.score_subgraph()`. That
method rebuilds tensors from the subgraph using
`src/gnn/graph_tensors.py`'s `build_graph_tensors()` - the *same* function
`training/dataset.py` calls to build tensors for training, extracted
specifically in this phase so live inference and offline training can
never silently diverge into two different tensor-construction
implementations - runs one forward pass with `torch.no_grad()`, and
returns `sigmoid(logit)` for the root application node: a real
probability in `[0, 1]`.

The resulting score is persisted (new `fraud_score` column on
`ingested_events`, via `repository.update_analysis()`) and the event's
`status` advances to `ANALYZED`. Both `POST /api/v1/applications/process`
and `GET /api/v1/applications/:id` return it as `fraudScore`. Like every
other step in this pipeline, the whole inference call is wrapped in a
broad `try/except` in the request handler: a scoring failure (or no
checkpoint having been loaded at all) leaves the event at `GRAPH_UPDATED`
with `fraudScore: null` and the request still returns 200, retried by the
next redelivery exactly like the graph-update step is. See
`ARCHITECTURE.md` section 7 and `DATA_FLOW.md`'s Phase 14 update for the
full pipeline framing, and `tests/test_inference.py` for the automated
coverage - including a test that runs the real committed checkpoint
against a real extracted subgraph and gets a real score back, not a
mocked one.

This phase does **not** interpret or threshold the score in any way - no
deterministic rule engine, no hybrid combination with rule-based signals,
no `CLEARED`/`FRAUD_DETECTED` decision, no incident creation. `fraudScore`
is a raw model output, reported as-is; Phase 15 is where it starts to
mean something actionable.

## Phase 15: rule engine, hybrid scoring, and the real CLEARED/FRAUD_DETECTED decision

### What was built

`central-fraud-intelligence/src/fraud_detection/rules.py`'s
`evaluate_rules(subgraph)` is a deterministic rule engine that runs on
the *same* bounded subgraph (`GraphStore.get_relevant_subgraph()`,
Phase 11) Phase 14's GNN already fetches for a given application - it is
reused, not re-fetched, in the common case (see `src/api/
applications.py`). It walks the root application's borrower node's
`BORROWER_HAS_PHONE`/`EMAIL`/`NAME`/`ADDRESS` edges to find shared
tokens, then walks each token back out to *other* borrowers (explicitly
excluding the root borrower itself, so the same person re-applying is
never mistaken for a second identity - see pattern 17 below), and
separately checks `APPLICATION_HAS_DOCUMENT` for a byte-identical file
reused by a different application. Every signal found is weighted
(phone/email 0.30, name 0.20, address 0.12, shared document 0.40 - see
`rules.py`'s own comments for why address is lowest and document is
highest) and scaled by a time-proximity multiplier: full weight within
24 hours, decaying linearly to a 0.05 floor by 30 days. This is a
deliberate design decision, not an oversight: "application velocity" is
not a separate rule, it is folded into every shared-token check, because
`scenario_generator.py` builds several SUSPICIOUS patterns (1, 9) and
GENUINE hard negatives (15, 19) as the exact same graph topology,
differing only in timing - a rule that ignored timing could not tell
them apart. A hub bonus (up to +0.25) is added when three or more
distinct other identities share one token (pattern 12). The final
`rule_score` is clamped to `[0, 1]`.

`src/fraud_detection/scoring.py`'s `combine_signals(gnn_score,
rule_result)` merges Phase 14's GNN probability with this phase's
`rule_score` as a weighted average: `0.6 * gnn_score + 0.4 * rule_score`.
GNN got the larger weight because Phase 13's offline evaluation measured
it at 97.67% accuracy on held-out data, materially ahead of a purely
graph-shape-blind baseline - it earns the larger vote. When no GNN
checkpoint is loaded (`gnn_score is None`), `final_risk_score` is the
raw `rule_score`, unscaled by 0.4 - deliberately, so a missing model
never structurally caps the maximum achievable risk score at 0.4 and
makes `FRAUD_DETECTED` unreachable from rules alone. `FRAUD_DETECTED` is
returned when `final_risk_score >= 0.5`, else `CLEARED` - both are now
real, persisted status values for the first time since the status
progression was documented back in Phase 8.

Both new modules are wired into `POST /api/v1/applications/process`
immediately after the existing GNN-inference step, gated the same
status-based, retry-safe way every prior pipeline step is (`if status ==
"ANALYZED":`, never a negative check - see Phase 14's checkpoint notes
for why that distinction matters on redelivery). A rule-engine or
combination failure is caught, logged, and leaves the event parked at
`ANALYZED` with the new fields null, retried automatically on the next
redelivery - never a 500, never a partially-applied decision. `POST` and
`GET /api/v1/applications/:id` both now return `ruleScore`,
`triggeredRules` (the specific signals that fired, with per-signal
detail), and `finalRiskScore` alongside the existing `fraudScore`.

Automated coverage: `tests/test_fraud_detection.py` (15 tests) covers
the rule engine and `combine_signals()` directly against real graph
fixtures built the same way `tests/test_graph.py`/`test_subgraph.py`
build theirs - including the hard-negative cases (`pattern 15`, `pattern
19`, `pattern 17`'s same-borrower-repeat) scoring near zero, and the
hub-bonus and document-reuse cases scoring high. `tests/test_inference.py`
was updated for the fact that a successfully-scored event now
progresses automatically past `ANALYZED` to a real terminal status
within the same request (previously it stopped at `ANALYZED`, since
nothing consumed `fraudScore` yet) and gained a new dedicated test
proving the rule engine's own failure-retry-safety guarantee
(`evaluate_rules` monkeypatched to raise: status parks at `ANALYZED`,
then a later redelivery with the failure removed reaches
`FRAUD_DETECTED` correctly). Full suite: **117/117 passing** as of this
phase (was 101/101 at the Phase 14 checkpoint).

### Real evaluation against the committed dataset - two modes, both honestly reported

`central-fraud-intelligence/scripts/evaluate_fraud_detection.py` replays
all 129 real applications from the committed `seed-data/scenarios/
scenarios.json` dataset through the *exact* production pipeline
functions (graph update → GNN inference → rule engine → combination -
not a reimplementation) using the real trained Phase 13 checkpoint, and
compares the resulting `CLEARED`/`FRAUD_DETECTED` decision against each
scenario's `expected` label. It runs in two modes, both real, both run
in this environment, full output in `models/hybrid_scoring_eval.json`:

- **`interleaved`** (the realistic mode): all 129 applications share one
  graph and are replayed in true global chronological order, exactly as
  a production deployment would receive events from all four platforms
  interleaved.
- **`isolated`** (a diagnostic mode, added specifically to test a
  hypothesis - see below): each of the 51 scenarios is replayed alone,
  against a freshly truncated graph, with zero other scenario's traffic
  present, then the graph is truncated again before the next scenario.

| Metric | interleaved | isolated |
|---|---|---|
| Accuracy | 73.64% | 71.32% |
| Precision | 95.12% | 94.94% |
| Recall | 72.22% | 69.44% |
| F1 | 0.821 | 0.802 |
| Hard-negative accuracy | 83.33% (15/18) | 83.33% (15/18) |
| False positives | 4 | 4 |
| False negatives | 30 | 33 |

This is materially lower accuracy than Phase 13's standalone offline GNN
evaluation (97.67%), and that gap is reported honestly rather than
hidden or explained away with an untested guess. Two real findings,
both investigated with actual code and actual data in this environment,
explain it:

**Finding 1 - the false negatives are a structural property of
real-time single-pass scoring, not a dataset-interleaving artifact.**
The `isolated` mode was built specifically to test a hypothesis written
into `rules.py`'s own module docstring *before* any evaluation had run:
that `scenario_generator.py`'s scenarios all sharing one common base
epoch (`2026-01-01T00:00:00Z`) causes unrelated scenarios' events to
land at near-identical global timestamps, so a fraud ring's early
member could be scored before its co-conspirators' events - from
*other* scenarios that merely happen to interleave in time - have
arrived. If that were the dominant cause, `isolated` mode (zero other
scenarios' traffic, ever) should have scored close to Phase 13's 97.67%.
It did not - it scored *slightly worse* than `interleaved` (71.32% vs.
73.64%). Cross-referencing every false-negative application in both
modes against its own scenario's internal submission order found the
real explanation: **100% of false negatives in both modes (30/30
interleaved, 33/33 isolated) are exactly the first-submitted application
within their own scenario** - never a second, third, fourth, or fifth
member. This is not cross-scenario collision; it is the unavoidable
structural property that whichever ring member applies first has zero
corroborating siblings in the graph at the moment *it* is scored, full
stop, even with every other scenario in the dataset held to zero
traffic. `rules.py`'s module docstring has been corrected to state this
verified explanation instead of the original, now-disproven, hypothesis.
This is an honest limitation of single-pass real-time scoring generally
(a redelivery or a later re-scan of that same first application, once
its siblings have arrived, would score it correctly - this project just
doesn't have a re-scan trigger yet, which is a reasonable future
candidate - still not built as of Phase 17 either, not built here since
this checkpoint's scope was the initial hybrid decision, not a
re-scoring scheduler).

**Finding 2 - a second, distinct, genuine GNN limitation: the trained
model flags some legitimate same-borrower repeat applications at very
high probability.** Three `legitimate_repeated_application` (pattern 17
- same borrower, same platform, two applications weeks/months apart, no
other identity involved at all) scenarios produced `fraudScore ≈
0.9999` from the GNN in both evaluation modes (`ruleScore` correctly
stayed `0.0` in every one - the rule engine's explicit same-borrower
exclusion, tested directly in `test_fraud_detection.py`, is working as
designed and is not the source of this false positive), pushing
`finalRiskScore` to `0.6 * 0.9999 ≈ 0.6`, past the 0.5 threshold, to a
false `FRAUD_DETECTED`. This reproduces identically in `isolated` mode
(same three applications, same near-1.0 score, confirming it is not
cross-scenario noise either) - it is a real, repeatable property of the
trained checkpoint itself when scoring a returning applicant's second
submission, and it is the same general failure mode already documented
honestly in Phase 13 as this model's one known offline false positive
(`SYN-C-00124`, `normal_genuine_baseline`, also scored 1.0 there). This
project is not attempting to explain the GNN's internal reasoning
further in this phase (that is squarely Phase 16's "explainability"
scope, not this one's) - it is reported here as a real, measured,
unresolved limitation of the current checkpoint: legitimate repeat
applicants are one real class of false positive this system can
currently produce, and a human reviewer or a future retraining pass
(more `legitimate_repeated_application` instances in the training mix)
is the honest path to reducing it, not a claim that it's already fixed.

Both findings, and the full 34/37-item misclassification detail behind
them, are in `models/hybrid_scoring_eval.json` - nothing in this section
summarizes a number that isn't traceable back to that file.

## Phase 16: explainability

Nothing new to measure here - `src/explainability/explain.py`'s
`generate_explanation()` is a pure, deterministic function of the score
fields Phase 14/15 already compute and this file already reports (there
is no separate "explanation accuracy" to evaluate; it presents an
already-measured decision, it doesn't make a new one). See
`ARCHITECTURE.md` section 8's explainability subsection for the design
and `TESTING.md`'s "Phase 16: explainability tests" section for the
automated coverage. Two things worth recording here specifically because
they reference the real findings reported earlier in this file:

- Every `FRAUD_DETECTED` decision with an empty `triggeredRules` (i.e.
  driven entirely by the GNN, with zero corroborating rule signal) now
  carries an explicit caveat naming this Phase 15 section's finding 2
  (the legitimate-repeat-applicant false-positive class) and
  recommending manual review - a concrete, evidence-grounded use of a
  real finding from this file, not a generic disclaimer.
- Every `CLEARED` decision carries a caveat naming finding 1 (the
  first-arrival blind spot) - stating plainly that the decision reflects
  only information available at processing time and is not automatically
  revisited if related applications arrive later, since this project
  still has no re-scan/re-scoring trigger as of this phase either.

## Phase 17: fraud incidents

Also nothing new to measure - incident creation (`src/incidents/`)
persists a snapshot of a decision this file already reports, it does not
change what the GNN or rule engine outputs or how `combine_signals()`
decides `CLEARED`/`FRAUD_DETECTED`. See `ARCHITECTURE.md` section 8's
"Fraud incidents" subsection for the design. One consequence worth
recording here specifically, since it's a direct, deliberate response to
this file's own finding 2: a `FRAUD_DETECTED` decision driven entirely
by the GNN (empty `triggeredRules` - finding 2's exact shape) still
creates a full, real incident rather than a suppressed or downgraded
one. That was a genuine design choice, not the only reasonable one -
silently declining to persist an incident for a known-unreliable
decision class was considered and rejected, because it would hide the
limitation rather than surface it (the same reasoning Phase 16 already
applied to the explanation text itself). The incident instead carries
the same `primaryDriver: "gnn_only"` classification and manual-review
caveat Phase 16 generates, so this known limitation stays visible to
whoever reviews the incident, rather than disappearing from the record
because the model that flagged it isn't fully trusted here.

## Phase 20: a real, unresolved finding — live single-request inference collapses to near-zero, unlike offline replay

Phase 20's job was multi-process integration testing, not model
evaluation, but running the real checkpoint through the real live
`POST /api/v1/applications/process` path for the first time (every test
before Phase 20 either forced `gnn_scorer` to `None` or substituted a
fixed-value `_StubScorer` — see `tests/conftest.py`/`tests/
test_incidents.py`) surfaced something worth recording here rather than
letting `TROUBLESHOOTING.md` be the only place it's written down.

**What was observed:** for a genuinely new, live-submitted application
(via the real HTTP endpoint, real `GraphStore.apply_event()`, real
`GraphStore.get_relevant_subgraph()`, real `GNNScorer.score_subgraph()`)
that clearly shares a phone number and email address with another
application already in the graph, `score_subgraph()` returned exactly
`0.0` — the underlying raw logit was approximately **-121 to -125**, an
extremely confident prediction in the "not suspicious" direction, for an
application the deterministic rule engine correctly scored `rule_score
= 0.6` (the maximum two-signal combination: shared phone `0.30` + shared
email `0.30`). This was checked at two very different local-subgraph
sizes for the same application (a 21-node cold-start subgraph, and a
211-node subgraph after seeding the same live database with a full
replay of the real, already-committed 129-application Phase 12 scenario
dataset) — both produced essentially the same deeply negative logit, so
overall graph size is not the explanation.

**What this is not:** it is not the already-documented "first-arrival
blind spot" (finding 1 below/`rules.py`'s own docstring) — that's about a
ring's *first* member having no corroborating siblings yet. This
application had a real, already-graphed corroborating sibling (the
matching phone+email edges existed in Postgres before this application
was scored), and the collapse happened anyway.

**What it is not proven to be, either:** a bug in `src/gnn/
graph_tensors.py`'s feature construction was considered (e.g. some
degree-based feature computed differently live vs. during the
`evaluate_fraud_detection.py`/training replay) but not confirmed - degree
features there are raw local counts, not normalized by total graph size,
which doesn't obviously explain a fixed graph-size-independent collapse.
A distribution-shift explanation (the specific *shape* of a genuinely
live, single-request subgraph - built by exactly one `apply_event()` call
immediately followed by exactly one `get_relevant_subgraph()` call -
differing systematically from what an incrementally-replayed batch script
constructs even for logically-equivalent graph content) is the leading
hypothesis but is not verified. Given the severe overfitting already
noted in the Phase 13 section above (final training loss `3.86e-5`), a
model this tightly fit to its exact 86-application training set
generalizing poorly to novel input shapes it never saw during training is
plausible on its face, but this file will not claim a root cause it
hasn't actually verified - that would violate the same "NO FAKE SUCCESS"
discipline this document has followed since Phase 13.

**Why this wasn't fixed in Phase 20:** Phase 20's scope is multi-process
integration testing, not model debugging - retraining, re-engineering
features, or attempting a live/offline-parity fix all carry real risk of
silently changing this project's only already-evaluated, already-reported
model (the 97.67%/73.6% figures elsewhere in this file). This is recorded
here, honestly and in detail, as a high-priority open item for whichever
future phase next touches GNN training or inference - see
`TROUBLESHOOTING.md`'s matching Phase 20 entries for the reproduction
steps and for how Phase 20 worked around it (the real rule-only pipeline
path, fixed the same session - see the entry above this one in
`TROUBLESHOOTING.md`) to still verify the rest of the pipeline (incident
creation, notification fan-out, cross-platform correctness) for real.

**Update - Phase 27: root-caused and fixed.** The leading distribution-shift
hypothesis above was correct, and specific: `src/gnn/features.py`'s
timestamp feature encoded `submittedAt` as an offset from a fixed absolute
calendar epoch. Every training example's offset fell inside a bounded
range by construction (the Phase 12 synthetic dataset's timestamps all
sit within a few synthetic months of that epoch); a genuinely live
request scored at real wall-clock "now" produces an offset far outside
that range - unbounded out-of-distribution extrapolation, not a subgraph-
shape or graph-size effect. Fixed by making the timestamp feature relative
to a reference point computed from the graph being scored (the earliest
dated `Application` node among the ones actually present), with a
defensive clamp to the empirically-measured training range, in
`_reference_epoch()`/`_scaled_timestamp()`. This is a feature-encoding
fix, not a retrain: `models/gnn_checkpoint.pt` is byte-identical to the
Phase 13 checkpoint referenced throughout this file. Verified: the exact
Phase 20 repro now scores `1.000` instead of `0.000`; the existing
checkpoint's offline evaluation accuracy is unchanged at 73.64% (the
relative-reference computation is a no-op for `evaluate_fraud_detection.py`'s
own interleaved-replay method, which already scores whole graphs at once);
`test_gnn.py` (21/21) and the full suite (265/265) pass. A first attempt -
clamping the old absolute-epoch feature to its measured training maximum
without making it relative - was tried and genuinely failed (live score
stayed `0.000` even when clamped), because the model had specifically learned to
read large absolute offsets as "a genuine reapplicant returning months
later," not as generically out-of-range; that negative result and the
full technical writeup are in `TROUBLESHOOTING.md`'s Phase 27 section, and
`KNOWN_ISSUES.md` item A.1 is updated accordingly. This closes the one
open item this section flagged as high priority.

## Explicit non-claims

This file will not claim the model detects "all possible fraud" — only the
fraud patterns defined and generated by the Phase 12 scenario generator,
whose coverage is documented in `DEMO_SCENARIOS.md`. Metrics are reported
per the exact test suite that produced them and are not generalized beyond
that suite.

Phase 13 adds one more explicit non-claim: the 97.67% eval accuracy
reported above is **not** claimed to be representative of real-world
performance. It was measured on 43 synthetic applications drawn from the
same 17-pattern catalog, generated by the same code, with the same random
seed's structural conventions, as the 86 applications the model trained
on — a favorable, low-diversity setting relative to production data. The
near-zero final training loss (Phase 13 section, item 6) is itself
evidence of this: the model fits this specific small dataset very well,
which is a different claim from "generalizes to fraud patterns/data it
has not seen a close relative of." Phase 21 (false-positive testing) and
real production data (once available) are what would actually validate
generalization — nothing in this file substitutes for that.

## Phase 21: false-positive testing at statistically meaningful scale

### Why this phase exists

Phase 15's real false-positive measurement (see that section above) was
real — it ran the actual pipeline against actual generated data — but it
rested on a sample too small to trust in either direction: only 12
GENUINE applications total (9 of them hard negatives), of which 4 were
false positives. A rate computed from 12 examples has enormous variance;
Phase 21's whole purpose is to re-measure the same thing on a sample an
order of magnitude larger, using the same already-reviewed generator
rather than inventing a new one.

### What changed

1. **`datasets/scenario_generator.py`**: `generate_catalog()` gained a
   new `include_pattern_21: bool = False` parameter, default off, so
   nothing about the committed `scenarios.json` (what Phase 13's
   checkpoint actually trained on) changes on a plain re-run of
   `scripts/generate_scenarios.py`. This is verified directly by
   `tests/test_scenario_generator.py::test_pattern_21_is_excluded_by_default_and_included_when_requested`.

2. **A new pattern 21, `legitimate_repeated_application_three_times`**,
   added specifically to adversarially stress-test a weakness Phase 15
   itself had already measured but couldn't fully characterize: pattern
   17 (`legitimate_repeated_application` — the same borrower re-applying
   to the same platform a month later) scored ~0.9999 fraud probability
   in Phase 15's evaluation, a genuine false positive. Pattern 21 asks
   the obvious follow-up: does a third application from the same
   returning borrower make that false positive worse, the same, or
   better? It is the same borrower, same platform, three applications a
   month apart each (not two) — a stronger, not weaker, instance of
   ordinary legitimate repeat-customer behavior, still unambiguously
   GENUINE, still `hard_negative=True` since it structurally resembles
   the derived `BORROWER_HAS_MULTIPLE_APPLICATIONS` signal without being
   fraud.

3. **`scripts/generate_fpr_dataset.py`** (new script): generates a
   separate dataset, `seed-data/scenarios/scenarios_fpr.json`, using the
   same seed (12345) but `instances_per_pattern=15` (vs. the committed
   dataset's 3 — a 5x increase) and `include_pattern_21=True`. This is
   270 scenarios, 690 applications total. Verified for real against a
   real `GraphStore` on the test Postgres database: `Applied 690/690
   events. Graph now has 3050 nodes, 3990 edges.` — every event was
   genuinely ingestable, not just structurally plausible. This dataset
   is evaluation-only; it was never used for training and does not
   replace `scenarios.json`.

### Evaluation results (real, both run against the actual production pipeline functions)

Run via `scripts/evaluate_fraud_detection.py` against
`scenarios_fpr.json`, writing
`central-fraud-intelligence/models/hybrid_scoring_eval_fpr.json`. Both
modes score all 690 applications (0 unscored) at the production
threshold of 0.5.

**Interleaved mode** (one shared graph, global chronological order — the
realistic mode):
- accuracy 68.12%, precision 84.78%, recall 72.22%, F1 0.78
- confusion matrix: TP 390, FP 70, TN 80, FN 150
- hard-negative accuracy: 55.56% (75/135 correct, 60 wrong)

**Isolated mode** (graph truncated before each scenario — diagnostic,
removes cross-scenario graph effects):
- accuracy 65.94%, precision 84.27%, recall 69.44%, F1 0.7614
- confusion matrix: TP 375, FP 70, TN 80, FN 165
- hard-negative accuracy: 55.56% (75/135 correct, 60 wrong — identical
  count to interleaved, though not necessarily the identical 60
  applications)

The two modes mostly agree, with one real, notable exception:
`mixed_fraud_genuine_neighborhood` scores 15/15 (100%) in interleaved
mode but 0/15 (0%) in isolated mode. This is consistent with what the
pattern is designed to test (a scenario that deliberately bridges a real
suspicious scenario and a real genuine one via shared tokens, per
`tests/test_scenario_generator.py`'s
`test_mixed_fraud_genuine_neighborhood_bridges_a_suspicious_and_a_genuine_scenario`)
— in isolated mode, the bridging scenario's graph is truncated before
the neighboring scenarios it depends on are applied, so the signal the
pattern is built around may simply not be present yet when it's scored.
This is flagged as an observation, not a diagnosed root cause — it was
not investigated further this phase.

### Per-pattern breakdown (identical between the two modes except where noted above)

| Pattern | Correct/Total | Accuracy |
|---|---|---|
| legitimate_family_shared_address | 30/30 | 100% |
| legitimate_cross_platform_applicant | 30/30 | 100% |
| same_address_with_additional_signal | 30/30 | 100% |
| shared_document_fingerprint | 30/30 | 100% |
| mixed_fraud_genuine_neighborhood | 15/15 interleaved, 0/15 isolated | 100% / 0% |
| multiple_identities_one_controller | 60/75 | 80% |
| collusive_graph_pattern | 45/60 | 75% |
| rapid_multi_platform_applications | 45/60 | 75% |
| same_borrower_4_platforms | 45/60 | 75% |
| suspicious_application_cluster | 30/45 | 66.7% |
| same_borrower_3_platforms | 30/45 | 66.7% |
| conflicting_identity_information | 15/30 | 50% |
| legitimate_repeated_application (pattern 17) | 15/30 | 50% |
| same_borrower_2_platforms | 15/30 | 50% |
| same_email_different_identities | 15/30 | 50% |
| same_phone_different_identities | 15/30 | 50% |
| **normal_genuine_baseline (pattern 18)** | **5/15** | **33.3%** |
| **legitimate_repeated_application_three_times (pattern 21, new)** | **0/45** | **0%** |

Two results stand out as the real findings of this phase:

**Pattern 21 scores 0/45 — every single instance misclassified.** The
adversarial hypothesis behind adding this pattern was confirmed, and
more strongly than expected: a third legitimate repeat application from
the same borrower doesn't just fail to fix pattern 17's ~50% false-
positive rate, it makes it categorically worse (0% vs. 50%). This is a
real, measured regression as a function of a returning borrower's
history length, on data that is unambiguously legitimate by
construction.

**`normal_genuine_baseline` — the simplest possible GENUINE case (a
single applicant, single application, no shared signals with anything
else at all) — is only 33.3% accurate (5/15 correct).** This is the
most concerning single number in this phase's results: if the model
cannot reliably clear the simplest, least ambiguous genuine case in the
entire catalog, its unreliability is not confined to intentionally
tricky hard negatives.

### The platform-bias finding

`normal_genuine_baseline`'s platform assignment is randomized
(`rng.choice(PLATFORMS)` in `_normal_genuine_baseline` —
confirmed by reading the function, not assumed) — unlike patterns 17 and
21, which hardcode their platform (see root-cause hypothesis below). Of
the 15 baseline instances generated, the platform distribution was A=2,
D=3, B=4, C=6. All 10 misclassifications were on platform B (4/4 wrong)
or platform C (6/6 wrong); all 5 platform A (2/2) and platform D (3/3)
instances were correctly cleared. Example real scores from the
misclassified applications (all `expected=GENUINE`, all scored
`FRAUD_DETECTED`):

| Application | Platform | fraudScore | ruleScore | finalRiskScore |
|---|---|---|---|---|
| SYN-B-00223 | b | 0.9999969 | 0.0 | 0.5999981 |
| SYN-B-00358 | b | 0.9999975 | 0.0 | 0.5999985 |
| SYN-C-00133 | c | 1.0 | 0.0 | 0.6 |
| SYN-C-00178 | c | 1.0 | 0.0 | 0.6 |

In every one of these, `ruleScore` is correctly 0.0 (the rule engine has
no legitimate signal to fire on for an isolated, single application with
no shared tokens) — the false positive is entirely attributable to the
GNN score, saturated at ~1.0, dragging `finalRiskScore` up to ~0.6 via
`combine_signals()`'s `GNN_WEIGHT=0.6`. Because the baseline's platform
assignment is genuinely randomized rather than hardcoded, this is
evidence the model's behavior really does vary by which platform an
application is on — this is not an artifact of the generator always
putting baseline cases on the same platform.

### Root-cause hypothesis (evidenced, not proven)

`datasets/scenario_generator.py` hardcodes pattern 17
(`_legitimate_repeated_application`) to `PLATFORMS[1]` ("b") and pattern
21 (`_legitimate_repeated_application_three_times`) to `PLATFORMS[2]`
("c") — never randomized (confirmed by `grep -n "PLATFORMS\[" datasets/
scenario_generator.py`, lines 510 and 563). Combined with Phase 13's
documented severe overfitting (`finalTrainLoss: 3.86e-5` — see that
section above), a plausible explanation is that the model learned a
spurious platform-based shortcut during training — e.g. "applications on
platform b or c are more likely fraud" — rather than a genuine identity-
or graph-structure-based signal, because in the small training set every
repeat-application example happened to live on one specific platform.
The `normal_genuine_baseline` result above (randomized platform, but
100% of failures land on b/c) is consistent with this hypothesis, but
does not prove it — no ablation (e.g. retraining with those patterns'
platform assignment randomized, or holding platform out as a feature)
was run this phase to confirm it. This is stated as a hypothesis, not a
finding, deliberately: `finalTrainLoss` and this eval's platform
correlation are both real, measured facts; the causal link between them
is inference, not measurement.

### Synthesis with Phase 20

Phase 20's live multi-process integration run found the opposite-
looking symptom: a genuinely novel single live request scored a GNN
probability near zero (see Phase 20's section in this file, if present,
or `TROUBLESHOOTING.md`'s Phase 20 entry). Phase 21's finding here is a
near-1.0 score on simple genuine data, but concentrated on specific
platforms. These are not contradictory findings — both are best
understood as the same underlying cause (severe overfitting to a small,
low-diversity training set, per Phase 13's own documented near-zero
training loss) producing unreliable, unstable output outside the narrow
slice of input space the model was actually trained on, with the
specific failure mode (over-confident SUSPICIOUS vs. under-confident
GENUINE) depending on which part of that space a given input happens to
land in.

### What this phase does not claim

This phase does not fix the false-positive rate, does not retrain the
model, and does not confirm the platform-bias root-cause hypothesis with
an ablation. It measures, honestly, at a real and much more trustworthy
sample size (690 applications vs. Phase 15's 12), that the current
checkpoint's false-positive behavior on the hard-negative and baseline
patterns is worse than Phase 15's small sample suggested, and it
narrows down a specific, evidenced (not proven) hypothesis for why.
Retraining with the platform-hardcoding issue fixed, and/or a larger and
more platform-diverse training set, would be the natural next step to
actually test the hypothesis — out of scope for this phase, which is
measurement, not remediation.

## Phase 31: tightening the rule engine and retraining the GNN — the two-party correlated-evidence cap, the platform-bias fix, and a real before/after measurement

**User report**: "the explanation is not up to the mark - if two people
genuinely apply to different lenders and happen to share one detail,
that shouldn't automatically be treated as fraud. The risk score also
feels like a random number, not something an actually well-trained
model produced." Asked, via a clarifying question, whether to fix the
rule engine, retrain the GNN, or both — chose both.

### Stage 3: `src/fraud_detection/rules.py` — a two-party correlated-evidence cap

Before this phase, `evaluate_rules()` summed every fired token-edge
signal (phone 0.30, email 0.30, name 0.20, address 0.12) independently,
uncapped. Two applications that are genuinely the same real person
applying to two different lenders will typically match on several of
these fields at once (same phone, same name, same address) — under the
old scoring, that stacked to a high rule score even though the four
matches are not four independent pieces of evidence; they are one piece
of evidence (same person) counted four times.

Added `TWO_PARTY_TOKEN_SCORE_CAP = 0.35`: when every fired token signal
in an application's evaluation traces back to the same single other
borrower (a two-party match, not a hub/chain touching several other
borrowers), the signals' combined contribution is capped at 0.35 rather
than summed without limit. The cap does not apply when signals point to
different other borrowers (a real multi-party pattern), and it does not
touch the document-hash signal (0.40, unrelated to token matching) or
the hub bonus (already gated on `other_count > 2`). `explain.py` was
updated to add an explicit caveat whenever the cap fired, naming which
signals were involved and stating plainly that a `FRAUD_DETECTED`
verdict alongside this caveat means the model found independent
suspicion beyond the capped shared details, not that the shared details
alone caused it — directly answering the "explanation is not up to the
mark" complaint. 18 new/updated test assertions across
`tests/test_fraud_detection.py` and `tests/test_explainability.py`;
full suite green (277 passed) after this stage alone.

**Honest finding, surfaced rather than hidden**: re-running
`scripts/evaluate_fraud_detection.py` against the *old* checkpoint
immediately after this change produced byte-identical aggregate and
per-pattern results to the pre-change baseline. Diagnosing why: with
`GNN_WEIGHT = 0.6` and `RULE_WEIGHT = 0.4` in `combine_signals()`, a
GNN score effectively saturated near 0.0 or 1.0 (see Phase 21's
overfitting finding) already decides the outcome by itself regardless
of what the 0.4-weighted rule term does underneath it. The rule-engine
change is real and correctly implemented (verified directly at the
`evaluate_rules()` unit level), but with the checkpoint that existed
going into this phase, it had no reachable effect on the final
`CLEARED`/`FRAUD_DETECTED` decision. This is exactly what motivated
prioritizing Stage 4 rather than treating Stage 3 alone as "done."

### Stage 4: retraining — platform-bias fix, a larger dataset, and L2/dropout regularization

Three concrete changes, all aimed at the root causes Phase 21 already
identified (severe overfitting on 86 training applications; patterns 17
and 21 hardcoding their platform assignment to `PLATFORMS[1]`/
`PLATFORMS[2]`, giving the model a spurious platform-based shortcut):

1. **Platform-bias fix**: `datasets/scenario_generator.py`'s
   `_legitimate_repeated_application()` and
   `_legitimate_repeated_application_three_times()` now draw their
   platform with `rng.choice(PLATFORMS)` instead of a hardcoded index,
   matching the randomization `_normal_genuine_baseline()` already
   used. Confirmed this doesn't break either pattern's own structural
   tests (they only assert "same platform/borrower within the
   pattern", never a specific platform value).
2. **Larger, more varied dataset**: regenerated
   `seed-data/scenarios/scenarios.json` with
   `--instances-per-pattern 6` (was 3) — 102 scenarios / 258
   applications (was 51 / 129), re-verified event-by-event against a
   real Postgres test database (`scripts/generate_scenarios.py`'s
   built-in verification: all 258 events applied without error). This
   cascaded into four hardcoded test-count assertions in
   `tests/test_gnn.py` (51→102 scenarios, 129→258 applications,
   86→172 train, 43→86 eval), all updated to the real regenerated
   counts, not guessed.
3. **Regularization**: added a `--weight-decay` CLI flag to
   `training/train.py`, wired into `optim.Adam(..., weight_decay=...)`
   (default `0.0`, preserving old behavior when unset). Retrained with
   `--weight-decay 1e-4 --dropout 0.2` (dropout was 0.1).

Even with both regularizers, the retrained run still drove training
loss to `0.0004` and internal held-out eval accuracy to `1.0`
(`14/14` GENUINE, `72/72` SUSPICIOUS) — reported here exactly as
measured, not smoothed over. This internal eval split is drawn from the
*same* generator patterns as the training set (different token values,
same structural shape per pattern), so a single held-out instance of a
pattern is not a strong test of generalization to genuinely novel
input shape; `scripts/evaluate_fraud_detection.py`'s full hybrid-pipeline
replay (below) is the more meaningful measurement, and its own numbers
are far more mixed than "1.0" — see below.

**A real methodology bug caught mid-phase, not glossed over**:
the first post-retraining run of `scripts/evaluate_fraud_detection.py`
reported a suspicious `1.0` interleaved accuracy (`216/216` correct, 0
FP, 0 FN). Investigating rather than reporting it: `scripts/
generate_scenarios.py`'s verification step (run earlier in this phase
to regenerate the dataset) calls `GraphStore.truncate_all()` then
applies all 258 events directly against the graph tables — but leaves
that fully-connected graph state sitting in the test database
afterward, it does not truncate on exit. `evaluate_fraud_detection.py`'s
`_score_interleaved()` assumes it's starting from an empty graph and
replays events one at a time to simulate realistic incremental
arrival; run right after the verification step, every "first" event it
replayed was actually being scored against a graph that already had
every other event in it, collapsing the incremental-arrival simulation
entirely and inflating the score. Fixed by explicitly truncating both
the event-repository and graph-store tables before every real
evaluation run below; this is a pre-existing sequencing hazard between
these two scripts (not introduced by this phase's dataset regeneration
choice specifically) worth remembering for any future re-run.

### Real, measured before/after — main dataset (`scenarios.json`)

Both rows below are `scripts/evaluate_fraud_detection.py` runs against
a freshly truncated database — "OLD" is the checkpoint/dataset/rules
that existed before this phase (51 scenarios / 129 applications);
"NEW" is Stage 3 + Stage 4 combined (102 scenarios / 258 applications).

| metric (interleaved mode) | OLD | NEW |
|---|---|---|
| accuracy | 0.7364 (95/129) | 0.7326 (189/258) |
| precision | 0.9512 | 0.9804 |
| recall | 0.7222 | 0.6944 |
| hard-negative accuracy | 0.8333 (30/36) | 0.9167 (33/36) |
| isolated-mode accuracy | 0.7132 | 0.7326 |

The interleaved/isolated gap the rules.py docstring calls the "early-
arrival blind spot" (2.3 points OLD) is gone in the NEW run — both
modes now score identically (0.7326). Hard-negative accuracy (the
metric most directly tied to the user's complaint) improved. Recall
dropped slightly (72.2%→69.4%); the per-pattern breakdown
(`models/hybrid_scoring_eval.json`) shows every purely-GENUINE pattern
now at 100% correct except `legitimate_repeated_application` (75%,
still 3 false positives), while several two-party SUSPICIOUS patterns
(`same_phone_different_identities`, `same_borrower_2_platforms`,
`same_address_with_additional_signal`, `conflicting_identity_
information`) sit at 50%. Checked the actual misclassified rows rather
than assuming: their `ruleScore` is `0.0` and `fraudScore` is `~0.01`,
consistent with these being the *first*-arriving application of a
two-application pair in the interleaved chronological ordering — the
documented early-arrival blind spot, not a new regression from the
two-party cap (whose 0.35 threshold a single 0.30-weight phone/email
signal doesn't even reach).

### Real, measured before/after — the large-scale FPR dataset (Phase 21's `scenarios_fpr.json`, 270 scenarios / 690 applications)

This is the dataset that originally exposed the platform-bias problem,
re-run here on a freshly truncated database against both the old and
new checkpoints for a direct, apples-to-apples comparison at the same
statistically meaningful scale Phase 21 used:

| metric (interleaved mode) | OLD (Phase 21) | NEW (Phase 31) |
|---|---|---|
| accuracy | 0.6812 | 0.7565 |
| precision | 0.8478 | 0.9921 |
| recall | 0.7222 | 0.6944 |
| hard-negative accuracy | 0.5556 (75/135) | **1.0000 (135/135)** |
| `normal_genuine_baseline` pattern | 0.3333 (5/15) | 0.8000 (12/15) |
| `legitimate_repeated_application` pattern | 0.5000 (15/30) | **1.0000 (30/30)** |
| `legitimate_repeated_application_three_times` pattern | **0.0000 (0/45)** | **1.0000 (45/45)** |

The specific pattern Phase 21 evidenced as the platform-bias victim
(`legitimate_repeated_application_three_times`, hardcoded to
`PLATFORMS[2]`) went from *zero* correct out of 45 to all 45 correct —
the strongest single piece of evidence this phase has that the
platform-hardcoding hypothesis was real and the fix addressed it, not
just a hypothesis anymore. Hard-negative accuracy across the whole
690-application set went from barely-better-than-a-coin-flip (55.6%)
to perfect (100%). `normal_genuine_baseline` improved sharply
(33.3%→80.0%) but is not fully fixed: the 3 remaining misclassified
applications have `ruleScore: 0.0` (correctly, nothing to flag) and
`fraudScore: ~0.9999` — the GNN alone, with no rule support, is still
occasionally near-maximally confident on a lone genuine baseline
application with no connections at all. That is a real, unresolved
residual calibration issue on this specific pattern, reported here
rather than rounded up to "fixed."

### What this phase does and does not claim

Fixed and verified: the rule engine no longer sums correlated
same-person evidence without limit (Stage 3), the explanation
surfaces when that cap applied (Stage 3), the documented
platform-hardcoding training bias is gone by the strongest available
evidence (a 0%→100% pattern-level flip on the exact dataset that
found it), and hard-negative/false-positive accuracy improved
substantially on both the main dataset and the large-scale FPR
dataset. Not fixed: recall on genuinely fraudulent two-party overlaps
is now the weaker side of the tradeoff (many first-arrival
misclassifications, a pre-existing structural limitation of
incremental scoring rather than a new one), and a small residual GNN
overconfidence issue remains on isolated genuine baseline
applications. `models/gnn_checkpoint.pt`/`.meta.json`,
`models/baseline_metrics.json`, `models/hybrid_scoring_eval.json`, and
`models/hybrid_scoring_eval_fpr.json` were all regenerated for real
this phase and reflect these exact numbers.
