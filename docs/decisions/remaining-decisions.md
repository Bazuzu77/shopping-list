# What remains to be decided

Updated 2026-11-28. The main shopping behavior is agreed. This page separates genuinely unfinished work from questions already answered.

## Decisions we do not need to ask again

| Situation | Agreed result |
| --- | --- |
| An old offline addition might belong to a new shopping trip | Ask whether to combine it with the current need or keep it separate |
| A purchase prevented deletion, then someone undoes that purchase | Keep the item or list visible |
| People combine custom units | Require confirmation that they mean the same measure and scale |
| People change an item's unit in conflicting ways | Ask for a choice; keep affected purchases available for correction without converting them silently |
| A name needed during conflict handling is now taken | Keep both records accessible and show that a name needs choosing |
| Someone loses their old mailbox | Recover access through the email provider; support does not bypass proof of account ownership |
| Someone loses household access | Do not offer an export that exposes the removed household's data |
| Who handles sign-in? | Prefer an established hosted service; the actual provider is not selected |

These are the D1–D7 decisions in the [offline design](../architecture/sync-data-design.md). They are approved, not open questions.

## Work still needed

| Work | What needs to be written or checked | Tracked in |
| --- | --- | --- |
| Choose sign-in service | Email codes, optional passwords, recovery, email changes, account deletion, signing out other devices, costs and limitations | [#5](https://github.com/Bazuzu77/shopping-list/issues/5) |
| Finish the exchange rules | Exact messages sent between phone and server, replies, and examples with expected results | [#8](https://github.com/Bazuzu77/shopping-list/issues/8) |
| Catch up after a long absence | What information must be kept after older update history is removed, so valid offline work survives and deleted data cannot return | [#8](https://github.com/Bazuzu77/shopping-list/issues/8) |
| Check two remaining proposals | Same-name products created with different default units; a new competing edit arriving after a conflict was resolved | [#8](https://github.com/Bazuzu77/shopping-list/issues/8) |
| Protect data on phones | Account separation, encrypted storage support, sign-out, lost membership and revoked sessions | [#6](https://github.com/Bazuzu77/shopping-list/issues/6) |
| Choose and test notifications | Delivery provider, correct recipients, grouping and safe retries | [#16](https://github.com/Bazuzu77/shopping-list/issues/16) |
| Deletion and backup handling | How long backups remain, how deletions are reapplied after recovery, and what we can honestly promise users | [#17](https://github.com/Bazuzu77/shopping-list/issues/17) |
| Prepare the actual build and release | Supported tool versions, database changes, device checks, hosting setup, monitoring and release requirements | Feature plans and [#19](https://github.com/Bazuzu77/shopping-list/issues/19) |

The proposed technical defaults are not all approved promises. These include maximum quantities, how often phones check for updates, how long update history is retained, and how briefly alerts are grouped. Their purpose and suggested values remain in the [engineering reference](../architecture/sync-data-design.md#engineering-reference).

## Details the builder can normally settle

Write down and test these without restarting a long series of small product questions:

- How names are compared for exact matches, how similar-name suggestions are ranked, and how alphabetical ordering works.
- What makes two notes identical, how new offline entries are ordered, and how rounding ties are handled.
- How the app and server identify the same change, its earlier dependencies and the last state the person had seen.
- How merged records retain their original identities and purchase history.
- How more than two people are included in a conflict and its notifications.
- How a partial purchase follows the same deletion-conflict rules as a full purchase.

If a technical choice would change an agreed user outcome, explain that consequence before changing the rule.

## Facts needed before committing to services or release

- Service budgets and constraints for sign-in, email and push.
- Actual DigitalOcean setup, domain, backups and capacity. Inspect before modifying existing services.
- App name and identifiers, Apple/Google developer accounts, available test devices, initial languages and beta audience.
- The Go server remains the planning baseline. The historical Spring Boot question is not permission to change it.

No paid provider, deployment or publication is selected by this page.

## Next steps

Use the [architecture epic](https://github.com/Bazuzu77/shopping-list/issues/33) for design work and the [first-version plan](../superpowers/plans/2026-11-25-foundation-offline-core.md) for the first build milestone. Do not rewrite the completed design discussion from scratch.

This rewrite changes presentation and removes obsolete “unapproved” labels from already-approved decisions. It does not claim that implementation or testing is complete.
