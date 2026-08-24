# PR Review: feature/PP-2040: Add order dashboard stats bar with sieving

**PR:** https://github.com/Proofed/B2BWebserver/pull/2435
**Jira:** https://proofed.atlassian.net/browse/PP-2040
**Status:** Code Review
**Author:** gaurav-proofed
**Size:** 34 files, +3,974 / −53 (all inside `apps/creative-portal`)
**Reviewed at:** `3e5981d7f` (branch is 6 commits behind `develop`)

---

## What this means for users (non-technical summary)

1. **The order list can narrow itself when nobody touched it.** After you use the bar to focus on one job status, if the last matching order leaves and later a matching order comes back, the narrowing switches itself back on. You get a near-empty dashboard with no memory of asking for it, and it can happen while you are looking at something else on the page. This is the one finding I would hold the merge for.
2. **A bulk action can reach an order that is no longer in your list.** If you select the whole list and then narrow the view with a chip, an order that gets completed or moves out of your filter stays quietly selected. The count in the bulk bar is higher than the rows you can see, and a bulk action, cancel included, will reach that order.
3. **After a page reload the "select all" tick can lie.** Reload with the whole list selected and a chip active: the header checkbox comes back ticked but nothing is actually selected and the bulk bar is missing. If you then tick one row and clear the chip, the selection jumps from that one order to every order in the view.
4. **Colour-blind and low-vision admins cannot read the status breakdown.** When you open a job type's breakdown, two of the counts are marked only by a small coloured dot with no wording: one green, one red. To a red-green colour-blind viewer they are the same olive dot twice, so there is no way to tell current work from overdue work without hovering each one.
5. **Keyboard users cannot tell what half the chips do before clicking them.** Seven of the chips show only a symbol and a number. The name appears in a mouse-hover tooltip only, which never shows on keyboard focus, so a keyboard user presses a chip and the table changes without them ever learning which filter they applied.

Nothing here risks data loss or exposes another customer's data. Items 1 to 3 are correctness bugs in the new narrowing feature; 4 and 5 are accessibility gaps the author has already flagged for a decision.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
| --- | --- | --- |
| **1.1** Bar shown beneath the view title whenever a view/filter is applied; hidden if none; shown with all counts at 0 when a filter returns no orders | `isStatsBarVisible = areFiltersEnabled && hasStats` (`index.tsx:116`), rendered inside `StickyHeading`. Reuses the same flag that gates the table's "No filters selected" empty state, so the two cannot disagree | ✅ Addressed |
| **1.2** Eight stats left to right, always visible including at 0: All, Overdue, On Hold, AI, Service, Review, QA, Return | `CURRENT_JOBS_STATS_CONFIG` (`OrdersStatsBar/consts.ts:93-112`) in exactly that order; `countOrderStats` seeds every id at 0 before counting (`utils.ts:127-129`) | ✅ Addressed |
| **2.1** `All` = number of orders in the view | `All` has no predicate, so it counts every row. Correctly counts **orders**, not the collapsible group row | ✅ Addressed |
| **2.2** `Overdue` = rows flagged overdue (reuse the existing overdue flag) | Extracted `isCurrentJobOverdue` (`api/orders/utils.ts:88-101`), now shared by the Status filter and the bar. Verified a behaviour-preserving extraction: the filter still passes `currentDate` explicitly, so the `now = new Date()` default is never silently used there. Good call not to use `OrderTableSchema.overdue`, which is declared but never populated | ✅ Addressed |
| **2.3** `On Hold` = current jobs on hold across all job types | `predicate: (facts) => facts.jobStatus === "On Hold"` (`consts.ts:109`) | ✅ Addressed |
| **2.4** AI / Service / Review / QA / Return = current jobs of that type | Reuses the Current Job filter's own parsers, so the bar and the filter cannot drift. AI collapses Pre-edit and Post-edit | ✅ Addressed (blocked from sign-off by PP-2027, per the ticket's own Dependencies note) |
| **2.5** Counts always describe the full unsieved view | Counted from `ordersResult` before the sieve is applied (`hooks.ts:500-512`). Proven by mutation: feeding the narrowed set instead fails 3 tests | ✅ Addressed |
| **3.1** Exactly one selection at all times; `All` = no narrowing | Single `selectedStatId`; `isSelected: stat.id === selectedStatId` makes two selections structurally impossible | ✅ Addressed |
| **3.2** Sieving alters only which rows are shown — never row content, a filter, or the saved view, and never triggers Save View's unsaved state | Filter isolation is real and verified by reading, not by trusting the description: `useOrdersStatsSieve` receives no filter setter and returns none, so it cannot touch `currentEncodedFilters`, cannot flip `isSameView`, and is absent from the React Query key. **However**, because the sieve runs before grouping, an order-group's header re-describes itself from the survivors — "2 Orders" above a group of 5 | ⚠️ Partial |
| **3.3** A zero-match sieve shows the empty state but keeps the bar and its counts | `reconcileStatsBarState` deliberately keeps a top-level selection alive at 0 (`utils.ts:175-186`); tested | ✅ Addressed |
| **4.1** Clicking a job type reveals its per-status counts inline and does **not** sieve; one breakdown at a time; only statuses present appear | `isSievable: false` on job types; `onStatClick` clears a selection belonging to another job type; `visibleChildren` filtered to `counts > 0`. Well covered by tests | ✅ Addressed |
| **4.2** Clicking a status sieves to rows with a current job of that type and status | `getJobStatusStatId` predicates match on both type and status; tested against the rows kept, not merely that a predicate exists | ✅ Addressed |
| **5.1** A filter/saved-view change recalculates counts; a still-valid selection is retained, otherwise resets to `All` | Counts recalculate correctly. The reset to `All` is **derived but never persisted**, so a dropped selection resurrects itself — see Issue 1 | ⚠️ Partial |
| **5.2** Bar state persists per saved view for the browser session | Per-view `sessionStorage` bucket keyed by saved-view id, with ad-hoc filters sharing one bucket. Works, but the key is not user-scoped and nothing clears storage on logout — see Issue 8 | ⚠️ Partial |
| **VR 1** Sieving never alters filters, saved views, or row content | Filters and saved views: verified clean. Row content: the group summary recount above | ⚠️ Partial |
| **VR 2** Never more than one selection or more than one breakdown at a time | Enforced in `onStatClick` and again defensively in `buildStatsBarGroups`; tested from several click sequences | ✅ Addressed |
| **VR 3** Future jobs never counted; counts never recalculated against the sieved rows | Only `liveStatus` (the current job) is parsed; counts come from the unsieved set | ✅ Addressed |
| **VR 4** Completed / Cancelled stats do not appear in this view | `CLOSED_ORDERS_STATS_CONFIG` is empty, so the bar hides itself there, and reconciliation validates ids against the active config so a sieve cannot leak in. Tested | ✅ Addressed |

### Changes beyond the Jira scope

All of it is defensible, and the author discloses each one in the PR body:

- **`retainedRowIds` on `useTableSelection`** — necessary, not scope creep. Hiding rows would otherwise prune a bulk selection irrecoverably, and `refetchOnWindowFocus: "always"` means that fires on any tab switch. It does, however, carry Issues 2, 3 and 4.
- **Extracting `isCurrentJobOverdue` and `ORDER_STATUS_ICONS`** — both verified behaviour-preserving. The `ORDER_STATUS_ICONS` extraction is byte-identical across all eight status/icon pairs.
- **A `sessionStorage` crash fix** — a literal `null` or the string `"undefined"` took the dashboard to the error boundary and survived a reload. Verified the fix is complete: every read path funnels through the deserializer, and junk under a valid view token is absorbed by reconciliation on every branch.
- **The header-checkbox selection fix** — a genuine pre-existing bug (ticking the header destroyed sieved-out selections). The fix is right in principle; Issues 3 and 4 are gaps in its edges, not in its premise.

