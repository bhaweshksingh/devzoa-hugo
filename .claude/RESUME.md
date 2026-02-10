# Devzoa Hugo Site - Resume Notes

## Current State

The site is fully built and functional. All content, layouts, config, Docker, CI/CD, and README are complete.

### What's Done

- **Hugo site** fully working at `localhost:9099` (run `hugo server --port 9099`)
- **18 blog posts** with full content from Medium RSS feed and original devzoa.com (Wayback Machine)
- **5 new Medium posts** (2025-2026): Temporal/Go, LLM Gateway, OpenTelemetry, Go Strings, Data Residency
- **13 posts** from original website with images
- **Homepage**: Hero, 6 services cards, about section, 5 testimonials
- **Pages**: About, Contact, Privacy, 4 service pages
- **Navigation**: Responsive navbar with Services dropdown
- **Footer**: 4-column with address, links, services, social
- **Docker**: `docker-compose.yml` with production (port 9099) and dev (port 1313) profiles
- **CI/CD**: `.github/workflows/deploy.yml` for GitHub Pages via Actions
- **README**: Full setup/build/deploy instructions

### Blog Content Sources

- **Medium RSS**: `https://medium.com/feed/@bhaweshkumarsingh` (10 articles with full HTML content)
- **Original site**: Downloaded from Wayback Machine to `/Users/bhawesh/work/personal/devzoa-website/`
- **Blog images**: Mix of Medium CDN downloads and original site images in `static/images/blog/`

## Pending: Enable GitHub Pages

The GitHub Actions deploy workflow fails because Pages isn't enabled on the repo yet.

### Fix (requires `gh auth login` first):

```bash
# 1. Authenticate with GitHub CLI
gh auth login

# 2. Enable GitHub Pages with Actions build type
gh api repos/bhaweshksingh/devzoa-hugo/pages \
  --method POST \
  --field build_type="workflow" \
  --field source='{"branch":"main","path":"/"}'

# 3. Verify Pages is enabled
gh api repos/bhaweshksingh/devzoa-hugo/pages

# 4. Re-run the failed workflow (or just push a new commit)
gh workflow run deploy.yml
```

**Why this is needed**: The `actions/configure-pages@v5` step in the deploy workflow calls `GET /repos/{owner}/{repo}/pages` to configure the Pages site. If Pages hasn't been enabled via the API or repo settings, it returns 404 "Not Found".

**Alternative (manual via GitHub UI)**:
1. Go to https://github.com/bhaweshksingh/devzoa-hugo/settings/pages
2. Under "Build and deployment" > Source, select **GitHub Actions**
3. Save

## Pending: Decap CMS (Optional)

Decap CMS was mentioned in the original requirements but hasn't been set up yet. To add it:

1. Create `static/admin/index.html` with Decap CMS script tags
2. Create `static/admin/config.yml` with collections for blog, services, pages
3. Configure GitHub OAuth backend
4. See: https://decapcms.org/docs/add-to-your-site/

## Quick Commands

```bash
# Local dev
hugo server --port 9099 --buildDrafts

# Docker production
docker compose up --build

# Docker dev with live reload
docker compose --profile dev up devzoa-dev

# Build for production
hugo --minify

# Add new blog post
hugo new content blog/my-new-post.md
```

## File Structure

```
devzoa-hugo/
├── .claude/RESUME.md          # This file
├── .github/workflows/deploy.yml
├── content/
│   ├── blog/                  # 18 blog posts + _index.md
│   ├── services/              # 4 service pages + _index.md
│   ├── about.md
│   ├── contact.md
│   └── privacy.md
├── data/
│   ├── services.json          # 6 homepage services
│   └── testimonials.json      # 5 testimonials
├── layouts/
│   ├── _default/baseof.html   # Base template (Bootstrap 5, Font Awesome 6)
│   ├── _default/single.html   # Single page template
│   ├── _default/list.html     # List template with card grid
│   ├── partials/nav.html      # Navbar with Services dropdown
│   ├── partials/footer.html   # 4-column footer
│   └── index.html             # Homepage (hero, services, about, testimonials)
├── static/
│   ├── css/style.css
│   ├── js/main.js
│   └── images/                # logo, hero, favicons, blog images
├── hugo.toml                  # Site config
├── Dockerfile                 # Multi-stage: Hugo build -> nginx
├── docker-compose.yml         # Production + dev profiles
├── nginx.conf
└── README.md
```
