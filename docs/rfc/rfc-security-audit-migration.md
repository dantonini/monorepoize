# RFC: Security Audit Migration

**Status:** Implemented  
**Date:** 2026-04-17  
**Author:** Daniele Antonini

---

## Summary

Introduce a security audit awareness layer to `monorepo-import` that covers three independent concerns: a **pre-import audit phase** that runs the source repo's high/critical vulnerability check before any transformation; a **post-import audit phase** that runs the monorepo-level pnpm audit after all imports and `pnpm install` complete; and a **CI/pipeline detection step** that scans each source repo for audit commands in GitHub Actions workflows, Dockerfiles, Makefiles, and shell scripts, and emits a migration note with the equivalent pnpm command. The pre-import phase is **non-blocking by default** (warns on failure); the post-import phase is **blocking by default** (fails on regression). CI detection is always informational.

---

## Motivation

Several source repos enforce a vulnerability policy — typically "no HIGH or CRITICAL vulnerabilities in production dependencies" — via an audit command in their CI pipeline:

- yarn (classic): `yarn audit --level high --groups dependencies`
- pnpm: `pnpm audit --audit-level high --prod`

When a repo is imported into a monorepo, two problems arise:

1. **Lockfile regeneration can change vulnerability exposure.** `monorepo-import` migrates the source repo's lockfile to pnpm and then re-runs `pnpm install` in the workspace context. The resolver may select different transitive dependency versions than the original lockfile, potentially introducing HIGH/CRITICAL vulnerabilities that were absent before the migration, or resolving away others. Without an explicit check, these changes are invisible.

2. **CI audit commands are not automatically migrated.** The source repo's GitHub Actions workflow (or Dockerfile or Makefile) contains an audit invocation tailored to the original package manager. After migration the command is wrong (`yarn audit` no longer applies; the workspace uses pnpm). The `monorepo-import` script does not currently touch CI files, so these stale commands are left silently broken. Full automation of CI migration is out of scope (audit commands appear in many locations and formats), but the script can at least detect them and tell the developer what to update.

---

## Goals

1. Run each source repo's audit command (detected from CI/scripts or inferred from package manager) before importing, capturing the vulnerability baseline in its original environment.
2. After all imports and `pnpm install`, run `pnpm audit --audit-level high --prod` at the monorepo root and block on any HIGH/CRITICAL finding that was not present in the pre-import baseline.
3. Detect audit command invocations in common CI/pipeline locations in each source repo and emit a migration note with the pnpm equivalent, so developers know what to update manually.
4. Keep both audit phases independently suppressible.
5. Leave the existing test phases and all other phases unaffected.

## Non-goals

- Automatically rewriting GitHub Actions workflows, Dockerfiles, or Makefiles. The correct target command depends on context (workspace root vs. package-scoped invocation, matrix jobs, caching setup) and cannot be reliably inferred.
- Fixing vulnerabilities found by the audit. The script reports and stops; remediation is the developer's responsibility.
- Auditing packages that were already in the monorepo before this import run.
- Providing a tolerance threshold beyond HIGH/CRITICAL (i.e., no `--audit-level moderate` support is added).
- Running audit for packages without a lockfile (e.g., private packages with no published dependencies).

---

## Background

### Audit command variants seen in source repos

```bash
# yarn classic (most common in current repos)
yarn audit --level high --groups dependencies

# pnpm (some newer repos)
pnpm audit --audit-level high --prod
```

Both commands exit non-zero if any finding at the specified level or above is present.

### Where audit commands appear in CI

In order of frequency:

| Location | Pattern |
|---|---|
| `.github/workflows/*.yml` | `run: yarn audit …` or `run: pnpm audit …` |
| `Dockerfile` | `RUN yarn audit …` |
| `Makefile` | `audit: yarn audit …` |
| `scripts/*.sh` | `yarn audit …` or `pnpm audit …` |
| `package.json` scripts | `"audit": "yarn audit --level high --groups dependencies"` |

### pnpm workspace audit behaviour

