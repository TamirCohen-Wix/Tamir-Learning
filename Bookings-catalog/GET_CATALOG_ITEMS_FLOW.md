# getCatalogItems() API Flow Documentation

## Complete Function Call Chain

### Entry Point
1. **`BookingsCatalogService.getCatalogItems(request: GetCatalogItemsRequest)`**
   - Main entry point for the Platform Catalog SPI
   - Logs request and permissions
   - Signs call scope with service identity
   - Calls `enrichAllBookingsItems()`

### Phase 1: Initial Setup & Booking Fetching
2. **`BookingsCatalogService.enrichAllBookingsItems(catalogReferences)`**
   - Exposes metrics for item count
   - Calls `fetchBookingsById()` to get all bookings
   - Calls `prepareBookingGroups()` to separate single vs multi-service bookings
   - Exposes metrics for booking partition
   - Calls `enrichBookingsItems()` and `enrichMultiServiceBookingsItems()` in parallel

3. **`BookingsCatalogService.fetchBookingsById(catalogReferences)`**
   - Extracts unique booking IDs
   - For each booking ID, calls `getBookingById()`
   - Returns map of booking IDs to bookings

4. **`BookingsCatalogService.getBookingById(bookingId)`**
   - Logs context check
   - Calls `context.rpc.bookings.consistentQuery()` (external RPC call)
   - Returns first booking from response

5. **`BookingsCatalogService.prepareBookingGroups(bookingsById, catalogReferences)`**
   - Groups bookings by multi-service booking ID
   - Partitions catalog references into single vs multi-service
   - Returns tuple with partitioned references and booking groups

### Phase 2: Single Bookings Enrichment Path
6. **`BookingsCatalogService.enrichBookingsItems(bookingsById, catalogReferences)`**
   - Extracts all bookings from map
   - Calls `PrioritizedBookings.groupSortedByEventId()` for classes
   - Calls `PrioritizedBookings.groupSortedByScheduleId()` for courses
   - Calls `doEnrichBookingsItems()` with prioritized bookings

7. **`PrioritizedBookings.groupSortedByEventId(bookings)`**
   - Filters bookings with slots and event IDs
   - Groups by event ID
   - Calls `PrioritizedBookings.toPrioritized()` for each group

8. **`PrioritizedBookings.groupSortedByScheduleId(bookings)`**
   - Filters bookings with schedules
   - Groups by schedule ID
   - Calls `PrioritizedBookings.toPrioritized()` for each group

9. **`PrioritizedBookings.toPrioritized(bookings)`**
   - Sorts bookings by creation date and ID
   - Assigns priority indices
   - Returns `PrioritizedBookings` instance

10. **`BookingsCatalogService.doEnrichBookingsItems(catalogReferences, bookingsById, classBookingsByEvent, courseBookingsBySchedule)`**
    - For each catalog reference, calls `enrichItem()` with prioritized bookings
    - Uses optimized availability checking

### Phase 3: Multi-Service Bookings Enrichment Path
11. **`BookingsCatalogService.enrichMultiServiceBookingsItems(catalogReferences, bookingIdsByMultiServiceBookingId, bookingsById)`**
    - Creates map of catalog references by ID
    - For each multi-service booking group, calls `enrichBookingsForMultiServiceBookingId()`
    - Sequences all futures and flattens results

12. **`BookingsCatalogService.enrichBookingsForMultiServiceBookingId(multiServiceBookingId, bookingIds, bookingsById, catalogReferencesById)`**
    - Gets bookings for the multi-service booking
    - Calls `availabilityAdapter.getAvailableSpotsForMultiServiceBooking()`
    - For each booking ID, calls `enrichItem()` with shared availability

13. **`AvailabilityAdapter.getAvailableSpotsForMultiServiceBooking(multiServiceBookingId, multiServiceBooking)`**
    - Checks if any booking requires validation
    - Checks if all bookings should ignore policy violations
    - Calls `getMultiServiceBookingAvailability()`
    - Calls `checkPartiallyRequestedMultiServiceBooking()`
    - Calls `getMultiServiceBookingAvailabilityOpenSpots()`

14. **`AvailabilityAdapter.getMultiServiceBookingAvailability(multiServiceBookingId)`**
    - Logs request
    - Calls `multiServiceBookings.getMultiServiceBookingAvailability()` (external RPC)
    - Logs response
    - Returns availability response

15. **`AvailabilityAdapter.checkPartiallyRequestedMultiServiceBooking(multiServiceBookingRequested, multiServiceBookingInfo)`**
    - Checks if requested bookings count < total bookings count
    - Returns boolean

