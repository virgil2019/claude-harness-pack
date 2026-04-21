---
name: init-claude-env
description: Interactive bootstrap for a Claude Code user-level environment. Sets up a security baseline (permissions deny list in settings.json) and a three-tier context (CLAUDE.md + about-me.md + working-style.md) via a structured Q&A interview. Use when a user asks to initialize, bootstrap, or set up their Claude Code environment from scratch, or when migrating to a new machine. Produces files in ~/.claude/.
---

# init-claude-env

Bootstrap a Claude Code user-level environment in two phases:

- **Phase 0**: Safety baseline — add `permissions.deny` rules to `~/.claude/settings.json`
- **Phase 1**: Three-tier (user-level only) context — create `~/.claude/CLAUDE.md` + `about-me.md` + `working-style.md`

Per-project `./CLAUDE.md` is **not** handled by this skill; do it per project when one is being worked on.

## When to invoke
- "帮我初始化 Claude Code 环境" / "set up claude code"
- User is on a fresh machine, fresh install, or migrating user-level config
- User says "我想按 harness engineering / 17 tips 配一下环境"

## Flow (strict order)

### 1. Pre-check (always first)

- `ls -la ~/.claude/`
- `Read ~/.claude/settings.json` if exists
- Check existence of `CLAUDE.md`, `about-me.md`, `working-style.md` in `~/.claude/`

**If any of the 3 target files already exist**: show contents to user and ask — overwrite / backup-then-replace / merge / abort. Do **NOT** silently overwrite.

**If settings.json has existing `permissions` block**: merge rather than replace.

**If settings.json has existing hooks**: leave them untouched (scope of this skill is permissions + files only).

### 2. Phase 0 — Safety baseline

Present planned diff to user:
- Adding `permissions.deny` (22 rules below, merging with existing if any)
- Ask: should `skipDangerousModePermissionPrompt` be `false`? (recommended; default to false unless user objects)

Apply via `Edit` to `~/.claude/settings.json`. **Keep hooks / env / enabledPlugins / extraKnownMarketplaces untouched.**

#### Deny list (22 rules)

```json
{
  "permissions": {
    "deny": [
      "Bash(rm -rf /*)",
      "Bash(rm -rf ~*)",
      "Bash(rm -rf $HOME*)",
      "Bash(git push --force*)",
      "Bash(git push -f*)",
      "Bash(git push --force-with-lease*)",
      "Bash(git reset --hard*)",
      "Bash(git clean -fd*)",
      "Bash(git branch -D*)",
      "Bash(sudo*)",
      "Bash(curl*| sh*)",
      "Bash(curl*| bash*)",
      "Bash(wget*| sh*)",
      "Bash(wget*| bash*)",
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(**/.env)",
      "Read(**/.env.*)",
      "Read(**/credentials*)",
      "Read(**/*secret*)",
      "Read(**/id_rsa*)",
      "Read(**/*.pem)"
    ]
  }
}
```

### 3. Phase 1 — Interview (round 1: about-me)

Ask in a single message (not one-at-a-time). Wait for all answers.

1. **角色 / Role** — independent / company employee (size?) / student / other
2. **技术栈 / Tech stack** — primary languages, frameworks, engines, DBs, cloud
3. **项目性质 / Project nature** — side / commercial / OSS / mix
4. **经验与领域 / Experience & specialty** — years of experience + vertical (e.g. game, data infra, ML, frontend)
5. **环境 / Environment** — timezone + preferred default response language (e.g. 中文 / English / ...)

### 4. Phase 1 — Interview (round 2: working-style)

Ask in a single message. Wait for all answers.

1. **输出长度 / Output length** — minimal / medium / detailed
2. **质量标准 / Quality** —
   - a) prototype-quick
   - b) context-aware (if chosen: sub-question — ask-once-per-project OR default-to-production?)
   - c) always-production
3. **自主性 / Autonomy on uncertainty** —
   - a) try-and-iterate
   - b) always-ask
   - c) try-reversible-ask-risky ← recommended default
