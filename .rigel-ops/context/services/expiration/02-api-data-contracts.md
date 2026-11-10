---
service_id: ticketing-app/expiration
service_name: expiration
prompt_id: 02-api-data-contracts
prompt_label: API & Data Contracts
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.17529
schema_version: "2.0"
---

# Expiration Service API Contract & Data Model Documentation

## Overview

The expiration service is a **worker process** that does not expose any HTTP endpoints. It is an event-driven microservice that:

1. **Subscribes** to NATS Streaming events (`order:created`)
2. **Schedules** delayed jobs via Bull/Redis queue
3. **Publishes** NATS Streaming events (`expiration:complete`)

---

## Part A: HTTP API Endpoints

**This service exposes no HTTP endpoints.**

The expiration service is a background worker that communicates exclusively through:
- NATS Streaming (event consumption and publishing)
- Redis (Bull queue for job scheduling)

There is no Express/HTTP server initialized in `src/index.ts` (line 1-41).

---

## Part B: Event-Driven API

### Inbound Events (Subscriptions)

#### `order:created` (Subject: `Subjects.OrderCreated`)

| Attribute | Value |
|-----------|-------|
| **Listener Class** | `OrderCreatedListener` |
| **File** | `src/events/listeners/order-created-listener.ts` |
| **Queue Group** | `expiration-service` (hardcoded at line 7) |
| **Subject** | `Subjects.OrderCreated` (from `@b.anik/ticketing-common`) |

**Event Payload Schema** (derived from test file at `src/events/listeners/__tests__/order-created-listener.test.ts`, lines 7-16, and orders service at `services/orders/src/routes/new.ts`, lines 56-66):

```typescript
interface OrderCreatedEvent {
  subject: Subjects.OrderCreated;  // 'order:created'
  data: {
    id: string;              // MongoDB ObjectId as hex string
    version: number;         // Optimistic concurrency version, starts at 0
    status: OrderStatus;     // Enum: 'created' | 'cancelled' | 'awaiting:payment' | 'complete'
    userId: string;          // MongoDB ObjectId as hex string
    expiresAt?: string;      // ISO 8601 datetime string (optional)
    ticket: {
      id: string;            // MongoDB ObjectId as hex string
      price: number;         // Ticket price in smallest currency unit
    };
  };
}
```

**Processing Logic** (`src/events/listeners/order-created-listener.ts`, lines 9-33):

1. **Validate expiration**: If `data.expiresAt` is undefined/null, resolve immediately without scheduling (line 12-15)
2. **Calculate delay**: `delay = new Date(data.expiresAt).getTime() - new Date().getTime()` (line 17)
3. **Schedule job**: Add job to Bull queue `order:expiration` with payload `{ orderId: data.id }` and delay option (lines 19-26)
4. **Acknowledge**: Resolve promise to ack the NATS message (line 28)

**Error Handling**:
- If queue add fails, promise rejects and message is not acknowledged (line 30)

---

### Outbound Events (Publications)

#### `expiration:complete` (Subject: `Subjects.ExpirationComplete`)

| Attribute | Value |
|-----------|-------|
| **Publisher Class** | `ExpirationCompletePublisher` |
| **File** | `src/events/publishers/expiration-complete-publisher.ts` |
| **Subject** | `Subjects.ExpirationComplete` (from `@b.anik/ticketing-common`) |
| **Triggered By** | Bull queue processor when delayed job fires |

**Event Payload Schema** (derived from `src/queues/expiration-queue.ts`, lines 16-18, and consumer at `services/orders/src/events/listeners/__tests__/expiration-complete-listener.test.ts`, lines 28-30):

```typescript
interface ExpirationCompleteEvent {
  subject: Subjects.ExpirationComplete;  // 'expiration:complete'
  data: {
    orderId: string;  // MongoDB ObjectId as hex string - references the expired order
  };
}
```

**Publishing Logic** (`src/queues/expiration-queue.ts`, lines 15-19):

1. Queue processor receives job with `{ orderId: string }` payload
2. Creates new `ExpirationCompletePublisher` instance with NATS client
3. Publishes event with `{ orderId: job.data.orderId }`

---

### Job Queue

#### `order:expiration` (Bull Queue)

| Attribute | Value |
|-----------|-------|
| **Queue Name** | `order:expiration` |
| **File** | `src/queues/expiration-queue.ts` |
| **Backend** | Redis at `process.env.REDIS_HOST` |
| **Purpose** | Delays expiration processing until order expiry time |

**Job Payload Interface** (`src/queues/expiration-queue.ts`, lines 5-7):

```typescript
interface Payload {
  orderId: string;  // MongoDB ObjectId as hex string
}
```

**Job Options**:

| Option | Type | Description |
|--------|------|-------------|
| `delay` | `number` | Milliseconds until job should be processed (calculated from `expiresAt - now`) |

