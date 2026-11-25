# Data Model and Synchronization Proposal

Date: 2026-11-25
Status: **Design direction and presented defaults approved on 2026-11-25; implementation not yet authorized.**

Approval record: the user replied “approved” to the proposal summary, including D1–D7 and the hosted-identity direction. Those outcomes now govern subsequent planning. Earlier “propose”/“needs approval” wording below records the design discussion and is superseded for those approved defaults. Numeric engineering defaults, an exact provider, paid services and deployment remain unselected. Catalog-default and late-candidate mechanisms remain technical proposals to check against the existing approved rules, not separately confirmed new behavior.

Source of product requirements: [product-behavior.md](../specs/product-behavior.md). Decision history: [PLAN.md](../../PLAN.md). This document proposes technical mechanisms, identifies the few additional product choices they need, and does not replace approved behavior silently. All numeric operational limits below are proposed engineering defaults, not user-approved promises.

## 1. Recommendation and alternatives

**Recommend an explicit operation protocol with a server-authoritative change feed**, local SQLite projection and durable outbox. Keep Go/PostgreSQL on DigitalOcean and Expo/React Native. Serialize mutations within each household initially; households remain independent. This is a modular monolith, not a distributed event platform.

| Approach | Benefit | Cost / fit |
| --- | --- | --- |
| Versioned row updates and reject every stale row | Small initial implementation | Makes unrelated offline edits conflict and cannot express the approved purchase/merge rules without rebuilding an operation model |
| Generic CRDT/local-first synchronization framework | Reuses replication primitives | Custom deletion, first-server-accepted resolutions, policy-at-discovery and permanent erasure still need a domain protocol; changes the selected stack/dependency commitments |
| Explicit operations + authoritative feed | Directly implements approved rules, inspectable retries and attribution | Requires a tested reducer and causal/conflict design; recommended |

Not every policy is commutative. First accepted names/resolutions, last accepted policy changes and policy-at-discovery intentionally depend on server acceptance order. Guarantee: **clients with the same accepted change stream and no pending edits converge**, retries have no duplicate effect, and independent purchases are never discarded. Test arbitrary operation-delivery order only against invariants that actually promise order independence.

Nearby sync remains deferred. Domain operations are transport-independent, but authoritative choices cannot become final solely because peers exchange them. A future nearby feature must reconcile provisional decisions with the server's acceptance rules.

## 2. Data boundaries

Use client-generated random UUIDs for objects and operations; UUID ordering is a stable tie-breaker, never evidence of action time. Store household ID on every shared row and use household-scoped foreign keys so references cannot point across households. Private account/session data lives outside the household feed.

| Logical record | Purpose and key relationships |
| --- | --- |
| Account | Stable app account ID; external identity subject, verified email mapping, deletion state; never use email as identity |
| Device / app session | Account-owned device identity, active/revoked session, push registrations; device ID is not authentication |
| Household | Owner membership, lifecycle status, transactional change head |
| Membership | Household + account, stable membership ID, normalized household display name, active/revoked grant generation |
| Invitation | Household/inviter, hashed random token, expiry, consumption/revocation; online-only acceptance |
| Unit | Household unit ID, name, active/retired state, integer/decimal mode, optional known dimension/scale metadata |
| Product | Household product ID, normalized name, usual unit, active/retired state; no requested or purchased quantity |
| Shopping list | Household list ID, name claim or pending-name state, active/deleted/purged state |
| Demand | List + product identity + unit + generation identity; requested amount, assignee, notes, visibility state |
| Demand contribution | Original addition identity and attribution retained through merges; does not mean additive requested quantities |
| Purchase | Stable purchase ID, originating demand, purchaser, quantity and unit/name snapshots |
| Purchase amendment | Correction or reversal of a specific purchase; original record remains attributable |
| Operation / outcome | Stable submitted intent, payload fingerprint, causal references, accepted/rejected state and result references |
| Field candidates | Causal write heads for each independently mergeable field; compacted committed base where possible |
| Alias / merge decision | Original product/unit/demand IDs to canonical identities, selected surviving defaults, causal merge context |
| Conflict / resolution | Entity/field, candidates, previous common value, bound policy, conflict revision, accepted resolution |
| Notification / delivery outbox | Durable private recipient notice, read state, optional push job; separate from authoritative data |
| Checkpoint / reset manifest | Device/account/household grant scoped feed position, snapshot boundary and purge markers |

