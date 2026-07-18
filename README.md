# Hermes Skills

A public collection of reusable skill definitions for the Hermes Agent (and other agentic systems). Each skill lives in its own folder under `skills/` and contains a `SKILL.md` with frontmatter + instructions.

## Skills

| Skill | Description |
|-------|-------------|
| `n8n-self-host` | Deploy a self-hosted, auto-updating n8n instance via Docker Compose, behind nginx + Let's Encrypt, with a systemd timer for daily image updates. Covers real pitfalls (volume uid 1000, invalid N8N_EMAIL_MODE, broken watchtower on modern Docker API). |

## How to use

Copy a skill folder into your agent's skills directory (e.g. `~/.hermes/skills/<category>/<name>/`). The agent loads `SKILL.md` on demand. All values in skill files use `{{template}}` placeholders — substitute before running.

## Contributing

Add a folder under `skills/<category>/<name>/` with a `SKILL.md`. Keep secrets out; use `{{placeholders}}`.
