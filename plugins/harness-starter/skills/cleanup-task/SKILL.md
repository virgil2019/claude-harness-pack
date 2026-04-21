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
USERNAME=$(grep -E "^${ALIAS}:" ~/.claude/gh-accounts.yml 2>/dev/null | sed 's/^[^:]*: *//' | tr -d '"'"'"' ')
TOKEN=$(gh auth token -u "$USERNAME" 2>/dev/null)
[ -n "$TOKEN" ] || { echo "gh not authed for $USERNAME. Run: gh auth login"; exit 1; }
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

### 4. Perform cleanup

```bash
# Leave the worktree first (can't remove while inside)
cd "$MAIN_REPO"

# Remove the worktree
git worktree remove "$WORKTREE_PATH"

# Delete the local branch
#   -d if PR was MERGED (safe: git will verify branch is reachable from main)
#   -D if CLOSED-not-merged (user explicitly accepted losing local-only changes)
#   -D if OPEN (user explicitly accepted)
if [ "$STATE" = "MERGED" ]; then
  git branch -d "$BRANCH"
else
  git branch -D "$BRANCH"
fi

# Prune stale remote-tracking refs
git fetch --prune origin
```

### 5. Report

Tell user:
- Worktree removed: `<path>`
- Local branch deleted: `<branch>`
- Remote branch: still on GitHub (if PR was merged, GitHub's auto-delete-on-merge may have already removed it — check)
- PR URL: `<url>` (if it existed)
- To restore later: `git fetch origin <branch>:<branch>` (only works if remote branch still exists)

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

## Relationship to other skills

- **`finish-task`**: opens the PR. Does NOT cleanup (PR still open / reviewers may need branch).
- **`cleanup-task`** (this skill): ran by user **after merge** to remove local artifacts.
- Together they bracket the task lifecycle: start-task → work → finish-task → (merge on GitHub) → cleanup-task.
