---
service_id: ticketing-app/auth
service_name: auth
prompt_id: 05-configuration-infrastructure
prompt_label: Configuration & Infrastructure
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.1993
schema_version: "2.0"
---

# Configuration Analysis: Auth Service

## Part A: Environment Variables

The auth service accesses environment variables in the following locations:

| Variable Name | Used In (file:line) | Purpose | Required | Default Value | Sensitive |
|---------------|---------------------|---------|----------|---------------|-----------|
| `JWT_KEY` | `services/auth/src/index.ts:6-8` | Secret key used to sign JWTs for user authentication | Yes | None (throws error if missing) | **Yes** |
| `JWT_KEY` | `services/auth/src/routes/sign-in.ts:37` | Signs JWT containing user `id` and `email` during login | Yes | — | **Yes** |
| `JWT_KEY` | `services/auth/src/routes/sign-up.ts:42` | Signs JWT containing user `id` and `email` during registration | Yes | — | **Yes** |
| `JWT_KEY` | `services/auth/src/test/setup.ts:7` | Set to test value `jwt-key` for test environment | — | `jwt-key` (test only) | **Yes** |
| `MONGO_URI` | `services/auth/src/index.ts:10-12, 15` | MongoDB connection string for user data storage | Yes | None (throws error if missing) | **Yes** |
| `NODE_ENV` | `services/auth/src/app.ts:30` | Determines if cookies should use `secure: true` (disabled when `NODE_ENV === 'test'`) | No | Implicitly `undefined` in production | No |

### Environment Variable Validation

The service performs startup validation in `services/auth/src/index.ts:5-12`:

```
if (!process.env.JWT_KEY) {
  throw new Error('JWT_KEY must be defined');
}

if (!process.env.MONGO_URI) {
  throw new Error('MONGO_URI must be defined');
}
```

If either required variable is missing, the application throws an error and fails to start.

---

## Part B: Secrets Management

### Kubernetes Secrets

The auth service uses Kubernetes-native secret management. No external secrets manager (Vault, AWS Secrets Manager, Azure Key Vault, etc.) is used.

| Secret Name | Secret Key | What It Contains | Where It's Fetched | When It's Fetched | What Breaks If Missing |
|-------------|------------|------------------|-------------------|-------------------|------------------------|
| `jwt-secret` | `JWT_KEY` | Secret key for signing JWT tokens | `infra/k8s/auth-depl.yaml:19-23` (Kubernetes secretKeyRef) | Pod startup (injected as env var) | Application throws error at startup; all authentication endpoints fail |

### Secret Creation

Per `README.md:117-118`, the Kubernetes secret must be created manually before deployment:

```bash
kubectl create secret generic jwt-secret --from-literal=JWT_KEY=<your_secret_key>
```

### Secret Consumption Pattern

In `infra/k8s/auth-depl.yaml:18-23`:
```yaml
env:
  - name: JWT_KEY
    valueFrom:
      secretKeyRef:
        name: jwt-secret
        key: JWT_KEY
```

The secret is injected as an environment variable at pod startup time, then accessed via `process.env.JWT_KEY` at runtime.

---

## Part C: Config Files

### `services/auth/package.json`

**Purpose:** Node.js package manifest defining dependencies, scripts, and metadata.

| Key | Value | Description |
|-----|-------|-------------|
| `name` | `auth` | Package identifier |
| `version` | `1.0.0` | Package version |
| `main` | `index.js` | Entry point (not used; ts-node runs `.ts` files) |
| `scripts.dev` | `nodemon src/index.ts` | Development server with hot reload |
| `scripts.start` | `ts-node src/index.ts` | Production startup command |
| `scripts.test` | `jest` | Run tests |
| `scripts.test:ci` | `jest --ci` | CI-mode tests (no watch, fail on first error) |
| `scripts.test:watch` | `jest --watchAll` | Watch mode tests |
| `scripts.typecheck` | `tsc --noEmit` | Type check without emit |
| `scripts.lockfile` | `make-dedicated-lockfile` | Generate dedicated lockfile for Docker builds |

**Environment-specific overrides:** None — all configuration is identical across environments.

### `services/auth/tsconfig.json`

**Purpose:** TypeScript compiler configuration.

| Key | Value | Description |
|-----|-------|-------------|
| `compilerOptions.target` | `es2016` | ECMAScript target version |
| `compilerOptions.module` | `commonjs` | Module system (Node.js compatible) |
| `compilerOptions.rootDir` | `./src` | Source root directory |
| `compilerOptions.noEmit` | `true` | Don't emit JS files (ts-node handles execution) |
| `compilerOptions.esModuleInterop` | `true` | Enables default imports from CommonJS |
| `compilerOptions.forceConsistentCasingInFileNames` | `true` | Enforce consistent file name casing |
| `compilerOptions.strict` | `true` | Enable all strict type checking options |
| `compilerOptions.skipLibCheck` | `true` | Skip type checking of declaration files |
| `compilerOptions.types` | `["jest"]` | Include Jest type definitions |