`pnpm audit` audits the entire workspace from the monorepo root; it does not accept `--filter` to scope to a single package. The `--prod` flag restricts findings to production dependencies (excludes `devDependencies`). This means the post-import audit always reflects the merged dependency graph, which is the correct baseline for a monorepo.

---

## Proposed approach

### Capability A — Pre-import audit phase

Runs once per source repo, in the same temporary clone used by the existing pre-import test phase (or a new isolated clone if `--no-pre-test` is set), **after** `$pm install --frozen-lockfile` has completed. Running after install ensures the installed tree matches what the audit will inspect, producing results consistent with the repo's own CI.

Detects the package manager from the lockfile and runs the appropriate command with JSON output:

```bash
# yarn classic (>= 1.22 required for stable --json output)
yarn audit --level high --groups dependencies --json

# pnpm
pnpm audit --audit-level high --prod --json

# npm (fallback)
npm audit --audit-level high --omit=dev --json
```

**yarn version requirement:** yarn < 1.22 produced inconsistent and undocumented JSON audit output. The script enforces a minimum of yarn 1.22 for the pre-import audit phase; if the source repo pins an older yarn version, the phase is skipped with a `[audit-skip]` warning. All current repos using yarn classic already pin 1.22.x so this requirement is benign in practice.

**Behavior on failure:** Non-blocking by default. A HIGH/CRITICAL finding in the source repo is a pre-existing condition — it predates the migration and blocking on it would prevent importing repos that have known/accepted vulnerabilities awaiting a fix. The failure is logged clearly as `[audit-warn]` and recorded internally so Phase B can compute a delta.

`--audit-strict` (opt-in flag) promotes pre-import audit failures to fatal, for teams that want to enforce a clean baseline before any migration proceeds.

### Capability B — Post-import audit phase

Runs once at the workspace root, after all repos are imported, `resolve_workspace_refs()` completes, and `pnpm install` succeeds:

```bash
cd <monorepo>
pnpm audit --audit-level high --prod --json
```

`pnpm audit` does not accept `--filter` and always audits the entire workspace regardless of the starting directory. Running once at the workspace root is therefore equivalent to running per-package, and avoids redundant network round-trips.

**Incremental-import guard:** When the monorepo already had packages before this import run (i.e., `pnpm-lock.yaml` exists at bootstrap time), a baseline of the pre-existing workspace vulnerabilities is captured **before** the import loop and stored as `_MONOREPO_PRE_BASELINE`. This prevents findings from pre-existing packages from being mistaken for migration-induced regressions.

**Delta comparison:** The post-import audit output is compared against the union of all pre-import baselines plus `_MONOREPO_PRE_BASELINE`. If Phase A found zero HIGH/CRITICAL vulnerabilities across all imported repos and Phase B finds any, those are migration-induced regressions — the script **fails fatally**. If Phase A already recorded pre-existing findings, the script emits a warning distinguishing pre-existing from new findings but only fails on the new ones.

If `--no-pre-audit` was passed (no baseline), Phase B compares against `_MONOREPO_PRE_BASELINE` only and fails on any finding not already present in the monorepo.

**Behavior on failure:** Blocking by default. Unlike pre-existing vulnerabilities, migration-induced regressions are directly attributable to the import process and must be resolved before the result is committed.

### Capability C — CI/pipeline detection

Runs as part of the pre-import audit phase (`run_pre_import_audit`), using the same temporary clone as Phase A. Consequently it is gated by `--no-pre-audit`: passing `--no-audit` or `--no-pre-audit` suppresses both Phase A and CI detection. This is a pragmatic trade-off — CI detection reuses the already-installed clone for free rather than requiring a separate clone.

Scans the following paths inside the source repo for audit command patterns:

```
.github/workflows/*.yml
.github/workflows/*.yaml
Dockerfile
Dockerfile.*
docker-compose*.yml
Makefile
GNUmakefile
scripts/**/*.sh
package.json  (scripts section)
```

Search pattern: `(yarn|pnpm|npm)\s+audit`

For each match, emits a migration note at the end of the import run (not inline, to avoid noise during execution):

