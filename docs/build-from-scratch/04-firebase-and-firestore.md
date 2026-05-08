# Part 04 — Firebase Auth & Firestore

> Goal of this part: a real user can sign in from a tiny test client,
> the backend verifies their ID token, and a `users/{uid}` document
> appears in Firestore on first sign-in.

## 1. Concepts you have to internalize

There are *three* identities at play, and confusing them costs hours:

- **User identity (`uid`).** A 28-char string Firebase assigns when
  a user signs up. Permanent. We use it as the document key everywhere
  in Firestore. Never email.
- **ID token.** A short-lived (1h) JWT the *client* gets after
  signing in. The phone sends it as `Authorization: Bearer <token>`
  on every request. The backend verifies it and pulls the `uid` out.
- **Service-account credentials.** A JSON file your *backend* uses to
  talk to Firestore/Storage as itself. Never sent to clients.

## 2. Generate a backend service-account key

```bash
gcloud iam service-accounts create backend-sa \
  --display-name="Backend service account" \
  --project=<<YOUR_GCP_PROJECT_ID>>

gcloud projects add-iam-policy-binding <<YOUR_GCP_PROJECT_ID>> \
  --member="serviceAccount:backend-sa@<<YOUR_GCP_PROJECT_ID>>.iam.gserviceaccount.com" \
  --role="roles/datastore.user"

gcloud projects add-iam-policy-binding <<YOUR_GCP_PROJECT_ID>> \
  --member="serviceAccount:backend-sa@<<YOUR_GCP_PROJECT_ID>>.iam.gserviceaccount.com" \
  --role="roles/storage.admin"

gcloud projects add-iam-policy-binding <<YOUR_GCP_PROJECT_ID>> \
  --member="serviceAccount:backend-sa@<<YOUR_GCP_PROJECT_ID>>.iam.gserviceaccount.com" \
  --role="roles/firebaseauth.admin"

gcloud iam service-accounts keys create google-credentials.json \
  --iam-account=backend-sa@<<YOUR_GCP_PROJECT_ID>>.iam.gserviceaccount.com
```

Move the JSON to `backend/google-credentials.json` (already gitignored).
Update `.env`:

```
GOOGLE_APPLICATION_CREDENTIALS=google-credentials.json
GCP_PROJECT_ID=<<YOUR_GCP_PROJECT_ID>>
```

**Treat this file like a password.** Anyone with it can read/write your
production database. Never commit it. In Part 19 we'll move secrets to
Google Secret Manager.

## 3. Add Firebase Admin SDK

Append to `backend/requirements.txt`:

```
firebase-admin==6.5.0
google-cloud-firestore==2.20.0
google-cloud-storage==2.18.0
```

```bash
pip install -r requirements.txt
```

## 4. The Firestore client singleton

Create `backend/database/_client.py`:

```python
# in backend/database/_client.py
import json
import os

import firebase_admin
from firebase_admin import credentials
from google.cloud import firestore


def _initialize_firebase_app() -> None:
    if firebase_admin._apps:
        return
    sa_json = os.environ.get("SERVICE_ACCOUNT_JSON")
    if sa_json:
        creds = credentials.Certificate(json.loads(sa_json))
        firebase_admin.initialize_app(creds)
    else:
        firebase_admin.initialize_app()  # uses GOOGLE_APPLICATION_CREDENTIALS


_initialize_firebase_app()

db: firestore.Client = firestore.Client()


def doc_id_from_seed(seed: str) -> str:
    """Stable Firestore-safe ID derived from any string."""
    import hashlib
    return hashlib.sha256(seed.encode()).hexdigest()[:20]
```

Why two ways to load credentials? In *local dev*, `google-credentials.json`
is fine. In *production*, you'll inject the same JSON as a single
`SERVICE_ACCOUNT_JSON` environment variable (see Part 19 — Cloud Run /
Kubernetes secrets work that way).

## 5. The auth dependency

Create `backend/utils/auth.py`:

```python
# in backend/utils/auth.py
from typing import Annotated

import firebase_admin.auth
from fastapi import Depends, Header, HTTPException, status

from utils.settings import get_settings


def get_current_user_uid(
    authorization: Annotated[str | None, Header()] = None,
) -> str:
    if not authorization or not authorization.lower().startswith("bearer "):
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "missing bearer token")

    token = authorization.split(" ", 1)[1].strip()

    # Local dev backdoor: token = ADMIN_KEY + "<uid>"
    s = get_settings()
    if s.admin_key and token.startswith(s.admin_key):
        return token[len(s.admin_key):]  # noqa: E501

    try:
        decoded = firebase_admin.auth.verify_id_token(token, check_revoked=False)
        return decoded["uid"]
    except Exception as e:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, f"invalid token: {e}")
```

