---
service_id: ticketing-app/tickets
service_name: tickets
prompt_id: 02-api-data-contracts
prompt_label: API & Data Contracts
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.24393000000000004
schema_version: "2.0"
---

# API Contracts & Data Models: Tickets Service

## Part A: HTTP API Endpoints

### GET /api/tickets/health

- **Controller**: `services/tickets/src/routes/health.ts`, anonymous inline handler (line 5)
- **Authentication**: None
- **Authorization**: None
- **Request**:
  - Path parameters: None
  - Query parameters: None
  - Request body: None
  - Headers: None required
- **Processing**:
  1. Immediately returns an empty 200 response
- **Response**:
  - **Success (200 OK)**:
    ```json
    {}
    ```
  - **Error cases**: None
- **Side effects**: None

---

### GET /api/tickets

- **Controller**: `services/tickets/src/routes/show.ts`, anonymous async handler (lines 24-28)
- **Authentication**: None (health router is registered before `currentUserMiddleware`)
- **Authorization**: None (public endpoint)
- **Request**:
  - Path parameters: None
  - Query parameters: None
  - Request body: None
  - Headers: None required
- **Processing**:
  1. Calls `Ticket.find({})` to retrieve all tickets from MongoDB
  2. Returns the array of tickets
- **Response**:
  - **Success (200 OK)**:
    ```typescript
    {
      tickets: Array<{
        id: string;        // MongoDB ObjectId as string (transformed from _id)
        title: string;
        price: number;
        userId: string;
        version: number;
        orderId?: string;  // Present when ticket is reserved
      }>
    }
    ```
  - **Error cases**: None expected (empty array returned if no tickets)
- **Side effects**: None

---

### GET /api/tickets/:id

- **Controller**: `services/tickets/src/routes/show.ts`, anonymous async handler (lines 8-22)
- **Authentication**: None
- **Authorization**: None (public endpoint)
- **Request**:
  - Path parameters:
    | Name | Type | Description |
    |------|------|-------------|
    | `id` | string | MongoDB ObjectId of the ticket |
  - Query parameters: None
  - Request body: None
  - Headers: None required
- **Processing**:
  1. Validates `id` parameter with `mongoose.isValidObjectId(id)` (line 11)
  2. If invalid ObjectId format, throws `NotFoundError` (line 12)
  3. Calls `Ticket.findById(id)` to retrieve the ticket (line 15)
  4. If ticket not found, throws `NotFoundError` (line 18)
  5. Returns the ticket document
- **Response**:
  - **Success (200 OK)**:
    ```typescript
    {
      ticket: {
        id: string;
        title: string;
        price: number;
        userId: string;
        version: number;
        orderId?: string;
      }
    }
    ```
  - **Error cases**:
    - **404 Not Found** (invalid ObjectId format or ticket not found):
      ```typescript
      {
        errors: Array<{ message: string }>
      }
      ```
      Error message: "Not found" (from `NotFoundError` in common module)
- **Side effects**: None

---

### POST /api/tickets

- **Controller**: `services/tickets/src/routes/new.ts`, anonymous async handler (lines 17-39)
- **Authentication**: `requireAuthMiddleware` (line 13) — requires valid JWT in session cookie
- **Authorization**: Any authenticated user can create tickets
- **Request**:
  - Path parameters: None
  - Query parameters: None
  - Request body (validated by express-validator, lines 14-16):
    | Field | Type | Required | Validation | Description |
    |-------|------|----------|------------|-------------|
    | `title` | string | Yes | Must be non-empty after trimming | Title of the ticket |
    | `price` | number | Yes | Must be a float greater than 0 | Price of the ticket |
  - Headers:
    - `Cookie`: Session cookie containing base64-encoded `{ jwt: <token> }` where JWT payload has `{ id: string, email: string }` (extracted by `currentUserMiddleware`)
- **Processing**:
  1. `requireAuthMiddleware` verifies `req.currentUser` exists (throws 401 if not)
  2. `validateRequestMiddleware` checks validation results (throws 400 with errors array if invalid)
  3. Extracts `title` and `price` from request body (line 18)
  4. Builds ticket document with `Ticket.build({ title, price, userId: req.currentUser.id })` (lines 20-24)
  5. Saves ticket to MongoDB with `ticket.save()` (line 26)
  6. Publishes `ticket:created` event to NATS with ticket data (lines 28-34)
  7. Returns created ticket with 201 status
