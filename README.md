# churi12.github.io

Personal blog, served by GitHub Pages (Jekyll, minimal theme).

## How to add a post

Create a file in `_posts/` named `YYYY-MM-DD-some-title.md` with front matter:

```
---
layout: post
title: "Your title"
date: 2026-06-17
---

Body in markdown.
```

Commit and push to `main`. GitHub builds and publishes automatically within a minute or two. Live at https://churi12.github.io

No local build needed. To preview locally (optional) you would need Ruby + `bundle exec jekyll serve`, but pushing is enough.
