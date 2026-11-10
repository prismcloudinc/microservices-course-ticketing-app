---
service_id: ticketing-app/nats-test
service_name: nats-test
prompt_id: 05-configuration-infrastructure
prompt_label: Configuration & Infrastructure
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.12498499999999998
schema_version: "2.0"
---

# Configuration & Environment Analysis: nats-test

## Part A: Environment Variables

| Variable Name | Used In (file:line) | Purpose | Required | Default Value | Sensitive |
|---|---|---|---|---|---|
| *(none found)* | — | — | — | — | — |

**Analysis**: The `nats-test` package does **not use any environment variables**. All configuration values are hardcoded directly in the source files.

### Hardcoded Configuration Values (In Lieu of Environment Variables)

| Value | Used In (file:line) | Purpose | Notes |
|---|---|---|---|
| `'ticketing'` | `nats-test/src/publisher.ts:4` | NATS Streaming cluster ID | Must match the `-cid` argument in NATS server deployment |
| `'abc'` | `nats-test/src/publisher.ts:4` | Publisher client ID | Static client ID for the publisher instance |
| `'http://localhost:4222'` | `nats-test/src/publisher.ts:5` | NATS server URL | Assumes port-forward from Kubernetes pod |
| `'ticketing'` | `nats-test/src/listener.ts:9` | NATS Streaming cluster ID | Same as publisher |
| `'http://localhost:4222'` | `nats-test/src/listener.ts:10` | NATS server URL | Same as publisher |
| `5 * 1000` (5000ms) | `nats-test/src/events/base-listener.ts:17` | Acknowledgment wait timeout | Constructor parameter default |
| `'payments-service'` | `nats-test/src/events/ticket-created-listener.ts:8` | Queue group name | Used for subscription grouping |
| `'payments-service'` | `nats-test/src/events/ticket-updated-listener.ts:8` | Queue group name | Same for ticket-updated events |

---

## Part B: Secrets Management

**No secrets management systems are used in this package.**

The package does not:
- Import or call any secrets manager clients (AWS Secrets Manager, HashiCorp Vault, Azure Key Vault, etc.)
- Reference Parameter Store, SSM, or similar services
- Use any credential retrieval mechanisms

The NATS Streaming connection does not require authentication in this configuration — the Kubernetes NATS deployment (`infra/k8s/nats-depl.yaml:17-33`) runs without authentication enabled.

---

## Part C: Config Files

### `nats-test/package.json`

| Key | Value | Description |
|---|---|---|
| `name` | `"nats-test"` | Package name |
| `version` | `"1.0.0"` | Package version |
| `main` | `"index.js"` | Entry point (not actually used; scripts invoke specific files) |
| `scripts.publish` | `"nodemon --watch src/publisher.ts src/publisher.ts"` | Runs publisher with file watching |
| `scripts.listen` | `"nodemon --watch src/listener.ts src/listener.ts"` | Runs listener with file watching |
| `dependencies.node-nats-streaming` | `"^0.3.2"` | NATS Streaming client library |
| `devDependencies.@types/node` | `"^20.12.7"` | Node.js type definitions |
| `devDependencies.node-ts` | `"^6.0.1"` | TypeScript runtime execution (note: likely meant `ts-node`) |
| `devDependencies.nodemon` | `"^3.1.0"` | File watcher for development |
| `devDependencies.typescript` | `"^5.4.5"` | TypeScript compiler |

### `nats-test/tsconfig.json`

| Key | Value | Description |
|---|---|---|
| `compilerOptions.target` | `"es2016"` | JavaScript output target |
| `compilerOptions.module` | `"commonjs"` | Module system |
| `compilerOptions.rootDir` | `"./src"` | Source directory |
| `compilerOptions.noEmit` | `true` | No compilation output (relies on ts-node for runtime) |
| `compilerOptions.esModuleInterop` | `true` | Enable ES module interop |
| `compilerOptions.forceConsistentCasingInFileNames` | `true` | Case-sensitive imports |
| `compilerOptions.strict` | `true` | Strict type checking |
| `compilerOptions.skipLibCheck` | `true` | Skip type checking of declaration files |

### `.nvmrc` (Repository Root)

| Value | Description |
|---|---|
| `20` | Node.js version 20 |

### Environment-Specific Overrides

**None found.** The package has no environment-specific configuration files (no `config/development.json`, `config/production.json`, etc.). All values are hardcoded for local development use only.

---

## Part D: Infrastructure Configuration

### Package-Level Infrastructure

The `nats-test` package has **no infrastructure files**:
- No Dockerfile
- No docker-compose.yml
- No Kubernetes manifests
- No Terraform, CloudFormation, or Pulumi configs
- No CI/CD pipeline files

This is a **local development/demo tool**, not a deployed service.

### Related Infrastructure: NATS Server Deployment

The package connects to a NATS Streaming server deployed via Kubernetes. The relevant manifest is:

#### `infra/k8s/nats-depl.yaml`

**Deployment Resource**:

| Setting | Value | Description |
|---|---|---|
| `metadata.name` | `nats-depl` | Deployment name |
| `spec.replicas` | `1` | Single replica |
| `spec.template.spec.containers[0].image` | `nats-streaming:0.17.0` | NATS Streaming image version |

**Container Arguments** (lines 19-33):

| Argument | Value | Description |
|---|---|---|
| `-p` | `4222` | Client connection port |
| `-m` | `8222` | Monitoring/HTTP port |
| `-hbi` | `5s` | Heartbeat interval |
| `-hbt` | `5s` | Heartbeat timeout |
| `-hbf` | `2` | Heartbeat failure count before disconnect |
| `-SD` | (flag) | Enable STAN debug logging |
| `-cid` | `ticketing` | Cluster ID (must match client connection) |

**Service Resource** (lines 34-51):

| Port Name | Protocol | Port | Target Port | Purpose |
|---|---|---|---|---|
| `client` | TCP | 4222 | 4222 | NATS client connections |
| `monitoring` | TCP | 8222 | 8222 | NATS monitoring dashboard |

**Environment Variable Injection**: None — the NATS deployment does not inject environment variables into the container.

### Workspace Configuration

The package is included in the pnpm workspace (`pnpm-workspace.yaml:5`):

```yaml
packages:
  - 'services/*'
  - 'packages/*'
  - 'apps/*'
  - 'nats-test'
```

---

## Part E: Feature Flags

**No feature flags are used in this package.**

The package does not:
- Import or reference any feature flag libraries (LaunchDarkly, Split, ConfigCat, etc.)
- Use environment-based conditional logic for feature enablement
- Contain any `if (featureEnabled)` or similar toggle patterns
- Reference any remote config services

---

## Summary

The `nats-test` package is a minimal demonstration/testing tool for NATS Streaming patterns. It is designed exclusively for local development:

1. **Zero environment variable usage** — all values are hardcoded
2. **Zero secrets management** — NATS connection is unauthenticated
3. **Two config files** — `package.json` (dependencies/scripts) and `tsconfig.json` (TypeScript settings)
4. **No deployment infrastructure** — requires manual `kubectl port-forward` to connect to the Kubernetes-deployed NATS server
5. **No feature flags** — straightforward demo code with no conditional features

The hardcoded `localhost:4222` URL and fixed cluster ID `ticketing` indicate this package is intended for use with `kubectl port-forward <nats-pod-name> 4222:4222` as documented in `nats-test/README.md:9`.
