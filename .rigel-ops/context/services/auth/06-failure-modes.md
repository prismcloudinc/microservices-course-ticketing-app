---
service_id: ticketing-app/auth
service_name: auth
prompt_id: 06-failure-modes
prompt_label: Failure Modes & Edge Cases
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.24416500000000002
schema_version: "2.0"
---

# Error Handling & Failure Mode Analysis: Auth Service

## Part A: Error Handling Architecture

### Global Error Handler

**Implementation**: `errorHandlerMiddleware` from `@b.anik/ticketing-common` external npm package  
**Location**: `services/auth/src/app.ts:48`

```typescript
app.use(errorHandlerMiddleware);
```

The error handler middleware is provided by an external npm package (`@b.anik/ticketing-common@^0.1.2`). The source code is not present in this repository (maintained in a separate git submodule at `github.com/berkinanik/microservices-course-common-module`), so the exact implementation details cannot be verified.

**Error Types Caught** (inferred from usage patterns):
- `NotFoundError` — thrown for unmatched routes (`services/auth/src/app.ts:46`)
- `BadRequestError` — thrown for validation failures and invalid credentials (`services/auth/src/routes/sign-in.ts:23,29`, `services/auth/src/routes/sign-up.ts:25`)
- Validation errors — handled by `validateRequestMiddleware` from express-validator integration
- Generic `Error` — thrown during startup for missing environment variables (`services/auth/src/index.ts:7,11`)

**Async Error Handling**: The service uses `express-async-errors` (`services/auth/src/app.ts:2`) which patches Express to automatically catch rejected promises in async route handlers without explicit try/catch.

### Logging

**Library**: None — uses native `console.log` only  
**Location**: `services/auth/src/index.ts:16,18,23`

```typescript
console.log('Connected to MongoDB!');  // Line 16
console.log(err);                       // Line 18 - on connection failure
console.log('Listening on port 3000!'); // Line 23
```

**Log Levels**: None — all output uses `console.log`  
**Structured Fields**: None — plain text output only  
**Gap**: No structured logging library (e.g., winston, pino, bunyan) is configured. Error context, request IDs, timestamps, and other operational metadata are not captured.

### External Error Tracking

**Status**: Not configured  
**Evidence**: No imports or configuration found for error tracking services (Sentry, Datadog, New Relic, Bugsnag, Rollbar, or similar).

---

## Part B: Integration Point Failure Behavior

### MongoDB Failure

**Connection Code**: `services/auth/src/index.ts:14-20`

```typescript
try {
  await mongoose.connect(process.env.MONGO_URI!);
  console.log('Connected to MongoDB!');
} catch (err) {
  console.log(err);
  process.exit(1);
}
```

| Aspect | Behavior |
|--------|----------|
| **Detection** | Exception thrown by `mongoose.connect()` |
| **Immediate Response** | Logs error to console, terminates process with `exit(1)` |
| **Retry Policy** | None — no retry logic implemented |
| **Circuit Breaker** | None |
| **Fallback** | None — service cannot start without MongoDB |
| **Alerting** | None — relies on Kubernetes liveness probe failure (`/api/users/health`) and pod restart |
| **Dead Letter Queue** | Not applicable |
| **Recovery** | Kubernetes will restart the pod; service re-attempts connection on restart |

**Runtime Query Failures**: No explicit error handling observed for runtime MongoDB operations (e.g., `User.findOne()`, `user.save()`). The `express-async-errors` package will catch unhandled rejections and pass them to `errorHandlerMiddleware`, but specific handling (retry, fallback) is not implemented.

**Connection Options**: `services/auth/src/index.ts:15`
```typescript
await mongoose.connect(process.env.MONGO_URI!);
```
No connection options specified — uses Mongoose defaults. Missing:
- Connection timeout configuration
- Socket timeout configuration  
- Retry writes configuration
- Pool size configuration

### External API Clients

The auth service does not make outbound HTTP calls to external services. It is a self-contained authentication service.

### Message Broker (NATS)

**Status**: Not used by auth service  
**Evidence**: The auth service does not import `node-nats-streaming` or any NATS wrapper. Unlike other services (orders, tickets, payments) that use NATS for event publishing, the auth service operates independently.

### Cache

**Status**: Not configured  
**Evidence**: No Redis, Memcached, or other caching layer is used.

---

## Part C: Data Consistency Edge Cases

### Crash Mid-Operation

**Sign-up Scenario** (`services/auth/src/routes/sign-up.ts:30-46`):
1. User document created with `User.build()`
2. Document saved with `user.save()`
3. JWT generated
4. Session cookie set

**Risk**: If the service crashes after step 2 but before step 4:
- User record exists in database
- User receives an error response
- User can sign in with their credentials (data is consistent)
- No orphaned or partial records (MongoDB document writes are atomic)

**Risk**: If crash occurs during `user.save()`:
- Mongoose driver handles connection failures
- MongoDB single-document operations are atomic
- Either the user is fully created or not at all

### Duplicate Message Processing (Idempotency)

**Status**: Not applicable  
**Rationale**: Auth service does not consume messages from a message broker. All operations are synchronous HTTP requests.

