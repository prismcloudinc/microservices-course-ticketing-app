---
service_id: ticketing-app/orders
service_name: orders
prompt_id: 01-service-identity
prompt_label: Service Identity & Anatomy
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.24999
schema_version: "2.0"
---

# Service Anatomy: Orders Service

## Section 1: Identity

| Attribute | Value |
|-----------|-------|
| **Package Name** | `orders` (`services/orders/package.json`, line 2) |
| **Version** | `1.0.0` (`services/orders/package.json`, line 3) |
| **Purpose** | Manages creation, retrieval, and cancellation of orders for tickets in the ticketing application |
| **Product/Platform** | Ticketing Application (microservices-based event ticketing platform) |
| **Runtime** | Node.js 20 (pinned via `Dockerfile` line 1: `FROM node:20-alpine` and `.nvmrc`: `20`) |
| **Framework** | Express.js v4.19.2 (resolved in `pnpm-lock.yaml`, line 22-23) |
| **Database** | MongoDB, using Mongoose ORM v8.3.3 (resolved in `pnpm-lock.yaml`, line 33-34), connected via environment variable `MONGO_URI` (`src/index.ts`, line 47) |
| **Message Broker** | NATS Streaming Server, using `node-nats-streaming` v0.3.2 (`pnpm-lock.yaml`, line 39-40) |

### Connection Method
- MongoDB: Direct connection via `mongoose.connect(process.env.MONGO_URI)` (`src/index.ts`, line 47)
- NATS: Connection via singleton wrapper using `natsWrapper.connect()` with cluster ID, client ID, and URL from environment variables (`src/index.ts`, lines 29-33)

---

## Section 2: Entry Points

| File Path | Trigger | Description |
|-----------|---------|-------------|
| `services/orders/src/index.ts` | Process start (CLI: `pnpm dev` or `pnpm start`) | Main application entry point that connects to NATS and MongoDB, starts event listeners, and launches HTTP server on port 3000 |

### HTTP Endpoints (triggered by HTTP requests)

| Route Handler | Method | Path | Description |
|---------------|--------|------|-------------|
| `src/routes/health.ts` | GET | `/api/orders/health` | Health check endpoint for Kubernetes liveness probes |
| `src/routes/show.ts` | GET | `/api/orders` | Returns all orders for the authenticated user |
| `src/routes/show.ts` | GET | `/api/orders/:orderId` | Returns a specific order by ID for the authenticated user |
| `src/routes/new.ts` | POST | `/api/orders` | Creates a new order for a ticket with 15-minute expiration window |
| `src/routes/cancel.ts` | PATCH | `/api/orders/:orderId/cancel` | Cancels an existing order owned by the authenticated user |

### Event Listeners (triggered by NATS messages)

| Listener | Event Subject | Description |
|----------|---------------|-------------|
| `src/events/listeners/ticket-created-listener.ts` | `ticket:created` | Creates a local copy of a ticket when a new ticket is created in the tickets service |
| `src/events/listeners/ticket-updated-listener.ts` | `ticket:updated` | Updates the local ticket copy when a ticket is updated in the tickets service |
| `src/events/listeners/expiration-complete-listener.ts` | `expiration:complete` | Cancels an order when its expiration timer fires (unless already complete or cancelled) |
| `src/events/listeners/payment-awaited-listener.ts` | `payment:awaited` | Updates order status to `AwaitingPayment` when payment processing begins |
| `src/events/listeners/payment-complete-listener.ts` | `payment:complete` | Updates order status to `Complete` when payment succeeds |

---

## Section 3: Directory Map

### `services/orders/` (root)

| File | Description |
|------|-------------|
| `.dockerignore` | Excludes `node_modules`, `dist`, `build`, and `.turbo` from Docker builds |
| `Dockerfile` | Multi-stage Docker build using `node:20-alpine`, installs pnpm@9, copies package files, and runs `pnpm run dev` |
| `jest.config.js` | Jest configuration extending `@repo/jest-config` with test setup file path |
| `package.json` | Package manifest defining scripts, dependencies, and devDependencies |
| `pnpm-lock.yaml` | Lockfile with resolved dependency versions |
| `tsconfig.json` | TypeScript configuration targeting ES2016, CommonJS modules, strict mode enabled |

