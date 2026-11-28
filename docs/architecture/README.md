# Architecture and technical design

This is the entry point for the app's architecture. Updated 2026-11-28.

Tracked by [Epic #33: Architecture and technical design](https://github.com/Bazuzu77/shopping-list/issues/33). The epic owns sign-in selection (#5) and synchronization contracts (#8); implementation stays in the feature epics.

## Artifacts

| Document | What it provides | Status |
| --- | --- | --- |
| [System overview](system-overview.md) | Components, data ownership, security boundaries and deployment shape | Summary of the existing design baseline; not proof of implementation |
| [Data model and synchronization design](sync-data-design.md) | Offline operations, conflict rules, deletion, recovery and D1–D7 | Direction and presented defaults approved; some precise contracts remain open |
| [Approved product behavior](../specs/product-behavior.md) | What users should experience | Approved requirements |
| [Foundation and offline core plan](../superpowers/plans/2026-11-25-foundation-offline-core.md) | First bounded implementation tasks and checks | Prepared for implementation review |
| [Remaining decisions](../decisions/remaining-decisions.md) | Provider, operating constraints and other engineering work | Open items; later approvals supersede historical entries |

## Remaining architecture work

- [Choose the sign-in service — #5](https://github.com/Bazuzu77/shopping-list/issues/5).
- [Finish precise synchronization contracts and worked examples — #8](https://github.com/Bazuzu77/shopping-list/issues/8).
- Concrete database migrations, full endpoint definitions and deployment configuration belong to the implementation plans for their features. Do not mistake the logical model for finished schemas or the overview for running infrastructure.

Keep user-facing decisions in the product behavior document, architecture mechanisms here, and live progress on GitHub issues. Updating architecture must not silently change an approved user outcome.
