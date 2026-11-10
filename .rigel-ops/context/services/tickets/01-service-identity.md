---
service_id: ticketing-app/tickets
service_name: tickets
prompt_id: 01-service-identity
prompt_label: Service Identity & Anatomy
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.25989999999999996
schema_version: "2.0"
---

# Service Anatomy: Tickets Service

## Section 1: Identity

| Property | Value |
|----------|-------|
| **Package Name** | `tickets` (`services/tickets/package.json:2`) |
| **Version** | `1.0.0` (`services/tickets/package.json:3`) |
| **Purpose** | Handles ticket creation, management, and availability for a ticketing application microservices architecture |
| **Product/Platform** | Part of the Ticketing Application monorepo (`microservices-course-ticketing-app`) — an event-driven microservices system for buying and selling tickets |
| **Runtime** | Node.js 20 (pinned in `.nvmrc:1` and `services/tickets/Dockerfile:1` as `node:20-alpine`) |
| **Framework** | Express.js v4.18.3 (`services/tickets/package.json:23`) |
| **Database** | MongoDB via Mongoose v8.2.4 (`services/tickets/package.json:26`); connection method: `mongoose.connect()` with connection string from `MONGO_URI` environment variable (`services/tickets/src/index.ts:47`); uses `mongoose-update-if-current` plugin v1.4.0 for optimistic concurrency control (`services/tickets/src/models/ticket.ts:56`) |

### Environment Variables Required

| Variable | Purpose | Source |
|----------|---------|--------|
| `JWT_KEY` | Secret key for JWT token verification | Kubernetes secret (`infra/k8s/tickets-depl.yaml:19-23`) |
| `MONGO_URI` | MongoDB connection string | Kubernetes env (`infra/k8s/tickets-depl.yaml:24-25`), default: `mongodb://tickets-mongo-srv:27017/tickets` |
| `NATS_URL` | NATS Streaming Server URL | Kubernetes env (`infra/k8s/tickets-depl.yaml:26-27`), default: `http://nats-srv:4222` |
| `NATS_CLUSTER_ID` | NATS cluster identifier | Kubernetes env (`infra/k8s/tickets-depl.yaml:28-29`), default: `ticketing` |
| `NATS_CLIENT_ID` | Unique NATS client identifier | Kubernetes fieldRef to `metadata.name` (`infra/k8s/tickets-depl.yaml:30-32`) |

---

## Section 2: Entry Points

| File Path | Trigger | Description |
|-----------|---------|-------------|
| `services/tickets/src/index.ts` | Process start (`pnpm run dev` or `pnpm run start`) | Main application entry point — connects to NATS Streaming, starts event listeners, connects to MongoDB, and starts Express HTTP server on port 3000 |
| `services/tickets/src/routes/health.ts` | HTTP GET `/api/tickets/health` | Kubernetes liveness probe endpoint — returns 200 OK for health checks |
| `services/tickets/src/routes/new.ts` | HTTP POST `/api/tickets` | Creates a new ticket (requires authentication) — validates title/price, saves to MongoDB, publishes `ticket:created` event |
| `services/tickets/src/routes/show.ts` | HTTP GET `/api/tickets` | Lists all tickets in the system |
| `services/tickets/src/routes/show.ts` | HTTP GET `/api/tickets/:id` | Retrieves a single ticket by ID |
| `services/tickets/src/routes/update.ts` | HTTP PUT `/api/tickets/:id` | Updates an existing ticket (requires authentication and ownership) — validates inputs, prevents editing reserved tickets, publishes `ticket:updated` event |
| `services/tickets/src/routes/delete.ts` | HTTP DELETE `/api/tickets/:id` | Deletes a ticket (requires authentication and ownership) — prevents deleting reserved tickets |
| `services/tickets/src/events/listeners/order-created-listener.ts` | NATS event `order:created` | Marks a ticket as reserved by setting `orderId` when an order is created for that ticket, then publishes `ticket:updated` event |
| `services/tickets/src/events/listeners/order-cancelled-listener.ts` | NATS event `order:cancelled` | Releases a ticket reservation by clearing `orderId` when an order is cancelled, then publishes `ticket:updated` event |

