# WishlistService API Documentation

> __⚠ Internal Documentation ⚠__
>
> This document provides comprehensive technical documentation for the WishlistService implementation.
> This is an internal development document for the Wix Server Onboarding project.

## Overview

The **WishlistService** is a platformized gRPC service built using **Loom Prime** framework that manages wishlists for Wix sites. It provides full CRUD operations, bulk operations, domain events, and integration with other Wix services.

### Key Features

- **Full CRUD Operations**: Create, read, update, and delete wishlists
- **Bulk Operations**: Bulk create, update, and delete with per-item metadata
- **Domain Events**: Automatic emission of CREATED, UPDATED, DELETED, and ACTION events
- **Authorization**: Member-based authorization using CallScope identity
- **Petri Experiments**: Feature flag support for VIP wishlist filtering
- **Visibility Events**: Metrics and logging for VIP wishlist access
- **GDPR Support**: PII handling and data extensions
- **Tag Management**: Support for public and private tags
- **Cross-Service Integration**: Integration with OnboardingMembers service

## Architecture

### Technology Stack

- **Framework**: Loom Prime
- **Data Layer**: SDL (Simple Data Layer)
- **API Protocol**: gRPC with Protocol Buffers
- **Build System**: Bazel
- **Language**: Scala 2.13.17
- **Mapping**: AutoMapper v2

### Service Configuration

- **App Definition ID**: `e820d2c9-9f80-4d2a-9015-5a86e1db7e60`
- **Service Maturity**: `ALPHA`
- **Service Exposure**: `PRIVATE`
- **Entity FQDN**: `wix.tamir.wishlist.v1.wishlist`

## Service Methods

### CRUD Operations

#### CreateWishlist

Creates a new wishlist for the authenticated site member.

**Request**: `CreateWishlistRequest`
- `wishlist`: Wishlist object to create

**Response**: `CreateWishlistResponse`
- `wishlist`: Created wishlist with generated ID and timestamps

**Implementation Details**:
- Automatically extracts member ID from `CallScope`
- Sets `memberId` field from identity context
- Computes `size` field from items array
- Emits `CREATED` domain event

**HTTP Endpoint**: `POST /v1/wishlists`

**Permission**: `NILE_WORKSHOP.CREATE_WISHLIST`

---

#### GetWishlist

Retrieves a wishlist by ID with optional VIP filtering based on Petri experiment.

**Request**: `GetWishlistRequest`
- `wishlist_id`: GUID of the wishlist to retrieve

**Response**: `GetWishlistResponse`
- `wishlist`: Requested wishlist

**Implementation Details**:
- Checks Petri experiment `specsTamirVipWishlistsOnly`
- If experiment is enabled and wishlist is VIP, exposes visibility event
- Returns `NOT_FOUND` if VIP filtering is enabled and wishlist is not VIP

**HTTP Endpoint**: `GET /v1/wishlists/{wishlist_id}`

**Permission**: `NILE_WORKSHOP.GET_WISHLIST`

---

#### UpdateWishlist

Updates an existing wishlist using field mask for partial updates.

**Request**: `UpdateWishlistRequest`
- `wishlist`: Wishlist object with fields to update
- `field_mask`: Set of field paths to update

**Response**: `UpdateWishlistResponse`
- `wishlist`: Updated wishlist

**Implementation Details**:
- Uses field mask to apply partial updates
- Requires `revision` for optimistic locking
- Increments revision on successful update
- Emits `UPDATED` domain event with modified fields only

**HTTP Endpoint**: `PATCH /v1/wishlists/{wishlist.id}`

**Permission**: `NILE_WORKSHOP.UPDATE_WISHLIST`

---

#### DeleteWishlist

Deletes a wishlist. Only the owner can delete their wishlist.

**Request**: `DeleteWishlistRequest`
- `wishlist_id`: GUID of the wishlist to delete

**Response**: `DeleteWishlistResponse`

**Implementation Details**:
- Uses authorization constraint to ensure only owner can delete
- Emits `DELETED` domain event with full entity included

