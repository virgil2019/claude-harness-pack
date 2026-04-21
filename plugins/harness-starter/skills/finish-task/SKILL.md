---
name: finish-task
description: Wraps up a task in a git worktree. Scope-checks the diff against the plan, walks through done criteria, ensures clean commits, selects the right GitHub account (by SSH alias → gh username mapping in ~/.claude/gh-accounts.yml), pushes the branch, and opens a PR via gh. Uses per-command `GH_TOKEN=$(gh auth token -u X) gh ...` for account selection; **MUST NOT** use `gh auth switch` (stateless by design, race-safe for parallel worktrees). After PR is opened, offers optional retrospective and local cleanup (worktree + branch deletion, never on main / master / dev / develop). Invoke when user says "完成任务", "准备合入", "open PR", "finish task", or similar. Assumes the worktree was created by start-task and contains .task.md. Does NOT auto-merge.
---

# finish-task

## When to invoke

- User says "完成任务" / "finish task" / "准备合入" / "open PR" / "我搞定了"
- Assumes you are currently in a worktree created by `start-task`
- `.task.md` should exist in worktree root

## Pre-flight

```bash
# Must be in a worktree (not primary working tree)
GIT_DIR=$(git rev-parse --git-dir)
GIT_COMMON_DIR=$(git rev-parse --git-common-dir)
[ "$GIT_DIR" != "$GIT_COMMON_DIR" ] || { echo "Not in a worktree"; exit 1; }

# .task.md should exist
[ -f .task.md ] || echo "Warning: .task.md not found — continue without plan reference?"

# Base branch
BASE=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
[ -n "$BASE" ] || BASE=$(grep -m1 "^- \*\*Base\*\*:" .task.md 2>/dev/null | sed 's/.*: //')
```

## Flow

### 1. Scope check

```bash
git fetch origin "$BASE"
echo "=== Files changed ==="
git diff "origin/$BASE"...HEAD --name-only
echo ""
echo "=== Stat ==="
git diff "origin/$BASE"...HEAD --stat
```

Read `.task.md`'s "Affected" line. Compare. If files outside the planned scope appear:
- Show the unexpected files to user
- Ask: expected or accidental? If accidental, stop and let user clean up

### 2. Verify done criteria

Read `.task.md`, walk through each `- [ ]` item:
- **Auto-verifiable** (e.g. "run tests", "build passes"): run the command, show output, mark `- [x]` if green
- **Manual** (e.g. "UI looks right"): ask user to confirm, mark `- [x]` on confirmation
- Update `.task.md` in place so the final state records what was verified

If any criterion fails or is unconfirmed: **stop**. Do not proceed to PR.

### 3. Ensure clean commit state

```bash
git status --short
```

If not clean:
- Ask user how to handle staged / unstaged changes
- Help compose commit message(s); prefer conventional commit style matching `<type>` from `.task.md`

### 4. Detect GitHub account

```bash
REMOTE_URL=$(git remote get-url origin)
# Expected: git@<alias>:owner/repo.git
ALIAS=$(echo "$REMOTE_URL" | sed -nE 's@^git@([^:]+):.*$@\1@p')
```

- If no alias extracted (HTTPS remote, or non-SSH): warn, ask user which gh account to use
- If alias extracted: look it up in `~/.claude/gh-accounts.yml`

### 5. Look up gh username from mapping

```bash
USERNAME=$(grep -E "^${ALIAS}:" ~/.claude/gh-accounts.yml 2>/dev/null | sed 's/^[^:]*: *//' | tr -d '"'"'")
```

- If no mapping found:
  - Show `gh auth status` output so user can see authed usernames
  - Prompt: "Which gh username for `${ALIAS}`?"
  - Offer to write the mapping to `~/.claude/gh-accounts.yml` so next time it's automatic
- If mapping exists but gh not authed for that user:
  - Instruct user: `gh auth login --hostname github.com --git-protocol ssh --web` (then select right account)
  - Halt

### 6. Get the token for this account (⚠️ MUST use GH_TOKEN, NEVER `gh auth switch`)

```bash
TOKEN=$(gh auth token -u "$USERNAME" 2>/dev/null)
[ -n "$TOKEN" ] || { echo "gh is not authed for $USERNAME. Run: gh auth login"; exit 1; }
```

**🚫 FORBIDDEN in this skill**:
- `gh auth switch ...` (modifies globally active account → races with parallel worktrees)
- `export GH_TOKEN=...` (leaks token to later commands / subshells)

**✅ REQUIRED pattern for every gh call**:
```bash
GH_TOKEN="$TOKEN" gh <subcommand> ...
```

Token is scoped to a single command. Stateless. Race-safe across parallel worktrees. Do not deviate from this pattern — if the user (or another skill invocation) is simultaneously using gh elsewhere, `gh auth switch` will clobber their active account.

### 7. Push branch

```bash
git push -u origin HEAD
```

If push fails (e.g. non-ff): stop, show output, ask user — do NOT force-push.

