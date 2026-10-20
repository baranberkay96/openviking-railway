# OpenViking v0.5 Context Database for AI Agents: Railway template spec

## Listing

| Field | Value |
|---|---|
| Name | **OpenViking v0.5 Context Database for AI Agents** (`v0.5` = major.minor of the pinned release v0.5.1) |
| Description | One self-hosted store for agent memory, knowledge and skills, with an API. (74 chars) |
| Category | AI/ML |
| Tags | openviking, context-database, memory, rag, ai-agents, mcp, skills, vector-search |
| Icon | `https://cdn.jsdelivr.net/gh/volcengine/OpenViking@v0.5.1/docs/images/ov-logo-square.png` (upstream logo pinned to the v0.5.1 tag, 200 OK, image/png) |

The description is delivered as written. Memory (sessions and extracted memories), knowledge (resources with L0/L1/L2 layers and semantic search) and skills share one workspace behind one REST API, plus an MCP endpoint and the Studio web console. All of it needs a model key, which is the template's one required input (see Required input).

## Base market templates

| Market template | Topology | Kept | Problems fixed |
|---|---|---|---|
| `openviking` "OpenViking" (8 active, 16 projects; upstream's deployment guide links to it) | One service, `ghcr.io/volcengine/openviking:latest`, volume `/app/.openviking`, `OPENVIKING_CONF_CONTENT` seed with `${VAR}` placeholders, generated root key, required `OPENAI_API_KEY`, bot on | **Base.** Volume path, seed-once config through upstream's `OPENVIKING_CONF_CONTENT` idea, `${VAR}` placeholders so keys are not written into the file, generated root key, one required key | Floating `latest` tag; healthcheck on `/health` (liveness) instead of `/ready`; restart `ON_FAILURE`; auth mode inferred instead of explicit; models, providers and base URLs fixed in a JSON blob (only the OpenAI key is a variable) with the Codex default model name `gpt-5.6-terra` on the API-key `openai` provider; bot shell execution left on (`direct` backend, `allow_exec` default true); bot logs written to a file on the volume instead of the Railway log; no public base URL, so OAuth/MCP links depend on forwarded headers; changing the seed later has no effect and there is no way to re-apply it |
| `openviking-aug-26-agent-memory` "OpenViking" (0 active, 2 projects) | One service built from `volcengine/OpenViking` (default branch) | Nothing | No variables, volume or domain in the serialized config: the upstream entrypoint finds no `ov.conf` and answers 503 "pending_initialization" until someone writes a config by hand; builds an unpinned branch (Rust + C++ + Node build, long and fragile) |

The brief mentioned a third listing; only the two files above exist in `_market/`. There is no `_upstream/` copy for this app, so nothing was read there. No market text is reused.

## Upstream checked (volcengine/OpenViking v0.5.1)

Read through `gh api repos/volcengine/OpenViking/contents/{path}?ref=v0.5.1` and from the published image's own layers (registry API, no Docker): `docker-compose.yml`, `Caddyfile`, `Dockerfile`, `deploy/docker/openviking-entrypoint.sh`, `deploy/docker/pending_health_server.py`, `deploy/helm/openviking/values.yaml` and templates, `pyproject.toml`, `uv.lock`, and the docs `guides/03-deployment.md`, `01-configuration.md`, `04-authentication.md`, `06-mcp-integration.md`, `08-encryption.md`, `11-oauth.md`, `12-public-access.md`, `15-snapshot.md`, `17-vikingbot.md`, `19-deployment-checklist.md`, `22-gateway-operations.md`, `concepts/05-storage.md`, `configuration/01-server.md`, `gateway/13-configuration.md`. Source details are listed in NOTICE.md.

## Pinned versions

| Service | Image | Version | Released | Checked |
|---|---|---|---|---|
| OpenViking | `ghcr.io/volcengine/openviking:v0.5.1` | v0.5.1, image label `org.opencontainers.image.revision=e855796c83f9eb575d9b0f3082926660730f6a63` (= the v0.5.1 tag commit) | 2026-10-19 (GitHub release 13:37 UTC, image created 13:38 UTC) | 2026-10-20 |

The tag is an OCI index with linux/amd64 (`sha256:c592e9be…eb71`, about 313 MB compressed) and linux/arm64. Python 3.13.16, OpenViking 0.5.1, openviking-sdk 0.1.15. The image includes the `bot`, `gemini`, `opengauss` and `expiry` extras; it does **not** include `local-embed` (no `llama_cpp` in the image), so upstream's key-free `local` embedding provider cannot run in it.

**Upgrade:** read the release notes, copy a backup of the volume (or take a Railway volume backup), change the image tag, redeploy. Upstream notes that new vector record formats are one-way: a downgrade needs the backup, not just the old tag.

## Architecture

```
 Agents / SDKs / ov CLI ── HTTPS, X-API-Key or Authorization: Bearer {user key}
 MCP clients (Claude Code, Cursor, ...) ── https://{domain}/mcp (same keys, or OAuth 2.1)
 Browser ── https://{domain}/studio (Studio sign-in with a root or user key)
                │
                ▼
   OpenViking :1933 (public, Railway TLS)   uvicorn, 1 worker, bound 0.0.0.0
   REST /api/v1/*, /mcp, /studio, /ready, /health, OAuth /.well-known/*
   ├─ VikingBot gateway on 127.0.0.1:18790 (child process, Studio chat, shell exec off)
   ├─ RAGFS (Rust, in-process)  ─┐
   └─ embedded vector DB        ─┴─ volume /app/.openviking (ov.conf + data/)
                │
                └──► OpenAI-compatible API (or another provider): embeddings + LLM
```

## Service: OpenViking

| Setting | Value |
|---|---|
| Source | image `ghcr.io/volcengine/openviking:v0.5.1` |
| Start | `/bin/sh -c '...; exec openviking-server --host 0.0.0.0 --port "$PORT" [--with-bot --disable-bot-logging]'` (see below) |
| Healthcheck | `/ready` (readiness: RAGFS, vector DB, API key manager, embedder initialized), timeout 600 s |
| Public domain | yes, target port 1933 |
| Volume | `/app/.openviking` (`ov.conf`, workspace `data/`, OAuth DB, bot workspace) |
| Restart | ALWAYS |
| Replicas | 1 (the embedded vector DB takes an exclusive lock on the workspace) |

**Start command.** It replaces the image's `openviking-entrypoint` with the same two steps, done visibly:

1. Config: `OPENVIKING_CONF_MODE=seed` (default) writes `OPENVIKING_CONF_CONTENT` to `/app/.openviking/ov.conf` only when the file does not exist yet, which is upstream's documented behaviour. `overwrite` rewrites the file from the variable on every start and keeps the previous one as `ov.conf.previous`. The file is written with mode 0600 through a temp file and `mv`.
2. Server: `openviking-server --host 0.0.0.0 --port $PORT`, plus `--with-bot --disable-bot-logging` when `OPENVIKING_WITH_BOT=1`. Upstream's entrypoint cannot pass `--disable-bot-logging`, so the bot's output would otherwise go to a file on the volume; here it appears in the Railway log. Invalid values of the two switches stop the start with a clear message.

If Railway passes the command as arguments to the image entrypoint instead of replacing it, `openviking-entrypoint` execs any unknown first argument (`/bin/sh`), so the result is the same.

**Configuration through variables.** The seeded `ov.conf` contains `${VAR}` placeholders. OpenViking (and VikingBot) run `os.path.expandvars` on the file text before parsing it, at every start. So keys, providers, models, base URLs, log level and OAuth follow the Railway variables after a redeploy even in seed mode, and the file on the volume never holds the secrets. Keep every referenced variable defined: an unset variable stays a literal `${...}`.

Seeded `ov.conf` (the value of `OPENVIKING_CONF_CONTENT`):

- `storage`: workspace `/app/.openviking/data` (absolute), RAGFS `local`, vector DB `local` (collection `context`).
- `server`: host `0.0.0.0`, port 1933, `workers: 1`, **`auth_mode: "api_key"`** with `root_api_key: "${OPENVIKING_ROOT_API_KEY}"`, `cors_origins: ["*"]` (upstream default; browsers still need a key).
- `embedding.dense` and `vlm`: provider, base URL, key and model from the `OV_*` variables; dimension from `OV_EMBEDDING_DIMENSION`.
- `log`: level `${OV_LOG_LEVEL}`, output stdout.
- `oauth.enabled: "${OV_OAUTH_ENABLED}"`.
- `bot.sandbox`: `direct` backend with `allow_exec: false`.

**Auth (fails closed).** `auth_mode` is explicit, so a missing root key stops startup instead of falling back to dev mode (dev mode also refuses `0.0.0.0`). The root key is `${{secret(48)}}` (letters and digits). Every data route (`/api/v1/*`, `/mcp`, WebDAV, bot proxy) needs a user or admin key; the root key only administers accounts and users and cannot read tenant data. Unauthenticated: `/health`, `/ready`, the Studio static files, `/docs` + `/openapi.json` (schema only), OAuth discovery/registration endpoints, and `/metrics` (404 unless Prometheus is enabled).

**VikingBot.** On by default, as in the image and the market template; it powers the chat in Studio and can connect chat channels. In server-managed mode it uses the caller's own key per request, so no extra key is needed. Its default `direct` sandbox lets the agent run shell commands inside this container, which holds the root key and the model keys; the seed sets `allow_exec: false`, so any signed-in user cannot escalate through the bot. File tools stay restricted to the bot workspace. `OPENVIKING_WITH_BOT=0` runs the API only (upstream's Helm chart does that) and saves about 0.3 GB.

