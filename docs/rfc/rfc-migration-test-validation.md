# RFC: Migration Test Validation

**Status:** Draft  
**Date:** 2026-04-17  
**Author:** Daniele Antonini

---

## Summary

Introduce two test-execution phases to `monorepo-import` that are **enabled by default**: a **pre-import phase** that runs each source repo's test suite before the history is rewritten, and a **post-import phase** that re-runs the same suite inside the monorepo after all imports, `resolve_workspace_refs()`, and `pnpm install` have completed. Either phase can be suppressed with `--no-pre-test` / `--no-post-test`; both can be suppressed together with `--no-test`. Failures in either phase are fatal and abort the script immediately.

---

## Motivation

`monorepo-import` restructures a repository's file tree, rewrites its history, renames its `package.json` `name` field, migrates its lockfile, and adjusts internal dependency references. Any of these transformations could silently break a test suite. Today the script provides no way to verify that:

1. The source repo's tests were passing *before* the migration started (a failing baseline invalidates any post-migration verdict).
2. The tests still pass *after* the repo is absorbed into the monorepo workspace.

Without these checkpoints, teams must run tests manually before and after every import, or accept that regressions may go undetected. Because `monorepo-import` is a local developer tool, test output is available immediately and the feedback loop is tight — running tests by default imposes no friction beyond the time the suites take.

---

## Goals

1. Run each source repo's test suite in its original environment before any transformation, and abort immediately if they fail.
2. Re-run all imported packages' test suites inside the monorepo workspace after `resolve_workspace_refs()` and `pnpm install` complete, so cross-package dependency wiring is fully resolved before tests execute. Abort immediately on any failure.
3. Make both phases independently suppressible for exceptional cases (known-failing baseline, test-free package batch import).
4. Detect whether a package has a runnable test script and skip gracefully when it does not, rather than treating the absence of tests as a failure.
5. Keep the existing `--no-install` fast-path intact: when `--no-install` is passed, the post-import phase runs a targeted `pnpm install --filter <name>` per package to ensure workspace symlinks are in place before testing.

## Non-goals

- Interpreting or fixing test failures. The script surfaces the failure and stops; remediation is the developer's responsibility.
- Running tests for packages that were already present in the monorepo before this import run (only newly imported packages are tested).
- Environment variable injection. The script inherits the caller's shell environment; orchestration of secrets or external services is the developer's responsibility.
- Providing a timeout mechanism. Test suite duration is unconstrained; callers that need a deadline should wrap the script.

---

## Proposed CLI changes

### New flags

| Flag | Default | Description |
|---|---|---|
| `--no-test` | — | Suppress both pre-import and post-import test phases. Equivalent to `--no-pre-test --no-post-test`. |
| `--no-pre-test` | — | Suppress the pre-import phase. Useful when the source repo's baseline is already known-good. |
| `--no-post-test` | — | Suppress the post-import phase. Useful when post-migration testing is deferred to CI. |

Both phases are active unless explicitly suppressed. There is no `--test` flag; omitting the suppress flags is how tests are enabled.

### Flag interaction matrix

| Flags passed | Pre-import test | Post-import test |
|---|---|---|
| *(none — default)* | run | run |
| `--no-test` | skip | skip |
| `--no-pre-test` | skip | run |
| `--no-post-test` | run | skip |
| `--no-pre-test --no-post-test` | skip | skip |

### Updated usage block

```
Options:
  ...
  --no-test            Skip both pre-import and post-import test phases.
  --no-pre-test        Skip running the source repo's tests before importing.
  --no-post-test       Skip running the imported packages' tests inside the monorepo.
  ...
```

---

## Algorithm changes

### Phase 1 — pre-import test (per repo, inside the existing loop)

Phase 1 inserts immediately before `import_repo()` in the per-repo loop:

```bash
for each <repo>:
  name = derive_package_name(repo)

  # Phase 1 — pre-import test
  if not --no-pre-test:
    pre_test_result = run_pre_import_tests(repo, name)
    if pre_test_result == SKIP:
      warn "<name>: no test script found, skipping pre-import test"
    elif pre_test_result == FAIL:
      die "<name>: pre-import tests failed — fix source repo before migrating"
    # PASS falls through

  import_repo(repo, name, ...)   # existing pipeline, unchanged
```

### Phase 2 — post-import test (after all repos, after resolve and install)

Phase 2 runs once after the full import loop, after `resolve_workspace_refs()` and `pnpm install`:

