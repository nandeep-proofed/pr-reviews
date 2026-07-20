# PR Review: feature/PP-1991: Raise AI feedback limits to 100k words / 50 MB

**PR:** https://github.com/Proofed/B2BWebserver/pull/2377
**Jira:** https://proofed.atlassian.net/browse/PP-1991
**Status:** In Progress

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Word limit 30,000 → 100,000 | `MAX_AI_REVIEW_FEEDBACK_WORD_COUNT` 30_000 → 100_000 in `apps/creative-portal/api/aiReviewFeedback/consts.ts`; consumed by both the client eligibility hook and the server submit handler | ✅ Addressed |
| File-size limit 25 MB → 50 MB | `MAX_AI_REVIEW_FEEDBACK_FILE_SIZE_BYTES` 25 → 50 MB in `packages/shared/api/aiReviewFeedback/consts.ts`; single source of truth for server busboy limit, 413 message, client eligibility + tooltip | ✅ Addressed |
| New thresholds replace old ones **everywhere** limits are enforced, across all formats (DOCX, RTF, TXT, HTML, ODT, PDF) | Both enforcement points (client `useAiFeedbackEligibility`, server `submit` handler) read the shared constants; gating is format-agnostic (word count / byte size, not per-format) | ✅ Addressed |
| Documents at/below new limits trigger existing flow unchanged | No flow/trigger code touched; only threshold constants changed | ✅ Addressed |
| Exactly 100,000 words / 50 MB treated as within limits (eligible) | Both word gates use strict `>`; client file gate is `file.size > MAX_AI_FILE_BYTES` (strict); busboy fileSize limit fires only when exceeded — boundary is eligible | ✅ Addressed |
| Documents over new limits fall back to manual form | Unchanged fallback path; only the numeric threshold moved | ✅ Addressed |
| Unchanged: Proofed-platform only, `aiEnabled`, env toggle, formats, polling, cancel, failure fallback | None of that code touched | ✅ Addressed (scope guard respected) |
| User-facing copy updated to new limits (limits tooltip + submission error messaging) | `AI_UNAVAILABLE_TOOLTIP` "over 30k words" → "over 100k words" (MB already interpolated); both server 413 messages interpolate the constants (`…50 MB upload limit`, `…100,000-word AI feedback limit`) so they auto-update | ✅ Addressed |

**Scope creep:** None. The PR is limited to the two threshold values, the one hardcoded copy string, two stale code comments, and one over-limit test fixture. No refactors or bonus fixes. (A follow-up commit from this review also reworded stale comments in the `streamingMultipart` test — see issue #1.)

---

## Architecture Analysis

The change is a pure threshold bump against an already-well-factored gating layer. The two limits live in exactly two constants:

- **File size** — `packages/shared/api/aiReviewFeedback/consts.ts` is the documented single source of truth. It feeds the server busboy `fileSize` limit + the 413 message, and the client `AiFeedbackPanel` eligibility/tooltip (via the `MAX_AI_FILE_BYTES` alias). Bumping it moves client and server in lock step, so the client can never admit a file the server would reject.
- **Word count** — `apps/creative-portal/api/aiReviewFeedback/consts.ts` is read by both the client `useAiFeedbackEligibility` hook (`workItemSize > MAX_…`) and the server submit handler's mirror gate (`order.workItemSize > MAX_…`).

Because every enforcement point and every user-facing message dereferences these constants (the only hardcoded literal was the `"30k"` token in the tooltip, which this PR fixes), the change is complete and internally consistent. Boundary semantics (strict `>` everywhere, busboy firing only on exceed) satisfy the "exactly at the limit is eligible" acceptance criterion without any additional work.

---

## Issues Found

### 1. Stale "25 MB production cap" comments in the streaming-upload test — ✅ RESOLVED (commit `55ca398fd`)

**[File: packages/shared/utils/streamingMultipart.test.ts]**

**Function/Class:** "bounds memory by highWaterMark…" test (lines ~191–254)

**Severity:** low

**Problem:** The test's `it(...)` title and several inline comments described its 400-chunk fixture as "the 25 MB production cap" / "matches the production upload cap". `streamingMultipart` has exactly one consumer — `apps/creative-portal/api/aiReviewFeedback/strategy/ai-review-feedback.ts`, which streams the reviewer file to Proofed.ai — so "the production upload cap" *is* `MAX_AI_REVIEW_FEEDBACK_FILE_SIZE_BYTES`, the constant this PR raises to 50 MB. (An earlier draft of this review called it an "unrelated" test — that was wrong; it is the AI-feedback upload path's own utility test.) The comments were therefore stale after the bump.

**Impact:** Cosmetic only. The test asserts streaming/back-pressure mechanics (`peak <= 128 KB`) using a hardcoded chunk count that is independent of the cap value, so behavior and pass/fail were unaffected — only the "production cap" wording misdescribed why `400` was chosen.

**Resolution:** Reworded the title and comments to decouple them from the cap (the 25 MB payload is now documented as an arbitrarily large stream chosen only to exercise back-pressure, explicitly noting it need not track the cap). Payload size and all assertions unchanged. Deliberately **not** resized to 50 MB — doubling the payload would double this memory test's runtime (it carries a 15 s timeout) for zero extra coverage, since the guarantee holds at any large size. Verified: 9/9 `streamingMultipart` tests pass, ESLint clean.

### 2. `yarn bump-packages` not run for the shared-package change (process) — ✅ RESOLVED (not required, mark N/A)

