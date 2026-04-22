# LiteLLM + Open WebUI Stack (Docker Compose) + Cloudflare Tunnel

This is a step-by-step, copy/paste-friendly runbook for:

1) running LiteLLM + Open WebUI locally with Docker Compose, and
2) (optionally) publishing Open WebUI to the internet using Cloudflare Tunnel.

Source code lives here as well: https://github.com/dixon400/myllm

This repo uses:
- Docker Compose file: `Docker-compose.yml`
- LiteLLM routing: `litellm-config.yml`
- Secrets/env vars: `.env` (gitignored)

---

## 1) What you are building

Docker Compose runs three services:

1. **litellm** — an OpenAI-compatible gateway that can route to multiple providers (OpenAI, Anthropic, Groq, Ollama, etc.)
2. **open-webui** — a chat UI that talks to LiteLLM
3. **litellm-db** — Postgres for LiteLLM state

Optional: publish Open WebUI at a hostname like `https://chat.yourdomain.com` via Cloudflare Tunnel.

---

## 2) Prerequisites

Install:

- Docker Desktop + `docker compose`
- `git`
- `curl`

Optional:

- `ollama` (only if you want local models)
- `cloudflared` (only if you want a public URL)

Cloudflare note (important):

- If you want a friendly public hostname (example: `chat.yourdomain.com`), you need **a domain name** and you need that domain’s DNS managed by Cloudflare.

Quick checks:

```bash
docker --version
docker compose version
git --version
curl --version
```

If using Cloudflare Tunnel:

```bash
cloudflared --version
```

If using local Ollama:

```bash
ollama --version
ollama list
```

---

## 3) Get the code (mkdir → clone → cd)

Pick a working folder (example: `~/projects`) and clone:

```bash
mkdir -p ~/projects
cd ~/projects
git clone https://github.com/dixon400/myllm.git
cd myllm
```

---

## 4) Create your `.env`

This repo expects secrets and connection strings in `.env`.

1) Copy the template:

```bash
cp .env.example .env
```

2) Edit `.env` and set at minimum:

- `LITELLM_MASTER_KEY`
- Postgres values: `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, `DATABASE_URL`
- Provider keys for the models you actually use (OpenAI / Anthropic / Groq / Ollama Cloud)

Sanity check (prints only variable names that exist, not values):

```bash
grep -E '^(LITELLM_MASTER_KEY|POSTGRES_DB|POSTGRES_USER|POSTGRES_PASSWORD|DATABASE_URL|OPENAI_API_KEY|ANTHROPIC_API_KEY|GROQ_API_KEY|OLLAMA_CLOUD_API_BASE|OLLAMA_CLOUD_API_KEY)=' .env
```

---

## 5) Confirm core config files (quick skim)

### 5.1 Compose wiring

Open WebUI is configured to use LiteLLM:

- `OPENAI_API_BASE_URL=http://litellm:4000/v1`
- `OPENAI_API_KEY=${LITELLM_MASTER_KEY}`

LiteLLM loads routing config from:

- `./litellm-config.yml:/app/config.yaml`

LiteLLM connects to Postgres using:

- `DATABASE_URL` from `.env`

### 5.2 LiteLLM master key

In `litellm-config.yml`, the master key is read from the environment:

```yaml
general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
```

That’s the same `LITELLM_MASTER_KEY` Open WebUI uses.

---

## 6) Start the stack

From the project root:

```bash
docker compose -f Docker-compose.yml up -d --force-recreate
```

Check containers:

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

Expected: `open-webui`, `litellm`, and `litellm-db` are `Up`.

---

## 7) Validate it locally

### 7.1 Open WebUI

- Browse to `http://localhost:3000`

### 7.2 LiteLLM model list

```bash
set -a && source .env && curl -s http://localhost:4000/v1/models \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY"
```

You should see JSON with a non-empty `data` array.

Optional pretty print:

```bash
set -a && source .env && curl -s http://localhost:4000/v1/models \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  | python3 -m json.tool | head -n 80
```

---

## 8) Keep model IDs fresh (provider IDs change)

When models stop working, the most common cause is a model ID changing upstream.

Local Ollama:

```bash
ollama list
```

Groq model catalog:

```bash
set -a && source .env && curl -s https://api.groq.com/openai/v1/models \
  -H "Authorization: Bearer $GROQ_API_KEY" \
  -H "Content-Type: application/json"
```

After editing `litellm-config.yml`, apply changes by recreating containers:

```bash
docker compose -f Docker-compose.yml up -d --force-recreate
```

---

## 9) Publish Open WebUI with Cloudflare Tunnel (optional)

### 9.1 What you need

- A Cloudflare account
- A **domain name** you control
- The domain’s DNS managed by Cloudflare (nameservers pointed to Cloudflare)

If you don’t have a domain yet, you’ll need to purchase/register one first, then add it to Cloudflare and switch the registrar nameservers to Cloudflare.

