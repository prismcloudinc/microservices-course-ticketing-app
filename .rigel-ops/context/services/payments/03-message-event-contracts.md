---
service_id: ticketing-app/payments
service_name: payments
prompt_id: 03-message-event-contracts
prompt_label: Message & Event Contracts
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.18473000000000003
schema_version: "2.0"
---

# Payments Service — Message & Event Contracts

This document catalogs all inter-service communications for the **payments** service, including events produced, events consumed, workflow orchestrations, HTTP calls to other services, and external API integrations.

---

## Part A: Events/Messages PRODUCED by this Service

### Event: `payment:awaited`

- **Destination**: NATS Streaming channel `payment:awaited` (subject: `Subjects.PaymentAwaited`)
- **Event key/type**: `payment:awaited`
- **Producer location**: `services/payments/src/events/listeners/order-created-listener.ts` → `OrderCreatedListener.onMessage()` (line 21-23)
- **Trigger**: Automatically published after the service receives and processes an `order:created` event and successfully creates a local Order record
- **Payload**:
  ```json
  {
    "orderId": "string — MongoDB ObjectId of the newly created order, sourced from the saved Order document's id (order.id)"
  }
  ```
- **Headers/attributes**: None explicitly set; NATS Streaming default message attributes apply
- **Known consumers**: 
  - **Orders service** — `PaymentAwaitedListener` in `services/orders/src/events/listeners/payment-awaited-listener.ts` (updates order status to `AwaitingPayment` and publishes `order:updated` event)

---

### Event: `payment:complete`

- **Destination**: NATS Streaming channel `payment:complete` (subject: `Subjects.PaymentComplete`)
- **Event key/type**: `payment:complete`
- **Producer location**: `services/payments/src/routes/new.ts` → `router.post('/api/payments', ...)` (lines 56-58)
- **Trigger**: Published after:
  1. User submits POST `/api/payments` with valid `token` and `orderId`
  2. Order ownership and status validation passes
  3. Stripe `charges.create()` returns a successful charge (`charge.paid === true`)
  4. Payment document is saved to MongoDB
- **Payload**:
  ```json
  {
    "orderId": "string — MongoDB ObjectId of the order, sourced from payment.order.id after the Payment document is saved"
  }
  ```
- **Headers/attributes**: None explicitly set; NATS Streaming default message attributes apply
- **Known consumers**: 
  - **Orders service** — `PaymentCompleteListener` in `services/orders/src/events/listeners/payment-complete-listener.ts` (updates order status to `Complete` and publishes `order:updated` event)

---

## Part B: Events/Messages CONSUMED by this Service

### Consumer: `order:created`

- **Source**: NATS Streaming channel `order:created` (subject: `Subjects.OrderCreated`)
- **Consumer group/subscription**: `payments-service` (`services/payments/src/events/listeners/queue-group-name.ts:1`)
- **Handler location**: `services/payments/src/events/listeners/order-created-listener.ts` → `OrderCreatedListener.onMessage()` (lines 9-30)
- **Event key/type**: Subject-based routing via NATS Streaming
- **Expected payload** (derived from test file `order-created-listener.test.ts:9-19`):
  ```json
  {
    "id": "string — MongoDB ObjectId of the order",
    "version": "number — optimistic concurrency version (starts at 0)",
    "status": "string — OrderStatus enum value (Created, Cancelled, AwaitingPayment, Complete)",
    "userId": "string — MongoDB ObjectId of the user who placed the order",
    "expiresAt": "string — ISO 8601 date string indicating order expiration time",
    "ticket": {
      "id": "string — MongoDB ObjectId of the ticket",
      "price": "number — ticket price in dollars"
    }
  }
  ```
- **Processing logic**:
  1. Builds a new local Order document using `Order.build()` with:
     - `id`: from `data.id` (preserves same ObjectId as Orders service)
     - `version`: from `data.version`
     - `price`: from `data.ticket.price`
     - `userId`: from `data.userId`
     - `status`: from `data.status`
  2. Saves the Order document to local MongoDB
  3. Publishes `payment:awaited` event with `{ orderId: order.id }`
  4. Resolves the promise to acknowledge message processing
- **Output**:
  - **DB write**: Creates new Order document in `payments` MongoDB database
  - **Event published**: `payment:awaited` with `{ orderId }` payload
- **Error handling**: Rejects the promise on any error, which causes NATS to redeliver the message (standard NATS Streaming retry behavior)

---

### Consumer: `order:cancelled`

- **Source**: NATS Streaming channel `order:cancelled` (subject: `Subjects.OrderCancelled`)
- **Consumer group/subscription**: `payments-service`
- **Handler location**: `services/payments/src/events/listeners/order-cancelled-listener.ts` → `OrderCancelledListener.onMessage()` (lines 8-28)
- **Event key/type**: Subject-based routing via NATS Streaming
- **Expected payload** (derived from test file `order-cancelled-listener.test.ts:24-30`):
  ```json
  {
    "id": "string — MongoDB ObjectId of the order",
    "version": "number — new version after cancellation (previous version + 1)",
    "ticket": {
      "id": "string — MongoDB ObjectId of the ticket"
    }
  }
  ```
- **Processing logic**:
  1. Finds existing Order using `Order.findByEvent({ id, version })` — this looks for an order with `_id === data.id` and `version === data.version - 1` (optimistic concurrency check)
  2. If order not found, throws `Error('Order not found')`
  3. Updates order with `status: OrderStatus.Cancelled` and `version: data.version`
  4. Saves the updated Order document
  5. Resolves the promise to acknowledge message processing