### `services/orders/src/`

| File | Description |
|------|-------------|
| `app.ts` | Express application setup: configures middleware (JSON body parsing, cookie sessions, current user), mounts route handlers, and error handling |
| `index.ts` | Application bootstrap: validates environment variables, connects to NATS and MongoDB, starts event listeners, and launches HTTP server on port 3000 |
| `nats-wrapper.ts` | Singleton class wrapping NATS Streaming client; provides `connect()` method and exposes `client` property for publishing/subscribing |

### `services/orders/src/__mocks__/`

| File | Description |
|------|-------------|
| `nats-wrapper.ts` | Jest mock for NATS wrapper providing mock implementations of `connect`, `publish`, `on`, and `close` methods for testing |

### `services/orders/src/events/`

| File | Description |
|------|-------------|
| `index.ts` | Barrel export re-exporting all publishers and listeners |

### `services/orders/src/events/listeners/`

| File | Description |
|------|-------------|
| `index.ts` | Exports `startListeners()` function that initializes all event listeners with the NATS client |
| `queue-group-name.ts` | Exports the queue group name `'orders-service'` for NATS subscription load balancing |
| `ticket-created-listener.ts` | Listener for `ticket:created` events; creates a local Ticket document mirroring the tickets service data |
| `ticket-updated-listener.ts` | Listener for `ticket:updated` events; finds ticket by event ID and previous version, updates title/price/version |
| `expiration-complete-listener.ts` | Listener for `expiration:complete` events; cancels the order if not already completed/cancelled, publishes `order:cancelled` |
| `payment-awaited-listener.ts` | Listener for `payment:awaited` events; updates order status to `AwaitingPayment`, publishes `order:updated` |
| `payment-complete-listener.ts` | Listener for `payment:complete` events; updates order status to `Complete`, publishes `order:updated` |

### `services/orders/src/events/listeners/__tests__/`

| File | Description |
|------|-------------|
| `ticket-created-listener.test.ts` | Tests for TicketCreatedListener verifying ticket creation and promise resolution |
| `ticket-updated-listener.test.ts` | Tests for TicketUpdatedListener verifying ticket updates, version handling, and error cases |
| `expiration-complete-listener.test.ts` | Tests for ExpirationCompleteListener verifying order cancellation, event publishing, and edge cases |
| `payment-awaited-listener.test.ts` | Tests for PaymentAwaitedListener verifying status update to AwaitingPayment and event publishing |
| `payment-complete.listener.test.ts` | Tests for PaymentCompleteListener verifying status update to Complete and event publishing |

### `services/orders/src/events/publishers/`

| File | Description |
|------|-------------|
| `index.ts` | Barrel export re-exporting all publisher classes |
| `order-created-publisher.ts` | Publisher class for `order:created` events; extends base Publisher with OrderCreatedEvent type |
| `order-cancelled-publisher.ts` | Publisher class for `order:cancelled` events; extends base Publisher with OrderCancelledEvent type |
| `order-updated-publisher.ts` | Publisher class for `order:updated` events; extends base Publisher with OrderUpdatedEvent type |

### `services/orders/src/models/`

| File | Description |
|------|-------------|
| `index.ts` | Barrel export re-exporting Order, Ticket models and OrderStatus enum |
| `order.ts` | Mongoose model for orders with fields: userId, status (enum), expiresAt, ticket (ref); uses `mongoose-update-if-current` for optimistic concurrency control |
| `ticket.ts` | Mongoose model for local ticket copies with fields: title, price, version; includes `isReserved()` method to check if ticket has non-cancelled order, and `findByEvent()` static for version-aware lookups |

### `services/orders/src/models/__tests__/`

