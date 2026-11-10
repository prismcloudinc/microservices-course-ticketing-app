---
service_id: ticketing-app/payments
service_name: payments
prompt_id: 06-failure-modes
prompt_label: Failure Modes & Edge Cases
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.26105500000000004
schema_version: "2.0"
---

# Error Handling & Failure Modes Analysis: Payments Service

## Part A: Error Handling Architecture

### Global Error Handler

| Attribute | Value | Source |
|-----------|-------|--------|
| **Error Middleware** | `errorHandlerMiddleware` | `src/app.ts:40` |
| **Provider** | `@b.anik/ticketing-common` npm package | `src/app.ts:9` |
| **Async Error Handling** | `express-async-errors` | `src/app.ts:2` |

**Observations:**

The global error handler is `errorHandlerMiddleware` imported from the external `@b.anik/ticketing-common` package (`src/app.ts:9-10`). The actual implementation is not present in this repository — it's an external npm package. The error handler is mounted last in the middleware chain (`src/app.ts:40`).

The service uses `express-async-errors` (`src/app.ts:2`) to automatically catch errors thrown in async route handlers, eliminating the need for explicit try-catch blocks in routes.

### Error Types Caught

Based on usage throughout the codebase, the common library provides these error classes:

| Error Class | HTTP Status | Usage Location |
|-------------|-------------|----------------|
| `NotFoundError` | 404 (inferred) | `src/routes/new.ts:30`, `src/routes/show.ts:33`, `src/app.ts:38` |
| `NotAuthorizedError` | 401 (inferred) | `src/routes/new.ts:33` |
| `BadRequestError` | 400 (inferred) | `src/routes/new.ts:36,46` |

A catch-all 404 route throws `NotFoundError` for any unmatched routes (`src/app.ts:37-39`).

### Logging

| Attribute | Value | Source |
|-----------|-------|--------|
| **Logging Library** | None — uses `console.log` | `src/index.ts:38,41,52,54,59` |
| **Log Levels Used** | `console.log` only | Observed in source |
| **Structured Logging** | No | — |

**Logging instances:**
- `src/index.ts:38` — "Connected to NATS!"
- `src/index.ts:41` — "NATS connection closed!"
- `src/index.ts:52` — "Connected to MongoDB!"
- `src/index.ts:54` — Startup errors (logs raw error object)
- `src/index.ts:59` — "Listening on port 3000!"

**Gap:** No structured logging library (winston, pino, bunyan) is used. Error logging at startup (`console.log(err)` at `src/index.ts:54`) logs the raw error without context.

### External Error Tracking

| Service | Present | Evidence |
|---------|---------|----------|
| Sentry | No | No imports or configuration found |
| Datadog | No | No imports or configuration found |
| New Relic | No | No imports or configuration found |
| Any APM | No | No evidence in codebase |

**Gap:** No external error tracking or monitoring service is configured.

---

## Part B: Integration Point Failure Behavior

### 1. MongoDB Failure

**Source:** `src/index.ts:51`

| Aspect | Behavior |
|--------|----------|
| **Detection** | Connection failure throws exception from `mongoose.connect()` |
| **Immediate response** | Exception caught in try-catch block (`src/index.ts:32-56`) |
| **Retry policy** | None — single attempt, then `process.exit(1)` (`src/index.ts:55`) |
| **Circuit breaker** | None |
| **Fallback** | None — service cannot start without MongoDB |
| **Alerting** | None — error logged to console only (`src/index.ts:54`) |
| **Recovery** | Manual — requires pod restart by Kubernetes |

**Runtime query failures:** Mongoose operations (e.g., `Order.findById`, `Payment.save()`) throw exceptions that propagate to `errorHandlerMiddleware`. No retry or fallback logic exists for individual query failures.

**Gap:** No connection pooling configuration, no retry logic for transient failures, no health check for MongoDB connection state.

### 2. NATS Streaming Server Failure

