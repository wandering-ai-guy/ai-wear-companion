# Part 06 — Users, Profiles, Onboarding

> Goal of this part: a complete `users` domain — model, database
> module, router, and onboarding flow — so the app can collect
> language, timezone, name, and consent before recording starts.

## 1. The user document shape

The `users/{uid}` doc is the root of everything we know about a user.
Keep it small (Firestore charges per read), and push large data into
subcollections (memories, conversations, etc.). Decide your schema
*before* you start writing anywhere; renaming top-level fields later
is painful.

The minimal schema:

```python
# in backend/models/user.py
from datetime import datetime
from typing import Literal, Optional

from pydantic import BaseModel, Field


SubscriptionTier = Literal["free", "pro", "team"]


class UserProfile(BaseModel):
    uid: str
    email: Optional[str] = None
    display_name: Optional[str] = None
    avatar_url: Optional[str] = None
    language: str = "en"  # ISO 639-1
    timezone: str = "UTC"  # IANA TZ
    onboarding_completed: bool = False
    subscription_tier: SubscriptionTier = "free"
    speech_profile_recorded: bool = False
    consent_recording: bool = False
    consent_terms: bool = False
    fcm_tokens: list[str] = Field(default_factory=list)
    apns_tokens: list[str] = Field(default_factory=list)
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)


class UserPatch(BaseModel):
    """Anything a client is allowed to PATCH."""
    display_name: Optional[str] = None
    language: Optional[str] = None
    timezone: Optional[str] = None
    onboarding_completed: Optional[bool] = None
    consent_recording: Optional[bool] = None
    consent_terms: Optional[bool] = None
```

## 2. The database module

```python
# in backend/database/users.py
from datetime import datetime
from typing import Optional

from google.cloud import firestore

from database._client import db
from models.user import UserProfile


_COL = "users"


def get_user(uid: str) -> Optional[UserProfile]:
    snap = db.collection(_COL).document(uid).get()
    if not snap.exists:
        return None
    return UserProfile(**snap.to_dict())


def upsert_user(profile: UserProfile) -> UserProfile:
    profile.updated_at = datetime.utcnow()
    db.collection(_COL).document(profile.uid).set(profile.model_dump(mode="json"), merge=True)
    return profile


def patch_user(uid: str, patch: dict) -> UserProfile:
    patch = {k: v for k, v in patch.items() if v is not None}
    patch["updated_at"] = datetime.utcnow().isoformat()
    db.collection(_COL).document(uid).set(patch, merge=True)
    snap = db.collection(_COL).document(uid).get()
    return UserProfile(**snap.to_dict())


def add_push_token(uid: str, kind: str, token: str) -> None:
    field = "fcm_tokens" if kind == "fcm" else "apns_tokens"
    db.collection(_COL).document(uid).update(
        {field: firestore.ArrayUnion([token])}
    )
```

Why a separate database module? It is the **only** place that knows
about Firestore. Routers and utils don't import the Firestore client
directly. This makes future migrations (e.g. moving to Postgres) a
matter of rewriting these files.

## 3. The router

```python
# in backend/routers/users.py
from fastapi import APIRouter, Depends, HTTPException

from database.users import get_user, patch_user, upsert_user
from models.user import UserPatch, UserProfile
from utils.auth import get_current_user_uid


router = APIRouter(prefix="/v1/users", tags=["users"])


@router.get("/me", response_model=UserProfile)
def read_me(uid: str = Depends(get_current_user_uid)) -> UserProfile:
    user = get_user(uid)
    if user is None:
        user = upsert_user(UserProfile(uid=uid))
    return user


@router.patch("/me", response_model=UserProfile)
def update_me(patch: UserPatch, uid: str = Depends(get_current_user_uid)) -> UserProfile:
    return patch_user(uid, patch.model_dump(exclude_unset=True))


@router.post("/me/onboarding/complete", response_model=UserProfile)
def complete_onboarding(uid: str = Depends(get_current_user_uid)) -> UserProfile:
    user = get_user(uid)
    if user is None:
        raise HTTPException(404, "user not found")
    if not user.consent_recording or not user.consent_terms:
        raise HTTPException(400, "consent required before onboarding can complete")
    return patch_user(uid, {"onboarding_completed": True})


@router.delete("/me")
def delete_me(uid: str = Depends(get_current_user_uid)):
    """Right-to-be-forgotten. Cascades through every subcollection."""
    from database.users import db, _COL  # local: import only here to avoid leaks
    # Future parts add more cascades.
    db.collection(_COL).document(uid).delete()
    return {"deleted": True}
```

Wait — the second-to-last line breaks our import rule (it imports the
module-level `db` from `database/users.py`). Fix it: in
`database/users.py`, add a top-level helper:

```python
# in backend/database/users.py
def cascade_delete(uid: str) -> None:
    """Delete root user doc + all known subcollections."""
    user_ref = db.collection(_COL).document(uid)
    # Add subcollection deletion as you add features:
    for sub in ["conversations", "memories", "action_items", "chat_messages"]:
        for snap in user_ref.collection(sub).stream():
            snap.reference.delete()
    user_ref.delete()
```

