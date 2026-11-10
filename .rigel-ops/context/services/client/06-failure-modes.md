---
service_id: ticketing-app/client
service_name: client
prompt_id: 06-failure-modes
prompt_label: Failure Modes & Edge Cases
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.21476000000000003
schema_version: "2.0"
---

# Error Handling & Failure Mode Analysis: client

## Part A: Error Handling Architecture

### Global Error Handler

**Status: No global error handler exists**

The client application lacks:
- **Error boundaries**: No `error.tsx` or `global-error.tsx` files exist in the `src/app/` directory
- **Loading states**: No `loading.tsx` files exist for suspense fallbacks
- **Custom 404 page**: No `not-found.tsx` file exists
- **Next.js middleware**: No `middleware.ts` file exists for request interception

Error handling is implemented at the component level through the HTTP client wrapper.

### Error Transformation

The HTTP layer (`src/http/handleRequest.ts` lines 5-41) transforms errors:

```typescript
// Line 23-39
catch (err) {
  if (err instanceof AxiosError) {
    return Promise.resolve({
      ok: false,
      status: err.response?.status ?? 500,
      errors: (
        err.response?.data as {
          errors: {
            message: string;
            field?: string;
          }[];
        }
      ).errors,
    });
  }
  return Promise.reject(err);
}
```

**Error types caught:**
- `AxiosError` instances are normalized to `CommonResponse` format
- Status defaults to 500 if no response status exists
- Non-Axios errors are re-thrown (unhandled)

**Critical Gap**: The code assumes `err.response?.data.errors` always exists. Network errors, timeouts, or malformed API responses will cause a runtime crash when accessing `.errors` property on undefined (`handleRequest.ts` line 35).

### Logging

| Aspect | Status |
|--------|--------|
| Logging library | **None** — no logging library is configured |
| Console statements | **None** — no `console.log/error/warn` calls in source code |
| Log levels | **Not applicable** |
| Structured fields | **Not applicable** |
| External error tracking | **None** — no Sentry, Datadog, or similar integration observed |

### User-Facing Error Display

Uses `sonner` toast library (`src/utils/toast-errors.ts` lines 1-9):

```typescript
export const toastErrors = (errors: CommonError[]) => {
  for (const error of errors) {
    toast.error(`${error.field ? `Error in ${error.field}:<br />` : ''}${error.message}`);
  }
};
```

**Configuration** (`layout.tsx` line 30): `<Toaster visibleToasts={5} pauseWhenPageIsHidden />`

---

## Part B: Integration Point Failure Behaviors

### Backend API (via Ingress Controller)

The client communicates with backend microservices through the Kubernetes ingress controller.

#### Client-Side Requests (`src/http/buildClient.ts`)

| Aspect | Value |
|--------|-------|
| HTTP client | Axios v1.6.8 |
| Base URL | `/` (relative, proxied through ingress) |
| Timeout | **Not configured** — uses Axios default (no timeout) |
| Retry policy | **None** |
| Circuit breaker | **None** |

#### Server-Side Requests (`src/http/buildClientServer.ts`)

| Aspect | Value |
|--------|-------|
| HTTP client | Axios v1.6.8 |
| Base URL | `http://ingress-nginx-controller.ingress-nginx.svc.cluster.local` (hardcoded, line 16) |
| Host header | `ticketing.dev` (hardcoded, line 10) |
| Cookie forwarding | Yes — forwards `cookie` header from request (`line 12-13`) |
| Timeout | **Not configured** |
| Retry policy | **None** |
| Circuit breaker | **None** |

#### Failure Detection & Response

**Server-side page data fetching:**

| Page | File | Error Handling |
|------|------|----------------|
| Home | `src/app/page.tsx` line 11-12 | Returns empty array `[]` on any error — **silently fails** |
| Orders list | `src/app/orders/page.tsx` lines 11-15 | Returns empty array `[]` on any error — **silently fails** |
| Order detail | `src/app/orders/[orderId]/page.tsx` lines 16-20 | Calls `notFound()` on any error |
| Ticket detail | `src/app/tickets/[ticketId]/page.tsx` lines 22-36 | Calls `notFound()` on any error |
| Current user | `src/services/get-current-user.ts` lines 8-10 | Returns `null` on error — **silently fails** |

**Client-side mutations:**

