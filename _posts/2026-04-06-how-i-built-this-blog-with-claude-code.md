---
layout: post
title: "How I Built This Blog in 2 Hours with Claude Code"
date: 2026-04-06
categories: [engineering]
tags: [ai, claude-code, productivity, jekyll]
excerpt: "One prompt, 2 hours of actual work, one live site. Here's how I used Claude Code as a thinking partner and a builder to go from zero to a deployed personal blog."
---

I wanted a personal site. Not a Medium page, not a Substack — something I own, something minimal, something that shows who I am as an engineer.

But before writing any code, I needed to make a decision.

## Step 1: Choosing the Platform

I didn't just Google "best blogging platform" and pick the top result. I used Claude Code as a **thinking partner** — asked it to lay out the options, compare trade-offs, and help me reason through the decision.

The candidates:

| Platform | Pros | Cons |
|----------|------|------|
| **GitHub Pages + Jekyll** | Free hosting, custom domain optional, full control, git-based workflow, markdown posts | No CMS, manual deploys, limited to static content |
| **Vercel + Next.js** | Modern stack, SSR/SSG, great DX | Overkill for a blog, more moving parts |
| **Hugo** | Blazing fast builds | Go templating is less intuitive, smaller ecosystem |
| **WordPress** | Easy CMS, huge plugin ecosystem | Hosting cost, security overhead, not engineer-native |
| **Substack / Medium** | Zero setup | No ownership, no customisation, not a portfolio |

My decision: **GitHub Pages + Jekyll.**

Why? I'm a software engineer. I live in git. I don't need a CMS — I write in markdown. I don't need a custom domain right now — `harry-zhao-au.github.io` is fine. Free hosting, zero infrastructure, push to `main` and it's live.

The point isn't that Claude Code told me what to pick. It laid out the options clearly, and **I made the trade-off call** based on what I actually need. That's how I use AI — as a research accelerator, not a decision maker.

## Step 2: The Prompt

With the platform decided, I gave Claude Code the brief:

> I'd like to build a personal tech blog using github.io. It's my personal brand showcase — should cover my GitHub projects and my personal thoughts.

That's it. No detailed spec. No wireframes. Just the intent.

## What I Had in Mind

I knew the layout I wanted — I just needed someone to build it:

```
┌─────────────────────────────────────────┐
│              Harry Zhao                  │
│     Senior Software Engineer at ...     │
│  Distributed Systems · AI · Cloud       │
│         [GitHub] [LinkedIn] [X]         │
│          harry.zhao@email.com           │
├─────────────────────────────────────────┤
│                                         │
│  One paragraph — who I am, what I do    │
│  Links to Blog / Projects / About       │
│                                         │
├─────────────────────────────────────────┤
│  Projects                               │
│  · Project A — description (tech stack) │
│  · Project B — description (tech stack) │
│                                         │
├─────────────────────────────────────────┤
│  Always Learning                        │
│  · LeetCode stats                       │
│  · Certifications                       │
│                                         │
├─────────────────────────────────────────┤
│  Latest Posts                           │
│  · Post title 1                         │
│  · Post title 2                         │
│                                         │
└─────────────────────────────────────────┘
```

Key decisions — all mine:
- **No cards, no shadows, no hero images.** Just text, links, and whitespace.
- **Data-driven.** Projects come from a YAML file, not hardcoded HTML.
- **Dark mode.** Via `prefers-color-scheme`, not a toggle button.
- **Minimal style.** Inspired by [maxim.mazurok.com](https://maxim.mazurok.com) — dashed underlines, system fonts, 700px max-width.

## Step 3: The Build

```
 Me: "Build a personal blog on github.io"
  │
  ▼
 Claude Code: scaffolds Jekyll site
  │   _config.yml, layouts, includes, Sass pipeline
  │   Pages: home, blog, portfolio, about, contact
  │
  ▼
 Me: "I want it more minimal, like this reference site"
  │
  ▼
 Claude Code: rewrites all CSS
  │   Strips everything to bare essentials
  │   Adds dark mode, dashed links, system font stack
  │
  ▼
 Me: "Update homepage layout — profile on top, projects from YAML"
  │
  ▼
 Claude Code: rebuilds home.html
  │   Data-driven project list from _data/projects.yml
  │   Social links with inline SVG icons
  │   Learning section, latest posts
  │
  ▼
 Me: "Add my LeetCode stats, AWS cert, fix LinkedIn link"
  │
  ▼
 Claude Code: incremental fixes, each a clean commit
  │
  ▼
 Live on GitHub Pages ✓
```

Total active time: **about 2 hours.** And honestly, the most time-consuming part wasn't the coding — it was digging up all my profile links. Finding my LeetCode stats, HackerRank URL, AWS certification badge, and getting the LinkedIn URL right took longer than the entire site scaffold.

## What I Delegated vs. What I Kept

**I delegated:**
- Platform research and comparison
- Jekyll boilerplate and config
- Sass pipeline setup
- HTML structure and responsive layout
- SVG icons and social links
- Dark mode media queries

**I kept:**
- The platform decision — I evaluated the trade-offs and chose
- The visual direction — I picked the reference site, I decided "no frameworks"
- Content — my bio, my project descriptions, what goes on the homepage
- Layout decisions — profile on top, no nav on homepage, data-driven projects
- Each iteration — I reviewed every change and directed the next one

The AI did the research and wrote the code. I made every decision.

## What Surprised Me

**The bottleneck was me, not the tool.** Gathering my own content — profile links, stats, descriptions — took more time than building the site. The engineering work was essentially instant.

**How few prompts it took.** The initial scaffold was one prompt. The style overhaul was one prompt. The homepage redesign was one prompt. Most of the "work" was me looking at the result and saying "change this, keep that."

**It understood "minimal."** When I said I wanted something like the reference site, it didn't just copy — it extracted the principles (system fonts, no shadows, dashed underlines, constrained width) and applied them consistently.

## The Architecture

What Claude Code produced is well-structured:

```
Blog/
├── _config.yml
├── _data/
│   ├── projects.yml        ← add projects here, not in HTML
│   └── navigation.yml
├── _layouts/
│   ├── default.html        ← base layout with nav
│   ├── home.html           ← standalone homepage
│   └── post.html
├── _includes/
│   ├── header.html
│   └── footer.html
├── _sass/
│   ├── _variables.scss     ← colors, fonts, breakpoints
│   ├── _base.scss          ← reset + typography
│   ├── _layout.scss        ← page structure
│   ├── _components.scss    ← post list, project list, etc.
│   └── _syntax.scss        ← code highlighting
├── _posts/                 ← blog posts in markdown
└── assets/css/main.scss    ← Sass entry point
```

No gem theme dependency. No CSS frameworks. Everything is self-contained.

## The Takeaway

The real productivity gain isn't that AI writes code faster. It's that **the entire loop — from research to decision to implementation — compresses.** Instead of:

```
Research → Compare → Decide → Google → Read docs → Write code → Debug → Repeat
```

It becomes:

```
Ask → Evaluate options → Decide → Describe intent → Review output → Adjust → Done
```

I still made every decision. I still wrote every word of content. But the gap between "I want this" and "this exists" collapsed from days to 2 hours — and most of that was finding my own profile links.

When the engineering is instant, the bottleneck becomes knowing what you want. That's where experience matters more than ever.

---

*The full source is on GitHub: [harry-zhao-au.github.io](https://github.com/Harry-Zhao-AU/harry-zhao-au.github.io).*
