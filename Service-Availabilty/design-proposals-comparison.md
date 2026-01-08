# ListScheduleTimeSlots: Design Proposals Comparison

## Executive Summary

This document presents design proposals for extending the `ListEventTimeSlots` API (renamed conceptually to `ListScheduleTimeSlots`) to support both classes and courses with unified schedule-level availability information.

| Proposal | Name | Complexity | Backend Changes | Risk | Supports Functionality 1 | Supports Functionality 2 |
|----------|------|------------|-----------------|------|--------------------------|--------------------------|
| **A** | Minimal Enhancement | Low | Small | Low | ❌ Client-side | ✅ |
| **B** | Unified Schedule Availability | Medium | Medium | Medium | ✅ Server-side | ✅ |
| **C** | Dedicated Course API | High | Large | High | ✅ Server-side | ✅ (separate endpoint) |

### Two Core Functionalities

| Functionality | Description | Use Case |
|---------------|-------------|----------|
| **1. Schedule Availability** | Is this schedule bookable as a whole? | Quick check before showing booking UI |
| **2. TimeSlots Query** | List all time slots with full details | Calendar view, session selection |

---

## Current State

### Current System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              CURRENT CLIENT FLOWS                                   │
└─────────────────────────────────────────────────────────────────────────────────────┘

CLASS AVAILABILITY:
┌──────────┐     ┌────────────────────┐     ┌────────────────────┐
│  Client  │ ──► │ Services V2 Query  │ ──► │ ListEventTimeSlots │
└──────────┘     │ (get serviceId,    │     │ (get TimeSlots)    │
                 │  bookingPolicies)  │     └────────────────────┘
                 └────────────────────┘              │
                                                     ▼
                                               TimeSlots[] with:
                                               ✅ bookable (per slot)
                                               ✅ remainingCapacity (per slot)
                                               ✅ bookingPolicyViolations (per slot)
                                               ❌ scheduleId (NOT populated!)

COURSE AVAILABILITY:
┌──────────┐     ┌────────────────────┐     ┌────────────────────┐
│  Client  │ ──► │ Services V2 Query  │ ──► │  Schedule Server   │
└──────────┘     │ (get serviceId,    │     │ (get capacity,     │
                 │  bookingPolicies)  │     │  participants)     │
                 └────────────────────┘     └────────────────────┘
                          │                          │
                          └──────────┬───────────────┘
                                     ▼
                          Client-side calculation of:
                          - isFullyBooked
                          - isTooEarlyToBook
                          - isTooLateToBook
                          - isBookable
```

### Current Implementation Issues

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              ISSUES WITH CURRENT IMPLEMENTATION                      │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ISSUE 1: scheduleId Not Populated                                                  │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━                                                  │
│  • Event object has scheduleId available                                            │
│  • TimeSlot.scheduleId exists in proto (field 14)                                  │
│  • But mapToTimeSlots() does NOT populate it                                        │
│                                                                                     │
│  ISSUE 2: Course vs Class Not Distinguished                                         │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━                                          │
│  • Client cannot tell if TimeSlot is for CLASS or COURSE                           │
│  • Event type available but not exposed                                             │
│                                                                                     │
│  ISSUE 3: Course Booking Requires Multiple API Calls                               │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━                                │
│  • Services V2 Query (bookingPolicies)                                              │
│  • Schedule Server (capacity, participants)                                         │
│  • Calendar Events (sessions)                                                       │
│  • Client calculates availability                                                   │
│                                                                                     │
│  ISSUE 4: Single Cursor for Multiple Services                                       │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━                                       │
│  • Pagination is time-based, not service-based                                      │
│  • Can't paginate per-service independently                                         │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Key Differences: Classes vs Courses

| Aspect | Class | Course |
|--------|-------|--------|
| Booking target | Individual event (session) | Entire schedule |
| Capacity scope | Per-event | Per-schedule (propagated to all events) |
| Participation | Event-level | Schedule-level |
| Booking entity | `eventId` | `scheduleId` |
| Sessions | Independent (can book any) | All-or-nothing |
| BookingPolicies | Applied per session | Applied to first session |

---

## Open Questions & Answers

### Q1: Are Booking Policies per Service?

**Answer: YES**

Booking policies are defined at the **service level**. All time slots of the same service share the same booking policy.

```
Service (serviceId)
    └── BookingPolicy
            ├── limitEarlyBooking (tooEarlyToBook)
            ├── limitLateBooking (earliestBookingDate / tooLateToBook)  
            └── bookAfterStartPolicy (course-specific)
```

**Implication:** If one slot of a service is `tooEarlyToBook`, all earlier slots of the same service are also `tooEarlyToBook`.

---

### Q2: What is `bookingPolicyViolations` in Request vs Response?

**Answer: Request = FILTER, Response = CALCULATED VALUES**

| Context | Purpose | Example |
|---------|---------|---------|
| **In Request** | Filter slots by violation status | `{tooEarlyToBook: true}` → return only slots that are too early |
| **In Response** | Calculated violation status per slot | Shows why slot is not bookable |

**Current Implementation Flow:**

```
Request                          Backend Processing                      Response
────────────────────────────────────────────────────────────────────────────────────

bookingPolicyViolations: {    →  1. Fetch policies via SPI          →  bookingPolicyViolations: {
  tooEarlyToBook: true           2. Calculate violations per slot        tooEarlyToBook: true,
}                                3. Filter slots matching request        tooLateToBook: false,
                                 4. Return matching slots                 bookOnlineDisabled: false
                                                                       }
```

---

### Q3: Do We Currently Support Multiple serviceIDs?

**Answer: YES**

The current implementation fully supports multiple service IDs:

```
Request: {
  serviceIds: ["service-1", "service-2", "service-3"]
}

Backend Processing:
├── Query events for ALL services in single Calendar query
├── Fetch policies per service (batched by 8)
└── Return mixed TimeSlots ordered by start date

