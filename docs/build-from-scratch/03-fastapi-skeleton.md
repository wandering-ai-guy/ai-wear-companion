# Part 03 — FastAPI Skeleton & First Endpoint

> Goal of this part: stand up a FastAPI service that runs locally,
> serves `GET /v1/health`, has structured logging, lives behind a
> tunnel that the internet can reach, and is wired up so future parts
> can drop in routers without touching `main.py`.

## 1. Create the virtual environment

```bash
cd backend
python3.11 -m venv .venv
source .venv/bin/activate
python -V    # → Python 3.11.x
```

You'll do `source .venv/bin/activate` every time you open a new
terminal. Tip: install [direnv](https://direnv.net/) and put
`source .venv/bin/activate` in `.envrc` so it's automatic.

## 2. Initial dependencies

Open `backend/requirements.txt` and put exactly this for now (we'll
grow it part by part):

```
fastapi==0.118.0
uvicorn[standard]==0.30.5
python-dotenv==1.0.1
pydantic==2.8.2
pydantic-settings==2.10.1
httpx==0.28.0
structlog==24.4.0
sentry-sdk[fastapi]==2.18.0
```

Install:

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

## 3. The `.env.template`

Create `backend/.env.template` (commit this to git) with placeholders:

```
ENV=development
PORT=8080
LOG_LEVEL=INFO

# Cloud
GCP_PROJECT_ID=
GOOGLE_APPLICATION_CREDENTIALS=

# Auth bypass for local dev (Part 04 will explain).
ADMIN_KEY=

# To be filled in later parts.
OPENAI_API_KEY=
DEEPGRAM_API_KEY=
PINECONE_API_KEY=
PINECONE_INDEX_NAME=
REDIS_DB_HOST=
REDIS_DB_PORT=
REDIS_DB_PASSWORD=
ENCRYPTION_SECRET=
SENTRY_DSN=

# Frontend / clients
BASE_API_URL=
ALLOWED_ORIGINS=*
```

Then copy it to `.env` (this one is **gitignored**):

```bash
cp .env.template .env
echo "ADMIN_KEY=$(openssl rand -hex 16)" >> .env
```

In `backend/.gitignore` (create or append):

```
.env
.venv
__pycache__/
*.pyc
_temp/
_samples/
_segments/
_speech_profiles/
google-credentials.json
```

## 4. Settings module (typed env)

Create `backend/utils/settings.py`:

```python
# in backend/utils/settings.py
from functools import lru_cache
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8", extra="ignore")

    env: str = "development"
    port: int = 8080
    log_level: str = "INFO"

    gcp_project_id: str = ""
    google_application_credentials: str = ""

    admin_key: str = ""

    openai_api_key: str = ""
    deepgram_api_key: str = ""
    pinecone_api_key: str = ""
    pinecone_index_name: str = ""

    redis_db_host: str = ""
    redis_db_port: int = 0
    redis_db_password: str = ""

    encryption_secret: str = ""
    sentry_dsn: str = ""

    base_api_url: str = ""
    allowed_origins: str = "*"

    @property
    def allowed_origins_list(self) -> list[str]:
        if self.allowed_origins.strip() == "*":
            return ["*"]
        return [o.strip() for o in self.allowed_origins.split(",") if o.strip()]


@lru_cache(maxsize=1)
def get_settings() -> Settings:
    return Settings()
```

Why `lru_cache`? Pydantic Settings reads the `.env` file on first
instantiation; the cache makes sure we only do that once per process.

## 5. Structured logging

Create `backend/utils/logging.py`:

```python
# in backend/utils/logging.py
import logging
import sys

import structlog


def configure_logging(level: str = "INFO") -> None:
    logging.basicConfig(
        format="%(message)s",
        stream=sys.stdout,
        level=getattr(logging, level.upper(), logging.INFO),
    )
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso", utc=True),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            structlog.processors.JSONRenderer(),
        ],
        wrapper_class=structlog.make_filtering_bound_logger(
            getattr(logging, level.upper(), logging.INFO)
        ),
        logger_factory=structlog.stdlib.LoggerFactory(),
        cache_logger_on_first_use=True,
    )


def get_logger(name: str) -> structlog.stdlib.BoundLogger:
    return structlog.get_logger(name)
```

Why structlog: it gives you JSON logs in production (great for
Datadog/Sentry/etc) and pretty logs locally if you swap the renderer.
We always want our log lines machine-parsable in the cloud.

## 6. The health router

Create `backend/routers/health.py`:

```python
# in backend/routers/health.py
from fastapi import APIRouter
from pydantic import BaseModel

from utils.settings import get_settings

router = APIRouter(prefix="/v1", tags=["health"])


class HealthResponse(BaseModel):
    status: str
    env: str
    version: str


@router.get("/health", response_model=HealthResponse)
def health() -> HealthResponse:
    s = get_settings()
    return HealthResponse(status="ok", env=s.env, version="0.0.1")
```

## 7. The application entry point

Create `backend/main.py`:

```python
# in backend/main.py
import os
from contextlib import asynccontextmanager

import sentry_sdk
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from utils.logging import configure_logging, get_logger
from utils.settings import get_settings


settings = get_settings()
configure_logging(settings.log_level)
log = get_logger(__name__)

if settings.sentry_dsn:
    sentry_sdk.init(
        dsn=settings.sentry_dsn,
        environment=settings.env,
        traces_sample_rate=0.05,
    )


@asynccontextmanager
async def lifespan(app: FastAPI):
    log.info("startup", env=settings.env, port=settings.port)
    # Future parts will start Redis pools, HTTP clients, etc., here.
    yield
    log.info("shutdown")


app = FastAPI(
    title=os.environ.get("APP_NAME", "<<YOUR_BRAND>> API"),
    version="0.0.1",
    lifespan=lifespan,
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.allowed_origins_list,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# === Routers (the only thing that touches main.py from now on) ===
from routers import health  # noqa: E402

app.include_router(health.router)


# Root, useful for "is this thing on" checks at the load balancer.
@app.get("/")
def root():
    return {"name": app.title, "version": app.version}
```

## 8. Run it

```bash
cd backend
uvicorn main:app --reload --host 0.0.0.0 --port 8080 --env-file .env
```

In a second terminal:

```bash
http :8080/v1/health
# or
curl localhost:8080/v1/health | jq
```

You should see:

```json
{"status": "ok", "env": "development", "version": "0.0.1"}
```

Open <http://localhost:8080/docs> in a browser to see the auto-generated
Swagger UI. This is the cheapest, best onboarding gift you can give to
yourself and any future collaborator.

## 9. Make it reachable from the internet (ngrok)

The mobile app, Stripe webhooks, and OAuth callbacks all need a public
HTTPS URL. We'll use **ngrok** for development.

```bash
brew install ngrok/ngrok/ngrok
ngrok config add-authtoken <YOUR_NGROK_TOKEN>
```

In your ngrok dashboard, claim a **static domain** (free tier gives
you one). Then run:

```bash
ngrok http --domain=<<YOUR_BRAND>>-dev.ngrok-free.app 8080
```

Now your local backend is reachable at
`https://<<YOUR_BRAND>>-dev.ngrok-free.app/v1/health` from anywhere on
the internet. Save that URL in `.env`:

```
BASE_API_URL=https://<<YOUR_BRAND>>-dev.ngrok-free.app
```

## 10. Add a Makefile (or `justfile`) for common chores

Create `backend/Makefile`:

```makefile
.PHONY: install run lint format test

install:
	python3.11 -m venv .venv
	. .venv/bin/activate && pip install -U pip && pip install -r requirements.txt

run:
	. .venv/bin/activate && uvicorn main:app --reload --host 0.0.0.0 --port 8080 --env-file .env

format:
	. .venv/bin/activate && black --line-length 120 --skip-string-normalization .

lint:
	. .venv/bin/activate && python -m pyflakes .

test:
	. .venv/bin/activate && pytest -q
```

So that from now on, `make run`, `make format`, `make test`.

## 11. Commit

```bash
git add backend/ scripts/ .vscode/
git commit -m "feat(part-03): fastapi skeleton with health endpoint"
git push
```

## What you should have right now

- [ ] `python3.11`, all deps installed in a `.venv`.
- [ ] `uvicorn main:app --reload` starts cleanly.
- [ ] `GET /v1/health` returns 200 with JSON.
- [ ] `/docs` page works.
- [ ] ngrok forwards a public HTTPS URL to your laptop.
- [ ] `make run`, `make format` work.

You now have a real backend. Everything else in this manual just adds
routers, utilities, and dependencies to this skeleton.

---

Next: [Part 04 — Firebase Auth & Firestore](./04-firebase-and-firestore.md).
