---
service_id: ticketing-app/payments
service_name: payments
prompt_id: 05-configuration-infrastructure
prompt_label: Configuration & Infrastructure
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.2617900000000001
schema_version: "2.0"
---

# Configuration & Environment Document: Payments Service

This document covers the configuration, environment variables, secrets, and infrastructure settings for the **payments** service within the ticketing application monorepo.

---

## Part A: Environment Variables

### Payments Service Environment Variables

| Variable Name | Used In (file:line) | Purpose | Required | Default Value | Sensitive |
|---|---|---|---|---|---|
| `JWT_KEY` | `services/payments/src/index.ts:8-10`, `services/payments/src/test/utils.ts:16` | Secret key for JWT token verification; used by common library's `currentUserMiddleware` to decode session cookies | Yes | None | **Yes** |
| `STRIPE_KEY` | `services/payments/src/index.ts:12-14`, `services/payments/src/stripe.ts:3` | Stripe API secret key for payment processing via `stripe.charges.create()` | Yes | None | **Yes** |
| `MONGO_URI` | `services/payments/src/index.ts:16-18,51` | MongoDB connection string for the payments database | Yes | None | **Yes** (contains credentials in production) |
| `NATS_URL` | `services/payments/src/index.ts:20-22,36` | NATS Streaming Server URL for event messaging | Yes | None | No |
| `NATS_CLUSTER_ID` | `services/payments/src/index.ts:24-26,34` | NATS cluster identifier (must match the cluster's configured ID) | Yes | None | No |
| `NATS_CLIENT_ID` | `services/payments/src/index.ts:35` | NATS client identifier; uses Kubernetes pod name in production | Yes (implicit) | None | No |
| `NODE_ENV` | `services/payments/src/app.ts:23` | Runtime environment; controls cookie-session `secure` flag (disabled when `=== 'test'`) | No | undefined | No |

### Client Application Environment Variables

| Variable Name | Used In (file:line) | Purpose | Required | Default Value | Sensitive |
|---|---|---|---|---|---|
| `STRIPE_CLIENT_KEY` | `apps/client/src/app/orders/[orderId]/page.tsx:22` | Stripe publishable key for client-side payment form | Yes | `''` (empty string) | No (public key) |

### Other Services Environment Variables (Context)

| Variable Name | Services Using It | Purpose | Sensitive |
|---|---|---|---|
| `JWT_KEY` | auth, tickets, orders, payments | Shared JWT signing/verification secret | **Yes** |
| `MONGO_URI` | auth, tickets, orders, payments | Per-service MongoDB connection string | **Yes** |
| `NATS_URL` | tickets, orders, payments, expiration | NATS Streaming Server URL | No |
| `NATS_CLUSTER_ID` | tickets, orders, payments, expiration | NATS cluster identifier | No |
| `NATS_CLIENT_ID` | tickets, orders, payments, expiration | NATS client identifier (pod name) | No |
| `REDIS_HOST` | expiration | Redis hostname for Bull queue | No |

---

## Part B: Secrets Management

### Kubernetes Secrets

The application uses Kubernetes native secrets (referenced via `secretKeyRef` in deployment manifests). **No external secrets management services** (AWS Secrets Manager, HashiCorp Vault, Azure Key Vault, GCP Secret Manager) were found in the codebase.

| Secret Name | Key | What It Contains | Where Fetched (K8s manifest:line) | When Fetched | What Breaks If Missing |
|---|---|---|---|---|---|
| `jwt-secret` | `JWT_KEY` | JWT signing secret for authentication tokens | `infra/k8s/payments-depl.yaml:19-23`, `infra/k8s/auth-depl.yaml:19-23`, `infra/k8s/tickets-depl.yaml:19-23`, `infra/k8s/orders-depl.yaml:19-23` | Container startup (injected as env var) | Application fails to start with error `"JWT_KEY must be defined"` (`services/payments/src/index.ts:9`) |
| `stripe-secret` | `STRIPE_KEY` | Stripe API secret key for server-side payment processing | `infra/k8s/payments-depl.yaml:34-38` | Container startup (injected as env var) | Application fails to start with error `"STRIPE_KEY must be defined"` (`services/payments/src/index.ts:13`) |
| `stripe-client-secret` | `STRIPE_CLIENT_KEY` | Stripe publishable key for client-side Stripe.js | `infra/k8s/client-depl.yaml:19-23` | Container startup (injected as env var) | Payment checkout form renders without Stripe key; payments will fail client-side |

### Secret Creation Commands (from README.md:115-131)

```bash
kubectl create secret generic jwt-secret --from-literal=JWT_KEY=<your_secret_key>
kubectl create secret generic stripe-secret --from-literal=STRIPE_KEY=<your_stripe_secret_key>
kubectl create secret generic stripe-client-secret --from-literal=STRIPE_CLIENT_KEY=<your_stripe_client_key>
```

### Test Environment Mock Values

In test environments (`services/payments/src/test/setup.ts:10`):
- `JWT_KEY` is set to `'jwt-key'`
- Stripe client is mocked (`services/payments/src/__mocks__/stripe.ts`)

---

## Part C: Config Files

### Payments Service Configuration

#### `services/payments/tsconfig.json`

| Key | Value | Description |
|---|---|---|
| `compilerOptions.target` | `es2016` | ECMAScript target version |
| `compilerOptions.module` | `commonjs` | Module system |
| `compilerOptions.rootDir` | `./src` | Source directory |
| `compilerOptions.noEmit` | `true` | Type checking only (ts-node handles transpilation) |
| `compilerOptions.esModuleInterop` | `true` | Enables ES module interop |
| `compilerOptions.forceConsistentCasingInFileNames` | `true` | Case-sensitive imports |
| `compilerOptions.strict` | `true` | Strict type checking |
| `compilerOptions.skipLibCheck` | `true` | Skip type checking of declaration files |
| `compilerOptions.types` | `["jest"]` | Include Jest type definitions |

#### `services/payments/jest.config.js`

| Key | Value | Description |
|---|---|---|
| `preset` | `ts-jest` | Uses ts-jest for TypeScript support (from `@repo/jest-config`) |
| `testEnvironment` | `node` | Node.js test environment (from `@repo/jest-config`) |
| `verbose` | `true` | Verbose test output (from `@repo/jest-config`) |
| `modulePaths` | `['<rootDir>/src']` | Module resolution paths (from `@repo/jest-config`) |
| `watchPathIgnorePatterns` | `['globalConfig']` | Ignored paths in watch mode (from `@repo/jest-config`) |
| `setupFilesAfterEnv` | `['./src/test/setup.ts']` | Test setup file for MongoDB memory server and mocks |

#### `services/payments/package.json` (scripts)

| Script | Command | Description |
|---|---|---|
| `dev` | `nodemon src/index.ts` | Development mode with file watching |
| `start` | `ts-node src/index.ts` | Production start |
| `test` | `jest` | Run tests |
| `test:ci` | `jest --ci` | CI test run |
| `test:watch` | `jest --watchAll` | Watch mode testing |
| `typecheck` | `tsc --noEmit` | Type checking |
| `lockfile` | `make-dedicated-lockfile` | Generate dedicated lockfile for Docker build |

### Monorepo Configuration

#### `turbo.json`

| Pipeline Task | Dependencies | Caching | Description |
|---|---|---|---|
| `lockfile` | None | Yes | Generate dedicated lockfiles per service |
| `test` | `typecheck` | Yes | Run tests after type checking passes |
| `test:ci` | None | Yes | CI test run |
| `test:watch` | `typecheck` | No (`cache: false`) | Watch mode testing |
| `typecheck` | None | Yes | Type checking |

#### `pnpm-workspace.yaml`

| Key | Value | Description |
|---|---|---|
| `packages` | `['services/*', 'packages/*', 'apps/*', 'nats-test']` | Workspace package locations |

#### `.nvmrc`

| Setting | Value | Description |
|---|---|---|
| Node version | `20` | Required Node.js version |

### No Environment-Specific Config Files

The codebase does not contain environment-specific configuration files (e.g., `config/development.json`, `config/production.json`). Environment differences are handled entirely through environment variables injected at runtime.

---

## Part D: Infrastructure Configuration

### Docker Configuration

#### `services/payments/Dockerfile`

| Setting | Value | Description |
|---|---|---|
| Base image | `node:20-alpine` | Alpine Linux with Node.js 20 |
| Working directory | `/usr/app` | Application directory |
| Package manager | `pnpm@9` | Installed globally via npm |
| Start command | `pnpm run dev` | Runs nodemon in development mode |

#### `apps/client/Dockerfile`

| Setting | Value | Description |
|---|---|---|
| Base image | `node:alpine` | Alpine Linux with Node.js (unspecified version) |
| Working directory | `/usr/app` | Application directory |
| Package manager | `pnpm@9` | Installed globally via npm |
| Start command | `pnpm run dev` | Runs Next.js in development mode |

### Kubernetes Deployments

#### `infra/k8s/payments-depl.yaml`

| Setting | Value | Description |
|---|---|---|
| Deployment name | `payments-depl` | Kubernetes deployment identifier |
| Replicas | `1` | Single replica |
| Container image | `berkinanik/ticketing-payments:latest` | Docker image |
| Container port | `3000` | HTTP port (implied by liveness probe) |
| Liveness probe path | `/api/payments/health` | Health check endpoint |
| Liveness probe interval | `5s` | Check interval |
| Liveness probe initial delay | `5s` | Startup delay |
| Service name | `payments-srv` | Kubernetes service identifier |
| Service port | `3000` → `3000` | Port mapping |

**Environment Variable Injection:**

| Variable | Source | Value/Reference |
|---|---|---|
| `JWT_KEY` | `secretKeyRef: jwt-secret.JWT_KEY` | From Kubernetes secret |
| `STRIPE_KEY` | `secretKeyRef: stripe-secret.STRIPE_KEY` | From Kubernetes secret |
| `MONGO_URI` | Literal | `mongodb://payments-mongo-srv:27017/payments` |
| `NATS_URL` | Literal | `http://nats-srv:4222` |
| `NATS_CLUSTER_ID` | Literal | `ticketing` |
| `NATS_CLIENT_ID` | `fieldRef: metadata.name` | Kubernetes pod name |

#### `infra/k8s/payments-mongo-depl.yaml`

| Setting | Value | Description |
|---|---|---|
| Container image | `mongo` | Official MongoDB image (latest) |
| Service name | `payments-mongo-srv` | Service DNS name |
| Service port | `27017` | MongoDB default port |

#### `infra/k8s/nats-depl.yaml`

| Setting | Value | Description |
|---|---|---|
| Container image | `nats-streaming:0.17.0` | NATS Streaming Server |
| Client port | `4222` | NATS client connections |
| Monitoring port | `8222` | NATS monitoring endpoint |
| Cluster ID | `ticketing` (via `-cid ticketing` arg) | NATS cluster identifier |
| Heartbeat interval | `5s` (`-hbi 5s`) | Client heartbeat interval |
| Heartbeat timeout | `5s` (`-hbt 5s`) | Heartbeat timeout |
| Heartbeat fail count | `2` (`-hbf 2`) | Max missed heartbeats |
| Debug mode | Enabled (`-SD`) | Standalone debug mode |

#### `infra/k8s/expiration-redis-depl.yaml`

| Setting | Value | Description |
|---|---|---|
| Container image | `redis` | Official Redis image (latest) |
| Service name | `expiration-redis-srv` | Service DNS name |
| Service port | `6379` | Redis default port |

#### `infra/k8s/ingress-srv.yaml`

| Setting | Value | Description |
|---|---|---|
| Ingress class | `nginx` | Uses nginx ingress controller |
| Host | `ticketing.dev` | Development hostname |
| Regex support | Enabled (`use-regex: 'true'`) | Path regex matching |

**Route Mapping:**

| Path | Backend Service | Port |
|---|---|---|
| `/api/users/?(.*)` | `auth-srv` | 3000 |
| `/api/tickets/?(.*)` | `tickets-srv` | 3000 |
| `/api/orders/?(.*)` | `orders-srv` | 3000 |
| `/api/payments/?(.*)` | `payments-srv` | 3000 |
| `/?(.*)` | `client-srv` | 3000 |

### Skaffold Configuration (`skaffold.yaml`)

| Setting | Value | Description |
|---|---|---|
| API version | `skaffold/v2alpha3` | Skaffold schema version |
| Deploy method | `kubectl` | Uses kubectl for deployment |
| Manifests | `infra/k8s/*` | All Kubernetes manifests |
| Local push | `false` | Images not pushed to registry |
| File sync | `src/**/*.ts` → `.` | TypeScript hot-reload for services |

**Build Artifacts:**

| Image | Context | Dockerfile |
|---|---|---|
| `berkinanik/ticketing-auth` | `services/auth` | `Dockerfile` |
| `berkinanik/ticketing-tickets` | `services/tickets` | `Dockerfile` |
| `berkinanik/ticketing-orders` | `services/orders` | `Dockerfile` |
| `berkinanik/ticketing-expiration` | `services/expiration` | `Dockerfile` |
| `berkinanik/ticketing-payments` | `services/payments` | `Dockerfile` |
| `berkinanik/ticketing-client` | `apps/client` | `Dockerfile` |

### CI/CD Configuration (`.github/workflows/tests.yml`)

| Setting | Value | Description |
|---|---|---|
| Trigger | `pull_request` (excluding `infra/**`), `workflow_dispatch` | PR and manual triggers |
| Concurrency group | `dev-ci-${{ github.event.pull_request.number \|\| github.ref }}` | Cancel in-progress runs |
| Runner | `ubuntu-latest` | GitHub-hosted runner |
| Node.js version | `20` | Matches `.nvmrc` |
| Package manager | `pnpm@9` | With cache enabled |

**Job Steps:**

1. Checkout repository
2. Install pnpm@9
3. Install Node.js 20 with pnpm cache
4. Load/cache MongoDB binaries (`~/.cache/mongodb-binaries`)
5. Install dependencies (`pnpm install --frozen-lockfile`)
6. Run typecheck (`pnpm typecheck`)
7. Run tests (`pnpm test:ci`)

---

## Part E: Feature Flags

**No feature flag systems were found in the codebase.**

The following were searched:
- Remote config services (LaunchDarkly, Unleash, Flagsmith, Split.io)
- Feature flag libraries
- Environment-based feature toggles (`FEATURE_*` variables)
- Conditional feature enabling patterns

### Environment-Based Behavioral Toggles (Not Feature Flags)

| Pattern | Location | Description |
|---|---|---|
| `NODE_ENV !== 'test'` | `services/payments/src/app.ts:23` | Controls cookie-session `secure` flag; disabled during tests |

### Hardcoded Configuration Values

| Constant | Value | Location | Description |
|---|---|---|
| `EXPIRATION_WINDOW_SECONDS` | `900` (15 minutes) | `services/orders/src/routes/new.ts:16` | Order expiration timeout; hardcoded, not configurable via env |
| Stripe API version | `2024-04-10` | `services/payments/src/stripe.ts:4` | Stripe API version; hardcoded |
| HTTP port | `3000` | `services/payments/src/index.ts:58` | Server port; hardcoded |
| Queue name | `order:expiration` | `services/expiration/src/queues/expiration-queue.ts:9` | Bull queue name; hardcoded |
| NATS queue group | `payments-service` | `services/payments/src/events/listeners/queue-group-name.ts` | NATS queue group name (inferred from standard pattern) |

---

## Summary: Payments Service Configuration Matrix

| Category | Item | Source | Required at Runtime |
|---|---|---|---|
| **Secrets** | `JWT_KEY` | K8s secret `jwt-secret` | Yes |
| **Secrets** | `STRIPE_KEY` | K8s secret `stripe-secret` | Yes |
| **Database** | `MONGO_URI` | K8s deployment literal | Yes |
| **Messaging** | `NATS_URL` | K8s deployment literal | Yes |
| **Messaging** | `NATS_CLUSTER_ID` | K8s deployment literal | Yes |
| **Messaging** | `NATS_CLIENT_ID` | K8s pod metadata | Yes (auto-populated) |
| **Runtime** | `NODE_ENV` | Not set in K8s (defaults to undefined) | No |
