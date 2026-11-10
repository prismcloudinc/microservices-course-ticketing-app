---
service_id: ticketing-app/client
service_name: client
prompt_id: 04-business-logic
prompt_label: Business Logic Extraction
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.288825
schema_version: "2.0"
---

# Business Processes: client

## Process: User Sign Up

**Trigger**: User navigates to `/auth/sign-up` and submits the registration form

**Input**: 
- `email`: string (from form input named "email")
- `password`: string (from form input named "password")

**Preconditions**: None (page accessible to anyone)

**Steps**:
1. [`SignUpPage` in `src/app/auth/sign-up/page.tsx`] — Renders `AuthForm` component with `action="signUp"` prop
2. [`AuthForm` in `src/app/auth/AuthForm.tsx`] — Client component renders email/password form
3. [User submits form] → `handleSubmit` function (line 19) invoked
4. [`handleSubmit` in `src/app/auth/AuthForm.tsx`] — Prevents default form submission, extracts form data
   - Creates `FormData` from form element
   - Extracts `email` as string
   - Extracts `password` as string
5. [`buildClient` in `src/http/buildClient.ts`] — Creates axios client with `baseURL: '/'` for client-side requests
6. [`post` via `getRequestHandlers` in `src/http/getRequestHandlers.ts`] — Sends POST request to `/api/users/sign-up` with `{ email, password }`
7. [`handleRequest` in `src/http/handleRequest.ts`] — Executes axios request and normalizes response
   - If response successful: Returns `{ ok: true, status, data }` where data contains `{ user: { id, email } }`
   - If `AxiosError` thrown: Returns `{ ok: false, status, errors }` extracted from `err.response.data.errors`
8. [Back in `handleSubmit`] — Processes response:
   - If `res.ok === true`:
     - Calls `toast.success('Account created successfully!')` (sonner library)
     - Calls `router.push('/')` to navigate to home page
     - Calls `router.refresh()` to invalidate cached data
   - If `res.ok === false`:
     - Calls `toastErrors(res.errors)` to display each error

**Business rules**:
- The form requires both email and password fields (HTML `required` attribute)
- The endpoint `/api/users/sign-up` is served by the auth microservice (call target not statically resolvable — external service)
- Validation errors from the backend are displayed as individual toast notifications

**Output**: 
- On success: Toast notification "Account created successfully!", redirect to home page `/`
- On error: One or more toast error notifications, user remains on sign-up page

**Post-conditions**: 
- On success: User session cookie set by backend service (cookie handling not visible in client code)
- Navigation to home page triggers `RootLayout` to fetch current user on next render

**Error scenarios**:
- Network error: `AxiosError` caught, error response displayed via toast
- Validation errors (e.g., invalid email format, password too short): Backend returns `{ errors: [...] }`, each error displayed via `toastErrors` which iterates and calls `toast.error()` for each error with format `"Error in {field}: {message}"` if field present, else just `"{message}"`

---

## Process: User Sign In

**Trigger**: User navigates to `/auth/sign-in` and submits the authentication form

**Input**: 
- `email`: string (from form input named "email")
- `password`: string (from form input named "password")

**Preconditions**: None (page accessible to anyone)

**Steps**:
1. [`SignInPage` in `src/app/auth/sign-in/page.tsx`] — Renders `AuthForm` component with `action="signIn"` prop
2. [`AuthForm` in `src/app/auth/AuthForm.tsx`] — Client component renders email/password form
3. [User submits form] → `handleSubmit` function (line 19) invoked
4. [`handleSubmit` in `src/app/auth/AuthForm.tsx`] — Prevents default form submission, extracts form data
   - Creates `FormData` from form element
   - Extracts `email` as string
   - Extracts `password` as string
5. [`buildClient` in `src/http/buildClient.ts`] — Creates axios client with `baseURL: '/'`
6. [`post` via `getRequestHandlers`] — Sends POST request to `/api/users/sign-in` with `{ email, password }`
7. [`handleRequest` in `src/http/handleRequest.ts`] — Executes request and normalizes response
8. [Back in `handleSubmit`] — Processes response:
   - If `res.ok === true`:
     - Calls `toast.success('Signed in successfully!')`
     - Calls `router.push('/')` to navigate to home page
     - Calls `router.refresh()` to invalidate cached data
   - If `res.ok === false`:
     - Calls `toastErrors(res.errors)` to display each error

