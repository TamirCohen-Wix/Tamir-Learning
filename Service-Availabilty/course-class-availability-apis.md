# Course & Class Availability - APIs and Calculations

This document describes the APIs used and calculations performed by each widget to determine and display course/class availability.

---

## Data Sources Overview

### 1. BookingPolicy (Per-Service Settings)

**Source:** Property on the Service object, returned from `queryServices` API (`@wix/ambassador-bookings-services-v2-service`)

**Contains settings configured by the business owner for how the service can be booked:**

| Field | Description |
|-------|-------------|
| `limitLateBookingPolicy.enabled` | Whether late booking restriction is active |
| `limitLateBookingPolicy.latestBookingInMinutes` | Minutes before service starts that booking closes |
| `limitEarlyBookingPolicy.enabled` | Whether early booking restriction is active |
| `limitEarlyBookingPolicy.earliestBookingInMinutes` | Minutes before service starts that booking opens |
| `bookAfterStartPolicy.enabled` | Whether booking is allowed after course/class started |
| `participantsPolicy.maxParticipantsPerBooking` | Max participants per single booking |
| `resourcesPolicy.autoAssignAllowed` | Whether staff can be auto-assigned |
| `customPolicyDescription.description` | Cancellation policy text |

---

### 2. Schedule (Per-Service Data)

**Source:** `ScheduleServer.list()` API (`@wix/ambassador-schedule-server`)

**Contains runtime data about the service schedule:**

| Field | Description |
|-------|-------------|
| `capacity` | Total number of spots available |
| `totalNumberOfParticipants` | Current number of booked participants |
| `firstSessionStart` | Date/time of first session |
| `lastSessionEnd` | Date/time of last session |
| `status` | Schedule status (CREATED, etc.) |

---

### 3. BookingsSettings (Business-Wide Settings)

**Source:** `getBookingsSettings` API (`@wix/ambassador-bookings-v2-bookings-settings`)

**Contains business-wide settings that apply to all services:**

| Field | Description |
|-------|-------------|
| `multiServiceAppointments.enabled` | Whether multi-service booking is allowed |
| `staffSelection.strategy` | CUSTOMER_CANNOT_CHOOSE, CUSTOMER_MAY_CHOOSE, or CUSTOMER_MUST_CHOOSE |
| `staffSelection.timing` | BEFORE_SLOT_SELECTION or AFTER_SLOT_SELECTION |
| `locationSelection.timing` | BEFORE_SLOT_SELECTION or AFTER_SLOT_SELECTION |
| `siteProperties` | Business site configuration |
| `features` | Enabled feature flags |

---

## Widget 1: BookOnline / ServiceListWidget

**Package:** `bookings-service-list-widget`

### APIs Called

| API | Package | Purpose |
|-----|---------|---------|
| `queryServices` | `@wix/ambassador-bookings-services-v2-service` | Gets all services with their schedule and bookingPolicy |
| `ScheduleServer.list()` | `@wix/ambassador-schedule-server` | Gets capacity and participant count (COURSE only) |
| `queryEventsClassDays` | Custom GraphQL | Gets which weekdays classes occur (CLASS only) |
| `getBookingsSettings` | `@wix/ambassador-bookings-v2-bookings-settings` | Gets business-wide settings |

### Calculations

| Calculation | Input Data | Formula | Purpose |
|-------------|------------|---------|---------|
| Spots Left | `schedule.capacity`, `schedule.totalNumberOfParticipants` | `capacity - totalNumberOfParticipants` | Show "X spots left" message |
| Is Too Late to Book | `bookingPolicy.limitLateBookingPolicy`, `schedule.firstSessionStart` | `now > (firstSessionStart - latestBookingInMinutes)` | Determine if booking window closed |
| Is Passed End Date | `schedule.lastSessionEnd` | `lastSessionEnd < now` | Determine if course has ended |
| Can Book After Start | `bookingPolicy.bookAfterStartPolicy.enabled` | Direct boolean check | Allow mid-course booking |

### Course vs Class Behavior

| Service Type | Behavior |
|--------------|----------|
| **COURSE** | Fetches availability data (spots left) from Schedule Server. Calculates isTooLateToBook and isPassedEndDate. |
| **CLASS** | Fetches offered days (weekdays) from Events GraphQL API. No spots calculation - classes repeat weekly. |

