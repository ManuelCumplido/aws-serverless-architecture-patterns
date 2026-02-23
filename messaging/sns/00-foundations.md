# SNS Foundations

## 1. Problem

Distributed systems require a reliable way to send events and notifications to multiple components without creating strong dependencies between them.

Synchronous communication introduces latency bottlenecks and failure propagation.
A publish/subscribe system enables scalable, decoupled, and real-time message distribution.

Amazon SNS (Simple Notification Service) is a managed publish/subscribe service
that allows applications to send events and notifications to multiple subscribers
in real time using a push-based model.

---

## 2. What is Amazon SNS?

Amazon SNS is a fully managed publish/subscribe messaging service
designed to handle large volumes of messages with low delivery delay.

It allows:

- Distributing one message to multiple subscribers (Fan-Out Pattern)
- Integration with multiple protocols
- Real-time push notifications
- Decoupled microservice communication

SNS is push-based: once a message is published to a topic,
it is immediately delivered to all active subscribers.

### High-Level Architecture Overview

![alt text](images/sns-architecture-overview.png)

---

## 3. Core Concepts

### 3.1 Topics

A Topic is a logical access point for publishing messages.

Producers publish messages to a topic.
Subscribers receive those messages.

Topics can be:

- Standard (default)
- FIFO (ordered + deduplication)

---

### 3.2 Subscriptions

A subscription connects a topic to an endpoint.

Supported protocols include:

- AWS Lambda
- Amazon SQS
- HTTP / HTTPS
- Email
- SMS
- Mobile Push (APNS / FCM)

Each subscription receives a copy of the published message.

---

### 3.3 Fan-Out Pattern

SNS supports fan-out architectures:

Producer → SNS Topic → Multiple Subscribers

Example:

- One event triggers:
  - A Lambda function
  - An SQS queue
  - A mobile push notification
  - An HTTP webhook

This enables horizontal scalability and independent service evolution.

---

### 3.4 Message Attributes and Filtering

SNS supports message attributes and subscription filter policies.

This allows selective message routing without requiring
custom filtering logic in consumers.

Example:

- Only deliver messages where `eventType = "DEVICE_ALERT"`

Filtering reduces unnecessary downstream processing.

---

## 4. Delivery Guarantees

SNS operates under:

- At-least-once delivery
- No strict ordering (unless using FIFO topics)
- Possible message duplication

Consumers must therefore implement:

- Idempotency
- Safe retry handling

SNS retries delivery for certain protocols (e.g., HTTP endpoints),
but behavior depends on subscription type.

---

## 5. SNS for Mobile Push

SNS integrates with mobile platforms through:

- Platform Applications
- Platform Endpoints

Terminology:

- Platform Application → Represents an iOS or Android app.
- Platform Endpoint → Represents a specific device instance.
- TargetArn → Unique identifier used to publish to a device.

SNS abstracts direct interaction with:

- APNS (Apple Push Notification Service)
- FCM (Firebase Cloud Messaging)

This enables a unified backend integration for cross-platform push delivery.

---

## 6. Common Failure Modes

### 6.1 Endpoint Disabled

Occurs when:

- Device token becomes invalid
- App is uninstalled
- Platform provider rejects the token

Requires endpoint lifecycle management.

---

### 6.2 Invalid Message Format

Incorrect platform-specific payload structure
may cause delivery failure.

---

### 6.3 Service Limits

High message publish rates may exceed service limits.
Monitoring and mechanisms to control message flow are required.

---

### 6.4 Permission Misconfiguration

Improper IAM policies may prevent:

- Publishing to topics
- Creating platform endpoints
- Managing subscriptions

---

## 7. Observability Considerations

SNS is asynchronous.
Failures may not be immediately visible.

Important signals include:

- Publish error rates
- Delivery failure metrics
- Subscription-level failures
- Endpoint disablement rates

CloudWatch monitoring is critical for production systems.

---

## 8. Design Philosophy of This SNS Module

This module focuses on:

- Production-oriented SNS usage
- Mobile push system design
- Endpoint lifecycle management
- Failure handling strategies
- Observability patterns
- Real-world scalability considerations

Subsequent documents will build on these foundations
to explore advanced SNS-based architectures.

---

## 9. When Not to Use SNS

While SNS is a powerful event distribution service, it is not suitable for all use cases.

### 9.1 Strict Global Ordering Requirements

SNS Standard topics do not guarantee ordering.
If strict ordering across all consumers is required, consider:

- SQS FIFO
- Kinesis Data Streams

Even with SNS FIFO topics, ordering is limited to message groups.

---

### 9.2 Long-Term Event Retention or Replay

SNS does not provide long-term message storage or replay capability.

If event replay, auditing, or stream reprocessing is required, consider:

- Amazon Kinesis
- Amazon EventBridge with archive & replay
- Direct SQS integration with retention strategy

---

### Architectural Decision Principle

SNS should be selected when:

- Fan-out distribution is required
- Real-time push delivery is needed
- Decoupled microservice communication is desired
- Operational simplicity is prioritized

Service selection should always be based on access patterns,
delivery guarantees, and operational characteristics.