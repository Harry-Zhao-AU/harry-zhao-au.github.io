# harry-zhao-au.github.io

Personal portfolio and blog site built with Jekyll, hosted on GitHub Pages.

**Live**: https://harry-zhao-au.github.io

## Sections

- **Home** — profile, projects, learning platforms, latest posts
- **Blog** — software engineering articles and tutorials
- **Portfolio** — project showcase (data-driven via `_data/projects.yml`)
- **About** — professional summary, skills, and experience
- **Contact** — email, GitHub, LinkedIn

## Tech Stack

- Jekyll (GitHub Pages native)
- Custom Sass styling (no CSS frameworks)
- Dark mode support
- Responsive design

## Local Development (Optional)

GitHub Pages builds automatically on push. For local preview:

```bash
bundle install
bundle exec jekyll serve
# http://localhost:4000
```

## Adding a Blog Post

Create a file in `_posts/` with the naming convention `YYYY-MM-DD-title.md`:

```yaml
---
layout: post
title: "Your Title"
date: YYYY-MM-DD
categories: [category]
tags: [tag1, tag2]
excerpt: "Short description."
---

Your content in Markdown.
```

## Adding a Project

Edit `_data/projects.yml`:

```yaml
- name: "Project Name"
  description: "What it does."
  technologies: ["Lang1", "Lang2"]
  github: "https://github.com/Harry-Zhao-AU/repo-name"
  live: ""
```
