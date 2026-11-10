---
service_id: ticketing-app/tickets
service_name: tickets
prompt_id: 03-message-event-contracts
prompt_label: Message & Event Contracts
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.18946000000000002
schema_version: "2.0"
---

# Inter-Service Communication Contracts: Tickets Service

## Part A: Events/Messages PRODUCED by this Service

The Tickets service publishes events to NATS Streaming for downstream services to consume.

---

### Event: `ticket:created`

- **Destination**: NATS Streaming subject `ticket:created` (via `Subjects.TicketCreated` enum)
- **Event key/type**: `ticket:created`
- **Producer location**: `services/tickets/src/routes/new.ts` — anonymous async handler in `router.post('/api/tickets', ...)` (lines 17-39)
- **Publisher class**: `TicketCreatedPublisher` (`services/tickets/src/events/publishers/ticket-created-publisher.ts:3-5`)
- **Trigger**: HTTP POST request to `/api/tickets` — when an authenticated user creates a new ticket
- **Payload**:
  ```json
  {
    "id": "string — MongoDB ObjectId of the created ticket (ticket.id from services/tickets/src/routes/new.ts:29)",
    "version": "number — document version for optimistic concurrency control (ticket.version, starts at 0)",
    "title": "string — ticket title from request body",
    "price": "number — ticket price from request body",
    "userId": "string — ID of the user who created the ticket (from req.currentUser.id)"
  }
  ```
  Source: `services/tickets/src/routes/new.ts:28-34`
- **Headers/attributes**: None set explicitly; NATS Streaming handles message metadata
- **Known consumers** (from this codebase):
  - `TicketCreatedListener` in **orders** service (`services/orders/src/events/listeners/ticket-created-listener.ts`) — creates a local ticket replica for order validation

---

### Event: `ticket:updated`

- **Destination**: NATS Streaming subject `ticket:updated` (via `Subjects.TicketUpdated` enum)
- **Event key/type**: `ticket:updated`
- **Producer locations**:
  1. `services/tickets/src/routes/update.ts` — anonymous async handler in `router.put('/api/tickets/:id', ...)` (lines 23-60)
  2. `services/tickets/src/events/listeners/order-created-listener.ts` — `OrderCreatedListener.onMessage()` (lines 11-40)
  3. `services/tickets/src/events/listeners/order-cancelled-listener.ts` — `OrderCancelledListener.onMessage()` (lines 11-45)
- **Publisher class**: `TicketUpdatedPublisher` (`services/tickets/src/events/publishers/ticket-updated-publisher.ts:3-5`)
- **Triggers**:
  1. HTTP PUT request to `/api/tickets/:id` — when an authenticated owner updates their ticket's title or price
  2. `order:created` event received — when an order reserves this ticket (sets `orderId`)
  3. `order:cancelled` event received — when an order is cancelled (clears `orderId`)
- **Payload**:
  ```json
  {
    "id": "string — MongoDB ObjectId of the ticket",
    "version": "number — incremented document version after save",
    "title": "string — current ticket title",
    "price": "number — current ticket price",
    "userId": "string — ID of the ticket owner",
    "orderId": "string | undefined — ID of the order reserving this ticket, or undefined if unreserved"
  }
  ```
  Source (from update route): `services/tickets/src/routes/update.ts:49-55`
  Source (from order-created listener): `services/tickets/src/events/listeners/order-created-listener.ts:26-33`
  Source (from order-cancelled listener): `services/tickets/src/events/listeners/order-cancelled-listener.ts:31-38`
- **Headers/attributes**: None set explicitly
- **Known consumers** (from this codebase):
  - `TicketUpdatedListener` in **orders** service (`services/orders/src/events/listeners/ticket-updated-listener.ts`) — updates the local ticket replica with new title, price, and version

---

## Part B: Events/Messages CONSUMED by this Service

The Tickets service subscribes to NATS Streaming events from other services to maintain ticket reservation state.

---

### Consumer: `order:created`

- **Source**: NATS Streaming subject `order:created` (via `Subjects.OrderCreated` enum)
- **Consumer group/subscription**: `tickets-service` (`services/tickets/src/events/listeners/queue-group-name.ts:1`)
- **Handler location**: `services/tickets/src/events/listeners/order-created-listener.ts` — `OrderCreatedListener.onMessage()` (lines 11-40)
- **Event key/type**: `order:created`
- **Expected payload**:
  ```json
  {
    "id": "string — MongoDB ObjectId of the order",
    "version": "number — order document version",
    "status": "string — OrderStatus enum value (e.g., 'created')",
    "userId": "string — ID of the user who placed the order",
    "expiresAt": "string — ISO 8601 timestamp when the order expires",
    "ticket": {
      "id": "string — MongoDB ObjectId of the reserved ticket",
      "price": "number — ticket price at time of order"
    }
  }
  ```
  Source (from test data): `services/tickets/src/events/listeners/__tests__/order-created-listener.test.ts:22-32`
  Cross-referenced with producer: `services/orders/src/routes/new.ts:56-66`
- **Processing logic**:
  1. Extract `ticketId` from `data.ticket.id` (line 14)
  2. Find the ticket in MongoDB by ID (line 17)
  3. If ticket not found, throw `Error('Ticket not found')` (lines 19-21)
  4. Set `ticket.orderId = data.id` to mark as reserved (line 23)
  5. Save the ticket document (line 24)
  6. Publish `ticket:updated` event with new orderId (lines 26-33)
  7. Resolve the promise (line 35)
