# Compliance Posture

_Current-state position as of 2026-05-22. Written to be handed to a regulated-industry buyer's diligence team without amendment. Where the honest answer is "we don't have that," we say so and name the owner who will close it._

## SOC 2

**No SOC 2 Type 1 attestation exists. No auditor is engaged.**

We will not characterise SOC 2 work as "in flight" until those things are literally true.

**Timing commitment:** Type 1 kickoff ≤ 90 days after first paid design partner signs. Type 1 letter: 6 months from kickoff.

**Compensating controls today:**
- Deny-by-default policy engine
- Append-only SHA-256 hash-chained audit log (row-level deny-update triggers on Postgres)
- All writes through Zod-validated schema boundaries
- Two-person GitHub review on protected `main`
- No production hosted system to breach

→ [Full SOC 2 scope →](/governance/soc2)

## HIPAA

**We do not sign BAAs. We do not knowingly accept PHI today.**

In library mode, DecisionPlane is not a Business Associate — the decision and its inputs never leave the customer's process. This is the recommended pattern for healthcare prospects through end-of-2026.

If you want to use the hosted daemon with PHI, we will decline until BAA-ready.

## GDPR

**We are not a data controller or processor by default.**

In library mode: you run the engine, you own the data. We never see it.

In daemon mode: you host the daemon. We do not have access to your data unless you grant it.

**Right to erasure (Art. 17):** Audit records are never deleted — the hash chain is the product. The supported pattern is crypto-shredding: encrypt sensitive fields at write-time with a per-subject key; destroy the key to make plaintext unrecoverable while preserving chain integrity.

## Encryption

**SQLite (library mode):** not encrypted at rest by default. Customer is responsible for disk-level encryption in their environment.

**Postgres (daemon mode):** TLS-only DSN enforced in code. `assertTlsOnlyDsn()` rejects any `DATABASE_URL` without `sslmode=require` or `sslmode=verify-full`. No unencrypted Postgres connections are possible in daemon mode.

## What we tell buyers today

> Our library-first architecture means your decisioning data does not leave your environment, which is a stronger confidentiality posture than most SaaS vendors' SOC 2 Type 2 reports. We will begin SOC 2 Type 1 work within 90 days of you signing as a design partner.

Questions: [security@decisionplane.dev](mailto:security@decisionplane.dev)
