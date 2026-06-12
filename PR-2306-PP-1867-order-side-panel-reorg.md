# PR Review: feature/PP-1867 — Reorganize order side panel

**PR:** https://github.com/Proofed/B2BWebserver/pull/2306
**Jira:** https://proofed.atlassian.net/browse/PP-1867
**Status:** In Progress (Code Review)
**Reviewed at commit:** `2f703d061` (review-fix commit on top of the original PR)

> This is a **re-review** after the review-fix commit `2f703d061`. It supersedes the
> original review of the 7 points. Each point below carries its current status.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1–2. Job cards moved directly below customer info (not at bottom) | `OrderManagment/index.tsx` — `JobsContentWrapper` rendered after `GeneralOrderInfo` | ✅ Addressed |
| 3. Order ID + external reference replace type/industry in title | `OrderTitleInfo` — `title = externalReference`, `subtitle = order.id` | ✅ Addressed |
| 4. External reference single-line, ellipsis-truncated | `StyledHeadingTitle` truncate styles | ✅ Addressed |
| 5. Hover turns full text green | `StyledHeadingTitle` `:hover { color: green1 }` when interactive | ✅ Addressed |
| 6–7. Click expands / re-click collapses | `isExpanded` toggle + `data-expanded` CSS | ✅ Addressed |
| 8. Content type + industry first two Brief items | `workItemType` / `workItemSubject` Brief items | ✅ Addressed |
| 9. Created by / Created for in top info segment | Relocated to `GeneralOrderInfo` as `DetailsList` items | ✅ Addressed |
| 10. Hidden when no value | `hidden: !createdById` / `hidden: !createdForId` | ✅ Addressed |
| 11. Removed from third segment | Removed from `DetailedOrderInfo` | ✅ Addressed |

**Beyond Jira scope:** detailed-segment field reordering (Buffer up; Services/Workflow/Batch down) and panel spacing tweaks — presented as Figma alignment. Benign.

**Jira Testing-Notes checklist:** items **9–11 are still marked ❌ on the ticket** despite being implemented at this PR head. This is a stale QA checklist, not a code defect — **QA must re-verify and tick 9–11 before merge.**

---

## Architecture Analysis

The interactive title is driven by three optional, back-compatible props on the shared
`Heading` molecule (`isInteractive`, `isExpanded`, `onTitleClick`), all defaulting to
`undefined` so the ~dozen other `Heading` consumers are unaffected. Created-by/for query +
render logic was relocated **verbatim** from `DetailedOrderInfo` into `GeneralOrderInfo`.
The approach is clean and low-risk; no new shared primitives were introduced unnecessarily.

---

## Status of the 7 Original Review Points

| # | Sev | Point | Status |
|---|-----|-------|--------|
| 1 | HIGH | JobManagement loses Order ID; add `showOrderId` prop | ⚪ **No action — premise incorrect** |
| 2 | MED | Interactive title lacks keyboard/ARIA | ✅ **Fixed** (`2f703d061`) |
| 3 | MED | Blank title when no external reference | ❌ **Open — needs design decision** |
| 4 | LOW | Double separators when no jobs | ✅ **Fixed** (`2f703d061`) |
| 5 | LOW | Loading state for Created by/for dropped | ⚪ **No action — false positive** |
| 6 | LOW | Dead prop `isProofedUser` | ✅ **Fixed** (`2f703d061`) |
| 7 | LOW | State inline vs sibling `hooks.ts` | ❌ **Open — convention cleanup** |
| — | — | Heading interactive wiring untested | ✅ **Fixed** — 6 a11y tests added |

**Net: 4 fixed, 2 resolved as no-action (false positives), 2 minor open (1 design call, 1 optional cleanup).**

---

## Issues Found (still open)

### 1. Blank title for orders without an external reference

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/index.tsx]**

**Function/Class:** OrderManagement (title prop to OrderTitleInfo)

**Severity:** medium

**Problem:** When `order.externalReference` is absent, `title` falls back to `""`, so the large heading renders empty. Common for manually-created orders. The Order ID still shows as the subtitle, so the panel isn't identifier-less, but the primary heading line is blank.

**Impact:** Visually empty title row on reference-less orders. Not a crash; a design-completeness gap.

**Fix:** Product/design decision required — confirm intended fallback. If `"Order {ID}"` is desired:

```tsx
title={
  order.externalReference
    ? removeUniqueKeyFromExternalReference(order.externalReference)
    : `Order ${order.id}`
}
```

Leaving as-is is acceptable **if** design confirms an empty title is intended. Flagged for confirmation, not auto-changed.

### 2. OrderTitleInfo keeps state inline instead of a sibling hooks file

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/OrderTitleInfo.tsx]**

