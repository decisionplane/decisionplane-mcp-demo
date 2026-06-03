# Python SDK

The `decisionplane` Python package gives non-MCP Python agents — LangChain tools, custom orchestrators, FastAPI services, Celery tasks — direct access to the policy engine.

## Installation

```bash
pip install decisionplane
```

**Requires Python 3.10+. Runtime dependencies: `httpx`, `pydantic v2`.**

---

## Configuration

Set three environment variables:

```bash
export DECISIONPLANE_API_URL="http://localhost:3000"
export DECISIONPLANE_API_KEY="$(cat .dev-api-key)"   # dev mode
export DECISIONPLANE_ENVIRONMENT="production"         # optional, default
```

Or configure programmatically:

```python
from decisionplane import DecisionPlane

dp = DecisionPlane(
    api_url="https://dp.internal",
    api_key="sk-...",
    environment="staging",
    actor_id="my-service",       # optional, default "python-sdk-agent"
)
```

---

## Evaluating an action

```python
from decisionplane import DecisionPlane, DecisionDenied, ApprovalTimeout

dp = DecisionPlane.from_env()

try:
    result = dp.evaluate(
        action="k8s.deploy.service",
        resource={"service": "payments-api", "version": "v2.1.3"},
        context={"incident_id": "INC-4521"},
        reasoning="Hotfix for P0 bug",
    )
except DecisionDenied as e:
    print(f"Blocked: {e.reasoning}")
    raise

# result.decision  →  "ALLOW" | "REQUIRE_APPROVAL"
# DENY always raises DecisionDenied before returning.

if result.decision == "REQUIRE_APPROVAL":
    try:
        approval = dp.await_approval(result.approval_token, timeout=300)
        print(f"Approved by {approval.resolved_by}")
    except ApprovalTimeout:
        print("Nobody approved within 5 minutes — aborting")
        raise
```

### Resource mapping

The `resource` dict maps to the API's `{type, id}` shape:

- If the dict has `type` and `id` keys, they are used verbatim.
- Otherwise the first key becomes the resource type and its value the ID.

```python
# Explicit
dp.evaluate(action="...", resource={"type": "bucket", "id": "reports-eu"})

# Shorthand — becomes {type: "service", id: "payments-api"}
dp.evaluate(action="...", resource={"service": "payments-api", "version": "v2.1.3"})
```

---

## Async client

All methods have async equivalents on `AsyncDecisionPlane`:

```python
import asyncio
from decisionplane import AsyncDecisionPlane, DecisionDenied

async def main() -> None:
    async with AsyncDecisionPlane.from_env() as dp:
        result = await dp.evaluate(
            action="k8s.deploy.service",
            resource={"service": "payments-api"},
        )
        if result.decision == "REQUIRE_APPROVAL":
            await dp.await_approval(result.approval_token, timeout=300)
        print(f"Decision: {result.decision} (trace: {result.trace_id})")

asyncio.run(main())
```

---

## `@guarded` decorator

Gate any function with a one-liner:

```python
from decisionplane import guarded, async_guarded, DecisionDenied, ApprovalTimeout

@guarded(action="k8s.deploy.service", policy_hint="deployment-safety")
def deploy_to_production(service: str, version: str, *, dp_context=None):
    """Runs only when DecisionPlane ALLOWs (or approval resolves)."""
    return k8s.deploy(service, version)

# Caller passes context via dp_context:
deploy_to_production("payments-api", "v2.1.3", dp_context={"incident_id": "INC-4521"})
```

The decorator:

1. Binds the function arguments to build the resource dict automatically.
2. Calls `evaluate()` before the function body executes.
3. On **ALLOW** — runs the function and stores `trace_id` in a `contextvars.ContextVar` for nested correlation.
4. On **REQUIRE_APPROVAL** — blocks until the approval resolves; raises `ApprovalTimeout` if it expires.
5. On **DENY** — raises `DecisionDenied`; the function body is never reached.

Async variant for `async def` functions:

```python
@async_guarded(action="k8s.deploy.service")
async def deploy_to_production(service: str, version: str, *, dp_context=None):
    return await k8s.async_deploy(service, version)
```

### Supplying a client to the decorator

By default the decorator calls `DecisionPlane.from_env()` on first use. Pass a pre-configured client:

```python
dp = DecisionPlane(api_url="...", api_key="...", environment="staging")

@guarded(action="k8s.deploy.service", client=dp)
def deploy(service: str, *, dp_context=None): ...
```

---

## Error reference

| Exception | When raised |
|---|---|
| `DecisionDenied` | Policy returned DENY; `.reasoning` and `.trace_id` are set |
| `ApprovalTimeout` | REQUIRE_APPROVAL await exceeded timeout or the approval expired |
| `DecisionPlaneUnreachable` | Network error; respects `DECISIONPLANE_FAIL_MODE` |
| `DecisionPlaneHTTPError` | Unexpected 4xx/5xx response; `.status_code` and `.body` are set |
| `DecisionPlaneError` | Base class for all SDK errors |

### Fail-open mode

By default, if the engine is unreachable the SDK raises `DecisionPlaneUnreachable` (closed / deny-safe). Set `DECISIONPLANE_FAIL_MODE=open` to allow calls through when the engine is down:

```bash
export DECISIONPLANE_FAIL_MODE=open
```

Or pass `fail_open=True` to the constructor.

---

## Nested trace correlation

When a guarded call succeeds, the trace ID is stored in `current_trace_id` for the duration of the function body:

```python
from decisionplane import current_trace_id, guarded

@guarded(action="outer.action", client=dp)
def outer_operation(x: str, *, dp_context=None) -> str:
    trace = current_trace_id.get()   # correlate with nested calls
    return inner_operation(x)
```

---

## Next steps

- [Recipes](./python-sdk-recipes.md) — LangChain, FastAPI, Celery, plain scripts, migration guide
- [REST API reference](../reference/rest-api.md) — raw HTTP surface
- [Policy DSL](./policy-dsl.md) — write policies that the SDK evaluates against
