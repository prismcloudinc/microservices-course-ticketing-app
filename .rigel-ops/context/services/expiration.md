---
service_id: ticketing-app/expiration
service_name: expiration
service_type: microservice
language: typescript
path: services/expiration
last_analyzed: 2026-11-10T20:25:08.791Z
commit_sha: a403fff288018e2dbbe4bd55b6d96446a2ef30ed
api_endpoint_count: 0
dependency_count: 5
schema_version: "1.0"
---

# expiration

## Overview

**Type**: microservice
**Framework**: —
**Language**: typescript
**Path**: `services/expiration`

## Dependencies

### Internal
- **@b.anik/ticketing-common** (package) — Shared library for event types (OrderCreatedEvent, ExpirationCompleteEvent, Listener, Publisher, Subjects)
- **bull** (package) — Job queue library for Redis-backed task scheduling
- **node-nats-streaming** (package) — NATS Streaming client for event publishing and consuming
- **redis** (shared-db) — Redis instance at REDIS_HOST for Bull queue storage
- **nats-streaming** (api-call) — NATS Streaming cluster connection at NATS_URL with cluster ID and client ID

### External (npm)
- @b.anik/ticketing-common
- bull
- node-nats-streaming


## API Endpoints

| Method | Path | Handler | Auth |
|--------|------|---------|------|
_No API endpoints discovered._

## Communication

### Outbound
- **message-queue** → nats-streaming: {"description":"Subscribes to Subjects.OrderCreated event via NATS Streaming with queue group 'expiration-service'"}
- **message-queue** → redis: {"description":"Bull queue 'order:expiration' processes delayed jobs and triggers expiration events"}

### Inbound
_No inbound communication detected._
