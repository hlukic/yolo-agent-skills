---
name: seo-aeo-readiness
description: Audit and improve a website's SEO, social-preview, and AEO/GEO readiness for Google, Bing, social-share crawlers (Facebook, LinkedIn, WhatsApp, Twitter/X), and AI search crawlers (GPTBot, ChatGPT-User, ClaudeBot, PerplexityBot, Google-Extended). Use when a user wants to verify or improve how a page renders for crawlers, AI engines, and social previews, or before launching content/landing pages. Triggers on phrasings like "SEO audit", "AI search visibility", "schema markup", "OG preview", "make page crawler-readable", "llms.txt", or when the user pastes a URL and asks for an SEO/AEO check.
---

# SEO + AEO/GEO Readiness Playbook

A bounded, evidence-driven workflow for auditing and improving the readiness of any website for:

- **Traditional search** — Google, Bing
- **Social-share crawlers** — `facebookexternalhit`, `LinkedInBot`, `WhatsApp`, `Twitterbot`
- **AI search crawlers** — `GPTBot`, `ChatGPT-User`, `OAI-SearchBot`, `ClaudeBot`, `Claude-SearchBot`, `PerplexityBot`, `Google-Extended`, `Meta-ExternalAgent`

This skill is opinionated about **evidence before assertions** and **bounded runs**. It does not promise rankings or instant indexing. It produces:

1. A reproducible audit table
2. A safe, sanitized set of code/config changes
3. Live verification commands and their expected output
4. A short manual handoff for steps only a human can perform (Search Console submissions, social debugger re-scrape, etc.)

---

## 1. When to use this skill

Use it when any of the following apply:

- The user asks for an SEO, AEO, GEO, or "AI search" audit of a public site
- The user pastes a URL and wants to know what is missing or broken
- A new content page or landing page will be published and needs to be crawl-correct from day one
- A social share preview looks wrong (missing image, stale title, wrong description) and needs to be debugged
- A site is supposed to appear in AI answers (ChatGPT, Claude, Perplexity, Gemini) but does not, and the user wants to understand why
- Schema markup (JSON-LD) needs to be added, validated, or extended

Do **not** use it for:

- Paid acquisition (Google Ads, Meta Ads)
- Conversion-rate or A/B testing work
- Manual content writing without an SEO/AEO frame
- E-commerce product feed work (use a dedicated commerce skill)

---

## 2. Safety and publication guardrails

Before touching anything, accept these rules:

1. **Never invent claims.** Do not write JSON-LD `Review`, `aggregateRating`, `Award`, customer counts, certifications, or testimonials unless they exist in the rendered page. Schema must be truthful — Google and AI engines penalize fabricated structured data.
2. **Never hard-code secrets.** Tokens, API keys, app IDs, webhook secrets, deploy hooks belong in environment variables (`$FB_APP_ID`, `$VERCEL_TOKEN`, `$INDEXNOW_KEY`, etc.), never in the repo or in this skill's output.
3. **Do not touch revenue-critical paths without a checkpoint.** Checkout, payment, auth callbacks, and webhook handlers are out of scope for an SEO run. If the user explicitly asks for SEO changes inside a checkout flow, stop, confirm, and isolate.
4. **No destructive operations without explicit consent.** No `git push --force`, no `--amend`, no force-rewriting `main`, no deleting projects, no schema migrations.
5. **Preserve existing work.** Never `git add -A` or `git add .` blindly. Use path-specific staging so unrelated work-in-progress is not pulled into the SEO commit.
6. **Do not exfiltrate.** Do not paste private repo contents, customer data, environment files, or proprietary copy into external services, tickets, or screenshots.
7. **No claim of completion without proof.** Manual steps (Search Console submit, social debugger re-scrape) are not "done" until the human confirms.

---

## 3. Baseline inventory (always step 1)

Before any change, collect and report this set of facts:

| Item | How to obtain |
|---|---|
| Site URL(s) | Ask user; canonical + apex + `www.` variants |
| Repo path | Ask user; verify with `git remote -v` |
| Default branch | `git branch --show-current` |
| Build command + output directory | Read `package.json` scripts; check `dist/`, `build/`, `.next/`, `public/` |
| Deploy target | Vercel, Netlify, Cloudflare Pages, self-hosted; check `*.json` config or ask |
| Custom domain status | `curl -sI https://$SITE_URL` and check for HTTPS + redirects |
| Existing CMS/framework | Vite, Next.js, Astro, WordPress, plain HTML — read repo or page source |
| Existing static files | `robots.txt`, `sitemap.xml`, `llms.txt`, `humans.txt`, `.well-known/security.txt`, `og-image.*` |
| Existing JSON-LD inventory | Parse `<script type="application/ld+json">` blocks from rendered HTML |

Write this as a small table at the start of the run. Do not skip — it prevents wrong-direction work.

---

## 4. Crawler matrix

Hit the root URL with each crawler user-agent and record the HTTP status. Anything other than `200` is a red flag.

```bash
SITE_URL="https://example.com"
BOTS=(
  "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)|Googlebot"
  "Mozilla/5.0 (compatible; Bingbot/2.0; +http://www.bing.com/bingbot.htm)|Bingbot"
  "facebookexternalhit/1.1 (+http://www.facebook.com/externalhit_uatext.php)|facebookexternalhit"
  "LinkedInBot/1.0 (compatible; Mozilla/5.0; +https://www.linkedin.com)|LinkedInBot"
  "WhatsApp/2.0|WhatsApp"
  "Twitterbot/1.0|Twitterbot"
  "Mozilla/5.0 AppleWebKit/537.36 (KHTML, like Gecko; compatible; GPTBot/1.0; +https://openai.com/gptbot)|GPTBot"
  "Mozilla/5.0 AppleWebKit/537.36 (KHTML, like Gecko; compatible; ChatGPT-User/1.0; +https://openai.com/bot)|ChatGPT-User"
  "Mozilla/5.0 (compatible; ClaudeBot/1.0; +claudebot@anthropic.com)|ClaudeBot"
  "Mozilla/5.0 AppleWebKit/537.36 (KHTML, like Gecko; compatible; PerplexityBot/1.0; +https://perplexity.ai/perplexitybot)|PerplexityBot"
)
for entry in "${BOTS[@]}"; do
  ua="${entry%|*}"
  name="${entry##*|}"
  code=$(curl -s -o /dev/null -A "$ua" --max-time 10 -w "%{http_code}" "$SITE_URL")
  printf "%-20s %s\n" "$name" "$code"
done
```

Interpret results:

- `200` everywhere → good
- `403` for AI bots → site or CDN is blocking AI crawlers, likely intentional, confirm with user before changing
- `200` for browser-like UAs but `404`/`5xx` for crawler UAs → bot-specific routing or firewall issue
- `301`/`302` → check redirect chain; crawlers tolerate short chains but long ones leak signal

---

## 5. SEO essentials checklist

Verify each item with a `curl` + parse. Do not trust assumptions.

| Check | How |
|---|---|
| HTTPS only (no mixed content) | `curl -sI http://$SITE_URL` returns 301 to `https://` |
| Title (50-60 chars, unique per page) | `grep -oP '<title>\K[^<]+'` |
| Meta description (140-160 chars, unique) | `grep -oP 'name="description" content="\K[^"]+'` |
| Single H1 per page, semantically meaningful | Parse rendered HTML |
| Heading order is not chaotic (H1 → H2 → H3, no skipped levels in main content) | Inspect manually |
| `<link rel="canonical">` is self-referencing on canonical URL | `grep -oP 'rel="canonical" href="\K[^"]+'` |
| `<meta name="robots">` does not block production indexing accidentally | Look for `noindex` |
| `sitemap.xml` returns 200, lists current public URLs, every `<loc>` returns 200 | `curl -s $SITE_URL/sitemap.xml \| grep -oP '<loc>\K[^<]+' \| xargs -I {} curl -sI -o /dev/null -w "%{http_code} {}\n" {}` |
| `robots.txt` returns 200, references the sitemap, does not block production accidentally | `curl -s $SITE_URL/robots.txt` |
| `favicon` is present and not 404 | `curl -sI $SITE_URL/favicon.ico` |
| `og-image` returns 200 with correct `image/*` Content-Type (not HTML fallback) | `curl -sI $SITE_URL/og-image.png` |
| No internal links point to 404s | Crawl with a basic spider or run `linkinator` / similar |
| URLs are stable and human-readable (no `?id=42&utm=...` for canonical pages) | Inspect site map |
| 404 page itself returns HTTP 404, not 200 | `curl -sI $SITE_URL/this-does-not-exist` |

