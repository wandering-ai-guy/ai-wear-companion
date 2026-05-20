# Part 20 — Observability & Cost Monitoring

> Goal of this part: when something is slow or expensive, you can
> answer "why?" in under five minutes.

You can defer beautiful dashboards. You cannot defer logging,
metrics, and tracing — they're only useful if they're collected
*before* the incident.

## 1. Three pillars

| Pillar | What it tells you | Tool we use |
|--------|-------------------|-------------|
| Logs | What happened, in order, with context | Cloud Logging (auto from Cloud Run) + structlog JSON |
| Metrics | How often, how fast, how big | Cloud Monitoring + Prometheus |
| Traces | Where time was spent in a single request | Sentry Performance + LangSmith for LLM |

## 2. Structured logs (already done)

Part 03 wired structlog. Now use it.

Two rules:

- **Bind context once at the top of a request** so every line
  inside the handler has `uid`, `request_id`, `route`.
- **Log durations**, not just events. `log.info("processed", ms=int(t2-t1))`.

A small middleware:

```python
# in backend/utils/middleware.py
import time
import uuid

import structlog
from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware


class RequestContextMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        rid = request.headers.get("x-request-id", str(uuid.uuid4()))
        structlog.contextvars.clear_contextvars()
        structlog.contextvars.bind_contextvars(
            request_id=rid,
            method=request.method,
            path=request.url.path,
        )
        t0 = time.time()
        try:
            response = await call_next(request)
            dur_ms = int((time.time() - t0) * 1000)
            structlog.get_logger("http").info(
                "request_completed", status=response.status_code, duration_ms=dur_ms
            )
            response.headers["x-request-id"] = rid
            return response
        except Exception:
            dur_ms = int((time.time() - t0) * 1000)
            structlog.get_logger("http").exception("request_failed", duration_ms=dur_ms)
            raise
```

Wire in `main.py`:

```python
from utils.middleware import RequestContextMiddleware
app.add_middleware(RequestContextMiddleware)
```

Now every JSON log line has `request_id`, `uid` (set by your auth
dep), method, path, duration. In Cloud Logging you can filter
`jsonPayload.uid="..." AND jsonPayload.duration_ms>500`.

## 3. Prometheus metrics

Append to `requirements.txt`:

```
prometheus-client==0.21.1
prometheus-fastapi-instrumentator==7.0.0
```

```python
# in backend/main.py — after app = FastAPI(...)
from prometheus_fastapi_instrumentator import Instrumentator
Instrumentator().instrument(app).expose(app, endpoint="/internal/metrics")
```

Cloud Run automatically exports CPU/memory/request count to Cloud
Monitoring. The `/internal/metrics` endpoint adds *application-level*
counters (per-route latency histograms, error rates).

Custom metrics that have actually mattered:

```python
from prometheus_client import Counter, Histogram

LISTEN_BYTES = Counter("listen_bytes_total", "Bytes streamed to /v1/listen", ["codec"])
DG_LATENCY = Histogram("deepgram_segment_latency_ms", "Time from chunk to final segment")
LLM_TOKENS = Counter("llm_tokens_total", "Tokens used", ["model", "kind"])  # kind=in|out
LLM_COST_USD = Counter("llm_cost_usd_total", "Estimated $ spent", ["model"])
```

Increment them at the right places:

```python
LISTEN_BYTES.labels(codec=fmt.codec).inc(len(chunk))
LLM_TOKENS.labels(model="gemini-2.5-flash", kind="in").inc(rsp.usage_metadata.prompt_token_count)
LLM_TOKENS.labels(model="gemini-2.5-flash", kind="out").inc(rsp.usage_metadata.candidates_token_count)
```

## 4. Sentry: errors and a crude trace

Part 03 already inits Sentry. Two more things:

- Add the FastAPI integration so unhandled exceptions in routes
  surface automatically:

```python
sentry_sdk.init(
    dsn=settings.sentry_dsn,
    environment=settings.env,
    traces_sample_rate=0.05,
    profiles_sample_rate=0.05,
    integrations=[],  # FastAPI integration is auto-detected
)
```

- Tag spans for slow paths:

```python
with sentry_sdk.start_span(op="llm.post_process", description=cid):
    await post_process_conversation(uid, cid)
```

Set up Slack notifications for `error.rate > 1% over 5min` and `p95
latency > 1500ms`.

## 5. LangSmith: trace your LLM calls

Append to `.env`:

```
LANGSMITH_TRACING=true
LANGSMITH_API_KEY=...
LANGSMITH_PROJECT=<<YOUR_BRAND>>-prod
```

