# Part 10 — Memories, Embeddings, Vector DB

> Goal of this part: extract long-term facts ("memories") from a
> conversation, embed them into vectors, and store both the text and
> the vector so the chat agent in Part 11 can retrieve them.

## 1. What a "memory" is

A *memory* is a small, atomic, person-centric fact:

| Bad memory | Good memory |
|------------|-------------|
| "Talked about a trip to Italy with mom on Tuesday." | "User is planning a trip to Italy in November." |
| "Likes coffee." | "User prefers oat milk in espresso." |
| "Project X." | "User leads project X at company Acme; deadline is Dec 1." |

A memory is something that **stays true** for at least weeks. Don't
extract things like "user is on the train right now."

## 2. Pinecone setup

In your Pinecone console:

- Create a **serverless index**:
  - Name: `<<YOUR_BRAND>>-memories`
  - Dimension: **768** (matches Gemini `text-embedding-004`)
  - Metric: `cosine`
  - Cloud: AWS, Region: us-east-1 (or whichever is closest)

> Pinecone index dimensions are **immutable** — if you create at 1536
> and later switch embedding models, you have to create a fresh index
> and re-embed. Pick the dimension that matches your default model.
> If you later try Gemini's experimental `gemini-embedding-001`
> (3072 dims), that's a new index, not a column.

Append to `requirements.txt`:

```
pinecone==7.3.0
```

Update `.env`:

```
PINECONE_API_KEY=...
PINECONE_INDEX_NAME=<<YOUR_BRAND>>-memories
```

## 3. Vector DB module

```python
# in backend/database/vector_db.py
import logging
from typing import List, Optional, Tuple

from pinecone import Pinecone, ServerlessSpec

from utils.settings import get_settings

log = logging.getLogger(__name__)
_pc: Optional[Pinecone] = None
_index = None


def _ensure_index():
    global _pc, _index
    if _index is not None:
        return _index
    s = get_settings()
    if not s.pinecone_api_key:
        return None
    _pc = Pinecone(api_key=s.pinecone_api_key)
    name = s.pinecone_index_name
    if name not in [i.name for i in _pc.list_indexes()]:
        _pc.create_index(
            name=name, dimension=768, metric="cosine",
            spec=ServerlessSpec(cloud="aws", region="us-east-1"),
        )
    _index = _pc.Index(name)
    return _index


def upsert(uid: str, items: List[dict]) -> None:
    """items = [{id, vector, metadata}]"""
    idx = _ensure_index()
    if not idx:
        return
    payload = [{
        "id": f"{uid}:{x['id']}",
        "values": x["vector"],
        "metadata": {"uid": uid, **x.get("metadata", {})},
    } for x in items]
    idx.upsert(vectors=payload, namespace=uid)


def query(uid: str, vector: List[float], top_k: int = 8,
          flt: Optional[dict] = None) -> List[Tuple[str, float, dict]]:
    idx = _ensure_index()
    if not idx:
        return []
    r = idx.query(vector=vector, top_k=top_k, include_metadata=True,
                  namespace=uid, filter=flt or None)
    return [(m["id"], float(m["score"]), m.get("metadata") or {}) for m in r.matches]


def delete_for_user(uid: str) -> None:
    idx = _ensure_index()
    if idx:
        idx.delete(delete_all=True, namespace=uid)
```

The **namespace per uid** is the cheapest way to keep users' data
separate in Pinecone. Even if a query has a bug, you'll never get
back another user's vectors.

## 4. Embedding helper

Gemini embeddings come from the same `google-genai` SDK we already
added in Part 09.

