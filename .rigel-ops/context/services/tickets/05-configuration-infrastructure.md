---
service_id: ticketing-app/tickets
service_name: tickets
prompt_id: 05-configuration-infrastructure
prompt_label: Configuration & Infrastructure
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.22953
schema_version: "2.0"
---

# Configuration Analysis: Tickets Service

## Part A: Environment Variables

### Tickets Service Environment Variables

| Variable Name | Used In (file:line) | Purpose | Required | Default Value | Sensitive |
|---|---|---|---|---|---|
| `JWT_KEY` | `services/tickets/src/index.ts:8` (validation), `services/tickets/src/test/setup.ts:9` (test), `services/tickets/src/test/utils.ts:16` (test) | Secret key for JWT token verification used by the `@b.anik/ticketing-common` middleware to validate authentication cookies | Yes | None | Yes |
| `MONGO_URI` | `services/tickets/src/index.ts:12` (validation), `services/tickets/src/index.ts:47` (connection) | MongoDB connection string for the tickets database | Yes | None (K8s default: `mongodb://tickets-mongo-srv:27017/tickets`) | Yes |
| `NATS_URL` | `services/tickets/src/index.ts:16` (validation), `services/tickets/src/index.ts:32` (connection) | NATS Streaming Server connection URL | Yes | None (K8s default: `http://nats-srv:4222`) | No |
| `NATS_CLUSTER_ID` | `services/tickets/src/index.ts:20` (validation), `services/tickets/src/index.ts:30` (connection) | NATS cluster identifier for the streaming server | Yes | None (K8s default: `ticketing`) | No |
| `NATS_CLIENT_ID` | `services/tickets/src/index.ts:31` (connection) | Unique client identifier for NATS connection (per pod) | Yes | None (K8s: injected from pod `metadata.name`) | No |
| `NODE_ENV` | `services/tickets/src/app.ts:29` | Determines cookie-session `secure` flag behavior; when set to `'test'`, secure cookies are disabled | No | None | No |

### Environment Variable Validation

At startup (`services/tickets/src/index.ts:7-26`), the service performs synchronous validation and throws errors for missing required variables:

```typescript
if (!process.env.JWT_KEY) {
  throw new Error('JWT_KEY must be defined');
}
if (!process.env.MONGO_URI) {
  throw new Error('MONGO_URI must be defined');
}
if (!process.env.NATS_URL) {
  throw new Error('NATS_URL must be defined');
}
if (!process.env.NATS_CLUSTER_ID) {
  throw new Error('NATS_CLUSTER_ID must be defined');
}
```

**Note**: There is a duplicate check for `NATS_CLUSTER_ID` at lines 20-22 and 24-26 (appears to be a copy-paste error; the second check should likely validate `NATS_CLIENT_ID`). The `NATS_CLIENT_ID` variable is used at line 31 but never explicitly validated — a missing value would cause a runtime error during NATS connection.

---

## Part B: Secrets Management

### Kubernetes Secrets

The tickets service uses Kubernetes secrets for sensitive configuration. No external secrets management services (AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager, Azure Key Vault, etc.) are used in this codebase.

| Secret Name | Key | What It Contains | Where It's Fetched | When It's Fetched | What Breaks If Missing |
|---|---|---|---|---|---|
| `jwt-secret` | `JWT_KEY` | Symmetric secret key for JWT token signing/verification | `infra/k8s/tickets-depl.yaml:19-23` → injected as `JWT_KEY` env var | Application startup (pod initialization) | Application throws `Error('JWT_KEY must be defined')` and exits immediately (`services/tickets/src/index.ts:8-10`); all authentication fails |

### Secret Creation Instructions

From `README.md:117-118`:
```bash
kubectl create secret generic jwt-secret --from-literal=JWT_KEY=<your_secret_key>
```

### Other Secrets in the Platform (Not Used by Tickets Service)

| Secret Name | Used By | Purpose |
|---|---|---|
| `stripe-secret` (key: `STRIPE_KEY`) | payments service | Stripe API secret key for payment processing |
| `stripe-client-secret` (key: `STRIPE_CLIENT_KEY`) | client app | Stripe publishable key for frontend checkout |

---

## Part C: Config Files

### No Traditional Config Directory

The tickets service does not use a `config/`, `settings/`, or equivalent configuration directory. Configuration is managed through:

1. **Environment variables** (described in Part A)
2. **Package manifest and build configuration files** (described below)

### Configuration Files in `services/tickets/`

#### `package.json`

**Purpose**: Package manifest defining dependencies, scripts, and metadata.

