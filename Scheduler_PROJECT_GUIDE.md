# Wix Bookings/Scheduler Project Guide

## Table of Contents
1. [Project Overview](#project-overview)
2. [Repository Structure](#repository-structure)
3. [Technology Stack](#technology-stack)
4. [Core Modules](#core-modules)
5. [Architecture Patterns](#architecture-patterns)
6. [Build System](#build-system)
7. [Development Workflow](#development-workflow)
8. [Key Concepts](#key-concepts)
9. [Team Ownership](#team-ownership)
10. [Resources & Links](#resources--links)

---

## Project Overview

**Wix Bookings** is a comprehensive scheduling and booking platform that enables businesses to manage appointments, classes, services, and resources. This monorepo contains both backend services and frontend modules for the entire bookings ecosystem.

### Core Domain Entities
- **Bookings**: Customer appointments/reservations
- **Services**: Business offerings (classes, appointments, courses)
- **Resources**: Staff members, equipment, or locations
- **Schedules**: Time-based availability definitions
- **Sessions**: Specific time slots (booked or available)
- **Slots**: Available booking opportunities

### Booking Lifecycle
- `CREATED` → `PENDING` → `CONFIRMED` (or `DECLINED`)
- `WAITING_LIST` (for classes at capacity)
- `CANCELED` (customer-initiated)

---

## Repository Structure

```
scheduler/
├── bookings-backend/          # Backend services (Scala/gRPC)
│   ├── bookings/              # Legacy bookings service
│   ├── bookingsV2/            # New bookings API (Loom Prime)
│   ├── calendar-3/            # Calendar server v3
│   ├── services-2/            # Services management
│   ├── resources/             # Resource management
│   ├── schedules/             # Schedule management
│   ├── common/                # Shared utilities
│   └── [50+ other modules]    # See full list below
├── statics/                   # Frontend modules (TypeScript/React)
│   ├── viewer/                # Customer-facing UI
│   ├── owner/                 # Business owner UI
│   └── modules/               # Shared frontend modules
├── serverless/                # Serverless functions
├── platform/                  # Platform-level services
├── scripts/                   # Utility scripts
├── docs/                      # Documentation
└── tools/                     # Build tools
```

---

## Technology Stack

### Backend
- **Language**: Scala 2.12/2.13.17
- **Framework**: Loom Prime (Wix internal framework)
- **RPC**: gRPC with Protocol Buffers
- **Build System**: Bazel
- **Database**: MySQL (via SDL - Simple Data Layer)
- **Event Streaming**: Greyhound (Kafka-based)
- **Feature Flags**: Petri (Wix A/B testing framework)
- **JSON**: Jackson (with Scala module)
- **Time**: Joda Time
- **Testing**: ScalaTest, contract tests

### Frontend
- **Language**: TypeScript
- **Frameworks**: React, Angular (legacy)
- **Build**: Yarn workspaces, Webpack
- **Package Manager**: Yarn 4.3.1

### Infrastructure
- **Containerization**: Docker
- **CI/CD**: Falcon CI
- **Monitoring**: New Relic, Sentry, Grafana
- **Configuration**: Hoopoe Config

---

## Core Modules

### Backend Services (`bookings-backend/`)

#### Bookings & Checkout
- **`bookingsV2/`** - New bookings API (primary)
  - `bookings-service/` - Main bookings writer service
  - `bookings-reader/` - Read-only bookings queries
  - `bookings-confirmator/` - Booking confirmation logic
  - `bookings-gateway/` - API gateway
  - `bookings-pricing/` - Pricing calculations
  - `bookings-locks/` - Concurrency control
  - `bookings-multi-bookings-handler/` - Multi-service bookings
  - `bookings-payments-handler/` - Payment processing
  - `bookings-orders-handler/` - Order integration
  - `bookings-schedules-handler/` - Schedule integration
  - `bookings-attendance/` - Attendance tracking
  - `bookings-bi-handler/` - Business intelligence
  - `bookings-rollout-manager/` - Feature rollouts
- **`bookings/`** - Legacy bookings service (being phased out)
- **`checkout/`** - Checkout flow
- **`waiting-list/`** - Waitlist management
- **`booking-fees/`** - Fee calculations
- **`booking-policies/`** - Booking policies
- **`bookings-reports/`** - Reporting
- **`bookings-contacts-labeler/`** - Contact labeling
- **`bookings-premium-marker/`** - Premium feature detection
- **`waiver/`** - Waiver management
- **`bookings-benefits-policy-provider/`** - Benefits integration
- **`bookings-forms-*`** - Form handling
- **`bookings-provisioner/`** - Provisioning logic
- **`bookings-settings/`** - Settings management
- **`bookings-automations-2/`** - Automation workflows
- **`bookings-calendar-event-validator/`** - Event validation
- **`bookings-course-template-cloner/`** - Course template cloning
- **`bookings-data-streams-service/`** - Event streaming
- **`bookings-migration/`** - Migration tools

#### Calendar & Scheduling
- **`calendar-3/`** - Calendar server v3 (primary)
  - `schedules/` - Schedule management
  - `events/` - Event management
  - `participations/` - Participant management
  - `event-views/` - Event view queries
- **`calendar-2/`** - Calendar server v2 (legacy)
- **`calendar/`** - Legacy calendar
- **`schedules/`** - Schedule management
- **`external-calendar-2/`** - External calendar sync
- **`external-calendar-google/`** - Google Calendar integration
- **`external-calendar-nylas/`** - Nylas integrations (iCloud, MS)
- **`external-calendar-spi/`** - External calendar SPI
- **`calendar-feed-2/`** - Calendar feed service
- **`calendar-vc-agent/`** - Video conferencing agent
- **`calendar-migration/`** - Calendar migration tools
- **`service-availability/`** - Service availability calculations
- **`resource-availability/`** - Resource availability calculations
- **`availability-calendar/`** - Availability calendar
- **`schedule-buffer-time/`** - Buffer time management
- **`time-slots-config/`** - Time slot configuration

#### Catalog & Services
- **`services-2/`** - Services management (primary)
- **`services/`** - Legacy services
- **`services-catalog/`** - Service catalog
- **`services-migrator/`** - Service migration
- **`service-options-and-variants/`** - Service variants
- **`catalog-events-mapper/`** - Catalog event mapping
- **`catalog-rollout-manager/`** - Catalog rollouts
- **`categories/`** - Service categories
- **`categories-proxy/`** - Categories proxy

#### Resources & Staff
- **`resource-2/`** - Resource management (primary)
- **`resources/`** - Legacy resources
- **`resource-types/`** - Resource type definitions
- **`resource-availability/`** - Resource availability
- **`resources-sorting-service/`** - Resource sorting
- **`staff-members/`** - Staff member management
- **`staff-member-settings/`** - Staff settings

#### Business & Configuration
- **`business/`** - Business entity management
- **`conference-accounts/`** - Video conferencing accounts
- **`online-meeting-integrations/`** - Meeting integrations
- **`notifications/`** - Notification handling
- **`translations/`** - Translation management
- **`premium-marker/`** - Premium feature detection
- **`edm/`** - Event-driven messaging
- **`site-search-indexer/`** - Search indexing
- **`ssr-cache-invalidator/`** - Cache invalidation

#### Common & Utilities
- **`common/`** - Shared utilities and helpers
  - Rate limiting
  - Common converters
  - Loom defaults
  - Context adapters
- **`common-test-app/`** - Test utilities
- **`boogle/`** - Testing/development service

### Frontend Modules (`statics/`)

#### Viewer (Customer-Facing)
- **`viewer/bookings-widget/`** - Multi-service booking widget
- **`viewer/bookings-widget-viewer/`** - Single-service widget
- **`viewer/modules/bookings-offering-page/`** - Service offering page
- **`viewer/angular/scheduler-client/`** - Legacy Angular client

#### Owner (Business-Facing)
- **`owner/`** - Business owner UI modules
  - Calendar management
  - Settings
  - Forms
  - Reports

#### Shared
- **`modules/`** - Shared frontend modules
- **`bookings-common/`** - Common frontend utilities

### Serverless Functions (`serverless/`)
- **`auto-pilot/`** - Automated operations
- **`auto-tc-caller/`** - Automated test calling
- **`locations-timezone-updater/`** - Timezone updates
- **`physicalqueue/`** - Queue management
- **`regress-macher/`** - Regression matching
- **`schedule-boundary-updater/`** - Schedule updates
- **`service-page/`** - Service page generation

### Platform Services (`platform/`)
- **`pricing-plan-benefits/`** - Pricing plan benefits service

---

## Architecture Patterns

### Service Architecture
- **Loom Prime Applications**: Modern services use `prime_app` macro
- **gRPC Services**: All services expose gRPC APIs
- **Protocol Buffers**: API definitions in `.proto` files
- **SDL (Simple Data Layer)**: Database abstraction layer
- **Automapper**: Proto ↔ SDL entity mapping

### Service Structure
Each service typically follows this structure:
```
service-name/
├── BUILD.bazel              # Bazel build file
├── proto/                   # Protocol buffer definitions
│   ├── com/wixpress/.../
│   └── docs/                # API documentation
├── src/                     # Source code
│   └── com/wixpress/.../
├── test/                    # Unit tests
├── it/                      # Integration tests
├── test-resources/          # Test resources
│   ├── confidence-infra/    # Confidence test configs
│   └── sdl/                 # SDL schemas
└── resources/               # Runtime resources
    └── *.sql               # Database schemas
```

### Key Patterns
1. **Domain-Driven Design**: Services organized by domain
2. **CQRS**: Separate read/write services (e.g., `bookings-service` vs `bookings-reader`)
3. **Event-Driven**: Greyhound for async communication
4. **Feature Flags**: Petri for gradual rollouts
5. **Contract Testing**: Services have contract test modules
6. **Versioning**: V2/V3 services indicate newer implementations

### Data Flow
```
Client → gRPC API → Service Layer → Domain Logic → SDL → MySQL
                                    ↓
                              Greyhound Events → Other Services
```

---

## Build System

### Bazel
- **Primary build tool**: Bazel
- **Workspace**: `WORKSPACE` (auto-generated, don't edit directly)
- **Build files**: `BUILD.bazel` in each module
- **Scala version**: 2.12/2.13.17
- **Compiler options**: Defined in `scheduler.bzl`

### Key Build Macros
- **`prime_app`**: Loom Prime application definition
  ```bazel
  prime_app(
      name = "service-name",
      artifact = "com.wixpress.bookings.service-name",
      rpcs = ["com.wixpress.bookings.ServiceName"],
      sdl = {...},
      proto_deps = [...],
  )
  ```
- **`sources()`**: Auto-discover Scala sources
- **`scala_library`**: Standard Scala library
- **`bootstrap_jar`**: Executable JAR packaging

### Dependencies
- **Internal**: `@server_infra//...`, `@wix_framework//...`
- **External**: `@io_grpc_grpc_api`, `@com_google_protobuf_...`
- **Cross-repo**: `@meta_site//...`, `@crm//...`, `@p13n//...`

### Common Build Commands
```bash
# Build a specific target
bazel build //bookings-backend/bookingsV2/bookings-service:bookings-service

# Run tests
bazel test //bookings-backend/bookingsV2/bookings-service/...

# Build with test output
bazel test ... --test_output=all

# Query dependencies
bazel query 'deps(//bookings-backend/bookingsV2/bookings-service:bookings-service)'
```

### Frontend Build
- **Yarn workspaces**: Monorepo package management
- **Scripts**: Defined in root `package.json`
  - `yarn build` - Build all packages
  - `yarn test` - Run tests
  - `yarn watch` - Watch mode

---

## Development Workflow

### Getting Started
1. **Clone the repository**
2. **Install dependencies**:
   - Bazel (via Wix tooling)
   - Yarn 4.3.1 (for frontend)
   - Java/Scala toolchain
3. **Set up pre-commit hooks**:
   ```bash
   ./scripts/setup-pre-commit.sh
   ```

### Code Organization
- **Backend**: Scala code in `src/main/scala/`
- **Protos**: API definitions in `proto/`
- **Tests**: Unit tests in `test/`, integration in `it/`
- **Resources**: Configs, SQL schemas in `resources/`

### Development Guidelines
1. **Follow Wix Server Guild standards**
2. **Use Loom Prime patterns** for new services
3. **Proto-first**: Define APIs in `.proto` files first
4. **SDL schemas**: Define in `test-resources/sdl/`
5. **Feature flags**: Use Petri specs (not feature toggles) for new functionality in `services-2/`
6. **Error handling**: Use framework error types
7. **Logging**: Use framework logger (not `println`)
8. **Testing**: Write contract tests for public APIs

### Adding a New Service
1. Create module directory: `bookings-backend/new-service/`
2. Define proto: `proto/com/wixpress/bookings/new_service.proto`
3. Create `BUILD.bazel` with `prime_app` macro
4. Implement service in `src/`
5. Add SDL schema in `test-resources/sdl/`
6. Write tests in `test/` and `it/`
7. Add to `CODEOWNERS` if needed

### Debugging
- **Reproduce**: `bazel test ... --test_output=all`
- **Logs**: Check framework-recommended logger output
- **Errors**: Look up APIs in MCP-S docs before guessing
- **Dependencies**: Verify Bazel labels in docs

---

## Key Concepts

### Loom Prime
Wix's application framework providing:
- Dependency injection
- Configuration management
- gRPC server setup
- SDL integration
- Resource management

### SDL (Simple Data Layer)
Database abstraction layer:
- Entity definitions in YAML
- Automatic CRUD operations
- Query builders
- Transaction management

### Protocol Buffers
- API contracts defined in `.proto` files
- Generated Scala code via ScalaPB
- Field accessors (not `getX()` methods)
- FieldMask for partial updates

### gRPC
- All services expose gRPC APIs
- REST APIs generated from gRPC (via Wix infrastructure)
- CallScope for request context
- Authentication via framework

### Greyhound
- Kafka-based event streaming
- Async service communication
- Domain events
- Consumer groups

### Petri
- Feature flag framework
- A/B testing
- Gradual rollouts
- Call-scoped experiments

### CallScope
- Request context propagation
- Identity/tenancy information
- Never bypass platform authorization
- Available in all service methods

### Automapper
- Proto ↔ SDL entity mapping
- Transformers/translators
- Used in service layer

---

## Team Ownership

### Primary Teams
- **`@wix-private/ot-bookings-platform`**: Root owner (all platforms teams)
- **`@wix-private/ot-bookings-core-code`**: Core bookings & catalog
- **`@wix-private/ot-bookings-calendar-platform-code`**: Calendar & scheduling

### Module Ownership
See `CODEOWNERS` for detailed ownership:
- **Bookings & Checkout**: `ot-bookings-core-code`
- **Calendar**: `ot-bookings-calendar-platform-code`
- **Catalog**: `ot-bookings-core-code`
- **Platform services**: Various teams (e.g., `ot-pricing-plans-backend`)

---

## Resources & Links

### Documentation
- **Main README**: `README.MD`
- **Repo Maps**: `docs/repo-map.MD`, `docs/owner-repo-map.md`
- **Calendar Server Guide**: `docs/calendar-server/calendar-server-on-call-guidelines.md`
- **Monitoring**: `docs/monitoring-bizmgr-team.MD`

### External Documentation
- **Calendar 3 API**: https://dev.wix.com/docs/rest/business-management/calendar/introduction
- **Calendar 3 Overview**: [Google Presentation](https://docs.google.com/presentation/d/1-0c516tiz6FttvlV4yQHNJQHJNILl8U9KjWcYO3oC00)
- **External Calendar API**: https://dev.wix.com/docs/rest/business-solutions/bookings/calendar/external-calendar-v2/introduction

### Monitoring
- **Calendar Server Grafana**: https://grafana.wixpress.com/d/R1Qrvl5Vz/calendar-index
- **New Relic**: Various apps (see `docs/monitoring-bizmgr-team.MD`)
- **Sentry**: Per-project (see `docs/repo-map.MD`)

### Slack Channels
- **`#bookings-calendar-platform`**: Calendar platform discussions
- **`#calendar-server-team-production`**: Production support
- **`#bazel-support`**: Bazel questions (for WORKSPACE changes)

### Development Tools
- **MCP-S Server**: For Wix API documentation lookup
  - `WixREADME` - Start here for any Wix-related task
  - `SearchWixRESTDocumentation` - REST API docs
  - `SearchWixSDKDocumentation` - SDK docs
  - `ReadFullDocsArticle` - Full article content

### Key Files to Know
- **`WORKSPACE`**: Bazel workspace (auto-generated, don't edit)
- **`BUILD.bazel`**: Module build definitions
- **`scheduler.bzl`**: Shared Scala compiler options
- **`CODEOWNERS`**: Team ownership
- **`package.json`**: Frontend dependencies
- **`pom.xml`**: Maven parent (legacy)

---

## Common Tasks

### Adding a New RPC Method
1. Add method to `.proto` file
2. Implement in service class
3. Update tests
4. Regenerate proto code: `bazel build //...:proto_scala`

### Adding a Database Table
1. Create SQL schema in `resources/*.sql`
2. Define SDL entity in `test-resources/sdl/`
3. Update service to use SDL
4. Run migrations

### Adding a New Dependency
1. Check MCP-S docs for correct Bazel label
2. Add to `deps` or `proto_deps` in `BUILD.bazel`
3. Verify import paths match docs

### Debugging Build Errors
1. Check exact error message
2. Look up API/library in MCP-S
3. Verify Bazel label matches docs
4. Check import paths are correct
5. Ensure proto/gen code is in sync

---

## Important Notes

### Migration Status
- **Bookings**: `bookingsV2/` is primary, `bookings/` is legacy
- **Calendar**: `calendar-3/` is primary, `calendar-2/` and `calendar/` are legacy
- **Services**: `services-2/` is primary, `services/` is legacy
- **Resources**: `resource-2/` is primary, `resources/` is legacy

### Feature Flags
- **`services-2/`**: Use feature toggles (not Petri specs) for functionality changes
- **Other modules**: Use Petri specs for gradual rollouts

### Code Style
- **Scala**: Follow `scheduler.bzl` compiler options (strict warnings)
- **Formatting**: Scalafmt (see `.scalafmt.conf`)
- **Type safety**: Prefer strong types, avoid `Any`

### Security
- **Never bypass**: Platform authorization (use CallScope)
- **Secrets**: Use databag configs, never hardcode
- **Auth**: Framework handles authentication

---

## Next Steps

1. **Explore a service**: Start with `bookings-backend/bookingsV2/bookings-service/`
2. **Read proto docs**: Check `proto/docs/` in any service
3. **Run tests**: `bazel test //bookings-backend/bookingsV2/bookings-service/...`
4. **Check monitoring**: Review Grafana dashboards
5. **Join Slack**: Get added to relevant channels
6. **Read API docs**: Explore dev.wix.com documentation

---

*Last updated: Based on current repository structure*
*For questions, reach out to `@wix-private/ot-bookings-platform`*

