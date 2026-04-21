---
name: codify-feedback
description: Captures a user feedback/correction and codifies it into a durable rule (~/.claude/working-style.md, ~/.claude/CLAUDE.md, project ./CLAUDE.md, or the agent memory system). Invoke when the user explicitly says "记下来", "codify this", "下次别 X", "这是规则", "以后都 X", "永远不要 X", or similar codify intents. **Proactively invoke when detecting ≥2 similar corrections within the last 5 turns** — ask "要把这个 codify 吗？" before proceeding. Do NOT invoke for clear one-off stylistic preferences, or for lessons already captured in this session.
---

# codify-feedback

Convert a correction/feedback moment into a durable rule at the right scope. Short-cycle mechanism of the evolution loop.

## When to invoke

### Explicit (user asks):
- 记下来 / 记一下 / 记成规则 / codify this / save this rule
- 下次别 X / 下次要 Y / 以后都 X / 永远不要 X
- 这是规则 / this is a rule

### Proactive (agent detects):
- User corrected me ≥2 times on similar behavior in last 5 turns → ask "要 codify 吗？"
- Do NOT be annoying: max 1 proactive prompt per session per pattern

### Skip:
- One-off style preferences ("这次你写短点" — not a rule)
- Already captured this session (check context first)

## Flow

### 1. Clarify the rule

Confirm the exact rule in one sentence:
> "让我确认: 规则是 <one-sentence>, 对吗？"

If user's original phrasing is clear, paraphrase back.

### 2. Determine scope

Ask (or infer) which scope:

| Scope | Where | When |
|---|---|---|
| **Global** (对所有项目都适用) | `~/.claude/working-style.md` | 长期通用偏好 / red line |
| **Core global** (行为级别, 优先级高) | `~/.claude/CLAUDE.md` | 涉及 plan / safety / scope discipline |
| **Project** (只对当前 repo) | `./CLAUDE.md` (项目根) | 仅与此项目的技术栈 / 约定相关 |
| **Memory** (跨 session 上下文 / 非规则类事实) | agent memory system | 不是规则但要记得: 决策背景, 外部系统位置, 偏好 |

Default picks:
- Behavioral rule → working-style.md
- Safety / scope / safety-adjacent → CLAUDE.md
- Project-specific convention → project CLAUDE.md
- Context / reference / episodic → memory

### 3. Determine section

Within the chosen file, pick or create the right section:
- Red line → "严格避免 (red lines)"
- Output preference → "输出"
- Quality bar → "质量标准"
- Planning → "规划方式"
- Autonomy → "自主性"
- New concept → create new `## Section`

For memory:
- user / feedback / project / reference (per agent memory type rules)

### 4. Generate the edit

Show the user:
- Target file + target section
- Exact diff (before → after) OR full new memory entry
- Ask for approval

### 5. Apply

- If file: `Edit` it
- If memory: write the memory file + update MEMORY.md index

### 6. Confirm

- Path + line(s) changed
- Remind: global rules from `~/.claude/` take effect **next session**; project CLAUDE.md takes effect immediately in current session in that project

## Anti-patterns to avoid

- Writing verbose, vague rules. Prefer **imperative + specific**:
  - ❌ "Be careful about scope"
  - ✅ "修改无关代码前先汇报, 不要顺手改"
- Duplicating an existing rule. Check for overlap first.
- Adding to CLAUDE.md when working-style.md is the right place (CLAUDE.md is for high-priority behavioral rules, working-style for preferences/red-lines).
- Codifying a one-off — user may regret it later. If unsure, ask "这是一次性还是以后都要这样？"

## Do NOT

- Apply edits without showing the user first
- Codify vague feelings ("做得不够好") without concrete rule
- Overwrite existing rules silently (merge / append, or ask)
- Trigger more than once on the same pattern within a session
