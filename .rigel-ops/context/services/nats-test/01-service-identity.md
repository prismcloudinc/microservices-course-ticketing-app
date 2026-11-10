---
service_id: ticketing-app/nats-test
service_name: nats-test
prompt_id: 01-service-identity
prompt_label: Service Identity & Anatomy
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.11178
schema_version: "2.0"
---

# Service Anatomy: nats-test

## Section 1: Identity

| Attribute | Value |
|-----------|-------|
| **Package Name** | `nats-test` (`nats-test/package.json:2`) |
| **One-Sentence Purpose** | A standalone demo/test package demonstrating NATS Streaming pub/sub patterns with TypeScript base classes for listeners and publishers |
| **Product/Platform** | Part of the Ticketing Application microservices monorepo (course project from "Microservices with Node.js and React") |
| **Runtime** | Node.js 20 (per `.nvmrc:1`) |
| **Framework** | None - plain TypeScript with `node-nats-streaming` library |
| **Database** | None - this package does not use a database |

## Section 2: Entry Points

| File Path | Trigger | Description |
|-----------|---------|-------------|
| `nats-test/src/publisher.ts` | CLI invocation via `pnpm run publish` | Connects to NATS Streaming server and publishes a sample `ticket:created` event with hardcoded test data |
| `nats-test/src/listener.ts` | CLI invocation via `pnpm run listen` | Connects to NATS Streaming server and subscribes to `ticket:created` and `ticket:updated` events, logging received messages |

## Section 3: Directory Map

### `nats-test/` (root)

| File | Purpose |
|------|---------|
| `README.md` | Documentation with commands to run publisher/listener and Kubernetes port-forwarding instructions for NATS pod |
| `package.json` | Package manifest defining name `nats-test`, npm scripts (`publish`, `listen`), and dependencies |
| `tsconfig.json` | TypeScript configuration targeting ES2016, CommonJS modules, strict mode, with no emit (relies on `node-ts` for runtime compilation) |

### `nats-test/src/`

| File | Purpose |
|------|---------|
| `publisher.ts` | Entry point that connects to NATS Streaming cluster `ticketing` at `localhost:4222`, creates a `TicketCreatedPublisher`, and publishes a sample event (`id: '123', title: 'concert', price: 20`). Handles graceful shutdown on SIGINT/SIGTERM/SIGUSR2 signals. |
| `listener.ts` | Entry point that connects to NATS Streaming with a random client ID (via `crypto.randomBytes`), instantiates `TicketCreatedListener` and `TicketUpdatedListener`, and calls `listen()` on both. Handles graceful shutdown on SIGINT/SIGTERM/SIGUSR2 signals. |

### `nats-test/src/events/`

| File | Purpose |
|------|---------|
| `subjects.ts` | Defines an enum `Subjects` with event channel names: `TicketCreated = 'ticket:created'`, `TicketUpdated = 'ticket:updated'`, `OrderUpdated = 'order:updated'` |
| `base-listener.ts` | Abstract generic class `Listener<T extends Event>` providing NATS Streaming subscription logic: subscription options (deliver all available, manual ack, ack wait timeout, durable subscription), message parsing (JSON from string or Buffer), and automatic acknowledgment on successful `onMessage` completion |
| `base-publisher.ts` | Abstract generic class `Publisher<T extends Event>` providing promisified NATS Streaming publish logic with JSON serialization and console logging |
| `ticket-created-event.ts` | TypeScript interface `TicketCreatedEvent` defining the event shape: `subject: Subjects.TicketCreated`, `data: { id: string, title: string, price: number }` |
| `ticket-created-listener.ts` | Concrete listener extending `Listener<TicketCreatedEvent>`, subscribes to `ticket:created` with queue group `payments-service`, logs event data on message receipt |
| `ticket-created-publisher.ts` | Concrete publisher extending `Publisher<TicketCreatedEvent>`, publishes to `ticket:created` subject |
| `ticket-updated-event.ts` | TypeScript interface `TicketUpdatedEvent` defining the event shape: `subject: Subjects.TicketUpdated`, `data: { id: string, title: string, price: number }` |
| `ticket-updated-listener.ts` | Concrete listener extending `Listener<TicketUpdatedEvent>`, subscribes to `ticket:updated` with queue group `payments-service`, logs event data on message receipt |

## Section 4: Dependency Analysis

### Production Dependencies

| Package | Version | Usage in This Codebase |
|---------|---------|------------------------|
| `node-nats-streaming` | `^0.3.2` (resolved: `0.3.2` per `pnpm-lock.yaml:100-101`) | Imported in `src/publisher.ts:1`, `src/listener.ts:1`, `src/events/base-listener.ts:1`, and `src/events/base-publisher.ts:1`. Used to create NATS Streaming client connections via `nats.connect()`, subscribe to channels, and publish messages. Types `Stan` and `Message` are used in base classes. |

### Development Dependencies

| Package | Version | Usage in This Codebase |
|---------|---------|------------------------|
| `@types/node` | `^20.12.7` (resolved: `20.12.8` per `pnpm-lock.yaml:104-105`) | Provides TypeScript type definitions for Node.js built-ins. Used implicitly for `process` object (SIGINT/SIGTERM handlers) and `crypto` module (imported in `src/listener.ts:2`). |
| `node-ts` | `^6.0.1` (resolved: `6.0.1` per `pnpm-lock.yaml:107-108`) | Runtime TypeScript execution. Referenced by nodemon in `package.json:7-8` scripts to run `.ts` files directly without explicit compilation step. |
| `nodemon` | `^3.1.0` (resolved: `3.1.0` per `pnpm-lock.yaml:109-110`) | Used in npm scripts (`package.json:7-8`) to watch TypeScript source files and re-execute on changes for development workflow. |
| `typescript` | `^5.4.5` (resolved: `5.4.5` per `pnpm-lock.yaml:113-114`) | TypeScript compiler. Used for type-checking (via tsconfig with `noEmit: true`). Provides language support for generics, interfaces, and abstract classes used throughout the events module. |

## Section 5: Internal Package Usage

This package has **no internal/org-scoped dependencies**. It does not import from:
- `@b.anik/ticketing-common` (the shared common module used by other services)
- Any other workspace packages (services, apps, or packages)

The package is designed as a **standalone demonstration** of NATS Streaming patterns. While it is part of the pnpm workspace (per `pnpm-workspace.yaml:5`), it is **not imported by any other package** in the monorepo (verified via grep for import/require statements).

### Relationship to Production Services

The base classes (`Listener`, `Publisher`) and event interfaces in this package appear to be **prototypes/references** for the similar patterns used in the actual microservices. The production services (`services/tickets`, `services/orders`, `services/payments`, `services/expiration`) use the same `node-nats-streaming` library but implement their own wrapper classes, likely via the shared `@b.anik/ticketing-common` package (inferred from project structure and README, not verified as common module was not cloned in repository).

---

## Summary

`nats-test` is a standalone TypeScript demonstration package for NATS Streaming message broker patterns. It provides:

1. **Reusable base classes** for type-safe event publishing and listening with generics
2. **Example implementations** for `ticket:created` and `ticket:updated` events
3. **Development scripts** for testing pub/sub locally with Kubernetes-deployed NATS

The package requires a NATS Streaming server running on `localhost:4222` (typically via Kubernetes port-forwarding as documented in README). It is not a production service but a learning/testing tool within the microservices course project.
