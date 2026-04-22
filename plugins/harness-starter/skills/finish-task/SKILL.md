---
name: finish-task
description: Wraps up a task opened via start-task. Works in either mode — (A) in-place: user is on a branch in the primary tree, task file at `<REPO_ROOT>/.tasks/<slug>.md`; (B) worktree: user is in `.worktrees/<slug>/`, task file at `<worktree>/.task.md`. Scope-checks the diff against the plan, walks through done criteria, ensures clean commits, selects the right GitHub account (by SSH alias → gh username mapping in ~/.claude/gh-accounts.yml), pushes the branch, and opens a PR via gh. Uses per-command `GH_TOKEN=$(gh auth token -u X) gh ...` for account selection; **MUST NOT** use `gh auth switch`. After PR opens, offers optional retrospective. Local cleanup is NOT done here — use `cleanup-task` after merge. Invoke when user says "完成任务", "准备合入", "open PR", "finish task", or similar. Does NOT auto-merge.
---

# finish-task

## When to invoke

- User says "完成任务" / "finish task" / "准备合入" / "open PR" / "我搞定了"
- You are on a task branch created (or selected) via `start-task` — works in both modes:
  - **Mode A** (in-place): you are in the primary working tree, on a `<type>/<slug>` branch, task file at `<REPO_ROOT>/.tasks/<slug>.md`
  - **Mode B** (worktree): you are in `<REPO_ROOT>/.worktrees/<slug>/`, task file at `<worktree>/.task.md`

## Pre-flight — locate task file (handle both modes)

```bash
# Repo paths (absolute)
REPO_ROOT=$(git rev-parse --show-toplevel)
COMMON_DIR=$(git rev-parse --git-common-dir)
# If in a linked worktree, the primary repo root is above --git-common-dir
PRIMARY_ROOT=$(cd "$COMMON_DIR/.." && pwd)

BRANCH=$(git rev-parse --abbrev-ref HEAD)
# Derive slug from branch name: strip leading "<type>/"
SLUG="${BRANCH#*/}"

# Determine which mode we're in, based on where we are + where .task.md lives
TASK_FILE=""
if [ -f "$REPO_ROOT/.task.md" ]; then
  # Mode B: inside the worktree
  TASK_FILE="$REPO_ROOT/.task.md"
  MODE="B"
elif [ -f "$PRIMARY_ROOT/.tasks/$SLUG.md" ]; then
  # Mode A: primary tree, per-slug file
  TASK_FILE="$PRIMARY_ROOT/.tasks/$SLUG.md"
  MODE="A"
else
  echo "Warning: no task file found at either:"
  echo "  Mode B:  $REPO_ROOT/.task.md"
  echo "  Mode A:  $PRIMARY_ROOT/.tasks/$SLUG.md"
  echo "Proceed without plan reference? (lose scope check + auto-generated PR body)"
fi

# Base branch
BASE=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
[ -n "$BASE" ] || BASE=$(grep -m1 "^- \*\*Base\*\*:" "$TASK_FILE" 2>/dev/null | sed 's/.*: //')
```

> Use absolute `$TASK_FILE` in every subsequent Read/Write/Edit call. Bash's cwd may differ from session's; never use `./.task.md` relative.

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
# NOTE: use GH_USER (not USERNAME). zsh treats USERNAME as a special parameter
# bound to the system login name — any assignment is silently ignored, so the
# variable would carry `$USER` into downstream gh calls and pick the wrong account.
# Other zsh specials to avoid as local vars: HOME / PATH / PWD / UID / EUID /
# LOGNAME / SHELL / HOSTNAME / SECONDS / RANDOM / LINENO / COLUMNS / LINES.
GH_USER=$(grep -E "^${ALIAS}:" ~/.claude/gh-accounts.yml 2>/dev/null | sed 's/^[^:]*: *//' | tr -d '"'"'")
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
TOKEN=$(gh auth token -u "$GH_USER" 2>/dev/null)
[ -n "$TOKEN" ] || { echo "gh is not authed for $GH_USER. Run: gh auth login"; exit 1; }
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
  --body "$(cat "$TASK_FILE")" \
  --base "$BASE"
```

Note: prefix with `GH_TOKEN="$TOKEN"` **only for this command**. Do not `export` it — keep the shell's active gh account untouched.

### 9. Report

- PR URL (output from `gh pr create`)
- Reminder: **do not auto-merge** — user reviews + merges manually
- Cleanup **after PR is merged**: invoke the `cleanup-task` skill (it checks merge state via `gh` before deleting anything). Or do it manually:
  ```
  cd <main-repo>
  git worktree remove .worktrees/<slug>
  git branch -d <type>/<slug>     # -d only works if merged; use -D to force
  git fetch --prune origin
  ```
  **Do not delete local branch / worktree while the PR is still open** — reviewers may request changes that require editing this branch.

### 10. Offer retrospective (optional)

After reporting, ask:
> "要跑 retrospect-task 做个复盘吗? 把这次任务里的纠正 / 摩擦 / 可复用模式 固化下来."

- If yes → invoke `retrospect-task` skill
- If no → finish.
- Skip asking if `checkpoint` was run recently (last 10 turns) and covered most of the task.

### 11. Remind (do not act): cleanup after merge

**Do not offer to delete anything here** — the PR is still open; reviewers may need the branch for revisions.

Just remind the user:

> "PR 开完了。Merge 之后跑 `cleanup-task` skill 清理本地 worktree + 分支（会先查 PR 状态再删，安全）。或者手动: `cd <main-repo> && git worktree remove .worktrees/<slug> && git branch -d <branch> && git fetch --prune origin`."

## Error handling

| Issue | Response |
|---|---|
| Task file missing at both Mode A and Mode B locations | Ask user to proceed without plan reference (lose scope check + auto body) |
| On protected branch (main/master/dev/develop) | Warn loudly; ask user to confirm they really want to PR from protected branch — usually user missed start-task's branch-switch step |
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
- **Delete worktree / branch while PR is still open** — reviewers may request changes. Cleanup belongs in the separate `cleanup-task` skill, run after merge.
