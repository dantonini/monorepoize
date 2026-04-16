# monorepo-import

A CLI script that absorbs one or more existing GitHub repositories into a
**pnpm workspace monorepo**, preserving the full git history of every imported
project. It can bootstrap a new monorepo from scratch or add packages to an
already-existing one.

## Requirements

| Tool | Version | Notes |
|---|---|---|
| `git` | ≥ 2.38 | |
| `gh` | ≥ 2.40 | Must be authenticated (`gh auth login`) |
| `python3` | ≥ 3.6 | Bundled — ships on macOS and all common Linux distros |
| `pnpm` | ≥ 9 | |
| `bash` | ≥ 4 | macOS default is 3.x — `brew install bash` if needed |
| `asdf` | ≥ 0.14 | Only required with `--asdf` |

`git-filter-repo` is vendored at `scripts/vendor/git-filter-repo` — no
separate installation needed.

## Usage

```
./monorepo-import [options] <repo> [<repo>...]
```

### Arguments

| Argument | Description |
|---|---|
| `<repo>` | GitHub repo as `<owner>/<name>` or full URL. Accepts multiple values. |

### Options

| Option | Default | Description |
|---|---|---|
| `--monorepo <path>` | `./monorepo` | Target directory. Created if absent. |
| `--app-dir <dir>` | `apps` | Subdirectory for deployable applications. |
| `--pkg-dir <dir>` | `packages` | Subdirectory for shared library packages. |
| `--pkg` | — | Place the imported repo under `--pkg-dir` instead of `--app-dir`. |
| `--name <name>` | — | Override the target directory name (single import only). |
| `--branch <branch>` | repo default | Source branch to import. |
| `--asdf` | — | Write `.tool-versions` and manage Node/pnpm via asdf. Without this flag the script uses the system-wide `node` and `pnpm`. |
| `--extract-configs` | — | Extract identical config files (tsconfig, ESLint, Prettier) into shared `packages/` after import. Without this flag the script only reports duplicates. |
| `--dry-run` | — | Print all actions without executing any of them. |
| `--no-install` | — | Skip `pnpm install` after import. |
| `--help` | — | Print usage. |

## Examples

```bash
# Bootstrap a new monorepo from two repos (system node/pnpm)
./monorepo-import \
  translated/tos-connector-notification-service \
  translated/tos-connector-notification-service-frontend \
  --monorepo ./notification-monorepo

# Same, but pin Node/pnpm versions via asdf
./monorepo-import \
  translated/tos-connector-notification-service \
  translated/tos-connector-notification-service-frontend \
  --monorepo ./notification-monorepo \
  --asdf

# Add a shared library repo (goes under packages/ instead of apps/)
./monorepo-import translated/tos-connector-shared \
  --monorepo ./notification-monorepo \
  --pkg

# Preview what would happen without touching anything
./monorepo-import translated/tos-connector-notification-service \
  --monorepo ./notification-monorepo \
  --asdf \
  --dry-run
```

## What it does

### Monorepo layout (Turborepo-style)

```
<monorepo>/
├── .tool-versions          # only written with --asdf
├── package.json            # workspace root (private: true, engines pinned)
├── pnpm-workspace.yaml     # globs: apps/* and packages/*
├── .npmrc                  # engine-strict + NPM_TOKEN registry auth
├── .gitignore
├── apps/                   # imported deployable applications land here
└── packages/               # shared libs and extracted configs land here
```

### Per-import steps

1. Clones the source repo into a temp directory.
2. Rewrites the entire git history under `apps/<name>/` (or `packages/<name>/`) using the vendored `git-filter-repo`.
3. Renames all tags to `<name>/<tag>` to avoid collisions.
4. Merges into the monorepo with `--allow-unrelated-histories` so `git log apps/<name>/` returns the full original history.
5. Post-import cleanup: scoped package name (`@<org>/<name>`), `pnpm` packageManager field, `workspace:*` for internal deps, yarn.lock removal, redundant `.npmrc` removal.

### `--asdf` flag

When passed, the script:
- Fetches `.nvmrc` from each source repo via the GitHub API and picks the highest Node version.
- Writes `.tool-versions` with the resolved `nodejs` and `pnpm` versions.
- Calls `asdf install` before and after `pnpm install`.
- Updates `.tool-versions` on subsequent runs if a newly imported repo requires a higher Node version.

Without `--asdf`, the script uses the `node` and `pnpm` binaries already in `PATH` and validates them against the `engines` field in the root `package.json`.

### `--extract-configs` flag

After all imports, the script scans for config files shared across packages and groups them by content hash. It always prints a report:

```
✓ tsconfig.json       identical across 2/2 packages  → extractable
⚠ eslint.config.js   differs between packages        → manual review
```

With `--extract-configs`, identical configs are extracted into minimal shared packages:

| Config | Shared package | Consumer update |
|---|---|---|
| `tsconfig.json` | `packages/tsconfig/base.json` | `"extends": "@scope/tsconfig/base"` |
| Prettier | `packages/prettier-config/index.json` | `"prettier": "@scope/prettier-config"` in `package.json` |
| ESLint flat-config | `packages/eslint-config/<file>` | `import config from '@scope/eslint-config'` |
| ESLint legacy | `packages/eslint-config/<file>` | `"extends": ["@scope/eslint-config"]` |

Diverged configs are reported but never modified.

## Package naming

Imported package names are rewritten to `@<github-org>/<repo-name>` (e.g.
`@translated/tos-connector-notification-service`). Scoped names make
`workspace:*` cross-references unambiguous and prevent accidental resolution
against the public npm registry.

## Idempotency

Re-running the script on an already-imported repository is a no-op — the
script detects the existing `apps/<name>/` directory and skips with a warning.

## Design notes

Full rationale and decision log: [`docs/rfc/rfc-monorepo-migration.md`](docs/rfc/rfc-monorepo-migration.md).