---

## Section 3: Directory Map

### `services/tickets/` (root)

| File | Description |
|------|-------------|
| `.dockerignore` | Specifies files to exclude from Docker build context (`node_modules`, `dist`, `build`, `.turbo`) |
| `Dockerfile` | Multi-stage Docker build configuration using `node:20-alpine` base image, installs pnpm@9, copies dependencies and runs in dev mode |
| `jest.config.js` | Jest test configuration — extends `@repo/jest-config` base config, sets up test environment via `src/test/setup.ts` |
| `package.json` | Package manifest defining dependencies, scripts (`dev`, `start`, `test`, `typecheck`), and metadata |
| `pnpm-lock.yaml` | Lockfile for deterministic dependency installation |
| `tsconfig.json` | TypeScript configuration targeting ES2016, CommonJS modules, strict mode enabled |

### `services/tickets/src/`

| File | Description |
|------|-------------|
| `app.ts` | Express application configuration — sets up middleware (body-parser, cookie-session, currentUser), registers route handlers, configures 404 handler and error middleware |
| `index.ts` | Application bootstrap — validates environment variables, connects to NATS Streaming and MongoDB, starts event listeners, launches HTTP server on port 3000 |
| `nats-wrapper.ts` | Singleton wrapper class for NATS Streaming client — provides `connect()` method and exposes the client instance |

### `services/tickets/src/models/`

| File | Description |
|------|-------------|
| `index.ts` | Barrel export for models — re-exports from `./ticket` |
| `ticket.ts` | Mongoose model for Ticket documents — defines schema with `title` (string), `price` (number), `userId` (string), `orderId` (optional string), `version` (number); implements optimistic concurrency control via `mongoose-update-if-current` plugin; includes static `build()` factory method for TypeScript type safety |

### `services/tickets/src/models/__tests__/`

| File | Description |
|------|-------------|
| `ticket.test.ts` | Unit tests for Ticket model — tests optimistic concurrency control behavior and version incrementing on saves |

### `services/tickets/src/routes/`

| File | Description |
|------|-------------|
| `index.ts` | Barrel export for all route handlers — re-exports `newTicketRouter`, `showTicketRouter`, `updateTicketRouter`, `deleteTicketRouter`, `healthRouter` |
| `health.ts` | Health check endpoint (`GET /api/tickets/health`) — returns empty 200 response for Kubernetes liveness probes |
| `new.ts` | Create ticket endpoint (`POST /api/tickets`) — requires auth, validates title (non-empty) and price (> 0), creates Ticket document, publishes `ticket:created` event via NATS |
| `show.ts` | Read ticket endpoints — `GET /api/tickets/:id` returns single ticket or 404, `GET /api/tickets` returns all tickets |
| `update.ts` | Update ticket endpoint (`PUT /api/tickets/:id`) — requires auth and ownership, validates title/price, prevents editing reserved tickets (has `orderId`), publishes `ticket:updated` event |
| `delete.ts` | Delete ticket endpoint (`DELETE /api/tickets/:id`) — requires auth and ownership, prevents deleting reserved tickets (has `orderId`), removes ticket from database |

### `services/tickets/src/routes/__tests__/`

| File | Description |
|------|-------------|
| `new.test.ts` | Integration tests for POST `/api/tickets` — tests authentication requirement, validation errors, successful creation, and event publishing |
| `show.test.ts` | Integration tests for GET endpoints — tests 404 for missing tickets, successful retrieval of single ticket and all tickets |
| `update.test.ts` | Integration tests for PUT `/api/tickets/:id` — tests authentication, authorization, validation, reserved ticket protection, successful update, and event publishing |
| `delete.test.ts` | Integration tests for DELETE `/api/tickets/:id` — tests authentication, authorization, reserved ticket protection, and successful deletion |

### `services/tickets/src/events/`

| File | Description |
|------|-------------|
| `index.ts` | Barrel export for events — re-exports publishers and listeners, exports `startListeners()` function |

### `services/tickets/src/events/publishers/`

