---
service_id: ticketing-app/tickets
service_name: tickets
prompt_id: 07-integration-manifest
prompt_label: Outbound Integration Manifest
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.14509499999999997
schema_version: "2.0"
---

# External Connections Manifest: Tickets Service

## Part A: Services I Call (HTTP)

**No outbound HTTP calls found.**

This service does not make any outbound HTTP requests to other services. It only responds to incoming HTTP requests and communicates with other services via the NATS Streaming event bus.

Evidence: Searched for `axios`, `fetch`, `http.request`, `https.request`, `got`, `request(`, `superagent` patterns — only matches were in test files using `supertest` for testing the service's own endpoints (`services/tickets/src/routes/__tests__/*.test.ts`).

---

## Part B: Messages/Events I Produce

### -> NATS Streaming Channel: `ticket:created`

- **Producer file**: `services/tickets/src/events/publishers/ticket-created-publisher.ts`
- **Event types published**:

| Event Key/Type | Trigger (what causes this publish) | Payload Shape |
|----------------|-----------------------------------|---------------|
| `ticket:created` (`Subjects.TicketCreated`) | POST `/api/tickets` success — after saving a new ticket to MongoDB (`services/tickets/src/routes/new.ts:28-34`) | `{ id: string, version: number, title: string, price: number, userId: string }` |

**Publisher implementation** (`services/tickets/src/events/publishers/ticket-created-publisher.ts:3-5`):
```typescript
export class TicketCreatedPublisher extends Publisher<TicketCreatedEvent> {
  readonly subject = Subjects.TicketCreated;
}
```

---

### -> NATS Streaming Channel: `ticket:updated`

- **Producer file**: `services/tickets/src/events/publishers/ticket-updated-publisher.ts`
- **Event types published**:

| Event Key/Type | Trigger (what causes this publish) | Payload Shape |
|----------------|-----------------------------------|---------------|
| `ticket:updated` (`Subjects.TicketUpdated`) | PUT `/api/tickets/:id` success — after updating ticket title/price (`services/tickets/src/routes/update.ts:49-55`) | `{ id: string, version: number, title: string, price: number, userId: string, orderId?: string }` |
| `ticket:updated` (`Subjects.TicketUpdated`) | `order:created` event consumed — after setting `orderId` on ticket to reserve it (`services/tickets/src/events/listeners/order-created-listener.ts:26-33`) | `{ id: string, version: number, title: string, price: number, userId: string, orderId: string }` |
| `ticket:updated` (`Subjects.TicketUpdated`) | `order:cancelled` event consumed — after clearing `orderId` on ticket to release reservation (`services/tickets/src/events/listeners/order-cancelled-listener.ts:31-38`) | `{ id: string, version: number, title: string, price: number, userId: string, orderId: undefined }` |

**Publisher implementation** (`services/tickets/src/events/publishers/ticket-updated-publisher.ts:3-5`):
```typescript
export class TicketUpdatedPublisher extends Publisher<TicketUpdatedEvent> {
  readonly subject = Subjects.TicketUpdated;
}
```

---

## Part C: Messages/Events I Consume

### <- NATS Streaming Channel: `order:created`

- **Consumer group/subscription**: `tickets-service` (`services/tickets/src/events/listeners/queue-group-name.ts:1`)
- **Handler file**: `services/tickets/src/events/listeners/order-created-listener.ts`
- **Event types consumed**:

| Event Key/Type | Handler Function | What It Does | Output (DB write, HTTP call, event publish, workflow trigger) |
|----------------|-----------------|-------------|--------------------------------------------------------------|
| `order:created` (`Subjects.OrderCreated`) | `OrderCreatedListener.onMessage()` | Marks a ticket as reserved by setting `orderId` field to the order's ID | **DB write**: Updates ticket document with `orderId` field<br>**Event publish**: Publishes `ticket:updated` event with new reservation state |

