---
service_id: ticketing-app/orders
service_name: orders
prompt_id: 02-api-data-contracts
prompt_label: API & Data Contracts
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.269795
schema_version: "2.0"
---

# Orders Service API & Data Contracts

## Part A: HTTP API Endpoints

---

### [GET] /api/orders/health

- **Controller**: `services/orders/src/routes/health.ts`, anonymous arrow function (line 5)
- **Authentication**: None
- **Authorization**: None
- **Request**:
  - Path parameters: None
  - Query parameters: None
  - Request body: None
  - Headers: None required
- **Processing**: Returns immediate empty success response
- **Response**:
  - Success: `200 OK`
    ```json
    {}
    ```
  - Error cases: None
- **Side effects**: None

---

### [GET] /api/orders

- **Controller**: `services/orders/src/routes/show.ts`, anonymous async function (line 9)
- **Authentication**: JWT via `cookie-session` middleware, decoded by `currentUserMiddleware` (`src/app.ts`, line 30)
- **Authorization**: `requireAuthMiddleware` – throws `NotAuthorizedError` if `req.currentUser` is not set (`src/routes/show.ts`, line 9)
- **Request**:
  - Path parameters: None
  - Query parameters: None
  - Request body: None
  - Headers: `Cookie` header with `session` cookie containing base64-encoded `{ jwt: <JWT_TOKEN> }`
- **Processing**:
  1. Extracts `userId` from `req.currentUser.id`
  2. Queries `Order.find({ userId })` with `.populate('ticket')` to include ticket details (`src/routes/show.ts`, lines 10-12)
  3. Returns all orders owned by the authenticated user
- **Response**:
  - Success: `200 OK`
    ```typescript
    {
      orders: Array<{
        id: string;           // MongoDB ObjectId
        userId: string;       // User who created the order
        status: OrderStatus;  // "created" | "cancelled" | "awaiting-payment" | "complete"
        expiresAt?: string;   // ISO 8601 datetime string (optional)
        version: number;      // Optimistic concurrency version
        ticket: {
          id: string;         // MongoDB ObjectId
          title: string;
          price: number;
          version: number;
        }
      }>
    }
    ```
  - Error cases:
    - `401 Unauthorized`: User not authenticated
      ```json
      { "errors": [{ "message": "Not authorized" }] }
      ```
- **Side effects**: None

---

### [GET] /api/orders/:orderId

- **Controller**: `services/orders/src/routes/show.ts`, anonymous async function (line 19)
- **Authentication**: JWT via `cookie-session` middleware
- **Authorization**: 
  1. `requireAuthMiddleware` – requires authenticated user (line 21)
  2. Ownership check – `order.userId !== req.currentUser.id` throws `NotAuthorizedError` (lines 36-38)
- **Request**:
  - Path parameters:
    | Name | Type | Description |
    |------|------|-------------|
    | `orderId` | string (MongoDB ObjectId) | Required. Must be a valid MongoDB ObjectId. Validated by `express-validator` with `isValidObjectId()` |
  - Query parameters: None
  - Request body: None
  - Headers: `Cookie` header with session JWT
- **Processing**:
  1. Validates `orderId` parameter is a valid MongoDB ObjectId (`src/routes/show.ts`, lines 22-26)
  2. Queries `Order.findById(orderId).populate('ticket')` (line 30)
  3. Throws `NotFoundError` if order not found (lines 32-34)
  4. Throws `NotAuthorizedError` if order belongs to different user (lines 36-38)
  5. Returns the order with populated ticket
- **Response**:
  - Success: `200 OK`
    ```typescript
    {
      order: {
        id: string;
        userId: string;
        status: OrderStatus;
        expiresAt?: string;
        version: number;
        ticket: {
          id: string;
          title: string;
          price: number;
          version: number;
        }
      }
    }
    ```
  - Error cases:
    - `400 Bad Request`: Invalid `orderId` format (not validated by `validateRequestMiddleware` – validation present but middleware not used)
    - `401 Unauthorized`: User not authenticated or order belongs to different user
      ```json
      { "errors": [{ "message": "Not authorized" }] }
      ```
    - `404 Not Found`: Order with given ID not found
      ```json
      { "errors": [{ "message": "Not found" }] }
      ```
