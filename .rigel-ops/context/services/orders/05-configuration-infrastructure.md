---
service_id: ticketing-app/orders
service_name: orders
prompt_id: 05-configuration-infrastructure
prompt_label: Configuration & Infrastructure
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.24842
schema_version: "2.0"
---

# Configuration & Environment Analysis: Orders Service

## Part A: Environment Variables

### Orders Service (`services/orders/`)

| Variable Name | Used In (file:line) | Purpose | Required | Default Value | Sensitive |
|---|---|---|---|---|---|
| `JWT_KEY` | `src/index.ts:8`, `src/test/setup.ts:9`, `src/test/utils.ts:16` | Secret key for JWT token validation/signing (session authentication) | Yes (startup check at line 8-10) | None | Yes |
| `MONGO_URI` | `src/index.ts:12`, `src/index.ts:47` | MongoDB connection string for the orders database | Yes (startup check at line 12-14) | None | Yes (contains connection credentials in production) |
| `NATS_URL` | `src/index.ts:16`, `src/index.ts:32` | URL of the NATS Streaming Server for event messaging | Yes (startup check at line 16-18) | None | No |
| `NATS_CLUSTER_ID` | `src/index.ts:20`, `src/index.ts:24`, `src/index.ts:30` | NATS Streaming cluster identifier | Yes (startup checks at lines 20-26) | None | No |
| `NATS_CLIENT_ID` | `src/index.ts:31` | Unique client identifier for NATS connection (derived from pod name in K8s) | Yes (used but not explicitly validated) | None | No |
| `NODE_ENV` | `src/app.ts:23` | Runtime environment; controls cookie session `secure` flag | No | None | No |

**Note:** There is a duplicated validation check for `NATS_CLUSTER_ID` at lines 20-22 and 24-26 in `src/index.ts` (appears to be a copy-paste bug; the second should likely check `NATS_CLIENT_ID`).

### Test Environment Overrides

| Variable Name | Set In (file:line) | Test Value | Purpose |
|---|---|---|---|
| `JWT_KEY` | `src/test/setup.ts:9` | `'jwt-key'` | Provides deterministic JWT key for test authentication |

---

## Part B: Secrets Management

### Kubernetes Secrets

The Orders service retrieves secrets exclusively through Kubernetes `secretKeyRef` mechanism. No external secrets managers (AWS Secrets Manager, HashiCorp Vault, Azure Key Vault, etc.) are used.

| Secret Name | Key | Contains | Fetched In | When Fetched | What Breaks If Missing |
|---|---|---|---|---|---|
| `jwt-secret` | `JWT_KEY` | JWT signing/verification secret key | `infra/k8s/orders-depl.yaml:21-23` | Pod startup (injected as env var) | Service crashes at startup with "JWT_KEY must be defined" (`src/index.ts:9`) |

### Secret Creation Commands

From `README.md:117-119`:
```bash
kubectl create secret generic jwt-secret --from-literal=JWT_KEY=<your_secret_key>
```

### Additional Cluster-Wide Secrets (used by other services)

| Secret Name | Key | Service Using It |
|---|---|---|
| `stripe-secret` | `STRIPE_KEY` | payments |
| `stripe-client-secret` | `STRIPE_CLIENT_KEY` | client |

---

## Part C: Config Files

### `services/orders/package.json`

**Purpose:** Package manifest defining dependencies, scripts, and metadata.

| Key | Value | Description |
|---|---|---|
| `name` | `orders` | Package name |
| `version` | `1.0.0` | Package version |
| `main` | `index.js` | Entry point (unused; TypeScript runs via ts-node) |
| `scripts.dev` | `nodemon src/index.ts` | Development server with hot reload |
| `scripts.start` | `ts-node src/index.ts` | Production start command |
| `scripts.test` | `jest` | Run tests |
| `scripts.test:ci` | `jest --ci` | CI test run |
| `scripts.test:watch` | `jest --watchAll` | Watch mode tests |
| `scripts.typecheck` | `tsc --noEmit` | TypeScript type checking |
| `scripts.lockfile` | `make-dedicated-lockfile` | Generate standalone lockfile for Docker builds |

### `services/orders/tsconfig.json`

**Purpose:** TypeScript compiler configuration.

| Key | Value | Description |
|---|---|---|
| `compilerOptions.target` | `es2016` | ECMAScript target version |
| `compilerOptions.module` | `commonjs` | Module system |
| `compilerOptions.rootDir` | `./src` | Source root directory |
| `compilerOptions.noEmit` | `true` | Type-check only, no JS output |
| `compilerOptions.esModuleInterop` | `true` | CommonJS/ES module interop |
| `compilerOptions.forceConsistentCasingInFileNames` | `true` | Enforce consistent file casing |
| `compilerOptions.strict` | `true` | Enable all strict type checks |
| `compilerOptions.skipLibCheck` | `true` | Skip type checking declaration files |
| `compilerOptions.types` | `["jest"]` | Include Jest type definitions |

### `services/orders/jest.config.js`

**Purpose:** Jest test runner configuration.

