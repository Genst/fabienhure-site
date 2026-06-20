---
title: "Time-Coherent-Analytics-at-Scale"
date: 2026-04-29T09:15:00+01:00
draft: false
description: "Time-Coherent-Analytics-at-Scale presentation at STAC Summit London 2026"
event: "STAC Summit London"        # Conference / venue, e.g. "STAC Summit London"
location: "London, UK"     # City, optional
eventDate: 2026-04-29       # Date of the talk, e.g. 2025-11-12
pdf: "Time-Coherent-Analytics-at-Scale-Fabien-Hure-STAC-Summit-London-April-2026.pdf"          # Slide deck: a page-bundle file (e.g. "slides.pdf") or a /path URL
# video: ""      # Optional embeddable recording URL
# slidesURL: ""  # Optional external slides link (SpeakerDeck, Google Slides)
tags:
  - QuantFinance
  - Architecture
---

Modern pricing and risk stacks often behave as if everything changed on every tick: graphs are rebuilt, objects are rebound, and broad caches invalidated. In reality, most portfolios and market states change only marginally between ticks (and especially between days). This talk presents a practical architecture that exploits that property: snapshot barriers to guarantee “as-of” coherence, delta sets to represent change explicitly, and layered precomputations enabling a fast path for steady-state positions and a slow path for new/amended trades. We’ll show how deterministic compiled valuation plans remove runtime traversal overhead, and how minimal run/snapshot lineage makes results replayable and debuggable.
