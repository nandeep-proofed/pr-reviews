# PR Review: feature/PP-2095: Resolve side panel files to the source job's copy

**PR:** https://github.com/Proofed/B2BWebserver/pull/2453
**Jira:** https://proofed.atlassian.net/browse/PP-2095
**Status:** Jira not reachable in this session — the Atlassian MCP server is unauthenticated, so the requirement table below is built from the requirement numbers quoted in the PR body and in the code's own docblocks, **not** from the ticket. Re-run with Jira authorised before treating the mapping as verified.
**Base:** `develop` · **Reviewed at** `b5cfab1ee` (1 commit, 40 files, +2226 / −75)
**Scope:** requirements 1–5. Requirement 6 is PR #2463, which is stacked on this branch.

> **Note on the stack.** PR #2463 (reviewed separately) contains fixes to code introduced here — `getAiEditKind`'s legacy classification, `findLatestCompletedJob`'s ordering, the `FilesSection` reviewer credit, and a `resolveFrom` discriminator on the resolver. Issues 1, 2 and 3 below are **already fixed on #2463**. They are still reported against this PR because this PR is the one being merged into `develop` first: if #2453 lands alone, or if #2463 is reworked, these defects ship.

---

## What this means for users (non-technical summary)

1. **On the admin order panel, the newest edited copy can still be missed.** The panel is supposed to offer the most recent finished version of a customer's document. Because of how it asks for that version, when the AI pass is the *last* step on the order its output is skipped and the panel falls back to the untouched upload — the exact problem this ticket set out to fix, just in the one panel rather than both.
2. **On older orders, a file can be labelled as the wrong kind of edit.** Some jobs created before a naming change don't say which of the two AI passes they were. Those are all treated as the first pass, so a document produced by the *second* pass gets headed "Pre-edited" — naming a step that never ran. The same mislabel appears as a tag on the dashboard row.
3. **The "Reviewed by" / "QA'd by" credit can disappear from the Files section.** When the edited copy and the resolved copy turn out to be the same file, one of the two rows is correctly removed — but the person's name goes with it, so nobody is credited for work that was done.
4. **The displayed filename can say "Original" for a file that is not the original.** If the AI pass didn't record what kind of version it stored, the name shown in the panel gets "Original" stitched into it while the heading above says "Pre-edited". The file you actually download is named correctly; only the label on screen disagrees.
5. **The jobs dashboard makes one extra request per order on screen.** Not visible as an error, but it adds load time on the page creatives use most, and it grows with the number of orders listed.

---

## Jira Requirements vs Implementation

Requirement numbering taken from the PR body and the code's docblocks; **not** verified against the ticket.

| Jira Requirement | PR Implementation | Status |
| --- | --- | --- |
| Req 1.1 / 1.2 — the order panel offers the order's newest completed copy | `findLatestCompletedJob` written and documented for exactly this, but the order panel passes `jobId: order?.currentJobId`, so the preceding-job resolver runs instead and the function is never reached | ❌ Missing (Issue 1) |
| Req 2.1 — the job panel starts from the copy the preceding job produced | `findPrecedingCompletedJob` walks the order's `jobSequence` back to the nearest completed job; wired from `JobManagement` via `jobId={job.id}` | ✅ Addressed |
| Req 2.4 / validation rule 4 — both panels offer the same version and head it identically | One hook, `useSourceJobWorkItemVersion`, serves both — but the two panels pass different `jobId` semantics, so "by construction" does not hold in practice (see Issue 1) | ⚠️ Partial |
| Req 3 — front-end verification of PP-1873's stored versions | `useSearchWorkItemContentVersion` by `JobID`, with `pickProducedContentVersions` dropping the seeded input | ⚠️ Partial — author states it is **not yet proven on live data** |
| Req 3.3 / validation rule 7 — a failed AI pass neither offers its version nor suppresses the last good one | `isAiJobFailed` excluded from `isCompletedJob`, so the walk steps over it | ✅ Addressed |
| Req 4.2 — the sparkle replaces the "Original" heading on a resolved pass | `createFileItem` attaches `aiEditKind` to the primary slot; `FilePill` renders the indicator *or* the title, never both | ✅ Addressed |
| Req 4.3 — no pre-edit decoration on the filename | No decoration is added, but the `versionType` fallback can stitch `Original` into the name of a pass's copy | ⚠️ Partial (Issue 6) — author lists this as open with the PO |
| Req 4.5 / 5.5 — no creative-facing string contains "AI" | `AI_EDIT_LABELS` are "Pre-edited" / "Post-edited"; a test asserts the term never renders | ✅ Addressed |
| Req 5.1 — one shared indicator across placements | `packages/shared/components/atoms/AiEditIndicator`, tone drives colour and sparkle size | ✅ Addressed |
| Req 5.2 / 5.4 — the indicator appears on a pre-edited order's dashboard row | `hasCompletedPreEdit(orderJobs)` in the Order ID cell, fed by a per-row `useJobsQuery` | ⚠️ Partial — correct, but see Issues 2 and 5 |
| Validation rule 1 — the superseded original is no longer offered | The resolved version takes the primary slot; duplicate rows suppressed | ✅ Addressed (job panel); ❌ not reached on the order panel (Issue 1) |
| Validation rule 12 — human-only orders stay byte-identical | The override applies only when the source job is an AI job (`isEnabled = isFileBasedWorkItemFormat && !!aiEditKind`) | ✅ Addressed |

