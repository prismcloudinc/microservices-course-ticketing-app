---
service_id: ticketing-app/client
service_name: client
prompt_id: 03-message-event-contracts
prompt_label: Message & Event Contracts
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.19714500000000004
schema_version: "2.0"
---

# Communication Contracts: client

## Executive Summary

The `client` service is a Next.js frontend web application that communicates exclusively via HTTP with backend microservices through a Kubernetes ingress controller. **It does not produce or consume any message broker events** (Kafka, RabbitMQ, SQS, NATS, Redis pub/sub, EventBridge, etc.) and **does not use any workflow orchestration** (Temporal, Inngest, AWS Step Functions, Bull queues, etc.).

All inter-service communication is HTTP-based, routed through the nginx ingress controller to backend services.

---

## Part A: Events/Messages PRODUCED by this service

**None.**

This is a frontend web application. It does not publish messages to any message broker, event bus, or queue. All outbound communication is via HTTP requests to backend API endpoints.

---

## Part B: Events/Messages CONSUMED by this service

**None.**

This is a frontend web application. It does not consume messages from any message broker, event bus, or queue. Data is fetched via HTTP requests triggered by page loads or user actions.

---

## Part C: Workflow Orchestrations

**None.**

No durable workflow systems (Temporal, AWS Step Functions, Inngest, Bull queues, or similar) are used in this service.

---

## Part D: HTTP Calls to Other Services

The client uses two HTTP client configurations:

1. **Client-side requests** (`buildClient.ts` lines 5-10): axios instance with `baseURL: '/'` — requests from the browser go through the page origin to the ingress controller
2. **Server-side requests** (`buildClientServer.ts` lines 7-20): axios instance with `baseURL: 'http://ingress-nginx-controller.ingress-nginx.svc.cluster.local'`, sets `host: ticketing.dev` header and forwards cookies for authentication

All requests route through the Kubernetes nginx ingress controller which proxies to the appropriate backend microservice based on path matching.

---

### Call: Auth Service — POST /api/users/sign-up

- **Client file**: `src/http/buildClient.ts` (client-side)
- **Caller**: `src/app/auth/AuthForm.tsx` line 27, `handleSubmit` function
- **Target service**: Auth service via ingress (`/api/users/*` path)
- **Method + Path**: `POST /api/users/sign-up`
- **Authentication**: None (registration endpoint)
- **Request body**:
  ```json
  {
    "email": "string — from form input",
    "password": "string — from form input"
  }
  ```
  (built at `AuthForm.tsx` lines 24-35)
- **Expected response**:
  ```json
  {
    "user": {
      "id": "string",
      "email": "string"
    }
  }
  ```
  (typed at `AuthForm.tsx` lines 28-32)
- **Error handling**: On failure, `res.ok === false`, errors array displayed via `toastErrors()` (`AuthForm.tsx` line 45)
- **Retry configuration**: None — single attempt only

---

### Call: Auth Service — POST /api/users/sign-in

- **Client file**: `src/http/buildClient.ts` (client-side)
- **Caller**: `src/app/auth/AuthForm.tsx` line 27, `handleSubmit` function
- **Target service**: Auth service via ingress
- **Method + Path**: `POST /api/users/sign-in`
- **Authentication**: None (login endpoint)
- **Request body**:
  ```json
  {
    "email": "string — from form input",
    "password": "string — from form input"
  }
  ```
- **Expected response**:
  ```json
  {
    "user": {
      "id": "string",
      "email": "string"
    }
  }
  ```
- **Error handling**: On failure, errors displayed via toast notifications
- **Retry configuration**: None

---

### Call: Auth Service — POST /api/users/sign-out

- **Client file**: `src/http/buildClient.ts` (client-side)
- **Caller**: `src/components/navbar/AuthButtons.tsx` line 21, `handleSignOut` function
- **Target service**: Auth service via ingress
- **Method + Path**: `POST /api/users/sign-out`
- **Authentication**: Cookie-based session (sent automatically by browser)
- **Request body**: None (empty POST)
- **Expected response**: Success response (structure not typed, only success/failure checked)
- **Error handling**: On exception, generic "Failed to logout" toast error (`AuthButtons.tsx` lines 27-28)
- **Retry configuration**: None

---

### Call: Auth Service — GET /api/users/current-user

- **Client file**: `src/http/buildClientServer.ts` (server-side)
- **Caller**: `src/services/get-current-user.ts` lines 6-13, wrapped with React `cache()`
- **Invoked from**: `src/app/layout.tsx` line 23 (on every page render), `src/app/orders/[orderId]/page.tsx` line 14
- **Target service**: Auth service via ingress
- **Method + Path**: `GET /api/users/current-user`
- **Authentication**: Cookie forwarded from incoming request (`buildClientServer.ts` lines 12-13)
- **Request body**: None
- **Expected response** (typed at `src/@types/user.ts` lines 1-5):
  ```json
  {
    "currentUser": {
      "id": "string",
      "email": "string"
    }
  }
  ```
  or `{ "currentUser": null }` if not authenticated
