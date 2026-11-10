---
service_id: ticketing-app/tickets
service_name: tickets
prompt_id: 04-business-logic
prompt_label: Business Logic Extraction
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.286885
schema_version: "2.0"
---

# Business Processes: Tickets Service

## Process: Service Initialization

**Trigger**: Process start via `pnpm run dev` or `pnpm run start` command

**Input**: Environment variables: `JWT_KEY`, `MONGO_URI`, `NATS_URL`, `NATS_CLUSTER_ID`, `NATS_CLIENT_ID`

**Preconditions**: None (this is the bootstrap process)

**Steps**:

1. [`start()` in `services/tickets/src/index.ts:7-57`] — Async startup function invoked immediately at module load
   - Validates that `JWT_KEY` environment variable is defined (`index.ts:8-10`)
     - If undefined: throws `Error('JWT_KEY must be defined')` and process crashes
   - Validates that `MONGO_URI` environment variable is defined (`index.ts:12-14`)
     - If undefined: throws `Error('MONGO_URI must be defined')` and process crashes
   - Validates that `NATS_URL` environment variable is defined (`index.ts:16-18`)
     - If undefined: throws `Error('NATS_URL must be defined')` and process crashes
   - Validates that `NATS_CLUSTER_ID` environment variable is defined (`index.ts:20-22` and duplicate at `index.ts:24-26`)
     - If undefined: throws `Error('NATS_CLUSTER_ID must be defined')` and process crashes

2. [`natsWrapper.connect()` in `services/tickets/src/nats-wrapper.ts:15-23`] — Establishes NATS Streaming connection
   - Calls `nats.connect()` with `clusterId`, `clientId`, and `url` parameters (`nats-wrapper.ts:16`)
   - Returns a Promise that resolves on `'connect'` event or rejects on `'error'` event (`nats-wrapper.ts:18-21`)
   - On success: logs `'Connected to NATS!'` (`index.ts:34`)

3. [`natsWrapper.client.on('close', ...)` in `services/tickets/src/index.ts:36-39`] — Registers NATS close handler
   - On connection close: logs `'NATS connection closed!'` and exits process with code 1

4. [Process signal handlers in `services/tickets/src/index.ts:41-43`] — Registers graceful shutdown handlers
   - `SIGINT`: closes NATS client
   - `SIGTERM`: closes NATS client
   - `SIGUSR2`: closes NATS client

5. [`startListeners()` in `services/tickets/src/events/listeners/index.ts:5-8`] — Initializes event listeners
   - Creates new `OrderCreatedListener(client)` and calls `listen()` (`index.ts:6`)
   - Creates new `OrderCancelledListener(client)` and calls `listen()` (`index.ts:7`)
   - Both listeners use queue group name `'tickets-service'` (`queue-group-name.ts:1`)

6. [`mongoose.connect()` in `services/tickets/src/index.ts:47`] — Establishes MongoDB connection
   - Connects using `MONGO_URI` environment variable
   - On success: logs `'Connected to MongoDB!'` (`index.ts:48`)

7. [`app.listen(3000)` in `services/tickets/src/index.ts:54-56`] — Starts HTTP server
   - Listens on port 3000
   - On success: logs `'Listening on port 3000!'`

**Business rules**:
- All five environment variables (`JWT_KEY`, `MONGO_URI`, `NATS_URL`, `NATS_CLUSTER_ID`, `NATS_CLIENT_ID`) must be defined for the service to start
- NATS connection must succeed before MongoDB connection is attempted
- Event listeners are started after NATS connection but before MongoDB connection
- If any connection fails in the try block, the error is logged and process exits with code 1 (`index.ts:49-52`)

**Output**: Running HTTP server on port 3000 with active NATS and MongoDB connections

**Post-conditions**:
- HTTP server accepting requests on port 3000
- NATS Streaming client connected and subscribed to `order:created` and `order:cancelled` events
- MongoDB connection established

**Error scenarios**:
- Missing `JWT_KEY`: throws Error, process crashes before try block
- Missing `MONGO_URI`: throws Error, process crashes before try block
- Missing `NATS_URL`: throws Error, process crashes before try block
- Missing `NATS_CLUSTER_ID`: throws Error, process crashes before try block
- NATS connection failure: error logged, `process.exit(1)` called (`index.ts:49-52`)
- MongoDB connection failure: error logged, `process.exit(1)` called (`index.ts:49-52`)

---

