# PR Review: feature/PP-1875 feedback truncation

**PR:** https://github.com/Proofed/B2BWebserver/pull/2300
**Jira:** https://proofed.atlassian.net/browse/PP-1875
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1a — Truncate feedback > 1000 chars in Admin order side panel + Creative user history | `JobSubmission` (admin) and `ReviewHistoryPreview` (creative) pass `isCollapsible: true` through `DescriptionList` → `RichTextEditor` | ✅ |
| 1b — ≤ 1000 chars shown in full, no control | `truncateHtmlByCharCount` returns `isTruncated: false`; component returns bare editor | ✅ |
| 1c — Count raw text, exclude HTML/markup | Char walk skips tags; entities counted as 1 | ✅ (counts inter-tag whitespace too — see issue #4) |
| 2a — "Read more" appears immediately below truncated text | `StyledCollapseWrapper` (flex column) renders editor then button | ✅ |
| 2b — Green underlined, no border/bg/padding | `StyledReadMoreButton`: `green1`, `text-decoration: underline`, `border:0; background:none; padding:0; appearance:none` | ✅ (confirm `green1` matches Figma green token) |
| 2c — Hover: pointer cursor only | `cursor: pointer`, no `:hover` rule | ✅ |
| 2d — Styled consistently across surfaces | Single shared `RichTextEditor` styling used everywhere | ✅ |
| 3a/3b/3c — Click expands inline, swaps to "Read less", same style | `isExpanded` state toggles label + `displayContent` | ✅ |
| 4a/4b/4c — "Read less" collapses to the same truncated point, restores "Read more" | Same memoized `truncateHtmlByCharCount` result reused | ✅ |
| 5a — All instances in Admin order side panel | `JobSubmission` covered; `ReviewSubmission` is an editable form (out of scope) | ✅ |
| 5b — All instances in Creative user history | `ReviewHistoryPreview` covered | ✅ |
| 5c — Reusable for future surfaces | Prop added to shared `RichTextEditor` + `DescriptionList`; also wired into `FeedbackPreview` popover | ✅ |
| Exactly 1000 / 1001 boundary | Unit tests cover both | ✅ |

**Scope notes:** `FeedbackPreview` (HistoryTable / FeedbackDots popover) is wired up as a bonus surface beyond the two required by the ticket — consistent with 5c. No scope creep beyond feedback rendering; API untouched as the ticket requires.

---

## Architecture Analysis

Rather than the separate `TruncatedRichText` molecule promised in the PR description, the implementation extends the existing shared `RichTextEditor` with two opt-in props (`isCollapsible`, `charLimit`). Truncation is computed by a pure helper `truncateHtmlByCharCount` that walks the raw HTML once, counts visible characters (tags skipped, entities = 1 char), records the cutoff, and re-closes any still-open tags from a tag stack. The toggle re-feeds either the truncated or full sanitised HTML to the Tiptap provider, which calls `setContent` on change.

This is a clean, reuse-first approach (single shared component, single helper, threaded through `DescriptionList`) and matches the codebase's atomic-design conventions. The helper is a hand-rolled HTML scanner rather than a DOM parser, which keeps it SSR-safe and dependency-free but introduces the parsing edge cases noted below.

The one structural smell is that **HTML sanitisation is now coupled to the truncation code path** (see issue #1).

---

## Issues Found

### 1. Sanitisation is now conditional on the truncation feature flag

**[File: apps/creative-portal/components/molecules/FeedbackPreview/index.tsx]**
**Function/Class:** FeedbackPreview
**Severity:** medium
**Problem:** The PR removes the explicit `sanitizeHtml(...)` wrapper (and its import) around the feedback content and relies on `RichTextEditor` to sanitise internally. But `RichTextEditor` only sanitises inside the `isTruncationEnabled` branch (`!editable && isCollapsible && typeof content === "string"`). FeedbackPreview happens to pass `editable={false}` + `isCollapsible`, so it is sanitised today — but the guarantee now depends on the `isCollapsible` prop staying set. If someone later drops `isCollapsible` here (e.g. deciding the popover shouldn't truncate), sanitisation silently disappears with no obvious signal.
**Impact:** Latent XSS/robustness regression: sanitisation moved from "always" to "only when collapsible". Actual exploitability is reduced because Tiptap's ProseMirror parser drops nodes/attributes outside its schema (defence in depth), but the explicit DOMPurify guarantee is no longer unconditional for read-only content.
**Fix:** Sanitise read-only content unconditionally in `RichTextEditor` so callers can safely stop sanitising, instead of gating it behind truncation. For example compute the sanitised value once and use it for both the truncated and non-truncated returns:

```typescript
const sanitizedContent =
  !editable && typeof content === "string"
    ? sanitizeHtml(content)
    : content;

const { displayContent, isTruncated } = useMemo(() => {
  if (!isTruncationEnabled || typeof sanitizedContent !== "string") {
    return { displayContent: sanitizedContent, isTruncated: false };
  }
  const result = truncateHtmlByCharCount(sanitizedContent, charLimit);
  return {
    displayContent:
      isExpanded || !result.isTruncated ? sanitizedContent : result.html,
    isTruncated: result.isTruncated
  };
}, [sanitizedContent, charLimit, isTruncationEnabled, isExpanded]);
```

Alternatively, keep the explicit `sanitizeHtml` call in `FeedbackPreview` so its safety doesn't depend on a sibling feature flag.

### 2. State and memo logic live directly in `index.tsx`

**[File: packages/shared/components/molecules/RichTextEditor/index.tsx]**
**Function/Class:** RichTextEditor
**Severity:** low
**Problem:** `RichTextEditor` is now non-trivial — it holds `useState` and a `useMemo` directly in `index.tsx`. CLAUDE.md explicitly states: *"Do not put `useState`/`useEffect`/`useCallback`/`useMemo` directly in `index.tsx` for non-trivial components"* — these belong in a sibling `hooks.ts` exporting a single `use<ComponentName>` hook.
**Impact:** Convention drift; `index.tsx` is no longer UI-only.
**Fix:** Extract the truncation/expand logic into `RichTextEditor/hooks.ts`:

```typescript
// hooks.ts
export const useRichTextEditor = ({ content, charLimit, editable, isCollapsible }) => {
  const [isExpanded, setIsExpanded] = useState(false);
  const isTruncationEnabled =
    !editable && isCollapsible && typeof content === "string";
  const { displayContent, isTruncated } = useMemo(() => { /* ... */ }, [...]);
  return { displayContent, isTruncated, isTruncationEnabled, isExpanded, toggleExpanded: () => setIsExpanded((v) => !v) };
};
```

`index.tsx` then destructures the hook and renders.

### 3. Cutoff at a block boundary can leave an empty trailing element

**[File: packages/shared/components/molecules/RichTextEditor/utils.ts]**
**Function/Class:** truncateHtmlByCharCount
**Severity:** low
**Problem:** When the 1000th visible character is the last character of one block and the next visible character starts a fresh block, the `cutoffIndex` lands at the start of the new block's text. The slice keeps the just-opened tag, and the open-tag stack then re-closes it, producing an empty element. e.g. `<p>{1000 chars}</p><p>more</p>` → truncated to `<p>{1000 chars}</p><p></p>`.
**Impact:** A stray empty paragraph/list-item renders as a blank line between the feedback and the "Read more" control. Cosmetic only.
**Fix:** After computing `truncated`, trim trailing empty elements before re-closing, or skip pushing a freshly opened tag onto the stack if no visible text was consumed inside it. Add a regression test for the block-boundary case.

### 4. Inter-tag whitespace counts toward the 1000-char limit

**[File: packages/shared/components/molecules/RichTextEditor/utils.ts]**
**Function/Class:** truncateHtmlByCharCount
**Severity:** low
**Problem:** Requirement 1c is "raw text content, not including any HTML or markup." The walker counts every non-tag character, including newlines/spaces that sit *between* block tags in pretty-printed HTML. Tiptap's `getHTML()` typically emits no inter-tag whitespace, so in practice this rarely bites, but pasted or server-stored HTML could include it and truncate slightly early.
**Impact:** Minor off-by-a-few-chars truncation for whitespace-heavy markup; not user-visible in the common case.
**Fix:** Optionally collapse/ignore whitespace runs between tags, or document that the count includes literal text whitespace. Low priority given the typical input.

### 5. Naive scanner mis-parses `>` inside attribute values

**[File: packages/shared/components/molecules/RichTextEditor/utils.ts]**
**Function/Class:** truncateHtmlByCharCount
**Severity:** low
**Problem:** Tag boundaries are found with `html.indexOf(">", index)`, which finds the first `>` even if it appears inside an attribute value (e.g. `<a title="a > b">`). This would split the tag mid-attribute and corrupt the remainder.
**Impact:** Malformed truncation output for such markup. Low risk because the content is `sanitizeHtml`-processed first and feedback HTML almost never contains `>` inside attribute values.
**Fix:** Acceptable to leave as-is given the sanitised, constrained input; if hardening is desired, skip over quoted attribute values when locating the closing `>`.

### 6. PR description describes a component that doesn't exist

**[File: PR description]**
**Function/Class:** —
**Severity:** low
**Problem:** The description states it "Adds shared `TruncatedRichText` molecule (`packages/shared/components/molecules/TruncatedRichText`)", but the actual implementation extends `RichTextEditor` with `isCollapsible`/`charLimit`. No `TruncatedRichText` molecule exists in the diff.
**Impact:** Reviewer/future-reader confusion; the description doesn't match the code.
**Fix:** Update the PR description to reflect the `RichTextEditor` extension approach.

### 7. Expanded state can persist across content changes in `DescriptionList`

**[File: packages/shared/components/molecules/DescriptionList/index.tsx]**
**Function/Class:** DescriptionList
**Severity:** low
**Problem:** The `RichTextEditor` inside the items map isn't keyed by content (the `Fragment` is keyed by `index`). If the same list position renders different feedback (e.g. switching jobs in the sidebar without a remount), `isExpanded` state may carry over, showing the new feedback already expanded with "Read less".
**Impact:** Minor UX inconsistency in edge navigation cases; usually masked by parent remounts.
**Fix:** If observed in manual testing, key the editor by a stable content identifier so state resets when the feedback changes.

---

## Tests

- ✅ `utils.test.ts` — 12 cases covering below/at/over limit, markup-ignored, entity = 1 char, void/self-closing tags, open-tag re-closing, nested lists, empty input.
- ✅ `index.test.tsx` — 9 cases covering no-toggle when not collapsible/short/at-limit, truncate + toggle expand/collapse, markup ignored, entities, and `editable` bypass.
- ✅ Boundary cases (exactly 1000 / 1001) explicitly tested — covers the two `❌` items left untested in the Jira testing notes.
- ⚠️ Tests mock the `RichTextEditorContextProvider`, so they verify the `content` prop handed to the provider but not that Tiptap actually re-renders truncated→full on click end-to-end. Reasonable unit boundary, but an integration/visual check is worth doing manually.
- ⚠️ No regression test for the empty-trailing-block boundary (issue #3).
- ⚠️ Consumer wiring (`isCollapsible` props) is untested, but the changes are one-liners — acceptable.
- ❓ Manual/visual confirmation against Figma (green token, control placement, hover) not evidenced beyond screenshots; CLAUDE.md asks for side-by-side Figma comparison.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Meets all acceptance criteria; minor edge artifacts |
| Regression risk | ⚠️ Medium — sanitisation now coupled to the truncation flag (#1); other consumers default `isCollapsible=false` so unaffected |
| Tests | ✅ Strong unit coverage of new logic |
| Code quality | ⚠️ Good, but hooks-in-`index.tsx` convention break (#2) |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve with suggestions**

1. Address #1 — make read-only sanitisation unconditional in `RichTextEditor` (or keep the explicit `sanitizeHtml` in `FeedbackPreview`) so the XSS guarantee doesn't depend on `isCollapsible`.
2. Address #2 — move `useState`/`useMemo` into `RichTextEditor/hooks.ts` per CLAUDE.md.
3. Fix #3 (empty trailing block) and add a boundary regression test.
4. Update the PR description (#6) to match the actual `RichTextEditor` approach.
5. Do a quick manual/Figma side-by-side to confirm the green token, control placement, and expand/collapse round-trip render correctly in Tiptap (tests mock the provider).