- **Error handling**: On failure, returns `null` (`get-current-user.ts` line 10)
- **Retry configuration**: None

---

### Call: Tickets Service — GET /api/tickets

- **Client file**: `src/http/buildClientServer.ts` (server-side)
- **Caller**: `src/app/page.tsx` lines 9-12, `Home` server component
- **Target service**: Tickets service via ingress
- **Method + Path**: `GET /api/tickets`
- **Authentication**: Cookie forwarded (though endpoint likely public)
- **Request body**: None
- **Expected response** (typed at `src/@types/ticket.ts` lines 10-12):
  ```json
  {
    "tickets": [
      {
        "id": "string",
        "title": "string",
        "price": "number",
        "userId": "string",
        "version": "number",
        "orderId": "string | undefined — present if ticket is reserved"
      }
    ]
  }
  ```
- **Error handling**: On failure, returns empty array (`page.tsx` line 12: `res.data?.tickets ?? []`)
- **Retry configuration**: None

---

### Call: Tickets Service — GET /api/tickets/:ticketId

- **Client file**: `src/http/buildClientServer.ts` (server-side)
- **Caller**: `src/app/tickets/[ticketId]/page.tsx` lines 22-36, `TicketPage` server component
- **Target service**: Tickets service via ingress
- **Method + Path**: `GET /api/tickets/{ticketId}` where `ticketId` is from URL params
- **Authentication**: Cookie forwarded
- **Request body**: None
- **Expected response** (typed at `src/@types/ticket.ts` lines 14-16):
  ```json
  {
    "ticket": {
      "id": "string",
      "title": "string",
      "price": "number",
      "userId": "string",
      "version": "number",
      "orderId": "string | undefined"
    }
  }
  ```
- **Error handling**: On failure (`res.ok === false`), calls `notFound()` returning 404 (`page.tsx` line 35)
- **Retry configuration**: None

---

### Call: Tickets Service — POST /api/tickets

- **Client file**: `src/http/buildClient.ts` (client-side)
- **Caller**: `src/app/tickets/new/page.tsx` lines 22-35, `handleSubmit` function
- **Target service**: Tickets service via ingress
- **Method + Path**: `POST /api/tickets`
- **Authentication**: Cookie-based session (browser sends automatically)
- **Request body** (built at lines 19-25):
  ```json
  {
    "title": "string — from form input",
    "price": "number — parsed and rounded to 2 decimal places from form input"
  }
  ```
- **Expected response**: Success response (type not explicitly defined, only `res.ok` checked)
- **Error handling**: On failure, errors displayed via `toastErrors()` (line 33)
- **Retry configuration**: None

---

### Call: Orders Service — GET /api/orders

- **Client file**: `src/http/buildClientServer.ts` (server-side)
- **Caller**: `src/app/orders/page.tsx` lines 10-15, `OrdersPage` server component
- **Target service**: Orders service via ingress
- **Method + Path**: `GET /api/orders`
- **Authentication**: Cookie forwarded from incoming request
- **Request body**: None
- **Expected response** (typed at `src/@types/order.ts` lines 15-17):
  ```json
  {
    "orders": [
      {
        "id": "string",
        "userId": "string",
        "status": "string — one of: 'created', 'cancelled', 'awaiting-payment', 'complete'",
        "expiresAt": "string | undefined — ISO timestamp",
        "ticket": {
          "id": "string",
          "price": "number"
        },
        "version": "number"
      }
    ]
  }
  ```
- **Error handling**: On failure, returns empty array (`page.tsx` line 14)
- **Retry configuration**: None

---

### Call: Orders Service — GET /api/orders/:orderId

- **Client file**: `src/http/buildClientServer.ts` (server-side)
- **Caller**: 
  - `src/app/orders/[orderId]/page.tsx` lines 16-19, `OrderPage` server component
  - `src/app/tickets/[ticketId]/page.tsx` lines 27-29 (conditional fetch if ticket has orderId)
- **Target service**: Orders service via ingress
- **Method + Path**: `GET /api/orders/{orderId}` where `orderId` is from URL params or ticket.orderId
- **Authentication**: Cookie forwarded
- **Request body**: None
- **Expected response** (typed at `src/@types/order.ts` lines 19-21):
  ```json
  {
    "order": {
      "id": "string",
      "userId": "string",
      "status": "string",
      "expiresAt": "string | undefined",
      "ticket": {
        "id": "string",
        "price": "number"
      },
      "version": "number"
    }
  }
  ```
- **Error handling**: 
  - In `OrderPage`: calls `notFound()` returning 404 on failure (line 19)
  - In `TicketPage`: returns `null` on failure (line 29)
