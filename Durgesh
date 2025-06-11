# Engineering Metrics Service - Architecture and Design Documentation

## High Level Overview

The Engineering Metrics Service is designed to provide real-time engineering metrics by integrating with GitHub Actions and PagerDuty. The service calculates two critical metrics:
- Change Failure Rate (CFR) = Number of incidents / Number of deployments
- Mean Time to Recovery (MTTR) = Total recovery time of incidents / Number of incidents * 100% (expressed as a percentage)

### System Interaction Flow

1. **GitHub Actions Integration**:
   - When a deployment workflow completes in GitHub Actions
   - GitHub sends a webhook payload to `/webhooks/github`
   - System records deployment status and timestamp
   - Deployment data is used for CFR calculation

2. **PagerDuty Integration**:
   - When an incident is created/updated/resolved in PagerDuty
   - PagerDuty sends a webhook payload to `/webhooks/pagerduty`
   - System tracks incident lifecycle (creation to resolution)
   - Incident data is used for both MTTR and CFR calculations

3. **Metric Calculation**:
   - API endpoints accept time range parameters
   - System queries relevant deployments and incidents
   - Calculations performed in real-time
   - Results returned in standardized format

### System Interaction Diagram

```
+-------------------+         +-------------------+         +-------------------+
|                   |         |                   |         |                   |
|   GitHub Actions  +-------->+                   |         |                   |
|                   |  (API)  |                   |         |                   |
+-------------------+         |                   |         |                   |
                             |                   |         |                   |
+-------------------+         |                   |         |                   |
|                   |         |                   |         |                   |
|    PagerDuty      +-------->+   Spring Boot    +-------->+   SQL Database    |
|                   |  (API)  |     Service      |  (JPA)  |   (H2, in-memory) |
+-------------------+         |                   |         +-------------------+
                             |                   |
+-------------------+        |                   |
|                   |        |                   |
|  Webhooks (PD/GH) +-------->                   |
|                   |        |                   |
+-------------------+        +-------------------+

```

## System Architecture

```mermaid
classDiagram
    %% Controllers
    class WebhookController {
        +githubWebhook(GithubWebhookWrapper)
        +pagerDutyWebhook(PagerDutyWebhookWrapper)
    }
    class MetricsController {
        +getMTTR(from, to, preset)
        +getChangeFailureRate(from, to, preset)
    }

    %% Services
    class MetricService {
        <<interface>>
        +calculateMTTR(from, to)
        +calculateChangeFailureRate(from, to)
    }
    class MetricServiceImpl {
        +calculateMTTR(from, to)
        +calculateChangeFailureRate(from, to)
    }

    %% Repositories
    class DeploymentRepository {
        <<interface>>
        +findByRunId(Long)
        +findByTimestampBetween(from, to)
    }
    class IncidentRepository {
        <<interface>>
        +findByIncidentNumber(Long)
        +findByStartTimeBetweenAndEndTimeIsNotNull(from, to)
    }

    %% Entities
    class Deployment {
        -Long id
        -LocalDateTime timestamp
        -String status
        -Long runId
    }
    class Incident {
        -Long id
        -LocalDateTime startTime
        -LocalDateTime endTime
        -String status
        -Long incidentNumber
    }

    %% DTOs
    class MetricResponse {
        -String from
        -String to
        -double value
    }
    class GithubWebhookWrapper {
        -String action
        -WorkflowRun workflowRun
    }
    class PagerDutyWebhookWrapper {
        -PagerDutyEvent event
    }
    class TimePreset {
        <<enumeration>>
        LAST_1_DAY
        LAST_3_DAYS
        LAST_7_DAYS
        LAST_30_DAYS
        CUSTOM
    }

    %% Relationships
    WebhookController --> DeploymentRepository
    WebhookController --> IncidentRepository
    MetricsController --> MetricService
    MetricServiceImpl ..|> MetricService
    MetricServiceImpl --> DeploymentRepository
    MetricServiceImpl --> IncidentRepository
    DeploymentRepository --> Deployment
    IncidentRepository --> Incident
    MetricService --> MetricResponse
    WebhookController --> GithubWebhookWrapper
    WebhookController --> PagerDutyWebhookWrapper
    MetricsController --> TimePreset
```

## Architecture Evolution and Decision Making

### Data Collection Strategy Evolution

#### 1. Initial Synchronous API Approach
**Intent**: Direct API calls to external services when metrics are requested

