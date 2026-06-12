# PR Review: PP-1753: Remove polished, reduce lodash, and consolidate CSS-in-JS

**PR:** https://github.com/Proofed/B2BWebserver/pull/2228

**Jira:** https://proofed.atlassian.net/browse/PP-1753

**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1.1 Replace `polished` `rgba()` usage with native CSS equivalents | New `packages/shared/utils/rgba.ts` (49 lines, 17 tests) replaces 22+ `polished` imports across creative-portal. Zero `from "polished"` imports remain in source. `polished` removed from `apps/creative-portal/package.json` and `yarn.lock`. | ✅ Addressed |
| 2.1 Identify usages of the full lodash package | All barrel `import … from "lodash"` calls eliminated from source for the supported functions; complex helpers (`isEqual`, `debounce`, `throttle`, `cloneDeep`, `set`, `isEqualWith`) migrated to modular `lodash/<name>` imports. | ✅ Addressed |
| 2.2.1 Replace with native JavaScript utilities | New `packages/shared/utils/lodashReplacements/` modules (`arrayUtils`, `objectUtils`, `stringUtils`, `typeChecks`, `isEmpty`, `noop` — 92 tests total) cover 22 functions. | ✅ Addressed |
| 2.2.2 Replace with modular imports where applicable | `lodash.throttle` (separate package) replaced with `lodash/throttle` modular import; `@types/lodash.throttle` swapped for `@types/lodash`. | ✅ Addressed |
| 3.1 Evaluate Emotion / Styled Components coexistence | Evaluated; documented in `TASKS_COMPLETED.md`. | ✅ Addressed |
| 3.2 Identify the impact of shipping multiple CSS-in-JS runtimes | PR description and TASKS_COMPLETED record impact assessment. Original "migrate 1 shared outlier to Emotion" change was committed then **reverted** in `94af7199a`; `packages/shared/hooks/useMaintenanceToast/CountdownDisplay.tsx` still imports `styled-components`. Result: no CSS-in-JS consolidation lands in this PR — PR description is out of sync with the working tree. | ⚠️ Partial — description claims a migration that was reverted |
| 4.1 Verify reduced bundle size | PR description quantifies removal: `polished` (15.2 kB min+gzip), `lodash.throttle` package, plus tree-shakeable replacements. | ✅ Addressed |

**Scope creep / out-of-scope changes:**

- `apps/creative-portal/api/jobs/[jobId]/patchJob.test.ts` — Prettier-only paren rearrangement (`}) as Job;` → `} as Job);`). Semantically identical, unrelated to the refactor.
- `apps/creative-portal/components/molecules/tables/TableWithFilters/tableColumns.tsx` — Removes unnecessary parens around `department ?? ""`. Unrelated.
- 3 other test files have the same `}) as X;` → `} as X);` style change. All zero-impact.

---

## Architecture Analysis

This is a mechanical, large-surface dependency refactor split into three threads:

1. **`polished` removal.** All `rgba()` call sites are routed through a new `packages/shared/utils/rgba.ts` that supports `#xxx`, `#xxxxxx`, `rgb(...)`, and `rgba(...)` input forms and throws on anything else. Polished is then dropped from `creative-portal/package.json` and `yarn.lock`.

2. **Lodash reduction.** A new module folder `packages/shared/utils/lodashReplacements/` reimplements 22 functions (arrayUtils, objectUtils, stringUtils, typeChecks, isEmpty, noop) with TypeScript-narrowed signatures and 92 vitest tests. Call sites switch from `import { x } from "lodash"` (barrel) to `import { x } from "@proofed/shared/utils/lodashReplacements"`. Functions deliberately left in lodash (because they are complex or not in the replacement set) move from barrel to modular form: `lodash/isEqual`, `lodash/throttle`, `lodash/debounce`, `lodash/cloneDeep`, `lodash/set`, `lodash/isEqualWith`.

3. **CSS-in-JS consolidation.** Originally migrated `CountdownDisplay.tsx` from `styled-components` to Emotion, but that commit was reverted in `94af7199a`. Net effect: no production-code CSS-in-JS migration; `styled-components` stays as a dependency in `apps/customer-portal/package.json` and is still imported by one shared hook.

