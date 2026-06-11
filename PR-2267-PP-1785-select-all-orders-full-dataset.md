# PR Review: PP-1785: Select all orders across full dataset with total count in bulk toolbar

**PR:** https://github.com/Proofed/B2BWebserver/pull/2267
**Jira:** https://proofed.atlassian.net/browse/PP-1785
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| FR1: "Select All" applies to entire result set (incl. non-loaded). | `toggleAllRowsSelected` operates on the hook's full `data` array (not `dataPaginated`). Backend already returns the full filtered set, so all ids are present. | ✅ Addressed |
| FR2: Newly loaded orders auto-select while "All" is active. | Cleanup effect (`useTableSelection.ts:172-200`) detects `dataHasChanged && isAllSelected` and replaces `selectedRowIds` with `Object.fromEntries(data.map(...))`. Unit test "merges newly-arrived ids …" exercises this. | ✅ Addressed |
| FR3: Toolbar shows "All orders selected (X)". | `BulkToolbar/index.tsx:156-161` returns `All orders selected (${selectedRowsCounter})` when both flags set. | ✅ Addressed |
| FR4: Filter change resets "All" + count reflects filtered set. | `!isSameView` branch resets `isAllSelected=false`, `selectedRowIds={}`. Count is `Object.keys(selectedRowIds).length` which matches the full filtered set when All is active. | ✅ Addressed |
| FR5: Deselecting after "All" → partial selection. | `toggleRowSelected` flips `setIsAllSelected(false)` on any explicit `isSelected===false` or implicit-deselect. Unit test "deselecting a single row …" verifies. | ✅ Addressed |
| FR6: "All" state distinct from manual multi-select. | `isAllSelected` (drives toolbar label) split from `isHeaderFullyChecked` (drives checkbox visual). Manual tick-every-row keeps `isAllSelected=false`. Unit test "distinguishes explicit 'Select All' …" verifies. | ✅ Addressed |

Scope creep / extras (out of Jira scope but in PR):
- Prettier reformat of `as` casts in `PaymentDetailsStep2/hooks.test.ts`, `PaymentDetailsStep2/hooks.ts`, `customer-portal/hooks/useNavigation/index.test.ts` — unrelated to PP-1785 (probably a lint-fix sweep). Harmless but should ideally land in a chore PR.

---

## Architecture Analysis

The fix correctly identifies that the prior implementation conflated "rendered slice" (`dataPaginated`, ~50 rows from the infinite-scroll cursor) with "selectable universe" (the full filtered set returned by the server in one call). Routing the header checkbox and bulk-toolbar derivations through the full `data` array fixes FR1 without any new API work.

The dual-flag design (`isAllSelected` vs `isHeaderFullyChecked`) is the right way to satisfy FR6 — the checkbox needs to reflect "every visible row checked" regardless of provenance, while the toolbar label needs to know whether the user pressed "Select All" or coincidentally ticked every row.

The cleanup `useEffect` (lines 154–208) handles the three relevant transitions in one place: (a) filter change → full reset, (b) data change + All mode → re-broadcast selection over new ids, (c) data change + manual mode → prune ids that left the dataset. Combining filter-reset and data-change handling in one effect is more concise than two effects but ties the FR2/FR4 fates together — if you ever need to relax FR2 (e.g. cap auto-select), you'll be untangling them.

The `BulkToolbar` lives in `packages/shared` and is consumed by `TeamMembersTable` and `ChargeTable` in addition to the orders table. The label literal "All orders selected" pre-dated this PR; the new `(N)` suffix now lights up for those consumers too if they ever pass `isAllRowsSelected + selectedRowsCounter`. Pre-existing naming smell, not a new bug.

---

## Issues Found

### 1. `toggleRowSelected` reads `selectedRowIds` from a stale closure inside a synchronous loop

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/hooks/useTableSelection.ts]**
**Function/Class:** `toggleRowSelected`
**Severity:** low
**Problem:** The "implicit deselect" detection reads from the captured `selectedRowIds` rather than from the functional updater's `prev`:

```typescript
setSelectedRowIds((prev) => {
  const shouldSelect = isSelected ?? !prev[rowId];
  ...
});

if (
  isSelected === false ||
  (isSelected === undefined && selectedRowIds[rowId])  // <- closure, not prev
) {
  setIsAllSelected(false);
}
```

