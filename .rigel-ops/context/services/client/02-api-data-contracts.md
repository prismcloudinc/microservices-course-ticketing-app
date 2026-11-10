---
service_id: ticketing-app/client
service_name: client
prompt_id: 02-api-data-contracts
prompt_label: API & Data Contracts
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.211005
schema_version: "2.0"
---

# API Contract & Data Model Documentation: client

## Overview

The `client` service is a Next.js 14 frontend application that provides the user interface for the ticketing platform. It does **not** have its own database — it communicates with backend microservices (auth, tickets, orders, payments) via HTTP through a Kubernetes ingress controller.

---

## Part A: HTTP API Endpoints

### A.1 Local Route Handlers

The client application exposes only one local API route:

---

### [GET] /health/check

- **Controller**: `apps/client/src/app/health/check/route.ts`, `GET` function (line 5)
- **Authentication**: None
- **Authorization**: None
- **Request**:
  - Path parameters: None
  - Query parameters: None
  - Request body: None
  - Headers: None required
- **Processing**: Returns an empty JSON object immediately. Uses `force-dynamic` to prevent response caching.
- **Response**:
  - Success: `200 OK` with body `{}`
  - Error cases: None — this endpoint cannot fail
- **Side effects**: None

---

### A.2 Backend API Calls (Proxied to Microservices)

The client makes HTTP calls to backend microservices through the Kubernetes ingress controller. These are not local endpoints but API contracts the client depends on.

#### Server-Side Calls

Server-side calls use `buildClientServer()` (`apps/client/src/http/buildClientServer.ts` lines 7-21) which:
- Sets base URL to `http://ingress-nginx-controller.ingress-nginx.svc.cluster.local`
- Forwards cookies from the incoming request for authentication
- Sets `Host: ticketing.dev` header for ingress routing

#### Client-Side Calls

Client-side calls use `buildClient()` (`apps/client/src/http/buildClient.ts` lines 5-11) which:
- Sets base URL to `/` (same origin, relies on ingress routing)
- Includes credentials automatically via browser

---

### [GET] /api/users/current-user

- **Called from**: `apps/client/src/services/get-current-user.ts` (line 9)
- **Caller context**: Server-side, in `RootLayout` via `getCurrentUser(headers)`
- **Purpose**: Fetch the currently authenticated user
- **Request**:
  - Headers: `Cookie` (forwarded from client request)
- **Expected Response**:
  - Success: `200 OK`
    ```typescript
    {
      currentUser: { id: string; email: string } | null
    }
    ```
- **Usage**: Determines authentication state for navbar rendering

---

### [POST] /api/users/sign-up

- **Called from**: `apps/client/src/app/auth/AuthForm.tsx` (line 33)
- **Caller context**: Client-side form submission
- **Purpose**: Register a new user account
- **Request**:
  - Body:
    ```typescript
    {
      email: string;    // from form input
      password: string; // from form input
    }
    ```
- **Expected Response**:
  - Success: `200/201` (inferred)
    ```typescript
    {
      user: {
        id: string;
        email: string;
      }
    }
    ```
  - Error:
    ```typescript
    {
      errors: Array<{ message: string; field?: string }>
    }
    ```
- **Side effects**: Sets authentication cookie, triggers router redirect to `/` and refresh

---

### [POST] /api/users/sign-in

- **Called from**: `apps/client/src/app/auth/AuthForm.tsx` (line 33)
- **Caller context**: Client-side form submission
- **Purpose**: Authenticate an existing user
- **Request**:
  - Body:
    ```typescript
    {
      email: string;
      password: string;
    }
    ```
- **Expected Response**:
  - Success: `200`
    ```typescript
    {
      user: {
        id: string;
        email: string;
      }
    }
    ```
  - Error:
    ```typescript
    {
      errors: Array<{ message: string; field?: string }>
    }
    ```
- **Side effects**: Sets authentication cookie, triggers router redirect to `/` and refresh

---

### [POST] /api/users/sign-out

- **Called from**: `apps/client/src/components/navbar/AuthButtons.tsx` (line 21)
- **Caller context**: Client-side button click
- **Purpose**: Log out the current user
- **Request**:
  - Body: None (empty POST)
- **Expected Response**:
  - Success: `200` (any response)
- **Side effects**: Clears authentication cookie, triggers router redirect to `/` and refresh

---

### [GET] /api/tickets

- **Called from**: `apps/client/src/app/page.tsx` (line 11)
- **Caller context**: Server-side, home page rendering
- **Purpose**: Fetch all available tickets
- **Request**:
  - Headers: `Cookie` (forwarded)
