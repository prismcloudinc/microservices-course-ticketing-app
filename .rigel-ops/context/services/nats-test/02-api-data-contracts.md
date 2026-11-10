---
service_id: ticketing-app/nats-test
service_name: nats-test
prompt_id: 02-api-data-contracts
prompt_label: API & Data Contracts
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.10749500000000001
schema_version: "2.0"
---

# API & Data Contract Specification: nats-test

## Overview

`nats-test` is a standalone NATS Streaming demonstration package—not an HTTP API service. It contains **no HTTP endpoints** and **no database**. The package demonstrates typed event-based messaging patterns using the `node-nats-streaming` library.

This document covers:
- Part A: HTTP API Endpoints — **None exist**
- Part B: Database Schema — **None exists**
- Part C: Type Definitions & DTOs — Event contracts and base classes

---

## Part A: HTTP API Endpoints

**No HTTP endpoints exist in this package.**

This package does not use Express, Fastify, or any HTTP framework. It is a CLI-invoked demonstration of NATS Streaming pub/sub patterns with two entry points:

| Entry Point | Script | Invocation |
|-------------|--------|------------|
| Publisher | `src/publisher.ts` | `pnpm run publish` |
| Listener | `src/listener.ts` | `pnpm run listen` |

Both connect directly to a NATS Streaming server via TCP at `localhost:4222`.

---

## Part B: Database Schema

**No database is used by this package.**

This package does not include any ORM, database driver, migrations, or data persistence layer.

---

## Part C: Type Definitions & DTOs

### Enums

#### Subjects
- **File**: `nats-test/src/events/subjects.ts:1-5`
- **Purpose**: Defines NATS Streaming channel/subject names for event routing

| Value | String Literal | Description |
|-------|----------------|-------------|
| `TicketCreated` | `'ticket:created'` | Channel for ticket creation events |
| `TicketUpdated` | `'ticket:updated'` | Channel for ticket update events |
| `OrderUpdated` | `'order:updated'` | Channel for order update events (defined but not implemented) |

```typescript
export enum Subjects {
  TicketCreated = 'ticket:created',
  TicketUpdated = 'ticket:updated',
  OrderUpdated = 'order:updated',
}
```

---

### Event Interfaces

#### TicketCreatedEvent
- **File**: `nats-test/src/events/ticket-created-event.ts:3-10`
- **Purpose**: Contract for ticket creation event payload
- **Used by**: `TicketCreatedPublisher`, `TicketCreatedListener`

```typescript
interface TicketCreatedEvent {
  subject: Subjects.TicketCreated;
  data: {
    id: string;
    title: string;
    price: number;
  };
}
```

| Field | Type | Description |
|-------|------|-------------|
| `subject` | `Subjects.TicketCreated` | Literal enum value `'ticket:created'` |
| `data.id` | `string` | Unique identifier for the created ticket |
| `data.title` | `string` | Title/name of the ticket (e.g., event name) |
| `data.price` | `number` | Price of the ticket |

---

#### TicketUpdatedEvent
- **File**: `nats-test/src/events/ticket-updated-event.ts:3-10`
- **Purpose**: Contract for ticket update event payload
- **Used by**: `TicketUpdatedListener`

```typescript
interface TicketUpdatedEvent {
  subject: Subjects.TicketUpdated;
  data: {
    id: string;
    title: string;
    price: number;
  };
}
```

| Field | Type | Description |
|-------|------|-------------|
| `subject` | `Subjects.TicketUpdated` | Literal enum value `'ticket:updated'` |
| `data.id` | `string` | Unique identifier for the updated ticket |
| `data.title` | `string` | Updated title of the ticket |
| `data.price` | `number` | Updated price of the ticket |

---

#### Event (Internal Base Interface)
- **File**: `nats-test/src/events/base-listener.ts:4-7` and `nats-test/src/events/base-publisher.ts:4-7`
- **Purpose**: Generic constraint interface for type-safe event handling
- **Used by**: `Listener<T>`, `Publisher<T>` abstract base classes

```typescript
interface Event {
  subject: Subjects;
  data: any;
}
```

| Field | Type | Description |
|-------|------|-------------|
| `subject` | `Subjects` | The NATS channel name (enum value) |
| `data` | `any` | The event payload (typed via generic in concrete implementations) |

---

### Abstract Base Classes

#### Listener\<T extends Event\>
- **File**: `nats-test/src/events/base-listener.ts:9-58`
- **Purpose**: Abstract base class for typed NATS Streaming subscribers with automatic acknowledgment
- **Extended by**: `TicketCreatedListener`, `TicketUpdatedListener`

**Class Members**:

| Member | Type | Description |
|--------|------|-------------|
| `subject` | `T['subject']` | Abstract; the NATS subject/channel to subscribe to |
| `queueGroupName` | `string` | Abstract; queue group name for competing consumers |
| `onMessage` | `(data: T['data'], msg: Message) => Promise<void>` | Abstract; message handler implementation |
| `client` | `Stan` (private) | NATS Streaming client connection |
| `ackWait` | `number` (protected) | Acknowledgment timeout in ms (default: 5000) |

