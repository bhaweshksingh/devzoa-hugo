# Devzoa Website

Modern static website for Devzoa, built with Hugo and deployed to GitHub Pages.

## Tech Stack

- **Hugo** - Static site generator (no Node.js dependencies)
- **Bootstrap 5** - CSS framework
- **Font Awesome 6** - Icons
- **Decap CMS** - Content management (optional)
- **GitHub Pages** - Hosting
- **GitHub Actions** - CI/CD
- **Docker** - Containerized builds

## Quick Start

### Local Development (Hugo)

```bash
# Install Hugo (macOS)
brew install hugo

# Run dev server
hugo server --buildDrafts --port 9099

# Open http://localhost:9099
```

### Docker (Production Build)

```bash
# Build and run on port 9099
docker compose up --build

# Open http://localhost:9099
```

### Docker (Development with Live Reload)

```bash
# Run with live reload on port 1313
docker compose --profile dev up devzoa-dev

# Open http://localhost:1313
```

## Project Structure

```
devzoa-hugo/
├── content/           # Markdown content
│   ├── blog/          # Blog posts
│   ├── services/      # Service pages
│   ├── about.md       # About page
│   ├── contact.md     # Contact page
│   └── privacy.md     # Privacy policy
├── data/              # JSON data files
│   ├── services.json  # Homepage services
│   └── testimonials.json  # Testimonials
├── layouts/           # HTML templates
│   ├── _default/      # Base templates
│   ├── partials/      # Reusable components
│   └── index.html     # Homepage template
├── static/            # Static assets
│   ├── css/           # Stylesheets
│   ├── js/            # JavaScript
│   └── images/        # Images and favicons
├── docker-compose.yml # Docker setup
├── Dockerfile         # Production Docker build
├── nginx.conf         # Nginx config for Docker
└── hugo.toml          # Hugo configuration
```

## Deployment

### GitHub Pages (Automatic)

1. Push to `main` branch
2. GitHub Actions builds and deploys automatically
3. Enable GitHub Pages in repo settings (Source: GitHub Actions)

### Manual Build

```bash
hugo --minify
# Output in ./public/ directory
```

## Adding Content

### New Blog Post

```bash
hugo new content blog/my-new-post.md
```

Edit the generated file in `content/blog/my-new-post.md`.

### New Service Page

Create a new file in `content/services/`:

```markdown
---
title: "New Service"
description: "Description of the service"
---

Your content here...
```

## Configuration

Edit `hugo.toml` to update:
- Site title and description
- Contact information
- Social media links
- Navigation menus
