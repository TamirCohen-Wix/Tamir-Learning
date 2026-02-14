---
description: "Grafana MCP — Skill Reference"
user-invocable: false
---

# Grafana MCP — Skill Reference

Server name: `grafana-mcp`

This server provides high-level Grafana operations: dashboards, alerts, incidents, Prometheus/Loki queries, Sift investigations, and OnCall. It has **33 tools**. Use this for operational context, NOT for raw log queries (use `grafana-datasource` for that).

---

## Tool Decision Matrix

| Need | Tool | Notes |
|------|------|-------|
| Find error patterns in logs | `find_error_pattern_logs` | High-level pattern detection |
| Find slow requests | `find_slow_requests` | Latency analysis |
| Query Prometheus metrics | `query_prometheus` | Requires datasource UID |
| Query Loki logs | `query_loki_logs` | Requires datasource UID |
| Loki log statistics | `query_loki_stats` | Volume/rate analysis |
| Find a dashboard by name | `search_dashboards` | Returns UID for other tools |
| Get dashboard details | `get_dashboard_by_uid` | Full dashboard JSON |
| Get panel queries | `get_dashboard_panel_queries` | Extract queries from panels |
| List/find datasources | `list_datasources` | Find UIDs |
| Get datasource by name/UID | `get_datasource_by_name` / `get_datasource_by_uid` | Details + UID |
| List alert rules | `list_alert_rules` | Filter by labels |
| Get specific alert | `get_alert_rule_by_uid` | Alert details |
| List incidents | `list_incidents` | Active/resolved |
| Get incident details | `get_incident` | Timeline, labels |
| Sift investigations | `list_sift_investigations` / `get_sift_investigation` | AI analysis |
| Sift analysis details | `get_sift_analysis` | Specific analysis results |
| Assertions (SLOs) | `get_assertions` | SLO status |
| OnCall schedules | `list_oncall_schedules` / `get_current_oncall_users` | Who's on call |
| List teams | `list_teams` / `list_oncall_teams` | Team discovery |
| Prometheus label discovery | `list_prometheus_label_names` / `list_prometheus_label_values` | Metric exploration |
| Prometheus metric discovery | `list_prometheus_metric_names` / `list_prometheus_metric_metadata` | Available metrics |
| Loki label discovery | `list_loki_label_names` / `list_loki_label_values` | Log stream exploration |
| Update dashboard | `update_dashboard` | Modify dashboard JSON |

---

## High-Level Log Analysis Tools

### `find_error_pattern_logs`

Finds recurring error patterns in logs. Use when you want to quickly identify the dominant errors for a service without writing SQL.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | **YES** | Service/app name |
| `labels` | object | **YES** | Label filters (e.g., `{"app": "bookings-service"}`) |
| `start` | string | optional | Start time (RFC 3339) |
| `end` | string | optional | End time (RFC 3339) |

### `find_slow_requests`

Finds slow requests for a service. Use when investigating latency issues.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | **YES** | Service/app name |
| `labels` | object | **YES** | Label filters |
| `start` | string | optional | Start time (RFC 3339) |
| `end` | string | optional | End time (RFC 3339) |

---

## Prometheus (via grafana-mcp)

### `query_prometheus`

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `datasourceUid` | string | **YES** | Get from `list_datasources` or `get_datasource_by_name` |
| `expr` | string | **YES** | PromQL expression |
| `startTime` | string | **YES** | RFC 3339 start time |
| `endTime` | string | optional | RFC 3339 end time |
| `queryType` | string | optional | `"range"` (default) or `"instant"` |
| `stepSeconds` | number | optional | Step interval for range queries |

**Workflow to query Prometheus:**
1. `list_datasources` with `type: "prometheus"` → get the datasource UID
2. (Optional) `list_prometheus_metric_names` → find available metrics
3. (Optional) `list_prometheus_label_names` / `list_prometheus_label_values` → find label filters
4. `query_prometheus` with the UID and PromQL expression

### Prometheus Discovery Tools

| Tool | Parameters | Use |
|------|-----------|-----|
| `list_prometheus_metric_names` | `datasourceUid`, `regex`, `limit`, `page` | Find metrics by pattern |
| `list_prometheus_metric_metadata` | `datasourceUid`, `metric`, `limit`, `limitPerMetric` | Get metric description/type |
| `list_prometheus_label_names` | `datasourceUid`, `matches`, `startRfc3339`, `endRfc3339`, `limit` | Find label keys |
| `list_prometheus_label_values` | `datasourceUid`, `labelName`, `matches`, `startRfc3339`, `endRfc3339`, `limit` | Find label values |

---

## Loki (via grafana-mcp)

### `query_loki_logs`

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `datasourceUid` | string | **YES** | Get from `list_datasources` |
| `logql` | string | **YES** | LogQL query |
| `limit` | number | **YES** | Max log lines |
| `direction` | string | optional | `"forward"` or `"backward"` |
| `startRfc3339` | string | optional | Start time |
| `endRfc3339` | string | optional | End time |