**Scope beyond the ticket.** Three moves ride along, all declared and all clean:
`AI_JOB_FILTER_BY_PREFIX` relocated from `api/orders/utils` to `api/jobTypes/consts` (the order filters now import it — I verified the old definition is deleted, not duplicated); `pickLatestContentVersion` / `hasSubmittedContentVersion` moved to `packages/shared/utils/workItemContentVersionRules` with the old module re-exporting both, so server call sites keep their import path; and `COMPLETED_JOB_STATUSES` added to `api/jobs/consts`. No scope creep worth flagging.

---

## Architecture Analysis

The core judgement here is good, and the PR explains itself unusually well.

- **Reusing the WYSIWYG rule rather than inventing a second one** is the right call and is what the ticket's API note asks for. The decision to avoid the `versionType`-grouped search is well-founded — it drops versions with no type, and the PR states that was the observed root cause of a pre-edited order still showing `Original`.
- **`pickProducedContentVersions` is a genuine improvement** over the count-based `hasSubmittedContentVersion` it sits beside: identifying the seed by lowest id and excluding it means a pass that stored *only* track changes can't leave the seeded input masquerading as its output. That precision is what makes the "never put the seed in the primary slot" guarantee hold, and it is tested.
- **The layering is right.** `AiEditKind` lives in `packages/shared/config` rather than next to the component, so `api/` utils can import it without a component → api inversion — the same reasoning as `workItemFormat.ts`. `workItemContentVersionRules.ts` was moved out of the module that reaches `Buffer`, with a re-export left behind. Both moves are explained in place.
- **`AiEditIndicator` is a model shared atom**: folder structure, `FC<Props>`, `stopForwarding`, theme colour keys rather than raw hex, a Storybook story with a conforming title, and a test that asserts the term "AI" never reaches the DOM. The two-size decision is derived from the Figma geometry and shown as arithmetic in a comment.
- **Where it falls down is the seam between the resolver and its two callers.** `useSourceJobWorkItemVersion` infers *which question to ask* from whether a `jobId` was passed. That is a silent switch: the order panel legitimately knows a job id, passes it, and silently gets the wrong mode. The function written for the order panel ends up unreachable and untested. The author reached the same conclusion on #2463 and replaced the inference with an explicit `resolveFrom` discriminator — which is the correct fix, and is the shape this PR should have had.

---

## Issues Found

### 1. The order panel asks the wrong question, and the function written to answer it is never reached

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/index.tsx]**

> **In plain terms:** The admin order panel is meant to show the newest finished version of the customer's document. Instead it looks at the step *before* the one the order is currently on. When the AI edit is the last step on the order, that means its output is skipped and the panel falls back to the customer's untouched upload — the very problem this ticket exists to fix.

**Function/Class:** `OrderManagement` → `useSourceJobWorkItemVersion`

**Severity:** high

**Confidence:** high

**Steps to reproduce:**

1. Sign in to the admin area and find a file-based order (DOCX/PDF/PPTX/XLSX) whose **final** job is a completed AI Post-edit — i.e. the order's current job is that AI job.
2. Open the order side panel and expand **Files**.
3. **Expected:** the primary file row is the AI pass's output, headed with the sparkle and "Post-edited" (req 1.1 / 1.2, validation rule 1).
4. **Actual:** the primary row is the customer's original upload under the plain "Original" heading. The pass's output is not offered.

**Problem:** The resolver picks its mode by inspecting whether a `jobId` was supplied:

```typescript
const sourceJob = jobId
  ? findPrecedingCompletedJob({ jobId, jobSequence, jobs })
  : findLatestCompletedJob({ jobSequence, jobs });
```