| File | Description |
|------|-------------|
| `index.ts` | Barrel export for publishers — re-exports `TicketCreatedPublisher` and `TicketUpdatedPublisher` |
| `ticket-created-publisher.ts` | Publisher for `ticket:created` events — extends base `Publisher` class from common module, publishes when a new ticket is created |
| `ticket-updated-publisher.ts` | Publisher for `ticket:updated` events — extends base `Publisher` class, publishes when a ticket is modified or reservation status changes |

### `services/tickets/src/events/listeners/`

| File | Description |
|------|-------------|
| `index.ts` | Listener initialization — exports `startListeners()` function that instantiates and starts `OrderCreatedListener` and `OrderCancelledListener` |
| `queue-group-name.ts` | Defines queue group name constant (`tickets-service`) for NATS message distribution |
| `order-created-listener.ts` | Listener for `order:created` events — marks ticket as reserved by setting `orderId`, publishes `ticket:updated` event to notify other services |
| `order-cancelled-listener.ts` | Listener for `order:cancelled` events — clears `orderId` to release ticket reservation, publishes `ticket:updated` event |

### `services/tickets/src/events/listeners/__tests__/`

| File | Description |
|------|-------------|
| `order-created-listener.test.ts` | Unit tests for OrderCreatedListener — tests setting `orderId` on ticket, error handling for missing tickets, and publishing of `ticket:updated` event |
| `order-cancelled-listener.test.ts` | Unit tests for OrderCancelledListener — tests clearing `orderId`, validation of ticket/order matching, and publishing of `ticket:updated` event |

### `services/tickets/src/test/`

| File | Description |
|------|-------------|
| `setup.ts` | Jest test environment setup — creates MongoDB memory server, sets `JWT_KEY` env var, mocks NATS wrapper, handles test isolation by clearing collections between tests |
| `utils.ts` | Test utility functions — exports `getCookieHeader()` function that generates valid JWT session cookies for authenticated test requests |

### `services/tickets/src/__mocks__/`

| File | Description |
|------|-------------|
| `nats-wrapper.ts` | Jest mock for NATS wrapper — provides mocked `connect()`, `client.publish()`, `client.on()`, and `client.close()` methods for testing without real NATS connection |

---

## Section 4: Dependency Analysis

### Production Dependencies

| Package | Version | Usage in This Codebase |
|---------|---------|------------------------|
| `@b.anik/ticketing-common` | ^0.1.2 | Shared library providing: `NotFoundError`, `NotAuthorizedError`, `BadRequestError` (error classes used in route handlers), `currentUserMiddleware` (extracts user from JWT), `requireAuthMiddleware` (auth enforcement), `validateRequestMiddleware` (validation error handling), `errorHandlerMiddleware` (global error handler), `Publisher`/`Listener` (NATS event base classes), `Subjects` (event type constants), `TicketCreatedEvent`/`TicketUpdatedEvent`/`OrderCreatedEvent`/`OrderCancelledEvent`/`OrderStatus` (TypeScript event interfaces). Imported in: `src/app.ts:6-10`, `src/routes/new.ts:4`, `src/routes/show.ts:3`, `src/routes/update.ts:1-7`, `src/routes/delete.ts:1-6`, `src/events/publishers/*.ts`, `src/events/listeners/*.ts` |
| `body-parser` | ^1.20.2 | Parses JSON request bodies via `json()` middleware. Imported in `src/app.ts:3`, used at `src/app.ts:25` |
| `cookie-session` | ^2.1.0 | Session management middleware storing JWT in cookies. Imported in `src/app.ts:4`, configured at `src/app.ts:26-30` with `signed: false` and `secure: true` (except in test) |
| `express` | ^4.18.3 | Web framework — creates application instance, defines routers. Imported in `src/app.ts:1`, `src/routes/*.ts` |
| `express-async-errors` | ^3.1.1 | Patches Express to handle async errors without explicit try/catch. Imported via side-effect in `src/app.ts:2` |
| `express-validator` | ^7.0.1 | Request validation middleware. Imported in `src/routes/new.ts:2`, `src/routes/update.ts:9`; uses `body()` for validating `title` and `price` fields |
| `jsonwebtoken` | ^9.0.2 | JWT signing/verification for authentication. Used indirectly through `@b.anik/ticketing-common` middleware; directly imported only in `src/test/utils.ts:1` for generating test cookies |
| `mongoose` | ^8.2.4 | MongoDB ODM. Imported in `src/index.ts:1` for connection, `src/models/ticket.ts:1` for schema/model definition, `src/routes/show.ts:4`, `src/routes/update.ts:10`, `src/routes/delete.ts:8` for `isValidObjectId()` validation |
| `mongoose-update-if-current` | ^1.4.0 | Optimistic concurrency control plugin for Mongoose. Imported in `src/models/ticket.ts:2`, applied at `src/models/ticket.ts:56` via `ticketSchema.plugin(updateIfCurrentPlugin)` |
| `node-nats-streaming` | ^0.3.2 | NATS Streaming client for pub/sub messaging. Imported in `src/nats-wrapper.ts:1` for connection, `src/events/listeners/index.ts:1` for `Stan` type, `src/events/listeners/*.ts:2` for `Message` type |