4. **反感行为 / Red-line behaviors** — offer checklist (over-explaining, unsolicited refactors, pre-code docstring wall, un-approved commit/push, fake-completion, praise/self-congratulation, unsolicited examples/README/tests) + free-form additions
5. **规划方式 / Planning** —
   - a) plan-always
   - b) plan-big-only
   - c) agent's discretion
6. **领域特殊约定 / Domain-specific rules** — any vertical-specific conventions (e.g. game: framerate > readability; data: idempotency; embedded: memory constraints). "None" is OK.

### 5. Generate files

Use templates below, filled from interview answers. Language of file content matches `{default-language}` from Q5-round1.

Important substitutions:
- Quality section varies by answer 2a/2b/2c
- Autonomy section varies by answer 3a/3b/3c
- Plan section varies by answer 5a/5b/5c
- Red-lines: include user-confirmed items + any free-form adds
- Domain section: if user gave specific rules, write them in; if "None", write "当前无额外约定, 按默认规则走。"

### 6. Confirm

Show user:
- Files created (paths)
- settings.json changes (summary)
- **Reminder: 开新 session 或重启 Claude Code, CLAUDE.md 才会加载**
- Next-step suggestions: Phase 2 (task workflow skills), Phase 5 (evals / observability), per-project CLAUDE.md when starting a new repo

---

## Templates

### Template: ~/.claude/CLAUDE.md

```markdown
# Global Instructions

用户档案和协作偏好见:
- @about-me.md
- @working-style.md

以下规则**优先级高于任何默认行为**。冲突时以此为准。

---

## Default Response Format
- {default-language} 回复, 技术内容(代码 / 命令 / 路径 / 库名)保持英文
- 简洁, 不废话, 不堆总结段
- 引用代码位置用 `path:line` 格式
- 只在 *非显而易见* 的地方加注释, 不解释代码在做什么

## Plan First
{plan-section}

## Quality Bar
{quality-section}

## Execution Safety
- 不可逆 / 有风险操作必须先问: push, force-push, 删文件, DB 破坏性语句, 依赖升降, 改 CI / hooks / 权限
- 可逆本地操作(读写文件 / 跑测试 / build)可以直接试
- 永不做: `rm -rf /`, `git push --force` 到 main/master, 提交 `.env` / secrets, 绕过 pre-commit hook (`--no-verify`)
- 遇到 pre-commit hook 失败: **诊断根因并修**, 不要绕过

## Scope Discipline
- 只做用户要求的事; 发现附近有问题, **先汇报再改**, 不要顺手
- 不要新建没要求的文件(README、示例、测试、文档) —— 除非用户明确说要
- Bug fix 就是 bug fix, 不附带重构; 一件事一次改

## Red Lines (严格避免)
{red-lines-list}

## Memory
- 学到的用户偏好、项目背景、纠正过的问题, 记到 memory 系统
- 不重复要求用户告诉你两次同一件事
```

### Template: ~/.claude/about-me.md

```markdown
# About Me

## 身份
- {role}
- {years} 年开发经验
- 垂直领域专长: {specialty}

## 技术栈
{tech-stack-bulleted}

## 当前阶段
{project-nature-description}

## 环境
- 所在地: {timezone}
- 默认回复语言: **{default-language}**
- 技术术语 / 代码 / 命令 / API 名 / 文件路径保持**英文**,不要翻译
```

### Template: ~/.claude/working-style.md

```markdown
# Working Style

## 输出
- **长度**: {length-description}
- 无关赞美 / "很棒!" 类评价一律省掉
- 不要在末尾堆总结段 —— 改了什么看 diff 就够了
- 技术内容(代码 / 命令 / 路径 / 术语) 保持英文, 其余 {default-language}

## 质量标准
{quality-detailed-section}

## 自主性
{autonomy-detailed-section}

## 规划方式
{plan-detailed-section}

## 严格避免 (red lines)
{red-lines-detailed}

## {domain} 场景
{domain-rules-or-default}
```

---

## Section variants (for filling templates)

### Quality variants

