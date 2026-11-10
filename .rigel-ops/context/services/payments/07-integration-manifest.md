---
service_id: ticketing-app/payments
service_name: payments
prompt_id: 07-integration-manifest
prompt_label: Outbound Integration Manifest
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.13278500000000001
schema_version: "2.0"
---

# Integration Manifest: Payments Service

## Part A: Services I Call (HTTP)

This service makes no outbound HTTP calls to internal microservices. All inter-service communication is handled via NATS Streaming events.

---

## Part B: Messages/Events I Produce

### -> NATS Streaming: `payment:awaited`
- **Producer file**: `services/payments/src/events/publishers/payment-awaited-publisher.ts`
- **Event types published**:

| Event Key/Type | Trigger (what causes this publish) | Payload Shape |
|----------------|-----------------------------------|---------------|
| `payment:awaited` (Subject: `Subjects.PaymentAwaited`) | After successfully processing an `order:created` event and saving a local Order copy | `{ orderId: string }` |

**Source evidence**: `services/payments/src/events/listeners/order-created-listener.ts:21-23`

---

### -> NATS Streaming: `payment:complete`
- **Producer file**: `services/payments/src/events/publishers/payment-complete-publisher.ts`
- **Event types published**:

| Event Key/Type | Trigger (what causes this publish) | Payload Shape |
|----------------|-----------------------------------|---------------|
| `payment:complete` (Subject: `Subjects.PaymentComplete`) | After successful Stripe charge and Payment record saved to database | `{ orderId: string }` |

**Source evidence**: `services/payments/src/routes/new.ts:56-58`

---

## Part C: Messages/Events I Consume

### <- NATS Streaming: `order:created`
- **Consumer group/subscription**: `payments-service` (`services/payments/src/events/listeners/queue-group-name.ts:1`)
- **Handler file**: `services/payments/src/events/listeners/order-created-listener.ts`
- **Event types consumed**:

| Event Key/Type | Handler Function | What It Does | Output |
|----------------|-----------------|-------------|--------|
| `order:created` (Subject: `Subjects.OrderCreated`) | `OrderCreatedListener.onMessage()` | Creates local Order record with id, version, price (from ticket.price), userId, status | DB write: Order document; Event publish: `payment:awaited` |

**Expected payload shape** (from test: `services/payments/src/events/listeners/__tests__/order-created-listener.test.ts:9-19`):
```typescript
{
  id: string;
  version: number;
  status: OrderStatus;
  userId: string;
  expiresAt: string; // ISO date string
  ticket: {
    id: string;
    price: number;
  };
}
```

---

### <- NATS Streaming: `order:cancelled`
- **Consumer group/subscription**: `payments-service`
- **Handler file**: `services/payments/src/events/listeners/order-cancelled-listener.ts`
- **Event types consumed**:

| Event Key/Type | Handler Function | What It Does | Output |
|----------------|-----------------|-------------|--------|
| `order:cancelled` (Subject: `Subjects.OrderCancelled`) | `OrderCancelledListener.onMessage()` | Finds Order by id and version-1, updates status to `Cancelled` | DB write: Order status update |

**Expected payload shape** (from test: `services/payments/src/events/listeners/__tests__/order-cancelled-listener.test.ts:24-30`):
```typescript
{
  id: string;
  version: number;
  ticket: {
    id: string;
  };
}
```

---

### <- NATS Streaming: `order:updated`
- **Consumer group/subscription**: `payments-service`
- **Handler file**: `services/payments/src/events/listeners/order-updated-listener.ts`
- **Event types consumed**:

| Event Key/Type | Handler Function | What It Does | Output |
|----------------|-----------------|-------------|--------|
| `order:updated` (Subject: `Subjects.OrderUpdated`) | `OrderUpdatedListener.onMessage()` | Finds Order by id and version-1, updates status and version | DB write: Order status and version update |

**Expected payload shape** (from test: `services/payments/src/events/listeners/__tests__/order-updated-listener.test.ts:21-28`):
```typescript
{
  id: string;
  version: number;
  status: OrderStatus;
}
```

---

## Part D: Workflows I Trigger

No durable workflows (Temporal, Step Functions, Inngest, etc.) are triggered by this service.

---

## Part E: Workflows/Activities I Execute

This service does not execute any durable workflows or activities.

---

## Part F: Cloud Services I Use

| Cloud Service | Operation | Resource | Purpose |
|---------------|-----------|----------|---------|
| None | — | — | — |

This service does not directly use any cloud SDK services (AWS, GCP, Azure). Infrastructure is self-hosted Kubernetes with MongoDB and NATS Streaming.

---

## Part G: External Systems I Integrate With

### <-> Stripe (Payment Processing)
- **Client/SDK file**: `services/payments/src/stripe.ts`
- **SDK**: `stripe` npm package v15.5.0
- **API Version**: `2024-04-10` (`services/payments/src/stripe.ts:4`)
- **Operations**:

| Operation | Method | Purpose | Request Shape | Response Shape |
|-----------|--------|---------|---------------|----------------|
| Create Charge | `stripe.charges.create()` | Process payment for an order | `{ currency: 'usd', amount: number, source: string }` | `{ id: string, paid: boolean, ... }` |

**Source evidence**: `services/payments/src/routes/new.ts:39-43`

**Auth**: Stripe secret key via `STRIPE_KEY` environment variable, passed to Stripe constructor (`services/payments/src/stripe.ts:3`)

**Data exchanged**:
- **Outbound**: Payment token (from client), amount in cents (order.price * 100), currency (hardcoded 'usd')
- **Inbound**: Charge ID (stored as `stripeId` in Payment record), payment status (`paid` boolean)

---

## Infrastructure Dependencies

| System | Connection String/URL | Env Var | Purpose |
|--------|----------------------|---------|---------|
| MongoDB | `mongodb://payments-mongo-srv:27017/payments` | `MONGO_URI` | Primary data store for Order copies and Payment records |
| NATS Streaming | `http://nats-srv:4222` | `NATS_URL` | Event bus for inter-service messaging |

**Source evidence**: `infra/k8s/payments-depl.yaml:25-28`

---

## Summary Block

```yaml
service: payments
platform: ticketing-app
connects_to:
  http_outbound: []
  events_produced:
    - destination: nats-streaming
      events:
        - payment:awaited
        - payment:complete
  events_consumed:
    - source: nats-streaming
      events:
        - order:created
        - order:cancelled
        - order:updated
  workflows_triggered: []
  workflows_executed: []
  cloud_services: []
  external:
    - system: stripe
      operations:
        - charges.create
```
