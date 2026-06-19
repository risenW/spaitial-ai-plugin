---
name: export-mesh
description: >
  This skill should be used when the user wants a polygon mesh from a finished SpAItial
  world — e.g. "export a mesh of req_…", "give me the .ply mesh", "I need a mesh for
  Blender / Unreal / Unity", "export a simplified / low-poly mesh", or "convert my world
  to a 3D model". Starts a mesh export through the SpAItial MCP server, waits for it to
  finish, and downloads the .ply file (full-resolution or simplified). Use create-world to
  generate the world first and manage-worlds to find a world's request_id.
metadata:
  version: "0.3.0"
---

# Export Mesh

Produce a downloadable `.ply` mesh from a completed SpAItial world via the **spaitial** MCP
connector. Background: `../create-world/references/api-reference.md`.

## Prerequisites

- The world must be `COMPLETED`. If you only have a prompt/image, run **create-world** first;
  to find an existing `request_id`, use **manage-worlds**.
- The `spaitial` connector must be enabled with the user's key in its config (carried via the
  `X-Spaitial-Api-Key` header, `${SPAITIAL_API_KEY}`) — never via chat. A `401` means the key
  is missing/wrong; point the user to the connector config.

## Mesh types

| type | Result |
|------|--------|
| `mesh` | Full-resolution reconstructed mesh (`.ply`) |
| `mesh-simplified` | Simplified mesh, optimized for real-time use |

Requesting either type starts the **same shared pipeline** — both become READY together. If the
user is unsure, ask; default to `mesh` for quality, `mesh-simplified` for game engines / web.

## Step 1 — Start the export

Call `start_mesh_export` with the `request_id` and the chosen `type`. A `RESOURCE_NOT_READY`
error means the world isn't finished yet — wait and retry.

## Step 2 — Poll until READY

Poll `get_mesh_export` (with `request_id` + `type`) every ~15s until `status` is `READY` or
`FAILED`. On `FAILED`, report the error and offer to retry.

## Step 3 — Download the .ply

When `READY`, `get_mesh_export` returns a **short-lived pre-signed `download_url`** (needs no
key):

- Open network: `curl -L -o "$(pwd)/mesh-<type>.ply" "<download_url>"`, then present the file.
  Mention it opens in Blender, MeshLab, Unreal, Unity, and most 3D tools.
- Sandboxed environment: hand the user the `download_url` as a direct link; if it expired, call
  `get_mesh_export` again to mint a fresh one.

## Notes
- Keys live in connector config only — never echo a key back or write it into a saved file.
