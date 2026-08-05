# PR Review: feature/PP-1910: Order dashboard column clean-up and compaction

**PR:** https://github.com/Proofed/B2BWebserver/pull/2369

**Jira:** https://proofed.atlassian.net/browse/PP-1910

**Status:** Code Review

**Head reviewed:** `d0a15bd0a40e74c6f2a5ffefc9de51600bd0bb3b`

**Base:** `feature/PP-1918-admin-area-navigation` — **not `develop`**. The PR body flags this and asks for a retarget once PP-1918 (#2368) lands. #2368 currently has two high-severity findings of its own, so this PR is stacked on unmerged, changing work.

**Scope:** 36 files, +650 / −489, 5 commits. Consolidates PP-1910–PP-1914. Display-only; no API changes.

**Method:** multi-agent review at high effort (4 finder angles × independent adversarial verification of every candidate location, 34 agents). 10 findings survived verification. `StatusSelectFilter/hooks.ts` was re-read in full at head to check the two filter findings directly.

---

## What this means for users (non-technical summary)

Two things an admin would actually notice, both caused by moving something out of its own column into a shared one:

1. **Delivery-speed icons vanish on completed orders.** The rocket / plane / adjust glyph used to sit in its own column, so it showed on every row. It now lives inside the Time Remaining column — and that column is deliberately hidden once you filter by Complete or Cancelled. So on the Closed Orders view, no order shows its delivery speed at all. Rapid, Expedited, Custom and Regular closed orders look identical.
2. **A filter that used to be locked in batch views is now editable.** Inside a batch or order-group drill-down, Current Job used to display as read-only text — you could see it but not change it. After merging it into the Status filter that lock was lost, so a user can now apply a job filter there and typically end up staring at an empty list, with no obvious reason why.

Everything else is smaller: a name may render as "Ella f" instead of "Ella F" if the source data is lowercase, one filter interaction can lose a selection with no undo, and the rest is tidy-up left behind by the deleted columns.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1.1 Remove the Current job column | `currentJob` column deleted from `tableColumns.tsx` | ✅ Addressed |
| 1.2 Remaining columns reflow into freed space | Governed by PP-1915, out of scope here | ✅ N/A |
| 2.1 Current job filter relocated under the Status filter | New `current-job` group prepended to `StatusSelectFilter.groups` | ✅ Addressed |
| 2.2 Filter and all its states unchanged — relocation only | ❌ Not held: the `isBatchFilter` / `showLabelOnly` read-only state was lost in the move (Issue 2) | ❌ Missing |
| 2.3.1 Current job multi-selectable with other job statuses | `onChangeJob` toggles within `draftJob`, no exclusivity between job values | ✅ Addressed |
| 2.3.2 Reciprocal deselection between order status and job/current-job | Both directions implemented — `onChangeStatus` clears `draftJob` on a finished status; `onChangeJob` strips finished statuses | ⚠️ Partial — correct behaviour, but implemented as a side effect inside a state updater with no restore path (Issue 3) |
| 2.4 Overlay max height 495px, scrolls beyond | New `contentMaxHeight` prop on `FilterDropdown` + custom scrollbar | ✅ Addressed |
| 3.1 Remove the Delivery speed column | `DeliverySpeedCell` column deleted | ✅ Addressed |
| 3.2 Delivery-speed glyph shown in Time Remaining for each applicable order | Rendered inside `InlineOrderDeadline` | ⚠️ Partial — not shown for Complete/Canceled orders or anywhere in the Closed Orders view (Issue 1) |
| 3.3 No icon for standard delivery speed | `DELIVERY_SPEED_ICON_MAP` lookup yields nothing for Regular | ✅ Addressed |
| 3.4 Calendar icon left of the delivery-speed icon during inline edit | Ordering handled in `InlineOrderDeadline` | ✅ Addressed |
| 4.1 Team names as `First L` | New `formatNameWithLastInitial` shared util | ⚠️ Partial — initial is not uppercased, so lowercase source data renders "Ella f", not the ticket's "Ella F" (Issue 5) |
| 4.2 Ellipsis when still too long | Existing cell truncation retained | ✅ Addressed |
| 4.3 Full name on hover | `title` attribute carries the full name | ✅ Addressed |
| 5.1 Rename "Content size" → "Words" | Header renamed | ✅ Addressed |
| 5.2 Value only, no "words" unit | Unit suffix dropped in `OrderSizeCell` | ✅ Addressed |
| 5.3 Blank cell when no word count | Empty render for missing counts | ✅ Addressed |
| Validation rule 1: no row loses current job info | Current job still shown via the Status column | ✅ Addressed |
| Validation rule 2: order status and job statuses never both active | Enforced in both directions in `onChangeStatus` / `onChangeJob` | ✅ Addressed |

**Beyond Jira scope:** a null-dereference fix in `FilterDropdown`'s scroll-shadow handler (declared in the PR body, and genuinely needed once the merged overlay became scrollable) and three new `-2` icon variants.

---

## Architecture Analysis

The approach is sound and the diff is smaller than the ticket count suggests: two columns deleted, one filter folded into another, two display formatters changed. Reusing `CurrentJobFilter`'s existing `getStatusOptions` / `AVAILABLE_JOB_OPTIONS` for the relocated group is the right instinct — the option-building logic is not re-implemented, only re-hosted.

The weakness is that both structural moves treat the deleted thing as *just markup*, when each carried behaviour the new host does not have:

- The `DeliverySpeedCell` column was **unconditional** — no status guard, not in `HIDDEN_COLUMNS.CLOSED_ORDERS`. Its new host, `InlineOrderDeadline`, sits behind two gates (`FINISHED_ORDER_STATUSES` early-return, and the `deadline` column id being hidden for closed orders). The glyph silently inherited both.
- The `currentJob` column passed `isBatchFilter` down to `FilterDropdown` as `showLabelOnly`. `StatusSelectFilter` has no such prop, so the read-only state for batch and order-group views was dropped rather than relocated — which is exactly what requirement 2.2 ("the filter and all of its states already exist and must remain unchanged") rules out.

Secondary theme: `useStatusSelectFilter` now re-implements the draft/clear/apply/dirty pattern that `useCurrentJobFilter` still runs for the History table, and the two `onChange*` handlers use inconsistent state-update styles (one mutates sibling state inside an updater, the other outside).

---

## Issues Found

### 1. The delivery-speed glyph disappears for finished orders and across the whole Closed Orders view

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/cells/DeadlineCell/index.tsx:84]**

