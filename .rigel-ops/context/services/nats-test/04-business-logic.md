---
service_id: ticketing-app/nats-test
service_name: nats-test
prompt_id: 04-business-logic
prompt_label: Business Logic Extraction
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.17301
schema_version: "2.0"
---

# Business Processes: nats-test

This document describes the business processes in the `nats-test` package, a standalone demonstration of NATS Streaming pub/sub patterns.

---

## Process: Publish Ticket Created Event

**Trigger**: CLI invocation via `pnpm run publish` which executes `nodemon --watch src/publisher.ts src/publisher.ts` (`nats-test/package.json:7`)

**Input**: Hardcoded test data within the script:
```typescript
{
  id: '123',
  title: 'concert',
  price: 20,
}
```
(`nats-test/src/publisher.ts:18-21`)

**Preconditions**:
- NATS Streaming server must be running at `http://localhost:4222` (typically via Kubernetes port-forwarding: `kubectl port-forward <nats-pod-name> 4222:4222` per `nats-test/README.md:9`)
- NATS Streaming cluster named `ticketing` must exist

**Steps**:

1. **`nats.connect()` in `nats-test/src/publisher.ts:4-6`** — Initiates connection to NATS Streaming server
   - Cluster ID: `'ticketing'` (literal)
   - Client ID: `'abc'` (literal)
   - Connection URL: `'http://localhost:4222'` (literal)
   - Returns a `Stan` client instance stored in `stan` variable

2. **`stan.on('connect', ...)` callback in `nats-test/src/publisher.ts:8-26`** — Executes when connection succeeds
   - Logs `'Publisher connected to NATS'` to console (`nats-test/src/publisher.ts:9`)

3. **Register close handler in `nats-test/src/publisher.ts:11-14`** — Sets up disconnect behavior
   - When `stan.on('close')` fires:
     - Logs `'Publisher disconnected from NATS'`
     - Calls `process.exit()` to terminate the process

4. **`new TicketCreatedPublisher(stan)` in `nats-test/src/publisher.ts:16`** — Instantiates the publisher
   - Constructor receives the NATS client (`nats-test/src/events/base-publisher.ts:12`)
   - Sets `subject` property to `Subjects.TicketCreated` which equals `'ticket:created'` (`nats-test/src/events/ticket-created-publisher.ts:6`, `nats-test/src/events/subjects.ts:2`)

5. **`publisher.publish(data)` in `nats-test/src/publisher.ts:17-22`** — Publishes the event
   - Delegates to `Publisher.publish()` in `nats-test/src/events/base-publisher.ts:14-26`
   - Creates a new Promise wrapping the publish operation
   - Calls `this.client.publish(this.subject, JSON.stringify(data), callback)` (`nats-test/src/events/base-publisher.ts:16`)
     - Subject: `'ticket:created'`
     - Payload: `'{"id":"123","title":"concert","price":20}'` (JSON-serialized)
   - If publish succeeds (no error in callback):
     - Logs `'>>>[ticket:created]: event published'` (`nats-test/src/events/base-publisher.ts:21`)
     - Resolves the Promise (`nats-test/src/events/base-publisher.ts:23`)
   - If publish fails (error in callback):
     - Rejects the Promise with the error (`nats-test/src/events/base-publisher.ts:17-18`)

6. **Error handling in `nats-test/src/publisher.ts:23-25`** — Catches publish failures
   - Logs the error to console via `console.log(err)`

**Business rules**:
- Event payload is JSON-serialized before publishing (`JSON.stringify(data)` in `nats-test/src/events/base-publisher.ts:16`)
- Client ID is static (`'abc'`), meaning only one publisher instance can connect with this configuration at a time (NATS Streaming enforces unique client IDs per cluster)
- The publisher does not exit after publishing; it remains connected until a termination signal is received

**Output**: 
- On success: Console output `'>>>[ticket:created]: event published'`
- On failure: Error logged to console

**Post-conditions**:
- A `ticket:created` event with payload `{"id":"123","title":"concert","price":20}` is available on the NATS Streaming server for subscribers
- Publisher remains connected to NATS Streaming until terminated

**Error scenarios**:
| Error | Cause | Handling | Location |
|-------|-------|----------|----------|
| NATS connection failure | Server unreachable at `localhost:4222` | No explicit handler; `stan.on('connect')` callback never fires | `nats-test/src/publisher.ts:8` |
| Publish callback error | NATS Streaming rejects the publish | Promise rejects, caught by try/catch, error logged via `console.log(err)` | `nats-test/src/publisher.ts:23-25` |

---

## Process: Graceful Publisher Shutdown

**Trigger**: Process signals SIGINT, SIGTERM, or SIGUSR2 (`nats-test/src/publisher.ts:28-30`)

