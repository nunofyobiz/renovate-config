# nunofyobiz/renovate-config

Renovate configurations for this organization

[![CI](https://github.com/nunofyobiz/renovate-config/actions/workflows/ci.yml/badge.svg)](https://github.com/nunofyobiz/renovate-config/actions/workflows/ci.yml)


# Local setup

A [pre-commit](https://pre-commit.com/) hook validates every `*.json5` config before
each commit, so problems are caught before they reach CI. Install it once per clone:

```bash
brew install pre-commit   # or: pipx install pre-commit
pre-commit install
```

To validate everything on demand (the same check CI runs):
```bash
pre-commit run --all-files
```


# Agent commit signing

`main` requires signed commits. When you run an agent (Claude Code, etc.) inside a
`claude/*` worktree, `scripts/setup-claude-worktree-git.sh` configures a per-worktree
identity that signs the agent's commits with a dedicated SSH key, authored as
`Claude Code (<you>)` — without touching your personal git config on other branches.

The one-time per-machine setup (SSH signing key, `~/.gitconfig.claude`, registering the
key on GitHub) is shared with our other repos and documented in StoryCut's guide:
<https://github.com/StoryCut/StoryCut/blob/main/docs/dev-guides/agents-signing.md>

Once that's done, run the script in a `claude/*` worktree to apply it:
```bash
./scripts/setup-claude-worktree-git.sh
```


# Commands

## Validate a config locally

For a specific config file, eg. `js-lib.json5`:
```bash
npx --yes --package renovate -- renovate-config-validator js-lib.json5
```