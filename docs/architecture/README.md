# App design — start here

Updated 2026-11-28. These documents explain the planned app in everyday language. You do not need to read code to understand the design.

## What would you like to understand?

| Your question | Read this |
| --- | --- |
| How does the whole app work? | [How the app fits together](system-overview.md) |
| What happens when two people work offline? | [How offline changes are combined](sync-data-design.md) |
| What exactly should users be able to do? | [How the app should behave](../specs/product-behavior.md) |
| What will we build first? | [The first working version](../superpowers/plans/2026-11-25-foundation-offline-core.md) |
| What still needs a decision or more detail? | [What remains to be decided](../decisions/remaining-decisions.md) |

**Status:** the overall direction and listed user-facing rules are approved. The app has not been implemented. Some technical choices remain open.

The sync document and first-version plan include expandable **Engineering reference** sections. Those preserve precise details for the person building the app; the main text explains their purpose.

## Where progress is tracked

[Architecture epic #33](https://github.com/Bazuzu77/shopping-list/issues/33) owns:

- [#5 — Choose a sign-in service](https://github.com/Bazuzu77/shopping-list/issues/5).
- [#8 — Finish the exact rules for exchanging offline changes](https://github.com/Bazuzu77/shopping-list/issues/8).

The [project plan](https://github.com/users/Bazuzu77/projects/1/views/2) shows the wider work. A document being available does not mean its feature is built or its tests have passed.

## Which record should we trust?

Use the current design pages for the current rules, and issues for work status. [PLAN.md](../../PLAN.md) records the discussion history; some older questions were answered later. The [original handover](../reference/CODEX_HANDOVER.md) stays unchanged as a historical source.

Changes to wording must preserve approved behavior. A new user-visible rule still needs to be identified as a new decision.
