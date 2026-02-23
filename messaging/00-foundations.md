# Messaging Foundations in AWS

## 1. Problem

Modern distributed systems require asynchronous communication
to achieve scalability, decoupling, and resilience.

Direct synchronous communication between services or devices
introduces tight coupling, latency bottlenecks, and failure propagation.

Messaging systems solve this by introducing event-driven interaction patterns.

---

## 2. Core Principles

### 2.1 Asynchronous Communication

Producers should not depend on the immediate availability of consumers.

Benefits:
- Loose coupling
- Independent scaling
- Improved fault tolerance

---

### 2.2 Event-Driven Architecture

Events represent state changes or business actions.

Examples:
- User registered
- Device status changed
- Reservation created
- Firmware update available

Messaging systems distribute these events to interested subscribers.

---

### 2.3 Decoupling

Publishers and subscribers must not know about each other.

This enables:
- Independent deployments
- Independent scaling
- Service evolution without system-wide impact

---

## 3. Messaging Services in AWS

AWS provides multiple messaging services, each designed for specific patterns:

### SNS (Simple Notification Service)
- Pub/Sub model
- Push-based delivery
- Fan-out support
- Mobile push integration
- No message persistence guarantee for long-term storage

Best for:
- Real-time notifications
- Fan-out patterns
- Event broadcasting

---

### SQS (Simple Queue Service)
- Queue-based model
- Pull-based consumption
- Durable message storage
- Backpressure handling

Best for:
- Decoupling services
- Work queues
- Retry control

---

### EventBridge
- Event routing with filtering rules
- SaaS and AWS service integrations
- Schema registry support

Best for:
- Cross-service event routing
- Integration-heavy systems

---

## 4. Delivery Guarantees

Understanding delivery guarantees is critical.

Most AWS messaging services operate with:

- At-least-once delivery
- No strict ordering (unless explicitly configured)
- Potential message duplication

Systems must therefore implement:

- Idempotency
- Retry strategies
- Dead-letter queues (when applicable)

---

## 5. Failure Modes in Messaging Systems

Common failure scenarios include:

- Consumer unavailability
- Message duplication
- Invalid payload contracts
- Expired device tokens (mobile push)
- Throttling limits
- Partial fan-out failure

Messaging systems must be designed with failure in mind.

---

## 6. Observability Considerations

Messaging systems require strong observability to detect silent failures.

Important signals:

- Publish failures
- Delivery failures
- Queue depth (SQS)
- Throttling metrics
    - Dead-letter queue activity

Without observability, asynchronous failures can go unnoticed.

---

## 7. Trade-offs

| Service | Strength | Trade-off |
|----------|----------|-----------|
| SNS | Real-time push, fan-out | Limited message durability |
| SQS | Reliable queuing | Consumer-managed scaling |
| EventBridge | Advanced routing | Added latency and cost |

There is no universal solution.
Service choice depends on system constraints and architectural goals.

---

## 8. Design Philosophy of This Module

This messaging module focuses on:

- Production-oriented patterns
- Real-world failure handling
- Scalability considerations
- Observability strategies
- System-level trade-offs

Each subsequent document builds upon these foundations.