---

## Widget 2: BookingServicePage (Service Details Widget)

**Package:** `bookings-service-details-widget`

### APIs Called

| API | Package | Purpose |
|-----|---------|---------|
| `queryServices` (by slug) | `@wix/ambassador-bookings-services-v2-service` | Gets single service with schedule and bookingPolicy |
| `ScheduleServer.list()` | `@wix/ambassador-schedule-server` | Gets capacity, participants, firstSessionStart, lastSessionEnd |
| `queryEvents` | `@wix/ambassador-calendar-v3-event` | Gets individual sessions/events for scheduling section |
| `getBookingsSettings` | `@wix/ambassador-bookings-v2-bookings-settings` | Gets business-wide booking settings |

### Calculations

| Calculation | Input Data | Formula | Purpose |
|-------------|------------|---------|---------|
| Is Fully Booked | `schedule.capacity`, `schedule.totalNumberOfParticipants` | `capacity - totalNumberOfParticipants <= 0` | Show "This course is full" message |
| Number of Spots Left | `schedule.capacity`, `schedule.totalNumberOfParticipants` | `capacity - totalNumberOfParticipants` | Display remaining spots |
| Is Too Early to Book | `bookingPolicy.limitEarlyBookingPolicy`, `schedule.firstSessionStart` | `minutesUntilStart >= earliestBookingInMinutes` | Show "Booking opens in X" message |
| Is Too Late to Book | `bookingPolicy.limitLateBookingPolicy`, `schedule.firstSessionStart` | `minutesUntilStart <= latestBookingInMinutes AND !bookAfterStart` | Show "Can no longer be booked" message |
| Is Service Available | `schedule.firstSessionStart`, `schedule.lastSessionEnd` | `firstSessionStart exists AND lastSessionEnd > now` | Check if service has future sessions |
| Is Service Started And Bookable | `schedule`, `bookingPolicy.bookAfterStartPolicy` | `firstSessionStart < now AND lastSessionEnd > now AND bookAfterStart.enabled` | Allow mid-course booking |
| Time Until Service Is Open | `schedule.firstSessionStart`, `bookingPolicy.earliestBookingInMinutes` | `firstSessionStart - earliestBookingInMinutes - now` | Display countdown to booking opening |

### BookingsSettings Usage

| Setting | Usage |
|---------|-------|
| `staffSelection.strategy` | Controls whether staff selection step is shown in booking flow |
| `locationSelection.timing` | Controls when location selection appears in booking flow |

### Course vs Class Behavior

| Service Type | Behavior |
|--------------|----------|
| **COURSE** | Full availability calculations: spots left, too late, too early, fully booked, service started and bookable. All booking policy checks apply. |
| **CLASS** | Only checks `isServiceAvailable` (has sessions and not ended). No spots/fully booked logic - classes repeat and have per-session capacity. |

### Availability Message Priority (COURSE only)

Messages are displayed in this priority order:

1. `isFullyBooked` → "This course is full"
2. `isTooEarlyToBook` → "Booking opens in X"
3. `isTooLateToBook` → "This course can no longer be booked"
4. `!isServiceAvailable` → "Not available"
5. `isServiceStartedAndBookable` → "Course started but still bookable"

---

## Widget 3: BookingCalendar (Calendar Widget)

**Package:** `bookings-calendar-widget`

### Important: COURSE Services Not Supported

The Calendar Widget explicitly excludes COURSE type services:

```
isCalendarFlow = !(service.type === ServiceType.COURSE) && isServiceBookOnlineEnabled(service)
```

**COURSE services must be booked through the BookingServicePage (Service Details Widget).**

### APIs Called

| API | Package | Service Type | Purpose |
|-----|---------|--------------|---------|
| `listEventTimeSlots` | `@wix/ambassador-bookings-availability-v2-time-slot` | CLASS | Gets available time slots for class services |
| `listAvailabilityTimeSlots` | `@wix/ambassador-bookings-availability-v2-time-slot` | APPOINTMENT (single) | Gets available time slots for single appointment |
| `listMultiServiceAvailabilityTimeSlots` | `@wix/ambassador-bookings-availability-v2-time-slot` | APPOINTMENT (multi) | Gets slots for multiple appointments |
| `getBookingsSettings` | `@wix/ambassador-bookings-v2-bookings-settings` | All | Gets staff/location selection settings |

