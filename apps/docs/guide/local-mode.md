# MCP Local Mode

Run DecisionPlane governance for AI agents with zero infrastructure — no account, no daemon, no network. Clone a repo, run one command, and your agent is gated by real policies in under 30 seconds.

## How it works

When `DECISIONPLANE_MODE=local` is set, the MCP server:

- **Loads policies** from `~/dp-policies/` (all `.yaml` files, hot-loaded on every evaluation)
- **Evaluates in-process** — no HTTP calls, no daemon required
- **Writes traces** to `~/.decisionplane/traces.jsonl` (append-only, persists across restarts)
- **Stores approvals** in `~/.decisionplane/approvals.jsonl`

## Quick start (30 seconds)

```bash
# 1. Install
npm install -g @decisionplane/mcp

# 2. Copy demo policies
git clone https://github.com/decisionplane/decisionplane-mcp-demo
mkdir -p ~/dp-policies
cp decisionplane-mcp-demo/policies/*.yaml ~/dp-policies/
```

Then add DecisionPlane to your Claude Desktop config (`~/Library/Application Support/Claude/claude_desktop_config.json` on macOS):

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

## Claude Code setup

Add to `.claude/settings.json` in your project (or `~/.claude/settings.json` globally):

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

## Decision flow

Every time the agent calls `dp_evaluate`, the engine checks the action against all policies in your directory. The most restrictive match wins:

| Decision | What happens |
|---|---|
| `ALLOW` | Action proceeds immediately |
| `REQUIRE_APPROVAL` | Agent pauses and calls `dp_await_approval`. Run `dp-mcp approve` in a terminal to resolve. |
| `DENY` | Agent cannot take the action |

## CLI approval flow

When a policy triggers `REQUIRE_APPROVAL`, the agent is paused waiting for a human decision. Run the approve command in any terminal:

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

One keystroke resolves the approval. The agent's `dp_await_approval` call returns immediately with the decision.

## Authoring policies

Policies are YAML files using the [Policy DSL](/guide/policy-dsl). Save any `.yaml` file to `~/dp-policies/` — no restart required.

Quick example — require approval for all production Kubernetes deployments:

```yaml
policy: k8s-deploy-guard
version: 1
applies_to:
  actions:
    - k8s.deploy.*

risk_factors:
  - source: environment_sensitivity
    weight: 1.0
    inputs:
      field: context.environment
      production_values: [prod, production]

thresholds:
  allow_below: 0.30
  approval_below: 0.90
  deny_above: 0.95

approval:
  required_roles: [operator]
  timeout_ms: 3600000
```

See the [Policy Schema reference](/reference/policy-schema) for the full DSL.

## Environment variables

| Variable | Default | Description |
|---|---|---|
| `DECISIONPLANE_MODE` | `http` | Set to `local` to enable local mode |
| `DECISIONPLANE_ACTOR_ID` | `local-agent` | Stable identifier for this agent instance |
| `DECISIONPLANE_ACTOR_ROLE` | `agent` | Role assigned to this actor |
| `DECISIONPLANE_POLICY_DIR` | `~/dp-policies/` | Directory containing `.yaml` policy files |
| `DECISIONPLANE_TRACE_FILE` | `~/.decisionplane/traces.jsonl` | Append-only trace log |
| `DECISIONPLANE_APPROVAL_STORE` | `~/.decisionplane/approvals.jsonl` | Approval persistence file |

## Audit trail

All evaluations are logged to `~/.decisionplane/traces.jsonl`:

```bash
# View recent decisions
tail -5 ~/.decisionplane/traces.jsonl | jq '{trace_id, action, decision, risk_score}'

# Filter to approvals only
jq 'select(.decision == "REQUIRE_APPROVAL")' ~/.decisionplane/traces.jsonl

# Count decisions by type
jq -r '.decision' ~/.decisionplane/traces.jsonl | sort | uniq -c
```

Traces persist across MCP server restarts and Claude Desktop restarts.

## Upgrading to hosted mode

Local mode is a zero-friction starting point. When you're ready for multi-user approvals, policy sync across teams, or a real-time audit dashboard, upgrade to hosted mode:

1. Sign up at [decisionplane.io](https://decisionplane.io)
2. Replace the local env vars with your API credentials:

```json
"env": {
  "DECISIONPLANE_API_URL": "https://api.decisionplane.io",
  "DECISIONPLANE_API_KEY": "your-api-key",
  "DECISIONPLANE_ACTOR_ID": "claude-desktop"
}
```

Your policies migrate directly — the YAML format is identical in both modes.

## Security considerations

Local mode is designed for development and individual use. Key differences from hosted mode:

- **Single-user approvals** — the `dp-mcp approve` CLI is the only approval path; no Slack, web UI, or multi-reviewer flow
- **No policy versioning** — policies are read from disk; there's no git-backed change audit
- **Flat file storage** — traces and approvals are local JSONL files, not a queryable database

These are acceptable tradeoffs for a developer-first demo. For production use, hosted mode provides the full governance stack.
