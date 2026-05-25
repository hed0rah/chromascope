# Architecture

How ArtScope is shaped and why. Read this before changing any of the cross-page contracts.

## Four pages, one config, one shell

The site is four pages. All four share one nav, one design-token file, one painting-config file, and one theme system. They differ in what they do with that shared context.

```
                        painting.json
                        tokens.css
                        forensic.css (index only)
                              |
              +-------+-------+-------+--------+
              |       |       |       |        |
              v       v       v       v        v
        index.html  explorer.html  lab.html  system.html
        (forensic)  (deep zoom)    (color)   (bonus)
              |       |       |       |        |
              +-------+---+---+---+---+--------+
                          |       |
                          v       v
                      tokens.css  assets/
                                  (images, tiles,
                                   thumbnails)
```

**index.html** is the forensic deep dive. The painting, the verdict on the claims attached to it, the real-science backdrop, the in-house image analysis, the canonical scans, the toolkit, the structural critique of the misinformation that prompted the document, the cited sources. Painting-specific prose throughout; only the *structure* is reusable.

**explorer.html** is the deep-zoom annotated viewer. OpenSeadragon serving the DZI tile pyramid with single-image fallback. Hotspot overlays, geometry grid, minimap, contrast filter drawer (the "do the AI trick yourself" experience), region inspector with nearest-pigment classification, viewport permalinks. Almost entirely config-driven.

**lab.html** is the color analysis lab. Drag a rectangle on the painting, get RGB and HSV histograms, k-means dominant colors (top 8), classification into the palette pigments by nearest Euclidean RGB distance, full color statistics, plus a horizontal compare strip that holds up to six saved regions side by side. The hotspots from `painting.json` show up as quick-select buttons. Fully config-driven.

**system.html** is the bonus: the derived design system. Twelve pigments sampled from the painting and named, two themes, components reference, design tokens listing. Demoted from front page (it used to be `index.html`) because the painting deserves the front door. Config-driven.

## Why this shape

### Why a single config file
Because the painting is the project's center of gravity, and everything else (palette, hotspots, measurements, themes, brand identity) hangs off it. One JSON file means one source of truth, one diff to review when adding a painting, one place to look when debugging a runtime issue.

### Why painting-specific prose in `index.html` but config-driven everything else
Because forensic prose is irreducibly per-painting. The claims attached to *Las Meninas* aren't the claims attached to *The Last Supper*. The verdict, the real-science section, the structural-critique angle, the citation list — all rewrites, not parameter swaps. Pretending otherwise would mean a "templating system" so generic it would produce slop.

The explorer, the lab, and the system page are different. They operate on *coordinates and colors*, both of which are well-defined JSON-serializable values. So they get the same shell across paintings; only the input changes.

This split is the boundary between "engine" and "deployment." Engine is everything in this repo. Deployment is engine + a populated `painting.json` + a populated `index.html` + the asset bundle.

### Why CSS variables for the palette but hand-edited tokens.css
Because the palette IS the design — twelve named pigments in a specific ordering pulled from the painting. Auto-deriving them from `painting.json` at runtime via injected CSS variables is possible (and listed in HANDOVER section 6.A as future work) but adds a layer of indirection that nobody wanted when the project started. For now: extract the palette via `scripts/build_assets.py --refresh-palette`, then hand-edit `tokens.css` to wire the variables.

### Why no build step
Because the project is small enough that introducing webpack/vite/parcel buys nothing. Vanilla HTML, vanilla CSS, vanilla JS, three Python scripts that anyone with Pillow and NumPy can run. The cost of editing files directly is lower than the cost of maintaining a build pipeline.

### Why em-dashes are out
Because the project is partly about not laundering AI-generated content as serious writing, and em-dashes are an unmistakable AI tell. Periods, commas, parens, colons, semicolons cover every actual case. The constraint is enforced in the prose, not the code.

## Data flow

### Page load (any of the four)

```
1. HTML parses
2. tokens.css loads (page-styled, dark theme by default)
3. (index only) forensic.css loads
4. JS runs:
   - Restore theme from localStorage if present
   - fetch('painting.json')
   - Render: nav brand, hero, measurements, palette, hotspots, etc.
   - Bind event handlers
5. (explorer / lab) Load painting image into OpenSeadragon / canvas
6. Idle, awaiting interaction
```

### Lab analysis flow

```
User drags rectangle on painting canvas
         |
         v
setSelection({x, y, w, h})
         |
         v
analyzeSelection()
   |
   |---> ctx.getImageData(x, y, w, h)  --> ImageData
   |
   |---> Compute in one pass:
   |       per-channel histograms (R, G, B)
   |       HSV hue histogram (skip washed-out)
   |       running sums for mean RGB, HSV, luminance
   |       circular mean for dominant hue
   |
   |---> kmeansColors(data, k=8, iter=8)  --> dominant colors
   |
   |---> classifyPigments(data, palette)  --> pigment distribution
   |
   v
Renderers:
   renderSelectionInfo, renderStats, renderRgbHist,
   renderHueWheel, renderDominant, renderPigments
```

K-means and pigment classification each downsample to keep latency under 50ms even on a whole-painting selection. K-means: every Nth pixel up to 8000 samples. Pigment classification: every Nth pixel up to 20000 samples. Both adequate for visual results.

### Tile pyramid generation flow