> **In plain terms:** The little rocket / plane / adjust icon that tells you how fast an order is being delivered used to have its own column, so it appeared on every row. It has been moved inside the Time Remaining column — but that column is intentionally hidden for completed and cancelled orders. The result: on the Closed Orders view nobody can tell a Rapid order from a Regular one any more.

**Function/Class:** DeadlineCell → DeadlineCellContent → InlineOrderDeadline

**Severity:** high

**Steps to reproduce:**

1. Log in as a Proofed Admin and open the order management dashboard.
2. Note that active orders with Rapid / Expedited / Custom delivery show a green icon in the Time Remaining column. ✅ working as intended.
3. Open the **Status** filter, select **Complete**, and Apply.
4. **Expected:** each closed row still shows its delivery-speed icon, as it did before this PR (the old Delivery speed column was always visible).
5. **Actual:** the Time Remaining column is hidden for the closed-orders layout, so **no** row shows a delivery-speed icon anywhere on the page.
6. Also reproducible without filtering: scroll the unfiltered list to any Complete or Canceled row — that row shows no icon either.

**Problem:** The glyph's only render path is now `InlineOrderDeadline`, reached exclusively through `DeadlineCellContent`, which opens with `if (FINISHED_ORDER_STATUSES.includes(orderStatus)) return null;` (partials/DeadlineCellContent/index.tsx:59). That component lives in the column with id `"deadline"`, which `HIDDEN_COLUMNS.CLOSED_ORDERS` (consts.tsx:133) removes entirely whenever the status filter includes a closed status. The deleted `DeliverySpeedCell` column had neither gate.

**Impact:** Requirement 3.2 asks for the glyph "for each applicable order" and says nothing about dropping it for finished ones; before this PR every closed row carried it. Ops staff reconciling completed rapid orders lose that signal entirely.

