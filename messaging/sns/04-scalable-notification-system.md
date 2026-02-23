# Designing a Scalable Notification System with SNS

## 1. Why Scalability Matters

As user base grows, notification systems face:

- Increased publish volume
- Multiple device endpoints per user
- Too many retries happening at the same time
- Many device tokens becoming invalid at once
- Handling Android and iOS differently

A naive implementation may work at small scale, but will fail under high concurrency or large fan-out scenarios.

A scalable design must handle:

- High throughput
- Idempotency
- Fault isolation
- Observability
- Multi-device support

---

## 2. High-Level Architecture

Core components:

- Application Service (business trigger)
- Notification Service Layer
- SNS Platform Application
- SNS Platform Endpoints
- Mobile Devices
- Database (endpoint registry)
- Observability layer (CloudWatch)

Flow:

1. Business event occurs (e.g., device state change).
2. Application calls Notification Service.
3. Notification Service retrieves active endpoints.
4. SNS publish is executed.
5. Delivery status is monitored.
6. Endpoint lifecycle updated if needed.

---

## 3. Multi-Device Per User Strategy

Users may have:

- iPhone
- Android phone
- Tablet
- Multiple sessions

Database must support:

- Multiple endpoints per user
- Filtering only ACTIVE endpoints
- Efficient lookup by userId

Recommended model:

PK: USER#<userId>  
SK: DEVICE#<deviceId>  

Attributes:

- endpointArn
- platform
- status (ACTIVE / DISABLED)
- lastUpdatedAt

Never assume one device per user.

---

## 4. Publish Strategy at Scale

### 4.1 Direct Publish per Endpoint

Loop through endpoints:

```javascript
for (const endpoint of activeEndpoints) {
  await sns.publish({...});
}
```

Simple but may cause:

- Hitting AWS request limits (Throttling)
- Increased latency
- Too many retries making the problem worse

---

### 4.2 Parallel Publish with Concurrency Control

When sending notifications to multiple endpoints,
avoid publishing to all of them at the same time without limits.

Instead, send notifications in controlled parallel batches.

Example:

```javascript
const poolLimit = 10;

for (let i = 0; i < endpoints.length; i += poolLimit) {
  const batch = endpoints.slice(i, i + poolLimit);

  await Promise.all(
    batch.map(endpoint =>
      sns.publish({
        MessageStructure: "json",
        TargetArn: endpoint.endpointArn,
        Message: messagePayload
      }).promise()
    )
  );
}
```

In this example:

- Only 10 publish operations run at the same time.
- Once those finish, the next batch starts.
- The system avoids overwhelming SNS or Lambda concurrency limits.

This prevents:

- Reaching AWS request limits
- Increasing failure rates under heavy load
- Cascading retry scenarios

Do not publish to hundreds or thousands of endpoints simultaneously without controlling concurrency.

---

### 4.3 Topic-Based Fan-Out (Advanced Pattern)

Alternative approach:

1. Create SNS Topic per user.
2. Subscribe endpoints to that topic.
3. Publish once to topic.

Pros:

- Single publish call
- Native SNS fan-out
- Reduced backend logic

Cons:

- Higher infrastructure management complexity
- Subscription lifecycle overhead

Use when:

- High multi-device users
- Broadcast-like use cases

---

## 5. Retry and Failure Isolation

At scale, failures will occur.

Design must handle:

- EndpointDisabled
- Temporary provider failures
- SNS throttling

Best practice:


- Handle publish failures with clear try/catch logic and structured logs (include endpointArn and notification type)
- If the failure indicates the endpoint is no longer valid (e.g., EndpointDisabled), mark it as DISABLED in the database and stop sending to it
- Do not retry forever; use a maximum retry count and/or a DLQ for failed messages
- For temporary errors, retry with increasing delays (e.g., 1s, 2s, 4s) instead of retrying immediately

Do not retry blindly.

---

## 6. Traffic Control and Burst Handling

A rapid increase in notifications sent within a short period of time (burst scenarios)

Potential problems:

- A large number of Lambda functions running at the same time
- SNS limiting or rejecting publish requests due to high volume
- Other systems receiving too many requests at once

Mitigation patterns:

- Use SQS as a queue before publishing notifications, so messages are processed gradually instead of all at once
- Limit how many notifications can be sent per second
- Combine multiple related updates into a single notification when possible
- If many updates happen quickly, send only the final update instead of every intermediate change

Example:

Instead of sending 10 temperature updates,
send only the final state.

---

## 7. Observability at Scale

Monitor:

- Publish success rate
- Publish error rate
- EndpointDisabled trend
- Delivery latency
- Per-platform failure ratio

Critical signal:

A spike in EndpointDisabled after an app release.

This often indicates:

- Token handling regression
- Mobile integration bug

Observability must include:

- Structured logs
- CloudWatch metrics
- Set limits that send alerts when error levels become too high

---

## 8. Idempotency at Scale

Mobile clients may:

- Retry token registration
- Retry login
- Send duplicate requests

System must ensure:

- No duplicate endpoints
- No duplicate notifications
- Safe retries

Techniques:

- Conditional writes (e.g., DynamoDB)
- Idempotency keys
- State validation before publish

---

## 9. Security Considerations

Protect:

- Platform Application credentials
- Device tokens (if stored)
- Publish permissions

IAM best practices:

- Restrict SNS publish to specific Platform Application ARN
- Separate roles for registration vs publish
- Avoid wildcard SNS permissions

---

## 10. Cost Optimization

Cost drivers:

- Publish requests
- Lambda invocations
- Logging volume

Optimization strategies:

- Aggregate non-critical notifications
- Avoid duplicate publishes
- Disable verbose logs in production
- Remove old or inactive endpoints that are no longer valid

---

## 11. Architectural Principles

A scalable notification system should:

- Decouple business logic from delivery logic
- Support multi-device users
- Be lifecycle-aware
- Prevent too many retries from happening at the same time
- Handle burst traffic safely
- Provide clear observability

SNS provides distribution capability.
Scalability depends on backend design discipline.