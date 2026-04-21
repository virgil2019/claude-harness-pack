# claude-harness-pack

A Claude Code plugin that packages seven reusable skills for harness engineering — the practice of shaping the environment around coding agents so they work reliably.

## What's inside

A single plugin, `harness-starter`, bundling seven skills organized around four concerns:

| Concern | Skills |
|---|---|
| Environment bootstrap | `init-claude-env` |
| Note-taking | `take-note` |
| Task workflow (worktree + PR) | `start-task`, `finish-task` |
| Evolution loop (learn as you go) | `codify-feedback`, `checkpoint`, `retrospect-task` |

See each skill's `SKILL.md` for detailed behavior.

## Install

Add the marketplace in your `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "claude-harness-pack": {
      "source": { "source": "github", "repo": "virgil2019/claude-harness-pack" }
    }
  }
}
```

Then inside Claude Code:

```
/plugin install harness-starter@claude-harness-pack
```

## Skills quick reference

- **`init-claude-env`** — Interactive Q&A bootstrap for a fresh `~/.claude/` (safety baseline + three-tier context).
- **`take-note`** — Appends a timestamped note to `~/memo.md`. Trigger: "记笔记" / "take a note".
- **`start-task`** — Creates a git worktree on a new branch with a plan + done criteria written to `.task.md`. Default workflow for non-trivial coding tasks.
- **`finish-task`** — Scope-checks the diff, verifies done criteria, pushes the branch, opens a PR via `gh` using the correct account (SSH alias → gh username mapping in `~/.claude/gh-accounts.yml`). Race-safe via per-command `GH_TOKEN`. Does NOT auto-merge.
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
