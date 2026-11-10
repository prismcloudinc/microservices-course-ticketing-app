---
service_id: ticketing-app/auth
service_name: auth
prompt_id: 03-message-event-contracts
prompt_label: Message & Event Contracts
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.127755
schema_version: "2.0"
---

# Messaging & Communication Contracts: Auth Service

## Executive Summary

The auth service is a **standalone HTTP-only authentication service** that does not participate in the event-driven messaging architecture used by other services in the ticketing application monorepo.

After exhaustive analysis of all source files in `services/auth/src/`:
- **No message broker producers found** — no Kafka, RabbitMQ, NATS, SQS, Redis pub/sub, or EventBridge integrations
- **No message consumers found** — no event listeners, queue workers, or subscription handlers
- **No workflow orchestrations found** — no Temporal, Step Functions, Inngest, or Bull queue usage
- **No outbound HTTP calls to other services found** — no axios, fetch, got, or HTTP client libraries
- **No external API integrations found** — no third-party service calls

---

## Part A: Events/Messages PRODUCED by this service

**None identified.**

The auth service does not publish any events or messages to any message broker. Unlike other services in the monorepo (tickets, orders, payments, expiration) which have `nats-wrapper.ts` and `events/publishers/` directories, the auth service has no such infrastructure.

**Evidence:**
- No `nats-wrapper.ts` file exists in `services/auth/src/`
- No `events/` directory exists in `services/auth/src/`
- Grep for `publish|emit|send|producer|kafka|rabbit|nats|redis|sqs|eventbridge|queue` returns no relevant matches in service source code (`services/auth/src/`)
- `package.json` contains no messaging library dependencies — only `express`, `mongoose`, `jsonwebtoken`, `cookie-session`, `body-parser`, and validation libraries

---

## Part B: Events/Messages CONSUMED by this service

**None identified.**

The auth service does not subscribe to or consume any events from message brokers.

**Evidence:**
- No listener classes or event handler files exist in `services/auth/src/`
- Grep for `subscribe|consume|listener|handler|on\(|worker` returns no relevant matches in service source code
- No NATS connection or subscription code exists in `services/auth/src/index.ts` — only MongoDB connection and Express server startup

---

## Part C: Workflow Orchestrations

**None identified.**

The auth service does not define, start, or participate in any durable workflow orchestrations.

**Evidence:**
- No Temporal, Inngest, Bull, or Step Functions client imports
- No workflow definitions or activity implementations
- Grep for `EventEmitter|Bull|Temporal|Inngest|StepFunction` returns no matches in `services/auth/`

---

## Part D: HTTP Calls to Other Services

**None identified.**

The auth service does not make any outbound HTTP calls to other internal services.

**Evidence:**
- No `axios`, `fetch`, `got`, `http`, or `https` imports in any source file
- All imports in route handlers are limited to: `express`, `express-validator`, `jsonwebtoken`, `mongoose`, and `@b.anik/ticketing-common` (`services/auth/src/routes/*.ts`)
- The service only receives inbound HTTP requests (via ingress at `/api/users/*`) and returns responses

---

## Part E: Webhooks and External API Calls

**None identified.**

The auth service does not call any external third-party APIs or expose webhook endpoints.

**Evidence:**
- No external API client libraries in `package.json` (no Stripe, Twilio, SendGrid, etc.)
- No environment variables for external service URLs or API keys (only `JWT_KEY` and `MONGO_URI` per `services/auth/src/index.ts:6-12`)
- All HTTP endpoints are internal authentication operations

---

## Appendix: Service Communication Architecture Context

### How Auth Service Fits in the Ticketing Platform

The auth service operates as an **isolated, synchronous authentication boundary**:

| Communication Path | Direction | Mechanism |
|--------------------|-----------|-----------|
| Client → Auth | Inbound | HTTP via Kubernetes Ingress (`/api/users/*` → `auth-srv:3000`) |
| Auth → MongoDB | Outbound | Mongoose ODM to `auth-mongo-srv:27017/auth` |

### Contrast with Other Services

Other services in the monorepo participate in NATS Streaming event-driven communication:

| Service | Has `nats-wrapper.ts` | Has Event Publishers | Has Event Listeners |
|---------|----------------------|---------------------|---------------------|
| auth | ❌ No | ❌ No | ❌ No |
| tickets | ✅ Yes | ✅ `ticket-created`, `ticket-updated` | ✅ `order-created`, `order-cancelled` |
| orders | ✅ Yes | ✅ `order-created`, `order-cancelled`, `order-updated` | ✅ `ticket-*`, `expiration-complete`, `payment-*` |
| payments | ✅ Yes | ✅ `payment-awaited`, `payment-complete` | ✅ `order-created`, `order-cancelled`, `order-updated` |
| expiration | ✅ Yes | ✅ `expiration-complete` | ✅ `order-created` |

**Architectural implication:** User creation/signup does not trigger events to other services. Other services that need user identity rely on JWT validation via the shared `currentUserMiddleware` from `@b.anik/ticketing-common`, not on user-related events.

### Infrastructure Configuration

From `infra/k8s/auth-depl.yaml`:
- Deployment: 1 replica of `berkinanik/ticketing-auth:latest`
- Service: `auth-srv` on port 3000
- No NATS connection environment variables (unlike other services which have `NATS_CLIENT_ID`, `NATS_URL`, `NATS_CLUSTER_ID`)

---

## Summary

The auth service is a **communication endpoint, not a participant** in the platform's event-driven architecture. It:

1. **Exposes HTTP endpoints** for client authentication (`sign-up`, `sign-in`, `sign-out`, `current-user`)
2. **Persists user data** to its dedicated MongoDB instance
3. **Issues JWTs** that other services validate via shared middleware
4. **Does not emit events** when users are created or authenticated
5. **Does not consume events** from other services
6. **Does not call** other services or external APIs

This design treats authentication as a synchronous concern where other services verify identity through JWT inspection rather than event subscription.
