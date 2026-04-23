# index.html Analysis — Fluted Glass FBM Wallpaper

## 1. File Location and Size
- **Path:** `./index.html` (`/Users/karomkar/Pictures/plash/fluted-glass/index.html`)
- **Size:** 20K (single file, self-contained)

## 2. Overall Structure (order of elements)
```
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Fluted Glass FBM</title>
    <style> ... </style>           ← 1 style block
  </head>
  <body>
    <canvas id="canvas"></canvas>  ← 1 DOM element
    <script> ... </script>         ← 1 script block (~300 lines)
  </body>
</html>
```
- Extremely minimal DOM: just `html > head + body > canvas + script`
- No external resources (no CSS files, no JS files, no images, no fonts)

## 3. CSS Summary
Single `<style>` block with 3 rules:
| Rule | Purpose |
|------|---------|
| `html, body` | Zero margin, full height, black background, `overflow: hidden` |
| `canvas` | `display: block`, `100vw × 100vh` — fills viewport |

No animations, transitions, or transforms in CSS. No media queries. No custom properties.

## 4. JavaScript Summary
Single `<script>` block, well-organized into 9 numbered sections:

| Section | Purpose |
|---------|---------|
| 1. WebGL Context | Gets `webgl2` context with `antialias: false`, `premultipliedAlpha: false` |
| 2. Shaders | Vertex shader (pass-through) + Fragment shader (all visual work) |
| 3. Shader Compilation | `compileShader()` helper |
| 4. Program Linking | Creates/links/uses GL program |
| 5. Geometry | Full-screen quad via `TRIANGLE_STRIP` (4 vertices, no index buffer) |
| 6. Uniform Locations | Caches `uResolution`, `uTime`, `uDevicePixelRatio`, `uSpatialOffset` |
| 7. Randomization | Random time seed + 2D spatial offset per reload |
| 8. Resize Handler | Caps DPR at 1.0, updates canvas size + viewport + uniforms |
| 9. Render Loop | `requestAnimationFrame` with 24fps cap |

### Fragment Shader Pipeline (per-pixel):
1. Pixel → world coordinates (aspect-corrected, zoomed, offset)
2. Diagonal drift over time
3. Fluted glass stripe local coords + lens refraction offset
4. Domain warp with 2 `valueNoise` calls (decorrelated X/Y)
5. Single `valueNoise` sample → `pow(2.0)` → `smoothstep(0.2, 0.8)`
6. Static grain layer (arithmetic hash, no sin())
7. Glass edge highlights (`fwidth()`-adaptive)
8. Shadow gradient per stripe
9. Vignette (per-edge `smoothstep` + `sqrt` softening)
10. Output brightness

## 5. Animations, Transitions, GPU-Intensive Operations
- **Primary animation:** Full-screen WebGL2 fragment shader running continuously
- **GPU work per frame:** Every pixel evaluates:
  - 4 `valueNoise` calls (each = 4 `hash` + 2 `mix` + smoothstep) = ~16 hash evaluations
  - `pow`, `smoothstep`, `fwidth`, `sqrt` per fragment
  - Grain computation (arithmetic hash)
- **No CSS animations or transitions**
- **No CSS transforms or will-change**

### Performance Mitigations Already in Place:
- DPR capped at 1.0 (saves ~55–75% fragments on retina)
- 24fps cap (saves ~60% GPU duty vs 60fps)
- `antialias: false` (no MSAA overhead)
- FBM reduced to 2 octaves for warp, single noise for main (comment says 4 octaves but loop is only 2)
- All tunables are `const` (compiler can fold)
- Resolution/DPR uniforms only uploaded on resize, not per-frame

## 6. requestAnimationFrame / setInterval / setTimeout Usage
- **`requestAnimationFrame`**: Used in render loop (section 9)
  - Fires at display refresh rate (60–120Hz)
  - Frame-skipping logic: only renders when `currentTime - lastFrameTime >= frameDurationMs` (41.67ms = 24fps)
  - Recursive: `renderFrame` calls `requestAnimationFrame(renderFrame)` at end
  - **Never cancelled** — runs indefinitely (appropriate for wallpaper; Plash pauses WebView when occluded)
