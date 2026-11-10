---
service_id: ticketing-app/nats-test
service_name: nats-test
prompt_id: 03-message-event-contracts
prompt_label: Message & Event Contracts
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.107605
schema_version: "2.0"
---

# Message & Event Contract Analysis: nats-test

This document catalogs all message, event, and inter-service communication contracts for the `nats-test` package.

> **Note**: This is a standalone demonstration/test package for NATS Streaming pub/sub patterns. It is not a production service but a learning tool within the ticketing application microservices monorepo.

---

## Part A: Events/Messages PRODUCED by this Service

### Event: `ticket:created`

| Attribute | Value |
|-----------|-------|
| **Destination** | NATS Streaming subject: `ticket:created` |
| **Event key/type** | `ticket:created` (from `Subjects.TicketCreated` enum) |
| **Producer location** | `nats-test/src/events/ticket-created-publisher.ts` → `TicketCreatedPublisher` class; invoked from `nats-test/src/publisher.ts:16-22` |
| **Trigger** | Manual CLI invocation via `pnpm run publish` script (`nats-test/package.json:7`) |

**Payload** (defined in `nats-test/src/events/ticket-created-event.ts:5-9`):
```json
{
  "id": "string — unique ticket identifier",
  "title": "string — ticket title/name",
  "price": "number — ticket price"
}
```

**Sample payload published** (hardcoded in `nats-test/src/publisher.ts:18-21`):
```json
{
  "id": "123",
  "title": "concert",
  "price": 20
}
```

**Headers/attributes**: None set explicitly. The base publisher (`nats-test/src/events/base-publisher.ts:14-26`) serializes data as JSON string via `JSON.stringify(data)` and publishes to the subject.

**Publishing mechanism** (`nats-test/src/events/base-publisher.ts:14-26`):
- Wraps `stan.publish()` in a Promise
- Serializes payload as JSON string
- Logs `>>>[ticket:created]: event published` on success
- Rejects promise on error

**Known consumers**: The `TicketCreatedListener` in this same package (for testing). In a production context, this pattern would be consumed by services in the `payments-service` queue group (inferred from listener implementation).

---

## Part B: Events/Messages CONSUMED by this Service

### Consumer: `ticket:created`

| Attribute | Value |
|-----------|-------|
| **Source** | NATS Streaming subject: `ticket:created` |
| **Consumer group/subscription** | Queue group: `payments-service` |
| **Handler location** | `nats-test/src/events/ticket-created-listener.ts` → `TicketCreatedListener.onMessage()` (lines 10-13) |
| **Event key/type** | `Subjects.TicketCreated` = `'ticket:created'` |

**Expected payload** (from `nats-test/src/events/ticket-created-event.ts:5-9`):
```json
{
  "id": "string — unique ticket identifier",
  "title": "string — ticket title/name", 
  "price": "number — ticket price"
}
```

**Subscription options** (configured in `nats-test/src/events/base-listener.ts:19-31`):
- `setDeliverAllAvailable()` — delivers all historical messages from the beginning
- `setManualAckMode(true)` — requires explicit acknowledgment
- `setAckWait(5000)` — 5 second timeout for acknowledgment (configurable via constructor, default 5000ms)
- `setDurableName('payments-service')` — durable subscription matching queue group name

**Processing logic** (`nats-test/src/events/ticket-created-listener.ts:10-13`):
1. Receives parsed message data (JSON parsed by base class)
2. Logs event data to console: `console.log('Event data:', data)`
3. Returns resolved Promise (no actual processing — demo only)

**Output**: 
- Console log only (demonstration handler)
- No database writes
- No HTTP calls
- No downstream events

**Error handling** (`nats-test/src/events/base-listener.ts:47-49`):
- On `onMessage()` success: calls `msg.ack()` to acknowledge
- On `onMessage()` failure: logs error via `console.error`; message is NOT acknowledged, will be redelivered by NATS Streaming after ackWait timeout

---

### Consumer: `ticket:updated`

| Attribute | Value |
|-----------|-------|
| **Source** | NATS Streaming subject: `ticket:updated` |
| **Consumer group/subscription** | Queue group: `payments-service` |
| **Handler location** | `nats-test/src/events/ticket-updated-listener.ts` → `TicketUpdatedListener.onMessage()` (lines 10-13) |
| **Event key/type** | `Subjects.TicketUpdated` = `'ticket:updated'` |

**Expected payload** (from `nats-test/src/events/ticket-updated-event.ts:5-9`):
```json
{
  "id": "string — unique ticket identifier",
  "title": "string — ticket title/name",
  "price": "number — ticket price"
}
```

**Subscription options** (same as `ticket:created`, configured in `nats-test/src/events/base-listener.ts:19-31`):
- `setDeliverAllAvailable()` — delivers all historical messages
- `setManualAckMode(true)` — requires explicit acknowledgment
- `setAckWait(5000)` — 5 second timeout for acknowledgment
- `setDurableName('payments-service')` — durable subscription

**Processing logic** (`nats-test/src/events/ticket-updated-listener.ts:10-13`):
1. Receives parsed message data (JSON parsed by base class)
2. Logs event data to console: `console.log('Event data:', data)`
3. Returns resolved Promise (no actual processing — demo only)

**Output**:
- Console log only (demonstration handler)
- No database writes
- No HTTP calls
- No downstream events

**Error handling**: Same as `ticket:created` consumer (ack on success, log and leave unacked on failure).

---

## Part C: Workflow Orchestrations

**None.** This package does not implement or invoke any durable workflow systems (Temporal, AWS Step Functions, Inngest, Bull queues, or similar).

---

## Part D: HTTP Calls to Other Services

**None.** This package does not make any outbound HTTP calls. All communication is via NATS Streaming message broker only.

---

## Part E: Webhooks and External API Calls

**None.** This package does not integrate with any external third-party systems.

---

## Additional Notes

### NATS Streaming Connection Configuration

Both publisher and listener connect to NATS Streaming with the following configuration (`nats-test/src/publisher.ts:4-6`, `nats-test/src/listener.ts:9-11`):

| Parameter | Value | Source |
|-----------|-------|--------|
| Cluster ID | `ticketing` | Hardcoded |
| Client ID | `abc` (publisher) or random 8-char hex (listener) | Hardcoded / `crypto.randomBytes(4).toString('hex')` |
| URL | `http://localhost:4222` | Hardcoded |

### Graceful Shutdown

Both entry points handle shutdown signals (`nats-test/src/publisher.ts:28-30`, `nats-test/src/listener.ts:25-27`):
- `SIGINT` → `stan.close()`
- `SIGTERM` → `stan.close()`
- `SIGUSR2` → `stan.close()`

On close, the disconnect handler (`stan.on('close', ...)`) logs the disconnection and calls `process.exit()`.

### Defined but Unused Subject

The `Subjects` enum (`nats-test/src/events/subjects.ts:4`) defines `OrderUpdated = 'order:updated'` which has **no corresponding publisher or listener** implemented in this package.

### Base Class Architecture

The package provides reusable abstract base classes for type-safe event handling:

| Class | Location | Purpose |
|-------|----------|---------|
| `Publisher<T extends Event>` | `nats-test/src/events/base-publisher.ts` | Generic publisher with promisified publish, JSON serialization |
| `Listener<T extends Event>` | `nats-test/src/events/base-listener.ts` | Generic listener with subscription options, message parsing, auto-ack on success |

These classes use TypeScript generics to enforce type safety between event interfaces and their publishers/consumers.
