# CODEX HANDOVER — Family Grocery App

**Prepared:** 2026-11-18  
**Status:** Planning and handover. No code or repository implementation is confirmed by this document.  
**Target:** Production-quality, scalable, offline-first shared grocery application for couples and families, distributed through the Apple App Store and Google Play, with a backend on DigitalOcean.

> **For Codex:** This handover consolidates the project decisions and the detailed implementation specifications discussed in ChatGPT. Treat it as product context and a design baseline, **not authorization to implement the entire application**. Start with the Phase 1 inspection and proposal only. Inspect the actual workspace and obtain human approval before implementation. Do not assume that an existing repository, service, domain, developer account, or cloud configuration is available until inspected.

## 0. Executive summary and decision ledger

The **domain should be easy**, allowing fast iterations and real household testing; the **engineering should be serious**, with a clear path toward a scalable commercial product. Avoid both tutorial-grade shortcuts and premature enterprise architecture.

### Agreed product requirements

- Shared grocery shopping lists for spouses/families, supporting multiple households and more than two members in the data model.
- A single underlying shared list with **two views**: common list (responsible-person icons) and grouped-by-responsible-person view, including unassigned items.
- Track **who added** an item, **who is responsible** for buying it, and **who actually purchased** it as separate concepts.
- Preserve **two independent purchase operations** if two members buy the same item; show two purchase indicators. The user suggested an underline/other marking; final visualization is explicitly undecided.
- Reliable offline editing and synchronization through the backend; a future **Sync Nearby** in-app button for direct offline phone-to-phone transfer.
- Early iOS App Store and Android distribution, not only a browser-based prototype.
- Deploy backend to the user's **existing DigitalOcean** infrastructure after inspecting suitability.

### Selected technical baseline (previously proposed and accepted for planning; verify before implementation)

| Layer | Baseline | Status |
|---|---|---|
| Mobile | React Native, Expo, TypeScript, Expo Router | Selected planning baseline |
| Local persistence | SQLite via `expo-sqlite` | Selected planning baseline |
| Backend | Go, `net/http`, `pgx`, `sqlc`, REST/OpenAPI | Selected planning baseline; user asked why Go rather than Spring Boot |
| Server persistence | PostgreSQL 16, versioned SQL migrations | Selected planning baseline |
| Runtime architecture | Modular monolith, stateless API instances | Agreed principle |
| Cloud | DigitalOcean Droplet, Docker/Compose, Caddy, HTTPS | Selected planning baseline; actual server must be inspected |
| Automation | GitHub Actions, Expo EAS Build/Submit | Selected planning baseline |
| Cloud sync | Durable client operations and server-side idempotency | Required; detailed conflict semantics **not yet approved** |
| Live updates | SSE, WebSocket, or simpler invalidation/polling | **Open decision**, do not assume WebSocket is mandatory |
| Identity | Email-based sign-in + anonymous local use | Proposal to review, not a locked implementation |
| Nearby sync | User-triggered, direct peer-to-peer transport | **Deferred**; preserve transport independence |

### Explicitly deferred

Payments/subscriptions, ads, AI recommendations, recipes, meal planning, grocery price comparison, barcode scanning, advanced analytics, native nearby-device discovery/pairing, UWB/NameDrop, Kafka, Redis, Kubernetes, microservices, multi-instance deployment.

### Decisions requiring human review

1. Authentication provider/flow; anonymous local data ownership and migration into accounts.
2. Synchronization protocol and formal conflict rules for edits, assignments, deletes, reversals, and operations arriving out of order. **Do not invent these silently.**
3. Treatment of multiple purchases of one item: quantity, completion state, reversal, and UI indication.
4. Real-time notification transport (SSE, WebSocket, or short polling); notifications are not authoritative state.
5. Mobile state layer and migration library; pick minimal supported dependencies.
6. Server migrations tool, backup tooling, release infrastructure, domain and hosting isolation.
7. Whether a working anonymous/offline mode meets the desired first App Store release scope and how invitations are activated after signup.

---

# SPEC-00 — Product and Functional Requirements

## Purpose

