# Production architecture: streaming from a browser (or any untrusted client)

Optional detail for building a real application around `wss://api.interhuman.ai/v1/stream/analyze`, where media is captured client-side (browser camera/microphone) and the API key must stay secret. If you are calling the endpoint from a trusted server-side process with a file on disk, you do not need this document — follow SKILL.md.

## Three rules (non-negotiable)

1. **Never ship the API key to the browser.** Mint a short-lived, capped **client token** server-side and hand that to the client instead.
2. **Let the browser stream directly to Interhuman with the client token.** The token is designed for exactly this, and the API enforces its caps.
3. **Send media segments exactly as the recorder produces them.** Never re-slice, reassemble, or reinterpret the media container in between.

A naive implementation that hardcodes the API key in client code works in a local demo and leaks the credential in production. Do not generate it.

## Why you can't ship the API key

The stream endpoint authenticates with the API key (`ih_live_...`) as a Bearer credential on the WebSocket handshake. If that key appears anywhere in client code, it ships to the user — JS bundle, network tab, devtools. That is a leaked production credential.

Also: never place the API key in a client-bundled environment variable (`NEXT_PUBLIC_*`, `VITE_*`, `REACT_APP_*`). Same leak.

The key stays server-side. What the browser gets instead is a **client token**: a short-lived credential the backend mints from `POST /v1/client_tokens`, scoped and capped so that leaking it costs a few minutes of bounded usage, not the account.

## The architecture

```
browser ──1─▶ your backend ──2─POST /v1/client_tokens (API key)─▶ Interhuman
browser ◀──capped client token── your backend
browser ──3─wss /v1/stream/analyze + client token─▶ Interhuman
browser ◀──analysis events (JSON text)── Interhuman
```

Two parts, one job each:

- **A token route on your backend** calls `POST /v1/client_tokens` with the API key and returns the minted token to the browser. Stateless, single-request work — a serverless function is fine.
- **The browser** connects directly to `wss://api.interhuman.ai/v1/stream/analyze` with the client token, streams recorded segments up as binary frames, and receives analysis events back. The caps embedded in the token are enforced by the API itself.

## Mint a client token

Use the TypeScript SDK (`@interhumanai/sdk`) in the token route, or call the endpoint directly:

```typescript
import { AuthClient } from "@interhumanai/sdk";

const auth = new AuthClient();
const token = await auth.createClientToken({
  apiKey: process.env.INTERHUMAN_API_KEY,   // never leaves the server
  scopes: ["interhumanai.stream"],           // default
  expiresIn: 300,                            // seconds; clamped to 60–3600
  maxDurationSeconds: 600,                   // max wall-clock session length
  maxVideoSeconds: 600,                      // total video budget for this token
  allowedOrigins: ["https://app.example.com"],
});
// hand token.access_token to the browser
```

Raw HTTP equivalent: `POST https://api.interhuman.ai/v1/client_tokens` with JSON body `{"api_key": "...", "scopes": ["interhumanai.stream"], "expires_in": 300, ...}`.

Every cap is optional, but set them deliberately — they are the blast radius if a token leaks:

| Cap | Meaning | Default |
| --- | --- | --- |
| `expires_in` | Token time-to-live in seconds | 300 (clamped to 60–3600) |
| `max_duration_seconds` | Max wall-clock duration of a session opened with the token | unset |
| `max_bytes` | Max cumulative video bytes a session may send | unset |
| `max_concurrent` | Max simultaneous sessions on one token | 1 |
| `max_video_seconds` | Total seconds of video the token may process — spans upload, stream, and real-time combined | unset |
| `allowed_origins` | Browser `Origin` allow-list; connections from other origins are rejected | unset |

Tokens can be **revoked** early (`POST /v1/client_tokens/revoke`, or `auth.revokeClientToken(...)` in the SDK) — new requests are rejected immediately and any live session is torn down on its next chunk. Useful when a token outlives the thing it was minted for, like a user logging out mid-session.

## Connect from the browser

The SDK's `StreamClient` handles connection, authentication, typed events, and graceful shutdown:

```typescript
import { StreamClient, StaticTokenProvider } from "@interhumanai/sdk";

const clientToken = await fetch("/api/stream/session").then((r) => r.text());

const stream = new StreamClient({
  tokenProvider: new StaticTokenProvider(clientToken),
});

stream.on("signal.detected", (e) => console.log(e.data.signal_type));
stream.on("engagement.updated", (e) => console.log(e.data));

await stream.connect();
await stream.waitForSessionReady();
stream.updateConfig({ include: ["conversation_quality_overall"] });
// now start recording and call stream.sendVideo(chunk) per segment
```

Without the SDK: browsers cannot set an `Authorization` header on a `WebSocket`, so pass the token via the subprotocol pair the endpoint accepts:

```javascript
const ws = new WebSocket("wss://api.interhuman.ai/v1/stream/analyze", [
  "access_token",
  clientToken,
]);
```

## Send media the way the recorder produces it

`MediaRecorder` with a timeslice emits self-contained, valid WebM segments. Send each one as a single binary frame the moment it is produced:

```javascript
const recorder = new MediaRecorder(mediaStream, {
  mimeType: "video/webm;codecs=vp8,opus",
  videoBitsPerSecond: 1_000_000, // 1 Mbps is plenty for analysis
});

recorder.addEventListener("dataavailable", async (event) => {
  if (!event.data || event.data.size === 0) return;
  stream.sendVideo(await event.data.arrayBuffer()); // one segment = one binary frame
});

recorder.start(3000); // self-contained segment every 3 s (~400 KB at 1 Mbps)
```

What does NOT work:

- **One recording as one giant frame.** The endpoint enforces a 32 MB max WebSocket message size (`ih6002`), and large single messages fail unreliably even under the cap (socket reset, close code `1006`, no close frame). Small and many beats big and one.
- **Re-slicing media in transit.** Parsing WebM/EBML and cutting your own boundaries drops trailing metadata or mis-cuts real live-muxed browser output → `ih5004` (malformed segment). Parsers that pass tests against synthetic ffmpeg files still fail on real `MediaRecorder` output.
- **Concatenating timesliced segments later.** The container duration is only written on `stop()`; stitched segments make a broken file. Stream each segment live and independently.

## End the session cleanly

Analysis events stream in throughout the recording; accumulate them client-side as they arrive — there is no end-of-session summary to wait for.

When the user stops recording, request a graceful shutdown instead of closing the socket: send `session.close` (SDK: `stream.requestClose()`). The server acknowledges with `session.closing` (`data.max_drain_seconds` is the longest to wait), finishes analyzing the video it already accepted — emitting the remaining envelopes plus `signal.ended` for still-active signals — then sends `session.ended` and closes with code 1000. Listen for `session.ended` (or `close`) rather than closing early, which discards trailing analysis.

```javascript
recorder.stop();
stream.requestClose();
stream.on("session.ended", () => {
  mediaStream.getTracks().forEach((t) => t.stop());
});
```

## Debugging: symptom → cause

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `401` on the WebSocket upgrade | Client token expired or revoked, or the subprotocol isn't the `access_token, <token>` pair | Mint a fresh token; use `new WebSocket(url, ["access_token", token])` |
| Connection rejected despite a valid token | Page `Origin` missing from the token's `allowed_origins` | Mint the token with the right origins per environment |
| Processing refused mid-session | Token's `max_video_seconds` budget exhausted (it spans upload, stream, and real-time) | Mint a new token; size the budget to the session length |
| `ih6002`, or socket drops mid-send with close code `1006` and no close frame | A WebSocket message is too large — over the 32 MB cap, or large enough to be unreliable | Send small timesliced segments (1–3 s each), one per frame |
| `ih5004` (malformed segment) | The media container was re-sliced, reassembled, or truncated between the recorder and Interhuman | Send recorder-produced segments verbatim; never rebuild the stream from parsed parts |
| Two sessions/sockets per recording in development | React StrictMode double-invokes effects and state updaters | Guard session start with a synchronous ref check, not state |
| Segments from test files analyze fine; real recordings fail | Pipeline only tested against synthetic (ffmpeg) files | Always test with real `MediaRecorder` output from a real browser |

## References

- Client tokens API: https://interhumanai-realtime-internal.mintlify.app/api-reference/client-tokens
- TypeScript SDK: https://interhumanai-realtime-internal.mintlify.app/sdk-reference/typescript-sdk
- Endpoint reference: https://docs.interhuman.ai/api-reference/stream-analyze
- Error codes: https://docs.interhuman.ai/api-reference/error-handling
