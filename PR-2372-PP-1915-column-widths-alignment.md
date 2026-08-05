# PR Review: feature/PP-1915: Order dashboard column widths, alignment & polish

**PR:** https://github.com/Proofed/B2BWebserver/pull/2372

**Jira:** https://proofed.atlassian.net/browse/PP-1915

**Status:** Code Review

**Head reviewed:** `13cd91e8cfdc2ac010baccb4d9f8e33569754e84`

**Base:** `feature/PP-1910-dashboard-column-cleanup` — **not `develop`**. This is the third PR in a stack: #2368 (PP-1918) ← #2369 (PP-1910) ← **#2372 (PP-1915)**. Both parents are open and both have open findings.

**Scope:** 14 files, +338 / −37, 4 commits.

**Method:** multi-agent review at high effort — 4 finder lenses fanned out, then an independent adversarial verifier per distinct `file:line`. 31 agents, 27 verified findings collapsing to 10 distinct defects. (A first run was discarded: all four finders died on a session limit before reading anything, so its "no findings" result was meaningless.)

---

## What this means for users (non-technical summary)

1. **An admin can accidentally select every order in the list and not know it.** While the table is still loading, the "select all" checkbox at the top of the list is now clickable. Clicking it selects nothing visible — but the moment the orders finish loading, *all* of them become selected, the bulk-actions toolbar appears, and the selection survives a page refresh. The next bulk action (skip job, add job) is applied to every order in the result set. This is the one finding that can change data rather than just pixels.
2. **The Status column now sits on the wrong side.** The rule that pushed status chips to the right edge was deleted, so they render on the left of their column with a gap of empty space at the table's right edge — the opposite of what the ticket asks for. A test was added that claims the right-alignment works, so CI stays green while the screen is wrong.
3. **Two column headings still don't line up with their values.** "Customer" now over-corrects inside a batch or order group and sits 10px too far left; "Status" never got the correction at all, so it sits 10px too far right. Fixing this misalignment was the point of the ticket.
4. **A filter heading on the Partners page changed appearance,** even though this ticket only covers the order dashboard — the grey pill now stays on permanently once a filter is applied, instead of appearing on hover.
5. **Smaller things:** service names are cut off with "…" even when there's plenty of room, the table visibly snaps narrower when you scroll to the end of the list, and the list can jump sideways again when the order side panel opens.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1. At ≥1280px, columns use the Figma widths | Explicit widths in `tableColumns.tsx` / `consts.tsx` | ⚠️ Partial — the loading skeletons still use the *old* wider values, so the real widths only apply once skeletons unmount (Issue 9) |
| 2. Left group (checkbox → Services) left-aligned as the table scales | Flexible spacer column via `utils/insertSpacerColumn` absorbs slack | ✅ Addressed |
| 3. Right group (Time Remaining → Status) right-aligned as the table scales | Spacer anchors the group right — but the Status **content** rule was deleted | ❌ Missing — Status content falls back to `text-align: left` (Issue 2) |
| 4. Accommodate the reduced content width from the upcoming sidebar nav | `overflow: auto` restored so the table scrolls instead of clipping when narrowed | ✅ Addressed |
| Testing note 4: no column misaligns against its group | Customer / Time Remaining given `-10px` compensation | ❌ Missing — Customer over-shifts in batch view (Issue 3), Status never compensated (Issue 5) |
| Testing note 5: header row stays pinned while scrolling | Unchanged by this PR | ✅ N/A |

**Beyond Jira scope (scope creep):**

