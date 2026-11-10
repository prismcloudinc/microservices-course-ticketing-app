---
service_id: ticketing-app/client
service_name: client
service_type: web-app
framework: nextjs
language: typescript
path: apps/client
last_analyzed: 2026-11-10T20:25:08.791Z
commit_sha: a403fff288018e2dbbe4bd55b6d96446a2ef30ed
api_endpoint_count: 1
dependency_count: 9
schema_version: "1.0"
---

# client

## Overview

**Type**: web-app
**Framework**: nextjs
**Language**: typescript
**Path**: `apps/client`

## Dependencies

### Internal
- **ingress-nginx** (api-call) — HTTP client to http://ingress-nginx-controller.ingress-nginx.svc.cluster.local via axios
- **user-service** (api-call) — GET /api/users/current-user to fetch current user
- **axios** (package) — HTTP client library for API communication
- **next** (package) — Next.js framework for server-side rendering and API routes
- **react** (package) — React UI library
- **react-dom** (package) — React DOM rendering
- **react-stripe-checkout** (package) — Stripe checkout integration
- **sonner** (package) — Toast notification library
- **clsx** (package) — Utility for constructing className strings

### External (npm)
- axios
- clsx
- next
- react
- react-dom
- react-stripe-checkout
- sonner


## API Endpoints

| Method | Path | Handler | Auth |
|--------|------|---------|------|
| GET | /health/check | apps/client/src/app/health/check/route.ts:GET | — |

## Communication

### Outbound
- **http** → ingress-nginx: {"description":"Kubernetes ingress nginx controller at http://ingress-nginx-controller.ingress-nginx.svc.cluster.local serving as API gateway"}
- **http** → user-service: {"description":"HTTP GET to /api/users/current-user via ingress"}

### Inbound
_No inbound communication detected._