```python
# in backend/utils/embeddings.py
from typing import List, Literal

from google.genai import types as genai_types

from utils.llm.clients import EMBEDDING_MODEL, gemini_client


# Gemini supports task-specific embedding for better retrieval quality.
# Pass "RETRIEVAL_DOCUMENT" when *indexing* and "RETRIEVAL_QUERY" when
# *searching*. Same vector space; just better aligned scores.
TaskType = Literal["RETRIEVAL_DOCUMENT", "RETRIEVAL_QUERY", "SEMANTIC_SIMILARITY"]


async def embed(
    texts: List[str],
    task_type: TaskType = "RETRIEVAL_DOCUMENT",
    model: str = EMBEDDING_MODEL,
) -> List[List[float]]:
    if not texts:
        return []
    client = gemini_client()
    rsp = await client.aio.models.embed_content(
        model=model,
        contents=texts,
        config=genai_types.EmbedContentConfig(task_type=task_type),
    )
    # rsp.embeddings is a list of ContentEmbedding; .values is the float list.
    return [e.values for e in rsp.embeddings]
```

When you embed *for storage* (memories, conversation summaries), use
the default `RETRIEVAL_DOCUMENT`. When you embed *a chat question*
just before querying Pinecone, pass `task_type="RETRIEVAL_QUERY"`.
We'll show that in Part 11.

## 5. Memory model + database

```python
# in backend/models/memory.py
from datetime import datetime
from typing import Literal, Optional
from pydantic import BaseModel, Field
import uuid


Visibility = Literal["private", "shared"]
Category = Literal["preference", "fact", "goal", "relationship", "other"]


class Memory(BaseModel):
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    uid: str
    text: str
    category: Category = "fact"
    visibility: Visibility = "private"
    source_conversation_id: Optional[str] = None
    embedding_id: Optional[str] = None  # pinecone vector id
    created_at: datetime = Field(default_factory=datetime.utcnow)
    last_seen_at: datetime = Field(default_factory=datetime.utcnow)
    confidence: float = 0.8
```

```python
# in backend/database/memories.py
from typing import List, Optional

from database._client import db
from models.memory import Memory

_USERS = "users"
_MEM = "memories"


def add(uid: str, m: Memory) -> Memory:
    db.collection(_USERS).document(uid).collection(_MEM).document(m.id) \
        .set(m.model_dump(mode="json"))
    return m


def list_(uid: str, limit: int = 200) -> List[Memory]:
    snaps = (db.collection(_USERS).document(uid).collection(_MEM)
             .order_by("last_seen_at", direction="DESCENDING").limit(limit).stream())
    return [Memory(**s.to_dict()) for s in snaps]


def get(uid: str, mid: str) -> Optional[Memory]:
    s = db.collection(_USERS).document(uid).collection(_MEM).document(mid).get()
    return Memory(**s.to_dict()) if s.exists else None


def delete(uid: str, mid: str) -> None:
    db.collection(_USERS).document(uid).collection(_MEM).document(mid).delete()
```

## 6. Memory extractor (LLM)

Replace the stub from Part 09:

```python
# in backend/utils/llm/memories.py
import json
import logging
from typing import List

from database.memories import add as add_memory
from database import vector_db
from models.memory import Memory
from utils.embeddings import embed
from utils.llm.clients import FAST_MODEL, GenerationConfig, gemini_client

log = logging.getLogger(__name__)

_PROMPT = """Extract a short list of LONG-TERM, FIRST-PERSON-ABOUT-THE-USER facts from this transcript.
Return STRICT JSON: {"memories": [{"text": str, "category": str, "confidence": float}]}
Categories must be one of: preference, fact, goal, relationship, other.
Skip ephemeral facts (today's mood, the weather, "we are at lunch right now").
Skip third-party facts (about other people the user mentions).
If nothing qualifies, return {"memories": []}.
"""


async def extract_memories(uid: str, cid: str, transcript: str) -> None:
    if not transcript or len(transcript) < 200:
        return

    rsp = await gemini_client().aio.models.generate_content(
        model=FAST_MODEL,
        contents=transcript[:18000],
        config=GenerationConfig(
            system_instruction=_PROMPT,
            temperature=0.1,
            response_mime_type="application/json",
        ),
    )
    payload = json.loads(rsp.text or '{"memories": []}')
    items = payload.get("memories", []) or []
    if not items:
        return

    texts = [i["text"] for i in items if isinstance(i.get("text"), str)]
    vecs = await embed(texts)

    saved: List[Memory] = []
    for it, vec in zip(items, vecs):
        m = Memory(
            uid=uid,
            text=it["text"],
            category=it.get("category", "fact"),
            confidence=float(it.get("confidence", 0.7)),
            source_conversation_id=cid,
        )
        add_memory(uid, m)
        m.embedding_id = m.id
        saved.append(m)

    vector_db.upsert(uid, [{
        "id": m.id,
        "vector": v,
        "metadata": {
            "kind": "memory",
            "category": m.category,
            "text": m.text[:512],
            "conversation_id": m.source_conversation_id,
        },
    } for m, v in zip(saved, vecs)])
```