- **Output**:
  - **DB write**: Updates existing Order document, setting `status` to `Cancelled` and incrementing `version`
- **Error handling**: Rejects the promise on any error (order not found, version mismatch, DB error), causing NATS to redeliver the message

---

### Consumer: `order:updated`

- **Source**: NATS Streaming channel `order:updated` (subject: `Subjects.OrderUpdated`)
- **Consumer group/subscription**: `payments-service`
- **Handler location**: `services/payments/src/events/listeners/order-updated-listener.ts` → `OrderUpdatedListener.onMessage()` (lines 8-33)
- **Event key/type**: Subject-based routing via NATS Streaming
- **Expected payload** (derived from test file `order-updated-listener.test.ts:24-28`):
  ```json
  {
    "id": "string — MongoDB ObjectId of the order",
    "version": "number — new version number",
    "status": "string — OrderStatus enum value (Created, Cancelled, AwaitingPayment, Complete)"
  }
  ```
- **Processing logic**:
  1. Destructures `id`, `status`, `version` from event data
  2. Finds existing Order using `Order.findByEvent({ id, version })` — looks for order with matching id and `version === data.version - 1`
  3. If order not found, throws `Error('Order not found')`
  4. Updates order with new `status` and `version` values
  5. Saves the updated Order document
  6. Resolves the promise to acknowledge message processing
- **Output**:
  - **DB write**: Updates existing Order document with new `status` and `version`
- **Error handling**: Rejects the promise on any error, causing NATS message redelivery

---

## Part C: Workflow Orchestrations

**None detected.** The payments service does not use any durable workflow systems (Temporal, AWS Step Functions, Inngest, Bull queues, etc.). All orchestration is event-driven via NATS Streaming pub/sub.

---

## Part D: HTTP Calls to Other Services

**None detected.** The payments service communicates with other microservices exclusively through NATS Streaming events. No outbound HTTP client calls (axios, fetch, got, etc.) to internal services were found in the codebase.

---

## Part E: Webhooks and External API Calls

### External: Stripe — Create Charge

- **Client file**: `services/payments/src/stripe.ts` (lines 1-5)
- **SDK**: `stripe` npm package version 15.5.0
- **API Version**: `2024-04-10` (configured at `stripe.ts:4`)
- **Caller location**: `services/payments/src/routes/new.ts` → `router.post('/api/payments', ...)` (lines 39-43)
- **Operation**: `stripe.charges.create()`
- **Input** (constructed at `new.ts:39-43`):
  ```json
  {
    "currency": "usd",
    "amount": "number — order.price * 100 (Stripe expects amount in cents/minor currency unit)",
    "source": "string — Stripe payment token from request body (tokenized card/payment method)"
  }
  ```
- **Output** (Stripe Charge object, relevant fields used):
  ```json
  {
    "id": "string — Stripe charge ID (e.g., 'ch_xxx'), stored in Payment.stripeId",
    "paid": "boolean — indicates if the charge was successful"
  }
  ```
- **Error handling**:
  - If `charge.paid === false`: throws `BadRequestError('Payment failed')` returning HTTP 400 (`new.ts:45-47`)
  - Stripe SDK errors propagate up and are caught by Express error handling middleware, returning a formatted error response
  - No explicit retry logic implemented; failures result in immediate HTTP error response to the client

---

## Event Flow Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           PAYMENTS SERVICE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  CONSUMES (from Orders service):                                             │
│  ├── order:created ──────┬──► Creates local Order copy                       │
│  │                       └──► Publishes payment:awaited                      │
│  ├── order:cancelled ────────► Updates Order status to Cancelled             │
│  └── order:updated ──────────► Syncs Order status and version                │
│                                                                              │
│  PRODUCES:                                                                   │
│  ├── payment:awaited ────────► Consumed by Orders service                    │
│  │                             (triggers status change to AwaitingPayment)   │
│  └── payment:complete ───────► Consumed by Orders service                    │
│                                (triggers status change to Complete)          │
│                                                                              │
│  EXTERNAL CALLS:                                                             │
│  └── Stripe API ─────────────► charges.create() for payment processing       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## NATS Streaming Configuration

| Configuration | Value | Source |
|--------------|-------|--------|
| Cluster ID | `process.env.NATS_CLUSTER_ID` | `index.ts:33` |
| Client ID | `process.env.NATS_CLIENT_ID` | `index.ts:35` |
| URL | `process.env.NATS_URL` | `index.ts:36` |
| Queue Group | `payments-service` | `queue-group-name.ts:1` |

The queue group ensures that when multiple instances of the payments service are running, each message is delivered to only one instance (competing consumers pattern).

---

## Known Event Producers (consumed by this service)

| Event | Producer Service | Evidence |
|-------|-----------------|----------|
| `order:created` | Orders | `services/orders/src/events/publishers/order-created-publisher.ts` |
| `order:cancelled` | Orders | `services/orders/src/events/publishers/order-cancelled-publisher.ts` |
| `order:updated` | Orders | `services/orders/src/events/publishers/order-updated-publisher.ts` |

---

## Known Event Consumers (of events this service produces)

| Event | Consumer Service | Handler | Evidence |
|-------|-----------------|---------|----------|
| `payment:awaited` | Orders | `PaymentAwaitedListener` | `services/orders/src/events/listeners/payment-awaited-listener.ts` |
| `payment:complete` | Orders | `PaymentCompleteListener` | `services/orders/src/events/listeners/payment-complete-listener.ts` |
