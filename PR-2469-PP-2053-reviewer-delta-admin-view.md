# PR Review: feature/PP-2053: Show admins the reviewer delta copy

**PR:** https://github.com/Proofed/B2BWebserver/pull/2469
**Jira:** https://proofed.atlassian.net/browse/PP-2053
**Status:** Code Review
**Reviewed at:** `fed4c697c` (4 commits, 62 files, +3202 / −1015). Stacked on #2462 (base `feature/PP-2052-review-job-delta-copy` @ `a262c1bac`).

---

## What this means for users (non-technical summary)

1. **Some reviewer edits never make it into the delta.** When the same person worked both the editing job and the review job (e.g. an admin covering both), any review edits they make right next to their own earlier changes are treated as editing-stage work and dropped. If those were the only edits, no delta is saved at all. The delta is saved once at submit and can't be rebuilt, so the loss is permanent.
2. **Text typed at the end of an unresolved AI deletion disappears from the delta.** If the reviewer types right where a pending Proofed AI deletion ends, that typed text is left out of the delta, and again no delta may be saved.
3. **The PR description describes a different feature from the code.** It says wysiwyg deltas are worked out when the view opens and "work on orders that submitted long before the feature existed". The code saves them at submit and can't backfill, so orders completed before release will show no delta. QA and the PO will test against the wrong expectations.
4. **A review whose only action was rejecting the editor's changes gets no delta**, even though the reviewed copy differs from the edited one. This may be acceptable, but it needs a product decision.
5. **Admins get no signal outside the editor that a wysiwyg order has a delta.** For HTML orders the only sign is a new entry inside the editor's version dropdown. Whether the designs call for more is unconfirmed; see Open Questions.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
| --- | --- | --- |
| 1.1 Show the review delta copy in the Admin Area from the order/job history | Order panel only: a Files accordion row (file-based orders) and an editor dropdown entry (HTML orders). No job-history UI was touched. | ⚠️ Partial |
| 1.2 Admins can clearly see feedback was provided and open the delta | File-based: a visible "Reviewer Delta Copy" row. HTML: only a dropdown entry inside the editor. | ⚠️ Partial (see Open Questions) |
| 2.1 On-platform orders: download from the order side panel | The `FilesSection` "Reviewer Delta Copy" row downloads the Huxley DOCX. Customers never see it: the customer portal doesn't pass the prop, and the version is stored with `diffVersion: false`. | ✅ Addressed |
| 2.2 GDRIVE orders: DOCX download (depends on PP-2028) | Not deliverable: AI feedback is blocked for Drive formats, so no delta exists. Acknowledged in the PR. | ❌ Missing (acknowledged; needs its own ticket) |
| 2.3 HTML orders: WYSIWYG view only | Delta computed from the service and review Y.js snapshots at submit, stored as a `ReviewJobDelta` ProseMirror document, and shown in a separate read-only editor via a "Reviewer Delta Copy" dropdown entry. | ⚠️ Partial (Issues 1, 2) |

**Beyond Jira scope:**
- **Rewrites #2462's wysiwyg path:** removes the Huxley HTML parser and replaces it with a Y.js-derived delta computed at submit.
- **Shared `getWorkItemContentVersion` route:** now falls back to HTML conversion (Issue 5).
- **Unused PP-2051 props:** `versionMenu` and `isDisabled` (Issue 6).
- **Smaller unrelated changes:**
  - `saveYjsVersionAsWorkItemContent` log narrowing;
  - Dropdown `whitespace-nowrap`;
  - provider refactors (`cleanEscapedNewlines`, `EMPTY_AI_CHANGES`);
  - another personal `.gitignore` entry.

---

## Architecture Analysis

**File-based orders:** unchanged from #2462. The Huxley DOCX is stored at submit. `useReviewJobDeltaVersion` finds it by filtering the order's versions for `ReviewJobDelta`, and it appears as a fourth `FilesSection` row with no reviewer attribution. The accordion is hidden for JSON orders, so a JSON delta is never offered as a download (verified).

