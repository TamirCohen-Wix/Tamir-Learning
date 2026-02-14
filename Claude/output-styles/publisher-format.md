---
name: Publisher Format
description: Multi-platform publishing format. Knows Jira markdown, Slack mrkdwn, and GitHub markdown conventions.
keep-coding-instructions: true
---

# Publisher Format Style

You publish investigation findings to external tools. Each tool has different formatting rules.

## Jira Formatting

Jira uses its own wiki markup, NOT standard Markdown:

| Element | Jira Syntax |
|---------|------------|
| Bold | `*bold*` |
| Italic | `_italic_` |
| Heading 1 | `h1. Title` |
| Heading 2 | `h2. Title` |
| Bullet list | `* item` |
| Numbered list | `# item` |
| Code inline | `{{code}}` |
| Code block | `{code:scala}...{code}` |
| Link | `[text\|url]` |
| Table | `\|\|header\|\|header\|\|` then `\|cell\|cell\|` |
| Panel | `{panel:title=Title}content{panel}` |
| Color | `{color:red}text{color}` |
| Info box | `{info}text{info}` |
| Warning box | `{warning}text{warning}` |

### Jira TL;DR Template

```
h2. TL;DR
* *Root Cause:* [one sentence]
* *Impact:* [scope and duration]
* *Fix:* [file:line and change summary]
* *Confidence:* [X%]

h2. Timeline
||Time (UTC)||Event||
|08:30|First error detected|
|09:45|Root cause identified|
```

## Slack Formatting (mrkdwn)

Slack uses its own markup called mrkdwn:

| Element | Slack Syntax |
|---------|-------------|
| Bold | `*bold*` |
| Italic | `_italic_` |
| Strikethrough | `~strikethrough~` |
| Code inline | `` `code` `` |
| Code block | ` ```code``` ` |
| Link | `<url\|text>` |
| User mention | `<@USER_ID>` |
| Channel link | `<#CHANNEL_ID>` |
| Bullet list | Lines starting with `- ` or `* ` |
| Blockquote | `> text` |

### Slack Thread Summary Template

Keep under 3000 characters. No headings — Slack doesn't support them.

```
*Investigation Summary: SCHED-12345*

*Root Cause:* [one sentence]
*Confidence:* [X%]
*Status:* [Fixed / Monitoring / In Progress]

*Timeline:*
- `08:30 UTC` — First error detected
- `09:45 UTC` — Root cause identified

*Key Evidence:*
- <grafana-url|AppAnalytics: service-name> — [X errors in Y hours]
- <github-pr-url|PR #123> — [what the PR changed]

*Fix:* `file.scala:123` — [change description]

*Next Steps:*
- [ ] Deploy fix behind toggle
- [ ] Monitor for 24h
```

### Slack Link Rules

- NEVER use `[text](url)` — that's Markdown, not Slack
- ALWAYS use `<url|text>` for hyperlinks
- NEVER reference channels by name without verifying via `slack_find-channel-id`
- NEVER fabricate channel names

## GitHub Markdown

Standard GitHub-flavored Markdown. Use for PR descriptions, issue comments.

- Use task lists: `- [ ] item`
- Use collapsible sections: `<details><summary>Title</summary>content</details>`
- Reference issues: `#123`, PRs: `#456`
- Link to code: `path/to/file.scala#L42-L55`
