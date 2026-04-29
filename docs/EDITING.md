# BS-Cloth Project Page — Editing Guide

Every editable spot in `index.html` is marked with `TODO:` so you can find them
fast with Ctrl+F or `grep -n TODO docs/index.html`.

---

## 1. Hero — title, authors, venue

**File:** `docs/index.html`  
**Search:** `TODO: BS-Cloth title`

Replace:
```html
<h1 class="title is-1 publication-title">BS-Cloth: TODO Full Paper Title Here</h1>
```
with your actual paper title.

Then fill in author names and links inside the `<div class="is-size-5 publication-authors">` blocks:
- Replace `Author One`, `Author Two`, etc.
- Set `href="#"` to each author's personal/lab page.
- Add or remove `<span class="author-block">` entries as needed.
- Replace affiliation `<sup>` labels.
- Set the venue (`TODO: Venue Year` → e.g. `SIGGRAPH 2026`).

---

## 2. Resource buttons (arXiv, Paper, Code, …)

**Search:** `TODO: links`

Each button is a `<span class="link-block">` block.  
- Replace `href="#"` with real URLs (arXiv, OpenReview/PDF, GitHub, dataset, etc.).
- Add extra buttons by duplicating a block and changing the icon class (FontAwesome
  or Academicons: `ai ai-arxiv`, `fas fa-file-pdf`, `fab fa-github`, `fas fa-database`, …).

---

## 3. Teaser image and caption

**Search:** `TODO: teaser`

1. Drop your teaser figure at `docs/static/images/teaser.png`.
2. Edit the caption inside `<h2 class="subtitle has-text-centered">`.

---

## 4. Abstract

**Search:** `TODO: abstract`

Replace the three `<p>TODO …</p>` paragraphs with your paper's abstract text.

---

## 5. Method figure and pipeline bullets

**Search:** `TODO: method figure`

1. Replace `docs/static/images/pipeline.png` with your method diagram.
2. Update the three `<li>TODO …</li>` bullets to match your pipeline stages.

---

## 6. Demo video

**Search:** `TODO: demo video`

1. Drop your demo video at `docs/static/videos/demo.mp4`.
2. The `<source src="./static/videos/demo.mp4">` line already points there — no
   HTML change needed unless you rename the file.
3. Update the card title and description paragraph above the `<video>` tag.

---

## 7. Applications section

**Search:** `TODO: populate videoItems`

The Applications section is populated from two JavaScript arrays near the bottom
of `index.html`:

- **`videoItems`** — video-based application cards. Each entry: `{ title, description, filename }`.
  Drop `.mp4` files in `docs/media/` and set `filename` to match.
- **`imageApplications`** — image-based application cards. Each entry: `{ title, description, image: { src, alt, caption } }`.
  Drop images in `docs/static/images/` and update `src`.

Add or remove array entries freely; the page rebuilds automatically.

---

## 8. BibTeX

**Search:** `TODO: bibtex`  (or look for the `<pre><code>@article{bscloth` block)

Replace the placeholder fields with your actual BibTeX entry.

---

## 9. Re-enabling the Baseline Comparison table

The comparison section is commented out. To re-enable it:

1. Uncomment the `<section class="section" id="results">` block in `index.html`.
2. Also uncomment the `<a class="navbar-item" href="#results">Results</a>` nav link.
3. Export each method's output mesh as a `.glb` file and place it at:
   `docs/static/models/<slug>/<method>/output.glb`
4. Update the `methods` and `experiments` arrays in the `<script>` block:
   - `methods` — an array of method key strings matching your folder names.
   - `experiments` — array of `{ slug, prompt }` objects; `slug` must match the folder name.
5. If a result is missing/failed, add `"<slug>/<method>"` to the `missingResults` Set.

---

## 10. Styling

- **Project-specific styles** (colors, card borders, layout tweaks): `docs/stylesheet.css`
- **Bulma framework defaults** (buttons, columns, hero, etc.): `docs/static/css/bulma.min.css`
  — do not edit Bulma directly; override in `stylesheet.css` instead.
- **Page-level overrides and component styles**: the `<style>` block inside `<head>` in `index.html`.

---

## Meta-tags (SEO / social sharing)

Update in `<head>`:
- `<title>`, `<meta name="description">`, `<meta name="author">`, `<meta name="keywords">`
- All `og:` and `twitter:` tags
- `<link rel="canonical">`
- The `<script type="application/ld+json">` structured-data block (optional, helps Google)

Replace `TODO-your-github-username` with your actual GitHub username throughout.
