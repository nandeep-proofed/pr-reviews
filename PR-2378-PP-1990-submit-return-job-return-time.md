# PR Review: fix/PP-1990: Send returnTime when auto-assigning a return job

**PR:** https://github.com/Proofed/B2BWebserver/pull/2378
**Jira:** https://proofed.atlassian.net/browse/PP-1990
**Status:** In Progress (fix version oms-0.104.0)

> **Reviewer conflict of interest:** this PR was authored by the same assistant writing this review. Findings below are stated as plainly as possible, including two that materially undercut claims in the PR description — but an independent human review is still warranted before merge.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Selecting the ellipsis action on an **unassigned** return job must auto-assign the acting admin | `submitReturnJob.ts` already did this via a `UserAssign` PATCH; the PATCH was rejected by OMS for a missing `returnTime`. Now enriched via `enrichUserAssignPayload`, so the assign lands. | ✅ Addressed |
| …and submit the job in a single action, without requiring prior assignment | Assign → Active → Submitted chain is unchanged and now reachable, since the assign no longer 400s. Covered by the "submits the job through to Submitted" test. | ✅ Addressed |
| Retain the previous (valid) one-click process — no manual assignment workaround | No UI/menu change; the fix is confined to the BFF, so the existing one-click entry point is restored rather than replaced. | ✅ Addressed |
| Regression must be fixed (previous process allowed admin one-click submit) | Root cause is a genuine regression from the Dynamic Return Time work (`returnTime` made mandatory on `UserAssign`; `enrichUserAssignPayload` wired into `patchJob.ts`/`patchJobs.ts` but not these two routes). Fixed at that root cause, not the symptom. | ✅ Addressed |

**Scope beyond the ticket:** the bulk route (`submitReturnJobs.ts`) is fixed alongside the reported single-job route. Same defect, different admin surface, not reported by QA. Included deliberately at the author's request. Reasonable — the two routes are copies of each other and would otherwise drift — but it is scope creep relative to PP-1990 and expands what QA must cover.

**Ticket wording mismatch (worth recording on the ticket):** PP-1990's title and description say "Mark as Returned", but the action that actually breaks is the admin **"Submit"** item — the `isAdmin && isInQueueOrOnHold` branch in `useJobDropdownActions.ts:230`. "Mark as Returned" is the label rendered for Assigned/Active jobs (line 240-249), which skip the assign branch entirely and were never affected. The screenshot in the ticket confirms "Submit" + In Queue. Anyone testing this by literally following the ticket's steps may test the wrong menu item.

---

## Architecture Analysis

The fix is minimal and idiomatic: it reuses `enrichUserAssignPayload`, the helper introduced by the Dynamic Return Time work precisely to centralise `returnTime` computation, whose own header comment states it exists "so every assignment entry point … ends up sending OMS a complete payload". These two routes were simply never wired in — they synthesise their own `UserAssign` payload rather than forwarding a client one, so no caller-supplied `returnTime` could cover for them.

The one novel wrinkle is `requestType`. `patchJob.ts` and `patchJobs.ts` keep `requestType` in the body (stripping it only for `Review`), because their payloads carry it from the client. These two routes never had it in the body — it travels only in the header via `patchUpdateJob`'s third argument. The helper gates on `payload.requestType === "UserAssign"`, so the PR adds it purely to satisfy the gate and `omit`s it again before the call. This keeps the wire payload byte-identical to today apart from the added `returnTime`, which is the conservative choice and is locked in by a test. It is, however, a slightly awkward round-trip — worth a comment, which the merged code lacks (Issue 3).

---

## Issues Found

### 1. Single-job route enriches from the query `jobId` but patches the body `id`

**[File: apps/creative-portal/api/mixtures/jobs/[jobId]/submitReturnJob/submitReturnJob.ts]**

**Function/Class:** submitReturnJob

**Severity:** medium

**Problem:** The enricher is called with `jobId: Number(jobId)` — the value from `validatedData.query` — while the PATCH it feeds targets `job.id` from the **body** (`patchUpdateJob` builds its URL as `/${job.id}`). The schema validates `query.jobId` and `body.id` independently and never cross-checks them. The bulk route does not have this problem: it passes `jobData.id`, the same id it patches.

**Impact:** For a request where the two ids diverge, the enricher fetches job A's `returnTime` and writes it onto job B's assignment — a silent wrong-deadline write, not an error. The client always sends matching ids (`apiRoutes.mixturesSubmitReturnJob(jobUpdateData.id)`), so this is not reachable through the UI; it requires a hand-crafted request from an authenticated admin. Note the divergence is pre-existing in spirit — `fetchJobTasksByJobId(jobId, …)` on line 84 already uses the query id while the patches use the body id — but this PR adds the first consumer where the mismatch corrupts a *written* value rather than a read.

**Fix:** Enrich the job that is actually being patched, matching the bulk route:

```typescript
const enrichedAssignData = await enrichUserAssignPayload({
  requesterId,
  jobId: jobUpdateData.id,
  payload: { ...assignJobData, proofedUserId: requesterId, requestType: "UserAssign" as const }
});
```

