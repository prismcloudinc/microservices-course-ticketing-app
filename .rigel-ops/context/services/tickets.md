---
service_id: ticketing-app/tickets
service_name: tickets
service_type: microservice
framework: express
language: typescript
path: services/tickets
last_analyzed: 2026-11-10T20:25:08.791Z
commit_sha: a403fff288018e2dbbe4bd55b6d96446a2ef30ed
api_endpoint_count: 6
dependency_count: 3
schema_version: "1.0"
---

# tickets

## Overview

**Type**: microservice
**Framework**: express
**Language**: typescript
**Path**: `services/tickets`

## Dependencies

### Internal
- **mongoose** (shared-db) — MongoDB connection via MONGO_URI environment variable
- **nats** (message-queue) — NATS Streaming connection via NATS_URL, NATS_CLUSTER_ID, NATS_CLIENT_ID
- **@b.anik/ticketing-common** (package) — Common library for event types, middleware, and error handling

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
| GET | /api/tickets/health | src/routes/health.ts:healthRouter | — |
| POST | /api/tickets | src/routes/new.ts:newTicketRouter | JWT |
| GET | /api/tickets | src/routes/show.ts:showTicketRouter | — |
| GET | /api/tickets/:id | src/routes/show.ts:showTicketRouter | — |
| PUT | /api/tickets/:id | src/routes/update.ts:updateTicketRouter | JWT |
| DELETE | /api/tickets/:id | src/routes/delete.ts:deleteTicketRouter | JWT |

## Communication

### Outbound
- **message-queue** → order-service: {"description":"NATS Streaming listener for OrderCreatedEvent and OrderCancelledEvent subjects"}

### Inbound
_No inbound communication detected._