Response: {
  timeSlots: [
    { serviceId: "service-1", localStartDate: "2026-01-10T10:00" },
    { serviceId: "service-2", localStartDate: "2026-01-10T14:00" },
    { serviceId: "service-1", localStartDate: "2026-01-11T10:00" },
    ...
  ]
}
```

---

### Q4: How Does Pagination Work with Multiple Services?

**Answer: Single Cursor, Time-Based Pagination**

```
┌─────────────────────────────────────────────────────────────────┐
│                    PAGINATION MODEL                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  • Single cursor for entire response                            │
│  • Cursor encodes: fromLocalDate = lastSlot.startDate + 1sec   │
│  • ALL services paginate together by time                       │
│  • Cannot paginate per-service independently                    │
│                                                                 │
│  Page 1 (limit=3):                                              │
│  ┌─────────────────────────────────────────────┐               │
│  │ service-1 │ 2026-01-10 10:00 │             │               │
│  │ service-2 │ 2026-01-10 14:00 │             │               │
│  │ service-1 │ 2026-01-11 10:00 │ ← cursor    │               │
│  └─────────────────────────────────────────────┘               │
│                                                                 │
│  Page 2 (cursor = "2026-01-11T10:00:01"):                      │
│  ┌─────────────────────────────────────────────┐               │
│  │ service-2 │ 2026-01-11 14:00 │             │               │
│  │ service-1 │ 2026-01-12 10:00 │             │               │
│  │ ...                                        │               │
│  └─────────────────────────────────────────────┘               │
│                                                                 │
│  ⚠️ LIMITATION: If service-1 has 1000 slots and service-2     │
│     has 10 slots, you can't get all of service-2 without       │
│     paging through service-1 slots.                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Q5: Do We Enforce Policies Today for Classes?

**Answer: YES - Policies are Applied per TimeSlot**

```
Current Policy Enforcement Flow:
────────────────────────────────

1. Fetch booking policies from SPI
   └── bookingPolicyProvider.listBookingPolicies(serviceIds)

2. For each event, calculate violations
   └── TimeSlotPolicyViolationsCalculator(policy)
         .calculateBookingPolicyViolations(zonedStart)

3. Apply to TimeSlot
   ├── tooEarlyToBook = (now + earliestBookingInMinutes) > slotStart
   ├── tooLateToBook = (now + latestBookingInMinutes) > slotStart  
   └── bookOnlineDisabled = policy.onlineBookingDisabled

4. Calculate bookable
   └── bookable = hasCapacity && !tooEarly && !tooLate && !disabled

5. Optional: Filter by request.bookingPolicyViolations
   └── filterByBookingPolicyViolationsIfEnabled(slot, request.bookingPolicyViolations)
```

---

## Booking Policies Reference

### Policy Types and Their Effects

| Policy | Field in Service | Effect on TimeSlot | Applies To |
|--------|------------------|-------------------|------------|
| **Limit Early Booking** | `bookingPolicy.limitEarlyBookingPolicy.earliestBookingInMinutes` | `tooEarlyToBook = true` | All slots too far in future |
| **Limit Late Booking** | `bookingPolicy.limitLateBookingPolicy.latestBookingInMinutes` | `tooLateToBook = true` + `earliestBookingDate` | All slots too close to start |
| **Book After Start** | `bookingPolicy.bookAfterStartPolicy.enabled` | Allows booking started courses | Courses only |
| **Book Online Disabled** | `onlineBooking.enabled = false` | `bookOnlineDisabled = true` | All slots |

---

## Optimization Matrix: Request Fields × Policy Flags

### How Optimizations Can Be Applied

| Request Field | Current Use | Optimization Potential for Func. 1 | Optimization Potential for Func. 2 |
|---------------|-------------|-------------------------------------|-------------------------------------|
| `fromLocalDate` | Filter events ≥ date | For courses: if first session < fromDate and !bookAfterStart → unavailable | Query only from this date |
| `toLocalDate` | Filter events ≤ date | For courses: if last session < now → unavailable | Query only until this date |
| `serviceIds` | Filter by service | Determines which schedules to check | Filters events by service |
| `includeNonBookable` | Include/exclude non-bookable slots | If false + no bookable slot exists → unavailable early | Filter out non-bookable |
| `minBookableCapacity` | Filter by capacity ≥ N | If no event meets capacity → unavailable | Filter in Calendar query |

### Policy-Based Optimizations

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                          OPTIMIZATION DECISION TREE                                   │
├──────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  FOR COURSES (Schedule-Level Booking):                                               │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━                                                │
│                                                                                      │
│  STEP 1: Check if course ended                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐                │
│  │ IF lastSessionEnd < now                                         │                │
│  │   → UNAVAILABLE (courseEnded = true)                            │                │
│  │   → No need to fetch events                                     │                │
│  └─────────────────────────────────────────────────────────────────┘                │
│                                                                                      │
│  STEP 2: Check bookAfterStart policy                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐                │
│  │ IF firstSessionStart < now AND !bookAfterStartPolicy.enabled    │                │
│  │   → UNAVAILABLE (tooLateToBook = true)                          │                │
│  │   → No need to fetch events                                     │                │
│  └─────────────────────────────────────────────────────────────────┘                │
│                                                                                      │
│  STEP 3: Check capacity                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐                │
│  │ IF schedule.capacity - schedule.totalParticipants <= 0          │                │
│  │   → UNAVAILABLE (fullyBooked = true)                            │                │
│  │   → Can still return events for display                         │                │
│  └─────────────────────────────────────────────────────────────────┘                │
│                                                                                      │
│  STEP 4: Check tooEarlyToBook                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐                │
│  │ IF now + earliestBookingInMinutes < firstSessionStart           │                │
│  │   → UNAVAILABLE (tooEarlyToBook = true)                         │                │
│  │   → Can still return events for display                         │                │
│  └─────────────────────────────────────────────────────────────────┘                │
│                                                                                      │
│  FOR CLASSES (Event-Level Booking):                                                  │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━                                                  │
│                                                                                      │
│  STEP 1: Since policies are per-service, once we find one tooEarlyToBook,          │
│          all EARLIER slots are also tooEarlyToBook                                  │
│                                                                                      │
│  STEP 2: Similarly, once we find one tooLateToBook,                                 │
│          all LATER slots are also tooLateToBook (not yet implemented)               │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

### Optimization Combinations Table

