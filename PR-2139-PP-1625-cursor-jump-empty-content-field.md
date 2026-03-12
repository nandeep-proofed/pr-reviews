# PR Review: fix/PP-1625: Fix cursor jump in visually empty content field

**PR:** https://github.com/Proofed/B2BWebserver/pull/2139
**Jira:** https://proofed.atlassian.net/browse/PP-1625
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Text persists after revert and cannot be deleted | `filterTransaction` final guard now uses `transaction.doc` instead of manual step simulation, preventing silent schema failures | ✅ Addressed |
| After reverting changes, text should be removed | `handleRangeDeletion` now inserts empty paragraph when CF becomes visually empty, giving ProseMirror a valid cursor target | ✅ Addressed |
| User should be able to delete text to clear the block | `handleSingleCharDeletion` backward skip logic skips over already-deleted text; empty paragraph insertion on visually-empty CF | ✅ Addressed |
| Text jumps to block above when typing | `handleDeletion` range clamping restricts deletion to single CF boundary; `restrictSelectionToSingleContentField` plugin re-enabled | ✅ Addressed |
| No regression on PP-1520 (contentFields not removed) | Final guard explicitly checks `afterCounts.length < beforeCounts.length` to block CF removal; runs for ALL transactions including `skipTracking`/`skipTrackChanges` | ✅ Addressed |

---

## Architecture Analysis

This is a complex, multi-layered fix addressing a ProseMirror/contentField interaction bug. The approach uses three defensive layers:

1. **`filterTransaction` hardening** — The final guard now always runs (even for user input and skip* meta transactions), uses `transaction.doc` instead of unreliable manual step application, and explicitly blocks CF removal. The `addToHistory !== false` blanket allow (which was always true for undefined) is removed.

2. **`handleDeletion` range clamping** — Range deletions spanning multiple contentFields are clamped to the originating CF boundary, preventing cross-CF corruption.

3. **`handleDeletion` empty paragraph insertion** — When a CF becomes visually empty (all text has deletion marks), an empty paragraph is inserted as a cursor target, preventing orphan DOM text.

The `restrictSelectionToSingleContentField` plugin is also re-enabled (was previously disabled as a workaround).

---

## Issues Found

### 1. Duplicated `getAncestorContentFieldPos` implementation

**[File: packages/wysiwyg/src/extensions/trackChanges-v2/keyboard/handleDeletion.test.ts]**
**Function/Class:** getAncestorContentFieldPos
**Severity:** low
**Problem:** The test file re-implements `getAncestorContentFieldPos` instead of importing it from the source. The source function at `handleDeletion.ts:85` is not exported, so the test creates its own copy. If the production logic changes, the test's copy won't be updated.
**Impact:** Test could pass while production code has different behavior. Low risk since the function is simple.
**Fix:** Export `getAncestorContentFieldPos` from `handleDeletion.ts` and import it in the test. Alternatively, add a comment noting the duplication.

### 2. `isContentFieldVisuallyEmpty` only checks `trackChange` deletion marks, not all deletion types

