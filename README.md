# Interhuman Agent Skills

Agent Skills for the [Interhuman API](https://docs.interhuman.ai) — video upload and real-time stream analysis.

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
| **interhuman-post-processing** | Analyze pre-recorded video files via `POST /v1/upload/analyze`. Returns raw JSON, including `signals` and optional quality fields. |
| **interhuman-stream-analyze** | Analyze live or chunked video via WebSocket `wss://api.interhuman.ai/v1/stream/analyze`. Returns verbatim v1 event envelopes (`signal.detected`, `engagement.updated`, etc.). |

Authentication for integrations: send your API key directly as `Authorization: Bearer <api_key>` for any endpoint.

All skills are strict API wrappers: they return raw JSON from the Interhuman API without modification.

## V1 upload features

- Upload endpoint accepts `multipart/form-data` with required `file`.
- Optional `include[]` values:
  - `conversation_quality_overall`
  - `conversation_quality_timeline`
- Typical response fields:
  - `signals` (always present)
  - `engagement_state` (always present)
  - `conversation_quality` (optional, when requested)

## V1 stream features

- WebSocket URL: `wss://api.interhuman.ai/v1/stream/analyze`
- Client sends binary video segments (min 3s, max 32 MB each) and optional JSON session config (`include`)
- Server emits typed envelopes: `signal.detected`, `engagement.updated`, `conversation_quality.updated`, `error`
- Auth: `Authorization: Bearer <api_key>` or `Sec-WebSocket-Protocol: <api_key>`
- Strict wrapper: skills return raw JSON from the API without modification

## Related

- [Interhuman API docs](https://docs.interhuman.ai)
- [Agent Skills specification](https://agentskills.io)
- [Skills directory](https://skills.sh)