**Incoming payload shape** (from test file `services/tickets/src/events/listeners/__tests__/order-created-listener.test.ts:22-32`):
```typescript
{
  id: string,           // Order ID
  version: number,
  status: OrderStatus,  // e.g., OrderStatus.Created
  userId: string,
  expiresAt: string,    // ISO date string
  ticket: {
    id: string,         // Ticket ID being reserved
    price: number
  }
}
```

---

### <- NATS Streaming Channel: `order:cancelled`

- **Consumer group/subscription**: `tickets-service` (`services/tickets/src/events/listeners/queue-group-name.ts:1`)
- **Handler file**: `services/tickets/src/events/listeners/order-cancelled-listener.ts`
- **Event types consumed**:

| Event Key/Type | Handler Function | What It Does | Output (DB write, HTTP call, event publish, workflow trigger) |
|----------------|-----------------|-------------|--------------------------------------------------------------|
| `order:cancelled` (`Subjects.OrderCancelled`) | `OrderCancelledListener.onMessage()` | Releases a ticket reservation by clearing `orderId` field | **DB write**: Updates ticket document, clears `orderId` field<br>**Event publish**: Publishes `ticket:updated` event with cleared reservation |

**Incoming payload shape** (from test file `services/tickets/src/events/listeners/__tests__/order-cancelled-listener.test.ts:27-33`):
```typescript
{
  id: string,           // Order ID being cancelled
  version: number,
  ticket: {
    id: string          // Ticket ID to release
  }
}
```

---

## Part D: Workflows I Trigger

**None.**

No workflow orchestration tools (Temporal, Inngest, Step Functions, etc.) are used in this service. Evidence: Searched for `temporal`, `@temporalio`, `inngest`, `step-functions`, `stepfunctions`, `workflow` patterns — no matches found.

---

## Part E: Workflows/Activities I Execute

**None.**

This service is not a workflow worker or activity executor.

---

## Part F: Cloud Services I Use

**None.**

This service does not directly call any cloud provider SDKs. Evidence: Searched for `aws-sdk`, `@aws-sdk`, `@azure`, `@google-cloud`, `firebase`, `s3.`, `sqs.`, `sns.`, `dynamodb`, `secretsmanager`, `ssm` patterns — only matches were optional peer dependencies in `pnpm-lock.yaml` (MongoDB driver's optional AWS credential provider support, not actually installed or used).

**Infrastructure dependencies** (from Kubernetes manifests):
- **MongoDB**: `mongodb://tickets-mongo-srv:27017/tickets` (`infra/k8s/tickets-depl.yaml:25`) — in-cluster MongoDB deployment
- **NATS Streaming**: `http://nats-srv:4222` with cluster ID `ticketing` (`infra/k8s/tickets-depl.yaml:27-29`) — in-cluster NATS deployment

---

## Part G: External Systems I Integrate With

**None.**

No external third-party systems (payment processors, CRM/ERP, scheduling platforms, notification services, etc.) are directly integrated. Evidence: Searched for `stripe`, `paypal`, `twilio`, `sendgrid`, `mailgun`, `redis` patterns — no matches found.

---

## Summary Block

```yaml
service: tickets
platform: ticketing-app
connects_to:
  http_outbound: []
  events_produced:
    - destination: ticket:created
      broker: nats-streaming
      cluster_id: ticketing
      events:
        - ticket:created
    - destination: ticket:updated
      broker: nats-streaming
      cluster_id: ticketing
      events:
        - ticket:updated
  events_consumed:
    - source: order:created
      broker: nats-streaming
      cluster_id: ticketing
      queue_group: tickets-service
      events:
        - order:created
    - source: order:cancelled
      broker: nats-streaming
      cluster_id: ticketing
      queue_group: tickets-service
      events:
        - order:cancelled
  workflows_triggered: []
  workflows_executed: []
  cloud_services: []
  external: []
  databases:
    - type: mongodb
      connection_env_var: MONGO_URI
      default_uri: mongodb://tickets-mongo-srv:27017/tickets
```
