# System Design & Data Flow Diagram

## Complete System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT APPLICATIONS                             │
│                    (Web, Mobile, Third-party integrations)                  │
└────────────────────────────────────┬────────────────────────────────────────┘
                                      │
                                      │ gRPC API Calls
                                      │ Protocol: HTTP/2
                                      │ Format: Protocol Buffers
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         API SERVICE LAYER (gRPC)                            │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │              ResourcesService (gRPC Service)                        │   │
│  │                                                                       │   │
│  │  Endpoints:                                                          │   │
│  │  • searchResources()  → Search endpoint                              │   │
│  │  • queryResources()   → Query endpoint                               │   │
│  │  • createResource()  → Create operation                             │   │
│  │  • updateResource()  → Update operation                             │   │
│  │  • deleteResource()  → Delete operation                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│  Input/Output: Contract format (Resource proto)                               │
│  Field Format: camelCase (e.g., locationOptions, singleResource)            │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │
                    ┌────────────────┴────────────────┐
                    │                                 │
                    │ Translation Layer               │
                    │                                 │
        ┌───────────▼──────────┐      ┌──────────────▼──────────┐
        │  Contract → Domain   │      │  Domain → Contract      │
        │  Mappers             │      │  Mappers                │
        │                      │      │                         │
        │ ResourceProtoTo      │      │ ResourceDomainTo        │
        │ DomainMappers        │      │ ProtoMappers            │
        └───────────┬──────────┘      └──────────────┬──────────┘
                    │                                 │
                    └──────────────┬──────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        BUSINESS LOGIC LAYER                                 │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Domain Models (Scala)                           │   │
│  │                                                                       │   │
│  │  • ResourceDomain                                                    │   │
│  │  • LocationOptionsDomain                                            │   │
│  │  • CompositionDetailsDomain                                         │   │
│  │                                                                       │   │
│  │  Field Format: camelCase (e.g., locationOptions, compositionDetails)│   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Business Logic                                   │   │
│  │  • Validation                                                        │   │
│  │  • Business rules                                                    │   │
│  │  • Domain operations                                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │
                                     │ Domain objects
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SDL HANDLER LAYER                                   │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      SdlHandler                                     │   │
│  │                                                                       │   │
│  │  Operations:                                                         │   │
│  │  ┌───────────────────────────────────────────────────────────────┐  │   │
│  │  │ queryResources()                                             │  │   │
│  │  │  └─> sdl.query.fromPlatformizedQuery(query)                  │  │   │
│  │  │      └─> ✅ Uses translator                                   │  │   │
│  │  │          └─> ContractToDomainFieldPathTranslator             │  │   │
│  │  │                                                               │  │   │
│  │  │ searchResources()                                            │  │   │
│  │  │  └─> sdl.search.search(search).execute()                     │  │   │
│  │  │      └─> ❌ NO translator                                    │  │   │
│  │  │          └─> Direct to Nile Search                           │  │   │
│  │  │                                                               │  │   │
│  │  │ insert(), update(), delete(), etc.                           │  │   │
│  │  └───────────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │
                    ┌────────────────┴────────────────┐
                    │                                 │
                    ▼                                 ▼
