# PR Review: fix/PP-1822: WYSIWYG — replacing selected text reintroduces deleted text in Clean version

**PR:** https://github.com/Proofed/B2BWebserver/pull/2339
**Jira:** https://proofed.atlassian.net/browse/PP-1822
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Selecting text in the Clean version and typing to replace must NOT reintroduce previously deleted text | Typing over a selection is intercepted at `keydown` (backup at `beforeinput`) and rebuilt via `buildTrackedReplacement`, which never re-inserts deleted content — the unreliable `readDOMChange` path that re-materialised deleted text is bypassed | ✅ Addressed |
| Selected text is replaced with exactly the newly typed text | `buildTrackedReplacement` inserts `schema.text(text, …)` where `text` is the exact typed character; test "inserts exactly the typed text (no surrounding text absorbed)" asserts the insertion is only `"I"` | ✅ Addressed |
| Change should be a Replace, not an Add (implied correct track-changes semantics) | Original text is marked as a NEW deletion sharing one `data-change-id` with the insertion → UI groups as a single "Replace" card; asserted by `insertionId === newDeletionId` | ✅ Addressed |
| A prior deletion in/around the selection must remain its own change | Existing-deletion nodes are classified `"keep"` and left with their original `data-change-id`; asserted (`del-1` / `del-A` preserved) | ✅ Addressed (beyond literal scope) |
| First-attempt vs second-attempt (post-undo) inconsistency ("Add" first time, "Replace" second) | Root cause identified as a stale insertion `DecorationSet` mis-classifying original text; classifier now reads the track-change **mark only**. Regression test "is not fooled by a stale insertion decoration" covers it | ✅ Addressed (beyond literal scope) |
| Inserted text should keep the surrounding font / not inherit stray bold | `getOriginalFormattingMarks` inherits marks from the ORIGINAL replaced text, not the discarded pending insertion; bold/no-bold tests cover both directions | ✅ Addressed (beyond literal scope) |
| Related PP-1659 (formatting-triggered variant) | Not in scope of this PR — this is the text-replacement trigger only; the PP-1660/1659 format path (`handleFormatChange`) is untouched | ⚠️ Out of scope (correctly) |

**Scope note:** The PR is broader than the literal Jira text (which only describes deleted-text reappearing). The extra work — prior-deletion preservation, formatting inheritance, and the stale-decoration first-edit fix — are all facets of correctly handling replace-over-selection and are legitimately part of the root-cause fix rather than unrelated scope creep. This is a good thing, and each facet is backed by a regression test.

---

## Architecture Analysis

The fix targets the real root cause. Previously, typing over a selection in the Track-Changes editor was left to the browser + ProseMirror's `readDOMChange`, which produced a `ReplaceStep` whose inserted slice **absorbed adjacent struck-through deleted text** (typing "Y" over "Look" yielded an insertion of "You want to Y"), so the Clean version re-rendered the deleted text.

The PR moves interception **upstream of the DOM mutation**:

1. `createKeyDownHandler` — primary, earliest interceptor. Only acts on a single printable key (`event.key.length === 1`, no ctrl/meta/alt, not composing) with a non-empty selection. Prevents the browser from splitting the edit into `deleteContentBackward` + `insertText` (the cause of the "first time Add, second time Replace" behaviour).
2. `createBeforeInputHandler` — backup for input that isn't a plain keystroke (e.g. autocorrect's `insertReplacementText`). Resolves the replaced range from `event.getTargetRanges()` (correct even before the editor selection has synced).
3. `buildTrackedReplacement` — single source of truth. Segments the selection with `classifyOverwrittenNode` (`delete` own pending insertion / `keep` existing deletion / `mark` original), applies segments right-to-left, then inserts the typed text with explicit marks.

Crucially, `classifyOverwrittenNode` is shared by all three overwrite paths (`buildTrackedReplacement`, the `handleReplacement` fallback, and `getOriginalFormattingMarks`) so they can't drift, and it mirrors the established `handleRangeDeletion` pattern in `keyboard/handleDeletion.ts` (insertion → delete, original → mark). This is consistent with the repo's "single source of truth" convention.

