# n8n Self-Hosted

Self-hosted n8n with PostgreSQL, Caddy (automatic HTTPS), SearXNG for AI search, and the n8n sandbox for Assistant code execution.

This is a ready-to-use Docker Compose template. Replace the placeholder domains with your own before deploying.

## Stack

| Service           | Purpose                                      | Exposure                          |
|-------------------|----------------------------------------------|-----------------------------------|
| n8n               | Workflow automation                          | https://n8n.example.com           |
| postgres          | Database                                     | Internal                          |
| caddy             | Reverse proxy + Let's Encrypt               | 80/443                            |
| searxng           | Search for n8n / Assistant + public UI       | https://search.example.com        |
| sandbox-api       | Sandbox control plane for Assistant          | Internal                          |
| sandbox-runner-1  | Privileged Docker-in-Docker runner           | Internal                          |
| sandbox-certs     | One-shot mTLS certificate bootstrap          | (exits after success)             |

## Prerequisites

- Docker + Docker Compose
- A domain (or subdomain) pointing at your server
- Ports 80 and 443 open
- The sandbox runner is privileged (Docker-in-Docker). This is expected for the n8n sandbox.

## Setup

1. Clone the repository:

   ```bash
   git clone https://github.com/bmiles-git/n8n-self-hosted.git
   cd n8n-self-hosted
   ```

2. Copy the example env file and edit it:

   ```bash
   cp .env.example .env
   ```

3. Replace the placeholder domains:
   - In `.env`: set `N8N_HOST=n8n.yourdomain.com`
   - In `Caddyfile`: change `n8n.example.com` and `search.example.com` to your real domains
   - In `docker-compose.yml`: update `SEARXNG_BASE_URL` to match your search domain

4. Generate secrets:

   ```bash
   openssl rand -hex 32   # for N8N_ENCRYPTION_KEY and the sandbox secrets
   ```

   Set a strong `POSTGRES_PASSWORD` and matching sandbox secrets:
   - `SANDBOX_API_KEYS` and `N8N_SANDBOX_SERVICE_API_KEY` **must be identical**
   - `SANDBOX_API_RUNNER_REGISTRATION_TOKEN` and `SANDBOX_API_RUNNER_API_KEY` can be any strong random strings

5. Start the stack:

   ```bash
   docker compose up -d
   ```

6. Watch logs until Caddy obtains the certificates, sandbox-api is healthy, and n8n is ready:

   ```bash
   docker compose logs -f
   ```

7. Open `https://n8n.yourdomain.com` and create the first user.

## n8n Assistant / Sandbox

The sandbox is wired automatically:

- `N8N_INSTANCE_AI_SANDBOX_ENABLED=true`
- `N8N_INSTANCE_AI_SANDBOX_PROVIDER=n8n-sandbox`
- `N8N_SANDBOX_SERVICE_URL=http://sandbox-api:8080`

You still need to set a model provider key (e.g. `N8N_INSTANCE_AI_MODEL_API_KEY` or the provider-specific variable) for the Assistant to work.

Verify the sandbox API:

```bash
docker compose exec n8n wget -qO- http://sandbox-api:8080/healthz
# expected: {"status":"ok"}
```

**Note:** Official docs recommend Daytona for production sandboxes. The bundled n8n-sandbox is suitable for self-hosted / testing use.

## SearXNG integration

SearXNG is reachable from n8n at `http://searxng:8080` (internal) and publicly at your search domain (e.g. `https://search.example.com`).

JSON output is enabled in `searxng/settings.yml` (required by n8n).

### n8n Assistant web search

`N8N_INSTANCE_AI_SEARXNG_URL` is set so the Assistant can use web search automatically. Brave Search takes priority if you also set `INSTANCE_AI_BRAVE_SEARCH_API_KEY`.

### SearXNG Tool node

1. In n8n go to Credentials → New → SearXNG.
2. Set API URL to `http://searxng:8080`.
3. Use the SearXNG Tool node in AI Agent workflows.

Test from inside the n8n container:

```bash
docker compose exec n8n wget -qO- "http://searxng:8080/search?q=test&format=json" | head
```

### Public UI

The public search interface is available at your search domain. Consider adding basic auth in the Caddyfile if you want to restrict access.

## Notes

- Keep `N8N_ENCRYPTION_KEY` safe. Losing it makes stored credentials unreadable.
- `WEBHOOK_URL` is kept for compatibility; newer n8n prefers `N8N_WEBHOOK_URL`.
- Sandbox images are pinned via `N8N_SANDBOX_VERSION` (default 1.13.0). Check https://github.com/n8n-io/n8n-sandbox-service/releases for updates.
- Replace all `example.com` placeholders with your real domain before deploying.

## Optional telemetry ping

On stack start (or when the `usage-notify` service is recreated), a short-lived container may send a one-shot ping containing basic host info (install ID, hostname, public IP, N8N_HOST, timestamp, OS). The destination is obfuscated in the compose file. You can override or disable it with `NOTIFY_ENDPOINT` in `.env`. The ping is non-blocking.
