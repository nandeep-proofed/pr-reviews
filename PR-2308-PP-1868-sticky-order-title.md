# PR Review: feature/PP-1868: Sticky order title on scroll

**PR:** https://github.com/Proofed/B2BWebserver/pull/2308
**Jira:** https://proofed.atlassian.net/browse/PP-1868
**Status:** Code Review

> **Note:** This PR is **stacked** — its base is `feature/PP-1867-order-side-panel-reorg`, not `develop`. The diff/blast-radius below is read against the PR head. It must merge after (or be rebased on) PP-1867. No CI checks ran on this PR (base is a feature branch); the only PR comment is a Codex bot quota message, not a review.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1. When the order ID/title scroll out of their original position, they appear in the top bar | `useStickyOrderTitle` attaches an `IntersectionObserver` (`usehooks-ts`) to the panel's `Heading` row; when it leaves view, `isTitleStuck` flips true and `Styled.StickyTitle` is revealed (`data-visible="true"`) | ✅ Addressed |
| 2. Top-bar content must match the panel header | Sticky bar uses the same source values as `OrderTitleInfo`: `removeUniqueKeyFromExternalReference(order.externalReference)`, `order.id`, and `DocumentIcon` with `order.workItemFormat` | ✅ Addressed |
| 3. Scrolling back up removes the title from the top bar | Effect calls `setIsTitleStuck(!isIntersecting)`; on re-intersect it returns to `false`, hiding the bar (opacity 0, `aria-hidden`) | ✅ Addressed |
| 4. Appearance/disappearance uses an Asana-style scroll-reveal animation | `StickyTitle` transitions `opacity` + `translateY`, `Status` cross-fades `opacity` | ✅ Addressed (animation *fidelity* needs visual confirmation — not unit-testable) |
| 5. Title must NOT appear when the panel is at the top | `initialIsIntersecting: true` → starts not-stuck; `StickyTitle` renders `data-visible="false"`, `aria-hidden`, `pointer-events: none` | ✅ Addressed |
| API: no API changes | None made | ✅ Addressed |

**Scope note:** The PR also modifies a **shared** component (`packages/shared/components/molecules/Sidebar/styles.ts`) and `OrderSidebarWarningBar` padding. The Sidebar change reaches beyond this ticket's scope — see Issue #1.

---

## Architecture Analysis

The approach is clean and idiomatic:

- A small, single-responsibility hook (`useStickyOrderTitle`) owns the scroll detection and writes a boolean into the existing `OrderSidebarContext`. The header (`OrderSidebarHeader`) reads that boolean to cross-fade between the live status and the sticky title. This keeps the observer (in the scrollable body) decoupled from the consumer (in the sticky header) via context — a sensible choice given they're in different subtrees.
- The header `Container` becomes `position: sticky; top: 0; z-index: 2`, and the sticky title is an absolutely-positioned overlay (`inset: 0`) inside a new `HeaderLeft` wrapper, so the status ↔ title swap needs no layout reflow.
- The scroll container is `StyledSidebar` (`position: fixed; overflow: auto`), which fills the viewport vertically, so using the default (viewport) IntersectionObserver root works correctly here.

Both new files ship with focused unit tests. Overall the implementation is correct and the requirements are met. The findings below are about blast radius, a phantom dependency, and a few hardening/maintainability points — not correctness blockers.

---

## Issues Found

### 1. Shared `Sidebar` default top-padding change affects every other Default-size sidebar

**[File: packages/shared/components/molecules/Sidebar/styles.ts]**
**Function/Class:** StyledSidebar (`@media ${media.tablet}` padding, `SidebarSize.Default` branch)
**Severity:** medium
**Problem:** The change drops the Default-size top padding from `2rem` to `0rem` to let the order panel's sticky header sit flush. But `SidebarSize.Default` is the fallback for *any* `<Sidebar>` rendered without a `size` prop. The order panel is one such consumer, but so are several others that have no sticky header and no compensating padding: `InstantPaymentSidebar`, `DocDetailsSidebar`, `ApplyNowSidebar`, `ApplyNowBusinessSidebar` (all `<Sidebar {...{ isOpen, onClose, ref }}>` with no `size`/`style`). `NotificationsSidebar` forwards `size` from its caller and may also be affected. (`JobSidebar` is safe — it already overrides with inline `style={{ paddingTop: 24 }}`.)
**Impact:** Those panels lose 2rem of top breathing room at the tablet+ breakpoint; their close button and first heading shift flush against the top. At least `DocDetailsSidebar` and `ApplyNowSidebar` are real, user-facing panels, so this is a visual regression unrelated to PP-1868.
**Fix:** Scope the padding removal to the order sidebar instead of changing the shared default. The established pattern in this repo is the inline `style` prop (see `JobSidebar`):

