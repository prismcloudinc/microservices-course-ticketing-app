---
service_id: ticketing-app/payments
service_name: payments
prompt_id: 04-business-logic
prompt_label: Business Logic Extraction
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.23012000000000002
schema_version: "2.0"
---

# Business Processes: Payments Service

## Process: Create Payment

**Trigger**: HTTP POST request to `/api/payments`

**Input**: JSON request body containing:
- `token` (string): Stripe payment token generated on frontend
- `orderId` (string): MongoDB ObjectId of the order to pay for

**Preconditions**:
1. User must be authenticated (valid JWT in session cookie)
2. Request body must contain non-empty `token` and `orderId` fields
3. An Order document with matching `orderId` must exist in the local payments database
4. The authenticated user's ID must match the `userId` on the Order document
5. The Order's `status` must not be `Cancelled`

**Steps**:

1. `requireAuthMiddleware` in `@b.anik/ticketing-common` — Checks if `req.currentUser` is set. If not, throws `NotAuthorizedError`.

2. `body('token').not().isEmpty()` and `body('orderId').not().isEmpty()` in `src/routes/new.ts:21-22` — Express-validator middleware validates that both fields are non-empty strings.

3. `validateRequestMiddleware` in `@b.anik/ticketing-common` — Collects validation errors from express-validator. If errors exist, throws `BadRequestError` with error details.

4. `Order.findById(orderId)` in `src/routes/new.ts:27` — Queries MongoDB for an Order document with `_id` matching `orderId`.
   - **Database read**: `orders` collection, query by `_id`
   - If no order found: throws `NotFoundError` (HTTP 404)

5. Ownership check in `src/routes/new.ts:32-34` — Compares `order.userId` against `req.currentUser!.id`.
   - If `order.userId !== req.currentUser!.id`: throws `NotAuthorizedError` (HTTP 401)

6. Status check in `src/routes/new.ts:35-37` — Evaluates `order.status === OrderStatus.Cancelled`.
   - If order is cancelled: throws `BadRequestError` with message `'Cannot pay for a cancelled order'` (HTTP 400)

7. `stripe.charges.create()` in `src/routes/new.ts:39-43` — Calls Stripe API to create a charge.
   - **External call**: Stripe Charges API
   - **Parameters sent**:
     - `currency`: `'usd'` (hardcoded literal)
     - `amount`: `order.price * 100` (converts dollars to cents for Stripe)
     - `source`: `token` (from request body)

8. Payment verification in `src/routes/new.ts:45-47` — Checks `charge.paid` property from Stripe response.
   - If `charge.paid` is falsy: throws `BadRequestError` with message `'Payment failed'` (HTTP 400)

9. `Payment.build()` in `src/routes/new.ts:49-53` — Creates a new Payment document instance.
   - **Fields set**:
     - `order`: reference to the Order document
     - `stripeId`: `charge.id` from Stripe response
     - `userId`: `order.userId`

10. `payment.save()` in `src/routes/new.ts:54` — Persists the Payment document to MongoDB.
    - **Database write**: `payments` collection
    - **Fields written**: `order` (ObjectId ref), `stripeId` (string), `userId` (string)

11. `PaymentCompletePublisher.publish()` in `src/routes/new.ts:56-58` — Publishes event to NATS Streaming.
    - **Event published**: Subject `payment:complete` via `Subjects.PaymentComplete`
    - **Destination**: NATS Streaming server
    - **Payload**: `{ orderId: payment.order.id }`

12. Response in `src/routes/new.ts:60` — Returns HTTP 201 with JSON body.

**Business rules**:
- A user can only pay for orders they own (enforcement: `order.userId !== req.currentUser!.id` comparison at `src/routes/new.ts:32`)
- Cancelled orders cannot be paid for (enforcement: `order.status === OrderStatus.Cancelled` check at `src/routes/new.ts:35`)
- Stripe amounts are in cents, not dollars (enforcement: `order.price * 100` multiplication at `src/routes/new.ts:41`)
- Payment is only considered successful if `charge.paid` is truthy (enforcement: `!charge.paid` check at `src/routes/new.ts:45`)
- (inferred) The `payment:complete` event notifies other services (likely the Orders service) to mark the order as complete

