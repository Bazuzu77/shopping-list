# Family Grocery App — Consolidated Product Behavior

Updated: 2026-11-25

This document consolidates user-approved behavior from [PLAN.md](../../PLAN.md) and the [original handover](../reference/CODEX_HANDOVER.md). It introduces no new product decisions and does not authorize implementation. Later explicit decisions override earlier proposals. Outstanding choices and technical work are separated in [remaining-decisions.md](../decisions/remaining-decisions.md).

## 1. Scope and concepts

### Approved follow-up — 2026-11-25

The user approved the defaults presented with the [synchronization design](../architecture/sync-data-design.md): ambiguous additions across old/new shopping needs require a merge/separate choice; undoing a purchase that prevented deletion keeps the item/list visible; combining custom units requires same-measure-and-scale confirmation; conflicting unit changes require a choice and preserve affected purchases for explicit correction without conversion; occupied fallback names preserve both identities with a pending-name marker. Recovery relies on access to the current mailbox, with no support ownership bypass or revoked-household data export. Prefer an established hosted identity provider; no vendor or spending is approved. Sections D1–D7 of the design provide the concrete cases. These decisions supplement the rules below; implementation remains a separate milestone.

Build a native iOS/Android, offline-first household grocery app. Multiple households, members and named lists are supported. Backend planning baseline: Go modular monolith, PostgreSQL, DigitalOcean; mobile baseline: Expo/React Native, TypeScript and SQLite. Exact versions and providers need implementation-time verification.

Keep these concepts distinct:

| Concept | Meaning |
| --- | --- |
| Catalog product | Reusable household product definition, including its usual unit |
| List entry/demand | A product needed on a particular list, with requested quantity, unit, assignee and optional note |
| Purchase | An independently attributed buying action with its own quantity |
| Correction/reversal | A change to, or undo of, a specific purchase; not another purchase |
| Catalog retirement | Remove a product/unit from future selection while retaining existing references |
| List-entry deletion | Remove a particular shopping need, with restoration behavior |

Everything created within a household is available to its members unless explicitly restricted. Any current member may edit or remove shared content, including others' purchases. Account and membership administration have separate rules below. Preserve creator, assigned buyer, purchaser and subsequent editor attribution separately.

Deferred: nearby device transfer, subscriptions/payments, ads, AI, recipes, meal planning, price comparison, barcode scanning, advanced analytics and enterprise infrastructure. Similarity suggestions do not imply an external AI service.

## 2. Catalog and adding items

- A household has its own reusable product catalog, never a catalog exposing unrelated households' data. Buying/removing a list entry does not remove its catalog product.
- Search existing catalog products before creating a new definition. Suggest similar names with **Use existing item** or **Create separately**. Similarity alone never merges products.
- A saved product prefills its usual unit; requested quantity defaults to 1. Override quantity/unit for one entry without changing the catalog default. The default unit changes only through explicit catalog-product editing.
- A new entry defaults to the adding person as assignee. Later ordinary edits preserve assignment unless explicitly changed. Unassigned is supported.
- Exact matching for automatic list-entry merging ignores capitalization and extra spaces, and requires matching names and units. Two omitted units match; omitted versus specified units do not. Do not convert units implicitly.
- Merge eligible matching active entries only within the same list. Requested quantities use the household quantity-conflict rule; independent purchases add together. Preserve both original creators and histories. Different assignees trigger Choose buyer. Keep distinct notes and deduplicate identical notes.
- Reusing a product after seeing its previous entry completed or deleted creates a fresh demand with no inherited purchases. It does not recreate the catalog definition. Different lists have independent demands.
- Independently created similar products discovered on sync require **Merge** or **Keep separate**, using the standard manual workflow in section 5. Remember Keep separate for that product pair until one is renamed.
- A product-merge decision chooses the surviving existing product name and, if different, the default unit for future additions. Existing entries/purchases keep their units. Different-unit entries remain separate; eligible matching-unit entries use existing merge rules.
- Renaming a product from a list edits the shared catalog product and updates active entries throughout the household. Purchase history keeps the product name recorded at purchase time.
- Block a rename to another existing catalog product's name. If an offline rename collision is discovered on sync, restore the last accepted name and notify that editor in-app, preserving unrelated work. Competing renames use the manual workflow and retain the last common name pending a choice.

