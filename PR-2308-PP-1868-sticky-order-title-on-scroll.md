# PR Review: feature/PP-1868: Sticky order title on scroll

**PR:** https://github.com/Proofed/B2BWebserver/pull/2308
**Jira:** https://proofed.atlassian.net/browse/PP-1868
**Status:** Code Review
**Reviewed head:** `2b44bbe` (supersedes earlier review of `bf4d621` — the dropdown-clipping issue raised there is fixed by the new `StickyTitleClip` layer)

> **Note:** This PR is **stacked** — its base is `feature/PP-1867-order-side-panel-reorg`, not `develop`. PP-1867 must merge first; with squash-and-merge, this branch then needs a rebase/retarget onto `develop` or its diff will include all of PP-1867.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1. Order ID + title appear in the top bar when scrolled out of view | `useStickyOrderTitle` attaches an IntersectionObserver to the panel's title row (`rootMargin: -146px` top); when it stops intersecting, `showStickyTitle` flips via the new `StickyTitleContext`, revealing `StickyTitle` in the sticky `OrderSidebarHeader` | ✅ Addressed |
| 2. Top-bar content matches the panel header | Sticky bar derives the title with the same `removeUniqueKeyFromExternalReference(order.externalReference)` and the icon through the same `getDocumentIcon` map (`DocumentIcon format` prop) as `OrderManagement`/`OrderTitleInfo` | ✅ Addressed |
| 3. Removed from the top bar when scrolled back up | Observer re-intersects → `setShowStickyTitle(false)`; status badge fades back in. Hook also resets the flag on unmount (history view switch) | ✅ Addressed |
| 4. Asana-style scroll-reveal animation | Hand-off animation: outgoing element slides + fades over 0.2s, incoming fades in after a 0.3s delay (0.2s out + 0.1s gap), no slide-in. Thoroughly commented in `styles.ts`. Visual fidelity needs the attached video / manual check — cannot be verified statically | ✅ Addressed (verify visually) |
| 5. No sticky title when panel is at the top | `initialIsIntersecting: true` keeps the flag false on mount; default context/state is `false` | ✅ Addressed |

**Beyond Jira scope:**

- `GeneralOrderInfo`: Customer row now tinted `green1` when a partner link exists — unrelated to PP-1868 (see Issue 1).
- `Heading` molecule: truncation styles refactored from props to a `data-expanded` attribute and made `display: block; width: 100%` unconditionally — broader than this ticket, affects all Heading consumers (see Issue 5).
- `OrderTitleInfo` / `OrderManagment/styles.ts` flex fixes (`min-width: 0`, `flex-shrink: 0`, `align-items: flex-start`) — supporting changes for title truncation; reasonable to include.

---

## Architecture Analysis

The implementation is clean and idiomatic for this codebase:

- A new `contexts/stickyTitleContext` (consts/context/provider/types — matches the existing context folder pattern) carries a single `showStickyTitle` flag. `StickyTitleProvider` is nested inside `OrderSidebarProvider`, scoping it correctly to the order side panel and avoiding prop-drilling through `OrderSidebar`.
- `useStickyOrderTitle` (new hook in `OrderManagment/`) wraps `useIntersectionObserver` from `@proofed/shared/libs/usehooks` (a pure re-export of `usehooks-ts`, so the test mock of `usehooks-ts` intercepts correctly) and toggles the context flag.
- `OrderSidebarHeader`'s `Container` becomes `position: sticky; top: 0; z-index: 2` inside the shared `StyledSidebar` scroll container (`overflow: auto`), which is the correct sticky context.
- The status badge and sticky title cross-fade as two absolutely-positioned layers inside a new `HeaderLeft` wrapper; the slide-down is masked by a dedicated `StickyTitleClip` layer so the inline status-change Dropdown menu is **not** clipped (this fixes the issue flagged in the previous revision's review).
- The geometry is deliberate and internally consistent: the Sidebar's top padding is stripped (`style={{ paddingTop: 0 }}`), re-applied as `Panel` `padding-top: 2.48rem`, and the warning bar's `padding-top` increase (0.625 → 2.375rem = +1.75rem) compensates exactly for the close-button row the sticky header now overlays. It works, but the coupling spans three files (see Issue 4).
- The animation CSS is unusually well documented, including the a11y rationale for the `visibility` transitions (keeps the hidden status dropdown out of the tab order — nice touch).

---

## Issues Found

### 1. Out-of-scope visual change in GeneralOrderInfo

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/GeneralOrderInfo/index.tsx]**

**Function/Class:** GeneralOrderInfo

**Severity:** low

