# How the app should behave

Updated for readability: 2026-11-28

This is the current record of the shopping rules we have agreed. The wording has been simplified without changing those decisions. The app is not built yet.

For a short introduction, read [how the app fits together](../architecture/system-overview.md). For unfinished choices, see [what remains to be decided](../decisions/remaining-decisions.md). [PLAN.md](../../PLAN.md) and the [original handover](../reference/CODEX_HANDOVER.md) preserve the history; older proposals there may have been superseded.

## 1. Scope and concepts

### Approved follow-up — 2026-11-25

These later decisions are already approved and apply throughout this document:

| Situation | Agreed rule |
| --- | --- |
| An offline addition might belong to an old or a new shopping request | Ask whether to combine it with the current need or keep it separate |
| A purchase prevented deletion and is later undone | Keep the item or list visible |
| Two custom units are being combined | Confirm they mean the same measure and scale |
| Conflicting unit changes affect purchases | Ask for a choice and keep purchases available for correction; do not convert them silently |
| An old name needed during a conflict is now taken | Keep both records accessible and mark the unresolved name |
| The account's old mailbox is lost | Recover through the email provider; support does not bypass proof of ownership |
| A person loses household access | Do not offer an export of that household's data |
| Sign-in service | Prefer an established hosted provider; the actual provider is not selected |

