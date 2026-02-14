---
description: Investigate application logs by request ID for production debugging
allowed-tools:
  - mcp__grafana_datasource_mcp__query_app_logs
  - mcp__MCP-S__grafana-datasource__query_app_logs
  - mcp__grafana_datasource_mcp__query_access_logs
  - mcp__MCP-S__grafana-datasource__query_access_logs
  - Task
  - Skill
---

# Investigate Logs

Trace requests through application and access logs by request ID.

## Usage

```
/query-request [artifact_id] <request_id> [time_range] [level]
```

## When to Use

- Trace a specific request through the system
- Debug production issues with a request ID
- Analyze request flow and timing
- Find errors for a specific request

**NOT for general log browsing** - use `/query-app-logs` instead
**NOT for error aggregations** - use `/query-error-logs` instead

## Parameters

| Parameter | Required | Example |
|-----------|----------|---------|
| artifact_id | Optional | com.wixpress.bookings.bookings-service |
| request_id | Yes | 1769611570.535540810122211411840 |
| time_range | No | 1h, 6h, 24h (default: auto from request_id) |
| level | No | ERROR, WARN, INFO |

## Examples

### Basic request trace (cross-service discovery)
```
/query-request 1769611570.535540810122211411840
```

### Trace within specific service
```
/query-request com.wixpress.bookings.bookings-service 1769611570.535540810122211411840
```

### Errors only
```
/query-request 1769611570.535540810122211411840 1h ERROR
```

## Instructions

When invoked, perform the log tracing work directly using the MCP tools. For complex traces or parallel queries, delegate to a sub-agent.

**Environment-Aware Sub-Agent Invocation (for complex traces):**
- **Claude Code**: Use `Task` tool with `subagent_type: "general-purpose"` or `"request-tracer-agent"` if available
- **Cursor**: Use `@agent` or spawn a background agent
- **Other**: Use whatever sub-agent/task mechanism is available in the current environment

For simple traces, execute directly using the MCP tools.

## Mandatory Query Behavior

### Rule 1: No Query Expansion on Empty Results
When a query returns no data:
- DO NOT automatically expand the time range
- DO NOT add wildcards or loosen filters
- INSTEAD: Verify request_id format and time range accuracy
- Report: "No results found. Verify: [request_id, time range]"

### Rule 2: Single Datasource Commitment
- Use `mcp__grafana_datasource_mcp__query_app_logs` (or `mcp__MCP-S__grafana-datasource__query_app_logs`) and `mcp__grafana_datasource_mcp__query_access_logs` (or `mcp__MCP-S__grafana-datasource__query_access_logs`) - whichever is available
- If the tool fails, FAIL FAST - do not switch to alternative datasources
- Report the failure clearly and ask user for guidance

### Rule 3: MCP-First, Fail Fast
- Use MCP tools for all queries
- If MCP call fails (auth, timeout, error), stop immediately
- Report: "MCP tool failed: [error]. Cannot proceed without user action."

### Rule 4: Extract Timeframe from Request ID
Wix request IDs contain a Unix timestamp as the first part (before the dot):
- Format: `<unix_timestamp>.<random_id>` (e.g., `1769611570.535540810122211411840`)
- The first part `1769611570` is a Unix timestamp (seconds since epoch)
- **Calculate timeframe automatically**: 10 minutes before to 10 minutes after the timestamp

**Use the epoch skill for conversion:**
1. Extract the Unix timestamp from the request ID (part before the dot)
2. Invoke the `epoch` skill using the Skill tool to convert it to ISO 8601
3. Calculate fromTime (timestamp - 600) and toTime (timestamp + 600)

Example:
```
Request ID: 1769611570.535540810122211411840
-> Extract: 1769611570
-> Use Skill tool: skill="epoch", args="1769611570"
-> Get ISO 8601 result and calculate +/-10 minute range
```

**IMPORTANT**: Always use the epoch skill for timestamp conversion - do NOT manually calculate or ask the user for timeframe when request ID is provided.

### Rule 4b: Fallback Timeframes
- If request ID doesn't contain a valid timestamp, ask the user for timeframe
- NEVER expand timeframes speculatively
- If no data in specified range, report empty - don't auto-expand

### Rule 5: Minimal Token Usage
- Query app_logs first, then access_logs only if needed
- Use LIMIT 100 for initial trace
- Order by timestamp ASC to see flow chronologically

### Rule 6: Artifact Filtering is Optional for Request ID Queries
When tracing by request_id:
- **App Logs**: artifact_id filter is OPTIONAL - query across all services to see the full request flow
- **Access Logs**: nginx_artifact_name is still required by the tool (enforced)

