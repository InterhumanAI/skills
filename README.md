# Interhuman Agent Skills

Agent Skills for the [Interhuman API](https://docs.interhuman.ai) — authentication and video upload analysis.

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
npx skills add https://github.com/InterhumanAI/skills/tree/main/skills/interhuman-authentication
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
npx skills add InterhumanAI/skills --skill interhuman-authentication --skill interhuman-post-processing

# Install all skills
npx skills add InterhumanAI/skills --skill '*'

# Install to Cursor only (global)
npx skills add InterhumanAI/skills -g -a cursor -y
```

## Skills in this repo

| Skill | Description |
|-------|-------------|
| **interhuman-authentication** | Generate short-lived bearer tokens via `POST /v1/auth`. Use first; required before calling upload analysis. |
| **interhuman-post-processing** | Analyze pre-recorded video files via `POST /v1/upload/analyze`. Returns raw JSON, including `signals` and optional quality fields. |

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

## Related

- [Interhuman API docs](https://docs.interhuman.ai)
- [Agent Skills specification](https://agentskills.io)
- [Skills directory](https://skills.sh)