A lightweight household grocery coordinator. People can add products, assign buyers, shop independently, keep track of who did what, and reconcile changes despite intermittent connectivity. User testing is expected to happen with family and friends; user acquisition and commercial viability remain hypotheses, not established facts.

## First-release features

### Households and collaboration

- Create a household, see members, invite another member through a secure, expiring, revocable invitation.
- Support any reasonable number of members in the model, even if initial UI focuses on two.
- Enforce household boundaries on the server.
- A user can work with a private list without registration; authentication is required to share via cloud (pending identity-flow approval).

### Grocery lists and items

- Create/rename shopping lists; add, edit, delete, purchase, and restore items.
- Optional quantities and units; basic fast-entry interaction.
- Record item creator separately from responsibility and purchase actor.
- Display a single underlying list through common/grouped views. **Do not create or synchronize two separate copies based on responsible person.**

### Responsibility assignment and UI

- `assigned_to` is a nullable household-member identity; can change over time.
- **Common view:** all products, responsible-person icons/labels.
- **Grouped view:** separate sections for each responsible person and an unassigned section; number of sections scales with household members.
- Default assignment to item creator is a **proposal**, not a confirmed product requirement.

### Purchases and attribution

- Model distinct purchase actions, not a lone `purchased` Boolean.
- Distinguish two genuinely independent purchases from network retries of a single purchase.
- Track who performed each purchase, regardless of who was assigned.
- A product purchased independently by both spouses can display **two purchase indicators** plus some duplicate-marking treatment. Whether to underline, strike through, show a warning, and how quantity affects behavior are **left for later UI/product decision**.
- Support purchase reversal without silently deleting historical facts.
- “Signed operation” means **attributed to an authenticated actor and device** in the current specification, not necessarily a cryptographic signature. Digital signatures are not approved as a first-release requirement.

### Offline behavior

- Show cached/local lists and permit local mutations while disconnected.
- Persist user changes across application termination and restarts.
- Synchronize when network access returns; do not lose or double-apply changes.
- Conflict resolution must converge under repeated delivery and different delivery order, within explicitly documented semantics.
- Show useful connectivity/synchronization state to the user; never claim remote sync succeeded before confirmed.

## Excluded from v1

Native device-to-device transfer, subscriptions, retail promotions, AI, recipes, meal planning, barcode scanning, advanced shopping insights, payments, location-based tracking.

## Acceptance scenario

Two members join the same household. A creates milk and assigns B; B sees it in both view modes, modifies a list offline, and independently records a purchase. A also records a separate purchase offline. After connectivity returns, both phones converge and both purchase actions are preserved. Re-sending one operation causes no additional purchase. Unauthorized users cannot read or write this household.

---

# SPEC-01 — Mobile Application

## What this layer does

All user-facing screens and actions on iPhone and Android. It should feel native and remain responsive with no network.

## Technology list

| Technology | Plain-language description |
|---|---|
| React Native | Mobile user interface from a shared JavaScript/TypeScript codebase |
| TypeScript | Adds type checks to make code safer and easier to maintain |
| Expo | Tooling for device integration, local development and app builds |
| Expo Router | Maps app screens and routes to navigation |
| React Native testing tools | Verify core UI interactions and presentation |

## Behavior and screens

- Household/list selector; common and grouped item views.
- Fast add item; name, optional quantity/unit, responsibility picker.
- Item editing, purchase/reversal actions, history/attribution indicators.
- Invitations, member management, account settings and account deletion.
- Error, empty, offline and pending-sync states; loading indicators must not block local use.
- Accessible touch targets, readable labels, support for dynamic font settings and screen readers.
- Use phone-local SQLite read-model/state; API responses update local data rather than driving each screen directly.
- The **Sync Nearby** control can be represented as a future affordance, but must **not be presented as working** in the first release. Do not include a dead button in production.

## Deliverable

An installable and testable native app, with the offline grocery flow demonstrable before server integration.

---

# SPEC-02 — Local Database and Synchronization Engine

## What this layer does

Makes the app offline-first: users write locally; an outbox holds changes; the app reconciles with remote storage later. Synchronization rules must be separate from transport so future nearby transfer uses the same semantics.

## Technology list

