# myllm — LiteLLM + Open WebUI (Docker Compose) + Cloudflare Tunnel

A small, practical stack for running an OpenAI-compatible LLM gateway (**LiteLLM**) behind a chat UI (**Open WebUI**), with **Postgres** for LiteLLM state.

Code is available here: https://github.com/dixon400/myllm

## What you get

- **Open WebUI** at `http://localhost:3000`
- **LiteLLM** at `http://localhost:4000/v1` (OpenAI-compatible)
- **LiteLLM MCP proxy** at `http://localhost:8081` (makes LiteLLM MCP callable from Open WebUI)
- **Postgres** for LiteLLM
- Optional: publish Open WebUI to the internet via **Cloudflare Tunnel** (recommended)

## Prerequisites

- Docker Desktop + `docker compose`
- `git`
- (Optional) `ollama` installed if you want local models
- (Optional) `cloudflared` if you want a public URL
- If using Cloudflare Tunnel with a custom hostname, you need **a domain name** whose DNS is managed by Cloudflare (example: `chat.yourdomain.com`).

## Quick start (local)

1) Create a project folder and clone the repo

```bash
mkdir -p ~/projects
cd ~/projects
git clone https://github.com/dixon400/myllm.git
cd myllm
```

2) Create your `.env`

```bash
cp .env.example .env
```

Edit `.env` and set your keys (and Postgres variables).

3) Start the stack

```bash
docker compose -f Docker-compose.yml up -d --force-recreate
```

4) Open the UI

- Open WebUI: `http://localhost:3000`

## MCP from Open WebUI (LiteLLM-managed)

This repo defines MCP servers in `litellm-config.yml` (example: `petstore_mcp`).
To call those MCP tools from the Open WebUI frontend without duplicating MCP config in Open WebUI, use the included `litellm-mcp-proxy` container.

1) Recreate the stack:

```bash
docker compose -f Docker-compose.yml up -d --force-recreate
```

2) In Open WebUI: Admin Settings → External Tools → Add Server

- Type: MCP (Streamable HTTP)
- Server URL: `http://litellm-mcp-proxy:8080/petstore_mcp/mcp`
- Auth: Bearer (use the same value as `LITELLM_MASTER_KEY`)

If you previously added this as an OpenAPI server, delete/disable that connection first.

## Cloudflare Tunnel (public URL)

If you want `https://chat.yourdomain.com` to work:

- You need a Cloudflare account
- You need a domain name added to Cloudflare (DNS authority moved to Cloudflare)

Follow the step-by-step guide in [LLM_STACK_SETUP_AND_OPERATIONS_GUIDE.md](LLM_STACK_SETUP_AND_OPERATIONS_GUIDE.md).

## Reference docs

- Full setup + ops runbook: [LLM_STACK_SETUP_AND_OPERATIONS_GUIDE.md](LLM_STACK_SETUP_AND_OPERATIONS_GUIDE.md)
- Model discovery commands: [MODEL_DISCOVERY_COMMANDS.md](MODEL_DISCOVERY_COMMANDS.md)