- **Side effects**: None

---

### [POST] /api/orders

- **Controller**: `services/orders/src/routes/new.ts`, anonymous async function (line 18)
- **Authentication**: JWT via `cookie-session` middleware
- **Authorization**: `requireAuthMiddleware` (line 19)
- **Request**:
  - Path parameters: None
  - Query parameters: None
  - Request body:
    ```typescript
    {
      ticketId: string  // Required. Must be a valid MongoDB ObjectId.
                        // Validated via express-validator body('ticketId') with trim(), notEmpty(), isValidObjectId()
    }
    ```
  - Headers: `Cookie` header with session JWT
- **Processing** (`src/routes/new.ts`, lines 27-66):
  1. Validates `ticketId` is a non-empty valid MongoDB ObjectId (lines 21-26)
  2. Finds ticket by ID: `Ticket.findById(ticketId)` (line 31)
  3. Throws `BadRequestError('Ticket not found')` if ticket doesn't exist (lines 32-34)
  4. Checks if ticket is reserved via `ticket.isReserved()` (line 37)
  5. Throws `BadRequestError('Ticket is already reserved')` if reserved (lines 38-40)
  6. Calculates expiration: current time + 15 minutes (900 seconds) (lines 43-44)
  7. Creates new Order with `userId`, `status: OrderStatus.Created`, `expiresAt`, `ticket` (lines 47-52)
  8. Saves order to database (line 53)
  9. Publishes `order:created` event via `OrderCreatedPublisher` (lines 56-66)
- **Response**:
  - Success: `201 Created`
    ```typescript
    {
      order: {
        id: string;
        userId: string;
        status: "created";
        expiresAt: string;    // ISO 8601 datetime, 15 minutes from creation
        version: number;      // Initially 0
        ticket: {
          id: string;
          title: string;
          price: number;
          version: number;
        }
      }
    }
    ```
  - Error cases:
    - `400 Bad Request`: Validation error (invalid/missing `ticketId`)
      ```json
      { "errors": [{ "field": "ticketId", "message": "ticketId must be provided" }] }
      ```
    - `400 Bad Request`: Ticket not found
      ```json
      { "errors": [{ "message": "Ticket not found" }] }
      ```
    - `400 Bad Request`: Ticket already reserved
      ```json
      { "errors": [{ "message": "Ticket is already reserved" }] }
      ```
    - `401 Unauthorized`: User not authenticated
      ```json
      { "errors": [{ "message": "Not authorized" }] }
      ```
- **Side effects**:
  - **Publishes NATS event**: `order:created` (subject: `Subjects.OrderCreated`)
    ```typescript
    {
      id: string;             // Order ID
      version: number;        // Order version (0 on creation)
      status: OrderStatus;    // "created"
      userId: string;         // User ID from JWT
      expiresAt: string;      // ISO 8601 datetime
      ticket: {
        id: string;           // Ticket ID
        price: number;        // Ticket price
      }
    }
    ```
    Published via `OrderCreatedPublisher` (`src/events/publishers/order-created-publisher.ts`)

---

### [PATCH] /api/orders/:orderId/cancel

- **Controller**: `services/orders/src/routes/cancel.ts`, anonymous async function (line 17)
- **Authentication**: JWT via `cookie-session` middleware
- **Authorization**:
  1. `requireAuthMiddleware` (line 19)
  2. Ownership check – throws `NotAuthorizedError` if `order.userId !== req.currentUser.id` (lines 34-36)
- **Request**:
  - Path parameters:
    | Name | Type | Description |
    |------|------|-------------|
    | `orderId` | string (MongoDB ObjectId) | Required. Must be valid ObjectId. |
  - Query parameters: None
  - Request body: None
  - Headers: `Cookie` header with session JWT