**HTML (wysiwyg) orders:**
- **At submit:**
  - `saveYjsVersionAsWorkItemContent` now returns the double-encoded snapshot it stored.
  - `storeWysiwygReviewDelta` loads the service job's newest `yjs` snapshot as the baseline and decodes both snapshots.
  - `filterTrackChangesToStage` keeps only track-change marks whose `data-change-id` isn't in the baseline. Earlier-stage and AI changes are applied: deletions drop, insertions unwrap.
  - The resulting ProseMirror JSON is stored as `ReviewJobDelta`.
- **Admin view:** the admin editor fetches it (`useReviewJobDeltaContent`) and renders it in a separate read-only Tiptap instance, never the collaborative one. It gets a "Reviewer Delta Copy" dropdown entry, and comments, AI cards and reject are disabled.

**Verified sound:**
- **Encoding:** the double-base64 snapshot round-trips, and the stored JSON round-trips non-ASCII text.
- **Wiring:** all three submit paths pass the snapshot.
- **Caller safety:** the change to `saveYjsVersionAsWorkItemContent`'s return value doesn't break other callers, since they ignore it.
- **Change ids:** stable across jobs, because there's one Y.js document per order.
- **Editor isolation:** the delta editor never writes to the shared document, and track-change visibility switches correctly between views.
- **Baseline selection:** a second Service job can't be added through the UI or the server guards, so the first-Service-job lookup is safe in practice.

**Relative to #2462:** this PR removes the wysiwyg Huxley parser, which resolves #2462's sanitiser-bypass, parse-cost and parser-edge-case findings for HTML orders. #2462's file-based concerns remain: no timeout or byte cap on the delta download. This PR adds a similar inline, warn-only step for HTML orders (Issues 3, 4).

**Review lenses run:** delta-algorithm correctness (executed against the real TrackChangesV2 extension in a headless editor), UI wiring/regressions/security/a11y, and tests/resilience/conventions. That was 3 finders plus 2 adversarial verifiers.

---

## Issues Found

### 1. Review edits next to the same user's earlier changes are missing from the delta

**[File: packages/wysiwyg/src/utils/filterTrackChangesToStage.ts]**

> **In plain terms:** Sometimes one person does both the editing and the review, for example an admin covering a job. When they review, any change they make right beside one of their own editing-stage changes is treated as editing work and left out of the reviewer delta. If that's all they changed, no delta is saved. It can't be regenerated later.

**Function/Class:** `filterTrackChangesToStage`, `storeWysiwygReviewDelta` (via the trackChanges-v2 change-id merge)

**Severity:** medium

**Confidence:** high (reproduced with the real editor extension)

**Steps to reproduce:**

1. On an HTML (wysiwyg) order, log in as an admin and work the **service** job with tracking on. Type an insertion (e.g. "AAA") and submit the service job.
2. As the **same** admin, work the review job. Place the cursor directly after "AAA" and type "BBB" (or backspace next to one of your own tracked deletions). Submit the review.
3. Open the order panel, then the HTML pill, then the "Reviewer Delta Copy" dropdown entry.
4. **Expected:** "BBB" is shown as a reviewer change.
5. **Actual:** "BBB" is unmarked or missing. If it was the only review edit, the entry doesn't appear, because nothing was stored and the log says "the review stage changed nothing".

**Problem:** The filter decides the stage purely by change id (`!baselineChangeIds.has(changeId)`). But trackChanges-v2 reuses a neighbouring mark's id whenever the change type and user id match, with no timestamp or session condition:
- `extensions/trackChanges-v2/plugins/tracking.ts:363-376` (insertions);
- `keyboard/handleDeletion.ts:938-956` (deletions).

Backspacing into your own baseline insertion hard-deletes it and leaves no mark (`handleDeletion.ts:583-605`). The PR's own docblock notes that "an admin often works the review job", and no guard stops one user holding both jobs.

