---
name: start-task
description: Bootstraps a coding task. Offers two modes — (A) in-place: use current directory + current or newly-checked-out branch, write `.tasks/<slug>.md` for persistence; (B) worktree: create `.worktrees/<slug>/` on a new branch. Asks the user to pick mode. Invoke when the user says "开始任务", "新任务", "start a task", or when starting any non-trivial code change that should go through a PR. If the current dir is not a git repo or has no origin remote, delegates to init-repo first. **Task type (feat / fix / refactor / ...) is a classification, not a signal that a new repo is needed — pre-flight MUST run the actual git commands and base the delegate-or-not decision on their output, not on task description.**
---

# start-task

Open a new task. Two modes depending on whether you want a separate worktree or not.

## When to invoke

- User says "开始任务" / "start a task" / "新任务" / "做一下 X"
- Any non-trivial code change that will need review via PR
- **Skip** if: read-only exploration, trivial fix (single-line typo, format), or in-place quick patch

## Inputs (3, batch-ask if unclear)

1. **Task description** — one sentence
2. **Task type** — `feat` / `fix` / `refactor` / `docs` / `chore` / `perf` / `test`
3. **Quality target** — `prototype` / `commercial` / `production` (per `working-style.md`)

> ⚠️ **`feat` is a classification, not "new project"**. Working on a feature branch in an existing repo is `feat` — it does NOT imply `init-repo`. See Pre-flight step 0.

## Pre-flight

### 0. Bootstrap check — MUST actually run the git commands

Do NOT guess from task description or directory listing. **Run both commands via the Bash tool**, show their real output to the user, then decide.

```bash
git rev-parse --is-inside-work-tree 2>/dev/null && echo "IS_REPO=yes" || echo "IS_REPO=no"
git remote 2>/dev/null | grep -qx origin && echo "HAS_ORIGIN=yes" || echo "HAS_ORIGIN=no"
```

Decision rule:

| `IS_REPO` | `HAS_ORIGIN` | Action |
|---|---|---|
| yes | yes | **Proceed** to step 0.5 — any branch is OK, including `main` / `dev` / `feat/*` / whatever |
| no | any | Delegate to **init-repo**; message: "当前目录不是 git repo。先 init-repo 把仓库配好再继续。" |
| yes | no | Delegate to **init-repo**; message: "repo 存在但无 origin。先 init-repo 把 GitHub 那头接上。" |

❌ Signals that DO NOT constitute a reason to delegate to init-repo:
- Task type is `feat`
- Task description contains "new" / "feature" / "创建" / "建"
- Directory looks small or recently created
- User didn't explicitly say "I already have a repo"

Only the two git commands' output decides.

### 0.5. Resume detection — surface in-flight tasks

Don't silently create another worktree if an in-flight task already exists:

```bash
git worktree list --porcelain | awk '
  /^worktree / {wt=$2}
  /^branch / {br=$2}
  /^$/ {if (wt && br && wt != primary) print wt "\t" br; wt=""; br=""}
' primary="$(git rev-parse --show-toplevel)"
```

For each secondary worktree, a task is **in flight** if ANY of:
- `.task.md` exists in its root
- staged / uncommitted changes (`git -C <wt> status --porcelain` non-empty)
- commits ahead of `origin/<base>`

Show list; ask **Resume vs New Task**. Resume → abort start-task, tell user to `cd` there. New → continue.

### 0.8. Mode selection

Ask the user (one message):

> 选模式:
>
> **A. 就地模式** — 留在当前目录 `<cwd>` (分支 `<current-branch>`), 只把 plan 和 done criteria 写到 `<repo>/.tasks/<slug>.md` (持久化, 跨任务可回看). 不建 worktree, 不 cd.
>
> **B. 新 worktree 模式** — 建 `<repo>/.worktrees/<slug>/`, 新分支 `<type>/<slug>` (从 `origin/<base>` 分叉), cd 进去. 适合并行多任务或 main 保持干净.

Wait for the user's answer (A or B). Do not guess.

## Common setup (do AFTER mode is chosen, BEFORE the mode-specific flow)

### 1. Repo info