┌──────────────────────────────┐  ┌──────────────────────────────────────┐
│      SDL FRAMEWORK           │  │      NILE SEARCH                      │
│  (Simple Data Layer)         │  │      Infrastructure                   │
│                              │  │                                       │
│  ┌────────────────────────┐  │  │  ┌────────────────────────────────┐  │
│  │   MySQL Database       │  │  │  │  Elasticsearch Index           │  │
│  │                        │  │  │  │                                │  │
│  │  • CRUD operations     │  │  │  │  • Full-text search           │  │
│  │  • Transactions        │  │  │  │  • Filtering                   │  │
│  │  • Optimistic locking  │  │  │  │  • Aggregations               │  │
│  └────────────────────────┘  │  │  │  • WQL field paths (snake_case)│  │
│                              │  │  └────────────────────────────────┘  │
│  ┌────────────────────────┐  │  │                                       │
│  │   Domain Events        │  │  │  ┌────────────────────────────────┐  │
│  │                        │  │  │  │  constraint_filter Generator   │  │
│  │  • Created             │  │  │  │                                │  │
│  │  • Updated             │  │  │  │  • Analyzes CallScope          │  │
│  │  • Deleted             │  │  │  │  • Detects location restrictions│  │
│  │                        │  │  │  │  • Generates filter            │  │
│  │  ────────────────────> │──┼──┼─>│  • Uses Domain paths ❌        │  │
│  │  Auto-index to Nile    │  │  │  │  • Should use WQL paths ✅      │  │
│  └────────────────────────┘  │  │  └────────────────────────────────┘  │
│                              │  │                                       │
│  ┌────────────────────────┐  │  │                                       │
│  │   Field Translation    │  │  │                                       │
│  │                        │  │  │                                       │
│  │  ContractToDomain     │  │  │                                       │
│  │  FieldPathTranslator   │  │  │                                       │
│  │                        │  │  │                                       │
│  │  • Contract → Domain   │  │  │                                       │
│  │  • Domain → WQL        │  │  │                                       │
│  └────────────────────────┘  │  │                                       │
└──────────────────────────────┘  └──────────────────────────────────────┘
```

---

## Data Flow: Query Endpoint (✅ Working)

```
┌─────────────┐
│   CLIENT    │
│  Request    │
└──────┬──────┘
       │
       │ QueryResourcesRequest
       │ filter: { "type": "abc" }  (Contract, camelCase)
       ▼
┌─────────────────────────────────────┐
│  ResourcesService.queryResources()  │
└──────┬──────────────────────────────┘
       │
       │ Converts to CursorQuery
       ▼
┌─────────────────────────────────────┐
│  SdlHandler.queryResources()       │
│                                     │
│  sdl.query                          │
│    .fromPlatformizedQuery(query)    │
│    .execute()                       │
└──────┬──────────────────────────────┘
       │
       │ ✅ Translation happens here
       │
       ▼
┌─────────────────────────────────────┐
│  ContractToDomainFieldPathTranslator│
│                                     │
│  Input:  "type" (Contract)          │
│  Output: "type" (Domain)            │
│         "type" (WQL)                │
└──────┬──────────────────────────────┘
       │
       │ Translated query
       ▼
┌─────────────────────────────────────┐
│  SDL Framework                      │
│  Executes query on MySQL            │
│  Returns: ResourceDomain[]          │
└──────┬──────────────────────────────┘
       │
       │ Domain → Contract
       ▼
┌─────────────────────────────────────┐
│  Response                           │
│  QueryResourcesResponse             │
│  resources: Resource[] (Contract)   │
└─────────────────────────────────────┘
```

---

## Data Flow: Search Endpoint (❌ constraint_filter Issue)

```
┌─────────────┐
│   CLIENT    │
│  Request    │
└──────┬──────┘
       │
       │ SearchResourcesRequest
       │ filter: { "type": "abc" }  (Contract, camelCase)
       ▼
┌─────────────────────────────────────┐
│  ResourcesService.searchResources() │
└──────┬──────────────────────────────┘
       │
       │ Converts to CursorSearch
       ▼
┌─────────────────────────────────────┐
│  SdlHandler.searchResources()       │
│                                     │
│  sdl.search.search(search)          │
│    .execute()                       │
│                                     │
│  ❌ NO fromPlatformizedQuery()      │
│  ❌ NO translation                  │
└──────┬──────────────────────────────┘
       │
       │ Search request (no translation)
       ▼
┌─────────────────────────────────────┐
│  Nile Search Infrastructure         │
│                                     │
│  1. Receives search request         │
│  2. Analyzes CallScope              │
│     └─> Detects: User token         │
│     └─> Detects: Location restrictions
│  3. Generates constraint_filter     │
│     └─> Uses Domain paths ❌        │
│         "locationOptions.specific..."│
│                                     │
│  ❌ PROBLEM: Should use WQL paths   │
│     "single_resource.location_..."  │
└──────┬──────────────────────────────┘
       │
       │ Search + constraint_filter
       │ (wrong field paths)
       ▼
