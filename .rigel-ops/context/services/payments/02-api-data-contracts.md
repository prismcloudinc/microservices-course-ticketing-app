---
service_id: ticketing-app/payments
service_name: payments
prompt_id: 02-api-data-contracts
prompt_label: API & Data Contracts
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.21499000000000001
schema_version: "2.0"
---

# Payments Service - API & Data Model Contract

## Part A: HTTP API Endpoints

### GET /api/payments/health

- **Controller**: `services/payments/src/routes/health.ts`, anonymous arrow function (line 5)
- **Authentication**: None
- **Authorization**: None
- **Request**:
  - Path parameters: None
  - Query parameters: None
  - Request body: None
  - Headers: None required
- **Processing**: Returns immediately with HTTP 200
- **Response**:
  - **Success (200)**:
    ```json
    {}
    ```
  - **Error cases**: None
- **Side effects**: None

---

### GET /api/payments

- **Controller**: `services/payments/src/routes/show.ts`, anonymous async function (line 13)
- **Authentication**: `requireAuthMiddleware` — verifies JWT in session cookie, sets `req.currentUser`
- **Authorization**: Implicitly filters to only payments owned by authenticated user via `userId` query filter
- **Request**:
  - Path parameters: None
  - Query parameters: None
  - Request body: None
  - Headers:
    | Header | Required | Description |
    |--------|----------|-------------|
    | `Cookie` | Yes | Session cookie containing base64-encoded `{ jwt: <token> }` |
- **Processing**:
  1. `requireAuthMiddleware` extracts and validates JWT from session cookie
  2. Queries `Payment.find({ userId: req.currentUser.id })` 
  3. Populates the `order` reference on each payment
- **Response**:
  - **Success (200)**:
    ```typescript
    {
      payments: Array<{
        id: string;           // MongoDB ObjectId as string
        order: {              // Populated Order document
          id: string;
          version: number;
          price: number;
          userId: string;
          status: "created" | "cancelled" | "awaiting:payment" | "complete";
        };
        stripeId: string;     // Stripe charge ID
        userId: string;       // User who made the payment
      }>
    }
    ```
  - **Error cases**:
    | Status | Condition | Body |
    |--------|-----------|------|
    | 401 | No valid JWT in session | `{ errors: [{ message: "Not authorized" }] }` |
- **Side effects**: None

---

### GET /api/payments/:id

- **Controller**: `services/payments/src/routes/show.ts`, anonymous async function (line 26)
- **Authentication**: `requireAuthMiddleware` — verifies JWT in session cookie
- **Authorization**: Query filter ensures payment belongs to authenticated user via `userId` match
- **Request**:
  - Path parameters:
    | Name | Type | Description |
    |------|------|-------------|
    | `id` | string (MongoDB ObjectId) | Payment document ID |
  - Query parameters: None
  - Request body: None
  - Headers:
    | Header | Required | Description |
    |--------|----------|-------------|
    | `Cookie` | Yes | Session cookie containing base64-encoded `{ jwt: <token> }` |
- **Validation** (`services/payments/src/routes/show.ts:24`):
  - `id`: Must not be empty, must be valid MongoDB ObjectId (via `isValidObjectId`)
- **Processing**:
  1. `requireAuthMiddleware` validates JWT
  2. `validateRequestMiddleware` checks express-validator results
  3. Queries `Payment.findOne({ _id: req.params.id, userId: req.currentUser.id })`
  4. Populates the `order` reference
  5. Throws `NotFoundError` if no payment found (includes case where payment exists but belongs to different user)
- **Response**:
  - **Success (200)**:
    ```typescript
    {
      payment: {
        id: string;
        order: {
          id: string;
          version: number;
          price: number;
          userId: string;
          status: "created" | "cancelled" | "awaiting:payment" | "complete";
        };
        stripeId: string;
        userId: string;
      }
    }
    ```
  - **Error cases**:
    | Status | Condition | Body |
    |--------|-----------|------|
    | 400 | Invalid payment ID format | `{ errors: [{ message: "Payment ID is required", field: "id" }] }` |
    | 401 | No valid JWT in session | `{ errors: [{ message: "Not authorized" }] }` |
    | 404 | Payment not found or belongs to different user | `{ errors: [{ message: "Not found" }] }` |
- **Side effects**: None

---

### POST /api/payments

- **Controller**: `services/payments/src/routes/new.ts`, anonymous async function (line 24)
- **Authentication**: `requireAuthMiddleware` — verifies JWT in session cookie
- **Authorization**: 
  - Verifies `order.userId === req.currentUser.id` (line 32-34)
  - Throws `NotAuthorizedError` if user doesn't own the order
