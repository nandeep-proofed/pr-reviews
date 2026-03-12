# PR Review: fix/PP-1664: Fix Copy All to Clipboard losing formatting in Word

**PR:** https://github.com/Proofed/B2BWebserver/pull/2138
**Jira:** https://proofed.atlassian.net/browse/PP-1664
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| "Copy All to Clipboard" + paste into Word loses spacing (words run together) | Changed from `navigator.clipboard.writeText()` (plain text only) to `ClipboardItem` API with `text/html` MIME type using `serializeCleanContent()` for HTML serialization | ✅ Addressed |
| Bullet list formatting lost when pasting into Word | HTML representation via `DOMSerializer` preserves `<ul>/<li>` structure which Word interprets correctly | ✅ Addressed |
| Affects Clean version | Clean version path now generates HTML via `serializeCleanContent(fragment, schema)` | ✅ Addressed |
| Affects all orders | Fix applies to the shared `CopyButton` component used across all orders | ✅ Addressed |
| Workaround: manual select+copy still works | Not broken — this only changes the button behavior | ✅ No regression |

---

## Architecture Analysis

The fix is well-targeted. The root cause was that `CopyButton` used `navigator.clipboard.writeText()` which only writes `text/plain` to the clipboard. Word requires `text/html` to preserve formatting.

The approach:
1. New `copyHtmlToClipboard` utility writes both `text/html` and `text/plain` MIME types via the `ClipboardItem` API
2. For **Clean** version: uses existing `serializeCleanContent()` (ProseMirror `DOMSerializer`) to generate HTML, and `textBetween()` for plain text
3. For **non-Clean** version: uses `editor.getHTML()` for HTML and `editor.getText()` for plain text
4. Error fallback also produces both HTML and plain text

The `serializeCleanContent` function was already available and used in the track changes copy handler — it was just not being used in the "Copy All" button. Good reuse.

---

## Issues Found

### 1. No fallback for browsers without `ClipboardItem` API

**[File: packages/wysiwyg/src/utils/copyHtmlToClipboard.ts]**
**Function/Class:** copyHtmlToClipboard
**Severity:** medium
**Problem:** The ClipboardItem API is not supported in Firefox when the page is not served over HTTPS, and was only added to Firefox in v87. While modern browsers support it, there is no graceful fallback to `navigator.clipboard.writeText()` if ClipboardItem is unavailable.
**Impact:** Copy button silently fails in unsupported browsers, showing the error toast instead of at least copying plain text.
**Fix:** Add a fallback:

```typescript
export async function copyHtmlToClipboard(
  html: string,
  plainText: string
): Promise<void> {
  if (
    typeof ClipboardItem !== "undefined" &&
    navigator.clipboard?.write
  ) {
    const htmlBlob = new Blob([html], { type: "text/html" });
    const textBlob = new Blob([plainText], {
      type: "text/plain"
    });
    const clipboardItem = new ClipboardItem({
      "text/html": htmlBlob,
      "text/plain": textBlob
    });

    await navigator.clipboard.write([clipboardItem]);
  } else {
    // Fallback: at least copy plain text
    await navigator.clipboard.writeText(plainText);
  }
}
```

### 2. Non-Clean version now copies HTML but original only copied plain text

**[File: packages/wysiwyg/src/components/atoms/CopyButton/index.tsx]**
**Function/Class:** handleCopy
**Severity:** low
**Problem:** The non-Clean (Original/Track Changes) path changed from `editor.getText()` to `editor.getHTML()`. The original version only ever copied plain text for non-Clean mode. Now it copies full HTML including track change marks (`<ins>`, `<del>`, `data-change-type` attributes) which may produce confusing formatting when pasted into Word.
**Impact:** Pasting Original/Track Changes content into Word may show raw track change markup or unexpected formatting. The Jira ticket specifically mentions the Clean version.
**Fix:** Consider whether non-Clean mode should continue using `writeText()` for plain text only, or if the HTML should be sanitized to strip track change marks. At minimum, verify manually that pasting Original version into Word is acceptable.

### 3. `htmlToPlainText` is tested but not used in the PR diff

**[File: packages/wysiwyg/src/extensions/trackChanges-v2/utils/cleanContentForCopy.test.ts]**
**Function/Class:** htmlToPlainText
**Severity:** low
**Problem:** The PR adds tests for `htmlToPlainText` but `CopyButton` doesn't use this function — it uses `doc.textBetween()` for Clean mode and `editor.getText()` for non-Clean mode. The tests are valuable for existing callers (`trackChanges-v2/index.ts` line 203) but may give a false impression they're testing the PR's copy path.
**Impact:** No functional impact. Slightly misleading test coverage.
**Fix:** No code change needed — just a note that these tests cover the existing track changes copy handler, not the CopyButton path.

### 4. Empty HTML check instead of content check

**[File: packages/wysiwyg/src/components/atoms/CopyButton/index.tsx]**
**Function/Class:** handleCopy
**Severity:** low
**Problem:** The guard changed from `if (content)` to `if (html)`. An empty document would produce `html=""` from `getHTML()` but could produce non-empty `plainText` from `getText()`. More importantly, an editor with only whitespace would produce `html="<p></p>"` which is truthy but has no meaningful content.
**Impact:** Minor — could attempt clipboard write with empty-looking HTML. No user-facing issue since Word would just paste nothing visible.
**Fix:** No change needed — edge case is harmless.

---

## Tests

- ✅ `copyHtmlToClipboard` — 4 tests (dual MIME types, bullet preservation, rich text, error propagation)
- ✅ `htmlToPlainText` — 6 tests (paragraphs, spaces, lists, formatting, empty, nesting)
- ✅ All 90 existing wysiwyg tests pass
- ❌ No test for `CopyButton` component itself (integration test for the full copy flow)
- ⬜ Manual: Paste into Word from Clean version
- ⬜ Manual: Paste into Word from Original version
- ⬜ Manual: Paste into plain text editor (Notepad)
- ⬜ Manual: Fallback when clean content generation fails

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Fixes the root cause for Clean version |
| Regression risk | ⚠️ Medium — non-Clean path now copies HTML (behavior change), no ClipboardItem fallback |
| Tests | ✅ Good coverage for utilities; missing CopyButton integration test |
| Code quality | ✅ Clean, good reuse of existing serializeCleanContent |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve with suggestions**
1. Add `ClipboardItem` API fallback to `copyHtmlToClipboard` for browser compatibility
2. Verify manually that non-Clean (Original/Track Changes) version paste into Word is acceptable — the Jira ticket only mentions Clean mode
3. Consider keeping non-Clean mode as plain-text-only if HTML track change markup causes issues in Word