```mermaid
sequenceDiagram
    participant User
    participant API
    participant GitHub
    participant PagerDuty
    participant DB
    
    User->>API: Request Metrics
    API->>GitHub: Fetch Deployments
    GitHub-->>API: Return Deployments
    API->>PagerDuty: Fetch Incidents
    PagerDuty-->>API: Return Incidents
    API->>DB: Store Data
    API->>API: Calculate Metrics
    API-->>User: Return Results
```

**Pros:**
- Simple implementation
- Data always up-to-date
- No background processes needed
- Lower system complexity

**Cons:**
- High latency for each request
- Risk of API rate limiting
- Poor user experience
- Not scalable for frequent requests
- Potential timeouts on large data sets

#### 2. Background Job Approach
**Intent**: Periodic scheduled jobs to fetch and store data from external services

```mermaid
sequenceDiagram
    participant Scheduler
    participant GitHub
    participant PagerDuty
    participant DB
    participant API
    participant User

    loop Every N minutes
        Scheduler->>GitHub: Fetch New Deployments
        GitHub-->>Scheduler: Return Deployments
        Scheduler->>DB: Store Deployments
        Scheduler->>PagerDuty: Fetch New Incidents
        PagerDuty-->>Scheduler: Return Incidents
        Scheduler->>DB: Store Incidents
    end

    User->>API: Request Metrics
    API->>DB: Query Stored Data
    DB-->>API: Return Data
    API->>API: Calculate Metrics
    API-->>User: Return Results
```

**Pros:**
- Reduced load on external APIs
- Better response time for metric requests
- Predictable API usage patterns
- Can implement smart batching
- Easier rate limit management

**Cons:**
- Data staleness between jobs
- Missed events if job fails
- Complex job scheduling logic
- Resource intensive for frequent runs
- Maintenance overhead for job infrastructure

#### 3. Webhook-Based Architecture
**Intent**: Real-time event-driven data collection through webhooks

```mermaid
sequenceDiagram
    participant GitHub
    participant PagerDuty
    participant Webhook API
    participant DB
    participant Metrics API
    participant User

    par GitHub to System
        GitHub->>Webhook API: Deployment Event
        Webhook API->>DB: Store Deployment
        Webhook API-->>GitHub: 200 OK
    and PagerDuty to System
        PagerDuty->>Webhook API: Incident Event
        Webhook API->>DB: Store Incident
        Webhook API-->>PagerDuty: 200 OK
    end

    User->>Metrics API: Request Metrics
    Metrics API->>DB: Query Fresh Data
    DB-->>Metrics API: Return Data
    Metrics API->>Metrics API: Calculate Metrics
    Metrics API-->>User: Return Results
```

**Pros:**
- Real-time data updates
- No polling overhead
- Reduced API calls
- Better resource utilization
- Immediate metric accuracy
- Event-driven architecture
- Lower operational costs

**Cons:**
- Requires webhook endpoint security
- More complex initial setup
- Must handle duplicate events
- Network reliability concerns
- Webhook configuration maintenance
- Need for event validation

### Current Implementation: Hybrid Approach

Our implemented solution combines both webhook-based real-time updates and scheduled background fetching to achieve optimal reliability and data consistency.

```mermaid
sequenceDiagram
    participant GH as GitHub Actions
    participant PD as PagerDuty
    participant WH as WebhookController
    participant FS as FetchScheduler
    participant DB as Database
    participant MS as MetricService
    participant User

    rect rgb(0, 0, 0)
        Note over GH,DB: Webhook Flow
        par GitHub to System
            GH->>WH: Deployment Webhook Event
            WH->>DB: Store/Update Deployment
            WH-->>GH: 200 OK
        and PagerDuty to System
            PD->>WH: Incident Webhook Event
            WH->>DB: Store/Update Incident
            WH-->>PD: 200 OK
        end
    end

    rect rgb(0, 0, 0)
        Note over FS,DB: Scheduled Background Fetch
        loop Every configuredInterval
            FS->>GH: Fetch Recent Deployments
            GH-->>FS: Return Deployments
            FS->>DB: Update Missing Data
            FS->>PD: Fetch Recent Incidents
            PD-->>FS: Return Incidents
            FS->>DB: Update Missing Data
        end
    end

    rect rgb(0, 0, 0)
        Note over User,MS: Metric Calculation
        User->>MS: Request Metrics
        MS->>DB: Query Data
        DB-->>MS: Return Data
        MS->>MS: Calculate Metrics
        MS-->>User: Return Results
    end
```

#### Components

1. **Webhook Processing**
   - `WebhookController` handles incoming webhooks from GitHub Actions and PagerDuty
   - Immediate processing of deployment completions and incident status changes
   - Deduplication using unique identifiers (runId for deployments, incidentNumber for incidents)
   - Thread-safe operations using database constraints