### `services/auth/jest.config.js`

**Purpose:** Jest test runner configuration.

| Key | Value | Description |
|-----|-------|-------------|
| `...baseConfig` | (from `@repo/jest-config`) | Inherits shared config: `ts-jest` preset, `node` environment, verbose output |
| `setupFilesAfterEnv` | `['./src/test/setup.ts']` | Test setup file that starts MongoMemoryServer |

### `packages/jest-config/jest.config.js` (Shared Base Config)

| Key | Value | Description |
|-----|-------|-------------|
| `preset` | `ts-jest` | Use ts-jest for TypeScript support |
| `testEnvironment` | `node` | Run tests in Node.js environment |
| `verbose` | `true` | Display individual test results |
| `modulePaths` | `['<rootDir>/src']` | Module resolution paths |
| `watchPathIgnorePatterns` | `['globalConfig']` | Ignore paths during watch mode |

### `turbo.json` (Root)

**Purpose:** Turborepo pipeline configuration for monorepo orchestration.

| Pipeline | Configuration | Description |
|----------|---------------|-------------|
| `lockfile` | `{}` | No dependencies |
| `test` | `dependsOn: ["typecheck"]` | Run typecheck before tests |
| `test:ci` | `{}` | Independent task |
| `test:watch` | `dependsOn: ["typecheck"], cache: false` | No caching for watch mode |
| `typecheck` | `{}` | Independent task |

---

## Part D: Infrastructure Configuration

### `services/auth/Dockerfile`

**Purpose:** Multi-stage Docker build for the auth service container.

| Setting | Value | Description |
|---------|-------|-------------|
| Base image | `node:20-alpine` | Lightweight Node.js 20 Alpine image |
| Working directory | `/usr/app` | Application root inside container |
| Package manager | `pnpm@9` (installed globally) | pnpm version 9 |
| Start command | `pnpm run dev` | Runs nodemon for development |

**Note:** The Dockerfile uses `pnpm run dev` (with nodemon) rather than `pnpm start`, indicating it's primarily used for development via Skaffold. For production, a separate build process would be needed.

### `services/auth/.dockerignore`

| Excluded Path | Reason |
|---------------|--------|
| `node_modules` | Dependencies installed inside container |
| `dist` | Build artifacts |
| `build` | Build artifacts |
| `.turbo` | Turborepo cache |

### `infra/k8s/auth-depl.yaml`

**Purpose:** Kubernetes Deployment and Service for the auth microservice.

#### Deployment Configuration

| Setting | Value | Description |
|---------|-------|-------------|
| `metadata.name` | `auth-depl` | Deployment name |
| `spec.replicas` | `1` | Single replica |
| `spec.selector.matchLabels.app` | `auth` | Pod selector |
| Container name | `auth` | Container identifier |
| Container image | `berkinanik/ticketing-auth:latest` | Docker Hub image |

#### Environment Variables Injected

| Variable | Source | Value |
|----------|--------|-------|
| `JWT_KEY` | `secretKeyRef: jwt-secret.JWT_KEY` | From Kubernetes secret |
| `MONGO_URI` | Hardcoded | `mongodb://auth-mongo-srv:27017/auth` |

#### Liveness Probe

| Setting | Value |
|---------|-------|
| Path | `/api/users/health` |
| Port | `3000` |
| Initial delay | `5` seconds |
| Period | `5` seconds |

#### Service Configuration

| Setting | Value | Description |
|---------|-------|-------------|
| `metadata.name` | `auth-srv` | Service name (used by other services) |
| `spec.selector.app` | `auth` | Routes to auth pods |
| Port | `3000` → `3000` | Service port to container port |
| Type | `ClusterIP` (default) | Internal cluster access only |

### `infra/k8s/auth-mongo-depl.yaml`

**Purpose:** Kubernetes Deployment and Service for the auth service's MongoDB instance.

#### Deployment Configuration

| Setting | Value |
|---------|-------|
| `metadata.name` | `auth-mongo-depl` |
| `spec.replicas` | `1` |
| Container image | `mongo` (latest) |

#### Service Configuration

| Setting | Value |
|---------|-------|
| `metadata.name` | `auth-mongo-srv` |
| Port | `27017` → `27017` |

**Note:** No persistent volume is configured — data is ephemeral and lost on pod restart.

