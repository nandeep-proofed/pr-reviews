# PR Review: feature/PP-1867: Reorganize order side panel

**PR:** https://github.com/Proofed/B2BWebserver/pull/2306
**Jira:** https://proofed.atlassian.net/browse/PP-1867
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1–2. Job cards directly below customer info, not at panel bottom | `Styled.JobsContentWrapper` block moved from the end of the panel to immediately after `GeneralOrderInfo` in `OrderManagment/index.tsx` | ✅ Addressed |
| 3. Order ID + external reference replace content type/industry in title area | `OrderTitleInfo` now gets `title` = cleaned `externalReference`, `subtitle` = `order.id` | ✅ Addressed (see Issue 3 for the missing-reference edge case) |
| 4. External reference single-line, ellipsis-truncated | `StyledHeadingTitle` keeps `nowrap/hidden/ellipsis` when collapsed | ✅ Addressed |
| 5. Hover turns text green | `&:hover { color: theme.colors.green1 }` when `isInteractive` | ✅ Addressed |
| 6. Click expands to full text | `isExpanded` toggles `white-space: normal; word-break: break-word` | ✅ Addressed (mouse only — see Issue 2) |
| 7. Click again collapses | Same toggle; `key={order.id}` resets state across order navigation | ✅ Addressed |
| 8. Content type + industry first two Brief items | `workItemType` / `workItemSubject` now passed to `Brief`, which already renders them as "Type:" / "Industry:" first | ✅ Addressed |
| 9. Created by / Created for in top info segment | Rendered as `DetailsList` items in `GeneralOrderInfo`, query logic relocated verbatim | ✅ Addressed |
| 10. Hidden when no value | `hidden: !createdById` / `hidden: !createdForId` | ✅ Addressed |
| 11. Removed from third segment | Both rows + their queries removed from `DetailedOrderInfo` | ✅ Addressed |

**Beyond ticket scope:**

- "Order ID" and "External Reference" rows were **deleted entirely from `GeneralOrderInfo`** — but `JobManagement` also renders that component (see Issue 1).
- `DetailedOrderInfo` field reorder (Buffer up; Services/Workflow/Batch Name down) and `OrderSidebarHeader` bottom margin 4rem → 2rem — presented as Figma alignment; plausible, but not in the written requirements.
- Prettier-3 formatting churn across 9 unrelated files (aiReviewFeedback API/tests, ChargeTable, NewOrderForm, customer-portal `createOrderAndJobs`, shared `DotsLoader`, `richTextEditorContext`). Consistent with the repo's prettier 3.5.3 toolchain, so benign — but it inflates a 21-file diff for a UI reorg and pollutes `git blame`.

**Jira flag:** the ticket's Testing Notes currently show items 9–11 marked ❌ (created by/for hidden when empty, removed from third segment, type/industry out of title area). The PR head *does* implement all three — the ❌ marks likely predate the latest commits. QA should re-verify and the ticket checklist updated before merge.

---

## Architecture Analysis

The approach is sound and low-risk in shape: the layout move is a relocation of existing JSX blocks rather than a rewrite, the created-by/for query + render logic is moved verbatim from `DetailedOrderInfo` to `GeneralOrderInfo`, and the interactive-title behavior is added to the shared `Heading` molecule via three optional props that default to prior behavior (`isInteractive` undefined → identical ellipsis CSS), so the other `Heading` consumers (`JobManagement`, `DocumentCard`, etc.) are unaffected. Expansion state lives in `OrderTitleInfo` and is reset across orders via `key={order.id}` — a clean derivation-over-effect choice.

The one structural miss is that `GeneralOrderInfo` is shared between the order sidebar and the job sidebar, and the row removals were only reconciled against the order sidebar.

---

## Issues Found

### 1. JobManagement sidebar silently loses its Order ID row

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/GeneralOrderInfo/index.tsx]**

**Function/Class:** GeneralOrderInfo

**Severity:** high

**Problem:** The unconditional "Order ID" item (old key `"2"`) was deleted from `GeneralOrderInfo`'s `DetailsList`. In the order sidebar that's fine — the Order ID moved to the title subtitle. But `JobManagement/index.tsx:173` also renders `GeneralOrderInfo` (passing `orderId={job.orderId.toString()}`), and the job sidebar's title area was **not** changed — so the job panel now shows Customer / PO ID / Customer Reference with no Order ID anywhere.

**Impact:** Regression outside PP-1867's scope (the ticket covers the *order* side panel only). Creatives/admins working from the job sidebar lose the order identifier they previously had. The `orderId` prop is now only used internally for the group-dropdown active check, so TypeScript catches nothing.

**Fix:** Keep the row but let the order sidebar opt out:

```tsx
// types.ts
export interface GeneralOrderInfoProps {
  orderId: string;
  showOrderId?: boolean; // default true
  ...
}

// GeneralOrderInfo items array
{
  key: "2",
  title: "Order ID",
  description: `${orderId}`,
  hidden: !showOrderId
},
```

Then pass `showOrderId={false}` from `OrderManagment/index.tsx` (where the ID now lives in the title) and leave `JobManagement` untouched.

### 2. Expandable title is mouse-only — no keyboard or screen-reader access

**[File: apps/creative-portal/components/molecules/Heading/index.tsx]**

**Function/Class:** Heading

**Severity:** medium

**Problem:** The interactive title is an `h2`/`h3` (`StyledHeadingTitle as={titleTag}`) with a bare `onClick`. There's no `tabIndex`, no Enter/Space key handling, no `role="button"`, and no `aria-expanded`. (`eslint-plugin-jsx-a11y` can't flag it because the element type is hidden behind the styled component.)

**Impact:** Keyboard users cannot expand a truncated external reference at all; screen readers announce a plain heading with no hint that it's interactive or what state it's in. Jira requirements 5–7 describe click interaction, but shipping it click-only is an accessibility regression for a repo that lints with a11y plugins.

**Fix:** Render a real button inside the heading when interactive (cleanest), or at minimum add the interactive affordances:

```tsx
<StyledHeadingTitle
  as={titleTag}
  onClick={isInteractive ? onTitleClick : undefined}
  onKeyDown={
    isInteractive
      ? (event) => {
          if (event.key === "Enter" || event.key === " ") {
            event.preventDefault();
            onTitleClick?.();
          }
        }
      : undefined
  }
  role={isInteractive ? "button" : undefined}
  tabIndex={isInteractive ? 0 : undefined}
  aria-expanded={isInteractive ? isExpanded : undefined}
  {...{ isSmall, title, isInteractive, isExpanded }}
>
```

### 3. Blank title when an order has no external reference

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/index.tsx]**

**Function/Class:** OrderManagement

**Severity:** medium

**Problem:** `title` falls back to `""` when `order.externalReference` is falsy. Previously the title was always `order.workItemSubject`. Orders without an external reference (manually created orders commonly lack one) now render an empty `h2` next to the document icon, with only the Order ID subtitle above it.

**Impact:** The title area looks broken/empty for a real class of orders. Neither the ticket nor the Figma states list a "no external reference" fallback, so this is an unhandled state rather than a designed one.

**Fix:** Confirm the intended fallback with design; a reasonable default is showing the Order ID as the title when no reference exists (and dropping the duplicate subtitle), e.g.:

```tsx
const cleanedReference = order.externalReference
  ? removeUniqueKeyFromExternalReference(order.externalReference)
  : "";

<OrderTitleInfo
  key={order.id}
  title={cleanedReference || `Order ${order.id}`}
  ...
/>
```

### 4. Double separator when an order has no jobs

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/index.tsx]**

**Function/Class:** OrderManagement

**Severity:** low

**Problem:** The first `Styled.Main` ends with `<Separator />`, then comes the jobs block, then the second `Styled.Main` opens with another `<Separator />`. When `jobs` is empty the jobs block renders nothing, leaving two consecutive separators (plus an empty negative-margin wrapper) between `GeneralOrderInfo` and `DetailedOrderInfo`.

**Impact:** Doubled divider line / uneven spacing for jobless orders (e.g. just-created or canceled-early orders).

**Fix:** Gate the jobs wrapper and the second separator on `!!jobs?.length`:

```tsx
{!!jobs?.length && (
  <Styled.JobsContentWrapper>
    <Styled.JobsContent ...>
      <OrderJobs {...{ payPerWordUnit, chargePerWordUnit }} />
    </Styled.JobsContent>
  </Styled.JobsContentWrapper>
)}
<Styled.Main>
  {!!jobs?.length && <Separator />}
  ...
```

### 5. Created by / Created for loading state was dropped in the move

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/GeneralOrderInfo/index.tsx]**

**Function/Class:** GeneralOrderInfo

**Severity:** low

**Problem:** `DetailedOrderInfo` destructured `isFetching` from both queries and passed `isLoading={isLoadingUserDetails}` to the "Created for" `AssigneeView`. The relocated code only destructures `data`, and `AssigneeView` gets no `isLoading` prop.

**Impact:** While the personal-info / organization-member queries are in flight, the rows render an empty/blank assignee instead of the loading skeleton — a small UX regression versus the third-segment original, and contrary to the PR's "relocated verbatim" description.

**Fix:** Restore the fetching flags:

```tsx
const { data: userDetails, isFetching: isFetchingUserDetails } =
  useUserPersonalInfoQuery(createdById ?? 0, { ... });

const {
  data: createdForDetails,
  isFetching: isFetchingCreatedForDetails
} = useOrganizationMemberByIdQuery(createdForId ?? createdById ?? 0, { ... });

<AssigneeView
  organizationMemberId={createdForId}
  userDetails={createdForDetails}
  isLoading={isFetchingUserDetails || isFetchingCreatedForDetails}
/>
```

