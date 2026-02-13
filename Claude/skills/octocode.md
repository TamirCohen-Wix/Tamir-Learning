# Octocode — MCP Skill Reference

Server name: `octocode`

Octocode is the PRIMARY code search and repository exploration tool. It searches across GitHub repositories with rich query capabilities. It has **7 tools**.

**MANDATORY:** Always use Octocode as the FIRST choice for code search. Only fall back to local Grep/Read/Glob if Octocode returns an error or is unavailable.

---

## Tool Decision Matrix

| Need | Tool |
|------|------|
| Find code by keyword/pattern | `githubSearchCode` |
| Read a specific file | `githubGetFileContent` |
| Browse repository structure | `githubViewRepoStructure` |
| Find PRs by criteria | `githubSearchPullRequests` |
| Discover repositories | `githubSearchRepositories` |
| Search NPM/Python packages | `packageSearch` |

---

## Query Format

All Octocode tools accept a `queries` array. Each query object contains:

```json
{
  "queries": [
    {
      "mainResearchGoal": "Overall investigation objective",
      "query": {
        "researchGoal": "What this specific query aims to find",
        "reasoning": "Why this search will help achieve the goal",
        // ... tool-specific parameters
      }
    }
  ]
}
```

**IMPORTANT:** Always include `mainResearchGoal`, `researchGoal`, and `reasoning`. These help Octocode prioritize and contextualize results.

---

## Tool 1: `githubSearchCode`

Search code across GitHub repositories by keyword, file path, or content pattern.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mainResearchGoal` | string | **YES** | Overall investigation goal |
| `query.researchGoal` | string | **YES** | What this search aims to find |
| `query.reasoning` | string | **YES** | Why this search helps |
| `query.keywords` | string[] | **YES** | Search terms (code identifiers, error messages, etc.) |
| `query.repo` | string | optional | Repo (e.g., `wix-private/scheduler`) |
| `query.match` | string | optional | `"file"` (content match) or `"path"` (filename match) |

### Examples

```json
// Find where an error message is thrown
{
  "queries": [{
    "mainResearchGoal": "Investigate NullPointerException in bookings-service",
    "query": {
      "researchGoal": "Find the source code that throws this specific error",
      "reasoning": "The Grafana logs show this error — finding the source will reveal the condition",
      "keywords": ["SessionService", "NullPointerException"],
      "repo": "wix-private/scheduler",
      "match": "file"
    }
  }]
}

// Find proto definitions
{
  "queries": [{
    "mainResearchGoal": "Understand the API contract for bookings-service",
    "query": {
      "researchGoal": "Find the proto file defining the BookingsService gRPC API",
      "reasoning": "Proto files define the API contract — field names, types, and service methods",
      "keywords": ["BookingsService", "proto"],
      "repo": "wix-private/scheduler",
      "match": "path"
    }
  }]
}

// Find configuration
{
  "queries": [{
    "mainResearchGoal": "Find retry/timeout configuration for bookings-service",
    "query": {
      "researchGoal": "Locate retry and timeout settings",
      "reasoning": "Timeout-related errors may be caused by misconfigured retry/timeout values",
      "keywords": ["retry", "timeout", "backoff"],
      "repo": "wix-private/scheduler",
      "match": "file"
    }
  }]
}
```

### Search Tips

- Use **specific identifiers** (class names, method names, error strings) not generic words
- Use `match: "file"` for code content searches, `match: "path"` for finding files by name
- Search for **exact error messages** from Grafana logs to find the throwing code
- Search for **proto service names** to find API definitions
- Combine **multiple keywords** to narrow results: `["BookingEntry", "validate", "null"]`

---

## Tool 2: `githubGetFileContent`

Read specific file content from a repository.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mainResearchGoal` | string | **YES** | Overall goal |
| `query.researchGoal` | string | **YES** | Why you need this file |
| `query.reasoning` | string | **YES** | How it helps the investigation |
| `query.repo` | string | **YES** | Repository (e.g., `wix-private/scheduler`) |
| `query.path` | string | **YES** | File path within repo |
| `query.branch` | string | optional | Branch name (defaults to main/master) |
| `query.matchString` | string | optional | Highlight/search for this string in the file |
| `query.fullContent` | boolean | optional | Return full file (default: false, returns relevant excerpt) |
| `query.lineStart` | number | optional | Start line for range |
| `query.lineEnd` | number | optional | End line for range |

