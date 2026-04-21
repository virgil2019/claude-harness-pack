---
name: finish-task
description: Wraps up a task in a git worktree. Scope-checks the diff against the plan, walks through done criteria, ensures clean commits, selects the right GitHub account (by SSH alias → gh username mapping in ~/.claude/gh-accounts.yml), pushes the branch, and opens a PR via gh. Invoke when user says "完成任务", "准备合入", "open PR", "finish task", or similar. Assumes the worktree was created by start-task and contains .task.md. Does NOT auto-merge.
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

### 6. Get the token for this account (stateless, no active-account change)

```bash
TOKEN=$(gh auth token -u "$USERNAME" 2>/dev/null)
[ -n "$TOKEN" ] || { echo "gh is not authed for $USERNAME. Run: gh auth login"; exit 1; }
```

**Why GH_TOKEN (not `gh auth switch`)**: the token is scoped to the command it's passed to, so we don't touch the globally active gh account. This is **race-safe for parallel worktrees** — two simultaneous finish-tasks on different repos won't clobber each other's account selection.

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
  git worktree remove ../<repo>-<slug>
  git branch -d <type>/<slug>
  git fetch --prune origin
  ```

### 10. Offer retrospective (optional)

After reporting, ask:
> "要跑 retrospect-task 做个复盘吗? 把这次任务里的纠正 / 摩擦 / 可复用模式 固化下来."

- If yes → invoke `retrospect-task` skill
- If no → finish. User can always invoke later.
- Skip asking if `checkpoint` was run recently (last 10 turns) and covered most of the task.

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
- Delete the worktree or branch (user's responsibility post-merge)
- Skip scope check
- Silently modify `gh-accounts.yml` without telling user
- Use `gh auth switch` / `export GH_TOKEN` — always scope the token to a single command via `GH_TOKEN=... gh ...`