---

## Architecture Analysis

The shape of this PR is good, and unusually so for its size. Every rule lives in a pure function in `OrdersStatsBar/utils.ts` (facts, predicates, counting, sieving, reconciliation, chip building), the composition is data in `consts.ts` rather than JSX, and the component is purely presentational. That is what makes most of the behaviour testable without a renderer, and it is why PP-2041 should be a `consts.ts` change rather than a rewrite. Deriving one `OrderRowStatFacts` per row and having every predicate read only that is the right call: `liveStatus` is parsed once per row, and counting never touches an order object.

Two decisions deserve explicit credit:

- **Reusing the Status filter's own predicate for `Overdue`** rather than re-deriving it. The ticket said "reuse the existing overdue flag", and the obvious candidate (`OrderTableSchema.overdue`) is a decoy that is never populated. Extracting the filter's real predicate so the two cannot drift is the correct reading of the requirement, and leaving the filter's 34 tests untouched through the extraction is the right evidence that nothing changed.
- **The isolation boundary on `useOrdersStatsSieve`.** The hook's signature is the guarantee: no filter setter goes in, none comes out. I verified this by reading rather than trusting it, and Req 3.2's hardest clause holds by construction rather than by discipline.

The one architectural choice I would question is **sieving upstream of `groupOrdersCreative`**. It buys you a group that never leaves an orphaned header, and it costs you a group header that misreports its own size. The author names this trade-off in the PR body and in a source comment, and raises the consequence as an open question, so this is a decision for the PO rather than a defect — but it is the reason Req 3.2 lands on ⚠️ rather than ✅.

The genuine architectural weak point is elsewhere: **state that is derived on every render but persisted only on click.** `reconcileStatsBarState` is a pure function over what is in storage, and nothing ever writes its result back. That asymmetry is the direct cause of Issue 1, and the hook's own comment at `useOrdersStatsSieve.ts:120-123` shows the author saw the hazard and guarded only the click path against it.

---

## Issues Found

### 1. A cleared sieve resurrects itself and silently re-narrows the table

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/hooks/useOrdersStatsSieve.ts:90]**

> **In plain terms:** After you narrow the orders table to a single job status, if the last matching order leaves the view the table correctly goes back to showing everything. But the narrowing quietly switches itself back on the moment a matching order reappears, which can happen on its own within thirty seconds or when you switch back to the browser tab. You get a near-empty table without having clicked anything, and nothing explains why.

**Function/Class:** `useOrdersStatsSieve` / `reconcileStatsBarState`

**Severity:** high

**Confidence:** high

**Steps to reproduce:**

1. Sign in as a Proofed Admin and open the order dashboard with any filter or saved view applied, so the stats bar is visible.
2. Click a job type chip to open its status breakdown, then click one status chip that currently has exactly one matching order. The table narrows to that order.
3. Have that order move to a different status, so the chip's count falls to 0. Do not touch the stats bar.
4. The table correctly returns to the full list and the chip stops being highlighted. So far so good.
5. Now wait for an order to enter that same status again — up to 30 seconds for the auto-refresh, or switch away from the tab and back. Still do not click anything.
6. **Expected:** the table keeps showing the full list, because no narrowing is selected any more (Req 5.1: "otherwise it resets to `All`").
7. **Actual:** the status chip lights up again by itself and the table collapses back to just the matching orders.

**Problem:** `reconcileStatsBarState` is a pure derivation over what is in `sessionStorage`, and its result is never written back. `setStatsBarByView` is called only from `onStatClick` (`:158`). So when a breakdown-child selection is dropped to `All` because its count reached 0, the raw id stays in storage, and `isValidSelection` is re-evaluated from that raw id on every subsequent render.

**Evidence:**

`useOrdersStatsSieve.ts:80-96` derives the state without persisting it:

```ts
const persisted = statsBarByView[viewToken] ?? DEFAULT_STATS_BAR_STATE;

const { revealedStatId, selectedStatId } = useMemo(
  () =>
    isLoading
      ? persisted
      : reconcileStatsBarState(persisted, counts, config),
  [persisted, counts, config, isLoading]
);
```

and `OrdersStatsBar/utils.ts:184-186` validates a breakdown child purely on its live count, so the verdict flips back the moment the count returns:

```ts
const isValidSelection =
  isSievableTopLevel ||
  (isKnownBreakdownChild && (counts[selectedStatId] ?? 0) > 0);
```

Confirmed by execution against the real hook, with **no click** between the zero-count render and the count-returns render:

```
[3 after select Service:Assigned] selected=job:Service:Assigned predicate=DEFINED keptRows=["1"]
[4 assigned row removed]          selected=all                  predicate=undefined keptRows=["1","2","3","4"]
    storage={"ad-hoc":{"selectedStatId":"job:Service:Assigned",...}}   <-- reset never persisted
[5 original orders restored]      selected=job:Service:Assigned predicate=DEFINED keptRows=["1"]
```

Reachability was verified rather than assumed. `setStatsBarByView` appears at only four lines repo-wide, all in this hook, and the setter is called only at `:158`. `useSessionStorage` never writes the value back on read: its mount effect is `useEffect(() => { setStoredValue(readValue()); }, [key])`. And `isLoading` is genuinely `false` during a background poll — `finalLoadingState` is `ordersLoading || areOrdersLoading`, where `ordersLoading` is a mount-only latch (`hooks.ts:403-407`, never set back to `true`) and React Query holds `isLoading` false while cached data exists under an unchanged key. `autoRefreshQueryOptions()` sets `refetchInterval: 30000` and `refetchOnWindowFocus: "always"` on that same key, so reconciliation runs on every poll tick.

**Impact:** The dashboard silently collapses to a fraction of its rows with no user action. On an admin dashboard that reads as data loss, and the reader has no way to connect it to a chip they clicked minutes earlier. Only breakdown children (`job:<Type>:<Status>`) are affected — top-level `Overdue` and `On Hold` are deliberately kept alive at 0 by Req 3.3, which is correct — but breakdown children are the feature's main interaction, and small-count statuses bounce between 0 and 1 routinely as jobs move.

**Fix:** Make the reset durable. When `reconcileStatsBarState` returns a state that differs from `persisted`, commit it back through `setStatsBarByView`, guarded so it fires only on an actual difference and never while `isLoading`:

```ts
const reconciled = useMemo(
  () =>
    isLoading
      ? persisted
      : reconcileStatsBarState(persisted, counts, config),
  [persisted, counts, config, isLoading]
);

useEffect(() => {
  if (isLoading || isEqual(reconciled, persisted)) return;

  setStatsBarByView((previousStore) => ({
    ...previousStore,
    [viewToken]: reconciled
  }));
}, [isLoading, reconciled, persisted, setStatsBarByView, viewToken]);

const { revealedStatId, selectedStatId } = reconciled;
```

### 2. The whole `retainedRowIds` mechanism can be unplugged with the test suite still green

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/index.tsx:188]**
**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/table.tsx:196]**

> **In plain terms:** The PR's headline protection — that narrowing the view must not silently throw away orders you had already ticked — is connected by two lines of wiring, and if either were deleted the protection would stop working with no test failing anywhere. That means a future change could quietly undo it and ship. The feature works correctly today; the risk is that nothing is holding it in place.

**Function/Class:** `TableWithFilters` → `Table` → `useTableSelection`

**Severity:** high

**Confidence:** high

**How to spot it:** This is a test-coverage gap, not a user-reproducible bug — the wiring is correct on this branch. Delete `retainedRowIds: unsievedRowIds,` from the `<Table>` props at `index.tsx:188` **and** `retainedRowIds` from the `useTableSelection({ … })` call at `table.tsx:193-197`, then run the suite.

