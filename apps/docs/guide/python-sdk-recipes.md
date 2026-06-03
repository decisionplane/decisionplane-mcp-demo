# Python SDK Recipes

Practical integration patterns for the `decisionplane` Python SDK.

---

## LangChain tool wrapping

Gate a LangChain tool so policy is evaluated before the tool body runs:

```python
from langchain.tools import tool
from decisionplane import guarded, DecisionDenied

dp = DecisionPlane.from_env()

@tool
@guarded(action="langchain.tool.execute", client=dp)
def search_internal_docs(query: str, *, dp_context=None) -> str:
    """Search internal documentation."""
    return retriever.invoke(query)

# In an agent chain:
try:
    result = search_internal_docs.invoke(
        {"query": "payment service runbook"},
        dp_context={"session_id": "sess-42"},
    )
except DecisionDenied as e:
    result = f"[Access denied: {e.reasoning}]"
```

> The decorators must be applied inside-out: `@guarded` wraps the function first, `@tool` wraps the guarded version. LangChain sees a normal callable.

---

## FastAPI middleware

Check policy before every request reaches your route handler using a middleware:

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from decisionplane import AsyncDecisionPlane, DecisionDenied

app = FastAPI()
dp = AsyncDecisionPlane.from_env()

@app.middleware("http")
async def policy_gate(request: Request, call_next):
    action = f"http.{request.method.lower()}.{request.url.path.strip('/').replace('/', '.')}"
    try:
        await dp.evaluate(
            action=action,
            resource={"path": request.url.path},
            context={"client_ip": request.client.host if request.client else "unknown"},
        )
    except DecisionDenied as e:
        return JSONResponse(
            status_code=403,
            content={"error": "forbidden", "reason": e.reasoning, "trace_id": e.trace_id},
        )
    return await call_next(request)

@app.get("/admin/purge-cache")
async def purge_cache():
    return {"status": "purged"}
```

---

## Celery task wrapping

Evaluate policy before a Celery task body executes:

```python
from celery import Celery
from decisionplane import guarded, DecisionDenied

app = Celery("myapp")
dp = DecisionPlane.from_env()

@app.task
@guarded(action="celery.task.run", client=dp)
def process_payment(order_id: str, amount: float, *, dp_context=None):
    """Process a payment — guarded by DecisionPlane."""
    return payment_gateway.charge(order_id, amount)

# Dispatch with context:
process_payment.delay("order-8821", 99.99)
# To pass dp_context from the caller:
process_payment.apply_async(
    args=["order-8821", 99.99],
    kwargs={"dp_context": {"tenant": "acme-corp"}},
)
```

> For async Celery workers (e.g. Celery with gevent/eventlet) use `async_guarded` + `AsyncDecisionPlane`.

---

## Plain script

```python
#!/usr/bin/env python3
"""Deploy script gated by DecisionPlane."""
import sys
from decisionplane import DecisionPlane, DecisionDenied, ApprovalTimeout

def main() -> int:
    service = sys.argv[1]
    version = sys.argv[2]

    dp = DecisionPlane.from_env()
    try:
        result = dp.evaluate(
            action="k8s.deploy.service",
            resource={"service": service, "version": version},
            context={"triggered_by": "ci-pipeline"},
            reasoning=f"Automated deploy of {service}:{version}",
        )
    except DecisionDenied as e:
        print(f"Deploy denied: {e.reasoning}", file=sys.stderr)
        return 1

    if result.decision == "REQUIRE_APPROVAL":
        print(f"Approval required. Open: {result.approval_url}")
        print("Waiting up to 10 minutes for human approval…")
        try:
            approval = dp.await_approval(result.approval_token, timeout=600)
            print(f"Approved by {approval.resolved_by}")
        except ApprovalTimeout:
            print("Approval timed out — aborting deploy.", file=sys.stderr)
            return 1

    print(f"Deploying {service}:{version} (trace: {result.trace_id})")
    # … actual deploy logic …
    return 0

if __name__ == "__main__":
    sys.exit(main())
```

---

## Migration from a hand-rolled HTTP client

If you were previously calling `/v1/evaluate` directly, replace this pattern:

```python
# Before
import httpx, os, uuid

resp = httpx.post(
    f"{os.environ['DP_URL']}/v1/evaluate",
    headers={"Authorization": f"Bearer {os.environ['DP_KEY']}"},
    json={
        "actor": {"kind": "agent", "principal": {"id": "my-service"}},
        "action": "k8s.deploy.service",
        "resource": {"type": "service", "id": "payments-api"},
        "context": {},
    },
).raise_for_status().json()

if resp["decision"] == "DENY":
    raise RuntimeError(resp["reasoning"])
```

With this:

```python
# After
from decisionplane import DecisionPlane, DecisionDenied

dp = DecisionPlane.from_env()
result = dp.evaluate(
    action="k8s.deploy.service",
    resource={"service": "payments-api"},
)
# DecisionDenied raised automatically on DENY — no manual check needed
```

Key differences:

| Old (hand-rolled) | New (SDK) |
|---|---|
| Manual actor construction | Actor built from env vars / constructor |
| Manual DENY check | `DecisionDenied` raised automatically |
| Raw `approval_token` polling | `await_approval()` handles long-poll loop |
| No retry / network resilience | SDK uses `DECISIONPLANE_FAIL_MODE` |
| No trace correlation | `current_trace_id` context var |
