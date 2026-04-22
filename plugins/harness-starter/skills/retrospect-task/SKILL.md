---
name: retrospect-task
description: Comprehensive end-of-task retrospective. Reads .task.md + session context, walks through 4 structured questions (corrections, friction, reusable patterns, architecture/rule updates), and for each insight offers codification options. Invoke at task close (typically chained from finish-task) or when the user says "回顾一下", "retrospect", "task 复盘", "task 总结". Distinct from checkpoint (mid-task pulse): this is full, backward-looking, and one-shot.
---

# retrospect-task

End-of-task comprehensive review. The last step of the evolution loop.

## When to invoke

- After `finish-task` opens a PR (user may be prompted to run this)
- User says: "回顾一下" / "retrospect" / "task 复盘" / "总结下这个 task"
- A long / difficult task is ending and final codifications should happen before memory fades

## Pre-flight — locate task file (both start-task modes)

Same logic as `finish-task`:

```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
COMMON_DIR=$(git rev-parse --git-common-dir)
PRIMARY_ROOT=$(cd "$COMMON_DIR/.." && pwd)
BRANCH=$(git rev-parse --abbrev-ref HEAD)
SLUG="${BRANCH#*/}"

TASK_FILE=""
[ -f "$REPO_ROOT/.task.md" ] && TASK_FILE="$REPO_ROOT/.task.md"                    # Mode B
[ -z "$TASK_FILE" ] && [ -f "$PRIMARY_ROOT/.tasks/$SLUG.md" ] && \
  TASK_FILE="$PRIMARY_ROOT/.tasks/$SLUG.md"                                         # Mode A
```

- If no task file at either path, ask user to point to the right file (e.g., `.tasks/<name>.md` if slug differs from branch tail)
- If `checkpoint` was run during the task, read its entries in the Notes section to avoid re-covering already-codified lessons
- Use absolute `$TASK_FILE` in every Read/Write/Edit call below — never relative

## Flow

### 1. Summarize the task (brief, < 5 bullets)

Based on `.task.md` + session:
- Goal (original task description)
- Final scope (what was actually changed)
- Quality target vs actual quality achieved
- Duration (if timestamps available)
- Outcome: PR URL or failure mode

### 2. Four structured questions

Ask these in one message, wait for answers:

1. **纠正 / Corrections**
   - 过程中有哪些我做错然后被你纠正的? 哪些是值得固化的?
   (agent should pre-scan the session and list candidates as a hint, not force user to recall)

2. **摩擦 / Friction**
   - 有没有某个步骤特别慢 / 反复 / 尴尬? 根因是什么?

3. **可复用模式 / Reusable patterns**
   - 过程中有没有某个动作重复 ≥3 次? 值不值得抽成 skill?

4. **架构 / 规则 / 文档**
   - 有没有值得写进**项目 CLAUDE.md** (该项目的约定)? 或 `working-style.md` (全局规则)? 或 memory (跨 session 事实)?

### 3. Consolidate into action list

From user's answers, produce a concrete list:

```
Action list:
1. [rule] → working-style.md (新一条 red line: ...)
2. [rule] → 项目 CLAUDE.md (该项目用 X, 不用 Y)
3. [skill] → 新建 `<skill-name>` 用于 ...
4. [memory] → project memory: 该项目的 migration 脚本放在 ...
5. [skip] → 小 bug, 一次性, 不固化
```

### 4. Execute (delegate per-item)

For each item:
- **rule** → invoke `codify-feedback` flow
- **skill** → create skeleton at `~/.claude/skills/<name>/SKILL.md` (ask for name + description first)
- **memory** → write memory file + update MEMORY.md
- **skip** → do nothing

Show user diff / content before applying each. Batch approval OK ("全部执行" → proceed all).

### 5. Append to the task file (absolute path `$TASK_FILE`)

```markdown
### Retrospective @ <YYYY-MM-DD HH:MM>
- Summary: <1 line>
- Codified: <list>
- Deferred / skipped: <list>
- PR: <URL>
```

### 6. Close

Tell user:
- Codifications applied (paths)
- Reminder: global rules take effect next session; project CLAUDE.md + memory take effect now
- Suggest: `git worktree remove` after PR merges (if applicable)

## Anti-patterns

- **Don't re-surface lessons already codified via checkpoint or codify-feedback in this session** — read `.task.md` Notes first
- Don't force codification — "skip" is a valid outcome for any item
- Don't write long narrative retrospectives — keep it actionable
- Don't codify everything as a global rule — respect the scope decision

## Do NOT

- Modify code / branches / PRs
- Auto-merge the PR
- Delete `.task.md` (user keeps / trashes manually)
- Skip the `.task.md` append step (future retrospections will use it)