| File | Description |
|------|-------------|
| `order.test.ts` | Tests for Order model verifying optimistic concurrency control (version conflicts) and version incrementing |

### `services/orders/src/routes/`

| File | Description |
|------|-------------|
| `index.ts` | Barrel export re-exporting all route handlers |
| `health.ts` | Health check endpoint at `/api/orders/health` returning empty 200 response |
| `show.ts` | Handlers for GET `/api/orders` (list user's orders) and GET `/api/orders/:orderId` (get single order); requires authentication |
| `new.ts` | Handler for POST `/api/orders`; validates ticketId, checks ticket existence and reservation status, creates order with 15-minute expiration, publishes `order:created` event |
| `cancel.ts` | Handler for PATCH `/api/orders/:orderId/cancel`; validates ownership, rejects if already cancelled/completed, updates status to Cancelled, publishes `order:cancelled` event |

### `services/orders/src/routes/__tests__/`

| File | Description |
|------|-------------|
| `new.test.ts` | Tests for order creation endpoint: auth, validation, ticket existence, reservation checking, order creation, event publishing |
| `show.test.ts` | Tests for order retrieval endpoints: auth, user isolation, 404 handling, order population |
| `cancel.test.ts` | Tests for order cancellation endpoint: auth, ownership, already-cancelled/completed handling, status update, event publishing |

### `services/orders/src/test/`

| File | Description |
|------|-------------|
| `setup.ts` | Jest setup: creates in-memory MongoDB via `mongodb-memory-server`, mocks NATS wrapper, cleans collections between tests |
| `utils.ts` | Test utilities: `getCookieHeader()` function that generates signed JWT cookies for authenticated test requests |

---

## Section 4: Dependency Analysis

### Production Dependencies

| Package | Version (manifest) | Version (resolved) | Usage in Codebase |
|---------|-------------------|-------------------|-------------------|
| `@b.anik/ticketing-common` | ^0.1.2 | 0.1.2 | Shared library providing: `OrderStatus` enum (imported in `src/models/order.ts`, `src/routes/cancel.ts`, listeners); `Publisher`/`Listener` base classes (all publisher/listener files); Event types (`OrderCreatedEvent`, `OrderCancelledEvent`, `OrderUpdatedEvent`, `TicketCreatedEvent`, `TicketUpdatedEvent`, `ExpirationCompleteEvent`, `PaymentAwaitedEvent`, `PaymentCompleteEvent`); `Subjects` enum for event routing; Middleware (`currentUserMiddleware`, `requireAuthMiddleware`, `validateRequestMiddleware`, `errorHandlerMiddleware` in `src/app.ts`); Error classes (`NotFoundError`, `BadRequestError`, `NotAuthorizedError` in routes) |
| `body-parser` | ^1.20.2 | 1.20.2 | JSON body parsing middleware; imported in `src/app.ts` line 3 as `{ json }`, used on line 19: `app.use(json())` |
| `cookie-session` | ^2.1.0 | 2.1.0 | Cookie-based session middleware; imported in `src/app.ts` line 4, configured on lines 20-24 with `signed: false` and `secure: process.env.NODE_ENV !== 'test'` |
| `express` | ^4.18.3 | 4.19.2 | HTTP server framework; imported in `src/app.ts` line 1, all route files; creates Express app and routers |
| `express-async-errors` | ^3.1.1 | 3.1.1 | Async error handling; imported in `src/app.ts` line 2 for side-effect registration (enables throwing in async handlers) |
| `express-validator` | ^7.0.1 | 7.0.1 | Request validation; imported in `src/routes/new.ts` line 2 (`body`), `src/routes/show.ts` line 4 (`param`), `src/routes/cancel.ts` line 9 (`param`); validates `ticketId` and `orderId` parameters |
| `jsonwebtoken` | ^9.0.2 | 9.0.2 | JWT generation for test utilities; imported in `src/test/utils.ts` line 1, used to sign test authentication tokens on line 16 |
| `mongoose` | ^8.2.4 | 8.3.3 | MongoDB ODM; imported in `src/index.ts` (connection), `src/models/order.ts`, `src/models/ticket.ts` (schema definitions); provides Schema, Model, ObjectId validation |
| `mongoose-update-if-current` | ^1.4.0 | 1.4.0 | Optimistic concurrency control plugin; imported in `src/models/order.ts` line 4 as `updateIfCurrentPlugin`, applied to orderSchema on line 58 |
| `node-nats-streaming` | ^0.3.2 | 0.3.2 | NATS Streaming client; imported in `src/nats-wrapper.ts` line 1, `src/events/listeners/index.ts` line 1 (`Stan` type), individual listeners (`Message` type); provides `nats.connect()` and client methods |

### Development Dependencies

| Package | Version (manifest) | Version (resolved) | Usage in Codebase |
|---------|-------------------|-------------------|-------------------|
| `@repo/jest-config` | link:* | link:* | Internal monorepo package providing shared Jest configuration; imported in `jest.config.js` line 1 as `baseConfig`, spread into module.exports |
| `@types/body-parser` | ^1.19.5 | 1.19.5 | TypeScript type definitions for body-parser |
| `@types/cookie-session` | ^2.0.48 | 2.0.49 | TypeScript type definitions for cookie-session |
| `@types/express` | ^4.17.21 | 4.17.21 | TypeScript type definitions for Express |
| `@types/jsonwebtoken` | ^9.0.6 | 9.0.6 | TypeScript type definitions for jsonwebtoken |
| `@types/node` | ^20.11.30 | 20.12.8 | TypeScript type definitions for Node.js |
| `@types/supertest` | ^6.0.2 | 6.0.2 | TypeScript type definitions for supertest |
| `mongodb-memory-server` | ^9.1.8 | 9.2.0 | In-memory MongoDB for testing; imported in `src/test/setup.ts` line 1, creates ephemeral MongoDB instance in `beforeAll` |
| `nodemon` | ^3.1.0 | 3.1.0 | File watcher for development; used via `pnpm run dev` script (`nodemon src/index.ts`) |
| `supertest` | ^6.3.4 | 6.3.4 | HTTP assertion library; imported in all route test files as `request`, used to make HTTP requests against the Express app |
| `ts-node` | ^10.9.2 | 10.9.2 | TypeScript execution engine; used via `pnpm run start` script (`ts-node src/index.ts`) and by nodemon |
| `typescript` | ^5.4.2 | 5.4.5 | TypeScript compiler; used for type checking via `pnpm run typecheck` script |

---

## Section 5: Internal Package Usage

### `@b.anik/ticketing-common` (v0.1.2)

**Note**: This is a published npm package (`@b.anik/ticketing-common`), not a local internal package. However, it serves as a shared library for this microservices architecture and is documented here as per its organizational function.

#### Files Importing This Package

| File | Imports Used |
|------|-------------|
| `src/app.ts` | `NotFoundError`, `currentUserMiddleware`, `errorHandlerMiddleware` |
| `src/routes/new.ts` | `BadRequestError`, `requireAuthMiddleware`, `validateRequestMiddleware` |
| `src/routes/show.ts` | `NotAuthorizedError`, `NotFoundError`, `requireAuthMiddleware` |
| `src/routes/cancel.ts` | `BadRequestError`, `NotAuthorizedError`, `NotFoundError`, `OrderStatus`, `requireAuthMiddleware` |
| `src/models/order.ts` | `OrderStatus` |
| `src/events/publishers/order-created-publisher.ts` | `OrderCreatedEvent`, `Publisher`, `Subjects` |
| `src/events/publishers/order-cancelled-publisher.ts` | `OrderCancelledEvent`, `Publisher`, `Subjects` |
| `src/events/publishers/order-updated-publisher.ts` | `OrderUpdatedEvent`, `Publisher`, `Subjects` |
| `src/events/listeners/ticket-created-listener.ts` | `Listener`, `Subjects`, `TicketCreatedEvent` |
| `src/events/listeners/ticket-updated-listener.ts` | `Listener`, `Subjects`, `TicketUpdatedEvent` |
| `src/events/listeners/expiration-complete-listener.ts` | `ExpirationCompleteEvent`, `Listener`, `OrderStatus`, `Subjects` |
| `src/events/listeners/payment-awaited-listener.ts` | `Listener`, `OrderStatus`, `PaymentAwaitedEvent`, `Subjects` |
| `src/events/listeners/payment-complete-listener.ts` | `Listener`, `OrderStatus`, `PaymentCompleteEvent`, `Subjects` |
| `src/events/listeners/__tests__/expiration-complete-listener.test.ts` | `ExpirationCompleteEvent`, `OrderStatus` |
| `src/events/listeners/__tests__/ticket-created-listener.test.ts` | `TicketCreatedEvent` |
| `src/events/listeners/__tests__/ticket-updated-listener.test.ts` | `TicketUpdatedEvent` |
| `src/events/listeners/__tests__/payment-awaited-listener.test.ts` | `OrderStatus`, `PaymentAwaitedEvent` |
| `src/events/listeners/__tests__/payment-complete.listener.test.ts` | `OrderStatus`, `PaymentCompleteEvent` |

#### Exports Used

| Export | Type | Usage |
|--------|------|-------|
| `OrderStatus` | Enum | Order lifecycle states: `Created`, `AwaitingPayment`, `Complete`, `Cancelled`; used in Order model, route validation, and listeners |
| `Subjects` | Enum | NATS event subjects (channel names) for routing; e.g., `Subjects.OrderCreated`, `Subjects.TicketCreated` |
| `Publisher<T>` | Abstract Class | Base class for event publishers; extended by `OrderCreatedPublisher`, `OrderCancelledPublisher`, `OrderUpdatedPublisher` |
| `Listener<T>` | Abstract Class | Base class for event listeners with `onMessage()` abstract method and automatic message acknowledgement; extended by all listener classes |
| `OrderCreatedEvent` | Interface | Event payload type for order creation events |
| `OrderCancelledEvent` | Interface | Event payload type for order cancellation events |
| `OrderUpdatedEvent` | Interface | Event payload type for order update events |
| `TicketCreatedEvent` | Interface | Event payload type consumed when tickets service creates a ticket |
| `TicketUpdatedEvent` | Interface | Event payload type consumed when tickets service updates a ticket |
| `ExpirationCompleteEvent` | Interface | Event payload type consumed when expiration service signals order timeout |
| `PaymentAwaitedEvent` | Interface | Event payload type consumed when payments service begins processing |
| `PaymentCompleteEvent` | Interface | Event payload type consumed when payments service completes successfully |
| `currentUserMiddleware` | Middleware | Decodes JWT from cookie session and attaches `currentUser` to request; applied globally in `app.ts` |
| `requireAuthMiddleware` | Middleware | Throws `NotAuthorizedError` if `currentUser` is not set; used on all protected routes |
| `validateRequestMiddleware` | Middleware | Checks express-validator errors and throws `RequestValidationError` if present; used after validation chains |
| `errorHandlerMiddleware` | Middleware | Global error handler that formats errors into standardized JSON response; last middleware in chain |
| `NotFoundError` | Error Class | Thrown when requested resource doesn't exist; triggers 404 response |
| `BadRequestError` | Error Class | Thrown for invalid requests (ticket not found, already reserved, etc.); triggers 400 response |
| `NotAuthorizedError` | Error Class | Thrown when user lacks permission; triggers 401 response |

### `@repo/jest-config` (link:*)

**Type**: Internal monorepo package (workspace link)

**Location**: `packages/jest-config/`

**Files Importing**:
- `services/orders/jest.config.js` (line 1)

**Exports Used**:
- Default export (entire module): Provides base Jest configuration object with preset `ts-jest`, test environment `node`, verbose mode, and watch path ignore patterns

**Functionality Provided**: Centralized Jest configuration for TypeScript services in the monorepo, ensuring consistent test runner settings across all services.