**Fix:** Decide with the PO whether closed orders should keep the indicator. If yes, render the glyph in `DeadlineCellContent` *before* the finished-status early return, and keep the `deadline` column out of `HIDDEN_COLUMNS.CLOSED_ORDERS` (or give the closed layout a narrow speed-only cell):

```typescript
// DeadlineCellContent/index.tsx — glyph is status-independent
if (FINISHED_ORDER_STATUSES.includes(orderStatus)) {
  return deliverySpeedIcon ?? null;
}
```

If the loss is intended, say so in the ticket — it is a visible behaviour change that the requirements do not cover.

### 2. The Current Job filter is no longer read-only in batch and order-group views

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/tableColumns.tsx:229]**

> **In plain terms:** When you drill into a batch or an order group, the Current Job filter used to be locked — shown as plain text you could read but not change, because filtering by job inside a batch doesn't make sense. Folding it into the Status filter lost that lock, so a user can now apply it there and end up with an empty list and no clue why.

**Function/Class:** tableColumns — Status column header filter

**Severity:** high

**Steps to reproduce:**

1. Log in as a Proofed Admin and open the order dashboard.
2. Open a **batch** (or an order group) so the table shows only that batch's orders.
3. **Before this PR:** the Current Job filter in the header showed as inert text — not clickable.
4. **Now:** click the **Status** column header, choose **QA** under the "Current Job" group, and press **Apply**.
5. **Expected:** the job filter is not offered as an editable control in this view (requirement 2.2 says the filter's existing states must be preserved).
6. **Actual:** the filter applies, and the batch listing typically empties — indistinguishable from "this batch has no orders".

**Problem:** The deleted `currentJob` column passed `isBatchFilter: isBatchOrOrderGroupFilter` into `CurrentJobSelect`, which forwarded it to `FilterDropdown` as `showLabelOnly={!!isBatchFilter}`; in a batch view (`batchName` set) or an order-group view (`orderGroupId != null && !== 0`) the filter rendered as inert text (FilterDropdown/index.tsx:90). The replacement `<StatusSelectFilter>` receives `currentJobFilter` / `setCurrentJobFilter` but no `isBatchFilter`, and `StatusSelectFilter` never passes `showLabelOnly` at all.

**Impact:** Requirement 2.2 explicitly scopes this ticket to relocation with all existing states unchanged. This is the one place the PR changes behaviour the ticket said to preserve, and the symptom (an empty batch) reads as missing data rather than an active filter.

**Fix:** Thread the flag through and honour it:

```typescript
// tableColumns.tsx
<StatusSelectFilter
  {...{ currentJobFilter, setCurrentJobFilter, … }}
  isBatchFilter={isBatchOrOrderGroupFilter}
/>

// StatusSelectFilter/index.tsx
<FilterDropdown showLabelOnly={!!isBatchFilter} … />
```

If the intent is that the *status* half stays interactive while only the job half locks, that needs a per-group control in `FilterDropdown` and a note in the ticket.

### 3. `setDraftJob` is called inside the `setDraftStatus` updater, and the cleared job filter has no restore path

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/StatusSelectFilter/hooks.ts:60]**

> **In plain terms:** Picking "Complete" is meant to clear any Current Job selection — that rule is correct and in the ticket. The gap is that changing your mind doesn't bring it back. If you tick Complete, untick it again, then press Apply, the job filter you had applied before opening the dropdown is gone. You can see it happen in the panel, but there's no undo short of re-selecting it.

**Function/Class:** useStatusSelectFilter — `onChangeStatus`

**Severity:** medium

**Steps to reproduce:**

1. On the order dashboard, open the **Status** filter, select **QA** under Current Job, press **Apply**. The Status pill now shows as active.
2. Re-open the **Status** filter. QA is ticked, as expected.
3. Click **Complete**. The QA tick clears — this part is correct per requirement 2.3.2.
4. Change your mind: click **Complete** again to de-select it. The panel now has nothing selected.
5. Press **Apply**.
6. **Expected:** either the QA filter is restored when Complete is de-selected, or Apply is disabled because nothing meaningful changed.
7. **Actual:** Apply is enabled (the empty draft differs from the applied QA filter), so the QA filter is committed away and the table reloads with no filters at all.

