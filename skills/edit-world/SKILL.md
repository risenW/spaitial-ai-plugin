---
name: edit-world
description: >
  Use this skill when the user wants to edit or restyle an existing SpAItial world
  through the app-style panorama edit flow — e.g. "change the rug to yellow",
  "remove that object", "add this sofa", "make this room cyberpunk", "iterate on this
  world", or "create a new world from this edited pano". Edits the world panorama,
  supports up to 3 reference images, lets the user inspect each pano_ result, and
  finally creates a new world from the accepted panorama.
metadata:
  version: "0.5.0"
---

# Edit World

Edit an existing SpAItial world by editing its panorama first, iterating until the user is
satisfied, then creating a new world from the final `pano_...`.

## Tools

Use the hosted **spaitial** MCP connector:

`edit_panorama`, `get_panorama`, `get_panorama_download_url`, `create_world`,
`get_world_status`, `get_world`.

Read each tool's schema before calling it; the notes below describe the intended flow.

## Flow

1. Identify the source:
   - `request_id` (`req_...`) for an API-created world request.
   - world `id` (UUID) for an API-created world owned by the same user.
   - `panorama_id` (`pano_...`) when iterating on a prior edit.
2. Call `edit_panorama` with:
   - `source_type`: `request_id`, `world_id`, or `panorama_id`.
   - `source_id`: the matching id.
   - `prompt`: pass the user's edit instruction **verbatim**. Do not rewrite it unless the user asks.
   - optional references: up to 3 total via `reference_image_urls`, `reference_image_base64`,
     or `reference_file_ids`.
3. Show the returned JSON, especially `panorama_id`, `parent_panorama_id`, and `panorama_url`.
4. Call `get_panorama_download_url` and, when the environment allows it, download/save the image
   so the user can inspect it.
5. Stop and wait for user confirmation or the next edit instruction.
6. If the user wants another edit, call `edit_panorama` again with
   `source_type: "panorama_id"` and the previous `pano_...`.
7. When the user says the edit is good, call `create_world` with `panorama_id`.
8. Poll with `get_world_status`; fetch the completed world with `get_world`.

## Important behavior

- There is **no aspect-ratio option**. The API keeps the edited result as a panorama suitable
  for world generation.
- Reference images guide edits like "add this car" or "merge this style"; they do not replace
  the source panorama unless the prompt asks for that.
- `world_id` sources are limited to API-created worlds owned by the same user, but they are not
  tied to a single API key.
- Edited panoramas expire after 24 hours. Create the final world before expiry.
- Creating a world from a `pano_...` marks it consumed for visibility, but the pano can still be
  reused until expiry.

## Minimal examples

Prompt-only edit:

```json
{
  "source_type": "world_id",
  "source_id": "<world-uuid>",
  "prompt": "change the color od the rug and chair to yellow and add a sound system next to the window"
}
```

Edit with reference images:

```json
{
  "source_type": "panorama_id",
  "source_id": "pano_...",
  "prompt": "Add this sofa in the room",
  "reference_image_base64": ["data:image/png;base64,..."]
}
```

Create the final world:

```json
{
  "panorama_id": "pano_...",
  "title": "Edited room"
}
```