**Output**: HTTP 201 response with JSON body:
```json
{
  "success": true,
  "payment": {
    "id": "<payment_id>",
    "order": "<order_id>",
    "stripeId": "<stripe_charge_id>",
    "userId": "<user_id>"
  }
}
```

**Post-conditions**:
1. A new Payment document exists in the `payments` collection
2. A `payment:complete` event has been published to NATS

**Error scenarios**:
| Condition | Error Class | HTTP Status | Message |
|-----------|-------------|-------------|---------|
| No valid JWT in session | `NotAuthorizedError` | 401 | "Not authorized" |
| Missing/empty `token` or `orderId` | `BadRequestError` | 400 | Validation errors array |
| Order not found by ID | `NotFoundError` | 404 | "Not found" |
| Order belongs to different user | `NotAuthorizedError` | 401 | "Not authorized" |
| Order status is `Cancelled` | `BadRequestError` | 400 | "Cannot pay for a cancelled order" |
| Stripe charge returns `paid: false` | `BadRequestError` | 400 | "Payment failed" |

---

## Process: List User Payments

**Trigger**: HTTP GET request to `/api/payments`

**Input**: None (user ID extracted from JWT in session cookie)

**Preconditions**:
1. User must be authenticated (valid JWT in session cookie)

**Steps**:

1. `requireAuthMiddleware` in `@b.anik/ticketing-common` — Checks if `req.currentUser` is set. If not, throws `NotAuthorizedError`.

2. `Payment.find({ userId: req.currentUser!.id }).populate('order')` in `src/routes/show.ts:14-16` — Queries MongoDB for all Payment documents belonging to the authenticated user.
   - **Database read**: `payments` collection, query by `userId`
   - **Population**: Resolves `order` reference from `orders` collection
   - Returns empty array if no payments found (not an error)

3. Response in `src/routes/show.ts:18` — Returns HTTP 200 with payments array.

**Business rules**:
- A user can only retrieve their own payments (enforcement: query filter `userId: req.currentUser!.id` at `src/routes/show.ts:15`)
- (inferred) Payment records include their associated order details via population

**Output**: HTTP 200 response with JSON body:
```json
{
  "payments": [
    {
      "id": "<payment_id>",
      "order": { /* populated order document */ },
      "stripeId": "<stripe_charge_id>",
      "userId": "<user_id>"
    }
  ]
}
```

**Post-conditions**: None (read-only operation)

**Error scenarios**:
| Condition | Error Class | HTTP Status | Message |
|-----------|-------------|-------------|---------|
| No valid JWT in session | `NotAuthorizedError` | 401 | "Not authorized" |

---

## Process: Get Single Payment

**Trigger**: HTTP GET request to `/api/payments/:id`

**Input**: URL parameter `id` (MongoDB ObjectId of the payment)

**Preconditions**:
1. User must be authenticated (valid JWT in session cookie)
2. `id` parameter must be a valid MongoDB ObjectId format

**Steps**:

1. `requireAuthMiddleware` in `@b.anik/ticketing-common` — Checks if `req.currentUser` is set. If not, throws `NotAuthorizedError`.

2. `param('id').not().isEmpty().custom(isValidObjectId)` in `src/routes/show.ts:24` — Validates the `id` URL parameter.
   - Checks parameter is non-empty
   - Validates against `mongoose.isValidObjectId()`
   - Error message: `'Payment ID is required'`

3. `validateRequestMiddleware` in `@b.anik/ticketing-common` — Collects validation errors. If errors exist, throws `BadRequestError`.

