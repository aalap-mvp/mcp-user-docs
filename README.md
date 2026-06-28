# Connect to Taan

Taan is an AI music composer for video. Your agent hands over a creative brief; Taan returns finished music it can refine, download as mp3/wav, or mix back over the video. Taan speaks the [Model Context Protocol](https://modelcontextprotocol.io) — any MCP-aware agent (Claude, GPT, Gemini, open-source) can connect with one config block.

This page covers everything an agent developer needs to integrate Taan: endpoint, auth, tool surface, schema discovery, and operational limits.

---

## Quick start

You need two things:

1. **Endpoint:** `https://mcp.taan.ai/mcp` (Streamable HTTP transport)
2. **Bearer key:** `Authorization: Bearer sk_taan_<your_key>`

That's the entire connection contract. Everything else — the tool list, parameter schemas, validation rules — your agent discovers from the server itself when it connects.

### Get a key

Request one at [taan.ai/api](https://taan.ai/api) (or email `team@taan.ai`). Keys are scoped to a single workspace; one key per project is typical.

---

## Drop-in configuration

### Claude Desktop, Claude Code, Cursor, Cline, Continue, Zed

These all read MCP servers from a JSON config (`~/.claude.json`, Cursor's settings, etc). Add a `taan` entry:

```json
{
  "mcpServers": {
    "taan": {
      "url": "https://mcp.taan.ai/mcp",
      "headers": {
        "Authorization": "Bearer sk_taan_YOUR_KEY"
      }
    }
  }
}
```

Restart the client. The agent will discover Taan's tools automatically — no SDK to install.

### Python (`mcp` SDK)

```python
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client

async with streamablehttp_client(
    "https://mcp.taan.ai/mcp",
    headers={"Authorization": "Bearer sk_taan_YOUR_KEY"},
) as (read, write, _):
    async with ClientSession(read, write) as session:
        await session.initialize()
        tools = await session.list_tools()
        # then call session.call_tool("compose_quick", {"prompt": "..."}) etc.
```

### TypeScript (`@modelcontextprotocol/sdk`)

```ts
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StreamableHTTPClientTransport } from "@modelcontextprotocol/sdk/client/streamableHttp.js";

const transport = new StreamableHTTPClientTransport(
  new URL("https://mcp.taan.ai/mcp"),
  {
    requestInit: {
      headers: { Authorization: `Bearer ${process.env.TAAN_API_KEY}` },
    },
  },
);
const client = new Client({ name: "my-agent", version: "1.0.0" });
await client.connect(transport);
const { tools } = await client.listTools();
```

### OpenAI Agents SDK, LangGraph, custom loops

If your framework supports MCP servers natively, point it at `https://mcp.taan.ai/mcp` with the same bearer header. If it doesn't yet, use the Python or TypeScript MCP SDKs above and wrap the tool calls in your framework's tool interface.

---

## Tools

Once connected, your agent sees nine tools. Full input schemas live on the server — what's here is the orientation.

| Tool | When to call it |
|---|---|
| `create_project` | Brief-driven music. You construct the CreativeBrief; Aalap runs Loop 2 (music). Use when the agent has rich per-section creative direction. |
| `create_project_from_video` | Video-driven music. Hand over an uploaded mp4; Aalap runs Loop 1 (video → brief) then Loop 2 (brief → music). Use when the agent has a finished video and wants Aalap to read it. **Max 2 minutes.** |
| `request_video_upload` | Step 1 of the video flow: get a 15-minute presigned PUT URL into your account's upload prefix. PUT the bytes, then call `create_project_from_video` with the returned `gcs_uri`. |
| `get_project` | Fetch project state. Use to poll after either `create_project*` call with `wait=false`, or to inspect any past project. |
| `list_projects` | All projects on your key (most recent first). |
| `refine_music` | Iterate on an existing variant with free-form feedback ("darker strings in the second section"). Produces a new variant. Works for projects from either entry path. |
| `download` | Get a 15-minute presigned URL for a variant as mp3 or wav. |
| `compose_quick` | One-shot text-to-music. No project, no refinement — just a description in, mp3 out. Use when you don't need iteration. |
| `generate_sfx` | Single non-musical sound effect (impact, ambience, sting, riser). Routed through ElevenLabs; Taan handles credentials. |

### Picking the entry point

- **You have a finished or near-finished video** → `request_video_upload` then `create_project_from_video`. Aalap watches the video and authors the brief itself. Lowest agent effort, highest fidelity to what's on screen.
- **You have rich creative direction but no video (or the video isn't the source of truth)** → `create_project` with a hand-authored `CreativeBrief`. More work for the agent, but every section, transition, and emotion is under your control.
- **You just need a piece of music from a text prompt** → `compose_quick`. No project, no refinement.

### From-video flow in detail

```text
1. request_video_upload(filename="ad.mp4", content_type="video/mp4", size_bytes=12345678)
   → { upload_url, gcs_uri, content_type, expires_in_seconds }

2. PUT the video bytes to upload_url (use the same Content-Type as above).
   curl -X PUT --upload-file ad.mp4 -H "Content-Type: video/mp4" "$upload_url"

3. create_project_from_video(gcs_uri=<from step 1>, user_prompt="lean cinematic, no percussion in first 10s")
   → { job_id, status: "Analyzing", duration_seconds, message }

4. Poll get_project(job_id) until status == "Completed".
   Status progresses: Analyzing → Composing → Generating → Completed.

5. download(job_id, format="mp3") for the rendered variant.
```

**Constraints (enforced server-side):**

- **Duration**: 10s ≤ video ≤ 120s (2 minutes). Out-of-range uploads come back as `video_too_short` or `video_too_long` with the exact duration in the error message.
- **Size**: ≤ 250 MB. Caught at `request_video_upload` time.
- **Format**: `.mp4`, `.mov`, `.m4v`, or `.webm`. Anything ffprobe can't read returns `unsupported_format` with the underlying error attached.
- **The upload URL is one-time and scoped to your API key.** A `gcs_uri` from a different account is rejected at `create_project_from_video` time. Treat the gcs_uri as opaque — only the issuing account can consume it.
- **The source video is not persisted long-term.** Once Loop 1 has produced the brief, Aalap no longer needs the video; uploads auto-delete after 7 days. Persisted on the project: the derived CreativeBrief, music identity / timeline / composition plan, and rendered variants — all available to `refine_music`.

### `compose_quick` vs `create_project`

- **`compose_quick`** — a piece of music from a text prompt. No variants, no refinement, no project lifecycle. Right when your agent just needs a stinger or background track and won't iterate.
- **`create_project`** — a brief describing a video. Returns variants, supports refinement, downloads, mixing back. Right when the agent has produced a video and needs scored music aligned to its sections.

Wrong tool for the job is the most common integration mistake. Pick deliberately.

---

## The creative brief

`create_project` takes a `CreativeBrief` — a structured description of the video the music will score. The full JSON Schema is published two ways:

1. **As the `create_project` tool's `inputSchema`** — your agent reads it on `tools/list`. This is the canonical source.
2. **As a resource at `aalap://brief-schema`** — `resources/read` returns the same schema for reference.

### What the brief contains

- **Top-level metadata:** `type`, `duration`, `setting`, `time_era`, `target_audience`, `overall_emotion`, `overall_energy`, `advertising_objective`, `global_geography`, `story_overview`.
- **`sections`:** an ordered list of timed beats. Each has a `timestamp_range`, a `narrative` (intent + action), and three free-form descriptors — `emotion` (what the viewer feels), `mood` (the atmosphere), `scene_energy` (the pacing). Optional `important_timestamps` mark voiceover/sound/action moments inside the section that the music should align to.
- **`transitions`:** how consecutive sections are bridged. Name the technique (cut, J-cut, match cut, crossfade, dissolve) and what carries across.

### Constraints (enforced strictly)

- **Total duration:** 10s ≤ `duration` ≤ 240s (4 minutes).
- **Each section:** 3s ≤ duration ≤ 120s.
- **Section ids are unique;** transitions reference them.
- **Transitions tile sections:** exactly `len(sections) - 1` entries, in order.
- **No unknown fields.** The schema is `additionalProperties: false` — extra keys are rejected with a validation error naming the offending path.

If a brief fails validation, the error response tells you exactly which field and why. Construct briefs from the published schema, not from memory.

---

## Authentication

Every request to `mcp.taan.ai/mcp` must carry:

```
Authorization: Bearer sk_taan_<your_key>
```

- Keys never expire on their own; rotate via the dashboard if exposed.
- Each key is bound to one workspace; usage and billing aggregate per key.
- The MCP server itself stores no credentials — it forwards your bearer header to the Taan backend on every request. If the header is missing or invalid you'll get a `401`.

Treat the key like a database password: don't commit it, don't ship it in client-side bundles, don't paste it into screenshots.

---

## Smoke test (60 seconds)

Confirm your key works before wiring it into an agent:

```bash
KEY=sk_taan_YOUR_KEY

# 1. List tools — confirms auth + endpoint
curl -s -X POST https://mcp.taan.ai/mcp \
  -H "Authorization: Bearer $KEY" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'

# 2. Generate a quick clip — confirms the full pipeline
curl -s -X POST https://mcp.taan.ai/mcp \
  -H "Authorization: Bearer $KEY" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"compose_quick","arguments":{"prompt":"30-second warm acoustic guitar loop"}}}'
```

Both return SSE-framed JSON-RPC. The first lists nine tools; the second writes an mp3 to a temp path on the calling environment.

---

## Operational details

### Long-running calls

`create_project` and `refine_music` block until the music is ready — typically **30 to 90 seconds** for the default `wait=true`. `create_project_from_video` runs Loop 1 *and* Loop 2 (typically **90 to 180 seconds** end-to-end), so it defaults to `wait=false` — the call returns a `job_id` and you poll `get_project` until `status == "Completed"`. Configure your HTTP client and agent framework to tolerate tool calls up to **5 minutes** (Taan's server timeout) if you want to override the default and wait inline.

### Credits and billing

- Each plan grants a monthly credit budget (`Creator`: 15,000; `Agency`: 36,000).
- Project generations and refinements deduct credits at fixed rates published on the pricing page.
- Failed jobs (Taan-side errors) don't consume credits. User-cancelled jobs do.
- Current balance is visible on the dashboard and via `list_projects` metadata.

If you hit the credit ceiling mid-call, the tool returns an `upgrade_required` error. The error is machine-readable — an agent can route it to a "please upgrade" UI rather than crashing.

### Download URL expiry

`download` returns a presigned GCS URL valid for **~15 minutes**. Fetch the bytes promptly. If the URL expires before download completes, call `download` again — the underlying mp3/wav doesn't change, so a refreshed URL points at the same file.

### Rate limits

Per-key concurrency is capped at **10 in-flight tool calls** by default. Beyond that, requests queue server-side. Heavy users can request a raise via support.

### Errors you may see

| Error code | Meaning | What to do |
|---|---|---|
| `invalid_input` | The brief or arguments failed validation. The message names the offending field. | Fix the input. |
| `upgrade_required` | Free tier used or paid plan expired; the requested feature (download, advanced sfx) is gated. | Direct the user to upgrade. |
| `quota_exceeded` | Out of credits for the billing period. | Top up or wait for the cycle to roll over. |
| `not_found` | `job_id` doesn't exist on this key. | Check the id — projects are scoped per key. |
| `service_unavailable` | Transient upstream issue (model provider, storage). | Retry with exponential backoff (the SDK does this for you). |
| `internal_error` | Bug or unexpected state. Report it. | Email `support@taan.ai` with the `request_id` from the response. |

Every response carries an `X-Request-Id` header — include it in any bug report.

---

## Versioning and changelog

- Taan follows additive-only schema evolution. Adding optional fields, new tools, or new enum values is non-breaking. Removing or renaming requires **30 days' notice** via the changelog and an email to active keys.
- Subscribe to the [changelog feed](https://taan.ai/changelog) for new tools and schema additions.
- Major version bumps would change the URL path (e.g. `/mcp/v2`), so v1 callers are never silently broken.

---

## Support

- **Email:** `team@taan.ai` (response within 1 business day)
- **Status page:** [status.taan.ai](https://status.taan.ai) — Cloud Run uptime, scheduled maintenance
- **Issues / feature requests:** [github.com/taan-ai/feedback](https://github.com/taan-ai/feedback)

When reporting an issue, include:
- The `X-Request-Id` from the failed response.
- The tool name and (redacted) arguments.
- Your client (Claude Desktop, Cursor, custom Python, etc.) and SDK version.
