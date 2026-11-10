---
service_id: ticketing-app/orders
service_name: orders
prompt_id: 07-integration-manifest
prompt_label: Outbound Integration Manifest
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.16326500000000002
schema_version: "2.0"
---

# External Connections Manifest: Orders Service

## Part A: Services I Call (HTTP)

**No outbound HTTP clients found.**

The orders service does not make any outbound HTTP calls to other services. All inter-service communication is handled through NATS Streaming events.

---

## Part B: Messages/Events I Produce

### -> NATS Streaming: `order:created`
- **Producer file**: `services/orders/src/events/publishers/order-created-publisher.ts`
- **Event types published**:

| Event Key/Type | Trigger (what causes this publish) | Payload Shape |
|----------------|-----------------------------------|---------------|
| `order:created` (Subject: `Subjects.OrderCreated`) | Order creation via `POST /api/orders` (`services/orders/src/routes/new.ts`, lines 56-66) | `{ id: string, version: number, status: OrderStatus, userId: string, expiresAt?: string (ISO), ticket: { id: string, price: number } }` |

### -> NATS Streaming: `order:cancelled`
- **Producer file**: `services/orders/src/events/publishers/order-cancelled-publisher.ts`
- **Event types published**:

| Event Key/Type | Trigger (what causes this publish) | Payload Shape |
|----------------|-----------------------------------|---------------|
| `order:cancelled` (Subject: `Subjects.OrderCancelled`) | 1. User cancellation via `PATCH /api/orders/:orderId/cancel` (`services/orders/src/routes/cancel.ts`, lines 50-56) <br> 2. Expiration timeout (`services/orders/src/events/listeners/expiration-complete-listener.ts`, lines 28-34) | `{ id: string, version: number, ticket: { id: string } }` |

### -> NATS Streaming: `order:updated`
- **Producer file**: `services/orders/src/events/publishers/order-updated-publisher.ts`
- **Event types published**:

| Event Key/Type | Trigger (what causes this publish) | Payload Shape |
|----------------|-----------------------------------|---------------|
| `order:updated` (Subject: `Subjects.OrderUpdated`) | 1. Payment awaited notification received (`services/orders/src/events/listeners/payment-awaited-listener.ts`, lines 21-25) <br> 2. Payment complete notification received (`services/orders/src/events/listeners/payment-complete-listener.ts`, lines 23-27) | `{ id: string, version: number, status: OrderStatus }` |

---

## Part C: Messages/Events I Consume

### <- NATS Streaming: `ticket:created`
- **Consumer group/subscription**: `orders-service` (`services/orders/src/events/listeners/queue-group-name.ts`)
- **Handler file**: `services/orders/src/events/listeners/ticket-created-listener.ts`
- **Event types consumed**:

| Event Key/Type | Handler Function | What It Does | Output |
|----------------|-----------------|-------------|--------|
| `ticket:created` (Subject: `Subjects.TicketCreated`) | `TicketCreatedListener.onMessage()` | Creates a local copy of the ticket in the orders service database for order validation | DB write: Creates new `Ticket` document with `{ id, title, price, version }` |

**Expected payload shape** (from test file, line 9-15): `{ id: string, version: number, userId: string, title: string, price: number }`

### <- NATS Streaming: `ticket:updated`
- **Consumer group/subscription**: `orders-service`
- **Handler file**: `services/orders/src/events/listeners/ticket-updated-listener.ts`
- **Event types consumed**:

| Event Key/Type | Handler Function | What It Does | Output |
|----------------|-----------------|-------------|--------|
| `ticket:updated` (Subject: `Subjects.TicketUpdated`) | `TicketUpdatedListener.onMessage()` | Updates the local ticket copy with new title, price, and version (using optimistic concurrency via version-1 lookup) | DB write: Updates existing `Ticket` document |

**Expected payload shape** (from test file, line 27-32): `{ id: string, version: number, userId: string, title: string, price: number }`

### <- NATS Streaming: `expiration:complete`
- **Consumer group/subscription**: `orders-service`
- **Handler file**: `services/orders/src/events/listeners/expiration-complete-listener.ts`
- **Event types consumed**:

| Event Key/Type | Handler Function | What It Does | Output |
|----------------|-----------------|-------------|--------|
| `expiration:complete` (Subject: `Subjects.ExpirationComplete`) | `ExpirationCompleteListener.onMessage()` | Cancels the order if it's not already `Complete` or `Cancelled` | DB write: Sets order status to `Cancelled`; Event publish: `order:cancelled` |