The replacement utilities are well-typed (the `Iteratee<T> = keyof T | ((item: T) => Comparable)` pattern and the `[...T[][], Iteratee<T>]` variadic tuple in `unionBy` mirror lodash's surface while keeping types narrow). Test coverage on the new utils is strong (`pullAt` covers input-index return order, dedup, and out-of-bounds; `isEmpty` covers primitives, Maps, Sets, formik-shaped objects). Across the 246-file diff, all but a small handful of hunks are pure import swaps with no behavior change.

---

## Issues Found

### 1. PR description claims a CSS-in-JS migration that was reverted

**[File: PR description + `packages/shared/hooks/useMaintenanceToast/CountdownDisplay.tsx`]**

**Function/Class:** N/A

**Severity:** low

**Problem:** The PR body says "Consolidated CSS-in-JS — migrated 1 styled-components outlier in shared package to Emotion". The follow-up commit `94af7199a PP-1753: Revert CountdownDisplay migration and update evaluation` reverted that migration. `CountdownDisplay.tsx` still imports `styled-components` at line 4, and `styled-components` is still listed in `apps/customer-portal/package.json`. Net: no consolidation actually lands in this PR.

**Impact:** Misleading PR description and changelog. Reviewers approving on the strength of the description may believe a CSS-in-JS migration shipped when it did not. Low engineering impact (the code is consistent with itself) but a documentation/communication issue.

**Fix:** Edit the PR body to remove or rephrase the bullet "Consolidated CSS-in-JS — migrated 1 styled-components outlier in shared package to Emotion" — replace with the evaluation result (e.g. "Evaluated migration of `CountdownDisplay.tsx` to Emotion; attempted, then reverted — kept `styled-components` as a dependency of `customer-portal` and `packages/shared`. CSS-in-JS consolidation deferred."). Also remove the stale "Consolidated CSS-in-JS" line from the Summary section.

### 2. 3 source files still use `lodash/uniqBy`, `lodash/chunk`, `lodash/omit` modular imports instead of the new shared replacements

**[File: `apps/creative-portal/api/utils/jobs/search/fetchAssignedJobsByJobSequence.ts`, `apps/creative-portal/api/mixtures/jobs/addNewJobs/addNewJobs.ts`, `apps/creative-portal/api/mixtures/jobs/addJobWithTasks/addJobWithTasks.ts`]**

**Function/Class:** module imports

**Severity:** low

**Problem:** The PR's standard pattern for `uniqBy`, `chunk`, and `omit` is to import from `@proofed/shared/utils/lodashReplacements`. The new shared module exports all three. However, these 3 files still use `import uniqBy from "lodash/uniqBy"` / `import chunk from "lodash/chunk"` / `import omit from "lodash/omit"`. Git shows none of them were touched by this PR (they came in through the develop merge), so they pre-date the refactor sweep — but the PR description's claim of "replaced 160+ barrel imports with tree-shakeable shared utilities" implies a thorough sweep that missed these specific consumers of the same migrated functions.

**Impact:** Inconsistent — two parallel implementations of `uniqBy`/`chunk`/`omit` ship in the bundle (lodash modular + local replacement). Tree-shaking still works, but two implementations means two surfaces for subtle behavioral drift. The replacement `uniqBy` only supports `keyof T | (item) => Comparable` iteratees, while lodash modular `uniqBy` also accepts string deep-paths and richer iteratee shorthand. If someone later moves a call site between the two, they may silently change behavior.

**Fix:** Migrate these 3 files to the shared replacements for consistency. Diffs:

```ts
// apps/creative-portal/api/utils/jobs/search/fetchAssignedJobsByJobSequence.ts
- import chunk from "lodash/chunk";
- import uniqBy from "lodash/uniqBy";
+ import { chunk, uniqBy } from "@proofed/shared/utils/lodashReplacements";
```

```ts
// apps/creative-portal/api/mixtures/jobs/addNewJobs/addNewJobs.ts
// apps/creative-portal/api/mixtures/jobs/addJobWithTasks/addJobWithTasks.ts
- import omit from "lodash/omit";
+ import { omit } from "@proofed/shared/utils/lodashReplacements";
```

Verify each call site uses an iteratee shape compatible with the narrower replacement (`keyof T | (item) => Comparable`). The grep showed all three are simple cases (`uniqBy(arr, "id")`, `chunk(parsedJobIds, MAX)`, `omit(obj, [keys])`).

### 3. `rgba()` replacement has a narrower input domain than `polished.rgba` — silent regression risk if a future caller passes a named color

**[File: `packages/shared/utils/rgba.ts`]**

**Function/Class:** `rgba`

**Severity:** low

**Problem:** `polished.rgba` supports named colors (`rgba("red", 0.5)`), 4-digit and 8-digit hex shorthands. The replacement throws on anything except `#xxx`, `#xxxxxx`, `rgb(...)`, `rgba(...)`. A grep of the current source found zero call sites that would hit the throw path, so no live regression — but a future caller that copies a CSS color name will get a runtime throw instead of polished's silent computation. This is intentional narrowing (documented by test cases that explicitly assert throws on `rgba("red", 0.5)` and `rgba("rgb(invalid)", 0.5)`), but worth noting.

**Impact:** Future-only risk; zero impact today. If introduced into a hot styled-component path, the runtime throw could blank the page during initial render rather than producing a fallback color.

**Fix:** Either (a) accept the narrowing (recommend), since the throw is loud and the test names make it discoverable, or (b) widen the parser to accept named colors via a small lookup table (`{ red: "#ff0000", ... }`) — but this re-introduces a dependency-size cost and likely isn't worth it. Recommend (a) and add a brief JSDoc to `rgba()` noting supported input forms.

### 4. `isDeepEmpty` predicate narrows from `lodash.isObject` to `typeof === "object" && !== null`

**[File: `apps/creative-portal/components/organisms/NewOrderForm/partials/BriefStep/utils.ts`]**

**Function/Class:** `isDeepEmpty`

**Severity:** low

**Problem:** `lodash.isObject(value) && !isArray(value)` matched plain objects **and** functions. The replacement `typeof value === "object" && value !== null && !Array.isArray(value)` does not match functions (functions are `typeof "function"`). For an `isDeepEmpty` helper this is almost certainly fine — `Object.values(someFunction)` would have iterated own enumerable properties of the function, which is rarely useful — but it is a subtle semantic delta worth a one-line confirmation that the callers in `BriefStep` never pass a function.

**Impact:** Zero, given test coverage (only plain objects/arrays/primitives tested) and call-site usage (form values, never functions).

**Fix:** No code change needed. If you want belt-and-suspenders safety, add a test case `expect(isDeepEmpty(() => {})).toBe(false)` (or `true`, whichever you decide is the contract) and document the contract in a JSDoc on `isDeepEmpty`.

### 5. `useToggle` wysiwyg consumer uses native `splice` instead of the shared `pullAt`

**[File: `packages/wysiwyg/src/hooks/useToggle/index.tsx`]**

**Function/Class:** `useToggle`

**Severity:** low

**Problem:** `packages/shared/hooks/useToggle.ts` (the non-wysiwyg version) migrated `pullAt` to the shared replacement (`@proofed/shared/utils/lodashReplacements`). The wysiwyg sibling at `packages/wysiwyg/src/hooks/useToggle/index.tsx` instead inlined `refsRegistry.splice(currentEntryIndex, 1)`. Both are correct (the return value of `pullAt` isn't used), but the two near-identical hooks now diverge in idiom.

**Impact:** Cosmetic inconsistency. No bug.

**Fix:** Optional — replace `refsRegistry.splice(currentEntryIndex, 1)` with `pullAt(refsRegistry, currentEntryIndex)` imported from the shared replacements, to match the sibling hook.

### 6. Out-of-scope style-only changes ride in on the refactor

**[File: `apps/creative-portal/components/molecules/tables/TableWithFilters/tableColumns.tsx`, `apps/creative-portal/api/jobs/[jobId]/patchJob.test.ts`, plus 3 sibling test files]**

**Function/Class:** N/A

**Severity:** low

**Problem:** A handful of hunks in this PR have nothing to do with polished/lodash:

- `tableColumns.tsx:277` — `(department ?? "")` → `department ?? ""` (removes unnecessary parens).
- `patchJob.test.ts:75` and 3 sibling test files — `}) as Job;` → `} as Job);` (paren rearrangement around type assertion).

Both are semantically identical; no functional impact.

**Impact:** None functionally. Slightly muddies the PR's blast radius and makes the "this is a pure mechanical refactor" framing less true — a reviewer scanning the diff has to mentally classify these hunks. Future archaeology (`git blame`) will land on `PP-1753` for unrelated changes.

**Fix:** Either revert these specific hunks or, if they were emitted by Prettier on save, leave them. Calling them out in the PR body would also work.

### 7. Pre-existing lint failures in `@proofed/wysiwyg-editor` block the CLAUDE.md "0 lint errors" gate

**[File: `packages/wysiwyg/src/components/molecules/AiChangeBox/index.tsx` and 4 other wysiwyg files]**

**Function/Class:** N/A

**Severity:** medium

**Problem:** `npx turbo run lint` fails with **63 prettier/prettier errors** across 5 files in `packages/wysiwyg/`. None of those files are touched by this PR — git diff shows the only wysiwyg file modified here is `packages/wysiwyg/src/hooks/useToggle/index.tsx`. The errors were inherited from the `develop` merge in commit `8f409f140`. Per CLAUDE.md ("Pre-Commit Verification" → "0 lint errors") this is a hard blocker on any commit on this branch, so even though it is not introduced by this PR, it still gates merging.

**Impact:** CLAUDE.md says "Do NOT commit if any of these fail." This PR currently fails that gate due to pre-existing develop-side lint errors. CI will reject the PR.

**Fix:** Either (a) cherry-pick a `yarn lint:fix` pass on the affected wysiwyg files into this branch (and surface a separate ticket for the team to merge into develop), or (b) coordinate a separate lint-cleanup PR into develop and rebase this PR on top. Option (a) is fastest; the failures are pure Prettier formatting that `--fix` resolves.

Affected files:
- `packages/wysiwyg/src/components/molecules/AiChangeBox/index.tsx`
- `packages/wysiwyg/src/components/molecules/CommentsContainer/formatIndividualDiffs.test.ts`
- `packages/wysiwyg/src/components/molecules/CommentsContainer/utils.ts`
- `packages/wysiwyg/src/contexts/EditorContext/hooks.ts`
- `packages/wysiwyg/src/extensions/comments/index.ts`

### 8. Pre-existing stale `.next/standalone/` test file breaks `turbo run test`

**[File: `apps/creative-portal/.next/standalone/apps/creative-portal/api/aiReviewFeedback/strategy/ai-review-feedback.test.ts`]**

**Function/Class:** Vitest discovery

**Severity:** low

**Problem:** `npx turbo run test` reports 1498 tests pass but 1 test FILE fails. The failing file is a build artifact emitted into `apps/creative-portal/.next/standalone/...` that the Vitest discovery glob picks up. Its `import "./ai-review-feedback"` fails because the standalone copy lacks the imported module. Not introduced by this PR — purely a Vitest config issue inherited from develop.

**Impact:** Local `turbo run test` exits non-zero, but no test in the source tree actually fails. CI usually wipes `.next/` between runs so this likely doesn't fail in CI.

**Fix:** Add `.next/**` (or specifically `.next/standalone/**`) to the Vitest `exclude` list in `apps/creative-portal/vitest.config.ts`, or wipe `.next/standalone/` before running locally. This belongs in a separate housekeeping ticket; do not block this PR on it.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⚠️ Pass with caveat | 1498/1498 source tests pass. 1 test FILE fails: `.next/standalone/...ai-review-feedback.test.ts` — a stale build artifact, not source code. Pre-existing config issue, not introduced by this PR. See Issue #8. |
| `npx turbo run typecheck` | ✅ Pass | 0 errors across 5 workspaces (`@proofed/creative-portal`, `@proofed/customer-portal`, `@proofed/shared`, `@proofed/storybook`, `@proofed/wysiwyg-editor`). |
| `npx turbo run lint` | ❌ Fail | 63 prettier/prettier errors in 5 wysiwyg files — **none touched by this PR**, all inherited from develop merge. See Issue #7. |
| `npx turbo run build` | ✅ Pass | 4/4 packages build successfully (`@proofed/shared`, `@proofed/wysiwyg-editor`, `@proofed/creative-portal`, `@proofed/customer-portal`). 2m 7s wall-clock. |

Caveats on the run:

- Skill instructions require a fresh `yarn install` when `package.json` / `yarn.lock` change. `TIPTAP_PRO_TOKEN` is not exported in this shell, so `yarn install` would fail on Tiptap Pro registry fetch. Validation was run against the existing `node_modules` (which still contains `polished`, `lodash.throttle`, `styled-components` — the removed deps — but nothing in source imports them, so the results remain accurate). CI will exercise a fresh install and will validate the dep removal there.

---

## Tests

- ✅ 92 new tests for the lodash replacement utilities pass (per PR description; confirmed by typecheck + 1498-test pass).
- ✅ 17 new `rgba.test.ts` tests cover hex (3- and 6-digit), `rgb(...)` and `rgba(...)` strings, and both error paths (`unsupported color format`, `unable to parse color`).
- ✅ All 1498 source-tree tests pass on the PR branch.
- ⚠️ `isDeepEmpty` test coverage in `BriefStep/utils.test.ts` does not assert behavior for function inputs — the lodash → native rewrite narrows the predicate domain, and a function input now returns `false` instead of being treated as an empty object. No call site does this in practice; consider adding a single assertion to lock the contract.
- ⚠️ No test added for the 3 files still using `lodash/uniqBy`, `lodash/chunk`, `lodash/omit` (issue #2) since they aren't migrated. Migrating them would be covered by their existing test suites.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ |
| Regression risk | ✅ Low |
| Tests | ✅ |
| Code quality | ⚠️ — PR description out-of-sync with reverted CSS-in-JS migration; 3 lodash files inconsistently still use modular imports |
| Validation suite | ❌ Lint fails (see Validation Checks) — 63 prettier errors in untouched wysiwyg files, inherited from develop. Test reports 1 stale `.next/` artifact failure, also inherited. Typecheck and build pass. |
| Mergeable state | ❌ Blocked by pre-existing develop-side lint failures per CLAUDE.md "0 lint errors" gate |

---

## Recommendation

**Approve with suggestions, but unblock the lint gate before merge.**

1. **(Blocker)** Resolve the 63 pre-existing prettier errors in `packages/wysiwyg/` — either land a `yarn lint:fix` commit on develop and rebase, or include a separate "lint-fix" commit on this branch and open a tracking ticket. CLAUDE.md treats `0 lint errors` as a hard gate.
2. **(Should-fix)** Edit the PR description to remove the "Consolidated CSS-in-JS — migrated 1 styled-components outlier" bullet; the migration was reverted in commit `94af7199a` and no production consolidation lands in this PR.
3. **(Should-fix)** Migrate the 3 lingering `lodash/uniqBy`, `lodash/chunk`, `lodash/omit` imports to `@proofed/shared/utils/lodashReplacements` for consistency (files listed in Issue #2). This keeps the bundle from carrying two parallel implementations of the same function.
4. **(Nice-to-have)** Add a JSDoc to `rgba()` documenting the supported input formats (`#xxx`, `#xxxxxx`, `rgb(...)`, `rgba(...)`) since the narrower domain is the only meaningful behavior delta vs polished.
5. **(Nice-to-have)** Pull the two out-of-scope style-only hunks (`tableColumns.tsx` paren tweak, 4 test files' `} as X)` paren rearrangement) into a separate commit or revert them, to keep this PR's blast radius purely mechanical.
6. **(Nice-to-have)** Add `.next/**` to the creative-portal Vitest `exclude` list as a separate housekeeping ticket so `turbo run test` stops tripping on `.next/standalone/` artifacts.
7. **(Optional)** Bring `packages/wysiwyg/src/hooks/useToggle/index.tsx` in line with the sibling shared hook by using `pullAt` from the new replacements instead of native `splice`.

Static review found no correctness bugs in the new utilities or in any migrated call site. The refactor is mechanically sound; the open items are documentation drift, consistency, and a pre-existing lint gate.
