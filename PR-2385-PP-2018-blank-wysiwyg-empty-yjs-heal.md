# PR Review: fix/PP-2018: Fix blank WYSIWYG when AI pre-edit leaves an empty Y.JS doc

**PR:** https://github.com/Proofed/B2BWebserver/pull/2385
**Jira:** https://proofed.atlassian.net/browse/PP-2018
**Status:** Code Review

---

## ✅ Resolution update (2026-07-23)

**All four issues are resolved.** The branch was merged with current `develop` and the follow-up findings were addressed.

| # | Severity | Issue | Status | Fixed in |
|---|---|---|---|---|
| 1 | high | Stale base removes `onlyUpdate`/`jobDescription` PP-1938 needs | ✅ Resolved | merge `1516cb6d4` |
| 2 | medium | Non-atomic delete+recreate fails the assign/accept endpoint | ✅ Resolved | `41a6247b2` |
| 3 | low | `convertDocument` runs twice per heal | ✅ Resolved | `41a6247b2` |
| 4 | low | Unconditional Sentry alert on empty content | ✅ Resolved | `41a6247b2` |

**Validation after the fixes:** `typecheck` clean · `eslint` clean · full creative-portal suite **2078 passed** (+4 new tests covering the delete-failure / 404 / recreate-failure paths and the update-path Sentry gating). PR now reports `mergeable: MERGEABLE` / `mergeStateStatus: CLEAN`.

Per-issue detail is inline in the **Issues Found** section below.

---

## TL;DR

The actual bug fix is **correct, well-reasoned, and well-tested**. The root-cause analysis (JSON PATCH `updateDocument` merges rather than replaces, so the follow-up content never landed in the persisted Y.JS binary) is confirmed by the tiptap API layer — `createDocument` POSTs (`?format=json`, sets), `updateDocument` PATCHes (`?format=json`, merges).