**Default behavior** (no artifact specified by user):
- Query app_logs WITHOUT artifact_id filter to discover all services involved
- This reveals cross-service calls and the complete request journey

**When user specifies artifact(s)**:
- Honor the user's request and filter to those specific artifacts
- Use this to focus investigation on particular services

**Query strategy**:
1. First query: `id_to_app_mv` to discover all artifacts that handled the request (fast, lightweight lookup)
2. Use discovered artifacts for focused app_logs and access_logs queries
3. If user asks to focus on specific services, add artifact_id filter
4. For access_logs, use discovered artifacts as nginx_artifact_name (required by tool)

## App Logs Schema (logs_db.app_logs)

| Column | Type | Description |
|--------|------|-------------|
| `timestamp` | DateTime | When the log was written |
| `request_id` | String | Request correlation ID |
| `artifact_id` | String | Service identifier |
| `level` | String | DEBUG, INFO, WARN, ERROR |
| `message` | String | Log message |
| `data` | String (JSON) | Structured log data (visibility events) |
| `stack_trace` | String | Exception stack trace |
| `caller` | String | Code location or component name |
| `error_class` | String | Exception class name |
| `error_code` | String | Application error code |
| `dc` | String | Data center |
| `hostname` | String | Pod/host name |
| `pod_name` | String | Kubernetes pod name |
| `meta_site_id` | String | Tenant/MetaSite ID |

## Access Logs Schema (nginx table)

| Column | Type | Description |
|--------|------|-------------|
| `timestamp` | DateTime | Request time |
| `request_id` | String | Request correlation ID |
| `nginx_artifact_name` | String | Service identifier |
| `nginx_request_method` | String | HTTP method |
| `nginx_request_uri` | String | Request URI |
| `http_status_code` | Int | Response status |
| `request_time` | Float | Request duration (milliseconds) |
| `nginx_http_user_agent` | String | Client user agent |
| `nginx_remote_addr` | String | Client IP |

## SQL Query Templates

### Artifact Discovery (Step 1 - Always Run First)
**Use `id_to_app_mv` to quickly discover all artifacts that handled the request:**
```sql
SELECT DISTINCT nginx_artifact_name
FROM logs_db.id_to_app_mv
WHERE request_id = '<REQUEST_ID>'
  AND $__timeFilter(timestamp)
LIMIT 500
```

### Basic query by request_id (App Logs) - Cross-Service Discovery
**Use this after artifact discovery to get detailed logs:**
```sql
SELECT timestamp, artifact_id, level, message, caller, error_class
FROM logs_db.app_logs
WHERE $__timeFilter(timestamp)
  AND request_id = '<REQUEST_ID>'
ORDER BY timestamp ASC
LIMIT 100
```

### Filtered by specific artifact (when user requests focus)
```sql
SELECT timestamp, level, message, data, stack_trace, caller, error_class, error_code
FROM logs_db.app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT_ID>'
  AND request_id = '<REQUEST_ID>'
ORDER BY timestamp ASC
LIMIT 100
```

### With level filter
```sql
SELECT timestamp, artifact_id, level, message, data, stack_trace, caller, error_class, error_code
FROM logs_db.app_logs
WHERE $__timeFilter(timestamp)
  AND request_id = '<REQUEST_ID>'
  AND level = '<LEVEL>'
ORDER BY timestamp ASC
LIMIT 100
```

### Errors only
```sql
SELECT timestamp, artifact_id, level, message, data, stack_trace, caller, error_class, error_code
FROM logs_db.app_logs
WHERE $__timeFilter(timestamp)
  AND request_id = '<REQUEST_ID>'
  AND level IN ('ERROR', 'WARN')
ORDER BY timestamp ASC
LIMIT 100
```

### Access Logs Query
**MUST include nginx_artifact_name filter (enforced by tool)**
```sql
SELECT timestamp, nginx_request_method, nginx_request_uri, http_status_code, request_time, request_id
FROM nginx
WHERE $__timeFilter(timestamp)
  AND nginx_artifact_name = '<ARTIFACT_ID>'
  AND request_id = '<REQUEST_ID>'
ORDER BY timestamp ASC
LIMIT 100
```

## Output Format

### Request Flow Summary

