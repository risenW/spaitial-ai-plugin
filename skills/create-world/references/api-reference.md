# SpAItial API reference (v1)

Condensed reference for the SpAItial world-generation API. Full spec:
`https://api.spaitial.ai/v1/openapi.json` · Swagger: `https://api.spaitial.ai/v1/docs`

## Base URL & auth

```
https://api.spaitial.ai
Authorization: Bearer spt_live_<…>   # or spt_test_<…>
```

Scopes by endpoint: `worlds:create` (POST /v1/worlds), `worlds:read` (all GETs),
`worlds:write` (PATCH, cancel, exports POST), `files:create` (POST /v1/files),
`files:read` (GET /v1/files). Panorama edits use `worlds:create`; reading edited
panoramas uses `worlds:read`.

## Endpoints

```
POST   /v1/worlds                                     Create a world job
GET    /v1/worlds/requests                            List jobs for this key
GET    /v1/worlds/requests/:id                        Full job result
GET    /v1/worlds/requests/:id/status                 Cheap status poll (cached 3s)
PATCH  /v1/worlds/requests/:id                        Update title / visibility
POST   /v1/worlds/requests/:id/cancel                 Best-effort cancel
GET    /v1/worlds/requests/:id/splat                  302 -> signed splat URL
GET    /v1/worlds/requests/:id/panorama               302 -> signed panorama URL
POST   /v1/worlds/requests/:id/exports/:type          Start an export (mesh | mesh-simplified)
GET    /v1/worlds/requests/:id/exports/:type          Export status (READY -> download_url)
GET    /v1/worlds/requests/:id/exports                List export statuses
POST   /v1/files                                      Upload an input file -> file_id
GET    /v1/files                                       List uploads
POST   /v1/panoramas/edit                             Edit world/request/panorama -> pano_...
GET    /v1/panoramas                                  List edited panoramas
GET    /v1/panoramas/:id                              Get edited panorama metadata
GET    /v1/panoramas/:id/download                     302 -> signed edited-panorama URL
GET    /v1/models                                      List generation models
```

## POST /v1/worlds — request body

```json
{
  "input": { "type": "url", "image_url": "https://..." },
  "model": "default",
  "title": "My world",
  "output_format": "spz",
  "validation": { "skip": true, "error_on_fail": false },
  "visibility": { "is_public": false, "is_listed": false },
  "webhook": { "url": "https://example.com/hooks/spaitial" }
}
```

| Field | Default | Notes |
|-------|---------|-------|
| `input.type` | — | `url` \| `base64` \| `file_id` \| `text` \| `panorama_id` (discriminated) |
| `model` | account default | `GET /v1/models` to list (`default`, `experimental`, …) |
| `title` | unset | ≤200 chars |
| `output_format` | `spz` | `spz` (default) or `sog` (PlayCanvas-optimized, +~25s) |
| `validation.skip` | `true` | Skip suitability check (saves cost/latency) |
| `validation.error_on_fail` | `false` | With `skip:false`, 422 on flagged input |
| `visibility.is_public` | `false` | Anyone with `viewer_url` can view |
| `visibility.is_listed` | `false` | Public gallery; requires `is_public:true` |
| `webhook.url` | unset | HTTPS callback on terminal state |

### Input shapes

```json
{ "input": { "type": "url",     "image_url": "https://example.com/p.jpg" } }
{ "input": { "type": "url",     "image_url": "https://example.com/pano.jpg", "is_pano": true } }
{ "input": { "type": "base64",  "image_base64": "data:image/png;base64,iVBOR..." } }
{ "input": { "type": "file_id", "file_id": "file_...", "is_pano": false } }
{ "input": { "type": "text",    "prompt": "a cozy sunlit reading nook with bookshelves" } }
{ "input": { "type": "panorama_id", "panorama_id": "pano_..." } }
```

`is_pano:true` tells the API the image is an equirectangular 360° panorama; it starts after the
image-to-panorama stage and skips suitability validation. Content moderation still applies.

`panorama_id` starts from an edited panorama returned by `POST /v1/panoramas/edit`. Generation
starts at pano2video, keeps lineage to the source world, and can be reused until the panorama
expires.

## Panorama editing

Edit the panorama behind an API-created world, iterate until satisfied, then create a new world:

```json
{
  "source": { "type": "world_id", "world_id": "<world-uuid>" },
  "prompt": "change the rug and chair to yellow",
  "images": [{ "type": "url", "image_url": "https://example.com/reference.jpg" }]
}
```

`source.type` is `request_id`, `world_id`, or `panorama_id`. `world_id` / `request_id` must
refer to an API-created completed world owned by the same user; it may have been created by a
different API key. `panorama_id` iterates on a previous edit artifact.

`images` is optional and accepts at most 3 references (`url`, `base64`, or `file_id`). There is
no aspect-ratio field; the API keeps the output as a panorama suitable for world generation.

Response:

```json
{
  "panorama_id": "pano_...",
  "status": "READY",
  "panorama_url": "https://api.spaitial.ai/v1/panoramas/pano_.../download",
  "parent_panorama_id": null,
  "consumed": false,
  "expires_at": "2026-06-20T12:00:00Z"
}
```

### Idempotency
Send `Idempotency-Key: <uuid>` to retry a POST safely. Same key + same body -> cached response;
same key + different body -> `409 IDEMPOTENCY_KEY_REUSED`. Not a retry token for FAILED jobs.

