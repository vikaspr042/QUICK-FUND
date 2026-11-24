# KNOWN ISSUES — consolidated, final (Phase 26)

Every item below was found by real testing, real measurement, or a real,
confirmed sandbox constraint — never guessed at or asserted without
evidence. This file consolidates the open items scattered across
`TROUBLESHOOTING.md`, `SECURITY.md`, `ML_MODEL.md`, `PERFORMANCE.md`, and
`BUILD_STATE.json`'s `known_errors` (26 phases' worth) into one place, so
whoever picks this project up next doesn't have to reconstruct the full
picture from 26 separate phase entries. Each item links back to its full
investigation for detail; this file is a summary, not a replacement.

Nothing here is fixed simply by being listed. Some items are genuinely
unresolved defects; others are sandbox constraints with no fix available
in any environment this project has been built in. Both kinds are listed
honestly, distinguished explicitly.

## A. Fraud-detection accuracy (real, unresolved — needs ML work, not testing)

1. **GNN live-inference collapse — FIXED in Phase 27, root cause
   confirmed and verified.** Previously: the trained Phase 13 checkpoint
   returned a near-zero score (raw logit ≈ -121 to -125) for every
   genuinely live-submitted application, regardless of input. Phase 27
   root-caused this for real (`central-fraud-intelligence/src/gnn/
   features.py`'s `_scaled_timestamp()`): the timestamp feature measured
   absolute distance from a fixed 2026-01-01 calendar epoch, and a live
   application's `submittedAt` is real wall-clock time — every day past
   that fixed date pushed the feature further outside anything the model
   was ever trained on (training data never exceeded a measured value of
   5.0; a live September 2026 request measured ~8.7 and growing daily).
   Reproduced directly: the identical graph structure (shared phone+email
   cluster) scored 1.000 at the training epoch, 0.33 one month later
   (in-range), and 0.000 at a real live timestamp — isolating this one
   feature, not graph size, as the cause (a clamp-only attempt was tried
   first and did NOT fix it — the model had also learned to associate
   large absolute offsets with the dataset's own "genuine reapplication
   months later" pattern). **Fix:** the feature is now computed relative
   to the earliest related application actually present in the graph/
   subgraph being scored, not a fixed calendar date, falling back to the
   old fixed-epoch behavior only when there's nothing to compare against
   (an isolated application, where the collapse was never the reported
   problem). No retraining needed — the existing checkpoint's weights are
   unchanged; the fix is verified to be a no-op on the offline evaluation
   (73.64% accuracy, unchanged from the pre-fix baseline) while
   restoring correct live behavior (re-verified live-timestamp repro:
   rapid multi-platform cluster now scores 1.000 live, a genuinely
   ~5-months-apart pair scores 0.14 live — the intended distinction,
   working under real wall-clock time). Full central suite re-run after
   the fix: 265/265 passing. See `TROUBLESHOOTING.md`'s Phase 27 entry
   for the full repro script and reasoning.
2. **False-positive rate and a real, evidenced (not proven) platform-bias
   correlation.** A 690-application re-measurement found hard-negative
   accuracy of only 55.56% (75/135); the adversarial "third repeat
   application" pattern was misclassified 100% of the time; even the
   single simplest genuine case in the whole catalog was only 33.3%
   accurate (5/15), with every one of its 10 misclassifications landing
   on platforms B or C while every A/D instance was correctly cleared —
   notable because that pattern's platform assignment is genuinely
   randomized in the generator, ruling out a generator artifact.
   Hypothesis (not confirmed by ablation): two related adversarial
   patterns hardcode their platform in the generator, which combined with
   Phase 13's already-documented severe training overfit may have taught
   the model a spurious platform-based shortcut. Retraining with platform
   assignment randomized is the natural next step to actually test this.
   See `ML_MODEL.md`'s Phase 21 section for the full synthesis, including
   the earlier, related Phase 15 finding (legitimate same-borrower repeat
   applications scored ~0.9999 by the GNN alone, while the rule engine
   correctly stayed at 0.0).
3. **Hybrid-scoring first-arrival blind spot** (structural, not a code
   defect). Real-time single-pass scoring cannot detect a fraud ring's
   first-arriving member — at scoring time none of its co-conspirators'
   events have arrived yet, so its subgraph looks identical to an
   isolated genuine applicant's. Confirmed via real two-mode evaluation:
   100% of false negatives in both interleaved and isolated replay are
   exactly the first-submitted application in their scenario. A
   re-scan/re-scoring trigger once later ring members arrive would
   correct this on a subsequent pass; no such trigger exists in this
   project. See `TROUBLESHOOTING.md`'s Phase 15 entry.

## B. Security hardening (real, deliberately deferred — each with a stated reason)

4. **`starlette` (transitive FastAPI dependency): 14 pip-audit
   advisories** at the installed version. A fix needs a `starlette`
   major-version bump, likely forcing a `fastapi` bump too — most
   advisory categories don't apply to this app's actual code paths
   (confirmed by grep — no `StaticFiles`/`HTTPEndpoint`/`FileResponse`
   usage), but 2 of 7 CVE categories do touch this app's real
   `Form()`-parsing usage.
5. **`vite`/`esbuild` dev-server CVE** (all four frontends) — dev-only,
   does not affect the production `vite build` artifact shipped in
   Docker images. Full fix needs a breaking `vite@8.3.0` upgrade, first
   found at Phase 1, re-confirmed still present at Phase 22.
6. **No rate limiting** on any public-facing endpoint (login, application
   submission, central ingestion) on any of the 5 services.
7. **No CSRF token** for the Phase 19 dashboard's mutating routes — relies
   only on the session cookie's `SameSite=Lax` attribute.
8. **Secrets rotation procedure not documented.** Phase 25 gave production
   secrets a concrete *storage* mechanism (AWS Secrets Manager, one
   secret per sensitive value — see `DEPLOYMENT.md`'s Cloud: AWS section)
   but writing and testing an actual rotation Lambda was judged out of
   scope for a phase with no real AWS account to test it against.

Items 4-7 were all found and deliberately not fixed in Phase 22, each
judged to need its own dedicated, carefully-tested pass rather than a
hasty fix squeezed into an already-large security-testing phase — the
same reasoning applied to major-version dependency bumps since the
vite/esbuild finding at Phase 1. See `SECURITY.md`'s Phase 22 section.

## C. Performance (real, understood, not fixed — architecture-level change)

9. **No Postgres connection pooling.** Every repository in
   `central-fraud-intelligence` opens a brand-new `psycopg.connect()` per
   method call — confirmed by reading `_connect()` in every repository. A
   single ingestion request performs a dozen-plus separate connect-and-
   auth round-trips. Leading, code-grounded explanation for the measured
   ~280-320ms single-request latency floor and sub-linear concurrency
   scaling (see `PERFORMANCE.md`). Not a correctness bug — every
   connection is used correctly and closed — purely a latency/throughput
   cost. Introducing real pooling (e.g. `psycopg_pool`) across every
   repository is an architecture-level change, out of a performance-
   *testing* phase's scope.

## D. Infrastructure / deployment (real, confirmed sandbox constraints — not code defects)

10. **`docker compose up --build` has never been run to completion in any
    sandbox this project has used.** The Docker daemon itself genuinely
    works (verified with a real registry-free `FROM scratch` build+run),
    but every container registry this project's Dockerfiles need
    (`registry-1.docker.io`, `ghcr.io`, `mirror.gcr.io` — all three
    tested directly, not assumed) returns `403` under this sandbox's
    egress policy, confirmed via the proxy's own diagnostic log as a
    named policy denial. `scripts/verify_compose.sh` verifies everything
    about the Docker artifacts that doesn't need a registry (compose
    structure, every Dockerfile's `COPY` paths, port uniqueness, real
    per-platform database-network isolation, the registry-free daemon
    capability probe) and passes on all of it; it honestly reports the
    one check that needs registry access as **skipped**, not passed.
11. **Cloud deployment (AWS) has never been exercised.** Phase 25 wrote
    real, reviewed Terraform IaC and a CI/CD pipeline, but no AWS account
    or credentials exist in any sandbox this project has used, and the
    Terraform CLI itself could not even be installed
    (`releases.hashicorp.com` also blocked, confirmed the same way).
    Real substitute check actually run: every `.tf` file parsed as
    syntactically valid HCL2, and the CI workflow parsed as valid YAML —
    both explicitly weaker than `terraform validate`, reported as such.
    Per this project's standing rule (`DEPLOYMENT.md`), no cloud
    deployment is ever claimed to have happened, and none has.
12. **A real MongoDB has never been reachable in any sandbox this project
    has used** (Docker Hub blocked for `mongo:7`; `fastdl.mongodb.org`/
    `downloads.mongodb.org` also blocked when a local binary install was
    considered; Ubuntu's `mongodb` apt package was dropped years ago over
    SSPL licensing). All four loan platforms' backend test suites use the
    same in-memory repository implementation the application code itself
    is written against — the same interface, same app/service/route code
    is exercised either way — but the real Mongoose driver code path is
    genuinely unexercised until Docker actually runs somewhere with
    registry access.
13. **PostgreSQL, by contrast, IS reachable in this build sandbox** (a
    normal Ubuntu package, never dropped) — this is why
    `central-fraud-intelligence`'s tests run against a real local
    Postgres database in every phase since Phase 8, a materially stronger
    verification path than the loan platforms get for MongoDB. Worth
    stating explicitly so the asymmetry isn't mistaken for an oversight.
14. **Update to item 10 — `docker compose up --build` has now actually
    been run to completion, for real, by a user with normal registry
    access (Phase 32), and it found two real bugs this project's own
    never-fully-run Dockerfiles had been carrying: (a)
    `central-fraud-intelligence/Dockerfile` never `COPY`'d `datasets/` or
    `training/` into the image (only `src`, `models`, `scripts` were),
    so any of the dev-tooling scripts that import `datasets.
    scenario_generator` or `training.dataset` (`scripts/
    generate_scenarios.py`, `scripts/evaluate_fraud_detection.py`,
    `training/train.py`, `training/baseline.py`) failed inside the
    running container with `ModuleNotFoundError`; and (b) `.dockerignore`
    separately excluded `datasets` wholesale, which would have silently
    defeated the fix to (a) even after adding the `COPY` line. Both
    fixed for real this phase.** Separately, and NOT fixed (a real
    design gap, not a one-line bug): those same dev-tooling scripts
    compute their file paths assuming the full project layout on disk
    (`training/dataset.py`'s `REPO_ROOT = Path(__file__).resolve().
    parents[2]`, which finds the real repo root when run from a dev
    checkout, but resolves to the filesystem root `/` inside the
    container, where only `central-fraud-intelligence`'s own contents
    exist under `/app`). These scripts were always meant to be run
    during development, before `docker build` — the same way this
    project's own retrained `models/gnn_checkpoint.pt` was actually
    produced — not inside the deployed container, where `seed-data/` was
    never copied in and the path math doesn't match the layout anyway.
    Running them inside a live container remains genuinely broken;
    running the live API itself is unaffected (it only ever reads
    `models/`, which is copied in correctly).

### Historical findings, superseded by later phases (kept for the record, not currently open)

- *"Docker daemon cannot start in this sandbox"* (Phase 20) was a real
  finding for that session. Phase 24 re-investigated from scratch in a
  different session and found the daemon genuinely does start (running
  `dockerd` directly instead of through the `service`/systemd wrapper) —
  item 10 above is the current, refined understanding: the daemon works,
  registry access is the real, remaining blocker. Both findings were real
  for their respective sessions; sandbox capabilities can and evidently
  do vary between sessions.

## What this means for anyone picking this project up for real production use

Every one of items 10-13 traces back to the same root cause, stated
plainly: **this project has never been built in a sandbox with normal
internet/container-registry/cloud access.** That has been the single most
recurring, honestly-documented constraint since Phase 1. The first thing
a team doing that should do is exactly what every phase's Docker/cloud
section has asked for: run `docker compose up --build` and
`scripts/verify_compose.sh` somewhere with real registry access, and run
`terraform init && terraform validate && terraform plan` in
`deploy/aws/terraform/` with real (even sandbox-tier) AWS credentials.
Everything else in this repository — all application logic, all 591
automated tests, all measured performance numbers — has been verified for
real, in this environment, repeatedly, across 26 phases; see `PROGRESS.md`
for the complete phase-by-phase record.

Items 1-3 (ML accuracy) and 4-9 (security/performance hardening) are real
engineering work, not sandbox limitations, and are ready to be picked up
by a future phase or a production-hardening effort with no further
investigation needed to get started — each has a stated hypothesis or
concrete next step above.