**Input**: Operating system signal

**Preconditions**:
- Publisher process is running
- NATS connection is established (or connecting)

**Steps**:

1. **Signal handler in `nats-test/src/publisher.ts:28-30`** — Registered for three signals:
   - `process.on('SIGINT', () => stan.close())` — Ctrl+C
   - `process.on('SIGTERM', () => stan.close())` — Termination request
   - `process.on('SIGUSR2', () => stan.close())` — Nodemon restart signal

2. **`stan.close()` invocation** — Closes the NATS Streaming connection
   - This triggers the `stan.on('close')` callback registered in `nats-test/src/publisher.ts:11-14`

3. **Close callback in `nats-test/src/publisher.ts:11-14`** — Executes on connection close
   - Logs `'Publisher disconnected from NATS'`
   - Calls `process.exit()` to terminate the Node.js process

**Business rules**:
- All three signals (SIGINT, SIGTERM, SIGUSR2) are handled identically — close the NATS connection
- SIGUSR2 is specifically used by nodemon for restart signals (inferred from typical nodemon behavior)
- The process always exits after closing the NATS connection

**Output**: Console message `'Publisher disconnected from NATS'`

**Post-conditions**:
- NATS Streaming connection is closed
- Node.js process terminates with exit code 0 (default)

**Error scenarios**: None explicitly handled. If `stan.close()` fails, the close callback may not fire.

---

## Process: Listen for Ticket Events

**Trigger**: CLI invocation via `pnpm run listen` which executes `nodemon --watch src/listener.ts src/listener.ts` (`nats-test/package.json:8`)

**Input**: None — listener waits passively for events

**Preconditions**:
- NATS Streaming server must be running at `http://localhost:4222`
- NATS Streaming cluster named `ticketing` must exist

**Steps**:

1. **Generate random client ID in `nats-test/src/listener.ts:7`** — Creates unique identifier
   - Uses `randomBytes(4).toString('hex')` from Node.js `crypto` module
   - Produces an 8-character hex string (e.g., `'a1b2c3d4'`)
   - Allows multiple listener instances to connect simultaneously

2. **`nats.connect()` in `nats-test/src/listener.ts:9-11`** — Initiates connection to NATS Streaming server
   - Cluster ID: `'ticketing'` (literal)
   - Client ID: dynamically generated hex string from step 1
   - Connection URL: `'http://localhost:4222'` (literal)

3. **`stan.on('connect', ...)` callback in `nats-test/src/listener.ts:13-23`** — Executes when connection succeeds
   - Logs `'Listener#<clientId> connected to NATS'` (`nats-test/src/listener.ts:14`)

4. **Register close handler in `nats-test/src/listener.ts:16-19`** — Sets up disconnect behavior
   - When `stan.on('close')` fires:
     - Logs `'Listener#<clientId> disconnected from NATS'`
     - Calls `process.exit()` to terminate the process

5. **`new TicketCreatedListener(stan).listen()` in `nats-test/src/listener.ts:21`** — Creates and starts first listener
   - Constructor: stores NATS client, sets `ackWait` to default `5 * 1000` (5000ms) (`nats-test/src/events/base-listener.ts:14-17`)
   - Sets `subject` to `Subjects.TicketCreated` = `'ticket:created'` (`nats-test/src/events/ticket-created-listener.ts:7`)
   - Sets `queueGroupName` to `'payments-service'` (`nats-test/src/events/ticket-created-listener.ts:8`)
   - Calls `listen()` method (detailed in separate process below)

6. **`new TicketUpdatedListener(stan).listen()` in `nats-test/src/listener.ts:22`** — Creates and starts second listener
   - Constructor: stores NATS client, sets `ackWait` to default `5 * 1000` (5000ms)
   - Sets `subject` to `Subjects.TicketUpdated` = `'ticket:updated'` (`nats-test/src/events/ticket-updated-listener.ts:7`)
   - Sets `queueGroupName` to `'payments-service'` (`nats-test/src/events/ticket-updated-listener.ts:8`)
   - Calls `listen()` method (detailed in separate process below)

**Business rules**:
- Each listener instance gets a unique randomly-generated client ID (8-char hex), allowing multiple listeners to run concurrently
- Both listeners use the same queue group `'payments-service'`, enabling load balancing — only one listener in the queue group receives each message (inferred from NATS Streaming queue group semantics)
- The listener process subscribes to both `ticket:created` and `ticket:updated` subjects simultaneously

**Output**: Console message `'Listener#<clientId> connected to NATS'`

**Post-conditions**:
- Two active subscriptions are established:
  1. `ticket:created` with queue group `payments-service`
  2. `ticket:updated` with queue group `payments-service`
- Listener process remains running, waiting for events

