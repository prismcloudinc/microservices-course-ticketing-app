---
service_id: ticketing-app/expiration
service_name: expiration
prompt_id: 01-service-identity
prompt_label: Service Identity & Anatomy
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.18121500000000001
schema_version: "2.0"
---

# Service Anatomy: expiration

## Section 1: Identity

| Attribute | Value |
|-----------|-------|
| **Package Name** | `expiration` (`services/expiration/package.json`, line 2) |
| **Version** | `1.0.0` (`services/expiration/package.json`, line 3) |
| **One-sentence Purpose** | Listens for order creation events and schedules expiration jobs to cancel orders after a timeout period |
| **Product/Platform** | Microservices Ticketing Application (`@b.anik/ticketing-common` scope, `README.md` line 205) |
| **Runtime** | Node.js 20 (`services/expiration/Dockerfile`, line 1: `FROM node:20-alpine`; confirmed by `.nvmrc`: `20`) |
| **Framework** | None — this is a worker process without HTTP server; uses `node-nats-streaming` for event consumption |
| **Database** | Redis (connection via `bull` library at `src/queues/expiration-queue.ts`, line 9-12; no ORM — Redis is used as job queue storage only) |

### Connection Details

| Infrastructure | Client Library | Connection Config |
|----------------|----------------|-------------------|
| NATS Streaming Server | `node-nats-streaming` | `process.env.NATS_URL`, `process.env.NATS_CLUSTER_ID`, `process.env.NATS_CLIENT_ID` (`src/index.ts`, lines 5-22) |
| Redis | `bull` | `process.env.REDIS_HOST` (`src/queues/expiration-queue.ts`, line 11) |

## Section 2: Entry Points

| File Path | Trigger | Description |
|-----------|---------|-------------|
| `src/index.ts` | Process start (via `pnpm run dev` or `pnpm start`) | Connects to NATS Streaming Server, registers signal handlers for graceful shutdown, and starts the `OrderCreatedListener` event listener |

### Event-Driven Entry Points

| Event Listener | Subject | Queue Group | Action |
|----------------|---------|-------------|--------|
| `OrderCreatedListener` (`src/events/listeners/order-created-listener.ts`) | `Subjects.OrderCreated` (from common lib) | `expiration-service` | Receives order creation events and schedules a delayed job in the Bull queue based on `expiresAt` field |

### Job Processor Entry Points

| Queue | Processor | Action |
|-------|-----------|--------|
| `order:expiration` (`src/queues/expiration-queue.ts`, line 9) | Bull queue processor (`src/queues/expiration-queue.ts`, lines 15-18) | When a delayed job fires, publishes an `ExpirationComplete` event with the `orderId` |

## Section 3: Directory Map

### `services/expiration/`

Root directory of the expiration microservice.

| File | Description |
|------|-------------|
| `.dockerignore` | Excludes `node_modules`, `dist`, `build`, and `.turbo` directories from Docker build context |
| `Dockerfile` | Builds Node.js 20 Alpine image, installs pnpm 9, copies package files, installs dependencies, and runs `pnpm run dev` |
| `jest.config.js` | Extends shared `@repo/jest-config` and configures `src/test/setup.ts` as the test setup file |
| `package.json` | Package manifest defining name `expiration`, scripts (`dev`, `start`, `test`, etc.), and dependencies |
| `pnpm-lock.yaml` | Lockfile for reproducible dependency installation |
| `tsconfig.json` | TypeScript configuration: targets ES2016, CommonJS modules, strict mode, rootDir `./src`, noEmit true |

### `services/expiration/src/`

Main source directory.

| File | Description |
|------|-------------|
| `index.ts` | Application entry point: validates environment variables (`NATS_URL`, `NATS_CLUSTER_ID`), connects to NATS via `natsWrapper`, registers signal handlers (`SIGINT`, `SIGTERM`, `SIGUSR2`) for graceful shutdown, and starts event listeners |
| `nats-wrapper.ts` | Singleton class `NatsWrapper` that wraps `node-nats-streaming` client; provides `connect()` method and `client` getter |

### `services/expiration/src/__mocks__/`

Jest mock files for unit testing.