**Source:** `src/nats-wrapper.ts`, `src/index.ts:33-48`

| Aspect | Behavior |
|--------|----------|
| **Detection** | Connection emits `'error'` event; closed connection emits `'close'` event |
| **Immediate response** | `'close'` event triggers `process.exit(1)` (`src/index.ts:40-43`) |
| **Retry policy** | None — single connection attempt at startup |
| **Circuit breaker** | None |
| **Fallback** | None — service terminates on NATS disconnect |
| **Alerting** | Logs "NATS connection closed!" to console (`src/index.ts:41`) |
| **Recovery** | Kubernetes restarts pod; no in-process reconnection |

**Connection handling:**
```typescript
// src/index.ts:40-43
natsWrapper.client.on('close', () => {
  console.log('NATS connection closed!');
  process.exit(1);
});
```

**Gap:** No automatic reconnection. Any NATS disconnect causes immediate process termination.

### 3. NATS Event Publishing Failure

**Source:** `src/routes/new.ts:56-58`, publisher base class in `nats-test/src/events/base-publisher.ts`

| Aspect | Behavior |
|--------|----------|
| **Detection** | Callback receives error in `client.publish()` |
| **Immediate response** | Promise rejects with error |
| **Retry policy** | None |
| **Circuit breaker** | None |
| **Fallback** | None |
| **Dead letter queue** | None |

**Critical issue at `src/routes/new.ts:56-58`:**
```typescript
new PaymentCompletePublisher(natsWrapper.client).publish({
  orderId: payment.order.id,
});
```

The `publish()` call is **not awaited**. If publishing fails:
1. The HTTP response has already been sent (201 success)
2. The `payment:complete` event is lost
3. The error is silently swallowed (no `.catch()`)

**Gap:** Fire-and-forget event publishing can silently lose events.

### 4. NATS Event Listener Failure

**Source:** `src/events/listeners/*.ts`, base listener in `nats-test/src/events/base-listener.ts`

| Aspect | Behavior |
|--------|----------|
| **Detection** | `onMessage()` rejects Promise |
| **Immediate response** | Error logged to console, message NOT acknowledged |
| **Retry policy** | NATS redelivers unacknowledged messages after `ackWait` (5 seconds default per `nats-test/src/events/base-listener.ts:16`) |
| **Circuit breaker** | None |
| **Fallback** | None |
| **Dead letter queue** | None — NATS Streaming has no built-in DLQ |
| **Recovery** | Manual intervention required for poison messages |

**Message acknowledgment pattern (from base-listener):**
```typescript
// nats-test/src/events/base-listener.ts:47-49
this.onMessage(parsedData, msg)
  .then(() => msg.ack())
  .catch(console.error);
```

Messages are only acknowledged on successful processing. Failed messages are redelivered indefinitely.

**Gap:** No poison message handling — a permanently failing message will loop forever, blocking the queue group.

### 5. Stripe API Failure

**Source:** `src/routes/new.ts:39-47`, `src/stripe.ts`

| Aspect | Behavior |
|--------|----------|
| **Detection** | `stripe.charges.create()` throws exception, or returns `charge.paid: false` |
| **Immediate response** | Exception propagates to error handler; `paid: false` throws `BadRequestError('Payment failed')` |
| **Retry policy** | None |
| **Circuit breaker** | None |
| **Fallback** | None |
| **Alerting** | None |
| **Rate limiting** | Not handled — relies on Stripe's error responses |

**Stripe error handling at `src/routes/new.ts:39-47`:**
```typescript
const charge = await stripe.charges.create({
  currency: 'usd',
  amount: order.price * 100,
  source: token,
});

if (!charge.paid) {
  throw new BadRequestError('Payment failed');
}
```

**Stripe SDK configuration (`src/stripe.ts:1-5`):**
```typescript
export const stripe = new Stripe(process.env.STRIPE_KEY!, {
  apiVersion: '2024-04-10',
});
```