- **Expected Response**:
  - Success: `200 OK`
    ```typescript
    {
      tickets: Array<{
        id: string;
        title: string;
        price: number;
        userId: string;
        version: number;
        orderId?: string;
      }>
    }
    ```
- **Usage**: Renders ticket grid on home page

---

### [GET] /api/tickets/:ticketId

- **Called from**: `apps/client/src/app/tickets/[ticketId]/page.tsx` (line 22)
- **Caller context**: Server-side, ticket detail page rendering
- **Purpose**: Fetch a single ticket by ID
- **Request**:
  - Path parameters:
    | Name | Type | Description |
    |------|------|-------------|
    | ticketId | string | MongoDB ObjectId of the ticket |
  - Headers: `Cookie` (forwarded)
- **Expected Response**:
  - Success: `200 OK`
    ```typescript
    {
      ticket: {
        id: string;
        title: string;
        price: number;
        userId: string;
        version: number;
        orderId?: string;
      }
    }
    ```
  - Error: Any error triggers `notFound()` (404 page)

---

### [POST] /api/tickets

- **Called from**: `apps/client/src/app/tickets/new/page.tsx` (line 23)
- **Caller context**: Client-side form submission
- **Purpose**: Create a new ticket for sale
- **Request**:
  - Body:
    ```typescript
    {
      title: string;  // from form input
      price: number;  // parsed to 2 decimal places
    }
    ```
- **Expected Response**:
  - Success: `200/201` (inferred)
  - Error:
    ```typescript
    {
      errors: Array<{ message: string; field?: string }>
    }
    ```
- **Side effects**: Triggers router redirect to `/` and refresh

---

### [GET] /api/orders

- **Called from**: `apps/client/src/app/orders/page.tsx` (line 11)
- **Caller context**: Server-side, orders list page rendering
- **Purpose**: Fetch all orders for the current user
- **Request**:
  - Headers: `Cookie` (forwarded, required for auth)
- **Expected Response**:
  - Success: `200 OK`
    ```typescript
    {
      orders: Array<{
        id: string;
        userId: string;
        status: 'created' | 'cancelled' | 'awaiting-payment' | 'complete';
        expiresAt?: string;  // ISO date string
        ticket: {
          id: string;
          price: number;
        };
        version: number;
      }>
    }
    ```
- **Usage**: Renders order list (reversed for newest-first display)

---

### [GET] /api/orders/:orderId

- **Called from**: 
  - `apps/client/src/app/orders/[orderId]/page.tsx` (line 16)
  - `apps/client/src/app/tickets/[ticketId]/page.tsx` (line 28)
- **Caller context**: Server-side, order detail or ticket detail page
- **Purpose**: Fetch a single order by ID
- **Request**:
  - Path parameters:
    | Name | Type | Description |
    |------|------|-------------|
    | orderId | string | MongoDB ObjectId of the order |
  - Headers: `Cookie` (forwarded)
- **Expected Response**:
  - Success: `200 OK`
    ```typescript
    {
      order: {
        id: string;
        userId: string;
        status: 'created' | 'cancelled' | 'awaiting-payment' | 'complete';
        expiresAt?: string;
        ticket: {
          id: string;
          price: number;
        };
        version: number;
      }
    }
    ```
  - Error: On order detail page triggers `notFound()`; on ticket detail page returns `null`

---

### [POST] /api/orders

- **Called from**: `apps/client/src/app/tickets/[ticketId]/PurchaseButton.tsx` (line 20)
- **Caller context**: Client-side button click
- **Purpose**: Create a new order to reserve a ticket
- **Request**:
  - Body:
    ```typescript
    {
      ticketId: string;  // ID of the ticket to purchase
    }
    ```
- **Expected Response**:
  - Success: `200/201` (inferred)
  - Error:
    ```typescript
    {
      errors: Array<{ message: string; field?: string }>
    }
    ```
- **Side effects**: Triggers router refresh to show order info

---

### [POST] /api/payments

- **Called from**: `apps/client/src/app/orders/[orderId]/CheckoutForm.tsx` (line 23)
- **Caller context**: Client-side Stripe checkout callback
- **Purpose**: Process payment for an order using Stripe
- **Request**:
  - Body:
    ```typescript
    {
      token: string;   // Stripe token ID from StripeCheckout
      orderId: string; // ID of the order being paid
    }
    ```
