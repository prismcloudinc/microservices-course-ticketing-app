---
service_id: ticketing-app/tickets
service_name: tickets
prompt_id: 06-failure-modes
prompt_label: Failure Modes & Edge Cases
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.260785
schema_version: "2.0"
---

# Error Handling Analysis: Tickets Service

## Part A: Error Handling Architecture

### Global Error Handler

**Location**: Error handling is provided by `errorHandlerMiddleware` from the external package `@b.anik/ticketing-common` (version 0.1.2).

**Registration**: `services/tickets/src/app.ts:48`
```typescript
app.use(errorHandlerMiddleware);
```

**Error Types Caught**:
- The service uses `express-async-errors` (`services/tickets/src/app.ts:2`) which patches Express to automatically catch async errors without explicit try/catch blocks.
- Custom error classes from `@b.anik/ticketing-common`:
  - `NotFoundError` — used for 404 responses (`services/tickets/src/routes/show.ts:3`, `services/tickets/src/routes/update.ts:4`, `services/tickets/src/routes/delete.ts:4`, `services/tickets/src/app.ts:46`)
  - `NotAuthorizedError` — used for 401 responses (`services/tickets/src/routes/update.ts:3`, `services/tickets/src/routes/delete.ts:3`)
  - `BadRequestError` — used for 400 responses (`services/tickets/src/routes/update.ts:2`, `services/tickets/src/routes/delete.ts:2`)

**Error Response Transformation**: Unknown — the `errorHandlerMiddleware` is from an external package whose source is not available in this repository. Based on test assertions (e.g., `services/tickets/src/routes/__tests__/update.test.ts:114`), errors are transformed into a response with an `errors` array containing objects with a `message` field.

### Logging

**Library**: Native `console` object — no structured logging library is used.

**Log Levels**: Only `console.log()` is used:
- `services/tickets/src/index.ts:34` — "Connected to NATS!"
- `services/tickets/src/index.ts:37` — "NATS connection closed!"
- `services/tickets/src/index.ts:48` — "Connected to MongoDB!"
- `services/tickets/src/index.ts:50` — Error object on startup failure
- `services/tickets/src/index.ts:55` — "Listening on port 3000!"

**Structured Fields**: None. All logging is unstructured plaintext or raw error objects.

### External Error Tracking

**Status**: None detected. No Sentry, Datadog, New Relic, Bugsnag, or similar error monitoring services are configured in the codebase.

---

## Part B: Integration Point Failure Behavior

### MongoDB Failure

**Connection Location**: `services/tickets/src/index.ts:47`

**Detection**:
- Connection errors are caught in the try/catch block at `services/tickets/src/index.ts:49-52`
- Runtime query failures (e.g., `Ticket.findById()`) throw errors handled by `express-async-errors` and the global error middleware

**Immediate Response**:
- Connection failure: Error logged via `console.log(err)`, process exits with code 1 (`services/tickets/src/index.ts:50-51`)
- Runtime query failure: Error propagates to `errorHandlerMiddleware`

**Retry Policy**: None. No retry logic for MongoDB connections or queries.

**Circuit Breaker**: None implemented.

**Fallback**: None. No degraded behavior or cached responses on database failure.

**Alerting**: None. Kubernetes liveness probe (`infra/k8s/tickets-depl.yaml:34-39`) will detect if the service crashes, but no proactive alerting is configured.

**Dead Letter Queue**: Not applicable for database operations.

**Recovery**: Manual restart via Kubernetes pod replacement after connection restoration.

---

### NATS Streaming Failure

**Connection Location**: `services/tickets/src/index.ts:29-33` via `NatsWrapper.connect()` at `services/tickets/src/nats-wrapper.ts:15-23`

**Detection**:
- Connection errors trigger the `error` event handler (`services/tickets/src/nats-wrapper.ts:21`)
- Connection close triggers the `close` event handler (`services/tickets/src/index.ts:36-39`)

**Immediate Response**:
- Connection failure: Promise rejects, error logged, process exits with code 1 (`services/tickets/src/index.ts:49-51`)
- Connection closed during runtime: Process exits with code 1 (`services/tickets/src/index.ts:38`)

**Retry Policy**: None. No retry logic for NATS connections.

**Circuit Breaker**: None implemented.

**Fallback**: None. Service crashes on NATS connection failure.

**Alerting**: None configured in the application. Depends on Kubernetes pod restart detection.

