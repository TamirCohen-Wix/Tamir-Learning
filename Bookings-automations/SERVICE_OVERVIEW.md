# Bookings Automations Service v2 - Service Overview

## Executive Summary

The **Bookings Automations Service v2** (`bookings-automations-2`) is a Loom Prime-based service that acts as a bridge between Wix Bookings domain events and the Wix Automations Platform. It listens to booking-related domain events, enriches them with additional data, and triggers automations in the Wix Automations system.

### Key Responsibilities
- **Event Processing**: Subscribes to domain events from Bookings, Calendar, and Attendance services
- **Data Enrichment**: Fetches and enriches event data with booking details, orders, services, forms, etc.
- **Automation Triggering**: Reports events to the Wix Automations Platform (ESB) to trigger user-configured automations
- **Trigger Management**: Provides trigger schemas, validation, and refresh capabilities via SPI
- **File Management**: Generates and stores ICS calendar files for email attachments

---

## Service Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Wix Automations Platform                      │
│                    (ESB Config Resolver)                         │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ reportEvent / bulkReportEvent
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              Bookings Automations Service v2                     │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │         AutomationsService (Main Service)                  │  │
│  │  - Domain Event Handlers                                  │  │
│  │  - Event Routing & Filtering                              │  │
│  │  - Delayed Execution                                      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │         TriggerProviderService (SPI Service)               │  │
│  │  - Dynamic Schema Generation                              │  │
│  │  - Configuration Validation                               │  │
│  │  - Payload Refresh                                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │         Trigger System                                    │  │
│  │  - 17+ Trigger Implementations                            │  │
│  │  - Base Trigger Framework                                │  │
│  │  - Filtering & Validation                                │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │         TriggerEnricher                                   │  │
│  │  - Data Fetching (Bookings, Orders, Services, etc.)      │  │
│  │  - ICS File Generation                                    │  │
│  │  - Form Field Resolution                                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │         Reproducers                                       │  │
│  │  - Event Transformation                                  │  │
│  │  - Internal Event Publishing                             │  │
│  │  - Delayed Execution Management                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │         Repositories                                      │  │
│  │  - FileRepository (ICS files)                            │  │
│  │  - SentTriggerDataStoreRepository                        │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ Domain Events
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              External Services (RPC Clients)                     │
│  - Bookings Service v2                                          │
│  - Calendar Service v3                                           │
│  - Services Service v2                                           │
│  - Orders & Payments                                             │
│  - Forms Service                                                 │
│  - Site Properties                                               │
│  - And 20+ more...                                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Core Components

### 1. AutomationsService
**Location**: `src/com/wixpress/bookings/automations/v2/AutomationsService.scala`

The main service that handles domain events and routes them to appropriate triggers.

**Key Methods**:
- `handleBookingDomainEventForStatusChanges` - Handles booking status changes
- `handleAttendanceDomainEvent` - Handles attendance/check-in events
- `handleEventDomainEventForSessionsUpdated` - Handles calendar event updates
- `handleDelayedExecutionEvent` - Handles delayed trigger execution

**Event Processing Flow**:
1. Receives domain event via subscription
2. Filters events to determine if any trigger applies
3. For booking changes, delays execution by 6 seconds (to allow concurrent operations to complete)
4. Routes to appropriate trigger(s)
5. Trigger enriches data and reports to Automations Platform

### 2. Trigger System

**Base Classes**:
- `Trigger[E]` - Base trait for all triggers
- `TriggerWithFilter[E]` - Triggers that filter events before processing
- `TriggerWithDynamicSchema[E]` - Triggers with dynamic field schemas
- `TriggerWithRefresh[E]` - Triggers that support payload refresh
- `TriggerWithValidation[E]` - Triggers with configuration validation

**Available Triggers** (17 total):