### Examples

```json
// Read a specific file
{
  "queries": [{
    "mainResearchGoal": "Understand booking validation logic",
    "query": {
      "researchGoal": "Read the BookingValidator class to understand validation rules",
      "reasoning": "Grafana shows validation errors — need to see what conditions trigger them",
      "repo": "wix-private/scheduler",
      "path": "bookings-service/src/main/scala/com/wixpress/bookings/BookingValidator.scala",
      "fullContent": true
    }
  }]
}

// Read with highlight
{
  "queries": [{
    "mainResearchGoal": "Find null check for session ID",
    "query": {
      "researchGoal": "Find where sessionId is validated",
      "reasoning": "NPE in SessionService — need to see if null check exists",
      "repo": "wix-private/scheduler",
      "path": "sessions-server/src/main/scala/com/wixpress/bookings/sessions/SessionService.scala",
      "matchString": "sessionId"
    }
  }]
}
```

---

## Tool 3: `githubViewRepoStructure`

Browse repository directory tree. ALWAYS use this first when investigating a new repository.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mainResearchGoal` | string | **YES** | Overall goal |
| `query.researchGoal` | string | **YES** | Why you need to see the structure |
| `query.reasoning` | string | **YES** | How it helps |
| `query.repo` | string | **YES** | Repository |
| `query.path` | string | optional | Subdirectory path (default: root) |
| `query.depth` | number | optional | Directory depth (default: 2-3) |

### Examples

```json
// Browse repo root
{
  "queries": [{
    "mainResearchGoal": "Understand scheduler monorepo layout",
    "query": {
      "researchGoal": "Map the top-level directory structure",
      "reasoning": "Need to find which directories contain which services",
      "repo": "wix-private/scheduler",
      "depth": 2
    }
  }]
}

// Browse specific service
{
  "queries": [{
    "mainResearchGoal": "Understand bookings-service structure",
    "query": {
      "researchGoal": "See the internal structure of bookings-service",
      "reasoning": "Need to find proto dirs, source dirs, and config files",
      "repo": "wix-private/scheduler",
      "path": "bookings-service",
      "depth": 3
    }
  }]
}
```

---

## Tool 4: `githubSearchPullRequests`

Search for PRs by criteria. Critical for finding changes that may have caused or fixed an issue.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mainResearchGoal` | string | **YES** | Overall goal |
| `query.researchGoal` | string | **YES** | What PRs you're looking for |
| `query.reasoning` | string | **YES** | Why these PRs matter |
| `query.repo` | string | **YES** | Repository |
| `query.keywords` | string[] | optional | Keywords in PR title/body |
| `query.state` | string | optional | `"open"`, `"closed"`, `"merged"` |
| `query.merged` | boolean | optional | Only merged PRs |
| `query.mergedAfter` | string | optional | ISO date (e.g., `"2026-01-25"`) |
| `query.mergedBefore` | string | optional | ISO date |
| `query.createdAfter` | string | optional | ISO date |
| `query.createdBefore` | string | optional | ISO date |
| `query.author` | string | optional | GitHub username |
| `query.type` | string | optional | `"pr"` or `"issue"` |

### Examples

