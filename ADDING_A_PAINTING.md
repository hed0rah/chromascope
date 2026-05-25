# Adding a painting

The reproducible recipe for taking ArtScope from "engine seed" to "deployment for a new painting." Twelve steps. Plan for 2-4 hours the first time, an hour after that.

## Prerequisites

- Python 3.8+ with `pillow` and `numpy` installed (`pip install pillow numpy`)
- A high-resolution scan of the painting (5,000+ px wide ideal, 9,000+ px excellent)
- Permission to use the scan (public-domain Wikimedia source, IIIF manifest from a museum, your own photography, etc.)
- Time and attention. Most of the value is in the prose and the curation, not the scripts.

## Step 1. Pick the painting

The shells assume:
- One visible iconic subject (or composition) on a roughly rectangular field
- Color information matters (you wouldn't choose a near-monochrome work)
- 8-12 identifiable named regions you'd want to annotate as hotspots
- Either an "AI-revealed hidden meaning" claim attached to it, or a genuine art-historical claim worth deep-diving

Candidates that fit well: *The Garden of Earthly Delights* (Bosch), *Las Meninas* (Velázquez), *Guernica* (Picasso), *The Night Watch* (Rembrandt), *Composition VIII* (Kandinsky), *The Great Wave off Kanagawa* (Hokusai).

Candidates that fit poorly: pure abstract works without subject regions, sculptures, anything with so few colors that pigment extraction collapses.

## Step 2. Clone the engine

```bash
git clone https://github.com/<you>/ArtScope.git my-painting
cd my-painting
```

If your deployment will live in a sibling directory (recommended pattern: `ArtScope/` for the engine, `mypainting/` for the deployment), copy the engine files into the deployment directory instead.

## Step 3. Drop in the source scan

```bash
cp /path/to/painting_source.jpg assets/<slug>_full.jpg
```

Where `<slug>` is your painting id (e.g. `garden_delights`, `meninas`, `night_watch`).

## Step 4. Generate the tile pyramid

```bash
python scripts/make_tiles.py assets/<slug>_full.jpg -o assets -n <slug>
```

This writes `assets/<slug>.dzi` (an XML manifest) and `assets/<slug>_files/` (a folder of small JPEG tiles). For a 9000px source: ~30-90 seconds, ~1400 tiles total, disk roughly equal to the source JPEG.

## Step 5. Copy and fill `painting.json`

```bash
cp painting.template.json painting.json
```

Open `painting.json`. Walk top-to-bottom replacing every value. The schema reference at `docs/SCHEMA.md` lists every field with its type and meaning.

The non-obvious bits:

- **`id`** must match the `-n <slug>` argument you used in step 4.
- **`scan.width` and `scan.height`** must match the *actual* pixel dimensions of your source JPEG. Get them with `python -c "from PIL import Image; print(Image.open('assets/<slug>_full.jpg').size)"`.
- **`pigments`** is the big one. See step 7 below.
- **`hotspots`** is the second big one. See step 8.
- Delete or leave the `_comment_*` fields. The runtime ignores them.

## Step 6. Pick the motif and hero images

These are derived crops, not the full painting.

```bash
# Pick a portrait-orientation crop of the most iconic figure or detail.
# Save as assets/motif.jpg. Rough target: 800x1000 px.

# Pick a wide landscape strip across the painting -- the background
# atmosphere, the middle band, etc. Save as assets/hero_bg.jpg.
# Rough target: 1600x600 px.
```

You can use any image editor, or scripted crops via Pillow:

```python
from PIL import Image
img = Image.open("assets/<slug>_full.jpg")
W, H = img.size
# motif: 30%-50% wide, 25%-65% tall (adjust to find your subject)
img.crop((int(W*0.30), int(H*0.25), int(W*0.50), int(H*0.65))).save("assets/motif.jpg", quality=85)
# hero: full width, middle 40%
img.crop((0, int(H*0.30), W, int(H*0.70))).save("assets/hero_bg.jpg", quality=85)
```

## Step 7. Curate the palette

You need 12 named pigments sampled from the painting. The script in `scripts/build_assets.py` handles the *mechanical* part (sampling rectangles, computing the median color, generating thumbnails). The *curation* is on you.

### Process

1. **Open the painting in an image viewer.** Identify 12 distinct color zones. Aim for visual variety: a warm highlight, a cool shadow, a saturated red, a muddy green, a bright blue, a deep umber, etc. The system page renders them in a 4-column grid; 12 fits cleanly.

2. **For each zone, write down its bounding rectangle** as `(x0, y0, x1, y1)` in 0..1 fractions. A 1000-pixel-wide image has its left edge at x=0 and right edge at x=1, so the central column is roughly `x in [0.4, 0.6]`.

3. **Decide whether to hue-filter.** For mixed regions (e.g. a face that has both warm skin and cool shadow), filter by a hue band to isolate one tone. For uniform regions (e.g. a plain wall), leave `hue_range: null`.

4. **Name each pigment.** Two flavors:
   - *Evocative names tied to the painting:* "Refectory Cream," "Bread Crust," "Plaster Tarnish."
   - *Historical pigment names:* "Lead White," "Ultramarine," "Vermillion."
   Either works. The historical name goes in the `historical` field regardless.

5. **Write each pigment object** into `painting.json`'s `pigments` array.

### Auto-derive the hex values

Once `painting.json` has the 12 pigment definitions (regions, hue ranges, names) but the `hex` values are placeholders, run:

```bash
python scripts/build_assets.py --refresh-palette
```

This samples each region with its hue filter, computes a median, writes the resulting hex back into `painting.json`, and generates a thumbnail for each pigment at `assets/pigments/<slug>.jpg`.

After this step, `tokens.css` still has the *Last Supper* palette hardcoded. To use the new palette in the system page, hand-edit the color variables in `:root` and `[data-theme="light"]` blocks to map to your new pigments. (Future work: lift this out of CSS into a JS-injected variable block. See HANDOVER section 6.A.)

## Step 8. Curate the hotspots

8-12 named regions. Each gets a hotspot button in the lab, a clickable overlay in the explorer, and a card with prose summary in the explorer sidebar.

```jsonc
{
  "id":      "lowercase-hyphenated-slug",
  "name":    "Display name",
  "region":  [x0, y0, x1, y1],   // 0..1 fractions
  "summary": "1-3 sentences shown when the hotspot is active."
}
```

Then generate the hotspot thumbnails:

```bash
python scripts/build_assets.py
```

This regenerates `assets/hotspots/<id>.jpg` for each hotspot in the config. Without `--refresh-palette`, it leaves the pigment hex values alone.

## Step 9. Run the forensic analysis

```bash
python scripts/run_analysis.py
```

Output:

- `assets/analysis_geometry.jpg`: compositional grid overlay (geometric center, golden-ratio columns, detected subject centroid)
- `assets/analysis_stretch.jpg`: side-by-side original vs aggressive dark-region contrast stretch
- `assets/analysis_mirror.jpg`: original vs horizontally flipped
- A YAML block printed to stdout with measured values

Copy the YAML block into the analysis page's `<pre class="data">` block in section 04.01 once you have `index.html` populated.

The geometry algorithm finds the centroid of skin-tone pixels in the central band. If your painting doesn't have a clearly central figure (e.g. *Composition VIII*), this measurement will be meaningless. Either refine the algorithm for your case or use a different anchor measurement.

## Step 10. Write the forensic essay

```bash
cp index.template.html index.html
```

Walk through `index.html` top-to-bottom. Every `[TEMPLATE: ...]` marker is content to replace.

### Sections to populate

1. **Title and lead** (above the abstract). Two-line h1 with painting name and "// a forensic" subtitle. 2-3 sentence lead orienting a first-time reader.

2. **Abstract** (section 01). Three paragraphs:
   - What viral/persistent claim or question prompted this document
   - Your verdict and primary evidence
   - What the document does (mirror the section structure)

3. **Method note panel** (after the abstract). Honest disclosure of who and what wrote the document. If AI was a collaborator, say so directly.

4. **Claims table** (section 02). One row per specific claim. Use the verdict classes:
   - `v-true` (success badge): Verified
   - `v-mixed` (warning badge): Distorted
   - `v-false` (danger badge): Fabricated
   - `v-fringe` (info badge): Fringe (pre-AI)

5. **Real science** (section 03). One `note-panel.verify` per real study. Each panel: study identifier, panel-head verdict label, 2-4 sentences on the work.

6. **Our own analysis** (section 04). The figures generated in step 9, with figcaptions explaining each one. The YAML measurements block from `run_analysis.py` output.

7. **High-quality scans** (section 05). One `note-panel` per canonical scan source, with the URL.

8. **Tools** (section 06). 8-15 cards across 4 categories. Most can carry over from the Last Supper version verbatim; some are domain-specific.

9. **Why the story spreads** (section 07). The structural critique. If your painting has no associated misinformation, this section can become something else: "Why this painting attracts misreadings" or "The interpretive history" or just delete the section.

10. **Sources** (section 08). The citation list. Use `ctype` classes:
    - `peer`: peer-reviewed journal
    - `museum`: institutional / industry source
    - `farm`: explicitly cited as unreliable for the source-pattern point

### Editorial principles

- One claim per row. One paragraph per idea.
- Quote no source for more than 15 words. Paraphrase everything else.
- Em-dashes are out. Use periods, commas, parens, colons.
- Don't bury the verdict.
- Cite at every claim of fact.

## Step 11. Test locally

```bash
python -m http.server 8000
```

Then open `http://localhost:8000` in a browser.

Why not `file://`: the pages `fetch('painting.json')`, which file:// blocks under same-origin policy. Also CORS on tile loading and the lab's `getImageData` calls.

Check:

- Homepage renders the forensic with all template markers replaced
- Explorer loads the DZI, hotspots fly correctly
- Lab samples the painting, histograms render, k-means returns sensible top colors
- System page renders the new palette and pigment thumbnails
- Theme toggle works on all four pages
- All thumbnails load (no 404s in console)

## Step 12. Deploy

Static site. Upload the populated deployment directory to your host. On grivt, AWS S3, GitHub Pages, Netlify, anywhere.

Worth checking on the deployed version:

- `painting.json` returns 200
- The `.dzi` file is accessible and its `_files/` directory is browseable (or at least the tiles inside are)
- Fonts load from Google Fonts CDN
- OpenSeadragon loads from jsDelivr

If you want zero CDN dependency: download OpenSeadragon and the fonts, ship locally. Adds about 500 KB.

---

That's it. The first painting takes a few hours of curation + writing. The second one is faster because you've internalized the pattern. The recipe is the same.
