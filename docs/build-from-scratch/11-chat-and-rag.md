# Part 11 — AI Chat & Agentic RAG

> Goal of this part: a `POST /v1/chat/messages` endpoint that turns a
> user question into an LLM answer, where the LLM has access to
> tools that read the user's conversations, memories, and action
> items.

The reference Omi backend uses Anthropic's tool-using API for this.
You can use OpenAI's function calling or Anthropic tools — the
shape is similar. We'll show OpenAI here to keep it simple, then note
how to swap to Anthropic.

## 1. Chat data model

```python
# in backend/models/chat.py
from datetime import datetime
from typing import List, Literal, Optional
from pydantic import BaseModel, Field
import uuid


Role = Literal["user", "assistant", "system", "tool"]


class ChatMessage(BaseModel):
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    uid: str
    session_id: str
    role: Role
    content: str
    tool_calls: Optional[list] = None
    created_at: datetime = Field(default_factory=datetime.utcnow)


class ChatSession(BaseModel):
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    uid: str
    title: Optional[str] = None
    created_at: datetime = Field(default_factory=datetime.utcnow)


class SendRequest(BaseModel):
    session_id: Optional[str] = None
    message: str


class SendResponse(BaseModel):
    session_id: str
    user_message: ChatMessage
    assistant_message: ChatMessage
```

## 2. Database module

```python
# in backend/database/chat.py
from typing import List, Optional

from database._client import db
from models.chat import ChatMessage, ChatSession

_USERS = "users"
_SESSIONS = "chat_sessions"
_MESSAGES = "chat_messages"


def get_or_create_session(uid: str, session_id: Optional[str]) -> ChatSession:
    if session_id:
        snap = db.collection(_USERS).document(uid).collection(_SESSIONS).document(session_id).get()
        if snap.exists:
            return ChatSession(**snap.to_dict())
    s = ChatSession(uid=uid, id=session_id or ChatSession().id)
    db.collection(_USERS).document(uid).collection(_SESSIONS).document(s.id) \
        .set(s.model_dump(mode="json"))
    return s


def append(uid: str, m: ChatMessage) -> None:
    db.collection(_USERS).document(uid).collection(_MESSAGES).document(m.id) \
        .set(m.model_dump(mode="json"))


def history(uid: str, session_id: str, limit: int = 30) -> List[ChatMessage]:
    snaps = (db.collection(_USERS).document(uid).collection(_MESSAGES)
             .where("session_id", "==", session_id)
             .order_by("created_at").limit(limit).stream())
    return [ChatMessage(**s.to_dict()) for s in snaps]
```

## 3. The tools the LLM can call

The "agentic" pattern: we give the LLM a few small functions it can
invoke to look up real data. The LLM decides when to call which.

We expose three tools to start:

| Tool | What it does |
|------|--------------|
| `search_memories` | Vector search over user's memories. |
| `search_conversations` | Vector search over user's conversation summaries. |
| `list_action_items` | Returns the user's open tasks. |

Add Part 12's `action_items.py` later — for now we'll stub
`list_action_items`.

```python
# in backend/utils/chat/tools.py
import logging
from typing import List

from database import vector_db
from utils.embeddings import embed

log = logging.getLogger(__name__)


async def search_memories(uid: str, query: str, k: int = 5) -> List[dict]:
    [vec] = await embed([query])
    rows = vector_db.query(uid, vec, top_k=k, flt={"kind": "memory"})
    return [{"id": rid, "score": score, **(meta or {})} for rid, score, meta in rows]


async def search_conversations(uid: str, query: str, k: int = 5) -> List[dict]:
    [vec] = await embed([query])
    rows = vector_db.query(uid, vec, top_k=k, flt={"kind": "conversation"})
    return [{"id": rid, "score": score, **(meta or {})} for rid, score, meta in rows]


async def list_action_items(uid: str, only_open: bool = True) -> List[dict]:
    return []  # filled in Part 12


TOOL_SPEC = [
    {
        "type": "function",
        "function": {
            "name": "search_memories",
            "description": "Semantic search over the user's long-term memories.",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string"},
                    "k": {"type": "integer", "default": 5},
                },
                "required": ["query"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "search_conversations",
            "description": "Find conversations relevant to a topic.",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string"},
                    "k": {"type": "integer", "default": 5},
                },
                "required": ["query"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "list_action_items",
            "description": "Returns the user's open tasks.",
            "parameters": {
                "type": "object",
                "properties": {
                    "only_open": {"type": "boolean", "default": True},
                },
            },
        },
    },
]


async def dispatch(uid: str, name: str, args: dict) -> dict:
    if name == "search_memories":
        return {"results": await search_memories(uid, args["query"], int(args.get("k", 5)))}
    if name == "search_conversations":
        return {"results": await search_conversations(uid, args["query"], int(args.get("k", 5)))}
    if name == "list_action_items":
        return {"results": await list_action_items(uid, bool(args.get("only_open", True)))}
    return {"error": f"unknown tool {name}"}
```

## 4. Chat orchestration loop

