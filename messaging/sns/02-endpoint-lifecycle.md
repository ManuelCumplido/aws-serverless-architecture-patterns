# Endpoint Lifecycle Management in SNS Mobile Push

## 1. Why Endpoint Lifecycle Matters

In mobile push architectures, reliability issues rarely originate
from SNS itself.

Most failures occur due to improper endpoint lifecycle management.

An SNS Platform Endpoint represents a specific device instance.
If its state becomes invalid and is not handled properly,
publish operations will continuously fail.

A robust lifecycle strategy is therefore critical.

---

## 2. Endpoint Lifecycle Stages

An SNS Platform Endpoint typically moves through the following states:

1. Created
2. Active
3. Disabled
4. Re-registered
5. Deleted (optional cleanup)

Understanding these transitions prevents silent delivery degradation.

---

## 3. Token Registration Flow

### 3.1 Initial Registration

1. Mobile app retrieves device token from APNS / FCM.
2. App sends token to backend.
3. Backend:
   - Creates SNS Platform Endpoint.
   - Stores TargetArn associated with userId and deviceId. (database)

Best Practices:

- Avoid creating duplicate endpoints.
- Validate existing endpoint before creation.
- Ensure idempotent registration logic.

Interview Insight:

> A common mistake is creating a new endpoint on every login
> instead of updating the existing one.

---

## 4. Token Refresh Handling

Device tokens may change due to:

- App reinstall
- OS updates
- Provider rotation
- Security updates

When token changes:

1. App sends updated token.
2. Backend must:
   - Update the existing endpoint.
   - Or recreate endpoint if invalid.

Never assume tokens are permanent.

Interview Insight:

> Token refresh events must be treated as first-class lifecycle events,
> not as edge cases.

---

## 5. Handling EndpointDisabled

SNS may mark an endpoint as Disabled when:

- App is uninstalled.
- Token is invalid.
- Provider rejects the message.

If ignored:

- Publish operations will repeatedly fail.
- Logs will accumulate errors.
- Operational noise increases.

Recommended Handling Strategy:

1. Detect EndpointDisabled response.
2. Mark endpoint as inactive in database.
3. Prevent future publish attempts.
4. Trigger re-registration on next user login.

This reduces repeated failure attempts.

---

## 6. Database Design Considerations

Store:

- userId
- deviceId
- TargetArn
- platform
- endpointStatus (ACTIVE / DISABLED)
- lastUpdatedAt

Do not store:

- Sensitive provider credentials
- Raw token without encryption (if stored)

Design should support:

- Multiple devices per user
- Efficient lookup by userId
- Fast filtering by active endpoints

---

## 7. Idempotency Strategy

Lifecycle operations must be idempotent.

Scenarios:

- Duplicate token registration requests
- Retries due to network failures
- Concurrent login sessions

Idempotent Design:

- Check if TargetArn exists before creating.
- Update endpoint only if token changed.
- Avoid duplicate records.

Interview Insight:

> Endpoint lifecycle APIs must be idempotent,
> as mobile clients may retry requests automatically.

---

## 8. Cleanup Strategy

Optional but recommended:

Periodic cleanup of:

- Long-disabled endpoints
- Inactive devices
- Stale TargetArns

Possible approaches:

- Scheduled Lambda cleanup job
- Event-driven cleanup based on failure rate
- Soft-delete strategy

---

## 9. Failure Patterns Observed in Production

Common operational issues:

- Massive EndpointDisabled spikes after app release.
- Forgotten token refresh logic.
- Creating new endpoint on every login.
- Publishing without checking endpoint status.

Monitoring lifecycle trends is as important as monitoring publish metrics.

---

## 10. Architectural Principles

Effective endpoint lifecycle management should:

- Be idempotent
- Be observable
- Prevent repeated failure loops
- Support multi-device users
- Handle token refresh gracefully

SNS provides delivery infrastructure.
Endpoint lifecycle ownership remains the responsibility of the backend system.