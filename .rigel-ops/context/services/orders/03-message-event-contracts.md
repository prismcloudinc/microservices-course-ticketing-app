---
service_id: ticketing-app/orders
service_name: orders
prompt_id: 03-message-event-contracts
prompt_label: Message & Event Contracts
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.19635000000000002
schema_version: "2.0"
---

# Orders Service - API Contracts

## Part A: Events/Messages PRODUCED by this Service

### Event: `order:created`
- **Destination**: NATS Streaming channel `order:created` (via `Subjects.OrderCreated`)
- **Event key/type**: `order:created`
- **Producer location**: `services/orders/src/routes/new.ts`, `router.post('/api/orders', ...)` (lines 56-66)
- **Trigger**: HTTP POST request to `/api/orders` when a user creates a new order for an available ticket
- **Payload**:
  ```json
  {
    "id": "string — order._id (MongoDB ObjectId as string)",
    "version": "number — order.version (starts at 0, managed by mongoose-update-if-current)",
    "status": "string — order.status (OrderStatus enum value, always 'created' at publish time)",
    "userId": "string — req.currentUser.id (from JWT in cookie session)",
    "expiresAt": "string | undefined — order.expiresAt.toISOString() (ISO 8601 timestamp, 15 minutes from creation)",
    "ticket": {
      "id": "string — ticket._id (MongoDB ObjectId as string)",
      "price": "number — ticket.price (from local Ticket document)"
    }
  }
  ```
- **Headers/attributes**: None explicitly set; NATS Streaming uses default message attributes
- **Known consumers**:
  - `tickets` service (`services/tickets/src/events/listeners/order-created-listener.ts`)
  - `payments` service (`services/payments/src/events/listeners/order-created-listener.ts`)
  - `expiration` service (`services/expiration/src/events/listeners/order-created-listener.ts`)

---

### Event: `order:cancelled`
- **Destination**: NATS Streaming channel `order:cancelled` (via `Subjects.OrderCancelled`)
- **Event key/type**: `order:cancelled`
- **Producer locations**:
  1. `services/orders/src/routes/cancel.ts`, `router.patch('/api/orders/:orderId/cancel', ...)` (lines 50-56)
  2. `services/orders/src/events/listeners/expiration-complete-listener.ts`, `onMessage()` (lines 28-34)
- **Triggers**:
  1. HTTP PATCH request to `/api/orders/:orderId/cancel` when user manually cancels their order
  2. `expiration:complete` event received, indicating the order's 15-minute window has expired (and order is not already Complete or Cancelled)
- **Payload**:
  ```json
  {
    "id": "string — order._id (MongoDB ObjectId as string)",
    "version": "number — order.version (incremented after status change to Cancelled)",
    "ticket": {
      "id": "string — order.ticket._id (MongoDB ObjectId as string, from populated ticket)"
    }
  }
  ```
- **Headers/attributes**: None explicitly set
- **Known consumers**:
  - `tickets` service (`services/tickets/src/events/listeners/order-cancelled-listener.ts`)
  - `payments` service (`services/payments/src/events/listeners/order-cancelled-listener.ts`)

---

### Event: `order:updated`
- **Destination**: NATS Streaming channel `order:updated` (via `Subjects.OrderUpdated`)
- **Event key/type**: `order:updated`
- **Producer locations**:
  1. `services/orders/src/events/listeners/payment-awaited-listener.ts`, `onMessage()` (lines 21-25)
  2. `services/orders/src/events/listeners/payment-complete-listener.ts`, `onMessage()` (lines 23-27)
- **Triggers**:
  1. `payment:awaited` event received — updates order status to `AwaitingPayment`
  2. `payment:complete` event received — updates order status to `Complete`
- **Payload**:
  ```json
  {
    "id": "string — order._id (MongoDB ObjectId as string)",
    "version": "number — order.version (incremented after status change)",
    "status": "string — order.status (OrderStatus enum: 'awaiting:payment' or 'complete')"
  }
  ```
- **Headers/attributes**: None explicitly set
- **Known consumers**:
  - `payments` service (`services/payments/src/events/listeners/order-updated-listener.ts`)

---

## Part B: Events/Messages CONSUMED by this Service

