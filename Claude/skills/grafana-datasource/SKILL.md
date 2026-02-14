---
description: "Grafana Datasource — MCP Skill Reference"
user-invocable: false
---

# Grafana Datasource — MCP Skill Reference

Server name: `grafana-datasource`

This server provides direct SQL/PromQL/LogQL access to Wix production data stores. It has **11 tools** — each targeting a specific datasource. Choose the right tool for your data need.

---

## Tool Decision Matrix

| Question | Tool | Why |
|----------|------|-----|
| App-level errors, messages, stack traces? | `query_app_logs` | ClickHouse app_logs table |
| HTTP status codes, request URIs, latency? | `query_access_logs` | ClickHouse nginx access logs |
| Panorama slow/error events? | `query_panorama` | ClickHouse panorama_events |
| BI events (user actions, conversions)? | `query_bi_events` | ClickHouse BI events |
| Domain events (Kafka published events)? | `query_domain_events` | ClickHouse domain events |
| Metric time series (CPU, memory, rate)? | `query_prometheus` | Prometheus (14-day retention) |
| Metric time series (longer retention)? | `query_prometheus_aggr` | Prometheus aggregated (90-day) |
| Free-text log lines, labels, streams? | `query_loki` | Loki log streams (LogQL) |
| Universal Grafana panel-style query? | `grafana_query` | Any datasource (advanced) |
| List all available datasources? | `list_datasources` | Discovery/debugging |

---

## CRITICAL: Common Parameters

All SQL-based tools (`query_app_logs`, `query_access_logs`, `query_panorama`, `query_bi_events`, `query_domain_events`) share the same parameter format:

| Parameter | Type | Required | Format |
|-----------|------|----------|--------|
| **`sql`** | string | **YES** | SQL query with `$__timeFilter(timestamp)` in WHERE clause |
| **`fromTime`** | string | **YES** | ISO 8601 UTC: `2026-01-27T00:00:00.000Z` (MUST end `.000Z`) |
| **`toTime`** | string | **YES** | ISO 8601 UTC: `2026-01-29T23:59:59.000Z` (MUST end `.000Z`) |

### FATAL MISTAKE: Using `query` instead of `sql`
```
WRONG: query_app_logs(query: "SELECT ...")   → "Invalid arguments: sql Required"
RIGHT: query_app_logs(sql: "SELECT ...", fromTime: "...", toTime: "...")
```

### MANDATORY: `$__timeFilter(timestamp)` macro
Every SQL query MUST include `$__timeFilter(timestamp)` in the WHERE clause. This macro is replaced server-side with the actual time range from `fromTime`/`toTime`.

```sql
-- WRONG (no time filter macro)
SELECT * FROM app_logs WHERE artifact_id = 'com.wixpress.bookings.bookings-service'

-- RIGHT
SELECT * FROM app_logs WHERE $__timeFilter(timestamp) AND artifact_id = 'com.wixpress.bookings.bookings-service'
```

---

## Tool 1: `query_app_logs`

Application-level logs from all Wix services. This is the primary debugging tool.

### Schema: `app_logs`

| Column | Type | Description | Filterable |
|--------|------|-------------|------------|
| `timestamp` | DateTime | Log write time | YES (via macro) |
| `artifact_id` | String | Service ID (e.g., `com.wixpress.bookings.bookings-service`) | YES |
| `level` | String | `DEBUG`, `INFO`, `WARN`, `ERROR` | YES |
| `message` | String | Log message text | YES (LIKE) |
| `data` | String (JSON) | Structured payload (details, params, etc.) | YES (JSONExtract) |
| `stack_trace` | String | Exception stack trace | YES (LIKE) |
| `caller` | String | Code location / component name | YES (LIKE) |
| `error_class` | String | Exception class name | YES |
| `error_code` | String | Application error code | YES |
| `request_id` | String | Request correlation ID | YES |
| `meta_site_id` | String | Tenant / MetaSite ID | YES |

### SQL Templates