**OAuth.** On, with `OPENVIKING_PUBLIC_BASE_URL=https://${{RAILWAY_PUBLIC_DOMAIN}}` as issuer. Claude.ai, ChatGPT and other OAuth MCP clients can add `https://{domain}/mcp`; approval happens in Studio and needs a valid user key. API-key clients are unaffected. `OV_OAUTH_ENABLED=false` removes the OAuth routes.

**Storage backends (what OpenViking supports and why local).** Content: RAGFS `local`, `s3` or `memory`. Vectors: `local` (embedded), `cuvs`, `http`, `volcengine`, `vikingdb`, `opengauss`. Task store: file or MySQL; account store: file, MySQL or Redis+MySQL; Redis only for multi-instance caches/locks. Upstream's compose and Helm both run single-node local storage on one volume, and multi-instance setups need a remote vector service that has no Railway-hosted open-source option in this stack. So the template uses one volume and no Bucket, Postgres or Redis; adding them would not add a working feature.

**Caddy dropped.** Upstream's compose calls its Caddy service a "legacy reverse proxy" on port 1934, kept for old deployments; it only proxies every path to 1933 and optionally terminates TLS. Railway's edge already terminates TLS and routes the domain to 1933, so a Caddy service would have no function.

**Rule 8.** `PORT=1933` is set and passed to `--port` (8.1). The server is single-process uvicorn, so it binds `0.0.0.0`, not `::` (8.2). No other service in this template connects to it over the private network, so the dual-stack rule (8.14) has no internal caller to serve; a Railway service added later reaches it at `http://openviking.railway.internal:1933` in environments with IPv4 private DNS (new environments). No Dockerfile is added, so there is nothing to build (8.10).