However, the PR was **based on a stale `develop`** (its base is `5ccc1b5c7`, before PP-1938 merged). PP-1938 (#2367) rewrote the exact same function **and** added a new caller (`overlayPostEditAiChanges.ts`) that depends on the two parameters this PR deletes (`onlyUpdate`, `jobDescription`). GitHub reported `mergeable_state: dirty`, which was a hard blocker.

> **Update (2026-07-23):** This has since been resolved — `develop` was merged in and reconciled with PP-1938 (the `onlyUpdate` update path is kept), and the three follow-up findings below were fixed. See the **Resolution update** section above. GitHub now reports `MERGEABLE` / `CLEAN`.

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

### 1. Stale base — deletes `onlyUpdate` / `jobDescription` that PP-1938's `overlayPostEditAiChanges` depends on — ✅ RESOLVED (merge `1516cb6d4`)

> **Resolution:** `develop` was merged into the branch and reconciled with PP-1938 exactly as prescribed below. `decodeBlockFormatContent` and the `jobDescription` threading are preserved; the `onlyUpdate` update path (incl. `updateOrderDocumentWithAiChangesWithPositions`) is restored. `typecheck`/`build`/`test` pass and the predicted test-file conflicts did **not** materialize — `createOrderDocumentInTiptapFromBlocksFormat.test.ts` auto-merged cleanly and `overlayPostEditAiChanges.test.ts` (8 tests) passes. GitHub now reports `mergeable: MERGEABLE`.

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

### 2. Self-heal delete + recreate is non-atomic and surfaces as an endpoint failure — ✅ RESOLVED (`41a6247b2`)

> **Resolution:** The heal's `deleteDocument` + recreate is now wrapped in a try/catch that reports to Sentry (`operation: "wysiwyg.heal-empty-doc"`) and **swallows**, so a transient tiptap failure degrades to the next-open retry instead of failing job assignment (`patchJob`) or candidate creation (`createJobCandidate`). A **404 from `deleteDocument`** is treated as success (doc already gone = desired post-state) so the recreate still runs. New tests cover the non-404 delete error, the 404-tolerance path, and the recreate-throws path.

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

### 3. `convertDocument` runs twice per heal (redundant work) — ✅ RESOLVED (`41a6247b2`)

> **Resolution:** Extracted `seedOrderDocumentFromBuiltJson(...)`, which accepts the already-built `{ workItemContentBlocks, idMapping, json }` payload. The heal converts **once** for its emptiness pre-check and passes that same payload straight to the seed step — no second decode / id-mapping / `convertDocument`. The public `createOrderDocumentInTiptapFromBlocksFormat` is now a thin build-then-seed wrapper, so its behaviour is unchanged.

**[File: apps/creative-portal/api/utils/wysiwyg/createOrderDocumentInTiptapFromBlocksFormat.ts]**

**Function/Class:** createOrderDocumentInTiptapFromBlocksFormatFromJobIfNotExist / buildTiptapJsonFromWorkItemContent

**Severity:** low

**Problem:** On the heal path, `buildTiptapJsonFromWorkItemContent(workItemContentVersion.content, order.id)` is called to check `isEmptyTiptapDocContent(rebuiltContent)` before deleting — then `createOrderDocumentInTiptapFromBlocksFormat` immediately calls `buildTiptapJsonFromWorkItemContent` **again** internally on the same content, re-running decode, a fresh `generateAiChangeIdMappingForBlocks`, and `convertDocument`. The helper's own comment says it "converts once up front" but the result is discarded rather than reused. `convertDocument` is a blocks→JSON conversion (potentially non-trivial), and regenerating the id-mapping produces a second, throwaway set of UUIDs.

**Impact:** Doubles the conversion cost on every heal and does redundant id-mapping work. No correctness impact (the discarded mapping is unused), just wasted effort on a path that runs on editor open.

**Fix:** Either pass the already-built JSON (and blocks/mapping) into a create variant that accepts pre-converted content, or accept the minor duplication and drop the "converts once up front" wording from the comment so it doesn't imply reuse.

---

### 4. Empty-content Sentry alert fires for any genuinely-empty converted doc — ✅ RESOLVED (`41a6247b2`)

> **Resolution:** The empty-content `reportError` (Sentry) now fires on the **create path only** (`!onlyUpdate`). The post-edit overlay update path (PP-1938) can legitimately be handed an empty payload, so it logs a `log.warn` without paging. A test asserts no `wysiwyg.empty-yjs-content` alert is raised when `onlyUpdate` is true. The whitespace-only classification in `nodeHasMeaningfulContent` is left as-is (accepted consciously — a whitespace-only doc is effectively empty).

**[File: apps/creative-portal/api/utils/wysiwyg/createOrderDocumentInTiptapFromBlocksFormat.ts]**

**Function/Class:** createOrderDocumentInTiptapFromBlocksFormat

**Severity:** low

**Problem:** `createOrderDocumentInTiptapFromBlocksFormat` calls `reportError(new Error("PP-2018: building Y.JS document from empty content"), …)` whenever `isEmptyTiptapDocContent` is true, on the assumption that "callers only reach here with content they believe is real." That assumption holds on the PR's own base (the heal guards against an empty source), but once `overlayPostEditAiChanges` is a caller again (post-rebase), and for any future caller, a legitimately-empty document would raise a Sentry error rather than a warn. Also note `nodeHasMeaningfulContent` treats whitespace-only text (`"   "` → `trim().length === 0`) as empty, so a doc that is intentionally just whitespace would be flagged.

**Impact:** Potential Sentry noise / false alerts if a legitimately empty or whitespace-only document is ever built. Minor.

**Fix:** Keep the `log.warn` unconditionally, but consider gating the `reportError` (Sentry) to the paths where empty truly indicates a regression, or accept the noise consciously. Not blocking — flagging for awareness during the rebase since the caller set changes.

---

## Validation Checks

**Original review (stale base):**

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⏭️ Skipped | Skipped — user opted out. (PR self-reports 1806 passing on its stale base.) |
| `npx turbo run typecheck` | ⏭️ Skipped | Skipped — user opted out. **Expected to FAIL after rebase** due to Issue 1 (excess props in `overlayPostEditAiChanges.ts`). |
| `npx turbo run lint` | ⏭️ Skipped | Skipped — user opted out. |
| `npx turbo run build` | ⏭️ Skipped | Skipped — user opted out. **Expected to FAIL after rebase** (same cause as typecheck). |

**After merge + fixes (2026-07-23):**

| Check | Result | Notes |
|---|---|---|
| creative-portal `test` | ✅ Pass | **2078 passed** (232 files), incl. the 22 `createOrder...` tests + 8 `overlayPostEditAiChanges` tests |
| creative-portal `typecheck` | ✅ Pass | `tsc --noEmit` clean — the predicted `TS2353` is gone (Issue 1 resolved) |
| `eslint` (changed files) | ✅ Pass | clean after prettier auto-fix |

> The Issue 1 prediction (typecheck/build fail after merge; test files conflict) was correct as a *risk* but did not occur in practice, because the merge reconciled with PP-1938 rather than overwriting it — the update path was kept, so no excess-props error, and the test files auto-merged.

---

## Tests

- ✅ Strong new unit coverage on the fix itself: seeds `createDocument` with real content; does **not** create the empty placeholder; does **not** issue a separate `updateDocument`; creates the initial version; delegates to the AI-changes path when changes exist.
- ✅ Heal coverage: deletes placeholder + rebuilds from source; leaves a doc with real content alone; leaves an image-only (non-text) doc alone; still heals a childless paragraph carrying a `text-align` attr; skips + reports when the rebuild source is also empty (no churn).
- ✅ New `saveYjsVersionAsWorkItemContent.test.ts`: persists `yjs` (+ `yjsOriginal` for SERVICE), warns on empty payload, logs + `reportError` + rethrows on failure.
- ✅ **Delete-failure coverage added** (Issue 2): heal does not throw on a non-404 `deleteDocument` error (degrades to next-open retry); tolerates a 404 and still recreates; does not throw when the recreate itself fails.
- ✅ **Update-path Sentry gating covered** (Issue 4): no `wysiwyg.empty-yjs-content` alert when `onlyUpdate` is true.
- ✅ The edited test files did **not** end up conflicting on merge — they auto-merged cleanly and the mocks were verified against PP-1938's shape (`overlayPostEditAiChanges.test.ts` green).
- ✅ Full creative-portal suite executed after the fixes: **2078 passed**.

---

## Summary

| Aspect | Status (original → now) |
|---|---|
| Correctness | ⚠️ → ✅ Merged with `develop`; update path PP-1938 needs is preserved |
| Regression risk | ❌ High → ✅ Resolved — post-edit overlay compiles and its 8 tests pass |
| Tests | ✅ Good coverage → ✅ + delete-failure/404/recreate-failure + update-path gating cases (2078 pass) |
| Code quality | ✅ Clean, well-commented, thoughtful emptiness/churn guards |
| Validation suite | ⏭️ Skipped → ✅ typecheck + lint + full creative-portal suite pass |
| Mergeable state | ❌ Dirty → ✅ `MERGEABLE` / `CLEAN` |

---

## Recommendation

**Original: Request changes.** → **Now: all points addressed** (merge `1516cb6d4` + `41a6247b2`).

1. ✅ **Blocker resolved** — merged onto current `develop` and reconciled with PP-1938 (Issue 1): `decodeBlockFormatContent` + `jobDescription` threading preserved, `onlyUpdate` update path kept.
2. ✅ **Validation re-run** — `typecheck` / `lint` / full creative-portal `test` (2078) all pass.
3. ✅ **Self-heal hardened** (Issue 2) — delete + recreate degrades to next-open retry instead of failing the assign/accept endpoint, and tolerates 404-on-delete.
4. ✅ **Nice-to-haves done** — double `convertDocument` removed (Issue 3); Sentry `reportError` gated to the create path (Issue 4); delete-failure tests added.

The underlying fix was sound; it was a "correct change on the wrong base." Now rebased-via-merge, re-validated, and the follow-up findings addressed — **ready for re-review / approval.**

Remaining reviewer judgment call (non-blocking): on a recreate failure the heal reports to Sentry twice — inner `wysiwyg.create-order-document` (what failed) + outer `wysiwyg.heal-empty-doc` (heal context). Kept deliberately for triage context; trivial to collapse to one if the team prefers.
