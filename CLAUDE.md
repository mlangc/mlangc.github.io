# CLAUDE.md

This file provides guidance to AI coding assistants when working with code in this repository.

## Overview

This is a Jekyll-based personal blog hosted on GitHub Pages at https://mlangc.github.io/. The content focuses on Java/JVM topics (concurrency, benchmarking, memory models).

## Project Layout

All Jekyll source lives under `docs/`. The repository root contains only `README.md` and `LICENSE`.

- `docs/_posts/` — blog posts in Markdown with `.markdown` extension
- `docs/assets/drawings/` — diagrams as `.drawio` source files with pre-exported `.drawio.png` companions
- `docs/assets/img/` — benchmark screenshots and other images
- `docs/_config.yml` — Jekyll configuration (theme, plugins, site metadata)
- `docs/_site/` — generated output, not committed (gitignored)

## Development Commands

All commands must be run from the `docs/` directory:

```bash
cd docs

# Serve locally with live reload
bundle exec jekyll serve

# Build without serving
bundle exec jekyll build

# Install/update gems
bundle install
bundle update
```

The local server runs at http://localhost:4000 by default.

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

## Theme and Configuration

The site uses the `minima` theme via `jekyll-remote-theme` pinned to a specific commit (`jekyll/minima@1e8a445`). `show_excerpts: true` is set so the `excerpt` front matter field appears on the home page.

## Role of AI Assistants

Your primary role is that of a critical reviewer. Don't modify any files unless explicitly asked.
