# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Personal portfolio and blog site for Harry Zhao, built with Jekyll and hosted on GitHub Pages at https://harry-zhao-au.github.io.

## Git Configuration

**Critical**: This machine's global git config uses a corporate account. This repo must use the personal account:
- **Do not use `gh` CLI** — it authenticates with the corporate account
- Local git config is set: `user.name=Harry-Zhao-AU`, `user.email=harry.zhao19@gmail.com`
- Remote uses SSH: `git@github.com:Harry-Zhao-AU/harry-zhao-au.github.io.git`
- Commit messages should be short and simple (e.g. "fix contact link", "add new post")
- Do not add Co-Authored-By lines to commits

## Development

No local Ruby/Jekyll required — GitHub Pages builds server-side on push to `main`.

Optional local preview (requires Ruby):
```
bundle install
bundle exec jekyll serve
```

## Architecture

- **Custom theme** — no gem theme dependency. All layouts, includes, and Sass are self-contained.
- **Data-driven content** — projects in `_data/projects.yml`, navigation in `_data/navigation.yml`. Add projects by editing YAML, not HTML.
- **Homepage layout** (`_layouts/home.html`) is standalone with no header nav (handled in `_layouts/default.html` via conditional).
- **Sass pipeline** — entry point is `assets/css/main.scss` (must keep empty front matter dashes). Partials in `_sass/`: `_variables`, `_base`, `_layout`, `_components`, `_syntax`.
- **Dark mode** — supported via `prefers-color-scheme: dark` media queries throughout Sass files.
- **Minimal style** — inspired by maxim.mazurok.com. No CSS frameworks, no cards/shadows, dashed link underlines, system font stack, 700px max-width.

## Adding Content

**New blog post**: Create `_posts/YYYY-MM-DD-slug-title.md` with front matter:
```yaml
---
layout: post
title: "Post Title"
date: YYYY-MM-DD
categories: [category]
tags: [tag1, tag2]
excerpt: "Short description."
---
```

**New project**: Add entry to `_data/projects.yml`.

## Plugins

Only GitHub Pages whitelisted plugins: `jekyll-feed`, `jekyll-seo-tag`, `jekyll-sitemap`.