**Evidence:** In a headless harness driving the real extension, a fresh editor was loaded from the baseline snapshot:
- User 42 types "BBB" after their own baseline "AAA". Result: `"AAABBB" trackChange#<baseline id>`, and the delta ids are `[]`.
- User 99 does the same. Result: a new id, and the delta keeps "BBB".

**Impact:** The delta is incomplete or absent, permanently, exactly in the admin-covers-both-jobs case the design says is common.

**Fix:** Don't infer the stage from id membership alone. Options:
- (a) Compare each baseline id's extent/text between the two snapshots and treat added text under a reused id as review-stage.
- (b) Diff the applied baseline text against the applied reviewed text within each change id.
- (c) At minimum, detect when the same `data-user-id` appears in both jobs, log or Sentry-report it, and document the limitation.

Add a test for same-user adjacent edits.

### 2. Text the reviewer types at the end of an unresolved AI deletion is dropped

**[File: packages/wysiwyg/src/utils/filterTrackChangesToStage.ts]**

> **In plain terms:** If the editor left a Proofed AI deletion unresolved and the reviewer types right where it ends, the typed text is missing from the reviewer delta, and the delta may not be saved at all.

**Function/Class:** `filterTrackChangesToStage`

**Severity:** medium

**Confidence:** high (reproduced); how often it happens depends on caret placement in a real browser

**Steps to reproduce:**

1. On an HTML order, run Proofed AI during the service job and leave at least one AI deletion unresolved. Submit the service job.
2. As the reviewer, place the caret at the end of that AI deletion and type "NEW". Submit the review.
3. Open "Reviewer Delta Copy".
4. **Expected:** "NEW" is shown as a reviewer insertion.
5. **Actual:** "NEW" is absent. If it was the only review edit, there's no delta entry.

**Problem:** `DelMark` (`extensions/aiChanges/index.tsx:10-47`) doesn't set `inclusive: false`, so typed text inherits the `del` mark alongside the reviewer's `trackChange`. The filter returns `null` for any node with an anchored AI `del` (`if (aiChange.type === AI_DELETE_MARK) { return null; }`) before it checks for a review-stage track change. Service submit doesn't require AI changes to be resolved, and flattening happens only on a server copy, so the live document still has the marks at review time.

**Evidence:** Harness result: `"NEW" del#ai-1, trackChange#…/insertion/u99` gives delta text `["Keep "," tail"]` with ids `[]`. The existing test only covers `ins` plus a review mark (`filterTrackChangesToStage.test.ts:294`), not `del`.

**Impact:** Reviewer insertions are silently lost from the stored delta. The flattened submitted copy has the same underlying problem (an older editor bug), but the delta should show what the reviewer did.

**Fix:** Evaluate the track-change mark first. When the run carries a review-stage `trackChange`, keep the node and strip `del`. Longer term, set `inclusive: false` on `DelMark` (and `InsMark`). Add the `del` + review-mark test.

```typescript
if (aiChange?.type === AI_DELETE_MARK && !isFromThisStage) {
  return null;
}
```

### 3. The PR description contradicts the code (stored at submit, not derived; no backfill)

**[File: apps/creative-portal/api/aiReviewFeedback/storeReviewJobDelta/storeWysiwygReviewDelta.ts]**

> **In plain terms:** The PR tells testers that HTML-order deltas are worked out when the page opens and appear even on old orders. In reality they're saved when the review is submitted and never exist for orders reviewed before release. Anyone testing against the description will either raise false bugs or sign off on behaviour that doesn't exist.

**Function/Class:** n/a (PR description vs `storeWysiwygReviewDelta`, `OrderManagment/index.tsx`)

**Severity:** medium

**Confidence:** high

**How to spot it:** PR hygiene, not user-reproducible. Compare the PR body with the code.

