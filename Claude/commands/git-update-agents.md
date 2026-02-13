# Git Update Agents

Sync all Claude configuration files to the Tamir-Learning repo for version tracking.

## What to sync

Copy these directories from `~/.claude/` to `/Users/tamirc/Projects/Tamir-Learning/Claude/`:
- `agents/` — all agent `.md` files
- `commands/` — all command `.md` files
- `skills/` — all skill `.md` files
- `settings.json` — user-level settings
- `settings.local.json` — local settings overrides
- `projects/-Users-tamirc-IdeaProjects-scheduler/memory/MEMORY.md` — copy as `memory/scheduler/MEMORY.md`

## What NOT to sync
- `.credentials.json`, `history.jsonl`, `stats-cache.json` — sensitive/ephemeral
- `cache/`, `debug/`, `downloads/`, `file-history/`, `ide/`, `paste-cache/`, `plans/`, `plugins/`, `session-env/`, `shell-snapshots/`, `statsig/`, `tasks/`, `telemetry/`, `todos/` — runtime data

## Steps

1. Run `rsync` to copy the files (preserving directory structure, deleting removed files)
2. `git add -A` in the Tamir-Learning repo
3. `git diff --cached --stat` to show what changed
4. `git commit` with a message summarizing the changes
5. `git push`

## Execution

```bash
# Sync agents, commands, skills
rsync -av --delete ~/.claude/agents/ /Users/tamirc/Projects/Tamir-Learning/Claude/agents/
rsync -av --delete ~/.claude/commands/ /Users/tamirc/Projects/Tamir-Learning/Claude/commands/
rsync -av --delete ~/.claude/skills/ /Users/tamirc/Projects/Tamir-Learning/Claude/skills/

# Sync settings
cp ~/.claude/settings.json /Users/tamirc/Projects/Tamir-Learning/Claude/settings.json
cp ~/.claude/settings.local.json /Users/tamirc/Projects/Tamir-Learning/Claude/settings.local.json

# Sync project memory
mkdir -p /Users/tamirc/Projects/Tamir-Learning/Claude/memory/scheduler/
cp ~/.claude/projects/-Users-tamirc-IdeaProjects-scheduler/memory/MEMORY.md /Users/tamirc/Projects/Tamir-Learning/Claude/memory/scheduler/MEMORY.md

# Commit and push
cd /Users/tamirc/Projects/Tamir-Learning
git add -A Claude/
git diff --cached --stat
git commit -m "Update Claude agents and config"
git push
```

Summarize what changed in the commit message based on `git diff --cached --stat`.
