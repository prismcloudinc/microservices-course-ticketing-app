---
service_id: ticketing-app/expiration
service_name: expiration
prompt_id: 03-message-event-contracts
prompt_label: Message & Event Contracts
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.17833000000000002
schema_version: "2.0"
---

# Events and Message Contracts: expiration service

## Part A: Events/Messages PRODUCED by this service

### Event: order:created → expiration:complete (NATS Streaming)

#### Event: `expiration:complete`

- **Destination**: NATS Streaming subject `Subjects.ExpirationComplete` (inferred as `expiration:complete` based on naming convention from other subjects like `order:created`, `order:cancelled`, `ticket:created`)
- **Event key/type**: `Subjects.ExpirationComplete` enum value from `@b.anik/ticketing-common`
- **Producer location**: `src/events/publishers/expiration-complete-publisher.ts` — `ExpirationCompletePublisher.publish()` (lines 3-5)
- **Trigger**: Bull queue job processor fires after calculated delay (`src/queues/expiration-queue.ts`, lines 15-18)

- **Payload**: The exact message structure built at `src/queues/expiration-queue.ts` lines 16-18:
  ```json
  {
    "orderId": "string — job.data.orderId, the order ID from the queued job"
  }
  ```

- **Headers/attributes**: No custom headers set; uses base `Publisher` class from `@b.anik/ticketing-common` which handles NATS message serialization

- **Known consumers**: 
  - `orders` service: `ExpirationCompleteListener` at `services/orders/src/events/listeners/expiration-complete-listener.ts` — cancels the order and publishes `order:cancelled` event

---

## Part B: Events/Messages CONSUMED by this service

### Consumer: `order:created` (NATS Streaming)

- **Source**: NATS Streaming subject `Subjects.OrderCreated` (confirmed as `order:created` via test at `services/orders/src/routes/__tests__/new.test.ts` line 146)
- **Consumer group/subscription**: `expiration-service` (hardcoded at `src/events/listeners/order-created-listener.ts` line 7)
- **Handler location**: `src/events/listeners/order-created-listener.ts` — `OrderCreatedListener.onMessage()` (lines 9-33)
- **Event key/type**: `Subjects.OrderCreated` enum from `@b.anik/ticketing-common`

- **Expected payload**: Destructured from `OrderCreatedEvent['data']` type. Structure determined from test data (`src/events/listeners/__tests__/order-created-listener.test.ts` lines 7-16) and actual publish call (`services/orders/src/routes/new.ts` lines 56-66):
  ```json
  {
    "id": "string — MongoDB ObjectId hex string, the order ID",
    "version": "number — optimistic concurrency version, starts at 0",
    "status": "string — OrderStatus enum value (e.g., 'created')",
    "userId": "string — MongoDB ObjectId hex string, the user who created the order",
    "expiresAt": "string | undefined — ISO 8601 date string, when the order expires",
    "ticket": {
      "id": "string — MongoDB ObjectId hex string, the ticket ID",
      "price": "number — ticket price in currency units"
    }
  }
  ```

- **Processing logic**:
  1. Check if `data.expiresAt` is defined; if not, resolve immediately without scheduling (line 12-14)
  2. Calculate delay in milliseconds: `new Date(data.expiresAt).getTime() - new Date().getTime()` (line 17)
  3. Add job to Bull queue `expirationQueue` with payload `{ orderId: data.id }` and delay option (lines 19-25)
  4. Resolve the promise on success, reject on error (lines 28-30)

- **Output**: 
  - Adds a delayed job to Redis-backed Bull queue `order:expiration`
  - No immediate database writes
  - No immediate outbound events

- **Error handling**: 
  - Wraps processing in try/catch
  - Rejects promise on error (line 30), which should trigger NATS message redelivery via base `Listener` class behavior
  - No explicit dead letter queue in this service
  - No explicit retry logic in listener itself

---

## Part C: Workflow Orchestrations (if applicable)

### Bull Queue: `order:expiration`

This is a Redis-backed delayed job queue, not a full workflow orchestration system, but it serves as the core scheduling mechanism.

#### Queue Definition

- **File path**: `src/queues/expiration-queue.ts`
- **Queue name**: `order:expiration` (line 9)
- **Redis connection**: `process.env.REDIS_HOST` (line 11)
- **Job payload interface** (lines 5-7):
  ```typescript
  interface Payload {
    orderId: string;
  }
  ```

#### Queue Processor

- **File path**: `src/queues/expiration-queue.ts` (lines 15-18)
- **Processing logic**:
  1. Receive job with payload `{ orderId: string }`
  2. Instantiate `ExpirationCompletePublisher` with NATS client
  3. Publish `expiration:complete` event with `{ orderId: job.data.orderId }`
- **Retry policy**: Bull default retry behavior (configurable but not explicitly set in this codebase)
- **Error handling**: Not explicitly defined; Bull's default behavior applies

#### Job Addition (from OrderCreatedListener)

- **Location**: `src/events/listeners/order-created-listener.ts` lines 19-26
- **Delay calculation**: `new Date(expiresAt).getTime() - new Date().getTime()` milliseconds
- **Job options**: `{ delay: <calculated_delay> }`

---

