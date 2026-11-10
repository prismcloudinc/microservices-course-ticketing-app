---
service_id: ticketing-app/@repo/jest-config
service_name: "@repo/jest-config"
prompt_id: 05-configuration-infrastructure
prompt_label: Configuration & Infrastructure
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.10883
schema_version: "2.0"
---

# Configuration Analysis: @repo/jest-config

## Overview

`@repo/jest-config` is a minimal shared configuration package containing only two files:
- `packages/jest-config/package.json` (7 lines)
- `packages/jest-config/jest.config.js` (7 lines)

The package exports a static Jest configuration object with no runtime environment variable access, secrets management, or feature flags.

---

## Part A: Environment Variables

### Direct Environment Variable References

**None** — The `@repo/jest-config` package contains no `process.env` references, `import.meta.env` calls, or any other environment variable access patterns.

| Variable Name | Used In (file:line) | Purpose | Required | Default Value | Sensitive |
|---------------|---------------------|---------|----------|---------------|-----------|
| *(none)* | — | — | — | — | — |

### Indirect Environment Variables (set by consumers)

The services that consume this configuration set environment variables in their test setup files (`setupFilesAfterEnv`). These are not part of `@repo/jest-config` itself but are relevant context:

| Variable Name | Set In (file:line) | Purpose | Required | Default Value | Sensitive |
|---------------|-------------------|---------|----------|---------------|-----------|
| `JWT_KEY` | `services/auth/src/test/setup.ts:7` | JWT signing key for test authentication | Yes (for tests) | `'jwt-key'` (hardcoded test value) | Yes (credential) — but uses non-sensitive test stub |
| `JWT_KEY` | `services/tickets/src/test/setup.ts:9` | JWT signing key for test authentication | Yes (for tests) | `'jwt-key'` (hardcoded test value) | Yes (credential) — but uses non-sensitive test stub |
| `JWT_KEY` | `services/orders/src/test/setup.ts:9` | JWT signing key for test authentication | Yes (for tests) | `'jwt-key'` (hardcoded test value) | Yes (credential) — but uses non-sensitive test stub |
| `JWT_KEY` | `services/payments/src/test/setup.ts:10` | JWT signing key for test authentication | Yes (for tests) | `'jwt-key'` (hardcoded test value) | Yes (credential) — but uses non-sensitive test stub |

---

## Part B: Secrets Management

**None** — The `@repo/jest-config` package contains no secrets management integration:

- No AWS Secrets Manager (`@aws-sdk/client-secrets-manager`) references
- No HashiCorp Vault client usage
- No AWS SSM Parameter Store calls
- No Azure Key Vault access
- No Google Secret Manager usage

The package is a static configuration exporter with no runtime secret retrieval.

---

## Part C: Config Files

### `packages/jest-config/package.json`

**Purpose**: Package manifest defining the npm package identity and exports.

| Key | Value | Description |
|-----|-------|-------------|
| `name` | `@repo/jest-config` | Scoped package name for internal monorepo consumption |
| `version` | `1.0.0` | Semantic version |
| `exports."."` | `./jest.config.js` | Maps the package root import to the Jest configuration file |

**Environment-specific overrides**: None — identical across all environments.

### `packages/jest-config/jest.config.js`

**Purpose**: Base Jest configuration exported to all consuming services.

| Key | Value | Description |
|-----|-------|-------------|
| `preset` | `'ts-jest'` | Uses ts-jest for TypeScript transpilation during tests |
| `testEnvironment` | `'node'` | Tests execute in Node.js environment (not jsdom) |
| `verbose` | `true` | Displays individual test results with suite hierarchy |
| `modulePaths` | `['<rootDir>/src']` | Enables absolute imports from the `src` directory in consuming services |
| `watchPathIgnorePatterns` | `['globalConfig']` | Prevents watch-mode re-runs when mongodb-memory-server writes `globalConfig` files |

**Environment-specific overrides**: None — this is a static base configuration.

---

## Part D: Infrastructure Configuration

### Build System Configuration

#### `turbo.json` (root)

**Purpose**: Turborepo pipeline configuration for monorepo task orchestration.

| Pipeline Task | Configuration | Relevance to @repo/jest-config |
|--------------|---------------|-------------------------------|
| `test` | `dependsOn: ["typecheck"]` | Services run `typecheck` before tests; tests use the shared Jest config |
| `test:ci` | `{}` (no dependencies) | CI test execution, uses shared Jest config |
| `test:watch` | `dependsOn: ["typecheck"], cache: false` | Watch mode testing, caching disabled |

**Memory/timeout/handler settings**: Not specified (uses Turborepo defaults).

**Environment variable injection**: None in pipeline — tasks inherit from shell environment.

### CI/CD Configuration

#### `.github/workflows/tests.yml`

**Purpose**: GitHub Actions workflow for automated test execution on pull requests.

| Setting | Value | Description |
|---------|-------|-------------|
| Trigger | `pull_request` (excluding `infra/**` paths), `workflow_dispatch` | Runs on PRs and manual dispatch |
| Runner | `ubuntu-latest` | GitHub-hosted Ubuntu runner |
| Node version | `20` | Matches `engines.node: ">=20"` in root `package.json` |
| pnpm version | `9` | Matches `engines.pnpm: ">=9"` in root `package.json` |
| Concurrency group | `dev-ci-${{ github.event.pull_request.number || github.ref }}` | Cancels in-progress runs for same PR |
| MongoDB binary cache | `~/.cache/mongodb-binaries` | Caches mongodb-memory-server binaries for faster test setup |

**Test execution command**: `pnpm test:ci` — invokes Turborepo which runs Jest with `@repo/jest-config` base configuration in each service.

**Environment variable injection in CI**: None explicitly set — tests run with default GitHub Actions environment plus the hardcoded `JWT_KEY='jwt-key'` set in each service's setup file.

### Package Manager Configuration

#### `pnpm-workspace.yaml`

**Purpose**: Defines monorepo workspace packages.

| Pattern | Description |
|---------|-------------|
| `'services/*'` | All service directories |
| `'packages/*'` | **Includes `packages/jest-config`** |
| `'apps/*'` | Frontend applications |
| `'nats-test'` | NATS testing utilities |

---

## Part E: Feature Flags

**None** — The `@repo/jest-config` package contains no feature flag systems:

- No LaunchDarkly, Unleash, Split.io, or similar feature flag SDK references
- No `FF_*` or `FEATURE_*` prefixed environment variables
- No conditional configuration based on environment or feature toggles
- No remote config service integrations

The configuration is static and applies uniformly to all consumers.

---

## Summary

The `@repo/jest-config` package is a **pure static configuration package** with:

- **0 environment variables** accessed directly
- **0 secrets** retrieved or managed
- **2 config files** (both static, no environment overrides)
- **0 feature flags**

The package's sole purpose is to centralize Jest configuration settings (`ts-jest` preset, Node.js test environment, verbose output, module path resolution, and watch ignore patterns) for consistent test execution across the monorepo's backend services.

Consumers extend this base configuration via local `jest.config.js` files that add `setupFilesAfterEnv` pointing to service-specific test setup modules.