- **Request**:
  - Path parameters: None
  - Query parameters: None
  - Request body:
    ```typescript
    {
      token: string;    // Stripe payment token (from Stripe.js/Elements)
      orderId: string;  // MongoDB ObjectId of the order to pay for
    }
    ```
  - Headers:
    | Header | Required | Description |
    |--------|----------|-------------|
    | `Cookie` | Yes | Session cookie containing base64-encoded `{ jwt: <token> }` |
    | `Content-Type` | Yes | `application/json` |
- **Validation** (`services/payments/src/routes/new.ts:21-23`):
  - `token`: Must not be empty
  - `orderId`: Must not be empty
- **Processing**:
  1. `requireAuthMiddleware` validates JWT
  2. `validateRequestMiddleware` checks express-validator results
  3. Queries `Order.findById(orderId)` (line 27)
  4. Throws `NotFoundError` if order doesn't exist (line 29-31)
  5. Throws `NotAuthorizedError` if `order.userId !== req.currentUser.id` (line 32-34)
  6. Throws `BadRequestError("Cannot pay for a cancelled order")` if `order.status === OrderStatus.Cancelled` (line 35-37)
  7. Calls `stripe.charges.create({ currency: 'usd', amount: order.price * 100, source: token })` (line 39-43)
  8. Throws `BadRequestError("Payment failed")` if `charge.paid === false` (line 45-47)
  9. Creates Payment document via `Payment.build({ order, stripeId: charge.id, userId: order.userId })` (line 49-53)
  10. Saves payment to database (line 54)
  11. Publishes `payment:complete` event with `{ orderId: payment.order.id }` (line 56-58)
- **Response**:
  - **Success (201)**:
    ```typescript
    {
      success: true;
      payment: {
        id: string;
        order: string;      // Order ObjectId (not populated)
        stripeId: string;   // Stripe charge ID (e.g., "ch_...")
        userId: string;
      }
    }
    ```
  - **Error cases**:
    | Status | Condition | Body |
    |--------|-----------|------|
    | 400 | Missing/empty `token` or `orderId` | `{ errors: [{ message: "Invalid value", field: "token" | "orderId" }] }` |
    | 400 | Order is cancelled | `{ errors: [{ message: "Cannot pay for a cancelled order" }] }` |
    | 400 | Stripe charge failed (`paid: false`) | `{ errors: [{ message: "Payment failed" }] }` |
    | 401 | No valid JWT in session | `{ errors: [{ message: "Not authorized" }] }` |
    | 401 | User doesn't own the order | `{ errors: [{ message: "Not authorized" }] }` |
    | 404 | Order not found | `{ errors: [{ message: "Not found" }] }` |
- **Side effects**:
  - **Stripe API call**: Creates a charge via `stripe.charges.create()`
  - **Database write**: Inserts new Payment document
  - **Event published**: `payment:complete` to NATS Streaming with payload `{ orderId: string }`

---

## Part B: Database Schema

### Order (Local Copy)

- **Database collection name**: `orders`
- **Purpose**: Local replica of Order documents from the Orders service, used to validate payment requests and track order state for payment processing. Enables the Payments service to operate independently without synchronous calls to the Orders service.

#### Fields

| Field | Type | Required | Default | Index | Description |
|-------|------|----------|---------|-------|-------------|
| `_id` | ObjectId | Yes | None | Primary | Matches the Order ID from Orders service (set explicitly via `Order.build()`) |
| `version` | Number | Yes | None | None | Optimistic concurrency version, must match `event.version - 1` for updates |
| `price` | Number | Yes | None | None | Order total in USD (dollars, not cents) |
| `userId` | String | Yes | None | None | User ID who owns the order |
| `status` | String (enum) | Yes | None | None | Current order status from `OrderStatus` enum |

#### Relationships

- **Referenced by**: `Payment.order` — Payment documents reference this Order via ObjectId

#### Enums

**OrderStatus** (from `@b.anik/ticketing-common`):

| Value | Description |
|-------|-------------|
| `created` | Order has been created, awaiting further action |
| `cancelled` | Order has been cancelled, cannot be paid |
| `awaiting:payment` | Order is awaiting payment |
| `complete` | Order has been paid and completed |

#### Static Methods

| Method | Signature | Description | Source |
|--------|-----------|-------------|--------|
| `build` | `(attrs: OrderAttrs) => OrderDoc` | Creates Order with explicit `_id` from event data. Validates `id` is valid ObjectId. | `order.ts:55-66` |
| `findByEvent` | `(event: { id: string; version: number }) => Promise<OrderDoc \| null>` | Finds order by ID where `version === event.version - 1` for optimistic concurrency | `order.ts:68-76` |

#### Common Query Patterns

