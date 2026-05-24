---
name: website-redesigner
description: Use when someone wants to redesign an existing website, rebuild a site in HTML with a new look and feel, refresh a brand's web presence, or modernize a site while keeping its existing content and page structure.
argument-hint: "[website URL]"
allowed-tools: WebFetch, WebSearch, Read, Write, Edit, Bash, Glob, Grep
---

## What This Skill Does

Scrapes an existing website, maps its full structure, collects brand and design direction from the user, then generates a completely redesigned static HTML version — preserving all content and topical structure with working links throughout.

## Required Skills

- **frontend-design** — must be installed globally
- **nano-banana-images** — for any AI image generation in Phase 3.5 (owns the prompt schema + Kie.ai pipeline). If absent, fall back to scraped/placeholder images only.
- **ui-ux-pro-max** — optional but recommended for complex sites. Install with:
  ```bash
  npx skills add https://github.com/nextlevelbuilder/ui-ux-pro-max-skill --skill ui-ux-pro-max
  ```
  If not installed, handle IA planning inline in Phase 5.

---

## Phase 1: Site Scrape & Map

1. Accept the website URL from `$ARGUMENTS`. If not provided, ask: "What is the URL of the site you want to redesign?"

2. **Fetch the XML sitemap first — this is the authoritative URL list.**

   Try in order:
   - `[domain]/sitemap_index.xml` → if found, fetch each referenced child sitemap to collect all URLs
   - `[domain]/sitemap.xml` → fallback if no index exists

   From the sitemap(s), extract every URL and note its type (page vs. post/resource). Filter out obvious staging/draft pages (slugs like `home-duplicate`, `home-test`, `staging-*`, `draft-*`) — flag these to the user and skip them.

   If no sitemap exists, note it and proceed with nav-based discovery only.

3. `WebFetch` the homepage. Extract:
   - Page title and meta description
   - All navigation links (header + footer nav)
   - H1 and H2 headings
   - All internal page links
   - External links (social profiles, contact links)
   - Image URLs (src attributes)
   - Any phone numbers, email addresses, or addresses visible in the HTML

4. Cross-reference the sitemap URL list with the nav links. Any URLs in the sitemap but **not** in the nav are hidden pages — fetch and include them. Any URLs in the nav but not the sitemap are likely dynamic or excluded — still fetch them.

5. Fetch all discovered pages in parallel (up to depth 2 for nav links). For each page, extract the same fields as step 3. If a page fetch fails, note it as `[FETCH FAILED]` and continue.

6. Build a structured **Image Inventory** from all images collected across every page:

   ```
   Image Inventory:
   | URL | Pages found on | Section context | Inferred role |
   |-----|---------------|----------------|---------------|
   | https://…/hero.jpg | Home | Above fold, full-width | Hero image |
   | https://…/team.jpg | About, Home | Card, sidebar | Team / portrait |
   ```

   - Record every unique `<img src>` URL found across all scraped pages
   - Note every page it appears on — duplicates across pages are expected and fine; duplicates **within** a single page are a problem to fix later
   - Infer the image's role from surrounding HTML (hero, card thumbnail, team/portrait, background, icon, logo)
   - Flag any URL that appears in multiple sections of the same page

7. Build a **Site Map** in this format:

   ```
   Page: Home
   URL: https://example.com/
   Title: [page title]
   Purpose: [inferred from content]
   Status: ✓ scraped / ✗ fetch failed

   Page: About
   URL: https://example.com/about/
   ...
   ```

5. Present the site map to the user. Ask: "Does this capture all the pages? Any pages to add, remove, or rename before I start?"

6. **Do not proceed to Phase 2 until the user confirms the site map.**

---

## Phase 2: Brand Brief Collection

Ask the user for brand and design direction. Collect everything in one message:

```
To redesign your site, I need a few design details:

1. Logo — Do you have a logo URL or file path? (Or should I use a placeholder?)
2. Primary color — What's your main brand color? (hex code, or describe it)
3. Accent color — Any secondary or accent colors?
4. Fonts — Any font preferences, or should I choose?
5. Tone & feel — How should the site feel? (e.g., "bold and modern", "warm and approachable", "luxury", "playful", "professional")
6. Inspiration — Any sites you love the look of?
7. Industry context — Anything about your business I should know that wasn't obvious from the site?
```

Wait for the user's response before proceeding.

---

## Phase 3: Missing Info Collection

Review the data collected in Phase 1. List everything that was NOT found on the scraped site. Ask the user to supply it in one message:

```
Here's what I found and what I still need:

✓ Found: [list]
✗ Not found (please provide if you have them):
  - Social media URLs (Instagram, Facebook, LinkedIn, X/Twitter, TikTok, YouTube)
  - Phone number
  - Email address
  - Physical address
  - Tagline or main value proposition
  - Certifications, awards, or trust badges
  - Primary CTA (e.g., "Book a Free Consultation", "Get a Quote", "Shop Now")
```

Only ask for what's genuinely missing. Skip items that were already scraped.

---

## Phase 3.5: Image Strategy

Review the Image Inventory from Phase 1. Present it to the user and ask what to do about images before building pages.

**Step 1 — Flag problems in the inventory:**
- Duplicate: same URL used in 2+ sections on the same page
- Gap: a section that semantically needs an image (hero, team card, service card) but none was scraped or the available images are already used elsewhere
- Thin: only 1-2 unique images exist for a site with many image-heavy sections

**Step 2 — Classify each needed image** by `nano-banana-images` domain mode and aspect ratio:

| Section type | Domain mode | Aspect ratio |
|---|---|---|
| Hero (full-width) | Landscape / Cinema | 16:9 |
| About / firm overview | Editorial / Cinema | 4:3 |
| Team / attorney headshot | Portrait | 3:4 |
| Service card thumbnail | Editorial | 16:9 |
| Background / texture | Abstract | 16:9 or wider |
| Office / location | Landscape | 16:9 |

**Step 3 — Present recommendations to the user in one message:**

```
Here's my image assessment for the redesign:

Scraped images available (X unique):
  ✓ [URL] — [inferred role] — keep as-is? or replace with generated?
  ✓ [URL] — [inferred role] — keep as-is? or replace with generated?

Sections that need additional or replacement images:
  • Hero: recommend Landscape 16:9 — [brief description matching brand/industry]
  • Service cards (N needed): recommend Editorial 16:9 — [brief description]
  • [other gaps]

Which would you like me to generate via AI?
Reply with section names (e.g. "hero, service cards"), "all", or "none — use scraped images only".
```

**Step 4 — For each image the user approves for generation:**
1. Invoke the `nano-banana-images` skill with the domain mode, aspect ratio, and brand context from the Brand Brief. That skill owns the prompt schema, the realism/negative-prompt stack, and the Kie.ai execution pipeline — don't reproduce them here; pass it the context and let it construct the prompt. (Honor the image-integrity rules below regardless of what that skill can produce.)
2. Save generated images to `output/[domain]/assets/images/[section-slug].[ext]`
3. Record the mapping: section → local file path (e.g. `hero → assets/images/hero.jpg`)

**Step 5 — Build the Image Map** (used throughout Phase 6):
```
Image Map:
  hero           → assets/images/hero.jpg          (generated)
  about          → https://example.com/about.jpg   (scraped)
  service-card-1 → assets/images/service-1.jpg     (generated)
  service-card-2 → https://example.com/service.jpg (scraped)
```

**Step 6 — Favicon: pull from the live site, don't design from scratch.**