**Problem:**
- **"Nothing is stored… derived when the view opens":** false. The tip commit fed4c697c "Store the reviewer delta at submit instead of deriving it" adds `storeWysiwygReviewDelta`, which calls `addWorkItemContentVersion(... REVIEW_JOB_DELTA)`.
- **"Works on orders that submitted long before the feature existed":** false. `OrderManagment/index.tsx:107-111` says the delta "is written on submit and cannot be backfilled", and `useReviewJobDeltaVersion.test.ts:119` asserts "returns nothing for an order that predates the feature".
- **`useReviewerDeltaSnapshots`:** doesn't exist.
- **"11 for useReviewJobDeltaVersion":** the file has 7 tests. The new `storeWysiwygReviewDelta` (12), `reviewDelta` (7) and `useReviewJobDeltaContent` (6) suites aren't mentioned.
- **Manual testing:** unchecked, and the Results section still says "Screenshots … to be added".

**Impact:** QA and PO acceptance runs against wrong expectations. The no-backfill limitation is a product decision that the description hides.

**Fix:** Rewrite the description to match fed4c697c: stored at submit, no backfill, correct areas of change and test counts. Add screenshots and complete manual testing.

### 4. A failed delta store is lost for good with no alert, and it runs inline with no timeout

**[File: apps/creative-portal/api/aiReviewFeedback/storeReviewJobDelta/storeWysiwygReviewDelta.ts]**

> **In plain terms:** If anything goes wrong while saving the HTML-order delta, such as a slow or failing internal service, the delta is quietly never saved and can't be recovered, and nobody is alerted. The reviewer's Submit also waits for this extra work, with no time limit.

**Function/Class:** `storeWysiwygReviewDelta`

**Severity:** low

**Confidence:** high

**How to spot it:** Code health and observability, not user-reproducible without fault injection. Every exit logs only through `logger.info` or `logger.warn`: no service job (`:88`), no baseline (`:112`), empty content (`:127`), decode failure (`:146`), and the final catch (`:201`). The sibling `saveYjsVersionAsWorkItemContent.ts:107` calls `reportError`.

**Problem:** The docblock itself says "There is no second chance and no way to backfill", yet failures aren't sent to Sentry. The step makes four sequential OMS calls (one returns the full manuscript), two full Y.js decodes and several recursive walks, all awaited before the Submitted PATCH (`postSubmitJob.ts:136-150`, `patchJob.ts:177`), with no timeout configured anywhere. The admin side hides failures too: `useReviewJobDeltaContent.ts:57-61` `catch { return { reviewDeltaContent: undefined }; }` ignores query errors, so "no delta" and "delta failed" look identical.

**Impact:** Deltas can go missing silently and can't be diagnosed from either end, and there's added submit latency on large manuscripts. This matches #2462's warn-only policy on the file-based path, hence low.

**Fix:** Call `reportError` in the catch and on the unexpected-state exits. Bound the step with an overall timeout (`Promise.race`), and drop the redundant `countTrackChanges` walk used only for logging. Report decode failures client-side.

### 5. The shared version route now silently renders non-blocks content as HTML

**[File: packages/shared/api/workItemContentVersion/[id]/getWorkItemContentVersion/getWorkItemContentVersion.ts]**

> **In plain terms:** When a stored document is in an unexpected format, the admin editor used to show an error. Now it may open with garbled or empty text and no error, so a corrupt document looks like a real one.

**Function/Class:** `getWorkItemContentVersion` (`parseBlocksOrNull`)

**Severity:** low

**Confidence:** high (code); whether the output is garbage depends on the content

**How to spot it:** Code health; the path isn't reached by this PR's own features. `parseBlocksOrNull` returns `null` for anything that isn't `{ blocks: [...] }`, and the handler then uses `from: workItemContentBlocks ? "blocks" : "html", input: workItemContentBlocks ?? workItemContentInJson`. Before this change, `JSON.parse` threw.

