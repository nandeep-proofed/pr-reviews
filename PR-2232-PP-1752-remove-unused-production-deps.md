# PR Review: PP-1752: Remove dead/phantom dependencies and fix dep placement

**PR:** https://github.com/Proofed/B2BWebserver/pull/2232
**Jira:** https://proofed.atlassian.net/browse/PP-1752
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Remove `core-js` from customer portal where not referenced | Removed from `apps/customer-portal/package.json` dependencies; verified no source imports | ✅ Addressed |
| Move `happy-dom` to devDependencies in wysiwyg | Moved in `packages/wysiwyg/package.json` | ✅ Addressed (but see note — package appears entirely unused) |
| Move `axios-mock-adapter` to devDependencies in creative portal | Moved in `apps/creative-portal/package.json` | ✅ Addressed |
| All applications must build successfully after removal | `npx turbo run build` passes (creative + customer + storybook + wysiwyg) | ✅ Addressed |
| All tests must run without dependency errors | `npx turbo run test` passes — 1,498 tests in creative-portal alone | ✅ Addressed |
| Only runtime-required packages may exist in production dependencies | PR adds **scope creep** (good kind): also drops `tiptap-commands`, `@mdx-js/react@1`, `node-fetch`+`@types/node-fetch`, `@apollo/federation` resolution — all verified dead | ✅ Addressed |

**Scope creep notes (all positive):**
- PR exceeds Jira scope by also removing `tiptap-commands`, `@mdx-js/react@1`, `node-fetch`/`@types/node-fetch`, `@apollo/federation`. All verified to have zero source imports. Welcome additions.
- PR description correctly justifies keeping `mobx` (redoc peer dep) and `file-loader` (wysiwyg webpack config). Both verified.

---

## Architecture Analysis

The dependency-removal core of the PR is **clean and correct**. Each removed package was independently verified for zero source imports (ignoring `node_modules` / `.next` artifacts):

- `core-js` — no `import 'core-js'` / `from 'core-js'` anywhere in `apps/customer-portal/`. Next.js 14's `next/babel` preset injects polyfills on demand; no app-level polyfill loader exists. Safe to drop.
- `node-fetch` + `@types/node-fetch` (direct) — no source imports. `@types/node-fetch@^2.6.4` remains as a transitive of `googleapis`. Safe.
- `@mdx-js/react@1` — no source imports. `theme-ui > @theme-ui/mdx` keeps a `^1 || ^2` peer requirement now satisfied by the hoisted `@mdx-js/react@2.3.0`. Build verifies this resolves.
- `tiptap-commands` — Tiptap v1 package, no imports. Safe.
- `@apollo/federation` resolution — no Apollo consumers in the tree. Safe.
- `axios-mock-adapter` → devDeps — only consumer is `apps/creative-portal/setup/api/mocks.ts`, which imports only types (`from "axios-mock-adapter/types"`) and whose `mockEndpoints` export is never called. Move is safe.
- `happy-dom` → devDeps — no consumers in `packages/wysiwyg/src/` or its `vitest.config.ts` (which uses `environment: "jsdom"`). Move is safe but the package appears to be fully unused; could be dropped entirely.

**The problem is everything else in the diff.** Seven non-package.json files received Prettier formatting reverts that conflict with the project's Prettier 3 setup and re-introduce exactly the regression that PP-1858 (commit `61f79fa05` on develop, *"restore develop formatting on files churned by stale local prettier (v2 shim)"*) explicitly fixed. The PR branch's local Prettier evidently emitted the legacy v2 format `({...} as T)` and `: x ?? y` instead of the v3 format `({...}) as T` and `: (x ?? y)`. Validating against develop confirms each of these touched files now newly fails `npx turbo run lint`.

---

## Issues Found

### 1. PR re-introduces the stale-Prettier formatting fixed by PP-1858 — lint regression

**[File: apps/creative-portal/api/jobs/\[jobId\]/patchJob.test.ts]**

**Function/Class:** `makeJob` (lines 67–75; also same pattern in 6 other files)

**Severity:** high

**Problem:** The PR reverts Prettier 3's canonical output `}) as Job` back to the Prettier 2 legacy `} as Job)`. The same pattern is reverted across all 7 non-package.json files this PR touches:

- `apps/creative-portal/api/jobs/[jobId]/patchJob.test.ts` (line 75)
- `apps/creative-portal/api/mixtures/orders/getBulkActionsData/getBulkActionsData.test.ts` (lines 81, 92)
- `apps/creative-portal/api/orders/utils.test.ts` (lines 112, 229)
- `apps/creative-portal/api/utils/jobs/mergeJobPutBody.test.ts` (line 18)
- `apps/creative-portal/components/molecules/tables/TableWithFilters/tableColumns.tsx` (line 277 — `: (department ?? "")` → `: department ?? ""`)
- `apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/hooks.tsx` (lines 430–431 — nested ternary indentation regressed)
- `apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/DetailedOrderInfo.tsx` (line 220 — `: (createdForId ?? createdById)` → `: createdForId ?? createdById`)
- `apps/customer-portal/api/utils/mixtures/orders/createOrder/__tests__/utils.test.ts` (line 42)
- `apps/customer-portal/api/utils/mixtures/orders/createOrder/utils.ts` (lines 127, 194)
- `apps/customer-portal/components/molecules/OrderPriceSection/__tests__/utils.test.ts` (line 94)

This is the **exact regression PP-1858 (commit `61f79fa05`) addressed** — the commit message reads: *"restore develop formatting on files churned by stale local prettier (v2 shim)"*. The current PR is undoing that work on every file it touches.

**Impact:** `npx turbo run lint` fails. Measured deltas vs develop:
- customer-portal: develop has 22 errors → PR has 26 (**+3 new failing files**, all PR-touched)
- creative-portal: develop has 79 errors → PR has 85 (**+7 new failing files**, 5 are exactly the PR-touched test/source files; 2 others appear to be coincidental drift in PR branch state)

Per CLAUDE.md *"Do NOT commit if any of these fail"*, lint failure is a hard merge blocker. Even setting that policy aside, the PR is introducing regressions, not maintaining the baseline.

**Fix:** Run `yarn lint:fix` (or `yarn format`) at the repo root with the workspace-local Prettier 3, then re-commit. Verify the output matches develop's format on the same expressions, e.g.:

```typescript
// correct (matches develop after PP-1858)
const makeJob = (overrides: Partial<Job> = {}): Job =>
  ({
    id: 1,
    ...overrides
  }) as Job;

// also correct
const customerText =
  department && organisation !== department
    ? `${organisation}: ${department}`
    : (department ?? "");
```

Important: do NOT run a global `yarn format` that would also touch the ~100 unrelated files already failing lint on develop. Restrict the fix to the 7 files this PR is actually modifying. Alternative if `yarn lint:fix` overreaches: `git checkout origin/develop -- <each file>` and then re-apply the PR's actual intended changes — but those 7 files have **no intended changes** beyond the formatting revert itself, so a straight `git checkout origin/develop -- <file>` for all 7 is the cleanest fix. After that, only `package.json` / `yarn.lock` files should remain in the PR diff.

---

### 2. devDependencies blocks broken out of alphabetical order

**[File: apps/creative-portal/package.json]**

**Function/Class:** `devDependencies`

**Severity:** low

**Problem:** `axios-mock-adapter` was moved to `devDependencies` at line 83 — between `@sentry/types` and `@storybook/addon-essentials`. All other `@`-scoped entries are grouped first; the inserted unscoped name breaks the consistent alphabetical-by-package-name ordering.

**Impact:** Cosmetic and review-friction only; no functional impact. But it makes future diffs noisier and conflicts more likely.

**Fix:** Move `"axios-mock-adapter": "^1.21.2"` further down so it sits alphabetically between other unscoped entries (e.g. after `autoprefixer` if that exists in dev, otherwise after the `@`-scoped block ends).

---

### 3. Same alphabetical-order issue in wysiwyg devDependencies

**[File: packages/wysiwyg/package.json]**

**Function/Class:** `devDependencies`

**Severity:** low

**Problem:** `happy-dom` was inserted at line 65 — between `tsconfig-paths-webpack-plugin` (line 64) and `jsdom` (line 66). `happy-dom` should sit before `jsdom` *and* before `tsconfig-paths-webpack-plugin` alphabetically. The surrounding block isn't strictly sorted as-is, but the move arbitrarily widened the disorder rather than placing it correctly.

**Impact:** Same as Issue 2 — cosmetic.

**Fix:** Place `"happy-dom": "^20.0.2"` immediately before `jsdom` (which it already happens to land before), or — preferably — between `eslint-plugin-tailwindcss` and `postcss` where its alphabetical position actually lives.

---

### 4. `happy-dom` move is half a job — package is unused

**[File: packages/wysiwyg/package.json]**

**Function/Class:** `devDependencies > happy-dom`

**Severity:** low

**Problem:** Nothing in `packages/wysiwyg/src/`, `packages/wysiwyg/vitest.config.ts`, or any other config file references `happy-dom` or `"happy-dom"`. The vitest config explicitly sets `environment: "jsdom"`. The package is dead weight even in devDependencies.