The existing favicon is part of the brand whether the client thinks of it that way or not — recognized in tabs, bookmarks, and search-result snippets. **Default to pulling it.** Only design a new one if the client explicitly asks for a favicon rebrand. (Trillium build lesson: a hand-designed teal "T" monogram had to be ripped out and replaced with the live site's existing tooth-outline favicon — wasted work.)

1. **Find the master.** Check the live site in this order — stop at the first hi-res hit:
   - The `<link rel="icon" href="…">` and `<link rel="apple-touch-icon" …>` tags in the live page `<head>` (often point to the largest source).
   - For WordPress sites: search `wp-content/uploads/` for `Favicon*.png`, `cropped-Favicon*.png`, `cropped-cropped-*Favicon*.png` — the WP customizer keeps a hi-res master here (commonly 512×512). This is usually the best source.
   - `[domain]/favicon.ico` (legacy fallback — multi-res but small).
2. **Sample the background color** from a corner pixel of the master (via PIL: `Image.open(...).convert("RGB").getpixel((4,4))`). You'll need the hex for `theme-color` and `site.webmanifest`.
3. **Resample to all required sizes from the master** (PIL `Image.LANCZOS`) and save under `output/[domain]/`:
   - `favicon-16x16.png`, `favicon-32x32.png`
   - `apple-touch-icon.png` (180×180)
   - `android-chrome-192x192.png`, `android-chrome-512x512.png`
   - Multi-resolution `favicon.ico` — PIL: `img16.save("favicon.ico", format="ICO", sizes=[(16,16),(32,32),(48,48)], append_images=[img32, img48])`
4. **Write a `site.webmanifest`** with `theme_color` = sampled hex, `name` / `short_name`, the two `android-chrome-*` icons, `display: "standalone"`, `start_url: "/"`.
5. **Save the master** to `assets/images/brand/favicon-master.png` so future re-renders don't need to re-fetch the live site.
6. **Don't synthesize an SVG favicon unless the live site already uses one.** Adding `favicon.svg` when the source is raster-only creates a maintenance fork (the SVG can drift from the PNGs). Keep ICO + PNGs only.
7. **Wire the favicon block into every page's `<head>`** — see the SEO head template in Phase 6.

Tool note: **PIL/Pillow is the safe default** — works without ImageMagick or `rsvg-convert`, which often aren't installed. A small `build_favicon.py` builder (read master → resample → write all sizes → patch every `output/*.html` to insert the favicon `<link>` block after the canonical) is the right shape: idempotent, re-runnable, parity-safe with the builder source.

**Image integrity rules (do not violate these even if the user says "all"):**
- **Never AI-generate a face for a named real person.** Team members, dentists, attorneys, named staff — if you have no real photo, use an initials/monogram avatar or a styled placeholder and ask the client to supply real headshots. A fabricated face of a real, named individual is a misrepresentation, not a stand-in.
- **Never AI-generate documentary photos** — volunteer trips, charity work, event coverage, "our community" imagery. These make truth claims about things that happened. If the real photos can't be scraped, show the verbatim intro copy with a "more photos available — contact us" note rather than inventing scenes.
- **Generic, person-free imagery is the safe zone for AI** (empty clinic interiors, abstract textures, hero atmospheres). Even then, treat every AI image as a **stand-in pending client sign-off** — label it as such. Expect the client may replace or remove them (the Trillium client ultimately removed all AI imagery).
- **Don't upscale tiny scraped images.** Live sites often serve 250–300px thumbnails that look blurry at hero/large sizes. Use them only at small sizes (avatars, thumbnails), or replace.
- **Match photos to people by filename, not `alt` text.** Scraped `alt` attributes are frequently wrong/mislabeled (one person's photo carrying another's name). Flag the matching for client sign-off.

**Scraping fallback for JS-loaded / page-builder galleries:** when a gallery's images are injected by JS at runtime (no static `<img src>` in the fetched HTML — common with Divi shortcodes and WordPress galleries), hit the WordPress REST API directly to recover the real URLs:
```
[domain]/wp-json/wp/v2/pages?slug=[page-slug]
```
The returned `content.rendered` contains the real, full-resolution image URLs embedded in the shortcode markup. This recovered real documentary photos that WebFetch alone couldn't see.

**Do not proceed to Phase 4 until the Image Map is complete** (even if the user chose "none" — confirm the scraped-only map).

---

## Phase 4: Output Format

Ask the user: "How would you like the redesigned site delivered?"

- **Option A — Self-contained pages:** One `.html` file per page with all CSS and JS inline. Easiest to preview in a browser immediately.
- **Option B — Static site folder:** Shared `assets/style.css` and `assets/main.js` linked from each page. Better for a real deployment.

Save all output to: `output/[domain]/`

Wait for the user's choice before proceeding.

---

## Phase 5: UX/IA Planning

**If `ui-ux-pro-max` is installed**, invoke it with this brief:

```
Review the following site map and content structure for a website redesign.
Keep the topical mapping intact — do not remove or significantly reorder existing content.
Plan the information architecture: navigation structure, page hierarchy, and internal linking strategy.
Flag any pages that would benefit from being merged, split, or renamed for clarity.
Produce a concise IA plan ready to hand off to a frontend designer.

Site Map:
[paste the approved site map from Phase 1]

Brand Brief:
[paste the brand brief from Phase 2]
```

**If `ui-ux-pro-max` is not installed**, produce the IA plan inline:
- Define the nav structure (primary + any dropdown groupings)
- Identify pages that should be merged (e.g., thin hub pages with no unique content), split, or renamed
- Define the internal linking strategy (which pages link to which)
- Note any pages better suited as sections of another page rather than standalone files

Present the IA plan to the user. Ask: "Does this structure look right before I start designing?"

**Do not proceed to Phase 6 until the user approves the IA plan.**

---

## Phase 6: Redesign (Batched, frontend-design)

Build pages in batches. **Get explicit user approval after each batch before starting the next.**

Define the shared header/nav and footer HTML/CSS once (Batch 1) and reuse it on every subsequent page — do not redesign the header or footer for each page.

### Build method: hand-write small sites, script large ones

Choose the build mechanism up front based on page count — this decision shapes everything after it.

- **Under ~8 pages:** hand-write each `.html` file directly. Copy the shared header/footer block verbatim onto each.
- **8+ pages (and especially 20+):** **do not hand-write repetitive pages.** Build a small set of **single-source-of-truth Python builder scripts** that emit the HTML. This is dramatically faster, keeps every page consistent, and makes site-wide revisions a one-line change + re-emit. The Trillium build (95 pages) used this and it was the difference between viable and not.

**Builder architecture that worked:**
- One base module (e.g. `build_base.py`) defines the shared chrome **once** as constants/functions: `HEADER`, `FOOTER`, a `doc(title, head, body)` page-shell wrapper, the asset-version constant `V` (the `?v=N` cache-buster), and reusable section helpers (`page_hero()`, `section()`, `prose()`, `cta_band()`, `anchor_nav()`).
- Topic-specific builders (`build_hubs.py`, `build_standalone.py`, `build_kids.py`, …) **import** those helpers so chrome is never redefined. Each builder owns one batch of pages and a list of `(slug, title, content)` records.
- Run a builder to (re)emit its pages. Re-running is idempotent — overwrites are expected.
- Save builders alongside the project (or in `/tmp/` if disposable, but a long project benefits from keeping them in-repo). **The builders are the source of truth, not the emitted HTML.**

**Builder ↔ output parity (the #1 source of bugs in this skill):**
- The shared header/menu/footer lives in **two places**: the live builder constant (`HEADER`/`FOOTER`) **and** every already-emitted `.html` file. When you change shared chrome, you must update **both** — change the constant *and* either re-emit every page or run a one-off patch script (`glob` all `output/**/*.html`, regex-replace, write back) over the existing files. If you only change one, the pages silently drift apart.
- After any chrome change, verify parity: `grep -c` the old string across all output (must be 0) and confirm the new string is present in both a sample page and the builder.
- **Hand-built pages are exempt from the builder** — e.g. a bespoke `index.html` is edited directly, not emitted. Track which pages are hand-built vs. emitted so chrome changes hit both sets.

**Cache-buster discipline:** every CSS/JS edit requires bumping `?v=N` site-wide (bump the `V` constant, re-emit, and patch any hand-built pages). A stale cache is indistinguishable from a broken change — clients will report "it still looks wrong" when they're seeing the old asset. When a client says a fixed thing "still looks off," suspect the cache first.

**Batch 1: Homepage**

Invoke the `frontend-design` skill with:
- The brand brief (colors, fonts, tone, logo)
- The scraped homepage content (H1, headings, copy, CTAs, images)
- The approved IA plan (navigation structure)
- Output format (Option A or B from Phase 4)

After generating, define the shared `<header>`, `<nav>`, and `<footer>` as reusable HTML blocks. Save them as reference for all subsequent batches.

After Batch 1: Ask "Does the homepage look right? Any changes to the overall style, nav, or footer before I build the rest of the site?"

**Batch 2: Primary service/category pages**

One page per primary category. Use the shared header/footer. Apply the brand brief and page-specific scraped content.

After Batch 2: Ask "Does this batch look right? Anything to adjust?"

**Batch 3: Subservice/detail pages**

One page per subservice or detail page. Use shared header/footer.

After Batch 3: Ask for approval before continuing.

**Batch 4: Supporting pages**

About, Contact, Team, Pricing, and any other supporting pages identified in the site map. Use shared header/footer.

---

### Per-Page Rules (apply to every page in every batch)

**SEO `<head>` — required on every page:**
```html
<title>[scraped page title, or: H1 text — Site Name]</title>
<meta name="description" content="[scraped meta description, or first 155 chars of intro copy]">
<link rel="canonical" href="[original live URL for this page]">
<meta property="og:title" content="[same as title]">
<meta property="og:description" content="[same as description]">
<meta property="og:type" content="website">
<meta property="og:url" content="[canonical URL]">
<meta property="og:image" content="[hero image URL or path]">
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="[same as title]">
<meta name="twitter:description" content="[same as description]">
<meta name="twitter:image" content="[same as og:image]">
<link rel="icon" href="/favicon.ico" sizes="any">
<link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png">
<link rel="manifest" href="/site.webmanifest">
<meta name="theme-color" content="[sampled hex from favicon master — see Phase 3.5 Step 6]">
```
Source priority: scraped `<title>` and `<meta name="description">` first; fall back to H1 + page purpose. If neither is available, use `[TODO: add page title]` — never invent one.

**Content fidelity — no invented copy:**
- Every `<h1>`, `<h2>`, `<h3>` must match the scraped heading exactly — no rewording, no condensing, no elaboration
- Every paragraph must be a direct copy from the scraped page — no paraphrasing, no summarizing into new sentences
- Never use a scraped fact as a heading if it wasn't a heading on the live site (e.g. do not promote "James Clarke, former OpenText Deputy GC" into an `<h2>` tag)
- Never invent pull-quotes or asides. If you use a highlighted panel, its text must be a verbatim sentence from the page.
- Eyebrow/kicker labels and button labels are allowed UI microcopy (they're design chrome, not content).

**Allowed exceptions to verbatim (confirm the policy with the client once, then apply consistently):**
- **Heading casing → Title Case.** Source sites are wildly inconsistent (ALL CAPS, lowercase, "DentAL"). Normalizing *casing only* (never wording) to clean Title Case is a deliberate, expected exception. Long-form editorial (blog posts) is often already cased — leave it.
- **Clear factual / clinical errors.** On a professional site (medical, legal, finance) the verbatim rule yields to accuracy: fix obvious misspellings of domain terms ("Orthodonist"→"Orthodontist", "root planning"→"root planing"), broken grammar, merged words, and stray characters the scraper or the source introduced. Keep URL slugs and image filenames on the original spelling so links don't break.
- **Wrong-locale boilerplate.** Source copy often reuses templated text from another market — wrong city ("in Toronto" on a Waterloo site), wrong nationality ("Americans" for a Canadian clinic), wrong currency. Correct to the actual business locale.
- **Leaked third-party / competitor names.** Scraped copy can contain a *different* business's name left over from a template or a shared vendor ("Riverfront Dental" appearing in Trillium's body text). Replace with the correct client name — but first confirm it isn't a genuinely affiliated entity.

**Every correction is a flag, not a silent edit.** Keep a running "Source-content corrections made (flag to client)" log (in the project's CLAUDE.md / a notes file) listing each fix with page and before→after. The client owns their copy; they sign off on every change you made to it.

**Scraper / page-builder artifacts to clean up (these are not content):**
- **Duplicated modules.** Page builders (Divi, Elementor) emit separate desktop/mobile copies of the same text — the scrape shows each paragraph/section twice. De-duplicate to the single intended sequence.
- **Single-item `<ul>` stacks.** A "list" scraped as several consecutive one-item `<ul>`s is a builder artifact — merge into the one intended multi-item list (verbatim items, original order).
- **Stray microcopy.** Navigation/UI fragments like "BACK", "CLICK HERE TO REQUEST AN APPOINTMENT", or a former web vendor's sales testimonial get swept into the scrape — drop them (render real CTAs/links from your own chrome instead).
- **Bold `<p>` labels acting as headings.** A run of bold-paragraph labels that clearly function as section headings (a glossary of terms, a list of named people) may be rendered as `<h3>` — this is the one case where a non-`<h>` element becomes a heading. Flag it for sign-off; it is *not* license to promote arbitrary body sentences.

**No `[TODO]` boxes in delivered pages.** A heading that looks like it has no body text is almost always a sub-heading whose paragraph sits elsewhere in the section — look again before assuming content is missing. Re-fetch the live page to get the real copy. If content is genuinely absent, **ask the client** — never ship a visible `[TODO]` placeholder. (Use `[TODO: copy content from live page — [URL]]` only as an internal marker for `[FETCH FAILED]` pages you'll resolve before delivery.)

**Empty `<meta name="description">` on the live page:** write a fallback description from the page's own intro copy (per the head-tag rule) — do not invent marketing claims, and do not leave it blank.

**Image de-duplication — per page:**
- Before writing HTML for each page, note which images are assigned from the Image Map
- Each image URL or generated file path may appear at most once per page
- If a design section needs an image but all available images are already used: reference the Image Map for an alternate, or design the section without a photo (styled stat block, icon, CSS background, etc.)

**Other rules:**
- Internal links: use relative paths matching the output file structure (e.g., `about.html`, `services/teeth-whitening.html`)
- Social/contact links: use the values provided in Phase 3
- Every page ends with a clear CTA using the value provided in Phase 3
- If any content was missing (`[FETCH FAILED]` pages), use a `[TODO: add content for this page]` placeholder and note it to the user

### Link Integrity Check (per batch)

Before presenting each batch to the user, verify every `<a href>` in the generated HTML:
- External links: must start with `http://` or `https://`
- Internal links: must match an output file that exists or will exist in the site
- Anchor links: must point to a real `id` on the same page

Flag any broken links and fix them before presenting.

### Design Review (per batch, before user approval)

After the link integrity check, invoke the `web-design-guidelines` skill on every HTML file in the batch:

1. Fix any **critical** findings immediately — accessibility failures (missing alt text, broken landmarks, insufficient contrast, missing form labels), missing `<head>` SEO fields, broken interactive elements
2. Note **minor** findings (spacing, typography, visual polish) in a brief summary alongside the batch output — present them to the user but do not block approval

Only present the batch to the user for approval **after** critical findings are resolved.

---

## Phase 7: Review & Final Output

After all batches are approved:

### Coverage audit — run this BEFORE you tell the client it's done

The single worst mistake in a redesign is declaring "every page is built" when it isn't. Nav/menu coverage is **not** proof of completeness — a site can have content-rich pages that aren't linked from any menu. On the Trillium build, an early "nothing left unbuilt" claim was wrong: a later sitemap diff found ~17 substantial (1,000+ word) educational pages that had never been built. Do not repeat this. Run a real diff:

1. **Sitemap-vs-build diff.** Take the authoritative URL list from the Phase 1 sitemap. For each URL, confirm a corresponding output page exists (or is a deliberate, documented drop). List every URL with no built page.
2. **Separate genuinely-unbuilt pages from aliases/redirects.** Before treating a "missing" URL as content to build, check what it actually serves: `curl -sI [url]` to see redirects. A URL that **301-redirects** to a page you already built is an alias — do not build it twice. A bare slug may 301 to the homepage while the real content lives at a nested path; build the nested one. A self-canonical 200 page with unique content is genuinely unbuilt — build it.
3. **Section-level heading diff** (per page, for the pages you did build): compare the built page's `<h1>/<h2>/<h3>` set against the live page's headings. Missing sections mean dropped content. (Account for your own documented re-casings/corrections — those aren't misses.)
4. **Full-site link + a11y audit.** Confirm: 0 broken internal links, 0 dead anchors (`href="#x"` with no `id="x"`), 0 `[TODO]` placeholders, every `<img>` has `alt` + explicit `width`/`height`. Script this over `output/**/*.html` rather than spot-checking.
5. **Document the result honestly.** State the page count built, the URLs deliberately dropped (with reason), and any pages still pending a client decision. If you found unbuilt pages, say so plainly — don't paper over it.

### Final output

1. List all files produced:
   ```
   output/[domain]/
     index.html
     about.html
     services.html
     services/[subservice].html
     contact.html
     ...
     assets/style.css   (Option B only)
     assets/main.js     (Option B only)
   ```

2. Confirm with the user: "Your redesigned site is complete. All files are in `output/[domain]/`. Open `index.html` in a browser to preview."

3. Flag anything that still needs human attention:
   - `[TODO]` placeholders
   - Pages where WebFetch failed
   - Images that couldn't be scraped and need replacement

---

## Output Structure

```
output/
  [domain]/
    index.html
    [page-slug].html
    [category]/
      [subpage-slug].html
    assets/              (Option B only)
      style.css
      main.js
      images/            (AI-generated images from Phase 3.5)
        [section-slug].[ext]
```

---

## Guardrails

- Never proceed past Phase 1 without user confirming the site map
- Never proceed past Phase 3.5 without completing the Image Map (even if the user chose "none — use scraped only")
- Never proceed past Phase 5 without user approving the IA plan
- Never skip a batch approval gate
- Never skip the web-design-guidelines review before presenting a batch to the user
- **Never invent or paraphrase headings or body copy** — all `<h1>`/`<h2>`/`<h3>` and paragraph text must be verbatim from the scrape; use `[TODO: copy content from live page]` if content is missing
- **Each image URL or generated file may appear at most once per page** — de-duplicate before writing HTML; resolve conflicts using the Image Map or imageless design sections
- **Every page must have a complete `<head>`** with title, meta description, canonical URL, Open Graph, and Twitter Card tags derived from scraped data
- If the site has 20+ pages, warn the user and suggest prioritizing key pages first: "This site has [N] pages. Would you like to start with the most important pages first and add the rest later?"
- If WebFetch is blocked (bot protection, login wall), tell the user and ask them to paste the page content manually
- Do not reproduce the old site design — the goal is a fresh redesign, not a copy
- **For 8+ page sites, build via Python builder scripts, not by hand** — and keep builder ↔ output parity on every shared-chrome change
- **Never declare the build complete without running the Phase 7 coverage audit** (sitemap-vs-build diff, not nav coverage)
- **Never AI-generate faces of named real people or documentary photos** — stand-ins only for generic, person-free imagery, and even those pending client sign-off
- **Every edit to a client's own copy is a flag, not a silent change** — log it with before→after for sign-off
- **Bump the `?v=N` cache-buster site-wide on every CSS/JS change** — and suspect a stale cache first when a client says a fixed thing still looks wrong
- **Menu/nav labels must be verbatim with the live site** — don't invent groupings (e.g. a "More Services" bucket) or abbreviate labels; mirror the live site's own categorization
- **Pull the favicon from the live site — don't design a new one** unless the client explicitly asks for a favicon rebrand. The live favicon is recognized brand; replacing it is a rebrand decision, not an asset task. See Phase 3.5 Step 6 for the resample-and-wire pipeline.
- **Google Maps URLs must query by `Business+Name,+Address`, not address alone** — applies to both the `<iframe class="map-embed" src="…/maps?q=…&output=embed">` and every click-to-open address link (`<a href="…/maps?q=…">`) in the topbar/footer/contact section. The business-name query resolves to the verified GBP, so the pin is labelled with the business and the click lands on the live listing (hours, reviews, Directions) — a real NAP-consistency / outbound-entity / engagement win for local SEO. See Field notes for the URL shape.

---

## Field notes (hard-won, from real builds)

**URL / slug consolidation.** Legacy sites accumulate duplicate URL trees (e.g. an old `/procedures/...` hierarchy alongside newer flat service slugs). Consolidate to the newer set and drop the legacy duplicates — but keep each page's `<link rel="canonical">` pointing at the **real live URL** even when your output filename differs (e.g. you build `root-canal-therapy-waterloo.html` but canonical → the live `/endodontics/root-canal-therapy-waterloo/`). Confirm redirects with `curl -sI` before deciding a URL is a duplicate vs. unique content.

**Third-party embed widgets** (reviews, booking, chat). These are `<script>` + `<iframe>` blocks that load external content at runtime:
- The iframe usually **auto-resizes to its content height via `postMessage`**. A large CSS `min-height` floor on it then creates a glaring empty void (especially on a colored section). Set a modest floor and let the script size it; if dead space remains, the widget's content is genuinely shorter than the section — get the rendered height before dialing in spacing.
- To make a widget "feel native" inside a colored section, match the section background to the widget and override heading color explicitly. Element selectors like `h2 { color: var(--ink) }` beat *inherited* white from the section, so you need an explicit `.section--dark h2 { color:#fff }` — inheritance alone won't win on specificity.

**Google Maps embeds — query by business name + address, not address alone.** The keyless iframe embed at `https://www.google.com/maps?q=<query>&output=embed` shows whatever Google's search resolves the query to. With just an address (`q=550+King+Street+N+Waterloo+ON`) the pin is labelled with the bare address. Include the verified business name in the query — `q=Trillium+Dental+Centre,+550+King+Street+N,+Waterloo,+ON+N2L+5W6` — and Google resolves to the verified business listing, so the pin is labelled with the business name. That's what the client expects to see and what reinforces NAP consistency for local SEO. Define the iframe once (e.g. a `MAP` constant in your builder base) and reuse it on every page that needs a map (typically contact + homepage). The same business-name-first query also works as the `href` for click-to-open address links in the topbar/footer so every map reference lands on the business, not a generic address pin.

**Accessibility — bake into every page, not a final pass:** skip link to `#main`, `aria-hidden="true"` on decorative SVGs, `alt` + explicit `width`/`height` on every `<img>`, labelled form fields, `:focus-visible` styles, honor `prefers-reduced-motion`, and list transition properties explicitly (never `transition: all`). Decorative icons are inline SVG, not `<img>`.

**Decisions drift over a long build — write them down as you go.** A multi-batch redesign accrues client decisions, source corrections, dropped pages, and open questions. Maintain a project `CLAUDE.md` (conventions, brand tokens, NAP/contact facts, per-batch status) and a `TODO.md` (open questions, thin pages, pending client calls). Future-you and the client both rely on this; reconstructing it from the HTML later is painful.