### 6. Dead prop `isProofedUser` left on DetailedOrderInfoProps

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/types.ts]**

**Function/Class:** DetailedOrderInfoProps

**Severity:** low

**Problem:** `createdById`, `createdForId`, and `creator` were removed from the interface, but `isProofedUser?: boolean` — which only existed to support the removed Created by/for rendering — was left behind. Nothing reads or passes it.

**Impact:** Dead API surface; misleads the next reader into thinking `DetailedOrderInfo` still has creator-aware behavior.

**Fix:** Delete `isProofedUser?: boolean;` from `DetailedOrderInfoProps`.

### 7. New local state added inline instead of a hooks file

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/OrderTitleInfo.tsx]**

**Function/Class:** OrderTitleInfo

**Severity:** low

**Problem:** `useState`/`useCallback` were added directly into the component body. CLAUDE.md's convention is that non-trivial component state lives in a sibling `hooks.ts` exporting `useOrderTitleInfo`, with the component file staying UI-only. (The partial is a legacy flat file, so full folderization is arguably out of scope — but the new state could still follow the hook convention.)

**Impact:** Convention drift in a codebase that explicitly documents the pattern.

**Fix:** Extract a `useOrderTitleInfo` hook (in a sibling file or, if folderizing, `OrderTitleInfo/hooks.ts`) returning `{ isExpanded, toggleExpanded }`.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⏭️ | Skipped — user opted out |
| `npx turbo run typecheck` | ⏭️ | Skipped — user opted out |
| `npx turbo run lint` | ⏭️ | Skipped — user opted out |
| `npx turbo run build` | ⏭️ | Skipped — user opted out |

---

## Tests

- ✅ New `OrderTitleInfo.test.tsx` (4 tests): default non-interactive render, interactive flag, expand/collapse toggle, no-toggle when non-interactive.
- ✅ `GeneralOrderInfo.test.tsx` extended (4 tests): Created by row, Created for row, both hidden when IDs absent, Client API icon fallback — covers Jira requirements 9–10 directly.
- ⚠️ `OrderTitleInfo.test.tsx` mocks `Heading` entirely, so the real `Heading` interactive wiring (`onClick` gating, prop forwarding, expanded/collapsed CSS switch) has no test. A small `Heading` test exercising `isInteractive`/`isExpanded` would close the gap.
- ❌ No test would have caught Issue 1 — nothing asserts the Order ID row for the `JobManagement` consumer (and no longer can, since the row is gone).
- ⏭️ Full validation suite skipped — user opted out. PR description reports scoped `test`/`typecheck`/`eslint` passes by the author; not independently verified.
- Manual test plan: PR includes before/after screenshots; Jira Testing Notes items 9–11 are currently marked ❌ and need re-verification by QA against the latest commits.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ Jira requirements met for the order panel, but a cross-consumer regression (Issue 1) and an unhandled empty-reference state (Issue 3) |
| Regression risk | ⚠️ Medium — shared `GeneralOrderInfo` change leaks into the job sidebar; `Heading`/`Brief` changes are safely back-compatible |
| Tests | ⚠️ Good coverage of new behavior at the partial level; real `Heading` wiring untested; suite not run |
| Code quality | ✅ Clean relocation-style changes, optional back-compat props, state reset via `key`; minor convention drift and dead prop |
| Validation suite | ⏭️ Skipped — user opted out; re-run required before merge |
| Mergeable state | ⚠️ GitHub reports clean, but validation was not run locally — cannot confirm per CLAUDE.md's zero-failure bar |

---

## Recommendation

**Request changes**

1. **Restore an Order ID row for the JobManagement sidebar** (Issue 1) — add a `showOrderId` opt-out prop (or equivalent) so the job panel doesn't lose the identifier.
2. **Decide and implement the no-external-reference title fallback** (Issue 3) — confirm with design; blank `h2` is not an acceptable state.
3. **Add keyboard/ARIA support to the interactive title** (Issue 2) — `role="button"`, `tabIndex`, Enter/Space handling, `aria-expanded`.
4. Restore the Created by/for loading skeleton (Issue 5), gate the jobs-block separators for jobless orders (Issue 4), and drop the dead `isProofedUser` prop (Issue 6).
5. Re-run QA on Jira Testing Notes items 9–11 (currently ❌ on the ticket despite being implemented at the PR head) and update the ticket.
6. **Run the full validation suite** (`npx turbo run test / typecheck / lint / build`) before merging — it was skipped for this review, and CLAUDE.md treats any failure as a hard blocker.
7. Optional: split the 9 files of Prettier-3 reformatting into a separate chore commit/PR to keep the feature diff reviewable.