### `infra/k8s/ingress-srv.yaml`

**Purpose:** Nginx Ingress configuration routing external traffic to services.

| Setting | Value |
|---------|-------|
| Ingress class | `nginx` |
| Regex enabled | `true` |
| Host | `ticketing.dev` |

#### Auth Service Routing

| Path Pattern | Backend Service | Port |
|--------------|-----------------|------|
| `/api/users/?(.*)` | `auth-srv` | `3000` |

### `skaffold.yaml`

**Purpose:** Skaffold configuration for local Kubernetes development.

| Setting | Value | Description |
|---------|-------|-------------|
| API version | `skaffold/v2alpha3` | Skaffold schema version |
| Deploy manifests | `infra/k8s/*` | All K8s manifests in infra/k8s |
| Local push | `false` | Don't push to remote registry |

#### Auth Service Artifact

| Setting | Value |
|---------|-------|
| Image name | `berkinanik/ticketing-auth` |
| Context | `services/auth` |
| Dockerfile | `Dockerfile` |
| Sync pattern | `src/**/*.ts` → `.` (hot reload) |

### `.github/workflows/tests.yml`

**Purpose:** GitHub Actions CI workflow for running tests on pull requests.

| Setting | Value |
|---------|-------|
| Trigger | Pull requests (excluding `infra/**`) |
| Runner | `ubuntu-latest` |
| Node.js version | `20` |
| pnpm version | `9` |
| Concurrency group | `dev-ci-${{ github.event.pull_request.number || github.ref }}` |

#### Steps

1. Checkout code
2. Install pnpm@9
3. Install Node.js 20 (with pnpm cache)
4. Load MongoDB binary cache
5. Install dependencies (`pnpm install --frozen-lockfile`)
6. Run typecheck (`pnpm typecheck`)
7. Run tests (`pnpm test:ci`)

### `.nvmrc`

| Value | Description |
|-------|-------------|
| `20` | Specifies Node.js version 20 for nvm |

---

## Part E: Feature Flags

### Environment-Based Toggles

| Flag/Condition | Location | What It Controls | Default Behavior | Override Mechanism |
|----------------|----------|------------------|------------------|-------------------|
| `process.env.NODE_ENV !== 'test'` | `services/auth/src/app.ts:30` | Cookie `secure` attribute — when `true`, cookies only sent over HTTPS | `secure: true` in production | Set `NODE_ENV=test` to disable HTTPS-only cookies |

### Cookie Session Configuration

From `services/auth/src/app.ts:27-32`:
```javascript
cookieSession({
  signed: false,
  secure: process.env.NODE_ENV !== 'test',
})
```

| Setting | Value | Description |
|---------|-------|-------------|
| `signed` | `false` (hardcoded) | Cookies are not signed — JWT provides integrity |
| `secure` | Dynamic (`true` unless `NODE_ENV === 'test'`) | Requires HTTPS in production |

### No External Feature Flag Systems

A search for feature flag libraries and patterns yielded no results:
- No LaunchDarkly, Unleash, ConfigCat, or similar integrations
- No `ENABLE_*`, `DISABLE_*`, or `FLAG_*` environment variables
- No `isFeatureEnabled()` or similar function calls

The auth service does not implement any feature flagging beyond the `NODE_ENV`-based cookie security toggle.

---

## Summary

### Required Environment Variables for Deployment

| Variable | Source | Example Value |
|----------|--------|---------------|
| `JWT_KEY` | Kubernetes Secret `jwt-secret` | (user-provided secret) |
| `MONGO_URI` | Hardcoded in `auth-depl.yaml` | `mongodb://auth-mongo-srv:27017/auth` |

### Sensitive Values

| Item | Location | Protection |
|------|----------|------------|
| `JWT_KEY` | K8s Secret → env var | Kubernetes RBAC; not logged |
| `MONGO_URI` | K8s manifest (hardcoded) | Contains no credentials (local cluster) |
| User passwords | MongoDB | Hashed with scrypt (64-byte output + random salt) |

### Hardcoded Configuration Values

| Value | Location | Description |
|-------|----------|-------------|
| Port `3000` | `services/auth/src/index.ts:22` | HTTP server listen port |
| `/api/users/*` routes | Various route files | URL path prefix |
| `cookie.signed = false` | `services/auth/src/app.ts:29` | Cookie signing disabled |
| `trust proxy = true` | `services/auth/src/app.ts:23` | Express trusts X-Forwarded-* headers |
| Password hash length `64` | `services/auth/src/services/password.ts:9` | scrypt output length |
| Salt length `8` bytes | `services/auth/src/services/password.ts:8` | Random salt for password hashing |