When `toggleRowSelected` is invoked multiple times synchronously in the same render (e.g. via `toggleGroupSelection` looping `orderIds.forEach((id) => toggleRowSelected(id, shouldSelect))` in `table.tsx:177`), every call reads the same `selectedRowIds` snapshot. In `toggleGroupSelection`, `shouldSelect` is always explicit (`true`/`false`), so the `isSelected === undefined` branch never fires and the bug is masked. But future call sites that pass `undefined` repeatedly will hit the staleness.

**Impact:** Today this is latent — no caller currently invokes the toggle path synchronously without an explicit boolean. But the logic reads as "fires `setIsAllSelected(false)` if we were just deselecting", and that contract silently breaks under synchronous batched calls.
**Fix:** Compute the deselect signal inside the functional updater and stage the side-effect via a ref, e.g.:

```typescript
const wasDeselectRef = useRef(false);

const toggleRowSelected = useCallback(
  (rowId: string, isSelected?: boolean) => {
    setSelectedRowIds((prev) => {
      const wasSelected = !!prev[rowId];
      const shouldSelect = isSelected ?? !wasSelected;
      if (!shouldSelect && wasSelected) wasDeselectRef.current = true;
      const next = { ...prev };
      if (shouldSelect) next[rowId] = true;
      else delete next[rowId];
      return next;
    });

    if (wasDeselectRef.current) {
      wasDeselectRef.current = false;
      setIsAllSelected(false);
    }
  },
  [setSelectedRowIds]
);
```

This also drops `selectedRowIds` from the dep array and stops the callback from re-creating on every selection change.

### 2. `isAllSelected` lingers across a transient "empty filtered dataset" window

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/hooks/useTableSelection.ts]**
**Function/Class:** cleanup `useEffect` (lines 154–208)
**Severity:** low
**Problem:** When the data array goes `[…5 ids…] → [] → [6 new unrelated ids]` while the filter doesn't change (e.g. all five orders complete and disappear via auto-refresh, then a brand-new order arrives), the path is:
- empty step: `dataHasChanged=true` but `data.length > 0` is false → outer `if` is skipped, `previousDataIds.current` is NOT updated, `isAllSelected` stays `true`.
- refill step: `dataHasChanged=true`, `data.length > 0` → enters `isAllSelected` branch → silently selects the 6 unrelated new ids.

The other "clear on empty" effect (line 216) only fires when `!areFiltersEnabled`, so an actively-filtered view can sit in this stuck state indefinitely.

**Impact:** Long-lived dashboard sessions can accumulate "All Selected" semantics across what the user perceives as different cohorts of orders. The toolbar will read `All orders selected (6)` for the new cohort with no user action — matches FR2's literal wording but the user never opted into "All" for the new orders.
**Fix:** Either (a) reset `isAllSelected` when data goes to 0 even if filters are enabled, or (b) update `previousDataIds.current` in the empty step so the refill is treated like a fresh dataset:

```typescript
} else if (data.length === 0 && isAllSelected) {
  setIsAllSelected(false);
}
```

Worth confirming the intended behavior with the PM — FR2 reads as "newly arriving orders within the same query", which a 5→0→6 turnover arguably violates.

### 3. Test "selects current rows when stale ids carry over with a matching count" doesn't actually exercise the stale-id path

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/hooks/__tests__/useTableSelection.test.ts]**
**Function/Class:** last `it()` block (lines 269–296)
**Severity:** low
**Problem:** The test pre-seeds `mocks.setSelectedRowIds({ "stale-1": …, "stale-2": …, "stale-3": … })` then mounts the hook. On mount, the cleanup `useEffect` immediately detects `dataHasChanged=true` (`previousDataIds` was empty) and the cleanup branch wipes every id not in `currentDataIds`, so by the time `toggleAllRowsSelected()` is called, `selectedRowIds` is already `{}`. The test passes for the right outcome but not for the reason the name implies — the code path under test is "click Select All from a clean slate", not "click Select All while stale ids are present".
**Impact:** Low. The behaviour is still correct, and the stale-id guard inside `toggleAllRowsSelected` (`data.every((row) => selectedRowIds[row.id])`) is conceptually covered, just not by this test. Mostly a documentation-correctness issue.
**Fix:** Either rename to something like `"toggleAllRowsSelected ignores ids not in current data"`, or write a separate test that bypasses the cleanup effect (e.g. seed `previousDataIds.current` indirectly by re-rendering with stable data so the cleanup is a no-op, then inject stale ids and toggle).

