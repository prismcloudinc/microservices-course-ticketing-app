---
service_id: ticketing-app/payments
service_name: payments
prompt_id: 01-service-identity
prompt_label: Service Identity & Anatomy
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.214665
schema_version: "2.0"
---

# Service Anatomy: Payments Service

## Section 1: Identity

| Attribute | Value | Source |
|-----------|-------|--------|
| **Package name** | `payments` | `services/payments/package.json:2` |
| **Purpose** | Processes Stripe payments for orders in the ticketing application, tracks payment records, and emits payment-related events | Inferred from source code |
| **Product/platform** | Ticketing Application (Microservices Course) | `README.md` |
| **Runtime** | Node.js 20 | `Dockerfile:1` (`node:20-alpine`), `.nvmrc:1` |
| **Framework** | Express 4.18.3 | `package.json:23` |
| **Database** | MongoDB (via `payments-mongo-srv:27017/payments`) | `infra/k8s/payments-depl.yaml:25` |
| **ORM/Query library** | Mongoose 8.2.4 | `package.json:27` |
| **Connection method** | Direct connection via `mongoose.connect()` using `MONGO_URI` environment variable | `src/index.ts:51` |

### Environment Variables Required

| Variable | Purpose | Source |
|----------|---------|--------|
| `JWT_KEY` | JWT token verification secret | `src/index.ts:8-10` |
| `STRIPE_KEY` | Stripe API secret key for payment processing | `src/index.ts:12-14` |
| `MONGO_URI` | MongoDB connection string | `src/index.ts:16-18` |
| `NATS_URL` | NATS Streaming Server URL | `src/index.ts:20-22` |
| `NATS_CLUSTER_ID` | NATS cluster identifier | `src/index.ts:24-26` |
| `NATS_CLIENT_ID` | NATS client identifier (pod name in K8s) | `src/index.ts:35` |

---

## Section 2: Entry Points

| File Path | Trigger | Description |
|-----------|---------|-------------|
| `src/index.ts` | Application startup (manual or container start) | Main entry point: connects to NATS and MongoDB, starts event listeners, and launches Express HTTP server on port 3000 |
| `src/routes/new.ts` | HTTP POST `/api/payments` | Creates a new payment by charging Stripe, saves payment record, publishes `payment:complete` event |
| `src/routes/show.ts` | HTTP GET `/api/payments` | Returns all payments for the authenticated user |
| `src/routes/show.ts` | HTTP GET `/api/payments/:id` | Returns a specific payment by ID for the authenticated user |
| `src/routes/health.ts` | HTTP GET `/api/payments/health` | Kubernetes liveness probe endpoint returning HTTP 200 |
| `src/events/listeners/order-created-listener.ts` | NATS event `order:created` | Creates local Order record, publishes `payment:awaited` event |
| `src/events/listeners/order-cancelled-listener.ts` | NATS event `order:cancelled` | Updates local Order status to Cancelled |
| `src/events/listeners/order-updated-listener.ts` | NATS event `order:updated` | Updates local Order status and version |

---

## Section 3: Directory Map

### Root: `services/payments/`

| File | Description |
|------|-------------|
| `.dockerignore` | Excludes `node_modules`, `dist`, `build`, `.turbo` from Docker context |
| `Dockerfile` | Multi-stage Docker build using `node:20-alpine`, installs pnpm 9, runs `pnpm run dev` |
| `jest.config.js` | Jest configuration extending `@repo/jest-config`, sets up test setup file |
| `package.json` | Package manifest with dependencies and npm scripts |
| `pnpm-lock.yaml` | pnpm lockfile with resolved dependency versions |
| `tsconfig.json` | TypeScript configuration targeting ES2016, CommonJS modules, strict mode |

### Directory: `src/`

| File | Description |
|------|-------------|
| `app.ts` | Express application setup: configures body-parser, cookie-session, mounts routes and error handling middleware |
| `index.ts` | Application bootstrap: validates env vars, connects to NATS and MongoDB, starts event listeners, launches HTTP server on port 3000 |
| `nats-wrapper.ts` | Singleton wrapper for NATS Streaming client connection |
| `stripe.ts` | Exports configured Stripe client instance using `STRIPE_KEY` with API version `2024-04-10` |

### Directory: `src/__mocks__/`