These are logical responsibilities, not a mandate for one table per row. Migrations and exact columns belong in the implementation plan. Purchase/product snapshots preserve historical descriptions, but account deletion replaces attribution with Deleted member rather than retaining a personal-name snapshot forever.

### Quantity representation and constraints

Represent quantities in integer thousandths (`quantity_milli`, BIGINT), transmitting decimal strings at the API boundary and converting without binary floating point. Propose positive values up to 1,000,000 units per individual entry/purchase; validate aggregate overflow separately. Count units and omitted units require multiples of 1000. Decimal input rounds half-up to three fractional digits; positive input rounding to zero gets the approved prompt. Do not round fractional cartons into whole cartons: integer-only validation is separate.

Use Unicode NFC, full case folding, trim and whitespace collapse for names; preserve original spelling for display. Accent stripping, plural reduction and spelling distance belong only in suggestions, not uniqueness. Use one versioned normalization definition in TypeScript/Go conformance fixtures. Exact matching considers catalog identity and unit equivalence; a retired record's reused name does not mean the same identity.

## 3. Local write and sync contract

Every local mutation atomically writes an operation and updates the optimistic view in one SQLite transaction. Maintain committed server state separately from the optimistic overlay (separate tables or explicit layers); do not blindly replay server events onto a state that already includes pending operations.

Illustrative operation envelope:

```json
{
  "schema_version": 1,
  "operation_id": "uuid",
  "household_id": "uuid",
  "membership_grant_id": "uuid",
  "device_id": "uuid",
  "device_sequence": 42,
  "base_checkpoint": "opaque-server-issued-token",
  "depends_on": ["earlier-local-operation-uuid"],
  "target": {"type": "demand", "id": "uuid"},
  "kind": "demand.quantity.set",
  "observed_field_heads": ["operation-uuid"],
  "payload": {"quantity": "3.000"}
}
```

Client timestamps may be included for approximate history display, never conflict authority. Actor identity and session come from credentials. Validate device ownership and grant generation against server state. A rejoined member receives a new grant; old queued work cannot silently acquire new authority.

Proposed API surface:

- `POST /v1/households/{id}/sync/operations`: up to 100 operations / 256 KiB per batch; returns outcome per operation.
- `GET /v1/households/{id}/sync/changes?cursor=...`: authoritative projection changes, aliases, tombstones and conflict revisions; bounded pages.
- `GET /v1/households/{id}/sync/snapshot`: consistent reset/bootstrap state at a specified head, paged under a stable snapshot token.
- Separate account feed for private notifications, household access removal and account/session status. An unauthorized household endpoint must not expose its contents just to report removal.
- Online identity, membership and permanent-deletion commands use the same transactional household coordination/idempotency machinery, not contradictory row-update paths.

Outcome classes: `accepted`, `accepted_with_conflict`, `rejected(reason)`, `dependency_pending(missing_ids)`; transient failures are retryable, not durable rejection. Retrying an operation with the same canonical payload returns its prior outcome. Reusing an ID with a different payload is rejected. A prior success may later be superseded by an accepted concurrent operation under explicit domain rules; history and feed show that transition.

### Causal validation

The opaque checkpoint proves a contiguous server frontier was supplied to this grant; it is not a license to reference other households. `observed_field_heads` and `depends_on` represent the specific state on which the user acted. Resolve their transitive graph, validate household/entity membership and known schema, and defer unknown dependencies. A reversal received before its purchase waits; never fabricate the purchase or drop the reversal. A rejected dependency makes downstream intents blocked for revalidation, not silently retargeted. Reconnecting clients query outcomes for ambiguous in-flight operations before replacing anything.

Two writes are concurrent when neither causally observes the other. A later informed edit supersedes heads it observes. Updating quantity must not implicitly overwrite an unobserved assignment or note.

### One household transaction