- **Response**:
  - **Success (201 Created)**:
    ```typescript
    {
      ticket: {
        id: string;
        title: string;
        price: number;
        userId: string;
        version: number;  // Starts at 0
      }
    }
    ```
  - **Error cases**:
    - **401 Unauthorized** (not authenticated):
      ```typescript
      {
        errors: [{ message: "Not authorized" }]
      }
      ```
    - **400 Bad Request** (validation failure):
      ```typescript
      {
        errors: Array<{
          message: string;  // e.g., "Title is required" or "Price must be greater than 0"
          field?: string;   // e.g., "title" or "price"
        }>
      }
      ```
- **Side effects**:
  - Publishes `ticket:created` event to NATS Streaming with subject `ticket:created` (line 28)
  - Event payload:
    ```typescript
    {
      id: string;
      version: number;
      title: string;
      price: number;
      userId: string;
    }
    ```

---

### PUT /api/tickets/:id

- **Controller**: `services/tickets/src/routes/update.ts`, anonymous async handler (lines 23-60)
- **Authentication**: `requireAuthMiddleware` (line 19) — requires valid JWT in session cookie
- **Authorization**: Only the ticket owner (`ticket.userId === req.currentUser.id`) can update; ticket must not be reserved (`orderId` must be unset)
- **Request**:
  - Path parameters:
    | Name | Type | Description |
    |------|------|-------------|
    | `id` | string | MongoDB ObjectId of the ticket to update |
  - Query parameters: None
  - Request body (validated by express-validator, lines 20-22):
    | Field | Type | Required | Validation | Description |
    |-------|------|----------|------------|-------------|
    | `title` | string | Yes | Must be non-empty after trimming | New title |
    | `price` | number | Yes | Must be a float greater than 0 | New price |
  - Headers:
    - `Cookie`: Session cookie with JWT
- **Processing**:
  1. `requireAuthMiddleware` verifies authentication (throws 401 if not authenticated)
  2. Validators check `title` and `price` (throws 400 if invalid)
  3. `validateRequestMiddleware` aggregates validation errors
  4. Validates `id` parameter with `isValidObjectId()` (line 26)
  5. If invalid ObjectId, throws `NotFoundError` (line 27)
  6. Finds ticket by ID (line 30)
  7. If ticket not found, throws `NotFoundError` (line 33)
  8. If ticket has `orderId` (is reserved), throws `BadRequestError('Cannot edit a reserved ticket')` (lines 36-38)
  9. If `ticket.userId !== req.currentUser.id`, throws `NotAuthorizedError` (lines 40-42)
  10. Updates `ticket.title` and `ticket.price` from request body (lines 44-45)
  11. Saves updated ticket (line 47) — version is auto-incremented by `mongoose-update-if-current` plugin
  12. Publishes `ticket:updated` event to NATS (lines 49-55)
  13. Returns updated ticket
- **Response**:
  - **Success (200 OK)**:
    ```typescript
    {
      ticket: {
        id: string;
        title: string;
        price: number;
        userId: string;
        version: number;  // Incremented from previous version
        orderId?: string; // Will not be present on success (reserved tickets can't be updated)
      }
    }
    ```
  - **Error cases**:
    - **401 Unauthorized** (not authenticated):
      ```typescript
      { errors: [{ message: "Not authorized" }] }
      ```
    - **401 Unauthorized** (not the ticket owner):
      ```typescript
      { errors: [{ message: "Not authorized" }] }
      ```
    - **404 Not Found** (invalid ObjectId or ticket not found):
      ```typescript
      { errors: [{ message: "Not found" }] }
      ```
    - **400 Bad Request** (validation failure):
      ```typescript
      { errors: Array<{ message: string; field?: string }> }
      ```
    - **400 Bad Request** (ticket is reserved):
      ```typescript
      { errors: [{ message: "Cannot edit a reserved ticket" }] }
      ```
- **Side effects**:
  - Publishes `ticket:updated` event to NATS Streaming with subject `ticket:updated`
  - Event payload:
    ```typescript
    {
      id: string;
      version: number;
      title: string;
      price: number;
      userId: string;
      // Note: orderId not included in update from HTTP endpoint (only from event listeners)
    }
    ```

