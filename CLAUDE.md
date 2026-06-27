# CLAUDE.md — churi12.github.io

Personal technical blog for Miguel Santos (GitHub: **Churi12**. Static Jekyll site hosted on GitHub Pages.
Live at **https://churi12.github.io**.

## What this is

- **Jekyll, no theme.** All layouts are hand-written in `_layouts/`; all styling
  is one hand-written file, `assets/style.css`. Do NOT add a theme gem (minima
  etc.) — a theme was tried and removed because it fought the custom CSS.
- **Built by a GitHub Actions workflow** (`.github/workflows/jekyll.yml`), not the
  legacy Pages builder. Pages `build_type` is set to `workflow`. This matters:
  the legacy builder did not apply custom layouts/CSS correctly.
- **Design direction: "the terminal as editorial."** Dark, layered-ink palette
  (base `#0e1217`, never pure black), off-white text, one muted teal signal
  (`#19bd95` / links `#38d2ad`) used ONLY for prompts, status dots, tags, links.
  Type is IBM Plex Sans (display/body) + IBM Plex Mono (structure/code). The
  signature elements are the `$ whoami` shell-prompt hero and the `kubectl`-style
  status-line post metadata (green dot · ISO date · tag). Favicon is a vector
  `>_` prompt in `assets/favicon.svg`. Avoid the AI-cliché "pure-black + neon-green
  hacker" look — the restraint is the point.

## Layout / file map

- `_config.yml` — site config. NOTE: no `email:` field (it renders publicly).
- `_layouts/default.html` — HTML shell: fonts, favicon, sticky nav, footer.
- `_layouts/home.html` — hero + post feed (cards). Hero copy comes from the
  `heading:` front matter in `index.md`, NOT the site title (avoid repeating the
  name; it already appears in nav + footer).
- `_layouts/post.html` — article: back link, title, status-line byline, body.
- `_layouts/page.html` — simple pages (About).
- `assets/style.css` — the entire stylesheet. Starts with empty `---` front
  matter so Jekyll processes it.
- `_posts/YYYY-MM-DD-slug.md` — posts.

## How to add a post

Create `_posts/YYYY-MM-DD-some-slug.md`:

```
---
layout: post
title: "Sentence-case title with a real hook"
date: 2026-06-18
author: Miguel Santos
tags: [loki]          # first tag shows as the status-line chip; use the tool name
---

Body in Markdown.
```

Commit, push to `main`, the Actions workflow builds and deploys in ~1 minute.

## House rules (learned the hard way)

- **Verify live, cache-busted.** GitHub Pages CDN caches ~600s and the browser on
  top of that. After deploying, append `?cb=<something>` to URLs when checking, or
  the user will see a stale version and think it is broken. Most "it's broken"
  reports in this project were stale cache, not real bugs.
- **Screenshot before pushing.** Ruby here is too old for local Jekyll, but Chrome
  is at `/Applications/Google Chrome.app/Contents/MacOS/Google Chrome`. Render a
  static preview (inline the CSS, mirror the layout HTML) and `--screenshot` it,
  then look at it. A picture catches what greps miss.
- **Voice:** first-person, plain, specific, no filler or corporate tone. Every
  post ties back to something concrete (usually an upstream contribution Miguel
  made: "I hit this, read the source, fixed it, here's what I learned"). Sentence
  case everywhere. No em-dashes-as-decoration, no hype.
- **Identity is LOCAL to this repo only.** `git config --local user.name churi12`
  / `user.email miguel_cristinha@hotmail.com`. NEVER touch global git config — it
  holds the work (Kuehne+Nagel) email and must stay untouched.
- **Pushing:** the GitHub token in the macOS keychain (host github.com) is
  Churi12's, with `repo` scope. Pull it with `git credential fill`, never print it.
- **No JS frameworks, no external build step, vanilla CSS only.** Must stay a
  plain static site GitHub Pages can build.

## Existing posts (each maps to a real upstream PR)

- `2026-06-17` Loki retention vs object-store lifecycle policies → grafana/loki#22437
- `2026-06-18` Karpenter provisions, kube-scheduler binds → aws/karpenter-provider-aws#9264
- `2026-06-18` Running loki-canary to catch silent log loss → grafana/loki#22438
- `2026-06-18` Integration-testing the Mimir compactor → grafana/mimir#15709
- `2026-06-18` A timestamp was busting my Go build cache in Alloy → grafana/alloy#6541
- `2026-06-18` Zone-aware Mimir pointed its ingress at a missing service → grafana/mimir#15740
- `2026-06-23` ExternalDNS created the AAAA record, then forgot it existed → kubernetes-sigs/external-dns#6517
- `2026-06-27` A secret got printed as a Go struct into my scrape target → grafana/alloy#6605

## Tool icons (added 2026-06-18)

Each post shows a small tool mark keyed on its first tag, via `_includes/tool-icon.html`
(rendered in `home.html` feed cards, left of the title, and in `post.html` header as a
labeled badge). The marks are ORIGINAL minimal geometry in the site's teal/mono language
(loki = stacked log lines, mimir = concentric rings, karpenter = scaling chevrons,
alloy = fused diamonds, external-dns = two linked nodes / a record resolving to an
address), NOT the upstream brand logos — keeps the palette and avoids
trademark issues. Unknown tags fall back to a generic `>_` prompt glyph. To add a tool,
add a `when` arm to the include and (optionally) note it here. SVGs use `currentColor`;
styling is in the "Tool icons" block of `assets/style.css`.