- **Processing** (`src/routes/cancel.ts`, lines 25-56):
  1. Validates `orderId` is valid MongoDB ObjectId (lines 20-24)
  2. Finds order: `Order.findById(orderId).populate('ticket')` (line 28)
  3. Throws `NotFoundError` if order not found (lines 30-32)
  4. Throws `NotAuthorizedError` if order belongs to different user (lines 34-36)
  5. Throws `BadRequestError('Order is already cancelled')` if status is `Cancelled` (lines 38-40)
  6. Throws `BadRequestError('Order is already completed')` if status is `Complete` (lines 42-44)
  7. Sets `order.status = OrderStatus.Cancelled` (line 46)
  8. Saves order (line 47)
  9. Publishes `order:cancelled` event (lines 50-56)
- **Response**:
  - Success: `200 OK`
    ```typescript
    {
      order: {
        id: string;
        userId: string;
        status: "cancelled";
        expiresAt?: string;
        version: number;      // Incremented by 1 after save
        ticket: {
          id: string;
          title: string;
          price: number;
          version: number;
        }
      }
    }
    ```
  - Error cases:
    - `400 Bad Request`: Order already cancelled
      ```json
      { "errors": [{ "message": "Order is already cancelled" }] }
      ```
    - `400 Bad Request`: Order already completed
      ```json
      { "errors": [{ "message": "Order is already completed" }] }
      ```
    - `401 Unauthorized`: User not authenticated or order belongs to different user
      ```json
      { "errors": [{ "message": "Not authorized" }] }
      ```
    - `404 Not Found`: Order not found
      ```json
      { "errors": [{ "message": "Not found" }] }
      ```
- **Side effects**:
  - **Publishes NATS event**: `order:cancelled` (subject: `Subjects.OrderCancelled`)
    ```typescript
    {
      id: string;           // Order ID
      version: number;      // Updated order version
      ticket: {
        id: string;         // Ticket ID (now unreserved)
      }
    }
    ```
    Published via `OrderCancelledPublisher` (`src/events/publishers/order-cancelled-publisher.ts`)

---

## Part B: Database Schema

### Order

- **Database collection name**: `orders` (Mongoose model name: `'Order'`, `src/models/order.ts`, line 64)
- **Purpose**: Represents a user's order for a ticket, tracking the order lifecycle from creation through payment or cancellation

**Fields**:

| Field | Type | Required | Default | Index | Description |
|-------|------|----------|---------|-------|-------------|
| `_id` | ObjectId | Yes (auto) | Auto-generated | Primary | MongoDB document ID, transformed to `id` in JSON output |
| `userId` | String | Yes | - | No | ID of the user who created the order; from JWT `currentUser.id` |
| `status` | String (enum) | No | `'created'` | No | Order lifecycle state; validated against `OrderStatus` enum values |
| `expiresAt` | Date | No | - | No | Datetime when order expires and ticket becomes available again |
| `ticket` | ObjectId (ref: 'Ticket') | No | - | No | Reference to the Ticket document being ordered |
| `version` | Number | Yes (auto) | 0 | No | Optimistic concurrency control version; incremented on each save |

**Relationships**:
- `ticket` → `Ticket` (many-to-one): Each order references exactly one ticket via ObjectId. Populated with `.populate('ticket')` in queries (`src/routes/show.ts`, lines 12, 30; `src/routes/cancel.ts`, line 28)

**Enums**:

`OrderStatus` (imported from `@b.anik/ticketing-common`, values observed in client code at `apps/client/src/constants/order.ts`):

| Value | String | Business Meaning |
|-------|--------|------------------|
| `Created` | `'created'` | Initial state when order is placed; ticket is reserved |
| `Cancelled` | `'cancelled'` | Order cancelled by user or expired; ticket released |
| `AwaitingPayment` | `'awaiting-payment'` | Payment processing has begun |
| `Complete` | `'complete'` | Payment successful; ticket sold |

**Optimistic Concurrency Control**:
- Uses `mongoose-update-if-current` plugin (`src/models/order.ts`, line 58)
- Version key renamed from `__v` to `version` (line 57)
- Concurrent saves to same document version will throw an error (tested in `src/models/__tests__/order.test.ts`)