- **`CLAUDE.md` pre-commit policy rewritten** (+24/−8) to run only affected tests instead of the full suite — a repo-wide process change bundled into a UI PR, and used in this PR's own checklist to justify not running the creative suite (Issue 10).
- **Shared `packages/shared/components/molecules/Sidebar`** top offset changed — affects both apps (Issue 4).
- **`FilterHeadingTrigger` active pill** — reaches every `FilterDropdown` consumer, including the Partners page (Issue 7).
- **`scrollbar-gutter: stable` removed** from `TableWrapper` (Issue 8) — declared in the PR body, but it reverts the fix from PP-1806 (#2247).

---

## Architecture Analysis

The core idea is good and well-executed: rather than fighting the table's auto layout with per-column percentages, the PR gives every real column its Figma width and injects a single flexible **spacer column** in the middle (`utils/insertSpacerColumn`, with its own unit test). The left group then anchors left, the right group anchors right, and all slack lands in the middle — which is exactly what requirements 2 and 3 describe, and it degrades sensibly when the side panel narrows the table.

Three weaknesses run through the findings:

1. **Alignment was fixed per-header rather than systematically.** Three headers use the padded pill trigger; two got a `-10px` compensation, one didn't, and the one compensation that did land assumes a pill that isn't rendered in batch/order-group view. A single wrapper (or making the trigger's padding non-visual) would have covered all three and been immune to the `showLabelOnly` branch.
2. **A CSS rule was deleted while its contract was documented as still true.** The `.column-status` rule went away, but `tableColumns.tsx` still sets `className: "column-status"`, a *new* comment says the content is "right-aligned via the `.column-status` rule in styles.ts", and a *new* test asserts that behaviour. The test passes because it asserts the class name, not the computed style — so the regression is invisible to CI and actively documented as working.
3. **Presentation changes reach outside the ticket's surface.** A shared-package offset, a shared trigger style, and a repo-wide testing policy all ride along in a PR whose stated scope is "order dashboard column widths".

Worth calling out positively: `insertSpacerColumn` is a pure, tested helper rather than inline logic in the column builder, which is the right shape for this.

---

## Issues Found

### 1. Select-all is clickable while loading and latches, silently selecting every order

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/table.tsx:293]**

> **In plain terms:** While the order list is still loading, the "select all" tick box at the top is now live. Ticking it appears to do nothing — but when the orders finish loading, every one of them is selected, the bulk-actions bar appears, and the selection is remembered even if the page is refreshed. The next bulk action an admin runs then hits every order in the list rather than the ones they meant.

**Function/Class:** `showExtraColumns` in `table.tsx`; `toggleAllRowsSelected` in `hooks/useTableSelection.ts`

**Severity:** Blocker

**Confidence:** high

**Steps to reproduce:**

1. Log in as an admin on the order dashboard with some filters applied.
2. Change the month or a filter so a new fetch starts (`isLoading` true, `orders` empty).
3. While the loader is showing in the table body, click the **select-all** checkbox in the header.
4. **Expected:** the checkbox is not offered while there is no data (as before this PR, where `dataPaginated.length === 0` suppressed it) — or, if offered, clicking it selects nothing and can be un-clicked.
5. **Actual:** nothing appears selected, and clicking again does not undo it — the state cannot be cleared while loading.
6. Wait for the fetch to resolve. **Actual:** every order in the result set is selected and the BulkToolbar appears. Refresh the page — the selection is still there (persisted to sessionStorage).

**Problem:** `showExtraColumns` was widened to `dataPaginated.length > 0 || isLoading`, so the selection header renders over an empty dataset. `toggleAllRowsSelected()` guards on `data.length > 0 && …`; with `data === []` that guard is false, so it takes the else branch and sets `{ isAllSelected: true, selectedRowIds: {} }`. `isHeaderFullyChecked` requires `orderRows.length > 0`, so the checkbox never renders as checked and the user gets no feedback — and a second click re-runs the identical branch rather than toggling off.

**Evidence:** `table.tsx:291-295` (the widened `showExtraColumns` and the `SelectionHeader` render), `hooks/useTableSelection.ts` (`toggleAllRowsSelected` guard and the `isAllSelected` cleanup effect that calls `setSelectedRowIds(Object.fromEntries(data.map(...)))` once data arrives).

**Impact:** A bulk action (skip job, add job) applied to every order in the result set instead of an intended subset, with no visible cue at the moment of the mistake. This is a data-integrity risk, not a layout one — it is the reason this PR is a Request-changes rather than an Approve-with-suggestions.

**Fix:** Keep the columns reserved for layout, but do not render an interactive selection control without data. Split the two concerns:

```typescript
// reserve the columns during loading (the layout-shift fix) …
const reserveExtraColumns = dataPaginated.length > 0 || isLoading;
// … but only offer selection when there is something to select
const canSelectRows = dataPaginated.length > 0;
```

Render the header cell when `reserveExtraColumns` and the `SelectionHeader` only when `canSelectRows`. Independently, make `toggleAllRowsSelected()` a no-op when `data.length === 0` rather than falling into the latching else branch — that guard is worth having regardless of this caller.

### 2. The deleted `.column-status` rule leaves Status left-aligned, with a new test asserting otherwise

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/styles.ts:141]**

