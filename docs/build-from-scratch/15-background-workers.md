# Part 15 — Background Workers & Cron

> Goal of this part: a `pusher` service that handles long-running and
> bursty work without blocking the API, plus scheduled jobs.

## 1. Why a separate service?

Three reasons:

1. **Audio batching to GCS.** We want a 60-second batch of WAV audio
   uploaded per conversation for replay/debug. Doing it inline in the
   API is wasteful.
2. **Webhook fan-out.** When a conversation completes, we ping every
   "app" (Part 16+ plan, optional v1) the user installed. That's many
   HTTP calls.
3. **Failure isolation.** A misbehaving webhook should not slow down
   the next user's transcript.

The `pusher` is essentially "the same Python codebase, deployed as a
worker process that listens on Redis Pub/Sub or its own
WebSocket from the API."

## 2. Communication pattern: Redis Pub/Sub

Simplest possible queue. The API publishes events; the pusher
subscribes.

```python
# in backend/database/redis_pubsub.py
import json
import logging
from typing import Iterable, Iterator

from database import redis_db

log = logging.getLogger(__name__)
CHANNEL_PREFIX = "<<YOUR_BRAND>>:events:"


def publish(event_type: str, payload: dict) -> None:
    c = redis_db._get_client()
    if not c:
        return
    try:
        c.publish(f"{CHANNEL_PREFIX}{event_type}", json.dumps(payload))
    except Exception:
        log.warning("publish failed", exc_info=True)


def subscribe(event_types: Iterable[str]) -> Iterator[tuple[str, dict]]:
    c = redis_db._get_client()
    if not c:
        return
    p = c.pubsub()
    p.subscribe(*[f"{CHANNEL_PREFIX}{e}" for e in event_types])
    for msg in p.listen():
        if msg.get("type") != "message":
            continue
        ch = msg["channel"].decode().removeprefix(CHANNEL_PREFIX)
        try:
            yield ch, json.loads(msg["data"])
        except Exception:
            continue
```

When a conversation finalizes, the API does:

```python
from database.redis_pubsub import publish

publish("conversation.completed", {"uid": uid, "conversation_id": cid})
```

## 3. The pusher entry point

```python
# in pusher/main.py
import asyncio
import logging
import os
import signal

from database.redis_pubsub import subscribe
from utils.executors import critical_executor
from pusher.handlers import handle_conversation_completed, handle_audio_batch

logging.basicConfig(level=logging.INFO)
log = logging.getLogger(__name__)


async def main():
    log.info("pusher.starting")
    loop = asyncio.get_running_loop()

    def _shutdown(*_):
        log.info("pusher.shutting_down")
        loop.stop()

    for sig in (signal.SIGINT, signal.SIGTERM):
        loop.add_signal_handler(sig, _shutdown)

    # subscribe runs sync; offload it
    def reader():
        for kind, payload in subscribe([
            "conversation.completed",
            "audio.batch.flush",
        ]):
            asyncio.run_coroutine_threadsafe(_dispatch(kind, payload), loop)

    await loop.run_in_executor(critical_executor, reader)


async def _dispatch(kind: str, payload: dict):
    try:
        if kind == "conversation.completed":
            await handle_conversation_completed(payload)
        elif kind == "audio.batch.flush":
            await handle_audio_batch(payload)
    except Exception:
        log.exception("pusher.handler.error kind=%s", kind)


if __name__ == "__main__":
    asyncio.run(main())
```

## 4. Concrete handlers

```python
# in pusher/handlers.py
import logging

from utils.http_client import get_webhook_client, get_semaphore
from utils.storage import upload_bytes

log = logging.getLogger(__name__)


async def handle_conversation_completed(p: dict) -> None:
    uid, cid = p["uid"], p["conversation_id"]
    # 1) (optional) ping installed apps' webhooks
    # 2) (optional) trigger PDF/email export
    log.info("conversation.completed uid=%s cid=%s", uid, cid)
    # placeholder for fan-out


async def handle_audio_batch(p: dict) -> None:
    uid, cid = p["uid"], p["conversation_id"]
    audio_bytes = bytes.fromhex(p["audio_hex"])  # hex-encoded raw audio
    bucket = "<<YOUR_BRAND>>-prod-audio"
    path = f"{uid}/{cid}/segment_{p['ts']}.opus"
    upload_bytes(bucket, path, audio_bytes, content_type="audio/opus")
    log.info("audio.uploaded gs://%s/%s", bucket, path)
```