### Concurrent Modifications

**Scenario**: Two simultaneous sign-up requests with the same email

**Handling** (`services/auth/src/routes/sign-up.ts:22-25`):
```typescript
const existingUser = await User.findOne({ email });
if (existingUser) {
  throw new BadRequestError('Email in use');
}
```

**Gap**: This is a read-then-write pattern vulnerable to race conditions:
1. Request A: `findOne({ email })` returns null
2. Request B: `findOne({ email })` returns null  
3. Request A: Creates user with email
4. Request B: Creates duplicate user with email

**Missing**: No unique index on email field in Mongoose schema (`services/auth/src/models/user.ts:18-28`):
```typescript
const userSchema = new mongoose.Schema({
  email: {
    type: String,
    required: true,
    // No 'unique: true' constraint
  },
  // ...
});
```

Without a database-level unique constraint, duplicate emails can be inserted under concurrent load.

### Optimistic Concurrency Control

**Status**: Not implemented  
**Evidence**: The User model (`services/auth/src/models/user.ts`) does not use `mongoose-update-if-current` plugin (unlike the Order model in the orders service which uses it at `services/orders/src/models/order.ts:4,58`).

**Impact**: Low — auth service performs create-only operations (sign-up) and read operations (sign-in, current-user). User records are not updated after creation in the current implementation.

### Stale Data

**Scenario**: Not applicable — auth service does not cache user data or call downstream services that might return stale data.

---

## Part D: Known Gotchas

### Hardcoded Values

| Location | Value | Recommendation |
|----------|-------|----------------|
| `services/auth/src/index.ts:22` | Port `3000` | Should use `process.env.PORT \|\| 3000` |
| `services/auth/src/routes/sign-up.ts:16` | Password length `{ min: 4, max: 20 }` | Consider making configurable |
| `services/auth/src/services/password.ts:8` | Salt length `8` bytes (randomBytes) | Document or make configurable |
| `services/auth/src/services/password.ts:9` | Key length `64` bytes (scrypt output) | Document or make configurable |

### Missing Timeout Configuration

| Component | Issue | Location |
|-----------|-------|----------|
| MongoDB connection | No connection timeout set | `services/auth/src/index.ts:15` |
| Express server | No request timeout configured | `services/auth/src/index.ts:22` |
| body-parser | No request body size limit | `services/auth/src/app.ts:26` — `app.use(json())` uses default 100kb |

### Missing Signal Handlers

**Location**: `services/auth/src/index.ts`  
**Issue**: No graceful shutdown handlers for `SIGTERM`, `SIGINT`, or `SIGUSR2`

**Contrast**: The orders service implements these handlers (`services/orders/src/index.ts:41-43`):
```typescript
process.on('SIGINT', () => natsWrapper.client.close());
process.on('SIGTERM', () => natsWrapper.client.close());
process.on('SIGUSR2', () => natsWrapper.client.close());
```

**Impact**: When Kubernetes terminates the pod, the auth service does not:
- Close MongoDB connections gracefully
- Stop accepting new requests
- Wait for in-flight requests to complete

### Security Considerations

| Issue | Location | Detail |
|-------|----------|--------|
| Unsigned cookies | `services/auth/src/app.ts:29` | `signed: false` means session cookie content is not verified for tampering |
| JWT_KEY assertion | `services/auth/src/routes/sign-in.ts:37`, `sign-up.ts:42` | Uses `process.env.JWT_KEY!` with non-null assertion; crash if undefined at runtime despite startup check |

### Missing Rate Limiting

**Status**: Not configured  
**Impact**: Sign-in endpoint (`/api/users/sign-in`) is vulnerable to brute-force attacks. No rate limiting is applied at the application level or ingress configuration (`infra/k8s/ingress-srv.yaml`).

### Missing Database Index

**Issue**: Email field lacks unique index  
**Location**: `services/auth/src/models/user.ts:20-23`

```typescript
email: {
  type: String,
  required: true,
  // Missing: unique: true
},
```

**Impact**: Potential duplicate user accounts under concurrent sign-up requests.

### Kubernetes Deployment Observations

**Liveness Probe** (`infra/k8s/auth-depl.yaml:26-31`):
```yaml
livenessProbe:
  httpGet:
    path: /api/users/health
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 5
```

**Gap**: No readiness probe configured. The service is considered ready immediately, even if MongoDB connection is still being established.

### Declared But Unused Dependency

**Location**: `services/auth/package.json:39`
```json
"tickets": "^0.2.2"
```

This dependency is declared in devDependencies but no import from `tickets` is found in the auth service source code.

---

## Summary of Gaps

| Category | Gap | Severity (inferred) |
|----------|-----|---------------------|
| Logging | No structured logging | Medium |
| Error Tracking | No external error tracking | Medium |
| Database | No unique index on email | High |
| Database | No connection timeout/retry | Medium |
| Security | No rate limiting on sign-in | High |
| Security | Unsigned session cookies | Medium |
| Resilience | No graceful shutdown handlers | Low |
| Resilience | No readiness probe | Low |
| Configuration | Hardcoded port 3000 | Low |
