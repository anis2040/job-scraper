# Hosting JobPilot AI (low traffic)

Guide for hosting this app for yourself and a few friends at the lowest practical cost.

Your app is a **single Flask server** that serves the API, user data, and (after a build) the React UI. It stores everything on disk (`profiles/<user_id>/` with SQLite), needs **LaTeX (`pdflatex`)** for PDF resumes/cover letters, and supports **Google sign-in** so each friend gets isolated data.

**Recommended path:** small VPS or home PC + **Docker Compose**. The image bundles Python, Gunicorn, TeX, and the React build. Profile data is bind-mounted to `./data/profiles` so rebuilds do not wipe users.

**Scaling beyond one machine?** In **prod** (`FLASK_DEBUG=false`), follow the phased plan in [STORAGE_MIGRATION.md](STORAGE_MIGRATION.md) — PDFs to external storage first; shared SQL later (not SQLite on S3). **Dev** keeps the current on-disk layout.

---

## Docker Compose (recommended)

### How profiles stay safe

| Location | What happens on rebuild / `compose down` |
|----------|------------------------------------------|
| `./data/profiles` on the **host** (bind mount) | **Kept** — this is your durable data |
| Files inside the image | Replaced on rebuild (app code only) |

Do **not** use `docker compose down -v` unless you intend to delete data (this setup uses a bind mount, not a named volume, so `-v` is less relevant — deleting `./data/profiles` is what wipes users).

Back up with:

```bash
tar -czf jobpilot-profiles-$(date +%F).tar.gz data/profiles
```

### 1. Prerequisites

- Docker Engine + Docker Compose plugin
- (Optional) a domain + reverse proxy for HTTPS in production

### 2. Configure env

```bash
cp .env.example .env
```

Edit `.env`:

| Variable | Purpose |
|----------|---------|
| `SECRET_KEY` | Long random string (`python -c "import secrets; print(secrets.token_hex(32))"`) |
| `FLASK_DEBUG` | `false` in production |
| `AUTH_DISABLED` | `0` (force Google login) |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | From Google Cloud Console |
| `OAUTH_REDIRECT_URI` | Must match Google redirect URI exactly |

Local Docker example:

```env
OAUTH_REDIRECT_URI=http://localhost:5050/auth/callback/google
```

Production example:

```env
OAUTH_REDIRECT_URI=https://yourdomain.com/auth/callback/google
```

On Linux, set your host user so the bind mount is writable:

```bash
echo "HOST_UID=$(id -u)" >> .env
echo "HOST_GID=$(id -g)" >> .env
```

(macOS Docker Desktop usually works with the defaults `1000:1000`.)

### 3. Google OAuth

