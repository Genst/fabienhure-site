# fabienhure-site

Source repository for **fabienhure.com**.

This repository contains the Hugo source files used to generate and publish my personal website and blog.  
The site is intentionally minimal and content-focused.

---

## Overview

The website hosts:

- Long-form writing on quantitative finance, analytics platforms, and engineering trade-offs
- An About page and contact information
- A simple blog structure with no CMS or dynamic backend

The site is built using:

- **Hugo** (static site generator)
- **Markdown** for content
- **Cloudflare Pages** for hosting and delivery

---

## Repository Structure

```text
.
├── archetypes/        # Front-matter templates (default.md, presentations.md)
├── content/
│   ├── about.md
│   ├── contact.md
│   ├── blog/          # Long-form posts
│   └── presentations/ # Talks, with embedded PDF decks
├── docs/
│   └── relaunch-plan.md  # Roadmap and open items
├── layouts/           # Theme overrides (math, mermaid, presentations)
├── static/
├── themes/PaperMod/   # Theme (git submodule)
├── hugo.toml          # Site configuration
└── README.md
```

## Authoring

```bash
hugo new blog/my-post.md            # new draft post (uses archetypes/default.md)
hugo new presentations/my-talk.md   # new talk (uses archetypes/presentations.md)
hugo server -D                      # local preview, including drafts
```

Posts and talks start as `draft: true`; set `draft: false` to publish. Categories use the
fixed set: Quant Finance, Quantara, AI, Engineering, Architecture. Set `math: true` for
posts with LaTeX (`$...$` / `$$...$$`) and `mermaid: true` for diagrams. See
`docs/relaunch-plan.md` for the open items (Cloudflare token, OG image, newsletter).
