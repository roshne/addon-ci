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

## Consumers

- BetterMacroIcons
- HideCooldownManager
- LoadTimer
- ActionBarMaster

Forks (MinimapButtonButton, AlreadyKnown) keep their own workflows because they
track upstream. Altoholic_Reborn is retired/archived.

## Planned

- `release.yml` — reusable CurseForge publishing (BigWigsMods/packager) for when
  the first addon ships to CurseForge.