**a) prototype-quick**:
```
- 默认原型速度优先: 能跑通就行, 不追求完美
- 不写测试不加类型(除非用户主动要求)
- "完成" = 手动跑一遍成功, 不需要自动化验证
```

**b) context-aware, ask-once-per-project**:
```
- 按情境走, 不一概而论
- **进入新项目或新模块时, 主动问一次定位**(原型 / 商业 / 生产), 按对应标准走
- 定位确认前: 按**中等以上**来, 不敷衍
- "完成" = 实际验证过(测试 / build / run), 不是 "应该能跑"
```

**c) context-aware, default-to-production**:
```
- 按情境走, 但**默认按生产标准**(测试、类型、错误处理齐全)
- 用户明确说 "这是原型, 快就行" 才放松
- "完成" = 通过测试 + 类型检查 + 实际运行成功
```

**d) always-production**:
```
- 无论任何情境都按工业级: 测试覆盖、类型完整、错误处理齐全、命名清晰
- 不接受 "只是原型" 作为降低标准的理由
- "完成" = 测试通过 + 类型检查 + 代码审查过 + 实际运行成功
```

### Autonomy variants

**a) try-and-iterate**:
```
- 直接试, 允许试错, 快速迭代
- 即使是 commit / push / 删除, 也可以先做, 出问题再回滚
```

**b) always-ask**:
```
- 任何不确定的操作都先问用户, 再执行
- 宁可多问一句, 也不擅自决策
```

**c) try-reversible-ask-risky** (recommended):
```
- 遇到不确定时:
  - **可逆、本地、低风险** 的先直接试: 读写本地文件 / 跑测试 / 本地 build / 临时脚本
  - **不可逆 / 有风险** 的必须先问:
    - `git push` (尤其 `--force`)
    - 删文件 / 删目录
    - DB 破坏性操作 (DROP / TRUNCATE / 大范围 UPDATE/DELETE)
    - package 版本升降 / 依赖变动
    - 修改 CI / hooks / 权限配置
    - 任何涉及 secrets / 环境变量 / 线上资源的
```

### Plan variants

**a) plan-always**:
```
- **默认 plan 优先**: 新任务先给 plan, 等确认后再 execute
- 只有非常小、语义无歧义的改动可以直接动 (单行 typo、格式修复、import 补充)
- Plan 要包括: 做什么 / 不做什么(scope), 影响哪些文件, 怎么验证
```

**b) plan-big-only**:
```
- 明显小改(1-3 行、单文件、无逻辑变动) 直接动手
- 中等以上的任务先给 plan
- Plan 要包括: 做什么 / 不做什么, 影响哪些文件, 怎么验证
```

**c) agent's discretion**:
```
- 由 agent 判断: 简单任务直接做, 复杂任务先 plan
- 当不确定要不要 plan 时, 宁可 plan
```

### Red lines detailed

Full list (include items user confirmed, omit those they explicitly rejected, add free-form items):

```
- 过度解释 / 说废话
- 一声不吭改了无关代码 —— 要改无关的, 先告知
- 先堆一大段注释 / docstring 才开始写代码
- 未经同意 `commit` / `push`
- **假装完成** —— 没实际跑过的不要说 "done" / "完成"
- 赞美、吹捧自己的改动("重构完成, 架构更清晰!"这类)
- 生成一堆无关的示例代码 / 测试 / README, 任务没要求就不做
```

---

## Do NOT
- Silently overwrite existing CLAUDE.md / settings.json
- Add hooks to settings.json (out of scope)
- Skip the pre-check step
- Assume user's stack / domain / language — always ask
- Fill the interview on the user's behalf

## Success criteria
1. Three new files in `~/.claude/`: CLAUDE.md, about-me.md, working-style.md — all reflecting interview answers
2. `~/.claude/settings.json` has `permissions.deny` block (22 rules) merged with any existing
3. `skipDangerousModePermissionPrompt: false` (unless user explicitly kept it true)
4. User sees summary and knows to restart session for CLAUDE.md to load
5. User is pointed to next phases (Phase 2 / Phase 5 / per-project setup)