1. Check authenticated account/session, active grant and resource limits. Lock the household row; authorization is rechecked inside the transaction.
2. Resolve idempotency, aliases and dependencies; validate current purge/retirement/name/unit constraints.
3. Persist accepted intent, update causal candidates, run field reducers, lifecycle precedence and conflict discovery.
4. Persist projections, conflict revisions, private notices and delivery jobs atomically.
5. Increment the locked household head transactionally and write the change record at that sequence; commit before acknowledgment.

Do not use `MAX(sequence)` over an independently allocated PostgreSQL sequence as a safe checkpoint: transactions can commit out of allocation order. The locked transactional household head makes committed changes contiguous. Per-household serialization is an initial correctness trade-off; it does not serialize all households globally. Account actions touching several households take locks in stable ID order or use a resumable deletion job with account access blocked immediately.

### Client catch-up

Authenticate/renew, receive access changes, upload dependency-ordered local intents, then pull authoritative pages (and repeat if new local work arrives). In one SQLite transaction per page, apply committed changes, mark accepted local IDs, preserve rejected work as policy permits, rebuild affected optimistic projections and advance the cursor. The returned page cursor advances only after commit. Never advance from a push acknowledgment past unrelated unseen changes.

Use exponential backoff with jitter; foreground open/reconnect/local changes trigger sync. Proposed foreground polling every 15 seconds while a household list is open; background push is a hint, with catch-up on next execution. Do not promise reliable mobile background execution. UI shows pending/offline/access-loss distinctly.

## 4. Field conflicts and manual resolution

Retain concurrent candidate writes rather than overwriting a row on arrival. Identical normalized candidates collapse without conflict. A materialized value derives from the approved rule:

- Requested amounts: max/min or manual, with policy version captured at discovery. Independent demands merged into one supply candidate requested values, not sums.
- Assignment: Unassigned pending choice.
- Product/unit names and purchase corrections: last common committed value pending choice.
- Notes: union distinct current text variants; a concurrent clear removes only text it observed, not an unseen edit. A later informed clear of all visible variants clears them. Both editors get an in-app notice.
- Independent purchases: sum effective quantities per unique purchase ID. A reversal terminally cancels that purchase for counting; replayed reversals are harmless. Recording a genuinely new purchase uses a new ID.

Manual resolution command references `conflict_id`, `revision`, all observed candidate IDs and a chosen value/action. Lock conflict with household transaction. A valid first resolution closes that exact conflict revision; competing stale resolutions return `already_resolved` with accepted result. Never let a queued old resolution apply to another conflict silently.

If an additional causally concurrent candidate arrives before resolution, extend candidates and increment the revision, retaining the original bound policy. A stale resolution must refresh; if the candidate arrives after closure, create a successor conflict using the accepted resolution as visible base. **Proposed product clarification:** show one new attention item only if the late candidate changes the decision; do not silently discard a late intention or reopen the identical old notification.

Generalize “both people” to all distinct active member actors whose competing actions constitute a multi-party conflict. Store unique `(conflict_revision, recipient, notice_type)` keys. Any resolver can act, but uninvolved users do not receive push solely because they could resolve.

## 5. Products, demand generations and aliases

One active normalized product name per household is the proposed catalog rule. Units are demand/default metadata, not a reason to create two identical product definitions. Concurrent exact-name product creation canonicalizes identities and preserves both creator records. If usual units disagree, **propose a manual default-unit choice**, without changing existing demand units. Until resolved, explicit units remain on existing demands; new additions ask for a unit rather than invent one.

Keep aliases from every merged ID to a canonical UUID. Choose automatic canonical IDs deterministically, e.g. lowest UUID, not first phone clock. A manually selected display name does not require changing storage identity. Alias resolution is household-scoped, cycle-free and transitive; canonical target deletion is not bypassable through an old alias. Operations refer to original identities plus known alias context so historical facts retain origin.

### A product is not a shopping cycle

Each addition carries a fresh `demand_id`, known matching active demand IDs and a frontier of observed prior demand completions/deletions. Same product/unit/list is necessary but not sufficient for all-time merging. Keep those generation facts through aliases.

Examples and proposed resolution:

| Situation | Outcome |
| --- | --- |
| Two phones independently add milk to the same still-open shopping need | Merge compatible demands; requested rule applies; preserve all purchases |
| Phone sees yesterday's milk completed, then adds it today | New generation; yesterday's purchases cannot fulfill today |
| Phone queues edits/purchases to an existing demand that subsequently completes elsewhere | Apply to that specific generation; actual independent purchases remain counted there |
| Offline phone adds milk without knowing an old demand completed, while another phone starts today's demand | **New choice D1:** preserve separate candidate demand generations and ask Merge into current need / Keep as separate need; do not guess from arrival time |

D1 is an approved safety exception to simple exact-name automatic merging across ambiguous generations. Use temporary separate entries marked Needs a choice, with the existing manual workflow. A merge must not transitively bridge a known completed generation into a fresh one through a third stale addition. Maintain incompatible-generation constraints and validate them after product/unit canonicalization.

For merge-eligible unfinished entries, preserve demand contributions, history, notes, exact purchases and assignments; do not duplicate counts by traversing both original and canonical IDs. Different units remain separate unless an explicitly approved equivalent-unit merge unifies them. Combine only same-scale units; known built-in dimension/scale mismatches are blocked. **D3:** for arbitrary custom units, require explicit same-measure confirmation; text similarity cannot establish equivalence.

Renaming a catalog product propagates to current active entry labels; purchase records use immutable product/unit description snapshots. Identity remains stable. Display-name attribution is resolved separately and follows current member name or Deleted member rules.

## 6. Visibility, deletion and unit races

Maintain delete/restore causal actions, not one last-write-wins Boolean. Permanent purge has an absorbing registry entry; it dominates any old alias, restoration, purchase or checkpoint. Deleted visibility and purchase quantities are independent dimensions.

- Concurrent soft delete/restore: delete wins; later restore observing that deletion wins normally.
- Concurrent delete/edit: deleted projection, with preserved list edits where specified.
- Concurrent delete/purchase: the purchase establishes a durable visibility override for the conflicting deletion. An informed later deletion observing that purchase removes it normally.
- **D2:** if that purchase is later undone, keep the already-visible item/list rather than quietly resurrecting an old deletion. The override persists as a causal fact, not as “current purchased total > 0”. This makes replay independent of when undo arrived.
- Restoring a completed historical demand alongside fresh active demand is history-only visibility: do not add another active/Purchased row or change the fresh need. Keep lineage for audit.
- Retiring catalog products/units does not purge referenced demands. Existing or offline-created entries retain retired references; new informed selections cannot choose retired objects without restoration.

### Purchase versus unit changes

Purchases retain the unit seen at action time. An original-unit purchase concurrent with a unit-change intent rejects/supersedes that unit-change intent, preserving the purchase and notifying the editor. New purchases causally depending on the rejected unit change cannot be silently relabeled. **D4:** propose keeping those dependent purchases blocked for the buyer to correct/re-enter with an explicit unit; never automatically count incompatible measures together. This must be tested with three operations, not just a two-operation race.

All reversals permit a later informed unit change while historical units remain fixed. A delayed purchase not covered by those reversals may require D4 revalidation. An integer-mode change is invalid if affected existing exact quantities are fractional; validate household-wide in the serialized transaction. Unit name/identity combining must also check historical fractional constraints before choosing an integer survivor.

### Purge and retention

On permanent list deletion: write a payload-free purge marker for every affected demand identity/alias, revoke their mutation eligibility, purge list item/purchase content from active projections/log payloads and private notification bodies, and deliver removal markers. Rejected queued work is discarded under the approved rule. Keep minimal IDs/outcome fingerprints to stop retries resurrecting data; do not retain full sensitive event payloads merely to maintain an event log.

Old append-only replay cannot be the sole recovery mechanism after erasure. Recovery uses a current authorized snapshot plus retained changes/purge registry. Backup restoration must reapply deletion records before service resumes. Backup lifetime and physical erasure guarantees remain operational design/approval gates, not claims made here.

Account deletion revokes access immediately, invalidates invitations, transfers/closes households and replaces retained shared attribution with a non-account tombstone. Remove email and account mappings from shared responses. Same-email registration gets a new immutable account identity; old outboxes remain ineligible. For a last-member household closure, purge its user data and retain only necessary denial markers.

