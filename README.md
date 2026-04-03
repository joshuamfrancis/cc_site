# blog.bitcraft.stream

A personal blog built with [Hugo](https://gohugo.io/) and deployed to GitHub Pages at [blog.bitcraft.stream](https://blog.bitcraft.stream). Topics include technology, engineering, and ideas. The site uses a custom theme with light/dark mode support, a sidebar, tag/category taxonomies, and post navigation.

## Prerequisites

- [Hugo Extended](https://gohugo.io/installation/) v0.135.0 or later
- Git

## Creating New Content

### New blog post

```bash
hugo new blog/your-post-title.md
```

This generates `content/blog/your-post-title.md` pre-populated with front matter from the archetype. Open the file and fill in the fields:

```yaml
---
title: "Your Post Title"
date: 2026-04-03T10:00:00-05:00
draft: true          # set to false when ready to publish
tags: ["tag-one", "tag-two"]
categories: ["category"]
featured_image: ""   # optional: relative path to an image
summary: "One-sentence description shown in post cards."
---

Your content here in Markdown.
```

Posts stay unpublished while `draft: true`. Change it to `draft: false` before pushing to make them live.

### Content sections

| Section | Path | Command |
|---|---|---|
| Blog post | `content/blog/` | `hugo new blog/<slug>.md` |
| Project | `content/projects/` | edit `content/projects/_index.md` |
| About | `content/about/` | edit `content/about/_index.md` |
| Contact | `content/contact/` | edit `content/contact/_index.md` |

## Validating Locally

Start the development server with draft posts included:

```bash
hugo server --buildDrafts
```

Open [http://localhost:1313](http://localhost:1313) in your browser. The server watches for file changes and reloads automatically.

To preview exactly what will be published (drafts excluded, minified output):

```bash
hugo --minify
```

The built site is output to `public/`. You can serve it statically with any local HTTP server to do a final check before pushing.

## Pushing to the Remote Repository

Commits pushed to the `main` branch trigger the GitHub Actions pipeline (`.github/workflows/deploy.yml`), which builds the site with Hugo Extended and deploys it to GitHub Pages automatically.

```bash
# Stage your changes
git add .

# Commit with a descriptive message
git commit -m "add post: your post title"

# Push to main — deployment starts immediately
git push origin main
```

The workflow will:
1. Build the site with `hugo --minify`
2. Inject the `CNAME` file for the custom domain
3. Upload the artifact and deploy to GitHub Pages

Monitor the deploy under the **Actions** tab of the repository. The live site updates within a minute of the workflow completing.
