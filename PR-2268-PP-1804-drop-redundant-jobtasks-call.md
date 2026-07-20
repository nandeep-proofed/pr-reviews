# PR Review: fix/PP-1804: Drop redundant /jobTasks call in Orders Services column

**PR:** https://github.com/Proofed/B2BWebserver/pull/2268

**Jira:** https://proofed.atlassian.net/browse/PP-1804

**Status:** Code Review (Jira) / `mergeable_state: clean` (GitHub)

---

## Jira Requirements vs Implementation

PP-1804 is a **spike / investigation** story — "Investigate root causes and solutions for slow load times with large order volumes" — not an implementation ticket. Acceptance criteria are about producing a prioritised list of recommendations, confirming feasibility of incremental/deferred loading, and highlighting quick wins.

This PR ships a **quick win**, plus a substantial refactor that prepares the table for the bigger-ticket optimisations in the spike's recommendation list. Strictly the spike's ACs are about producing findings, not code — so judging this PR against the ticket's ACs is awkward. It looks like an implementation ticket was implied but never created.

| Jira Requirement (PP-1804 spike) | PR Implementation | Status |
|---|---|---|
| Root causes of slow load identified and documented | Identified one (~50 redundant `/jobTasks` calls per page) and removed it; identified per-row React re-render cost and addressed via cell extraction + `React.memo(Cell)` and `React.memo(Row)` | ⚠️ Partial — fixes are shipping, but no documented spike findings / recommendations doc was attached to the ticket |
| Prioritised list of recommendations with effort estimates | Not in PR | ❌ Missing (may exist elsewhere — Confluence/ticket comments) |
| Feasibility of incremental/deferred loading for low-priority fields | Implicit yes — `useJobSearch` (Services column) is already gated by `enabled: !isBulkLoading`; new `OrderRowSkeleton` defers paint until row in viewport | ⚠️ Partial |
| Quick wins highlighted separately | This PR IS a quick win | ✅ Addressed |
| Findings shared with team in walkthrough/demo | Out of PR scope | — |
| Follow-up implementation tickets drafted | PP-1808 referenced (`Replace bottom-of-table spinner with dynamic row skeletons`) — its commit is bundled here, not in its own PR | ⚠️ See "PR scope" finding |

**Out-of-scope changes flagged:** the PR description claims a 14-line surgical fix to `ServicesCellContent`. The actual diff is **40 files, +1752 / -562**:

- 11 new column-cell components extracted from `tableColumns.tsx` (`CurrentJobCell`, `CustomerCell`, `DateCell`, `DeadlineCell`, `DeliverySpeedCell`, `FormatCell`, `IdCell`, `OrderSizeCell`, `SelectionCell`, `StatusCell`, `TeamCell`, `TeamScoreCell`, `ServicesCell`) — all wrapped in `React.memo`.
- `tableColumns.tsx` shrunk -422 / +60; `table.tsx` shrunk -131 / +69 as inline JSX moved out to the cell components.
- New `OrderRowSkeleton` partial replaces a bottom-of-table spinner with per-row skeletons — this is **PP-1808**, not PP-1804 (commit `d1beba8fe`).
- Selection cell decoupled from `tableColumns` so click latency cost stops cascading to the whole column factory (commit `e6a3ecd65`).

A reviewer working from the PR description alone will under-review this PR.

---

## Architecture Analysis

The actual change set decomposes into four logical pieces:

**1. The named change (`079a2e88d`).** `ServicesCellContent` was firing a per-row `useJobTaskQuery` to read `taskName` strings, ~50 requests per Orders page load. The PR replaces that with `serviceJob.description.split(" & ")`. The convention is established in this codebase:

- `apps/creative-portal/services/orders/utils.ts:73-75` already does `service.includes(" & ") ? service.split(" & ") : [service]` for the same purpose.
- `apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/OrderJobs/partials/JobCard.tsx:163` renders `job.description` verbatim for Service-type jobs.

So removing the extra fetch and deriving task names from `description` is consistent with how this codebase already treats service-job descriptions.

**2. Cell extraction + memoization (`324bd47cf`, `14d53181a`).** Each per-column cell is now its own component file under `cells/`, with `React.memo(...)` wrapping the export and a `displayName`. The intent is that when `RowComponent` (already `React.memo`-wrapped at `partials/Row/index.tsx:72`) re-renders for unrelated reasons, individual memoized cells short-circuit instead of running every cell's render logic per row per parent render.

