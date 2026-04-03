# Blog Site Tasks

## Requirements

- [x] Generate a blog site that will have its source files in markdown format.
- [x] The site will be published as static pages with GitHub Pages.
- [x] There will be a pipeline that will build and refresh the GitHub Pages on pull request to main branch.
- [x] Content authoring will be performed locally. The site should be runnable locally to author content, validate final sites prior to push to GitHub.
- [x] The site should have capabilities like in site https://www.noidea.dog/blog/staff-engineer-communities
- [x] The project should be created in the current folder.

## Implementation Notes

### Approach
- **Static site generator:** Hugo (single binary, no Node.js required, blazing fast builds)
- **Custom theme:** Built inline (no external theme dependency) for full control over design
- **CSS:** Custom stylesheet processed via Hugo Pipes (minified + fingerprinted)
- **CI/CD:** GitHub Actions with official GitHub Pages deployment actions

### Design (inspired by noidea.dog/blog)
- Clean, minimal aesthetic with light background and dark text
- Sticky horizontal navigation bar (Blog, Projects, About, Contact)
- Blog listing with sidebar (recent posts, tags, categories)
- Single post view with metadata (date, reading time, tags), prev/next navigation
- Responsive design with hamburger menu on mobile
- Footer with social links (GitHub, Email, etc.)
- System font stack for instant loading

### File Structure
```
cc_site/
├── hugo.toml                          # Site configuration
├── .gitignore
├── .github/workflows/deploy.yml       # CI/CD pipeline
├── archetypes/default.md              # Template for new posts
├── assets/css/main.css                # Custom stylesheet
├── content/
│   ├── blog/                          # Blog posts (markdown)
│   ├── about/                         # About page
│   ├── projects/                      # Projects page
│   └── contact/                       # Contact page
├── layouts/
│   ├── _default/baseof.html           # Base HTML skeleton
│   ├── _default/single.html           # Generic page template
│   ├── _default/list.html             # Generic list/taxonomy template
│   ├── blog/single.html               # Blog post template
│   ├── blog/list.html                 # Blog listing template
│   ├── partials/                      # Reusable template fragments
│   ├── index.html                     # Homepage
│   └── 404.html                       # Custom 404 page
└── static/images/                     # Static assets
```

### Pipeline (`.github/workflows/deploy.yml`)
- **On pull request to main:** Runs Hugo build (`hugo --minify`) to validate site compiles successfully
- **On push to main:** Builds site + deploys to GitHub Pages via `actions/deploy-pages@v4`
- Uses `peaceiris/actions-hugo@v3` for Hugo setup
- Concurrency control prevents overlapping deployments

### Local Development
```bash
# Start dev server with live reload (includes draft posts)
hugo server --buildDrafts

# Create a new blog post
hugo new blog/my-new-post.md

# Build for production
hugo --minify
```

### Before First Deploy - GitHub Setup Required
1. Create a GitHub repository and push this code
2. Go to **Settings > Pages** in the repo
3. Set Source to **GitHub Actions** (not "Deploy from a branch")
4. Update `baseURL` in `hugo.toml` to match your actual GitHub Pages URL (e.g., `https://<username>.github.io/<repo>/`)

## Status: COMPLETE
All tasks have been implemented and verified locally. The site builds cleanly (34 pages, ~70ms) and renders correctly with the Hugo dev server.