```json
// PRs merged around incident time
{
  "queries": [{
    "mainResearchGoal": "Find PRs that could have caused the incident on Jan 27",
    "query": {
      "researchGoal": "Find PRs merged in the 24 hours before the incident started",
      "reasoning": "Recent code changes are the most common cause of production issues",
      "repo": "wix-private/scheduler",
      "merged": true,
      "mergedAfter": "2026-01-26",
      "mergedBefore": "2026-01-28"
    }
  }]
}

// PRs that fixed the issue
{
  "queries": [{
    "mainResearchGoal": "Find the fix PR for the bookings timeout issue",
    "query": {
      "researchGoal": "Find PRs merged after the incident that mention the error",
      "reasoning": "If the issue stopped, a fix PR was likely merged",
      "repo": "wix-private/scheduler",
      "keywords": ["timeout", "bookings", "fix"],
      "merged": true,
      "mergedAfter": "2026-01-27"
    }
  }]
}

// PRs by specific author
{
  "queries": [{
    "mainResearchGoal": "Check recent changes by the service owner",
    "query": {
      "researchGoal": "Find PRs by the assignee around the incident time",
      "reasoning": "The Jira assignee may have made recent changes",
      "repo": "wix-private/scheduler",
      "author": "john-doe",
      "merged": true,
      "mergedAfter": "2026-01-20",
      "mergedBefore": "2026-01-28"
    }
  }]
}
```

---

## Tool 5: `githubSearchRepositories`

Discover GitHub repositories by keywords, topics, stars, etc.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mainResearchGoal` | string | **YES** | Overall goal |
| `query.researchGoal` | string | **YES** | What repos you need |
| `query.reasoning` | string | **YES** | Why |
| `query.keywords` | string[] | optional | Search terms |
| `query.topics` | string[] | optional | Repository topics |
| `query.owner` | string | optional | Owner (e.g., `wix-private`) |
| `query.stars` | string | optional | Star count filter (e.g., `">10"`) |

Use when a bug spans multiple repositories and you need to find related services.

---

## Tool 6: `packageSearch`

Search for NPM or Python packages.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mainResearchGoal` | string | **YES** | Overall goal |
| `query.researchGoal` | string | **YES** | What package |
| `query.reasoning` | string | **YES** | Why |
| `query.keyword` | string | **YES** | Package name/keyword |
| `query.registry` | string | optional | `"npm"` or `"pypi"` |

---

## MANDATORY Workflow for Bug Investigation

When investigating code for a production bug, ALWAYS follow this order:

### Step 1: Repository Structure
```
githubViewRepoStructure → understand repo layout, find proto dirs, service dirs
```

### Step 2: Proto-First Discovery
```
githubSearchCode → find proto/protobuf files for the service
githubGetFileContent → read proto files to understand API contracts
```

### Step 3: Error Source Location
```
githubSearchCode → search for exact error messages from Grafana logs
githubGetFileContent → read the source files where errors are thrown
```

### Step 4: Flow Tracing
```
githubSearchCode → find callers of the erroring code
githubGetFileContent → read caller code to understand the full call chain
```

### Step 5: Configuration
```
githubSearchCode → find BUILD.bazel, application.conf, feature toggles
githubGetFileContent → read config files for retry/timeout/toggle settings
```

### Step 6: PR/Change Analysis
```
githubSearchPullRequests → find PRs merged around incident time
```

### Step 7: Cross-Repository (if needed)
```
githubSearchRepositories → find related repos
Repeat steps 1-6 for each related repo
```

---

## Common Patterns for Wix Scheduler Monorepo

| Looking for | Search approach |
|------------|-----------------|
| Service entry point | `githubSearchCode` with `["extends", "ServiceEntry"]` in `wix-private/scheduler` |
| gRPC handler | `githubSearchCode` with `["override def", "<methodName>"]` |
| Proto definitions | `githubSearchCode` with `["service", "<ServiceName>"]`, `match: "path"`, filter `.proto` |
| Feature toggle | `githubSearchCode` with `["feature_toggles", "<toggle-name>"]` |
| BUILD.bazel config | `githubGetFileContent` with path `<service>/BUILD.bazel` |
| Retry/timeout config | `githubSearchCode` with `["retryPolicy", "timeout"]` |
| Domain event handler | `githubSearchCode` with `["Greyhound", "consume", "<topic>"]` |
| SDL/DB operations | `githubSearchCode` with `["ScalikeJdbc", "sql\""]` |
