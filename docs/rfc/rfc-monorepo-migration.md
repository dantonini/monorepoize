# RFC: Mono-repository Migration Script

**Status:** Draft  
**Date:** 2026-04-16  
**Author:** Daniele Antonini

---

## Summary

A CLI script (`monorepo-import`) that absorbs one or more existing GitHub repositories into a pnpm workspace monorepo, **preserving the full git history** of every imported project. The script can bootstrap a new monorepo from scratch or add packages to an already-existing one.

---

## Motivation

`tos-connector-notification-service` (Koa/Node.js backend) and `tos-connector-notification-service-frontend` (React/Vite) are two tightly coupled services that share the same Node.js runtime version (22.21.1), the same `@translated` private registry token, nearly identical ESLint/Prettier configurations, and a common release cadence. Keeping them in separate repositories creates:

- Duplicated toolchain boilerplate (`.nvmrc`, `.eslintrc`, `tsconfig.json` base, Prettier config)
- No cross-package type sharing without publishing
- Split CI pipelines that cannot test the pair atomically
- Dependency drift (both pin `axios ^1.15.0` and `zod ^3.22.4` independently)

A pnpm workspace monorepo solves all of these. The same migration pattern applies to any future pair of related `tos-connector-*` repos.

---

## Goals

1. Merge N repositories into a single monorepo while preserving 100 % of each repo's commit history, authorship, and tags.
2. Re-root each repo's history under its workspace directory (`apps/<name>/` or `packages/<name>/`) so `git log apps/<name>/` shows only that package's history.
3. Support both **bootstrap** (creates the monorepo from scratch) and **import** (adds repos to an existing pnpm monorepo).
4. Optionally set up a reproducible Node/pnpm environment via `asdf` when `--asdf` is passed; without the flag the script relies on the system-wide `node` and `pnpm` binaries already in `PATH`.
5. Migrate `yarn.lock` files to `pnpm-lock.yaml` during import.
6. Keep the script idempotent: re-running it on an already-imported repo is a no-op with a clear warning.

## Non-goals

- Merging or deduplicating application code (ESLint rules, `tsconfig` bases, shared utilities). That is a follow-up refactoring task for each team.
- Migrating CI/CD pipelines automatically. Guidance is provided in the output, but existing per-repo workflows are left in place initially.
- Handling monorepos that use npm or Yarn workspaces. Only pnpm workspaces are targeted.
- Automatic conflict resolution when two repos contain a file at the same root path (e.g. both have `.eslintrc`). The script surfaces conflicts and stops.

---

## Background: git history preservation strategy

### Option A — `git subtree add` (rejected)

`git subtree` merges the foreign tree as a single squash or as a merge commit. It preserves the files but **does not** replay individual commits under the subdirectory path; `git log --follow` stops at the merge boundary.

### Option B — `git filter-branch` (rejected)

Rewrites history in-place but is slow, has known edge cases with merge commits, and is officially deprecated.

### Option C — `git filter-repo --to-subdirectory-filter` + remote merge (selected)

`git filter-repo` (the recommended replacement for `filter-branch`) rewrites the imported repo's history so every commit's tree is moved under `<target-dir>/<name>/` (e.g. `apps/<name>/`). The rewritten repo is then added as a remote and merged with `--allow-unrelated-histories`. The result is a linear or merge-commit-connected DAG where `git log apps/<name>/` faithfully returns the full per-package history.

Steps per imported repo:

```
1. gh repo clone <org>/<repo> /tmp/import/<repo>
2. cd /tmp/import/<repo>
3. git filter-repo --to-subdirectory-filter <target-dir>/<name>/
4. cd <monorepo>
5. git remote add import-<name> /tmp/import/<repo>
6. git fetch import-<name>
7. git merge --allow-unrelated-histories --no-edit import-<name>/<default-branch>
8. git remote remove import-<name>
9. rm -rf /tmp/import/<repo>
```

Tags from the source repo are fetched and renamed to `<name>/<tag>` to avoid collisions with existing monorepo tags.

---

## Target monorepo layout

```
<monorepo-root>/
├── .tool-versions                  # asdf: nodejs + pnpm versions (only written with --asdf)
├── package.json                    # workspace root (private: true)
├── pnpm-workspace.yaml             # workspace globs
├── pnpm-lock.yaml                  # generated on first `pnpm install`
├── .npmrc                          # shared registry config (NPM_TOKEN)
├── .gitignore                      # merged/union of imported .gitignores
├── apps/                           # deployable applications (Turborepo convention)
│   ├── tos-connector-notification-service/
│   │   ├── package.json            # name unchanged, packageManager updated
│   │   ├── tsconfig.json
│   │   ├── Dockerfile
│   │   ├── .env.*                  # preserved as-is
│   │   └── src/
│   └── tos-connector-notification-service-frontend/
│       ├── package.json
│       ├── vite.config.ts
│       ├── Dockerfile
│       ├── .env.*
│       └── src/
└── packages/                       # shared internal libraries and configs (initially empty)
```

