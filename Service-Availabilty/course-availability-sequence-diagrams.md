# Course Availability - Sequence Diagrams

This document describes the frontend flows for checking course availability and fetching course sessions in the Wix Bookings Service Details Widget.

---

## Diagram 1: Checking if a Course is Available for Booking

This flow determines whether a course can be booked by the user, and what message to display if it cannot.

```mermaid
sequenceDiagram
    autonumber
    participant Client as Client (Widget)
    participant Controller as Controller<br/>(init-widgets.ts)
    participant BookingsApi as BookingsApi<br/>(BookingsApi.ts)
    participant PolicyMapper as PolicyMapper<br/>(booking-policy.mapper.ts)
    participant BodyVM as BodyViewModel<br/>(bodyViewModel.ts)
    participant ServicesAPI as Services V2 API<br/>(@wix/ambassador-bookings-services-v2)
    participant ScheduleAPI as Schedule Server API<br/>(@wix/ambassador-schedule-server)

    Client->>Controller: Page Load (serviceSlug)
    
    rect rgb(200, 220, 255)
        Note over Controller,ScheduleAPI: Step 1: Fetch Service & Schedule Data
        
        Controller->>BookingsApi: queryServicesBySlug(serviceSlug)
        BookingsApi->>ServicesAPI: HTTP: queryServices()
        ServicesAPI-->>BookingsApi: Service Response
        Note right of ServicesAPI: - type: COURSE<br/>- onlineBooking.enabled<br/>- bookingPolicy.*<br/>- schedule.firstSessionStart<br/>- schedule.lastSessionEnd
        BookingsApi-->>Controller: Service
        
        Controller->>Controller: isCourseService(service)?
        
        alt Is Course Service
            Controller->>BookingsApi: listSchedule(serviceId)
            BookingsApi->>ScheduleAPI: ScheduleServer.list()<br/>includeTotalNumberOfParticipants: true
            ScheduleAPI-->>BookingsApi: Schedule Response
            Note right of ScheduleAPI: - capacity<br/>- totalNumberOfParticipants<br/>- firstSessionStart<br/>- lastSessionEnd
            BookingsApi-->>Controller: Schedule
        end
    end

    rect rgb(255, 230, 180)
        Note over Controller,BodyVM: Step 2: Calculate Booking Policy
        
        Controller->>PolicyMapper: mapServiceToBookingPolicy(service, schedule, locale)
        
        Note over PolicyMapper: Calculates:<br/>• isBookable = onlineBooking.enabled<br/>• isFullyBooked = (capacity - participants) ≤ 0<br/>• isTooEarlyToBook = booking window not open<br/>• isTooLateToBook = deadline passed<br/>• isServiceAvailable = has sessions & not ended<br/>• isServiceStartedAndBookable = started + bookAfterStart<br/>• numberOfSpotsLeft = capacity - participants
        
        PolicyMapper-->>Controller: BookingPolicyDto
    end

    rect rgb(220, 210, 255)
        Note over Controller,BodyVM: Step 3: Determine Final Availability
        
        Controller->>BodyVM: bodyViewModelFactory(bookingsPolicy, service)
        
        Note over BodyVM: getServiceAvailability():<br/>• if (isFullyBooked OR !isServiceAvailable OR isTooEarly)<br/>    → return false<br/>• if (isTooLateToBook)<br/>    → return isServiceStartedAndBookable<br/>• else → return isBookable
        
        Note over BodyVM: getAvailabilityMessageType():<br/>→ FULLY_BOOKED | TOO_EARLY | TOO_LATE<br/>   | NOT_AVAILABLE | STARTED_AND_BOOKABLE
        
        BodyVM-->>Controller: BodyViewModel { isBookable, messageType }
    end

    Controller->>Client: setProps({ viewModel })
    
    rect rgb(230, 230, 230)
        Note over Client: Step 4: Render UI
        
        Note over Client: Body.tsx renders:<br/>• if isBookable → Show "Book Now" button<br/>• if messageType = TOO_LATE<br/>    → "This course can no longer be booked"<br/>• if messageType = FULLY_BOOKED<br/>    → "Fully Booked" message<br/>• etc.
    end
```

### File References