**[File: packages/wysiwyg/src/extensions/trackChanges-v2/keyboard/handleDeletion.ts]**
**Function/Class:** isContentFieldVisuallyEmpty
**Severity:** medium
**Problem:** The function checks for `isTrackChangeMark(mark) && mark.attrs["data-change-type"] === ChangeType.DELETION` but doesn't check for V1 deletion marks (`mark.type.name === "deletion"`) or AI deletion marks (`mark.type.name === "del"`). The `cleanContentForCopy.ts` utility uses `isAnyDeletionMark` which handles all three types.
**Impact:** If a CF contains only V1 or AI deletion-marked text, `isContentFieldVisuallyEmpty` returns `false` (thinks there's visible content), and the empty paragraph won't be inserted. The cursor jump bug would still occur in those scenarios.
**Fix:** Use the existing `isAnyDeletionMark` helper from `cleanContentForCopy.ts`, or replicate its full check:

```typescript
import { isAnyDeletionMark } from "../utils/aiChangesHelpers";

// In isContentFieldVisuallyEmpty:
const hasDeletionMark = node.marks.some(isAnyDeletionMark);
```

### 3. Re-enabling `restrictSelectionToSingleContentField` without testing

**[File: packages/wysiwyg/src/extensions/contentField/index.tsx]**
**Function/Class:** addProseMirrorPlugins
**Severity:** medium
**Problem:** The `restrictSelectionToSingleContentField` plugin was previously disabled with the comment "might interfere with typing". It's now re-enabled without any changes to the plugin itself. The old concern may still be valid.
**Impact:** Could cause regressions in typing behavior within contentFields. The plugin wasn't part of the Jira requirements (which focus on deletion/revert behavior).
**Fix:** Either add tests specifically for `restrictSelectionToSingleContentField` to validate it doesn't interfere with typing, or keep it disabled and note in the PR description why it was re-enabled and what testing was done.

### 4. Backward skip-deleted logic doesn't check CF boundary

**[File: packages/wysiwyg/src/extensions/trackChanges-v2/keyboard/handleDeletion.ts]**
**Function/Class:** handleSingleCharDeletion (backward skip block)
**Severity:** low
**Problem:** The backward skip loop uses `blockStart` (paragraph start) as its boundary, but doesn't check the contentField boundary. If all text in a paragraph is deleted and the loop reaches `blockStart`, it stops. However, the `getAncestorContentFieldPos` check isn't applied, so in edge cases the skip could theoretically cross structural boundaries.
**Impact:** Low — `blockStart` is a paragraph-level boundary which is always within a single CF. But the defensive layering elsewhere (range clamping, filterTransaction) would catch issues regardless.
**Fix:** No change needed — the paragraph boundary is sufficient.

### 5. `shouldPreventMerge` test duplicates production logic

**[File: packages/wysiwyg/src/extensions/trackChanges-v2/keyboard/handleDeletion.test.ts]**
**Function/Class:** shouldPreventMerge
**Severity:** low
**Problem:** The test re-implements the merge prevention condition as a standalone function rather than testing the actual `handleSingleCharDeletion` function's behavior. This is a unit test of the condition itself, not of the production code.
**Impact:** Test validates the logic is correct in isolation but doesn't guarantee it's correctly wired in production. The companion test for the old (buggy) condition is a nice touch for documentation.
**Fix:** Acceptable for this type of logic test. Consider adding an integration-level test that exercises the actual handler.

### 6. Empty paragraph insertion sets `skipTracking` meta twice in range deletion

**[File: packages/wysiwyg/src/extensions/trackChanges-v2/keyboard/handleDeletion.ts]**
**Function/Class:** handleRangeDeletion
**Severity:** low
**Problem:** When a CF becomes visually empty after range deletion, the code inserts an empty paragraph with `tr.setMeta("skipTracking", true)` and dispatches. But `handleRangeDeletion` already sets `skipTracking` at the end of the function (line after the new block). The early return avoids the duplicate, but if the `isContentFieldVisuallyEmpty` check is false, the flow continues to the original `setMeta` + `dispatch`. This is correct behavior, just worth noting the two dispatch paths.
**Impact:** None — the early return in the visually-empty branch is correct.
**Fix:** No change needed.

---

## Tests

- ✅ `contentField.test.ts` — 11 new tests (meta allows, CF protection, reject scenarios)
- ✅ `handleDeletion.test.ts` — 15 new tests (ancestor position, boundary condition, meta checks, skip-deleted logic)
- ✅ All 26 extension tests pass
- ✅ Old buggy condition documented with failing test case
- ❌ No test for `isContentFieldVisuallyEmpty` utility
- ❌ No test for `restrictSelectionToSingleContentField` re-enablement
- ⬜ Manual: Select all text in CF, press Backspace, type — cursor stays in same CF
- ⬜ Manual: Undo/redo still works correctly
- ⬜ Manual: Accept/reject tracked deletions still works

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Multi-layered fix addressing root causes |
| Regression risk | ⚠️ Medium — `restrictSelectionToSingleContentField` re-enabled without dedicated tests; `isContentFieldVisuallyEmpty` may miss V1/AI deletion types |
| Tests | ✅ Good coverage (26 tests); ⚠️ Missing coverage for new utility and re-enabled plugin |
| Code quality | ✅ Well-structured with clear comments referencing PP-1625 |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve with suggestions**
1. Update `isContentFieldVisuallyEmpty` to use `isAnyDeletionMark` to handle all deletion types (V1, V2, AI)
2. Add a test or manual verification note for `restrictSelectionToSingleContentField` re-enablement
3. Consider exporting `getAncestorContentFieldPos` to avoid test duplication
