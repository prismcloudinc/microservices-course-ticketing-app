---
service_id: ticketing-app/@repo/jest-config
service_name: "@repo/jest-config"
prompt_id: 01-service-identity
prompt_label: Service Identity & Anatomy
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.10192000000000001
schema_version: "2.0"
---

# Service Anatomy: @repo/jest-config

## Section 1: Identity

| Attribute | Value |
|-----------|-------|
| **Package Name** | `@repo/jest-config` |
| **Version** | `1.0.0` |
| **Purpose** | Shared Jest configuration package providing standardized test settings for all backend services in the ticketing monorepo |
| **Product/Platform** | Ticketing application microservices platform (inferred from root `package.json` name: `ticketing-app`) |
| **Runtime** | Node.js >=20 (from root `package.json` engines field, line 29) |
| **Framework** | None — this is a configuration-only package |
| **Database** | None — this package does not interact with any database |

### Package Manifest (`packages/jest-config/package.json`)
```json
{
  "name": "@repo/jest-config",
  "version": "1.0.0",
  "exports": {
    ".": "./jest.config.js"
  }
}
```

## Section 2: Entry Points

This package has **no runtime entry points**. It is a configuration-only shared library that exports a Jest configuration object.

| File Path | Trigger | Description |
|-----------|---------|-------------|
| `packages/jest-config/jest.config.js` | Imported at test time | Exports the base Jest configuration object consumed by service-level `jest.config.js` files |

## Section 3: Directory Map

### `packages/jest-config/`
**Purpose**: Contains the shared Jest configuration for the monorepo's backend services.

| File | Description |
|------|-------------|
| `package.json` | Package manifest defining the package name (`@repo/jest-config`), version (`1.0.0`), and exports mapping the root import to `jest.config.js` |
| `jest.config.js` | CommonJS module exporting a Jest configuration object with settings for TypeScript testing via `ts-jest` |

### Configuration Details (`packages/jest-config/jest.config.js`)

```javascript
module.exports = {
  preset: 'ts-jest',           // Uses ts-jest for TypeScript transpilation
  testEnvironment: 'node',     // Tests run in Node.js environment (not jsdom)
  verbose: true,               // Displays individual test results with test suite hierarchy
  modulePaths: ['<rootDir>/src'], // Allows absolute imports from the src directory
  watchPathIgnorePatterns: ['globalConfig'], // Ignores globalConfig files in watch mode
};
```

## Section 4: Dependency Analysis

### Direct Dependencies

The `packages/jest-config/package.json` declares **no dependencies** (neither `dependencies` nor `devDependencies`).

### Implicit Dependencies (from root `package.json`)

The package implicitly depends on the following packages installed at the monorepo root level:

| Package | Version | Purpose in this codebase |
|---------|---------|--------------------------|
| `ts-jest` | `^29.1.2` | Required by the `preset: 'ts-jest'` configuration. Provides TypeScript support for Jest by transpiling `.ts` files during test execution. Declared in root `package.json` line 24. |
| `jest` | `^29.7.0` | The test framework itself. Consumers of this config run Jest which interprets the exported configuration. Declared in root `package.json` line 22. |
| `@types/jest` | `^29.5.12` | TypeScript type definitions for Jest. Provides types for `describe`, `it`, `expect`, etc. Declared in root `package.json` line 21. |

## Section 5: Internal Package Usage

### `@repo/jest-config` (this package)

This package is consumed by all five backend services in the monorepo.

#### Importing Services

| Service | Import Location | Usage |
|---------|-----------------|-------|
| `services/auth` | `services/auth/jest.config.js` line 1 | Imports as `baseConfig`, spreads into local config, adds `setupFilesAfterEnv` |
| `services/orders` | `services/orders/jest.config.js` line 1 | Imports as `baseConfig`, spreads into local config, adds `setupFilesAfterEnv` |
| `services/tickets` | `services/tickets/jest.config.js` line 1 | Imports as `baseConfig`, spreads into local config, adds `setupFilesAfterEnv` |
| `services/expiration` | `services/expiration/jest.config.js` line 1 | Imports as `baseConfig`, spreads into local config, adds `setupFilesAfterEnv` |
| `services/payments` | `services/payments/jest.config.js` line 1 | Imports as `baseConfig`, spreads into local config, adds `setupFilesAfterEnv` |

#### Dependency Declaration

Each service declares the dependency using pnpm workspace linking:

```json
"devDependencies": {
  "@repo/jest-config": "link:*"
}
```

This syntax is used in:
- `services/auth/package.json` line 29
- `services/orders/package.json` line 31
- `services/tickets/package.json` line 31
- `services/expiration/package.json` line 24
- `services/payments/package.json` line 32

#### Exports Used

Each consuming service uses the **default export** (the entire configuration object):

```javascript
const baseConfig = require('@repo/jest-config');

module.exports = {
  ...baseConfig,
  setupFilesAfterEnv: ['./src/test/setup.ts'],
};
```

The package exports via the `exports` field in `package.json`:
```json
"exports": {
  ".": "./jest.config.js"
}
```

#### Functionality Provided

1. **TypeScript transpilation** — The `preset: 'ts-jest'` setting enables services to write tests in TypeScript without manual compilation
2. **Node.js test environment** — The `testEnvironment: 'node'` ensures tests run in a Node.js context appropriate for backend services
3. **Verbose output** — The `verbose: true` setting provides detailed test results during development
4. **Absolute imports** — The `modulePaths: ['<rootDir>/src']` allows test files to import modules using absolute paths from the `src` directory
5. **Watch mode optimization** — The `watchPathIgnorePatterns: ['globalConfig']` prevents Jest from rerunning tests when `globalConfig` files change (these are typically generated by mongodb-memory-server)

### Monorepo Integration

The package is registered in the pnpm workspace via `pnpm-workspace.yaml`:

```yaml
packages:
  - 'services/*'
  - 'packages/*'
  - 'apps/*'
  - 'nats-test'
```

The `packages/*` glob pattern includes `packages/jest-config`, making it available to other workspace packages via the `link:*` protocol.

### Test Pipeline (from `turbo.json`)

```json
"pipeline": {
  "test": {
    "dependsOn": ["typecheck"]
  },
  "test:ci": {},
  "test:watch": {
    "dependsOn": ["typecheck"],
    "cache": false
  }
}
```

The test commands across services use this shared configuration when executing:
- `pnpm test` — runs Jest with the shared config
- `pnpm test:ci` — runs Jest in CI mode
- `pnpm test:watch` — runs Jest in watch mode