| Scenario | tooEarlyToBook | tooLateToBook | bookAfterStart | Func. 1 Optimization | Func. 2 Optimization |
|----------|----------------|---------------|----------------|----------------------|----------------------|
| Course ended | N/A | N/A | N/A | Skip all queries, return `courseEnded: true` | Still fetch events for display |
| Course started, no bookAfterStart | false | true | false | Return unavailable immediately | Fetch events, mark as non-bookable |
| Course started, bookAfterStart enabled | false | false | true | Check capacity + remaining sessions | Fetch events, course still bookable |
| Course not started, too early | true | false | N/A | Return `tooEarlyToBook` + `earliestBookingDate` | Fetch events, mark as non-bookable |
| Course not started, within window | false | false | N/A | Check capacity only | Fetch events normally |
| Fully booked course | N/A | N/A | N/A | Return `fullyBooked: true` | Fetch events, mark as non-bookable |

---

## Current Implementation Flow

### Current Sequence Diagram

```
┌───────────────────────────────────────────────────────────────────────────────────────┐
│                        CURRENT ListEventTimeSlots FLOW                                 │
└───────────────────────────────────────────────────────────────────────────────────────┘

    Client                EventTimeSlots           SPI                Calendar V3
       │                       │                    │                     │
       │  ListEventTimeSlots   │                    │                     │
       │ ─────────────────────►│                    │                     │
       │                       │                    │                     │
       │                       │ getProviderConfig  │                     │
       │                       │───────────────────►│                     │
       │                       │◄───────────────────│                     │
       │                       │                    │                     │
       │                       │                    │    queryEvents      │
       │                       │────────────────────────────────────────►│
       │                       │◄────────────────────────────────────────│
       │                       │                    │                     │
       │                       │ listBookingPolicies│                     │
       │                       │───────────────────►│                     │
       │                       │◄───────────────────│                     │
       │                       │                    │                     │
       │                       │ [For each event]   │                     │
       │                       │ Calculate:         │                     │
       │                       │ - policyViolations │                     │
       │                       │ - bookable         │                     │
       │                       │ - capacity         │                     │
       │                       │                    │                     │
       │  TimeSlots[]          │                    │                     │
       │◄──────────────────────│                    │                     │
       │                       │                    │                     │
       │  ⚠️ scheduleId NOT populated!             │                     │
       │  ⚠️ No schedule-level availability!       │                     │
```

### Current Decision Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           CURRENT DECISION FLOW                                      │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  START                                                                              │
│    │                                                                                │
│    ▼                                                                                │
│  ┌─────────────────────────────────────┐                                           │
│  │ Validate request                    │                                           │
│  │ - fromLocalDate required            │                                           │
│  │ - maxSlotsPerDay needs toLocalDate  │                                           │
│  └─────────────────┬───────────────────┘                                           │
│                    │                                                                │
│                    ▼                                                                │
│  ┌─────────────────────────────────────┐                                           │
│  │ Get SPI configuration               │                                           │
│  │ - eventFilter (type: CLASS)         │                                           │
│  │ - eventServiceIdField               │                                           │
│  └─────────────────┬───────────────────┘                                           │
│                    │                                                                │
│                    ▼                                                                │
│  ┌─────────────────────────────────────┐                                           │
│  │ Build query filter                  │◄───────────────────────────┐              │
│  │ - providerBaseFilter                │                            │              │
│  │ + minBookableCapacity               │                            │              │
│  │ + eventFilter (from request)        │                            │              │
│  │ + serviceIds filter                 │                            │              │
│  └─────────────────┬───────────────────┘                            │              │
│                    │                                                 │              │
│                    ▼                                                 │              │
│  ┌─────────────────────────────────────┐                            │              │
│  │ Query Calendar V3 Events            │                            │              │
│  │ (with pagination)                   │                            │              │
│  └─────────────────┬───────────────────┘                            │              │
│                    │                                                 │              │
│                    ▼                                                 │              │
│  ┌─────────────────────────────────────┐                            │              │
│  │ Filter hidden services (optional)   │                            │              │
│  └─────────────────┬───────────────────┘                            │              │
│                    │                                                 │              │
│                    ▼                                                 │              │
│  ┌─────────────────────────────────────┐                            │              │
│  │ Fetch booking policies              │                            │              │
│  │ (batched by 8 services)             │                            │              │
│  └─────────────────┬───────────────────┘                            │              │
│                    │                                                 │              │
│                    ▼                                                 │              │
│  ┌─────────────────────────────────────┐                            │              │
│  │ For each event:                     │                            │              │
│  │ - Calculate policy violations       │                            │              │
│  │ - Calculate bookable                │                            │              │
│  │ - Map to TimeSlot                   │                            │              │
│  └─────────────────┬───────────────────┘                            │              │
│                    │                                                 │              │
│                    ▼                                                 │              │
│  ┌─────────────────────────────────────┐                            │              │
│  │ Apply filters:                      │                            │              │
│  │ - includeNonBookable                │                            │              │
│  │ - bookingPolicyViolations filter    │                            │              │
│  │ - maxSlotsPerDay                    │                            │              │
│  └─────────────────┬───────────────────┘                            │              │
│                    │                                                 │              │
│                    ▼                                                 │              │
│  ┌─────────────────────────────────────┐      ┌─────────────────┐   │              │
│  │ Enough slots?                       │─NO──►│ More to fetch?  │───┘              │
│  └─────────────────┬───────────────────┘      └─────────────────┘                  │
│                    │ YES                                                            │
│                    ▼                                                                │
│  ┌─────────────────────────────────────┐                                           │
│  │ Build response with cursor          │                                           │
│  └─────────────────┬───────────────────┘                                           │
│                    │                                                                │
│                    ▼                                                                │
│                  END                                                                │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

# Proposal A: Minimal Enhancement

## Overview

**Philosophy:** Make the smallest possible changes to enable course support without restructuring the API.

**Key Changes:**
1. Populate the existing `scheduleId` field in TimeSlot (currently empty - one line fix!)
2. Include COURSE events in the query (update SPI filter)