### Consumer: `ticket:created`
- **Source**: NATS Streaming channel `ticket:created` (via `Subjects.TicketCreated`)
- **Consumer group/subscription**: `orders-service` (`services/orders/src/events/listeners/queue-group-name.ts`, line 1)
- **Handler location**: `services/orders/src/events/listeners/ticket-created-listener.ts`, `TicketCreatedListener.onMessage()` (lines 10-28)
- **Event key/type**: Event is discriminated by channel name (`ticket:created`)
- **Expected payload** (from test file `ticket-created-listener.test.ts`, lines 9-15):
  ```json
  {
    "id": "string — MongoDB ObjectId of the ticket",
    "version": "number — version number (starts at 0)",
    "userId": "string — ID of user who created the ticket (not stored locally)",
    "title": "string — ticket title",
    "price": "number — ticket price"
  }
  ```
- **Processing logic**:
  1. Extract `id`, `title`, `price`, `version` from event data (line 12)
  2. Build a new local Ticket document with these values (lines 15-20)
  3. Save the Ticket document to MongoDB (line 21)
  4. Resolve promise to acknowledge message (line 23)
- **Output**: MongoDB write — creates a new Ticket document in the local `tickets` collection
- **Error handling**: On error, rejects the promise (line 25), which causes the base `Listener` class to not acknowledge the message (NATS will redeliver)

---

### Consumer: `ticket:updated`
- **Source**: NATS Streaming channel `ticket:updated` (via `Subjects.TicketUpdated`)
- **Consumer group/subscription**: `orders-service`
- **Handler location**: `services/orders/src/events/listeners/ticket-updated-listener.ts`, `TicketUpdatedListener.onMessage()` (lines 10-32)
- **Event key/type**: Event is discriminated by channel name (`ticket:updated`)
- **Expected payload** (from test file `ticket-updated-listener.test.ts`, lines 27-33):
  ```json
  {
    "id": "string — MongoDB ObjectId of the ticket",
    "version": "number — new version number (must be previous version + 1)",
    "userId": "string — ID of user who owns the ticket (not used)",
    "title": "string — updated ticket title",
    "price": "number — updated ticket price"
  }
  ```
- **Processing logic**:
  1. Find ticket by ID and previous version using `Ticket.findByEvent(data)` (line 13) — queries for `_id: event.id, version: event.version - 1`
  2. If ticket not found, throw error "Ticket not found" (lines 15-17)
  3. Update ticket's `title`, `price`, and `version` fields (lines 19-23)
  4. Save the updated Ticket document (line 24)
  5. Resolve promise to acknowledge message (line 26)
- **Output**: MongoDB write — updates existing Ticket document
- **Error handling**: On error (including version mismatch/out-of-order events), rejects the promise (line 28), message is not acknowledged and will be redelivered by NATS

---

### Consumer: `expiration:complete`
- **Source**: NATS Streaming channel `expiration:complete` (via `Subjects.ExpirationComplete`)
- **Consumer group/subscription**: `orders-service`
- **Handler location**: `services/orders/src/events/listeners/expiration-complete-listener.ts`, `ExpirationCompleteListener.onMessage()` (lines 11-41)
- **Event key/type**: Event is discriminated by channel name (`expiration:complete`)
- **Expected payload** (from test file `expiration-complete-listener.test.ts`, lines 28-30):
  ```json
  {
    "orderId": "string — MongoDB ObjectId of the order that has expired"
  }
  ```
- **Processing logic**:
  1. Find order by ID and populate ticket reference (line 14)
  2. If order not found, throw error "Order not found" (lines 16-18)
  3. If order status is already `Cancelled` or `Complete`, resolve immediately without action (lines 20-23)
  4. Set order status to `Cancelled` (line 25)
  5. Save the updated Order document (line 26)
  6. Publish `order:cancelled` event (lines 28-34)
  7. Resolve promise to acknowledge message (line 36)
- **Output**:
  - MongoDB write — updates Order document status to `Cancelled`
  - Event publish — emits `order:cancelled` event
- **Error handling**: On error, rejects the promise (line 38), message is not acknowledged

---

### Consumer: `payment:awaited`
- **Source**: NATS Streaming channel `payment:awaited` (via `Subjects.PaymentAwaited`)
- **Consumer group/subscription**: `orders-service`
- **Handler location**: `services/orders/src/events/listeners/payment-awaited-listener.ts`, `PaymentAwaitedListener.onMessage()` (lines 9-32)
- **Event key/type**: Event is discriminated by channel name (`payment:awaited`)
- **Expected payload** (from test file `payment-awaited-listener.test.ts`, lines 29-31):
  ```json
  {
    "orderId": "string — MongoDB ObjectId of the order being paid"
  }
  ```
