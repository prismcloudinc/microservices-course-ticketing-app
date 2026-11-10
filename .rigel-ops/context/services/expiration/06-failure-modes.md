---
service_id: ticketing-app/expiration
service_name: expiration
prompt_id: 06-failure-modes
prompt_label: Failure Modes & Edge Cases
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.27577000000000007
schema_version: "2.0"
---

# Error Handling & Failure Analysis: expiration Service

## Part A: Error Handling Architecture

### Global Error Handler

**Status**: No global error handler exists.

The expiration service is a worker process without an HTTP server. Unlike the other services in this monorepo (auth, orders, tickets, payments), which use Express and the shared `errorHandlerMiddleware` from `@b.anik/ticketing-common`, this service has no centralized error handling layer.

**Error handling is distributed across:**
1. Try-catch block in startup (`src/index.ts`, lines 35-38) — catches initial NATS connection errors
2. Promise rejection in `onMessage` (`src/events/listeners/order-created-listener.ts`, lines 29-31) — propagates to base listener
3. Base listener catch in `.catch(console.error)` (`nats-test/src/events/base-listener.ts`, line 49) — logs but does not ack

### Logging

| Aspect | Details |
|--------|---------|
| **Library** | None — uses native `console.log` and `console.error` |
| **Log Levels** | Implicit: `log` (info) and `error` only |
| **Structured Fields** | None — plain string messages |
| **Log Locations** | `src/index.ts` lines 23, 26, 36 |

**Observation**: There is no structured logging, correlation IDs, or request tracing. Debugging production issues requires parsing unstructured stdout.

### External Error Tracking

**Status**: None configured.

No integration with Sentry, Datadog, New Relic, Bugsnag, Rollbar, or any other error monitoring service was found in the codebase. Errors logged to console are the only visibility mechanism.

---

## Part B: Integration Point Failure Behavior

### NATS Streaming Server Failure

| Aspect | Behavior |
|--------|----------|
| **Detection** | Promise rejection from `natsWrapper.connect()` (`src/nats-wrapper.ts`, line 21); `close` event on client (`src/index.ts`, line 25) |
| **Immediate Response** | Logs error via `console.log(err)`, calls `process.exit(1)` (`src/index.ts`, lines 36-38) |
| **Retry Policy** | None — process terminates immediately on connection failure |
| **Circuit Breaker** | None |
| **Fallback** | None — service cannot operate without NATS |
| **Alerting** | None configured — relies on Kubernetes to detect pod failure and restart |
| **Dead Letter Queue** | Not applicable |
| **Recovery** | Kubernetes restarts the pod; on restart, the listener re-subscribes from durable subscription position |

**Code Path** (`src/index.ts`, lines 17-38):
```typescript
try {
  await natsWrapper.connect(
    process.env.NATS_CLUSTER_ID!,
    process.env.NATS_CLIENT_ID!,
    process.env.NATS_URL!,
  );
  // ...
  natsWrapper.client.on('close', () => {
    console.log('NATS connection closed!');
    process.exit(1);
  });
} catch (err) {
  console.log(err);
  process.exit(1);
}
```

**Gap**: No reconnection attempt is made. If NATS connection is lost after initial connect, the process exits immediately rather than attempting to reconnect.

---

### Redis (Bull Queue) Failure

| Aspect | Behavior |
|--------|----------|
| **Detection** | Not explicitly handled — Bull handles internally |
| **Immediate Response** | None — no `.on('error')` handler registered on the queue |
| **Retry Policy** | Bull default: 3 attempts with exponential backoff (not explicitly configured) |
| **Circuit Breaker** | None |
| **Fallback** | None |
| **Alerting** | None |
| **Dead Letter Queue** | None configured |
| **Recovery** | Bull automatically retries jobs; if Redis recovers, queue processing resumes |

**Code** (`src/queues/expiration-queue.ts`, lines 9-13):
```typescript
const expirationQueue = new Queue<Payload>('order:expiration', {
  redis: {
    host: process.env.REDIS_HOST,
  },
});
```

