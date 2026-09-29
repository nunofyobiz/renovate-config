# renovate-config — agent guide

`CLAUDE.md` is a symlink to this file. Source of truth.

When you update this file, **rewrite the affected section cohesively** — don't append patches to the
bottom. The next reader (human or agent) should be able to scan a section top-to-bottom without
archaeology.

---

## What this repo is

`nunofyobiz/renovate-config` holds the organization's shared [Renovate](https://docs.renovatebot.com/)
configuration presets — the `*.json5` files at the repo root (`default.json5`, `js-base.json5`,
`js-lib.json5`, `js-app.json5`, `org.json5`, `neon-local-postgres.json5`, `renovate.json5`). Other repos
extend these presets (e.g. `extends: ["github>nunofyobiz/renovate-config:js-lib.json5"]`). There is
**no application code and no `package.json`.** Two scripts run here: `scripts/validate-renovate-configs.sh`
(the validator, see below) and `scripts/setup-claude-worktree-git.sh` (fired by the `SessionStart` hook
in `.claude/settings.json` to configure per-worktree commit signing on agent branches).

## Presets and how they layer

| File | Purpose | Extends |
| --- | --- | --- |
| `default.json5` | Onboarding default for new repos; currently assumes every repo is a JS app | `js-app.json5` |
| `org.json5` | Org-wide settings: PR labels, CODEOWNERS-based reviewers, timezone | (nothing) |
| `js-base.json5` | Rules shared by JS apps and libraries: schedule, concurrency, `minimumReleaseAge`, automerge, priority | (nothing) |
| `js-lib.json5` | JS library config: Renovate's `config:js-lib` (widens dependency ranges) + library-only rules (e.g. disabling `engines` updates) | `config:js-lib`, `org.json5`, `js-base.json5` |
| `js-app.json5` | JS app config: Renovate's `config:js-app` (pins exact versions) | `config:js-app`, `org.json5`, `js-base.json5` |
| `neon-local-postgres.json5` | Add-on for repos using local Postgres via Neon: pins the postgres image version | (nothing; opt-in, no other preset extends it) |
| `renovate.json5` | This repo's own Renovate config | `js-lib.json5` |

Where a new rule belongs:
- Shared by both apps and libraries → `js-base.json5`.
- Library-only (e.g. the `engines` rule) → `js-lib.json5`.
- App-only → `js-app.json5`.
- Org-wide (labels, reviewers, timezone) → `org.json5`.
- A stack-specific opt-in that not every repo wants → its own add-on file, like `neon-local-postgres.json5`.

## Editing conventions

- `$schema` comes first in every preset (mixed quoted/unquoted key style across files — don't normalize
  it in unrelated edits).
- Most shared presets open with a header block comment explaining what they're for; use one for new
  presets. The existing `neon-local-postgres.json5` add-on and this repo's `renovate.json5` predate
  that convention.