| Technology | Plain-language description |
|---|---|
| SQLite | Relational database embedded in each phone |
| `expo-sqlite` | Expo access to device-local SQLite |
| TypeScript sync module | Applies and reconciles changes with documented conflict rules |
| HTTP client | Sends operations to server and fetches missing operations |

## Operation model (conceptual)

Each mutation has a stable globally unique `operation_id`, `entity_id`, operation type, payload, originating `device_id`, local creation metadata and synchronization state. Cloud operations also carry household scope and authenticated actor identity; the server establishes actor identity from credentials and membership, **never by trusting a client-supplied user ID**. Proposed types:

- `ITEM_ADDED`
- `ITEM_UPDATED`
- `ITEM_REMOVED`
- `ITEM_ASSIGNED`
- `ITEM_UNASSIGNED`
- `ITEM_PURCHASED`
- `PURCHASE_REVERTED`

The exact envelope, causal/version metadata, schema and merge algorithm are **not approved**. Specify them before coding the synchronization implementation.

## Required algorithmic behavior

1. Atomically commit a local state change and pending operation so process termination does not lose it.
2. When connected, send unsent operations with their existing IDs.
3. The server authenticates, authorizes and validates operations and applies each ID once.
4. Retrieve accepted/missing remote operations (or an equivalent proven sync protocol) using durable checkpoints.
5. Apply them transactionally to the local representation in a deterministic way.
6. Safely acknowledge synced operations only after server acceptance; handle timeouts and retries.
7. Recover missed notifications by fetching authoritative changes.

### Conflict/test matrix — must be resolved first

- Different independent additions, including similar product names.
- Same-operation retry versus two distinct purchases.
- Two concurrent edits to one product.
- Concurrent assignment to different members.
- Delete/edit, delete/purchase, and restore/delete conflicts.
- Reversal referencing an unavailable or not-yet-received purchase.
- Events received out of order, from clocks with different times.
- User account linking after anonymous local edits.
- Multiple households and loss of membership while offline.
- Sync interrupted by app termination / network change.

**Do not infer that operation IDs alone make the system conflict-free. Do not rely only on client wall-clock timestamps.** Choose and document deterministic semantics and prove convergence with tests before implementing full sync.

## Future-proofing

The engine consumes/emits operations without assuming HTTP. Nearby peer-to-peer transport is deferred and must not be a hidden dependency of first release.

---

# SPEC-03 — Go Backend API

## What this layer does

Coordinates accounts/households, verifies permissions, accepts operations, serves changes and keeps authoritative server records.

## Technology list

| Technology | Plain-language description |
|---|---|
| Go | Compiled language for a lightweight long-running API server |
| `net/http` | Built-in HTTP request handling |
| `pgx` | PostgreSQL driver for Go |
| `sqlc` | Turns explicitly written SQL queries into typed Go code |
| OpenAPI | Machine-readable definition of HTTP endpoints and payloads |
| Structured logging | Consistent service logs for diagnostics |

## Modules

Identity/authentication; households/membership/invitations; lists/items; operation intake and conflict resolution; synchronization; health/observability. Keep modules separated within **one deployable modular monolith**.

## Preliminary endpoints (not final API contract)

```text
POST /v1/households
POST /v1/households/{id}/invites
GET  /v1/households/{id}/members
GET  /v1/lists
POST /v1/lists
POST /v1/sync/operations
GET  /v1/sync/changes
GET  /health
```

Authentication and account-deletion routes to be designed separately. Since sync operations are the durable mutation protocol, avoid contradictory duplicate update paths unless explicitly justified.

## Service requirements

- Validate every mutation and membership server-side; no household ID alone grants access.
- Bind actor identity from authenticated context. Invitations must be random, expiring, revocable and scoped.
- Apply operations in database transactions with idempotency constraints and durable event order/checkpoints.
- Stateless process: do not store authoritative sessions or business state only in RAM.
- Bounded connection pools, request timeouts, size limits, rate limiting; parameterized SQL.
- Explicit API versioning, health checks, structured logs.
- Real-time transport is a later **approved-design decision**, not a substitute for a durable catch-up API.

## Deliverable

A tested Go service capable of servicing many independent households and later running multiple replicas behind a load balancer.

---

# SPEC-04 — PostgreSQL Database

