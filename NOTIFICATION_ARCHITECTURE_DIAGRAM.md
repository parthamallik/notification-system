# Notification System - Architecture Diagrams

## System Architecture Overview

```mermaid
graph TB
    subgraph "Ecommerce Services"
        OS[Order Service]
        US[User Service]
        PS[Product Service]
    end
    
    subgraph "Event Bus"
        EB[Kafka/RabbitMQ]
    end
    
    subgraph "Notification API Service"
        NC[Notification Controller]
        NS[Notification Service]
        SS[Scheduler Service]
        TS[Template Service]
        SEG[Segment Service]
    end
    
    subgraph "Queue System"
        CQ[Critical Queue]
        HQ[High Queue]
        MQ[Medium Queue]
        LQ[Low Queue]
        SQ[Scheduled Queue]
    end
    
    subgraph "Worker Service Instances"
        W1[Worker 1]
        W2[Worker 2]
        WN[Worker N]
    end
    
    subgraph "Channel Abstraction"
        CH[Channel Factory]
        EC[Email Channel]
        SC[SMS Channel]
        PC[Push Channel]
        IC[In-App Channel]
    end
    
    subgraph "External Providers"
        SG[SendGrid]
        TW[Twilio]
        FC[Firebase FCM]
        AP[Apple APNs]
    end
    
    subgraph "Database"
        PG[(PostgreSQL)]
        RD[(Redis Cache)]
    end
    
    OS -->|Events| EB
    US -->|Events| EB
    PS -->|Events| EB
    
    EB -->|Notification Events| NC
    NC --> NS
    NS --> TS
    NS --> SEG
    NS --> SS
    SS --> CQ
    SS --> HQ
    SS --> MQ
    SS --> LQ
    SS --> SQ
    
    NS --> PG
    TS --> PG
    SEG --> PG
    SEG --> RD
    
    CQ --> W1
    HQ --> W1
    MQ --> W2
    LQ --> W2
    SQ --> WN
    
    W1 --> CH
    W2 --> CH
    WN --> CH
    
    CH --> EC
    CH --> SC
    CH --> PC
    CH --> IC
    
    EC --> SG
    SC --> TW
    PC --> FC
    PC --> AP
    IC --> PG
    
    W1 --> PG
    W2 --> PG
    WN --> PG
    
    SG -->|Webhooks| W1
    TW -->|Webhooks| W1
    FC -->|Webhooks| W1
```

## Notification Flow - Immediate Notification

```mermaid
sequenceDiagram
    participant ES as Ecommerce Service
    participant EB as Event Bus
    participant API as Notification API
    participant DB as Database
    participant Q as Queue
    participant W as Worker
    participant CH as Channel
    participant P as Provider
    participant WH as Webhook Handler
    
    ES->>EB: Publish OrderCreated Event
    EB->>API: OrderCreated Event
    API->>API: Evaluate Segment
    API->>DB: Check Idempotency
    API->>DB: Create Notification Record
    API->>DB: Create Delivery Records
    API->>API: Render Template
    API->>Q: Enqueue Job (High Priority)
    API->>DB: Update Status: Queued
    Q->>W: Dequeue Job
    W->>DB: Check Idempotency (again)
    W->>CH: Get Channel (Email)
    CH->>P: Send Email (SendGrid)
    P-->>CH: MessageId
    CH-->>W: Delivery Result
    W->>DB: Update Status: Sent
    P->>WH: Webhook: Delivered
    WH->>DB: Update Status: Delivered
```

## Notification Flow - Scheduled Notification

```mermaid
sequenceDiagram
    participant MS as Marketing Service
    participant API as Notification API
    participant SCH as Scheduler
    participant DB as Database
    participant Q as Queue
    participant W as Worker
    participant CH as Channel
    
    MS->>API: Create Scheduled Notification
    Note over MS,API: scheduledAt: 2024-12-25T10:00:00Z
    API->>API: Evaluate Segment (10K users)
    API->>DB: Create Notification Record
    API->>DB: Create 10K Delivery Records
    API->>SCH: Schedule Notification
    SCH->>DB: Store Scheduled Job
    SCH->>Q: Enqueue with Delay
    Note over Q: Job available at scheduled time
    Q->>W: Dequeue Job (at scheduled time)
    W->>CH: Send to 10K users (batched)
    CH-->>W: Results
    W->>DB: Update Statuses
```

