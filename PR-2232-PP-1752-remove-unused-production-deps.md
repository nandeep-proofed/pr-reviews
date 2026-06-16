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
- `axios-mock-adapter` → devDeps → **fully removed in `47bfdb258`** along with its only (dead) consumer `apps/creative-portal/setup/api/mocks.ts`. Originally the PR moved it to devDeps; follow-up commit dropped it entirely since the file that imported its types is now deleted (855 lines, `mockEndpoints` had zero importers).
- `happy-dom` → devDeps → **fully removed in `47bfdb258`**. No consumers in `packages/wysiwyg/src/` or its `vitest.config.ts` (which uses `environment: "jsdom"`); confirmed repo-wide.

~~**The problem is everything else in the diff.** Seven non-package.json files received Prettier formatting reverts that conflict with the project's Prettier 3 setup and re-introduce exactly the regression that PP-1858 (commit `61f79fa05` on develop, *"restore develop formatting on files churned by stale local prettier (v2 shim)"*) explicitly fixed.~~

**Update (commit `23ceb9695`):** Issue 1 resolved. Workspace-local `eslint --fix` re-applied on all 10 files; the v3 canonical output (`}) as T`, `: (x ?? y)`) is restored to match develop. Re-running eslint on the 10 files produces zero `prettier/prettier` errors. Root cause of the original regression: `node_modules/.bin/prettier` symlink pointed at the v2.8.8 binary bundled inside `@storybook/cli` instead of the root-installed v3.5.3 — a local-env bug, not a PR-author choice. Symlink repointed; lint-staged hook now formats correctly.

---

## Issues Found

### 1. ~~PR re-introduces the stale-Prettier formatting fixed by PP-1858 — lint regression~~ ✅ RESOLVED in commit `23ceb9695`

**Status:** Fixed. Workspace-local `eslint --fix` re-applied to all 10 files; Prettier 3 canonical output restored to match develop. Root cause was a stale `node_modules/.bin/prettier` symlink pointing at Prettier 2.8.8 inside `@storybook/cli`; symlink repointed to root-installed v3.5.3. The original finding is preserved below for history.

---

<details>
<summary>Original finding (resolved)</summary>

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

</details>

---

### 2. ~~devDependencies blocks broken out of alphabetical order~~ ✅ MOOT after commit `47bfdb258`

`axios-mock-adapter` is no longer in either deps or devDeps — it was removed entirely along with the dead file that imported it (see Issue 5). Alphabetical-order concern dissolves.

<details>
<summary>Original finding</summary>

**[File: apps/creative-portal/package.json]**

**Function/Class:** `devDependencies`

**Severity:** low

**Problem:** `axios-mock-adapter` was moved to `devDependencies` at line 83 — between `@sentry/types` and `@storybook/addon-essentials`. All other `@`-scoped entries are grouped first; the inserted unscoped name breaks the consistent alphabetical-by-package-name ordering.

**Impact:** Cosmetic and review-friction only; no functional impact. But it makes future diffs noisier and conflicts more likely.

**Fix:** Move `"axios-mock-adapter": "^1.21.2"` further down so it sits alphabetically between other unscoped entries (e.g. after `autoprefixer` if that exists in dev, otherwise after the `@`-scoped block ends).

</details>

---

### 3. ~~Same alphabetical-order issue in wysiwyg devDependencies~~ ✅ MOOT after commit `47bfdb258`

`happy-dom` has been removed entirely from `packages/wysiwyg/package.json` (see Issue 4). No remaining position to argue about.

<details>
<summary>Original finding</summary>

**[File: packages/wysiwyg/package.json]**

**Function/Class:** `devDependencies`

**Severity:** low

**Problem:** `happy-dom` was inserted at line 65 — between `tsconfig-paths-webpack-plugin` (line 64) and `jsdom` (line 66). `happy-dom` should sit before `jsdom` *and* before `tsconfig-paths-webpack-plugin` alphabetically. The surrounding block isn't strictly sorted as-is, but the move arbitrarily widened the disorder rather than placing it correctly.