## What this layer does

Stores shared state and the durable history/reconciliation metadata needed for reliable synchronization.

## Technology list

| Technology | Plain-language description |
|---|---|
| PostgreSQL 16 | Main relational database on DigitalOcean |
| SQL migrations | Version-controlled changes to tables, constraints and indexes |
| `pgx` + `sqlc` | Typed SQL access from Go |

## Proposed entities

`users`, `devices`, `households`, `household_members`, `household_invitations`, `shopping_lists`, `shopping_items`, `operations`, `sync_checkpoints`, and possibly `purchase_events` (depending on whether purchase records are projection tables derived from the operation log). These are **conceptual** entities, not instructions to create one table per term.

## Invariants and constraints

- Globally stable IDs generated before sync where needed.
- Referential integrity and unique operation IDs to support idempotency.
- Indexed household-scoped queries and monotonic server change sequencing/checkpoints.
- Store `created_by`, `assigned_to`, and purchaser attribution separately.
- Avoid exposing unrelated households through queries or change feeds.
- Transactional operation persistence and state projection.
- Versioned migrations, reversible deployment strategy where practical.
- Explicit retention, tombstone and deletion policy compatible with slow/offline devices and privacy obligations.
- Backups outside the server and a tested restore process.

## Deliverable

Schema, migrations, SQL queries and PostgreSQL-backed integration tests.

---

# SPEC-05 — Cloud Infrastructure (DigitalOcean)

## What this layer does

Runs and protects the production API, database and deployment pipeline.

## Technology list

| Technology | Plain-language description |
|---|---|
| DigitalOcean Droplet | Existing Linux virtual server; inspect before reuse |
| Ubuntu 24.04 | Previously reported server OS; confirm actual environment |
| Docker | Packages the backend service and its dependencies |
| Docker Compose | Starts services in a predictable configuration |
| Caddy | HTTPS certificates and reverse proxy to the Go API |
| PostgreSQL 16 | Existing reported database version; check isolation and resources |
| Off-server backups | Recovery when the Droplet/storage is lost |

## Requirements

- Inventory server, disk/RAM/CPU, existing processes/databases, firewall, access, backups and domain before changes.
- Do not interfere with other services, notably existing career tooling possibly hosted on the same infrastructure.
- Separate database/schema/credentials for this app; restrict database network access.
- HTTPS public API, health checks, logs, least-privilege secrets/configuration.
- Versioned container images, deployment rollback plan, database backup/restore drill.
- No Kubernetes or managed DB required initially.
- Recognize that a single Droplet is **not high availability**; horizontal scalability is a design property, not current redundancy.

## Deliverable

Documented production deployment, recoverability and repeatable releases after explicit infrastructure approval.

---

# SPEC-06 — Authentication, Privacy and Security

## What this layer does

Ensures only authorized family members can access shared lists, while keeping friction low for local-only use.

## Candidate technology

| Technology | Plain-language description |
|---|---|
| Email-based login | Proposed simple account method without social login |
| Access/refresh credentials | Sessions that expire and can be revoked |
| Expo SecureStore | OS-backed storage for sensitive mobile credentials |
| HTTPS/TLS | Encrypted mobile-to-server traffic |
| Backend authorization | Checks household membership for every operation |

## Requirements

- Anonymous/private on-device use followed by controlled account attachment is proposed; **review identity migration design before implementation**.
- Cloud sharing requires securely authenticated user membership.
- Secure invitations, token expiry/revocation, rate limits and audit-relevant attribution.
- Clear privacy policy and transparent data collection/storage/deletion.
- In-app account deletion and publicly accessible deletion instructions/flow as required by stores.
- Data-deletion behavior must address household shared records and legal/privacy obligations: deleting one user's account must not silently corrupt another member's purchase history or reveal identities unnecessarily.
- No arbitrary client-supplied `actor_id` is proof of identity; a `device_id` is not an authentication credential.
- No secrets or service credentials checked into Git or bundled in client code.

## Deliverable

Working signup/signin, household access, secure local tokens, account deletion and security tests.

---

# SPEC-07 — Testing, CI/CD and Observability

## What this layer does

Keeps frequent releases safe and gives enough visibility to diagnose failures.