## Priority Queue Processing

```mermaid
graph LR
    subgraph "Priority Queues"
        CQ[Critical Queue<br/>Priority 0]
        HQ[High Queue<br/>Priority 1]
        MQ[Medium Queue<br/>Priority 2]
        LQ[Low Queue<br/>Priority 3]
    end
    
    subgraph "Worker Processing"
        W[Worker Service]
        P1[Process Critical]
        P2[Process High]
        P3[Process Medium]
        P4[Process Low]
    end
    
    CQ -->|Always Process First| P1
    HQ -->|Process After Critical| P2
    MQ -->|Process After High| P3
    LQ -->|Process After Medium| P4
    
    P1 --> W
    P2 --> W
    P3 --> W
    P4 --> W
```

## Channel Abstraction Pattern

```mermaid
classDiagram
    class INotificationChannel {
        <<interface>>
        +SendAsync(NotificationRequest) DeliveryResult
        +CanHandle(ChannelType) bool
        +GetCapabilities() ChannelCapabilities
    }
    
    class BaseNotificationChannel {
        <<abstract>>
        #ExecuteWithRetryAsync()
        #HandleError()
    }
    
    class EmailChannel {
        -IEmailProvider provider
        +SendAsync()
    }
    
    class SmsChannel {
        -ISmsProvider provider
        +SendAsync()
    }
    
    class PushChannel {
        -IPushProvider provider
        +SendAsync()
    }
    
    class InAppChannel {
        -IWebSocketService service
        +SendAsync()
    }
    
    class ChannelFactory {
        +CreateChannel(ChannelType) INotificationChannel
    }
    
    INotificationChannel <|.. BaseNotificationChannel
    BaseNotificationChannel <|-- EmailChannel
    BaseNotificationChannel <|-- SmsChannel
    BaseNotificationChannel <|-- PushChannel
    BaseNotificationChannel <|-- InAppChannel
    ChannelFactory --> INotificationChannel
```

## Retry & Failure Handling

```mermaid
flowchart TD
    Start[Worker Receives Job] --> Check{Check<br/>Idempotency}
    Check -->|Already Processed| Skip[Skip - Return]
    Check -->|Not Processed| Send[Send to Channel]
    Send --> Result{Result}
    Result -->|Success| UpdateSuccess[Update Status: Sent]
    Result -->|Failure| CheckRetry{Retry Count<br/>< Max?}
    CheckRetry -->|Yes| Increment[Increment Retry Count]
    Increment --> Backoff[Calculate Backoff Delay]
    Backoff --> Requeue[Re-queue with Delay]
    Requeue --> Wait[Wait for Backoff]
    Wait --> Send
    CheckRetry -->|No| DLQ[Move to Dead Letter Queue]
    DLQ --> Alert[Alert Operations]
    UpdateSuccess --> End[End]
    Alert --> End
    Skip --> End
```

## Scalability Architecture

```mermaid
graph TB
    subgraph "Load Balancer"
        LB[Application Load Balancer]
    end
    
    subgraph "API Service Instances"
        API1[API Instance 1]
        API2[API Instance 2]
        APIN[API Instance N]
    end
    
    subgraph "Queue System"
        Q[Redis Queue Cluster]
    end
    
    subgraph "Worker Service Instances"
        W1[Worker 1]
        W2[Worker 2]
        W3[Worker 3]
        WN[Worker N]
    end
    
    subgraph "Database Cluster"
        PG1[(PostgreSQL Primary)]
        PG2[(PostgreSQL Replica 1)]
        PG3[(PostgreSQL Replica 2)]
    end
    
    subgraph "Cache Cluster"
        R1[(Redis Cache 1)]
        R2[(Redis Cache 2)]
    end
    
    LB --> API1
    LB --> API2
    LB --> APIN
    
    API1 --> Q
    API2 --> Q
    APIN --> Q
    
    API1 --> PG1
    API2 --> PG1
    APIN --> PG1
    
    API1 --> R1
    API2 --> R2
    APIN --> R1
    
    Q --> W1
    Q --> W2
    Q --> W3
    Q --> WN
    
    W1 --> PG1
    W2 --> PG2
    W3 --> PG3
    WN --> PG2
    
    PG1 -.->|Replication| PG2
    PG1 -.->|Replication| PG3
```

