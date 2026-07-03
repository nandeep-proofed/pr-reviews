# PR Review: feature/PP-1938: Support AI Post-edit jobs in workflows (Creative Portal)

**PR:** https://github.com/Proofed/B2BWebserver/pull/2367
**Jira:** https://proofed.atlassian.net/browse/PP-1938
**Status:** In Progress

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| **1.1** Post-edit as distinct job, selectable only *after* service | `OptionalJobType.AI_POST_EDIT`; offered only in the optional (after-service) group; verified in dev app | ✅ |
| **1.2** Pre-edit selectable only *before* service; no longer after | Pre-service group offers `AI_PRE_EDIT`; optional-group add swapped to Post-edit | ✅ |
| **1.3** Post-edit uses the PP-1873 task type | **CONFIRMED against the live OMS `JobTaskTypes` endpoint** (b2btestapi, staging): `{ id: 36, name: "AI Post-edit", jobTypeId: 16, jobTypeName: "AI" }`. Exact case-sensitive match to the enum literal `AI_POST_EDIT = "AI Post-edit"` and all config-template/`JobTaskConfigurationName` keys; `jobTypeName: "AI"` matches the `JOB_NAME_TO_TYPE` mapping. Task type is deployed. (`AI Pre-edit` = id 35, also `jobTypeName: "AI"`; `AI Package` = id 25, `jobTypeName: "Service"`.) | ✅ |
| **1.4** Same sparkle treatment + removable | Sparkle for both pre/post; removable (only Review blocked) | ✅ |
| **1.5** Combined AI cap = 3 | Single `totalAiJobs` (pre + optional pre-legacy + post) `< 3` | ✅ |
| **2.1** Save/reload keeps position | `splitWorkflowOptions` buckets by jobSequence vs service; tested | ✅ |
| **3.1** Recognised as AI, never Service | `"ai post-edit" → JobType.AI` in `JOB_NAME_TO_TYPE` | ✅ |
| **3.2** Labelled distinctly in column *and* filter | Column returns raw prefix; filter split into two independent options | ✅ |
| **3.3** Order workflow: Post after, Pre before | Renders in jobSequence order; dead guard removed | ✅ |
| **Validation 1–4** (Pre not after, Post not before, >3 blocked, never Service/mislabelled) | Enforced structurally + mapping/label | ✅ |

**Scope note:** one out-of-strict-scope change — removing the dead `ENABLE_AI_PRE_EDIT` guard. Justified (directly about AI-job rendering, Req 3.3) and behaviour-preserving, but beyond the minimal diff.

---

## Architecture Analysis

The approach reuses the existing "AI collapses to `JobType.AI` for styling/status" model and only introduces granularity where the ticket demands distinction (the Current Job label + filter), via a filter-only `CurrentJobFilterValue = JobType | AiJobFilterValue` type rather than splitting `JobType.AI` (which would ripple through every `=== JobType.AI` styling check). Positions are enforced structurally — each type is offered in exactly one builder group — so the validation rules (Pre can't go after, Post can't go before) can't be violated through the UI. Legacy data is grandfathered by keeping `AI_PRE_EDIT` in `OptionalJobType` and bucketing on position (`splitWorkflowOptions`), not type. The granular filter value is threaded consistently through React state → the BFF matcher (`applyServerSideOrderFilters`) → the Yup request schema, so it survives validation and serialization.

---

## Issues Found

### 1. Legacy saved Order Management Views with `["AI"]` filter value

**[File: components/molecules/tables/TableWithFilters/partials/CurrentJobFilter/consts.ts]**

**Function/Class:** AVAILABLE_JOB_OPTIONS

**Severity:** low

**Problem:** The old filter option value was `JobType.AI` (`"AI"`). Saved OMVs / localStorage filters persisted before this change hold `currentJobFilter: ["AI"]`. That value no longer matches any entry in `AVAILABLE_JOB_OPTIONS` (now `"AI Pre-edit"` / `"AI Post-edit"`).

**Impact:** BFF filtering still works (a bare `"AI"` value falls through to the `JobType.AI` comparison and matches all AI orders), so results are correct. But in the dropdown the legacy value won't render as a selected chip, so a user reopening a saved view sees no AI checkbox ticked even though it's filtering. Cosmetic, affects only pre-existing saved views.

**Fix:** Acceptable to leave (self-heals when the user re-applies). If we want it clean, migrate `"AI"` → `"AI Pre-edit"` when loading a saved view. Low priority.

### 2. Column label reflects raw backend prefix casing

**[File: components/molecules/tables/TableWithFilters/utils.ts]**

**Function/Class:** getFormattedCurrentJobForOrderTable

**Severity:** low

**Problem:** For AI jobs the label now returns `liveStatus.split(":")[0].trim()` verbatim. If the OMS emits a different casing than expected (e.g. `"AI Post-Edit"`), the column shows that casing.

**Impact:** Display fidelity depends on the exact OMS task-name string. The live endpoint confirms the OMS uses `"AI Post-edit"` (matching the code), so this is correct today; noted only as a backend-coupling. Previously every AI job was hard-coded to `"AI Pre-edit"`, so this is strictly more correct.

