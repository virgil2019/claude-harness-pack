---
name: take-note
description: Captures a note to the best available backend — prefers an MCP note-taking tool (obsidian, notion, logseq, etc.) if one is connected, otherwise appends to ~/memo.md as a timestamped entry. Invoke when the user says "记笔记", "记录下来", "记一下", "存一下", "take a note", "save this as a memo", or similar note-taking intents. Capability-aware: adapts to whatever knowledge base is available; take-note itself has no hardcoded backend dependency.
---

# take-note

Capture a note using the **best available backend**. Capability-aware: falls back to `~/memo.md` only when no MCP note backend is connected.

## When to invoke

- User says 记笔记 / 记录下来 / 记一下 / 存一下 / save to memo / take a note
- User wants to capture an idea, TODO, reference, decision, or deferred plan

## Flow

### 1. Determine the content

- **Explicit**: user provides the note text → use verbatim
- **Implicit** (user says "记笔记" without specifying): treat the **current topic under discussion** as the content; **show a draft and confirm** before writing
- **Unclear**: ask "要记什么？"

### 2. Choose the backend (capability detection)

Scan available tools for a **note-taking backend**. A tool qualifies if its name or description indicates note / journal / memo / knowledge-base / vault intent. Typical examples (but the skill doesn't depend on any specific one):

- Obsidian MCP (create note, append to note, write to vault)
- Notion MCP (create page, append to page)
- Logseq / Roam / other knowledge-base MCPs

**Routing rules**:
- If **one** MCP note backend available → use it
- If **multiple** available → ask user once per session which to use (remember choice within session)
- If **none** → fall back to file backend (`~/memo.md`)
- If user explicitly says "记到 memo.md" / "记到文件" → force file backend
- If user explicitly says "记到 obsidian" / "记到 <specific backend>" → use that one directly

### 3. Write to the chosen backend

Get current time: `date "+%Y-%m-%d %H:%M"`. Timezone from `~/.claude/about-me.md` if available.

**File backend (`~/memo.md`)** — original behavior:
- If file doesn't exist, create with `# Memo\n\n` at top
- Append:

  ```markdown
  ## YYYY-MM-DD HH:MM · <topic tag>

  <content>

  ---
  ```

- `<topic tag>` = 2-5 word label (e.g. "云端同步方案", "bug: X broke on Y")

**MCP note backend**:
- Use the MCP tool's idiomatic form for appending / creating a note
- Preferred note title: `YYYY-MM-DD · <topic tag>`
- Put full content in the note body, preserving code blocks and markdown
- If the MCP supports tags / folders, use a default tag like `inbox` or `quick-note` (configurable in user's CLAUDE.md / working-style.md if they want to override)

### 4. Confirm

Tell user where the note landed:

- File: `已记录到 ~/memo.md` (mention line range if helpful)
- MCP: `已记录到 <backend name>: <note location / URL>`

## Formatting rules

- Preserve user's original wording when they give explicit text
- Multi-part content → bullets or numbered lists
- Code / commands → preserve in code blocks
- File backend: headings inside the note use `###` or lower (never `#` / `##` — reserved for file structure)

## Append-only discipline

- **Never overwrite** existing notes (file or MCP)
- **Never delete** — user manages their own pruning
- **Never reorder** existing entries
- Only add new entries

## Out of scope

- Summarize or search existing notes (use Read / Grep / MCP search tools directly)
- Edit past entries
- Migrate notes between backends (one-time ops user can script)

## Design note

This skill is **capability-aware**, not backend-locked. It adapts to what's present:
- No MCP note tool → behaves like a simple append-to-file
- Any MCP note tool → prefers it (assumes user wanted that tool to be primary)

Users who want to force file-only can explicitly say "记到 memo.md", or add a rule to their `working-style.md`:

```markdown
## Notes
- Always use ~/memo.md for take-note (do not route to MCP backends)
```
