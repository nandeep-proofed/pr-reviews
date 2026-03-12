# PR Review: fix/PP-1727: Fix AI change side bubbles missing in read-only editor

**PR:** https://github.com/Proofed/B2BWebserver/pull/2148
**Jira:** https://proofed.atlassian.net/browse/PP-1727
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| AI side bubbles/cards visible when AI track changes are in content | Extracts AI changes from submitted YJS base64 state via `getAiChangesFromStateUpdate` | ✅ Addressed |
| Applies to completed/submitted orders (read-only mode) | Falls back to `submittedAiChanges` when no `collaborationProvider` | ✅ Addressed |
| Correct mapping between editor mark `data-id` and AI-change IDs | `getAiChange(id)` falls back to local `aiChanges` state by ID lookup | ✅ Addressed |

**Bonus fix (not in Jira):** Reject button hidden in read-only mode — sensible UX improvement since you can't reject changes on a completed order.

---

## Architecture Analysis

The fix follows the existing pattern cleanly:
- `getAiChangesFromStateUpdate` mirrors `getThreadsFromStateUpdate` (same file, same pattern)
- `submittedAiChanges` is extracted in provider via `useMemo` (only recomputes when base64 data changes)
- Fallback logic in `useAiChanges` is minimal and targeted

---

## Issues Found

### 1. Hardcoded `"__ai_changes__"` string instead of using shared constant

**[File: packages/wysiwyg/src/utils/getJsonFromStateUpdate.ts]**
**Function/Class:** getAiChangesFromStateUpdate
**Severity:** low
**Problem:** The function hardcodes `"__ai_changes__"` while `AiChangesProvider` exports `AI_CHANGES_PREFIX` for this exact purpose. The test file also defines its own local `AI_CHANGES_KEY` constant.
**Impact:** If the key ever changes in `AiChangesProvider`, these would silently diverge, causing AI changes to not load in read-only mode.
**Fix:** Import and reuse the shared constant:

```typescript
import { AI_CHANGES_PREFIX } from "../providers/AiChangesProvider";
// ...
.getArray<Y.Map<unknown>>(AI_CHANGES_PREFIX)
```

### 2. `getAiChanges()` hook method not updated

**[File: packages/wysiwyg/src/hooks/useAiChanges/index.ts]**
**Function/Class:** getAiChanges
**Severity:** low
**Problem:** `getAiChange(id)` now falls back to local state, but `getAiChanges()` still returns `[]` when there's no provider. If any component uses `getAiChanges()` instead of the `aiChanges` state directly, it won't get submitted changes.
**Impact:** Inconsistency — some consumers get data, others don't, depending on which method they call.
**Fix:** Add the same fallback:

```typescript
const getAiChanges = useCallback(
  (options?: GetAiChangesOptions) => {
    if (!collaborationProvider) {
      return aiChanges; // fallback to local state
    }
    return collaborationProvider.getAiChanges(options);
  },
  [collaborationProvider, aiChanges]
);
```

### 3. Multiple `Y.Doc` instantiations in same file

**[File: packages/wysiwyg/src/utils/getJsonFromStateUpdate.ts]**
**Function/Class:** getAiChangesFromStateUpdate, getThreadsFromStateUpdate, getJsonFromStateUpdate
**Severity:** low
**Problem:** Each function creates its own `Y.Doc` and calls `Y.applyUpdate`. If the provider calls multiple of these for the same state, the YJS document is decoded 2-3 times.
**Impact:** Minor performance overhead. Not a problem now since they're called in different `useMemo` hooks.
**Fix:** Pre-existing pattern, not introduced by this PR. No change needed now, but a shared helper could be more efficient in the future.

### 4. No test for `AiChangeBox` read-only behavior

**[File: packages/wysiwyg/src/components/molecules/AiChangeBox/index.tsx]**
**Function/Class:** AiChangeBox
**Severity:** low
**Problem:** The reject button hide logic (`!isReadOnly`) has no unit test.
**Impact:** No automated verification that `RejectChangeButton` is hidden in read-only mode.
**Fix:** Add a test asserting `RejectChangeButton` is not rendered when `isReadOnly={true}` and is rendered when `isReadOnly={false}`.

### 5. `getAiChange` callback stability

**[File: packages/wysiwyg/src/hooks/useAiChanges/index.ts]**
**Function/Class:** getAiChange
**Severity:** low
**Problem:** Adding `aiChanges` to the `getAiChange` dependency array means the callback is recreated on every AI changes state update.
**Impact:** Could trigger unnecessary re-renders in consuming components that depend on the `getAiChange` reference. Low severity since it only applies in read-only mode where changes are static.
**Fix:** No change needed — impact is minimal in read-only mode.

---

## Tests

- ✅ `getAiChangesFromStateUpdate` — 3 tests (empty state, extraction, rejected changes)
- ✅ All 83 existing wysiwyg tests pass
- ✅ Typecheck passes
- ✅ Lint passes
- ❌ No test for `AiChangeBox` read-only reject button visibility
- ⬜ Manual: Open completed order in view-only mode → AI change side bubbles visible
- ⬜ Manual: Reject button hidden in read-only mode
- ⬜ Manual: Reject button still visible in live editor mode
- ⬜ Manual: Track change cards still render correctly alongside AI change cards

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ All Jira requirements addressed |
| Regression risk | ✅ Low — fallback-only changes, live editor path untouched |
| Tests | ✅ Present for new utility; ⚠️ Missing for UI behavior |
| Code quality | ✅ Clean, follows existing patterns |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve with suggestions**
1. Use shared `AI_CHANGES_PREFIX` constant instead of hardcoded string
2. Add `getAiChanges()` fallback for consistency with `getAiChange()`
3. Consider adding a unit test for reject button visibility in read-only mode
