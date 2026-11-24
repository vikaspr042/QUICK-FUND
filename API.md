# API

Status: **grown incrementally** — every endpoint listed as "live" here must
exist and actually respond in this repository (and, where applicable, be
covered by an automated test) before it's documented as such; endpoints are
added to the live table in the same phase that implements them, never in
advance.

## Live

### `loan-platform-a` (QuickFund) — Phase 2 (seed data: Phase 6, outbox: Phase 7, profile: Phase 30)

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/health` | none | `{ ok: true, service: "loan-platform-a" }` — Docker healthcheck target |
| GET | `/api/v1/health` | none | Adds platform name + build phase |
| POST | `/api/v1/auth/register` | none | `{ name, email, password, phone }` -> `{ user, token }` |
| POST | `/api/v1/auth/login` | none | `{ email, password }` -> `{ user, token }` |
| POST | `/api/v1/auth/logout` | none | client-side token discard; server ack |
| GET | `/api/v1/profile` | user | **New in Phase 30.** `{ id, name, email, phone, address }` for the authenticated user - separated out of the loan application form (see the Phase 30 paragraph below). |
| PATCH | `/api/v1/profile` | user | **New in Phase 30.** `{ name?, phone?, address? }` (at least one required) -> the updated profile. `email` is deliberately not editable here (it's the login identity). 400 if no fields given or a given field is empty/non-string. |
| POST | `/api/v1/applications` | user | `{ applicant, loan, documents? }` -> created application (assigns `applicationId`, e.g. `A1001`); `applicant` is now populated from the user's own saved profile by the frontend (ApplyLoan.jsx), not re-typed - the API itself is unchanged and still validates it the same way. |
| GET | `/api/v1/applications/history` | user | this user's own applications, newest first |
| GET | `/api/v1/applications/:applicationId/status` | user | 404 if unknown, 403 if not the owner |
| GET | `/api/v1/repayments` | user | this user's own repayment installments |
| POST | `/api/v1/repayments/:repaymentId/pay` | user | marks a PENDING installment PAID (demo mode, no real payment gateway); 409 if already paid, 403 if not the owner |
| POST | `/api/v1/admin/login` | none | same as user login but 403s if the account's role isn't `admin` |
| GET | `/api/v1/admin/applications` | admin | optional `?status=SUBMITTED\|UNDER_REVIEW\|APPROVED\|REJECTED` |
| GET | `/api/v1/admin/applications/:applicationId/repayments` | admin | repayment schedule for one application |
| POST | `/api/v1/admin/applications/:applicationId/approve` | admin | `{ note? }`; generates the repayment schedule; 409 on an invalid status transition |
| POST | `/api/v1/admin/applications/:applicationId/reject` | admin | `{ note? }`; 409 on an invalid status transition |
| POST | `/api/v1/admin/applications/:applicationId/request-review` | admin | `{ note? }`; SUBMITTED -> UNDER_REVIEW only |
| POST | `/api/v1/fraud-notifications/receive` | `X-API-Key` (this platform's own `CENTRAL_FRAUD_API_KEY`) | Real as of Phase 18. The endpoint central's `NotificationDeliveryWorker` targets for every notification naming this platform. `X-Idempotency-Key` (central's own `notification_id`, e.g. `NOTIF-INC-evt-1-a`) is required and becomes this alert's `alertId` - the body itself carries no notification id of its own (see `ARCHITECTURE.md` section 8's "Fraud notifications" subsection). Body: `{ incidentId, triggeringApplicationId, triggeringPlatformId, finalRiskScore, primaryDriver, explanationSummary, affectedApplications, affectedPlatforms, createdAt }` -> `{ received: true, alertId, duplicate, alert }`. Persists into a local `fraud_alerts` Mongo collection, idempotent on `alertId` (upsert - a redelivery never creates a duplicate). 401 if `X-API-Key` is missing/wrong, 400 if `X-Idempotency-Key` or a required body field is missing. |
| GET | `/api/v1/admin/fraud-alerts` | admin | Real as of Phase 18. Every fraud alert this platform has received, newest first. No filtering/pagination yet. |
| POST | `/api/v1/admin/fraud-alerts/:alertId/review` | admin | Real as of Phase 19. The one-way not-acknowledged -> acknowledged transition (see `models/FraudAlert.js`'s module docstring for why this is a boolean, not a multi-state workflow, unlike central's own incident review). `{ note? }` -> the updated alert (`acknowledged: true`, `acknowledgedBy`, `acknowledgedNote`, `acknowledgedAt`). 401 if not an admin, 404 if no alert with that `alertId` exists, 409 if it's already acknowledged. |

Covered by 92 automated tests (`loan-platform-a/backend/tests/`) — auth,
profile get/update (Phase 30, 9 tests), authorization/JWT edge cases,
application validation/ownership, admin status transitions, repayment
generation/payment, one full end-to-end flow, the seed script (Phase 6,
now also asserting a seeded user's address is really persisted - Phase
30), the outbox event creation + delivery worker (Phase 7),
tokenization/encryption of the outbox payload (Phase 9, 13 tests — see
`SECURITY.md`), and receiving/listing/acknowledging fraud alerts (Phase 18,
14 tests + Phase 19, 5 tests for acknowledgement).

### `loan-platform-b` (TrustLend) — Phase 3 (seed data: Phase 6, outbox: Phase 7, profile: Phase 30)

Same endpoint surface as loan-platform-a (including the Phase 18
`POST /api/v1/fraud-notifications/receive` and
`GET /api/v1/admin/fraud-alerts` endpoints, the Phase 19
`POST /api/v1/admin/fraud-alerts/:alertId/review` endpoint, and the Phase
30 `GET`/`PATCH /api/v1/profile` endpoints), with one schema difference:
`POST /api/v1/applications`'s `loan` object additionally requires
`institutionName` and `courseName` (education-loan-specific — 400 if
missing). Covered by 93 automated tests (the same 92 as loan-platform-a,
plus one for the education-field validation).

### `loan-platform-c` (SecureCredit) — Phase 4 (seed data: Phase 6, outbox: Phase 7, profile: Phase 30)

Same endpoint surface as loan-platform-a (including the Phase 18
`POST /api/v1/fraud-notifications/receive` and
`GET /api/v1/admin/fraud-alerts` endpoints, the Phase 19
`POST /api/v1/admin/fraud-alerts/:alertId/review` endpoint, and the Phase
30 `GET`/`PATCH /api/v1/profile` endpoints), with one business-rule
difference: `POST /api/v1/applications`'s `loan.amount` is capped at 50000
(INR) and `loan.tenureMonths` at 12 — a 400 naming the ceiling exceeded if
either is over, enforced in `applicationService.submit()` and mirrored as
a Mongoose `max` validator. Application IDs use the `C` prefix (e.g.
`C1001`). Covered by 95 automated tests (the same 92 as loan-platform-a,
plus 3 for the micro-loan ceilings: amount over, tenure over, and
exactly-at-ceiling accepted).

### `loan-platform-d` (FinBridge) — Phase 5 (seed data: Phase 6, outbox: Phase 7, profile: Phase 30)

Same endpoint surface as loan-platform-a, with one schema difference:
`POST /api/v1/applications`'s `loan` object additionally requires
`merchantName` (any non-empty string) and `productCategory` (must be one
of `electronics`, `appliances`, `furniture`, `fashion`, `education-tech`,
`other` — a categorical/enum check, distinct in kind from loan-platform-b's
required-field check and loan-platform-c's numeric-ceiling check). 400 if
either is missing or `productCategory` isn't in the allowed list.
Application IDs use the `D` prefix (e.g. `D1001`). Same endpoint surface
as loan-platform-a for the Phase 18
`POST /api/v1/fraud-notifications/receive` and
`GET /api/v1/admin/fraud-alerts` endpoints, the Phase 19
`POST /api/v1/admin/fraud-alerts/:alertId/review` endpoint, and the Phase
30 `GET`/`PATCH /api/v1/profile` endpoints, too. Covered by 94 automated
tests (the same 92 as loan-platform-a, plus 2 for the merchant/category
validation).

### Profile / loan-application separation + repayment gateway visibility (all four loan platforms) — Phase 30

**What changed and why**: the user reported that every platform's "Apply
for a loan" form asked for the applicant's name/email/phone/address fresh
on every submission, mixed in with the loan-specific fields - there was
no place those personal details lived on their own, and no dedicated
profile page. Root cause confirmed by reading `ApplyLoan.jsx` on all four
platforms: identical pattern, `applicant` fields as ordinary form state
alongside `loan` fields, submitted together. A real repayment feature
(`GET /api/v1/repayments`, `POST /api/v1/repayments/:id/pay`, `Repayments.jsx`,
already linked in every platform's nav) already existed in demo/simulated
mode ("Pay now (demo)", no real payment gateway) - the user's separate
"add a repayment gateway" ask was addressed by confirming this and adding
a summary strip (outstanding balance, paid so far, next due date) to that
existing page, not by building a new feature from nothing.

**Backend** (`models/User.js`, `repositories/userRepository.js`,
`services/profileService.js` (new), `routes/profile.routes.js` (new),
`app.js`): `User` gained an `address` field; the repository gained
`update(id, fields)` (both the Mongoose and in-memory implementations);
the new profile service/routes expose it as `GET`/`PATCH /api/v1/profile`
(see the per-platform tables above) - email is deliberately not editable
through this endpoint. `scripts/seed.js` updated to persist each seeded
user's `address` onto their real `User` record (`userRepository.update()`),
not just merge it into an in-memory value for the seed script's own use -
without this fix a seeded demo account would have been blocked by
ApplyLoan's new profile-complete gate.

**Frontend** (`pages/Profile.jsx` (new), `pages/ApplyLoan.jsx`,
`pages/Repayments.jsx`, `App.jsx`, `api.js`): a new Profile page
(view/edit name, phone, address; email shown read-only) with its own nav
link. `ApplyLoan.jsx` no longer asks for applicant fields at all - it
fetches the saved profile, shows it as a read-only summary with an "Edit
profile" link, and blocks submission (with a link to Profile) until the
profile has a real address on file; the `applicant` object the backend
still requires is built from the fetched profile. `Repayments.jsx` gained
the summary strip described above. No visual/UX change to any other page.

**Tests**: 9 new backend tests per platform (`tests/profile.test.js` -
auth required on both routes, get/update, partial updates, empty-field
and non-string-field rejection, email immutability), plus one seed-data
regression test per platform confirming a seeded user's address is
actually persisted, not just held in memory. Full suite re-run on all
four platforms: **A 92/92, B 93/93, C 95/95, D 94/94** (all real
integration tests against a real Express app, in-memory repositories -
see `TESTING.md`). All four frontend production builds
(`vite build`) verified to succeed with no errors.

**Honest scope note**: not verified against a live browser/Docker
environment in this sandbox (see `PROGRESS.md`'s Phase 8 note on why -
MongoDB/Docker Hub access is blocked here) - verification is the same
level this project has always used for the loan platforms' frontends
(real integration tests + a successful production build), not a
live-rendered screenshot, which for these four platforms would need a
real MongoDB this sandbox cannot reach.

### Outbox mechanism (all four loan platforms) — Phase 7, payload content updated Phase 9

Not an HTTP endpoint - an internal mechanism, per platform, that is what
eventually lets the central application learn about applications and
status changes without ever querying a platform's database directly
(ARCHITECTURE.md). `applicationService.submit()` and every successful
`adminService` transition (approve/reject/request-review) write one
`application_outbox` row each (`APPLICATION_SUBMITTED` /
`APPLICATION_STATUS_CHANGED`), and `src/outbox/deliveryWorker.js` polls
for PENDING rows every 5s and POSTs each one, with an `X-Idempotency-Key`
(the event's `eventId`) and `X-API-Key` (`CENTRAL_FRAUD_API_KEY`), to
`{CENTRAL_FRAUD_API_URL}/api/v1/applications/process` - retrying with
capped exponential backoff (8 attempts, 1s base, 30s cap) before marking an
event `FAILED`. Verified for real in each platform's `tests/outbox.test.js`
against a local stub HTTP server, and end-to-end against a real running
central app (see `PROGRESS.md`'s Phase 8 session log).

As of Phase 9, `applicationService.submit()` tokenizes/encrypts the
applicant fields (`tokenizeApplicant()`/`encryptField()`,
`src/tokenization/`) *before* constructing the outbox row, so the raw
`applicant` object is never present in `application_outbox.payload` at
all - see `SECURITY.md`'s Phase 9 section and `DATA_FLOW.md`'s Step 3
update for the exact payload shape now sent.

### Central Fraud Intelligence + GNN (`central-fraud-intelligence`) — Phase 1 + Phase 8 + Phase 9 + Phase 10 + Phase 14 + Phase 15 + Phase 16 + Phase 17 + Phase 18 + Phase 19 + Phase 29

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/` | none | **Changed in Phase 29.** Redirects (303) to `/dashboard` - a browser hitting the bare service URL now lands on the actual dashboard UI (further redirected to `/dashboard/login` or `/dashboard/overview` depending on its cookie) instead of a JSON blob. |
| GET | `/api` | none | **New in Phase 29.** The JSON discovery payload `GET /` used to return directly - unchanged in content, just moved here now that `/` is a browser-facing redirect. `{ service, version, status, message }`. |
| GET | `/health` | none | `{ ok: true, service: "central-fraud-intelligence" }` |
| GET | `/api/v1/health` | none | Adds version + env |
| POST | `/api/v1/admin/login` | none | Real as of Phase 9. `{ email, password }` -> `{ email, token }`; bcrypt-checks against the `admin_users` Postgres table, issues a 24h HS256 JWT (`src/security/jwt_auth.py`). 401 for both wrong password and unknown email (anti-enumeration, deliberately identical response). No public register endpoint - admin accounts are provisioned via `scripts/create_admin.py` (mirrors every loan platform's own `createAdmin.js`). |
| POST | `/api/v1/applications/process` | `X-API-Key` (per-platform shared secret) | Real as of Phase 8 — the endpoint every loan platform's outbox delivery worker targets. `{ eventId, eventType, applicationId, platformId, ... }` -> `{ eventId, status, duplicate, fraudScore, ruleScore, triggeredRules, finalRiskScore, explanation, incident, ... }`. `status` is now one of `RECEIVED`/`GRAPH_UPDATED`/`ANALYZED`/`CLEARED`/`FRAUD_DETECTED` (see `ARCHITECTURE.md` section 7's status progression). Idempotent on `eventId`/`X-Idempotency-Key` (real Postgres `ON CONFLICT DO NOTHING`, not just an app-level check). 401 if `X-API-Key` missing/invalid, 403 if it authenticates a different platform than `platformId` claims, 400 if `X-Idempotency-Key` doesn't match `eventId`, 422 if the event shape is malformed, 503 if the central database is unreachable (either during event persistence or the graph-update step). As of Phase 9 the payload it receives already arrives tokenized/encrypted (see the outbox mechanism note above) - this endpoint does not itself tokenize or decrypt anything. As of Phase 10, the same request also updates the persistent graph (`src/graph/`) - node/edge upserts keyed on the event's tokens, real Postgres rows plus an in-memory NetworkX materialization - which is what advances `status` from `RECEIVED` to `GRAPH_UPDATED`; see `DATA_FLOW.md`'s Step 4 update and `ARCHITECTURE.md` section 6. As of Phase 14, once `status` is `GRAPH_UPDATED`, the same request also runs real GNN inference (`src/gnn/inference.py`'s `GNNScorer`, a trained `HeteroGraphSAGE` checkpoint from Phase 13) against the event's bounded subgraph, producing a real `fraudScore` in `[0, 1]` (higher = more suspicious) and advancing `status` to `ANALYZED`. This step is best-effort: if no checkpoint was loaded at startup, or scoring fails for any reason, `fraudScore` is `null`, `status` stays at `GRAPH_UPDATED`, and the request still returns 200 - a missing/broken model never fails ingestion (retried by the next redelivery, same as the graph step). As of Phase 15, once `status` is `ANALYZED`, the same request also runs the deterministic rule engine (`src/fraud_detection/rules.py`) against the same bounded subgraph and combines it with `fraudScore` (`src/fraud_detection/scoring.py`'s `combine_signals()`, a 0.6/0.4 weighted average - see `ARCHITECTURE.md` section 8), producing a real `ruleScore` in `[0, 1]`, a `triggeredRules` array (the specific shared-token/document-reuse signals that fired, each with its own weight and detail), and a `finalRiskScore` in `[0, 1]` that decides `status`: `FRAUD_DETECTED` if `finalRiskScore >= 0.5`, else `CLEARED`. This step is best-effort the same way: a rule-engine or combination failure leaves `ruleScore`/`triggeredRules`/`finalRiskScore` `null` and `status` at `ANALYZED`, retried by the next redelivery. As of Phase 16, the response also carries `explanation`: `{ summary: <human-readable string>, technical: { decision, finalRiskScore, threshold, gnn: {score, configuredWeight, effectiveWeight, weightedContribution}, rules: {score, configuredWeight, effectiveWeight, weightedContribution, triggeredRules}, primaryDriver, caveats: [...] } }` (`src/explainability/explain.py`'s `generate_explanation()`) - computed fresh from the fields already in this same response, not a new pipeline step or a new persisted column; `null` only if explanation formatting itself unexpectedly fails (defensive, not expected in practice). As of Phase 17, once `status` reaches `FRAUD_DETECTED`, the same request also creates a real, persisted incident (`src/incidents/`) and the response carries `incident`: `{ incidentId, triggeringEventId, triggeringApplicationId, triggeringPlatformId, finalRiskScore, ruleScore, fraudScore, triggeredRules, explanationSummary, explanationTechnical, primaryDriver, affectedApplications: [{applicationId, platformId}, ...], affectedPlatforms: [...], reviewStatus, createdAt }` - `null` if `status` isn't `FRAUD_DETECTED`, or if incident creation hasn't succeeded yet (best-effort, retried by redelivery - see `src/incidents/incidents.py`'s module docstring). `affectedApplications`/`affectedPlatforms` are derived from the same bounded subgraph the rule engine/GNN already scored, so an incident can genuinely span more than one application/platform. Idempotent on `eventId` the same way event persistence is (`incidentId` is deterministically `INC-<eventId>`) - a redelivered event never creates a second incident for itself. As of Phase 18, whenever an incident was just created, the same request also creates one real, persisted notification row per platform in that incident's `affectedPlatforms` (`src/notifications/`) and the response carries `notifications`: `[ { notificationId, incidentId, platformId, status, attempts, lastError, nextAttemptAt, sentAt, createdAt }, ... ]` - `[]` if there's no incident. Notification *creation* is synchronous here and idempotent (`notification_id` is deterministically `NOTIF-<incidentId>-<platformId>`, a redelivered event never creates duplicates); *delivery* to each platform is out of band, done by a background worker (`src/notifications/worker.py`'s `NotificationDeliveryWorker`), never inside this request - see each notification's own `status` (PENDING/SENT/FAILED) for its current delivery state. |
| GET | `/api/v1/applications/:applicationId` | admin (`Authorization: Bearer <JWT>`) | Real as of Phase 8; auth added Phase 9 (previously open - see `SECURITY.md`). Every event received so far for one application, each now including its `fraudScore` (Phase 14), `ruleScore`, `triggeredRules`, `finalRiskScore` (Phase 15, all `null`/`[]` if that event hasn't reached that pipeline step yet), `explanation` (Phase 16, same shape as the POST endpoint's, computed fresh per event), `incident` (Phase 17, same shape as the POST endpoint's, `null` unless that event's status is `FRAUD_DETECTED` and its incident exists), and `notifications` (Phase 18, same shape as the POST endpoint's, `[]` if there's no incident). 401 if the token is missing/invalid/expired, 404 if no events for that application. |
| GET | `/api/v1/incidents` | admin (`Authorization: Bearer <JWT>`) | Real as of Phase 17. Every persisted incident, newest first, optionally bounded by `?limit=` (default 100). `{ incidentCount, incidents: [ {same shape as the POST endpoint's `incident`}, ... ] }`. 401 if the token is missing/invalid/expired, 503 if the central database is unreachable. |
| GET | `/api/v1/incidents/:incidentId` | admin (`Authorization: Bearer <JWT>`) | Real as of Phase 17. One incident by its `incidentId` (same shape as the POST endpoint's `incident`, now also including `reviewHistory` - Phase 19, see below). 401 if the token is missing/invalid/expired, 404 if no incident with that id exists, 503 if the central database is unreachable. |
| POST | `/api/v1/incidents/:incidentId/review` | admin (`Authorization: Bearer <JWT>`) | Real as of Phase 19 - the transition endpoint Phase 17 deferred (`src/incidents/incidents.py`'s module docstring point 6). `{ newStatus, note? }` -> the updated incident. Valid transitions: `OPEN` -> any of `UNDER_REVIEW`/`CONFIRMED`/`DISMISSED`/`ESCALATED`; `UNDER_REVIEW` -> `CONFIRMED`/`DISMISSED`/`ESCALATED` (not back to `OPEN`); `CONFIRMED`/`DISMISSED`/`ESCALATED` are terminal. Each transition appends `{ action, by, note, at }` to the incident's `reviewHistory` (mirrors `loan-platform-a`'s `ReviewEventSchema`) - `by` is the authenticated admin's email. 401 if not an admin, 400 if `newStatus` isn't a real review status, 404 if no incident with that id exists, 409 if the transition isn't valid from the incident's current `reviewStatus`, 503 if the central database is unreachable. |
| GET | `/api/v1/fraud-notifications` | admin (`Authorization: Bearer <JWT>`) | Real as of Phase 18. Every persisted notification, newest first, optionally bounded by `?limit=` (default 100). `{ notificationCount, notifications: [ {same shape as the POST /applications/process endpoint's `notifications` entries, now also including `acknowledged`/`acknowledgedBy`/`acknowledgedNote`/`acknowledgedAt` - Phase 19}, ... ] }`. 401 if the token is missing/invalid/expired, 503 if the central database is unreachable. |
| GET | `/api/v1/fraud-notifications/:notificationId` | admin (`Authorization: Bearer <JWT>`) | Real as of Phase 18. One notification by its `notificationId`. 401 if the token is missing/invalid/expired, 404 if no notification with that id exists, 503 if the central database is unreachable. |
| POST | `/api/v1/fraud-notifications/:notificationId/review` | admin (`Authorization: Bearer <JWT>`) | Real as of Phase 19 - the endpoint Phase 18 deferred, built as a one-way not-acknowledged -> acknowledged transition (deliberately simpler than the incident review state machine above - see `src/notifications/notifications.py`'s module docstring point 4 for why). `{ note? }` -> the updated notification. 401 if not an admin, 404 if no notification with that id exists, 409 if it's already acknowledged, 503 if the central database is unreachable. |
| GET | `/api/v1/graph/application/:applicationId` | admin (`Authorization: Bearer <JWT>`) | Real as of Phase 19. The bounded k-hop neighborhood around one application node (`src/graph/graph_store.py`'s `get_relevant_subgraph()`, real since Phase 11 - no new traversal logic, only this read API over it). Optional `?hops=` / `?maxNodes=` query params (clamped to `[1,10]`/`[1,2000]` at the API layer - see `src/api/graph.py`'s module docstring). Same response shape `get_relevant_subgraph()` itself returns: `{ root, hops, nodeCount, edgeCount, truncated, nodes: [...], edges: [...] }`. 401 if not an admin, 404 if no graph node exists for that application. |
| GET | `/api/v1/graph/entity/:nodeKey` | admin (`Authorization: Bearer <JWT>`) | Real as of Phase 19. Same as the endpoint above, rooted at any other node type instead - requires `?nodeType=` (one of `platform`/`application`/`borrower`/`name_token`/`email_token`/`phone_token`/`address_token`/`document_hash`). 401 if not an admin, 400 if `nodeType` is missing or not one of those, 404 if no graph node exists for that `(nodeType, nodeKey)` pair. |
| GET | `/api/v1/metrics` | admin (`Authorization: Bearer <JWT>`) | Real as of Phase 19. Operational counts only (event/incident/notification counts by status, graph node/edge counts) - explicitly NOT performance/latency data, which is Phase 23's concern (see `src/api/metrics.py`'s module docstring for the full scope boundary). `{ events: {total, byStatus}, incidents: {total, byReviewStatus}, notifications: {total, byStatus, acknowledged, unacknowledged}, graph: {nodeCount, edgeCount} }`. Every count is a real, live `GROUP BY`/`COUNT` query - no caching, no pre-aggregation. 401 if not an admin, 503 if the central database is unreachable. |

### Dashboard (`central-fraud-intelligence`, `/dashboard/*`) — Phase 19, extended Phase 28

Not JSON - a server-rendered Jinja2 admin dashboard (`src/web/`), the
central app's first-ever HTML surface (see that package's module docstring
for the framework decision over a React/Vite SPA). Auth is a cookie
(`central_admin_token`) carrying the exact same JWT the JSON API's
`Authorization: Bearer` header does - the same admin account logs into
both. Every action is a plain HTML `<form>` POST (no client-side JS at
all), each one a thin wrapper over the JSON-API-backing repositories
above - no new business logic.

| Method | Path | Description |
|---|---|---|
| GET | `/dashboard/` | Redirects to `/dashboard/overview` if logged in, else `/dashboard/login`. Changed in Phase 28 - used to jump straight to `/dashboard/incidents`. |
| GET, POST | `/dashboard/login` | Login form / form submission. Sets the `central_admin_token` HttpOnly cookie on success, redirects to `/dashboard/overview`. |
| POST | `/dashboard/logout` | Clears the cookie, redirects to `/dashboard/login`. |
| GET | `/dashboard/overview` | **New in Phase 28.** The dashboard's landing page: stat cards for applications processed/cleared/fraud-detected/still-processing, incidents, notifications, and graph size (the same live counts `GET /api/v1/metrics` computes), a "how central fraud intelligence decides" explainer showing the real GNN/rule weights and threshold, and a "recently processed applications" table linking into the page below. |
| GET | `/dashboard/applications/:applicationId` | **New in Phase 28.** "The working of the GNN" for one real application: every event received for it, each rendered as a scoring-pipeline visualization (GNN score bar, rule score bar with its triggered rules, combined final-risk-score bar against the fraud-detected threshold) plus the human-readable explanation - the exact same `generate_explanation()` output `GET /api/v1/applications/:id` already returns, rendered instead of raw JSON. |
| GET | `/dashboard/incidents` | Lists incidents (mirrors `GET /api/v1/incidents`). |
| GET | `/dashboard/incidents/:incidentId` | Incident detail: full fields, review-history table, a review `<form>` offering only the valid next transitions for its current `reviewStatus`, and the notifications sent for it. |
| POST | `/dashboard/incidents/:incidentId/review` | Submits the review form; redirects back to the detail page. |
| GET | `/dashboard/notifications` | Lists notifications (mirrors `GET /api/v1/fraud-notifications`) with an "Acknowledge" button per unacknowledged row. |
| POST | `/dashboard/notifications/:notificationId/review` | Acknowledges one notification; redirects back to the list. |
| GET | `/dashboard/graph/application/:applicationId` | Renders the application's bounded neighborhood (mirrors `GET /api/v1/graph/application/:id`) as node/edge tables - deliberately not a canvas/SVG visualization (see `src/web/dashboard.py`'s module docstring for that stated scope limit). |
| GET | `/dashboard/graph/entity/:nodeKey?nodeType=` | Same, rooted at any other node type. |

Every page but `/dashboard/login` redirects (303) to `/dashboard/login`
when the cookie is missing/invalid/expired, rather than a JSON 401 - the
right behavior for a browsed page, unlike the JSON API endpoints above.
