---
service_id: ticketing-app/client
service_name: client
prompt_id: 07-integration-manifest
prompt_label: Outbound Integration Manifest
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.14720000000000003
schema_version: "2.0"
---

# External Connections Manifest: client

## Part A: Services I Call (HTTP)

All HTTP calls from this service are routed through the Kubernetes ingress controller (`ingress-nginx-controller.ingress-nginx.svc.cluster.local`). Server-side requests go directly to the ingress with `host: ticketing.dev` header; client-side requests use relative paths `/` which the browser resolves against the current domain.

### -> Auth Service (via Ingress)
- **Client file**: `apps/client/src/http/buildClient.ts` (client-side), `apps/client/src/http/buildClientServer.ts` (server-side)
- **Base URL env var**: None — hardcoded `http://ingress-nginx-controller.ingress-nginx.svc.cluster.local` for server-side (`buildClientServer.ts` line 16), `/` for client-side (`buildClient.ts` line 7)
- **Auth**: Cookie-based authentication via `cookie` header forwarding (`buildClientServer.ts` line 12-13)
- **Calls**:

| Method | Path | Purpose | Request Body Shape | Response Shape |
|--------|------|---------|--------------------|----------------|
| POST | `/api/users/sign-up` | User registration | `{ email: string, password: string }` | `{ user: { id: string, email: string } }` |
| POST | `/api/users/sign-in` | User login | `{ email: string, password: string }` | `{ user: { id: string, email: string } }` |
| POST | `/api/users/sign-out` | User logout | None | `{}` |
| GET | `/api/users/current-user` | Get current authenticated user | None | `{ currentUser: { id: string, email: string } \| null }` |

**Source files**:
- Sign-up/Sign-in: `apps/client/src/app/auth/AuthForm.tsx` line 33
- Sign-out: `apps/client/src/components/navbar/AuthButtons.tsx` line 21
- Current user: `apps/client/src/services/get-current-user.ts` line 9

### -> Tickets Service (via Ingress)
- **Client file**: `apps/client/src/http/buildClient.ts` (client-side), `apps/client/src/http/buildClientServer.ts` (server-side)
- **Base URL env var**: None — same as above
- **Auth**: Cookie-based authentication via `cookie` header forwarding
- **Calls**:

| Method | Path | Purpose | Request Body Shape | Response Shape |
|--------|------|---------|--------------------|----------------|
| GET | `/api/tickets` | List all available tickets | None | `{ tickets: Ticket[] }` where `Ticket = { id: string, title: string, price: number, userId: string, version: number, orderId?: string }` |
| GET | `/api/tickets/:ticketId` | Get a specific ticket by ID | None | `{ ticket: Ticket }` |
| POST | `/api/tickets` | Create a new ticket | `{ title: string, price: number }` | `{ ticket: Ticket }` |

**Source files**:
- List tickets: `apps/client/src/app/page.tsx` line 11
- Get ticket: `apps/client/src/app/tickets/[ticketId]/page.tsx` line 22
- Create ticket: `apps/client/src/app/tickets/new/page.tsx` line 23

### -> Orders Service (via Ingress)
- **Client file**: `apps/client/src/http/buildClient.ts` (client-side), `apps/client/src/http/buildClientServer.ts` (server-side)
- **Base URL env var**: None — same as above
- **Auth**: Cookie-based authentication via `cookie` header forwarding
- **Calls**:

| Method | Path | Purpose | Request Body Shape | Response Shape |
|--------|------|---------|--------------------|----------------|
| GET | `/api/orders` | List all orders for current user | None | `{ orders: Order[] }` where `Order = { id: string, userId: string, status: OrderStatus, expiresAt?: string, ticket: { id: string, price: number }, version: number }` |
| GET | `/api/orders/:orderId` | Get a specific order by ID | None | `{ order: Order }` |
| POST | `/api/orders` | Create a new order (reserve a ticket) | `{ ticketId: string }` | `{ order: Order }` |

**Source files**:
- List orders: `apps/client/src/app/orders/page.tsx` line 11
- Get order: `apps/client/src/app/orders/[orderId]/page.tsx` line 16, `apps/client/src/app/tickets/[ticketId]/page.tsx` line 28
- Create order: `apps/client/src/app/tickets/[ticketId]/PurchaseButton.tsx` line 20

### -> Payments Service (via Ingress)
- **Client file**: `apps/client/src/http/buildClient.ts`
- **Base URL env var**: None — same as above
- **Auth**: Cookie-based authentication via browser cookies
- **Calls**:

| Method | Path | Purpose | Request Body Shape | Response Shape |
|--------|------|---------|--------------------|----------------|
| POST | `/api/payments` | Create a payment for an order | `{ token: string, orderId: string }` | unknown — not determinable from the material provided |

**Source file**: `apps/client/src/app/orders/[orderId]/CheckoutForm.tsx` line 23

## Part B: Messages/Events I Produce

**None** — This is a frontend web application that does not directly publish messages to any broker, queue, or event bus.

## Part C: Messages/Events I Consume

**None** — This is a frontend web application that does not subscribe to any message sources.

## Part D: Workflows I Trigger

**None** — No durable workflow systems (Temporal, Step Functions, Inngest, etc.) are triggered by this service.

## Part E: Workflows/Activities I Execute

**None** — This is not a workflow worker or library.

## Part F: Cloud Services I Use

**None** — This service does not make direct SDK calls to cloud services (AWS, GCP, Azure, etc.). All backend interactions are routed through the HTTP API gateway.

## Part G: External Systems I Integrate With

### <-> Stripe (Frontend Checkout SDK)
- **Client/SDK file**: `apps/client/src/app/orders/[orderId]/CheckoutForm.tsx`
- **SDK**: `react-stripe-checkout` v2.6.3 (`package.json` line 24)
- **Operations**: 
  - Renders Stripe Checkout modal for card collection
  - Passes payment token to backend after successful card entry
- **Data exchanged**:
  - **To Stripe (via SDK)**: `stripeKey` (publishable key), `amount` (in cents), `email` (current user's email)
  - **From Stripe (via SDK)**: `Token` object containing `token.id` (payment token string)
- **Auth**: `STRIPE_CLIENT_KEY` environment variable containing Stripe publishable key (`apps/client/src/app/orders/[orderId]/page.tsx` line 22)
- **Note**: The `react-stripe-checkout` component communicates directly with Stripe's servers from the browser. The actual payment processing (charging the card) occurs in the backend payments service.

---

## Environment Variables

| Variable | Purpose | Used In |
|----------|---------|---------|
| `STRIPE_CLIENT_KEY` | Stripe publishable API key for frontend checkout | `apps/client/src/app/orders/[orderId]/page.tsx` line 22 |

---

## Summary Block

```yaml
service: client
platform: ticketing-app
connects_to:
  http_outbound:
    - target: auth-service
      endpoints:
        - POST /api/users/sign-up
        - POST /api/users/sign-in
        - POST /api/users/sign-out
        - GET /api/users/current-user
    - target: tickets-service
      endpoints:
        - GET /api/tickets
        - GET /api/tickets/:ticketId
        - POST /api/tickets
    - target: orders-service
      endpoints:
        - GET /api/orders
        - GET /api/orders/:orderId
        - POST /api/orders
    - target: payments-service
      endpoints:
        - POST /api/payments
  events_produced: []
  events_consumed: []
  workflows_triggered: []
  workflows_executed: []
  cloud_services: []
  external:
    - system: stripe
      operations:
        - Stripe Checkout SDK (card collection modal)
```
