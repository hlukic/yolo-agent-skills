# Skills

One folder per skill or playbook.

## Recommended layout

```text
skills/<skill-name>/
  SKILL.md          required, frontmatter + body
  README.md         optional, for tool-specific notes
  examples/         optional, sanitized inputs/outputs
  agents/           optional, adapter files for specific platforms
```

## Available skills

| Skill | Summary |
|---|---|
| [`seo-aeo-readiness`](seo-aeo-readiness/) | Audit and improve a site's SEO, AEO/GEO readiness, JSON-LD, social previews, crawler visibility, and machine-readable files, with bounded run-control and a final-report template. |

## Rule for every public skill

Every skill in this folder must be:

- **Public-safe** — no secrets, tokens, cookies, deploy hooks, or webhook signing keys
- **Generic** — uses placeholders such as `https://example.com`, `$SITE_URL`,
  `$FB_APP_ID`, `$VERCEL_TOKEN` instead of real values
- **Sanitized** — no private client URLs, real project IDs (`prj_…`, `team_…`,
  `app_…`), internal repo paths (`/root/projekti/...`, `/Users/.../private/...`),
  or named customers
- **Self-contained** — the `SKILL.md` body should be readable and runnable
  without depending on a private repo, internal wiki, or hidden context
- **Honest** — no fabricated review counts, certifications, or schema markup
  that does not match the rendered page

Before opening a PR, scan the diff for these patterns and replace anything
sensitive with a placeholder. When in doubt, leave it out.
