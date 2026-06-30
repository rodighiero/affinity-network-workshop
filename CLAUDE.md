# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A standalone Jekyll site that renders a **publication similarity graph** with D3. Each node is a publication, positioned near the papers it most resembles. Similarity is computed **offline** from each paper's title and abstract; the browser only fit-scales a pre-baked layout into the viewport and draws edges — no force simulation runs client-side.

## Commands

```bash
# View the site (no build step needed — all data is committed)
bundle install
bundle exec jekyll serve            # http://localhost:4000/publication-network/

# Regenerate the graph after editing _publications/*.md
pip install -r requirements.txt     # numpy, pyyaml, sentence-transformers, transformers
KMP_DUPLICATE_LIB_OK=TRUE python3 scripts/build-network.py   # needs Node.js on PATH

# Run just the offline layout step against existing data
node scripts/layout-network.js < _data/network.json
```

There are no tests. Deployment is automatic: pushing to `main` triggers `.github/workflows/*.yml`, which builds with Jekyll and publishes to GitHub Pages.

## Architecture

The graph is **precomputed offline and committed** (`_data/network.json`); the page is a thin renderer. Two non-obvious split points to keep in mind:

1. **`scripts/layout-network.js` is the single source of truth for graph geometry.** All force constants, the seeded RNG (LCG), canvas size (`856×600`), `STRONG_SIM = 0.70`, and `NODE_RADIUS = 3` live here. `_layouts/network.html` reuses `NODE_RADIUS` for drawing only and never runs layout math — it just scales the baked `x`/`y` into the live stage (`fitLayout()`, object-fit: contain). If you change geometry, change it here and re-bake.

2. **The pipeline order** (`scripts/build-network.py`): parse Markdown → strip footnotes/headings/links/code/bibliography → machine-translate non-English abstracts → embed title+abstract with `BAAI/bge-base-en-v1.5` (768-dim, normalized) → pairwise cosine similarity → shell out to `layout-network.js` to bake positions → write `network.json`. Python calls Node via `subprocess`; the Python and JS sides must agree on the data contract (`{nodes, similarity, translations}` in, `{canvas, positions, links}` out).

### `_data/network.json` contract
- `nodes`: each has `slug`, `title`, `url`, baked `x`/`y`, and optional `tr: true` (translation node).
- `similarity`: dense N×N cosine matrix, diagonal zeroed. The page reads full rows for the "three closest" panel.
- `links`: each node's **single strongest** neighbour, only if that best score clears `STRONG_SIM`; otherwise the node is unconnected.
- `canvas`: the `{w, h}` the positions were baked into.

### Translations (`translation_of` frontmatter)
A publication with `translation_of: <slug>` is the same work in another language. It joins the layout as a regular node but its **only** edge is a forced `1.00` dashed link to its original, and it is never a similarity-link candidate for any other node (so an original links to its strongest *distinct* neighbour, not its own translation). The pipeline validates that every `translation_of` points to an existing, non-translation publication and aborts otherwise.

### Publications source
`_publications/*.md` are source data for the pipeline **only** — Jekyll never renders them (underscore-prefixed, and `scripts/` is in `_config.yml` `exclude`). Each file has YAML frontmatter (`title`, optional `lang` defaulting to `en`, optional `translation_of`) followed by a Markdown abstract. Node `url`s are site-relative slugs; the renderer prefixes them with `SITE_BASE` (`https://dariorodighiero.com`, defined in `_layouts/network.html`).

## Gotchas
- `baseurl: "/publication-network"` in `_config.yml` scopes asset/font URLs for the GitHub Pages **project** subpath. Clear it if moving to a domain root.
- Translation cache (`_data/translations-cache.json`) is keyed by content hash + lang, so editing a non-English abstract re-translates; unchanged ones are reused.
- The whole site is one layout (`_layouts/network.html`) containing markup, CSS (`{% include %}`'d from `_includes/`), and the D3 renderer inline.