| Key | Value | Description |
|---|---|---|
| `...baseConfig` | (from `@repo/jest-config`) | Inherits shared Jest config |
| `setupFilesAfterEnv` | `['./src/test/setup.ts']` | Test setup file path |

### `packages/jest-config/jest.config.js`

**Purpose:** Shared Jest configuration for all services.

| Key | Value | Description |
|---|---|---|
| `preset` | `ts-jest` | Use ts-jest preset |
| `testEnvironment` | `node` | Node.js test environment |
| `verbose` | `true` | Verbose test output |
| `modulePaths` | `['<rootDir>/src']` | Module resolution paths |
| `watchPathIgnorePatterns` | `['globalConfig']` | Paths to ignore in watch mode |

### Root-Level Config Files

#### `turbo.json`

**Purpose:** Turborepo pipeline configuration for monorepo task orchestration.

| Pipeline Task | Configuration | Description |
|---|---|---|
| `lockfile` | `{}` | Generate lockfiles (no dependencies) |
| `test` | `dependsOn: ["typecheck"]` | Tests depend on type checking |
| `test:ci` | `{}` | CI tests (no dependencies) |
| `test:watch` | `dependsOn: ["typecheck"], cache: false` | Watch mode (no caching) |
| `typecheck` | `{}` | Type checking (no dependencies) |

#### `pnpm-workspace.yaml`

**Purpose:** pnpm workspace configuration.

| Key | Value |
|---|---|
| `packages` | `['services/*', 'packages/*', 'apps/*', 'nats-test']` |

---

## Part D: Infrastructure Configuration

### Dockerfile (`services/orders/Dockerfile`)

**Purpose:** Container build definition for the Orders service.

| Instruction | Value | Description |
|---|---|---|
| `FROM` | `node:20-alpine` | Base image: Node.js 20 on Alpine Linux |
| `WORKDIR` | `/usr/app` | Working directory in container |
| `RUN` | `npm i -g pnpm@9` | Install pnpm globally |
| `COPY` | `package.json .`, `pnpm-lock.yaml .` | Copy dependency manifests |
| `RUN` | `pnpm i` | Install dependencies |
| `COPY` | `. .` | Copy application source |
| `CMD` | `["pnpm", "run", "dev"]` | Default command (dev mode with nodemon) |

**Note:** The Dockerfile uses `pnpm run dev` which runs nodemon for hot-reloading. In production, this should be changed to `pnpm run start` or a compiled version.

### Kubernetes Deployment (`infra/k8s/orders-depl.yaml`)

**Purpose:** Kubernetes Deployment and Service for Orders.

#### Deployment Configuration

| Setting | Value | Description |
|---|---|---|
| `metadata.name` | `orders-depl` | Deployment name |
| `spec.replicas` | `1` | Number of pod replicas |
| `spec.selector.matchLabels.app` | `orders` | Pod selector label |
| `containers[0].name` | `orders` | Container name |
| `containers[0].image` | `berkinanik/ticketing-orders:latest` | Container image |

#### Environment Variable Injection

| Env Var | Source | Value/Reference |
|---|---|---|
| `JWT_KEY` | `secretKeyRef` | `jwt-secret.JWT_KEY` |
| `MONGO_URI` | Literal | `mongodb://orders-mongo-srv:27017/orders` |
| `NATS_URL` | Literal | `http://nats-srv:4222` |
| `NATS_CLUSTER_ID` | Literal | `ticketing` |
| `NATS_CLIENT_ID` | `fieldRef` | `metadata.name` (pod name, ensures uniqueness) |

#### Liveness Probe

| Setting | Value |
|---|---|
| `httpGet.path` | `/api/orders/health` |
| `httpGet.port` | `3000` |
| `initialDelaySeconds` | `5` |
| `periodSeconds` | `5` |

#### Service Configuration

| Setting | Value |
|---|---|
| `metadata.name` | `orders-srv` |
| `spec.selector.app` | `orders` |
| `ports[0].name` | `orders` |
| `ports[0].protocol` | `TCP` |
| `ports[0].port` | `3000` |
| `ports[0].targetPort` | `3000` |

### Kubernetes MongoDB Deployment (`infra/k8s/orders-mongo-depl.yaml`)

**Purpose:** MongoDB instance for the Orders service.

| Setting | Value | Description |
|---|---|---|
| `metadata.name` | `orders-mongo-depl` | Deployment name |
| `spec.replicas` | `1` | Single replica |
| `containers[0].image` | `mongo` | Official MongoDB image (latest) |
| `service.name` | `orders-mongo-srv` | Service name |
| `service.port` | `27017` | MongoDB default port |

**Note:** No persistent volume is configured; data is ephemeral.

### NATS Streaming Server (`infra/k8s/nats-depl.yaml`)

**Purpose:** Event message broker shared by all services.

| Setting | Value | Description |
|---|---|---|
| `containers[0].image` | `nats-streaming:0.17.0` | NATS Streaming version |
| `args` | Various | Configuration flags |
| `-p` | `4222` | Client port |
| `-m` | `8222` | Monitoring port |
| `-hbi` | `5s` | Heartbeat interval |
| `-hbt` | `5s` | Heartbeat timeout |
| `-hbf` | `2` | Heartbeat fail count |
| `-SD` | (flag) | Debug mode |
| `-cid` | `ticketing` | Cluster ID |
| `service.ports[0]` | `4222` (client) | Client connections |
| `service.ports[1]` | `8222` (monitoring) | Monitoring endpoint |