16. **`AvailabilityAdapter.getMultiServiceBookingAvailabilityOpenSpots(ignorePolicyViolations, availability, multiServiceBookingPartiallyRequested)`**
    - Checks if bookable, has violations, or is partially requested
    - Returns remaining capacity or 0

### Phase 4: Individual Item Enrichment
17. **`BookingsCatalogService.enrichItem(bookingCatalogReference, bookingsById, multiServiceBookingAvailableSpots)`**
    - Gets booking from map
    - Creates futures for parallel execution:
      - `availabilityAdapter.getAvailableSpots()` - availability
      - `getSchedule()` - schedule info
      - `getPriceInfo()` - pricing
      - `getServiceV2()` - service details
      - `listCourseSessions()` - course sessions (conditional)
      - `getServiceVariants()` - service variants (conditional)
    - Checks feature toggles
    - Executes all futures in parallel
    - Calls `itemBuilder.bookingToItem()` to build final item

18. **`AvailabilityAdapter.getAvailableSpots(booking, multiServiceBookingAvailableSpots, ...)`**
    - If part of multi-service booking, returns provided spots
    - Otherwise calls `getAvailableSpotsForBooking()`

19. **`AvailabilityAdapter.getAvailableSpotsForBooking(maybeBooking, ...)`**
    - Calls `shouldValidateAvailability()` and `ignoreBookingPolicyViolations()`
    - If validation needed, calls `getBookingOpenSpots()`
    - Otherwise returns booking's participant count

20. **`AvailabilityAdapter.shouldValidateAvailability(booking)`**
    - Checks booking's flow control settings
    - Returns true if validation should occur

21. **`AvailabilityAdapter.ignoreBookingPolicyViolations(booking)`**
    - Checks booking's flow control settings
    - Returns true if violations should be ignored

22. **`AvailabilityAdapter.getBookingOpenSpots(booking, ignorePolicyViolations, ...)`**
    - If considering multiple bookings:
      - For slots: calls `getSlotAvailabilityOpenSpots()` with prioritized bookings
      - For schedules: calls `getScheduleAvailabilityOpenSpots()` with prioritized bookings
    - Otherwise:
      - For slots: calls `getSlotAvailabilityOpenSpots()` without prioritization
      - For schedules: calls `getScheduleAvailabilityOpenSpots()` without prioritization

23. **`AvailabilityAdapter.getSlotAvailabilityOpenSpots(slot, ignorePolicyViolations, booking, prioritizedBookings, shouldConsiderMultipleBookings)`**
    - Builds `GetSlotAvailabilityRequest`
    - Calls `getSlotAvailabilityFromService()`
    - Checks if slot is not bookable or has violations
    - If considering multiple bookings, calls `calculateOpenSpots()`
    - Otherwise returns open spots from availability

24. **`AvailabilityAdapter.getSlotAvailabilityFromService(request)`**
    - Logs request
    - Calls `bookings.getSlotAvailability()` (external RPC)
    - Recovers from errors, returning 0 spots
    - Logs response

25. **`AvailabilityAdapter.getScheduleAvailabilityOpenSpots(scheduleId, bookingId, ignorePolicyViolations, prioritizedBookings, shouldConsiderMultipleBookings)`**
    - Builds `GetScheduleAvailabilityRequest`
    - Calls `getAvailabilityScheduleFromBookingsService()`
    - Checks if schedule is empty or has violations
    - If considering multiple bookings, calls `calculateOpenSpots()`
    - Otherwise returns open spots from availability

26. **`AvailabilityAdapter.getAvailabilityScheduleFromBookingsService(request)`**
    - Logs request
    - Calls `bookings.getScheduleAvailability()` (external RPC)
    - Logs response

27. **`AvailabilityAdapter.calculateOpenSpots(bookingId, prioritizedBookings, slotAvailability, scheduleAvailability)`**
    - Gets open spots from availability
    - Gets participant count for booking
    - If single booking, checks if spots >= participants
    - If multiple bookings, calls `calculateRemainingSpots()`
    - Returns available spots or 0

28. **`AvailabilityAdapter.calculateRemainingSpots(totalOpenSpots, bookingId, prioritizedBookings)`**
    - Gets higher priority bookings
    - Calculates spots allocated to higher priority
    - Returns max(0, total - allocated)

29. **`PrioritizedBookings.getHigherPriorityBookings(bookingId)`**
    - Gets target booking's priority
    - Filters bookings with lower priority index
    - Returns sequence of higher priority bookings

30. **`BookingsCatalogService.getSchedule(maybeBooking)`**
    - Calls `getScheduleIdFromBooking()`
    - If schedule ID exists, calls `context.rpc.schedules.get()` (external RPC)
    - Returns schedule