**Problem:** The Customer row's `Flex` now gets `color: "green1"` whenever `customerPartnerHref` exists. This tints the `IconOrganisation` icon green. Nothing in PP-1868 (sticky title) calls for this; it has no relationship to the scroll behavior.

**Impact:** Scope creep — an unreviewed visual change rides along with a scroll-behavior ticket. If it's a deliberate design fix it belongs in its own ticket (or in PP-1867, the reorg PR this stacks on); if accidental it's a visual regression.

**Fix:** Either move this change to PP-1867/its own ticket, or confirm with design and call it out explicitly in the PR description.

### 2. IntersectionObserver root is the viewport — sticky title can flash during the sidebar slide animation

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/useStickyOrderTitle.ts]**

**Function/Class:** useStickyOrderTitle

**Severity:** medium

**Problem:** The observer uses the default root (viewport). The sidebar is a `position: fixed` panel that animates open/closed with `translateX(calc(100% + 2rem))` over 0.4s. While the panel is (partially) translated off-screen with its content mounted, the title row is outside the viewport, so `isIntersecting` goes `false` and `setShowStickyTitle(true)` fires — even though the user never scrolled.

**Impact:** A possible flash of the sticky title (and the status badge hidden) during the open/close transition or on re-open, violating acceptance criterion 5 ("must not appear when the title is already visible") in that transient window.

**Fix:** Pass the sidebar scroll container as the observer `root`. Since root and target then translate together, horizontal panel movement no longer breaks intersection, and only real vertical scrolling toggles the flag. As a bonus, the `rootMargin` no longer needs to encode the 72px page-header offset (only the ~74px bar height), reducing the Issue 3 coupling:

```typescript
const { ref: titleRef, isIntersecting } = useIntersectionObserver({
  threshold: 0,
  initialIsIntersecting: true,
  root: scrollContainerRef.current, // StyledSidebar element
  rootMargin: `-${STICKY_BAR_HEIGHT_PX}px 0px 0px 0px`
});
```

At minimum, manually verify there is no flash when opening/closing the panel while an order with scrollable content is loaded.

### 3. Hardcoded 146px reveal offset couples to header geometry and root font size

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/consts.ts]**

**Function/Class:** STICKY_TITLE_REVEAL_OFFSET_PX

**Severity:** low

**Problem:** `146 = 72px page header + 74px sticky bar at 16px root font size`. The bar height is authored in rem (`min-height: 4.625rem`), so browser font scaling or any future header-height change silently drifts the reveal trigger. The author documents this limitation honestly in the JSDoc, including the suggested runtime-measurement fix.

**Impact:** With non-default font scaling the sticky title appears slightly too early/late (cosmetic). If `OrderSidebar` is ever rendered without `isUnderHeader`, the 72px component of the offset is wrong.

**Fix:** Acceptable as-is given the documentation. Adopting the `root`-based observer from Issue 2 removes the 72px component; measuring the bar via `getBoundingClientRect` removes the rest. Worth doing if accessibility scaling becomes a requirement.

### 4. Sticky-bar geometry is coupled across three files, and the "re-applied" padding doesn't match the original

**[File: apps/creative-portal/components/organisms/sidebars/OrderSidebar/index.tsx]**

**Function/Class:** OrderSidebar / Styled.Panel / WarningBarWrapper

**Severity:** low

**Problem:** Making the header sticky required: (a) `style={{ paddingTop: 0 }}` on the shared Sidebar, (b) `Panel` `padding-top: 2.48rem` in `OrderManagment/styles.ts`, and (c) `WarningBarWrapper` `padding-top` 0.625 → 2.375rem (compensating for the ~1.75rem close-button row the sticky header now overlays). Each site is commented, but the numbers only work together: the shared Sidebar's original top padding is **2rem**, not the 2.48rem the comment claims is "re-applied", and the 1.75rem compensation assumes the close row stays `1.5rem + 0.25rem`.

**Impact:** Any change to the shared Sidebar's padding, the close row, or the bar's `min-height` silently breaks the alignment in a way no test catches. The 0.48rem discrepancy also means content sits slightly lower than before — presumably tuned to the Figma design, but the comment is misleading.

**Fix:** No structural change required for this PR (the comments are good mitigation). Correct the comment to say the value was tuned to the design rather than "re-applied", and consider deriving the warning-bar offset and panel padding from shared constants next time this area is touched.

### 5. Heading truncation styles are now unconditional — affects all Heading consumers

**[File: apps/creative-portal/components/molecules/Heading/styles.ts]**

**Function/Class:** StyledHeadingTitle

**Severity:** low