**Does NOT include:**
- `isCourse` flag (client can determine from serviceId)
- Server-side schedule availability calculation
- Any new fields or messages

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         PROPOSAL A: MINIMAL ENHANCEMENT                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌───────────────┐                                                                  │
│  │    Client     │                                                                  │
│  │               │                                                                  │
│  │ ┌───────────┐ │     ┌─────────────────────────────────────────┐                 │
│  │ │ Services  │─┼────►│           Services V2 API                │                 │
│  │ │ V2 Query  │ │     │  (get serviceId, bookingPolicies, type) │                 │
│  │ └───────────┘ │     └─────────────────────────────────────────┘                 │
│  │       │       │                                                                  │
│  │       ▼       │                                                                  │
│  │ ┌───────────┐ │     ┌─────────────────────────────────────────┐                 │
│  │ │ List      │─┼────►│         ListEventTimeSlots               │                 │
│  │ │ TimeSlots │ │     │  (now returns scheduleId!)               │                 │
│  │ └───────────┘ │     └──────────────────┬──────────────────────┘                 │
│  │       │       │                        │                                         │
│  │       ▼       │                        ▼                                         │
│  │ ┌───────────┐ │     ┌─────────────────────────────────────────┐                 │
│  │ │ Client    │ │     │           Calendar V3 Events             │                 │
│  │ │ Calculate │ │     │  (now queries CLASS + COURSE)            │                 │
│  │ │ Avail.    │ │     └─────────────────────────────────────────┘                 │
│  │ └───────────┘ │                                                                  │
│  │               │                                                                  │
│  └───────────────┘                                                                  │
│                                                                                     │
│  Changes: ● scheduleId now populated in TimeSlot                                    │
│           ● Filter includes COURSE events                                           │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## Sequence Diagram

```
    Client             Services V2       EventTimeSlots        Calendar V3
       │                    │                  │                    │
       │ Query Service      │                  │                    │
       │───────────────────►│                  │                    │
       │◄───────────────────│                  │                    │
       │ {type, policies}   │                  │                    │
       │                    │                  │                    │
       │ ListEventTimeSlots(serviceId)         │                    │
       │──────────────────────────────────────►│                    │
       │                    │                  │                    │
       │                    │                  │ Query CLASS+COURSE │
       │                    │                  │───────────────────►│
       │                    │                  │◄───────────────────│
       │                    │                  │                    │
       │ TimeSlots[] with scheduleId!          │                    │
       │◄──────────────────────────────────────│                    │
       │                    │                  │                    │
       │ Calculate schedule │                  │                    │
       │ availability       │                  │                    │
       │ (CLIENT-SIDE)      │                  │                    │
       │                    │                  │                    │
```

## Decision Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                      PROPOSAL A: DECISION FLOW (CLIENT-SIDE)                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  [Same as current backend flow for ListEventTimeSlots]                              │
│                                                                                     │
│  Plus one line added in mapToTimeSlots():                                           │
│    TimeSlot(..., scheduleId = event.scheduleId)                                     │
│                                                                                     │
│  CLIENT must still calculate schedule availability:                                  │
│                                                                                     │
│                  ┌───────────────────┐                                              │
│                  │ Get TimeSlots     │                                              │
│                  └─────────┬─────────┘                                              │
│                            │                                                        │
│                            ▼                                                        │
│         ┌──────────────────────────────────────┐                                    │
│         │ Is service type COURSE?              │                                    │
│         │ (from Services V2 query)             │                                    │
│         └───────────────┬──────────────────────┘                                    │
│                         │                                                           │
│              ┌──────────┴──────────┐                                               │
│              ▼                     ▼                                               │
│         ┌────────┐            ┌────────┐                                           │
│         │ CLASS  │            │ COURSE │                                           │
│         └────┬───┘            └────┬───┘                                           │
│              │                     │                                               │
│              ▼                     ▼                                               │
│    ┌─────────────────┐    ┌─────────────────┐                                      │
│    │ Each slot is    │    │ Client calcs:   │                                      │
│    │ independent     │    │ - fullyBooked   │                                      │
│    │ (use as-is)     │    │ - tooEarly      │                                      │
│    └─────────────────┘    │ - tooLate       │                                      │
│                           │ - ended         │                                      │
│                           └─────────────────┘                                      │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## Class Diagram (Changes)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        PROPOSAL A: CLASS DIAGRAM CHANGES                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────┐                                           │
│  │           TimeSlot                  │                                           │
│  ├─────────────────────────────────────┤                                           │
│  │ - serviceId: String                 │                                           │
│  │ - localStartDate: String            │                                           │
│  │ - localEndDate: String              │                                           │
│  │ - bookable: Boolean                 │                                           │
│  │ - totalCapacity: Int                │                                           │
│  │ - remainingCapacity: Int            │                                           │
│  │ - bookableCapacity: Int             │                                           │
│  │ - eventInfo: EventInfo              │                                           │
│  │ - bookingPolicyViolations: ...      │                                           │
│  │ - nonBookableReasons: ...           │                                           │
│  │ - scheduleId: String ◄──────────── NOW POPULATED (was null)                     │
│  └─────────────────────────────────────┘                                           │
│                                                                                     │
│  ┌─────────────────────────────────────┐                                           │
│  │    EventTimeSlotsProviderConfig     │                                           │
│  ├─────────────────────────────────────┤                                           │
│  │ - eventFilter: Struct ◄──────────── CHANGED: {"type": {"$in": ["CLASS","COURSE"]}}
│  │ - eventServiceIdField: String       │                                           │
│  └─────────────────────────────────────┘                                           │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## Pros & Cons

### ✅ Pros

| Benefit | Description |
|---------|-------------|
| **Minimal code changes** | ~5 lines of code change |
| **Low risk** | No restructuring, easy rollback |
| **Fast implementation** | Hours, not days |
| **Backwards compatible** | No breaking changes |
| **Enables course support** | scheduleId now available for booking |

### ❌ Cons

| Drawback | Description |
|----------|-------------|
| **Client calculates** | Availability logic remains on frontend |
| **No Functionality 1** | Cannot get quick schedule availability |
| **No optimizations** | Still fetches all events even if unavailable |
| **Duplicate data** | Course info repeated per session |

---

