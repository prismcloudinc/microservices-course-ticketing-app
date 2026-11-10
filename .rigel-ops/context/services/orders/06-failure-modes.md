---
service_id: ticketing-app/orders
service_name: orders
prompt_id: 06-failure-modes
prompt_label: Failure Modes & Edge Cases
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.26530000000000004
schema_version: "2.0"
---

# Error Handling & Failure Mode Analysis: Orders Service

## Part A: Error Handling Architecture

### Global Error Handler

The global error handler is `errorHandlerMiddleware` imported from the `@b.anik/ticketing-common` npm package (`services/orders/src/app.ts`, line 9 and 41). This middleware is registered last in the middleware chain.

**Source Code Reference:**
```typescript
// services/orders/src/app.ts, line 41
app.use(errorHandlerMiddleware);
```

The actual implementation is in the external `@b.anik/ticketing-common` package (not available in this codebase). Based on usage patterns observed in route tests, it:
- Catches thrown errors from route handlers
- Formats errors into a standardized JSON response with an `errors` array
- Each error object contains `message` and optionally `field` properties
- Maps error types to HTTP status codes (400 for `BadRequestError`, 401 for `NotAuthorizedError`, 404 for `NotFoundError`)

**Error Types Used:**
| Error Type | HTTP Status | Usage Locations |
|-----------|-------------|-----------------|
| `NotFoundError` | 404 | `app.ts:39`, `show.ts:33`, `cancel.ts:31` |
| `BadRequestError` | 400 | `new.ts:33,39`, `cancel.ts:39,43` |
| `NotAuthorizedError` | 401 | `show.ts:37`, `cancel.ts:35` |

**Async Error Handling:**
The service uses `express-async-errors` (`app.ts`, line 2) to enable throwing errors from async route handlers without explicit try-catch or next() calls.

### Logging

**Logging Library:** None - only basic `console.log()` is used.

**Log Statements in the Service:**
| File:Line | Message | Level |
|-----------|---------|-------|
| `index.ts:34` | `'Connected to NATS!'` | info (console.log) |
| `index.ts:37` | `'NATS connection closed!'` | info (console.log) |
| `index.ts:48` | `'Connected to MongoDB!'` | info (console.log) |
| `index.ts:50` | `console.log(err)` | error (console.log) |
| `index.ts:55` | `'Listening on port 3000!'` | info (console.log) |

**Missing Logging:**
- No structured logging with fields (service name, trace ID, timestamp)
- No log levels (DEBUG, INFO, WARN, ERROR)
- No logging of incoming requests or outgoing responses
- No logging of event listener processing or errors
- No logging correlation IDs for distributed tracing

### External Error Tracking

**None detected.** No error monitoring service (e.g., Sentry, Datadog, New Relic, etc.) is configured. Errors are only written to stdout via console.log.

---

## Part B: Integration Point Failure Behavior

### MongoDB Failure

**Connection Mechanism:** `services/orders/src/index.ts`, lines 47-51
```typescript
await mongoose.connect(process.env.MONGO_URI!);
```

| Aspect | Behavior |
|--------|----------|
| **Detection** | Promise rejection from `mongoose.connect()` throws, caught in try-catch block |
| **Immediate Response** | Error logged to console, process exits with code 1 (`index.ts:50-51`) |
| **Retry Policy** | None at application level; relies on Mongoose driver defaults |
| **Circuit Breaker** | None |
| **Fallback** | None — process terminates |
| **Alerting** | None — Kubernetes liveness probe failure would eventually trigger pod restart and generate events |
| **Dead Letter Queue** | N/A |
| **Recovery** | Kubernetes restarts the pod; manual intervention required if MongoDB is down persistently |

**Runtime MongoDB Errors:**
- Query/save operations in route handlers have no explicit error handling
- Errors propagate up through `express-async-errors` to `errorHandlerMiddleware`
- Database errors during event listener processing cause the promise to reject, message is not acknowledged, NATS will redeliver