## 3. Quantities, purchases and completion

- Unit omitted: whole numbers only. Decimal-capable units permit three decimal places; accept comma or dot as decimal separator. Automatically round excess precision without a prompt unless a positive amount rounds to zero; then ask for a larger amount or smaller unit. Unit-specific integer rules still apply.
- **Purchased** defaults to the remaining amount known on that phone; allow manual quantity adjustment. Offline phones can both purchase the same remaining amount.
- Independent purchases sum, including offline purchases. One person buys 1 and another buys 3: total 4. Replaying one operation adds nothing again. Never apply requested-quantity max/min rules to independent purchases.
- Completed items offer **Add purchase**, requiring an explicitly entered extra quantity. Preserve each purchase and buyer indicator.
- Effective purchased quantity determines completion against current demand. Increasing demand or reducing effective purchases reopens an item if something remains. Lowering demand below purchases completes the item and displays excess, such as **2 of 1 bought · 1 extra**.
- Anyone in the household may correct/undo purchases. Preserve original facts and attribution in history. Undo contributes zero and leaves current requested demand unchanged.
- Competing corrections to one purchase require manual choice; keep the last common agreed purchase amount meanwhile. Undo wins over a concurrent correction to that same purchase; no choice is required. Notify both editors in-app only.
- A unit can change before purchases or after all purchases have been undone. While effective purchases remain, a different unit requires a fresh item. History retains original units.
- If a unit change conflicts with an offline purchase in the original unit, keep the original unit and purchase; notify the unit editor in-app.

## 4. Units

- Provide a broad shopping-unit catalog. Count/package units use integers (pieces, cartons, bottles, cans, boxes, packs, bags, etc.); weight, volume and length units allow decimals (kg, g, litres, ml, metres, etc.). The detailed starter list is a planning selection, not a fixed user-authored list.
- Members may create household-shared custom units. Whole-number/decimal selection is optional; skipped means whole numbers.
- Block changing a unit to whole-number-only while fractional quantities use it. Do not invalidate existing records.
- Exact same-name custom units created offline merge automatically if their quantity types match. Different types require manual resolution; temporarily permit decimals, and disallow an integer-only resolution while fractional records exist.
- Similar-unit creation prompts offer **Use existing unit** or **Create separately**. Use existing adopts that unit's existing quantity rules. Unlike product pairs, do not permanently remember Keep separate to suppress future creation suggestions.
- Similar units discovered after offline creation require **Use one unit** or **Keep separate** with the standard manual workflow. Do not repeatedly reopen the same resolved sync conflict.
- Competing unit renames use the manual workflow, retaining the previous name until resolved. A duplicate rename is blocked, with Use existing unit offered. When invoked while editing an item, this changes only that item, subject to purchase/unit restrictions.
- Explicit unit combination is a separate household-wide action: choose the surviving existing unit, update list entries and product defaults, and retain original unit names in purchase history. An integer-only survivor is unavailable while either unit has fractional quantities recorded.
- Only combine equivalent measures of the same scale, such as litre/liter. Preserve numbers; no g-to-kg conversions or silent reinterpretation.
- Retiring a unit preserves it on existing entries/history but removes it from new selections. An entry created offline with a subsequently retired unit is preserved. Apply the same rule to catalog-product retirement.

## 5. Conflict rules and notifications

### Standard manual-choice workflow

Any current household member may resolve. Notify both involved editors/creators in-app and by push, not uninvolved household members. First valid resolution accepted by the server stands; offline choices remain pending until accepted. Other resolvers see the result and may make a subsequent normal edit.

