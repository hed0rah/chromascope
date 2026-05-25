# HANDOVER &middot; Cenacolo / Lets_Eat_lastsupperanalysis

A static site whose primary product is a forensic deep dive into Leonardo's *Last Supper* and a deep-zoom annotated explorer over the same painting. A derived design system (palette + tokens + components, all sampled from the painting) ships as bonus material at `/system.html`. The site is designed to be **template-ified**: swap `painting.json` plus the asset set, and the same shell can host an equivalent deep dive on any other painting.

**Project directory:** `C:\Users\joshr\PROJECTZ\Lets_Eat_lastsupperanalysis\`
**Deploy target:** grivt domain (static, no server side needed)
**Stack:** vanilla HTML + CSS + JS, no build step, OpenSeadragon for the deep-zoom viewer
**Authorship:** hed0rah with claude (Anthropic) as research + drafting collaborator

---

## 0. State of the project

What's working and shipped in `site_v2/`:

| File | State | Notes |
|---|---|---|
| `index.html` | new front page | The deep-dive forensic. The painting, the verified/distorted/fabricated claims table, real-science section, the in-house image analysis, scans, tools, and the structural pattern of the AI-content-farm hoax. Authored: hed0rah + claude. |
| `explorer.html` | v2 | DZI tile source with single-image fallback. Minimap, filter drawer (brightness/contrast/saturate/invert), region inspector (click to read RGB/HSV/nearest pigment), viewport permalink, expanded keyboard nav. |
| `lab.html` | new | The color analysis page. Drag any rectangle on the painting (or click a hotspot to jump to one) and get: full color statistics, RGB histogram, hue distribution wheel, k-means dominant colors, classification into the palette pigments, plus a compare strip that holds up to six regions side by side. Fully config-driven. |
| `system.html` | bonus | Derived design system: 12 pigments sampled from the painting, two themes, components reference, tokens listing. Demoted from front page; lives at /system.html as a side product. Config-driven. |
| `tokens.css` | v1, hardcoded palette | Currently encodes the Cenacolo (Last Supper) palette directly. To template-ify properly, palette should be lifted into config-driven CSS vars. See section 6. |
| `painting.json` | v2 | Single source of truth for everything painting-specific. Schema documented in section 3. |
| `make_tiles.py` | done | DZI pyramid generator. PIL only. Tested. |
| `build_assets.py` | done | Rebuilds pigment + hotspot thumbnails from the config. Optional `--refresh-palette` re-samples hex values from the source. |
| `run_analysis.py` | done | Regenerates the three forensic figures (geometry overlay, stretch comparison, mirror comparison) plus the measured-values data block. Run on the 9600 source to upgrade the analysis page from the 5381-px figures. |
| `assets/last_supper.dzi` + `assets/last_supper_files/` | from 5381 source | Replace by regenerating from the 9600 source. See section 4. |

What's NOT done:
- `index.html` (the forensic) is still tied to the Last Supper. The prose, the claims table, and the citations are painting-specific by nature. For a second painting these are a content rewrite, not a config change. See section 6.B.
- `tokens.css` palette colors are hardcoded. For a second painting they'd need to be regenerated from the new palette. See section 6 for the lift.
- The 9600px tile pyramid hasn't been generated yet (the network sandbox can't reach upload.wikimedia.org). See section 4.

---

## 1. File structure

```
Lets_Eat_lastsupperanalysis/
|
|-- index.html              <-- forensic deep dive (homepage)
|-- explorer.html           <-- deep-zoom annotated viewer
|-- system.html             <-- design system bonus (was old homepage)
|-- tokens.css              <-- shared CSS variables, components, layout
|-- painting.json           <-- THE config, single source of truth
|-- make_tiles.py           <-- DZI pyramid generator (run locally once)
|-- HANDOVER.md             <-- this file
|
|-- assets/
|   |-- last_supper.dzi             <-- DZI manifest (tile pyramid)
|   |-- last_supper_files/          <-- tile directories, levels 0..N
|   |   |-- 0/  1/  2/  ... 14/
|   |-- last_supper_full.jpg        <-- fallback single image
|   |-- motif.jpg                   <-- hero motif (Christ crop)
|   |-- hero_bg.jpg                 <-- hero background atmosphere strip
|   |-- texture_tile.jpg            <-- plaster craquelure (200x200 tile)
|   |-- pattern_band.jpg            <-- frieze ornament for borders
|   |-- analysis_geometry.jpg       <-- compositional overlay (analysis page)
|   |-- analysis_stretch.jpg        <-- contrast-stretched comparison
|   |
|   |-- pigments/                   <-- per-pigment sample-region thumbs
|   |   |-- refectory_cream.jpg
|   |   |-- linen_tabula.jpg
|   |   |-- ... (12 total, snake_case of pigment name)
|   |
|   |-- hotspots/                   <-- per-hotspot crops shown in sidebar
|       |-- jesus.jpg
|       |-- john-peter-judas.jpg
|       |-- ... (10 total, matches hotspot.id)
|
|-- archives/                       <-- previous versions / scratch
```

**File naming conventions:**
- Pigment thumbnails: `assets/pigments/<name.lower().replace(' ', '_')>.jpg`. Pigment objects in `painting.json` may also carry a `sample_thumb` override.
- Hotspot thumbnails: `assets/hotspots/<hotspot.id>.jpg`. The `id` field is the hyphenated slug.
- Tile pyramid: `assets/<painting_id>.dzi` + `assets/<painting_id>_files/`. The painting id from `painting.json` becomes the base name.

---

## 2. Architecture overview

The site is **three static pages backed by one JSON config**. Pages fetch the config at load time and populate everything painting-specific. Nothing dynamic, no build step, no framework.

```
                  +-------------------+
                  |   painting.json   |   <-- only thing you edit per painting
                  +-------------------+
                       |       |       |       |
            +----------+       |       |       +----------+
            v                  v       v                  v
       index.html         explorer.html  lab.html      system.html
       (forensic         (deep zoom)    (color         (design system
        deep dive)                       analysis)      bonus)
            |                  |              |              |
            +----------+-------+------+-------+------+-------+
                       |                            |
                       v                            v
                  tokens.css                    assets/
                  (design                       (images, tiles)
                   system)
