# Changelog

All notable changes to this project are documented in this file.

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
