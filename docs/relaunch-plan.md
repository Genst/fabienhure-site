# fabienhure.com — Relaunch Plan

Tracking document for the site upgrade from "occasional-essay blog" to a publishing
platform with a talks portfolio. Created 2026-06-20. Update the status boxes as work lands.

## Goals

1. **Host presentations** — add a proper section for talks (starting with STAC), with
   self-hosted PDF slide decks.
2. **Restart regular publishing** — across Quant Finance, Quantara, AI, Engineering, and
   Architecture, with the structure and ergonomics to sustain a cadence.

## Locked decisions (from kickoff Q&A, 2026-06-20)

- **Presentations format:** self-hosted **PDF embed** — title/date/venue + abstract +
  inline PDF viewer + download link.
- **Taxonomy:** five top-level categories — **Quant Finance, Quantara, AI, Engineering,
  Architecture** — surfaced via a `Topics` menu entry. Tags remain free-form beneath.
- **Analytics:** **Cloudflare Web Analytics**, wired behind a token param so nothing ships
  until the token is set. (Open item: token.)
- **Audience/retention:** surface **RSS** prominently now; **newsletter** wanted but
  provider not yet chosen — see "Newsletter decision" below.

## Open items needing the user

- [x] **Cloudflare Web Analytics token** — set in `hugo.toml`; beacon verified emitting.
- [x] **OG share image** — `static/images/og-default.png` (1200×630) in place; `og:image`
      verified.
- [x] **STAC presentation** — `content/presentations/stac-summit-london-2026/` live with
      embedded PDF; `example-stac-talk.md` placeholder removed.
- [ ] **Newsletter provider** — deferred by the user for now. Recommendation below stands;
      hook left in place for when it's wanted.

## Newsletter decision (for later)

Recommendation: **Buttondown**. Privacy-friendly, developer-oriented, has a free tier,
and supports **RSS-to-email** so publishing a post can auto-send the issue (no separate
writing step). Alternatives: Substack (more built-in reach/discovery but heavier branding
and a social layer), MailerLite/ConvertKit (more marketing-oriented). A hook is left in
`extend_head.html` / footer so adding the embed later is a small, isolated change.

## Workstreams

Ordered by impact. Status as of 2026-06-20.

### A. Math rendering (KaTeX) — DONE
- [x] `layouts/partials/extend_head.html` loads KaTeX auto-render, gated on `.Param "math"`.
- [x] Goldmark passthrough enabled in `hugo.toml` so `$...$` / `$$...$$` reach KaTeX.
- [x] Verified: `katex` assets present on the `math: true` post.

### B. Analytics (Cloudflare Web Analytics) — DONE (token pending)
- [x] `[params.analytics.cloudflare] token` param added to `hugo.toml`.
- [x] CF beacon emitted from `extend_head.html` only when the token is set (verified absent
      while empty).
- [ ] **OPEN:** paste the actual token.

### C. Presentations section — DONE (real talk pending assets)
- [x] `content/presentations/_index.md` (section landing).
- [x] `layouts/presentations/list.html` (list of talks with event/location/date).
- [x] `layouts/presentations/single.html` (venue/date + abstract + inline PDF embed +
      download; optional video/slides URL).
- [x] `archetypes/presentations.md`.
- [x] `[[menu.main]]` entry for Presentations.
- [x] Draft template talk at `content/presentations/example-stac-talk.md` (draft, not built
      in prod) doubles as a guide.
- [ ] **OPEN:** add real STAC talk(s) once PDF + abstract + dates are provided.

### D. Publishing ergonomics — DONE
- [x] `archetypes/default.md` — standard blog front matter.
- [x] Publishing flow documented in `README.md`.

### E. Sharing / OpenGraph — DONE (PNG pending)
- [x] Site-wide `images = ["images/og-default.png"]` fallback in `hugo.toml` (verified in
      `og:image` meta).
- [x] `static/images/og-default.svg` source provided.
- [x] Per-post `cover` convention documented in `archetypes/default.md`.
- [ ] **OPEN:** export `static/images/og-default.png` (1200×630) from the SVG.

### F. Taxonomy & navigation — DONE
- [x] `Topics` menu entry (→ `/categories/`).
- [x] Determinism post categories normalised to Architecture / Quant Finance.
- [x] Category/tag pages render.

### G. Homepage + internal linking — DONE
- [x] `homeInfoParams` rewritten with credibility line + Blog/Presentations links.
- [x] Two posts cross-linked as a coherence→determinism arc (Hugo `ref` shortcodes).

### H. RSS prominence — DONE
- [x] RSS social icon (→ `/index.xml`) and `ShowRssButtonInSectionTermList` enabled.

### I. Housekeeping — DONE
- [x] `README.md` updated (`config.toml` → `hugo.toml`, real structure, authoring guide).

## Deprecation warnings — RESOLVED

Hugo emitted three deprecation warnings (Hugo v0.158); all now fixed, build is warning-free:

- `languageCode` config key → replaced with top-level `locale = "en-gb"` in `hugo.toml`.
  (Note: a `[languages.en]` block broke template lookup with this theme; top-level `locale`
  is the working fix.)
- `.Language.LanguageDirection` → fixed via project override `layouts/baseof.html`
  (uses `.Language.Direction`).
- `.Language.LanguageCode` → fixed via project overrides
  `layouts/_partials/templates/opengraph.html` and `layouts/rss.xml` (use `.Language.Locale`).

These three override files are patched copies of PaperMod templates. When PaperMod upstream
adopts the new API, delete the overrides and let the theme provide them again.

## Phase 2 — Branding & look-and-feel (2026-06-20)

Direction chosen: **Slate & Violet** (accent `#7c6cf0`, slate dark surfaces, serif
headings). Implemented as four commits:

- **Design system** — `assets/css/extended/custom.css`: accent (light/dark), serif
  headings, and accent styling for links, nav, blockquotes, table headers, tags, selection.
- **Series** — `series` taxonomy, per-post "Part N of M" banner (`layouts/partials/series.html`
  via `layouts/single.html` override), Series menu entry, generated landing page. Two
  existing posts seeded as the "Platform Coherence" series.
- **Diagram system** — Mermaid recoloured to the palette, re-renders on theme toggle,
  centred, with a caption convention (italic line directly after a diagram).
- **Homepage + wordmark** — `static/favicon.svg` monogram used as favicon (SVG + PNG
  fallbacks) and header logo via `[params.label]`.

Follow-up refinements (2026-06-20):

- **Removed AI design tell** — dropped the coloured left accent bar/tint from the series
  tile and reverted blockquotes to a neutral rule.
- **Code presentation** — class-based syntax highlighting (github / github-dark scoped under
  `.dark`, backgrounds deferred to `--code-bg`); `assets/css/extended/syntax.css`.
- **KaTeX** — inherits content colour for dark mode.
- **Per-post OG images** — auto-generated branded title cards (`assets/images/og-base.png`
  base + `layouts/partials/og-image.html`), used as og:image unless a post sets a cover.

Still open: a portrait on About (awaiting the photo — square ≥1000×1000); optional web-font
serif; newsletter (parked).

## Changelog

- 2026-06-20 — Plan created; decisions locked from kickoff Q&A.
- 2026-06-20 — Workstreams A–I implemented; clean prod build green (29 pages).
- 2026-06-20 — CF token + OG PNG added by user. Real STAC talk
  (Time-Coherent-Analytics-at-Scale) added; placeholder removed. All Hugo deprecation
  warnings resolved. Newsletter deferred.