| Action | File | Error Handling |
|--------|------|----------------|
| Sign in/up | `src/app/auth/AuthForm.tsx` lines 37-47 | `toastErrors()` on error |
| Create ticket | `src/app/tickets/new/page.tsx` lines 27-35 | `toastErrors()` on error |
| Purchase ticket | `src/app/tickets/[ticketId]/PurchaseButton.tsx` lines 20-27 | `toastErrors()` on error |
| Submit payment | `src/app/orders/[orderId]/CheckoutForm.tsx` lines 23-31 | `toastErrors()` on error |
| Sign out | `src/components/navbar/AuthButtons.tsx` lines 19-29 | Uses `.catch()` — **inconsistent pattern** |

**Fallback behavior**: No degraded mode, cached fallbacks, or graceful degradation exists. Failed API calls either show toasts (client-side) or silently return empty/null (server-side).

**Alerting**: Not applicable — this is a frontend application with no alerting infrastructure.

**Dead letter queue**: Not applicable — no message processing.

**Recovery**: Page refresh required to retry failed requests.

---

### Stripe Integration

| Aspect | Details |
|--------|---------|
| Library | `react-stripe-checkout` v2.6.3 |
| API key source | `process.env.STRIPE_CLIENT_KEY` (`src/app/orders/[orderId]/page.tsx` line 22) |
| Fallback | Falls back to empty string `''` if env var missing |
| Rendering | Component renders `null` if `stripeKey` is falsy (`CheckoutForm.tsx` lines 34-42) |

#### Failure Detection

| Failure Type | Detection | Response |
|--------------|-----------|----------|
| Missing Stripe key | Checked via `stripeKey ?` conditional | Checkout button not rendered — **silent failure, no user feedback** |
| Payment API error | POST to `/api/payments` fails | `toastErrors()` shows API error messages |
| Stripe JS load failure | Not explicitly handled | Unknown — relies on library defaults |
| Token generation failure | Not explicitly handled | Unknown — relies on library defaults |

**Gap**: No explicit handling for Stripe JavaScript SDK load failures or `react-stripe-checkout` token generation errors.

---

## Part C: Data Consistency Edge Cases

### Crash Mid-Operation

| Scenario | Behavior |
|----------|----------|
| Browser closes during form submission | Request may complete server-side; client state lost — **no recovery mechanism** |
| Server-side render fails mid-fetch | Next.js error page shown (default 500) — **no custom error boundary** |
| Network disconnect during mutation | `AxiosError` caught, toast shown, but **no offline support or retry queue** |

**No transaction management**: All API calls are fire-and-forget from the client perspective.

### Duplicate Message Processing

**Not directly applicable** — the client does not process messages. However:

| Scenario | Risk |
|----------|------|
| User double-clicks submit button | Multiple requests sent — **no debouncing or button disable during submission** |
| User refreshes after form submit | Form resubmission possible — **no POST-redirect-GET pattern for mutations** |

**Observation**: Client components use `router.push('/')` and `router.refresh()` after mutations, but there's no explicit loading state to prevent double submissions.

### Concurrent Modifications

| Scenario | Behavior |
|----------|----------|
| Two tabs editing same ticket | Last write wins — **no optimistic locking check client-side** |
| Ticket reserved while user viewing | User sees stale "available" state until page refresh |
| Order expires while user on checkout page | `ExpirationTimer` component triggers `router.refresh()` after expiry — **handles this case** |

**ExpirationTimer behavior** (`src/components/ExpirationTimer.tsx` lines 17-41):
- Polls client-side timer every 1 second
- When `timeLeft <= 0`, optionally triggers `router.refresh()` after 1 second delay
- Relies on server-side order status for actual expiration enforcement

### Stale Data Handling

| Page | Cache Behavior |
|------|----------------|
| Server-side fetches | Uses React's `cache()` for request deduplication within single render (`build-server-service.ts` line 12) |
| Client-side data | No caching — fresh request each time |
| Route cache | Next.js default behavior; `force-dynamic` only set on `/health/check` route |

**Gap**: No explicit cache invalidation strategy. Users may see stale data until they manually refresh or navigate away and back.

---

## Part D: Known Gotchas

### TODO/FIXME/HACK/WORKAROUND Comments

**None found** — grep search returned no matches.

### Hardcoded Values That Should Be Configurable

