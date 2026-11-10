---
service_id: ticketing-app/nats-test
service_name: nats-test
prompt_id: 06-failure-modes
prompt_label: Failure Modes & Edge Cases
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.13761
schema_version: "2.0"
---

# Error Handling Analysis: nats-test

## Part A: Error Handling Architecture

### Global Error Handler

**None.** This is a standalone demo/test package with no global error handling mechanism:

- No Express/Koa error middleware (no HTTP server exists)
- No exception filter or error boundary
- No `process.on('unhandledRejection')` handler
- No `process.on('uncaughtException')` handler

The only process-level handlers are signal handlers for graceful shutdown:
- `SIGINT`, `SIGTERM`, `SIGUSR2` → calls `stan.close()` (`src/publisher.ts:28-30`, `src/listener.ts:25-27`)

### Logging Library

**None.** Uses only built-in `console` methods:

| Method | Usage Location | Purpose |
|--------|---------------|---------|
| `console.log` | `src/publisher.ts:9,24`, `src/listener.ts:14`, `src/events/base-listener.ts:41-43`, `src/events/base-publisher.ts:21`, `src/events/ticket-created-listener.ts:11`, `src/events/ticket-updated-listener.ts:11` | Informational output |
| `console.error` | `src/events/base-listener.ts:49` | Error logging |

**No structured fields.** All logging is unstructured text.

### External Error Tracking

**None.** No integration with:
- Error monitoring services (Sentry, Bugsnag, Rollbar)
- Cloud alarms (CloudWatch, GCP Error Reporting)
- APM tools (New Relic, Datadog, Dynatrace)

---

## Part B: Integration Point Failure Behavior

### NATS Streaming Connection Failure

**Detection:**
- The NATS client uses an event-emitter pattern. Only `'connect'` and `'close'` events are handled (`src/publisher.ts:8,11`, `src/listener.ts:13,16`)
- **No `'error'` event handler** — connection failures are undetected
- If NATS is unreachable, the process hangs indefinitely waiting for the `'connect'` event that never fires

**Immediate response:**
- None. The application provides no feedback if connection fails.

**Retry policy:**
- None implemented in application code
- The `node-nats-streaming` library may have internal reconnection logic, but it is not configured or documented here

**Circuit breaker:**
- None

**Fallback:**
- None. If NATS is unavailable, the application cannot function.

**Alerting:**
- None

**Dead letter queue:**
- Not applicable for connection failures

**Recovery:**
- Manual process restart required after NATS becomes available

---

### NATS Message Publishing Failure

**Detection:**
- Callback-based error detection in `publish()` method (`src/events/base-publisher.ts:16-18`)
- If publish fails, the callback receives an error parameter

**Immediate response:**
- Promise rejection with the error (`src/events/base-publisher.ts:18`)
- Caller in `src/publisher.ts:17-25` catches and logs via `console.log(err)`

**Retry policy:**
- None. Failed publishes are logged and discarded.

**Circuit breaker:**
- None

**Fallback:**
- None. Failed messages are lost.

**Alerting:**
- None

**Dead letter queue:**
- None

**Recovery:**
- Manual republish required

---

### NATS Message Consumption Failure

**Detection:**
- `onMessage()` returns a Promise; rejection is caught in `.catch(console.error)` (`src/events/base-listener.ts:49`)

**Immediate response:**
- Error logged to console
- **Message is NOT acknowledged** — `msg.ack()` is only called on success (`src/events/base-listener.ts:48`)

**Retry policy:**
- NATS will redeliver after `ackWait` timeout expires (default: 5000ms, `src/events/base-listener.ts:16`)
- No explicit retry count limit — NATS Streaming handles this based on server configuration
- No exponential backoff in application code

**Circuit breaker:**
- None

**Fallback:**
- None. Failed messages are retried indefinitely (subject to NATS server limits).

**Alerting:**
- None

**Dead letter queue:**
- None configured in application code
- NATS Streaming may have server-side dead letter configuration, but not controlled here

**Recovery:**
- Automatic redelivery via NATS after ack timeout

---

### JSON Parse Failure (Message Deserialization)

**Detection:**
- `parseMessage()` calls `JSON.parse()` without try/catch (`src/events/base-listener.ts:56`)

**Immediate response:**
- **Uncaught exception** — `JSON.parse()` throws `SyntaxError` on invalid JSON
- The exception propagates, causing the Promise to reject
- Caught by `.catch(console.error)` in `listen()` (`src/events/base-listener.ts:49`)
- Message is NOT acknowledged (same behavior as `onMessage` failure)