The order panel supplies one:

```typescript
} = useSourceJobWorkItemVersion({
  isFileBasedWorkItemFormat,
  jobId: order?.currentJobId,
  jobSequence: order?.jobSequence,
  jobs
});
```

so the `findLatestCompletedJob` branch is taken by neither caller — the job panel passes `jobId={job.id}` unconditionally, and the order panel passes `order.currentJobId`. `findLatestCompletedJob`'s own docblock says this is exactly wrong: *"if the order's current job is itself a completed AI Post-edit, its output IS the latest state of the document and must be offered (PP-2095 req 1.1, 1.2). Walking back from `currentJobId` would skip exactly that job."*

**Evidence:**

- `apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/index.tsx:92` — `jobId: order?.currentJobId`
- `apps/creative-portal/hooks/useSourceJobWorkItemVersion.ts:70-72` — the ternary above
- `apps/creative-portal/utils/findCompletedJob.ts:61-70` — the docblock that contradicts the wiring
- `apps/creative-portal/components/organisms/sidebars/contents/JobManagement/index.tsx:217` — `<JobFiles {...{ order }} jobId={job.id} isInitialOpen />`, so the job panel always passes an id too
- A repo-wide grep for `findLatestCompletedJob` returns only its definition and the unreachable ternary arm — no other caller, and **no test**

Concrete path: for an order with `jobSequence = "10,20"` where job 10 is a completed human job, job 20 is a completed AI Post-edit, and `currentJobId = 20` — `findPrecedingCompletedJob({ jobId: 20 })` slices to `[10]`, returns the human job, `getAiEditKind` returns `undefined`, `isEnabled` is false, and the hook returns `{ isLoading: false }` with no resolved version.