```
=== Application Log Trace: <request_id> ===
Time Range: <from> to <to>
Total Entries: <count>
Services Involved: <list of artifact_ids discovered>

### Request Flow Summary
| Timestamp | Service | Caller | Operation | Duration |
|-----------|---------|--------|-----------|----------|
| 16:09:31.137 | bookings-service | bookings-catalog | ConsistentQuery request | - |
| 16:09:31.153 | bookings-service | bookings-catalog | ConsistentQuery response | 16ms |
| 16:09:31.200 | availability-calendar | slot-checker | CheckAvailability | 47ms |

### Key Details
- Booking ID: <id>
- Tenant: <meta_site_id>
- Feature Toggles: useBookingsPlatform=true
- Domain Events Published: topic, partition, offset

### Errors Found
[Any errors with stack traces]
```

### HTTP Request Trace

```
=== HTTP Request Trace: <request_id> ===
Artifact: <artifact_id>
Time Range: <from> to <to>

[2024-01-15 10:30:15.123] POST /_api/bookings-service/v2/bookings
  Status: 200 | Duration: 847ms
```

## Highlight Patterns

When analyzing logs, highlight:
- **Errors and warnings** - prominently
- **High latency operations** - >100ms
- **Stack traces** - summarized
- **Domain events** - published to Kafka
- **SDL operations** - and their durations
- **Feature toggles** - conduction results

## Common Log Message Types

| Message Pattern | Description |
|-----------------|-------------|
| `*/Query request` | gRPC query request received |
| `*/Query response` | gRPC query response with duration |
| `visibility.GenericEvent` | Custom visibility log (check `data.details`) |
| `visibility.ScalikeVisibilityEvent` | SDL operation with SQL and duration |
| `visibility.ProducedRecord` | Domain event published to Kafka |
| `Experiment Conduction Summary` | Feature toggle/experiment conducted |

## Advanced Queries

### Find all requests for a meta_site_id with errors
```sql
SELECT DISTINCT request_id, timestamp, message, error_class
FROM logs_db.app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT_ID>'
  AND meta_site_id = '<META_SITE_ID>'
  AND level = 'ERROR'
ORDER BY timestamp DESC
LIMIT 100
```

### Aggregate error counts by type
```sql
SELECT error_class, error_code, count() as cnt
FROM logs_db.app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT_ID>'
  AND level = 'ERROR'
GROUP BY error_class, error_code
ORDER BY cnt DESC
LIMIT 50
```

### Find slow requests (from access logs)
```sql
SELECT timestamp, nginx_request_uri, http_status_code, request_time
FROM nginx
WHERE $__timeFilter(timestamp)
  AND nginx_artifact_name = '<ARTIFACT_ID>'
  AND request_time > 1000
ORDER BY request_time DESC
LIMIT 50
```

## Troubleshooting

### No logs found
- Check if request_id format is correct
- Verify timestamp extraction from request_id is accurate
- DO NOT expand time range automatically - ask user

### Too many logs
- Add level filter (`level = 'ERROR'`)
- Add artifact_id filter to focus on specific service
- Add caller filter
- DO NOT narrow time range automatically - ask user

### Query timeout
- Reduce time range
- Add more specific filters
- Use LIMIT with smaller value
- Report the issue - do not auto-retry

## Tool Configuration

Use whichever tool variant is available in your environment.

### App Logs
```
mcp__grafana_datasource_mcp__query_app_logs
  sql: "<SQL with $__timeFilter(timestamp), request_id filter, LIMIT>"
        # artifact_id is OPTIONAL when filtering by request_id
  fromTime: "<ISO 8601 start>"
  toTime: "<ISO 8601 end>"
```

```
mcp__MCP-S__grafana-datasource__query_app_logs
  sql: "<SQL with $__timeFilter(timestamp), request_id filter, LIMIT>"
        # artifact_id is OPTIONAL when filtering by request_id
  fromTime: "<ISO 8601 start>"
  toTime: "<ISO 8601 end>"
```

### Access Logs
```
mcp__grafana_datasource_mcp__query_access_logs
  sql: "<SQL with $__timeFilter(timestamp), nginx_artifact_name filter, LIMIT>"
        # nginx_artifact_name is REQUIRED (enforced by tool)
  fromTime: "<ISO 8601 start>"
  toTime: "<ISO 8601 end>"
```

```
mcp__MCP-S__grafana-datasource__query_access_logs
  sql: "<SQL with $__timeFilter(timestamp), nginx_artifact_name filter, LIMIT>"
        # nginx_artifact_name is REQUIRED (enforced by tool)
  fromTime: "<ISO 8601 start>"
  toTime: "<ISO 8601 end>"
```