**Fix:** None needed — verified against the live OMS endpoint (Q1 resolved).

### 3. Post-edit config template inherits Pre-edit's allowSkip inconsistency

**[File: components/pages/partners/[partnerId]/projects/[projectId]/settings/consts.tsx]**

**Function/Class:** DEFAULT_JOB_TASK_CONFIGURATION_TEMPLATES_* → "AI Post-edit"

**Severity:** low (informational)

**Problem:** The Post-edit template mirrors Pre-edit exactly, including `allowDeletion: false, requireCreation: true`. The builder, however, pushes AI jobs with `allowSkip: true, skipByDefault: false`. So there's a mismatch between the stored config template (→ `allowSkip: false` on load) and the builder-added value — but this is a **pre-existing** discrepancy inherited from Pre-edit, not new.

**Impact:** None observed; Post-edit behaves identically to Pre-edit (the ticket's intent). Flagged so a reviewer knows it was a deliberate mirror, not an oversight.

**Fix:** None unless the team wants to reconcile it for both Pre and Post together (out of scope here).

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` (creative-portal) | ✅ | **1696 / 1696 pass** (181 files). Re-run fresh on branch. |
| PP-1938 ticket-specific tests | ✅ | **82 / 82 pass** across the 5 touched test files (`api/orders/utils`, `TableWithFilters/utils`, `tableColumns`, `WorkflowBuilderModal/utils`, `getWorkflowsList`) |
| `npx turbo run test` (full monorepo) | ⚠️ | 1 failure — `@proofed/shared` `formatWordQuantity.test.ts` (`'10,00,000 words'` vs `'1,000,000 words'`). **Locale-only** (en-IN lakh grouping), **not code**: passes 23/23 under `LANG=en_US.UTF-8`. Pre-existing on `develop`, in a package this PR doesn't touch. |
| `npx turbo run typecheck` | ✅ | 0 errors (creative-portal) |
| `npx turbo run lint` | ✅ | 0 errors on changed files; lint-staged clean at commit |
| `npx turbo run build` | ⚠️ | **Environmental** — non-deterministic `MODULE_NOT_FOUND` at Next page-data collection (`/authentication-method`, `/availability`, `date-fns` chunk). **Reproduces on clean `develop` with changes stashed**; webpack compile succeeds every run. Not caused by this PR. |

_All creative-portal tests (where 100% of this PR's changes live) pass. The single monorepo failure is a pre-existing locale artifact in `@proofed/shared`, confirmed green under `en_US`/`en_GB` (i.e. CI)._

---

## Tests

**Fresh run on branch:** ticket-specific 82/82 ✅ · creative-portal full suite 1696/1696 ✅ · full monorepo 1 pre-existing locale failure in `@proofed/shared` (not this PR).

- ✅ Recognition (`"ai post-edit" → JobType.AI`, case-insensitive) — `api/orders/utils.test.ts`
- ✅ Independent Pre/Post filtering incl. legacy bare-AI union — `api/orders/utils.test.ts`
- ✅ Distinct column labels (Pre / Post / bare-AI fallback) — `TableWithFilters/utils.test.ts`
- ✅ Legacy after-service Pre-edit grandfathered into optional bucket — `getWorkflowsList.test.ts`
- ✅ Builder sparkle icon + Post-edit add path — `WorkflowBuilderModal/utils.test.ts`
- ⚠️ Combined cap-of-3 lives in the component render (`index.tsx`) and isn't covered by an automated test — only the builder add-item path is. Manually verified in the dev app. (Low)

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Matches all ticket requirements (1.3 confirmed against live OMS endpoint) |
| Regression risk | ✅ Low — additive; type-widening is a superset; one cosmetic legacy-view edge |
| Tests | ✅ Good coverage (⚠️ cap-of-3 render path manual-only) |
| Code quality | ✅ Idiomatic, minimal, well-commented |
| Validation suite | ✅ creative-portal test 1696/1696, typecheck, lint all pass · ⚠️ 1 pre-existing locale-only failure in `@proofed/shared` + environmental build fail (neither caused by this PR) |
| Mergeable state | ⚠️ Confirm CI build green (local build fails environmentally) |

---

## Recommendation

**Approve with suggestions** — the implementation is correct, well-scoped, and tested; the three issues found are all low severity. Before merge:

1. **Confirm CI build is green.** Local `turbo build` fails environmentally (proven pre-existing on clean `develop` — compile succeeds, page-data collection throws a non-deterministic `MODULE_NOT_FOUND` on unrelated pages). Rely on CI's clean environment as the authoritative build gate.
2. ~~**Verify the OMS task-type string** (Q1 / Req 1.3).~~ ✅ **RESOLVED** — confirmed against the live OMS `JobTaskTypes` endpoint (staging): `{ id: 36, name: "AI Post-edit", jobTypeId: 16, jobTypeName: "AI" }`, an exact case-sensitive match to the code. No change needed.
3. *(Optional)* Product decision on whether Post-edit should be pinned to the final step vs. offered at every after-service gap.

_Update: Q1 / Req 1.3 verified as resolved against the live OMS endpoint after the initial review — the only remaining pre-merge gate is the (environmental) CI build._