```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
REPO_NAME=$(basename "$REPO_ROOT")

BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
[ -n "$BASE_BRANCH" ] || BASE_BRANCH=$(git branch --list main master | head -1 | tr -d '* ')
```

Note on `REPO_ROOT`: returns the **current** working tree's root. If user ran start-task from inside another worktree, `$REPO_ROOT` is that worktree's root. Mode A + being inside an existing worktree is a sensible chain (nest `.tasks/` inside); Mode B + being inside a worktree means the new worktree nests too — git allows it but prefer running Mode B from the primary working tree.

### 2. Normalize names

- **Slug**: lowercase ASCII kebab-case from the task description, ≤ 40 chars
  - `"Add OAuth for GitHub login"` → `add-oauth-github-login`
- **Branch name (proposed)**: `<type>/<slug>`, e.g. `feat/add-oauth-github-login`

### 3. Produce plan (3–6 bullets)

- **Scope IN**: what this task does
- **Scope OUT**: what it explicitly does NOT
- **Affected**: best-guess files/dirs
- **Verify**: how to confirm it works

### 4. Produce done criteria (3–5 verifiable checkboxes)

```markdown
- [ ] Functionality: <specific condition>
- [ ] Verification: <command / manual check>
- [ ] Scope: no changes outside <declared area>
```

Tailor strictness to quality target:
- `prototype`: functionality + manual verify
- `commercial`: + basic tests where relevant
- `production`: + tests + type check + lint + no regressions

---

## Mode A · 就地模式

### A.1. Branch handling

```bash
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
```

Decide:

- **Protected** (`main` / `master` / `dev` / `develop` / `release/*` / `hotfix/*`):
  - **Strongly recommend** switching to a new branch. Ask:
    > 当前在保护分支 `<branch>`. 要切到 `<type>/<slug>` 再开工吗? (推荐 yes, 否则你在保护分支上直接改代码, finish-task 那边开 PR 也不顺)
  - Yes → `git checkout -b <type>/<slug>`
  - No → warn, proceed (record in `.tasks/<slug>.md` Notes: "working directly on `<branch>`, non-standard")
- **Task-type** (`feat/*` / `fix/*` / `refactor/*` / `chore/*` / `docs/*` / `perf/*` / `test/*`):
  - Use as-is. Tell user: "继续用当前分支 `<branch>`".
  - Adjust slug to match branch tail if user wants (e.g., branch `feat/add-oauth` → slug stays `add-oauth`).
- **Other** (user-defined name):
  - Use it. Advise: "以后建议用 `<type>/<slug>` 命名格式。"

### A.2. Ensure `.tasks/` is gitignored

```bash
GITIGNORE="$REPO_ROOT/.gitignore"
if ! grep -qxF ".tasks/" "$GITIGNORE" 2>/dev/null; then
  echo ".tasks/" >> "$GITIGNORE"
fi
```

### A.3. Write task file via `Write` tool, **absolute path**

Target: `$REPO_ROOT/.tasks/<slug>.md` (compute absolute path first).

```bash
mkdir -p "$REPO_ROOT/.tasks"
```

Then use the **Write** tool with the absolute path (not a relative path):

```
Write(file_path="<absolute: $REPO_ROOT/.tasks/<slug>.md>", content=<task file markdown>)
```

