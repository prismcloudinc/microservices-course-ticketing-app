---
service_id: ticketing-app/auth
service_name: auth
prompt_id: 01-service-identity
prompt_label: Service Identity & Anatomy
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.19352
schema_version: "2.0"
---

# Service Anatomy: Auth Service

## Section 1: Identity

| Property | Value |
|----------|-------|
| **Package Name** | `auth` (`services/auth/package.json:2`) |
| **Purpose** | User authentication and authorization service for the ticketing application |
| **Product/Platform** | Part of the `ticketing-app` monorepo — a microservices-based ticket marketplace (`package.json:2`) |
| **Runtime** | Node.js 20 (pinned in `.nvmrc`, Dockerfile uses `node:20-alpine`) |
| **Framework** | Express.js ^4.18.3 (`services/auth/package.json:22`) |
| **Database** | MongoDB — Mongoose ^8.2.4 ORM, connection via `MONGO_URI` environment variable (`services/auth/src/index.ts:15`) |

### Environment Variables Required

| Variable | Description | Source |
|----------|-------------|--------|
| `JWT_KEY` | Secret key for signing JWTs | `services/auth/src/index.ts:6-8` |
| `MONGO_URI` | MongoDB connection string | `services/auth/src/index.ts:10-12`, Kubernetes sets to `mongodb://auth-mongo-srv:27017/auth` (`infra/k8s/auth-depl.yaml:25`) |

---

## Section 2: Entry Points

| File | Trigger | Description |
|------|---------|-------------|
| `services/auth/src/index.ts` | Application startup (`pnpm start` / `pnpm dev`) | Bootstraps MongoDB connection and starts Express HTTP server on port 3000 |

### HTTP Routes Exposed

| Route | Method | File | Description |
|-------|--------|------|-------------|
| `/api/users/health` | GET | `services/auth/src/routes/health.ts` | Kubernetes liveness probe endpoint; returns HTTP 200 |
| `/api/users/current-user` | GET | `services/auth/src/routes/current-user.ts` | Returns currently authenticated user from JWT or `null` if unauthenticated |
| `/api/users/sign-up` | POST | `services/auth/src/routes/sign-up.ts` | Creates new user account, hashes password, issues JWT in session cookie |
| `/api/users/sign-in` | POST | `services/auth/src/routes/sign-in.ts` | Validates credentials, issues JWT in session cookie on success |
| `/api/users/sign-out` | POST | `services/auth/src/routes/sign-out.ts` | Clears session cookie (requires authentication) |

---

## Section 3: Directory Map

### `services/auth/`

| Path | Purpose |
|------|---------|
| `services/auth/` | Root directory for the auth microservice |

| File | Description |
|------|-------------|
| `.dockerignore` | Excludes `node_modules`, `dist`, `build`, `.turbo` from Docker context |
| `Dockerfile` | Multi-stage Docker build using `node:20-alpine`, installs pnpm@9, runs `pnpm dev` |
| `jest.config.js` | Jest configuration extending `@repo/jest-config`, sets test setup file |
| `package.json` | Package manifest with dependencies and npm scripts |
| `pnpm-lock.yaml` | Lockfile with resolved dependency versions |
| `tsconfig.json` | TypeScript configuration targeting ES2016, CommonJS modules, strict mode |

### `services/auth/src/`

| Path | Purpose |
|------|---------|
| `services/auth/src/` | Main source code directory |

| File | Description |
|------|-------------|
| `app.ts` | Express application setup — configures middlewares (body-parser, cookie-session, currentUserMiddleware), mounts routers, sets up 404 handler and error middleware |
| `index.ts` | Application entry point — validates env vars, connects to MongoDB, starts HTTP server on port 3000 |

### `services/auth/src/models/`

| Path | Purpose |
|------|---------|
| `services/auth/src/models/` | Mongoose schema and model definitions |

| File | Description |
|------|-------------|
| `index.ts` | Barrel export for User model |
| `user.ts` | Defines `User` Mongoose model with `email` and `password` fields; includes pre-save hook to hash passwords using `Password.toHash()`; implements `build()` static method for type-safe document creation; transforms JSON output to rename `_id` to `id` and omit `password` and `__v` |

### `services/auth/src/routes/`

| Path | Purpose |
|------|---------|
| `services/auth/src/routes/` | Express route handlers |