**Expected payload shape** (from test file, line 28-30): `{ orderId: string }`

### <- NATS Streaming: `payment:awaited`
- **Consumer group/subscription**: `orders-service`
- **Handler file**: `services/orders/src/events/listeners/payment-awaited-listener.ts`
- **Event types consumed**:

| Event Key/Type | Handler Function | What It Does | Output |
|----------------|-----------------|-------------|--------|
| `payment:awaited` (Subject: `Subjects.PaymentAwaited`) | `PaymentAwaitedListener.onMessage()` | Updates order status to `AwaitingPayment` | DB write: Sets order status to `AwaitingPayment`; Event publish: `order:updated` |

**Expected payload shape** (from test file, line 29-31): `{ orderId: string }`

### <- NATS Streaming: `payment:complete`
- **Consumer group/subscription**: `orders-service`
- **Handler file**: `services/orders/src/events/listeners/payment-complete-listener.ts`
- **Event types consumed**:

| Event Key/Type | Handler Function | What It Does | Output |
|----------------|-----------------|-------------|--------|
| `payment:complete` (Subject: `Subjects.PaymentComplete`) | `PaymentCompleteListener.onMessage()` | Updates order status to `Complete` | DB write: Sets order status to `Complete`; Event publish: `order:updated` |

**Expected payload shape** (from test file, line 27-29): `{ orderId: string }`

---

## Part D: Workflows I Trigger

**No workflow triggers found.**

The orders service does not trigger any durable workflows (no Temporal, Step Functions, Inngest, or similar workflow orchestration).

---

## Part E: Workflows/Activities I Execute

**Not a workflow worker.**

The orders service does not execute workflow activities.

---

## Part F: Cloud Services I Use

| Cloud Service | Operation | Resource | Purpose |
|---------------|-----------|----------|---------|
| *None* | — | — | — |

The orders service does not directly use any cloud SDK services. It connects to MongoDB and NATS Streaming, which are self-hosted infrastructure components (as configured in Kubernetes manifests).

---

## Part G: External Systems I Integrate With

**No external third-party systems.**

The orders service communicates exclusively with internal infrastructure:
- **MongoDB**: Order and Ticket data persistence (`MONGO_URI` env var, `services/orders/src/index.ts` line 47)
- **NATS Streaming**: Event-driven messaging (`NATS_URL`, `NATS_CLUSTER_ID`, `NATS_CLIENT_ID` env vars, `services/orders/src/index.ts` lines 29-33)

---

## Infrastructure Connections

### Database: MongoDB
- **Connection file**: `services/orders/src/index.ts` (line 47)
- **Connection string env var**: `MONGO_URI`
- **Kubernetes URI**: `mongodb://orders-mongo-srv:27017/orders` (`infra/k8s/orders-depl.yaml` line 25)
- **Collections used**:
  - `orders` - Order documents (`services/orders/src/models/order.ts`)
  - `tickets` - Local ticket copies for reservation checking (`services/orders/src/models/ticket.ts`)

### Message Broker: NATS Streaming
- **Connection file**: `services/orders/src/nats-wrapper.ts`
- **Environment variables**:
  - `NATS_URL`: NATS server URL (Kubernetes: `http://nats-srv:4222`)
  - `NATS_CLUSTER_ID`: Cluster identifier (Kubernetes: `ticketing`)
  - `NATS_CLIENT_ID`: Unique client identifier (Kubernetes: derived from pod name)
- **Queue group**: `orders-service` (ensures only one instance processes each message)

---

## Summary Block

```yaml
service: orders
platform: ticketing-app
connects_to:
  http_outbound: []
  events_produced:
    - destination: nats-streaming
      events:
        - order:created
        - order:cancelled
        - order:updated
  events_consumed:
    - source: nats-streaming
      events:
        - ticket:created
        - ticket:updated
        - expiration:complete
        - payment:awaited
        - payment:complete
  workflows_triggered: []
  workflows_executed: []
  cloud_services: []
  external: []
  databases:
    - type: mongodb
      connection_env: MONGO_URI
      collections:
        - orders
        - tickets
  message_brokers:
    - type: nats-streaming
      connection_env: NATS_URL
      queue_group: orders-service
```
