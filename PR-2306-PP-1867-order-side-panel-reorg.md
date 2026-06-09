# PR Review: feature/PP-1867: Reorganize order side panel

**PR:** https://github.com/Proofed/B2BWebserver/pull/2306
**Jira:** https://proofed.atlassian.net/browse/PP-1867
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1. Job cards moved directly below the customer information section | `Styled.JobsContentWrapper` moved up in `OrderManagment/index.tsx`, now sits between `GeneralOrderInfo` and the `DetailedOrderInfo`/Brief block | ✅ Addressed |
| 2. Job cards no longer at the bottom of the panel | The bottom `JobsContentWrapper` was removed; jobs now render mid-panel | ✅ Addressed |
| 3. Order ID + external reference replace content type/industry in the title area | `OrderTitleInfo` now gets `title={externalReference}` and `subtitle={order.id}` | ✅ Addressed |
| 4. External reference single line, ellipsis-truncated by default | `StyledHeadingTitle` applies `white-space: nowrap; overflow: hidden; text-overflow: ellipsis` while not expanded | ✅ Addressed |
| 5. Hover turns the full text green (interactive cue) | `StyledHeadingTitle` `&:hover { color: green1 }` gated on `isInteractive` | ✅ Addressed |
| 6. Clicking expands to full text | `OrderTitleInfo` `toggleExpanded` → `isExpanded` → `white-space: normal; word-break: break-word` | ✅ Addressed |
| 7. Clicking again collapses | Same boolean toggle | ✅ Addressed |
| 8. Content type + industry as first two Brief items | `Brief` now receives `workItemType` (renders "Type:") and `workItemSubject` (renders "Industry:"), which are already the first two list items | ✅ Addressed |
| 9. Created by / Created for shown in the top info segment | Added as `DetailsList` items in `GeneralOrderInfo` (DetailsList is `display:flex; flex-wrap:wrap` with column items → renders the horizontal "people row" matching Figma) | ✅ Addressed |
| 10. Hide Created by / Created for when they have no value | `hidden: !createdById` / `hidden: !createdForId` | ✅ Addressed |
| 11. Remove Created by / Created for from the third segment | Removed from `DetailedOrderInfo` (logic + queries + imports) | ✅ Addressed |

**Beyond Jira scope (reasonable supporting changes):**
- Reorder `DetailedOrderInfo` fields (Buffer moved up; Services / Workflow / Batch Name moved down) — matches the Figma third-segment order.
- `JobCard` drops `mt="3rem"` and passes `withSpacing={false}` to `WorkflowBox` (pre-existing prop) — supports the new vertical rhythm.
- `OrderSidebarHeader` bottom margin `4rem → 2rem` — supports tighter spacing.

All in-scope for a "spring clean / layout reorganisation" story.

---

## Architecture Analysis

The approach is a clean relocation, not a rewrite:

- The Created-by / Created-for query + render logic (`useUserPersonalInfoQuery`, `useOrganizationMemberByIdQuery`, the `isProofedUser` / `isClientApi` branching, `AssigneeView` / `IconLogoApi`) is moved **verbatim** from `DetailedOrderInfo` (third segment) into `GeneralOrderInfo` (top segment). No new behaviour was introduced, which keeps regression risk low.
- The title becomes interactive by **extending the shared `Heading` molecule** with three optional props (`isInteractive`, `isExpanded`, `onTitleClick`) rather than forking it — consistent with the repo's reuse-first convention. The expand/collapse state is owned by `OrderTitleInfo` via a `hooks`-free `useState`/`useCallback` (this is a small partial, so inline state is acceptable).
- The top "people row" reuses `DetailsList` (horizontal flex-wrap of column cells), so Customer / Created by / Created for naturally lay out side-by-side as in the Figma. The third segment correctly stays on `DescriptionList` (vertical rows).
- `Brief` already supported `workItemType` / `workItemSubject`; the PR just wires them through, so requirement 8 is a one-line change with no component edit.

I verified the design against the Figma node (77531-35944): title (Order ID subtitle + truncated external reference), expanded title state, horizontal people row, Brief "Type/Industry" first, and the reordered third segment all match.

Regression check on the shared `Heading` molecule: the other five consumers (`OrderInfo`, `HTMLContentPill`, `JobTable/consts`, `PayoutsTable`, `DocumentsTable`) don't pass the new props, so `isInteractive`/`isExpanded` are `undefined` → the title falls back to the original nowrap/ellipsis styling and `onClick` is `undefined`. No behavioural change for them. `GeneralOrderInfo`'s other consumer (`JobManagement`) doesn't pass `createdById`/`createdForId`/`creator`, so the new rows stay hidden and both queries stay `enabled: false`. Mergeable state is reported `clean`.

---

## Issues Found

### 1. Title expand state leaks across order navigation

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/OrderTitleInfo.tsx]**
**Function/Class:** OrderTitleInfo
**Severity:** low
**Problem:** `isExpanded` is local component state. When the side panel switches to a different order without unmounting `OrderTitleInfo` (e.g. via `onNavigateToGroupOrder` group navigation), the component instance is reused and `isExpanded` persists. A user who expands order A's long external reference and then navigates to order B will see B's title rendered expanded even though they never clicked it.
**Impact:** Minor visual inconsistency — the next order's title shows un-truncated when it should default to the collapsed/ellipsis state. Not breaking; data is correct.
**Fix:** Reset the expanded state when the title (order) changes, or key the component by order id. Either:

```tsx
// Option A — reset on title change
useEffect(() => setIsExpanded(false), [title]);
```

```tsx
// Option B — remount per order at the call site (OrderManagment/index.tsx)
<OrderTitleInfo key={order.id} ... />
```

### 2. Empty heading title for orders without an external reference

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/index.tsx]**
**Function/Class:** OrderManagement (OrderTitleInfo usage)
**Severity:** low
**Problem:** `title={order.externalReference ? removeUniqueKeyFromExternalReference(order.externalReference) : ""}`. When an order has no external reference, the large title line renders empty, leaving only the small Order-ID subtitle. Previously the title always showed the content type (`workItemSubject`), so the heading was never blank.
**Impact:** Orders without an external reference get a visually empty heading area. May be acceptable if all orders in practice carry an external reference, but worth confirming.
**Fix:** Provide a fallback for the empty case, e.g. fall back to the work-item subject or the order id string, and keep `isTitleExpandable` only when there is a real external reference (already correct):

```tsx
title={
  order.externalReference
    ? removeUniqueKeyFromExternalReference(order.externalReference)
    : order.workItemSubject ?? order.id.toString()
}
```

### 3. PR description does not match the final implementation

**[File: PR description]**
**Function/Class:** —
**Severity:** low
**Problem:** The summary states `useCreatedByForRowItems now returns a named { createdBy, createdFor } object (was DetailsListItem[]), with its types extracted to a new types.ts` and "horizontal people row layout **instead of as `DetailsList` rows**". None of this is in the diff — there is no `useCreatedByForRowItems` hook in the codebase, and `GeneralOrderInfo` does use `DetailsList`. The description appears to describe an earlier abandoned approach.
**Impact:** Misleads reviewers about what changed. The actual implementation is fine; only the narrative is stale.
**Fix:** Update the PR summary to describe the merged approach (Created by/for moved into `GeneralOrderInfo` as `DetailsList` items; no new hook).

### 4. Jira testing-notes items 9–11 still marked ❌

**[File: Jira PP-1867 testing notes]**
**Function/Class:** —
**Severity:** low
**Problem:** The ticket's testing checklist still shows items 9 (hide Created by/for when empty), 10 (removed from third segment) and 11 (type/industry removed from title) as ❌. The code does address all three (`hidden` flags, removal from `DetailedOrderInfo`, new title content).
**Impact:** QA status is out of sync with the implementation.
**Fix:** Ask QA to re-verify and tick 9–11; no code change required.

---

## Tests

- ✅ `OrderTitleInfo.test.tsx` (new): covers default non-interactive title, `isTitleExpandable` flips `isInteractive`, click toggles `isExpanded` both ways, and no toggle when non-interactive. Good coverage of requirements 4–7's state logic.
- ✅ `GeneralOrderInfo.test.tsx` (extended): covers Created by present (req 9), Created for present, both hidden when absent (req 10), and the Client-API icon fallback. DetailsList is mocked to emit `detail-<title>` test ids, so the assertions are self-consistent.
- ⚠️ No unit test for the Brief content-type/industry wiring (req 8) or job-card repositioning (req 1) — both are integration-level / one-line prop wiring, so this is acceptable but worth a manual-test note.
- ✅ The styling change to the shared `Heading` is CSS-only; the interactive behaviour is exercised through `OrderTitleInfo.test.tsx`.
- ℹ️ Tests not run locally for this review (the PR branch is not checked out; the working tree holds unrelated PP-1861 changes). PR description reports the GeneralOrderInfo + OrderTitleInfo suites and typecheck/eslint as passing; the test structure is sound.

**Manual test plan to confirm:** open an order with a long external reference → title truncates with ellipsis; hover → turns green; click → expands; click again → collapses; navigate to another order in the group → title should default back to collapsed (see Issue 1); verify Brief shows Type then Industry first; verify Created by/for appear in the top row only when present; verify an order with no external reference renders acceptably (see Issue 2).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Matches all 11 requirements and the Figma design |
| Regression risk | ✅ Low — shared `Heading` props are optional/back-compatible; `GeneralOrderInfo`'s other consumer unaffected; queries gated |
| Tests | ⚠️ Good unit coverage for the new logic; integration-level wiring (Brief/job placement) untested |
| Code quality | ✅ Clean relocation, reuses `DetailsList` / `AssigneeView`, extends rather than forks `Heading` |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve with suggestions**

1. (Optional, low) Reset `isExpanded` when the order changes so the title doesn't stay expanded across group navigation (Issue 1).
2. (Optional, low) Add a fallback title for orders with no external reference to avoid an empty heading (Issue 2).
3. Update the PR description to match the merged implementation (Issue 3).
4. Have QA re-verify and tick testing-notes items 9–11 (Issue 4).

None of these block merge; they are polish items. The core reorganisation is correct, well-scoped, and design-accurate.
