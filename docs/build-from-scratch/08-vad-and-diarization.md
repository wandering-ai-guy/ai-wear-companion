# Part 08 — VAD & Speaker Diarization

> Goal of this part: stop spending money transcribing silence (VAD),
> figure out *who* is speaking in each segment (diarization), and
> match speakers to known people the user has recorded a "speech
> profile" for (speaker identification).

This is where you start to feel like you're building a real product
instead of a Deepgram wrapper.

## 1. The three problems

Don't conflate them. They are different layers:

| Problem | Question | Output |
|---------|----------|--------|
| **VAD** (Voice Activity Detection) | Is there speech in this 30 ms window? | bool |
| **Diarization** | When does speaker A stop and speaker B start? | timeline of (start, end, speaker_idx) |
| **Speaker identification** | Is this speaker the user, "Mom," or unknown? | matches anonymous speaker_idx ↔ Person.id |

You can ship without diarization. You cannot ship a "second brain"
without speaker identification — otherwise everything in the
transcript looks like the user said it.

## 2. VAD: client-side first

The cheapest VAD runs **on the phone**. The mobile app uses
[silero-vad](https://github.com/snakers4/silero-vad) (or a simpler
energy threshold) and **simply doesn't send chunks below a threshold.**
Deepgram doesn't get billed for nothing.