**Error scenarios**:
| Error | Cause | Handling | Location |
|-------|-------|----------|----------|
| NATS connection failure | Server unreachable at `localhost:4222` | No explicit handler; `stan.on('connect')` callback never fires | `nats-test/src/listener.ts:13` |

---

## Process: Subscribe to NATS Channel (Base Listener)

**Trigger**: Call to `listen()` method on a concrete `Listener` subclass (`nats-test/src/events/base-listener.ts:33`)

**Input**: 
- `subject`: Channel name (e.g., `'ticket:created'`)
- `queueGroupName`: Queue group identifier (e.g., `'payments-service'`)
- `ackWait`: Acknowledgment timeout in milliseconds (default: `5000`)

**Preconditions**:
- NATS Streaming client (`Stan`) must be connected
- Concrete listener class must define `subject`, `queueGroupName`, and `onMessage()` method

**Steps**:

1. **`subscriptionOptions()` in `nats-test/src/events/base-listener.ts:19-31`** — Builds subscription configuration
   - Calls `this.client.subscriptionOptions()` to get options builder
   - `.setDeliverAllAvailable()` — Deliver all messages from the beginning of the stream (replay mode)
   - `.setManualAckMode(true)` — Requires explicit acknowledgment; messages re-delivered if not acked
   - `.setAckWait(this.ackWait)` — Sets timeout to `5000` ms (default); message redelivered if not acked within this window
   - `.setDurableName(this.queueGroupName)` — Names the durable subscription using the queue group name, enabling message tracking across reconnects

2. **`this.client.subscribe(...)` in `nats-test/src/events/base-listener.ts:34-38`** — Creates the subscription
   - Arguments:
     - `this.subject`: channel name (e.g., `'ticket:created'`)
     - `this.queueGroupName`: queue group (e.g., `'payments-service'`)
     - `this.subscriptionOptions()`: options object from step 1
   - Returns a `Subscription` object

3. **`subscription.on('message', ...)` in `nats-test/src/events/base-listener.ts:40-50`** — Registers message handler
   - Callback receives a `Message` object when an event arrives

**Business rules**:
- `setDeliverAllAvailable()` means new subscribers receive all historical messages — useful for state rebuilding (inferred)
- `setManualAckMode(true)` enables at-least-once delivery — messages are redelivered until explicitly acknowledged
- `setAckWait(5000)` means a message not acknowledged within 5 seconds is redelivered to another subscriber
- `setDurableName()` using the queue group name ensures message tracking survives listener restarts — messages delivered to the durable subscription before the restart won't be redelivered after it

**Output**: A live subscription waiting for messages

**Post-conditions**:
- Subscription is active on the NATS Streaming server
- Message callback is ready to process incoming events

**Error scenarios**: None explicitly handled in `listen()`. If subscription fails, the error propagates.

---

## Process: Handle Incoming NATS Message (Base Listener)

**Trigger**: NATS Streaming delivers a message to the subscribed channel (`nats-test/src/events/base-listener.ts:40`)

**Input**: `Message` object from NATS Streaming containing:
- Raw data (string or Buffer)
- Sequence number
- Acknowledgment methods

**Preconditions**:
- Subscription is active
- Message payload is valid JSON

**Steps**:

1. **Log message receipt in `nats-test/src/events/base-listener.ts:41-43`** — Records arrival
   - Logs `'>>>[<subject>][<queueGroupName>]: message#<sequence> received'`
   - Example: `'>>>[ticket:created][payments-service]: message#42 received'`

2. **`parseMessage(msg)` in `nats-test/src/events/base-listener.ts:53-57`** — Deserializes JSON payload
   - Calls `msg.getData()` to retrieve raw data
   - If data is a string: `JSON.parse(data)` directly
   - If data is a Buffer: `JSON.parse(data.toString('utf-8'))` first converts to string
   - Returns parsed JavaScript object

3. **`this.onMessage(parsedData, msg)` in `nats-test/src/events/base-listener.ts:47`** — Invokes concrete handler
   - Call target: Abstract method implemented by subclass
   - For `TicketCreatedListener` (`nats-test/src/events/ticket-created-listener.ts:10-14`):
     - Logs `'Event data:'` followed by the data object
     - Returns `Promise.resolve()` immediately (no async work)
   - For `TicketUpdatedListener` (`nats-test/src/events/ticket-updated-listener.ts:10-14`):
     - Logs `'Event data:'` followed by the data object
     - Returns `Promise.resolve()` immediately (no async work)

4. **Acknowledge message in `nats-test/src/events/base-listener.ts:48`** — Confirms processing
   - `.then(() => msg.ack())` — On successful `onMessage()` completion, calls `msg.ack()`
   - This tells NATS Streaming the message was successfully processed
   - Prevents message redelivery