```python
# in backend/utils/chat/orchestrator.py
import json
import logging

from utils.chat.tools import TOOL_SPEC, dispatch
from utils.llm.clients import openai_client

log = logging.getLogger(__name__)

SYSTEM = """You are <<YOUR_BRAND>>, an AI assistant with persistent memory of the user's life.
Use the tools to look up specific facts before answering. Cite tool results inline ("(from your conversation '<title>')").
Be concise (≤180 words unless the user asks for more). Never invent facts not present in tool outputs.
If you don't have evidence, say so honestly."""


async def chat(uid: str, history: list[dict], user_message: str) -> str:
    """Returns the assistant's final reply text."""
    msgs = [{"role": "system", "content": SYSTEM}]
    msgs.extend(history)
    msgs.append({"role": "user", "content": user_message})

    client = openai_client()
    for hop in range(4):  # cap tool-use loops
        rsp = await client.chat.completions.create(
            model="gpt-4o-mini",
            messages=msgs,
            tools=TOOL_SPEC,
            tool_choice="auto",
            temperature=0.3,
        )
        choice = rsp.choices[0]
        if not choice.message.tool_calls:
            return choice.message.content or ""

        msgs.append({
            "role": "assistant",
            "content": choice.message.content or "",
            "tool_calls": [tc.model_dump() for tc in choice.message.tool_calls],
        })

        for tc in choice.message.tool_calls:
            try:
                args = json.loads(tc.function.arguments or "{}")
            except Exception:
                args = {}
            result = await dispatch(uid, tc.function.name, args)
            msgs.append({
                "role": "tool",
                "tool_call_id": tc.id,
                "content": json.dumps(result)[:8000],
            })

    return "I had to give up after a few attempts to look this up."
```

## 5. Chat router

```python
# in backend/routers/chat.py
from fastapi import APIRouter, Depends

from database.chat import append, get_or_create_session, history
from models.chat import ChatMessage, SendRequest, SendResponse
from utils.auth import get_current_user_uid
from utils.chat.orchestrator import chat
from utils.rate_limit import with_rate_limit

router = APIRouter(prefix="/v1/chat", tags=["chat"])


@router.post(
    "/messages",
    response_model=SendResponse,
)
async def send(
    body: SendRequest,
    uid: str = Depends(with_rate_limit(get_current_user_uid, "chat", limit=30, window_seconds=60)),
):
    sess = get_or_create_session(uid, body.session_id)
    user_msg = ChatMessage(uid=uid, session_id=sess.id, role="user", content=body.message)
    append(uid, user_msg)

    prior = history(uid, sess.id, limit=20)
    convo = [{"role": m.role, "content": m.content} for m in prior if m.role in ("user", "assistant")]

    answer = await chat(uid, convo[:-1], body.message)  # exclude the just-added user msg

    asst_msg = ChatMessage(uid=uid, session_id=sess.id, role="assistant", content=answer)
    append(uid, asst_msg)

    return SendResponse(session_id=sess.id, user_message=user_msg, assistant_message=asst_msg)
```

Register in `main.py`.

## 6. Streaming responses

The above returns a single JSON blob. For a chat-like experience the
mobile app wants tokens as they're produced. Add an SSE endpoint:

```python
# in backend/routers/chat.py
import asyncio
from fastapi.responses import StreamingResponse


@router.post("/messages/stream")
async def send_stream(
    body: SendRequest,
    uid: str = Depends(with_rate_limit(get_current_user_uid, "chat", limit=30, window_seconds=60)),
):
    sess = get_or_create_session(uid, body.session_id)
    user_msg = ChatMessage(uid=uid, session_id=sess.id, role="user", content=body.message)
    append(uid, user_msg)

    async def gen():
        # naive: wait for the full answer, then chunk-stream it.
        prior = history(uid, sess.id, limit=20)
        convo = [{"role": m.role, "content": m.content} for m in prior if m.role in ("user","assistant")]
        full = await chat(uid, convo[:-1], body.message)
        for i in range(0, len(full), 40):
            yield f"data: {full[i:i+40]}\n\n"
            await asyncio.sleep(0.02)
        asst_msg = ChatMessage(uid=uid, session_id=sess.id, role="assistant", content=full)
        append(uid, asst_msg)
        yield f"event: done\ndata: {{\"id\": \"{asst_msg.id}\"}}\n\n"

    return StreamingResponse(gen(), media_type="text/event-stream")
```

True token-by-token streaming uses `client.chat.completions.create(stream=True)`
and proxies deltas. Same idea, more wire shenanigans. Don't bother
until v2.

## 7. Anthropic alternative

Swap `openai_client().chat.completions.create(...)` with:

```python
from utils.llm.clients import anthropic_client
client = anthropic_client()
rsp = await client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=1024,
    system=SYSTEM,
    messages=[...],
    tools=[...],   # Anthropic tool spec is slightly different
)
```

Anthropic's tool spec uses `input_schema` instead of OpenAI's
`parameters`. The reference repo's `utils/retrieval/` shows an
in-production Anthropic agentic loop with 18+ tools — read it once
when you're ready for that.

## 8. Test it

```bash
http POST :8080/v1/chat/messages \
  Authorization:"Bearer ${ADMIN_KEY}uid_test_42" \
  message="What did I talk about with my mom yesterday?" | jq
```

(Naturally, you need to have actually had a conversation about your
mom for the answer to be interesting.)

## 9. Commit

```bash
git add backend
git commit -m "feat(part-11): chat + agentic rag"
git push
```

## What you should have right now

- [ ] `POST /v1/chat/messages` returns an answer that uses tools.
- [ ] `chat_sessions/`, `chat_messages/` exist in Firestore.
- [ ] Tools log is visible in your backend logs.
- [ ] (Optional) `POST /v1/chat/messages/stream` gives SSE chunks.
- [ ] Rate limit caps abusive callers at 30/min.

---

Next: [Part 12 — Action Items, Calendar, Task Integrations](./12-action-items-and-integrations.md).