## Data Model Relationships

```mermaid
erDiagram
    NOTIFICATIONS ||--o{ NOTIFICATION_DELIVERIES : has
    NOTIFICATION_DELIVERIES ||--o{ DELIVERY_STATUS_HISTORY : has
    NOTIFICATIONS }o--|| NOTIFICATION_TEMPLATES : uses
    USERS ||--o{ NOTIFICATION_DELIVERIES : receives
    USERS ||--o{ USER_SEGMENTS : belongs_to
    NOTIFICATIONS ||--o| DEAD_LETTER_QUEUE : may_move_to
    
    NOTIFICATIONS {
        uuid id PK
        uuid idempotency_key UK
        string notification_type
        string channel_type
        string priority
        string template_id FK
        jsonb metadata
        jsonb segment_definition
        timestamp scheduled_at
        string status
    }
    
    NOTIFICATION_DELIVERIES {
        uuid id PK
        uuid notification_id FK
        uuid recipient_id FK
        string channel_type
        string status
        string provider_message_id
        timestamp sent_at
        timestamp delivered_at
        int retry_count
    }
    
    DELIVERY_STATUS_HISTORY {
        uuid id PK
        uuid delivery_id FK
        string status
        string provider_message_id
        jsonb metadata
        timestamp created_at
    }
    
    NOTIFICATION_TEMPLATES {
        string id PK
        string name
        string channel_type
        text subject_template
        text body_template
        text html_body_template
        jsonb variables
    }
    
    USER_SEGMENTS {
        uuid user_id PK
        string segment_id PK
        string segment_name
        timestamp added_at
    }
    
    DEAD_LETTER_QUEUE {
        uuid id PK
        uuid notification_id FK
        uuid delivery_id FK
        jsonb job_data
        text failure_reason
        jsonb error_details
        int retry_count
    }
```

## Monitoring & Observability Stack

```mermaid
graph TB
    subgraph "Application Services"
        API[API Service]
        W[Worker Service]
    end
    
    subgraph "Metrics Collection"
        PM[Prometheus]
        API -->|Metrics| PM
        W -->|Metrics| PM
    end
    
    subgraph "Logging"
        SL[Serilog]
        ELK[ELK Stack]
        API -->|Logs| SL
        W -->|Logs| SL
        SL --> ELK
    end
    
    subgraph "Tracing"
        OT[OpenTelemetry]
        JA[Jaeger]
        API -->|Traces| OT
        W -->|Traces| OT
        OT --> JA
    end
    
    subgraph "Visualization"
        GF[Grafana]
        KI[Kibana]
        PM --> GF
        ELK --> KI
        JA --> GF
    end
    
    subgraph "Alerting"
        AM[Alertmanager]
        PD[PagerDuty]
        PM --> AM
        AM --> PD
    end
```

---

## How to View These Diagrams

### Option 1: GitHub/GitLab
- These Mermaid diagrams will render automatically in GitHub/GitLab markdown viewers

### Option 2: VS Code
- Install "Markdown Preview Mermaid Support" extension
- Open markdown file and use preview

### Option 3: Online Tools
- Copy diagram code to [Mermaid Live Editor](https://mermaid.live/)
- Export as PNG/SVG

### Option 4: CLI Tools
- Install Mermaid CLI: `npm install -g @mermaid-js/mermaid-cli`
- Convert to image: `mmdc -i diagram.md -o diagram.png`

---

*These diagrams provide visual representation of the notification system architecture. They complement the detailed design document.*