```
[audit-migrate] tos-connector-notification-service:
  .github/workflows/ci.yml:34  yarn audit --level high --groups dependencies
  → Equivalent pnpm command (monorepo root): pnpm audit --audit-level high --prod
  These files were NOT modified. Update them manually before merging the monorepo.
```

This detection runs regardless of `--no-audit` because it is informational and has no side effects.

#### Accepted vulnerability files

While scanning each source repo, the script also checks for common files that record accepted/suppressed CVEs:

```
.auditignore
audit-exceptions.json
.nsprc
.auditrc
audit.json
.audit-ci.json
audit-ci.json
```

If any such file is found, the script emits a migration note alongside the audit command findings:

```
[audit-warn] tos-connector-notification-service:
  Found accepted vulnerability file: .auditignore
  This file was NOT applied during pre-import or post-import audit phases.
  Review it manually and decide whether to carry the exceptions into the monorepo.
```

The script does not parse or honour these files. Applying them correctly requires understanding the target package manager's suppression format, the monorepo workspace context, and policy decisions that are outside the script's scope.

---

## Proposed CLI changes

### New flags

| Flag | Default | Description |
|---|---|---|
| `--no-audit` | — | Suppress both pre-import and post-import audit phases. CI detection still runs. |
| `--no-pre-audit` | — | Suppress the pre-import phase only. Post-import phase runs with no baseline (treats zero findings as baseline). |
| `--no-post-audit` | — | Suppress the post-import phase only. Pre-import phase still runs (baseline recorded but not used for blocking). |
| `--audit-strict` | — | Promote pre-import audit failures to fatal. Requires a clean baseline in every source repo before import proceeds. |

### Flag interaction matrix

| Flags passed | Pre-import audit | Post-import audit | CI detection |
|---|---|---|---|
| *(none — default)* | warn on failure | fail on regression | always |
| `--no-audit` | skip | skip | always |
| `--no-pre-audit` | skip | fail (no baseline) | always |
| `--no-post-audit` | warn on failure | skip | always |
| `--audit-strict` | fail on any finding | fail on regression | always |

### Updated usage block

```
Options:
  ...
  --no-audit           Skip both pre-import and post-import audit phases.
  --no-pre-audit       Skip running the source repo's vulnerability audit before importing.
  --no-post-audit      Skip running pnpm audit in the monorepo after importing.
  --audit-strict       Treat pre-import audit failures as fatal (default: warn only).
  ...
```

---

## Algorithm changes

### Pre-import phase: shared clone

Both `run_pre_import_tests` and `run_pre_import_audit` share a single temporary clone per source repo, managed by the idempotent helper `make_pre_import_clone(repo)`. The helper clones the repo, runs `asdf install` if `.tool-versions` is present, detects the package manager, and runs `$pm install --frozen-lockfile`. Subsequent calls for the same repo are no-ops. The clone is deleted by `cleanup_pre_import_clone(repo)` in the main loop after both phases complete.

This eliminates the double-clone that would otherwise occur when both tests and audit are active.

### Phase A — pre-import audit (per repo, alongside pre-import tests)

```bash
# Before import loop: capture pre-existing monorepo audit state (incremental-import guard)
if not --no-post-audit and not --no-install and pnpm-lock.yaml exists in monorepo:
  write pnpm audit --audit-level high --prod --json output to tmp file
  _MONOREPO_PRE_BASELINE = parse_findings(tmp_file, "pnpm")

for each <repo>:
  # [existing] Phase 1 — pre-import tests
  if not --no-pre-test:
    run_pre_import_tests(repo)   # calls make_pre_import_clone internally

  # Phase A — pre-import audit (CI detection runs inside this)
  if not --no-pre-audit:
    run_pre_import_audit(repo)   # calls make_pre_import_clone internally (no-op if tests already ran)

  cleanup_pre_import_clone(repo)   # delete the shared clone
  import_repo(repo, ...)           # [existing]
```

### Phase B — post-import audit (after all repos, after resolve and install)