| Conflict | Behavior while unresolved / outcome |
| --- | --- |
| Requested quantity | Household rule: larger wins (default), smaller wins, or Ask us to choose. Ask temporarily uses larger amount with Needs a choice |
| Assignment | Unassigned / Choose buyer until manually resolved |
| Product/item name | Last common name / Needs a choice until manually resolved |
| Quantity on one purchase record | Last common agreed purchased quantity until manually resolved |
| Custom-unit name | Previous name until manually resolved |
| Same-name custom-unit quantity type | Decimals permitted temporarily; manual choice must preserve existing fractions |
| Similar product/unit merge | Preserve candidates until explicit merge/separate decision |
| Note edit versus note edit | Preserve both text versions; no manual choice; in-app notice to both editors |
| Note clear versus note edit | Keep edited text; in-app notice to both editors |
| Purchase undo versus correction | Undo wins; in-app notice to both editors |

The note rules above include the final decisions approved after the previous batch save. Notes belong to list entries, not automatically to catalog defaults.

### Household quantity policy

- Any member can change the household's shared larger/smaller/ask setting. Ordinary later edits can intentionally reduce quantities; this is not a permanent maximum.
- Automatic max/min resolution stays quiet. Manual conflicts use the standard workflow.
- A setting change affects new conflicts only, never recalculating resolved quantities. Existing manual conflicts remain pending under their original rule.
- New conflicts discovered on sync use the current household rule even if operations originated under an older setting.
- Competing offline setting changes: last valid setting change accepted by the server wins; notify both editors in-app. This differs from first-accepted manual resolutions.

### Notification lifecycle

- All deletion conflicts notify both involved people in-app and by push; distinguish deletion from purchase undo.
- Group conflicts arriving together into notifications while retaining individual details and respecting each recipient/channel. In-app-only events must not become push alerts through grouping.
- Unresolved conflicts stay in **Needs attention** until resolved, even after a notice is read.
- Opening informational notices marks them read, retaining notification history. Reading does not resolve a conflict.
- Membership loss, rejected offline renames and offline unit-change rejection have the explicit explanations described elsewhere. Duplicate list/member-name collisions notify only the affected creator/editor in-app.
- Sync must work without notifications. No confirmed remote-sync indicator before server acceptance.

## 6. Deletion, restoration and names

| Event | Approved result |
| --- | --- |
| Item deletion concurrent with purchase | Item stays visible; purchase preserved; notify both |
| Item deletion concurrent with name/quantity edit | Item stays deleted; notify both |
| Deliberate deletion after seeing purchase | Remove item from list; preserve history |
| Concurrent restore/delete of item or list | Deletion wins; notify both; subsequent informed restoration still works |
| Whole-list deletion concurrent with purchase | List remains visible; purchase preserved; notify both |
| Whole-list deletion concurrent with additions/edits only | List stays deleted; retain those changes for restoration; notify both |
| Permanent list deletion versus pending offline changes | Discard pending changes on reconnection; in-app explanation; never resurrect list |

- Restore an item with its quantities, assignment and purchase history. If its assigned member has left, use Unassigned and preserve historical attribution.
- Restoring an unfinished item into a matching active demand uses the existing merge rules.
- Restoring an already completed item must not change a fresh matching active demand's display, quantities, assignment or completion. Do not add a duplicate visible entry or count old purchases toward current demand; retain historical facts.
- Whole-list deletion moves it to **Deleted lists**, recoverable by any household member until explicit Delete permanently. Confirm irreversible removal of the list, items and purchase history; reusable catalog products survive list deletion.
- Retired products/units remain in existing references/history, not future selections. Offer restoration on attempted recreation of a retired name.
- Declining restoration permits a new list/product/unit with the same name and a new identity. Later restoration requires a different name if now occupied. Do not silently replace old references or history.
- Active list names are household-unique, ignoring capitalization/extra spaces. Concurrent offline creations preserve both lists: first accepted keeps name; other remains accessible as **Name (rename needed)** with an in-app notice to its creator.

## 7. Lists and presentation

- Multiple named lists per household; common and grouped-by-assignee views represent the same underlying data.
- Start in common view; remember each person's choice locally. New active entries go at the bottom in addition order, with deterministic offline ordering to be designed.
- Completed entries go to a collapsible **Purchased** section at the bottom. Grouped view has one shared Purchased section beneath all active buyer groups, including Unassigned.
- Purchased initially collapses and remembers each person's choice on their phone. Preserve buyers and excess indicators. Reopened entries return to the active area.
- Optional entry notes preserve distinct text when duplicate entries merge, e.g. “ripe; small ones”; identical notes appear once. Same-note concurrent-edit behavior is specified in section 5.