## Process: Create Ticket

**Trigger**: HTTP POST request to `/api/tickets`

**Input**: 
- Request body: `{ title: string, price: number }`
- Session cookie containing JWT with user identity

**Preconditions**:
- User must be authenticated (valid JWT in session cookie)

**Steps**:

1. [`json()` middleware in `services/tickets/src/app.ts:25`] — Parses JSON request body
   - Populates `req.body` with parsed JSON

2. [`cookieSession()` middleware in `services/tickets/src/app.ts:26-31`] — Extracts session from cookie
   - Configuration: `signed: false`, `secure: true` (unless `NODE_ENV === 'test'`)
   - Populates `req.session` with decoded session object

3. [`currentUserMiddleware` from `@b.anik/ticketing-common` in `services/tickets/src/app.ts:36`] — Extracts current user from JWT
   - Reads JWT from `req.session.jwt`
   - Verifies and decodes JWT
   - Populates `req.currentUser` with `{ id, email }` payload
   - If no valid JWT: `req.currentUser` remains undefined

4. [`requireAuthMiddleware` from `@b.anik/ticketing-common` in `services/tickets/src/routes/new.ts:13`] — Enforces authentication
   - If `req.currentUser` is undefined: throws `NotAuthorizedError` (results in HTTP 401)

5. [`body('title').trim().notEmpty()` validation in `services/tickets/src/routes/new.ts:14`] — Validates title
   - Trims whitespace from title
   - If empty after trim: adds validation error `'Title is required'`

6. [`body('price').isFloat({ gt: 0 })` validation in `services/tickets/src/routes/new.ts:15`] — Validates price
   - Checks if price is a float greater than 0
   - If invalid: adds validation error `'Price must be greater than 0'`

7. [`validateRequestMiddleware` from `@b.anik/ticketing-common` in `services/tickets/src/routes/new.ts:16`] — Aggregates validation errors
   - If any validation errors exist: throws `RequestValidationError` (results in HTTP 400)

8. [`Ticket.build()` in `services/tickets/src/routes/new.ts:20-24`] — Creates ticket document instance
   - Factory function defined in `services/tickets/src/models/ticket.ts:51-53`
   - Creates new Ticket with `title`, `price`, and `userId` from `req.currentUser.id`

9. [`ticket.save()` in `services/tickets/src/routes/new.ts:26`] — Persists ticket to MongoDB
   - **Database write**: Inserts new document into `tickets` collection
   - **Fields written**: `title` (string), `price` (number), `userId` (string), `version` (number, starts at 0)
   - Returns saved document with generated `_id` and `version: 0`

10. [`new TicketCreatedPublisher(natsWrapper.client).publish()` in `services/tickets/src/routes/new.ts:28-34`] — Publishes event to NATS
    - **Event published**: `ticket:created` (`Subjects.TicketCreated`)
    - **Payload**: `{ id, version, title, price, userId }`
    - Publisher class defined in `services/tickets/src/events/publishers/ticket-created-publisher.ts:3-5`

11. [Response in `services/tickets/src/routes/new.ts:36-38`] — Returns created ticket
    - HTTP status: 201 Created
    - Response body: `{ ticket: <saved ticket document> }`

**Business rules**:
- Title must not be empty after trimming whitespace
- Price must be a float strictly greater than 0 (not 0 or negative)
- Only authenticated users can create tickets
- The ticket owner (`userId`) is automatically set from the authenticated user's JWT `id` claim — callers cannot specify their own `userId`
- Version field is managed by `mongoose-update-if-current` plugin, starting at 0

**Output**: 
```json
{
  "ticket": {
    "id": "<mongodb ObjectId>",
    "title": "<provided title>",
    "price": <provided price>,
    "userId": "<authenticated user id>",
    "version": 0
  }
}
```

**Post-conditions**:
- New ticket document exists in MongoDB `tickets` collection
- `ticket:created` event published to NATS Streaming

**Error scenarios**:
- Not authenticated (no valid JWT): HTTP 401, `NotAuthorizedError` thrown by `requireAuthMiddleware`
- Empty title: HTTP 400, `RequestValidationError` with message `'Title is required'`
- Missing title: HTTP 400, `RequestValidationError` with message `'Title is required'`
- Price ≤ 0: HTTP 400, `RequestValidationError` with message `'Price must be greater than 0'`
- Missing price: HTTP 400, `RequestValidationError` with message `'Price must be greater than 0'`

---