**3. PP-1808 row skeletons (`d1beba8fe`).** Bottom-of-page spinner replaced with N actual `<tr>` skeleton rows (where `N = Math.min(MAX_SKELETON_ROWS, remaining)`) — better perceived progress on long lists. `MAX_SKELETON_ROWS = 5`. Column-specific skeletons via a `COLUMN_SKELETONS` map keyed by column id.

**4. Selection cell decoupling (`e6a3ecd65`).** `SelectionCell` lives in its own folder under `cells/SelectionCell/` (with a sibling `Header.tsx`); the `tableColumns` factory's selection-column hunk is now just a thin `createSelectionColumnCell` callback in `table.tsx` so a shift-select click no longer re-renders the entire column factory.

The architecture is sound. There are some real concerns about whether the memoization will actually deliver (see Issue 2 below), but the structural direction is good and the unrelated changes from earlier commits (`c6778c315`, `03a739a0c`) were rolled back when called out — net diff against `origin/develop` outside `TableWithFilters/` is empty.

---

## Issues Found

### 1. PR description severely understates scope

**[File: PR description on github.com/Proofed/B2BWebserver/pull/2268]**

**Function/Class:** N/A

**Severity:** medium

**Problem:** The PR title and body advertise a single-file change ("This PR switches `ServicesCellContent` to derive task names…"). The `Areas of Change` section lists only `ServicesCellContent/index.tsx` and its new test. Reality is **40 files, +1752 / -562**, including extraction of 13 cell components, a new `OrderRowSkeleton` partial belonging to ticket **PP-1808**, and a large refactor of `table.tsx` and `tableColumns.tsx`.

**Impact:** Reviewer is primed for a small targeted fix and will likely under-review. CI/test reviewers reading just the PR description will not know to check, e.g., the row-virtualisation skeleton math, the `React.memo` ergonomics, or the column-id sync between `OrderRowSkeleton` and `tableColumns`. Also makes git history misleading at a squash-merge level — the commit summary becomes "Drop redundant /jobTasks call" but the squashed change actually rewrites the Orders table.

**Fix:** Either (a) update the PR body to enumerate every change and link the bundled tickets explicitly, or (b) split off the PP-1808 work and the cell-extraction refactor into separate PRs that can be reviewed independently. Recommended split:

- PR-A: PP-1804 — Drop redundant /jobTasks call (commit `079a2e88d` only — the 2 files actually described)
- PR-B: PP-1804 — Memoize cell factories + decouple selection cell (commits `324bd47cf`, `14d53181a`, `e6a3ecd65`)
- PR-C: PP-1808 — Replace bottom-of-table spinner with row skeletons (commit `d1beba8fe`)

### 2. `React.memo` on cells likely under-delivers because the Cell closure spreads full `CellProps`

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/tableColumns.tsx]**

**Function/Class:** every `Cell:` callback in `getTableColumns`, e.g. lines 110-115, 122-125, 147-153, 161-163, 171-173, 194-199, etc.

**Severity:** medium

**Problem:** Each column's `Cell` is declared as `(cellProps) => <FooCell {...cellProps} isBulkLoading={isBulkLoading} />`. react-table v7 builds a fresh `cellProps` object per render that includes `cell`, `column`, `row`, `value`, and other refs. Several of those (notably `cell` and the cell-specific helpers) are reconstructed each render. Spreading the entire object into a `React.memo`-wrapped child means the shallow-equality check inside memo compares against fresh prop references for `cell`/`column` even when `row` and `value` are stable, defeating the memoization in the steady state.

The included `memoization.test.tsx` doesn't catch this — it builds `cellProps` manually with only `{ row, value }`:

```ts
// cells/__tests__/fixtures.ts:30-35
({
  row: { original: row },
  value,
  ...extra
}) as ...
```

So the test demonstrates that `React.memo(TeamCell)` works given a controlled, minimal `cellProps` object, but it doesn't prove that the production code path (full react-table cellProps spread) yields stable refs across renders.

**Impact:** The headline architectural payoff of this PR ("memoize cell factories to enable `React.memo(Row)`" per commit `324bd47cf`) may not materialise. The structural cost is paid (one folder + one types file per cell), but the runtime win may be negligible. The unit tests pass either way, so we can't tell from CI.

**Fix:** Destructure only the props each cell actually consumes inside the column factory, and pass them by name. Example for `FormatCell`:

```tsx
// instead of
Cell: (cellProps: CellProps<OrdersTableFlatRow, DocumentFormat | undefined>) =>
  <FormatCell {...cellProps} />

// do
Cell: ({ row, value }: CellProps<OrdersTableFlatRow, DocumentFormat | undefined>) =>
  <FormatCell row={row} value={value} />
```

And for cells that take `isBulkLoading`:

```tsx
Cell: ({ row }: CellProps<OrdersTableFlatRow, unknown>) =>
  <IdCell row={row} isBulkLoading={isBulkLoading} />
```

Then add a memoization test that drives a real `react-table` instance (or a faithful CellProps fake) across re-renders to assert the spy is called once, not on every parent re-render.

### 3. `description.split(" & ")` is silently destructive if a task name ever contains " & "

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/ServicesCellContent/index.tsx]**

**Function/Class:** `ServicesCellContent`, line 42-44

**Severity:** low

**Problem:** The new code splits `serviceJob.description` on the literal `" & "` separator. The old code used the authoritative `taskName` array from `/api/jobTasks`. If a service is ever named with `&` in its display name (e.g., `"Research & Development"` as a single task), the split breaks it into two `<li>`s.

The convention is established (`services/orders/utils.ts:73`, `JobCard.tsx:163`), so this is not new risk for the codebase — but it's an implicit coupling to a backend naming rule that has no compile-time or runtime check.

**Impact:** Silent mis-rendering if backend taxonomy ever adds an `&`-containing service name. No alarm; no test would catch the regression.

**Fix:** Add a short comment documenting the assumption, mirroring how `services/orders/utils.ts:73-75` guards itself with `service.includes(" & ") ? ... : [service]`. Example:

```ts
// `description` is the joined service-task list from /api/jobs/search, with
// " & " as the separator (same convention as services/orders/utils.ts and
// JobCard service-job headings). Assumes no individual task name contains
// " & " — if a backend rename ever does, we'll silently split that name.
const TASK_NAME_SEPARATOR = " & ";
```

Lower-effort alternative: type-narrow `description` to a tagged string template, or expose a single `getServiceTaskNames(description)` helper used by both call sites, so the convention has one definition.

