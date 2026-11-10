---
service_id: ticketing-app/auth
service_name: auth
prompt_id: 07-integration-manifest
prompt_label: Outbound Integration Manifest
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.14320000000000002
schema_version: "2.0"
---

# Integration Manifest: Auth Service

## Overview

The auth service is a user authentication microservice within the ticketing-app platform. It provides sign-up, sign-in, sign-out, and current-user retrieval functionality. This service is intentionally isolated from the event-driven architecture used by other services (tickets, orders, payments, expiration) — it does **not** publish or consume NATS events.

---

## Part A: Services I Call (HTTP)

**None identified.**

The auth service does not make any outbound HTTP calls to other services. All route handlers (`services/auth/src/routes/*.ts`) operate solely on the local User model and session management. No HTTP client libraries (axios, fetch, got) are imported in the source code.

---

## Part B: Messages/Events I Produce

**None identified.**

Unlike other services in this microservices architecture (tickets, orders, payments, expiration), the auth service does **not** use NATS Streaming. Verified by:
- No `nats-wrapper.ts` file exists in `services/auth/src/`
- No imports from `node-nats-streaming` or NATS-related modules in source files
- No `events/` directory or publisher classes
- No `NATS_CLIENT_ID`, `NATS_URL`, or `NATS_CLUSTER_ID` environment variables in Kubernetes deployment (`infra/k8s/auth-depl.yaml:18-25`)

---

## Part C: Messages/Events I Consume

**None identified.**

The auth service does not subscribe to any message queues or event streams. There are no listener classes or event handlers. (See Part B for verification details.)

---

## Part D: Workflows I Trigger

**None identified.**

No Temporal, Inngest, Step Functions, or other workflow orchestration patterns are used in this service.

---

## Part E: Workflows/Activities I Execute

**None identified.**

This service is not a workflow worker.

---

## Part F: Cloud Services I Use

| Cloud Service | Operation | Resource | Purpose |
|---------------|-----------|----------|---------|
| MongoDB | CRUD operations | `auth` database via `MONGO_URI` (`mongodb://auth-mongo-srv:27017/auth`) | User document storage and retrieval |

**Source references:**
- Connection: `services/auth/src/index.ts:15` — `mongoose.connect(process.env.MONGO_URI!)`
- Model operations: `services/auth/src/models/user.ts` — User schema with `findOne()`, `save()`, `build()` methods
- Kubernetes config: `infra/k8s/auth-depl.yaml:24-25` — `MONGO_URI` set to `mongodb://auth-mongo-srv:27017/auth`

---

## Part G: External Systems I Integrate With

**None identified.**

The auth service does not integrate with any external third-party systems such as:
- Payment processors (Stripe, etc.)
- Email services (SendGrid, Mailgun, etc.)
- SMS services (Twilio, etc.)
- Analytics platforms
- CRM/ERP systems

---

## Database Details

### MongoDB — User Collection

| Field | Type | Purpose |
|-------|------|---------|
| `_id` | ObjectId | Primary key (transformed to `id` in JSON responses) |
| `email` | String | User's email address (required) |
| `password` | String | Hashed password in `hash.salt` format using scrypt (required, excluded from JSON responses) |

**Collection operations:**
- `User.findOne({ email })` — Sign-in credential lookup, duplicate email check (`services/auth/src/routes/sign-in.ts:20`, `services/auth/src/routes/sign-up.ts:22`)
- `User.build({ email, password })` + `user.save()` — User registration (`services/auth/src/routes/sign-up.ts:30-35`)

---

## Shared Library Usage

The service imports from `@b.anik/ticketing-common` (external npm package):

| Import | Usage | Source |
|--------|-------|--------|
| `NotFoundError` | 404 handler middleware | `services/auth/src/app.ts:7` |
| `BadRequestError` | Invalid credentials/duplicate email errors | `services/auth/src/routes/sign-in.ts:5`, `services/auth/src/routes/sign-up.ts:5` |
| `currentUserMiddleware` | JWT extraction from session cookie | `services/auth/src/app.ts:8` |
| `errorHandlerMiddleware` | Consistent error response formatting | `services/auth/src/app.ts:9` |
| `requireAuthMiddleware` | Authentication guard for sign-out | `services/auth/src/routes/sign-out.ts:3` |
| `validateRequestMiddleware` | Request validation error handling | `services/auth/src/routes/sign-in.ts:5`, `services/auth/src/routes/sign-up.ts:5` |

This is a utility library dependency, not an external service connection.

---

## Environment Variables

| Variable | Description | Source |
|----------|-------------|--------|
| `JWT_KEY` | Secret for signing JWTs | `services/auth/src/index.ts:6-8`, K8s secret `jwt-secret` |
| `MONGO_URI` | MongoDB connection string | `services/auth/src/index.ts:10-12`, defaults to `mongodb://auth-mongo-srv:27017/auth` |

---

## Summary Block

```yaml
service: auth
platform: ticketing-app
connects_to:
  http_outbound: []
  events_produced: []
  events_consumed: []
  workflows_triggered: []
  workflows_executed: []
  cloud_services:
    - service: MongoDB
      resources:
        - auth-mongo-srv:27017/auth
  external: []
```
