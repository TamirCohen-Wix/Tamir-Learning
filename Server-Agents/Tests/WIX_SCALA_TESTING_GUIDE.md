# Comprehensive Testing Guide for Wix Scala Services

A complete guide for writing tests for Scala services in the Wix enterprise ecosystem, using Loom Prime + Nile + Bazel.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Core Testing Stack](#2-core-testing-stack)
3. [File & Package Structure](#3-file--package-structure)
4. [Base Test Architecture](#4-base-test-architecture)
5. [Context Traits Pattern](#5-context-traits-pattern)
6. [Nile-Generated Proto Artifacts](#6-nile-generated-proto-artifacts)
7. [Error and API Gateway Matchers](#7-error-and-api-gateway-matchers)
8. [Fake Services Pattern](#8-fake-services-pattern)
9. [Loom Prime Testkits & E2E Tests](#9-loom-prime-testkits--e2e-tests)
10. [Greyhound (Kafka) Testing](#10-greyhound-kafka-testing)
11. [Bazel Wiring](#11-bazel-wiring)
12. [Best Practices](#12-best-practices)
13. [Complete Examples](#13-complete-examples)
14. [Troubleshooting](#14-troubleshooting)
15. [Quick Reference](#15-quick-reference)

---

## 1. Overview

This guide covers testing standards for Scala backend services in the Wix ecosystem. The testing architecture follows **Wix Server Guild standards** and uses:

- **Specs2** with JUnit integration for the test framework
- **Mockito** (via Specs2) for mocking
- **Bazel** for build configuration (`prime_app` macro)
- **Nile** for proto-generated test artifacts (matchers, randoms, clients)
- **Layered architecture** with Base Test classes and Context traits
- **Fake Services** for in-memory test implementations

### Testing Pyramid

```
            ┌─────────────────┐
            │     E2E Tests   │  ← Minimal (only when necessary)
            │   (IT tests)    │
            └────────┬────────┘
                     │
         ┌───────────┴───────────┐
         │  Integration Tests    │  ← For component interactions
         │  (with real SDL/DB)   │
         └───────────┬───────────┘
                     │
    ┌────────────────┴────────────────┐
    │        Unit Tests (Default)     │  ← Majority of tests
    │  In-memory service tests with   │
    │  mocks, fakes, and testkits     │
    └─────────────────────────────────┘
```

**Key Principle**: Prefer **in-memory unit tests** with mocks and fakes over E2E tests. E2E tests are slower, more fragile, and harder to maintain.

---

## 2. Core Testing Stack

### 2.1 Framework Imports

```scala
import org.specs2.mutable.SpecWithJUnit      // Base test class
import org.specs2.matcher.{FutureMatchers, Matchers}
import org.specs2.mock.Mockito               // Mocking framework
import org.specs2.specification.Scope        // Context trait base
import org.specs2.concurrent.ExecutionEnv    // For async tests
```

### 2.2 Wix Framework Libraries

```scala
// Test infrastructure
import com.wixpress.grpc.testkit.TestCallScope
import com.wixpress.grpc.CallScope

// Error matchers
import com.wixpress.framework.errors.WixResponseStatus
import com.wixpress.framework.errors.it.ErrorMatchers.{beApplicationError, beValidationError}

// API Gateway matchers
import com.wixpress.wixerd.ApiGatewayTestkitMatchers.haveSignatureBy
import com.wixpress.grpc.testkit.CallScopeMatchers._

// Random generators
import com.wixpress.hoopoe.ids.randomGuidAsString
import com.wixpress.hoopoe.test.{randomBoolean, randomInt, randomStr, randomEmail}

// Async utilities
import com.wixpress.framework.loom.prime.AwaitOps._
```

### 2.3 Key Libraries Overview

| Library | Purpose | Bazel Target |
|---------|---------|--------------|
| Specs2 | Test framework | `@org_specs2_specs2_*` |
| Mockito | Mocking | Included in Specs2 |
| TestCallScope | gRPC call scope for tests | `@server_infra//framework/grpc/testkit` |
| Error Matchers | Test error responses | `@server_infra//framework/errors/matchers` |
| API Gateway Testkit | Test signed calls | `@server_infra//iptf/wixerd/api-gateway-*` |
| Hoopoe Utils | Random generators | `@wix_framework//hoopoe-common/hoopoe-utest` |
| Dynamic Config Testkit | Test config values | `@velocity_infra//conductor/dynamic-config/library/testkit` |

---

## 3. File & Package Structure

### 3.1 Directory Layout

```
bookings-backend/your-service-name/
├── BUILD.bazel                              # Bazel build config
├── src/                                     # Production code
│   └── main/scala/com/wixpress/your/package/v1/
│       ├── YourService.scala
│       ├── YourServiceImpl.scala
│       └── ...
├── test/                                    # Unit tests
│   └── com/wixpress/your/package/v1/
│       ├── YourServiceBaseTest.scala        # Base test + base contexts
│       ├── FakeExternalService.scala        # Fake service implementation
│       ├── YourFeatureTest.scala            # Test class
│       └── ...
├── it/                                      # Integration/E2E tests
│   └── com/wixpress/your/package/v1/
│       ├── ITEnv.scala                      # IT environment setup
│       ├── ITEnvSupport.scala               # IT environment trait
│       └── YourServiceE2E.scala             # E2E test
└── test-resources/                          # Test resource files
    └── sdl/
        └── ...
```

### 3.2 Naming Conventions

#### ⚠️ Critical Naming Rules from Loom Prime

| Type | Naming Pattern | Example | Notes |
|------|---------------|---------|-------|
| Test Suites | End with `Test` | `YourFeatureTest.scala` | Will be discovered and run |
| E2E Test Suites | End with `E2E` | `YourServiceE2E.scala` | Will be discovered and run |
| Base Test Classes | End with `BaseTest` | `YourServiceBaseTest.scala` | **NOT** discovered as tests |
| Test Helpers | End with `Helper`, `Utils`, `Support` | `TestHelper.scala` | **NOT** discovered as tests |
| Fake Services | Start with `Fake` | `FakeResourceService.scala` | Clear purpose indication |

**⚠️ CRITICAL**: Helper files must **NOT** end with `Test` or `E2E`:
- ✅ Good: `MyServiceBaseTest.scala`, `TestHelpers.scala`, `ITEnvSupport.scala`
- ❌ Bad: `MyServiceBaseTestHelper.scala` ending with `Test`, `MyServiceEnvE2E.scala` for helpers

---

## 4. Base Test Architecture

### 4.1 Two-Layer Architecture

Every test module should follow a two-layer architecture:

```
┌─────────────────────────────────────────────────────────┐
│  YourServiceBaseTest (abstract class)                   │
│  ├── extends SpecWithJUnit                              │
│  ├── with FutureMatchers, Matchers, Mockito             │
│  └── defines: trait Ctx extends Scope                   │
│       ├── TestCallScope                                 │
│       ├── Service context/builder                       │
│       ├── Mock/fake services                            │
│       └── Helper methods (given*, a*, etc.)             │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│  YourFeatureTest extends YourServiceBaseTest            │
│  ├── "Feature description" should {                     │
│  │     "scenario description" in new FeatureCtx {       │
│  │         // Arrange, Act, Assert                      │
│  │     }                                                │
│  │  }                                                   │
│  └── private trait FeatureCtx extends Ctx { ... }       │
└─────────────────────────────────────────────────────────┘
```

### 4.2 Base Test Class Template

```scala
package com.wixpress.your.package.v1

import org.specs2.mutable.SpecWithJUnit
import org.specs2.matcher.{FutureMatchers, Matchers}
import org.specs2.mock.Mockito
import org.specs2.specification.Scope
import org.specs2.concurrent.ExecutionEnv

import com.wixpress.grpc.testkit.TestCallScope
import com.wixpress.wixerd.ApiGatewayTestkitMatchers.haveSignatureBy
import com.wixpress.framework.errors.WixResponseStatus
import com.wixpress.framework.errors.it.ErrorMatchers.{beApplicationError, beValidationError}
import com.wixpress.hoopoe.ids.randomGuidAsString

import scala.concurrent.Future
import scala.concurrent.duration._

abstract class YourServiceBaseTest
    extends SpecWithJUnit
    with FutureMatchers
    with Matchers
    with Mockito {

  sequential  // Ensures tests run sequentially (prevents race conditions)

  trait Ctx extends Scope {
    // 1. Call scope for gRPC calls
    implicit val callScope: TestCallScope = TestCallScope()

    // 2. Configuration and secrets
    val config = YourServiceConfig(
      someConfigValue = "test-value"
    )
    val secrets = YourServiceSecrets(
      appSecret = randomGuidAsString
    )

    // 3. Fake services for external dependencies
    val fakeExternalService = new FakeExternalService()
    val externalService = fakeExternalService.mock

    // 4. Mock services for internal dependencies
    val internalClient = mock[InternalServicePlatformizedClientMethods]

    // 5. Context builder setup
    val contextBuilder = overrideContextBuilder(
      YourServiceTestContextBuilder(
        secrets = secrets,
        config = config,
        rpc = YourServiceTestContextBuilder.mockRpc.copy(
          externalService = externalService,
          internalClient = internalClient
        )
      )
    )

    // 6. Service under test
    val blockingService: BlockingYourServiceLoomPrimed =
      contextBuilder
        .blockingPlatformized(
          YourService(contextBuilder, Some(CollectingDynamicConfig())),
          90000.seconds
        )
        .yourService

    val asyncService: YourServiceLoomPrimed =
      contextBuilder.platformized(YourService(contextBuilder)).yourService

    // 7. Default mock responses
    internalClient.someMethod(any())(any()) returns Future.successful(SomeResponse())

    // 8. Helper methods
    def givenExternalData(data: ExternalData): ExternalData =
      fakeExternalService.givenData(data)

    def aExternalData(
      id: String = randomGuidAsString,
      name: String = randomStr()
    ): ExternalData =
      ExternalData().withId(id).withName(name)

    def overrideContextBuilder(
      builder: YourServiceTestContextBuilder
    ): YourServiceTestContextBuilder = builder

    def mustFailWith(f: => Future[_], msgPart: String) =
      f must throwA[Exception].like { 
        case e => e.getMessage must contain(msgPart) 
      }.await
  }

  // Specialized contexts for common scenarios
  trait EmptyDataCtx extends Ctx {
    // Context with no data setup
  }

  trait SingleItemCtx extends Ctx {
    val itemId = randomGuidAsString
    val item = aExternalData(id = itemId, name = "Test Item")
    givenExternalData(item)
  }

  trait MultipleItemsCtx extends Ctx {
    val item1 = aExternalData(id = randomGuidAsString)
    val item2 = aExternalData(id = randomGuidAsString)
    val item3 = aExternalData(id = randomGuidAsString)
    givenExternalData(item1)
    givenExternalData(item2)
    givenExternalData(item3)
  }
}
```

---

## 5. Context Traits Pattern

### 5.1 Context Hierarchy

Contexts use **progressive specialization**:

```scala
// Level 1: Base context (in BaseTest)
trait Ctx extends Scope {
  // Common setup, mocks, helpers
}

// Level 2: Feature-specific context
trait YourFeatureCtx extends Ctx {
  val featureSpecificId = randomGuidAsString
  val startDate = LocalDateTime.now().plusDays(1)
  val endDate = startDate.plusDays(1)
  // Feature-specific setup
}

// Level 3: Scenario-specific context
trait SingleItemWithDefaultConfigCtx extends YourFeatureCtx {
  val itemId = randomGuidAsString
  val item = aItem(itemId, ...)
  givenItems(Seq(item))
}

// Level 4: Test-specific context (private to test class)
private trait MySpecificTestContext extends YourFeatureCtx {
  // Very specific setup for this test only
  givenSomeSpecificData(...)
}
```

### 5.2 Context Rules

1. **Single Inheritance**: Each context extends exactly one parent
2. **Progressive Specialization**: Each level adds more specific setup
3. **Isolation**: Each test gets a fresh context instance (`new MyContext`)
4. **Override Methods**: Use `override` to customize parent behavior

### 5.3 Using Contexts in Tests

```scala
class YourFeatureTest extends YourServiceBaseTest {

  "YourFeature" should {
    
    // Option 1: Use existing context directly
    "return empty when no data exists" in new EmptyDataCtx {
      val result = blockingService.listItems(ListItemsRequest())
      result.items must beEmpty
    }

    // Option 2: Use specialized context
    "return single item" in new SingleItemCtx {
      val result = blockingService.getItem(GetItemRequest(id = Some(itemId)))
      result.item must beSome(item)
    }

    // Option 3: Create test-specific context inline
    "handle custom scenario" in new Ctx {
      // Inline setup for this specific test
      val specialItem = aExternalData(id = "special-id", name = "Special")
      givenExternalData(specialItem)
      
      val result = blockingService.getItem(GetItemRequest(id = Some("special-id")))
      result.item must beSome(specialItem)
    }
  }

  // Option 4: Define private context for this test file
  private trait SpecialScenarioCtx extends Ctx {
    val specialData = ...
    givenSpecialSetup()
  }
}
```

---

## 6. Nile-Generated Proto Artifacts

Nile generates several test artifacts from your proto files:

### 6.1 Generated Artifacts Overview

| Artifact | Bazel Target | Purpose |
|----------|--------------|---------|
| Messages | `:proto_scala_messages` | Proto message classes |
| Matchers | `:proto_scala_matchers` | Specs2 matchers for proto messages |
| Randoms | `:proto_scala_randoms` | Random message generators |
| Clients | `:proto_scala_platformized_clients` | Platformized gRPC clients |

### 6.2 Request Matchers

```scala
// Import matchers for your service
import matchers.com.wixpress.your.package.v1.YourServiceMatchers._
import matchers.com.wixpress.your.package.v1.QueryMatchers._
import matchers.com.wixpress.your.package.v1.QueryMatchers.CursorQueryMatchers._

// Use in expectations for downstream calls
"builds correct downstream request" in new Ctx {
  val expectedFilter = Struct().withFields(Map("status" -> Value().withStringValue("ACTIVE")))
  val expectedPaging = CursorPaging(limit = Some(1000))

  val expectedQueryMatcher = beCursorQuery(
    filter = beSome(expectedFilter),
    pagingMethod = beCursorPaging(===(expectedPaging))
  )

  // Mock downstream call with matcher
  downstreamClient.listItems(
    beListItemsRequest(
      query = beSome(expectedQueryMatcher)
    )
  )(haveSignatureBy("appDefId")) returns
    Future.successful(ListItemsResponse(...))

  // Execute
  blockingService.listItems(SomeRequest(...))

  // Verify call was made with expected request
  there was one(downstreamClient).listItems(
    beListItemsRequest(query = beSome(expectedQueryMatcher))
  )(haveSignatureBy("appDefId"))
}
```

### 6.3 Response Matchers

```scala
import matchers.com.wixpress.your.package.v1.YourServiceMatchers._
import matchers.com.wixpress.your.package.v1.ItemMatchers._

"returns expected items" in new Ctx {
  // Setup
  givenItems(Seq(item1, item2))

  // Execute
  val response = blockingService.listItems(ListItemsRequest(...))

  // Assert with matchers
  response must beListItemsResponse(
    items = contain(
      beItem(
        id = beSome("item-1"),
        status = beSome(Status.ACTIVE),
        name = beSome(contain("expected"))
      ),
      beItem(
        id = beSome("item-2"),
        status = beSome(Status.PENDING)
      )
    ).inOrder,
    pagingMetadata = beSome(
      bePagingMetadata(
        count = beSome(2),
        cursors = beSome(beCursors(next = beSome("cursor-123")))
      )
    )
  )
}
```

### 6.4 Random Generators

```scala
import com.wixpress.your.package.v1.YourProtoRandoms._
import com.wixpress.framework.proto.{GeneratorContext, OptionStrategy}

// Basic random generation
val randomItem = RandomItem()
  .withId(randomGuidAsString)
  .withName("Test Item")
  .update(_.someField.set(...))

// With GeneratorContext for option strategy
implicit val generatorContext: GeneratorContext = 
  GeneratorContext(optionStrategy = OptionStrategy.AlwaysSome)

val randomRequest = RandomCreateItemRequest()

// Random request for specific RPC
val request = YourServiceRandomRequests.YourService.RandomRequestForListItems()
```

---

## 7. Error and API Gateway Matchers

### 7.1 Error Matchers

```scala
import com.wixpress.framework.errors.WixResponseStatus
import com.wixpress.framework.errors.it.ErrorMatchers.{beApplicationError, beValidationError}

// Validation error matcher
"reject invalid input" in new Ctx {
  val invalidRequest = CreateItemRequest() // Missing required fields

  blockingService.createItem(invalidRequest) must beValidationError(
    responseMessage = beMatching(".*required.*")
  )
}

// Application error matcher
"return specific application error" in new Ctx {
  val conflictingRequest = CreateItemRequest(id = Some("existing-id"))
  givenItemExists("existing-id")

  blockingService.createItem(conflictingRequest) must beApplicationError(
    WixResponseStatus.FailedPrecondition,
    responseMessage = beEqualTo("Item already exists"),
    code = beEqualTo("ITEM_ALREADY_EXISTS"),
    description = contain("already exists")
  )
}

// Not found error
"return not found for missing item" in new Ctx {
  blockingService.getItem(GetItemRequest(id = Some("non-existent"))) must beApplicationError(
    WixResponseStatus.NotFound,
    code = beEqualTo("ITEM_NOT_FOUND")
  )
}
```

### 7.2 Custom Error Matchers

```scala
import io.grpc.{Status, StatusRuntimeException}
import org.specs2.matcher.Matcher

trait CustomErrorMatchers {
  this: Matchers =>

  def throwApiErrorWith(status: Status, text: String): Matcher[Any] = 
    throwA[StatusRuntimeException].like {
      case e: StatusRuntimeException =>
        e.getStatus.getCode.toString ==== status.getCode.toString
        e.getStatus.getDescription must contain(text)
    }

  def throwInvalidArgumentException(withDescription: String): Matcher[Any] =
    throwA[StatusRuntimeException](s"INVALID_ARGUMENT: .*$withDescription.*")
}
```

### 7.3 API Gateway Matchers

```scala
import com.wixpress.wixerd.ApiGatewayTestkitMatchers.haveSignatureBy
import com.wixpress.grpc.testkit.CallScopeMatchers._

// Assert downstream call is signed with correct app
"calls downstream with correct signature" in new Ctx {
  val appDefId = "my-app-def-id"
  
  downstreamClient.someMethod(any())(haveSignatureBy(appDefId)) returns
    Future.successful(SomeResponse())

  blockingService.triggerDownstreamCall(request)

  there was one(downstreamClient).someMethod(any())(haveSignatureBy(appDefId))
}

// Assert call scope has correct meta site ID
"propagates meta site ID" in new Ctx {
  there was one(downstreamClient).someMethod(any())(
    beCallScopeWithMetaSiteId(expectedMetaSiteId)
  )
}

// Assert call has same aspects as original
"preserves call scope aspects" in new Ctx {
  there was one(downstreamClient).someMethod(any())(haveSameAspects(callScope))
}
```

### 7.4 CallScope Matchers

```scala
import com.wixpress.grpc.testkit.CallScopeMatchers._

trait CallScopeMatchers {
  def beSignedCallScope: Matcher[CallScope] =
    beSignedCallScopeBy("default-app-def-id")

  def beSignedCallScopeBy(appId: String): Matcher[CallScope] =
    beSome(appId) ^^ (
      (_: CallScope)
        .aspect[IdentityResponse]
        .toList
        .flatMap(_.identificationData)
        .flatMap(_.identities)
        .find(_.identitySelector.isService)
        .map(_.getService.appDefId) aka "CallScope.*.service.appDefId"
    )

  def beCallScopeWithMetaSiteId(metaSiteId: String): Matcher[CallScope] =
    beSome(metaSiteId) ^^ (
      (_: CallScope).aspect[MetaSiteContext].map(_.metaSiteId)
    )
}
```

---

## 8. Fake Services Pattern

### 8.1 Purpose

Fake services provide **in-memory implementations** of external dependencies:
- Realistic behavior (filtering, searching)
- No external dependencies
- Fast execution
- Deterministic results

### 8.2 Implementation Pattern

```scala
import com.wixpress.hoopoe.common.inmemory.{FilterParser, InMemoryObjectSearcher}
import org.specs2.mock.Mockito.mock
import scala.concurrent.Future

class FakeExternalService {
  private var items: Seq[Item] = Vector.empty
  
  val mock = Mockito.mock[ExternalServicePlatformizedClientMethods]

  // Wire mock to use in-memory data
  mock.queryItems(any)(any) answers { args =>
    val request = args.head.asInstanceOf[QueryItemsRequest]
    val filter = request.getQuery.filter
    
    // Parse filter if using query language
    val filterClause = filter.flatMap(FilterParser.parse(_).toOption)
    
    // Filter items using InMemoryObjectSearcher
    val result = InMemoryObjectSearcher.search(
      items, 
      filterClause, 
      sort = None, 
      paging = None,
      cursorEncoder = None
    )
    
    Future.successful(QueryItemsResponse()
      .withItems(result.hits)
      .withPagingMetadata(PagingMetadata(count = Some(result.hits.size)))
    )
  }

  mock.getItem(any)(any) answers { args =>
    val request = args.head.asInstanceOf[GetItemRequest]
    val item = items.find(_.getId == request.getId)
    Future.successful(GetItemResponse(item = item))
  }

  mock.createItem(any)(any) answers { args =>
    val request = args.head.asInstanceOf[CreateItemRequest]
    val newItem = request.getItem.withId(randomGuidAsString)
    items = items :+ newItem
    Future.successful(CreateItemResponse(item = Some(newItem)))
  }

  mock.deleteItem(any)(any) answers { args =>
    val request = args.head.asInstanceOf[DeleteItemRequest]
    items = items.filterNot(_.getId == request.getId)
    Future.successful(DeleteItemResponse())
  }

  // Helper methods for test setup
  def givenItem(item: Item): Item = {
    items = items.filterNot(_.getId == item.getId) :+ item
    item
  }

  def givenItems(newItems: Seq[Item]): Unit =
    newItems.foreach(givenItem)

  def clear(): Unit =
    items = Vector.empty

  def allItems: Seq[Item] = items
}
```

### 8.3 Fake Service with Time-Based Filtering

```scala
class FakeTimeBasedService(defaultTimeZone: String = "UTC") {
  private var timeSlots: Seq[TimeSlot] = Seq.empty
  val mock = Mockito.mock[TimeBasedServicePlatformizedClientMethods]

  mock.queryTimeSlots(any)(any) answers { args =>
    val request = args.head.asInstanceOf[QueryTimeSlotsRequest]
    val timeZone = ZoneId.of(request.timeZone.getOrElse(defaultTimeZone))

    // Filter by date range
    val from = LocalDateTime.parse(request.getFromDate).atZone(timeZone).toInstant
    val to = LocalDateTime.parse(request.getToDate).atZone(timeZone).toInstant

    val slotsInRange = timeSlots
      .filter(s => s.getStart.asInstant.isBefore(to))
      .filter(s => s.getEnd.asInstant.isAfter(from))
      .map(enrichWithAdjustedDates(_, timeZone))

    // Apply additional query filters
    val filter = request.getQuery.filter
    val filterClause = filter.flatMap(FilterParser.parse(_).toOption)
    val filteredSlots = InMemoryObjectSearcher.search(slotsInRange, filterClause, None, None, None).hits

    Future.successful(QueryTimeSlotsResponse().withTimeSlots(filteredSlots))
  }

  def givenTimeSlot(slot: TimeSlot): TimeSlot = {
    val enrichedSlot = enrichWithUtcDates(slot)
    timeSlots = enrichedSlot +: timeSlots.filterNot(_.getId == slot.getId)
    enrichedSlot
  }

  private def enrichWithUtcDates(slot: TimeSlot): TimeSlot = {
    // Convert local dates to UTC
    slot.update(_.start.utcDate := ...).update(_.end.utcDate := ...)
  }

  private def enrichWithAdjustedDates(slot: TimeSlot, timeZone: ZoneId): TimeSlot = {
    // Adjust dates to requested timezone
    ...
  }
}
```

---

## 9. Loom Prime Testkits & E2E Tests

### 9.1 When to Use E2E Tests

**Default recommendation**: Prefer **in-memory service tests** with testkits and mocks over E2Es.

E2E tests are:
- ❌ Slower
- ❌ More fragile
- ❌ Harder to set up
- ❌ More expensive to maintain

Loom Prime already ensures composition, config, and proto wiring work correctly.

**Use E2E only when**:
- Testing database (SDL) interactions
- Testing Kafka (Greyhound) producers/consumers
- Testing external service integrations
- Verifying service composition and wiring

### 9.2 Enabling E2E in `prime_app`

```python
prime_app(
    name = "my-service",
    artifact = "com.myorg.myservice.my-service",
    service = "com.myorg.myservice.v1.MyService",
    with_e2e = True,   # opt-in for E2E tests
    # ...
)
```

This generates **IT environment classes**:
- `MyServiceITEnvironment`
  - `testEnvBuilder()` → `TestEnv`
  - `myServiceClient` (async)
  - `myServiceBlockingClient()` (blocking)

### 9.3 IT Environment Setup

```scala
package com.wixpress.your.package.v1

import com.wixpress.framework.loom.ManagedApp
import com.wixpress.framework.test.env.{TestEnv, TestEnvBuilder}
import com.wixpress.framework.test.rpc.LiteEmbeddedRpcServer
import com.wixpress.greyhound.KafkaManagedService
import com.wixpress.grpc.testkit.TestClients
import com.wixpress.hoopoe.config.TestConfigFactory.aTestEnvironmentFor
import com.wixpress.petri.util.LoomPetriTestkit
import com.wixpress.wixerd.ApiGatewayLiteTestkit
import io.grpc.ManagedChannel

object ITEnv {
  // Ports
  private val servicePort = 3210
  private val grpcPort = 3211
  private val managementPort = 9004
  private val petriPort = 9910
  private val rpcServerPort = 9070

  // Configure test environment
  def emitConfig(): Unit = aTestEnvironmentFor[YourServiceConfig](
    "your-service-name",
    "some.config.key" -> "test-value",
    "databag_passwd.appSecret" -> "test-secret"
  )

  // Testkits
  val apiGatewayTestKit = ApiGatewayLiteTestkit.randomPort()
  val petriTestkit = LoomPetriTestkit(petriPort, managementPort)

  // External service mocks (if needed)
  val externalServiceTestkit = new ExternalServiceTestkit()
  val rpcServer: LiteEmbeddedRpcServer = LiteEmbeddedRpcServer()
    .withServerPort(rpcServerPort)
    .withService[ExternalServiceApi](externalServiceTestkit.service)
    .build()

  // Main service
  private val app = new ManagedApp(
    YourServer, 
    port = servicePort, 
    grpcPort = grpcPort, 
    managementPort = managementPort
  )

  // Build test environment
  lazy val environment: TestEnv = TestEnvBuilder()
    .withEnvironmentConfigurer(() => ApiGatewayLiteTestkit.emitConfig())
    .withMainServiceConfigurer(() => emitConfig())
    .withCollaborators(
      KafkaManagedService(),     // If using Kafka
      MySqlManagedService(),     // If using SDL/MySQL
      apiGatewayTestKit,
      petriTestkit,
      rpcServer
    )
    .withMainService(app)
    .build()

  // gRPC channel
  lazy val grpcChannel: ManagedChannel = 
    TestClients.grpcChannel(s"localhost:$grpcPort")
}
```

### 9.4 IT Environment Support Trait

```scala
package com.wixpress.your.package.v1

import com.wixpress.framework.test.env.{GlobalTestEnvSupport, TestEnv}
import org.specs2.mutable.SpecificationWithJUnit
import org.specs2.specification.Scope

trait ITEnvSupport extends SpecificationWithJUnit with GlobalTestEnvSupport {
  sequential

  def testEnv: TestEnv = ITEnv.environment
}

trait BaseContext extends Scope {
  // Common context for IT tests
}
```

### 9.5 E2E Test Example

```scala
package com.wixpress.your.package.v1

import com.wix.core.testkit.identification.IdentityFactory
import com.wixpress.callscope.CallScopeUtils
import com.wixpress.framework.testkit.AwaitOps._
import com.wixpress.grpc.CallScope
import matchers.com.wixpress.your.package.v1.YourServiceMatchers._
import org.specs2.concurrent.ExecutionEnv
import org.specs2.specification.BeforeEach

import java.util.UUID

class YourServiceE2E(implicit ee: ExecutionEnv) 
    extends ITEnvSupport 
    with BeforeEach {

  override protected def before: Any = {
    ITEnv.petriTestkit.clearAndFetch()
    // Clean up test data if needed
  }

  "YourService" >> {
    "createItem" >> {
      "creates item successfully" in new BaseContext {
        implicit val callScope: CallScope = CallScopeUtils.callScopeWith(
          IdentityFactory.responseForExternalAppSite(
            "appDefId",
            UUID.randomUUID().toString,
            Set("YOUR_SERVICE.CREATE_ITEM")
          )
        )

        val request = CreateItemRequest(
          item = Some(Item().withName("Test Item"))
        )

        // Using async client
        val response = ITEnv.yourServiceClient
          .createItem(request)
          .await

        response must beCreateItemResponse(
          item = beSome(beItem(
            name = beSome("Test Item"),
            id = beSome[String]
          ))
        )
      }

      "returns validation error for invalid input" in new BaseContext {
        implicit val callScope: CallScope = testCallScope()

        val invalidRequest = CreateItemRequest() // Missing required fields

        ITEnv.yourServiceClient
          .createItem(invalidRequest) must beValidationError(
            responseMessage = beMatching(".*required.*")
          ).awaitFor(5.seconds)
      }
    }
  }

  private def testCallScope(): CallScope = CallScopeUtils.callScopeWith(
    IdentityFactory.responseForExternalAppSite(
      "appDefId",
      UUID.randomUUID().toString,
      Set.empty
    )
  )
}
```

### 9.6 Singleton TestEnv for Multiple E2E Suites

To avoid port collisions:

```scala
package com.wixpress.your.package.v1

import com.wixpress.framework.test.env.TestEnv

object YourServiceTestEnv {
  lazy val testEnv: TestEnv = YourServiceITEnvironment.testEnvBuilder().build()
}

// Use in all E2E test classes
class YourServiceFeature1E2E extends ITEnvSupport {
  override def testEnv: TestEnv = YourServiceTestEnv.testEnv
  // ...
}

class YourServiceFeature2E2E extends ITEnvSupport {
  override def testEnv: TestEnv = YourServiceTestEnv.testEnv
  // ...
}
```

---

## 10. Greyhound (Kafka) Testing

### 10.1 In-Memory Producer for Unit Tests

```scala
import com.wixpress.greyhound.DetailedProduceResult
import com.wixpress.grpc.CallScope
import scala.concurrent.Future

class InMemoryCallScopedProducer[T] extends CallScopedProducer[T] {
  private var messages: Seq[(String, T, CallScope)] = Seq.empty

  override def produce(key: String, message: T)(implicit callScope: CallScope): Future[DetailedProduceResult] = {
    messages = messages :+ (key, message, callScope)
    Future.successful(DetailedProduceResult.success())
  }

  def producedMessages: Seq[(String, T, CallScope)] = messages
  
  def clear(): Unit = messages = Seq.empty

  def lastMessage: Option[(String, T, CallScope)] = messages.lastOption
}
```

### 10.2 Using In-Memory Producer in Tests

```scala
trait Ctx extends Scope {
  val eventsProducer = new InMemoryCallScopedProducer[ItemNotification]
  
  val blockingService = contextBuilder
    .withEventsProducer(eventsProducer)
    .blockingPlatformized(...)
    .yourService

  def validateProducedNotification(expected: ItemNotification): Unit = {
    eventsProducer.producedMessages must contain { case (_, message, _) =>
      message must beEqualTo(expected)
    }
  }
}

"produces notification on create" in new Ctx {
  val request = CreateItemRequest(item = Some(testItem))
  
  blockingService.createItem(request)
  
  validateProducedNotification(ItemNotification(
    event = ItemEvent.CREATED,
    item = Some(testItem)
  ))
}
```

### 10.3 IT Tests with Real Kafka

```scala
import com.wixpress.greyhound.KafkaGreyhoundAdmin
import com.wixpress.greyhound.producer.builder.ProducerMaker
import com.wixpress.greyhound.consumer.ConsumerMaker

object ITEnv {
  // Kafka topic management
  def createTopics(): Unit = {
    val admin = new KafkaGreyhoundAdmin
    admin.createTopicIfNotExists("your-service-notifications")
    admin.createTopicIfNotExists("your-service-events")
  }

  lazy val environment: TestEnv = TestEnvBuilder()
    .withCollaborators(
      KafkaManagedService(),
      // ...
    )
    .withMainService(app)
    .build()
}

class YourServiceKafkaE2E extends ITEnvSupport with BeforeAll {
  
  override def beforeAll(): Unit = {
    ITEnv.createTopics()
  }

  "produces event to Kafka" in new BaseContext {
    implicit val cs: CallScope = testCallScope()
    
    // Create item
    val response = ITEnv.yourServiceClient.createItem(request).await
    
    // Wait for Kafka message
    eventually {
      val messages = consumeMessages("your-service-notifications")
      messages must contain(expectedNotification)
    }
  }
}
```

### 10.4 Consumer Testing

```scala
import com.wixpress.greyhound.testkit.GreyhoundTestkit

trait Ctx extends Scope {
  val greyhoundTestkit = GreyhoundTestkit()
  
  val consumer = new YourEventConsumer(
    service = blockingService,
    // ...
  )

  def produceEvent(event: SomeEvent): Unit = {
    greyhoundTestkit.produce("topic-name", event)
  }

  def waitForConsumer(): Unit = {
    Thread.sleep(100) // Or use eventually/await patterns
  }
}

"consumes event and updates state" in new Ctx {
  val event = SomeEvent(itemId = "123", action = Action.UPDATE)
  
  produceEvent(event)
  waitForConsumer()
  
  // Verify side effect
  there was one(mockRepository).update(any())
}
```

---

## 11. Bazel Wiring

### 11.1 Using `prime_app` Macro

```python
load("@server_infra//framework/loom-prime:macros.bzl", "prime_app")

prime_app(
    name = "your-service-name",
    artifact = "com.wixpress.your.package.your-service-name",
    defaults = "com.wix.bookings.common.loom.defaults.BookingsLoomDefaults.Defaults",
    
    # Service RPC definitions
    rpcs = ["com.wixpress.your.package.v1.YourService"],
    
    # SDL database configuration
    sdl = {
        "your_database": {
            "schema": {
                "your_entity": "com.wixpress.your.package.domain.YourEntity",
            },
        },
    },
    
    # Proto dependencies
    proto_deps = [
        "//bookings-backend/common:proto",
        "@meta_site//reloose/reloose-protos/src/main/proto",
    ],
    
    # Runtime dependencies
    deps = [
        "//bookings-backend/common/src/main/scala/com/wix/bookings/common/core",
        "@server_infra//framework/loom/application/src/main/scala/com/wixpress/framework/loom",
        "@io_grpc_grpc_api",
    ],
    
    # Test-only dependencies
    deps_test = [
        # Specs2
        "@org_specs2_specs2_common_2_12",
        "@org_specs2_specs2_core_2_12",
        "@org_specs2_specs2_matcher_2_12",
        
        # Proto test utilities
        ":proto_scala_matchers",
        ":proto_scala_randoms",
        "//bookings-backend/downstream-service:proto_scala_matchers",
        
        # Wix testkits
        "@server_infra//framework/errors/matchers/src/main/scala/com/wixpress/framework/errors/it",
        "@server_infra//framework/grpc/testkit/main/scala/com/wixpress/grpc/testkit",
        "@server_infra//iptf/wixerd/api-gateway-client-parent/api-gateway-lite-testkit/src/main/scala/com/wixpress/wixerd",
        "@wix_framework//hoopoe-common/hoopoe-utest",
        
        # Dynamic config testkit
        "@velocity_infra//conductor/dynamic-config/library/testkit",
    ],
    
    # IT test dependencies (only if with_e2e = True)
    deps_it_test = [
        "@wix_framework//hoopoe-it-framework",
        # ...
    ],
    
    # Configuration
    configs = {
        "appSecret": "databag_passwd('com.wixpress.your.package.app-secret', 'appSecret')",
    },
    
    # Feature flags
    features = [
        "opt-out-domain-event-strict-consumer-classes",
    ],
    
    # Enable E2E tests (opt-in)
    with_e2e = False,  # Set to True when needed
)
```

### 11.2 Common Bazel Targets

| Target | Description |
|--------|-------------|
| `:proto` | Proto definitions |
| `:proto_scala` | Generated Scala code |
| `:proto_scala_messages` | Message classes |
| `:proto_scala_matchers` | Test matchers |
| `:proto_scala_randoms` | Random generators |
| `:proto_scala_platformized_clients` | Platformized gRPC clients |
| `:test` | Unit tests |
| `:it` | Integration tests |

### 11.3 Running Tests

```bash
# Run all tests for a service
bazel test //bookings-backend/your-service-name:test

# Run specific test file
bazel test //bookings-backend/your-service-name:test --test_filter="YourFeatureTest"

# Run with output
bazel test //bookings-backend/your-service-name:test --test_output=all

# Run tests matching a pattern
bazel test //bookings-backend/your-service-name:test --test_filter="*Integration*"

# Run IT tests
bazel test //bookings-backend/your-service-name:it

# Run all tests in module
bazel test //bookings-backend/your-service-name/...

# Query test dependencies
bazel query 'deps(//bookings-backend/your-service-name:test)'
```

---

## 12. Best Practices

### 12.1 ✅ DO

#### Test Organization
```scala
// Use sequential for tests that share state
abstract class MyBaseTest extends SpecWithJUnit {
  sequential
}

// Clear test names
"Feature name" should {
  "scenario description" in new Context {
    // Test code
  }
}

// Group related tests
"ListItems" should {
  "single item scenarios" >> {
    "returns item when exists" in new SingleItemCtx { ... }
    "returns empty when not found" in new EmptyCtx { ... }
  }
  "multiple items scenarios" >> {
    "returns all items" in new MultipleItemsCtx { ... }
    "filters by status" in new MultipleItemsCtx { ... }
  }
}
```

#### Helper Methods
```scala
trait Ctx extends Scope {
  // given* methods for setup
  def givenItem(item: Item): Item = fakeService.givenItem(item)
  def givenItemExists(id: String): Item = givenItem(aItem(id = id))
  
  // a* methods for builders
  def aItem(
    id: String = randomGuidAsString,
    name: String = randomStr(),
    status: Status = Status.ACTIVE
  ): Item = Item().withId(id).withName(name).withStatus(status)
  
  // verify* methods for assertions
  def verifyItemCreated(expected: Item): MatchResult[Any] =
    fakeService.allItems must contain(expected)
}
```

#### Use Nile Matchers
```scala
// ✅ Good: Use Nile matchers
response must beListItemsResponse(
  items = contain(beItem(id = beSome("123")))
)

// ❌ Bad: Manual comparison
response.items.head.id must beSome("123")
```

#### Test Isolation
```scala
// ✅ Good: Fresh context per test
"test 1" in new Ctx { ... }
"test 2" in new Ctx { ... }  // Fresh state

// ❌ Bad: Shared mutable state outside context
var sharedState = ...
"test 1" in new Ctx { sharedState = ... }
"test 2" in new Ctx { /* uses stale sharedState */ }
```

### 12.2 ❌ DON'T

#### Don't Mix Concerns
```scala
// ❌ Bad: Multiple scenarios in one test
"handle all edge cases" in new Ctx {
  // Test create
  val created = blockingService.create(...)
  // Test update
  val updated = blockingService.update(...)
  // Test delete
  blockingService.delete(...)
}

// ✅ Good: One scenario per test
"creates item" in new Ctx { ... }
"updates item" in new Ctx { ... }
"deletes item" in new Ctx { ... }
```

#### Don't Rely on Test Order
```scala
// ❌ Bad: Tests depend on order
"1. create item" in new Ctx {
  createdItem = blockingService.create(...)
}
"2. update created item" in new Ctx {
  blockingService.update(createdItem.id, ...)  // Depends on previous test
}

// ✅ Good: Independent tests
"updates existing item" in new SingleItemCtx {
  blockingService.update(itemId, ...)
}
```

#### Don't Use E2E When Unit Tests Suffice
```scala
// ❌ Bad: E2E test for simple logic
class SimpleValidationE2E extends ITEnvSupport {
  "rejects invalid input" in { ... }  // Overkill!
}

// ✅ Good: Unit test for simple logic
class SimpleValidationTest extends BaseTest {
  "rejects invalid input" in new Ctx {
    blockingService.create(invalidInput) must beValidationError(...)
  }
}
```

#### Don't Duplicate Setup
```scala
// ❌ Bad: Duplicated setup
"test 1" in new Ctx {
  val item = Item().withId("123").withName("Test").withStatus(ACTIVE)
  givenItem(item)
  ...
}
"test 2" in new Ctx {
  val item = Item().withId("456").withName("Test").withStatus(ACTIVE)
  givenItem(item)
  ...
}

// ✅ Good: Extract to context
trait ActiveItemCtx extends Ctx {
  val item = aItem(status = ACTIVE)
  givenItem(item)
}
```

---

## 13. Complete Examples

### 13.1 Complete Base Test

```scala
package com.wixpress.bookings.items.v1

import org.specs2.mutable.SpecWithJUnit
import org.specs2.matcher.{FutureMatchers, Matchers}
import org.specs2.mock.Mockito
import org.specs2.specification.Scope
import org.specs2.concurrent.ExecutionEnv

import com.wixpress.grpc.testkit.TestCallScope
import com.wixpress.wixerd.ApiGatewayTestkitMatchers.haveSignatureBy
import com.wixpress.framework.errors.WixResponseStatus
import com.wixpress.framework.errors.it.ErrorMatchers.{beApplicationError, beValidationError}
import com.wixpress.hoopoe.ids.randomGuidAsString
import com.wixpress.hoopoe.test.{randomStr, randomBoolean}
import com.wixpress.bookings.items.v1.ItemRandoms._

import scala.concurrent.Future
import scala.concurrent.duration._

abstract class ItemsServiceBaseTest
    extends SpecWithJUnit
    with FutureMatchers
    with Matchers
    with Mockito {

  sequential

  trait Ctx extends Scope {
    implicit val callScope: TestCallScope = TestCallScope()

    // Fake services
    val fakeResourceService = new FakeResourceService()
    val fakeScheduleService = new FakeScheduleService()

    // Mock services
    val notificationClient = mock[NotificationServicePlatformizedClientMethods]

    // Context builder
    val contextBuilder = ItemsServiceTestContextBuilder(
      secrets = ItemsServiceSecrets(appSecret = randomGuidAsString),
      config = ItemsServiceConfig(),
      rpc = ItemsServiceTestContextBuilder.mockRpc.copy(
        resourceService = fakeResourceService.mock,
        scheduleService = fakeScheduleService.mock,
        notificationClient = notificationClient
      )
    )

    // Service under test
    val blockingService: BlockingItemsServiceLoomPrimed =
      contextBuilder
        .blockingPlatformized(ItemsService(contextBuilder), 90.seconds)
        .itemsService

    // Default mock responses
    notificationClient.sendNotification(any())(any()) returns Future.successful(SendNotificationResponse())

    // Helper methods
    def givenResource(resource: Resource): Resource =
      fakeResourceService.givenResource(resource)

    def givenSchedule(schedule: Schedule): Schedule =
      fakeScheduleService.givenSchedule(schedule)

    def aItem(
      id: String = randomGuidAsString,
      name: String = randomStr(),
      status: ItemStatus = ItemStatus.ACTIVE,
      resourceId: Option[String] = None
    ): Item = RandomItem()
      .withId(id)
      .withName(name)
      .withStatus(status)
      .update(_.resourceId.setIfDefined(resourceId))

    def aResource(
      id: String = randomGuidAsString,
      name: String = randomStr()
    ): Resource = RandomResource().withId(id).withName(name)

    def aSchedule(
      id: String = randomGuidAsString,
      resourceId: String = randomGuidAsString
    ): Schedule = RandomSchedule().withId(id).withResourceId(resourceId)
  }

  // Specialized contexts
  trait EmptyStateCtx extends Ctx

  trait SingleResourceCtx extends Ctx {
    val resource = aResource()
    givenResource(resource)
  }

  trait ResourceWithScheduleCtx extends SingleResourceCtx {
    val schedule = aSchedule(resourceId = resource.getId)
    givenSchedule(schedule)
  }
}
```

### 13.2 Complete Feature Test

```scala
package com.wixpress.bookings.items.v1

import matchers.com.wixpress.bookings.items.v1.ItemsServiceMatchers._
import matchers.com.wixpress.bookings.items.v1.ItemMatchers._
import com.wixpress.framework.errors.WixResponseStatus

class CreateItemTest extends ItemsServiceBaseTest {

  "CreateItem" should {
    
    "create item with valid input" in new ResourceWithScheduleCtx {
      val request = CreateItemRequest(
        item = Some(aItem(name = "New Item", resourceId = Some(resource.getId)))
      )

      val response = blockingService.createItem(request)

      response must beCreateItemResponse(
        item = beSome(beItem(
          name = beSome("New Item"),
          resourceId = beSome(resource.getId),
          status = beSome(ItemStatus.ACTIVE)
        ))
      )
    }

    "send notification on create" in new ResourceWithScheduleCtx {
      val request = CreateItemRequest(
        item = Some(aItem(resourceId = Some(resource.getId)))
      )

      blockingService.createItem(request)

      there was one(notificationClient).sendNotification(
        beSendNotificationRequest(
          eventType = beSome(EventType.ITEM_CREATED)
        )
      )(haveSignatureBy("items-service-app-def-id"))
    }

    "return validation error for missing name" in new ResourceWithScheduleCtx {
      val request = CreateItemRequest(
        item = Some(Item().withResourceId(resource.getId))
      )

      blockingService.createItem(request) must beValidationError(
        responseMessage = beMatching(".*name.*required.*")
      )
    }

    "return not found for non-existent resource" in new EmptyStateCtx {
      val request = CreateItemRequest(
        item = Some(aItem(resourceId = Some("non-existent")))
      )

      blockingService.createItem(request) must beApplicationError(
        WixResponseStatus.NotFound,
        code = beEqualTo("RESOURCE_NOT_FOUND")
      )
    }
  }

  // Private context for edge case
  private trait DuplicateNameCtx extends ResourceWithScheduleCtx {
    val existingItem = aItem(name = "Existing", resourceId = Some(resource.getId))
    fakeItemRepository.givenItem(existingItem)
  }

  "handle duplicate name" in new DuplicateNameCtx {
    val request = CreateItemRequest(
      item = Some(aItem(name = "Existing", resourceId = Some(resource.getId)))
    )

    blockingService.createItem(request) must beApplicationError(
      WixResponseStatus.AlreadyExists,
      code = beEqualTo("ITEM_NAME_EXISTS")
    )
  }
}
```

### 13.3 Complete E2E Test

```scala
package com.wixpress.bookings.items.v1

import com.wix.core.testkit.identification.IdentityFactory
import com.wixpress.callscope.CallScopeUtils
import com.wixpress.framework.testkit.AwaitOps._
import com.wixpress.grpc.CallScope
import matchers.com.wixpress.bookings.items.v1.ItemsServiceMatchers._
import org.specs2.concurrent.ExecutionEnv
import org.specs2.specification.BeforeEach

import java.util.UUID
import scala.concurrent.duration._

class ItemsServiceE2E(implicit ee: ExecutionEnv) 
    extends ITEnvSupport 
    with BeforeEach {

  override protected def before: Any = {
    ITEnv.petriTestkit.clearAndFetch()
    ITEnv.cleanDatabase()
  }

  "ItemsService E2E" >> {
    
    "full CRUD lifecycle" >> {
      
      "create, read, update, delete" in new E2EContext {
        // Create
        val createRequest = CreateItemRequest(
          item = Some(Item().withName("E2E Test Item"))
        )
        val created = client.createItem(createRequest).await.getItem
        created.getName must beEqualTo("E2E Test Item")

        // Read
        val getResponse = client.getItem(GetItemRequest(id = created.id)).await
        getResponse.item must beSome(created)

        // Update
        val updateRequest = UpdateItemRequest(
          item = Some(created.withName("Updated Name")),
          updateMask = Some(FieldMask(paths = Seq("name")))
        )
        val updated = client.updateItem(updateRequest).await.getItem
        updated.getName must beEqualTo("Updated Name")

        // Delete
        client.deleteItem(DeleteItemRequest(id = updated.id)).await
        
        // Verify deleted
        client.getItem(GetItemRequest(id = updated.id)) must 
          beApplicationError(WixResponseStatus.NotFound).awaitFor(5.seconds)
      }
    }
    
    "produces Kafka event on create" in new E2EContext {
      val request = CreateItemRequest(
        item = Some(Item().withName("Kafka Test"))
      )

      val response = client.createItem(request).await

      // Verify Kafka message was produced
      eventually(timeout = 10.seconds) {
        val messages = ITEnv.consumeItemEvents()
        messages must contain { event: ItemEvent =>
          event.eventType == EventType.CREATED &&
          event.getItem.getName == "Kafka Test"
        }
      }
    }
  }

  trait E2EContext extends Scope {
    implicit val callScope: CallScope = CallScopeUtils.callScopeWith(
      IdentityFactory.responseForExternalAppSite(
        "items-service-app-def-id",
        UUID.randomUUID().toString,
        Set("ITEMS_SERVICE.MANAGE")
      )
    )

    val client = ITEnv.itemsServiceBlockingClient()(callScope)
  }
}
```

---

## 14. Troubleshooting

### 14.1 Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| `Helper file treated as test` | File ends with `Test` or `E2E` | Rename to `*BaseTest`, `*Helper`, `*Utils` |
| `TestCallScope not found` | Missing import or dependency | Add `@server_infra//framework/grpc/testkit` to deps |
| `Matcher not found` | Wrong import | Import `matchers.com.wixpress...` for Nile matchers |
| `Mock not working` | Using wrong mock syntax | Use `any()` for any arg, `===()` for exact match |
| `Future assertion fails` | Not awaiting | Add `.await` or use `must beEqualTo(...).await` |
| `Port collision in IT` | Multiple TestEnv instances | Use singleton `lazy val testEnv` |

### 14.2 Debugging Tips

```scala
// Print mock invocations
println(mockService.toString)

// Debug async failures
response.await(30.seconds) // Longer timeout

// Capture verification failures
there was one(mock).method(...) // Fails? Add:
println(Mockito.mockingDetails(mock).printInvocations())

// Debug proto matchers
response must beResponse(
  field = beSome[String].eventually  // Retries
)
```

### 14.3 Error Messages

```scala
// Better error messages with aliases
response.items.headOption.map(_.id) must 
  beSome("expected-id") aka "first item id"

// Custom matcher messages
def beActiveItem: Matcher[Item] = 
  beSome(ItemStatus.ACTIVE) ^^ ((_: Item).status aka "item status")
```

---

## 15. Quick Reference

### 15.1 Imports Cheat Sheet

```scala
// Core testing
import org.specs2.mutable.SpecWithJUnit
import org.specs2.matcher.{FutureMatchers, Matchers}
import org.specs2.mock.Mockito
import org.specs2.specification.Scope

// Wix testing
import com.wixpress.grpc.testkit.TestCallScope
import com.wixpress.framework.errors.it.ErrorMatchers._
import com.wixpress.wixerd.ApiGatewayTestkitMatchers.haveSignatureBy
import com.wixpress.grpc.testkit.CallScopeMatchers._

// Randoms
import com.wixpress.hoopoe.ids.randomGuidAsString
import com.wixpress.hoopoe.test.{randomStr, randomBoolean, randomInt}

// Nile matchers (replace with your package)
import matchers.com.wixpress.your.package.v1.YourServiceMatchers._
import com.wixpress.your.package.v1.YourProtoRandoms._
```

### 15.2 Matchers Cheat Sheet

```scala
// Basic
x must beEqualTo(expected)
x must beSome(value)
x must beNone
x must beEmpty
x must not(beEmpty)
x must haveSize(5)

// Collections
list must contain(item)
list must contain(exactly(item1, item2))
list must contain(exactly(...)).inOrder

// Strings
s must contain("substring")
s must beMatching("regex.*pattern")
s must startWith("prefix")
s must endWith("suffix")

// Futures
future must beEqualTo(value).await
future must beEqualTo(value).awaitFor(5.seconds)

// Errors
result must beValidationError(responseMessage = beMatching(".*"))
result must beApplicationError(WixResponseStatus.NotFound, code = beEqualTo("CODE"))

// Proto matchers
response must beListItemsResponse(items = haveSize(1))
item must beItem(id = beSome("123"), status = beSome(Status.ACTIVE))
```

### 15.3 Mock Cheat Sheet

```scala
// Setup
val mock = mock[ServiceClient]
mock.method(any())(any()) returns Future.successful(Response())
mock.method(===(specificArg))(any()) returns Future.successful(Response())
mock.method(any())(any()) throws new RuntimeException("error")

// Verification
there was one(mock).method(any())(any())
there was two(mock).method(...)(...)
there was no(mock).method(...)(...)
there was one(mock).method(beTypedEqualTo(expectedArg))(any())
there was one(mock).method(any())(haveSignatureBy("appDefId"))

// Answers (dynamic responses)
mock.method(any())(any()) answers { args =>
  val request = args.head.asInstanceOf[Request]
  Future.successful(Response(id = request.id))
}
```

### 15.4 Bazel Cheat Sheet

```bash
# Build
bazel build //bookings-backend/service:target

# Test
bazel test //bookings-backend/service:test
bazel test //bookings-backend/service:test --test_filter="TestClass"
bazel test //bookings-backend/service:test --test_output=all

# IT tests
bazel test //bookings-backend/service:it

# Query
bazel query 'deps(//bookings-backend/service:test)'
bazel query 'rdeps(//..., //bookings-backend/service:lib)'

# Clean
bazel clean --expunge
```

---

## Summary Checklist

When creating tests for a new Wix Scala service:

- [ ] Create base test class (`YourServiceBaseTest.scala`)
  - [ ] Extends `SpecWithJUnit` + `FutureMatchers` + `Matchers` + `Mockito`
  - [ ] Defines `trait Ctx extends Scope`
  - [ ] Sets up `TestCallScope`, context builder, mocks
  - [ ] Provides helper methods (`given*`, `a*`)

- [ ] Use Nile-generated artifacts
  - [ ] Import proto matchers (`matchers.com.wixpress...`)
  - [ ] Import proto randoms (`YourProtoRandoms`)
  - [ ] Add to `deps_test` in BUILD.bazel

- [ ] Create fake services for external dependencies
  - [ ] In-memory storage
  - [ ] Mock object with realistic behavior
  - [ ] Helper methods for test setup

- [ ] Write tests using context pattern
  - [ ] One scenario per test
  - [ ] Use `new ContextName` for each test
  - [ ] Use Nile matchers for assertions
  - [ ] Use error matchers for error cases

- [ ] Follow naming conventions
  - [ ] Tests: `*Test.scala` or `*E2E.scala`
  - [ ] Helpers: `*BaseTest.scala`, `*Helper.scala`, `*Utils.scala`
  - [ ] Fakes: `Fake*.scala`

- [ ] Configure Bazel correctly
  - [ ] Add proto matchers/randoms to `deps_test`
  - [ ] Add testkits to `deps_test`
  - [ ] Enable `with_e2e` only if needed

---

*Last updated: January 2026*
*Applies to: Wix Bookings/Scheduler monorepo and similar Loom Prime services*