- **Retry configuration**: None

---

### Call: Orders Service — POST /api/orders

- **Client file**: `src/http/buildClient.ts` (client-side)
- **Caller**: `src/app/tickets/[ticketId]/PurchaseButton.tsx` lines 19-27, `handlePurchase` function
- **Target service**: Orders service via ingress
- **Method + Path**: `POST /api/orders`
- **Authentication**: Cookie-based session
- **Request body** (built at line 20):
  ```json
  {
    "ticketId": "string — from component props"
  }
  ```
- **Expected response**: Success response (type not explicitly defined, only `res.ok` checked)
- **Error handling**: On failure, errors displayed via `toastErrors()` (line 25)
- **Retry configuration**: None

---

### Call: Payments Service — POST /api/payments

- **Client file**: `src/http/buildClient.ts` (client-side)
- **Caller**: `src/app/orders/[orderId]/CheckoutForm.tsx` line 23, `handleToken` callback
- **Trigger**: Called after Stripe checkout completes and returns a token
- **Target service**: Payments service via ingress
- **Method + Path**: `POST /api/payments`
- **Authentication**: Cookie-based session
- **Request body** (built at line 23):
  ```json
  {
    "token": "string — Stripe token ID from react-stripe-checkout callback",
    "orderId": "string — from order prop"
  }
  ```
- **Expected response**: Success response (type not explicitly defined, only `res.ok` checked)
- **Error handling**: On failure, errors displayed via `toastErrors()` (line 29)
- **Retry configuration**: None

---

## Part E: Webhooks and External API Calls

### External: Stripe — Checkout Tokenization

- **Client file**: `src/app/orders/[orderId]/CheckoutForm.tsx` lines 34-41
- **Library**: `react-stripe-checkout` package (version ^2.6.3)
- **Configuration**: Stripe publishable key from `process.env.STRIPE_CLIENT_KEY` (`src/app/orders/[orderId]/page.tsx` line 22)
- **Operation**: Client-side card tokenization via Stripe Checkout pop-up
- **Flow**:
  1. User clicks the Stripe checkout button (rendered by `StripeCheckout` component)
  2. Stripe's hosted form collects card details (never touches the client server)
  3. Stripe returns a `Token` object to the `handleToken` callback
  4. Token ID (`token.id`) is sent to backend `/api/payments` endpoint
- **Input sent to Stripe** (component props at lines 36-39):
  - `stripeKey`: Stripe publishable key
  - `amount`: `order.ticket.price * 100` (price in cents)
  - `email`: `currentUser.email` (for receipt)
- **Output received**: `Token` object with `id` field (typed import from `react-stripe-checkout`)
- **Error handling**: Stripe checkout handles errors internally; if checkout is cancelled, callback is not invoked
- **Note**: This is client-side tokenization only. The actual charge happens server-side in the payments service.

---

## HTTP Client Architecture Summary

| Context | Client Factory | Base URL | Headers |
|---------|---------------|----------|---------|
| Client-side (browser) | `buildClient()` | `/` (relative, goes to ingress) | Browser defaults |
| Server-side (SSR/RSC) | `buildClientServer(headers)` | `http://ingress-nginx-controller.ingress-nginx.svc.cluster.local` | `host: ticketing.dev`, forwarded `cookie` |

Both clients return the same interface with `get` and `post` methods that wrap axios and normalize responses into `CommonResponse<T>` type (`src/http/types.ts` lines 3-22).

---

## Service Dependency Map

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Browser / User                               │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Next.js Client Application                        │
│                                                                      │
│  ┌──────────────────┐    ┌──────────────────────────────────────┐   │
│  │ Client Components│    │ Server Components (SSR/RSC)          │   │
│  │ buildClient()    │    │ buildClientServer(headers)           │   │
│  │ baseURL: /       │    │ baseURL: ingress-nginx...cluster.local│   │
│  └────────┬─────────┘    └───────────────┬──────────────────────┘   │
└───────────┼──────────────────────────────┼──────────────────────────┘
            │                              │
            ▼                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│              Kubernetes Ingress Controller (nginx)                   │
│              Host: ticketing.dev                                     │
└─────────────────────────────────────────────────────────────────────┘
            │
            ├──── /api/users/*     ────► Auth Service
            ├──── /api/tickets/*   ────► Tickets Service
            ├──── /api/orders/*    ────► Orders Service
            └──── /api/payments/*  ────► Payments Service

External:
┌─────────────────┐
│ Stripe Checkout │ ◄──── Client-side tokenization (react-stripe-checkout)
└─────────────────┘
```

---

## Environment Variables

| Variable | Usage | Location |
|----------|-------|----------|
| `STRIPE_CLIENT_KEY` | Stripe publishable key for checkout component | `src/app/orders/[orderId]/page.tsx` line 22 |