| Trigger | Key | Description |
|---------|-----|-------------|
| `SessionsBookedTrigger` | `wix_bookings-sessions_booked` | When booking is confirmed |
| `BookingCanceledTrigger` | `wix_bookings-booking_canceled` | When booking is canceled |
| `SessionUpdatedTrigger` | `wix_bookings-session_updated` | When session/appointment is updated |
| `AppointmentConfirmedTrigger` | `wix_bookings-appointment_confirmed` | When appointment is confirmed |
| `AppointmentDeclinedTrigger` | `wix_bookings-appointment_declined` | When appointment is declined |
| `AppointmentRequiresConfirmationTrigger` | `wix_bookings-appointment_requires_confirmation` | When appointment needs confirmation |
| `DoubleBookedTrigger` | `wix_bookings-double_booked` | When double booking detected |
| `AnyCheckInTrigger` | `wix_bookings-any_check_in` | On any check-in |
| `LatestCheckInTrigger` | `wix_bookings-latest_check_in` | On latest check-in |
| `NthCheckInTrigger` | `wix_bookings-nth_check_in` | On Nth check-in |
| `NoShowTrigger` | `wix_bookings-no_show` | When participant doesn't show |
| `SessionStartedTrigger` | `wix_bookings-session_starts` | When session starts |
| `SessionEndedTrigger` | `wix_bookings-session_ends` | When session ends |
| `CourseSessionsUpdatedTrigger` | `wix_bookings-course_sessions_updated` | When course sessions updated |
| `NoSessionsLeftTrigger` | `wix_bookings-no_sessions_left` | When pricing plan credits exhausted |
| `FailedToApplyBookingFeeTrigger` | `wix_bookings-failed_to_apply_booking_fee` | When booking fee application fails |
| `FailedToCollectAppliedBookingFeesTrigger` | `wix_bookings-failed_to_collect_applied_booking_fees` | When fee collection fails |

### 3. TriggerEnricher
**Location**: `src/com/wixpress/bookings/automations/v2/enricher/TriggerEnricher.scala`

Central component for data enrichment. Fetches data from multiple services to build complete trigger payloads.

**Key Responsibilities**:
- Fetch booking details, orders, payments
- Get service information and schedules
- Retrieve form submissions and custom fields
- Generate ICS calendar files
- Get conferencing details
- Fetch business properties
- Get intake form data (premium feature)

**Data Sources**:
- Bookings Service v2
- Calendar Service v3 (Events & Schedules)
- Services Service v2
- Orders & Payments APIs
- Forms Service v4
- Site Properties
- Meta Site API
- Data Extensions
- Booking Fees
- Payment Links
- And more...

### 4. Reproducers
**Location**: `src/com/wixpress/bookings/automations/v2/reproducer/`

Transform domain events into internal events for specialized processing.

**Reproducers**:
- `CheckInEventForBookingReproducer` - Converts Attendance events to CheckInEvent
- `SessionParticipationChangedEventReproducerForBookingsEvents` - Converts Booking events to SessionParticipationChangedEvent
- `SessionParticipationChangedEventReproducerForSessionsEvents` - Converts EventsView events to SessionParticipationChangedEvent
- `Delayers` - Manages delayed execution (6-second delay for booking changes)

### 5. Repositories

**FileRepository**:
- Stores ICS calendar files in DataStore
- 30-day TTL
- Used for email attachments

**SentTriggerDataStoreRepository**:
- Tracks sent triggers (idempotency)

---

## Data Flow Diagrams

### 1. Booking Confirmation Flow

```
┌─────────────────┐
│ Booking Service │
│  (Domain Event) │
└────────┬────────┘
         │
         │ BookingChanged (status → CONFIRMED)
         │
┌────────▼────────────────────────────────────────────┐
│ AutomationsService                                  │
│ handleBookingDomainEventForStatusChanges()         │
└────────┬────────────────────────────────────────────┘
         │
         │ Filter: SessionsBookedTrigger.filter()
         │
         │ [6-second delay via Delayer]
         │
┌────────▼────────────────────────────────────────────┐
│ SessionsBookedTrigger                               │
│ triggerImpl()                                       │
└────────┬────────────────────────────────────────────┘
         │
         │ Parallel Data Fetching:
         │ - getOrderDetailsByBookingId()
         │ - getBusinessProperties()
         │ - getServiceDetails()
         │ - getConferencingAndCreatedIcs()
         │ - getBookingPolicySnapshot()
         │ - getFormattedFormSubmission()
         │ - getFormFieldsV2()
         │ - getPaymentLinkUrl()
         │ - getAnonymousLinks()
         │ - getIntakeFormDataIfPremium()
         │
┌────────▼────────────────────────────────────────────┐
│ TriggerEnricher                                     │
│ (Fetches from 10+ services)                         │
└────────┬────────────────────────────────────────────┘
         │
         │ Enriched Payload
         │
┌────────▼────────────────────────────────────────────┐
│ Trigger.report()                                    │
│ - Build payload map                                 │
│ - Apply field overrides (if configured)            │
│ - Check automations config                          │
└────────┬────────────────────────────────────────────┘
         │
         │ ReportEventRequest
         │
┌────────▼────────────────────────────────────────────┐
│ ESB Config Resolver                                 │
│ (Wix Automations Platform)                          │
└─────────────────────────────────────────────────────┘
```