**Problem:** No test crosses the seam between `useTableWithFilters` and `useTableSelection`. `index.test.tsx:36` mocks `./hooks` wholesale and `:53` mocks `./table` entirely, so the prop never travels through a real boundary, and there is no `table.test.tsx` in the folder at all. The twelve well-written `retainedRowIds` unit tests all call `useTableSelection` directly with the prop already supplied, so they prove the hook honours it but never that anything hands it over.

**Evidence:** Both wiring lines were deleted and the filtered suite re-run:

```
Test Files  58 passed (58)
     Tests  512 passed (512)
```

Two further mutations in the same area were also uncaught: deriving `unsievedRowIds` from `sievedOrders` was *caught* (`hooks.test.ts:169`), but reverting the cleanup effect's `currentDataIds.size > 0` guard to `data.length > 0` was **not** — 512/512 green. By contrast the four core mutations were all caught cleanly: neutering `applyOrdersStatsSieve` failed 7 tests, neutering `reconcileStatsBarState` failed 7, counting from the narrowed set failed 3, and replacing the header-checkbox merge with a plain replace failed 1.

**Impact:** If the wiring regresses, the user-visible break is the exact bug this PR set out to fix: tick three orders, click a chip, and the two hidden ones are pruned out of the bulk selection permanently. It would ship with a green suite.

**Fix:** One assertion that crosses each boundary. For `index.tsx`, capture what the mocked `./table` receives:

```ts
let capturedTableProps: Record<string, unknown> = {};

vi.mock("./table", () => ({
  default: (props: Record<string, unknown>) => {
    capturedTableProps = props;

    return <div data-testid="orders-table" />;
  }
}));

// then, in the test:
expect(capturedTableProps.retainedRowIds).toEqual(
  new Set(["order-1", "order-2"])
);
```

For `table.tsx`, spy on `./useTableSelection` and assert it is called with a `retainedRowIds` set.

### 3. While the view is narrowed with everything selected, departed orders stay selected and reach bulk actions

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/hooks/useTableSelection.ts:285]**

> **In plain terms:** Select the whole list, then use a chip to narrow what is shown. If an order then finishes or moves out of your filter, it stays quietly selected even though it is no longer part of the list. The count on the bulk bar is higher than the number of rows you can see, and a bulk action — including cancel — will reach that order. It sorts itself out as soon as you clear the chip, so the exposure lasts only while the view is narrowed.

**Function/Class:** `useTableSelection` — the filter/data cleanup effect

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. Open the orders table with a filter that returns many orders.
2. Tick the header checkbox to select the whole list, with no chip active.
3. Click a stats-bar chip so only some rows are shown.
4. Wait for the background refresh (about 30 seconds) after one of the selected orders changes so it no longer matches the filter.
5. Open the bulk bar's More menu and choose Cancel.
6. **Expected:** the departed order is no longer selected, the count matches the rows in view, and only current orders are cancelled.
7. **Actual:** the departed order is still selected, the count includes it, and it gets cancelled.

**Problem:** The pruning branch runs only when `isAllSelected` is false, and the "All" branch no-ops whenever `isViewWhole` is false. So in All mode with a sieve active, the effect neither re-materialises nor prunes — it does nothing at all. On `develop` this could not happen: `retainedRowIds` was always `undefined`, which made `isViewWhole` unconditionally true.

**Evidence:** `useTableSelection.ts:274-304`:

```ts
const isViewWhole = data.length === currentDataIds.size;
const isAllReassertDeferred = isSameView && isAllSelected && !isViewWhole;
...
} else if (isAllSelected) {
  if (isViewWhole) {
    setSelectedRowIds(Object.fromEntries(data.map((row) => [row.id, true] as const)));
  }
  // <-- no else: a deferred All neither re-asserts nor prunes
} else {
  setSelectedRowIds((prev) => { /* prunes against currentDataIds */ });
}
```

Confirmed by execution against the real hook, where order `2` leaves the retained set while the view is sieved to row `1`:

```
A/step4 order 2 left the view: { isAllSelected: true, selected: [ '1', '2', '3' ],
                                 selectedOrderIdsPayload: [ '1', '2', '3' ] }
A/control manual ticks (not All):  selected: [ '1', '3' ]     <-- prunes correctly
A/develop-shape (no retainedRowIds): selected: [ '1', '3' ]   <-- prunes correctly
A/self-heal after un-sieve:          selected: [ '1', '3' ]   <-- recovers
```

The payload chain was read end to end rather than inferred: `Object.keys(selectedRowIds)` (`table.tsx:421`) → `useBulkActions.ts:36-40` → `provider.tsx:145` → `POST { orderIds }` in `services/mixtures/orders/getBulkActionsData/index.ts:12-15` → the fetched `orders` → `handleCancelOrder` (`provider.tsx:564-592`), which maps over those fetched orders and sets each to `Canceled`. There is no client-side re-validation against the current table data, and the fetch is by explicit id, so a departed order does come back and is acted on.

**Impact:** A bulk cancel can cancel an order that has left the view. Bounded, which is why this is medium rather than high: several toolbar actions are gated by `every`-style checks over `currentJobs`, so an out-of-band order tends to *hide* those pills rather than mis-apply them; the destructive path needs the More menu plus a confirmation modal; and the state self-heals when the sieve is lifted. The inflated count on the bulk bar is a reliable tell.

Worth noting the PR's own test at `useTableSelection.test.ts:978` ("drops a hidden selection once the order leaves the view") asserts exactly this contract for the non-All case. The All case violates a contract the PR itself established, so this is a gap rather than a deliberate choice.

**Fix:** Keep deferring the re-assert, but still prune. Give the All branch an `else` that intersects the existing map with `currentDataIds`, reusing the logic already in the final branch:

```ts
} else if (isAllSelected) {
  if (isViewWhole) {
    setSelectedRowIds(
      Object.fromEntries(data.map((row) => [row.id, true] as const))
    );
  } else {
    // Departed ids must still go, even while the re-assert waits for
    // the un-sieve — otherwise a bulk action reaches an order that has
    // left the view.
    setSelectedRowIds((previousSelected) =>
      Object.fromEntries(
        Object.keys(previousSelected)
          .filter((selectedId) => currentDataIds.has(selectedId))
          .map((selectedId) => [selectedId, true] as const)
      )
    );
  }
}
```

### 4. After a reload with Select All plus a sieve, the header checkbox lies — and then selects everything

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/hooks/useTableSelection.ts:274]**

> **In plain terms:** Select the whole list, narrow the view with a chip, then reload the page. The header checkbox comes back ticked, but nothing is actually selected and the bulk bar is missing. Worse: if you then tick a single row and clear the chip, the selection jumps from that one order to every order in the view — so you can be one click away from having the whole list armed for a bulk action while believing you picked one.

**Function/Class:** `useTableSelection` — the restore effect plus the cleanup effect

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. Open the orders table with a filter that returns several orders.
2. Tick the header checkbox to select the whole list.
3. Click a stats-bar chip so only some rows are shown.
4. Reload the page and wait for the orders to load.
5. **Expected:** either the header is unticked, or it is ticked and the bulk bar shows a matching count.
6. **Actual:** the header is ticked and not indeterminate, but nothing is selected and no bulk bar appears.
7. Now tick one visible row, then click `All` to clear the narrowing.
8. **Expected:** one order selected.
9. **Actual:** every order in the view is selected.