**Problem:** Two separate concerns in the same three lines:

```typescript
setDraftStatus((prev) => {
  …
  if (doArraysIntersect(next, FINISHED_STATUSES)) {
    setDraftJob(DEFAULT_JOB_FILTER_STATE);   // side effect inside an updater
  }

  return next;
});
```

(a) State updaters must be pure — React may invoke them more than once (it does in StrictMode dev), and nesting a setter inside one is a documented anti-pattern. The sibling handler `onChangeJob` performs the mirror-image reciprocal clear *outside* its updater, so the codebase already contains the correct pattern two lines below.

(b) The old code cleared the job filter only at apply time and only if the committed status set still intersected `FINISHED_STATUSES`. Now the clear happens in the draft and is never reversed when the finished status is de-selected, while `onApplyClick` commits `setCurrentJobFilter(draftJob)` unconditionally.

**Impact:** (b) is visible in the overlay — the tick does clear, so the user is not kept in the dark — but there is no way back, and losing an applied filter to a mis-click with no undo is a real annoyance on a dashboard people filter all day. (a) is latent today; it will surface as double-invocation weirdness the moment StrictMode or a concurrent-rendering upgrade lands.

**Fix:** Move the reciprocal clear out of the updater to match `onChangeJob`, and consider restoring the previous job selection when the finished status is de-selected:

```typescript
const onChangeStatus = useCallback((status: TFilterableOrderStatus) => {
  const next = computeNextStatus(draftStatus, status);

  setDraftStatus(next);

  if (doArraysIntersect(next, FINISHED_STATUSES)) {
    setDraftJob(DEFAULT_JOB_FILTER_STATE);
  }
}, [draftStatus]);
```

### 4. `onChangeJob` decides add-vs-remove from a closure while updating with a functional updater

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/StatusSelectFilter/hooks.ts:70]**

> **In plain terms:** The code that ticks and unticks a job option checks one copy of the current selection but writes to another. Today those two copies agree, so nothing breaks — but if they ever disagree, clicking a ticked option would add it a second time instead of removing it, leaving a row you can't untick and an Apply button that never settles.

**Function/Class:** useStatusSelectFilter — `onChangeJob`

**Severity:** low

**Steps to reproduce:** Not reproducible by hand on the current code — the `useCallback` dependency keeps the two copies in sync. It becomes reachable if a job click is processed in the same batch as the status-driven clear in Issue 3a. The symptom to watch for in QA: a Current Job row that stays ticked after you click it to unselect, with **Apply** never returning to its disabled state.

**Problem:** `const isRemoving = draftJob.includes(jobType)` reads the render-time closure while `setDraftJob((prev) => …)` operates on the latest state — two different sources of truth in one handler.

**Impact:** If a handler from an older render ever executes, an already-selected job takes the `[...prev, jobType]` branch. `draftJob` becomes `["QA", "QA"]`, the row stays ticked, `isEqual(sortBy(draftJob), sortBy(currentJobFilter))` never matches so Apply stays permanently enabled, and Apply commits the duplicate.

**Fix:** Derive the branch inside the updater, so there is one source of truth:

```typescript
setDraftJob((prev) =>
  prev.includes(jobType)
    ? prev.filter((item) => item !== jobType)
    : [...prev, jobType]
);
```

The `setDraftStatus` call that follows still needs `isRemoving`; compute it once from `draftJob` for that purpose only, or lift both into a single reducer.

### 5. `formatNameWithLastInitial` does not uppercase the initial

**[File: packages/shared/utils/formatNameWithLastInitial.ts:5]**

> **In plain terms:** Team names are meant to show as "Ella F". If the surname is stored in lowercase in the database, the column shows "Ella f" instead — a stray lowercase letter that looks like a bug to whoever is reading the dashboard.

**Function/Class:** formatNameWithLastInitial

**Severity:** low

**Steps to reproduce:**

