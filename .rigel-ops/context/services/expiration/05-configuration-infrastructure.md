---
service_id: ticketing-app/expiration
service_name: expiration
prompt_id: 05-configuration-infrastructure
prompt_label: Configuration & Infrastructure
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.153285
schema_version: "2.0"
---

# Configuration & Environment Analysis: expiration

## Part A: Environment Variables

| Variable Name | Used In (file:line) | Purpose | Required | Default Value | Sensitive |
|---|---|---|---|---|---|
| `NATS_URL` | `src/index.ts:5,21` | URL for connecting to NATS Streaming Server (e.g., `http://nats-srv:4222`) | Yes (validated at startup, throws if missing) | None | No |
| `NATS_CLUSTER_ID` | `src/index.ts:9,13,19` | Identifier for the NATS cluster to connect to | Yes (validated at startup, throws if missing) | None | No |
| `NATS_CLIENT_ID` | `src/index.ts:20` | Unique client identifier for this service instance within the NATS cluster | Yes (used with non-null assertion, not explicitly validated) | None | No |
| `REDIS_HOST` | `src/queues/expiration-queue.ts:11` | Hostname of the Redis server used by the Bull queue for job storage | No explicit validation (used directly, may fail at queue creation if undefined) | None | No |

### Validation Details

The startup validation in `src/index.ts` (lines 5-15) checks only `NATS_URL` and `NATS_CLUSTER_ID`. Note: There is a duplicate check for `NATS_CLUSTER_ID` at lines 9 and 13 — the second check should likely validate `NATS_CLIENT_ID` instead (potential bug).

`REDIS_HOST` is not validated at startup. If undefined, the Bull queue will attempt to connect to `undefined` as the Redis host, which will fail at runtime when the queue processor is invoked.

---

## Part B: Secrets Management

**No secrets management systems are used by this service.**

The service does not integrate with any secrets managers, vault clients, AWS Parameter Store, Azure Key Vault, or similar systems. All configuration is provided via environment variables injected at deployment time through Kubernetes manifests.

There are no sensitive secrets (API keys, passwords, certificates, tokens) required by this service:
- NATS Streaming connection does not use authentication in the current configuration
- Redis connection does not use authentication in the current configuration

---

## Part C: Config Files

### `package.json`

**Purpose:** Package manifest defining dependencies, scripts, and metadata.

| Key | Value | Description |
|-----|-------|-------------|
| `name` | `expiration` | Package name for the service |
| `version` | `1.0.0` | Current service version |
| `main` | `index.js` | Main entry point (not used; service runs via ts-node) |
| `scripts.dev` | `nodemon src/index.ts` | Development script with hot reload |
| `scripts.start` | `ts-node src/index.ts` | Production start script (runs TypeScript directly) |
| `scripts.test` | `jest` | Run tests |
| `scripts.test:ci` | `jest --ci` | Run tests in CI mode |
| `scripts.test:watch` | `jest --watchAll` | Run tests in watch mode |
| `scripts.typecheck` | `tsc --noEmit` | TypeScript type checking without emitting files |
| `scripts.lockfile` | `make-dedicated-lockfile` | Generate dedicated lockfile for Docker builds |

**No environment-specific overrides** — this file is the same across all environments.

### `tsconfig.json`

**Purpose:** TypeScript compiler configuration.

| Key | Value | Description |
|-----|-------|-------------|
| `compilerOptions.target` | `es2016` | ECMAScript target version |
| `compilerOptions.module` | `commonjs` | Module system (CommonJS for Node.js) |
| `compilerOptions.rootDir` | `./src` | Root directory for source files |
| `compilerOptions.noEmit` | `true` | Don't emit compiled JavaScript (ts-node handles execution) |
| `compilerOptions.esModuleInterop` | `true` | Enable ES module interoperability |
| `compilerOptions.forceConsistentCasingInFileNames` | `true` | Enforce consistent file name casing |
| `compilerOptions.strict` | `true` | Enable all strict type checking options |
| `compilerOptions.skipLibCheck` | `true` | Skip type checking of declaration files |
| `compilerOptions.types` | `["jest"]` | Include Jest type definitions |

**No environment-specific overrides** — TypeScript config is identical across all environments.

### `jest.config.js`

**Purpose:** Jest test framework configuration.

| Key | Value | Description |
|-----|-------|-------------|
| `...baseConfig` | (spread from `@repo/jest-config`) | Inherits shared monorepo Jest configuration |
| `setupFilesAfterEnv` | `['./src/test/setup.ts']` | Test setup file that mocks NATS and queue dependencies |

**No environment-specific overrides** — test configuration is identical for local and CI environments.

---

## Part D: Infrastructure Configuration

### `Dockerfile`

**Purpose:** Builds the containerized expiration service for deployment.

| Setting | Value | Description |
|---------|-------|-------------|
| Base Image | `node:20-alpine` | Node.js 20 on Alpine Linux (minimal footprint) |
| Working Directory | `/usr/app` | Container working directory |
| Package Manager | `pnpm@9` | Installed globally via npm |
| Default Command | `pnpm run dev` | Runs development mode with nodemon (hot reload) |

**Environment variable injection:** None in Dockerfile — all environment variables are injected at runtime via Kubernetes.

