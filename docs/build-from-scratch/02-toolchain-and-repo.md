# Part 02 — Local Toolchain & Repo Layout

> Goal of this part: install every command-line tool you'll need, lay
> out the repository directories, and configure your editor so you can
> work on backend, app, and infrastructure side-by-side.

This part is mechanical. Once it's done you won't think about it again.

## 1. Pick (and stick to) one OS

Either macOS or Linux. Windows works for the *backend* (use WSL2), but
the iOS app **must** be built on macOS, and the BLE testing flow is
much smoother on macOS too. If you have any choice in the matter,
develop on a Mac.

The rest of this manual assumes a Mac (or Linux for the backend-only
parts).

## 2. Core toolchain

Open Terminal and install Homebrew if you don't have it:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Then install the toolchain in one go:

```bash
brew install \
  git \
  python@3.11 \
  pyenv \
  pipx \
  node \
  ffmpeg \
  opus \
  redis \
  jq \
  gh \
  google-cloud-sdk \
  openssl \
  watchman
```

What each one is for:

- **git, gh** — source control + GitHub CLI.
- **python@3.11, pyenv** — Python runtime. Pin 3.11; some dependencies
  break on 3.12.
- **pipx** — installs Python CLIs in isolated environments.
- **node** — Firebase CLI, mobile build helpers.
- **ffmpeg, opus** — audio decoding/encoding. Required to debug
  wearable audio offline.
- **redis** — local Redis server for development.
- **jq** — JSON munging in scripts.
- **google-cloud-sdk (gcloud)** — talk to your GCP project from the
  command line.
- **openssl** — generate random secrets.
- **watchman** — Flutter and React Native need it.

Now install Python tools:

```bash
pipx install black
pipx install poetry
pipx install httpie  # nicer curl
pip3 install --user pre-commit
```

And the Firebase CLI:

```bash
npm install -g firebase-tools
```

## 3. Mobile toolchain

Install Flutter following the official instructions:
<https://docs.flutter.dev/get-started/install/macos>. Use the **stable**
channel.

Quick check:

```bash
flutter doctor
```

You probably need to:

1. **Install Xcode** from the App Store (~10 GB).
2. Run `sudo xcode-select --switch /Applications/Xcode.app`.
3. Run `sudo xcodebuild -runFirstLaunch` and accept the license.
4. **Install CocoaPods** (`brew install cocoapods`).
5. **Install Android Studio** (https://developer.android.com/studio)
   and let it download the Android SDK + emulator.
6. Run `flutter doctor --android-licenses` and answer "y" to all.

Re-run `flutter doctor` until every line has a green check.

## 4. Sign in to your clouds

```bash
gcloud auth login
gcloud config set project <<YOUR_GCP_PROJECT_ID>>
gcloud auth application-default login --project <<YOUR_GCP_PROJECT_ID>>
firebase login
gh auth login
```

Test:

```bash
gcloud projects list           # should include your project
firebase projects:list         # should include your project
gh repo view                   # should show the repo you cloned
```

## 5. Editor: VS Code (recommended)

1. Install [VS Code](https://code.visualstudio.com/).
2. Install these extensions:

   - **Python** (Microsoft)
   - **Pylance** (Microsoft)
   - **Black Formatter** (Microsoft)
   - **Flutter** (Dart Code)
   - **Dart** (Dart Code)
   - **Even Better TOML**
   - **Docker** (Microsoft)
   - **GitLens**
   - **GitHub Pull Requests**
   - **YAML** (Red Hat)
   - **REST Client** (humao) — useful for hitting your own API.
   - **EditorConfig**

3. From the repo root, create `.vscode/settings.json`:

```json
{
  "editor.formatOnSave": true,
  "editor.rulers": [120],
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

## 6. Repository layout

This is the directory structure you'll grow into. Create the empty
folders now so future parts have a place to land:

```bash
cd <<YOUR_BRAND>>
mkdir -p \
  backend/{routers,utils,database,models,tests/unit,tests/integration,migrations,scripts} \
  pusher \
  diarizer \
  agent-proxy \
  app/lib/{features,services,models,common,l10n} \
  app/assets \
  branding/<<YOUR_BRAND>>/{ios,android,web,mobile} \
  infra/{terraform,helm,docker} \
  docs \
  scripts \
  firmware
touch backend/{__init__.py,main.py,requirements.txt,Dockerfile,.env.template} \
      backend/{routers,utils,database,models}/__init__.py
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

## 7. The dependency hierarchy rule

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

## 8. Pre-commit hooks

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

Then:

```bash
pre-commit install
pre-commit run --all-files
```

The first run will reformat everything to canonical style. After that,
every commit will run the hooks automatically.

## 9. Useful repo-root scripts

Create `scripts/format-all.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
black --line-length 120 --skip-string-normalization backend pusher diarizer agent-proxy
( cd app && dart format --line-length 120 lib test )
```

Create `scripts/run-backend.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
cd backend
source .venv/bin/activate
uvicorn main:app --reload --host 0.0.0.0 --port 8080 --env-file .env
```

`chmod +x scripts/*.sh`.

## 10. Commit and push

```bash
git add .
git commit -m "chore(part-02): toolchain, layout, pre-commit"
git push
```

## What you should have right now

- [ ] All CLI tools (`git`, `python3.11`, `flutter`, `gcloud`, `firebase`,
  `gh`, `redis`, `ffmpeg`, `opus`) installed.
- [ ] `flutter doctor` is all green checks.
- [ ] Logged into GCP, Firebase, GitHub from the CLI.
- [ ] VS Code with all the extensions and a working `settings.json`.
- [ ] Folder skeleton committed to your repo.
- [ ] Pre-commit hooks installed and run.

---

Next: [Part 03 — FastAPI Skeleton & First Endpoint](./03-fastapi-skeleton.md).
