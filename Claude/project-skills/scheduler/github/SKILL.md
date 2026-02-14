---
description: "GitHub — MCP Skill Reference"
user-invocable: false
---

# GitHub — MCP Skill Reference

Server name: `github`

This server provides GitHub repository operations: commits, PRs, issues, code search, branching, and file management. It has **23 tools**.

**Note:** For code search, prefer `octocode` (richer query capabilities). Use `github` for PR details, commit listings, branch comparisons, and write operations (creating PRs, issues, comments).

---

## Tool Decision Matrix

| Need | Tool |
|------|------|
| List commits on a branch | `list_commits` |
| List PRs for a repo | `list_pull_requests` |
| Get specific PR details | `get_pull_request` |
| Get PR review comments | `get_pull_request_comments` |
| Get PR reviews | `get_pull_request_reviews` |
| Compare two branches | `compare_branches` |
| Search code | `search_code` |
| Search issues/PRs | `search_issues` |
| Search repositories | `search_repositories` |
| Get file contents | `get_file_contents` |
| Create a PR | `create_pull_request` |
| Create a branch | `create_branch` |
| Create/update a file | `create_or_update_file` |
| Push multiple files | `push_files` |
| Create an issue | `create_issue` |
| List issues | `list_issues` |
| Update an issue | `update_issue` |
| Add issue comment | `add_issue_comment` |
| Create PR review | `create_pull_request_review` |
| Merge a PR | `merge_pull_request` |
| Update PR branch | `update_pull_request_branch` |
| Reply to PR comment | `reply-to-pull-request-comment` |

---

## Key Investigation Tools

### `list_commits`

List commits on a branch. Use to find changes around incident time.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `owner` | string | **YES** | Repo owner (e.g., `wix-private`) |
| `repo` | string | **YES** | Repo name (e.g., `scheduler`) |
| `sha` | string | optional | Branch name or SHA |
| `page` | number | optional | Page number |
| `perPage` | number | optional | Results per page |

### `list_pull_requests`

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `owner` | string | **YES** | Repo owner |
| `repo` | string | **YES** | Repo name |
| `state` | string | optional | `"open"`, `"closed"`, `"all"` |
| `sort` | string | optional | `"created"`, `"updated"`, `"popularity"` |
| `direction` | string | optional | `"asc"` or `"desc"` |
| `branch` | string | optional | Filter by head branch |

### `get_pull_request`

| Parameter | Type | Required |
|-----------|------|----------|
| `owner` | string | **YES** |
| `repo` | string | **YES** |
| `pull_number` | number | **YES** |

Returns full PR details including body, commits, files changed, merge status.

### `get_pull_request_comments`

| Parameter | Type | Required |
|-----------|------|----------|
| `owner` | string | **YES** |
| `repo` | string | **YES** |
| `pull_number` | number | **YES** |

### `get_pull_request_reviews`

| Parameter | Type | Required |
|-----------|------|----------|
| `owner` | string | **YES** |
| `repo` | string | **YES** |
| `pull_number` | number | **YES** |

### `compare_branches`

Compare two refs (branches, tags, commits). Shows the diff between them.

| Parameter | Type | Required |
|-----------|------|----------|
| `owner` | string | **YES** |
| `repo` | string | **YES** |
| `base` | string | **YES** |
| `head` | string | **YES** |

### `search_code`

| Parameter | Type | Required |
|-----------|------|----------|
| `query` | string | **YES** |
| `page` | number | optional |
| `perPage` | number | optional |

GitHub code search query syntax: `"error message" repo:wix-private/scheduler language:scala`

### `search_issues`

Search issues and PRs across GitHub.

| Parameter | Type | Required |
|-----------|------|----------|
| `query` | string | **YES** |
| `page` | number | optional |
| `perPage` | number | optional |

Query syntax: `"fix timeout" repo:wix-private/scheduler is:pr is:merged`

### `get_file_contents`

| Parameter | Type | Required |
|-----------|------|----------|
| `owner` | string | **YES** |
| `repo` | string | **YES** |
| `path` | string | **YES** |
| `branch` | string | optional |

---

## Investigation Workflow

### Finding the Cause

1. **List recent commits:**
   ```
   list_commits(owner: "wix-private", repo: "scheduler", sha: "master", perPage: 30)
   ```

2. **List recently merged PRs:**
   ```
   list_pull_requests(owner: "wix-private", repo: "scheduler", state: "closed", sort: "updated", direction: "desc")
   ```

3. **Search for fix PRs:**
   ```
   search_issues(query: "fix timeout bookings repo:wix-private/scheduler is:pr is:merged")
   ```

4. **Get PR details:**
   ```
   get_pull_request(owner: "wix-private", repo: "scheduler", pull_number: 12345)
   ```

5. **Compare what changed:**
   ```
   compare_branches(owner: "wix-private", repo: "scheduler", base: "abc123", head: "def456")
   ```

### When to Use `github` vs `octocode`

| Operation | Use `github` | Use `octocode` |
|-----------|-------------|----------------|
| Search code content | OK (basic) | **PREFERRED** (rich queries) |
| Read a file | OK | **PREFERRED** (matchString, lineRange) |
| List commits | **YES** | No |
| PR details | **YES** | PR search only |
| PR comments/reviews | **YES** | No |
| Branch comparison | **YES** | No |
| Create PR/issue/branch | **YES** | No |
| Post comments | **YES** | No |
| Repo structure | Limited | **PREFERRED** (depth control) |
