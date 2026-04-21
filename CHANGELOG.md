# Changelog

All notable changes to this project are documented in this file.

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
