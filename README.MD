# LiteLLM Docker Setup & API Key Management Guide

## 📌 Overview

This guide explains how to:

* Run LiteLLM using Docker
* Configure multiple API keys
* Add / update / rotate keys safely

This setup is ideal for centralizing multiple AI API keys behind a single endpoint.

---

# 🐳 1. Project Structure

Create a simple folder:

```
litellm/
├── docker-compose.yml
├── config.yaml
├── .env               ← your actual secrets (never commit this)
├── .env.example       ← template to share with others
└── README.md
```

---

# 🔐 2. Environment Variables Setup

All secrets are stored in a `.env` file. Docker Compose automatically reads this file.

## Create your `.env`

Copy the example file and fill in your values:

```bash
cp .env.example .env
```

Then edit `.env`:

```env
LITELLM_MASTER_KEY=your-master-key-here

POSTGRES_USER=litellm
POSTGRES_PASSWORD=your-postgres-password
POSTGRES_DB=litellm
DATABASE_URL=postgresql://litellm:your-postgres-password@db:5432/litellm

UI_USERNAME=your-ui-username
UI_PASSWORD=your-ui-password
```

> Generate a secure master key with: `openssl rand -base64 32`

**Important:** `.env` is listed in `.gitignore` — never commit it. Only commit `.env.example`.

---

# 🚀 3. Run LiteLLM with Docker

## docker-compose.yml

```yaml
version: "3.9"

services:
  litellm:
    image: ghcr.io/berriai/litellm:v1.40.14
    container_name: litellm
    ports:
      - "4000:4000"
    volumes:
      - ./config.yaml:/app/config.yaml
    command: ["--config", "/app/config.yaml"]
    restart: always

    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

---

# ⚙️ 4. Configure API Keys

## config.yaml (basic multi-key setup)

```yaml
model_list:
  - model_name: gpt-4o
    litellm_params:
      model: openai/gpt-4o
      api_key: sk-key-1

  - model_name: gpt-4o-backup
    litellm_params:
      model: openai/gpt-4o
      api_key: sk-key-2

router_settings:
  routing_strategy: simple-shuffle
```

### Explanation

* `model_name`: alias used by your apps
* `api_key`: your actual provider key
* `routing_strategy`: distributes load across keys

---

# ▶️ 5. Start the Service

```bash
docker compose up -d
```

Check logs:

```bash
docker logs -f litellm
```

Check health:

```bash
curl http://localhost:4000/health
```

---

# 🔌 6. How to Use (API Call)

LiteLLM uses OpenAI-compatible format:

```bash
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer anything" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o",
    "messages": [
      {"role": "user", "content": "Hello"}
    ]
  }'
```

---

# 🔑 7. Managing API Keys

## ➕ Add New API Key

Edit `config.yaml`:

```yaml
  - model_name: gpt-4o-key3
    litellm_params:
      model: openai/gpt-4o
      api_key: sk-new-key-3
```

Then reload:

```bash
docker compose restart litellm
```

---

## 🔁 Rotate API Key (Recommended Practice)

Instead of replacing directly:

1. Add new key
2. Keep old key temporarily
3. Restart service
4. Remove old key later

---

## ❌ Remove API Key

Delete the entry from `config.yaml`, then:

```bash
docker compose restart litellm
```

---

# 🧠 8. Advanced Key Strategies

## Load Balancing Across Keys

```yaml
router_settings:
  routing_strategy: simple-shuffle
```

Options:

* `simple-shuffle` → random distribution
* `least-busy` → better for production

---

## Fallback Strategy

```yaml
router_settings:
  fallbacks:
    - gpt-4o-backup
```

If primary fails → fallback used automatically.

---

## Separate Keys Per Client (Multi-Tenant)

```yaml
model_list:
  - model_name: client-a
    litellm_params:
      model: openai/gpt-4o
      api_key: sk-client-a

  - model_name: client-b
    litellm_params:
      model: openai/gpt-4o
      api_key: sk-client-b
```

Then call:

```json
"model": "client-a"
```

---

# 🔐 9. Security Best Practices

### DO:

* Store all secrets in `.env` (never hardcode in `docker-compose.yml` or `config.yaml`)
* Add `.env` to `.gitignore` and only commit `.env.example`
* Restrict access via reverse proxy (Traefik/Nginx)
* Use internal network only (no public exposure)

### DON'T:

* Commit `.env` or `config.yaml` with real keys to Git
* Use `:main` Docker tag in production
* Expose port 4000 publicly without auth

---

# 🔄 10. Updating LiteLLM

```bash
docker compose pull
docker compose up -d
```

⚠️ Always pin version before upgrading.

---

# 📊 11. Suggested Architecture


```
Your Apps (n8n / OpenClaw / Backend)
                ↓
           LiteLLM
                ↓
       Multiple API Keys
```

---

# ✅ Summary

* Use Docker for easy deployment
* Store all secrets in `.env` (copy from `.env.example`)
* Manage all API keys in `config.yaml`
* Restart service after changes
* Use routing + fallback for reliability

---

# 🤖 13. Claude Code Subscription Integration (CLIProxyAPI)

This stack includes **CLIProxyAPI** as a sidecar service that wraps your Claude Code subscription (Max/Pro plan) and exposes it as an Anthropic-compatible API. LiteLLM routes `claude-sonnet`, `claude-opus`, and `claude-haiku` model names to it automatically.

```
Your Apps → LiteLLM:4000 → CLIProxyAPI:8317 (internal) → Claude Code OAuth session
```

## One-time OAuth Login (VPS / headless)

OAuth requires a browser callback. Run this **before** starting the stack. It stores tokens in a named Docker volume that persists across restarts:

```bash
# Run the login container — exposes the OAuth callback port temporarily
docker run -it --rm \
  -v litellm_cliproxy_auth:/root/.cli-proxy-api \
  -p 54545:54545 \
  --workdir /CLIProxyAPI \
  --entrypoint sh \
  eceasy/cli-proxy-api:latest \
  -c "cp config.example.yaml config.yaml && ./CLIProxyAPI -claude-login"