# Proposal B: Unified Schedule Availability

## Overview

**Philosophy:** Add schedule-level availability at the response level to support both classes and courses with server-side calculations.

**Key Changes:**
1. Populate `scheduleId` in TimeSlot
2. Add `scheduleInfoByServiceId` map to response (works for both classes and courses)
3. Add optional `provideScheduleAvailabilityOnly` flag for quick availability check
4. Server calculates schedule-level availability with optimizations

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    PROPOSAL B: UNIFIED SCHEDULE AVAILABILITY                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌───────────────┐                                                                  │
│  │    Client     │                                                                  │
│  │               │                                                                  │
│  │ ┌───────────┐ │     ┌─────────────────────────────────────────┐                 │
│  │ │ List      │─┼────►│         ListEventTimeSlots               │                 │
│  │ │ TimeSlots │ │     │  (enhanced with schedule info)           │                 │
│  │ └───────────┘ │     └──────────────────┬──────────────────────┘                 │
│  │       │       │                        │                                         │
│  │       │       │         ┌──────────────┼──────────────┐                         │
│  │       │       │         ▼              ▼              ▼                         │
│  │       │       │    ┌─────────┐   ┌──────────┐   ┌──────────┐                    │
│  │       │       │    │ Events  │   │ Schedules│   │ Policies │                    │
│  │       │       │    │   API   │   │   API    │   │   SPI    │                    │
│  │       │       │    └─────────┘   └──────────┘   └──────────┘                    │
│  │       │       │                        │                                         │
│  │       ▼       │                        ▼                                         │
│  │ ┌───────────┐ │     ┌─────────────────────────────────────────┐                 │
│  │ │ Response  │◄┼─────│  ListEventTimeSlotsResponse             │                 │
│  │ │           │ │     │  + scheduleInfoByServiceId              │                 │
│  │ │ Ready to  │ │     │  (schedule availability pre-calculated) │                 │
│  │ │ use!      │ │     └─────────────────────────────────────────┘                 │
│  │ └───────────┘ │                                                                  │
│  │               │                                                                  │
│  └───────────────┘                                                                  │
│                                                                                     │
│  New Features:                                                                      │
│  ● scheduleInfoByServiceId map in response (for classes AND courses)               │
│  ● provideScheduleAvailabilityOnly flag for quick check                            │
│  ● Server-side availability calculation with optimizations                          │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## Sequence Diagram

### Normal Flow (Functionality 2: Full TimeSlots)

```
    Client              EventTimeSlots         Schedules API       Calendar V3      PolicySPI
       │                      │                     │                  │                │
       │ ListEventTimeSlots   │                     │                  │                │
       │ {serviceIds}         │                     │                  │                │
       │─────────────────────►│                     │                  │                │
       │                      │                     │                  │                │
       │                      │─────────────────────────────────────────────────────────┤
       │                      │              PARALLEL EXECUTION                          │
       │                      │─────────────────────────────────────────────────────────┤
       │                      │                     │                  │                │
       │                      │ listSchedules       │                  │                │
       │                      │────────────────────►│                  │                │
       │                      │                     │                  │                │
       │                      │                     │  queryEvents     │                │
       │                      │─────────────────────────────────────►│                │
       │                      │                     │                  │                │
       │                      │                     │                  │ listPolicies   │
       │                      │────────────────────────────────────────────────────────►│
       │                      │                     │                  │                │
       │                      │◄────────────────────┼──────────────────┼────────────────┤
       │                      │                     │                  │                │
       │                      │ Build:              │                  │                │
       │                      │ - TimeSlots[]       │                  │                │
       │                      │ - scheduleInfo      │                  │                │
       │                      │                     │                  │                │
       │ Response:            │                     │                  │                │
       │ - timeSlots[]        │                     │                  │                │
       │ - scheduleInfoByServiceId                  │                  │                │
       │◄─────────────────────│                     │                  │                │
```

### Quick Check Flow (Functionality 1: Schedule Availability Only)

```
    Client              EventTimeSlots         Schedules API       Calendar V3      PolicySPI
       │                      │                     │                  │                │
       │ ListEventTimeSlots   │                     │                  │                │
       │ {serviceIds,         │                     │                  │                │
       │  provideSchedule     │                     │                  │                │
       │  AvailabilityOnly}   │                     │                  │                │
       │─────────────────────►│                     │                  │                │
       │                      │                     │                  │                │
       │                      │ listSchedules       │                  │                │
       │                      │────────────────────►│                  │                │
       │                      │◄────────────────────│                  │                │
       │                      │                     │                  │                │
       │                      │ listPolicies        │                  │                │
       │                      │────────────────────────────────────────────────────────►│
       │                      │◄────────────────────────────────────────────────────────│
       │                      │                     │                  │                │
       │                      │ OPTIMIZATION:       │                  │                │
       │                      │ Check if can skip   │                  │                │
       │                      │ event query         │                  │                │
       │                      │                     │                  │                │
       │                      │─────┐               │                  │                │
       │                      │     │ If course ended OR                │                │
       │                      │     │ (started AND !bookAfterStart)     │                │
       │                      │     │ → Skip event query!               │                │
       │                      │◄────┘               │                  │                │
       │                      │                     │                  │                │
       │                      │ ELSE: Query first   │                  │                │
       │                      │ bookable event      │                  │                │
       │                      │─────────────────────────────────────►│                │
       │                      │◄────────────────────────────────────────│                │
       │                      │                     │                  │                │
       │ Response:            │                     │                  │                │
       │ - timeSlots: [first] │ (or empty)          │                  │                │
       │ - scheduleInfoByServiceId                  │                  │                │
       │◄─────────────────────│                     │                  │                │
```

