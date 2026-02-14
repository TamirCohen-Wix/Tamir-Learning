---
description: Query application logs via Grafana for debugging and monitoring
allowed-tools:
  - mcp__grafana_datasource_mcp__query_app_logs
  - mcp__MCP-S__grafana-datasource__query_app_logs
  - Task
---

# Query App Logs

Query application logs for analysis, debugging, and monitoring.

## Usage

```
/query-app-logs artifact: <artifact_id> [level: ERROR] [search: pattern] [time: 1h]
```

## When to Use

- General log browsing and pattern search
- Error analysis and aggregations
- SDL operation tracing
- Domain event publishing logs

**NOT for request-specific tracing** - use `/investigate-logs` instead
**NOT for error aggregations by tenant** - use `/query-error-logs` instead

## Parameters

| Parameter | Required | Example |
|-----------|----------|---------|
| artifact | Yes | com.wixpress.bookings.bookings-service |
| level | No | ERROR, WARN, INFO, DEBUG |
| search | No | timeout, EntityNotFound |
| time | No | 1h, 6h, 24h (default: 1h) |
| caller | No | SDL, grpc-handler |

## Examples

### Recent errors
```
/query-app-logs artifact: com.wixpress.bookings.bookings-service level: ERROR
```

### Search for timeouts
```
/query-app-logs artifact: com.wixpress.bookings.bookings-service search: timeout time: 6h
```

### SDL operations
```
/query-app-logs artifact: com.wixpress.bookings.bookings-service caller: SDL
```

## Instructions

When invoked, perform the log query work directly using the MCP tools. If the query is complex or involves multiple independent queries, delegate to a sub-agent.

**Environment-Aware Sub-Agent Invocation (for complex queries):**
- **Claude Code**: Use `Task` tool with `subagent_type: "general-purpose"` or `"app-logs-agent"` if available
- **Cursor**: Use `@agent` or spawn a background agent
- **Other**: Use whatever sub-agent/task mechanism is available in the current environment

For simple queries, execute directly using the MCP tool.

## Mandatory Query Behavior

### Rule 1: No Query Expansion on Empty Results
When a query returns no data:
- DO NOT automatically expand the time range
- DO NOT add wildcards or loosen filters
- INSTEAD: Inspect your filtering - verify artifact_id spelling, check if filters are too restrictive
- Report: "No results found. Verify: [list of filters used]"

### Rule 2: Single Datasource Commitment
- Use `mcp__grafana_datasource_mcp__query_app_logs` or `mcp__MCP-S__grafana-datasource__query_app_logs` (whichever is available)
- If the tool fails, FAIL FAST - do not switch to alternative datasources
- Report the failure clearly and ask user for guidance

### Rule 3: MCP-First, Fail Fast
- Use MCP tools for all queries
- If MCP call fails (auth, timeout, error), stop immediately
- Report: "MCP tool failed: [error]. Cannot proceed without user action."

### Rule 4: Strict Timeframes
- NEVER expand timeframes speculatively
- Default timeframe: Use skill parameter or 1h
- If user doesn't specify timeframe, ASK: "What time range? (1h/6h/24h)"
- If no data in specified range, report empty - don't auto-expand

### Rule 5: Minimal Token Usage
- Start with COUNT queries before fetching full data
- Use LIMIT 20-50 for initial exploration
- Only fetch more if user explicitly requests

### Rule 6: Verify Before Querying
- Confirm artifact_id format looks correct (e.g., `com.wixpress.*`)
- Don't guess artifact names - ask user if unclear

## App Logs Schema

| Column | Type | Description |
|--------|------|-------------|
| `timestamp` | DateTime | When the log was written |
| `artifact_id` | String | Service identifier |
| `level` | String | DEBUG, INFO, WARN, ERROR |
| `message` | String | Log message |
| `data` | String (JSON) | Structured log data (visibility events) |
| `stack_trace` | String | Exception stack trace |
| `caller` | String | Code location or component name |
| `error_class` | String | Exception class name |
| `error_code` | String | Application error code |
| `request_id` | String | Request correlation ID |
| `dc` | String | Data center |
| `hostname` | String | Pod/host name |
| `pod_name` | String | Kubernetes pod name (rollout/canary/stable) |
| `meta_site_id` | String | Tenant/MetaSite ID |

## SQL Query Templates

