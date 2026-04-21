---
name: take-note
description: Appends a note to ~/memo.md with a timestamp header. Invoke when the user says "记笔记", "记录下来", "记一下", "存一下", "take a note", "save this as a memo", or similar note-taking intents. Each note gets a date+time header and is appended (never overwrites existing content).
---

# take-note

Append a note to `~/memo.md`. Timestamped, scannable, append-only.

## When to invoke

- User says 记笔记 / 记录下来 / 记一下 / 存一下 / save to memo / take a note
- User wants to capture an idea, TODO, reference, decision, or deferred plan

## Flow

### 1. Determine the content

- **Explicit**: user provides the note text directly → use it verbatim
- **Implicit** (user says "记笔记" without saying what): treat the **current topic under discussion** as the content; **show a draft and confirm** before writing
- **Unclear**: ask "要记什么？"

### 2. Append to `~/memo.md`

- If file doesn't exist, create with `# Memo\n\n` at the top
- Get current time: prefer `date "+%Y-%m-%d %H:%M"` via Bash; fall back to today's date from context
- Timezone: use the one in `~/.claude/about-me.md` if available
- Append a new section in this format:

```markdown
## YYYY-MM-DD HH:MM · <topic tag>

<content>

---
```

- `<topic tag>` = 2-5 word label (e.g. "云端同步方案", "bug: X broke on Y", "decision: 选 Postgres over Mongo") — makes the file scannable later

### 3. Confirm

Tell user: `已记录到 ~/memo.md` (optionally mention line range).

## Formatting rules

- Preserve user's original wording when they give explicit text
- Multi-part content → bullets or numbered lists
- Code / commands → preserve in code blocks
- Headings inside the note → use `###` or lower (never `#` or `##` which are reserved for the file structure)

## Append-only discipline

- **Never overwrite** existing notes
- **Never delete** — user manages their own pruning
- **Never reorder** existing entries
- Only add to the END of the file

## Out of scope (do NOT do in this skill)

- Summarize or search existing notes (separate task; use Read / Grep)
- Edit past entries
- Sync memo.md anywhere
