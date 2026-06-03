# Audit Record Schema

All schema types are defined in `@decisionplane/core` using Zod. The types below reflect the frozen v1 schema.

## DecisionRequest

```typescript
interface DecisionRequest {
  id: string           // UUID — caller-generated
  action: string       // e.g. "refund.process"
  actor: ActorV1       // see below
  resource?: {
    type: string       // e.g. "refund"
    id: string         // e.g. "order-8821"
    attributes: Record<string, unknown>
  }
  context: Record<string, unknown>
  requestedAt: string  // ISO 8601 datetime
}
```

## ActorV1

```typescript
interface ActorV1 {
  kind: 'human' | 'agent' | 'service'
  principal: {
    id: string
    display_name?: string
    tenant_id?: string  // Multi-tenant organization / tenant context ID
  }
  roles?: string[]
  tags?: string[]
}
```

## DecisionRecord

```typescript
interface DecisionRecord {
  id: string           // UUID — engine-generated
  requestId: string    // UUID — links to DecisionRequest.id
  outcome: 'approve' | 'deny' | 'escalate'
  reason: string       // plain-English policy reason
  decidedAt: string    // ISO 8601 datetime
  decidedBy: string    // engine/policy identifier
  policyVersion: string// e.g. "refund-safety@3"
  prevHash: string     // SHA-256 of the previous record
  hash: string         // SHA-256 of this record (includes prevHash)
  request?: DecisionRequest  // embedded input snapshot
  latencyMs?: number   // evaluation latency in milliseconds (optional)
}
```

## Hash chain

Each `hash` is computed as:

```
SHA-256(prevHash || JSON.stringify(cleanRecord))
```

where `cleanRecord` is the `DecisionRecord` omitting the optional `request` object and the `hash` itself.

The first record in the chain uses a genesis `prevHash` of `000...0` (64 hex zeros).

`verifyChain()` re-derives every hash in insert order and reports the first mismatch.

## Override record

Overrides are `DecisionRecord` instances where:
- `decidedBy` is set to the override actor ID
- `reason` is the override reason
- `request.id` points to the original request being overridden

The original record is never mutated.

## Outcome enum

```typescript
type Outcome = 'approve' | 'deny' | 'escalate'
```

| Value | Meaning |
| --- | --- |
| `approve` | Action is permitted |
| `deny` | Action is blocked |
| `escalate` | Requires human review before proceeding |
