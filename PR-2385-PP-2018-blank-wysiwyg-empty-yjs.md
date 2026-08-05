# PR Review: fix/PP-2018: Fix blank WYSIWYG when AI pre-edit leaves an empty Y.JS doc

**PR:** https://github.com/Proofed/B2BWebserver/pull/2385
**Jira:** https://proofed.atlassian.net/browse/PP-2018
**Status:** Code Review → **Suggestions resolved (2026-07-23)**

---

## Resolution Update (2026-07-23)

All actionable review points were re-verified as valid against the current PR-branch code, then resolved. Awareness-only notes are acknowledged with no change. Latest commit: `614dd8b75`.

| # | Issue | Valid? | Action taken |
|---|---|---|---|
| 1 | Widened error-swallow now covers first-time create, not just heal | ✅ Valid (real behavior change) | Kept the more-resilient swallow (intended — candidate creation shouldn't be coupled to Tiptap availability); **clarified the comment** to state it covers both the first-time create and the heal, each best-effort with next-open retry + Sentry alert. |
| 2 | Double Sentry event when heal-recreate throws | ✅ Valid | Added a `reportOnError` flag to `seedOrderDocumentFromBuiltJson`; the self-heal calls it with `reportOnError: false`, so one failure now raises a **single** `wysiwyg.heal-empty-doc` event (no duplicate `wysiwyg.create-order-document`). |
| 3 | Empty-content alert also fires on `overlayPostEditAiChanges` create path | ✅ Valid (awareness) | No change — create-path empty content is accepted as always alert-worthy per the ticket's diagnostic intent. |
| 4 | Test gap: heal where rebuild source carries AI changes | ✅ Valid (coverage gap) | Added a test asserting the heal deletes then recreates via `createOrderDocumentWithAiChangesWithPositions` (create-safe path), with an invocation-order check (delete before recreate). |
| 5 | Multi-empty-paragraph docs now within heal scope | ✅ Valid (awareness) | No change — matches stated intent; conservative classifier + source-non-empty churn guard make accidental deletion of real content effectively impossible. |
| 6 | Misleading "document created" log on the update path | ✅ Valid | Branched the success log: `onlyUpdate ? "[TIPTAP] Y.JS document updated" : "[TIPTAP] Y.JS document created"`. |
| 7 | Pre-existing: `checkIfOrderDocumentExistsInTiptap` reads `error.response.status` without optional chaining | ✅ Valid but out of scope | **Skipped** — pre-existing and unrelated to this PR (PR-scope discipline). Flagged for a separate follow-up. |

### Post-fix validation (creative-portal)

| Check | Result |
|---|---|
| `turbo run typecheck` | ✅ 0 errors |
| `turbo run lint` (`--max-warnings 0`) | ✅ 0 warnings |
| `turbo run test` | ✅ 2079 passed / 232 files |

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Fix Y.JS TrackedChange creation when AI pre-edit made no changes, so the WYSIWYG opens without the manual Y.JS-deletion workaround | No-change create path now seeds `createDocument` with the real content directly (no empty placeholder + separate merge-`updateDocument`), which is the root cause of the empty Y.JS binary | ✅ Addressed |
| Handle already-affected orders (52827, 52830, …) so they recover without manual reopen | Self-heal: `isEmptyTiptapDocContent` treats the `{ type: "doc", content: [{ type: "paragraph" }] }` placeholder as needing rebuild; deletes the stale doc and recreates from the source work item content version, idempotently on editor open | ✅ Addressed (beyond strict scope; backfill script explicitly deferred as a follow-up) |
| Add logging on Y.JS file creation covering created / skipped / failed and the "no changes" path | `log.info` (created/updated), `log.warn` + Sentry `reportError` (skipped-empty / empty-content), `log.error` + `reportError` (failed) in both `createOrderDocumentInTiptapFromBlocksFormat` and `saveYjsVersionAsWorkItemContent` | ✅ Addressed |
| Diagnostic-only because not reproducible in TEST | Sentry `reportError` added on all failure/empty paths so production occurrences surface | ✅ Addressed |

Scope note: the self-heal (delete + recreate of existing broken docs) goes slightly beyond the literal Jira ask (which is worded around *creation*), but it directly serves the ticket's listed affected orders and is a reasonable, well-guarded addition.

---

## Architecture Analysis

The root cause is precise and the fix targets it directly. The old no-AI-changes branch did `createDocument(emptyPlaceholder)` followed by a separate JSON `updateDocument(realContent)`; Tiptap's JSON PATCH default mode *merges* rather than *replaces*, so the real content never landed in the persisted Y.JS binary and the saved `yjs` work item content version held only the empty paragraph. Passing content straight to `createDocument` (which *sets* content) mirrors the AI-changes path, whose output was always correct.

The refactor is clean and improves reuse:
- `buildTiptapJsonFromWorkItemContent` — decode + id-mapping + convert (shared).
- `seedOrderDocumentFromBuiltJson` — the create/update dispatch, so the self-heal reuses the already-converted JSON instead of decoding + converting twice.
- `isEmptyTiptapDocContent` / `nodeHasMeaningfulContent` — a **structural** emptiness check (walks the node tree) rather than a serialized-JSON substring scan, so images / rules / tables are never misclassified as empty and deleted. The classifier is deliberately conservative (errs toward "has content"), which is the right bias for a destructive delete.

The heal is correctly guarded: it only deletes when the live doc is empty **and** the rebuild source converts to non-empty content (churn guard), tolerates a 404 on delete (already-gone is the desired post-state), and is wrapped so a transient Tiptap failure degrades to a next-open retry instead of failing the caller's assignment/candidate-creation endpoint.

Verified against the surrounding code:
- `getWysiwygOrderDocumentName` ignores `jobId` (uses `orders/{orderId}` only), so the heal's `deleteDocument` (called with `jobId`) and `seedOrderDocumentFromBuiltJson`'s recreate (called without `jobId`) target the **same** document — no name mismatch.
- `createOrderDocumentWithAiChangesWithPositions` → `createTiptapDocumentWithAiChangesWithPositions` internally calls `tiptapApi.createDocument`, so if the rebuild source carries AI changes, recreation after a delete is create-safe (not a broken post-delete `updateDocument`).
- `tiptapApi.deleteDocument` exists and is Axios-based, so `deleteError?.response?.status !== 404` is a valid discriminant.
- `createDocumentVersion("Initial version")` is still emitted once on every create/update path (behavior preserved).

Overall this is a solid, root-cause fix with strong diagnostics. The issues below were low-severity design/robustness observations, not correctness defects — all now resolved or acknowledged (see Resolution Update).

---

## Issues Found

### 1. Error-swallowing now also covers first-time document creation (`document === null`), not just the heal — ✅ RESOLVED (comment clarified)

**[File: apps/creative-portal/api/utils/wysiwyg/createOrderDocumentInTiptapFromBlocksFormat.ts]** — Severity: low

The `try { … } catch` that swallows failures wraps `seedOrderDocumentFromBuiltJson` for both the heal case (`isEmptyExistingDoc`) and the fresh-create case (`document === null`). A transient Tiptap failure during first creation now returns success to the assignment/candidate endpoint while leaving the order document-less (self-heals next open with a Sentry alert). This is the intended more-resilient behavior; the comment now explicitly states the swallow covers both paths.

### 2. Double Sentry event when the heal's recreate throws — ✅ RESOLVED (single event via `reportOnError`)

**[File: …createOrderDocumentInTiptapFromBlocksFormat.ts]** — Severity: low

`seedOrderDocumentFromBuiltJson`'s catch reported `wysiwyg.create-order-document` and rethrew; the heal's outer catch then reported `wysiwyg.heal-empty-doc` — two events per failure. Fixed by passing `reportOnError: false` from the heal so the outer catch owns the single event. Regression test added.

### 3. Empty-content Sentry alert fires on the `overlayPostEditAiChanges` first-time-create path too — ✅ ACKNOWLEDGED (no change)

**[File: …createOrderDocumentInTiptapFromBlocksFormat.ts]** — Severity: low

Create-path empty content is accepted as always alert-worthy. If overlay-create can be legitimately empty, a future flag can suppress it (analogous to `onlyUpdate`).

### 4. Test-coverage gap: heal where the rebuild source carries AI changes — ✅ RESOLVED (test added)

**[File: …createOrderDocumentInTiptapFromBlocksFormat.test.ts]** — Severity: low

Added a heal test with `extractAiChangesFromBlocks` returning a non-empty array; asserts `createOrderDocumentWithAiChangesWithPositions` is invoked after `deleteDocument` (invocation-order check) and the plain `createDocument` branch is not taken.

### 5. Behavior note: multi-empty-paragraph docs are now within heal scope — ✅ ACKNOWLEDGED (no change)

**[File: …createOrderDocumentInTiptapFromBlocksFormat.ts]** — Severity: low

Matches stated intent; the source-non-empty churn guard + idempotency make this safe.

### 6. Misleading "document created" log on the update path — ✅ RESOLVED (branched message)

**[File: …createOrderDocumentInTiptapFromBlocksFormat.ts]** — Severity: low

Success log now reads `"[TIPTAP] Y.JS document updated"` on the `onlyUpdate` path and `"[TIPTAP] Y.JS document created"` on create.

### 7. Pre-existing: `checkIfOrderDocumentExistsInTiptap` reads `error.response.status` without optional chaining — ✅ VALID, SKIPPED (out of scope)

**[File: …createOrderDocumentInTiptapFromBlocksFormat.ts]** — Severity: low

Real, but pre-existing and unrelated to this PR; left for a separate follow-up per PR-scope discipline. The new delete path already uses the safer `?.response?.status`.

---

## Tests

- ✅ New-code tests added for both changed source files — meets the "every PR must include tests" requirement.
- ✅ Heal path well covered: deletes placeholder + rebuilds; leaves real-content / image-only docs untouched; still heals a childless paragraph with a `text-align` attr; no-churn when source is also empty; best-effort on non-404 delete error; tolerates 404; doesn't fail caller when recreate throws; **recreates via the AI-changes path when the source carries AI changes (added)**; **single Sentry event on recreate failure (added)**.
- ✅ Create path covered: seeds real content (not placeholder), no separate `updateDocument`, creates initial version, delegates to AI path when changes exist.
- ✅ Empty-content covered: warns + Sentry on create; no Sentry on `onlyUpdate`.
- ✅ `saveYjsVersionAsWorkItemContent` covered: yjs + yjsOriginal (SERVICE), warn on empty payload, error-log + rethrow + `reportError` on failure.
- ⚠️ Not automatable: production-only reproduction — relies on manual verification on order 21270 noted in the PR.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Root-cause fix; no correctness defects found |
| Regression risk | ✅ Low — widened error-swallowing scope confirmed intentional (Issue 1); heal scope acknowledged (Issue 5); both guarded |
| Tests | ✅ Strong new-code coverage; prior gap (Issue 4) closed |
| Code quality | ✅ Clean refactor, good reuse, conservative destructive-op guard, thorough comments |
| Validation suite | ✅ typecheck 0 / lint 0 / 2079 tests passed (post-fix) |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve** — all actionable review points resolved (Issues 1, 2, 4, 6), awareness notes acknowledged (3, 5), and the pre-existing nit (7) deferred as out-of-scope. Mandatory validation suite re-run and green. Ready to merge.
