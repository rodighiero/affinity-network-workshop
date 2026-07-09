# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

The companion website for the **Affinity Network Workshop** (a four-day workshop on network design, Bolzano, July 6–9, 2026). It's a small Jekyll site with six pages, and everything analytical runs **client-side in the browser** — nothing is precomputed, committed, or uploaded:

- **Home** (`index.html`, permalink `/`) — workshop description, learning outcomes, approach, and a day-by-day schedule.
- **Text** (`text.html`, permalink `/text/`) — the "microscope": drop in **one** document and read it in layers (measurements, vocabulary, grammar, entities, readability, sentiment, keywords, recurring phrases, and a paragraph-by-paragraph narrative arc). English only.
- **Exercises** (`exercises.html`, permalink `/exercises/`) — a static lab of eight close-reading tasks for the Text page, used in the workshop's text-analysis session. Pure content (no JS); styled by `exercises.css`. Each task card maps to Text-page sections and pairs a *Task* with what it teaches (*Learn*); the last few point toward the Network page.
- **Network** (`network.html`, permalink `/network/`) — the "telescope": drop in a **set** of text files and watch them arrange into a similarity network. Embedding, similarity, and force layout all run in the browser.
- **Audio** (`audio.html`, permalink `/audio/`) — the telescope for **sound**: drop in a **set of short audio clips** and watch them arrange into a similarity network by how they *sound*. Embedding (CLAP), similarity, and force layout all run in the browser; click a node to hear its clip. Reuses the Network page's similarity/force-layout engine.
- **Compare** (`compare.html`, permalink `/compare/`) — two documents side by side: drop **one file in each of two slots**, and each *paragraph* becomes a node in a similarity network, coloured by its source document. Cross-document links are drawn as prominent "bridges" (shared themes = *common language*), within-document links stay faint (each text's own texture = *specific language*), and a three-column panel below the graph lists each document's own vocabulary and the words they share. Reuses the Network page's embedding/similarity/force-layout engine.

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

### The Audio page (`audio.html`)

A near-copy of the Network builder for **sound** instead of text, so it shares `network.css` + `studio.css` (plus its own `audio.css`) and the same `#network-view` markup, force simulation, elevation map, zoom, and PNG/SVG export. The differences are all about audio:

- **Input** — dropped `.wav`/`.mp3`/`.ogg`/`.m4a` files (or the synthesized sample set). Each clip is decoded to a **mono Float32Array at 48 kHz** by `fileToPCM()` (`AudioContext.decodeAudioData` + an `OfflineAudioContext` for resample/downmix — done at *stage* time, so the file list can show durations). Every clip is a node `{ name, url, pcm, duration }`, where `name` is the filename (no `titleFromText`), `url` is an object URL for playback, and the array is `clips` (playing the role `docs` plays on the Network page).
- **Embedding** — **CLAP** (`Xenova/clap-htsat-unfused`) via Transformers.js, loaded through `AutoProcessor` + `ClapAudioModelWithProjection` (the `dtype: 'q8'` audio tower only, ~34 MB — *not* the 500–600 MB text/combined ONNX). `warmModel()` preloads both as soon as clips are staged. Per clip: `processor(pcm)` → `audioModel(...)` → `audio_embeds` (a fixed **512-dim** vector), L2-normalized (`l2normalize()`, since CLAP output isn't pre-normalized) so cosine is a dot product. Everything downstream — `computeSimilarity`, `buildLinks`, the force graph, stats, export — is reused unchanged.
- **Click-to-play** — a single shared `Audio` element; clicking a node plays that clip (clicking the playing one stops it), gated on "not dragged" via a small movement threshold in the `d3.drag` handlers. The playing node carries a `.playing` class (accent `--playing` in `audio.css`).
- **Sample set** — synthesized in-browser (`SAMPLE_SPECS` → `synthClip()` → `encodeWAV()`), so **no binary audio is committed**. Five families (pure tones, chords, noise, rhythmic pulses, gliding sweeps) that CLAP clusters cleanly.
- No keyword/edge-keyword layer (audio has no vocabulary); the Network page's TF-IDF code is dropped here, and only the **Label size** label control is kept (labels are just filenames). CLAP pads/truncates to a fixed ~10 s window, so short clips work best.

### The Compare page (`compare.html`)

A near-copy of the Network builder, so it shares `network.css` + `studio.css` (plus its own `compare.css`) and the same `#network-view` markup, embedding pipeline, force simulation, elevation map, zoom, and PNG/SVG export. The differences are all about **two documents, compared at the paragraph level**:

- **Input** — two dropzones (`stageFile('A'|'B', …)`), each holding a single file into `docA` / `docB`. On build, `buildParas()` runs `splitParagraphs()` (blank-line split, lifted from the Text page, minus any leading Markdown heading) over each document; every paragraph becomes a node `{ text, doc, name }`, where `name` is a `snippet()` of the opening words. These paragraph nodes — not whole files — are what get embedded and laid out (the array is `paras`, playing the role `docs` plays on the Network page).
- **Links** — `buildLinks(threshold)` gives each paragraph **two** candidate edges: its strongest neighbour *within* its own document and its strongest *cross*-document neighbour, each kept only above the threshold. Cross-document edges are tagged `kind:'bridge'` and styled prominently (`.link--bridge`); within-document edges stay faint. Bridges are the **common language**; the faint within-document web is each text's **specific language**.
- **Colour** — nodes carry `.node--a` / `.node--b`; the two accent colours are the CSS custom properties `--doc-a` / `--doc-b` (set on `#network-view` in `compare.css`, light/dark variants), read back by the export code so PNG/SVG match. A small A/B legend sits over the graph.
- **Vocabulary panel** — a three-column section below the graph (`#vocab-panel`): *A-only · common · B-only*. `computeVocab()` tokenises the two **full** documents (same `TOKEN_RE` + `STOPWORDS` as the Network page) and splits terms into shared (in both) vs. exclusive (in one), ranked by raw count and sized by weight. Note: the Network page's TF-IDF `computeKeywords`/`edgeKeywords`/`nodeKeywords` are **not** used here — with only two documents a shared word's IDF is `log(2/2)=0`, so a plain frequency count is used instead. There is no edge-keyword layer on this page.
- A built-in `SAMPLE_A` / `SAMPLE_B` pair (two takes on "the city") lets the page be tried without user files.

### The Text analyzer (`text.html`)

Input is a dropped `.txt`/`.md` file, text typed/pasted into the box beside the dropzone, or the bundled sample — all three feed the same staged text (files and the sample also fill the paste box). On **Analyze** it's broken into nine sections. Each has a right-hand aside — one short paragraph explaining the left column, with a single reference/library link, top-aligned to the figure rather than the heading — and each stat card / figure caption carries a plain-language definition shown as a styled tooltip on hover/focus (`STAT_DESC` + the generic `[data-tip]` CSS, also used by grade bars, keyword/phrase captions and paragraph headers):

1. **Measurements** — word/sentence/paragraph/character counts, average sentence length, longest sentence, average word length, and a ~200 wpm reading-time estimate, plus a *sentence-length* histogram (5-word buckets).
2. **Vocabulary** — unique words, lexical diversity (type–token ratio), lexical density (content words ÷ all words), and average uses per word, plus a *word-frequency spectrum* histogram (how many words occur 1×, 2×, … 11+×).
3. **Grammar** — noun : verb, adjectives : noun, and adverbs : verb ratios, plus active/passive **voice** (sentence-level via `#Passive`), a stacked *grammatical composition* bar (nouns/verbs/adjectives/adverbs/other, with a note that "other" is function words + punctuation).
4. **Entities** — distinct people/places/organizations counts, plus chip lists of the actual named entities per type (deduped, with a mention count when a name recurs; heuristic, so expect stray words).
5. **Readability** — Flesch reading-ease, syllables/word, and complex-word (3+ syllable) and long-word (>6 letter) shares, plotted on a labelled ease scale, plus a *grade-level by formula* bar chart comparing Flesch–Kincaid, Gunning fog, SMOG and Coleman–Liau on a fixed 0–24 axis (syllables estimated from vowel groups; assumes English).
6. **Sentiment** — lexicon-based tone from **AFINN** (`js/afinn.js`, bundled): summed word valences (a preceding negator flips the sign), shown as overall tone / net score-per-100 / positive & negative counts, a Negative–Neutral–Positive meter, and chip lists of the contributing words.
7. **Keywords** — two lists: *most frequent* (raw count) and *most distinctive* (count × the word's rank in general English, from `js/en-freq.js`, so terms rare in English but common here rise). Both over lemmatised nouns/verbs; top 10 with a "show top 20" toggle.
8. **Recurring phrases** — noun phrases (head noun singularised), split on stopwords into runs of ≥2 content words (kept if ≥2×), shown two ways: *most frequent* and *strongly bound* (PMI collocation), as pills sized by their metric.
9. **Paragraph by paragraph** — the document's narrative arc as a table, one row per paragraph: words, reading ease (a diverging bar around 0, since Flesch can go negative), sentiment (diverging), and the three most distinctive words. Regex tokens + AFINN + keyness only — no per-paragraph compromise parse, so it stays instant.

Two small bundled data assets under `js/` (imported via Liquid `relative_url` so the baseurl is right in CI): **`en-freq.js`** (top-5000 rank-ordered English words, for keyword/paragraph distinctiveness) and **`afinn.js`** (AFINN-165 sentiment lexicon).

**compromise** (`compromise@14` from esm.sh, imported by the inline module) is created once and drives most of the page: it provides the shared **word/sentence token base** for Measurements and Vocabulary (via `doc.terms()` and `doc.sentences()`, so every section counts the same way — only paragraphs and characters stay plain, since compromise has no concept of them), plus part-of-speech tagging, voice detection, named-entity recognition, and lemmatisation for Grammar, Entities, Keywords and Recurring phrases. So the whole page needs a one-time CDN fetch (~200 KB, browser-cached), then runs instantly. Readability's syllable heuristic, Sentiment, and the Paragraph table are the non-compromise analyses. The whole page assumes **English**. Key helpers: `renderStats` (stat cards + tooltips), `renderBars`/`renderPhrasePills` (keyword bars, sized phrase pills), `renderSentHist`/`renderFreqSpectrum`/`renderGradeBars` (histograms and bar charts), `renderGrammar` (composition bar), `renderEntities`/`entityList` (entity chips, reused for sentiment words), `renderSentiment` (meter), `paragraphRow`/`renderParagraphs` (narrative table), `buildLemmatizer` (surface→lemma map — nouns singularised in one whole-doc pass, verbs infinitivised per unique form because the whole-view `toInfinitive()` is O(n²)-slow), and `keywordCounts`/`keyphraseCounts`/`lemmatizedTerms` (all take the compromise `doc` plus that lemmatizer).

## Conventions

- Vanilla ES5-style JS in the inline modules (function expressions, no build step). Keep it dependency-free beyond D3 (vendored, `js/d3.v7.min.js`), Transformers.js (CDN, Network page only), and compromise (CDN, Text page only).
- CSS uses design tokens from `main.css` (`--fg`, `--bg`, `--muted`, `--border`, `--text-xs`, `--tracking-caps`, …) and supports light/dark via `html.dark`.
- After a JS change, a quick check is `node --check` on the extracted module plus `jekyll build`.
