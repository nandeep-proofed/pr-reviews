# PR Review: fix/PP-2066: Show a skeleton while the order panel loads

**PR:** https://github.com/Proofed/B2BWebserver/pull/2466
**Jira:** https://proofed.atlassian.net/browse/PP-2066
**Status:** Code Review

Reviewed at head `c044fb09c`. Re-verified 23 September 2026 with the full validation suite; Issue 1 fixed the same day at `be7130653`.

---

## Verification (23 September 2026)

Re-checked at `c044fb09c`, the head this review was written against. Every claim held.

| Claim | Outcome |
| --- | --- |
| Three render states: collapsed → `null`, loading → skeleton, errored → `null` | Confirmed at `OrderSummary/index.tsx:134` and `:138-140`, the exact lines cited |
| Issue 1: labels and column split hand-copied in the skeleton | Confirmed. `OrderSummarySkeleton/index.tsx:23-44` against `OrderAndBriefDetails/utils.tsx:86, 96-99, 103, 105, 115` — every cited line was accurate. Now fixed, see Issues Found |
| No test ties the two sets of labels together | Confirmed at the time: the label test asserted four literals typed in the test file itself, and the loaded-state test mocks the real panel, so a rename in `getOrderDetailLists` would not have failed anything |
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

- **Validation ran clean.** The whole `@proofed/customer-portal` suite passes, along with typecheck, lint at `--max-warnings 0`, and a production build. Numbers in Validation Checks below. The original review had recorded these as skipped and unverified.
- **Base drift.** The branch is 23 behind `develop`, with no merge conflicts, and GitHub reports the PR mergeable and clean. Validation ran on the PR head, not on a merge with current `develop`, so a green CI run on the current base is still worth having.

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

### 1. Skeleton field labels were a second copy of the real panel's labels (DRY) — ✅ Fixed at `be7130653`

**[File: apps/customer-portal/components/molecules/tables/OrderTable/partials/OrderSummarySkeleton/index.tsx]**

> **In plain terms:** Nothing was wrong for users. But if someone later renamed a field in the order panel (say "Industry" became "Subject"), the loading placeholder would have kept the old name and the label would have visibly changed when the data arrived.

**Function/Class:** `FIELD_SKELETONS`, `ORDER_FIELD_COLUMNS`

**Severity:** low

**Confidence:** high

**Problem (as found):** the eight label strings and their left/right column grouping were typed out by hand in the skeleton, though they already existed in `getOrderDetailLists`. Nothing tied the two together, and no test compared them: the skeleton test asserted the hardcoded strings against themselves.

**Evidence (as found):** `OrderSummarySkeleton/index.tsx:24` and `:35-44` against the same literals in `OrderAndBriefDetails/utils.tsx:86, 96-99, 103, 105, 115`.

**Scope of the duplication:** a DRY finding on the **strings only**. The label text was one piece of knowledge written in two files, which is what drifts on a rename. The surrounding structure is not: the skeleton deliberately lists only the always-present rows, while `getOrderDetailLists` also builds the conditional ones and marks them `hidden`. Those are two separate decisions that happen to overlap.

**Resolution:** a new `OrderAndBriefDetails/consts.ts` exports `ORDER_FIELD_LABELS`, covering all twelve labels including the conditional rows. `getOrderDetailLists` and `OrderSummarySkeleton` both read from it, so a rename moves the loading and loaded states together. The structure was left alone, as above. Verified after the change: 284 tests pass across 22 files for `OrderTable orders`, typecheck and lint clean.

```typescript
// OrderAndBriefDetails/consts.ts
export const ORDER_FIELD_LABELS = {
  contentType: "Content Type:",
  createdBy: "Created By:",
  files: "Files:",
  industry: "Industry:",
  language: "Language:",
  platform: "Platform:",
  project: "Project:",
  purchaseOrderId: "PO ID:",
  size: "Size:",
  styleGuide: "Style Guide:",
  submitted: "Submitted:",
  totalPrice: "Total Price:"
} as const;
```

---

## Open Questions

- **Failed details request:** when `orders/{id}/details` fails after retries, the row stays open with the chevron flipped over an empty panel. The only feedback is the global toast. This matches pre-PR behaviour and the PR states it deliberately. Is that acceptable to product, or should the panel show an inline "Couldn't load this order" line? See `OrderSummary/index.tsx:138-140`.
- **Screen readers:** the skeleton is `aria-hidden` and nothing announces loading (no `aria-busy` on the detail cell, no visually-hidden "Loading order details"). Screen-reader users get silence during the wait, as before this PR. Is an announcement wanted? See `OrderSummarySkeleton/index.tsx:60`.
- **Follow-up tickets:** are there tickets for the two Jira suggestions this PR doesn't take on? Those are parallelising the fetch chain, and a `staleTime` on the hover prefetch so an expand doesn't refetch details that were fetched moments earlier. Also the `FilesSection` re-expand loading bar the PR lists as a known trade-off.

---

## Validation Checks

Run 23 September 2026, scoped to `@proofed/customer-portal`. The first three were re-run after the Issue 1 fix.

| Check | Result | Notes |
| --- | --- | --- |
| `turbo run test` | ✅ Passed | 47 files, 478 tests passed at `c044fb09c`; 22 files, 284 passed for `OrderTable orders` after the fix |
| `turbo run typecheck` | ✅ Passed | `tsc --noEmit` clean, before and after the fix |
| `turbo run lint` | ✅ Passed | `eslint --max-warnings 0` clean, before and after the fix |
| `turbo run build` | ✅ Passed | Production build succeeded in 616s at `c044fb09c`, no warnings |

---

## Tests

- ✅ `OrderSummary/index.test.tsx` (new, 8 tests). It covers all three states, the skeleton → panel transition, the error state, collapsed-with-cached-data, always-present labels, left-out conditional rows, and the section headings.
- ✅ `useDeepLinkMonth.test.ts`: 11 tests, two of them new for the repeat deep link (re-sync, and `isResolved` false while pending). According to the PR, both fail on `develop`.
- ✅ Issue 1's drift risk is now structural rather than test-enforced: both states read the same constants, so the labels cannot diverge.
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
| Tests | ✅ (478 passed across 47 files) |
| Accessibility | ⚠️ (no loading announcement; pre-existing, see Open Questions) |
| Error handling | ✅ (error state handled; UX question open) |
| Security | ✅ (no new inputs, requests or data exposure) |
| Code quality | ✅ (the one DRY finding is fixed) |
| Validation suite | ✅ Test, typecheck, lint and build all pass |
| Mergeable state | ✅ Clean (GitHub `CLEAN`, 23 behind `develop`, no conflicts) |

---

## Recommendation

**Approve**

1. ~~Close the label duplication (Issue 1).~~ Done at `be7130653`.
2. Get a product answer on the failed-load panel and the screen-reader announcement (Open Questions).
3. Raise follow-up tickets for the fetch-chain parallelisation and the hover-prefetch `staleTime`, if they don't already exist.
4. Validation passes on the branch. Confirm CI is green on the current `develop` base before merging, since the branch is 23 commits behind.