- **Output**:
  - **DB write**: Updates `Ticket` document in MongoDB, setting `orderId` field
  - **Event published**: `ticket:updated` event to notify other services of reservation
- **Error handling**:
  - On any error, the promise is rejected (line 37)
  - NATS Streaming will not acknowledge the message, causing automatic redelivery based on subscription settings
  - No explicit dead letter queue configured in this service

---

### Consumer: `order:cancelled`

- **Source**: NATS Streaming subject `order:cancelled` (via `Subjects.OrderCancelled` enum)
- **Consumer group/subscription**: `tickets-service` (`services/tickets/src/events/listeners/queue-group-name.ts:1`)
- **Handler location**: `services/tickets/src/events/listeners/order-cancelled-listener.ts` — `OrderCancelledListener.onMessage()` (lines 11-45)
- **Event key/type**: `order:cancelled`
- **Expected payload**:
  ```json
  {
    "id": "string — MongoDB ObjectId of the cancelled order",
    "version": "number — order document version",
    "ticket": {
      "id": "string — MongoDB ObjectId of the ticket to release"
    }
  }
  ```
  Source (from handler type annotation): `services/tickets/src/events/listeners/order-cancelled-listener.ts:12`
  Source (from test data): `services/tickets/src/events/listeners/__tests__/order-cancelled-listener.test.ts:27-33`
  Cross-referenced with producer: `services/orders/src/routes/cancel.ts:50-56`
- **Processing logic**:
  1. Extract `ticketId` from `data.ticket.id` (line 17)
  2. Find ticket where `_id` matches ticketId AND `orderId` matches `data.id` (lines 19-22)
  3. If ticket not found (no match on both criteria), throw `Error('Ticket not found')` (lines 24-26)
  4. Set `ticket.orderId = undefined` to release reservation (line 28)
  5. Save the ticket document (line 29)
  6. Publish `ticket:updated` event with orderId cleared (lines 31-38)
  7. Resolve the promise (line 40)
- **Output**:
  - **DB write**: Updates `Ticket` document in MongoDB, clearing `orderId` field
  - **Event published**: `ticket:updated` event to notify other services of release
- **Error handling**:
  - On any error, the promise is rejected (line 42)
  - NATS Streaming will not acknowledge the message, causing automatic redelivery
  - No explicit dead letter queue configured in this service

---

## Part C: Workflow Orchestrations

**None identified.** The Tickets service does not use durable workflow systems such as Temporal, AWS Step Functions, Inngest, or Bull queues. All event handling is reactive via NATS Streaming listeners.

---

## Part D: HTTP Calls to Other Services

**None identified.** The Tickets service does not make any outbound HTTP calls to other internal microservices. All inter-service communication is performed asynchronously via NATS Streaming events.

Verification: Searched for `axios`, `fetch`, `got`, `httpx`, `request(`, `http.request`, `https://`, `http://`, `baseURL`, and `API_URL` patterns — no matches found in the tickets service source code.

---

## Part E: Webhooks and External API Calls

**None identified.** The Tickets service does not integrate with any external third-party systems (no payment processors, CRM/ERP, notification services, or cloud APIs).

---

## Summary: Communication Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              TICKETS SERVICE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  INBOUND (Consumed)                    OUTBOUND (Produced)                  │
│  ─────────────────                    ─────────────────────                 │
│                                                                             │
│  [order:created] ──────┐          ┌──────► [ticket:created]                │
│    from: orders        │          │          to: orders                     │
│    queue: tickets-svc  │          │                                         │
│                        ▼          │                                         │
│              ┌─────────────────┐  │                                         │
│              │   MongoDB       │  │                                         │
│              │   Ticket Doc    │──┼───────► [ticket:updated]               │
│              │   ─────────     │  │          to: orders                     │
│              │   title         │  │                                         │
│              │   price         │  │                                         │
│              │   userId        │  │                                         │
│              │   orderId?      │  │                                         │
│              │   version       │  │                                         │
│              └─────────────────┘  │                                         │
│                        ▲          │                                         │
│  [order:cancelled] ────┘          │                                         │
│    from: orders                   │                                         │
│    queue: tickets-svc             │                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Event Type Reference

| Subject | Direction | Payload Fields | Source/Destination |
|---------|-----------|----------------|-------------------|
| `ticket:created` | Produced | id, version, title, price, userId | → orders service |
| `ticket:updated` | Produced | id, version, title, price, userId, orderId? | → orders service |
| `order:created` | Consumed | id, version, status, userId, expiresAt, ticket.id, ticket.price | ← orders service |
| `order:cancelled` | Consumed | id, version, ticket.id | ← orders service |

---

## NATS Configuration

| Property | Value | Source |
|----------|-------|--------|
| **Cluster ID** | `ticketing` (from env `NATS_CLUSTER_ID`) | `infra/k8s/tickets-depl.yaml:28-29` |
| **Client ID** | Pod name (from `metadata.name`) | `infra/k8s/tickets-depl.yaml:30-32` |
| **URL** | `http://nats-srv:4222` (from env `NATS_URL`) | `infra/k8s/tickets-depl.yaml:26-27` |
| **Queue Group** | `tickets-service` | `services/tickets/src/events/listeners/queue-group-name.ts:1` |

---

## Graceful Shutdown

The service handles NATS connection cleanup on process termination (`services/tickets/src/index.ts:41-43`):
- `SIGINT` — closes NATS client
- `SIGTERM` — closes NATS client  
- `SIGUSR2` — closes NATS client

On NATS connection close, the process exits with code 1 (`services/tickets/src/index.ts:36-39`).