[The offline guide](../architecture/sync-data-design.md#seven-important-decisions-already-agreed) explains these as decisions D1–D7.

Build an iPhone and Android app that works without internet and shares shopping within households. Support several households, members and named lists. The [system overview](../architecture/system-overview.md) describes the planned technology.

These words refer to different things:

| Concept | Meaning |
| --- | --- |
| Catalog product | A saved household product, such as milk, with its usual unit |
| List entry / shopping need | What is needed on a particular list, with an amount, unit, assigned buyer and optional note |
| Purchase | One buying action, with its own amount and person who bought it |
| Correction/reversal | Fixing or undoing a particular purchase, rather than recording a new purchase |
| Catalog retirement | Hide a product/unit from new selections while keeping it on existing entries and history |
| List-entry deletion | Remove a particular shopping need, which can be restored under the rules below |

Everything created within a household is available to its members unless explicitly restricted. Any current member may edit or remove shared content, including others' purchases. Account and membership settings have separate rules below. Keep track of who added an item, who should buy it, who actually bought it and who later edited it.

Later work includes nearby sharing, supermarket connections, choosing among approved stores, recipes, meal planning, barcode scanning, shopping insights, optional AI, monetization and eventually a household timetable. See the [future roadmap](../backlog/README.md). Similar-name suggestions do not require an external AI service.

## 2. Catalog and adding items

- A household has its own reusable product catalog that never exposes another household's products. Buying/removing a list entry does not remove its catalog product.
- Search existing catalog products before creating a new product. Suggest similar names with **Use existing item** or **Create separately**. Similarity alone never merges products.
- A saved product fills in its usual unit; requested quantity defaults to 1. Override quantity/unit for one entry without changing the catalog default. The default unit changes only through explicit catalog-product editing.
- A new entry defaults to the adding person as assigned buyer. Later ordinary edits preserve assignment unless explicitly changed. Unassigned is supported.
- Exact matching for automatic list-entry merging ignores capitalization and extra spaces, and requires matching names and units. Two omitted units match; omitted versus specified units do not. Do not convert units implicitly.
- Combine matching unfinished entries only when they belong to the same shopping need and list. Requested amounts follow the household's larger/smaller/ask rule; separate purchases add together. Keep both creators and both histories. Different assigned buyers trigger Choose buyer. Keep different notes; show identical notes only once.
- Reusing a product after seeing its previous entry completed or deleted creates a new shopping need with no old purchases counted against it. Reuse the saved product. Each list has its own shopping needs.
- Independently created similar products found when phones synchronize require **Merge** or **Keep separate**, using the standard manual workflow in section 5. Remember Keep separate for that product pair until one is renamed.
- A product-merge decision chooses which existing product name to keep and, if different, the default unit for future additions. Existing entries/purchases keep their units. Different-unit entries remain separate; matching-unit entries combine only when the usual rules allow it.
- Renaming a product from a list edits the shared catalog product and updates active entries throughout the household. Purchase history keeps the product name recorded at purchase time.
- Block a rename to another existing catalog product's name. If an offline rename collision is found when phones synchronize, put back the last accepted name and notify that editor in-app, without discarding other changes. Competing renames use the manual workflow and keep the name both editors previously knew until someone chooses.

## 3. Quantities, purchases and completion

- Unit omitted: whole numbers only. Decimal-capable units permit three decimal places; accept comma or dot as decimal separator. Automatically round excess precision without a prompt unless a positive amount rounds to zero; then ask for a larger amount or smaller unit. Unit-specific integer rules still apply.
- **Purchased** defaults to the remaining amount known on that phone; allow manual quantity adjustment. Offline phones can both purchase the same remaining amount.
- Separate purchases add together, including purchases recorded offline. One person buys 1 and another buys 3: total 4. Sending the same saved purchase again adds nothing. Never use the larger/smaller rule to discard a separate purchase.
- Completed items offer **Add purchase**, requiring an explicitly entered extra quantity. Preserve each purchase and buyer indicator.
- Compare the requested amount with purchases after corrections and undo. Increasing the request or reducing the amount bought puts the item back on the active list if something remains. Lowering the request below the amount bought completes the item and displays excess, such as **2 of 1 bought · 1 extra**.
- Anyone in the household may correct/undo purchases. Keep the original purchase and who made it in history. An undone purchase counts as zero; the current requested amount stays unchanged.
- Competing corrections to one purchase require manual choice; keep the amount both editors previously agreed on meanwhile. If someone undoes that purchase while another person independently corrects it, undo wins; no choice is required. Notify both editors in-app only.
- A unit can change before purchases or after all purchases have been undone. While any purchases still count, a different unit requires a fresh item. History retains original units.
- If a unit change conflicts with an offline purchase in the original unit, keep the original unit and purchase; notify the unit editor in-app.

## 4. Units

**Example:** cartons accept 1 or 2, not 1.5. Litres can accept 1.5. “Litre” and “liter” may be combined if they mean the same thing; grams and kilograms must not be treated as interchangeable numbers.

- Provide a broad shopping-unit catalog. Count/package units use integers (pieces, cartons, bottles, cans, boxes, packs, bags, etc.); weight, volume and length units allow decimals (kg, g, litres, ml, metres, etc.). The detailed starter list is a planning selection, not a fixed user-authored list.
- Members may create household-shared custom units. Whole-number/decimal selection is optional; skipped means whole numbers.
- Block changing a unit to whole-number-only while fractional quantities use it. Do not invalidate existing records.
- Custom units with exactly the same name created offline merge automatically if their whole-number/decimal settings match. Different types require manual resolution; allow decimals until someone chooses, and disallow an integer-only resolution while existing amounts contain fractions.
- Similar-unit creation prompts offer **Use existing unit** or **Create separately**. Use existing adopts that unit's existing quantity rules. Unlike product pairs, do not permanently remember Keep separate to suppress future creation suggestions.
- Similar units discovered after offline creation require **Use one unit** or **Keep separate** with the standard manual workflow. Do not repeatedly reopen the same resolved sync conflict.
- Competing unit renames use the manual workflow, retaining the previous name until resolved. A duplicate rename is blocked, with Use existing unit offered. When invoked while editing an item, this changes only that item, subject to purchase/unit restrictions.
- Explicit unit combination is a separate household-wide action: choose the surviving existing unit, update list entries and product defaults, and retain original unit names in purchase history. An integer-only survivor is unavailable while either unit has fractional quantities recorded.
- Only combine equivalent measures of the same scale, such as litre/liter. Preserve numbers; no g-to-kg conversions or silent reinterpretation.
- Retiring a unit preserves it on existing entries/history but removes it from new selections. An entry created offline with a subsequently retired unit is preserved. Apply the same rule to catalog-product retirement.

## 5. Conflict rules and notifications

A **conflict** means two people made incompatible changes without seeing each other's action. A later change made after seeing the earlier one is normally an intentional edit, not this kind of conflict.

**Example:** changing 2 cartons to 3 and changing 2 cartons to 1 offline creates a conflict. Seeing the shared result of 3 and deliberately changing it to 1 is an ordinary edit.

### Standard manual-choice workflow

Any current household member may resolve. Notify both people involved in-app and by push, not uninvolved household members. The first valid choice accepted by the server stands; offline choices remain pending until accepted. Other resolvers see the result and can make a normal edit afterwards.

| Conflict | Behavior while unresolved / outcome |
| --- | --- |
| Requested quantity | Household rule: larger wins (default), smaller wins, or Ask us to choose. Ask temporarily uses larger amount with Needs a choice |
| Assignment | Unassigned / Choose buyer until manually resolved |
| Product/item name | Last common name / Needs a choice until manually resolved |
| Quantity on one purchase record | Last common agreed purchased quantity until manually resolved |
| Custom-unit name | Previous name until manually resolved |
| Same-name custom-unit quantity type | Decimals permitted temporarily; manual choice must preserve existing fractions |
| Similar product/unit merge | Keep both possibilities until someone chooses to combine or separate them |
| Note edit versus note edit | Preserve both text versions; no manual choice; in-app notice to both editors |
| Note clear versus note edit | Keep edited text; in-app notice to both editors |
| Purchase undo versus correction | Undo wins; in-app notice to both editors |

The note rules above include the final decisions approved after the previous batch save. Notes belong to list entries, not automatically to saved product defaults.

### Household quantity policy

- Any member can change the household's shared larger/smaller/ask setting. Ordinary later edits can intentionally reduce quantities; this is not a permanent maximum.
- Automatic larger/smaller resolution stays quiet. Manual conflicts use the standard workflow.
- A setting change affects new conflicts only, never recalculating resolved quantities. Existing manual conflicts remain pending under their original rule.
- New conflicts found when phones synchronize use the current household rule even if the changes were made under an older setting.
- Competing offline setting changes: last valid setting change accepted by the server wins; notify both editors in-app. This differs from manual choices, where the first accepted answer wins.

### Notification lifecycle

- All deletion conflicts notify both involved people in-app and by push; distinguish deletion from purchase undo.
- Group conflicts arriving together into notifications while retaining individual details and respecting each recipient/channel. In-app-only events must not become push alerts through grouping.
- Unresolved conflicts stay in **Needs attention** until resolved, even after a notice is read.
- Opening informational notices marks them read, retaining notification history. Reading does not resolve a conflict.
- Membership loss, rejected offline renames and offline unit-change rejection have the explicit explanations described elsewhere. Duplicate list/member-name collisions notify only the affected creator/editor in-app.
- Sync must work without notifications. Do not show a change as successfully shared before the server confirms it.

## 6. Deletion, restoration and names

**Example:** Alice deletes milk without knowing Bob bought it offline. Bob's purchase keeps it visible. If Alice first sees that purchase and then deliberately deletes the item, her deletion is allowed.

| Event | Approved result |
| --- | --- |
| Item deletion without knowing about the other person's purchase | Item stays visible; purchase preserved; notify both |
| Item deletion without knowing about the other person's name/quantity edit | Item stays deleted; notify both |
| Deliberate deletion after seeing purchase | Remove item from list; preserve history |
| Restore and delete of an item/list without seeing each other's action | Deletion wins; notify both; subsequent informed restoration still works |
| Whole-list deletion without knowing about the other person's purchase | List remains visible; purchase preserved; notify both |
| Whole-list deletion without knowing about the other person's additions/edits only | List stays deleted; retain those changes for restoration; notify both |
| Permanent list deletion versus pending offline changes | Discard pending changes on reconnection; in-app explanation; never bring the list back |

- Restore an item with its quantities, assignment and purchase history. If its assigned member has left, use Unassigned and keep who originally did what in history.
- Restoring an unfinished item into a matching active shopping need uses the existing merge rules.
- Restoring an already completed item must not change a new matching shopping need's display, quantities, assignment or completion. Do not add a duplicate visible entry or count old purchases toward the current shopping need; keep the history.
- Whole-list deletion moves it to **Deleted lists**, recoverable by any household member until explicit Delete permanently. Confirm irreversible removal of the list, items and purchase history; reusable catalog products survive list deletion.
- Retired products/units remain in existing entries/history, not future selections. Offer restoration on attempted recreation of a retired name.
- Declining restoration permits a new list/product/unit with the same name, stored as a separate record. Later restoration requires a different name if now occupied. Do not silently replace old references or history.
- Active list names are household-unique, ignoring capitalization/extra spaces. Independent offline creations preserve both lists: first accepted keeps name; other remains accessible as **Name (rename needed)** with an in-app notice to its creator.

## 7. Lists and presentation

- Multiple named lists per household; common and views grouped by assigned buyer represent the same underlying data.
- Start in common view; remember each person's choice locally. New active entries go at the bottom in addition order, with a consistent ordering rule for offline additions still to be specified.
- Completed entries go to a collapsible **Purchased** section at the bottom. Grouped view has one shared Purchased section beneath all active buyer groups, including Unassigned.
- Purchased initially collapses and remembers each person's choice on their phone. Preserve buyers and excess indicators. Reopened entries return to the active area.
- Optional entry notes preserve distinct text when duplicate entries merge, e.g. “ripe; small ones”; identical notes appear once. What happens when people edit the same note independently is specified in section 5.

## 8. Accounts, households and onboarding

- An account is mandatory; no guest mode. Signup verifies email by code; password is optional and can be added later. Password and email-code sign-in access the same account, not separate identities or mandatory two-factor authentication.
- Keep users signed in across ordinary launches. If sign-in expires while offline, keep using saved shopping data and save edits. Before sharing again, renew sign-in or sign in to the same account and check household membership.
- Warn on explicit sign-out with pending work, permit sign-out and preserve work privately for the same account. Hide it while signed out and from other accounts. Being explicitly signed out or having access revoked is different from ordinary sign-in expiry while offline.
- Recover passwords by emailed-code sign-in, then set a new password. Password change/reset signs out other devices, keeping current device signed in. Offer Sign out all other devices separately.
- Email changes verify both old and new addresses with fresh codes; preserve account and memberships. If the old mailbox is lost, use the email provider's recovery process. Support does not bypass proof of account ownership (D5).
- Account-security actions (email/password/account deletion) and household membership management require internet. Shopping and household display-name edits can operate offline under their rules.
- Signup requests a display name. Member display names within a household are unique, ignoring capitalization and extra spaces. A joining member with a collision chooses a household-specific name; their display names in other households stay unchanged.
- Users may update names, with old and new activity showing the updated name for the same person. Block occupied names. Offline races: first accepted change wins; other editor keeps old name and receives in-app request for another.
- After signup, create a household or join via invitation. Creating asks for empty versus common-grocery starter catalog; initialize only the catalog and create an empty renameable **Shopping list**.
- Carry invitation context through signup/sign-in. Show household and inviter with explicit **Join household**; check the invitation again before accepting it.
- Any member can invite. Shareable invite links expire after 7 days and admit one authenticated person; opening alone does not consume a link. Inviter or current admin may cancel it. Inviter departure/removal cancels pending links, not accepted memberships.
- Only current owner/admin removes other members; everyone may leave. Owner can choose a successor; otherwise transfer alphabetically among remaining members (using display names is proposed; the exact alphabetical comparison still needs to be specified).
- Departed/removed members' unfinished assigned entries become Unassigned; preserve history. On reconnect after removal, reject unsynced operations and explain access loss, hide household and block cached access. An offline phone cannot immediately learn removal.
- Last-member departure closes the household, invalidates invitations and permanently deletes remaining lists/history with a warning. Apply this also when the last member deletes their account.
- Account deletion requires a fresh emailed code and final confirmation. Where other household members remain, preserve shared lists/purchases showing **Deleted member** as the person, transferring administration as above. Later signup with the same email creates a new account without old memberships or pending work being restored.

## 9. Examples we must test

These are checks for the future app, not tests we have already run:

1. Restart after an offline edit preserves both the visible change and the reminder to send it.
2. Two separate purchases of 2 cartons remain 4 after reconnect; repeated delivery of one contributes nothing extra.
3. Requested amounts changed to 2 versus 3 resolve to 3 under default policy, not 5; ordinary subsequent reduction remains possible.
4. Purchase correction 1 versus 3 keeps prior quantity pending a choice; an independent undo makes that purchase count as zero.
5. Item/list delete versus purchase retains actual purchase; sending an old change again never brings back a permanently deleted list.
6. Catalog reuse creates a new shopping need without counting old purchases; equivalent units merge without changing numbers.
7. Notes edited independently preserve both versions; clear/edit preserves edited text with in-app notices.
8. Policy updates do not reopen resolved conflicts or replace existing pending manual rules.
9. Removed member cannot sync or access cached household data once removal is known; switching accounts cannot adopt pending work.
10. Recoverable deletion, permanent deletion, account deletion and same-email re-registration obey their distinct rules.
11. Manual resolution is accepted once for the relevant conflict; stale resolutions do not silently overwrite it.
12. Push failure does not prevent catching up with the server's confirmed updates or in-app conflict visibility.

Before full synchronization is built, also test different arrival orders, missing earlier changes, merged records, permanent deletions, more than two editors, unit changes and changes to household settings.