**Gaps**:
1. No error event handler on the queue (`expirationQueue.on('error', ...)` not registered)
2. No failed job handler (`expirationQueue.on('failed', ...)` not registered)
3. No stalled job handler (`expirationQueue.on('stalled', ...)` not registered)
4. No explicit retry configuration — relies on Bull defaults
5. `REDIS_HOST` environment variable is not validated at startup (unlike `NATS_URL` and `NATS_CLUSTER_ID`)

---

### Job Processing Failure (Bull Worker)

| Aspect | Behavior |
|--------|----------|
| **Detection** | Uncaught exception in processor function |
| **Immediate Response** | Job marked as failed by Bull; Bull may retry based on default settings |
| **Retry Policy** | Bull default: 3 attempts |
| **Fallback** | None — no fallback publishing mechanism |
| **Dead Letter Queue** | None configured — failed jobs remain in failed state |

**Code** (`src/queues/expiration-queue.ts`, lines 15-19):
```typescript
expirationQueue.process(async (job) => {
  await new ExpirationCompletePublisher(natsWrapper.client).publish({
    orderId: job.data.orderId,
  });
});
```

**Gap**: The processor function has no try-catch. If publishing to NATS fails (e.g., connection lost), the error bubbles up uncaught. Bull will retry the job, but there's no logging of the failure reason.

---

### Message (OrderCreated Event) Processing Failure

| Aspect | Behavior |
|--------|----------|
| **Detection** | Rejection from `onMessage()` promise |
| **Immediate Response** | Base listener catches rejection, logs via `console.error`, does NOT acknowledge message |
| **Retry Policy** | NATS redelivers unacked messages after `ackWait` (5 seconds, hardcoded in base listener) |
| **Fallback** | None |
| **Dead Letter Queue** | None — NATS Streaming has no built-in DLQ |

**Code in base listener** (`nats-test/src/events/base-listener.ts`, lines 47-49):
```typescript
this.onMessage(parsedData, msg)
  .then(() => msg.ack())
  .catch(console.error);
```

**Key Behavior**: If `onMessage` throws/rejects, the message is NOT acknowledged. NATS will redeliver it after `ackWait` (5 seconds). This provides at-least-once delivery, but messages can be processed multiple times.

---

## Part C: Data Consistency Edge Cases

### Service Crash Mid-Operation

**Scenario**: Service crashes after `expirationQueue.add()` but before `resolve()` is called.

**Actual Behavior**:
1. Job is added to Redis queue — this is persisted
2. Message is NOT acked (crash prevents ack)
3. NATS redelivers the message
4. Listener adds another job with the same `orderId`
5. **Two expiration jobs exist** for the same order

**Impact**: Duplicate `ExpirationComplete` events will be published. The orders service handles this gracefully (`src/events/listeners/expiration-complete-listener.ts`, lines 20-23):
```typescript
if (order.status === OrderStatus.Cancelled || order.status === OrderStatus.Complete) {
  return resolve();
}
```

**Assessment**: Downstream service is idempotent, but expiration service is not.

---

### Duplicate Message Processing (Idempotency)

**Scenario**: NATS redelivers `OrderCreated` event (network timeout, slow processing, restart).

**Actual Behavior**: No idempotency check in `OrderCreatedListener` — job is added unconditionally.

**Code** (`src/events/listeners/order-created-listener.ts`, lines 18-26):
```typescript
await expirationQueue.add(
  { orderId: data.id },
  { delay },
);
```

**Gap**: Should use `jobId` option to make job addition idempotent:
```typescript
await expirationQueue.add(
  { orderId: data.id },
  { delay, jobId: data.id }  // Missing: this would prevent duplicates
);
```

---

### Workflow Activity/Job Retry

**Scenario**: Bull job processor fails and is retried.

**Actual Behavior**: `ExpirationCompletePublisher.publish()` is called again on retry, potentially publishing duplicate events.

**Impact**: Orders service receives duplicate `ExpirationComplete` events. As noted above, it handles this gracefully.

---

### Concurrent Modifications

**Not applicable**: The expiration service does not write to a database. It only:
1. Reads events from NATS
2. Writes jobs to Redis queue
3. Publishes events to NATS

No concurrent modification scenarios exist within this service.

---

### Stale Data

**Not applicable**: The service doesn't cache data or make decisions based on data freshness. It passes `orderId` through without transformation.

---

### Negative Delay Edge Case