The `handleReplacement` (readDOMChange fallback) rewrite is a genuine improvement independent of the interception: the old code re-inserted the entire deleted slice and then `addMark`'d a fresh deletion mark over it, which **clobbered the `data-change-id` of any pre-existing deletion** in the range. The new code keeps existing deletions untouched. Because the existing PP-1774 scenario tests (unchanged by this PR) still exercise this function, they act as a regression guard for the rewrite.

Overall: correct approach, well-factored, follows existing conventions, and the `enabled`/`empty`/`sameParent` gates keep the new keydown interception from firing outside its intended case.

---

## Issues Found

All issues found are **low severity**. No correctness-breaking or medium/high issues were identified in the static review.

### 1. Cross-block (multi-paragraph) replace-over-selection still falls through to the buggy native path

**[File: packages/wysiwyg/src/extensions/trackChanges-v2/plugins/tracking.ts]**

**Function/Class:** buildTrackedReplacement / createKeyDownHandler / createBeforeInputHandler

**Severity:** low

**Problem:** `buildTrackedReplacement` returns `null` when the selection is not within a single text block (`!$from.sameParent($to) || !$from.parent.isTextblock`). Both handlers then return `false`, so a selection that spans a paragraph boundary is handled by the native `readDOMChange` path — the exact path this PR replaces. `handleReplacement` runs for it, but it operates on the slice the browser already built, so the "absorbed surrounding text" corruption is not prevented for multi-block selections.

**Impact:** The reported symptom can still occur when the user selects across two or more paragraphs and types. This is an edge case relative to the ticket's single-paragraph reproduction, and there is no test covering it, so it may be mistaken for "fully fixed."

**Fix:** Acceptable to ship as a documented limitation, but call it out in the PR description / manual test plan so QA doesn't assume multi-block is covered. If a fuller fix is wanted later, `buildTrackedReplacement` could iterate per-textblock across the selection rather than bailing.

### 2. `resolveBeforeInputRange` can yield out-of-range positions if `posAtDOM` returns `-1`

**[File: packages/wysiwyg/src/extensions/trackChanges-v2/plugins/tracking.ts]**

**Function/Class:** resolveBeforeInputRange

**Severity:** low