## Process: List All Tickets

**Trigger**: HTTP GET request to `/api/tickets`

**Input**: None (no request body, no route parameters)

**Preconditions**: None (public endpoint, no authentication required)

**Steps**:

1. [`json()` middleware in `services/tickets/src/app.ts:25`] — Parses JSON request body (no-op for GET)

2. [`cookieSession()` middleware in `services/tickets/src/app.ts:26-31`] — Extracts session from cookie

3. [`currentUserMiddleware` from `@b.anik/ticketing-common` in `services/tickets/src/app.ts:36`] — Extracts current user (optional for this route)

4. [`Ticket.find({})` in `services/tickets/src/routes/show.ts:25`] — Queries all tickets
   - **Database read**: Finds all documents in `tickets` collection with no filter
   - Returns array of ticket documents

5. [Response in `services/tickets/src/routes/show.ts:27`] — Returns all tickets
   - HTTP status: 200 OK
   - Response body: `{ tickets: <array of ticket documents> }`

**Business rules**:
- All tickets are returned regardless of ownership or reservation status
- No pagination, filtering, or sorting is applied
- Empty array returned if no tickets exist

**Output**:
```json
{
  "tickets": [
    {
      "id": "<mongodb ObjectId>",
      "title": "<title>",
      "price": <price>,
      "userId": "<owner user id>",
      "version": <version number>,
      "orderId": "<order id if reserved, otherwise absent>"
    }
  ]
}
```

**Post-conditions**: None (read-only operation)

**Error scenarios**: None specific to this endpoint (only infrastructure failures)

---

## Process: Get Single Ticket

**Trigger**: HTTP GET request to `/api/tickets/:id`

**Input**: 
- Route parameter: `id` (string, expected to be MongoDB ObjectId)

**Preconditions**: None (public endpoint, no authentication required)

**Steps**:

1. [`json()` middleware in `services/tickets/src/app.ts:25`] — Parses JSON request body (no-op for GET)

2. [`cookieSession()` middleware in `services/tickets/src/app.ts:26-31`] — Extracts session from cookie

3. [`currentUserMiddleware` from `@b.anik/ticketing-common` in `services/tickets/src/app.ts:36`] — Extracts current user (optional for this route)

4. [`isValidObjectId(id)` check in `services/tickets/src/routes/show.ts:11-13`] — Validates ID format
   - Uses `mongoose.isValidObjectId()` to check if ID is valid MongoDB ObjectId format
   - If invalid: throws `NotFoundError` (results in HTTP 404)

5. [`Ticket.findById(id)` in `services/tickets/src/routes/show.ts:15`] — Queries ticket by ID
   - **Database read**: Finds single document in `tickets` collection by `_id`
   - Returns ticket document or `null`

6. [Existence check in `services/tickets/src/routes/show.ts:17-19`] — Validates ticket exists
   - If `ticket` is `null`: throws `NotFoundError` (results in HTTP 404)

7. [Response in `services/tickets/src/routes/show.ts:21`] — Returns ticket
   - HTTP status: 200 OK
   - Response body: `{ ticket: <ticket document> }`

**Business rules**:
- ID must be a valid MongoDB ObjectId format
- Ticket must exist in the database
- Any user (authenticated or not) can view any ticket

**Output**:
```json
{
  "ticket": {
    "id": "<mongodb ObjectId>",
    "title": "<title>",
    "price": <price>,
    "userId": "<owner user id>",
    "version": <version number>,
    "orderId": "<order id if reserved, otherwise absent>"
  }
}
```

**Post-conditions**: None (read-only operation)

**Error scenarios**:
- Invalid ObjectId format: HTTP 404, `NotFoundError`
- Ticket does not exist: HTTP 404, `NotFoundError`

---

## Process: Update Ticket

**Trigger**: HTTP PUT request to `/api/tickets/:id`

**Input**:
- Route parameter: `id` (string, expected to be MongoDB ObjectId)
- Request body: `{ title: string, price: number }`
- Session cookie containing JWT with user identity

**Preconditions**:
- User must be authenticated
- Ticket must exist
- Ticket must not be reserved (no `orderId`)
- User must be the ticket owner

**Steps**:

1. [`json()` middleware in `services/tickets/src/app.ts:25`] — Parses JSON request body

2. [`cookieSession()` middleware in `services/tickets/src/app.ts:26-31`] — Extracts session from cookie

