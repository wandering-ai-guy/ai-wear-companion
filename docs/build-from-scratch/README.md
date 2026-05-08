# Build Your Own AI Wearable Backend & Mobile App — From Scratch

> A long-form, opinionated, step-by-step manual for someone who understands
> high-level programming concepts but has limited hands-on experience.

## What this manual is

This is a complete blueprint for building, from a blank folder, a system that
mirrors the capabilities of the Omi platform you see in this repository:

- A **mobile app** (iOS + Android) with your own brand and code.
- A **custom backend** that powers it: live audio streaming, real-time speech
  to text, speaker diarization, AI summaries, memories, agentic chat, push
  notifications, billing, and more.
- **External device** integration (a BLE audio wearable like Omi/Friend), so
  the app can stream audio from a button-sized device, not just the phone.

You don't need to follow it in one sitting. The work is broken into
**twenty-two numbered parts**. Treat each part as a separate "session" — finish
it, run the example, commit your code, then come back later for the next.

## Reading order

The parts are numbered. Read them top to bottom; each part assumes you have
done the previous ones.

| #  | Part | What you'll have when finished |
|----|------|--------------------------------|
| 00 | [Overview & Architecture](./00-overview.md) | A mental model of every moving piece you'll build. |
| 01 | [Branding, Naming, Cloud Accounts](./01-branding-and-accounts.md) | A name, a logo, and signed-up accounts on every cloud you'll need. |
| 02 | [Local Toolchain & Repo Layout](./02-toolchain-and-repo.md) | A working laptop, a Git repo, a VS Code workspace. |
| 03 | [FastAPI Skeleton & First Endpoint](./03-fastapi-skeleton.md) | A `GET /health` endpoint running locally and reachable from the internet. |
| 04 | [Firebase Auth & Firestore](./04-firebase-and-firestore.md) | Real users can sign up; their data lives in your database. |
| 05 | [Redis, Encryption, Shared Utilities](./05-redis-encryption-utils.md) | Caching, locks, rate-limiting, and per-user encryption work end to end. |
| 06 | [Users, Profiles, Onboarding](./06-users-and-onboarding.md) | A `/v1/users/me` endpoint, language/timezone, onboarding flow. |
| 07 | [Live Audio Streaming + STT](./07-audio-streaming-stt.md) | A WebSocket that turns microphone audio into a live transcript. |
| 08 | [VAD & Speaker Diarization](./08-vad-and-diarization.md) | The transcript knows when someone is speaking and *who* is speaking. |
| 09 | [Conversations Lifecycle & LLM Post-Processing](./09-conversations-and-llm.md) | Conversations auto-close, get titled, summarized, categorized. |
| 10 | [Memories, Embeddings, Vector DB](./10-memories-and-embeddings.md) | The system extracts long-term facts about the user and stores them for recall. |
| 11 | [AI Chat & Agentic RAG](./11-chat-and-rag.md) | A "talk to your second brain" chat that uses tools and your past data. |
| 12 | [Action Items, Calendar, Task Integrations](./12-action-items-and-integrations.md) | Tasks are extracted, synced to Todoist/Google Calendar/etc. |
| 13 | [Push Notifications](./13-push-notifications.md) | Phones receive proactive messages from your backend. |
| 14 | [External Device (BLE Wearable)](./14-external-device-ble.md) | Your app talks to a real audio wearable over Bluetooth, with OTA firmware. |
| 15 | [Background Workers & Cron](./15-background-workers.md) | A separate "pusher" service handles fan-out, batching, and slow jobs. |
| 16 | [Search: Typesense + Vectors](./16-search.md) | Users can search their entire transcript history fast. |
| 17 | [Privacy, Encryption, Log Sanitization](./17-privacy-and-security.md) | PII never appears in your logs; data is encrypted at rest. |
| 18 | [Testing Strategy](./18-testing.md) | Unit + integration tests, a preflight check, a CI pipeline. |
| 19 | [Containerization & Deployment](./19-deployment.md) | Your backend runs on real cloud infrastructure with TLS and autoscaling. |
| 20 | [Observability & Cost Monitoring](./20-observability.md) | You can see what's happening, why it's slow, and how much it costs. |
| 21 | [Mobile App Wiring & Branding](./21-mobile-app.md) | A Flutter app that uses your backend, your icon, your name. |
| 22 | [Launch Checklist](./22-launch-checklist.md) | A final pass: legal, app stores, billing, support. |

## Conventions used in this manual

- We use **Python 3.11** for the backend, **FastAPI** for the API framework,
  and **Flutter 3.x** for the mobile app. These are not the only correct
  choices, but they match what works in production today, and they let you
  copy-and-adapt code from the Omi reference where helpful.
- Code blocks that look like `# in path/to/file.py` mean: open that file in
  your editor and put the code there. We never expect you to memorize a long
  block — paste it, run it, then read it.
- Anywhere you see `<<YOUR_BRAND>>`, `<<YOUR_DOMAIN>>`, or similar in
  CAPS-WITH-UNDERSCORES, that is a value you fill in for your own product.
- "Mildly technical" means: you should be able to read every command and
  understand roughly what it does. If you can't, search the web first, then
  ask a senior friend, then ask the LLM you're using.
- We deliberately avoid time estimates ("this takes 2 days"). Different people
  go at different speeds. Instead each part lists **what subsystems change**
  and **what dependencies/risks** to be aware of, so you can decide where to
  spend your effort.

## How to actually use this

1. **Don't try to be original yet.** First copy this exact structure. You can
   refactor later. Originality at the architectural level, before you've ever
   shipped, is a trap.
2. **Run the example at the end of every part.** If a part doesn't end with
   "make a request and see X happen," you have not finished that part.
3. **Commit at the end of every part** with a message like
   `feat(part-04): firebase auth working`. You will want to roll back when you
   break something later.
4. **Spend money carefully.** Many cloud providers have generous free tiers,
   but a misconfigured loop can rack up real charges. Set billing alerts in
   Part 01.
5. **Read the source.** This manual was written next to a working
   reference implementation under `backend/`. When something here is too
   abstract, search that codebase for the exact filename and you'll find a
   battle-tested example.

When you are ready, open [Part 00 — Overview & Architecture](./00-overview.md).