| File | Description |
|------|-------------|
| `nats-wrapper.ts` | Jest mock for NATS wrapper: mocks `connect`, `client.publish`, `client.on`, `client.close` |
| `stripe.ts` | Jest mock for Stripe: mocks `charges.create` returning `{ id: 'stripe-id', paid: true }` |

### Directory: `src/events/`

| File | Description |
|------|-------------|
| `index.ts` | Re-exports all publishers and listeners |

### Directory: `src/events/listeners/`

| File | Description |
|------|-------------|
| `index.ts` | Exports `startListeners()` function that initializes all event listeners with the NATS client |
| `queue-group-name.ts` | Exports constant `queueGroupName = 'payments-service'` for NATS subscription queue group |
| `order-created-listener.ts` | Listens for `order:created` events: creates local Order record with id, version, price, userId, status; publishes `payment:awaited` event |
| `order-cancelled-listener.ts` | Listens for `order:cancelled` events: finds Order by event id/version, updates status to Cancelled |
| `order-updated-listener.ts` | Listens for `order:updated` events: finds Order by event id/version, updates status and version |

### Directory: `src/events/listeners/__tests__/`

| File | Description |
|------|-------------|
| `order-cancelled-listener.test.ts` | Tests for OrderCancelledListener: verifies order status update, version handling, error on not found |
| `order-created-listener.test.ts` | Tests for OrderCreatedListener: verifies order creation, PaymentAwaited event publishing |
| `order-updated-listener.test.ts` | Tests for OrderUpdatedListener: verifies order status and version updates |

### Directory: `src/events/publishers/`

| File | Description |
|------|-------------|
| `index.ts` | Re-exports PaymentCompletePublisher and PaymentAwaitedPublisher |
| `payment-awaited-publisher.ts` | Publisher for `payment:awaited` event with subject `Subjects.PaymentAwaited` |
| `payment-complete-publisher.ts` | Publisher for `payment:complete` event with subject `Subjects.PaymentComplete` |

### Directory: `src/models/`

| File | Description |
|------|-------------|
| `index.ts` | Re-exports Order and Payment models |
| `order.ts` | Mongoose model for Order: fields `version`, `price`, `userId`, `status`; static methods `build()` and `findByEvent()` for optimistic concurrency |
| `payment.ts` | Mongoose model for Payment: fields `order` (ref to Order), `stripeId`, `userId`; static method `build()` |

### Directory: `src/routes/`

| File | Description |
|------|-------------|
| `index.ts` | Re-exports all route handlers: `newPaymentRouter`, `showPaymentRouter`, `healthRouter` |
| `health.ts` | Health check route: `GET /api/payments/health` returns HTTP 200 with empty object |
| `new.ts` | Create payment route: `POST /api/payments` validates token/orderId, checks order ownership and status, creates Stripe charge, saves Payment, publishes `payment:complete` |
| `show.ts` | Show payments routes: `GET /api/payments` returns user's payments; `GET /api/payments/:id` returns single payment |

### Directory: `src/routes/__tests__/`

| File | Description |
|------|-------------|
| `new.test.ts` | Tests for POST /api/payments: auth checks, order validation, Stripe API calls, payment persistence, event publishing |
| `show.test.ts` | Tests for GET /api/payments and GET /api/payments/:id: auth checks, filtering by userId, 404 handling |

### Directory: `src/test/`

| File | Description |
|------|-------------|
| `setup.ts` | Jest global setup: creates MongoMemoryServer, mocks NATS and Stripe, clears collections between tests |
| `utils.ts` | Test utility: `getCookieHeader()` generates JWT-based cookie for authenticated test requests |

---

## Section 4: Dependency Analysis

### Production Dependencies

