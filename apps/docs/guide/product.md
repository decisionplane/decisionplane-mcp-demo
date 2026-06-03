# What is DecisionPlane?

**DecisionPlane is the policy and audit layer between your application and every automated decision that can hurt someone.**

---

## The Problem

Automated systems make thousands of high-stakes decisions every day — approving refunds, executing remediations, denying healthcare claims, triggering trades. Most engineering teams ship those decisions as raw code: a function returns `true`, a webhook fires, money moves. Nobody questions the architecture until the first regulator asks for the audit trail, the first chargeback dispute lands in court, or an AI agent approves $200,000 in fraudulent refunds over a weekend with no human in the loop.

The problem isn't that automation is wrong. The problem is that ungoverned automation accumulates silent risk. When a decision is embedded in application logic — a conditional, a model output, a workflow step — it has no provenance. You can't prove what inputs produced it, which rule triggered it, or who was accountable. When that decision turns out to be wrong, you can't reproduce the reasoning, you can't demonstrate due diligence to a regulator, and you can't make the victim whole with any paper trail behind you.

The rise of AI agents makes this worse by an order of magnitude. An AI agent acting autonomously can chain dozens of high-stakes decisions in seconds, each dependent on the last, with no natural checkpoint for human review. Without an explicit trust layer, the question "who authorized this?" has no answer — and increasingly, that is the question regulators, boards, and courts are asking first.

---

## What DecisionPlane Does

DecisionPlane intercepts every automated decision at the moment it is made, evaluates it against a policy you control, and writes an immutable, tamper-evident record that can answer four questions — forever: **what was decided, why, by whose authority, and what if it was wrong.**

It is not an orchestrator. It is not an observability tool. It is the governance layer that turns automated decisions from anonymous code paths into auditable, overridable, legally defensible events.

---

## Who It Is For

### Developers building AI agents that take real actions

Your agent calls an API, submits a form, cancels an order, or sends a message. Each of those actions is a decision with real-world consequences. Without a trust layer, your agent is acting without authority — it has no policy boundary, no escalation path, and no audit trail if something goes wrong.

DecisionPlane gives your agent a `evaluate` call it makes before any consequential action. The engine approves, denies, or routes the action to a human for review. The agent gets a structured verdict with a reason and a record ID. You get an audit chain you can read back to any stakeholder.

### Fintech and healthcare engineering teams under regulatory scrutiny

MiFID II, the CFPB, HIPAA, SOC 2 — regulators in financial services and healthcare share a common expectation: if an automated system makes a decision that affects a customer, you must be able to reconstruct that decision completely. Who requested it, what data it saw, which rule applied, and what the outcome was.

DecisionPlane was designed for this use case. Every `DecisionRecord` captures the full request snapshot, the matched policy version, the actor provenance chain, and a SHA-256 position in the audit chain. Compliance exports, retention schedules, and HMAC-keyed tamper detection are first-class features, not afterthoughts.

### Platform teams managing multiple AI services at scale

You have five teams shipping AI-powered features. Each team has its own model, its own prompts, its own action surface. Policy enforcement is scattered across codebases. When the CTO asks "what can our AI actually do on behalf of a user?", nobody has a complete answer.

DecisionPlane centralizes policy authority. Org-scoped policy namespaces mean each team owns its rules. The shared audit chain means every decision — regardless of which service made it — is visible in one place. When a new AI feature ships, the platform team wires it into DecisionPlane and the governance is inherited, not negotiated.

---

## The Four Guarantees

DecisionPlane is built around four questions that every automated decision must be able to answer. These are not UX conventions — they are the schema. See [The Four Pillars →](/guide/four-pillars) for the full technical specification.

### 1. What was decided?

Every `DecisionRecord` captures the outcome (`approve`, `deny`, or `escalate`), a plain-English reason string from the matching policy rule, and the full request snapshot. There is never any ambiguity about what happened.