```sql
-- 1. Error overview (ALWAYS run first to understand scope)
SELECT message, count() as cnt
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
  AND level = 'ERROR'
GROUP BY message
ORDER BY cnt DESC
LIMIT 100

-- 2. Error details with stack traces
SELECT timestamp, request_id, message, caller, error_class, error_code, stack_trace
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
  AND level = 'ERROR'
ORDER BY timestamp DESC
LIMIT 50

-- 3. Filter by specific error class
SELECT timestamp, request_id, message, data, stack_trace
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
  AND error_class = '<ERROR_CLASS>'
ORDER BY timestamp DESC
LIMIT 50

-- 4. Filter by tenant (meta_site_id)
SELECT timestamp, request_id, level, message, caller
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
  AND meta_site_id = '<MSID>'
  AND level IN ('ERROR', 'WARN')
ORDER BY timestamp DESC
LIMIT 100

-- 5. Trace a single request across all services
SELECT timestamp, artifact_id, level, message, caller, error_class
FROM logs_db.app_logs
WHERE $__timeFilter(timestamp)
  AND request_id = '<REQUEST_ID>'
ORDER BY timestamp ASC
LIMIT 100

-- 6. Artifact discovery from request_id
SELECT DISTINCT nginx_artifact_name
FROM logs_db.id_to_app_mv
WHERE request_id = '<REQUEST_ID>'
  AND $__timeFilter(timestamp)
LIMIT 500

-- 7. Error aggregation by class
SELECT error_class, error_code, count() as cnt
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
  AND level = 'ERROR'
GROUP BY error_class, error_code
ORDER BY cnt DESC
LIMIT 50

-- 8. Error timeline (pattern over time)
SELECT toStartOfHour(timestamp) as hour, count() as cnt
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
  AND level = 'ERROR'
GROUP BY hour
ORDER BY hour ASC

-- 9. Search message content
SELECT timestamp, level, message, caller, data
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
  AND message LIKE '%<SEARCH_TERM>%'
ORDER BY timestamp DESC
LIMIT 50

-- 10. Parse JSON data column
SELECT timestamp, message,
  JSONExtractString(data, 'details') as details,
  JSONExtractString(data, 'bookingId') as booking_id
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
  AND data != ''
ORDER BY timestamp DESC
LIMIT 50

-- 11. SDL / DB operations
SELECT timestamp, message, data, caller
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
  AND caller LIKE '%SDL%'
ORDER BY timestamp DESC
LIMIT 50

-- 12. Domain events published to Kafka
SELECT timestamp, message, data
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
  AND message LIKE '%ProducedRecord%'
ORDER BY timestamp DESC
LIMIT 50

-- 13. Feature toggle conduction results
SELECT timestamp, message, data
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
  AND message LIKE '%Experiment Conduction Summary%'
ORDER BY timestamp DESC
LIMIT 50
```

### Wix Log Message Patterns

| Pattern | Description | Where to look |
|---------|-------------|---------------|
| `visibility.GenericEvent` | Custom visibility logs | `data.details` |
| `visibility.ScalikeVisibilityEvent` | SDL operation with SQL/timing | `data` |
| `visibility.ProducedRecord` | Domain event published to Kafka | `data` |
| `Experiment Conduction Summary` | Feature toggle conduction | `data` |
| `*/Query request` | gRPC query request received | `message` |
| `*/Query response` | gRPC query response with duration | `message` |

---

## Tool 2: `query_access_logs`

HTTP/nginx access logs. Shows request routing, status codes, latency.

### Schema: `nginx`

| Column | Type | Description | Filterable |
|--------|------|-------------|------------|
| `timestamp` | DateTime | Request time | YES (via macro) |
| `nginx_artifact_name` | String | Service identifier | **REQUIRED** |
| `nginx_request_method` | String | HTTP method (GET, POST, etc.) | YES |
| `nginx_request_uri` | String | Request URI path | YES (LIKE) |
| `http_status_code` | Int | Response status (200, 500, etc.) | YES |
| `request_time` | Float | Duration in ms | YES (> for slow) |
| `request_id` | String | Request correlation ID | YES |

### MANDATORY: `nginx_artifact_name` filter
The tool ENFORCES that `nginx_artifact_name` appears in the SQL. Omitting it causes an error.