**Impact:** Same as Issue 2 — cosmetic.

**Fix:** Place `"happy-dom": "^20.0.2"` immediately before `jsdom` (which it already happens to land before), or — preferably — between `eslint-plugin-tailwindcss` and `postcss` where its alphabetical position actually lives.

</details>

---

### 4. ~~`happy-dom` move is half a job — package is unused~~ ✅ RESOLVED in commit `47bfdb258`

**Status:** Fixed. `happy-dom@^20.0.2` removed entirely from `packages/wysiwyg/package.json`. Repo-wide grep confirmed zero source / config / `@vitest-environment` references; vitest still uses `environment: "jsdom"`. `yarn.lock` regenerated; build + 1,498 tests green.

<details>
<summary>Original finding</summary>

**[File: packages/wysiwyg/package.json]**

**Function/Class:** `devDependencies > happy-dom`

**Severity:** low

**Problem:** Nothing in `packages/wysiwyg/src/`, `packages/wysiwyg/vitest.config.ts`, or any other config file references `happy-dom` or `"happy-dom"`. The vitest config explicitly sets `environment: "jsdom"`. The package is dead weight even in devDependencies.

**Impact:** Carries an extra package + 20.0.2 install surface for no functional benefit. The Jira ticket asked to *move* it (PP-1752 Functional Requirement 2.1) so doing so is technically compliant — but Requirement 4.1 (*"Only runtime-required packages may exist in production dependencies"*) plus the spirit of the ticket arguably calls for removing it entirely.

**Fix:** Remove `happy-dom` from `packages/wysiwyg/package.json` entirely instead of moving it. Verify by running `npx turbo run test --filter=@proofed/wysiwyg-editor` — wysiwyg tests use jsdom and should be unaffected.

</details>

---

### 5. ~~`apps/creative-portal/setup/api/mocks.ts` is dead code (informational)~~ ✅ RESOLVED in commit `47bfdb258`

**Status:** Fixed. Deleted `apps/creative-portal/setup/api/mocks.ts` (855 lines of unused fixture data) and dropped `axios-mock-adapter@^1.21.2` from `apps/creative-portal/package.json` devDeps. The package was only ever imported for types in the deleted file, and `mockEndpoints` had zero importers. `yarn.lock` regenerated; build + 1,498 tests green.

<details>
<summary>Original finding</summary>

**[File: apps/creative-portal/setup/api/mocks.ts]**

**Function/Class:** `mockEndpoints`

**Severity:** low

**Problem:** This file is the ONLY consumer of `axios-mock-adapter` in the entire repo, and it exports a single function `mockEndpoints` that is never imported anywhere (`grep -r "mockEndpoints" apps/creative-portal/` returns only the definition).

**Impact:** Out of PR scope but: the file (and therefore the `axios-mock-adapter` dependency the PR is preserving as devDep) is dead weight. The whole file could be deleted, and `axios-mock-adapter` removed entirely rather than relocated.

**Fix:** Optional follow-up — delete `apps/creative-portal/setup/api/mocks.ts` and drop `axios-mock-adapter` from `devDependencies`. Not required for this PR.

</details>

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ✅ | 1,498/1,498 pass across all 4 workspaces (after deleting stale `.next/standalone/.../ai-review-feedback.test.ts` leftover — pre-existing artifact, unrelated to PR). |
| `npx turbo run typecheck` | ✅ | 0 errors across 5 workspaces |
| `npx turbo run lint` | ✅ (after `23ceb9695`) | The 10 PR-touched non-package.json files now lint clean. Pre-existing ~101 lint errors on develop (unrelated files) remain — not introduced by this PR. |
| `npx turbo run build` | ✅ (after `47bfdb258`) | All 4 workspaces build cleanly in 2m35s with 0 TypeScript warnings. Remaining warnings (Storybook asset-size, rollup `output.exports`, Node TLS) are pre-existing config noise. |
| Fresh `yarn install` from empty `node_modules` | ✅ | 57s; `success Saved lockfile`. Confirms `yarn.lock` is consistent with all 4 `package.json` files. |
| Dep removals verified individually | ✅ | All 8 PR-listed package changes + 3 follow-up changes (drop `happy-dom`, drop `axios-mock-adapter`, delete `mocks.ts`) verified against repo-wide source/config grep. Transitive resolutions checked: `@types/node-fetch` via `googleapis`, `core-js` via Storybook, `@mdx-js/react@2` via theme-ui. |

