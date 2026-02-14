---
description: "Feature Toggle (FT-release) — MCP Skill Reference"
user-invocable: false
---

# Feature Toggle (FT-release) — MCP Skill Reference

Server name: `FT-release`

This server manages Wix feature toggles (gradual feature releases). It has **7 tools** for searching, querying, inspecting, and creating feature releases.

---

## Tool Decision Matrix

| Need | Tool |
|------|------|
| Search toggles by name | `search-feature-toggles` |
| Query with filters and pagination | `query-feature-toggles` |
| Get full toggle details | `get-feature-toggle` |
| Count toggles by ownership | `get-feature-toggle-counter` |
| List release strategies | `list-strategies` |
| List releases for a toggle | `list-releases` |
| Create a new release | `create-feature-release` |

---

## Tool 1: `search-feature-toggles`

Quick text search for feature toggles.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `searchText` | string | optional | Search term (name, description) |
| `status` | string | optional | Filter by status |
| `ownershipTag` | string | optional | Filter by ownership tag |

### Examples

```
-- Search by name
search-feature-toggles(searchText: "bookings")

-- Search by ownership
search-feature-toggles(ownershipTag: "bookings-service", status: "active")
```

---

## Tool 2: `query-feature-toggles`

Advanced query with pagination and sorting.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | string | optional | Toggle ID |
| `displayName` | string | optional | Display name filter |
| `status` | string | optional | Status filter |
| `ownershipTag` | string | optional | Ownership tag |
| `limit` | number | optional | Results per page |
| `offset` | number | optional | Pagination offset |
| `sortField` | string | optional | Field to sort by |
| `sortOrder` | string | optional | `"asc"` or `"desc"` |

---

## Tool 3: `get-feature-toggle`

Get full details for a specific feature toggle.

| Parameter | Type | Required |
|-----------|------|----------|
| `featureToggleId` | string | **YES** |

Returns: toggle configuration, current status, release history, strategy details, ownership.

**Use this after finding a toggle ID** via search/query to get the complete picture.

---

## Tool 4: `get-feature-toggle-counter`

Count toggles by ownership tag.

| Parameter | Type | Required |
|-----------|------|----------|
| `ownershipTag` | string | **YES** |

---

## Tool 5: `list-strategies`

Lists all available release strategies. No parameters.

Available strategy types (common):
- `VISITOR_ID` — Roll out by visitor
- `USER_ID` — Roll out by logged-in user
- `ACCOUNT_ID` — Roll out by account
- `METASITE_ID` — Roll out by site (metaSiteId)
- `RANDOM` — Random percentage rollout

---

## Tool 6: `list-releases`

List releases for a specific feature toggle.

| Parameter | Type | Required |
|-----------|------|----------|
| `featureToggleId` | string | optional |
| `limit` | number | optional |
| `offset` | number | optional |

---

## Tool 7: `create-feature-release`

Create a new feature release.

| Parameter | Type | Required |
|-----------|------|----------|
| `featureToggleId` | string | **YES** |
| `codeOwnerTag` | string | **YES** |
| `displayName` | string | **YES** |
| `strategyType` or `strategyId` | string | **YES** (one) |
| `description` | string | optional |

---

## Investigation Workflow

Feature toggles are a common cause and fix mechanism for production issues:

1. **Find relevant toggles:**
   ```
   search-feature-toggles(searchText: "<service-name>")
   search-feature-toggles(ownershipTag: "<artifact_id>")
   ```

2. **Check toggle details:**
   ```
   get-feature-toggle(featureToggleId: "<id>")
   ```
   Look for:
   - Recent status changes (toggle turned on/off around incident time)
   - Rollout percentage (partial rollout may explain why only some users are affected)
   - Strategy type (METASITE_ID strategy means site-specific behavior)

3. **Check release history:**
   ```
   list-releases(featureToggleId: "<id>")
   ```
   Look for releases created around the incident timeframe.

### Toggle as Root Cause

A feature toggle change can be the root cause when:
- Toggle was enabled/disabled shortly before the incident
- Toggle controls the code path that's erroring
- Toggle rollout percentage was changed
- New feature behind toggle has a bug

### Toggle as Fix

Toggles are commonly used to quickly fix issues by:
- Disabling a buggy feature (kill switch)
- Reducing rollout percentage to limit blast radius
- Switching traffic to a fallback code path

### Finding Toggles in Code

Use `octocode` to find toggle definitions:
```
githubSearchCode(keywords: ["feature_toggles", "<toggle-name>"], repo: "wix-private/scheduler")
```

Toggles are typically defined in `BUILD.bazel` files under `feature_toggles` rules.