| Package | Version | Usage in This Codebase |
|---------|---------|------------------------|
| `@b.anik/ticketing-common` | ^0.1.2 | Shared library providing error classes (`NotFoundError`, `BadRequestError`, `NotAuthorizedError`), middlewares (`currentUserMiddleware`, `errorHandlerMiddleware`, `requireAuthMiddleware`, `validateRequestMiddleware`), event base classes (`Listener`, `Publisher`), event types (`OrderCreatedEvent`, `OrderCancelledEvent`, `OrderUpdatedEvent`, `PaymentAwaitedEvent`, `PaymentCompleteEvent`), and enums (`OrderStatus`, `Subjects`). Imported in `src/app.ts:6-10`, `src/routes/new.ts:2-10`, `src/routes/show.ts:1-5`, all event listeners and publishers, and `src/models/order.ts:1` |
| `body-parser` | ^1.20.2 | Used in `src/app.ts:3` - imports `json` function for JSON body parsing middleware |
| `cookie-session` | ^2.1.0 | Used in `src/app.ts:4` - provides session middleware with JWT stored in cookies |
| `express` | ^4.18.3 | Core web framework - used in `src/app.ts:1`, `src/routes/*.ts` for Router and request handling |
| `express-async-errors` | ^3.1.1 | Imported in `src/app.ts:2` - enables async/await error handling without try-catch in routes |
| `express-validator` | ^7.0.1 | Used in `src/routes/new.ts:2` and `src/routes/show.ts:6` for `body()` and `param()` validation |
| `jsonwebtoken` | ^9.0.2 | Used in `src/test/utils.ts:1` for generating test JWT tokens (production JWT verification is in common library) |
| `mongoose` | ^8.2.4 | MongoDB ODM - used in `src/index.ts:1`, `src/models/*.ts` for schema definition, `src/routes/show.ts:9` for `isValidObjectId` |
| `mongoose-update-if-current` | ^1.4.0 | Declared but no direct import found in payments service source code - likely used by common library for optimistic concurrency (the Order model implements manual version checking via `findByEvent`) |
| `node-nats-streaming` | ^0.3.2 | Used in `src/nats-wrapper.ts:1` - provides NATS Streaming client (`Stan` type) for event messaging |
| `stripe` | ^15.5.0 | Used in `src/stripe.ts:1` - Stripe SDK for payment processing, `charges.create()` called in `src/routes/new.ts:39` |

### Development Dependencies

| Package | Version | Usage in This Codebase |
|---------|---------|------------------------|
| `@repo/jest-config` | link:* | Base Jest configuration - imported in `jest.config.js:1` |
| `@types/body-parser` | ^1.19.5 | TypeScript types for body-parser |
| `@types/cookie-session` | ^2.0.48 | TypeScript types for cookie-session |
| `@types/express` | ^4.17.21 | TypeScript types for Express |
| `@types/jsonwebtoken` | ^9.0.6 | TypeScript types for jsonwebtoken |
| `@types/node` | ^20.11.30 | TypeScript types for Node.js |
| `@types/supertest` | ^6.0.2 | TypeScript types for supertest |
| `mongodb-memory-server` | ^9.1.8 | Used in `src/test/setup.ts:1` for in-memory MongoDB during tests |
| `nodemon` | ^3.1.0 | Development file watcher - referenced in `package.json:7` dev script |
| `supertest` | ^6.3.4 | Used in `src/routes/__tests__/*.test.ts` for HTTP integration tests |
| `ts-node` | ^10.9.2 | TypeScript execution - referenced in `package.json:9` start script |
| `typescript` | ^5.4.2 | TypeScript compiler - referenced in `package.json:13` typecheck script |

---

## Section 5: Internal Package Usage

### `@b.anik/ticketing-common` (version 0.1.2)

**Note**: This is an externally published npm package (not a workspace package) that serves as the shared common library for all services in the ticketing application.

| File | Imports Used | Functionality Provided |
|------|--------------|------------------------|
| `src/app.ts` | `NotFoundError`, `currentUserMiddleware`, `errorHandlerMiddleware` | Error class for 404 responses; middleware to extract current user from JWT in cookie; centralized error formatting middleware |
| `src/routes/new.ts` | `BadRequestError`, `NotAuthorizedError`, `NotFoundError`, `OrderStatus`, `requireAuthMiddleware`, `validateRequestMiddleware` | Error classes for validation/auth failures; enum for order statuses; auth requirement middleware; request validation middleware |
| `src/routes/show.ts` | `NotFoundError`, `requireAuthMiddleware`, `validateRequestMiddleware` | Error class and auth/validation middlewares |
| `src/models/order.ts` | `OrderStatus` | Enum defining order statuses: `Created`, `Cancelled`, `AwaitingPayment`, `Complete` |
| `src/events/listeners/order-created-listener.ts` | `Listener`, `OrderCreatedEvent`, `Subjects` | Base listener class for NATS subscription; typed event interface; subjects enum |
| `src/events/listeners/order-cancelled-listener.ts` | `Listener`, `OrderCancelledEvent`, `OrderStatus`, `Subjects` | Base listener class; typed event interface; order status enum; subjects enum |
| `src/events/listeners/order-updated-listener.ts` | `Listener`, `OrderUpdatedEvent`, `Subjects` | Base listener class; typed event interface; subjects enum |
| `src/events/publishers/payment-awaited-publisher.ts` | `PaymentAwaitedEvent`, `Publisher`, `Subjects` | Base publisher class for NATS events; typed event interface; subjects enum |
| `src/events/publishers/payment-complete-publisher.ts` | `PaymentCompleteEvent`, `Publisher`, `Subjects` | Base publisher class; typed event interface; subjects enum |