> **In plain terms:** Status chips are supposed to sit at the right-hand edge of the table. The styling rule that did that was removed, so they now sit on the left of their column with empty space to their right — the opposite of what the ticket asks for. A new test was added at the same time that claims this works, so nothing fails and the problem is easy to miss.

**Function/Class:** `.column-status:last-of-type` block (removed); `tableColumns.tsx` Status column; `tableColumns.test.tsx`

**Severity:** High

**Confidence:** high

**Steps to reproduce:**

1. Open the order dashboard at ≥1280px as an admin.
2. Look at the Status column, the last column in the right-hand group.
3. **Expected:** per requirement 3, the workflow chips hug the right edge of the table.
4. **Actual:** the chips render flush LEFT inside the widened 9rem column, leaving ~2rem of dead space at the table's right edge.

**Problem:** The removed block was the only `.column-status` rule under `TableWithFilters/`. `tableColumns.tsx:241` still sets `className: "column-status"` and carries a new comment saying the content is "right-aligned via the `.column-status` rule in styles.ts", and `tableColumns.test.tsx:589` asserts that contract. With `column.align` undefined, `Styled.TableCellContent` falls back to `text-align: left`.

**Evidence:** `git grep column-status` on the branch returns the class name in `tableColumns.tsx` and the test, and zero CSS rules under `TableWithFilters/`. The test asserts the class name is applied, not the computed alignment, so it passes against the broken state.

**Impact:** Requirement 3 is not met, and the new test gives false assurance that it is — the most expensive kind of gap, because a future reviewer will trust it.

**Fix:** Restore the alignment on the column definition rather than via a class the CSS no longer defines, so the test and the behaviour cannot drift again:

```typescript
{
  id: "status",
  align: "right",
  // …
}
```

Then change the test to assert `align` (or the computed style), not the class name.

### 3. The Customer header's `-10px` compensation over-shifts in batch and order-group views

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/tableColumns.tsx:138]**

> **In plain terms:** The "Customer" heading was nudged 10px left so it lines up with the customer names below it. That nudge assumes the heading is drawn inside a padded grey pill. Inside a batch or order group there is no pill — so the nudge has nothing to cancel out, and the heading ends up 10px too far left, overlapping toward the "ID" column. The misalignment the fix was meant to remove is reintroduced in the opposite direction.

**Function/Class:** Customer column header wrapper in `tableColumns.tsx`

**Severity:** Medium

**Confidence:** high

**Steps to reproduce:**

1. Open the order dashboard as an admin.
2. Drill into a **batch** or an **order group** (sets `isBatchOrOrderGroupFilter`).
3. Look at the "Customer" column heading against the customer values below it.
4. **Expected:** heading left edge aligned with the values.
5. **Actual:** the heading is drawn 10px to the left of the column's content edge, encroaching toward the "ID" heading.

**Problem:** `isBatchOrOrderGroupFilter` is forwarded as `isBatchFilter` → `showLabelOnly`, and `FilterDropdown/index.tsx` short-circuits on that flag: `if (showLabelOnly) return <Text>{label}</Text>;` — no `TriggerWrapper`, therefore no `padding: 0.5rem 10px`. The `-10px` wrapper is applied unconditionally.

**Evidence:** `FilterDropdown/index.tsx` first branch (bare `<Text>` return); `tableColumns.tsx:138` (`marginLeft: "-10px"` with no condition).

