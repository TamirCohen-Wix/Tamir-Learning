---
description: STRICT Test-Driven Development for Ninja/Java Loom Prime. Forces Red-Green-Refactor cycle. Will NOT write implementation code before a failing test exists. (user)
allowed-tools:
  - Read
  - Edit
  - Write
  - Glob
  - Grep
  - Bash(bazel:*)
  - Bash(/usr/local/bin/bazel:*)
  - TaskCreate
  - TaskUpdate
---

# Ninja TDD Skill (Strict)

STRICT Test-Driven Development workflow for Ninja (Java) services using JUnit 5, Mockito, and Loom Prime with Virtual Threads. This skill enforces the Red-Green-Refactor cycle without exceptions.

## Usage
```
/tdd-ninja <feature-or-bug-description>
```

## Arguments
- `$ARGUMENTS` - Description of the feature to implement or bug to fix

## STRICT ENFORCEMENT RULES

**These rules are NON-NEGOTIABLE:**

1. **NO IMPLEMENTATION WITHOUT A FAILING TEST** - You MUST write a test first and see it fail before writing ANY implementation code
2. **RED PHASE IS MANDATORY** - Every test MUST be run and MUST fail before proceeding
3. **ONE TEST AT A TIME** - Write one test, see it fail, implement, see it pass, repeat
4. **NO SHORTCUTS** - Do not skip the Red phase even for "obvious" implementations
5. **TRACK EVERY STEP** - Use TaskCreate/TaskUpdate to track the TDD cycle meticulously

## Instructions

### Step 1: Understand & Plan

1. Parse `$ARGUMENTS` to understand the requirement
2. If unclear, ask clarifying questions - DO NOT PROCEED with assumptions
3. Create a task list with test cases to implement (use TaskCreate):
   ```
   - [ ] Write test: [test case 1 description]
   - [ ] Run test (must fail - RED)
   - [ ] Implement minimal code (GREEN)
   - [ ] Refactor if needed
   - [ ] Write test: [test case 2 description]
   - ... repeat for all cases
   ```

### Step 2: Locate Test Infrastructure

Find existing test files and patterns:
```java
// Look for TestApp and TestContextBuilder
*TestApp.java
*TestContextBuilder.java

// Test directories
test/           // Unit tests (hexagon tests)
it/             // Integration/E2E tests
```

### Step 3: Write the Failing Test (RED)

Create or update the test file using this structure:

```java
import com.wixpress.framework.RequestContextExtension;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import static com.wixpress.project.v1.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;
import static org.mockito.Mockito.verify;

class MyFeatureTest {
    private MyServiceNinjaTestContextBuilder contextBuilder;
    private MyServiceNinjaService service;
    private DependencyClient dependencyClient;

    @RegisterExtension
    RequestContextExtension requestContext = RequestContextExtension.empty();

    @BeforeEach
    void setUp() {
        var app = MyServiceTestApp.create(context -> context);
        contextBuilder = app.contextBuilder();
        service = app.myService();

        // Get RPC clients (already Mockito mocks)
        dependencyClient = contextBuilder.rpc().dependencyClient();
    }

    @Test
    void should_do_expected_behavior_when_given_condition() {
        // Arrange (Given)
        var request = SomeRequest.newBuilder()
                .setId("test-id")
                .build();
        givenDependencyReturns(expectedData);

        // Act (When)
        var response = service.doSomething(request);

        // Assert (Then)
        assertThat(response).hasExpectedField("expectedValue");
        verify(dependencyClient).someMethod(any());
    }

    // Helper methods - use given* prefix
    private void givenDependencyReturns(Data data) {
        when(dependencyClient.getData(any())).thenReturn(
            DataResponse.newBuilder().setData(data).build()
        );
    }
}
```

**Test Writing Rules:**
- Use descriptive test names: `should_[expected]_when_[condition]`
- Follow Arrange-Act-Assert pattern strictly
- One assertion focus per test
- Use `given*` prefix for mock setup methods
- RPC clients from `contextBuilder.rpc()` are already Mockito mocks

### Step 4: Run the Test (VERIFY RED)

**THIS STEP IS MANDATORY - DO NOT SKIP**

```bash
/usr/local/bin/bazel test //path/to:test_test_runner --test_timeout=30000
```

**The test MUST fail.** Report to the user:
- The test name
- The failure reason
- Confirmation this is the expected failure

If the test passes unexpectedly:
- STOP and investigate
- Either the feature exists or the test is wrong
- DO NOT proceed to implementation

### Step 5: Implement Minimal Code (GREEN)

**ONLY AFTER CONFIRMING THE TEST FAILS:**

1. Write the MINIMUM code to make the test pass
2. NO extra features, NO optimizations, NO "while I'm here" additions
3. Follow existing patterns in the codebase

**Ninja/Java Guidelines:**

- **Virtual Threads** - Write synchronous-looking code, JVM handles async:
  ```java
  // Good - looks synchronous but is non-blocking
  var data = rpcClient.fetchData(request.getId());
  return buildResponse(data);
  ```

- **Parallel Operations** - Use Structured Concurrency:
  ```java
  import java.util.concurrent.StructuredTaskScope;

  try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
      var task1 = scope.fork(() -> rpcClient.call1());
      var task2 = scope.fork(() -> rpcClient.call2());
      scope.join();
      scope.throwIfFailed();
      return combine(task1.get(), task2.get());
  }
  ```

