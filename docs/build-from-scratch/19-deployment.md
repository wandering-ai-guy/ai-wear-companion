# Part 19 — Containerization & Deployment

> Goal of this part: a Docker image of the backend running on a real
> cloud, with TLS, autoscaling, secrets management, and a reproducible
> deploy pipeline.

There are three reasonable targets. Pick **one** and stick with it.

| Target | Best for | Notes |
|--------|----------|-------|
| **Cloud Run** (GCP) | v1, low ops, autoscale to zero | We use it here. |
| **GKE** (Kubernetes) | Heavy WebSocket traffic, fine-grained scaling | Reference repo uses this. |
| **Fly.io** | Multi-region, low cost, dead simple | Great for solo founders. |

We'll set up Cloud Run, then add a brief "scale up to GKE later" note.

## 1. Backend Dockerfile

`backend/Dockerfile`:

```dockerfile
FROM python:3.11-slim AS base
ENV PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1

RUN apt-get update && apt-get install -y --no-install-recommends \
    ffmpeg libopus0 libsndfile1 build-essential git \
  && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade pip && pip install --no-cache-dir -r requirements.txt

COPY . /app

ENV PORT=8080
EXPOSE 8080

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080", "--workers", "2", "--loop", "uvloop", "--ws", "websockets"]
```

Build & run locally:

```bash
cd backend
docker build -t <<YOUR_BRAND>>-api:dev .
docker run --rm -p 8080:8080 --env-file .env <<YOUR_BRAND>>-api:dev
```

Hit `http://localhost:8080/v1/health` to confirm.

## 2. Push to Google Artifact Registry

```bash
gcloud artifacts repositories create <<YOUR_BRAND>>-images \
  --repository-format=docker \
  --location=us-central1 \
  --project=<<YOUR_GCP_PROJECT_ID>>

gcloud auth configure-docker us-central1-docker.pkg.dev

docker tag <<YOUR_BRAND>>-api:dev \
  us-central1-docker.pkg.dev/<<YOUR_GCP_PROJECT_ID>>/<<YOUR_BRAND>>-images/api:0.1.0

docker push \
  us-central1-docker.pkg.dev/<<YOUR_GCP_PROJECT_ID>>/<<YOUR_BRAND>>-images/api:0.1.0
```

## 3. Secrets via Google Secret Manager

Don't bake `.env` into the image, and don't paste secrets into the
Cloud Run UI. Use Secret Manager.

```bash
# Enable
gcloud services enable secretmanager.googleapis.com

# Create one secret per value
echo -n "$GEMINI_API_KEY" | gcloud secrets create GEMINI_API_KEY --data-file=-
echo -n "$DEEPGRAM_API_KEY" | gcloud secrets create DEEPGRAM_API_KEY --data-file=-
# … etc for every var in .env

# Or in bulk via a script:
while IFS='=' read -r k v; do
  [[ -z "$k" || "$k" == "#"* ]] && continue
  printf "%s" "$v" | gcloud secrets create "$k" --data-file=- 2>/dev/null \
    || (printf "%s" "$v" | gcloud secrets versions add "$k" --data-file=-)
done < .env.production

# Allow the Cloud Run service account to read them:
SA="$(gcloud iam service-accounts list --filter="displayName:Default compute service account" --format='value(email)')"
for s in GEMINI_API_KEY DEEPGRAM_API_KEY PINECONE_API_KEY ENCRYPTION_SECRET REDIS_DB_PASSWORD ; do
  gcloud secrets add-iam-policy-binding "$s" \
    --member="serviceAccount:$SA" --role="roles/secretmanager.secretAccessor"
done
```

## 4. Deploy to Cloud Run

```bash
gcloud run deploy <<YOUR_BRAND>>-api \
  --image=us-central1-docker.pkg.dev/<<YOUR_GCP_PROJECT_ID>>/<<YOUR_BRAND>>-images/api:0.1.0 \
  --region=us-central1 \
  --platform=managed \
  --allow-unauthenticated \
  --memory=1Gi --cpu=2 \
  --concurrency=80 \
  --min-instances=1 --max-instances=20 \
  --timeout=3600 \
  --session-affinity \
  --service-account="$SA" \
  --set-env-vars=ENV=production,LOG_LEVEL=INFO,GCP_PROJECT_ID=<<YOUR_GCP_PROJECT_ID>> \
  --set-secrets=GEMINI_API_KEY=GEMINI_API_KEY:latest,\
DEEPGRAM_API_KEY=DEEPGRAM_API_KEY:latest,\
PINECONE_API_KEY=PINECONE_API_KEY:latest,\
ENCRYPTION_SECRET=ENCRYPTION_SECRET:latest,\
REDIS_DB_PASSWORD=REDIS_DB_PASSWORD:latest
```

The flags that matter:

- `--timeout=3600` — Cloud Run's default request timeout is 5 min;
  we need 1 hour for the listen WebSocket.
- `--session-affinity` — sticks a client to the same instance, which
  matters for stateful WS connections.
- `--concurrency=80` — number of concurrent requests *per instance*.
  Tune based on memory headroom.
- `--min-instances=1` — keeps one always-warm. Without this you pay
  the cold-start penalty per request, which is brutal for live audio.

## 5. Custom domain + TLS

