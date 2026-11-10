---
service_id: ticketing-app/client
service_name: client
prompt_id: 05-configuration-infrastructure
prompt_label: Configuration & Infrastructure
generated_at: 2026-11-10T20:25:08.800Z
cost_usd: 0.23378000000000002
schema_version: "2.0"
---

# Configuration Reference: client

## Part A: Environment Variables

| Variable Name | Used In (file:line) | Purpose | Required | Default Value | Sensitive |
|---|---|---|---|---|---|
| `STRIPE_CLIENT_KEY` | `apps/client/src/app/orders/[orderId]/page.tsx:22` | Stripe publishable key for client-side payment processing via `react-stripe-checkout` component | No (code defaults to empty string) | `''` (empty string) | Yes — this is a Stripe publishable key used for client-side payment integration |

### Notes on Environment Variables

- The `STRIPE_CLIENT_KEY` is the **only** environment variable accessed via `process.env` in the client service.
- No `import.meta.env` pattern (Vite-style) is used.
- No `NEXT_PUBLIC_*` prefixed variables are used, even though Next.js supports them for client-side exposure.
- The key is read server-side in a React Server Component and passed as a prop to the `CheckoutForm` client component (`apps/client/src/app/orders/[orderId]/page.tsx:41`).

---

## Part B: Secrets Management

### Secret Retrieval Mechanism

The client service does **not** directly call any secrets management services (e.g., AWS Secrets Manager, HashiCorp Vault, Azure Key Vault, GCP Secret Manager, AWS SSM Parameter Store).

Instead, secrets are injected via **Kubernetes Secrets** at deployment time.

| Secret Name | Kubernetes Secret Object | Key | What It Contains | Where Injected | When Fetched | What Breaks If Missing |
|---|---|---|---|---|---|---|
| `STRIPE_CLIENT_KEY` | `stripe-client-secret` | `STRIPE_CLIENT_KEY` | Stripe publishable/public API key for client-side Stripe Checkout | `infra/k8s/client-depl.yaml:18-23` | Pod startup (injected as environment variable) | The `CheckoutForm` component renders `null` instead of the Stripe checkout button, preventing users from completing payments (`apps/client/src/app/orders/[orderId]/CheckoutForm.tsx:34-41`) |

### Secret Creation Command

From `README.md:129-130`:
```bash
kubectl create secret generic stripe-client-secret --from-literal=STRIPE_CLIENT_KEY=<your_stripe_client_key>
```

---

## Part C: Config Files

### `apps/client/next.config.mjs`

**Purpose:** Next.js framework configuration.

| Key | Value | Description |
|-----|-------|-------------|
| (empty object) | `{}` | No custom Next.js configuration is applied; all defaults are used |

### `apps/client/tsconfig.json`

**Purpose:** TypeScript compiler configuration.

| Key | Value | Description |
|-----|-------|-------------|
| `compilerOptions.lib` | `["dom", "dom.iterable", "esnext"]` | Target libraries for type checking |
| `compilerOptions.allowJs` | `true` | Allow JavaScript files |
| `compilerOptions.skipLibCheck` | `true` | Skip type checking of declaration files |
| `compilerOptions.strict` | `true` | Enable all strict type-checking options |
| `compilerOptions.noEmit` | `true` | Do not emit output files (Next.js handles compilation) |
| `compilerOptions.esModuleInterop` | `true` | Enable interoperability between CommonJS and ES Modules |
| `compilerOptions.module` | `"esnext"` | Use ESNext module syntax |
| `compilerOptions.moduleResolution` | `"bundler"` | Use bundler-style module resolution |
| `compilerOptions.resolveJsonModule` | `true` | Allow importing JSON files |
| `compilerOptions.isolatedModules` | `true` | Ensure each file can be transpiled independently |
| `compilerOptions.jsx` | `"preserve"` | Preserve JSX for Next.js to transform |
| `compilerOptions.incremental` | `true` | Enable incremental compilation |
| `compilerOptions.plugins` | `[{"name": "next"}]` | Use Next.js TypeScript plugin |
| `compilerOptions.paths["~/*"]` | `["./src/*"]` | Path alias for imports |
| `include` | `[".eslintrc.cjs", "next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts", "**/*.cjs", "**/*.mjs"]` | Files to include in compilation |
| `exclude` | `["node_modules"]` | Files to exclude from compilation |

