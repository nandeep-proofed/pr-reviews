# PR Review: fix/PP-1822: WYSIWYG — replacing selected text reintroduces deleted text in Clean version

**PR:** https://github.com/Proofed/B2BWebserver/pull/2339
**Jira:** https://proofed.atlassian.net/browse/PP-1822
**Status:** Code Review (open — awaiting reviewer `gaurav-proofed`)

> **Why is this PR still open?** It is in Jira status **Code Review** and is simply waiting on a human reviewer. There are **no human review comments** and **no requested changes** on GitHub. The only PR comment is an automated notice from the Codex bot that its credit limit was reached, so the automated AI review never ran — that is not a code objection, just a missing automated pass. GitHub reports the branch as mergeable (`clean`, no conflicts). The findings below are this review's own analysis of what a reviewer should check before approving.

> **Verification pass (rev 2):** Every finding below was re-verified against the source and the installed `prosemirror-view`. One originally-reported issue (a `posAtDOM === -1` crash path) was **retracted as a false positive** — see Issue 2. The remaining findings (1, 3, 4) were confirmed valid.

> **Follow-up commit (rev 3):** The confirmed findings were actioned in commit **`aa125a1b4`** on the PR branch (`+284 / −3`, 2 files):
> - **Issue 3 — RESOLVED:** 10 unit tests added for the keydown/beforeinput interception layer. Package tests now **262 passing** (was 252) / 4 skipped.
> - **Issue 4 — RESOLVED:** an explicit "intentional divergence" comment added to `classifyOverwrittenNode`.
> - **Issue 1 — DEFERRED + DOCUMENTED (maintainer decision):** the cross-block limitation is now called out in a code comment; a proper fix (reusing the DeletedBreak block-join machinery) is a follow-up ticket. No behaviour change.
> - **Issue 2 — no change (retracted false positive).**

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

After the verification pass, the valid findings are **all low severity**. No correctness-breaking or medium/high issues were identified. One originally-reported issue was retracted (Issue 2). Issues 1, 3 and 4 were actioned in commit `aa125a1b4`.

### 1. Cross-block (multi-paragraph) replace-over-selection still falls through to the native path

**Verdict: Valid — DEFERRED + DOCUMENTED in `aa125a1b4` (maintainer decision).**

**[File: packages/wysiwyg/src/extensions/trackChanges-v2/plugins/tracking.ts]**

**Function/Class:** buildTrackedReplacement / createKeyDownHandler / createBeforeInputHandler

**Severity:** low

**Problem:** `buildTrackedReplacement` returns `null` when the selection is not within a single text block (`!$from.sameParent($to) || !$from.parent.isTextblock`, lines 528–530). Both handlers then return `false` (keydown 802–804, beforeinput 742–744), so a selection that spans a paragraph boundary is handled by the native `readDOMChange` path — the exact path this PR replaces. `handleReplacement` additionally bails on block content (`if (hasBlockContent) return;`, line 212), so a cross-block replacement is not tracked by the new logic at all.

**Impact:** For selections spanning two or more paragraphs, the interception does not apply. The reported symptom (deleted text reintroduced, or the edit going untracked) can therefore still occur in the multi-block case. This is an edge case relative to the ticket's single-paragraph reproduction.

**Resolution:** Investigation showed a correct cross-block replace must delete the paragraph break too, which requires reusing the `deletedBreak` block-join machinery in the keyboard deletion handler (`handleRangeDeletion` — `tr.join` + `deletedBreak` marker, with list / `contentField` guards). That is a larger, higher-risk change; per maintainer decision it is **deferred to a follow-up ticket** rather than bolted on partially (an inconsistent partial join would re-introduce the very reappearing-text corruption this PR fixes). A `KNOWN LIMITATION (PP-1822)` comment now documents this at the guard. **Action for the team:** note the limitation in the PR description / QA plan and file the follow-up ticket for tracked cross-block replace.

### 2. ~~`resolveBeforeInputRange` can yield out-of-range positions if `posAtDOM` returns `-1`~~ — RETRACTED (false positive)

**Verdict: Invalid — retracted after verifying `prosemirror-view` source.**

**[File: packages/wysiwyg/src/extensions/trackChanges-v2/plugins/tracking.ts]**

**Function/Class:** resolveBeforeInputRange

**Severity:** none (not a defect)

**Original claim:** that `view.posAtDOM(...)` could return `-1`, slip past the `typeof start === "number"` guard, and later throw inside `buildTrackedReplacement` uncaught.