Better still, add a cross-field check to `submitReturnJobRequestSchema` so `body.id` must equal `query.jobId`, which closes the pre-existing ambiguity for every consumer in the route.

### 2. Bulk route issues two extra OMS round-trips per assigned job

**[File: apps/creative-portal/api/mixtures/jobs/submitReturnJobs/submitReturnJobs.ts]**

**Function/Class:** submitReturnJobs

**Severity:** low

**Problem:** The PR passes neither `currentJob` nor `siblingJobs`, so `enrichUserAssignPayload` self-fetches: one `fetchJob` plus (when the job has no stored `returnTime`) one `fetchJobsByOrderId` per job. `patchJobs.ts` deliberately avoids exactly this, prefetching siblings once per unique order and passing them through — with a comment calling the prefetch "the N+1 guard", and a test asserting it.

**Impact:** A bulk submit of N In Queue jobs adds up to 2N OMS calls, all inside a `Promise.all`. Not a correctness bug, and the route already calls `fetchOrderById` per job without deduping (pre-existing), so it is consistent with the surrounding code's existing looseness rather than a new class of problem. It scales poorly if bulk return submission is ever used on large selections.

**Fix:** Mirror the `patchJobs.ts` prefetch — build a `Map<orderId, Job[]>` once for the orders in the batch, then pass `currentJob`/`siblingJobs` into the helper. Reasonable to defer if bulk return submits are always small; if deferred, it should be a follow-up ticket rather than left silent.

### 3. Explanatory comments present pre-commit are absent from the merged code

**[File: apps/creative-portal/api/mixtures/jobs/[jobId]/submitReturnJob/submitReturnJob.ts]**

**Function/Class:** submitReturnJob (and the equivalent block in submitReturnJobs.ts)

**Severity:** low

**Problem:** Both assign blocks were written with comments explaining *why* `requestType` is added and immediately `omit`ted (it gates the helper but travels in the header on these routes, unlike the sibling PATCH routes). The committed blobs contain neither comment. This was verified against the commit object itself, not just the diff. The cause is undetermined: `eslint --fix` + `prettier --write` provably preserve the comments when re-run against the same file, and neither the `lint-staged` config nor the repo's Claude hooks (`code-review-graph update`) rewrite source. The `lint-staged` backup stash shows the same 431/2 line counts as the final commit, meaning the comments were already gone *before* the hook's tasks ran.

**Impact:** The `requestType` round-trip is the least obvious line in the diff and reads as redundant without the rationale — a likely target for a well-meaning "cleanup" that would silently reintroduce the 400. Documentation only; no runtime effect.

**Fix:** Restore a short comment above each enrich call, e.g.:

```typescript
// OMS rejects UserAssign without a returnTime, and this auto-assign
// synthesises its own payload, so nothing upstream supplies one.
// requestType only gates the helper — it reaches OMS via the header.
```

### 4. Pre-existing: the enricher's 409 message describes a condition it does not detect

**[File: apps/creative-portal/api/utils/jobs/enrichUserAssignPayload.ts]**

**Function/Class:** enrichUserAssignPayload