## Variables

| Variable | Default | Purpose |
|---|---|---|
| `PORT` | `1933` | Listen port; Railway's domain and healthcheck use it |
| `OPENVIKING_ROOT_API_KEY` | `${{secret(48, alnum)}}` | Root key for the Admin API and Studio admin pages |
| `OPENAI_API_KEY` | **required** | OpenAI-compatible key for embeddings and LLM |
| `OPENAI_API_BASE` | `https://api.openai.com/v1` | Shared base URL (OpenRouter, Azure-style gateways, LiteLLM, ...) |
| `OV_VLM_PROVIDER` / `OV_VLM_MODEL` | `openai` / `gpt-5.4` | LLM for L0/L1 summaries, memory extraction, bot (upstream's OpenAI wizard preset) |
| `OV_VLM_API_KEY` / `OV_VLM_API_BASE` | `${{OPENAI_API_KEY}}` / `${{OPENAI_API_BASE}}` | Override to use another LLM provider |
| `OV_EMBEDDING_PROVIDER` / `OV_EMBEDDING_MODEL` / `OV_EMBEDDING_DIMENSION` | `openai` / `text-embedding-3-small` / `1536` | Embedding model (upstream's OpenAI preset); fix before adding data |
| `OV_EMBEDDING_API_KEY` / `OV_EMBEDDING_API_BASE` | references to the shared key / base | Override for a separate embedding provider |
| `OV_OAUTH_ENABLED` | `true` | OAuth 2.1 for MCP clients |
| `OV_LOG_LEVEL` | `INFO` | Server log level |
| `OPENVIKING_PUBLIC_BASE_URL` | `https://${{RAILWAY_PUBLIC_DOMAIN}}` | Issuer and link base for OAuth and MCP uploads/downloads |
| `OPENVIKING_WITH_BOT` | `1` | VikingBot on/off |
| `OPENVIKING_CONF_MODE` | `seed` | `seed` or `overwrite` |
| `OPENVIKING_CONF_CONTENT` | seed JSON above | Template for `ov.conf` |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | `30` | Graceful stop before a redeploy |

Optional, not predefined: `OPENVIKING_GATEWAY_ENABLED=true` turns on OpenViking Gateway under `/gateway` (an LLM gateway that adds memory to Claude Code, Codex and OpenAI-compatible clients; upstreams are configured in Studio). `OPENVIKING_GATEWAY_MANAGEMENT_KEY` (32+ chars) then restricts who may change upstreams.

## Required input

`OPENAI_API_KEY`, one key. The image cannot start without a model key: its key-free `local` embedder is not installed, and an `openai` LLM section without a key fails validation at boot. The same key serves embeddings and the LLM through any OpenAI-compatible endpoint; the `OV_*` variables split them across providers (for example Volcengine Ark, Jina, Voyage, Gemini, DashScope, Azure, LiteLLM).

## First login

1. Open `https://{domain}/studio`, open the connection/identity settings and paste `OPENVIKING_ROOT_API_KEY` from the service's Variables tab.
2. Create an account with its first admin user in Studio's users page; the user key is shown once. Or with curl: `POST /api/v1/admin/accounts` with `X-API-Key: {root key}` and body `{"account_id":"default","admin_user_id":"admin"}`; the response contains `user_key`.
3. Sign in to Studio with that user key to browse resources, memories and skills, and to chat with the bot.
4. Agents use the user key: REST (`X-API-Key` or `Authorization: Bearer`), the Python SDK (`openviking_sdk.SyncHTTPClient(url=..., api_key=...)`), the `ov` CLI (`ovcli.conf` with `url` and `api_key`), or MCP at `https://{domain}/mcp`.

## Improvements over market templates

1. **Pinned v0.5.1** official image (market: `latest`, or an unpinned source build).
2. **Readiness healthcheck** `/ready` with a 600 s window, `PORT` set, restart `ALWAYS`, 30 s draining.
3. **Auth fails closed**: explicit `auth_mode: api_key`, 48-character generated root key, no secret written into `ov.conf`.
4. **Every model setting is a variable**: provider, model, base URL, key and embedding dimension for LLM and embeddings separately, defaulting to one shared key; defaults follow upstream's OpenAI preset (`gpt-5.4`, `text-embedding-3-small`/1536).
5. **Safer VikingBot**: shell execution disabled in the seed; bot output in the Railway log (`--disable-bot-logging`); one switch to run the API alone.
6. **OAuth 2.1 for MCP** on Railway's HTTPS domain with an explicit public base URL, so web MCP clients (Claude.ai, ChatGPT) can connect after approval with a user key.
7. **Config lifecycle**: `seed` keeps upstream behaviour and Studio edits; `overwrite` re-applies the template's config and keeps `ov.conf.previous`; clear errors for empty or invalid settings.
8. **Visible logs and absolute paths**: server log to stdout at INFO, workspace `/app/.openviking/data` on the volume.
9. **No dead services**: Caddy omitted with the reason recorded; storage choices documented against upstream's backends.

## Resource estimate

Not measured (no Docker this round). Basis: image contents (FastAPI, LiteLLM, tree-sitter parsers, embedded vector engine, VikingBot), upstream Helm defaults (requests 0.5 CPU / 1 GiB, limits 2 CPU / 4 GiB, 20 GiB PVC) and comparable Python services on Railway. Embeddings and LLM run at the provider, so CPU stays low.

| Service | Idle RAM / CPU | Typical RAM / CPU | Volume |
|---|---|---|---|
| OpenViking (server + VikingBot) | 0.8 GB / 0.02 vCPU | 1.5 GB / 0.3 vCPU (bursts while parsing and indexing) | 0.2 GB idle, 2–5 GB typical |

Railway prices (RAM $10/GB-month, vCPU $20/vCPU-month, volume $0.15/GB-month):

- **Idle ≈ $8.50/month** (0.8 × 10 + 0.02 × 20 + 0.2 × 0.15).
- **Typical ≈ $21.50/month** (1.5 × 10 + 0.3 × 20 + 3 × 0.15).
- `OPENVIKING_WITH_BOT=0` saves roughly $3/month. Model calls are billed by the provider (every added file gets an LLM summary and an embedding).

Minimum plan: **Hobby**. Trial's 1 GB per service is too tight with the bot under load. Hobby volumes are capped at 5 GB per service; large corpora (upstream suggests 20 GiB) need Pro.

## Verification checklist

On first Railway deploy:

- [ ] Log shows `[railway] wrote /app/.openviking/ov.conf from OPENVIKING_CONF_CONTENT (seed mode)`, `Vikingbot gateway started`, then `OpenViking HTTP Server is running on 0.0.0.0:1933`; the deployment turns healthy on `/ready`.
- [ ] `curl https://{domain}/ready` returns 200 with `agfs`, `vectordb`, `api_key_manager`, `embedding` ok; `/health` shows `"auth_mode":"api_key"`.
- [ ] `curl 'https://{domain}/api/v1/fs/ls?uri=viking://resources'` without a key returns 401; with the root key it is refused for tenant data.
- [ ] `POST /api/v1/admin/accounts` with the root key returns a `user_key`; `GET /api/v1/fs/ls?uri=viking://` with that key returns 200.
- [ ] Add a resource with the user key (SDK `client.add_resource(path=..., wait=True)` or Studio upload) and wait: L0/L1 summaries appear and `find` returns it (proves embedding + LLM).
- [ ] Session: add messages, commit, then find the extracted memory under `viking://~/memories` (extraction is asynchronous).
- [ ] Studio at `/studio`: root key opens admin pages; user key shows data; the bot chat answers; asking the bot to run a shell command is refused (no exec tool).
- [ ] MCP: `claude mcp add --transport http openviking https://{domain}/mcp --header "Authorization: Bearer {user-key}"` lists tools; without the header 401 with a `WWW-Authenticate` resource-metadata hint.
- [ ] OAuth: `https://{domain}/.well-known/oauth-authorization-server` shows the `https://{domain}` issuer.
- [ ] Redeploy: data, users and keys survive; log says `keeping existing ... ov.conf`.
- [ ] Change `OV_LOG_LEVEL` to `DEBUG` and redeploy: placeholders are re-expanded without touching the file.
- [ ] `OPENVIKING_WITH_BOT=0`: server starts without the bot.

## Open questions

- **Not run anywhere yet.** The start command was checked with `sh -n`/`bash -n`/`dash -n` and exercised with a stub server; the image itself was not started. VikingBot startup on Railway is the largest unknown: if it fails, the server exits before listening. Fallback: `OPENVIKING_WITH_BOT=0`.
- **Typed placeholders.** `dimension` and `oauth.enabled` are strings after expansion (`"1536"`, `"true"`); Pydantic 2.12 lax mode converts them, and the fields are plain `int`/`bool` (not strict), but this was read, not executed.
- **Default model ids** (`gpt-5.4`, `text-embedding-3-small`) come from upstream's v0.5.1 setup wizard; availability on the user's OpenAI account is not checked. `gpt-5.4` is a large model for per-file summaries; a cheaper model can be set in `OV_VLM_MODEL`.
- **Studio model form** is read-only when `ov.conf` contains `$` placeholders (upstream behaviour). Model settings are changed through Railway variables, or in Studio's verbatim file editor.
- **Studio "Enable gateway"** writes `ov.conf`; with placeholders in the file its behaviour was not verified. `OPENVIKING_GATEWAY_ENABLED=true` is the documented alternative.
- **API keys at rest**: upstream stores user keys in plaintext JSON on the volume by default; `encryption.api_key_hashing.enabled` (Argon2id, 64 MB per verification) can be added to `ov.conf`, at a CPU cost per request.
- **Secrets with JSON-special characters** (`"` or `\`) in a key would break the expanded JSON; provider keys normally do not contain them.
- **Legacy IPv6-only environments**: the server binds IPv4 only (Rule 8.2), so another service there would reach it via the public URL.