```sql
-- Basic access log query
SELECT timestamp, nginx_request_method, nginx_request_uri, http_status_code, request_time, request_id
FROM nginx
WHERE $__timeFilter(timestamp)
  AND nginx_artifact_name = '<ARTIFACT>'
ORDER BY timestamp DESC
LIMIT 100

-- Find slow requests (>1 second)
SELECT timestamp, nginx_request_uri, http_status_code, request_time
FROM nginx
WHERE $__timeFilter(timestamp)
  AND nginx_artifact_name = '<ARTIFACT>'
  AND request_time > 1000
ORDER BY request_time DESC
LIMIT 50

-- Error status codes
SELECT timestamp, nginx_request_method, nginx_request_uri, http_status_code, request_time
FROM nginx
WHERE $__timeFilter(timestamp)
  AND nginx_artifact_name = '<ARTIFACT>'
  AND http_status_code >= 500
ORDER BY timestamp DESC
LIMIT 50

-- Status code distribution
SELECT http_status_code, count() as cnt
FROM nginx
WHERE $__timeFilter(timestamp)
  AND nginx_artifact_name = '<ARTIFACT>'
GROUP BY http_status_code
ORDER BY cnt DESC
```

---

## Tool 3: `query_panorama`

Panorama events — structured operational events with severity. Useful for understanding service health beyond simple logs.

### Schema: `panorama_events`

| Column | Type | Description |
|--------|------|-------------|
| `date_created` | DateTime | Event time |
| `full_artifact_id` | String | Service ID (equivalent to artifact_id) |
| `log_level` | String | Severity level |
| `message` | String | Event message |

**Note:** Uses `date_created` and `full_artifact_id` (not `timestamp` and `artifact_id`).

```sql
SELECT date_created, full_artifact_id, log_level, message
FROM panorama_events
WHERE $__timeFilter(date_created)
  AND full_artifact_id = '<ARTIFACT>'
  AND log_level = 'ERROR'
ORDER BY date_created DESC
LIMIT 50
```

---

## Tool 4: `query_bi_events`

Business Intelligence events — user-facing actions, conversions, funnel events.

```sql
SELECT *
FROM bi_events
WHERE $__timeFilter(timestamp)
LIMIT 20
```

Use when investigating user-facing impact of bugs (e.g., "did users fail to complete booking?").

---

## Tool 5: `query_domain_events`

Domain events published to Kafka topics.

```sql
SELECT *
FROM domain_events
WHERE $__timeFilter(timestamp)
LIMIT 20
```

Use when investigating event-driven workflows (e.g., "was the booking-created event published?").

---

## Tool 6: `query_prometheus`

Prometheus metrics with **14-day retention**. Uses PromQL.

| Parameter | Type | Required | Format |
|-----------|------|----------|--------|
| `expr` | string | **YES** | PromQL expression |
| `from` | string | optional | ISO 8601 (defaults to -1h) |
| `to` | string | optional | ISO 8601 (defaults to now) |

```
-- Request rate
rate(http_requests_total{artifact_id="com.wixpress.bookings.bookings-service"}[5m])

-- Error rate
rate(http_requests_total{artifact_id="com.wixpress.bookings.bookings-service", status_code=~"5.."}[5m])

-- P99 latency
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{artifact_id="com.wixpress.bookings.bookings-service"}[5m]))

-- JVM memory
jvm_memory_bytes_used{artifact_id="com.wixpress.bookings.bookings-service"}

-- GC pauses
rate(jvm_gc_collection_seconds_sum{artifact_id="com.wixpress.bookings.bookings-service"}[5m])
```

---

## Tool 7: `query_prometheus_aggr`

Same as `query_prometheus` but with **90-day retention** (aggregated, lower resolution). Use for long-term trends.

---

## Tool 8: `query_loki`

Loki log streams using LogQL. Use for free-text search across log streams when ClickHouse queries aren't sufficient.

| Parameter | Type | Required | Format |
|-----------|------|----------|--------|
| `query` | string | **YES** | LogQL expression |
| `limit` | number | **YES** | Max lines to return |
| `from` | string | optional | ISO 8601 |
| `to` | string | optional | ISO 8601 |

```
-- Search by label and content
{app="com.wixpress.bookings.bookings-service"} |= "error" |= "timeout"

-- JSON parsing
{app="com.wixpress.bookings.bookings-service"} | json | level="ERROR"

-- Regex filter
{app="com.wixpress.bookings.bookings-service"} |~ "(?i)null.*pointer"
```

---

## Tool 9: `grafana_query`