### 2. Check-In Event Flow

```
┌─────────────────┐
│ Attendance      │
│ Service         │
│ (Domain Event)  │
└────────┬────────┘
         │
         │ AttendanceMarkedAsNotAttended
         │
┌────────▼────────────────────────────────────────────┐
│ AutomationsService                                  │
│ handleAttendanceDomainEvent()                       │
└────────┬────────────────────────────────────────────┘
         │
┌────────▼────────────────────────────────────────────┐
│ CheckInEventForBookingReproducer                    │
│ reproduce()                                         │
│ - Fetch booking by attendance.bookingId            │
│ - Extract contactId                                 │
└────────┬────────────────────────────────────────────┘
         │
         │ Publish to internal topic
         │
┌────────▼────────────────────────────────────────────┐
│ Internal Event: CheckInEvent                        │
│ (bookings-automations-2-internal-check-in-event)   │
└────────┬────────────────────────────────────────────┘
         │
         │ Subscribed by multiple handlers:
         │
    ┌────┴────┬──────────┬──────────┬──────────┐
    │         │          │          │          │
┌───▼───┐ ┌──▼───┐ ┌────▼────┐ ┌───▼────┐
│ Any   │ │ Nth  │ │ Latest  │ │ NoShow │
│ Check │ │Check │ │ Check   │ │Trigger │
│ In    │ │ In   │ │ In      │ │        │
└───────┘ └──────┘ └─────────┘ └────────┘
```

### 3. Delayed Execution Flow

```
┌─────────────────────────────────────┐
│ BookingChanged Event                │
└────────────┬────────────────────────┘
             │
             │ Filter matches trigger
             │
┌────────────▼────────────────────────┐
│ Delayer.delay()                     │
│ - Serialize payload to JSON        │
│ - Publish DelayedExecutionEvent    │
│   (8-second Kafka delay)           │
└────────────┬────────────────────────┘
             │
             │ [8 seconds later]
             │
┌────────────▼────────────────────────┐
│ AutomationsService                  │
│ handleDelayedExecutionEvent()       │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ Delayers.onDelayedEvent()           │
│ - Deserialize JSON                  │
│ - Call registered callback          │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ Trigger.triggerImpl()               │
│ (Actual trigger execution)          │
└─────────────────────────────────────┘
```

### 4. Trigger Provider Service Flow (SPI)

```
┌─────────────────────────────────────┐
│ Wix Automations Platform            │
│ (Building automation UI)            │
└────────────┬────────────────────────┘
             │
             │ getDynamicSchema()
             │ validateConfiguration()
             │ refreshPayload()
             │
┌────────────▼────────────────────────┐
│ TriggerProviderService               │
│ (SPI Implementation)                 │
└────────────┬────────────────────────┘
             │
    ┌────────┴────────┐
    │                 │
┌───▼────┐      ┌────▼─────┐
│Dynamic │      │Validation│
│Schema  │      │Triggers  │
│Triggers│      │          │
└────────┘      └──────────┘
```

---

## Integration Points

### Domain Event Subscriptions

The service subscribes to domain events from:

1. **Bookings Service v2**
   - `BookingChanged` - Status changes, updates
   - Subscription groups: `status-changes`, `status-changes-2`, `session-participation-changed`

2. **Calendar Service v3**
   - `EventUpdatedWithMetadata` - Session updates
   - `EventsViewProjectionUpdated` - Session participation changes

3. **Attendance Service v2**
   - `AttendanceMarkedAsNotAttended` - Check-in events

4. **Booking Fees Service**
   - `FailedToApplyBookingFeeToOrder`
   - `FailedToCollectAppliedBookingFees`

5. **Pricing Plans**
   - `CreditNotification` - When credits exhausted

### RPC Dependencies (33 services)

**Bookings Domain**:
- `BookingsService` v2
- `BookingsReader` v2
- `BookingPolicySnapshots`
- `AnonymousBookingActions`
- `BookingFees`
- `BookingsPaymentLinks`
- `BookingsSettings`