**Problem:** In All mode only the flag is persisted, with `ids: []`. On `develop` the cleanup effect unconditionally re-materialised the map from `data`; this PR gates that on `isViewWhole`, which is false while a sieve is active. Since the sieve is also `sessionStorage`-backed it survives the reload, so the re-assert is deferred indefinitely, leaving `isAllSelected: true` over an empty map. `SelectionHeader` renders checked off the flag alone (`cells/SelectionCell/Header.tsx:23-26`), and because nothing is selected, `indeterminate` is false too. The escalation follows from `toggleRowSelected(id, true)` preserving `isAllSelected`, so ticking one row leaves All latched over a one-entry map, and lifting the sieve fires the deferred re-assert across the whole view.

**Evidence:** Confirmed by execution, starting from `isLoading: true, data: []` to model a real page load:

```
B/step1 loading, nothing yet:       { isAllSelected: true, selected: [] }
B/step2 data landed while sieved:   { isAllSelected: true, selectedRowsCount: 0, selected: [],
                                      isHeaderFullyChecked: true, indeterminate: false,
                                      bulkToolbarVisible: false }
B/develop-shape after reload:       { isAllSelected: true, selectedRowsCount: 1, selected: [ '1' ] }
B/escalation after ticking one row: { isAllSelected: true, selected: [ '1' ] }
B/escalation after un-sieve:        { isAllSelected: true, selected: [ '1', '2', '3' ] }
```

`isHeaderFullyChecked` was computed with the exact expression from `Header.tsx:23-26`. The develop-shape line confirms this state was unreachable before the PR. Effect ordering was verified from source: the restore effect is declared at `:61` and the persist effect at `:356`, so restore reads storage before persist can overwrite it; and with `initializeWithValue: false` the sieve is read back one render after mount, far sooner than the orders query resolves.

**Impact:** A misleading selection UI after every such reload, and a real path to arming a bulk action over the whole view while believing one order is selected. Not covered by any of the 30 existing tests — the `retainedRowIds` block never seeds `sessionStorage`.

**Fix:** Do not let the flag outlive its map. Either drop `isAllSelected` on restore when a sieve is active, or downgrade it in the deferred branch when the selection map is empty:

```ts
if (isAllReassertDeferred && !Object.keys(selectedRowIds).length) {
  setSelection((prev) => ({ ...prev, isAllSelected: false }));
}
```

Fixing Issue 3's missing `else` does not fix this one — they are separate branches of the same effect and need separate handling.

### 5. `Active` and `Overdue` in a breakdown are told apart by colour alone

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/OrdersStatsBar/consts.ts:76]**

> **In plain terms:** When you open a job type's breakdown, two of the counts are marked only by a small coloured dot with no wording next to it — one green for work in progress, one red for work that has run past its deadline. To someone with red-green colour blindness these are the same shape, the same size and near-identical shades, so there is no way to tell which number is which without hovering the mouse over each one and waiting for a tooltip. That is roughly one in twelve men.

**Function/Class:** `buildJobTypeStat` → `StatChip`

**Severity:** medium

**Confidence:** high

**WCAG:** 1.4.1 Use of Color (Level A)

**Steps to reproduce:**

1. Open the order dashboard with a saved view or filter applied so the stats bar appears.
2. Click a job type chip, for example Service, to open its per-status breakdown.
3. Look at the Active and Overdue chips without hovering.
4. **Expected:** each count is identifiable by something other than hue — a word, an abbreviation, or a different shape.
5. **Actual:** both are a 6px circle of identical geometry, differing only in fill. The status name appears only in a mouse-hover tooltip.
6. Re-check through Chrome DevTools → Rendering → Emulate vision deficiencies → deuteranopia.
7. **Expected:** the two remain distinguishable. **Actual:** they render as `#999976` and `#86864E` — a 1.30:1 difference, effectively the same olive dot twice.

**Problem:** The two assets are byte-identical apart from the fill, and both are routed through `DOT_ICONS` so they render at the same size in the same 24px slot. Every breakdown child is built with `isIconOnly: true`, and `StatChip/index.tsx:57` renders `{isIconOnly ? null : <Styled.Text>{label}</Styled.Text>}`, so the word never appears. There is no size, shape or position difference either: `visibleChildren` is filtered to counts above zero, so in the common case where only Active and Overdue have rows, the two dots sit adjacent.

**Evidence:** `assets/svg/icons/dot-active-green.svg:2` is `<rect x="5" y="5" width="6" height="6" rx="3" fill="#00B373"/>` and `dot-active-red.svg:2` is the same rect with `fill="#DC3855"`. Measured dot-to-dot contrast: 1.62:1 with normal vision, **1.30:1 simulated deuteranopia**, 1.19:1 tritanopia. Neither the `aria-label` nor the `title` rescues it — `aria-label` is not a *visual* means (that is 1.3.1's territory, not 1.4.1's, and WCAG failure F73 is explicit about it), and a native `title` is hover-only, so it meets neither half of technique G183's "hover **and** focus" requirement.

**Impact:** Colour-blind admins cannot read the breakdown — which is the feature's primary drill-down — without hovering each chip individually. A mouse user can recover the meaning, which is why this is medium rather than high; a touch user cannot.

**Fix:** Give Overdue a shape that differs from Active rather than only a colour. `Overdue` is already modelled as a flag rather than a status (`consts.ts:51-54` says so), so a clock or warning glyph would be both accessible and more truthful. Alternatively drop `isIconOnly` for these two statuses so the label renders — `Styled.Bar` already has `flex-wrap: wrap`, so the extra width is safe.

One correction to the PR's own note on this: on `develop`, `components/atoms/JobStatus/utils.ts:37-41` already swaps green for red on the `Active` case, so overdue-vs-not was *already* colour-carried there — but that component always renders a visible label alongside. What this PR newly introduces is a dot with **no label at all**, and two differently-coloured dots side by side in one row.

### 6. The `Active` dot fails non-text contrast, and the obvious fix makes Issue 5 worse

**[File: apps/creative-portal/assets/svg/icons/dot-active-green.svg:2]**

> **In plain terms:** The green dot that marks the "currently active" count is too faint against the white heading to be reliably made out by someone with reduced contrast sensitivity, and it gets fainter still when the chip is hovered or selected. Since that dot is the only thing indicating what the number means, anyone who cannot resolve it just sees an unlabelled number.

**Function/Class:** `Styled.Icon` in `StatChip/styles.ts` — the dot assets bypass its colour pinning

**Severity:** medium

**Confidence:** high

**WCAG:** 1.4.11 Non-text Contrast (Level AA)

**How to spot it:** Open a job type's breakdown and sample the green dot against the surrounding heading background with any contrast checker, then hover the chip and re-measure. Compare with the red dot on the same row, which passes.

**Problem:** `#00B373` against the sticky heading's white reaches **2.729:1**, below the 3:1 floor; on the hover and selected surface (`navyBlue5` composited over white, `#F7F8FA`) it drops to **2.574:1**. The dot is the sole visual carrier of what its number counts, so the "decorative graphic" exemption does not apply — remove it and the chip conveys nothing. `Styled.Icon` pins `currentColor` glyphs to `primary`, but the two dot assets hard-code their own `fill` and bypass that, so the green dot is the only glyph in the bar under 3:1.

**Evidence:** Ratios recomputed from the assets and `packages/shared/theme/theme.tsx`:

| Foreground | Background | Ratio | Passes 3:1? |
| --- | --- | --- | --- |
| Active `#00B373` | `#ffffff` heading | **2.729:1** | ✗ |
| Active `#00B373` | `#F7F8FA` hover/selected | **2.574:1** | ✗ |
| Overdue `#DC3855` | `#ffffff` | 4.430:1 | ✓ |
| `green2 #008F5C` | `#ffffff` | 4.131:1 | ✓ |
| Other glyphs `primary #001E62` | `#ffffff` | 15.418:1 | ✓ |

