# addon-ci -- Claude Instructions

Centralized, **reusable** GitHub Actions workflows (`on: workflow_call`) for roshne's WoW addon
repos, so CI/release tooling lives in one place instead of being copy-pasted (and drifting) across
every repo. **This is not an addon and not an app** -- it ships no runtime code; consumers call its
workflows with `uses: roshne/addon-ci/.github/workflows/<file>.yml@main`. The repo is **public**.

My personal `~/.claude/CLAUDE.md` governs *how I work* -- the review gate, escalation, git &
shipping, commit mechanics, search-tool routing, and shell choice. It is **not restated here**; this
file covers only what is specific to this repo.

**Commit convention (as the log shows):** Conventional Commits `type(scope): subject`, the subject
usually ending `(#N)`. Types seen: `feat`, `fix`, `docs`, `chore`, `ci`. Scope is the workflow/subsystem
touched -- `release`, `publish`, `push-notify`, `python-app`, ... Match what the log already shows.

---

## Ground truth

The **workflow YAML on disk is the source** (`.github/workflows/*.yml`); cite `file:line`.
`README.md` documents each workflow and the exact caller snippet a consumer pastes -- keep the two
in sync when a workflow's inputs, secrets, or trigger change. Six reusable workflows exist today:
`lua-test`, `release`, `publish`, `discord`, `push-notify`, `python-app`.

**There is deliberately no `CONTEXT.md`.** A workflows-only repo has no paid-for-once toolchain
ledger beyond the YAML itself plus the README -- a `CONTEXT.md` would be dead weight (same call as
`battlenet-api-research`; see `claude-md-authoring.md` in claude-memory-sync).

---

## Irreversible: the `@main` contract every consumer resolves by

Consumers pin `roshne/addon-ci/.github/workflows/<file>.yml@main`, so **this repo has no review seam
of its own** -- whatever lands on `main` is live for every consumer on its next run.

- **A workflow file's path is the contract.** Renaming or moving `<file>.yml` breaks every caller's
  `uses:` line invisibly -- the failure lands in the *consumer's* pipeline, caught by no check here.
- **A backward-incompatible change** to a workflow's inputs, required secrets, or behaviour ripples
  to all `@main` consumers at once. Treat it as a shipped-identifier change -> **stop and escalate**
  per personal's **Escalation**, and land it so existing callers keep working (new inputs defaulted).
- Consumers today: BetterMacroIcons, HideCooldownManager, LoadTimer, ActionBarMaster.

---

## Testing & checks

**This repo has no CI of its own** -- every workflow is `on: workflow_call`, so nothing runs on
addon-ci's own pushes or PRs. The consumers' pipelines are the only place a change is exercised, so a
change here is **unvalidated until a downstream run**. Lint the YAML locally (`actionlint`) before
pushing, and after a behaviour change trigger or watch a real consumer run rather than assuming
green. Per personal's **Done means**, name what only a downstream run can prove.

---

## Code style

Follows personal's **Code style** baseline. This repo's individuality:

- It is almost entirely **GitHub Actions YAML with embedded `bash`** in `run:` blocks (plus
  `python-app.yml`, which *invokes* Python tooling -- ruff/mypy/pytest -- but hosts no Python code of
  its own). The bash baseline applies inside every `run:` block -- `set -euo pipefail`, quote every expansion.

---

## Key gotchas

- **Never interpolate untrusted text into a `run:` shell via `${{ ... }}`.** A release body, PR
  title, tag, or webhook payload spliced straight into a shell command is a code-injection hole. Pass
  it through `env:` and reference `"$VAR"` inside the script. (Paid for once: commit `f0e8483 fix:
  stop interpolating release body directly into shell`.)