**Calendar Domain**:
- `EventsService` v3
- `SchedulesService` v3
- `EventViews` v3

**Services Domain**:
- `ServicesService` v2

**Forms & Submissions**:
- `FormsService`
- `FormSchemaService`
- `FormSubmissionService`
- `IntakeFormsService`
- `IntakeFormSubmissionsService`

**Orders & Payments**:
- `Orders` (Ecom)
- `Payments` (Ecom)
- `OrderManagementService` (Membership)

**Platform Services**:
- `MetaSiteReadApi`
- `SitePropertiesV4`
- `DataExtensionSchemaService`
- `FeaturesManager`
- `BookingsPlatformConfigService`

**Automations Platform**:
- `EsbConfigResolver` - Main integration point

### SPI Services

**BookingAutomationsConfiguration SPI**:
- Allows third-party apps to:
  - Enable/disable automations
  - Override trigger keys
  - Override field keys
  - Control automation triggering

---

## Key Features

### 1. Delayed Execution
- **6-second delay** for booking change events
- Allows concurrent operations (conferencing, orders, sessions) to complete
- Implemented via Kafka delay mechanism

### 2. Event Reproduction
- Transforms domain events into internal events
- Enables specialized processing (e.g., check-in events)
- Uses internal Kafka topics

### 3. Data Enrichment
- Parallel fetching from multiple services
- Comprehensive payload building
- Handles failures gracefully (ignoreFailures flag)

### 4. ICS File Generation
- Generates calendar files for email attachments
- Stores in DataStore with 30-day TTL
- Supports CREATE, UPDATE, CANCEL methods

### 5. Idempotency
- Uses booking ID + TTL for idempotency keys
- Prevents duplicate automation triggers
- TTL based on booking end time

### 6. Configuration Overrides
- Supports trigger key overrides (e.g., for Wix Meetings)
- Field key overrides for data transformation
- App-level configuration via SPI

### 7. Dynamic Schemas
- Triggers provide dynamic field schemas
- Based on service configuration
- Supports form fields, data extensions

### 8. Validation
- Trigger-level configuration validation
- Service filter validation
- Form field validation

---

## Error Handling

### Exception Remapping
- Uses `NonLastRetryExceptionRemapper`
- Retry strategy: 30s, 30s, 1m, 1m, 5m, 10m, 30m, 1h, 3h
- Handles transient failures gracefully

### Ignored Errors
- Missing tenant ID (automations not installed)
- Meta-site not found (site deleted)
- Logged but not retried

### Bulk Operations
- Batches up to 100 events
- Handles partial failures
- Throws `BulkOperationException` for failures

---

## Feature Toggles

1. `buildCustomFormFieldsBasedOnFormPopulation` - Custom form fields
2. `overrideWixBookingAutomationsTriggers` - Trigger overrides
3. `addAnonymousLinksByPolicy` - Anonymous booking links
4. `useCorrectFormFieldTypes` - Form field type handling
5. `useFallbackTitleForMissingSchemaTitle` - Schema title fallback
6. `useBookingsConfigForFormsNamespace` - Forms namespace config

---

## Database/Storage

### DataStore Tables

1. **bookings-automations-v2-files**
   - Stores ICS calendar files
   - Partition key: file ID
   - TTL: 30 days
   - Namespace: BookingsAllDcs

2. **Sent Trigger Tracking** (via SentTriggerDataStoreRepository)
   - Tracks sent triggers for idempotency

---

## Testing

**Test Structure**:
- `specs/bookings/BookingsAutomationsSpec.scala` - Main test suite
- Uses testkit for RPC clients
- Mocks external services

---

## Deployment

- **Framework**: Loom Prime
- **Service Name**: `wix.bookings.automations.v2.AutomationsService`
- **SPI Service**: `com.wixpress.bookings.automations.spi.v2.BookingAutomationsConfiguration`
- **Secondary Service**: `com.wixpress.esb.spi.trigger.v1.TriggerProviderService`

---

## Summary

The Bookings Automations Service v2 is a sophisticated event-driven service that:

1. **Listens** to booking-related domain events
2. **Enriches** events with comprehensive data from multiple services
3. **Transforms** events into automation triggers
4. **Reports** triggers to the Wix Automations Platform
5. **Manages** trigger schemas, validation, and refresh via SPI

It serves as the critical bridge between the Bookings domain and the broader Wix Automations ecosystem, enabling users to create powerful automated workflows based on booking events.

