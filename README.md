# ChromaScope

A single-file forensic color lab for paintings, scans, photos, and screenshots. Drop in any image, zoom into it, read pixel values live, drag-select regions for full color statistics, and run forensic filters to test "hidden detail" claims yourself.

No build step. No framework. No CDN. Everything is inline in `index.html`. Open it in a browser or serve the folder.

## Run it

```
# option 1: just open the file
open index.html        # works in Chrome / Firefox for local images

# option 2: serve the folder (needed for cross-origin URL loading)
python -m http.server 8000
# then open http://localhost:8000
```

Load an image three ways: the Load button, drag-drop anywhere on the page, or paste from the clipboard (Ctrl+V). You can also pull one from a URL, though many hosts block cross-origin pixel reads.

Nothing uploads anywhere. All processing is in the browser.

## What it does

**Explore.** Deep-zoom viewport with wheel zoom, pan, fit, 1:1, a minimap, and a live inspector HUD that reads RGB / HSV / luminance and source-pixel coordinates as you move the cursor.

**Measure.** Click two points for pixel distance, percent-of-width, and angle. Useful for checking compositional claims directly.

**Analyze.** Drag-select any region and get mean RGB and HSV, luminance mean / std / range, dominant hue, an RGB histogram, a polar hue-distribution wheel, and k-means dominant colors (click any swatch to copy the hex).

**Forensic filters** (display only, analysis always reads the raw source):
- display: brightness, contrast, saturation, hue-rotate, invert, grayscale
- shadow stretch: the "AI reveals hidden text" trick, expand the dark range and watch noise and craquelure appear out of nothing
- threshold, posterize, Sobel edges
- single-channel isolation (R / G / B)
- fake IR / fake UV, clearly labelled as visual simulation, not multispectral capture
- mirror flip and ghost-blend to inspect symmetry

**Palette.** Extract a 12-color palette from the whole image (k-means), rename the swatches, then classify any selection against them as a distribution of nearest pigments.

**Regions.** Save selections to a compare strip with thumbnails and dominant-color swatches. Rename them, click a thumbnail to fly back to it. Saved regions carry normalized coordinates.

**Export.** Palette as CSS custom properties or JSON, or a `painting.json` skeleton (saved regions become hotspots, palette becomes pigments) that feeds the engine below.

### Keyboard

`V` select, `H` pan (or hold Space), `I` inspect, `M` measure, `P` eyedropper, `+` / `-` zoom, `0` fit, `1` actual size, `Esc` clear.

## How it handles big images

The image is drawn to an internal analysis canvas capped at 2200 px on the long side so a gigapixel source does not blow browser memory. Analysis runs against that canvas; the inspector and selection report coordinates back in original-source pixels. For pixel-perfect work at native resolution, use a real toolchain (Pillow / NumPy), not a browser.

## The engine bridge (phase 2)

ChromaScope is the standalone studio and the curation front-end for **ArtScope**, a config-driven engine that turns one `painting.json` plus an asset bundle into a four-page deep-dive site (forensic essay, deep-zoom explorer, color lab, design system). The studio's Export panel emits the `painting.json` skeleton the engine consumes.

The engine seed lives alongside this file:

- `index.template.html` forensic-essay shell with placeholder prose
- `painting.template.json` config schema with inline notes
- `docs` equivalents in the repo root:
  - `SCHEMA.md` the `painting.json` field reference
  - `ARCHITECTURE.md` the four-page information architecture and data flow
  - `ADDING_A_PAINTING.md` the step-by-step recipe
  - `HANDOVER.md` the deep background on the original Last Supper project

The engine still needs its app pages (explorer, lab, system, tokens) and the tile / palette scripts ported in from the Last Supper working copy before it can render. That is the next phase. The studio stands on its own today.

## License

MIT.