## 7. Also embed conversations for retrieval

In addition to memories, we want to embed *each conversation summary*
so the chat agent can retrieve "the conversation about X."

In `utils/llm/post_process.py`, after writing the title/summary:

```python
# add at the bottom of post_process_conversation, before the action_items call
if data.get("summary"):
    [vec] = await embed([data["summary"]])
    vector_db.upsert(uid, [{
        "id": f"conv:{cid}",
        "vector": vec,
        "metadata": {
            "kind": "conversation",
            "title": data.get("title") or "",
            "category": data.get("category") or "other",
        },
    }])
```

(Add `from utils.embeddings import embed` and
`from database import vector_db` to the imports.)

## 8. Memories REST router

```python
# in backend/routers/memories.py
from fastapi import APIRouter, Depends, HTTPException

from database.memories import add, delete, get, list_
from database import vector_db
from models.memory import Memory
from utils.auth import get_current_user_uid
from utils.embeddings import embed


router = APIRouter(prefix="/v1/memories", tags=["memories"])


@router.get("", response_model=list[Memory])
def list_endpoint(uid: str = Depends(get_current_user_uid)):
    return list_(uid)


@router.post("", response_model=Memory)
async def create(body: Memory, uid: str = Depends(get_current_user_uid)):
    body.uid = uid
    add(uid, body)
    [vec] = await embed([body.text])
    vector_db.upsert(uid, [{
        "id": body.id,
        "vector": vec,
        "metadata": {"kind": "memory", "category": body.category, "text": body.text[:512]},
    }])
    return body


@router.delete("/{mid}")
def delete_endpoint(mid: str, uid: str = Depends(get_current_user_uid)):
    if not get(uid, mid):
        raise HTTPException(404)
    delete(uid, mid)
    return {"ok": True}
```

Register in `main.py`.

## 9. Deduping

When a user has 200+ conversations, "I'm planning a trip to Italy"
will get extracted dozens of times. Dedupe by:

1. Before saving, embed the candidate.
2. Query Pinecone for the top-1 similar memory.
3. If `score > 0.92`, update `last_seen_at` on the existing memory
   instead of creating a duplicate.

```python
sim = vector_db.query(uid, vec, top_k=1, flt={"kind": "memory"})
if sim and sim[0][1] > 0.92:
    existing_id = sim[0][0].split(":", 1)[1]
    from database._client import db
    db.collection("users").document(uid).collection("memories").document(existing_id) \
        .update({"last_seen_at": datetime.utcnow().isoformat()})
    continue  # skip insert
```

## 10. Composite index

Add this to `firestore.indexes.json` so the memories list query
sorted by `last_seen_at` works:

```json
{
  "collectionGroup": "memories",
  "queryScope": "COLLECTION",
  "fields": [
    {"fieldPath": "last_seen_at", "order": "DESCENDING"}
  ]
}
```

`firebase deploy --only firestore:indexes`.

## 11. Commit

```bash
git add backend
git commit -m "feat(part-10): memories + embeddings + pinecone"
git push
```

## What you should have right now

- [ ] A Pinecone index with namespace-per-uid.
- [ ] Conversation post-processing extracts ≤10 memories per call.
- [ ] Each memory's embedding is upserted to Pinecone.
- [ ] `GET /v1/memories` returns the list, `DELETE` removes one.
- [ ] Conversation summaries are also embedded for retrieval.
- [ ] Duplicate memories are merged via cosine ≥ 0.92.

---

Next: [Part 11 — AI Chat & Agentic RAG](./11-chat-and-rag.md).
