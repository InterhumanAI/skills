---
name: interhuman-post-processing
description: Analyze a pre-recorded video file with the Interhuman API V2 upload job (POST /v2/upload/analyze, then GET /v2/upload/jobs/{job_id}), preferably through the official TypeScript (@interhumanai/sdk) or Python (interhumanai) SDK. Use when the user wants social signals, engagement, or conversation quality for a complete recording. Returns the API's job envelope JSON without modification.
---

# Interhuman Post-Processing Analysis (V2 upload)

Analyze a complete recording with the Inter-2 model. On V2, uploading a file creates an asynchronous **job**. The submit returns a job envelope, and you read the result from the job when it reaches `completed` or `failed`.

- **Recommended endpoints**: `POST /v2/upload/analyze` (submit) and `GET /v2/upload/jobs/{job_id}` (read the job)
- **Recommended client**: the official SDK for the user's environment. Use raw HTTP only as a fallback (see [Raw HTTP fallback](#raw-http-fallback)).

## When to Use

Use this skill when:
- The video is already recorded and complete (a file on disk, an upload from a user, a `Blob`)
- You need signals, per-window engagement, or the Conversation Quality Index for the whole file

Do NOT use this skill for:
- Live or ongoing video. Use **interhuman-stream-analyze** (`WS /v2/stream/analyze`) instead.

## Choose the Client

Pick the SDK that matches the project's language and runtime:

| Environment | Use | Install |
|-------------|-----|---------|
| TypeScript / JavaScript (Node 18+, or a browser with a client token) | [`@interhumanai/sdk`](https://docs.interhuman.ai/sdk-reference/typescript-sdk) | `npm install @interhumanai/sdk` |
| Python 3.10+ | [`interhumanai`](https://docs.interhuman.ai/sdk-reference/python-sdk) | `pip install interhumanai` |
| Any other language, a shell script, or an environment where neither SDK can be installed | [Raw HTTP fallback](#raw-http-fallback) | none |

The V2 upload methods need **SDK 1.3.0 or later** for `include: signals`. Both SDKs are versioned together.

Do not use the SDK's `upload.analyze()`. It calls the V1 endpoint (`POST /v1/upload/analyze`) and is not a V2 call. For V2, use `upload.submit()` and then read the job with `waitForJob()` / `getJob()` in TypeScript, or `wait_for_job()` / `get_job()` in Python.

## Credentials

- Keep the API key server-side, for example in `INTERHUMAN_API_KEY`. Pass it to the SDK as `accessToken` (TypeScript) or `access_token` (Python). Over raw HTTP, send `Authorization: Bearer <api_key>`.
- The credential needs the **`interhumanai.upload`** scope. The same scope covers submitting a job and reading it back.
- **Never put the API key in browser code.** For a browser upload, mint a short-lived client token on your server with [`POST /v1/client_tokens`](https://docs.interhuman.ai/api-reference/client-tokens) and `scopes: ["interhumanai.upload"]`. Hand only the token to the browser, and pass it as `accessToken`. The token's `max_video_seconds` budget covers both upload and stream.

See [Authentication](https://docs.interhuman.ai/api-reference/authentication) for details.

## Input Requirements

With `model: inter-2`, the file must meet all of the following:

- **Container**: mp4, mov, avi, mkv, webm, or mpeg-ts
- **Tracks**: both a video track and an audio track. A file without video is rejected with `ih5001`, and a file without audio with `ih5008`.
- **Duration**: at least 3 seconds (`ih4007`) and at most 30 minutes, the deployment default (`ih4004`)
- **Size**: at most 32 MB (`413`, `ih4003`)
- **Content**: real picture and real sound. A black screen or a silent track passes validation but gives meaningless results.

Validation runs at submit, so these problems fail the submit request itself with an HTTP error, not the job.

## Request Fields

`POST /v2/upload/analyze` takes `multipart/form-data` with these fields:

| Field | Required | Values |
|-------|----------|--------|
| `file` | yes | The media file |
| `model` | yes | `inter-2`, which reads picture and sound together. An unknown value is rejected with `ih4005`. |
| `wait_seconds` | no | `0`–`120`, default `0`. Holds the request open for up to this long waiting for the job to finish. A value above the bound is rejected with `ih4005`. |
| `include[]` | no, repeatable | `signals`, `conversation_quality_overall`, `conversation_quality_timeline` |

Here is what each `include[]` flag adds to `job.result`:
- `signals` adds `result.signals`: every window's signals in one chronological list, with same-type signals in adjacent windows merged into one span (the V1 whole-file list, without `modality`). It is an empty list when nothing was detected.
- `conversation_quality_overall` adds `result.conversation_quality.overall`.
- `conversation_quality_timeline` adds `result.conversation_quality.timeline`.

Each section is absent unless its flag was sent.

## Job Lifecycle

1. **Submit.** The response is a job envelope:
   - `202 Accepted`: the job is pending (`queued` or `running`). Poll it.
   - `200 OK`: only when `wait_seconds > 0` and the job reached a terminal state in time. **Terminal does not mean success**: the job can be `completed` (with `result`) or `failed` (with `error`).
2. **Poll** `GET /v2/upload/jobs/{job_id}` until `status` is `completed` or `failed`. `status_url` in the envelope is a path relative to `https://api.interhuman.ai`, not an absolute URL. The status moves `queued` → `running` → `completed` | `failed`.
3. **Bound the wait.** Set a timeout (for example 10 minutes) and a poll interval (for example 2 seconds). Keep the polling path even when you send `wait_seconds`, because the submit can still return `202`.
4. **Check `status` before reading `result`.** A failed job carries `error` (`error_id`, `message`, and optionally `correlation_id` and `link`). `ih1004` means the job was interrupted, so submit the file again.
5. **Store the result yourself.** The job is readable until `expires_at`, 1 hour after submission. After that, or for an id that does not belong to the account, the read answers `404` with `ih4021`.
6. **Retry on 503.** `ih1002` (the instance holds too many jobs) and `ih1003` (the job store or model backend is unavailable) are transient, so retry after a short delay.

## Example: TypeScript SDK (recommended for JS/TS)

```typescript
import { readFile } from "node:fs/promises";
import { basename } from "node:path";
import {
  InterhumanClient,
  InterhumanApiError,
  UploadJobIncludeFlag,
  UploadJobTimeoutError,
  UploadModel,
} from "@interhumanai/sdk";

const client = new InterhumanClient({
  accessToken: process.env.INTERHUMAN_API_KEY!, // server-side only; a client token in browsers
});

const path = "/path/to/video.mp4";

try {
  // POST /v2/upload/analyze -> job envelope (queued, or terminal if waitSeconds sufficed)
  const submitted = await client.upload.submit({
    file: { data: await readFile(path), filename: basename(path), contentType: "video/mp4" },
    model: UploadModel.Inter2, // "inter-2"
    waitSeconds: 30, // optional; 0 answers at once
    include: [
      UploadJobIncludeFlag.Signals, // "signals"
      UploadJobIncludeFlag.ConversationQualityOverall, // "conversation_quality_overall"
    ],
  });

  // GET /v2/upload/jobs/{job_id} until completed or failed. A failed job is returned, not thrown.
  const job = await client.upload.waitForJob(submitted, {
    timeoutMs: 10 * 60 * 1000,
    pollIntervalMs: 2000,
  });

  // The SDK returns the parsed API JSON unchanged, so this prints the job envelope verbatim.
  console.log(JSON.stringify(job));
  if (job.status === "failed") process.exitCode = 1; // job.error has error_id / message
} catch (err) {
  if (err instanceof UploadJobTimeoutError) {
    // Still pending after timeoutMs. The job keeps running; err.job is the last envelope read.
    console.log(JSON.stringify(err.job));
  } else if (err instanceof InterhumanApiError) {
    // The submit or a read was rejected: err.status, err.errorId, err.body (the API error JSON).
    console.log(JSON.stringify(err.body ?? { status: err.status, message: err.message }));
  } else {
    throw err;
  }
  process.exitCode = 1;
}
```

- In a browser, pass the `File` or `Blob` itself as `file`, and build the client with `accessToken: clientToken`.
- `client.upload.getJob(jobId)` reads a job once, which is useful when the job id was stored and you resume later.

## Example: Python SDK (recommended for Python)

```python
import asyncio
import json
import os

from interhumanai import (
    InterhumanAPIError,
    InterhumanClient,
    UploadJobIncludeFlag,
    UploadJobTimeoutError,
    UploadModel,
)


async def main() -> None:
    client = InterhumanClient(access_token=os.environ["INTERHUMAN_API_KEY"])

    try:
        # POST /v2/upload/analyze -> job envelope
        submitted = await client.upload.submit(
            "/path/to/video.mp4",
            model=UploadModel.INTER_2,  # "inter-2"
            wait_seconds=30,  # optional; 0 answers at once
            include=[
                UploadJobIncludeFlag.SIGNALS,
                UploadJobIncludeFlag.CONVERSATION_QUALITY_OVERALL,
            ],
            content_type="video/mp4",
        )
        # GET /v2/upload/jobs/{job_id} until completed or failed. A failed job is returned, not raised.
        job = await client.upload.wait_for_job(submitted, timeout=600, poll_interval=2)
    except UploadJobTimeoutError as err:
        # Still pending after `timeout`. The job keeps running; err.job is the last envelope read.
        print(json.dumps(err.job.model_dump(mode="json", exclude_unset=True)))
        raise SystemExit(1)
    except InterhumanAPIError as err:
        # The submit or a read was rejected: err.status, err.error_id, err.body (the API error JSON).
        print(json.dumps(err.body or {"status": err.status, "message": str(err)}))
        raise SystemExit(1)

    # SDK objects are pydantic models. exclude_unset=True keeps exactly the fields the API sent.
    print(json.dumps(job.model_dump(mode="json", exclude_unset=True)))
    if job.status.value == "failed":  # job.error has error_id / message
        raise SystemExit(1)


asyncio.run(main())
```

- `file` also accepts raw `bytes` or an open binary file object.
- `await client.upload.get_job(job_id)` reads a job once.

## Raw HTTP Fallback

Use raw HTTP only when neither SDK fits: another language, a shell script, an environment where you cannot install packages, or when the user needs the exact response bytes. Implement the same lifecycle yourself: submit, poll with a bound, check `status`, and handle `404`/`ih4021` and `503`.

```bash
# Submit. Answers 202 with the job envelope (or 200 if wait_seconds sufficed).
curl -sS -X POST https://api.interhuman.ai/v2/upload/analyze \
  -H "Authorization: Bearer $INTERHUMAN_API_KEY" \
  -F "file=@/path/to/video.mp4;type=video/mp4" \
  -F "model=inter-2" \
  -F "include[]=signals" \
  -F "include[]=conversation_quality_overall"

# Read the job until status is completed or failed (JOB_ID = job_id from the submit).
curl -sS https://api.interhuman.ai/v2/upload/jobs/$JOB_ID \
  -H "Authorization: Bearer $INTERHUMAN_API_KEY"
```

```python
import os
import time

import requests

API = "https://api.interhuman.ai"
headers = {"Authorization": f"Bearer {os.environ['INTERHUMAN_API_KEY']}"}

with open("/path/to/video.mp4", "rb") as f:
    response = requests.post(
        f"{API}/v2/upload/analyze",
        headers=headers,
        files={"file": ("video.mp4", f, "video/mp4")},
        data={"model": "inter-2", "include[]": ["signals", "conversation_quality_overall"]},
        timeout=300,
    )
response.raise_for_status()  # 202 pending, or 200 terminal
job = response.json()

deadline = time.monotonic() + 600
while job["status"] not in ("completed", "failed"):
    if time.monotonic() > deadline:
        raise TimeoutError(f"job {job['job_id']} still {job['status']}")
    time.sleep(2)
    response = requests.get(f"{API}{job['status_url']}", headers=headers, timeout=30)
    if response.status_code == 503:
        continue  # ih1002 / ih1003: transient, retry
    response.raise_for_status()  # 404 ih4021: unknown or expired job
    job = response.json()

print(response.text)  # the terminal job envelope, verbatim
```

## Response Format

Both routes return the same **job envelope**:

| Field | Always | Description |
|-------|--------|-------------|
| `job_id` | yes | 32-character hex id |
| `status` | yes | `queued`, `running`, `completed`, `failed` |
| `model` | yes | The model submitted, e.g. `inter-2` |
| `created_at`, `expires_at` | yes | ISO 8601 UTC timestamps |
| `status_url` | yes | `/v2/upload/jobs/{job_id}`, relative to the API base URL |
| `result` | when `completed` | The analysis (below) |
| `error` | when `failed` | Standard error body: `error_id`, `message`, `correlation_id`, `link` |

Fields that do not apply are omitted, not sent as `null`.

`result` contains:
- `duration_seconds`, `window_seconds`
- `windows[]`, one entry per fixed-length window in file order. Each has `index`, `start_seconds`, `end_seconds`, `engagement_status` (`engaged`, `neutral`, or `disengaged`), and `signals[]`. A window signal's `start`/`end` is the window's span, so a signal that lasts several windows appears in each of them.
- `signals[]`, only with `include[]=signals`. This is the merged whole-file list. Do not rebuild it by concatenating `windows[].signals[]`, because that gives one entry per window instead of merged spans.
- `conversation_quality`, only with a CQI flag. It has `overall` and/or `timeline`: `quality_index`, `clarity`, `authority`, `energy`, `rapport`, `learning`, each 0–100.

Each signal has these fields, and **no `modality`**:
- `type`: `agreement`, `confidence`, `confusion`, `disagreement`, `frustration`, `hesitation`, `interest`, `skepticism`, `stress`, or `uncertainty`. These are the published types. The SDK `SignalType` unions may list extra values; see [Social signals](https://docs.interhuman.ai/explanations/social-signals).
- `start`, `end`: seconds from the start of the file
- `probability`: `high`, `medium`, or `low`. Omitted when the model gave none.
- `rationale`: a short evidence-based explanation. Omitted when the model gave none.

### Example: completed job

```json
{
  "job_id": "3f1c2b7a9d4e4c8fa1b2c3d4e5f60718",
  "status": "completed",
  "model": "inter-2",
  "created_at": "2026-09-11T10:00:00Z",
  "expires_at": "2026-09-11T11:00:00Z",
  "status_url": "/v2/upload/jobs/3f1c2b7a9d4e4c8fa1b2c3d4e5f60718",
  "result": {
    "duration_seconds": 10.0,
    "window_seconds": 5.0,
    "windows": [
      {
        "index": 0,
        "start_seconds": 0.0,
        "end_seconds": 5.0,
        "engagement_status": "engaged",
        "signals": [
          {
            "type": "confidence",
            "start": 0.0,
            "end": 5.0,
            "probability": "high",
            "rationale": "Steady eye contact, upright posture and a firm, even tone throughout."
          }
        ]
      },
      {
        "index": 1,
        "start_seconds": 5.0,
        "end_seconds": 10.0,
        "engagement_status": "neutral",
        "signals": []
      }
    ],
    "signals": [
      {
        "type": "confidence",
        "start": 0.0,
        "end": 5.0,
        "probability": "high",
        "rationale": "Steady eye contact, upright posture and a firm, even tone throughout."
      }
    ]
  }
}
```

### Example: failed job

```json
{
  "job_id": "3f1c2b7a9d4e4c8fa1b2c3d4e5f60718",
  "status": "failed",
  "model": "inter-2",
  "created_at": "2026-09-11T10:00:00Z",
  "expires_at": "2026-09-11T11:00:00Z",
  "status_url": "/v2/upload/jobs/3f1c2b7a9d4e4c8fa1b2c3d4e5f60718",
  "error": {
    "error_id": "ih5002",
    "correlation_id": "550e8400-e29b-41d4-a716-446655440000",
    "link": "https://docs.interhuman.ai/api-reference/error-handling#ih5002-corrupted-video",
    "message": "Unable to process the video: the file is corrupted or cannot be read."
  }
}
```

## Error Responses

HTTP errors return JSON with `error_id`, `message`, `correlation_id`, and `link`.

| Status | Codes | Meaning |
|--------|-------|---------|
| `200` | — | Terminal job within `wait_seconds`. Check `status`. |
| `202` | — | Job accepted and pending. Poll it. |
| `400` | `ih4001`, `ih4002`, `ih4004`, `ih4005`, `ih4007` | Missing file, unsupported container, too long, invalid `model`/`wait_seconds`, too short |
| `401` | `ih2001`, `ih2002`, `ih2010`, `ih2011`, `ih2014` | Invalid or missing API key, or an expired, invalid, or revoked client token |
| `403` | `ih2003` | Credential lacks `interhumanai.upload` |
| `404` | `ih4021` | Job unknown to this account, or expired |
| `413` | `ih4003` | File over 32 MB |
| `422` | `ih5001`, `ih5002`, `ih5008` | No video stream, unreadable container, or no audio stream |
| `429` | `ih3002`, `ih3003`, `ih3006` | Concurrency limit, usage quota, or client-token video budget |
| `503` | `ih1002`, `ih1003` | Temporarily unable to take or read jobs. Retry shortly. |

The SDKs raise these as `InterhumanApiError` (TypeScript) or `InterhumanAPIError` (Python), with the API body attached. A job that fails *after* it was accepted is not an HTTP error: it is a `failed` envelope. See [Error handling](https://docs.interhuman.ai/api-reference/error-handling).

## Output Rules

**CRITICAL**: This skill is a strict wrapper. You MUST:

1. Return the job envelope JSON exactly as the API returned it: the terminal envelope, or the last envelope read if the wait timed out. For an HTTP error, return the API error body.
2. Not summarize, transform, rename, filter, or reorder fields, windows, or signals.
3. Not add fields the API did not send, such as a V1-style `modality` or `null` placeholders.
4. Not add commentary or interpretation.
5. Serialize SDK objects faithfully:
   - **TypeScript**: the SDK returns the parsed API JSON unchanged, so use `JSON.stringify(job)`.
   - **Python**: the SDK returns pydantic models, so use `job.model_dump(mode="json", exclude_unset=True)`. `exclude_unset=True` drops SDK defaults the API never sent. A plain `model_dump()` would add `null` fields. Enum values become their strings and timestamps are re-rendered as ISO 8601. If the user needs the exact bytes, use the raw HTTP fallback and return the response body.

## V1 Compatibility

`POST /v1/upload/analyze` (Inter-1, synchronous) and the SDK's `upload.analyze()` remain available only for existing integrations. They are not the recommended path. V1 answers with the analysis in the response (`signals`, `engagement_state`, and signals carrying `modality`). V2 answers with a job and a windowed result. To move an integration, follow [Migrate from V1 to V2](https://docs.interhuman.ai/api-reference/migrate-v1-to-v2).

## References

- [Inter-2 Upload Analyze](https://docs.interhuman.ai/api-reference/upload-analyze-v2)
- [Migrate from V1 to V2](https://docs.interhuman.ai/api-reference/migrate-v1-to-v2)
- [TypeScript SDK](https://docs.interhuman.ai/sdk-reference/typescript-sdk) and [Python SDK](https://docs.interhuman.ai/sdk-reference/python-sdk)
- [Authentication](https://docs.interhuman.ai/api-reference/authentication) and [Client Tokens](https://docs.interhuman.ai/api-reference/client-tokens)
- [Error handling](https://docs.interhuman.ai/api-reference/error-handling)