**Business rules**:
- The form requires both email and password fields (HTML `required` attribute)
- The endpoint `/api/users/sign-in` is served by the auth microservice (call target not statically resolvable — external service)

**Output**: 
- On success: Toast notification "Signed in successfully!", redirect to home page `/`
- On error: One or more toast error notifications

**Post-conditions**: 
- On success: User session cookie set by backend service
- Navigation to home page triggers `RootLayout` to fetch current user, which will now return the authenticated user

**Error scenarios**:
- Invalid credentials: Backend returns errors array, displayed via toast
- Network error: `AxiosError` caught, error response displayed

---

## Process: User Sign Out

**Trigger**: Authenticated user clicks "Sign out" button in navigation bar

**Input**: None (no request body)

**Preconditions**: 
- User must be authenticated (button only renders when `currentUser` is truthy in `AuthButtons` component, line 34)

**Steps**:
1. [`AuthButtons` in `src/components/navbar/AuthButtons.tsx`] — Renders "Sign out" button when `currentUser` is truthy
2. [User clicks button] → `handleSignOut` function (line 19) invoked
3. [`buildClient` in `src/http/buildClient.ts`] — Creates axios client with `baseURL: '/'`
4. [`post` via `getRequestHandlers`] — Sends POST request to `/api/users/sign-out` with no body
5. [`handleRequest` in `src/http/handleRequest.ts`] — Executes request
6. [Back in `handleSignOut`] — Processes response:
   - In `.then()` block (regardless of response content):
     - Calls `toast.success('Signed out successfully!')`
     - Calls `router.push('/')` to navigate to home page
     - Calls `router.refresh()` to invalidate cached data
   - In `.catch()` block:
     - Calls `toast.error('Failed to logout')`

**Business rules**:
- Sign out is treated as successful regardless of the actual response status (the `.then()` handler runs for any non-error response)
- The endpoint `/api/users/sign-out` is served by the auth microservice (call target not statically resolvable — external service)

**Output**: 
- On success: Toast notification "Signed out successfully!", redirect to home page `/`
- On error: Toast notification "Failed to logout"

**Post-conditions**: 
- Session cookie cleared by backend service
- Navigation to home page triggers `RootLayout` to fetch current user, which will now return null

**Error scenarios**:
- Network error or any exception: `.catch()` handler displays "Failed to logout" toast

---

## Process: Fetch Current User (Server-Side)

**Trigger**: Every page request to the application (via `RootLayout`)

**Input**: HTTP request headers (specifically the `cookie` header)

**Preconditions**: None

**Steps**:
1. [`RootLayout` in `src/app/layout.tsx`] — Server component that wraps all pages
2. [Line 23] — Calls `await getCurrentUser(headers)`
3. [`getCurrentUser` in `src/services/get-current-user.ts`] — Wrapped with `buildServerService` for caching
4. [`buildServerService` in `src/services/build-server-service.ts`] — 
   - Checks if running on server (throws error if `typeof window !== 'undefined'`)
   - Wraps callback with React's `cache()` function for request deduplication
5. [Back in `getCurrentUser`] — Executes the cached service function:
   - Calls `buildClientServer(headers)` to create server-side HTTP client
6. [`buildClientServer` in `src/http/buildClientServer.ts`] — Creates axios client for server-to-service communication:
   - Sets `baseURL: 'http://ingress-nginx-controller.ingress-nginx.svc.cluster.local'` (Kubernetes internal service URL)
   - Sets `host: 'ticketing.dev'` header (line 10)
   - Forwards `cookie` header from incoming request if present (lines 12-13)
7. [`get` via `getRequestHandlers`] — Sends GET request to `/api/users/current-user`
8. [`handleRequest` in `src/http/handleRequest.ts`] — Executes request
9. [Back in `getCurrentUser`] — Processes response:
   - If `res.ok === true`: Returns `res.data.currentUser` (type: `{ id: string; email: string }`)
   - If `res.ok === false`: Returns `null`
10. [Back in `RootLayout`] — Passes `currentUser` to `Navbar` component