1. Find (or create) a team member whose surname is stored lowercase, e.g. `{firstname: "Ella", lastname: "fitzgerald"}`.
2. Assign them to an order and open the dashboard.
3. **Expected:** the Team column reads "Ella F" (the ticket's own example, req 4.1).
4. **Actual:** it reads "Ella f", while the hover tooltip shows "Ella fitzgerald".

**Problem:** `lastName.charAt(0)` is used verbatim. Backend casing is not normalised anywhere, and every other initial-derivation in the repo uppercases — `packages/wysiwyg/.../CommentHeader/index.tsx:17` uses `lastName?.charAt(0).toUpperCase()`, and `apps/creative-portal/utils/capitalizeFirstLetter.ts` exists for exactly this.

**Impact:** Inconsistent casing across rows in the same column reads as a data bug to the user.

**Fix:**

```typescript
const initial = Array.from(lastName)[0]?.toUpperCase() ?? "";
```

`Array.from` also avoids splitting a surrogate pair into a replacement glyph for non-BMP first characters.

### 6. Deleted `currentJob` column orphans its CSS rule and skeleton entry

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/tableColumns.tsx:174]**

> **In plain terms:** The column was removed, but two leftovers still reference it — a styling rule that now applies to nothing, and the grey "loading" placeholder that still draws a bar where the deleted column used to be. Nothing breaks; it's clutter that will confuse the next person to touch this table.

**Function/Class:** tableColumns / OrderRowSkeleton

**Severity:** low

**How to spot it:** Watch the dashboard's loading state closely — the skeleton row still renders a placeholder for a column that no longer exists.

**Problem:** `tableColumns.tsx` was the only place setting `className: "column-current-job"`, so `TableWithFilters/styles.ts:170` (`&.column-content, &.column-current-job { color: primary }`) now matches nothing. `partials/OrderRowSkeleton/index.tsx` still maps `currentJob: <BasicSkeleton width="6rem" />` under a comment reading "Keep in sync with tableColumns.tsx column ids".

**Impact:** Dead CSS plus a comment that is now false — the next person editing the skeleton map has to re-derive which ids are real.

**Fix:** Drop `&.column-current-job` from the selector and remove the `currentJob` key from the skeleton map.

### 7. Draft/apply machinery duplicated with the still-live `useCurrentJobFilter`

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/StatusSelectFilter/hooks.ts:122]**

> **In plain terms:** The "pick options, then press Apply" behaviour now exists twice — once for the dashboard's merged filter and once for the History table's own Current Job filter, which is still in use. Any future fix to how Apply or Clear behaves has to be made in both places, or the two screens will start behaving differently.

**Function/Class:** useStatusSelectFilter vs useCurrentJobFilter

**Severity:** low

**How to spot it:** Compare the Status filter on the order dashboard with the Current Job filter on the History table — identical draft/Apply/Clear behaviour, two separate implementations.

**Problem:** `useCurrentJobFilter` (CurrentJobFilter/hooks.ts) is still mounted by `components/molecules/HistoryTable/consts.tsx:32` and implements the identical sequence: draft state seeded from the committed filter, `onClearAllClick`, `onApplyClick`, `onClickOutsideHandler` restoring the draft, `isEqual(sortBy(draft), sortBy(committed))` for `disableApplyButton`. The PR re-types it for `draftJob` rather than factoring it out.

**Impact:** Neither copy resets its draft when the committed filter changes externally (e.g. a "clear all filters" action elsewhere on the page); whoever fixes that has to find both, and the Order dashboard and History table will drift.

**Fix:** Extract a `useDraftFilter<T>(committed, setCommitted)` returning `{ draft, setDraft, onApplyClick, onClearAllClick, onRestore, isDirty }` and have both hooks consume it. Per CLAUDE.md's reuse-first convention, that belongs in `packages/shared/hooks`.

### 8. A duplicate `consts` module still lists the deleted `currentJob` column

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/consts.tsx:133]**

> **In plain terms:** There are two config files with the same name holding the same settings. This PR updated one of them; the other still names the deleted column. Today the app happens to read the updated one, so nothing is wrong — but it's a trap set for whoever tidies up next.

**Function/Class:** HIDDEN_COLUMNS

**Severity:** low

**How to spot it:** Not user-visible. `TableWithFilters/consts.tsx` and `TableWithFilters/consts/index.ts` both export `HIDDEN_COLUMNS`; only the first was updated.

