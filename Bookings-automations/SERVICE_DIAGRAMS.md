# Bookings Automations Service v2 - Visual Diagrams

## Service Architecture Overview

```mermaid
graph TB
    subgraph "External Domain Events"
        BE[Bookings Service v2<br/>Domain Events]
        CE[Calendar Service v3<br/>Domain Events]
        AE[Attendance Service v2<br/>Domain Events]
        FE[Booking Fees<br/>Domain Events]
        PE[Pricing Plans<br/>Credit Notifications]
    end

    subgraph "Bookings Automations Service v2"
        AS[AutomationsService<br/>Main Handler]
        TP[TriggerProviderService<br/>SPI Service]
        
        subgraph "Trigger System"
            T1[SessionsBookedTrigger]
            T2[BookingCanceledTrigger]
            T3[SessionUpdatedTrigger]
            T4[CheckInTriggers]
            T5[SessionStart/EndTriggers]
            T6[Other Triggers...]
        end
        
        TE[TriggerEnricher<br/>Data Fetching]
        
        subgraph "Reproducers"
            R1[CheckInEventReproducer]
            R2[SessionParticipationReproducer]
            D[Delayers]
        end
        
        subgraph "Repositories"
            FR[FileRepository<br/>ICS Files]
            SR[SentTriggerRepository]
        end
    end
    
    subgraph "External RPC Services"
        BS[Bookings Service]
        CS[Calendar Service]
        SS[Services Service]
        OS[Orders & Payments]
        FS[Forms Service]
        SP[Site Properties]
        MS[Meta Site]
    end
    
    subgraph "Wix Automations Platform"
        ESB[ESB Config Resolver]
        AP[Automations Platform]
    end

    BE --> AS
    CE --> AS
    AE --> AS
    FE --> AS
    PE --> AS
    
    AS --> R1
    AS --> R2
    AS --> D
    AS --> T1
    AS --> T2
    AS --> T3
    AS --> T4
    AS --> T5
    AS --> T6
    
    T1 --> TE
    T2 --> TE
    T3 --> TE
    T4 --> TE
    T5 --> TE
    T6 --> TE
    
    TE --> BS
    TE --> CS
    TE --> SS
    TE --> OS
    TE --> FS
    TE --> SP
    TE --> MS
    
    T1 --> ESB
    T2 --> ESB
    T3 --> ESB
    T4 --> ESB
    T5 --> ESB
    T6 --> ESB
    
    ESB --> AP
    
    TP --> T1
    TP --> T2
    TP --> T3
    
    TE --> FR
    T1 --> SR
```

## Booking Confirmation Flow

```mermaid
sequenceDiagram
    participant BS as Bookings Service
    participant AS as AutomationsService
    participant D as Delayer
    participant ST as SessionsBookedTrigger
    participant TE as TriggerEnricher
    participant ESB as ESB Config Resolver
    participant AP as Automations Platform

    BS->>AS: BookingChanged (status=CONFIRMED)
    AS->>ST: filter(event)
    ST-->>AS: true (matches)
    AS->>D: delay(bookingId, bookingChanged)
    D->>D: Publish DelayedExecutionEvent<br/>(8s Kafka delay)
    
    Note over D: Wait 8 seconds
    
    D->>AS: handleDelayedExecutionEvent()
    AS->>ST: triggerImpl(bookingChanged)
    
    par Parallel Data Fetching
        ST->>TE: getOrderDetailsByBookingId()
        ST->>TE: getBusinessProperties()
        ST->>TE: getServiceDetails()
        ST->>TE: getConferencingAndCreatedIcs()
        ST->>TE: getBookingPolicySnapshot()
        ST->>TE: getFormattedFormSubmission()
        ST->>TE: getFormFieldsV2()
        ST->>TE: getPaymentLinkUrl()
        ST->>TE: getAnonymousLinks()
        ST->>TE: getIntakeFormDataIfPremium()
    end
    
    TE-->>ST: Enriched Data
    
    ST->>ST: Build Payload Map
    ST->>ST: Apply Field Overrides (if configured)
    ST->>ST: Check Automations Config
    
    ST->>ESB: reportEvent(triggerKey, payload, idempotency)
    ESB->>AP: Trigger Automation
    AP-->>ESB: activationIds
    ESB-->>ST: ReportResult
```

## Check-In Event Flow

