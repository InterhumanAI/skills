---
name: interhuman-stream-analyze
description: Analyze live or chunked video in real time over the Interhuman API V2 WebSocket (wss://api.interhuman.ai/v2/stream/analyze), preferably through the official TypeScript (@interhumanai/sdk) or Python (interhumanai) SDK with the V2 API version selected explicitly. Use for live streams, MediaRecorder chunks, WebSocket, or /v2/stream/analyze. Returns each JSON event envelope without modification.
---

# Interhuman Stream Analyze (V2)

Analyze live video with the Inter-2 model over a WebSocket. You send binary media chunks, and the server sends back typed JSON event envelopes as analysis progresses.

- **Recommended endpoint**: `wss://api.interhuman.ai/v2/stream/analyze`
- **Recommended client**: the official SDK for the user's environment, with **V2 selected explicitly**. Use a raw WebSocket only as a fallback (see [Raw WebSocket fallback](#raw-websocket-fallback)).

## When to Use

Use this skill when:
- Analyzing live or ongoing video, such as a camera or microphone in a browser, or a capture pipeline
- Sending media in pieces as they are produced, such as `MediaRecorder` timeslices
- You need incremental events (`signal.detected`, `engagement.updated`, ...) while the session runs

Do NOT use this skill for:
- A complete recorded file you want analyzed as a whole. Use **interhuman-post-processing** (`POST /v2/upload/analyze`) instead.

## Choose the Client

| Environment | Use | Install |
|-------------|-----|---------|
| Browser, or Node (Node 22+ has a global `WebSocket`; Node 18/20 pass a `ws` factory) | [`@interhumanai/sdk`](https://docs.interhuman.ai/sdk-reference/typescript-sdk) | `npm install @interhumanai/sdk` |
| Python 3.10+ (server-side) | [`interhumanai`](https://docs.interhuman.ai/sdk-reference/python-sdk) | `pip install interhumanai` |
| Any other language, or an environment where neither SDK can be installed | [Raw WebSocket fallback](#raw-websocket-fallback) | a WebSocket library |

**Select V2 explicitly.** Both SDKs still default to V1, so switching to the SDK alone is not enough:

- TypeScript: `client.stream({ apiVersion: "v2" })`, or `new StreamClient({ tokenProvider, apiVersion: "v2" })`
- Python: `client.stream(api_version="v2")`

## Credentials

- The credential needs the **`interhumanai.stream`** scope. Without it the session is refused with an `ih2003` error envelope and close code `4003`.
- **Server-side** (Node or Python), keep the API key in the environment, for example `INTERHUMAN_API_KEY`, and pass it as `accessToken` / `access_token`.
- **Browser**: never ship the API key, and never put it in a `NEXT_PUBLIC_`, `VITE_`, or `REACT_APP_` variable. Your backend mints a short-lived, capped client token with [`POST /v1/client_tokens`](https://docs.interhuman.ai/api-reference/client-tokens) (`scopes: ["interhumanai.stream"]`, plus caps such as `expiresIn`, `maxDurationSeconds`, `maxVideoSeconds`, and `allowedOrigins`) and returns only the token to the browser.
- **Transport**: browsers cannot set headers on a WebSocket, so the credential travels as the subprotocol pair `Sec-WebSocket-Protocol: access_token, <credential>`, as in `new WebSocket(url, ["access_token", token])`. The TypeScript SDK does this for you. Server-side clients may send `Authorization: Bearer <credential>` instead, which is what the Python SDK does.

See [Authentication](https://docs.interhuman.ai/api-reference/authentication) and [Build with the stream endpoint](https://docs.interhuman.ai/how-to/build-with-the-stream-endpoint).

## Session Lifecycle

1. **Connect** to `/v2/stream/analyze`. The session opens on model `inter-2` by default. You may name it with the `?model=inter-2` query parameter (SDK: `model: "inter-2"`). An invalid value is refused with `ih4005` and close `1008`.
2. **Wait for `session.ready`** before sending media. It carries the session limits: `session_idle_timeout_seconds`, `session_max_duration_seconds`, `min_segment_size_bytes`, `max_segment_size_bytes`, and `max_segment_duration_seconds`. It also carries `supported_session_config_options`: the `include` flags and the `model` values this credential may select. A session that is refused after the handshake gets an `error` envelope and a close **without** `session.ready`. For example, `ih1003` with close `1013` means no backend serves the model right now; nothing is analyzed or billed, so retry later. The SDKs' `waitForSessionReady()` / `wait_for_session_ready()` reject or raise in that case.
3. **Configure (optional)** with a JSON text frame. The server acknowledges it with `session.updated`:
   ```json
   { "include": ["conversation_quality_overall", "conversation_quality_timeline"], "model": "inter-2" }
   ```
   - `include` opts in to `conversation_quality.updated` sections. Each config frame fully replaces the previous `include`.
   - `model` is session state. Omitting it keeps the current model, and changing it is accepted only before the first media chunk.
4. **Send media** as binary frames: one continuous WebM or fragmented-MP4 stream **with both video and audio**, because Inter-2 needs sound as well as a picture. Send chunks in order, exactly as the recorder produced them. The first chunk carries the container header. A browser can use `MediaRecorder` with `video/webm;codecs=vp9,opus` and `recorder.start(3000)`. Keep each frame small (1–3 s of media); the hard limit is `max_segment_size_bytes`, 32 MB by default (`ih6002`). Do not re-slice, re-mux, or concatenate the recorder's output (`ih5004`). Do not send a later chunk on a new session without the chunks before it.
5. **Read events** as they arrive, and keep them client-side. There is no end-of-session summary.
6. **Close gracefully**: send `{"type": "session.close"}` (SDK: `requestClose()` / `request_close()`) *after* the last media chunk. The server answers `session.closing` with `data.max_drain_seconds`, rejects further media, finishes the accepted analysis (including `signal.ended` for still-active signals), sends `session.ended`, and closes with code `1000`. Wait for `session.ended` or the close rather than closing the socket yourself. An immediate `close()` discards the trailing analysis.

## Example: TypeScript SDK (Node, server-side)

```typescript
import { readFile } from "node:fs/promises";
import { IncludeFlag, InterhumanClient } from "@interhumanai/sdk";

const client = new InterhumanClient({
  accessToken: process.env.INTERHUMAN_API_KEY!, // server-side only
  // Node 18/20: webSocket: (url, protocols) => new WebSocket(url, protocols) as any,  (import WebSocket from "ws")
});

const stream = client.stream({ apiVersion: "v2" }); // explicit: the SDK default is still "v1"

// Every server envelope, verbatim. The SDK passes the parsed JSON through unchanged.
stream.on("message", (event) => console.log(JSON.stringify(event)));
const closed = new Promise<{ code: number; reason: string }>((resolve) => stream.once("close", resolve));

await stream.connect();
await stream.waitForSessionReady(); // rejects if the session is refused (e.g. ih2003, ih1003)
stream.updateConfig({ include: [IncludeFlag.ConversationQualityOverall] });

// Send media chunks in order, as your recorder or pipeline produces them.
// A short, complete recording with video + audio can go as a single chunk.
stream.sendVideo(await readFile("/path/to/clip.webm"));

stream.requestClose(); // after the last chunk: drain, session.ended, close 1000
const info = await closed;
if (info.code !== 1000) process.exitCode = 1;
```

Typed handlers are available too, such as `stream.on("signal.detected", (e) => ...)`, `"engagement.updated"`, `"error"`, and `"session.ended"`. `"socketError"` reports transport failures, which are separate from the `error` envelope.

## Example: TypeScript SDK (browser, client token)

```typescript
import { StaticTokenProvider, StreamClient } from "@interhumanai/sdk";

// Your backend route mints the client token with POST /v1/client_tokens.
const clientToken = await fetch("/api/interhuman/stream-token").then((r) => r.text());

const stream = new StreamClient({
  tokenProvider: new StaticTokenProvider(clientToken),
  apiVersion: "v2",
});
stream.on("message", (event) => console.log(JSON.stringify(event)));

const media = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
const mimeType = ["video/webm;codecs=vp9,opus", "video/webm;codecs=vp8,opus", "video/webm"].find(
  (type) => MediaRecorder.isTypeSupported(type),
);
const recorder = new MediaRecorder(media, mimeType ? { mimeType } : {});

// Keep this handler synchronous so every blob goes out in order before session.close.
recorder.addEventListener("dataavailable", (event) => {
  if (event.data.size > 0) stream.sendVideo(event.data);
  if (recorder.state === "inactive") stream.requestClose(); // after the final blob
});
stream.on("session.ended", () => media.getTracks().forEach((track) => track.stop()));

await stream.connect();
await stream.waitForSessionReady();
recorder.start(3000); // one blob every 3 s
// Later, to stop: recorder.stop(). The handler sends the last blob, then session.close.
```

## Example: Python SDK (server-side)

```python
import asyncio
import json
import os

from interhumanai import IncludeFlag, InterhumanClient, UnknownEvent


async def send_media(stream) -> None:
    # Send chunks in order, as your recorder or pipeline produces them.
    # A short, complete recording with video + audio can go as a single chunk.
    with open("/path/to/clip.webm", "rb") as f:
        await stream.send_video(f.read())
    await stream.request_close()  # after the last chunk: drain, session.ended, close 1000


async def main() -> None:
    client = InterhumanClient(access_token=os.environ["INTERHUMAN_API_KEY"])
    stream = client.stream(api_version="v2")  # explicit: the SDK default is still "v1"

    async with stream:  # connects; closes on exit
        await stream.wait_for_session_ready()  # raises if refused (e.g. ih2003, ih1003)
        await stream.update_config(include=[IncludeFlag.CONVERSATION_QUALITY_OVERALL])
        sender = asyncio.create_task(send_media(stream))

        # Iteration yields every envelope (session.ready included) until the server closes.
        async for event in stream:
            payload = (
                event.raw
                if isinstance(event, UnknownEvent)
                else event.model_dump(mode="json", exclude_unset=True)
            )
            print(json.dumps(payload))
        await sender

    if stream.close_info and stream.close_info.code != 1000:
        raise SystemExit(1)


asyncio.run(main())
```

The Python SDK yields pydantic models. `model_dump(mode="json", exclude_unset=True)` keeps exactly the fields the server sent, including explicit `null`s. Without it, V2 signal events would gain a `modality: null` that the API never sends. Event types this SDK version does not know arrive as `UnknownEvent` with the original envelope in `.raw`.

## Raw WebSocket Fallback

Use a raw WebSocket only when neither SDK fits. You then own the whole lifecycle above: authentication, waiting for `session.ready`, ordering, `session.close`, and close codes.

```javascript
// Browser: credential via the subprotocol pair; use a client token, never the API key.
const ws = new WebSocket("wss://api.interhuman.ai/v2/stream/analyze", ["access_token", clientToken]);
ws.binaryType = "arraybuffer";
ws.addEventListener("message", (e) => {
  const event = JSON.parse(e.data);
  console.log(e.data); // verbatim envelope
  if (event.type === "session.ready") {
    ws.send(JSON.stringify({ include: ["conversation_quality_overall"] }));
    // start sending recorder chunks with ws.send(blob) ... then:
    // ws.send(JSON.stringify({ type: "session.close" }));
  }
});
```

```python
import asyncio
import json
import os

import websockets


async def main() -> None:
    headers = {"Authorization": f"Bearer {os.environ['INTERHUMAN_API_KEY']}"}
    async with websockets.connect(
        "wss://api.interhuman.ai/v2/stream/analyze", additional_headers=headers, max_size=None
    ) as ws:
        async for message in ws:  # ends when the server closes
            print(message)  # verbatim envelope
            event = json.loads(message)
            if event["type"] == "session.ready":
                await ws.send(json.dumps({"include": ["conversation_quality_overall"]}))
                with open("/path/to/clip.webm", "rb") as f:
                    await ws.send(f.read())
                await ws.send(json.dumps({"type": "session.close"}))


asyncio.run(main())
```

## Server Messages

Every server message is a JSON text frame with `type`, `timestamp` (ISO 8601), `correlation_id`, and `data`. Branch on `type`. All times in `data` (`start`, `end`, `ranges`) are **absolute, session-cumulative seconds** across every chunk sent on the connection.

| `type` | `data` fields |
|--------|---------------|
| `session.ready` | `session_idle_timeout_seconds`, `session_max_duration_seconds`, `max_segment_duration_seconds` (or `null`), `min_segment_size_bytes`, `max_segment_size_bytes`, `supported_session_config_options` (`include`, `model`) |
| `session.updated` | `include`, `model` (the active model) |
| `signal.detected` | `signal_type`, `start`, `probability`, `rationale`. A signal became active. |
| `signal.updated` | `signal_type`, `start`, `probability`, `rationale`. An active signal's probability or rationale changed. |
| `signal.ended` | `signal_type`, `end`. A signal stopped being active. |
| `engagement.updated` | `state` (`engaged`, `neutral`, or `disengaged`), `start`. Sent when the level changes. |
| `conversation_quality.updated` | `overall` and/or `timeline`, per the session's `include`. Values 0–100: `quality_index`, `clarity`, `authority`, `energy`, `rapport`, `learning`. |
| `coverage.degraded` | `ranges[]`, `reason` (`video_gap`). Analyzed with partial video; informational. |
| `coverage.dropped` | `ranges[]`. Skipped under backpressure or a stream gap; not billed; informational. |
| `session.closing` | `max_drain_seconds` |
| `session.ended` | `reason` (`client_shutdown`). This is the last message; the server then closes with code `1000`. |
| `error` | `code`, `message`, `link` (or `null`), `segment` (or `null`) |

A deployment may send fields beyond the ones listed here. For example, `session.ready` and `session.updated` can carry `goal_dimensions`. Pass them through untouched and do not depend on them.

**V2 signal payloads have no `modality` field**, because the session's `model` names the evidence. Do not read or require `data.modality`, and do not add it.

`signal_type` is one of `agreement`, `confidence`, `confusion`, `disagreement`, `frustration`, `hesitation`, `interest`, `skepticism`, `stress`, or `uncertainty`. These are the published types; see [Social signals](https://docs.interhuman.ai/explanations/social-signals). The SDK `SignalType` unions may list extra values. `probability` is `high`, `medium`, or `low`.

```json
{
  "type": "signal.detected",
  "timestamp": "2025-01-01T00:00:00.000000Z",
  "correlation_id": "550e8400-e29b-41d4-a716-446655440000",
  "data": {
    "signal_type": "agreement",
    "start": 3.0,
    "probability": "high",
    "rationale": "Subject nodded repeatedly while maintaining eye contact."
  }
}
```

Full schemas: [Inter-2 Streaming Analyze](https://docs.interhuman.ai/api-reference/stream-analyze-v2) (AsyncAPI channel `stream_analyze_v2`).

## Errors and Close Codes

Errors arrive as `error` envelopes (`data.code`, `data.message`, `data.link`, `data.segment`). Some are followed by a close:

| Code | Meaning | What to do |
|------|---------|------------|
| `ih2003` (close `4003`) | Credential lacks `interhumanai.stream` | Use a key or token with the scope |
| `ih4005` (close `1008`) | Invalid `model` query parameter | Use `inter-2` or omit it |
| `ih1003` (close `1013`) | No backend serves the model right now | Retry the session later |
| `ih6001` | Invalid session config frame | Fix the frame; the session stays open |
| `ih6002` | Message too large | Send smaller chunks (1–3 s) |
| `ih5004` | Malformed video segment | Send recorder output verbatim, in order |
| `ih5001` | No usable video in the stream | Record with a camera track |
| `ih6003` / `ih6004` | Idle timeout / maximum session duration | Open a new session |

With subprotocol authentication (browsers, and the TypeScript SDK everywhere), the two handshake refusals `ih4005` and `ih1003` currently reach the client as a failed connection: a transport error or close `1006`, without the `error` envelope. Server-side clients that send `Authorization` receive the envelope and close code. Treat an immediate failure before `session.ready` on a correct URL and model as "retry later".

An HTTP `401` on the upgrade means an expired, revoked, or invalid credential, or a subprotocol that is not the `access_token, <credential>` pair. A connection rejected despite a valid client token usually means the page `Origin` is missing from the token's `allowed_origins`. See [Error handling](https://docs.interhuman.ai/api-reference/error-handling).

## Output Rules

**CRITICAL**: This skill is a strict wrapper. You MUST:

1. Return each server envelope as one JSON object, exactly as received, in arrival order, including `session.*`, `coverage.*`, and `error` envelopes.
2. Not summarize, transform, rename, filter, or merge events.
3. Not add fields the API did not send, such as a V1-style `modality` or SDK default `null`s.
4. Not add commentary or interpretation.
5. Serialize SDK objects faithfully:
   - **TypeScript**: use `JSON.stringify(event)`, from the `"message"` handler.
   - **Python**: use `event.model_dump(mode="json", exclude_unset=True)`, or `event.raw` for an `UnknownEvent`. Timestamps are re-rendered as ISO 8601. If the user needs the exact frame text, use the raw WebSocket fallback.

## V1 Compatibility

`WS /v1/stream/analyze` (Inter-1) is still served for existing integrations, and it is what the SDKs open when no API version is given. It is not the recommended path. V1 signal payloads carry `modality`, V1 reports `model` as `null`, and V1 rejects a config frame that names a `model`. To move an integration, follow [Migrate from V1 to V2](https://docs.interhuman.ai/api-reference/migrate-v1-to-v2): change the path (or set the SDK's API version to V2), stop reading `modality`, and record with audio.

## References

- [Inter-2 Streaming Analyze](https://docs.interhuman.ai/api-reference/stream-analyze-v2)
- [Stream analysis quickstart](https://docs.interhuman.ai/getting-started/stream-analyze-quickstart)
- [Build with the stream endpoint](https://docs.interhuman.ai/how-to/build-with-the-stream-endpoint)
- [Migrate from V1 to V2](https://docs.interhuman.ai/api-reference/migrate-v1-to-v2)
- [TypeScript SDK](https://docs.interhuman.ai/sdk-reference/typescript-sdk) and [Python SDK](https://docs.interhuman.ai/sdk-reference/python-sdk)
- [Authentication](https://docs.interhuman.ai/api-reference/authentication) and [Client Tokens](https://docs.interhuman.ai/api-reference/client-tokens)
- [Error handling](https://docs.interhuman.ai/api-reference/error-handling)