### `apps/client/tailwind.config.ts`

**Purpose:** Tailwind CSS configuration.

| Key | Value | Description |
|-----|-------|-------------|
| `content` | `["./src/pages/**/*.{js,ts,jsx,tsx,mdx}", "./src/components/**/*.{js,ts,jsx,tsx,mdx}", "./src/app/**/*.{js,ts,jsx,tsx,mdx}"]` | Files to scan for Tailwind classes |
| `theme.extend.backgroundImage["gradient-radial"]` | `"radial-gradient(var(--tw-gradient-stops))"` | Custom radial gradient utility |
| `theme.extend.backgroundImage["gradient-conic"]` | `"conic-gradient(from 180deg at 50% 50%, var(--tw-gradient-stops))"` | Custom conic gradient utility |
| `plugins` | `[]` | No additional Tailwind plugins |

### `apps/client/.eslintrc.cjs`

**Purpose:** ESLint linting configuration.

| Key | Value | Description |
|-----|-------|-------------|
| `parser` | `"@typescript-eslint/parser"` | Use TypeScript parser |
| `parserOptions.project` | `true` | Enable project-wide type checking |
| `plugins` | `["@typescript-eslint"]` | TypeScript ESLint plugin |
| `extends` | `["next/core-web-vitals", "plugin:@typescript-eslint/recommended-type-checked", "plugin:@typescript-eslint/stylistic-type-checked"]` | Extended configurations |
| `rules["@typescript-eslint/consistent-type-imports"]` | `["error", {...}]` | Enforce consistent type import syntax |
| `rules["@typescript-eslint/no-explicit-any"]` | `"error"` | Disallow `any` type |
| `rules["@typescript-eslint/no-non-null-assertion"]` | `"error"` | Disallow non-null assertions |
| `rules["import/order"]` | `[...]` | Enforce consistent import ordering |
| `rules["no-console"]` | `"warn"` | Warn on console statements |

### `apps/client/.prettierrc.mjs`

**Purpose:** Prettier code formatting configuration.

| Key | Value | Description |
|-----|-------|-------------|
| `plugins` | `["prettier-plugin-tailwindcss"]` | Tailwind class sorting plugin |
| `arrowParens` | `"always"` | Always include parentheses around arrow function parameters |
| `printWidth` | `100` | Maximum line width |
| `singleQuote` | `true` | Use single quotes |
| `semi` | `true` | Include semicolons |
| `tabWidth` | `2` | Indent width |
| `trailingComma` | `"all"` | Include trailing commas everywhere |
| `useTabs` | `false` | Use spaces instead of tabs |
| `endOfLine` | `"lf"` | Use Unix line endings |

### `apps/client/postcss.config.cjs`

**Purpose:** PostCSS transformation configuration.

| Key | Value | Description |
|-----|-------|-------------|
| `plugins.tailwindcss` | `{}` | Enable Tailwind CSS plugin |
| `plugins.autoprefixer` | `{}` | Enable vendor prefix plugin |

### `apps/client/package.json`

**Purpose:** Package manifest and scripts.

| Key | Value | Description |
|-----|-------|-------------|
| `name` | `"client"` | Package name |
| `version` | `"0.1.0"` | Package version |
| `private` | `true` | Prevent accidental publishing |
| `imports["~/*"]` | `"./src/*"` | Import path alias |
| `scripts.dev` | `"next dev"` | Start development server |
| `scripts.build` | `"next build"` | Build for production |
| `scripts.start` | `"next start"` | Start production server |
| `scripts.lint` | `"eslint ."` | Run ESLint |
| `scripts.typecheck` | `"tsc --noEmit"` | Run TypeScript type checking |