```mermaid
sequenceDiagram
    participant AS as Attendance Service
    participant ASvc as AutomationsService
    participant R as CheckInEventReproducer
    participant BC as BookingsClient
    participant M as Messaging
    participant ACI as AnyCheckInTrigger
    participant NCI as NthCheckInTrigger
    participant LCI as LatestCheckInTrigger
    participant NS as NoShowTrigger
    participant ESB as ESB Config Resolver

    AS->>ASvc: AttendanceMarkedAsNotAttended
    ASvc->>R: reproduce(event)
    R->>BC: getBooking(attendance.bookingId)
    BC-->>R: Booking
    R->>R: Extract contactId
    R->>M: publishToCheckInEvent(CheckInEvent)
    
    Note over M: Internal Kafka Topic<br/>bookings-automations-2-internal-check-in-event
    
    par Multiple Subscribers
        M->>ACI: handleCheckInEventForAnyCheckIn()
        M->>NCI: handleCheckInEventForNthCheckIn()
        M->>LCI: handleCheckInEventForLatestCheckIn()
        M->>NS: handleCheckInEventForNoShow()
    end
    
    ACI->>ESB: reportEvent(any_check_in)
    NCI->>ESB: reportEvent(nth_check_in)
    LCI->>ESB: reportEvent(latest_check_in)
    NS->>ESB: reportEvent(no_show)
```

## Trigger System Architecture

```mermaid
classDiagram
    class Trigger {
        <<trait>>
        +triggerKey: String
        +entityIdType: String
        +trigger(event: E): Future[Unit]
        +report(entityId, payload): Future[Unit]
        +cancel(entityId): Future[Unit]
    }
    
    class TriggerWithFilter {
        <<trait>>
        +filter(event: E): Boolean
    }
    
    class TriggerWithDynamicSchema {
        <<trait>>
        +getDynamicSchema(): Future[Struct]
    }
    
    class TriggerWithRefresh {
        <<trait>>
        +refreshPayload(): Future[Struct]
    }
    
    class TriggerWithValidation {
        <<trait>>
        +validate(): Future[Boolean]
    }
    
    class SessionsBookedTrigger {
        +triggerKey: "wix_bookings-sessions_booked"
        +filter(BookingChanged): Boolean
        +triggerImpl(): Future[Unit]
    }
    
    class BookingCanceledTrigger {
        +triggerKey: "wix_bookings-booking_canceled"
    }
    
    class AnyCheckInTrigger {
        +triggerKey: "wix_bookings-any_check_in"
    }
    
    Trigger <|-- TriggerWithFilter
    Trigger <|-- TriggerWithDynamicSchema
    Trigger <|-- TriggerWithRefresh
    Trigger <|-- TriggerWithValidation
    TriggerWithFilter <|-- SessionsBookedTrigger
    TriggerWithFilter <|-- BookingCanceledTrigger
    Trigger <|-- AnyCheckInTrigger
```

## Data Enrichment Flow

```mermaid
graph LR
    subgraph "Trigger Execution"
        T[Trigger.triggerImpl]
    end
    
    subgraph "TriggerEnricher - Parallel Fetching"
        direction TB
        TE[TriggerEnricher]
        TE1[getOrderDetailsByBookingId]
        TE2[getBusinessProperties]
        TE3[getServiceDetails]
        TE4[getConferencingAndCreatedIcs]
        TE5[getBookingPolicySnapshot]
        TE6[getFormattedFormSubmission]
        TE7[getFormFieldsV2]
        TE8[getPaymentLinkUrl]
        TE9[getAnonymousLinks]
        TE10[getIntakeFormDataIfPremium]
    end
    
    subgraph "External Services"
        direction TB
        BS[Bookings Service]
        OS[Orders Service]
        SS[Services Service]
        CS[Calendar Service]
        FS[Forms Service]
        SP[Site Properties]
        MS[Meta Site]
        PL[Payment Links]
        IF[Intake Forms]
    end
    
    subgraph "Payload Building"
        PB[Build Payload Map]
        FO[Apply Field Overrides]
        AC[Check Automations Config]
    end
    
    subgraph "Reporting"
        R[reportEvent to ESB]
    end
    
    T --> TE
    TE --> TE1
    TE --> TE2
    TE --> TE3
    TE --> TE4
    TE --> TE5
    TE --> TE6
    TE --> TE7
    TE --> TE8
    TE --> TE9
    TE --> TE10
    
    TE1 --> OS
    TE2 --> SP
    TE3 --> SS
    TE4 --> CS
    TE5 --> BS
    TE6 --> FS
    TE7 --> FS
    TE8 --> PL
    TE9 --> BS
    TE10 --> IF
    
    TE1 -.-> PB
    TE2 -.-> PB
    TE3 -.-> PB
    TE4 -.-> PB
    TE5 -.-> PB
    TE6 -.-> PB
    TE7 -.-> PB
    TE8 -.-> PB
    TE9 -.-> PB
    TE10 -.-> PB
    
    PB --> FO
    FO --> AC
    AC --> R
```

## SPI Integration Flow