```tsx
// apps/creative-portal/components/organisms/sidebars/OrderSidebar/index.tsx
<Sidebar
  {...{ onClose, isOpen, size, ref, isUnderHeader }}
  hasDropShadow={false}
  withoutOverlay
  style={{ paddingTop: 0 }}
>
```

Then revert `styles.ts` back to `"2rem 2rem 4rem 4rem"`. (Alternatively, add a dedicated size/variant if more order-specific layout is coming.)

### 2. `usehooks-ts` is imported in creative-portal but not declared as its dependency

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/useStickyOrderTitle.ts]**
**Function/Class:** useStickyOrderTitle (import)
**Severity:** medium
**Problem:** The hook does `import { useIntersectionObserver } from "usehooks-ts"`, but `usehooks-ts` is **not** in `apps/creative-portal/package.json` — only `packages/shared/package.json` declares it (`^3.1.0`). The import resolves today only because Yarn workspace hoisting lifts it to the root `node_modules` (a phantom/undeclared dependency). Additionally, the repo already exposes a re-export barrel `packages/shared/libs/usehooks.ts` (`export * from "usehooks-ts"`), which is the intended consumption path.
**Impact:** Fragile — a future hoisting change, a shared-package version bump, or a stricter install (PnP / `nohoist`) can break the app build with no code change. Also bypasses the project's own usehooks barrel convention.
**Fix:** Import via the shared barrel:

```ts
import { useIntersectionObserver } from "@proofed/shared/libs/usehooks";
```

(or, if a direct import is preferred, add `"usehooks-ts": "^3.1.0"` to `apps/creative-portal/package.json`).

### 3. Reveal offset is a hard-coded magic number coupled to the header geometry

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/consts.ts]**
**Function/Class:** STICKY_TITLE_REVEAL_OFFSET_PX
**Severity:** low
**Problem:** `STICKY_TITLE_REVEAL_OFFSET_PX = 115` is used as the observer's top `rootMargin`. The IntersectionObserver root is the viewport (top 0), but the order sidebar is rendered `isUnderHeader` (`StyledSidebar` `top: 72px`) and the sticky header is ~74px tall (`min-height: 4.625rem`). So the reveal boundary (115px from the viewport top) is decoupled from where the title actually disappears under the sticky bar (~146px from the viewport top). The constant was clearly hand-tuned against the running app, but it silently assumes the header height and the 72px header offset never change.
**Impact:** If the global header height, the `isUnderHeader` offset, or the sticky bar height changes, the reveal point drifts — risking a small scroll window where the real title is already hidden under the bar but the sticky title hasn't appeared yet (or vice-versa). Maintainability/robustness, not a current bug.
**Fix:** Prefer deriving the offset from the actual sticky header height (e.g. measure the header node / a `ResizeObserver`, or share a single header-height constant). At minimum, add a comment documenting that `115` ≈ header offset + bar height and must be kept in sync.

### 4. Hidden status block stays keyboard-focusable when the title is stuck

**[File: apps/creative-portal/components/molecules/OrderSidebarHeader/styles.ts]**
**Function/Class:** Status (`&[data-hidden="true"]`)
**Severity:** low
**Problem:** When stuck, `Status` is hidden with `opacity: 0; pointer-events: none`, but it is not `display:none`/`visibility:hidden`/`aria-hidden`. The status **dropdown trigger** therefore remains in the tab order and reachable by screen readers while visually invisible. (The reverse element, `StickyTitle`, *does* correctly toggle `aria-hidden`.)
**Impact:** Minor a11y/focus inconsistency — keyboard users can tab to and open an invisible status dropdown while the sticky title is showing.
**Fix:** Mirror `StickyTitle`'s treatment on `Status`, e.g. add `visibility: hidden` to the `data-hidden="true"` rule, and/or set `aria-hidden`/`inert` on `Styled.Status` when `isTitleStuck` is true.

