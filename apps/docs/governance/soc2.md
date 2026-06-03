# SOC 2 Scope

_Buyer-facing extract. Full posture in [Compliance Posture](/governance/compliance)._

## Current position

**No SOC 2 Type 1 attestation exists. No auditor is engaged.**

DecisionPlane will not characterise SOC 2 work as "in flight," "in progress," or "underway" in any sales conversation until those things are literally true.

## Intended scope (when Type 1 work begins)

| Dimension | Position |
| --- | --- |
| **In-scope systems** | The hosted daemon (when launched); the DecisionPlane build/release pipeline; the source-code repositories. |
| **Out-of-scope** | Customer infrastructure the SDK runs inside; third-party LLM/agent platforms that call the SDK; the BSL-licensed source distribution. |
| **In-scope TSCs** | Security and Confidentiality. |
| **Deferred TSCs** | Availability and Processing Integrity → Type 2. Privacy → deferred until we handle PHI/PII at meaningful volume. |
| **In-scope criteria** | CC1–CC9; C1.1–C1.2 |

## Timing commitment

| Milestone | Trigger | Target |
| --- | --- | --- |
| **Type 1 kickoff** | First paid design partner signs | **≤ 90 days from signing** |
| **Type 1 letter** | Kickoff date | **6 months from kickoff** |
| **Type 2 observation** | Type 1 letter issued | Begins immediately |

## Auditor

Not selected. Shortlist: Prescient Assurance, A-LIGN, Insight Assurance. Selection blocked on first paying customer.

## Compensating controls today

- Deny-by-default policy engine
- Append-only SHA-256 hash-chained audit log (Postgres row-level deny-update triggers)
- ULID identifiers for every primitive
- All writes through Zod-validated schema boundaries
- Two-person GitHub review on protected `main`
- TLS-only Postgres DSN enforced in code
- No production hosted system to breach

## What we tell buyers

> We do not have SOC 2. We will begin Type 1 work within 90 days of you signing as a design partner, and we will scope it to the systems your data actually touches. In the meantime, our library-first architecture means your decisioning data does not leave your environment, which is a stronger position than most SaaS vendors' Type 2 reports.
