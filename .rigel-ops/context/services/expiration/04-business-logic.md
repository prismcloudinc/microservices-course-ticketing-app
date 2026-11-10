---
service_id: ticketing-app/expiration
service_name: expiration
prompt_id: 04-business-logic
prompt_label: Business Logic Extraction
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.16779500000000003
schema_version: "2.0"
---

# Business Processes: expiration Service

## Process: Service Initialization

**Trigger**: Process start via `pnpm run dev` (uses `nodemon src/index.ts`) or `pnpm start` (uses `ts-node src/index.ts`)

**Input**: Environment variables required at startup:
- `NATS_URL` (string, required)
- `NATS_CLUSTER_ID` (string, required)
- `NATS_CLIENT_ID` (string, required)
- `REDIS_HOST` (string, used by Bull queue at module load time)

**Preconditions**: 
- NATS Streaming Server must be reachable at `NATS_URL`
- Redis must be reachable at `REDIS_HOST` (used by Bull queue when module is loaded)

**Steps**:

1. `start() in src/index.ts` — Validates environment variables before any connections are made:
   - If `process.env.NATS_URL` is falsy: throws `Error('NATS_URL must be defined')` (line 6)
   - If `process.env.NATS_CLUSTER_ID` is falsy: throws `Error('NATS_CLUSTER_ID must be defined')` (line 10)
   - Note: There is a duplicate check for `NATS_CLUSTER_ID` at line 13-14 (appears to be a bug — `NATS_CLIENT_ID` is not validated)

2. `natsWrapper.connect() in src/nats-wrapper.ts` — Establishes connection to NATS Streaming Server:
   - Calls `nats.connect(clusterId, clientId, { url })` (line 16) with the three environment variables
   - Returns a Promise that:
     - Resolves on `'connect'` event (line 19)
     - Rejects on `'error'` event (line 21)

3. `start() in src/index.ts` (continued) — Registers event handlers for the NATS client:
   - Registers `'close'` event handler on `natsWrapper.client` that logs "NATS connection closed!" and calls `process.exit(1)` (lines 25-28)

4. `start() in src/index.ts` (continued) — Registers OS signal handlers for graceful shutdown:
   - `SIGINT`: calls `natsWrapper.client.close()` (line 30)
   - `SIGTERM`: calls `natsWrapper.client.close()` (line 31)
   - `SIGUSR2`: calls `natsWrapper.client.close()` (line 32)

5. `startListeners(client) in src/events/listeners/index.ts` — Instantiates and starts event listeners:
   - Creates `new OrderCreatedListener(client)` (line 5)
   - Calls `.listen()` on the listener instance

6. `listen() in base-listener (from nats-test/src/events/base-listener.ts, lines 33-50)` — Subscribes to NATS subject:
   - Creates subscription via `this.client.subscribe(this.subject, this.queueGroupName, this.subscriptionOptions())` (lines 34-38)
   - Subscription options (configured in `subscriptionOptions()` at lines 19-30):
     - `setDeliverAllAvailable()` — delivers all historical messages
     - `setManualAckMode(true)` — requires explicit acknowledgment
     - `setAckWait(5000)` — 5 second timeout for acknowledgment (default `ackWait`)
     - `setDurableName(queueGroupName)` — uses `'expiration-service'` for durable subscription
   - Registers `'message'` event handler that:
     - Logs message receipt with subject, queue group, and sequence number
     - Parses message data as JSON via `parseMessage(msg)` (lines 53-57)
     - Calls `this.onMessage(parsedData, msg)` (the subclass implementation)
     - On Promise resolve: calls `msg.ack()` to acknowledge
     - On Promise reject: calls `console.error` (no re-queue mechanism — message will timeout and redeliver after `ackWait`)

**Business rules**:
- If `NATS_URL` is not set, the service fails to start with an error
- If `NATS_CLUSTER_ID` is not set, the service fails to start with an error
- `NATS_CLIENT_ID` is not validated but is required by NATS — if missing, the `nats.connect()` call will fail (behavior delegated to `node-nats-streaming` library)
- When the NATS connection closes (for any reason), the process exits with code `1`
- Signal handlers (`SIGINT`, `SIGTERM`, `SIGUSR2`) trigger graceful NATS disconnection, which then triggers `process.exit(1)` via the `'close'` handler

**Output**: None (no response — this is a startup process)

**Post-conditions**:
- NATS client is connected and stored in `natsWrapper._client` singleton
- `OrderCreatedListener` is subscribed to `Subjects.OrderCreated` with queue group `'expiration-service'`
- Signal handlers are registered for graceful shutdown
- Bull queue `order:expiration` is initialized with connection to Redis at `REDIS_HOST` (happens at module load time when `src/queues/expiration-queue.ts` is imported)

