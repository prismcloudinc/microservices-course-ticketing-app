---
repository: microservices-course-ticketing-app
type: monorepo
service_count: 8
last_analyzed: 2026-11-10T20:25:08.791Z
commit_sha: a403fff288018e2dbbe4bd55b6d96446a2ef30ed
schema_version: "1.0"
---

# Architecture Overview

## Service Inventory

| Service | Type | Framework | APIs | Dependencies |
|---------|------|-----------|------|--------------|
| nats-test | shared-library | — | 0 | 1 internal |
| @repo/jest-config | shared-library | — | 0 | 1 internal |
| tickets | microservice | express | 6 | 3 internal |
| payments | microservice | express | 4 | 6 internal |
| orders | microservice | express | 5 | 6 internal |
| expiration | microservice | — | 0 | 5 internal |
| auth | microservice | express | 5 | 8 internal |
| client | web-app | nextjs | 1 | 9 internal |

## Repository Structure

- **Type**: monorepo
- **Services**: 8
- **Shared Libraries**: 2
- **Config Files**: .prettierrc, turbo.json, pnpm-workspace.yaml