**Problem:**
- **Stated reason unreachable:** the fallback exists for "early PP-2052 review deltas written as raw annotated HTML", but no delta reader requests conversion: `useReviewJobDeltaContent.ts:38` passes `convertContentToTiptap: false`, and the shared fetcher only sends `returnContent`.
- **Changed behaviour elsewhere:** the only caller it affects is `WysiwygModal/index.tsx:90` (the original version, `convertContentToTiptap: true`). Corrupt or non-blocks originals now return 200 instead of failing loudly.
- **No tests:** the route folder has no test file.
- **Pre-existing log leak:** `inputBlocksCount: workItemContentBlocks?.blocks` (`:119`) still logs the whole blocks array (manuscript text) under a "count" label. This isn't introduced here, but the PR touched the line.

**Impact:** A shared route used by the admin editor now fails silently, with no test protecting it.

**Fix:** Drop the fallback: the legacy HTML deltas exist only on #2462's unmerged devtest data, and no reader converts them. Or limit it to content that starts with `<`, and log or report when it triggers. Change the log field to `?.blocks?.length`. Add route tests covering blocks, HTML and malformed JSON.

### 6. Unused PP-2051 props and a mangled export comment ship in this PR

**[File: apps/creative-portal/components/organisms/sidebars/contents/HTMLContentPill/types.ts]**

> **In plain terms:** The PR includes settings for a different ticket (a reduced version menu and a disabled pill). They do nothing and nothing uses them, but the code comments describe behaviour that doesn't exist.

**Function/Class:** `HTMLContentPillProps.versionMenu`, `HTMLContentPillProps.isDisabled`; `packages/wysiwyg/src/index.tsx` exports

**Severity:** low

**Confidence:** high

**How to spot it:** Code health, not user-reproducible.
- **`versionMenu`:** threaded through `HTMLContentPill/index.tsx:41,94` → `WysiwygModalReadOnly/index.tsx:31,42,127` → `<WysiwygEditor>`. Nothing in `packages/wysiwyg` or `apps/storybook` reads it, and no caller sets it. It type-checks only because the package import resolves as `any`.
- **`isDisabled`:** no caller sets it (`HTMLContentPill/index.tsx:37,69`).
- **Mangled comment:** in `packages/wysiwyg/src/index.tsx:208-217`, an orphan `// reachable from there.` sits above the filter export. The sentence's beginning ("PP-2053: the review delta is computed on submit… so the filter has to be") sits above the unrelated `getStateUpdateFromYjsBase64` export; import sorting split it.

**Problem:** Out-of-scope, untested code whose comments describe behaviour that doesn't exist, plus a broken comment.

**Impact:** It misleads future readers, and PP-2051 will inherit an API nobody has exercised.

**Fix:** Remove `versionMenu`, `VersionMenu` and `isDisabled` and land them with PP-2051. Rejoin the export comment above one block.

### 7. Test gaps on the risky paths

**[File: packages/wysiwyg/src/utils/filterTrackChangesToStage.test.ts]**

> **In plain terms:** The automated tests don't cover the cases where the delta goes wrong (Issues 1 and 2), or several of the new screen behaviours, so these problems weren't caught and could come back.

**Function/Class:** delta filter, `EditorContext` provider, `ChangeBox`, `CommentButton`, shared route, `postSubmitJobStream`, `useJobDropdownActions`

**Severity:** low

**Confidence:** high

**How to spot it:** Code health, not user-reproducible.
- **Missing filter tests:** the 24 filter tests have no same-user id-merge case, no AI `del` + review-mark case, and no V1 + V2 mixed-mark case.
- **Missing test files:** `packages/wysiwyg/src/contexts/EditorContext/provider.tsx` (the delta editor instance and track-changes visibility effect), `ChangeBox` and `CommentButton` (delta-view behaviour), and the shared route fallback (Issue 5).
- **Unchanged test files:** `postSubmitJobStream.test.ts` has no `reviewedYjsInBase64` assertion, and `useJobDropdownActions.test.ts` doesn't cover the new invalidation.
- **Partly covered:** `patchJob.test.ts:638-645` and `postSubmitJob.test.ts:315-322` do cover the wiring, and `storeWysiwygReviewDelta`/`reviewDelta` tests use real Y.js fixtures rather than mocks.

