# claude-harness-pack

A Claude Code plugin that packages nine reusable skills for harness engineering — the practice of shaping the environment around coding agents so they work reliably.

## What's inside

A single plugin, `harness-starter`, bundling nine skills organized around four concerns:

| Concern | Skills |
|---|---|
| Environment bootstrap | `init-claude-env` |
| Note-taking | `take-note` |
| Task workflow (worktree + PR) | `init-repo`, `start-task`, `finish-task`, `cleanup-task` |
| Evolution loop (learn as you go) | `codify-feedback`, `checkpoint`, `retrospect-task` |

See each skill's `SKILL.md` for detailed behavior.

## Install

Inside Claude Code, two slash commands:

```
/plugin marketplace add virgil2019/claude-harness-pack
/plugin install harness-starter@claude-harness-pack
```

The first command writes to your `~/.claude/settings.json` automatically (no manual JSON editing needed). The second installs the plugin and activates the nine skills.

## Skills quick reference

- **`init-claude-env`** — Interactive Q&A bootstrap for a fresh `~/.claude/` (safety baseline + three-tier context).
- **`take-note`** — Capability-aware note capture: prefers an MCP note backend (Obsidian / Notion / etc.) when available, else appends to `~/memo.md`. Trigger: "记笔记" / "take a note".
- **`init-repo`** — Bootstraps a brand-new git + GitHub repo from an empty directory (interactive: name / account / visibility / description).
- **`start-task`** — Two modes: (A) in-place, write `.tasks/<slug>.md` and stay in the current dir; (B) worktree, create `.worktrees/<slug>/` and auto-spawn a new Claude Code session in tmux / iTerm2 / Terminal / Zellij. Auto-delegates to `init-repo` if not in a git repo.
- **`finish-task`** — Scope-checks the diff against the plan, verifies done criteria, pushes the branch, opens a PR via `gh` using the correct account (SSH alias → gh username mapping in `~/.claude/gh-accounts.yml`). Race-safe via per-command `GH_TOKEN`. Does NOT auto-merge.
- **`cleanup-task`** — Post-merge cleanup. Queries PR state via `gh` before deleting; refuses to touch protected branches; atomic single-shot to avoid leaving Bash stuck in a half-deleted worktree.
- **`codify-feedback`** — Captures a correction in the moment and writes it to the right scope (working-style / CLAUDE.md / project / memory). Proactively triggers when detecting ≥2 similar corrections within 5 turns.
- **`checkpoint`** — Mid-task lightweight pulse. Scans recent turns for lessons while the task is still in progress.
- **`retrospect-task`** — End-of-task comprehensive review. Four-question structured retrospective that feeds codify-feedback.

## Design principles

1. **Safety first** — permissions deny list, no force-push, no auto-merge, no hook bypass.
2. **Plan before execute** — worktree + plan + done criteria are always produced before code changes.
3. **Scope discipline** — `finish-task` enforces that the diff stays within the plan's stated scope.
4. **Deterministic routing** — gh account selected from git remote URL via SSH alias, not "active account" state. Race-safe for parallel worktrees.
5. **Evolution loop** — every task is an opportunity to update rules / create skills / refine memory. Three granularities: real-time (codify-feedback), mid-task (checkpoint), end-of-task (retrospect-task).

## Dependencies

- [`gh`](https://cli.github.com/) CLI (for `finish-task`)
- `git` ≥ 2.5 (for worktree support)
- `~/.claude/gh-accounts.yml` (mapping; `finish-task` will help populate)

## Updating

When a new version is published, in Claude Code:

```
/plugin update harness-starter@claude-harness-pack
```

See `CHANGELOG.md` for version history.

## License

MIT — see `LICENSE`.
