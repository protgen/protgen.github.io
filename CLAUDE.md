# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

ProtoMech visualization tool — a static frontend (deployed via GitHub Pages at protgen.github.io) for tracing circuits in protein language models. Paper: https://arxiv.org/pdf/2602.12026. The Colab linked from the in-app guide popup (`amirgroup-codes/ProtoMech` → `ProtoMech.ipynb`) is what generates the JSON inputs this tool consumes.

## Running

No build, no bundler, no tests, no package.json. Pure HTML/CSS/vanilla JS.

```
python -m http.server 8000
# open http://localhost:8000/index.html
```

You **must** serve over HTTP — opening `index.html` via `file://` breaks because the app `fetch()`es `examples/examples.csv` and per-example JSON files.

## Architecture

Three files do all the work: [index.html](index.html), [css/styles.css](css/styles.css), and [js/app.js](js/app.js) (~2900 lines, all globals, no modules). When editing, expect to grep within `app.js` rather than navigate a module tree.

### Data model

An "example circuit" is a directory under [examples/](examples/) containing:

- `activation_indices.json` — flat array of `[layer, pos, value, latentIdx]` tuples; indexed at load time into `activationData[layer][pos] = [{value, latentIdx}, ...]`.
- `seq.txt` — the amino acid sequence (single line).
- `top_activations.json` — per layer/latent, the top activating sequences (used by the right-side detail panel).
- `virtual_weights.json` *(optional)* — edges between latents across layers; drives the "virtual weights" overlay and the canvas's auto-edge creation.
- `canvas-state.json` *(optional)* — pre-saved canvas layout that auto-loads with the example.

[examples/examples.csv](examples/examples.csv) is the registry. Adding a new built-in example = drop a directory under `examples/<model>/<name>/`, then append a row (id, name, path, model). The model dropdown is populated from the unique `model` values; layer count is parsed from the model string with `/(\d+)\s*layer/`.

### Chunked virtual_weights (GitHub 100MB limit)

GitHub rejects blobs >100MB. [split_weights.py](split_weights.py) walks `examples/`, and for any `virtual_weights.json` over 100MB, splits it into `virtual_weights_part{N}.json` (~50MB each) plus a `virtual_weights_manifest.json` of `{"parts": N}`, then deletes the original. `loadVirtualWeights()` in `app.js` tries the manifest first and falls back to the single file, so both layouts work transparently. **Run `python split_weights.py` after adding/regenerating a large virtual_weights.json before committing**, or the push will fail.

### UI structure

Two stacked, scroll-synced views:

1. **Grid** ([app.js](js/app.js): `renderGrid`, `renderLayerLabels`, `renderSequence`) — rows = layers (top = highest layer), columns = sequence positions; each cell holds the latent boxes that activated there. Cell width is computed by `computeColumnWidths()` from the max latent count at that position across all layers, so columns vary in width.
2. **Canvas** (`renderNode`, `updateEdges`, `addNodeToCanvas`) — free-form area where the user composes a circuit by right-clicking grid latents to add nodes. Edges between canvas nodes are drawn from `virtualWeightsData`.

A separate "virtual weights overlay" (`renderVirtualWeightsInGrid`, `updateGridEdgePositions`) draws edges directly on top of the grid, gated by `virtualWeightsThreshold` (% of edges by absolute magnitude — defaults are computed to cap at ~1000 edges).

### Two load paths, kept in sync manually

The app has two entry points that both produce the same in-memory state:

- `loadExampleData(path)` — fetches files from a built-in example directory.
- The `btnLoad` click handler (line ~385) — reads `File` objects from the dropzone for "Load Custom Circuit".

Both paths duplicate the same parsing/threshold/render sequence. **If you change how data is parsed, indexed, or how virtual-weights thresholds initialize, update both code paths.** Same for `resetAppState()` which both rely on.

### Position numbering

User-visible positions = internal `pos + 1 + positionOffset` via `displayPos()`. Internally everything is 0-indexed; only the UI adds the offset. The Settings popup exposes `positionOffset` for users whose sequences are sub-sequences of a larger protein.

## Conventions

- All state is module-globals at the top of [js/app.js](js/app.js) (`activationData`, `canvasNodes`, `edges`, `virtualWeightsData`, etc.). There is no framework, no reactivity — every mutation is followed by an explicit re-render call.
- DOM is built via string concatenation + `innerHTML`, not `createElement`. Match the existing style when adding UI.
- Icons come from two libraries already loaded in [index.html](index.html): `lucide` (via `<i data-lucide="...">`, re-initialized with `lucide.createIcons()`) and Font Awesome.