- **No `setInterval`**
- **No `setTimeout`**

## 7. Frequently-Firing Event Listeners
| Event | Handler | Frequency | Notes |
|-------|---------|-----------|-------|
| `resize` | `handleResize()` | Low (only on window resize) | Updates canvas dimensions, viewport, uniforms |

- **No `scroll` listener** (overflow: hidden, no scrollable content)
- **No `mousemove` listener**
- **No `pointermove` listener**
- **No `wheel` listener**
- **No keyboard listeners**
- **No touch listeners**

This is clean — only the necessary `resize` listener.

## 8. Image/Media Usage
- **No images** (`<img>`, CSS `background-image`, etc.)
- **No video or audio elements**
- **No textures loaded into WebGL** (all visuals are procedural)
- **No fonts loaded** (no text rendered)
- **No fetch/XHR requests**

Fully self-contained, zero network dependencies.

## 9. DOM Complexity Assessment
- **Total DOM elements:** 5 (`html`, `head`, `meta`, `title`, `style`) + 3 (`body`, `canvas`, `script`) = **8 elements**
- **Complexity: Minimal** — this is about as simple as an HTML document can be
- No shadow DOM, no iframes, no SVG, no MutationObservers
- Single canvas element — all rendering is WebGL, not DOM-based

## 10. Code Organization Issues

### Positive Observations:
- Excellent section numbering (1–9) with clear headers
- Thorough decision comments explaining every non-obvious choice
- Performance budget documented upfront
- Tunable constants clearly grouped and documented
- Logical ordering: context → shaders → compile → link → geometry → uniforms → randomize → resize → render

### Issues Found:

#### 10a. FBM Function Mismatch (Bug or Dead Code)
- `fbmMain()` (lines ~170–180) is defined with a comment saying "4 octaves" but the loop only runs **2 octaves** — identical to `fbmWarp()`
- **`fbmMain()` is never called anywhere** — the main noise uses a single `valueNoise()` call instead
- `fbmWarp()` is also **never called** — warp uses direct `valueNoise()` calls
- Both FBM functions are dead code. The comments reference a previous design that was simplified.

#### 10b. Minor: Grain Section Placement
- The grain computation is inside the fragment shader's `main()` but could benefit from being extracted into a helper function for consistency with the noise functions above it.

#### 10c. No Error Handling for Program Linking
- `gl.linkProgram(program)` doesn't check `gl.getProgramParameter(program, gl.LINK_STATUS)` — shader compilation errors are caught but link errors are not.

#### 10d. No Cleanup / Disposal
- No cleanup of WebGL resources on page unload. Acceptable for a wallpaper (Plash manages lifecycle), but not best practice for reusable code.

## CPU/GPU/Battery Impact Summary

| Factor | Assessment |
|--------|-----------|
| GPU load per frame | Moderate — full-screen fragment shader with ~4 noise evals + math per pixel |
| Frame rate | **Capped at 24fps** — significant power savings vs 60fps |
| Resolution | **DPR capped at 1.0** — major savings on retina displays |
| CPU load | Minimal — only `requestAnimationFrame` callback + 1 uniform upload per frame |
| Memory | Minimal — 1 shader program, 1 buffer, 1 canvas |
| Network | Zero — fully self-contained |
| Battery impact | **Moderate-low** for an always-on wallpaper. The 24fps cap and DPR cap are the two biggest wins. Main concern is that the GPU never fully idles while the wallpaper is visible. |

### Potential Further Optimizations:
1. Lower `TARGET_FPS` to 15 or even 12 — drift is slow enough to tolerate it
2. Reduce `STRIPE_COUNT` — fewer stripes = fewer `fwidth()` discontinuities
3. Add `requestIdleCallback` or visibility-based pausing if not relying on Plash's occlusion detection
4. Remove dead `fbmWarp()` and `fbmMain()` functions — reduces shader compile time slightly