| Step | File | Lines | Purpose |
|------|------|-------|---------|
| 1-3 | `src/components/BookingServicePage/controller-logic/init-widgets.ts` | 139-153 | Controller initialization, API calls |
| 1 | `src/api/BookingsApi.ts` | 55-70 | `queryServicesBySlug()` - fetches Service data |
| 3 | `src/api/BookingsApi.ts` | 88-100 | `listSchedule()` - fetches Schedule for courses |
| 4 | `src/mappers/booking-policy.mapper.ts` | entire file | Maps Service + Schedule → BookingPolicyDto |
| 5 | `src/service-page-view-model/body-view-model/bodyViewModel.ts` | 26-76 | Determines final availability & message type |
| 6-7 | `src/components/BookingServicePage/Widget/Body/Body.tsx` | 40-77 | Renders availability messages |

---

## Diagram 2: Fetching and Presenting Course Sessions (Events)

This flow fetches all sessions/events for a course and transforms them into a displayable schedule.

```mermaid
sequenceDiagram
    autonumber
    participant Client as Client (Widget)
    participant Controller as Controller<br/>(init-widgets.ts)
    participant Fetcher as SchedulingFetcher<br/>(scheduling-fetcher.ts)
    participant BookingsApi as BookingsApi<br/>(BookingsApi.ts)
    participant ScheduleVM as SchedulingSectionVM<br/>(schedulingSectionViewModel.ts)
    participant CalendarAPI as Calendar V3 API<br/>(@wix/ambassador-calendar-v3-event)
    participant ScheduleRPC as Schedule RPC<br/>(schedule.api.ts)

    Client->>Controller: Page Load (after service fetched)
    
    Controller->>Controller: getServiceSchedulingDataByLocation(locationId)
    Controller->>Client: setProps({ scheduleViewModel: LOADING })

    rect rgb(200, 220, 255)
        Note over Controller,ScheduleRPC: Step 1: Fetch Sessions
        
        Controller->>Fetcher: getServiceSchedulingData(service, settings, ...)
        
        Fetcher->>Fetcher: isCourseService(service)?
        
        Note over Fetcher: Calculate date range:<br/>from = schedule.firstSessionStart<br/>to = schedule.lastSessionEnd
        
        Fetcher->>Fetcher: Check experiment:<br/>useQueryEventsInServicePage
        
        alt useQueryEvents = true (New API)
            Fetcher->>BookingsApi: queryEvents({ from, to, scheduleId, timezone })
            BookingsApi->>CalendarAPI: HTTP: queryEvents()
            Note right of CalendarAPI: Params:<br/>- fromLocalDate<br/>- toLocalDate<br/>- filter: { appId, scheduleId }
            CalendarAPI-->>BookingsApi: Event[]
            Note right of CalendarAPI: Response:<br/>- start.utcDate<br/>- end.utcDate<br/>- resources[].name (staff)<br/>- location.name
            BookingsApi-->>Fetcher: { sessions: Event[] }
        else useQueryEvents = false (Legacy)
            Fetcher->>ScheduleRPC: httpClient.request(getSchedule(...))
            Note over ScheduleRPC: Calls:<br/>• getSessions() via schedule-server<br/>• getCatalogDataForSchedule()
            ScheduleRPC-->>Fetcher: { sessions: CatalogSessionDto[] }
            Note right of ScheduleRPC: Response:<br/>- startDate<br/>- endDate<br/>- durationInMinutes<br/>- staffName<br/>- locationName
        end
        
        Fetcher-->>Controller: ServiceSchedulingData { sessions }
    end

    rect rgb(255, 230, 180)
        Note over Controller,ScheduleVM: Step 2: Transform to View Model
        
        Controller->>ScheduleVM: schedulingSectionViewModelFactory({ sessions, businessInfo, ... })
        
        Note over ScheduleVM: For each session:<br/>• Format date (Intl.DateTimeFormat)<br/>• Extract startTime (hour:minute)<br/>• Calculate duration text<br/>• Get staff name<br/>• Get location name
        
        Note over ScheduleVM: Group by day:<br/>• sortSessions() → by date, location, staff<br/>• formatDays() → SchedulingDayViewModel[]
        
        ScheduleVM-->>Controller: SchedulingSectionViewModel
        Note right of ScheduleVM: Returns:<br/>{ schedulingDaysViewModel: [...],<br/>  status: SUCCESS,<br/>  firstSessionDate,<br/>  lastSessionDate,<br/>  isBookable }
    end

    Controller->>Client: setProps({ scheduleViewModel })

    rect rgb(220, 210, 255)
        Note over Client: Step 3: Render Schedule UI
        
        Note over Client: Widget.tsx → Scheduling.tsx:<br/><br/>if status = SUCCESS:<br/>  ┌─────────────────────────────┐<br/>  │ Mon, Jan 10                 │<br/>  │  • 10:00 AM - 45 min        │<br/>  │    John Smith - Main Studio │<br/>  │  • 2:00 PM - 45 min         │<br/>  │    Jane Doe - Room A        │<br/>  │                             │<br/>  │ Wed, Jan 12                 │<br/>  │  • 10:00 AM - 45 min        │<br/>  │    John Smith - Main Studio │<br/>  └─────────────────────────────┘<br/><br/>if status = EMPTY:<br/>  → Show empty state message<br/><br/>if status = LOADING:<br/>  → Show loading spinner
    end
```