**Subscription Options** (`subscriptionOptions()` method at line 19-30):
- `setDeliverAllAvailable()`: Delivers all historical messages from the beginning
- `setManualAckMode(true)`: Requires explicit message acknowledgment
- `setAckWait(this.ackWait)`: Sets max time to wait for ack (default 5 seconds)
- `setDurableName(this.queueGroupName)`: Uses queue group name as durable subscription name

**Message Processing** (`listen()` method at line 33-51):
1. Subscribes to the subject with queue group and options
2. On message receipt, logs sequence number
3. Parses message data as JSON (handles both string and Buffer)
4. Calls `onMessage()` handler
5. On successful handler completion, acknowledges the message (`msg.ack()`)
6. On handler error, logs to console (message will be redelivered after ackWait)

---

#### Publisher\<T extends Event\>
- **File**: `nats-test/src/events/base-publisher.ts:9-27`
- **Purpose**: Abstract base class for typed NATS Streaming publishers with promise-based API
- **Extended by**: `TicketCreatedPublisher`

**Class Members**:

| Member | Type | Description |
|--------|------|-------------|
| `subject` | `T['subject']` | Abstract; the NATS subject/channel to publish to |
| `client` | `Stan` (protected) | NATS Streaming client connection |

**Publish Method** (`publish(data: T['data'])` at line 14-26):
- Accepts typed event data matching `T['data']`
- Serializes data to JSON string
- Publishes to the subject via NATS client
- Returns a Promise that resolves on successful publish or rejects on error
- Logs success message to console

---

### Concrete Implementations

#### TicketCreatedListener
- **File**: `nats-test/src/events/ticket-created-listener.ts:6-15`
- **Purpose**: Listens for `ticket:created` events
- **Extends**: `Listener<TicketCreatedEvent>`

| Property | Value | Description |
|----------|-------|-------------|
| `subject` | `Subjects.TicketCreated` (`'ticket:created'`) | NATS channel |
| `queueGroupName` | `'payments-service'` | Queue group for load balancing |

**Behavior**: Logs received event data to console, then resolves immediately (line 10-13).

---

#### TicketUpdatedListener
- **File**: `nats-test/src/events/ticket-updated-listener.ts:6-15`
- **Purpose**: Listens for `ticket:updated` events
- **Extends**: `Listener<TicketUpdatedEvent>`

| Property | Value | Description |
|----------|-------|-------------|
| `subject` | `Subjects.TicketUpdated` (`'ticket:updated'`) | NATS channel |
| `queueGroupName` | `'payments-service'` | Queue group for load balancing |

**Behavior**: Logs received event data to console, then resolves immediately (line 10-13).

---

#### TicketCreatedPublisher
- **File**: `nats-test/src/events/ticket-created-publisher.ts:5-7`
- **Purpose**: Publishes `ticket:created` events
- **Extends**: `Publisher<TicketCreatedEvent>`

| Property | Value | Description |
|----------|-------|-------------|
| `subject` | `Subjects.TicketCreated` (`'ticket:created'`) | NATS channel |

---

### Entry Point Configurations

#### Publisher Entry Point
- **File**: `nats-test/src/publisher.ts:1-30`

**Connection Configuration** (line 4-6):
| Parameter | Value | Description |
|-----------|-------|-------------|
| Cluster ID | `'ticketing'` | NATS Streaming cluster name |
| Client ID | `'abc'` | Static client identifier |
| URL | `'http://localhost:4222'` | NATS Streaming server address |

**Published Test Data** (line 18-21):
```typescript
{
  id: '123',
  title: 'concert',
  price: 20
}
```

**Signal Handlers** (line 28-30): Graceful shutdown on `SIGINT`, `SIGTERM`, `SIGUSR2`.

---

#### Listener Entry Point
- **File**: `nats-test/src/listener.ts:1-27`

**Connection Configuration** (line 9-11):
| Parameter | Value | Description |
|-----------|-------|-------------|
| Cluster ID | `'ticketing'` | NATS Streaming cluster name |
| Client ID | `randomBytes(4).toString('hex')` | Random 8-character hex string |
| URL | `'http://localhost:4222'` | NATS Streaming server address |

**Subscriptions Started** (line 21-22):
1. `TicketCreatedListener` on `ticket:created`
2. `TicketUpdatedListener` on `ticket:updated`

**Signal Handlers** (line 25-27): Graceful shutdown on `SIGINT`, `SIGTERM`, `SIGUSR2`.

---

## Summary

| Category | Count | Notes |
|----------|-------|-------|
| HTTP Endpoints | 0 | Not an HTTP service |
| Database Tables | 0 | No database |
| Event Types | 2 | `TicketCreatedEvent`, `TicketUpdatedEvent` |
| Abstract Base Classes | 2 | `Listener<T>`, `Publisher<T>` |
| Concrete Listeners | 2 | `TicketCreatedListener`, `TicketUpdatedListener` |
| Concrete Publishers | 1 | `TicketCreatedPublisher` |
| NATS Subjects Defined | 3 | `ticket:created`, `ticket:updated`, `order:updated` (unused) |

This package serves as a **prototype/reference implementation** for NATS Streaming pub/sub patterns. The base classes demonstrate type-safe event handling with generics, durable subscriptions, competing consumers via queue groups, and manual acknowledgment for reliable message delivery.