**Impact:** Requirement/testing-note 4 ("no column misaligns against its group") fails in batch and order-group views — a view this PR's manual QA notes do not mention checking.

**Fix:** Make the compensation follow the padding it cancels. Either apply it conditionally:

```typescript
style={{ marginLeft: isBatchOrOrderGroupFilter ? undefined : "-10px" }}
```

or better, remove the need for compensation entirely by making the trigger's horizontal padding non-offsetting (e.g. pad the pill via a pseudo-element or negative inset on `TriggerWrapper` itself), which also fixes Issue 5 for free.

### 4. Shared `Sidebar` hardcodes creative-portal's header height, which is wrong below `tabletSm`

**[File: packages/shared/components/molecules/Sidebar/styles.ts:19]**

> **In plain terms:** The order side panel is positioned by assuming the header is always 60px tall. On smaller screens the header is 96px, so the panel starts too high and the header covers its top 36px — including the close button, which means the panel can't be dismissed without scrolling. The value is also a copy of a number that lives in the creative portal, so the next time that number changes, this quietly breaks again.

**Function/Class:** `StyledSidebar` `top`

**Severity:** Medium

**Confidence:** high

**Steps to reproduce:**

1. Reduce the viewport below the `tabletSm` breakpoint.
2. Open an order to bring up the side panel.
3. **Expected:** the panel starts flush below the header, with its close/back bar visible.
4. **Actual:** the panel is pinned 60px from the top while the header occupies 96px, so the header paints over the panel's top 36px (`StyledSidebar` uses `z-index: zIndices.header - 1`), hiding the sticky close/back bar.

**Problem:** `top` is hardcoded to `3.75rem`, duplicating `HEADER_HEIGHT` from `apps/creative-portal/components/styles/styles.ts` inside a shared package. That value only applies at `media.tabletSm` and above; below it `StyledHeaderContent` is `height: 6rem`. The previous `72px` had the same class of problem but hid 12px less.

**Evidence:** `packages/shared/components/molecules/Sidebar/styles.ts:19`; `apps/creative-portal/components/organisms/Header/styles.ts` (`StyledHeaderContent` height inside `@media ${media.tabletSm}`).

**Impact:** An undismissable panel on small viewports, plus a cross-package coupling that cannot be kept in sync — a shared package should not encode one app's header height.

**Fix:** Pass the offset in as a prop (or a CSS custom property set by the consuming layout) so the shared component has no opinion about any app's header:

```typescript
// consumer supplies it; shared component just consumes the variable
top: var(--app-header-height, 3.75rem);
```

and have the creative portal set `--app-header-height` from `HEADER_HEIGHT`, breakpoint-aware.

### 5. The Status header never got the `-10px` compensation its two siblings did

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/tableColumns.tsx:246]**

> **In plain terms:** Three column headings sit inside the same padded pill. Two of them were nudged to line up with the values underneath; the third — "Status" — was not, so it still sits 10px to the right of its own column's values. It's the same misalignment the ticket was raised to fix, left in place for one of the three.

**Function/Class:** Status column header (`StatusSelectFilter` → `FilterHeadingTrigger`)

**Severity:** Medium

**Confidence:** high

**Steps to reproduce:**

1. Open the order dashboard at ≥1280px as an admin.
2. Compare each of the three pill-style headings — Customer, Time Remaining, Status — with the values directly beneath them.
3. **Expected:** all three aligned with their values.
4. **Actual:** Customer and Time Remaining are pixel-aligned; Status starts 10px to the right of its values.

**Problem:** `StatusSelectFilter` passes `withPadding` (`0.5rem 10px`) to `FilterHeadingTrigger` exactly as the Customer filter does, but received no compensating negative margin, while the Time Remaining header got `margin-left: -10px` on `TimeRemainingSortByDropdownWrapper`.

**Evidence:** `tableColumns.tsx:246` (Status header, no margin) vs `:138` (Customer, `marginLeft: "-10px"`) and the `TimeRemainingSortByDropdownWrapper` rule.

