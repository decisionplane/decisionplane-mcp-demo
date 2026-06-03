# Architecture

DecisionPlane is an **audited AI decisioning engine**. Every action your AI agents, services, or users request passes through a policy engine that produces a tamper-evident audit record before allowing, denying, or escalating the request.

## Core Concepts

| Concept | What it does |
|---------|-------------|
| **Decision** | A single allow/deny/escalate verdict on an action request |
| **Policy** | YAML rule that matches requests by actor, action, and context |
| **Audit Chain** | Append-only log of decisions, each linked by HMAC-SHA256 hash |
| **Actor** | The entity making the request — agent, human, or service |
| **Org** | An isolated tenant — separate policies, tokens, audit log, and metrics |

## Architecture Diagram

The diagram below shows the full system: your application (left), the daemon (centre), and the storage layer (right). The bottom row shows the four deployment modes.

<!-- diagram rendered at /guide/architecture -->

**Data flow for a single decision:**

```
Your App
  └─▶ SDK / HTTP / MCP / Envoy
        └─▶ Auth Middleware  (token → org_id + role)
              └─▶ Policy Engine  (first-matching rule wins)
                    └─▶ Audit Chain  (HMAC-SHA256 append)
                          └─▶ Postgres  (RLS-scoped by org)
                                └─▶ Decision record returned
```

## Components

### Policy Engine

Evaluates requests against YAML policies. Two formats are supported:

- **Rules format** — explicit conditions with operators (`eq`, `matches`, `in`, `lte`, etc.)
- **Plan DSL format** — risk-factor scoring with thresholds (`allow_below`, `approval_below`, `deny_above`)

Policies are org-scoped. Each org has its own policy directory and version namespace.

### Audit Chain

Every decision writes an immutable record linked to the previous by hash:

```
GENESIS (0x000...000)
  └─▶ Record #1  prevHash=GENESIS  hash=HMAC(key, prevHash + record)
        └─▶ Record #2  prevHash=hash#1  hash=HMAC(key, prevHash + record)
              └─▶ ...
```

`GET /v1/audit/chain` walks the full chain and returns `{ ok, count, lastHash }`. Any mutation breaks the chain — tamper detection is automatic.

The `chainHmacKeyHex` org setting activates HMAC-SHA256 keying (required for production — satisfies EDPB Art. 8.3(c) pseudonymisation). Without it the chain falls back to plain SHA-256.

### Approval Flow

When a policy outcome is `escalate`, the decision enters an approval hold:

```
escalate → POST /v1/approvals/:id/approve  (or /reject)
         → webhook fired on resolution
         → original caller receives final verdict
```

Approvals have configurable timeouts and required roles.

### Org RBAC

Every API token is scoped to an organization. Row-Level Security in Postgres ensures queries from one org never see another org's data.

Built-in roles: `owner` → `admin` → `member` → `viewer`. Custom roles can be defined per org.

---

## Deployment Modes

### Library Mode (in-process)

The engine, policy evaluator, and audit log run **in-process** against a local SQLite database. No daemon, no Postgres, no network hop in the decision path.

```bash
npm install @decisionplane/sdk
```

Best for: single-service deployments, local dev, serverless functions.

### Daemon Mode (HTTP)

Fastify server on port 3000. Postgres-backed. All features enabled: multi-org RBAC, approval flow, webhooks, telemetry, retention.

```bash
# Docker
docker run -p 3000:3000 \
  -e DATABASE_URL=postgres://... \
  -e CHAIN_HMAC_KEY_HEX=$(openssl rand -hex 32) \
  ghcr.io/fa-lbaldwin/decisionplane:latest

# Kubernetes — see k8s/manifest.yaml in the repo
```

Best for: multi-service, multi-org, or production deployments.

### MCP Mode (AI Agents)

Exposes decisioning as MCP tools so Claude, Copilot, Cursor, and other AI agents can call them directly.

```bash
npx @decisionplane/mcp
```

Tools available to the agent: `evaluate`, `simulate`, `await-approval`, `explain`, `list-policies`.

Best for: giving AI agents governed, auditable access to actions.

### Envoy Ext-AuthZ (Service Mesh)

A gRPC adapter that plugs into Envoy's `ext_authz` filter. Every HTTP request to your services passes through DecisionPlane for policy evaluation — zero application code change required.

```bash
helm install decisionplane-envoy ./adapters/envoy-ext-authz/helm
```

Best for: k8s service meshes, API gateways, zero-trust architectures.