4. `Payment.findOne({ _id: req.params.id, userId: req.currentUser!.id }).populate('order')` in `src/routes/show.ts:27-30` — Queries MongoDB for a specific payment.
   - **Database read**: `payments` collection, query by `_id` AND `userId`
   - **Population**: Resolves `order` reference from `orders` collection
   - If no payment found (either doesn't exist or belongs to different user): throws `NotFoundError`

5. Response in `src/routes/show.ts:36` — Returns HTTP 200 with payment details.

**Business rules**:
- A user can only retrieve their own payments (enforcement: compound query filter `{ _id, userId }` at `src/routes/show.ts:28-29`)
- Payment ID must be a valid MongoDB ObjectId format (enforcement: `isValidObjectId` custom validator at `src/routes/show.ts:24`)
- (inferred) Returns 404 for both non-existent payments and payments belonging to other users, preventing enumeration attacks

**Output**: HTTP 200 response with JSON body:
```json
{
  "payment": {
    "id": "<payment_id>",
    "order": { /* populated order document */ },
    "stripeId": "<stripe_charge_id>",
    "userId": "<user_id>"
  }
}
```

**Post-conditions**: None (read-only operation)

**Error scenarios**:
| Condition | Error Class | HTTP Status | Message |
|-----------|-------------|-------------|---------|
| No valid JWT in session | `NotAuthorizedError` | 401 | "Not authorized" |
| Invalid ObjectId format | `BadRequestError` | 400 | "Payment ID is required" |
| Payment not found or belongs to another user | `NotFoundError` | 404 | "Not found" |

---

## Process: Handle Order Created Event

**Trigger**: NATS Streaming event on subject `order:created` (`Subjects.OrderCreated`)

**Input**: `OrderCreatedEvent['data']` payload containing:
- `id` (string): Order ID from Orders service
- `version` (number): Optimistic concurrency version
- `status` (OrderStatus): Order status enum value
- `userId` (string): User who created the order
- `expiresAt` (string): ISO timestamp when order reservation expires
- `ticket.id` (string): Ticket ID
- `ticket.price` (number): Ticket price in dollars

**Preconditions**:
1. NATS client must be connected
2. `OrderCreatedListener` must be started via `startListeners()` in `src/events/listeners/index.ts:7`
3. The `id` from the event must be a valid MongoDB ObjectId

**Steps**:

1. `OrderCreatedListener.onMessage()` in `src/events/listeners/order-created-listener.ts:9-30` — Event handler invoked by NATS subscription.

2. `Order.build()` in `src/events/listeners/order-created-listener.ts:12-18` — Creates a new Order document instance.
   - **ObjectId validation**: `isValidObjectId(id)` at `src/models/order.ts:58` — throws `Error` with message `[Order.build] Invalid ObjectId: ${id}` if invalid
   - **Fields set**:
     - `_id`: `data.id` (uses the same ID as the Orders service for cross-service consistency)
     - `version`: `data.version`
     - `price`: `data.ticket.price`
     - `userId`: `data.userId`
     - `status`: `data.status`

3. `order.save()` in `src/events/listeners/order-created-listener.ts:19` — Persists the Order document to MongoDB.
   - **Database write**: `orders` collection
   - **Fields written**: `_id`, `version`, `price`, `userId`, `status`

4. `PaymentAwaitedPublisher.publish()` in `src/events/listeners/order-created-listener.ts:21-23` — Publishes event to NATS Streaming.
   - **Event published**: Subject `payment:awaited` via `Subjects.PaymentAwaited`
   - **Destination**: NATS Streaming server
   - **Payload**: `{ orderId: order.id }`

5. Promise resolution in `src/events/listeners/order-created-listener.ts:25` — Signals successful processing to NATS for message acknowledgment.

**Business rules**:
- The local Order copy uses the same `_id` as the source order in the Orders service (enforcement: `_id: id` at `src/models/order.ts:62-63`)
- Price is extracted from `ticket.price`, not from order-level data (enforcement: `price: data.ticket.price` at `src/events/listeners/order-created-listener.ts:15`)
- (inferred) The `payment:awaited` event notifies other services that the Payments service is ready to accept payment for this order

**Output**: Promise resolves on success, rejects on failure (NATS handles acknowledgment)

**Post-conditions**:
1. A new Order document exists in the local `orders` collection mirroring the source order
2. A `payment:awaited` event has been published to NATS

**Error scenarios**:
| Condition | Behavior | Recovery |
|-----------|----------|----------|
| Invalid ObjectId in `data.id` | Error thrown from `Order.build()` | Promise rejects, NATS will redeliver message |
| MongoDB save failure | Error bubbles up | Promise rejects, NATS will redeliver message |
| NATS publish failure | Error bubbles up | Promise rejects, NATS will redeliver message |

---

## Process: Handle Order Cancelled Event

**Trigger**: NATS Streaming event on subject `order:cancelled` (`Subjects.OrderCancelled`)

**Input**: `OrderCancelledEvent['data']` payload containing:
- `id` (string): Order ID
- `version` (number): New version after cancellation
- `ticket.id` (string): Ticket ID

**Preconditions**:
1. NATS client must be connected
2. `OrderCancelledListener` must be started via `startListeners()` in `src/events/listeners/index.ts:8`
3. An Order document with matching `id` and `version - 1` must exist in local database (optimistic concurrency)

**Steps**:

1. `OrderCancelledListener.onMessage()` in `src/events/listeners/order-cancelled-listener.ts:8-29` — Event handler invoked by NATS subscription.

2. `Order.findByEvent(data)` in `src/events/listeners/order-cancelled-listener.ts:11` — Queries for order using optimistic concurrency.
   - **Database read**: `orders` collection
   - **Query**: `{ _id: event.id, version: event.version - 1 }` at `src/models/order.ts:72-75`
   - If no order found: throws `Error('Order not found')` at `src/events/listeners/order-cancelled-listener.ts:14`

3. `order.set()` in `src/events/listeners/order-cancelled-listener.ts:17-20` — Updates order fields.
   - **Fields updated**:
     - `status`: `OrderStatus.Cancelled`
     - `version`: `data.version`

4. `order.save()` in `src/events/listeners/order-cancelled-listener.ts:21` — Persists updated Order to MongoDB.
   - **Database write**: `orders` collection
   - **Fields written**: `status`, `version`

5. Promise resolution in `src/events/listeners/order-cancelled-listener.ts:23` — Signals successful processing to NATS.

**Business rules**:
- Optimistic concurrency control: order must have `version === event.version - 1` to be updated (enforcement: `findByEvent` query at `src/models/order.ts:72-75`)
- (inferred) This prevents out-of-order event processing and ensures data consistency across services
- Cancelled orders will be rejected by the Create Payment process (cross-process rule)

**Output**: Promise resolves on success, rejects on failure

**Post-conditions**:
1. The local Order document has `status` set to `Cancelled`
2. The local Order document has `version` incremented to match the event

**Error scenarios**:
| Condition | Behavior | Recovery |
|-----------|----------|----------|
| Order not found (wrong ID or version mismatch) | `Error('Order not found')` thrown | Promise rejects, NATS will redeliver message |
| MongoDB save failure | Error bubbles up | Promise rejects, NATS will redeliver message |

---

## Process: Handle Order Updated Event

**Trigger**: NATS Streaming event on subject `order:updated` (`Subjects.OrderUpdated`)

**Input**: `OrderUpdatedEvent['data']` payload containing:
- `id` (string): Order ID
- `version` (number): New version after update
- `status` (OrderStatus): Updated order status

**Preconditions**:
1. NATS client must be connected
2. `OrderUpdatedListener` must be started via `startListeners()` in `src/events/listeners/index.ts:9`
3. An Order document with matching `id` and `version - 1` must exist in local database

**Steps**:

1. `OrderUpdatedListener.onMessage()` in `src/events/listeners/order-updated-listener.ts:8-33` — Event handler invoked by NATS subscription.

2. Destructure event data in `src/events/listeners/order-updated-listener.ts:11` — Extracts `id`, `status`, `version` from event payload.

3. `Order.findByEvent({ id, version })` in `src/events/listeners/order-updated-listener.ts:13-16` — Queries for order using optimistic concurrency.
   - **Database read**: `orders` collection
   - **Query**: `{ _id: id, version: version - 1 }`
   - If no order found: throws `Error('Order not found')` at `src/events/listeners/order-updated-listener.ts:19`

4. `order.set()` in `src/events/listeners/order-updated-listener.ts:22-25` — Updates order fields.
   - **Fields updated**:
     - `version`: `version` from event
     - `status`: `status` from event

5. `order.save()` in `src/events/listeners/order-updated-listener.ts:26` — Persists updated Order to MongoDB.
   - **Database write**: `orders` collection
   - **Fields written**: `version`, `status`

6. Promise resolution in `src/events/listeners/order-updated-listener.ts:28` — Signals successful processing to NATS.

**Business rules**:
- Optimistic concurrency control: order must have `version === event.version - 1` to be updated (enforcement: `findByEvent` query at `src/models/order.ts:72-75`)
- Only `status` and `version` are synchronized from the Orders service; `price` and `userId` remain unchanged
- (inferred) This handles status transitions like `Created` → `AwaitingPayment` → `Complete`

**Output**: Promise resolves on success, rejects on failure

**Post-conditions**:
1. The local Order document has `status` updated to match the event
2. The local Order document has `version` incremented to match the event

**Error scenarios**:
| Condition | Behavior | Recovery |
|-----------|----------|----------|
| Order not found (wrong ID or version mismatch) | `Error('Order not found')` thrown | Promise rejects, NATS will redeliver message |
| MongoDB save failure | Error bubbles up | Promise rejects, NATS will redeliver message |

---

## Process: Health Check

**Trigger**: HTTP GET request to `/api/payments/health`

**Input**: None

**Preconditions**: None (no authentication required)

**Steps**:

1. Route handler in `src/routes/health.ts:5` — Immediately returns HTTP 200 with empty JSON object.

**Business rules**:
- (inferred) This endpoint is used by Kubernetes for liveness probes to determine if the service is responsive
- No database or external service connectivity is validated

**Output**: HTTP 200 response with JSON body: `{}`

**Post-conditions**: None

**Error scenarios**: None handled explicitly

---

## Process: Application Startup

**Trigger**: Container start or manual invocation of `npm start` / `npm run dev`

**Input**: Environment variables:
- `JWT_KEY`: JWT verification secret
- `STRIPE_KEY`: Stripe API secret key
- `MONGO_URI`: MongoDB connection string
- `NATS_URL`: NATS Streaming server URL
- `NATS_CLUSTER_ID`: NATS cluster identifier
- `NATS_CLIENT_ID`: NATS client identifier (typically Kubernetes pod name)

**Preconditions**: All required environment variables must be set

**Steps**:

1. Environment validation in `src/index.ts:8-26` — Checks each required environment variable.
   - If `JWT_KEY` missing: throws `Error('JWT_KEY must be defined')`
   - If `STRIPE_KEY` missing: throws `Error('STRIPE_KEY must be defined')`
   - If `MONGO_URI` missing: throws `Error('MONGO_URI must be defined')`
   - If `NATS_URL` missing: throws `Error('NATS_URL must be defined')`
   - If `NATS_CLUSTER_ID` missing: throws `Error('NATS_CLUSTER_ID must be defined')` (checked twice at lines 24-26 and 28-30)

2. `natsWrapper.connect()` in `src/index.ts:33-37` — Establishes connection to NATS Streaming server.
   - **External connection**: NATS Streaming
   - **Parameters**: `clusterId`, `clientId`, `url`
   - On success: logs `'Connected to NATS!'`

3. NATS close handler registration in `src/index.ts:40-43` — Registers callback for connection close.
   - On NATS close: logs `'NATS connection closed!'` and exits with code `1`

4. Signal handler registration in `src/index.ts:45-47` — Registers graceful shutdown handlers.
   - `SIGINT`: closes NATS client
   - `SIGTERM`: closes NATS client
   - `SIGUSR2`: closes NATS client (for nodemon restart)

5. `startListeners(natsWrapper.client)` in `src/index.ts:49` — Initializes all NATS event listeners.
   - Creates and starts `OrderCreatedListener` at `src/events/listeners/index.ts:7`
   - Creates and starts `OrderCancelledListener` at `src/events/listeners/index.ts:8`
   - Creates and starts `OrderUpdatedListener` at `src/events/listeners/index.ts:9`
   - All listeners use queue group `'payments-service'` at `src/events/listeners/queue-group-name.ts:1`

6. `mongoose.connect()` in `src/index.ts:51` — Establishes connection to MongoDB.
   - **External connection**: MongoDB
   - **Connection string**: from `MONGO_URI` environment variable
   - On success: logs `'Connected to MongoDB!'`

7. `app.listen(3000)` in `src/index.ts:58-60` — Starts HTTP server.
   - **Port**: `3000` (hardcoded literal)
   - On success: logs `'Listening on port 3000!'`

**Business rules**:
- All event listeners subscribe to queue group `'payments-service'` (inferred: ensures only one instance processes each message in a scaled deployment)
- Application exits with code `1` on NATS connection loss (enforcement: `process.exit(1)` at `src/index.ts:42`)
- Graceful shutdown closes NATS connection on process termination signals

**Output**: Running Express server on port 3000, connected to NATS and MongoDB

**Post-conditions**:
1. NATS client connected and event listeners active
2. MongoDB connection established
3. HTTP server accepting requests on port 3000

**Error scenarios**:
| Condition | Behavior | Recovery |
|-----------|----------|----------|
| Missing environment variable | Throws Error with variable name | Process crashes before starting |
| NATS connection failure | Error caught in try/catch | Logs error, calls `process.exit(1)` at `src/index.ts:55` |
| MongoDB connection failure | Error caught in try/catch | Logs error, calls `process.exit(1)` at `src/index.ts:55` |
| NATS connection closed after startup | Close handler fires | Logs message, calls `process.exit(1)` at `src/index.ts:42` |

---

## Data Flow Summary

### Inbound Events (Consumed)

| Event Subject | Publisher Service | Consumer Listener | Local Action |
|---------------|-------------------|-------------------|--------------|
| `order:created` | Orders Service (inferred) | `OrderCreatedListener` | Creates local Order copy, publishes `payment:awaited` |
| `order:cancelled` | Orders Service (inferred) | `OrderCancelledListener` | Updates local Order status to `Cancelled` |
| `order:updated` | Orders Service (inferred) | `OrderUpdatedListener` | Updates local Order status and version |

### Outbound Events (Published)

| Event Subject | Publisher | Trigger | Payload |
|---------------|-----------|---------|---------|
| `payment:awaited` | `PaymentAwaitedPublisher` | After `order:created` processing | `{ orderId }` |
| `payment:complete` | `PaymentCompletePublisher` | After successful payment creation | `{ orderId }` |

### Optimistic Concurrency Pattern

The `Order.findByEvent()` method at `src/models/order.ts:68-76` implements optimistic concurrency control:

```typescript
orderSchema.statics.findByEvent = async (event: {
  id: string;
  version: number;
}): Promise<OrderDoc | null> => {
  return Order.findOne({
    _id: event.id,
    version: event.version - 1,
  });
};
```

This ensures events are processed in order by requiring the local document's version to be exactly one less than the incoming event's version. If an event arrives out of order, the query returns `null`, the listener throws an error, and NATS redelivers the message later.
