# PR Review: feature/PP-1642 — Server-side returnTime enrichment + candidate-accept guard + Req 8 validations

**PR:** https://github.com/Proofed/B2BWebserver/pull/2305

**Jira:** https://proofed.atlassian.net/browse/PP-1642

**Status:** Code Review

> **Base note:** PR targets `feature/PP-1643-dynamic-return-times` (PR #2302), not `develop`. Diff was reviewed against that base. Re-evaluate after #2302 merges and GitHub auto-retargets.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Req 1 — Read mode display (`returnWindowsMinutes`, `maxReturnTime`, `returnTime`) | Handled in PP-1643 (parent branch). Not in this PR scope. | ✅ (out of scope) |
| Req 2 — Edit mode datetime pickers for Latest Return + Deadline | `JobCard.tsx` Job DD + User DD pickers (PP-1643). Mode toggle (Target/Shift) split in this PR. | ✅ |
| Req 3 — Manual time edits (no user change) → PUT with `returnWindowsMinutes`, `maxReturnTime`, `returnTime` | `useUpdateJobReturn.updateMaxReturnTime` mutates via `mutatePartialJob` (PUT). Cascade in Shift mode also fires PUTs. | ✅ |
| Req 4 — Standard assignment: compute `returnTime = MIN(anchor + window, maxReturnTime)` | `computeReturnTime` in `dynamicReturnTimes.ts` with first-job/prev-returnTime/prev-maxReturnTime branches. Server-side via `enrichUserAssignPayload`. | ✅ |
| Req 4.2 — Participating job rule (`Deleted != true && Status != Canceled`) | `isParticipating` filters `Canceled`. Comment notes OMS hard-purges deletions. | ✅ |
| Req 5 — Manual override: use admin-picked `returnTime` clamped at `maxReturnTime` | `override` branch in `computeReturnTime` (clamped). UI picker doesn't yet wire `override` through; production callers omit it. Acceptable since spec says future picker UI. | ⚠️ Partial (override path exists, no UI uses it yet) |
| Req 6 — Reassignment: preserve existing `returnTime` | `enrichUserAssignPayload` short-circuits when `job.returnTime` exists. `assignJobWithAcceptance` captures pre-`In Queue`-transition value from client cache and forwards as `payload.returnTime`. | ✅ |
| Req 6.3 — Backend clears `returnTime` on `onHold`/`inQueue` transition | Compensated by client capture before the `In Queue` PATCH. | ✅ |
| Req 7 — Candidate accept: PUT `returnTime` before accept; payload carries no timing | `postAcceptJob` resolves a fresh `returnTime` via `computeReturnTime`, passes it as the `returnTime` header on the candidate accept PUT (per §26.2). Body stays empty. | ✅ (note: passes as **header**, not via PUT body — see Architecture Analysis) |
| Req 8.1 — `returnTime <= maxReturnTime` | `useUpdateJobReturn.updateReturnTime` hard-guard + `JobCard.userPickerError` inline. | ✅ |
| Req 8.2 — `previousJob.maxReturnTime < currentJob.maxReturnTime` | `getUpstreamJobConstraint` + `updateMaxReturnTime` guard + `JobCard.jobPickerError` inline. | ✅ |
| Req 8.3 — `currentJob.maxReturnTime < nextJob.maxReturnTime` | `getNextJobMaxConstraint` + `updateMaxReturnTime` Target-mode guard + inline error. Shift mode skips intentionally (cascades). | ✅ |
| Req 8.4 — Previous-anchor unresolvable → block + error before OMS | `computeReturnTime` returns `{ ok: false, reason: "no-anchor" }` when `jobIndex < 0`; `enrichUserAssignPayload` / `postAcceptJob` throw `ApiError(409)`. | ✅ |
| Req 8 (additional) — order-deadline warning | Warning modal in `updateMaxReturnTime` when `picked > order.dueDateTime`. | ✅ |

---

## Architecture Analysis

The PR funnels every assignment entry point (AssignmentModal, accept-in-queue, accept-offered admin path, bulk accept-all) through one server-side helper, `enrichUserAssignPayload`, so the `returnTime` calculation is colocated with the OMS write rather than duplicated across React entry points. This avoids the classic risk of "one path forgets to compute the field." Three subtle design decisions stand out:

1. **Reassignment uses a client-side `preservedReturnTime` hop, not a server-side read.** OMS clears `returnTime` on the intermediate `In Queue` transition (TECHSVC-213) before the `UserAssign` PATCH, so the server-side enricher can't read it from a refetched job. The chosen workaround is: `assignJobWithAcceptance` reads `returnTime` from the client-cached `jobs` array *before* dispatching the status PATCH, then forwards it as `payload.returnTime` on the `UserAssign`. The enricher's "caller-supplied wins" early-return then preserves it. Functionally correct, but the captured value is only as fresh as the client cache; see Issue #1.

2. **Candidate accept passes `returnTime` as an HTTP header, not in the PUT body.** Per OMS §26.2 ("Request Schema: None"), candidate accept doesn't accept body fields. The implementation sends the recomputed timestamp via the `returnTime` header on the existing axios PUT to `/jobCandidate/:id`. **The PR description says "PUT before accept" but the implementation actually folds the value into the accept request itself.** This diverges from Jira Req 7.2's literal "persist it via OMS Job PUT before calling candidate accept" — but the comment in `postAcceptJob.ts` cites a later spec revision. Worth confirming with backend (see Issue #4).

3. **Bulk route pre-fetches sibling lists per unique `orderId` to kill N+1.** Good optimization but it fetches *all* orderIds in the batch even when only some are `UserAssign` (see Issue #2).

The `dynamicReturnTimes.ts` module is a clean extraction — pure helpers, deterministic, well-tested (20 cases across the test file). The discriminated-union return type for `computeReturnTime` is the right call for surfacing the "no anchor" failure to callers.

---

## Issues Found

### 1. Reassignment `preservedReturnTime` trusts client cache freshness

**[File: apps/creative-portal/contexts/orderSidebar/provider.tsx]**

**Function/Class:** `assignJobWithAcceptance`

**Severity:** medium

**Problem:** `preservedReturnTime` is read from the React Query `jobs` cache before the `In Queue` PATCH fires. If the cache is stale relative to OMS (e.g., another tab/admin reassigned in between, a background refetch hasn't completed, or the user opened the modal minutes ago), the forwarded `returnTime` may not match the value OMS had at capture time. Because the enricher trusts caller-supplied `returnTime` (line 58 of `enrichUserAssignPayload.ts`: `if (payload.returnTime) return payload;`), the wrong deadline can be committed without recomputation.

**Impact:** Edge-case correctness regression for reassignment under cache-staleness conditions — likely rare, but the symptom (silently wrong deadline) is hard to detect downstream because no validation runs against the source value.

**Fix:** Either (a) refetch the job once just before capture so `preservedReturnTime` matches OMS truth, or (b) move the capture server-side: in `enrichUserAssignPayload`, when `payload.requestType === "UserAssign"` and there is no `payload.returnTime`, do the `In Queue` status PATCH *inside* the same request after capturing the returnTime from `currentJob` (which the server fetched). Option (b) eliminates the round-trip race entirely and is the cleaner design — but it changes the route surface, so option (a) is the safer near-term fix:

```typescript
// In assignJobWithAcceptance, replace the cache lookup with a fresh fetch:
const freshJob = await fetchJob(jobId); // pre-status-transition state
const preservedReturnTime = freshJob?.returnTime;
```

### 2. Bulk patch fetches sibling lists for orderIds with no UserAssign

**[File: apps/creative-portal/api/mixtures/jobs/patchJobs/patchJobs.ts]**

**Function/Class:** `patchJobs` (lines 40–62)

**Severity:** low

**Problem:** The pre-fetch gate (`hasAssignment`) is batch-wide, but `uniqueOrderIds` covers *every* orderId in the batch. For a mixed batch like `[{UserAssign, orderA}, {Status, orderB}]`, we fetch siblings for both `orderA` and `orderB` even though only `orderA` needs it.

**Impact:** Extra OMS round-trip per mixed batch. Not a correctness issue; performance only.

**Fix:** Filter the orderIds by request type before fetching:

```typescript
const assignOrderIds = [
  ...new Set(
    jobs
      .filter((job) => job.requestType === "UserAssign")
      .map((job) => String(job.orderId))
  )
];
const orderJobsEntries = await Promise.all(
  assignOrderIds.map(async (orderId) =>
    [orderId, await fetchJobsByOrderId(orderId, requesterId)] as const
  )
);
```

### 3. `postAcceptJob` always recomputes even when the job has a valid `returnTime`

**[File: apps/creative-portal/api/utils/jobs/postAcceptJob.ts]**

**Function/Class:** `resolveReturnTimeForAccept`

**Severity:** low

**Problem:** Req 7.2 reads: *"If `returnTime` is `NULL`, calculate it … and persist it via OMS Job PUT before calling candidate accept."* The implementation always recomputes (and overrides) regardless of the existing value. The intentional comment ("any pre-existing returnTime from a prior offer cycle is stale") explains the design choice and the test `recomputes returnTime fresh even when the job already has one (candidate's window starts now)` enforces it, but this is a divergence from the literal Jira wording.

**Impact:** If a creative has already accepted-but-not-started a job (status transitioning between Offered → Active with a stored returnTime), accepting again would reset the deadline. In practice the candidate-accept flow only fires when the job is offered to a new candidate, so the stored `returnTime` *is* stale and the recomputation is correct — but the requirement and the implementation drifted, so the deviation should be confirmed with product.

**Fix:** No code change; confirm with PM/QA that "fresh window per candidate accept" is the intended behavior, then update the Jira requirement wording to match. If not, gate the recomputation on `!job.returnTime`.

### 4. Candidate accept ships `returnTime` as a header, not in a separate PUT

**[File: apps/creative-portal/api/utils/jobs/postAcceptJob.ts]**

**Function/Class:** `postAcceptJob` (lines 119–129)

**Severity:** medium

**Problem:** Req 7.2 says "persist it via OMS Job PUT before calling candidate accept." The implementation sends the value as the `returnTime` HTTP header on the candidate accept PUT itself — not as a separate Job PUT. The comment cites OMS §26.2 as the authority for this header pattern, but the PR description still says "PUT before accept." The PR description's manual checklist also describes "two network calls (PUT then accept PUT)" — actual behavior is one call.

**Impact:** If OMS hasn't actually been updated to accept the `returnTime` header on §26.2, the accept silently drops it and the job lands with `returnTime: null`. The PR description itself flags this risk: *"OMS rev 3.18 (03/03/26) does not yet list `returnTime` in §24.3 PATCH for `UserAssign`. The PP-1642 spec (rev 25/03/26) assumes a newer OMS revision."*

**Fix:** Before merging, confirm with backend that OMS reads the `returnTime` header on candidate accept (`PUT /jobCandidate/:id`). If not, add the explicit PUT to `/jobs/:id` first as Req 7.2 reads. Update the PR description's manual checklist to reflect actual network behavior either way.

### 5. `assignJobWithAcceptance` reassignment path is untested

**[File: apps/creative-portal/contexts/orderSidebar/provider.tsx]**

**Function/Class:** `assignJobWithAcceptance` (lines 224–268)

**Severity:** low

**Problem:** The reassignment-with-preservation logic is the most subtle piece of client-side timing handling in this PR (status PATCH → returnTime capture → UserAssign PATCH ordering). There are no unit tests for it; the orderSidebar provider has no `.test.ts` sibling.

**Impact:** A future refactor (reordering the status PATCH and the UserAssign, dropping the cache lookup, etc.) could silently break Req 6.1 with no test failure. The bug from Issue #1 would also be caught earlier with a test that asserts the captured timestamp matches the pre-transition cache snapshot.

**Fix:** Add a `provider.test.tsx` covering: (a) reassigning an Active job preserves `returnTime` in the final UserAssign payload, (b) reassigning an In Queue job (no transition) passes through correctly, (c) assigning an unassigned job sends no returnTime (server computes).

### 6. Bulk patch route has no integration test for the enrichment

**[File: apps/creative-portal/api/mixtures/jobs/patchJobs/patchJobs.ts]**

**Function/Class:** `patchJobs`

**Severity:** low

**Problem:** `enrichUserAssignPayload.test.ts` covers the helper in isolation but there is no test for the bulk route's pre-fetch+pass-through wiring: that one `fetchJobsByOrderId` fires per unique `orderId`, that pre-fetched siblings are reused (no extra fetch inside the helper), and that mixed batches (some UserAssign, some Status) don't crash. The unit tests for the helper assert `mockFetchJobs).toHaveBeenCalledTimes(1)` for the single-call case but never exercise the bulk route end-to-end.

**Impact:** Refactoring `patchJobs.ts` (e.g., changing the Map key shape from `String(orderId)` to a number, or extracting the pre-fetch into a util) could silently regress the N+1 protection.

**Fix:** Add a `patchJobs.test.ts` mirroring the style of `patchJob.test.ts`. At minimum: (a) batch of 3 jobs across 2 orderIds with UserAssign → assert `fetchJobsByOrderId` called twice (one per unique order), `enrichUserAssignPayload`'s internal `fetchJobsByOrderId` *not* called, (b) batch of 1 UserAssign + 1 Status → assert no extra fetch for the Status entry.

### 7. `getUpstreamJobConstraint` doesn't fall through to `returnTime` when the previous job is assigned

**[File: apps/creative-portal/api/utils/jobs/dynamicReturnTimes.ts]**

**Function/Class:** `getUpstreamJobConstraint` (lines 137–146)

**Severity:** low

**Problem:** The helper always returns `parseUtcDateString(prevJob.maxReturnTime)` as the floor. The User DD picker uses a *different* helper (`getUpstreamReturnTimeConstraint`) that respects the previous job's `returnTime`/`completedDatetime`. But the Job DD picker's upstream guard goes through `getUpstreamJobConstraint`, so when the previous job is already assigned with `returnTime` *before* its `maxReturnTime`, this job's Job DD can still be set tighter than the previous job's actual deadline, as long as it sits after `prevJob.maxReturnTime` (which is later). Conversely, the picker correctly blocks pushing this job's Job DD into the previous job's hard-stop range. So the existing logic matches Req 8.2's literal wording (`previousJob.maxReturnTime < currentJob.maxReturnTime`) but there is no symmetric guard for "previousJob's *committed* returnTime < currentJob.maxReturnTime" — though in practice `prevJob.maxReturnTime >= prevJob.returnTime` so this is rarely binding.

**Impact:** Theoretical edge case. Unlikely to fire in production data.

**Fix:** No change required if the team is satisfied with Req 8.2 as literally written. If a tighter floor is desired, extend `getUpstreamJobConstraint` to take the max of `prevJob.maxReturnTime` and (if present) `prevJob.returnTime`.

### 8. PR description's manual checklist claims "two network calls" for candidate accept — mismatch with implementation

**[File: PR description]**

**Function/Class:** N/A (documentation)

**Severity:** low

**Problem:** The PR body says: *"Accept an offered job with `returnTime: null` as a creative → confirm two network calls (PUT then accept PUT), UI shows computed deadline."* But the actual implementation issues one call: the candidate accept PUT with `returnTime` as a header (see Issue #4).

**Impact:** Reviewers / QA will look for the wrong DevTools signature. Will block the manual-testing checkbox unnecessarily.

**Fix:** Update the PR body to: *"… confirm one network call (candidate accept PUT) with a `returnTime` request header in the proper ISO format, UI shows computed deadline."*

---

## Tests

- ✅ 27 new Vitest cases across `dynamicReturnTimes.test.ts`, `enrichUserAssignPayload.test.ts`, `postAcceptJob.test.ts`, `useUpdateJobReturn.test.ts`
- ✅ `computeReturnTime` edge cases covered: first job, prev returnTime, prev maxReturnTime, cancelled prev, all-cancelled, jobIndex=-1 no-anchor, override clamp
- ✅ `enrichUserAssignPayload` covers: non-UserAssign passthrough, caller-supplied returnTime passthrough, reassignment preservation, fresh compute, 409 no-anchor, `currentJob` short-circuit
- ✅ `postAcceptJob` covers: candidate-missing 409, race-loss 409, fresh recompute, orderId-fast-path, mismatched orderId 409, no-anchor 409
- ✅ `useUpdateJobReturn`: User DD > maxReturnTime block, sequence violation block, Job DD ≤ prev.max block, deadline warning modal + callback, normal path, cascade ceiling, locked-User DD floor
- ⚠️ No test for `assignJobWithAcceptance` reassignment flow in `contexts/orderSidebar/provider.tsx` — see Issue #5
- ⚠️ No integration test for the bulk `patchJobs` route's pre-fetch wiring — see Issue #6
- ⚠️ `updateReturnWindow`'s `Math.round` fix (commit 6337ce88f) has no regression test
- ✅ Pre-existing tests still pass per PR description (1063/1063 — verified vs CI before merge)

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ Mostly correct; reassignment path has a cache-staleness edge case (Issue #1); candidate-accept design needs backend confirmation (Issue #4) |
| Regression risk | ⚠️ Medium — touches every assignment entry point (single PATCH, bulk PATCH, candidate accept, accept-in-queue, accept-offered, accept-all). Helper extraction kills the N+1 but adds a shared failure point |
| Tests | ⚠️ Solid unit coverage for the new helpers; gaps for the provider reassignment flow and the bulk route's pre-fetch wiring |
| Code quality | ✅ Pure helpers, discriminated-union returns, clear comments — well-structured |
| Mergeable state | ✅ Clean (per GitHub) |

---

## Recommendation

**Approve with suggestions** (gate on Issues #1 and #4 before merging to `develop`)

1. **Block on Issue #4**: confirm with backend that OMS reads the `returnTime` header on §26.2 candidate accept. If it doesn't, add the explicit Job PUT before the candidate accept PUT per Req 7.2 literal wording.
2. **Address Issue #1**: either move the `preservedReturnTime` capture server-side, or add a fresh `fetchJob` before the In Queue transition in `assignJobWithAcceptance`. The current client-cache read is a latent race-condition bug for reassignment.
3. **Update the PR body's manual-testing checklist** to reflect the actual single-call candidate accept (Issue #8) so QA looks for the right DevTools signature.
4. **Optional cleanups (post-merge OK)**: add the missing tests called out in Issues #5 and #6; tighten the bulk pre-fetch filter per Issue #2.
5. **Confirm with product (Issue #3)**: "always recompute on candidate accept" diverges from the literal Jira wording. Update either the spec or the code so they match.