**Scenario**: `OrderCreated` event arrives after the `expiresAt` time has already passed (delayed delivery, clock skew, or backlog processing).

**Code** (`src/events/listeners/order-created-listener.ts`, line 17):
```typescript
const delay = new Date(data.expiresAt).getTime() - new Date().getTime();
```

**Actual Behavior**: `delay` will be negative. Bull treats negative delays as immediate execution (`delay: 0`).

**Impact**: Job fires immediately, which is correct behavior — the order should be expired if its expiration time has passed.

**Assessment**: This is handled correctly by Bull's semantics, though the behavior is implicit rather than explicit.

---

## Part D: Known Gotchas

### Code Quality Issues

| File | Line | Finding |
|------|------|---------|
| `src/index.ts` | 9-14 | **Duplicate validation**: `NATS_CLUSTER_ID` is checked twice; `NATS_CLIENT_ID` is NOT checked despite being used on line 20 |
| `src/index.ts` | 20 | **Unvalidated required env var**: `process.env.NATS_CLIENT_ID!` is used with non-null assertion but never validated |
| `src/queues/expiration-queue.ts` | 11 | **Unvalidated env var**: `REDIS_HOST` is not validated at startup; if undefined, Redis connects to `localhost` which will fail in Kubernetes |
| `src/events/listeners/queue-group-name.ts` | 1 | **Unused file**: Exports `queueGroupName` constant, but `OrderCreatedListener` hardcodes `'expiration-service'` instead of importing it |

### Hardcoded Values

| File | Line | Value | Concern |
|------|------|-------|---------|
| `nats-test/src/events/base-listener.ts` | 16 | `ackWait = 5 * 1000` (5 seconds) | Not configurable; may be too short for slow operations |
| `infra/k8s/nats-depl.yaml` | 25-29 | `-hbi 5s -hbt 5s -hbf 2` | NATS heartbeat settings hardcoded in deployment manifest |

### Missing Health Checks

**Gap**: Unlike other services in this monorepo, the expiration service has no health check endpoint.

| Service | Health Check |
|---------|--------------|
| auth | `/api/users/health` (liveness probe configured) |
| orders | `/api/orders/health` (liveness probe configured) |
| tickets | `/api/tickets/health` (liveness probe configured) |
| payments | `/api/payments/health` (liveness probe configured) |
| **expiration** | **None** |

**Impact**: Kubernetes cannot detect if the worker is healthy. A service that is running but unable to process messages (e.g., NATS subscription lost but process didn't exit) will not be restarted.

### Missing Resource Limits

The Kubernetes deployment (`infra/k8s/expiration-depl.yaml`) does not specify resource limits or requests. Other deployments in this monorepo also lack them, but this is notable for a worker service that could potentially process many jobs.

### Timeout Values

| Component | Timeout | Configured In |
|-----------|---------|---------------|
| NATS ack wait | 5 seconds | Base listener (hardcoded) |
| Bull job processing | None | No `timeout` option set |
| Redis connection | Bull defaults | Not explicitly configured |

**Concern**: No job processing timeout is configured. A hung job processor will block the worker indefinitely.

### Rate Limits

No rate limiting is implemented on:
- Incoming events from NATS (processes all events as fast as possible)
- Job additions to Bull queue
- Event publishing to NATS

**Impact**: A burst of `OrderCreated` events could overwhelm Redis or NATS.

### Data Size Limits

No explicit limits are configured for:
- NATS message payload size (relies on NATS server defaults)
- Bull job data size (relies on Redis defaults)

---

## Summary of Critical Gaps

| Priority | Gap | Impact |
|----------|-----|--------|
| High | No idempotency check when adding jobs to queue | Duplicate jobs on message redelivery |
| High | No validation for `NATS_CLIENT_ID` and `REDIS_HOST` | Process may fail with confusing errors |
| High | No health check endpoint | Kubernetes cannot detect unhealthy workers |
| Medium | No error event handlers on Bull queue | Failures are silent |
| Medium | No structured logging | Difficult to debug production issues |
| Medium | No external error tracking | No visibility into production errors |
| Medium | No job processing timeout | Hung jobs block worker indefinitely |
| Low | Duplicate `NATS_CLUSTER_ID` validation | Code quality issue |
| Low | Unused `queue-group-name.ts` file | Dead code |
