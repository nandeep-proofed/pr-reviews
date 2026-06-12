# PR Review: feature/PP-1868: Sticky order title on scroll

**PR:** https://github.com/Proofed/B2BWebserver/pull/2308
**Jira:** https://proofed.atlassian.net/browse/PP-1868
**Status:** Code Review

**Note:** This is a **stacked PR** — its base is `feature/PP-1867-order-side-panel-reorg`, not `develop`. PP-1867 must merge first; with squash-and-merge, this branch will then need a rebase/retarget onto `develop` before merging, or its diff will include all of PP-1867.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1. Order ID + title appear in the top bar once scrolled out of view | `useStickyOrderTitle` attaches an `IntersectionObserver` (rootMargin `-146px`) to the title row; `StickyTitleContext` flips `showStickyTitle`, revealing `Styled.StickyTitle` in the now-`position: sticky` header | ✅ Addressed |
| 2. Top-bar content matches the panel header | Sticky bar derives icon from `getDocumentIcon(order.workItemFormat)`, ID from `order.id`, title via `removeUniqueKeyFromExternalReference(order.externalReference)` — identical sources to `OrderTitleInfo` in `OrderManagement` | ✅ Addressed |
| 3. Removed from top bar when scrolled back up | Observer re-intersects → `setShowStickyTitle(false)`; also reset on content unmount (history view) | ✅ Addressed |
| 4. Asana-style scroll-reveal animation | Staged hand-off (0.2s outgoing slide+fade, 0.1s gap, 0.2s incoming fade) implemented in `Status`/`StickyTitle` transitions with thorough comments | ✅ Addressed (visual check vs reference video recommended) |
| Testing note 5: not shown when title already visible at top | `initialIsIntersecting: true` prevents a flash on mount | ✅ Addressed |

**Beyond Jira scope:** `GeneralOrderInfo` customer-row icon now inherits `green1` when a partner link exists; `OrderManagment/styles.ts` moves the `& > div:not(:first-of-type)` margin into the mobile media query (slight desktop layout change); `OrderSidebarWarningBar` padding rework. These look like deliberate polish for the new layout but are not covered by PP-1868.

---

## Architecture Analysis

The approach is sound: a tiny dedicated context (`StickyTitleContext`) bridges the scroll state from the panel body (`OrderManagement`, which owns the observed title row) to the header (`OrderSidebarHeader`, which renders the sticky bar), avoiding prop-drilling through `OrderSidebar`. The header `Container` becomes `position: sticky` inside the sidebar's scroll container, and the status badge / sticky title cross-fade as two absolutely-positioned layers inside a new `HeaderLeft` wrapper. The hook + context are small, memoized, and unit-tested. Reveal offset and title width are documented constants rather than bare magic numbers.

The main structural problem is that `HeaderLeft` uses `overflow: hidden` to mask the slide animations, but the status-change Dropdown renders its menu **inline** (`position: absolute`, no portal) inside that same wrapper — see Issue 1.

---

## Issues Found

### 1. Status-change dropdown menu is clipped by the new `overflow: hidden` wrapper

**[File: apps/creative-portal/components/molecules/OrderSidebarHeader/styles.ts]**

**Function/Class:** `HeaderLeft` / `Status`

**Severity:** high

**Problem:** `HeaderLeft` has `overflow: hidden` (added to mask the slide-up/slide-down animations). The status-change `Dropdown` lives inside `Styled.Status` → `HeaderLeft`. The shared Dropdown does **not** portal its menu: `StyledDropdownContentWrapper` is `position: absolute; top: 100%` inside `StyledDropdownWrapper` (`position: relative`) — see `packages/shared/components/molecules/Dropdown/styles.ts`. Since the menu's containing-block chain stays inside `HeaderLeft`, the open menu is clipped to the header's ~74px box, leaving only a sliver (if anything) visible below the trigger. Before this PR, `Status` sat directly in `Container` (no `overflow: hidden`), so the menu rendered fully.

**Impact:** Admins can no longer use the status-change dropdown in the order side-panel header — the menu opens but is invisible/unclickable below the clip boundary. This is a core workflow regression, not just cosmetic.

**Fix:** Don't clip the layer that hosts the dropdown. One option: remove `overflow: hidden` from `HeaderLeft` and clip only the sticky-title layer with a dedicated non-interactive clip layer (the status badge's upward slide is largely masked by the scrollport edge anyway, and it fades to opacity 0 over the same 0.2s):