| File | Description |
|------|-------------|
| `index.ts` | Barrel export for all routers |
| `current-user.ts` | GET `/api/users/current-user` — returns `{ currentUser }` from `req.currentUser` populated by `currentUserMiddleware` |
| `health.ts` | GET `/api/users/health` — returns HTTP 200 with empty body for Kubernetes liveness probes |
| `sign-in.ts` | POST `/api/users/sign-in` — validates email/password with express-validator, verifies credentials against User model, issues JWT containing `{ id, email }` stored in session cookie |
| `sign-out.ts` | POST `/api/users/sign-out` — protected by `requireAuthMiddleware`, clears session by setting `req.session = null` |
| `sign-up.ts` | POST `/api/users/sign-up` — validates email format and password length (4-20 chars), checks for duplicate email, creates User document, issues JWT in session cookie, returns 201 with user data |

### `services/auth/src/routes/__tests__/`

| Path | Purpose |
|------|---------|
| `services/auth/src/routes/__tests__/` | Route integration tests using supertest |

| File | Description |
|------|-------------|
| `current-user.test.ts` | Tests that unauthenticated requests return `currentUser: null` and authenticated requests return user data |
| `sign-in.test.ts` | Tests successful login, invalid credentials, validation errors, and cookie setting |
| `sign-out.test.ts` | Tests that sign-out clears the session cookie |
| `sign-up.test.ts` | Tests successful registration, validation errors, duplicate email rejection, and cookie setting |

### `services/auth/src/services/`

| Path | Purpose |
|------|---------|
| `services/auth/src/services/` | Business logic services |

| File | Description |
|------|-------------|
| `index.ts` | Barrel export for Password service |
| `password.ts` | `Password` class with static methods: `toHash(password)` — hashes password using Node's `scrypt` with random 8-byte salt, returns `hash.salt` format; `compare(storedPassword, suppliedPassword)` — verifies password against stored hash |

### `services/auth/src/services/__test__/`

| Path | Purpose |
|------|---------|
| `services/auth/src/services/__test__/` | Unit tests for service layer |

| File | Description |
|------|-------------|
| `password.test.ts` | Tests `Password.toHash()` produces salted hash and `Password.compare()` correctly verifies passwords |

### `services/auth/src/test/`

| Path | Purpose |
|------|---------|
| `services/auth/src/test/` | Test infrastructure and utilities |

| File | Description |
|------|-------------|
| `setup.ts` | Jest setup file — starts `MongoMemoryServer`, connects Mongoose, sets `JWT_KEY` env var, clears all collections before each test, closes connections after all tests |
| `utils.ts` | Test helper functions: `signUp(agent, body)` — registers a test user; `signOut(agent, expect)` — signs out with optional status assertion |

---

## Section 4: Dependency Analysis

### Production Dependencies

| Package | Version | Usage in Codebase |
|---------|---------|-------------------|
| `@b.anik/ticketing-common` | ^0.1.2 | Shared library imported in `src/app.ts` and route files; provides `NotFoundError`, `BadRequestError`, `currentUserMiddleware`, `errorHandlerMiddleware`, `requireAuthMiddleware`, `validateRequestMiddleware` |
| `body-parser` | ^1.20.2 | Imported in `src/app.ts:3` via `{ json }` — parses JSON request bodies |
| `cookie-session` | ^2.1.0 | Imported in `src/app.ts:4` — manages signed sessions in cookies; configured with `signed: false` and `secure: true` (except in test environment) |
| `express` | ^4.18.3 | Core web framework; imported in `src/app.ts:1` and all route files; creates Express app, Router instances |
| `express-async-errors` | ^3.1.1 | Imported in `src/app.ts:2` — patches Express to automatically catch rejected promises in async route handlers |
| `express-validator` | ^7.0.1 | Imported in `src/routes/sign-in.ts:2` and `src/routes/sign-up.ts:2` via `{ body }` — validates request body fields (email format, password length) |
| `jsonwebtoken` | ^9.0.2 | Imported in `src/routes/sign-in.ts:3` and `src/routes/sign-up.ts:3` — generates JWTs containing user `id` and `email`, signed with `JWT_KEY` |
| `mongoose` | ^8.2.4 | Imported in `src/index.ts:1` and `src/models/user.ts:1` — MongoDB ODM; defines User schema, connects to database |

### Development Dependencies