> **Scenario:** A support bot denies a $750 refund. Six months later, the customer disputes the decision. You open the record and see: outcome `deny`, reason `"Amount 750 USD exceeds the auto-approve threshold of $500"`, policy version `refund-safety@3`, timestamp, and the full request that triggered it. The evidence is complete, self-contained, and produced at decision time — not reconstructed from logs.

### 2. Why?

The `policyVersion` field names the exact policy and version that matched. The embedded `request` snapshot captures every input the engine evaluated — amount, customer tier, actor roles, risk signals, score. The reasoning is fully reproducible and independently verifiable.

> **Scenario:** Your risk model is updated and a previously-approved pattern now denies. A trader asks why their order was blocked this morning but passed yesterday. You compare the two `DecisionRecord`s: same request, different `policyVersion`, different outcome. The answer is in the record.

### 3. By whose authority?

The `actor` field in every `DecisionRequest` carries full provenance: kind (`human`, `agent`, `system`), principal ID, roles, delegation chain depth, and MFA attestation. For overrides, `overriddenBy` names the human who intervened.

> **Scenario:** An AI agent in your incident response pipeline triggers an auto-rollback. Post-incident review asks: was this authorized? The record shows `actor.kind: "agent"`, `actor.principal.id: "incident-bot-v1"`, `actor.principal.roles: ["sre"]`, and a delegation chain tracing back to the on-call engineer who enabled autonomous mode at 2:17 AM. The authority chain is complete.

### 4. What if it was wrong?

The override path appends a new `DecisionRecord` to the audit chain, referencing the original `requestId` and recording who overrode it, why, and with what new outcome. The original record is never mutated. The SHA-256 hash chain guarantees that no record — original or override — can be quietly altered after the fact.

> **Scenario:** An AI agent denies a healthcare prior-authorization. The treating physician escalates. A clinical reviewer approves the override. The audit chain now has two records for the same request: the original denial with its full policy context, and the override with the reviewer's name, rationale, and timestamp. The appeals trail is self-contained in the chain — no separate ticketing system required.

---

## How It Compares

| Tool | What it is | The honest comparison |
|------|------------|----------------------|
| **OPA / Rego** | Static policy-as-code | OPA evaluates rules. DecisionPlane adds real-time scoring, an audit trail, actor provenance, and override management. OPA tells you what the policy says; DecisionPlane produces the record that proves what happened. |
| **Cedar** | Authorization / RBAC | Cedar is built for IAM and access control. It doesn't touch business decisioning, risk scoring, or compliance audit chains. Different buyer, different problem. |
| **LangGraph** | Agent orchestration | Complementary, not competing. LangGraph orchestrates the steps of an agent workflow. DecisionPlane is the trust layer *inside* that workflow — the call your agent makes before it acts. |
| **Datadog audit** | Observability | Datadog observes what happened. DecisionPlane decides what is allowed to happen. The audit record DecisionPlane produces is what Datadog (or your SIEM) ingests. |
| **OneTrust** | Privacy / GRC governance | Procurement-heavy, no developer experience, no real-time enforcement. DecisionPlane wins on latency (sub-millisecond in library mode), API-first design, and direct integration into the code path. |

**The one-line positioning:** OPA gives you a policy. DecisionPlane gives you a *decision* — with a record, a signal, and a guarantee.

---

## Get Started

Ready to add a trust layer to your next automated action?

| Path | What you'll build |
|------|-------------------|
| [**5-minute quickstart →**](/guide/quickstart) | Submit your first decision, apply an override, verify the audit chain |
| [**Integration guide →**](/guide/integration) | TypeScript, Python, MCP, REST, Envoy — pick your stack |
| [**GitHub →**](https://github.com/decisionplane/decisionplane) | Source code, examples, issues, discussions |

You can be producing verified audit records in under five minutes. The governance layer doesn't have to wait for the architecture review.