### 9.2 Login

```bash
cloudflared tunnel login
```

### 9.3 Create a tunnel

```bash
cloudflared tunnel create openwebui
```

### 9.4 Route a hostname to the tunnel

Example hostname:

- `chat.yourdomain.com`

Command:

```bash
cloudflared tunnel route dns openwebui chat.yourdomain.com
```

### 9.5 Create the tunnel config

Create `~/.cloudflared/config.yml`:

```yaml
tunnel: openwebui
credentials-file: ~/.cloudflared/<tunnel-uuid>.json
ingress:
  - hostname: chat.yourdomain.com
    service: http://localhost:3000
  - service: http_status:404
```

Replace `<tunnel-uuid>` with the UUID generated by `cloudflared tunnel create`.

### 9.6 Run the tunnel

```bash
cloudflared tunnel run openwebui
```

To keep it running across reboots (macOS):

```bash
cloudflared service install
cloudflared service start
```

### 9.7 Verify

```bash
dig +short chat.yourdomain.com
curl -I https://chat.yourdomain.com
```

---

## 10) Operations quick-check (daily / after edits)

```bash
set -a && source .env && \
echo "== Docker services ==" && docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}' && \
echo "\n== LiteLLM models ==" && curl -s http://localhost:4000/v1/models -H "Authorization: Bearer $LITELLM_MASTER_KEY" && \
echo "\n\n== LiteLLM logs (tail) ==" && docker logs --tail 80 litellm
```

---

## 11) Troubleshooting

### A) Open WebUI loads, but models/providers are missing

Checks:

```bash
docker logs --tail 200 litellm
set -a && source .env && curl -s http://localhost:4000/v1/models -H "Authorization: Bearer $LITELLM_MASTER_KEY"
```

Fixes:

- Ensure Open WebUI uses `OPENAI_API_KEY=${LITELLM_MASTER_KEY}`.
- Ensure `general_settings.master_key` in `litellm-config.yml` is `os.environ/LITELLM_MASTER_KEY`.
- Recreate and restart:

```bash
docker compose -f Docker-compose.yml up -d --force-recreate
docker compose -f Docker-compose.yml restart open-webui
```

### B) LiteLLM fails with `IsADirectoryError: /app/config.yaml`

Cause:
- Compose bind mount references wrong host filename and Docker created a directory.

Checks:

```bash
ls -la ./litellm-config.yml ./litellm-config.yaml
grep -n "litellm-config" Docker-compose.yml
```

Fix:
- Use correct mount:

```yaml
- ./litellm-config.yml:/app/config.yaml
```

Then recreate:

```bash
docker compose -f Docker-compose.yml up -d --force-recreate litellm
```

### C) Works locally, not remotely through Cloudflare hostname

Checks:

```bash
cloudflared tunnel list
cloudflared tunnel info openwebui
cat ~/.cloudflared/config.yml
dig +short chat.yourdomain.com
curl -I https://chat.yourdomain.com
```

Fixes:
- Ensure tunnel has active connectors.
- Ensure ingress hostname points to `http://localhost:3000`.
- Ensure DNS record exists and is proxied by Cloudflare.
- Ensure remote users use `https://chat.yourdomain.com`.
- Run cloudflared as persistent service (not just terminal session).

### D) Model appears in `/v1/models` but generation fails

Likely causes:
- Invalid/missing provider key.
- Deprecated provider model ID.
- Rate limit/quota/provider outage.

Checks:

```bash
set -a && source .env && env | grep -E '^(OPENAI_API_KEY|GROQ_API_KEY|ANTHROPIC_API_KEY|LITELLM_MASTER_KEY)=' | sed 's/=.*/=<set>/'
docker logs --tail 300 litellm
```

Fixes:
- Refresh model IDs from provider APIs.
- Update `litellm-config.yml`.
- Recreate stack.

### E) Full reset (last resort)

```bash
docker compose -f Docker-compose.yml down
docker compose -f Docker-compose.yml up -d --force-recreate
sleep 5
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
set -a && source .env && curl -s http://localhost:4000/v1/models -H "Authorization: Bearer $LITELLM_MASTER_KEY"
```

---

## 12) Security hardening recommendations

- Use a long random `LITELLM_MASTER_KEY`.
- Never expose LiteLLM directly to the internet unless required.
- Prefer exposing only Open WebUI through Cloudflare Tunnel.
- Keep `CORS_ALLOW_ORIGIN` restrictive in production.
- Rotate API keys periodically.
- Add Cloudflare Access policy if only specific users should access chat.

---

## 13) Known-good baseline (capture this after stable deployment)

When the stack is healthy, save a snapshot:

- `docker ps` output
- `curl /v1/models` output
- active `litellm-config.yml` model aliases
- Cloudflare tunnel hostname + `cloudflared tunnel info openwebui`

Keeping this baseline makes rollback and incident response much faster.
