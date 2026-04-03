---
title: "Building a Deployment Pipeline with GitHub Actions"
date: 2026-04-02T14:00:00-05:00
draft: false
tags: ["github-actions", "ci-cd", "devops", "tutorial"]
categories: ["devops"]
featured_image: ""
summary: "Learn how to set up an automated build and deployment pipeline for your static site using GitHub Actions and GitHub Pages."
---

Continuous deployment is essential for any modern web project. In this post, we'll set up a GitHub Actions pipeline that automatically builds and deploys our Hugo site whenever changes are merged to the main branch.

## The Pipeline Overview

Our pipeline has two main stages:

1. **Build** - Triggered on every pull request to validate the site builds correctly
2. **Deploy** - Triggered when changes are pushed to `main`, deploying to GitHub Pages

## Setting Up the Workflow

Create a file at `.github/workflows/deploy.yml` in your repository:

```yaml
name: Build and Deploy
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
```

This ensures our pipeline runs on both PRs (for validation) and pushes to main (for deployment).

## The Build Job

The build job installs Hugo, builds the site, and uploads the artifact:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: '0.135.0'
          extended: true
      - run: hugo --minify
```

## The Deploy Job

The deploy job only runs on the main branch and uses GitHub's official Pages deployment:

```yaml
  deploy:
    if: github.ref == 'refs/heads/main'
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/deploy-pages@v4
```

## Configuring GitHub Pages

Don't forget to configure your repository:

1. Go to **Settings > Pages**
2. Set the source to **GitHub Actions**
3. Update your `baseURL` in `hugo.toml` to match your Pages URL

## Benefits of This Approach

- **PR validation** - Catch build errors before merging
- **Automated deployment** - No manual steps needed
- **Rollback capability** - Revert to any previous commit
- **Build caching** - Fast builds with dependency caching

With this setup, your workflow becomes: write content, open a PR, review, merge, and your site is live!