---

### DELETE /api/tickets/:id

- **Controller**: `services/tickets/src/routes/delete.ts`, anonymous async handler (lines 13-40)
- **Authentication**: `requireAuthMiddleware` (line 13) — requires valid JWT in session cookie
- **Authorization**: Only the ticket owner can delete; ticket must not be reserved
- **Request**:
  - Path parameters:
    | Name | Type | Description |
    |------|------|-------------|
    | `id` | string | MongoDB ObjectId of the ticket to delete |
  - Query parameters: None
  - Request body: None
  - Headers:
    - `Cookie`: Session cookie with JWT
- **Processing**:
  1. `requireAuthMiddleware` verifies authentication (throws 401 if not)
  2. Validates `id` parameter with `isValidObjectId()` (line 16)
  3. If invalid ObjectId, throws `NotFoundError` (line 17)
  4. Finds ticket by ID (line 20)
  5. If ticket not found, throws `NotFoundError` (line 23)
  6. If `ticket.userId !== req.currentUser.id`, throws `NotAuthorizedError` (lines 25-27)
  7. If ticket has `orderId` (is reserved), throws `BadRequestError('Cannot delete a reserved ticket')` (lines 29-31)
  8. Deletes ticket with `Ticket.findOneAndDelete({ _id: ticket.id, userId: req.currentUser.id })` (lines 33-36)
  9. Returns deleted ticket document
- **Response**:
  - **Success (200 OK)**:
    ```typescript
    {
      ticket: {
        id: string;
        title: string;
        price: number;
        userId: string;
        version: number;
      }
    }
    ```
  - **Error cases**:
    - **401 Unauthorized** (not authenticated):
      ```typescript
      { errors: [{ message: "Not authorized" }] }
      ```
    - **401 Unauthorized** (not the ticket owner):
      ```typescript
      { errors: [{ message: "Not authorized" }] }
      ```
    - **404 Not Found** (invalid ObjectId or ticket not found):
      ```typescript
      { errors: [{ message: "Not found" }] }
      ```
    - **400 Bad Request** (ticket is reserved):
      ```typescript
      { errors: [{ message: "Cannot delete a reserved ticket" }] }
      ```
- **Side effects**: None (no event published on delete)

---

## Part B: Database Schema

### Ticket

- **Database table/collection name**: `tickets` (Mongoose pluralizes model name "Ticket")
- **Purpose**: Represents a ticket for sale in the ticketing platform. Tracks ownership, pricing, and reservation status.

**Fields**:

| Field | Type | Required | Default | Index | Description |
|-------|------|----------|---------|-------|-------------|
| `_id` | ObjectId | Yes | Auto-generated | Primary key (default MongoDB index) | Unique identifier for the ticket |
| `title` | String | Yes | None | None | Title/name of the ticket (e.g., "Concert", "Sports Event") |
| `price` | Number | Yes | None | None | Price of the ticket in dollars (must be > 0 per validation) |
| `userId` | String | Yes | None | None | User ID of the ticket owner/seller (from JWT) |
| `orderId` | String | No | `undefined` | None | Order ID when ticket is reserved; `undefined` when available |
| `version` | Number | Yes | 0 (first save) | None | Optimistic concurrency control version; increments on every save |

**Relationships**:
- `userId`: References a user in the Auth service (no foreign key constraint — stored as string, not ObjectId reference)
- `orderId`: References an order in the Orders service when ticket is reserved (no foreign key constraint — stored as string, not ObjectId reference)

**Enums**: None defined in this model (OrderStatus enum used in events is from common module)

**Compound indexes**: None

**JSON Transformation** (defined in schema options, `services/tickets/src/models/ticket.ts:41-48`):
```typescript
{
  toJSON: {
    transform: (_doc, ret) => {
      ret.id = ret._id;  // Adds `id` field
      delete ret.__v;    // Removes __v (version key moved to `version`)
      delete ret._id;    // Removes _id (replaced by `id`)
    },
  },
}
```

**Concurrency Control** (`services/tickets/src/models/ticket.ts:55-56`):
- Uses `mongoose-update-if-current` plugin
- Version key renamed from `__v` to `version`
- Prevents concurrent updates by checking version number in update queries

