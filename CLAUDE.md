# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Zenn publication repository for writing and managing technical articles and books. Zenn is a Japanese technical content platform similar to Medium or dev.to.

## Commands

```bash
# Preview content locally in browser
npx zenn preview

# Create new article
npx zenn new:article

# Create new book
npx zenn new:book

# List all articles
npx zenn list:articles

# List all books
npx zenn list:books
```

## Repository Structure

- `articles/` - Markdown files for individual articles (one file per article)
- `books/` - Book directories, each containing `config.yaml` and chapter markdown files
- `images/` - Image assets referenced in articles/books

## Article Format

Articles use frontmatter with these fields:
```yaml
---
title: "Article Title"
emoji: "🙌"
type: "tech"  # tech: 技術記事 / idea: アイデア
topics: [Topic1, Topic2]
published: false  # true to publish
publication_name: "org_name"  # optional, for organization publishing
---
```

## Book Format

Each book is a directory containing:
- `config.yaml` - Book metadata (title, summary, topics, chapters order)
- `cover.png` - Cover image
- Chapter markdown files listed in config.yaml

## Image References

Images are stored in `images/` and referenced as `/images/path/to/image.png` in markdown.