**HTTP Endpoint**: `DELETE /v1/wishlists/{wishlist_id}`

**Permission**: `NILE_WORKSHOP.DELETE_WISHLIST`

---

### Query Operations

#### QueryWishlists

Queries wishlists using WQL (Wix Query Language) with support for filtering, sorting, and paging.

**Request**: `QueryWishlistsRequest`
- `query`: CursorQuery with filter, sort, and paging options

**Response**: `QueryWishlistsResponse`
- `wishlists`: List of matching wishlists
- `paging_metadata`: Cursor paging metadata for pagination

**Implementation Details**:
- Supports up to 1,000 wishlists per request
- Uses cursor-based paging
- Supports filtering on `id` and `email` fields
- Supports sorting on both fields

**HTTP Endpoints**:
- `POST /v1/wishlists/query`
- `GET /v1/wishlists/query`

**Permission**: `NILE_WORKSHOP.GET_WISHLIST`

---

#### ListMyWishlists

Lists all wishlists owned by the authenticated site member.

**Request**: `ListMyWishlistsRequest`

**Response**: `ListMyWishlistsResponse`
- `wishlists`: List of wishlists owned by the caller
- `paging_metadata`: Cursor paging metadata

**Implementation Details**:
- Automatically filters by `memberId` from CallScope
- Uses authorization constraint for security

**HTTP Endpoint**: `GET /v1/wishlists/my`

**Permission**: `NILE_WORKSHOP.GET_WISHLIST`

---

### Item Management

#### AddItemToWishlist

Adds an item to an existing wishlist.

**Request**: `AddItemToWishlistRequest`
- `wishlist_id`: GUID of the wishlist
- `revision`: Current revision number
- `item`: WishlistItem to add

**Response**: `AddItemToWishlistResponse`
- `wishlist`: Updated wishlist with new item

**Implementation Details**:
- Uses SDL array append operation
- Automatically increments `size` field
- Requires revision for optimistic locking
- Emits `UPDATED` domain event

**HTTP Endpoint**: `POST /v1/wishlists/{wishlist_id}/items`

**Permission**: `NILE_WORKSHOP.UPDATE_WISHLIST`

---

### Bulk Operations

#### BulkCreateWishlists

Creates multiple wishlists in a single request.

**Request**: `BulkCreateWishlistsRequest`
- `wishlists`: List of 1-100 wishlists to create
- `return_entity`: Whether to return created entities in response

**Response**: `BulkCreateWishlistsResponse`
- `results`: Per-item results with metadata
- `bulk_action_metadata`: Overall operation metadata

**Implementation Details**:
- Processes up to 100 wishlists per request
- Returns per-item success/failure metadata
- Preserves order of input items
- Emits `CREATED` domain events for each wishlist

**HTTP Endpoint**: `POST /v1/bulk/wishlists/create`

**Permission**: `NILE_WORKSHOP.CREATE_WISHLIST`

---

#### BulkUpdateWishlists

Updates multiple wishlists in a single request.

**Request**: `BulkUpdateWishlistsRequest`
- `wishlists`: List of 1-100 masked wishlists to update
- `return_entity`: Whether to return updated entities in response

**Response**: `BulkUpdateWishlistsResponse`
- `results`: Per-item results with metadata
- `bulk_action_metadata`: Overall operation metadata

**Implementation Details**:
- Each wishlist can have its own field mask
- Requires `id` and `revision` for each wishlist
- Returns per-item success/failure metadata
- Emits `UPDATED` domain events for each wishlist

**HTTP Endpoint**: `POST /v1/bulk/wishlists/update`

**Permission**: `NILE_WORKSHOP.UPDATE_WISHLIST`

---

#### BulkDeleteWishlists

Deletes multiple wishlists in a single request.

**Request**: `BulkDeleteWishlistsRequest`
- `wishlist_ids`: List of 1-100 wishlist IDs to delete

**Response**: `BulkDeleteWishlistsResponse`
- `results`: Per-item results with metadata
- `bulk_action_metadata`: Overall operation metadata

