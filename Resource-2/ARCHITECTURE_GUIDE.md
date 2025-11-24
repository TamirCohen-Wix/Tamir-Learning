# Architecture Guide: Domain, WQL, Nile, Loom, Platformized Query, and Contract

## Concepts Overview

### 1. **Contract** (API Layer)
**What it is:**
- The **public API contract** defined in Protocol Buffers (`.proto` files)
- Represents the **external interface** that clients use
- Example: `Resource` proto message in `resource.proto`

**Characteristics:**
- Uses **camelCase** field names (e.g., `locationOptions`, `singleResource`)
- Defined in proto files
- Used in gRPC API endpoints
- Clients send/receive Contract objects

**Example:**
```protobuf
message Resource {
  string id = 1;
  SingleResource singleResource = 2;
  LocationOptions locationOptions = 3;
}
```

---

### 2. **Domain** (Business Logic Layer)
**What it is:**
- The **internal business domain model** used by your application
- Represents entities as they exist in your business logic
- Example: `ResourceDomain` case class

**Characteristics:**
- Uses **camelCase** field names (e.g., `locationOptions`, `compositionDetails`)
- Scala case classes
- Used internally for business logic
- Mapped from/to Contract via translators

**Example:**
```scala
case class ResourceDomain(
  id: UUID,
  compositionDetails: CompositionDetailsDomain,
  locationOptions: Option[LocationOptionsDomain]
)
```

**Why separate from Contract?**
- Allows internal representation to differ from API
- Enables evolution of internal model without breaking API
- Provides abstraction layer

---

### 3. **WQL (Wix Query Language)**
**What it is:**
- **Field path format** used by Nile Search for filtering and querying
- Defined in proto files using `wql` options
- Uses **snake_case** with specific prefixes

**Characteristics:**
- Uses **snake_case** field paths (e.g., `single_resource.location_options`)
- Defined in proto `wql` sections
- Used by Nile Search index
- Different from Contract/Domain field names

**Example:**
```protobuf
wql: {
  pattern: {
    field: "single_resource.location_options.available_in_all_locations"
    field: "single_resource.location_options.specific_location_options.business_locations.location_id"
  }
}
```

**Why different format?**
- Search index uses different naming convention
- Allows querying nested fields with dot notation
- Optimized for search/filter operations

---

### 4. **Nile (Nile Search)**
**What it is:**
- **Full-text search infrastructure** integrated with SDL
- Provides search, filtering, and aggregation capabilities
- Uses Elasticsearch under the hood

**Characteristics:**
- Automatically indexes data from MySQL via domain events
- Provides search API (`sdl.search.search()`)
- Generates `constraint_filter` automatically for authorization
- Expects WQL field paths (snake_case)

**Responsibilities:**
- Full-text search
- Flexible filtering
- Aggregations
- Automatic indexing from domain events
- Authorization constraint filtering

---

### 5. **Loom and Loom Prime**
**What they are:**
- **Deployment/runtime environments** for Wix services
- Different infrastructure setups for running applications

**Loom:**
- Traditional/legacy deployment environment
- Different configuration requirements
- Older infrastructure setup

**Loom Prime:**
- Modern deployment environment
- Newer infrastructure
- Different configuration syntax
- Preferred for new services

**In your code:**
- SDL configuration differs between Loom and Loom Prime
- BUILD.bazel files have different setups
- Same functionality, different deployment targets

---

### 6. **Platformized Query**
**What it is:**
- A **query format** that goes through field path translation
- Uses `fromPlatformizedQuery()` method in SDL
- Automatically translates Contract field paths → Domain → WQL

**Characteristics:**
- Used by `sdl.query.fromPlatformizedQuery()`
- Applies `ContractToDomainFieldPathTranslator`
- Converts API paths to correct format for database/search
- **Only used for Query endpoint, NOT Search endpoint**

**Example:**
```scala
sdl.query
  .fromPlatformizedQuery(query)  // Translates field paths
  .execute()
```

**Why it exists:**
- Handles field path translation automatically
- Converts between Contract, Domain, and WQL formats
- Ensures queries use correct field paths

---

## System Architecture & Data Flow

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                            │
│  (External API Consumers - Web, Mobile, etc.)                   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ gRPC API Calls
                             │ (Contract format)
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API SERVICE LAYER                           │
│  ResourcesService (gRPC Service)                                │
│  - Receives: Contract (Resource proto)                          │
│  - Returns: Contract (Resource proto)                           │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ Translation
                             │ Contract ↔ Domain
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    BUSINESS LOGIC LAYER                          │
│  - ResourceProtoToDomainMappers (Contract → Domain)             │
│  - ResourceDomainToProtoMappers (Domain → Contract)             │
│  - Business logic operations                                    │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ Domain objects
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      SDL HANDLER LAYER                           │
│  SdlHandler                                                     │
│  - sdl.query.fromPlatformizedQuery()  (Query)                  │
│  - sdl.search.search()              (Search)                    │
│  - sdl.insert(), sdl.update(), etc.                             │
└────────────────────────────┬────────────────────────────────────┘
                             │
                ┌────────────┴────────────┐
                │                         │
                ▼                         ▼
┌──────────────────────────┐  ┌──────────────────────────┐
│      SDL FRAMEWORK       │  │   NILE SEARCH            │
│  (Simple Data Layer)     │  │   Infrastructure        │
│                          │  │                          │
│  - MySQL Database        │  │  - Elasticsearch Index   │
│  - Domain Events         │  │  - WQL Field Paths       │
│  - Field Translation     │  │  - constraint_filter     │
│  - Query Operations      │  │  - Search Operations     │
└──────────────────────────┘  └──────────────────────────┘
```

---

### Data Flow: Query Endpoint (✅ Works Correctly)

```
1. CLIENT REQUEST
   └─> QueryResourcesRequest
       └─> filter: { "type": "abc123" }  (Contract format, camelCase)