## 7. Checkpoints, old devices and local isolation

Propose 90 days of incremental feed retention initially; this is not a limit on deleted-list recoverability or business history. An expired cursor returns `snapshot_required`. A snapshot includes current data, pending conflicts, generation metadata, alias mapping, and relevant purge markers at a stable head. Any later changes arrive from that head forward. Do not fabricate completion merely because a cursor expired.

Before reset, retain account-isolated outbox and reconcile known operation outcomes. Install snapshot and cursor atomically, then replay only still-authorized pending intents against it. Missing purged targets are discarded with the approved notice; unresolved dependencies remain blocked and visible rather than being assigned new IDs behind the user's back.

Keep one local account namespace/database and separate credentials in OS-backed storage. Propose encrypted SQLite for cached household content, with an account-bound key protected by OS storage; verify native build support before committing the dependency. Logout hides data and removes active credentials but retains protected pending work for the same account. Confirmed removal hides that household and stops reading its cache. Session revocation locks protected views on the next server contact, unlike mere offline token expiry.

**D5 proposed v1 recovery limits:** no support override that transfers an account without proving access to its current email; recover via mailbox provider. Rejected work after membership removal is not exported through a new path that reveals revoked household contents; retain only an access-loss explanation and purge the inaccessible household cache after handling operation outcomes. This narrows the unresolved recovery question and needs approval before implementation.

## 8. Authentication and notifications

Do not implement password cryptography or email-code verification from scratch as part of grocery-domain code. Keep `IdentityVerifier` / account-service integration separate from household authorization in Go.

| Choice | Trade-off | Proposal |
| --- | --- | --- |
| Hosted identity provider, e.g. Supabase Auth | Adds external identity processor and delivery/cost dependencies; reduces credential infrastructure work | Preferred direction, provider not yet selected |
| Self-host established identity service | Keeps hosting under user's control; adds upgrades, email delivery, abuse controls and recovery operations | Use if hosting/control constraints rule out hosted identity |
| Custom Go credential system | Maximum direct control; largest security/recovery implementation burden | Do not recommend for v1 |

Supabase's official documentation supports one email identity using password or passwordless methods and documents sign-out scopes, including other sessions. It also warns that access-token lifetime and user deletion require attention to actual session validity. These facts establish a candidate, not proof that every required account flow is integrated correctly. See [Users](https://supabase.com/docs/guides/auth/users), [Signing out](https://supabase.com/docs/guides/auth/signout), [User sessions](https://supabase.com/docs/guides/auth/sessions), [User management](https://supabase.com/docs/guides/auth/managing-user-data).

Business data still lives behind the Go API and on the approved PostgreSQL/DigitalOcean architecture. Use provider `(issuer, subject)` to map to app account, never a client email. Add an app session/grant check on all protected API operations so “signed out elsewhere” or deleted account cannot keep mutating solely with an unexpired JWT. Provider password/email operations must be integrated with this revocation gate; provider-side bypass/webhook lag must be tested before choosing a provider.

Fresh-code account deletion and two-address email change need proof tokens bound to the authenticated action/session, short-lived and single-use. Opening an invitation or receiving a code does not grant household membership. Invitation acceptance and revocation serialize with membership transactions. Hash invite tokens at rest; check one-use/7-day expiry at acceptance.

Propose Expo push transport with a durable PostgreSQL delivery outbox, in-app notice records and no Redis. Group eligible jobs per recipient for a short window (proposed 5 seconds); allow generic push text without product names. Retry transient errors and disable invalid device tokens. Delivery cannot be guaranteed exactly once; deduplicate server jobs and use stable notification IDs/collapse hints where supported. No correctness depends on push delivery. Verify transport details before implementation.

