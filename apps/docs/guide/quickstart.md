# Quickstart: Install → First Decision → Audit Chain

Get a verified audit chain in under 5 minutes.

## Prerequisites

- Node.js 22 LTS or later
- pnpm (or npm / yarn)

---

## Step 1 — Clone and install (60 seconds)

```bash
git clone https://github.com/decisionplane/decisionplane
cd decisionplane
pnpm install
```

Start the daemon in one terminal. It seeds a dev token, loads the bundled policies, and listens on port 3000:

```bash
pnpm dev
```

Expected output:

```
DecisionPlane daemon listening on http://localhost:3000
Dev API key → .dev-api-key
```

---

## Step 2 — Submit your first decision (90 seconds)

Create `quickstart.ts` in the repo root:

```typescript
import { DecisionPlaneClient } from '@decisionplane/sdk'
import { randomUUID } from 'crypto'
import { readFileSync } from 'fs'

const token = readFileSync('.dev-api-key', 'utf8').trim()

const client = new DecisionPlaneClient({
  mode: 'http',
  baseUrl: 'http://localhost:3000',
  token,
})

// --- Decision 1: low-risk refund (should be approved) ---
const refundId = randomUUID()

const record = await client.submitDecision(
  {
    id: refundId,
    action: 'refund.process',
    actor: {
      kind: 'agent',
      principal: { id: 'support-bot-v2', roles: ['support'] },
    },
    resource: {
      type: 'refund',
      id: 'order-8821',
      attributes: { amount: 50, currency: 'USD', customer_tier: 'standard' },
    },
    context: {},
    requestedAt: new Date().toISOString(),
  },
  randomUUID() // idempotency key
)

console.log('outcome     :', record.outcome)       // "approve"
console.log('reason      :', record.reason)        // plain-English policy reason
console.log('policyVersion:', record.policyVersion) // e.g. "refund-safety@3"
console.log('hash        :', record.hash)           // SHA-256 position in the chain
```

Run it:

```bash
npx tsx quickstart.ts
```

Expected:

```
outcome     : approve
reason      : Amount 50 USD is within the auto-approve threshold.
policyVersion: refund-safety@3
hash        : abc123...
```

---

## Step 3 — Apply an override (60 seconds)

Append to `quickstart.ts`:

```typescript
// --- Decision 2: high-risk refund (should be denied) ---
const highRisk = await client.submitDecision(
  {
    id: randomUUID(),
    action: 'refund.process',
    actor: {
      kind: 'agent',
      principal: { id: 'support-bot-v2', roles: ['support'] },
    },
    resource: {
      type: 'refund',
      id: 'order-9999',
      attributes: { amount: 750, currency: 'USD', customer_tier: 'standard' },
    },
    context: {},
    requestedAt: new Date().toISOString(),
  },
  randomUUID()
)

console.log('\nhigh-risk outcome:', highRisk.outcome) // "deny"

// --- Override: a supervisor approves it anyway ---
const override = await client.applyOverride(
  highRisk.id,
  {
    outcome: 'approve',
    reason: 'Manual review confirmed valid purchase — supervisor authorised.',
    overriddenBy: 'alice@example.com',
  },
  randomUUID()
)

console.log('override outcome:', override.outcome)  // "approve"
console.log('override hash   :', override.hash)     // new link in the chain
```

The original `highRisk` record is unchanged — the override is a separate append-only record that references it. This is how DecisionPlane answers *"what if it was wrong?"* without destroying the original evidence.

---

## Step 4 — Verify the audit chain (30 seconds)

Append to `quickstart.ts`:

```typescript
// --- Verify the whole chain ---
const chain = await client.verifyChain()

console.log('\nchain.ok       :', chain.ok)         // true
console.log('chain.length   :', chain.length)       // 3
console.log('chain.headHash :', chain.headHash)     // SHA-256 of the latest record
console.log('chain.verifiedAt:', chain.verifiedAt)

if (!chain.ok) {
  throw new Error(`Chain broken at record #${chain.brokenAt}: ${chain.message}`)
}
```

Run the whole file:

```bash
npx tsx quickstart.ts
```

Expected final output:

```
outcome     : approve
reason      : Amount 50 USD is within the auto-approve threshold.
policyVersion: refund-safety@3
hash        : abc123...

high-risk outcome: deny
override outcome: approve
override hash   : def456...

chain.ok       : true
chain.length   : 3
chain.headHash : def456...
chain.verifiedAt: 2026-05-24T...Z
```

`chain.ok: true` means no record has been tampered with. Each `hash` covers the prior record's hash plus the full record content — any edit anywhere in history breaks the chain.

---

## CLI shortcut

You can also verify the chain without any code:

```bash
curl -s -H "Authorization: Bearer $(cat .dev-api-key)" \
  http://localhost:3000/v1/audit/chain | jq .
```

---

## What you just proved

| Question | Where the answer lives |
| --- | --- |
| **What was decided?** | `record.outcome` + `record.reason` |
| **Why?** | `record.policyVersion` + input snapshot in `record.request` |
| **By whose authority?** | `actor` in the request; `overriddenBy` on the override |
| **What if it was wrong?** | Override appended, original intact, chain re-verifiable at any time |

---

## Next steps

- [Your First Decision (deep dive) →](/guide/first-decision) — full `DecisionRequest` schema and options
- [Inspect the Audit Chain →](/guide/audit-chain) — reading individual records, chain internals
- [The Four Pillars →](/guide/four-pillars) — the framework behind the four questions above
- [Policy DSL →](/guide/policy-dsl) — write your own policies
