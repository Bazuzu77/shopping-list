# Remaining Decisions and Design Work

Reviewed: 2026-11-25. Companion to [product-behavior.md](../specs/product-behavior.md).

The main product behavior is settled. Do not restart the earlier sequence of tiny approval questions. Draft coherent proposals for the substantive gaps below, use reasonable documented implementation choices for routine details, and ask only where a choice changes user-visible outcomes or operating commitments. No implementation or deployment is authorized yet.

## 1. Product gaps that affect outcomes

Approval update — 2026-11-25: the user approved the presented D1–D7 defaults and hosted-identity direction. Those entries below are retained as discovery history, not questions to ask again. Exact provider selection, operational limits, catalog-default/late-candidate compatibility checks and implementation validation remain engineering work. Next bounded milestone: [foundation and offline core plan](../superpowers/plans/2026-11-25-foundation-offline-core.md).

| Gap | Why it matters | Proposed direction — not yet approved |
| --- | --- | --- |
| Delayed additions across shopping cycles | A phone adds milk without knowing another demand was completed/deleted. Name matching alone cannot tell repeat shopping from duplication. | Define demand/cycle identity and preserve intentional new demand; present concrete offline examples with the sync proposal. |
| Purchase undone after overriding deletion | A concurrent purchase kept a deleted item/list visible. Undoing that purchase leaves an unclear visibility result. | Keep the already-restored visibility rather than silently delete again; confirm with related lifecycle cases together. |
| Product identity versus units | Catalog-name uniqueness and list name+unit matching are separate. Two same-name catalog creations can have different defaults without being different products. | One active catalog product per normalized name, with independent demand units; manual choice for incompatible defaults. Confirm how offline default conflicts resolve. |
| Custom-unit equivalence | Users can create arbitrary units; similar strings cannot prove same measure/scale. | Explicit equivalence confirmation for custom-unit combination; prohibit known incompatible built-in measures. Do not silently introduce conversion. |
| Account/access recovery | Verified access to both email addresses is required for email change. Losing the old mailbox or rejected offline work has no approved recovery path. | Specify support/no-recovery options and rejected-work retention without reopening removed-household access. |
| Cross-layer edge cases | Permanent deletion, retirement, merges, pending resolutions and renamed identities can overlap. | Draft a bounded set of rules for precedence and history; elevate only cases whose user outcome is not implied by approved rules. |

Notes: partial purchases are purchases and should be modeled consistently with the approved deletion/purchase rule; document this interpretation in the technical proposal rather than treating every partial amount as a new feature question. More than two editors can conflict: the protocol must generalize recipients and candidates without leaking unrelated household data.

## 2. Technical designs to prepare, not questions to repeat

1. **Synchronization contract.** Operation IDs, stable entities/aliases, causal metadata, transactions, accepted-change sequence/checkpoints, stale-device recovery, tombstones and conflict IDs. Define which guarantees depend on server acceptance order; do not promise universal delivery-order independence where approved rules explicitly use that order.
2. **Conflict policy versions.** Bind the current household policy when a conflict is discovered; retain it for pending conflicts. Specify resolution validity when additional concurrent operations arrive later.
3. **Merge model.** Separate catalog identities from demand generations and purchases; preserve exact quantities and attribution through merged products/units. Define partial restore merges without importing old completed purchases into fresh demand.
4. **Identity and sessions.** Select an authentication approach supporting one account with code/password, persistent sessions, revocation, online sensitive actions and offline expiry. Evaluate provider integration with the selected Go/PostgreSQL/DigitalOcean baseline; no provider is approved yet.
5. **Local storage/privacy.** Account-isolated SQLite/outbox and credentials; logout, removal, revoked sessions, permanent deletion and backup retention. User-facing deletion must not be confused with instantaneous physical removal from all backups. Review applicable privacy/store requirements against current official sources during this design.
6. **Notifications.** Push provider, authenticated device registration, in-app durable notices, grouping, read-versus-resolved state and no repeated delivery effects. Push is never the source of truth.
7. **Validation and usability.** Exact decimal storage/tie rounding, bounds, Unicode/name normalization, display-name collation, unit equivalence, similarity thresholds, note text identity and deterministic ordering. Record choices with useful acceptance cases; do not burden the user with implementation trivia.
8. **Foundation.** Resolve workspace Git/toolchain issues, pin currently supported dependencies, decide migration/state tooling and document reproducible checks. Historical inspection results in PLAN.md require re-verification.

## 3. User/environment facts needed before implementation or release

- Confirm whether the existing Go planning baseline stands or whether the earlier Spring Boot question needs a focused comparison. Do not silently change stack.
- Authentication/push/email service constraints and budget before selecting paid providers.
- Inspect actual DigitalOcean access, services, capacity, backup setup and domain; isolate existing workloads. Deployment requires explicit approval.
- App identity/name, bundle identifiers, developer-account availability, physical iOS/Android test devices, initial languages and beta audience before signing/release setup.

## 4. Recommended next deliverables

Update 2026-11-25: [sync-data-design.md](../architecture/sync-data-design.md) contains the design and a fresh builder review. D1–D7 and the hosted-identity direction are now approved. Catalog-default/late-candidate compatibility checks and provider selection remain engineering work. Implementation remains a later milestone; the foundation/offline plan is ready for review.

1. Review the consolidated behavior and this gap list as a whole, rather than reconfirming approved rules.
2. Prepare a synchronization/data-model proposal with the high-impact cases above, plus authentication/session integration options. Keep choices marked as proposals until approved.
3. After that design is approved, write a bounded implementation plan for the foundation and offline mobile slice. Keep the existing milestone boundaries; no full-app scaffold, infrastructure changes or publication is implied by documentation approval.

## 5. Consolidation audit

- Preserved the final choices where earlier proposals were superseded: deletion wins over restoration; all deletion conflicts notify; unit Keep separate is not remembered, product Keep separate is; account-required onboarding replaces guest use; ordinary decimal precision is rounded; retired names can be reused.
- Distinguishes list-entry edits from shared catalog editing and item-local unit substitution from explicit household-wide unit combination.
- Captures the last unsaved note answers: concurrent text keeps both versions with in-app notices to both editors; clear/edit keeps edited text with the same notices, no push.
- Preserves the chronological decision record in PLAN.md and the untouched source handover. This review does not claim tests, provider selection, completed technical design or implementation readiness.
