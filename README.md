# SpAItial AI

Generate explorable 3D worlds (Gaussian splats) from a text prompt, an image, or a 360° panorama using the [SpAItial](https://spaitial.ai) API — then download them, export a mesh, manage your library, and preview them via a hosted viewer link.

As of **0.4.0**, the plugin talks to SpAItial through the **hosted SpAItial MCP server** (`https://mcp.spaitial.ai/mcp`), which supports both header-based keys (Claude Code/Desktop/Cursor) and OAuth login (claude.ai web/mobile). Your API key is entered once — as a connector header or on the OAuth consent screen — and never pasted into the chat.

## Install

In Claude Code (or the `/plugin` command in any Claude client):

```shell
/plugin marketplace add risenW/spaitial-ai-plugin
/plugin install spaitial-ai@spaitial-ai
```

Or in the Claude app: **Customize → Plugins → Add marketplace → from a repository**, enter `risenW/spaitial-ai-plugin`, then click **Install** on **spaitial-ai**.

After installing, open the **spaitial** connector's settings and paste your SpAItial API key (`spt_live_…` or `spt_test_…`, from the [developers portal](https://developers.spaitial.ai)) — it's stored by Claude and sent as a header, never in the chat. See [Setup](#setup) for details.

## What it does

This plugin turns a single image or a text description into a navigable 3D "world" — a Gaussian splat you can orbit, pan, and zoom. It handles the full SpAItial workflow for you: submitting the job, waiting for generation, fetching the result, and delivering the files.

## Components

| Skill | What it does |
|-------|--------------|
| **create-world** | Generate a new world from a text prompt, image URL, local image file, or 360° panorama. Returns a shareable hosted viewer link and thumbnail, and optionally downloads the splat. |
| **manage-worlds** | List your past generations, check job status, rename a world, change its visibility (private / public / listed), or cancel a running job. |
| **export-mesh** | Turn a finished world into a downloadable `.ply` mesh — full-resolution or simplified for real-time use. |

Each completed world includes a **hosted `viewer_url`** that opens in any browser — no download or local setup needed — so you can explore the result immediately (and share it once the world is public).

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

With header clients (Code/Desktop/Cursor) the hosted server is **stateless and holds no credentials** — it forwards your key to `api.spaitial.ai` on each request, and you're billed to your own SpAItial account. With claude.ai web/mobile it uses OAuth and stores your key **encrypted** (retrievable only via your issued token). Either way it exposes these tools: `list_models`, `create_world`, `get_world_status`, `get_world`, `list_world_requests`, `get_splat_download_url`, `get_panorama_download_url`, `update_world`, `cancel_world`, `start_mesh_export`, `get_mesh_export`.

Because the connection runs over native remote MCP (not a local process), it works in sandboxed environments like Cowork without any network workaround.

## Setup

### 1. Get a SpAItial API key

Create an account and an API key at the [SpAItial developers portal](https://developers.spaitial.ai).

### 2. Provide the key (once, not in chat)

How you hand over the key depends on the client, because the hosted server supports **two auth modes** on the same endpoint:

- **Claude Code / Claude Desktop / Cursor (header / BYOK):** the header `X-Spaitial-Api-Key` is wired to `${SPAITIAL_API_KEY}`. Set the key in the connector's config field (Desktop) or export it (Code):

  ```bash
  export SPAITIAL_API_KEY="spt_live_your_key_here"
  ```

  These clients send the key as a header and bypass OAuth entirely.

- **claude.ai web + mobile (OAuth login):** these surfaces only support OAuth connectors — they can't attach a custom header. So when you click **Connect**, you'll be redirected to a SpAItial consent screen; **paste your `spt_…` key there once**. It's validated, then stored encrypted by the connector and sent to `api.spaitial.ai` on each call — never shown in chat.

If a tool call returns `401 UNAUTHORIZED`, the key is missing or wrong — re-check the connector config (header clients) or reconnect and re-enter the key (web/mobile).

## Usage

Just describe what you want in plain language:

- "Make a 3D world of a cozy sunlit reading nook" → **create-world** (text prompt)
- **Attach a photo in chat** and say "turn this into a world" → **create-world**
- "Turn this photo into a world: https://… " → **create-world** (image URL)
- "Build a world from this 360 panorama" (+ attach or link the pano) → **create-world** (`is_pano`)
- "Show me my recent worlds" / "Make world req_… public" → **manage-worlds**
- "Export a mesh of req_…" → **export-mesh**

## Notes & limitations

- **Downloads return links, not bytes.** `get_splat_download_url`, `get_panorama_download_url`, and `get_mesh_export` return short-lived (~5 min) pre-signed URLs. On open network the skill downloads them for you; in a locked-down sandbox it hands you the link to download in your browser (the link is unauthenticated and safe to open). Re-run the tool if a link expires.
- **Images go in as base64.** The MCP server has no file-upload tool, so images — whether attached in chat or read from a local path — are sent to `create_world` inline as base64 (≤25 MB decoded). You don't need to host them anywhere. If a photo is larger than ~25 MB, it's downscaled first; only very large images that still won't fit need an HTTPS URL.

## Privacy & distribution

Worlds are private by default; visibility only changes when you explicitly ask. With header clients the hosted MCP server stores no key — it forwards yours per request over HTTPS, and your key lives in Claude's connector config, never in generated files or the chat. With claude.ai web/mobile, OAuth stores your key encrypted-at-rest (decryptable only with your access token) and forwards it per request. In both cases the key is never written into generated files or the chat.