No timeout, retry, or idempotency key configured.

**Gaps:**
- No idempotency key — retries by clients could create duplicate charges
- No differentiation between card declined vs. Stripe service unavailable
- Network timeouts use Stripe SDK defaults (unknown)

---

## Part C: Data Consistency Edge Cases

### 1. Service Crash Mid-Operation

**Scenario: Crash during POST `/api/payments`**

The payment creation flow at `src/routes/new.ts:39-60`:
1. Stripe charge created (line 39-43)
2. Charge success verified (line 45-47)
3. Payment saved to MongoDB (line 49-54)
4. Event published (line 56-58)

| Crash Point | Consequence |
|-------------|-------------|
| After Stripe charge, before MongoDB save | Money charged, but no Payment record exists. Order remains in `AwaitingPayment` state indefinitely. |
| After MongoDB save, before event publish | Payment exists, but `payment:complete` event never fires. Downstream services (Orders) never learn payment succeeded. |
| During event publish (fire-and-forget) | Same as above — event silently lost. |

**Gap:** No transaction or saga pattern. No compensation logic. No idempotency protection.

### 2. Duplicate Message Processing

**Idempotency checks:** None found in codebase.

**OrderCreatedListener (`src/events/listeners/order-created-listener.ts:12-19`):**
```typescript
const order = Order.build({ id: data.id, ... });
await order.save();
```

If the same `order:created` event is delivered twice:
- Second `order.save()` will fail with MongoDB duplicate key error on `_id`
- Listener throws, message not acknowledged, redelivered indefinitely

**OrderCancelledListener / OrderUpdatedListener:**
Uses `findByEvent({ id, version: version - 1 })` which provides optimistic concurrency protection.

**Payment creation (`src/routes/new.ts`):**
No duplicate payment check. Same orderId + token submitted twice could create two Stripe charges and two Payment records.

**Gap:** No idempotency keys for payments. Duplicate events can cause duplicate charges.

### 3. Workflow Activity Retry

**Event listeners use manual version checking:**

`src/models/order.ts:68-76`:
```typescript
orderSchema.statics.findByEvent = async (event: {
  id: string;
  version: number;
}): Promise<OrderDoc | null> => {
  return Order.findOne({
    _id: event.id,
    version: event.version - 1,
  });
};
```

This implements optimistic concurrency — updates only succeed if the current version is exactly `version - 1`. Out-of-order events will fail to find the document and throw "Order not found".

**Gap:** Out-of-order event delivery causes listener failures and message redelivery loops, not graceful handling.

### 4. Concurrent Modifications

**Payment creation:** Two simultaneous requests for the same order:

Both requests reach `Order.findById(orderId)` (`src/routes/new.ts:27`):
- Both find order in `AwaitingPayment` status
- Both call `stripe.charges.create()` — two charges created
- Both save Payment records — two payments for one order

**Gap:** No optimistic locking on the POST `/api/payments` endpoint. Race condition allows double-charging.

### 5. Stale Data from Cache

No caching layer observed in the payments service. MongoDB is queried directly on each request.

---

## Part D: Known Gotchas

### TODO/FIXME/HACK/WORKAROUND Comments

**None found.** Grep for `TODO|FIXME|HACK|WORKAROUND|XXX|BUG` returned no matches in `services/payments/**/*.ts`.

### Hardcoded Values

| Location | Value | Issue |
|----------|-------|-------|
| `src/index.ts:58` | `3000` | HTTP port hardcoded, should be configurable via env var |
| `src/stripe.ts:4` | `'2024-04-10'` | Stripe API version hardcoded — pinned is correct, but version bumps require code change |
| `src/events/listeners/queue-group-name.ts:1` | `'payments-service'` | Queue group name hardcoded — acceptable, matches service name |

### Timeout Values