### Development Dependencies

| Package | Version | Usage in This Codebase |
|---------|---------|------------------------|
| `@repo/jest-config` | link:* | Internal workspace package providing shared Jest configuration base. Imported in `jest.config.js:1` |
| `@types/body-parser` | ^1.19.5 | TypeScript type definitions for body-parser |
| `@types/cookie-session` | ^2.0.48 | TypeScript type definitions for cookie-session |
| `@types/express` | ^4.17.21 | TypeScript type definitions for Express |
| `@types/jsonwebtoken` | ^9.0.6 | TypeScript type definitions for jsonwebtoken |
| `@types/node` | ^20.11.30 | TypeScript type definitions for Node.js runtime |
| `@types/supertest` | ^6.0.2 | TypeScript type definitions for supertest |
| `mongodb-memory-server` | ^9.1.8 | In-memory MongoDB server for testing. Imported in `src/test/setup.ts:1`, creates isolated DB per test run |
| `nodemon` | ^3.1.0 | Development file watcher for auto-restart. Used in `dev` script (`package.json:7`) |
| `supertest` | ^6.3.4 | HTTP assertion library for testing Express routes. Imported in all route test files (`src/routes/__tests__/*.test.ts`) |
| `ts-node` | ^10.9.2 | TypeScript execution engine. Used in `start` and `dev` scripts (`package.json:8-9`) |
| `typescript` | ^5.4.2 | TypeScript compiler. Used for type checking via `typecheck` script (`package.json:14`) |

---

## Section 5: Internal Package Usage

### `@b.anik/ticketing-common` (^0.1.2)

**Note**: This is an external npm package, not a local monorepo package. It is published to npm under the `@b.anik` scope. The source is maintained in a separate repository (`github.com/berkinanik/microservices-course-common-module`) and referenced as a git submodule in `packages/common/` (per `.gitmodules`), but the submodule is not checked out in this repository state.

| Import Location | Exports Used | Functionality Provided |
|-----------------|--------------|------------------------|
| `src/app.ts:6-10` | `NotFoundError`, `currentUserMiddleware`, `errorHandlerMiddleware` | `NotFoundError` — thrown for 404 responses; `currentUserMiddleware` — extracts JWT payload from session cookie to `req.currentUser`; `errorHandlerMiddleware` — global error handler converting errors to standardized JSON response |
| `src/routes/new.ts:4` | `requireAuthMiddleware`, `validateRequestMiddleware` | `requireAuthMiddleware` — throws 401 if no `currentUser`; `validateRequestMiddleware` — checks express-validator results and throws `RequestValidationError` if any |
| `src/routes/show.ts:3` | `NotFoundError` | Error class for 404 responses when ticket not found |
| `src/routes/update.ts:1-7` | `BadRequestError`, `NotAuthorizedError`, `NotFoundError`, `requireAuthMiddleware`, `validateRequestMiddleware` | `BadRequestError` — 400 response for reserved ticket edit attempt; `NotAuthorizedError` — 401 for non-owner edit attempt |
| `src/routes/delete.ts:1-6` | `BadRequestError`, `NotAuthorizedError`, `NotFoundError`, `requireAuthMiddleware` | Same error classes for delete operation authorization checks |
| `src/events/publishers/ticket-created-publisher.ts:1` | `Publisher`, `Subjects`, `TicketCreatedEvent` | `Publisher` — abstract base class for NATS event publishing with retry logic; `Subjects` — enum with event type strings (`ticket:created`); `TicketCreatedEvent` — TypeScript interface for event payload |
| `src/events/publishers/ticket-updated-publisher.ts:1` | `Publisher`, `Subjects`, `TicketUpdatedEvent` | Same as above for `ticket:updated` events |
| `src/events/listeners/order-created-listener.ts:1` | `Listener`, `OrderCreatedEvent`, `Subjects` | `Listener` — abstract base class for NATS event subscription with queue groups and ack handling; `OrderCreatedEvent` — TypeScript interface for incoming order creation events |
| `src/events/listeners/order-cancelled-listener.ts:1` | `Listener`, `OrderCancelledEvent`, `Subjects` | Same as above for `order:cancelled` events |
| `src/events/listeners/__tests__/order-created-listener.test.ts:1` | `OrderCreatedEvent`, `OrderStatus` | Event interface and status enum for constructing test data |
| `src/events/listeners/__tests__/order-cancelled-listener.test.ts:1` | `OrderCancelledEvent` | Event interface for constructing test data |

