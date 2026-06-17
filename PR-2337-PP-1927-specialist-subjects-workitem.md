# PR Review: feature/PP-1927: Switch Specialist Subjects API to WorkItem List

**PR:** https://github.com/Proofed/B2BWebserver/pull/2337
**Jira:** https://proofed.atlassian.net/browse/PP-1927
**Status:** Waiting for Deployment

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1.1 Source master subject list from WorkItem List (`category: subject`) replacing deprecated `SPECIALISTSUBJECTS` | `fetchAllSpecialistSubjects` now calls `fetchWorkItemList("subject")`, which posts header `category: "subject"` against the `workItem` (`WORKITEM`) directory entry | ✅ Addressed |
| 1.2 Retire `specialistSubjects → SPECIALISTSUBJECTS` mapping once unused | `specialistSubjects` entry removed from `packages/shared/config/b2bRoutesMap.ts`; repo-wide grep confirms no remaining `SPECIALISTSUBJECTS` reference | ✅ Addressed |
| 2.1 Preserve `SpecialistSubject` shape (`{ id, subject }`); map WorkItem `value → subject`, keep `id` | `.map(({ id, value }) => ({ id, subject: value }))`; `SpecialistSubject` type unchanged | ✅ Addressed |
| 2.2 Downstream consumers (`/api/specialistSubjects`, `makeSubjectIdDict`) keep working | Both consumers (`apps/creative-portal/api/specialistSubjects/getAll.ts`, `apps/creative-portal/api/users/[userId]/specialistSubjects/getUserSpecialistSubjects.ts`) untouched; receive the same `SpecialistSubject[]` shape | ✅ Addressed |
| 2.3 UI surfaces show the same subjects, same order, same selection/save behaviour | Same subjects ✅. **Order:** PR adds new client-side `localeCompare` alphabetical sort — diverges from "same order as before" unless the legacy service already returned alphabetic order (unconfirmed) | ⚠️ Partial |
| 2.4 Only active subject records listed | No client-side `active` filter; PR assumes backend filters before returning. Sample response in the spec shows `active: null`, suggesting the backend may not surface this signal | ⚠️ Partial (needs backend confirmation) |
| 3.1 `USERSUBJECTS` left untouched | Verified — `userSubjects` mapping preserved in `b2bRoutesMap.ts`; per-user fetch/add/remove utils unchanged | ✅ Addressed |

**Out-of-scope additions (intentional, called out in PR description):**

- New reusable `fetchWorkItemList(category, options?)` helper accepts the full `WorkItemCategory` union (`"type" | "format" | "subject"`) so future `type` / `format` migrations can reuse it. Forward-looking, justified.
- Client-side alphabetical sort (added in commit `77bb27c88`) — see ⚠️ in row 2.3.

---

## Architecture Analysis

The change is a narrow, surgical swap of a single transport call. The new `fetchWorkItemList` helper mirrors the established pattern in `packages/shared/api/utils/workItemContentVersion/fetchWorkItemContentVersions.ts`: it composes `prepareServiceAxiosConfig` with a header bag, then issues a typed `axios.get`. Placing it under `packages/shared/api/utils/workItem/` alongside a sibling `types.ts` (`WorkItemCategory`, `WorkItemListItem`) is consistent with how `workItemContentVersion` is organised, and leaves clean room for future `getWorkItemList`-style handlers.

The mapping layer lives where it should — in the app-level `fetchAllSpecialistSubjects` (the boundary between the WorkItem transport and the existing `SpecialistSubject` consumer shape) — so the shared helper stays generic. The deprecated `specialistSubjects` directory key is removed in the same change, leaving no dead config entry. Public contracts (`/api/specialistSubjects`, `makeSubjectIdDict`, `useSpecialistSubjectsQuery`, the `SpecialistSubject` type) are all unchanged, so blast radius beyond the two new files and the one rewritten util is zero.

The one deliberate widening — adding an alphabetical sort that the legacy implementation did not perform — is the only behavioural change worth scrutinising. Everything else is a like-for-like transport swap.

---

## Issues Found

### 1. Client-side alphabetical sort may diverge from the "same order as before" AC

**[File: apps/creative-portal/api/utils/specialistSubjects/fetchAllSpecialistSubjects.ts]**

**Function/Class:** fetchAllSpecialistSubjects

**Severity:** medium

**Problem:** The new implementation appends `.sort((a, b) => a.subject.localeCompare(b.subject))` (commit `77bb27c88`). The legacy `SPECIALISTSUBJECTS` call returned `data` straight from `axios.get` with no client-side sort. Acceptance Criterion 2.3 says the subjects must load "in the same order" as before; that wording is satisfied only if the legacy server response was already alphabetical. If it wasn't, this PR silently re-orders the list in every UI surface (Settings → Specialist Subjects, onboarding Step 8, team-member profile).