**Impact:** Testing note 4 fails for the Status column in the normal (non-batch) dashboard view.

**Fix:** Covered by the systematic fix suggested in Issue 3 — make the trigger's padding non-offsetting so no header needs a manual counter-margin. If the per-header approach is kept, add the same `-10px` here.

### 6. The empty-result layout shift is moved, not removed

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/table.tsx:293]**

> **In plain terms:** The fix stops the table jumping when data arrives — but only when data actually arrives. If a filter matches no orders, the two reserved columns are dropped the moment loading ends, and the whole header row jumps about 104px to the left as the "No results found" message appears. The jump has been relocated, not eliminated.

**Function/Class:** `showExtraColumns`

**Severity:** Medium

**Confidence:** high

**Steps to reproduce:**

1. Open the order dashboard as an admin.
2. Apply a filter combination that matches no orders.
3. Watch the header row as the request resolves.
4. **Expected:** no horizontal movement — the stated goal of this hunk.
5. **Actual:** during the fetch the expand (40px) and selection (64px) columns are present; when the empty result lands, both are removed and the header row jumps ~104px left alongside the "No results found." state.

**Problem:** `showExtraColumns = dataPaginated.length > 0 || isLoading` is false in the settled-empty state, so the reservation is released exactly when the empty state renders.

**Evidence:** `table.tsx:291-295`.

**Impact:** The layout-shift bug this hunk targets still reproduces on the empty-result path, which is a common outcome of filtering.

**Fix:** Reserve the columns whenever the table chrome is rendered at all, rather than tying reservation to row presence:

```typescript
const reserveExtraColumns = true; // or: !isEmptyStateStandalone
```

If the columns must collapse for the empty state, collapse them for the loading state too, so both transitions are shift-free.

### 7. The persistent active pill leaks onto pages outside this ticket

**[File: apps/creative-portal/components/molecules/FilterDropdown/partials/FilterHeadingTrigger/styles.ts:78]**

> **In plain terms:** The grey rounded "pill" behind an active filter heading used to appear only on hover. It now stays on permanently once a filter is applied — everywhere the component is used, including the Partners page, which this ticket does not cover and which was never checked against a design.

**Function/Class:** `TriggerWrapper` — `isActive && pillStyles`

**Severity:** Medium

**Confidence:** high

**Steps to reproduce:**

1. Open the **Partners** page as an admin.
2. Apply a partner status filter.
3. **Expected:** the "Status" heading appearance is unchanged by a PR scoped to the order dashboard.
4. **Actual:** the heading permanently renders a navy-3% rounded 2.5rem pill instead of showing it only on hover.

**Problem:** `TriggerWrapper` is the single trigger used by every `components/molecules/FilterDropdown` consumer. `apps/creative-portal/components/pages/partners/partials/StatusFilter/index.tsx` passes both `isActive` and `pillStyles`, so it inherits the change.

**Evidence:** `FilterHeadingTrigger/styles.ts:78`; `pages/partners/partials/StatusFilter/index.tsx` (passes `isActive` + `pillStyles`).

**Impact:** An un-specced visual change on a page outside the ticket, shipped with no test covering the Partners trigger. Low user harm, but it is exactly the kind of unreviewed cross-page drift that accumulates.

**Fix:** Scope the persistent pill to the consumers the ticket covers — add an opt-in prop (`hasPersistentActivePill`) defaulting to the old hover-only behaviour, and set it from the order dashboard's three filters. If the intent is platform-wide, get it confirmed by the designer and note it in the PR description.

### 8. `scrollbar-gutter: stable` removal reverts the PP-1806 fix

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/styles.ts:391]**

> **In plain terms:** A line that reserved space for the scrollbar was removed. That line was added specifically to stop the order list jumping sideways when the side panel opens. Removing it brings that jump back. The PR description does mention removing it — but not that it undoes an earlier bug fix.

**Function/Class:** `TableWrapper`

**Severity:** Medium

**Confidence:** medium — the mechanism and the origin are verified; whether the regression is acceptable is a product/author call

