# Part 17 — Privacy, Encryption, Log Sanitization

> Goal of this part: take the encryption helpers we built in Part 05
> and apply them everywhere user content lives, so a Firestore export
> is gibberish without the master key, and your logs never contain
> raw transcripts.

This is non-negotiable for a product that records conversations.

## 1. Threat model

Decide explicitly what you're defending against:

| Threat | Mitigation |
|--------|------------|
| Lost laptop with `google-credentials.json` | Use Secret Manager + workload identity in prod (Part 19); short-lived dev keys; revoke on offboarding. |
| Database export leaks (e.g. SQL injection on a sister service) | Per-user app-level encryption (this part). |
| One user reading another's data | Pinecone namespace per uid; Firestore rules; `uid` filter on every query. |
| Accidental log of a transcript | `sanitize()` / `sanitize_pii()` in every log call. |
| Stolen Firebase ID token | 1-hour expiry, refresh on backend; `check_revoked=True` in production. |
| Engineer / DBA reading raw data | Per-user keys derived from a secret only in Secret Manager; auditable access. |

Document this in `docs/THREAT_MODEL.md` for your team. It also shows
up in SOC 2 / HIPAA audits.

## 2. Encrypt at write — segments

Update `database/segments.py`:

```python
# in backend/database/segments.py
from typing import List
from database._client import db
from models.segment import TranscriptSegment
from utils.encryption import encrypt_for, decrypt_for

_USERS = "users"; _CONVS = "conversations"; _SEGS = "segments"


def _encrypt_seg(uid: str, seg: TranscriptSegment) -> dict:
    d = seg.model_dump(mode="json")
    if d.get("text"):
        d["text_enc"] = encrypt_for(uid, d.pop("text"))
    return d


def _decrypt_seg(uid: str, raw: dict) -> TranscriptSegment:
    if "text_enc" in raw:
        raw = {**raw, "text": decrypt_for(uid, raw.pop("text_enc"))}
    return TranscriptSegment(**raw)


def append_segment(uid: str, seg: TranscriptSegment) -> None:
    db.collection(_USERS).document(uid).collection(_CONVS) \
        .document(seg.conversation_id).collection(_SEGS).document(seg.id) \
        .set(_encrypt_seg(uid, seg))


def list_segments(uid: str, conversation_id: str) -> List[TranscriptSegment]:
    snaps = db.collection(_USERS).document(uid).collection(_CONVS) \
        .document(conversation_id).collection(_SEGS) \
        .order_by("start_ms").stream()
    return [_decrypt_seg(uid, s.to_dict()) for s in snaps]
```

Now Firestore stores `text_enc` (a base64 ciphertext) instead of
`text`. Direct exports are useless without your key.

Apply the same pattern to:

- `database/conversations.py` → encrypt `title` and `summary`.
- `database/memories.py` → encrypt `text`.
- `database/chat.py` → encrypt `content`.

Each domain module should expose **only** decrypted models to its
callers. Routers and tools never see ciphertext.

## 3. Encrypt at write — Pinecone metadata

We stored a 512-char `text` snippet in Pinecone metadata in Part 10.
That's *not* encrypted — and Pinecone metadata is what shows up if
their console is compromised. Two options:

- **Don't store text in metadata** at all. Only store the Firestore
  doc ID; resolve it after retrieval. This is what production-grade
  systems do.
- **Encrypt the metadata text** the same way (it'll show up as a
  base64 blob in their console).

Use the first option. Update Part 10's `vector_db.upsert(...)` calls
to drop the `text` field from metadata, and after Pinecone returns
matches, look up the actual texts from Firestore.

## 4. Encrypt at write — chat tool results

When the chat orchestrator's tool calls return memories or
conversations, **only the decrypted text** crosses the LLM boundary
(by definition — the LLM cannot read ciphertext). This is fine, but:

- Set `LANGSMITH_TRACING=false` unless you trust LangSmith with
  user content. If you enable it, document that in your privacy
  policy.
- Set OpenAI organization-level **data sharing OFF** (Console →
  Privacy). Without this, OpenAI may use your prompts to train.

## 5. Sanitize all logs