**Impact:** Low-vision users cannot perceive the only status indicator on the Active chip in every breakdown. The miss is real but marginal in magnitude — about 9% short of the floor — which is why medium rather than high.

**Fix:** **Do not apply the `green2` swap on its own.** It is numerically correct for this issue (4.131:1 on white, 3.897:1 on the selected surface) but it makes Issue 5 materially worse: under simulated deuteranopia `#008F5C` becomes `#7A7A5E` against the red's `#86864E`, a dot-to-dot difference of **1.157:1**, worse than the current 1.301:1. Darkening the green trades an AA contrast failure for a deeper Level A colour-only failure.

Fix them together: resolve Issue 5 first by giving the two statuses distinguishable non-colour forms, then pick a green that also clears 3:1. `green2` is the right token once the shapes differ.

### 7. Icon-only chips have no label a sighted keyboard user can reach

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/OrdersStatsBar/partials/StatChip/index.tsx:48]**

> **In plain terms:** Seven of the chips show only a symbol and a number. Their name appears in a mouse-hover tooltip, which browsers never show on keyboard focus. So a keyboard user tabs to a chip, sees a focus ring around a dot and a number, presses Enter, and the table changes — without ever learning which filter they just applied.

**Function/Class:** `StatChip`

**Severity:** low

**Confidence:** high

**WCAG:** 3.3.2 Labels or Instructions (Level A)

**How to spot it:** Tab through the stats bar in Chrome without using the mouse. The top-level Overdue and On Hold chips, plus every breakdown chip, show a glyph and a number with no word anywhere on screen at any point.

**Problem:** `title={isIconOnly ? label : undefined}` paired with `{isIconOnly ? null : <Styled.Text>{label}</Styled.Text>}` means the label exists only as a hover tooltip. Chrome, Safari and Edge show `title` on pointer hover only; Firefox alone surfaces it on focus.

**Evidence:** `StatChip/index.tsx:48` and `:57`, with `isIconOnly: true` set on the top-level Overdue and On Hold (`consts.ts:100`, `:107`) and on every breakdown child (`:77`, `:86`). The component's own docblock concedes the point: "the hover tooltip, which is the only place the word appears".

**Impact:** Keyboard-only sighted users cannot identify seven or more of the bar's controls before or after activating them. Rated low rather than higher because the accessible name is correct, so screen-reader users are unaffected, and native `title` tooltips are explicitly exempt from 1.4.13 — the gap is specifically between "labelled for assistive tech" and "labelled for sighted keyboard users".

**Fix:** The PR's stated reason for avoiding the shared atoms is accurate and I verified it: `packages/shared/components/atoms/Tooltip/index.tsx:100` is click-toggled, and `PopoverOnHover/index.tsx:57` hard-codes `id="popover-container"`, so six chips would emit six duplicate DOM ids. That makes this a shared-package fix rather than a local one. Since PP-2041 needs the same treatment, the cheapest correct path is to fix the hard-coded id in `packages/shared` and use the popover on focus as well as hover. Deferring to PP-2041 with a ticket is reasonable; shipping `title` permanently is not.

### 8. The stats-bar session key is not user-scoped, and nothing clears storage on logout

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/consts.tsx:232]**

> **In plain terms:** The bar remembers your chosen narrowing for the browser session, but it files that memory without recording who it belongs to, and signing out does not clear it. On a shared workstation, the next person to sign in can find their dashboard already narrowed by the previous person's choice.

**Function/Class:** `STORAGE_KEYS.STATS_BAR`

**Severity:** low

**Confidence:** high

**Steps to reproduce:**

1. Sign in as admin A, open a saved view, and narrow the bar to a job status.
2. Sign out. Without closing the tab, sign in as admin B.
3. Have B open a saved view whose id matches the one A was using.
4. **Expected:** B's dashboard opens unnarrowed.
5. **Actual:** B's dashboard opens pre-narrowed to A's choice.

**Problem:** The new key is a bare literal, unlike its sibling `getSelectedIdsStorageKey(userId)` two lines below, and the logout path (`contexts/userContext/provider.tsx:53-57`) only calls the logout mutation and redirects — there is no `sessionStorage.clear()` anywhere in the app.

**Impact:** Genuinely low. The stored values are ids from a static config (`"all"`, `"job:AI:Assigned"`) — no order ids, customer names, emails, or auth material — and the narrowing only ever operates over B's own authorised result set, so nothing of A's data is exposed. The observable leak is the set of numeric saved-view ids A visited, plus a confusing pre-narrowed dashboard. The selected chip is visibly highlighted, so there is a cue.

**Fix:** Scope it like its sibling — a `getStatsBarStorageKey(userId)` fed from `useUserContext()`. Better for the whole class of bug: clear `sessionStorage` in the `logout` callback, which would also fix the pre-existing unscoped `ordersTableFilters`.

### 9. The corrupt-storage fallback never reaches Sentry

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/OrdersStatsBar/consts.ts:177]**

> **In plain terms:** This PR fixes a crash where bad saved data took the whole dashboard down. The fix now quietly recovers instead of crashing — but it only writes a note to the browser's own console, which nobody monitors. If the same corruption starts happening again, the team will not find out.

**Function/Class:** `parseStatsBarStore`

**Severity:** low

**Confidence:** high

**How to spot it:** Code health, not user-reproducible — the recovery path works correctly. `catch (error) { console.warn(...); return EMPTY_STATS_BAR_STORE; }` with no Sentry capture.

**Problem:** The house pattern for client-side non-fatals is `reportError` (the wrapper over `Sentry.captureException` at `packages/shared/utils/throwSentryError.ts:214`, which adds central scrubbing and an `operation` tag), used at `AppErrorBoundary/index.tsx:42` and `AiFeedbackPanel/utils.ts:57`; `PaymentDetailsStep2/hooks.ts:149` uses `Sentry.captureMessage(..., { level: "warning" })` for exactly this shape of non-fatal guard.

**Impact:** A recurrence of the bug this PR was written to fix would degrade the feature silently with no event to group on.

**Fix:** `reportError(error, { operation: "orders-stats-bar.parse-store" })`. Note the immediate neighbours at `useTableSelection.ts:100` and `:376` also use bare `console.warn`, so this is convention drift rather than a regression introduced here.

**PII check: clean.** Only the caught error is logged, never the stored value, and the stored value holds no order or customer data. Nothing new goes to Sentry at all.

### 10. Two tests cannot fail

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/index.test.tsx:231]**
**[File: .../partials/OrdersStatsBar/partials/StatChip/styles.test.tsx:189]**

> **In plain terms:** Two of the new tests are written so that no change to the code could ever make them fail. They read as coverage of a rule the ticket cares about, but they are not checking anything.

**Function/Class:** `does not surface Save View while a stat is clicked`; `keeps its styling flags off the button element`

**Severity:** low

**Confidence:** high

**How to spot it:** Code health. In `index.test.tsx`, `./hooks` is mocked at `:36` and `renderTable` pins `isAnyFilterSelected = true, isSameView = true`; the Save View gate is `isAnyFilterSelected && !isSameView`, and `onStatClick` is a no-op `vi.fn()`. So the click plus the follow-up `expect(queryByTestId("save-view-control")).not.toBeInTheDocument()` re-reads two frozen constants. In `styles.test.tsx`, asserting `button.hasAttribute("isselected") === false` tests Emotion's own prop filtering for string tags — there is no `shouldForwardProp` in `styles.ts` for it to guard.

**Impact:** No false confidence about Req 3.2 overall — the genuine proof lives at `hooks.test.ts:194` and `useOrdersStatsSieve.test.ts:351`, both real and both mutation-proven. But the dead assertions read as component-level coverage that is not there.

