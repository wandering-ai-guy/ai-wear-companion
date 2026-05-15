# Part 02 — Local Toolchain & Repo Layout

> Goal of this part: install every command-line tool you'll need, lay
> out the repository directories, and configure your editor so you can
> work on backend, app, and infrastructure side-by-side.

This part is mechanical. Once it's done you won't think about it again.

## 1. OS assumption (Windows)

The rest of this manual assumes you're developing on **Windows 10
(build 19044+) or Windows 11**. Everything below works on both. We use
two shells side-by-side:

- **PowerShell 7** — for the backend, Python, gcloud, Firebase CLI,
  and most day-to-day work. This is the one you'll have open 90% of
  the time.
- **WSL 2 (Ubuntu)** — for the handful of POSIX-only tasks: building
  Docker images quickly, running shell scripts, running the BLE
  smoke tools, and (later) running the Linux build of the pusher
  service locally.

> **The one Windows caveat: iOS.** You cannot build an iOS app on
> Windows. Apple requires macOS and Xcode for any iOS build, ad-hoc
> install, or TestFlight upload. We have three workable options
> covered in Part 21:
>
> 1. **Cloud builders** (Codemagic, Bitrise, GitHub Actions
>    `macos-latest` runners) — sign and ship from CI; you never touch
>    a Mac yourself.
> 2. **A rented Mac in the cloud** (MacStadium, MacInCloud) — RDP
>    into a Mac when you need Xcode.
> 3. **Borrow a friend's Mac** for the App Store submission day.
>
> Everything else — backend, Android app, BLE testing, deployment —
> runs natively on Windows. Don't let "you need a Mac for iOS"
> block you from starting.

## 2. Enable WSL 2 once

Open **PowerShell as Administrator** and run:

```powershell
wsl --install -d Ubuntu-22.04
```

This downloads Ubuntu and reboots once. After reboot, an Ubuntu
terminal will open and ask you to set a Linux username and password
(this is separate from your Windows login).

When that's done, from PowerShell verify:

```powershell
wsl -l -v
# NAME             STATE           VERSION
# Ubuntu-22.04     Running         2
```

Pin the Ubuntu icon to your taskbar. You'll need it occasionally.

## 3. Core toolchain (Windows side)

Install a **package manager**: open PowerShell as Administrator and
run:

```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = `
  [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString( `
  'https://community.chocolatey.org/install.ps1'))
```

That installs **Chocolatey**. We use it because it has every package
we need, can be scripted, and updates with `choco upgrade all`. If
you prefer **winget** (built into Windows 11), the package names are
similar — substitute as you go.

Then install the toolchain:

```powershell
choco install -y `
  git `
  python311 `
  nodejs-lts `
  ffmpeg `
  redis-64 `
  jq `
  gh `
  gcloudsdk `
  openssl.light `
  vscode `
  docker-desktop `
  microsoft-windows-terminal
```

What each one is for:

- **git, gh** — source control + GitHub CLI.
- **python311** — Python 3.11 runtime. Pin 3.11; some dependencies
  break on 3.12.
- **nodejs-lts** — Firebase CLI, mobile build helpers.
- **ffmpeg** — audio decoding/encoding. Required to debug wearable
  audio offline.
- **redis-64** — local Redis server for development (we run it as a
  background Windows service).
- **jq** — JSON munging in scripts.
- **gcloudsdk** — the `gcloud` CLI for talking to your GCP project.
- **openssl.light** — generating random secrets.
- **vscode** — the editor.
- **docker-desktop** — building/running containers locally
  (Part 19). You'll be asked to enable WSL 2 integration the first
  time you launch it; say yes.
- **windows-terminal** — a much nicer terminal than the legacy
  console. Open it and add a "PowerShell 7" tab + an "Ubuntu" tab.

> **Restart PowerShell** after each big install so the new `PATH`
> entries are picked up. If `python --version` still says "not
> found," sign out of Windows and back in.

Test:

```powershell
git --version
python --version          # → Python 3.11.x
node --version
gcloud --version
gh --version
ffmpeg -version
redis-cli --version
docker --version
```

If any of those don't show a version, fix it before moving on.

### Opus audio codec on Windows

The `opuslib` Python package needs `libopus` available at runtime.
The simplest path is to ship `opus.dll` next to your Python venv,
or install via `vcpkg`:

```powershell
git clone https://github.com/microsoft/vcpkg %USERPROFILE%\vcpkg
cd %USERPROFILE%\vcpkg
.\bootstrap-vcpkg.bat
.\vcpkg install opus:x64-windows
.\vcpkg integrate install
# Copy the DLL to a folder that's on PATH:
copy installed\x64-windows\bin\opus.dll C:\Windows\System32\
```

If that feels heavy, you can also do all the Opus-decoding work
inside WSL (where `apt install libopus0` is one line) — see Part
07's "Audio processing utilities."

### Python tooling

```powershell
python -m pip install --upgrade pip
python -m pip install --user pipx
python -m pipx ensurepath
# restart PowerShell here so pipx is on PATH
pipx install black
pipx install poetry
pipx install httpie       # nicer curl
pip install --user pre-commit
```

### Firebase CLI

```powershell
npm install -g firebase-tools
```

## 4. Core toolchain (WSL/Ubuntu side, for parity)

Open the Ubuntu terminal and install the same set so scripts work
there too:

```bash
sudo apt update && sudo apt install -y \
  git python3.11 python3.11-venv python3-pip \
  ffmpeg libopus0 libopus-dev redis-tools jq \
  build-essential
```

You won't need most of this most of the time — but when a future
script in this manual is a `.sh` file, run it from this side.

## 5. Mobile toolchain (Flutter for Android)

Install Flutter using the official Windows guide:
<https://docs.flutter.dev/get-started/install/windows/mobile>. Use
the **stable** channel.

The short version:

1. Download the Flutter SDK zip from
   <https://docs.flutter.dev/release/archive>.
2. Extract to `C:\src\flutter` (avoid spaces in the path; "Program
   Files" will break things).
3. Add `C:\src\flutter\bin` to your **user** `PATH`:
   ```powershell
   [Environment]::SetEnvironmentVariable("PATH",
     "$env:USERPROFILE\AppData\Local\Pub\Cache\bin;C:\src\flutter\bin;" + $env:PATH,
     "User")
   ```
   Restart PowerShell so the change takes effect.
4. Run:
   ```powershell
   flutter doctor
   ```

You'll then need:

1. **Android Studio** — Chocolatey can install it (`choco install -y
   androidstudio`) or download from
   <https://developer.android.com/studio>. Open it once and let it
   download the Android SDK + a default emulator image.
2. From Android Studio → SDK Manager → "SDK Tools" tab, check
   **Android SDK Command-line Tools (latest)** and apply.
3. Accept the licenses:
   ```powershell
   flutter doctor --android-licenses
   ```
   Press `y` to all prompts.
4. **iOS toolchain row will show a red X** in `flutter doctor` on
   Windows. That is expected. Don't try to make it green; iOS builds
   happen in the cloud (Part 21).

Re-run `flutter doctor` until everything except the iOS line is a
green check. The Visual Studio (C++ desktop) row can also stay red
unless you plan to ship a Windows desktop build.

## 6. Sign in to your clouds

In PowerShell:

```powershell
gcloud auth login
gcloud config set project <<YOUR_GCP_PROJECT_ID>>
gcloud auth application-default login --project <<YOUR_GCP_PROJECT_ID>>
firebase login
gh auth login
```

`gcloud auth login` will open your default browser. The credentials
file ends up at
`$env:APPDATA\gcloud\application_default_credentials.json` — that's
the equivalent of `~/.config/gcloud/...` on Mac/Linux. The Google
SDK picks it up automatically.

Test:

```powershell
gcloud projects list           # should include your project
firebase projects:list         # should include your project
gh repo view                   # should show the repo you cloned
```

## 7. Editor: VS Code (recommended)

Already installed via Chocolatey. Install extensions:

- **Python** (Microsoft)
- **Pylance** (Microsoft)
- **Black Formatter** (Microsoft)
- **Flutter** (Dart Code)
- **Dart** (Dart Code)
- **Even Better TOML**
- **Docker** (Microsoft)
- **WSL** (Microsoft) — so you can open WSL folders inside VS Code.
- **GitLens**
- **GitHub Pull Requests**
- **YAML** (Red Hat)
- **REST Client** (humao) — useful for hitting your own API.
- **EditorConfig**

From the repo root, create `.vscode/settings.json`:

```json
{
  "editor.formatOnSave": true,
  "editor.rulers": [120],
  "files.eol": "\n",
  "[python]": {
    "editor.defaultFormatter": "ms-python.black-formatter",
    "editor.tabSize": 4
  },
  "[dart]": {
    "editor.defaultFormatter": "Dart-Code.dart-code",
    "editor.tabSize": 2
  },
  "python.analysis.typeCheckingMode": "basic",
  "python.testing.pytestEnabled": true,
  "python.testing.pytestArgs": ["backend/tests"],
  "files.exclude": {
    "**/.venv": true,
    "**/__pycache__": true,
    "**/.dart_tool": true
  }
}
```

The `"files.eol": "\n"` line is important on Windows: it forces LF
line endings, which is what Git on the server expects and what the
Linux Docker images expect when they run your Python files. Without
it, you'll occasionally see "no such file or directory" errors that
are really CRLF-in-shebang errors.

Also create `.gitattributes` at the repo root so collaborators don't
re-introduce CRLFs:

```
* text=auto eol=lf
*.bat text eol=crlf
*.ps1 text eol=crlf
```

## 8. Repository layout

This is the directory structure you'll grow into. Create the empty
folders now so future parts have a place to land.

In PowerShell, from the repo root:

```powershell
cd <<YOUR_BRAND>>
$dirs = @(
  "backend\routers", "backend\utils", "backend\database",
  "backend\models", "backend\tests\unit", "backend\tests\integration",
  "backend\migrations", "backend\scripts",
  "pusher", "diarizer", "agent-proxy",
  "app\lib\features", "app\lib\services", "app\lib\models",
  "app\lib\common", "app\lib\l10n", "app\assets",
  "branding\<<YOUR_BRAND>>\ios", "branding\<<YOUR_BRAND>>\android",
  "branding\<<YOUR_BRAND>>\web", "branding\<<YOUR_BRAND>>\mobile",
  "infra\terraform", "infra\helm", "infra\docker",
  "docs", "scripts", "firmware"
)
foreach ($d in $dirs) { New-Item -ItemType Directory -Force -Path $d | Out-Null }

$files = @(
  "backend\__init__.py", "backend\main.py", "backend\requirements.txt",
  "backend\Dockerfile", "backend\.env.template",
  "backend\routers\__init__.py", "backend\utils\__init__.py",
  "backend\database\__init__.py", "backend\models\__init__.py"
)
foreach ($f in $files) { New-Item -ItemType File -Force -Path $f | Out-Null }
```

The reasoning behind the layout:

| Folder | What it holds |
|--------|---------------|
| `backend/` | The main FastAPI service. |
| `backend/routers/` | One file per "feature" (e.g. `conversations.py`). Each defines an `APIRouter` and is included in `main.py`. |
| `backend/utils/` | Business logic, importable from routers. |
| `backend/database/` | Anything that talks to Firestore/Redis/etc. |
| `backend/models/` | Pydantic models (input/output schemas). |
| `backend/tests/` | Pytest tests. Unit = no network. Integration = Firestore/Redis. |
| `pusher/` | Long-running fan-out service. Mirrors backend layout. |
| `diarizer/` | GPU service for speaker embeddings. |
| `agent-proxy/` | (Optional) WebSocket bridge to per-user agent VMs. |
| `app/` | Flutter mobile app. |
| `branding/` | Logos, splash screens, app icons. |
| `infra/` | Terraform + Helm + Docker Compose. |
| `firmware/` | (Optional) Custom wearable firmware. |
| `docs/` | This manual lives here. Add other docs as you go. |
| `scripts/` | Repo-wide helper scripts (e.g. lint, format-all, deploy-dev). |

## 9. The dependency hierarchy rule

Burn this rule into your brain. You will follow it for the entire
backend:

```
database/  →  utils/  →  routers/  →  main.py
```

- **`database/`** never imports from `utils/`, `routers/`, or `main.py`.
- **`utils/`** can import from `database/` only.
- **`routers/`** can import from `utils/` and `database/`.
- **`main.py`** imports from `routers/` and wires the app together.

Two router files **never import each other**. If they need to share
code, push that code down into `utils/`.

This sounds bureaucratic but it pays off the very first time you
realize that a circular import has nothing to do with your bug.

## 10. Pre-commit hooks

These keep formatting consistent without you having to think.

Create `.pre-commit-config.yaml` at the repo root:

```yaml
repos:
  - repo: https://github.com/psf/black
    rev: 24.10.0
    hooks:
      - id: black
        language_version: python3.11
        args: ["--line-length", "120", "--skip-string-normalization"]
        files: "^backend/|^pusher/|^diarizer/|^agent-proxy/"
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: end-of-file-fixer
      - id: trailing-whitespace
      - id: check-yaml
      - id: check-added-large-files
        args: ["--maxkb=2048"]
```

Then, in PowerShell:

```powershell
pre-commit install
pre-commit run --all-files
```

The first run will reformat everything to canonical style. After that,
every commit will run the hooks automatically.

## 11. Useful repo-root scripts

We provide **two flavors** of each helper: a PowerShell `.ps1` for
day-to-day Windows use, and a bash `.sh` so the same script works in
CI on Linux runners (Part 18). Keep them in sync.

`scripts/format-all.ps1`:

```powershell
$ErrorActionPreference = "Stop"
black --line-length 120 --skip-string-normalization backend pusher diarizer agent-proxy
Push-Location app
dart format --line-length 120 lib test
Pop-Location
```

`scripts/run-backend.ps1`:

```powershell
$ErrorActionPreference = "Stop"
Set-Location backend
& ".\.venv\Scripts\Activate.ps1"
uvicorn main:app --reload --host 0.0.0.0 --port 8080 --env-file .env
```

The bash twins for CI (Linux only):

`scripts/format-all.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
black --line-length 120 --skip-string-normalization backend pusher diarizer agent-proxy
( cd app && dart format --line-length 120 lib test )
```

`scripts/run-backend.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
cd backend
source .venv/bin/activate
uvicorn main:app --reload --host 0.0.0.0 --port 8080 --env-file .env
```

> **Running `.ps1` scripts.** Windows blocks unsigned scripts by
> default. Run this once as your user:
> ```powershell
> Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
> ```
> Then `.\scripts\run-backend.ps1` works.

## 12. Make sure Redis is actually running

Chocolatey installs Redis as a Windows service:

```powershell
Start-Service Redis
Get-Service Redis        # STATUS should be Running
redis-cli ping           # should print PONG
```

If `Redis` is not a service name (older `redis-64` packages don't
register it), start the server manually:

```powershell
redis-server --service-install
redis-server --service-start
```

## 13. Commit and push

```powershell
git add .
git commit -m "chore(part-02): toolchain, layout, pre-commit"
git push
```

## What you should have right now

- [ ] PowerShell 7 + Windows Terminal + WSL 2 (Ubuntu) all working.
- [ ] All CLI tools (`git`, `python --version` = 3.11.x, `flutter`,
  `gcloud`, `firebase`, `gh`, `redis-cli`, `ffmpeg`, `docker`)
  installed and on PATH.
- [ ] `flutter doctor` is green except for the iOS line.
- [ ] Logged into GCP, Firebase, GitHub from the CLI.
- [ ] VS Code with all the extensions and a working `settings.json`.
- [ ] `.gitattributes` forces LF line endings.
- [ ] Folder skeleton committed to your repo.
- [ ] Pre-commit hooks installed and run.
- [ ] Redis service running locally (`redis-cli ping` returns
  `PONG`).

---

Next: [Part 03 — FastAPI Skeleton & First Endpoint](./03-fastapi-skeleton.md).
