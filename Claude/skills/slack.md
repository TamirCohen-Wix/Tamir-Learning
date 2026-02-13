# Slack — MCP Skill Reference

Server name: `Slack`

This server provides Slack workspace search, channel browsing, message posting, and thread interaction. It has **12 tools**.

---

## Tool Decision Matrix

| Need | Tool |
|------|------|
| Search for messages by keyword | `search-messages` |
| Read all replies in a thread | `slack_get_thread_replies` |
| Get recent messages in a channel | `slack_get_channel_history` |
| Find a channel by name | `slack_find-channel-id` |
| List all channels | `slack_list_channels` |
| Get a user's profile | `slack_get_user_profile` |
| Find a user by email | `slack_find-user-id-by-email` |
| Post a message | `slack_post_message` |
| Reply to a thread | `slack_reply_to_thread` |
| Add a reaction | `slack_add_reaction` |
| Join a public channel | `slack_join_public_channel` |

---

## Tool 1: `search-messages` (PRIMARY)

The most important Slack tool for investigations. Searches messages across the workspace.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `searchText` | string | optional | Search keywords |
| `exactPhrase` | boolean | optional | Match exact phrase (default: false) |
| `in` | string | optional | Channel name to search in (e.g., `#bookings-alerts`) |
| `from` | string | optional | User who sent the message |
| `after` | string | optional | Messages after this date (format: `YYYY-MM-DD`) |
| `before` | string | optional | Messages before this date (format: `YYYY-MM-DD`) |

### Search Strategy

**Search by KEYWORDS, not by date.** Slack search is keyword-first. Date filters (`after`, `before`) narrow results but keywords are the primary search mechanism.

```
-- Good: keyword-first searches
searchText: "bookings-service timeout"
searchText: "SCHED-45895"
searchText: "NullPointerException bookings"
searchText: "deployment bookings" after: "2026-01-25"

-- Bad: date-only search (no keywords = no results)
after: "2026-01-25" before: "2026-01-27"
```

### Recommended Search Patterns for Bug Investigation

Run MULTIPLE searches with different keyword strategies:

1. **Ticket ID:** `searchText: "SCHED-45895"`
2. **Error message:** `searchText: "NullPointerException SessionService"`
3. **Service name + keyword:** `searchText: "bookings-service error"` or `searchText: "bookings-service down"`
4. **Service name in alerts channel:** `searchText: "bookings-service"` `in: "#bookings-alerts"`
5. **Deployment-related:** `searchText: "deploy bookings"` `after: "2026-01-25"`
6. **Author-specific (from Jira assignee):** `searchText: "bookings"` `from: "john.doe"`

### MANDATORY: Fetch All Thread Replies

For EVERY thread found via `search-messages`:
1. Extract `channel_id` and `thread_ts` from the search result
2. Call `slack_get_thread_replies(channel_id, thread_ts)`
3. Read ALL replies — later replies often contain the resolution, root cause, or correction of the initial message

**Never conclude based on a thread's root message alone.**

---

## Tool 2: `slack_get_thread_replies`

| Parameter | Type | Required |
|-----------|------|----------|
| `channel_id` | string | **YES** |
| `thread_ts` | string | **YES** |

Returns all messages in a thread. The `thread_ts` is the timestamp of the root message (e.g., `"1706354400.123456"`).

---

## Tool 3: `slack_get_channel_history`

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `channel_id` | string | **YES** | Channel ID (not name — use `slack_find-channel-id` first) |
| `limit` | number | optional | Max messages to return |

Use for browsing recent messages in alert/incident channels.

---

## Tool 4: `slack_find-channel-id`

| Parameter | Type | Required |
|-----------|------|----------|
| `channelName` | string | **YES** |

Converts a channel name (e.g., `bookings-alerts`) to a channel ID. Required before using `slack_get_channel_history`.

---

## Tool 5: `slack_list_channels`

| Parameter | Type | Required |
|-----------|------|----------|
| `limit` | number | optional |
| `cursor` | string | optional |

Lists all channels. Use for discovery when you don't know the exact channel name.

---

## Tool 6: `slack_get_user_profile`

| Parameter | Type | Required |
|-----------|------|----------|
| `user_id` | string | **YES** |

---

## Tool 7: `slack_find-user-id-by-email`

| Parameter | Type | Required |
|-----------|------|----------|
| `email` | string | **YES** |

---

## Tool 8: `slack_post_message`

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `channel_id` | string | **YES** | Target channel ID |
| `text` | string | **YES** | Message text |
| `blocks` | array | optional | Block Kit blocks for rich formatting |

**Use sparingly during investigations.** Only post when explicitly instructed by the user.

---

## Tool 9: `slack_reply_to_thread`

| Parameter | Type | Required |
|-----------|------|----------|
| `channel_id` | string | **YES** |
| `thread_ts` | string | **YES** |
| `text` | string | **YES** |

---

## Tool 10: `slack_add_reaction`

| Parameter | Type | Required |
|-----------|------|----------|
| `channel_id` | string | **YES** |
| `timestamp` | string | **YES** |
| `reaction` | string | **YES** |

---

## Tool 11: `slack_join_public_channel`

| Parameter | Type | Required |
|-----------|------|----------|
| `channel_id` | string | **YES** |

---

## Investigation Workflow

1. **Search broadly first:** Run 3-5 keyword searches (ticket ID, error messages, service names)
2. **For each result:** Note channel_id and thread_ts
3. **Fetch ALL threads:** Call `slack_get_thread_replies` for every thread found
4. **Attribution:** Record which channel, who posted, when, and whether the info was confirmed/corrected in thread replies
5. **Report raw data only:** Quote messages verbatim, include timestamps, don't draw conclusions

## Common Wix Slack Channels

| Channel Pattern | Content |
|----------------|---------|
| `#<team>-alerts` | Automated alert notifications |
| `#<team>-incidents` | Incident discussions |
| `#<team>-dev` | Development discussion |
| `#<team>-releases` | Deployment notifications |

## Caution: Slack False Positives

- Slack search can return messages from unrelated contexts that share keywords
- Always verify the time window matches the investigation period
- Thread replies may reverse or correct the root message — read the FULL thread
- "Alert recovered" messages mean the issue was transient — important context