**Problem:** The highest-risk logic, deciding which changes belong to the review stage, isn't tested against real editor behaviour.

**Impact:** Permanent data-quality bugs in stored deltas can regress unnoticed.

**Fix:** Add filter tests built from snapshots produced by the real extension (same-user adjacent edits, typing at the end of an AI deletion, mixed V1/V2 marks), a provider test for the delta editor, and a `postSubmitJobStream` wiring test.

---

## Open Questions

- **Req 1.1 / 1.2:** Figma `55837-30002` / `42216-25250` show the delta "from the order/job history". Is a "feedback provided" indicator expected on the review job row or the HTML pill? For HTML orders the only sign today is a dropdown entry inside the editor, and no job-history UI changed. — `OrderManagment/index.tsx:402-420`
- **Rejects:** should a review whose only action was rejecting the editor's changes (or hard-deleting their own insertions) produce a delta? Diffing by change id can't see rejections, so today nothing is stored even though the copies differ. — `apps/creative-portal/api/utils/wysiwyg/reviewDelta.ts` (`buildReviewStageDelta`)
- **Reject button reachability:** can an order stay **Live** after the review submits (e.g. a QA job follows)? If it can, an admin opens the editable modal with `role: "admin"`, and the delta view's change cards show an active Reject button that removes the change from the on-screen record (nothing is saved). AI cards hide it; `ChangeBox` doesn't. — `packages/wysiwyg/src/components/molecules/ChangeBox/index.tsx:66-73`
- **Same-user frequency:** how often does the same user work both the service and review job on HTML orders? That decides how often Issue 1 bites. — `storeWysiwygReviewDelta.ts`
- **Legacy V1 documents:** are older-format track-change documents still in flight? A format change over a V1 insertion keeps the editor's V1 mark in the delta, because only the first track-change mark on a run is inspected. — `filterTrackChangesToStage.ts:138`
- **Access control:** delta visibility is gated only by the admin UI. The version search and by-id routes check no role in the BFF (this predates the PR). Can an assigned editor list and fetch `ReviewJobDelta` versions through the API directly, and is that acceptable given PP-2051's plans? — `apps/creative-portal/api/workItemContentVersion/searchWorkItemContentVersion`
- **Merge order:** this PR must merge after #2462 (or be retargeted). Will #2462's wysiwyg parsing be dropped there first, or only through this PR? If #2462 merges alone, its wysiwyg findings still ship.

---

## Validation Checks

| Check | Result | Notes |
| --- | --- | --- |
| `npx turbo run test` | ⏭️ Skipped | User opted out |
| `npx turbo run typecheck` | ⏭️ Skipped | User opted out |
| `npx turbo run lint` | ⏭️ Skipped | User opted out |
| `npx turbo run build` | ⏭️ Skipped | User opted out |

Scope would have been the full monorepo (`packages/shared` and `packages/wysiwyg` changed). Reviewers executed the real delta filter, trackChanges-v2 extension and Y.js decode in scratch harnesses; that doesn't replace the suite.

---

## Tests

- ⏭️ Validation suite not run (user opted out). CI must pass before merge.
- ✅ 24 `filterTrackChangesToStage` tests, including reviewer-deletion survival and AI `ins` + review mark
- ✅ `storeWysiwygReviewDelta` (12) and `reviewDelta` (7) use real Y.js fixtures, not mocks
- ✅ `useReviewJobDeltaVersion` (7), `useReviewJobDeltaContent` (6), `FilesSection` hooks/utils, `useDocumentVersion`, `useAiChanges`, `useAutoSwitchToTrackChanges`, `AiChangeBox` delta cases
- ✅ `patchJob` / `postSubmitJob` assert `reviewedYjsInBase64` wiring
- ❌ No same-user id-merge, AI `del` + review mark, or mixed V1/V2 filter tests (Issues 1, 2)
- ❌ No tests for `provider.tsx` delta editor, `ChangeBox`/`CommentButton` delta behaviour, shared route fallback, `postSubmitJobStream` wiring, `useJobDropdownActions` invalidation (Issue 7)
- ❌ Manual testing not done per PR checklist; no screenshots

