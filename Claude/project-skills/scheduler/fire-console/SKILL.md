---
description: "Fire Console — MCP Skill Reference for querying Wix domain objects (Bookings, Services, Events, etc.) via gRPC"
user-invocable: false
---

# Fire Console — MCP Skill Reference

Server name: `fire-console`

Fire Console provides **direct gRPC/JSON-RPC access** to Wix production services. Use it to fetch domain objects (Bookings, Services, Events, Orders, etc.) by invoking their Get/List/Query APIs. It also provides service discovery, schema inspection, user/site lookup, and impersonation.

---

## Tool Overview (12 tools)

| Tool | Purpose | When to Use |
|------|---------|-------------|
| `search_services` | Fuzzy search for artifacts/services/endpoints | Find artifact IDs when you don't know the exact name |
| `list_artifacts` | List all available artifacts | Browse the full service catalog |
| `get_artifact_info` | Get ownership info (team, Slack channel) | Find who owns a service |
| `list_services` | List services & methods on an artifact | Discover available APIs before invoking |
| `get_method_schema` | Get request/response schema for a method | Understand payload structure before calling |
| `get_instances` | Get running pods/instances | Find specific instances for debugging |
| `invoke_rpc` | **Call a gRPC/JSON-RPC method** | The main tool — fetch actual data |
| `get_cli_command` | Generate equivalent CLI command | Get reproducible commands for sharing |
| `find_user` | Search users by email/name/ID | Get user ID for impersonation |
| `find_site` | Search sites by name/URL/ID | Get metaSiteId for context |
| `get_client_spec_map` | Get apps installed on a site | Debug client-side app issues |
| `generate_server_signature` | Generate server auth token | Server-to-server API testing |

---

## CRITICAL: Naming Mismatch

**The artifact ID in fire-console is NOT always the same as the code module name.** This is the #1 source of confusion.

| What you see in code | Fire Console artifact ID |
|----------------------|--------------------------|
| bookings-service (bookings domain) | `com.wixpress.bookings.bookings-service` |
| services-2 (services domain) | `com.wixpress.bookings.services-2` |
| enriched bookings (BO query) | `com.wixpress.bookings.bookings-bo-enriched-query-bookings` |
| events-3 (calendar events) | `com.wixpress.calendar.events-3` |
| booking-policies | `com.wixpress.bookings.booking-policies` |
| service-availability | `com.wixpress.bookings.availability.service-availability` |
| availability-calendar | `com.wixpress.bookings.availability.availability-calendar` |

**When in doubt, use `search_services` first** to find the correct artifact ID. Search by service name or keyword — it supports fuzzy matching.

---

## Standard Workflow

### Step 1: Find the artifact

```
search_services(query: "bookings-service", type: "artifact")
→ com.wixpress.bookings.bookings-service
```

Or search by service/method name:
```
search_services(query: "EnrichedBookingService")
→ artifact: com.wixpress.bookings.bookings-bo-enriched-query-bookings
```

### Step 2: Discover available methods

```
list_services(artifactId: "com.wixpress.bookings.bookings-service")
→ Service: com.wixpress.bookings.bookings.v2.Bookings
    Methods: Query, CreateBooking, ConfirmBooking, CancelBooking, ...
```

### Step 3: Check the schema (optional but recommended)

```
get_method_schema(
  artifactId: "com.wixpress.bookings.bookings-service",
  service: "com.wixpress.bookings.bookings.v2.Bookings",
  method: "Query"
)
→ Shows request/response fields, filterable fields, enums
```

The schema response includes:
- `requestType` / `responseType` — proto message names
- `messageTypes` — all field definitions with types and annotations
- `enumTypes` — enum value lists
- `queryFields.filterableFields` — fields you can filter on in Query methods
- `classification.isRead` — whether the method is read-only

### Step 4: Invoke the RPC

```
invoke_rpc(
  target: { artifactId: "com.wixpress.bookings.bookings-service" },
  service: "com.wixpress.bookings.bookings.v2.Bookings",
  method: "Query",
  payload: { "query": {} },
  aspects: { "meta-site-id": "<MSID>" }
)
```