### Ingress Configuration (`infra/k8s/ingress-srv.yaml`)

**Purpose:** Nginx ingress controller routing rules.

| Setting | Value |
|---|---|
| `spec.rules[0].host` | `ticketing.dev` |
| `annotations.kubernetes.io/ingress.class` | `nginx` |
| `annotations.nginx.ingress.kubernetes.io/use-regex` | `true` |

#### Route Mappings

| Path Pattern | Backend Service | Port |
|---|---|---|
| `/api/users/?(.*)` | `auth-srv` | `3000` |
| `/api/tickets/?(.*)` | `tickets-srv` | `3000` |
| `/api/orders/?(.*)` | `orders-srv` | `3000` |
| `/api/payments/?(.*)` | `payments-srv` | `3000` |
| `/?(.*)` | `client-srv` | `3000` |

### Skaffold Configuration (`skaffold.yaml`)

**Purpose:** Local Kubernetes development workflow.

| Setting | Value |
|---|---|
| `apiVersion` | `skaffold/v2alpha3` |
| `deploy.kubectl.manifests` | `infra/k8s/*` |
| `build.local.push` | `false` |

#### Orders Service Build Artifact

| Setting | Value |
|---|---|
| `image` | `berkinanik/ticketing-orders` |
| `context` | `services/orders` |
| `docker.dockerfile` | `Dockerfile` |
| `sync.manual[0].src` | `src/**/*.ts` |
| `sync.manual[0].dest` | `.` |

### CI/CD Configuration (`.github/workflows/tests.yml`)

**Purpose:** GitHub Actions workflow for automated testing.

| Setting | Value | Description |
|---|---|---|
| `on.pull_request` | `paths-ignore: ['infra/**']` | Triggers on PRs, ignores infra changes |
| `on.workflow_dispatch` | `{}` | Manual trigger enabled |
| `concurrency.group` | `dev-ci-${{ github.event.pull_request.number || github.ref }}` | Prevents concurrent runs |
| `concurrency.cancel-in-progress` | `true` | Cancels in-progress runs |
| `permissions.actions` | `write` | Actions permission |
| `permissions.contents` | `read` | Read repository |

#### Job Steps

| Step | Tool/Action | Configuration |
|---|---|---|
| Checkout | `actions/checkout@v4` | - |
| Install pnpm | `pnpm/action-setup@v3` | `version: 9` |
| Install Node.js | `actions/setup-node@v4` | `node-version: 20`, `cache: 'pnpm'` |
| MongoDB cache | `actions/cache@v4` | `path: ~/.cache/mongodb-binaries` |
| Install deps | `pnpm install` | `--frozen-lockfile` |
| Typecheck | `pnpm typecheck` | - |
| Tests | `pnpm test:ci` | - |

---

## Part E: Feature Flags

**No feature flag systems detected** in the Orders service or the broader codebase.

The search for common feature flag patterns (LaunchDarkly, Unleash, Flagsmith, feature toggles, `FEATURE_*` environment variables) returned no relevant matches.

---

## Additional Hardcoded Configuration Values

These are compile-time constants found in the Orders service source code:

| Constant | Value | File:Line | Purpose |
|---|---|---|---|
| `EXPIRATION_WINDOW_SECONDS` | `900` (15 * 60) | `src/routes/new.ts:16` | Order reservation expiration window (15 minutes) |
| HTTP Port | `3000` | `src/index.ts:54` | Express server listening port |
| `queueGroupName` | `'orders-service'` | `src/events/listeners/queue-group-name.ts:1` | NATS queue group for load-balanced message delivery |
| Cookie session `signed` | `false` | `src/app.ts:22` | Cookies are not signed (JWT provides integrity) |
| Cookie session `secure` | `process.env.NODE_ENV !== 'test'` | `src/app.ts:23` | HTTPS-only cookies except in test environment |
| Trust proxy | `true` | `src/app.ts:16` | Trust X-Forwarded-* headers from ingress |

---

## Runtime Configuration Summary

### Required Environment Variables for Production

```
JWT_KEY=<kubernetes-secret>
MONGO_URI=mongodb://orders-mongo-srv:27017/orders
NATS_URL=http://nats-srv:4222
NATS_CLUSTER_ID=ticketing
NATS_CLIENT_ID=<pod-name>
```

### Service Dependencies

| Dependency | Internal Service Name | Port | Protocol |
|---|---|---|---|
| MongoDB | `orders-mongo-srv` | `27017` | MongoDB wire protocol |
| NATS Streaming | `nats-srv` | `4222` | NATS protocol |

### Network Topology

- Orders service listens on port `3000`
- Exposed via Kubernetes Service `orders-srv:3000`
- Routed externally via Ingress at path `/api/orders/*`
- Host: `ticketing.dev` (requires `/etc/hosts` entry for local development)
