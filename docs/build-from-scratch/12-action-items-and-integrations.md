# Part 12 — Action Items, Calendar, Task Integrations

> Goal of this part: extract actionable to-dos from conversations,
> store them, and let the user sync them to external services
> (Google Calendar, Todoist).

## 1. The action item model

```python
# in backend/models/action_item.py
from datetime import datetime
from typing import Optional
from pydantic import BaseModel, Field
import uuid


class ActionItem(BaseModel):
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    uid: str
    title: str
    description: Optional[str] = None
    due_at: Optional[datetime] = None
    completed: bool = False
    completed_at: Optional[datetime] = None
    source_conversation_id: Optional[str] = None
    external_ids: dict = Field(default_factory=dict)  # e.g. {"todoist": "12345", "google_cal": "..."}
    created_at: datetime = Field(default_factory=datetime.utcnow)
```

```python
# in backend/database/action_items.py
from typing import List, Optional
from database._client import db
from models.action_item import ActionItem

_USERS = "users"; _COL = "action_items"


def add(uid: str, item: ActionItem) -> ActionItem:
    db.collection(_USERS).document(uid).collection(_COL).document(item.id).set(
        item.model_dump(mode="json"))
    return item


def list_(uid: str, only_open: bool = True, limit: int = 200) -> List[ActionItem]:
    q = db.collection(_USERS).document(uid).collection(_COL)
    if only_open:
        q = q.where("completed", "==", False)
    return [ActionItem(**s.to_dict()) for s in q.order_by("created_at",
            direction="DESCENDING").limit(limit).stream()]


def get(uid: str, mid: str) -> Optional[ActionItem]:
    s = db.collection(_USERS).document(uid).collection(_COL).document(mid).get()
    return ActionItem(**s.to_dict()) if s.exists else None


def patch(uid: str, mid: str, p: dict) -> None:
    db.collection(_USERS).document(uid).collection(_COL).document(mid).set(p, merge=True)
```

## 2. Replace the stub from Part 09

```python
# in backend/utils/llm/action_items.py
from datetime import datetime
import logging
from typing import List

from database.action_items import add
from models.action_item import ActionItem

log = logging.getLogger(__name__)


async def extract_action_items(uid: str, cid: str, items: List[str]) -> None:
    for raw in items[:10]:  # cap at 10 per conversation
        title = (raw or "").strip()
        if not title or len(title) > 240:
            continue
        item = ActionItem(uid=uid, title=title, source_conversation_id=cid)
        add(uid, item)
```

You can make this LLM-richer by asking GPT to also produce
`description` and `due_at` directly. Keep it simple at first.

## 3. Wire up the chat tool

In `utils/chat/tools.py`, replace the stub:

```python
async def list_action_items(uid: str, only_open: bool = True) -> List[dict]:
    from database.action_items import list_
    return [m.model_dump(mode="json") for m in list_(uid, only_open=only_open, limit=50)]
```

## 4. REST endpoints

```python
# in backend/routers/action_items.py
from fastapi import APIRouter, Depends, HTTPException

from database.action_items import add, get, list_, patch
from models.action_item import ActionItem
from utils.auth import get_current_user_uid


router = APIRouter(prefix="/v1/action-items", tags=["action-items"])


@router.get("", response_model=list[ActionItem])
def list_endpoint(only_open: bool = True, uid: str = Depends(get_current_user_uid)):
    return list_(uid, only_open=only_open)


@router.post("", response_model=ActionItem)
def create(body: ActionItem, uid: str = Depends(get_current_user_uid)):
    body.uid = uid
    return add(uid, body)


@router.patch("/{mid}", response_model=ActionItem)
def update(mid: str, body: dict, uid: str = Depends(get_current_user_uid)):
    item = get(uid, mid)
    if not item:
        raise HTTPException(404)
    patch(uid, mid, body)
    return get(uid, mid)


@router.post("/{mid}/complete")
def complete(mid: str, uid: str = Depends(get_current_user_uid)):
    from datetime import datetime
    patch(uid, mid, {"completed": True, "completed_at": datetime.utcnow().isoformat()})
    return {"ok": True}
```

Register in `main.py`.

## 5. OAuth scaffolding for integrations

We'll integrate with **Google Calendar** and **Todoist** as examples.

The pattern is always the same:

1. User taps "Connect Todoist" in the mobile app.
2. App opens a webview pointed at
   `https://api.<<YOUR_DOMAIN>>/v1/integrations/<provider>/start`,
   which redirects to the provider's OAuth screen.
3. Provider sends an auth code to
   `https://api.<<YOUR_DOMAIN>>/v1/integrations/<provider>/callback`.
4. Backend exchanges the code for tokens, stores them encrypted under
   `users/{uid}/integrations/<provider>`.

Append to `requirements.txt`:

```
google-auth==2.32.0
google-auth-oauthlib==1.2.1
google-api-python-client==2.139.0
```

Create `backend/database/integrations.py`:

```python
# in backend/database/integrations.py
from typing import Optional
from database._client import db
from utils.encryption import encrypt_for, decrypt_for

_USERS = "users"; _COL = "integrations"


def save_tokens(uid: str, provider: str, tokens: dict) -> None:
    payload = {
        "provider": provider,
        "tokens_enc": encrypt_for(uid, __import__("json").dumps(tokens)),
    }
    db.collection(_USERS).document(uid).collection(_COL).document(provider).set(payload, merge=True)


def get_tokens(uid: str, provider: str) -> Optional[dict]:
    snap = db.collection(_USERS).document(uid).collection(_COL).document(provider).get()
    if not snap.exists:
        return None
    return __import__("json").loads(decrypt_for(uid, snap.to_dict()["tokens_enc"]))


def remove(uid: str, provider: str) -> None:
    db.collection(_USERS).document(uid).collection(_COL).document(provider).delete()
```

Note we encrypt the tokens with the **per-user** key from Part 05 —
defense in depth.

## 6. Google Calendar

In GCP console:

- APIs & Services → Credentials → Create OAuth client ID
  (Web application).
- Authorized redirect URIs:
  `https://api.<<YOUR_DOMAIN>>/v1/integrations/google/callback`
  AND your dev ngrok URL.
- Save the JSON; copy `client_id` and `client_secret` to `.env`.
- Enable the Google Calendar API in the same project.

```python
# in backend/routers/integrations_google.py
import json
import logging
from fastapi import APIRouter, Depends, HTTPException, Query
from fastapi.responses import RedirectResponse
from google_auth_oauthlib.flow import Flow

from database.integrations import remove, save_tokens
from utils.auth import get_current_user_uid
from utils.settings import get_settings

log = logging.getLogger(__name__)
router = APIRouter(prefix="/v1/integrations/google", tags=["integrations"])

SCOPES = [
    "https://www.googleapis.com/auth/calendar.events",
    "openid", "email", "profile",
]


def _flow(redirect_uri: str) -> Flow:
    s = get_settings()
    return Flow.from_client_config(
        {
            "web": {
                "client_id": __import__("os").environ["GOOGLE_OAUTH_CLIENT_ID"],
                "client_secret": __import__("os").environ["GOOGLE_OAUTH_CLIENT_SECRET"],
                "redirect_uris": [redirect_uri],
                "auth_uri": "https://accounts.google.com/o/oauth2/auth",
                "token_uri": "https://oauth2.googleapis.com/token",
            }
        },
        scopes=SCOPES,
        redirect_uri=redirect_uri,
    )


@router.get("/start")
def start(uid: str = Depends(get_current_user_uid)):
    redirect_uri = f"{get_settings().base_api_url}/v1/integrations/google/callback"
    flow = _flow(redirect_uri)
    url, state = flow.authorization_url(
        access_type="offline", prompt="consent", state=uid,
    )
    return {"url": url}


@router.get("/callback")
def callback(state: str = Query(...), code: str = Query(...)):
    redirect_uri = f"{get_settings().base_api_url}/v1/integrations/google/callback"
    flow = _flow(redirect_uri)
    flow.fetch_token(code=code)
    save_tokens(state, "google", json.loads(flow.credentials.to_json()))
    # Mobile app captures this redirect and closes the webview.
    return RedirectResponse(url="<<YOUR_BRAND>>://oauth/google/done")


@router.delete("")
def disconnect(uid: str = Depends(get_current_user_uid)):
    remove(uid, "google")
    return {"ok": True}
```

(You'll register a custom URL scheme `<<YOUR_BRAND>>://` in the
mobile app's iOS/Android manifests so the redirect actually opens
your app.)

## 7. Push an action item to Google Calendar

