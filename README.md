# Hermes Skills

A public, vendor-neutral collection of reusable infrastructure skills. Each skill deploys a real, production-grade piece of self-hosted software — hardened, with auto-updates, and documented pitfalls.

All skills are:
- **Template-driven** — every concrete value is a `{{placeholder}}` (domain, IP, email, paths, timezone). Substitute before use.
- **Secret-free** — no real credentials, IPs, or emails are committed.
- **Portable** — works with agentic systems that support skills, with any chat-based AI agent, or manually by a human.

---

## How to use a skill

Pick the path that matches your setup.

### Path A — You use Hermes Agent (or a Claude-style skills system)
Copy the skill folder into your agent's skills directory:
```bash
# Hermes
cp -r skills/n8n-self-host ~/.hermes/skills/devops/n8n-self-host
```
The agent auto-loads `SKILL.md` when your request matches its `description`. No copy-paste needed.

### Path B — You use any chat-based AI agent (Claude, ChatGPT, Cursor, a coding assistant, etc.)
You do NOT need a skills plugin. Open `skills/<name>/prompt.md` and paste its contents into the agent's chat. It is a complete, self-contained deployment prompt. The agent will ask for your `{{variables}}` and execute the steps.

### Path C — You have no agent (manual deploy)
Read `articles/<name>.md` (when present) for a full walkthrough with explanations, or use the raw files in `skills/<name>/references/` as drop-in configs. Substitute `{{placeholders}}` and run the commands yourself.

---

## Skills index

| Skill | What it deploys | Agent | Manual |
|-------|-----------------|-------|--------|
| `n8n-self-host` | n8n workflow automation via Docker + nginx + Let's Encrypt, daily systemd auto-update | [SKILL.md](skills/n8n-self-host/SKILL.md) · [prompt.md](skills/n8n-self-host/prompt.md) | [article](articles/yapclean-n8n-self-host.md) |

---

## Repository layout

```
hermes-skills/
├── README.md              # this file
├── CONTRIBUTING.md        # how to add a skill
├── skills/
│   └── <skill-name>/
│       ├── SKILL.md       # Hermes/Claude-style skill (frontmatter + steps)
│       ├── prompt.md      # portable chat prompt for ANY agent (no plugin needed)
│       └── references/    # ready-to-adapt configs (compose, nginx, systemd, etc.)
└── articles/              # long-form walkthroughs (optional, for manual users)
```

## Principles
1. No secrets in git. Ever.
2. Template everything that is environment-specific.
3. Every skill must document at least one real failure it prevents (the "lesson learned").
4. Prefer native OS mechanics (systemd, UFW) over extra containers where they are more reliable.

## License
MIT — reuse freely, attribution appreciated.