Any time you log a payload that *could* be user content, wrap it:

```python
from utils.log_sanitizer import sanitize, sanitize_pii

log.info("deepgram.response %s", sanitize(text)[:512])
log.info("user_message %s", sanitize_pii(body.message))
```

Add a pre-commit lint that fails when raw `response.text` shows up
in logs. Skeleton:

```bash
# in scripts/lint_log_calls.sh
#!/usr/bin/env bash
set -euo pipefail

if rg -n 'log\.(info|warning|error)\([^,)]*response\.text' backend/ pusher/ ; then
  echo "ERROR: raw response.text in a log call. Wrap it in sanitize()."
  exit 1
fi
```

Add it to `.pre-commit-config.yaml` as a local hook.

## 6. PII in URLs

Don't put `email=`, `phone=`, etc. in query strings — URLs end up in
nginx access logs. Same goes for IDs that are themselves PII (e.g.
`user_phone=+1...`). Always use POST bodies for those.

## 7. Right to be forgotten

A user must be able to:

- **Export** all their data (`GET /v1/users/me/export` — returns a
  signed URL to a JSON dump).
- **Delete** all their data (already implemented via
  `DELETE /v1/users/me`).

For deletion, also fan out to:

- Pinecone: `vector_db.delete_for_user(uid)`.
- Typesense: documents filter by `uid`.
- GCS: delete all blobs under `<uid>/`.
- Stripe: cancel any active subscriptions.
- Send a final confirmation email.

Add a `database/users.py:cascade_delete` that does *all* of the above.

## 8. Encryption secret rotation

Eventually you'll want to rotate the master `ENCRYPTION_SECRET`.
The trick is to support two secrets at once:

```python
# in backend/utils/encryption.py
_AAD = b"<<YOUR_BRAND>>:v1"
_SECRETS = []  # populated from env: PRIMARY first, OLD secrets after

def _user_key(uid: str, secret: bytes) -> bytes:
    return HKDF(...).derive(secret)

def decrypt_for(uid: str, blob: str) -> str:
    raw = base64.b64decode(blob)
    nonce, ct = raw[:12], raw[12:]
    last_err = None
    for s in _SECRETS:
        try:
            return AESGCM(_user_key(uid, s)).decrypt(nonce, ct, _AAD).decode()
        except Exception as e:
            last_err = e
    raise last_err
```

Store secrets like `ENCRYPTION_SECRET=primary,old1,old2`. New writes
always use `primary`; reads try every key. Rotate primary, leave
old keys around for 90 days, then drop them.

## 9. Audit logs

For every "sensitive" action (delete, export, integration token
read), append a row to `users/{uid}/audit/{ts}` with `actor_uid`,
`endpoint`, `ip`, and a rough timestamp. This is invaluable when a
user emails you "what did your system do at 3 AM."

## 10. Compliance markers (when ready)

- **Privacy Policy** + **Terms of Service** — link them on the
  signup screen and version them. Save the version the user
  consented to (Part 06).
- **DPA** (Data Processing Agreement) for B2B users.
- **GDPR** SAR/erasure responses within 30 days.
- **CCPA** "Do not sell my info" toggle (you don't sell anyway, but
  the toggle has to exist for CA users).
- **HIPAA**: only relevant if you market to healthcare. Don't.
- **SOC 2**: when you have ≥3 enterprise prospects. Use Vanta or
  Drata.

These are all later concerns; what matters today is *don't paint
yourself into a corner that makes them impossible later.*
The encryption design above doesn't.

## 11. Commit

```bash
git add backend scripts
git commit -m "feat(part-17): encrypt-at-rest segments/conv/memories + log lint"
git push
```

## What you should have right now

- [ ] Firestore docs for segments/conversations/memories/chat have
  `_enc` fields, not raw text.
- [ ] Pinecone metadata no longer contains user text.
- [ ] Logs are routed through `sanitize()`/`sanitize_pii()`.
- [ ] `DELETE /v1/users/me` cascades through Firestore, Pinecone,
  Typesense, GCS, Stripe.
- [ ] Encryption supports multiple keys for rotation.

---

Next: [Part 18 — Testing Strategy](./18-testing.md).
