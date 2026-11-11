# Tools Reference

Detailed tool-by-tool reference for the Vision MCP server. All tools are accessed exclusively via HTTP transport — there is no CLI or local daemon path.

## Common Parameters

All detection tools accept these shared parameters:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `imageUrl` | string | * | Public URL of the image to analyze. Preferred over base64 to avoid JSON payload size limits (10 MB max). Supports `http://`, `https://`, and `gs://` URIs. |
| `imageBase64` | string | * | Raw base64-encoded image data. Do NOT include the `data:image/...;base64,` prefix. Note: 10 MB JSON request limit applies. Base64 inflates size by ~33%. |
| `maxResults` | integer | no | Maximum number of results to return per feature. Default: 10. |

\* Either `imageUrl` or `imageBase64` must be provided.

### Additional Parameters

| Parameter | Type | Used By | Description |
|-----------|------|---------|-------------|
| `languageHints` | string | `text_detection`, `document_text_detection` | Comma-separated BCP-47 language codes (e.g. `"en,fr,zh-Hans"`). Helps improve OCR accuracy. |

## Tool Catalog

### `vision_crop_hints`

Suggests vertices for a crop region on an image.

**Parameters:** `imageUrl` / `imageBase64`, `maxResults`

**Example payload:**
```json
{
  "imageUrl": "https://example.com/photo.jpg",
  "maxResults": 5
}
```

**Typical output:** A list of crop hint annotations, each containing bounding polygon vertices and confidence scores.

---

### `vision_document_text_detection`

Extracts text from an image, optimized for dense text and documents (contracts, books, forms, receipts, multi-column layouts). Uses the `DOCUMENT_TEXT_DETECTION` feature which provides paragraph and block structure.

**Parameters:** `imageUrl` / `imageBase64`, `maxResults`, `languageHints`

**When to use:** Dense, multi-line, or multi-paragraph text. Structured documents.

**Example payload:**
```json
{
  "imageUrl": "https://example.com/receipt.jpg",
  "maxResults": 1,
  "languageHints": "en"
}
```

**Typical output:** `fullTextAnnotation` with pages, blocks, paragraphs, words, and symbols hierarchy. Full extracted text in `fullTextAnnotation.text`.

---

### `vision_face_detection`

Detects multiple faces within an image along with emotional state and headwear. Returns face bounding polygons, landmark positions (eyes, nose, mouth), and likelihood ratings for emotions (joy, sorrow, anger, surprise) and attributes (headwear, blur, under-exposure).

**Parameters:** `imageUrl` / `imageBase64`, `maxResults`

**Recommended image resolution:** 1600×1200

**Example payload:**
```json
{
  "imageUrl": "https://example.com/group_photo.jpg",
  "maxResults": 20
}
```

**Typical output:** Array of face annotations with `boundingPoly`, `fdBoundingPoly`, `landmarks`, `joyLikelihood`, `sorrowLikelihood`, `angerLikelihood`, `surpriseLikelihood`, `headwearLikelihood`, etc.

---

### `vision_image_properties`

Detects general attributes of the image, such as dominant colors.

**Parameters:** `imageUrl` / `imageBase64`, `maxResults`

**Example payload:**
```json
{
  "imageUrl": "https://example.com/design.jpg",
  "maxResults": 10
}
```

**Typical output:** `dominantColors` with `colors` array — each containing RGB values, pixel fraction, and score.

---

### `vision_label_detection`

Identifies general objects, locations, activities, animal species, products, and more.

**Parameters:** `imageUrl` / `imageBase64`, `maxResults`

**Example payload:**
```json
{
  "imageUrl": "https://example.com/scenery.jpg",
  "maxResults": 15
}
```

**Typical output:** Array of `LabelAnnotation` objects with `description` (label name), `score` (confidence), and `topicality`.

---

### `vision_landmark_detection`

Detects popular natural and human-made structures within an image. Returns landmark name, geographic coordinates (latitude/longitude), bounding polygon, and confidence score.

**Parameters:** `imageUrl` / `imageBase64`, `maxResults`

**Example payload:**
```json
{
  "imageUrl": "https://example.com/eiffel_tower.jpg",
  "maxResults": 5
}
```

**Typical output:** Array of `EntityAnnotation` objects with `description` (landmark name), `mid` (knowledge graph ID), `score`, `boundingPoly`, and `locations` with lat/lng.

---

### `vision_logo_detection`

Detects popular product logos within an image.

**Parameters:** `imageUrl` / `imageBase64`, `maxResults`

**Example payload:**
```json
{
  "imageUrl": "https://example.com/storefront.jpg",
  "maxResults": 10
}
```