## 5. Calling pusher from the listen WS

In `routers/listen.py`, every 60 seconds during the WS, dump the
buffered audio:

```python
import time
from database.redis_pubsub import publish

audio_60s = bytearray()

# every time we receive bytes
audio_60s += chunk
if len(audio_60s) >= fmt.bytes_per_second * 60:
    publish("audio.batch.flush", {
        "uid": uid,
        "conversation_id": conversation_id,
        "ts": int(time.time()),
        "audio_hex": bytes(audio_60s).hex(),
    })
    audio_60s.clear()
```

Hex encoding over Pub/Sub is wasteful for big payloads (audio is
binary). For real production, upload directly from the API and only
publish a *pointer*:

```python
publish("audio.batch.flush", {
    "uid": uid, "conversation_id": cid, "gs_path": "gs://.../segment.opus",
})
```

For now the hex approach gives you something working in 30 lines.

## 6. Cron jobs

Two patterns:

### A) Cloud Scheduler hitting an HTTP endpoint

For *daily* reports, *weekly* recaps, etc. Add an admin-protected
endpoint:

```python
# in backend/routers/jobs.py
from fastapi import APIRouter, Depends, Header, HTTPException

from utils.settings import get_settings

router = APIRouter(prefix="/v1/jobs", tags=["jobs"], include_in_schema=False)


def require_cron_token(x_cron_token: str = Header(default="")):
    expected = __import__("os").environ.get("CRON_SECRET", "")
    if not expected or x_cron_token != expected:
        raise HTTPException(403)


@router.post("/daily-summaries", dependencies=[Depends(require_cron_token)])
def daily_summaries():
    # iterate users, generate yesterday's summary, push notification
    ...
    return {"ok": True}
```

Then in GCP, Cloud Scheduler → Create Job → HTTPS POST →
`https://api.<<YOUR_DOMAIN>>/v1/jobs/daily-summaries` with header
`X-Cron-Token: <CRON_SECRET>` at `0 8 * * *`.

### B) An always-running worker loop

For per-minute / per-second tasks: a small async loop in `pusher`:

```python
# inside pusher/main.py
async def heartbeat_loop():
    while True:
        await asyncio.sleep(60)
        try:
            from pusher.handlers import minute_tick
            await minute_tick()
        except Exception:
            log.exception("minute_tick failed")
```

Add `asyncio.create_task(heartbeat_loop())` in `main()`.

## 7. Containerizing pusher

`pusher/Dockerfile`:

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY backend/requirements.txt /app/backend-requirements.txt
RUN pip install --no-cache-dir -r /app/backend-requirements.txt
COPY backend /app/backend
COPY pusher /app/pusher
ENV PYTHONPATH=/app/backend
CMD ["python", "-m", "pusher.main"]
```

This shares code with `backend/` by mounting both folders and
setting `PYTHONPATH`.

## 8. Failure handling

- **Idempotency.** Every handler should be safe to run twice.
  `handle_conversation_completed` shouldn't double-send a push
  notification. Use Redis as a dedupe set: `SETNX cooldown:cid:1`
  with a 1-hour TTL.
- **Backoff.** When an external webhook fails, store the payload in
  a `dead_letter_queue` collection in Firestore with an
  `attempt_count`. A nightly cron retries up to 5 times.
- **Crash recovery.** If pusher dies mid-job, the API didn't lose
  data — Pub/Sub is fire-and-forget by design, but the source-of-
  truth (segments, conversation status) lives in Firestore. The
  worst case is a missed push notification, which is recoverable.

## 9. Observability

Every handler logs `(event_type, uid, latency_ms, success)`. We'll
plug this into Datadog/Sentry in Part 20.

## 10. Commit

```bash
git add backend pusher
git commit -m "feat(part-15): pusher service + redis pubsub + cron scaffolding"
git push
```

## What you should have right now

- [ ] `database/redis_pubsub.py` with `publish/subscribe`.
- [ ] `pusher/main.py` runs as a separate Python process and reacts
  to events.
- [ ] Conversation completion publishes an event; pusher handles it.
- [ ] At least one Cloud Scheduler endpoint protected by `CRON_SECRET`.
- [ ] `pusher/Dockerfile` builds.

---

Next: [Part 16 — Search: Typesense + Vectors](./16-search.md).
