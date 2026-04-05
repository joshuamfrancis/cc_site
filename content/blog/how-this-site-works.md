---
title: "How This Site Works"
date: 2026-04-03T00:00:00-05:00
draft: false
tags: ["hugo", "static-sites", "web-development", "architecture"]
categories: ["behind-the-scenes"]
featured_image: ""
summary: "A deep dive into the architecture of this site — how Hugo templates, content, and config all wire together to produce every page you see."
---

This site is built with Hugo, a static site generator written in Go. There's no external theme — every layout, stylesheet, and component is hand-crafted. Here's how it all fits together.

## The Big Picture

Hugo takes three inputs and produces a static website:

1. **Content** — Markdown files in `/content/`
2. **Templates** — HTML files in `/layouts/`
3. **Config** — Settings in `hugo.toml`

At build time, Hugo matches each piece of content to a template, injects the rendered Markdown, and outputs plain `.html` files. No server, no database, no runtime.

## Configuration: `hugo.toml`

The root `hugo.toml` file is the source of truth for site-wide settings. This is where the navigation menu is defined:

```toml
[[menus.main]]
name = "Blog"
url = "/blog/"
weight = 1

[[menus.main]]
name = "Projects"
url = "/projects/"
weight = 2
```

Each menu entry maps a label to a URL path. Hugo doesn't automatically generate menus from your content — you wire them up explicitly here. Social links (GitHub, LinkedIn) and other site params also live in this file and are accessible in any template via `site.Params`.

## Content Structure

All written content lives under `/content/`, organized into sections:

```
content/
├── _index.md         # Homepage
├── blog/
│   ├── _index.md     # Blog section landing page
│   └── my-post.md    # Individual blog post
├── about/
│   └── _index.md
├── projects/
│   └── _index.md
└── contact/
    └── _index.md
```

Each Markdown file starts with **front matter** — a YAML block that provides metadata Hugo uses when rendering:

```yaml
---
title: "My Post Title"
date: 2026-04-04T10:00:00-05:00
draft: false
tags: ["hugo", "web"]
categories: ["tutorials"]
summary: "A short description shown on post cards."
featured_image: "/images/my-image.jpg"
---
```

The `tags` and `categories` fields feed into Hugo's taxonomy system, which auto-generates listing pages at `/tags/hugo/` and `/categories/tutorials/` — no extra work required.

## Layouts: How Content Becomes HTML

Hugo resolves which template to use based on the content's section and type. The lookup order roughly follows:

- Section-specific layout (e.g. `layouts/blog/single.html`)
- Default fallback (e.g. `layouts/_default/single.html`)

Here's the full mapping for this site:

| Content | Template |
|---|---|
| Homepage (`_index.md`) | `layouts/index.html` |
| Blog listing | `layouts/blog/list.html` |
| Individual blog post | `layouts/blog/single.html` |
| About / Projects / Contact | `layouts/{section}/list.html` |

### The Base Wrapper

Every page on the site is wrapped by `layouts/_default/baseof.html`. It defines the outer HTML shell:

```html
<!DOCTYPE html>
<html>
  {{ partial "head.html" . }}
  <body>
    {{ partial "header.html" . }}
    <main>
      {{ block "main" . }}{{ end }}
    </main>
    {{ partial "footer.html" . }}
  </body>
</html>
```

The `{{ block "main" . }}` is a slot that each child template fills in. For example, `layouts/blog/single.html` defines what goes in that slot for a blog post. All the surrounding chrome — nav bar, footer, meta tags — comes from `baseof.html` automatically.

## Partials: Reusable Components

Repeated UI elements are extracted into partials under `layouts/partials/`:

- **`head.html`** — `<meta>` tags, CSS links (with Hugo fingerprinting), dark mode init script
- **`header.html`** — Navigation bar. Loops over `site.Menus.main` and uses `IsMenuCurrent` to mark the active link
- **`footer.html`** — Social icons and copyright, pulling URLs from `site.Params`
- **`post-card.html`** — The preview card shown on the blog listing and homepage. Renders `featured_image`, title, date, reading time, and `summary`
- **`post-meta.html`** — The date, reading time, and tag badges shown on individual posts
- **`sidebar.html`** — Recent posts, tag cloud, and category list, each queried dynamically from site content

## Dynamic Content Queries

Even though the output is static HTML, Hugo templates can query your content at build time. The homepage and sidebar both pull recent blog posts using:

```go
{{ range where site.RegularPages "Section" "blog" | first 6 }}
  {{ partial "post-card.html" . }}
{{ end }}
```

This finds all pages in the `blog` section, takes the 6 most recent, and renders each as a post card. No manual curation needed — publish a new post and it appears automatically.

## Styling and Dark Mode

Stylesheets live in `/assets/css/` and are processed by Hugo's asset pipeline:

- `main.css` — Core layout and typography
- `syntax-light.css` / `syntax-dark.css` — Code block highlighting for each theme

Dark mode is handled entirely in the browser. A small script in `baseof.html` reads a `theme` key from `localStorage` on page load and sets `data-theme="dark"` on the `<html>` element. The CSS uses `[data-theme="dark"]` selectors to apply the alternate palette. The toggle button in the header writes back to `localStorage` when clicked, so the preference persists across pages.

## The Build and Deploy Cycle

Running `hugo` at the project root processes all content and templates, writing the final site to `/public/`. That directory is a complete, self-contained static site that can be hosted anywhere — the live site is served from `blog.bitcraft.stream`.

The whole thing — from Markdown to live HTML — involves no server-side code at runtime. It's just files.
