---
service_id: ticketing-app/nats-test
service_name: nats-test
service_type: shared-library
language: typescript
path: nats-test
last_analyzed: 2026-11-10T20:25:08.791Z
commit_sha: a403fff288018e2dbbe4bd55b6d96446a2ef30ed
api_endpoint_count: 0
dependency_count: 1
schema_version: "1.0"
---

# nats-test

## Overview

**Type**: shared-library
**Framework**: —
**Language**: typescript
**Path**: `nats-test`

## Dependencies

### Internal
- **node-nats-streaming** (package) — NATS Streaming client for pub/sub messaging

### External (npm)
- node-nats-streaming


## API Endpoints

| Method | Path | Handler | Auth |
|--------|------|---------|------|
_No API endpoints discovered._

## Communication

### Outbound
- **message-queue** → nats-streaming: {"description":"Publishes to NATS subject 'ticket:created' with ticket creation events"}

### Inbound
_No inbound communication detected._