Expo SQLite exposes transactional APIs including exclusive transactions, suitable for local state/outbox boundaries; transaction scoping must use its transaction connection. See [official SQLite reference](https://docs.expo.dev/versions/latest/sdk/sqlite/). Implementation still requires real-device crash/restart tests.

## 9. Proposed module boundaries

- `apps/mobile/src/domain`: types, exact quantities, normalized names, local optimistic reducers.
- `apps/mobile/src/storage`: migrations, committed projections, outbox and transaction boundary.
- `apps/mobile/src/sync`: upload/download, checkpoints, access handling, replay; no screen dependencies.
- `services/api/internal/{identity,households,catalog,lists,sync,notifications}`: bounded modules inside one deployable Go service.
- `contracts/`: OpenAPI plus versioned operation/change schemas and reducer conformance fixtures.
- `db/`: migrations and household-scoped queries. Exact file paths/interfaces belong in the next implementation plan.

TypeScript/Go reducers must pass identical fixture inputs/outputs. The phone may show a provisional state; the server produces the final accepted projection. Avoid implementing subtly different merge logic in screens, API handlers and background jobs.

## 10. Validation required before full implementation

Design acceptance tests, not executed tests:

1. Crash at every local transaction/ack/cursor boundary: no lost outbox or duplicated purchase.
2. Duplicate upload, payload mismatch, response timeout, server rollback and dependency reversal order.
3. Causal reduction versus concurrent max/min; policy switches before/after discovery; third candidate after resolution.
4. Separate purchase sum, same-purchase correction conflict, reversal-before-purchase and unit-change dependent purchase (D4).
5. Additions across completed/deleted generations; three-way alias merges never bridge incompatible cycles (D1).
6. Soft delete/restore/purchase permutations, later undo of visibility-winning purchase (D2), permanent purge absorbing all retries.
7. Catalog/default-unit conflicts, same-scale unit merge, integer constraints and historical snapshots.
8. Exact/similar names, restore with reused retired names, concurrent offline list/display-name collisions.
9. Note edit/edit and clear/edit convergence, distinct notification channel recipients and read-versus-resolved state.
10. Membership removal/rejoin grant changes; two accounts on one device; password/session revocation and user deletion.
11. Snapshot reset with pending operations, purged data and retained conflict candidates; backup restore reapplies purge registry.
12. Concurrent PostgreSQL transactions allocate committed household feed heads without missing changes; isolation across households.

Use real PostgreSQL transactions and actual SQLite integration tests, plus deterministic domain fixtures. For server-order-dependent policies, hold the accepted order fixed when comparing client convergence; test alternate accepted orders against their specified outcomes, not a false equality expectation.

## 11. Review decisions and next step

Fresh builder review identified two additional product cases. Proposed defaults for review:

- **D6 — competing unit edits:** when two people change the same unpurchased demand to different units offline, keep its previous unit pending a manual choice. Preserve purchases made against either proposed unit separately as pending facts; never sum incompatible quantities. Apply D4 correction handling where needed. This needs approval because the existing original-unit purchase rule does not settle two unit edits by themselves.
- **D7 — occupied fallback name:** a delayed rename conflict may need the last common name after another product/unit has already legitimately claimed it. Preserve the existing name claim and show the conflicted object under its previous label with a visible “name needs a choice” marker; that label is not a second active name claim. Resolution must select an available name or an eligible explicit merge. This narrow exception to displaying an ordinary previous name needs approval.

These are recorded gaps, not completed implementation rules. The next plan must specify causal-metadata compaction without losing the common-base values or generation constraints needed by arbitrarily old offline edits; feed retention alone must not erase them.

Review record (presented defaults approved; see approval scope at the top):

- D1: ambiguous stale additions across shopping generations get a choice instead of silently merging fresh demand with history.
- D2: undoing a purchase that rescued a deleted item/list does not silently hide it again.
- D3: custom-unit combination needs explicit equivalence confirmation; incompatible built-ins remain blocked.
- D4: purchases depending on a rejected unit change require correction, not relabeling or conversion.
- D5: v1 has no mailbox-ownership bypass and no revoked-household data export.
- D6–D7: competing unit edits and occupied fallback names use the proposed pending states above.
- Exact-name catalog creations with differing defaults merge identity but ask for the future default unit; a materially different late conflict candidate creates a new choice as described in sections 4–5.
- Hosted identity direction versus self-hosting, without approving a paid account or vendor yet.

The bounded [foundation/offline implementation plan](../superpowers/plans/2026-11-25-foundation-offline-core.md) is now prepared for review. Select provider/tooling based on verified constraints when its milestone requires it. Do not scaffold or deploy based solely on design approval.
