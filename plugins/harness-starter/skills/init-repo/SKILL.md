---
name: init-repo
description: Initializes a new git repository from the current directory. Interactively asks for repo name / GitHub account (by SSH alias) / visibility / description, runs git init, creates an initial commit, creates the GitHub repo via gh using the correct account, and sets the remote URL using the SSH alias from ~/.claude/gh-accounts.yml. Invoke when the user says "init repo", "建仓库", "新建仓库", "建项目", "开新项目", "setup repo", or automatically by start-task when the current dir is not a git repo or has no origin remote.
---

# init-repo

Bootstrap a brand-new GitHub repo from the current local directory. Handles "empty dir → fully working git + GitHub repo" in one skill invocation.

## When to invoke

- User says: "init repo" / "建仓库" / "新建仓库" / "建项目" / "开新项目" / "setup repo"
- Automatically chained by `start-task` when current dir has no `.git` or no `origin` remote
- Intent: turn a local directory (empty or with files) into a full git + GitHub repo, ready for task work

## Pre-flight

Determine current state:

```bash
IS_REPO=$(git rev-parse --is-inside-work-tree 2>/dev/null && echo yes || echo no)
HAS_ORIGIN=$(git remote 2>/dev/null | grep -qx origin && echo yes || echo no)
```

Decide scenario:
- `IS_REPO=no` → full bootstrap (git init + initial commit + GitHub create + remote + push)
- `IS_REPO=yes, HAS_ORIGIN=no` → partial (skip git init; ensure there's a commit; GitHub create + remote + push)
- `IS_REPO=yes, HAS_ORIGIN=yes` → **nothing to do** — confirm with user and stop

## Flow

### 1. Interview (4 questions in one message)

Ask:

1. **Repo name** — default: `$(basename "$(pwd)")`, user can override
2. **GitHub account** — read `~/.claude/gh-accounts.yml`, show `alias → username` mapping, let user pick by alias. If the file is missing or empty, ask user to fill it first
3. **Visibility** — `public` or `private`. **Default: `private`** (safer)
4. **Description** (optional, one short line)

Wait for all answers before proceeding. Confirm back the parsed values before acting.

### 2. git init (if needed)

```bash
[ "$IS_REPO" = "no" ] && git init -b main
```

### 3. Ensure initial commit

```bash
if ! git rev-parse HEAD >/dev/null 2>&1; then
  # No commits yet: create one
  if [ -z "$(ls -A)" ]; then
    printf "# %s\n" "$REPO_NAME" > README.md
    [ -n "$DESCRIPTION" ] && printf "\n%s\n" "$DESCRIPTION" >> README.md
  fi
  git add -A
  git -c user.name="$GIT_USER_NAME" -c user.email="$GIT_USER_EMAIL" \
      commit -m "Initial commit"
fi
```

Note: use `GIT_USER_NAME` / `GIT_USER_EMAIL` values associated with the chosen GitHub account (e.g., `${GH_USER}@users.noreply.github.com`). If global git config is already set, you can rely on that instead.

### 4. Create GitHub repo via gh (race-safe)

```bash
# NOTE: use GH_USER (not USERNAME). zsh treats USERNAME as a special parameter
# bound to the system login name — assignment is silently ignored. See the
# same note in finish-task's step 5 for the full list of zsh specials to avoid.
GH_USER=$(grep -E "^${SSH_ALIAS}:" ~/.claude/gh-accounts.yml | head -1 | sed 's/^[^:]*: *//' | tr -d '"'"'"' ')
[ -n "$GH_USER" ] || { echo "No mapping for $SSH_ALIAS in ~/.claude/gh-accounts.yml"; exit 1; }

TOKEN=$(gh auth token -u "$GH_USER" 2>/dev/null)
[ -n "$TOKEN" ] || { echo "gh not authed for $GH_USER. Run: gh auth login"; exit 1; }

VISIBILITY_FLAG="--private"
[ "$VISIBILITY" = "public" ] && VISIBILITY_FLAG="--public"

GH_TOKEN="$TOKEN" gh repo create "${GH_USER}/${REPO_NAME}" \
  ${VISIBILITY_FLAG} \
  ${DESCRIPTION:+--description "$DESCRIPTION"}
```

Use `GH_TOKEN=... gh` **inline** (not `gh auth switch`). Stateless, safe for parallel use.

### 5. Set remote with SSH alias

```bash
git remote add origin "git@${SSH_ALIAS}:${GH_USER}/${REPO_NAME}.git"
```

This ensures all future `git push / pull` on this repo route through the correct SSH key.

### 6. Push

```bash
git push -u origin main
```

If push fails: stop, show output, ask user. Never force-push.

### 7. Report

Summarize:
- Repo URL: `https://github.com/${GH_USER}/${REPO_NAME}`
- Visibility + description
- Remote URL (confirms SSH alias)
- Local branch (main) + commit SHA
- Suggested next step: run `start-task` to branch off and begin the first feature

## Error handling

| Issue | Response |
|---|---|
| `gh-accounts.yml` missing or empty | Ask user to add alias→username mapping first; show a template snippet |
| Chosen alias has no mapping | Ask user to add it; abort |
| gh not authed for that username | Instruct `gh auth login --git-protocol ssh --web`, halt |
| GitHub repo name already exists | Show error, ask for different name |
| Push rejected | Show output, ask user (never force-push) |
| Origin already exists pointing elsewhere | Warn, ask user to confirm / rename |

## Do NOT

- Create the GitHub repo before confirming user's answers
- Default to `--public` (always default to `--private` for safety)
- Use `gh auth switch` — always `GH_TOKEN=$(gh auth token -u X) gh ...` inline
- Force-push
- Overwrite an existing remote silently
- Proceed if gh-accounts.yml is missing — ask user to populate first

## Relationship to `start-task`

`start-task` should call `init-repo` when its pre-flight detects no `.git` or no `origin`. After `init-repo` completes, `start-task` resumes its normal flow (branch, worktree, plan, `.task.md`).
