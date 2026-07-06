# Production architecture: streaming from a browser (or any untrusted client)

Optional detail for building a real application around `wss://api.interhuman.ai/v1/stream/analyze`, where media is captured client-side (browser camera/microphone) and the API key must stay secret. If you are calling the endpoint from a trusted server-side process with a file on disk, you do not need this document — follow SKILL.md.

## Three rules (non-negotiable)

1. **Never open the WebSocket to Interhuman from the browser.** The endpoint authenticates with the API key (`ih_live_...`) on the handshake. A browser-opened socket ships the key to the client (JS bundle, network tab, devtools). Route media through a backend you control.
2. **Run that backend as a long-lived process, not a serverless function.** The socket stays open for the whole recording.
3. **Send media segments exactly as the recorder produces them.** Never re-slice, reassemble, or reinterpret the media container server-side.

A naive browser-direct implementation works in a local demo and fails in production. Do not generate it.

Also: never place the API key in a client-bundled environment variable (`NEXT_PUBLIC_*`, `VITE_*`, `REACT_APP_*`). Same leak.

## The architecture

```
browser ──1─▶ token route (your backend: mints short-lived signed token)
browser ──2─ws + token─▶ stream proxy (holds Interhuman API key)
proxy ──3─wss + Authorization: Bearer─▶ Interhuman /v1/stream/analyze
proxy ◀──analysis events (JSON text)── Interhuman
browser ◀──analysis events (JSON text)── proxy
```

Three parts, one job each:

- **Browser**: captures media, records timesliced segments, sends each as a binary frame to *your* proxy, receives JSON events back.
- **Token route**: mints a short-lived signed token authorizing the browser to connect to the proxy. Stateless single-request work — may be serverless.
- **Stream proxy**: holds the API key, opens one upstream socket to Interhuman per client session, adds `Authorization: Bearer <api_key>` upstream, relays binary media up and JSON events down **verbatim**. It must not parse, buffer, re-slice, or "fix" the media stream.

## Why the proxy cannot be a serverless function

Serverless platforms (Vercel functions, AWS Lambda, etc.) are built around a single request; the proxy owns a stateful socket for minutes. Module-level session state (e.g. a `Map` of sessions) works on localhost (one process) and breaks deployed: requests land on different instances, producing "session not found" errors that never reproduce locally. Run the proxy as its own small persistent service (Node `ws` server on Cloud Run, Fly.io, Railway, a VM). Keep the token route wherever your backend already lives.

## Send media the way the recorder produces it

`MediaRecorder` with a timeslice emits self-contained, valid WebM segments. Send each one as a single binary frame the moment it is produced:

```javascript
const recorder = new MediaRecorder(mediaStream, {
  mimeType: "video/webm;codecs=vp8,opus",
  videoBitsPerSecond: 1_000_000, // 1 Mbps is plenty for analysis
});

recorder.addEventListener("dataavailable", async (event) => {
  if (!event.data || event.data.size === 0) return;
  if (ws.readyState !== WebSocket.OPEN) return;
  ws.send(await event.data.arrayBuffer()); // one segment = one binary frame
});

recorder.start(3000); // self-contained segment every 3 s (~400 KB at 1 Mbps)
```

What does NOT work:

- **One recording as one giant frame.** The endpoint enforces a 32 MB max WebSocket message size (`ih6002`), and large single messages fail unreliably even under the cap (socket reset, close code `1006`, no close frame). Small and many beats big and one.
- **Re-slicing media server-side.** Parsing WebM/EBML on the server and cutting your own boundaries drops trailing metadata or mis-cuts real live-muxed browser output → `ih5004` (malformed segment). Parsers that pass tests against synthetic ffmpeg files still fail on real `MediaRecorder` output.
- **Concatenating timesliced segments later.** The container duration is only written on `stop()`; stitched segments make a broken file. Stream each segment live and independently.

## Authenticate the browser to the proxy

Browsers cannot set an `Authorization` header on a `WebSocket`. Pass a short-lived HMAC-signed token as the WebSocket subprotocol.

Mint (token route):

```javascript
import { createHmac } from "node:crypto";

export function mintStreamToken(sessionId) {
  const payload = Buffer.from(
    JSON.stringify({ sid: sessionId, exp: Date.now() + 180_000 })
  ).toString("base64url");
  const signature = createHmac("sha256", process.env.STREAM_TOKEN_SECRET)
    .update(payload) // sign the base64url STRING, not the raw JSON
    .digest("base64url");
  return `${payload}.${signature}`;
}
```

Connect (browser):

```javascript
const token = await fetch("/api/stream/session").then((r) => r.text());
const ws = new WebSocket(PROXY_WS_URL, [token]); // token as the only subprotocol
```

Verify on the proxy during the upgrade (`handleProtocols` with the `ws` package) and reject with 401 on mismatch. Two known traps:

- Compute the HMAC over the base64url-encoded payload string on **both** sides.
- A secret mismatch surfaces as a **silent 401 on the upgrade** — no body, generic browser error. Check that the token route and the proxy share the same secret before debugging anything else.

## The relay

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

  upstream.on("message", (data, isBinary) => {        // events down: text, verbatim
    if (!isBinary && client.readyState === WebSocket.OPEN) client.send(data.toString());
  });

  client.on("close", () => upstream.close());
  upstream.on("close", () => client.close());
});
```

Lifecycle rules:

- **Do not send media before the upstream is ready.** Open the client socket, wait for the proxy's ready signal, then `recorder.start(3000)`. Time out the connect (~10–15 s) with a real error.
- **Finish with a quiet window, not a "done" event.** Events stream in throughout the recording. After the last segment, keep the connection open until events go quiet (~10 s idle, hard cap ~60 s), then close.
- **Accumulate events client-side.** There is no end-of-session summary to wait for.

## Debugging: symptom → cause

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `ih6002`, or socket drop mid-send, close code `1006`, no close frame (`EPIPE` server-side) | WebSocket message too large | Timesliced 1–3 s segments, one per frame |
| `ih5004` (malformed segment) | Container re-sliced/reassembled/truncated between recorder and Interhuman | Relay recorder-produced segments verbatim |
| Bare `401` on WS upgrade, no body | Token secret mismatch between token route and proxy | Same secret both sides; HMAC over the encoded string both sides |
| Works locally, "session not found" deployed | Session state in module scope on serverless | Persistent-process proxy |
| Two sessions/sockets per recording in dev | React StrictMode double-invokes effects/updaters | Guard session start with a synchronous ref |
| Test files pass, real recordings fail | Pipeline tested only on synthetic (ffmpeg) files | Test with real `MediaRecorder` output |

Add a no-auth `/health` route on the proxy early, returning a boolean (never the value) per required environment variable.

## References

- Guide: https://docs.interhuman.ai/how-to/build-with-the-stream-endpoint
- Endpoint reference: https://docs.interhuman.ai/api-reference/stream-analyze
- Error codes: https://docs.interhuman.ai/api-reference/error-handling
