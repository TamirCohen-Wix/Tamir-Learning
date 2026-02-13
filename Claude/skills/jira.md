# Jira — MCP Skill Reference

Server name: `jira`

This server provides Jira issue management: querying, creating, updating, commenting, transitions, and release management. It has **16 tools**.

---

## Tool Decision Matrix

| Need | Tool |
|------|------|
| Fetch issues by JQL | `get-issues` |
| Get issue change history | `get-issue-changelog` |
| Create a new issue | `create-issue` |
| Update issue fields | `update-issue` |
| Add a comment | `comment-on-issue` |
| Link two issues | `create_issue_link` |
| Transition issue status | `transition-issue` |
| Get available transitions | `get-available-transitions` |
| Get user info | `get_user` |
| List projects | `list-projects` |
| List issue types | `list_issue_types` |
| List fields | `list_fields` |
| List link types | `list_link_types` |
| Get create metadata | `get-create-meta-data` |
| Delete an issue | `delete_issue` |
| Bulk move issues | `bulk-move-issues` |
| Create release version | `create-release-version` |

---

## Tool 1: `get-issues` (PRIMARY)

Fetch issues using JQL (Jira Query Language). This is the main tool for investigations.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectKey` | string | **YES** | Project key (e.g., `SCHED`, `BOOK`) |
| `jql` | string | optional | JQL query string |
| `fields` | string | optional | Comma-separated field names |
| `maxResults` | number | optional | Max issues to return |

### JQL Query Patterns

```
-- Fetch a specific ticket
jql: "key = SCHED-45895"
fields: "key,summary,status,priority,reporter,assignee,description,comment,created,updated"

-- Recent bugs in a project
jql: "project = SCHED AND type = Bug AND created >= -7d ORDER BY created DESC"
fields: "key,summary,status,priority,assignee,created"

-- Bugs assigned to a specific person
jql: "project = SCHED AND assignee = 'john.doe' AND status != Done"

-- Bugs by component
jql: "project = SCHED AND component = 'bookings-service' AND type = Bug"

-- Search by text in summary/description
jql: "project = SCHED AND text ~ 'NullPointerException' AND type = Bug"

-- Issues updated recently (may be related to incident)
jql: "project = SCHED AND updated >= '2026-01-25' AND updated <= '2026-01-28'"

-- Linked issues
jql: "issue in linkedIssues(SCHED-45895)"
```

### Important Fields

| Field | Description |
|-------|-------------|
| `key` | Issue key (e.g., SCHED-45895) |
| `summary` | Issue title |
| `description` | Full description (often contains reproduction steps, logs) |
| `comment` | All comments (often contains investigation findings, root cause) |
| `status` | Current status |
| `priority` | Priority level |
| `reporter` | Who reported it |
| `assignee` | Who is assigned |
| `created` | Creation date |
| `updated` | Last update date |
| `labels` | Issue labels |
| `components` | Components affected |
| `fixVersions` | Target fix versions |

**ALWAYS request the `comment` field** — comments often contain the most valuable investigation context (stack traces, deployment info, workarounds, root cause from previous investigations).

---

## Tool 2: `get-issue-changelog`

Get the change history for an issue. Useful to track when status changed, who changed it, and what fields were modified.

| Parameter | Type | Required |
|-----------|------|----------|
| `issueKey` | string | **YES** |
| `startAt` | number | optional |
| `maxResults` | number | optional |

---

## Tool 3: `create-issue`

| Parameter | Type | Required |
|-----------|------|----------|
| `projectKey` | string | **YES** |
| `summary` | string | **YES** |
| `issueTypeId` | string | **YES** |
| `description` | string | optional |
| `customFields` | object | optional |

Get available issue types with `list_issue_types` and create metadata with `get-create-meta-data`.

---

## Tool 4: `update-issue`

| Parameter | Type | Required |
|-----------|------|----------|
| `issueKey` | string | **YES** |
| `summary` | string | optional |
| `description` | string | optional |
| `customFields` | object | optional |

---

## Tool 5: `comment-on-issue`

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `issueKey` | string | **YES** | Issue key (e.g., SCHED-45895) |
| `comment` | string | **YES** | Comment text (supports Jira markup) |
| `link` | string | optional | URL to attach |

Use after investigation completes to post findings back to the ticket.

---

## Tool 6: `create_issue_link`

| Parameter | Type | Required |
|-----------|------|----------|
| `inwardIssueKey` | string | **YES** |
| `outwardIssueKey` | string | **YES** |
| `linkType` | string | **YES** |

Get available link types with `list_link_types`.

---

## Tool 7: `transition-issue`

Change issue status (e.g., "In Progress" → "Done").

| Parameter | Type | Required |
|-----------|------|----------|
| `issueKey` | string | **YES** |
| `transitionId` | string | **YES** |
| `comment` | string | optional |

**MUST call `get-available-transitions` first** to get valid transition IDs for the current issue state.

---

## Tool 8: `get-available-transitions`

| Parameter | Type | Required |
|-----------|------|----------|
| `issueKey` | string | **YES** |

Returns available status transitions for the issue's current state. Use before `transition-issue`.

---

## Tool 9: `get_user`

| Parameter | Type | Required |
|-----------|------|----------|
| `email` | string | **YES** |

---

## Investigation Workflow

1. **Fetch the ticket:** `get-issues` with `jql: "key = <TICKET>"` and full fields including `comment`
2. **Parse:** Extract summary, description, reproduction steps, any request_ids or MSIDs mentioned
3. **Related issues:** Search for related tickets: `jql: "text ~ '<error_message>' AND project = SCHED"`
4. **Change history:** If investigating whether a ticket was recently reopened/modified: `get-issue-changelog`
5. **Post findings:** After investigation, `comment-on-issue` with the root cause analysis

## JQL Operators Quick Reference

| Operator | Description | Example |
|----------|-------------|---------|
| `=` | Exact match | `status = "Open"` |
| `!=` | Not equal | `status != "Done"` |
| `~` | Contains text | `summary ~ "timeout"` |
| `!~` | Not contains | `summary !~ "test"` |
| `IN` | In list | `status IN ("Open", "In Progress")` |
| `>=`, `<=` | Comparison | `created >= "2026-01-01"` |
| `-7d` | Relative date | `created >= -7d` |
| `ORDER BY` | Sort | `ORDER BY created DESC` |
| `AND`, `OR` | Boolean | `status = "Open" AND priority = "High"` |