**Processor Behavior** (`src/queues/expiration-queue.ts`, lines 15-19):

When a delayed job fires:
1. Extracts `orderId` from `job.data`
2. Instantiates `ExpirationCompletePublisher` with `natsWrapper.client`
3. Publishes `ExpirationCompleteEvent` with `{ orderId }`

---

## Part C: Database Schema

**This service has no database schema.**

Redis is used only as a backing store for the Bull job queue. There are no persistent data models or schemas defined by this service.

### Redis Data Structures (Bull Queue Internal)

Bull automatically manages these Redis keys for the `order:expiration` queue:

| Key Pattern | Type | Purpose |
|-------------|------|---------|
| `bull:order:expiration:id` | String | Auto-incrementing job ID counter |
| `bull:order:expiration:delayed` | Sorted Set | Jobs waiting for delay to expire (scored by execution timestamp) |
| `bull:order:expiration:waiting` | List | Jobs ready to be processed |
| `bull:order:expiration:active` | List | Jobs currently being processed |
| `bull:order:expiration:completed` | Set | Successfully completed job IDs |
| `bull:order:expiration:failed` | Set | Failed job IDs |
| `bull:order:expiration:<jobId>` | Hash | Individual job data including payload |

---

## Part D: Type Definitions & DTOs

### Internal Types

#### `Payload` (Job Queue Payload)

| Attribute | Value |
|-----------|-------|
| **Name** | `Payload` |
| **File** | `src/queues/expiration-queue.ts`, lines 5-7 |
| **Usage** | Bull queue job data, passed to `expirationQueue.add()` |

```typescript
interface Payload {
  orderId: string;
}
```

---

### External Types (from `@b.anik/ticketing-common`)

These types are imported from the external npm package `@b.anik/ticketing-common` (version `^0.1.2` per `package.json`, line 19). Definitions below are reconstructed from usage patterns in this service and other services in the monorepo.

#### `OrderCreatedEvent`

| Attribute | Value |
|-----------|-------|
| **Name** | `OrderCreatedEvent` |
| **Package** | `@b.anik/ticketing-common` |
| **Imported In** | `src/events/listeners/order-created-listener.ts`, line 1 |
| **Usage** | Type parameter for `Listener<OrderCreatedEvent>` |

```typescript
interface OrderCreatedEvent {
  subject: Subjects.OrderCreated;
  data: {
    id: string;
    version: number;
    status: OrderStatus;
    userId: string;
    expiresAt?: string;
    ticket: {
      id: string;
      price: number;
    };
  };
}
```

#### `ExpirationCompleteEvent`

| Attribute | Value |
|-----------|-------|
| **Name** | `ExpirationCompleteEvent` |
| **Package** | `@b.anik/ticketing-common` |
| **Imported In** | `src/events/publishers/expiration-complete-publisher.ts`, line 1 |
| **Usage** | Type parameter for `Publisher<ExpirationCompleteEvent>` |

```typescript
interface ExpirationCompleteEvent {
  subject: Subjects.ExpirationComplete;
  data: {
    orderId: string;
  };
}
```

#### `Subjects` (Enum)

| Attribute | Value |
|-----------|-------|
| **Name** | `Subjects` |
| **Package** | `@b.anik/ticketing-common` |
| **Imported In** | `src/events/listeners/order-created-listener.ts`, `src/events/publishers/expiration-complete-publisher.ts` |

```typescript
enum Subjects {
  OrderCreated = 'order:created',
  ExpirationComplete = 'expiration:complete',
  // ... other subjects not used by this service
}
```

#### `OrderStatus` (Enum)

| Attribute | Value |
|-----------|-------|
| **Name** | `OrderStatus` |
| **Package** | `@b.anik/ticketing-common` |
| **Imported In** | `src/events/listeners/__tests__/order-created-listener.test.ts`, line 1 |
| **Usage** | Test fixtures only |

```typescript
enum OrderStatus {
  Created = 'created',
  Cancelled = 'cancelled',
  AwaitingPayment = 'awaiting:payment',
  Complete = 'complete',
}
```

#### `Listener<T>` (Abstract Class)

| Attribute | Value |
|-----------|-------|
| **Name** | `Listener` |
| **Package** | `@b.anik/ticketing-common` |
| **Imported In** | `src/events/listeners/order-created-listener.ts`, line 1 |
| **Usage** | Base class for `OrderCreatedListener` |

```typescript
abstract class Listener<T extends Event> {
  abstract subject: T['subject'];
  abstract queueGroupName: string;
  abstract onMessage(data: T['data'], msg?: Message): Promise<void>;
  
  constructor(client: Stan);
  listen(): void;
}
```

#### `Publisher<T>` (Abstract Class)