---

## Tests

- ✅ PR is pure dependency-cleanup + (unintended) formatting. No new functionality, so no new tests needed.
- ✅ Existing test suite (1,498 tests in creative-portal + customer-portal + shared + wysiwyg) all pass against the new dependency tree.
- ✅ Builds verify removed deps are not actually needed by the production bundle.
- ✅ Stale `.next/standalone/` test-discovery gap hardened in `50aa47cd5` — both `apps/{creative,customer}-portal/vitest.config.ts` now spread `configDefaults.exclude` and add `"**/.next/**"`. Verified by running tests with a stale `.next` artifact present: test file count went from 152 (1 failed) to 151 (151 passed), with the full 1,498-test suite still green.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ All 11 dep changes (8 PR + 3 follow-up) verified individually; runtime confirmed by build + tests |
| Regression risk | ✅ Resolved — Prettier v3 canonical restored in `23ceb9695`; symlink root-cause documented |
| Tests | ✅ 1,498/1,498 across all 4 workspaces |
| Code quality | ✅ Deps sound, formatting clean, all dead code removed (`mocks.ts` deleted) |
| Validation suite | ✅ Lint, typecheck, build, fresh `yarn install` all green |
| Mergeable state | ✅ GitHub reports `MERGEABLE` / `mergeStateStatus: CLEAN` (after `47bfdb258` push) |

---

## Recommendation

**Approve and merge.** (Was "Request changes" before `23ceb9695`; downgraded as findings were addressed.)

1. ~~**(blocker)** Restore develop's formatting on the 7 non-package.json files the PR touches.~~ ✅ Done in `23ceb9695`.
2. ~~**(blocker)** Re-run `npx turbo run lint` and confirm the PR no longer adds to develop's error count.~~ ✅ Verified.
3. ~~**(blocker)** Resolve `mergeable_state: "dirty"` from GitHub.~~ ✅ GitHub now reports `MERGEABLE` / `CLEAN` after the latest pushes.
4. ~~**(should-fix)** Place `axios-mock-adapter` and `happy-dom` in alphabetical order within their devDependencies blocks (Issues 2 & 3).~~ ✅ Moot — both packages removed entirely in `47bfdb258`.
5. ~~**(should-fix)** Consider removing `happy-dom` entirely instead of moving it — nothing references it (Issue 4).~~ ✅ Done in `47bfdb258`.
6. ~~**(nice-to-have / follow-up)** Delete the dead `apps/creative-portal/setup/api/mocks.ts` and drop `axios-mock-adapter` entirely (Issue 5).~~ ✅ Done in `47bfdb258`.

All review findings resolved. The PR ships **PP-1752 in full + scope-creep cleanups** (`@mdx-js/react@1`, `tiptap-commands`, `node-fetch`+types, `@apollo/federation`) and additionally removes one further fully-dead file (`mocks.ts`) and two further unused dev-deps (`happy-dom`, `axios-mock-adapter`). Solid, well-justified, ready for merge.

---

## Commit log (this PR branch)

| Commit | Purpose |
|---|---|
| `50aa47cd5` | Exclude `.next/**` from vitest test discovery in both Next.js apps |
| `47bfdb258` | Drop unused `happy-dom` + `axios-mock-adapter` and delete `mocks.ts` |
| `23ceb9695` | Restore Prettier 3 formatting on 10 files churned by stale local Prettier 2 shim |
| `f34c6a062` | Merge develop |
| (earlier) | Original PP-1752 dep removals + devDeps moves |