**Why it is invalid:** In the installed `prosemirror-view`, `posAtDOM` does **not** return `-1` — it throws a `RangeError` when the DOM position can't be mapped:

```javascript
posAtDOM(node, offset, bias = -1) {
    let pos = this.docView.posFromDOM(node, offset, bias);
    if (pos == null)
        throw new RangeError("DOM position not inside the editor");
    return pos;
}
```

Both `posAtDOM` calls sit **inside** the `try { … } catch (_) { /* fall through to the editor selection */ }` block, so any such throw is caught and the function correctly falls back to `view.state.selection`. There is therefore no `-1` value and no uncaught-throw path. The `typeof start === "number"` check is redundant (always true) but harmless — a cosmetic nit at most, not a bug. **No change required.**

### 3. The DOM-event wiring itself is not unit-tested

**Verdict: Valid — RESOLVED in `aa125a1b4`.**

**[File: packages/wysiwyg/src/extensions/trackChanges-v2/plugins/tracking.test.ts]**

**Function/Class:** createKeyDownHandler / createBeforeInputHandler / resolveBeforeInputRange

**Severity:** low

**Problem (original):** Tests exercised `buildTrackedReplacement` and `handleReplacement` directly, but the interception layer — key filtering (`ctrl/meta/alt/isComposing/length !== 1`), the `insertText` vs `insertReplacementText` branch, `event.data` vs `dataTransfer` extraction, and the `preventDefault`/`dispatch` flow — was only "verified manually in the editor". The `keydown` vs `beforeinput` split is the heart of the "first time wrong" fix, so it carried regression risk with zero automated coverage.

**Resolution:** A new `Scenario 4d` describe block adds **10 tests** driving the real plugin props (`plugin.props.handleKeyDown` and `plugin.props.handleDOMEvents.beforeinput`) via a mock view/event:
- keydown: a plain key over a selection dispatches a tracked replacement (inserts exactly the typed char) and calls `preventDefault`; and is correctly ignored for a modifier combo, a collapsed selection, a non-printable key (`length !== 1`), and an active IME composition.
- beforeinput: `insertText` and `insertReplacementText` (text pulled from `dataTransfer`) both dispatch a tracked replacement; and are ignored for a composition inputType, empty text, and a collapsed selection.

Package tests now **262 passing** (was 252) / 4 skipped; typecheck, lint and build all clean.

### 4. Own-insertion detection now ignores the local insertion DecorationSet — a narrow collaboration trade-off

**Verdict: Valid — RESOLVED (documented) in `aa125a1b4`.**

**[File: packages/wysiwyg/src/extensions/trackChanges-v2/plugins/tracking.ts]**

**Function/Class:** classifyOverwrittenNode

**Severity:** low

**Problem:** By design (and correctly, to fix the stale-decoration first-edit bug), `classifyOverwrittenNode` detects "the user's own pending insertion" from the **track-change mark only**, deliberately not the local insertion `DecorationSet`. The keyboard deletion path still consults `hasLocalInsertionDecoration` as a fallback for the PP-1774 Path B scenario, where Collaboration/Yjs strips the INSERTION mark — confirmed at `keyboard/handleDeletion.ts:602` and `:701`. This means the replace path and the delete path use *different* detection rules for the same concept, which a future maintainer could mistake for a bug and "fix".

**Resolution:** An explicit `INTENTIONAL DIVERGENCE — do NOT "align" this with the deletion path` comment was added to `classifyOverwrittenNode`, stating that re-adding the DecorationSet check here would resurrect the PP-1822 stale-decoration bug and that the two paths trade off different risks on purpose. (The underlying trade-off was already the correct call; this just makes it maintainer-proof.)

---

## Validation Checks

Run in place against the PR branch after the follow-up commit `aa125a1b4`. Node v22.12.0, Yarn 1.22.21. The PR package (`@proofed/wysiwyg-editor`) checks were re-run after the change; the two unrelated repo-wide failures below were already present before this PR.