| Attribute | Value |
|-----------|-------|
| **Name** | `Publisher` |
| **Package** | `@b.anik/ticketing-common` |
| **Imported In** | `src/events/publishers/expiration-complete-publisher.ts`, line 1 |
| **Usage** | Base class for `ExpirationCompletePublisher` |

```typescript
abstract class Publisher<T extends Event> {
  abstract subject: T['subject'];
  
  constructor(client: Stan);
  publish(data: T['data']): Promise<void>;
}
```

---

### Singleton Classes

#### `NatsWrapper`

| Attribute | Value |
|-----------|-------|
| **Name** | `NatsWrapper` |
| **File** | `src/nats-wrapper.ts`, lines 4-24 |
| **Export** | `natsWrapper` (singleton instance) |
| **Usage** | Provides NATS client access throughout the service |

```typescript
class NatsWrapper {
  private _client?: Stan;
  
  get client(): Stan;  // Throws if not connected
  
  async connect(
    clusterId: string,
    clientId: string,
    url: string
  ): Promise<void>;
}

export const natsWrapper: NatsWrapper;
```

---

## Part E: Environment Variables

| Variable | Required | Default | Description | Source |
|----------|----------|---------|-------------|--------|
| `NATS_URL` | Yes | - | NATS Streaming server URL (e.g., `http://nats-srv:4222`) | `src/index.ts`, line 5 |
| `NATS_CLUSTER_ID` | Yes | - | NATS cluster identifier (e.g., `ticketing`) | `src/index.ts`, line 9 |
| `NATS_CLIENT_ID` | Yes | - | Unique client identifier (typically pod name) | `src/index.ts`, line 20 |
| `REDIS_HOST` | Yes | - | Redis server hostname (e.g., `expiration-redis-srv`) | `src/queues/expiration-queue.ts`, line 11 |

---

## Part F: Service Interactions

### Upstream Services (Publishers to this service)

| Service | Event | Description |
|---------|-------|-------------|
| Orders Service | `order:created` | Triggers expiration scheduling when a new order is created |

### Downstream Services (Consumers of this service)

| Service | Event | Description |
|---------|-------|-------------|
| Orders Service | `expiration:complete` | Cancels the order if not yet paid/completed (`services/orders/src/events/listeners/expiration-complete-listener.ts`) |

### Infrastructure Dependencies

| Dependency | Purpose | Connection |
|------------|---------|------------|
| NATS Streaming Server | Event bus for publish/subscribe | `NATS_URL` (TCP connection via `node-nats-streaming`) |
| Redis | Bull queue job storage | `REDIS_HOST:6379` (TCP connection via `bull`) |

---

## Part G: Event Flow Sequence

```
┌───────────────┐         ┌───────────────────┐         ┌───────────────┐
│ Orders        │         │ Expiration        │         │ Orders        │
│ Service       │         │ Service           │         │ Service       │
└───────┬───────┘         └─────────┬─────────┘         └───────┬───────┘
        │                           │                           │
        │  order:created            │                           │
        │  {id, expiresAt, ...}     │                           │
        │──────────────────────────>│                           │
        │                           │                           │
        │                     ┌─────┴─────┐                     │
        │                     │ Calculate │                     │
        │                     │   delay   │                     │
        │                     └─────┬─────┘                     │
        │                           │                           │
        │                     ┌─────┴─────┐                     │
        │                     │ Add job   │                     │
        │                     │ to Bull   │                     │
        │                     │ queue     │                     │
        │                     └─────┬─────┘                     │
        │                           │                           │
        │                    [delay elapses]                    │
        │                           │                           │
        │                     ┌─────┴─────┐                     │
        │                     │ Process   │                     │
        │                     │ job       │                     │
        │                     └─────┬─────┘                     │
        │                           │                           │
        │                           │  expiration:complete      │
        │                           │  {orderId}                │
        │                           │──────────────────────────>│
        │                           │                           │
        │                           │                     ┌─────┴─────┐
        │                           │                     │ Cancel    │
        │                           │                     │ order if  │
        │                           │                     │ not paid  │
        │                           │                     └───────────┘
```

---

## Part H: Kubernetes Deployment Configuration

### expiration-depl (Deployment)

**File**: `infra/k8s/expiration-depl.yaml`

| Field | Value |
|-------|-------|
| Replicas | 1 |
| Container Image | `berkinanik/ticketing-expiration:latest` |
| Environment Variables | `NATS_URL=http://nats-srv:4222`, `NATS_CLUSTER_ID=ticketing`, `NATS_CLIENT_ID=${metadata.name}`, `REDIS_HOST=expiration-redis-srv` |

### expiration-redis-depl (Deployment + Service)

**File**: `infra/k8s/expiration-redis-depl.yaml`

| Resource | Field | Value |
|----------|-------|-------|
| Deployment | Container Image | `redis` |
| Deployment | Replicas | 1 |
| Service | Name | `expiration-redis-srv` |
| Service | Port | 6379 |