**[File: packages/shared/api/aiReviewFeedback/consts.ts]**

**Function/Class:** n/a (package versioning)

**Severity:** low

**Problem:** The PR modifies `@proofed/shared`, and the PR checklist item "`yarn bump-packages` run and committed" is unchecked. Investigated whether the team convention requires a bump.

**Finding — not required:** Version bumps in this repo are a **release-time** action, not per-PR:

- The root/app version is `0.91.0-0`, last changed by `fix/PP-964` (#1603). Every feature PR merged to `develop` since — including the directly comparable AI-feedback work (`PP-1720` #2291/#2314, `PP-1811`, `PP-1633`) and the just-merged `PP-1987` (#2374/#2375) — merged **without** bumping the version. The dedicated version changes in history are all `chore: Updated version to oms-X.Y.Z` / `cp-X.Y.Z` commits cut on the `rc-oms-*` / `rc-cp-*` release branches (matches CLAUDE.md's "Release branches" convention).
- `bump-packages` runs `yarn version --prerelease && yarn sync-versions` against the **app** version; `@proofed/shared` is pinned at `0.0.1` and consumed as an internal workspace dep (`transpilePackages`), never published — so it wouldn't even reflect the shared change. Running it here would spuriously increment the app prerelease and add needless merge-conflict surface against other in-flight branches.

**Fix:** No action needed — tick the checklist item as **N/A**. The version bump belongs to the release cut, not this feature PR.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⚠️ Partial | Targeted AI-feedback suites run green this session: `AiFeedbackPanel` + `services/aiReviewFeedback` + `useAiFeedbackEligibility` = 133 pass; `useReviewSubmissionFormState` + `AiUnavailableHeader` = 27 pass; `streamingMultipart` = 9 pass. Full-repo `turbo run test` not run (user opted to reuse earlier results). |
| `npx turbo run typecheck` | ✅ | `@proofed/creative-portal` + `@proofed/shared` — 0 errors. |
| `npx turbo run lint` | ✅ | ESLint on all changed files — 0 errors. |
| `npx turbo run build` | ⏭️ Not run | User opted to reuse earlier results; build was not executed. |

> Validation was run against the main worktree by temporarily applying the changed files (byte-identical between `develop` and the working branch), then reverting. The PR-branch worktree can't run a fresh `yarn install` because `TIPTAP_PRO_TOKEN` isn't set in this environment.

---

## Tests

- ✅ Over-limit word-count fixture updated (`60_000` → `120_000`) so it still represents an ineligible document above the new 100k limit — the one test-data change the threshold bump required.
- ✅ Existing boundary tests are threshold-agnostic: `AiFeedbackPanel/utils.test.ts` and `useAutoTriggerOnFileUpload.test.tsx` reference `MAX_AI_FILE_BYTES ± 1` (not literals), so they continue to assert correct boundary behavior at the new cap.
- ✅ Tooltip test (`AiUnavailableHeader/index.test.tsx`) asserts against the imported `AI_UNAVAILABLE_TOOLTIP` constant, so the copy change is covered without a literal update.
- ⚠️ No *new* test was added, which is appropriate here — this is a constant change fully exercised by the existing suite (eligibility strict-`>` boundary, tooltip copy, over-limit fixture). The project's "every PR must include tests" rule is satisfied by the updated fixture + existing coverage rather than net-new tests.
- ⚠️ Full-repo test run and build were not executed (user opted to reuse earlier partial results). Recommend a full `npx turbo run test && npx turbo run build` before merge.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ |
| Regression risk | ✅ Low |
| Tests | ⚠️ Adequate (fixture updated + existing coverage; full suite/build not run this review) |
| Code quality | ✅ |
| Validation suite | ⚠️ Partial (typecheck + lint pass; targeted tests pass; full test + build not run) |
| Mergeable state | ✅ Clean (GitHub `mergeable_state: clean`) |

---

## Recommendation

**Approve with suggestions**

The implementation correctly and completely satisfies every PP-1991 acceptance criterion. The gating layer was already factored around two shared constants, so the change is a clean two-value bump plus the one hardcoded tooltip token and a required over-limit test-fixture update. Boundary semantics (strict `>` everywhere) correctly make exactly 100,000 words / 50 MB eligible, matching the ACs. No scope creep, no regression risk to untouched gating (platform/`aiEnabled`/format/polling/fallback).

Before merge:

1. Run the full validation suite once in a properly provisioned environment: `npx turbo run test` (all workspaces) and `npx turbo run build`. This review only executed targeted AI-feedback tests + typecheck + lint (per user choice); the full test run and build were not performed here.
2. ~~(Optional, low) Update or reword the now-stale "25 MB production cap" comments in `packages/shared/utils/streamingMultipart.test.ts`~~ — ✅ **Done** in commit `55ca398fd` (comments/title reworded; 9/9 tests still pass).
3. ~~Confirm whether `yarn bump-packages` is required for the `@proofed/shared` constant change per team convention~~ — ✅ **Resolved: not required** (version bumps are release-time on `rc-*` branches; recent feature PRs don't bump; `@proofed/shared` is unpublished/internal). Mark the checklist item **N/A**.
4. Manual test per the ticket's Testing Notes: upload a 30k–100k-word document (≤50 MB) and confirm the AI Feedback card shows and drafts automatically; confirm a >100k-word or >50 MB document falls back to the manual form with the updated tooltip copy.