**Typical output:** Array of logo annotations with `description` (brand name), `mid`, `score`, and `boundingPoly`.

---

### `vision_object_localization`

Identifies multiple objects in an image and provides rectangular bounding boxes for each.

**Parameters:** `imageUrl` / `imageBase64`, `maxResults`

**Example payload:**
```json
{
  "imageUrl": "https://example.com/room.jpg",
  "maxResults": 20
}
```

**Typical output:** Array of `LocalizedObjectAnnotation` objects with `name`, `score`, and `boundingPoly` (normalized vertices).

---

### `vision_product_search`

Run Product Search against a pre-configured product set. Returns visually similar products with metadata, bounding boxes, and match scores. Requires a Vision API Product Search catalog to be set up in your Google Cloud project.

**Parameters:** `imageUrl` / `imageBase64`, `maxResults`

**Example payload:**
```json
{
  "imageUrl": "https://example.com/shoe.jpg",
  "maxResults": 5
}
```

**Typical output:** `productSearchResults` with grouped results, product labels, confidence scores, and bounding polygons.

---

### `vision_safe_search_detection`

Detects explicit content such as adult content or violent content within an image. Returns likelihood ratings for adult, spoof, medical, violence, and racy content.

**Parameters:** `imageUrl` / `imageBase64`, `maxResults`

**Example payload:**
```json
{
  "imageUrl": "https://example.com/uploaded_image.jpg"
}
```

**Typical output:** `safeSearchAnnotation` with likelihood enums (`UNKNOWN`, `VERY_UNLIKELY`, `UNLIKELY`, `POSSIBLE`, `LIKELY`, `VERY_LIKELY`) for each category.

---

### `vision_text_detection`

Detects and extracts sparse text from an image (e.g., street signs, labels, single words). Uses the `TEXT_DETECTION` feature. For dense multi-line documents, prefer `vision_document_text_detection`.

**Parameters:** `imageUrl` / `imageBase64`, `maxResults`, `languageHints`

**When to use:** Sparse or isolated text — signs, labels, product packaging, single-line text. NOT for documents or multi-paragraph content.

**Example payload:**
```json
{
  "imageUrl": "https://example.com/street_sign.jpg",
  "maxResults": 10,
  "languageHints": "en"
}
```

**Typical output:** Array of `EntityAnnotation` objects. The first element (`textAnnotations[0]`) contains the full extracted text in its `description` field. Subsequent elements are per-word annotations with bounding boxes.

---

### `vision_web_detection`

Detects Web references to an image, including web entities (labels inferred from the web), visually similar images, pages with matching images, and pages with fully or partially matching images.

**Parameters:** `imageUrl` / `imageBase64`, `maxResults`

**Example payload:**
```json
{
  "imageUrl": "https://example.com/mystery_object.jpg",
  "maxResults": 10
}
```

**Typical output:** `webDetection` object containing:
- `webEntities` — entities with `description` and `score`
- `fullMatchingImages` / `partialMatchingImages` — URLs of matching images
- `pagesWithMatchingImages` — URLs of pages containing similar images
- `visuallySimilarImages` — URLs of visually similar images

## Tool Decision Rules

Follow this priority when choosing which tool to use:

| User wants to… | Use this tool |
|----------------|---------------|
| Read text from a document, form, receipt, or book page | `vision_document_text_detection` |
| Read sparse text from a sign, label, or single word | `vision_text_detection` |
| Identify what objects/scenes are in an image | `vision_label_detection` |
| Find faces and analyze emotions | `vision_face_detection` |
| Check if an image contains explicit content | `vision_safe_search_detection` |
| Identify a famous building or natural landmark | `vision_landmark_detection` |
| Find brand/product logos in an image | `vision_logo_detection` |
| Get bounding boxes for objects | `vision_object_localization` |
| Find where an image appears on the web | `vision_web_detection` |
| Get dominant colors from an image | `vision_image_properties` |
| Get crop suggestions for an image | `vision_crop_hints` |
| Search for a product visually | `vision_product_search` |

### Composing Multiple Detections

You can call multiple tools sequentially on the same image. Common combinations:

- **Photo analysis:** `vision_label_detection` + `vision_face_detection` + `vision_safe_search_detection`
- **Document understanding:** `vision_document_text_detection` + `vision_label_detection`
- **Brand monitoring:** `vision_logo_detection` + `vision_web_detection` + `vision_label_detection`
- **Content moderation:** `vision_safe_search_detection` + `vision_text_detection`
- **Landmark tourism:** `vision_landmark_detection` + `vision_label_detection` + `vision_web_detection`
