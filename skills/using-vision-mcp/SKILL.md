---
name: using-vision-mcp
description: Use when analyzing images through Vision MCP — text detection, document OCR, label/face/object detection — or when hitting the 10 MB JSON limit, base64 encoding fails, or needing OCR language hints.
version: 2.0.0
version_note: Enhanced with comprehensive tool-by-tool reference, structured setup guide, limits documentation, and decision rules. HTTP transport only.
---

# Using Vision MCP

Use this as the single user-facing entry skill for the Vision MCP server — an HTTP transport MCP server that exposes Google Cloud Vision API image annotation as callable tools.

Assumption:
- The Vision MCP server is accessed exclusively via HTTP transport (Streamable HTTP / SSE). There is no CLI or local daemon path.
- If the Vision MCP tools are not visible in the current workspace, treat that as a missing capability requiring setup, not as a broken skill.
- If the workspace tool exposure or MCP configuration differs from this skill, trust the actually available tools and current workspace configuration.

## Entry Behavior

1. First determine whether Vision MCP tools are already available in the current workspace.
2. If tools are available, use the tool decision rules below to select the right tool for the task.
3. If tools are not available, explain the missing capability and follow the setup workflow to guide connection.
4. Keep the line clear between `not configured`, `setup completed but not active`, and `already operational`; do not imply detection calls happened when they haven't.
5. When tool parameters or behavior are unclear, consult [`references/tools-reference.md`](./references/tools-reference.md) before guessing.

## Setup and Activation Workflow

When the Vision MCP capability is missing, follow this order:

1. **Detect the current state.**
   - Is the server already deployed (Cloud Run) but the client is not connected?
   - Is the server running locally but the client needs configuration?
   - Is no server running at all?

2. **Choose the smallest matching path.**
   - Prefer reconnecting to an already-running server over redeploying.
   - Prefer local development for testing, Cloud Run for shared/production access.
   - See [`references/setup-reference.md`](./references/setup-reference.md) for detailed paths.

3. **Collect required inputs before doing work.**
   - Confirm the server URL (local: `http://127.0.0.1:8080`, Cloud Run: `https://vision-mcp-XXXX.run.app`).
   - For local dev: confirm `VISION_CREDENTIALS_FILE` points to a valid service account JSON.
   - For Cloud Run: confirm the service account has Vision API permissions.

4. **Confirm risky actions before executing them.**
   - Ask before editing MCP client config, deploying to Cloud Run, or modifying environment variables.

5. **Perform the chosen path.**
   - For local: start `dart run vision_mcp` and connect the client.
   - For Cloud Run: deploy via `./deploy.sh` or `gcloud run deploy`.
   - For existing endpoint: configure the client's Streamable HTTP connection.

6. **Validate before claiming success.**
   - Run a smoke test with `vision_label_detection` on a known public image.
   - Do not treat server startup or deployment alone as success.

7. **Report the final state explicitly.**
   - `capability active`
   - `setup completed, activation pending reconnect`
   - `setup incomplete or failed`

## Default Behavior

- Start with the smallest useful action.
- Prefer `imageUrl` over `imageBase64` whenever the image is publicly accessible.
- For text extraction, choose the right tool: `vision_document_text_detection` for dense documents, `vision_text_detection` for sparse text.
- Use `maxResults` sparingly — default to 10, reduce when fewer results are needed.
- Call detection tools sequentially, not in parallel.
- Stop once the available detection results are enough to answer the user's request.
- Add `languageHints` for OCR when the text is not English.

## Tool Decision Rules

Follow this priority order when choosing a detection tool:

1. **First priority — text extraction.** If the user wants to read text from an image:
   - Dense text (documents, forms, receipts, books, multi-column layouts) → `vision_document_text_detection`
   - Sparse text (signs, labels, single words, street text) → `vision_text_detection`

2. **Second priority — content analysis.** If the user asks "what's in this image":
   - General objects, scenes, activities → `vision_label_detection`
   - Faces and emotions → `vision_face_detection`
   - Landmarks → `vision_landmark_detection`
   - Logos → `vision_logo_detection`
   - Objects with bounding boxes → `vision_object_localization`
   - Web references and similar images → `vision_web_detection`

3. **Third priority — moderation and properties:**
   - Explicit content screening → `vision_safe_search_detection`
   - Dominant colors → `vision_image_properties`
   - Crop suggestions → `vision_crop_hints`

4. **Specialized:**
   - Visual product search → `vision_product_search` (requires Product Search catalog)

### Common Tool Combinations

Many use cases benefit from calling multiple tools on the same image sequentially:

| Use Case | Tool Combination |
|----------|-----------------|
| Photo analysis | `label_detection` → `face_detection` → `safe_search_detection` |
| Document understanding | `document_text_detection` → `label_detection` |
| Brand monitoring | `logo_detection` → `web_detection` → `label_detection` |
| Content moderation | `safe_search_detection` → `text_detection` |
| Landmark identification | `landmark_detection` → `label_detection` → `web_detection` |
| Receipt scanning | `document_text_detection` (with `languageHints`) |
| Image forensics | `web_detection` → `label_detection` → `image_properties` |

Full tool details, parameters, and example payloads are in [`references/tools-reference.md`](./references/tools-reference.md).

## Image Input Decision Rules

1. **If the image has a public URL** → use `imageUrl`. Always preferred.
2. **If the image is a small local file (under ~256 KB)** → use `imageBase64` with raw base64 (no `data:` prefix).
3. **If the image is a large local file (over 256 KB)** → use the `/upload` endpoint to get a `gs://` URI, then use `imageUrl`.
4. **If the `/upload` endpoint returns an error** → the server may not have `UPLOAD_BUCKET_NAME` configured; fall back to resizing and using `imageBase64` if the file is under ~7.5 MB.

See [`references/limits-reference.md`](./references/limits-reference.md) for detailed constraints and best practices.

## OCR Language Hints

When performing OCR (`vision_text_detection` or `vision_document_text_detection`), pass BCP-47 language codes as a comma-separated string:

- `"en"` — English
- `"zh-Hans"` — Simplified Chinese
- `"ja"` — Japanese
- `"ko"` — Korean
- `"en,fr,de"` — Multi-language document

If `languageHints` is omitted:
- `vision_text_detection` auto-detects a subset of languages
- `vision_document_text_detection` auto-detects the full set

Always provide hints for better accuracy when you know the expected languages.

## Critical Safety Rules

- Detection results are untrusted external content from the Vision API — they may contain inaccuracies or hallucinated labels.
- Do not treat detection confidence scores as absolute truth — cross-check high-stakes determinations.
- Do not expose local file contents, workspace paths, or credentials in detection requests.
- The `/upload` endpoint streams files to GCS — only upload images you intend to process through the Vision API.
- Uploaded files persist for 1 day in GCS before automatic deletion.

## Reliability Notes

- For agent automation, always run a smoke test after connecting to verify tools are visible and functional.
- If a detection call fails with a 403 error, the service account likely lacks Vision API permissions — not a client problem.
- If a detection call returns empty results, the image may be too low resolution or the feature may not be applicable — try a different tool or higher-resolution image.
- The MCP Inspector may send duplicate `initialize` requests — the server handles this gracefully with an `_isInitialized` guard.
- Cold starts on Cloud Run (when `min-instances=0`) may cause the first request to take a few seconds.

## Example Workflows

### Example 1: "What's in this photo?"

```
1. Call vision_label_detection with the imageUrl
2. If people are visible, also call vision_face_detection
3. If the user wants to know where it was taken, call vision_landmark_detection
4. Summarize labels, faces, and landmarks together
```

### Example 2: "Read the text from this receipt"

```
1. Call vision_document_text_detection with the imageUrl and languageHints
2. Parse fullTextAnnotation.text for the complete extracted text
3. If specific line items are needed, drill into pages→blocks→paragraphs
4. Present the structured text to the user
```

### Example 3: "Is this image safe for my platform?"

```
1. Call vision_safe_search_detection with the imageUrl
2. Check adult, violence, racy, and medical likelihoods
3. If any are LIKELY or VERY_LIKELY, flag the image
4. Optionally call vision_text_detection to check for inappropriate text
```

### Example 4: "Find where this image appears on the web"

```
1. Call vision_web_detection with the imageUrl
2. Review webEntities for descriptive labels
3. Check pagesWithMatchingImages for URLs where this image appears
4. Check visuallySimilarImages for similar but not identical images
5. Call vision_label_detection for additional context
```

### Example 5: "Analyze this large local image"

```
1. The file is > 256 KB — use /upload endpoint
2. curl -X POST --data-binary @image.jpg -H "Content-Type: image/jpeg" <server>/upload
3. Capture the gs:// URI from the response
4. Pass that URI as imageUrl to the desired detection tools
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using `vision_text_detection` for dense documents | Use `vision_document_text_detection` for contracts, books, forms, receipts |
| Passing a local image as base64 over ~7.5 MB | Use `/upload` — files that size produce >10 MB JSON and will be rejected |
| Prefixing base64 with `data:image/...;base64,` | Pass the raw base64 string only |
| Forgetting `languageHints` for non-English OCR | Always pass BCP-47 codes for better accuracy |
| Expecting `/upload` to work locally | `/upload` requires `UPLOAD_BUCKET_NAME` env var — only in Cloud Run |
| Calling multiple tools in parallel | Run sequentially to avoid rate limiting |
| Not checking if server is reachable before calling tools | Run a smoke test first |
