# YOLO Agent Skills

Curated public collection of practical agent skills and playbooks for Codex,
Claude Code, OpenClaw, and similar coding-agent tools.

The goal is to share reusable operational know-how that other developers can
drop into their own agent setup, without exposing private client data,
credentials, internal infrastructure, or proprietary business material.

## Available skills

| Skill | What it does |
|---|---|
| [`seo-aeo-readiness`](skills/seo-aeo-readiness/) | Audit and improve a site's SEO, AEO/GEO readiness, structured data (JSON-LD), social-preview metadata, crawler visibility, and machine-readable files (`robots.txt`, `sitemap.xml`, `llms.txt`). Includes a bounded run-control protocol and a final-report template. |

More skills will be added over time. PRs welcome, see the publishing rule
below.

## How to use a skill

The skills here are plain Markdown files (`SKILL.md`) with a small YAML
frontmatter (`name`, `description`). They work with several tools, depending on
what you use:

- **Claude Code:** copy or symlink the skill folder into the agent's skills
  directory so the `Skill` tool can discover it. The frontmatter `name` and
  `description` drive triggering.
- **Codex / Codex CLI:** point at the skill folder or paste the `SKILL.md`
  contents into the project instructions if your version does not yet support
  skill folders natively.
- **OpenClaw or other agent frameworks:** paste the `SKILL.md` body into the
  system prompt or instructions section of the agent you are configuring.
- **Anything else:** treat `SKILL.md` as a self-contained playbook. The
  guardrails, checklists, and validation commands are written to be readable
  and runnable without any tool-specific harness.

If a tool does not understand skill frontmatter, the prose body is still
useful: it documents *when* to use the playbook, *what* to check, and *how* to
validate the result.

## What belongs here

- Generic agent skills and workflows
- Sanitized prompts that are useful outside one private project
- SEO/AEO, deployment, debugging, review, and automation playbooks
- Templates and checklists others can adapt safely

## What does not belong here

- API keys, tokens, secrets, cookies, or credentials
- Private customer data or lead lists
- Internal server IPs, hidden endpoints, or private repo paths
- Exact production configs that reveal sensitive architecture
- Content copied from private chats without sanitization
- Real `fb:app_id`, deploy hook URLs, Stripe/payment IDs, webhook secrets

## Safety rule

Every example here uses placeholders such as `https://example.com`,
`$SITE_URL`, `$FB_APP_ID`, or `$VERCEL_TOKEN`. Before contributing or copying
a skill into a private project, replace placeholders with real values, but
**never commit those real values back here**. Treat this repo as
"public-safe by default" and assume anything pushed is permanent.

## Structure

```text
skills/        Reusable skill folders, one per skill
examples/      Optional sanitized examples
LICENSE        MIT
```

## Publishing rule

Before opening a PR or pushing a new skill, run a secret scan over the diff
(e.g. `rg -n "ghp_|sk_live|sk_test|Bearer |team_|prj_|deploy_hook"` plus any
project-specific tokens you might leak) and replace any project-specific
details with placeholders. When in doubt, leave it out.

## License

[MIT](LICENSE).