**Implementation Details**:
- Only owner can delete their wishlists
- Uses authorization constraint for security
- Returns per-item success/failure metadata
- Emits `DELETED` domain events for each wishlist

**HTTP Endpoint**: `POST /v1/bulk/wishlists/delete`

**Permission**: `NILE_WORKSHOP.DELETE_WISHLIST`

---

### Tag Operations

#### BulkUpdateWishlistTags

Synchronously updates tags on multiple wishlists by list of IDs.

**Request**: `BulkUpdateWishlistTagsRequest`
- `wishlist_ids`: List of 1-100 wishlist IDs
- `assign_tags`: Tags to assign
- `unassign_tags`: Tags to unassign

**Response**: `BulkUpdateWishlistTagsResponse`
- `results`: Per-item results with metadata
- `bulk_action_metadata`: Overall operation metadata

**Implementation Details**:
- If a tag appears in both assign and unassign lists, it will be assigned
- Returns error if both lists are empty

**HTTP Endpoint**: `POST /v1/bulk/wishlists/update-tags`

**Permission**: `NILE_WORKSHOP.UPDATE_WISHLIST`

---

#### BulkUpdateWishlistTagsByFilter

Asynchronously updates tags on multiple wishlists by filter.

**Request**: `BulkUpdateWishlistTagsByFilterRequest`
- `filter`: WQL filter expression (empty filter updates all)
- `assign_tags`: Tags to assign
- `unassign_tags`: Tags to unassign

**Response**: `BulkUpdateWishlistTagsByFilterResponse`
- `job_id`: GUID of the async job

**Implementation Details**:
- Runs asynchronously as a background job
- Empty filter updates all wishlists
- Returns job ID for tracking

**HTTP Endpoint**: `POST /v1/bulk/wishlists/update-tags-by-filter`

**Permission**: `NILE_WORKSHOP.UPDATE_WISHLIST`

---

### Extended Operations

#### GetWishlistOwnerDetailed

Retrieves detailed information about a wishlist owner by integrating with OnboardingMembers service.

**Request**: `GetWishlistOwnerDetailedRequest`
- `wishlist_id`: GUID of the wishlist

**Response**: `GetWishlistOwnerDetailedResponse`
- `owner`: Detailed owner information including ID, email, and full name

**Implementation Details**:
- Fetches wishlist to get `memberId`
- Calls OnboardingMembers service to get member details
- Combines data from both services

**HTTP Endpoint**: `GET /v1/wishlists/{wishlist_id}/owner-detailed`

**Permission**: `NILE_WORKSHOP.GET_WISHLIST`

---

### Domain Event Consumption

#### ConsumeMemberDomainEvents

Consumes domain events from OnboardingMember entity to handle member deletion.

**Request**: `DomainEvent` from OnboardingMember service

**Response**: `Empty`

**Implementation Details**:
- Listens for `DELETED` events from OnboardingMember
- Automatically deletes all wishlists owned by the deleted member
- Uses filter-based delete operation

**Exposure**: `PRIVATE` (internal use only)

---

## Domain Models

### WishlistDomain

The core domain model for wishlists stored in SDL.

```scala
case class WishlistDomain(
  id: UUID,                    // Auto-generated UUID
  revision: Long,              // Optimistic locking version
  createdDate: Instant,        // Creation timestamp
  updatedDate: Instant,        // Last update timestamp
  email: String,               // Owner email (PII, filterable)
  size: Int,                   // Number of items
  memberId: String,            // Owner site member ID
  items: Seq[WishlistItemDomain], // Wishlist items
  extendedFields: Option[ExtendedFields], // Data extensions
  tags: Option[TagsDomain]     // Public and private tags
)
```

**SDL Annotations**:
- `@Tenancy(TenancyType.INSTANCE)`: Instance-level tenancy
- `@TrashBin`: Soft delete support
- `@Id(IdGeneration.Auto)`: Auto UUID generation
- `@ContactEmail`: Email field annotation
- `@Pii(kind = Kind.Email)`: PII marking for GDPR