```
scripts/make_tiles.py source.jpg -o output -n basename
         |
         v
Load source via Pillow
         |
         v
Compute level count: log2(max(W, H) / tile_size) + 1
         |
         v
For each level from full resolution down to 1x1:
   For each tile (256x256 default):
      Crop, save as basename_files/<level>/<col>_<row>.jpg
         |
         v
Write basename.dzi XML manifest pointing at the file structure
```

OpenSeadragon consumes the `.dzi` manifest and auto-fetches tiles as the user pans and zooms. The pyramid is essential for sub-second response on gigapixel scans; without it, every viewport change would require re-downloading the full image.

## File contracts (don't break)

### `painting.json` keys consumed by each page

| Field | index | explorer | lab | system |
|---|---|---|---|---|
| `site_name`, `site_tagline` | nav brand only | yes | yes | yes |
| `painting.*` | embedded in prose | meta bar | n/a | meta bar |
| `scan.url`, `scan.fallback_url` | n/a | yes (load) | yes (load) | n/a |
| `scan.width`, `scan.height` | n/a | overlay viewBox | source-pixel readout | n/a |
| `motif_image`, `motif_caption` | n/a | n/a | n/a | hero |
| `hero_bg_image` | n/a | n/a | n/a | hero background |
| `design_principles.use` | n/a | n/a | n/a | conditional section |
| `measurements.*` | embedded in prose | n/a | n/a | measurements table |
| `extraction_method.*` | n/a | n/a | n/a | numbered pipeline |
| `pigments[]` | n/a | inspector classifier | quick swatches + classifier | palette grid |
| `themes.*` | toggle label | toggle label | toggle label | toggle label |
| `hotspots[]` | n/a | overlay + sidebar | quick-select buttons | n/a |

If you add a field, mention it in `docs/SCHEMA.md`. If you remove a field, audit this table.

### Cross-page navigation contract

The nav has four links plus a theme toggle. Order: Analysis, Explorer, Lab, System (bonus). The "active" page link has a primary-color underline. The bonus tag on System is a small italic qualifier rendered via `.nav-bonus` styled in tokens.css.

The brand on the left always reads `<site_name> // <site_tagline>` and links to `index.html`. The site is single-domain; no external links from the brand.

### Theme contract

Two themes: `dark` (default) and `light`. Persisted to `localStorage` under key `cenacolo-theme` (yes, the original site name is baked into the key; future deployments can change this to their site name or leave it).

Themes are toggled by setting `data-theme="dark"` or `data-theme="light"` on the `<html>` element. All four pages read the same key and use the same toggle button. tokens.css defines both theme blocks as `:root, [data-theme="dark"] { ... }` and `[data-theme="light"] { ... }`.

## Why four pages and not one big SPA

Because the four pages do genuinely different things and benefit from being independently shareable, deep-linkable, and progressively loadable. The forensic essay should be reachable at a clean URL without booting an OpenSeadragon instance. The explorer should be reachable without paying the cost of analysis-page CSS. The lab should be navigable from the hotspots regardless of whether the user has the painting open in the explorer tab.

Also: SPAs add a build step, a router, a bundler. ArtScope is meant to be readable in a text editor by someone who has never seen the project. A flat page-per-feature design satisfies that goal.

## What this architecture deliberately doesn't do

- **Auto-derive the palette into CSS variables at runtime.** Listed as future work. The current setup hand-edits `tokens.css` after running the palette extraction script. The friction is real but small (12 hex values).
- **Persist lab compare cards across sessions.** Compare cards live in-memory on the lab page and clear on navigation. Saving them to localStorage would be ~30 lines but raises questions about how to identify regions across painting swaps.
- **Render hotspots from `painting.json` on the index page.** The forensic page is for prose, not config readouts. The explorer is the right home for hotspot UI.
- **Support multiple paintings per deployment.** One painting per ArtScope deployment. A multi-painting gallery would need a different shell.
- **Translate.** The site is English-only. Adding i18n would require splitting prose out of `index.html` into JSON, which would defeat the purpose of inlining painting-specific writing.

## Maintenance pressure points

If something is going to break or get awkward as the project ages, it's likely to be one of these:

1. **`painting.json` schema drift.** Every page reads it, no schema validation runs. A bad config produces silent runtime errors. Mitigation: keep `docs/SCHEMA.md` in sync, run a manual schema lint when shipping a new painting.

2. **CSS variable drift between `tokens.css` and a refreshed palette.** The variables in `:root` reference pigment names; if you re-extract a palette and the new pigments have different names, the variable wiring breaks silently (CSS just falls back to the variable's default). Mitigation: lift the wiring to JS, or keep pigment naming stable across re-extractions.

3. **OpenSeadragon CDN dependency.** If jsDelivr serves a different version, the viewer behavior could change subtly. Mitigation: pin the version explicitly (already done in explorer.html) and consider vendoring.

4. **`run_analysis.py` assumptions.** The script finds the geometric centroid of skin-tone pixels in a central band. Works well for *The Last Supper* (frontal figures, central composition). Less well for paintings without central figures. Mitigation: refactor into a strategy pattern with per-painting analyzers, or expose the band parameters via config.

5. **Lab `getImageData` CORS.** When the painting is served from a different domain than the page (rare, but possible if you point `scan.fallback_url` at a CDN), `getImageData` will throw a SecurityError. Mitigation: serve same-origin, or proxy through your site.

These are listed not because they will break, but so that when something *does* go strange, you know where to look first.
