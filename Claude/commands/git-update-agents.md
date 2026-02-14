# Git Update Agents

Sync all Claude configuration files to the Tamir-Learning repo for version tracking.

**IMPORTANT: Delegate ALL work to a Bash sub-agent using the Task tool.** Do NOT run commands in the main context. This keeps the main conversation context clean.

## How to execute

Spawn a single **Bash** sub-agent (via the Task tool, `subagent_type: "Bash"`, `model: "haiku"`) with the full prompt below. Report only the sub-agent's final summary to the user.

### Sub-agent prompt

```
Sync Claude config files to Tamir-Learning repo and commit.

Run these commands:

# Sync directories
rsync -av --delete ~/.claude/agents/ /Users/tamirc/Projects/Tamir-Learning/Claude/agents/
rsync -av --delete ~/.claude/commands/ /Users/tamirc/Projects/Tamir-Learning/Claude/commands/
rsync -av --delete ~/.claude/skills/ /Users/tamirc/Projects/Tamir-Learning/Claude/skills/
rsync -av --delete ~/.claude/hooks/ /Users/tamirc/Projects/Tamir-Learning/Claude/hooks/
rsync -av --delete ~/.claude/output-styles/ /Users/tamirc/Projects/Tamir-Learning/Claude/output-styles/

# Sync individual files
cp ~/.claude/settings.json /Users/tamirc/Projects/Tamir-Learning/Claude/settings.json
cp ~/.claude/settings.local.json /Users/tamirc/Projects/Tamir-Learning/Claude/settings.local.json

# Sync project memory
mkdir -p /Users/tamirc/Projects/Tamir-Learning/Claude/memory/scheduler/
cp ~/.claude/projects/-Users-tamirc-IdeaProjects-scheduler/memory/MEMORY.md /Users/tamirc/Projects/Tamir-Learning/Claude/memory/scheduler/MEMORY.md

# Stage changes
cd /Users/tamirc/Projects/Tamir-Learning
git add -A Claude/

# Check for changes
git diff --cached --name-status

If no staged changes, respond "Nothing to sync — everything is up to date." and stop.

If there are changes, build a commit message:
- Title: "Update Claude config: " + comma-separated list of changed categories (derive category from first dir under Claude/, e.g. Claude/agents/foo.md → "agents", Claude/settings.json → "settings")
- Body: one bullet per file: "- New: path" for A, "- Updated: path" for M, "- Deleted: path" for D (strip the "Claude/" prefix from paths)
- End with: Co-Authored-By: Claude <noreply@anthropic.com>

Then commit (use HEREDOC for message) and push.

Respond with ONLY:
1. The list of changed files (one per line, with New/Updated/Deleted prefix)
2. "Pushed to Tamir-Learning." or "Nothing to sync."
```