Content format (same as Mode B's `.task.md`):

```markdown
# Task: <description>

- **Mode**: A (in-place)
- **Type**: <type>
- **Branch**: <branch-used>
- **Base**: <base-branch>
- **Quality**: <prototype|commercial|production>
- **Started**: <YYYY-MM-DD HH:MM>

## Plan
- Scope IN: ...
- Scope OUT: ...
- Affected: ...
- Verify: ...

## Done Criteria
- [ ] ...
- [ ] ...

## Notes
<!-- append progress, decisions, surprises here -->
```

### A.4. Report

- **Mode**: A (in-place)
- **Branch**: `<branch-used>`
- **Task file**: `$REPO_ROOT/.tasks/<slug>.md` (absolute)
- **No worktree** — edits happen in the current directory

> ✅ **继续在当前 session 工作**。不需要切目录，也不需要开新 Claude Code session。
> 完成后跑 `/finish-task`；想中途盘点跑 `/checkpoint`；想随时固化规则跑 `/codify-feedback`。

---

## Mode B · 新 worktree 模式

### B.1. Ensure `.worktrees/` is gitignored

```bash
GITIGNORE="$REPO_ROOT/.gitignore"
if ! grep -qxF ".worktrees/" "$GITIGNORE" 2>/dev/null; then
  echo ".worktrees/" >> "$GITIGNORE"
fi
```

### B.2. Create the worktree

```bash
git fetch origin "$BASE_BRANCH"
mkdir -p "$REPO_ROOT/.worktrees"
WORKTREE_PATH="$REPO_ROOT/.worktrees/<slug>"
# if collision, append -2, -3, ...
git worktree add "$WORKTREE_PATH" -b "<type>/<slug>" "origin/$BASE_BRANCH"
cd "$WORKTREE_PATH"
```

### B.3. Write `.task.md` via `Write` tool, **absolute path**

Target: `$WORKTREE_PATH/.task.md`.

```
Write(file_path="<absolute: $WORKTREE_PATH/.task.md>", content=<task file markdown with **Mode**: B>)
```

### B.4. Report

- **Mode**: B (worktree)
- **Worktree path**: `$WORKTREE_PATH`
- **Branch**: `<type>/<slug>` (forked from `origin/<base>@<short-SHA>`)
- **Task file**: `$WORKTREE_PATH/.task.md`

> 🔄 **下一步: 在 worktree 里开新 Claude Code session (强烈推荐)**
>
> 当前 session 的 cwd 心智模型还停留在 `$REPO_ROOT`; Bash 虽然 cd 到 worktree 了, 但 Read/Write/Edit 和 subagents 还按老 cwd 解析路径. 在新 session 里打开 worktree 目录, 一切自动归位.
>
> ```bash
> # 1. 退出当前 session  (Ctrl+D, 或 /exit)
> # 2. 切到 worktree
> cd "$WORKTREE_PATH"
> # 3. 启动新 Claude Code session
> claude
> # 4. 新 session 里说 "继续这个任务" / "resume" — Claude 会读 .task.md 自动接上下文
> ```
>
> ⚠️ **不推荐**留在当前 session 继续干活: cwd 心智错位, 容易出现 "Bash 跑对了但 Write 落错目录" 这类诡异问题. 如果确实要留 (比如任务很小), 记得**所有 Read/Write/Edit 都传绝对路径** `$WORKTREE_PATH/...`.

---

## ⚠️ CWD trap — read this before doing anything after start-task

`cd` only affects the **Bash tool**'s cwd. It does **NOT** affect `Read` / `Write` / `Edit` — those tools resolve paths against the **session's starting cwd**, not the Bash subshell's cwd.

Therefore, after start-task:

- ✅ **Use absolute paths** with Read / Write / Edit (`"/full/path/.task.md"`).
- ❌ Don't use `./.task.md` or `../foo`; they won't point where you expect.
- ✅ Bash commands (`git status`, `ls`, etc.) are OK — Bash retains the new cwd.

Internally, the skill should:
- Compute `REPO_ROOT` and `WORKTREE_PATH` once, early.
- Pass the absolute versions to every file operation afterwards.

## Error handling

| Issue | Response |
|---|---|
| `IS_REPO=no` or `HAS_ORIGIN=no` | Delegate to init-repo |
| On protected branch in Mode A, user refuses to switch | Warn, proceed, note in task file |
| Mode B worktree path exists | Auto-append `-2`, `-3`, ... up to 9; beyond that ask user |
| Mode B branch name collision | Append `-2`, etc., or ask |
| Dirty working tree in Mode B | Ask "proceed anyway? (worktree forks from origin, not your dirty state)" |
| No base branch detectable | Ask user for base branch name |

## Do NOT

- Delegate to init-repo based on task description — only git commands' output decides
- Silently overwrite an existing worktree
- Fork from local HEAD when local is dirty (always `origin/<base>`)
- Commit `.task.md` or `.tasks/` into the repo (they should be gitignored)
- Use relative paths with Read/Write/Edit — always absolute
- Skip producing a plan + done criteria (required for finish-task and retrospect-task)
- Do any actual coding in this skill — only setup