The author additionally reports (on #2463, and pins it with a test there) that OMS sets `currentJobId` to `-1` on a closed order — in which case `findIndex` returns `-1`, the `currentIndex <= 0` guard fires, and every closed file-based order falls back to the raw upload. I have not verified the OMS behaviour myself, so treat that as a second, broader trigger rather than a confirmed one.

**Impact:** Requirements 1.1 and 1.2 are not delivered on the order panel, and validation rule 1 (the superseded original must no longer be offered) fails there. Req 2.4 / validation rule 4 — "both panels offer the same version" — is asserted to hold "by construction" but does not: the two panels ask different questions through the same hook.

**Fix:** State the mode instead of inferring it — which is what #2463 does. Take that change:

```typescript
export enum SourceJobResolution {
  PrecedingJob = "precedingJob",
  LatestCompletedJob = "latestCompletedJob"
}
```

```typescript
const sourceJob =
  resolveFrom === SourceJobResolution.PrecedingJob
    ? findPrecedingCompletedJob({ jobId, jobSequence, jobs })
    : findLatestCompletedJob({ jobSequence, jobs });
```

with `resolveFrom` **required**, `OrderManagement` passing `LatestCompletedJob` and dropping `jobId` entirely, and `useJobFiles` passing `PrecedingJob`. Add the `findLatestCompletedJob` tests this PR is missing (Issue 4) before relying on that branch — as written it has ordering bugs of its own that #2463 also fixes.

---

### 2. A legacy bare-"AI" job is always labelled "Pre-edited", whichever pass it was

**[File: apps/creative-portal/api/jobTypes/consts.ts]**

> **In plain terms:** Some AI jobs created before a naming change don't record which of the two passes they were — they just say "AI". Every one of those is treated as the *first* pass. So on an older order, a document produced by the second pass is headed "Pre-edited", naming a step that never ran, and the same wrong tag shows on the dashboard row.

**Function/Class:** `AI_JOB_FILTER_BY_PREFIX` / `getAiJobFilterValue`, consumed by `getAiEditKind`

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. Find a file-based order (created before PP-1938) whose AI job carries the bare description `"AI"` and which ran **after** the human service job — i.e. it is a Post-edit.
2. Open the job panel for the following job, or the order's row on the jobs dashboard.
3. **Expected:** either "Post-edited", or no pass label at all if the pass cannot be determined.
4. **Actual:** the sparkle reads **"Pre-edited"** in the Files section, and the dashboard row carries the "Pre-edited" chip.

**Problem:** The lookup table folds the legacy spelling into the pre-edit bucket:

```typescript
export const AI_JOB_FILTER_BY_PREFIX: Record<
  string,
  AiJobFilterValue
> = {
  [JobType.AI.toLowerCase()]: AiJobFilterValue.PRE_EDIT,   // <-- bare "AI"
  ...
};
```

The docblock explains why — *"mirroring the client's `migrateLegacyJobFilter` so the two sides can't disagree"* — and that is sound for a **filter**, where a widened bucket only broadens a search. But `getAiEditKind` reuses the same table to produce a string a creative reads:

```typescript
const filterValue = getAiJobFilterValue(job.description);

if (filterValue === AiJobFilterValue.PRE_EDIT) {
  return AiEditKind.PreEdit;
}
```

A filter default and a user-facing label are different questions, and one table answers both.

**Evidence:** `apps/creative-portal/api/jobTypes/consts.ts:16` is the offending mapping; `apps/creative-portal/api/jobs/utils.ts:46-50` is the consumer that turns it into a label; `packages/shared/config/aiEdit.ts:20` is where `PreEdit` becomes the literal string `"Pre-edited"`. The same value reaches the dashboard through `hasCompletedPreEdit` (`api/jobs/utils.ts:63-67`) → `JobTable/consts.tsx:467`.

That legacy jobs are real is evidenced inside this repo: `api/wysiwyg/createDocumentAfterAiJobSubmission/createDocumentAfterAiJobSubmission.ts:93-103` carries a dedicated fallback for exactly this case, classifying an AI job with no explicit description by its id relative to the service job.

**Impact:** A creative is told which AI pass produced the document they are about to edit, and on legacy orders that statement can be wrong. Requirement 4.5 / 5.5's intent — the sparkle carries the meaning "AI" so the label must be accurate — is undermined.

**Fix:** Split the two questions, as #2463 does. Keep `getAiJobFilterValue` (filters, legacy alias included) and add a stricter variant for anything user-facing, then fall back to position relative to the service job, mirroring the WYSIWYG path:

```typescript
export const getExplicitAiJobFilterValue = (
  description?: string
): AiJobFilterValue | undefined => {
  const filterValue = getAiJobFilterValue(description);

  return filterValue &&
    description?.trim().toLowerCase() !== JobType.AI.toLowerCase()
    ? filterValue
    : undefined;
};
```

Note that #2463's version of `getAiEditKind` takes the order's jobs as a second argument for the positional fallback, which changes the signature — so this fix and Issue 1's fix are best taken together from that branch.

---

### 3. Reviewer / QA credit disappears when the duplicate file row is dropped

**[File: packages/shared/components/organisms/FilesSection/hooks.ts]**

> **In plain terms:** When the edited copy and the resolved copy turn out to be the same file, the Files section correctly shows it once instead of twice — but the "Reviewed by …" / "QA'd by …." line vanishes with the removed row. Work that was done ends up credited to nobody.

**Function/Class:** `useFilesSection` → `createFileItem`

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. Find a file-based order with a review job assigned to a named reviewer, where a completed AI pass's output is also the version the order records as its completed one.
2. Open the order side panel and expand **Files**.
3. **Expected:** one file row, headed with the sparkle, carrying "Reviewed by &lt;name&gt;".
4. **Actual:** one file row headed with the sparkle, and no reviewer or QA attribution anywhere in the section.

**Problem:** De-duplication disables the completed row's fetch:

```typescript
const isCompletedSameAsPrimary =
  !!completedWorkItemVersionId &&
  completedWorkItemVersionId === primaryWorkItemVersionId;
```

```typescript
enabled:
  Boolean(completedWorkItemVersionId) &&
  !isCompletedSameAsPrimary
```

so `completedVersion` is `undefined` and `createFileItem` returns `null` for that slot. The surviving row is built with `type: "original"`, and `createFileItem` withholds attribution from exactly that type:

```typescript
...(reviewerName &&
  !qaName &&
  type !== "original" && { reviewerName }),
...(qaName && type !== "original" && { qaName }),
```

The `type !== "original"` rule is right in general — the customer's untouched upload should never be credited — but it no longer holds once the primary slot has been re-pointed at a copy somebody produced.

**Evidence:** `packages/shared/components/organisms/FilesSection/hooks.ts:33-35` and `:70-74`; `packages/shared/components/organisms/FilesSection/utils.ts:106-109`. Reachable only from the creative portal's order panel, which is the sole caller that passes `reviewerName` / `qaName` (`OrderManagment/index.tsx:384-393`) — the customer portal passes neither (`OrderTable/partials/OrderAndBriefDetails/utils.tsx:141-147`), so there is no cross-app impact.

**Impact:** Attribution silently disappears from the Files section on the affected orders. No error, nothing to indicate the information was dropped rather than absent.

**Fix:** #2463's `representsCompletedWork` flag is the right shape — the primary slot declares that it stands in for the completed work, and `createFileItem` keys the credit off that instead of off `type`:

```typescript
const showsReviewerCredit =
  representsCompletedWork ?? type !== FileContentType.ORIGINAL;
```

---

### 4. `findLatestCompletedJob` has no tests, and no test covers the order panel's resolution

**[File: apps/creative-portal/utils/findCompletedJob.test.ts]**

> **In plain terms:** One of the two ways the panel can look up a document version is completely untested, and so is the panel that was supposed to use it. That is why the wiring mistake above shipped: nothing anywhere asserts what the admin order panel resolves to.

**Function/Class:** the test file's two `describe` blocks — both for `findPrecedingCompletedJob`

**Severity:** medium

**Confidence:** high

**How it is spotted:** Code health, not user-reproducible. `findCompletedJob.test.ts` contains `describe("findPrecedingCompletedJob - ticket order shapes")` and `describe("findPrecedingCompletedJob")` and nothing else; `useSourceJobWorkItemVersion.test.ts` has no case that omits `jobId`; there is no `OrderManagment` test covering the Files resolution.

**Problem:** Three gaps, each of which would have caught a defect above:

- **`findLatestCompletedJob`: zero cases.** The function is exported, documented, and — as Issue 1 establishes — currently unreachable. Nothing would have flagged either fact. It also has ordering bugs of its own that only surfaced on #2463: `sortJobsByJobSequence` parks jobs absent from the sequence at `Number.MAX_SAFE_INTEGER`, so `.reverse()` lets a stray row out-rank the real final pass, and with no `jobSequence` at all it ranks by job *type*, which always puts AI first and therefore can never see a Post-edit as the latest.
- **`useSourceJobWorkItemVersion`: no case for the no-`jobId` branch.** All 13 cases pass a `jobId`, so the order-panel mode is never exercised.
- **`getAiEditKind`: no case for the legacy bare-`"AI"` description.** `api/jobs/utils.test.ts:106` covers *"returns undefined for an AI job with an unknown description"*, but bare `"AI"` is not unknown — it is a table hit, and the case that is wrong (Issue 2) is the one not written.

**Impact:** Two of the three defects in this review are in code paths with no test at all. The PR's coverage is otherwise strong, which makes the gaps easy to miss.

**Fix:** Add a `describe("findLatestCompletedJob")` block covering: the last completed job in sequence order; a completed job that is the order's current one; skipping incomplete and failed jobs; a completed job absent from `jobSequence`; and the no-`jobSequence` fallback. Add a `useSourceJobWorkItemVersion` case asserting the order-panel mode resolves the current job's own completed pass. Add `getAiEditKind` cases for a bare-`"AI"` job on each side of the service job. (#2463 contains all of these — 30 `findCompletedJob` cases and 17 hook cases — so they can be pulled back.)

---

### 5. The jobs table issues one extra jobs request per order on screen

**[File: apps/creative-portal/components/molecules/tables/JobTable/consts.tsx]**

> **In plain terms:** To decide whether to show the "Pre-edited" tag, every order row on the jobs dashboard asks the server separately for that order's list of jobs. With a full dashboard that is a stack of extra requests on the page creatives use most, and it grows as the list grows.

**Function/Class:** the Order ID column's `Cell` in `generateColumnsForFlatRows`

**Severity:** medium

**Confidence:** high

**How it is spotted:** Open the jobs dashboard with DevTools' Network tab filtered to the jobs endpoint. One request fires per distinct order id in the table, on top of the two list queries. It is not an error state, so it is invisible without looking.

**Problem:**

```typescript
const { data: orderJobs } = useJobsQuery(orderId, {
  enabled: row.original.kind === "job"
});
```

This is a per-row hook in a table cell. The hook itself is placed correctly — above the `group-header` early return, so it is unconditional and there is no rules-of-hooks violation, and the file already calls `useJobTaskQuery` / `useOrderByIdQuery` / `useMasterReferenceQuery` the same way. React Query de-duplicates by key, so it is one request per *order*, not per row. But it is still a classic N+1: one list request plus one per listed order.

The comment weighs it against the only alternative it considers — expanding the order query, which *would* be worse because `expand` fans out to a job lookup and a job-task fetch per job server-side. It does not consider reusing data the page already holds. The author later identified that option and its limit on #2463: *"The jobs page does hold an `orderJobsMap`, but it covers only Review rows (PP-1987 scoped it that way), so feeding it in here silently dropped the indicator from every other row."*

**Evidence:** `apps/creative-portal/components/molecules/tables/JobTable/consts.tsx:432-434` (the hook) and `:467-469` (the consumer, `hasCompletedPreEdit(orderJobs)`).

**Impact:** Added latency and backend load on the highest-traffic page in the creative portal, scaling with the number of orders listed. Accepted deliberately, but worth a second look before it becomes precedent — this is now the fourth per-row query in the same file.

**Fix:** Not necessarily in this PR, but worth recording as a follow-up: widen the jobs page's `orderJobsMap` beyond Review rows (PP-1987) and feed it down, so the table reads one already-fetched map instead of issuing a query per order. If the per-row fetch stays, say so explicitly in the comment — that it is N requests per page render, not just "a separate lightweight fetch".

---

### 6. The displayed filename can read "Original" for a file that is not the original

**[File: packages/shared/components/organisms/FilesSection/utils.ts]**

> **In plain terms:** If the AI pass didn't record what kind of version it saved, the name shown in the panel has the word "Original" stitched into it — while the heading directly above it says "Pre-edited". The file you actually receive when you click Download is named correctly; only the label on screen contradicts itself.

**Function/Class:** `getFileNameAndExtensionFromVersion`, called by `createFileItem`

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. Find a file-based order whose completed AI pass stored a work item content version with **no** `versionType` — the PR states this happens and cites it as the root cause of an earlier bug.
2. Open the job panel for the following job and expand **Files**.
3. **Expected:** the primary row's name reflects the resolved version, with no "Original" in it (req 4.3).
4. **Actual:** the heading shows the sparkle and "Pre-edited", while the name below reads `<orderId>_Original_<name>.<ext>`.

**Problem:** The primary slot is always built with `type: "original"`, and the filename builder falls back to that type when the version carries no `versionType`:

```typescript
fileName: getFullFileName({
  orderId,
  versionType: versionType ?? upperFirst(type),
  extractedFileName,
  extension
}),
```

`upperFirst("original")` is `"Original"`, which `getFullFileName` then joins into the middle of the name. The PR's decision note says the slot *"displays the resolved version's real name instead"* — true when `versionType` is set, which is exactly the condition the PR elsewhere says cannot be relied on.

**Evidence:** `packages/shared/components/organisms/FilesSection/utils.ts:57` (the fallback), `:27-35` (`getFullFileName` joining it in), `:75-78` (the `type` passed for the primary slot). The download itself is unaffected — `useFilesSection`'s `downloadDocument` prefers the API's `responseFileName` over the displayed name (`hooks.ts:100-105`) — so this is a display inconsistency, not a wrong file.

**Impact:** A heading and a filename that contradict each other on the same row. The author lists req 4.3 filename handling as still open with the PO, so this is a known area rather than an oversight — but it is currently resolved in the direction the requirement guards against.

**Fix:** Needs the PO decision the author has already raised. Mechanically, the cleanest option is to stop deriving the middle segment from the *slot* when the slot has been re-pointed — omit it entirely for a resolved pass rather than defaulting it to `"Original"`:

```typescript
// A resolved pass's copy is not the original upload; if the version
// did not record its own type, name it without one rather than
// stamping the slot's default into it.
versionType: versionType ?? (aiEditKind ? undefined : upperFirst(type)),
```

---

### 7. `createFileItem` mixes the `FileContentType` enum with the raw string it stands for

**[File: packages/shared/components/organisms/FilesSection/utils.ts]**

> **In plain terms:** Two lines in the same function spell the same value two different ways. Nothing behaves differently; it just makes the rule harder to find and easy to change in one place and not the other.

**Function/Class:** `createFileItem`

**Severity:** low

**Confidence:** high

**How it is spotted:** Code health, not user-reproducible. Compare `utils.ts:86` with `:108-109`.

**Problem:** The section-title branch uses the enum, the attribution branch uses a literal:

```typescript
if (type === FileContentType.ORIGINAL) {      // line 86
```

```typescript
...(reviewerName &&
  !qaName &&
  type !== "original" && { reviewerName }),   // line 108
...(qaName && type !== "original" && { qaName }),
```

The `CreateFileItemParams["type"]` union is declared as raw string literals (`types.ts:29`), so both compile — but the two spellings encode the same rule and Issue 3's fix touches precisely these lines.

**Impact:** None functional. Worth folding into the Issue 3 fix, since that rewrites both lines anyway.

**Fix:** Use `FileContentType.ORIGINAL` in both, and consider typing `CreateFileItemParams["type"]` as the enum rather than a parallel string union.

---

## Open Questions

- Does OMS ever leave `order.currentJobId` pointing at a job that has already completed, or does it always advance to the next incomplete one? The answer sets how often Issue 1 bites in practice — and the author's separate claim that a closed order carries `currentJobId: -1` would make it universal for closed orders. — `apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/index.tsx:92`
- `JobFiles` is rendered only when `isFileBasedWorkItemFormat && isActive`, so the Files section does not appear on a submitted job or in history mode. `findPrecedingCompletedJob`'s docblock explicitly reasons about *"a submitted job, or in history mode"* — is the `isActive` gate intended to stay, and if so is req 2.1 satisfied for those views by some other surface? — `apps/creative-portal/components/organisms/sidebars/contents/JobManagement/index.tsx:215`
- When the version search fails outright (network or 5xx), `useSourceJobWorkItemVersion` returns no resolved id and the panel silently falls back to the original upload, with no toast and nothing captured. Is silent fallback the intended behaviour, or should a failure to resolve be distinguishable from "no pass ran"? — `apps/creative-portal/hooks/useSourceJobWorkItemVersion.ts:85-87`
- `getAiEditKind` reads only `job.description`. Is `description` guaranteed to be the job's *name* for AI jobs across all ages of data, or can it carry free text on some orders — in which case the table lookup would miss and no pass would be reported? — `apps/creative-portal/api/jobs/utils.ts:46`
- The PR states requirement 3 is **not yet proven on live data** and that orders A–E have not been walked. Has the corrected build been re-tested against a real pre-edited order since? — PR description, "Notes for the reviewer"

---

## Validation Checks

| Check | Result | Notes |
| --- | --- | --- |
| `npx turbo run test` | ⏭️ | Skipped — user opted out |
| `npx turbo run typecheck` | ⏭️ | Skipped — user opted out |
| `npx turbo run lint` | ⏭️ | Skipped — user opted out |
| `npx turbo run build` | ⏭️ | Skipped — user opted out |

The PR reports 278 creative-portal tests passing across 21 files in an affected-area sweep, and typecheck clean on `@proofed/shared` and `@proofed/creative-portal`. None of that was re-verified here.

The PR also declares **two pre-existing failures that block the full-suite gate**, both claimed to reproduce on a clean tree with this branch stashed: the `@proofed/creative-portal` full `vitest run` hangs deterministically after `getRateUnit.test.ts`, and `@proofed/customer-portal` typecheck fails with ~149 `TS2307` SVG errors for want of a `*.svg` module declaration. **Neither claim was verified in this review.** Because this PR changes `packages/shared`, the full unscoped suite — including `@proofed/customer-portal` — is the gate that matters, and it is precisely the one reported as already red. That needs resolving as its own piece of work rather than carried as a standing exception; two permanently-failing gates mean no PR touching `packages/shared` can be validated.

---

## Tests

- ✅ Coverage is broad and behavioural: `findCompletedJob` 16 cases (including the ticket's Order A–E shapes), `useSourceJobWorkItemVersion` 13, `api/jobs/utils` 16, `workItemContentVersionRules` 6, `FilesSection` hooks 6 + utils 21, `FilePill` 11, `AiEditIndicator` 8, `JobFiles/hooks` 4, `JobTable` cell 5.
- ✅ Assertions check meaningful outcomes, not render-without-throwing — e.g. *"never puts the job's seeded input in the primary slot"*, *"never labels the original upload with a pass it did not produce"*, *"keeps the displayed name in step with the downloaded file"*, *"does not leak its styling props to the DOM"*.
- ✅ Requirement-anchored edge cases are covered where the author reasoned about them: a failed AI pass, a version with no `versionType`, a pass that stored no downloadable file, the seed identified by lowest id rather than array position, and a test asserting the term "AI" never reaches a creative-facing string.
- ✅ `pickProducedContentVersions` is cross-checked against `hasSubmittedContentVersion` — a genuinely good test, since the two answer the same question by different means.
- ❌ **`findLatestCompletedJob` has no tests at all** (Issue 4).
- ❌ **No test exercises the order panel's resolution**, nor the hook's no-`jobId` branch (Issue 4).
- ❌ **No test for the legacy bare-`"AI"` description** — the one classification that is wrong (Issues 2 and 4).
- ✅ Both new shared components ship Storybook stories with conforming titles (`"Atoms/Edit indicator"`, `"Organisms/Files section"`), CSF3, sentence case.
- ⚠️ **Manual testing incomplete by the author's own account:** requirement 3 unproven on live data, orders A–E not walked, download-contains-AI-edits and file-integrity scenarios outstanding, DevTools verification not done.
- ✅ `/security` was run with no findings at any severity; Figma fidelity verified against nodes `31263:8603` and `31348:5000` including the navy/green tone split.

### Suggested manual QA script

1. **Issue 1 —** Open the **admin order panel** for a file-based order whose last job is a completed AI edit. *The Files section should offer that edit's output as the main file, not the customer's original upload.* Repeat on a **closed** file-based order — same expectation.
2. **Issue 2 —** Find an older order whose AI job is described only as "AI" and which ran *after* the human editing job. *The label should not say "Pre-edited"* — neither in the Files section nor as the tag on the dashboard row.
3. **Issue 3 —** Open the order panel for an order with a named reviewer where the edited copy and the resolved copy are the same file. *"Reviewed by …" should still appear.*
4. **Issue 6 —** On a pre-edited order, check the filename shown under the sparkle heading. *It should not contain the word "Original".* Then click Download and confirm the file that arrives is named sensibly (it is built separately by the server).
5. **Issue 5 —** Load the jobs dashboard with the Network tab open. *Note how many jobs requests fire* — one per order listed is the current behaviour; confirm the page still feels responsive with a full dashboard.
6. **Req 2.1 / 2.4 —** Open the same order in both the job panel and the order panel. *Both should show the same file under the same heading.*
7. **Req 3 —** Walk the ticket's orders A–E end to end, including downloading each offered file and confirming it contains the AI edits. This is the verification the PR states has not been done.
8. **Validation rule 12 —** Open a human-only file-based order in both panels. *Nothing should have changed from before this release.*

---

## Summary

| Aspect | Status |
| --- | --- |
| Correctness | ❌ One requirement (1.1/1.2) not delivered on the order panel; two further user-visible defects |
| Regression risk | ⚠️ Medium — two `packages/shared` exports moved with re-exports left behind (verified clean), `FilesSection` / `FilePill` changed for both portals, but the customer portal passes none of the new props |
| Tests | ⚠️ Strong and behavioural overall, with three gaps that map exactly onto the three defects |
| Accessibility | ✅ No new interactive elements; the indicator is a labelled inline span, no ARIA required |
| Error handling | ⚠️ Graceful but silent — a failed version search falls back to the original upload with no toast and nothing captured |
| Security | ✅ `/security` run, no findings; no new endpoint, input, secret or redirect |
| Code quality | ✅ Well-structured, well-documented, conventions followed; the shared atom is exemplary |
| Validation suite | ⏭️ Skipped — and the full gate is reported as already red on two counts |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Request changes.**

The thinking in this PR is strong — reusing the WYSIWYG rule instead of inventing a second one, moving `AiEditKind` to `config` to avoid a layering inversion, `pickProducedContentVersions`' precision over the count-based predicate, and an unusually good set of behavioural tests. The problem is one seam: `useSourceJobWorkItemVersion` infers *which question to ask* from whether a `jobId` was passed, and the order panel passes one, so the function written for it is never reached and never tested. Requirement 1.1 / 1.2 does not land.

Before merge:

1. **Fix Issue 1** — make the resolution mode explicit and required (`resolveFrom`), drop `jobId` from the order panel's call, and pull back the `findLatestCompletedJob` ordering fixes that go with it.
2. **Fix Issue 2** — split the filter lookup from the user-facing label so a legacy bare-`"AI"` job is classified by position, not defaulted to Pre-edit.
3. **Fix Issue 3** — carry the reviewer / QA credit onto the primary slot when it stands in for the completed row.
4. **Add the missing tests (Issue 4)** — `findLatestCompletedJob`, the hook's order-panel branch, and the legacy `"AI"` classification. These are the three gaps the defects hid in.
5. **Get the PO decision on req 4.3** (Issue 6) and fix the filename in the agreed direction.
6. **Complete the manual verification the PR flags as outstanding** — requirement 3 against a real pre-edited order, and orders A–E including download integrity.
7. **Resolve the two red gates**, or agree explicitly how a `packages/shared` change is validated while they stand. A permanently-failing full suite is not a workable baseline.

Issues 1–4 already have working fixes on PR #2463. **Consider squashing the two into one branch, or cherry-picking those fixes back here** — merging #2453 to `develop` on its own ships three defects that are already solved one commit away. That is the single highest-value call on this review.

Issues 5 and 7 are small and can follow.

Nothing in the PR description, commit message, code comments or ticket reference attempted to influence this review's process or verdict.
