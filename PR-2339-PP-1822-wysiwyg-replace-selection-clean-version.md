# PR Review: fix/PP-1822: WYSIWYG — replacing selected text reintroduces deleted text in Clean version

**PR:** https://github.com/Proofed/B2BWebserver/pull/2339
**Jira:** https://proofed.atlassian.net/browse/PP-1822
**Status:** Code Review (open — awaiting reviewer `gaurav-proofed`)

> **Why is this PR still open?** It is in Jira status **Code Review** and is simply waiting on a human reviewer. There are **no human review comments** and **no requested changes** on GitHub. The only PR comment is an automated notice from the Codex bot that its credit limit was reached, so the automated AI review never ran — that is not a code objection, just a missing automated pass. GitHub reports the branch as mergeable (`clean`, no conflicts). The findings below are this review's own analysis of what a reviewer should check before approving.

> **Verification pass (2nd revision):** Every finding below was re-verified against the source and the installed `prosemirror-view`. One originally-reported issue (a `posAtDOM === -1` crash path) was **retracted as a false positive** — see Issue 2. The remaining findings (1, 3, 4) were confirmed valid.

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

After the verification pass, the valid findings are **all low severity**. No correctness-breaking or medium/high issues were identified. One originally-reported issue was retracted (Issue 2).

### 1. Cross-block (multi-paragraph) replace-over-selection still falls through to the native path

**Verdict: Valid (confirmed in source).**

**[File: packages/wysiwyg/src/extensions/trackChanges-v2/plugins/tracking.ts]**

**Function/Class:** buildTrackedReplacement / createKeyDownHandler / createBeforeInputHandler

**Severity:** low

**Problem:** `buildTrackedReplacement` returns `null` when the selection is not within a single text block (`!$from.sameParent($to) || !$from.parent.isTextblock`, lines 528–530). Both handlers then return `false` (keydown 802–804, beforeinput 742–744), so a selection that spans a paragraph boundary is handled by the native `readDOMChange` path — the exact path this PR replaces. `handleReplacement` additionally bails on block content (`if (hasBlockContent) return;`, line 212), so a cross-block replacement is not tracked by the new logic at all.

**Impact:** For selections spanning two or more paragraphs, the interception does not apply. The reported symptom (deleted text reintroduced, or the edit going untracked) can therefore still occur in the multi-block case. This is an edge case relative to the ticket's single-paragraph reproduction, and there is no test covering it, so it may be mistaken for "fully fixed."

**Fix:** Acceptable to ship as a documented limitation, but call it out in the PR description / manual test plan so QA doesn't assume multi-block is covered. If a fuller fix is wanted later, `buildTrackedReplacement` could iterate per-textblock across the selection rather than bailing.

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

**Verdict: Valid (confirmed — test file only mentions the handlers in a comment).**

**[File: packages/wysiwyg/src/extensions/trackChanges-v2/plugins/tracking.test.ts]**

**Function/Class:** createKeyDownHandler / createBeforeInputHandler / resolveBeforeInputRange

**Severity:** low

**Problem:** Tests exercise `buildTrackedReplacement` and `handleReplacement` directly (good coverage of the builder), but the interception layer — key filtering (`ctrl/meta/alt/isComposing/length !== 1`), the `insertText` vs `insertReplacementText` branch, `event.data` vs `dataTransfer` extraction, `getTargetRanges` resolution, and the `preventDefault`/`dispatch` flow — is only "verified manually in the editor" per the code comments. Confirmed: `createKeyDownHandler` / `createBeforeInputHandler` / `resolveBeforeInputRange` are never invoked in the test file (only referenced once in a prose comment). The `keydown` vs `beforeinput` split is the heart of the "first time wrong" fix, so it carries real regression risk with zero automated coverage.

**Impact:** A future refactor of the handlers (e.g. changing the inputType filter) could silently reintroduce the bug and all unit tests would still pass.

**Fix:** Add a couple of thin handler tests using a real Tiptap `Editor` + jsdom (the file already imports `Editor` for the Yjs scenarios). At minimum assert: (a) a plain keystroke over a selection dispatches a `buildTrackedReplacement` transaction and calls `preventDefault`; (b) a ctrl/meta-modified key or collapsed selection is ignored; (c) a block-spanning selection falls through (returns false). Where full DOM-event simulation is impractical in jsdom, note it explicitly in the manual test plan instead.

### 4. Own-insertion detection now ignores the local insertion DecorationSet — a narrow collaboration trade-off

**Verdict: Valid (confirmed — the deletion path does use the DecorationSet fallback).**

**[File: packages/wysiwyg/src/extensions/trackChanges-v2/plugins/tracking.ts]**

**Function/Class:** classifyOverwrittenNode

**Severity:** low

**Problem:** By design (and correctly, to fix the stale-decoration first-edit bug), `classifyOverwrittenNode` detects "the user's own pending insertion" from the **track-change mark only**, deliberately not the local insertion `DecorationSet`. The keyboard deletion path still consults `hasLocalInsertionDecoration` as a fallback for the PP-1774 Path B scenario, where Collaboration/Yjs strips the INSERTION mark — confirmed at `keyboard/handleDeletion.ts:602` and `:701`. This means the replace path and the delete path use *different* detection rules for the same concept.

