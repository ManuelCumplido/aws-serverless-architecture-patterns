# SNS Message Formats for Mobile Push

## 1. Why Message Format Matters

Amazon SNS does not send a single universal payload format.

Instead, it forwards messages to platform providers (APNS, FCM)
using provider-specific JSON structures.

Incorrect formatting is one of the most common causes of:

- Silent notification failures
- Missing titles or bodies
- Notifications not appearing in background
- Platform-specific delivery inconsistencies

Understanding message structure is essential for reliable push delivery.

---

## 2. SNS Publish Structure Overview

When publishing to a mobile endpoint, SNS requires:

- TargetArn
- Message
- MessageStructure: "json"

Example:

{
  "MessageStructure": "json",
  "TargetArn": "arn:aws:sns:region:account:endpoint/...",
  "Message": "{ \"GCM\": \"{...}\", \"APNS\": \"{...}\" }"
}

Important:

The outer Message is a JSON string.
The inner platform payload must also be stringified.

This double-encoding is mandatory.

---

## 3. Android (FCM / GCM) Format

SNS still uses the key "GCM" for Android.

### 3.1 Basic Notification Payload

{
  "GCM": "{ 
    \"notification\": {
      \"title\": \"Temperature Updated\",
      \"body\": \"Your thermostat is now set to 22°C\"
    },
    \"data\": {
      \"deviceId\": \"abc123\",
      \"type\": \"TEMP_UPDATE\"
    }
  }"
}

### 3.2 Key Fields

- notification → UI-visible notification
- data → Custom key-value payload for app handling
- priority (optional) → high / normal

Best Practice:

Always include data.type to allow client-side routing logic.

---

## 4. iOS (APNS) Format

SNS supports:

- APNS (Production)
- APNS_SANDBOX (Development)

### 4.1 Basic APNS Payload

{
  "APNS": "{ 
    \"aps\": {
      \"alert\": {
        \"title\": \"Temperature Updated\",
        \"body\": \"Your thermostat is now set to 22°C\"
      },
      \"sound\": \"default\"
    },
    \"deviceId\": \"abc123\",
    \"type\": \"TEMP_UPDATE\"
  }"
}

### 4.2 Required Structure

APNS requires:

- Top-level aps object
- alert for visible notification
- Optional badge, sound, content-available

Incorrect nesting will cause silent drops.

---

## 5. Cross-Platform Unified Message Strategy

A robust backend typically builds:

{
  "default": "Fallback message",
  "GCM": "...",
  "APNS": "..."
}

- default is used for unsupported platforms.
- Each platform payload is tailored.

Never assume Android and iOS behave identically.

---

## 6. Silent Notifications

Silent pushes are used for:

- Background data refresh
- Device state synchronization
- IoT state updates

### Android Silent Example

{
  "GCM": "{ 
    \"data\": {
      \"type\": \"SYNC_REQUEST\"
    }
  }"
}

### iOS Silent Example

{
  "APNS": "{ 
    \"aps\": {
      \"content-available\": 1
    },
    \"type\": \"SYNC_REQUEST\"
  }"
}

Important:

iOS requires:
- content-available: 1
- Proper background modes enabled in the app

---

## 7. Common Formatting Mistakes

### 7.1 Not Stringifying Inner JSON

Incorrect:

"Message": {
  "GCM": { ... }
}

Correct:

"Message": "{ \"GCM\": \"{...}\" }"

---

### 7.2 Missing Platform-Specific Keys

- Missing aps in APNS
- Using notification only without data
- Forgetting MessageStructure: "json"

---

### 7.3 Payload Size Limits

- APNS: ~4KB limit
- FCM: ~4KB limit

Large payloads will fail silently or be rejected.

Avoid sending full objects.
Send identifiers instead.

---

## 8. Recommended Backend Abstraction

Avoid building raw SNS payloads across your codebase.

Instead:

- Create a Notification Builder module
- Accept structured internal input:
  - type
  - title
  - body
  - metadata
- Generate platform-specific payloads centrally

This ensures:

- Consistency
- Easier updates
- Safer platform migrations

Interview Insight:

Well-designed systems centralize SNS message formatting inside a dedicated
notification service module, instead of building raw platform-specific JSON
payloads directly inside multiple Lambda functions.

This improves:

- Maintainability
- Consistency
- Testability
- Easier platform updates

---

## 9. Observability and Debugging

When debugging message delivery:

Check:

- CloudWatch logs for publish errors
- SNS Delivery Status logs
- Endpoint attributes (Enabled flag)
- Provider response codes

Remember:

If formatting is wrong, SNS may accept the publish,
but the provider (APNS/FCM) may reject it.

---

## 10. Architectural Principles

A reliable SNS message format strategy should:

- Be platform-aware
- Support silent and visible pushes
- Minimize payload size
- Abstract formatting logic
- Be testable

SNS handles distribution.
Message correctness remains your responsibility.