3. [`currentUserMiddleware` from `@b.anik/ticketing-common` in `services/tickets/src/app.ts:36`] — Extracts current user from JWT

4. [`requireAuthMiddleware` from `@b.anik/ticketing-common` in `services/tickets/src/routes/update.ts:19`] — Enforces authentication
   - If `req.currentUser` is undefined: throws `NotAuthorizedError` (results in HTTP 401)

5. [`body('title').trim().notEmpty()` validation in `services/tickets/src/routes/update.ts:20`] — Validates title
   - Trims whitespace from title
   - If empty after trim: adds validation error `'Title is required'`

6. [`body('price').isFloat({ gt: 0 })` validation in `services/tickets/src/routes/update.ts:21`] — Validates price
   - Checks if price is a float greater than 0
   - If invalid: adds validation error `'Price must be greater than 0'`

7. [`validateRequestMiddleware` from `@b.anik/ticketing-common` in `services/tickets/src/routes/update.ts:22`] — Aggregates validation errors
   - If any validation errors exist: throws `RequestValidationError` (results in HTTP 400)

8. [`isValidObjectId(id)` check in `services/tickets/src/routes/update.ts:26-28`] — Validates ID format
   - Uses `mongoose.isValidObjectId()` to check if ID is valid MongoDB ObjectId format
   - If invalid: throws `NotFoundError` (results in HTTP 404)

9. [`Ticket.findById(id)` in `services/tickets/src/routes/update.ts:30`] — Queries ticket by ID
   - **Database read**: Finds single document in `tickets` collection by `_id`
   - Returns ticket document or `null`

10. [Existence check in `services/tickets/src/routes/update.ts:32-34`] — Validates ticket exists
    - If `ticket` is `null`: throws `NotFoundError` (results in HTTP 404)

11. [Reservation check in `services/tickets/src/routes/update.ts:36-38`] — Validates ticket is not reserved
    - If `ticket.orderId` is truthy: throws `BadRequestError('Cannot edit a reserved ticket')` (results in HTTP 400)

12. [Ownership check in `services/tickets/src/routes/update.ts:40-42`] — Validates user owns ticket
    - Compares `ticket.userId` with `req.currentUser.id`
    - If not equal: throws `NotAuthorizedError` (results in HTTP 401)

13. [Field assignment in `services/tickets/src/routes/update.ts:44-45`] — Updates ticket fields
    - Sets `ticket.title = req.body.title`
    - Sets `ticket.price = req.body.price`

14. [`ticket.save()` in `services/tickets/src/routes/update.ts:47`] — Persists updated ticket
    - **Database write**: Updates document in `tickets` collection
    - **Fields updated**: `title`, `price`, `version` (auto-incremented by `mongoose-update-if-current`)
    - Optimistic concurrency control: update only succeeds if document version matches

15. [`new TicketUpdatedPublisher(natsWrapper.client).publish()` in `services/tickets/src/routes/update.ts:49-55`] — Publishes event to NATS
    - **Event published**: `ticket:updated` (`Subjects.TicketUpdated`)
    - **Payload**: `{ id, version, title, price, userId }`
    - Note: `orderId` is not included in this payload (unlike event-driven updates)

16. [Response in `services/tickets/src/routes/update.ts:57-59`] — Returns updated ticket
    - HTTP status: 200 OK
    - Response body: `{ ticket: <updated ticket document> }`

**Business rules**:
- Only authenticated users can update tickets
- Title must not be empty after trimming whitespace
- Price must be a float strictly greater than 0
- Only the ticket owner (matching `userId`) can update the ticket
- Reserved tickets (those with an `orderId`) cannot be edited — the purpose is to prevent price changes while a purchase is in progress (inferred)
- Version is automatically incremented on each successful save via `mongoose-update-if-current` plugin
- Ownership check happens after reservation check

**Output**:
```json
{
  "ticket": {
    "id": "<mongodb ObjectId>",
    "title": "<new title>",
    "price": <new price>,
    "userId": "<owner user id>",
    "version": <incremented version number>
  }
}
```

**Post-conditions**:
- Ticket document updated in MongoDB with new `title`, `price`, and incremented `version`
- `ticket:updated` event published to NATS Streaming