---

## invoke_rpc — Detailed Reference

This is the primary tool. Key parameters:

### target (required — pick one)
- `target.artifactId` — Route via production routing (most common)
- `target.host` + `target.port` — Direct instance targeting
- `target.instance` — Specific instance identifier

### service (required)
Full service FQDN. Examples:
- `com.wixpress.bookings.bookings.v2.Bookings`
- `wix.bookings.services.v2.ServicesService`
- `wix.calendar.events.v3.EventsService`

### method (required)
Method name exactly as shown in `list_services`. Examples: `Query`, `GetEnrichedBooking`, `QueryServices`

### payload
JSON request body. For Query methods, this is typically:
```json
{ "query": {} }
```
Or with filters:
```json
{
  "query": {
    "filter": { "status": "CONFIRMED" },
    "cursor_paging": { "limit": 10 }
  }
}
```

### aspects
Key-value pairs for request context. **Almost always needed:**
```json
{
  "meta-site-id": "<MSID>",
  "user-id": "<USER_ID>"
}
```

### impersonation (alternative to manual aspects)
Auto-generates aspects from user info:
```json
{
  "userId": "<USER_GUID>",
  "metaSiteId": "<META_SITE_ID>"
}
```

### identities
For explicit identity context (server-to-server calls):
```json
[
  { "type": "user", "user": { "userId": "<GUID>" } }
]
```

---

## Bookings Domain Catalog

### 1. Enriched Bookings (booking + order data)

**Use case:** Connect booking IDs to order IDs and vice versa. Contains enriched data with payment, service, and contact info.

```
artifact: com.wixpress.bookings.bookings-bo-enriched-query-bookings
service:  com.wixpress.bookings.boBookingEnricher.EnrichedBookingService
```

| Method | Payload | Notes |
|--------|---------|-------|
| `GetEnrichedBooking` | `{ "bookingId": "<ID>" }` | Single booking with enriched data |
| `QueryEnrichedBookings` | `{ "query": { "filter": {...} } }` | Query with filters |

**Filterable fields:** bookingId, orderId, status, serviceId, contactId, startDate, endDate, etc.

### 2. Bookings (core booking domain)

**Use case:** CRUD operations on bookings. Query by status, date, contact.

```
artifact: com.wixpress.bookings.bookings-service
service:  com.wixpress.bookings.bookings.v2.Bookings
```

| Method | Payload | Notes |
|--------|---------|-------|
| `Query` | `{ "query": {} }` | Query bookings with WQL-style filters |
| `CountBookings` | `{ "query": {} }` | Count matching bookings |

Also exposes: `CreateBooking`, `ConfirmBooking`, `CancelBooking`, `DeclineBooking`, `RescheduleBooking`, `UpdateBooking`, `MarkBookingAsPending`

**Multi-service bookings** are on the same artifact under service `com.wixpress.bookings.bookings.v2.MultiServiceBookings`.

### 3. Services (service catalog)

**Use case:** Query booking services, their settings, categories, locations, policies, and forms.

```
artifact: com.wixpress.bookings.services-2
service:  wix.bookings.services.v2.ServicesService
```

| Method | Payload | Notes |
|--------|---------|-------|
| `GetService` | `{ "id": "<SERVICE_ID>" }` | Single service by ID |
| `QueryServices` | `{ "query": {} }` | Query with CursorQuery |
| `SearchServices` | `{ "search": {...} }` | Full-text search |
| `CountServices` | `{ "query": {} }` | Count matching services |
| `QueryPolicies` | `{ "query": {} }` | Query policies linked to services |
| `QueryCategories` | `{ "query": {} }` | Query service categories |
| `QueryLocations` | `{ "query": {} }` | Query service locations |
| `QueryBookingForms` | `{ "query": {} }` | Query booking forms |