| Key | Value | Description |
|---|---|---|
| `name` | `tickets` | Package identifier in the monorepo |
| `version` | `1.0.0` | Service version |
| `scripts.dev` | `nodemon src/index.ts` | Development mode with auto-reload |
| `scripts.start` | `ts-node src/index.ts` | Production start (TypeScript direct execution) |
| `scripts.test` | `jest` | Run tests |
| `scripts.test:ci` | `jest --ci` | Run tests in CI mode |
| `scripts.typecheck` | `tsc --noEmit` | TypeScript type checking |

**Environment-specific**: No environment-specific overrides in package.json.

#### `tsconfig.json`

**Purpose**: TypeScript compiler configuration.

| Key | Value | Description |
|---|---|---|
| `compilerOptions.target` | `es2016` | ECMAScript target version |
| `compilerOptions.module` | `commonjs` | Module system for Node.js |
| `compilerOptions.rootDir` | `./src` | Source files directory |
| `compilerOptions.noEmit` | `true` | Type-check only, no output files |
| `compilerOptions.esModuleInterop` | `true` | Enable ES module interop |
| `compilerOptions.strict` | `true` | Enable strict type checking |
| `compilerOptions.skipLibCheck` | `true` | Skip type checking of declaration files |
| `compilerOptions.types` | `["jest"]` | Include Jest type definitions |

#### `jest.config.js`

**Purpose**: Jest test framework configuration.

| Key | Value | Description |
|---|---|---|
| `...baseConfig` | Inherited from `@repo/jest-config` | Shared test configuration |
| `setupFilesAfterEnv` | `['./src/test/setup.ts']` | Test environment setup file |

**Inherited from `packages/jest-config/jest.config.js`**:

| Key | Value | Description |
|---|---|---|
| `preset` | `ts-jest` | Use ts-jest for TypeScript |
| `testEnvironment` | `node` | Node.js test environment |
| `verbose` | `true` | Verbose test output |
| `modulePaths` | `['<rootDir>/src']` | Module resolution paths |
| `watchPathIgnorePatterns` | `['globalConfig']` | Paths to ignore in watch mode |

#### `.dockerignore`

**Purpose**: Files excluded from Docker build context.

Excluded patterns:
- `node_modules`
- `dist`
- `build`
- `.turbo`

---

## Part D: Infrastructure Configuration

### Dockerfile (`services/tickets/Dockerfile`)

| Setting | Value | Description |
|---|---|---|
| Base image | `node:20-alpine` | Node.js 20 on Alpine Linux |
| Working directory | `/usr/app` | Application root in container |
| Package manager | `pnpm@9` | Installed globally |
| Startup command | `pnpm run dev` | Runs in development mode (uses nodemon) |

**Build steps**:
1. Copy `package.json` and `pnpm-lock.yaml`
2. Run `pnpm i` to install dependencies
3. Copy all source files
4. Start with `pnpm run dev`

### Kubernetes Deployment (`infra/k8s/tickets-depl.yaml`)

#### Deployment Configuration

| Setting | Value | Description |
|---|---|---|
| `metadata.name` | `tickets-depl` | Deployment name |
| `spec.replicas` | `1` | Number of pod replicas |
| `spec.selector.matchLabels.app` | `tickets` | Pod selector |
| `containers[0].name` | `tickets` | Container name |
| `containers[0].image` | `berkinanik/ticketing-tickets:latest` | Docker image |

#### Environment Variable Injection

| Env Var | Source | Value/Reference |
|---|---|---|
| `JWT_KEY` | `secretKeyRef` | `jwt-secret.JWT_KEY` |
| `MONGO_URI` | Literal | `mongodb://tickets-mongo-srv:27017/tickets` |
| `NATS_URL` | Literal | `http://nats-srv:4222` |
| `NATS_CLUSTER_ID` | Literal | `ticketing` |
| `NATS_CLIENT_ID` | `fieldRef` | `metadata.name` (pod name, ensures uniqueness) |

#### Liveness Probe

| Setting | Value |
|---|---|
| Type | HTTP GET |
| Path | `/api/tickets/health` |
| Port | `3000` |
| Initial delay | `5s` |
| Period | `5s` |

### Kubernetes Service (`infra/k8s/tickets-depl.yaml:41-53`)

| Setting | Value | Description |
|---|---|---|
| `metadata.name` | `tickets-srv` | Service name |
| `spec.selector.app` | `tickets` | Targets pods with `app: tickets` label |
| `spec.ports[0].protocol` | `TCP` | Network protocol |
| `spec.ports[0].port` | `3000` | Service port |
| `spec.ports[0].targetPort` | `3000` | Container port |

