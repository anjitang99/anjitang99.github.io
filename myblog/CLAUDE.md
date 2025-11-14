# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a personal Jekyll blog for 박도언 (Park Do-eon), hosted on GitHub Pages. The blog contains study records, thoughts, and investment/development notes, primarily written in Korean.

## Tech Stack

- **Static Site Generator**: Jekyll 4.4.1
- **Theme**: Minima 2.5
- **Plugins**: jekyll-feed
- **Deployment**: GitHub Pages (assumed from repository structure)

## Development Commands

### Local Development
```bash
# Install dependencies
bundle install

# Run local development server (with auto-reload)
bundle exec jekyll serve

# Build the site (output to _site/)
bundle exec jekyll build
```

**Important**: The development server must be restarted when `_config.yml` is modified, as it is not auto-reloaded.

### Creating New Posts

Posts must be created in the `_posts/` directory with the naming convention:
```
YEAR-MONTH-DAY-title.markdown
```

Required front matter:
```yaml
---
layout: post
title: "Your Post Title"
date: YYYY-MM-DD HH:MM:SS +0900
categories: category-name
---
```

## Project Structure

```
myblog/
├── _config.yml          # Jekyll configuration (requires server restart on changes)
├── _posts/              # Blog posts (YYYY-MM-DD-title.markdown format)
├── index.md             # Homepage (Korean language, custom layout)
├── about.markdown       # About page
├── 404.html             # Custom 404 page
├── Gemfile              # Ruby dependencies
└── Gemfile.lock         # Locked dependency versions
```

## Content Guidelines

### Language
- Primary content is in Korean
- Code and technical terms may use English
- Blog owner: 박도언 (working at Suhyup Bank Investment Finance Division)

### Blog Topics
- Study records (공부기록)
- Thought organization (생각정리)
- Investment and development records (투자·개발 기록)
- Real estate development investment
- Financial planning and credit analysis
- Fintech (IT and finance convergence)
- Swift/iOS development learning
- Credit analyst exam preparation

## Jekyll-Specific Notes

### Theme Customization
- Using Minima theme (can be overridden by creating files in root or `_layouts/`)
- Default layout pages can be found in the gem's directory

### Front Matter
- Custom `layout` is used in index.md (set to "default")
- Standard posts use `layout: post`
- Korean title support: "도언의 블로그"

### Build Configuration
- Base URL: "" (root)
- No special build exclusions configured
- Standard Jekyll feed plugin enabled

## Timezone
All dates use Korea Standard Time (+0900 timezone offset).