- **Expected Response**:
  - Success: `200/201` (inferred)
  - Error:
    ```typescript
    {
      errors: Array<{ message: string; field?: string }>
    }
    ```
- **Side effects**: Triggers router refresh to update order status

---

## Part B: Database Schema

**This service has no database.**

The `client` application is a frontend-only service that stores no data locally. All state is managed by backend microservices (auth, tickets, orders, payments) and accessed via API calls.

---

## Part C: Type Definitions & DTOs

### Domain Types

#### CurrentUser

- **File**: `apps/client/src/@types/user.ts` (line 1)
- **Definition**:
  ```typescript
  type CurrentUser = { id: string; email: string } | null;
  ```
- **Usage**: Represents the authenticated user state throughout the application
- **Where used**: 
  - `layout.tsx` (passed to Navbar)
  - `Navbar.tsx` (conditional rendering)
  - `AuthButtons.tsx` (show sign-in/sign-out)
  - `CheckoutForm.tsx` (Stripe email)
  - `OrderPage` (conditional checkout form)

---

#### CurrentUserResponse

- **File**: `apps/client/src/@types/user.ts` (line 3)
- **Definition**:
  ```typescript
  type CurrentUserResponse = {
    currentUser: CurrentUser;
  };
  ```
- **Usage**: Response shape from `/api/users/current-user`
- **Where used**: `get-current-user.ts` (line 3)

---

#### Ticket

- **File**: `apps/client/src/@types/ticket.ts` (line 1)
- **Definition**:
  ```typescript
  interface Ticket {
    id: string;
    title: string;
    price: number;
    userId: string;
    version: number;
    orderId?: string;  // Present if ticket is reserved
  }
  ```
- **Usage**: Represents a ticket for sale
- **Where used**: 
  - `page.tsx` (home page ticket list)
  - `[ticketId]/page.tsx` (ticket detail)
  - `TicketCard.tsx` (display component)

---

#### TicketsResponse

- **File**: `apps/client/src/@types/ticket.ts` (line 10)
- **Definition**:
  ```typescript
  interface TicketsResponse {
    tickets: Ticket[];
  }
  ```
- **Usage**: Response shape from `GET /api/tickets`
- **Where used**: `page.tsx` (line 4)

---

#### TicketResponse

- **File**: `apps/client/src/@types/ticket.ts` (line 14)
- **Definition**:
  ```typescript
  interface TicketResponse {
    ticket: Ticket;
  }
  ```
- **Usage**: Response shape from `GET /api/tickets/:ticketId`
- **Where used**: `[ticketId]/page.tsx` (line 8)

---

#### Order

- **File**: `apps/client/src/@types/order.ts` (line 3)
- **Definition**:
  ```typescript
  interface Order {
    id: string;
    userId: string;
    status: OrderStatus;
    expiresAt?: string;
    ticket: {
      id: string;
      price: number;
    };
    version: number;
  }
  ```
- **Usage**: Represents a ticket purchase order
- **Where used**:
  - `orders/page.tsx` (order list)
  - `[orderId]/page.tsx` (order detail)
  - `[orderId]/CheckoutForm.tsx` (payment)
  - `[ticketId]/page.tsx` (ticket reservation)
  - `OrderInfo.tsx` (order status display)

---

#### OrdersResponse

- **File**: `apps/client/src/@types/order.ts` (line 15)
- **Definition**:
  ```typescript
  interface OrdersResponse {
    orders: Order[];
  }
  ```
- **Usage**: Response shape from `GET /api/orders`
- **Where used**: `orders/page.tsx` (line 4)

---

#### OrderResponse

- **File**: `apps/client/src/@types/order.ts` (line 19)
- **Definition**:
  ```typescript
  interface OrderResponse {
    order: Order;
  }
  ```
- **Usage**: Response shape from `GET /api/orders/:orderId`
- **Where used**: 
  - `[orderId]/page.tsx` (line 7)
  - `[ticketId]/page.tsx` (line 8)

---

### Enums

#### OrderStatus

- **File**: `apps/client/src/constants/order.ts` (line 1)
- **Definition**:
  ```typescript
  enum OrderStatus {
    Created = 'created',
    Cancelled = 'cancelled',
    AwaitingPayment = 'awaiting-payment',
    Complete = 'complete',
  }
  ```
- **Values**:
  | Value | Business Meaning |
  |-------|------------------|
  | `created` | Order just created, payment window not yet active |
  | `cancelled` | Order was cancelled (ticket released) |
  | `awaiting-payment` | Order active, waiting for payment before expiration |
  | `complete` | Payment successful, ticket is purchased |
