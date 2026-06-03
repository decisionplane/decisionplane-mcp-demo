# Install the SDK

DecisionPlane runs **library-first** — the engine, policy evaluator, and audit log run in-process against a local SQLite database. No daemon, no Postgres, no network hop in the decision path.

## Prerequisites

- Node.js 22 LTS or later
- pnpm (or npm / yarn)

## Option A — Clone and run locally (5 minutes)

The fastest path to a running system:

```bash
git clone https://github.com/decisionplane/decisionplane
cd decisionplane
pnpm install

# Terminal 1 — start the daemon (auto-seeds a dev token)
pnpm dev

# Terminal 2 — run the quickstart
pnpm quickstart
```

`pnpm dev` starts Fastify on port 3000, loads policies from `./policies/`, creates `dev-tokens.db`, writes the token to `.dev-api-key`, and prints it to the console.

`pnpm quickstart` reads the token and runs four steps:

1. Submits a low-risk refund (`amount=50`) → `approve`
2. Submits an over-threshold refund (`amount=750`) → `deny`
3. Applies an override to the denied record
4. Calls `GET /v1/audit/chain` and asserts `ok === true`

Expected output:

```
✓ low-risk refund approved (id: 01HX...)
✓ high-risk refund denied  (id: 01HX...)
✓ override applied         (id: 01HX...)
✓ audit chain verified     length=3  head=abc123...
```

## Option B — Add the SDK to your project

```bash
npm install @decisionplane/sdk
# or
pnpm add @decisionplane/sdk
```

The SDK re-exports the stable `DecisionRequest` and `DecisionRecord` types so your code depends on the public API, not internal engine types.

Continue to: [Your First Decision →](/guide/first-decision)
