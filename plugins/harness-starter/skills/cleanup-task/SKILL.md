---
name: cleanup-task
description: Cleans up the local worktree + branch AFTER a PR has been merged. First queries the PR state via gh to decide whether cleanup is safe. Invoke when the user says "清理任务", "cleanup task", "清理分支", "删除分支", "PR merged 了清理一下", or similar post-merge cleanup intents. NEVER touches protected branches (main / master / dev / develop / release/* / hotfix/*). If the PR is still open, warns the user and asks for explicit confirmation before proceeding. Uses per-command `GH_TOKEN=$(gh auth token -u X) gh ...` — never `gh auth switch`.
---

# cleanup-task

Safely clean up a task's local artifacts (worktree + branch) **after its PR is merged**. Counterpart of `finish-task`.

## When to invoke

- User says: "清理任务" / "cleanup task" / "清理分支" / "删除分支" / "PR merged 了清理一下" / "cleanup after merge"
- Context: a PR was previously opened (usually via `finish-task`) and the user has merged it on GitHub
- Do NOT invoke as part of `finish-task` — the PR is still open there

## Pre-flight

### 1. Identify target branch + worktree

Two ways to invoke:

**A. From inside the worktree** (most common):
```bash
BRANCH=$(git rev-parse --abbrev-ref HEAD)
WORKTREE_PATH=$(pwd)
MAIN_REPO=$(git worktree list | head -1 | awk '{print $1}')
```

**B. From main repo (or elsewhere)**:
- Ask user: "Which branch/worktree to clean up?" — offer a list from `git worktree list` (skip main)
- Determine `BRANCH` and `WORKTREE_PATH` from user's pick

### 2. Safety filter (protected branches)

```bash
case "$BRANCH" in
  main|master|dev|develop|main/*|master/*|dev/*|develop/*|release/*|hotfix/*)
    echo "Protected branch ($BRANCH): refusing to clean up."
    exit 1 ;;
esac
```

If the branch is protected, **refuse and exit**. Never delete these.

### 3. Determine the correct gh account (same pattern as finish-task)

```bash
# cd to MAIN_REPO first to read origin from main repo (worktree shares config, but be explicit)
cd "$MAIN_REPO"

REMOTE_URL=$(git remote get-url origin)
ALIAS=$(echo "$REMOTE_URL" | sed -nE 's@^git@([^:]+):.*$@\1@p')
# NOTE: use GH_USER (not USERNAME). zsh treats USERNAME as a special parameter
# bound to the system login name — assignment is silently ignored. See the
# same note in finish-task's step 5 for the full list of zsh specials to avoid.
GH_USER=$(grep -E "^${ALIAS}:" ~/.claude/gh-accounts.yml 2>/dev/null | sed 's/^[^:]*: *//' | tr -d '"'"'"' ')
TOKEN=$(gh auth token -u "$GH_USER" 2>/dev/null)
[ -n "$TOKEN" ] || { echo "gh not authed for $GH_USER. Run: gh auth login"; exit 1; }
```

**⚠️ MUST use `GH_TOKEN=... gh ...` inline, NEVER `gh auth switch`.**

## Flow

### 1. Query PR state

```bash
PR_JSON=$(GH_TOKEN="$TOKEN" gh pr view "$BRANCH" --json state,mergedAt,closedAt,url 2>/dev/null)
# Possible states: OPEN, MERGED, CLOSED
STATE=$(echo "$PR_JSON" | jq -r '.state')
MERGED_AT=$(echo "$PR_JSON" | jq -r '.mergedAt // ""')
PR_URL=$(echo "$PR_JSON" | jq -r '.url // ""')
```

If `gh pr view` fails: no PR found for this branch → tell user, ask if they want to clean up anyway (maybe they deleted the PR, or it never existed).

### 2. Decide based on state

| State | Behavior |
|---|---|
| `MERGED` | ✅ Safe → confirm "PR #X merged at <time>. Cleanup now?" → proceed on yes |
| `CLOSED` (not merged) | ⚠️ Ask: "PR closed without merging. Still clean up? (will lose local changes)" → proceed only on explicit yes |
| `OPEN` | ⚠️ Warn: "PR is still OPEN. Are you sure? Reviewers may need this branch." → proceed only on explicit yes |
| (no PR found) | Ask user for context, then confirm |

**Never proceed silently**. Always an explicit yes from the user.

### 3. Verify worktree is clean

```bash
cd "$WORKTREE_PATH"
git diff --quiet && git diff --cached --quiet || {
  echo "Worktree has uncommitted changes at $WORKTREE_PATH. Aborting cleanup."
  echo "Commit / stash / discard those first."
  exit 1;
}
```

If dirty: abort, show `git status`, tell user to handle changes first.

### 4. Perform cleanup — **atomic, single Bash call**

Run all three steps in **ONE Bash tool call** (use `&&` / sequential chain). Reasons:
- Bash tool's persisted cwd is set once by the initial `cd`; if cleanup steps are split across multiple tool calls, any mid-step failure can leave the persisted cwd pointing at a half-deleted directory → subsequent `chdir()` fails → the session's Bash gets stuck
- Atomic cleanup also means one log to report

Template (decide `-d` vs `-D` based on `$STATE` from step 2):

```bash
cd "$MAIN_REPO" || { echo "cd to $MAIN_REPO failed, aborting"; exit 1; }

git worktree remove "$WORKTREE_PATH" \
  && { [ "$STATE" = "MERGED" ] && git branch -d "$BRANCH" || git branch -D "$BRANCH"; } \
  && git fetch --prune origin \
  && echo "✅ cleanup done"
```

Use `git branch -d` only when `$STATE` = `MERGED` (safe). Use `-D` (force) when `$STATE` = `CLOSED-not-merged` or `OPEN` — only after explicit user yes in step 2.

### 5. Report — **terse + exit hint**

Keep output to 5-6 lines total. No prose, no re-explanation of what cleanup-task does. Just facts + exit hint:

```
✅ 清理完成
· worktree removed: <WORKTREE_PATH>
· branch deleted: <BRANCH> (<-d or -D>)
· remote pruned

🚪 建议 /exit 关闭当前 session — 本 session 可能还缓存了已删目录的状态. 下次 `cd <MAIN_REPO> && claude` 起新 session.
```

> ⚠️ Do NOT append verbose recovery instructions ("如果 bash 卡了, 先 cd ~ 然后..."), do NOT list PR URL again, do NOT explain the fetch --prune purpose. The user already read this skill's description once; don't repeat.

## Error handling

| Issue | Response |
|---|---|
| Protected branch | Refuse, explain |
| Can't determine main repo / worktree path | Ask user to specify |
| gh not authed | Instruct `gh auth login`, halt |
| No PR found | Ask user whether branch has ever been pushed; proceed only on explicit confirmation |
| PR OPEN or CLOSED-not-merged | Extra warning, require explicit confirmation |
| Dirty worktree | Abort, show status, tell user to handle |
| `git worktree remove` fails | Show error, ask user (never `--force`) |
| `git branch -d` fails (unmerged) | If PR was MERGED, this means branch isn't actually on main — warn user, ask before using -D |

## Do NOT

- Use `gh auth switch` (violates the GH_TOKEN-inline rule — same as finish-task)
- `export GH_TOKEN=...`
- `--force` worktree removal
- Silently proceed when PR is open or closed-not-merged
- Delete a protected branch (main / master / dev / develop / release/* / hotfix/*)
- Delete the remote branch (that's GitHub's / user's call, not this skill's)
- Run on a dirty worktree
- **Split cleanup into multiple Bash tool calls** — atomic single call only; otherwise Bash's persisted cwd can end up pointing at the half-deleted worktree and all subsequent commands fail with chdir errors
- **Append verbose recovery instructions** after a successful cleanup — trust the exit hint, don't re-explain

## Relationship to other skills

- **`finish-task`**: opens the PR. Does NOT cleanup (PR still open / reviewers may need branch).
- **`cleanup-task`** (this skill): ran by user **after merge** to remove local artifacts.
- Together they bracket the task lifecycle: start-task → work → finish-task → (merge on GitHub) → cleanup-task.
