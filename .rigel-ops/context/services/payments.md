---
service_id: ticketing-app/payments
service_name: payments
service_type: microservice
framework: express
language: typescript
path: services/payments
last_analyzed: 2026-11-10T20:25:08.791Z
commit_sha: a403fff288018e2dbbe4bd55b6d96446a2ef30ed
api_endpoint_count: 4
dependency_count: 6
schema_version: "1.0"
---

# payments

## Overview

**Type**: microservice
**Framework**: express
**Language**: typescript
**Path**: `services/payments`

## Dependencies

### Internal
- **stripe** (package) — Stripe SDK for payment processing via stripe.charges.create()
- **mongodb** (shared-db) — MongoDB via mongoose for Order and Payment models
- **nats** (message-queue) — NATS Streaming for event-based communication via node-nats-streaming
- **@b.anik/ticketing-common** (package) — Shared library with common middleware, error classes, and event definitions
- **express** (package) — Express.js framework for HTTP API
- **mongoose** (package) — MongoDB ODM for data persistence

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
- stripe


## API Endpoints

| Method | Path | Handler | Auth |
|--------|------|---------|------|
| GET | /api/payments/health | src/routes/health.ts:healthRouter | — |
| POST | /api/payments | src/routes/new.ts:newPaymentRouter | JWT |
| GET | /api/payments | src/routes/show.ts:showPaymentRouter | JWT |
| GET | /api/payments/:id | src/routes/show.ts:showPaymentRouter | JWT |

## Communication

### Outbound
- **message-queue** → order-service: {"description":"Subscribes to NATS Subjects.OrderCreated, Subjects.OrderCancelled, Subjects.OrderUpdated events via OrderCreatedListener, OrderCancelledListener, OrderUpdatedListener"}
- **http** → stripe: {"description":"HTTP API calls to Stripe for creating charges with payment tokens"}

### Inbound
_No inbound communication detected._