**Graceful Shutdown**:
- SIGINT handler: `services/tickets/src/index.ts:41`
- SIGTERM handler: `services/tickets/src/index.ts:42`
- SIGUSR2 handler: `services/tickets/src/index.ts:43`

All handlers close the NATS client, which triggers the `close` event and exits the process.

---

### Event Publishing Failure (`TicketCreatedPublisher`, `TicketUpdatedPublisher`)

**Locations**:
- `services/tickets/src/routes/new.ts:28-34`
- `services/tickets/src/routes/update.ts:49-55`
- `services/tickets/src/events/listeners/order-created-listener.ts:26-33`
- `services/tickets/src/events/listeners/order-cancelled-listener.ts:31-38`

**Detection**: Publisher classes extend `Publisher` from `@b.anik/ticketing-common`. Error handling within the base class is unknown (source not available).

**Immediate Response**: `await` is used on `.publish()`, so errors would propagate up. In HTTP routes, this would result in a 500 error after the database write has already succeeded (see Part C for consistency implications).

**Retry Policy**: Unknown — depends on the external `Publisher` base class implementation.

**Circuit Breaker**: None visible in this service's code.

**Fallback**: None.

**Dead Letter Queue**: None configured in this service.

---

### Event Listening Failure (`OrderCreatedListener`, `OrderCancelledListener`)

**Locations**:
- `services/tickets/src/events/listeners/order-created-listener.ts:11-40`
- `services/tickets/src/events/listeners/order-cancelled-listener.ts:11-45`

**Detection**:
- Both listeners wrap their logic in a `Promise` that resolves on success or rejects on error
- `Ticket not found` error is thrown explicitly when the ticket doesn't exist (`order-created-listener.ts:20-21`, `order-cancelled-listener.ts:24-25`)

**Immediate Response**:
- Promise rejection — behavior depends on the base `Listener` class from `@b.anik/ticketing-common` (source not available)
- Tests confirm errors cause rejection (`services/tickets/src/events/listeners/__tests__/order-created-listener.test.ts:43-52`)

**Retry Policy**: Unknown — depends on the external `Listener` base class. NATS Streaming supports manual ack mode and redelivery, but whether this is configured is not visible.

**Message Acknowledgment**: The `Message` parameter is received but not explicitly used in the listener code (`_msg` prefix indicates unused). Ack handling is likely in the base `Listener` class.

**Dead Letter Queue**: None configured in this service.

**Queue Group**: `tickets-service` (`services/tickets/src/events/listeners/queue-group-name.ts:1`) — ensures only one instance processes each message.

---

## Part C: Data Consistency Edge Cases

### Mid-Operation Crash (Partial Writes)

**Scenario**: Service crashes between database write and event publish.

**HTTP Route Risk** (`POST /api/tickets`, `PUT /api/tickets/:id`):
1. `ticket.save()` succeeds (database modified)
2. Crash occurs before `publisher.publish()` completes
3. **Result**: Ticket exists in database but `ticket:created` or `ticket:updated` event never published

**Impact**: Other services (orders, etc.) won't know about the ticket or its update.

**Mitigation**: None. No outbox pattern, no transactional outbox, no saga coordinator.

**Event Listener Risk** (`OrderCreatedListener`, `OrderCancelledListener`):
1. `ticket.save()` succeeds (orderId set/cleared)
2. Crash occurs before `TicketUpdatedPublisher.publish()` completes
3. **Result**: Ticket reservation state updated locally but `ticket:updated` event not published

**Impact**: Orders service won't receive confirmation of ticket reservation status.

---

### Duplicate Message Processing (Idempotency)

**Status**: No explicit idempotency checks observed.

**OrderCreatedListener** (`services/tickets/src/events/listeners/order-created-listener.ts:17-24`):
- Simply finds ticket by ID and sets `orderId`
- If processed twice with same data, result is the same (idempotent by accident due to overwrite behavior)
- However, `ticket:updated` event will be published twice with incremented version numbers

**OrderCancelledListener** (`services/tickets/src/events/listeners/order-cancelled-listener.ts:19-22`):
- Uses `findOne` with both `_id` and `orderId` match
- If processed twice, second execution will fail with "Ticket not found" because `orderId` was already cleared
- **Risk**: This causes unnecessary error logs or potential message stuck in queue (depending on base class behavior)

---