**Severity:** low (pre-existing; out of this PR's scope)

**Problem:** The helper throws `409 "Cannot assign — previous job has no scheduled return time."` when `computeReturnTime` returns `{ok: false}`. But `computeReturnTime` only returns `ok: false` for `reason: "no-anchor"`, which fires solely when `jobIndex < 0` — i.e. the job was not found in its own sibling list. A previous job with no `returnTime` does **not** trigger it: `getPreviousJobAnchor` falls back to `previousJob.maxReturnTime`, and a first-in-sequence job anchors on `now`.

**Impact:** The message describes a scenario that cannot produce it, and hides the one that can. This actively misled the author of this PR into flagging a false pre-merge risk (see Recommendation). Anyone debugging a real 409 here will look at the wrong job.

**Fix:** Out of scope for PP-1990. Worth a follow-up ticket to reword to something like "Cannot assign — job not found in its order's job list." and surface `reason`.

---

## Validation Checks

Run against commit `8b4d1c3cd` with a clean working tree in place on the PR branch. Results carried over from the pre-commit run on this identical commit, at the author's election — not re-executed during the review.

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ✅ | `@proofed/creative-portal` 1695/1695 pass (includes the 9 new tests). `@proofed/wysiwyg-editor` 262 pass, `@proofed/customer-portal` 336 pass. |
| `npx turbo run typecheck` | ✅ | 0 errors across all workspaces. |
| `npx turbo run lint` | ⚠️ | 0 errors in the four changed files. Repo-wide the task **fails**, on `components/molecules/JobReturnTimesTray/index.test.tsx` (5 prettier errors) — pre-existing on `develop`, untouched by this PR. |
| `npx turbo run build` | ✅ | `✓ Compiled successfully`. Sentry sourcemap upload errors are a missing local auth token, not a code failure. |

**Also pre-existing on `develop`, not caused by this PR:** `@proofed/shared` `utils/formatWordQuantity.test.ts` fails locally (`expected '10,00,000 words' to be '1,000,000 words'`) — a machine-locale artifact (en-IN digit grouping), so `npx turbo run test` is red repo-wide on this machine.

---

## Tests

- ✅ 9 tests added across the two changed routes — meets the project requirement that every PR includes tests for new code
- ✅ Tests verified **non-vacuous**: stashing the source fix and re-running fails 4 of the 9, on exactly the `returnTime`/enricher assertions; the other 5 assert unchanged behaviour and correctly stay green
- ✅ Both the fixed path (assign is enriched) and the untouched path (Active jobs skip the assign) are covered, so the tests pin the fix's boundary rather than just its happy case
- ✅ The `requestType`-out-of-body detail is asserted — the subtlety most likely to regress
- ⚠️ `enrichUserAssignPayload` is mocked in both suites, so the route↔helper integration is not exercised. Consistent with `patchJobs.test.ts`'s convention, and the helper has its own suite, but it means no test proves a real `returnTime` reaches OMS for these routes
- ⚠️ No test covers the query-`jobId`-vs-body-`id` divergence in Issue 1 (the mock request uses matching ids, so the bug is invisible to the suite)
- ✅ Author attached before/after recordings to the PR demonstrating the fix end-to-end. These post-date the review's first draft and are what settle the "does it actually submit" question that the (mistaken) 409 warning raised
- ⚠️ Still unverified: whether OMS accepts the **past-dated** `returnTime` produced for a long-overdue job specifically — worth confirming against job 24458 on b2btest rather than a freshly-created job

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Fixes the root cause; the reported path should now succeed |
| Regression risk | ✅ Low — additive on one branch of two BFF routes; wire payload unchanged apart from the added field; creative-area paths provably untouched |
| Tests | ✅ Present, meaningful, verified non-vacuous |
| Code quality | ⚠️ Two inconsistencies with the sibling routes (Issues 1, 2) and missing rationale comments (Issue 3) |
| Validation suite | ⚠️ Pass for this PR's code; `lint` and `test` are red repo-wide on pre-existing `develop` failures |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve with suggestions** — the fix is correct and well-targeted; nothing here blocks merge.

**Correction to the PR description — this is the most important item.** The PR body and the Testing section warn that job 24458 may now return a 409 rather than submitting, because it is overdue with no dates. **That warning is wrong and should be struck.** Reading `computeReturnTime` shows the 409 fires only when `jobIndex < 0` (the job is absent from its own sibling list) — not when an upstream job lacks a `returnTime`, which falls back to `maxReturnTime` (or `now` for a first-in-sequence job). Job 24458 has a `maxReturnTime` (Tue 5 May, per the ticket screenshot), so it will anchor and compute a `returnTime` — clamped to that past `maxReturnTime` — and submit. The misleading error string in Issue 4 is what produced the false alarm.

The residual open question is narrower and worth QA's attention: OMS will receive a `returnTime` **in the past** for overdue jobs like this one. Whether OMS accepts that, or rejects it with a different validation error, is not knowable from this repo and is exactly what the b2btest re-run needs to establish.

Action items:

1. **Strike the 409 warning from the PR description** and the Jira ticket if it was repeated there — it will otherwise send QA looking for a failure mode that cannot occur.
2. **Fix Issue 1** before merge — a one-line change (`jobId: jobUpdateData.id`) that also makes the two routes consistent with each other. Cheap, and removes a silent wrong-write path.
3. **Restore the Issue 3 comments** — the `requestType` round-trip needs its rationale in the code.
4. **QA on b2btest against job 24458 specifically**, watching for whether OMS accepts a past-dated `returnTime`. Cover the bulk toolbar's "Mark as Returned" too, since this PR changes it; if `getBulkActionsConfig` cannot pass an In Queue/On Hold job to that route, say so and the bulk half can be treated as a no-op.
5. **Optional follow-ups, not for this PR:** the bulk prefetch (Issue 2) and the misleading 409 message (Issue 4).

---

## Post-review update — commit `6b9267d97`

Reviewed at `8b4d1c3cd`; the following landed after the review and are reflected on the PR.

| Item | Outcome |
|---|---|
| Issue 1 — enrich the body id, not the query jobId | **Fixed.** Now `jobId: jobUpdateData.id`, matching the bulk route. Pinned by a new test (`enriches the job the PATCH targets (body id), not the query jobId`) that passes a divergent query id; verified it fails against the old code. |
| Issue 3 — restore rationale comments | **Fixed.** Comments restored on both assign blocks and confirmed present in the committed blob this time. The earlier disappearance remains unexplained but did not recur. |
| PR description's 409 warning | **Struck**, with the correction and the real residual question (past-dated `returnTime`) recorded in its place. |
| Issue 2 — bulk prefetch | **Deferred**, now stated explicitly in the PR description rather than left silent. |
| Issue 4 — misleading 409 message | **Deferred** to a follow-up ticket (pre-existing, out of scope). |

Post-fix validation: `@proofed/creative-portal` 1696/1696 tests pass, typecheck 0 errors. The pre-existing repo-wide `lint`/`test` failures on `develop` are unchanged and unrelated.
