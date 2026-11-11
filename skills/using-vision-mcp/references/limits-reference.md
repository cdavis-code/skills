# Limits & Best Practices

## Constraints

### Image Size Limits

| Constraint | Limit |
|------------|-------|
| Maximum image file size | 20 MB |
| Maximum JSON request body | 10 MB |
| Base64 inflation | ~33% (a 7.5 MB file produces ~10 MB base64) |
| Safe base64 threshold | Files under ~7.5 MB — above this, use `imageUrl` or `/upload` |

### Resolution Recommendations

| Feature | Recommended Resolution |
|---------|----------------------|
| OCR (text / document) | 1024×768 |
| Face detection | 1600×1200 |
| Label / logo / landmark detection | 640×480 (sufficient) |
| Safe search | 640×480 (sufficient) |
| Crop hints | 640×480 (sufficient) |
| Object localization | 640×480 (sufficient) |
| Web detection | 640×480 (sufficient) |
| Image properties | Any resolution |

## Image Input Strategies

### Strategy 1: `imageUrl` (Preferred)

Use a public URL whenever the image is accessible online. This completely avoids the 10 MB JSON body limit and base64 encoding overhead.

Supported URL schemes:
- `http://` and `https://` — public web URLs
- `gs://` — Google Cloud Storage URIs (from the `/upload` endpoint)

```json
{
  "imageUrl": "https://example.com/photo.jpg",
  "maxResults": 10
}
```

### Strategy 2: `imageBase64` (Small Files Only)

For small local files, pass raw base64 directly. Do not include the `data:image/...;base64,` prefix.

```json
{
  "imageBase64": "/9j/4AAQSkZJRgABAQAAAQABAAD...",
  "maxResults": 10
}
```

**Only use this for files under ~7.5 MB.** Larger files will exceed the 10 MB JSON body limit.

### Strategy 3: `/upload` Endpoint (Large Local Files > 256 KB)

For local files over 256 KB that aren't publicly accessible, upload to GCS first:

```bash
curl -s -X POST \
  --data-binary @path/to/image.jpg \
  -H "Content-Type: image/jpeg" \
  https://vision-mcp-XXXX.run.app/upload
```

The response returns an `imageUrl` with a `gs://` URI:
```json
{"imageUrl": "gs://vision-mcp-uploads-vision-495818/3f2a1b4c-....jpg"}
```

Use that `imageUrl` in subsequent detection calls. Uploaded files have a 1-day deletion lifecycle.

**Supported content types for `/upload`:**
- `image/jpeg` → `.jpg`
- `image/png` → `.png`
- `image/gif` → `.gif`
- `image/webp` → `.webp`
- `image/tiff` → `.tiff`
- `image/bmp` → `.bmp`
- `application/pdf` → `.pdf`

## Best Practices

### General

1. **Prefer `imageUrl` over `imageBase64`** — avoids payload limits and is faster
2. **Use the right tool for the text density** — `vision_document_text_detection` for dense documents, `vision_text_detection` for sparse text
3. **Set `maxResults` lower than default** when you only need top results — reduces response size and latency
4. **Provide `languageHints` for OCR** — improves accuracy significantly for non-English text
5. **Resize images before uploading** if they exceed recommended resolutions — Vision API will downsample anyway

### Performance

6. **Batch detection calls sequentially**, not in parallel, to avoid rate limiting
7. **For multiple features on one image**, call each tool separately with the same image — the Vision API does not charge extra for calling multiple features on the same image in Cloud Run (each call incurs cost independently)
8. **Set `min-instances=0` for dev/test deployments** to avoid idle costs — instances scale to zero when idle, with cold starts on first request
9. **Use Cloud Run `--memory=512Mi` as a starting point** — the server is lightweight and doesn't need more for most workloads

### Security

10. **Never hardcode credentials in source code** — always use environment variables or ADC
11. **For Cloud Run deployments, use `--allow-unauthenticated` with caution** — consider requiring auth and using `gcloud run services proxy` if the data is sensitive
12. **The `/upload` endpoint requires `UPLOAD_BUCKET_NAME` to be configured** — it returns an error otherwise
13. **Uploaded files are stored for 1 day** — the bucket has a lifecycle policy; do not rely on long-term storage

### OCR-Specific

14. **For multi-language documents**, pass all relevant language codes in `languageHints`: `"en,fr,de,it,es"`
15. **For documents with small text**, use higher resolution images (at least 1024×768)
16. **`vision_document_text_detection`** returns structured output (pages→blocks→paragraphs→words→symbols) — parse this for layout-aware extraction
17. **`vision_text_detection`** returns a flat list — `textAnnotations[0].description` gives the full concatenated text

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using `vision_text_detection` for dense documents | Use `vision_document_text_detection` for contracts, books, forms, receipts |
| Using `vision_document_text_detection` for sparse text | Use `vision_text_detection` for signs, labels, single words |
| Passing a local image as base64 over ~7.5 MB | Use `imageUrl` or `/upload` — files that size produce >10 MB JSON and will be rejected |
| Prefixing base64 with `data:image/...;base64,` | Pass the raw base64 string only |
| Forgetting to set `languageHints` for non-English OCR | Always pass `languageHints` for better accuracy |
| Expecting `/upload` to work locally when `UPLOAD_BUCKET_NAME` is not set | The `/upload` endpoint requires a GCS bucket — only available in Cloud Run deployments with the env var configured |
| Calling multiple vision tools in parallel | Run them sequentially to avoid rate limiting and ensure consistent results |
| Using `maxResults` higher than needed | Set to the minimum you actually need (e.g., 3 instead of 10) |
| Not resizing high-resolution images before base64 encoding | Resize to recommended resolution first to stay under limits and reduce latency |