## Decision Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        PROPOSAL B: DECISION FLOW                                     │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  START                                                                              │
│    │                                                                                │
│    ▼                                                                                │
│  ┌─────────────────────────────────────┐                                           │
│  │ Validate request                    │                                           │
│  └─────────────────┬───────────────────┘                                           │
│                    │                                                                │
│                    ▼                                                                │
│  ┌─────────────────────────────────────┐                                           │
│  │ PARALLEL: Fetch schedules + policies│                                           │
│  └─────────────────┬───────────────────┘                                           │
│                    │                                                                │
│                    ▼                                                                │
│  ┌─────────────────────────────────────┐                                           │
│  │ For each schedule:                  │                                           │
│  │ Calculate schedule availability     │                                           │
│  └─────────────────┬───────────────────┘                                           │
│                    │                                                                │
│        ┌───────────┴───────────┐                                                   │
│        ▼                       ▼                                                   │
│  ┌──────────────────┐   ┌──────────────────┐                                       │
│  │ provideSchedule  │   │ Full query       │                                       │
│  │ AvailabilityOnly │   │ (default)        │                                       │
│  │ = true           │   │                  │                                       │
│  └────────┬─────────┘   └────────┬─────────┘                                       │
│           │                      │                                                  │
│           ▼                      ▼                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐               │
│  │                    OPTIMIZATION CHECK                           │               │
│  ├─────────────────────────────────────────────────────────────────┤               │
│  │                                                                 │               │
│  │  ┌─────────────────────────────────────────┐                   │               │
│  │  │ Course ended?                           │                   │               │
│  │  │ (lastSessionEnd < now)                  │                   │               │
│  │  └──────────────────┬──────────────────────┘                   │               │
│  │                     │ YES                                       │               │
│  │                     ├─────► Mark unavailable, skip events      │               │
│  │                     │                                           │               │
│  │                     │ NO                                        │               │
│  │                     ▼                                           │               │
│  │  ┌─────────────────────────────────────────┐                   │               │
│  │  │ Course started AND !bookAfterStart?     │                   │               │
│  │  └──────────────────┬──────────────────────┘                   │               │
│  │                     │ YES                                       │               │
│  │                     ├─────► Mark unavailable, skip events      │               │
│  │                     │                                           │               │
│  │                     │ NO                                        │               │
│  │                     ▼                                           │               │
│  │  ┌─────────────────────────────────────────┐                   │               │
│  │  │ Fully booked?                           │                   │               │
│  │  │ (capacity - participants <= 0)          │                   │               │
│  │  └──────────────────┬──────────────────────┘                   │               │
│  │                     │ YES                                       │               │
│  │                     ├─────► Mark unavailable                   │               │
│  │                     │                                           │               │
│  │                     │ NO                                        │               │
│  │                     ▼                                           │               │
│  │  ┌─────────────────────────────────────────┐                   │               │
│  │  │ Too early to book?                      │                   │               │
│  │  └──────────────────┬──────────────────────┘                   │               │
│  │                     │ YES                                       │               │
│  │                     ├─────► Mark unavailable + earliestDate    │               │
│  │                     │                                           │               │
│  │                     │ NO                                        │               │
│  │                     ▼                                           │               │
│  │                   AVAILABLE                                     │               │
│  │                                                                 │               │
│  └─────────────────────────────────────────────────────────────────┘               │
│                    │                                                                │
│        ┌───────────┴───────────┐                                                   │
│        ▼                       ▼                                                   │
│  ┌──────────────────┐   ┌──────────────────┐                                       │
│  │ provideSchedule  │   │ Full query:      │                                       │
│  │ AvailabilityOnly │   │ Query all events │                                       │
│  │ = true:          │   │ Apply policies   │                                       │
│  │                  │   │ Map to TimeSlots │                                       │
│  │ Query first      │   │                  │                                       │
│  │ bookable event   │   │                  │                                       │
│  │ OR return empty  │   │                  │                                       │
│  └────────┬─────────┘   └────────┬─────────┘                                       │
│           │                      │                                                  │
│           └──────────┬───────────┘                                                 │
│                      ▼                                                              │
│  ┌─────────────────────────────────────┐                                           │
│  │ Build response:                     │                                           │
│  │ - timeSlots[]                       │                                           │
│  │ - scheduleInfoByServiceId{}         │                                           │
│  └─────────────────┬───────────────────┘                                           │
│                    │                                                                │
│                    ▼                                                                │
│                  END                                                                │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## Class Diagram (Changes)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        PROPOSAL B: CLASS DIAGRAM CHANGES                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────┐                                           │
│  │    ListEventTimeSlotsRequest        │                                           │
│  ├─────────────────────────────────────┤                                           │
│  │ - providerId: String                │                                           │
│  │ - fromLocalDate: String             │                                           │
│  │ - toLocalDate: String               │                                           │
│  │ - timeZone: String                  │                                           │
│  │ - serviceIds: List<String>          │                                           │
│  │ - includeNonBookable: Boolean       │                                           │
│  │ - minBookableCapacity: Int          │                                           │
│  │ - eventFilter: Struct               │                                           │
│  │ - maxSlotsPerDay: Int               │                                           │
│  │ - cursorPaging: CursorPaging        │                                           │
│  │ - bookingPolicyViolations: ...      │                                           │
│  │ + provideScheduleAvailabilityOnly: Boolean ◄───────── NEW                       │
│  └─────────────────────────────────────┘                                           │
│                                                                                     │
│  ┌─────────────────────────────────────┐                                           │
│  │   ListEventTimeSlotsResponse        │                                           │
│  ├─────────────────────────────────────┤                                           │
│  │ - timeSlots: List<TimeSlot>         │                                           │
│  │ - timeZone: String                  │                                           │
│  │ - pagingMetadata: CursorPagingMeta  │                                           │
│  │ + scheduleInfoByServiceId: Map<String, ScheduleInfo> ◄──────── NEW              │
│  └─────────────────────────────────────┘                                           │
│                                                                                     │
│  ┌─────────────────────────────────────┐         ┌───────────────────────────────┐ │
│  │          ScheduleInfo               │◄────────│ NEW MESSAGE                   │ │
│  ├─────────────────────────────────────┤         │ (works for classes & courses) │ │
│  │ + scheduleId: String                │         └───────────────────────────────┘ │
│  │ + isAvailable: Boolean              │                                           │
│  │ + totalCapacity: Int                │                                           │
│  │ + remainingCapacity: Int            │                                           │
│  │ + firstSessionStart: String         │                                           │
│  │ + lastSessionEnd: String            │                                           │
│  │ + totalSessions: Int                │                                           │
│  │ + remainingSessions: Int            │                                           │
│  │ + startedAndBookable: Boolean       │                                           │
│  │ + nonBookableReasons: ScheduleNonBookableReasons                                │
│  └─────────────────────────────────────┘                                           │
│                     │                                                               │
│                     ▼                                                               │
│  ┌─────────────────────────────────────┐                                           │
│  │   ScheduleNonBookableReasons        │◄─────────── NEW MESSAGE                   │
│  ├─────────────────────────────────────┤                                           │
│  │ + fullyBooked: Boolean              │                                           │
│  │ + tooEarlyToBook: Boolean           │                                           │
│  │ + earliestBookingDate: Timestamp    │                                           │
│  │ + tooLateToBook: Boolean            │                                           │
│  │ + bookOnlineDisabled: Boolean       │                                           │
│  │ + scheduleEnded: Boolean            │                                           │
│  └─────────────────────────────────────┘                                           │
│                                                                                     │
│  ┌─────────────────────────────────────┐                                           │
│  │           TimeSlot                  │                                           │
│  ├─────────────────────────────────────┤                                           │
│  │ - serviceId: String                 │                                           │
│  │ - ... (existing fields)             │                                           │
│  │ - scheduleId: String ◄──────────── NOW POPULATED                                │
│  └─────────────────────────────────────┘                                           │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## Pros & Cons