**Problem:** `import { HIDDEN_COLUMNS } from "./consts"` in `table.tsx:29` resolves to `consts.tsx` under extension-first resolution, so this PR's edit does take effect — but the `consts/index.ts` copy still reads `CLOSED_ORDERS: ["currentJob", "deadline"]`.

**Impact:** Any change to resolution order, or deleting the `.tsx` file during a future cleanup, silently swaps in a config that hides a column id that no longer exists.

**Fix:** Delete the unused `consts/index.ts` (or fold both into one module) rather than editing one of two copies.

### 9. `deliverySpeed as DeliveryOption` casts an API `string` through an enum at the call site

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/cells/DeadlineCell/index.tsx:85]**

> **In plain terms:** The code assumes the delivery speed coming back from the server is always one of four known values. If the backend ever sends a fifth one — a new tier, or the same word in different casing — the icon just doesn't appear, and the order looks like a standard-speed order. No error, no warning, just a wrong-looking row.

**Function/Class:** DeadlineCell

**Severity:** low

**Steps to reproduce:**

1. Have the API return an order whose `deliverySpeed` is not exactly one of the four known values (e.g. a new tier, or `"rapid"` instead of `"Rapid"`).
2. Open the dashboard and find that order.
3. **Expected:** either the correct icon, or a visible signal that the value was not understood.
4. **Actual:** no icon at all — identical to a Regular order, which requirement 3.3 defines as a meaningful state.

**Problem:** `api/orders/types.ts:138` declares `deliverySpeed: string`, while `DELIVERY_SPEED_ICON_MAP` is a `Record<DeliveryOption, …>` keyed by exactly four values. The cast tells TypeScript to trust a value it has no evidence for.

**Impact:** A silent fallback into the "no special delivery speed" state is the worst possible failure mode here, because that state carries meaning.

**Fix:** Narrow once where the order is parsed, or make the lookup total:

```typescript
const getDeliverySpeedOptionIcon = (speed: string) =>
  DELIVERY_SPEED_ICON_MAP[speed as DeliveryOption] ?? null;
```

### 10. `IconAdjust2` orphans `IconAdjust`, with no rule for the `-2` variant set

**[File: packages/shared/assets/svg/icons/index.ts:68]**

> **In plain terms:** A new version of the "adjust" icon was added, and the old one is now used nowhere — but it's still shipped and still offered to developers alongside the new one, with nothing to say which is correct. The next person has a 50/50 chance of picking the wrong icon.

**Function/Class:** shared icon barrel

**Severity:** low

**How to spot it:** Not user-visible. Search the repo for `IconAdjust` — zero consumers remain, yet it stays exported.

**Problem:** Once `consts.tsx` swaps to `IconAdjust2`, `IconAdjust` (`adjust-icon.svg`) has zero consumers repo-wide — customer-portal's ShippingOptions uses only `IconPlane` / `IconRocket`. It stays exported from the barrel, and autocomplete now offers both names with nothing to distinguish them.

**Impact:** Every consumer of `@proofed/shared/assets/svg/icons` still carries the orphan. The file already carries `// TODO - Replace all instances and remove` markers for precisely this situation.

**Fix:** Delete `adjust-icon.svg` and its export now that nothing uses it, or add the same TODO marker to the three new `-2` icons so the duplication is tracked.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⚠️ Not run | PR head `d0a15bd` cannot be fetched in this environment — `git fetch` over HTTPS has no stored credentials, and the available SSH key (`Nandeep2750`) has no access to `Proofed/B2BWebserver`. Unchanged from the #2368 review. |
| `npx turbo run typecheck` | ⚠️ Not run | Same. |
| `npx turbo run lint` | ⚠️ Not run | Same. |
| `npx turbo run build` | ⚠️ Not run | Same. |
| Mergeable state | ✅ `clean` | Against its current base — but that base is `feature/PP-1918-admin-area-navigation`, not `develop`. |

**To unblock:** run `git fetch origin feature/PP-1910-dashboard-column-cleanup` in your own shell (prefix with `!` in this session) and the suite can be run against a detached worktree at the PR head.

