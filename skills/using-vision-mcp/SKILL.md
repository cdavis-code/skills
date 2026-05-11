---
name: using-vision-mcp
description: Use when analyzing images through Vision MCP — text detection, document OCR, label/face/object detection — or when hitting the 10 MB JSON limit, base64 encoding fails, or needing OCR language hints.
---

# Using Vision MCP

## Available Tools

All tools accept `imageUrl` (preferred) or `imageBase64` (raw base64 string — no `data:image/...;base64,` prefix), plus an optional `maxResults` (default: 10).

**Text in an image?** Use `vision_text_detection` for sparse text (signs, labels, street text). Use `vision_document_text_detection` for pages, forms, receipts, and any dense multi-line content.

| Tool | Notes |
|------|-------|
| `vision_crop_hints` | Suggests crop regions |
| `vision_document_text_detection` | Dense text / documents. Accepts `languageHints` |
| `vision_face_detection` | Faces, emotions, landmarks |
| `vision_image_properties` | Dominant colors |
| `vision_label_detection` | General objects, scenes, activities. Returns `description` + `score` per label |
| `vision_landmark_detection` | Famous natural and human-made structures |
| `vision_logo_detection` | Product logos |
| `vision_object_localization` | Object bounding boxes |
| `vision_product_search` | Product search |
| `vision_safe_search_detection` | Explicit content likelihood |
| `vision_text_detection` | Sparse text (signs, labels). Accepts `languageHints`. Full text in `textAnnotations[0].description` |
| `vision_web_detection` | Web entity and similar-image lookup |

## Key Limits

| Constraint | Limit |
|------------|-------|
| Image file size | 20 MB max |
| JSON request body | 10 MB max (base64 inflates by ~33%) |
| Safe base64 threshold | ~7.5 MB file size (exceeding produces >10 MB JSON) |
| OCR recommended resolution | 1024×768 |
| Face detection recommended | 1600×1200 |
| Other features | 640×480 is sufficient |

**Use `imageUrl` over `imageBase64`** whenever the image is publicly accessible — it avoids the JSON payload limit entirely.

## Handling Large Local Files (> 256 KB)

Base64 inflates file size by ~33%, so files over ~7.5 MB will exceed the 10 MB JSON limit. Use `/upload` for any local file over **256 KB** to avoid the encoding overhead entirely.

The server exposes a `/upload` endpoint that stores the file in GCS and returns a `gs://` URI. The base URL must match the deployed service endpoint.

**1. Upload the file and capture the response:**
```bash
curl -s -X POST \
  --data-binary @path/to/image.jpg \
  -H "Content-Type: image/jpeg" \
  https://vision-mcp-158920883829.northamerica-northeast2.run.app/upload
```

The response body contains the `imageUrl` to use — for example:
```json
{"imageUrl": "gs://vision-mcp-uploads-vision-495818/3f2a1b4c-....jpg"}
```

**2. Pass that exact `imageUrl` value to any vision tool:**
```json
{
  "imageUrl": "gs://vision-mcp-uploads-vision-495818/3f2a1b4c-....jpg",
  "maxResults": 10
}
```

The `gs://` URI comes from the response — do not construct it manually. The bucket has a 1-day deletion lifecycle.

## OCR Language Hints

Pass BCP-47 codes as a comma-separated string (e.g. `"en,fr,zh-Hans"`):
- `vision_text_detection` — auto-detects a subset of languages if omitted
- `vision_document_text_detection` — auto-detects the full set if omitted

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using `vision_text_detection` for dense documents | Use `vision_document_text_detection` for contracts, books, forms, receipts |
| Passing a local image as base64 over ~7.5 MB | Use `/upload` — files that size produce >10 MB JSON and will be rejected |
| Prefixing base64 with `data:image/...;base64,` | Pass the raw base64 string only |
