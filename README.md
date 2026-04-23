# kinetic-canvas

Animated full-screen WebGL2 canvases — generic motion art that also happens to
work well as live desktop wallpapers via [Plash](https://github.com/sindresorhus/Plash)
on macOS.

Each canvas is a single self-contained `index.html`: one fragment shader,
domain-warped FBM noise, no dependencies, no build step. Open it in any modern
browser and it runs.

## Canvases

### `dithered/`
**Dithered FBM** — binary black/white pixels via an 8×8 Bayer ordered-dithering
matrix, driven by domain-warped FBM noise. Slow organic motion with a crisp
retro/print aesthetic.

### `dotted/`
**Dotted FBM** — a halftone grid of white dots on black. Dot radii are modulated
by the same warped noise field, producing a breathing halftone look.

### `fluted/`
**Fluted Glass FBM** — animated noise refracted through vertical fluted glass
ridges. Each ridge acts as a cylindrical lens with edge highlights and shadow
gradients for a 3D glass appearance.

## Shared design

All three share the same engine:

- WebGL2 fragment shader, full-screen triangle-strip quad
- Domain-warped value-noise FBM (2-octave warp × 2 axes + main FBM)
- Diagonal drift for directional flow on top of the organic warp
- Per-reload randomized time and spatial offset — every launch looks different
- DPR capped at 1.0, frame rate capped (15–24 fps) — cheap enough to run always-on
- Per-edge vignette fade
- All tunables are GLSL `const` — edit the top of the fragment shader and reload

## Running locally

Just open the file:

```sh
open dithered/index.html
```

Or serve the repo over HTTP (any static server works):

```sh
python3 -m http.server 8000
# then visit http://localhost:8000/dithered/
```

Requires a browser with WebGL2 (Safari 15+, Chrome, Firefox, modern Edge).

## Using as a Plash live wallpaper

[Plash](https://github.com/sindresorhus/Plash) is a free macOS app that turns
any web page into your desktop wallpaper.

1. Install Plash from the [Mac App Store](https://apps.apple.com/app/plash/id1494023538).
2. Click the Plash menu-bar icon → **Open URL…**
3. Point it at one of the canvas files. Two options:
   - **Local file (simplest):** paste the absolute `file://` URL, e.g.
     `file:///Users/you/path/to/kinetic-canvas/dithered/index.html`
   - **Served over HTTP:** run a local static server (see above) and use
     `http://localhost:8000/dithered/`. Useful if you want to edit and hit
     reload without re-pasting URLs.
4. Recommended Plash settings:
   - **Reload interval:** off (these canvases don't need reloading — they
     animate continuously and re-randomize on every load)
   - **Browsing mode:** off (no interaction needed)
   - **Invert colors:** your call — the dithered and dotted canvases are white
     on black; inverting gives black on white.
   - **Opacity / Dimming:** taste. Plash dims automatically on top of bright
     windows, which works fine with these.

Plash pauses the WebView when the desktop is fully occluded by other windows,
so these canvases don't burn GPU when you can't see them.

### Tweaking a canvas for your setup

Every tunable lives as a `const` at the top of the fragment shader in each
`index.html` — look for the `Tunable constants` block. Common knobs:

- `SPEED` — animation speed (0 = frozen)
- `ZOOM` — noise field scale (lower = larger blobs)
- `BRIGHTNESS` / `DOT_COLOR` / `GLASS_COLOR` — overall brightness
- `FADE_TOP` / `FADE_BOTTOM` / `FADE_LEFT` / `FADE_RIGHT` — per-edge vignette

Edit, save, tell Plash to reload the page (or just re-open the URL).