```typescript
// styles.ts — HeaderLeft loses overflow: hidden
export const StickyTitleClip = styled.div`
  position: absolute;
  inset: 0;
  overflow: hidden;
  pointer-events: none;
`;
```

```tsx
// index.tsx
<Styled.StickyTitleClip>
  <Styled.StickyTitle ...>...</Styled.StickyTitle>
</Styled.StickyTitleClip>
```

Alternatively, render the status dropdown via a fixed/portal strategy (the shared Dropdown's `hasFloatedPosition` exists but its offset math is bespoke — verify before relying on it). Whichever route, manually verify the status dropdown opens fully on the PR branch before merge.

### 2. Global `Heading` style changes affect six unrelated consumers

**[File: apps/creative-portal/components/molecules/Heading/styles.ts]**

**Function/Class:** `StyledHeadingTitle`

**Severity:** medium

**Problem:** `display: block; width: 100%; min-width: 0; contain: layout` are now applied unconditionally, and `user-select: none` is applied to all interactive headings. `Heading` is consumed by `JobTable/consts.tsx`, `PayoutsTable/tableColumns.tsx`, `DocumentsTable/tableColumns.tsx`, `OrderInfo/index.tsx`, `HTMLContentPill/index.tsx`, and `OrderTitleInfo.tsx` — only the last one is in this ticket's scope. `contain: layout` also makes the title a containing block for absolute/fixed descendants and is unexplained — nothing in the title needs containment.

**Impact:** Possible layout drift in the job/payouts/documents tables and OrderInfo sidebar (e.g. `width: 100%` inside table cells changing ellipsis/click-target behavior). Low probability each, but none are visually verified by this PR's screenshots.

**Fix:** Visually spot-check the other five Heading consumers on the PR branch, and drop `contain: layout` unless there is a concrete reason (if there is one — e.g. preventing observer-driven reflow loops — document it in a comment like the other tuned values).

### 3. `user-select: none` prevents copying the order title

**[File: apps/creative-portal/components/molecules/Heading/styles.ts]**

**Function/Class:** `StyledHeadingTitle` (interactive variant)

**Severity:** low

**Problem:** Interactive (expandable) titles now have `user-select: none`. The expandable title in the order panel is the customer's external reference — text admins plausibly copy into emails/searches. It was selectable before this PR.

**Impact:** Users can no longer select/copy the order title text anywhere `Heading` is interactive.

**Fix:** Remove `user-select: none`, or confirm with design that accidental selection on click-to-expand is a bigger nuisance than losing copyability.

### 4. px-based reveal offset vs rem-based header geometry

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/consts.ts]**

**Function/Class:** `STICKY_TITLE_REVEAL_OFFSET_PX`

**Severity:** low

**Problem:** The observer's `rootMargin` is fixed at `-146px` (= 72px header + 74px sticky bar), but the sticky bar height is authored in rem (`min-height: 4.625rem`). `rootMargin` only accepts px/percent, so with a non-default root font size (browser accessibility font scaling) the rem-based bar grows while the 146px line doesn't, making the reveal fire too early/late. The coupling itself is well documented in the comment — this is the remaining unhandled case.

**Impact:** Misaligned reveal point for users with scaled fonts; the title can be hidden under the bar without the sticky title appearing (or both visible briefly).

**Fix:** Compute the offset at runtime (e.g. measure the sticky bar element, or derive `4.625 * parseFloat(getComputedStyle(document.documentElement).fontSize) + 72`), or accept and note the limitation.

### 5. Magic inline layout values instead of styled components / shared constants

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/index.tsx]**

**Function/Class:** `OrderManagement` (root `Box`), plus `OrderSidebar/index.tsx`

**Severity:** low