```mermaid
sequenceDiagram
    participant App as Third-Party App
    participant SPI as BookingAutomationsConfiguration SPI
    participant AC as AutomationsConfigClient
    participant T as Trigger
    participant ESB as ESB Config Resolver

    App->>SPI: Configure BookingsAutomationsConfig
    Note over SPI: Configuration stored
    
    T->>AC: getAutomationsConfig(appId, triggerKey)
    AC->>SPI: Fetch Configuration
    SPI-->>AC: BookingsAutomationsConfig
    
    alt Configuration Override
        AC-->>T: Config with overriddenTrigger
        T->>T: Use overriddenTrigger key
    else Field Override
        AC-->>T: Config with fieldKeyMap
        T->>T: Apply field key transformations
    else Disabled
        AC-->>T: Config with disableReport=true
        T->>T: Skip reporting
    end
    
    T->>ESB: reportEvent(triggerKey, payload)
```

## Delayed Execution Mechanism

```mermaid
stateDiagram-v2
    [*] --> BookingChangedEvent: Domain Event Received
    
    BookingChangedEvent --> FilterCheck: Check if trigger matches
    
    FilterCheck --> NoMatch: Filter returns false
    FilterCheck --> Match: Filter returns true
    
    NoMatch --> [*]: Skip processing
    
    Match --> Delay: Delayer.delay()
    
    Delay --> Serialize: Convert to JSON
    Serialize --> Publish: Publish DelayedExecutionEvent
    
    Publish --> KafkaDelay: Kafka delay (8 seconds)
    
    KafkaDelay --> Consume: Consume delayed event
    Consume --> Deserialize: Parse JSON payload
    Deserialize --> Callback: Execute registered callback
    
    Callback --> TriggerExecution: triggerImpl()
    TriggerExecution --> Enrichment: Fetch data
    Enrichment --> Report: Report to ESB
    Report --> [*]: Complete
```

## ICS File Generation Flow

```mermaid
graph TB
    subgraph "Trigger Execution"
        T[Trigger needs ICS file]
    end
    
    T --> TE[TriggerEnricher.getConferencingAndCreatedIcs]
    
    TE --> Check{Booking Type?}
    
    Check -->|Slot| GetEvent[Get Event from Calendar]
    Check -->|Schedule| GetSchedule[Get Schedule from Calendar]
    Check -->|Other| None[No ICS file]
    
    GetEvent --> CreateICS[IcsCreator.create]
    GetSchedule --> CreateICS
    
    CreateICS --> IcsEvent[IcsEvent object]
    IcsEvent --> IcsFile[IcsFile with data]
    
    IcsFile --> Store[FileRepository.storeFile]
    Store --> DataStore[(DataStore<br/>30-day TTL)]
    
    Store --> FileId[Return IcsFileId]
    FileId --> Payload[Add to trigger payload]
    
    Payload --> Email[Email automation<br/>includes ICS attachment]
    
    Email --> Download[DownloadIcsFile endpoint]
    Download --> DataStore
    DataStore --> Response[RawHttpResponse<br/>with ICS data]
```

## Error Handling & Retry Strategy

```mermaid
graph TB
    Start[Trigger Execution] --> Try[Attempt Operation]
    
    Try --> Success{Success?}
    
    Success -->|Yes| Complete[Complete]
    
    Success -->|No| CheckError{Error Type?}
    
    CheckError -->|Missing Tenant ID| Ignore[Log & Ignore<br/>Automations not installed]
    CheckError -->|Meta-site Not Found| Ignore[Log & Ignore<br/>Site deleted]
    CheckError -->|Transient Error| Retry[Retry with Backoff]
    CheckError -->|Permanent Error| Fail[Fail & Log]
    
    Retry --> RetrySchedule[Retry Schedule:<br/>30s, 30s, 1m, 1m,<br/>5m, 10m, 30m, 1h, 3h]
    
    RetrySchedule --> MaxRetries{Max Retries?}
    MaxRetries -->|No| Try
    MaxRetries -->|Yes| Fail
    
    Ignore --> Complete
    Fail --> Complete
```

## Component Dependencies

```mermaid
graph TB
    subgraph "Core Service"
        AS[AutomationsService]
        TP[TriggerProviderService]
    end
    
    subgraph "Trigger Layer"
        T[Triggers]
        TB[Base Trigger Classes]
    end
    
    subgraph "Enrichment Layer"
        TE[TriggerEnricher]
        Clients[RPC Clients]
    end
    
    subgraph "Event Processing"
        R[Reproducers]
        D[Delayers]
    end
    
    subgraph "Storage"
        FR[FileRepository]
        SR[SentTriggerRepository]
    end
    
    subgraph "External"
        ESB[ESB Config Resolver]
        Services[33 RPC Services]
    end
    
    AS --> T
    AS --> R
    AS --> D
    T --> TB
    T --> TE
    T --> FR
    T --> SR
    TE --> Clients
    Clients --> Services
    T --> ESB
    TP --> T
    R --> AS
    D --> AS
```

