# Deploy and Host OpenViking with Railway

OpenViking, an open-source project from Volcengine (AGPL-3.0), is a context database for AI agents. It keeps an agent's memory, its knowledge base and its skills in one workspace and serves them through a REST API, an MCP endpoint and the Studio web console. This template runs OpenViking v0.5.1 with persistent storage and authentication on.

## About Hosting OpenViking

OpenViking stores context like a file system (`viking://resources`, `viking://~/memories`, `viking://~/skills`). Every added file or conversation is summarized into short abstracts and overviews by an LLM and indexed with embeddings, so agents can load only the level of detail they need. Self-hosting it means running the Python server with its embedded file store and vector index on a persistent disk, configuring an embedding model and an LLM, and protecting the API with keys. This community template uses the official v0.5.1 image, keeps all data on a Railway volume, turns on API-key authentication with a generated root key, and enables OAuth so web MCP clients can connect over HTTPS.

## Common Use Cases

- Long-term memory for coding agents such as Claude Code, Codex, Cursor or OpenCode, shared across sessions and machines through MCP
- A searchable knowledge base of documents, code repositories and web pages that agents read at the right level of detail
- A shared skill library that several agents load on demand
- Per-user memory for chatbots and assistants, with separate accounts and keys for each team or customer
- Inspecting and editing what agents know through the Studio web console

## Dependencies for OpenViking Hosting

- An API key for an OpenAI-compatible endpoint (required). It covers embeddings and the LLM by default; other providers such as Volcengine Ark, Azure OpenAI, OpenRouter, Jina, Voyage, Gemini or LiteLLM can be set per model.
- A Railway volume for the configuration, files and vector index (included).

### Deployment Dependencies

- OpenViking repository: https://github.com/volcengine/OpenViking
- OpenViking documentation: https://docs.openviking.ai
- Server deployment guide: https://github.com/volcengine/OpenViking/blob/v0.5.1/docs/en/guides/03-deployment.md
- Model configuration: https://github.com/volcengine/OpenViking/blob/v0.5.1/docs/en/guides/01-configuration.md
- Authentication: https://github.com/volcengine/OpenViking/blob/v0.5.1/docs/en/guides/04-authentication.md
- MCP integration: https://github.com/volcengine/OpenViking/blob/v0.5.1/docs/en/guides/06-mcp-integration.md
- Container image: https://github.com/volcengine/OpenViking/pkgs/container/openviking

### Implementation Details

| Service | Image | Purpose |
|---|---|---|
| OpenViking | `ghcr.io/volcengine/openviking:v0.5.1` | REST API, MCP endpoint (`/mcp`), Studio (`/studio`) and the built-in VikingBot on port 1933 (public); file store and vector index on the volume `/app/.openviking`; healthcheck `/ready` |

**Required input**: `OPENAI_API_KEY`. The same key is used for embeddings (`text-embedding-3-small`) and for the LLM that writes summaries and extracts memories (`gpt-5.4`). To use another endpoint, change `OPENAI_API_BASE`; to split providers, set the `OV_VLM_*` and `OV_EMBEDDING_*` variables. Choose the embedding model before adding data: a different model needs a reindex.

**First login**

1. Open `https://{your-domain}/studio` and paste `OPENVIKING_ROOT_API_KEY` (from the service's Variables tab) into Studio's connection settings.
2. On the users page, create an account and its first admin user. Copy the user key that is shown; it is displayed once.
3. Use the user key everywhere else. The root key manages accounts and users only and cannot read data.
4. REST: `curl 'https://{your-domain}/api/v1/fs/ls?uri=viking://' -H "X-API-Key: {your-user-key}"`. Python: `pip install openviking-sdk`, then `openviking_sdk.SyncHTTPClient(url="https://{your-domain}", api_key="{your-user-key}")`.
5. MCP: `claude mcp add --transport http openviking https://{your-domain}/mcp --header "Authorization: Bearer {your-user-key}"`. Claude.ai, ChatGPT and other OAuth clients can add `https://{your-domain}/mcp` as a connector and approve access in Studio.

**Security defaults**: authentication mode `api_key` is set explicitly, so the server refuses to start without the generated root key. Every data, MCP and bot request needs a user key; only health, readiness, the API schema, OAuth discovery and the Studio page itself are public. The built-in bot (Studio chat) runs with shell execution turned off. Model keys stay in Railway variables and are referenced from `ov.conf` as `${VAR}`, not written into it.

**Configuration**: on first start the service writes `ov.conf` to the volume from `OPENVIKING_CONF_CONTENT`. Its `${VAR}` placeholders are filled from the service variables on every start, so changing a model, key or log level only needs a redeploy. Set `OPENVIKING_CONF_MODE=overwrite` to rewrite the file from the template variable on each start (the old file is kept as `ov.conf.previous`). `OPENVIKING_WITH_BOT=0` runs the API without the bot. `OPENVIKING_GATEWAY_ENABLED=true` adds OpenViking Gateway under `/gateway`.

**Scaling**: OpenViking runs as one process with an embedded vector index that locks its workspace, so scale it vertically (more RAM and CPU) rather than with replicas. Hobby volumes hold up to 5 GB; larger knowledge bases need the Pro plan.

**Pinning and upgrades**: the image tag is `v0.5.1`. To upgrade, read the release notes, back up the volume, change the tag and redeploy. Newer releases can write data that older ones cannot read, so keep the backup until the new version is confirmed.

**Resources**: about 0.8 GB RAM at idle (server plus bot), roughly $8 to $9 per month, and about $20 to $25 per month with regular use, plus the model provider's charges. Use the Hobby plan or higher.

### Why Deploy OpenViking on Railway?

Railway gives OpenViking a persistent volume, HTTPS on a public domain and variables for its keys and models, so your agents get a private context store with MCP and OAuth support in a few minutes. Your memories and documents stay on your own volume, the defaults are locked down, and models can be switched from the dashboard.