**Impact:** Likely benign-to-positive for the UX (alphabetical is the most natural order for a subject picker), but it is a behavioural change beyond the documented scope ("invisible to consumers") and a literal reading of AC 2.3. Reviewers/QA comparing pre- and post-migration lists side by side will see different orderings if the legacy backend wasn't alphabetic.

**Fix:** Either (a) confirm with the product owner that alphabetical sort is the desired post-migration order and update the Jira ticket / PR description to call this out explicitly, or (b) drop the `.sort(...)` and rely on whatever order the WorkItem service returns (matching the legacy behaviour of "trust the backend"). If keeping the sort, consider passing locale options explicitly to make the intent obvious:

```typescript
.sort((firstSubject, secondSubject) =>
  firstSubject.subject.localeCompare(secondSubject.subject, undefined, {
    sensitivity: "base"
  })
);
```

### 2. No client-side filter for inactive WorkItem records

**[File: apps/creative-portal/api/utils/specialistSubjects/fetchAllSpecialistSubjects.ts]**

**Function/Class:** fetchAllSpecialistSubjects

**Severity:** low

**Problem:** Jira's validation rule states "Inactive WorkItem subject records must not appear in the list." The WorkItem response shape (per the ticket) includes an `active` field, but `WorkItemListItem` in this PR only declares `{ id, value }` and the mapping does not consider `active`. The PR effectively assumes the WorkItem backend filters inactive rows before returning. The Jira sample response (`active: null`) does not confirm or deny this — `null` is ambiguous.

**Impact:** If the WorkItem service ever returns inactive subjects in the list response, every UI surface will display them and `makeSubjectIdDict` will index them. Risk is concentrated at devtest cut-over: a bad data shape only surfaces once `WORKITEM` is provisioned in a real environment.

**Fix:** Either confirm explicitly with the backend team that the `WORKITEM` list endpoint pre-filters by `active = true`, or add a defensive client-side filter. If filtering client-side, extend the type and the mapping in one step:

```typescript
// packages/shared/api/workItem/types.ts
export interface WorkItemListItem {
  id: number;
  value: string;
  active: boolean | null;
}

// fetchAllSpecialistSubjects.ts
return workItems
  .filter(({ active }) => active !== false)
  .map(({ id, value }) => ({ id, subject: value }))
  .sort(...);
```

(`active !== false` keeps `null`/`undefined` records visible, matching the "assume active unless explicitly inactive" reading of the sample response.)

### 3. Test name understates what it verifies

**[File: apps/creative-portal/api/utils/specialistSubjects/fetchAllSpecialistSubjects.test.ts]**

**Function/Class:** describe("fetchAllSpecialistSubjects") → it("maps WorkItem items ({ id, value }) to SpecialistSubject ({ id, subject })")

**Severity:** low

**Problem:** The test's mock returns `[Maths, English]` and the assertion expects `[English, Maths]`. The reordering is only correct because of the new `.sort()` call, so this single test now covers both the mapping AND the sort. A later test ("sorts the subjects alphabetically by name") covers sorting in isolation. The mapping test would be more precise — and survive future sort-policy changes — if its inputs were already alphabetical.

**Impact:** None at runtime. Maintainability only: a reader scanning failure output sees a "mapping" test fail when the regression is actually about ordering.

**Fix:** Change the mapping test's mock to already-sorted input so the assertion isolates mapping:

```typescript
mockedFetchWorkItemList.mockResolvedValue([
  { id: 2, value: "English" },
  { id: 1, value: "Maths" }
]);

const result = await fetchAllSpecialistSubjects();

expect(result).toEqual([
  { id: 2, subject: "English" },
  { id: 1, subject: "Maths" }
]);
```

### 4. PR description has a stale type snippet