Then in the router:

```python
@router.delete("/me")
def delete_me(uid: str = Depends(get_current_user_uid)):
    from database.users import cascade_delete
    cascade_delete(uid)
    return {"deleted": True}
```

Register in `main.py`:

```python
from routers import health, me, users
app.include_router(users.router)
```

(You can delete the old `routers/me.py` from Part 04 — `users.py`
supersedes it.)

## 4. Onboarding flow (server contract)

Your mobile app will call the backend in this order on first launch:

```
POST  /v1/auth/signup-or-login          ← (handled by Firebase SDK on the client)
GET   /v1/users/me                      ← creates user doc on first hit
PATCH /v1/users/me                      ← language, timezone, display_name
PATCH /v1/users/me                      ← consent_recording, consent_terms
POST  /v1/users/me/onboarding/complete  ← flips onboarding_completed
```

After that, the app shows the home screen.

## 5. Consent matters

You **must** record explicit consent for:

- Continuous audio recording (in many US states this is mandatory;
  in EU/UK/AU it is non-negotiable).
- Terms of Service / Privacy Policy.

Don't fake-default these to `true`. Set them only after the user
has actually scrolled and tapped a checkbox in the mobile app.
Save the timestamp + version of the document they agreed to.

Update the model:

```python
class UserProfile(BaseModel):
    ...
    consent_recording_at: Optional[datetime] = None
    consent_terms_at: Optional[datetime] = None
    consent_terms_version: Optional[str] = None
```

And in the patch, when the field becomes `True`, stamp a timestamp:

```python
def patch_user(uid: str, patch: dict) -> UserProfile:
    if patch.get("consent_recording") is True:
        patch.setdefault("consent_recording_at", datetime.utcnow().isoformat())
    if patch.get("consent_terms") is True:
        patch.setdefault("consent_terms_at", datetime.utcnow().isoformat())
        patch.setdefault("consent_terms_version", "v1.0")
    ...
```

## 6. People & contacts (optional)

The mobile app lets users tag who's in a conversation. We model
"people" as a subcollection of `users/{uid}/people/{person_id}`.

```python
# in backend/models/person.py
from typing import Optional
from pydantic import BaseModel, Field
import uuid


class Person(BaseModel):
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    name: str
    relationship: Optional[str] = None  # "friend", "colleague", "family"
    avatar_url: Optional[str] = None
    speaker_embedding_id: Optional[str] = None
```

```python
# in backend/database/people.py
from typing import List
from database._client import db
from models.person import Person

_USERS = "users"
_PEOPLE = "people"


def list_people(uid: str) -> List[Person]:
    return [Person(**s.to_dict()) for s in
            db.collection(_USERS).document(uid).collection(_PEOPLE).stream()]


def upsert_person(uid: str, p: Person) -> Person:
    db.collection(_USERS).document(uid).collection(_PEOPLE).document(p.id) \
        .set(p.model_dump(mode="json"), merge=True)
    return p
```

You'll add a `/v1/people` router with the same shape as `users.py`.

## 7. Tests

Create `backend/tests/unit/test_users_router.py`:

```python
from fastapi.testclient import TestClient
import pytest

import os
os.environ.setdefault("ADMIN_KEY", "k")
os.environ.setdefault("ENCRYPTION_SECRET", "x" * 32)

from main import app  # noqa: E402

UID = "test-user-1"
HEADERS = {"Authorization": f"Bearer k{UID}"}


@pytest.fixture
def client():
    with TestClient(app) as c:
        yield c


def test_get_me_creates_user(client, monkeypatch):
    # monkeypatch the firestore client to an in-memory dict
    from database import users as udb
    fake = {}
    class FakeDoc:
        def __init__(self, key): self.key = key
        @property
        def exists(self): return self.key in fake
        def to_dict(self): return fake.get(self.key)
        def get(self): return self
        def set(self, v, merge=False):
            fake[self.key] = (fake.get(self.key) or {}) | v if merge else v
        def update(self, v): fake[self.key].update(v)
    class FakeCol:
        def document(self, k): return FakeDoc(k)
    monkeypatch.setattr(udb.db, "collection", lambda name: FakeCol())
    r = client.get("/v1/users/me", headers=HEADERS)
    assert r.status_code == 200, r.text
    assert r.json()["uid"] == UID
```

You'll grow this test file as you add fields. Run:

```bash
cd backend
pip install pytest pytest-asyncio
pytest -q
```

## 8. Commit

```bash
git add backend
git commit -m "feat(part-06): users, onboarding, people"
git push
```

## What you should have right now

- [ ] `GET /v1/users/me` returns a profile, creating it on first call.
- [ ] `PATCH /v1/users/me` updates language, timezone, etc.
- [ ] `POST /v1/users/me/onboarding/complete` requires consent.
- [ ] `DELETE /v1/users/me` cascades.
- [ ] (optional) `/v1/people` CRUD lives.
- [ ] At least one passing pytest unit test for the users router.

---

Next: [Part 07 — Live Audio Streaming + STT](./07-audio-streaming-stt.md).
