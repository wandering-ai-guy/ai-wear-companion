# Part 18 — Testing Strategy

> Goal of this part: a `make test` that catches the mistakes you
> actually make, plus a CI workflow that runs it on every PR.

## 1. The pyramid you actually need

Forget the textbook 1000:100:10 ratios. For an audio-streaming AI
backend with lots of third-party integrations, what works:

- **Lots of unit tests** of pure functions: prompt formatting,
  encryption, audio framing, BLE parser. These are cheap and
  catch off-by-one and "did you remember to seed?" bugs.
- **A few integration tests** that hit a real Firestore (emulator),
  a real Redis, and a *fake* Deepgram. These catch wiring and import
  bugs.
- **End-to-end smoke tests** in CI that boot the whole API + a
  stubbed STT and run a fixed conversation through it. Catch
  regressions in the flow.

What you don't need at first: load tests, fuzzers, mutation tests.
They're all good things — and a distraction from shipping.

## 2. Layout

```
backend/tests/
  unit/
    test_audio_format.py
    test_encryption.py
    test_post_process_prompt.py
    ...
  integration/
    conftest.py
    test_users_router.py
    test_listen_ws.py
    ...
  fixtures/
    hello_16k.wav
    sample_conversation.json
```

`backend/test.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")"

source .venv/bin/activate
export ENCRYPTION_SECRET="test-secret-must-be-long-enough"
export ADMIN_KEY="testkey"
export PYTHONPATH="$(pwd)"

# Unit tests run anywhere
pytest -q tests/unit "$@"

# Integration tests skip if env not set
if [[ "${RUN_INTEGRATION:-0}" == "1" ]]; then
  pytest -q tests/integration "$@"
fi
```

`backend/test-preflight.sh`:

```bash
#!/usr/bin/env bash
set -e
python -V
which pytest
python - <<'PY'
import importlib
for m in ("fastapi", "uvicorn", "redis", "pydantic", "google.genai", "deepgram", "pinecone"):
    importlib.import_module(m)
print("ok")
PY
```

## 3. Patterns for unit tests

### Pre-mock heavy deps

`utils.llm.clients.gemini_client()` lazily creates a real client
when called. In tests, patch it before importing the module under
test:

```python
# in tests/unit/test_post_process.py
from types import SimpleNamespace
from unittest.mock import AsyncMock, patch

import pytest


@pytest.fixture
def fake_gemini():
    # Gemini's async API returns an object whose .text is a JSON string.
    rsp = SimpleNamespace(
        text='{"title":"t","summary":"s","category":"work","action_items":[],"language":"en"}'
    )
    client = SimpleNamespace(
        aio=SimpleNamespace(
            models=SimpleNamespace(
                generate_content=AsyncMock(return_value=rsp),
                embed_content=AsyncMock(return_value=SimpleNamespace(
                    embeddings=[SimpleNamespace(values=[0.0] * 768)],
                )),
            )
        )
    )
    with patch("utils.llm.post_process.gemini_client", return_value=client):
        yield client


@pytest.mark.asyncio
async def test_post_process_writes_title(fake_gemini, monkeypatch):
    from utils.llm import post_process
    seen = {}
    monkeypatch.setattr(post_process, "list_segments", lambda u, c: [type("S", (), {
        "speaker_id": "me", "text": "hi", "start_ms": 0, "end_ms": 100, "is_final": True
    })()])
    monkeypatch.setattr(post_process, "patch", lambda u, c, p: seen.update(p))
    monkeypatch.setattr(post_process, "mark_status", lambda u, c, s, **f: None)
    monkeypatch.setattr(post_process, "extract_action_items", AsyncMock())
    monkeypatch.setattr(post_process, "extract_memories", AsyncMock())
    await post_process.post_process_conversation("uid_test", "cid_test")
    assert seen["title"] == "t"
```

### Patch the module, not the source

Always `monkeypatch.setattr(post_process, "list_segments", ...)`,
not `monkeypatch.setattr("database.segments.list_segments", ...)`.
The first form patches the *imported reference* the function
actually uses. The second silently misses because the function
already imported its own copy.

## 4. Integration tests with the Firestore emulator

```bash
# install
gcloud components install cloud-firestore-emulator

# run (in a separate terminal during tests)
gcloud emulators firestore start --host-port=localhost:8181
```