**JSON Transformation** (`src/models/order.ts`, lines 47-54):
- `_id` → `id` (renamed)
- `__v` removed (version key uses `version` field)

**Common Query Patterns**:
- Find all orders for user: `Order.find({ userId }).populate('ticket')` (`src/routes/show.ts`, lines 10-12)
- Find single order by ID: `Order.findById(orderId).populate('ticket')` (`src/routes/show.ts`, line 30)
- Check if ticket is reserved: `Order.findOne({ ticket: ticketDoc, status: { $in: [Created, AwaitingPayment, Complete] } })` (`src/models/ticket.ts`, lines 77-83)

---

### Ticket

- **Database collection name**: `tickets` (Mongoose model name: `'Ticket'`, `src/models/ticket.ts`, line 87)
- **Purpose**: Local copy/replica of ticket data from the Tickets service; enables order service to check ticket availability without cross-service calls

**Fields**:

| Field | Type | Required | Default | Index | Description |
|-------|------|----------|---------|-------|-------------|
| `_id` | ObjectId | Yes | Set from external ticket ID | Primary | MongoDB document ID; matches ticket ID from Tickets service |
| `title` | String | Yes | - | No | Ticket title/name |
| `price` | Number | Yes | - | No | Ticket price in currency units |
| `version` | Number | Yes | - | No | Version from source Tickets service; used for event ordering |

**Relationships**:
- Referenced by `Order.ticket` (one-to-many): Multiple orders can reference the same ticket over time (only one active non-cancelled order at a time)

**Static Methods** (`src/models/ticket.ts`):

| Method | Signature | Description |
|--------|-----------|-------------|
| `build` | `(attrs: TicketAttrs) => TicketDoc` | Creates ticket with explicit `_id` from external service (lines 49-60) |
| `findByEvent` | `(event: {id, version}) => Promise<TicketDoc \| null>` | Finds ticket by ID with version-1 for event ordering (lines 62-70) |

**Instance Methods** (`src/models/ticket.ts`):

| Method | Signature | Description |
|--------|-----------|-------------|
| `isReserved` | `() => Promise<boolean>` | Checks if ticket has active (non-cancelled) order (lines 75-85) |

**JSON Transformation** (`src/models/ticket.ts`, lines 39-46):
- `_id` → `id` (renamed)
- `__v` removed

**Common Query Patterns**:
- Find by ID: `Ticket.findById(ticketId)` (`src/routes/new.ts`, line 31)
- Find by event (version-aware): `Ticket.findOne({ _id: event.id, version: event.version - 1 })` (`src/models/ticket.ts`, lines 66-69)
- Check reservation: `Order.findOne({ ticket: this, status: { $in: [...] } })` (`src/models/ticket.ts`, lines 77-83)

---

## Part C: Type Definitions & DTOs

### OrderAttrs

- **File**: `services/orders/src/models/order.ts`, lines 8-13
- **Definition**:
  ```typescript
  interface OrderAttrs {
    userId: string;
    status?: OrderStatus;
    expiresAt?: Date;
    ticket: TicketDoc;
  }
  ```
- **Usage**: Input attributes for `Order.build()` static method; used when creating new orders in route handlers

---

### OrderDoc

- **File**: `services/orders/src/models/order.ts`, lines 15-21
- **Definition**:
  ```typescript
  interface OrderDoc extends mongoose.Document {
    userId: string;
    status: OrderStatus;
    expiresAt?: Date;
    ticket: TicketDoc;
    version: number;
  }
  ```
- **Usage**: Type for Order Mongoose documents; used throughout route handlers, event listeners, and tests

---

### OrderModel

- **File**: `services/orders/src/models/order.ts`, lines 23-25
- **Definition**:
  ```typescript
  interface OrderModel extends mongoose.Model<any> {
    build(attrs: OrderAttrs): OrderDoc;
  }
  ```
- **Usage**: Type for the Order Mongoose model with `build` static method