**Build steps:**
1. Copy `package.json` and `pnpm-lock.yaml`
2. Run `pnpm i` to install dependencies
3. Copy remaining source files
4. Start with `pnpm run dev`

### `infra/k8s/expiration-depl.yaml`

**Purpose:** Kubernetes Deployment manifest for the expiration service.

| Setting | Value | Description |
|---------|-------|-------------|
| Deployment Name | `expiration-depl` | Kubernetes deployment identifier |
| Replicas | `1` | Single instance (job queue ensures no duplicate processing) |
| Container Image | `berkinanik/ticketing-expiration:latest` | Docker Hub image reference |
| Labels | `app: expiration` | Pod selector label |

**Environment Variables Injected:**

| Variable | Source | Value |
|----------|--------|-------|
| `NATS_URL` | Static value | `http://nats-srv:4222` |
| `NATS_CLUSTER_ID` | Static value | `ticketing` |
| `NATS_CLIENT_ID` | Kubernetes field reference | `metadata.name` (pod name, ensures uniqueness) |
| `REDIS_HOST` | Static value | `expiration-redis-srv` |

### `infra/k8s/expiration-redis-depl.yaml`

**Purpose:** Kubernetes Deployment and Service for the dedicated Redis instance.

| Setting | Value | Description |
|---------|-------|-------------|
| Deployment Name | `expiration-redis-depl` | Redis deployment identifier |
| Replicas | `1` | Single Redis instance |
| Container Image | `redis` | Official Redis image (latest tag) |
| Service Name | `expiration-redis-srv` | Internal DNS name for Redis access |
| Service Port | `6379` | Standard Redis port |
| Target Port | `6379` | Container port mapping |
| Protocol | `TCP` | Transport protocol |

**No persistence configured** — Redis data is ephemeral; job queue state is lost on pod restart (acceptable for this use case as jobs represent future events that may already be stale).

### `skaffold.yaml` (root)

**Purpose:** Local Kubernetes development workflow orchestration.

| Setting | Value | Description |
|---------|-------|-------------|
| API Version | `skaffold/v2alpha3` | Skaffold configuration version |
| Deploy Strategy | `kubectl` | Uses kubectl to apply manifests |
| Manifests Path | `infra/k8s/*` | Applies all K8s manifests |
| Image Name | `berkinanik/ticketing-expiration` | Local build image tag |
| Build Context | `services/expiration` | Docker build context directory |
| File Sync | `src/**/*.ts` → `.` | Hot-reload TypeScript files without rebuild |
| Push | `false` | Don't push images (local development only) |

### `turbo.json` (root)

**Purpose:** Turborepo monorepo build pipeline configuration.

| Pipeline Task | Dependencies | Caching | Description |
|---------------|--------------|---------|-------------|
| `lockfile` | None | Yes (default) | Generate dedicated lockfiles |
| `test` | `typecheck` | Yes (default) | Run tests after type checking |
| `test:ci` | None | Yes (default) | CI test execution |
| `test:watch` | `typecheck` | No (`cache: false`) | Watch mode testing |
| `typecheck` | None | Yes (default) | TypeScript type checking |

### `.github/workflows/tests.yml`

**Purpose:** GitHub Actions CI workflow for automated testing.

| Setting | Value | Description |
|---------|-------|-------------|
| Trigger | `pull_request`, `workflow_dispatch` | Runs on PRs and manual dispatch |
| Path Ignore | `infra/**` | Skips CI for infrastructure-only changes |
| Concurrency Group | `dev-ci-${{ github.event.pull_request.number || github.ref }}` | Cancels in-progress runs for same PR |
| Runner | `ubuntu-latest` | GitHub-hosted Ubuntu runner |
| Node Version | `20` | Matches service runtime |
| Package Manager | `pnpm@9` | Matches service package manager |

**No environment variables or secrets injected** — tests use mocked dependencies (`src/test/setup.ts` mocks `nats-wrapper` and `queues`).

### `.dockerignore`

**Purpose:** Excludes files from Docker build context.

| Pattern | Description |
|---------|-------------|
| `node_modules` | Installed dependencies (reinstalled in container) |
| `dist` | Compiled output (not used with noEmit) |
| `build` | Build artifacts |
| `.turbo` | Turborepo cache directory |

---

## Part E: Feature Flags

**No feature flags are used by this service.**

The codebase was searched for:
- Feature flag libraries (LaunchDarkly, Split.io, Unleash, Flipper)
- Feature toggle patterns
- Environment-based conditional feature enabling

No matches were found. The service behavior is consistent across all environments with no feature gating.

---

## Summary

The expiration service has a minimal configuration footprint:

| Category | Count | Notes |
|----------|-------|-------|
| Environment Variables | 4 | All infrastructure connection settings |
| Secrets | 0 | No authentication configured for NATS/Redis |
| Config Files | 3 | package.json, tsconfig.json, jest.config.js |
| Infrastructure Files | 5 | Dockerfile, 2 K8s manifests, skaffold.yaml, turbo.json |
| Feature Flags | 0 | None implemented |

**Potential Issue Identified:** `NATS_CLIENT_ID` is used with a non-null assertion (`!`) but is not validated at startup. Additionally, there is a duplicate validation check for `NATS_CLUSTER_ID` at lines 9 and 13 in `src/index.ts` — the second check appears to be a copy-paste error and should likely validate `NATS_CLIENT_ID` instead.