### Kubernetes MongoDB (`infra/k8s/tickets-mongo-depl.yaml`)

#### Deployment

| Setting | Value |
|---|---|
| `metadata.name` | `tickets-mongo-depl` |
| `spec.replicas` | `1` |
| `containers[0].image` | `mongo` (latest) |

#### Service

| Setting | Value |
|---|---|
| `metadata.name` | `tickets-mongo-srv` |
| `spec.ports[0].port` | `27017` |
| `spec.ports[0].targetPort` | `27017` |

### Ingress Configuration (`infra/k8s/ingress-srv.yaml:20-26`)

| Setting | Value | Description |
|---|---|---|
| Host | `ticketing.dev` | Virtual host for routing |
| Path | `/api/tickets/?(.*)` | URL pattern (regex enabled) |
| Path type | `Prefix` | Match type |
| Backend service | `tickets-srv` | Target service |
| Backend port | `3000` | Service port |

**Annotations**:
- `kubernetes.io/ingress.class: nginx` — Uses nginx ingress controller
- `nginx.ingress.kubernetes.io/use-regex: 'true'` — Enables regex path matching

### Skaffold Configuration (`skaffold.yaml:19-26`)

| Setting | Value |
|---|---|
| Image name | `berkinanik/ticketing-tickets` |
| Build context | `services/tickets` |
| Dockerfile | `Dockerfile` |
| Sync manual | `src/**/*.ts` → `.` |
| Push | `false` (local development) |

### Turbo Build Configuration (`turbo.json`)

| Pipeline | Dependencies | Cache |
|---|---|---|
| `lockfile` | None | Enabled |
| `test` | `typecheck` | Enabled |
| `test:ci` | None | Enabled |
| `test:watch` | `typecheck` | Disabled |
| `typecheck` | None | Enabled |

### CI/CD Configuration (`.github/workflows/tests.yml`)

| Setting | Value |
|---|---|
| Trigger | Pull requests (except `infra/**`), manual dispatch |
| Runner | `ubuntu-latest` |
| Node version | `20` |
| Package manager | `pnpm@9` |
| Caching | pnpm cache, MongoDB binaries |
| Steps | Checkout → Install pnpm → Setup Node → Cache MongoDB → Install deps → Typecheck → Test |

**Concurrency**: Groups by PR number or ref, cancels in-progress runs.

### Application Runtime Configuration

| Setting | Value | Source |
|---|---|---|
| HTTP port | `3000` | Hardcoded in `services/tickets/src/index.ts:54` |
| Trust proxy | `true` | `services/tickets/src/app.ts:22` |
| Cookie session signed | `false` | `services/tickets/src/app.ts:28` |
| Cookie session secure | `process.env.NODE_ENV !== 'test'` | `services/tickets/src/app.ts:29` |

---

## Part E: Feature Flags

### No Feature Flags Detected

The tickets service does not implement any feature flag system. The codebase was searched for:

- Feature flag service integrations (LaunchDarkly, Split.io, Optimizely, Flagsmith, Unleash, ConfigCat) — **none found**
- Environment-based feature toggles (`FEATURE_*`, `isEnabled`, `toggle`) — **none found**
- Conditional feature enabling patterns — **none found**

The only environment-based conditional logic is the cookie session `secure` flag (`services/tickets/src/app.ts:29`), which disables secure cookies in test environments:

```typescript
secure: process.env.NODE_ENV !== 'test'
```

This is a test infrastructure concern, not a feature flag.

---

## Summary Table: All Environment Variables Across the Platform

For context, here are all environment variables used across the ticketing platform (not just the tickets service):

| Variable | Services Using It | Purpose |
|---|---|---|
| `JWT_KEY` | auth, tickets, orders, payments | JWT token signing/verification |
| `MONGO_URI` | auth, tickets, orders, payments | MongoDB connection string |
| `NATS_URL` | tickets, orders, payments, expiration | NATS Streaming server URL |
| `NATS_CLUSTER_ID` | tickets, orders, payments, expiration | NATS cluster identifier |
| `NATS_CLIENT_ID` | tickets, orders, payments, expiration | Unique NATS client ID |
| `NODE_ENV` | auth, tickets, orders, payments | Runtime environment detection |
| `STRIPE_KEY` | payments | Stripe secret API key |
| `STRIPE_CLIENT_KEY` | client | Stripe publishable key |
| `REDIS_HOST` | expiration | Redis server hostname |
