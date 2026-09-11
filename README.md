# addon-ci

Centralized, reusable GitHub Actions workflows for my WoW addon repos, so CI
tooling lives in one place instead of being copy-pasted (and drifting) across
every repo.

## Workflows

### `lua-test.yml`

Lints with **luacheck** (Lua 5.1, matching WoW's interpreter) and runs
**busted** only when the repo has specs (a `.busted` file or any `spec/` dir).

Add this caller to a repo at `.github/workflows/ci.yml`:

```yaml
name: ci

on:
  push:
    branches: [main]
  pull_request:

jobs:
  ci:
    uses: roshne/addon-ci/.github/workflows/lua-test.yml@v1
```

### `release.yml`

Reusable auto-release for single-addon repos. On every run it checks commits
since the addon's last `<repo>-v*` tag (ignoring doc-only, `spec/`, and
`tools/` changes); if there's nothing new it's a no-op. Otherwise it bumps the
last numeric segment of the `.toc`'s `## Version:` field, builds a changelog
grouped by conventional-commit type, commits the bump, tags `<repo>-v<version>`,
and creates a GitHub release.

Schedule triggers only fire from the repo that owns the workflow file, so each
consumer needs its own thin caller at `.github/workflows/release.yml`:

```yaml
name: release

on:
  schedule:
    - cron: '0 14 * * *'
  workflow_dispatch:

permissions:
  contents: write

concurrency:
  group: release
  cancel-in-progress: false

jobs:
  release:
    uses: roshne/addon-ci/.github/workflows/release.yml@v1
    secrets: inherit
```

Requires a `RELEASE_TOKEN` repo secret — a PAT that can push past branch
protection (the default `GITHUB_TOKEN` is blocked when `enforce_admins` is on).

### `publish.yml`

Reusable CurseForge publisher for single-addon repos. Triggered when a GitHub
release is published, it reads `## X-Curse-Project-ID:` and `## Interface:`
from the repo-root `.toc`, zips the repo (excluding `spec/` and `tools/`), and
uploads to CurseForge via `itsmeow/curseforge-upload`. Safe to add before the
CurseForge project exists — it silently skips if `X-Curse-Project-ID` is
absent from the `.toc`.

```yaml
name: publish

on:
  release:
    types: [published]
  workflow_dispatch:

jobs:
  curseforge:
    uses: roshne/addon-ci/.github/workflows/publish.yml@v1
    secrets: inherit
```

Requires a `CURSEFORGE_API_TOKEN` repo secret (from CurseForge account →
Settings → API Tokens).

### `discord.yml`

Reusable Discord notification for single-addon repos. Posts a one-line summary
(PR number, title, merger) to a Discord webhook. Callers trigger on
`pull_request` closed so only merged PRs post — never per-push spam, and the
daily `chore(release): bump version` bot push stays silent. Safe to add before
the webhook exists — it skips cleanly if the `DISCORD_WEBHOOK` secret is unset.

```yaml
name: discord

on:
  pull_request:
    types: [closed]
    branches: [main]
  workflow_dispatch:

jobs:
  notify:
    if: github.event_name == 'workflow_dispatch' || github.event.pull_request.merged == true
    uses: roshne/addon-ci/.github/workflows/discord.yml@v1
    secrets: inherit
```

Requires a `DISCORD_WEBHOOK` repo secret (Discord channel → Integrations →
Webhooks). `workflow_dispatch` fires a test notification without merging
anything.

### `push-notify.yml`

Reusable push-to-`main` Discord notification, styled as the webhook's own
"Github-Repo-Updates" sender rather than Discord's hard-coded `/github`
integration name. A caller triggers on `push` to its default branch and gets
one embed per push. Safe to add before the webhook exists — it skips cleanly
(green) when the caller has no `DISCORD_PUSH_WEBHOOK` secret.

```yaml
name: push-notify

on:
  push:
    branches: [main]

jobs:
  notify:
    uses: roshne/addon-ci/.github/workflows/push-notify.yml@v1
    secrets:
      DISCORD_PUSH_WEBHOOK: ${{ secrets.DISCORD_PUSH_WEBHOOK }}
    # with: { runner: '["self-hosted","linux"]' }   # optional: run on your own runner
```

Requires a `DISCORD_PUSH_WEBHOOK` repo secret. Pass it **explicitly**, as
above — never `secrets: inherit`. A private repo in a free org can't reach
org-level secrets, so `inherit` still resolves to the inaccessible org secret
as an empty string and this workflow's guard skips green: CI passes and no
notification ever posts, silently.

### `python-app.yml`

Reusable Python CI — the Python analog of `lua-test.yml`: define-once,
call-everywhere, detect-then-run. `pytest` runs whenever `test-paths` exists;
`ruff check` runs once a ruff config exists (`ruff.toml`/`.ruff.toml` or
`[tool.ruff]` in `pyproject.toml`), and adds `ruff format --check` too if
`ruff-format: true`; `mypy` runs once a mypy config exists (`mypy.ini` or
`[tool.mypy]`/`[mypy]`). Each gate turns on the moment its config file shows
up — nothing to edit here or at the caller.

Add this job to an existing caller workflow — it has no trigger of its own:

```yaml
jobs:
  python:
    uses: roshne/addon-ci/.github/workflows/python-app.yml@v1
    with:
      package-dir: .                    # what ruff/mypy target
      test-paths: tests
      requirements: requirements-dev.txt
      # runner: '["self-hosted","disposable"]'   # optional: run on your own runner
```

No secret required. All inputs are optional — `python-versions` sets which
versions to matrix over (default `["3.12"]`), and `package-dir`, `test-paths`,
`requirements`, `ruff-format`, `mypy-paths`, and `runner` all default to the
layout shown above.

## Actions

Composite actions, used as a **step** inside the caller's own job (unlike the
reusable workflows above, which run as their own job on their own runner).

### `playwright-smoke`

Runs a Playwright suite against a URL from inside the calling job, with Chromium
provided by Playwright's official container image -- for the assertions jsdom
cannot make (computed styles, media queries, layout, a page reconnecting after
its server restarts). The suite's own `npm ci` + `npx playwright test` run inside
the container with the workspace bind-mounted, so no browsers are installed on
the runner itself (the container writes `node_modules/`, `playwright-report/`
and `test-results/` under `spec-dir` on the runner's disk -- see the re-own note
below); the container joins the host network so a base URL bound on the runner's
loopback (a port a DinD runner published from an inner container included) is
reachable as-is. On failure the HTML report and `test-results` are uploaded as
an artifact.

It has to be a composite action, not a reusable workflow: a `workflow_call` job
lands on its own runner and cannot see the app the caller just booted. And it
runs the suite in a container rather than `playwright install --with-deps` on
the runner because an ephemeral DinD runner has no apt to install browser
dependencies into; the official image is the path the std-lib's visual job
already proves.

`--network host` is Linux-only Docker semantics: both the self-hosted DinD pool
and `ubuntu-latest` are Linux dockerd, so this works identically on either, but
the action is not usable from a Windows- or macOS-hosted runner (Docker
Desktop's VM-backed engine does not support host networking the same way) --
moot today since the pinned Playwright image is Linux-only anyway and would
fail to run there first, but worth knowing if that ever changes.

```yaml
- uses: roshne/addon-ci/.github/actions/playwright-smoke@main
  with:
    base-url: http://127.0.0.1:8787
    spec-dir: scripts/shell-smoke
    mount-docker-socket: "true"
    container-env: |
      AC_CONTAINER=ac-ratchet
```

| Input | Required | Default | What |
|---|---|---|---|
| `base-url` | yes | -- | The URL the suite runs against; exported to the container as `BASE_URL`. |
| `spec-dir` | no | `.` | Directory (relative to the workspace) with the suite's own `package.json` + `package-lock.json` and `playwright.config.*`; `npm ci` and `npx playwright test` run there. |
| `playwright-image` | no | `mcr.microsoft.com/playwright:v1.63.0-noble` | The Playwright container image. Its version must match the suite's `@playwright/test` pin. |
| `mount-docker-socket` | no | `"false"` | `"true"` bind-mounts `/var/run/docker.sock` so the spec can drive containers through the Docker Engine API (stop/start the app and assert the page reconnects). This is root-equivalent control of the runner's daemon -- only for a suite you own, on a runner thrown away after the job. |
| `extra-args` | no | `""` | Extra arguments for `npx playwright test`, split on whitespace only -- shell quoting is not interpreted, so no argument can contain a space. |
| `report-name` | no | `playwright-report` | Artifact name for the report uploaded on failure. |
| `container-env` | no | `""` | Newline-separated `KEY=VALUE` pairs exported into the container. Blank lines ignored, whitespace trimmed; a line without `=` fails the step. |

Files the container writes under `spec-dir` (`node_modules/`, `playwright-report/`,
`test-results/`) are re-owned to the runner user afterwards, so a persistent
runner's next checkout still cleans.

The first consumer (`Rackbops/artifact-console`'s image ratchet) pins `@main`
for now: `v1` predates the action, so there is nothing for `@v1` to resolve
until the tag is next moved deliberately (see **Versioning**), and it switches
to `@v1` then. That is a third, org-repo category distinct from both
**Versioning**'s two named ones -- `@v1` for org callers, `@main` by design for
the personal-account consumers below -- not `@<sha>` (Versioning's own stated
interim-verification mechanism), because there is no tagged version yet to
verify *against*; `@main` is this consumer's only option until the first `v1`
move. Treat this case as specific to a brand-new action with no tag history
yet, not as precedent for an org repo floating on `@main` once `v1` exists.

## Versioning

Callers pin `@v1`, not `@main`. A change lands on `main` and is verified on
one caller first -- point that caller at `@<sha>` temporarily, or use a
scratch repo -- before it's *released* to everyone else by moving the tag
deliberately:

```bash
git tag -f v1 <sha> && git push -f origin v1
```

Merging to `main` alone changes nothing for existing callers; only moving the
tag does. A breaking change to a workflow's inputs or secrets gets a `v2` tag
instead, so callers move on their own schedule rather than breaking on their
next run.

`@main` still works for the personal-account repos listed below -- it's just
no longer the documented way to consume these workflows.

## Consumers

- BetterMacroIcons
- HideCooldownManager
- LoadTimer
- ActionBarMaster (also uses `release.yml` + `publish.yml`)

Forks (MinimapButtonButton, AlreadyKnown) keep their own workflows because they
track upstream. Altoholic_Reborn is retired/archived.