```

**The contract:**
- `painting.json` is the single source of truth for: painting metadata, scan info, palette (12 pigments), hotspots (10 regions), measurements, themes, motif/hero captions.
- `tokens.css` defines structural tokens (typography, spacing, layout, components) AND the painting's palette as CSS variables. The HTML pages don't know about colors directly; they read CSS variables that point to the palette.
- `explorer.html`, `lab.html`, and `system.html` are nearly content-free shells; they `fetch('painting.json')` and render. `index.html` (the forensic) carries painting-specific prose inline by design. The reusable structure is the shell + sectioning + sticky TOC, not the words. See section 6.B.

**Why this shape:** the user wanted real template-ification: swap config + assets + tile pyramid, get a new design system. Putting painting metadata in JSON keeps the diff small for a new painting. Putting palette in CSS variables keeps the styling logic out of the HTML.

**Theme switching:** two themes (`dark` = Cenacolo / refectory at twilight, `light` = Refectory / midday). Toggle in nav top-right. State persists via `localStorage` key `cenacolo-theme`. All four pages share the toggle.

---

## 2.5. Three-repo split: ArtScope / lastsupper / working dir

The Cenacolo project as actually deployed lives in **three directories** on the local filesystem:

```
PROJECTZ/
|
|-- Lets_Eat_lastsupperanalysis/   <- working copy. Where iteration happens.
|                                     Has the source JPEGs, the tile pyramid,
|                                     painting.json filled in, the analysis
|                                     prose written. Gitignored locally.
|
|-- lastsupper/                    <- the published deployment. A clean copy
|                                     of the working dir, deployable to grivt.
|                                     Carries images. Treat as build output.
|
|-- ArtScope/                      <- the engine, github-publishable. No
                                      painting-specific images, no Last-Supper
                                      prose, just the shells + tokens + scripts
                                      + schema docs. Drop in another painting's
                                      assets + edit painting.json and it works.
