# unlimited.surf Transfer API Worker

Cloudflare Worker adapter for `https://unlimited.surf` that exposes OpenAI-compatible `/v1/*` routes and Anthropic-compatible `/v1/messages` plus `/anthropic/*` aliases.

## Deploy from GitHub with Cloudflare

You can push this project to GitHub and let Cloudflare deploy it automatically on every commit.

### 1. Push the project to GitHub

Create a GitHub repository, then push this project directory to it. Do not commit your unlimited.surf API key.

### 2. Connect the GitHub repository in Cloudflare

1. Open the Cloudflare Dashboard.
2. Go to `Workers & Pages`.
3. Choose `Create`.
4. Choose `Import a repository` or `Connect to Git`.
5. Authorize Cloudflare to access your GitHub account if prompted.
6. Select the repository that contains this project.
7. Set the project root directory to `/` unless you put this project in a subdirectory.

### 3. Configure build and deploy settings

Use these settings in the Cloudflare deploy form:

```text
Framework preset: None
Build command: npm install
Deploy command: npx wrangler deploy
Root directory: /
Wrangler config: wrangler.toml
```

If Cloudflare shows only one command field for Workers, use:

```bash
npm install && npx wrangler deploy
```

### 4. Add the API key as a Cloudflare secret

After the Worker project is created, add this secret in the Worker settings:

```text
UNLIMITED_SURF_API_KEY=<your unlimited.surf key>
```

In the Cloudflare Dashboard this is usually under:

```text
Workers & Pages -> your Worker -> Settings -> Variables -> Secrets
```

Keep the key as a secret. Do not put it in `wrangler.toml`, `README.md`, or GitHub repository files.

### 5. Deploy and verify

Trigger a deploy from Cloudflare or push a new commit to GitHub. After deployment, open:

```text
https://<your-worker>.workers.dev/health
```

You should see a JSON response with `"ok": true`.

Then test the OpenAI-compatible route:

```bash
curl https://<your-worker>.workers.dev/v1/models
```

And test the Anthropic setup route:

```bash
curl https://<your-worker>.workers.dev/v1/setup
```

## Manual deploy with Wrangler

You can also deploy directly from your local machine:

```powershell
npm install -g wrangler
wrangler login
wrangler secret put UNLIMITED_SURF_API_KEY
wrangler deploy
```

Use your unlimited.surf key as the secret value. You can also omit the secret and pass a key per request with `Authorization: Bearer <key>` or `x-api-key: <key>`.

## OpenAI-compatible routes

Base URL:

```text
https://<your-worker>.workers.dev/v1
```

Supported routes:

- `GET /v1/models`
- `POST /v1/chat/completions`
- `POST /v1/responses`
- `POST /v1/search`
- `POST /v1/merge`
- `GET /v1/key`, `GET /v1/usage`
- `POST /v1/files`
- `POST /v1/files/extract`, `POST /v1/attachments/extract`
- `GET /v1/setup`, `GET /v1/codex`, `GET /v1/mcp`

Example:

```bash
curl https://<your-worker>.workers.dev/v1/chat/completions \
  -H "Authorization: Bearer <key>" \
  -H "Content-Type: application/json" \
  -d '{"model":"gateway-gpt-5","messages":[{"role":"user","content":"Hello"}],"stream":true}'
```

## Anthropic-compatible routes

Base URL:

```text
https://<your-worker>.workers.dev
```

Supported routes:

- `POST /v1/messages`
- `GET /v1/models`
- `POST /anthropic/v1/messages`
- `GET /anthropic/v1/models`
- `GET /v1/setup`, `GET /v1/codex`, `GET /v1/mcp`

Claude Code PowerShell example:

```powershell
$env:ANTHROPIC_BASE_URL = "https://<your-worker>.workers.dev"
$env:ANTHROPIC_AUTH_TOKEN = "<key>"
$env:ANTHROPIC_API_KEY = "<key>"
$env:ANTHROPIC_MODEL = "claude-opus-4-7-20260101"
claude
```

## Feature mapping

- Chat maps to upstream `POST /api/chat`.
- Web Search maps to upstream `POST /api/search` when you call `/v1/search`, pass `web_search_options`, pass `query`, or include a web search tool.
- Merge AI maps to upstream `POST /api/merge` when you call `/v1/merge`, pass `merge: true`, or pass `models` with 2+ model IDs.
- Models maps to upstream `GET /api/models` with a fallback catalog if the upstream call fails.
- Files maps upload/extract requests to upstream `POST /api/attachments/extract`; persistent file storage is not implemented unless you add KV/R2.
- Codex, Agent Setup, and MCP are exposed as setup/info endpoints. MCP servers still run in the client or IDE; this Worker only provides the model endpoint.
- Embeddings, audio, and images return `501` because unlimited.surf does not expose those APIs in the provided docs.

## Upstream raw proxy

Any `/api/*` request is forwarded to unlimited.surf with the configured key, so the original API remains available through the Worker.