### Base query with time filter
```sql
SELECT timestamp, level, message, caller, error_class, data, stack_trace
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
ORDER BY timestamp DESC
LIMIT 100
```

### With level filter
```sql
SELECT timestamp, level, message, caller, error_class, data, stack_trace
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
  AND level = '<LEVEL>'
ORDER BY timestamp DESC
LIMIT 100
```

### With message search (LIKE pattern)
```sql
SELECT timestamp, level, message, caller, error_class, data, stack_trace
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
  AND message LIKE '%<SEARCH>%'
ORDER BY timestamp DESC
LIMIT 100
```

### With caller filter
```sql
SELECT timestamp, level, message, caller, error_class, data, stack_trace
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
  AND caller LIKE '%<CALLER>%'
ORDER BY timestamp DESC
LIMIT 100
```

### Error aggregation by type
```sql
SELECT error_class, count() as cnt
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
  AND level = 'ERROR'
GROUP BY error_class
ORDER BY cnt DESC
LIMIT 20
```

### Log volume by level
```sql
SELECT level, count() as cnt
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
GROUP BY level
ORDER BY cnt DESC
LIMIT 10
```

## Wix Log Patterns

| Message Pattern | Description |
|-----------------|-------------|
| `visibility.GenericEvent` | Custom visibility logs (check `data.details`) |
| `visibility.ScalikeVisibilityEvent` | SDL operation with SQL and timing |
| `visibility.ProducedRecord` | Domain event published to Kafka |
| `Experiment Conduction Summary` | Feature toggle conduction |
| `*/Query request` | gRPC query request received |
| `*/Query response` | gRPC query response with duration |

## Parsing the `data` Column (JSON)

The `data` column contains structured JSON. Use ClickHouse JSON functions:

```sql
-- Extract nested field
SELECT JSONExtractString(data, 'details') as details
FROM app_logs
WHERE ...

-- Extract numeric field
SELECT JSONExtractInt(data, 'duration') as duration_ms
FROM app_logs
WHERE ...

-- Filter by JSON field value
SELECT *
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
  AND JSONExtractString(data, 'topic') = 'domain_events_wix.bookings.v2.booking'
LIMIT 50
```

## Common Queries

### Recent Errors
```sql
SELECT timestamp, level, message, caller, error_class, stack_trace
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
  AND level = 'ERROR'
ORDER BY timestamp DESC
LIMIT 50
```

### SDL Operations
```sql
SELECT timestamp, message, data, caller
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
  AND caller LIKE '%SDL%'
ORDER BY timestamp DESC
LIMIT 50
```

### Domain Events Published
```sql
SELECT timestamp, message, data
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
  AND message LIKE '%ProducedRecord%'
ORDER BY timestamp DESC
LIMIT 50
```

### Timeout Search
```sql
SELECT timestamp, level, message, caller, error_class, stack_trace
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
  AND (message LIKE '%timeout%' OR error_class LIKE '%Timeout%')
ORDER BY timestamp DESC
LIMIT 50
```

## Output Format

Present results as:

```
=== App Logs: <artifact_id> ===
Time Range: <from> to <to>
Filters: level=<level>, search=<pattern>
Total Entries: <count>

### Summary
- Errors: X
- Warnings: Y
- Most common error: <error_class>

### Log Entries
[timestamp] [level] [caller] message
  Error: <error_class>
  Stack: <first line of stack_trace>
```

## Troubleshooting

### No logs found
- Verify artifact_id matches exactly (check BUILD.bazel or deployment)
- DO NOT expand time range automatically - ask user
- Remove filters to see if any logs exist

### Too many logs
- Add level filter (`level: ERROR`)
- Add caller filter
- Narrow time range

### Query timeout
- Reduce time range
- Add more specific filters
- Use LIMIT with smaller value

## Tool Configuration

Use `mcp__grafana_datasource_mcp__query_app_logs` or `mcp__MCP-S__grafana-datasource__query_app_logs` (whichever is available):

```
mcp__grafana_datasource_mcp__query_app_logs
  sql: "<SQL with $__timeFilter(timestamp), artifact_id filter, LIMIT>"
  fromTime: "<ISO 8601 start>"
  toTime: "<ISO 8601 end>"
```

```
mcp__MCP-S__grafana-datasource__query_app_logs
  sql: "<SQL with $__timeFilter(timestamp), artifact_id filter, LIMIT>"
  fromTime: "<ISO 8601 start>"
  toTime: "<ISO 8601 end>"
```