**Error scenarios**:
- **Missing NATS_URL**: throws `Error('NATS_URL must be defined')`, caught in try/catch at line 35-38, logs error and calls `process.exit(1)`
- **Missing NATS_CLUSTER_ID**: throws `Error('NATS_CLUSTER_ID must be defined')`, caught in try/catch, logs error and calls `process.exit(1)`
- **NATS connection error**: Promise rejection from `natsWrapper.connect()` propagates to catch block (line 35), logs error and calls `process.exit(1)`
- **NATS client access before connect**: `natsWrapper.client` getter throws `Error('Cannot access NATS client before connecting')` if `_client` is undefined (`src/nats-wrapper.ts`, line 9)

---

## Process: Order Expiration Scheduling

**Trigger**: `OrderCreatedEvent` received on NATS subject `Subjects.OrderCreated` (from `@b.anik/ticketing-common`)

**Input**: `OrderCreatedEvent['data']` — the event payload containing order information:
- `id`: string — the order ID
- `version`: number — the order version (not used by this service)
- `status`: OrderStatus — the order status (not used by this service)
- `ticket`: `{ id: string, price: number }` — ticket information (not used by this service)
- `userId`: string — the user ID (not used by this service)
- `expiresAt`: string (optional) — ISO 8601 timestamp when the order should expire

**Preconditions**:
- Service must be running with active NATS subscription
- Redis must be available for the Bull queue

**Steps**:

1. `listen() handler in base-listener` — Receives raw NATS message and parses it:
   - Raw message data is extracted via `msg.getData()` (line 54)
   - Data is parsed as JSON: if string, via `JSON.parse(data)`, otherwise via `JSON.parse(data.toString('utf-8'))` (line 56)
   - Logs: `>>>[${this.subject}][${this.queueGroupName}]: message#${msg.getSequence()} received`

2. `onMessage(data, msg) in OrderCreatedListener (src/events/listeners/order-created-listener.ts, lines 9-33)` — Processes the order created event:
   - **If `data.expiresAt` is falsy** (undefined, null, or empty string) at line 12:
     - Immediately resolves the Promise without scheduling a job (lines 13-14)
     - Comment: "if the order has no expiration date, we don't need to schedule an expiration"
   - **If `data.expiresAt` has a value**:
     - Calculates delay in milliseconds: `new Date(data.expiresAt).getTime() - new Date().getTime()` (line 17)
     - Note: If `expiresAt` is in the past, `delay` will be negative — Bull handles this by executing the job immediately

3. `expirationQueue.add() in src/queues/expiration-queue.ts` (called at line 19-26) — Adds a delayed job to the Bull queue:
   - Job payload: `{ orderId: data.id }`
   - Job options: `{ delay: <calculated delay in milliseconds> }`
   - Bull stores this job in Redis with the configured delay

4. `listen() handler in base-listener (continued)` — Acknowledges the message:
   - On `onMessage()` Promise resolve: calls `msg.ack()` (line 48)
   - On `onMessage()` Promise reject: logs error via `console.error` (line 49), message is not acknowledged