**Steps to reproduce:**

1. Open the order dashboard as an admin with enough rows to show a scrollbar.
2. Open an order to expand the side panel, then close it.
3. **Expected:** the list does not shift horizontally (the behaviour PP-1806 delivered).
4. **Actual:** the list jumps by the scrollbar width as the gutter is no longer reserved.

**Problem:** `git log -S"scrollbar-gutter"` attributes the declaration to `3e1a1aa71` — "fix/PP-1806: Border bottom and orders shift glitch when sidepanel opens (#2247)". `TableWrapper` is the element whose `max-width` flips to `calc(100% - 720px)` when the side panel opens, which is precisely the transition PP-1806 stabilised.

**Evidence:** `styles.ts:391` (removal); `git log -S"scrollbar-gutter"` → commit `3e1a1aa71` (#2247).

**Impact:** A previously-fixed defect returns. The PR body describes this as removing "the stray reserved scrollbar gutter", which suggests the author believed it was vestigial rather than load-bearing — so this is likely a misread rather than a deliberate trade-off.

**Fix:** Restore the declaration, or — if it genuinely conflicts with the new `overflow: auto` behaviour — state in the PR why PP-1806's scenario no longer needs it, and re-test that scenario explicitly. A code comment referencing PP-1806 either way would stop the next person removing it again.

### 9. Loading skeletons still use the pre-PR widths, so the table snaps narrower after scrolling

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/tableColumns.tsx:161]**

> **In plain terms:** The grey "loading" placeholder bars are wider than the new column widths. Because the table sizes itself to its widest cell, the placeholders — which are on screen for most of the time while scrolling a long list — force three columns wider than the design specifies. When you reach the end of the list and the placeholders disappear, the whole table visibly snaps narrower.

**Function/Class:** `OrderRowSkeleton` vs `tableColumns.tsx` widths

**Severity:** Low

**Confidence:** high

**Steps to reproduce:**

1. Open the order dashboard as an admin and scroll the infinite list.
2. Observe the Content Size / Services / Customer column widths while skeleton rows are visible.
3. Scroll to the very end so the skeletons unmount.
4. **Expected:** column widths unchanged throughout, matching Figma.
5. **Actual:** those three columns render ~1.5rem wider while skeletons are present, then the table snaps narrower once they unmount.

**Problem:** The table has no `table-layout: fixed` — widths come from `<col style={{width}}>` via the shared Colgroup, so the widest cell wins. New content boxes are orderSize 4.5rem − 2rem padding = 2.5rem, services 7rem − 2rem = 5rem, customer 9.5rem − 2rem = 7.5rem, while `partials/OrderRowSkeleton/index.tsx` still renders 4rem / 6rem / 8rem skeletons for those ids — under a comment reading "Keep in sync with tableColumns.tsx column ids".

**Evidence:** `tableColumns.tsx` (new widths); `partials/OrderRowSkeleton/index.tsx` (unchanged skeleton widths + the sync comment).

**Impact:** Requirement 1 (Figma widths) is not met in the state the user spends most of their scrolling time in, and the PR's own layout-shift goal is undermined by a second shift it introduces.

**Fix:** Derive the skeleton widths from the column widths rather than restating them, so they cannot drift:

```typescript
// consts.tsx
export const COLUMN_CONTENT_WIDTH = {
  orderSize: "2.5rem",
  services: "5rem",
  customer: "7.5rem"
};
```

and have both `tableColumns.tsx` and `OrderRowSkeleton` read from it.

### 10. A repo-wide pre-commit policy change is bundled into a UI PR — and used to justify this PR's own testing gap

**[File: CLAUDE.md]**

> **In plain terms:** This PR also rewrites the team's rule about which tests must pass before committing — from "run everything" to "run only what you changed". That's a process decision for the whole team, not part of a column-widths ticket, and this PR then cites the new rule as the reason it didn't run the full test suite itself.

**Function/Class:** "Pre-Commit Verification (mandatory)" and "Working Rules" sections

**Severity:** Medium