### Environment-Specific Overrides

**No environment-specific configuration files exist** (e.g., no `config/dev.json`, `config/prod.json`, or environment-specific Next.js configuration).

The only differentiation is:
- **Development:** Runs via `pnpm dev` (Next.js dev server with hot reload) per `Dockerfile:12`
- **Production:** Would use `pnpm build && pnpm start` (not configured in Dockerfile — current Dockerfile runs dev mode)

---

## Part D: Infrastructure Configuration

### `apps/client/Dockerfile`

**Purpose:** Container image build definition.

| Setting | Value | Description |
|---------|-------|-------------|
| Base Image | `node:alpine` | Minimal Node.js image |
| Working Directory | `/usr/app` | Application directory inside container |
| Package Manager | `pnpm@9` | Installed globally via npm |
| Startup Command | `["pnpm", "run", "dev"]` | Runs Next.js in development mode |
| Exposed Ports | (implicit) 3000 | Default Next.js port |

**Note:** The Dockerfile runs `pnpm dev` (development mode), not `pnpm build && pnpm start` (production mode). This is typical for local Kubernetes development with Skaffold file sync.

### `infra/k8s/client-depl.yaml`

**Purpose:** Kubernetes Deployment and Service for the client application.

#### Deployment Configuration

| Setting | Value | Description |
|---------|-------|-------------|
| `metadata.name` | `client-depl` | Deployment name |
| `spec.replicas` | `1` | Number of pod replicas |
| `spec.selector.matchLabels.app` | `client` | Pod selector label |
| `spec.template.spec.containers[0].name` | `client` | Container name |
| `spec.template.spec.containers[0].image` | `berkinanik/ticketing-client:latest` | Container image |

#### Environment Variable Injection

| Env Var | Source | Secret Name | Secret Key |
|---------|--------|-------------|------------|
| `STRIPE_CLIENT_KEY` | `secretKeyRef` | `stripe-client-secret` | `STRIPE_CLIENT_KEY` |

#### Liveness Probe Configuration

| Setting | Value | Description |
|---------|-------|-------------|
| `httpGet.path` | `/health/check` | Health check endpoint |
| `httpGet.port` | `3000` | Port to probe |
| `initialDelaySeconds` | `5` | Delay before first probe |
| `periodSeconds` | `5` | Interval between probes |

#### Service Configuration

| Setting | Value | Description |
|---------|-------|-------------|
| `metadata.name` | `client-srv` | Service name |
| `spec.selector.app` | `client` | Pod selector |
| `spec.ports[0].name` | `client` | Port name |
| `spec.ports[0].protocol` | `TCP` | Protocol |
| `spec.ports[0].port` | `3000` | Service port |
| `spec.ports[0].targetPort` | `3000` | Container port |

### `infra/k8s/ingress-srv.yaml`

**Purpose:** Kubernetes Ingress for routing external traffic.

| Setting | Value | Description |
|---------|-------|-------------|
| `metadata.name` | `ingress-service` | Ingress resource name |
| `annotations["kubernetes.io/ingress.class"]` | `nginx` | Use nginx ingress controller |
| `annotations["nginx.ingress.kubernetes.io/use-regex"]` | `"true"` | Enable regex path matching |
| `spec.rules[0].host` | `ticketing.dev` | Hostname for routing |

#### Routing Rules (relevant to client)

| Path Pattern | Backend Service | Port |
|--------------|-----------------|------|
| `/?(.*)` (catch-all) | `client-srv` | `3000` |
| `/api/users/?(.*)` | `auth-srv` | `3000` |
| `/api/tickets/?(.*)` | `tickets-srv` | `3000` |
| `/api/orders/?(.*)` | `orders-srv` | `3000` |
| `/api/payments/?(.*)` | `payments-srv` | `3000` |

The client receives all non-API traffic as a catch-all route.

### `skaffold.yaml`

**Purpose:** Skaffold configuration for local Kubernetes development.

