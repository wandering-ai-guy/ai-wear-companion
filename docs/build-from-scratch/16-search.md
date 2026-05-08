# Part 16 — Search: Typesense + Vectors

> Goal of this part: a `GET /v1/search` endpoint that finds anything
> the user has ever said, fast — both keyword and semantic.

## 1. Two kinds of search

We already have **vector** search via Pinecone (Part 10). That's
great for "show me conversations *about* X." It's bad for "find the
exact phrase 'lasagna recipe'."

For exact-keyword search we add **Typesense**, an open-source
ElasticSearch-alternative with a generous free Typesense Cloud tier
and a single-binary install if you self-host.

The chat agent in Part 11 will get *both* tools.

## 2. Typesense setup

In Typesense Cloud:

1. Create a **cluster** (3 nodes minimum for HA, 1 node OK for dev).
2. Generate a `search-only` API key and an `admin` API key.
3. Note the `host` (`xxxxx.a1.typesense.net`) and `port` (443).

`.env`:

```
TYPESENSE_HOST=xxxxx.a1.typesense.net
TYPESENSE_HOST_PORT=443
TYPESENSE_API_KEY=<admin key>
```

Append to `requirements.txt`:

```
typesense==0.21.0
```

## 3. Schema

We want one collection that holds **conversation segments + memories**
(unified search). Define the schema once at startup:

```python
# in backend/database/typesense_db.py
import logging
import os
from typing import Optional

import typesense

log = logging.getLogger(__name__)
_client: Optional[typesense.Client] = None
COLLECTION = "<<YOUR_BRAND>>_search"

SCHEMA = {
    "name": COLLECTION,
    "fields": [
        {"name": "uid", "type": "string", "facet": True},
        {"name": "kind", "type": "string", "facet": True},  # "segment" | "memory" | "conversation"
        {"name": "conversation_id", "type": "string", "optional": True},
        {"name": "text", "type": "string"},
        {"name": "speaker_id", "type": "string", "optional": True, "facet": True},
        {"name": "category", "type": "string", "optional": True, "facet": True},
        {"name": "created_at_ts", "type": "int64"},
    ],
    "default_sorting_field": "created_at_ts",
}


def _client_or_none() -> Optional[typesense.Client]:
    global _client
    if _client is not None:
        return _client
    host = os.environ.get("TYPESENSE_HOST")
    if not host:
        return None
    _client = typesense.Client({
        "api_key": os.environ["TYPESENSE_API_KEY"],
        "nodes": [{"host": host, "port": int(os.environ.get("TYPESENSE_HOST_PORT", 443)), "protocol": "https"}],
        "connection_timeout_seconds": 5,
    })
    return _client


def ensure_collection() -> None:
    c = _client_or_none()
    if not c:
        return
    try:
        c.collections[COLLECTION].retrieve()
    except Exception:
        c.collections.create(SCHEMA)


def index_doc(doc: dict) -> None:
    c = _client_or_none()
    if not c:
        return
    try:
        c.collections[COLLECTION].documents.upsert(doc)
    except Exception:
        log.warning("typesense.upsert failed", exc_info=True)


def search(uid: str, q: str, kind: Optional[str] = None,
           per_page: int = 20) -> list[dict]:
    c = _client_or_none()
    if not c:
        return []
    filter_by = f"uid:={uid}"
    if kind:
        filter_by += f" && kind:={kind}"
    try:
        r = c.collections[COLLECTION].documents.search({
            "q": q,
            "query_by": "text",
            "filter_by": filter_by,
            "per_page": per_page,
            "sort_by": "_text_match:desc,created_at_ts:desc",
        })
    except Exception:
        log.warning("typesense.search failed", exc_info=True)
        return []
    return [hit["document"] for hit in r.get("hits", [])]
```

Call `ensure_collection()` in `main.py`'s lifespan.

## 4. Indexing on write

Every place we save a segment, memory, or conversation, also index:

```python
# in backend/database/segments.py — after the .set()
from database import typesense_db
typesense_db.index_doc({
    "id": f"seg:{seg.id}",
    "uid": uid,
    "kind": "segment",
    "conversation_id": seg.conversation_id,
    "text": seg.text,
    "speaker_id": seg.speaker_id or "",
    "created_at_ts": int(seg.created_at.timestamp()),
})
```

Same for memories and conversation summaries.

For existing data, write a one-off backfill script:

```python
# in backend/scripts/backfill_typesense.py
from database import typesense_db
from database._client import db
import time

typesense_db.ensure_collection()

for u in db.collection("users").stream():
    uid = u.id
    for c in db.collection("users").document(uid).collection("conversations").stream():
        cdata = c.to_dict()
        typesense_db.index_doc({
            "id": f"conv:{c.id}",
            "uid": uid, "kind": "conversation",
            "conversation_id": c.id,
            "text": (cdata.get("title") or "") + " — " + (cdata.get("summary") or ""),
            "category": cdata.get("category") or "",
            "created_at_ts": int(time.time()),
        })
```

## 5. Search router

```python
# in backend/routers/search.py
from typing import Optional, Literal
from fastapi import APIRouter, Depends, Query

from database import typesense_db
from utils.auth import get_current_user_uid


router = APIRouter(prefix="/v1/search", tags=["search"])


@router.get("")
def search_endpoint(
    q: str = Query(..., min_length=1),
    kind: Optional[Literal["segment", "memory", "conversation"]] = None,
    per_page: int = Query(20, ge=1, le=100),
    uid: str = Depends(get_current_user_uid),
):
    return {"results": typesense_db.search(uid, q, kind=kind, per_page=per_page)}
```

## 6. Hybrid search (optional but worth it)

The best search blends keyword (Typesense) and semantic (Pinecone):

```python
# in backend/utils/hybrid_search.py
from utils.embeddings import embed
from database import typesense_db, vector_db


async def hybrid(uid: str, q: str, k: int = 10):
    text_hits = typesense_db.search(uid, q, per_page=k)

    [vec] = await embed([q])
    vec_hits_raw = vector_db.query(uid, vec, top_k=k)
    vec_hits = [{"id": rid, "score": s, **(meta or {})} for rid, s, meta in vec_hits_raw]

    seen, merged = set(), []
    for h in text_hits + vec_hits:
        key = h.get("id") or h.get("conversation_id") or h.get("text")
        if key in seen:
            continue
        seen.add(key)
        merged.append(h)
    return merged[:k]
```

Expose as `/v1/search/hybrid?q=...`. The chat tools should prefer
this over either component alone.

## 7. Index lifecycle

- **On user delete:** delete documents `filter_by=uid:=<uid>` from
  Typesense. Add this to `cascade_delete()` in `database/users.py`.
- **On conversation delete:** delete docs with
  `conversation_id:=<cid>`.
- **Reindex:** if you ever change schema, `c.collections[COLLECTION].delete()`
  then re-create + run the backfill script.

## 8. Commit

```bash
git add backend
git commit -m "feat(part-16): typesense search + hybrid endpoint"
git push
```

## What you should have right now

- [ ] Typesense collection exists.
- [ ] Saving a segment/memory/conversation also indexes a doc.
- [ ] `GET /v1/search?q=...` returns hits filtered by `uid`.
- [ ] (Optional) `GET /v1/search/hybrid` blends keyword + vector.
- [ ] Backfill script populates an existing dataset.

---

Next: [Part 17 — Privacy, Encryption, Log Sanitization](./17-privacy-and-security.md).
