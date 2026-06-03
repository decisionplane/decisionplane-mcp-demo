# Policy Schema Reference

Policies are YAML files loaded by `@decisionplane/policy`. The JSON Schema is exported from `packages/policy/src/plan/json-schema.ts`.

## Top-level fields

```yaml
policy: <slug>       # required — kebab/dot slug
version: <int>       # required — monotonically increasing
description: <str>   # optional

applies_to:          # required
  actions: [...]     # required — action IDs or globs
  actors: {...}      # optional — actor selectors (AND-ed)

conditions: [...]    # optional — pre-screening checks
risk_factors: [...]  # optional — weighted risk sources
thresholds: [...]    # optional — score → outcome mapping
approval: {...}      # optional — approval routing for REQUIRE_APPROVAL
```

## applies_to.actors

All keys present are AND-ed:

```yaml
actors:
  kind: [human, agent]             # any-of
  roles: [sre, platform]           # any-of
  tags: [budget-allocated]         # all-of
  delegated_by_human: true         # root of chain is a human
  delegated_by_human_with_mfa: true# root human used MFA
  max_delegation_depth: 1          # 0=direct, 1=one hop, etc.
```

## conditions

Each condition is a field check. All conditions must pass or the request is denied:

```yaml
conditions:
  - field: resource.attributes.currency
    op: in
    value: [USD, EUR, GBP]
  - field: context.has_runbook
    op: eq
    value: true
```

Operators: `eq`, `neq`, `gt`, `lt`, `gte`, `lte`, `in`, `matches`, `contains`.

## risk_factors

```yaml
risk_factors:
  - name: refund_amount
    field: resource.attributes.amount
    weight: 0.6        # contribution to aggregate score
    ranges:            # piecewise linear mapping
      - max: 100
        score: 0.0
      - max: 500
        score: 0.5
      - score: 1.0    # catch-all (no max)

  - name: customer_tier
    field: resource.attributes.customer_tier
    weight: 0.4
    map:               # categorical mapping
      platinum: 0.2
      standard: 0.6
      new:      0.9
```

Aggregate score = `sum(factor.weight * factor.score)` / `sum(factor.weight)`. Score is always in `[0, 1]`.

## thresholds

```yaml
thresholds:
  - max: 0.4
    outcome: ALLOW
  - max: 0.75
    outcome: REQUIRE_APPROVAL
  - outcome: DENY      # catch-all (no max)
```

If no threshold matches, the outcome is `DENY` (deny-by-default).

## approval

Required when any threshold maps to `REQUIRE_APPROVAL`:

```yaml
approval:
  approvers:
    - kind: human
      roles: [support-lead, manager]
  timeoutMs: 300000   # 5 minutes; on timeout → DENY
```

## Full example

```yaml
policy: refund-safety
version: 3
description: "Autonomous refund approvals for customer support agents."

applies_to:
  actions:
    - refund.process
    - refund.batch.*
  actors:
    kind: [agent]
    roles: [support]
    delegated_by_human: true

conditions:
  - field: resource.attributes.currency
    op: in
    value: [USD, EUR, GBP]

risk_factors:
  - name: refund_amount
    field: resource.attributes.amount
    weight: 0.6
    ranges:
      - max: 100
        score: 0.0
      - max: 500
        score: 0.5
      - score: 1.0

  - name: customer_tier
    field: resource.attributes.customer_tier
    weight: 0.4
    map:
      platinum: 0.2
      standard: 0.6
      new:      0.9

thresholds:
  - max: 0.4
    outcome: ALLOW
  - max: 0.75
    outcome: REQUIRE_APPROVAL
  - outcome: DENY

approval:
  approvers:
    - kind: human
      roles: [support-lead]
  timeoutMs: 300000
```
