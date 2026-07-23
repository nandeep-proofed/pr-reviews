# PR Review: fix/PP-2018: Fix blank WYSIWYG when AI pre-edit leaves an empty Y.JS doc

**PR:** https://github.com/Proofed/B2BWebserver/pull/2385
**Jira:** https://proofed.atlassian.net/browse/PP-2018
**Status:** Code Review

---

## TL;DR

The actual bug fix is **correct, well-reasoned, and well-tested**. The root-cause analysis (JSON PATCH `updateDocument` merges rather than replaces, so the follow-up content never landed in the persisted Y.JS binary) is confirmed by the tiptap API layer — `createDocument` POSTs (`?format=json`, sets), `updateDocument` PATCHes (`?format=json`, merges).

However, the PR is **based on a stale `develop`** (its base is `5ccc1b5c7`, before PP-1938 merged). PP-1938 (#2367) rewrote the exact same function **and** added a new caller (`overlayPostEditAiChanges.ts`) that depends on the two parameters this PR deletes (`onlyUpdate`, `jobDescription`). GitHub reports `mergeable_state: dirty`. This is a hard blocker: the PR must be rebased onto current `develop` and re-reviewed, because a naive merge would either fail to compile or silently break the AI post-edit overlay feature.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1. Fix Y.JS TrackedChange creation when the AI pre-edit made **no changes**, so the WYSIWYG opens without the manual Y.JS-deletion workaround | New docs: `createDocument` seeded with the real converted content directly (mirrors the always-working AI-changes path), removing the create-empty-then-`updateDocument`-merge sequence that produced a blank Y.JS binary | ✅ Addressed |
| 1a. Recover the already-broken orders (52827, 52830, …) without the manual workaround | Self-heal: `createOrderDocumentInTiptapFromBlocksFormatFromJobIfNotExist` now treats the `{doc:[{paragraph}]}` placeholder as "needs rebuild", deletes it, and recreates from the source work-item content version. Runs pre-connect on editor open; idempotent; guarded so healthy docs are never touched | ✅ Addressed (goes beyond "fix creation" — see scope note) |
| 2. Add logging around Y.JS / TipTap version creation covering **created / skipped / failed** and specifically the "pre-edit made no changes" path | `log.info` (created/saved), `log.warn` (empty content / skipped heal / empty persisted payload), `log.error` + `reportError` (failed) added to both `createOrderDocumentInTiptapFromBlocksFormat` and `saveYjsVersionAsWorkItemContent`; `log.flush()` in `finally` for serverless | ✅ Addressed |
| Validation: WYSIWYG does not open blank/error for no-change pre-edit orders | Verified live on order 21270 (per PR description); unit tests cover create-seed, placeholder heal, and healthy-doc guard | ✅ Addressed (manual + unit) |

**Scope beyond the ticket (all defensible):**
- The **self-heal delete + recreate** of already-broken documents is a backfill mechanism the ticket did not explicitly request (it asked to fix *creation*). It is a reasonable recovery path, but it is also the source of the medium-severity issue below (non-atomic delete/recreate). The PR's own follow-up note even proposes a separate backfill script for the same orders.
- **Sentry `reportError`** alerts (not just Logtail logs) — reasonable diagnostic addition consistent with the "not reproducible in TEST" constraint.

---

## Architecture Analysis

The change is confined to two API utilities in the creative portal WYSIWYG seeding pipeline:

- `createOrderDocumentInTiptapFromBlocksFormat.ts` — extracts a shared `buildTiptapJsonFromWorkItemContent` helper (decode → id-mapping → `convertDocument` to JSON), adds a structural `isEmptyTiptapDocContent` / `nodeHasMeaningfulContent` emptiness check, rewrites the "no AI changes" branch to seed `createDocument` with real content, and adds delete-and-rebuild self-heal logic to the `…FromJobIfNotExist` entry point.
- `saveYjsVersionAsWorkItemContent.ts` — wraps the existing version-save flow in try/catch/finally with logging + Sentry, and warns on an empty persisted payload. No behavioural change to what is persisted.

The `isEmptyTiptapDocContent` design is genuinely good: it walks the node tree structurally rather than substring-scanning serialized JSON, so image-only / rule-only / table docs are correctly classified as non-empty (and there are explicit tests for the image-only and `text-align`-attr cases). The "don't heal when the rebuild source is also empty" churn guard is a thoughtful touch that prevents an every-open delete/recreate loop.

The problem is **not the logic in isolation — it is that the same function was concurrently rewritten on `develop` by PP-1938**, and the two rewrites are semantically incompatible (see Issue 1).

---

## Issues Found

### 1. Stale base — deletes `onlyUpdate` / `jobDescription` that PP-1938's `overlayPostEditAiChanges` depends on

**[File: apps/creative-portal/api/utils/wysiwyg/createOrderDocumentInTiptapFromBlocksFormat.ts]**

**Function/Class:** createOrderDocumentInTiptapFromBlocksFormat (and its `CreateOrderDocumentInTiptapFromBlocksFormatProps` interface)

**Severity:** high

**Problem:** The PR branch is based on `develop@5ccc1b5c7`, which predates PP-1938 (#2367, `7491fa7fb`). I confirmed via `git merge-base` that PP-1938 is **not** an ancestor of the PR branch but **is** on `develop`. PP-1938 (a) refactored the decode call in this same function (`decodeBlockFormatContent` replaced `decodeHtmlEntitiesInBlockFormatFile(decodeWorkItemContent(...))`), (b) added a `jobDescription` prop threaded into `aiChangesWithPositions`, and (c) added a **new caller**, `overlayPostEditAiChanges.ts`, whose final statement is:

```typescript
await createOrderDocumentInTiptapFromBlocksFormat({
  orderId,
  workItemContent,
  jobDescription: postJobDescription,
  onlyUpdate: document !== null   // <-- update-in-place, do NOT delete/recreate
});
```

This PR **removes both `onlyUpdate` and `jobDescription`** from the props and **deletes the entire `onlyUpdate` update path** (including the `updateOrderDocumentWithAiChangesWithPositions` call). GitHub already reports `mergeable_state: "dirty"`.

**Impact:** After rebasing onto `develop`:
- **Compile break:** `overlayPostEditAiChanges.ts` passes excess properties (`jobDescription`, `onlyUpdate`) → TypeScript object-literal error → `typecheck`/`build` fail.
- **Semantic regression (worse than the compile error):** the `onlyUpdate` path is **not dead code on `develop`** — the post-edit overlay relies on it to *update an existing document in place*, preserving the reviewer's edits and the AI-change array. This PR's replacement strategy is delete-and-recreate. If the params were simply dropped to make it compile, the post-edit overlay would either recreate the document (blowing away reviewer edits / duplicating the AI-change array) or hit the wrong branch entirely. The PP-1938 idempotency guard in `overlayPostEditAiChanges` assumes update semantics.

**Fix:** Rebase the branch onto current `develop` and reconcile with PP-1938 rather than overwriting it. Specifically:

- Preserve `decodeBlockFormatContent` and the `jobDescription` threading from PP-1938 inside the new `buildTiptapJsonFromWorkItemContent` helper.
- **Keep the `onlyUpdate` update path** — the post-edit overlay needs it. The PP-2018 fix is orthogonal: it only needs to change the *no-AI-changes create* branch (empty-placeholder + `updateDocument` → seed `createDocument` directly) and add the self-heal to `…FromJobIfNotExist`. It does not require removing the update path used by post-edit overlays.
- Re-run `typecheck`/`build`/`test` after the rebase; the test files (which also conflict — the base test imports `decodeBlockFormatContent`) must be reconciled too.

---

### 2. Self-heal delete + recreate is non-atomic and surfaces as an endpoint failure

**[File: apps/creative-portal/api/utils/wysiwyg/createOrderDocumentInTiptapFromBlocksFormat.ts]**

**Function/Class:** createOrderDocumentInTiptapFromBlocksFormatFromJobIfNotExist

**Severity:** medium

**Problem:** The heal path does `await tiptapApi.deleteDocument(...)` followed by `await createOrderDocumentInTiptapFromBlocksFormat(...)` as two separate un-transactional network calls. Two failure modes:

1. If `deleteDocument` **succeeds** but the subsequent `createDocument` throws (it now rethrows after logging), the order is left with **no document at all** — strictly worse than the empty placeholder it started from (though it self-corrects on the next open via the `document === null` branch).
2. If `deleteDocument` itself throws (e.g. the doc was concurrently removed → 404, or a transient tiptap error), the error propagates. Both invoking endpoints — `patchJob.ts` (the "on assignment" flow) and `createJobCandidate.ts` (the "accept from queue" flow) — `await` this inside their `try` and route throws to `handleEndpointError`, so a heal-time delete failure now **fails the assignment / candidate creation** for an order that would previously have just opened blank. In `createJobCandidate`, `addJobCandidate` runs *after* the heal, so it would be skipped entirely.

The pre-PR heal only ever created/updated; introducing a `delete` adds a distinct failure class (notably 404-on-already-deleted) to these user-facing endpoints.

**Impact:** A transient tiptap hiccup during the heal can block editor assignment / queue-accept, and a mid-heal failure can transiently leave an order document-less. This is exactly the kind of revenue-impacting, hard-to-reproduce path the ticket is already about.

**Fix:** Make the heal defensive so a heal failure degrades gracefully instead of failing the endpoint:

```typescript
// Tolerate a doc that has already been deleted, and never let a heal
// failure fail the assignment/accept — it self-heals on the next open.
try {
  if (isEmptyExistingDoc) {
    await tiptapApi.deleteDocument(documentName);
  }
  await createOrderDocumentInTiptapFromBlocksFormat({ orderId: order.id, workItemContent });
} catch (error) {
  reportError(error, { operation: "wysiwyg.heal-empty-doc", extra: { orderId: order.id, jobId: job.id } });
  // swallow: next editor open retries the heal
}
```

At minimum, treat a 404 from `deleteDocument` as success (the doc being gone is the desired post-state).

---

### 3. `convertDocument` runs twice per heal (redundant work)

**[File: apps/creative-portal/api/utils/wysiwyg/createOrderDocumentInTiptapFromBlocksFormat.ts]**

**Function/Class:** createOrderDocumentInTiptapFromBlocksFormatFromJobIfNotExist / buildTiptapJsonFromWorkItemContent

**Severity:** low

**Problem:** On the heal path, `buildTiptapJsonFromWorkItemContent(workItemContentVersion.content, order.id)` is called to check `isEmptyTiptapDocContent(rebuiltContent)` before deleting — then `createOrderDocumentInTiptapFromBlocksFormat` immediately calls `buildTiptapJsonFromWorkItemContent` **again** internally on the same content, re-running decode, a fresh `generateAiChangeIdMappingForBlocks`, and `convertDocument`. The helper's own comment says it "converts once up front" but the result is discarded rather than reused. `convertDocument` is a blocks→JSON conversion (potentially non-trivial), and regenerating the id-mapping produces a second, throwaway set of UUIDs.

**Impact:** Doubles the conversion cost on every heal and does redundant id-mapping work. No correctness impact (the discarded mapping is unused), just wasted effort on a path that runs on editor open.

**Fix:** Either pass the already-built JSON (and blocks/mapping) into a create variant that accepts pre-converted content, or accept the minor duplication and drop the "converts once up front" wording from the comment so it doesn't imply reuse.

---

### 4. Empty-content Sentry alert fires for any genuinely-empty converted doc

**[File: apps/creative-portal/api/utils/wysiwyg/createOrderDocumentInTiptapFromBlocksFormat.ts]**

**Function/Class:** createOrderDocumentInTiptapFromBlocksFormat

**Severity:** low

**Problem:** `createOrderDocumentInTiptapFromBlocksFormat` calls `reportError(new Error("PP-2018: building Y.JS document from empty content"), …)` whenever `isEmptyTiptapDocContent` is true, on the assumption that "callers only reach here with content they believe is real." That assumption holds on the PR's own base (the heal guards against an empty source), but once `overlayPostEditAiChanges` is a caller again (post-rebase), and for any future caller, a legitimately-empty document would raise a Sentry error rather than a warn. Also note `nodeHasMeaningfulContent` treats whitespace-only text (`"   "` → `trim().length === 0`) as empty, so a doc that is intentionally just whitespace would be flagged.

**Impact:** Potential Sentry noise / false alerts if a legitimately empty or whitespace-only document is ever built. Minor.

**Fix:** Keep the `log.warn` unconditionally, but consider gating the `reportError` (Sentry) to the paths where empty truly indicates a regression, or accept the noise consciously. Not blocking — flagging for awareness during the rebase since the caller set changes.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⏭️ Skipped | Skipped — user opted out. (PR self-reports 1806 passing on its stale base.) |
| `npx turbo run typecheck` | ⏭️ Skipped | Skipped — user opted out. **Expected to FAIL after rebase** due to Issue 1 (excess props in `overlayPostEditAiChanges.ts`). |
| `npx turbo run lint` | ⏭️ Skipped | Skipped — user opted out. |
| `npx turbo run build` | ⏭️ Skipped | Skipped — user opted out. **Expected to FAIL after rebase** (same cause as typecheck). |

> Validation was intentionally not run. On the isolated (stale) PR branch it would very likely pass — a false negative, because the breakage only manifests after merging with PP-1938. Re-run all four **after** rebasing onto `develop`.

---

## Tests

- ✅ Strong new unit coverage on the fix itself: seeds `createDocument` with real content; does **not** create the empty placeholder; does **not** issue a separate `updateDocument`; creates the initial version; delegates to the AI-changes path when changes exist.
- ✅ Heal coverage: deletes placeholder + rebuilds from source; leaves a doc with real content alone; leaves an image-only (non-text) doc alone; still heals a childless paragraph carrying a `text-align` attr; skips + reports when the rebuild source is also empty (no churn).
- ✅ New `saveYjsVersionAsWorkItemContent.test.ts`: persists `yjs` (+ `yjsOriginal` for SERVICE), warns on empty payload, logs + `reportError` + rethrows on failure.
- ⚠️ No test for the **non-atomic delete failure** (Issue 2) — e.g. `deleteDocument` rejecting, or `createDocument` failing after a successful delete.
- ⚠️ The two edited test files also conflict with `develop` (base imports `decodeBlockFormatContent`); they must be reconciled during the rebase, and the mocks re-verified against PP-1938's shape.
- ⏭️ Full suite not executed (user opted out) — re-run after rebase.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ Correct on its own base, but semantically incompatible with current `develop` (removes an update path PP-1938 needs) |
| Regression risk | ❌ High — breaks the PP-1938 AI post-edit overlay after merge (compile + behavioural) |
| Tests | ✅ Good coverage of the fix; ⚠️ missing delete-failure case + test files conflict on rebase |
| Code quality | ✅ Clean, well-commented, thoughtful emptiness/churn guards |
| Validation suite | ⏭️ Skipped — user opted out |
| Mergeable state | ❌ Dirty (`mergeable_state: dirty`; PP-1938 not in branch, confirmed via `git merge-base`) |

---

## Recommendation

**Request changes.**

1. **Blocker — rebase onto current `develop` and reconcile with PP-1938** (Issue 1). Preserve `decodeBlockFormatContent` + `jobDescription` threading, and **keep the `onlyUpdate` update path** that `overlayPostEditAiChanges` depends on. Scope the PP-2018 fix to (a) the no-AI-changes *create* branch and (b) the `…FromJobIfNotExist` self-heal — it does not require removing the update path.
2. **Re-run the validation suite after the rebase** (`test` / `typecheck` / `lint` / `build`). Validation was skipped for this review and would be a false negative on the stale branch; typecheck/build are expected to fail pre-rebase.
3. **Harden the self-heal** (Issue 2): wrap delete + recreate so a heal failure degrades to the next-open retry instead of failing the assign/accept endpoint, and tolerate 404-on-delete.
4. **Nice-to-have:** avoid the double `convertDocument` on the heal path (Issue 3); reconsider the unconditional Sentry `reportError` on empty content (Issue 4); add a test for the delete-failure path.

The underlying fix is sound and the diagnostics satisfy the ticket — this is a "correct change on the wrong base," not a wrong change. Once rebased and re-validated, it should be close to approvable.
