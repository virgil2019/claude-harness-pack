---
name: start-task
description: Bootstraps a non-trivial coding task by creating a git worktree on a new branch, producing a plan + done criteria, and writing them to .task.md in the worktree. Invoke when the user says "开始任务", "新任务", "start a task", or when starting any non-trivial code change that should go through the worktree + PR workflow. If the current dir is not a git repo or has no origin remote, delegates to init-repo first. Do not invoke for trivial fixes (single-line typo, format) or read-only exploration.
---

# start-task

## When to invoke

- User says "开始任务" / "start a task" / "新任务" / "做一下 X"
- Any non-trivial code change that will need review via PR
- **Skip** if: read-only exploration, trivial fix (single-line typo, format), or in-place quick patch

## Inputs

1. Task description (from user message or ask)
2. Task type — one of: `feat` / `fix` / `refactor` / `docs` / `chore` / `perf` / `test`
3. Quality target — one of: `prototype` / `commercial` / `production` (from working-style.md quality bar)

If any is unclear, ask before proceeding.

## Pre-flight

### 0. Bootstrap check — delegate to `init-repo` if needed

Before anything else, verify the current dir is a ready-to-work repo:

```bash
IS_REPO=$(git rev-parse --is-inside-work-tree 2>/dev/null && echo yes || echo no)
HAS_ORIGIN=$(git remote 2>/dev/null | grep -qx origin && echo yes || echo no)
```

If either is `no`:

- Tell user: **"当前目录不是 git repo (或没有 origin)。先 init-repo 把它配好再继续。"**
- **Invoke the `init-repo` skill** to interactively bootstrap
- After `init-repo` completes, resume this skill's flow from step 1 below
- If user declines, abort start-task

If both are `yes`, proceed.

### 1. Repo info

```bash
# Determine base branch (usually main or master)
BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
# If the above fails (no origin HEAD set), fall back:
if [ -z "$BASE_BRANCH" ]; then
  BASE_BRANCH=$(git branch --list main master | head -1 | tr -d '* ')
fi

# Repo name for path building
REPO_ROOT=$(git rev-parse --show-toplevel)
REPO_NAME=$(basename "$REPO_ROOT")
```

- Warn if working tree has uncommitted changes (don't block; ask user)
- Warn if not on base branch (worktree will fork from origin/BASE, not from current state)

## Flow

### 1. Normalize names

- **Slug**: lowercase ASCII kebab-case from description, ≤ 40 chars
  - "Add OAuth for GitHub login" → `add-oauth-github-login`
- **Branch**: `<type>/<slug>` (e.g. `feat/add-oauth-github-login`)
- **Worktree path**: `../<REPO_NAME>-<slug>/`
- If worktree path already exists, append `-2`, `-3`, ...

### 2. Produce plan (3-6 bullets)

- Scope **IN**: what this task does
- Scope **OUT**: explicitly what it does NOT do
- Best-guess affected files / directories
- Verification approach

### 3. Produce done criteria (3-5 checkboxes)

Each item must be **verifiable**:

```markdown
- [ ] Functionality: <specific condition>
- [ ] Verification: <command to run / manual check>
- [ ] Scope: no changes to <areas out of scope>
```

Tailor to quality target:
- `prototype`: functionality + manual verify only
- `commercial`: + basic tests where relevant
- `production`: + tests + type check + lint + no regressions

### 4. Create worktree

```bash
git fetch origin "$BASE_BRANCH"
git worktree add "../${REPO_NAME}-${SLUG}" -b "${TYPE}/${SLUG}" "origin/${BASE_BRANCH}"
cd "../${REPO_NAME}-${SLUG}"
```

### 5. Write `.task.md` in worktree root

```markdown
# Task: <description>

- **Type**: <type>
- **Branch**: <type>/<slug>
- **Base**: <base-branch>
- **Quality**: <prototype|commercial|production>
- **Started**: <YYYY-MM-DD HH:MM>  (use `date "+%Y-%m-%d %H:%M"`)

## Plan
- Scope IN: ...
- Scope OUT: ...
- Affected: ...
- Verify: ...

## Done Criteria
- [ ] ...
- [ ] ...

## Notes
<!-- agent/user append progress, decisions, surprises below -->
```

### 6. Gitignore `.task.md` locally (per-worktree, not committed)

```bash
echo ".task.md" >> .git/info/exclude
```

(Using `.git/info/exclude` instead of `.gitignore` because this preference is local, not repo-wide.)

### 7. Report to user

- Worktree path (absolute)
- Branch name
- Base branch + commit SHA forked from (`git rev-parse --short HEAD`)
- Note that `.task.md` was created and will be used by `finish-task`
- Remind: subsequent work happens **in this worktree**, not the original

## Error handling

- No git repo / no origin → **delegate to `init-repo`** (see Pre-flight step 0)
- No base branch detected even after init-repo: ask user for base branch name
- Dirty working tree: ask "proceed anyway? (worktree forks from origin, not your dirty state)"
- Worktree path exists: auto-append `-2`, `-3`, ... up to 9; beyond that, ask user
- Branch name collision: append `-2` etc; or ask

## Do NOT

- Silently overwrite an existing worktree
- Fork from local HEAD when local is dirty (always fork from `origin/<base>`)
- Commit `.task.md` into the repo
- Skip producing a plan / done criteria (these are required for finish-task)
- Do any actual coding in this skill — only setup