### `@repo/jest-config` (link:*)

| Import Location | Exports Used | Functionality Provided |
|-----------------|--------------|------------------------|
| `jest.config.js:1` | Default export (base config object) | Shared Jest configuration including `ts-jest` preset, `node` test environment, verbose output, and module path configuration. Located at `packages/jest-config/jest.config.js` |

---

## API Endpoints Summary

| Method | Path | Auth Required | Description |
|--------|------|---------------|-------------|
| GET | `/api/tickets/health` | No | Kubernetes liveness probe |
| GET | `/api/tickets` | No | List all tickets |
| GET | `/api/tickets/:id` | No | Get single ticket by ID |
| POST | `/api/tickets` | Yes | Create new ticket |
| PUT | `/api/tickets/:id` | Yes (owner) | Update existing ticket (not reserved) |
| DELETE | `/api/tickets/:id` | Yes (owner) | Delete ticket (not reserved) |

---

## Event Integration

### Published Events

| Event Subject | Publisher Class | Trigger | Payload Fields |
|---------------|-----------------|---------|----------------|
| `ticket:created` | `TicketCreatedPublisher` | POST `/api/tickets` success | `id`, `version`, `title`, `price`, `userId` |
| `ticket:updated` | `TicketUpdatedPublisher` | PUT `/api/tickets/:id` success, or order created/cancelled on ticket | `id`, `version`, `title`, `price`, `userId`, `orderId` (optional) |

### Consumed Events

| Event Subject | Listener Class | Queue Group | Action |
|---------------|----------------|-------------|--------|
| `order:created` | `OrderCreatedListener` | `tickets-service` | Sets `orderId` on referenced ticket to mark as reserved |
| `order:cancelled` | `OrderCancelledListener` | `tickets-service` | Clears `orderId` on referenced ticket to release reservation |

---

## Data Model

### Ticket

```typescript
interface TicketDoc {
  title: string;      // Required, validated as non-empty
  price: number;      // Required, validated as > 0
  userId: string;     // Owner's user ID from JWT
  orderId?: string;   // Set when ticket is reserved by an order
  version: number;    // Managed by mongoose-update-if-current for OCC
}
```

Schema defined in `services/tickets/src/models/ticket.ts:22-48`

---

## Infrastructure

| Component | Kubernetes Resource | Image | Port |
|-----------|---------------------|-------|------|
| Tickets Service | Deployment `tickets-depl` | `berkinanik/ticketing-tickets:latest` | 3000 |
| Tickets Service | Service `tickets-srv` | — | 3000 |
| Tickets MongoDB | Deployment `tickets-mongo-depl` | `mongo` | 27017 |
| Tickets MongoDB | Service `tickets-mongo-srv` | — | 27017 |

Ingress routing: `/api/tickets/?(.*)` → `tickets-srv:3000` (`infra/k8s/ingress-srv.yaml:20-26`)