Universal Grafana query tool — can query ANY datasource. Use when the specialized tools don't cover your use case or you need to replicate a specific dashboard panel.

| Parameter | Type | Required |
|-----------|------|----------|
| `query` | string | **YES** |
| `datasource` | string | **YES** |
| `from` | string | optional |
| `to` | string | optional |
| `limit` | number | optional |
| `queryType` | string | optional |

**When to use:** Only when specialized tools don't work, or when replicating a specific Grafana dashboard panel query.

---

## Tool 10: `list_datasources`

Lists all available Grafana datasources. Use for discovery — to find datasource names and UIDs needed by other tools.

| Parameter | Type | Required |
|-----------|------|----------|
| `typeFilter` | string | optional |

---

## Grafana AppAnalytics URL Construction

When providing results to users, include clickable Grafana URLs for verification:

### Base URL
```
https://grafana.wixpress.com/d/olcdJbinz/app-analytics?orgId=1
```

### Required Parameters
```
&from=<FROM_ISO_MS>&to=<TO_ISO_MS>   (Unix milliseconds)
&timezone=browser
&var-app=<ARTIFACT_ID>
&var-not_logger_name=greyhound
&var-level=ERROR                      (or ALL, WARN, INFO)
```

### Optional Filters
```
&var-fieldToFilter=meta_site_id
&var-fieldToFilterCondition=ILIKE
&var-fieldToFilterInput=<MSID>
```

### Full Example
```
https://grafana.wixpress.com/d/olcdJbinz/app-analytics?orgId=1&from=1706313600000&to=1706486400000&timezone=browser&var-app=com.wixpress.bookings.bookings-service&var-not_logger_name=greyhound&var-level=ERROR&var-fieldToFilter=meta_site_id&var-fieldToFilterCondition=ILIKE&var-fieldToFilterInput=abc123
```

### Converting ISO to Unix milliseconds
```bash
date -u -j -f "%Y-%m-%dT%H:%M:%S" "2026-01-27T00:00:00" "+%s000"
```

---

## Best Practices

1. **Start with error aggregation** — Always run `GROUP BY message/error_class` first to understand the error landscape before diving into individual logs.
2. **Use tight time ranges** — Start with 1h, expand only if explicitly needed. Never exceed 14 days.
3. **LIMIT everything** — Always include LIMIT. Start with 20-50 for exploration, up to 100 for detail.
4. **Cross-reference request_ids** — Find a request_id in app_logs, then trace it in access_logs for the full picture.
5. **Check multiple services** — If one service shows errors, check upstream/downstream services too.
6. **Use timeline queries** — `toStartOfHour(timestamp)` grouping reveals when issues started/stopped.
7. **Parse JSON data** — The `data` column often contains the most useful details. Use `JSONExtractString()`.

---

## Common Investigation Patterns

### Pattern 1: Context Loss Detection
When error logs have NULL `meta_site_id` but you expect MSID:
- This means the crash happens BEFORE MSID is set in the logging context
- Try searching by booking ID / order ID in the `data` field instead:
```sql
SELECT timestamp, message, data, request_id, meta_site_id
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
  AND data LIKE '%<BOOKING_ID>%'
ORDER BY timestamp ASC LIMIT 100
```
- Try searching by exact timestamp correlation (±1 second)

### Pattern 2: Retry Wave Detection
To detect retry behavior (multiple peaks with declining amplitude = retry waves):
```sql
SELECT toStartOfHour(timestamp) as hour, count() as cnt
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
  AND level = 'ERROR'
GROUP BY hour
ORDER BY hour ASC
```

### Pattern 3: Recovery Point Detection
To find what changed when errors stopped:
```sql
SELECT timestamp, level, message, data, request_id
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND artifact_id = '<ARTIFACT>'
  AND timestamp >= '<ERROR_DROP_TIME - 30min>'
  AND timestamp <= '<ERROR_DROP_TIME>'
ORDER BY timestamp ASC LIMIT 200
```

### Pattern 4: Cross-Service Request Tracing
Given a request_id from one service, find it across ALL services:
```sql
SELECT timestamp, artifact_id, level, message, caller
FROM app_logs
WHERE $__timeFilter(timestamp)
  AND request_id = '<REQUEST_ID>'
ORDER BY timestamp ASC LIMIT 100
```
