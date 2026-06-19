---
name: manage-worlds
description: >
  This skill should be used when the user wants to view or manage worlds they already
  generated with SpAItial — e.g. "list my worlds", "show my recent generations", "check
  the status of req_…", "is my world done yet?", "rename this world", "make world req_…
  public / private / listed", "share my world", or "cancel that running job". Lists jobs,
  polls status, downloads existing splats/panoramas, updates title and visibility, and
  cancels in-flight jobs via the SpAItial MCP server. Use the companion create-world skill
  to generate new worlds and export-mesh to produce mesh files.
metadata:
  version: "0.4.0"
---

# Manage Worlds

List, inspect, update, share, and cancel SpAItial worlds via the **spaitial** MCP connector
bundled with this plugin. Field meanings, statuses, and error codes are in
`../create-world/references/api-reference.md`.

## Connector / key

All calls go through the `spaitial` MCP tools, which carry the user's key via the
`X-Spaitial-Api-Key` header wired to `${SPAITIAL_API_KEY}` — entered once in Claude's
connector config, **never in chat**. If the tools are missing, ask the user to enable the
connector and set their key in its config; a `401 UNAUTHORIZED` means the key is missing/wrong.

## List recent worlds

Call `list_world_requests` (supports `limit` / `offset`). Present a compact table:
`request_id`, `status`, `title`, `created_at`, and `viewer_url` when present. Paginate with
`offset` if the user wants more.

## Check status of one job

- Quick check: `get_world_status` with `wait_seconds: 0` (instant; returns `status` + coarse `progress`).
- Full details (URLs, validation issues): `get_world`.

If the user is waiting on a running job, call `get_world_status` with `wait_seconds: 90` and
repeat until `COMPLETED`, `FAILED`, or `CANCELLED`. The server holds each call up to 90s and
returns early when terminal, so **don't busy-poll** — generation takes ~6–10 min (standard) to
~1 hour (HQ).

## Download an existing splat or panorama

Call `get_splat_download_url` or `get_panorama_download_url` (with the `request_id`). Each
returns a **short-lived (~5 min) pre-signed URL** that needs no key. On open network, download
it (`curl -L -o world.spz "<signed_url>"`, name the splat to match `splat_format`); in a
sandboxed environment, just hand the user the link, re-minting it if it expired. To preview a
`.sog` locally, set up the bundled viewer as described in **create-world** (`assets/viewer.html`).

## Rename / change visibility

Call `update_world` (with `request_id` and the fields to change). Only works once the world is
`COMPLETED` (otherwise `RESOURCE_NOT_READY`); at least one field is required; `is_listed: true`
requires `is_public: true`. Confirm with the user before making a world **public** or
**listed** (listing makes it eligible for the public gallery), then report the new visibility.

## Cancel a running job

Call `cancel_world` (with `request_id`). Best-effort: the API reports `CANCELLED` and in-flight
processing stops within seconds. Cancelling does not refund a fully completed job.

## Notes
- `request_id` (the operation) and the world `id` (the artifact) are different IDs — operate on
  `request_id` for all of these tools.
- Keys live in connector config only — never echo a key back or write it into a saved file.