**Confidence:** high

**How to spot it:** Not user-visible. `CLAUDE.md` is +24/−8 in a PR whose other 13 files are all dashboard styling.

**Problem:** The section changes from `npx turbo run test` ("ALL tests must pass — 0 failures") to running only "the test files you added or modified", the tests for modified symbols, and the impact radius, with `--filter`-scoped typecheck/lint/build. The PR's own checklist then reads: "full creative suite deferred to CI per updated targeted-testing policy". A PR that relaxes a verification rule and simultaneously relies on the relaxation is self-justifying — the reviewer is asked to accept the weaker gate and the weaker evidence in the same review.

**Evidence:** `CLAUDE.md` diff (Pre-Commit Verification + the "Tests required" / "Build clean" bullets); PR body "Testing" section.

**Impact:** The rationale is reasonable — most Vitest wall-clock is jsdom setup, and the graph MCP genuinely can identify affected tests. But it is a change to the team's quality gate, it deserves its own review and team agreement, and bundling it here means it merges on the strength of a styling review. Note also that this PR touches `packages/shared`, which the new policy itself says requires the **full** unscoped suite.

**Fix:** Split `CLAUDE.md` into its own PR so the policy is reviewed on its merits. In this PR, run the full suite as the current (unmodified) policy requires — and as the new policy would also require, since `packages/shared/components/molecules/Sidebar` is in the diff.

---

## Open Questions

- Was removing `scrollbar-gutter: stable` (Issue 8) a deliberate trade-off against PP-1806, or was it read as vestigial? The PR body calls it "the stray reserved scrollbar gutter", which reads like the latter — please confirm.
- The Status column keeps `className: "column-status"` after its only rule was deleted (Issue 2). Is the class still consumed anywhere outside `TableWithFilters/` (a test id, an E2E selector), or should it be removed along with the rule?
- `SPACER_COLUMN_WIDTH.SIDEPANEL_OPEN` collapses the spacer to 2rem when the panel opens. With the panel open, is the right-hand group still expected to hug the right edge, or is the intent that the groups converge? The Figma node covers the ≥1280px full-width case only.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⚠️ Not run | PR head `13cd91e` cannot be fetched in this environment — `git fetch` over HTTPS has no stored credentials and the available SSH key has no repo access. Unchanged from the #2368 / #2369 reviews. |
| `npx turbo run typecheck` | ⚠️ Not run | Same. |
| `npx turbo run lint` | ⚠️ Not run | Same. |
| `npx turbo run build` | ⚠️ Not run | Same. The PR itself reports the production build had "env-flaky, non-deterministic failures" and leaves that checklist item unticked. |
| Mergeable state | ✅ `clean` | Against its base — but the base is `feature/PP-1910-dashboard-column-cleanup`, two PRs deep in an unmerged stack. |

**To unblock:** run `git fetch origin feature/PP-1915-order-dashboard-column-widths-alignment` in your own shell (prefix with `!` in this session) and the suite can run against a detached worktree at the PR head.

**Scope note:** this PR touches `packages/shared`, so the **full unscoped** suite is required — both by the current `CLAUDE.md` and by the revised policy this PR proposes.

**Author's reported state (unverified here):** affected `TableWithFilters` + `FilterHeadingTrigger` tests and the full `@proofed/shared` suite (1,353) green; typecheck and lint green; full creative-portal suite **not** run; production build **not** confirmed.

---

## Tests

- ✅ `insertSpacerColumn` ships with a dedicated unit test (+104 lines) — the right level for a pure helper.
- ✅ `tableColumns.test.tsx` adds width assertions for the new Figma widths.
- ❌ **The new Status alignment test asserts a class name, not the alignment** (Issue 2), so it passes against a broken column. This is worse than no test — it documents the regression as working.
- ❌ No test covers the select-all-while-loading path (Issue 1), despite it being reachable from a two-click sequence and having data consequences.
- ❌ No test covers the batch/order-group header alignment (Issue 3) or the Partners trigger (Issue 7).
- ⚠️ Skeleton widths are not asserted against column widths (Issue 9), which is what let them drift.
- ⚠️ Full creative-portal suite not executed by the author or this review.