- **Proto Builders** - Always use builders for proto messages:
  ```java
  return Response.newBuilder()
          .setId(id)
          .setName(name)
          .build();
  ```

- **Dependency Injection** - Extract specific dependencies in AppBuilder:
  ```java
  // Good
  public MyServiceImpl(RpcClient client, Visibility visibility) {
      this.client = client;
      this.visibility = visibility;
  }
  ```

- **Visibility/Logging**:
  ```java
  import com.wixpress.framework.ninja.LogEvent;

  visibility.expose(LogEvent.info("Processing request"));
  visibility.expose(LogEvent.info("Details", Map.of("id", id)));
  ```

### Step 6: Run the Test (VERIFY GREEN)

```bash
/usr/local/bin/bazel test //path/to:test_test_runner --test_timeout=30000
```

**The test MUST pass now.**

If it fails:
- Analyze the failure
- Fix the implementation (NOT the test, unless the test is genuinely wrong)
- DO NOT move to the next test until this one passes

Report to the user:
- Confirmation the test passes
- Mark the task as completed (use TaskUpdate)

### Step 7: Refactor (Optional)

ONLY refactor if:
- There's obvious code duplication
- Names can be more descriptive
- Logic can be simplified without changing behavior

After refactoring, run the test again:
```bash
/usr/local/bin/bazel test //path/to:test_test_runner --test_timeout=30000
```

### Step 8: Repeat

For each additional test case:
1. Create task for next test case
2. Go back to Step 3 (Write the Failing Test)
3. Continue the cycle

### Step 9: Final Verification

Run ALL related tests:
```bash
/usr/local/bin/bazel test //path/to/package:test_test_runner --test_timeout=30000
```

Ensure no regressions.

## Mocking Patterns

### RPC Client Mocking (Primary)
```java
// RPC clients from contextBuilder.rpc() are already Mockito mocks
var rpcClient = contextBuilder.rpc().someRpc();

// Mock success
when(rpcClient.getData(any())).thenReturn(
    DataResponse.newBuilder().setData(data).build()
);

// Mock error
when(rpcClient.getData(any())).thenThrow(
    RpcClientExceptionBuilder.newRpcClientException()
        .withStatus(WixResponseStatus.NotFound)
        .withResponseMessage("Entity not found")
        .asApplicationException()
        .withDetails("NOT_FOUND", "Entity not found")
        .build()
);

// Verify calls
verify(rpcClient).getData(any());
```

### Feature Toggle Mocking
```java
var app = MyServiceTestApp.create(context -> {
    context.featureToggle().mock()
        .myToggleName()
        .returns(true);
    return context;
});
```

### Clock Mocking
```java
var app = MyServiceTestApp.create(context -> {
    context.clock().setNow(Instant.parse("2024-01-15T10:00:00Z"));
    return context;
});

// Advance time in test
contextBuilder.clock().advance(Duration.ofHours(1));
```

### SDL Test Data
```java
@BeforeEach
void setUp() {
    var app = MyServiceTestApp.create(context -> context);
    contextBuilder = app.contextBuilder();

    // Insert test data
    contextBuilder.sdl().insert(
        MyEntity.newBuilder()
            .setId("test-id")
            .setName("Test")
            .build()
    );
}
```

### Config Mocking
```java
@BeforeEach
void setUp() {
    var config = MyServiceNinjaConfig.builder()
        .actionKey("test-action-key")
        .maxRetries(1)
        .build();

    var app = MyServiceTestApp.create(context -> context.config(config));
    service = app.myService();
}
```

## Testing Visibility Events

```java
import com.wixpress.framework.ninja.CollectingVisibility;
import static com.wixpress.framework.ninja.visibility.Assertions.assertThat;

class MyServiceTest {
    private CollectingVisibility visibility;

    @BeforeEach
    void setUp() {
        var app = MyServiceTestApp.create(context -> context);
        visibility = contextBuilder.build().visibility();
        service = app.myService();
    }

    @Test
    void should_log_action_invoked() {
        service.invoke(request);

        assertThat(visibility)
            .hasLogEvent()
            .withMessage("ActionInvoked")
            .withLevel(Level.INFO);
    }
}
```

## Generated AssertJ Matchers

Ninja generates AssertJ assertions for proto messages:

```java
import static com.wixpress.project.v1.Assertions.assertThat;

assertThat(response).hasId("expected-id");
assertThat(response).hasName("expected-name");
assertThat(response)
    .hasId("id")
    .hasStatus(Status.ACTIVE)
    .hasCreatedAt(expectedTimestamp);
```

## VIOLATIONS - DO NOT DO THESE

- Writing implementation before a failing test
- Skipping the Red phase for "simple" changes
- Writing multiple tests before running any
- Implementing more than needed to pass the current test
- Modifying tests to pass instead of fixing implementation
- Using blocking I/O in hot paths without Virtual Threads
- Passing entire context to service instead of specific dependencies

## Examples

```
/tdd-ninja Add validation that action key must not be empty
/tdd-ninja Fix: Service returns 500 when external RPC times out
/tdd-ninja Implement retry logic with exponential backoff for RPC calls
```