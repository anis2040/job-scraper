# JobPilot AI

JobPilot AI helps you search jobs, score them against your profile, and generate tailored resumes and cover letters.

## Docker (hosted / friends)

See [docs/HOSTING.md](docs/HOSTING.md) for the full guide. Short version:

```bash
cp .env.example .env   # set SECRET_KEY + Google OAuth
mkdir -p data/profiles # durable user data (survives rebuilds)
docker compose up -d --build
# open http://localhost:5050/login
```

Profiles, SQLite DBs, and generated PDFs live in `./data/profiles` on the host.

For the planned move to object storage (PDFs) and shared SQL (scaling), see [docs/STORAGE_MIGRATION.md](docs/STORAGE_MIGRATION.md).

## React Dev Mode

Use the root dev launcher when you want the React app and Flask backend together.

```bash
npm run dev
```

What it does automatically:

- Creates `.venv` if it does not exist yet
- Installs Python packages from `requirements.txt`
- Installs frontend packages in `frontend/`
- Starts Flask on `http://localhost:5050`
- Starts Vite on `http://localhost:5173`

This works on both macOS and Windows as long as `python`/`python3` and `node`/`npm` are installed.

If you prefer OS-specific launchers:

| macOS | Windows |
|---|---|
| `./dev.sh` | `dev.bat` |

> If `dev.sh` is not executable, run `chmod +x dev.sh` once.

## Legacy Flask Launcher

The old server-rendered UI is still available and uses the original setup/start scripts.

| First run | Later runs |
|---|---|
| `./setup.sh` / `setup.bat` | `./start.sh` / `start.bat` |

That flow opens the Flask app directly at `http://localhost:5050`.

## Common Commands

```bash
# React app + backend
npm run dev

# Backend only
npm run backend

# Frontend only
npm run frontend

# Frontend production build
npm --prefix frontend run build

# Frontend tests
npm --prefix frontend run test
```

After building the frontend, Flask serves the compiled SPA at `http://localhost:5050/app`.

## Setup Wizard

On first launch, the app guides you through:

1. Checking prerequisites such as Node.js, an AI provider, and `pdflatex`
2. Creating or importing your profile
3. Generating search settings from your profile and starting the first fetch

For Windows PDF generation, install [MiKTeX](https://miktex.org/download) and enable on-the-fly package installation.

## Features

- Fetches jobs from multiple sources including LinkedIn, StepStone, Greenhouse, Jobicy, and Himalayas
- Supports multiple profiles with separate jobs, configs, and generated documents
- Scores jobs with keyword matching and optional semantic matching
- Builds ATS-oriented resumes and matching cover letters per job
- Lets you manage AI providers, models, and token usage from the UI

## Multi-user auth (Google)

JobPilot supports Google sign-in for hosted / multi-user use. Each Google account gets an isolated data directory under `profiles/<user_id>/`.

**Local development (default):** if `FLASK_DEBUG` is on and Google OAuth is not configured, auth is disabled and data lives under `profiles/_local/`.

**Enable Google login** — create an OAuth 2.0 Web client in [Google Cloud Console](https://console.cloud.google.com/) and set:

| Env var | Example |
|---|---|
| `GOOGLE_CLIENT_ID` | from Google Cloud |
| `GOOGLE_CLIENT_SECRET` | from Google Cloud |
| `SECRET_KEY` | long random string |
| `OAUTH_REDIRECT_URI` | `http://localhost:5050/auth/callback/google` (dev) |
| `AUTH_DISABLED` | `0` to force auth even in debug; `1` to force local mode |
| `FLASK_DEBUG` | `false` in production |

Authorized redirect URI in Google Cloud must match `OAUTH_REDIRECT_URI` (or the auto URL from Flask).

## AI Providers

JobPilot can use Groq, Anthropic Claude, and Gemini. API keys are entered in the app and saved **per user** to `profiles/<user_id>/.env` (not the project root `.env`). Root `.env` is only for server config (OAuth, `SECRET_KEY`, etc.).

| Provider | Env var (per user) |
|---|---|
| Groq | `GROQ_API_KEY` |
| Anthropic | `ANTHROPIC_API_KEY` |
| Gemini | `GEMINI_API_KEY` or `GOOGLE_API_KEY` |

## Output

Generated files are stored under the active profile for the signed-in user:

```text
profiles/<user_id>/<profile-slug>/<CompanyName>/resumes/
profiles/<user_id>/<profile-slug>/<CompanyName>/cover-letters/
```

