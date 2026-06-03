# Governance

DecisionPlane is built for regulated environments. This section documents our license, compliance posture, audit-log retention policy, and SOC 2 scope — written to be handed to a diligence team without amendment.

## Pages

| Document | What it covers |
| --- | --- |
| [License](/governance/license) | BSL-1.1 → Apache-2.0 transition, Usage Limitation |
| [Compliance Posture](/governance/compliance) | SOC 2, HIPAA, GDPR — current-state positions, not aspirations |
| [Audit-Log Retention](/governance/retention) | Tiered hot/warm/cold model with regulatory floors |
| [SOC 2 Scope](/governance/soc2) | Intended scope and kickoff trigger |

## Quick facts

- **License:** BSL-1.1. Converts to Apache-2.0 four years after each release. Usage Limitation excludes competing decisioning-PaaS products. Source is always public.
- **Data residency:** Library mode — your data never leaves your process. Daemon mode — you host it, you own the runtime. We are not a data processor by default.
- **SOC 2:** Not in flight. Type 1 kickoff ≤ 90 days after first paid design partner. See [SOC 2 Scope →](/governance/soc2).
- **HIPAA:** We do not sign BAAs. Library mode is the recommended pattern for healthcare — the SDK never sees PHI if you run it inside your environment.
- **Audit chain:** SHA-256 hash-chained, append-only. `verifyChain()` is callable at any time by any authenticated caller.

Questions? Email [security@decisionplane.dev](mailto:security@decisionplane.dev).
