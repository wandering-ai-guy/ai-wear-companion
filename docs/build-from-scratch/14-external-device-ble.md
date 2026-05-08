# Part 14 — External Device (BLE Wearable)

> Goal of this part: pair a Bluetooth audio wearable with the mobile
> app, decode the audio frames it sends, forward them to the
> `/v1/listen` WebSocket, and accept firmware updates over the air.

If you're not building hardware, you can still follow the protocol
section — it's the contract any third-party wearable must speak to
work with your app.

## 1. The BLE protocol

The reference Omi/Friend wearable exposes three BLE services:

| Service | UUID | Purpose |
|---------|------|---------|
| Battery | `0x180F` (standard) | Battery level (0–100). Notifications. |
| Device Info | `0x180A` (standard) | Manufacturer, model, firmware version. |
| Audio | `19B10000-E8F2-537E-4F6C-D104768A1214` | Audio stream + codec selector. |

The audio service has two characteristics:

| Characteristic | UUID | Purpose |
|----------------|------|---------|
| Audio Data | `19B10001-...` | Streamed audio samples (notify) |
| Codec Type | `19B10002-...` | One byte selecting the codec |

Codec values:

| Value | Codec | Sample rate | Bit depth |
|-------|-------|-------------|-----------|
| 0 | PCM | 16 kHz | 16-bit mono |
| 1 | PCM | 8 kHz | 16-bit mono |
| 10 | µ-law | 16 kHz | 8-bit |
| 11 | µ-law | 8 kHz | 8-bit |
| 20 | Opus | 16 kHz | 16-bit |

**Default for new devices: Opus at 16 kHz** (small over the air,
high quality, best battery).

## 2. The audio frame structure

Each BLE notification on the Audio Data characteristic is:

```
+-------------------+--------+----------------------+
| Packet Number (2) | Index (1) |  Audio Payload    |
+-------------------+--------+----------------------+
        little-endian       ← raw codec bytes →
```

- **Packet Number** (uint16, little-endian) — overall counter;
  wraps at 65535. Used to detect drops.
- **Index** (uint8) — position of this notification within the
  packet. BLE MTU is small (~250 bytes); a 320-byte PCM packet
  arrives as 2–3 notifications with the same packet number and
  ascending index.
- **Audio Payload** — codec-specific bytes.

A "packet" in this protocol contains **160 samples** (10 ms at
16 kHz). With Opus, that's typically 30–60 bytes. With PCM 16-bit,
that's 320 bytes — fragmented over BLE.

## 3. Reassembly

On the phone, you can't blindly forward each notification — they
need reassembly into whole packets:

```python
# Pseudocode (Dart in the actual app)
buffer_by_packet_number: dict[int, list[bytes]] = {}

def on_notification(value: bytes):
    pkt = int.from_bytes(value[0:2], "little")
    idx = value[2]
    payload = value[3:]
    pieces = buffer_by_packet_number.setdefault(pkt, [])
    while len(pieces) <= idx:
        pieces.append(b"")
    pieces[idx] = payload

    # Heuristic: deliver when we get a notification with idx==0 for the *next* packet
    if idx == 0 and pkt > min(buffer_by_packet_number.keys()):
        prev = pkt - 1 if pkt - 1 in buffer_by_packet_number else None
        if prev is not None:
            yield_complete_packet(b"".join(buffer_by_packet_number.pop(prev)))
```

In Dart you'll write this with `flutter_blue_plus` or
`flutter_reactive_ble`. The Omi reference app has it in
`app/lib/services/devices/...`.

## 4. Forwarding to the backend

Once you have a complete audio packet on the phone:

- If codec is **Opus**, forward the bytes as-is via WebSocket. The
  backend tells Deepgram `encoding=opus`.
- If codec is **PCM 16 kHz**, forward as-is.
- If codec is **PCM 8 kHz** or **µ-law 8 kHz**, on-device upsample
  to 16 kHz before sending — Deepgram supports 8 kHz too, but our
  diarizer model assumes 16 kHz.

Open the WebSocket in the phone app on app launch (after auth):

```
wss://api.<<YOUR_DOMAIN>>/v1/listen?token=<FIREBASE_ID_TOKEN>&codec=opus&sample_rate=16000&language=en
```

Pause/resume the stream when the user toggles the device on/off.

## 5. Battery + device info reporting

The phone reads battery level once per minute and posts to:

```
POST /v1/devices/me/heartbeat
{
  "battery_pct": 73,
  "firmware": "1.0.4",
  "rssi": -54,
  "codec": "opus"
}
```

Backend:

```python
# in backend/routers/devices.py
from datetime import datetime
from fastapi import APIRouter, Depends
from pydantic import BaseModel

from database._client import db
from utils.auth import get_current_user_uid


router = APIRouter(prefix="/v1/devices", tags=["devices"])


class Heartbeat(BaseModel):
    battery_pct: int | None = None
    firmware: str | None = None
    rssi: int | None = None
    codec: str | None = None


@router.post("/me/heartbeat")
def heartbeat(body: Heartbeat, uid: str = Depends(get_current_user_uid)):
    db.collection("users").document(uid).collection("devices").document("primary").set(
        {**body.model_dump(exclude_unset=True), "updated_at": datetime.utcnow().isoformat()},
        merge=True,
    )
    return {"ok": True}
```

The mobile app shows the latest battery % from this doc. When
battery < 15 %, your push system from Part 13 fires a "charge me"
notification.

## 6. OTA firmware updates

**Don't** roll your own firmware update unless you also build the
hardware. If you ship the same Omi/Nordic-based hardware, use Nordic
DFU (the Nordic Device Firmware Update protocol). The backend's job
is just to host the firmware blobs and tell the app what to install.

Backend endpoint:

```python
# in backend/routers/firmware.py
from fastapi import APIRouter, Depends
from pydantic import BaseModel

from utils.storage import signed_url
from utils.auth import get_current_user_uid

router = APIRouter(prefix="/v1/firmware", tags=["firmware"])

LATEST = {
    "v1.x": {"version": "1.0.4", "url_path": "firmware/v1.0.4.zip", "min_app": "1.2.0"},
}


class FirmwareInfo(BaseModel):
    version: str
    url: str
    min_app: str


@router.get("/latest", response_model=FirmwareInfo)
def latest(family: str = "v1.x", uid: str = Depends(get_current_user_uid)):
    f = LATEST[family]
    return FirmwareInfo(
        version=f["version"],
        url=signed_url("<<YOUR_BRAND>>-prod-firmware", f["url_path"], expires_in_seconds=3600),
        min_app=f["min_app"],
    )
```

You upload the firmware blob to GCS:

```bash
gsutil cp firmware.zip gs://<<YOUR_BRAND>>-prod-firmware/firmware/v1.0.4.zip
```

The phone app fetches `/v1/firmware/latest`, compares with the
device's reported `firmware`, downloads via the signed URL, and
runs DFU.

## 7. Compatibility shim for third-party SDKs

If you want third-party developers to write drivers for *other*
wearables, document the **opus 16 kHz packet** as the lingua franca:

> A device is compatible with <<YOUR_BRAND>> if a phone-side adapter
> can deliver it as a stream of binary WebSocket messages each
> containing one Opus-encoded 16 kHz mono frame, with no header.

Then your Python/Swift/RN SDKs (`sdks/`) provide a `MyDeviceAdapter`
abstract base class with one method: `Stream<Uint8List> audioFrames()`.

## 8. Privacy & "do not record" affordances

A wearable that records always is unusual; privacy law (UK/EU/some
US states) requires the user to inform people they're recording, and
sometimes to obtain consent from third parties.

Build into the device + app:

- A **physical button** that mutes the mic and lights an LED
  indicating "not recording."
- A **soft button** in the mobile app that pauses recording for N
  minutes.
- A **"do not record" geofence** for places (e.g. medical clinics)
  the user marks as private.
- A **post-conversation deletion** with one tap.

Backend support is just an endpoint that updates a "recording
allowed" flag the device polls before broadcasting audio frames.

## 9. Commit

```bash
git add backend
git commit -m "feat(part-14): device heartbeat + firmware OTA endpoints"
git push
```

## What you should have right now

- [ ] You can articulate the BLE service/characteristic UUIDs and
  the 3-byte audio header.
- [ ] Your mobile app reassembles fragmented BLE notifications into
  whole packets.
- [ ] Audio is forwarded to `/v1/listen` over WebSocket.
- [ ] Battery, firmware, RSSI are reported via heartbeat.
- [ ] `GET /v1/firmware/latest` returns a signed URL the device can
  download from.
- [ ] Privacy controls exist (mute button, pause, delete-on-finish).

---

Next: [Part 15 — Background Workers & Cron](./15-background-workers.md).
