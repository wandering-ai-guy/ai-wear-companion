# Part 07 — Live Audio Streaming + STT

> Goal of this part: open a WebSocket from the mobile app to the
> backend, receive audio frames, forward them to Deepgram for live
> transcription, save the resulting transcript segments to Firestore,
> and stream them back to the client.

This is the heart of the product. It's also the hardest part of this
manual. Take your time.

## 1. The wire format

For v1, we accept **two** input formats over the WebSocket:

| Codec      | Sample rate | Bit depth | Notes |
|------------|-------------|-----------|-------|
| PCM 16-bit | 16 kHz      | 16        | Default. Smallest code path. |
| Opus       | 16 kHz      | 16        | Wearables encode in Opus to save bandwidth. |

Each frame the client sends is exactly one of these. We negotiate
which one via a query string on connect:

```
wss://api.<<YOUR_DOMAIN>>/v1/listen?codec=pcm16&sample_rate=16000&language=en
wss://api.<<YOUR_DOMAIN>>/v1/listen?codec=opus&sample_rate=16000&language=en
```

For both, the client sends **binary** WebSocket messages (each one
~20–250 ms of audio). We send **text JSON** messages back, e.g.:

```json
{"type": "transcript.partial", "segment": {"id": "...", "text": "hello world", "speaker": "0", "start_ms": 320, "end_ms": 1280}}
{"type": "transcript.final",   "segment": {...}}
{"type": "conversation.opened", "id": "..."}
{"type": "conversation.closed", "id": "..."}
{"type": "error", "message": "..."}
```

Document this contract somewhere your mobile devs will read. Keep it
versioned (`/v1/listen`, `/v2/listen`, …) so you can break it later
without breaking shipped clients.

## 2. Deepgram setup

In your Deepgram console:

- Generate a **scoped API key** with `usage:read` + `manage:write` +
  `live:read` permissions.
- Copy the key to `.env` as `DEEPGRAM_API_KEY=...`.

Append to `requirements.txt`:

```
deepgram-sdk==4.8.1
opuslib==3.0.1
PyOgg @ git+https://github.com/TeamPyOgg/PyOgg@6871a4f234e8a3a346c4874a12509bfa02c4c63a
```

(`opuslib` is for decoding Opus frames if you ever need to *inspect*
audio server-side. The Deepgram SDK can ingest Opus directly.)

## 3. Audio processing utilities

Create `backend/utils/audio.py`:

```python
# in backend/utils/audio.py
from dataclasses import dataclass
from typing import Optional


@dataclass(frozen=True)
class AudioFormat:
    codec: str           # "pcm16" or "opus"
    sample_rate: int     # 16000 or 8000
    channels: int = 1
    bit_depth: int = 16

    @property
    def bytes_per_second(self) -> int:
        if self.codec == "pcm16":
            return self.sample_rate * self.channels * (self.bit_depth // 8)
        return 0  # opus is variable

    @classmethod
    def parse(cls, codec: str, sample_rate: Optional[int], language: str) -> "AudioFormat":
        codec = (codec or "pcm16").lower()
        sample_rate = sample_rate or 16000
        if codec not in {"pcm16", "opus"}:
            raise ValueError(f"unsupported codec: {codec}")
        if sample_rate not in {8000, 16000}:
            raise ValueError(f"unsupported sample rate: {sample_rate}")
        return cls(codec=codec, sample_rate=sample_rate)
```

## 4. Deepgram streaming adapter

Create `backend/utils/stt/deepgram.py`:

```python
# in backend/utils/stt/deepgram.py
import asyncio
import json
import logging
from typing import AsyncIterator, Awaitable, Callable, Optional

import websockets

from utils.audio import AudioFormat
from utils.settings import get_settings

log = logging.getLogger(__name__)

OnTranscriptCallback = Callable[[dict], Awaitable[None]]


def _dg_url(fmt: AudioFormat, language: str) -> str:
    encoding = "linear16" if fmt.codec == "pcm16" else "opus"
    qs = (
        f"encoding={encoding}"
        f"&sample_rate={fmt.sample_rate}"
        f"&channels={fmt.channels}"
        f"&model=nova-3"
        f"&language={language}"
        f"&interim_results=true"
        f"&smart_format=true"
        f"&endpointing=400"
        f"&punctuate=true"
        f"&diarize=false"  # we'll do diarization ourselves in Part 08
    )
    return f"wss://api.deepgram.com/v1/listen?{qs}"


class DeepgramStream:
    """One-per-listen WebSocket session adapter."""

    def __init__(
        self,
        fmt: AudioFormat,
        language: str = "en",
        on_transcript: Optional[OnTranscriptCallback] = None,
    ):
        self.fmt = fmt
        self.language = language
        self.on_transcript = on_transcript
        self._ws: Optional[websockets.WebSocketClientProtocol] = None
        self._reader_task: Optional[asyncio.Task] = None

    async def start(self) -> None:
        token = get_settings().deepgram_api_key
        if not token:
            raise RuntimeError("DEEPGRAM_API_KEY missing")
        self._ws = await websockets.connect(
            _dg_url(self.fmt, self.language),
            extra_headers={"Authorization": f"Token {token}"},
            max_size=8 * 1024 * 1024,
            ping_interval=5,
        )
        self._reader_task = asyncio.create_task(self._reader())
        log.info("deepgram.started codec=%s sr=%d", self.fmt.codec, self.fmt.sample_rate)

    async def send_audio(self, chunk: bytes) -> None:
        if self._ws is None:
            return
        await self._ws.send(chunk)

    async def keep_alive(self) -> None:
        if self._ws is None:
            return
        await self._ws.send(json.dumps({"type": "KeepAlive"}))

    async def close(self) -> None:
        try:
            if self._ws is not None:
                await self._ws.send(json.dumps({"type": "CloseStream"}))
                await self._ws.close()
        finally:
            if self._reader_task:
                self._reader_task.cancel()

    async def _reader(self) -> None:
        assert self._ws is not None
        try:
            async for raw in self._ws:
                if isinstance(raw, bytes):
                    continue
                msg = json.loads(raw)
                if msg.get("type") == "Results" and self.on_transcript:
                    await self.on_transcript(msg)
        except websockets.ConnectionClosed:
            log.info("deepgram.closed")
        except Exception:
            log.exception("deepgram.reader.error")
```