### `pnpm-workspace.yaml`

```yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

### Root `package.json`

```json
{
  "name": "@translated/monorepo",
  "private": true,
  "scripts": {
    "build": "pnpm -r build",
    "lint":  "pnpm -r lint:check",
    "test":  "pnpm -r test"
  },
  "engines": {
    "node": ">=22.21.1",
    "pnpm": ">=9"
  }
}
```

### `.tool-versions` (asdf, only with `--asdf`)

```
nodejs 22.21.1
pnpm   9.15.4
```

Written only when `--asdf` is passed. The script reads `.nvmrc` from each imported repo to determine the Node version and picks the highest one. The pnpm version is resolved to the latest stable `9.x` at script runtime via `gh api` or `npm view pnpm dist-tags`.

---

## Script interface

```
Usage:
  monorepo-import [options] <repo> [<repo>...]

Arguments:
  <repo>       GitHub repo in <owner>/<name> or full URL form.
               Accepts multiple space-separated values.

Options:
  --monorepo <path>    Target directory. Created if absent. Default: ./monorepo
  --app-dir <dir>      Subdirectory for deployable applications. Default: apps
  --pkg-dir <dir>      Subdirectory for shared library packages. Default: packages
  --pkg                Place the imported repo under --pkg-dir instead of --app-dir.
  --name <name>        Override the target directory name for a single import.
  --branch <branch>    Source branch to import. Default: the repo's default branch.
  --asdf               Write .tool-versions and manage Node/pnpm via asdf.
                       Without this flag the script uses the system-wide node and pnpm.
  --extract-configs    After all imports, extract identical config files (tsconfig,
                       ESLint, Prettier) into shared packages under packages/.
                       Without this flag the script only reports duplicates.
  --dry-run            Print what would happen without modifying anything.
  --no-install         Skip `pnpm install` after import.
  --help

Examples:
  # Bootstrap a new monorepo from two repos (system node/pnpm)
  monorepo-import \
    translated/tos-connector-notification-service \
    translated/tos-connector-notification-service-frontend \
    --monorepo ./notification-monorepo

  # Same, but manage Node/pnpm versions via asdf
  monorepo-import \
    translated/tos-connector-notification-service \
    translated/tos-connector-notification-service-frontend \
    --monorepo ./notification-monorepo \
    --asdf

  # Add a shared library repo (goes under packages/ instead of apps/)
  monorepo-import translated/tos-connector-salesforce \
    --monorepo ./notification-monorepo \
    --pkg
```

---

## Script algorithm (pseudocode)

```bash
monorepo-import():
  # 0. Preflight
  require_commands git gh pnpm git-filter-repo
  if --asdf: require_commands asdf
  validate each <repo> is accessible via gh

  target_dir = --pkg ? <pkg-dir> : <app-dir>   # apps/ by default

  # 1. Resolve or create monorepo
  if monorepo does not exist:
    git init <monorepo>
    write pnpm-workspace.yaml, root package.json, .npmrc, .gitignore
    if --asdf:
      write .tool-versions
      asdf install  # installs node + pnpm declared in .tool-versions
    git add -A && git commit -m "chore: initialize pnpm monorepo"
  else:
    assert it is a git repo with pnpm-workspace.yaml

  for each <repo>:
    # 2. Determine target name and destination
    name = derive_package_name(repo)   # strips org prefix, uses repo name
    dest = <monorepo>/<target_dir>/<name>

    # 3. Idempotency check
    if dest exists in git tree:
      warn "<target_dir>/<name> already present, skipping"
      continue

    # 4. Clone into temp dir
    tmp = mktemp -d
    gh repo clone <repo> $tmp/src -- --no-local

    # 5. Rewrite history under subdirectory
    cd $tmp/src
    git filter-repo --to-subdirectory-filter <target_dir>/<name>/

    # 6. Fetch tags, rename to avoid collisions
    git tag -l | while read tag; do
      git tag <name>/$tag $tag
      git tag -d $tag
    done

    # 7. Merge into monorepo
    cd <monorepo>
    git remote add import-<name> $tmp/src
    git fetch import-<name> --tags
    git merge --allow-unrelated-histories --no-edit \
      -m "chore: import <repo> history into <target_dir>/<name>" \
      import-<name>/<branch>
    git remote remove import-<name>

    # 8. Post-import fixups inside the package dir
    cd <monorepo>/<target_dir>/<name>
    migrate_packagejson()       # remove packageManager yarn field, set pnpm
    remove yarn.lock            # pnpm-lock.yaml generated on install
    if --asdf:
      migrate_nvmrc_to_tool_versions()   # update root .tool-versions if needed
    stage_and_commit("chore(<name>): post-import cleanup")

    # 9. Cleanup
    rm -rf $tmp

  # 10. Install
  if not --no-install:
    cd <monorepo>
    if --asdf: asdf install
    pnpm install
    git add pnpm-lock.yaml && git commit -m "chore: add pnpm-lock.yaml"

  # 11. Shared config analysis (always runs)
  report = detect_shared_configs(<monorepo>, all imported <target_dir>/<name> dirs)
  print summary of identical / diverged config files across packages
  if --extract-configs:
    extract_shared_configs(report)
    git add -A && git commit -m "chore: extract shared configs into packages/"
