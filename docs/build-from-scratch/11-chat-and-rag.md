# Part 11 — AI Chat & Agentic RAG

> Goal of this part: a `POST /v1/chat/messages` endpoint that turns a
> user question into an LLM answer, where the LLM has access to
> tools that read the user's conversations, memories, and action
> items.

We use **Gemini's function-calling API** for this. Gemini, OpenAI,
and Anthropic all expose the same conceptual pattern (declare tools
→ model emits tool calls → you execute them → feed results back),
and the chat loop below is small enough that you can swap providers
later if you ever need to.

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
from utils.llm.clients import FunctionDeclaration, Schema, Tool, Type

log = logging.getLogger(__name__)


async def search_memories(uid: str, query: str, k: int = 5) -> List[dict]:
    # Use RETRIEVAL_QUERY task type when embedding a search query.
    [vec] = await embed([query], task_type="RETRIEVAL_QUERY")
    rows = vector_db.query(uid, vec, top_k=k, flt={"kind": "memory"})
    return [{"id": rid, "score": score, **(meta or {})} for rid, score, meta in rows]


async def search_conversations(uid: str, query: str, k: int = 5) -> List[dict]:
    [vec] = await embed([query], task_type="RETRIEVAL_QUERY")
    rows = vector_db.query(uid, vec, top_k=k, flt={"kind": "conversation"})
    return [{"id": rid, "score": score, **(meta or {})} for rid, score, meta in rows]


async def list_action_items(uid: str, only_open: bool = True) -> List[dict]:
    return []  # filled in Part 12


# Gemini tool declarations. The schema uses google.genai types — these
# are typed wrappers around the same JSON-Schema dialect, with `type`
# as an enum (Type.STRING / Type.INTEGER / …).
TOOLS = [
    Tool(function_declarations=[
        FunctionDeclaration(
            name="search_memories",
            description="Semantic search over the user's long-term memories.",
            parameters=Schema(
                type=Type.OBJECT,
                properties={
                    "query": Schema(type=Type.STRING),
                    "k": Schema(type=Type.INTEGER),
                },
                required=["query"],
            ),
        ),
        FunctionDeclaration(
            name="search_conversations",
            description="Find conversations relevant to a topic.",
            parameters=Schema(
                type=Type.OBJECT,
                properties={
                    "query": Schema(type=Type.STRING),
                    "k": Schema(type=Type.INTEGER),
                },
                required=["query"],
            ),
        ),
        FunctionDeclaration(
            name="list_action_items",
            description="Returns the user's open tasks.",
            parameters=Schema(
                type=Type.OBJECT,
                properties={
                    "only_open": Schema(type=Type.BOOLEAN),
                },
            ),
        ),
    ]),
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

Gemini's chat API uses **`Content` parts** instead of OpenAI-style
message lists. Each turn is a `Content(role=..., parts=[...])` where
parts can be text, a `function_call`, or a `function_response`.

```python
# in backend/utils/chat/orchestrator.py
import logging
from typing import List

from google.genai import types as gt

from utils.chat.tools import TOOLS, dispatch
from utils.llm.clients import PRO_MODEL, GenerationConfig, gemini_client

log = logging.getLogger(__name__)

SYSTEM = """You are <<YOUR_BRAND>>, an AI assistant with persistent memory of the user's life.
Use the tools to look up specific facts before answering. Cite tool results inline ("(from your conversation '<title>')").
Be concise (≤180 words unless the user asks for more). Never invent facts not present in tool outputs.
If you don't have evidence, say so honestly."""


def _history_to_contents(history: list[dict]) -> List[gt.Content]:
    """Convert a [{role, content}] list to Gemini Content turns.
    Gemini uses 'user' and 'model' (not 'assistant')."""
    out: List[gt.Content] = []
    for m in history:
        role = "model" if m["role"] == "assistant" else "user"
        out.append(gt.Content(role=role, parts=[gt.Part.from_text(m["content"])]))
    return out


async def chat(uid: str, history: list[dict], user_message: str) -> str:
    """Returns the assistant's final reply text."""
    contents = _history_to_contents(history)
    contents.append(gt.Content(role="user", parts=[gt.Part.from_text(user_message)]))

    config = GenerationConfig(
        system_instruction=SYSTEM,
        temperature=0.3,
        tools=TOOLS,
        # Gemini will call tools automatically unless told otherwise.
        # AUTO = the model decides. Use ANY to force a function call.
        tool_config=gt.ToolConfig(
            function_calling_config=gt.FunctionCallingConfig(mode="AUTO"),
        ),
    )

    client = gemini_client()
    for hop in range(4):  # cap tool-use loops
        rsp = await client.aio.models.generate_content(
            model=PRO_MODEL,
            contents=contents,
            config=config,
        )
        candidate = rsp.candidates[0]
        parts = candidate.content.parts or []

        # Collect any function_call parts; if there are none, we're done.
        fn_calls = [p.function_call for p in parts if getattr(p, "function_call", None)]
        if not fn_calls:
            # Concatenate any text parts the model produced.
            return "".join(getattr(p, "text", "") or "" for p in parts)

        # Append the model turn to history so the next call sees its own calls.
        contents.append(candidate.content)

        # Run each tool and append a function_response part per call.
        response_parts: List[gt.Part] = []
        for call in fn_calls:
            args = dict(call.args or {})
            result = await dispatch(uid, call.name, args)
            response_parts.append(
                gt.Part.from_function_response(name=call.name, response=result)
            )
        contents.append(gt.Content(role="user", parts=response_parts))

    return "I had to give up after a few attempts to look this up."
```

The shape is the same as the OpenAI loop — call → maybe tool calls →
run tools → call again with results — but the wire format is
Gemini's. We use `gemini-2.5-pro` here (better reasoning); swap to
`FAST_MODEL` if cost matters more than answer quality.

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

True token-by-token streaming uses Gemini's
`client.aio.models.generate_content_stream(...)` and proxies deltas.
Same idea, more wire shenanigans. Don't bother until v2.

## 7. Anthropic / OpenAI alternative

The orchestrator is intentionally thin so you can swap providers if
Gemini ever lets you down. Two real-world patterns:

- **Anthropic Claude.** Add `anthropic_client()` in
  `utils/llm/clients.py`, then in the chat loop call
  `client.messages.create(model="claude-sonnet-4-5", system=SYSTEM,
  messages=[...], tools=[...])`. Anthropic's tool spec uses
  `input_schema` instead of `parameters`; the dispatch logic is
  otherwise identical.
- **OpenAI GPT-4o / o4-mini.** Add `openai_client()` and call
  `client.chat.completions.create(model=..., messages=..., tools=...,
  tool_choice="auto")`. Each tool call comes back as
  `choice.message.tool_calls[i].function.{name, arguments}`.

The reference repo's `utils/retrieval/` shows an in-production
Anthropic agentic loop with 18+ tools — read it once when you're
ready to scale this up.

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
