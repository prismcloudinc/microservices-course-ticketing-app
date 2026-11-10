---
service_id: ticketing-app/orders
service_name: orders
service_type: microservice
framework: express
language: typescript
path: services/orders
last_analyzed: 2026-11-10T20:25:08.791Z
commit_sha: a403fff288018e2dbbe4bd55b6d96446a2ef30ed
api_endpoint_count: 5
dependency_count: 6
schema_version: "1.0"
---

# orders

## Overview

**Type**: microservice
**Framework**: express
**Language**: typescript
**Path**: `services/orders`

## Dependencies

### Internal
- **mongodb** (shared-db) — MongoDB connection via Mongoose (MONGO_URI environment variable)
- **@b.anik/ticketing-common** (package) — Common ticketing library for events, middleware, and types
- **express** (package) — Web framework for HTTP server
- **mongoose** (package) — ODM for MongoDB database operations
- **jsonwebtoken** (package) — JWT authentication and verification
- **express-validator** (package) — Request validation middleware

### External (npm)
- @b.anik/ticketing-common
- body-parser
- cookie-session
- express
- express-async-errors
- express-validator
- jsonwebtoken
- mongoose
- mongoose-update-if-current
- node-nats-streaming


## API Endpoints

| Method | Path | Handler | Auth |
|--------|------|---------|------|
| GET | /api/orders/health | src/routes/health.ts:healthRouter | — |
| GET | /api/orders | src/routes/show.ts:showOrderRouter | JWT |
| GET | /api/orders/:orderId | src/routes/show.ts:showOrderRouter | JWT |
| POST | /api/orders | src/routes/new.ts:newOrderRouter | JWT |
| PATCH | /api/orders/:orderId/cancel | src/routes/cancel.ts:cancelOrderRouter | JWT |

## Communication

### Outbound
- **message-queue** → nats-streaming: {"description":"NATS Streaming pub/sub via node-nats-streaming (NATS_URL, NATS_CLUSTER_ID environment variables)"}
- **event-bus** → ticket-service: {"description":"Subscribes to TicketCreatedEvent and TicketUpdatedEvent via NATS"}
- **event-bus** → expiration-service: {"description":"Subscribes to ExpirationCompleteEvent via NATS"}
- **event-bus** → payment-service: {"description":"Subscribes to PaymentAwaitedEvent and PaymentCompleteEvent via NATS"}

### Inbound
_No inbound communication detected._