### 5. Toggling `isTitleStuck` re-renders all OrderSidebarContext consumers

**[File: apps/creative-portal/contexts/orderSidebar/provider.tsx]**
**Function/Class:** OrderSidebarProvider (value `useMemo`)
**Severity:** low
**Problem:** `isTitleStuck` is added to the memoized context `value` (and its deps). The order sidebar context has many consumers (`OrderManagement`, `OrderJobs`, `GeneralOrderInfo`, the header, etc.); each stuck-toggle recreates `value` and re-renders all of them.
**Impact:** Low in practice — the IntersectionObserver fires only on threshold crossings (not per scroll frame) and React bails on identical `setState`, so this is at most two re-render cascades per scroll-through. Still, a purely presentational flag is being routed through a broad shared context.
**Fix:** Acceptable as-is. If you want it leaner later, isolate the flag (a dedicated tiny context/store, or drive the reveal with CSS on a `position: sticky` sentinel) so a scroll state change doesn't invalidate the whole order context. `setIsTitleStuck` is a stable `useState` setter, so omitting it from the deps array is correct.

### 6. Hard-coded sticky title text width

**[File: apps/creative-portal/components/molecules/OrderSidebarHeader/index.tsx]**
**Function/Class:** OrderSidebarHeader (sticky title `<Text>` `sx.width: "17.875rem"`)
**Severity:** low
**Problem:** The title text width is pinned to `17.875rem`. It has `overflow: hidden` + `textOverflow: ellipsis`, so it degrades gracefully, but the magic width is fragile if the panel/header width changes.
**Impact:** Cosmetic only.
**Fix:** Consider `flex: 1; min-width: 0` within the flex row (with the existing ellipsis) so the title fills available space rather than a fixed width, or extract the value as a named constant alongside the other sticky-title styles.

---

## Tests

- ✅ New hook has unit tests — `useStickyOrderTitle.test.tsx` covers not-stuck (on-screen), stuck (scrolled out), and reset-on-unmount, mocking `usehooks-ts` and the context.
- ✅ New header behavior has unit tests — `OrderSidebarHeader/index.test.tsx` covers: hidden when not stuck (req 5), revealed with matching order ID + title when stuck (reqs 1 & 2), order ID shown but title omitted when `externalReference` is empty, and *not* rendered in history (backward) mode.
- ⚠️ No automated coverage of the scroll-reveal **animation** (req 4) — inherent jsdom limitation; relies on manual/visual verification.
- ⚠️ All manual-testing checkboxes in the PR description are unticked; no manual test notes recorded. The Jira testing notes (5 scenarios) should be walked through against the running app, especially the `isUnderHeader` reveal timing (Issue #3).
- ✅ Meets the project rule that new code ships with tests.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ (requirements met) |
| Regression risk | ⚠️ Medium (shared Sidebar default padding hits 4+ other panels) |
| Tests | ✅ (good unit coverage; animation needs manual verification) |
| Code quality | ⚠️ (phantom `usehooks-ts` dep; magic-number offset; minor a11y) |
| Mergeable state | ✅ Clean (but stacked on PP-1867, not `develop`) |

---

## Recommendation

**Approve with suggestions** — the feature is correct and well-tested; please address the two medium items before merge.

1. **Scope the padding change (Issue #1):** revert the shared `Sidebar` default and instead pass `style={{ paddingTop: 0 }}` from `OrderSidebar` (matching `JobSidebar`'s pattern), then sanity-check the other Default sidebars (`DocDetails`, `ApplyNow`) are unaffected.
2. **Fix the phantom dependency (Issue #2):** import `useIntersectionObserver` from `@proofed/shared/libs/usehooks`, or declare `usehooks-ts` in `apps/creative-portal/package.json`.
3. **Verify visually (Issue #3 + req 4):** run the order panel with `isUnderHeader`, confirm the reveal timing has no no-title gap, and tick the manual test scenarios from the Jira ticket.
4. *(Optional)* tidy the minor a11y (Issue #4) and width (Issue #6) items.