### Workflow Activity Retry (Side Effect Safety)

**Event Listeners**: Both listeners perform two side effects:
1. Database update (`ticket.save()`)
2. Event publish (`TicketUpdatedPublisher.publish()`)

On retry after partial completion:
- If database update succeeded but publish failed: retry will update database again (incrementing version) and publish
- Version numbers will be incorrect relative to actual data state
- **Risk**: Version mismatch can cause optimistic concurrency failures in downstream services

---

### Concurrent Modification (Race Conditions)

**Optimistic Concurrency Control**: Implemented via `mongoose-update-if-current` plugin (`services/tickets/src/models/ticket.ts:56`)

**Behavior** (tested in `services/tickets/src/models/__tests__/ticket.test.ts:5-26`):
- Concurrent modifications to the same ticket will fail with a version mismatch error
- The second save attempt throws an error matching `/version 0/i`

**HTTP Endpoint Protection**:
- `PUT /api/tickets/:id` (`services/tickets/src/routes/update.ts`): Uses `findById` then `save()` — protected by OCC
- `DELETE /api/tickets/:id` (`services/tickets/src/routes/delete.ts:34-37`): Uses `findOneAndDelete` with both `_id` and `userId` — single atomic operation

**Event Listener Protection**:
- Both listeners use `findById` then `save()` — protected by OCC
- If two events try to modify the same ticket simultaneously, one will fail with version error

**Gap**: When OCC conflict occurs, the operation simply fails with an error. No automatic retry logic to re-fetch and retry the operation.

---

### Stale Data (Cache Invalidation)

**Status**: No caching layer detected. All reads go directly to MongoDB. This eliminates cache invalidation concerns but means every request hits the database.

---

## Part D: Known Gotchas

### TODO/FIXME/HACK/WORKAROUND Comments

**None found** in `services/tickets/` source files.

---

### Hardcoded Values

| Value | Location | Issue |
|-------|----------|-------|
| Port `3000` | `services/tickets/src/index.ts:54-55` | Should be configurable via environment variable |
| MongoDB database name `tickets` | `infra/k8s/tickets-depl.yaml:25` | Embedded in connection string |
| NATS cluster ID `ticketing` | `infra/k8s/tickets-depl.yaml:29` | Hardcoded in deployment config |
| NATS heartbeat intervals | `infra/k8s/nats-depl.yaml:25-28` | `-hbi 5s -hbt 5s -hbf 2` hardcoded in args |
| Liveness probe timing | `infra/k8s/tickets-depl.yaml:38-39` | `initialDelaySeconds: 5`, `periodSeconds: 5` |
| User ID in tests | `services/tickets/src/test/utils.ts:5`, `services/tickets/src/routes/__tests__/*.test.ts` | `user-id-123` used across multiple test files |

---

### Timeout Values

| Component | Timeout | Location | Assessment |
|-----------|---------|----------|------------|
| NATS heartbeat interval | 5 seconds | `infra/k8s/nats-depl.yaml:25` | Reasonable for local cluster |
| NATS heartbeat timeout | 5 seconds | `infra/k8s/nats-depl.yaml:26` | Reasonable for local cluster |
| NATS heartbeat fail count | 2 | `infra/k8s/nats-depl.yaml:27` | 10 seconds to detect failure |
| Liveness probe period | 5 seconds | `infra/k8s/tickets-depl.yaml:39` | Aggressive, may cause unnecessary restarts under load |
| MongoDB connection | None configured | `services/tickets/src/index.ts:47` | Uses Mongoose defaults |
| HTTP request body parsing | None configured | `services/tickets/src/app.ts:25` | Uses body-parser defaults (100kb limit) |

**Missing Timeouts**:
- No connection timeout for MongoDB
- No query timeout for MongoDB operations
- No publish timeout for NATS events
- No HTTP request timeout at application level (depends on ingress/load balancer)

---

### Rate Limits

**Status**: No rate limiting implemented at the application level.

**External Dependencies**:
- No external APIs are called (this is an internal microservice)
- MongoDB has no rate limiting
- NATS Streaming has no rate limiting configured

---

### Data Size Limits

