# Part 05 — Redis, Encryption, Shared Utilities

> Goal of this part: a Redis cache, AES-256-GCM per-user encryption,
> a shared async HTTP client pool, a sanitizing logger, and a
> rate-limiting middleware. These are the plumbing that every later
> part depends on.

## 1. Redis (Upstash)

In your Upstash console, create a free database:

- Region: same as your GCP region.
- TLS: enabled.
- Eviction: `noeviction` (we use Redis as cache + locks; we don't want
  data silently disappearing).

Copy the connection details into `.env`:

```
REDIS_DB_HOST=...upstash.io
REDIS_DB_PORT=6379
REDIS_DB_PASSWORD=...
```

Append to `requirements.txt`:

```
redis==5.0.8
```

Create `backend/database/redis_db.py`:

```python
# in backend/database/redis_db.py
import logging
from contextlib import contextmanager
from typing import Optional

import redis as redis_lib

from utils.settings import get_settings

log = logging.getLogger(__name__)
_client: Optional[redis_lib.Redis] = None


def _get_client() -> Optional[redis_lib.Redis]:
    global _client
    if _client is not None:
        return _client
    s = get_settings()
    if not s.redis_db_host:
        return None
    _client = redis_lib.Redis(
        host=s.redis_db_host,
        port=s.redis_db_port or 6379,
        password=s.redis_db_password or None,
        ssl=True,
        socket_timeout=2,
        socket_connect_timeout=2,
        decode_responses=False,
    )
    return _client


def get(key: str) -> Optional[bytes]:
    """Fail-open get — returns None on any Redis error."""
    try:
        c = _get_client()
        return c.get(key) if c else None
    except Exception as e:
        log.warning("redis.get failed: %s", e)
        return None


def set(key: str, value: bytes | str, ex: Optional[int] = None) -> bool:
    try:
        c = _get_client()
        if not c:
            return False
        c.set(key, value, ex=ex)
        return True
    except Exception as e:
        log.warning("redis.set failed: %s", e)
        return False


@contextmanager
def lock(key: str, ttl_seconds: int = 30):
    """Best-effort distributed lock. Yields True if acquired."""
    c = _get_client()
    acquired = False
    if c:
        try:
            acquired = bool(c.set(f"lock:{key}", b"1", nx=True, ex=ttl_seconds))
        except Exception as e:
            log.warning("redis.lock failed: %s", e)
    try:
        yield acquired
    finally:
        if acquired and c:
            try:
                c.delete(f"lock:{key}")
            except Exception:
                pass
```

The **fail-open** rule: if Redis is unreachable, every call returns
`None`/`False` and the request keeps going. Redis is a performance and
deduping aid, never a hard dependency for correctness. This single
property has saved more outages than any other design choice in the
reference codebase.

## 2. Per-user encryption (AES-256-GCM)

We store transcripts and chat messages in Firestore. Even though
Firestore is encrypted at rest by Google, we add a *per-user* layer
on top so:

- a leaked Firestore export is gibberish without the secret,
- different users can't read each other's data even if a query goes
  wrong.

Generate a master secret once, store it in `.env` and in Secret Manager:

```bash
echo "ENCRYPTION_SECRET=$(openssl rand -base64 48)" >> backend/.env
```

Append to `requirements.txt`:

```
cryptography==46.0.5
```

Create `backend/utils/encryption.py`:

```python
# in backend/utils/encryption.py
import base64
import os

from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
from cryptography.hazmat.primitives.kdf.hkdf import HKDF

from utils.settings import get_settings

_AAD = b"<<YOUR_BRAND>>:v1"


def _user_key(uid: str) -> bytes:
    """Derive a 32-byte AES key from the master secret + uid using HKDF."""
    secret = get_settings().encryption_secret.encode()
    if not secret:
        raise RuntimeError("ENCRYPTION_SECRET not set")
    return HKDF(
        algorithm=hashes.SHA256(),
        length=32,
        salt=b"<<YOUR_BRAND>>:salt",
        info=uid.encode(),
    ).derive(secret)


def encrypt_for(uid: str, plaintext: str) -> str:
    """Returns base64(nonce || ciphertext || tag)."""
    if plaintext is None:
        return None  # type: ignore
    key = _user_key(uid)
    nonce = os.urandom(12)
    ct = AESGCM(key).encrypt(nonce, plaintext.encode("utf-8"), _AAD)
    return base64.b64encode(nonce + ct).decode("ascii")


def decrypt_for(uid: str, blob: str) -> str:
    if blob is None:
        return None  # type: ignore
    raw = base64.b64decode(blob.encode("ascii"))
    nonce, ct = raw[:12], raw[12:]
    return AESGCM(_user_key(uid)).decrypt(nonce, ct, _AAD).decode("utf-8")
```