---

## 6. Social preview metadata

The 12 required Open Graph + Twitter tags:

```html
<meta property="og:title"        content="..." />
<meta property="og:description"  content="..." />
<meta property="og:image"        content="https://$SITE_URL/og-image.png" />
<meta property="og:image:width"  content="1200" />
<meta property="og:image:height" content="630" />
<meta property="og:image:type"   content="image/png" />
<meta property="og:image:alt"    content="..." />
<meta property="og:url"          content="https://$SITE_URL/" />
<meta property="og:type"         content="website" />
<meta property="og:locale"       content="en_US" />
<meta property="og:site_name"    content="..." />

<meta name="twitter:card"        content="summary_large_image" />
<meta name="twitter:title"       content="..." />
<meta name="twitter:description" content="..." />
<meta name="twitter:image"       content="https://$SITE_URL/og-image.png" />
```

Optional but recommended where the project owns a Facebook App ID:

```html
<meta property="fb:app_id" content="$FB_APP_ID" />
```

**Rules:**

- `og:image` must be an absolute HTTPS URL, served with `Content-Type: image/png` or `image/jpeg`. Browsers tolerate other extensions; Facebook does not.
- Recommended size: `1200 × 630` (1.91:1 ratio). Under `200 × 200` is rejected.
- `og:url` should be the canonical URL of the page, not an internal staging URL.
- `fb:app_id` is **optional**. Do not hardcode a real App ID into a public skill. Read it from `$FB_APP_ID` env var or a project config file. If the project does not own a Facebook App, omit the tag entirely; do not invent one.
- Do not duplicate OG tags inside the body — they belong in `<head>`.

After deploy, force-rescrape:

- **Facebook:** https://developers.facebook.com/tools/debug/ → paste URL → **Scrape Again**
- **LinkedIn:** https://www.linkedin.com/post-inspector/ → paste URL → Inspect
- **Twitter/X:** open a draft post, paste URL, confirm preview renders from `twitter:*` tags

---

## 7. AEO / GEO readiness

AEO (Answer Engine Optimization) and GEO (Generative Engine Optimization) target AI engines that synthesize answers from crawled web content. Beyond classic SEO, they need:

### 7.1 Semantic HTML

- Use `<header>`, `<nav>`, `<main>`, `<section>`, `<article>`, `<aside>`, `<footer>` — not a wall of `<div>`s.
- Use `<h1>`-`<h6>` in order to communicate structure.
- Use lists, definition lists, tables where the content is structured. AI engines lift these intact when citing.

### 7.2 Visible, answer-focused copy

AI engines cite text they can see in the rendered HTML. Hidden-only content (e.g., behind a JS click) is not cited. For each page:

- The first ~300 words should answer the implied question of the URL.
- Concrete numbers, prices, timelines, and locations should appear in body text, not only in images or JSON-LD.
- "Who, what, where, when, how much, why us" should be answerable from the page text alone.

### 7.3 JSON-LD schema (only when truthful)

Add `<script type="application/ld+json">` blocks for entities the page actually represents:

| Type | When to use |
|---|---|
| `Organization` | The site represents a business or org. Include `name`, `url`, `logo`, `sameAs` (official social profiles), `contactPoint` |
| `LocalBusiness` (or subtype like `Dentist`, `LegalService`) | The business serves a physical location |
| `Person` | An individual founder, author, expert — with a canonical `@id` shared across their owned sites |
| `WebSite` | Always add on the homepage. Add `potentialAction: SearchAction` **only** if the site has a real search endpoint |
| `WebPage` | Per-page descriptor, linking `isPartOf` to `WebSite` |
| `Service` + `Offer` / `OfferCatalog` / `AggregateOffer` | The page describes a paid service with concrete pricing |
| `BreadcrumbList` | Multi-level navigation paths |
| `FAQPage` | The page renders a FAQ in visible HTML — questions and answers must match the DOM |
| `HowTo` | The page describes a step-by-step process |
| `Article` / `BlogPosting` | Editorial content with an author and date |

