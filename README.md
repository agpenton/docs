# Asdrubal's Corner

A personal knowledge base for DevOps engineering. Automation over repetition.
Infrastructure as code. CI/CD that actually works. Real-world knowledge —
distilled, documented, and ready to use.

**Live site:** <https://agpenton.github.io/docs/>

---

## What is this

This is a static documentation site built with [Hugo](https://gohugo.io/) and
the [Doks](https://getdoks.org/) theme (powered by the
[Thulite](https://thulite.io/) framework). It serves as a personal reference
for DevOps topics including:

- **Infrastructure as Code** — Terraform, Ansible, and Pulumi recipes for
  building repeatable, version-controlled infrastructure.
- **CI/CD Pipelines** — GitHub Actions, GitLab CI, and Jenkins patterns that
  ship code fast, safely, and automatically.
- **Cloud & Containers** — Kubernetes, Docker, AWS, GCP — architectural
  patterns and operational runbooks for cloud-native workloads.
- **Observability & SRE** — Prometheus, Grafana, OpenTelemetry, and alerting
  strategies for production confidence.

Content is organized under `content/docs/` into sections:

```
content/docs/
├── devops/      # CI/CD, cloud, containers, tools
├── IaC/         # Infrastructure as Code guides
└── topics/      # Cross-cutting topics and references
```

---

## Tech stack

| Layer                 | Technology                                            |
| --------------------- | ----------------------------------------------------- |
| Static site generator | [Hugo](https://gohugo.io/) ≥ 0.141.0 (extended)       |
| Theme                 | [Doks](https://getdoks.org/) via `@thulite/doks-core` |
| Package manager       | npm                                                   |
| Deployment            | GitHub Actions → GitHub Pages                         |
| CSS                   | Bootstrap 5 + custom SCSS via Dart Sass               |
| JS bundling           | esbuild (via Hugo Pipes)                              |
| Search                | FlexSearch (client-side, no backend)                  |

---

## Prerequisites

Install these tools before running the project locally:

### Hugo (extended, ≥ 0.141.0)

The **extended** variant is required for Dart Sass support.

```bash
# macOS
brew install hugo

# Verify — must show "extended"
hugo version
```

For Linux or Windows, download the extended binary from the
[Hugo releases page](https://github.com/gohugoio/hugo/releases). This project
was developed and tested with `v0.160.1+extended`.

### Node.js (≥ 20.11.0) and npm

```bash
# macOS — using nvm (recommended)
nvm install 20
nvm use 20

# Verify
node --version   # v20.x.x or higher
npm --version
```

### Dart Sass

Hugo uses Dart Sass for SCSS compilation (libsass is deprecated).

```bash
# macOS
brew install sass/sass/sass

# Verify
sass --version
```

---

## Local development

### 1. Clone the repository

```bash
git clone https://github.com/agpenton/docs.git
cd docs
```

### 2. Install dependencies

```bash
npm install
```

This installs the Doks/Thulite framework packages declared in `package.json`,
including `@thulite/doks-core`, Bootstrap, FlexSearch, and build tooling.

### 3. Start the dev server

```bash
npm run dev
```

The site will be available at <http://localhost:1313/docs/>. Hugo watches for
file changes and hot-reloads automatically. Use `--disableFastRender` (already
set in the script) for accurate partial re-renders.

### 4. Build for production

```bash
npm run build
```

Output is written to `public/`. The build minifies HTML, fingerprints assets,
and runs garbage collection on unused resources.

---

## Project structure

```
.
├── .github/
│   └── workflows/
│       └── pages.yml          # CI/CD: build + deploy to GitHub Pages
├── assets/
│   ├── js/
│   │   └── custom.js          # Custom JS extension point
│   └── scss/
│       └── common/
│           ├── _custom.scss   # Custom styles
│           └── _variables-custom.scss  # Bootstrap/Doks variable overrides
├── config/
│   ├── _default/
│   │   ├── hugo.toml          # Site config, baseURL, taxonomies, permalinks
│   │   ├── menus.toml         # Navbar links and social icons
│   │   ├── module.toml        # Hugo virtual filesystem mounts (npm packages)
│   │   └── params.toml        # Doks feature flags, SEO, search settings
│   └── postcss.config.js      # PostCSS with autoprefixer and PurgeCSS
├── content/
│   ├── _index.md              # Homepage content (title, lead, feature cards)
│   └── docs/
│       ├── devops/            # DevOps section
│       ├── IaC/               # Infrastructure as Code section
│       └── topics/            # Topics section
├── layouts/
│   ├── baseof.html            # Base HTML template (navbar + footer wrapper)
│   ├── home.html              # Homepage layout (hero + feature cards + credits)
│   └── _partials/
│       ├── footer/
│       │   └── script-footer-custom.html  # Custom footer scripts
│       └── head/
│           ├── head.html                  # <head> partial override
│           └── script-header-priority.html # Injects color-mode.js before paint
├── static/
│   └── images/                # Static images
├── hugo_stats.json            # HTML element catalog for PurgeCSS
├── package.json
└── package-lock.json
```

---

## Adding content

Create a new doc page with the Hugo CLI:

```bash
# Creates content/docs/<section>/<slug>.md with the correct frontmatter
npm run create -- docs/<section>/<slug>.md
```

Or create the Markdown file manually. Each page should include this frontmatter:

```yaml
---
title: "Page Title"
description: "One-sentence description (110–160 characters for SEO)."
summary: ""
date: 2024-01-01T00:00:00+00:00
lastmod: 2024-01-01T00:00:00+00:00
draft: false
weight: 10
toc: true
seo:
  title: "" # leave empty to use page title
  description: "" # leave empty to use description
  noindex: false
---
```

Set `draft: false` when ready to publish. Pages with `draft: true` are excluded
from production builds but visible in `npm run dev`.

---

## Configuration

Key settings are split across files in `config/_default/`:

| File          | What it controls                                                     |
| ------------- | -------------------------------------------------------------------- |
| `hugo.toml`   | Site title, base URL, pagination, taxonomies, permalink patterns     |
| `params.toml` | Color mode, navbar button, FlexSearch, section nav, versioning flags |
| `menus.toml`  | Top navigation links (`[[main]]`) and social icons (`[[social]]`)    |
| `module.toml` | npm package mounts into Hugo's virtual filesystem                    |

To change the site title or base URL, edit `config/_default/hugo.toml`:

```toml
title   = "Your Site Title"
baseurl = "https://<your-username>.github.io/<repo-name>/"
```

---

## Deployment

The site deploys automatically to GitHub Pages on every push to `main` via the
workflow at `.github/workflows/pages.yml`.

The workflow:

1. Installs Hugo extended `0.147.0` and Dart Sass
2. Runs `npm ci` to install npm dependencies
3. Runs `npm run build -- --baseURL "<pages-url>/"` (base URL injected by the
   configure-pages action)
4. Uploads `public/` as a Pages artifact and deploys it

To deploy to your own GitHub Pages:

1. Fork or clone this repo
2. In your repo, go to **Settings → Pages → Source** and set it to
   **GitHub Actions**
3. Update `baseurl` in `config/_default/hugo.toml` to match your Pages URL:
   ```toml
   baseurl = "https://<your-username>.github.io/<repo-name>/"
   ```
4. Push to `main` — the workflow runs automatically

No secrets or tokens are required; the workflow uses the built-in
`GITHUB_TOKEN`.

---

## Color mode

The site supports light mode, dark mode, and automatic (follows the OS
preference). The toggle button is in the top-right of the navbar.

- The preference is stored in `localStorage` under the key `theme`
- On page load, `color-mode.js` is injected synchronously into `<head>` to
  apply the correct theme before any paint, preventing a flash of wrong theme
- The OS preference is respected automatically when no stored preference exists,
  and re-applied live when the OS switches

---

## License

MIT — see [LICENSE](LICENSE) for details.  
Content © 2024 Asdrubal Gonzalez Penton.