### File References

| Step | File | Lines | Purpose |
|------|------|-------|---------|
| 1 | `src/components/BookingServicePage/controller-logic/init-widgets.ts` | 343-414 | Triggers session fetch by location |
| 2-4 | `src/components/BookingServicePage/controller-logic/scheduling-fetcher.ts` | 38-77 | Determines course type & date range |
| 5a | `src/api/BookingsApi.ts` | 102-131 | `queryEvents()` - new Calendar V3 API |
| 5b | `src/api/schedule.api.ts` | entire file | `getSchedule()` - legacy serverless RPC |
| 6 | `src/service-page-view-model/scheduling-section-view-model/schedulingSectionViewModel.ts` | entire file | Transforms sessions → display view model |
| 7-8 | `src/components/BookingServicePage/Widget/Widget.tsx` | 81-101 | Renders scheduling section |

---

## Backend APIs Summary

| Purpose | API Package | Endpoint |
|---------|-------------|----------|
| **Get Service Details** | `@wix/ambassador-bookings-services-v2-service` | `queryServices()` |
| **Get Schedule (capacity, participants)** | `@wix/ambassador-schedule-server` | `ScheduleServer.Schedules().list()` |
| **Get Sessions (new)** | `@wix/ambassador-calendar-v3-event` | `queryEvents()` |
| **Get Sessions (legacy)** | Serverless RPC via `schedule.api.ts` | `getSchedule()` → `getSessions()` |
| **Get Business Settings** | `@wix/ambassador-bookings-v2-bookings-settings` | `getBookingsSettings()` |

---

## Key Data Types

### BookingPolicyDto

```typescript
// File: src/types/shared-types.d.ts

interface BookingPolicyDto {
  isBookable: boolean;              // Is online booking enabled?
  isFullyBooked: boolean;           // Are all spots taken?
  isTooEarlyToBook: boolean;        // Is booking window not open yet?
  timeUntilServiceIsOpen: string;   // Human-readable time until booking opens
  isTooLateToBook: boolean;         // Has booking deadline passed?
  isServiceAvailable: boolean;      // Does service have sessions & not ended?
  isServiceStartedAndBookable: boolean; // Started but still bookable?
  numberOfSpotsLeft?: number;       // Remaining capacity
  cancellationPolicy?: string;      // Cancellation policy text
}
```

### CatalogSessionDto (Legacy)

```typescript
// File: src/types/shared-types.d.ts

interface CatalogSessionDto {
  startDate: string;         // ISO date string
  endDate: string;           // ISO date string
  durationInMinutes: number; // Session duration
  staffName: string;         // Staff member name
  locationName?: any;        // Location name (optional)
}
```

### Event (New Calendar V3)

```typescript
// From: @wix/ambassador-calendar-v3-event/types

interface Event {
  start: { utcDate: Date };
  end: { utcDate: Date };
  resources: Array<{ id: string; name: string }>;  // Staff members
  location: { id: string; name: string };
  scheduleId: string;
  // ... other fields
}
```

### SchedulingSectionViewModel