**Common Query Patterns** (inferred from route handlers):
1. **Find by ID**: `Ticket.findById(id)` — used in show, update, delete routes
2. **Find all**: `Ticket.find({})` — used in list endpoint
3. **Find by ID and orderId**: `Ticket.findOne({ _id: ticketId, orderId: orderId })` — used in `OrderCancelledListener` to verify order-ticket relationship before releasing reservation
4. **Delete by ID and userId**: `Ticket.findOneAndDelete({ _id: ticket.id, userId: req.currentUser.id })` — atomic delete with ownership check

---

## Part C: Type Definitions & DTOs

### TicketAttrs

- **Name**: `TicketAttrs`
- **File path**: `services/tickets/src/models/ticket.ts:4-8`
- **Definition**:
  ```typescript
  interface TicketAttrs {
    title: string;
    price: number;
    userId: string;
  }
  ```
- **Where used**: Input interface for `Ticket.build()` static factory method; used when creating new tickets in POST `/api/tickets` handler and in event listeners

---

### TicketDoc

- **Name**: `TicketDoc`
- **File path**: `services/tickets/src/models/ticket.ts:10-16`
- **Definition**:
  ```typescript
  export interface TicketDoc extends mongoose.Document {
    title: string;
    price: number;
    userId: string;
    version: number;
    orderId?: string;
  }
  ```
- **Where used**: Return type for Mongoose queries; used throughout route handlers and event listeners for type-safe ticket document access

---

### TicketModel

- **Name**: `TicketModel`
- **File path**: `services/tickets/src/models/ticket.ts:18-20`
- **Definition**:
  ```typescript
  interface TicketModel extends mongoose.Model<any> {
    build(attrs: TicketAttrs): TicketDoc;
  }
  ```
- **Where used**: Internal type for the Mongoose model with static `build` method

---

### TicketCreatedEvent (inferred from publisher usage)

- **Name**: `TicketCreatedEvent['data']`
- **File path**: External — `@b.anik/ticketing-common` package
- **Definition** (inferred from publisher call in `services/tickets/src/routes/new.ts:28-34`):
  ```typescript
  interface TicketCreatedEventData {
    id: string;
    version: number;
    title: string;
    price: number;
    userId: string;
  }
  ```
- **Where used**: Event payload published when a ticket is created via POST `/api/tickets`

---

### TicketUpdatedEvent (inferred from publisher usage)

- **Name**: `TicketUpdatedEvent['data']`
- **File path**: External — `@b.anik/ticketing-common` package
- **Definition** (inferred from publisher calls in `services/tickets/src/routes/update.ts:49-55` and `services/tickets/src/events/listeners/order-created-listener.ts:26-33`):
  ```typescript
  interface TicketUpdatedEventData {
    id: string;
    version: number;
    title: string;
    price: number;
    userId: string;
    orderId?: string;  // Included when ticket is reserved or unreserved
  }
  ```
- **Where used**: Event payload published when ticket is updated via PUT `/api/tickets/:id` or when reservation status changes via event listeners

---

### OrderCreatedEvent (inferred from listener usage)

- **Name**: `OrderCreatedEvent['data']`
- **File path**: External — `@b.anik/ticketing-common` package
- **Definition** (inferred from `services/tickets/src/events/listeners/__tests__/order-created-listener.test.ts:22-32` and `services/orders/src/routes/new.ts:56-66`):
  ```typescript
  interface OrderCreatedEventData {
    id: string;              // Order ID
    version: number;         // Order version
    status: OrderStatus;     // OrderStatus.Created
    userId: string;          // User who placed the order
    expiresAt: string;       // ISO date string when order expires
    ticket: {
      id: string;            // Ticket ID being reserved
      price: number;         // Ticket price at time of order
    };
  }
  ```
- **Where used**: Consumed by `OrderCreatedListener` to mark tickets as reserved by setting `orderId`

---

### OrderCancelledEvent (inferred from listener usage)

- **Name**: `OrderCancelledEvent['data']`
- **File path**: External — `@b.anik/ticketing-common` package
- **Definition** (inferred from `services/tickets/src/events/listeners/__tests__/order-cancelled-listener.test.ts:27-33` and `services/orders/src/routes/cancel.ts:50-56`):
  ```typescript
  interface OrderCancelledEventData {
    id: string;        // Order ID
    version: number;   // Order version
    ticket: {
      id: string;      // Ticket ID to unreserve
    };
  }
  ```