| Pattern | Location | Description |
|---------|----------|-------------|
| Find by ID | `routes/new.ts:27` | `Order.findById(orderId)` — retrieve order for payment validation |
| Find by event (OCC) | `listeners/order-cancelled-listener.ts:11`, `listeners/order-updated-listener.ts:13-15` | `Order.findByEvent({ id, version })` — find order with matching ID and version-1 for safe concurrent updates |

#### JSON Transformation

Configured in schema `toJSON` transform (`order.ts:45-52`):
- Renames `_id` to `id`
- Removes `_id` and `__v` fields

---

### Payment

- **Database collection name**: `payments`
- **Purpose**: Records completed Stripe payments, linking the Stripe charge to the local order copy and the user who made the payment.

#### Fields

| Field | Type | Required | Default | Index | Description |
|-------|------|----------|---------|-------|-------------|
| `_id` | ObjectId | Yes | Auto-generated | Primary | Unique payment identifier |
| `order` | ObjectId (ref: Order) | Yes | None | None | Reference to the local Order document |
| `stripeId` | String | Yes | None | None | Stripe charge ID returned from `stripe.charges.create()` |
| `userId` | String | Yes | None | None | User ID who made the payment (copied from order.userId) |

#### Relationships

- **References**: `Order` — via `order` field (ObjectId reference with `ref: 'Order'`)
- Payment documents are populated with order data using `.populate('order')` in read operations

#### Static Methods

| Method | Signature | Description | Source |
|--------|-----------|-------------|--------|
| `build` | `(attrs: PaymentAttrs) => PaymentDoc` | Creates new Payment document from attributes | `payment.ts:47-49` |

#### Common Query Patterns

| Pattern | Location | Description |
|---------|----------|-------------|
| Find by user | `routes/show.ts:14-16` | `Payment.find({ userId }).populate('order')` — list all payments for a user |
| Find by ID and user | `routes/show.ts:27-30` | `Payment.findOne({ _id: id, userId }).populate('order')` — get single payment owned by user |
| Find by order | `routes/__tests__/new.test.ts:122-124` | `Payment.findOne({ order: orderId })` — find payment for specific order (used in tests) |

#### JSON Transformation

Configured in schema `toJSON` transform (`payment.ts:36-44`):
- Renames `_id` to `id`
- Removes `_id` and `__v` fields

---

## Part C: Type Definitions & DTOs

### OrderAttrs

- **File**: `services/payments/src/models/order.ts:4-10`
- **Used in**: `Order.build()` static method, event listeners when creating local order copies
- **Definition**:
  ```typescript
  interface OrderAttrs {
    id: string;           // Order ID from Orders service (becomes _id)
    version: number;      // Optimistic concurrency version
    price: number;        // Order total in USD
    userId: string;       // Owner user ID
    status: OrderStatus;  // Current order status
  }
  ```

---

### OrderDoc

- **File**: `services/payments/src/models/order.ts:12-17`
- **Used in**: Return type of `Order.findById()`, `Order.findByEvent()`, stored in `Payment.order` when populated
- **Definition**:
  ```typescript
  interface OrderDoc extends mongoose.Document {
    version: number;
    price: number;
    userId: string;
    status: OrderStatus;
  }
  ```

---

### OrderModel

- **File**: `services/payments/src/models/order.ts:19-22`
- **Used in**: Mongoose model typing for static methods
- **Definition**:
  ```typescript
  interface OrderModel extends mongoose.Model<OrderDoc> {
    build(attrs: OrderAttrs): OrderDoc;
    findByEvent(event: { id: string; version: number }): Promise<OrderDoc | null>;
  }
  ```

---

### PaymentAttrs

- **File**: `services/payments/src/models/payment.ts:4-8`
- **Used in**: `Payment.build()` static method, `POST /api/payments` route when creating new payment
- **Definition**:
  ```typescript
  interface PaymentAttrs {
    order: OrderDoc;      // Full Order document reference
    stripeId: string;     // Stripe charge ID
    userId: string;       // User who made the payment
  }
  ```

---

### PaymentDoc

- **File**: `services/payments/src/models/payment.ts:10-14`
- **Used in**: Return type of Payment queries, response body in payment endpoints
- **Definition**:
  ```typescript
  interface PaymentDoc extends mongoose.Document {
    order: OrderDoc;      // Populated when using .populate('order')
    stripeId: string;
    userId: string;
  }
  ```

---

### PaymentModel

- **File**: `services/payments/src/models/payment.ts:16-18`
- **Used in**: Mongoose model typing for static methods
- **Definition**:
  ```typescript
  interface PaymentModel extends mongoose.Model<PaymentDoc> {
    build(attrs: PaymentAttrs): PaymentDoc;
  }
  ```

---

### JWT Payload (currentUser)