---

### TicketAttrs

- **File**: `services/orders/src/models/ticket.ts`, lines 4-9
- **Definition**:
  ```typescript
  interface TicketAttrs {
    id: string;
    title: string;
    price: number;
    version: number;
  }
  ```
- **Usage**: Input attributes for `Ticket.build()` static method; used when creating local ticket copies from events

---

### TicketDoc

- **File**: `services/orders/src/models/ticket.ts`, lines 11-16
- **Definition**:
  ```typescript
  interface TicketDoc extends mongoose.Document {
    title: string;
    price: number;
    version: number;
    isReserved(): Promise<boolean>;
  }
  ```
- **Usage**: Type for Ticket Mongoose documents; used in order creation, event listeners

---

### TicketModel

- **File**: `services/orders/src/models/ticket.ts`, lines 18-21
- **Definition**:
  ```typescript
  interface TicketModel extends mongoose.Model<TicketDoc> {
    build(attrs: TicketAttrs): TicketDoc;
    findByEvent(event: { id: string; version: number }): Promise<TicketDoc | null>;
  }
  ```
- **Usage**: Type for the Ticket Mongoose model with static methods

---

### OrderStatus (enum)

- **File**: Imported from `@b.anik/ticketing-common`, re-exported from `services/orders/src/models/order.ts`, line 6
- **Definition** (observed in `apps/client/src/constants/order.ts`):
  ```typescript
  enum OrderStatus {
    Created = 'created',
    Cancelled = 'cancelled',
    AwaitingPayment = 'awaiting-payment',
    Complete = 'complete',
  }
  ```
- **Usage**: Order status field type; used in Order model, route handlers, event listeners

---

### JWT Payload (currentUser)

- **File**: `services/orders/src/test/utils.ts`, lines 4-7, 9-12
- **Definition** (inferred from test utility):
  ```typescript
  interface CurrentUser {
    id: string;
    email: string;
  }
  ```
- **Usage**: Decoded JWT payload attached to `req.currentUser` by `currentUserMiddleware`; used for user identification in all authenticated routes

---

### Cookie Session Structure

- **File**: `services/orders/src/test/utils.ts`, lines 16-28
- **Definition** (inferred):
  ```typescript
  interface Session {
    jwt: string;  // JWT token string
  }
  // Cookie value: base64(JSON.stringify(session))
  ```
- **Usage**: Session stored in `session` cookie; configured with `signed: false` and `secure: NODE_ENV !== 'test'` (`src/app.ts`, lines 20-25)

---

### Event Payloads

#### OrderCreatedEvent (published)

- **File**: Publishers at `services/orders/src/events/publishers/order-created-publisher.ts`; payload constructed in `src/routes/new.ts`, lines 56-66
- **Definition**:
  ```typescript
  interface OrderCreatedEvent {
    subject: Subjects.OrderCreated;  // 'order:created'
    data: {
      id: string;             // Order ID
      version: number;        // Order version
      status: OrderStatus;    // Order status
      userId: string;         // User ID
      expiresAt?: string;     // ISO 8601 datetime
      ticket: {
        id: string;           // Ticket ID
        price: number;        // Ticket price
      };
    };
  }
  ```
- **Usage**: Published when a new order is created

---

#### OrderCancelledEvent (published)

- **File**: Publishers at `services/orders/src/events/publishers/order-cancelled-publisher.ts`; payload constructed in `src/routes/cancel.ts`, lines 50-56 and `src/events/listeners/expiration-complete-listener.ts`, lines 28-34
- **Definition**:
  ```typescript
  interface OrderCancelledEvent {
    subject: Subjects.OrderCancelled;  // 'order:cancelled'
    data: {
      id: string;           // Order ID
      version: number;      // Updated order version
      ticket: {
        id: string;         // Ticket ID
      };
    };
  }
  ```
- **Usage**: Published when an order is cancelled by user or by expiration

---

#### OrderUpdatedEvent (published)

