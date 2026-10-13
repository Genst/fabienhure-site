# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
hugo server -D              # local preview including drafts (http://localhost:1313)
hugo                        # production build → public/ (drafts excluded; run before publishing)
hugo new blog/my-post.md            # new draft blog post (uses archetypes/default.md)
hugo new blog/my-post/index.md      # same, as a page bundle (for posts with charts or data)
hugo new presentations/my-talk.md   # new talk (uses archetypes/presentations.md)
```

## Architecture

Hugo static site with the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme (git submodule at `themes/PaperMod/`). Deployed on Cloudflare Pages. Requires Hugo ≥ 0.146 (PaperMod's check in `baseof.html`).

**Content** lives in `content/`:
- `blog/` — long-form Markdown posts, either a single file or a page bundle (`blog/<slug>/index.md` with `charts/` and `data/` alongside)
- `presentations/` — talk pages (each as `presentations/<slug>/index.md` with a PDF bundle or external link)
- `about.md`, `contact.md` — standalone pages

`temp/` (gitignored) is scratch space for drafts in progress; nothing in it is built. Every file in a page bundle is published, referenced or not.

**Front-matter conventions:**
- `draft: true` until ready to publish
- `description` is shown under the title and used for OG/RSS; `summary` overrides the list-page excerpt (usually the same standfirst)
- `categories` must be one of: `Quant Finance`, `Quantara`, `AI`, `Engineering`, `Architecture`
- `series` groups related posts (displayed via `layouts/partials/series.html`; e.g. `Quantara Internals`, `Platform Coherence`)
- `math: true` enables KaTeX rendering (`$...$` inline, `$$...$$` display); `mermaid: true` enables Mermaid diagrams
- `cover.image` overrides the auto-generated OG card

**Shortcodes** (`layouts/_shortcodes/`):
- `chart` — inlines an SVG from the page bundle in a `<figure>` with a Markdown caption, an optional CSV download link (`data`, `datalabel`) and a "Full size" link. Inlined (not `<img>`) so the SVG's palette can follow the site theme toggle.
- `metric-table` — renders a grouped-column table from a bundle CSV (one column group per metric, one column per method, rows grouped by a CSV column), bolding the value closest to zero in each group. Usage is documented at the top of the file.

**Layouts** (all override PaperMod defaults; Hugo's new `_partials/`, `_shortcodes/`, `_markup/` directories sit alongside the older `partials/`):
- `layouts/partials/extend_head.html` — KaTeX CDN (conditional on `math: true`) and Cloudflare Web Analytics
- `layouts/partials/extend_footer.html` — Mermaid loader, re-rendered with a light or dark palette when the theme toggles
- `layouts/partials/og-image.html` — OG image logic: uses explicit `cover.image` → auto-generated title card from `assets/images/og-base.png` → site-wide fallback `static/images/og-default.png`; called from `layouts/_partials/templates/opengraph.html`
- `layouts/_partials/share_icons.html` — share buttons (LinkedIn, X, HN, Reddit, email, copy-link)
- `layouts/_markup/render-codeblock-mermaid.html` — Mermaid fenced-code block renderer
- `layouts/presentations/` — custom list and single templates for the presentations section
- `layouts/single.html`, `layouts/baseof.html`, `layouts/rss.xml` — near-copies of PaperMod's templates: `single.html` adds the series partial; `baseof.html` and `rss.xml` swap deprecated Hugo calls (`.Language.Direction`, `site.Language.Locale`). After updating the PaperMod submodule, diff these against the theme's versions and carry over upstream changes.

**Theme mode:** PaperMod sets `html[data-theme="light"|"dark"]` (from the toggle, else the OS preference). Custom CSS and JS should key on that attribute, not on `prefers-color-scheme` or a `body.dark` class.

**Styling:** `assets/css/extended/custom.css` and `syntax.css` are PaperMod's extension points; they are loaded automatically. Post body styles target `.md-content`.

**Charts:** SVG charts for posts are generated outside this repo (with the analysis that produced them). Each SVG carries its own light/dark palette under `.viz-root`, switched by `prefers-color-scheme` when opened on its own and by `[data-theme]` when inlined by the `chart` shortcode. Keep both selectors when regenerating.

**OG image generation:** Hugo's built-in image processing stamps the page title onto `assets/images/og-base.png` at build time, producing a per-post branded card. No external tooling required.

## Commits

Conventional-commit style with a scope, e.g. `feat(blog): add post on …`, `feat(layouts): …`, `style(css): …`.
