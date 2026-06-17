# PR Review: fix/PP-1822: WYSIWYG — replacing selected text reintroduces deleted text in Clean version

**PR:** https://github.com/Proofed/B2BWebserver/pull/2339
**Jira:** https://proofed.atlassian.net/browse/PP-1822
**Status:** In Progress (Bug, Priority: Medium)
**Commits:** `13a01a4aa` (fix) · `509a080a1` (review follow-up — unify classifier, drop duplication)

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Replacing selected text must NOT reintroduce previously deleted text in the Clean version | `buildTrackedReplacement` re-classifies the overwritten selection: own pending insertion dropped, prior deletion preserved, original text marked as the new deletion. Deleted text is never re-inserted into the clean output. | ✅ Addressed |
| Trigger is text **replacement** (overwrite), not formatting (distinct from PP-1659) | Typing-over-selection is intercepted at `keydown` (+ `beforeinput` backup) and routed through a single clean tracked replacement, bypassing the unreliable `readDOMChange` path. | ✅ Addressed |
| Specifically occurs when selecting a word **linked to an edit** (Orlin's repro: `~~You want to l~~Look`) | Covered directly by regression tests (Scenario 4b/4c use the exact `You want to l` / `Look` case). | ✅ Addressed |
| Selected text is replaced with the newly typed text (and only that) | Inserted text is built from the exact typed character; surrounding original/struck text is never absorbed into the insertion. | ✅ Addressed |
| (Implicit) inserted text should keep correct formatting, not stray bold | `getOriginalFormattingMarks` makes the insertion inherit the **original** replaced text's formatting, not the discarded insertion's. | ✅ Addressed (beyond literal Jira text) |

**Scope note:** The PR is tightly scoped to the bug — only `packages/wysiwyg/src/extensions/trackChanges-v2/plugins/tracking.ts` and its test file. No scope creep / unrelated refactors.

---

## Architecture Analysis

The fix addresses the **root cause**, not the symptom. Typing over a selection in Track Changes previously relied on ProseMirror's `readDOMChange` reconciliation, which — with struck-through deleted text present in the DOM — produced a malformed `ReplaceStep` (absorbed surrounding text, mis-attributed marks). The PR introduces a deterministic interception layer:

- **`keydown`** (primary) — earliest point, before any DOM mutation, with the full selection intact; `preventDefault()` stops the browser's split delete+insert sequence.
- **`beforeinput`** (backup) — for non-keystroke input (autocorrect's `insertReplacementText`), using `getTargetRanges()` to read the exact replaced range even before selection sync.
- **`buildTrackedReplacement`** (shared core) — segments the overwritten range (drop own insertion / keep prior deletion / mark original as new deletion), then inserts exactly the typed text with the original formatting. New deletion + insertion share one change-id → single "Replace" card; prior deletions keep their own ids → stay separate "Delete" cards.
- **`handleReplacement`** (appendTransaction fallback) — rewritten to the same segmentation so any path that still reaches it (paste/IME/programmatic) produces correct output.
- **`classifyOverwrittenNode`** (follow-up commit `509a080a1`) — single source of truth for the "own insertion → drop / existing deletion → keep / original → mark" decision, shared by all three of the above. Detection is **mark-only** everywhere; the local-decoration fallback is retained only in the keyboard deletion handler (PP-1774 Path B).

This is the right altitude: one shared mechanism feeds every input path, rather than patching individual symptoms.

---

## Issues Found

### 1. `handleReplacement` classifier diverged from the other two (own-insertion detection) — ✅ RESOLVED in `509a080a1`

**[File: packages/wysiwyg/src/extensions/trackChanges-v2/plugins/tracking.ts]**

**Function/Class:** handleReplacement

**Severity:** medium (resolved)

**Problem (original):** `buildTrackedReplacement` and `getOriginalFormattingMarks` detected "own insertion" from the track-change **mark only**, but `handleReplacement` still ORed in `hasLocalInsertionDecoration(...)`. On the `readDOMChange` fallback path, a stale insertion decoration overlapping **original** text would classify that text as own-insertion and drop it (the same data-loss class this PR fixes on the primary path).

**Resolution:** Follow-up commit `509a080a1` makes `handleReplacement` use the shared `classifyOverwrittenNode` helper (mark-only). The `readDOMChange` fallback can no longer mis-classify original text via a stale decoration. The DecorationSet fallback remains intact in the keyboard deletion handler, so PP-1774 (collab mark-stripping) protection is unchanged there. Accepted trade-off: in the rare case where Collaboration strips an own-insertion's mark *and* that insertion is overwritten via the fallback path, it would be marked as a struck-through deletion instead of removed — a cosmetic mis-categorization, no data loss.

### 2. Classification logic duplicated three times — ✅ RESOLVED in `509a080a1`

**[File: packages/wysiwyg/src/extensions/trackChanges-v2/plugins/tracking.ts]**

**Function/Class:** getOriginalFormattingMarks / handleReplacement / buildTrackedReplacement

**Severity:** low (resolved)

**Problem (original):** The "is this node own-insertion / existing-deletion / original" classification was repeated in three functions.

**Resolution:** Extracted into a single `classifyOverwrittenNode(node, userId)` helper used by all three. Net −7 lines; the three paths can no longer drift apart.

### 3. Mixed-format selection inherits only the first segment's formatting

**[File: packages/wysiwyg/src/extensions/trackChanges-v2/plugins/tracking.ts]**

**Function/Class:** getOriginalFormattingMarks

**Severity:** low

**Problem:** Returns the first original text node's marks for the whole replacement. Replacing bold "He" + plain "llo" with one char makes the inserted char bold.

**Impact:** Minor formatting nuance on mixed-format selections; defensible (matches "take the start's formatting" behavior). Not a crash or data loss.

**Fix:** Left as-is by design (documented choice).

---

## Validation Checks

> Run in-place on the PR branch. **Every failure below is pre-existing on `develop` and lives in files/packages this PR does not touch.** The PR's own files and package are clean.

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ❌ (pre-existing, unrelated) | `@proofed/customer-portal` → `api/orders/getOrdersForOrganizationMember.test.ts` fails with HTTP 500 (network/env). `@proofed/wysiwyg-editor` tests **pass** (incl. the PP-1822 regression tests). |
| `npx turbo run typecheck` | ❌ (pre-existing, unrelated) | Only `tsconfig.json` deprecation errors in wysiwyg (`TS5101 baseUrl`, `TS5107 moduleResolution=node10`). **No real type errors** in code (`tsc --noEmit --ignoreDeprecations 6.0` on the changed file → clean). |
| `npx turbo run lint` | ❌ (pre-existing, unrelated) | 63 prettier errors in **other** wysiwyg files (`AiChangeBox/index.tsx`, `CommentsContainer/utils.ts` & test, `EditorContext/hooks.ts`, `comments/index.ts`). The changed file (`tracking.ts`) and its test lint **clean**. |
| `npx turbo run build` | ❌ (pre-existing, unrelated) | `@proofed/customer-portal` → `Cannot find module 'iron-session/next'`. `@proofed/wysiwyg-editor` **builds clean** (`created lib/index.js, lib/index.esm.js`). |

**PR-scoped checks (the files/package this PR changes):** ✅ wysiwyg tests pass (incl. PP-1822 regression), ✅ changed files lint-clean, ✅ no real type errors, ✅ wysiwyg builds clean. Re-verified after the `509a080a1` follow-up: **53 track-changes tests pass.**

---

## Tests

- ✅ PR adds tests for the new code (project requirement met) — `tracking.test.ts`.
- ✅ Scenario 4b — replace over prior deletion + own insertion (drops own insertion, preserves prior deletion, marks original as new deletion).
- ✅ Scenario 4c — `buildTrackedReplacement`: exact typed text (no surrounding absorption); collapsed selection ignored; bold NOT inherited from a discarded insertion; bold preserved from genuine original; prior deletion kept; **stale-decoration regression (the PP-1822 root cause)**.
- ✅ Scenario 10 — PP-1774 local-insertion-decoration infrastructure still intact after the classifier refactor.
- ✅ wysiwyg package suite passes; track-changes suite: 53 passing / 4 skipped (pre-existing Yjs/browser-DOM integration tests).
- ⚠️ Full-monorepo `turbo test` reports red, but only due to the unrelated pre-existing `customer-portal` 500 test (see Validation Checks).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Fixes the root cause; matches Jira expected result |
| Regression risk | ✅ Low — isolated to one plugin; `keydown`/`beforeinput` guards tightly scoped; refactor is behaviour-preserving for marked cases; 53 track-changes tests pass |
| Tests | ✅ Strong — regression tests cover the exact Jira repro + the stale-decoration root cause |
| Code quality | ✅ Duplication and classifier asymmetry resolved in `509a080a1` (shared `classifyOverwrittenNode`) |
| Validation suite | ❌ Failures present — **all pre-existing & unrelated** (customer-portal test/build, repo-wide tsconfig/lint); PR-scoped checks all green |
| Mergeable state | ⚠️ GitHub clean; turbo suite red only on pre-existing develop issues |

---

## Recommendation

**Approve.**

1. **The PP-1822 fix is correct, well-tested, and root-cause.** It satisfies every Jira requirement and the reviewer's reproduction (`~~You want to l~~Look`).
2. **Code-review follow-ups #1 and #2 are now resolved** in `509a080a1` — all three overwrite paths share one mark-only `classifyOverwrittenNode`, removing the duplication and the `handleReplacement` decoration asymmetry. Issue #3 is a documented, defensible formatting choice. The refactor was verified behaviour-preserving for all marked (tested) cases.
3. **The red `turbo` validation is NOT caused by this PR** — all four failures are pre-existing on `develop` and confined to files/packages this PR doesn't touch (customer-portal test 500, customer-portal `iron-session/next` build, repo `tsconfig` deprecation, lint in other wysiwyg files). The PR's own files/package pass test, lint, typecheck, and build. Recommend tracking those separately, not against PP-1822.