**Error scenarios**:
- Not authenticated: HTTP 401, `NotAuthorizedError` thrown by `requireAuthMiddleware`
- Empty title: HTTP 400, `RequestValidationError` with message `'Title is required'`
- Price ≤ 0: HTTP 400, `RequestValidationError` with message `'Price must be greater than 0'`
- Invalid ObjectId format: HTTP 404, `NotFoundError`
- Ticket does not exist: HTTP 404, `NotFoundError`
- Ticket is reserved (has `orderId`): HTTP 400, `BadRequestError` with message `'Cannot edit a reserved ticket'`
- User is not the owner: HTTP 401, `NotAuthorizedError`
- Optimistic concurrency conflict: thrown by Mongoose when document version doesn't match — error message contains `'version'` (observed in tests at `ticket.test.ts:25`)

---

## Process: Delete Ticket

**Trigger**: HTTP DELETE request to `/api/tickets/:id`

**Input**:
- Route parameter: `id` (string, expected to be MongoDB ObjectId)
- Session cookie containing JWT with user identity

**Preconditions**:
- User must be authenticated
- Ticket must exist
- Ticket must not be reserved (no `orderId`)
- User must be the ticket owner

**Steps**:

1. [`json()` middleware in `services/tickets/src/app.ts:25`] — Parses JSON request body (no-op for DELETE)

2. [`cookieSession()` middleware in `services/tickets/src/app.ts:26-31`] — Extracts session from cookie

3. [`currentUserMiddleware` from `@b.anik/ticketing-common` in `services/tickets/src/app.ts:36`] — Extracts current user from JWT

4. [`requireAuthMiddleware` from `@b.anik/ticketing-common` in `services/tickets/src/routes/delete.ts:13`] — Enforces authentication
   - If `req.currentUser` is undefined: throws `NotAuthorizedError` (results in HTTP 401)

5. [`isValidObjectId(id)` check in `services/tickets/src/routes/delete.ts:16-18`] — Validates ID format
   - Uses `mongoose.isValidObjectId()` to check if ID is valid MongoDB ObjectId format
   - If invalid: throws `NotFoundError` (results in HTTP 404)

6. [`Ticket.findById(id)` in `services/tickets/src/routes/delete.ts:20`] — Queries ticket by ID
   - **Database read**: Finds single document in `tickets` collection by `_id`
   - Returns ticket document or `null`

7. [Existence check in `services/tickets/src/routes/delete.ts:22-24`] — Validates ticket exists
   - If `ticket` is `null`: throws `NotFoundError` (results in HTTP 404)

8. [Ownership check in `services/tickets/src/routes/delete.ts:26-28`] — Validates user owns ticket
   - Compares `ticket.userId` with `req.currentUser.id`
   - If not equal: throws `NotAuthorizedError` (results in HTTP 401)

9. [Reservation check in `services/tickets/src/routes/delete.ts:30-32`] — Validates ticket is not reserved
   - If `ticket.orderId` is truthy: throws `BadRequestError('Cannot delete a reserved ticket')` (results in HTTP 400)

10. [`Ticket.findOneAndDelete()` in `services/tickets/src/routes/delete.ts:34-37`] — Deletes ticket from database
    - **Database write**: Deletes document from `tickets` collection
    - Filter: `{ _id: ticket.id, userId: req.currentUser.id }` (double-checks ownership)
    - Returns deleted document

11. [Response in `services/tickets/src/routes/delete.ts:39`] — Returns deleted ticket
    - HTTP status: 200 OK
    - Response body: `{ ticket: <deleted ticket document> }`

**Business rules**:
- Only authenticated users can delete tickets
- Only the ticket owner (matching `userId`) can delete the ticket
- Reserved tickets (those with an `orderId`) cannot be deleted — the purpose is to prevent removal while a purchase is in progress (inferred)
- Ownership check happens before reservation check (different order than update)
- Delete operation includes redundant ownership filter in `findOneAndDelete` query
- Note: No `ticket:deleted` event is published (unlike create and update)

**Output**:
```json
{
  "ticket": {
    "id": "<mongodb ObjectId>",
    "title": "<title>",
    "price": <price>,
    "userId": "<owner user id>",
    "version": <version number>
  }
}
```

**Post-conditions**:
- Ticket document removed from MongoDB `tickets` collection
- Note: No event is published to notify other services of deletion

**Error scenarios**:
- Not authenticated: HTTP 401, `NotAuthorizedError` thrown by `requireAuthMiddleware`
- Invalid ObjectId format: HTTP 404, `NotFoundError`
- Ticket does not exist: HTTP 404, `NotFoundError`
- User is not the owner: HTTP 401, `NotAuthorizedError`
- Ticket is reserved (has `orderId`): HTTP 400, `BadRequestError` with message `'Cannot delete a reserved ticket'`

