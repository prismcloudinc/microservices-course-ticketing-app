---
service_id: ticketing-app/orders
service_name: orders
prompt_id: 04-business-logic
prompt_label: Business Logic Extraction
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.263485
schema_version: "2.0"
---

# Business Processes: Orders Service

## Process: Create Order

**Trigger**: HTTP POST request to `/api/orders`

**Input**: JSON body with `ticketId` (string, must be a valid MongoDB ObjectId)

**Preconditions**:
1. User must be authenticated (JWT cookie must contain valid user session) — enforced by `requireAuthMiddleware` in `services/orders/src/routes/new.ts`, line 20
2. `ticketId` must be a non-empty valid MongoDB ObjectId — enforced by express-validator chain in `services/orders/src/routes/new.ts`, lines 21-25

**Steps**:

1. [`requireAuthMiddleware` in `@b.anik/ticketing-common`] — Checks `req.currentUser` exists; if not, throws `NotAuthorizedError`

2. [Validation chain in `services/orders/src/routes/new.ts`, lines 21-25] — Validates `ticketId`:
   - Trims whitespace from input
   - Checks input is not empty
   - Validates input is a valid MongoDB ObjectId using `isValidObjectId(input)`
   - If validation fails: `validateRequestMiddleware` throws `RequestValidationError` with field `ticketId`

3. [`Ticket.findById` in `services/orders/src/routes/new.ts`, line 31] — Database read: Queries MongoDB `Ticket` collection by `_id` matching `ticketId`
   - If ticket not found: Throws `BadRequestError('Ticket not found')` (line 33)

4. [`ticket.isReserved()` in `services/orders/src/models/ticket.ts`, lines 75-84] — Database read: Queries MongoDB `Order` collection:
   ```javascript
   Order.findOne({
     ticket: this,
     status: { $in: [OrderStatus.Created, OrderStatus.AwaitingPayment, OrderStatus.Complete] }
   })
   ```
   - If an order exists with this ticket and status is `Created`, `AwaitingPayment`, or `Complete`: Returns `true` (ticket is reserved)
   - If no such order exists: Returns `false` (ticket is available)
   - If ticket is reserved: Throws `BadRequestError('Ticket is already reserved')` (line 39)

5. [Expiration calculation in `services/orders/src/routes/new.ts`, lines 43-44] — Calculates expiration timestamp:
   - Creates new Date object
   - Adds `EXPIRATION_WINDOW_SECONDS` (900 seconds = 15 minutes, defined on line 16) to current time

6. [`Order.build()` and `order.save()` in `services/orders/src/routes/new.ts`, lines 47-53] — Database write: Creates new document in `Order` collection:
   - Fields written:
     - `userId`: `req.currentUser.id` (from JWT)
     - `status`: `OrderStatus.Created`
     - `expiresAt`: Calculated timestamp (current time + 15 minutes)
     - `ticket`: Reference to the Ticket document
     - `version`: `0` (initialized by `mongoose-update-if-current` plugin)

7. [`OrderCreatedPublisher.publish()` in `services/orders/src/routes/new.ts`, lines 56-66] — Event published to NATS:
   - Subject: `order:created` (via `Subjects.OrderCreated`)
   - Payload:
     ```javascript
     {
       id: order.id,
       version: order.version,
       status: order.status,
       userId: order.userId,
       expiresAt: order.expiresAt.toISOString(),  // ISO 8601 string
       ticket: {
         id: ticket.id,
         price: ticket.price
       }
     }
     ```

**Business Rules**:
- A ticket is considered "reserved" if there exists an order for that ticket with status `Created`, `AwaitingPayment`, or `Complete` (`services/orders/src/models/ticket.ts`, lines 79-81)
- A ticket is considered "available" if no order exists, OR the only orders for that ticket have status `Cancelled` (inferred from the exclusion of `Cancelled` in the `$in` array)
- Orders expire 15 minutes (900 seconds) after creation (`services/orders/src/routes/new.ts`, line 16)
- Each user can only create orders for available tickets; the same ticket cannot have multiple active orders simultaneously

**Output**: HTTP 201 response with JSON body:
```javascript
{
  order: {
    id: "<order_id>",
    userId: "<user_id>",
    status: "created",
    expiresAt: "<ISO_timestamp>",
    ticket: { id: "<ticket_id>", title: "<title>", price: <number> },
    version: 0
  }
}
```