| File | Description |
|------|-------------|
| `nats-wrapper.ts` | Mock implementation of `natsWrapper` with stubbed `connect()`, `client.publish()`, `client.on()`, and `client.close()` methods for testing without a real NATS connection |

### `services/expiration/src/events/`

Event handling layer (listeners and publishers).

| File | Description |
|------|-------------|
| `index.ts` | Re-exports all exports from `./listeners` and `./publishers` subdirectories |

### `services/expiration/src/events/listeners/`

NATS event listeners.

| File | Description |
|------|-------------|
| `index.ts` | Exports `startListeners()` function that instantiates and starts `OrderCreatedListener` |
| `order-created-listener.ts` | Extends `Listener<OrderCreatedEvent>` from common library; subscribes to `Subjects.OrderCreated` with queue group `expiration-service`; on message, calculates delay from `expiresAt`, adds job to `expirationQueue` with `orderId` payload |
| `queue-group-name.ts` | Exports constant `queueGroupName = 'expiration-service'` (currently unused — listener hardcodes the value) |

### `services/expiration/src/events/listeners/__tests__/`

Unit tests for event listeners.

| File | Description |
|------|-------------|
| `order-created-listener.test.ts` | Tests `OrderCreatedListener`: verifies job is added to expiration queue with correct `orderId` and positive delay; verifies promise resolves; verifies no job is added when `expiresAt` is undefined |

### `services/expiration/src/events/publishers/`

NATS event publishers.

| File | Description |
|------|-------------|
| `index.ts` | Re-exports all exports from `./expiration-complete-publisher.ts` |
| `expiration-complete-publisher.ts` | Extends `Publisher<ExpirationCompleteEvent>` from common library; publishes to `Subjects.ExpirationComplete` |

### `services/expiration/src/queues/`

Bull queue definitions and processors.

| File | Description |
|------|-------------|
| `index.ts` | Re-exports all exports from `./expiration-queue.ts` |
| `expiration-queue.ts` | Creates Bull queue `order:expiration` connected to Redis at `REDIS_HOST`; defines job payload interface `{ orderId: string }`; processes jobs by publishing `ExpirationComplete` event via `ExpirationCompletePublisher` |

### `services/expiration/src/queues/__mocks__/`

Jest mocks for queue testing.

| File | Description |
|------|-------------|
| `index.ts` | Re-exports from `./expiration-queue.ts` mock |
| `expiration-queue.ts` | Mock `expirationQueue` object with `add` function as a Jest mock that resolves to undefined |

### `services/expiration/src/test/`

Test setup and utilities.

| File | Description |
|------|-------------|
| `setup.ts` | Jest setup: mocks `../nats-wrapper` and `../queues` modules; clears all mocks before each test |

## Section 4: Dependency Analysis

### Production Dependencies

| Package | Version | Usage in Codebase |
|---------|---------|-------------------|
| `@b.anik/ticketing-common` | `^0.1.2` | Shared ticketing library providing base classes and types for event-driven architecture. **Imports found:** `Listener`, `OrderCreatedEvent`, `Subjects` (`src/events/listeners/order-created-listener.ts`, line 1); `ExpirationCompleteEvent`, `Publisher`, `Subjects` (`src/events/publishers/expiration-complete-publisher.ts`, line 1); `OrderCreatedEvent`, `OrderStatus` (`src/events/listeners/__tests__/order-created-listener.test.ts`, line 1) |
| `bull` | `^4.12.2` | Redis-backed job queue library. **Import found:** `Queue` class imported and instantiated as `expirationQueue` with name `order:expiration` (`src/queues/expiration-queue.ts`, line 1, 9). Used for scheduling delayed expiration jobs. |
| `node-nats-streaming` | `^0.3.2` | NATS Streaming client for publish-subscribe messaging. **Import found:** `nats`, `Stan` type imported (`src/nats-wrapper.ts`, line 1). Used to connect to NATS cluster and provide the client instance. `Message` type imported (`src/events/listeners/order-created-listener.ts`, line 2). |

### Development Dependencies

