# Memory

## Production Master Pipeline

### Investigation Cycle (CORRECT ORDER)
1. Bug Context (Jira fetch + bug-context agent)
2. Parallel Data Fetch (Grafana, Production, Codebase, Slack — all 4 in parallel)
3. Hypothesis Generation (based on fetched evidence)
4. Verification (confidence score 0-100%)
   - If >= 85%: proceed to fix plan
   - If < 85%: verifier specifies per-agent targeted data requests, loop back to step 2
   - Max 3 iterations

### Sub-Agent Rules
- Each agent's `.md` file contains a pre-loaded tool list — agents use `select:` syntax, no keyword search
- Agent files live in `~/.claude/agents/` (USER scope), command files in `~/.claude/commands/` (USER scope)
- When making pipeline changes, always update the relevant agent/command files
- **File location consistency:** Agents and commands are USER-scoped (`~/.claude/`). Only auto-memory is project-scoped. Never create project-scoped duplicates of agents/commands.

### Agent-to-Tool Mapping
| Agent | MCP Tool Categories | Key Tools |
|-------|-------------------|-----------|
| slack-analyzer | slack (6 tools) | `search-messages`, `get_channel_history`, `get_thread_replies` |
| production-analyzer | github (10), devex (12), gradual-feature-release (4) | `list_commits`, `list_pull_requests`, `find_commits_by_date_range` |
| codebase-semantics | octocode (6) | `githubSearchCode`, `githubGetFileContent`, `githubViewRepoStructure` |
| grafana-analyzer | grafana-datasource (10) | `query_app_logs`, `query_access_logs`, `query_prometheus` |
| bug-context | jira (4) | `get-issues`, `get-issue-changelog` |
| hypotheses | any (as needed) | grafana, feature toggles, octocode |
| verifier | any (as needed) | grafana, devex, github, feature toggles, jira |
| fix-list | gradual-feature-release (6) | `search-feature-toggles`, `create-feature-release` |
| documenter | jira (2), slack (1) | `comment-on-issue`, `find-channel-id` |

### MCP Tools
- Sub-agents DO have access to ToolSearch and MCP tools — they can load and use them directly
- Orchestrator checks MCP-s connection in Step 0.6 and authenticates if needed
- Agents use `ToolSearch("select:<exact_tool_name>")` — tool lists are in their `.md` files
- Known tool name prefix: `mcp__mcp-s__`
- Key tool categories: jira, grafana-datasource, slack, github, gradual-feature-release, devex, octocode

### Output Directory
- Production Master creates a directory per run: `debug-<TICKET>-<YYYY-MM-DD-HHmmss>/`
- Each sub-agent writes its output to `<sub-agent-name>.md` in that directory
- Sub-agents must be told the output file path and instructed to write their report there

### Grafana Queries
- Sub-agents should run LIVE queries via MCP tools (discovered via ToolSearch), NOT build URLs
- Let them choose the appropriate query tool based on what they need (app logs, access logs, prometheus, loki, etc.)
- Key artifact IDs: `com.wixpress.bookings.reader.bookings-reader`, `com.wixpress.bookings.notifications-server`
- Always compare baseline vs incident vs post-incident periods

### Jira
- Load tool: `ToolSearch("select:mcp__mcp-s__jira__get-issues")`, then fetch with JQL `key = SCHED-XXXXX`
- Fields: key, summary, status, priority, reporter, assignee, description, comment

### Documentation Reports
- Keep reports SHORT and DIRECT — under 60 lines ideal
- No repetition — state the root cause ONCE, not 5 times in different sections
- Structure: TL;DR -> Evidence (numbers) -> Causal Chain -> Fix -> References
- Skip verbose sections: lessons learned, people involved, hypotheses explored
- People won't read long reports

### Slack Message Posting Rules
- **NEVER reference a Slack channel by name or link without verifying it exists first** — use `slack_find-channel-id` to confirm
- **NEVER fabricate channel names** — if you don't know the exact channel, omit the reference or say "the relevant team channel"
- **Verify ALL hyperlinks before posting** — broken links undermine credibility
- When posting investigation summaries to Slack threads, keep links to verified resources only (Grafana URLs, Jira tickets, GitHub PRs)

## Codebase Patterns

### Feature Toggles
- Feature toggles are defined in `BUILD.bazel` (`feature_toggles = ["name"]`) and managed via **Wix Dev Portal** (NOT Petri)
- Use `gradual-feature-release` MCP tools to search/query toggle status on Wix Dev Portal
- Legacy: some services still have Petri specs but new toggles should use Wix Dev Portal

### Rate Limiting
- bookings-reader uses `LoomPrimeRateLimiter` with MSID as entity key
- `FeatureLimitExceeded` extends `WixApplicationRuntimeException` with `ResourceExhausted`
- VIP policies managed via Fire Console, documented in `docs/rate-limit-configuration-guide.md`

### TimeCapsule + Greyhound
- TimeCapsule is built on Greyhound (confirmed by stack traces)
- Retry config in `notifications-server-config.json.erb`: 1, 10, 20, 60, 120, 360 minutes