**Fix:** Drop the click and the second assertion in the first, or rename it to what it proves ("the Save View gate reads only filter state"). Delete the second, or keep it with a comment saying it is a React-warning regression guard.

For balance: the rest of `StatChip/styles.test.tsx` earns its place, contrary to what 204 lines of style assertions suggests. `holds the icon slot at 24px whatever the glyph size` does real arithmetic and sources the glyph size from `STATUS_ICON_SIZE` so a design tweak does not false-fail it; `draws its own ring on keyboard focus` walks Emotion's inserted `cssRules` because jsdom never applies `:focus-visible`, which is the correct technique for a real WCAG 2.4.7 regression. The only brittleness is four hard-coded `rgb()` literals that a palette change would break with no bug present — cheap to fix by deriving them from `theme.colors.*` as line 174 already does.

### 11. The bar shows a row of hard zeros while a new filter loads

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/index.tsx:116]**

> **In plain terms:** Change a filter or switch saved view and, for one fetch, every count in the bar reads zero — which looks exactly like an empty queue rather than "still loading". The skeleton rows in the table underneath do give the game away, so this is mild.

**Function/Class:** `isStatsBarVisible` / `countOrderStats`

**Severity:** low

**Confidence:** high

**Steps to reproduce:**

1. Open the dashboard with a filter applied and note the counts.
2. Change a filter, or switch to a different saved view, and watch the bar during the fetch.
3. **Expected:** the previous counts hold, or the counts show a loading treatment.
4. **Actual:** all eight read 0 until the new data lands.

**Problem:** A filter change mints a new React Query key and the query has no `keepPreviousData`/`placeholderData`, so `data` is `undefined` under the fresh key, `ordersResult` is `[]`, and `countOrderStats([], config)` seeds every id at 0 and never increments. The `isLoading` guard inside `useOrdersStatsSieve` suppresses only *reconciliation* — it does nothing to the counts or the render.

**Impact:** A one-fetch window where the bar misreports an empty view. Mitigated in practice: `table.tsx:760` renders skeleton rows during the same window, so the loading context is visible. The 30s poll and the focus refetch reuse their key and keep their data, so they never zero anything — only a filter or saved-view change does, which is exactly when a reader is looking.