- **Processing logic**:
  1. Find order by ID and populate ticket reference (line 12)
  2. If order not found, throw error "Order not found" (lines 14-16)
  3. Set order status to `AwaitingPayment` (line 18)
  4. Save the updated Order document (line 19)
  5. Publish `order:updated` event with new status (lines 21-25)
  6. Resolve promise to acknowledge message (line 27)
- **Output**:
  - MongoDB write — updates Order document status to `AwaitingPayment`
  - Event publish — emits `order:updated` event
- **Error handling**: Note: There is a bug on line 29 where `reject;` should be `reject(error);` — error swallowed and promise may hang

---

### Consumer: `payment:complete`
- **Source**: NATS Streaming channel `payment:complete` (via `Subjects.PaymentComplete`)
- **Consumer group/subscription**: `orders-service`
- **Handler location**: `services/orders/src/events/listeners/payment-complete-listener.ts`, `PaymentCompleteListener.onMessage()` (lines 9-35)
- **Event key/type**: Event is discriminated by channel name (`payment:complete`)
- **Expected payload** (from test file `payment-complete.listener.test.ts`, lines 27-29):
  ```json
  {
    "orderId": "string — MongoDB ObjectId of the order that was paid"
  }
  ```
- **Processing logic**:
  1. Extract `orderId` from event data (line 12)
  2. Find order by ID (line 14)
  3. If order not found, throw error "Order not found" (lines 16-18)
  4. Set order status to `Complete` (line 20)
  5. Save the updated Order document (line 21)
  6. Publish `order:updated` event with new status (lines 23-27)
  7. Resolve promise to acknowledge message (line 29)
- **Output**:
  - MongoDB write — updates Order document status to `Complete`
  - Event publish — emits `order:updated` event
- **Error handling**: On error, rejects the promise (line 31), message is not acknowledged

---

## Part C: Workflow Orchestrations

**None identified.**

This service does not use durable workflow systems (Temporal, AWS Step Functions, Inngest, Bull queues, or similar). Order state transitions are managed through event-driven choreography rather than orchestration.

---

## Part D: HTTP Calls to Other Services

**None identified.**

This service does not make outbound HTTP calls to other internal services. All inter-service communication is handled through NATS Streaming events. The service maintains a local copy of ticket data (synchronized via `ticket:created` and `ticket:updated` events) rather than making synchronous HTTP calls to the tickets service.

---

## Part E: Webhooks and External API Calls

**None identified.**

This service does not integrate with external third-party systems directly. All external integrations (e.g., payment processing) are handled by other services in the architecture (e.g., the `payments` service), which communicate with the orders service via NATS events.

---

## Summary: Message Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              ORDERS SERVICE                                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  INBOUND EVENTS (CONSUMED)                  OUTBOUND EVENTS (PRODUCED)               │
│  ────────────────────────                   ─────────────────────────               │
│                                                                                      │
│  ticket:created ──────────┐                 ┌─────────► order:created                │
│  (from: tickets)          │                 │           (to: tickets, payments,      │
│                           │                 │                expiration)             │
│  ticket:updated ──────────┤                 │                                        │
│  (from: tickets)          │                 │                                        │
│                           │                 │                                        │
│  expiration:complete ─────┼───► ORDERS ─────┼─────────► order:cancelled              │
│  (from: expiration)       │     SERVICE     │           (to: tickets, payments)      │
│                           │                 │                                        │
│  payment:awaited ─────────┤                 │                                        │
│  (from: payments)         │                 │                                        │
│                           │                 │                                        │
│  payment:complete ────────┘                 └─────────► order:updated                │
│  (from: payments)                                       (to: payments)               │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Appendix: OrderStatus Enum Values

The `OrderStatus` enum (from `@b.anik/ticketing-common`) defines the following states:

| Status | Description | Transitions To |
|--------|-------------|----------------|
| `Created` | Initial state when order is created | `AwaitingPayment`, `Cancelled` |
| `AwaitingPayment` | Payment processing has begun | `Complete`, `Cancelled` |
| `Complete` | Payment successful, order fulfilled | (terminal state) |
| `Cancelled` | Order cancelled (user action or expiration) | (terminal state) |

---

## Appendix: NATS Connection Configuration

**Connection established in**: `services/orders/src/index.ts` (lines 29-33)

| Environment Variable | Purpose |
|---------------------|---------|
| `NATS_CLUSTER_ID` | NATS Streaming cluster identifier |
| `NATS_CLIENT_ID` | Unique client identifier for this service instance |
| `NATS_URL` | NATS Streaming server URL |

**Queue Group**: All listeners subscribe with queue group `orders-service`, ensuring only one instance of the orders service processes each message (load balancing).