| Package | Version | Usage in Codebase |
|---------|---------|-------------------|
| `@repo/jest-config` | `link:*` | Internal monorepo package providing shared Jest configuration. **Import found:** `jest.config.js` line 1: `require('@repo/jest-config')` |
| `@types/node` | `^20.11.30` | TypeScript type definitions for Node.js runtime APIs. Declared but implicitly used by TypeScript compilation. |
| `nodemon` | `^3.1.0` | Development server with auto-restart on file changes. Used in `dev` script (`package.json`, line 7): `nodemon src/index.ts` |
| `ts-node` | `^10.9.2` | TypeScript execution engine for Node.js. Used in `start` script (`package.json`, line 9): `ts-node src/index.ts`; also used by nodemon for `dev` |
| `typescript` | `^5.4.2` | TypeScript compiler. Used in `typecheck` script (`package.json`, line 13): `tsc --noEmit` |

## Section 5: Internal Package Usage

### `@b.anik/ticketing-common` (org-scoped npm package, not local)

This is an external npm package (not a monorepo workspace package) that provides shared infrastructure for the ticketing microservices. Based on imports found in this service:

| Import | File(s) | Purpose |
|--------|---------|---------|
| `Listener` | `src/events/listeners/order-created-listener.ts` | Base class for NATS event listeners; `OrderCreatedListener` extends it |
| `Publisher` | `src/events/publishers/expiration-complete-publisher.ts` | Base class for NATS event publishers; `ExpirationCompletePublisher` extends it |
| `Subjects` | `src/events/listeners/order-created-listener.ts`, `src/events/publishers/expiration-complete-publisher.ts` | Enum containing NATS subject names (`OrderCreated`, `ExpirationComplete`, etc.) |
| `OrderCreatedEvent` | `src/events/listeners/order-created-listener.ts`, `src/events/listeners/__tests__/order-created-listener.test.ts` | TypeScript interface defining the shape of order creation event data (`id`, `version`, `status`, `ticket`, `userId`, `expiresAt`) |
| `ExpirationCompleteEvent` | `src/events/publishers/expiration-complete-publisher.ts` | TypeScript interface defining the shape of expiration complete event data (`orderId`) |
| `OrderStatus` | `src/events/listeners/__tests__/order-created-listener.test.ts` | Enum for order status values (e.g., `OrderStatus.Created`) used in test fixtures |

### `@repo/jest-config` (monorepo internal package)

| Import | File | Purpose |
|--------|------|---------|
| Default export (config object) | `jest.config.js` | Provides shared Jest configuration: `preset: 'ts-jest'`, `testEnvironment: 'node'`, `verbose: true`, `modulePaths: ['<rootDir>/src']`, `watchPathIgnorePatterns: ['globalConfig']` (from `packages/jest-config/jest.config.js`) |

## Appendix: Kubernetes Deployment

The service is deployed via Kubernetes manifests in `infra/k8s/`:

### `infra/k8s/expiration-depl.yaml`

- Deployment name: `expiration-depl`
- Replicas: 1
- Container image: `berkinanik/ticketing-expiration:latest`
- Environment variables:
  - `NATS_URL`: `http://nats-srv:4222`
  - `NATS_CLUSTER_ID`: `ticketing`
  - `NATS_CLIENT_ID`: Derived from pod metadata name
  - `REDIS_HOST`: `expiration-redis-srv`

### `infra/k8s/expiration-redis-depl.yaml`

- Deployment name: `expiration-redis-depl`
- Container image: `redis`
- Service name: `expiration-redis-srv`
- Port: 6379

## Appendix: Event Flow Diagram

```
┌─────────────────┐                    ┌─────────────────────┐                    ┌─────────────┐
│  Orders Service │  order:created     │  Expiration Service │  expiration:complete│  Orders     │
│                 │ ────────────────►  │                     │ ────────────────►  │  Service    │
│                 │  (with expiresAt)  │                     │  (with orderId)    │             │
└─────────────────┘                    └─────────────────────┘                    └─────────────┘
                                                │
                                                │ adds job with delay
                                                ▼
                                       ┌─────────────────┐
                                       │   Redis Queue   │
                                       │ order:expiration│
                                       └─────────────────┘
                                                │
                                                │ processes after delay
                                                ▼
                                       publishes ExpirationComplete
```