- **File**: `services/payments/src/test/utils.ts:3-7` (test utility reveals shape)
- **Used in**: Decoded from session cookie by `currentUserMiddleware`, accessed as `req.currentUser`
- **Definition**:
  ```typescript
  interface JwtPayload {
    id: string;       // User ID
    email: string;    // User email
  }
  ```

---

## Part D: Event Contracts

### Events Consumed

#### order:created

- **Listener**: `OrderCreatedListener` (`services/payments/src/events/listeners/order-created-listener.ts`)
- **Subject**: `Subjects.OrderCreated` (string: `"order:created"`)
- **Queue Group**: `payments-service`
- **Event Data Shape** (from test file `order-created-listener.test.ts:9-19`):
  ```typescript
  {
    id: string;              // Order ID (MongoDB ObjectId)
    version: number;         // Always 0 for new orders
    status: OrderStatus;     // Typically "created"
    userId: string;          // User who placed the order
    expiresAt: string;       // ISO date string (not used by this service)
    ticket: {
      id: string;            // Ticket ID
      price: number;         // Ticket price in USD
    }
  }
  ```
- **Processing**:
  1. Creates local Order document via `Order.build({ id, version, price: ticket.price, userId, status })`
  2. Saves order to database
  3. Publishes `payment:awaited` event with `{ orderId: order.id }`

---

#### order:cancelled

- **Listener**: `OrderCancelledListener` (`services/payments/src/events/listeners/order-cancelled-listener.ts`)
- **Subject**: `Subjects.OrderCancelled` (string: `"order:cancelled"`)
- **Queue Group**: `payments-service`
- **Event Data Shape** (from test file `order-cancelled-listener.test.ts:24-30`):
  ```typescript
  {
    id: string;              // Order ID
    version: number;         // New version after cancellation
    ticket: {
      id: string;            // Ticket ID
    }
  }
  ```
- **Processing**:
  1. Finds order via `Order.findByEvent({ id, version })` (expects local version = event.version - 1)
  2. Throws error if order not found
  3. Updates order: `{ status: OrderStatus.Cancelled, version: data.version }`
  4. Saves updated order

---

#### order:updated

- **Listener**: `OrderUpdatedListener` (`services/payments/src/events/listeners/order-updated-listener.ts`)
- **Subject**: `Subjects.OrderUpdated` (string: `"order:updated"`)
- **Queue Group**: `payments-service`
- **Event Data Shape** (from test file `order-updated-listener.test.ts:24-28`):
  ```typescript
  {
    id: string;              // Order ID
    version: number;         // New version
    status: OrderStatus;     // New status
  }
  ```
- **Processing**:
  1. Finds order via `Order.findByEvent({ id, version })`
  2. Throws error if order not found
  3. Updates order: `{ version, status }`
  4. Saves updated order

---

### Events Published

#### payment:awaited

- **Publisher**: `PaymentAwaitedPublisher` (`services/payments/src/events/publishers/payment-awaited-publisher.ts`)
- **Subject**: `Subjects.PaymentAwaited` (string: `"payment:awaited"`)
- **Triggered by**: `OrderCreatedListener.onMessage()` after creating local order copy
- **Event Data Shape**:
  ```typescript
  {
    orderId: string;         // Order ID that is awaiting payment
  }
  ```

---

#### payment:complete

- **Publisher**: `PaymentCompletePublisher` (`services/payments/src/events/publishers/payment-complete-publisher.ts`)
- **Subject**: `Subjects.PaymentComplete` (string: `"payment:complete"`)
- **Triggered by**: `POST /api/payments` route after successful Stripe charge and Payment save
- **Event Data Shape** (from `routes/new.ts:56-58`):
  ```typescript
  {
    orderId: string;         // Order ID that was paid
  }
  ```

---

## Part E: External Service Dependencies

### Stripe

- **Client Configuration**: `services/payments/src/stripe.ts`
- **API Version**: `2024-04-10`
- **Methods Used**:
  | Method | Parameters | Response Fields Used |
  |--------|------------|---------------------|
  | `stripe.charges.create()` | `{ currency: 'usd', amount: number, source: string }` | `id` (charge ID), `paid` (boolean) |

### NATS Streaming

- **Wrapper**: `services/payments/src/nats-wrapper.ts`
- **Connection Parameters**:
  - Cluster ID: `NATS_CLUSTER_ID` env var (default: `ticketing` per K8s config)
  - Client ID: `NATS_CLIENT_ID` env var (pod name in K8s)
  - URL: `NATS_URL` env var (default: `http://nats-srv:4222`)
- **Queue Group**: `payments-service` — ensures only one listener instance processes each event

### MongoDB

- **Connection**: `mongoose.connect(MONGO_URI)` in `src/index.ts:51`
- **Database**: `payments` (from K8s config: `mongodb://payments-mongo-srv:27017/payments`)
