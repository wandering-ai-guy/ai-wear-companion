# Part 01 — Branding, Naming & Cloud Accounts

> Goal of this part: pick a name, register a domain, and create accounts on
> every cloud service you'll need so that future parts never get blocked
> waiting on a credit-card form.

This part is mostly clicking around in web consoles. It is boring. **Do
not skip it** — every account you don't create today is a 10-minute
interruption later.

## 1. Pick a brand

Choose three things and write them down somewhere you won't lose them
(a Notion page, a 1Password note, anywhere):

| Field | Example | Constraints |
|-------|---------|-------------|
| Product name | `Memora` | 1–12 letters, easy to spell out loud |
| Reverse-domain ID | `me.memora.app` | iOS bundle ID + Android package ID |
| Primary color | `#1F6FEB` | One hex color you'll repeat everywhere |

Put the name through:

- [Namecheap](https://www.namecheap.com/) — is `<<YOUR_BRAND>>.com` /
  `.app` / `.ai` available?
- [USPTO TESS](https://tmsearch.uspto.gov/) — are there active US
  trademarks in software?
- iOS App Store + Google Play search — same name already shipped?
- LinkedIn, X (Twitter), Instagram — handles available?

Don't proceed until you have a name where **all of the above** check out.

We'll refer to your name as `<<YOUR_BRAND>>` everywhere from here on.

## 2. Register the domain and create a logo

- Buy the `.com` (and `.app` if you're feeling rich) at Namecheap or
  Cloudflare Registrar. Cloudflare is cheaper but doesn't sell every TLD.
- Generate a logo. For day one, that's literally fine to:
  - use [Stable Diffusion](https://www.stablediffusionweb.com/) or
    [DALL·E 3](https://chat.openai.com/) to generate a single-color glyph,
  - clean it up in [Figma](https://www.figma.com/),
  - export at: 1024×1024 PNG (App Store icon),
    512×512 PNG (Play Store icon),
    1024×1024 SVG (mobile in-app),
    32×32 favicon.
- Save the assets at `branding/<<YOUR_BRAND>>/` in the repo (we'll create
  the repo in Part 02).

You can always rebrand later. Don't spend a week on this.

## 3. Cloud accounts you must create

The order does not matter, but you should have credentials for every one
of these before you start coding. Each row links to the relevant signup
page; **enable two-factor authentication on every account.**

### Critical (the system breaks without these)

| Service | What it's for | Free tier? | Sign up |
|---------|---------------|------------|---------|
| **Google Cloud Platform** | Firebase Auth, Firestore, Cloud Storage, Cloud Run | $300 trial credit | https://cloud.google.com/ |
| **Firebase** (same account as GCP) | Auth UI, push notifications, console | included with GCP | https://console.firebase.google.com/ |
| **OpenAI** | LLM + embeddings | pay-as-you-go, $5 minimum | https://platform.openai.com/ |
| **Deepgram** | Speech-to-text | $200 starter credit | https://console.deepgram.com/signup |
| **Pinecone** | Vector database | 1 starter index free | https://app.pinecone.io/ |
| **Upstash** | Redis (serverless) | 10K commands/day free | https://console.upstash.com/ |
| **GitHub** | Source control + CI | free | https://github.com/ |
| **Stripe** | Subscription billing | free until you charge | https://dashboard.stripe.com/register |
| **Sentry** | Error tracking | 5K events/month free | https://sentry.io/signup/ |

### Recommended (you'll add them by Part 13)

| Service | What it's for | Free tier? |
|---------|---------------|------------|
| **Apple Developer Program** | iOS app signing + APNs push | $99/year |
| **Google Play Console** | Android publishing | $25 one-time |
| **PostHog** or **Amplitude** | Product analytics | generous free tier |
| **LangSmith** | LLM tracing | 5K traces/month free |
| **Twilio** | Phone calls + SMS | pay-as-you-go |

### Optional (you'll add them in later parts if you want the feature)

- **Anthropic** (Claude) — alternative LLM, https://console.anthropic.com/
- **Modal** — serverless GPUs for VAD/diarization, https://modal.com/
- **Hume AI** — emotion detection, https://hume.ai/
- **ElevenLabs** — TTS, https://elevenlabs.io/
- **Perplexity** — web-search-as-a-service, https://www.perplexity.ai/
- **Hugging Face** — pretrained model downloads, https://huggingface.co/
- **Typesense Cloud** — full-text search, https://cloud.typesense.org/
- **Neo4j AuraDB** — knowledge graph, https://console.neo4j.io/

## 4. The "secrets" sheet

Open a new 1Password vault (or a `.env.dev` file you keep out of Git) and
add a row for each of these as you create them. We'll use these names in
every later part:

```
# === Critical ===
GCP_PROJECT_ID=
GCP_PROJECT_NUMBER=
FIREBASE_API_KEY=
FIREBASE_AUTH_DOMAIN=
GOOGLE_APPLICATION_CREDENTIALS=  # path to service-account.json
OPENAI_API_KEY=
DEEPGRAM_API_KEY=
PINECONE_API_KEY=
PINECONE_INDEX_NAME=
REDIS_DB_HOST=
REDIS_DB_PORT=
REDIS_DB_PASSWORD=
ENCRYPTION_SECRET=        # generate later in Part 05
ADMIN_KEY=                # generate later: `openssl rand -hex 16`
SENTRY_DSN=

# === Recommended ===
STRIPE_API_KEY=
STRIPE_WEBHOOK_SECRET=
APNS_KEY_ID=
APPLE_TEAM_ID=
LANGSMITH_API_KEY=
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=

# === Optional ===
ANTHROPIC_API_KEY=
HUGGINGFACE_TOKEN=
ELEVENLABS_API_KEY=
PERPLEXITY_API_KEY=
HUME_API_KEY=
NEO4J_URI=
NEO4J_USER=
NEO4J_PASSWORD=
```

Don't fill in the values yet for everything. Just have the empty rows
ready so you remember to fill them in as we go.

## 5. Set billing alerts

This is the most important step in this part. Cloud bills are how
weekend hobby projects become stories on Hacker News titled "I owe Google
$8,000."

- **GCP:** Billing → Budgets & alerts → New budget. Set a $50/month cap
  with email alerts at 50%, 90%, and 100%. Add a second alert at $200
  for "things are clearly very wrong."
- **OpenAI:** Settings → Billing → Usage limits. Hard limit $50/month
  while you're developing.
- **Deepgram:** Console → Billing → Set spend cap.
- **Pinecone:** stays free on the starter plan; just don't add a
  second index by accident.
- **Upstash:** Console → Settings → Billing alerts. Free tier is
  generous; this is more for once you launch.
- **AWS** (if you ever add it): Billing → Budgets. Same drill.

## 6. Create the GCP project (the only thing we'll actually *create* now)

You'll be in the GCP console a lot. Do this once now, so when Part 04
needs it, you're not blocked.

1. Go to https://console.cloud.google.com/.
2. Top bar → "Select a project" → "New project."
3. Name: `<<YOUR_BRAND>>-prod`. Project ID: GCP will suggest something
   like `<<YOUR_BRAND>>-prod-12345`. Write that ID down — you'll use it
   forever.
4. Once created, go to "APIs & Services" → "Library" and **enable** these
   APIs (search and click "Enable" for each):
   - Cloud Resource Manager API
   - Firebase Management API
   - Cloud Firestore API
   - Cloud Storage API
   - Identity Toolkit API (Firebase Auth)
   - Cloud Build API (you'll need this in Part 19)
   - Cloud Run Admin API (Part 19)
   - Secret Manager API (Part 19)
5. Go to https://console.firebase.google.com/, click "Add project," and
   *select your existing GCP project* — do not create a new one.
6. In Firebase, go to "Build → Authentication → Get started." Enable:
   - Email/Password
   - Google
   - Apple
7. In Firebase, go to "Build → Firestore Database → Create database":
   - Production mode (we'll write security rules in Part 04).
   - Region: pick the one nearest your users (e.g. `us-central1`).
   - You can only pick this *once* per project. Be deliberate.

Stop here. We will create the service-account JSON, the Storage bucket,
the Pinecone index, etc., on demand in their respective parts.

## 7. Repository setup

1. Create an empty private repo on GitHub: `<<YOUR_BRAND>>/<<YOUR_BRAND>>`.
2. Clone it locally:
   ```bash
   git clone git@github.com:<<YOUR_BRAND>>/<<YOUR_BRAND>>.git
   cd <<YOUR_BRAND>>
   ```
3. Add a `.gitignore` for Python, Flutter, and Node:
   ```bash
   curl -fsSL https://www.toptal.com/developers/gitignore/api/python,flutter,node,visualstudiocode,macos > .gitignore
   ```
4. Add a `LICENSE` file (MIT is fine for v1, you can change later).
5. Commit: `git add . && git commit -m "chore: bootstrap" && git push`.

## What you should have right now

- [ ] A product name that's not trademarked, with a domain.
- [ ] A logo at 1024×1024.
- [ ] A GCP project + Firebase initialized.
- [ ] Firestore in production mode.
- [ ] Email + Google + Apple sign-in enabled in Firebase Auth.
- [ ] Accounts on OpenAI, Deepgram, Pinecone, Upstash, GitHub, Stripe,
  Sentry — all with 2FA + billing caps.
- [ ] An empty private GitHub repo with `.gitignore` and `LICENSE`.
- [ ] A "secrets" doc with empty rows for every key you'll fill in later.

If any checkbox is unchecked, do not move on. The rest of this manual
assumes these exist.

---

Next: [Part 02 — Local Toolchain & Repo Layout](./02-toolchain-and-repo.md).