#### Specific Exports Used Summary

| Export | Type | Description |
|--------|------|-------------|
| `NotFoundError` | Class | Error thrown when a resource is not found (returns HTTP 404) |
| `BadRequestError` | Class | Error thrown for invalid requests (returns HTTP 400) |
| `NotAuthorizedError` | Class | Error thrown for unauthorized access (returns HTTP 401) |
| `currentUserMiddleware` | Middleware | Extracts and decodes JWT from session cookie, sets `req.currentUser` |
| `errorHandlerMiddleware` | Middleware | Catches all errors and formats them into standardized JSON response |
| `requireAuthMiddleware` | Middleware | Throws NotAuthorizedError if `req.currentUser` is not set |
| `validateRequestMiddleware` | Middleware | Checks express-validator results and throws BadRequestError on validation failures |
| `OrderStatus` | Enum | Values: `Created`, `Cancelled`, `AwaitingPayment`, `Complete` |
| `Subjects` | Enum | NATS event subject names: `OrderCreated`, `OrderCancelled`, `OrderUpdated`, `PaymentAwaited`, `PaymentComplete` |
| `Listener` | Abstract Class | Base class for NATS event listeners with typed event data |
| `Publisher` | Abstract Class | Base class for NATS event publishers with typed event data |
| `OrderCreatedEvent` | Interface | Event data shape for order creation events |
| `OrderCancelledEvent` | Interface | Event data shape for order cancellation events |
| `OrderUpdatedEvent` | Interface | Event data shape for order update events |
| `PaymentAwaitedEvent` | Interface | Event data shape for payment awaited events |
| `PaymentCompleteEvent` | Interface | Event data shape for payment complete events |

### `@repo/jest-config` (workspace link)

| File | Import | Functionality Provided |
|------|--------|------------------------|
| `jest.config.js` | Base config object via `require()` | Provides shared Jest configuration (transform rules, test environment settings) extended by this service |

---

## API Routes Summary

| Method | Endpoint | Auth Required | Description |
|--------|----------|---------------|-------------|
| GET | `/api/payments/health` | No | Health check for Kubernetes liveness probe |
| GET | `/api/payments` | Yes | List all payments for current user |
| GET | `/api/payments/:id` | Yes | Get single payment by ID (owned by current user) |
| POST | `/api/payments` | Yes | Create new payment for an order |

### POST `/api/payments` Request Body

```json
{
  "token": "stripe_payment_token",
  "orderId": "mongo_object_id"
}
```

---

## Events Architecture

### Events Consumed (Listeners)

| Event Subject | Handler | Action |
|---------------|---------|--------|
| `order:created` | `OrderCreatedListener` | Creates local Order copy, publishes `payment:awaited` |
| `order:cancelled` | `OrderCancelledListener` | Updates Order status to Cancelled |
| `order:updated` | `OrderUpdatedListener` | Syncs Order status and version |

### Events Published

| Event Subject | Publisher | Trigger |
|---------------|-----------|---------|
| `payment:awaited` | `PaymentAwaitedPublisher` | After receiving `order:created` event |
| `payment:complete` | `PaymentCompletePublisher` | After successful Stripe charge and Payment save |

---

## Data Models

### Order (local copy)

| Field | Type | Description |
|-------|------|-------------|
| `_id` | ObjectId | Matches the Order service's order ID |
| `version` | Number | Optimistic concurrency version |
| `price` | Number | Order total in dollars |
| `userId` | String | Owner's user ID |
| `status` | String (enum) | One of: Created, Cancelled, AwaitingPayment, Complete |

### Payment

| Field | Type | Description |
|-------|------|-------------|
| `_id` | ObjectId | Auto-generated |
| `order` | ObjectId (ref) | Reference to local Order document |
| `stripeId` | String | Stripe charge ID |
| `userId` | String | User who made the payment |
