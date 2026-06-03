# SDK API Reference

`@decisionplane/sdk` — thin client for the DecisionPlane decision engine.

## Installation

```bash
npm install @decisionplane/sdk
# or
pnpm add @decisionplane/sdk
```

## DecisionPlaneClient

```typescript
import { DecisionPlaneClient } from '@decisionplane/sdk'
```

### Constructor

```typescript
// HTTP mode (daemon)
const client = new DecisionPlaneClient({
  mode: 'http',
  baseUrl: 'http://localhost:3000',
  token: process.env.DECISIONPLANE_API_KEY,
  retries: { attempts: 3, backoffMs: 200 },  // optional
})

// Transport mode (library / custom)
const client = new DecisionPlaneClient({
  transport: myTransport,
})
```

### Methods

#### `health()`

```typescript
const result = await client.health()
// { ok: boolean; policy_version: string }
```

#### `submitDecision(req, idempotencyKey, opts?)`

Submit a decision request to the engine. Returns an immutable `DecisionRecord`.

```typescript
const record = await client.submitDecision(
  {
    id: randomUUID(),
    action: 'refund.process',
    actor: { kind: 'agent', principal: { id: 'bot-v2', roles: ['support'] } },
    resource: { type: 'refund', id: 'order-8821', attributes: { amount: 50 } },
    context: {},
    requestedAt: new Date().toISOString(),
  },
  randomUUID()  // idempotency key
)
```

#### `simulate(req, opts?)`

Run the policy engine without writing to the audit store. Useful for CI-time policy tests.

```typescript
const result = await client.simulate(request)
// Same shape as submitDecision, but no audit record is written.
```

#### `getDecision(recordId)`

```typescript
const record = await client.getDecision(recordId)
// DecisionRecord | null
```

#### `applyOverride(recordId, body, idempotencyKey, opts?)`

Append an override record. Never mutates the original.

```typescript
const override = await client.applyOverride(recordId, {
  outcome: 'approve',           // "approve" | "deny" | "escalate"
  reason: 'Supervisor sign-off.',
  overriddenBy: 'alice@example.com',
}, randomUUID())
```

#### `verifyChain()`

Verify the append-only audit hash chain.

```typescript
const result = await client.verifyChain()
// { ok: boolean; length: number; headHash: string | null; verifiedAt: string; brokenAt?: number }
```

#### `listPolicies(opts?)`

```typescript
const { policies } = await client.listPolicies({ agentKind: 'support-agent' })
// policies: PolicySummary[]
```

#### `awaitApproval(token, opts?)`

Long-poll for a `REQUIRE_APPROVAL` outcome. Blocks until a human approves or denies, or until `timeoutMs`.

```typescript
const approval = await client.awaitApproval(token, { timeoutMs: 30_000 })
```

## Types

See [Audit Record Schema →](/reference/audit-schema) and [Policy Schema →](/reference/policy-schema) for the full type definitions.

```typescript
import type {
  DecisionRequest,
  DecisionRecord,
  Outcome,
  OverrideBody,
  ChainResult,
} from '@decisionplane/sdk'
```