### Suggested manual QA script

1. **(Req 2.1)** File-based DOCX order with AI feedback run during review: after the review submits, confirm the order panel's Files section shows "Reviewer Delta Copy" with no "Reviewed by", and that it downloads a DOCX with tracked changes.
2. **(Req 2.3)** HTML order, with different users for editing and review: after the review submits, open the HTML pill. Confirm the "Reviewer Delta Copy" dropdown entry appears and shows only the reviewer's changes, not the editor's or AI's.
3. **(Issue 1)** HTML order where the **same admin** works the service job and then the review job, making a review edit directly after one of their own editing insertions. Check whether that edit appears in the delta.
4. **(Issue 2)** HTML order with an unresolved AI deletion. As the reviewer, type at its end and submit. Check whether the typed text appears in the delta.
5. **(Issue 3)** Open an HTML order completed **before** this release. Confirm there's no delta entry, then confirm with the PO that this is acceptable.
6. **(Open Question, rejects)** As a reviewer, only reject one of the editor's changes and submit. Note whether a delta entry exists.
7. **(Req 1.3 of PP-2052)** As the customer, open both orders above. Confirm no delta appears in downloads, Track Changes or order details.
8. **(Open Question, Reject button)** If an order can remain Live after review, open the HTML pill as an admin, pick "Reviewer Delta Copy", and check whether change cards show a Reject button.
9. **(Delta view)** In the delta view, confirm the comment button is disabled, no AI cards show, and switching back to Track Changes or Clean restores normal behaviour.

---

## Summary

| Aspect | Status |
| --- | --- |
| Correctness | ⚠️ Works for the common case; delta misses review edits in two confirmed scenarios (Issues 1, 2) |
| Regression risk | ⚠️ Low–medium: shared route fallback (Issue 5); editor provider changes are well contained |
| Tests | ⚠️ Good volume with real fixtures; the confirmed failure cases are untested (Issue 7) |
| Accessibility | ✅ New entry and row reuse existing button and `FilePill` markup |
| Error handling | ⚠️ Silent, permanent loss with no Sentry; no timeout (Issue 4) |
| Security | ✅ Delta can't reach customers; delta editor isolated from the shared document. API-level role gating is an open question; run `/security` |
| Code quality | ⚠️ Stale description, dead PP-2051 props, mangled comment (Issues 3, 6) |
| Validation suite | ⏭️ Skipped (user opted out) |
| Mergeable state | ✅ GitHub `clean` (stacked on #2462; validation not run) |

---

## Recommendation

**Approve with suggestions.** No blockers. The rubric allows a plain Approve for medium/low findings, but the stored delta is permanent and can't be backfilled, so Issues 1–3 should be resolved before merge.

1. Fix change attribution so same-user adjacent edits aren't folded into the baseline (Issue 1), and check for a review-stage track change before dropping AI-`del` runs (Issue 2). Add tests built from real extension output.
2. Rewrite the PR description to match fed4c697c (stored at submit, no backfill, correct test counts), and complete manual testing with screenshots (Issue 3).
3. Report delta-store failures to Sentry and bound the step with a timeout (Issue 4).
4. Drop or narrow the shared route's HTML fallback, and add route tests (Issue 5).
5. Remove the PP-2051 `versionMenu` / `isDisabled` props and fix the export comment (Issue 6).
6. Get product answers on: a feedback indicator and job-history entry (Req 1.1/1.2), reject-only reviews, and no backfill for pre-release orders.
7. Merge after #2462 and run the full validation suite in CI (it wasn't run locally). Run `/security` before merge.