**Author's reported state (unverified here):** "All existing tests pass" is ticked, with typecheck and lint described as clean (`tsc --noEmit`, `eslint --max-warnings 0`) and every changed/new test file plus the full `@proofed/shared` suite verified. Note this is a stronger claim than #2368's, where the author explicitly could not complete the run — but #2369 sits on top of #2368's branch, which has two test suites (`organisms/Header`, `organisms/SideNav`) that do not execute at all, so a clean run here does not mean the merged result is green.

---

## Tests

- ✅ A unit test accompanies the new `formatNameWithLastInitial` util.
- ❌ No test covers the delivery-speed glyph's visibility for finished orders or in the closed-orders layout (Issue 1) — the exact behaviour the relocation changed.
- ❌ No test covers `StatusSelectFilter`'s reciprocal deselection, the draft-clear/no-restore path (Issue 3), or the batch/order-group read-only state (Issue 2). All three are pure hook logic and are cheap to test directly against `useStatusSelectFilter`.
- ⚠️ The `FilterDropdown` null-dereference fix has no regression test, despite being a real crash the author hit during development.
- ⚠️ Full suite not executed by this review — see Validation Checks.

### Suggested manual QA script

Run these against the PR branch before merge — each maps to an issue above:

1. **Closed orders + delivery speed** — filter by Complete; confirm whether delivery-speed icons should appear (Issue 1, needs a PO decision).
2. **Batch drill-down** — open a batch, confirm the Current Job group is not editable (Issue 2).
3. **Filter undo** — apply Current Job = QA, then tick and untick Complete, then Apply; confirm QA survives (Issue 3).
4. **Job toggle** — tick and untick several Current Job options in one dropdown session; confirm Apply returns to disabled when the selection matches what was applied (Issue 4).
5. **Team names** — check a member whose surname is stored lowercase renders "Ella F" (Issue 5).
6. **Overlay height** — with all three groups expanded, confirm the overlay caps at 495px and scrolls (req 2.4).
7. **Words column** — confirm the header reads "Words", values have no unit, and orders with no word count show a blank cell (req 5.1–5.3).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ Two high-severity behaviour regressions, both from moved code losing its host's guards |
| Regression risk | ⚠️ Medium — contained to the order dashboard, but the batch/order-group filter change affects a workflow the ticket intended to leave untouched |
| Tests | ⚠️ New util covered; the two regressions and the filter's reciprocal logic are not |
| Code quality | ✅ Good overall — small, focused diff; reuses existing option builders; a genuine latent bug found and fixed along the way |
| Validation suite | ⚠️ Not run here (PR head unfetchable); author reports test/typecheck/lint clean |
| Mergeable state | ⚠️ `clean` against `feature/PP-1918-admin-area-navigation` — must be retargeted to `develop` after #2368 merges, then re-verified |

---

## Recommendation

**Request changes**

1. **Restore the read-only state for batch and order-group views (Issue 2).** Requirement 2.2 scopes this ticket to relocation only; this is the one place the PR changes behaviour the ticket said to preserve.
2. **Decide and fix the delivery-speed glyph on finished/closed orders (Issue 1).** Either render it before the `FINISHED_ORDER_STATUSES` early return and keep it visible in the closed-orders layout, or get the loss confirmed as intended and recorded in the ticket.
3. **Move the reciprocal clear out of the `setDraftStatus` updater (Issue 3)** to match `onChangeJob`, and decide whether de-selecting the finished status should restore the job selection.
4. **Derive add-vs-remove inside the `setDraftJob` updater (Issue 4)** so the handler has one source of truth.
5. **Uppercase the last-name initial (Issue 5)** so the output matches the ticket's "Ella F".
6. **Clean up what the deleted columns left behind (Issues 6, 8, 10):** the `.column-current-job` CSS rule, the `currentJob` skeleton entry, the duplicate `consts/index.ts`, and the orphaned `IconAdjust`.
7. **Add hook-level tests for `useStatusSelectFilter`** covering reciprocal deselection, the draft-clear path, and Apply's commit semantics — the merged filter is now the single control for two independent filter values and has no coverage.
8. **Retarget to `develop` once #2368 merges,** then re-run the full validation suite against the retargeted head. Consider factoring the shared draft-filter hook (Issue 7) at that point rather than in this PR.
