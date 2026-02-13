---
name: interhuman-post-processing
description: Wrapper for Interhuman API POST /v0/upload/analyze endpoint. Analyzes completed video files and returns raw JSON responses with detected social signals. Use when the user wants to analyze a pre-recorded video file (not live streams). Returns the exact JSON response from the API without modification.
---

# Interhuman Post-Processing Analysis

Wrapper for the Interhuman API upload endpoint that analyzes completed video files and returns social-intelligence signals.

## When to Use

Use this skill when:
- Analyzing a pre-recorded video file (MP4, AVI, MOV, MKV, MPEG-TS, MPEG-2-TS, WebM)
- The video file is already complete (not a live stream)
- You need to get all detected signals for the entire video at once

Do NOT use this skill for:
- Live video streams (use `interhuman-stream` instead)
- Real-time analysis of ongoing video feeds

## Required Inputs

1. **API Access Token**: Bearer token obtained from the `interhuman-authentication` skill (use `interhumanai.upload` scope)
2. **Video File**: Binary video file to analyze
   - Size: 10 KB minimum, 32 MB maximum
   - Formats: mp4, avi, mov, mkv, mpeg-ts, mpeg-2-ts, webm

## Authentication

Before using this skill, you must obtain an access token using the `interhuman-authentication` skill with the `interhumanai.upload` scope. Use the returned `access_token` in the `Authorization` header as `Bearer <access_token>`.

## API Call Instructions

### Endpoint Details

- **Base URL**: `https://api.interhuman.ai`
- **Endpoint**: `/v0/upload/analyze`
- **Method**: POST
- **Content-Type**: `multipart/form-data`
- **Authentication**: Bearer token in `Authorization` header

### Request Format

Send the video file as `multipart/form-data` with a single field named `file` containing the binary video data.

### Example: cURL

```bash
curl -X POST https://api.interhuman.ai/v0/upload/analyze \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -F "file=@/path/to/video.mp4;type=video/mp4"
```

### Example: Python

```python
import requests

access_token = "YOUR_ACCESS_TOKEN"
video_path = "/path/to/video.mp4"

with open(video_path, "rb") as f:
    files = {"file": (os.path.basename(video_path), f, "video/mp4")}
    response = requests.post(
        "https://api.interhuman.ai/v0/upload/analyze",
        headers={"Authorization": f"Bearer {access_token}"},
        files=files,
        timeout=300,
    )

# Return the raw JSON response
print(response.json())
```

### Example: JavaScript/Node.js

```javascript
import fs from "fs";
import FormData from "form-data";
import fetch from "node-fetch";

const formData = new FormData();
formData.append("file", fs.createReadStream("path/to/video.mp4"));

const response = await fetch("https://api.interhuman.ai/v0/upload/analyze", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.ACCESS_TOKEN}`
  },
  body: formData
});

const json = await response.json();
console.log(json);
```

## Response Format

The API returns a JSON object with a `signals` array. Each signal object contains:

- **type** (string): The detected social signal type. Possible values: `agreement`, `confidence`, `confusion`, `disagreement`, `disengagement`, `engagement`, `frustration`, `hesitation`, `interest`, `skepticism`, `stress`, `uncertainty`
- **start** (number): Start time of the signal in seconds relative to video start
- **end** (number): End time of the signal in seconds relative to video start

### Example Response

```json
{
  "signals": [
    {
      "type": "agreement",
      "start": 2.5,
      "end": 8.2
    },
    {
      "type": "uncertainty",
      "start": 12.3,
      "end": 19.1
    }
  ]
}
```

## Error Responses

On error, the API returns JSON with:

- **detail** (string): Error message
- **status_code** (integer): HTTP status code
- **extra** (object): Additional error information

### Status Codes

- `200`: Success
- `400`: Bad request (invalid file format or parameters)
- `401`: Unauthorized (missing or invalid token)
- `403`: Forbidden (token lacks required scope)
- `413`: Payload too large (file exceeds 32 MB)
- `422`: Unprocessable entity (file missing or invalid)
- `500`: Internal server error

## Output Rules

**CRITICAL**: This skill is a strict wrapper. You MUST:

1. Return the exact JSON response from the API without any modification
2. Do NOT summarize, transform, or rename fields
3. Do NOT extract or filter signals
4. Do NOT add commentary or interpretation
5. Preserve all fields exactly as received from the API

The response should be the raw JSON object returned by the API, passed through verbatim.