Append to `requirements.txt`:

```
langsmith==0.4.37
```

The Gemini Python SDK does **not** auto-export to LangSmith. The
simplest wiring is a small wrapper around `gemini_client()` that
opens a LangSmith run before each call:

```python
# in backend/utils/llm/tracing.py
import os
from contextlib import asynccontextmanager
from typing import Any

from langsmith import traceable


@traceable(run_type="llm", name="gemini.generate_content")
async def traced_generate(model: str, contents: Any, config: Any, client):
    return await client.aio.models.generate_content(
        model=model, contents=contents, config=config,
    )
```

Then in `post_process.py` / `orchestrator.py`, call
`traced_generate(...)` instead of the raw client call. You'll see
every Gemini call in LangSmith with prompt + response + latency.

Disable in dev unless you have a reason — your dev prompts often
contain test PII you don't want in a third-party UI.

## 6. Cost monitoring

Three cost lenses:

1. **Per-user cost.** Add a Firestore subcollection
   `users/{uid}/usage/{yyyy-mm}` with running counters for STT minutes,
   LLM tokens, embedding tokens. Increment them next to the
   Prometheus counters above. Surface them on a /v1/users/me/usage
   endpoint and in your support tooling.
2. **Per-feature cost.** Tag every LLM call with a `feature` label
   (`post_process`, `chat`, `memory_extract`). When you fix a slow
   prompt, track the percent reduction.
3. **Cloud bills.** GCP exports billing to BigQuery; build a simple
   weekly digest. (`bq query "SELECT service.description, SUM(cost) … GROUP BY 1 ORDER BY 2 DESC LIMIT 10"`).

Set up Slack alerts for spikes:

- Daily LLM spend > 1.5× the trailing 7-day average.
- Daily STT minutes > 2× trailing 7-day.
- Hourly Pinecone reads > 100K.

## 7. Per-user fair-use limits

Some users will record 24/7 and burn your margins. Soft-cap them:

```python
# in backend/utils/fair_use.py
import time
from database import redis_db

LIMITS = {"free": 4 * 60, "pro": 12 * 60}  # minutes/day


def can_listen(uid: str, tier: str) -> bool:
    today = time.strftime("%Y%m%d")
    key = f"fair_use:{uid}:{today}"
    used = int((redis_db.get(key) or b"0").decode())
    return used < LIMITS.get(tier, LIMITS["free"])


def record_minutes(uid: str, minutes: int) -> None:
    today = time.strftime("%Y%m%d")
    c = redis_db._get_client()
    if not c:
        return
    c.incrby(f"fair_use:{uid}:{today}", int(minutes))
    c.expire(f"fair_use:{uid}:{today}", 60 * 60 * 36)
```

Check `can_listen(uid, user.subscription_tier)` at the top of
`/v1/listen`; reject with code 1008 if exceeded.

## 8. SLOs & on-call

Define three SLOs for v1:

- **Listen WS availability:** 99.5% of `/v1/listen` connection
  attempts succeed.
- **Transcript latency:** p95 first-final-token ≤ 1.5 s after first
  audio frame.
- **Conversation post-processing:** p95 finalize-to-completed ≤ 8 s.

Set a "page me" rule (PagerDuty/Opsgenie/Slack) when any SLO
violates by 2× for >15 min.

## 9. Auditing

Keep an audit log of *who* did *what* on every privacy-relevant
action — Part 17 set up the table. Mirror it to BigQuery monthly so
you can answer compliance questions.

## 10. A concrete dashboard

Make one Cloud Monitoring dashboard with these panels (you'll spend
your life looking at it):

- Requests per second by route.
- p95 latency by route.
- Error rate by route.
- Active WebSocket connections.
- LLM tokens/minute by model.
- LLM cost (USD)/minute.
- STT minutes/minute.
- Pinecone reads & writes/minute.
- Pusher backlog (size of Pub/Sub channel).
- Cloud Run instance count.

## 11. Commit

```bash
git add backend
git commit -m "feat(part-20): structured logs, prom metrics, sentry, langsmith, fair-use"
git push
```

## What you should have right now

- [ ] Every log line is JSON with `request_id` + `uid` + duration.
- [ ] `/internal/metrics` exposes Prometheus metrics.
- [ ] Sentry catches unhandled exceptions; Slack alerts fire.
- [ ] LangSmith shows every LLM call (production-only).
- [ ] Per-user usage counters in Firestore.
- [ ] Fair-use cap blocks abusive listen sessions.
- [ ] One main monitoring dashboard.

---

Next: [Part 21 — Mobile App Wiring & Branding](./21-mobile-app.md).
