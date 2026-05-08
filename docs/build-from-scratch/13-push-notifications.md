# Part 13 — Push Notifications

> Goal of this part: backend-initiated push notifications to phones,
> via Firebase Cloud Messaging (FCM). FCM works for both Android and
> iOS (it bridges to APNs internally).

## 1. The two notification flavors

| Kind | Trigger | Example |
|------|---------|---------|
| **Reactive** | Right after some event | "Conversation summary ready." |
| **Proactive** | Cron, ML-driven | "I noticed you mentioned X — set a reminder?" |

For v1 just ship reactive. Proactive comes from a cron job in Part 15.

## 2. Mobile app side (just so you know what's needed)

The Flutter app uses the `firebase_messaging` plugin to:

- Request notification permissions.
- Get an FCM registration token.
- Send the token to the backend.

We covered storing the token in Part 06's `add_push_token` already.
Add a register endpoint:

```python
# in backend/routers/notifications.py
from fastapi import APIRouter, Depends
from pydantic import BaseModel
from typing import Literal

from database.users import add_push_token
from utils.auth import get_current_user_uid

router = APIRouter(prefix="/v1/notifications", tags=["notifications"])


class RegisterTokenBody(BaseModel):
    token: str
    kind: Literal["fcm", "apns"] = "fcm"


@router.post("/register-token")
def register(body: RegisterTokenBody, uid: str = Depends(get_current_user_uid)):
    add_push_token(uid, body.kind, body.token)
    return {"ok": True}
```

## 3. The send helper

```python
# in backend/utils/notifications.py
import logging
from typing import Optional

import firebase_admin.messaging as fcm

from database.users import get_user

log = logging.getLogger(__name__)


def send_push(uid: str, title: str, body: str,
              data: Optional[dict] = None, link: Optional[str] = None) -> int:
    user = get_user(uid)
    if not user:
        return 0
    tokens = list(user.fcm_tokens or [])
    if not tokens:
        return 0
    msg = fcm.MulticastMessage(
        tokens=tokens,
        notification=fcm.Notification(title=title, body=body),
        data={k: str(v) for k, v in (data or {}).items()},
        android=fcm.AndroidConfig(priority="high"),
        apns=fcm.APNSConfig(
            payload=fcm.APNSPayload(aps=fcm.Aps(sound="default", badge=1)),
        ),
    )
    response = fcm.send_each_for_multicast(msg)
    # prune dead tokens
    if response.failure_count:
        from database._client import db
        from google.cloud import firestore as fs
        for tok, resp in zip(tokens, response.responses):
            if not resp.success and getattr(resp.exception, "code", "") == \
               "registration-token-not-registered":
                db.collection("users").document(uid).update(
                    {"fcm_tokens": fs.ArrayRemove([tok])}
                )
    return response.success_count
```

## 4. APNs note

FCM bridges to APNs *for you*, but you still need to:

1. Generate an **APNs auth key** at
   https://developer.apple.com/account/resources/authkeys/list and
   download the `.p8` file.
2. In Firebase Console → Project Settings → Cloud Messaging → "Apple
   app configuration" → upload the `.p8`, plus your team ID and key
   ID.
3. In your iOS app, add **Background Modes → Remote notifications**
   in Xcode and request permission via `firebase_messaging`.

After that, FCM tokens from iOS work as transparently as Android.

## 5. Sending on conversation completion

In `utils/llm/post_process.py`, after marking the conversation
`completed`:

```python
from utils.notifications import send_push
send_push(
    uid,
    title=data.get("title") or "Conversation ready",
    body=(data.get("summary") or "")[:140],
    data={"type": "conversation.completed", "conversation_id": cid},
    link=f"<<YOUR_BRAND>>://conversations/{cid}",
)
```

The mobile app reads `data.type` from the notification payload to
know what screen to open.

## 6. Notification model + history

So users can see what was sent (and resend if needed):

```python
# in backend/models/notification.py
from datetime import datetime
from pydantic import BaseModel, Field
import uuid


class StoredNotification(BaseModel):
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    uid: str
    title: str
    body: str
    data: dict = {}
    sent_at: datetime = Field(default_factory=datetime.utcnow)
    read: bool = False
```

```python
# in backend/database/notifications.py
from database._client import db
from models.notification import StoredNotification


def append(uid: str, n: StoredNotification) -> None:
    db.collection("users").document(uid).collection("notifications") \
        .document(n.id).set(n.model_dump(mode="json"))
```

Wrap your `send_push` to also store a copy.

## 7. In-app rate limiting

Don't spam users. A practical rule:

- Max **1** "conversation.completed" notification per 90 seconds per
  user. Use Redis:

```python
import time
from database import redis_db

def can_notify(uid: str, kind: str, cooldown_seconds: int = 90) -> bool:
    key = f"notif:{uid}:{kind}"
    if redis_db.get(key) is not None:
        return False
    redis_db.set(key, b"1", ex=cooldown_seconds)
    return True
```

Wrap `send_push(...)` calls with `if can_notify(uid, "conversation.completed"): ...`.

## 8. Test it

From the running backend:

```bash
http POST :8080/v1/notifications/register-token \
  Authorization:"Bearer ${ADMIN_KEY}uid_test_42" token=<FAKE_FCM_TOKEN>
```

Then trigger a fake send via a one-shot script:

```python
# in backend/scripts/test_push.py
from utils.notifications import send_push
print(send_push("uid_test_42", "Hello", "From your backend",
                data={"type": "test"}))
```

Run on a test phone signed in to the same Firebase project — you
should see a notification pop up.

## 9. Commit

```bash
git add backend
git commit -m "feat(part-13): push notifications via fcm"
git push
```

## What you should have right now

- [ ] Mobile app registers FCM tokens with the backend.
- [ ] `send_push(uid, ...)` works end-to-end on a test device.
- [ ] On conversation completion, the user gets a push.
- [ ] Per-user, per-kind cooldown prevents spam.

---

Next: [Part 14 — External Device (BLE Wearable)](./14-external-device-ble.md).
