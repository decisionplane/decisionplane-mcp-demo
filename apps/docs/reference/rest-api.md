# REST API Reference (Daemon Mode)

The daemon exposes the same decision engine over HTTP. All endpoints are under `/v1/`. All requests require `Authorization: Bearer <token>`.

## Authentication

```bash
# Dev token (auto-seeded by pnpm dev)
export TOKEN=$(cat .dev-api-key)
curl -H "Authorization: Bearer $TOKEN" http://localhost:3000/v1/healthz
```

## Endpoints

### GET /v1/healthz

Liveness probe.

```json
{ "ok": true, "policy_version": "2026-05-23T12:00:00.000Z" }
```

### POST /v1/decisions

Submit a decision request.

**Headers:** `Idempotency-Key: <uuid>` (required)

**Body:** `DecisionRequest`

**Response:** `DecisionRecord`

```bash
curl -X POST http://localhost:3000/v1/decisions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "action": "refund.process",
    "actor": { "kind": "agent", "principal": { "id": "bot-v2", "roles": ["support"] } },
    "resource": { "type": "refund", "id": "order-8821", "attributes": { "amount": 50 } },
    "context": {},
    "requestedAt": "2026-05-23T12:00:00.000Z"
  }'
```

### GET /v1/decisions/:recordId

Fetch a recorded decision by ID.

**Response:** `DecisionRecord | null`

### POST /v1/decisions/:recordId/overrides

Apply an override. Appends a new record; never mutates the original.

**Headers:** `Idempotency-Key: <uuid>` (required)

**Body:**
```json
{
  "outcome": "approve",
  "reason": "Supervisor manual review.",
  "overriddenBy": "alice@example.com"
}
```

**Response:** `DecisionRecord` (the new override record)

### GET /v1/audit/chain

Verify the append-only hash chain across all records.

**Response:**
```json
{
  "ok": true,
  "length": 42,
  "headHash": "abc123...",
  "verifiedAt": "2026-05-23T12:00:00.000Z"
}
```

### GET /v1/policies

List loaded policies.

**Query params:** `agentKind=<string>` (optional filter)

**Response:**
```json
{
  "policy_version": "2026-05-23T12:00:00.000Z",
  "policies": [
    { "id": "refund-safety", "version": 3, "description": "..." }
  ]
}
```

### POST /v1/evaluate

Simulate a decision without writing to the audit store.

**Body:** `DecisionRequest`

**Response:** `DecisionRecord` (not persisted)

## Error responses

```json
{ "error": "Unauthorized", "statusCode": 401 }
{ "error": "Not Found", "statusCode": 404 }
{ "error": "Validation error: ...", "statusCode": 400 }
```