### Suggested manual QA script

Run against the PR branch at ≥1280px unless stated otherwise:

1. **Select-all while loading** (Issue 1) — apply filters, change the month, click select-all during the loader, wait for data; confirm no orders end up selected. Then refresh and confirm nothing is selected.
2. **Status alignment** (Issue 2) — confirm status chips hug the right edge with no dead space.
3. **Batch view headers** (Issue 3) — drill into a batch; confirm "Customer" lines up with the values below it.
4. **Status header** (Issue 5) — confirm "Status" lines up with its values, like Customer and Time Remaining.
5. **Empty result** (Issue 6) — apply a filter matching no orders; confirm the header row does not jump as the empty state appears.
6. **Partners page** (Issue 7) — apply a partner status filter; confirm the heading looks as it did before this PR.
7. **Side panel + scrollbar** (Issue 8) — open and close the order side panel; confirm the list does not shift horizontally.
8. **Skeleton widths** (Issue 9) — scroll the list to the end; confirm the table does not snap narrower when skeletons unmount.
9. **Small viewport side panel** (Issue 4) — below `tabletSm`, open an order; confirm the panel's close button is reachable.
10. **Ticket acceptance** — confirm left group (checkbox → Services) is left-anchored and right group (Time Remaining → Status) right-anchored, and that the header row stays pinned while scrolling.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ❌ One Blocker (select-all latching over an empty dataset) plus a missed ticket requirement |
| Regression risk | ❌ High — a shared-package offset, a shared trigger style reaching the Partners page, and a reverted PP-1806 fix |
| Tests | ❌ A new test asserts alignment behaviour that no longer exists; the Blocker path is untested |
| Accessibility | ✅ n/a — no interactive semantics changed (the selection control's availability is a correctness issue, not an a11y one) |
| Error handling | ✅ n/a — no new async or failure paths |
| Security | ✅ Presentation-only; no new sinks or inputs. `/security` still required per CLAUDE.md before merge |
| Code quality | ✅ Good — `insertSpacerColumn` is a clean, tested abstraction and the spacer approach is the right solution to the ticket |
| Validation suite | ⚠️ Not run here; author ran a partial suite and did not confirm the build |
| Mergeable state | ⚠️ `clean` against `feature/PP-1910-dashboard-column-cleanup` — third in a stack of three unmerged PRs |

---

## Recommendation

**Request changes**

1. **Fix the select-all latch (Issue 1).** Separate "reserve the columns" from "offer selection", and make `toggleAllRowsSelected()` a no-op on an empty dataset. This is the only finding here that can change data, and it should be fixed before anything else.
2. **Restore Status right-alignment (Issue 2)** via `align: "right"` on the column definition, and change the new test to assert the alignment rather than the class name.
3. **Make the header compensation systematic (Issues 3 and 5)** — remove the need for per-header `-10px` counter-margins so the batch/`showLabelOnly` branch and the Status header are both covered.
4. **Decide on `scrollbar-gutter` (Issue 8)** — restore it, or explain why PP-1806's scenario no longer needs it and re-test that scenario.
5. **Scope the active pill (Issue 7)** to the order dashboard, or get the platform-wide change confirmed by the designer.
6. **Take the header offset out of the shared package (Issue 4)** — pass it in, breakpoint-aware, rather than hardcoding one app's constant.
7. **Sync the skeleton widths (Issue 9)** by deriving both from one constant.
8. **Split `CLAUDE.md` into its own PR (Issue 10)** and run the full suite for this one, as both the current and the proposed policy require for a `packages/shared` change.
9. **Run the full validation suite** against `13cd91e` and confirm the production build — the PR currently leaves that checklist item unticked with "env-flaky" failures unexplained.
10. **Retarget the stack.** This PR sits on #2369, which sits on #2368; both parents have open findings. Re-run this review against `develop` once the stack lands, since the alignment behaviour depends on the column set #2369 changes.
