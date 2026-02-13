# Interhuman Agent Skills

Agent Skills for the [Interhuman API](https://docs.interhuman.ai)—authentication, video upload analysis, and real-time streaming analysis.

Compatible with **Cursor**, **Claude Code**, **Codex**, **OpenCode**, and [other agents supported by the skills CLI](https://github.com/vercel-labs/skills#supported-agents).

## Install a Skill

```bash
npx skills add <owner>/interhuman-agent-skills
```

Replace `<owner>` with your GitHub username or organization (e.g. `interhuman/interhuman-agent-skills`).

### Source formats

```bash
# GitHub shorthand (owner/repo)
npx skills add <owner>/interhuman-agent-skills

# Full GitHub URL
npx skills add https://github.com/<owner>/interhuman-agent-skills

# Direct path to a skill
npx skills add https://github.com/<owner>/interhuman-agent-skills/tree/main/skills/interhuman-authentication
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
npx skills add <owner>/interhuman-agent-skills --list

# Install specific skills
npx skills add <owner>/interhuman-agent-skills --skill interhuman-authentication --skill interhuman-post-processing

# Install all skills
npx skills add <owner>/interhuman-agent-skills --skill '*'

# Install to Cursor only (global)
npx skills add <owner>/interhuman-agent-skills -g -a cursor -y
```

## Skills in this repo

| Skill | Description |
|-------|-------------|
| **interhuman-authentication** | Generate short-lived bearer tokens via `POST /v0/auth`. Use first; required by the other two skills. |
| **interhuman-post-processing** | Analyze pre-recorded video files via `POST /v0/upload/analyze`. Returns detected social signals as raw JSON. |
| **interhuman-stream** | Real-time analysis of live video via WebSocket `/v0/stream/analyze`. Relays server messages as received. |

All skills are strict API wrappers: they return raw JSON from the Interhuman API without modification.

## Related

- [Agent Skills specification](https://agentskills.io)
- [Skills directory](https://skills.sh)
- [skills CLI (vercel-labs/skills)](https://github.com/vercel-labs/skills)
- [Interhuman API docs](https://docs.interhuman.ai)