```bash
# existing post-loop steps
resolve_workspace_refs()

if not --no-install:
  pnpm install
  git add pnpm-lock.yaml && git commit -m "chore: add pnpm-lock.yaml"

# Phase 2 — post-import tests for all newly imported packages
if not --no-post-test:
  for each newly imported <name>:
    post_test_result = run_post_import_tests(name)
    if post_test_result == SKIP:
      warn "<name>: no test script found in monorepo package, skipping post-import test"
    elif post_test_result == FAIL:
      die "<name>: post-import tests FAILED — monorepo state is intact, fix and re-run pnpm --filter <pkg_name> test"
    # PASS falls through

# existing: --extract-configs (runs after tests)
```

Running Phase 2 after `resolve_workspace_refs()` and `pnpm install` ensures that `workspace:*` references are resolved and all symlinks are in place before any package's tests execute. This is the correct baseline for validating that the migration preserved test correctness in a workspace context.

### `run_pre_import_tests(repo, name)` detail

```bash
run_pre_import_tests(repo, name):
  tmp = mktemp -d
  gh repo clone <repo> $tmp/pretest -- --no-local
  cd $tmp/pretest

  # Detect package manager from lockfile
  pm = detect_package_manager()   # pnpm-lock.yaml → pnpm, yarn.lock → yarn, else npm

  # Check for test script
  test_script = jq -r '.scripts.test // empty' package.json
  if [[ -z "$test_script" || "$test_script" == "echo \"Error: no test specified\"" ]]:
    rm -rf $tmp; return SKIP

  # Install and run
  $pm install --frozen-lockfile   # (or --immutable for yarn berry)
  $pm test
  exit_code=$?

  rm -rf $tmp
  return exit_code == 0 ? PASS : FAIL
```

The pre-import clone is separate from the import clone created by `import_repo()`. Both happen in isolated temp directories and neither interferes with the other.

### `run_post_import_tests(name)` detail

```bash
run_post_import_tests(name):
  cd <monorepo>

  # Resolve the scoped package name used as the pnpm filter
  pkg_name = jq -r '.name' <target_dir>/<name>/package.json
  # e.g. "@translated/tos-connector-notification-service"

  # Check for test script
  test_script = jq -r '.scripts.test // empty' <target_dir>/<name>/package.json
  if [[ -z "$test_script" || "$test_script" == "echo \"Error: no test specified\"" ]]:
    return SKIP

  # When --no-install was passed, workspace symlinks may be absent.
  # Run a targeted install scoped to this package before testing.
  if OPT_NO_INSTALL:
    pnpm install --filter "$pkg_name"

  pnpm --filter "$pkg_name" test
  return $? == 0 ? PASS : FAIL
```

---

## Test command detection

The script treats a package as having a runnable test suite when **all** of the following are true:

1. `package.json` contains a `scripts.test` key.
2. The value is not the npm placeholder `"echo \"Error: no test specified\""` (written by `npm init` when no test framework is configured).
3. The value is not empty.

If any condition fails, the phase is skipped with a `[test-skip]` warning. This avoids false failures on packages that have no tests yet.

---

## Error handling and exit codes

| Situation | Behaviour | Exit code |
|---|---|---|
| Pre-import tests fail | `die` immediately; no import performed for this or subsequent repos | 1 |
| Pre-import tests skip (no script) | Warn and continue import | — |
| Post-import tests fail | `die` immediately; monorepo git state is intact | 1 |
| Post-import tests skip (no script) | Warn and continue | — |
| `pnpm install --filter` (for `--no-install` path) fails | Treated as post-import test failure; `die` | 1 |
| Test process receives signal | Signal propagated; script exits with that signal | — |

The monorepo git state is never rolled back on a test failure. All import commits are already in the history when Phase 2 runs. The developer fixes the failing package and re-runs `pnpm --filter <pkg_name> test` directly to verify the fix without re-running the full import.

---

## Interaction with existing flags

| Flag combination | Notes |
|---|---|
| `--dry-run` | Both test phases are printed as log lines but not executed, consistent with the existing `run()` wrapper behaviour. |
| `--no-install` | Phase 2 runs a targeted `pnpm install --filter <name>` per package before testing. Phase 1 is unaffected (uses source repo's own package manager in isolation). |
| `--extract-configs` | Config extraction runs *after* Phase 2 passes. A failing post-import test prevents config extraction from running. |

---

## Open questions

1. **Pre-import test failures as warnings vs. hard stops for subsequent repos** — The current proposal calls `die` on the first pre-import failure, aborting the entire run. An alternative is to fail the specific repo's import but continue importing the remaining repos, collecting all failures and reporting them at the end. This trades early-abort clarity for a more complete failure report on a multi-repo run. Deferred until a concrete need arises.