```bash
gcloud beta run domain-mappings create \
  --service=<<YOUR_BRAND>>-api \
  --domain=api.<<YOUR_DOMAIN>> \
  --region=us-central1
```

Cloud Run prints a CNAME / A record. Add it to your DNS provider.
TLS is auto-provisioned via Google-managed cert in ~minutes.

## 6. Pusher service

Same drill, separate service:

```bash
gcloud run deploy <<YOUR_BRAND>>-pusher \
  --image=us-central1-docker.pkg.dev/.../pusher:0.1.0 \
  --region=us-central1 --platform=managed \
  --memory=512Mi --cpu=1 \
  --min-instances=1 --max-instances=5 \
  --no-allow-unauthenticated \
  --service-account="$SA" \
  --set-env-vars=...
```

`--no-allow-unauthenticated` because nothing on the public internet
should hit pusher. The API talks to it via Pub/Sub or by service-to-
service ID tokens.

## 7. Diarizer (GPU)

Cloud Run has a GPU offering (T4 / L4) but it's pricier and the cold
start is bad. For dev, [Modal](https://modal.com/) is the lowest-effort
option:

```python
# diarizer/modal_app.py
import modal

image = modal.Image.from_registry("nvidia/cuda:12.1.1-cudnn8-runtime-ubuntu22.04", add_python="3.11") \
  .apt_install("ffmpeg", "libsndfile1") \
  .pip_install_from_requirements("requirements.txt") \
  .env({"HUGGINGFACE_TOKEN": modal.Secret.from_name("hf-token")})

stub = modal.App("diarizer", image=image)


@stub.cls(gpu="T4", container_idle_timeout=120, allow_concurrent_inputs=4)
class Diarizer:
    @modal.enter()
    def load(self):
        # initialize models (same as our diarizer/main.py module-level code)
        ...

    @modal.method()
    def diarize(self, wav_bytes: bytes):
        ...
```

`modal deploy diarizer/modal_app.py`. Modal gives you a stable URL
the backend calls. T4 GPU with 0–1 instances scales to zero idle.

## 8. Domain checklist

DNS:

- `<<YOUR_DOMAIN>>` (apex) → static landing page on Vercel/Netlify.
- `api.<<YOUR_DOMAIN>>` → Cloud Run.
- `agent.<<YOUR_DOMAIN>>` (later) → agent-proxy service.
- `mx` → Google Workspace / Fastmail / SES.

## 9. Mobile-app config

Once deployed, set in your Flutter app's `--dart-define` per build:

```bash
flutter build ios --dart-define=BASE_API_URL=https://api.<<YOUR_DOMAIN>>
```

Don't bake API URLs into source. Different flavors (dev/staging/prod)
should swap them via build args.

## 10. CI/CD

`.github/workflows/deploy.yml` (manual trigger):

```yaml
name: deploy-backend
on: { workflow_dispatch: { inputs: { tag: { required: true } } } }

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions: { contents: read, id-token: write }
    steps:
      - uses: actions/checkout@v4
      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/<num>/locations/global/workloadIdentityPools/gh/providers/gh
          service_account: deployer@<proj>.iam.gserviceaccount.com
      - uses: google-github-actions/setup-gcloud@v2
      - name: Build & push
        run: |
          gcloud auth configure-docker us-central1-docker.pkg.dev
          docker build -t us-central1-docker.pkg.dev/<proj>/.../api:${{ github.event.inputs.tag }} backend/
          docker push us-central1-docker.pkg.dev/<proj>/.../api:${{ github.event.inputs.tag }}
      - name: Deploy
        run: |
          gcloud run deploy <<YOUR_BRAND>>-api \
            --image=us-central1-docker.pkg.dev/<proj>/.../api:${{ github.event.inputs.tag }} \
            --region=us-central1 --platform=managed
```

Set up Workload Identity Federation between GitHub and GCP so
the workflow has no long-lived secrets.

## 11. Scaling beyond Cloud Run

When you hit limits (>5K concurrent listen WS, sustained
GPU-bound diarization), move to **GKE Autopilot**. The reference
repo has Helm charts under `backend/charts/`:

- `backend-listen` — REST + WS API, 4Gi mem, HPA on RPS.
- `pusher` — bursty, scaled on Pub/Sub backlog.
- `diarizer` — GPU node pool, scaled on queue length.
- `agent-proxy` — small.

Read those charts when the time comes. The single biggest reason to
move is **per-pod CPU stickiness for WebSockets** — Cloud Run will
restart instances aggressively if traffic dips, and that's a poor
fit for hour-long voice sessions.

## 12. Commit

```bash
git add backend pusher diarizer infra .github
git commit -m "feat(part-19): docker + cloud run + secret manager + ci/cd"
git push
```

## What you should have right now

- [ ] `docker build` produces a working image.
- [ ] Image pushed to Artifact Registry.
- [ ] Cloud Run service running, reachable at `https://api.<<YOUR_DOMAIN>>`.
- [ ] Secrets injected from Secret Manager, not env files.
- [ ] DNS + TLS working.
- [ ] Pusher deployed as its own service.
- [ ] (Optional) Modal-hosted diarizer.
- [ ] CI workflow that builds and deploys on demand.

---

Next: [Part 20 — Observability & Cost Monitoring](./20-observability.md).