## Statuses

```
PENDING -> PROCESSING -> COMPLETED
                      \-> FAILED
                      \-> CANCELLED
```

`progress` (0–1) is coarse. `GET /…/status` returns `{ "status": "...", "progress": 0.4 }`.

## Result envelope — GET /v1/worlds/requests/:id

```json
{
  "request_id": "req_...",
  "status": "COMPLETED",
  "world": {
    "id": "<world-uuid>",
    "title": "Cozy reading nook",
    "splat_url": "https://api.spaitial.ai/v1/worlds/requests/req_.../splat",
    "splat_format": "spz",
    "thumbnail_url": "https://img.spaitial.ai/.../thumbnail.webp",
    "panorama_url": "https://api.spaitial.ai/v1/worlds/requests/req_.../panorama",
    "viewer_url": "https://app.spaitial.ai/worlds/<world-uuid>",
    "visibility": { "is_public": false, "is_listed": false }
  },
  "validation": { "passed": true, "issues": [] }
}
```

`splat_url` / `panorama_url` are **stable API endpoints**, not signed URLs. Each GET returns a
`302` to a fresh 5-minute signed URL — always download with `curl -L` and auth header.

## Files

```bash
# Upload
curl -sX POST https://api.spaitial.ai/v1/files \
  -H "Authorization: Bearer $SPAITIAL_API_KEY" -F "file=@photo.jpg"   # -> { "file_id": "file_..." }
# List
curl -s "https://api.spaitial.ai/v1/files?limit=20&offset=0" \
  -H "Authorization: Bearer $SPAITIAL_API_KEY"
```
Uploads: 24h TTL, owner-scoped, single-use. Status is `available` | `consumed` | `expired`.

## Exports (mesh)

```bash
# Start (or re-fetch) — type is mesh | mesh-simplified
curl -sX POST "https://api.spaitial.ai/v1/worlds/requests/$REQ_ID/exports/mesh" \
  -H "Authorization: Bearer $SPAITIAL_API_KEY"
# Poll
curl -s "https://api.spaitial.ai/v1/worlds/requests/$REQ_ID/exports/mesh" \
  -H "Authorization: Bearer $SPAITIAL_API_KEY"     # READY -> { "download_url": "...", ... }
# Download (follow the redirect)
curl -L -o mesh.ply "<download_url>" -H "Authorization: Bearer $SPAITIAL_API_KEY"
```
Requesting either mesh type starts one shared pipeline; both become READY together. `mesh` is a
full-resolution `.ply`; `mesh-simplified` is optimized for real-time.

## PATCH (update) & cancel

```bash
curl -X PATCH "https://api.spaitial.ai/v1/worlds/requests/$REQ_ID" \
  -H "Authorization: Bearer $SPAITIAL_API_KEY" -H "Content-Type: application/json" \
  -d '{ "title": "New title", "visibility": { "is_public": true, "is_listed": true } }'

curl -X POST "https://api.spaitial.ai/v1/worlds/requests/$REQ_ID/cancel" \
  -H "Authorization: Bearer $SPAITIAL_API_KEY"
```
PATCH needs at least one field; `is_listed:true` requires `is_public:true`. Returns
`409 RESOURCE_NOT_READY` if the world isn't COMPLETED. Cancel is best-effort.

## Error envelope

```json
{ "error": { "code": "VALIDATION_FAILED", "message": "...", "details": {} } }
```

| Code | HTTP | Meaning |
|------|------|---------|
| `UNAUTHORIZED` | 401 | Missing/invalid key |
| `FORBIDDEN` | 403 | Key lacks scope |
| `MODEL_NOT_FOUND` / `MODEL_FORBIDDEN` / `MODEL_UNAVAILABLE` | 400/403/503 | Model issue |
| `INVALID_INPUT` | 400 | Malformed body / unsupported input |
| `INSUFFICIENT_CREDITS` | 402 | Top up at developers portal |
| `MODERATION_REJECTED` | 403 | Content moderation blocked input |
| `VALIDATION_FAILED` | 422 | Suitability check rejected (`details.validation.issues`) |
| `FILE_NOT_FOUND` / `FILE_EXPIRED` | 404 | Bad/old/consumed `file_id` |
| `REQUEST_NOT_FOUND` | 404 | Unknown `request_id` |
| `RESOURCE_NOT_READY` | 409 | World not COMPLETED yet |
| `IDEMPOTENCY_KEY_REUSED` | 409 | Same key, different body |
| `RATE_LIMIT_EXCEEDED` | 429 | Back off; see `Retry-After` |
| `INTERNAL_ERROR` | 500 | Retry with backoff |

## Rate limits

`POST /v1/worlds` 10/min · `GET /…/status` 300/min · `GET /…/splat|/panorama` 120/min ·
`POST /v1/files` 20/min · everything else 120/min. Responses carry `X-RateLimit-*`.

## Webhooks (optional)

Set `webhook.url` on POST. Callback headers include `X-Spaitial-Event`
(`world.completed|failed|cancelled|export.completed|export.failed`), `X-Spaitial-Request-ID`,
`X-Spaitial-Delivery-ID` (dedupe on this), and `X-Spaitial-Signature: sha256=<hmac(body, secret)>`.
HTTPS only, 30s timeout, up to 5 retries. Verify with the webhook secret from the key settings page.
