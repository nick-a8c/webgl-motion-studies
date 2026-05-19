# MotionKit

A single-file browser studio for designing interactive motion graphics. Built
on Three.js, no build step, no `npm install`. Everything lives in one
`index.html`.

## Live demo

👉 **[nick-a8c.github.io/motionkit](https://nick-a8c.github.io/motionkit/)**

## What's inside

Three pages share the same canvas and renderer:

- **Landing** — animated hero (211 small star outlines arranged along a big
  star, with a cursor-thickening hover halo).
- **Playground** — gallery of seven tunable animations, each with its own
  controls panel, the shared overlay pass (Gradient · Pixelate · Halftone),
  and full snapshot/HTML/PNG/SVG export.
- **Lab** — blank-canvas builder with layered subjects (text + SVG),
  triangulated vector silhouettes you can deform per-vertex, EFFECTS layers
  with a cascading motion + pixel pipeline, and a per-tool save-to-Playground
  flow.

### Playground tools

| Tool | What it is |
|---|---|
| **Composer** | Recipe-driven: pick a Source (Circle, Lissajous, Grid, or uploaded SVG), one or more Primitives (round-robin), and a chain of motion Effects. Per-instance line-thickness gradient included. |
| **Glyph** | Animated shapes traveling along Hershey-font letterform paths. Includes Composer-style Primitive + Effects pickers as a sub-recipe in the middle column. |
| **Vinyl** | Text mask sliced into concentric ring grooves with per-segment Perlin rotation. |
| **Waves** | Stacked sine rows displaced by an optional text/SVG silhouette force. Two-button Fill/Outline SVG mode. |
| **Dots** | Bitmap-style dot grid that morphs between text frames. Three-color palette (background / foreground / off cells). |
| **Particles** | Field-driven particle system with halftone-style attraction and repulsion. |
| **Halftone** | Bayer-dithered halftone over text / SVG / solid / shift-click point cloud. Includes a wave animation + cursor halo. |

### Lab features

- Layered subjects (Text / SVG) with **Outline / Solid Fill / Particles /
  Halftone** styles. Reorder via drag-and-drop, ↑/↓ buttons, or × / eye
  icons. The top row of the Layers panel composites in front on the canvas,
  and new layers spawn at the top of the stack.
- **Solid Fill** and **Outline** render as **real triangulated polygon
  meshes** (marching-squares contour extraction for text, direct path
  triangulation for SVG, hole containment by signed-area, midpoint
  subdivision for deformation density). Effects deform vertices per-frame;
  SVG export walks the deformed boundary as a single `<path
  fill-rule="evenodd">`.
- **EFFECTS layers** cascade motion (Position noise, Flow field, Wave flow,
  Skew, Scale pulse, Repulsion, Orbit, Glitch, Pulse, Cursor attractor) onto
  every layer beneath them in the stack. Pixel effects (Tint, Grain,
  Chromatic aberration, Pixelate, Ripple, Hue rotate, Invert, Halftone,
  Bloom, Kaleidoscope, Edge detect, Rounded corners) run as a post-process
  chain.
- **Per-style defaults** swap in sensible starting values on style change.
- **Font dropdown** with 16 entries (System mono + 15 Google Fonts).
- **Save to Playground** persists a Lab tool to `localStorage` and surfaces
  it as a removable card in the Playground grid.

## Exports

| Format | What it carries |
|---|---|
| **JSON** | Lossless snapshot. v2 schema with layers, effects, transparent flag, inlined SVG assets. |
| **PNG** | Oversampled to 2048² (×1), 4096² (×2), or 6144² (×3). Honors the Transparent toggle. |
| **SVG** | Real vector elements — polylines for outlines, circles for particle / halftone instances, and `<path fill-rule="evenodd">` paths for deformed Solid Fill / Outline meshes. |
| **HTML** | Self-contained `.html` with Three.js + Line2 from CDN, the animation's IIFE inlined, baked params, and optional PAN / ZOOM interaction (per-export checkboxes). |

## Run

```bash
# Just open in a browser:
open index.html

# Or serve it (HTTP behaves better than file:// in some browsers):
python3 -m http.server 8000
# then visit http://localhost:8000
```

No build step, no `npm install`. Three.js and the Line2 modules are loaded
from CDN at runtime.

## Hosting

The repo is set up for **GitHub Pages** — `index.html` at the root, no
config files needed. To enable on a fresh fork:

1. Push to GitHub.
2. Repo **Settings** → **Pages** → **Deploy from a branch** → `main` /
   `(root)` → **Save**.
3. ~30 seconds later your site is live at
   `https://<username>.github.io/<repo-name>/`.

If you'd rather host elsewhere (Netlify, Vercel, your own server), upload
`index.html` and you're done.

## Controls

Each tool has its own collapsible panel of sliders, color pickers, toggles,
and inputs. Hovering shows labels; values update live, no apply step.
Per-tool section open/collapsed state is remembered within the session, so
switching tools and coming back keeps the panel exactly how you left it.

- **Reset** restores per-tool defaults.
- **Pause** + **Restart** affect the time clock.
- **Shift-drag** on the canvas scrubs the timeline while paused.
- **Drag** pans the camera (HTML export honors the PAN checkbox).
- **Scroll** zooms (HTML export honors the ZOOM checkbox).
- **Double-click** resets pan + zoom + scrub.
- **Light / Dark mode** toggle at the bottom — choice persists in
  `localStorage`.

The text input on every tool supports the pipe character `|` for line breaks
(up to 3 lines).

## Architecture (brief)

```
index.html
├── SECTION 1     Shared utilities (MotionNoise: value, looping, Perlin, FBM)
├── SECTION 2     Glyph animation
├── SECTION 3     Vinyl animation
├── SECTION 3.4   Waves
├── SECTION 3.46  Dots
├── SECTION 3.47  Particles (internal id: "gradient")
├── SECTION 3.48  Halftone
├── SECTION 3.5   Composer (segment registry + recipe engine)
└── SECTION 4     Main app (renderer, page nav, exports, Lab module)
```

Animations register themselves via `window.MotionAnimations.push({...})`.
Each declares:

- `id`, `label` — registry identifiers
- `bg`, `metaColor` — initial clear color + meta-text color
- `controls` — schema-driven UI (sliders, text inputs, color pickers,
  toggles, checkboxes, choices)
- `build(group)` — called once when the animation is selected; sets up
  Three.js objects on the provided group
- `update(t, params, state)` — called every animation frame
- `attractor(state, mx, my)` — optional cursor hook

State persistence: each animation's parameters live in their own object and
survive across switches. Only the explicit Reset button restores defaults.

The Lab module is a single IIFE near the bottom of Section 4 — its recipe
model, layer-state aliasing, marching-squares triangulation, mesh
deformation, and HTML/PNG/SVG/JSON export are all there. See `Handoff.md`
for a deeper map.

## Dependencies

- [Three.js r128](https://threejs.org/) — core 3D library
- Line2 / LineMaterial / LineGeometry from Three.js examples (variable-
  thickness lines)
- Hershey simplex Roman font data (public domain, embedded inline; ~7KB)
- Google Fonts (loaded once on Lab init for the font dropdown)

All loaded from CDN at runtime. No local dependencies.

## Browser support

Targets modern evergreen browsers with WebGL. Tested on Chrome, Safari,
Firefox.

## Credits

- Hershey simplex Roman font by Allen V. Hershey, 1967 (public domain)
- Three.js by mrdoob and contributors

## License

All rights reserved. No license is granted for use, modification, or
redistribution.