### `query_loki_stats`

Returns volume/rate statistics for a LogQL query. Use to gauge log volume before fetching actual lines.

Same parameters as `query_loki_logs` minus `limit` and `direction`.

### Loki Discovery Tools

| Tool | Parameters | Use |
|------|-----------|-----|
| `list_loki_label_names` | `datasourceUid`, `startRfc3339`, `endRfc3339` | Find label keys |
| `list_loki_label_values` | `datasourceUid`, `labelName`, `startRfc3339`, `endRfc3339` | Find label values |

---

## Dashboards

### `search_dashboards`

| Parameter | Type | Required |
|-----------|------|----------|
| `query` | string | **YES** |

Returns dashboard list with UIDs. Use UID with other dashboard tools.

### `get_dashboard_by_uid`

| Parameter | Type | Required |
|-----------|------|----------|
| `uid` | string | **YES** |

Returns full dashboard JSON (panels, variables, queries).

### `get_dashboard_panel_queries`

| Parameter | Type | Required |
|-----------|------|----------|
| `uid` | string | **YES** |

Extracts just the queries from each panel. Useful for replicating dashboard queries via other tools.

### `update_dashboard`

| Parameter | Type | Required |
|-----------|------|----------|
| `dashboard` | object | **YES** |

Full dashboard JSON to save. Use with caution.

---

## Alerts & Incidents

### `list_alert_rules`

| Parameter | Type | Required |
|-----------|------|----------|
| `page` | number | optional |
| `limit` | number | optional |
| `label_selectors` | string | optional |

Filter by labels: `label_selectors: "app=bookings-service,severity=critical"`

### `get_alert_rule_by_uid`

| Parameter | Type | Required |
|-----------|------|----------|
| `uid` | string | **YES** |

### `list_incidents`

| Parameter | Type | Required |
|-----------|------|----------|
| `status` | string | optional (active/resolved) |
| `drill` | boolean | optional |
| `limit` | number | optional |

### `get_incident`

| Parameter | Type | Required |
|-----------|------|----------|
| `id` | string | optional |

---

## Sift (AI Analysis)

Grafana Sift provides automated investigation of incidents.

### `list_sift_investigations`

| Parameter | Type | Required |
|-----------|------|----------|
| `limit` | number | optional |

### `get_sift_investigation`

| Parameter | Type | Required |
|-----------|------|----------|
| `id` | string | **YES** |

### `get_sift_analysis`

| Parameter | Type | Required |
|-----------|------|----------|
| `investigationId` | string | **YES** |
| `analysisId` | string | **YES** |

---

## Assertions (SLOs)

### `get_assertions`

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `startTime` | string | **YES** | RFC 3339 |
| `endTime` | string | **YES** | RFC 3339 |
| `entityName` | string | optional | Service/entity name |
| `entityType` | string | optional | Entity type filter |
| `env` | string | optional | Environment |
| `site` | string | optional | Site/location |
| `namespace` | string | optional | Kubernetes namespace |

---

## OnCall

### `list_oncall_schedules`

| Parameter | Type | Required |
|-----------|------|----------|
| `teamId` | string | optional |
| `scheduleId` | string | optional |
| `page` | number | optional |

### `get_current_oncall_users`

| Parameter | Type | Required |
|-----------|------|----------|
| `scheduleId` | string | **YES** |

### `list_oncall_teams`

| Parameter | Type | Required |
|-----------|------|----------|
| `page` | number | optional |

### `list_oncall_users`

| Parameter | Type | Required |
|-----------|------|----------|
| `userId` | string | optional |
| `username` | string | optional |
| `page` | number | optional |

### `get_oncall_shift`

| Parameter | Type | Required |
|-----------|------|----------|
| `shiftId` | string | **YES** |

---

## When to Use `grafana-mcp` vs `grafana-datasource`

| Use Case | Server |
|----------|--------|
| Raw SQL queries on app/access logs | `grafana-datasource` |
| Quick error pattern detection | `grafana-mcp` (`find_error_pattern_logs`) |
| Prometheus with UID-based datasource | `grafana-mcp` (`query_prometheus`) |
| Prometheus without knowing UID | `grafana-datasource` (`query_prometheus`) |
| Dashboard discovery and inspection | `grafana-mcp` |
| Alert rules and incidents | `grafana-mcp` |
| Sift AI investigations | `grafana-mcp` |
| SLO/assertion checking | `grafana-mcp` |
| OnCall schedule lookup | `grafana-mcp` |
| Loki with datasource UID | `grafana-mcp` (`query_loki_logs`) |
| Loki without UID | `grafana-datasource` (`query_loki`) |

**General rule:** Use `grafana-datasource` for direct data queries. Use `grafana-mcp` for operational context (dashboards, alerts, incidents, oncall, Sift).