The `ADMIN_KEY` backdoor matters: it lets you `curl` your own API
without juggling Firebase tokens. Set `ADMIN_KEY=abc123` in `.env`,
then send `Authorization: Bearer abc123uid_test_42` and the backend
acts like the user `uid_test_42` is signed in.

Never enable this on a production deployment. We'll gate it on `env`
in Part 19.

## 6. A protected endpoint

Add a router. Create `backend/routers/me.py`:

```python
# in backend/routers/me.py
from fastapi import APIRouter, Depends

from database._client import db
from utils.auth import get_current_user_uid

router = APIRouter(prefix="/v1", tags=["me"])


@router.get("/me")
def get_me(uid: str = Depends(get_current_user_uid)):
    snap = db.collection("users").document(uid).get()
    data = snap.to_dict() if snap.exists else None
    if data is None:
        # Lazy-create on first call.
        data = {
            "uid": uid,
            "created_at": __import__("datetime").datetime.utcnow().isoformat(),
            "language": "en",
            "subscription_tier": "free",
        }
        db.collection("users").document(uid).set(data)
    return data
```

Register it in `main.py`:

```python
# in backend/main.py — add to the imports block
from routers import health, me

app.include_router(health.router)
app.include_router(me.router)
```

## 7. Firestore security rules

Open `firebase.json` (let `firebase init firestore` create one if it
doesn't exist), and create `firestore.rules`:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Users can read/write only their own root user doc, and nothing else.
    // The backend (with admin creds) is exempt from these rules.
    match /users/{uid} {
      allow read, update: if request.auth != null && request.auth.uid == uid;
      allow create: if request.auth != null && request.auth.uid == uid;
      allow delete: if false;
    }

    // Block all direct client access to subcollections — those go through the API.
    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

Deploy them:

```bash
firebase deploy --only firestore:rules
```

The reasoning: clients should *never* talk to Firestore directly,
because we encrypt + permission everything in the backend. Rules above
are a final safety net against a leaked key.

## 8. Required composite indexes

Some collections need composite indexes (sorting + filtering on
multiple fields). Define them in `firestore.indexes.json`:

```json
{
  "indexes": [
    {
      "collectionGroup": "dev_api_keys",
      "queryScope": "COLLECTION",
      "fields": [
        {"fieldPath": "user_id", "order": "ASCENDING"},
        {"fieldPath": "created_at", "order": "DESCENDING"}
      ]
    }
  ],
  "fieldOverrides": []
}
```

Deploy:

```bash
firebase deploy --only firestore:indexes
```

You'll add more rows here as future parts introduce new queries. The
error message you'll see if you forget is "The query requires an index"
with a clickable link — Firebase actually tells you exactly what's
missing.

## 9. Test it

With your backend running:

```bash
# fake an authenticated call using the dev backdoor
curl -H "Authorization: Bearer ${ADMIN_KEY}uid_test_42" \
  https://<<YOUR_BRAND>>-dev.ngrok-free.app/v1/me | jq
```

You should see the user document return. Open the Firebase Console →
Firestore → `users` collection. You should see `uid_test_42`.

## 10. Optional: a tiny HTML page that signs in for real

Create `backend/scripts/test-auth.html` (for local testing only):

```html
<!doctype html>
<html><body>
<button id="signin">Sign in with Google</button>
<pre id="out"></pre>
<script type="module">
  import { initializeApp } from "https://www.gstatic.com/firebasejs/10.13.0/firebase-app.js";
  import { getAuth, GoogleAuthProvider, signInWithPopup }
    from "https://www.gstatic.com/firebasejs/10.13.0/firebase-auth.js";

  const app = initializeApp({
    apiKey: "<<FIREBASE_API_KEY>>",
    authDomain: "<<FIREBASE_AUTH_DOMAIN>>",
    projectId: "<<GCP_PROJECT_ID>>",
  });
  const auth = getAuth(app);
  document.getElementById("signin").onclick = async () => {
    const cred = await signInWithPopup(auth, new GoogleAuthProvider());
    const token = await cred.user.getIdToken();
    const r = await fetch("<<BASE_API_URL>>/v1/me", {
      headers: { Authorization: "Bearer " + token },
    });
    document.getElementById("out").textContent = await r.text();
  };
</script>
</body></html>
```

Open it via `python3 -m http.server` from the same directory, click the
button, sign in, and confirm you see your real user document.

## 11. Commit

```bash
git add backend firestore.rules firestore.indexes.json firebase.json
git commit -m "feat(part-04): firebase auth + firestore wired"
git push
```

## What you should have right now

- [ ] A `backend-sa` service account JSON, gitignored.
- [ ] `from database._client import db` works in a Python REPL.
- [ ] `GET /v1/me` returns 401 without a token, 200 with one.
- [ ] First call creates `users/{uid}` in Firestore.
- [ ] Security rules deployed; direct client access blocked.

---

Next: [Part 05 — Redis, Encryption, Shared Utilities](./05-redis-encryption-utils.md).