**Problem:** `view.posAtDOM(...)` is typed to return a `number`, so the `typeof start === "number"` guard is always true and does not reject a `-1` (the value ProseMirror returns when the DOM node can't be mapped into the document). A `{ from: -1, to: … }` then flows into `buildTrackedReplacement`, whose `state.doc.resolve(from)` will throw. The surrounding `try/catch` only wraps the `posAtDOM` block, not the downstream `buildTrackedReplacement` call, so the throw would surface uncaught in the `beforeinput` DOM handler.

**Impact:** Very unlikely in practice (target ranges from `insertText`/`insertReplacementText` point inside the contenteditable), but if it did occur the keystroke would be dropped and an error logged instead of falling back to native handling.

**Fix:** Reject non-positive/out-of-bounds positions before returning:

```typescript
const max = view.state.doc.content.size;

if (
  start >= 0 &&
  end >= 0 &&
  start <= max &&
  end <= max
) {
  return { from: Math.min(start, end), to: Math.max(start, end) };
}
// else fall through to the editor selection
```

### 3. The DOM-event wiring itself is not unit-tested

**[File: packages/wysiwyg/src/extensions/trackChanges-v2/plugins/tracking.test.ts]**

**Function/Class:** createKeyDownHandler / createBeforeInputHandler / resolveBeforeInputRange

**Severity:** low

**Problem:** Tests exercise `buildTrackedReplacement` and `handleReplacement` directly (good coverage of the builder), but the interception layer — key filtering (`ctrl/meta/alt/isComposing/length !== 1`), the `insertText` vs `insertReplacementText` branch, `event.data` vs `dataTransfer` extraction, `getTargetRanges` resolution, and the `preventDefault`/`dispatch` flow — is only "verified manually in the editor" per the code comments. The `keydown` vs `beforeinput` split is the heart of the "first time wrong" fix, so it carries real regression risk with zero automated coverage.

**Impact:** A future refactor of the handlers (e.g. changing the inputType filter) could silently reintroduce the bug and all unit tests would still pass.

**Fix:** Add a couple of thin handler tests using a real Tiptap `Editor` + jsdom (the file already imports `Editor` for the Yjs scenarios). At minimum assert: (a) a plain keystroke over a selection dispatches a `buildTrackedReplacement` transaction and calls `preventDefault`; (b) a ctrl/meta-modified key or collapsed selection is ignored; (c) a block-spanning selection falls through (returns false). Where full DOM-event simulation is impractical in jsdom, note it explicitly in the manual test plan instead.

### 4. Typing over a selection now bypasses input rules within a text block

**[File: packages/wysiwyg/src/extensions/trackChanges-v2/plugins/tracking.ts]**

**Function/Class:** createKeyDownHandler

**Severity:** low

**Problem:** When tracking is enabled and a printable key is typed over a non-empty in-block selection, the handler `preventDefault`s and dispatches its own transaction (with `skipTracking`), so ProseMirror's normal text-input pipeline — including `handleTextInput`/input rules — never runs for that keystroke.

**Impact:** Any input-rule/markdown-style shortcut that would trigger on replace-over-selection won't fire while tracking is on. In practice input rules almost always fire on collapsed-caret typing (e.g. "1. " at line start), so real-world impact is minimal, but it is a behavioural change worth being aware of.

**Fix:** No change required; note the behaviour. If input rules on replacement ever matter, the handler could re-run them against the rebuilt transaction, but that is out of scope here.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⏭️ Skipped | Skipped — user opted out. Must be run before merge (per CLAUDE.md). PR claims 252 passing in `@proofed/wysiwyg-editor` (4 Yjs/browser-DOM tests skipped). |
| `npx turbo run typecheck` | ⏭️ Skipped | Skipped — user opted out. |
| `npx turbo run lint` | ⏭️ Skipped | Skipped — user opted out. |
| `npx turbo run build` | ⏭️ Skipped | Skipped — user opted out. |

---

## Tests

- ⏭️ Validation suite not run — user opted out. Test/typecheck/lint/build results are unverified and must be run against the branch before merge.
- ✅ New unit tests added for the core builder: replace-over-selection inserts exactly the typed text; overwritten own-insertion dropped; prior deletion preserved with its own id; new deletion shares the insertion's id (Replace grouping).
- ✅ Formatting-inheritance tests both ways: no stray bold from a discarded bold insertion; genuine bold original text stays bold.
- ✅ Regression test for the actual root cause: stale insertion decoration over original text must stay a Replace and keep formatting.
- ✅ Collapsed-selection guard test (`buildTrackedReplacement` returns null → normal insertion path still runs).
- ✅ Existing PP-1774 scenario tests are left unchanged, so they guard the `handleReplacement` rewrite.
- ⚠️ Gap: the `keydown`/`beforeinput` handlers and `resolveBeforeInputRange` have no automated coverage (see Issue #3); the DOM-event behaviour is manual-only.
- ⚠️ Gap: no test for cross-block replace-over-selection (see Issue #1).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Sound for the in-block case; ⚠️ multi-block still falls through |
| Regression risk | ✅ Low — new keydown path is tightly gated (`enabled` + non-empty + same textblock); `handleReplacement` rewrite is guarded by unchanged PP-1774 tests |
| Tests | ⚠️ Strong builder coverage; interception layer untested |
| Code quality | ✅ Well-factored, single-source classifier, matches existing `handleRangeDeletion` convention |
| Validation suite | ⏭️ Skipped — user opted out (must run before merge) |
| Mergeable state | ✅ GitHub reports `clean` (no conflicts). Note: validation not run, so this reflects merge-conflict state only. |

---

## Recommendation

**Approve with suggestions** — contingent on the validation suite passing.

1. **Blocker (process):** The mandatory `test` / `typecheck` / `lint` / `build` suite was **not run** (user opted to skip). Per CLAUDE.md this must pass before merge — re-run it against `fix/PP-1822-…` and confirm green (the PR's "252 passing" claim is for the wysiwyg package only and is unverified here).
2. Add the small robustness guard in `resolveBeforeInputRange` for `posAtDOM === -1` (Issue #2).
3. Add thin handler-level tests for the `keydown`/`beforeinput` interception, or record it in the manual test plan (Issue #3).
4. Call out in the PR description that **cross-paragraph** replace-over-selection is not covered by the interception and can still reintroduce deleted text (Issue #1) — so QA and the ticket's acceptance check are scoped correctly.

The core fix is correct, root-cause-focused, and well-tested for the single-text-block case that matches the ticket's reproduction. No high- or medium-severity code issues were found.