- `/* **** SECTION: … **** */` banners group related entries inside `packageRules`.
- `NOTE(tag)` comments hold long explanations and are referenced elsewhere as "See NOTE(tag)"
  (e.g. `js-base.json5`'s `NOTE(buildAllBranches)` and `NOTE(runAllDay)`).
- Comments explain *why* a setting exists, often linking to Renovate docs.
- Presets reference each other with the full `github>nunofyobiz/renovate-config:<file>` path, never a
  relative path.

## How changes ship

Merging to `main` **is** the release — there are no version tags or releases. Consuming repos extend
`github>nunofyobiz/renovate-config:<file>`, which reads straight from `main`, so a merged change is live
for every consuming repo on its next Renovate run. Preset file names are a public API: renaming or
deleting one breaks every repo that extends it. The validator only checks each file in isolation — it
doesn't show how presets combine downstream in a consuming repo — so call out the downstream effect of a
change in the PR description.

## Verification ritual

Before claiming a task done — whenever you add or edit any `*.json5` preset — run the validator:

```
bash scripts/validate-renovate-configs.sh
```

It validates every `*.json5` config one file at a time with the exact `renovate` version pinned in
[`.pre-commit-config.yaml`](./.pre-commit-config.yaml) (via `npx renovate-config-validator --strict`),
prints a per-config PASS/FAIL table, and exits non-zero on any failure. This is the same check CI runs.
Node ≥ 22 is the only prerequisite (pinned in `.nvmrc`); run `nvm use` first only if a bare invocation
reports an engine error.

Two traps:
- The validator discovers configs via `git ls-files '*.json5'`, so a **new** preset file must be
  `git add`-ed before the validator will see it at all.
- The **first** run downloads Renovate (~300 MB) through `npx` and is slow; it needs network access.
  Later runs reuse the cached install and are fast.

## Permission-prompt posture (command shape)

Most permission prompts are a **command-shape problem, not a missing allowlist entry** — the engine
auto-approves a command only when *every* part of it is independently safe, so several wrappers defeat
an otherwise-allowlisted command. The allowlist in [`.claude/settings.json`](./.claude/settings.json)
covers the safe reads and the validator script; you keep prompts low by *how* you invoke things:

- **Prefer the built-in `Grep` / `Glob` / `Read` tools** over shell `grep` / `git grep` / `find` /
  `cat` / `sed` / `head` and pipelines. They don't go through the Bash permission path, so they never
  prompt — and they're faster. Concretely:
  - `sed -n 'A,Bp' file` / `head -n N file` → `Read` with `offset` / `limit`.
  - `grep … file` / `git grep …` → the `Grep` tool. Scope with `glob` (e.g. `**/*.json5`) and exclude
    paths with a negated glob instead of a `| grep -v` pipe.
  - **Reading or searching N files → one `Grep`/`Read`, never a `for f in …; do grep …; done` loop.**
  - Drop `echo "=== … ==="` section separators and `|| echo "no matches"` fallbacks entirely — the
    native tools label their output and report empty results for free. A stray `echo` is enough to
    force a prompt on an otherwise-allowlisted block.
- **One command per tool call — never `&&` / `;` / `|` / `$(…)` chains or `for`/`while` loops.** A
  compound line is auto-approved only if every segment independently clears, so even an all-allowlisted
  chain (`git add … && git commit … && git log …`) prompts. Run the steps as separate calls. Loops and
  command substitution prompt structurally — reach for the native tools above instead of trying to
  allowlist your way around it.
- **Run the validator verbatim** — `bash scripts/validate-renovate-configs.sh`, with no env-var prefix
  (`TMPDIR=…`) and no `2>&1 | tail` / `2>/dev/null` capture wrapper. Run it bare and read the output;
  the redirect/pipe is itself what prompts.
- **Commit with `-m` (or `-F <file>`), never a heredoc.** `git commit <<'EOF' … EOF` is an input
  redirect and prompts on *every* commit.
- **Don't `cd` / `git -C <path>` into the worktree you're already in** — an out-of-cwd path triggers a
  prompt. The cwd already *is* the repo; run `git status`, the validator, etc. directly.
- **Don't prefix a git subcommand with `git -c <k>=<v>` or a global flag** (`--no-pager`,
  `-c color.ui=never`) — the leading flag defeats the `Bash(git <subcmd> *)` allowlist match and
  prompts, and it's redundant anyway: the harness already returns uncolored, unpaged output. Run the
  subcommand plain.
- **Don't reach for `nvm ls` to diagnose Node state** — it's degraded/useless on the sandbox and tells
  you nothing; it's allowlisted only as a harmless seatbelt for when it's run anyway. Node-version
  setup is handled centrally, not per-repo.

Consequential actions stay **deliberately gated** (they *should* prompt): bare `npx` (arbitrary code
execution — only the validator *script* is allowlisted, not raw `npx`), `gh pr merge`, `gh issue
create`, and destructive git (`git reset --hard`, `git clean -fd`, bare `git push --force`). Expanding
the allowlist is the smallest lever, not the first — fix the root cause in command shape before
reaching for it.

## Commits and PRs

- **Conventional Commits**, enforced by `commitlint` in CI (`commitlint.config.cjs`). Types: `feat`,
  `fix`, `refactor`, `chore`, `docs`, `test`, `perf`, `style`, `ci`, `build`. The PR title is itself a
  Conventional Commit. CI also rejects `WIP` / `DNM` markers in commit messages and the PR title.
  Commit headers are capped at 100 characters (the `@commitlint/config-conventional` default); body
  lines are capped at 200 (raised from the default in `commitlint.config.cjs` to fit Dependabot commits).
- **Atomic commits** — one cohesive change each.
- On an **unmerged branch**, amending / squashing / reordering is fine; update an open PR with
  `git push --force-with-lease` (never bare `--force`). Rebase onto `main` rather than merging it in, and
  never leave a `Revert "…"` commit on the branch — `.github/semantic.yml` sets `allowMergeCommits: false`
  and `allowRevertCommits: false` and validates both the PR title and every commit.
- `main` requires signed commits. Signing is configured automatically only on `claude/*` and `agent/*`
  branches (see README.md → "Agent commit signing"); other branches need a manually-signed commit or a
  rebase before merging.
- CODEOWNERS (`@bigpopakap`) is the reviewer, assigned automatically via `org.json5`'s
  `reviewersFromCodeOwners`.
- Run the validator before pushing. Open a PR; don't merge unless asked.

## CLAUDE.md ↔ AGENTS.md

`CLAUDE.md` is a symbolic link to this file. If you find yourself editing both, you've broken the
symlink — restore it with `ln -sf AGENTS.md CLAUDE.md` from the repo root.