What this gives us:

- An async context with `start/send_audio/close`.
- Each Deepgram "Results" message hits our `on_transcript` callback.
- A `keep_alive` we'll fire every ~5s when the user is silent so
  Deepgram doesn't drop the connection.

## 5. Transcript segment model

```python
# in backend/models/segment.py
from datetime import datetime
from typing import Optional
from pydantic import BaseModel, Field
import uuid


class TranscriptSegment(BaseModel):
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    conversation_id: str
    text: str
    speaker_id: Optional[str] = None
    start_ms: int
    end_ms: int
    is_final: bool = False
    language: str = "en"
    created_at: datetime = Field(default_factory=datetime.utcnow)
```

```python
# in backend/database/segments.py
from typing import List
from database._client import db
from models.segment import TranscriptSegment

_USERS = "users"
_CONVS = "conversations"
_SEGS = "segments"


def append_segment(uid: str, seg: TranscriptSegment) -> None:
    """Encrypted-on-write — see Part 17 for hooking in encryption."""
    db.collection(_USERS).document(uid) \
        .collection(_CONVS).document(seg.conversation_id) \
        .collection(_SEGS).document(seg.id) \
        .set(seg.model_dump(mode="json"))


def list_segments(uid: str, conversation_id: str) -> List[TranscriptSegment]:
    snaps = db.collection(_USERS).document(uid) \
        .collection(_CONVS).document(conversation_id) \
        .collection(_SEGS).order_by("start_ms").stream()
    return [TranscriptSegment(**s.to_dict()) for s in snaps]
```

## 6. The WebSocket router

Create `backend/routers/listen.py`:

```python
# in backend/routers/listen.py
import asyncio
import json
import logging
import time
import uuid
from typing import Optional

import firebase_admin.auth
from fastapi import APIRouter, Query, WebSocket, WebSocketDisconnect, WebSocketException, status

from database import redis_db
from database.segments import append_segment
from models.segment import TranscriptSegment
from utils.audio import AudioFormat
from utils.settings import get_settings
from utils.stt.deepgram import DeepgramStream

log = logging.getLogger(__name__)

router = APIRouter(prefix="/v1", tags=["listen"])


def _verify_uid_from_token(token: str) -> str:
    s = get_settings()
    if s.admin_key and token.startswith(s.admin_key):
        return token[len(s.admin_key):]
    return firebase_admin.auth.verify_id_token(token, check_revoked=False)["uid"]


@router.websocket("/listen")
async def listen(
    ws: WebSocket,
    token: Optional[str] = Query(None),
    codec: str = Query("pcm16"),
    sample_rate: Optional[int] = Query(16000),
    language: str = Query("en"),
):
    # 1. Auth (always before accept(), use 1008 not HTTPException)
    if not token:
        await ws.close(code=status.WS_1008_POLICY_VIOLATION)
        return
    try:
        uid = _verify_uid_from_token(token)
    except Exception as e:
        log.warning("listen.auth_failed: %s", e)
        await ws.close(code=status.WS_1008_POLICY_VIOLATION)
        return

    # 2. Lock — only one listen WS per user.
    with redis_db.lock(f"listen:{uid}", ttl_seconds=120) as locked:
        if not locked:
            await ws.close(code=status.WS_1008_POLICY_VIOLATION, reason="already-streaming")
            return

        try:
            fmt = AudioFormat.parse(codec, sample_rate, language)
        except ValueError as e:
            await ws.close(code=status.WS_1003_UNSUPPORTED_DATA, reason=str(e))
            return

        await ws.accept()

        conversation_id = str(uuid.uuid4())
        await ws.send_text(json.dumps({"type": "conversation.opened", "id": conversation_id}))

        loop = asyncio.get_running_loop()
        last_audio_ts = loop.time()

        async def on_transcript(msg: dict) -> None:
            try:
                ch = msg.get("channel", {})
                alt = (ch.get("alternatives") or [{}])[0]
                text = (alt.get("transcript") or "").strip()
                if not text:
                    return
                start = int(msg.get("start", 0) * 1000)
                duration = int(msg.get("duration", 0) * 1000)
                seg = TranscriptSegment(
                    conversation_id=conversation_id,
                    text=text,
                    start_ms=start,
                    end_ms=start + duration,
                    is_final=bool(msg.get("is_final")),
                    language=language,
                )
                if seg.is_final:
                    append_segment(uid, seg)
                kind = "transcript.final" if seg.is_final else "transcript.partial"
                await ws.send_text(json.dumps({"type": kind, "segment": seg.model_dump(mode="json")}))
            except Exception:
                log.exception("on_transcript failed")

        dg = DeepgramStream(fmt=fmt, language=language, on_transcript=on_transcript)
        await dg.start()

        async def keepalive() -> None:
            while True:
                await asyncio.sleep(5)
                if loop.time() - last_audio_ts > 4:
                    try:
                        await dg.keep_alive()
                    except Exception:
                        return

        ka_task = asyncio.create_task(keepalive())

        try:
            while True:
                msg = await ws.receive()
                if msg["type"] == "websocket.disconnect":
                    raise WebSocketDisconnect(code=msg.get("code", 1000))
                if "bytes" in msg and msg["bytes"]:
                    chunk = msg["bytes"]
                    last_audio_ts = loop.time()
                    await dg.send_audio(chunk)
                elif "text" in msg and msg["text"]:
                    # Optional client commands. Ignored for now.
                    pass
        except WebSocketDisconnect:
            log.info("listen.disconnected uid=%s conv=%s", uid, conversation_id)
        except Exception:
            log.exception("listen.error")
        finally:
            ka_task.cancel()
            await dg.close()
            try:
                await ws.send_text(json.dumps({"type": "conversation.closed", "id": conversation_id}))
            except Exception:
                pass
```

Register in `main.py`:

```python
from routers import health, users, listen
app.include_router(listen.router)
```

## 7. The "WebSocket vs HTTPException" land mine

Note we used `await ws.close(code=...)` and never raised
`HTTPException`. If you raise `HTTPException` *before* `accept()`, the
ASGI server returns a 5xx to the client with no clean handshake — and
load balancers like GCP's HTTP(S) LB will count those as service-level
500s, which can wreck your SLO. **Always close the socket politely.**

## 8. Test it locally

There's a small Python script we can use to send raw audio. Save a
test WAV file `hello.wav` (mono, 16kHz, 16-bit PCM) and run:

```python
# in backend/scripts/listen_client.py
import asyncio
import os
import sys
import wave

import websockets


async def main(path: str):
    token = "test-uid"  # use admin-key bypass
    admin = os.environ["ADMIN_KEY"]
    uri = f"ws://localhost:8080/v1/listen?token={admin}{token}&codec=pcm16&sample_rate=16000&language=en"
    async with websockets.connect(uri) as ws:
        async def reader():
            async for msg in ws:
                print(msg)
        r = asyncio.create_task(reader())
        with wave.open(path, "rb") as w:
            assert w.getnchannels() == 1 and w.getsampwidth() == 2 and w.getframerate() == 16000
            chunk_ms = 50
            chunk = w.readframes(int(16000 * chunk_ms / 1000))
            while chunk:
                await ws.send(chunk)
                await asyncio.sleep(chunk_ms / 1000)
                chunk = w.readframes(int(16000 * chunk_ms / 1000))
        await asyncio.sleep(3)
        r.cancel()


asyncio.run(main(sys.argv[1]))
```

Run:

```bash
python -m scripts.listen_client hello.wav
```

You should see a stream of partial / final transcript JSON messages.

## 9. What's not in this part yet

- **Diarization** (who is speaking) — Part 08.
- **End-of-conversation auto-finalize** — Part 09.
- **Saving the audio file** to Cloud Storage — that goes in the
  pusher service in Part 15.
- **Encryption of segment text at rest** — Part 17.

Right now the simplest thing that works: each WS connection = one
new conversation, ends when the socket closes.

## 10. Commit

```bash
git add backend
git commit -m "feat(part-07): /v1/listen WS + Deepgram live transcription"
git push
```

## What you should have right now

- [ ] WebSocket at `/v1/listen` accepts auth.
- [ ] PCM 16 kHz frames stream into Deepgram.
- [ ] Partial + final transcripts come back to the client.
- [ ] Final segments get written to `users/{uid}/conversations/{id}/segments`.
- [ ] Listen lock prevents concurrent connections per user.

---

Next: [Part 08 — VAD & Speaker Diarization](./08-vad-and-diarization.md).