**Post-conditions**:
1. New `Order` document exists in MongoDB with status `Created`
2. `order:created` event published to NATS Streaming

**Error Scenarios**:
| Condition | Error Class | HTTP Status | Message |
|-----------|-------------|-------------|---------|
| No authentication cookie/JWT | `NotAuthorizedError` | 401 | — |
| Missing or invalid `ticketId` | `RequestValidationError` | 400 | `ticketId must be provided` |
| Ticket does not exist in local replica | `BadRequestError` | 400 | `Ticket not found` |
| Ticket is already reserved | `BadRequestError` | 400 | `Ticket is already reserved` |

---

## Process: List User's Orders

**Trigger**: HTTP GET request to `/api/orders`

**Input**: None (uses authenticated user's ID from session)

**Preconditions**:
1. User must be authenticated — enforced by `requireAuthMiddleware` in `services/orders/src/routes/show.ts`, line 9

**Steps**:

1. [`requireAuthMiddleware` in `@b.anik/ticketing-common`] — Checks `req.currentUser` exists; if not, throws `NotAuthorizedError`

2. [`Order.find()` in `services/orders/src/routes/show.ts`, lines 10-12] — Database read: Queries MongoDB `Order` collection:
   ```javascript
   Order.find({ userId: req.currentUser.id }).populate('ticket')
   ```
   - Filters orders by the authenticated user's ID
   - Populates the `ticket` reference field with full ticket document

**Business Rules**:
- Users can only see their own orders; orders belonging to other users are not returned (inferred from the filter `userId: req.currentUser.id`)
- All order statuses are returned (no status filtering)

**Output**: HTTP 200 response with JSON body:
```javascript
{
  orders: [
    {
      id: "<order_id>",
      userId: "<user_id>",
      status: "<status>",
      expiresAt: "<ISO_timestamp>",
      ticket: { id: "<ticket_id>", title: "<title>", price: <number>, version: <number> },
      version: <number>
    },
    // ... additional orders
  ]
}
```

**Post-conditions**: None (read-only operation)

**Error Scenarios**:
| Condition | Error Class | HTTP Status |
|-----------|-------------|-------------|
| No authentication cookie/JWT | `NotAuthorizedError` | 401 |

---

## Process: View Single Order

**Trigger**: HTTP GET request to `/api/orders/:orderId`

**Input**: URL parameter `orderId` (must be a valid MongoDB ObjectId)

**Preconditions**:
1. User must be authenticated — enforced by `requireAuthMiddleware` in `services/orders/src/routes/show.ts`, line 20
2. `orderId` must be a valid MongoDB ObjectId — enforced by express-validator in `services/orders/src/routes/show.ts`, lines 21-26

**Steps**:

1. [`requireAuthMiddleware` in `@b.anik/ticketing-common`] — Checks `req.currentUser` exists; if not, throws `NotAuthorizedError`

2. [Validation chain in `services/orders/src/routes/show.ts`, lines 21-26] — Validates `orderId`:
   - Trims whitespace
   - Checks not empty
   - Validates is valid MongoDB ObjectId
   - If validation fails: Field error for `orderId`

3. [`Order.findById()` in `services/orders/src/routes/show.ts`, line 30] — Database read: Queries MongoDB `Order` collection by `_id` and populates `ticket` reference
   - If order not found: Throws `NotFoundError()` (line 33)

4. [Ownership check in `services/orders/src/routes/show.ts`, lines 36-38] — Compares `order.userId` with `req.currentUser.id`
   - If not equal: Throws `NotAuthorizedError()` (line 37)

**Business Rules**:
- Users can only view orders they own; attempting to view another user's order results in 401 (not 404, which would leak information about order existence)

**Output**: HTTP 200 response with JSON body:
```javascript
{
  order: {
    id: "<order_id>",
    userId: "<user_id>",
    status: "<status>",
    expiresAt: "<ISO_timestamp>",
    ticket: { id: "<ticket_id>", title: "<title>", price: <number>, version: <number> },
    version: <number>
  }
}
```

**Post-conditions**: None (read-only operation)

**Error Scenarios**:
| Condition | Error Class | HTTP Status |
|-----------|-------------|-------------|
| No authentication cookie/JWT | `NotAuthorizedError` | 401 |
| Invalid `orderId` format | `RequestValidationError` | 400 |
| Order does not exist | `NotFoundError` | 404 |
| Order belongs to different user | `NotAuthorizedError` | 401 |

---

## Process: Cancel Order

**Trigger**: HTTP PATCH request to `/api/orders/:orderId/cancel`

**Input**: URL parameter `orderId` (must be a valid MongoDB ObjectId)

**Preconditions**:
1. User must be authenticated — enforced by `requireAuthMiddleware` in `services/orders/src/routes/cancel.ts`, line 19
2. `orderId` must be a valid MongoDB ObjectId — enforced by express-validator in `services/orders/src/routes/cancel.ts`, lines 20-24
3. Order must exist
4. Order must belong to the authenticated user
5. Order must not already be `Cancelled` or `Complete`

**Steps**:

1. [`requireAuthMiddleware` in `@b.anik/ticketing-common`] — Checks `req.currentUser` exists; if not, throws `NotAuthorizedError`

2. [Validation chain in `services/orders/src/routes/cancel.ts`, lines 20-24] — Validates `orderId` is valid MongoDB ObjectId

3. [`Order.findById()` in `services/orders/src/routes/cancel.ts`, line 28] — Database read: Queries MongoDB `Order` collection by `_id` and populates `ticket` reference
   - If order not found: Throws `NotFoundError()` (line 31)

4. [Ownership check in `services/orders/src/routes/cancel.ts`, lines 34-36] — Compares `order.userId` with `req.currentUser.id`
   - If not equal: Throws `NotAuthorizedError()` (line 35)

5. [Status check in `services/orders/src/routes/cancel.ts`, lines 38-44] — Validates order is in a cancellable state:
   - If `order.status === OrderStatus.Cancelled`: Throws `BadRequestError('Order is already cancelled')` (line 39)
   - If `order.status === OrderStatus.Complete`: Throws `BadRequestError('Order is already completed')` (line 43)

6. [`order.save()` in `services/orders/src/routes/cancel.ts`, lines 46-47] — Database write: Updates the `Order` document:
   - Field changed: `status` → `OrderStatus.Cancelled`
   - The `mongoose-update-if-current` plugin increments `version` automatically

7. [`OrderCancelledPublisher.publish()` in `services/orders/src/routes/cancel.ts`, lines 50-56] — Event published to NATS:
   - Subject: `order:cancelled` (via `Subjects.OrderCancelled`)
   - Payload:
     ```javascript
     {
       id: order.id,
       version: order.version,
       ticket: {
         id: order.ticket.id
       }
     }
     ```

**Business Rules**:
- Orders can only be cancelled by the user who created them
- Orders with status `Cancelled` cannot be cancelled again (idempotency protection with error feedback)
- Orders with status `Complete` cannot be cancelled (payment has already been processed) (inferred purpose)
- Orders with status `Created` or `AwaitingPayment` can be cancelled

**Output**: HTTP 200 response with JSON body:
```javascript
{
  order: {
    id: "<order_id>",
    userId: "<user_id>",
    status: "cancelled",
    expiresAt: "<ISO_timestamp>",
    ticket: { id: "<ticket_id>", title: "<title>", price: <number>, version: <number> },
    version: <incremented_version>
  }
}
```

**Post-conditions**:
1. `Order` document updated with `status: Cancelled` and incremented `version`
2. `order:cancelled` event published to NATS Streaming
3. The ticket associated with this order becomes available for new orders (since cancelled orders are excluded from `isReserved()` check)

**Error Scenarios**:
| Condition | Error Class | HTTP Status | Message |
|-----------|-------------|-------------|---------|
| No authentication cookie/JWT | `NotAuthorizedError` | 401 | — |
| Invalid `orderId` format | `RequestValidationError` | 400 | `orderId is required` |
| Order does not exist | `NotFoundError` | 404 | — |
| Order belongs to different user | `NotAuthorizedError` | 401 | — |
| Order already cancelled | `BadRequestError` | 400 | `Order is already cancelled` |
| Order already completed | `BadRequestError` | 400 | `Order is already completed` |

---

## Process: Ticket Synchronization - Created

**Trigger**: NATS event with subject `ticket:created` (published by tickets service)

**Input**: Event payload `TicketCreatedEvent['data']`:
```javascript
{
  id: string,
  title: string,
  price: number,
  userId: string,
  version: number
}
```

**Preconditions**: None (event-driven, always processes)

**Steps**:

1. [`TicketCreatedListener.onMessage()` in `services/orders/src/events/listeners/ticket-created-listener.ts`, lines 10-28] — Extracts `id`, `title`, `price`, `version` from event data

2. [`Ticket.build()` in `services/orders/src/events/listeners/ticket-created-listener.ts`, lines 15-20] — Creates new Ticket document:
   - Note: `Ticket.build()` in `services/orders/src/models/ticket.ts`, lines 49-59 validates the `id` is a valid ObjectId and uses it as `_id` to maintain ID consistency across services

3. [`ticket.save()` in `services/orders/src/events/listeners/ticket-created-listener.ts`, line 21] — Database write: Inserts new document in `Ticket` collection:
   - Fields written:
     - `_id`: Original ticket ID from tickets service
     - `title`: Ticket title
     - `price`: Ticket price
     - `version`: Version number (starts at 0)

**Business Rules**:
- The orders service maintains a local replica of ticket data for autonomous operation
- Ticket IDs are preserved from the source service (tickets service) to maintain referential integrity
- The `userId` field from the event is not stored (orders service doesn't need to know who created the ticket)

**Output**: Promise resolves with no return value on success

**Post-conditions**:
1. New `Ticket` document exists in orders service MongoDB with the same ID as in tickets service
2. NATS message is acknowledged (handled by base `Listener` class)

**Error Scenarios**:
| Condition | Behavior |
|-----------|----------|
| Invalid ObjectId in event | `Ticket.build()` throws `Error('[Ticket.build] Invalid ObjectId: <id>')` — promise rejects, message not acknowledged |
| Duplicate ticket ID | MongoDB throws duplicate key error — promise rejects, message not acknowledged |

---

## Process: Ticket Synchronization - Updated

**Trigger**: NATS event with subject `ticket:updated` (published by tickets service)

**Input**: Event payload `TicketUpdatedEvent['data']`:
```javascript
{
  id: string,
  title: string,
  price: number,
  userId: string,
  version: number
}
```

**Preconditions**:
1. Ticket with matching `id` must exist in local replica
2. Local ticket version must be exactly `event.version - 1` (optimistic concurrency control)

**Steps**:

1. [`TicketUpdatedListener.onMessage()` in `services/orders/src/events/listeners/ticket-updated-listener.ts`, lines 10-31] — Receives event data

2. [`Ticket.findByEvent()` in `services/orders/src/models/ticket.ts`, lines 62-70] — Database read: Queries MongoDB `Ticket` collection with version-aware lookup:
   ```javascript
   Ticket.findOne({
     _id: event.id,
     version: event.version - 1
   })
   ```
   - This ensures events are processed in order; if version is out of sequence, ticket won't be found
   - If ticket not found (either doesn't exist or version mismatch): Throws `Error('Ticket not found')` (line 17)

3. [`ticket.set()` and `ticket.save()` in `services/orders/src/events/listeners/ticket-updated-listener.ts`, lines 19-24] — Database write: Updates the `Ticket` document:
   - Fields updated:
     - `title`: New title from event
     - `price`: New price from event
     - `version`: New version from event (explicitly set, not auto-incremented)

**Business Rules**:
- Events must be processed in order; version `n` can only be applied if local version is `n-1`
- If an event arrives out of order (e.g., version 3 arrives before version 2), the listener will reject it by throwing "Ticket not found"
- The rejected event will not be acknowledged, causing NATS to redeliver it later (inferred from base Listener behavior)
- The `userId` field from the event is not stored in the local replica

**Output**: Promise resolves with no return value on success

**Post-conditions**:
1. `Ticket` document updated with new `title`, `price`, and `version`
2. NATS message is acknowledged (handled by base `Listener` class)

**Error Scenarios**:
| Condition | Behavior |
|-----------|----------|
| Ticket doesn't exist | Throws `Error('Ticket not found')` — promise rejects |
| Version mismatch (out of order) | Throws `Error('Ticket not found')` — promise rejects |

---

## Process: Order Expiration

**Trigger**: NATS event with subject `expiration:complete` (published by expiration service)

**Input**: Event payload `ExpirationCompleteEvent['data']`:
```javascript
{
  orderId: string
}
```

**Preconditions**:
1. Order with matching `orderId` must exist

**Steps**:

1. [`ExpirationCompleteListener.onMessage()` in `services/orders/src/events/listeners/expiration-complete-listener.ts`, lines 11-40] — Receives event data

2. [`Order.findById()` in `services/orders/src/events/listeners/expiration-complete-listener.ts`, line 14] — Database read: Queries MongoDB `Order` collection by `_id` and populates `ticket` reference
   - If order not found: Throws `Error('Order not found')` (line 17)

3. [Status check in `services/orders/src/events/listeners/expiration-complete-listener.ts`, lines 20-23] — Checks if order is already in a terminal state:
   - If `order.status === OrderStatus.Cancelled` OR `order.status === OrderStatus.Complete`: Returns early (resolves promise without changes)
   - This handles race conditions where payment completes just before expiration fires

4. [`order.save()` in `services/orders/src/events/listeners/expiration-complete-listener.ts`, lines 25-26] — Database write: Updates the `Order` document:
   - Field changed: `status` → `OrderStatus.Cancelled`
   - `version` is auto-incremented by `mongoose-update-if-current` plugin

5. [`OrderCancelledPublisher.publish()` in `services/orders/src/events/listeners/expiration-complete-listener.ts`, lines 28-34] — Event published to NATS:
   - Subject: `order:cancelled` (via `Subjects.OrderCancelled`)
   - Payload:
     ```javascript
     {
       id: order.id,
       version: order.version,
       ticket: {
         id: order.ticket.id
       }
     }
     ```

**Business Rules**:
- Orders that have not been paid within the expiration window (15 minutes from creation) are automatically cancelled
- Orders already in `Complete` status are not cancelled (payment succeeded in time)
- Orders already in `Cancelled` status are not modified (idempotent handling)
- Cancellation via expiration releases the ticket for new orders

**Output**: Promise resolves with no return value

**Post-conditions** (if order was active):
1. `Order` document updated with `status: Cancelled` and incremented `version`
2. `order:cancelled` event published to NATS Streaming
3. The associated ticket becomes available for new orders

**Post-conditions** (if order was already cancelled/complete):
1. No database changes
2. No events published
3. Promise resolves successfully (idempotent)

**Error Scenarios**:
| Condition | Behavior |
|-----------|----------|
| Order doesn't exist | Throws `Error('Order not found')` — promise rejects |

---

## Process: Payment Awaited

**Trigger**: NATS event with subject `payment:awaited` (published by payments service)

**Input**: Event payload `PaymentAwaitedEvent['data']`:
```javascript
{
  orderId: string
}
```

**Preconditions**:
1. Order with matching `orderId` must exist

**Steps**:

1. [`PaymentAwaitedListener.onMessage()` in `services/orders/src/events/listeners/payment-awaited-listener.ts`, lines 9-32] — Receives event data

2. [`Order.findById()` in `services/orders/src/events/listeners/payment-awaited-listener.ts`, line 12] — Database read: Queries MongoDB `Order` collection by `_id` and populates `ticket` reference
   - If order not found: Throws `Error('Order not found')` (line 15)

3. [`order.save()` in `services/orders/src/events/listeners/payment-awaited-listener.ts`, lines 18-19] — Database write: Updates the `Order` document:
   - Field changed: `status` → `OrderStatus.AwaitingPayment`
   - `version` is auto-incremented by `mongoose-update-if-current` plugin

4. [`OrderUpdatedPublisher.publish()` in `services/orders/src/events/listeners/payment-awaited-listener.ts`, lines 21-25] — Event published to NATS:
   - Subject: `order:updated` (via `Subjects.OrderUpdated`)
   - Payload:
     ```javascript
     {
       id: order.id,
       version: order.version,
       status: order.status  // "awaiting:payment"
     }
     ```

**Business Rules**:
- When payment processing begins, the order transitions from `Created` to `AwaitingPayment`
- This status transition is tracked for visibility into the payment flow (inferred)

**Output**: Promise resolves with no return value

**Post-conditions**:
1. `Order` document updated with `status: AwaitingPayment` and incremented `version`
2. `order:updated` event published to NATS Streaming

**Error Scenarios**:
| Condition | Behavior |
|-----------|----------|
| Order doesn't exist | Throws `Error('Order not found')` — promise rejects |

**Note**: There is a bug in line 29 where `reject` is referenced without calling it: `reject;` instead of `reject(error);`. This means errors after the try block may not properly reject the promise.

---

## Process: Payment Complete

**Trigger**: NATS event with subject `payment:complete` (published by payments service)

**Input**: Event payload `PaymentCompleteEvent['data']`:
```javascript
{
  orderId: string
}
```

**Preconditions**:
1. Order with matching `orderId` must exist

**Steps**:

1. [`PaymentCompleteListener.onMessage()` in `services/orders/src/events/listeners/payment-complete-listener.ts`, lines 9-34] — Receives event data, extracts `orderId`

2. [`Order.findById()` in `services/orders/src/events/listeners/payment-complete-listener.ts`, line 14] — Database read: Queries MongoDB `Order` collection by `_id`
   - Note: Does not populate `ticket` reference (unlike other listeners)
   - If order not found: Throws `Error('Order not found')` (line 18)

3. [`order.save()` in `services/orders/src/events/listeners/payment-complete-listener.ts`, lines 20-21] — Database write: Updates the `Order` document:
   - Field changed: `status` → `OrderStatus.Complete`
   - `version` is auto-incremented by `mongoose-update-if-current` plugin

4. [`OrderUpdatedPublisher.publish()` in `services/orders/src/events/listeners/payment-complete-listener.ts`, lines 23-27] — Event published to NATS:
   - Subject: `order:updated` (via `Subjects.OrderUpdated`)
   - Payload:
     ```javascript
     {
       id: order.id,
       status: order.status,  // "complete"
       version: order.version
     }
     ```

**Business Rules**:
- When payment succeeds, the order transitions to `Complete` status
- Completed orders cannot be cancelled (enforced by cancel route)
- Completed orders keep the ticket reserved permanently (ticket remains unavailable for new orders because `Complete` status is included in `isReserved()` check)

**Output**: Promise resolves with no return value

**Post-conditions**:
1. `Order` document updated with `status: Complete` and incremented `version`
2. `order:updated` event published to NATS Streaming
3. The ticket associated with this order is permanently reserved

**Error Scenarios**:
| Condition | Behavior |
|-----------|----------|
| Order doesn't exist | Throws `Error('Order not found')` — promise rejects |

---

## Process: Health Check

**Trigger**: HTTP GET request to `/api/orders/health`

**Input**: None

**Preconditions**: None (unauthenticated endpoint)

**Steps**:

1. [Route handler in `services/orders/src/routes/health.ts`, line 5] — Returns immediately with empty response

**Business Rules**: None

**Output**: HTTP 200 response with empty JSON body `{}`

**Post-conditions**: None

**Error Scenarios**: None (this endpoint always succeeds)

---

## Cross-Cutting Concerns

### Optimistic Concurrency Control

The Order model uses `mongoose-update-if-current` plugin (`services/orders/src/models/order.ts`, line 58) which:
- Automatically increments the `version` field on each `save()` operation
- Fails with a version conflict error if attempting to save a document that was modified by another process since it was loaded
- This prevents lost updates when multiple services or requests modify the same order concurrently

Verified behavior from tests (`services/orders/src/models/__tests__/order.test.ts`):
- Version starts at `0` and increments by `1` on each save
- Concurrent modifications to stale instances throw error matching `/version 0/i`

### Queue Group Membership

All listeners use queue group name `'orders-service'` (`services/orders/src/events/listeners/queue-group-name.ts`, line 1). This ensures:
- Only one instance of the orders service processes each event (when multiple replicas are running)
- Load balancing across service replicas
- No duplicate processing of events

### Event Listener Initialization

On service startup (`services/orders/src/index.ts`, line 45), `startListeners()` initializes all five listeners:
1. `TicketCreatedListener`
2. `TicketUpdatedListener`
3. `ExpirationCompleteListener`
4. `PaymentAwaitedListener`
5. `PaymentCompleteListener`

Each listener subscribes to its respective NATS subject via the base `Listener.listen()` method from `@b.anik/ticketing-common`.

### Order Status State Machine

Based on observed code paths, orders follow this state machine:

```
Created ──────────────────────────┬──────────────────────────> Cancelled
    │                             │                              ^
    │ (payment:awaited)           │ (user cancel or             │
    v                             │  expiration:complete)       │
AwaitingPayment ──────────────────┴──────────────────────────────
    │
    │ (payment:complete)
    v
Complete ─────────────────────────> [terminal, cannot cancel]
```

Valid transitions:
- `Created` → `AwaitingPayment` (via `payment:awaited` event)
- `Created` → `Cancelled` (via user cancel, `expiration:complete` event)
- `AwaitingPayment` → `Complete` (via `payment:complete` event)
- `AwaitingPayment` → `Cancelled` (via user cancel, `expiration:complete` event)

Invalid transitions (blocked by code):
- `Complete` → `Cancelled` (throws `BadRequestError` in cancel route; skipped in expiration listener)
- `Cancelled` → any state (throws `BadRequestError` in cancel route; skipped in expiration listener)
