---
service_id: ticketing-app/nats-test
service_name: nats-test
prompt_id: 07-integration-manifest
prompt_label: Outbound Integration Manifest
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.07913500000000001
schema_version: "2.0"
---

# Integration Manifest: nats-test

> **Service Type**: Shared Library / Demo Package  
> **Runtime**: Node.js 20  
> **Primary Dependency**: `node-nats-streaming` ^0.3.2

This package is a standalone demonstration/test utility for NATS Streaming pub/sub patterns. It connects directly to a NATS Streaming server and provides reusable base classes for type-safe event publishing and listening.

---

## Part A: Services I Call (HTTP)

**None.** This package makes no outbound HTTP calls to other services. The only network communication is via the NATS Streaming protocol to the message broker.

---

## Part B: Messages/Events I Produce

### -> NATS Streaming: `ticket:created`
- **Producer file**: `nats-test/src/events/ticket-created-publisher.ts`
- **Entry point**: `nats-test/src/publisher.ts:16-22`
- **Connection**: NATS Streaming cluster `ticketing` at `http://localhost:4222` (`nats-test/src/publisher.ts:4-6`)
- **Client ID**: Hardcoded `abc` (`nats-test/src/publisher.ts:4`)

| Event Key/Type | Trigger (what causes this publish) | Payload Shape |
|----------------|-----------------------------------|---------------|
| `ticket:created` | Manual CLI invocation via `pnpm run publish` | `{ id: string, title: string, price: number }` (interface at `nats-test/src/events/ticket-created-event.ts:5-9`) |

**Sample published data** (`nats-test/src/publisher.ts:18-21`):
```typescript
{ id: '123', title: 'concert', price: 20 }
```

---

## Part C: Messages/Events I Consume

### <- NATS Streaming: `ticket:created`
- **Consumer group/subscription**: `payments-service` (durable subscription)
- **Handler file**: `nats-test/src/events/ticket-created-listener.ts`
- **Entry point**: `nats-test/src/listener.ts:21`
- **Connection**: NATS Streaming cluster `ticketing` at `http://localhost:4222` (`nats-test/src/listener.ts:9-11`)
- **Client ID**: Random 8-character hex string via `crypto.randomBytes(4)` (`nats-test/src/listener.ts:7`)

| Event Key/Type | Handler Function | What It Does | Output |
|----------------|-----------------|-------------|--------|
| `ticket:created` | `TicketCreatedListener.onMessage()` (`nats-test/src/events/ticket-created-listener.ts:10-13`) | Logs event data to console | Console output only (demo purposes) |

**Subscription options** (from base class `nats-test/src/events/base-listener.ts:19-31`):
- Deliver all available messages from beginning
- Manual acknowledgment mode
- Ack wait timeout: 5000ms (default)
- Durable subscription name: `payments-service`

---

### <- NATS Streaming: `ticket:updated`
- **Consumer group/subscription**: `payments-service` (durable subscription)
- **Handler file**: `nats-test/src/events/ticket-updated-listener.ts`
- **Entry point**: `nats-test/src/listener.ts:22`
- **Connection**: Same as above

| Event Key/Type | Handler Function | What It Does | Output |
|----------------|-----------------|-------------|--------|
| `ticket:updated` | `TicketUpdatedListener.onMessage()` (`nats-test/src/events/ticket-updated-listener.ts:10-13`) | Logs event data to console | Console output only (demo purposes) |

---

## Part D: Workflows I Trigger

**None.** This package does not trigger any durable workflows.

---

## Part E: Workflows/Activities I Execute

**None.** This package is not a workflow worker.

---

## Part F: Cloud Services I Use

| Cloud Service | Operation | Resource | Purpose |
|---------------|-----------|----------|---------|
| *None* | — | — | — |

This package has no direct cloud SDK integrations. It expects a NATS Streaming server to be available (typically via Kubernetes port-forwarding as documented in `nats-test/README.md:7-11`).

---

## Part G: External Systems I Integrate With

### <-> NATS Streaming Server
- **Client/SDK file**: Uses `node-nats-streaming` package (imported in `nats-test/src/publisher.ts:1`, `nats-test/src/listener.ts:1`)
- **Operations**: 
  - `nats.connect()` - Establish connection to cluster
  - `stan.publish()` - Publish messages to subjects
  - `stan.subscribe()` - Subscribe to subjects with queue group
  - `stan.close()` - Graceful disconnect
- **Connection details**:
  - Cluster ID: `ticketing`
  - URL: `http://localhost:4222` (hardcoded, no env var)
  - Monitoring URL: `http://localhost:8222` (documented in README)
- **Auth**: None configured (connects without credentials)

---

## Additional Notes

### Defined but Unused Event Types
The `Subjects` enum (`nats-test/src/events/subjects.ts:1-5`) defines an additional subject that has no corresponding publisher or listener implementation in this package:
- `OrderUpdated = 'order:updated'` — defined but not used

### Infrastructure Requirements
Per `nats-test/README.md`, this package requires:
1. A NATS Streaming server running on `localhost:4222`
2. Typically accessed via Kubernetes port-forward: `kubectl port-forward <nats-pod-name> 4222:4222`

---

## Summary Block

```yaml
service: nats-test
platform: ticketing-app
connects_to:
  http_outbound: []
  events_produced:
    - destination: ticket:created
      events:
        - ticket:created
  events_consumed:
    - source: ticket:created
      events:
        - ticket:created
    - source: ticket:updated
      events:
        - ticket:updated
  workflows_triggered: []
  workflows_executed: []
  cloud_services: []
  external:
    - system: NATS Streaming
      operations:
        - connect
        - publish
        - subscribe
        - close
```
