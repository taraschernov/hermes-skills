# Prompt: Publish a skill + article to the Hermes Skills collection

You are assisting with publishing reusable infrastructure content to a public GitHub collection.
Follow this procedure exactly. Do not invent repos, tokens, or paths.

## What to publish
Two artifacts (provided by the user, already written):
1. A **skill folder** — `skills/<skill-name>/` containing at least `SKILL.md` and `prompt.md`, plus optional `references/`.
2. An **article** (long-form walkthrough) — `articles/<skill-name>.md`.

Both MUST be template-driven: every environment-specific value is a `{{placeholder}}`
(`{{domain}}`, `{{SERVER_IP}}`, `{{email}}`, `{{HOME}}`, `{{TIMEZONE}}`).
NEVER commit real secrets, IPs, emails, or domain names. If the provided files contain
real values, STOP and tell the user to anonymize before publishing.

## Target repository
- Owner/repo: `taraschernov/hermes-skills` (public)
- Branch: `main`
- Structure convention (already established):
  ```
  hermes-skills/
  ├── README.md            # has a "Skills index" table — ADD A ROW for the new skill
  ├── CONTRIBUTING.md
  ├── skills/<skill-name>/{SKILL.md,prompt.md,references/}
  └── articles/<skill-name>.md
  ```

## Execution protocol
1. **Confirm access.** Test GitHub connectivity WITHOUT credentials first:
   `ssh -T git@github.com 2>&1 | head -3`
   - If it prints `Hi <user>! You've successfully authenticated` → proceed with SSH.
   - If `Permission denied (publickey)` → do NOT guess. Tell the user to either
     (a) add an SSH key (`ssh-keygen -t ed25519`, paste `*.pub` to github.com/settings/keys), or
     (b) provide a GitHub PAT (repo scope). Never ask for or store a password.
2. **Clone or pull** the collection:
   `git clone git@github.com:taraschernov/hermes-skills.git` (or `git pull` if already cloned).
   Work in a clean local copy.
3. **Place files:**
   - Copy the skill folder to `skills/<skill-name>/`.
   - Copy the article to `articles/<skill-name>.md`.
4. **Update the index.** Edit `README.md` → "Skills index" table; add one row:
   `| \`<skill-name>\` | <one-line what-it-deploys> | [SKILL.md](skills/<skill-name>/SKILL.md) · [prompt.md](skills/<skill-name>/prompt.md) | [article](articles/<skill-name>.md) |`
5. **Validate:**
   - YAML files under `references/` must quote any path containing `{{...}}`.
   - No real secrets/IPs/emails remain in any file (grep for them).
6. **Commit & push:**
   ```bash
   git add -A
   git commit -m "Add skill + article: <skill-name>"
   git push origin main
   ```
7. **Report** the public URL of the new skill folder and article.

## If the repo does not exist yet
Tell the user to create an EMPTY public repo named `hermes-skills` under `taraschernov`,
then re-run step 2. Do not attempt to create repos without a PAT.

## Guardrails
- Public repo: nothing sensitive may be pushed. When in doubt, anonymize.
- Do not modify other skills' folders.
- If anything is ambiguous (missing files, unclear skill name), ask the user — do not guess.
