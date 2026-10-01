# Interhuman Agent Skills

Agent Skills for the [Interhuman API](https://docs.interhuman.ai) — V2 video upload and real-time stream analysis, using the official TypeScript and Python SDKs.

Compatible with **Cursor**, **Claude Code**, **Codex**, **OpenCode**, and other agents.

## Install a Skill

```bash
npx skills add InterhumanAI/skills
```

Install directly using the InterhumanAI organization name.

### Source formats

```bash
# GitHub shorthand (owner/repo)
npx skills add InterhumanAI/skills

# Full GitHub URL
npx skills add https://github.com/InterhumanAI/skills

# Direct path to a skill
npx skills add https://github.com/InterhumanAI/skills/tree/main/skills/interhuman-post-processing
```

### Options

| Option | Description |
|--------|-------------|
| `-g, --global` | Install to user directory instead of project |
| `-s, --skill <names>` | Install specific skills (use `'*'` for all) |
| `-l, --list` | List available skills without installing |
| `-y, --yes` | Skip confirmation prompts |
| `--all` | Install all skills to all detected agents |

### Examples

```bash
# List skills in this repository
npx skills add InterhumanAI/skills --list

# Install specific skills
npx skills add InterhumanAI/skills --skill interhuman-post-processing
npx skills add InterhumanAI/skills --skill interhuman-stream-analyze

# Install all skills
npx skills add InterhumanAI/skills --skill '*'

# Install to Cursor only (global)
npx skills add InterhumanAI/skills -g -a cursor -y
```

## Skills in this repo

| Skill | Description |
|-------|-------------|
| **interhuman-post-processing** | Analyze a recorded video file as a V2 upload job: `POST /v2/upload/analyze`, then `GET /v2/upload/jobs/{job_id}` until `completed` or `failed`. Returns the job envelope JSON unmodified. |
| **interhuman-stream-analyze** | Analyze live or chunked video over the V2 WebSocket `wss://api.interhuman.ai/v2/stream/analyze`. Returns each event envelope (`session.ready`, `signal.detected`, `engagement.updated`, `session.ended`, ...) unmodified. |

Both skills target the **V2 (Inter-2) endpoints** and cover only upload and stream.

### SDK first

The skills use the official SDKs wherever they support the workflow and fall back to raw HTTP/WebSocket only when neither SDK fits:

| Language | Package | Upload | Stream |
|----------|---------|--------|--------|
| TypeScript / JavaScript | [`@interhumanai/sdk`](https://docs.interhuman.ai/sdk-reference/typescript-sdk) (1.3.0+) | `client.upload.submit()`, then `waitForJob()` / `getJob()` | `client.stream({ apiVersion: "v2" })` |
| Python 3.10+ | [`interhumanai`](https://docs.interhuman.ai/sdk-reference/python-sdk) (1.3.0+) | `client.upload.submit()`, then `wait_for_job()` / `get_job()` | `client.stream(api_version="v2")` |

Both SDKs still default the stream to V1, so the skills always select V2 explicitly. `upload.analyze()` is the V1 call and is not used for V2.

### Credentials

- Server-side: pass your API key to the SDK (`accessToken` / `access_token`), or send `Authorization: Bearer <api_key>` over raw HTTP.
- Scopes: `interhumanai.upload` for upload and job reads, and `interhumanai.stream` for stream.
- Browsers: never ship the API key. Mint a short-lived client token server-side with [`POST /v1/client_tokens`](https://docs.interhuman.ai/api-reference/client-tokens) and use it instead. Browser WebSockets send it as the subprotocol pair `access_token, <client_token>`.

See [Authentication](https://docs.interhuman.ai/api-reference/authentication).

### V2 upload

- `multipart/form-data` with `file` and `model=inter-2` (required), plus optional `wait_seconds` (0–120) and `include[]` (`signals`, `conversation_quality_overall`, `conversation_quality_timeline`).
- The file must carry both video and audio, run 3 s–30 min, and be at most 32 MB.
- The submit answers `202` with a pending job, or `200` with a terminal one when `wait_seconds` sufficed. A terminal job is either `completed` (with `result`) or `failed` (with `error`).
- `result` has `duration_seconds`, `window_seconds`, and per-window `windows[]` (`engagement_status`, `signals[]`), plus `signals` and `conversation_quality` when requested. V2 signals carry no `modality`.
- Jobs are readable for 1 hour (`expires_at`). After that, reads answer `404` with `ih4021`.

### V2 stream

- Wait for `session.ready`. Optionally send a JSON config (`include`, `model`) and get back `session.updated`.
- Send binary WebM or fragmented-MP4 chunks with video and audio, in recorder order.
- End with `{"type": "session.close"}`. The server answers `session.closing`, finishes the analysis, sends `session.ended`, and closes with code 1000.
- Events: `signal.detected` / `signal.updated` / `signal.ended`, `engagement.updated`, `conversation_quality.updated`, `coverage.degraded` / `coverage.dropped`, and `error`. Times are session-cumulative seconds, and signal payloads carry no `modality`.

All skills are strict API wrappers: they return the API's JSON without modification. Each skill documents how to serialize SDK objects faithfully.

### V1 compatibility

`POST /v1/upload/analyze` and `WS /v1/stream/analyze` (Inter-1) are still served for existing integrations but are not the recommended path. See [Migrate from V1 to V2](https://docs.interhuman.ai/api-reference/migrate-v1-to-v2).

## Related

- [Interhuman API docs](https://docs.interhuman.ai)
- [Inter-2 Upload Analyze](https://docs.interhuman.ai/api-reference/upload-analyze-v2) and [Inter-2 Streaming Analyze](https://docs.interhuman.ai/api-reference/stream-analyze-v2)
- [TypeScript SDK](https://docs.interhuman.ai/sdk-reference/typescript-sdk) and [Python SDK](https://docs.interhuman.ai/sdk-reference/python-sdk)
- [Agent Skills specification](https://agentskills.io)
- [Skills directory](https://skills.sh)