```

### `migrate_packagejson()` detail

- Rewrite `"name"` to `@translated/<repo-name>` (e.g. `@translated/tos-connector-notification-service`)
- Set `"packageManager": "pnpm@<version>"` (matching `.tool-versions` when `--asdf` is used, otherwise the version resolved via `npm view pnpm dist-tags`)
- Remove the `"packageManager": "yarn@..."` entry
- If the package has internal `@translated/*` dependencies that exist in this monorepo, replace their version specifiers with `"workspace:*"`
- Leave all other dependency versions unchanged

### `.npmrc` handling

The root `.npmrc` is initialized with:
```
engine-strict=true
//registry.npmjs.org/:_authToken=${NPM_TOKEN}
```
Per-package `.npmrc` files are removed after merge if their content is a subset of the root's.

---

## Shared config extraction

### Detection (always runs)

After all imports are complete the script scans each imported package for the following files and groups them by SHA-256 content hash:

| File glob | Shared package target |
|---|---|
| `tsconfig.json` | `packages/tsconfig/` |
| `.prettierrc`, `.prettierrc.json`, `prettier.config.*` | `packages/prettier-config/` |
| `eslint.config.*` (flat-config) | `packages/eslint-config/` |
| `.eslintrc`, `.eslintrc.{json,js,yaml,yml}` (legacy) | `packages/eslint-config/` |

The script always prints a report:
```
[shared-configs] tsconfig.json       — identical across 2/2 packages  ✓ extractable
[shared-configs] .prettierrc         — identical across 2/2 packages  ✓ extractable
[shared-configs] eslint.config.js    — differs between packages       ⚠ manual review needed
```

### Extraction (`--extract-configs`)

For each config group where all instances are **identical**, the script:

1. Creates a minimal shared package under `packages/<config-pkg>/`:
   ```
   packages/tsconfig/
   ├── package.json    # { "name": "@scope/tsconfig", "private": true }
   └── base.json       # the extracted config
   ```

2. Rewrites each consumer package to reference the shared one:

   | Config type | Before | After |
   |---|---|---|
   | `tsconfig.json` | standalone `compilerOptions` | `{ "extends": "@scope/tsconfig/base" }` with only per-package overrides retained |
   | Prettier | `.prettierrc` with content | `"prettier": "@scope/prettier-config"` in `package.json`; `.prettierrc` removed |
   | ESLint flat-config | `eslint.config.js` with rules | `import config from "@scope/eslint-config"; export default config;` with per-package overrides spread-merged |
   | ESLint legacy | `.eslintrc` with rules | `{ "extends": ["@scope/eslint-config"] }` with per-package overrides retained |

3. Adds `"@scope/<config-pkg>": "workspace:*"` to each consumer's `devDependencies`.

4. Commits the result as `chore: extract shared configs into packages/`.

**Diverged configs** (where instances differ) are reported but never touched. The developer resolves them manually as a follow-up.

### Scope inference

The `@scope` prefix used for generated package names is inferred from the existing workspace packages' `name` fields (e.g. if all packages are `@translated/*`, the shared packages become `@translated/tsconfig`, `@translated/eslint-config`, `@translated/prettier-config`). If no consistent scope is found, the script prompts the user.

---

## Conflict handling

| Situation | Behaviour |
|---|---|
| Same tag exists in two repos | Rename to `<name>/<tag>` on import; abort if collision persists |
| File at root level already exists (e.g., `.eslintrc`) | Warn, leave both under their package paths; root-level file is not touched |
| `<target_dir>/<name>/` already tracked in git | Skip with warning; never overwrite |
| `git merge` produces conflicts | Abort merge, print conflicting paths, instruct user to resolve manually |
| `pnpm install` fails | Print error; monorepo git state is already clean — user can fix and re-run `pnpm install` |

---

## Tool requirements

| Tool | Version | Required | Purpose |
|---|---|---|---|
| `git` | ≥ 2.38 | Always | VCS operations, `--allow-unrelated-histories` |
| `git filter-repo` | ≥ 2.38 | Always | History rewriting — vendored at `scripts/vendor/git-filter-repo`; requires Python 3 |
| `python3` | ≥ 3.6 | Always | Runtime for the vendored `git-filter-repo` script |
| `gh` | ≥ 2.40 | Always | Authenticated repo cloning, API calls |
| `pnpm` | ≥ 9 | Always | Workspace management, lockfile generation |
| `asdf` | ≥ 0.14 | Only with `--asdf` | Reproducible Node/pnpm environment via `.tool-versions` |

The script will check for each required tool at startup and print a clear installation message for any that are missing. When `--asdf` is not passed, the script verifies that `node` and `pnpm` are reachable in `PATH` and meets the minimum version requirements.

---

## asdf integration (`--asdf` only)

When `--asdf` is passed, the script manages the Node/pnpm toolchain via asdf. On first run (bootstrap) it writes `.tool-versions`:

```
nodejs <max-nvmrc-version-across-imported-repos>
pnpm   <latest-9.x-stable>
```

It then calls `asdf install` inside the monorepo directory. If `asdf` is not configured for auto-reshim, the script calls `asdf reshim nodejs` and `asdf reshim pnpm` explicitly.

On subsequent import runs on an existing monorepo, if the newly imported repo's `.nvmrc` declares a *higher* Node version than what is in `.tool-versions`, the script:
1. Updates `.tool-versions`
2. Calls `asdf install`
3. Commits the change

If the version is *lower*, it is silently ignored (the higher version is backwards compatible within the same major).

**Without `--asdf`** the script skips all of the above. It verifies that the system `node` and `pnpm` binaries satisfy the minimum version constraints declared in the workspace root `package.json` `engines` field and aborts with a clear message if they do not. No `.tool-versions` file is written.

---

## CI/CD migration notes (out of scope for script, guidance only)

Each imported package keeps its own `Dockerfile` and `deploy.sh` unchanged. Teams should:

1. Replace per-repo GitHub Actions workflows with monorepo-aware equivalents using path filters (`on.push.paths: apps/<name>/**` or `packages/<name>/**` depending on where the repo was imported).
2. The root `.npmrc` supersedes per-package `.npmrc` for registry authentication — ensure `NPM_TOKEN` is available as a repository secret in the new monorepo.
3. Dependabot: consolidate per-package `dependabot.yml` files into a single root-level file with `directory` pointing to each package.

---

## Open questions

1. ~~**Package naming convention**~~ — **Resolved: scoped.** The `name` field in each package's `package.json` is rewritten to `@translated/<repo-name>` (e.g. `@translated/tos-connector-notification-service`). Scoped names make `workspace:*` references unambiguous, prevent accidental resolution against the public npm registry, and are consistent with the shared config packages created by `--extract-configs`.

2. ~~**`apps/` vs `packages/` split`**~~ — **Resolved: Turborepo-style.** Deployable applications (both frontends and backend services) live under `apps/`; shared internal libraries and configs live under `packages/`. The script defaults to `apps/`; pass `--pkg` to target `packages/` instead.

3. ~~**Shared configs**~~ — **Partially resolved: detect always, extract with `--extract-configs`.** See the *Shared config extraction* section below.

4. **Tag strategy** — `<name>/<tag>` is the proposed collision-avoidance scheme. An alternative is `<name>-<tag>` (no slash). GitHub's UI handles both, but slash-namespaced tags display as a hierarchy.

5. ~~**`git-filter-repo` availability`**~~ — **Resolved: vendor the script.** `git-filter-repo` is a single self-contained Python file with no external dependencies — no pip or package manager required, only a system Python 3 interpreter (present on macOS and all common Linux distros). The script will be vendored at `scripts/vendor/git-filter-repo` inside the monorepoize repository and invoked directly (`python3 scripts/vendor/git-filter-repo …`), pinned to a specific upstream release commit. This eliminates the runtime pre-requisite entirely.