Set the env var so the SDK targets the emulator:

```bash
export FIRESTORE_EMULATOR_HOST=localhost:8181
```

Now `database._client.db` writes and reads from the emulator. Your
tests can do:

```python
# tests/integration/conftest.py
import os
import pytest


@pytest.fixture(autouse=True, scope="session")
def _emulator():
    assert os.environ.get("FIRESTORE_EMULATOR_HOST"), "Start the emulator first"
    yield
```

For Redis, run a real local one (`brew services start redis`) and
point `REDIS_DB_HOST=localhost`, `REDIS_DB_PORT=6379`.

## 5. WebSocket integration test

```python
# tests/integration/test_listen_ws.py
import asyncio
import os
import wave

import pytest
import websockets

from main import app  # ensure routes registered
from utils.settings import get_settings


@pytest.mark.asyncio
async def test_listen_returns_partial(monkeypatch, fake_deepgram):
    # fake_deepgram is a fixture that patches DeepgramStream.start/_reader
    # to emit a canned partial then a final.
    s = get_settings()
    uri = f"ws://localhost:8080/v1/listen?token={s.admin_key}testuid&codec=pcm16&sample_rate=16000"
    async with websockets.connect(uri) as ws:
        # send 1 second of silence
        silence = b"\x00\x00" * 16000
        await ws.send(silence)
        msg = await asyncio.wait_for(ws.recv(), timeout=3)
        assert "transcript" in msg
```

A test like this needs your backend running. Either start it as a
subprocess in a fixture, or use `httpx.AsyncClient(app=app)` for the
REST tests and a separate uvicorn process for WS.

## 6. Stubbing third parties

For deterministic tests, replace each third-party call with a
fixture-loaded stub:

| Real | Stub returns |
|------|--------------|
| Deepgram WS | a canned sequence of partial/final transcripts |
| Gemini `generate_content` | the literal text in `tests/fixtures/llm_<name>.txt` |
| Gemini `embed_content` | a deterministic 768-d vector from `hashlib.sha256(text).digest()` |
| Pinecone | an in-memory dict |
| FCM | a no-op |

## 7. CI: GitHub Actions

`.github/workflows/backend.yml`:

```yaml
name: backend
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }
      - name: Install
        run: |
          pip install -U pip
          pip install -r backend/requirements.txt
          pip install pytest pytest-asyncio
      - name: Lint
        run: |
          black --check --line-length 120 --skip-string-normalization backend pusher
      - name: Unit tests
        env:
          ENCRYPTION_SECRET: ci-test-secret-12345678
          ADMIN_KEY: ci
        run: |
          cd backend
          PYTHONPATH=. pytest -q tests/unit
```

For integration tests that need Firestore + Redis, use a service
container or the emulator setup above. Skip them on PRs from forks
(no secrets) and run them on `main`.

## 8. Smoke test against ngrok before release

```bash
# in scripts/smoke.sh
set -euo pipefail
BASE="${BASE_API_URL:-http://localhost:8080}"

curl -fsS "$BASE/v1/health" >/dev/null
echo "✓ health"

# auth flow with admin bypass
TOKEN="${ADMIN_KEY}smoke_test_uid"
curl -fsS -H "Authorization: Bearer $TOKEN" "$BASE/v1/users/me" >/dev/null
echo "✓ /me"

curl -fsS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"message":"hello"}' "$BASE/v1/chat/messages" >/dev/null
echo "✓ /chat"
```

Run `bash scripts/smoke.sh` after every deploy.

## 9. Test data discipline

- **Never** copy production data into a dev environment.
- **Always** use synthetic data fixtures committed to the repo.
- For the speech profile fixture, record yourself reading a public
  domain text (gitignore the WAV; commit only its embedding).

## 10. Commit

```bash
git add backend .github
git commit -m "test(part-18): unit/integration scaffolding + ci"
git push
```

## What you should have right now

- [ ] `make test` runs unit tests in <30 seconds.
- [ ] At least 5 unit tests covering encryption, prompts, audio
  parsing, BLE header, redis lock.
- [ ] An integration test that hits the Firestore emulator.
- [ ] A WebSocket integration test with a stubbed Deepgram.
- [ ] A GitHub Actions workflow that runs on every PR.

---

Next: [Part 19 — Containerization & Deployment](./19-deployment.md).