┌─────────────────────────────────────┐
│  Nile Search Backend                │
│  (Elasticsearch)                    │
│                                     │
│  ❌ FAILS: Field path mismatch      │
│     Expected: WQL paths (snake_case)│
│     Got: Domain paths (camelCase)   │
└─────────────────────────────────────┘
```

---

## Field Path Translation Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    FIELD PATH TRANSLATION                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  CONTRACT FORMAT (API Layer)                                    │
│                                                                   │
│  locationOptions.specificLocationOptions.businessLocations       │
│  .locationId                                                     │
│                                                                   │
│  Format: camelCase                                               │
│  Used by: API clients                                            │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     │ ResourceProtoToDomainMappers
                     │ (Contract → Domain)
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│  DOMAIN FORMAT (Business Layer)                                 │
│                                                                   │
│  locationOptions.specificLocationOptions.businessLocations       │
│  .locationId                                                     │
│                                                                   │
│  Format: camelCase                                               │
│  Used by: Business logic                                         │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     │ ContractToDomainFieldPathTranslator
                     │ (via fromPlatformizedQuery)
                     │
                     │ OR
                     │
                     │ ResourceTranslator (for nested fields)
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│  WQL FORMAT (Search Index)                                      │
│                                                                   │
│  single_resource.location_options.specific_location_options      │
│  .business_locations.location_id                                 │
│                                                                   │
│  Format: snake_case with prefix                                  │
│  Used by: Nile Search (Elasticsearch)                           │
└─────────────────────────────────────────────────────────────────┘

Translation Status:
  ✅ Query endpoint:  Contract → Domain → WQL (via fromPlatformizedQuery)
  ❌ Search endpoint: Contract → (no translation) → Nile Search
  ❌ constraint_filter: Domain → (no translation) → Nile Search ❌
```

---

## Component Communication Matrix

| Component | Input Format | Output Format | Translation | Used By |
|-----------|-------------|---------------|-------------|---------|
| **Contract** | camelCase (proto) | camelCase (proto) | N/A | API clients |
| **Domain** | camelCase (Scala) | camelCase (Scala) | Via mappers | Business logic |
| **WQL** | snake_case | snake_case | Via translator | Nile Search |
| **Platformized Query** | Contract | WQL | ✅ Automatic | Query endpoint |
| **Search** | Contract | WQL | ❌ None | Search endpoint |
| **constraint_filter** | Domain | WQL | ❌ None | Nile Search |

---

## Deployment Environments

```
┌─────────────────────────────────────────────────────────────────┐
│                    DEPLOYMENT LAYER                             │
└─────────────────────────────────────────────────────────────────┘

┌──────────────────────────┐      ┌──────────────────────────┐
│      LOOM                │      │    LOOM PRIME            │
│  (Legacy Environment)    │      │  (Modern Environment)    │
│                          │      │                          │
│  • Traditional setup     │      │  • Modern infrastructure │
│  • Different config      │      │  • Different config      │
│  • Same functionality    │      │  • Same functionality    │
│                          │      │                          │
│  BUILD.bazel:            │      │  BUILD.bazel:            │
│  sdl_ninja_domain        │      │  sdl_scala_domain        │
└──────────────────────────┘      └──────────────────────────┘
           │                                │
           └────────────┬───────────────────┘
                        │
                        ▼
            ┌───────────────────────┐
            │   SDL Framework       │
            │   (Same for both)     │
            └───────────────────────┘
```

---

## Summary: The Translation Gap

```
┌─────────────────────────────────────────────────────────────────┐
│                    TRANSLATION STATUS                           │
└─────────────────────────────────────────────────────────────────┘

✅ QUERY ENDPOINT:
   Contract → fromPlatformizedQuery() → Translator → WQL → MySQL
   
❌ SEARCH ENDPOINT:
   Contract → sdl.search.search() → (no translation) → Nile Search
   
❌ CONSTRAINT_FILTER:
   Domain → (generated by Nile) → (no translation) → Nile Search
   
   Result: Field path mismatch error!
```

**The Fix Needed:**
Make constraint_filter go through translation: Domain → WQL before sending to Nile Search.

