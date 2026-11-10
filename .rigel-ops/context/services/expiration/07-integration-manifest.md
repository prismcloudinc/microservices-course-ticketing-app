---
service_id: ticketing-app/expiration
service_name: expiration
prompt_id: 07-integration-manifest
prompt_label: Outbound Integration Manifest
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.098765
schema_version: "2.0"
---

# Integration Manifest: expiration

## Part A: Services I Call (HTTP)

**None.** This service does not make any outbound HTTP calls. It is a worker service that communicates exclusively via NATS Streaming events and Redis-backed job queues.

---

## Part B: Messages/Events I Produce

### -> NATS Streaming: `expiration:complete`

- **Publisher file**: `services/expiration/src/events/publishers/expiration-complete-publisher.ts`
- **Invoked from**: `services/expiration/src/queues/expiration-queue.ts` (line 16)
- **Subject**: `Subjects.ExpirationComplete` (from `@b.anik/ticketing-common`)
- **Event types published**:

| Event Key/Type | Trigger (what causes this publish) | Payload Shape |
|----------------|-----------------------------------|---------------|
| `ExpirationComplete` | Bull queue job processes after scheduled delay expires | `{ orderId: string }` |

**Publisher implementation** (`expiration-complete-publisher.ts`, lines 1-5):
```typescript
export class ExpirationCompletePublisher extends Publisher<ExpirationCompleteEvent> {
  readonly subject = Subjects.ExpirationComplete;
}
```

**Trigger code** (`expiration-queue.ts`, lines 15-18):
```typescript
expirationQueue.process(async (job) => {
  await new ExpirationCompletePublisher(natsWrapper.client).publish({
    orderId: job.data.orderId,
  });
});
```

---

## Part C: Messages/Events I Consume

### <- NATS Streaming: `order:created`

- **Consumer group/subscription**: `expiration-service` (`order-created-listener.ts`, line 7)
- **Handler file**: `services/expiration/src/events/listeners/order-created-listener.ts`
- **Subject**: `Subjects.OrderCreated` (from `@b.anik/ticketing-common`)
- **Event types consumed**:

| Event Key/Type | Handler Function | What It Does | Output |
|----------------|-----------------|-------------|--------|
| `OrderCreated` | `OrderCreatedListener.onMessage()` | Calculates delay based on `expiresAt` timestamp; schedules a delayed job in the Bull queue | Adds job to Redis Bull queue `order:expiration` with `{ orderId: string }` payload and calculated delay |

**Event payload shape** (from test file `order-created-listener.test.ts`, lines 7-16):
```typescript
{
  id: string;
  version: number;
  status: OrderStatus;
  ticket: {
    id: string;
    price: number;
  };
  userId: string;
  expiresAt?: string;  // ISO 8601 timestamp, optional
}
```

**Handler logic** (`order-created-listener.ts`, lines 9-32):
- If `expiresAt` is undefined, resolves immediately without scheduling a job
- Otherwise, calculates delay as `new Date(data.expiresAt).getTime() - new Date().getTime()`
- Adds job to `expirationQueue` with `{ orderId: data.id }` and the calculated delay

---

## Part D: Workflows I Trigger

**None.** This service does not use Temporal, Step Functions, Inngest, or any other durable workflow orchestration system.

---

## Part E: Workflows/Activities I Execute

**None.** This is not a workflow worker.

---

## Part F: Cloud Services I Use

| Cloud Service | Operation | Resource | Purpose |
|---------------|-----------|----------|---------|
| Redis | Queue operations (add job, process job) via Bull library | Queue name: `order:expiration` | Job scheduling and delayed execution for order expiration; stores jobs with configurable delays |

**Redis connection details** (`expiration-queue.ts`, lines 9-13):
```typescript
const expirationQueue = new Queue<Payload>('order:expiration', {
  redis: {
    host: process.env.REDIS_HOST,
  },
});
```

**Environment variable**: `REDIS_HOST`  
**Kubernetes default value**: `expiration-redis-srv` (`infra/k8s/expiration-depl.yaml`, line 28)  
**Redis port**: 6379 (default, `infra/k8s/expiration-redis-depl.yaml`, line 29)

---

## Part G: External Systems I Integrate With

**None.** This service does not integrate with any external third-party systems (payment processors, CRM/ERP, notification services, etc.).

---

## Infrastructure Dependencies

### NATS Streaming Server

- **Client library**: `node-nats-streaming` v0.3.2 (`package.json`, line 21)
- **Connection file**: `services/expiration/src/nats-wrapper.ts`
- **Environment variables**:
  | Variable | Purpose | K8s Default Value |
  |----------|---------|-------------------|
  | `NATS_URL` | NATS server URL | `http://nats-srv:4222` |
  | `NATS_CLUSTER_ID` | NATS Streaming cluster identifier | `ticketing` |
  | `NATS_CLIENT_ID` | Unique client identifier | Pod metadata name |

**Connection code** (`nats-wrapper.ts`, lines 15-23):
```typescript
async connect(clusterId: string, clientId: string, url: string) {
  this._client = nats.connect(clusterId, clientId, { url });
  return new Promise<void>((resolve, reject) => {
    this.client.on('connect', () => resolve());
    this.client.on('error', (err) => reject(err));
  });
}
```

### Dedicated Redis Instance

- **Deployment**: `expiration-redis-depl` (`infra/k8s/expiration-redis-depl.yaml`)
- **Service**: `expiration-redis-srv`
- **Image**: `redis` (official Docker image)
- **Port**: 6379

---

## Summary Block

```yaml
service: expiration
platform: ticketing-app
connects_to:
  http_outbound: []
  events_produced:
    - destination: nats-streaming/expiration:complete
      events:
        - ExpirationComplete
  events_consumed:
    - source: nats-streaming/order:created
      consumer_group: expiration-service
      events:
        - OrderCreated
  workflows_triggered: []
  workflows_executed: []
  cloud_services:
    - service: redis
      resources:
        - queue:order:expiration
  external: []
  infrastructure:
    - service: nats-streaming
      env_vars:
        - NATS_URL
        - NATS_CLUSTER_ID
        - NATS_CLIENT_ID
    - service: redis
      env_vars:
        - REDIS_HOST
```
