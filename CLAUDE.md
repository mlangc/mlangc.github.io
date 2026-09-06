# CLAUDE.md

This file provides guidance to AI coding assistants when working with code in this repository.

## Overview

This is a Jekyll-based personal blog hosted on GitHub Pages at https://mlangc.github.io/. The content focuses on Java/JVM topics (concurrency, benchmarking, memory models).

## Project Layout

All Jekyll source lives under `docs/`. The repository root contains `README.md`, `LICENSE`, and `.github/` (the Pages build workflow and Dependabot config).

- `docs/_posts/` — blog posts in Markdown with `.markdown` extension
- `docs/assets/drawings/` — diagrams as `.drawio` source files with pre-exported `.drawio.png` companions
- `docs/assets/img/` — benchmark screenshots and other images
- `docs/_config.yml` — Jekyll configuration (theme, plugins, site metadata)
- `docs/Gemfile.lock` — committed; carries a `CHECKSUMS` section so CI resolves the exact gem versions tested locally
- `docs/_site/` — generated output, not committed (gitignored)
- `.github/workflows/pages.yml` — builds the site with Jekyll and deploys it to GitHub Pages
- `.github/dependabot.yml` — weekly update PRs for the `bundler` and `github-actions` ecosystems

## Development Commands

All commands must be run from the `docs/` directory:

```bash
cd docs

# Serve locally with live reload
bundle exec jekyll serve

# Build without serving
bundle exec jekyll build

# Install gems from the committed lockfile
bundle install
```

The local server runs at http://localhost:4000 by default.

`docs/Gemfile.lock` is committed and the CI build runs with `BUNDLE_FROZEN`, so
it fails if the lockfile is out of sync and verifies every gem against the
`CHECKSUMS` section. Any change to `Gemfile` or `Gemfile.lock` (including a local
`bundle update`) must be committed. Routine gem bumps normally arrive as
Dependabot PRs rather than manual updates.

## Blog Post Format

Posts use the naming convention `YYYY-MM-DD-slug.markdown` and require this front matter:

```yaml
---
layout: post
title: "Post Title"
date: YYYY-MM-DD
categories: Java Concurrency   # space-separated, no brackets
excerpt: "One-sentence summary shown on the home page."
---
```

## Diagrams

Diagrams are created with [draw.io](https://app.diagrams.net/). Each diagram is stored as:
- `assets/drawings/YYYY-MM-DD-name.drawio` — editable source
- `assets/drawings/YYYY-MM-DD-name.drawio.png` — exported PNG referenced in posts

When updating a diagram, export an updated PNG alongside the `.drawio` source.

## Build and Deployment

The site is built and deployed by the `.github/workflows/pages.yml` GitHub Actions
workflow running Jekyll 4.4.x — not the `github-pages` gem. The workflow runs on
every push to `main` (and via `workflow_dispatch`), builds from `docs/`, and
publishes to the `github-pages` environment. Every action in the workflow is
pinned to a full commit SHA.

## Theme and Configuration

The site uses the `minima` theme via `jekyll-remote-theme`, pinned to a full
commit SHA (not a branch or tag) in `docs/_config.yml` so the fetched theme
content is immutable. `show_excerpts: true` is set so the `excerpt` front matter
field appears on the home page.

## Role of AI Assistants

You have these roles in this repository:
* You are a critical reviewer of my blog posts. Don't modify any files in this context unless explicitly asked.
* You are helping with the setup of the blog.

Please don't commit unless explicitly asked and confirm commit messages with me. They should summarize changes at a high level
and rarely exceed one or two sentences in the message body.