- **Where used**: Consumed by `OrderCancelledListener` to release ticket reservation by clearing `orderId`

---

### OrderStatus (referenced in events)

- **Name**: `OrderStatus`
- **File path**: External — `@b.anik/ticketing-common` package
- **Definition** (inferred from test imports and usage):
  ```typescript
  enum OrderStatus {
    Created = 'created',
    Cancelled = 'cancelled',
    AwaitingPayment = 'awaiting:payment',
    Complete = 'complete'
  }
  ```
- **Where used**: Status field in `OrderCreatedEvent`; imported in `services/tickets/src/events/listeners/__tests__/order-created-listener.test.ts:1`

---

### JWT Session Payload

- **Name**: Session JWT payload
- **File path**: `services/tickets/src/test/utils.ts:4-6` (structure inferred from test helper)
- **Definition**:
  ```typescript
  interface JwtPayload {
    id: string;    // User ID
    email: string; // User email
  }
  ```
- **Where used**: Extracted by `currentUserMiddleware` into `req.currentUser`; used for authentication and authorization in all protected routes

---

### currentUser (from middleware)

- **Name**: `req.currentUser`
- **File path**: Set by `currentUserMiddleware` from `@b.anik/ticketing-common`
- **Definition** (inferred from usage in route handlers):
  ```typescript
  interface CurrentUser {
    id: string;
    email: string;
  }
  // Available as req.currentUser | undefined
  ```
- **Where used**: Accessed in POST, PUT, DELETE handlers to get authenticated user's ID for ownership checks

---

### API Error Response Shape

- **Name**: Error response
- **File path**: Produced by `errorHandlerMiddleware` from `@b.anik/ticketing-common`
- **Definition** (inferred from test assertions):
  ```typescript
  interface ErrorResponse {
    errors: Array<{
      message: string;
      field?: string;  // Present for validation errors
    }>;
  }
  ```
- **Where used**: All error responses (400, 401, 404) follow this shape

---

## Event Integration Summary

### Published Events

| Event Subject | Publisher Class | File Path | Trigger |
|---------------|-----------------|-----------|---------|
| `ticket:created` | `TicketCreatedPublisher` | `services/tickets/src/events/publishers/ticket-created-publisher.ts` | POST `/api/tickets` success |
| `ticket:updated` | `TicketUpdatedPublisher` | `services/tickets/src/events/publishers/ticket-updated-publisher.ts` | PUT `/api/tickets/:id` success; `order:created` received; `order:cancelled` received |

### Consumed Events

| Event Subject | Listener Class | File Path | Queue Group | Action |
|---------------|----------------|-----------|-------------|--------|
| `order:created` | `OrderCreatedListener` | `services/tickets/src/events/listeners/order-created-listener.ts` | `tickets-service` | Sets `orderId` on ticket to mark as reserved; publishes `ticket:updated` |
| `order:cancelled` | `OrderCancelledListener` | `services/tickets/src/events/listeners/order-cancelled-listener.ts` | `tickets-service` | Clears `orderId` on ticket to release reservation; publishes `ticket:updated` |

---

## Middleware Chain

Order of middleware execution as configured in `services/tickets/src/app.ts`:

1. `json()` (body-parser) — parses JSON request bodies (line 25)
2. `cookieSession({ signed: false, secure: <not-test> })` — session management (lines 26-31)
3. `healthRouter` — handles `/api/tickets/health` before auth middleware (line 33)
4. `currentUserMiddleware` — extracts JWT from session cookie into `req.currentUser` (line 36)
5. Route handlers (`newTicketRouter`, `showTicketRouter`, `updateTicketRouter`, `deleteTicketRouter`) with their specific middleware:
   - `requireAuthMiddleware` — throws 401 if no `currentUser` (on protected routes)
   - `body()` validators from express-validator (on POST/PUT)
   - `validateRequestMiddleware` — throws 400 if validation fails
6. `app.all('*')` catch-all — throws `NotFoundError` for unmatched routes (lines 45-47)
7. `errorHandlerMiddleware` — converts errors to standardized JSON responses (line 48)