### NATS Streaming Failure

**Connection Mechanism:** `services/orders/src/nats-wrapper.ts`, lines 15-23 and `index.ts`, lines 29-44

| Aspect | Behavior |
|--------|----------|
| **Detection** | `'error'` event on NATS client triggers promise rejection; `'close'` event triggers process exit |
| **Immediate Response** | Connection error → logged and process exits (`index.ts:50-51`); Runtime close → process exits (`index.ts:36-39`) |
| **Retry Policy** | None at application level |
| **Circuit Breaker** | None |
| **Fallback** | None — process terminates |
| **Alerting** | None |
| **Dead Letter Queue** | None configured; NATS Streaming has its own redelivery mechanism |
| **Recovery** | Kubernetes restarts the pod |

**NATS Client Configuration (from `nats-depl.yaml`):**
- Heartbeat interval: 5s (`-hbi 5s`)
- Heartbeat timeout: 5s (`-hbt 5s`)
- Heartbeat fail count: 2 (`-hbf 2`)

**Message Processing Errors:**
Based on the base listener pattern from `nats-test/src/events/base-listener.ts`, lines 40-49:
```typescript
subscription.on('message', (msg: Message) => {
  // ...
  this.onMessage(parsedData, msg)
    .then(() => msg.ack())
    .catch(console.error);
});
```
- If `onMessage()` resolves → message is acknowledged
- If `onMessage()` rejects → error is logged via `console.error`, message is **not acknowledged**
- NATS will redeliver after `ackWait` timeout (default 5 seconds, `base-listener.ts`, line 17)

**Event Publishing Errors:**
From `nats-test/src/events/base-publisher.ts`, lines 14-26:
- Publishing is Promise-based
- If NATS client callback returns error, promise rejects
- **Critical Gap:** In route handlers, `publish()` calls are **not awaited** (see Part D finding)

### No Other Integration Points Detected

The following are **not present** in this service:
- HTTP clients to external services
- Redis/cache layer
- Payment provider integration (payments handled by separate service)
- Cloud services directly (e.g., S3, SQS)
- Third-party APIs

---

## Part C: Data Consistency Edge Cases

### Service Crash Mid-Operation

**Order Creation (`new.ts`, lines 47-66):**
```typescript
await order.save();   // DB write completes

// CRASH POINT — if crash occurs here:

new OrderCreatedPublisher(natsWrapper.client).publish({...}); // Event may never publish
```

**Risk:** Order saved to MongoDB but `order:created` event never published. Other services (expiration, payments) never learn about the order.

**Impact:** 
- Ticket remains logically reserved but expiration timer never starts
- User cannot purchase ticket, but ticket appears unavailable to others indefinitely

**Order Cancellation (`cancel.ts`, lines 46-56):** Same pattern — database write followed by event publish with crash risk between.

**Event Listeners (`expiration-complete-listener.ts`, lines 24-34):**
```typescript
await order.save();  // Status changed to Cancelled

// CRASH POINT

await new OrderCancelledPublisher(this.client).publish({...});
```
**Risk:** Order cancelled in DB but downstream services not notified.

### Duplicate Message Processing (Idempotency)

**TicketCreatedListener (`ticket-created-listener.ts`):**
- Creates a new ticket document with `Ticket.build()` and `ticket.save()`
- **No idempotency check** — processing the same event twice throws a MongoDB duplicate key error on `_id`
- The error rejects the promise, causing NATS to redeliver infinitely until manual intervention

**PaymentAwaitedListener / PaymentCompleteListener:**
- Look up order by ID and update status
- If already in target status, updates still occur (no short-circuit)
- Generally safe but wastes resources

**ExpirationCompleteListener (`expiration-complete-listener.ts`, lines 20-23):**
```typescript
if (order.status === OrderStatus.Cancelled || order.status === OrderStatus.Complete) {
  return resolve();  // Safe — idempotent for terminal states
}
```
**This is correct** — demonstrates awareness of the issue but pattern is not consistently applied.