## Part D: HTTP Calls to Other Services

**None.** 

The expiration service is a pure event-driven worker. It contains:
- No HTTP client libraries (no axios, fetch, got, or http imports)
- No REST API endpoints (no Express server)
- No outbound HTTP requests to other services

All inter-service communication is via NATS Streaming events.

---

## Part E: Webhooks and External API Calls

**None.**

The expiration service makes no calls to external third-party systems. Its only external connections are:
- **NATS Streaming Server**: `process.env.NATS_URL` (default: `http://nats-srv:4222`) — for event pub/sub
- **Redis**: `process.env.REDIS_HOST` (default: `expiration-redis-srv`) — for Bull queue storage

---

## Infrastructure Configuration Summary

| Component | Connection String | Purpose |
|-----------|-------------------|---------|
| NATS Streaming | `NATS_URL` / `http://nats-srv:4222` | Event pub/sub for `order:created` and `expiration:complete` |
| NATS Cluster ID | `NATS_CLUSTER_ID` / `ticketing` | NATS cluster identifier |
| NATS Client ID | `NATS_CLIENT_ID` / pod metadata name | Unique client identifier |
| Redis | `REDIS_HOST` / `expiration-redis-srv` | Bull queue backend on port 6379 |

Source: `infra/k8s/expiration-depl.yaml` lines 18-28, `infra/k8s/expiration-redis-depl.yaml`

---

## Event Flow Diagram

```
┌─────────────────┐                    ┌─────────────────────┐                    ┌─────────────┐
│  Orders Service │                    │  Expiration Service │                    │   Orders    │
│                 │                    │                     │                    │   Service   │
│  POST /api/orders                    │                     │                    │             │
│        │        │                    │                     │                    │             │
│        ▼        │  order:created     │                     │                    │             │
│  OrderCreated   │ ─────────────────► │ OrderCreatedListener│                    │             │
│  Publisher      │  {id, version,     │        │            │                    │             │
│                 │   status, userId,  │        ▼            │                    │             │
│                 │   expiresAt,       │  Calculate delay    │                    │             │
│                 │   ticket{id,price}}│        │            │                    │             │
└─────────────────┘                    │        ▼            │                    │             │
                                       │  expirationQueue    │                    │             │
                                       │  .add({orderId},    │                    │             │
                                       │   {delay})          │                    │             │
                                       │        │            │                    │             │
                                       │        ▼            │                    │             │
                                       │ ┌─────────────────┐ │                    │             │
                                       │ │   Redis Queue   │ │                    │             │
                                       │ │order:expiration │ │                    │             │
                                       │ └────────┬────────┘ │                    │             │
                                       │          │ (delay)  │                    │             │
                                       │          ▼          │                    │             │
                                       │   Queue Processor   │ expiration:complete│             │
                                       │   ExpirationComplete│ ────────────────►  │ Expiration  │
                                       │   Publisher         │  {orderId}         │ Complete    │
                                       │                     │                    │ Listener    │
                                       └─────────────────────┘                    │      │      │
                                                                                  │      ▼      │
                                                                                  │ Cancel order│
                                                                                  │ Publish     │
                                                                                  │ order:      │
                                                                                  │ cancelled   │
                                                                                  └─────────────┘
```

---

## Event Payload Reference

### OrderCreatedEvent (consumed)

```typescript
// Subject: 'order:created'
// Source: @b.anik/ticketing-common
interface OrderCreatedEvent {
  subject: Subjects.OrderCreated;
  data: {
    id: string;           // Order ID (MongoDB ObjectId)
    version: number;      // Optimistic concurrency version
    status: OrderStatus;  // Enum: Created, Cancelled, AwaitingPayment, Complete
    userId: string;       // User ID (MongoDB ObjectId)
    expiresAt?: string;   // ISO 8601 date string
    ticket: {
      id: string;         // Ticket ID (MongoDB ObjectId)
      price: number;      // Ticket price
    };
  };
}
```

### ExpirationCompleteEvent (produced)

```typescript
// Subject: 'expiration:complete' (inferred)
// Source: @b.anik/ticketing-common
interface ExpirationCompleteEvent {
  subject: Subjects.ExpirationComplete;
  data: {
    orderId: string;  // Order ID to be expired
  };
}
```

---

## Notes

1. **Type definitions**: The exact types for `OrderCreatedEvent`, `ExpirationCompleteEvent`, `Subjects`, `Listener`, and `Publisher` are defined in the external npm package `@b.anik/ticketing-common` (not available in this repository). The structures above are inferred from usage in source code and tests.

2. **Subject string values**: The NATS subject string `expiration:complete` is inferred from the naming pattern used by other subjects (`order:created`, `order:cancelled`, `ticket:created`). The actual string value is defined in `@b.anik/ticketing-common`.

3. **Graceful shutdown**: The service registers handlers for `SIGINT`, `SIGTERM`, and `SIGUSR2` signals to close the NATS connection cleanly (`src/index.ts` lines 30-32).

4. **Single replica**: Kubernetes deployment runs with 1 replica (`infra/k8s/expiration-depl.yaml` line 6). The queue group `expiration-service` ensures only one listener processes each message in a scaled deployment.
