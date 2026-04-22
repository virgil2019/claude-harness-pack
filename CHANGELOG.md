# Changelog

All notable changes to this project are documented in this file.

## [0.6.0] — 2026-04-22

### Added — `start-task` Mode A (in-place)

`start-task` now asks the user to pick a mode before creating anything:

- **Mode A · 就地模式**: stay in current directory. If the current branch is protected (`main`/`master`/`dev`/`develop`), offers to `git checkout -b <type>/<slug>`. Writes task file to `<REPO_ROOT>/.tasks/<slug>.md` — **persistent across task cleanups**, scannable as a historical record. Adds `.tasks/` to `.gitignore` on first use.
- **Mode B · 新 worktree 模式**: original behavior — `.worktrees/<slug>/` + new branch + cd + `.task.md`.

Picks addressed:
- Users who pre-check-out their own branch no longer get a forced duplicate worktree.
- Lightweight tasks in an existing checkout are first-class (no worktree tax).
- Historical task records survive `cleanup-task` (Mode A files outlive the worktree).

### Changed

- **`start-task` pre-flight**: hardened to MUST actually run `git rev-parse --is-inside-work-tree` and `git remote` via Bash, then decide. Explicit rule: **task type `feat` does NOT imply "new repo"** — only git output signals init-repo delegation. Fixes the earlier misrouting where Mode-B-on-feature-branch went to init-repo.
- **`start-task`**: requires `Write` with **absolute paths** for task file (both modes). Documents the cwd trap — `cd` only affects Bash, not Read/Write/Edit.
- **`finish-task`**: locates task file at either `<REPO_ROOT>/.task.md` (Mode B) or `<PRIMARY_ROOT>/.tasks/<slug>.md` (Mode A). Derives slug from branch name. No longer hard-requires being in a worktree.
- **`retrospect-task`**: same dual-location lookup as finish-task.

### Notes

Existing v0.5.x tasks (only Mode B existed) continue to work — their `.task.md` at worktree root is still recognized.

## [0.5.1] — 2026-04-22

### Fixed
- **Bash variable `USERNAME` was silently ignored under zsh**, causing `finish-task` / `cleanup-task` / `init-repo` to fall back to default gh account (e.g. opening a PR under the wrong user). `USERNAME` is a zsh special parameter bound to the system login name; assignments don't stick. Renamed to `GH_USER` in all three skills, with inline comments documenting the zsh specials to avoid as local variables (`HOME` / `PATH` / `PWD` / `UID` / `EUID` / `LOGNAME` / `SHELL` / `HOSTNAME` / `SECONDS` / `RANDOM` / `LINENO` / `COLUMNS` / `LINES`). Affects Claude Code installations where the Bash tool runs zsh (most macOS setups).

## [0.5.0] — 2026-04-21

### Added
- **`cleanup-task`** skill — dedicated post-merge cleanup. Queries PR state via `gh` before deleting; only proceeds with explicit user confirmation. Uses `-d` (safe delete) for merged branches, `-D` (force) only when user explicitly accepts losing local state (e.g. PR closed-not-merged). Never touches protected branches (main / master / dev / develop / release/* / hotfix/*). Same GH_TOKEN-inline discipline as finish-task.

### Changed
- **`finish-task` step 11 reverted**: no longer offers to delete local worktree + branch. Rationale: PR is still open at that point; reviewers may need the branch for revisions. Step 11 now just reminds the user to run `cleanup-task` **after merge**.

## [0.4.0] — 2026-04-21

### Added
- **`finish-task` step 11**: after opening the PR, offers to clean up local worktree + branch. Only applies to task branches (`feat/*` / `fix/*` / `refactor/*` / `chore/*` / `docs/*` / `perf/*` / `test/*`); **never offers cleanup for protected branches** (`main` / `master` / `dev` / `develop` / `release/*` / `hotfix/*`). User must explicitly confirm. Remote branch + PR stay on GitHub.

### Changed
- **`finish-task` GH_TOKEN enforcement hardened**: description, step 6, and Do-NOT list now explicitly forbid `gh auth switch` and `export GH_TOKEN`. Every `gh` call MUST be prefixed `GH_TOKEN="$TOKEN" gh ...` inline. Prevents race conditions when multiple worktrees invoke finish-task simultaneously.

## [0.3.0] — 2026-04-21

### Changed
- **`start-task`** now creates worktrees **inside the repo** at `<repo>/.worktrees/<slug>/` (previously sibling-level at `../<repo>-<slug>/`). Cleaner workspace root, all worktrees co-located, auto-gitignored.
- `start-task` automatically adds `.worktrees/` to `.gitignore` on first use if not already present.
- **`finish-task`** cleanup hint updated to the new worktree location.

## [0.2.0] — 2026-04-21

### Added
- **`init-repo`** skill — interactively bootstraps a brand-new git + GitHub repo from the current directory (asks for repo name / account / visibility / description, runs git init, creates initial commit, creates the GitHub repo via `gh`, sets remote with SSH alias from `~/.claude/gh-accounts.yml`).

### Changed
- **`start-task`** now delegates to `init-repo` when the current dir is not a git repo or has no `origin` remote. Previously it aborted with an error — now it bootstraps gracefully.
- **`take-note`** is now **capability-aware**: detects MCP note-taking backends (Obsidian, Notion, Logseq, etc.) at invocation time and prefers them when available; falls back to `~/memo.md` only when no MCP note backend is connected. No hardcoded dependency on any specific tool.

## [0.1.0] — 2026-04-21

Initial release.

### Plugin: `harness-starter`

Bundles seven skills:

- `init-claude-env` — Interactive bootstrap for a fresh `~/.claude/` (safety baseline + three-tier context).
- `take-note` — Append timestamped notes to `~/memo.md`.
- `start-task` — Git worktree + branch + plan + done criteria.
- `finish-task` — Scope check + done verification + push + PR (race-safe via `GH_TOKEN`).
- `codify-feedback` — Real-time codification of user corrections into durable rules.
- `checkpoint` — Mid-task lightweight pulse.
- `retrospect-task` — End-of-task comprehensive review.
