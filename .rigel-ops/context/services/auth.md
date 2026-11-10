---
service_id: ticketing-app/auth
service_name: auth
service_type: microservice
framework: express
language: typescript
path: services/auth
last_analyzed: 2026-11-10T20:25:08.791Z
commit_sha: a403fff288018e2dbbe4bd55b6d96446a2ef30ed
api_endpoint_count: 5
dependency_count: 8
schema_version: "1.0"
---

# auth

## Overview

**Type**: microservice
**Framework**: express
**Language**: typescript
**Path**: `services/auth`

## Dependencies

### Internal
- **mongoose** (package) — MongoDB ODM for user data persistence
- **mongodb** (shared-db) — MongoDB connection via MONGO_URI environment variable
- **jsonwebtoken** (package) — JWT token generation and verification using JWT_KEY
- **express** (package) — HTTP server framework
- **express-validator** (package) — Request validation middleware
- **@b.anik/ticketing-common** (package) — Common utilities including error handlers, middleware (currentUserMiddleware, requireAuthMiddleware, validateRequestMiddleware, errorHandlerMiddleware)
- **cookie-session** (package) — Session management via cookies
- **body-parser** (package) — Request body parsing middleware

### External (npm)
- @b.anik/ticketing-common
- body-parser
- cookie-session
- express
- express-async-errors
- express-validator
- jsonwebtoken
- mongoose


## API Endpoints

| Method | Path | Handler | Auth |
|--------|------|---------|------|
| GET | /api/users/health | src/routes/health.ts:healthRouter | — |
| GET | /api/users/current-user | src/routes/current-user.ts:currentUserRouter | JWT |
| POST | /api/users/sign-up | src/routes/sign-up.ts:signUpRouter | None |
| POST | /api/users/sign-in | src/routes/sign-in.ts:signInRouter | None |
| POST | /api/users/sign-out | src/routes/sign-out.ts:signOutRouter | JWT |

## Communication

### Outbound
_No outbound communication detected._

### Inbound
_No inbound communication detected._
