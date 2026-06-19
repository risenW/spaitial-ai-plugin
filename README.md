# SpAItial AI

Generate explorable 3D worlds (Gaussian splats) from a text prompt, an image, or a 360° panorama using the [SpAItial](https://spaitial.ai) API — then download them, export a mesh, manage your library, and preview them in a built-in PlayCanvas viewer.

As of **0.3.0**, the plugin talks to SpAItial through the **hosted SpAItial MCP server** (`https://mcp.spaitial.ai/mcp`) instead of making raw HTTP calls. Your API key is entered once in Claude's connector configuration and sent as a request header — never pasted into the chat.

## What it does

This plugin turns a single image or a text description into a navigable 3D "world" — a Gaussian splat you can orbit, pan, and zoom. It handles the full SpAItial workflow for you: submitting the job, waiting for generation, fetching the result, and delivering the files.

## Components

| Skill | What it does |
|-------|--------------|
| **create-world** | Generate a new world from a text prompt, image URL, local image file, or 360° panorama. Returns a shareable viewer link and thumbnail, optionally downloads the splat, and can open a local PlayCanvas viewer. |
| **manage-worlds** | List your past generations, check job status, rename a world, change its visibility (private / public / listed), or cancel a running job. |
| **export-mesh** | Turn a finished world into a downloadable `.ply` mesh — full-resolution or simplified for real-time use. |

The plugin also bundles a self-contained **PlayCanvas viewer** (`viewer.html`) that runs in any modern browser with no install, so you can inspect a generated world locally.

## How it connects (MCP, bring-your-own-key)

The plugin declares one MCP server in `.mcp.json`:

```jsonc
{
  "mcpServers": {
    "spaitial": {
      "type": "http",
      "url": "https://mcp.spaitial.ai/mcp",
      "headers": { "X-Spaitial-Api-Key": "${SPAITIAL_API_KEY}" }
    }
  }
}
```

The hosted server is **stateless and holds no credentials** — it forwards your key to `api.spaitial.ai` on each request, and you're billed to your own SpAItial account. It exposes these tools: `list_models`, `create_world`, `get_world_status`, `get_world`, `list_world_requests`, `get_splat_download_url`, `get_panorama_download_url`, `update_world`, `cancel_world`, `start_mesh_export`, `get_mesh_export`.

Because the connection runs over native remote MCP (not a local process), it works in sandboxed environments like Cowork without any network workaround.

## Setup

### 1. Get a SpAItial API key

Create an account and an API key at the [SpAItial developers portal](https://developers.spaitial.ai).

### 2. Provide the key (once, not in chat)

Enable the **spaitial** connector that ships with this plugin and enter your key in its configuration. Concretely, the header `X-Spaitial-Api-Key` is wired to `${SPAITIAL_API_KEY}`:

- **Claude Desktop / Cowork:** set the key in the connector's config field. Treat it as a secret — it's stored by Claude, not written into the chat or any generated file.
- **Claude Code / terminal:** export it in your environment so `${SPAITIAL_API_KEY}` resolves:

  ```bash
  export SPAITIAL_API_KEY="spt_live_your_key_here"
  ```

If a tool call returns `401 UNAUTHORIZED`, the key is missing or wrong — re-check the connector config.

## Usage

Just describe what you want in plain language:

- "Make a 3D world of a cozy sunlit reading nook" → **create-world** (text prompt)
- **Attach a photo in chat** and say "turn this into a world" → **create-world** (the attached image is sent inline as base64)
- "Turn this photo into a world: https://… " → **create-world** (image URL)
- "Build a world from this 360 panorama" (+ attach or link the pano) → **create-world** (`is_pano`)
- "Show me my recent worlds" / "Make world req_… public" → **manage-worlds**
- "Export a mesh of req_…" → **export-mesh**

## Notes & limitations

- **Downloads return links, not bytes.** `get_splat_download_url`, `get_panorama_download_url`, and `get_mesh_export` return short-lived (~5 min) pre-signed URLs. On open network the skill downloads them for you; in a locked-down sandbox it hands you the link to download in your browser (the link is unauthenticated and safe to open). Re-run the tool if a link expires.
- **Images go in as base64.** The MCP server has no file-upload tool, so images — whether attached in chat or read from a local path — are sent to `create_world` inline as base64 (≤25 MB decoded). You don't need to host them anywhere. If a photo is larger than ~25 MB, it's downscaled first; only very large images that still won't fit need an HTTPS URL.

## Privacy & distribution

Worlds are private by default; visibility only changes when you explicitly ask. The hosted MCP server stores no key — it forwards yours per request over HTTPS. Your key lives in Claude's connector config, never in generated files or the chat. If you'd prefer users *log in* rather than paste a key, the server can be extended with OAuth (see the server's README).