```bash
# [existing] resolve_workspace_refs()
# [existing] pnpm install + commit
# [existing] Phase 2 — post-import tests

# Phase B — single workspace-wide audit
if not --no-post-audit:
  if --no-install:
    warn "post-import audit skipped — no node_modules installed"
  else:
    write (pnpm audit --audit-level high --prod --json) to tmp file
    findings = parse_findings(tmp_file, "pnpm")
    combined_baseline = union(_AUDIT_BASELINES[*]) ∪ _MONOREPO_PRE_BASELINE
    new_findings = findings - combined_baseline
    if new_findings is not empty:
      die "post-import audit FAILED — migration introduced new HIGH/CRITICAL vulnerabilities"
    elif findings is not empty:
      warn "pre-existing HIGH/CRITICAL vulnerabilities carried over from source repo(s)"

# [existing] --extract-configs
```

### `run_pre_import_audit(repo)` detail

```bash
run_pre_import_audit(repo):
  make_pre_import_clone(repo)   # idempotent; installs deps if not already done
  src = _PRE_CLONE_DIR[repo]/src
  pm  = _PRE_CLONE_PM[repo]

  detect_audit_ci_commands(repo, src)   # Capability C

  if pm == "yarn":
    yarn_ver = $(cd $src && yarn --version)
    if yarn_ver < 1.22:
      warn "yarn $yarn_ver < 1.22 — --json output unreliable; skipping"
      _AUDIT_BASELINES[repo] = "[]"
      return

  case $pm in
    yarn) cmd = "yarn audit --level high --groups dependencies --json" ;;
    pnpm) cmd = "pnpm audit --audit-level high --prod --json" ;;
    npm)  cmd = "npm audit --audit-level high --omit=dev --json" ;;
  esac

  # Write to file — avoids SIGPIPE when audit output is large (see Implementation notes)
  write ($cmd output in $src, exit code ignored) to audit_file
  findings = parse_findings(audit_file, pm)
  rm audit_file

  _AUDIT_BASELINES[repo] = findings

  if findings is not empty:
    if --audit-strict: die
    else: warn with finding list
  else:
    log_ok "no HIGH/CRITICAL vulnerabilities"
```

### `detect_audit_ci_commands(repo, src)` detail

```bash
detect_audit_ci_commands(repo, src):
  # grep for "(yarn|pnpm|npm) audit" in:
  #   *.yml, *.yaml, Dockerfile, Dockerfile.*, Makefile, GNUmakefile, *.sh
  #   (node_modules excluded)
  # separately scan package.json scripts section via python
  # check for known exception files: .auditignore, audit-ci.json, etc.
  # append all findings to _AUDIT_CI_NOTES[] for summary at end of run
```

Migration notes from `_AUDIT_CI_NOTES` are printed in a summary block at the very end of the import run, after all other output.

---

## Delta comparison strategy

Finding identity is based on **package name + affected version range**, normalised as the string `"pkg:range:severity"`. Advisory IDs are not used because they differ between npm, yarn, and pnpm registries.

The combined baseline fed into the post-import delta is:

```
combined = union(_AUDIT_BASELINES[repo] for each imported repo)
         ∪ _MONOREPO_PRE_BASELINE
```

A finding is "new" — and therefore fatal — if it is present in the post-import workspace audit but absent from `combined`. Pre-existing findings (present in `combined`) produce a warning but do not block the import.

A known limitation: the baseline only retains HIGH/CRITICAL entries. Low/moderate findings that were present pre-migration are not tracked, so they cannot regress into HIGH/CRITICAL without being detected as "new".

---

## Error handling and exit codes

| Situation | Behaviour | Exit code |
|---|---|---|
| Pre-import audit finds HIGH/CRITICAL (default) | `[audit-warn]` log; baseline recorded; import continues | — |
| Pre-import audit finds HIGH/CRITICAL (`--audit-strict`) | `die` immediately; no import performed | 1 |
| Post-import audit finds new findings (not in any baseline) | `die`; monorepo git state intact | 1 |
| Post-import audit finds only pre-existing findings | `[audit-warn]` log; import succeeds | — |
| Audit command itself fails to run (tool missing, network error) | `[audit-warn]` log; phase skipped | — |
| CI detection finds audit commands | Informational note at end of run | — |