2. API SERVICE LAYER
   └─> ResourcesService.queryResources()
       └─> Converts: QueryResourcesRequest → CursorQuery

3. SDL HANDLER
   └─> SdlHandler.queryResources()
       └─> sdl.query.fromPlatformizedQuery(query)
           │
           ├─> ContractToDomainFieldPathTranslator
           │   └─> Translates: Contract paths → Domain paths → WQL paths
           │
           └─> Executes query with translated field paths

4. SDL FRAMEWORK
   └─> Executes query on MySQL
       └─> Returns: ResourceDomain objects

5. RESPONSE
   └─> Domain → Contract translation
       └─> QueryResourcesResponse (Contract format)
```

**Key Point:** `fromPlatformizedQuery()` handles all field path translation automatically.

---

### Data Flow: Search Endpoint (❌ constraint_filter Issue)

```
1. CLIENT REQUEST
   └─> SearchResourcesRequest
       └─> filter: { "type": "abc123" }  (Contract format, camelCase)

2. API SERVICE LAYER
   └─> ResourcesService.searchResources()
       └─> Converts: SearchResourcesRequest → CursorSearch

3. SDL HANDLER
   └─> SdlHandler.searchResources()
       └─> sdl.search.search(search).execute()
           │
           ├─> NO fromPlatformizedQuery() ❌
           ├─> NO field path translation ❌
           │
           └─> Passes search directly to Nile Search

4. NILE SEARCH INFRASTRUCTURE
   └─> Receives search request
       │
       ├─> Analyzes CallScope (authorization context)
       │   └─> Detects: User token with location restrictions
       │
       └─> Generates constraint_filter
           └─> Uses Domain field paths (camelCase) ❌
           └─> Should use WQL paths (snake_case) ✅
           │
           └─> constraint_filter: {
                 "locationOptions.specificLocationOptions.businessLocations.locationId": {...}
               }
               │
               └─> ❌ WRONG FORMAT - Should be:
                   "single_resource.location_options.specific_location_options.business_locations.location_id"

5. NILE SEARCH BACKEND
   └─> Receives search + constraint_filter
       └─> ❌ FAILS: Field path mismatch
           └─> Expects WQL paths (snake_case)
           └─> Gets Domain paths (camelCase)
```

**Key Problem:** constraint_filter bypasses translation because it's generated by Nile Search infrastructure, not your code.

---

### Field Path Translation Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    FIELD PATH TRANSLATION                        │
└─────────────────────────────────────────────────────────────────┘

Contract Format (API)
  └─> locationOptions.specificLocationOptions.businessLocations.locationId
      │
      │ ResourceTranslator
      │ (Contract → Domain)
      ▼
Domain Format (Internal)
  └─> locationOptions.specificLocationOptions.businessLocations.locationId
      │
      │ ContractToDomainFieldPathTranslator
      │ (via fromPlatformizedQuery)
      │
      │ OR
      │
      │ ResourceTranslator (for nested fields)
      ▼
WQL Format (Search Index)
  └─> single_resource.location_options.specific_location_options.business_locations.location_id
```

**Translation happens:**
- ✅ **Query endpoint:** Via `fromPlatformizedQuery()` → Full translation
- ❌ **Search endpoint:** Direct call → No translation
- ❌ **constraint_filter:** Generated by Nile → No translation

---

### Component Communication

```
┌──────────────┐
│   Contract   │  ← Public API interface
│  (Proto)     │     - camelCase
│              │     - External format
└──────┬───────┘
       │
       │ ResourceProtoToDomainMappers
       │ (Contract → Domain)
       ▼
┌──────────────┐
│   Domain     │  ← Internal business model
│  (Scala)     │     - camelCase
│              │     - Business logic
└──────┬───────┘
       │
       │ ContractToDomainFieldPathTranslator
       │ (via fromPlatformizedQuery)
       │
       ├─────────────────┬─────────────────┐
       │                 │                 │
       ▼                 ▼                 ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│     WQL      │  │     SDL      │  │    Nile      │
│  (snake_case)│  │   (MySQL)    │  │  (Elastic)   │
│              │  │              │  │              │
│ Search Index │  │ Database     │  │ Search       │
│ Field Paths  │  │ Operations   │  │ Operations   │
└──────────────┘  └──────────────┘  └──────────────┘
```

---

## Key Takeaways

1. **Contract = API Layer**
   - External interface (proto)
   - camelCase field names
   - What clients see

2. **Domain = Business Layer**
   - Internal representation (Scala)
   - camelCase field names
   - Business logic operates here

3. **WQL = Search Index Format**
   - snake_case field paths
   - Used by Nile Search
   - Defined in proto `wql` sections

4. **Platformized Query = Translation Layer**
   - Automatically translates field paths
   - Used by Query endpoint
   - **NOT used by Search endpoint**

5. **Nile Search = Search Infrastructure**
   - Full-text search
   - Generates constraint_filter automatically
   - Expects WQL field paths

6. **Loom/Loom Prime = Deployment Environments**
   - Different runtime environments
   - Different configuration syntax
   - Same functionality

---

## The Problem: constraint_filter Translation Gap

```
Query Endpoint Flow:
  Contract → fromPlatformizedQuery() → Translator → WQL ✅

Search Endpoint Flow:
  Contract → sdl.search.search() → (no translation) → Nile Search
  + constraint_filter (generated by Nile) → Domain paths → WQL ❌
  
  Result: Field path mismatch!
```

**Solution needed:** Make constraint_filter use WQL field paths, or translate it before sending to Nile Search.