## 8. Accounts, households and onboarding

- An account is mandatory; no guest mode or anonymous-account migration. Signup verifies email by code; password is optional and can be added later. Password and email-code sign-in access the same account, not separate identities or mandatory two-factor authentication.
- Keep users signed in across ordinary launches. Offline expiry permits cached shopping and durable local edits; renew or reauthenticate to the same account and recheck membership before syncing.
- Warn on explicit sign-out with pending work, permit sign-out and preserve work privately for the same account. Hide it while signed out and from other accounts. Explicit revocation/logout is distinct from offline expiry.
- Recover passwords by emailed-code sign-in, then set a new password. Password change/reset signs out other devices, keeping current device signed in. Offer Sign out all other devices separately.
- Email changes verify both old and new addresses with fresh codes; preserve account and memberships. Lost access to the old mailbox has no approved recovery flow.
- Account-security actions (email/password/account deletion) and household membership management require internet. Shopping and household display-name edits can operate offline under their rules.
- Signup requests a display name. Household names are unique ignoring capitalization/extra spaces. A joining member with a collision chooses a household-specific name; other household names stay unchanged.
- Users may update names, with attribution showing the updated name linked to the same identity. Block occupied names. Offline races: first accepted change wins; other editor keeps old name and receives in-app request for another.
- After signup, create a household or join via invitation. Creating asks for empty versus common-grocery starter catalog; initialize only the catalog and create an empty renameable **Shopping list**.
- Carry invitation context through signup/sign-in. Show household and inviter with explicit **Join household**; revalidate before acceptance.
- Any member can invite. Shareable invite links expire after 7 days and admit one authenticated person; opening alone does not consume a link. Inviter or current admin may cancel it. Inviter departure/removal cancels pending links, not accepted memberships.
- Only current owner/admin removes other members; everyone may leave. Owner can choose a successor; otherwise transfer alphabetically among remaining members (display-name ordering proposed; collation must be specified).
- Departed/removed members' unfinished assigned entries become Unassigned; preserve history. On reconnect after removal, reject unsynced operations and explain access loss, hide household and block cached access. An offline phone cannot immediately learn removal.
- Last-member departure closes the household, invalidates invitations and permanently deletes remaining lists/history with a warning. Apply this also when the last member deletes their account.
- Account deletion requires a fresh emailed code and final confirmation. Where other household members remain, preserve shared lists/purchases with **Deleted member** attribution, transferring administration as above. Later signup with the same email is a fresh identity without restored memberships or old pending work.

## 9. Implementation acceptance examples

These are required future checks, not claims of executed tests:

1. Restart after offline mutation preserves both local state and queued operation.
2. Two separate purchases of 2 cartons remain 4 after reconnect; repeated delivery of one contributes nothing extra.
3. Demand edits 2 versus 3 resolve to 3 under default policy, not 5; ordinary subsequent reduction remains possible.
4. Purchase correction 1 versus 3 keeps prior quantity pending a choice; concurrent undo contributes zero.
5. Item/list delete versus purchase retains actual purchase; permanent list deletion never resurrects through replay.
6. Catalog reuse creates fresh demand without importing prior purchases; equivalent units merge without changing numbers.
7. Notes edited concurrently preserve both versions; clear/edit preserves edited text with in-app notices.
8. Policy updates do not reopen resolved conflicts or replace existing pending manual rules.
9. Removed member cannot sync or access cached household data once removal is known; switching accounts cannot adopt pending work.
10. Recoverable deletion, permanent deletion, account deletion and same-email re-registration obey their distinct rules.
11. Manual resolution is accepted once for the relevant conflict; stale resolutions do not silently overwrite it.
12. Push failure does not prevent authoritative catch-up or in-app conflict visibility.

Formal conflict coverage must additionally include operation permutations, causal gaps, merge aliases, deletion markers, multi-member conflicts, unit changes and policy versioning before full synchronization is implemented.