| Check | Result | Notes |
|---|---|---|
| `@proofed/wysiwyg-editor` test | ✅ Pass | **262 passed / 4 skipped** (up from 252 — the 10 new Scenario 4d handler tests). |
| `@proofed/wysiwyg-editor` typecheck | ✅ Pass | 0 type errors. |
| `@proofed/wysiwyg-editor` lint | ✅ Pass | 0 errors. |
| `@proofed/wysiwyg-editor` build | ✅ Pass | Rollup build succeeds (only the pre-existing "mixing named and default exports" warning on `src/index.tsx`). |
| repo-wide `npx turbo run test` | ⚠️ 1 unrelated failure | `packages/shared/utils/formatWordQuantity.test.ts` ("expected `1,000,000 words`, received `10,00,000 words`") — an **en-IN locale** number-grouping artifact of the run environment, **pre-existing and unrelated** (not in `packages/wysiwyg`). |
| repo-wide `npx turbo run lint` | ⚠️ 1 unrelated failure | `apps/creative-portal/components/molecules/JobReturnTimesTray/index.test.tsx` (5 `prettier/prettier` errors) — **pre-existing and unrelated**. |

**Verdict on validation:** The PR's own package passes test / typecheck / lint / build cleanly, and the pre-commit hook (`lint-staged`) ran on commit. The two repo-wide failures live in `packages/shared` and `apps/creative-portal` and are not introduced by this PR (which only touches `packages/wysiwyg`); they should be flagged/fixed on `develop` so a full `turbo run` can go green.

---

## Tests

- ✅ **PR package tests pass:** `@proofed/wysiwyg-editor` — 262 passed / 4 skipped.
- ✅ Core builder coverage: replace-over-selection inserts exactly the typed text; overwritten own-insertion dropped; prior deletion preserved with its own id; new deletion shares the insertion's id (Replace grouping).
- ✅ Formatting-inheritance tests both ways: no stray bold from a discarded bold insertion; genuine bold original text stays bold.
- ✅ Regression test for the actual root cause: stale insertion decoration over original text must stay a Replace and keep formatting.
- ✅ Collapsed-selection guard test (`buildTrackedReplacement` returns null → normal insertion path still runs).
- ✅ **NEW (Issue #3):** 10 keydown/beforeinput interception tests (Scenario 4d) — key/inputType filtering, `dataTransfer` extraction, IME/modifier/collapsed guards, and `preventDefault`/`dispatch`.
- ✅ Existing PP-1774 scenario tests are left unchanged, so they guard the `handleReplacement` rewrite.
- ⚠️ Remaining gap (accepted): no test for cross-block replace-over-selection — deferred with Issue #1 as a documented limitation / follow-up.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Sound for the in-block case; ⚠️ multi-block deferred as documented limitation (Issue #1) |
| Regression risk | ✅ Low — new keydown path is tightly gated (`enabled` + non-empty + same textblock); `handleReplacement` rewrite is guarded by unchanged PP-1774 tests |
| Tests | ✅ Builder + interception layer now both covered (262 passing) |
| Code quality | ✅ Well-factored, single-source classifier, matches existing `handleRangeDeletion` convention; intentional divergence documented |
| Validation suite | ✅ PR package green (test/typecheck/lint/build). Two repo-wide failures exist (`formatWordQuantity` test, `JobReturnTimesTray` lint) but are pre-existing and unrelated to this PR |
| Mergeable state | ✅ GitHub reports `clean` (no conflicts) |

---

## Recommendation

**Approve with suggestions** — the confirmed findings have been addressed in commit `aa125a1b4`.

1. ✅ **Issue #3 (resolved):** interception-layer unit tests added (262 passing).
2. ✅ **Issue #4 (resolved):** intentional-divergence comment added.
3. ✅ **Issue #1 (deferred + documented):** cross-block limitation documented in code; **file a follow-up ticket** for tracked cross-block replace (needs the DeletedBreak block-join machinery), and call the limitation out in the PR description / QA plan.
4. **Not a blocker for this PR, but flag to the team:** a clean repo-wide `npx turbo run test` and `npx turbo run lint` currently fail on two **unrelated, pre-existing** items — `packages/shared/utils/formatWordQuantity.test.ts` (en-IN locale grouping in the run environment) and `apps/creative-portal/.../JobReturnTimesTray/index.test.tsx` (prettier). Neither is in `packages/wysiwyg`. CLAUDE.md requires a green run before merge, so these should be resolved on `develop` (or confirmed as environment-only).
5. Optional nit: drop the redundant `typeof … === "number"` guard in `resolveBeforeInputRange` (always true), or leave as-is — no functional impact (Issue #2).

The core fix is correct, root-cause-focused, and now well-tested for both the builder and the interception layer of the single-text-block case that matches the ticket's reproduction. No high- or medium-severity code issues remain.