### ✅ Pros

| Benefit | Description |
|---------|-------------|
| **Server-side calculation** | `isAvailable` computed on backend |
| **Supports both functionalities** | Quick check + full query |
| **Unified for classes AND courses** | Same response structure |
| **No data duplication** | ScheduleInfo at response level |
| **Performance optimizations** | Can skip event queries when unavailable |
| **Backwards compatible** | New fields are optional |

### ❌ Cons

| Drawback | Description |
|----------|-------------|
| **Additional dependency** | Needs SchedulesAdapter |
| **More complex backend** | ~200 lines of new code |
| **Testing complexity** | More scenarios to test |
| **Migration needed** | Frontend must adopt new response fields |

---

# Proposal C: Dedicated Course API

## Overview

**Philosophy:** Create a separate endpoint specifically for course availability, keeping the existing API unchanged.

**Key Changes:**
1. New endpoint: `/v2/time-slots/course` or new RPC `ListCourseTimeSlots`
2. Course-centric response structure
3. Existing `ListEventTimeSlots` remains classes-only

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                      PROPOSAL C: DEDICATED COURSE API                                │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌───────────────┐                                                                  │
│  │    Client     │                                                                  │
│  │               │                                                                  │
│  │ ┌───────────┐ │                                                                  │
│  │ │ Is Course?│ │                                                                  │
│  │ └─────┬─────┘ │                                                                  │
│  │       │       │                                                                  │
│  │   ┌───┴───┐   │                                                                  │
│  │   ▼       ▼   │                                                                  │
│  │ ┌───┐   ┌───┐ │     ┌─────────────────────────────────────────┐                 │
│  │ │NO │   │YES│─┼────►│       ListCourseTimeSlots (NEW)         │                 │
│  │ └─┬─┘   └───┘ │     │  (course-centric, schedule-level)       │                 │
│  │   │           │     └─────────────────────────────────────────┘                 │
│  │   │           │                                                                  │
│  │   │           │     ┌─────────────────────────────────────────┐                 │
│  │   └───────────┼────►│       ListEventTimeSlots (unchanged)    │                 │
│  │               │     │  (classes only, event-level)            │                 │
│  │               │     └─────────────────────────────────────────┘                 │
│  │               │                                                                  │
│  └───────────────┘                                                                  │
│                                                                                     │
│  Two Separate APIs:                                                                 │
│  ● ListEventTimeSlots → Classes (unchanged)                                         │
│  ● ListCourseTimeSlots → Courses (NEW)                                              │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## Sequence Diagram

```
    Client              CourseTimeSlots        Schedules API       Calendar V3      PolicySPI
       │                      │                     │                  │                │
       │ ListCourseTimeSlots  │                     │                  │                │
       │ {serviceIds}         │                     │                  │                │
       │─────────────────────►│                     │                  │                │
       │                      │                     │                  │                │
       │                      │ listSchedules       │                  │                │
       │                      │────────────────────►│                  │                │
       │                      │◄────────────────────│                  │                │
       │                      │                     │                  │                │
       │                      │ listPolicies        │                  │                │
       │                      │────────────────────────────────────────────────────────►│
       │                      │◄────────────────────────────────────────────────────────│
       │                      │                     │                  │                │
       │                      │ [Optional] querySessions              │                │
       │                      │─────────────────────────────────────►│                │
       │                      │◄────────────────────────────────────────│                │
       │                      │                     │                  │                │
       │ CourseAvailability[] │                     │                  │                │
       │ (course-centric)     │                     │                  │                │
       │◄─────────────────────│                     │                  │                │
```

