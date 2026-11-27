# rig-ci

Central reusable GitHub Actions workflows for the CI of ditshej's projects.
A project's `.github/workflows/ci.yml` only calls them and passes its inputs.
A change here reaches every project that calls `@v1`, without a commit in the
projects.

## Workflows

| Workflow | Job | For |
| --- | --- | --- |
| `laravel-quality.yml` | `quality` | Laravel: Pint, Larastan, Prettier (when installed), Pest with 70 % coverage, frontend build |
| `php-quality.yml` | `quality` | PHP without Laravel: Pint, PHPStan, Prettier (when installed), Pest with 70 % coverage |
| `node-quality.yml` | `quality` | Node: Prettier, TypeScript (with a `tsconfig.json`), tests with 70 % coverage |
| `security.yml` | `security` | every project: CLAUDE.md length (warning), lockfiles, `composer audit`, `npm audit` |
| `gitleaks.yml` | `gitleaks` | every project: secrets in the commits |

Projects on another stack keep their own `quality` job (and their own
dependency audit) in their `ci.yml` and call only `security.yml` and
`gitleaks.yml`.

## Inputs

| Workflow | Input | Default |
| --- | --- | --- |
| `laravel-quality.yml`, `php-quality.yml` | `php-version` (version of production) | required |
| | `node-version` (used only with a `package-lock.json`) | `'22'` |
| | `php-extensions` (for `setup-php`, e.g. `'mbstring, sqlite3'`) | `''` |
| | `timeout-minutes` | `20` |
| `laravel-quality.yml` | `migrate` (create `database/database.sqlite`, run `php artisan migrate --force`) | `false` |
| `node-quality.yml` | `node-version` (version of production) | `''` |
| | `node-version-file` (relative to the repository root, e.g. `'app/.nvmrc'`) | `''` |
| | `working-directory` (folder with the `package.json`; all steps run there) | `'.'` |
| | `timeout-minutes` | `20` |
| `security.yml` | `php-version` (for `composer audit`) | `'8.3'` |
| | `php-extensions` (for `setup-php`) | `''` |
| | `working-directory` (folder with the `package.json` for the npm checks; Composer is checked in the root) | `'.'` |
| `gitleaks.yml` | none | |

A project that raises `timeout-minutes` gives the reason in its `CLAUDE.md`.
`node-quality.yml` needs exactly one of `node-version` and `node-version-file`;
with both or neither the job fails.
`security.yml` skips a `composer audit` step with a notice when its package
list in `composer.lock` is empty (e.g. only `php` in `require`).

## Calling the workflows

Laravel example. For PHP without Laravel use `php-quality.yml`; for Node use
`node-quality.yml` with `node-version` only.

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

# A new push to a pull request cancels the older run; runs on main finish.
concurrency:
  group: ci-${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  quality:
    uses: ditshej/rig-ci/.github/workflows/laravel-quality.yml@v1
    with:
      php-version: '8.3'
      # Optional:
      # php-extensions: 'mbstring, sqlite3'
      # migrate: true
  security:
    uses: ditshej/rig-ci/.github/workflows/security.yml@v1
    with:
      php-version: '8.3'
  gitleaks:
    uses: ditshej/rig-ci/.github/workflows/gitleaks.yml@v1
    permissions:
      contents: read
      pull-requests: read
```

Node project with its `package.json` in a subfolder:

```yaml
jobs:
  quality:
    uses: ditshej/rig-ci/.github/workflows/node-quality.yml@v1
    with:
      node-version-file: app/.nvmrc
      working-directory: app
  security:
    uses: ditshej/rig-ci/.github/workflows/security.yml@v1
    with:
      working-directory: app
```

Notes:

- Keep `concurrency` in the calling `ci.yml`, not in the workflows here.
- The `gitleaks` job needs `pull-requests: read`. A called workflow cannot get
  more rights than the calling job grants.
- The checks are named `<calling job> / <job>`: `quality / quality`,
  `security / security`, `gitleaks / gitleaks`. Required status checks use
  these names.
- A project adds its own jobs next to these calls. Do not add `paths` or
  `paths-ignore`: required checks would never report on a pull request that
  only changes ignored files.
- Runs, logs and Actions minutes belong to the calling project. Secrets stay in
  the project; none of these workflows needs one besides the automatic
  `GITHUB_TOKEN`.

## Versions

- Releases are tags `v1.x.y`. The major tag `v1` moves to the latest `v1`
  release. Projects call `@v1`.
- A change that needs a change in the projects (a new required input, a removed
  input, a renamed job) is a new major version (`v2`). Every project switches
  on purpose.
- Test a change before moving `v1`: call the workflow from one project with
  `@<branch>` on a test branch, check the run, then release.

## Rules

1. **Nothing internal here.** No server names, deploy targets, project names
   or secrets. This repository is public. Deploy workflows stay in the
   projects.
2. **Only the owner moves the tags.** Whoever moves `v1` changes the CI of
   every project. A ruleset protects the tags `v*`.

Changes come as pull requests. Agents do not commit files under
`.github/workflows/`; the owner commits them.