**Function/Class:** OrderTitleInfo

**Severity:** low

**Problem:** `useState`/`useCallback` live directly in the component, and it is a flat `.tsx` file. CLAUDE.md requires non-trivial state in a sibling `use<Component>` hook and forbids flat `.tsx` component files (folder with `index.tsx` + `hooks.ts` + `types.ts`).

**Impact:** Convention drift only — no functional effect. The state here is genuinely trivial (one boolean toggle), so the cost/benefit of the restructure is low.

**Fix:** Promote `OrderTitleInfo.tsx` → `OrderTitleInfo/` folder with `index.tsx` (UI) + `hooks.ts` (`useOrderTitleInfo` holding `isExpanded`/`toggleExpanded`). Optional cleanup; can be deferred.

---

## Notes on the two "no action" points (deep analysis)

**#1 — JobManagement Order ID (originally HIGH): premise is incorrect.**
The job sidebar renders the order ID itself via its own heading —
`JobManagement/index.tsx:129 → subtitle={String(order.id)}`. The `orderId` passed into
`GeneralOrderInfo` there is used only for the group-dropdown `isActive` check; it was never
rendered as a visible row for the job panel. So no identifier is lost and a `showOrderId`
opt-out prop would be dead weight. Recommend a 10-second visual confirm; no code change.

**#5 — Created-by/for loading state (LOW): false positive.**
`AssigneeView` renders `<SkeletonBox/>` whenever `!userDetails` — **identical on `develop`
and this branch**. `AssigneeViewProps` never had an `isLoading` prop; `develop` spread an
`isLoading` that `AssigneeView` silently ignored (JSX spread attributes skip TS excess-property
checks). So skeleton feedback is fully preserved; "restoring" it would only re-add a dead prop.

---

## Validation Checks

Run against the PR branch at `2f703d061` (in-place; build/typecheck reused from earlier same-session runs on this exact commit).

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⚠️ | 1233 pass / **1 pre-existing failure** in `packages/shared/utils/formatWordQuantity.test.ts` — Indian-locale digit grouping (`"10,00,000"` vs `"1,000,000"`), environment-specific, **not in this PR's diff**. OrderManagment (117) + new Heading (6) suites pass. |
| `npx turbo run typecheck` | ✅ | creative-portal `tsc --noEmit` clean. |
| `npx turbo run lint` | ⚠️ | **The 4 changed files lint clean (exit 0).** Full-suite failures are pre-existing: 63 prettier errors in `packages/wysiwyg` (untouched package) + ~40 pre-existing files in creative-portal (JobTable, ChargeTable, api/*, etc.). None are this PR's files. Out of scope per PR-scope discipline. |
| `npx turbo run build` | ✅ | 4/4 successful, clean. |

---

## Tests

- ✅ New `Heading/index.test.tsx` — 6 tests covering the interactive-title a11y wiring (role/tabIndex/aria-expanded present only when interactive; click + Enter + Space invoke callback; other keys ignored).
- ✅ Existing `OrderTitleInfo.test.tsx` (4) and `GeneralOrderInfo.test.tsx` (16) pass.
- ✅ PR meets the "tests for new code" requirement — the new behavior (keyboard/ARIA) is covered.
- ⚠️ No automated coverage for the blank-title fallback (#3) — pending the design decision.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Requirements met; 5 of 7 review points fixed/resolved |
| Regression risk | ✅ Low — interactive props are opt-in; other `Heading` consumers unaffected |
| Tests | ✅ New a11y tests added; targeted suites green |
| Code quality | ✅ Clean; one optional convention cleanup (#7) outstanding |
| Validation suite | ⚠️ build + typecheck pass; test/lint failures are **all pre-existing & out-of-scope** (verified not in PR diff) |
| Mergeable state | ✅ Clean (GitHub `mergeable_state: clean`) |

---

## Recommendation

**Approve with suggestions.** The 3 must-fix review points are resolved: title a11y (#2)
and double-separator (#4) are fixed; the JobManagement-Order-ID (#1) and loading-state (#5)
concerns are false positives confirmed by code analysis. Remaining open items are minor:

1. **#3 (design call):** Confirm intended title fallback for reference-less orders — empty vs `"Order {ID}"`. Only code change still potentially needed.
2. **#7 (optional):** Extract `OrderTitleInfo` state to a sibling `hooks.ts` + folderize, to satisfy the component-structure convention. Deferrable.
3. **QA:** Re-verify and tick Jira Testing-Notes items 9–11 (implemented but still marked ❌ on the ticket).
4. **Pre-existing failures** (`formatWordQuantity` locale test; wysiwyg + misc creative-portal lint) are **not introduced by this PR** — flag to the team separately; do not block this PR on them.