### 4. No regression coverage for "All mode + ids leave the dataset"

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/hooks/__tests__/useTableSelection.test.ts]**
**Function/Class:** test suite
**Severity:** low
**Problem:** The FR2 test only covers `data` growing (`[1,2,3] → [1,2,3,4,5]`). The mirror case — an id leaving the dataset while All mode is active (`[1,2,3] → [1,3,5]`) — isn't exercised. Per the `isAllSelected` branch the resulting `selectedRowIds` will be `{1: true, 3: true, 5: true}` (dropping 2, picking up 5). That's the right behaviour but worth pinning down so a future refactor doesn't silently flip it to "remember 2".
**Impact:** Light. Risk that a future change breaks the "follow the dataset" semantic without a failing test.
**Fix:** Add a third `rerender` step to the FR2 test that swaps the dataset and asserts the dropped id is gone and the new id is in.

### 5. PR description states 6 unit tests; the file has 8

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/hooks/__tests__/useTableSelection.test.ts]**
**Function/Class:** N/A
**Severity:** low
**Problem:** PR body: "6 cases covering FR2, FR4, FR5, FR6 and the golden path". The test file contains 8 `it()` blocks (including the sessionStorage restore test and the stale-ids test). The runner confirms `Tests 8 passed (8)`.
**Impact:** None functional, just a doc/changelog discrepancy.
**Fix:** Update the PR description count to 8 (and the description of what each test covers if useful for reviewers).

### 6. Pre-existing: duplicate `BULK_ACTIONS_CONFIG` defined in both `consts.tsx` and `consts/index.ts`

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/consts.tsx]**
**Function/Class:** `BULK_ACTIONS_CONFIG` (also at `consts/index.ts:27`)
**Severity:** low
**Problem:** Two `consts` modules co-exist (`consts.tsx` and `consts/index.ts`), each exporting `BULK_ACTIONS_CONFIG` and `TABLE_CONFIG`. TypeScript module resolution picks the folder index over the sibling `.tsx`, so the `.tsx` defs are dead. Not introduced by this PR (predates it), but flagged because the PR's tests import from `"../../consts"` and inherit the ambiguity.
**Impact:** Future edits to constants may silently land in the dead file.
**Fix:** Delete `consts.tsx` and consolidate into the `consts/` folder, or vice-versa. Out of scope for this PR but worth a follow-up ticket.

---

## Tests

- ✅ New `useTableSelection.test.ts` — 8 cases, all pass (`yarn app:creative-portal test useTableSelection` → 8/8).
- ✅ New `BulkToolbar.test.tsx` covers all three label branches (FR3 + pre-existing partial + pre-existing All-without-counter).
- ✅ FR2, FR4, FR5, FR6 each have at least one dedicated unit test.
- ⚠️ Missing: All-mode + id-removal scenario (see issue 4).
- ⚠️ Missing: Manual QA walkthrough in dev against a >50-order filter (PR description's `[ ] Manual QA …` is unchecked). FR1 ultimately hinges on this — please verify before merge.
- ⚠️ No E2E coverage (acknowledged in the PR body).
- ⚠️ Cross-table regression check: `TeamMembersTable` / `ChargeTable` still use `BulkToolbar`; the new `(N)` branch in `toolbarLabel` only triggers when `isAllRowsSelected && selectedRowsCounter > 0` — confirm by visual inspection that those callers haven't started showing "All orders selected (N)" with a misleading number.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ |
| Regression risk | ⚠️ Medium (shared BulkToolbar label change touches other tables; edge case where `isAllSelected` lingers across data turnover) |
| Tests | ⚠️ Good for unit, missing manual QA and one mirror-case test |
| Code quality | ✅ |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve with suggestions.**

The fix is sound, the FR↔code mapping is clean, and unit coverage is meaningful. Block-on items before merge:

1. Complete the manual QA pass against a >50-order filter (PR checkbox still unchecked). This is the only way to verify FR1 + FR2 + FR4 in the real lazy-load path.
2. Sanity-check `TeamMembersTable` and `ChargeTable` BulkToolbar labels still render correctly — the label-template change is in a shared component.

Nice-to-haves (can land as follow-ups):

3. Refactor `toggleRowSelected` per issue 1 to drop `selectedRowIds` from the dep array (perf + closure correctness).
4. Decide the desired behaviour for issue 2 (lingering `isAllSelected` across empty-dataset transition) with the PM and either fix or document.
5. Add the missing FR2 mirror-case test (issue 4) and rename the misleading test (issue 3).
6. Open a chore ticket to deduplicate the two `consts` modules (issue 6).