## Decision Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        PROPOSAL C: DECISION FLOW                                     │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  CLIENT DECISION:                                                                   │
│  ━━━━━━━━━━━━━━━━                                                                   │
│                                                                                     │
│  ┌─────────────────────────────────────┐                                           │
│  │ Get service type from Services V2   │                                           │
│  └─────────────────┬───────────────────┘                                           │
│                    │                                                                │
│         ┌──────────┴──────────┐                                                    │
│         ▼                     ▼                                                    │
│  ┌────────────────┐    ┌────────────────┐                                          │
│  │ type = CLASS   │    │ type = COURSE  │                                          │
│  └───────┬────────┘    └───────┬────────┘                                          │
│          │                     │                                                    │
│          ▼                     ▼                                                    │
│  ┌────────────────┐    ┌────────────────┐                                          │
│  │ Call List      │    │ Call List      │                                          │
│  │ EventTimeSlots │    │ CourseTimeSlots│                                          │
│  └────────────────┘    └────────────────┘                                          │
│                                                                                     │
│  BACKEND FLOW (ListCourseTimeSlots):                                               │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━                                                │
│                                                                                     │
│  [Same optimization flow as Proposal B for course availability calculation]         │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## Class Diagram (Changes)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        PROPOSAL C: CLASS DIAGRAM CHANGES                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────┐                                           │
│  │   ListCourseTimeSlotsRequest        │◄─────────── NEW SERVICE                   │
│  ├─────────────────────────────────────┤                                           │
│  │ + serviceIds: List<String>          │                                           │
│  │ + fromLocalDate: String             │                                           │
│  │ + toLocalDate: String               │                                           │
│  │ + timeZone: String                  │                                           │
│  │ + includeSessions: Boolean          │                                           │
│  │ + cursorPaging: CursorPaging        │                                           │
│  └─────────────────────────────────────┘                                           │
│                                                                                     │
│  ┌─────────────────────────────────────┐                                           │
│  │   ListCourseTimeSlotsResponse       │◄─────────── NEW                           │
│  ├─────────────────────────────────────┤                                           │
│  │ + courses: List<CourseAvailability> │  ← Course-centric (not TimeSlots)         │
│  │ + timeZone: String                  │                                           │
│  │ + pagingMetadata: CursorPagingMeta  │                                           │
│  └─────────────────────────────────────┘                                           │
│                                                                                     │
│  ┌─────────────────────────────────────┐                                           │
│  │       CourseAvailability            │◄─────────── NEW                           │
│  ├─────────────────────────────────────┤                                           │
│  │ + serviceId: String                 │                                           │
│  │ + scheduleId: String                │                                           │
│  │ + bookable: Boolean                 │                                           │
│  │ + totalCapacity: Int                │                                           │
│  │ + remainingCapacity: Int            │                                           │
│  │ + firstSessionStart: String         │                                           │
│  │ + lastSessionEnd: String            │                                           │
│  │ + totalSessions: Int                │                                           │
│  │ + remainingSessions: Int            │                                           │
│  │ + sessions: List<CourseSession>     │  ← Optional, if includeSessions=true      │
│  │ + bookingPolicyViolations: ...      │                                           │
│  │ + nonBookableReasons: ...           │                                           │
│  └─────────────────────────────────────┘                                           │
│                                                                                     │
│  ┌─────────────────────────────────────┐                                           │
│  │         CourseSession               │◄─────────── NEW                           │
│  ├─────────────────────────────────────┤                                           │
│  │ + eventId: String                   │                                           │
│  │ + localStartDate: String            │                                           │
│  │ + localEndDate: String              │                                           │
│  │ + isPast: Boolean                   │                                           │
│  │ + isCancelled: Boolean              │                                           │
│  └─────────────────────────────────────┘                                           │
│                                                                                     │
│  [ListEventTimeSlots remains UNCHANGED - classes only]                              │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## Pros & Cons

### ✅ Pros

| Benefit | Description |
|---------|-------------|
| **Purpose-built** | Optimized for course use case |
| **Clean semantics** | Course-centric response |
| **Independent evolution** | Can add course features without affecting classes |
| **No breaking changes** | Existing API untouched |

### ❌ Cons

| Drawback | Description |
|----------|-------------|
| **API fragmentation** | Two endpoints for event-based availability |
| **Client complexity** | Must know which API to call |
| **Duplicate logic** | Some shared code with EventTimeSlots |
| **Maintenance cost** | Two APIs to maintain |
| **Redundant** | If Proposal B handles both, this is unnecessary |

---

# Comparison Matrix

## Feature Comparison

| Feature | Proposal A | Proposal B | Proposal C |
|---------|-----------|-----------|-----------|
| **scheduleId in response** | ✅ | ✅ | ✅ |
| **Functionality 1 (quick check)** | ❌ Client-side | ✅ Server-side | ✅ Server-side |
| **Functionality 2 (full query)** | ✅ | ✅ | ✅ |
| **Single unified API** | ✅ | ✅ | ❌ |
| **Works for classes** | ✅ | ✅ | ❌ (separate API) |
| **Works for courses** | ✅ | ✅ | ✅ |
| **Minimal changes** | ✅ | ❌ | ❌ |
| **Performance optimizations** | ❌ | ✅ | ✅ |
| **Clear semantics** | ⚠️ | ✅ | ✅ |

## Implementation Effort

| Aspect | Proposal A | Proposal B | Proposal C |
|--------|-----------|-----------|-----------|
| **New messages** | 0 | 2 | 4 |
| **Backend code** | ~5 lines | ~200 lines | ~400 lines |
| **New dependencies** | None | SchedulesAdapter | SchedulesAdapter |
| **Test cases** | ~5 | ~30 | ~50 |
| **Documentation** | Minimal | Moderate | Full API docs |
| **Estimated effort** | Hours | 1-2 weeks | 3-4 weeks |

## Risk Assessment

| Risk | Proposal A | Proposal B | Proposal C |
|------|-----------|-----------|-----------|
| **Breaking changes** | None | Low (additive) | Medium |
| **Performance regression** | None | Low | Low |
| **Integration issues** | Very Low | Medium | High |
| **Client migration** | Easy | Moderate | Hard |
| **Rollback difficulty** | Easy | Easy | Hard |

---

# Recommendation

## Primary Recommendation: Proposal B

**I recommend Proposal B (Unified Schedule Availability)** for the following reasons:

### 1. Achieves Both Functionalities

| Functionality | How Proposal B Delivers |
|---------------|-------------------------|
| **1. Quick availability check** | `provideScheduleAvailabilityOnly=true` + `scheduleInfoByServiceId` |
| **2. Full TimeSlots query** | Standard flow with `scheduleInfoByServiceId` included |

### 2. Works for Both Service Types

- Classes: `scheduleInfoByServiceId` contains class schedule availability
- Courses: Same structure, with course-specific calculations

### 3. Enables Performance Optimizations

- Can skip event queries when unavailable
- Parallel execution of schedules + policies fetch
- Smart early termination for courses

### 4. Backwards Compatible

- All new fields are optional
- Existing clients continue to work
- Gradual migration possible

## Alternative Path

If time is critical:

1. **Week 1**: Implement Proposal A (populate scheduleId) - immediate value
2. **Week 3-4**: Evolve to Proposal B (add scheduleInfoByServiceId)

This provides incremental value while building toward the complete solution.
