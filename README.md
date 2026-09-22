# nupsea.github.io

Personal portfolio, technical blog, and project showcase for **Anup Sethuram** — Data Platform Engineer exploring scalable data systems, streaming architectures, and Generative AI workflows.

Deployed live at [nupsea.github.io](https://nupsea.github.io).

---

## 🛠️ Tech Stack

- **Framework**: [Astro 5](https://astro.build) (Static Site Generation)
- **Styling**: [Tailwind CSS v4](https://tailwindcss.com) with `@tailwindcss/typography`
- **Math Rendering**: [KaTeX](https://katex.org) via `remark-math` and `rehype-katex`
- **Analytics**: Google Analytics 4 (with strict CSP rules)
- **Deployment**: GitHub Pages via GitHub Actions

---

## 📁 Project Structure

```text
/
├── public/
│   ├── blog/                   # Blog images organized by article slug
│   ├── favicon.svg             # Site favicon
│   └── talks/                  # PDF slide decks for presentations
├── src/
│   ├── content/
│   │   ├── blog/               # Technical deep-dives & Luminary articles (.md)
│   │   ├── thoughts/           # Essays on philosophy, systems, and musings (.md)
│   │   └── config.ts           # Astro content collection schemas (Zod)
│   ├── layouts/
│   │   ├── BlogPost.astro      # Article layout (TOC, progress bar, reading time)
│   │   └── Layout.astro        # Base site shell (navbar, mobile drawer, footer)
│   ├── pages/
│   │   ├── index.astro         # Homepage & featured projects
│   │   ├── projects.astro      # GenAI & Data Engineering projects catalog
│   │   ├── talks.astro         # Conferences & tech talks
│   │   ├── blog/               # Blog index with project/topic filters
│   │   └── thoughts/           # Essays and musings index
│   └── styles/
│       └── global.css          # Theme variables, prose typography, custom scrollbars
└── astro.config.mjs            # Astro integrations & Vite configuration
```

---

## ✍️ Publishing Articles (Luminary & Markdown)

This repository serves as the publication target for the **Luminary** project.

### Markdown Schema (`src/content/config.ts`)

```yaml
---
title: "Article Title"
description: "Brief summary of the article."
pubDate: "Sep 22 2026"
updatedDate: "Sep 22 2026"      # optional
project: "luminary"              # optional: links post to project series
series: "Luminary Chronicles"    # optional: series badge
tags: ["Luminary", "RAG"]        # optional: topic tags
heroImage: "/path/to/hero.png"   # optional: top banner
---
```

*Note: All fields beyond `title`, `description`, and `pubDate` are optional or have defaults. Articles with "luminary" in the title or slug are automatically attributed to the Luminary project.*

### Media & Assets
Place article images in `public/blog/<slug>/` and reference them using `/blog/<slug>/filename.png`.

---

## 🧞 Local Development

All commands are run from the repository root:

| Command | Action |
| :--- | :--- |
| `npm install` | Installs dependencies |
| `npm run dev` | Starts local dev server at `http://localhost:4321` |
| `npm run build` | Builds production-ready static files into `./dist/` |
| `npm run preview` | Previews the production build locally before pushing |

---

## 🚀 Deployment

Pushes to the `master` branch trigger the GitHub Actions workflow in `.github/workflows/deploy.yml` which builds and deploys the site to GitHub Pages.