```

### What goes in ArtScope (the github-publishable engine)

```
ArtScope/
|-- README.md                     <- the project pitch + quickstart
|-- HANDOVER.md                   <- this document, lightly edited
|-- LICENSE                       <- pick one (MIT recommended)
|-- .gitignore                    <- ignore /assets/*.jpg, /assets/*_files/, etc.
|
|-- index.template.html           <- forensic shell with placeholder prose
|-- explorer.html                 <- engine, untouched
|-- lab.html                      <- engine, untouched
|-- system.html                   <- engine, untouched
|-- tokens.css                    <- engine; the per-painting palette goes
|                                    here at the top of the :root block but is
|                                    flagged as a "swap me" section
|-- painting.template.json        <- schema with placeholder values + comments
|
|-- scripts/
|   |-- make_tiles.py
|   |-- build_assets.py
|   |-- run_analysis.py
|   `-- ... future scripts
|
`-- docs/
    |-- SCHEMA.md                 <- painting.json full reference
    |-- ADDING_A_PAINTING.md      <- the 12-step recipe from section 5
    `-- ARCHITECTURE.md           <- the 4-page IA + data flow
```

### What goes in lastsupper (the deployment)

A full copy of the working dir, minus the working-dir cruft (drafts, archives, etc.). Has the asset bundle (`last_supper_full.jpg`, the DZI tiles, all the hotspot and pigment thumbnails) plus the painting-specific `painting.json` and the populated `index.html` with the forensic prose. This is what gets uploaded to grivt.

### What stays in Lets_Eat_lastsupperanalysis (the working copy)

Everything plus the `archives/`, intermediate analysis figures, scratch scripts, anything Claude Code or I produce that isn't ready for deploy. Treat this as `src/`, treat `lastsupper/` as `dist/`.

### Extraction recipe (working dir &rarr; ArtScope)

When you sit down to make ArtScope the github repo, the move is:

```bash
# from the project root, with Lets_Eat_lastsupperanalysis containing the work-in-progress
mkdir -p ArtScope/scripts ArtScope/docs

# copy the engine pieces verbatim
cp Lets_Eat_lastsupperanalysis/explorer.html ArtScope/
cp Lets_Eat_lastsupperanalysis/lab.html      ArtScope/
cp Lets_Eat_lastsupperanalysis/system.html   ArtScope/
cp Lets_Eat_lastsupperanalysis/tokens.css    ArtScope/
cp Lets_Eat_lastsupperanalysis/make_tiles.py    ArtScope/scripts/
cp Lets_Eat_lastsupperanalysis/build_assets.py  ArtScope/scripts/
cp Lets_Eat_lastsupperanalysis/run_analysis.py  ArtScope/scripts/

# the forensic page becomes a template -- the prose stripped, structure kept
cp Lets_Eat_lastsupperanalysis/index.html ArtScope/index.template.html
# then by hand: replace the section bodies with <!-- TEMPLATE: ... --> comments,
# leave the nav + section shells + claims-table structure + cited-sources list

# the config becomes a schema-comment template
cp Lets_Eat_lastsupperanalysis/painting.json ArtScope/painting.template.json
# then by hand: replace every value with a placeholder, add // hint comments on
# what each field expects. Tools like jsonc-eslint handle these fine.

# scaffold docs
echo "# ArtScope" > ArtScope/README.md     # write actual pitch + quickstart
cp Lets_Eat_lastsupperanalysis/HANDOVER.md ArtScope/HANDOVER.md
# split out SCHEMA.md, ARCHITECTURE.md, ADDING_A_PAINTING.md
```

The lastsupper deployment is just:

```bash
mkdir -p lastsupper
cp -r Lets_Eat_lastsupperanalysis/* lastsupper/
rm -rf lastsupper/archives lastsupper/HANDOVER.md
# optionally rm lastsupper/run_analysis.py and other scripts; they don't need
# to ship on the web. Or keep them so someone reading the source can see the
# pipeline.
```

---

## 3. The `painting.json` schema

Full reference. Every field is consumed by at least one page. Required unless marked optional.

```jsonc
{
  "id": "last-supper",                         // string, slug; used for tile pyramid base name
  "site_name": "Cenacolo",                     // string, brand shown in nav + footer
  "site_tagline": "design system",             // string, brand subtitle

  "painting": {
    "title": "The Last Supper",                // string, English/conventional name
    "title_native": "L'Ultima Cena",           // string, original-language name
    "artist": "Leonardo da Vinci",
    "date_start": "1495",                       // string (not int, allows ranges)
    "date_end": "1498",
    "location": "Santa Maria delle Grazie, Milan",
    "dimensions_cm": [880, 460],               // [width, height] in cm
    "technique": "tempera + oil on dry plaster (gesso, pitch, mastic ground)",
    "current_state": "..."                     // string, freeform condition note
  },

  "scan": {
    "url": "assets/last_supper.dzi",           // primary tile source (DZI manifest)
    "fallback_url": "assets/last_supper_full.jpg",  // single-image fallback if DZI fails
    "width": 5381,                             // px, ACTUAL dimensions of the scan
    "height": 2926,
    "source": "Wikimedia Commons, ...",        // attribution string
    "reference_scan_url": "https://...",       // optional, gigapixel reference
    "free_url": "https://commons.wikimedia.org/...",  // optional, public-domain source
    "free_url_dimensions": [9600, 4800]
  },

  "motif_image": "assets/motif.jpg",           // hero foreground image
  "motif_caption": "Christ // 49.6%, 51.3%",   // caption shown over motif
  "hero_bg_image": "assets/hero_bg.jpg",       // hero background strip

  "design_principles": {
    "use": false,                              // bool, if true a "principles" section renders
    "note": "..."                              // optional, ignored if use=false
  },

  "measurements": {                            // populates the data table on index page
    "centroid_x_pct": 49.60,
    "centroid_y_pct": 51.30,
    "horizon_y_pct": 51.30,
    "vanishing_point_offset_px": 21.5,
    "golden_ratio_left_pct": 38.20,
    "golden_ratio_right_pct": 61.80,
    "global_luminance_mean": 100.8,
    "global_luminance_std": 43.1,
    "pixel_count": 15744806,
    "estimated_original_pigment_survival_pct": 20,
    "restoration_years": 21,
    "restoration_range": "1978-1999",
    "reference_scan_gigapixels": 16.1,
    "reference_scan_panoramas": 1042,
    "peer_reviewed_decoded_papers": 0
  },

  "extraction_method": {
    "name": "HSV-filtered region sampling",
    "steps": [                                 // array of strings, rendered as pipeline
      "Convert RGB to HSV in normalized [0,1] space",
      "Define rectangular regions over known pigment patches",
      "..."
    ],
    "code_repo_path": "scripts/extract_palette.py"   // string, informational only
  },

  "pigments": [                                // EXACTLY 12, ordered light to dark by convention
    {
      "name": "Refectory Cream",               // display name
      "hex": "#E8DCC8",                        // 6-digit hex
      "comment": "warm plaster highlight",     // short technical note (1 line)
      "sample_region_pct": [0.44, 0.30, 0.49, 0.36],  // [x0, y0, x1, y1] as 0..1 fractions
      "hue_range": null,                       // [low, high] in 0..1 or null = unfiltered
      "historical": "lead white (basic lead carbonate) + warm aging",  // historical pigment ID
      "sample_thumb": "assets/pigments/refectory_cream.jpg"   // auto-generated, optional
    }
  ],

  "themes": {
    "dark":  { "name": "Cenacolo (dark)", "subtitle": "refectory at twilight", "is_default": true },
    "light": { "name": "Refectory (light)", "subtitle": "refectory at midday", "is_default": false }
  },

  "hotspots": [                                // EXACTLY 10 by convention (the 13 figures + table + ceiling + scar)
    {
      "id": "jesus",                           // slug, becomes thumbnail filename + URL hash anchor
      "name": "Christ",                        // display name
      "region": [0.47, 0.41, 0.555, 0.62],     // [x0, y0, x1, y1] as 0..1 fractions of the SCAN
      "summary": "..."                         // 1-3 sentence prose, shown in sidebar
    }
  ]
}
```

**Region coordinate system:** `[x0, y0, x1, y1]` as fractions of scan width/height. `x0=0.47` means 47% across from left edge. The explorer converts these to OpenSeadragon viewport rects accounting for the actual aspect ratio.

---

## 4. The DZI tile pyramid pipeline

The explorer can load two kinds of image sources:
1. **DZI tile pyramid** (preferred). Hundreds of small tiles, only the visible ones load. First paint is instant. Memory stays flat.
2. **Single full image** (fallback). One big JPEG. Slow first paint, eats RAM, fine for testing.

The user has `last_supper_full.jpg` at 9600x4800 (~40 MB) in the project directory. **The tile pyramid currently shipped is from the 5381px version. Regenerate from the 9600 to unlock the deeper zoom.**

### Generate the pyramid

```bash
# from inside the project directory
python make_tiles.py assets/last_supper_full.jpg -o assets -n last_supper
```

Args:
- positional: path to source JPEG
- `-o assets`: write `.dzi` and `_files/` into the assets folder
- `-n last_supper`: name them `last_supper.dzi` and `last_supper_files/`

Output for the 9600 source: ~1400 tiles, ~30-90 seconds, total disk roughly the same as the source JPEG. The script is PIL-only. No libvips, no system deps beyond `pip install pillow`.

### Update painting.json after regeneration

If dimensions changed (i.e., you regenerated from a larger source than what's currently in `painting.json`), bump these fields:

```jsonc
"scan": {
  "url": "assets/last_supper.dzi",
  "fallback_url": "assets/last_supper_full.jpg",
  "width": 9600,                      // <-- update
  "height": 4800                      // <-- update
},
"measurements": {
  "pixel_count": 46080000             // <-- update (W * H)
}
```

The hotspot coordinates and pigment sample regions are in normalized 0..1 fractions, so they don't need to change when the scan resolution changes.

### When DZI fails (file:// CORS)

OpenSeadragon's `open-failed` handler in `explorer.html` falls back to single-image mode automatically. If you see DZI fail on `file://` it's because browsers won't fetch local tiles over file:// for security reasons. Solutions: serve over `python -m http.server`, or upload to grivt where this isn't a concern.

---

## 5. How to add a NEW painting (the template-ification recipe)

Goal: replicate the system for, say, *The Garden of Earthly Delights* or *Las Meninas* or *Composition VIII*.

### Step 1. Pick the painting + grab the highest-res public-domain scan

Wikimedia Commons usually has a multi-megapixel scan. Google Arts & Culture has gigapixel scans but they're behind a tile API. Start with Wikimedia.

### Step 2. Clone the project structure

```bash
cp -r Lets_Eat_lastsupperanalysis Lets_Eat_<newpainting>
cd Lets_Eat_<newpainting>
rm assets/last_supper.dzi
rm -rf assets/last_supper_files
rm assets/last_supper_full.jpg
rm assets/motif.jpg
rm assets/hero_bg.jpg
rm -rf assets/pigments assets/hotspots
```

### Step 3. Drop in the new scan as `assets/<id>_full.jpg`

Where `<id>` is your slug for this painting (e.g. `garden_delights`, `meninas`).

### Step 4. Generate the tile pyramid

```bash
python make_tiles.py assets/<id>_full.jpg -o assets -n <id>
```

### Step 5. Extract a palette

You need 12 named pigments sampled from the painting. The script in `scripts/extract_palette.py` (referenced in `painting.json.extraction_method.code_repo_path` but not yet checked in. **TODO: extract the inline Python from this handover and commit it**) is roughly:

```python
import json
import numpy as np
from PIL import Image
from colorsys import rgb_to_hsv

img = np.asarray(Image.open("assets/<id>_full.jpg").convert("RGB"), dtype=np.float32) / 255.0
H, W, _ = img.shape

def sample(region_pct, hue_range=None, sat_percentile=70):
    """Sample a pigment from a rectangular region with optional HSV filtering."""
    x0, y0, x1, y1 = region_pct
    crop = img[int(y0*H):int(y1*H), int(x0*W):int(x1*W)].reshape(-1, 3)
    if hue_range is not None:
        hsv = np.array([rgb_to_hsv(*p) for p in crop])
        h = hsv[:, 0]
        lo, hi = hue_range
        if lo < hi:
            mask = (h >= lo) & (h <= hi)
        else:  # wraps around 0
            mask = (h >= lo) | (h <= hi)
        crop = crop[mask]
        hsv = hsv[mask]
    else:
        hsv = np.array([rgb_to_hsv(*p) for p in crop])
    # take top N% by saturation
    s = hsv[:, 1]
    threshold = np.percentile(s, 100 - sat_percentile)
    crop = crop[s >= threshold]
    rgb = np.median(crop, axis=0)
    return "#{:02X}{:02X}{:02X}".format(*(int(c*255) for c in rgb))
```

Process: open the painting in a tool, eyeball rectangular regions over distinct color zones, write down the (x0, y0, x1, y1) as normalized fractions, optionally filter by hue band. Iterate until you have 12 good ones. Name them after either historical pigments or evocative aged-color labels (Refectory Cream, Cinabro Russet, etc.).

### Step 6. Generate per-pigment sample thumbnails

```python
import os
from PIL import Image
img = Image.open("assets/<id>_full.jpg").convert("RGB")
W, H = img.size
os.makedirs("assets/pigments", exist_ok=True)
for pig in pigments:  # pigments = list of dicts from your palette work above
    slug = pig["name"].lower().replace(" ", "_")
    x0, y0, x1, y1 = pig["sample_region_pct"]
    crop = img.crop((int(W*x0), int(H*y0), int(W*x1), int(H*y1)))
    cw, ch = crop.size
    s = 320 / max(cw, ch)
    crop = crop.resize((max(60, int(cw*s)), max(60, int(ch*s))), Image.LANCZOS)
    crop.save(f"assets/pigments/{slug}.jpg", quality=80, optimize=True)
```

### Step 7. Identify 10 hotspots

Hotspots are clickable annotated regions in the explorer. For *The Last Supper* they're the 13 figures grouped into 4 triads + table + scar + ceiling + window. For *The Garden of Earthly Delights* they might be the three panels and the iconic creatures. Pick 10 regions, give each:
- `id`: slug used as anchor + thumbnail filename
- `name`: display name
- `region`: [x0, y0, x1, y1] as 0..1 fractions
- `summary`: 1-3 sentences

Then generate thumbnails:

```python
os.makedirs("assets/hotspots", exist_ok=True)
for hs in hotspots:
    x0, y0, x1, y1 = hs["region"]
    crop = img.crop((int(W*x0), int(H*y0), int(W*x1), int(H*y1)))
    cw, ch = crop.size
    s = 160 / max(cw, ch)
    crop = crop.resize((max(60, int(cw*s)), max(60, int(ch*s))), Image.LANCZOS)
    crop.save(f"assets/hotspots/{hs['id']}.jpg", quality=80, optimize=True)
```

### Step 8. Build the asset images (motif, hero_bg, texture_tile, pattern_band)

These are derived crops/transformations of the source:
- `motif.jpg`: a portrait-orientation crop of the most iconic figure or detail (used as hero foreground)
- `hero_bg.jpg`: a wide landscape strip used as faded hero background
- `texture_tile.jpg`: a 200x200 crop of an interesting surface texture (craquelure, brush strokes, weave), tileable when needed
- `pattern_band.jpg`: a thin horizontal slice of a frieze or decorative element, tileable horizontally

For the Last Supper these were hand-picked. For the new painting you can pick by eye or write a script that scores regions by entropy / saturation variance.

### Step 9. Write the new `painting.json`

Take `Lets_Eat_lastsupperanalysis/painting.json` as a template, replace all painting-specific values. The schema reference in section 3 lists every field.

### Step 10. Update the palette CSS in `tokens.css` (KEY STEP)

`tokens.css` currently hardcodes the Last Supper palette as CSS variables in both light and dark themes. You need to retune these for the new painting's palette. See section 6 below. This is the part of the system that hasn't been fully template-ified yet, so there's manual work involved.

### Step 11. Rewrite `index.html` (the forensic)

The forensic essay is painting-specific by nature. Treat it as a content rewrite. The HTML shell (nav, layout, sticky TOC, section structure, the claims table, the note-panel cards, the cited-sources list) is fully reusable. Only the prose, the claims, and the citations change per painting.

### Step 12. Test

```bash
python -m http.server 8000
# open http://localhost:8000 in a browser
```

Check:
- Homepage shows the new painting metadata, palette, measurements
- Explorer loads the DZI, hotspots fly correctly, region inspector returns sensible nearest-pigment matches
- Theme toggle works on all three pages
- All thumbnails load (404s in console = missing pigment/hotspot files)

---

## 6. Known gaps and follow-up work

These are intentional left-aheads. Pick them off as time allows.

### A. Lift the palette out of `tokens.css` into JS-generated CSS variables

**Current state:** `tokens.css` has the 12-color Cenacolo palette hardcoded as variables in `:root` (dark) and `[data-theme="light"]` blocks. To use a new painting's palette you have to manually edit `tokens.css`.

**Better:** generate a `:root { --pigment-1: ...; ... }` style block from `painting.json` at runtime. Sketch:

```js
// in a shared loader, executed on every page
function applyPalette(config) {
  const root = document.documentElement.style;
  config.pigments.forEach((p, i) => {
    root.setProperty(`--pigment-${i+1}`, p.hex);
    root.setProperty(`--pigment-${p.name.toLowerCase().replace(/ /g, '-')}`, p.hex);
  });
  // Also re-derive semantic tokens (primary, accent, etc.) from a config-driven mapping:
  //   config.themes.dark.semantic_map = { primary: 4, secondary: 7, accent: 6, ... }
}
```

Then `tokens.css` would reference `var(--pigment-jesus-blue)` or `var(--primary)` and the actual color comes from the config.

**Why not done yet:** the Last Supper deployment doesn't need it (palette is stable), and it's a sharp refactor that touches every component style. Worth doing the first time someone wants a second painting.

### B. Refactor `index.html` (the forensic) for shared shell + config-driven facts

**Current state:** `index.html` has the entire forensic essay inline plus a hardcoded "claims verdict table."

**Better:** split into:
- A shared shell (nav, sticky TOC, layout grid) reusable across paintings
- A `analysis.content.md` (or `.html`) file per painting holding only the prose
- A `claims_table` array in `painting.json` (or sibling) that drives the verdict table

Then for a new painting, you write the markdown content + the claims_table data, and the shell renders it.

### C. Mobile pass on the explorer

The explorer is desktop-oriented. The sidebar collapses to a bottom panel on narrow viewports but the filter drawer + minimap overlap controls awkwardly. Either redesign the bottom toolbar to wrap better on mobile, or hide the secondary controls (filters/inspect/permalink) behind a single "advanced" menu.

### D. Permalink robustness

`copyPermalink()` works on https but the `navigator.clipboard` API fails on `file://`. Currently falls back to an alert with the URL. On grivt this isn't an issue. Could add a visible toast component for the success state.

### E. Inspect mode CORS

The region inspector reads pixel RGB via `ctx.getImageData`. This fails when the image is loaded from a different origin without proper CORS headers. On the same origin (which is what you'll have on grivt) it works fine. On `file://` it sometimes fails depending on browser. Currently shows a graceful "could not read pixel" message. Wikimedia hotlinking would also fail this.

### F. Search / "go to" widget

For paintings with many hotspots, a text search box that types-ahead-to-fly would help. Not needed for 10 spots, but for a 30-spot painting it would.

### G. Comparison view

Split-screen with a reference image (e.g., Giampietrino's 1520 copy of the Last Supper, which preserves details lost in Leonardo's version). Could be a tab inside the explorer or a separate `compare.html` page.

### H. IR/UV simulation (clearly labeled as simulation)

For didactic purposes: buttons that apply channel-inversion or hue-shift to simulate infrared or ultraviolet response. NOT actual hyperspectral data; clearly labeled as visual approximation. Pedagogical, ties into the "AI revealed hidden text" debunking narrative.

### I. Measurement tool

Click two points, get the pixel-space and percentage-space distance. Useful for letting users verify the compositional claims themselves.

### J. Commit the palette/asset extraction scripts

The inline scripts in section 5 (palette extraction, thumbnail generation) should live in `scripts/` as real Python files with `__main__` handlers and CLI args, alongside `make_tiles.py`. Currently they only exist as snippets here.

---

## 7. Local development

```bash
cd C:\Users\joshr\PROJECTZ\Lets_Eat_lastsupperanalysis
python -m http.server 8000
# then open http://localhost:8000
```

Why not file://: the HTML pages `fetch('painting.json')`, which file:// blocks under same-origin policy in most browsers. Also CORS for tile loading. Always serve over HTTP locally.

If `python` is Python 2 on the system: `python3 -m http.server 8000`.

Python deps:
```bash
pip install pillow numpy
```

That covers all three scripts: `make_tiles.py` (PIL only), `build_assets.py` (PIL only), `run_analysis.py` (PIL + numpy).

No node, no bundler, no transpile.

### Regenerating assets after editing painting.json

```bash
# rebuild pigment + hotspot thumbnails from the config
python build_assets.py

# OR with palette re-sampling from the source image
python build_assets.py --refresh-palette
```

### Regenerating the forensic figures from a higher-res source

```bash
# default: reads assets/last_supper_full.jpg, writes assets/analysis_*.jpg
python run_analysis.py

# explicit
python run_analysis.py --source assets/last_supper_full.jpg --out assets
```

The script also prints a YAML-style block of measured values to stdout. Paste those into the `<pre class="data">` block in `index.html` section 04.01 to keep documented numbers in sync with the source.

---

## 8. Deployment

Static site. Drop the whole directory into wherever grivt serves files from. Verify:

```
https://<grivt>/Lets_Eat_lastsupperanalysis/
https://<grivt>/Lets_Eat_lastsupperanalysis/explorer.html
https://<grivt>/Lets_Eat_lastsupperanalysis/system.html
```

Things to check after deploy:
- `painting.json` returns 200 (otherwise the pages will fall back to mostly-empty rendering)
- The `last_supper.dzi` file returns 200 and the `_files/` directory is listable (or at least the tiles inside are accessible)
- Hotspot and pigment thumbnails return 200 (404s show as broken thumbs in sidebar)
- Fonts load from Google Fonts CDN (Cormorant Garamond, Inter, JetBrains Mono, Shippori Mincho)
- OpenSeadragon loads from jsDelivr

If you want zero CDN dependency: download OpenSeadragon and the fonts, ship locally. Roughly +500 KB.

---

## 9. Quick reference: where to look when X breaks

| Symptom | Look at |
|---|---|
| Homepage shows empty palette / measurements | `painting.json` 404 or invalid JSON. Check browser console + network tab. |
| Explorer shows single big image, no tiles | DZI failed to load. Check `assets/last_supper.dzi` exists. Open it: should be XML. Check `assets/last_supper_files/` is accessible. |
| Explorer black screen | OpenSeadragon CDN unreachable, or `fallback_url` also missing. |
| Hotspot thumbnails broken | `assets/hotspots/<id>.jpg` missing. Filename must match `hotspot.id` in config. |
| Pigment thumbnails broken | `assets/pigments/<snake_case_name>.jpg` missing. Or set `sample_thumb` on the pigment to override path. |
| Region inspector "could not read pixel" | CORS; you're on `file://`. Serve over HTTP. |
| Theme doesn't persist across pages | `localStorage` blocked (incognito sometimes). The theme will still work, it just resets per page. |
| Colors look wrong after theme switch | `tokens.css` palette mapping. Light/dark sections under `:root` and `[data-theme="light"]`. |

---

## 10. Open questions for the human

- **Domain name:** which subdomain on grivt? `lasteupper.grivt...`? `cenacolo.grivt...`? `lets-eat.grivt...`? Affects nothing in code, just the link the user shares.
- **Next painting?** Once the template flow is exercised once, the second one is much faster. Candidate suggestions in section 6.
- **Analytics?** Site has none. Add Plausible / Umami if wanted. Not needed for v1.

---

*Last updated: 2026-05-14. Built across two Claude sessions; this handover is the source of truth.*
