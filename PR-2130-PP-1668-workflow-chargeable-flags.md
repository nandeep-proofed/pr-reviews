# PR Review: fix/PP-1668: Fix Workflow default chargeable flags for Review and QA jobs

**PR:** https://github.com/Proofed/B2BWebserver/pull/2130
**Jira:** https://proofed.atlassian.net/browse/PP-1668
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Hourly Review → **Chargeable** | Base template `DEFAULT_JOB_TASK_CONFIGURATION_TEMPLATES` Review changed `false→true` (line ~322), inherited by hourly extended via spread | ✅ Addressed |
| Per-word Review → **Not chargeable** (keep as-is) | `DEFAULT_JOB_TASK_CONFIGURATION_TEMPLATES_PER_WORDS_EXTENDED` has its own Review at line 2211 with `chargeable: false` — not touched by base change | ✅ Correct, no regression |
| QA on all workflows → **Not chargeable** | Hourly extended QA (line ~1117): `true→false`. Per-words extended QA (line ~2057): `true→false` | ✅ Addressed |

---

## Architecture Analysis

Clean, minimal changes — only boolean flag flips in config constants. The base template `DEFAULT_JOB_TASK_CONFIGURATION_TEMPLATES` is spread into `DEFAULT_JOB_TASK_CONFIGURATION_TEMPLATES_PER_HOURLY_EXTENDED`, so changing Review in the base correctly affects hourly. The per-words extended template has its own explicit Review definition with `false`, so it's unaffected.

---

## Issues Found

### 1. Base template change affects hourly extended only by accident

**[File: apps/creative-portal/components/pages/partners/[partnerId]/projects/[projectId]/settings/consts.tsx]**
**Function/Class:** DEFAULT_JOB_TASK_CONFIGURATION_TEMPLATES
**Severity:** low
**Problem:** The base template's Review is changed to `chargeable: true`. This works because hourly extended spreads the base and doesn't override Review, while per-words extended has its own explicit Review with `false`. However, this relationship is implicit.
**Impact:** If someone later removes the per-words Review override, it would inherit `true` from the base — silently breaking per-word behavior.
**Fix:** Add a comment in the base template noting this dependency: `// NOTE: Per-words extended overrides this to chargeable: false`

### 2. Base template has no direct test coverage

**[File: apps/creative-portal/components/pages/partners/[partnerId]/projects/[projectId]/settings/__tests__/consts.test.ts]**
**Function/Class:** Test suite
**Severity:** low
**Problem:** Tests only verify `_PER_HOURLY_EXTENDED` and `_PER_WORDS_EXTENDED`. The base template's Review chargeable flag is not directly tested.
**Impact:** Since the base is only used as a spread source, the extended tests implicitly cover it. But a direct assertion would be more robust.
**Fix:** Add a test case for `DEFAULT_JOB_TASK_CONFIGURATION_TEMPLATES` Review chargeable values directly.

### 3. Excessive mocking in test file

**[File: apps/creative-portal/components/pages/partners/[partnerId]/projects/[projectId]/settings/__tests__/consts.test.ts]**
**Function/Class:** Test suite
**Severity:** low
**Problem:** The test file has 11 `vi.mock()` calls to stub out transitive dependencies of `consts.tsx`. This is brittle — if `consts.tsx` gains a new JSX import, the test breaks.
**Impact:** Maintenance burden. Any new import in `consts.tsx` requires updating the test mocks.
**Fix:** Acceptable trade-off for testing a deeply-nested config file. No change needed, but worth noting.

### 4. Merge conflicts

**[File: N/A]**
**Function/Class:** N/A
**Severity:** high
**Problem:** The PR has merge conflicts with the current develop branch (`mergeable_state: dirty`).
**Impact:** Cannot be merged until conflicts are resolved.
**Fix:** Rebase the branch onto latest `develop` and resolve conflicts.

---

## Tests

- ✅ Hourly extended Review chargeable: `true` for all speed types
- ✅ Hourly extended QA chargeable: `false` for all speed types
- ✅ Per-words extended Review chargeable: `false` for all speed types
- ✅ Per-words extended QA chargeable: `false` for all speed types
- ✅ `vitest.config.ts` path aliases added for test resolution
- ❌ Base template not directly tested

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ All three acceptance criteria met |
| Regression risk | ✅ Low — config-only changes, per-word Review unaffected |
| Tests | ✅ Present, cover key scenarios |
| Code quality | ✅ Clean, minimal changes |
| Mergeable state | ❌ Dirty — needs rebase |

---

## Recommendation

**Approve after resolving merge conflicts**
1. Rebase onto latest `develop` and resolve conflicts
2. Consider adding a comment in base template noting the per-words override dependency
