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
    uses: roshne/addon-ci/.github/workflows/lua-test.yml@main
```

Update the workflow here once and every repo pointing at `@main` picks it up on
its next run.

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
    uses: roshne/addon-ci/.github/workflows/release.yml@main
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
    uses: roshne/addon-ci/.github/workflows/publish.yml@main
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
    uses: roshne/addon-ci/.github/workflows/discord.yml@main
    secrets: inherit
```

Requires a `DISCORD_WEBHOOK` repo secret (Discord channel → Integrations →
Webhooks). `workflow_dispatch` fires a test notification without merging
anything.

## Consumers

- BetterMacroIcons
- HideCooldownManager
- LoadTimer
- ActionBarMaster (also uses `release.yml` + `publish.yml`)

Forks (MinimapButtonButton, AlreadyKnown) keep their own workflows because they
track upstream. Altoholic_Reborn is retired/archived.