**Impact:** Carries an extra package + 20.0.2 install surface for no functional benefit. The Jira ticket asked to *move* it (PP-1752 Functional Requirement 2.1) so doing so is technically compliant — but Requirement 4.1 (*"Only runtime-required packages may exist in production dependencies"*) plus the spirit of the ticket arguably calls for removing it entirely.

**Fix:** Remove `happy-dom` from `packages/wysiwyg/package.json` entirely instead of moving it. Verify by running `npx turbo run test --filter=@proofed/wysiwyg-editor` — wysiwyg tests use jsdom and should be unaffected.

---

### 5. `apps/creative-portal/setup/api/mocks.ts` is dead code (informational)

**[File: apps/creative-portal/setup/api/mocks.ts]**

**Function/Class:** `mockEndpoints`

**Severity:** low

**Problem:** This file is the ONLY consumer of `axios-mock-adapter` in the entire repo, and it exports a single function `mockEndpoints` that is never imported anywhere (`grep -r "mockEndpoints" apps/creative-portal/` returns only the definition).

**Impact:** Out of PR scope but: the file (and therefore the `axios-mock-adapter` dependency the PR is preserving as devDep) is dead weight. The whole file could be deleted, and `axios-mock-adapter` removed entirely rather than relocated.

**Fix:** Optional follow-up — delete `apps/creative-portal/setup/api/mocks.ts` and drop `axios-mock-adapter` from `devDependencies`. Not required for this PR.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ✅ | 1,498/1,498 pass in creative-portal; all 4 workspaces pass. **Note**: First run failed on a stale `.next/standalone/.../ai-review-feedback.test.ts` (pre-existing leftover build artifact, unrelated to PR). Cleaned and re-ran — clean pass. |
| `npx turbo run typecheck` | ✅ | 0 errors across 5 workspaces |
| `npx turbo run lint` | ❌ | **PR introduces +3 newly failing files in customer-portal** and **+7 in creative-portal** (every test/source file the PR touched). Separately, ~101 lint errors exist pre-existing on develop in unrelated files. |
| `npx turbo run build` | ✅ | All 4 builds clean (Storybook output-files warning is config noise, not failure) |

---

## Tests

- ✅ PR is pure dependency-cleanup + (unintended) formatting. No new functionality, so no new tests needed.
- ✅ Existing test suite (1,498 tests in creative-portal + customer-portal + shared + wysiwyg) all pass against the new dependency tree.
- ✅ Builds verify removed deps are not actually needed by the production bundle.
- ⚠️ Stale `.next/standalone/` triggered a test discovery failure; cleaning resolved it. Not a blocker, but worth noting that vitest's `include: ["**/*.test.{ts,tsx}"]` lacks a `.next/**` exclude — a separate hardening task.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Dependency removals are sound; runtime verified by build |
| Regression risk | ⚠️ Medium — stale-prettier reverts re-introduce a fix that already shipped; pure cosmetic but creates noisy diffs and live lint failures |
| Tests | ✅ All pass after cleaning stale build artifact |
| Code quality | ⚠️ Sound on the deps; broken on the formatting + alphabetical ordering |
| Validation suite | ❌ Lint fails with +10 new failing files attributable to this PR |
| Mergeable state | ❌ Dirty (per CLAUDE.md "Do NOT commit if any of these fail"); GitHub also reports `mergeable_state: "dirty"` (likely separate yarn.lock conflict that needs a fresh resolve) |

---

## Recommendation

**Request changes**

1. **(blocker)** Restore develop's formatting on the 7 non-package.json files the PR touches. Cleanest path: `git checkout origin/develop -- <each of the 7 files>` since the PR's intended scope is dependency cleanup only and none of those files needs any change. After that, re-verify the PR diff contains only `package.json` files and `yarn.lock`.
2. **(blocker)** Re-run `npx turbo run lint` and confirm the PR no longer adds to develop's error count.
3. **(blocker)** Resolve the `mergeable_state: "dirty"` from GitHub (the PR currently has a merge conflict — likely on `yarn.lock` after recent develop merges). Re-resolve and re-push.
4. **(should-fix)** Place `axios-mock-adapter` and `happy-dom` in alphabetical order within their devDependencies blocks (Issues 2 & 3).
5. **(should-fix)** Consider removing `happy-dom` entirely instead of moving it — nothing references it (Issue 4).
6. **(nice-to-have / follow-up)** Delete the dead `apps/creative-portal/setup/api/mocks.ts` and drop `axios-mock-adapter` entirely (Issue 5).

Once 1–3 land, this PR is straightforwardly mergeable — the actual dependency-cleanup work is solid, well-justified, and matches the Jira ticket's intent (and then some).