1. Create a project in [Google Cloud Console](https://console.cloud.google.com/).
2. Configure **OAuth consent screen** (External is fine for friends).
3. Create an **OAuth 2.0 Web client**.
4. Add the same redirect URI as `OAUTH_REDIRECT_URI`.
5. If the app is in **Testing** mode, add friends as **test users**.

Each Google account gets `profiles/google_<id>/` under `./data/profiles`.

### 4. Start

```bash
mkdir -p data/profiles
docker compose up -d --build
```

Open [http://localhost:5050/login](http://localhost:5050/login).

Useful commands:

```bash
docker compose logs -f app    # logs
docker compose restart app    # restart
docker compose up -d --build  # rebuild after git pull (keeps ./data/profiles)
docker compose down           # stop (keeps ./data/profiles)
```

### 5. Production HTTPS (VPS)

Still need a host (Oracle free VM, Hetzner, etc.). Install Docker there, then put **Caddy** or nginx on the host (or a second Compose service) in front of port 5050.

Example Caddyfile on the host:

```
yourdomain.com {
    reverse_proxy localhost:5050
}
```

Point DNS A record at the server. Set `OAUTH_REDIRECT_URI=https://yourdomain.com/auth/callback/google` and add that URI in Google Cloud.

### 6. Smoke test

1. Open `/login` and sign in with Google  
2. Complete setup wizard  
3. Run a small job fetch  
4. Generate a test resume (confirms `pdflatex` inside the container)

### Migrating existing local profiles into Docker

If you already have a local `profiles/` folder:

```bash
mkdir -p data
cp -R profiles data/profiles
# or: mv profiles data/profiles
docker compose up -d --build
```

---

## Best low-cost host options

| Option | Approx. cost | Best for | Caveat |
|--------|-------------|----------|--------|
| **Oracle Cloud Always Free VM** | **$0/mo** | Cheapest real server | Signup can be finicky; you manage the VM |
| **Hetzner CX22 (or similar)** | **~€4–5/mo** | Easiest “real VPS” experience | Small monthly fee |
| **Old laptop / home PC + Tailscale** | **$0** (electricity only) | Private use among friends | You maintain uptime, backups, and networking |
| **Fly.io / Railway with a volume** | **~$0–7/mo** | Managed deploy without much sysadmin | Needs persistent volume; more config than a VPS |
| **Render free tier** | $0 | — | **Poor fit**: sleeps when idle, ephemeral disk |

Docker Compose works on any of the VPS / home options above. The app is **not** a good fit for serverless (long-running fetch jobs, local files, SQLite, `pdflatex`).

---

## Manual deploy (without Docker)

Use this only if you prefer bare metal / systemd.

### Environment variables

Same as `.env.example` (root `.env` on the server — not committed).

With `FLASK_DEBUG=false`, session cookies become secure (required for HTTPS). **Do not** use the default `SECRET_KEY` in production.

### Build frontend

```bash
npm ci --prefix frontend
npm --prefix frontend run build
```

Flask serves the SPA when `frontend/dist` exists (`/`, `/login`, `/app`, etc.). Vite assets are served from `/assets/`.

### System packages (Ubuntu/Debian)

```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip \
  texlive-latex-extra texlive-fonts-recommended
```

### Gunicorn + systemd

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

`/etc/systemd/system/jobpilot.service`:

```ini
[Unit]
Description=JobPilot AI
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/job-scraper
EnvironmentFile=/home/ubuntu/job-scraper/.env
ExecStartPre=/home/ubuntu/job-scraper/.venv/bin/python -c "from startup import run_startup; run_startup()"
ExecStart=/home/ubuntu/job-scraper/.venv/bin/gunicorn -w 2 -b 127.0.0.1:5050 web:app
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now jobpilot
```

Put Caddy/nginx in front for HTTPS. Back up the `profiles/` directory regularly.

---

## Home server + Tailscale (private)

1. Run `docker compose up -d --build` on a home machine.  
2. Use [Tailscale](https://tailscale.com/) so friends join your private network.  
3. Open `http://<tailscale-ip>:5050/login` (or Tailscale Funnel for HTTPS).

Cost: **$0/month** for hosting; you handle uptime and backups of `./data/profiles`.

---

## Ongoing costs

| Item | Cost |
|------|------|
| VPS | $0 (Oracle) or ~€4–5/mo (Hetzner) |
| Domain | ~$10–15/year (optional with Tailscale) |
| AI APIs | $0 if everyone uses free Groq/Gemini tiers |
| Your time | Updates, backups, occasional restarts |

Server `.env` is only for OAuth and `SECRET_KEY`. Each person enters their own AI keys in **AI Settings**; those are stored under their folder in `./data/profiles`.

---

## Pre-launch checklist

- [ ] Copied `.env.example` → `.env` with strong `SECRET_KEY`
- [ ] `FLASK_DEBUG=false`, `AUTH_DISABLED=0`
- [ ] Google OAuth client + matching `OAUTH_REDIRECT_URI`
- [ ] Friends added as OAuth test users (if app is in Testing mode)
- [ ] `mkdir -p data/profiles` before first start
- [ ] `docker compose up -d --build` healthy
- [ ] `/login` works; resume PDF generation works
- [ ] Backup plan for `./data/profiles`
- [ ] (Production) HTTPS reverse proxy in front of port 5050