---

## Process: Reserve Ticket (Order Created)

**Trigger**: NATS Streaming event `order:created` (subject: `Subjects.OrderCreated`)

**Input**: Event payload:
```typescript
{
  id: string;           // Order ID
  version: number;
  status: OrderStatus;
  userId: string;
  expiresAt: string;    // ISO date string
  ticket: {
    id: string;         // Ticket ID to reserve
    price: number;
  };
}
```

**Preconditions**:
- Ticket with matching ID must exist in database

**Steps**:

1. [`OrderCreatedListener.onMessage()` in `services/tickets/src/events/listeners/order-created-listener.ts:11-40`] — Handles incoming event
   - Extracts `ticketId` from `data.ticket.id` (`order-created-listener.ts:14`)

2. [`Ticket.findById(ticketId)` in `services/tickets/src/events/listeners/order-created-listener.ts:17`] — Queries ticket by ID
   - **Database read**: Finds single document in `tickets` collection by `_id`
   - Returns ticket document or `null`

3. [Existence check in `services/tickets/src/events/listeners/order-created-listener.ts:19-21`] — Validates ticket exists
   - If `ticket` is `null`: throws `Error('Ticket not found')` and rejects promise

4. [Set orderId in `services/tickets/src/events/listeners/order-created-listener.ts:23`] — Marks ticket as reserved
   - Sets `ticket.orderId = data.id` (the order ID)

5. [`ticket.save()` in `services/tickets/src/events/listeners/order-created-listener.ts:24`] — Persists reservation
   - **Database write**: Updates document in `tickets` collection
   - **Fields updated**: `orderId`, `version` (auto-incremented)

6. [`new TicketUpdatedPublisher(this.client).publish()` in `services/tickets/src/events/listeners/order-created-listener.ts:26-33`] — Publishes update event
   - **Event published**: `ticket:updated` (`Subjects.TicketUpdated`)
   - **Payload**: `{ id, version, title, price, userId, orderId }`
   - Note: This payload includes `orderId` unlike HTTP update endpoint

7. [Promise resolution in `services/tickets/src/events/listeners/order-created-listener.ts:35`] — Completes processing
   - Resolves promise to signal successful message processing
   - Base `Listener` class will acknowledge the message

**Business rules**:
- When an order is created for a ticket, the ticket becomes "reserved" by storing the order ID
- A reserved ticket cannot be edited or deleted through HTTP endpoints
- The ticket's version is incremented to maintain concurrency across services
- Queue group name `'tickets-service'` ensures only one instance of this service processes each message (`queue-group-name.ts:1`)
- The purpose of publishing `ticket:updated` after setting `orderId` is to synchronize the reserved state with other services (inferred)

**Output**: Promise resolves (message acknowledged)

**Post-conditions**:
- Ticket document has `orderId` field set to the order ID
- Ticket document version incremented (e.g., 0 → 1)
- `ticket:updated` event published with reservation data

**Error scenarios**:
- Ticket not found: `Error('Ticket not found')` thrown, promise rejected (`order-created-listener.ts:20`)
  - Message will not be acknowledged; NATS will redeliver

---

## Process: Release Ticket Reservation (Order Cancelled)

**Trigger**: NATS Streaming event `order:cancelled` (subject: `Subjects.OrderCancelled`)

**Input**: Event payload:
```typescript
{
  id: string;           // Order ID
  version: number;
  ticket: {
    id: string;         // Ticket ID to release
  };
}
```

**Preconditions**:
- Ticket with matching ID must exist in database
- Ticket must have `orderId` matching the cancelled order ID

**Steps**:

1. [`OrderCancelledListener.onMessage()` in `services/tickets/src/events/listeners/order-cancelled-listener.ts:11-45`] — Handles incoming event
   - Extracts `ticketId` from `data.ticket.id` (`order-cancelled-listener.ts:17`)

2. [`Ticket.findOne()` in `services/tickets/src/events/listeners/order-cancelled-listener.ts:19-22`] — Queries ticket by ID and orderId
   - **Database read**: Finds document in `tickets` collection matching both `_id` and `orderId`
   - Query: `{ _id: ticketId, orderId: data.id }`
   - Returns ticket document or `null`

