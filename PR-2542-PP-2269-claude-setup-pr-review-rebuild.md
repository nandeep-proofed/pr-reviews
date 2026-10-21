# PR Review: chore/PP-2269: Restructure Claude Code setup, add codebase rules and rebuild /pr-review

**PR:** https://github.com/Proofed/B2BWebserver/pull/2542 (head `0d6e77fd6`, author nandeep-proofed)
**Jira:** https://proofed.atlassian.net/browse/PP-2269
**Status:** In Progress

Reviewed 14 Oct 2026, static only (validation skipped by the reviewer). Scope: 69 files. No app code changes; `packages/shared` gains four test files only.

**Updated 21 Oct 2026 (head `3985e53c6`):** all 19 issues are resolved: 15 fixed in `3985e53c6`, and 4 settled by the author's decision (#4 implemented as "Claude never pushes to `develop`"; #15, #16 and #17 kept as designed). Each issue below carries a **Status** line, and "Resolution" at the end has the summary, the Windows checks still owed and a follow-up `/security-check` on the new head.

**Updated 14 Oct 2026 (same head):** added a second pass on whether the PR keeps every rule from develop's `CLAUDE.md`, `.cursor/rules` and the skills it deletes. See "Rules migration coverage" and issues #15–#19.

---

## What this means for users (non-technical summary)

No end user of the creative portal, customer portal or Auth Service is affected; this is developer tooling. For **developers**:

1. On a Windows machine, the shared package's test run now fails (32–35 new tests), so anyone touching `packages/shared` on Windows sees red tests that are not theirs.
2. On Windows, `/release-tag` can't cut a release any more: the new script can't start `yarn`, and its error wrongly says `develop` doesn't compile.
3. After this merges, Claude can push straight to `develop` without asking. `develop` has no branch protection, and a push there deploys all three apps to devtest.
4. Re-running `/pr-review` after the author pushes can forget that an earlier comment was a Blocker. It can then offer to dismiss your "Request changes" review while that Blocker is still open.
5. On Windows, `/pr-review` never posts committable suggestions, because its suggestion check can't start Prettier, TypeScript or ESLint. This fails safely: the suggestions are left out.
6. `/pr-review` no longer runs the tests or the build on the PR. It reads the PR's automatic checks instead, and this repo has none on pull requests, so a PR with failing tests can get a review with no Blocker.
7. Before a release, nothing runs the full test suite, lint or a dependency audit any more: `/release-check` is deleted, and `/release-tag` only checks that `develop` compiles and builds.
8. Claude can read Figma designs only through the community Figma connection. If its token is wrong or expired, as it is on this machine today, Claude can't read any design.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
| --- | --- | --- |
| Claude follows the team rules in each area | 12 path-scoped rules in `.claude/rules/`; most named helpers/paths verified to exist | ⚠️ Partial — a few facts are wrong (#12); the move from `CLAUDE.md` / `.cursor/rules` dropped one convention and two checks (#15, #16, #18) |
| `/pr-review` → single review with line comments, posted only after approval | Private draft → "Post review" gate → one review; verified in workflow + gh-helpers | ⚠️ Partial — re-run and dedupe defects (#3, #5, #6, #7); suggestion check dead on Windows (#8) |
| `/implement` runs a self-review before committing | `implement/SKILL.md:112-125` (built-in `/code-review`, test-per-branch, reuse check) | ✅ Addressed (not exercised live, per the PR) |
| Blocked from secret files; asks before merge, force-push, branch delete | env/settings.local deny rules ✅; merge ask ✅; force-push ask ✅ | ⚠️ Partial — `:branch` delete rule unproven; `gh pr merge --merge` without a number isn't denied; push to `develop` is unprompted (#4) |
| Credential or badly formatted commit message is rejected | `.husky/pre-commit` + `scripts/check-staged-secrets.sh`, `.husky/commit-msg` + `scripts/check-commit-message.sh` | ✅ Addressed (hooks work; their tests fail on Windows, #1) |
| Automation documented in `docs/CLAUDE_AUTOMATION.md` | 301-line doc + doc-coverage test | ⚠️ Partial — guardrail claims at `:76-77` don't match the rules (#4) |
| No change to application behaviour | No app code; shared tests aren't exported or imported | ✅ Addressed |

**Beyond the ticket:**
- `/release-tag` logic moved into `scripts/release-tag.mjs` (457 lines).
- The official `figma` MCP server was removed from `.mcp.json`; `figma-console` is now the only Figma server (#17).
- The lint-staged change adds `--max-warnings 0`, plus Prettier for `scripts/`, `.claude/`, CSS and JSON.
- `.markdownlintignore` added.

The first two are not in the ticket. The rest are reasonable; the Figma change needs the team's agreement (#17).

---

## Architecture Analysis

`CLAUDE.md` is slimmed, and task guidance moves into rule files that load only when Claude touches matching paths. `/pr-review` becomes a multi-agent Workflow script (`.claude/workflows/pr-review-audit.js`):

- the built-in `/code-review` is the bug finder;
- rule-driven file-group agents check the diff against the rules;
- an adversarial verifier checks each finding;
- a private draft review is submitted only after the user approves.

`gh-helpers.mjs` wraps `gh`, and `check-suggestions.mjs` proves committable suggestions in a throwaway worktree. Git hooks add a secret scan and a commit-message check. The new tests sit in `packages/shared` and drive the scripts directly.

The design is sound, and the shell handling is careful: argument arrays everywhere, no `shell: true`, and ids and SHAs validated. The defects are concentrated in two places:

1. **Windows portability.** The team's setup is Git for Windows with `core.autocrlf=true` in the system gitconfig, and the repo has no `.gitattributes`.
2. **The `/pr-review` re-run path**, which the PR says has not been exercised on a real PR.

---

## Rules migration coverage

The PR body says of `CLAUDE.md`: "Nothing is dropped, only moved." This pass checks that claim for develop's `CLAUDE.md`, the five `.cursor/rules/*.mdc` files and the skills the PR deletes.

**Method:**
- Rule-by-rule comparison of develop (`bfc2d0747`) with the PR head.
- Develop hasn't changed `CLAUDE.md`, `.cursor/` or `.claude/` since the PR's merge base (`545ff88a8`), so nothing added to develop since then is at risk.
- A finder agent compared the Cursor files; every rule it reported lost was re-checked against the code.

| Source on develop | Where it is in the PR | Result |
| --- | --- | --- |
| `CLAUDE.md` Code Style: component folder, `index.tsx` UI-only, props order, JSX spread, styles co-location, styled-component hygiene | `.claude/rules/components.md` | ✅ All moved |
| `CLAUDE.md` Storybook Guidelines | `.claude/rules/storybook.md` | ✅ Word for word. It drops a reference to `COMPONENT_FOLDER_ISSUES.md`, which doesn't exist |
| `CLAUDE.md` Environment Configuration & Validation (PP-2119) | `.claude/rules/env-config.md` | ✅ Word for word |
| `CLAUDE.md` design-fidelity discipline | `figma-ui` skill | ⚠️ Points 2–5 word for word; point 1 reversed (#17) |
| `CLAUDE.md` code-review-graph setup, extras and gaps | `docs/CODE_REVIEW_GRAPH.md` | ✅ |
| `CLAUDE.md` Reuse-first conventions, Git Workflow, Pre-Commit Verification, Key Architectural Notes | `CLAUDE.md` | ✅ Kept; the `LoadingWrapper` path is corrected to `molecules/` |
| `.cursor/rules/*.mdc` (2,546 lines) | `components.md`, `api-routes.md`, `pages-ssr.md` and the other rules | ⚠️ One convention lost (#18) and two scripts left undocumented (#19); the rest is covered, generic, or contradicted by the code |
| `/commit` skill | `CLAUDE.md` "Git Workflow" + the `commit-msg` hook | ✅ Except the "show the message and ask" step (see below) |
| `/test` skill, "Test Requirements for PRs" | `testing.md` "What needs a test" | ✅ |
| `/review-pr` checklist | `pr-review/checklist.md` + the rule files | ⚠️ The validation run is gone (#15); the new-dependency licence check too (#19) |
| `/release-check` skill | deleted | ❌ Pre-release checks lost (#16) |

**Changed on purpose, and correct against the code:**
- `FC<Props>` becomes `FC<ButtonProps>`-style names. 524 component `types.ts` / `types.d.ts` files use `<Name>Props`; none use plain `Props`.
- "Styled Components + MobX" for the customer portal is dropped. There are no `mobx` imports and one `styled-components` import (`packages/shared/hooks/useMaintenanceToast/CountdownDisplay.tsx:4`).
- `/implement` no longer logs work to `TASKS_COMPLETED.md`, which doesn't exist.
- Most of the Cursor files is sample code that contradicts the repo, and is right to go. Examples:
  - `withIronSessionSsr` (0 uses) and `apps/*/middleware.ts` (none);
  - Prettier at width 80 with single quotes (the repo uses 70 and double quotes);
  - a barrel `packages/shared/index.ts` (none) and kebab-case file names;
  - theme keys `primaryDark` / `error`, which aren't in `packages/shared/theme/theme.tsx`.
- **Commits and pushes.** The working rule "Commit and push: after completing significant work, commit and push" (develop `CLAUDE.md:240`) is removed, and so is `/commit`'s step that showed the message and asked first. `git commit` and `git push` are now allowed with no prompt (`settings.json:12-13`), and `/implement` commits by itself. The push half is #4.

**`yarn bump-packages` is still required:**
- `/pr-create` keeps "Run `yarn bump-packages` before PR if not already done (ask first)" (`pr-create/SKILL.md:56`).
- So do the PR template, `DEVELOPMENT_WORKFLOW.md:224` and `docs/CONTRIBUTING.md:55`.
- `/implement` never mentioned it.
- Only `/release-check`'s `yarn bump-packages:rc` is gone, and `/release-tag` already bumps versions through `yarn push-release`.

---

## Issues Found

### 1. The new tests fail on Windows checkouts

> **Status (21 Oct 2026):** ✅ Fixed in `3985e53c6`: `.gitattributes` (LF for `*.mjs`, `*.cjs`, `*.sh`, `.husky/*`), tests build paths with `path.resolve`, `sh` tests skipped where `sh` is missing. Tested on macOS only; needs one Windows run.

**[File: packages/shared/claudeScripts.test.ts, packages/shared/claudePrReview.test.ts, packages/shared/gitHooks.test.ts]**

> **In plain terms:** A developer on Windows who runs the shared package's tests now gets dozens of failures that have nothing to do with their change. The project rule says not to commit while tests fail, so this blocks normal work in `packages/shared` on Windows.

**Function/Class:** the release-tag, check-suggestions/gh-helpers and git-hook test suites

**Severity:** Blocker (a failing test gate; Windows machines only)

**Confidence:** high (reproduced)

**Steps to reproduce:**

1. On Windows with Git for Windows' default `core.autocrlf=true`, check out the PR branch.
2. Run `yarn package:shared test claude`, from Git Bash and then from PowerShell.
3. **Expected:** 130 tests pass, as they do on macOS/Linux.
4. **Actual:**
   - From Git Bash: 32 tests fail with `SyntaxError: Invalid or unexpected token`, and 3 more fail on paths.
   - From PowerShell: 31 tests also fail with `spawnSync sh ENOENT`, because `sh` is not on the PowerShell PATH.

**Problem:** There are three separate causes.

- **(a) Line endings.** The `.mjs` scripts begin with `#!/usr/bin/env node`. They are checked out with CRLF (`git ls-files --eol` → `i/lf w/crlf`), and Vitest can't parse a CRLF shebang in a module outside the package root.
- **(b) Hard-coded POSIX paths.** Three tests use POSIX paths that `path.resolve` turns into `D:\repo\…`. Examples: `insideWorktree("/repo/.pr-review/1-abc", …)` at `claudePrReview.test.ts:1371-1384`, the `restyleFile("/repo", …)` test, and the format-hook test.
- **(c) `sh` on the PATH.** `spawnSync("sh", …)` at `gitHooks.test.ts:51`, `gitHooks.test.ts:138` and `claudeScripts.test.ts:202` needs `sh` on the PATH.

**Evidence:** I copied the real `claudeScripts.test.ts` and the CRLF `scripts/release-tag.mjs` into a scratch folder and ran them there:

- release-tag block with CRLF: **8 failed** with `SyntaxError`;
- the same block after converting to LF: **9 passed**.

`node -e` with `path.resolve("/repo/.pr-review/1-abc","apps/a/x.ts")` gives `D:\repo\.pr-review\1-abc\apps\a\x.ts`, so the `startsWith` check is false. `Get-Command sh` in PowerShell finds nothing.

**Impact:** The shared workspace's test gate is red on Windows. No CI runs tests on PRs, so nothing else will catch it.

**Fix:**
- Add a `.gitattributes`. Developers who already have the files checked out then need to check them out again to get LF:
  ```gitattributes
  *.mjs text eol=lf
  *.sh text eol=lf
  .husky/* text eol=lf
  ```
- Build the expected paths in the three tests with `path.resolve` / `path.join`.
- Skip the `sh` tests when `sh` isn't available, or resolve Git's `sh.exe`.

### 2. `/release-tag` can't run `yarn` on Windows

> **Status (21 Oct 2026):** ✅ Fixed in `3985e53c6`: `yarn` runs through a shell on Windows only (`needsShell`); a step that never started reports `could not start: ENOENT`, and `push-release` says nothing was tagged. Tested on macOS only; needs one Windows run.

**[File: scripts/release-tag.mjs]**

> **In plain terms:** A developer on Windows can no longer cut a release with `/release-tag`. The run stops halfway with a message saying `develop` doesn't compile, which is false and sends them looking for a problem that isn't there.

**Function/Class:** `step`, `pushRelease`

**Severity:** High

**Confidence:** high (reproduced)

**Steps to reproduce:**

1. On Windows, run `/release-tag oms-x.y.z`, or `node scripts/release-tag.mjs oms-x.y.z`.
2. **Expected:** typecheck, build, tag and push, the same as the old skill.
3. **Actual:** `git checkout develop` and `git pull` run, then "Step failed: health check (develop must compile…) (exit null)".

**Problem:** `spawnSync("yarn", …)` without a shell can't find `yarn.cmd` on Windows. Affected lines:

- `release-tag.mjs:240` (`step`);
- `:253` (`pushRelease`);
- the calls at `:437-442`.

`step` also reports only `result.status` (null) and hides `result.error` (ENOENT).

**Evidence:** I ran `spawnSync("yarn", ["--version"])` on this machine:

- without a shell: `status null`, `ENOENT`;
- with `shell: true`: `1.22.22`.

`docs/RELEASE_TAG.md:175-197` documents the PowerShell, cmd and Git Bash setup, so Windows is a supported release machine. The old skill ran these commands through Bash, where `yarn` resolves.

**Impact:** This is a regression: releases can't be cut from Windows. `--dry-run` never starts yarn, so the preview passes and only the real run fails.

**Fix:** Pass `shell: process.platform === "win32"` to the yarn calls. This is safe because the tag is checked against `TAG_PATTERN` first. Also include `result.error?.code` in the step's error.

### 3. A re-run forgets that an earlier comment was a Blocker

> **Status (21 Oct 2026):** ✅ Fixed in `3985e53c6`: `threads` reads each comment's severity (`threadSeverity`), `previousStillOpen` keeps it, and the workflow returns `stillOpenBlockers`. The template keeps Request changes and blocks the dismiss offer while one is open.

**[File: .claude/skills/pr-review/gh-helpers.mjs, .claude/workflows/pr-review-audit.js, .claude/skills/pr-review/output-template.md]**

> **In plain terms:** You review a PR and post "Request changes" for a serious problem. The author then pushes an unrelated change, and you re-run the review. The new review can come back as "Approve with suggestions" and offer to dismiss your old "Request changes", while the serious problem is still in the code.

**Function/Class:** `threads`, `threadTitle`, the `previousStillOpen` build, and the dismiss rule in the template

**Severity:** High

**Confidence:** high (from code; the PR says this path hasn't run on a real PR)

**Steps to reproduce:**

1. `/pr-review` posts a 🔴 Blocker on file A, and the review is Request changes.
2. The author pushes a change to file B only, and `/pr-review` runs again (incremental).
3. **Expected:** the Blocker on A is still open, so the recommendation stays Request changes.
4. **Actual:**
   - the Blocker becomes a "still open" item with no severity;
   - the new findings have no Blocker, so the recommendation drops;
   - the skill offers to dismiss the old review.

**Problem:** The severity is lost between the comment and the re-run:

- The comment format puts the severity on its own label line (`output-template.md:85`, `_<kind>_ | _<severity>_ | _<effort>_`) and the title on a separate `**<title>**` line (`:87`).
- `threadTitle` (`gh-helpers.mjs:149-151`) reads only the first `**…**` line, and `threads` (`:203-218`) stores no severity.
- `previousStillOpen` (`pr-review-audit.js:1005-1047`) therefore has none either.
- `output-template.md:26-27` then says: "holds a `CHANGES_REQUESTED` review by `me` and this run has no Blocker … Offer to dismiss it".

**Impact:** The merge can be unblocked while a Blocker is still open. Dismissing needs the user's yes, but the user is told it's safe.

**Fix:** Parse the severity label in `threads` and carry it into `previousStillOpen`. Count still-open Blockers in both the recommendation and the dismiss check.

### 4. Push and merge guardrails don't match what the docs promise

> **Status (21 Oct 2026):** ✅ Fixed in `3985e53c6` (decision: Claude must not push to `develop`): PreToolUse hook `scripts/claude-guard-push.mjs` refuses `develop`, `HEAD:develop`, `:develop`, `--all`, `--mirror` and a bare push on `develop`; the `:*` rule is replaced by the hook asking before a `:branch` delete; merge deny rules now catch `gh pr merge --merge` with no number; `CLAUDE_AUTOMATION.md` corrected. ⚠️ Branch protection on `develop` needs a repo admin, not done.

**[File: .claude/settings.json, docs/CLAUDE_AUTOMATION.md]**

> **In plain terms:** After this PR, Claude can push straight to `develop` without asking. A push there deploys all three apps to devtest, and nothing on GitHub stops it. Two other "safety" rules don't do what the documentation says either.

**Function/Class:** permission rules

**Severity:** Medium

**Confidence:**
- high for the unprompted push and the merge gap;
- medium for exactly how the `:branch` rule behaves (not tested live; the PR says these rules weren't re-tested).

**Steps to reproduce:**

1. In a session with the PR's settings, on a local `develop` with commits, have Claude run `git push`. Variants: `git push origin HEAD:develop`, or `/pr-create` started on `develop`.
2. **Expected:** a prompt, the same as before this PR, when every push needed approval.
3. **Actual:** `Bash(git push:*)` (`settings.json:13`) runs it without a prompt.

**Problem:** Three rules don't do what the docs say.

- **Unprompted pushes to `develop`.** `git push` moved from "prompt every time" to "allowed". `gh api …/branches/develop` returns `"protected": false`, the rulesets list is `[]`, and all three deploy workflows fire on a push to `develop`.
- **The `:branch` delete rule.** `Bash(git push * :*)` (`settings.json:27`) can't express a literal colon, because a trailing `:*` is Claude Code's legacy wildcard suffix. So either it never fires and `git push origin :develop` (delete) is unprompted, or it fires on every two-argument push. Either way, `CLAUDE_AUTOMATION.md:77` ("`:branch` … asks" and "`git push -u origin <branch>` still runs without a prompt") can't be true on both counts.
- **The squash-only deny rules.** The deny rules `gh pr merge * --merge*` / `-m` / `--rebase` / `-r` (`settings.json:80-85`) need an argument before the flag. `gh pr merge --merge` (the current branch's PR) falls through to "ask". The repo allows merge commits and rebase merges (`allow_merge_commit: true`), and the doc says these were "Tested".

**Impact:** The "safer by default" acceptance criterion is weaker than documented. The worst case is an unreviewed push that deploys.

**Fix:**
- Ask a repo admin to protect `develop` (require a PR). That is the real guard, and it covers people too.
- Add ask rules for `develop` pushes, e.g. `Bash(git push*develop*)`.
- Replace the colon rule with a PreToolUse hook check, or drop the claim.
- Add the four deny rules `Bash(gh pr merge --merge:*)`, `Bash(gh pr merge -m:*)`, `Bash(gh pr merge --rebase:*)` and `Bash(gh pr merge -r:*)`.
- Correct `CLAUDE_AUTOMATION.md:76-77`.

### 5. A run whose bug review failed is still drafted, and its footer says the bug review ran

> **Status (21 Oct 2026):** ✅ Fixed in `3985e53c6`: a failed bug review sets `incomplete.bugReviewFailed`, nothing is drafted, and `bugReviewSource` is `"failed"`. A failed Jira check is now named in "In short" and the footer.

**[File: .claude/workflows/pr-review-audit.js]**

> **In plain terms:** If the main bug-finding step fails, the review is still prepared and offered for posting with only the style and rules comments. Its footer says the bug review ran. The reviewer thinks the PR was checked for bugs when it wasn't.

**Function/Class:** `lensesFailed`, `isIncomplete`, `bugReviewSource`

**Severity:** Medium

**Confidence:** high

**How to spot it:** This is not reproducible from the UI; it needs the convert or behaviour agent to fail. Evidence:

- `isIncomplete` (`pr-review-audit.js:1368`) is `notChecked.length > 0 || cutShort.length > 0`. It ignores `lensesFailed` (`:629-635`).
- `bugReviewSource` (`:618-620`) is `"code-review"` whenever the `/code-review` text exists, even when converting it failed.
- `output-template.md` never renders `lensesFailed`. Only the return shape at `full-audit.md:144` lists it.

**Problem:** A failed bug review isn't treated as an incomplete run, and the user never sees that it failed.

**Impact:** This contradicts the PR's claim that an incomplete run is never drafted.

**Fix:** Add `lensesFailed.length > 0` to `isIncomplete`, or render `lensesFailed` in the summary and footer.

### 6. Two different findings on the same line are merged into one

> **Status (21 Oct 2026):** ✅ Fixed in `3985e53c6`: same-line findings merge only when their titles share at least half their words; different issues stay separate.

**[File: .claude/workflows/pr-review-audit.js]**

> **In plain terms:** If two reviewers flag different problems on the same line, for example a missing test and a bug, the second one's title, fix and code are thrown away. If the first one then turns out to be wrong, both disappear.

**Function/Class:** `addFindings`

**Severity:** Medium

**Confidence:** high

**How to spot it:** This is code health, not reproducible from the UI. `pr-review-audit.js:651` keys on `` `${finding.file}:${finding.line}` `` only. On a collision, `:653-672` keeps the first finding and appends only `problem` (`existing.problem | finding.problem`). It can also raise the severity under the first finding's title.

**Problem:** `full-audit.md:179` says to dedupe on "same file + line + issue", but the code drops the "issue" part.

**Impact:** Real findings are silently lost or mislabelled.

**Fix:** Key on file, line and lens/title, or keep collisions as separate items.

### 7. Re-runs can post comments that were already made (plausible)

> **Status (21 Oct 2026):** ✅ Fixed in `3985e53c6`: re-run findings that repeat an open thread, or sit in a file outside the incremental scope, are dropped in code (`repeatsDropped`); the convert prompt no longer contradicts the shared context.

**[File: .claude/workflows/pr-review-audit.js]**

> **In plain terms:** On a second review of the same PR, the tool can post the same comments again on lines it already commented on.

**Function/Class:** `convertTask`

**Severity:** Medium

**Confidence:** medium. Whether duplicates appear depends on which instruction the agent follows; nothing in code prevents them.

**How to spot it:**

- `/code-review` always reviews the whole PR (`full-audit.md:31`).
- The convert prompt (`pr-review-audit.js:525`) says "Don't add issues of your own, and don't drop any", while the shared context (`:379-381`) says "do not report again".
- `previous` is used only at `:379`, `:745` and `:748`. No code filters the converted findings against earlier threads or against `changedSinceLastReview`.

**Problem:** On a re-run, duplicate detection rests entirely on two prompt instructions that contradict each other.

**Impact:** Duplicate inline comments land on threads that are already open.

**Fix:** Before verifying, drop findings whose file and title match a `previous` thread. When incremental, also drop files outside `changedFiles`.

### 8. The suggestion check never works on Windows

> **Status (21 Oct 2026):** ✅ Fixed in `3985e53c6`: tools start from their Node entry points (`TOOL_ENTRIES`); real symlink (never a junction), unlinked before the worktree is removed; a failed link no longer stops the check. Run end to end on macOS (valid suggestion passed, a type error failed, `node_modules` untouched); needs one Windows run.

**[File: .claude/skills/pr-review/check-suggestions.mjs]**

> **In plain terms:** On Windows, `/pr-review` can never post a one-click "commit suggestion", because the step that checks suggestions can't start its tools. Nothing breaks; suggestions are just always left out.

**Function/Class:** `run`, `bin`, `tsconfigFor`

**Severity:** Medium

**Confidence:** high

**How to spot it:**

- `run` (`:118-125`) calls `spawnSync(bin("tsc"|"eslint"|"prettier"))` without a shell. `bin` is `node_modules/.bin/<name>`, which is an extensionless `sh` shim (only the `.cmd` variants run natively). The result is ENOENT, the same mechanism as #2.
- `symlinkSync` (`:215`) needs Developer Mode on Windows (EPERM without it).
- The workflow drops every suggestion when `checked:false` (`pr-review-audit.js:1337-1345`).

**Problem:** On Windows the check can never run, so it always returns `checked:false`.

**Impact:** A headline feature doesn't work for Windows reviewers. It fails safely.

**Fix:**
- Run `process.execPath` with the JS entry points (`typescript/bin/tsc`, `eslint/bin/eslint.js`, `prettier/bin/prettier.cjs`).
- **Don't switch the link to a junction**: `git worktree remove --force` (`:369`) follows a junction and deletes the real `node_modules`. The finder reproduced this in a scratch repo, and we had the same incident on 27 Aug. Unlink the links before removing the worktree.

### 9. The format-on-edit hook does nothing on Windows

> **Status (21 Oct 2026):** ✅ Fixed in `3985e53c6`: backslashes converted to `/` before the path check. Needs one Windows run.

**[File: scripts/claude-format-on-edit.sh]**

> **In plain terms:** The new "format each file Claude edits" step never runs on Windows. The commit hook still formats files, so the only effect is unformatted diffs between commits.

**Function/Class:** the path check

**Severity:** Low

**Confidence:** high (reproduced)

**How to spot it:** `scripts/claude-format-on-edit.sh:25` has `case "$file" in "$root"/*)`. Here `root` is `D:/Gaurav/2.0/B2BWebserver` (from `git rev-parse`) and Claude's `file_path` is `D:\Gaurav\…`. I tested it: `NO_MATCH`.

**Problem:** On Windows the path check never matches, so the hook exits before running Prettier.

**Impact:** Edits Claude makes on Windows stay unformatted until commit.

**Fix:** Normalise the path first: `file=$(printf '%s' "$file" | tr '\\' '/')`.

### 10. The suggestion check claims more than it checks

> **Status (21 Oct 2026):** ✅ Fixed in `3985e53c6`: `lintErrorsFrom` / `typeErrorsFrom` return null when the tool crashed or didn't start, and that fails the check. Token comment, doc and PR body now say the OS keyring is not isolated.

**[File: .claude/skills/pr-review/check-suggestions.mjs]**

> **In plain terms:** The tool says a suggested fix "passes ESLint" even when ESLint crashed and never checked it. The code comment also says the PR's config runs without your GitHub access, which isn't fully true.

**Function/Class:** `lintErrors`, `scrubbedEnv`

**Severity:** Low

**Confidence:** high

**How to spot it:**

- **ESLint status.** `:253-258` runs `JSON.parse(result.stdout || "[]")` with no status check. If ESLint exits 2 with empty stdout for both the baseline and the check, the suggestion is "ok".
- **Token scrubbing.** `:105-116` says "It runs without the reviewer's tokens", but it only filters env variable names. `gh` keeps its login in the OS keyring, so PR-supplied `.eslintrc.js` / `.prettierrc.js` code can still call `gh auth token`. This is the same exposure as running the PR's tests locally, so the risk isn't new; the claim is just wrong.

**Problem:** Two of the check's claims are stronger than what the code does.

**Impact:**
- A suggestion can be posted as passing ESLint when ESLint never ran.
- The comment and the PR body overstate the token isolation.

**Fix:**
- Treat `status === 2` or `result.error` as "ESLint did not run".
- Reword the comment and the PR body, or skip the check when the PR changes any lint/prettier config.

### 11. A malformed Slack webhook is printed in full

> **Status (21 Oct 2026):** ✅ Fixed in `3985e53c6`: `webhookProblem` validates the URL first; `slackFailure` reports only an error code, never the message. Leak reproduced on Node 24 before the fix.

**[File: scripts/release-tag.mjs]**

> **In plain terms:** If the Slack webhook setting has a typo, such as stray quotes or no `https://`, the release script prints the full secret webhook address on screen and into the Claude transcript.

**Function/Class:** `notifySlack`

**Severity:** Low

**Confidence:** high (reproduced)

**How to spot it:** `release-tag.mjs:366-369` prints `error.message`. For an unparsable URL, Node's `fetch` message is `Failed to parse URL from "https://hooks.slack.com/services/…"`. I reproduced this on Node 22.22.1.

**Problem:** The error path echoes the secret value.

**Impact:** The secret leaks into the terminal and the transcript. It only happens when the value is malformed.

**Fix:** Validate with `new URL(webhook)` first, and print a fixed message instead of `error.message`.

### 12. A few facts in the rule files are wrong

> **Status (21 Oct 2026):** ✅ Fixed in `3985e53c6`: Tiptap facts (`^3.10.7`, 3.11.0 in the lock) in `dependencies.md` and `CLAUDE.md`; reference components now `organisms/OrderStatusCardInfo` and `molecules/HistoryTable`; `envs.ts` note corrected; globs added for `packages/shared/libs/**`, `apps/*/setup/**` and `apps/*/setup/storybook/**`.

**[File: .claude/rules/dependencies.md, .claude/rules/components.md, .claude/rules/utils-hooks.md, .claude/rules/storybook.md]**

> **In plain terms:** A few of the new "how we do things" notes give Claude wrong facts: a wrong editor version, example folders that don't exist, and a config file one app doesn't have. Claude may follow them.

**Function/Class:** rule text

**Severity:** Low

**Confidence:** high

**How to spot it:**
- **Tiptap pin.** `dependencies.md:24` (and `CLAUDE.md`) say "Tiptap 3.10.7 … pinned". Root `resolutions` are `^3.10.7`, and `yarn.lock` resolves 3.11.0. This was carried over from the old `CLAUDE.md`.
- **Reference component paths.** At `components.md:17`, `molecules/OrderStatusCardInfo` is really `organisms/`, and `tables/HistoryTable` is really `molecules/HistoryTable`. `SearchBar` uses `styles.tsx` and `types.d.ts`, which the same rule forbids. This was also carried over.
- **`config/envs.ts`.** `utils-hooks.md:40` says each app has `config/envs.ts`, but the creative portal doesn't.
- **Unmatched folders.** No `paths:` glob matches `packages/shared/libs/**` or `apps/*/setup/**`. The second holds the Storybook decorators that `storybook.md:12` points to.

**Problem:** These statements in the rule files don't match the code.

**Impact:** Claude may follow a wrong fact, or miss a rule when editing files the globs don't match.

**Fix:** Correct the four statements, and add the two globs.

### 13. Two docs still describe the old flow

> **Status (21 Oct 2026):** ✅ Fixed in `3985e53c6`: `DEVELOPMENT_WORKFLOW.md` puts Pull Request (6) before Code Review (7); `test-runner.md` no longer names deleted skills.

**[File: DEVELOPMENT_WORKFLOW.md, .claude/agents/test-runner.md]**

> **In plain terms:** The workflow guide says to run the code review before opening the PR. The new review only works on an open PR, so following the guide in order stops at that step.

**Function/Class:** docs

**Severity:** Low

**Confidence:** high

**How to spot it:**
- **Step order.** `DEVELOPMENT_WORKFLOW.md:25-28` (and sections 6/7) put "Code Review (with /pr-review)" before "Pull Request (with /pr-create)". But `pr-review/SKILL.md:33` says "If the branch has no open PR, say so and stop".
- **Deleted skills.** `test-runner.md:3` still names the deleted `commit` and `test` skills.

**Problem:** These docs weren't updated for the new skills.

**Impact:** A developer following the guide in order hits a stop, and the agent description points at skills that no longer exist.

**Fix:** Swap the step order, and update the agent description.

### 14. The secret scan reads the whole repo when nothing is staged

> **Status (21 Oct 2026):** ✅ Fixed in `3985e53c6`: the scan exits when nothing is added or modified. New test (commit a fake secret, stage nothing or a clean file, expect exit 0); confirmed it fails when the scan covers the whole index.

**[File: scripts/check-staged-secrets.sh, packages/shared/gitHooks.test.ts]**

> **In plain terms:** On a commit with no staged files (an `--amend` of the message only, or `--allow-empty`), the credential check scans the entire repository instead of the change. Today nothing matches, so it's harmless. If a token-shaped string is ever committed, unrelated commits will be blocked with a confusing message.

**Function/Class:** the scan pipeline

**Severity:** Low

**Confidence:** high

**How to spot it:**
- **Empty input.** GNU `xargs` runs once with no file arguments: `printf '' | xargs -0 echo` prints `RAN_WITH_NO_ARGS`. The pipeline then runs `git grep --cached … --` over the whole index.
- **No test catches it.** The finder mutated the script to always scan the whole index, and all 31 tests in `gitHooks.test.ts` still passed. `gitHooks.test.ts:180-195` never commits a file holding a fake secret.

**Problem:** With nothing staged, the scan's scope silently becomes the whole repository.

**Impact:** None today. A token-shaped string committed in the future would block unrelated commits.

**Fix:**
- Use `xargs -0 -r`, or exit early when the staged list is empty.
- Add a test: commit a fake secret, then stage nothing / stage a clean file, and expect exit 0.

### 15. `/pr-review` no longer runs tests, typecheck, lint or build, and PRs have no CI

> **Status (21 Oct 2026):** ⏸️ Kept by decision: `/pr-review` reads CI checks and does not run tests itself.

**[File: .claude/skills/pr-review/SKILL.md, .claude/skills/pr-review/full-audit.md]**

> **In plain terms:** The old review ran the tests and the build on the PR. The new one only reads the PR's automatic checks, and this repo has none on pull requests. A PR whose tests fail can get a review with no Blocker; the review only notes "No CI checks on this PR".

**Function/Class:** the B.6 validation step

**Severity:** Medium

**Confidence:** high

**How to spot it:**
- **Old:** `/review-pr` had "B.6 Run Validation Checks" (`review-pr/SKILL.md:324`) and "ALWAYS run `npx turbo run test`, `typecheck`, `lint`, and `build` against the PR branch … Any failure is a hard blocker" (`:685`).
- **New:** "Don't run tests, typecheck, lint or build: read the PR's CI checks (B.6)" (`pr-review/SKILL.md:77`). When no checks are reported, it says so, and that is "Not a Blocker by itself" (`full-audit.md:247`).
- **No PR CI:** the workflows in `.github/workflows/` fire on pushes to `develop`, on tags or by manual dispatch; none has a `pull_request` trigger. The PR-check workflow is #2489, which the PR body says is unmerged and failing at install.

**Problem:** The validation step now depends on CI checks that this repo doesn't run on PRs yet.

**Impact:** Until #2489 merges, nothing in the review flow runs the tests on a PR.

**Fix:** When `gh pr checks` reports no checks, fall back to running the affected checks locally (scoped as in `CLAUDE.md` "Pre-Commit Verification"). Or merge #2489 before this PR.

### 16. Deleting `/release-check` drops the pre-release checks

> **Status (21 Oct 2026):** ⏸️ Kept by decision: `/release-check` stays deleted.

**[File: .claude/skills/release-check/SKILL.md (deleted), CLAUDE.md]**

> **In plain terms:** Before a release, nothing now runs the full test suite, lint or a dependency security audit. `/release-tag` only checks that `develop` compiles and builds, so a release can go out with failing tests unless someone runs them by hand.

**Function/Class:** the release flow

**Severity:** Medium

**Confidence:** high

**How to spot it:**
- **The deleted skill:**
  - ran `npx turbo run test`, typecheck and lint (`release-check/SKILL.md:26`);
  - searched for `TODO` / `FIXME` / `HACK` (`:38`);
  - checked `BUGS_FOUND.md`, which exists, for open critical bugs (`:39`);
  - flagged `@ts-ignore` / `@ts-nocheck` (`:40`);
  - ran `yarn audit` (`:42`).
- **`/release-tag`** runs only `yarn typecheck` and `yarn build` (`release-tag/SKILL.md:15`).
- **`CLAUDE.md:173`** still says to run the full suite "before a release", but no skill or script does it now. The PR calls `/release-check` "unused".

**Problem:** The release flow lost its test, lint and audit gate.

**Impact:** Failing tests, lint errors or a known-vulnerable dependency can reach an rc branch unnoticed.

**Fix:** Add `yarn test` and `yarn lint` to the health check in `scripts/release-tag.mjs`, before it branches. Or keep `/release-check` and have `/release-tag` run it first.

### 17. The Figma rule is reversed and the official server removed

> **Status (21 Oct 2026):** ⏸️ Kept by decision: `figma-console` is the only Figma server (more tools, key-based).

**[File: .claude/skills/figma-ui/SKILL.md, .mcp.json]**

> **In plain terms:** Today Claude reads Figma designs through Figma's official connection, and uses a second, community connection only when the first fails. After this PR it can use only the community one. If that connection's token is wrong or expired, Claude can't read any design until someone fixes it, and that is the situation on this machine today.

**Function/Class:** design-fidelity rule 1

**Severity:** Medium

**Confidence:** high

**How to spot it:**
- **Develop:** `CLAUDE.md:245-251` says "Always try the official `figma` server first — it's the canonical path; `figma-console` is the escape hatch, not the default."
- **PR:** `figma-ui/SKILL.md:14` says "`figma-console` is the only Figma server in `.mcp.json`; don't use another Figma server or connector", and the PR deletes the `figma` entry from `.mcp.json`.
- **This session:** `figma-console` failed to connect with HTTP 401 (token rejected), while the official `figma` server connected.

**Problem:** This is a policy change that isn't in the ticket, and it removes the fallback that works.

**Impact:** With one expired or missing token, Figma-driven UI work (`/implement`, the `ticket-context` agent, `figma-ui`) can't fetch designs. The rule says to report the failure and stop, so the work waits on a token fix.

**Fix:** Agree the change with the team first. Unless there is a reason to drop the official server, keep both servers in `.mcp.json` and restore develop's order: official first, `figma-console` when it fails.

### 18. The `forwardRef` `displayName` convention was dropped

> **Status (21 Oct 2026):** ✅ Fixed in `3985e53c6`: `components.md` asks for `<Component>.displayName` with `forwardRef`.

**[File: .claude/rules/components.md]**

> **In plain terms:** Most of our shared components that pass a ref also set a display name, so they show up by name in React's developer tools and in test output. The new rules don't mention it and lint doesn't check it, so new components will quietly skip it.

**Function/Class:** `components.md` "Folder and files"

**Severity:** Low

**Confidence:** high

**How to spot it:**
- **The Cursor rules** set it in their component samples (`component-patterns.mdc:66` `Button.displayName = 'Button'`, and `:125`).
- **The code:** 36 of the 42 non-test files that call `forwardRef` set `X.displayName`, e.g. `packages/shared/components/molecules/Modal/index.tsx:75`. The other six are `Fields/Upload`, `Fields/Upload/FormikUpload`, the shared `Dropdown` and three `packages/wysiwyg` components (`Input`, `Textarea`, `Dropdown`).
- **The PR:** `components.md:16` says only "`forwardRef` only when a ref is needed".
- **Lint:** `packages/eslint-config/index.js` doesn't set `react/display-name`, and a finder's `eslint --print-config` showed it resolves to "off".

**Problem:** The PR says nothing was dropped, but this convention is missing.

**Impact:** New ref-forwarding components show up as an unnamed `ForwardRef` in React DevTools and in test output.

**Fix:** Add to `components.md:16`: "set `<Component>.displayName` when the `forwardRef` callback is anonymous."

### 19. Small gaps the migration left

> **Status (21 Oct 2026):** ✅ Fixed in `3985e53c6`: `dev:customer-portal-ssl` and `test:e2e-ui` listed in `CLAUDE.md` Commands; `dependencies.md` asks for a size, maintenance and licence check on a new package.

**[File: .cursor/rules/README.mdc, .cursor/rules/monorepo-best-practices.mdc (deleted), .claude/rules/dependencies.md]**

> **In plain terms:** Two working shortcuts are no longer written down anywhere: running the customer portal over HTTPS locally, and the visual Playwright test runner. The review also no longer asks whether a new package's size, upkeep and licence are acceptable.

**Function/Class:** docs and rule text

**Severity:** Low

**Confidence:** high

**How to spot it:**
- **Scripts.** `package.json:33` (`dev:customer-portal-ssl`) and `:29` (`test:e2e-ui`) exist. At the PR head, no file other than `package.json` mentions either. On develop, only the deleted Cursor files did.
- **New-dependency check.** The old review checklist asked to "check size/maintenance/license" for a new package (`review-pr/SKILL.md:652`). `dependencies.md:32` keeps only "search first".

**Problem:** Two pieces of guidance were in files the PR deletes, and weren't carried over.

**Impact:** Minor: the scripts are harder to discover, and a heavy or badly licensed package is less likely to be questioned in review.

**Fix:**
- List both scripts in `CLAUDE.md` "Commands" or the README.
- Add "check its size, maintenance and licence" to `dependencies.md:32`.

---

## Open Questions

- Does `figma-console` expose `figma_get_variables` and `figma_export_tokens`? The PR renamed the old `figma_get_token_values` to these. I couldn't confirm the names because `figma-console` failed to connect in this session (HTTP 401). — `.claude/skills/figma-ui/SKILL.md:12`
- Does `Bash(git push * :*)` never fire, or fire on every two-argument push? A one-minute test in a session with the PR's settings would settle #4's exact wording. — `.claude/settings.json:27`

---

## Validation Checks

| Check | Result | Notes |
| --- | --- | --- |
| `npx turbo run test` | Skipped — user opted out | Partial evidence from the review: the new release-tag tests **fail** on a Windows CRLF checkout (reproduced, #1). A finder ran all 130 on LF with POSIX `sh`, and all passed except the 3 Windows-path cases. |
| `npx turbo run typecheck` | Skipped — user opted out | A finder ran `tsc --noEmit -p packages/shared` with the PR's config: exit 0. |
| `npx turbo run lint` | Skipped — user opted out | A finder ran `eslint --max-warnings 0` on the four test files with the PR's config: exit 0. |
| `npx turbo run build` | Skipped — user opted out | No app code changed. |

The repo runs no CI on pull requests, so nothing has validated this branch apart from the author's local runs.

---

## Tests

- ✅ Four new test files (130 tests) cover the commit-message hook, the secret patterns, release-tag argument and Slack logic, the doc-coverage list and the workflow with fake agents.
- ❌ They fail on Windows checkouts (#1).
- ⚠️ No test covers the secret-scan scope (#14) or the re-run severity carry-over (#3).
- ⚠️ The PR says the re-run, Request-changes, committable-suggestion and `/implement` self-review paths have not been exercised on a real PR.

### Suggested manual QA script

This is developer tooling, so QA means a developer on a Windows machine:

1. Check out the branch and run `yarn package:shared test claude` from Git Bash and from PowerShell. All should pass (#1).
2. Run `node scripts/release-tag.mjs oms-0.0.0 --dry-run`, then a real run on a throwaway tag in a fork. It should reach the build step (#2).
3. In a session with the PR's settings, ask Claude to run `git push --dry-run origin HEAD:develop` and `gh pr merge --merge --help`. Each should prompt or be denied (#4).
4. Run `/pr-review` on a PR with a Blocker, push an unrelated change, and run it again. The recommendation should stay Request changes (#3).
5. Edit a `.ts` file through Claude on Windows. It should be reformatted (#9).

---

## Summary

| Aspect | Status |
| --- | --- |
| Correctness | ⚠️ Re-run path defects (#3, #5–#7) |
| Regression risk | ⚠️ Medium: `/release-tag` on Windows (#2); shared tests on Windows (#1) |
| Tests | ❌ Fail on Windows; two gaps |
| Accessibility | n/a |
| Error handling | ⚠️ #2 hides ENOENT; #5 hides a failed bug review |
| Security | ⚠️ Unprompted push to unprotected `develop` (#4). `/security-check` was run by the author on `1fd162f6e`; the later commit `0d6e77fd6` was not re-checked. |
| Code quality | ✅ Careful shell handling (argument arrays, id and SHA validation); rules mostly accurate |
| Rules migration | ⚠️ Nearly complete: one convention lost (#18), the review and release validation gone (#15, #16), the Figma policy reversed (#17) |
| Validation suite | Skipped — user opted out |
| Mergeable state | ✅ Clean (merge-tree with `develop` at `bfc2d0747` has no conflicts; 17 commits behind, none touching these files) |

---

## Recommendation

**Request changes.** #1 is a failing test gate on Windows machines.

1. Fix #1: add `.gitattributes` (LF for `*.mjs`, `*.sh`, `.husky/*`), use platform paths in the 3 tests, and handle a missing `sh`.
2. Fix #2: run `yarn` through a shell on Windows in `scripts/release-tag.mjs`.
3. Fix #3: carry the severity of earlier threads into the recommendation and the dismiss offer.
4. #4: ask an admin to protect `develop`, add the `develop`-push ask rules and the four merge deny rules, and correct `CLAUDE_AUTOMATION.md:76-77`.
5. Pre-merge asks: #5–#8, plus from the rules migration:
   - #15: a local validation fallback in `/pr-review` until PR CI exists;
   - #16: put the test and lint gate back into the release flow;
   - #17: agree the Figma change with the team, or keep the official server.
6. The rest (#9–#14, #18, #19) are optional, low-effort clean-ups.
7. Before merge, re-run the suite on a Windows checkout and `/security-check` on the final head.

---

## Resolution (21 Oct 2026, head `3985e53c6`)

| # | Issue | Status |
| --- | --- | --- |
| 1 | Tests fail on Windows | ✅ Fixed, needs a Windows run |
| 2 | `/release-tag` can't run `yarn` on Windows | ✅ Fixed, needs a Windows run |
| 3 | Re-run forgets an earlier Blocker | ✅ Fixed |
| 4 | Push and merge guardrails | ✅ Fixed (decision: no Claude push to `develop`); branch protection needs an admin |
| 5 | Failed bug review still drafted | ✅ Fixed |
| 6 | Same-line findings merged | ✅ Fixed |
| 7 | Re-runs repeat comments | ✅ Fixed |
| 8 | Suggestion check on Windows | ✅ Fixed, needs a Windows run |
| 9 | Format hook on Windows | ✅ Fixed, needs a Windows run |
| 10 | Suggestion check overclaims | ✅ Fixed |
| 11 | Slack webhook printed | ✅ Fixed |
| 12 | Wrong facts in rules | ✅ Fixed |
| 13 | Docs describe the old flow | ✅ Fixed |
| 14 | Secret scan reads the whole repo | ✅ Fixed |
| 15 | `/pr-review` doesn't run tests | ⏸️ Kept by decision |
| 16 | `/release-check` deleted | ⏸️ Kept by decision |
| 17 | Figma: `figma-console` only | ⏸️ Kept by decision |
| 18 | `forwardRef` `displayName` | ✅ Fixed |
| 19 | Undocumented scripts, dependency checks | ✅ Fixed |

**Open questions:** `figma-console` does expose `figma_get_variables` and `figma_export_tokens`. The `Bash(git push * :*)` rule was removed; the push guard now asks before a `:branch` delete.

**Tests:** the four automation test files now hold 162 tests (from 130), all passing on macOS. Shared typecheck exits 0, and ESLint `--max-warnings 0` is clean on the test files.

### Still to check on Windows

1. Re-checkout the scripts once in Git Bash so they get LF:
   ```bash
   git ls-files -z '*.mjs' '*.cjs' '*.sh' '.husky/*' | xargs -0 rm -f
   git checkout -- '*.mjs' '*.cjs' '*.sh' '.husky/*'
   ```
   Then `yarn package:shared test claude`: all pass in Git Bash; in PowerShell the `sh` tests are skipped and nothing fails (#1).
2. `node scripts/release-tag.mjs oms-0.0.0 --dry-run` finishes; ideally one real run on a throwaway tag in a fork (#2).
3. `/pr-review` on a PR that gets a committable suggestion posts it (#8).
4. Claude edits a badly spaced `.ts` file, and it is reformatted right after (#9).
5. `git commit --allow-empty -m "test: Empty"` passes without scanning (#14).
6. Not Windows-specific: Claude running `git push origin develop` is refused (#4).

### Follow-up `/security-check` on `3985e53c6`

Verdict **NEEDS ATTENTION**: 2 medium, 4 low, no critical or high. None of these is fixed yet; they await the author's decision.

- **Medium:** a tag push (`git push origin oms-…`, `git push --tags`, `HEAD:refs/tags/…`) deploys to devtest with no prompt, and the push guard only covers `develop`.
- **Medium:** the suggestion check runs the PR's own ESLint and Prettier config, which is JavaScript, with the reviewer's user rights (token env vars are removed; `~/.ssh` and the `gh` keyring are not protected).
- **Low:** the suggestion check follows a symlinked PR file when it writes.
- **Low:** `gh-helpers.mjs edit-review` uploads any local file; only `edit-comment` is limited to `.pr-review/`.
- **Low:** the push guard can be fooled by shell quoting (`dev""elop`).
- **Low:** the secret scan misses `sk-ant-`, `sntrys_`, `ATATT3`, `npm_` and `ghr_` tokens.
- **Suggested:** a narrow Bash allow-list for `review-lens` and `review-verifier`, and moving `git commit` / `git push` to `ask`. These are policy decisions.
