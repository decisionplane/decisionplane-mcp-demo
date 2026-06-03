# Inspect the Audit Chain

Every decision — including overrides — is appended to an SHA-256 hash chain. Each record's `hash` is derived from its own content plus the previous record's `hash`, so any tampering is detectable by anyone with read access.

## Verify chain integrity

```typescript
const result = await client.verifyChain()

console.log(result.ok)        // true = no tampering detected
console.log(result.length)    // number of records in the chain
console.log(result.headHash)  // SHA-256 of the most recent record
console.log(result.verifiedAt)// ISO timestamp of the verification run

if (!result.ok) {
  console.error(`Chain broken at record #${result.brokenAt}: ${result.message}`)
}
```

`verifyChain()` calls `GET /v1/audit/chain` on the daemon, which re-computes the hash chain across every record in the audit store and returns the first broken link if any exists.

## What the chain proves

| Property | How it's enforced |
| --- | --- |
| **Append-only** | All writes go through the engine — no direct DB mutations. |
| **Tamper-evidence** | Each `hash` covers the prior `prevHash` + record content. |
| **Override visibility** | Overrides are separate records referencing `requestId`, not edits to the original. |
| **Chain continuity** | `verifyChain()` reads all records in insert order and recomputes. |

## Reading individual records

```typescript
// Fetch a single decision by its record ID
const record = await client.getDecision(recordId)

if (record) {
  console.log(record.outcome)
  console.log(record.reason)
  console.log(record.policyVersion)
  console.log(record.prevHash)  // links to the previous record
  console.log(record.hash)      // this record's position in the chain
}
```

## CLI shortcut

```bash
# From the repo root, after pnpm dev is running
curl -s -H "Authorization: Bearer $(cat .dev-api-key)" \
  http://localhost:3000/v1/audit/chain | jq .
```

Expected output:

```json
{
  "ok": true,
  "length": 3,
  "headHash": "abc123...",
  "verifiedAt": "2026-05-23T12:00:00.000Z"
}
```

---

## HMAC keying for production deployments

By default the chain uses plain SHA-256, which is sufficient for development and internal audit. For production deployments that handle personal data, you can bind a secret HMAC-SHA-256 key to each org so that chain hashes are pseudonymised and cannot be cross-correlated without the key.

### Setting the key

Generate a cryptographically secure 32-byte key and encode it as 64 lowercase hex characters:

```bash
openssl rand -hex 32
```

Then apply it to your org:

```bash
curl -X PATCH https://your-host/v1/orgs/<orgId>/settings \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"chainHmacKeyHex": "<64-char hex>"}'
```

Or use the **Org Admin → Generate** button in the UI, which generates the key client-side using `crypto.getRandomValues()` and opens a copy-once modal before saving.

### Security properties

- The key is write-only: `GET /v1/orgs/:orgId/settings` never returns the value, only `chainHmacKeyHexConfigured: true | false`.
- If the key is lost, existing chain records cannot be re-verified — back it up externally (e.g. a secrets manager).
- Satisfies EDPB pseudonymisation requirements (Art. 8.3(c)) for audit records containing personal data.

---

You have now completed the tutorial path. The three questions the chain answers:

1. **What was decided?** — `DecisionRecord.outcome` + `reason`
2. **Why?** — `policyVersion` + input snapshot in `request`
3. **By whose authority?** — `actor` in the request, `overriddenBy` if overridden
4. **What if wrong?** — override record appended, original intact, chain verifiable

→ Dive deeper: [The Four Pillars →](/guide/four-pillars)
