# Production architecture: streaming from a browser (or any untrusted client)

Optional detail for building a real application around `wss://api.interhuman.ai/v1/stream/analyze`, where media is captured client-side (browser camera/microphone) and the API key must stay secret. If you are calling the endpoint from a trusted server-side process with a file on disk, you do not need this document — follow SKILL.md.

## Rules

1. **Never ship the API key to the client.** The key (`ih_live_...`) is a full production credential. It must not appear in client code, JS bundles, or client-bundled environment variables (`NEXT_PUBLIC_*`, `VITE_*`, `REACT_APP_*`).
2. **Pick one of two architectures below.** Both are valid; choose by where the analysis events should be delivered. Direct-from-browser with a client token is the simplest path. A proxy through your backend is the right choice when your backend should receive the events or the media.
3. **Send media segments exactly as the recorder produces them** — on every path. Never re-slice, reassemble, or reinterpret the media container in transit.

A naive implementation that hardcodes the API key in client code works in a local demo and leaks the credential in production. Do not generate it.

## Option A: direct from the browser with a client token

The browser can open the WebSocket to Interhuman itself — safely — using a **client token**: a short-lived, capped credential your backend mints server-side with the API key.

```
browser ──1─▶ your backend ──2─POST /v1/client_tokens (API key)─▶ Interhuman
browser ◀──capped client token── your backend
browser ──3─wss /v1/stream/analyze + client token─▶ Interhuman
browser ◀──analysis events (JSON text)── Interhuman
```

The token route is stateless single-request work (serverless is fine). Events are delivered to the browser; no media touches your servers.

Mint server-side (TypeScript SDK `@interhumanai/sdk`, or raw `POST /v1/client_tokens`):

```typescript
import { AuthClient } from "@interhumanai/sdk";

const auth = new AuthClient();
const token = await auth.createClientToken({
  apiKey: process.env.INTERHUMAN_API_KEY,   // never leaves the server
  scopes: ["interhumanai.stream"],           // default
  expiresIn: 300,                            // seconds; clamped to 60–3600
  maxDurationSeconds: 600,                   // max wall-clock session length
  maxVideoSeconds: 600,                      // total video budget (spans upload/stream/real-time)
  allowedOrigins: ["https://app.example.com"],
});
// hand token.access_token to the browser
```

All caps are optional but set them deliberately — they bound the damage if a token leaks. `max_concurrent` defaults to 1. Tokens can be revoked early (`POST /v1/client_tokens/revoke` / `auth.revokeClientToken(...)`); a live session is torn down on its next chunk.

Connect from the browser — SDK:

```typescript
import { StreamClient, StaticTokenProvider } from "@interhumanai/sdk";

const stream = new StreamClient({ tokenProvider: new StaticTokenProvider(clientToken) });
stream.on("signal.detected", (e) => console.log(e.data.signal_type));
await stream.connect();
await stream.waitForSessionReady();
stream.updateConfig({ include: ["conversation_quality_overall"] });
```

Or raw WebSocket (browsers cannot set an `Authorization` header; use the subprotocol pair):

```javascript
const ws = new WebSocket("wss://api.interhuman.ai/v1/stream/analyze", [
  "access_token",
  clientToken,
]);
```

## Option B: proxy through your backend

Choose this when your backend should be the one receiving the analysis events (server-side scoring, storage, value-add on top of the signals) or must handle the media itself. The browser streams to *your* WebSocket server, which relays to Interhuman:

```
browser ──ws──▶ proxy (holds API key) ──wss + Authorization: Bearer──▶ Interhuman
browser ◀──events (as relayed/processed by you)── proxy ◀──events── Interhuman
```

Requirements:

- **Long-lived process, never a serverless function.** The proxy owns a stateful socket for minutes. Module-level session state on serverless works on localhost (one process) and breaks deployed — requests land on different instances, producing "session not found" errors that never reproduce locally. Run it as its own small persistent service (Cloud Run, Fly.io, Railway, a VM).
- **Relay media verbatim.** Binary segments up exactly as received; do not parse, buffer, or re-slice (see Rules, and the media section below).
- **Authenticate the browser to the proxy** with a short-lived token passed as the WebSocket subprotocol (browsers cannot set headers). A minted client token works; so does your own HMAC-signed token:

```javascript
import { createHmac } from "node:crypto";

export function mintProxyToken(sessionId) {
  const payload = Buffer.from(
    JSON.stringify({ sid: sessionId, exp: Date.now() + 180_000 })
  ).toString("base64url");
  const signature = createHmac("sha256", process.env.STREAM_TOKEN_SECRET)
    .update(payload) // sign the base64url STRING, not the raw JSON
    .digest("base64url");
  return `${payload}.${signature}`;
}
```

  Known traps: compute the HMAC over the base64url-encoded payload string on **both** sides, and note that a secret mismatch surfaces as a **silent 401 on the WS upgrade** — no body, generic browser error.

Relay skeleton:

```javascript
import { WebSocket, WebSocketServer } from "ws";

const wss = new WebSocketServer({ server, handleProtocols: verifyToken });

wss.on("connection", (client) => {
  const upstream = new WebSocket("wss://api.interhuman.ai/v1/stream/analyze", {
    headers: { Authorization: `Bearer ${process.env.INTERHUMAN_API_KEY}` },
  });

  upstream.on("open", () => {
    upstream.send(JSON.stringify({ include: ["conversation_quality_overall"] }));
    client.send(JSON.stringify({ type: "proxy.ready" })); // your own signal
  });

  client.on("message", (data, isBinary) => {          // media up: binary, verbatim
    if (isBinary && upstream.readyState === WebSocket.OPEN) upstream.send(data);
  });

  upstream.on("message", (data, isBinary) => {        // events down: process or relay
    if (!isBinary && client.readyState === WebSocket.OPEN) client.send(data.toString());
  });

  client.on("close", () => upstream.close());
  upstream.on("close", () => client.close());
});
```

Do not send media before the upstream is ready: open the client socket, wait for the ready signal, then start recording. Time out the connect (~10–15 s) with a real error.

## Send media the way the recorder produces it

`MediaRecorder` with a timeslice emits self-contained, valid WebM segments. Send each one as a single binary frame the moment it is produced:

```javascript
const recorder = new MediaRecorder(mediaStream, {
  mimeType: "video/webm;codecs=vp8,opus",
  videoBitsPerSecond: 1_000_000, // 1 Mbps is plenty for analysis
});

recorder.addEventListener("dataavailable", async (event) => {
  if (!event.data || event.data.size === 0) return;
  ws.send(await event.data.arrayBuffer()); // one segment = one binary frame
  // SDK equivalent: stream.sendVideo(await event.data.arrayBuffer())
});

recorder.start(3000); // self-contained segment every 3 s (~400 KB at 1 Mbps)
```

What does NOT work:

- **One recording as one giant frame.** The endpoint enforces a 32 MB max WebSocket message size (`ih6002`), and large single messages fail unreliably even under the cap (socket reset, close code `1006`, no close frame). Small and many beats big and one.
- **Re-slicing media in transit.** Parsing WebM/EBML and cutting your own boundaries drops trailing metadata or mis-cuts real live-muxed browser output → `ih5004` (malformed segment). Parsers that pass tests against synthetic ffmpeg files still fail on real `MediaRecorder` output.
- **Concatenating timesliced segments later.** The container duration is only written on `stop()`; stitched segments make a broken file. Stream each segment live and independently.

## End the session cleanly

Analysis events stream in throughout the recording; accumulate them as they arrive — there is no end-of-session summary. When the user stops, request a graceful shutdown instead of closing the socket: send `session.close` (SDK: `stream.requestClose()`). The server acknowledges with `session.closing` (`data.max_drain_seconds` is the longest to wait), finishes analyzing accepted video — emitting remaining envelopes plus `signal.ended` for still-active signals — then sends `session.ended` and closes with code 1000. Listen for `session.ended` (or `close`) rather than closing early, which discards trailing analysis.

## Debugging: symptom → cause

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `401` on the WS upgrade (direct) | Client token expired/revoked, or subprotocol isn't the `access_token, <token>` pair | Mint a fresh token; use `new WebSocket(url, ["access_token", token])` |
| Connection rejected despite valid token | Page `Origin` not in the token's `allowed_origins` | Mint with the right origins per environment |
| Processing refused mid-session | Token's `max_video_seconds` budget exhausted | Mint a new token; size the budget to the session |
| Bare `401` on WS upgrade (proxy) | HMAC token secret mismatch between token route and proxy | Same secret both sides; HMAC over the encoded string both sides |
| `ih6002`, or socket drop mid-send, close `1006`, no close frame | WebSocket message too large | Timesliced 1–3 s segments, one per frame |
| `ih5004` (malformed segment) | Container re-sliced/reassembled/truncated in transit | Send recorder-produced segments verbatim |
| Works locally, "session not found" deployed | Session state in module scope on serverless | Persistent-process proxy |
| Two sessions/sockets per recording in dev | React StrictMode double-invokes effects/updaters | Guard session start with a synchronous ref |
| Test files pass, real recordings fail | Pipeline tested only on synthetic (ffmpeg) files | Test with real `MediaRecorder` output |

Add a no-auth `/health` route on your backend early, returning a boolean (never the value) per required environment variable.

## References

- Client tokens API: https://interhumanai-realtime-internal.mintlify.app/api-reference/client-tokens
- TypeScript SDK: https://interhumanai-realtime-internal.mintlify.app/sdk-reference/typescript-sdk
- Guide: https://docs.interhuman.ai/how-to/build-with-the-stream-endpoint
- Endpoint reference: https://docs.interhuman.ai/api-reference/stream-analyze
- Error codes: https://docs.interhuman.ai/api-reference/error-handling
