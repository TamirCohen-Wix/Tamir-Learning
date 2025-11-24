# Bookings Catalog Service

## Table of Contents
1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Service Classes](#service-classes)
4. [Module Structure](#module-structure)
5. [Data Flow](#data-flow)
6. [API Endpoints](#api-endpoints)
7. [External Dependencies](#external-dependencies)
8. [Feature Toggles](#feature-toggles)
9. [Error Handling](#error-handling)
10. [Testing](#testing)

---

## Project Overview

The **Bookings Catalog Service** is a Scala microservice built with Wix's Loom-Prime framework that implements the **Platform Catalog SPI** for bookings. It transforms booking data into ecommerce catalog items with pricing, availability, and description lines.

### Purpose
- Convert bookings into ecommerce catalog items
- Enrich bookings with pricing, availability, schedules, and service details
- Support both single bookings and multi-service bookings
- Provide internationalized description lines
- Integrate with multiple internal services (pricing, schedules, services, events)

### Key Features
- ✅ Real-time availability checking
- ✅ Dynamic pricing calculation
- ✅ Multi-service booking support
- ✅ Internationalization (i18n) support
- ✅ Description lines generation
- ✅ Payment option resolution
- ✅ Add-on validation and transformation
- ✅ Optimized parallel data fetching

---

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Platform Catalog SPI                          │
│                    (Ecommerce Platform)                          │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ gRPC
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              BookingsCatalogService                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Main Service Layer                                       │   │
│  │  - getCatalogItems()                                      │   │
│  │  - queryCatalogItems()                                   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Adapters Layer                                           │   │
│  │  - BookingsAdapter                                        │   │
│  │  - ServicesAdapter                                        │   │
│  │  - EventsAdapter                                          │   │
│  │  - ServiceOptionsAndVariantsAdapter                       │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Mappers Layer                                           │   │
│  │  - ItemBuilder                                           │   │
│  │  - DescriptionLinesBuilder                                │   │
│  │  - ServiceV2Mappers                                       │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Utils Layer                                              │   │
│  │  - AvailabilityAdapter                                    │   │
│  │  - BookingUtils                                           │   │
│  │  - PrioritizedBookings                                    │   │
│  │  - BookingsCatalogTranslator                              │   │
│  │  - TimeFormatter                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Domain Models                                            │   │
│  │  - ServiceV2Domain                                        │   │
│  │  - AddOnGroupDetailDomain                                 │   │
│  │  - AddOnDomain                                            │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ RPC Calls
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  Bookings     │   │  Pricing      │   │  Schedules   │
│  Service      │   │  Service      │   │  Service      │
└───────────────┘   └───────────────┘   └───────────────┘
        │                    │                    │
        └────────────────────┼────────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  Services     │   │  Events       │   │  Multi-Service│
│  Service      │   │  Service      │   │  Bookings     │
└───────────────┘   └───────────────┘   └───────────────┘
```

### Design Patterns

1. **Adapter Pattern**: Wraps external service clients to provide clean interfaces
2. **Builder Pattern**: Constructs complex objects (Item, DescriptionLines) step by step
3. **Mapper Pattern**: Transforms between protobuf messages and domain objects
4. **Domain-Driven Design**: Pure domain models independent of infrastructure

---

## Service Classes

### Primary Services

#### `BookingsCatalogService`
**Location**: `src/com/wixpress/bookings/bookings/catalog/BookingsCatalogService.scala`

Main service implementing the Platform Catalog SPI. Handles:
- Fetching and enriching catalog items
- Separating single vs multi-service bookings
- Coordinating parallel data fetching
- Error handling and logging

**Key Methods**:
- `getCatalogItems(request: GetCatalogItemsRequest)`: Main entry point
- `queryCatalogItems(request: QueryCatalogItemsRequest)`: Query by service properties
- `enrichAllBookingsItems()`: Orchestrates enrichment process
- `enrichItem()`: Enriches individual booking items

#### `DescriptionLinesService`
**Location**: `src/com/wixpress/bookings/bookings/catalog/DescriptionLinesService.scala`

Secondary service for retrieving description lines for bookings. Provides human-readable information about bookings.

**Key Methods**:
- `listBookingDescriptionLines(request: ListBookingDescriptionLinesRequest)`: Lists description lines for a booking

---

## Module Structure

### Adapters (`adapters/`)

Adapters wrap external service clients to provide clean, testable interfaces.

#### `BookingsAdapter`
- **Purpose**: Interface to the Bookings service
- **Key Methods**:
  - `getBookingById(bookingId: String)`: Fetches a booking by ID

#### `ServicesAdapter`
- **Purpose**: Interface to the Services service
- **Key Methods**:
  - `getServiceV2(serviceId: String)`: Fetches service details with add-on groups

#### `EventsAdapter`
- **Purpose**: Interface to the Events service
- **Key Methods**:
  - `listCourseSessions(booking: Option[Booking])`: Lists course sessions for a schedule

#### `ServiceOptionsAndVariantsAdapter`
- **Purpose**: Interface to the Service Options and Variants service
- **Key Methods**:
  - `getServiceVariants(serviceId: String)`: Fetches service variants for dynamic pricing

### Mappers (`mappers/`)

Mappers transform data between different representations.

#### `ItemBuilder`
- **Purpose**: Builds ecommerce catalog items from booking data
- **Key Methods**:
  - `bookingToItem()`: Main transformation method
  - `createAvailableItem()`: Creates item with availability
  - `createUnavailableItem()`: Creates item without availability
  - `resolvePaymentOption()`: Resolves payment options
  - `validateBookedAddons()`: Validates add-ons

#### `DescriptionLinesBuilder`
- **Purpose**: Builds description lines for catalog items
- **Key Methods**:
  - `buildDescriptionLines()`: Main builder method
  - `generateBookingStartDateDescriptionLine()`: Date/time description
  - `generateBookingDurationOrNumberOfSessionsDescriptionLine()`: Duration/sessions
  - `generateStaffDescriptionLine()`: Staff information
  - `generateDynamicPriceDescriptionLine()`: Dynamic pricing info

#### `ServiceV2Mappers`
- **Purpose**: AutoMapper transformers for service domain models
- **Transformers**:
  - `ServiceV2ToDomainTransformer`: Service protobuf → domain
  - `AddOnGroupDetailToDomainTransformer`: Add-on group protobuf → domain

### Domain Models (`domain/`)

Pure Scala case classes representing business entities.

#### `ServiceV2Domain`
- Represents a service with all its details
- Fields: id, servicePageUrl, paymentDomain, media, addOnGroups, taxableAddress

#### `AddOnGroupDetailDomain`
- Represents an add-on group with its add-ons
- Fields: groupId, groupName, addOns

#### `AddOnDomain`
- Represents an individual add-on
- Fields: addOnId, name, price, addOnInfo

#### `TaxableAddressDomain`
- Represents taxable address information
- Fields: taxableAddressType

### Utils (`utils/`)

Stateless utility functions for common operations.

#### `AvailabilityAdapter`
- **Purpose**: Checks booking and multi-service booking availability
- **Key Methods**:
  - `getAvailableSpots()`: Gets available spots for a booking
  - `getAvailableSpotsForMultiServiceBooking()`: Gets availability for multi-service bookings
  - `getSlotAvailabilityOpenSpots()`: Checks slot availability
  - `getScheduleAvailabilityOpenSpots()`: Checks schedule availability

#### `BookingUtils`
- **Purpose**: Utility functions for extracting booking information
- **Key Methods**:
  - `getServiceIdFromBooking()`: Extracts service ID from booking

#### `PrioritizedBookings`
- **Purpose**: Manages prioritized bookings for optimized availability checks
- **Key Methods**:
  - `groupSortedByEventId()`: Groups bookings by event ID
  - `groupSortedByScheduleId()`: Groups bookings by schedule ID
  - `toPrioritized()`: Creates prioritized booking groups

#### `BookingsCatalogTranslator`
- **Purpose**: Handles translations and date formatting
- **Key Methods**:
  - `localeDateFormatter()`: Formats dates by locale
  - `getTranslatedLabel()`: Gets translated labels

#### `TimeFormatter`
- **Purpose**: Date/time parsing and duration formatting
- **Key Methods**:
  - `calculateBookingDuration()`: Calculates booking duration
  - `parseDate()`: Parses date strings

#### `ResourceKeys`
- **Purpose**: Constants for translation resource keys

#### `TranslatedString`
- **Purpose**: Simple case class for user and UOU translations

#### `BookingsCatalogVisibilityEvents`
- **Purpose**: Custom visibility events for logging and metrics

### Errors (`errors/`)

Custom exceptions for the bookings catalog domain.

#### `BookingsCatalogExceptions`
- `FailedToResolveServiceException`: Service cannot be resolved
- `UnavailableSelectedPaymentOptionException`: Payment option unavailable
- `UndefinedSelectedPaymentOptionException`: Payment option not defined
- `NoAvailablePaymentOptionsException`: No payment options available
- `AddOnNotFoundException`: Add-on not found
- `InvalidAddOnException`: Add-on is invalid

---

## Data Flow

### Complete Request Flow for `getCatalogItems()`

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CLIENT REQUEST                                            │
│              GetCatalogItemsRequest                                           │
│              (catalogReferences: booking IDs)                                │
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
│  │  └─ For each booking ID:                                   │           │
│  │     └─ getBookingById()                                    │           │
│  │        └─ RPC: bookings.consistentQuery()                  │           │
│  ├─ prepareBookingGroups()                                    │           │
│  │  └─ Group by multi-service booking ID                     │           │
│  │  └─ Partition into single vs multi-service                │           │
│  └─ FutureUtil.inParallel(                                    │           │
│     ├─ enrichBookingsItems() ────────────────┐              │           │
│     └─ enrichMultiServiceBookingsItems() ────┼──────────────┐          │
│                                               │              │           │
└───────────────────────────────────────────────┼──────────────┼───────────┘
                                                │              │
                    ┌───────────────────────────┘              │
                    │                                          │
                    ▼                                          │
┌───────────────────────────────────────────────────────────────┐            │
│ enrichBookingsItems() (Single Bookings Path)                 │            │
│ ├─ PrioritizedBookings.groupSortedByEventId()               │            │
│ ├─ PrioritizedBookings.groupSortedByScheduleId()            │            │
│ └─ doEnrichBookingsItems()                                   │            │
│    └─ For each catalog reference:                            │            │
│       └─ enrichItem() ────────────────────────────┐         │            │
└────────────────────────────────────────────────────┼─────────┼───────────┘
                                                    │         │
                    ┌───────────────────────────────┘         │
                    │                                          │
                    ▼                                          │
┌───────────────────────────────────────────────────────────────┐            │
│ enrichMultiServiceBookingsItems() (Multi-Service Path)        │            │
│ └─ For each multi-service booking group:                     │            │
│    └─ enrichBookingsForMultiServiceBookingId()                │            │
│       ├─ availabilityAdapter.getAvailableSpotsForMultiServiceBooking() │  │
│       │  └─ RPC: multiServiceBookings.getMultiServiceBookingAvailability() │
│       └─ For each booking:                                    │            │
│          └─ enrichItem() ────────────────────────────┐      │            │
└───────────────────────────────────────────────────────┼────────┼──────────┘
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
│ │ ├─ getAvailableSpotsForBooking()                         │  │            │
│ │ └─ getBookingOpenSpots()                                 │  │            │
│ │    ├─ getSlotAvailabilityOpenSpots()                    │  │            │
│ │    │  └─ RPC: bookings.getSlotAvailability()             │  │            │
│ │    └─ getScheduleAvailabilityOpenSpots()                  │  │            │
│ │       └─ RPC: bookings.getScheduleAvailability()         │  │            │
│ └─────────────────────────────────────────────────────────┘  │            │
│                                                               │            │
│ ┌─────────────────────────────────────────────────────────┐  │            │
│ │ getSchedule()                                            │  │            │
│ │ └─ RPC: schedules.get()                                  │  │            │
│ └─────────────────────────────────────────────────────────┘  │            │
│                                                               │            │
│ ┌─────────────────────────────────────────────────────────┐  │            │
│ │ getPriceInfo()                                           │  │            │
│ │ └─ RPC: bookingsPricingService.calculatePrice()          │  │            │
│ └─────────────────────────────────────────────────────────┘  │            │
│                                                               │            │
│ ┌─────────────────────────────────────────────────────────┐  │            │
│ │ getServiceV2()                                           │  │            │
│ │ ├─ RPC: servicesService.getService()                    │  │            │
│ │ └─ RPC: addOnGroupsService.listAddOnGroupsByServiceId() │  │            │
│ └─────────────────────────────────────────────────────────┘  │            │
│                                                               │            │
│ ┌─────────────────────────────────────────────────────────┐  │            │
│ │ listCourseSessions() (conditional)                      │  │            │
│ │ └─ RPC: eventsService.queryEvents()                     │  │            │
│ └─────────────────────────────────────────────────────────┘  │            │
│                                                               │            │
│ ┌─────────────────────────────────────────────────────────┐  │            │
│ │ getServiceVariants() (conditional)                       │  │            │
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
│ │ └─ validateBookedAddons()                               │  │            │
│ └─────────────────────────────────────────────────────────┘  │            │
│                                                               │            │
│ ┌─────────────────────────────────────────────────────────┐  │            │
│ │ createAvailableItem()                                   │  │            │
│ │ ├─ resolveServiceImage()                                │  │            │
│ │ ├─ calculateQuantityAvailable()                         │  │            │
│ │ └─ createItem()                                         │  │            │
│ │    ├─ getProductName()                                  │  │            │
│ │    ├─ getServiceProperties()                            │  │            │
│ │    ├─ priceInfoToItemPrice()                            │  │            │
│ │    ├─ priceInfoToPriceDescription()                     │  │            │
│ │    ├─ priceInfoToDepositAmount()                        │  │            │
│ │    ├─ descriptionLinesBuilder.buildDescriptionLines()   │  │            │
│ │    ├─ buildModifierGroups()                             │  │            │
│ │    └─ mapBookingsTaxableAddressToEcomTaxableAddress()   │  │            │
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

### Parallel Execution Points

1. **Initial Fetching**: `enrichBookingsItems()` and `enrichMultiServiceBookingsItems()` run in parallel
2. **Item Enrichment**: For each item, these run in parallel:
   - Availability check
   - Schedule fetch
   - Price calculation
   - Service details fetch
   - Course sessions fetch (conditional)
   - Service variants fetch (conditional)
3. **Multi-Service Booking**: All bookings in a multi-service booking are enriched in parallel after getting shared availability

### Key Data Transformations

1. **Booking → PrioritizedBookings**: Bookings sorted by creation date and grouped by event/schedule
2. **Booking → ServiceV2Domain**: Protobuf service mapped to domain model with AutoMapper
3. **Booking → Item**: Complete transformation with pricing, availability, description lines, modifiers
4. **PriceInfo → Payment Options**: Payment info converted to ecommerce payment option types
5. **Service + Add-ons → Modifier Groups**: Service add-ons transformed to ecommerce modifier groups

---

## API Endpoints

### Primary Service: Platform Catalog SPI

#### `getCatalogItems`
- **Request**: `GetCatalogItemsRequest`
  - `catalogReferences`: List of booking IDs
- **Response**: `GetCatalogItemsResponse`
  - `items`: List of enriched catalog items
- **Purpose**: Retrieves and enriches catalog items for given booking IDs
- **Flow**: See [Data Flow](#data-flow) section above

#### `queryCatalogItems`
- **Request**: `QueryCatalogItemsRequest`
- **Response**: `QueryCatalogItemsResponse`
  - `items`: List of basic catalog items (services)
- **Purpose**: Queries catalog items by service properties (not specific bookings)
- **Note**: Currently returns basic items only (feature toggle controlled)

### Secondary Service: Description Lines Service

#### `listBookingDescriptionLines`
- **Request**: `ListBookingDescriptionLinesRequest`
  - `booking_id`: The booking ID
- **Response**: `ListBookingDescriptionLinesResponse`
  - `description_lines`: List of description lines
- **Purpose**: Retrieves human-readable description lines for a booking

---

## External Dependencies

### Internal Services (RPC)

1. **Bookings Service** (`com.wixpress.bookings.bookings.v2.Bookings`)
   - `consistentQuery()`: Fetch bookings by ID
   - `getSlotAvailability()`: Check slot availability
   - `getScheduleAvailability()`: Check schedule availability

2. **Multi-Service Bookings Service** (`com.wixpress.bookings.bookings.v2.MultiServiceBookings`)
   - `getMultiServiceBookingAvailability()`: Check multi-service booking availability

3. **Schedules Service** (`com.wix.bookings.schedules.api.v1.Schedules`)
   - `get()`: Get schedule details

4. **Pricing Service** (`com.wixpress.bookings.pricing.BookingsPricingService`)
   - `calculatePrice()`: Calculate booking price

5. **Services Service** (`wix.bookings.services.v2.ServicesService`)
   - `getService()`: Get service details
   - `queryServices()`: Query services by app ID

6. **Add-On Groups Service** (`wix.bookings.services.v2.AddOnGroupsService`)
   - `listAddOnGroupsByServiceId()`: Get add-on groups for a service

7. **Service Options and Variants Service** (`wix.bookings.catalog.v1.ServiceOptionsAndVariantsService`)
   - `getServiceOptionsAndVariantsByServiceId()`: Get service variants

8. **Events Service** (`wix.calendar.events.v3.EventsService`)
   - `queryEvents()`: Query course sessions

### External Libraries

#### Core Frameworks
- **Loom-Prime**: Wix's microservice framework
- **gRPC**: RPC framework for service communication
- **Protocol Buffers**: Data serialization

#### Transformation & Mapping
- **AutoMapper**: Object transformation library
- **Struct Builder**: Protobuf struct construction

#### Internationalization
- **Babel**: Translation and localization framework
- **Wix Localization**: Locale handling

#### Error Handling
- **Wix Error Framework**: Standardized error handling

#### Logging & Metrics
- **Visibility Framework**: Logging and metrics
- **Custom Logger**: Structured logging

#### Feature Toggles
- **Petri/Wix Laboratory**: Feature toggle framework

#### Utilities
- **Joda Time**: Date/time handling
- **Hoopoe JSON**: JSON mapping
- **Simple Data Layer (SDL)**: Data layer annotations

---

## Feature Toggles

### Available Feature Toggles

1. **`shouldPopulateDescriptionLines`**
   - **Purpose**: Enable/disable description lines population
   - **Impact**: Controls whether course sessions and service variants are fetched

2. **`populatePricingPlanWithOfflinePayment`**
   - **Purpose**: Include offline payment in pricing plan
   - **Impact**: Affects payment option resolution

3. **`shouldPopulateStaffDescriptionLineByResourceSelection`**
   - **Purpose**: Populate staff description line based on resource selection
   - **Impact**: Affects staff description line generation

4. **`enableQueryCatalogItems`**
   - **Purpose**: Enable query catalog items endpoint
   - **Impact**: Controls whether `queryCatalogItems` returns data

### Petri Experiments

- **`PopulateDescriptionLinesWithStaffAndTime`**: Wix Wellness specific enrichment

---

## Error Handling

### Error Handling Strategy

1. **Graceful Degradation**: Missing optional data (service, price) results in `None`, item still created
2. **Validation Errors**: Invalid data (add-ons, payment options) results in unavailable item (quantity = 0)
3. **Critical Errors**: Service resolution failures logged and handled gracefully
4. **RPC Errors**: Recovered with appropriate fallbacks

### Error Types

#### Recoverable Errors
- Service not found → Returns `None`, item created as unavailable
- Price calculation failure → Returns `None`, item created with zero price
- Add-on validation failure → Item created as unavailable

#### Non-Recoverable Errors
- Booking not found → Item excluded from response
- Critical validation failures → Item created as unavailable with error logging

### Error Logging

All errors are logged using the Visibility framework:
- `GenericEvent(Log.Error, ...)`: Error events
- `BookingsCatalogErrorLogWithMetricEvent`: Errors with associated metrics
- Structured logging with context

---

## Testing

### Test Structure

Tests are located in `test/com/wixpress/bookings/bookings/catalog/`:

- **`BookingsCatalogBaseTest`**: Base test class with common setup
- **`BookingsCatalogServiceTest`**: Main service tests
- **`DescriptionLinesServiceTest`**: Description lines service tests
- **`BookingsCatalogAvailabilityValidationTest`**: Availability validation tests
- **`BookingsCatalogMultiServiceBookingsAvailabilityValidationTest`**: Multi-service booking tests
- **`BookingsCatalogModifiersTest`**: Modifier/add-on tests
- **`BookingsCatalogPaymentOptionTest`**: Payment option tests

### Test Utilities

- Wix testkit utilities for mocking
- Feature toggle testing support
- RPC client mocking

---

## Class Reference

### Service Layer

| Class | Location | Purpose |
|-------|----------|---------|
| `BookingsCatalogService` | `BookingsCatalogService.scala` | Main catalog service |
| `DescriptionLinesService` | `DescriptionLinesService.scala` | Description lines service |

### Adapters

| Class | Location | Purpose |
|-------|----------|---------|
| `BookingsAdapter` | `adapters/BookingsAdapter.scala` | Bookings service adapter |
| `ServicesAdapter` | `adapters/ServicesAdapter.scala` | Services service adapter |
| `EventsAdapter` | `adapters/EventsAdapter.scala` | Events service adapter |
| `ServiceOptionsAndVariantsAdapter` | `adapters/ServiceOptionsAndVariantsAdapter.scala` | Service variants adapter |

### Mappers

| Class | Location | Purpose |
|-------|----------|---------|
| `ItemBuilder` | `mappers/ItemBuilder.scala` | Builds catalog items |
| `DescriptionLinesBuilder` | `mappers/DescriptionLinesBuilder.scala` | Builds description lines |
| `ServiceV2Mappers` | `mappers/ServiceV2Mappers.scala` | Service domain mappers |

### Domain Models

| Class | Location | Purpose |
|-------|----------|---------|
| `ServiceV2Domain` | `domain/ServiceV2Domain.scala` | Service domain model |
| `AddOnGroupDetailDomain` | `domain/ServiceV2Domain.scala` | Add-on group domain |
| `AddOnDomain` | `domain/ServiceV2Domain.scala` | Add-on domain |
| `TaxableAddressDomain` | `domain/ServiceV2Domain.scala` | Taxable address domain |

### Utils

| Class | Location | Purpose |
|-------|----------|---------|
| `AvailabilityAdapter` | `utils/AvailabilityAdapter.scala` | Availability checking |
| `BookingUtils` | `utils/BookingUtils.scala` | Booking utilities |
| `PrioritizedBookings` | `utils/PrioritizedBookings.scala` | Prioritized booking groups |
| `BookingsCatalogTranslator` | `utils/BookingsCatalogTranslator.scala` | Translation utilities |
| `TimeFormatter` | `utils/TimeFormatter.scala` | Date/time formatting |
| `ResourceKeys` | `utils/ResourceKeys.scala` | Translation keys |
| `TranslatedString` | `utils/TranslatedString.scala` | Translation wrapper |
| `BookingsCatalogVisibilityEvents` | `utils/BookingsCatalogVisibilityEvents.scala` | Visibility events |

### Errors

| Class | Location | Purpose |
|-------|----------|---------|
| `BookingsCatalogExceptions` | `errors/BookingsCatalogExceptions.scala` | Custom exceptions |

---

## Performance Optimizations

1. **Parallel Execution**: Multiple RPC calls executed in parallel using `FutureUtil.inParallel`
2. **Prioritized Bookings**: Groups bookings by event/schedule to optimize availability checks
3. **Conditional Fetching**: Course sessions and variants only fetched if feature toggle enabled
4. **Multi-Service Booking Optimization**: Shared availability check for all bookings in a group
5. **Lazy Evaluation**: Futures evaluated only when needed

---

## Internationalization

The service supports internationalization through:
- **Babel Translator**: Translation service with resource files
- **Translation Resources**: Located in `resources/catalog-translations/`
- **Supported Locales**: 30+ languages (en, es, fr, de, etc.)
- **User and UOU Translations**: Supports both user and unit-of-use translations

---

## Security

1. **Service Identity**: All cross-service calls use `serverSigner.addServiceIdentity()`
2. **Authorization Checks**: Permission checks for price calculation
3. **Input Validation**: All input data validated
4. **Error Sanitization**: Internal errors not exposed to clients

---

## Build & Deployment

### Build System
- **Bazel**: Build system
- **Build File**: `BUILD.bazel`

### Entry Point
- **Service**: `com.wixpress.bookings.bookings.catalog.BookingsCatalogService`
- **Artifact**: `com.wixpress.bookings.bookings.catalog.bookings-catalog`

### Runtime Dependencies
- Translation resources: `resources/catalog-translations/`

---

## Summary

The Bookings Catalog Service is a sophisticated microservice that:

1. **Transforms** booking data into ecommerce catalog items
2. **Enriches** bookings with pricing, availability, and service details
3. **Supports** both single and multi-service bookings
4. **Provides** internationalized description lines
5. **Integrates** with 8+ internal services
6. **Optimizes** performance through parallel execution and prioritized booking groups
7. **Handles** errors gracefully with appropriate fallbacks
8. **Validates** data at multiple levels (add-ons, payment options, availability)

The architecture follows clean code principles with clear separation of concerns:
- **Adapters** for external service integration
- **Mappers** for data transformation
- **Domain models** for business logic
- **Utils** for reusable functionality

This service is critical for the ecommerce platform's ability to display and sell bookings as catalog items.

