# The first working version

**Goal:** prove that shopping amounts and purchases are saved correctly on one phone, even without internet or after closing the app.

**Status:** this is a build plan, not a completed app. It uses clearly marked test accounts. Real sign-in and sharing between phones come later.

## What you will be able to try

- Add a saved product to a shopping list.
- Set an amount using a whole-number or decimal unit.
- Buy the remaining amount or enter an amount manually.
- Undo a purchase.
- Close and reopen the app without losing saved work.
- See purchased items in a collapsed section at the bottom.
- Tell that a change is waiting to sync, without a false “shared successfully” message.

## Four pieces of work

| Step | What we build | How we know it works |
| --- | --- | --- |
| [1 — Quantities](https://github.com/Bazuzu77/shopping-list/issues/1) | App setup and reliable number entry | Decimal commas and dots work; whole-unit fractions are rejected; rounding follows the rules |
| [2 — Purchases](https://github.com/Bazuzu77/shopping-list/issues/2) | Requested, bought and remaining amounts | Independent purchases add; retries do not; undo preserves the requested amount; new needs do not inherit old purchases |
| [3 — Saved offline work](https://github.com/Bazuzu77/shopping-list/issues/3) | Save the visible change and its reminder to sync together | Crash and restart checks on a real phone database show both saved or neither saved |
| [4 — Shopping screen](https://github.com/Bazuzu77/shopping-list/issues/4) | A simple development screen using those rules | A full add, buy, restart and undo example works; test data is unavailable in release mode |

## A practical acceptance check

1. Add 2 litres of milk.
2. Record a purchase of 0.5 litres.
3. Restart the app: 1.5 litres should remain.
4. Buy the remaining amount, then add a fresh milk request.
5. Undo a purchase from the old request: the new request must be unchanged.
6. Try a fraction of a carton: the app must reject it.
7. Enter 0.0004 litres: the app must ask for a larger amount or smaller unit rather than save zero.

Also switch between two test accounts and confirm their data and preferences do not leak into each other.

## What this first version does not prove

It does not yet provide real accounts, household invitations, shared synchronization, conflict resolution, push alerts, similarity suggestions, deletion/restoration, custom-unit management, shared notes or release readiness.

Tests of fake storage do not prove that the phone's actual storage survives a crash. Record which Android/iOS devices or simulators were really checked.

## What comes afterwards

Real sign-in and protected local access; the shared server; catching up between phones; the remaining household and shopping features; notifications; then release preparation.

The [product rules](../../specs/product-behavior.md) and [offline design](../../architecture/sync-data-design.md) remain the requirements. This plan does not select paid services or authorize deployment.

## Engineering reference

The exact files, function names, tests and execution steps are preserved below for the implementer.

<details>
<summary>Open the implementation checklist and technical details</summary>

# Foundation and Offline Core Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans for native execution, or superpowers:subagent-driven-development if the user selects delegation. Steps use checkbox syntax for tracking.

**Goal:** Build a testable development-only mobile slice that durably records shopping needs, purchases and reversals offline without confusing old purchases with new demand.

**Architecture:** Pure TypeScript domain functions sit behind an account-scoped SQLite store. Each local command atomically stores its operation and optimistic projection; committed state remains separate for later server integration. A development harness supplies a clearly labeled fixture identity; it is not production authentication or guest onboarding.

**Tech Stack:** Expo/React Native, Expo Router, TypeScript, expo-sqlite; Vitest for pure domain tests. Verify compatible supported versions against official documentation at execution, then pin the selected set and lockfile. Go/PostgreSQL remain the later server baseline.

**Spec:** [approved product behavior](../../specs/product-behavior.md), [sync design](../../architecture/sync-data-design.md), [roadmap](../../../PLAN.md).

**Status:** Ready for implementation review; no code has been created. Design approval does not approve deployment or paid services.

## Global constraints

- “Independent purchases: sum effective quantities per unique purchase ID.”
- “A reversal terminally cancels that purchase for counting; replayed reversals are harmless.”
- “A later informed edit supersedes heads it observes.”
- “Keep one local account namespace/database and separate credentials in OS-backed storage.”
- Decimal units support three decimal places; whole-number units and omitted units require integers.
- Purchase defaults to the remaining amount, with explicit manual amount entry supported.
- New demand after observed completion never inherits old purchases.
- No production guest mode, simulated successful sync, nearby-sync button, cloud changes or signing setup.
- Tests use synthetic accounts only. Production account access stays disabled until real authentication, revocation and protected storage are implemented.
- The proposed 1,000,000-unit bound is an engineering validation default for this slice, not a confirmed product promise; define it once for later review.

## Review focus

1. Decimal comma, rounding ties and tiny positive input must produce exact results or the zero-rounding prompt (Task 1).
2. A crash between outbox and projection writes must leave both committed or neither (Task 3).
3. A retry after an unknown result must not duplicate a purchase (Tasks 2–3).
4. A completed need followed by a new need must not inherit historical purchases (Task 2).
5. Switching fixture accounts must never expose the other account's rows or queued work (Tasks 3–4).

## Scope boundary

Deliver one local list with catalog reuse, explicit units, quantities, assignment to the fixture member, purchase/manual purchase, reversal and a collapsed Purchased section. Keep stable household/list/product/demand/purchase identities even though the harness creates only one list.

Defer network transport, Go service, production identity, invitation/membership flows, push, similarity search, multi-device conflict resolution, merging, deletion/restoration, custom-unit administration, shared notes, backup erasure and release packaging to separate plans. Do not expose controls for deferred behaviors. This slice proves local durability and accounting; it is not a shippable application or evidence that full sync works.

Preserve operation IDs, dependency IDs, field heads and generation context now. Do not implement causal compaction in this slice: retain synthetic operation history. A later synchronization plan must specify common-base retention, purge-safe metadata and reset/rebase fixtures before enabling retention or real user data.

## File map

All paths below are relative to the workspace root.

| Files | Responsibility |
| --- | --- |
| `apps/mobile/package.json`, `package-lock.json`, `app.config.ts`, `tsconfig.json`, `vitest.config.ts` | Pinned mobile/test setup; native build capability |
| `apps/mobile/src/domain/types.ts`, `quantity.ts`, `quantity.test.ts` | IDs, exact quantities and input validation |
| `apps/mobile/src/domain/commands.ts`, `reduce.ts`, `reduce.test.ts` | Local commands, operation identity and accounting |
| `apps/mobile/src/storage/migrations.ts`, `database.ts`, `store.ts` | Account-scoped schema and atomic submission |
| `apps/mobile/src/dev/storage-checks.ts`, `fixture.ts` | On-device real SQLite checks and synthetic identity |
| `apps/mobile/app/_layout.tsx`, `index.tsx`, `dev/storage-checks.tsx` | Development route gate and small shopping interface |
| `apps/mobile/src/components/QuantityInput.tsx`, `DemandRow.tsx` | Shared amount input and list row |
| `docs/development/offline-core.md` | Reproduction commands and verified device evidence |

## Task 1: Exact quantities and executable mobile test setup

**Interfaces:** `Milli = bigint`; `UnitMode = 'integer' | 'decimal'`. `parseQuantity(text: string, mode: UnitMode): QuantityResult`, where the discriminated result is `{kind:'value', milli:Milli}`, `{kind:'rounds_to_zero'}` or `{kind:'invalid', reason:string}`. `formatQuantity(milli: Milli): string` emits a canonical decimal string. Storage and JSON use strings, never JSON bigint or floating-point arithmetic.

- [ ] Inspect applicable AGENTS.md, Git ownership/status and available tooling before changes. Use an isolated branch/worktree at execution. Do not repair ownership with broad global trust or overwrite existing files. Verify official Expo/React/React Native compatibility and record exact versions; install only what the selected milestone requires.
- [ ] Create the minimal package/config files and test script `test:unit` so tests execute in this package. Keep native configuration compatible with later encrypted SQLite support; encryption itself is not claimed by this fixture build.
- [ ] Add `quantity.test.ts` assertions: decimal `1,2345` becomes `1235n`; `0.0004` returns `rounds_to_zero`; integer `1.5` is invalid; `2` becomes `2000n`; blank, zero, negative, mixed separators, exponent notation and nonnumeric strings are invalid; `formatQuantity(1235n)` is `1.235`; exceeding the central bound is invalid.
- [ ] Run `npm --prefix apps/mobile run test:unit -- quantity.test.ts`; expect assertion/import failure for missing quantity implementation, not a tooling failure.
- [ ] Implement `types.ts` and `quantity.ts` using digit-string parsing and half-up rounding. Reject fractional whole-unit input before rounding. Export `MAX_INDIVIDUAL_MILLI = 1000000000n` from this module only.
- [ ] Rerun the same tests; require all passing, then `npm --prefix apps/mobile exec -- tsc --noEmit` with the package directory as working directory if npm does not resolve the intended tsconfig. Record the exact successful command and checkpoint the task in Git.

## Task 2: Local commands and purchase accounting

**Interfaces:** `LocalState` contains maps of products, demands, purchases and applied operation IDs. Demand fields include `id`, `listId`, `productId`, `unitId`, `unitMode`, `requestedMilli`, `assigneeId`, and `observedCompletedDemandIds`. Purchase fields include `id`, `demandId`, `quantityMilli`, unit/name snapshots and `reversed`. All IDs are UUID strings.

`LocalCommand` is a discriminated union: `demand.add`, `demand.quantity.set`, `purchase.add`, `purchase.reverse`. `LocalOperation` wraps the command with operation/account/household/member-grant/device IDs, device sequence, nullable initial checkpoint, dependency IDs and observed field-head IDs. It is a local contract; a later server contract must validate it before upload.

`reduce(state: LocalState, operation: LocalOperation): LocalState` is pure and throws a typed `DomainError` for invalid intent. `remaining(state: LocalState, demandId: string): Milli` returns `max(requested - effective purchases, 0)`. `effectivePurchased(state, demandId): Milli` includes each unreversed purchase exactly once.

- [ ] Add reducer tests: demand 5000, distinct purchases 2000 and 3000 yields zero remaining; replaying the same operation changes nothing; reversing the 2000 purchase yields 2000 remaining; increasing demand to 7000 then reversing preserves requested 7000; a new demand ID for the same product starts with no purchases; historical snapshots remain unchanged after catalog label changes in the input state.
- [ ] Add validation tests: purchase must match its demand unit; no default purchase when remaining is zero; manual overpurchase stays recorded; reversal of unknown purchase fails for local submission; reused operation or purchase ID with a differing payload fails. Preserve a canonical payload fingerprint with applied IDs so replay mismatch is distinguishable from retry.
- [ ] Run `npm --prefix apps/mobile run test:unit -- reduce.test.ts`; verify expected missing-behavior failures.
- [ ] Implement `commands.ts` and `reduce.ts`. Inject UUID generation and clock at command creation, never inside the reducer. Reject cross-household references. Demand add reuses an explicitly selected active demand when present; a deliberately fresh demand requires a fresh ID and observed completion context. Do not implement speculative remote merging.
- [ ] Rerun both domain suites, typecheck and checkpoint the task. Tests must verify immutable inputs and explicit purchase identity, not merely rendered totals.

## Task 3: Atomic account-scoped SQLite storage

**Interfaces:** `openAccountStore(accountId: string): Promise<LocalStore>`. `LocalStore.submit(operation: LocalOperation): Promise<LocalState>`, `read(): Promise<LocalState>`, `pending(): Promise<LocalOperation[]>`, `close(): Promise<void>`. A store is bound to its account; submission for another account rejects. Test-only `FaultPoint = 'after_outbox' | 'after_projection'` is injectable into the storage adapter, not a production user option.

Schema: versioned migrations; `outbox(operation_id PRIMARY KEY, fingerprint, payload, device_sequence, status)`; `committed_state`; `optimistic_state`; `metadata` for account binding and sequence. Encode bigint recursively through explicit domain serialization. The fixture has no authoritative feed, so submitted operations stay pending and committed state contains only fixture bootstrap data. Never mark pending work remotely accepted.

- [ ] Write `storage-checks.ts` to exercise actual Expo SQLite: submit then reopen preserves projection/outbox; injected failures at both fault points roll back both writes; identical retry returns existing state with one outbox row; differing payload rejects; separate account databases stay isolated; migration rerun preserves existing data; sequence increments roll back on failure.
- [ ] Expose the checks only in a development route. Run on an available iOS/Android simulator or device; initial checks must fail due to absent storage behavior. If no native runtime is available, record that blocker explicitly; a mock database is not evidence of durability.
- [ ] Implement migrations and the store using `withExclusiveTransactionAsync` and its transaction connection for all reads/writes belonging to submission. Validate fingerprint first, reduce the current optimistic state, insert the outbox record and update projection atomically. Do not invoke network or UI callbacks inside the transaction.
- [ ] Run real SQLite checks, requiring all passing. Force-stop/relaunch the app after a successful purchase and verify both row and pending operation survive. Record OS/device/runtime and results in `docs/development/offline-core.md`; checkpoint the task.

## Task 4: Development shopping flow and handoff evidence

**Interfaces:** UI calls only the Task 3 store and Task 2 domain selectors. `QuantityInput` accepts `mode: UnitMode`, `value: string`, `onChange(value: string): void`; it renders validation from Task 1. `DemandRow` receives a demand, effective purchase/remaining totals and callbacks; it performs no persistence itself.

- [ ] Add a development fixture with one synthetic account, household, member and empty Shopping list. Gate initialization and `/dev` routes behind development mode; a nondevelopment build must show a setup-unavailable screen and must not seed or load synthetic shopping data.
- [ ] Implement catalog selection/reuse, fresh demand amount default 1, fixture member assignment, whole and decimal unit inputs, amount editing, purchase remaining, explicit manual purchase and reversal. After a completed demand is deliberately added again, create a fresh demand ID with observed completion context. Disable submission while a store transaction is pending to prevent accidental double taps; deliberate later purchases remain distinct.
- [ ] Implement Purchased at the bottom, initially collapsed, with the preference stored per fixture account. Show local pending status without a successful-sync indicator. Render overpurchase separately from remaining zero. Failed writes keep entered text and show a retryable message; retries reuse the operation ID until outcome is known.
- [ ] Run a device scenario: add 2 litres milk, purchase 0.5, restart, verify 1.5 remaining, purchase remaining, add fresh milk demand, reverse an old purchase and verify the fresh demand is unaffected. Repeat count-unit validation with cartons and verify `0.0004` litres prompts instead of silently creating zero.
- [ ] Switch between two synthetic accounts, verify separate data/preferences/outboxes, and verify the release configuration cannot enter fixture routes. Run all domain tests, typecheck and on-device SQLite checks once after final edits.
- [ ] Record verified checks, unmet device coverage and remaining milestones in the development guide. Self-review against the scope boundary; checkpoint the task. No implementation-complete claim while the native persistence checks remain blocked.

## Acceptance and later milestones

Accept this slice only when the stated domain assertions and actual SQLite checks pass and the development shopping scenario survives restart. Document Android/iOS coverage honestly; do not equate one simulator with both platforms. No server, authentication, encryption, multi-device convergence or store-release readiness claim follows from this milestone.

Next plans, in dependency order: identity/provider feasibility and protected local access; Go/PostgreSQL operation acceptance and feed; client catch-up/rebase and causal conflict fixtures including D1–D7; remaining household/catalog/list features and notification delivery; deployment and release. Provider costs, backup guarantees and infrastructure access are checked when their plan makes them concrete.

## Self-review record

Scope checked against the approved design: this milestone implements only local exact quantities, purchase identity/reversal, new-demand separation and durable outbox boundaries. Deferred features are listed explicitly, not treated as satisfied. Interfaces use the same names across tasks. Each review-focus risk has assertions in its owning task. Future causal compaction is deliberately disabled rather than guessed. No application tests have been executed while writing this plan.

</details>