5. **Error handling in `nats-test/src/events/base-listener.ts:49`** — Handles processing failures
   - `.catch(console.error)` — If `onMessage()` rejects, logs the error
   - Note: `msg.ack()` is NOT called on error, so the message will be redelivered after `ackWait` timeout (5 seconds)

**Business rules**:
- Message acknowledgment only occurs after `onMessage()` completes successfully (Promise resolves)
- If `onMessage()` throws or rejects, the message is NOT acknowledged — it will be redelivered after the `ackWait` timeout (5000ms)
- Data format must be valid JSON; `JSON.parse()` will throw on malformed data, causing redelivery
- Both string and Buffer payloads are supported; Buffers are converted to UTF-8 strings before parsing

**Output**: 
- Console log: `'>>>[<subject>][<queueGroupName>]: message#<sequence> received'`
- Console log from concrete handler: `'Event data:'` plus the parsed data object

**Post-conditions**:
- On success: Message is acknowledged and won't be redelivered
- On failure: Message remains unacknowledged and will be redelivered after 5000ms

**Error scenarios**:
| Error | Cause | Handling | Location |
|-------|-------|----------|----------|
| JSON parse error | Malformed JSON payload | `JSON.parse()` throws; caught by `.catch(console.error)`; message redelivered | `nats-test/src/events/base-listener.ts:56` |
| `onMessage()` rejection | Handler logic fails | Caught by `.catch(console.error)`; message redelivered | `nats-test/src/events/base-listener.ts:49` |

---

## Process: Graceful Listener Shutdown

**Trigger**: Process signals SIGINT, SIGTERM, or SIGUSR2 (`nats-test/src/listener.ts:25-27`)

**Input**: Operating system signal

**Preconditions**:
- Listener process is running
- NATS connection is established (or connecting)

**Steps**:

1. **Signal handler in `nats-test/src/listener.ts:25-27`** — Registered for three signals:
   - `process.on('SIGINT', () => stan.close())` — Ctrl+C
   - `process.on('SIGTERM', () => stan.close())` — Termination request
   - `process.on('SIGUSR2', () => stan.close())` — Nodemon restart signal

2. **`stan.close()` invocation** — Closes the NATS Streaming connection
   - Triggers the `stan.on('close')` callback registered in `nats-test/src/listener.ts:16-19`

3. **Close callback in `nats-test/src/listener.ts:16-19`** — Executes on connection close
   - Logs `'Listener#<clientId> disconnected from NATS'` (clientId is the random hex string)
   - Calls `process.exit()` to terminate the Node.js process

**Business rules**:
- Same handling logic as publisher shutdown
- All active subscriptions are implicitly closed when the client disconnects
- Due to durable subscription configuration, the listener's position in the message stream is preserved for reconnection

**Output**: Console message `'Listener#<clientId> disconnected from NATS'`

**Post-conditions**:
- NATS Streaming connection is closed
- Both subscriptions (`ticket:created` and `ticket:updated`) are terminated
- Node.js process terminates with exit code 0 (default)
- Durable subscription state is preserved on the NATS Streaming server (inferred from `setDurableName()` configuration)

**Error scenarios**: None explicitly handled.

---

## Event Type Definitions

The package defines the following event types for type-safe pub/sub:

### TicketCreatedEvent (`nats-test/src/events/ticket-created-event.ts`)

```typescript
{
  subject: 'ticket:created',  // Subjects.TicketCreated
  data: {
    id: string,
    title: string,
    price: number
  }
}
```

### TicketUpdatedEvent (`nats-test/src/events/ticket-updated-event.ts`)

```typescript
{
  subject: 'ticket:updated',  // Subjects.TicketUpdated
  data: {
    id: string,
    title: string,
    price: number
  }
}
```

### Subjects Enum (`nats-test/src/events/subjects.ts`)

| Enum Value | String Literal |
|------------|----------------|
| `TicketCreated` | `'ticket:created'` |
| `TicketUpdated` | `'ticket:updated'` |
| `OrderUpdated` | `'order:updated'` |

Note: `OrderUpdated` is defined but has no corresponding event interface or listener implementation in this package.

---

## Summary of External Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| `node-nats-streaming` | `^0.3.2` | NATS Streaming client for pub/sub messaging |

## Infrastructure Requirements

| Resource | Value | Source |
|----------|-------|--------|
| NATS Streaming Cluster | `ticketing` | Hardcoded in `publisher.ts:4` and `listener.ts:9` |
| NATS Streaming URL | `http://localhost:4222` | Hardcoded in `publisher.ts:5` and `listener.ts:10` |
| Monitoring URL | `localhost:8222` | Documented in `README.md:11` |