- **File**: Publishers at `services/orders/src/events/publishers/order-updated-publisher.ts`; payload constructed in `src/events/listeners/payment-awaited-listener.ts`, lines 21-25 and `src/events/listeners/payment-complete-listener.ts`, lines 23-27
- **Definition**:
  ```typescript
  interface OrderUpdatedEvent {
    subject: Subjects.OrderUpdated;  // 'order:updated'
    data: {
      id: string;           // Order ID
      version: number;      // Updated order version
      status: OrderStatus;  // New order status
    };
  }
  ```
- **Usage**: Published when order status changes to `AwaitingPayment` or `Complete`

---

#### TicketCreatedEvent (consumed)

- **File**: Listener at `services/orders/src/events/listeners/ticket-created-listener.ts`; payload shape from test at `src/events/listeners/__tests__/ticket-created-listener.test.ts`, lines 9-15
- **Definition**:
  ```typescript
  interface TicketCreatedEvent {
    subject: Subjects.TicketCreated;  // 'ticket:created'
    data: {
      id: string;           // Ticket ID
      version: number;      // Ticket version (0 on creation)
      userId: string;       // Ticket creator user ID
      title: string;        // Ticket title
      price: number;        // Ticket price
    };
  }
  ```
- **Usage**: Consumed to create local ticket replica

---

#### TicketUpdatedEvent (consumed)

- **File**: Listener at `services/orders/src/events/listeners/ticket-updated-listener.ts`; payload shape from test at `src/events/listeners/__tests__/ticket-updated-listener.test.ts`, lines 27-33
- **Definition**:
  ```typescript
  interface TicketUpdatedEvent {
    subject: Subjects.TicketUpdated;  // 'ticket:updated'
    data: {
      id: string;           // Ticket ID
      version: number;      // New ticket version
      userId: string;       // Ticket owner user ID
      title: string;        // Updated title
      price: number;        // Updated price
    };
  }
  ```
- **Usage**: Consumed to update local ticket replica

---

#### ExpirationCompleteEvent (consumed)

- **File**: Listener at `services/orders/src/events/listeners/expiration-complete-listener.ts`; payload shape from test at `src/events/listeners/__tests__/expiration-complete-listener.test.ts`, lines 28-30
- **Definition**:
  ```typescript
  interface ExpirationCompleteEvent {
    subject: Subjects.ExpirationComplete;  // 'expiration:complete'
    data: {
      orderId: string;      // Order ID to cancel
    };
  }
  ```
- **Usage**: Consumed to auto-cancel expired orders

---

#### PaymentAwaitedEvent (consumed)

- **File**: Listener at `services/orders/src/events/listeners/payment-awaited-listener.ts`; payload shape from test at `src/events/listeners/__tests__/payment-awaited-listener.test.ts`, lines 29-31
- **Definition**:
  ```typescript
  interface PaymentAwaitedEvent {
    subject: Subjects.PaymentAwaited;  // 'payment:awaited'
    data: {
      orderId: string;      // Order ID being paid
    };
  }
  ```
- **Usage**: Consumed to update order status to `AwaitingPayment`

---

#### PaymentCompleteEvent (consumed)

- **File**: Listener at `services/orders/src/events/listeners/payment-complete-listener.ts`; payload shape from test at `src/events/listeners/__tests__/payment-complete.listener.test.ts`, lines 27-29
- **Definition**:
  ```typescript
  interface PaymentCompleteEvent {
    subject: Subjects.PaymentComplete;  // 'payment:complete'
    data: {
      orderId: string;      // Order ID that was paid
    };
  }
  ```
- **Usage**: Consumed to update order status to `Complete`

---

### Error Response Shape

- **File**: Middleware from `@b.anik/ticketing-common`; shape inferred from test assertions (`src/routes/__tests__/new.test.ts`, lines 49-50, 69-71)
- **Definition**:
  ```typescript
  interface ErrorResponse {
    errors: Array<{
      message: string;
      field?: string;   // Present for validation errors
    }>;
  }
  ```
- **Usage**: Standard error response format for all error types (validation, not found, not authorized, bad request)