**Booking policy provider** (same artifact, different service):
```
service: com.wixpress.bookings.availability.spi.v2.BookingPolicyProvider
method:  ListBookingPolicies
```

### 4. Calendar Events

**Use case:** Query calendar events (sessions, appointments) by date range, schedule, status.

```
artifact: com.wixpress.calendar.events-3
service:  wix.calendar.events.v3.EventsService
```

| Method | Payload | Notes |
|--------|---------|-------|
| `GetEvent` | `{ "id": "<EVENT_ID>" }` | Single event by ID |
| `QueryEvents` | `{ "query": {...}, "from_local_date": "...", "to_local_date": "..." }` | Query with date range |
| `ListEvents` | `{ ... }` | List events |

**QueryEvents request fields:**
- `from_local_date` / `to_local_date` — LOCAL_DATE_TIME format (e.g., `"2026-02-14T00:00:00"`)
- `time_zone` — e.g., `"America/New_York"`
- `query.filter` — filter by `schedule_id`, `status`, `type`, `app_id`, `resources.id`, etc.
- `query.cursor_paging.limit` — page size (max 1000)

**Event statuses:** `CONFIRMED`, `PENDING_CONFIRMATION`, `CANCELLED`
**Recurrence types:** `NONE`, `MASTER`, `INSTANCE`, `EXCEPTION`

### 5. Booking Policies

**Use case:** Query booking policies (cancellation, reschedule, waitlist, late/early booking limits).

```
artifact: com.wixpress.bookings.booking-policies
service:  wix.bookings.v1.BookingPoliciesService
```

| Method | Payload | Notes |
|--------|---------|-------|
| `GetBookingPolicy` | `{ "id": "<POLICY_ID>" }` | Single policy |
| `QueryBookingPolicies` | `{ "query": {} }` | Query with CursorQuery |
| `CountBookingPolicies` | `{ "query": {} }` | Count policies |
| `GetStrictestBookingPolicy` | `{ ... }` | Get the strictest combined policy |

**Policy includes:** `limit_early_booking_policy`, `limit_late_booking_policy`, `book_after_start_policy`, `cancellation_policy`, `reschedule_policy`, `waitlist_policy`, `participants_policy`, `cancellation_fee_policy`

### 6. Availability

**Use case:** Query available time slots for a service.

```
artifact: com.wixpress.bookings.availability.service-availability
```

**Availability Time Slots:**
```
service: com.wixpress.bookings.availability.v2.AvailabilityTimeSlots
```

| Method | Payload | Notes |
|--------|---------|-------|
| `ListAvailabilityTimeSlots` | `{ "service_id": "<ID>", "from_local_date": "...", "to_local_date": "...", "time_zone": "..." }` | Get available slots |
| `GetAvailabilityTimeSlot` | `{ ... }` | Get single slot details |

**Event Time Slots:**
```
service: com.wixpress.bookings.availability.v2.EventTimeSlots
```

| Method | Payload | Notes |
|--------|---------|-------|
| `ListEventTimeSlots` | `{ ... }` | List slots tied to events |
| `GetEventTimeSlot` | `{ ... }` | Single event slot |

**Multi-Service Availability:**
```
service: com.wixpress.bookings.availability.v2.MultiServiceAvailabilityTimeSlots
```

---

## Query Patterns

### WQL/CursorQuery (most Query methods)

Most Query methods accept a standard `CursorQuery` structure:

```json
{
  "query": {
    "filter": {
      "fieldName": "value",
      "fieldName": { "$in": ["val1", "val2"] },
      "fieldName": { "$gt": "2026-01-01T00:00:00" }
    },
    "sort": [
      { "field_name": "createdDate", "order": "DESC" }
    ],
    "cursor_paging": {
      "limit": 50
    }
  }
}
```

**Filter operators:** `$eq` (default), `$in`, `$gt`, `$gte`, `$lt`, `$lte`, `$ne`, `$contains`, `$startsWith`

**Important:** Use the field names from `queryFields.filterableFields` in the schema response. These are in `snake_case` (proto convention), not `lowerCamelCase`.