```typescript
// File: src/service-page-view-model/scheduling-section-view-model/schedulingSectionViewModel.ts

interface SchedulingSectionViewModel {
  schedulingDaysViewModel?: SchedulingDayViewModel[];
  status: SchedulingSectionStatus;  // LOADING | EMPTY | FAILED | SUCCESS
  firstSessionDate?: string;        // Formatted first session date
  lastSessionDate?: string;         // Formatted last session date
  isBookable: boolean;
}

interface SchedulingDayViewModel {
  date: string;                     // Formatted date (e.g., "Mon, Jan 10")
  dailySessions: DailySession[];
}

interface DailySession {
  startTime: string;        // Formatted time (e.g., "10:00 AM")
  durationText: string;     // Human-readable duration (e.g., "45 min")
  durationAriaLabel: string;
  staffName: string;
  locationName?: string;
}
```

---

## Availability Decision Logic

```mermaid
flowchart TD
    A[Start: Check Course Availability] --> B{isBookable?<br/>onlineBooking.enabled}
    B -->|No| Z1[⊘ NOT BOOKABLE<br/>Online booking disabled]
    B -->|Yes| C{isFullyBooked?<br/>capacity - participants ≤ 0}
    C -->|Yes| Z2[⊘ FULLY BOOKED<br/>No spots left]
    C -->|No| D{isServiceAvailable?<br/>has sessions & not ended}
    D -->|No| Z3[⊘ NOT AVAILABLE<br/>No sessions or ended]
    D -->|Yes| E{isTooEarlyToBook?<br/>booking window not open}
    E -->|Yes| Z4[⊘ TOO EARLY<br/>Come back later]
    E -->|No| F{isTooLateToBook?<br/>booking deadline passed}
    F -->|No| Z5[◉ BOOKABLE<br/>Show Book Now button]
    F -->|Yes| G{isServiceStartedAndBookable?<br/>bookAfterStartPolicy.enabled}
    G -->|Yes| Z6[◉ BOOKABLE<br/>Course started but open]
    G -->|No| Z7[⊘ TOO LATE<br/>This course can no<br/>longer be booked]

    style Z1 fill:#ffcc80,stroke:#e65100
    style Z2 fill:#ffcc80,stroke:#e65100
    style Z3 fill:#ffcc80,stroke:#e65100
    style Z4 fill:#ffcc80,stroke:#e65100
    style Z7 fill:#ffcc80,stroke:#e65100
    style Z5 fill:#90caf9,stroke:#1565c0
    style Z6 fill:#90caf9,stroke:#1565c0
```

**Legend:**
- 🔵 **Blue boxes** = Course is BOOKABLE (success)
- 🟠 **Orange boxes** = Course is NOT BOOKABLE (blocked)

---

## Architecture Overview

```mermaid
graph TB
    subgraph Frontend ["Frontend (Service Details Widget)"]
        Controller[Controller<br/>init-widgets.ts]
        BookingsApi[BookingsApi.ts]
        PolicyMapper[booking-policy.mapper.ts]
        SchedulingFetcher[scheduling-fetcher.ts]
        BodyVM[bodyViewModel.ts]
        ScheduleVM[schedulingSectionViewModel.ts]
        Widget[Widget.tsx]
    end
    
    subgraph Backend ["Backend APIs"]
        ServicesV2[Services V2 API<br/>queryServices]
        ScheduleServer[Schedule Server<br/>list schedules]
        CalendarV3[Calendar V3 API<br/>queryEvents]
        BookingsSettings[Bookings Settings API<br/>getBookingsSettings]
    end
    
    Controller --> BookingsApi
    BookingsApi --> ServicesV2
    BookingsApi --> ScheduleServer
    BookingsApi --> CalendarV3
    BookingsApi --> BookingsSettings
    
    Controller --> PolicyMapper
    Controller --> SchedulingFetcher
    SchedulingFetcher --> BookingsApi
    
    PolicyMapper --> BodyVM
    SchedulingFetcher --> ScheduleVM
    
    BodyVM --> Widget
    ScheduleVM --> Widget
    
    style Frontend fill:#bbdefb,stroke:#1976d2
    style Backend fill:#ffe0b2,stroke:#e65100
```

**Legend:**
- 🔵 **Blue** = Frontend components
- 🟠 **Orange** = Backend APIs
