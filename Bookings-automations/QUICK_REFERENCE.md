# Bookings Automations Service v2 - Quick Reference

## Service at a Glance

**Purpose**: Bridge between Wix Bookings domain events and Wix Automations Platform

**Framework**: Loom Prime

**Main Service**: `wix.bookings.automations.v2.AutomationsService`

**SPI Service**: `com.wixpress.esb.spi.trigger.v1.TriggerProviderService`

---

## Key Files

| File | Purpose |
|------|---------|
| `AutomationsService.scala` | Main service, handles domain events |
| `TriggerProviderService.scala` | SPI service for trigger schemas/validation |
| `TriggerEnricher.scala` | Data enrichment from external services |
| `trigger/base/Trigger.scala` | Base trigger framework |
| `reproducer/*.scala` | Event transformation |
| `db/FileRepository.scala` | ICS file storage |

---

## All Triggers (17 total)

| Trigger | Key | Event Source |
|---------|-----|--------------|
| `SessionsBookedTrigger` | `wix_bookings-sessions_booked` | BookingChanged → CONFIRMED |
| `BookingCanceledTrigger` | `wix_bookings-booking_canceled` | BookingChanged → CANCELED |
| `SessionUpdatedTrigger` | `wix_bookings-session_updated` | EventUpdatedWithMetadata |
| `AppointmentConfirmedTrigger` | `wix_bookings-appointment_confirmed` | BookingChanged |
| `AppointmentDeclinedTrigger` | `wix_bookings-appointment_declined` | BookingChanged |
| `AppointmentRequiresConfirmationTrigger` | `wix_bookings-appointment_requires_confirmation` | BookingChanged |
| `DoubleBookedTrigger` | `wix_bookings-double_booked` | BookingChanged |
| `AnyCheckInTrigger` | `wix_bookings-any_check_in` | CheckInEvent |
| `LatestCheckInTrigger` | `wix_bookings-latest_check_in` | CheckInEvent |
| `NthCheckInTrigger` | `wix_bookings-nth_check_in` | CheckInEvent |
| `NoShowTrigger` | `wix_bookings-no_show` | CheckInEvent |
| `SessionStartedTrigger` | `wix_bookings-session_starts` | SessionParticipationChangedEvent |
| `SessionEndedTrigger` | `wix_bookings-session_ends` | SessionParticipationChangedEvent |
| `CourseSessionsUpdatedTrigger` | `wix_bookings-course_sessions_updated` | EventUpdatedWithMetadata |
| `NoSessionsLeftTrigger` | `wix_bookings-no_sessions_left` | CreditNotification |
| `FailedToApplyBookingFeeTrigger` | `wix_bookings-failed_to_apply_booking_fee` | FailedToApplyBookingFeeToOrder |
| `FailedToCollectAppliedBookingFeesTrigger` | `wix_bookings-failed_to_collect_applied_booking_fees` | FailedToCollectAppliedBookingFees |

---

## Domain Event Subscriptions

| Handler | Entity | Group | Purpose |
|---------|--------|-------|---------|
| `handleBookingDomainEventForStatusChanges` | `Booking` | `status-changes` | First list of triggers |
| `handleBookingDomainEventForStatusChanges2` | `Booking` | `status-changes-2` | Second list of triggers |
| `handleBookingDomainEventForSessionParticipationChanged` | `Booking` | `session-participation-changed` | Session participation |
| `handleEventDomainEventForSessionsUpdated` | `Event` | `course-sessions-updated` | Course updates |
| `handleEventsViewDomainEventForSessionParticipationChanged` | `EventsView` | `session-participation-changed` | Session participation |
| `handleAttendanceDomainEvent` | `Attendance` | - | Check-in events |
| `handleBookingFeeDomainEvent` | `BookingFee` | - | Fee failures |
| `handleCreditNotification` | - | - | Pricing plan credits |

---

## Internal Events

| Event | Topic | Purpose |
|-------|-------|---------|
| `CheckInEvent` | `bookings-automations-2-internal-check-in-event` | Check-in processing |
| `SessionParticipationChangedEvent` | `bookings-automations-2-internal-session-participation-changed-event` | Session start/end |
| `DelayedExecutionEvent` | `bookings-automations-2-internal-delayed-execution-event` | Delayed trigger execution |

---

## Data Enrichment Sources

**TriggerEnricher** fetches data from:

- **Bookings**: Booking details, policy snapshots, anonymous links
- **Calendar**: Events, schedules, conferencing details
- **Services**: Service details, form IDs
- **Orders & Payments**: Order details, payment status, payment links
- **Forms**: Form submissions, custom fields, intake forms
- **Site**: Business properties, site URL, meta site
- **Data Extensions**: Custom schema fields
- **Booking Fees**: Fee information
- **Pricing Plans**: Member order details

---

## Key Patterns

### 1. Delayed Execution
- **Why**: Allow concurrent operations (conferencing, orders) to complete
- **How**: 6-second delay via Kafka for booking changes
- **Implementation**: `Delayer` → `DelayedExecutionEvent` → callback

### 2. Event Reproduction
- **Why**: Transform domain events into specialized internal events
- **Examples**: 
  - `Attendance` → `CheckInEvent`
  - `BookingChanged` → `SessionParticipationChangedEvent`

### 3. Parallel Data Fetching
- **Why**: Minimize latency
- **How**: `FutureUtil.inParallel()` fetches from multiple services simultaneously

### 4. Idempotency
- **Key**: Booking ID
- **TTL**: Based on booking end time
- **Purpose**: Prevent duplicate automation triggers

---

## Configuration Overrides (SPI)

**BookingAutomationsConfiguration SPI** allows:

1. **Global Toggle**: `enabled` - Enable/disable all configs
2. **Trigger Disable**: `allow_booking_automation_triggering` - Emergency brake
3. **Trigger Override**: `trigger_overrides` - Redirect to custom triggers
4. **Field Override**: `trigger_field_overrides` - Transform field keys

**Example**:
```json
{
  "enabled": true,
  "allow_booking_automation_triggering": true,
  "trigger_overrides": {
    "wix_bookings-sessions_booked": {
      "automation_trigger_key": "wix_meetings-meeting_booked"
    }
  },
  "trigger_field_overrides": {
    "staff_member_name": {
      "automation_trigger_field_key": "host_name"
    }
  }
}
```

---

## Feature Toggles

1. `buildCustomFormFieldsBasedOnFormPopulation`
2. `overrideWixBookingAutomationsTriggers`
3. `addAnonymousLinksByPolicy`
4. `useCorrectFormFieldTypes`
5. `useFallbackTitleForMissingSchemaTitle`
6. `useBookingsConfigForFormsNamespace`

---

## Retry Strategy

**Non-blocking retries**:
- 30s, 30s, 1m, 1m, 5m, 10m, 30m, 1h, 3h

**Ignored errors** (logged but not retried):
- Missing tenant ID (automations not installed)
- Meta-site not found (site deleted)

---

## ICS File Management

- **Storage**: DataStore table `bookings-automations-v2-files`
- **TTL**: 30 days
- **Methods**: CREATE, UPDATE, CANCEL
- **Endpoint**: `GET /ics/{id}`

---

## Common Workflows

### Adding a New Trigger

1. Create trigger class extending `Trigger` or `TriggerWithFilter`
2. Implement `triggerImpl()` method
3. Use `TriggerEnricher` for data fetching
4. Build payload map
5. Call `report()` with idempotency
6. Add to `Triggers` case class
7. Register in `AutomationsService.apply()`
8. Add handler if needed

### Debugging Trigger Execution

1. Check logs for trigger key and entity ID
2. Verify filter returns `true`
3. Check data enrichment (all futures complete)
4. Verify payload map construction
5. Check ESB response for activation IDs
6. Review BI logs for trigger reporting

### Testing Trigger

1. Use `BookingsAutomationsSpec`
2. Mock RPC clients
3. Create test domain events
4. Verify trigger filtering
5. Verify payload content
6. Verify ESB reporting

---

## Key Constants

- **Bookings App ID**: `13d21c63-b5ec-5912-8397-c3a5ddb27a97`
- **Meetings App ID**: `6646a75c-2027-4f49-976c-58f3d713ed0f`
- **Book In Form App ID**: `3e4e322b-6d9b-4da0-9f53-e22c4a44a49e`
- **BI Source ID**: `16`
- **Kafka Delay**: `8 seconds`
- **Booking Change Delay**: `6 seconds`
- **File TTL**: `30 days`

---

## Important Notes

⚠️ **Delayed Execution**: Booking changes are delayed 6 seconds to allow concurrent operations

⚠️ **Parallel Processing**: Two trigger lists for booking changes to allow concurrent processing

⚠️ **Idempotency**: Always use booking ID + TTL for idempotency keys

⚠️ **Error Handling**: Use `ignoreFailures` flag for non-critical data fetching

⚠️ **Field Overrides**: Field key overrides apply to ALL triggers when configured

⚠️ **Meetings Integration**: Temporary workaround for Wix Meetings triggers (will be removed)

