# Part 22 — Launch Checklist

> Goal of this part: a final pass before you tell strangers your
> product exists. Nothing here is research — it's a list of things
> that, if any one is missed, will cost you a lot of pain on day 2.

Print this. Tick boxes only when you've personally verified.

## Engineering

### Backend hygiene
- [ ] All env vars come from Secret Manager, not `.env`.
- [ ] `ADMIN_KEY` bypass is disabled in production (code path is
      gated on `ENV == "development"`).
- [ ] `/internal/metrics` is **not** publicly reachable (block at LB
      or via auth header).
- [ ] CORS allowlist is your real domains, not `*`.
- [ ] Firestore security rules deny direct reads to
      conversations/memories/etc.; only the user-doc read is allowed.
- [ ] Composite indexes for every query are deployed.
- [ ] All sensitive Firestore fields are stored encrypted (`*_enc`).
- [ ] Pinecone metadata contains no plaintext user content.
- [ ] Logs are JSON; no raw transcripts; PII sanitized.
- [ ] HTTP timeouts set on every outbound client (Deepgram, Gemini,
      Pinecone, webhooks).
- [ ] No `requests.*`, `time.sleep`, `Thread().start().join()` in
      async code (`scripts/lint_async_blockers.py` clean).
- [ ] Rate limits enforced on `/v1/chat`, `/v1/listen`, `/v1/search`,
      `/v1/users/me/export`.
