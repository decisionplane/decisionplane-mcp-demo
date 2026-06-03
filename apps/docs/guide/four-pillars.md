# The Four Pillars

Every automated decision must answer four durable questions to be legally defensible, auditable, and overridable. DecisionPlane is built around these four pillars — they are not a UX choice, they are the schema.

## 1. What was decided?

The `DecisionRecord` captures the outcome (`approve`, `deny`, or `escalate`) and a `reason` string from the policy that matched. The full request snapshot is embedded in the record so there is never any ambiguity about what inputs produced the outcome.

```typescript
record.outcome      // "approve" | "deny" | "escalate"
record.reason       // "Refund amount within low-risk threshold (≤$100)"
record.decidedAt    // ISO timestamp
```

## 2. Why?

The `policyVersion` field names the exact policy and version that matched. The `request` snapshot captures every input the engine saw — amount, customer tier, actor roles — so the reasoning is fully reproducible.

```typescript
record.policyVersion   // "refund-safety@3"
record.request         // full DecisionRequest snapshot
```

In the DSL evaluator, risk factors and weights are recorded alongside the score that crossed the threshold. The outcome is never a black box.

## 3. By whose authority?

The `actor` field in every `DecisionRequest` carries provenance end-to-end: kind (`human`, `agent`, `system`), principal ID, roles, delegation chain depth, and MFA attestation. For overrides, `overriddenBy` names the human who intervened.

```typescript
record.request.actor.kind              // "agent"
record.request.actor.principal.id     // "support-bot-v2"
record.request.actor.principal.roles  // ["support"]
record.decidedBy                       // engine / policy ID
```

The `delegated_by_human_with_mfa` policy condition lets you require that an MFA-authenticated human is somewhere in the delegation chain before certain actions are approved.

## 4. What if it was wrong?

The override path appends a new `DecisionRecord` to the audit chain. It references the original `requestId` and records who overrode it, why, and with what new outcome. The original record is never mutated.

The SHA-256 hash chain guarantees that no record — original or override — can be quietly altered after the fact. Anyone with read access can run `verifyChain()` and detect tampering.

```typescript
// Override appends a new record. Original is unchanged.
const override = await client.applyOverride(record.id, {
  outcome: 'approve',
  reason: 'Supervisor manual review confirmed.',
  overriddenBy: 'alice@example.com',
}, idempotencyKey)
```

---

These four pillars are why financial regulators, incident commanders, and compliance teams can accept automated decisions from DecisionPlane — the evidence is always there, it is always tamper-evident, and the human override path is always open.