## Technology list

| Technology | Plain-language description |
|---|---|
| GitHub | Source control and code review |
| GitHub Actions | Automated lint/test/build/deploy workflows |
| Go test | Backend unit and behavior tests |
| PostgreSQL integration tests | Validate actual SQL/transaction behavior, not just mocks |
| React Native test tooling | UI and local data-flow tests |
| Expo EAS Build/Submit | Create and deliver native store builds |
| Logs, health checks, metrics | Basic production troubleshooting |

## Testing priorities

- Authorization and isolation of households.
- Product addition, editing, deletion, assignment and views.
- Two independent purchase events versus operation replay.
- Full sync conflict matrix from SPEC-02; test order-independence where promised.
- Local persistence after restart, interrupted upload/downloading and retry.
- Real PostgreSQL migrations, transaction boundaries and query/index behavior.
- Account deletion and invitation security.
- App flows on real iOS and Android devices before publication.

## CI/CD outline

On pull request: formatting/linting, type checking, backend/mobile tests, contract/schema checks. On approved release: build versioned Go image, deploy to DigitalOcean, confirm health, have rollback available; independently build/sign mobile binaries and submit via Expo EAS after appropriate review.

**Never claim a test ran unless it was run and its result was inspected.**

---

# SPEC-08 — App Store and Google Play Release

## What this layer does

Makes the app eligible for actual testing and public distribution, rather than a WebView/PWA-only prototype.

## Technology/services

- Expo EAS Build and EAS Submit.
- Apple Developer Program, App Store Connect, TestFlight.
- Google Play Console, testing tracks, production release.

## First-release requirements

- Real, functioning native app; not a minimal wrapper around a website.
- Choose product name, bundle IDs and versioning early.
- App icon, screenshots, plain-language store descriptions, support contact.
- Accessible privacy policy, appropriate data disclosures and permissions.
- In-app account deletion if account creation exists, plus any required public deletion channel.
- App review credentials/sample household or accessible demo mode; reviewers must be able to exercise shared behavior.
- Keep SDK/runtime versions aligned with the **current** platform submission rules.
- Conduct TestFlight and Google Play testing, and check whether the developer account falls under Google's closed-test requirements.
- Verify official policy sources at time of submission: rules and minimum SDKs can change.

## Deliverable

Real testing builds and store submissions, with approval-dependent publication.

---

# SPEC-09 — Deferred Nearby Offline Synchronization

## Future user journey

Both spouses open the app without internet. One presses **Sync Nearby**; the app discovers an authorized nearby household device, establishes a direct link and exchanges missing changes. This does **not** require touching phone tops, NameDrop, or UWB.

## Possible technologies (research before implementation)

- iOS supported peer discovery/network APIs and a small Swift native module exposed to Expo/React Native.
- A separate Android nearby transport as needed.
- Reuse the same transport-agnostic operation/synchronization semantics as cloud sync.

## Constraints

- Do not depend on intercepting Apple's system-controlled NameDrop gesture.
- Check current framework support, device/OS constraints, permission requirements, Expo native build setup and background limitations against official documentation when this feature is scheduled.
- Respect household authorization and pairing trust; peer discovery alone does not authorize syncing private data.
- **Not in first public release**. No fake working button in production.

---

# Development roadmap and exit criteria

## Phase 1 — Foundation (Codex's immediate assignment: proposal only)

- Inspect actual workspace and existing files, tools and Git state.
- Review compatibility/versions of Expo, React Native, Go, PostgreSQL and selected build tools against current official documentation.
- Propose minimal monorepo structure, docs and architectural decision records.
- Outline local development workflow and tests.
- List unresolved high-impact decisions requiring explicit approval.
- **Stop after presenting the plan. Do not scaffold or deploy until approval.**

**Expected layout (proposal, not mandatory):**

```text
family-grocery/
├── apps/
│   └── mobile/
├── services/
│   └── api/
├── db/
│   └── migrations/
├── docs/
│   ├── architecture/
│   ├── specs/
│   └── decisions/
├── infra/
│   └── docker/
├── .github/
│   └── workflows/
└── README.md
```

