---
layout: home

hero:
  name: "DecisionPlane"
  text: "The agentic trust layer."
  tagline: "Make every automated decision auditable, overridable, and legally defensible — for any agent, anywhere in your stack."
  actions:
    - theme: brand
      text: Get started free →
      link: /guide/quickstart
    - theme: alt
      text: What is DecisionPlane? →
      link: /guide/product
    - theme: alt
      text: Architecture & Integration →
      link: /guide/architecture
    - theme: alt
      text: GitHub →
      link: https://github.com/decisionplane/decisionplane

features:
  - icon: "⚙️"
    title: What it does
    details: A policy-governed decision engine and append-only audit chain. Submit a decision (refund, remediation, trade, takedown) and DecisionPlane returns approve / deny / require_approval plus an immutable record with the input, policy version, risk signals, and actor provenance.
    link: /guide/four-pillars
    linkText: How it works →
  - icon: "📒"
    title: Why audited decisions
    details: Auditors, regulators, and customers will ask four questions when an automated decision goes wrong — what, why, by whose authority, what if wrong. DecisionPlane is engineered so you can answer all four with a single record id, every time.
    link: /governance/compliance
    linkText: Compliance posture →
  - icon: "🔌"
    title: How it fits
    details: A library, a daemon, or a managed service — your call. Drops into agent loops (LangGraph, MCP, custom), reverse proxies (Envoy ext_authz), and pipelines (Temporal, Step Functions). Same engine, same audit record, swap the vertical with one field.
    link: /guide/deployment-modes
    linkText: Deployment modes →
---

<div class="vp-doc" style="max-width: 960px; margin: 4rem auto 0; padding: 0 24px;">

## Install in 30 seconds

```bash
npm install @decisionplane/mcp
```

Or clone the repo for the full quickstart:

```bash
git clone https://github.com/decisionplane/decisionplane
cd decisionplane && pnpm install && pnpm dev
```

→ [Read the 5-minute quickstart](/guide/quickstart) · [Browse the SDK reference](/reference/sdk) · [npm package](https://www.npmjs.com/package/@decisionplane/mcp)

## Where DecisionPlane sits in the market

| Tool | What it is | Why DecisionPlane is different |
| --- | --- | --- |
| **OPA / Rego** | Static policy-as-code | We add real-time signals, scoring, an audit UX, and managed compliance exports. |
| **Cedar** | Authorization / RBAC | Different buyer (IAM). Cedar doesn't touch business decisioning. |
| **LangGraph** | Agent orchestration | Complementary. We are the trust layer *inside* the loop, not the orchestrator. |
| **Datadog audit** | Observability | They observe. We enforce, decide, and produce the record that's observed. |
| **OneTrust** | Privacy/GRC governance | Slow procurement, no dev experience. We win on latency and API-first. |

**Positioning, in one line:** OPA gives you a policy. DecisionPlane gives you a *decision* — with a signal, an audit trail, and a guarantee.

## Built for high-stakes decisioning

| Vertical | Example | Why it matters |
| --- | --- | --- |
| **Fintech ops** | Refund approvals, dispute resolution | Chargeback evidence, regulator audit trail |
| **Incident remediation** | Auto-scaling, auto-rollback | Post-incident review, SRE accountability |
| **Trade guardrails** | Pre-trade risk checks | Market conduct, MiFID II obligation |
| **Healthcare claims/UM** | Coverage decisions | Appeals process, legal defensibility |

## Integrate in minutes

| Path | Time | Guide |
|------|------|-------|
| TypeScript / Node SDK | 5 min | [Integration guide →](/guide/integration#typescript--node-sdk) |
| Python SDK | 5 min | [Integration guide →](/guide/integration#python-sdk) |
| MCP (Claude / AI agents) | 5 min | [Integration guide →](/guide/integration#mcp-ai-agents) |
| REST API | 10 min | [Integration guide →](/guide/integration#rest-api-direct) |
| Envoy Ext-AuthZ (k8s) | 20 min | [Integration guide →](/guide/integration#envoy-ext-authz) |

→ [Full integration guide](/guide/integration) · [Architecture overview](/guide/architecture) · [5-minute quickstart](/guide/quickstart)

## Links

- **Docs:** [Quickstart](/guide/quickstart) · [Architecture](/guide/architecture) · [Integration](/guide/integration) · [SDK reference](/reference/sdk) · [Governance](/governance/)
- **Code:** [github.com/decisionplane/decisionplane](https://github.com/decisionplane/decisionplane) · [decisionplane-mcp-demo](https://github.com/decisionplane/decisionplane-mcp-demo)
- **Packages:** [`@decisionplane/mcp` on npm](https://www.npmjs.com/package/@decisionplane/mcp)
- **License:** [BSL-1.1 → Apache-2.0 after 4 years](/governance/license)

</div>
