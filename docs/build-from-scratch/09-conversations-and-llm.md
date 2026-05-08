# Part 09 — Conversations Lifecycle & LLM Post-Processing

> Goal of this part: a `conversations/{id}` document that auto-opens
> on first audio, auto-closes after silence, and gets a title +
> summary + category from an LLM.

## 1. Conversation states

A conversation moves through these states, in order:

```
in_progress  →  processing  →  completed
                    │
                    └──→ failed (LLM error / no transcript)
```

- **`in_progress`** — the WS is open, audio is flowing, transcripts
  appending.
- **`processing`** — silence threshold crossed, LLM jobs kicked off.
- **`completed`** — title, summary, action items, memories generated;
  visible on the timeline.
- **`failed`** — terminal state; user can manually re-process.

## 2. Models

```python
# in backend/models/conversation.py
from datetime import datetime
from typing import List, Literal, Optional
from pydantic import BaseModel, Field
import uuid


Status = Literal["in_progress", "processing", "completed", "failed"]
Category = Literal["personal", "work", "social", "education", "health", "other"]


class Conversation(BaseModel):
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    uid: str
    status: Status = "in_progress"
    title: Optional[str] = None
    summary: Optional[str] = None
    category: Optional[Category] = None
    started_at: datetime = Field(default_factory=datetime.utcnow)
    ended_at: Optional[datetime] = None
    duration_ms: int = 0
    audio_object_path: Optional[str] = None  # gs://bucket/path
    photo_paths: List[str] = Field(default_factory=list)
    geolocation: Optional[dict] = None  # {"lat": x, "lng": y}
```

## 3. Database module

```python
# in backend/database/conversations.py
from datetime import datetime
from typing import List, Optional

from database._client import db
from models.conversation import Conversation

_USERS = "users"
_CONVS = "conversations"


def get(uid: str, cid: str) -> Optional[Conversation]:
    snap = db.collection(_USERS).document(uid).collection(_CONVS).document(cid).get()
    return Conversation(**snap.to_dict()) if snap.exists else None


def upsert(c: Conversation) -> Conversation:
    db.collection(_USERS).document(c.uid).collection(_CONVS).document(c.id).set(
        c.model_dump(mode="json"), merge=True
    )
    return c


def list_for(uid: str, limit: int = 50) -> List[Conversation]:
    snaps = (db.collection(_USERS).document(uid).collection(_CONVS)
             .order_by("started_at", direction="DESCENDING").limit(limit).stream())
    return [Conversation(**s.to_dict()) for s in snaps]


def patch(uid: str, cid: str, patch: dict) -> None:
    db.collection(_USERS).document(uid).collection(_CONVS).document(cid).set(patch, merge=True)


def mark_status(uid: str, cid: str, status: str, **fields) -> None:
    p = {"status": status, "updated_at": datetime.utcnow().isoformat(), **fields}
    patch(uid, cid, p)
```

## 4. Open/close in the listen handler

Update Part 07's `listen.py`:

- On WS accept, create a `Conversation` and `upsert` it.
- On WS close, mark `ended_at` and trigger `processing` jobs.

```python
# inside listen() handler, replacing the conversation_id init:
conv = Conversation(uid=uid)
conversation_id = conv.id
upsert(conv)
await ws.send_text(json.dumps({"type": "conversation.opened", "id": conv.id}))

# ... inside finally block, before closing dg:
duration_ms = int((loop.time() - last_audio_ts_at_open) * 1000)  # approx
mark_status(uid, conv.id, "processing", ended_at=datetime.utcnow().isoformat(),
            duration_ms=duration_ms)
asyncio.create_task(post_process_conversation(uid, conv.id))
```

(`post_process_conversation` is what we build next.)

## 5. The end-of-silence timer (the *other* way to close)

We must also auto-close conversations that the user "abandons." A
common heuristic:

> If no audio frames have arrived for **120 seconds** (still WS open),
> finalize the conversation and start a new one when audio resumes.

Implement this as a watchdog task inside `listen`:

```python
SILENCE_FINALIZE_S = 120

async def watchdog():
    nonlocal conversation_id, conv  # noqa: PLW0127
    while True:
        await asyncio.sleep(15)
        if loop.time() - last_audio_ts > SILENCE_FINALIZE_S:
            mark_status(uid, conversation_id, "processing",
                        ended_at=datetime.utcnow().isoformat())
            asyncio.create_task(post_process_conversation(uid, conversation_id))
            # open a fresh conversation under the same WS
            conv = Conversation(uid=uid)
            conversation_id = conv.id
            upsert(conv)
            await ws.send_text(json.dumps({"type": "conversation.closed", "id": conv.id}))
            await ws.send_text(json.dumps({"type": "conversation.opened", "id": conv.id}))
```

## 6. The LLM client wrapper

Append to `requirements.txt`:

```
openai==1.104.2
anthropic>=0.52.0
tiktoken==0.7.0
```

Create `backend/utils/llm/clients.py`:

```python
# in backend/utils/llm/clients.py
import logging
from typing import Optional

import anthropic
import openai

from utils.settings import get_settings

log = logging.getLogger(__name__)


def openai_client() -> openai.AsyncOpenAI:
    return openai.AsyncOpenAI(api_key=get_settings().openai_api_key)


def anthropic_client() -> Optional[anthropic.AsyncAnthropic]:
    key = __import__("os").environ.get("ANTHROPIC_API_KEY", "")
    return anthropic.AsyncAnthropic(api_key=key) if key else None
```

Create `backend/utils/llm/post_process.py`:

```python
# in backend/utils/llm/post_process.py
import json
import logging
from typing import List

from database.conversations import mark_status, patch
from database.segments import list_segments
from models.segment import TranscriptSegment
from utils.llm.clients import openai_client

log = logging.getLogger(__name__)

_PROMPT = """You will receive a verbatim conversation transcript with speaker labels.
Return STRICT JSON with these keys:
  title       (string, ≤60 chars, no quotes)
  summary     (string, 3–5 sentences, third person)
  category    (one of: personal, work, social, education, health, other)
  action_items (array of strings, imperative; can be empty)
  language    (ISO 639-1 of the dominant language)

Be brief. Do not invent facts. If the transcript is too short or empty, return all empty fields.
"""


def _format(segs: List[TranscriptSegment]) -> str:
    lines = []
    for s in segs:
        spk = s.speaker_id or "?"
        lines.append(f"[{spk}] {s.text}")
    return "\n".join(lines)


async def post_process_conversation(uid: str, cid: str) -> None:
    try:
        segs = list_segments(uid, cid)
        if not segs:
            mark_status(uid, cid, "completed", title="(empty conversation)")
            return

        transcript = _format(segs)
        client = openai_client()

        rsp = await client.chat.completions.create(
            model="gpt-4o-mini",
            response_format={"type": "json_object"},
            messages=[
                {"role": "system", "content": _PROMPT},
                {"role": "user", "content": transcript[:18000]},
            ],
            temperature=0.2,
        )
        data = json.loads(rsp.choices[0].message.content or "{}")

        patch(uid, cid, {
            "title": data.get("title") or None,
            "summary": data.get("summary") or None,
            "category": data.get("category") or "other",
        })
        mark_status(uid, cid, "completed")

        # action items + memories follow in Parts 10 & 12
        from utils.llm.action_items import extract_action_items
        await extract_action_items(uid, cid, data.get("action_items") or [])
        from utils.llm.memories import extract_memories
        await extract_memories(uid, cid, transcript)
    except Exception:
        log.exception("post_process failed")
        mark_status(uid, cid, "failed")
```

We import action-items + memories extractors that we'll fill in
during Parts 10 and 12. For now, stub them so the code runs:

```python
# in backend/utils/llm/action_items.py
async def extract_action_items(uid: str, cid: str, items: list[str]) -> None:
    pass

# in backend/utils/llm/memories.py
async def extract_memories(uid: str, cid: str, transcript: str) -> None:
    pass
```

## 7. The conversations REST router