- **Where used**:
  - `orders/page.tsx` (conditional "Pay now" button)
  - `[orderId]/page.tsx` (conditional checkout form)
  - `OrderInfo.tsx` (status-specific messaging)

---

### HTTP Client Types

#### CommonResponse

- **File**: `apps/client/src/http/types.ts` (line 22)
- **Definition**:
  ```typescript
  type CommonSuccessResponse<T> = {
    status: number;
    ok: true;
    data: T;
    errors?: never;
  };

  type CommonErrorResponse = {
    status: number;
    ok: false;
    data?: never;
    errors: CommonError[];
  };

  type CommonResponse<T> = CommonSuccessResponse<T> | CommonErrorResponse;
  ```
- **Usage**: Discriminated union for all HTTP responses
- **Where used**: Return type for all `get()` and `post()` calls

---

#### CommonError

- **File**: `apps/client/src/http/types.ts` (line 17)
- **Definition**:
  ```typescript
  type CommonError = {
    message: string;
    field?: string;
  };
  ```
- **Usage**: Standardized error format from backend services
- **Where used**: 
  - `handleRequest.ts` (error parsing)
  - `toast-errors.ts` (error display)

---

#### RequestHandler

- **File**: `apps/client/src/http/types.ts` (line 24)
- **Definition**:
  ```typescript
  type RequestHandler = <T = unknown, D = unknown>(
    url: string,
    data?: D,
    config?: AxiosRequestConfig<D>,
  ) => Promise<CommonResponse<T>>;
  ```
- **Usage**: Type for `get()` and `post()` methods
- **Where used**: `getRequestHandlers.ts` (lines 7, 10)

---

### Component Props Interfaces

#### AuthForm Props

- **File**: `apps/client/src/app/auth/AuthForm.tsx` (line 11)
- **Definition**:
  ```typescript
  interface Props {
    action: 'signUp' | 'signIn';
  }
  ```

---

#### CheckoutForm Props

- **File**: `apps/client/src/app/orders/[orderId]/CheckoutForm.tsx` (line 12)
- **Definition**:
  ```typescript
  interface Props {
    order: Order;
    stripeKey: string;
    currentUser: NonNullable<CurrentUser>;
  }
  ```

---

#### PurchaseButton Props

- **File**: `apps/client/src/app/tickets/[ticketId]/PurchaseButton.tsx` (line 11)
- **Definition**:
  ```typescript
  interface Props {
    ticketId: string;
    disabled: boolean;
  }
  ```

---

#### OrderInfo Props

- **File**: `apps/client/src/app/tickets/[ticketId]/OrderInfo.tsx` (line 7)
- **Definition**:
  ```typescript
  interface Props {
    order: Order;
  }
  ```

---

#### AuthButtons Props

- **File**: `apps/client/src/components/navbar/AuthButtons.tsx` (line 11)
- **Definition**:
  ```typescript
  interface Props {
    currentUser: CurrentUser;
  }
  ```

---

#### Navbar Props

- **File**: `apps/client/src/components/navbar/Navbar.tsx` (line 7)
- **Definition**:
  ```typescript
  interface Props {
    currentUser: CurrentUser;
  }
  ```

---

#### TicketCard Props

- **File**: `apps/client/src/components/TicketCard.tsx` (line 4)
- **Definition**:
  ```typescript
  interface Props {
    ticket: Ticket;
  }
  ```

---

#### ExpirationTimer Props

- **File**: `apps/client/src/components/ExpirationTimer.tsx` (line 7)
- **Definition (inline)**:
  ```typescript
  {
    expiresAt: string;
    isRefreshEnabled?: boolean;  // default: false
  }
  ```

---

### Service Types

#### ServerService

- **File**: `apps/client/src/services/build-server-service.ts` (line 5)
- **Definition**:
  ```typescript
  type ServerService<T> = (headers: typeof headersFn) => Promise<T>;
  ```
- **Usage**: Generic type for cached server-side data fetching functions
- **Where used**: `buildServerService()` return type

---

## Summary

| Category | Count |
|----------|-------|
| Local API Routes | 1 |
| Backend API Calls | 10 |
| Domain Types | 7 |
| Enums | 1 |
| HTTP Types | 3 |
| Component Props | 8 |
| Service Types | 1 |

### Environment Variables

| Variable | File | Usage |
|----------|------|-------|
| `STRIPE_CLIENT_KEY` | `[orderId]/page.tsx` (line 22) | Stripe publishable key for checkout form |
