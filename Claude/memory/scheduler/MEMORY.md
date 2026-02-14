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
- **ALL subagents use `model: "sonnet"`** — no exceptions, no Opus tiering

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
| skeptic | any (as needed) | cross-examines two hypotheses, applies 5-point checklist |
| fix-list | gradual-feature-release (6) | `search-feature-toggles`, `create-feature-release` |
| documenter | jira (2), slack (1) | `comment-on-issue`, `find-channel-id` |

### MCP Tools
- Sub-agents DO have access to ToolSearch and MCP tools — they can load and use them directly
- Orchestrator checks ALL 6 MCP servers in Step 0.3 (hard gate) — ALL must pass, no exceptions
- Agents use `ToolSearch("select:<exact_tool_name>")` — tool lists are in their `.md` files
- **Local fallback (e.g., `gh` CLI, local git) requires explicit user approval** and must be recorded in the agent's trace file

### MCP Server Map
| Server Prefix | Categories | Notes |
|---|---|---|
| `mcp__mcp-s__` | jira, slack, github, grafana-datasource, gradual-feature-release, context7 | Multi-tool server |
| `mcp__octocode__` | octocode (githubSearchCode, githubGetFileContent, etc) | Separate server, NOT under mcp-s |
| `mcp__Slack__` | slack (post_message, reply_to_thread, add_reaction) | Dedicated Slack with write access |
| `mcp__grafana-mcp__` | grafana-mcp (dashboards, alerts, prometheus) | Grafana Cloud MCP |
| `mcp__grafana-datasource__` | grafana-datasource (same as mcp-s) | Duplicate of mcp-s grafana tools |
| `mcp__github__` | github (same as mcp-s) | Duplicate of mcp-s github tools |
| `mcp__jira__` | jira (same as mcp-s) | Duplicate of mcp-s jira tools |
| `mcp__FT-release__` | gradual-feature-release (same as mcp-s) | Duplicate of mcp-s FT tools |
| `mcp__context-7__` | context7 (resolve-library-id, query-docs) | Duplicate of mcp-s context7 tools |

**Key insight:** Most tools exist on both `mcp-s` AND dedicated servers. Pipeline uses `mcp-s` prefix by default. Octocode is the exception — only on `mcp__octocode__`.

### Output Directory
- Location: `.claude/debug/` inside the repo root (or `./debug/` outside a repo)
- Directory name: `debug-<TASK-SLUG>-<YYYY-MM-DD-HHmmss>/` where TASK-SLUG is Jira ticket ID or auto-generated summary
- Each agent gets its OWN subdirectory: `<debug-dir>/<agent-name>/`
- Output files are versioned: `<agent-name>-output-V<N>.md` (N = invocation count for that agent in this run)
- **Trace files**: `<agent-name>-trace-V<N>.md` — written alongside output files, contain input + action log
- Example: `debug-SCHED-4353-2026-02-11-143000/grafana-analyzer/grafana-analyzer-output-V1.md`
- Example: `debug-SCHED-4353-2026-02-11-143000/grafana-analyzer/grafana-analyzer-trace-V1.md`
- `findings-summary.md` and `report.md` stay at the debug dir root
- Sub-agents must be told both OUTPUT_FILE and TRACE_FILE paths
- **Trace isolation**: Trace files are NEVER passed to other agents. Only the human operator reads them.

### Agent Task-Driven Design
- Agents do NOT know about orchestration steps ("Step 3", "Step 4", "parallel", "primary")
- The orchestrator passes a `TASK` input that tells the agent exactly what to do
- codebase-semantics has two report types: "Report Type A" (error propagation) and "Report Type B" (PR analysis)
- The orchestrator selects the report type via the TASK field

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

### Agent Teams (Hypothesis Phase)
- Steps 5-6 use agent teams (experimental) when `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` is set
- 3 teammates: hypothesis-tester-A, hypothesis-tester-B, skeptic
- Tasks: A and B test competing theories in parallel, skeptic is blocked by both, cross-examines and produces verdict
- Fallback: if env var not set, uses sequential subagent approach (old Steps 5-6)
- Setting lives in `~/.claude/settings.json` under `env`
- Skeptic agent: `~/.claude/agents/skeptic.md` — replaces verifier role within teams
- `verifier.md` still exists for sequential fallback path

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