---

### WishlistItemDomain

Domain model for wishlist items.

```scala
case class WishlistItemDomain(
  id: String,        // Item ID (5-100 characters)
  itemType: String    // Item type (1-100 characters)
)
```

---

### TagsDomain

Domain model for wishlist tags.

```scala
case class TagsDomain(
  privateTags: Option[TagListDomain],  // Private tags
  publicTags: Option[TagListDomain]    // Public tags
)

case class TagListDomain(
  tagIds: Seq[String]  // List of tag IDs
)
```

---

## Mappers

### AutoMapper Transformers

The service uses AutoMapper v2 for bidirectional transformations between domain and contract models.

#### Domain to Contract

```scala
Transformer[WishlistDomain, Wishlist, Unit]
```

**Computed Fields**:
- `vip`: Computed as `items.size >= 5`
- `email`: Converted from String to Optional StringValue

#### Contract to Domain

```scala
Transformer[Wishlist, WishlistDomain, WishlistTransformerContext]
```

**Computed Fields**:
- `email`: Converted from Optional StringValue to String (empty string if None)
- `size`: Computed from `items.size`
- `memberId`: Set from `WishlistTransformerContext`

#### Update Translator

```scala
Translator[Wishlist, WishlistDomain, WishlistTransformerContext]
```

**Ignored Paths**:
- `vip`: Read-only computed field, not updated

---

## Identity and Authorization

### IdentityExtractor

Extracts site member identity from `CallScope` for authorization.

```scala
def memberId()(implicit callScope: CallScope): String
```

**Implementation**:
- Uses `IdentityContext.extract()` to get identities
- Extracts `SiteMember` identity
- Throws `MissingSiteMemberIdentityError` if identity is missing

**Error**: `MISSING_SITE_MEMBER` (Unauthenticated)

---

### Authorization Patterns

1. **Owner-based Authorization**:
   - `ListMyWishlists`: Filters by `memberId` from CallScope
   - `DeleteWishlist`: Uses authorization constraint
   - `BulkDeleteWishlists`: Uses authorization constraint

2. **Identity Stamping**:
   - `CreateWishlist`: Sets `memberId` from CallScope
   - `BulkCreateWishlists`: Sets `memberId` for each wishlist

---

## Domain Events

### Event Types

The service emits the following domain events:

1. **CREATED**: Emitted when a wishlist is created
   - Exposure: `PUBLIC`
   - Maturity: `ALPHA`
   - Includes full entity

2. **UPDATED**: Emitted when a wishlist is updated
   - Exposure: `PUBLIC`
   - Maturity: `ALPHA`
   - Includes only modified fields

3. **DELETED**: Emitted when a wishlist is deleted
   - Exposure: `PUBLIC`
   - Maturity: `ALPHA`
   - Includes full entity

4. **ACTION**: Emitted when tags are modified
   - Exposure: `PUBLIC`
   - Maturity: `ALPHA`
   - Action message: `TagsModified`
   - Custom slug: `tags_modified`

### Event Configuration

```protobuf
domain_events: {
  deleted_include_entity: true
  updated_include_modified_fields: true
}
domain_events_read_permission: "FAKE_DOMAIN_CHANGE_ME.WISHLIST_DOMAIN_EVENTS_READ"
```

---

## Petri Experiments

### VipWishlistsOnly

Petri experiment that controls VIP wishlist filtering.

**Spec Definition**:
- Control: `"false"` (disabled)
- Variant: `"true"` (enabled)
- Scope: All user types in `tamir-wishlist`
- Owner: `server-onboarding`

**Usage**:
- Checked in `GetWishlist` method
- When enabled, only VIP wishlists (5+ items) are returned
- Non-VIP wishlists return `NOT_FOUND`
- Visibility events are exposed for VIP wishlist access

---

## Visibility Events

### GetVipWishlistEvent

Visibility event exposed when a VIP wishlist is accessed.

