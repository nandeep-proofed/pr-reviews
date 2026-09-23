# PR Review: fix/PP-2066: Show a skeleton while the order panel loads

**PR:** https://github.com/Proofed/B2BWebserver/pull/2466
**Jira:** https://proofed.atlassian.net/browse/PP-2066
**Status:** Code Review

Reviewed at head `c044fb09c`. Re-verified 23 September 2026 at the same head (see Verification).

---

## Verification (23 September 2026)

The PR has not moved since this review: head is still `c044fb09c` (PR last updated 15 September). Every claim below was re-checked against the code on that head.

| Claim | Outcome |
| --- | --- |
| Three render states: collapsed → `null`, loading → skeleton, errored → `null` | Confirmed at `OrderSummary/index.tsx:134` and `:138-140`, the exact lines cited |
| Issue 1: labels and column split hand-copied in the skeleton | Confirmed. `OrderSummarySkeleton/index.tsx:23-44` against `OrderAndBriefDetails/utils.tsx:86, 96-99, 103, 105, 115` — every cited line is accurate |
| No test ties the two sets of labels together | Confirmed. The label test asserts four literals typed in the test file itself, and the loaded-state test mocks the real panel, so a rename in `getOrderDetailLists` would not fail anything |
| Conditional rows left out of the skeleton, and marked `hidden` in the real panel | Confirmed for Created By, PO ID, Size and Files |
| Skeleton is `aria-hidden`, nothing announces loading | Confirmed at line 60; no `aria-busy`, `aria-live` or visually-hidden text anywhere in these partials |
| Hover prefetch still has no `staleTime` | Confirmed at `useOrderPrefetch.ts:16-19` |
| React Query 4.36.1 (the version the error-state reasoning relies on) | Confirmed as the installed version |
| Reuses `SkeletonBox`, already imported by `OrderAndBriefDetails/utils.tsx` | Confirmed, same import in both |
| `Content` is a flex column with `gap: 1rem`, so the service bars do not touch | Confirmed at `OrderSummarySection/styles.ts:17-20` |
| `useDeepLinkMonth` clears the latch when the deep link goes away | Confirmed in the diff, four lines of code |
| 8 tests in `OrderSummary/index.test.tsx`, 2 new in `useDeepLinkMonth.test.ts` | Both counts confirmed |
| Jira requirements table | Confirmed against the ticket, including that parallelising and de-duplicating are explicit "not prescriptive" suggestions, that the repeated calls come from a shared cache key with no `staleTime`, and that the 8 September update adds the cross-month search fix |

Two things this review had left open are now settled:

- **Validation was run.** `yarn app:customer-portal test --run OrderTable orders` gives 22 files, 284 passed, exactly the numbers the PR description claims. The original review had recorded these as unverified.
- **Base drift.** The branch is 3 ahead and 23 behind `develop`, with no merge conflicts, and GitHub still reports the PR mergeable and clean. A fresh CI run on the current base is still worth having before merge.

Nothing in the original review needed correcting.

---

## What this means for users (non-technical summary)

The review found nothing a user would notice as broken. Expanding an order now shows a placeholder panel straight away instead of a blank row. Searching the same order twice in one session now opens it instead of showing "This order was not found". The one open behaviour question: if the order details fail to load, the row still sits open over an empty panel (the same as before this PR), with only the error toast to explain it.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
| --- | --- | --- |
| Show a loading skeleton/spinner while the first fetch is in flight, not `null` | `OrderSummary` renders `OrderSummarySkeleton` while expanded with no `orderDetail`; nothing on error; `null` when collapsed | ✅ Addressed |
| Parallelise the sequential fetch chain (suggested, "not prescriptive") | Not attempted | ⚠️ Not in this PR |
| De-duplicate repeated `workItemContentVersion` / `supportDocuments` / `orders/{id}/details` calls (suggested) | Partly: collapsed rows no longer mount the panel, so hovering a row stops firing `FilesSection` version requests. The hover-prefetch still has no `staleTime`, so expanding a hovered row refetches details in the background | ⚠️ Partial |
| (8 Sep update) Cross-month search for an already-opened order shows "not found" | `useDeepLinkMonth` clears `resolvedOrderId` when `orderId` goes away, so a repeat deep link re-syncs the month | ✅ Addressed |