**Business rules**:
- Orders without an `expiresAt` field are ignored — no expiration job is scheduled (inferred: orders that don't expire, possibly already-completed orders)
- The delay is calculated as `expiresAt timestamp - current timestamp`, in milliseconds
- If the `expiresAt` timestamp is in the past, the delay will be negative — Bull processes negative delays immediately (delegated to Bull library behavior)
- The job payload contains only `orderId` — no other order data is stored in the queue
- Message acknowledgment happens only after the job is successfully added to the queue

**Output**: 
- Promise resolution (void) — indicates successful processing
- NATS message acknowledgment sent via `msg.ack()`

**Post-conditions**:
- If `expiresAt` was present: A job with payload `{ orderId: string }` is stored in the Redis-backed Bull queue `order:expiration` with the calculated delay
- If `expiresAt` was absent: No job is created, no state change

**Error scenarios**:
- **Redis unavailable**: `expirationQueue.add()` will reject, causing `onMessage()` to reject. The catch block (line 29-31) calls `reject(err)`. The message is not acknowledged, and NATS will redeliver after `ackWait` (5 seconds)
- **Invalid expiresAt format**: If `new Date(data.expiresAt)` cannot parse the string, it returns `Invalid Date`, and `.getTime()` returns `NaN`. The delay becomes `NaN`, and behavior is delegated to Bull (likely immediate execution or error)
- **JSON parse error in base-listener**: If message data is not valid JSON, `JSON.parse()` throws. Error propagates to the `'message'` handler catch block, logged via `console.error`, message not acknowledged

---

## Process: Order Expiration Execution

**Trigger**: Bull queue `order:expiration` job delay has elapsed — Bull executes the processor callback

**Input**: Bull `Job<Payload>` object where `Payload` is `{ orderId: string }`

**Preconditions**:
- Expiration job was previously scheduled via `OrderCreatedListener`
- The configured delay has elapsed
- NATS client must be connected (for publishing)

**Steps**:

1. `expirationQueue.process() callback in src/queues/expiration-queue.ts (lines 15-19)` — Bull invokes the processor when the job's delay has elapsed:
   - Extracts `orderId` from `job.data.orderId`

2. `new ExpirationCompletePublisher(natsWrapper.client).publish() in src/queues/expiration-queue.ts (line 16-18)` — Creates publisher and publishes event:
   - Instantiates `ExpirationCompletePublisher` with the NATS client singleton
   - Calls `.publish({ orderId: job.data.orderId })`

3. `publish(data) in base-publisher (nats-test/src/events/base-publisher.ts, lines 14-26)` — Publishes event to NATS:
   - Subject: `Subjects.ExpirationComplete` (from `@b.anik/ticketing-common`, defined in `ExpirationCompletePublisher` at `src/events/publishers/expiration-complete-publisher.ts` line 4)
   - Data: JSON stringified payload `{ orderId: string }` (line 16)
   - Returns a Promise that:
     - Resolves on successful publish (callback with no error)
     - Rejects on publish error
   - Logs: `>>>[${this.subject}]: event published`

**Business rules**:
- Each expiration job results in exactly one `ExpirationComplete` event being published
- The event payload contains only `orderId` — the consumer (Orders service) is responsible for looking up the order and determining what action to take
- The expiration service does not check if the order still exists or its current status — it simply publishes the expiration signal

**Output**: 
- `ExpirationCompleteEvent` published to NATS on subject `Subjects.ExpirationComplete`
- Event payload: `{ orderId: string }`

**Post-conditions**:
- Bull job is marked as completed (implicit on processor callback completion)
- `ExpirationComplete` event is delivered to all subscribers (including the Orders service's `ExpirationCompleteListener`)

**Error scenarios**:
- **NATS publish failure**: `publish()` rejects with the error. Bull processor callback throws, and Bull may retry the job depending on its configuration (default: no automatic retry configured in this codebase — behavior delegated to Bull defaults)
- **NATS client not connected**: `natsWrapper.client` getter throws `Error('Cannot access NATS client before connecting')` if the connection was lost. This would cause the Bull job to fail.

---

## Process: Graceful Shutdown

**Trigger**: OS signal received: `SIGINT`, `SIGTERM`, or `SIGUSR2`

**Input**: None (signal is received by the process)

**Preconditions**: Service is running with active NATS connection

**Steps**:

1. `Signal handler in src/index.ts (lines 30-32)` — Responds to the signal:
   - All three signals (`SIGINT`, `SIGTERM`, `SIGUSR2`) trigger the same action: `natsWrapper.client.close()`

2. `client.close() from node-nats-streaming` — Closes the NATS connection:
   - Disconnects from NATS Streaming Server
   - Emits `'close'` event on the client

3. `'close' event handler in src/index.ts (lines 25-28)` — Handles connection closure:
   - Logs: `'NATS connection closed!'`
   - Calls `process.exit(1)`

**Business rules**:
- All three standard termination signals trigger the same shutdown behavior
- The NATS connection is closed first, which then triggers the `'close'` handler
- Process exits with code `1` (non-zero, typically indicating error — though in this case it's a controlled shutdown)
- Note: Bull queue shutdown is not explicitly handled — any in-flight or pending jobs remain in Redis and would be picked up if the service restarts (delegated to Bull's default behavior)

**Output**: None (process terminates)

**Post-conditions**:
- NATS connection is closed
- Process has terminated with exit code `1`
- Bull queue state persists in Redis (not cleared on shutdown)

**Error scenarios**:
- **NATS client not accessible**: If `natsWrapper.client` throws (client not initialized), the signal handler will fail. However, this scenario is unlikely since signals are only registered after successful connection.

---

## Downstream Effect: Order Cancellation (Context from Orders Service)

This documents how the `ExpirationComplete` event is consumed. This code is in the **orders** service (`services/orders/src/events/listeners/expiration-complete-listener.ts`), not the expiration service, but is included for full process understanding.

**Trigger**: `ExpirationCompleteEvent` received by Orders service

**Input**: `{ orderId: string }`

**Steps** (in Orders service):

1. `ExpirationCompleteListener.onMessage() in services/orders/src/events/listeners/expiration-complete-listener.ts`:
   - Fetches order: `Order.findById(data.orderId).populate('ticket')` (line 14)
   - **If order not found**: throws `Error('Order not found')` (line 17)
   - **If order status is `OrderStatus.Cancelled` or `OrderStatus.Complete`** (line 20):
     - Resolves immediately without changes (inferred: order was already finalized before expiration fired)
   - **Otherwise**:
     - Sets `order.status = OrderStatus.Cancelled` (line 25)
     - Calls `order.save()` (line 26)
     - Publishes `OrderCancelledEvent` with `{ id, version, ticket: { id } }` (lines 28-33)

**Business rules** (in Orders service, for context):
- Expiration only affects orders that are still in a non-finalized state (not `Cancelled`, not `Complete`)
- If an order was completed (paid) or cancelled before the expiration timer fired, the expiration event is ignored
- Expiration triggers a cascade: the `OrderCancelled` event will notify other services (e.g., Tickets service releases the ticket, Payments service cancels pending payments)
