---
name: create-world
description: >
  This skill should be used when the user wants to generate a 3D world, scene, or
  Gaussian splat with SpAItial — e.g. "make a 3D world of…", "turn this photo into a
  3D scene", "generate a splat from this image", "create a world from a text prompt",
  or "build a world from this 360 panorama". Handles text prompts, image URLs, local
  image files, and equirectangular panoramas; submits the job through the SpAItial MCP
  server, waits for it to finish, returns the viewer link and thumbnail, downloads the
  splat, and can open a local PlayCanvas viewer. Triggers on mentions of spaitial,
  world generation, .spz/.sog splat files, or 3D scene generation from an image.
metadata:
  version: "0.4.0"
---

# Create World

Generate a 3D Gaussian-splat "world" from a text prompt, image, or 360° panorama using the
hosted **SpAItial MCP server** (the `spaitial` connector bundled with this plugin), then
deliver the result. Background on field meanings, statuses, and error codes lives in
`references/api-reference.md`.

## How this works

This plugin talks to SpAItial through MCP tools, not raw HTTP. When the `spaitial` connector
is enabled you'll have these tools (names as exposed by the server):

`list_models`, `create_world`, `get_world_status`, `get_world`, `list_world_requests`,
`get_splat_download_url`, `get_panorama_download_url`, `update_world`, `cancel_world`,
`start_mesh_export`, `get_mesh_export`.

Always read each tool's own input schema (the MCP client shows it) for exact parameter names
rather than guessing — the notes below describe intent.

## Step 1 — Make sure the connector is set up

The server is **bring-your-own-key**: every call must carry the user's SpAItial API key, sent
as the `X-Spaitial-Api-Key` header. The plugin wires that header to `${SPAITIAL_API_KEY}`, so
the key is entered **once in Claude's connector/plugin configuration (or as an environment
variable), never pasted into the chat.**

- If the `spaitial` tools aren't available, ask the user to enable the plugin's **spaitial**
  connector and provide their key in its config field (format `spt_live_…` / `spt_test_…`,
  from https://developers.spaitial.ai).
- If a tool call returns `401 UNAUTHORIZED`, the key is missing or wrong — point them to the
  connector config. Do **not** ask them to paste the key into the conversation.

## Step 2 — Determine the input

Ask only for what's missing. `create_world` accepts these input shapes:

| User has… | Pass to `create_world` | Notes |
|-----------|------------------------|-------|
| A description in words | a text **prompt** | charged for image + world generation |
| An HTTPS image link | an **image_url** | HTTPS only, ≤25 MB, JPEG/PNG/WebP/GIF |
| A 360° panorama | the URL/base64 **+ `is_pano: true`** | skips the image-to-pano + suitability stages |
| An image **attached in the chat** | read the attached file → **base64** | most common; see note below |
| A file on their computer (path) | read it → **base64** | see note below |

> **Images from the chat or disk:** the MCP server exposes **no file-upload tool**, so there's
> no `file_id` path — send the image **inline as base64** via `image_base64` (≤25 MB decoded;
> JPEG/PNG/WebP/GIF).
>
> - **Attached in the conversation:** when the user drops an image into the chat, it's saved to
>   disk (in Cowork, under the session's uploads folder). Locate that file, base64-encode its
>   bytes, and pass the string to `create_world`. Don't ask the user to re-host or paste a URL —
>   the attachment is enough.
> - **Local path:** same flow for any file path the user gives.
> - **Too big?** If a photo exceeds ~25 MB, downscale/recompress it first (e.g. `sips`,
>   `ffmpeg`, or Pillow) and send the smaller version. Only fall back to asking for an HTTPS URL
>   if it still won't fit.

Also collect (or default sensibly):
- **title** — short caption (≤200 chars); default something derived from the prompt/file.
- **output_format** — `sog` if the user wants the bundled PlayCanvas viewer / PlayCanvas
  tooling (adds ~25s); otherwise the default `spz`.
- **visibility** — keep **private** unless the user explicitly asks for public/listed.

## Step 3 — Create the world

Call `create_world` with the chosen input and options. It returns a **`request_id`**. Tell the
user generation typically takes ~5–10 minutes. (Text-prompt worlds are billed for both image
and world generation.)

If the call errors, read the error envelope and explain it (e.g. `INSUFFICIENT_CREDITS`,
`MODERATION_REJECTED`, `INVALID_INPUT`) — see the error table in `references/api-reference.md`.

## Step 4 — Wait for generation

Generation is **takes a few minutes**: standard models take ~6–10 minutes, HQ models up to ~60 minutes. **Do not
busy-poll.** Call `get_world_status` with `wait_seconds: 90` — the server holds the request and
returns early the moment the status is terminal, so each call is ~90s of waiting in a single
tool round-trip. Just repeat the call until the status is `COMPLETED`, `FAILED`, or `CANCELLED`.

- One `get_world_status` call ≈ 90s. A standard world is ~5–7 calls; an HQ world is dozens —
  that's expected, and far cheaper than polling every few seconds.
- Use `wait_seconds: 0` (or omit) only for a quick one-off status check.
- Don't loop forever — if it's still running after a long while (or the user steps away), tell
  them it's still processing and they can re-check later with the **manage-worlds** skill using
  the `request_id`. On `FAILED`/`CANCELLED`, call `get_world` for the reason and offer to retry
  (a retry is a fresh `create_world`).

## Step 5 — Fetch the result

Call `get_world` with the `request_id` and surface from the world object:
- **`viewer_url`** — hosted SpAItial viewer (works for the owner, or if the world is public).
- **`thumbnail_url`** — preview image.
- `splat_format`, `request_id`, and the world `id`.

## Step 6 — Deliver the files

### Download the splat
Call `get_splat_download_url` (with the `request_id`) — it returns a **short-lived (~5 min)
pre-signed URL**. That URL needs no API key, so:

- **If this environment has open network** (e.g. a normal terminal): download it and save into
  the working/outputs folder, naming it to match `splat_format`:
  ```bash
  curl -L -o "$(pwd)/world.spz" "<signed_url>"
  ```
- **If downloads are blocked here** (sandboxed environments): just give the user the signed URL
  as a direct download link. If it has expired, call `get_splat_download_url` again to mint a
  fresh one. (The link is unauthenticated, so it's safe to hand over.)

### Open the local PlayCanvas viewer (when requested)
The plugin bundles a self-contained viewer at `assets/viewer.html` (sibling to this file):

1. Generate with `output_format: "sog"` for best compatibility and download it as `world.sog`.
2. Read `assets/viewer.html` and write a copy into the same folder as `world.sog`.
3. Present `viewer.html`; tell the user to open it in a browser and **drag `world.sog` onto the
   page** (loads local splats via the file picker / drag-and-drop, avoiding browser
   local-file restrictions). Supports `.sog`, `.compressed.ply`, and `.ply`.

Don't try to serve the file over a localhost server from a sandbox — drag-and-drop is reliable.

### Export a mesh / manage the world
For `.ply` mesh export, hand off to **export-mesh**; for list/rename/visibility/cancel, hand
off to **manage-worlds**. Both reuse the same `request_id`.

## Notes
- The `request_id` (the operation) and the world `id` (the artifact) are different IDs — use
  `request_id` for status, get, downloads, exports, update, and cancel.
- `list_models` lists available generation models if the user wants a non-default model.
- Keys live in connector config only — never echo a key back or write it into a saved file.