### Pagination

Responses include `paging_metadata` with:
- `count` — number of results returned
- `cursors.next` — cursor for next page
- `has_next` — whether more results exist

To get the next page:
```json
{
  "query": {
    "cursor_paging": {
      "cursor": "<next_cursor_from_previous_response>",
      "limit": 50
    }
  }
}
```

---

## Context: aspects vs impersonation

### When you have a metaSiteId (most common):
```
aspects: { "meta-site-id": "<MSID>" }
```

### When you need to act as a specific user:
```
impersonation: { "userId": "<USER_GUID>", "metaSiteId": "<MSID>" }
```

### When you need server-to-server identity:
```
identities: [
  { "type": "serverSignature", "serverSignature": { "appDefId": "<APP_DEF_ID>" } }
]
```

### Finding user/site IDs:
```
find_user(query: "user@email.com")
→ userId, accountId, email, name

find_site(query: "my-site", userId: "<UID>", accountId: "<AID>")
→ metaSiteId, siteName, siteUrl
```

---

## Common Pitfalls

### 1. Wrong artifact ID
**Symptom:** "Service not found" or empty response
**Fix:** Use `search_services` to verify the artifact ID. Code module names often differ from artifact IDs.

### 2. Missing aspects
**Symptom:** "Unauthorized" or "Permission denied"
**Fix:** Most bookings APIs require `meta-site-id` in aspects. Add it.

### 3. Service name mismatch
**Symptom:** "Method not found"
**Fix:** Use `list_services(artifactId: ...)` to get the exact service FQDN. Some services use `com.wixpress.` prefix, others use `wix.` prefix.

### 4. Empty query returns nothing
**Symptom:** No results with `{ "query": {} }`
**Fix:** Ensure you're providing the right context (aspects with meta-site-id). An empty query returns results for the **current site context only**.

### 5. Confusing proto snake_case with API camelCase
**Symptom:** Filter doesn't work
**Fix:** In `invoke_rpc` payloads, use **snake_case** field names (proto convention): `meta_site_id`, `start_date`, `contact_id`. The `fieldMask` in Wix REST APIs uses camelCase, but gRPC uses snake_case.

### 6. Large schema responses
**Symptom:** `get_method_schema` returns extremely large responses
**Fix:** This is normal — schemas include all nested message types. Focus on the `Request` type fields and `queryFields.filterableFields` for Query methods.

---

## Quick Reference: Read-Only Methods by Domain

| Domain | Artifact | Get | Query/List | Count |
|--------|----------|-----|------------|-------|
| Enriched Bookings | `bookings-bo-enriched-query-bookings` | `GetEnrichedBooking` | `QueryEnrichedBookings` | - |
| Bookings | `bookings-service` | - | `Query` | `CountBookings` |
| Services | `services-2` | `GetService` | `QueryServices`, `SearchServices` | `CountServices` |
| Events | `events-3` (calendar) | `GetEvent` | `QueryEvents`, `ListEvents` | - |
| Booking Policies | `booking-policies` | `GetBookingPolicy` | `QueryBookingPolicies` | `CountBookingPolicies` |
| Availability Slots | `service-availability` | `GetAvailabilityTimeSlot` | `ListAvailabilityTimeSlots` | - |
| Event Slots | `service-availability` | `GetEventTimeSlot` | `ListEventTimeSlots` | - |

All artifact IDs above are shorthand — prefix with `com.wixpress.bookings.` (or `com.wixpress.calendar.` for events).

---

## Discovering New Services

To find services beyond the bookings domain:

```
# Search by keyword
search_services(query: "orders")
search_services(query: "ecom")
search_services(query: "contacts")

# Search by method name
search_services(query: "GetOrder")

# Browse all artifacts (very large list)
list_artifacts()

# Get ownership info
get_artifact_info(artifactId: "com.wixpress.bookings.bookings-service")
→ Ownership Tag: bookings-core, Slack Channel: bookings-core
```