**Problem:** The base style adds `display: block; width: 100%; min-width: 0;` + `theme.textStyles.truncate` unconditionally (the expanded state opts out via `data-expanded="true"`). The truncate-vs-expand logic is behaviorally equivalent to the old prop-based CSS, but the new `display`/`width` declarations apply to every Heading consumer: `JobTable`, `DocumentsTable`, `PayoutsTable`, `OrderInfo`, `HTMLContentPill`, `OrderTitleInfo`.

**Impact:** In all current consumers the title's direct parent is the block-level `StyledHeadingWrapper`, so `width: 100%` is effectively a no-op and the regression risk is low — but those surfaces aren't covered by tests and weren't visually re-verified in this PR. Note also that `isExpanded` remains in `StyledHeadingProps` though the styled layer no longer reads it (type-level dead weight only; the prop is still consumed in `index.tsx` for `data-expanded`).

**Fix:** Spot-check the Heading in the Job/Documents/Payouts tables and OrderInfo sidebar on the PP-1867 base branch. Optionally move `isExpanded` from `StyledHeadingProps` to `HeadingProps` since the styled layer no longer uses it.

### 6. Open status dropdown is hidden mid-interaction when the user scrolls down

**[File: apps/creative-portal/components/molecules/OrderSidebarHeader/styles.ts]**

**Function/Class:** Status

**Severity:** low

**Problem:** The status-change `Dropdown` renders inline inside `Status`. If the user opens it and then scrolls down, `Status` transitions to `visibility: hidden; pointer-events: none`, hiding the open menu without an explicit close.

**Impact:** Minor UX edge case — the menu vanishes rather than closing; no broken state since the whole subtree becomes invisible and unfocusable together (the `visibility` transition handling here is otherwise careful and correct).

**Fix:** Acceptable as-is. If polish is wanted later, close the dropdown when `showStickyTitle` flips to `true`.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⏭️ Skipped | User opted out of validation run |
| `npx turbo run typecheck` | ⏭️ Skipped | User opted out of validation run |
| `npx turbo run lint` | ⏭️ Skipped | User opted out of validation run |
| `npx turbo run build` | ⏭️ Skipped | User opted out of validation run |

---

## Tests

- ✅ `useStickyOrderTitle.test.tsx` — covers hidden-while-visible, reveal-on-scroll-out, and reset-on-unmount (3 tests).
- ✅ `OrderSidebarHeader/index.test.tsx` — covers sticky-title hidden/visible states, status-badge hand-off, missing `externalReference`, and history (backward) mode (4 tests). The `usehooks-ts` mock is valid since `@proofed/shared/libs/usehooks` is a pure re-export.
- ⚠️ The header test mocks `./styles` entirely, so the CSS state machine (the actual animation/visibility behavior) is untested — acceptable for jsdom, but the tests assert data attributes, not user-visible behavior.
- ⚠️ No test updates for the `Heading` molecule refactor (behavioral equivalence asserted only by reading the CSS).
- ⏭️ Validation suite (test/typecheck/lint/build) **not run** — user opted out. PR meets the "tests for new code" requirement on paper, but suite passage is unverified.
- Manual test plan needed: scroll-reveal animation fidelity vs the Asana reference, open/close transition flash (Issue 2), warning-bar (overdue order) layout, and Heading rendering in tables (Issue 5).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Meets all 5 Jira acceptance criteria (animation fidelity needs visual check) |
| Regression risk | ⚠️ Medium — shared `Heading` molecule + Sidebar padding changes touch surfaces outside this ticket |
| Tests | ⚠️ Good unit coverage of new code; suite not executed |
| Code quality | ✅ Clean context/hook architecture, exemplary CSS comments |
| Validation suite | ⏭️ Skipped — user opted out |
| Mergeable state | ⚠️ GitHub reports clean, but validation unverified; stacked on PP-1867 |

---

## Recommendation

**Approve with suggestions**

1. **Run the validation suite before merging** (`npx turbo run test / typecheck / lint / build`) — it was skipped in this review, and CLAUDE.md treats any failure as a merge blocker.
2. **Merge order:** this PR must land after `feature/PP-1867-order-side-panel-reorg`; do not retarget to `develop` before PP-1867 merges.
3. Verify there is no sticky-title flash during the sidebar open/close slide (Issue 2); if there is, scope the IntersectionObserver `root` to the sidebar scroll container.
4. Confirm the `GeneralOrderInfo` green Customer-row tint is intentional and covered by a ticket/design (Issue 1), or split it out.
5. Spot-check Heading rendering in JobTable / DocumentsTable / PayoutsTable / OrderInfo on the base branch (Issue 5).
6. Fix the misleading "2.48rem re-applied" comment (original shared Sidebar top padding is 2rem) when next touching the file (Issue 4).
