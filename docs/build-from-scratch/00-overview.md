# Part 00 — Overview & Architecture

> Goal of this part: by the end you should be able to draw the system on a
> napkin and explain, out loud, what every box does and how data flows from
> "user speaks into a wearable" to "an AI chat that remembers what they said."

## What you are building, in one paragraph

A user wears a small Bluetooth audio device (or just uses their phone's
microphone). The audio is streamed in real time to your servers. Your servers
turn the audio into text, figure out who is speaking, decide when a
"conversation" has ended, then ask a large language model to summarize it,
extract action items, and pull out long-term facts ("memories"). The user can
search and chat with everything they have ever said, on any device, with push
notifications and offline sync. You charge a monthly subscription, optionally
sell the wearable, and let third parties build "apps" on top of it.

That paragraph is the elevator pitch. The rest of this manual is the wiring
diagram.

## The cast of characters

We can split the system into **four planes**:

```
┌──────────────────────────────────────────────────────────────────────────┐
│                            CLIENT PLANE                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────────┐      │
│  │ Wearable   │  │ Phone Mic  │  │ Mobile App │  │ Desktop App    │      │
│  │ (BLE audio)│  │ (Flutter)  │  │ (Flutter)  │  │ (optional)     │      │
│  └─────┬──────┘  └────┬───────┘  └────┬───────┘  └────────┬───────┘      │
│        │ BLE          │ HTTPS/WS      │ HTTPS/WS          │              │
└────────┼──────────────┼───────────────┼───────────────────┼──────────────┘
         │              │               │                   │
         └──── via mobile app ──────────┘                   │
                        │                                   │
                        ▼                                   ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                            EDGE / API PLANE                              │
│                                                                          │
│  ┌────────────────────┐  ┌────────────────────┐  ┌────────────────────┐  │
│  │  Backend API       │  │   Pusher           │  │   Agent-Proxy      │  │
│  │  (FastAPI)         │  │   (FastAPI, WS)    │  │   (FastAPI, WS)    │  │
│  │                    │  │                    │  │                    │  │
│  │  REST + WebSocket  │  │  Realtime fan-out  │  │  Bridges chat to   │  │
│  │  for *everything*  │  │  Audio batching    │  │  user's agent VM   │  │
│  └─────────┬──────────┘  └─────────┬──────────┘  └─────────┬──────────┘  │
└────────────┼────────────────────── ┼────────────────────── ┼─────────────┘
             │                       │                       │
             ▼                       ▼                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                          BUSINESS LOGIC PLANE                            │
│                                                                          │
│  ┌────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐   │
│  │  STT       │  │  Diarizer   │  │  VAD        │  │  LLM            │   │
│  │  Deepgram  │  │  pyannote   │  │  pyannote   │  │  Gemini 2.5     │   │
│  └────────────┘  └─────────────┘  └─────────────┘  └─────────────────┘   │
│                                                                          │
│  ┌────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐   │
│  │ Embedding  │  │ Vector DB   │  │ Search      │  │ Translation     │   │
│  │ (Gemini)   │  │ Pinecone    │  │ Typesense   │  │ Google Translate│   │
│  └────────────┘  └─────────────┘  └─────────────┘  └─────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
             │                                         │
             ▼                                         ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                            STORAGE PLANE                                 │
│                                                                          │
│  ┌────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐   │
│  │ Firestore  │  │ Redis       │  │ Cloud       │  │ Knowledge Graph │   │
│  │ (primary)  │  │ (cache,     │  │ Storage     │  │ (Neo4j, opt.)   │   │
│  │            │  │  rate lim.) │  │ (audio,     │  │                 │   │
│  │            │  │             │  │  photos)    │  │                 │   │
│  └────────────┘  └─────────────┘  └─────────────┘  └─────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

You will build all four planes, mostly bottom-up: storage first, business
logic next, then the API, and finally the clients.

## What lives in each box

### Client plane

- **Wearable.** A button-sized device with a microphone, a battery, and a
  Bluetooth chip. It exposes three BLE services: battery, device-info, and
  audio. The audio service streams encoded audio frames to the phone in tiny
  chunks (Part 14). For day one of your build, **you can skip the wearable**
  and use the phone microphone instead.
- **Mobile app.** A Flutter application that runs on iOS and Android. It
  scans for the wearable, holds the BLE connection, opens a WebSocket to your
  backend, displays transcripts, conversations, memories, chat, and so on.
- **Desktop app (optional).** A native macOS/Windows app for power users.
  Skip this for v1.

### API plane

- **Backend API.** Your main FastAPI service. Exposes REST endpoints for
  conversations, memories, chat, payment, etc., and a WebSocket at
  `/v1/listen` (or `/v4/listen` in this repo) where audio is streamed.
- **Pusher.** A second FastAPI service that exists only to handle the
  *fan-out* of real-time audio: pushing chunks to STT, batching audio for
  storage, calling diarization, kicking off LLM jobs at the end of a
  conversation, etc. We split this out from the main API for two reasons:
  (1) it's the only service that does long-lived stateful work, and we want
  to scale it independently, and (2) crashing the pusher should not crash
  the REST API.
- **Agent-Proxy (optional, advanced).** A third tiny service whose only job
  is to bridge a user's chat WebSocket to a private "agent VM" running their
  personal AI agent. You only need this if you want users to run their own
  isolated agents. Skip until Part 15.

### Business logic plane

These are mostly *third-party services* called from the API plane, but you
will write a thin wrapper around each one so you can swap providers later.

- **STT (Speech-To-Text):** Deepgram is the default because it has the
  lowest live-streaming latency. AssemblyAI and Google's Cloud
  Speech-to-Text are alternatives.
- **Diarizer:** Splits "who is talking" into separate speaker IDs. We use
  the open-source `pyannote/speaker-diarization` model, hosted as its own
  microservice on a GPU. For v1 you can skip this and pretend everyone is
  speaker 0.
- **VAD (Voice Activity Detection):** Says "yes there is speech in this
  100ms chunk" or "no, this is silence." We use it to gate the STT (don't
  pay for transcribing silence) and to detect end-of-conversation.
- **LLM:** **Google Gemini** — specifically `gemini-2.5-flash` for fast,
  cheap calls (post-processing, memory extraction, tool dispatch) and
  `gemini-2.5-pro` for harder reasoning (chat answers). We use a thin
  wrapper so you can swap to Anthropic Claude or another provider later
  without rewriting routers.
- **Embeddings:** Turn each conversation/memory into a vector. Google's
  `text-embedding-004` (**768 dims**) is the default. Cheap, fast, and
  in the same vendor as the LLM, which simplifies billing alerts.
- **Vector DB:** Pinecone serverless. (Alternatives: Qdrant, Weaviate.)
- **Search:** Typesense for full-text search of transcripts.
- **Translation:** Google Cloud Translate (optional, multi-language users).

### Storage plane

- **Firestore.** Document database from Google. Stores users, conversations,
  memories, action items, etc. Encrypted at rest (we add another layer of
  per-user app-level encryption on top — see Part 17).
- **Redis.** In-memory cache. Used for rate limiting, locks (so you can't
  open two listen WebSockets for the same user at once), short-lived state.
  Hosted via Upstash for serverless.
- **Cloud Storage** (Google Cloud Storage / S3). Stores raw audio recordings
  and image attachments. Generally NOT served directly to clients — instead
  we serve through signed URLs.
- **Knowledge Graph** (Neo4j, optional). Stores entity relationships
  ("person X works at company Y") for advanced retrieval. Optional.

## The two flows you must know cold

There are dozens of features, but they all reduce to two data flows. If you
understand these two, you understand the system.

### Flow 1: Real-time audio → live transcript

```
Wearable ──BLE──► Mobile App ──WS──► Backend API ──WS──► Pusher
                                          │
                                          ├──► Deepgram (STT, streaming)
                                          ├──► VAD service
                                          ├──► Diarizer service
                                          │
                                          ▼
                                      Firestore (transcript_segments)
                                          │
                                          ▼
                                      Mobile App (live updates via WS)
