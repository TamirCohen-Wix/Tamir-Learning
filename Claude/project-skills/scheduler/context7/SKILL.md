---
description: "Context7 — MCP Skill Reference"
user-invocable: false
---

# Context7 — MCP Skill Reference

Server name: `context-7`

This server provides access to up-to-date library documentation. It has **2 tools**. Use it when you need to look up API docs, framework features, or library-specific patterns.

---

## Tool 1: `resolve-library-id`

Resolves a library name to its Context7 ID. MUST be called first before `query-docs`.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | **YES** | Library description or name (e.g., `"scala play framework"`) |
| `libraryName` | string | **YES** | Exact library name (e.g., `"playframework/playframework"`) |

Returns a library ID that you pass to `query-docs`.

---

## Tool 2: `query-docs`

Query documentation for a specific library.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `libraryId` | string | **YES** | ID from `resolve-library-id` |
| `query` | string | **YES** | What you want to know (e.g., `"how to configure retry policy"`) |

Returns relevant documentation excerpts.

---

## Workflow

Always follow this two-step process:

1. **Resolve the library:**
   ```
   resolve-library-id(query: "ScalikeJDBC database library", libraryName: "scalikejdbc/scalikejdbc")
   ```

2. **Query the docs:**
   ```
   query-docs(libraryId: "<id-from-step-1>", query: "connection pool configuration")
   ```

---

## When to Use

- Understanding a library API that appears in error stack traces
- Looking up configuration options for frameworks used in the codebase
- Verifying correct usage patterns when analyzing potential bugs
- Checking if a library behavior matches the bug hypothesis

## Common Libraries in Wix Bookings Context

| Library | Use Case |
|---------|----------|
| ScalikeJDBC | Database access (SDL operations) |
| Greyhound | Kafka event streaming |
| gRPC / Protobuf | Service-to-service communication |
| Play Framework | HTTP server |
| Guice | Dependency injection |
| Specs2 / ScalaTest | Testing |
