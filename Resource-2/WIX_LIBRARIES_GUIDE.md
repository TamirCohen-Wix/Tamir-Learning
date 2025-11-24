# Complete Guide to Wix Libraries, Classes, and APIs Used in Wishlist Service

This comprehensive guide explains all Wix libraries, classes, and APIs used in this Scala server project, including Scala syntax examples from the actual codebase.

## Table of Contents

1. [Loom Prime Framework](#loom-prime-framework)
2. [Simple Data Layer (SDL)](#simple-data-layer-sdl)
3. [AutoMapper v2](#automapper-v2)
4. [CallScope and Identity Management](#callscope-and-identity-management)
5. [gRPC and Protocol Buffers](#grpc-and-protocol-buffers)
6. [Error Handling](#error-handling)
7. [Visibility Framework](#visibility-framework)
8. [Petri Experiments](#petri-experiments)
9. [Domain Events](#domain-events)
10. [RPC Clients](#rpc-clients)
11. [Testing Framework](#testing-framework)
12. [Bazel Build System](#bazel-build-system)

---

## Loom Prime Framework

**Loom Prime** is Wix's framework for building Scala server applications. It provides the infrastructure for gRPC services, dependency injection, and service lifecycle management.

### Key Components

#### Service Trait Extension

Your service extends `WishlistServiceLoomPrimed`, which is generated from your proto definition:

```scala
class WishlistService(
  sdl: SimpleDataLayer[WishlistDomain],
  onboardingMembers: OnboardingMembersPlatformizedClientMethods,
  petri: WishlistServicePetriExperiments,
  visibility: Visibility[Log with Metrics]
)(implicit ec: ExecutionContext) extends WishlistServiceLoomPrimed
```

**Scala Syntax Notes:**
- `extends WishlistServiceLoomPrimed` - Extends the generated service trait
- `(implicit ec: ExecutionContext)` - Implicit execution context for async operations
- Multiple constructor parameters define dependencies

#### Service Factory Pattern

The companion object provides a factory method that configures the service:

```scala
object WishlistService {
  val AppDefId = "e820d2c9-9f80-4d2a-9015-5a86e1db7e60"
  
  def apply(contextBuilder: WishlistServiceContextBuilder)
           (implicit ec: ExecutionContext): WishlistService = {
    val context = contextBuilder.build(sdlBuilder = ...)
    new WishlistService(...)
  }
}
```

**Scala Syntax Notes:**
- `object WishlistService` - Companion object (singleton)
- `def apply(...)` - Factory method (allows `WishlistService(contextBuilder)` syntax)
- `implicit ec: ExecutionContext` - Required for Future operations

---

## Simple Data Layer (SDL)

**SDL** is Wix's data persistence layer that provides type-safe database operations with automatic schema management, multi-tenancy, and GDPR support.

### Domain Model Annotations

Domain classes must be annotated with SDL-specific annotations:

```scala
@Tenancy(tenancyType = TenancyType.INSTANCE, appDefId = "e820d2c9-9f80-4d2a-9015-5a86e1db7e60")
@TrashBin
case class WishlistDomain(
  @Id(idGeneration = IdGeneration.Auto)
  id: UUID,
  revision: Long,
  createdDate: Instant,
  updatedDate: Instant,
  @ContactEmail
  @Pii(kind = Kind.Email, filterable = FilterableOptions.FilterableWithFastHash)
  email: String,
  size: Int,
  memberId: String,
  items: Seq[WishlistItemDomain],
  extendedFields: Option[ExtendedFields],
  tags: Option[TagsDomain],
  @Pii(kind = Kind.Name, filterable = FilterableOptions.FilterableWithFastHash)
  fullName: Option[String],
)
```

**Scala Syntax Notes:**
- `@Tenancy(...)` - Annotation for multi-tenancy configuration
- `@TrashBin` - Enables soft delete functionality
- `@Id(idGeneration = IdGeneration.Auto)` - Auto-generates UUID
- `@Pii(...)` - Marks Personally Identifiable Information fields
- `case class` - Immutable data class with automatic equals/hashCode
- `Option[T]` - Scala's nullable type (None or Some(value))
- `Seq[T]` - Immutable sequence (like List)

### SDL Operations

#### Insert (Create)

```scala
val wishlistDomainCreated <- sdl.insert(
  request.getWishlist.mapTo[WishlistDomain, WishlistTransformerContext](
    WishlistTransformerContext(memberId)
  )
)
```

**Scala Syntax Notes:**
- `<-` - For-comprehension syntax for Future operations
- `sdl.insert(...)` - Returns `Future[WishlistDomain]`
- `.mapTo[...]` - AutoMapper transformation

#### Get (Read)

```scala
val wishlistDomain <- sdl.get(request.wishlistId)
```

**Scala Syntax Notes:**
- `sdl.get(id)` - Returns `Future[Option[WishlistDomain]]`
- Returns `None` if entity doesn't exist

#### Patch (Update with Field Mask)

```scala
val updatedWishlistDomain <- sdl.patch(domain.id.toString, revision)
  .withFieldMask(fieldMaskPaths, domain)
  .execute()
```

**Scala Syntax Notes:**
- `sdl.patch(id, revision)` - Builder pattern for partial updates
- `.withFieldMask(...)` - Specifies which fields to update
- `.execute()` - Executes the operation, returns `Future[WishlistDomain]`
- Requires `revision` for optimistic locking

#### Patch with Array Operations

```scala
val updatedWishlistDomain <- sdl.patch(request.wishlistId, revision)
  .appendItemToArray("items", itemDomain)
  .changeBy("size", 1)
  .execute()
```

**Scala Syntax Notes:**
- `.appendItemToArray(fieldName, value)` - Adds item to array field
- `.changeBy(fieldName, delta)` - Increments/decrements numeric field
- Method chaining for multiple operations

#### Delete

```scala
_ <- sdl.delete.single(request.wishlistId)
  .withAuthorizationConstraint(equal(WishlistService.MemberIdField, memberId))
  .execute()
```

**Scala Syntax Notes:**
- `sdl.delete.single(id)` - Deletes single entity
- `.withAuthorizationConstraint(...)` - Adds authorization check
- `_ <-` - Ignores return value (returns `Future[Unit]`)

#### Delete by Filter

```scala
_ <- sdl.delete.byFilter(equal(WishlistService.MemberIdField, event.entityId))
  .execute()
```

**Scala Syntax Notes:**
- `sdl.delete.byFilter(...)` - Deletes all entities matching filter
- Used for cascading deletes

#### Query

```scala
val response <- sdl.query
  .withAuthorizationConstraint(equal(WishlistService.MemberIdField, memberId))
  .execute()
```

**Scala Syntax Notes:**
- `sdl.query` - Builder for query operations
- `.withAuthorizationConstraint(...)` - Filters by authorization
- Returns `Future[QueryResponse[WishlistDomain]]`

#### Query with Cursor

```scala
val response <- sdl.query
  .withCursor(nextCursor)
  .execute()
```

**Scala Syntax Notes:**
- `.withCursor(cursor)` - Continues pagination from cursor
- Used for iterating through large result sets

#### Query from Platformized Query

```scala
val response <- sdl.query
  .fromPlatformizedQuery(request.getQuery.mapTo[CursorQuery])
  .execute()
```

**Scala Syntax Notes:**
- `.fromPlatformizedQuery(...)` - Converts proto query to SDL query
- Supports WQL (Wix Query Language) with filtering, sorting, paging

### Bulk Operations

#### Bulk Insert

```scala
val insertBulkResults <- sdl.insert.bulk(
  request.wishlists.map(_.mapTo[WishlistDomain, WishlistTransformerContext](
    WishlistTransformerContext(memberId)
  ))
)
  .withReturnEntity(request.returnEntity)
  .execute()
```

**Scala Syntax Notes:**
- `sdl.insert.bulk(entities)` - Bulk create operation
- `.withReturnEntity(true)` - Returns created entities in response
- Returns `Future[BulkInsertResult[WishlistDomain]]` with per-item metadata

#### Bulk Patch

```scala
val updateBulkResultsWithEntities <- sdl.patch.bulk
  .withFieldMaskEntries(request.wishlists.flatMap(maskedWishlist =>
    maskedWishlist.wishlist.map(wishlist => {
      val wishlistDomain = wishlist.mapTo[WishlistDomain, WishlistTransformerContext](...)
      FieldMaskPatchEntry[WishlistDomain](
        entityId = wishlistDomain.id.toString,
        fieldMask = maskedWishlist.fieldMask.map(_.paths).getOrElse(Nil),
        entity = wishlistDomain,
        revision = wishlist.getRevision
      )
    })
  ))
  .withReturnEntity(request.returnEntity)
  .execute()
```

**Scala Syntax Notes:**
- `sdl.patch.bulk` - Bulk update operation
- `.withFieldMaskEntries(...)` - Each entry can have different field mask
- `flatMap` - Flattens nested collections
- `getOrElse(Nil)` - Provides default empty list if None

#### Bulk Delete

```scala
val deleteBulkResults <- sdl.delete.bulk(request.wishlistIds)
  .withAuthorizationConstraint(equal(WishlistService.MemberIdField, memberId))
  .execute()
```

**Scala Syntax Notes:**
- `sdl.delete.bulk(ids)` - Bulk delete operation
- Returns per-item success/failure metadata

### SDL Configuration

SDL is configured in the service factory method:

```scala
val context = contextBuilder.build(sdlBuilder =
  _.withWishlist(_
    .withTenantExtractor(TenantExtractors.instance(contextBuilder.entity.appDefId))
    .withDomainEventsEnabled(_.mapTo[Wishlist])
    .withContractToDomainFieldPathTranslator(
      AutomapperSdlFieldPathTranslator[Wishlist, WishlistDomain, WishlistTransformerContext]
    )
    .withGdpr(SdlHarvesterPrimeConfig(
      contextBuilder.entity.appDefId,
      contextBuilder.secrets.appSecret,
      SdlHarvesterConfig.uouAndUserSupport
    ))
    .withPermanentSiteDeletedHandler
    .withPatchByFilter()
    .withDeleteByFilter()
  )
)
```

**Scala Syntax Notes:**
- `_.withWishlist(_` - Builder pattern with underscore placeholders
- `_.mapTo[Wishlist]` - Maps domain to contract for domain events
- Method chaining for configuration

### Filter DSL

SDL provides a type-safe filter DSL:

```scala
import com.wixpress.infra.sdl.api.FilterDSL.equal

.withAuthorizationConstraint(equal(WishlistService.MemberIdField, memberId))
```

**Scala Syntax Notes:**
- `equal(fieldName, value)` - Creates equality filter
- Type-safe field names prevent typos

---

## AutoMapper v2

**AutoMapper v2** is Wix's type-safe transformation library for converting between domain models and protocol buffer contracts.

### Transformer (One-Way Transformation)

Transformers convert from one type to another:

```scala
implicit val wishlistDomainTransformer: Transformer[WishlistDomain, Wishlist, Unit] =
  Transformer.define[WishlistDomain, Wishlist]
    .withFieldComputed(_.vip, source => source.items.size >= 5)
    .withFieldComputed(_.email, source => if (source.email.isEmpty) None else Some(source.email))
    .withFieldComputed(_.ownerFullName, source => source.fullName)
    .buildTransformer
```

**Scala Syntax Notes:**
- `Transformer[Source, Target, Context]` - Type parameters for transformation
- `Transformer.define[...]` - Builder pattern
- `.withFieldComputed(targetField, source => computation)` - Computes target field from source
- `source => ...` - Lambda function (anonymous function)
- `_.vip` - Placeholder syntax for field accessor
- `.buildTransformer` - Builds the transformer

### Transformer with Context

When transformation needs additional context:

```scala
implicit val wishlistTransformer: Transformer[Wishlist, WishlistDomain, WishlistTransformerContext] =
  Transformer.define[Wishlist, WishlistDomain, WishlistTransformerContext]
    .withFieldComputed(_.email, (_: WishlistTransformerContext) => (source: Wishlist) => 
      source.email.getOrElse("")
    )
    .withFieldComputed(_.size, (_: WishlistTransformerContext) => (source: Wishlist) => 
      source.items.size
    )
    .withFieldConst(_.memberId, (context: WishlistTransformerContext) => context.memberId)
    .buildTransformer
```

**Scala Syntax Notes:**
- `WishlistTransformerContext` - Custom context type
- `.withFieldComputed(_, context => source => computation)` - Context-aware computation
- `.withFieldConst(_, context => value)` - Constant value from context
- Curried functions: `context => source => result`

### Field Renaming

Transformers can rename fields:

```scala
implicit val wishListItemDomainTransformer: Transformer[WishlistItemDomain, WishlistItem, Unit] =
  Transformer.define[WishlistItemDomain, WishlistItem]
    .withFieldRenamed(_.itemType, _.`type`)
    .buildTransformer
```

**Scala Syntax Notes:**
- `.withFieldRenamed(sourceField, targetField)` - Maps field names
- `` _.`type` `` - Backticks allow using reserved keywords as identifiers

### Translator (Partial Update)

Translators are used for updates, ignoring computed/read-only fields:

```scala
implicit val wishlistTranslator: Translator[Wishlist, WishlistDomain, WishlistTransformerContext] =
  Translator.derive[Wishlist, WishlistDomain, WishlistTransformerContext]
    .withIgnorePaths("vip", "ownerFullName")
    .build
```

**Scala Syntax Notes:**
- `Translator.derive[...]` - Automatically derives translator from types
- `.withIgnorePaths(...)` - Fields to skip during updates
- Used with field masks for partial updates

### Using Transformers

Transformers are used via the `.mapTo` extension method:

```scala
val wishlistDomain = request.getWishlist.mapTo[WishlistDomain, WishlistTransformerContext](
  WishlistTransformerContext(memberId)
)
```

**Scala Syntax Notes:**
- `.mapTo[Target, Context](context)` - Implicit transformer converts the object
- Requires implicit transformer in scope
- Type inference determines which transformer to use

### Using Translators with Field Masks

```scala
val domain = request.getWishlist.mapTo[WishlistDomain, WishlistTransformerContext](
  WishlistTransformerContext(memberId),
  Some(Configuration(Some(Partial(fieldMask = fieldMaskPaths))))
)
```

**Scala Syntax Notes:**
- `Configuration(Some(Partial(fieldMask = paths)))` - Configures field mask
- Only specified fields are transformed
- Used for partial updates

---

## CallScope and Identity Management

**CallScope** carries request context including identity, tenant information, and metadata for each gRPC call.

### CallScope Parameter

All service methods accept implicit CallScope:

```scala
override def createWishlist(request: CreateWishlistRequest)
                          (implicit callScope: CallScope): Future[CreateWishlistResponse] = {
  // ...
}
```

**Scala Syntax Notes:**
- `implicit callScope: CallScope` - Implicit parameter (automatically passed)
- Must be last parameter group
- Framework provides CallScope automatically

### Extracting Site Member ID

```scala
import com.wixpress.wixerd.enricher.IdentityContext

def memberId()(implicit callScope: CallScope): String = {
  IdentityContext.extract(callScope).identities.siteMember match {
    case Some(identity: IdentityContext.SiteMember) => identity.id
    case _ => throw new MissingSiteMemberIdentityError()
  }
}
```

**Scala Syntax Notes:**
- `IdentityContext.extract(callScope)` - Extracts identity from CallScope
- `.identities.siteMember` - Accesses site member identity
- Pattern matching: `case Some(...) => ... case _ => ...`
- Type ascription: `identity: IdentityContext.SiteMember`

### Alternative: CallScopeUtils

```scala
import com.wixpress.callscope.CallScopeUtils

val memberId = CallScopeUtils.siteMemberId(cs)
```

**Scala Syntax Notes:**
- `CallScopeUtils.siteMemberId(cs)` - Convenience method
- Returns `Option[String]` (may be None)

### Creating CallScope for Tests

```scala
import com.wixpress.grpc.testkit.TestCallScope
import com.wixpress.metasite.reloose.api.{App, MetaSiteContext}
import com.wix.core.testkit.identification.IdentityFactory

def csForMemberId(appDefId: String, instanceId: String, siteMemberId: String): CallScope = {
  val msContext = MetaSiteContext(
    metaSiteId = randomGuidAsString,
    apps = Seq(App(appDefId, instanceId))
  )
  val identity = IdentityFactory.responseForInstanceWithSiteMemberWithTargetAccount(
    appDefId, instanceId, siteMemberId, targetAccountId = randomGuidAsString
  )
  CallScopeUtils.callScopeWith(identity, Some(msContext))
}
```

**Scala Syntax Notes:**
- `Seq(...)` - Creates sequence (immutable list)
- Named parameters: `metaSiteId = ...`
- `Some(msContext)` - Wraps value in Option

---

## gRPC and Protocol Buffers

**gRPC** is used for service communication, and **Protocol Buffers** define the API contracts.

### gRPC Status Errors

```scala
import io.grpc.{Status, StatusRuntimeException}

throw new StatusRuntimeException(Status.NOT_FOUND)
// or
throw Status.NOT_FOUND
  .withDescription("Entity not found")
  .asRuntimeException()
```

**Scala Syntax Notes:**
- `Status.NOT_FOUND` - gRPC status code enum
- `.withDescription(...)` - Adds error message
- `.asRuntimeException()` - Converts to throwable

### Protocol Buffer Field Access

```scala
val wishlist = request.getWishlist  // Get optional field
val email = wishlist.email          // Get Option[String]
val id = wishlist.getId             // Get required field
```

**Scala Syntax Notes:**
- `.getWishlist` - Returns `Option[Wishlist]` for optional fields
- `.email` - Direct access returns `Option[String]`
- `.getId` - Returns `String` for required fields
- Generated by ScalaPB from proto definitions

### Field Masks

```scala
val fieldMaskPaths = request.getFieldMask.paths
```

**Scala Syntax Notes:**
- `request.getFieldMask` - Returns `Option[FieldMask]`
- `.paths` - Returns `Seq[String]` of field paths
- Used for partial updates

---

## Error Handling

### WixApplicationRuntimeException

Custom application errors extend this class:

```scala
import com.wixpress.framework.errors.{WixApplicationRuntimeException, WixResponseStatus}
import com.wixpress.framework.errors.Details.ApplicationDetails

class MissingSiteMemberIdentityError() extends WixApplicationRuntimeException(
  responseStatus = WixResponseStatus.Unauthenticated,
  responseMessage = "Site member identity is missing",
  applicationDetails = ApplicationDetails("MISSING_SITE_MEMBER")
)
```

**Scala Syntax Notes:**
- `extends WixApplicationRuntimeException(...)` - Extends base exception class
- Named parameters in constructor
- `ApplicationDetails(code)` - Custom error code

### WixResponseStatus Values

```scala
WixResponseStatus.NotFound
WixResponseStatus.Unauthenticated
WixResponseStatus.PermissionDenied
WixResponseStatus.InvalidArgument
WixResponseStatus.Internal
```

---

## Visibility Framework

**Visibility** framework provides structured logging and metrics collection.

### Visibility Service

```scala
import com.wixpress.framework.visibility.{GenericEvent, Log, Visibility}

class WishlistService(
  visibility: Visibility[Log with Metrics],
  // ...
)
```

**Scala Syntax Notes:**
- `Visibility[Log with Metrics]` - Type parameter specifies capabilities
- `Log with Metrics` - Intersection type (has both traits)

### Exposing Generic Events

```scala
visibility.expose(GenericEvent(Log.Info, s"[getWishlist] wishlistId=${request.wishlistId}"))
```

**Scala Syntax Notes:**
- `GenericEvent(level, message)` - Creates log event
- `Log.Info` - Log level enum
- String interpolation: `s"... ${variable} ..."`

### Custom Visibility Events

```scala
import com.wixpress.framework.visibility.{Log, Metrics, Target, VisibilityEventDefaultDetails}

case class GetVipWishlistEvent(wishlistId: String) 
  extends VisibilityEventDefaultDetails[Log with Metrics] {
  override def visibilityTarget: Target[Log with Metrics] =
    Log.Info(s"get wishlist vip $wishlistId") + Metrics.Metric("get-vip-wishlist")
}
```

**Scala Syntax Notes:**
- `extends VisibilityEventDefaultDetails[...]` - Base class for custom events
- `override def` - Overrides base class method
- `Log.Info(...) + Metrics.Metric(...)` - Combines log and metric
- `case class` - Immutable data class

### Using Custom Events

```scala
visibility.expose(GetVipWishlistEvent(wishlist.getId))
```

**Scala Syntax Notes:**
- `GetVipWishlistEvent(...)` - Creates event instance
- Automatically logs and emits metric

---

## Petri Experiments

**Petri** is Wix's feature flag and A/B testing framework.

### Petri Experiments Interface

```scala
class WishlistService(
  petri: WishlistServicePetriExperiments,
  // ...
)
```

**Scala Syntax Notes:**
- `WishlistServicePetriExperiments` - Generated interface from specs
- Methods correspond to experiment specs

### Conducting Experiments

```scala
val isVipOnly = petri.conductSpecsTamirVipWishlistsOnly.contains("true")
```

**Scala Syntax Notes:**
- `petri.conductSpecsTamirVipWishlistsOnly` - Returns `Option[String]`
- `.contains("true")` - Checks if experiment is enabled
- Method name matches spec name in `petri_specs` in BUILD.bazel

### Experiment Spec Definition

```scala
import com.wixpress.petri.experiments.domain.ScopeDefinition
import com.wixpress.petri.petri.SpecDefinition

object VipWishlistsOnly extends SpecDefinition {
  override def specName: String = "specs.tamir.VipWishlistsOnly"
  
  override def buildSpec: ExperimentSpecBuilder = {
    ExperimentSpecBuilder()
      .withScope(ScopeDefinition.siteWide)
      .withTestGroups("true", "false")
  }
}
```

**Scala Syntax Notes:**
- `extends SpecDefinition` - Base trait for experiment specs
- `override def` - Overrides trait methods
- Builder pattern for configuration

---

## Domain Events

**Domain Events** are automatically emitted by SDL when entities are created, updated, or deleted.

### Proto Configuration

Domain events are configured in the entity proto:

```protobuf
message Wishlist {
  option (.wix.api.entity) = {
    fqdn: "wix.tamir.wishlist.v1.wishlist"
    domain_events: {
      deleted_include_entity: true
      updated_include_modified_fields: true
    }
  };

  option (.wix.api.domain_event) = {
    event_type: CREATED
    exposure: PUBLIC
    maturity: ALPHA
  };
}
```

### SDL Configuration

```scala
.withDomainEventsEnabled(_.mapTo[Wishlist])
```

**Scala Syntax Notes:**
- `.withDomainEventsEnabled(...)` - Enables domain events
- `_.mapTo[Wishlist]` - Maps domain to contract for events
- Events are automatically emitted, no manual code needed

### Consuming Domain Events

```scala
override def consumeMemberDomainEvents(event: OnboardingMemberDomainEvent)
                                     (implicit callScope: CallScope): Future[Unit] = {
  event.body match {
    case _: OnboardingMemberDomainEvent.Body.Deleted => 
      for {
        _ <- sdl.delete.byFilter(equal(WishlistService.MemberIdField, event.entityId))
          .execute()
      } yield ()
    case _ => Future.unit
  }
}
```

**Scala Syntax Notes:**
- Pattern matching on `event.body` - Discriminated union type
- `case _: Type =>` - Type pattern matching
- `Future.unit` - Completed Future with Unit value
- `for { ... } yield ()` - For-comprehension for Future operations

---

## RPC Clients

**RPC Clients** are generated from proto definitions for calling other services.

### Platformized Client

```scala
class WishlistService(
  onboardingMembers: OnboardingMembersPlatformizedClientMethods,
  // ...
)
```

**Scala Syntax Notes:**
- `OnboardingMembersPlatformizedClientMethods` - Generated client interface
- `Platformized` suffix indicates it handles authentication automatically

### Calling RPC Methods

```scala
val ownerDetails <- onboardingMembers.getOnboardingMember(
  GetOnboardingMemberRequest(wishlistDomain.memberId)
)
```

**Scala Syntax Notes:**
- `.getOnboardingMember(...)` - RPC method call
- Returns `Future[GetOnboardingMemberResponse]`
- Authentication handled automatically by platform

---

## Testing Framework

### Test Base Class

```scala
import org.specs2.mutable.SpecificationWithJUnit
import org.specs2.specification.Scope

class WishlistServiceTest(implicit ee: ExecutionEnv) 
  extends SpecificationWithJUnit with Mockito with VisibilityMatchers {
  // ...
}
```

**Scala Syntax Notes:**
- `extends SpecificationWithJUnit` - Base class for Specs2 tests
- `with Mockito` - Mixin for mocking
- `with VisibilityMatchers` - Mixin for visibility assertions
- `implicit ee: ExecutionEnv` - Execution environment for async tests

### Test Context Trait

```scala
trait Ctx extends Scope {
  val instanceId: String = randomGuidAsString
  val contextBuilder = WishlistServiceTestContextBuilder(...)
  implicit val callScope: CallScope = csForMemberId(...)
  val service: BlockingWishlistServiceLoomPrimed = 
    contextBuilder.blockingPlatformized(WishlistService(contextBuilder))
}
```

**Scala Syntax Notes:**
- `trait Ctx extends Scope` - Test context trait
- `val` - Immutable value
- `implicit val` - Implicit value (automatically passed)
- `BlockingWishlistServiceLoomPrimed` - Synchronous wrapper for testing

### Error Matchers

```scala
import com.wixpress.framework.errors.it.ErrorMatchers._
import com.wixpress.framework.errors.WixResponseStatus

service.getEntity(GetEntityRequest(nonExistentId)) must
  beApplicationError(
    status = WixResponseStatus.NotFound,
    code = beEqualTo(io.grpc.Status.Code.NOT_FOUND.toString)
  )
```

**Scala Syntax Notes:**
- `must beApplicationError(...)` - Specs2 matcher syntax
- `beEqualTo(...)` - Equality matcher
- `beSome(...)` - Option matcher

### Bulk Operation Matchers

```scala
import matchers.com.wix.tamir.wishlist.v1.upstream.wix.common.BulkMatchers._

response must beBulkCreateWishlistsResponse()
  .withResults(haveSize(2))
  .withBulkActionMetadata(beSome(beBulkActionMetadata()
    .withTotalSuccesses(===(2))
    .withTotalFailures(===(0))
  ))
```

**Scala Syntax Notes:**
- `beBulkCreateWishlistsResponse()` - Generated matcher
- `.withResults(...)` - Chained matcher for nested fields
- `haveSize(2)` - Collection size matcher
- `===(2)` - Exact equality matcher

---

## Bazel Build System

**Bazel** is the build system used for Wix Scala services.

### prime_app Macro

```bazel
load("@server_infra//framework/loom-prime:macros.bzl", "prime_app")

prime_app(
    name = "wishlist-tamir",
    artifact = "com.wixpress.tamir.wishlist.wishlist-tamir",
    service = "wix.tamir.wishlist.v1.WishlistService",
    sdl = {
        "use_shared_poc_cluster": {
            "wishlist_tamir": {
                "wishlist": {
                    "api_entity": "wix.tamir.wishlist.v1.wishlist",
                    "entity": "com.wixpress.tamir.wishlist.v1.domain.WishlistDomain",
                    "trash_bin": {
                        "cleanup_enabled": True,
                        "cleanup_use_partitions": True,
                    },
                },
            },
        },
    },
    deps = [
        "@server_infra//aglianico/automapper",
        "@server_infra//iptf/simple-data-layer/src/main/scala/com/wixpress/infra/sdl/api",
        # ... more deps
    ],
)
```

**Bazel Syntax Notes:**
- `prime_app(...)` - Macro that generates service build rules
- `name` - Target name
- `service` - Fully qualified service class name
- `sdl` - SDL configuration dictionary
- `deps` - List of Bazel dependency labels

---

## Common Scala Patterns in Wix Services

### For-Comprehensions

```scala
for {
  wishlistDomain <- sdl.get(request.wishlistId)
  wishlist = wishlistDomain.mapTo[Wishlist]
  result = if (condition) {
    GetWishlistResponse().withWishlist(wishlist)
  } else {
    throw new StatusRuntimeException(Status.NOT_FOUND)
  }
} yield result
```

**Scala Syntax Notes:**
- `for { ... } yield ...` - For-comprehension for Future operations
- `<-` - Extracts value from Future
- `=` - Binds value (not a Future)
- `yield` - Returns final result

### Pattern Matching

```scala
event.body match {
  case _: OnboardingMemberDomainEvent.Body.Deleted => // handle delete
  case _ => Future.unit  // ignore other events
}
```

**Scala Syntax Notes:**
- `match { case ... => ... }` - Pattern matching expression
- `case _: Type =>` - Type pattern
- `case _ =>` - Default case

### Option Handling

```scala
val email = source.email.getOrElse("")  // Default value
val id = source.id.getOrElse(throw new Error("Missing ID"))  // Throw if None
source.email.map(email => email.toUpperCase)  // Transform if Some
```

**Scala Syntax Notes:**
- `.getOrElse(default)` - Provides default if None
- `.map(f)` - Transforms value if Some, returns None if None

### Collection Operations

```scala
val results = request.wishlists.map(_.mapTo[WishlistDomain, ...])  // Transform each
val successes = results.count(_.getItemMetadata.getSuccess)  // Count matching
val filtered = results.filter(_.getItemMetadata.getSuccess)  // Filter matching
```

**Scala Syntax Notes:**
- `.map(f)` - Transforms each element
- `.count(predicate)` - Counts matching elements
- `.filter(predicate)` - Filters matching elements
- `_` - Placeholder for element

---

## Summary

This guide covers all major Wix libraries and frameworks used in the Wishlist Service:

1. **Loom Prime** - Service framework and dependency injection
2. **SDL** - Data persistence with annotations and operations
3. **AutoMapper v2** - Type-safe transformations
4. **CallScope** - Request context and identity
5. **gRPC** - Service communication
6. **Error Handling** - Custom exceptions and status codes
7. **Visibility** - Logging and metrics
8. **Petri** - Feature flags
9. **Domain Events** - Event-driven architecture
10. **RPC Clients** - Service-to-service communication
11. **Testing** - Specs2 framework with matchers
12. **Bazel** - Build system configuration

Each section includes Scala syntax explanations to help understand the code patterns used throughout the project.