Suggested docs after approval: `README.md`, `docs/architecture/overview.md`, `docs/specs/product.md`, `docs/specs/mobile.md`, `docs/specs/sync.md`, `docs/specs/api.md`, `docs/specs/database.md`, `docs/specs/infrastructure.md`, `docs/specs/security.md`, `docs/specs/release.md` and short ADRs in `docs/decisions/`. Avoid empty boilerplate documents.

## Phase 2 — Offline mobile MVP

Implement SQLite-backed grocery lists, item mutations, responsibility assignment, common/grouped UI, purchase events, local persistence. Use **clearly labeled demo identities** to develop attribution without claiming secure user authentication.

**Exit:** One phone works offline, survives restarts and correctly displays assignments and independent purchases.

## Phase 3 — Identity and shared households

Implement registration/login, invitation/membership, household authorization, account deletion, and anonymous-to-account data migration if approved.

**Exit:** Two authenticated devices join a household securely.

## Phase 4 — Cloud synchronization

Approve protocol/consistency model **first**, then implement durable operations, upload/download, server idempotency, conflict rules, checkpoints, background/reconnect triggering and, if needed, change notifications.

**Exit:** Two devices converge after offline edits; identical operation replay is harmless, independent purchases remain distinct, security and conflict tests pass.

## Phase 5 — Production infrastructure

Inspect actual DigitalOcean setup, get explicit approval, isolate services, configure HTTPS, deploy, instrument, back up and test restoring.

**Exit:** Real phones can use a stable HTTPS backend and a recovery procedure is demonstrated.

## Phase 6 — Beta and public release

Build and test native apps, complete privacy/account-deletion requirements, obtain testers, fix issues, prepare listings and submit to stores.

**Exit:** Public availability after Apple/Google approval, without inventing review outcomes.

---

# Working agreement with Codex

1. Inspect repository and existing project state before proposing edits.
2. Restate the current phase, scope and desired acceptance criteria.
3. Explain options/trade-offs in plain language; document substantive choices in ADRs.
4. Do not silently replace the selected stack or add extra infrastructure.
5. Seek approval for decisions listed as open; **do not invent conflict-resolution semantics**.
6. Implement in small reviewable increments once phase scope is approved.
7. Test using the actual commands/tooling available; report exactly what passed, failed or wasn't run.
8. Keep files/docs synchronized with verified implementation state.
9. Never claim deployment, App Store publication or cloud access without direct verification.
10. Do not advance phases or deploy without explicit user authorization.

## Immediate prompt for Codex

> Read `CODEX_HANDOVER.md` in full. **Work on Phase 1 planning only.** Inspect the current workspace (do not assume files or a Git repo exist), verify tool/library compatibility with official current documentation where accessible, propose a minimal monorepo layout and a short implementation plan, flag all unapproved high-impact decisions, and stop for approval. Do not create app code, modify DigitalOcean resources, deploy, or implement synchronization until authorized.

---

# First-release definition of done

- Two authenticated household members can use a genuinely shared grocery list.
- A private local-only list is usable without login if the selected anonymous flow is approved.
- The two responsibility views reflect **one** shared underlying data set.
- Item creator, assigned buyer and actual purchaser are distinct and correctly attributed.
- Separate purchase operations survive synchronization; retries cannot fabricate duplicates.
- Offline changes survive app restart and reconcile after reconnection under tested rules.
- Household access is enforced and accounts can be deleted.
- Backend runs safely on DigitalOcean; HTTPS, backups and restoration are verified.
- Critical API, database and synchronization paths have automated tests.
- iOS and Android builds satisfy current submission requirements; actual store release is subject to review approval.

# Caveats and provenance

- This document consolidates the detailed `SPEC-00` through `SPEC-09` and the subsequent Codex handover in the project chat. It is a **planning artifact**, not evidence of implemented features.
- Earlier conversation references to an existing Ubuntu 24.04 / PostgreSQL 16 DigitalOcean host are historical reports; verify live infrastructure before deploying.
- Prior competitor research suggested the grocery-list category is highly competitive. The proposed product's differentiation and commercial viability remain **unvalidated**.
- References to specific frameworks and policies reflect prior planning; Codex must independently verify current versions, API availability, licensing and store requirements before implementation.
