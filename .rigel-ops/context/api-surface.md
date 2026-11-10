---
repository: microservices-course-ticketing-app
total_endpoints: 21
last_analyzed: 2026-11-10T20:25:08.791Z
schema_version: "1.0"
---

# API Surface

**Total Endpoints**: 21

### tickets

| Method | Path | Handler | Description |
|--------|------|---------|-------------|
| GET | /api/tickets/health | src/routes/health.ts:healthRouter | Health check endpoint |
| POST | /api/tickets | src/routes/new.ts:newTicketRouter | Create a new ticket |
| GET | /api/tickets | src/routes/show.ts:showTicketRouter | List all tickets |
| GET | /api/tickets/:id | src/routes/show.ts:showTicketRouter | Get ticket by ID |
| PUT | /api/tickets/:id | src/routes/update.ts:updateTicketRouter | Update ticket by ID |
| DELETE | /api/tickets/:id | src/routes/delete.ts:deleteTicketRouter | Delete ticket by ID |

### payments

| Method | Path | Handler | Description |
|--------|------|---------|-------------|
| GET | /api/payments/health | src/routes/health.ts:healthRouter | Health check endpoint |
| POST | /api/payments | src/routes/new.ts:newPaymentRouter | Create a new payment charge via Stripe |
| GET | /api/payments | src/routes/show.ts:showPaymentRouter | Fetch all payments for current user |
| GET | /api/payments/:id | src/routes/show.ts:showPaymentRouter | Fetch payment by ID |

### orders

| Method | Path | Handler | Description |
|--------|------|---------|-------------|
| GET | /api/orders/health | src/routes/health.ts:healthRouter | Health check endpoint |
| GET | /api/orders | src/routes/show.ts:showOrderRouter | List all orders for the current user |
| GET | /api/orders/:orderId | src/routes/show.ts:showOrderRouter | Fetch a specific order by ID |
| POST | /api/orders | src/routes/new.ts:newOrderRouter | Create a new order for a ticket |
| PATCH | /api/orders/:orderId/cancel | src/routes/cancel.ts:cancelOrderRouter | Cancel an existing order |

### auth

| Method | Path | Handler | Description |
|--------|------|---------|-------------|
| GET | /api/users/health | src/routes/health.ts:healthRouter | Health check endpoint |
| GET | /api/users/current-user | src/routes/current-user.ts:currentUserRouter | Get current authenticated user |
| POST | /api/users/sign-up | src/routes/sign-up.ts:signUpRouter | Register a new user |
| POST | /api/users/sign-in | src/routes/sign-in.ts:signInRouter | Authenticate user and create session |
| POST | /api/users/sign-out | src/routes/sign-out.ts:signOutRouter | Logout user and clear session |

### client

| Method | Path | Handler | Description |
|--------|------|---------|-------------|
| GET | /health/check | apps/client/src/app/health/check/route.ts:GET | Health check endpoint |