| Setting | Value | Description |
|---------|-------|-------------|
| `apiVersion` | `skaffold/v2alpha3` | Skaffold API version |
| `deploy.kubectl.manifests` | `["infra/k8s/*"]` | Kubernetes manifests to deploy |
| `build.local.push` | `false` | Don't push images to registry |

#### Client Build Artifact

| Setting | Value | Description |
|---------|-------|-------------|
| `image` | `berkinanik/ticketing-client` | Image name |
| `context` | `apps/client` | Build context directory |
| `docker.dockerfile` | `Dockerfile` | Dockerfile path |
| `sync.manual[0].src` | `src/**/*.{ts,tsx}` | Files to sync without rebuild |
| `sync.manual[0].dest` | `.` | Destination in container |

### `.github/workflows/tests.yml`

**Purpose:** GitHub Actions CI/CD pipeline.

| Setting | Value | Description |
|---------|-------|-------------|
| `on.pull_request.paths-ignore` | `["infra/**"]` | Skip for infra-only changes |
| `on.workflow_dispatch` | `{}` | Allow manual triggers |
| `jobs.test.runs-on` | `ubuntu-latest` | Runner environment |
| `Node.js version` | `20` | Node.js version used in CI |
| `pnpm version` | `9` | pnpm version used in CI |
| `jobs.test.steps` | Checkout → Install pnpm → Install Node → Cache MongoDB → Install deps → Typecheck → Test | CI steps |

### `turbo.json` (Root)

**Purpose:** Turborepo pipeline configuration affecting the client.

| Pipeline | `dependsOn` | `cache` | Description |
|----------|-------------|---------|-------------|
| `lockfile` | `[]` | (default) | Generate dedicated lockfiles |
| `test` | `["typecheck"]` | (default) | Run tests after typecheck |
| `test:ci` | `[]` | (default) | CI test run |
| `test:watch` | `["typecheck"]` | `false` | Watch mode tests |
| `typecheck` | `[]` | (default) | TypeScript type checking |

### Hardcoded Configuration Values

The following values are hardcoded in source code rather than externalized:

| Value | Location | Purpose |
|-------|----------|---------|
| `http://ingress-nginx-controller.ingress-nginx.svc.cluster.local` | `apps/client/src/http/buildClientServer.ts:16` | Internal Kubernetes DNS for server-side API calls |
| `ticketing.dev` | `apps/client/src/http/buildClientServer.ts:10` | Host header for ingress routing |
| `/` | `apps/client/src/http/buildClient.ts:7` | Base URL for client-side API calls |
| `3000` | `infra/k8s/client-depl.yaml:28,41-42` | Application port (Next.js default) |

---

## Part E: Feature Flags

**No feature flag systems were found in the client service.**

The codebase does not use:
- Remote feature flag services (LaunchDarkly, Optimizely, Split, Statsig, Unleash, ConfigCat, Flagsmith)
- Environment-based feature toggles (no `process.env.FEATURE_*` or `process.env.ENABLE_*` patterns)
- Local feature flag libraries

### Conditional UI Rendering (Not Feature Flags)

The following conditional rendering exists but represents **business logic**, not feature flags:

| Condition | Location | What It Controls |
|-----------|----------|------------------|
| `!!currentUser` | `apps/client/src/app/orders/[orderId]/page.tsx:38` | Shows checkout form only for authenticated users |
| `order.status === OrderStatus.AwaitingPayment` | `apps/client/src/app/orders/[orderId]/page.tsx:38` | Shows checkout form only for orders awaiting payment |
| `!!ticket.orderId` | `apps/client/src/app/tickets/[ticketId]/page.tsx:43` | Disables purchase button if ticket is already reserved |
| `stripeKey ? <StripeCheckout/> : null` | `apps/client/src/app/orders/[orderId]/CheckoutForm.tsx:34` | Hides Stripe button if key is missing |
| `isRefreshEnabled` | `apps/client/src/components/ExpirationTimer.tsx:9,25` | Enables auto-refresh on timer expiration (prop-driven) |
