# decisionplane-mcp-demo

> Seven battle-tested policies for governing AI agents — install in 30 seconds, no account required.

This repo is the official policy bundle for [DecisionPlane](https://github.com/decisionplane/decisionplane).
Clone it, drop the policies into `~/dp-policies/`, add the MCP server to Claude Desktop or Claude Code,
and every consequential action your AI agent attempts is gated by real governance — locally, with no
network calls and no daemon.

[![npm](https://img.shields.io/npm/v/@decisionplane/mcp?label=%40decisionplane%2Fmcp)](https://www.npmjs.com/package/@decisionplane/mcp)
[![License](https://img.shields.io/badge/license-Apache--2.0-blue)](LICENSE)

---

## Demo

<!-- asciinema cast — record with: asciinema rec demo.cast -->
<!-- embed: [![asciicast](https://asciinema.org/a/XXXXXX.svg)](https://asciinema.org/a/XXXXXX) -->

> 📹 **Demo cast coming soon.** To record your own: `asciinema rec demo.cast` then follow the
> [30-second quickstart](#-30-second-quickstart) below.

---

## ⚡ 30-second quickstart

```bash
# 1. Install the MCP server
npm install -g @decisionplane/mcp

# 2. Copy the demo policies
git clone https://github.com/decisionplane/decisionplane-mcp-demo
mkdir -p ~/dp-policies
cp decisionplane-mcp-demo/policies/*.yaml ~/dp-policies/
```

### Claude Desktop

Merge `claude-desktop-config.json` into your Claude Desktop config
(`~/Library/Application Support/Claude/claude_desktop_config.json` on macOS,
`%APPDATA%\Claude\claude_desktop_config.json` on Windows):

```json
{
  "mcpServers": {
    "decisionplane": {
      "command": "dp-mcp",
      "env": {
        "DECISIONPLANE_MODE": "local",
        "DECISIONPLANE_ACTOR_ID": "claude-desktop",
        "DECISIONPLANE_ACTOR_ROLE": "agent"
      }
    }
  }
}
```

Restart Claude Desktop. Your agent is now governed.

### Claude Code

Add `claude-code-settings.json` to `.claude/settings.json` in your project (or
`~/.claude/settings.json` globally):

```json
{
  "mcpServers": {
    "decisionplane": {
      "command": "dp-mcp",
      "env": {
        "DECISIONPLANE_MODE": "local",
        "DECISIONPLANE_ACTOR_ID": "claude-code",
        "DECISIONPLANE_ACTOR_ROLE": "agent"
      }
    }
  }
}
```

---

## Decision flow

Every time your agent calls `dp_evaluate`, the engine checks the action against all policies in
`~/dp-policies/`. The most restrictive match wins:

| Decision | What happens |
|---|---|
| `ALLOW` | Action proceeds immediately |
| `REQUIRE_APPROVAL` | Agent pauses and calls `dp_await_approval`. Run `dp-mcp approve` in a terminal to resolve. |
| `DENY` | Agent cannot take the action |

### Approving a pending action

When a policy triggers `REQUIRE_APPROVAL`, the agent is paused. Run the approve command in any terminal:

```
$ dp-mcp approve

DecisionPlane — 1 pending approval
────────────────────────────────────────────────────────────

[1] appr_1748025600000_a1b2c3d4
    Action   : k8s.deploy.deployment
    Resource : deployment/payments-api
    Actor    : claude-desktop
    Risk     : 0.420
    Reasoning: Hotfix for INC-4521
    Expires  : 28m44s
    [a]pprove / [d]eny / [s]kip ?
```

One keystroke resolves the approval. The agent's `dp_await_approval` call returns immediately.

---

## Included policies

| File | Actions governed | ALLOW | REQUIRE_APPROVAL | DENY |
|---|---|---|---|---|
| `k8s-deploy-guard.yaml` | `k8s.deploy.*` | runbook + ≤1 svc + business hours | runbook + many svcs or off-hours | no runbook |
| `data-deletion.yaml` | `data.delete.*`, `db.delete.*` | dry-run passed, 1 row, no PII | moderate rows or any PII row | no dry-run, or ≥100k rows |
| `external-spend.yaml` | `spend.create.*` | < $500 USD | $500 – $10k | ≥ $10k |
| `k8s-namespace-mutation.yaml` | `k8s.namespace.*` | dev namespace, 0 workloads | staging-scale workloads | `kube-system` / `kube-public` or ≥20 workloads |
| `github-merge-protection.yaml` | `github.pr.merge` | CI green, < 50 lines | CI green, 50–2000 lines | CI failing, or > 2000 lines |
| `payments-chargeback.yaml` | `payments.reverse.*`, `payments.contest.*` | ≤ $500 reversal, clean history | high-value or repeat disputes | account under investigation |
| `database-schema-migration.yaml` | `db.schema.*` | migration tested, 1 table | staging-scale tables/rows | not migration-tested, or ≥10 tables |

### Quick evaluation examples

Try these with the `dp_simulate` tool in Claude Desktop after setup (or via the SDK directly):

**k8s deploy — ALLOW** (runbook present, 2 services, business hours)
```json
{ "action": "k8s.deploy.deployment", "context": { "has_runbook": true, "affected_services": 2 } }
```

**k8s deploy — REQUIRE_APPROVAL** (large blast radius + overnight window)
```json
{ "action": "k8s.deploy.deployment", "context": { "has_runbook": true, "affected_services": 8 } }
```

**kube-system delete — DENY** (system namespace is unconditionally blocked)
```json
{ "action": "k8s.namespace.delete", "resource": { "type": "k8s.namespace", "id": "kube-system" }, "context": {} }
```

**Spend $1,200 — REQUIRE_APPROVAL** (above the $500 auto-allow cap)
```json
{ "action": "spend.create.saas", "context": { "amount": 1200, "currency": "USD" } }
```

---

## Audit trail

Every evaluation is logged to `~/.decisionplane/traces.jsonl`:

```bash
# View recent decisions
tail -5 ~/.decisionplane/traces.jsonl | jq '{trace_id, action, decision, risk_score}'

# Filter to approvals only
jq 'select(.decision == "REQUIRE_APPROVAL")' ~/.decisionplane/traces.jsonl

# Count decisions by type
jq -r '.decision' ~/.decisionplane/traces.jsonl | sort | uniq -c
```

---

## Authoring your own policies

Policies are YAML files using the [Policy DSL](https://decisionplane.io/docs/guide/policy-dsl).
Save any `.yaml` file to `~/dp-policies/` — no restart required.

The files in `policies/` are a working starting point. Each file includes inline comments explaining
the scoring math and a **Demo quick reference** section showing which context values produce ALLOW,
REQUIRE_APPROVAL, and DENY.

---

## Upgrading to hosted mode

Local mode is a zero-friction starting point for a single developer. When you're ready for
multi-user approvals, policy sync across teams, or a real-time audit dashboard, upgrade to
[decisionplane.io](https://decisionplane.io):

```json
"env": {
  "DECISIONPLANE_API_URL": "https://api.decisionplane.io",
  "DECISIONPLANE_API_KEY": "your-api-key",
  "DECISIONPLANE_ACTOR_ID": "claude-desktop"
}
```

Your policies migrate directly — the YAML format is identical in both modes.

---

## Links

- **Main repo**: [github.com/decisionplane/decisionplane](https://github.com/decisionplane/decisionplane)
- **npm package**: [@decisionplane/mcp](https://www.npmjs.com/package/@decisionplane/mcp)
- **Docs**: [decisionplane.io/docs](https://decisionplane.io/docs)
- **Policy DSL reference**: [decisionplane.io/docs/reference/policy-schema](https://decisionplane.io/docs/reference/policy-schema)

---

## License

Apache-2.0 — see [LICENSE](LICENSE).