Scope: the `useDeepLinkMonth` fix goes beyond the original ticket, but the 8 September ticket update explicitly adds it to PP-2066. That is not scope creep.

---

## Architecture Analysis

- **`OrderSummary`:** the old `if (!orderDetail) return null` becomes three states: collapsed → `null`, expanded and loading → skeleton, expanded and errored → `null`. All hooks still run above the early returns, so hook order is stable.
- **Error state:** I checked it against React Query 4.36.1. A refetch of a query that never had data resets `status` to `loading`. So after a failed hover-prefetch, expanding the row shows the skeleton again rather than staying blank.
- **`OrderSummarySkeleton`:** reuses the real `OrderSummarySection`, `DescriptionList` and `Separator`, plus the `SkeletonBox` that `OrderAndBriefDetails/utils.tsx` already imports. The column split matches `getOrderDetailLists`: left column Submitted / Project / Platform / Industry / Content Type, right column Style Guide / Language / Total Price. It lists only the rows `getOrderDetailLists` always renders; conditional rows carry `hidden`. `Content` is a flex column with `gap: 1rem`, so the two service bars do not touch.
- **`useDeepLinkMonth`:** the reset mirrors the existing `openedOrderIdRef` and `reportedMissingOrderIdRef` resets. I traced the paths: A → undefined → A re-syncs, A → B → A re-syncs, and `isResolved` goes back to `false` until the cached order query answers. That keeps `useAllOrdersQuery` disabled (via `shouldFetchOrders`) and stops `isTargetOrderMissing` from firing against the month still on screen.

Lenses applied (small diff, 5 files): correctness, regressions/contract, reuse/duplication, React/error handling, conventions/a11y. Security and performance lenses: nothing applicable. The change is presentational plus one state reset, with no new inputs, requests or data exposure.

---

## Issues Found

### 1. Skeleton field labels are a second copy of the real panel's labels

**[File: apps/customer-portal/components/molecules/tables/OrderTable/partials/OrderSummarySkeleton/index.tsx]**

> **In plain terms:** Nothing is wrong today. But if someone later renames a field in the order panel (say "Industry" becomes "Subject"), the loading placeholder will keep the old name and the label will visibly change when the data arrives.

**Function/Class:** `FIELD_SKELETONS`, `ORDER_FIELD_COLUMNS`

**Severity:** low

**Confidence:** high

**How to spot it:** code health only; there is no user-reproducible path today. Compare `OrderSummarySkeleton/index.tsx:23-44` with `OrderAndBriefDetails/utils.tsx:84-135`.

**Problem:** The eight label strings and their left/right column grouping are typed out by hand in the skeleton. They already exist in `getOrderDetailLists`. Nothing ties the two together, and no test compares them. The skeleton test asserts the hardcoded strings against themselves.

**Evidence:** `OrderSummarySkeleton/index.tsx:24` `"Submitted:": <SkeletonBox withLoading sx={{ width: "11rem" }} />,` … `:35-44` `const ORDER_FIELD_COLUMNS = [["Submitted:", "Project:", "Platform:", "Industry:", "Content Type:"], ["Style Guide:", "Language:", "Total Price:"]];`. The same literals appear in `OrderAndBriefDetails/utils.tsx:86,96-99,103,105,115`.

**Impact:** Label drift between the loading and loaded states after any future rename. A reviewer has to remember to update two files.

**Fix:** Move the labels into an `OrderAndBriefDetails/consts.ts` (e.g. `ORDER_FIELD_LABELS`), then use them from both `getOrderDetailLists` and the skeleton:

```typescript
// OrderAndBriefDetails/consts.ts
export const ORDER_FIELD_LABELS = {
  submitted: "Submitted:",
  project: "Project:",
  platform: "Platform:",
  industry: "Industry:",
  contentType: "Content Type:",
  styleGuide: "Style Guide:",
  language: "Language:",
  totalPrice: "Total Price:"
} as const;
```