3. [Existence check in `services/tickets/src/events/listeners/order-cancelled-listener.ts:24-26`] — Validates ticket exists with matching order
   - If `ticket` is `null`: throws `Error('Ticket not found')` and rejects promise
   - This catches both "ticket doesn't exist" and "ticket's orderId doesn't match cancelled order"

4. [Clear orderId in `services/tickets/src/events/listeners/order-cancelled-listener.ts:28`] — Releases reservation
   - Sets `ticket.orderId = undefined`

5. [`ticket.save()` in `services/tickets/src/events/listeners/order-cancelled-listener.ts:29`] — Persists release
   - **Database write**: Updates document in `tickets` collection
   - **Fields updated**: `orderId` (set to undefined/removed), `version` (auto-incremented)

6. [`new TicketUpdatedPublisher(this.client).publish()` in `services/tickets/src/events/listeners/order-cancelled-listener.ts:31-38`] — Publishes update event
   - **Event published**: `ticket:updated` (`Subjects.TicketUpdated`)
   - **Payload**: `{ id, version, title, price, userId, orderId }` where `orderId` is `undefined`

7. [Promise resolution in `services/tickets/src/events/listeners/order-cancelled-listener.ts:40`] — Completes processing
   - Resolves promise to signal successful message processing

**Business rules**:
- When an order is cancelled, the associated ticket reservation is released
- The query validates both ticket ID and order ID match — this prevents releasing a reservation that belongs to a different order (inferred: handles race conditions where ticket was re-reserved)
- A released ticket can be edited and deleted again through HTTP endpoints
- The ticket's version is incremented to maintain concurrency across services
- Queue group name `'tickets-service'` ensures only one instance processes each message

**Output**: Promise resolves (message acknowledged)

**Post-conditions**:
- Ticket document has `orderId` field cleared (undefined)
- Ticket document version incremented (e.g., 1 → 2 if previously reserved, or 2 → 3 per test at `order-cancelled-listener.test.ts:89`)
- `ticket:updated` event published with cleared reservation

**Error scenarios**:
- Ticket not found: `Error('Ticket not found')` thrown, promise rejected (`order-cancelled-listener.ts:25`)
- Ticket's `orderId` doesn't match cancelled order ID: `Error('Ticket not found')` thrown (same error, verified in test at `order-cancelled-listener.test.ts:55-64`)
  - Message will not be acknowledged; NATS will redeliver

---

## Process: Health Check

**Trigger**: HTTP GET request to `/api/tickets/health`

**Input**: None

**Preconditions**: None

**Steps**:

1. [Route handler in `services/tickets/src/routes/health.ts:5`] — Returns health status
   - HTTP status: 200 OK
   - Response body: `{}` (empty object)

**Business rules**:
- This endpoint is registered before `currentUserMiddleware` (`app.ts:33` vs `app.ts:36`), so authentication is not applied
- Used for Kubernetes liveness probes (inferred from typical health endpoint usage)

**Output**:
```json
{}
```

**Post-conditions**: None (stateless check)

**Error scenarios**: None specific to this endpoint

---

## Cross-Cutting Concerns

### Optimistic Concurrency Control

The `mongoose-update-if-current` plugin (`models/ticket.ts:56`) implements optimistic concurrency control:
- Every ticket document has a `version` field, starting at 0
- On each `save()`, the plugin increments the version
- The update query includes `{ version: <current version> }` in its filter
- If another process modified the document (changing its version), the update fails with an error containing `'version'` (`ticket.test.ts:25`)
- This prevents lost updates when multiple processes modify the same ticket

### JSON Transformation

The ticket schema's `toJSON` transform (`models/ticket.ts:41-47`):
- Renames `_id` to `id`
- Removes `__v` field
- Removes `_id` field (after copying to `id`)

### Cookie Session Configuration

Configured in `app.ts:26-31`:
- `signed: false` — cookies are not cryptographically signed
- `secure: true` — cookies only sent over HTTPS (disabled when `NODE_ENV === 'test'`)

### Error Handling

All errors are handled by `errorHandlerMiddleware` from `@b.anik/ticketing-common` (`app.ts:48`), which converts error classes to standardized JSON responses:
- `NotFoundError` → HTTP 404
- `NotAuthorizedError` → HTTP 401
- `BadRequestError` → HTTP 400
- `RequestValidationError` → HTTP 400 with validation details

### 404 Catch-All

Any unmatched route throws `NotFoundError` (`app.ts:45-47`), resulting in HTTP 404.