### 4. No dedicated tests for 8 of the new cell components

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/cells/*]**

**Function/Class:** `CurrentJobCell`, `CustomerCell`, `DeadlineCell`, `DeliverySpeedCell`, `IdCell`, `OrderSizeCell`, `ServicesCell`, `StatusCell`, `TeamCell`, `TeamScoreCell`

**Severity:** low

**Problem:** Of the 13 new cell components, only 4 have dedicated tests (`DateCell`, `FormatCell`, `SelectionCell`, and `ServicesCellContent` via the partials layer). `TeamCell` is exercised indirectly by `__tests__/memoization.test.tsx`. The remaining 9 components have no unit tests, despite each containing real branching logic (group-header vs order row, `isBulkLoading` skeleton path, value derivation).

CLAUDE.md says: "All new code must have corresponding tests."

**Impact:** Coverage gap. The cells are thin wrappers, but each has at least one branch (group-header vs order) and a skeleton path that could regress unnoticed. Combined with Issue 2, there's no end-to-end guard that the memoization saves any work.

**Fix:** Add minimal tests per cell — at least asserting the group-header path returns null (or its specific partial), the order-row path renders the inner content component, and the `isBulkLoading: true` path renders the skeleton. The fixtures in `cells/__tests__/fixtures.ts` already exist and make this cheap.

### 5. `hasMoreItems` uses `<=` causing an off-by-one against `pageNumber * pageSize == data.length`

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/table.tsx]**

**Function/Class:** `Table`, line 145-148

**Severity:** low

**Problem:**

```tsx
const hasMoreItems = useMemo(
  () => !!(pageNumber * pageSize <= data.length),
  [pageNumber, pageSize, data.length]
);
```

When `pageNumber * pageSize` exactly equals `data.length`, `hasMoreItems` is `true` even though there is nothing left to load. `nextBatchSize` then becomes `min(MAX, 0) = 0`, so no skeletons render (good) — but `fetchMoreOnBottomReached` (line 445) will still increment `pageNumber` past the data length on scroll. `Array.slice` tolerates over-bounds indices, so this is functionally harmless, but it means `pageNumber` grows unboundedly while scrolling at the end of a fully-loaded list.

**Impact:** Minor wasted state updates; no user-visible bug.

**Fix:** Change to strict `<`:

```tsx
() => pageNumber * pageSize < data.length
```

### 6. `OrderRowSkeleton.COLUMN_SKELETONS` is hand-synced with `tableColumns.tsx` column ids

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/OrderRowSkeleton/index.tsx]**

**Function/Class:** `COLUMN_SKELETONS`, lines 12-31

**Severity:** low

**Problem:** The component carries a `// Keep in sync with tableColumns.tsx column ids.` comment because the skeleton map is keyed by string literals (`"format"`, `"id"`, `"customer"`, ...). The column ids in `tableColumns.tsx` are also string literals at each `id:` field. There is no shared source of truth, so renaming a column id silently disables that column's skeleton (the `?? null` fallback hides the breakage).

**Impact:** Drift risk during future refactors; a renamed column id silently downgrades to an empty skeleton cell.

**Fix:** Extract the column ids as named constants in `consts.tsx` (one place already has `COLUMN_CONFIG.SELECTION_COLUMN_ID`, `COLUMN_CONFIG.CHEVRON_COLUMN_ID` — extend the pattern):

```ts
// consts.tsx
export const ORDER_COLUMN_ID = {
  FORMAT: "format",
  ID: "id",
  CUSTOMER: "customer",
  ORDER_SIZE: "orderSize",
  // ...
} as const;
```

Then both `tableColumns.tsx` and `OrderRowSkeleton` import the same map. Bonus: at the call site, the `COLUMN_SKELETONS` map can be typed as `Record<keyof typeof ORDER_COLUMN_ID, ReactNode>` to make missing entries a TypeScript error.

### 7. `vi.mock("services/jobTasks", ...)` in `ServicesCellContent/index.test.tsx` is dead weight (mostly)

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/ServicesCellContent/index.test.tsx]**

**Function/Class:** the mock setup at lines 17-20 + the test at lines 121-136

**Severity:** low

**Problem:** The component no longer imports `services/jobTasks`. The mock is set up only so the final test can assert `mockUseJobTaskQuery` was not called. With no import path, this is technically a meaningful regression guard (re-introducing the import would cause the call) but the assertion is a tautology while the import isn't there.

**Impact:** Negligible — file just has 6 lines of mock that don't really exercise anything until/unless someone reintroduces the import. Probably fine to leave as a fingerprint test against regression.

**Fix:** Optional. If you want to keep the regression guard, add a comment explaining why the mock and assertion exist despite the import being removed:

```tsx
// Guard against accidentally re-introducing the per-row /jobTasks fetch:
// if a future change imports useJobTaskQuery again, this assertion will fail.
vi.mock("services/jobTasks", () => ({
  useJobTaskQuery: (...args: unknown[]) => mockUseJobTaskQuery(...args)
}));
```

Otherwise drop the mock and the final test.

### 8. Pre-existing lint failures in `packages/wysiwyg/` block merge per CLAUDE.md

**[File: packages/wysiwyg/src/extensions/comments/index.ts, packages/wysiwyg/src/components/molecules/CommentsContainer/utils.ts, packages/wysiwyg/src/contexts/EditorContext/hooks.ts]**

**Function/Class:** N/A — 62 `prettier/prettier` errors across three files

**Severity:** medium

**Problem:** `npx turbo run lint` fails with 62 prettier errors in the `@proofed/wysiwyg-editor` package. None of those files are touched by this PR (confirmed via `git diff origin/develop...HEAD -- packages/wysiwyg/` — empty). CLAUDE.md is explicit:

> Do NOT commit if any of these fail. Fix the issues first, even if they are in code you did not change. If a pre-existing failure is unrelated to your work, flag it to the team but still do not commit on top of it.

The PR description acknowledges pre-existing failures but only refers to **build** failures in `components/styles/Assets/*`. The current build is actually clean — but the lint failures the PR description doesn't mention are real.

**Impact:** Strictly per CLAUDE.md, the PR is not mergeable as-is. Practically, the failures are entirely unrelated and likely landed via a prior merge into `develop`.

**Fix:** Two options:
1. **Recommended:** Auto-fix the `packages/wysiwyg/` prettier issues as a sibling commit (or chore PR before this one) so the whole workspace lints clean. `yarn lint:fix` should resolve all 62 since they're prettier-only.
2. Flag the failure to the team and get explicit sign-off to merge on top.

The author should not silently merge while CLAUDE.md's pre-commit gate is failing.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ✅ | 157 test files, 1470/1470 tests pass (creative-portal). 4/4 workspace tasks successful. |
| `npx turbo run typecheck` | ✅ | 5/5 successful (0 type errors across creative-portal, customer-portal, shared, wysiwyg, storybook). |
| `npx turbo run lint` | ❌ | 62 `prettier/prettier` errors in `packages/wysiwyg/` — three files not touched by this PR. Pre-existing on develop. **See Issue 8.** |
| `npx turbo run build` | ✅ | 4/4 successful. (One turbo-config warning: "no output files found for task `@proofed/storybook#build`" — config-level, not a build failure.) |

---

## Tests

- ✅ `ServicesCellContent/index.test.tsx` (new, +137 lines, 6 tests) — covers happy path, single-task description, loading state, bulk loading, empty description, and the regression guard against `useJobTaskQuery`.
- ✅ `cells/SelectionCell/index.test.tsx` (new, +265) — covers selection, group-header, shift-select.
- ✅ `cells/DateCell/index.test.tsx` (new, +100).
- ✅ `cells/FormatCell/index.test.tsx` (new, +95).
- ✅ `cells/__tests__/memoization.test.tsx` (new, +92) — but only tests `TeamCell`, and uses a controlled `{ row, value }` cellProps fake. See Issue 2 — the test doesn't prove the production memoization works against real react-table cellProps.
- ✅ `partials/OrderRowSkeleton/index.test.tsx` (new, +108).
- ❌ No dedicated tests for `CurrentJobCell`, `CustomerCell`, `DeadlineCell`, `DeliverySpeedCell`, `IdCell`, `OrderSizeCell`, `ServicesCell`, `StatusCell`, `TeamScoreCell` (see Issue 4).
- ✅ All existing tests pass (1470/1470).
- ⚠️ No E2E test (Playwright) added. The PR's behavioural promise (zero `/api/jobTasks` requests on Orders page load) is verifiable manually but not automated. A network-assertion E2E would catch regression.

Validation suite was run against the PR branch — see Validation Checks above.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ The /jobTasks removal is correct; description-split convention is established elsewhere in the codebase |
| Regression risk | ⚠️ Medium — PR description hides ~38 files of refactor and PP-1808 mixed in. Cell memoization gain may not materialise in production (Issue 2). |
| Tests | ⚠️ 5 new test files added (good), but 9 cells lack dedicated tests, and the memoization test doesn't exercise the real react-table cell-render path |
| Code quality | ✅ Refactor direction is sound; well-structured; conventions followed; props in alphabetical order; `displayName` set; `React.memo` consistently applied |
| Validation suite | ❌ Lint fails (62 errors in `packages/wysiwyg/`) — pre-existing, unrelated, but blocking per CLAUDE.md |
| Mergeable state | ❌ Per CLAUDE.md ("Do NOT commit if any of these fail"). GitHub `mergeable_state: clean`, but local lint is failing on `packages/wysiwyg/` |

---

## Recommendation

**Request changes** (with the option of approval-with-suggestions if the lint and scope-description issues are addressed).

**Blockers (must address before merge):**
1. Fix the pre-existing `packages/wysiwyg/` prettier failures (run `yarn lint:fix` and commit) — or get explicit team sign-off — so `npx turbo run lint` is clean per CLAUDE.md (**Issue 8**).
2. Update the PR description to honestly describe the 40-file scope, OR split the PR into the three logical chunks (named change / refactor / PP-1808). Reviewers should not be primed for a 14-line surgical fix when the diff rewrites the Orders table (**Issue 1**).

**Strong suggestions (address before merge ideally):**
3. Validate that `React.memo(Cell)` actually short-circuits in production — either by destructuring `Cell:` closures down to `{ row, value }` (which mechanically guarantees stable refs) or by adding a profiling-style test that drives a real `useTable` instance and asserts cell render count is bounded across re-renders (**Issue 2**).
4. Add a comment documenting the `" & "` separator assumption in `ServicesCellContent` (**Issue 3**).

**Nice to have:**
5. Add minimal unit tests for the remaining 9 untested cells (**Issue 4**).
6. Tighten `hasMoreItems` to `<` (**Issue 5**).
7. Promote column ids to shared constants so `OrderRowSkeleton.COLUMN_SKELETONS` cannot silently drift from `tableColumns.tsx` (**Issue 6**).