```scala
case class GetVipWishlistEvent(wishlistId: String)
  extends VisibilityEventDefaultDetails[Log with Metrics]
```

**Target**:
- `Log.Info`: Logs at info level
- `Metrics.Metric("get-vip-wishlist")`: Emits metric

**Trigger**: When `GetWishlist` is called on a VIP wishlist with experiment enabled

---

## SDL Configuration

### Service Object Configuration

```scala
object WishlistService {
  val AppDefId = "e820d2c9-9f80-4d2a-9015-5a86e1db7e60"
  
  def apply(contextBuilder: WishlistServiceContextBuilder): WishlistService
}
```

### SDL Builder Configuration

- **Tenant Extractor**: Instance-level tenancy
- **Domain Events**: Enabled with contract mapping
- **Field Path Translator**: AutoMapper-based translation
- **GDPR**: Harvester config with UOU and user support
- **Permanent Site Deleted Handler**: Enabled
- **Patch by Filter**: Enabled
- **Delete by Filter**: Enabled

---

## Build Configuration

### Bazel BUILD.bazel

**Key Dependencies**:
- `@server_infra//framework/loom-prime`: Loom Prime framework
- `@server_infra//iptf/simple-data-layer`: SDL support
- `@server_infra//aglianico/automapper`: AutoMapper v2
- `@server_infra//framework/grpc/call-scope`: CallScope support

**SDL Configuration**:
- Uses shared POC cluster
- Trash bin with cleanup enabled
- Partition-based cleanup

**RPC Clients**:
- `wix.onboarding.members.v1.OnboardingMembers`

**Petri Specs**:
- `specs.tamir.VipWishlistsOnly`

---

## Error Handling

### Standard gRPC Errors

- `NOT_FOUND`: Wishlist not found or filtered out
- `PERMISSION_DENIED`: Unauthorized access attempt
- `INVALID_ARGUMENT`: Validation errors

### Custom Application Errors

- `MISSING_SITE_MEMBER`: Site member identity missing from CallScope
- `EMPTY_ASSIGN_AND_UNASSIGN_LISTS`: Both tag lists empty in bulk update

---

## Testing

### Test Structure

Tests are located in `test/com/wixpress/tamir/wishlist/v1/WishlistServiceTest.scala`.

**Test Context**:
- Uses `WishlistServiceTestContextBuilder`
- Provides test SDL instance
- Mocks RPC clients
- Sets up test CallScope with site member identity

**Test Patterns**:
- CRUD operation tests
- Authorization tests
- Bulk operation tests
- Error scenario tests
- Petri experiment tests

---

## Best Practices

### Code Patterns

1. **Always extract identity from CallScope**: Never trust client-provided identity
2. **Use field masks for updates**: Support partial updates
3. **Increment revision**: For optimistic locking
4. **Use authorization constraints**: For security
5. **Return per-item metadata**: In bulk operations
6. **Log visibility events**: For monitoring and metrics

### SDL Patterns

1. **Use AutoMapper**: For domain-contract transformations
2. **Enable domain events**: For event-driven architecture
3. **Configure GDPR**: For PII handling
4. **Use trash bin**: For soft deletes
5. **Enable filter operations**: For flexible queries

---

## References

- [Loom Prime Documentation](https://github.com/wix-private/server-infra/tree/master/framework/loom-prime)
- [SDL Documentation](https://github.com/wix-private/server-infra/tree/master/iptf/simple-data-layer)
- [AutoMapper Documentation](https://github.com/wix-private/server-infra/tree/master/aglianico/automapper)
- [Wix API Guidelines](https://dev.wix.com/docs/rnd-general/articles/p13n-guidelines-aips/introduction)
- [CRUD Methods AIP](https://dev.wix.com/docs/rnd-general/articles/p13n-guidelines-aips/guidance-aips/standard-methods/4001-crud)

---

## Version History

- **v1.0.0** (ALPHA): Initial implementation
  - Basic CRUD operations
  - Bulk operations
  - Domain events
  - Petri experiments
  - Visibility events
  - Tag management
  - Cross-service integration