### Workflow Activity Retry Safety

**TicketUpdatedListener (`ticket-updated-listener.ts`, lines 13-24):**
```typescript
const ticket = await Ticket.findByEvent(data);  // Requires version - 1
if (!ticket) {
  throw new Error('Ticket not found');
}
ticket.set({ title: data.title, price: data.price, version: data.version });
await ticket.save();
```

**Safe for retries:** Uses version-aware lookup via `Ticket.findByEvent()` (`ticket.ts`, lines 62-70):
```typescript
ticketSchema.statics.findByEvent = async (event) => {
  return Ticket.findOne({
    _id: event.id,
    version: event.version - 1,  // Requires previous version
  });
};
```
If event is retried after successful processing, `findByEvent` returns null because version no longer matches, causing message to be rejected (not acked) and redelivered forever.

**Design Issue:** Out-of-order events cause permanent rejection. If version 2 arrives before version 1, version 2 is rejected repeatedly until version 1 is processed.

### Concurrent Modifications (Race Conditions)

**Optimistic Concurrency Control:**
The Order model uses `mongoose-update-if-current` plugin (`order.ts`, lines 57-58):
```typescript
orderSchema.set('versionKey', 'version');
orderSchema.plugin(updateIfCurrentPlugin);
```

**Behavior:** If two concurrent saves attempt to modify the same Order document, the second save fails with a version conflict error.

**Test Evidence (`order.test.ts`, lines 6-34):**
```typescript
await expect(trySecondInstanceSave).rejects.toThrow(/version 0/i);
```

**Gap:** Version conflicts bubble up as unhandled errors. No retry logic wraps the save operation.

**Ticket Reservation Race (`new.ts`, lines 31-40):**
```typescript
const ticket = await Ticket.findById(ticketId);
// ...
const isReserved = await ticket.isReserved();
if (isReserved) {
  throw new BadRequestError('Ticket is already reserved');
}
// RACE WINDOW: Another request could reserve here
const order = Order.build({...});
await order.save();
```

**Risk:** Two users requesting the same ticket simultaneously could both pass `isReserved()` check and create two orders for the same ticket.

**Mitigation (Partial):** The `isReserved()` method checks for existing orders, but the check and create are not atomic.

### Stale Data

**Local Ticket Copy:**
The Orders service maintains a local copy of tickets via event listeners. If `ticket:updated` events are delayed or lost:
- Orders could be created with stale ticket prices
- Price discrepancy between services

**No cache invalidation TTL** — tickets are updated only via events.

---

## Part D: Known Gotchas

### Critical Bugs

#### 1. PaymentAwaitedListener Swallows Errors
**File:** `services/orders/src/events/listeners/payment-awaited-listener.ts`  
**Line:** 29

```typescript
} catch (error) {
  reject;  // BUG: should be reject(error)
}
```

**Impact:** Errors during payment awaited processing are swallowed. The promise never resolves or rejects, causing the listener to hang. NATS will redeliver after ack timeout, potentially causing infinite redelivery or resource exhaustion.

**Fix Required:** Change to `reject(error);`

#### 2. Event Publishing Not Awaited
**File:** `services/orders/src/routes/new.ts`  
**Lines:** 56-66

```typescript
new OrderCreatedPublisher(natsWrapper.client).publish({...});  // Not awaited

return res.status(201).send({ order });
```

**Also in:**
- `services/orders/src/routes/cancel.ts`, lines 50-56

**Impact:** 
- Response sent to client before event is published
- If publish fails, client receives success but downstream services never notified
- Fire-and-forget pattern masks NATS failures

#### 3. Duplicate Environment Variable Check
**File:** `services/orders/src/index.ts`  
**Lines:** 20-26

