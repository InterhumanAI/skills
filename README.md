# Interhuman Agent Skills

Agent Skills for the [Interhuman API](https://docs.interhuman.ai) — authentication, video upload analysis, and real-time streaming analysis.

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
| **interhuman-authentication** | Generate short-lived bearer tokens via `POST /v0/auth`. Use first; required by the other two skills. |
| **interhuman-post-processing** | Analyze pre-recorded video files via `POST /v0/upload/analyze`. Returns detected social signals as raw JSON. |
| **interhuman-stream** | Real-time analysis of live video via `ws /v0/stream/analyze`. Relays server messages as received. |

All skills are strict API wrappers: they return raw JSON from the Interhuman API without modification.

## Related

- [Interhuman API docs](https://docs.interhuman.ai)
- [Agent Skills specification](https://agentskills.io)
- [Skills directory](https://skills.sh)