31. **`BookingsCatalogService.getScheduleIdFromBooking(maybeBooking)`**
    - Extracts schedule ID from booking's booked entity
    - Returns schedule ID if available

32. **`BookingsCatalogService.getPriceInfo(booking)`**
    - Logs permissions check
    - Calls `context.rpc.bookingsPricingService.calculatePrice()` (external RPC)
    - Recovers from errors, returning None if service resolution fails

33. **`BookingsCatalogService.getServiceV2(maybeBooking)`**
    - Calls `BookingUtils.getServiceIdFromBooking()`
    - If service ID exists:
      - Calls `context.rpc.servicesService.getService()` (external RPC)
      - Calls `context.rpc.addOnGroupsService.listAddOnGroupsByServiceId()` (external RPC)
      - Maps service to domain using `ServiceV2Mappers.ServiceV2ToDomainTransformer`
      - Maps add-on groups using `ServiceV2Mappers.AddOnGroupDetailToDomainTransformer`
      - Combines service with add-on groups
    - Recovers from NotFound errors

34. **`BookingUtils.getServiceIdFromBooking(maybeBooking)`**
    - Extracts service ID from booking's booked entity
    - Returns service ID

35. **`BookingsCatalogService.getServiceVariants(maybeBooking)`**
    - Calls `BookingUtils.getServiceIdFromBooking()`
    - If service ID exists, calls `context.rpc.serviceOptionsAndVariantsService.getServiceOptionsAndVariantsByServiceId()` (external RPC)
    - Recovers from errors

36. **`BookingsCatalogService.listCourseSessions(maybeBooking)`**
    - If booking has schedule:
      - Builds query filter for schedule ID
      - Calls `context.rpc.eventsService.queryEvents()` (external RPC)
      - Returns events
    - Otherwise returns empty sequence

### Phase 5: Item Building
37. **`ItemBuilder.bookingToItem(bookingCatalogReference, booking, openSpotsForBooking, schedule, maybePriceInfo, maybeServiceV2domain, ...)`**
    - Wraps in Try for error handling
    - Gets service domain or throws exception
    - Calls `isDepositPriceInfo()`
    - Calls `resolvePaymentOption()`
    - Calls `validateBookedAddons()`
    - Calls `createAvailableItem()` or `createUnavailableItem()` on error

38. **`ItemBuilder.isDepositPriceInfo(priceInfo)`**
    - Checks if deposit amount > 0
    - Returns boolean

39. **`ItemBuilder.resolvePaymentOption(booking, isDeposit, serviceV2, shouldPopulatePricingPlanWithOfflinePayment)`**
    - Gets service payment options
    - If booking has undefined payment option:
      - Calls `validateUnselectedPaymentOption()`
      - Calls `resolveSelectedPaymentOption()`
    - Otherwise:
      - Calls `validateSelectedPaymentOption()`
    - Calls `isOfflineReminderExists()`
    - Calls `mapBookingsPaymentOptionToEcomPaymentOption()`

40. **`ItemBuilder.validateUnselectedPaymentOption(servicePaymentOptions, bookingId)`**
    - Checks if multiple options are available
    - Throws exception if multiple options exist

41. **`ItemBuilder.validateSelectedPaymentOption(booking, servicePaymentOptions)`**
    - Checks if selected option is available
    - Throws exception if unavailable

42. **`ItemBuilder.resolveSelectedPaymentOption(servicePaymentOptions, isCustomRate, bookingId)`**
    - Priority: Custom rate -> Online -> In-person -> Pricing plan
    - Throws exception if no options available

43. **`ItemBuilder.isOfflineReminderExists(service, booking, shouldPopulatePricingPlanWithOfflinePayment)`**
    - Checks add-on payment option
    - Checks if booking has add-ons with prices
    - Returns boolean

44. **`ItemBuilder.mapBookingsPaymentOptionToEcomPaymentOption(isDeposit, selectedPaymentOption, offlineReminderExists)`**
    - Maps booking payment option to ecommerce payment option type
    - Returns payment option type

45. **`ItemBuilder.validateBookedAddons(booking, serviceV2domain)`**
    - For each booked add-on:
      - Finds add-on group in service
      - Finds add-on in group
      - Validates add-on type (duration vs quantity)
      - Throws exceptions if invalid

46. **`ItemBuilder.createAvailableItem(...)`**
    - Calls `resolveServiceImage()`
    - Calls `calculateQuantityAvailable()`
    - Builds coupon scopes
    - Calls `createItem()`

47. **`ItemBuilder.resolveServiceImage(service)`**
    - Extracts image from service media
    - Returns image