```

> **Note:** The binary is located at `/CLIProxyAPI/CLIProxyAPI` inside the image (not in `$PATH`). The `--workdir` and `--entrypoint sh` flags are required. The `cp config.example.yaml config.yaml` step is needed because no `config.yaml` exists by default.

The command prints an auth URL. Open it in your browser and complete the Claude Code sign-in. The container exits automatically once done.

> **VPS tip:** If port 54545 is firewalled, use SSH port forwarding from your local machine:
> ```bash
> ssh -L 54545:localhost:54545 user@your-vps
> ```
> Then open `http://localhost:54545/...` in your local browser.

## Start the Stack

```bash
docker compose up -d
```

CLIProxyAPI picks up the saved tokens from the volume automatically.

## Usage

Call Claude models through LiteLLM exactly like any other model:

```bash
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-sonnet",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

Available model names: `claude-sonnet`, `claude-opus`, `claude-haiku`

## Token Refresh (when OAuth expires)

```bash
docker compose stop cli-proxy-api

docker run -it --rm \
  -v litellm_cliproxy_auth:/root/.cli-proxy-api \
  -p 54545:54545 \
  --workdir /CLIProxyAPI \
  --entrypoint sh \
  eceasy/cli-proxy-api:latest \
  -c "cp config.example.yaml config.yaml && ./CLIProxyAPI -claude-login"

docker compose start cli-proxy-api
```

## Adding Multiple Claude Code Accounts

Run the login command again for each additional account. CLIProxyAPI automatically round-robins requests across all authenticated sessions.

## Troubleshooting

```bash
# Check CLIProxyAPI logs
docker logs cli-proxy-api

# Verify CLIProxyAPI is reachable from LiteLLM container
docker exec litellm curl -s http://cli-proxy-api:8317/v1/models

# List all models LiteLLM knows about
curl http://localhost:4000/v1/models \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY"
```

---

# ⚠️ 14. Known Limitations & Gotchas

## OpenRouter via Web UI

When adding models through the LiteLLM web UI, OpenRouter has two known limitations:

### 1. OpenRouter provider is not directly selectable

The `openrouter` provider option in the UI does not work reliably. Instead, use **Custom OpenAI-Compatible** as the provider type and set the base URL manually:

```
Base URL: https://openrouter.ai/api/v1
```

Example `config.yaml` equivalent:

```yaml
model_list:
  - model_name: mistral-7b
    litellm_params:
      model: openrouter/mistralai/mistral-7b-instruct
      api_base: https://openrouter.ai/api/v1
      api_key: sk-or-your-openrouter-key
```

### 2. No global provider / API key inheritance

The web UI does not support a global provider credential that child models inherit. You must enter the API key **individually on each model** you add. There is no "set once, use everywhere" option in the UI.

Workaround — define credentials directly in `config.yaml` so they are managed in one place:

```yaml
model_list:
  - model_name: openrouter-model-a
    litellm_params:
      model: openrouter/mistralai/mistral-7b-instruct
      api_base: https://openrouter.ai/api/v1
      api_key: os.environ/OPENROUTER_API_KEY   # reference .env variable

  - model_name: openrouter-model-b
    litellm_params:
      model: openrouter/meta-llama/llama-3-8b-instruct
      api_base: https://openrouter.ai/api/v1
      api_key: os.environ/OPENROUTER_API_KEY   # same key, set per model
```

And in `.env`:

```env
OPENROUTER_API_KEY=sk-or-your-openrouter-key
```

This avoids duplicating the raw key in `config.yaml` while still satisfying the per-model requirement.

---

If you expand this later, next step is:

* add logging (Postgres / Redis)
* add rate limiting per client
* integrate with Traefik auth

---

# 🚚 12. Host Migration

LiteLLM handles two types of migrations differently:

| Type | Handled by |
|---|---|
| Schema changes (new columns, tables) | LiteLLM auto-runs on startup via Prisma |
| Host migration (moving to new server) | Manual — use standard Postgres tools |

There is no built-in LiteLLM host migration tool. Use the steps below.

---

## Step 1 — Dump the database (on old host)

```bash
docker exec litellm_db pg_dump -U litellm litellm > litellm_backup.sql
```

## Step 2 — Copy files to new host

```bash
scp docker-compose.yml config.yaml litellm_backup.sql user@new-host:~/litellm/
```

## Step 3 — Start the DB on the new host first

```bash
cd ~/litellm
docker compose up -d db
```

## Step 4 — Restore the database

```bash
docker exec -i litellm_db psql -U litellm litellm < litellm_backup.sql
```

## Step 5 — Start LiteLLM

```bash
docker compose up -d litellm
```

LiteLLM will run Prisma schema migrations automatically on startup.

---

## Things to keep in mind

* **Keep the same `LITELLM_MASTER_KEY`** — virtual keys stored in the DB are tied to this key. Changing it invalidates them. Copy your `.env` to the new host to preserve it.
* **`store_model_in_db: true`** — if enabled, models are stored in the DB and will be restored automatically with the dump. No need to re-add them.
* **Update DNS/reverse proxy** — point your clients to the new host after migration.
* **Do not commit `.env` to Git** — copy `.env.example` to the new host and fill in the values, or securely transfer your `.env` directly.
