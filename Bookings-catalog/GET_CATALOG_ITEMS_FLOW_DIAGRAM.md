# getCatalogItems() Data Flow Diagram

## Visual Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CLIENT REQUEST                                            │
│              GetCatalogItemsRequest                                          │
│              (catalogReferences: booking IDs)                               │
└────────────────────────┬────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  BookingsCatalogService.getCatalogItems()                                    │
│  ├─ log() - Log request                                                     │
│  ├─ logIsPermittedPriceCalculate() - Check permissions                     │
│  ├─ logCallScopeAspects() - Log context                                     │
│  └─ serverSigner.addServiceIdentity() - Sign call scope                    │
└────────────────────────┬────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  enrichAllBookingsItems()                                                    │
│  ├─ Expose metrics (item count)                                              │
│  ├─ fetchBookingsById() ───────────────────────────────────────┐           │
│  ├─ prepareBookingGroups() ────────────────────────────────────┤           │
│  │                                                               │           │
│  │  ┌───────────────────────────────────────────────────────────▼───┐      │
│  │  │ fetchBookingsById()                                           │      │
│  │  │  └─ For each booking ID:                                     │      │
│  │  │     └─ getBookingById()                                      │      │
│  │  │        └─ RPC: bookings.consistentQuery() ────┐             │      │
│  │  └────────────────────────────────────────────────┼─────────────┘      │
│  │                                                    │                    │
│  │  ┌────────────────────────────────────────────────▼─────────────┐      │
│  │  │ prepareBookingGroups()                                       │      │
│  │  │  └─ Group by multi-service booking ID                        │      │
│  │  │  └─ Partition into single vs multi-service                   │      │
│  │  └──────────────────────────────────────────────────────────────┘      │
│  │                                                                          │
│  └─ FutureUtil.inParallel(                                                 │
│     ├─ enrichBookingsItems() ────────────────┐                            │
│     └─ enrichMultiServiceBookingsItems() ────┼──────────────┐            │
│                                               │              │            │
└───────────────────────────────────────────────┼──────────────┼────────────┘
                                                │              │
                    ┌───────────────────────────┘              │
                    │                                          │
                    ▼                                          │
┌───────────────────────────────────────────────────────────────┐            │
│ enrichBookingsItems() (Single Bookings Path)                  │            │
│ ├─ PrioritizedBookings.groupSortedByEventId()                │            │
│ │  └─ toPrioritized() - Sort by creation date                │            │
│ ├─ PrioritizedBookings.groupSortedByScheduleId()             │            │
│ │  └─ toPrioritized() - Sort by creation date                │            │
│ └─ doEnrichBookingsItems()                                    │            │
│    └─ For each catalog reference:                             │            │
│       └─ enrichItem() ────────────────────────────┐          │            │
└────────────────────────────────────────────────────┼──────────┼────────────┘
                                                    │          │
                    ┌───────────────────────────────┘          │
                    │                                          │
                    ▼                                          │
┌───────────────────────────────────────────────────────────────┐            │
│ enrichMultiServiceBookingsItems() (Multi-Service Path)        │            │
│ └─ For each multi-service booking group:                      │            │
│    └─ enrichBookingsForMultiServiceBookingId()                │            │
│       ├─ availabilityAdapter.getAvailableSpotsForMultiServiceBooking() │   │
│       │  ├─ shouldValidateAvailability()                      │            │
│       │  ├─ ignoreBookingPolicyViolations()                   │            │
│       │  ├─ getMultiServiceBookingAvailability()             │            │
│       │  │  └─ RPC: multiServiceBookings.getMultiServiceBookingAvailability() │
│       │  ├─ checkPartiallyRequestedMultiServiceBooking()      │            │
│       │  └─ getMultiServiceBookingAvailabilityOpenSpots()     │            │
│       └─ For each booking:                                    │            │
│          └─ enrichItem() ────────────────────────────┐        │            │
└───────────────────────────────────────────────────────┼────────┼────────────┘
                                                       │        │
                    ┌──────────────────────────────────┘        │
                    │                                          │
                    ▼                                          │
┌───────────────────────────────────────────────────────────────┐            │
│ enrichItem() - Individual Item Enrichment                     │            │
│                                                               │            │
│ Creates parallel futures:                                     │            │
│                                                               │            │
│ ┌─────────────────────────────────────────────────────────┐  │            │
│ │ availabilityAdapter.getAvailableSpots()                  │  │            │
│ │ ├─ isPartOfMultiServiceBooking()                        │  │            │
│ │ └─ getAvailableSpotsForBooking()                        │  │            │
│ │    ├─ shouldValidateAvailability()                      │  │            │
│ │    ├─ ignoreBookingPolicyViolations()                  │  │            │
│ │    └─ getBookingOpenSpots()                             │  │            │
│ │       ├─ getSlotAvailabilityOpenSpots()                 │  │            │
│ │       │  ├─ getSlotAvailabilityFromService()            │  │            │
│ │       │  │  └─ RPC: bookings.getSlotAvailability()      │  │            │
│ │       │  ├─ isSlotNotBookable()                         │  │            │
│ │       │  ├─ isSlotLocked()                              │  │            │
│ │       │  ├─ slotHasPolicyViolations()                   │  │            │
│ │       │  └─ calculateOpenSpots()                        │  │            │
│ │       │     ├─ PrioritizedBookings.getHigherPriorityBookings() │      │
│ │       │     └─ calculateRemainingSpots()                │  │            │
│ │       └─ getScheduleAvailabilityOpenSpots()              │  │            │
│ │          ├─ getAvailabilityScheduleFromBookingsService() │  │            │
│ │          │  └─ RPC: bookings.getScheduleAvailability()  │  │            │
│ │          ├─ anyPolicyViolations()                       │  │            │
│ │          └─ calculateOpenSpots()                        │  │            │
│ └─────────────────────────────────────────────────────────┘  │            │
│                                                               │            │
│ ┌─────────────────────────────────────────────────────────┐  │            │
│ │ getSchedule()                                           │  │            │
│ │ ├─ getScheduleIdFromBooking()                          │  │            │
│ │ └─ RPC: schedules.get()                                │  │            │
│ └─────────────────────────────────────────────────────────┘  │            │
│                                                               │            │
│ ┌─────────────────────────────────────────────────────────┐  │            │
│ │ getPriceInfo()                                           │  │            │
│ │ └─ RPC: bookingsPricingService.calculatePrice()         │  │            │
│ └─────────────────────────────────────────────────────────┘  │            │
│                                                               │            │
│ ┌─────────────────────────────────────────────────────────┐  │            │
│ │ getServiceV2()                                           │  │            │
│ │ ├─ BookingUtils.getServiceIdFromBooking()               │  │            │
│ │ ├─ RPC: servicesService.getService()                     │  │            │
│ │ │  └─ ServiceV2Mappers.ServiceV2ToDomainTransformer     │  │            │
│ │ ├─ RPC: addOnGroupsService.listAddOnGroupsByServiceId()  │  │            │
│ │ │  └─ ServiceV2Mappers.AddOnGroupDetailToDomainTransformer │            │
│ │ └─ Combine service + add-on groups                       │  │            │
│ └─────────────────────────────────────────────────────────┘  │            │
│                                                               │            │
│ ┌─────────────────────────────────────────────────────────┐  │            │
│ │ listCourseSessions() (conditional)                      │  │            │
│ │ └─ RPC: eventsService.queryEvents()                     │  │            │
│ └─────────────────────────────────────────────────────────┘  │            │
│                                                               │            │
│ ┌─────────────────────────────────────────────────────────┐  │            │
│ │ getServiceVariants() (conditional)                      │  │            │
│ │ ├─ BookingUtils.getServiceIdFromBooking()               │  │            │
│ │ └─ RPC: serviceOptionsAndVariantsService.getServiceOptionsAndVariantsByServiceId() │
│ └─────────────────────────────────────────────────────────┘  │            │
│                                                               │            │
│ └─ FutureUtil.inParallel(all futures)                        │            │
│                                                               │            │
│    └─ itemBuilder.bookingToItem() ────────────────────────┐  │            │
└────────────────────────────────────────────────────────────┼─┼────────────┘
                                                              │ │
                    ┌─────────────────────────────────────────┘ │
                    │                                           │
                    ▼                                           │
┌───────────────────────────────────────────────────────────────┐            │
│ ItemBuilder.bookingToItem() - Build Catalog Item              │            │
│                                                               │            │
│ ┌─────────────────────────────────────────────────────────┐  │            │
│ │ Validation & Payment Resolution                          │  │            │
│ │ ├─ isDepositPriceInfo()                                 │  │            │
│ │ ├─ resolvePaymentOption()                               │  │            │
│ │ │  ├─ validateUnselectedPaymentOption()                 │  │            │
│ │ │  ├─ validateSelectedPaymentOption()                   │  │            │
│ │ │  ├─ resolveSelectedPaymentOption()                    │  │            │
│ │ │  ├─ isOfflineReminderExists()                         │  │            │
│ │ │  └─ mapBookingsPaymentOptionToEcomPaymentOption()     │  │            │
│ │ └─ validateBookedAddons()                                │  │            │
│ └─────────────────────────────────────────────────────────┘  │            │
│                                                               │            │
│ ┌─────────────────────────────────────────────────────────┐  │            │
│ │ createAvailableItem() or createUnavailableItem()        │  │            │
│ │ ├─ resolveServiceImage()                                │  │            │
│ │ ├─ calculateQuantityAvailable()                         │  │            │
│ │ └─ createItem()                                         │  │            │
│ │    ├─ getProductName()                                  │  │            │
│ │    ├─ getServiceProperties()                            │  │            │
│ │    │  └─ getStartDate()                                 │  │            │
│ │    ├─ priceInfoToItemPrice()                            │  │            │
│ │    ├─ priceInfoToPriceDescription()                     │  │            │
│ │    ├─ priceInfoToDepositAmount()                        │  │            │
│ │    ├─ isPriceZero()                                     │  │            │
│ │    ├─ descriptionLinesBuilder.buildDescriptionLines()   │  │            │
│ │    │  ├─ getStartDate()                                 │  │            │
│ │    │  ├─ getStaffName()                                 │  │            │
│ │    │  ├─ getContactName()                               │  │            │
│ │    │  ├─ generateNumberOfParticipantsDescriptionLine()   │  │            │
│ │    │  ├─ generateBookingStatusDescriptionLine()         │  │            │
│ │    │  ├─ generateCustomRateDescriptionLine()            │  │            │
│ │    │  ├─ generateBookingStartDateDescriptionLine()      │  │            │
│ │    │  │  └─ BookingsCatalogTranslator.localeDateFormatter() │          │
│ │    │  ├─ generateBookingDurationOrNumberOfSessionsDescriptionLine() │   │
│ │    │  │  ├─ TimeFormatter.calculateBookingDuration()    │  │            │
│ │    │  │  └─ generateNumberOfSessionsMessage()            │  │            │
│ │    │  ├─ generateStaffDescriptionLine()                  │  │            │
│ │    │  │  ├─ getStaffSelectionMethod()                   │  │            │
│ │    │  │  └─ getStaffNameDescriptionLine()                │  │            │
│ │    │  ├─ generateDynamicPriceDescriptionLine()           │  │            │
│ │    │  │  └─ mapDynamicCustomParticipants()               │  │            │
│ │    │  └─ generateBookingLocationDescriptionLine()         │  │            │
│ │    ├─ buildModifierGroups()                              │  │            │
│ │    └─ mapBookingsTaxableAddressToEcomTaxableAddress()    │  │            │
│ └─────────────────────────────────────────────────────────┘  │            │
└───────────────────────────────────────────────────────────────┼────────────┘
                                                                 │
                    ┌───────────────────────────────────────────┘
                    │
                    ▼
┌───────────────────────────────────────────────────────────────┐
│ Response Assembly                                              │
│ ├─ Flatten enriched items (remove None)                       │
│ ├─ Expose visibility event                                     │
│ └─ GetCatalogItemsResponse(items)                              │
└────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────┐
│                    CLIENT RESPONSE                              │
│              GetCatalogItemsResponse                            │
│              (items: Seq[Item])                                 │
└────────────────────────────────────────────────────────────────┘
```

## Parallel Execution Points

1. **Initial Fetching**: `enrichBookingsItems()` and `enrichMultiServiceBookingsItems()` run in parallel
2. **Item Enrichment**: For each item, these run in parallel:
   - Availability check
   - Schedule fetch
   - Price calculation
   - Service details fetch
   - Course sessions fetch (conditional)
   - Service variants fetch (conditional)
3. **Multi-Service Booking**: All bookings in a multi-service booking are enriched in parallel after getting shared availability

## Key Data Transformations

1. **Booking → PrioritizedBookings**: Bookings sorted by creation date and grouped by event/schedule
2. **Booking → ServiceV2Domain**: Protobuf service mapped to domain model with AutoMapper
3. **Booking → Item**: Complete transformation with pricing, availability, description lines, modifiers
4. **PriceInfo → Payment Options**: Payment info converted to ecommerce payment option types
5. **Service + Add-ons → Modifier Groups**: Service add-ons transformed to ecommerce modifier groups

## Error Handling Points

1. **Booking Not Found**: Returns None, item excluded from response
2. **Service Not Found**: Returns None, item created as unavailable
3. **Price Calculation Failure**: Returns None, item created with zero price
4. **Add-on Validation Failure**: Item created as unavailable (quantity = 0)
5. **Payment Option Validation Failure**: Item created as unavailable
6. **Availability Check Failure**: Returns 0 spots, item created as unavailable

## Performance Optimizations

1. **Prioritized Bookings**: Groups bookings by event/schedule to optimize availability checks
2. **Parallel Execution**: Multiple RPC calls executed in parallel
3. **Conditional Fetching**: Course sessions and variants only fetched if feature toggle enabled
4. **Multi-Service Booking**: Shared availability check for all bookings in a group