48. **`ItemBuilder.calculateQuantityAvailable(booking, openSpotsForBooking)`**
    - Gets number of participants
    - Returns 1 if spots >= participants, 0 otherwise

49. **`ItemBuilder.createItem(...)`**
    - Calls `getProductName()`
    - Calls `getServiceProperties()`
    - Calls `priceInfoToItemPrice()`
    - Calls `priceInfoToPriceDescription()`
    - Calls `priceInfoToDepositAmount()`
    - Calls `isPriceZero()`
    - Calls `descriptionLinesBuilder.buildDescriptionLines()`
    - Calls `buildModifierGroups()`
    - Calls `mapBookingsTaxableAddressToEcomTaxableAddress()`
    - Creates and returns Item

50. **`ItemBuilder.getProductName(booking)`**
    - Extracts title from booking's booked entity
    - Returns product name with translations

51. **`ItemBuilder.getServiceProperties(booking)`**
    - Calls `getStartDate()`
    - Parses start date to timestamp
    - Returns service properties

52. **`ItemBuilder.getStartDate(booking)`**
    - Extracts start date from slot-based booking
    - Returns start date

53. **`ItemBuilder.priceInfoToItemPrice(priceInfo, paymentOptionType)`**
    - If not membership payment, extracts calculated price
    - Formats to 2 decimal places
    - Returns price string

54. **`ItemBuilder.priceInfoToPriceDescription(priceInfo)`**
    - Extracts price description from price info
    - Returns price description with translations

55. **`ItemBuilder.priceInfoToDepositAmount(priceInfo, paymentOptionType)`**
    - If deposit payment, extracts deposit amount
    - Formats to 2 decimal places
    - Returns deposit string

56. **`ItemBuilder.isPriceZero(price)`**
    - Parses price as BigDecimal
    - Returns true if zero

57. **`DescriptionLinesBuilder.buildDescriptionLines(booking, maybeServiceV2domain, serviceOptionsAndVariants, courseSessions, ...)`**
    - If should enrich (Wix Wellness):
      - Calls `getStartDate()`, `getStaffName()`, `getContactName()`
      - Creates enriched description lines
    - Otherwise if should populate:
      - Calls various generate methods:
        - `generateNumberOfParticipantsDescriptionLine()`
        - `generateBookingStatusDescriptionLine()`
        - `generateCustomRateDescriptionLine()`
        - `generateBookingStartDateDescriptionLine()`
        - `generateBookingDurationOrNumberOfSessionsDescriptionLine()`
        - `generateStaffDescriptionLine()`
        - `generateDynamicPriceDescriptionLine()`
        - `generateBookingLocationDescriptionLine()`
      - Flattens and returns description lines

58. **`ItemBuilder.buildModifierGroups(booking, service)`**
    - Groups booked add-ons by group ID
    - For each group, creates modifiers
    - Returns modifier groups

59. **`ItemBuilder.mapBookingsTaxableAddressToEcomTaxableAddress(maybeServiceV2domain)`**
    - Maps taxable address type to ecommerce format
    - Returns taxable address

### Phase 6: Response Assembly
60. **`BookingsCatalogService.getCatalogItems()` (continuation)**
    - Flattens enriched items (removes None values)
    - Exposes visibility event with results
    - Returns `GetCatalogItemsResponse` with items

## External RPC Calls
1. `context.rpc.bookings.consistentQuery()` - Fetch bookings
2. `context.rpc.bookings.getSlotAvailability()` - Get slot availability
3. `context.rpc.bookings.getScheduleAvailability()` - Get schedule availability
4. `context.rpc.multiServiceBookings.getMultiServiceBookingAvailability()` - Get multi-service booking availability
5. `context.rpc.schedules.get()` - Get schedule details
6. `context.rpc.bookingsPricingService.calculatePrice()` - Calculate price
7. `context.rpc.servicesService.getService()` - Get service details
8. `context.rpc.addOnGroupsService.listAddOnGroupsByServiceId()` - Get add-on groups
9. `context.rpc.serviceOptionsAndVariantsService.getServiceOptionsAndVariantsByServiceId()` - Get service variants
10. `context.rpc.eventsService.queryEvents()` - Get course sessions

## Feature Toggle Checks
1. `context.featureToggle.shouldPopulateDescriptionLines()` - Enable description lines
2. `context.featureToggle.shouldPopulateStaffDescriptionLineByResourceSelection()` - Staff description by resource
3. `context.featureToggle.populatePricingPlanWithOfflinePayment()` - Offline payment in pricing plan
4. `context.petri.conductSpecsPopulateDescriptionLinesWithStaffAndTime` - Wix Wellness enrichment

