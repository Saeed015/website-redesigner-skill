# website-redesigner

A Claude Code skill that turns an existing live website into a freshly redesigned static HTML site — preserving every page, every heading, and every paragraph of the source content **verbatim**, while letting you replace the look, the architecture, and the assets.

It is opinionated: a redesign is not a content rewrite, and a content rewrite is not a redesign. This skill enforces that distinction. If you want new copy, that's a separate job.

---

## What it does

1. Scrapes the live site (XML sitemap first, then nav-based discovery as fallback).
2. Builds a verifiable site map + image inventory.
3. Collects your brand brief — colors, fonts, tone, logo, inspiration.
4. Plans the information architecture.
5. Generates fresh static HTML pages, batch-by-batch, with you approving each batch.
6. Audits coverage against the sitemap before declaring "done."

Output lands in `output/[domain]/` — either self-contained pages (Option A) or a shared-asset folder structure (Option B, recommended for 8+ pages).

---

## Install

```bash
npx skills add https://github.com/Saeed015/website-redesigner-skill --skill website-redesigner
```

Or clone directly into your skills directory:

```bash
git clone https://github.com/Saeed015/website-redesigner-skill ~/.claude/skills/website-redesigner
```

---

## Use

In Claude Code:

```
/website-redesigner https://example.com
```

The skill will walk you through 7 phases, pausing at gates for your input and approval. You can also invoke it without an argument and it will ask for the URL.

---

## The 7 phases

| Phase | What happens | Gate |
|---|---|---|
| **1 — Site scrape & map** | XML sitemap → all pages fetched → headings, copy, images, links extracted → site map built | You confirm the site map |
| **2 — Brand brief** | Logo, primary/accent color, fonts, tone, inspiration, industry context | You reply |
| **3 — Missing info** | NAP, socials, tagline, CTAs, awards — only asks for what wasn't scraped | You fill the gaps |
| **3.5 — Image strategy** | Inventory → classification (hero / portrait / card / etc.) → optional AI generation via `nano-banana-images` → favicon pulled from the live site | You confirm the image map |
| **4 — Output format** | Option A (self-contained pages) or Option B (shared-asset folder) | You choose |
| **5 — UX/IA planning** | Nav structure, page hierarchy, internal linking; delegates to `ui-ux-pro-max` if installed | You approve |
| **6 — Redesign (batched)** | Built in batches (homepage → primary services → subservices → supporting pages), each through the `frontend-design` skill + link integrity + `web-design-guidelines` review | You approve each batch |
| **7 — Coverage audit** | Sitemap-vs-build diff, redirect-aware (no double-building aliases), heading-level diff per page, full link/a11y scan | You see the honest result |

Each gate is a hard stop. The skill won't barrel ahead.

---

## What it will not do

These are the rules built into the skill, not preferences. They exist because skipping them produces sites that look fine but misrepresent the business.

- **No invented copy.** Every `<h1>`/`<h2>`/`<h3>` and every paragraph is verbatim from the scrape. The skill won't paraphrase, condense, summarize, or promote a body sentence into a heading.
- **No AI faces of named real people.** Team members, dentists, attorneys, anyone identified by name on the live site. If a headshot can't be scraped, the skill uses an initials placeholder and asks the client.
- **No AI documentary photos.** Volunteer trips, community events, charity work — anything that makes a truth claim. Generic person-free imagery only.
- **No silent edits to client copy.** Every correction (clinical-term fix, locale swap, leaked competitor name, scraper typo) gets logged with before→after for client sign-off.
- **No "all pages built" claim without a sitemap diff.** Nav coverage is not proof of completeness. The Phase 7 audit is mandatory.
- **No favicon redesign by default.** The existing favicon is brand. It gets pulled from the live site and resampled, not replaced.
- **No invented menu groupings.** Nav labels mirror the live site's own categorization.

These constraints make the skill slower than "just rewrite it." That's the point.

---

## Required companion skills

| Skill | Required? | Used for |
|---|---|---|
| [`frontend-design`](https://github.com/anthropics/skills) | **Yes** | Generates each page's visual design from the brand brief |
| [`nano-banana-images`](https://github.com/Saeed015) | Recommended | AI image generation in Phase 3.5 (owns the Kie.ai prompt pipeline). Without it, the skill falls back to scraped/placeholder images. |
| [`ui-ux-pro-max`](https://github.com/nextlevelbuilder/ui-ux-pro-max-skill) | Optional | Delegated IA planning in Phase 5. Without it, the skill plans IA inline. |
| [`web-design-guidelines`](https://github.com/anthropics/skills) | Recommended | Per-batch design + a11y review before each user-approval gate |

---

## Build architecture for large sites

For 8+ pages, the skill builds a small set of Python builder scripts as the single source of truth, rather than hand-writing repetitive HTML:

- **`build_base.py`** defines shared chrome once: header, footer, page-shell wrapper, asset-version cache-buster, reusable section helpers.
- Topic builders (`build_hubs.py`, `build_standalone.py`, etc.) import those helpers.
- Re-running a builder is idempotent and re-syncs its pages to the builder state — a clean way to flush patch-vs-builder drift.

This decision is made up front in Phase 6. A 95-page site is the difference between viable and not.

---

## Field-tested on

A single-location healthcare practice. 111 static pages built across 6 service hubs, 5 standalone services, 44 sub-service child pages, 16 chrome/info pages, 11 blog posts, 8 volunteer galleries. Pre-launch SEO audit scored the rebuild at 87/100 vs. the live WordPress source at 59/100 (+28 points).

The hard-won lessons from that build — favicon pipeline, third-party widget styling, scraper artifact cleanup, builder-↔-output parity, the verbatim-rule exceptions for clinical-accuracy fixes — are all baked into the skill.

---

## Full spec

See [`SKILL.md`](SKILL.md) for the complete phase-by-phase instructions, guardrails, and field notes. The README is the introduction; SKILL.md is what Claude Code actually loads when the skill runs.

---

## License

MIT.