**Fix:** Thread `finalLoadingState` into the bar and either skeleton the counts via `@proofed/shared/components/atoms/LoadingWrapper` (per the repo's reuse-first convention) or dim them. `keepPreviousData: true` on the orders query would also hold the previous counts through the transition, but that query is not this PR's to own — the author flagged this rather than changing it, which was the right call.

### 12. Minor convention and simplification items

**Severity:** low · **Confidence:** high · **How to spot it:** all code health, none user-reproducible.

- **`findStat` walks the whole tree after a match** (`OrdersStatsBar/utils.ts:85`) and `countOrderStats` re-runs `forEachStat` once per row (`:131-137`). No shared tree-walk helper exists to reuse, so this is simplification rather than missed reuse. `const flattenStats = (config) => config.flatMap((stat) => [stat, ...(stat.children ?? [])])` collapses all three helpers, and hoisting one `flattenStats(config)` above the row loop removes the repeated traversal.
- **`CLOSED_ORDERS_STATS_CONFIG` is permanently empty** (`consts.ts:120`), so `getOrdersStatsConfig`, the `isFilteringClosedOrders` param and `hasStats` all collapse to one negation today. Deliberate PP-2041 groundwork and documented as such — fine to keep, worth knowing it is scaffolding.
- **`StatsBarRow` is a single-declaration styled div** (`TableWithFilters/styles.ts:163`) two lines from a sibling that expresses the same concern as a theme-ui `pb` prop. `<Box pb="0.75rem">` in `index.tsx` would be more consistent with its neighbour.
- **`SESSION_STORAGE_OPTIONS`** (`consts.ts:183`) has a fully generic name among siblings that all carry the feature (`EMPTY_STATS_BAR_STORE`, `DEFAULT_STATS_BAR_STATE`, `parseStatsBarStore`). `STATS_BAR_SESSION_STORAGE_OPTIONS` reads better at the import site.
- **`index.test.tsx:196` walks `parentElement!.parentElement!.parentElement!`** to find the pinned heading; any wrapper added or removed breaks it with no bug present. Give `Styled.StickyHeading` a `data-testid`.
- **Naming mismatch:** `useOrdersStatsSieve.test.ts:219` is named `sieves to a whole job type when its chip is the selection` but its body asserts that clicking a job type sieves *nothing*. The inline comment explains why; the name says the opposite of the assertion.

---

## Open Questions

These did not survive verification as defects, or are decisions rather than bugs. Several are already in the PR body, which is to the author's credit — I am recording my assessment, not re-raising them.

- **A group of five re-describes itself as "2 Orders" while sieved.** Confirmed by execution (`count` 5 → 2, deadline range narrowing) and no test covers it. But I do not think it should be reported as a defect: Req 3.2 pairs "row content" with "any filter, or the saved view", which reads as a filter-isolation requirement about an order row's own fields rather than about a derived group-summary row; both alternatives are worse (a stale "5 Orders" above two rows, or orphaned headers); and the author already put this in writing twice, including as open question #1 for Nicola. **Needs a PO decision, not a code change.** — `hooks.ts:564`
- **Does a top-level sieve surviving a filter change match the intent of Req 5.1?** The mechanism is real: `Overdue`/`On Hold` are always "still valid" because they exist in the config, so the narrowing persists across a filter change and into batch/order-group mode via the shared `"ad-hoc"` bucket. I checked whether this is a defect and concluded it is not — Req 3.3 forbids dropping a sieve just because it matches nothing, `useOrdersStatsSieve.test.ts:487` asserts exactly this, and the selected chip stays visibly highlighted throughout. The genuinely unspecified part is batch mode, which the ticket never mentions and the author already flagged. If the PO wants a batch drill-in to start clean, folding `batchName`/`orderGroupId` into the view token is a one-line change. — `OrdersStatsBar/consts.ts:143`
- **A collapsed job type reports "collapsed" while still showing one child chip.** I had this as a possible WCAG 4.1.2 failure and it does not survive: with no `aria-controls` and no grouping role, `aria-expanded="false"` makes no formal assertion about a sibling, and both controls expose an accurate name, role and state. The sharper version is narrower and not a conformance issue: when a job type has exactly one non-zero status that also holds the selection, expanding and collapsing render identical content, so the chip reports a state change that produced no visible change. Worth a look, not worth blocking. — `OrdersStatsBar/utils.ts:261`
- **Should a sieve announce itself to a screen reader?** There is no `role="status"` anywhere in the tree, so activating a chip changes the table silently. Strictly, 4.1.3 does not require inventing status messages where none are presented visually, and `aria-pressed` does convey that the toggle flipped — so a strict auditor could rule it not applicable. The harm is still asymmetric (sighted users see the row change instantly) and the fix is a few lines. Author's instinct here is right. — `index.tsx:164`
- **Focus can be dropped when a breakdown child unmounts under the poll.** A revealed child is filtered out the moment its count hits zero, so a keyboard user standing on it loses focus to the document. Narrow window, low stakes, and probably not worth a focus-restoration harness — but it is a consequence of the 30s poll worth knowing about. — `OrdersStatsBar/utils.ts:258`
- **Sieving discards a manual group collapse.** The author discloses this as pre-existing (`mergeExpandedGroupIds` treats a group absent from the previous data as new and force-expands it) and states it is byte-identical to develop. I agree it is pre-existing; the sieve simply triggers it far more often, so it may now be worth a follow-up ticket. — `table.tsx:123`
- **`retainedRowIds` remains externally overridable.** It is a member of `OrdersTableProps` and `{...props}` is spread *after* `retainedRowIds: unsievedRowIds`, so a parent could override the internal value. No caller does, so this is latent only — but destructuring it out of `props` as `batchName`/`orderGroupId` are would make it impossible. — `index.tsx:188`
- **`msw` is not a declared dependency anywhere in the repo**, so 14 story files fail typecheck on a clean checkout, including this PR's own `TableWithFilters/index.stories.tsx`. Fully pre-existing (identical on the current `develop`-based branch) and not this PR's to fix, but someone should own it. — `apps/creative-portal/package.json`
- **`packages/eslint-config/index.js:84` may be a no-op.** It passes `"optionality-order": "required-first"` to `perfectionist/sort-interfaces`; recent plugin versions renamed that option to `groupKind`, in which case it is silently ignored and the rule enforces flat alphabetical instead. Every interface in this PR follows CLAUDE.md regardless, so nothing here is affected — but the config itself is worth checking. — `packages/eslint-config/index.js:84`
- **The orders query's inline `select` defeats memoisation** (`hooks.ts:334`), so `ordersResult` is a fresh array every render and the new fact index, counts, sieve and retained-id set all recompute on every render — including every checkbox tick. At 500 rows that is roughly 19,000 predicate calls per interaction. Pre-existing (the `select` was already inline) and the author acknowledges it in a comment, but this PR is what puts real work behind it. Wrapping the `select` in `useCallback` is a one-line change that would let these memos — and the pre-existing sorts and grouping — actually memoise. — `hooks.ts:334`

---

## Verified clean (checked and found no issue)

Recording these so the same ground is not re-covered, and because several are places a reviewer would reasonably expect a problem:

- **Prototype pollution through the `sessionStorage` round-trip: not reachable.** Confirmed by execution, not by argument. Both hops use `CreateDataProperty` rather than `Set`, so a `__proto__` key from `JSON.parse` stays an own data property: `JSON.parse('{"__proto__":{"polluted":"yes"},…}')` yields own props `['__proto__','ad-hoc']` with `({}).polluted === undefined`, and the object spread at `useOrdersStatsSieve.ts:158` preserves that. The lookup key cannot reach `__proto__` harmfully either — it is `String(viewId ?? "ad-hoc")` over a `number`, and even a tampered read returns `Object.prototype`, which destructures to `undefined`/`undefined` and falls back to `All`.
- **`retainedRowIds` does not weaken an authorisation boundary.** Every id in a bulk payload originates in `ordersResult` — the server's own response to this user's filtered search — because `unsievedRowIds` is built from it. The sieve narrows that set; it can never widen it. Issues 3 and 4 are knowing-consent and correctness problems, not privilege escalation.
- **The `isCurrentJobOverdue` extraction is exactly behaviour-preserving.** `currentDate` is a function-local `new Date()` passed explicitly at the only filter call site, so the `now = new Date()` default is never silently used there.
- **The `ORDER_STATUS_ICONS` extraction is a pure move** — all eight status/icon pairs byte-identical, `ORDER_STATUS_OPTIONS` untouched, and `FINISHED_STATUSES`/`UNFINISHED_STATUSES` unchanged.
- **The crash fix is complete.** Every read path funnels through the deserializer, including the mount effect, both storage listeners, and `setValue`'s functional-update read. Junk under a valid view token (`42`, `null`, a string, an array) is absorbed on every branch.
- **No new request surface.** The sieve is genuinely absent from the query key (`[SEARCH_ORDERS_FOR_TABLE, payload]`, no `selectedStatId`), and `useOrdersStatsSieve` imports no axios or query hook. Zero added `axios|fetch(|useQuery|useMutation` in the diff.
- **No XSS sinks, no secrets.** Zero `dangerouslySetInnerHTML|innerHTML|eval|new Function|document.write` in the added lines; all dynamic values reach the DOM as JSX text or `aria-label`/`title`, sourced from a static config rather than order data. No `process.env`/`NEXT_PUBLIC_` or credential-shaped additions.
- **No `packages/shared` or `packages/wysiwyg` file is touched**, so there is no cross-app breaking change. `TableWithFilters` has a single page consumer.
- **`useStickyHeadingHeight` absorbs the taller heading** — it measures via `getBoundingClientRect` + `ResizeObserver` and the only consumer is a `calc()`, so no fixed-height assumption exists.
- **Conventions are in good order.** All eleven new/changed interfaces satisfy required-first-then-optional with each half alphabetical; import sorting matches the perfectionist config including the trailing `styles` group; `VoidFunction` correctly not used (no parameterless callback is introduced); no empty styled components, no `line-height: 0`, no stray centring on single-SVG icon wrappers; `index.tsx` files are UI-only; stories follow CSF3 with sentence-case titles and no external URLs; no `console.log`, `@ts-ignore`, `eslint-disable`, commented-out code, or ticket-less TODO.
- **`OrdersStatsBar/mocks.ts` does not ship in the bundle.** It matches an established convention (10 such files in the repo) and is imported only by tests — `index.stories.tsx` builds its own fixtures.
- **The native-`title` decision is justified**, not laziness: the shared `Tooltip` is click-toggled and `PopoverOnHover` hard-codes a DOM id. Both verified by reading the source.
- **No existing stats bar is duplicated.** `batchStats` is a per-batch percent-complete row; `TinyCounter`, `Chips` and `BufferChip` are none of them a selectable stat chip.

---

## Validation Checks

Scope: **`@proofed/creative-portal` only** — every changed file lives in that workspace and no shared package is touched, so a filtered run is the correct coverage.

Run in a detached worktree at the PR head (`3e5981d7f`). One environment note: a fresh worktree lacks the gitignored, Next-generated `apps/creative-portal/next-env.d.ts`, whose absence produced 9 spurious `TS2307` errors on image imports in files this PR never touches. I restored that generated file and re-ran, then diffed the error set against a baseline run on the pre-existing checkout.

| Check | Result | Notes |
| --- | --- | --- |
| `npx turbo run typecheck --filter=@proofed/creative-portal` | ✅ | 14 errors, **byte-identical to the baseline** (`diff` clean) — all `msw`-missing in story files, all pre-existing. **0 introduced by this PR.** No error lands in any PR-changed file except the pre-existing `msw` import in `TableWithFilters/index.stories.tsx`, which is present on develop too |
| `npx turbo run lint --filter=@proofed/creative-portal` | ✅ | Clean at `--max-warnings 0`. 0 errors, 0 warnings |
| `npx turbo run test --filter=@proofed/creative-portal` | ⏳ | see below |
| `npx turbo run build --filter=@proofed/creative-portal` | ⚠️ | **Blocked by the same pre-existing `msw` gap, not by this PR.** Exit 1 with exactly one failure: `./components/molecules/SearchBar/index.stories.tsx:4:36 — Type error: Cannot find module 'msw'`, in a file this PR never touches. No other compile or module error. `next build` type-checks as part of the build, and the typecheck error set was already proven byte-identical to the baseline, so this failure is transitively pre-existing. Root cause is a stale local install: `msw@^2.13.4` is declared in `apps/storybook/package.json` and present in `yarn.lock`, but is not installed anywhere on disk in either checkout — a root `yarn` would resolve it |

Independently, the scoped suite the author cited was reproduced during the review: **58 files / 512 tests green** for `TableWithFilters`, and all four core mutation checks were caught (see Issue 2 for the two that were not).

---

## Tests

- ✅ Tests exist for all new code, and they are mostly good — the pure-function split makes `utils.test.ts` (533 lines) and `useOrdersStatsSieve.test.ts` (570 lines) genuinely behavioural rather than snapshot-shaped.
- ✅ The load-bearing behaviours are mutation-proven, which is unusual and worth crediting. Neutering `applyOrdersStatsSieve` fails 7 tests; neutering `reconcileStatsBarState` fails 7; counting from the narrowed set (Req 2.5) fails 3; replacing the header-checkbox merge with a plain replace fails 1; deriving the retained ids from the sieved rows fails 1; disabling the deferred All re-assert fails 1.
- ✅ Edge cases are well covered: absent `liveStatus`, half-absent status, unknown job name, AI Pre/Post collapse, missing and empty `currentJobReturnTime`, a row missing from the facts index, the empty closed-orders config, the `isLoading` branch, and corrupt `sessionStorage` across seven shapes (`null`, `"undefined"`, bare string, number, array, malformed JSON, empty string). `counts the orders inside a group, not the group's own row` cross-checks against the real grouping utils, which is a genuine off-by-group-header trap.
- ✅ Req 3.2's filter isolation is proven twice, including a pin on the hook's whole return surface — a better choice than a name regex.
- ❌ **The `retainedRowIds` wiring has zero test execution end to end** (Issue 2). Both wiring points can be deleted with 512/512 green.
- ❌ **Issue 1 is uncovered, and the test that appears to cover it does not.** Removing the interposed `onStatClick` from `does not resurrect a selection reconciliation has dropped` makes it fail — so it proves the write path composes from reconciled state, not what its name promises.
- ❌ **Issue 3's deferred-prune branch is uncovered**, as is Issue 4's reload path (the `retainedRowIds` block never seeds `sessionStorage`). The sibling mutation of reverting `currentDataIds.size > 0` to `data.length > 0` was also not caught — the same untested corner.
- ⚠️ Two assertions cannot fail (Issue 10). Req 1.1's "shown with zeros when a filter returns nothing" half is untested — the only `orders: []` test sets `isLoading: true`. Structurally safe, but one `renderSieve({ orders: [] })` assertion would close it.
- ⚠️ Req 3.1 is only weakly covered: `selectedIdOf` uses `.find()`, which would mask a second selected chip. Structurally impossible anyway, so I would not spend a test here.

### Suggested manual QA script

1. **Issue 1 (highest priority).** Narrow to a job status with exactly one matching order. Wait for that order to move out of the status — the table should return to the full list. Then wait up to 30 seconds, or switch tabs and back, for an order to enter that status again. **The table must not re-narrow on its own.**
2. **Issue 3.** Select the whole list with the header checkbox, then narrow with a chip. Wait for the background refresh after one of the selected orders leaves the filter. Check that the count on the bulk bar matches the rows you can see, and that a bulk action does not reach the departed order.
3. **Issue 4.** Select the whole list, narrow with a chip, reload the page. The header checkbox must not show ticked with nothing selected. Then tick one row, clear the chip, and confirm exactly one order is still selected.
4. **Issues 5 and 6.** Open a job type's breakdown and view it through Chrome DevTools → Rendering → Emulate vision deficiencies → deuteranopia. Confirm you can still tell the Active count from the Overdue count.
5. **Issue 7.** Tab through the stats bar using only the keyboard. Confirm you can tell what each chip does before pressing it.
6. **Issue 11.** Change a filter and watch the bar during the fetch — note whether the row of zeros reads as "empty" or as "loading".
7. **Ticket acceptance criteria not covered by automated tests:** apply a view or filter that returns no orders and confirm the bar shows with all counts at 0 and the list shows its empty state (scenario 6); confirm with no view or filter applied the bar is hidden.
8. **Ticket dependency:** the job-type counts cannot be signed off until PP-2027 resolves, per the ticket's own Dependencies note. Re-verify all counts once PR #2427 lands, since it caps the orders response at 500 rows and every count under-reports on larger views until then.

---

## Summary

| Aspect | Status |
| --- | --- |
| Correctness | ⚠️ Three confirmed behavioural bugs (Issues 1, 3, 4), all execution-proven; the core counting and sieving logic is sound |
| Requirements coverage | ⚠️ 14 of 18 fully addressed; Req 3.2, 5.1, 5.2 and VR 1 partial |
| Regression risk | ⚠️ Medium — confined to `useTableSelection`, whose only production caller is updated; the two extractions are verified behaviour-preserving and no shared package is touched |
| Tests | ⚠️ Strong where they exist and mutation-proven, but the headline selection guarantee has no end-to-end execution and all three confirmed bugs are uncovered |
| Accessibility | ⚠️ One Level A and one AA failure on the new controls (Issues 5, 6, 7); keyboard operation, focus ring, roles and accessible names are otherwise correct |
| Error handling | ✅ Crash fix verified complete; corrupt input narrowed on every path |
| Security | ✅ Prototype pollution not reachable (proven by execution); no XSS sink, no secret, no new request surface, no authorisation weakening. `/security` was run by the author; the two later production changes are covered by this review |
| Code quality | ✅ Genuinely good — pure-function core, data-driven composition, conventions clean, comments that explain *why* |
| Validation suite | ⚠️ typecheck ✅ (0 introduced), lint ✅; test and build pending at time of writing |
| Mergeable state | ✅ Clean, but **6 commits behind `develop`** — needs a merge before it goes in |

---

## Recommendation

**Request changes** — for Issue 1, which is a user-visible bug in the feature's core interaction, and Issue 2, which leaves the PR's own headline guarantee unheld by any test.

This is high-quality work. The pure-function core, the data-driven composition, the reuse of the Status filter's own overdue predicate, and the isolation boundary on `useOrdersStatsSieve` are all the right calls, and the PR body is unusually honest — several of the things I checked hardest were already disclosed in it. Four of the six issues below are edges of a fix that was itself correct to make.

**Before merge:**

1. **Persist the reconciled state (Issue 1).** A cleared sieve must stay cleared. Add the effect that writes `reconcileStatsBarState`'s result back when it differs from what is stored, and split the misleading test into one that covers the no-click path — it will fail until the source is fixed.
2. **Cover the `retainedRowIds` wiring (Issue 2).** Two assertions, one per boundary. The mechanism works today; nothing is holding it there.
3. **Prune in the deferred branch (Issue 3).** A bulk cancel reaching an order that has left the view is worth closing even though the window is bounded and self-healing.
4. **Stop the flag outliving its map (Issue 4).** Separate branch from Issue 3 — fixing one will not fix the other. The escalation (one tick plus an un-sieve selects everything) is the part that matters.

**Before merge, needs a design answer rather than code:**

5. **Issues 5 and 6 together.** Do not apply the `green2` swap on its own — it fixes the contrast failure and deepens the colour-only failure. Give Active and Overdue distinguishable shapes first, then take `green2`. This needs Nicola, and it is the author's own open question #5 with a measured answer attached.

**Can follow up, with a ticket:**

6. Issue 7 (icon-only labels) requires the `packages/shared` popover id fix, and PP-2041 needs the same treatment — sensible to do once, there.
7. Issues 8 through 12, plus the Open Questions. Clearing `sessionStorage` on logout (Issue 8) would fix a pre-existing sibling too, and wrapping the query's `select` in `useCallback` is a one-line win for the whole table.

**Also required by CLAUDE.md before merge:** rebase or merge `develop` (6 commits behind) and re-run validation. The author's `/security` review predates the header-checkbox fix and the storage deserializer; both are covered above and neither adds a request surface, so a re-run is not needed on my reading.

Two ticket-level notes for whoever signs this off: the job-type counts cannot be accepted until **PP-2027** resolves (the ticket says so), and every count under-reports until **#2427** lifts the 500-row cap. Finally, **PP-2040 scenario 6 and PP-2041 VR 4 contradict each other** on the empty-view case — this PR follows PP-2040, which is the right call for this ticket, but PP-2041 will need reconciling.
