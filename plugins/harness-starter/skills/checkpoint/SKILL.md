---
name: checkpoint
description: Mid-task lightweight pulse / review. Walks through recent interactions to extract potential lessons before the task ends — so the rest of the task can benefit from them. Invoke when the user says "checkpoint 一下", "中间盘点", "先停一下看看", "做个 checkpoint", or similar mid-task review intents. Produces a short progress summary + a shortlist of candidate codifications, then delegates each to codify-feedback. Distinct from retrospect-task (which is end-of-task and comprehensive). Good for long-running tasks.
---

# checkpoint

Mid-task pulse. Scan recent interactions for lessons; feed them back **before** the task ends.

## When to invoke

- User says: "checkpoint 一下" / "中间盘点" / "先停一下看看" / "复盘下到现在"
- Typical usage: long tasks, session approaching context limits, after a rough stretch of corrections
- NOT for: short tasks (< 20 turns), read-only exploration, task already ending (use retrospect-task instead)

## Flow

### 1. Progress snapshot (< 30s, don't ramble)

Summarize status relative to `.task.md`:
- Done criteria: X / Y completed
- Currently working on: <current sub-task>
- Blocked on: <if any>

### 2. Scan recent turns for friction

Review last ~20 turns (or since last checkpoint, if any). Look for:

- **Corrections**: points where user said "no" / "not like that" / "stop"
- **Reversals**: I did X, then undid it based on user feedback
- **Repetition**: same mistake / same question multiple times
- **Latency**: steps that took many tool calls or back-and-forth
- **Surprises**: things neither of us expected

Don't list everything; pick the top 3-5 signals.

### 3. Present candidate lessons

Show as short bullets:
```
候选 lessons:
1. <pattern> —— 建议 codify 成 <rule summary>, scope: <working-style | project CLAUDE.md | memory>
2. ...
```

Ask user: "以上哪些值得 codify？"

### 4. Delegate each accepted → `codify-feedback`

For each accepted lesson, invoke codify-feedback flow (or inline do it).

### 5. Update `.task.md`

Append to Notes section:
```markdown
### Checkpoint @ <YYYY-MM-DD HH:MM>
- Progress: <X/Y done criteria>
- Codified: <rule 1>, <rule 2>
- Deferred (user declined): <rule 3>
- Continue with: <next step>
```

### 6. Offer to continue

"继续刚才的任务?" / 自然接回主线。

## Anti-patterns

- Don't turn checkpoint into a big retrospection — that's retrospect-task's job
- Don't re-surface lessons already codified earlier in this session
- Don't force codify — if user says "no these are one-off", accept and move on
- Don't break main task flow for > 5 minutes

## Do NOT

- Run a full retrospective (use retrospect-task for that)
- Modify code or task state
- Edit `.task.md` beyond appending to Notes
