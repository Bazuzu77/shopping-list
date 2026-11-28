# Implementation backlog

Repository: [Bazuzu77/shopping-list](https://github.com/Bazuzu77/shopping-list). All 19 work items are published as GitHub issues. Issues track live progress; [issues.json](issues.json) preserves their initial scope and links locally.

## Start here

Read [approved behavior](../specs/product-behavior.md), [sync design and approval](../architecture/sync-data-design.md), then the selected issue. F01–F04 have a detailed [implementation plan](../superpowers/plans/2026-11-25-foundation-offline-core.md). Later stages need a bounded plan before implementation. [Remaining decisions](../decisions/remaining-decisions.md) tracks engineering choices. [PLAN.md](../../PLAN.md) is historical; later approvals supersede older open questions.

## GitHub Project

[Plan: ordered epics](https://github.com/users/Bazuzu77/projects/1/views/2) · [Work board](https://github.com/users/Bazuzu77/projects/1/views/3) · [Future](https://github.com/users/Bazuzu77/projects/1/views/4) · [Dependency diagram](dependencies.md).

The project contains all 33 issues. It is private to the owner and authorized collaborators; the repository remains public. Roadmap order controls display, not dates or hard dependencies. Architecture comes first and the timetable stays last. Actual blockers are recorded through native issue relationships.

## Epics and future roadmap

Current work is grouped into six tracking epics. Future epics are placeholders, outside the first release; their detailed behavior and implementation are not approved. Their order is indicative, with the shared household timetable explicitly farthest away. The original handover remains unchanged.

- [Epic: Architecture and technical design](https://github.com/Bazuzu77/shopping-list/issues/33) — #5, #8; [design artifacts](../architecture/README.md)
- [Epic: Offline shopping](https://github.com/Bazuzu77/shopping-list/issues/20) — #1, #2, #3, #4
- [Epic: Accounts and households](https://github.com/Bazuzu77/shopping-list/issues/21) — #6, #7
- [Epic: Sync and collaboration](https://github.com/Bazuzu77/shopping-list/issues/22) — #9, #10, #11, #16
- [Epic: Products and shopping lists](https://github.com/Bazuzu77/shopping-list/issues/23) — #12, #13, #14, #15
- [Epic: Privacy, testing and release](https://github.com/Bazuzu77/shopping-list/issues/24) — #17, #18, #19
- [Future epic: Device-to-device sync](https://github.com/Bazuzu77/shopping-list/issues/25) — Future
- [Future epic: Supermarket integrations](https://github.com/Bazuzu77/shopping-list/issues/26) — Future
- [Future epic: Barcode scanning](https://github.com/Bazuzu77/shopping-list/issues/27) — Future
- [Future epic: Recipes and meal planning](https://github.com/Bazuzu77/shopping-list/issues/28) — Future
- [Future epic: Shopping insights](https://github.com/Bazuzu77/shopping-list/issues/29) — Future
- [Future epic: Optional AI suggestions](https://github.com/Bazuzu77/shopping-list/issues/30) — Future
- [Future epic: Subscriptions and monetization](https://github.com/Bazuzu77/shopping-list/issues/31) — Future
- [Future epic: Shared household timetable](https://github.com/Bazuzu77/shopping-list/issues/32) — Farthest future

Native relationships verified on 2026-11-28: 19 sub-issues under the six current epics, plus 22 blocked-by links. Checklists and backlinks remain useful navigation. [epics.json](epics.json) and [relationships.json](relationships.json) preserve the grouping and dependency map. This is not a custom GitHub issue-type configuration or a milestone assignment.

## Individual tasks

| Issue | Stage | Depends on |
| --- | --- | --- |
| [#1 Set up the app and handle quantities correctly](https://github.com/Bazuzu77/shopping-list/issues/1) | Foundation | None |
| [#2 Track what is needed and what was bought](https://github.com/Bazuzu77/shopping-list/issues/2) | Foundation | [#1](https://github.com/Bazuzu77/shopping-list/issues/1) |
| [#3 Keep offline changes safe after closing the app](https://github.com/Bazuzu77/shopping-list/issues/3) | Foundation | [#2](https://github.com/Bazuzu77/shopping-list/issues/2) |
| [#4 Build the first offline shopping screen](https://github.com/Bazuzu77/shopping-list/issues/4) | Foundation | [#3](https://github.com/Bazuzu77/shopping-list/issues/3) |
| [#5 Choose a sign-in service](https://github.com/Bazuzu77/shopping-list/issues/5) | Identity | None |
| [#6 Add sign-in and protect saved account data](https://github.com/Bazuzu77/shopping-list/issues/6) | Identity | [#5](https://github.com/Bazuzu77/shopping-list/issues/5), [#3](https://github.com/Bazuzu77/shopping-list/issues/3) |
| [#7 Create households and invite members](https://github.com/Bazuzu77/shopping-list/issues/7) | Households | [#6](https://github.com/Bazuzu77/shopping-list/issues/6) |
| [#8 Define how offline changes are exchanged](https://github.com/Bazuzu77/shopping-list/issues/8) | Synchronization | None; #2 is related work |
| [#9 Save shared changes safely on the server](https://github.com/Bazuzu77/shopping-list/issues/9) | Synchronization | [#8](https://github.com/Bazuzu77/shopping-list/issues/8), [#7](https://github.com/Bazuzu77/shopping-list/issues/7) |
| [#10 Sync phones after working offline](https://github.com/Bazuzu77/shopping-list/issues/10) | Synchronization | [#9](https://github.com/Bazuzu77/shopping-list/issues/9), [#3](https://github.com/Bazuzu77/shopping-list/issues/3) |
| [#11 Resolve conflicts without losing shopping intent](https://github.com/Bazuzu77/shopping-list/issues/11) | Synchronization | [#10](https://github.com/Bazuzu77/shopping-list/issues/10) |
| [#12 Reuse products and prevent duplicates](https://github.com/Bazuzu77/shopping-list/issues/12) | Catalog and lists | [#11](https://github.com/Bazuzu77/shopping-list/issues/11) |
| [#13 Support useful units and custom units](https://github.com/Bazuzu77/shopping-list/issues/13) | Catalog and lists | [#12](https://github.com/Bazuzu77/shopping-list/issues/12) |
| [#14 Add multiple lists, buyers and notes](https://github.com/Bazuzu77/shopping-list/issues/14) | Catalog and lists | [#13](https://github.com/Bazuzu77/shopping-list/issues/13) |
| [#15 Delete and restore lists and items correctly](https://github.com/Bazuzu77/shopping-list/issues/15) | Catalog and lists | [#14](https://github.com/Bazuzu77/shopping-list/issues/14) |
| [#16 Show useful in-app notices and push alerts](https://github.com/Bazuzu77/shopping-list/issues/16) | Notifications | [#11](https://github.com/Bazuzu77/shopping-list/issues/11), [#6](https://github.com/Bazuzu77/shopping-list/issues/6) |
| [#17 Handle account deletion and lost access](https://github.com/Bazuzu77/shopping-list/issues/17) | Privacy and recovery | [#15](https://github.com/Bazuzu77/shopping-list/issues/15), [#7](https://github.com/Bazuzu77/shopping-list/issues/7) |
| [#18 Test offline use and collaboration end to end](https://github.com/Bazuzu77/shopping-list/issues/18) | Validation | [#16](https://github.com/Bazuzu77/shopping-list/issues/16), [#17](https://github.com/Bazuzu77/shopping-list/issues/17) |
| [#19 Prepare hosting and a beta release](https://github.com/Bazuzu77/shopping-list/issues/19) | Release | [#18](https://github.com/Bazuzu77/shopping-list/issues/18) |

## Keep context durable

- Keep issues short and human readable: purpose, Done when, dependencies and links to technical details.
- Check dependencies and current repository state before starting. No issue is completed merely because it was planned.
- Record progress, verification results, blockers and linked commits/PRs on the issue. Close only after its acceptance checks pass.
- Update the specification when an approved decision changes, then update affected issues; do not rely on chat memory.
- Stable backlog IDs in issue bodies prevent duplicate creation during publication retries.
- Repository/backlog setup is authorized. Application implementation, paid services and deployment are separate commitments.