**Business rules**:
- The current user is fetched on every page load (cached per request via React's `cache()`)
- If the user is not authenticated, `currentUser` will be `null`
- The server-side client forwards cookies to preserve session state across internal service calls
- The `host: 'ticketing.dev'` header is required for the ingress controller to route the request correctly

**Output**: 
- `currentUser`: `{ id: string; email: string }` if authenticated, `null` if not

**Post-conditions**: None (read-only operation)

**Error scenarios**:
- If `/api/users/current-user` returns an error response, the function returns `null` (user treated as unauthenticated)
- If called on client-side: Throws `Error('This function should only be called on the server')` (line 9 in `build-server-service.ts`)

---

## Process: Browse Tickets (Home Page)

**Trigger**: User navigates to `/` (home page)

**Input**: HTTP request headers (for authentication context)

**Preconditions**: None (page accessible to anyone)

**Steps**:
1. [`Home` in `src/app/page.tsx`] — Server component
2. [Line 9] — Calls `buildClientServer(headers)` to create server-side HTTP client
3. [`buildClientServer` in `src/http/buildClientServer.ts`] — Creates axios client with Kubernetes internal URL and cookie forwarding
4. [Line 10-12] — Sends GET request to `/api/tickets`:
   - If `res.ok === true` (response has data): Returns `res.data.tickets` (array of `Ticket` objects)
   - If `res.ok === false` or `res.data` is undefined: Returns empty array `[]` (via `??` operator)
5. [Lines 24-31] — Renders ticket list:
   - Maps over `tickets` array
   - For each ticket, renders a `Link` to `/tickets/${ticket.id}` containing a `TicketCard`

**Business rules**:
- All tickets are displayed regardless of authentication status
- Tickets are rendered in the order returned by the API (no client-side sorting)
- Each ticket links to its detail page
- If the API request fails, an empty list is shown (no error message displayed)

**Output**: 
- HTML page with:
  - "Tickets" heading
  - "Create ticket" button (links to `/tickets/new`)
  - Grid of ticket cards, each showing title and formatted price

**Post-conditions**: None (read-only operation)

**Error scenarios**:
- API error: Empty array returned, page renders with no tickets listed
- No explicit error handling or user notification for failed ticket fetch

---

## Process: View Ticket Details

**Trigger**: User navigates to `/tickets/:ticketId`

**Input**: 
- `ticketId`: string (from URL path parameter)
- HTTP request headers (for authentication context)

**Preconditions**: Ticket with given ID must exist

**Steps**:
1. [`TicketPage` in `src/app/tickets/[ticketId]/page.tsx`] — Server component with dynamic route
2. [Line 18] — Calls `buildClientServer(headers)` to create server-side HTTP client
3. [Line 20] — Initializes `order` variable as `null`
4. [Lines 22-36] — Fetches ticket from `/api/tickets/${ticketId}`:
   - If `res.ok === true`:
     - Extracts `ticket` from response data
     - If `ticket.orderId` is truthy (ticket is reserved):
       - Fetches order from `/api/orders/${ticket.orderId}`
       - If order fetch succeeds: Sets `order` to `res.data.order`
       - If order fetch fails: `order` remains `null`
     - Returns `ticket`
   - If `res.ok === false`:
     - Calls `notFound()` from `next/navigation` (returns 404 page)
5. [Lines 38-64] — Renders ticket detail page:
   - Renders ticket title and price
   - Renders `PurchaseButton` component with `disabled={!!ticket.orderId}` (disabled if already reserved)
   - If ticket has no `orderId`: Shows "Reserve your seat now only for {price}"
   - If ticket has `orderId` but `order` is null (inferred: user doesn't own the order): Shows "This ticket is already reserved" with link to home
   - If `order` exists: Renders `OrderInfo` component

**Business rules**:
- A ticket is considered "reserved" if `ticket.orderId` is truthy
- The purchase button is disabled when a ticket is already reserved
- If the ticket has an order but the order fetch fails (inferred: possibly due to authorization), user sees "This ticket is already reserved"
- Order information is only displayed if the order fetch succeeds

**Output**: 
- HTML page with ticket details, reservation status, and purchase/order options

**Post-conditions**: None (read-only operation)

**Error scenarios**:
- Ticket not found: `notFound()` called, Next.js renders 404 page
- Order fetch fails for reserved ticket: Page renders with "This ticket is already reserved" message

---

## Process: Create Ticket

**Trigger**: User navigates to `/tickets/new` and submits the ticket creation form

**Input**: 
- `title`: string (from form input named "title")
- `price`: number (from form input named "price", parsed and formatted)

**Preconditions**: 
- User should be authenticated (inferred — not enforced in client code, backend enforces)

**Steps**:
1. [`NewTicketPage` in `src/app/tickets/new/page.tsx`] — Client component with `'use client'` directive
2. [User fills form] — Two input fields:
   - Title: text input, required
   - Price: text input, required, with `onBlur` handler for validation/formatting
3. [Price field blur handler, lines 65-77]:
   - If input is empty: Returns early (no validation)
   - Parses input as float, rounds to 2 decimal places using `Math.round(parseFloat(value) * 100) / 100`
   - If `isNaN(userInput)`: Throws error with message "Invalid price value"
   - Sets input value to `userInput.toFixed(2)` (formatted string)
   - In catch block: Calls `toast.error('Invalid price value')`
4. [User submits form] → `handleSubmit` function (line 15) invoked
5. [`handleSubmit` in `src/app/tickets/new/page.tsx`] — Prevents default, extracts data:
   - Extracts `title` as string
   - Parses `price`: `+parseFloat(formData.get('price')).toFixed(2)` (converts to number with 2 decimal precision)
6. [`buildClient` in `src/http/buildClient.ts`] — Creates client-side HTTP client
7. [`post` via `getRequestHandlers`] — Sends POST request to `/api/tickets` with `{ title, price }`
8. [`handleRequest` in `src/http/handleRequest.ts`] — Executes request
9. [Back in `handleSubmit`] — Processes response:
   - If `res.ok === true`:
     - Calls `toast.success('Ticket created successfully!')`
     - Calls `router.push('/')` to navigate to home page
     - Calls `router.refresh()` to invalidate cached data
   - If `res.ok === false`:
     - Calls `toastErrors(res.errors)` to display each error

**Business rules**:
- Price is validated and formatted on blur to 2 decimal places
- Price validation uses `isNaN()` check — non-numeric input shows error toast
- The form requires both title and price fields (HTML `required` attribute)
- The endpoint `/api/tickets` is served by the tickets microservice (call target not statically resolvable — external service)

**Output**: 
- On success: Toast notification "Ticket created successfully!", redirect to home page `/`
- On error: One or more toast error notifications

**Post-conditions**: 
- On success: New ticket created in backend database
- Home page will show the new ticket after refresh

**Error scenarios**:
- Invalid price format (on blur): Toast error "Invalid price value"
- Backend validation errors: Displayed via `toastErrors`
- Unauthenticated user (inferred): Backend would return 401 error, displayed via toast

---

## Process: Purchase Ticket (Create Order)

**Trigger**: User clicks "Purchase" button on a ticket detail page

**Input**: 
- `ticketId`: string (passed as prop to `PurchaseButton` component)

**Preconditions**: 
- Ticket must not already be reserved (`ticket.orderId` must be falsy for button to be enabled)
- User should be authenticated (inferred — not enforced in client code, backend enforces)

**Steps**:
1. [`PurchaseButton` in `src/app/tickets/[ticketId]/PurchaseButton.tsx`] — Client component
2. [Button rendered with `disabled={!!ticket.orderId}`] — Disabled if ticket already reserved
3. [User clicks button] → `handlePurchase` function (line 19) invoked
4. [`buildClient` in `src/http/buildClient.ts`] — Creates client-side HTTP client
5. [`post` via `getRequestHandlers`] — Sends POST request to `/api/orders` with `{ ticketId }`
6. [`handleRequest` in `src/http/handleRequest.ts`] — Executes request
7. [Back in `handlePurchase`] — Processes response:
   - If `res.ok === true`:
     - Calls `toast.success('Order created successfully')`
     - Calls `router.refresh()` to reload the page with updated data
   - If `res.ok === false`:
     - Calls `toastErrors(res.errors)` to display each error

**Business rules**:
- The purchase button is disabled when `disabled` prop is true (ticket already reserved)
- Creating an order reserves the ticket (inferred from `ticket.orderId` being set after order creation)
- The endpoint `/api/orders` is served by the orders microservice (call target not statically resolvable — external service)

**Output**: 
- On success: Toast notification "Order created successfully", page refreshes to show order info
- On error: One or more toast error notifications

**Post-conditions**: 
- On success: Order created in backend, ticket marked as reserved
- Page refresh will show `OrderInfo` component with order details

**Error scenarios**:
- Ticket already reserved: Backend returns error, displayed via toast
- Unauthenticated user: Backend would return error, displayed via toast

---

## Process: View Orders List

**Trigger**: User navigates to `/orders`

**Input**: HTTP request headers (for authentication context)

**Preconditions**: 
- User should be authenticated (orders link only shows in navbar when `currentUser` is truthy — see `Navbar.tsx` line 18)

**Steps**:
1. [`OrdersPage` in `src/app/orders/page.tsx`] — Server component
2. [Line 10] — Calls `buildClientServer(headers)` to create server-side HTTP client
3. [Lines 11-15] — Fetches orders from `/api/orders`:
   - If `res.ok === true`: Returns `res.data.orders.toReversed()` (array reversed for newest-first display)
   - If `res.ok === false`: Returns empty array `[]`
4. [Lines 17-59] — Renders orders list:
   - If `orders.length > 0`:
     - Maps over orders, renders each with:
       - Order ID
       - Ticket ID (as link to `/tickets/${order.ticket.id}`)
       - Price: `${order.ticket.price}`
       - Status: `{order.status}`
       - If `order.status === OrderStatus.AwaitingPayment`:
         - Renders `ExpirationTimer` if `order.expiresAt` exists
         - Renders "Pay now" button linking to `/orders/${order.id}`
   - If `orders.length === 0`:
     - Shows "No orders found."
     - Renders "Find some tickets" button linking to home

**Business rules**:
- Orders are displayed in reverse order (newest first) using `.toReversed()`
- Only orders with status `AwaitingPayment` show the expiration timer and "Pay now" button
- Order status is displayed as the raw enum value (e.g., "created", "awaiting-payment", "complete", "cancelled")
- Price is displayed with `$` prefix directly (not using `formatPrice` utility)

**Output**: 
- HTML page with list of user's orders, or "No orders found" message

**Post-conditions**: None (read-only operation)

**Error scenarios**:
- API error: Empty array returned, "No orders found" message displayed
- Unauthenticated user: Backend would return error, empty array displayed

---

## Process: View Order Details

**Trigger**: User navigates to `/orders/:orderId`

**Input**: 
- `orderId`: string (from URL path parameter)
- HTTP request headers (for authentication context)

**Preconditions**: 
- Order with given ID must exist
- User should own the order (inferred — backend enforces)

**Steps**:
1. [`OrderPage` in `src/app/orders/[orderId]/page.tsx`] — Server component
2. [Line 14] — Calls `await getCurrentUser(headers)` to get current user
3. [Line 15] — Calls `buildClientServer(headers)` to create server-side HTTP client
4. [Lines 16-20] — Fetches order from `/api/orders/${orderId}`:
   - If `res.ok === true`: Returns `res.data.order`
   - If `res.ok === false`: Calls `notFound()` (returns 404 page)
5. [Line 22] — Gets Stripe public key from `process.env.STRIPE_CLIENT_KEY ?? ''`
6. [Lines 24-46] — Renders order detail page:
   - Renders order heading with ID
   - Renders ticket ID (as link to `/tickets/${order.ticket.id}`)
   - Renders price: `{order.ticket.price}`
   - Renders status: `{order.status}`
   - If `currentUser` is truthy AND `order.status === OrderStatus.AwaitingPayment`:
     - Renders `ExpirationTimer` if `order.expiresAt` exists (without refresh enabled)
     - Renders `CheckoutForm` with order, stripeKey, and currentUser props

**Business rules**:
- The checkout form only renders if:
  - User is authenticated (`currentUser` is truthy)
  - Order status is `AwaitingPayment`
- The Stripe key comes from environment variable `STRIPE_CLIENT_KEY`, defaults to empty string if not set
- If `stripeKey` is empty, the `CheckoutForm` component will render `null` (see `CheckoutForm.tsx` line 34)

**Output**: 
- HTML page with order details and Stripe checkout form (if applicable)

**Post-conditions**: None (read-only operation)

**Error scenarios**:
- Order not found: `notFound()` called, Next.js renders 404 page
- Order belongs to different user: Backend returns error, `notFound()` triggered

---

## Process: Process Payment (Stripe Checkout)

**Trigger**: User completes Stripe checkout form on order detail page

**Input**: 
- `token`: Stripe token object (from `react-stripe-checkout` component)
- `orderId`: string (from order passed to `CheckoutForm`)

**Preconditions**: 
- User must be authenticated
- Order status must be `AwaitingPayment`
- Valid `stripeKey` environment variable must be set

**Steps**:
1. [`CheckoutForm` in `src/app/orders/[orderId]/CheckoutForm.tsx`] — Client component
2. [Lines 34-41] — Renders `StripeCheckout` component (from `react-stripe-checkout`):
   - `token={handleToken}` — callback when user completes checkout
   - `stripeKey={stripeKey}` — Stripe publishable key
   - `amount={order.ticket.price * 100}` — amount in cents
   - `email={currentUser.email}` — pre-fills user email
   - If `stripeKey` is falsy: Renders `null` (no checkout button)
3. [User completes Stripe checkout] → `handleToken` callback (line 22) invoked with token object
4. [`buildClient` in `src/http/buildClient.ts`] — Creates client-side HTTP client
5. [`post` via `getRequestHandlers`] — Sends POST request to `/api/payments` with `{ token: token.id, orderId: order.id }`
6. [`handleRequest` in `src/http/handleRequest.ts`] — Executes request
7. [Back in `handleToken`] — Processes response:
   - If `res.ok === true`:
     - Calls `toast.success('Payment successful')`
     - Calls `router.refresh()` to reload page with updated order status
   - If `res.ok === false`:
     - Calls `toastErrors(res.errors)` to display each error

**Business rules**:
- The amount sent to Stripe is the ticket price multiplied by 100 (converts dollars to cents)
- The token ID (not the full token object) is sent to the backend
- The endpoint `/api/payments` is served by the payments microservice (call target not statically resolvable — external service)

**Output**: 
- On success: Toast notification "Payment successful", page refreshes to show updated order status
- On error: One or more toast error notifications

**Post-conditions**: 
- On success: Payment recorded in backend, order status changes (inferred: to `Complete`)
- Page refresh will show updated order status

**Error scenarios**:
- Stripe token creation fails: Handled by `react-stripe-checkout` component (not visible in this code)
- Backend payment processing fails: Error response displayed via `toastErrors`
- Order expired: Backend would return error, displayed via toast

---

## Process: Expiration Timer Display

**Trigger**: Component mounts on pages showing order with expiration time

**Input**: 
- `expiresAt`: string (ISO timestamp of expiration)
- `isRefreshEnabled`: boolean (optional, defaults to false)

**Preconditions**: `expiresAt` must be a valid date string

**Steps**:
1. [`ExpirationTimer` in `src/components/ExpirationTimer.tsx`] — Client component
2. [Line 15] — Initializes `timeLeft` state as `undefined`
3. [Lines 17-41] — Sets up interval on mount via `useEffect`:
   - Defines `updateTimer` function:
     - Calculates `newTimeLeft = Math.round((new Date(expiresAt).getTime() - new Date().getTime()) / 1000)` (seconds until expiration)
     - If `newTimeLeft <= 0`:
       - Clears interval
       - Sets `timeLeft` to `0`
       - If `isRefreshEnabled === true`: Sets 1-second timeout then calls `router.refresh()`
     - Else: Sets `timeLeft` to `newTimeLeft`
   - Creates interval that calls `updateTimer` every 1000ms
   - Calls `updateTimer` immediately
   - Returns cleanup function that clears interval
4. [Lines 43-57] — Renders based on state:
   - If `expiresAt` is falsy: Returns `null`
   - If `timeLeft` is `undefined`: Returns "Loading..." (italic)
   - If `timeLeft > 0`: Returns "Expires in **{timeLeft}** seconds"
   - If `timeLeft === 0`: Returns "Expired" (italic)

**Business rules**:
- Timer updates every 1 second
- Time left is calculated in seconds, rounded using `Math.round()`
- When timer reaches 0 and `isRefreshEnabled` is true, the page refreshes after a 1-second delay (inferred: to show updated order status as expired/cancelled)
- The timer only renders when `expiresAt` is provided

**Output**: 
- Visual countdown showing seconds until expiration
- Optional page refresh on expiration

**Post-conditions**: 
- If `isRefreshEnabled` and timer expires: Page data refreshes, showing updated order status

**Error scenarios**: None explicitly handled — invalid `expiresAt` would cause `NaN` display

---

## Process: Health Check

**Trigger**: HTTP GET request to `/health/check` (typically from Kubernetes liveness probe)

**Input**: None

**Preconditions**: None

**Steps**:
1. [`GET` handler in `src/app/health/check/route.ts`] — Next.js Route Handler
2. [Line 3] — Sets `export const dynamic = 'force-dynamic'` to disable caching
3. [Lines 5-10] — Returns `NextResponse.json({}, { status: 200 })`

**Business rules**:
- Response is always empty object `{}` with status `200`
- Route is forced to be dynamic (not cached) via `force-dynamic` export

**Output**: 
- HTTP Response: `{}` with status code `200`

**Post-conditions**: None

**Error scenarios**: None — always returns 200 OK

---

## Process: Display Order Status Information

**Trigger**: `OrderInfo` component rendered on ticket detail page when order exists

**Input**: 
- `order`: Order object with `status`, `id`, `expiresAt`, `ticket` properties

**Preconditions**: Order must exist and be accessible to the current user

**Steps**:
1. [`OrderInfo` in `src/app/tickets/[ticketId]/OrderInfo.tsx`] — Server component
2. [Line 12-38] — Renders based on `order.status`:
   - If `order.status === OrderStatus.Created` (value: `'created'`):
     - Renders "Please refresh the page to see the payment options."
   - If `order.status === OrderStatus.AwaitingPayment` (value: `'awaiting-payment'`):
     - Renders "Your order (#{order.id}) is awaiting payment."
     - If `order.expiresAt` exists: Renders `ExpirationTimer` with `isRefreshEnabled={true}`
     - Renders "Pay now" button linking to `/orders/${order.id}`
   - If `order.status === OrderStatus.Complete` (value: `'complete'`):
     - Renders "Your order is complete." and "Enjoy the event!"
   - If `order.status === OrderStatus.Cancelled` (value: `'cancelled'`): No specific rendering (falls through to nothing)

**Business rules**:
- Status `Created` instructs user to refresh (inferred: order processing may be async, awaiting transition to `AwaitingPayment`)
- Status `AwaitingPayment` shows expiration timer with auto-refresh enabled
- Status `Complete` shows success message
- Status `Cancelled` has no explicit handling — would show only the "Great choice!" heading

**Output**: 
- Status-specific message and actions for the order

**Post-conditions**: None (read-only/display operation)

**Error scenarios**: None explicitly handled

---

## Summary of API Endpoints Called

| Endpoint | Method | Called From | Purpose |
|----------|--------|-------------|---------|
| `/api/users/sign-up` | POST | `AuthForm.tsx` | User registration |
| `/api/users/sign-in` | POST | `AuthForm.tsx` | User authentication |
| `/api/users/sign-out` | POST | `AuthButtons.tsx` | User logout |
| `/api/users/current-user` | GET | `get-current-user.ts` | Get authenticated user |
| `/api/tickets` | GET | `page.tsx` (home) | List all tickets |
| `/api/tickets` | POST | `new/page.tsx` | Create new ticket |
| `/api/tickets/:ticketId` | GET | `[ticketId]/page.tsx` | Get ticket details |
| `/api/orders` | GET | `orders/page.tsx` | List user's orders |
| `/api/orders` | POST | `PurchaseButton.tsx` | Create order (reserve ticket) |
| `/api/orders/:orderId` | GET | `[ticketId]/page.tsx`, `[orderId]/page.tsx` | Get order details |
| `/api/payments` | POST | `CheckoutForm.tsx` | Process Stripe payment |

All API endpoints are served by backend microservices through the Kubernetes ingress controller (call targets not statically resolvable — external services).