- [ ] Idempotency keys on `POST /v1/payment/*` (so a retried Stripe
      webhook doesn't double-charge).

### Mobile app hygiene
- [ ] Production flavor uses production Firebase config; staging
      and dev are isolated.
- [ ] App version + build number bumped on every release.
- [ ] Crash reporting (Sentry / Firebase Crashlytics) wired up.
- [ ] `BASE_API_URL` is built into the binary, not user-editable.
- [ ] BLE permission strings are clear in `Info.plist` /
      `AndroidManifest.xml` (the App Store will reject vague ones).
- [ ] Microphone permission strings explain *why*.
- [ ] All user-visible strings go through l10n.
- [ ] Privacy Policy + Terms of Service links live in Settings and
      on the sign-in screen.
- [ ] App handles auth-token refresh transparently; users don't
      see "401 — please log in" mid-conversation.

### Hardware (if you ship a wearable)
- [ ] Firmware version reported correctly via Device Info Service.
- [ ] OTA update tested at least 3 times on real devices.
- [ ] Mute button is wired and visibly indicates recording state.
- [ ] Battery telemetry is accurate within ±5%.
- [ ] FCC / CE / UKCA certification done (if shipping in those
      regions).
- [ ] Manufacturing test plan: fixture jig, golden firmware, QC
      yield > 95%.

## Compliance / Legal

- [ ] Privacy Policy published at `<<YOUR_DOMAIN>>/privacy`.
- [ ] Terms of Service at `<<YOUR_DOMAIN>>/terms`.
- [ ] Cookie / tracker disclosure at `<<YOUR_DOMAIN>>/cookies`
      (if you have a marketing site).
- [ ] DPA template ready for B2B prospects.
- [ ] GDPR Article 30 (Records of Processing) doc filled.
- [ ] Subprocessor list maintained (Google Gemini, Deepgram, Pinecone,
      Firebase, Stripe, Sentry, etc.) and linked from privacy.
- [ ] If under-13 users are possible: COPPA-compliant flow and a
      hard age gate.
- [ ] If California users: a "Do Not Sell" toggle in Settings.
- [ ] If recording in two-party-consent states (CA, FL, MA, MD,
      others): in-app reminder when entering one.

## App stores

### iOS
- [ ] App Store Connect set up; bundle ID matches.
- [ ] Privacy "nutrition label" filled accurately (location, audio,
      contacts, ID, etc.).
- [ ] Screenshots for every required device size.
- [ ] App preview video ≤ 30s.
- [ ] Reviewer notes explain how the wearable is paired (with a demo
      account that has speech profiles + sample data).
- [ ] Encryption export compliance answered (ATS+ standard
      cryptography → "no" to ITSAppUsesNonExemptEncryption).
- [ ] Background modes match what the app actually does.
- [ ] In-app purchase flow tested with sandbox account.

### Android
- [ ] Play Console listing complete.
- [ ] Closed testing track used by ≥12 testers for ≥14 days before
      production release (Google enforces this for new accounts).
- [ ] Data safety form filled accurately.
- [ ] Foreground-service permission justified (audio recording).
- [ ] Targeted API level meets current requirement.
- [ ] In-app purchase tested via Play license testers.

## Billing (Stripe)

- [ ] Live mode keys configured in Secret Manager.
- [ ] Webhook endpoint `/v1/payment/webhook` set up in Stripe
      dashboard with the live signing secret.
- [ ] All product price IDs exist in Stripe and match the IDs in
      `utils/subscription.py` (validation runs at startup).
- [ ] Failed-payment dunning email enabled.
- [ ] Cancel button in Settings actually cancels at period end.
- [ ] Invoices include your business name + tax ID.

## Operations

- [ ] PagerDuty / Opsgenie schedule with primary + secondary on-call.
- [ ] Runbooks: "Deepgram down," "Firestore quota exceeded," "Cost
      spike," "Suspected breach." Each ≤1 page.
- [ ] Backups: Firestore → BigQuery export weekly; GCS bucket
      versioning on critical buckets only.
- [ ] DR drill: spin up the backend in a different region, point
      DNS at it, verify a happy-path test passes.
- [ ] Status page (statuspage.io / instatus.com) wired to your
      monitoring SLOs.
- [ ] Customer support inbox (`hello@<<YOUR_DOMAIN>>` →
      Help Scout / Front / Linear-issues).

## Cost guardrails

- [ ] Daily LLM spend alarm.
- [ ] Daily STT minutes alarm.
- [ ] Per-user fair-use cap enforced (Part 20).
- [ ] Pinecone serverless on the smallest viable plan.
- [ ] Cloud Run autoscale max-instances set; no runaway scaling.
- [ ] Kill switch: a single env var (`MAINTENANCE=1`) that returns
      503 from `/v1/listen` and `/v1/chat` while leaving `/health`
      and `/me` up — for emergency cost / abuse stops.

## Marketing & growth (light)

- [ ] Landing page at `<<YOUR_DOMAIN>>` explains what the product
      *is* in 8 seconds.
- [ ] Sign-up email confirmed: a real human reads the welcome
      sequence.
- [ ] Discord / Slack community link.
- [ ] Public roadmap (GitHub Projects or Linear public board).
- [ ] Analytics: PostHog / Amplitude tracking 5–10 key events
      (signup, first conversation, first chat message,
      subscription started, churn).
- [ ] Email marketing pipeline (Loops / Customer.io) for a 5-touch
      welcome sequence.
- [ ] Press kit page with logo, screenshots, founder photos.

## Day-2 followups (don't block launch on these)

- [ ] SOC 2 Type 1 (when you have ≥3 enterprise prospects).
- [ ] Localized App Store listings.
- [ ] Affiliate / referral program.
- [ ] Self-serve "build your own app" docs (the `/sdks` and
      developer-portal story).
- [ ] Hardware retail / channel partnerships.
- [ ] Voice-first interactions ("Hey <<YOUR_BRAND>>, what was that
      thing about Italy?").

## Launch day

- [ ] An "incident channel" Slack room.
- [ ] At least 2 engineers on standby (one frontend, one backend).
- [ ] LangSmith + Sentry + the main dashboard open on a TV.
- [ ] A pre-prepared email + tweet thread + LinkedIn post ready to
      schedule.
- [ ] First 100 invitations to friends and trusted users.

---

# You're done.

If you've built every part of this manual, you have:

- A FastAPI backend with WebSocket streaming, real-time transcription,
  diarization, LLM summaries, vector + keyword search, push
  notifications, billing, encryption, observability, autoscaling.
- A separate pusher worker for fan-out + cron.
- A diarizer microservice on a GPU.
- A Flutter mobile app, branded, on iOS + Android, paired to a BLE
  wearable.
- A repeatable deploy pipeline.
- A privacy / security posture that won't embarrass you in an audit.

The rest is product, distribution, and time. Keep the manual close
when you onboard the next engineer.

If something here turned out to be wrong or out of date, open a PR
against this manual. The reference codebase under `backend/`, `app/`,
`pusher/`, `diarizer/`, and `agent-proxy/` is the source of truth —
this manual is a friendly translation of it.

Good luck. Ship.