| Package | Version | Usage in Codebase |
|---------|---------|-------------------|
| `@repo/jest-config` | `link:*` | Internal workspace package imported in `jest.config.js:1` — provides shared Jest configuration |
| `@types/body-parser` | ^1.19.5 | TypeScript type definitions for body-parser |
| `@types/cookie-session` | ^2.0.48 | TypeScript type definitions for cookie-session; enables typed `req.session` |
| `@types/express` | ^4.17.21 | TypeScript type definitions for Express |
| `@types/jsonwebtoken` | ^9.0.6 | TypeScript type definitions for jsonwebtoken |
| `@types/node` | ^20.11.30 | TypeScript type definitions for Node.js built-ins (`crypto`, `util`) |
| `@types/supertest` | ^6.0.2 | TypeScript type definitions for supertest |
| `mongodb-memory-server` | ^9.1.8 | Imported in `src/test/setup.ts:1` — provides in-memory MongoDB instance for integration tests |
| `nodemon` | ^3.1.0 | Dev server with file watching; used by `pnpm dev` script |
| `supertest` | ^6.3.4 | Imported in all route test files — HTTP assertion library for testing Express routes |
| `tickets` | ^0.2.2 | Declared in package.json but no import found in auth service source code — declared but no import found |
| `ts-node` | ^10.9.2 | TypeScript execution engine; used by `pnpm start` and nodemon |
| `typescript` | ^5.4.2 | TypeScript compiler; used for type checking |

---

## Section 5: Internal Package Usage

### `@b.anik/ticketing-common` (External npm package — org-scoped shared module)

**Note:** This is published to npm as `@b.anik/ticketing-common` and maintained in a separate repository (`github.com/berkinanik/microservices-course-common-module`) as a git submodule. The common package source is not present in this repository, so exports are identified from import statements only.

| Import Location | Exports Used | Functionality Provided |
|-----------------|--------------|------------------------|
| `src/app.ts:7-10` | `NotFoundError` | Custom error class thrown for unmatched routes (404 handler) |
| `src/app.ts:7-10` | `currentUserMiddleware` | Express middleware that extracts JWT from session cookie and populates `req.currentUser` |
| `src/app.ts:7-10` | `errorHandlerMiddleware` | Express error handler that formats errors into consistent JSON responses |
| `src/routes/sign-out.ts:3` | `requireAuthMiddleware` | Express middleware that returns 401 if `req.currentUser` is not set |
| `src/routes/sign-in.ts:5` | `BadRequestError` | Custom error class for 400 responses (invalid credentials, duplicate email) |
| `src/routes/sign-in.ts:5` | `validateRequestMiddleware` | Express middleware that checks express-validator results and throws on validation errors |
| `src/routes/sign-up.ts:5` | `BadRequestError` | Same as above |
| `src/routes/sign-up.ts:5` | `validateRequestMiddleware` | Same as above |

### `@repo/jest-config` (Internal workspace package)

| Import Location | Exports Used | Functionality Provided |
|-----------------|--------------|------------------------|
| `jest.config.js:1` | Default export (full config object) | Shared Jest configuration with `ts-jest` preset, `node` test environment, verbose output, module paths configuration, and watch path ignore patterns (`packages/jest-config/jest.config.js`) |

---

## Kubernetes Deployment

The auth service is deployed to Kubernetes with the following configuration (`infra/k8s/auth-depl.yaml`):

| Resource | Name | Details |
|----------|------|---------|
| Deployment | `auth-depl` | 1 replica, image `berkinanik/ticketing-auth:latest` |
| Service | `auth-srv` | ClusterIP on port 3000 |
| Database | `auth-mongo-depl` / `auth-mongo-srv` | Standalone MongoDB on port 27017 (`infra/k8s/auth-mongo-depl.yaml`) |

**Ingress routing** (`infra/k8s/ingress-srv.yaml:14-19`): All requests to `/api/users/*` are routed to `auth-srv:3000`.

---

## Data Model

### User Collection

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `email` | String | Required | User's email address |
| `password` | String | Required | Hashed password in `hash.salt` format (scrypt, 64-byte output) |
| `_id` | ObjectId | Auto-generated | Transformed to `id` in JSON output |

**Schema transformations** (`src/models/user.ts:30-36`):
- `_id` → `id` rename
- `password` field removed from JSON output
- `__v` field removed from JSON output

**Pre-save hook** (`src/models/user.ts:41-48`): Automatically hashes password on creation or modification.