2. **Scheduled Background Fetching**
   - `FetchScheduler` runs periodic jobs to fetch missing data
   - Smart timestamp tracking to fetch only new records
   - Handles scenarios where webhooks might have been missed
   - Configurable schedule through application properties

#### Advantages Over Previous Approaches

1. **Reliability**
   - No single point of failure for data collection
   - Webhooks provide real-time updates
   - Scheduled fetching acts as a backup mechanism

2. **Data Consistency**
   - Deduplication handles duplicate webhook events
   - Background job fills any gaps in data
   - Consistent view of metrics even if webhooks fail

3. **Performance**
   - Low latency for metric calculations
   - Reduced API calls compared to pure polling
   - No request-time external API calls

### Scalability Considerations

#### Current Implementation
- Thread-safe operations using database constraints
- Configurable scheduled job intervals
- Efficient database queries with proper indexing

#### Future Scalability Enhancements

1. **Message Queue Integration**
```mermaid
flowchart LR
    subgraph External
        GH[GitHub Webhook]
        PD[PagerDuty Webhook]
    end
    
    subgraph Application
        LB[Load Balancer]
        Q[JMS Queue]
        P1[Processor 1]
        P2[Processor 2]
        DB[(Database)]
    end

    GH --> LB
    PD --> LB
    LB --> Q
    Q --> P1
    Q --> P2
    P1 --> DB
    P2 --> DB
```

- JMS queue for webhook processing
- Multiple processor instances
- Horizontal scaling capability
- Better handling of webhook bursts

2. **Caching Strategy**
```mermaid
flowchart LR
    subgraph Application
        API[API Layer]
        Cache[Redis Cache]
        Service[Metric Service]
        DB[(Database)]
    end

    API --> Cache
    Cache -. Cache Miss .-> Service
    Service --> DB
    DB --> Service
    Service --> Cache
```

- Redis cache for frequently accessed metrics
- Configurable time-based cache invalidation
- Cache warming during off-peak hours
- Reduced database load

3. **Database Optimization**
- Partitioning by date ranges
- Archival strategy for old data
- Read replicas for metric calculations
- Multiple database shards for different time ranges

### Implementation Benefits

1. **Operational**
   - Easy monitoring of data collection
   - Clear separation of concerns
   - Simplified troubleshooting
   - Configurable components

2. **Development**
   - Clean architecture
   - Easy to extend
   - Well-defined interfaces
   - Testable components

3. **Business**
   - Reliable metrics
   - Real-time updates when possible
   - Consistent historical data
   - Scalable solution

## Key Components

### 1. Data Collection System

#### Webhook Integration
- Dedicated endpoints `/webhooks/github` and `/webhooks/pagerduty`
- Real-time data ingestion for immediate metric updates
- Deduplication logic using unique identifiers
- Status updates for existing records

### 2. Persistence Layer
- In-memory H2 database for development simplicity
- JPA entities: `Deployment` and `Incident`
- Unique constraints on external identifiers
- Repository layer with specialized query methods

### 3. External Integration Layer
- `RestTemplate` based HTTP clients
- Dedicated client classes: `GithubClient` and `PagerDutyClient`
- Externalized configuration for API tokens and endpoints
- Error handling with fallback to empty lists

### 4. Metric Calculation Engine
- `MetricService` for core business logic
- On-demand calculation of MTTR and CFR
- Time window-based metric computation
- Comprehensive error handling and logging

### 5. REST API Layer
- `MetricsController` exposing metric endpoints
- Support for custom time ranges
- Parameter validation
- Global exception handling

### 6. Logging System
- Configurable through `logback-spring.xml`
- File and console appenders
- Configurable log directory
- DEBUG level logging by default

## Design Principles

1. **Single Responsibility**
   - Each component handles one specific aspect
   - Clear separation between data collection and metric calculation

2. **Open/Closed Principle**
   - Extensible metric calculation system
   - Easy to add new metrics without changing existing code

3. **Interface Segregation**
   - Separate interfaces for different types of metrics
   - Clean API boundaries


## Monitoring and Reliability
 **Logging**
   - Structured logging for all operations
   - Audit trail for metric calculations
   - Error tracking and alerting

## Future Enhancements

1. **Scalability**
   - Migration to persistent database
   - Caching layer for frequent queries
   - Load balancing for webhook endpoints

2. **Features**
   - Additional engineering metrics
   - Metric visualization dashboard
   - Historical trend analysis
   - Custom metric definitions
