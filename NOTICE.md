# NOTICE

This Railway template is a community package. It is not affiliated with or endorsed by Beijing Volcano Engine Technology Co., Ltd., ByteDance or the OpenViking project.

## What this template runs (unmodified upstream image)

| Component | Source | License |
|---|---|---|
| OpenViking server, Studio, VikingBot, OpenViking Gateway | `ghcr.io/volcengine/openviking:v0.5.1`, built by upstream from https://github.com/volcengine/OpenViking at tag v0.5.1 (image label `org.opencontainers.image.revision=e855796c83f9eb575d9b0f3082926660730f6a63`, `org.opencontainers.image.licenses=AGPL-3.0`) | GNU Affero General Public License v3.0; source files carry `Copyright (c) 2026 Beijing Volcano Engine Technology Co., Ltd.` and `SPDX-License-Identifier: AGPL-3.0` |
| Python dependencies inside the image (FastAPI, Uvicorn, LiteLLM, OpenAI SDK, tree-sitter, Pydantic and others) | installed by upstream's Dockerfile from `uv.lock` | Their own licenses (mostly MIT, BSD and Apache-2.0); see each package's metadata in the image |

The image is used as published: no file in it is changed, added or removed. This package contains only Railway settings: environment variables, a seed `ov.conf` (JSON built from field names documented by upstream) and a start command that writes that file and calls the image's own `openviking-server` program. None of these files is copied from upstream or from another template.

## AGPL-3.0 notes

These are practical notes, not legal advice; consult your own counsel for your situation.

- OpenViking is licensed under the AGPL-3.0. Running the unmodified official image and letting users reach it over a network does not change the license of OpenViking, and its complete corresponding source is published by upstream at https://github.com/volcengine/OpenViking/tree/v0.5.1.
- If you **modify** OpenViking (for example by building your own image with patched code) and let users interact with that modified version over a network, AGPL-3.0 section 13 requires you to offer those users the corresponding source of your modified version.
- Changing configuration (`ov.conf`, environment variables, models, providers) is not a modification of the program.
- Data you store in OpenViking (memories, documents, skills) is yours and is not covered by the program's license.
- Model providers you connect (OpenAI or others) have their own terms and pricing.

## Upstream material checked (OpenViking v0.5.1, 2026-10-20)

Read through the GitHub API at `ref=v0.5.1` (`gh api repos/volcengine/OpenViking/contents/{path}`), for configuration facts only:

- `docker-compose.yml` (openviking + legacy Caddy on 1934, volume `~/.openviking:/app/.openviking`, gateway variables), `Caddyfile`, `Dockerfile` (stages, extras `bot gemini opengauss expiry`, `WORKDIR /app/.openviking`, entrypoint), `.dockerignore`
- `deploy/docker/openviking-entrypoint.sh` (config seeding from `OPENVIKING_CONF_CONTENT`, bot switch, port resolution) and `deploy/docker/pending_health_server.py`
- `deploy/helm/openviking/values.yaml`, `templates/deployment.yaml`, `templates/configmap.yaml`, `deploy/helm/README.md` (single replica, bot off in Kubernetes, resources, `${OPENVIKING_ROOT_API_KEY}` placeholder pattern, log to stdout)
- `pyproject.toml` (license AGPL-3.0, extras; `local-embed` = `llama-cpp-python`), `uv.lock` (pydantic 2.12.5), `LICENSE`
- Docs under `docs/en/`: `guides/01-configuration.md`, `03-deployment.md`, `04-authentication.md`, `06-mcp-integration.md`, `08-encryption.md`, `11-oauth.md`, `12-public-access.md`, `15-snapshot.md`, `17-vikingbot.md`, `19-deployment-checklist.md`, `20-private-deployment.md`, `22-gateway-operations.md`, `concepts/05-storage.md`, `concepts/11-multi-tenant.md`, `configuration/01-server.md`, `gateway/11-security-and-data.md`, `gateway/13-configuration.md`, `api/01-overview.md`, `api/07-system.md`, `api/08-admin.md`, `faq/faq.md`, `about/02-changelog.md`
- Source: `openviking_cli/utils/config/config_loader.py` (`os.path.expandvars` before JSON parsing), `embedding_config.py` (providers, key rules, default `local` embedder), `vlm_config.py` (key required for non-LiteLLM providers, `is_available`), `vectordb_config.py` (`project` alias), `oauth_config.py`, `log_config.py`, `open_viking_config.py` (top-level sections), `setup_wizard.py` (OpenAI preset `text-embedding-3-small`/1536, `gpt-5.4`), `openviking/server/config.py` (`ServerConfig`), `openviking/server/bootstrap.py` (CLI flags, bot start and wait, `--disable-bot-logging`), `openviking/server/app.py` and `routers/*.py` (route authentication, `/ready`, `/metrics`, OAuth mounting, config-file and restart endpoints), `openviking/server/mcp_endpoint.py` (MCP auth), `openviking/config/config_file.py` (Studio editor and `$` placeholders), `openviking_cli/utils/ollama.py`, `bot/vikingbot/config/schema.py` and `loader.py` (sandbox defaults, server-managed auth), `vikingbot/agent/tools/factory.py` (exec tool gated by `allow_exec`), `vikingbot/sandbox/backends/direct.py`

Image facts were read through the GHCR registry API without Docker: tag list, OCI index (amd64 + arm64), image config (env, entrypoint, labels) and the file list and Python sources of the virtualenv layer (to confirm installed packages, e.g. no `llama_cpp`). No upstream code was executed.

## Market templates

The two Railway marketplace templates in `_market/` (`openviking`, `openviking-aug-26-agent-memory`) were read for their service layout and settings. They carry no license. Ideas that come from upstream itself (seeding `ov.conf` through `OPENVIKING_CONF_CONTENT`, `${VAR}` placeholders, volume at `/app/.openviking`) were re-implemented from upstream's documentation and source; no README, description or configuration text from the market templates is reused.

## Logo

The marketplace icon is OpenViking's own `docs/images/ov-logo-square.png`, served from the v0.5.1 tag via jsDelivr, used only to identify the software.
