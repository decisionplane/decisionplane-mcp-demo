# Your First Decision

This guide walks through submitting a decision request, reading the result, and applying an override. It assumes you have the daemon running from the [install guide](/guide/install).

## 1. Build a DecisionRequest

Every decision starts with a `DecisionRequest` — a Zod-validated record that captures who is asking, what they want to do, and what resource they're acting on.

```typescript
import { DecisionPlaneClient } from '@decisionplane/sdk'
import { randomUUID } from 'crypto'

const client = new DecisionPlaneClient({
  mode: 'http',
  baseUrl: 'http://localhost:3000',
  token: process.env.DECISIONPLANE_API_KEY,
})

const request = {
  id: randomUUID(),
  action: 'refund.process',
  actor: {
    kind: 'agent',
    principal: {
      id: 'support-bot-v2',
      roles: ['support'],
    },
  },
  resource: {
    type: 'refund',
    id: 'order-8821',
    attributes: {
      amount: 50,
      currency: 'USD',
      customer_tier: 'standard',
    },
  },
  context: {},
  requestedAt: new Date().toISOString(),
}
```

## 2. Submit the decision

```typescript
const idempotencyKey = randomUUID()
const record = await client.submitDecision(request, idempotencyKey)

console.log(record.outcome)      // "approve" | "deny" | "escalate"
console.log(record.reason)       // plain-English policy reason
console.log(record.policyVersion)// e.g. "refund-safety@3"
console.log(record.hash)         // SHA-256 position in the audit chain
```

The `DecisionRecord` is immutable. It captures the full `request` snapshot, the `outcome`, the `reason`, and the `hash` that links it to the previous record in the chain.

## 3. Apply an override (when you disagree)

Overrides never mutate the original record. They append a new record to the chain that references the original:

```typescript
import { randomUUID } from 'crypto'

const override = await client.applyOverride(
  record.id,
  {
    outcome: 'approve',
    reason: 'Manual review confirmed valid purchase — supervisor authorised.',
    overriddenBy: 'alice@example.com',
  },
  randomUUID() // idempotency key
)

console.log(override.outcome)     // "approve"
console.log(override.hash)        // new position in the chain
```

The original `record` is unchanged. Both records are visible in the audit chain, answering *what if it was wrong?* with a full, tamper-evident trail.

## Next: inspect the audit chain

→ [Verify the audit chain →](/guide/audit-chain)
