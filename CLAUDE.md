# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

The companion website for the **Affinity Network Workshop** (a four-day workshop on network design, Bolzano, July 6–9, 2026). It's a small Jekyll site with three pages, and everything analytical runs **client-side in the browser** — nothing is precomputed, committed, or uploaded:

- **Home** (`index.html`, permalink `/`) — workshop description, learning outcomes, approach, and a day-by-day schedule.
- **Text** (`text.html`, permalink `/text/`) — the "microscope": drop in **one** document and read it in layers (counts, readability, keywords, key phrases, and where terms appear).
- **Network** (`network.html`, permalink `/network/`) — the "telescope": drop in a **set** of text files and watch them arrange into a similarity network. Embedding, similarity, and force layout all run in the browser.

> History: the site previously rendered a *pre-baked* publication graph computed offline in Python. That is gone, along with its `_publications/` abstracts and `scripts/`. If you find references to `_data/network.json`, a `network` layout, or "baked positions", they're stale — nothing in the current site produces or consumes them.

## Commands

```bash
bundle install
bundle exec jekyll serve            # http://localhost:4000/
bundle exec jekyll build --quiet    # sanity-check a change builds
```

There are no tests. Deployment is automatic: pushing to `main` triggers `.github/workflows/deploy.yml`, which builds with Jekyll and publishes to GitHub Pages. The Pages `--baseurl` is derived from the repo name in CI (`steps.pages.outputs.base_path`), so renaming the repo needs no config change. Locally `baseurl` is empty (serves at root).

## Architecture

Standard Jekyll. One layout, `_layouts/default.html`, wraps every page: it builds `<head>`, inlines the CSS (`{% include %}` of `nunito.css`, `main.css`, then each sheet named in the page's `styles:` front matter), renders `{% include site-nav.html %}`, and holds the shared dark-mode toggle.

**Each tool page is a single inline `<script type="module">`** — there is no build step and no shared JS file. Styling splits across `_includes/`: `network.css` (graph visuals: nodes, links, labels, stage — `#network-view`, `.node`, `.link`), `studio.css` (the builder chrome shared by both tools: control columns, sliders, toggles, overlays, dropzone), and `text.css` (the Text page's analysis layout). Note the `studio.*` filename/class names are a leftover from when the builder page was called "Studio"; the name stuck.

### The Network builder (`network.html`)

On **Build network**:

1. **Read** dropped `.txt`/`.md` files. Each is a node; its name is the file's first Markdown heading (`# …`) via `titleFromText()`, else the filename.
2. **Embed** each doc with `Xenova/multilingual-e5-small` through Transformers.js (`@huggingface/transformers@3` from the CDN), mean-pooled and normalized. Multilingual model → no translation step needed. The model (~110 MB) is fetched from the HF hub, browser-cached, and **preloaded** as soon as files are staged (`warmModel()`), so the load overlaps with the user reviewing files.
3. **Similarity** — pairwise cosine (`computeSimilarity()` builds a dense N×N matrix).
4. **Links** — `buildLinks(threshold)`: each node links to its single strongest neighbour, only if that score clears the live **Connection threshold**.
5. **Layout** — a live `d3.forceSimulation` the user can drag; forces map to the sliders.

**Live controls** (all mutate the running simulation, no rebuild):

- **Threshold** → rebuilds links; **Repulsion** → `charge` strength; **Gravity** → `forceX`/`forceY` strength toward centre; **Node spacing** → `collide` radius; **Link distance** → `forceLink` distance.
- **Node labels** / **Shorten titles** toggles; hover always shows the full title.
- **Stats overlay** (top-right of the graph): files, edges, clusters (union-find over links), unconnected, strongest/avg similarity, and "built in Ns". Recomputes live on threshold change.
- **PNG export** at 1×/2×/4×: `exportPNG(scale)` clones the SVG, inlines colours/fonts (external CSS doesn't apply to an SVG-in-`<img>`), and rasterises at the scaled size for crisp output. Label fonts may fall back to a system sans in the export.
- A built-in `SAMPLE_DOCS` corpus lets the page be tried without user files.

### The Text analyzer (`text.html`)

On **Analyze**, a single dropped `.txt`/`.md` file (or a bundled sample) is read and broken into five sections, each with a "How it's built" / "Algorithm" aside explaining the method:

1. **Overview** — word/sentence/paragraph counts, average sentence length, lexical diversity (type–token ratio), hapax count, and a ~200 wpm reading-time estimate.
2. **Readability** — Flesch reading-ease and Flesch–Kincaid grade level (syllables estimated from vowel groups; the heuristic assumes English), plotted on a labelled ease scale.
3. **Keywords** — ranked single terms.
4. **Key phrases** — ranked multi-word phrases.
5. **Where terms appear** — positional distribution of selected terms across the document.

Pure string/statistics work — no model download, so it runs instantly.

## Conventions

- Vanilla ES5-style JS in the inline modules (function expressions, no build step). Keep it dependency-free beyond D3 (vendored, `js/d3.v7.min.js`) and Transformers.js (CDN, Network page only).
- CSS uses design tokens from `main.css` (`--fg`, `--bg`, `--muted`, `--border`, `--text-xs`, `--tracking-caps`, …) and supports light/dark via `html.dark`.
- After a JS change, a quick check is `node --check` on the extracted module plus `jekyll build`.