**Impact:** In a live collaboration session where Yjs has stripped the INSERTION mark from a user's own not-yet-accepted insertion, overwriting that insertion via replace-over-selection would classify it as `"mark"` (original) and turn it into a struck-through deletion, instead of dropping it — a PP-1774-style residue confined to the replace path. This is a narrow corner (requires the mark to have been stripped) and is not covered by tests (the 4 Yjs/browser-DOM tests are skipped). It is an accepted trade-off: consulting the DecorationSet here is exactly what caused the PP-1822 stale-decoration bug.

**Fix:** No change required for this ticket — the trade-off is the correct call. Worth a one-line code comment noting the intentional divergence from the deletion path so a future maintainer doesn't "fix" the inconsistency by re-adding the DecorationSet check. If the Yjs-strip corner is ever reported against the replace path, revisit with a detection that distinguishes stale decorations from live ones.

---

## Validation Checks

Run in place against `fix/PP-1822-wysiwyg-replacing-selected-text-reintroduces-deleted-text-clean-version` (local HEAD `658d4c8f2`, which is the PR tip merged up to `develop`). Node v22.12.0, Yarn 1.22.21.

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⚠️ 1 unrelated failure | **PR package `@proofed/wysiwyg-editor`: 252 passed / 4 skipped — clean.** The single repo-wide failure is `packages/shared/utils/formatWordQuantity.test.ts` ("expected `1,000,000 words`, received `10,00,000 words`") — an **en-IN locale** number-grouping artifact of the run environment, **pre-existing and unrelated** to this PR (this PR touches only `packages/wysiwyg`). |
| `npx turbo run typecheck` | ✅ Pass | 5/5 workspaces, 0 type errors (wysiwyg-editor typechecks clean). |
| `npx turbo run lint` | ⚠️ 1 unrelated failure | Failure is in `apps/creative-portal/components/molecules/JobReturnTimesTray/index.test.tsx` (5 `prettier/prettier` errors) — **pre-existing and unrelated**; no `packages/wysiwyg` file is implicated. The wysiwyg workspace lints clean. |
| `npx turbo run build` | ✅ Pass | 4/4 workspaces built successfully; wysiwyg Rollup build and both Next.js apps compiled with no errors. |

**Verdict on validation:** Both failures (`formatWordQuantity` test, `JobReturnTimesTray` lint) are pre-existing issues on the branch that have **nothing to do with the PP-1822 change** — they live in `packages/shared` and `apps/creative-portal`, while this PR only modifies `packages/wysiwyg`. The PR's own package passes test, typecheck, lint and build cleanly. Per CLAUDE.md a fully green `turbo run` is required before merge, so these two unrelated failures should be flagged to the team (they will block a clean run for everyone), but they are **not** introduced by and **not** the responsibility of this PR.

---

## Tests

- ✅ **PR package tests pass:** `@proofed/wysiwyg-editor` — 252 passed / 4 skipped (the 4 skipped are the Yjs/browser-DOM scenarios, as the PR states).
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
| Validation suite | ⚠️ PR package green (test/typecheck/lint/build). Two repo-wide failures exist (`formatWordQuantity` test, `JobReturnTimesTray` lint) but are pre-existing and unrelated to this PR |
| Mergeable state | ✅ GitHub reports `clean` (no conflicts) |

---

## Recommendation

**Approve with suggestions.**

The core fix is correct, root-cause-focused, and well-tested for the single-text-block case that matches the ticket's reproduction. The PR's own package passes test / typecheck / lint / build. After the verification pass, no high- or medium-severity code issues remain (the one crash-path concern was retracted as a false positive — Issue 2).

1. **Not a blocker for this PR, but flag to the team:** a clean repo-wide `npx turbo run test` and `npx turbo run lint` currently fail on two **unrelated, pre-existing** items — `packages/shared/utils/formatWordQuantity.test.ts` (en-IN locale grouping in the run environment) and `apps/creative-portal/.../JobReturnTimesTray/index.test.tsx` (prettier). Neither is in `packages/wysiwyg`. CLAUDE.md requires a green run before merge, so these should be resolved on `develop` (or confirmed as environment-only) so the branch can go green.
2. Add thin handler-level tests for the `keydown`/`beforeinput` interception, or record it in the manual test plan (Issue #3).
3. Call out in the PR description that **cross-paragraph** replace-over-selection is not covered by the interception and can still reintroduce deleted text (Issue #1) — so QA and the ticket's acceptance check are scoped correctly.
4. Add a one-line comment marking the intentional divergence between the replace path (mark-only detection) and the deletion path (DecorationSet fallback) so it isn't "fixed" later (Issue #4).
5. Optional nit: drop the redundant `typeof … === "number"` guard in `resolveBeforeInputRange` (always true), or leave as-is — no functional impact (Issue #2).
