# Part 03 — FastAPI Skeleton & First Endpoint

> Goal of this part: stand up a FastAPI service that runs locally,
> serves `GET /v1/health`, has structured logging, lives behind a
> tunnel that the internet can reach, and is wired up so future parts
> can drop in routers without touching `main.py`.

## 1. Create the virtual environment

In **PowerShell**:

```powershell
cd backend
py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -V    # → Python 3.11.x
```

The `py -3.11` launcher comes with the Chocolatey `python311`
install and picks the right interpreter even if you also have a
newer Python on your system. If `py` isn't on PATH, use
`python -m venv .venv` after confirming `python --version` is 3.11.

You'll do `.\.venv\Scripts\Activate.ps1` every time you open a new
PowerShell window in this folder. Your prompt should now start with
`(.venv)`.

> **If activation fails with "running scripts is disabled"** —
> you skipped the execution-policy step in Part 02. Run:
> ```powershell
> Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
> ```
> and try again.

> **If you prefer `cmd.exe`** instead of PowerShell, the activation
> command is `.\.venv\Scripts\activate.bat`. Everything else in
> this manual is PowerShell.

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

```powershell
python -m pip install --upgrade pip
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
GEMINI_API_KEY=
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

```powershell
Copy-Item .env.template .env
# Generate a random ADMIN_KEY for local dev:
$key = -join ((1..32) | ForEach-Object { '{0:x}' -f (Get-Random -Maximum 16) })
Add-Content .env "ADMIN_KEY=$key"
```

(Or, if you'd rather use OpenSSL: `openssl rand -hex 16 | Out-File
-Encoding ASCII -Append .env` — but `Add-Content` is fewer surprises
on Windows because it doesn't add a BOM.)

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

    gemini_api_key: str = ""
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

With the venv active:

```powershell
cd backend
uvicorn main:app --reload --host 0.0.0.0 --port 8080 --env-file .env
```

In a **second** PowerShell window:

```powershell
http :8080/v1/health
# or, using the curl shipped with Windows 10+ (it's really curl.exe):
curl.exe http://localhost:8080/v1/health
```

> **Heads-up:** in PowerShell, the alias `curl` points to
> `Invoke-WebRequest`, which is *not* curl-compatible and prints a
> different object shape. Always type `curl.exe` (or use `http` from
> `httpie`) when you mean the real curl.

You should see:

```json
{"status": "ok", "env": "development", "version": "0.0.1"}
```

If Windows Defender Firewall asks "Allow Python to communicate on
networks?" — say yes for **Private networks** only. The public-
internet exposure happens through ngrok, not through your laptop's
IP directly.

Open <http://localhost:8080/docs> in a browser to see the auto-generated
Swagger UI. This is the cheapest, best onboarding gift you can give to
yourself and any future collaborator.

## 9. Make it reachable from the internet (ngrok)

The mobile app, Stripe webhooks, and OAuth callbacks all need a public
HTTPS URL. We'll use **ngrok** for development.

Install via Chocolatey, then add your auth token:

```powershell
choco install -y ngrok
ngrok config add-authtoken <YOUR_NGROK_TOKEN>
```

In your ngrok dashboard, claim a **static domain** (free tier gives
you one). Then in a third PowerShell window run:

```powershell
ngrok http --domain=<<YOUR_BRAND>>-dev.ngrok-free.app 8080
```

Now your local backend is reachable at
`https://<<YOUR_BRAND>>-dev.ngrok-free.app/v1/health` from anywhere on
the internet. Save that URL in `.env`:

```
BASE_API_URL=https://<<YOUR_BRAND>>-dev.ngrok-free.app
```

## 10. Add per-task helper scripts (PowerShell)

`make` doesn't ship with Windows, and chasing a working `mingw32-make`
is more pain than it's worth. We'll use `.ps1` scripts in `backend/`
that wrap the common chores. Each one expects you to have already
activated the venv.

`backend\tools\install.ps1`:

```powershell
$ErrorActionPreference = "Stop"
py -3.11 -m venv .venv
& ".\.venv\Scripts\Activate.ps1"
python -m pip install -U pip
pip install -r requirements.txt
```

`backend\tools\run.ps1`:

```powershell
$ErrorActionPreference = "Stop"
& ".\.venv\Scripts\Activate.ps1"
uvicorn main:app --reload --host 0.0.0.0 --port 8080 --env-file .env
```

`backend\tools\format.ps1`:

```powershell
$ErrorActionPreference = "Stop"
& ".\.venv\Scripts\Activate.ps1"
black --line-length 120 --skip-string-normalization .
```

`backend\tools\test.ps1`:

```powershell
$ErrorActionPreference = "Stop"
& ".\.venv\Scripts\Activate.ps1"
pytest -q
```

Usage (from `backend/`):

```powershell
.\tools\run.ps1
.\tools\format.ps1
.\tools\test.ps1
```

> **If you really want `make`:** install [GNU Make for Windows] via
> `choco install make`, then write the equivalent Makefile using
> `.venv\Scripts\activate.bat &&` instead of `source .venv/bin/activate`.
> Most engineers find the `.ps1` flavor easier to read.

[GNU Make for Windows]: https://community.chocolatey.org/packages/make

## 11. Commit

```powershell
git add backend\ scripts\ .vscode\
git commit -m "feat(part-03): fastapi skeleton with health endpoint"
git push
```

## What you should have right now

- [ ] Python 3.11 venv at `backend\.venv`, activated in your shell.
- [ ] `uvicorn main:app --reload` starts cleanly.
- [ ] `GET /v1/health` returns 200 with JSON.
- [ ] `/docs` page works.
- [ ] ngrok forwards a public HTTPS URL to your laptop.
- [ ] `.\tools\run.ps1`, `.\tools\format.ps1` work.

You now have a real backend. Everything else in this manual just adds
routers, utilities, and dependencies to this skeleton.

---

Next: [Part 04 — Firebase Auth & Firestore](./04-firebase-and-firestore.md).