For the backend, we add a *server-side* VAD too as a safety net (so
a misbehaving client can't burn money). Use a deployed serverless
service.

## 3. The VAD/diarizer microservice

We create a tiny FastAPI service that loads pyannote models and
exposes two endpoints. Hosted separately so:

- it can run on a GPU,
- the heavyweight CUDA dependency doesn't bloat your main backend
  Docker image,
- it can scale on its own.

Create `diarizer/main.py`:

```python
# in diarizer/main.py
import io
import logging
import os
from typing import List

import numpy as np
import torch
from fastapi import FastAPI, File, UploadFile, HTTPException
from pyannote.audio import Pipeline, Inference, Model
from pyannote.core import Segment
import soundfile as sf

logging.basicConfig(level=logging.INFO)
log = logging.getLogger(__name__)

HF_TOKEN = os.environ["HUGGINGFACE_TOKEN"]
DEVICE = torch.device("cuda" if torch.cuda.is_available() else "cpu")

diarization_pipe = Pipeline.from_pretrained(
    "pyannote/speaker-diarization-3.1", use_auth_token=HF_TOKEN
).to(DEVICE)
embedding_model = Model.from_pretrained("pyannote/embedding", use_auth_token=HF_TOKEN).to(DEVICE)
embedding_inference = Inference(embedding_model, window="whole")
vad_pipe = Pipeline.from_pretrained(
    "pyannote/voice-activity-detection", use_auth_token=HF_TOKEN
).to(DEVICE)

app = FastAPI(title="diarizer")


def _load_wav(file_bytes: bytes) -> tuple[np.ndarray, int]:
    audio, sr = sf.read(io.BytesIO(file_bytes), dtype="float32")
    if audio.ndim > 1:
        audio = audio.mean(axis=1)
    return audio, sr


@app.post("/v1/vad")
async def vad(audio: UploadFile = File(...)):
    waveform, sr = _load_wav(await audio.read())
    out = vad_pipe({"waveform": torch.from_numpy(waveform).unsqueeze(0), "sample_rate": sr})
    segs = [{"start": float(s.start), "end": float(s.end)} for s in out.get_timeline().support()]
    return {"segments": segs}


@app.post("/v1/diarization")
async def diarization(audio: UploadFile = File(...)):
    waveform, sr = _load_wav(await audio.read())
    out = diarization_pipe({"waveform": torch.from_numpy(waveform).unsqueeze(0), "sample_rate": sr})
    segs = []
    for turn, _, speaker in out.itertracks(yield_label=True):
        segs.append({"start": float(turn.start), "end": float(turn.end), "speaker": speaker})
    return {"segments": segs}


@app.post("/v1/embedding")
async def embedding(audio: UploadFile = File(...)):
    waveform, sr = _load_wav(await audio.read())
    emb = embedding_inference({
        "waveform": torch.from_numpy(waveform).unsqueeze(0),
        "sample_rate": sr,
    })
    return {"embedding": emb.tolist(), "dim": int(emb.shape[-1])}
```

`diarizer/requirements.txt`:

```
fastapi==0.118.0
uvicorn[standard]==0.30.5
pyannote.audio==3.3.1
torch==2.4.0
torchaudio==2.4.0
soundfile==0.12.1
numpy==1.26.4
```

`diarizer/Dockerfile`:

```dockerfile
FROM nvidia/cuda:12.1.1-cudnn8-runtime-ubuntu22.04
RUN apt-get update && apt-get install -y python3.11 python3-pip ffmpeg libsndfile1 && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
ENV PORT=8080
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

Sign up on [Hugging Face](https://huggingface.co/), accept the gated
license for `pyannote/speaker-diarization-3.1`, generate a read-token,
and put it in `HUGGINGFACE_TOKEN`.

In Part 19 you'll deploy this. For now you can run it on a friend's
GPU box, or a temporary [Modal](https://modal.com/) function.

## 4. Speech profile recording

The first time the user opens the app, ask them to read this aloud
(15–30 seconds):

> "I'm enrolling my voice with <<YOUR_BRAND>> so it can recognize me
> in conversations. The quick brown fox jumps over the lazy dog."

The mobile app records 16 kHz mono WAV, hits a backend endpoint that:

1. Forwards to `diarizer:/v1/embedding`.
2. Stores the resulting embedding under
   `users/{uid}/speech_profiles/me`.
3. Sets `users/{uid}.speech_profile_recorded = True`.

Add the endpoint:

```python
# in backend/routers/speech_profile.py
import io
import logging
import wave
from typing import Annotated

from fastapi import APIRouter, Depends, HTTPException, UploadFile, File

from database._client import db
from utils.auth import get_current_user_uid
from utils.http_client import get_stt_client
from utils.settings import get_settings

log = logging.getLogger(__name__)
router = APIRouter(prefix="/v1/speech-profile", tags=["speech-profile"])


@router.post("/upload")
async def upload(uid: Annotated[str, Depends(get_current_user_uid)],
                 audio: UploadFile = File(...)):
    raw = await audio.read()
    if len(raw) < 16000 * 2 * 5:  # less than 5 seconds of 16 kHz 16-bit mono
        raise HTTPException(400, "audio too short — record at least 5 seconds")

    s = get_settings()
    diarizer_url = (s.base_api_url or "") and ""  # placeholder
    diarizer_url = (
        # fall back to env or default
        __import__("os").environ.get("DIARIZER_URL", "http://localhost:9000")
    )

    client = get_stt_client()
    files = {"audio": ("speech.wav", raw, "audio/wav")}
    r = await client.post(f"{diarizer_url}/v1/embedding", files=files)
    r.raise_for_status()
    emb = r.json()["embedding"]

    db.collection("users").document(uid).collection("speech_profiles") \
        .document("me").set({"embedding": emb, "person_id": "me", "label": "self"})
    db.collection("users").document(uid).set(
        {"speech_profile_recorded": True}, merge=True
    )
    return {"ok": True, "dim": len(emb)}
```

Register in `main.py`:

```python
from routers import speech_profile
app.include_router(speech_profile.router)
```

## 5. End-to-end flow during a live conversation

This is the bit that ties it together. Update the `/v1/listen` flow
from Part 07 so that, in addition to streaming to Deepgram, it also:

1. Buffers ~4 seconds of audio in memory.
2. Every 4 s, posts the buffer to `diarizer:/v1/diarization`.
3. Every final segment Deepgram returns is *re-tagged* with the
   speaker that overlaps its `start_ms..end_ms` from the diarization
   response.
4. For each unique anonymous speaker label, compute its embedding
   once (when we have ≥3 s of its audio) and run a cosine similarity
   against the user's stored speech profiles to label it (`me`,
   `mom`, or `unknown`).

Conceptually:

```
    audio_buffer (~4s circular)
         │
         ▼
   diarizer.diarize()      ← every 4s, in background
         │
         └── timeline of (start, end, anon_speaker)
                      │
                      ▼
               speaker_id_resolver
       (anon_speaker → person_id via embeddings)
                      │
                      ▼
   apply to final transcript_segments before persisting
```

The reason we don't enable Deepgram's built-in diarization
(`diarize=true`) is **speaker recognition**: Deepgram diarization
gives you `speaker_0`, `speaker_1`, but it has no idea that
`speaker_0` is "Mom" — and across two different conversations,
their `speaker_0` is not stable. By doing it ourselves with
embeddings + a stored profile, we get **stable, named speakers**.

Sketch of the buffer + dispatch logic (you'd put this inside the
`listen` handler in `routers/listen.py`):

```python
# inside listen() — pseudocode in plain English

audio_buffer = bytearray()
PCM_BYTES_PER_SECOND = fmt.bytes_per_second
DIARIZE_EVERY_S = 4
diarization_segments: list[dict] = []  # latest known timeline

async def diarize_loop():
    while True:
        await asyncio.sleep(DIARIZE_EVERY_S)
        if len(audio_buffer) < PCM_BYTES_PER_SECOND * 2:
            continue
        wav_bytes = pcm16_to_wav(audio_buffer, sample_rate=fmt.sample_rate)
        audio_buffer.clear()  # one-shot (or: keep last 1s overlap)
        try:
            r = await get_stt_client().post(
                f"{DIARIZER_URL}/v1/diarization",
                files={"audio": ("c.wav", wav_bytes, "audio/wav")},
                timeout=8.0,
            )
            diarization_segments[:] = r.json().get("segments", [])
        except Exception:
            log.warning("diarization call failed", exc_info=True)
```

`pcm16_to_wav` is a 6-line helper in `utils/audio.py`:

```python
def pcm16_to_wav(pcm: bytes, sample_rate: int) -> bytes:
    import io, wave
    buf = io.BytesIO()
    with wave.open(buf, "wb") as w:
        w.setnchannels(1); w.setsampwidth(2); w.setframerate(sample_rate)
        w.writeframes(pcm)
    return buf.getvalue()
```

When a *final* Deepgram segment arrives in `on_transcript`, look up
which speaker overlaps it most:

```python
def speaker_for(start_ms: int, end_ms: int) -> str | None:
    best, best_overlap = None, 0
    for d in diarization_segments:
        s_ms, e_ms = int(d["start"] * 1000), int(d["end"] * 1000)
        overlap = max(0, min(end_ms, e_ms) - max(start_ms, s_ms))
        if overlap > best_overlap:
            best_overlap, best = overlap, d["speaker"]
    return best
```

Set `seg.speaker_id = speaker_for(seg.start_ms, seg.end_ms)` before
saving and emitting.

## 6. Mapping anonymous speakers to known people

After 3+ seconds of audio is attributed to "anon-A," compute its
embedding once and compare:

```python
import numpy as np

def cosine(a, b):
    a, b = np.asarray(a), np.asarray(b)
    return float(a @ b / (np.linalg.norm(a) * np.linalg.norm(b) + 1e-9))


async def label_speaker(uid: str, anon_speaker: str, audio_clip: bytes) -> str | None:
    r = await get_stt_client().post(
        f"{DIARIZER_URL}/v1/embedding",
        files={"audio": ("c.wav", audio_clip, "audio/wav")},
        timeout=8.0,
    )
    emb = r.json()["embedding"]
    # compare against known profiles
    profiles = db.collection("users").document(uid).collection("speech_profiles").stream()
    best, best_score = None, 0.0
    for p in profiles:
        score = cosine(emb, p.to_dict()["embedding"])
        if score > best_score:
            best_score, best = score, p.id
    return best if best_score > 0.65 else None  # 0.65 ≈ "same speaker"
```

The threshold (0.65 here) is empirical and depends on your embedding
model. Tune it with a few test clips.

## 7. Privacy considerations

- Tell the user, in plain English, that you store a *vector*
  derived from their voice (not the audio itself), and link to a
  page explaining what that vector can and cannot be used for.
- Let them delete the speech profile from the app. That deletion
  should cascade in `cascade_delete()` from Part 06.
- Never use a stranger's voice to enroll them without consent.
  When an unknown speaker is detected, label them
  `unknown_<conversation_id>_<idx>` — never store their embedding
  past the conversation.

## 8. Optional: "unknown speakers in the moment"

Inside one conversation, you *can* group anon-A samples without
labeling them. The mobile app shows them as
`Speaker 1`, `Speaker 2` and lets the user tap to assign them to a
contact. If they do, you store *that conversation's* anon-A
embedding under `users/{uid}/people/{person_id}.speaker_embedding_id`
so the next conversation auto-labels.

## 9. Commit

```bash
git add backend diarizer
git commit -m "feat(part-08): vad + diarizer microservice + speech profile"
git push
```

## What you should have right now

- [ ] A `diarizer/` service that exposes `/v1/vad`, `/v1/diarization`,
  `/v1/embedding`.
- [ ] A `POST /v1/speech-profile/upload` endpoint that stores a
  per-user embedding.
- [ ] The listen WebSocket buffers audio and re-tags transcript
  segments with stable speaker labels.
- [ ] Cosine similarity ≥ 0.65 maps anonymous speaker to "me" /
  "mom" / "dad."

---

Next: [Part 09 — Conversations Lifecycle & LLM Post-Processing](./09-conversations-and-llm.md).