| Component | Timeout | Source | Assessment |
|-----------|---------|--------|------------|
| NATS ackWait | 5 seconds | `nats-test/src/events/base-listener.ts:16` (inferred for common lib) | Short — complex event handlers may exceed this |
| NATS heartbeat | 5s interval, 5s timeout, 2 failures | `infra/k8s/nats-depl.yaml:19-29` (`-hbi 5s -hbt 5s -hbf 2`) | Reasonable for failure detection |
| MongoDB | Default (unset) | No configuration found | Unknown — relies on Mongoose/driver defaults |
| Stripe | Default (unset) | No configuration found | Unknown — relies on Stripe SDK defaults |
| HTTP server | None | No `server.setTimeout()` call | Relies on Node.js defaults |

**Kubernetes liveness probe (`infra/k8s/payments-depl.yaml:39-44`):**
- Initial delay: 5 seconds
- Period: 5 seconds
- Path: `/api/payments/health`

Assessment: 5-second period is aggressive — may cause unnecessary restarts during GC pauses or brief NATS hiccups.

### Rate Limits on External APIs

| API | Rate Limit | Handling | Source |
|-----|------------|----------|--------|
| Stripe | Yes (per Stripe docs) | Not handled — no retry-after parsing, no backoff | No evidence in code |

**Gap:** Stripe rate limiting (HTTP 429) will propagate as unhandled errors.

### Data Size Limits

| Component | Limit | Source | Handling |
|-----------|-------|--------|----------|
| Express body size | Default (~100kb) | No `json({ limit: ... })` in `src/app.ts:19` | Requests exceeding limit return 413 |
| NATS message size | 1MB default | NATS server configuration | Not validated in publishers |
| MongoDB document size | 16MB | MongoDB limit | Not validated in application |

### Missing `NATS_CLIENT_ID` Validation

**`src/index.ts:28-30`:**
```typescript
if (!process.env.NATS_CLUSTER_ID) {
  throw new Error('NATS_CLUSTER_ID must be defined');
}
```

This check is duplicated (same as lines 24-26) and **`NATS_CLIENT_ID` is never validated**, yet it's used at line 35.

### Graceful Shutdown Issues

**Signal handlers (`src/index.ts:45-47`):**
```typescript
process.on('SIGINT', () => natsWrapper.client.close());
process.on('SIGTERM', () => natsWrapper.client.close());
process.on('SIGUSR2', () => natsWrapper.client.close());
```

When NATS client closes, the `'close'` event handler calls `process.exit(1)` (`src/index.ts:42`).

**Issues:**
1. Closing NATS triggers exit code 1 (error) even during graceful shutdown
2. HTTP server is not closed — in-flight requests may be dropped
3. MongoDB connection is not closed

### Event Publishing Not Awaited

**`src/routes/new.ts:56-58`:**
```typescript
new PaymentCompletePublisher(natsWrapper.client).publish({
  orderId: payment.order.id,
});
```

**`src/events/listeners/order-created-listener.ts:21-23`:**
```typescript
await new PaymentAwaitedPublisher(this.client).publish({
  orderId: order.id,
});
```

The `PaymentCompletePublisher` in the route is **not awaited**, while `PaymentAwaitedPublisher` in the listener **is awaited**. Inconsistent pattern — the route silently loses publish failures.

### No Index on Payment.userId

**`src/models/payment.ts`:**
The schema defines `userId: { type: String, required: true }` but no index.

`GET /api/payments` (`src/routes/show.ts:14-15`) queries:
```typescript
const payments = await Payment.find({ userId: req.currentUser!.id })
```

**Gap:** Missing index on `userId` will cause full collection scans as payment volume grows.

### Version Not Incremented on Order Updates

**`src/events/listeners/order-cancelled-listener.ts:17-20`:**
```typescript
order.set({
  status: OrderStatus.Cancelled,
  version: data.version,
});
await order.save();
```

The version is set from the incoming event, not auto-incremented. This assumes the source service correctly increments versions. If the Payments service ever becomes a source of truth (unlikely but possible in future changes), this pattern would break optimistic concurrency.
