# Setup Reference

The Vision MCP server is accessed exclusively via HTTP transport (Streamable HTTP / SSE). There is no CLI or local daemon path.

## Supported Connection Paths

### Path 1: Local Development Server

Use when you are running the server locally with `dart run vision_mcp`.

**Prerequisites:**
- Dart SDK ^3.9.0
- A Google Cloud project with the Vision API enabled
- For local auth: a service account JSON key file

**1. Start the server:**

```bash
cd mcp
export VISION_CREDENTIALS_FILE=/absolute/path/to/service_account.json
dart run vision_mcp
```

The server starts on `http://127.0.0.1:8080` (or the `PORT` env var if set).

**2. Connect an MCP client:**

Any MCP client that supports HTTP/SSE transport can connect. Examples:

- **MCP Inspector** (testing):
  ```bash
  npx @modelcontextprotocol/inspector --stdio false --http http://127.0.0.1:8080
  ```
  Or open the browser UI at `http://localhost:5173`, select **Via Proxy**, Server URL: `http://127.0.0.1:8080`.

- **Client configuration** (e.g., Claude Desktop, Qoder, etc.):
  Add a Streamable HTTP connection pointing to `http://127.0.0.1:8080`.

### Path 2: Cloud Run Deployment

Use when the server is deployed to Google Cloud Run and accessible via a public URL or authenticated proxy.

**Prerequisites:**
- Google Cloud project with Vision API, Cloud Run, Secret Manager, and Cloud Build APIs enabled
- gcloud CLI installed and authenticated
- Cloud Run service account with Vision API permissions (ADC is automatic)

**1. Deploy:**

```bash
cd mcp
./deploy.sh -p YOUR_PROJECT_ID
```

Or manually:

```bash
gcloud builds submit --tag gcr.io/YOUR_PROJECT_ID/vision-mcp .
gcloud run deploy vision-mcp \
  --image=gcr.io/YOUR_PROJECT_ID/vision-mcp \
  --region=northamerica-northeast2 \
  --allow-unauthenticated \
  --memory=512Mi --cpu=1 --min-instances=0
```

**2. Get the service URL:**

```bash
gcloud run services describe vision-mcp \
  --region=northamerica-northeast2 \
  --format="value(status.url)"
```

**3. Connect an MCP client:**

If the service is `--allow-unauthenticated`, connect directly to the URL:
```bash
npx @modelcontextprotocol/inspector --stdio false --https https://vision-mcp-XXXX.run.app
```

If the service requires authentication, use `gcloud run services proxy` to create an authenticated local tunnel:
```bash
gcloud run services proxy vision-mcp \
  --region=northamerica-northeast2 --port=8080
```
Then connect to `http://localhost:8080`.

### Path 3: Existing HTTP Endpoint

Use when a Vision MCP server is already deployed and reachable at a known URL (deployed by someone else or in another workspace).

**1. Collect prerequisites:**
- Confirm the endpoint URL
- Confirm whether the endpoint requires authentication
- If authenticated, confirm you have the required credentials or proxy access

**2. Connect an MCP client:**
Configure your MCP client with a Streamable HTTP connection to the known URL. If authentication is required, use `gcloud run services proxy` or equivalent tunneling.

## Authentication Mechanisms

The Vision MCP server supports two authentication modes, tried in order:

### 1. Application Default Credentials (ADC) — Cloud Run

Automatically used when running in Cloud Run or any GCE/GKE environment with a metadata server. The server calls the metadata server at `http://metadata.google.internal` to obtain short-lived OAuth tokens. No configuration needed — the Cloud Run service account needs Vision API permissions.

### 2. Service Account JWT — Local Development

Fallback for local development. Set the `VISION_CREDENTIALS_FILE` environment variable to the absolute path of a service account JSON key file. The server uses JWT authentication with the `cloud-platform` scope.

```bash
export VISION_CREDENTIALS_FILE=/absolute/path/to/service_account.json
```

## Prerequisites Checklist

Before connecting, verify:

- [ ] Vision API is enabled in your Google Cloud project
- [ ] For Cloud Run: service account has `roles/vision.user` (or broader) permission
- [ ] For local dev: `VISION_CREDENTIALS_FILE` points to a valid service account JSON
- [ ] Server is running and listening (check startup log: "MCP HTTP server listening on port …")
- [ ] No firewall or network restrictions block the server port

## Validation Target

After setup, validate connectivity:

1. The MCP client should report successful initialization
2. Tools list should include all `vision_*` tools (12 tools)
3. Run a smoke test with `vision_label_detection` on a known public image:
   ```json
   {
     "imageUrl": "https://raw.githubusercontent.com/faithoflifedev/google_vision_workspace/main/packages/google_vision_cli/example/cn_tower.jpg",
     "maxResults": 3
   }
   ```

Result states:
- `capability active` — tools are visible and a detection call succeeds
- `setup completed, activation pending reconnect` — server is running but client needs reconnection
- `setup incomplete or failed` — server is not reachable or auth is misconfigured

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| "ADC initialization failed" in logs | Not in Cloud Run and no credentials file set | Set `VISION_CREDENTIALS_FILE` |
| "Credentials file not found" | Path in `VISION_CREDENTIALS_FILE` is wrong | Use an absolute path |
| 403 responses from tools | Service account lacks Vision API permissions | Grant `roles/vision.user` |
| "Method not allowed" (405) | Wrong HTTP method | Use POST for MCP requests |
| MCP Inspector sends duplicate initialize | Known inspector behavior | Server handles this with `_isInitialized` guard |
| Connection refused | Server not running or wrong port | Check server is running, verify port |
| Large file base64 rejected | Over 10 MB JSON request body | Use `/upload` endpoint or `imageUrl` |