**[File: PR #2337 description]**

**Function/Class:** N/A (PR body, "Areas of Change" bullet 2)

**Severity:** low

**Problem:** The body says `WorkItemListItem ({ Id, value })` — capital `Id`. The committed type and the Jira spec both use lowercase `id` (the third commit, `425732a32`, fixed this in code). The PR description was never updated to match.

**Impact:** Confuses reviewers who cross-reference the description against the code. Cosmetic.

**Fix:** Update the PR description bullet to `WorkItemListItem ({ id, value })`.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ✅ | 1616/1616 pass across all 4 workspaces (171 files, ~40s). New tests in this PR (4 in `fetchAllSpecialistSubjects.test.ts` + 4 in `fetchWorkItemList.test.ts`) all pass. |
| `npx turbo run typecheck` | ✅ | 0 errors across 5 workspaces. |
| `npx turbo run lint` | ❌ | 63 Prettier errors in `packages/wysiwyg/src/extensions/comments/index.ts` and `packages/wysiwyg/src/extensions/aiSuggestion/index.ts`. **Pre-existing on `develop`** — neither file is touched by this PR. Per CLAUDE.md ("0 lint errors") this technically blocks merge; treatment is up to the team. |
| `npx turbo run build` | ✅ | All 4 app/package builds complete (~2m35s). 0 type warnings. Storybook emits an unrelated "no output files found" warning that also exists on `develop`. |

---

## Tests

- ✅ `fetchAllSpecialistSubjects.test.ts` (new, 4 tests): category passed through, mapping `{ id, value } → { id, subject }`, alphabetical sort, empty array. Mocks `fetchWorkItemList` via `vi.mock`; standard pattern.
- ✅ `fetchWorkItemList.test.ts` (new, 4 tests): service name + `category` header, category passthrough (`"format"`), `options` forwarded to `prepareServiceAxiosConfig`, raw response returned untouched. Mocks `axios` and `prepareServiceAxiosConfig`; mirrors the established test style.
- ⚠️ No integration / contract test that the helper actually hits the `WORKITEM` directory entry with the right URL — relies on `getServiceMetadata` resolving `WORKITEM` at runtime. This is the same risk profile as every other directory-routed call in the repo and is consistent with how `fetchWorkItemContentVersions` is covered.
- ⚠️ The "Manual testing completed" box in the PR description is intentionally unchecked, gated on `WORKITEM` being provisioned in devtest. This is a real handoff risk: until that infra prerequisite is met, end-to-end behaviour is unverified. The PR author called this out explicitly.

The new-code test coverage is adequate; nothing else in the PR is reachable without these tests passing.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ Implementation matches the spec, but two AC details (alphabetical sort vs "same order", `active` filtering) need product/backend confirmation |
| Regression risk | ✅ Low — public contracts (`SpecialistSubject`, `/api/specialistSubjects`, `makeSubjectIdDict`, `useSpecialistSubjectsQuery`) are unchanged; the only behavioural delta is the new client-side sort |
| Tests | ✅ Adequate — 8 new unit tests cover both the helper and the mapping layer |
| Code quality | ✅ Follows existing helper pattern (`fetchWorkItemContentVersions`), reuses `prepareServiceAxiosConfig`, no new abstractions beyond what the migration needs |
| Validation suite | ❌ Lint failures exist — but in files unchanged by this PR and unchanged vs `develop` (i.e. a pre-existing repo-level issue, not a regression) |
| Mergeable state | ✅ Clean (GitHub-reported `mergeable_state: "clean"`) |

---

## Recommendation

**Approve with suggestions**

1. **(Blocker per CLAUDE.md, but pre-existing) Lint failures in `packages/wysiwyg`.** 63 Prettier errors in `packages/wysiwyg/src/extensions/comments/index.ts` and `.../aiSuggestion/index.ts`. They exist verbatim on `develop` and this PR doesn't touch wysiwyg, so the failure is not introduced here — but CLAUDE.md's pre-commit rule is strict ("Do NOT commit if any of these fail. Fix the issues first, even if they are in code you did not change"). Either fix in a separate PR before merging this one, or land this PR knowing the team is already in policy violation on `develop`. Flag to the wysiwyg owners.
2. **Confirm AC 2.3 ("same order as before") with the product owner.** Either (a) keep the client-side alphabetical sort and explicitly amend the ticket / PR description to say "post-migration order is alphabetical" or (b) drop the `.sort(...)` to literally preserve the legacy backend's ordering. See Issue 1.
3. **Confirm with the backend team that the `WORKITEM` list endpoint pre-filters inactive subjects.** If it doesn't, add an `active`-aware filter and extend `WorkItemListItem` accordingly. See Issue 2.
4. **Block merge on manual verification in devtest.** The PR author has correctly left "Manual testing" unchecked pending `WORKITEM` provisioning. Do not merge until the directory entry is in place and the Settings / onboarding / team-member surfaces have been validated end-to-end (`api-key` resolution, header passthrough, id integrity vs existing `USERSUBJECTS` assignments — AC validation rule 3).
5. **(Cosmetic) Update the PR description type snippet** from `{ Id, value }` to `{ id, value }` and tighten the mapping test (Issues 3 + 4).