The HKDF step is what gives each user a unique key derived from the
single master secret. If a single user ever needs to be cryptoshredded,
you rotate that user's HKDF salt entry in a separate table; we'll add
that in Part 17.

## 3. The shared async HTTP client

Many services we'll integrate (Deepgram, Pinecone, Gemini's webhook
clients, your own webhooks) need an `httpx.AsyncClient`. Creating one
per request is wasteful — we want a shared, pooled, semaphore-bounded
client.

Append to `requirements.txt`:

```
httpx[http2]==0.28.0
```

Create `backend/utils/http_client.py`:

```python
# in backend/utils/http_client.py
import asyncio
from typing import Dict, Tuple

import httpx

_clients: Dict[str, httpx.AsyncClient] = {}
_semaphores: Dict[Tuple[int, str], asyncio.Semaphore] = {}


def _make_client(timeout: httpx.Timeout) -> httpx.AsyncClient:
    return httpx.AsyncClient(
        timeout=timeout,
        http2=True,
        limits=httpx.Limits(max_connections=200, max_keepalive_connections=50),
    )


def get_webhook_client() -> httpx.AsyncClient:
    if "webhook" not in _clients:
        _clients["webhook"] = _make_client(httpx.Timeout(30.0, connect=2.0))
    return _clients["webhook"]


def get_stt_client() -> httpx.AsyncClient:
    if "stt" not in _clients:
        _clients["stt"] = _make_client(httpx.Timeout(60.0, connect=3.0))
    return _clients["stt"]


def get_semaphore(name: str, limit: int) -> asyncio.Semaphore:
    """Event-loop-bound semaphore. Different loops (e.g. tests) get their own."""
    loop_id = id(asyncio.get_event_loop())
    key = (loop_id, name)
    if key not in _semaphores:
        _semaphores[key] = asyncio.Semaphore(limit)
    return _semaphores[key]


async def close_all_clients() -> None:
    for c in list(_clients.values()):
        await c.aclose()
    _clients.clear()
```

Wire it into `main.py`:

```python
# in backend/main.py — inside lifespan()
from utils.http_client import close_all_clients

@asynccontextmanager
async def lifespan(app: FastAPI):
    log.info("startup", env=settings.env)
    yield
    await close_all_clients()
    log.info("shutdown")
```

The "never use `requests.get` in async code" rule lives or dies on
this client. Block this once and your entire FastAPI server stops
serving until the call returns — including health checks. Read the
"Async I/O" section of the existing `backend/CLAUDE.md` once.

## 4. Log sanitizer

PII in logs is a compliance and a leakage problem. Create
`backend/utils/log_sanitizer.py`:

```python
# in backend/utils/log_sanitizer.py
import re

_EMAIL = re.compile(r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}")
_PHONE = re.compile(r"\+?\d[\d\s\-().]{7,}\d")
_TOKEN = re.compile(r"(eyJ[a-zA-Z0-9_\-]+\.[a-zA-Z0-9_\-]+\.[a-zA-Z0-9_\-]+)")


def sanitize(text: str | bytes | None) -> str:
    if text is None:
        return ""
    if isinstance(text, bytes):
        try:
            text = text.decode("utf-8", errors="replace")
        except Exception:
            return f"<{len(text)} bytes>"
    text = _EMAIL.sub("<email>", text)
    text = _PHONE.sub("<phone>", text)
    text = _TOKEN.sub("<jwt>", text)
    if len(text) > 2048:
        text = text[:2048] + "...<truncated>"
    return text


def sanitize_pii(text: str | None) -> str:
    """Aggressive: replace anything looking like a name with `<pii>`."""
    if text is None:
        return ""
    cleaned = sanitize(text)
    cleaned = re.sub(r"\b[A-Z][a-z]+\s+[A-Z][a-z]+\b", "<pii>", cleaned)
    return cleaned
```

Rule: every `log.info(some_response_text)` becomes
`log.info(sanitize(some_response_text))`. Same for exceptions whose
messages contain raw API responses.

## 5. Rate limiting middleware

A 5-line Lua script in Redis that gives us robust per-uid rate limits:

Create `backend/utils/rate_limit.py`:

```python
# in backend/utils/rate_limit.py
import time
from typing import Annotated

from fastapi import Depends, HTTPException, status

from database import redis_db


_LUA = """
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
redis.call('ZREMRANGEBYSCORE', key, 0, now - window * 1000)
local count = redis.call('ZCARD', key)
if count >= limit then
  return 0
end
redis.call('ZADD', key, now, now)
redis.call('PEXPIRE', key, window * 1000)
return 1
"""


def with_rate_limit(uid_dep, policy: str, limit: int = 60, window_seconds: int = 60):
    """FastAPI dependency factory."""
    def dependency(uid: Annotated[str, Depends(uid_dep)]) -> str:
        c = redis_db._get_client()
        if not c:
            return uid  # fail-open if Redis is down
        key = f"rl:{policy}:{uid}"
        now_ms = int(time.time() * 1000)
        try:
            allowed = c.eval(_LUA, 1, key, limit, window_seconds, now_ms)
        except Exception:
            return uid
        if not allowed:
            raise HTTPException(status.HTTP_429_TOO_MANY_REQUESTS, "rate limit exceeded")
        return uid

    return dependency
```

Usage example (we'll wire it into routers later):

```python
from utils.auth import get_current_user_uid
from utils.rate_limit import with_rate_limit

@router.post("/v1/chat/messages")
def send(uid: str = Depends(with_rate_limit(get_current_user_uid, "chat", limit=30))):
    ...
```

## 6. Cloud Storage helper

Create a bucket. In GCP Console → Cloud Storage → Buckets → Create:

- Name: `<<YOUR_BRAND>>-prod-audio` (must be globally unique).
- Region: same as Firestore.
- Storage class: Standard.
- Public access prevention: ON.
- Versioning: OFF.

Append to `requirements.txt`:

```
google-cloud-storage==2.18.0
```

Create `backend/utils/storage.py`:

```python
# in backend/utils/storage.py
from datetime import timedelta

from google.cloud import storage

from utils.settings import get_settings

_client: storage.Client | None = None


def _bucket(name: str):
    global _client
    if _client is None:
        _client = storage.Client(project=get_settings().gcp_project_id)
    return _client.bucket(name)


def upload_bytes(bucket: str, path: str, data: bytes, content_type: str = "application/octet-stream") -> str:
    b = _bucket(bucket).blob(path)
    b.upload_from_string(data, content_type=content_type)
    return f"gs://{bucket}/{path}"


def signed_url(bucket: str, path: str, expires_in_seconds: int = 600) -> str:
    return _bucket(bucket).blob(path).generate_signed_url(
        expiration=timedelta(seconds=expires_in_seconds), method="GET"
    )
```

Set the bucket name in `.env`:

```
BUCKET_AUDIO=<<YOUR_BRAND>>-prod-audio
BUCKET_PHOTOS=<<YOUR_BRAND>>-prod-photos
```

## 7. Smoke test the new utilities

Create `backend/scripts/smoke_utils.py`:

```python
# in backend/scripts/smoke_utils.py
from utils.encryption import encrypt_for, decrypt_for
from utils.log_sanitizer import sanitize, sanitize_pii
from database import redis_db


uid = "uid_test_42"
ct = encrypt_for(uid, "hello world")
print("ciphertext:", ct[:40], "...")
print("decrypted :", decrypt_for(uid, ct))

print("sanitized :", sanitize("My token is eyJabc.def.ghi and email a@b.c"))
print("pii       :", sanitize_pii("Alice Smith called Bob Jones"))

print("redis ping:", redis_db.set("ping", b"pong", ex=10), redis_db.get("ping"))
```

Run:

```bash
cd backend
python -m scripts.smoke_utils
```

Expected output: ciphertext/plaintext round-trips, sanitized strings,
and `b'pong'`.

## 8. Commit

```bash
git add backend
git commit -m "feat(part-05): redis, encryption, http client, log sanitizer, rate limit"
git push
```

## What you should have right now

- [ ] `redis_db.set/get/lock` works against Upstash.
- [ ] `encrypt_for/decrypt_for` round-trips.
- [ ] `sanitize` and `sanitize_pii` strip emails/phones/JWTs/names.
- [ ] `get_webhook_client()` returns a shared `httpx.AsyncClient`.
- [ ] `with_rate_limit` blocks the 61st request in 60 seconds.
- [ ] A Cloud Storage bucket exists and you can upload bytes to it.

---

Next: [Part 06 — Users, Profiles, Onboarding](./06-users-and-onboarding.md).
