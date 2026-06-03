# Policy DSL

Policies are YAML files that define when an action is approved, denied, or escalated. The DSL is implemented in `packages/policy/` and evaluated in-process by the core engine.

## Structure

```yaml
policy: refund-safety
version: 3
description: "Controls autonomous refund approvals for customer support."

applies_to:
  actions:
    - refund.process
    - refund.batch.*
  actors:
    kind: [agent]
    roles: [support]
    delegated_by_human: true  # a human must be somewhere in the chain

conditions:
  - field: resource.attributes.currency
    op: in
    value: [USD, EUR, GBP]

risk_factors:
  - name: refund_amount
    field: resource.attributes.amount
    weight: 0.6
    ranges:
      - max: 100    # ≤$100 → score 0 (low risk)
        score: 0.0
      - max: 500    # $101–500 → score 0.5
        score: 0.5
      - score: 1.0  # >$500 → score 1 (high risk)

  - name: customer_tier
    field: resource.attributes.customer_tier
    weight: 0.4
    map:
      platinum: 0.2   # trusted tier, lower risk
      standard: 0.6
      new:      0.9

thresholds:
  - max: 0.4   # aggregate score ≤0.4 → approve
    outcome: ALLOW
  - max: 0.75  # 0.4–0.75 → require a human to sign off
    outcome: REQUIRE_APPROVAL
  - outcome: DENY  # >0.75 → deny outright
```

## Key fields

| Field | Required | Purpose |
| --- | --- | --- |
| `policy` | Yes | Unique slug used in `DecisionRecord.policyVersion` |
| `version` | Yes | Monotonically increasing integer |
| `applies_to.actions` | Yes | Action IDs or globs that activate this policy |
| `applies_to.actors` | No | Actor selectors — all keys are AND-ed |
| `conditions` | No | Pre-screening checks; first failure → `DENY` |
| `risk_factors` | No | Weighted risk sources; aggregate score in `[0, 1]` |
| `thresholds` | No | Maps aggregate score to outcome |
| `approval` | No | Who can approve and timeout for `REQUIRE_APPROVAL` |

## Deny-by-default

If no policy matches a request, the engine returns `deny`. This is not configurable — the default must be deny. To allow an action, write a policy that explicitly matches it.

## Loading policies

```bash
# Policies live in ./policies/ by default
ls policies/
# refund-safety.yaml  trade-guardrails.yaml  ...

# Or specify a custom path
DECISIONPLANE_POLICY_DIR=/etc/dp/policies pnpm dev
```

## Testing policies

```typescript
import { DecisionPlaneClient } from '@decisionplane/sdk'

// Simulate without recording in the audit log
const result = await client.simulate(request)
console.log(result.outcome, result.reason)
```

`simulate()` runs the full policy engine but does not write to the audit store — useful for CI-time policy tests.
