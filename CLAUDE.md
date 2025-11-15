# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Hugo-based personal blog (酱汁可乐 / colawithsauce's blog) deployed to GitHub Pages at https://blog.colawithsauce.cn/. The site is written in Chinese (zh-cn) and uses the PaperMod theme.

## Content Workflow - Org-mode to Markdown

**Critical**: This blog uses an Emacs org-mode based workflow. Blog posts are NOT directly edited in `content/posts/`.

- **Source of truth**: All blog posts originate from `content-org/all-posts.org` (and other `.org` files in `content-org/`)
- **Generated content**: The `content/posts/*.md` files are auto-generated from org-mode files using `ox-hugo` (Emacs package)
- **Emacs configuration**: The `.dir-locals.el` file configures org-mode settings:
  - `org-hugo-section`: "posts"
  - `org-hugo-base-dir`: "~/colawithsauce.github.io/"
  - `org-log-done`: t

**Workflow**:
1. Edit content in `content-org/*.org` files (especially `all-posts.org`)
2. Export to Hugo markdown using `ox-hugo` in Emacs (typically `C-c C-e H A` to export all)
3. Hugo builds the site from the generated markdown files

## Development Commands

### Local Development
```bash
# Serve the site locally with drafts
hugo server -D

# Serve without drafts (production-like)
hugo server

# Build the site (output to ./public)
hugo --minify
```

### Content Management
```bash
# Create new content (but prefer using org-mode workflow)
hugo new posts/my-post.md

# Check Hugo version
hugo version
```

### Theme Management
```bash
# Update PaperMod theme (it's a git submodule)
git submodule update --remote --merge
```

## Architecture

### Directory Structure
- `content-org/`: Org-mode source files (primary content source)
  - `all-posts.org`: Main blog post collection in org format
- `content/posts/`: Generated markdown files (DO NOT edit directly)
- `layouts/`: Custom Hugo layouts and partials
  - `_default/list.html`: Custom list template
  - `partials/`: Custom partials including MathJax support
- `themes/PaperMod/`: PaperMod theme (git submodule)
- `public/`: Generated static site (git-ignored)
- `static/`: Static assets

### Configuration
- `hugo.toml`: Main Hugo configuration
  - Base URL: https://blog.colawithsauce.cn/
  - Theme: PaperMod
  - Language: zh-cn
  - Menu items: Categories, Tags, Diary
  - Goldmark renderer with `unsafe = true` for raw HTML

### Deployment
- **Automated**: GitHub Actions workflow (`.github/workflows/hugo.yml`)
- **Trigger**: Push to `master` branch
- **Process**:
  1. Installs Hugo 0.120.4 Extended
  2. Installs Dart Sass
  3. Checks out code with submodules
  4. Builds with `hugo --minify`
  5. Deploys to GitHub Pages

### Post Metadata Format
Posts use Hugo front matter in TOML format:
```toml
+++
title = "Post Title"
author = ["colawithsauce"]
date = 2024-08-21T16:33:00+08:00
categories = ["Category1", "Category2"]
draft = false
+++
```

## Important Notes

1. **Never directly edit** `content/posts/*.md` files - they are generated from org-mode
2. When adding new content, work with `.org` files in `content-org/`
3. The blog owner uses Emacs with `ox-hugo` for content management
4. PaperMod theme is a git submodule - update carefully
5. Hugo version in CI is 0.120.4 Extended
6. Site supports raw HTML via `unsafe = true` in Goldmark renderer
7. Timezone is +08:00 (China Standard Time)
