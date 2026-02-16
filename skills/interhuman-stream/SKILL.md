---
name: interhuman-stream
description: Wrapper for Interhuman API WebSocket /v0/stream/analyze endpoint. Analyzes live video streams in real-time and relays raw JSON messages as received. Use when the user wants to analyze live camera input or continuous video streams (not pre-recorded files). Returns all WebSocket messages exactly as received from the API without modification.
---

# Interhuman Streaming Analysis

Wrapper for the Interhuman API WebSocket streaming endpoint that analyzes live video streams and returns social-intelligence signals in real-time.

## When to Use

Use this skill when:
- Analyzing live video streams from a camera or continuous feed
- You need real-time analysis as video is being captured
- Processing ongoing video input segment by segment

Do NOT use this skill for:
- Pre-recorded video files (use `interhuman-post-processing` instead)
- One-time file uploads

## Required Inputs

1. **API Access Token**: Bearer token obtained from the `interhuman-authentication` skill (use `interhumanai.stream` scope)
2. **Video Stream Source**: Live video feed that can be segmented into binary chunks
   - Segment size: 10 KB minimum, 20 MB maximum per segment
   - Formats: mp4, avi, mov, mkv, mpeg-ts, mpeg-2-ts, webm (typically WebM for browser-based streams)

## Authentication

Before using this skill, you must obtain an access token using the `interhuman-authentication` skill with the `interhumanai.stream` scope. 

**Important**: Browser WebSocket API cannot send custom headers. For browser-based applications, pass the token via the `protocols` parameter (see JavaScript example below). For Node.js/Python backends, use the `Authorization` header in connection options.

## WebSocket Connection

### Endpoint Details

- **WebSocket URL**: `wss://api.interhuman.ai/v0/stream/analyze`
- **Authentication**: Bearer token passed via `Sec-WebSocket-Protocol` header (browsers) or `Authorization` header (Node.js/Python)
- **Protocol**: WebSocket (WSS)

### Connection Example: JavaScript (Browser)

**Note**: Browser WebSocket API cannot send custom headers. Use the `protocols` parameter to pass the token:

```javascript
const token = "YOUR_ACCESS_TOKEN";
// Pass token via protocols parameter (Sec-WebSocket-Protocol header)
const protocols = ['access_token', token];
const socket = new WebSocket("wss://api.interhuman.ai/v0/stream/analyze", protocols);

socket.onopen = () => console.log("WebSocket connected");

socket.onmessage = (event) => {
  const message = JSON.parse(event.data);
  console.log("Server message:", message);
};

socket.onerror = (error) => console.error("WebSocket error:", error);
socket.onclose = () => console.log("WebSocket closed");
```

### Connection Example: JavaScript (Node.js)

For Node.js environments using the `ws` library, you can use custom headers:

```javascript
import WebSocket from 'ws';

const token = "YOUR_ACCESS_TOKEN";
const socket = new WebSocket("wss://api.interhuman.ai/v0/stream/analyze", {
  headers: {
    Authorization: `Bearer ${token}`
  }
});

socket.on('open', () => console.log("WebSocket connected"));
socket.on('message', (data) => {
  const message = JSON.parse(data.toString());
  console.log("Server message:", message);
});
socket.on('error', (error) => console.error("WebSocket error:", error));
socket.on('close', () => console.log("WebSocket closed"));
```

### Connection Example: Python

```python
import asyncio
import json
import websockets

token = "YOUR_ACCESS_TOKEN"
url = "wss://api.interhuman.ai/v0/stream/analyze"

async def main():
    async with websockets.connect(
        url, extra_headers={"Authorization": f"Bearer {token}"}
    ) as ws:
        print("WebSocket connected")
        async for message in ws:
            data = json.loads(message)
            print("Server message:", data)

asyncio.run(main())
```

## Sending Video Segments

Send binary video segments over the WebSocket connection. Each segment should be:

- Binary encoded (WebM format recommended for browser streams)
- Between 10 KB and 20 MB in size
- Sent as raw binary data (not JSON-encoded)

### Example: JavaScript (Browser)

```javascript
async function startStreaming(socket, stream) {
  const recorder = new MediaRecorder(stream, {
    mimeType: "video/webm;codecs=vp8,opus"
  });

  recorder.ondataavailable = async (event) => {
    const size = event.data?.size ?? 0;
    if (
      event.data &&
      size >= 10_000 && // 10 KB minimum
      size <= 20 * 1024 * 1024 && // 20 MB maximum
      socket.readyState === WebSocket.OPEN
    ) {
      const buffer = await event.data.arrayBuffer();
      socket.send(buffer);
    }
  };

  recorder.start(5000); // produce ~5s segments
  return recorder;
}
```

### Example: Python

```python
async def send_segments(ws, chunks):
    for chunk in chunks:  # chunks is an iterable of binary WebM blobs (10 KB - 20 MB each)
        await ws.send(chunk)
```

## Server Message Types

The server sends JSON messages with different `status` values. You will receive these message types:

### 1. Processing Message

Sent immediately when a video segment is received:

```json
{
  "status": "processing",
  "segment": 1,
  "bytes": 1200000
}
```

### 2. Result Message

Sent when analysis completes for a segment:

```json
{
  "status": "result",
  "segment": 1,
  "signals": [
    {
      "type": "agreement",
      "start": 0.0,
      "end": 5.0
    },
    {
      "type": "engagement",
      "start": 2.3,
      "end": 4.8
    }
  ]
}
```

Each signal in the `signals` array contains:
- **type** (string): Signal type (`agreement`, `confidence`, `confusion`, `disagreement`, `disengagement`, `engagement`, `frustration`, `hesitation`, `interest`, `skepticism`, `stress`, `uncertainty`)
- **start** (number): Start time in seconds relative to segment start
- **end** (number): End time in seconds relative to segment start

### 3. Completion Message

Sent when processing finishes for a segment:

```json
{
  "status": "completed",
  "segment": 1
}
```

### 4. Error Message

Sent when an error occurs:

```json
{
  "status": "error",
  "segment": 2,
  "error": "Video chunk too large: 31000000 bytes (maximum: 20971520 bytes)"
}
```

## Message Flow

For each video segment you send, you will typically receive:

1. `processing` message (immediately upon receipt)
2. `result` message (when analysis completes)
3. `completed` message (when processing finishes)

If an error occurs, you'll receive an `error` message instead.

## Output Rules

**CRITICAL**: This skill is a strict wrapper. You MUST:

1. Relay all WebSocket messages exactly as received from the API
2. Do NOT modify, transform, or filter messages
3. Do NOT summarize or combine multiple messages
4. Do NOT add commentary or interpretation
5. Preserve message ordering (messages arrive in sequence)
6. Return each message as a separate JSON object, exactly as received

Each message from the server should be passed through verbatim without any modification.
