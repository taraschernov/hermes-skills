# Contributing a skill

A skill is a folder under `skills/<skill-name>/` with at least `SKILL.md` and `prompt.md`.

## Minimum structure
```
skills/<skill-name>/
├── SKILL.md          # Hermes/Claude format: YAML frontmatter + markdown steps
├── prompt.md         # portable chat prompt (self-contained, no plugin required)
└── references/       # optional: drop-in configs (compose, nginx, systemd, env)
```

## SKILL.md format
```markdown
---
name: <skill-name>
description: One sentence. When to use it + what it deploys. Include trigger phrases.
---

# <Skill title>

## When to use
- bullet list of triggers

## Assumptions / inputs needed
- list

## Step-by-step (verified working)
### 1. ...
### 2. ...

## Verification checklist
- how to confirm success

## Common errors → fixes
- real failure -> real fix
```

## prompt.md format
A complete system prompt any chat agent can run. Start with a role line, list `{{variables}}`, then numbered hard requirements. Reference `./references/` files at the end. No frontmatter.

## Rules
1. No real secrets, IPs, emails, or domain names. Use `{{domain}}`, `{{SERVER_IP}}`, `{{email}}`, `{{HOME}}`, `{{TIMEZONE}}`.
2. Every skill must include at least one "lesson learned" (a real pitfall it avoids).
3. Prefer OS-native mechanics over extra containers when more reliable.
4. Keep `references/` files valid templates (quote paths containing `{{...}}` in YAML).
5. Add a row to the skills index table in the root README.

## Add to the index
Edit `README.md` → "Skills index" table, add a column entry linking `SKILL.md`, `prompt.md`, and the article (if any).

## Commit & push
```bash
git add -A
git commit -m "Add skill: <skill-name>"
git push
```