| File | Line | Value | Issue |
|------|------|-------|-------|
| `src/http/buildClientServer.ts` | 10 | `'ticketing.dev'` | Hardcoded host header — not configurable for different environments |
| `src/http/buildClientServer.ts` | 16 | `'http://ingress-nginx-controller.ingress-nginx.svc.cluster.local'` | Hardcoded K8s service URL — assumes specific namespace and service name |
| `src/app/layout.tsx` | 30 | `visibleToasts={5}` | Toast limit hardcoded — cosmetic but not user-configurable |
| `src/components/ExpirationTimer.tsx` | 26-28 | `setTimeout(..., 1000)` | 1 second delay before refresh hardcoded |

### Timeout Values

| Component | Timeout | Assessment |
|-----------|---------|------------|
| Axios client (client-side) | **Not set** | Uses Axios default (unlimited) — **risky for hung connections** |
| Axios client (server-side) | **Not set** | Uses Axios default (unlimited) — **risky for hung connections** |
| `ExpirationTimer` interval | 1000ms | Appropriate for countdown display |
| `ExpirationTimer` refresh delay | 1000ms | Appropriate for post-expiry refresh |

### Rate Limits

**No rate limiting observed**:
- No client-side request throttling
- No handling of 429 (Too Many Requests) responses
- Backend rate limits (if any) would surface as standard API errors via `toastErrors()`

### Data Size Limits

| Type | Limit | Location |
|------|-------|----------|
| Request body | **Not configured** | Uses Next.js/Axios defaults |
| File uploads | **Not applicable** | No file upload functionality observed |
| Toast messages | No truncation | `toast-errors.ts` displays full message — long errors may overflow |

### Critical Code Issues

#### 1. Unsafe Error Response Access (`src/http/handleRequest.ts` lines 28-35)

```typescript
errors: (
  err.response?.data as {
    errors: { message: string; field?: string }[];
  }
).errors,
```

**Problem**: If `err.response` is undefined (network error, DNS failure, timeout) or `err.response.data` doesn't have an `errors` property, this will throw `TypeError: Cannot read properties of undefined`.

**Impact**: Network errors or malformed API responses crash instead of being handled gracefully.

#### 2. Inconsistent Error Handling Pattern (`src/components/navbar/AuthButtons.tsx` lines 27-28)

```typescript
.catch(() => {
  toast.error('Failed to logout');
});
```

**Problem**: Uses `.catch()` instead of checking `res.ok` like all other client mutations. This catches thrown errors (e.g., non-AxiosError from `handleRequest.ts` line 39), but ignores the `CommonResponse` error format.

**Impact**: Logout failures show generic message instead of specific API error details.

#### 3. Missing Loading States

**No loading indicators** exist for:
- Server-side page data fetching (no `loading.tsx` files)
- Client-side form submissions (buttons not disabled during request)
- Initial page hydration

**Impact**: Users may interact with stale UI or trigger duplicate submissions.

#### 4. Silent Server-Side Failures

Home page (`src/app/page.tsx` line 11-12):
```typescript
const tickets = await client
  .get<TicketsResponse>('/api/tickets')
  .then((res) => res.data?.tickets ?? []);
```

**Problem**: API failures return empty array with no user notification.

**Impact**: Users see empty ticket list with no indication that data failed to load.

#### 5. ESLint Disable Comment (`src/components/ExpirationTimer.tsx` line 40-41)

```typescript
// eslint-disable-next-line react-hooks/exhaustive-deps
}, [expiresAt]);
```

**Problem**: Suppresses exhaustive-deps warning — `router` and `isRefreshEnabled` are not in dependency array.

**Impact**: If `isRefreshEnabled` changes, the effect won't re-run with updated value. Minor issue in practice as these props typically don't change.

---

## Summary of Error Handling Gaps

| Category | Gap | Severity |
|----------|-----|----------|
| Global error boundary | None exists | High |
| Network error handling | Crashes on errors without response body | High |
| Request timeouts | Not configured | High |
| Loading states | None implemented | Medium |
| Retry logic | None implemented | Medium |
| Logging/monitoring | None implemented | Medium |
| Silent server failures | Multiple pages hide API errors | Medium |
| Rate limit handling | Not implemented | Low |
| Inconsistent patterns | Sign-out uses different error handling | Low |