```

1. Phone app opens a WebSocket to `wss://api.<<YOUR_DOMAIN>>/v1/listen`.
2. It either captures the phone mic or forwards Bluetooth audio chunks.
3. Backend forwards each chunk to:
   - Deepgram (which streams back partial + final transcripts), and
   - the pusher (which buffers ~4s of audio and sends it to the diarizer +
     speaker-identification service).
4. As soon as a "final" transcript fragment comes back, the backend tags it
   with a speaker ID, encrypts it, writes it to Firestore, and pushes it
   back to the mobile app over the same WebSocket.
5. After ~2 minutes of silence, the conversation is "finalized," and Flow 2
   takes over.

### Flow 2: Finalized transcript → memory + chat

```
                Conversation ends (silence > 2 min)
                            │
                            ▼
                    LLM post-processor
                  (title, summary, category)
                            │
              ┌─────────────┼──────────────┐
              ▼             ▼              ▼
        Action items   Memories      Embeddings
        (Firestore)    (Firestore)    (Pinecone)
                                          │
                                          ▼
                              Chat agent uses these
                              for retrieval-augmented
                              generation when the user
                              asks a question.
```

1. When a conversation is finalized, the backend asks an LLM:
   - "Give me a 1-line title and a 5-bullet summary of this conversation."
   - "What category? (work, social, study, …)"
   - "What are the action items?"
   - "What facts about the user can we extract as long-term memories?"
2. Each answer is stored in its own Firestore collection.
3. The full transcript and each memory are turned into embeddings and stored
   in Pinecone, indexed by `user_id`.
4. Later, when the user opens chat and asks "what did I say to my mom about
   the trip?", the chat endpoint:
   - turns the question into an embedding,
   - queries Pinecone for the nearest conversations,
   - feeds those conversations into a Claude/GPT prompt as context,
   - streams the answer back over WebSocket.

Everything else in the system — phone calls, calendar sync, Todoist
integration, app marketplace — is built on top of these two flows.

## What we are *not* doing in this manual

To keep scope manageable, the manual will not cover:

- The web personas/landing site (see `web/personas-open-source/`).
- A native iOS/Swift desktop app (see `desktop/`).
- A custom firmware for the wearable from zero (we use the existing Omi or
  Friend device hardware spec; we *do* cover the BLE protocol and OTA
  updates so you can swap firmware later).
- Compliance certifications (SOC 2, HIPAA, ISO 27001). Those are real work
  but separate from "build the product."

## What success looks like at the end

When you are done with all 22 parts, you should be able to:

1. Hand a stranger an iPhone with your app installed.
2. They sign up with Google, finish onboarding.
3. They pair a wearable (or use the phone mic).
4. They have a conversation. The transcript shows up in real time.
5. Two minutes later, the conversation is closed and shows a title +
   summary.
6. They open the chat tab and ask "what did I just talk about?" and get a
   correct, streaming answer.
7. They get a push notification two hours later: "I noticed you mentioned
   X — want me to remind you tomorrow?"
8. They see their billing page, can subscribe, and you can see their
   payment in Stripe.

That is the entire core product. Everything else (apps marketplace, OAuth
integrations, knowledge graph, OTA, etc.) is a layer of polish.

---

Next: [Part 01 — Branding, Naming, Cloud Accounts](./01-branding-and-accounts.md).