```python
# in backend/utils/integrations/google_calendar.py
import logging
from typing import Optional

from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build

from database.integrations import get_tokens, save_tokens

log = logging.getLogger(__name__)


def _creds(uid: str) -> Optional[Credentials]:
    t = get_tokens(uid, "google")
    if not t:
        return None
    creds = Credentials(
        token=t["token"],
        refresh_token=t.get("refresh_token"),
        client_id=t.get("client_id"),
        client_secret=t.get("client_secret"),
        token_uri="https://oauth2.googleapis.com/token",
        scopes=t.get("scopes"),
    )
    if creds.expired and creds.refresh_token:
        from google.auth.transport.requests import Request
        creds.refresh(Request())
        save_tokens(uid, "google", __import__("json").loads(creds.to_json()))
    return creds


def create_calendar_event(uid: str, title: str, when_iso: str, duration_min: int = 30) -> Optional[str]:
    creds = _creds(uid)
    if not creds:
        return None
    service = build("calendar", "v3", credentials=creds, cache_discovery=False)
    body = {
        "summary": title,
        "start": {"dateTime": when_iso, "timeZone": "UTC"},
        "end": {"dateTime": when_iso, "timeZone": "UTC"},
    }
    ev = service.events().insert(calendarId="primary", body=body).execute()
    return ev.get("id")
```

That `service.events().insert(...)` is a *synchronous* Google API
call. Inside an async route, run it via the executor:

```python
import asyncio
from utils.executors import critical_executor

loop = asyncio.get_running_loop()
event_id = await loop.run_in_executor(critical_executor, create_calendar_event, uid, title, when_iso, 30)
```

You'll need a small `utils/executors.py`:

```python
# in backend/utils/executors.py
from concurrent.futures import ThreadPoolExecutor

critical_executor = ThreadPoolExecutor(max_workers=8, thread_name_prefix="critical")
storage_executor = ThreadPoolExecutor(max_workers=16, thread_name_prefix="storage")
```

## 8. Todoist

Same shape, simpler API:

```python
# in backend/utils/integrations/todoist.py
import httpx
from typing import Optional

from database.integrations import get_tokens

API = "https://api.todoist.com/rest/v2"


async def create_task(uid: str, title: str, due: Optional[str] = None) -> Optional[str]:
    t = get_tokens(uid, "todoist")
    if not t:
        return None
    async with httpx.AsyncClient(timeout=10.0) as c:
        r = await c.post(
            f"{API}/tasks",
            headers={"Authorization": f"Bearer {t['access_token']}"},
            json={"content": title, "due_string": due} if due else {"content": title},
        )
    if r.status_code >= 300:
        return None
    return r.json().get("id")
```

OAuth setup is in Todoist's developer console; same callback dance
as Google.

## 9. Wire integrations into action item creation

In `utils/llm/action_items.py`, after `add(uid, item)`:

```python
import asyncio
from utils.integrations.todoist import create_task as todoist_create
from utils.executors import critical_executor
from utils.integrations.google_calendar import create_calendar_event

async def _push_external(uid: str, item):
    todoist_id = await todoist_create(uid, item.title)
    if todoist_id:
        item.external_ids["todoist"] = str(todoist_id)
    if item.due_at:
        loop = asyncio.get_running_loop()
        gcal_id = await loop.run_in_executor(
            critical_executor, create_calendar_event, uid, item.title,
            item.due_at.isoformat(), 30,
        )
        if gcal_id:
            item.external_ids["google_cal"] = gcal_id
    if item.external_ids:
        from database.action_items import patch
        patch(uid, item.id, {"external_ids": item.external_ids})
```

Call `await _push_external(uid, item)` after `add(uid, item)`.

## 10. Composite indexes

For `action_items` filtered by `completed` and ordered by `created_at`:

```json
{
  "collectionGroup": "action_items",
  "queryScope": "COLLECTION",
  "fields": [
    {"fieldPath": "completed", "order": "ASCENDING"},
    {"fieldPath": "created_at", "order": "DESCENDING"}
  ]
}
```

## 11. Commit

```bash
git add backend
git commit -m "feat(part-12): action items + google/todoist integrations"
git push
```

## What you should have right now

- [ ] Conversations produce action items.
- [ ] `GET /v1/action-items` returns the list.
- [ ] Mobile app can OAuth into Google + Todoist.
- [ ] Action items push to Calendar/Todoist; IDs stored under
  `external_ids`.

---

Next: [Part 13 — Push Notifications](./13-push-notifications.md).
