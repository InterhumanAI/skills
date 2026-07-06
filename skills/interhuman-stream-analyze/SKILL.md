  ---
name: interhuman-stream-analyze
description: Connect to Interhuman v1 WebSocket stream analyze for real-time video segment analysis. Use for live streams, chunked video, WebSocket, or /v1/stream/analyze. Returns verbatim JSON event envelopes without modification.
---

# Interhuman Stream Analyze (v1)

Wrapper for the Interhuman API v1 WebSocket streaming endpoint that analyzes video segments in real time and returns social-intelligence events.

Use **v1 only** (`/v1/stream/analyze`). Do not use the legacy v0 endpoint (`/v0/stream/analyze`).

## When to Use

Use this skill when:
- Analyzing live or ongoing video feeds over WebSocket
- Sending video in time-bounded segments (chunks) as they become available
- You need incremental events (`signal.detected`, `engagement.updated`, etc.) as analysis progresses

Do NOT use this skill for:
- Analyzing a single completed video file in one request — use **interhuman-post-processing** (`POST /v1/upload/analyze`) instead

## Required Inputs

1. **API Key**: Bearer credential or `Sec-WebSocket-Protocol` value (see Authentication)
2. **Video segments**: Binary chunks sent over the WebSocket
   - Duration: at least **3 seconds** per segment
   - Size: maximum **32 MB** per segment
   - Formats: mp4, avi, mov, mkv, mpeg-ts, mpeg-2-ts, webm

   **Content requirements** (both video and audio must be meaningful):

   - Include real visual content; a valid segment with no meaningful video (e.g. a black or blank screen) is discouraged.
   - Include real audio; a valid segment with no meaningful audio (e.g. muted or silent track) is discouraged.
   - The API analyzes observable social cues from picture and sound—placeholder or empty media reduces result quality.

## Authentication

Prefer the API key in the `Authorization` header when your WebSocket client supports it:

- `Authorization: Bearer <api_key>`

If the client cannot set `Authorization` on the WebSocket handshake, pass the API key via subprotocol:

- `Sec-WebSocket-Protocol: <api_key>`

Optional correlation for your own logs (not echoed in responses):

- `X-Client-Request-Id: <your-request-id>`

The server assigns a connection `correlation_id` returned in every event envelope and in the `X-Correlation-ID` HTTP response header on upgrade.

## Connection

- **URL**: `wss://api.interhuman.ai/v1/stream/analyze`
- **Protocol**: WebSocket

## Client Workflow

1. Open a WebSocket to `wss://api.interhuman.ai/v1/stream/analyze` with authentication headers.
2. Optionally send a **session config** JSON message (can be sent or updated at any time).
3. Send **binary** video segment frames (one segment per message).
4. For each incoming server message, parse JSON and relay it verbatim (see Output Rules).
5. Close the WebSocket when the session ends.

Times in v1 server events (`start`, `end`) use **absolute session-cumulative** seconds across all segments sent in the connection.

## Session Config

Send a JSON text frame with optional fields:

```json
{
  "include": ["conversation_quality_overall", "conversation_quality_timeline"]
}
```

| Field | Values | Purpose |
|-------|--------|---------|
| `include` | `conversation_quality_overall`, `conversation_quality_timeline` | Opt in to `conversation_quality.updated` event sections |

- `conversation_quality_overall`: cumulative CQI across every window emitted so far in the session
- `conversation_quality_timeline`: single per-window CQI entry for the current analysis window

Unknown `include` values are ignored. Omit fields you do not need.

## Server Messages

Every server message is a JSON envelope with `type`, `timestamp`, `correlation_id`, and `data`. Narrow on `type`:

| `type` | Summary |
|--------|---------|
| `signal.detected` | Social signal in the current window |
| `engagement.updated` | Engagement state for the current window |
| `conversation_quality.updated` | CQI sections per session `include` flags |
| `error` | Processing failure |

For full field definitions and examples, see [reference.md](reference.md).

## Signal Types

`signal.detected` uses `data.signal_type`. Supported values:

`agreement`, `confidence`, `confusion`, `disagreement`, `disengagement`, `engagement`, `frustration`, `hesitation`, `interest`, `skepticism`, `stress`, `uncertainty`

`data.probability` is one of: `high`, `medium`, `low`.

## Example: Python

```python
import asyncio
import json

import websockets

API_KEY = "YOUR_API_KEY"
VIDEO_PATH = "/path/to/segment.mp4"
WS_URL = "wss://api.interhuman.ai/v1/stream/analyze"

async def main():
    headers = {"Authorization": f"Bearer {API_KEY}"}
    async with websockets.connect(WS_URL, additional_headers=headers) as ws:
        await ws.send(
            json.dumps({"include": ["conversation_quality_overall"]})
        )

        with open(VIDEO_PATH, "rb") as f:
            await ws.send(f.read())

        async for message in ws:
            event = json.loads(message)
            print(json.dumps(event))  # verbatim envelope

asyncio.run(main())
```

## Example: JavaScript (Node.js)

```javascript
import fs from "fs";
import WebSocket from "ws";

const apiKey = process.env.INTERHUMAN_API_KEY;
const videoPath = "path/to/segment.mp4";

const ws = new WebSocket("wss://api.interhuman.ai/v1/stream/analyze", {
  headers: { Authorization: `Bearer ${apiKey}` },
});

ws.on("open", () => {
  ws.send(JSON.stringify({ include: ["conversation_quality_overall"] }));
  ws.send(fs.readFileSync(videoPath));
});

ws.on("message", (data) => {
  const event = JSON.parse(data.toString());
  console.log(JSON.stringify(event)); // verbatim envelope
});
```

If `Authorization` is not supported, connect with subprotocol auth:

```javascript
const ws = new WebSocket("wss://api.interhuman.ai/v1/stream/analyze", apiKey);
```

## Building a Browser App (optional)

If media is captured in a **browser or any untrusted client** (live camera/microphone, end-user app), never ship the API key to it. Read [production-architecture.md](production-architecture.md) before generating code. Two valid architectures — choose by where the analysis events should be delivered:

1. **Direct from the browser with a client token** (simplest): your backend mints a short-lived, capped token via `POST /v1/client_tokens` and hands it to the browser, which opens the WebSocket to Interhuman directly (`access_token` subprotocol, or the TypeScript SDK's `StreamClient`).
2. **Proxy through your backend**: when your backend should receive the events (server-side value-add, storage) or handle the media. Must be a long-lived process that relays segments verbatim.

On either path, send `MediaRecorder` timesliced segments exactly as produced; never re-slice or reassemble media in transit.

Skip this section when calling the endpoint from a trusted server-side process.

## Error Responses

Errors arrive as envelopes with `type: "error"`:

- **data.code** (string): Machine-readable error id (e.g. `ih6002`)
- **data.message** (string): Human-readable explanation
- **data.link** (string, optional): Documentation URL
- **data.segment** (integer, optional): Caller segment when the error maps to a specific chunk

See [error handling](https://docs.interhuman.ai/api-reference/error-handling) for error code details.

## Output Rules

**CRITICAL**: This skill is a strict wrapper. You MUST:

1. Return each JSON event envelope from the API without any modification
2. Do NOT summarize, transform, or rename fields
3. Do NOT extract or filter events
4. Do NOT add commentary or interpretation
5. Preserve all fields exactly as received from the API

Each server message should be passed through verbatim as a single JSON object.

## Additional Resources

- [reference.md](reference.md) — v1 envelope schemas and example payloads
- [production-architecture.md](production-architecture.md) — browser apps: client tokens, proxy option, media segmenting, and debugging
