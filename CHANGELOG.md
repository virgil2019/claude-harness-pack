# Changelog

All notable changes to this project are documented in this file.

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