| Constraint | Limit | Source | Risk |
|------------|-------|--------|------|
| HTTP request body | 100KB (body-parser default) | `services/tickets/src/app.ts:25` | Not explicitly configured |
| Ticket title | No max length | `services/tickets/src/models/ticket.ts:24-26` | Could store arbitrarily long titles |
| Ticket price | `isFloat({ gt: 0 })` | `services/tickets/src/routes/new.ts:15`, `services/tickets/src/routes/update.ts:21` | No upper bound |
| MongoDB document size | 16MB (MongoDB limit) | N/A | Unlikely to hit with ticket data |
| NATS message size | 1MB default | NATS Streaming config | Event payloads should stay small |

---

### Duplicate Environment Variable Check

**Location**: `services/tickets/src/index.ts:20-26`

```typescript
if (!process.env.NATS_CLUSTER_ID) {
  throw new Error('NATS_CLUSTER_ID must be defined');
}

if (!process.env.NATS_CLUSTER_ID) {
  throw new Error('NATS_CLUSTER_ID must be defined');
}
```

**Issue**: `NATS_CLUSTER_ID` is checked twice. The second check should likely be `NATS_CLIENT_ID`.

**Impact**: `NATS_CLIENT_ID` is not validated at startup. If missing, `natsWrapper.connect()` will be called with `undefined` as the client ID.

---

### Missing Environment Variable Validation

**NATS_CLIENT_ID**: As noted above, this variable is used (`services/tickets/src/index.ts:31`) but never validated. The duplicate `NATS_CLUSTER_ID` check suggests this was an oversight.

---

### Cookie Session Security

**Location**: `services/tickets/src/app.ts:27-30`

```typescript
cookieSession({
  signed: false,
  secure: process.env.NODE_ENV !== 'test',
})
```

**Observations**:
- `signed: false` — cookies are not signed, relying on JWT internal signature
- `secure` is false in test environment — expected behavior
- No `maxAge` configured — session cookies expire when browser closes
- No `httpOnly` explicitly set — defaults to `true` which is secure
- No `sameSite` configured — defaults may vary by browser

---

### Missing Readiness Probe

**Location**: `infra/k8s/tickets-depl.yaml`

**Issue**: Only `livenessProbe` is configured (lines 34-39). No `readinessProbe` is defined.

**Impact**: Kubernetes may route traffic to pods that are starting up but haven't yet connected to NATS or MongoDB. The service may return 500 errors until connections are established.

---

### Trust Proxy Setting

**Location**: `services/tickets/src/app.ts:22`

```typescript
app.set('trust proxy', true);
```

**Issue**: Unconditionally trusts proxy headers. In some deployment scenarios, this could allow IP spoofing if the service is accidentally exposed directly.

---

### Event Publishing After Database Write (No Transactional Outbox)

**Locations**:
- `services/tickets/src/routes/new.ts:26-34`
- `services/tickets/src/routes/update.ts:47-55`
- `services/tickets/src/events/listeners/order-created-listener.ts:24-33`
- `services/tickets/src/events/listeners/order-cancelled-listener.ts:29-38`

**Pattern**:
```typescript
const savedTicket = await ticket.save();   // Database write
await publisher.publish({...});             // Event publish
```

**Issue**: If publish fails after save succeeds, data inconsistency occurs. The Transactional Outbox pattern would address this by writing the event to a database table in the same transaction, then having a separate process publish from the outbox.

---

### Delete Operation Without Event

**Location**: `services/tickets/src/routes/delete.ts:34-39`

**Issue**: When a ticket is deleted, no `ticket:deleted` event is published. Other services that have cached or replicated ticket data will not know the ticket was deleted.

**Code**:
```typescript
const deletedTicket = await Ticket.findOneAndDelete({
  _id: ticket.id,
  userId: req.currentUser!.id,
});

return res.status(200).send({ ticket: deletedTicket });
// No event publication
```

---

### Listener Promise Anti-Pattern

**Locations**:
- `services/tickets/src/events/listeners/order-created-listener.ts:12`
- `services/tickets/src/events/listeners/order-cancelled-listener.ts:15`

**Pattern**:
```typescript
return new Promise<void>(async (resolve, reject) => {
  try {
    // async operations
    resolve();
  } catch (err) {
    reject(err);
  }
});
```

**Issue**: This is the "explicit promise construction antipattern." Since `onMessage` is already an async function, the outer Promise wrapper is unnecessary and adds complexity. The inner `async` function means exceptions thrown before awaits won't be caught properly.

**Better Pattern**:
```typescript
async onMessage(data, msg): Promise<void> {
  // async operations directly
}
```