**Rules:**

- Two valid organization patterns: multiple flat `<script>` blocks, or one `<script>` with a `@graph` array. Pick one and stay consistent.
- Cross-reference entities by `@id` instead of inlining the same entity in multiple blocks.
- For multi-domain ecosystems where one person founded several sites, host the canonical `Person` block on one domain (`https://$PRIMARY_DOMAIN/#person-founder`) and reference it from `Organization.founder` on the other domains.
- Personal social profiles (the founder's individual LinkedIn) belong in `Person.sameAs`, never in `Organization.sameAs`. `Organization.sameAs` is for URLs that represent the organization itself (company LinkedIn, Crunchbase, GitHub org, etc.).
- `areaServed` should be an array of `Country` + `AdministrativeArea` + `City` when geographic relevance matters, not a single scalar.

Validate with:

- https://search.google.com/test/rich-results — Google's eligibility test
- https://validator.schema.org/ — stricter Schema.org spec validation

---

## 8. Machine-readable files

| File | Purpose | When to add |
|---|---|---|
| `robots.txt` | Crawler control | Always |
| `sitemap.xml` | URL discovery | Always |
| `llms.txt` | Plain-text site summary for AI engines (proposed standard) | When the site has stable content worth being cited |
| `humans.txt` | Credit + stack disclosure | Optional, low-impact |
| `.well-known/security.txt` | RFC 9116 security-contact disclosure | Recommended for any production site |

### 8.1 `robots.txt` AI crawler allowlist

```
User-agent: *
Allow: /

User-agent: GPTBot
Allow: /

User-agent: ChatGPT-User
Allow: /

User-agent: OAI-SearchBot
Allow: /

User-agent: ClaudeBot
Allow: /

User-agent: Claude-SearchBot
Allow: /

User-agent: PerplexityBot
Allow: /

User-agent: Google-Extended
Allow: /

User-agent: Meta-ExternalAgent
Allow: /

User-agent: facebookexternalhit
Allow: /

Sitemap: https://$SITE_URL/sitemap.xml
```

Explicit `Allow` blocks add no permission beyond the `User-agent: *` default, but they are a clear signal of intent and easier to audit.

### 8.2 `llms.txt` structure

A simple plain-text file (not HTML, not JSON). Suggested sections:

```
# $SITE_NAME

> One-line description of what this site is.

## About

What the site is, who runs it, where it operates.

## Products or services

Bulleted list with names, prices (if public), brief descriptions.

## Process

Step-by-step list of how someone goes from interest to purchase or contact.

## Coverage / geography

If the service is location-bound, name the served regions explicitly. Add typical search queries.

## Q&A

Pre-answer common buying questions.

## Contact

Web, email, phone (if public).

## Language

Primary language.
```

Plain language. No marketing fluff. AI engines summarize the first ~1000 words harder than the rest — front-load the most important facts.

### 8.3 `security.txt` template

```
Contact: mailto:security@$SITE_URL
Expires: $EXPIRES_ISO_8601
Preferred-Languages: en
Canonical: https://$SITE_URL/.well-known/security.txt
Policy: https://$SITE_URL/security-policy
```

`Expires` must be a future ISO-8601 datetime. Refresh annually.

---

## 9. Content gap analysis

Before adding pages or schema, check whether the existing copy answers the questions a buyer actually asks. For each public-facing product or service page, the visible text should answer:

| Question | Where to check |
|---|---|
| What does this product or service do? | First paragraph |
| Who is it for? | Hero or first section |
| How much does it cost? | Pricing section |
| What is included vs. not included? | Pricing or features table |
| How long does delivery / onboarding take? | Process section |
| Where do you operate? | Footer or contact, plus `areaServed` schema |
| Who runs the company? | Founder/about section, `Organization.founder` schema |
| What proof exists? (Case studies, reviews, certifications) | Only include if real; never fabricate |
| How does the buying / onboarding process work? | Numbered or stepped section, `HowTo` schema |
| How does a buyer start? | Prominent CTA |

Gaps here are usually a bigger lever than more JSON-LD on a thin page. Fix the visible copy first, then mirror it in schema.

---

## 10. Validation workflow

Run, in this order, after any change:

1. **Build sanity** — `npm run build` (or framework equivalent). Must complete with `0` errors.
2. **Lint/tests if present** — `npm test`, `npm run lint`. Do not skip failing checks; investigate.
3. **Deploy and confirm** — push or trigger deploy; wait for the deploy host to report `READY`/`SUCCEEDED`. If auto-deploy did not pick up the commit within 2 minutes, use the deploy CLI/API to push manually rather than retrying the same push.
4. **Live HTTP check** — every URL touched must return `200`. Use the crawler matrix from §4.
5. **JSON-LD strict parse** — every `<script type="application/ld+json">` block must parse as valid JSON. Recurse into `@graph` arrays. Zero parse errors expected.
6. **Social-tag grep** — confirm all 12 required OG/Twitter tags are present (see §6).
7. **Sitemap `<loc>` HTTP scan** — every URL in the sitemap returns `200`.
8. **Search Console / Bing Webmaster submit** — human step; document URLs to submit and present to the user as a checklist.
9. **Social-debugger re-scrape** — human step; required when OG tags changed, so cached previews refresh.
10. **Monitor if the project has one** — re-run the project's regression check after deploy, confirm no baseline metric regressed (JSON-LD count, key phrases, bot UA status).

A simple validation one-liner for JSON-LD:

```bash
curl -sA "Googlebot/2.1" "https://$SITE_URL/" | python3 -c "
import sys, re, json
html = sys.stdin.read()
blocks = re.findall(r'<script type=\"application/ld\+json\">(.*?)</script>', html, re.DOTALL)
ok = bad = 0
types = []
for b in blocks:
    try:
        d = json.loads(b)
        ok += 1
        if isinstance(d, dict):
            if '@graph' in d and isinstance(d['@graph'], list):
                for e in d['@graph']:
                    if isinstance(e, dict) and '@type' in e:
                        types.append(e['@type'])
            elif '@type' in d:
                types.append(d['@type'])
    except json.JSONDecodeError as e:
        bad += 1
        print(f'PARSE FAIL: {e}')
print(f'blocks: {len(blocks)}  parse OK: {ok}  FAIL: {bad}')
print(f'@types: {types}')
"
```

---

## 11. Run control

A real SEO/AEO change set takes hours, not minutes. To keep it from drifting into infinite loops:

1. **Bounded runs.** Pick a single goal per run (audit, JSON-LD pass, landing page, deploy verification). Do not mix.
2. **Mini-plan before each run.** At most 5 concrete steps with a stop condition for each.
3. **Checkpoints every 30-45 min.** Each checkpoint reports:
   - What was attempted
   - What changed (diff or summary)
   - What evidence proves the change worked (curl output, build log, deploy URL, monitor exit code)
   - Next step
   - One of three flags: `CONTINUE` (clear next step, no input needed), `NEED INPUT` (decision or access required), `BLOCKED` (external wall, can't proceed without something outside the run).
4. **Two-strikes rule.** If the same failure repeats twice with two different reasonable attempts, stop and present 2-3 alternative approaches to the user instead of a third retry of the same path.
5. **No "almost done" claims.** A step is complete only when its proof is captured. Manual steps (Search Console submit, social debugger re-scrape) are not complete until the user confirms.

---

## 12. Final report format

Every run ends with a structured summary the user can read in under two minutes:

```
SEO/AEO Run Summary — $YYYY-MM-DD

Scope
- Target URL(s): ...
- What was attempted: ...

Changes
- Files modified: ... (count + paths)
- Files added: ...
- Commit SHA(s): ...
- Pushed to branch: ...

Live evidence
- HTTP 200 for: ...
- JSON-LD parse: N/N OK, @types: [...]
- OG/Twitter: 12/12 present
- Sitemap: M <loc> URLs all 200
- Bot UA matrix: pass/fail per bot
- Monitor exit code (if applicable): ...

Manual steps required (no auto, human only)
- Google Search Console:
  - URL inspection + Request indexing for: ...
  - Sitemap submit/resubmit
- Bing Webmaster Tools:
  - URL submission for: ...
  - Sitemap submit
- Facebook Sharing Debugger:
  - Scrape Again for: ...
- LinkedIn Post Inspector (optional):
  - Inspect for: ...

Remaining risks / open questions
- ...

Recommended next run
- ...
```

If something is genuinely blocked, mark it `BLOCKED` and name the external dependency. Do not turn a blocked item into a vague "for later" without naming what is needed.

---

## Quick start: smallest viable audit (≈5 minutes)

If the user pastes only a URL and asks "what is missing":

```bash
SITE_URL="https://example.com"

# 1) Root HTTP
curl -sI "$SITE_URL" | head -1

# 2) Crawler matrix on /
for ua in "Googlebot/2.1" "GPTBot/1.0" "ClaudeBot/1.0" "PerplexityBot/1.0" "facebookexternalhit/1.1"; do
  printf "%-25s %s\n" "$ua" "$(curl -s -o /dev/null -A "$ua" --max-time 10 -w "%{http_code}" "$SITE_URL")"
done

# 3) Static files
for f in robots.txt sitemap.xml llms.txt og-image.png .well-known/security.txt; do
  c=$(curl -sI -o /dev/null -w "%{http_code}" "$SITE_URL/$f")
  ct=$(curl -sI "$SITE_URL/$f" | grep -i '^content-type' | awk '{print $2}' | tr -d '\r')
  printf "[%s] %-30s %s\n" "$c" "$f" "$ct"
done

# 4) Title + canonical + JSON-LD inventory
curl -sA "Googlebot/2.1" "$SITE_URL/" | python3 -c "
import sys, re, json
html = sys.stdin.read()
title = re.search(r'<title>([^<]+)</title>', html)
canon = re.search(r'rel=\"canonical\" href=\"([^\"]+)\"', html)
blocks = re.findall(r'<script type=\"application/ld\+json\">(.*?)</script>', html, re.DOTALL)
types = []
for b in blocks:
    try:
        d = json.loads(b)
        if isinstance(d, dict):
            if '@graph' in d and isinstance(d['@graph'], list):
                types += [e.get('@type') for e in d['@graph'] if isinstance(e, dict)]
            elif '@type' in d:
                types.append(d['@type'])
    except: pass
print('title    :', title.group(1) if title else 'MISSING')
print('canonical:', canon.group(1) if canon else 'MISSING')
print('JSON-LD  :', len(blocks), 'blocks,', '@types:', types)
"
```

That gives enough to write the audit findings table and start the conversation.

---

## Anti-patterns to refuse

If asked to do any of these, decline and explain:

- "Add Review schema with 5 stars and 200 reviews" — fabricated; will be penalized.
- "Make AI think we're a Fortune 500" — fabricated; harms trust signals.
- "Add `noindex` to everything except the homepage" — usually wrong; ask why.
- "Cloak content for crawlers" — Google's quality guidelines call this out specifically; long-term penalty risk.
- "Disallow GPTBot but keep ChatGPT-User" — internally inconsistent; explain the difference and get explicit user decision.
- "Stuff the `og:title` with keywords" — clipped by Facebook, looks like spam.

---

## References

- Schema.org vocabulary: https://schema.org/
- Open Graph protocol: https://ogp.me/
- Google Rich Results Test: https://search.google.com/test/rich-results
- Facebook Sharing Debugger: https://developers.facebook.com/tools/debug/
- LinkedIn Post Inspector: https://www.linkedin.com/post-inspector/
- Bing Webmaster Tools: https://www.bing.com/webmasters
- IndexNow protocol: https://www.indexnow.org/
- RFC 9116 (security.txt): https://www.rfc-editor.org/rfc/rfc9116
- llms.txt proposal: https://llmstxt.org/

End of skill.
