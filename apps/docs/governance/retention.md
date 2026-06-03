# Audit-Log Retention

_Anchored in [Compliance Posture](/governance/compliance). This is the buyer-facing extract._

## Core position

The SDK keeps every audit record **forever** by default. This is intentional — an audit chain you can trim is not an audit chain.

"Forever in one SQLite file" is operationally unrealistic at scale, so the retention model is tiered. Tiering does not break the chain: every tier carries chain continuity via shard manifests.

## Tier model

| Tier | Where | Default | Purpose |
| --- | --- | --- | --- |
| **Hot** | SQLite WAL (library) or Postgres (daemon) | Indefinite | Sub-millisecond reads for `getDecision`, `verifyChain` |
| **Warm** | Customer-controlled object storage (S3 / GCS / Azure Blob). Per-day NDJSON.gz shards with manifest hashes. | Indefinite | Cheap, auditable, forensic reads |
| **Cold / Legal hold** | Customer-managed (e.g. S3 Glacier + object lock + compliance retention) | Customer-set | WORM-equivalent for regulated obligations |

## Regulatory floors

Enforced in code when the customer flags scope. The SDK refuses to delete or expire below the floor.

| Scope tag | Floor | Source |
| --- | --- | --- |
| `scope: "sox"` | **7 years** in warm or cold | Sarbanes-Oxley §802/§1102 |
| `scope: "hipaa"` | **6 years** from creation or last access | 45 CFR §164.530(j) |
| `scope: "pci"` | **1 year hot, 1 year accessible** | PCI-DSS v4.0 §10.5.1 |
| `scope: "gdpr"` | No floor. Customer-defined. | — |
| No tag | 1 year warm when warm tier enabled; otherwise hot forever | — |

## Shard format and manifest integrity

Warm-tier shards are written as gzip-compressed NDJSON files (`traces.jsonl.gz`) under date-partitioned object storage keys. Each record is a full `TraceRecord` JSON line including `prev_hash` / `hash` for cross-tier chain continuity.

Each day's shard file is accompanied by a `manifest.json` containing:
- Per-shard metadata: `record_count`, `first_hash`, `last_hash`, `shard_hash` (SHA-256 of raw bytes)
- A `manifest_hash` (HMAC-SHA256 over the canonical shard list) keyed by `WARM_MANIFEST_SECRET`
- A `chain_tip_hash` linking the last warm-tier record into any subsequent cold-tier or hot-tier chain

`WARM_MANIFEST_SECRET` is a **platform-controlled secret** (≥ 32 characters, high-entropy random) set by DecisionPlane ops at deploy time. It is not per-customer; all tenant manifests are signed by the same platform key.

### What auditors can verify without the secret

An auditor with access to a shard export can verify:
1. **Record-level hash chain** — every record's `prev_hash` → `hash` chain from genesis via `dp audit verify-chain --warm-dir <export-dir>`
2. **Per-shard byte integrity** — decompress the `.jsonl.gz` and compute SHA-256 of the raw bytes against `shard_record.shard_hash` in the manifest

What requires `WARM_MANIFEST_SECRET`:
- That the shard *list* in a manifest has not been altered (shard added or removed). For regulated-industry audits, the operator discloses `WARM_MANIFEST_SECRET` to the auditor under NDA.

A future enhancement will add asymmetric manifest signing (Ed25519 public key published) to allow public-key verification without sharing the secret.

## GDPR erasure

Audit records are never deleted in response to a right-to-erasure request. The supported pattern is **crypto-shredding**:

1. Encrypt sensitive fields at write-time with a per-subject key.
2. To honour Art. 17, destroy the key.
3. Ciphertext remains in the chain — chain integrity preserved — plaintext is unrecoverable.

## `dp audit verify-chain`

Walks the full hash chain across hot and warm tiers and reports any break.

```
# Hot tier only (SQLite export)
dp audit verify-chain --db audit.db

# Warm tier from a local directory (offline auditor use — no credentials needed)
dp audit verify-chain --warm-dir /path/to/shard/export

# Hot + warm tiers with manifest integrity check
dp audit verify-chain --db audit.db --warm-dir /path/to/shards \
  --verify-manifests --manifest-secret-env WARM_MANIFEST_SECRET

# Warm tier from S3
dp audit verify-chain --s3-bucket my-audit-shards --s3-prefix shards/

# Output
PASS  hot=0  warm=10000000  total=10000000  lastHash=<hex>  elapsed=48230ms
```

## What ships today (v1, DEC-110)

- **Hot tier** — SQLite WAL and Postgres, indefinite retention.
- **Warm tier** — Per-day NDJSON.gz shards to S3/GCS/Azure Blob via `WarmTierWriter`.
- **Cold tier** — `buildArchivalEvent` emits `decision.archived` when records move to cold storage.
- **Scope tags and regulatory floors** — enforced at application layer (`checkRetentionFloor`) and DB layer (Postgres trigger in migration 0003).
- **`dp audit verify-chain`** — hot + warm cross-tier chain verification.
- **Shard manifests** — HMAC-SHA256 integrity protection.