```python
# in backend/routers/conversations.py
from fastapi import APIRouter, Depends, HTTPException

from database.conversations import get, list_for, patch
from database.segments import list_segments
from models.conversation import Conversation
from utils.auth import get_current_user_uid
from utils.llm.post_process import post_process_conversation

router = APIRouter(prefix="/v1/conversations", tags=["conversations"])


@router.get("", response_model=list[Conversation])
def list_(uid: str = Depends(get_current_user_uid)):
    return list_for(uid)


@router.get("/{cid}", response_model=Conversation)
def detail(cid: str, uid: str = Depends(get_current_user_uid)):
    c = get(uid, cid)
    if not c:
        raise HTTPException(404, "not found")
    return c


@router.get("/{cid}/segments")
def segments(cid: str, uid: str = Depends(get_current_user_uid)):
    return [s.model_dump(mode="json") for s in list_segments(uid, cid)]


@router.post("/{cid}/reprocess")
async def reprocess(cid: str, uid: str = Depends(get_current_user_uid)):
    if not get(uid, cid):
        raise HTTPException(404, "not found")
    patch(uid, cid, {"status": "processing"})
    await post_process_conversation(uid, cid)
    return {"ok": True}


@router.delete("/{cid}")
def delete(cid: str, uid: str = Depends(get_current_user_uid)):
    if not get(uid, cid):
        raise HTTPException(404, "not found")
    # cascading delete of subcollections handled in database/conversations.py
    from database._client import db
    base = db.collection("users").document(uid).collection("conversations").document(cid)
    for s in base.collection("segments").stream():
        s.reference.delete()
    base.delete()
    return {"ok": True}
```

Register in `main.py`:

```python
from routers import conversations
app.include_router(conversations.router)
```

## 8. The "merge" endpoint

Users sometimes have two conversations split by 3 seconds of silence
that should really be one. Provide a merge endpoint:

```python
@router.post("/{cid}/merge/{other_cid}")
def merge(cid: str, other_cid: str, uid: str = Depends(get_current_user_uid)):
    """Append other_cid's segments into cid, recompute, delete other_cid."""
    base = get(uid, cid); other = get(uid, other_cid)
    if not base or not other:
        raise HTTPException(404, "missing")
    if base.uid != uid or other.uid != uid:
        raise HTTPException(403)
    other_segs = list_segments(uid, other_cid)
    for s in other_segs:
        s.conversation_id = cid
        from database.segments import append_segment
        append_segment(uid, s)
    delete(other_cid, uid)  # also cleans subs
    patch(uid, cid, {"status": "processing"})
    import asyncio
    asyncio.create_task(post_process_conversation(uid, cid))
    return {"ok": True}
```

## 9. Cost guardrails

GPT-4o-mini is cheap (~$0.15 / M input tokens), but a 2-hour
conversation at 100 wpm ≈ 12K tokens. Multiply by users and that adds
up. Practical caps:

- Truncate transcript to 18 000 chars before sending to LLM.
- Skip post-processing if `len(segments) < 3`.
- Cache the result by hash of the transcript (so reprocessing is
  free) — implement with `redis_db.get/set`.
- Track per-user token spend (`utils/llm/usage.py` — Part 20).

## 10. Test it

End-to-end: stream `hello.wav` to the listen WebSocket, close the
socket, wait ~5 seconds, then:

```bash
curl -H "Authorization: Bearer ${ADMIN_KEY}uid_test_42" \
  http://localhost:8080/v1/conversations | jq '.[0]'
```

Expect a populated `title`, `summary`, `status: "completed"`.

## 11. Commit

```bash
git add backend
git commit -m "feat(part-09): conversation lifecycle + LLM post-processing"
git push
```

## What you should have right now

- [ ] Each WS open creates a `Conversation` doc.
- [ ] WS close (or 2 min silence) flips it to `processing`.
- [ ] LLM produces title + summary + category.
- [ ] `GET /v1/conversations` returns the list.
- [ ] `POST /v1/conversations/{id}/reprocess` works.
- [ ] `DELETE` cascades segments.

---

Next: [Part 10 — Memories, Embeddings, Vector DB](./10-memories-and-embeddings.md).