**Cheaper alternative (added 23 September):** sharing the constants removes the duplicated strings but not the duplicated structure — the skeleton deliberately lists only the always-present rows, while `getOrderDetailLists` also builds the conditional ones, so the two files stay coupled by hand either way. A test that renders both states and asserts the skeleton's labels against the loaded panel's labels catches the drift this issue describes, without either file importing the other. Either route closes the gap; neither is required to merge.

---

## Open Questions

- **Failed details request:** when `orders/{id}/details` fails after retries, the row stays open with the chevron flipped over an empty panel. The only feedback is the global toast. This matches pre-PR behaviour and the PR states it deliberately. Is that acceptable to product, or should the panel show an inline "Couldn't load this order" line? See `OrderSummary/index.tsx:138-140`.
- **Screen readers:** the skeleton is `aria-hidden` and nothing announces loading (no `aria-busy` on the detail cell, no visually-hidden "Loading order details"). Screen-reader users get silence during the wait, as before this PR. Is an announcement wanted? See `OrderSummarySkeleton/index.tsx:60`.
- **Follow-up tickets:** are there tickets for the two Jira suggestions this PR doesn't take on? Those are parallelising the fetch chain, and a `staleTime` on the hover prefetch so an expand doesn't refetch details that were fetched moments earlier. Also the `FilesSection` re-expand loading bar the PR lists as a known trade-off.

---

## Validation Checks

| Check | Result | Notes |
| --- | --- | --- |
| `yarn app:customer-portal test --run OrderTable orders` | ✅ Passed | Run 23 Sep 2026: 22 files, 284 passed, matching the PR description |
| `npx turbo run typecheck` | Not run | The PR reports a clean scoped run on `@proofed/customer-portal` |
| `npx turbo run lint` | Not run | The PR reports `lint --max-warnings 0` clean |
| `npx turbo run build` | Not run | Skipped: user opted out |

---

## Tests

- ✅ `OrderSummary/index.test.tsx` (new, 8 tests). It covers all three states, the skeleton → panel transition, the error state, collapsed-with-cached-data, always-present labels, left-out conditional rows, and the section headings.
- ✅ `useDeepLinkMonth.test.ts`: two new tests for the repeat deep link (re-sync, and `isResolved` false while pending). According to the PR, both fail on `develop`.
- ⚠️ The label assertions only check the skeleton's own hardcoded strings, so they would not catch the drift in Issue 1.
- ⚠️ No test for the collapse → re-expand cycle on an already-loaded row (panel back instantly, no skeleton flash). This is low risk: the component stays mounted and the query stays cached.

### Suggested manual QA script

1. Throttle to Slow 4G, search for an order from a different month, and confirm the placeholder panel (real headings and labels, grey bars) appears immediately and is replaced by the real panel.
2. Expand a row by chevron without hovering it first (keyboard or fast click) and confirm the same placeholder shows.
3. Search order X, change month, search order X again: it should open with no "This order was not found" toast (ticket 8 Sep update).
4. Block `orders/{id}/details` in DevTools, expand a row, and confirm the placeholder goes away after retries and the error toast shows (Open Question 1).
5. Expand a Complete GDOC order and a PDF order, and confirm the panel only grows (never shrinks) when data lands.

---

## Summary

| Aspect | Status |
| --- | --- |
| Correctness | ✅ |
| Regression risk | ✅ Low |
| Tests | ✅ (284 passed, verified 23 Sep) |
| Accessibility | ⚠️ (no loading announcement; pre-existing, see Open Questions) |
| Error handling | ✅ (error state handled; UX question open) |
| Security | ✅ (no new inputs, requests or data exposure) |
| Code quality | ✅ (one low duplication finding) |
| Validation suite | Partial: scoped tests run and passed; typecheck, lint and build not run here |
| Mergeable state | ✅ Clean (GitHub `CLEAN`, 23 behind `develop`, no conflicts) |

---

## Recommendation

**Approve**

1. Optional: close the label duplication (Issue 1), either by sharing the labels or by adding a test that compares the two states.
2. Get a product answer on the failed-load panel and the screen-reader announcement (Open Questions).
3. Raise follow-up tickets for the fetch-chain parallelisation and the hover-prefetch `staleTime`, if they don't already exist.
4. Scoped tests pass on the PR head. Confirm CI is green on the current `develop` base before merging.