### Calculations

| Calculation | Input Data | Formula | Purpose |
|-------------|------------|---------|---------|
| Flow Detection | `service.type` | Switch on CLASS vs APPOINTMENT | Determines which API to call |
| Open Spots Filter | `remainingCapacity` | `remainingCapacity >= 1` | Filter out fully booked slots |
| Only Available Slots | Widget settings | `minBookableCapacity: 1, includeNonBookable: false` | Show only bookable slots if setting enabled |
| Too Late To Book Filter | Booking policy violations | `bookingPolicyViolations.tooLateToBook: false` | Exclude slots past booking window |
| Business Locations Filter | Selected location settings | Filter by `location.businessLocation.id` | Filter slots by location |
| Staff Filter | Filter options | Filter by `resources.id` | Filter slots by selected staff member |

### BookingsSettings Usage

| Setting | Usage |
|---------|-------|
| `staffSelection.strategy` | Determines if staff filter is shown/required |
| `staffSelection.timing` | When to prompt for staff selection |
| `multiServiceAppointments.enabled` | Whether to use multi-service appointment flow |

### Service Type Behavior

| Service Type | API Used | Capacity Handling |
|--------------|----------|-------------------|
| **COURSE** | Not supported | N/A - Use BookingServicePage |
| **CLASS** | `listEventTimeSlots` | Filters by `remainingCapacity >= 1`. Can show non-bookable slots. |
| **APPOINTMENT (single)** | `listAvailabilityTimeSlots` | Assumes 1 participant needed per slot |
| **APPOINTMENT (multi)** | `listMultiServiceAvailabilityTimeSlots` | Coordinates availability across multiple services |

---

## Summary: What Each Data Source Controls

| Data Source | Origin | Scope | Controls |
|-------------|--------|-------|----------|
| `service.bookingPolicy` | Query Services API | Per service | Late/early booking limits, after-start booking, max participants, cancellation policy |
| `schedule` | Schedule Server API | Per service | Capacity, current participants, first/last session dates |
| `bookingsSettings` | Get Bookings Settings API | Business-wide | Staff selection rules, location selection rules, multi-service support |

---

## File References

### BookOnline / ServiceListWidget
- `packages/bookings-service-list-widget/src/api/BookingsApi.ts` - API calls
- `packages/bookings-service-list-widget/src/utils/serviceDetails/displayOptions.ts` - Availability calculations
- `packages/bookings-service-list-widget/src/actions/getAdditionalServicesData/getAdditionalServicesData.ts` - Data fetching logic
- `packages/bookings-service-list-widget/src/components/BookOnline/Widget/Body/ServiceCard/ServiceInfo/CourseAvailability/CourseAvailability.tsx` - UI component

### BookingServicePage (Service Details Widget)
- `packages/bookings-service-details-widget/src/api/BookingsApi.ts` - API calls
- `packages/bookings-service-details-widget/src/mappers/booking-policy.mapper.ts` - Policy calculations
- `packages/bookings-service-details-widget/src/service-page-view-model/body-view-model/bodyViewModel.ts` - Message type logic
- `packages/bookings-service-details-widget/src/components/BookingServicePage/controller-logic/init-widgets.ts` - Initialization

### BookingCalendar (Calendar Widget)
- `packages/bookings-calendar-widget/src/api/fetchAvailability.ts` - Availability fetching
- `packages/bookings-calendar-widget/src/api/BookingAPIS/listEventTimeSlots.ts` - Class slots API
- `packages/bookings-calendar-widget/src/utils/serviceUtils/serviceUtils.ts` - Service type handling

### Shared Modules
- `packages/modules/bookings-calendar-catalog-viewer-mapper/src/domain/bookingsSettings/bookingsSettings.ts` - BookingsSettings utilities
- `packages/modules/bookings-calendar-catalog-viewer-mapper/src/domain/service/service.ts` - Service utilities
