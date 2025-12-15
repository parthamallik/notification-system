# Robust Notification System Design for Ecommerce Application

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [Requirements](#requirements)
3. [Non-Functional Requirements (NFRs)](#non-functional-requirements-nfrs)
4. [System Architecture](#system-architecture)
5. [Technology Stack & Tool Choices](#technology-stack--tool-choices)
6. [Design Patterns & Principles](#design-patterns--principles)
7. [Component Design](#component-design)
8. [Data Models](#data-models)
9. [Event Flow & Processing](#event-flow--processing)
10. [Scalability & Performance](#scalability--performance)
11. [Resilience & Reliability](#resilience--reliability)
12. [Security & Compliance](#security--compliance)
13. [Monitoring & Observability](#monitoring--observability)
14. [Implementation Phases](#implementation-phases)
15. [Trade-offs & Design Decisions](#trade-offs--design-decisions)

---

## Executive Summary

This document outlines the design for a robust, scalable notification system for an ecommerce application. The system is designed to handle high-volume notifications across multiple channels (email, SMS, push, in-app) with guaranteed delivery, idempotency, priority handling, and event-driven scheduling.

**Key Capabilities:**
- Multi-channel notification delivery (extensible abstraction)
- User segmentation and targeting
- Priority-based processing
- Event-driven scheduling (no polling)
- Horizontal scalability
- Delivery status tracking
- Idempotent operations
- Resilient failure handling

---

## Requirements

### Functional Requirements

1. **Segmented User Notifications**
   - Support complex user segmentation (demographics, behavior, purchase history, preferences)
   - Dynamic segment evaluation
   - Real-time segment updates

2. **Notification Medium Abstraction**
   - Pluggable notification channels (Email, SMS, Push, In-App, Webhook)
   - Easy addition of new channels without code changes
   - Channel-specific configuration and templates

3. **Priority Assignment**
   - Multiple priority levels (Critical, High, Medium, Low)
   - Priority-based queue processing
   - Priority-aware rate limiting

4. **Resiliency & Idempotency**
   - At-least-once delivery guarantee
   - Idempotent notification sending (deduplication)
   - Retry mechanisms with exponential backoff
   - Dead letter queue for failed notifications

5. **Delivery Status Tracking**
   - Real-time status updates (Pending, Queued, Sending, Delivered, Failed, Bounced)
   - Delivery receipts from providers
   - Status history and audit trail

6. **Horizontal Scalability**
   - Stateless services for easy scaling
   - Distributed queue processing
   - Database read replicas
   - Caching layer

7. **Event-Driven Scheduling**
   - Async event-based notification scheduling
   - No database polling
   - Support for immediate and scheduled notifications
   - Timezone-aware scheduling

---

## Non-Functional Requirements (NFRs)

### Performance
- **Throughput**: Support 100K+ notifications/hour per worker instance
- **Latency**: 
  - Critical priority: < 1 second from event to queue
  - High priority: < 5 seconds
  - Medium/Low: < 30 seconds
- **Processing Time**: < 500ms per notification (excluding external provider latency)
- **Queue Depth**: Handle up to 1M pending notifications

### Scalability
- **Horizontal Scaling**: Linear scaling with worker instances
- **Database**: Support 10M+ notification records with efficient queries
- **Concurrent Users**: Support 100K+ active users
- **Peak Load**: Handle 10x normal load during flash sales/promotions

### Availability
- **Uptime**: 99.9% availability (8.76 hours downtime/year)
- **Graceful Degradation**: Continue operating with partial failures
- **Multi-Region**: Support multi-region deployment for disaster recovery

### Reliability
- **Delivery Guarantee**: At-least-once delivery (idempotency handles duplicates)
- **Data Durability**: All notification records persisted before processing
- **Failure Recovery**: Automatic recovery from transient failures
- **Message Loss**: Zero message loss tolerance

### Security
- **Data Privacy**: PII encryption at rest and in transit
- **Access Control**: Role-based access to notification management
- **Rate Limiting**: Per-user and per-channel rate limits
- **Audit Logging**: Complete audit trail for compliance

### Maintainability
- **Code Quality**: Modular, testable, well-documented code
- **Configuration**: Externalized configuration (no hardcoded values)
- **Monitoring**: Comprehensive metrics, logs, and alerts
- **Documentation**: API documentation, runbooks, architecture diagrams

### Cost Efficiency
- **Resource Optimization**: Efficient queue processing, connection pooling
- **Provider Costs**: Smart routing to minimize external provider costs
- **Storage**: Efficient data retention and archival policies

---

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Ecommerce Application                          │
│  (Order Service, User Service, Product Service, etc.)                   │
└────────────────────────────-──┬─────────────────────────────────────────┘
                                │
                                │ Events (OrderCreated, PaymentReceived, etc.)
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      Event Bus / Message Broker                         │
│  (Apache Kafka / RabbitMQ / AWS SQS / Azure Service Bus)                │
└──────────────────────────-────┬─────────────────────────────────────────┘
                                │
                                │ Notification Events
                                ▼
┌────────────────────────────────────────────────────────────────────────┐
│                    Notification API Service                            │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Notification Controller                                         │  │
│  │  - Create Notification Request                                   │  │
│  │  - Query Notification Status                                     │  │
│  │  - Manage Templates                                              │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Notification Service                                            │  │
│  │  - Segment Evaluation                                            │  │
│  │  - Template Rendering                                            │  │
│  │  - Priority Assignment                                           │  │
│  │  - Idempotency Check                                             │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Scheduler Service                                               │  │
│  │  - Schedule Immediate Notifications                              │  │
│  │  - Schedule Future Notifications                                 │  │
│  │  - Timezone Handling                                             │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└───────────────────────────-───┬────────────────────────────────────────┘
                                │
                                │ Queue Jobs
                                ▼
┌───────────────────────────────────────────────────────────────────────┐
│                    Distributed Queue System                           │
│  (Redis + Bull / RabbitMQ / AWS SQS / Azure Service Bus)              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │
│  │ Critical Q   │  │ High Q       │  │ Medium Q     │                 │
│  └──────────────┘  └──────────────┘  └──────────────┘                 │
│  ┌──────────────┐  ┌──────────────┐                                   │
│  │ Low Q        │  │ Scheduled Q  │                                   │
│  └──────────────┘  └──────────────┘                                   │
└─────────────────────────-─────┬───────────────────────────────────────┘
                                │
                                │ Consume Jobs
                                ▼
┌────────────────────────────────────────────────────────────────────────┐
│                    Notification Worker Service                         │
│  (Multiple Instances for Horizontal Scaling)                           │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Notification Processor                                          │  │
│  │  - Job Consumption                                               │  │
│  │  - Retry Logic                                                   │  │
│  │  - Dead Letter Queue                                             │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Channel Abstraction Layer                                       │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐          │  |
│  │  │ Email    │  │ SMS      │  │ Push     │  │ In-App   │          │  |
│  │  │ Provider │  │ Provider │  │ Provider │  │ Provider │          │  |
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘          │  |
│  └──────────────────────────────────────────────────────────────────┘  │
└─────────────────────────-─────┬────────────────────────────────────────┘
                                │
                                │ Delivery Status Updates
                                ▼
┌────────────────────────────────────────────────────────────────────────┐
│                    PostgreSQL Database                                 │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  - notifications (requests, status, metadata)                    │  │
│  │  - notification_deliveries (per-recipient tracking)              │  │
│  │  - notification_templates                                        │  │
│  │  - user_segments                                                 │  │
│  │  - delivery_status_history                                       │  │
│  │  - dead_letter_queue                                             │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
                                │
                                │ Webhooks / Callbacks
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    External Notification Providers                      │
│  (SendGrid, Twilio, Firebase, OneSignal, etc.)                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Component Breakdown

#### 1. **Notification API Service**
- **Purpose**: Entry point for notification requests
- **Responsibilities**:
  - Accept notification creation requests
  - Evaluate user segments
  - Render templates
  - Enqueue notifications
  - Query notification status
  - Manage templates and configurations

#### 2. **Event Bus / Message Broker**
- **Purpose**: Decouple notification system from ecommerce services
- **Responsibilities**:
  - Receive events from ecommerce services
  - Route events to notification service
  - Support event replay for debugging

#### 3. **Scheduler Service**
- **Purpose**: Handle scheduled notifications without polling
- **Responsibilities**:
  - Schedule immediate notifications (enqueue immediately)
  - Schedule future notifications (enqueue with delay)
  - Handle timezone conversions
  - Support recurring notifications

#### 4. **Distributed Queue System**
- **Purpose**: Reliable job queue with priority support
- **Responsibilities**:
  - Priority-based queues
  - Delayed job scheduling
  - Job persistence
  - Retry management

#### 5. **Notification Worker Service**
- **Purpose**: Process notification jobs
- **Responsibilities**:
  - Consume jobs from queues
  - Route to appropriate channel provider
  - Update delivery status
  - Handle retries and failures
  - Process dead letter queue

#### 6. **Channel Abstraction Layer**
- **Purpose**: Pluggable notification channels
- **Responsibilities**:
  - Abstract channel-specific logic
  - Handle channel-specific errors
  - Support channel-specific rate limiting
  - Provide unified interface

#### 7. **Database**
- **Purpose**: Persistent storage for notifications and metadata
- **Responsibilities**:
  - Store notification requests
  - Track delivery status
  - Store templates
  - Maintain audit logs

---

## Technology Stack & Tool Choices

### Core Technologies

| Component | Technology Choice | Reasoning |
|-----------|------------------|-----------|
| **API Framework** | Node.js / Express / TypeScript | Fast development, excellent async model, large ecosystem, type safety with TypeScript |
| **Worker Framework** | Node.js / TypeScript | Excellent for background processing, great for queue consumers, non-blocking I/O |
| **Database** | PostgreSQL 15+ | ACID compliance, JSONB support for flexible metadata, excellent performance |
| **Queue System** | Redis + Bull / RabbitMQ | Redis: Fast, in-memory, supports delayed jobs. Bull: Excellent Node.js queue library. RabbitMQ: Enterprise-grade, better for complex routing |
| **Event Bus** | Apache Kafka / RabbitMQ / AWS EventBridge | Kafka: High throughput, event replay. RabbitMQ: Simpler, good for most cases |
| **Caching** | Redis | Fast, supports complex data structures, already in use |
| **Database Access** | Prisma / TypeORM / Knex.js | Prisma: Type-safe, migrations. TypeORM: Rich ORM features. Knex.js: Lightweight query builder |

### Notification Providers

| Channel | Provider Options | Reasoning |
|---------|-----------------|-----------|
| **Email** | SendGrid, AWS SES, Mailgun | High deliverability, good APIs, webhook support |
| **SMS** | Twilio, AWS SNS, Vonage | Reliable, global coverage, delivery receipts |
| **Push (Mobile)** | Firebase Cloud Messaging, Apple Push Notification Service | Native mobile support, high delivery rates |
| **Push (Web)** | Web Push API, OneSignal | Browser-native, no app required |
| **In-App** | Custom WebSocket / Socket.io | Real-time, low latency, user engagement |

### Infrastructure & DevOps

| Component | Technology Choice | Reasoning |
|-----------|------------------|-----------|
| **Containerization** | Docker | Consistent environments, easy deployment |
| **Orchestration** | Kubernetes / Docker Swarm | Auto-scaling, service discovery, health checks |
| **Service Discovery** | Consul / Kubernetes DNS | Dynamic service registration |
| **Configuration Management** | Consul / etcd / Environment Variables | Centralized config, dynamic updates |
| **Secrets Management** | HashiCorp Vault / AWS Secrets Manager | Secure credential storage |

### Monitoring & Observability

| Component | Technology Choice | Reasoning |
|-----------|------------------|-----------|
| **Logging** | Winston / Pino + ELK Stack | Structured logging, centralized aggregation, excellent Node.js support |
| **Metrics** | Prometheus + Grafana | Industry standard, rich visualization |
| **Tracing** | OpenTelemetry / Jaeger | Distributed tracing, performance analysis |
| **Alerting** | Prometheus Alertmanager / PagerDuty | Real-time alerts, on-call management |
| **APM** | Application Insights / New Relic | Application performance monitoring |

### Testing

| Type | Technology Choice | Reasoning |
|------|------------------|-----------|
| **Unit Testing** | Jest / Mocha + Chai | Node.js standard, excellent tooling, great TypeScript support |
| **Integration Testing** | Testcontainers | Real database/Redis in tests |
| **Load Testing** | k6 / JMeter | Realistic load simulation |
| **Chaos Engineering** | Chaos Monkey / Gremlin | Resilience testing |

---

## Design Patterns & Principles

### 1. **Strategy Pattern** (Channel Abstraction)
```typescript
interface INotificationChannel {
    sendAsync(request: NotificationRequest): Promise<DeliveryResult>;
    canHandle(channelType: NotificationChannelType): boolean;
    getCapabilities(): ChannelCapabilities;
}

class EmailChannel implements INotificationChannel { }
class SmsChannel implements INotificationChannel { }
class PushChannel implements INotificationChannel { }
```

**Benefits**: Easy to add new channels, testable, follows Open/Closed Principle

### 2. **Factory Pattern** (Channel Creation)
```typescript
interface INotificationChannelFactory {
    createChannel(type: NotificationChannelType): INotificationChannel;
}
```

### 3. **Repository Pattern** (Data Access)
```typescript
interface INotificationRepository {
    createAsync(notification: Notification): Promise<Notification>;
    getByIdAsync(id: string): Promise<Notification | null>;
    updateStatusAsync(id: string, status: NotificationStatus): Promise<void>;
}
```

### 4. **Unit of Work Pattern** (Transaction Management)
```typescript
interface IUnitOfWork {
    notifications: INotificationRepository;
    saveChangesAsync(): Promise<number>;
    beginTransactionAsync(): Promise<void>;
    commitTransactionAsync(): Promise<void>;
}
```

### 5. **Observer Pattern** (Event-Driven Architecture)
- Services publish events
- Notification service subscribes to relevant events
- Decoupled, scalable

### 6. **Circuit Breaker Pattern** (Resilience)
```typescript
class CircuitBreaker {
    private failureCount = 0;
    private state: CircuitState = CircuitState.Closed;
    
    async executeAsync<T>(operation: () => Promise<T>): Promise<T> {
        if (this.state === CircuitState.Open) {
            throw new CircuitBreakerOpenException();
        }
            
        try {
            const result = await operation();
            this.onSuccess();
            return result;
        } catch (ex) {
            this.onFailure();
            throw ex;
        }
    }
}
```

### 7. **Retry Pattern with Exponential Backoff**
```typescript
async function retryAsync<T>(
    operation: () => Promise<T>,
    maxRetries: number = 3,
    initialDelay: number = 1000
): Promise<T> {
    for (let i = 0; i < maxRetries; i++) {
        try {
            return await operation();
        } catch (ex) {
            if (i < maxRetries - 1) {
                const delay = initialDelay * Math.pow(2, i);
                await new Promise(resolve => setTimeout(resolve, delay));
            } else {
                throw new MaxRetriesExceededException();
            }
        }
    }
    throw new MaxRetriesExceededException();
}
```

### 8. **Idempotency Pattern**
- Use idempotency keys (UUIDs) for all notification requests
- Check database before processing
- Prevent duplicate processing

### 9. **Saga Pattern** (Distributed Transactions)
- For multi-step notification workflows
- Compensating actions for failures

### 10. **CQRS (Command Query Responsibility Segregation)**
- Separate read and write models
- Optimize for different access patterns
- Read replicas for status queries

---

## Component Design

### 1. Notification API Service

#### Controllers
```typescript
import { Router, Request, Response } from 'express';

const router = Router();

router.post('/api/notifications', async (req: Request, res: Response) => {
    // Validate request
    // Evaluate segments
    // Create notification records
    // Enqueue jobs
    // Return notification ID
});

router.get('/api/notifications/:id', async (req: Request, res: Response) => {
    // Return notification status and delivery details
});

router.get('/api/notifications/:id/deliveries', async (req: Request, res: Response) => {
    // Return per-recipient delivery status
});
```

#### Services
```typescript
interface INotificationService {
    createNotificationAsync(request: CreateNotificationRequest): Promise<Notification>;
    getStatusAsync(notificationId: string): Promise<NotificationStatus>;
    getDeliveryStatusesAsync(notificationId: string): Promise<DeliveryStatus[]>;
}

interface ISegmentService {
    evaluateSegmentAsync(segment: SegmentDefinition): Promise<string[]>;
    userMatchesSegmentAsync(userId: string, segment: SegmentDefinition): Promise<boolean>;
}

interface ITemplateService {
    renderTemplateAsync(templateId: string, data: object): Promise<string>;
    getTemplateAsync(templateId: string): Promise<Template>;
}
```

### 2. Scheduler Service

```typescript
interface ISchedulerService {
    scheduleImmediateAsync(request: NotificationRequest): Promise<void>;
    scheduleDelayedAsync(request: NotificationRequest, scheduledTime: Date): Promise<void>;
    scheduleRecurringAsync(request: NotificationRequest, rule: RecurrenceRule): Promise<void>;
    cancelScheduledAsync(notificationId: string): Promise<void>;
}

class SchedulerService implements ISchedulerService {
    constructor(
        private queueService: IQueueService,
        private cache: IDistributedCache,
        private repository: INotificationRepository
    ) {}
    
    async scheduleDelayedAsync(
        request: NotificationRequest, 
        scheduledTime: Date
    ): Promise<void> {
        const delay = scheduledTime.getTime() - Date.now();
        if (delay <= 0) {
            await this.scheduleImmediateAsync(request);
            return;
        }
        
        // Store in database for persistence
        await this.repository.createScheduledNotificationAsync(request, scheduledTime);
        
        // Enqueue with delay (Redis/Bull supports delayed jobs)
        await this.queueService.enqueueDelayedAsync(
            QueueNames.NotificationQueue,
            request,
            delay
        );
    }
}
```

### 3. Notification Worker Service

```typescript
import { Queue, Job } from 'bull';
import { EventEmitter } from 'events';

class NotificationProcessor extends EventEmitter {
    constructor(
        private queueService: IQueueService,
        private channelFactory: INotificationChannelFactory,
        private repository: INotificationRepository,
        private logger: ILogger
    ) {
        super();
    }
    
    async start(): Promise<void> {
        // Process critical priority queue
        const criticalTask = this.processQueueAsync(QueueNames.CriticalQueue);
            
        // Process high priority queue
        const highTask = this.processQueueAsync(QueueNames.HighQueue);
            
        // Process medium priority queue
        const mediumTask = this.processQueueAsync(QueueNames.MediumQueue);
            
        // Process low priority queue
        const lowTask = this.processQueueAsync(QueueNames.LowQueue);
            
        await Promise.all([criticalTask, highTask, mediumTask, lowTask]);
    }
    
    private async processQueueAsync(queueName: string): Promise<void> {
        const queue = this.queueService.getQueue(queueName);
        
        queue.process(async (job: Job) => {
            try {
                await this.processNotificationAsync(job);
            } catch (ex) {
                await this.handleFailureAsync(job, ex);
                throw ex;
            }
        });
    }
    
    private async processNotificationAsync(job: Job): Promise<void> {
        const { idempotencyKey, channelType, notificationId, recipientId, request } = job.data;
        
        // Check idempotency
        if (await this.repository.isProcessedAsync(idempotencyKey)) {
            this.logger.warn('Duplicate notification detected', { key: idempotencyKey });
            return;
        }
        
        // Get channel
        const channel = this.channelFactory.createChannel(channelType);
        
        // Send notification
        const result = await channel.sendAsync(request);
        
        // Update status
        await this.repository.updateDeliveryStatusAsync(
            notificationId,
            recipientId,
            result.status,
            result.providerMessageId
        );
    }
}
```

### 4. Channel Abstraction Layer

```typescript
interface INotificationChannel {
    readonly channelType: NotificationChannelType;
    sendAsync(request: NotificationRequest): Promise<DeliveryResult>;
    canHandle(channelType: NotificationChannelType): boolean;
    getCapabilities(): ChannelCapabilities;
}

abstract class BaseNotificationChannel implements INotificationChannel {
    abstract readonly channelType: NotificationChannelType;
    
    constructor(
        protected logger: ILogger,
        protected configuration: IConfiguration
    ) {}
    
    canHandle(channelType: NotificationChannelType): boolean {
        return this.channelType === channelType;
    }
    
    abstract sendAsync(request: NotificationRequest): Promise<DeliveryResult>;
    
    protected async executeWithRetryAsync(
        operation: () => Promise<DeliveryResult>,
        maxRetries: number = 3
    ): Promise<DeliveryResult> {
        for (let i = 0; i < maxRetries; i++) {
            try {
                return await operation();
            } catch (ex) {
                if (i < maxRetries - 1) {
                    this.logger.warn(ex, `Retry ${i + 1} for channel ${this.channelType}`);
                    await new Promise(resolve => setTimeout(resolve, Math.pow(2, i) * 1000));
                } else {
                    throw new MaxRetriesExceededException();
                }
            }
        }
        throw new MaxRetriesExceededException();
    }
}

class EmailChannel extends BaseNotificationChannel {
    readonly channelType = NotificationChannelType.Email;
    
    constructor(
        private emailProvider: IEmailProvider,
        logger: ILogger,
        configuration: IConfiguration
    ) {
        super(logger, configuration);
    }
    
    async sendAsync(request: NotificationRequest): Promise<DeliveryResult> {
        const emailRequest: EmailRequest = {
            to: request.recipient.email,
            subject: request.subject,
            body: request.body,
            htmlBody: request.htmlBody
        };
        
        return await this.executeWithRetryAsync(async () => {
            const response = await this.emailProvider.sendAsync(emailRequest);
            return {
                status: DeliveryStatus.Sent,
                providerMessageId: response.messageId,
                sentAt: new Date()
            };
        });
    }
}
```

---

## Data Models

### Database Schema

```sql
-- Notifications table (main notification requests)
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    idempotency_key UUID UNIQUE NOT NULL,
    notification_type VARCHAR(100) NOT NULL, -- order_confirmation, payment_received, etc.
    channel_type VARCHAR(50) NOT NULL, -- email, sms, push, in_app
    priority VARCHAR(20) NOT NULL DEFAULT 'medium', -- critical, high, medium, low
    template_id VARCHAR(100),
    subject VARCHAR(500),
    body TEXT,
    html_body TEXT,
    metadata JSONB, -- Flexible metadata storage
    segment_definition JSONB, -- Segment criteria
    scheduled_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(50) DEFAULT 'pending', -- pending, queued, processing, completed, failed
    created_by UUID REFERENCES users(id),
    
    INDEX idx_notifications_status (status),
    INDEX idx_notifications_priority (priority),
    INDEX idx_notifications_scheduled_at (scheduled_at),
    INDEX idx_notifications_idempotency_key (idempotency_key),
    INDEX idx_notifications_created_at (created_at)
);

-- Notification deliveries (per-recipient tracking)
CREATE TABLE notification_deliveries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notification_id UUID NOT NULL REFERENCES notifications(id) ON DELETE CASCADE,
    recipient_id UUID NOT NULL REFERENCES users(id),
    recipient_email VARCHAR(255),
    recipient_phone VARCHAR(50),
    channel_type VARCHAR(50) NOT NULL,
    status VARCHAR(50) DEFAULT 'pending', -- pending, queued, sending, delivered, failed, bounced
    provider_message_id VARCHAR(255), -- External provider's message ID
    sent_at TIMESTAMP,
    delivered_at TIMESTAMP,
    failed_at TIMESTAMP,
    failure_reason TEXT,
    retry_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE(notification_id, recipient_id), -- Prevent duplicate deliveries
    INDEX idx_deliveries_notification_id (notification_id),
    INDEX idx_deliveries_recipient_id (recipient_id),
    INDEX idx_deliveries_status (status),
    INDEX idx_deliveries_provider_message_id (provider_message_id)
);

-- Delivery status history (audit trail)
CREATE TABLE delivery_status_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    delivery_id UUID NOT NULL REFERENCES notification_deliveries(id) ON DELETE CASCADE,
    status VARCHAR(50) NOT NULL,
    provider_message_id VARCHAR(255),
    metadata JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_history_delivery_id (delivery_id),
    INDEX idx_history_created_at (created_at)
);

-- Notification templates
CREATE TABLE notification_templates (
    id VARCHAR(100) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    channel_type VARCHAR(50) NOT NULL,
    subject_template TEXT,
    body_template TEXT,
    html_body_template TEXT,
    variables JSONB, -- Template variable definitions
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_templates_channel_type (channel_type),
    INDEX idx_templates_is_active (is_active)
);

-- User segments (cached segment memberships)
CREATE TABLE user_segments (
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    segment_id VARCHAR(100) NOT NULL,
    segment_name VARCHAR(255),
    added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    PRIMARY KEY (user_id, segment_id),
    INDEX idx_segments_segment_id (segment_id)
);

-- Dead letter queue (failed notifications)
CREATE TABLE dead_letter_queue (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notification_id UUID REFERENCES notifications(id),
    delivery_id UUID REFERENCES notification_deliveries(id),
    job_data JSONB NOT NULL,
    failure_reason TEXT,
    error_details JSONB,
    retry_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    processed_at TIMESTAMP,
    
    INDEX idx_dlq_created_at (created_at),
    INDEX idx_dlq_processed_at (processed_at)
);

-- Channel configurations
CREATE TABLE channel_configurations (
    channel_type VARCHAR(50) PRIMARY KEY,
    provider_name VARCHAR(100) NOT NULL,
    configuration JSONB NOT NULL, -- Provider-specific config (API keys, endpoints, etc.)
    rate_limit_per_minute INT,
    rate_limit_per_hour INT,
    is_enabled BOOLEAN DEFAULT true,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Domain Models (TypeScript)

```typescript
interface Notification {
    id: string;
    idempotencyKey: string;
    notificationType: string;
    channelType: NotificationChannelType;
    priority: NotificationPriority;
    templateId?: string;
    subject?: string;
    body?: string;
    htmlBody?: string;
    metadata: Record<string, any>;
    segmentDefinition?: SegmentDefinition;
    scheduledAt?: Date;
    createdAt: Date;
    updatedAt: Date;
    status: NotificationStatus;
    createdBy?: string;
    
    deliveries?: NotificationDelivery[];
}

interface NotificationDelivery {
    id: string;
    notificationId: string;
    recipientId: string;
    recipientEmail?: string;
    recipientPhone?: string;
    channelType: NotificationChannelType;
    status: DeliveryStatus;
    providerMessageId?: string;
    sentAt?: Date;
    deliveredAt?: Date;
    failedAt?: Date;
    failureReason?: string;
    retryCount: number;
    createdAt: Date;
    updatedAt: Date;
    
    statusHistory?: DeliveryStatusHistory[];
}

enum NotificationChannelType {
    Email = 'email',
    Sms = 'sms',
    Push = 'push',
    InApp = 'in_app',
    Webhook = 'webhook'
}

enum NotificationPriority {
    Critical = 0,
    High = 1,
    Medium = 2,
    Low = 3
}

enum NotificationStatus {
    Pending = 'pending',
    Queued = 'queued',
    Processing = 'processing',
    Completed = 'completed',
    Failed = 'failed',
    Cancelled = 'cancelled'
}

enum DeliveryStatus {
    Pending = 'pending',
    Queued = 'queued',
    Sending = 'sending',
    Delivered = 'delivered',
    Failed = 'failed',
    Bounced = 'bounced',
    Unsubscribed = 'unsubscribed'
}

interface SegmentDefinition {
    criteria: SegmentCriteria[];
    operator: SegmentOperator; // 'AND' | 'OR'
}

interface SegmentCriteria {
    field: string; // age, purchase_count, last_purchase_date
    operator: string; // equals, greater_than, in, etc.
    value: any;
}

type SegmentOperator = 'AND' | 'OR';
```

---

## Event Flow & Processing

### Flow 1: Immediate Notification (Order Confirmation)

```
1. Order Service → Event Bus
   Event: OrderCreated
   {
     orderId: "123",
     userId: "456",
     total: 99.99,
     items: [...]
   }
   
2. Notification API Service (Event Handler)
   - Receives OrderCreated event
   - Evaluates: Should send notification? (Yes, order confirmation)
   - Creates notification request:
     {
       notificationType: "order_confirmation",
       channelType: "email",
       priority: "high",
       templateId: "order_confirmation_email",
       recipientId: "456",
       data: { orderId, total, items }
     }
   - Checks idempotency (orderId + userId)
   - Renders template with order data
   - Creates notification record in DB (status: pending)
   - Creates delivery record (status: pending)
   - Enqueues job to High Priority Queue
   - Updates notification status to "queued"
   
3. Queue System (Redis/Bull)
   - Job added to High Priority Queue
   - Job contains: notificationId, deliveryId, channelType, rendered content
   
4. Notification Worker
   - Consumes job from High Priority Queue
   - Gets channel (EmailChannel)
   - Calls channel.SendAsync()
   - EmailChannel calls SendGrid API
   - SendGrid responds with messageId
   - Worker updates delivery status to "sent"
   - Worker updates notification status to "processing"
   
5. SendGrid Webhook (Async)
   - SendGrid sends webhook when email delivered
   - Webhook handler updates delivery status to "delivered"
   - Updates notification status to "completed" (if all deliveries done)
```

### Flow 2: Scheduled Notification (Flash Sale Reminder)

```
1. Marketing Service → Notification API
   POST /api/notifications
   {
     notificationType: "flash_sale_reminder",
     channelType: "push",
     priority: "medium",
     segmentDefinition: {
       criteria: [
         { field: "last_purchase_date", operator: "greater_than", value: "30_days_ago" },
         { field: "preferences.categories", operator: "contains", value: "electronics" }
       ]
     },
     scheduledAt: "2024-12-25T10:00:00Z",
     templateId: "flash_sale_push"
   }
   
2. Notification API Service
   - Validates request
   - Evaluates segment (finds 10,000 matching users)
   - Creates notification record (status: pending, scheduledAt set)
   - Creates 10,000 delivery records (status: pending)
   - Scheduler Service enqueues job with delay
   - Job scheduled for 2024-12-25T10:00:00Z
   
3. Queue System
   - Job stored with delay until scheduledAt
   - Job will be available at scheduled time
   
4. At Scheduled Time (2024-12-25T10:00:00Z)
   - Queue makes job available
   - Worker consumes job
   - Processes all 10,000 deliveries in batches
   - Updates statuses as notifications are sent
```

### Flow 3: Failed Notification with Retry

```
1. Worker attempts to send notification
   - Calls SMS provider (Twilio)
   - Twilio returns 500 error (temporary failure)
   
2. Worker handles failure
   - Catches exception
   - Increments retry_count
   - Checks retry_count < max_retries (3)
   - Calculates backoff delay: 2^retry_count seconds
   - Re-enqueues job with delay
   - Updates delivery status to "pending" (for retry)
   
3. After backoff delay
   - Job becomes available again
   - Worker retries
   - Twilio succeeds
   - Updates status to "sent"
   
4. If all retries fail
   - Moves to Dead Letter Queue
   - Updates status to "failed"
   - Logs error details
   - Sends alert to operations team
```

---

## Scalability & Performance

### Horizontal Scaling Strategy

1. **Stateless Services**
   - API and Worker services are stateless
   - No session state stored in services
   - All state in database or cache
   - Easy to add/remove instances

2. **Load Balancing**
   - API services behind load balancer (round-robin, least connections)
   - Worker services consume from shared queues
   - Automatic scaling based on queue depth

3. **Database Scaling**
   - **Read Replicas**: Status queries use read replicas
   - **Connection Pooling**: Efficient connection management
   - **Partitioning**: Partition notifications table by date (monthly)
   - **Indexing**: Strategic indexes for common queries

4. **Queue Scaling**
   - Multiple worker instances process same queue
   - Priority queues ensure critical notifications processed first
   - Queue sharding for very high volume (by notification type)

5. **Caching Strategy**
   - **Template Cache**: Redis cache for rendered templates
   - **Segment Cache**: Cache segment evaluation results
   - **User Preferences Cache**: Cache user notification preferences
   - **Rate Limit Cache**: Redis for distributed rate limiting

### Performance Optimizations

1. **Batch Processing**
   - Process multiple deliveries in single transaction
   - Batch database updates
   - Batch API calls to providers (where supported)

2. **Async Processing**
   - All I/O operations async/await
   - Non-blocking queue operations
   - Parallel processing where safe

3. **Database Optimizations**
   - Efficient queries (avoid N+1)
   - Use JSONB indexes for metadata queries
   - Archive old notifications (move to cold storage)

4. **Provider Optimizations**
   - Connection pooling for provider APIs
   - Batch API calls where possible
   - Circuit breakers to avoid cascading failures

### Capacity Planning

**Assumptions:**
- 1M active users
- Average 10 notifications/user/month = 10M notifications/month
- Peak: 5x normal = 50M notifications/month during promotions
- 30 days/month = ~1.67M notifications/day = ~70K notifications/hour

**Scaling Calculations:**
- Each worker can process ~1,000 notifications/hour (conservative)
- Need ~70 worker instances for peak load
- With 3x headroom: ~210 worker instances
- Auto-scale: 10 instances (normal) to 210 instances (peak)

**Database Capacity:**
- 10M notifications/month × 12 months = 120M notifications/year
- With 5-year retention: 600M notifications
- Partition by month: 50M notifications/partition (manageable)

---

## Resilience & Reliability

### Idempotency

1. **Idempotency Keys**
   - Every notification request has unique idempotency key
   - Format: `{eventType}:{entityId}:{userId}:{timestamp}`
   - Example: `order_created:123:456:2024-12-25T10:00:00Z`

2. **Duplicate Detection**
   - Check idempotency_key in database before processing
   - If exists and completed, return existing result
   - If exists and processing, wait or return status

3. **Idempotent Operations**
   - Notification creation is idempotent
   - Delivery sending is idempotent (check delivery record)
   - Status updates are idempotent

### Retry Strategy

1. **Exponential Backoff**
   - Initial delay: 1 second
   - Max retries: 3
   - Backoff: 2^retry_count seconds
   - Max delay: 60 seconds

2. **Retry Conditions**
   - Retry on: Network errors, 5xx errors, timeouts
   - Don't retry on: 4xx errors (client errors), invalid credentials

3. **Dead Letter Queue**
   - After max retries, move to DLQ
   - Manual review and reprocessing
   - Alert operations team

### Circuit Breaker

1. **Purpose**: Prevent cascading failures
2. **Implementation**:
   - Track failure rate per provider
   - Open circuit after 50% failure rate (last 100 requests)
   - Half-open after 60 seconds
   - Close after successful requests

3. **Fallback**:
   - When circuit open, queue notification for later
   - Or route to backup provider

### Graceful Degradation

1. **Provider Failures**
   - If primary provider fails, try backup provider
   - If all providers fail, queue for later

2. **Database Failures**
   - Use cache for read operations
   - Queue writes for later processing

3. **Queue Failures**
   - Persist to database as backup
   - Recover from database on queue recovery

### Data Durability

1. **Write-Ahead Logging**
   - All notifications written to database before queueing
   - Ensures no data loss

2. **Queue Persistence**
   - Redis persistence enabled (AOF + RDB)
   - RabbitMQ durable queues

3. **Backup & Recovery**
   - Daily database backups
   - Point-in-time recovery
   - Disaster recovery plan

---

## Security & Compliance

### Data Privacy

1. **PII Encryption**
   - Encrypt email addresses, phone numbers at rest
   - Use TLS for data in transit
   - Key management via Vault/Secrets Manager

2. **Data Retention**
   - Automatic archival after retention period
   - GDPR right to deletion
   - Anonymization of old data

3. **Access Control**
   - Role-based access (RBAC)
   - API authentication (JWT/OAuth)
   - Audit logging for all operations

### Rate Limiting

1. **Per-User Limits**
   - Max notifications per user per day: 100
   - Max notifications per user per hour: 20

2. **Per-Channel Limits**
   - Email: 10,000/hour
   - SMS: 1,000/hour
   - Push: 50,000/hour

3. **Global Limits**
   - Max notifications per minute: 100,000
   - Circuit breaker on provider limits

### Compliance

1. **CAN-SPAM (Email)**
   - Unsubscribe links in all emails
   - Honor unsubscribe requests immediately
   - Include physical address

2. **TCPA (SMS)**
   - Opt-in required for SMS
   - Honor opt-out requests
   - Include opt-out instructions

3. **GDPR**
   - Consent management
   - Right to deletion
   - Data portability

---

## Monitoring & Observability

### Metrics

1. **Business Metrics**
   - Notifications sent per hour/day
   - Delivery rate (delivered/sent)
   - Open rate (email)
   - Click rate (email)
   - Bounce rate

2. **Technical Metrics**
   - Queue depth (per priority)
   - Processing latency (p50, p95, p99)
   - Error rate (4xx, 5xx)
   - Retry rate
   - Dead letter queue size

3. **Provider Metrics**
   - Provider response time
   - Provider error rate
   - Provider quota usage

### Logging

1. **Structured Logging**
   - JSON format
   - Correlation IDs
   - Log levels: Error, Warn, Info, Debug

2. **Key Events Logged**
   - Notification created
   - Notification queued
   - Delivery attempted
   - Delivery succeeded/failed
   - Retry attempts
   - Dead letter queue entries

### Tracing

1. **Distributed Tracing**
   - Trace ID across all services
   - Span for each operation
   - Visualize request flow

2. **Performance Tracing**
   - Identify bottlenecks
   - Database query performance
   - External API latency

### Alerting

1. **Critical Alerts**
   - Queue depth > 100K
   - Error rate > 5%
   - Dead letter queue > 1K
   - Provider circuit breaker open

2. **Warning Alerts**
   - Queue depth > 50K
   - Processing latency > 10s (p95)
   - Retry rate > 10%

---

## Implementation Phases

### Phase 1: Foundation (Weeks 1-4)
- [ ] Database schema design and migration
- [ ] Basic API service (create, status endpoints)
- [ ] Single channel implementation (Email)
- [ ] Basic queue system (Redis + Bull)
- [ ] Worker service (single priority queue)
- [ ] Idempotency implementation
- [ ] Basic monitoring

### Phase 2: Core Features (Weeks 5-8)
- [ ] Multi-channel support (SMS, Push)
- [ ] Priority queues
- [ ] Retry mechanism
- [ ] Delivery status tracking
- [ ] Template system
- [ ] Basic segmentation
- [ ] Dead letter queue

### Phase 3: Advanced Features (Weeks 9-12)
- [ ] Event-driven scheduling
- [ ] Advanced segmentation
- [ ] Scheduler service
- [ ] Webhook handlers (delivery receipts)
- [ ] Circuit breakers
- [ ] Rate limiting
- [ ] Caching layer

### Phase 4: Scale & Optimize (Weeks 13-16)
- [ ] Horizontal scaling
- [ ] Database read replicas
- [ ] Performance optimization
- [ ] Load testing
- [ ] Chaos engineering
- [ ] Documentation
- [ ] Runbooks

---

## Trade-offs & Design Decisions

### 1. Event Bus vs Direct API Calls
**Decision**: Event Bus (Kafka/RabbitMQ)
**Rationale**: 
- Decouples services
- Supports event replay
- Better scalability
**Trade-off**: Additional infrastructure complexity

### 2. Redis vs RabbitMQ for Queue
**Decision**: Redis + Bull (or RabbitMQ for enterprise)
**Rationale**:
- Redis: Fast, simple, supports delayed jobs
- RabbitMQ: Better routing, enterprise features
**Trade-off**: Redis simpler but less features

### 3. Database vs Queue for Scheduling
**Decision**: Queue with delayed jobs + Database persistence
**Rationale**:
- Queue handles timing efficiently
- Database ensures durability
**Trade-off**: Slight complexity but best of both worlds

### 4. At-Least-Once vs Exactly-Once Delivery
**Decision**: At-Least-Once with Idempotency
**Rationale**:
- Simpler to implement
- Idempotency handles duplicates
- Exactly-once is complex and may not be needed
**Trade-off**: Potential duplicates (handled by idempotency)

### 5. Synchronous vs Asynchronous Status Updates
**Decision**: Asynchronous (webhooks)
**Rationale**:
- Non-blocking
- Real-time updates
- Provider-driven
**Trade-off**: Requires webhook infrastructure

### 6. Single vs Multi-Region
**Decision**: Start single-region, design for multi-region
**Rationale**:
- Simpler initial implementation
- Can scale to multi-region later
**Trade-off**: Higher latency for global users initially

---

## Conclusion

This design provides a robust, scalable notification system that meets all specified requirements and additional NFRs. The architecture is:

- **Extensible**: Easy to add new channels
- **Scalable**: Horizontal scaling support
- **Resilient**: Retry, circuit breakers, idempotency
- **Observable**: Comprehensive monitoring
- **Maintainable**: Clean architecture, well-documented

The iterative implementation approach allows for incremental delivery while building towards the complete system.

---

## Next Steps

1. **Review & Refinement**: Review this design with stakeholders
2. **Proof of Concept**: Build POC for critical path (Email channel, basic queue)
3. **Detailed Design**: Create detailed API specs, database migration scripts
4. **Implementation**: Begin Phase 1 implementation
5. **Testing**: Unit, integration, load, chaos testing
6. **Deployment**: Gradual rollout with monitoring

---

*Document Version: 1.0*  
*Last Updated: 2024-12-25*  
*Author: System Design Team*