(`git push` uses SSH routing via the remote URL's alias — unaffected by gh / GH_TOKEN.)

### 8. Open PR

Build title:
- `<type>: <description>` (from `.task.md` header)

Build body:
- Full `.task.md` content (plan + done criteria + notes)

```bash
GH_TOKEN="$TOKEN" gh pr create \
  --title "${TYPE}: ${DESCRIPTION}" \
  --body "$(cat .task.md)" \
  --base "$BASE"
```

Note: prefix with `GH_TOKEN="$TOKEN"` **only for this command**. Do not `export` it — keep the shell's active gh account untouched.

### 9. Report

- PR URL (output from `gh pr create`)
- Reminder: **do not auto-merge** — user reviews + merges manually
- Cleanup hint after merge:
  ```
  cd <original-repo>
  git worktree remove .worktrees/<slug>
  git branch -d <type>/<slug>
  git fetch --prune origin
  ```

### 10. Offer retrospective (optional)

After reporting, ask:
> "要跑 retrospect-task 做个复盘吗? 把这次任务里的纠正 / 摩擦 / 可复用模式 固化下来."

- If yes → invoke `retrospect-task` skill (needs `.task.md`, so must run BEFORE cleanup in step 11)
- If no → proceed to step 11.
- Skip asking if `checkpoint` was run recently (last 10 turns) and covered most of the task.

### 11. Offer local cleanup (worktree + branch)

**Safety filter (protected branches)**: determine current branch and skip this step entirely if it matches any protected pattern:

```bash
BRANCH=$(git rev-parse --abbrev-ref HEAD)
case "$BRANCH" in
  main|master|dev|develop|main/*|master/*|dev/*|develop/*|release/*|hotfix/*)
    # Never offer to delete these — skip step 11 silently
    echo "Protected branch ($BRANCH): skipping cleanup offer."
    exit 0 ;;  # or just `return`, depending on skill runtime
esac
```

Only proceed if branch looks like a task branch (e.g., `feat/*`, `fix/*`, `refactor/*`, `chore/*`, `docs/*`, `perf/*`, `test/*`).

**Ask the user**:
> "PR 已开。要顺手清理本地吗？会删除 worktree `.worktrees/<slug>/` 和本地分支 `${BRANCH}`。（远程分支和 PR 依然在 GitHub 上，merge 后再 fetch 就行。）"

Options:
- **Yes** — perform cleanup now
- **No** — keep local state; remind user to do manual cleanup after merge

**If user says yes**:

```bash
# 1. Find the main repo path (can't remove worktree while inside it)
MAIN_REPO=$(git worktree list | head -1 | awk '{print $1}')

# 2. Find this worktree's path by branch name
WORKTREE_PATH=$(git worktree list | awk -v b="[${BRANCH}]" '$3 == b {print $1}')

# 3. Leave the worktree
cd "$MAIN_REPO"

# 4. Check clean (should already be from step 3, but double-check)
git -C "$WORKTREE_PATH" diff --quiet && git -C "$WORKTREE_PATH" diff --cached --quiet || {
  echo "Worktree has uncommitted changes. Aborting cleanup."; exit 1;
}

# 5. Remove the worktree (plain, not --force)
git worktree remove "$WORKTREE_PATH"

# 6. Force-delete the local branch (-D because PR isn't merged yet)
git branch -D "$BRANCH"

# 7. Optional: prune stale remote tracking
git fetch --prune origin
```

**Confirm to user**:
- Worktree removed (path)
- Local branch deleted (`${BRANCH}`)
- Remote branch + PR **still on GitHub** — nothing lost
- To restore later: `git fetch origin ${BRANCH}:${BRANCH}`

**If user says no**:
- Remind: after PR is merged, run manual cleanup:
  ```
  cd <main-repo>
  git worktree remove .worktrees/<slug>
  git branch -d <type>/<slug>    # -d (safe) works after merge; use -D if not yet merged
  git fetch --prune origin
  ```

## Error handling

| Issue | Response |
|---|---|
| Not in worktree | Abort, explain |
| `.task.md` missing | Ask user to proceed without plan reference (lose scope check + auto body) |
| Dirty working tree | Help commit or stash |
| No SSH alias in remote URL | Ask user which gh account to use |
| No mapping in gh-accounts.yml | Prompt to add; offer to update the file |
| gh not authed for user | Instruct `gh auth login`, halt |
| Push rejected (non-ff) | Stop, show output, ask user. **Never force-push.** |
| gh pr create fails | Show error, ask user |

## Do NOT

- Auto-merge the PR
- Force-push
- **Use `gh auth switch`** (see step 6 — MUST use `GH_TOKEN=... gh ...` inline, every time)
- `export GH_TOKEN=...` (same reason — leaks token to later commands)
- Skip scope check
- Silently modify `gh-accounts.yml` without telling user
- Offer cleanup for protected branches (main / master / dev / develop / release/* / hotfix/*)
- Delete a worktree that has uncommitted changes (abort instead)
- Act on cleanup without user's explicit "yes"