**Problem:** `style={{ width: "530px", paddingTop: "2.48rem" }}` (the `2.48rem` compensates for the sidebar's removed top padding — an oddly precise value with no explanation) and `style={{ paddingTop: 0 }}` on `Sidebar`. Project convention puts layout CSS in `styles.ts`; the two values are also coupled to each other and to the sticky header geometry but live in different files with no cross-reference, unlike the nicely documented `STICKY_TITLE_REVEAL_OFFSET_PX`.

**Impact:** Future header-geometry changes will silently break the spacing; inline styles bypass the theme and the established styling pattern.

**Fix:** Move both into the respective `styles.ts` (e.g. extend `Styled.Main`/a wrapper in `OrderManagment/styles.ts` and a styled override in `OrderSidebar`), with a comment tying them to the sticky-header geometry.

### 6. Sticky-title text styles inline in `index.tsx`

**[File: apps/creative-portal/components/molecules/OrderSidebarHeader/index.tsx]**

**Function/Class:** `OrderSidebarHeader` (sticky title `Text`)

**Severity:** low

**Problem:** The sticky title `Text` carries a seven-property `sx` block (width, ellipsis, line-height, etc.) in `index.tsx`, while the project convention is `index.tsx` is UI-only with CSS in the sibling `styles.ts` (where `STICKY_TITLE_TEXT_WIDTH` already lives).

**Impact:** Convention drift; the width constant and the styles that use it are split across files.

**Fix:** Add a `StickyTitleText` styled component in `styles.ts` consuming `STICKY_TITLE_TEXT_WIDTH` directly, and render `<Styled.StickyTitleText>{stickyTitle}</Styled.StickyTitleText>`.

### 7. `Heading` keeps two sources of truth for expanded state

**[File: apps/creative-portal/components/molecules/Heading/index.tsx]**

**Function/Class:** `Heading` / `StyledHeadingTitle`

**Severity:** low

**Problem:** The expanded styling moved from prop-driven Emotion CSS to a `data-expanded` attribute selector, but `isExpanded`/`isInteractive` are still forwarded as styled-component props (and still listed in `stopForwarding`). Two mechanisms now express the same state; the props are only used for the `isInteractive` hover block.

**Impact:** Confusing for the next reader; easy for the attribute and prop paths to drift apart.

**Fix:** Pick one mechanism — either keep the data attribute and drop `isExpanded` from the styled props, or revert to prop-driven CSS. If the data attribute exists for a concrete reason (e.g. avoiding Emotion class regeneration during scroll-driven re-renders), say so in a comment.

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

- ✅ `useStickyOrderTitle.test.tsx` — 3 cases: hidden while intersecting, revealed when not, reset on unmount
- ✅ `OrderSidebarHeader/index.test.tsx` — 4 cases: hidden/visible states, ID-only when no external reference, suppressed in history mode
- ✅ Test mocks `usehooks-ts` while the source imports `@proofed/shared/libs/usehooks` — valid, since that file is a pure `export * from "usehooks-ts"` re-export
- ⚠️ No test covers the `Heading` style refactor (Issue 2) — behavior is visual, so manual verification of the six consumers is the right tool
- ⚠️ Animation hand-off (requirement 4) is CSS-only and untestable in jsdom — needs the manual side-by-side check against the Asana reference video
- ⏭️ Suite execution skipped — user opted out; "all existing tests pass" is unverified in this review

---

## Summary

| Aspect | Status |
|---|---|
| Correctness (vs Jira) | ✅ All requirements implemented |
| Regression risk | ❌ High — status dropdown clipped (Issue 1); shared `Heading` styles touch 5 out-of-scope consumers (Issue 2) |
| Tests | ✅ Good unit coverage for the new behavior |
| Code quality | ⚠️ Solid structure and unusually good comments, but several convention/magic-value issues (5–7) |
| Validation suite | ⏭️ Skipped — user opted out |
| Mergeable state | ⚠️ GitHub reports clean, but validation was not run and Issue 1 is a blocker |

---

## Recommendation

**Request changes**

1. **Fix Issue 1 (blocker):** the status-change dropdown menu is clipped by `HeaderLeft`'s `overflow: hidden` — restructure the clipping so the dropdown escapes, and manually verify the menu opens fully on the PR branch.
2. Visually spot-check the five out-of-scope `Heading` consumers (job/payouts/documents tables, OrderInfo, HTMLContentPill) for layout drift from the `StyledHeadingTitle` changes; drop or justify `contain: layout`.
3. Reconsider `user-select: none` on interactive titles (copyability regression).
4. Run the full validation suite (`npx turbo run test / typecheck / lint / build`) before merging — it was skipped in this review, so test/type/lint status is unverified.
5. Merge-order: land PP-1867 first, then rebase/retarget this PR onto `develop` (squash-merge of a stacked PR against a stale base will drag PP-1867's diff along).
6. Optional polish: Issues 4–7 (px/rem reveal offset, inline magic values, sx-in-index, duplicated expanded-state mechanisms).
