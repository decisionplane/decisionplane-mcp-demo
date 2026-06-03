# Integration Guide

This page covers everything you need to integrate DecisionPlane into an existing application — from initial setup to production org onboarding.

## 1 — Choose Your Integration Path

| Path | When to use | Setup time |
|------|-------------|-----------|
| [TypeScript/Node SDK](#typescript--node-sdk) | Node services, Next.js, Bun | 5 min |
| [Python SDK](#python-sdk) | Python services, notebooks, LLM chains | 5 min |
| [REST API](#rest-api-direct) | Any language, curl, Postman | 10 min |
| [MCP Tools](#mcp-ai-agents) | Claude, Copilot, Cursor, AI agents | 5 min |
| [Envoy Ext-AuthZ](#envoy-ext-authz) | k8s service mesh, API gateways | 20 min |

---

## 2 — Start the Daemon

All paths except pure library mode require the daemon running.

**Docker (fastest):**
```bash
docker run -d --name dp \
  -p 3000:3000 \
  -e AUDIT_BACKEND=sqlite \
  -e AUDIT_DB=/data/audit.db \
  -v dp-data:/data \
  ghcr.io/fa-lbaldwin/decisionplane:latest
```

**From source:**
```bash
git clone https://github.com/decisionplane/decisionplane
cd decisionplane
pnpm install && pnpm dev
```

The daemon prints your dev token to the console and writes it to `.dev-api-key`.

**Kubernetes:** see `k8s/manifest.yaml` in the repo. Set `CHAIN_HMAC_KEY_HEX` and `DATABASE_URL` as secrets.

---

## 3 — Authorization

Every request requires a Bearer token. Tokens are org-scoped and carry one or more scopes.

### Create a token

```bash
# Using the dev token printed at startup
curl -s -X POST http://localhost:3000/v1/tokens \
  -H "Authorization: Bearer $(cat .dev-api-key)" \
  -H "Content-Type: application/json" \
  -d '{
    "scopes": ["decisions:write", "decisions:read", "audit:read"],
    "description": "my-service prod token"
  }'
```

Response:
```json
{
  "token": "dpk_live_...",
  "record": { "id": "...", "prefix": "dpk_live_xxx", "scopes": [...] }
}
```

::: warning
The token is shown **once**. Store it in your secret manager immediately.
:::

### Available scopes

| Scope | Grants |
|-------|--------|
| `decisions:write` | Submit decisions (`POST /v1/decisions`) |
| `decisions:read` | Read decision records |
| `decisions:override` | Apply overrides to existing decisions |
| `audit:read` | Read and verify the audit chain |
| `policies:read` | List and read policies |
| `policies:write` | Create, update, delete policies |
| `approvals:read` | Read pending approvals |
| `approvals:write` | Approve or reject escalated decisions |
| `traces:export` | Export trace data |
| `tokens:admin` | Manage API tokens |
| `orgs:admin` | Manage org settings, members, and roles |
| `webhooks:admin` | Configure webhook endpoints |

### Using the token

```bash
# All API calls use Bearer token auth
curl -H "Authorization: Bearer dpk_live_..." \
  http://localhost:3000/v1/decisions
```

---

## TypeScript / Node SDK

```bash
npm install @decisionplane/sdk
```

### Evaluate a decision

```typescript
import { DecisionPlaneClient } from '@decisionplane/sdk'

const client = new DecisionPlaneClient({
  mode: 'http',
  baseUrl: 'http://localhost:3000',
  token: process.env.DECISION_PLANE_TOKEN,
})

const result = await client.evaluate({
  id: crypto.randomUUID(),
  action: 'refund.issue',
  actor: {
    kind: 'agent',
    principal: { id: 'agent-payments-v2', display_name: 'Payments Agent' },
    claims_audit: {
      bound_by: 'http_auth',
      bound_at: new Date().toISOString(),
      bound_fields: ['principal.id'],
      claimed_fields: [],
    },
  },
  context: { amount: 250, currency: 'USD', customerId: 'cust_123' },
  requestedAt: new Date().toISOString(),
})

if (result.outcome === 'approve') {
  // proceed
} else if (result.outcome === 'escalate') {
  // wait for approval — result.approvalId is set
} else {
  // denied — result.reason explains why
}
```

### Library mode (no daemon)

Run the engine in-process against local SQLite — no network, no Docker:

```typescript
const client = new DecisionPlaneClient({ mode: 'local' })
```

Policies load from `./policies/*.yaml`. Audit log writes to `./audit.db`. Identical API to HTTP mode.

### Handle approvals

```typescript
if (result.outcome === 'escalate') {
  const approval = await client.awaitApproval(result.approvalId, {
    timeoutMs: 30_000,
    pollIntervalMs: 2_000,
  })
  if (approval.state === 'approved') { /* proceed */ }
}
```

---

## Python SDK

```bash
pip install decisionplane
```

```python
from decisionplane import DecisionPlaneClient
import uuid, os

client = DecisionPlaneClient(
    base_url="http://localhost:3000",
    token=os.environ["DECISION_PLANE_TOKEN"],
)

result = client.evaluate({
    "id": str(uuid.uuid4()),
    "action": "data.export",
    "actor": {
        "kind": "human",
        "principal": {"id": "user_42", "display_name": "Alice"},
        "claims_audit": {
            "bound_by": "http_auth",
            "bound_at": "2026-06-01T00:00:00Z",
            "bound_fields": ["principal.id"],
            "claimed_fields": [],
        },
    },
    "context": {"record_count": 50000, "destination": "external"},
    "requestedAt": "2026-06-01T00:00:00Z",
})

print(result.outcome)   # "approve" | "deny" | "escalate"
print(result.reason)
```

See [Python SDK Recipes](/guide/python-sdk-recipes) for LangChain, FastAPI, and async usage.

---

## REST API (Direct)

### Submit a decision

```bash
curl -s -X POST http://localhost:3000/v1/decisions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "id": "'"$(uuidgen)"'",
    "action": "deploy.production",
    "actor": {
      "kind": "agent",
      "principal": { "id": "ci-agent", "display_name": "CI Pipeline" },
      "claims_audit": {
        "bound_by": "http_auth",
        "bound_at": "2026-06-01T12:00:00Z",
        "bound_fields": ["principal.id"],
        "claimed_fields": []
      }
    },
    "context": { "service": "payments-api", "environment": "production" },
    "requestedAt": "2026-06-01T12:00:00Z"
  }'
```

Response:
```json
{
  "id": "01HX...",
  "outcome": "approve",
  "reason": "Deploy within business hours by authorized agent",
  "policyVersion": "acme/v3",
  "decidedAt": "2026-06-01T12:00:01Z",
  "hash": "a3f9c2..."
}
```

### Verify the audit chain

```bash
curl -s http://localhost:3000/v1/audit/chain \
  -H "Authorization: Bearer $TOKEN"
# { "ok": true, "count": 42, "lastHash": "e9a03f..." }
```

See [REST API Reference](/reference/rest-api) for the full endpoint list.

---

## MCP (AI Agents)

Add DecisionPlane as an MCP server so AI agents can call decisioning tools directly.

### Setup

```bash
# Start the MCP server
DECISION_PLANE_URL=http://localhost:3000 \
DECISION_PLANE_TOKEN=$TOKEN \
npx @decisionplane/mcp
```

Or add to your `claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "decisionplane": {
      "command": "npx",
      "args": ["@decisionplane/mcp"],
      "env": {
        "DECISION_PLANE_URL": "http://localhost:3000",
        "DECISION_PLANE_TOKEN": "dpk_live_..."
      }
    }
  }
}
```

### Tools available to the agent

| Tool | Description |
|------|-------------|
| `evaluate` | Submit a decision request and get an outcome |
| `simulate` | Dry-run evaluation (no audit record written) |
| `await-approval` | Block until an escalated decision is resolved |
| `explain` | Get plain-language explanation of why a decision was made |
| `list-policies` | List active policies for the current org |

The agent submits decisions with its own identity in the `actor` field — every action is attributed and auditable.

---

## Envoy Ext-AuthZ

Plug DecisionPlane into Envoy's external authorization filter. Every HTTP request to your services is evaluated before reaching your code — zero application changes required.

### Install via Helm

```bash
helm install dp-authz ./adapters/envoy-ext-authz/helm \
  --set decisionplane.url=http://decisionplane-daemon:3000 \
  --set decisionplane.token=$TOKEN
```

### Envoy filter config

```yaml
http_filters:
  - name: envoy.filters.http.ext_authz
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthz
      grpc_service:
        envoy_grpc:
          cluster_name: decisionplane-authz
      transport_api_version: V3
```

The adapter maps HTTP method + path + headers into a `DecisionRequest` and returns `200 OK` for `approve`, `403 Forbidden` for `deny`, or a redirect for `escalate`.

---

## 4 — Org Onboarding

### Create your first org

```bash
# With a platform bootstrap token (tokens:admin scope)
curl -s -X POST http://localhost:3000/v1/orgs \
  -H "Authorization: Bearer $BOOTSTRAP_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "slug": "acme-corp", "name": "Acme Corp" }'
```

Response includes an org-scoped token. **Store it** — it is shown once.

### Add team members

```bash
curl -s -X POST http://localhost:3000/v1/orgs/$ORG_ID/members \
  -H "Authorization: Bearer $ORG_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "principalId": "alice@acme.com", "role": "admin" }'
```

### Configure org settings

```bash
# Set per-org HMAC key for audit chain (required for production)
HMAC_KEY=$(openssl rand -hex 32)

curl -s -X PATCH http://localhost:3000/v1/orgs/$ORG_ID/settings \
  -H "Authorization: Bearer $ORG_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"chainHmacKeyHex\": \"$HMAC_KEY\",
    \"auditRetentionDays\": 2555,
    \"rateLimitRpm\": 1000,
    \"features\": { \"approvals\": true, \"webhooks\": true, \"traces\": true }
  }"
```

::: warning
Back up `chainHmacKeyHex` externally. If lost, the existing chain cannot be re-verified.
:::

---

## 5 — Writing Your First Policy

Policies live in `./policies/` (library mode) or `/data/policies/` (daemon/Docker).

### Rules format (simple conditions)

```yaml
# policies/refund.yaml
version: "1.0"
rules:
  - id: refund-auto-approve
    description: Auto-approve refunds under $100
    conditions:
      - field: amount
        op: lte
        value: 100
    outcome: approve
    reason: Refund within auto-approval threshold
    priority: 30

  - id: refund-deny-large
    description: Deny refunds over $1000
    conditions:
      - field: amount
        op: gt
        value: 1000
    outcome: deny
    reason: Exceeds maximum refund limit
    priority: 20
```

### Plan DSL format (risk scoring)

```yaml
# policies/payments.yaml
policy: payments.risk
version: 1
applies_to:
  actions: [payment.charge]

risk_factors:
  - source: blast_radius
    weight: 0.70
    inputs:
      field: context.amount
      soft_limit: 500
      hard_limit: 5000

thresholds:
  allow_below: 0.10
  approval_below: 0.70
  deny_above: 0.70
```

See [Policy DSL Reference](/guide/policy-dsl) for the full schema.

---

## 6 — Features & Capabilities

| Feature | Description | Endpoint / Config |
|---------|-------------|-------------------|
| **Decisions** | Submit and retrieve verdicts | `POST /v1/decisions` |
| **Audit chain** | Tamper-evident HMAC-linked log | `GET /v1/audit/chain` |
| **Approval flow** | Human-in-the-loop escalation | `POST /v1/approvals/:id/approve` |
| **Policy CRUD** | Manage policies via API or UI | `GET|POST /v1/policies` |
| **Overrides** | Retroactively modify a decision | `POST /v1/decisions/:id/overrides` |
| **Webhooks** | Fire callbacks on deny/escalate | `POST /v1/webhooks` |
| **Traces** | Per-request actor telemetry | `GET /v1/traces` |
| **Org RBAC** | Multi-tenant with roles | `POST /v1/orgs` |
| **Metrics** | Prometheus + per-org aggregates | `GET /metrics`, `GET /v1/orgs/:id/metrics` |
| **Retention** | Configurable 7-year archive | `auditRetentionDays` org setting |
| **MCP tools** | AI agent integration | `npx @decisionplane/mcp` |
| **Envoy adapter** | Service mesh auth | Helm chart |
| **Python SDK** | Native Python client | `pip install decisionplane` |
| **Chain HMAC** | EDPB Art.8.3(c) pseudonymisation | `chainHmacKeyHex` org setting |
