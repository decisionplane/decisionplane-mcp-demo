# Library vs Daemon Mode

DecisionPlane has two deployment modes. The decision path is identical in both — the only difference is whether the engine runs in-process or over HTTP.

## Library mode (default)

The engine, policy evaluator, and SQLite audit log run in the same process as your application. No network hop, no extra service to operate.

```typescript
import { DecisionPlaneClient } from '@decisionplane/sdk'
import { createLocalTransport } from '@decisionplane/sdk/local'

const transport = await createLocalTransport({
  dbPath: './audit.db',
  policyDir: './policies',
})

const client = new DecisionPlaneClient({ transport })
```

**When to use:** development, CI, serverless functions, or any environment where you own the runtime and want zero infrastructure.

**Security posture:** decisions and inputs never leave your process. For HIPAA or strict data-residency requirements, library mode is the recommended pattern (see [Compliance →](/governance/compliance)).

## Daemon mode

The daemon exposes the same engine over HTTP (Fastify). Multiple services share one audit store (Postgres in production, SQLite in dev).

```bash
# Start the daemon
pnpm dev          # SQLite, auto-seeds a dev token

# Or with Postgres
DATABASE_URL=postgres://... pnpm --filter @decisionplane/daemon start
```

```typescript
const client = new DecisionPlaneClient({
  mode: 'http',
  baseUrl: 'http://localhost:3000',
  token: process.env.DECISIONPLANE_API_KEY,
})
```

**When to use:** multi-service architectures where you want a shared, centralised audit trail, or when you need the REST API for non-TypeScript callers.

## Comparison

| | Library mode | Daemon mode |
| --- | --- | --- |
| Latency | In-process (microseconds) | HTTP round-trip |
| Audit store | SQLite (per-process) | SQLite (dev) / Postgres (prod) |
| Infrastructure | None | Fastify + database |
| Auth | N/A (in-process) | Bearer token |
| Multi-service | No | Yes |
| PHI / data residency | Customer owns runtime | Customer owns daemon host |

## Docker / Kubernetes

A production-ready `Dockerfile` and `docker-compose.yml` are included in the repo root. The `k8s/` directory has Kubernetes manifests.

```bash
docker compose up     # daemon + Postgres, port 3000
```

See the [containerization walkthrough](https://github.com/decisionplane/decisionplane/blob/main/docs/walkthroughs/2026-05-22_0900_containerization_and_k8s.md) for cluster deployment.