The monorepo git state is never rolled back on an audit failure. All import commits are already in history when Phase B runs, consistent with the test phase behaviour.

---

## Interaction with existing flags

| Flag combination | Notes |
|---|---|
| `--dry-run` | Both audit phases are printed as log lines but not executed. CI detection still scans and reports. |
| `--no-install` | Post-import audit requires an installed `node_modules` tree. When `--no-install` is passed, Phase B is automatically skipped with a warning (`[audit-skip] --no-install: skipping post-import audit — no lockfile installed`). |
| `--no-test` | Audit phases are independent; `--no-test` does not affect them. |
| `--extract-configs` | Audit runs before config extraction. A fatal Phase B failure prevents config extraction from running. |

---

## Resolved design decisions

| # | Question | Decision |
|---|---|---|
| 1 | **JSON output / yarn version** | Enforce yarn >= 1.22. No regex fallback. If the source repo pins an older version, the pre-import audit phase is skipped with `[audit-skip]`. |
| 2 | **Pre-import audit timing** | Run after `$pm install --frozen-lockfile`, reusing the pre-import test clone via the shared `make_pre_import_clone` helper. More accurate than pre-install, no extra clone cost. |
| 3 | **Post-import audit scope** | Run a single `pnpm audit` at the workspace root (pnpm does not support `--filter` for audit). Scoping to imported packages is achieved via delta comparison against the union of pre-import baselines plus `_MONOREPO_PRE_BASELINE` (captured before the import loop when a lockfile already exists), which excludes findings from packages that were in the monorepo before this run. |
| 4 | **Accepted vulnerability files** | Detect common suppression files (`.auditignore`, `audit-ci.json`, etc.) and emit a migration warning. Do not parse or apply them — honouring exceptions correctly is out of scope. |
| 5 | **Post-import blocking default** | Strict by default: post-import audit failures are fatal. Pre-import failures remain non-blocking by default (use `--audit-strict` to promote). |

---

## Implementation notes

### Shared pre-import clone

Both `run_pre_import_tests` and `run_pre_import_audit` call `make_pre_import_clone(repo)`, which is idempotent: the first call clones and installs, subsequent calls return immediately. `cleanup_pre_import_clone(repo)` is called once in the main loop after both phases finish. This means a repo with both tests and audit enabled incurs only one clone and one `$pm install`, not two.

### SIGPIPE hazard with `python3 - <<'PY'` in a pipeline

An early version of `_parse_audit_findings` used `python3 - "$pm" <<'PY' ... PY` and was called as:

```bash
findings=$(printf '%s' "$audit_output" | _parse_audit_findings "$pm")
```

This produces a silent SIGPIPE (exit 141) when the audit output is large. The cause: bash's `<<'PY'` heredoc replaces python3's stdin with the script source, so the piped audit data is never consumed. Once python3 exits, the pipe's read-end closes while `printf` is still blocked on a full pipe buffer — SIGPIPE.

The fix: `_parse_audit_findings` now takes a file path as its second argument. Callers write the audit output to a temp file and pass the path:

```bash
audit_file=$(mktemp)
(cd "$src" && eval "$audit_cmd" 2>/dev/null || true) > "$audit_file"
findings=$(_parse_audit_findings "$pm" "$audit_file")
rm -f "$audit_file"
```

Python reads the audit data from the file (`open(sys.argv[2]).read()`), and python3's stdin is used exclusively for the heredoc script source. No pipe is involved. The same pattern is used in all three call sites: `run_pre_import_audit`, `run_post_import_audit`, and the `_MONOREPO_PRE_BASELINE` capture in `main`.

### pnpm audit JSON format

pnpm 9+ emits the npm audit v2 format (`{"vulnerabilities": {...}}`). Older pnpm versions used an `{"advisories": {...}}` format. The parser handles both. npm uses the v2 format natively. yarn classic outputs one JSON object per line (NDJSON), not a single JSON document.