```typescript
if (!process.env.NATS_CLUSTER_ID) {
  throw new Error('NATS_CLUSTER_ID must be defined');
}

if (!process.env.NATS_CLUSTER_ID) {  // Duplicate check
  throw new Error('NATS_CLUSTER_ID must be defined');
}
```

**Impact:** `NATS_CLIENT_ID` is never validated despite being used on line 31. If missing, `nats.connect()` receives `undefined`, causing runtime failure.

### Hardcoded Values

| Value | File:Line | Description | Should Be |
|-------|-----------|-------------|-----------|
| `15 * 60` (900 seconds) | `new.ts:16` | Order expiration window | Environment variable |
| `3000` | `index.ts:54-55` | HTTP server port | Environment variable |
| `5 * 1000` (5 seconds) | `base-listener.ts:17` | NATS ack wait timeout | Environment variable |
| `'orders-service'` | `queue-group-name.ts:1` | NATS queue group | Environment variable or derived from service name |

### Timeout Values

| Timeout | Location | Value | Assessment |
|---------|----------|-------|------------|
| NATS ack wait | `base-listener.ts:17` | 5 seconds | May be too short for DB operations under load |
| Order expiration | `new.ts:16` | 15 minutes | Reasonable for ticket purchase flow |
| K8s liveness probe | `orders-depl.yaml:38-39` | 5s initial, 5s period | Aggressive — may false-positive during slow startups |

### Missing Validations

| Gap | File:Line | Impact |
|-----|-----------|--------|
| `validateRequestMiddleware` not called | `cancel.ts:20-25` | `orderId` validation defined but middleware not used — validation never runs |
| No body size limit | `app.ts:19` | `json()` middleware has no limit option; vulnerable to large payload attacks |

### No Rate Limits

External API rate limits are not applicable (no external HTTP calls). However:
- No rate limiting on incoming HTTP requests
- No rate limiting on NATS event processing
- MongoDB connection pool not configured (using defaults)

### Data Size Limits

| Concern | Status |
|---------|--------|
| Request body size | Unlimited (no `limit` option on `json()` middleware) |
| MongoDB document size | 16MB (MongoDB default) — not enforced in schema |
| NATS message size | Default 1MB — not configured |
| Mongoose field lengths | No `maxlength` validators on string fields (`title` in Ticket) |

### Missing Database Indexes

No custom indexes defined in models (`order.ts`, `ticket.ts`). The following queries would benefit from indexes:

| Query | File:Line | Suggested Index |
|-------|-----------|-----------------|
| `Order.find({ userId })` | `show.ts:10-12` | `{ userId: 1 }` |
| `Order.findOne({ ticket, status: {$in: [...]} })` | `ticket.ts:77-82` | `{ ticket: 1, status: 1 }` |
| `Ticket.findOne({ _id, version })` | `ticket.ts:66-69` | Compound `{ _id: 1, version: 1 }` |

---

## Summary of Critical Gaps

| Priority | Issue | Location | Risk Level |
|----------|-------|----------|------------|
| P0 | PaymentAwaitedListener swallows errors (`reject` instead of `reject(error)`) | `payment-awaited-listener.ts:29` | High — silent data loss |
| P0 | NATS_CLIENT_ID not validated | `index.ts:20-26` | High — startup crash on missing env var |
| P1 | Event publishing not awaited | `new.ts:56`, `cancel.ts:50` | Medium — inconsistent data between services |
| P1 | No idempotency in TicketCreatedListener | `ticket-created-listener.ts` | Medium — infinite retry on duplicate |
| P1 | Ticket reservation race condition | `new.ts:31-40` | Medium — double booking possible |
| P2 | No structured logging | Entire service | Medium — debugging/monitoring difficulty |
| P2 | No external error tracking | Entire service | Medium — silent failures in production |
| P2 | validateRequestMiddleware not used in cancel route | `cancel.ts:25` | Low — parameter injection possible |
| P3 | Hardcoded port and expiration window | `index.ts:54`, `new.ts:16` | Low — inflexible deployments |