**Retry policy:**
- Same as message consumption failure — NATS will redeliver
- **Poison message risk:** Invalid JSON will be retried forever since the message content cannot be fixed

---

## Part C: Data Consistency Edge Cases

### Process Crash Mid-Operation

| Scenario | Behavior | Evidence |
|----------|----------|----------|
| Publisher crashes after `stan.close()` called | Graceful — NATS connection closed cleanly | `src/publisher.ts:28-30` |
| Publisher crashes during `publish()` | Message may or may not have been delivered — unknown state | No transactional guarantees in code |
| Listener crashes before `msg.ack()` | Message redelivered after ack timeout (5s default) | `src/events/base-listener.ts:48` |
| Listener crashes after `msg.ack()` | Message considered processed — no rollback | N/A |

### Duplicate Message Processing (Idempotency)

**No idempotency checks exist.**

- The concrete listeners (`ticket-created-listener.ts`, `ticket-updated-listener.ts`) simply log the event data and resolve
- No message deduplication by ID
- No database checks for prior processing
- NATS redeliveries or duplicates (at-least-once delivery) will be processed multiple times

### Workflow Activity Retry (Side Effect Safety)

**Not applicable** — this package does not implement workflows or activities. The listeners have no side effects beyond logging.

### Concurrent Modification

**Handled by NATS queue groups:**
- Both listeners use queue group `'payments-service'` (`src/events/ticket-created-listener.ts:8`, `src/events/ticket-updated-listener.ts:8`)
- NATS ensures only one member of a queue group receives each message
- However, no optimistic locking or conflict resolution exists in application code

### Stale Data from Downstream Services

**Not applicable** — no downstream service calls or caching exists in this package.

---

## Part D: Known Gotchas

### TODO/FIXME/HACK/WORKAROUND Comments

**None found** in the codebase.

---

### Hardcoded Values That Should Be Configurable

| Value | Location | Issue |
|-------|----------|-------|
| `'http://localhost:4222'` | `src/listener.ts:10`, `src/publisher.ts:5` | NATS URL hardcoded; should be environment variable for deployment flexibility |
| `'ticketing'` | `src/listener.ts:9`, `src/publisher.ts:4` | NATS cluster ID hardcoded |
| `'abc'` | `src/publisher.ts:4` | Publisher client ID hardcoded; causes client ID conflict if multiple publishers run |
| `5 * 1000` | `src/events/base-listener.ts:16` | Ack wait timeout hardcoded as constructor default |
| `'payments-service'` | `src/events/ticket-created-listener.ts:8`, `src/events/ticket-updated-listener.ts:8` | Queue group name hardcoded |

---

### Timeout Values

| Timeout | Value | Location | Appropriateness |
|---------|-------|----------|-----------------|
| `ackWait` | 5000ms (5 seconds) | `src/events/base-listener.ts:16` | Default is reasonable for demo purposes; production may need longer for slow operations |

**Missing timeouts:**
- No connection timeout — if NATS is unreachable, process hangs forever
- No publish timeout — relies on underlying library behavior

---

### Rate Limits on External APIs

**Not applicable** — NATS Streaming does not have API rate limits in the traditional sense. Message throughput is limited by NATS server configuration, not client-side.

---

### Data Size Limits

**No validation exists in application code.**

- Message payloads are JSON-serialized via `JSON.stringify(data)` (`src/events/base-publisher.ts:16`)
- No maximum payload size enforced
- NATS Streaming server has a default max payload size (typically 1MB), but this is not documented or validated client-side
- No field length validation on event data (`id`, `title`, `price`)

---

## Summary of Critical Gaps

| Gap | Severity | Impact |
|-----|----------|--------|
| No connection error handling | High | Process hangs if NATS unreachable; no visibility into failure |
| No unhandledRejection handler | Medium | Unhandled promise rejections may crash Node.js (depending on version) |
| No structured logging | Medium | Difficult to diagnose issues in production |
| Poison message vulnerability | Medium | Invalid JSON messages will retry forever |
| No publish retry mechanism | Medium | Failed publishes are lost |
| Hardcoded NATS URL | Low (demo package) | Cannot deploy to different environments without code change |
| No idempotency | Low (demo package) | Duplicate processing if messages redelivered |
