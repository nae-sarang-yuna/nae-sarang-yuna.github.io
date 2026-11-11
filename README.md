# GitHub Pages: static homepage at `/` and a Quarto blog at `/blog`

The approach is a single repo where GitHub Actions renders the Quarto blog and combines it with your static homepage, then deploys everything as one site. This avoids the usual conflicts you get from `quarto publish gh-pages`, which wants to own the whole site.

**Final structure:**
```
my-site/
├── index.html              ← homepage (served at /)
├── style.css
├── blog/                   ← Quarto project (served at /blog/)
│   ├── _quarto.yml
│   ├── index.qmd
│   ├── about.qmd
│   └── posts/
└── .github/workflows/publish.yml
```

---

## Step 0: Prerequisites

Install:
- [Git](https://git-scm.com/) and a GitHub account
- [Quarto](https://quarto.org/docs/get-started/) (check with `quarto --version`)

---

## Step 1: Create the repo

On GitHub, create a new **public** repository (private works only on paid plans).

- Named **`<your-username>.github.io`** → site lives at `https://<username>.github.io/`
- Any other name, e.g. `my-site` → site lives at `https://<username>.github.io/my-site/`

Then clone it:
```bash
git clone https://github.com/<username>/<repo>.git
cd <repo>
```

---

## Step 2: Create the homepage

Create `index.html`:
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>My Site</title>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <main>
    <h1>Hi, I'm Your Name</h1>
    <p>Welcome to my homepage.</p>
    <nav>
      <a href="blog/">Read the blog →</a>
    </nav>
  </main>
</body>
</html>
```

Create `style.css`:
```css
body { font-family: system-ui, sans-serif; margin: 0; }
main { max-width: 40rem; margin: 4rem auto; padding: 0 1rem; line-height: 1.6; }
a { color: #0a58ca; }
```

> Use **relative links** (`blog/`, not `/blog/`). That way the site works whether it is hosted at the domain root or under `/<repo>/`.

---

## Step 3: Create the Quarto blog

From the repo root:
```bash
quarto create project blog blog
```

This creates a `blog/` folder with a ready-made blog template (`_quarto.yml`, `index.qmd`, `about.qmd`, `posts/`).

Preview it locally:
```bash
quarto preview blog
```

Edit `blog/_quarto.yml` to set your title and, optionally, a link back to the homepage:
```yaml
project:
  type: website

website:
  title: "My Blog"
  navbar:
    right:
      - text: "Home"
        href: ../index.html
      - about.qmd
      - icon: github
        href: https://github.com/<username>
format:
  html:
    theme: cosmo
    css: styles.css
```

Quarto's output uses relative paths, so it works under `/blog/` without extra configuration.

---

## Step 4: Add a `.gitignore`

Create `.gitignore`:
```
_site/
.quarto/
/.quarto/
*_files/
.DS_Store
```

(Rendered output shouldn't be committed, since the Action builds it.)

---

## Step 5: Add the deployment workflow

Create `.github/workflows/publish.yml`:
```yaml
name: Build and deploy site

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: quarto-dev/quarto-actions/setup@v2

      - name: Render Quarto blog
        run: quarto render blog

      - name: Assemble site
        run: |
          mkdir -p _site
          cp index.html style.css _site/
          cp -r blog/_site _site/blog

      - uses: actions/upload-pages-artifact@v3
        with:
          path: _site

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

If your homepage has more files (images, JS, an `assets/` folder), add them to the `cp` lines, e.g. `cp -r assets _site/`.

---

## Step 6: Push to GitHub

```bash
git add .
git commit -m "Initial site: homepage + Quarto blog"
git push origin main
```

(If your default branch is `master`, either rename it to `main` or change `branches: [main]` in the workflow.)

---

## Step 7: Enable GitHub Pages

1. On GitHub, go to your repo → **Settings** → **Pages**
2. Under **Build and deployment → Source**, choose **GitHub Actions**

If your first workflow run failed because Pages wasn't enabled yet, go to the **Actions** tab and re-run it (or push a small commit).

---

## Step 8: Verify

1. Open the **Actions** tab and wait for the workflow to go green.
2. Visit:
   - `https://<username>.github.io/` → your static homepage
   - `https://<username>.github.io/blog/` → your Quarto blog

---

## Day-to-day workflow

**Write a new post:**
```bash
mkdir blog/posts/my-new-post
```
Create `blog/posts/my-new-post/index.qmd`:
```markdown
---
title: "My new post"
author: "Your Name"
date: "2026-11-11"
categories: [news]
---

Hello world!
```
Then preview with `quarto preview blog`, commit, and push. The Action redeploys automatically.

---

## Gotchas and troubleshooting

- **Blog posts with code (Python/R/Jupyter):** the Action needs those environments. Easiest fix is to render locally and use freeze. Add this to `blog/_quarto.yml`:
  ```yaml
  execute:
    freeze: auto
  ```
  Then commit the generated `blog/_freeze/` folder. The Action will reuse stored results instead of re-executing the code. (Alternatively, add Python/R setup steps to the workflow.)
- **`/blog` vs `/blog/`:** GitHub Pages redirects automatically, but link to `blog/` with the trailing slash.
- **Broken CSS or links on the blog:** this is usually caused by absolute paths (starting with `/`). Switch to relative paths.
- **404 on the whole site:** confirm Pages source is set to **GitHub Actions** and that the workflow succeeded.
- **Jekyll interfering:** not an issue here, since you're deploying via Actions rather than the legacy branch method.
- **Custom domain (optional):** in Settings → Pages, add your domain, and put a `CNAME` file containing it into `_site/` (add `cp CNAME _site/` to the assemble step).