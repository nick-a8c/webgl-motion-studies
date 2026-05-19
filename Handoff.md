# Handoff — MotionKit

Single-file Three.js animation studio. HTML + CSS + every animation + every
page lives in `index.html` (~11.5k lines, ~430KB). No build step, no
`node_modules`, no bundler. Drop into any browser, or serve locally.

Last checkpoint: **2026-05-20**. Repo: <https://github.com/nick-a8c/motionkit>
(deployed at <https://nick-a8c.github.io/motionkit/>).

### 2026-05-20 micro-session

Three small but user-visible fixes:

- **Lab layer stacking now matches the Layers panel.** Layer index 0 (top of
  the panel) draws last so it composites in front. Implemented as
  `applyLayerRenderOrder()` called at the end of `rerender()` — sets
  `renderOrder = (n - 1 - i)` on each layer's group and traverses children.
  Reordering via ↑/↓ or drag-and-drop already calls `rerender()`, so the
  ordering follows automatically.
- **"+ Add layer" now inserts at index 0** (top of the stack) instead of
  pushing to the end. Mirrors the existing `+ Effects` behaviour. The new
  layer becomes active. Combined with the renderOrder fix above, a freshly
  added shape layer visibly sits on top.
- **Playground section collapse state persists per tool within the session.**
  A new module-level `SECTION_OPEN_STATE` map (keyed by `anim.id`) remembers
  every header toggle. `buildControls` consults it on tool re-entry; the
  per-tool `defaultOpenSections` whitelist is only applied for sections the
  user has not yet touched. Resolves the prior behaviour where switching
  Composer → Glyph → Composer reset everything back to defaults.

Earlier backups in `backups/`. The 2026-05-18 backup predates the Lab vector
fill / outline mesh, the deformed-mesh SVG export, the six new Lab effects,
the per-tool save-to-Playground flow, the export-row redesign, and the polish
pass — all listed below.

---

## Run it

```bash
cd "MotionKit"
python3 -m http.server 8765
# open http://localhost:8765
```

`.claude/launch.json` (in the `_01` workspace root) declares the same command
for the in-IDE preview, with `--directory` pointing at this folder.

---

## Top-level structure

- **Pages** (top nav): `Landing` · `Playground` · `Lab`. Landing is now the
  default on first load. `Playground` is the renamed `Create` page; the
  internal page id is still `create` to avoid breaking saved state.
- **Playground tools** (cards on the right): Composer · Glyph · Vinyl · Waves ·
  Dots · Particles · Halftone (7 in total). Poles and Distortion were removed
  this session — their IIFEs, `CATEGORY_ENTRIES`, and `HTML_EXPORT_SECTIONS`
  entries are gone.
- **Lab** is a separate top-level page with a fundamentally different
  architecture: layered subjects (text/SVG) + EFFECTS layers + global pixel
  post-processing. Every Lab fill / outline is now a **real triangulated
  vector mesh** that the effect pipeline deforms per-vertex.

---

## File map (`index.html`)

Every JS section is delimited by a top-level `// SECTION X.Y:` comment. The
HTML export for Playground tools relies on those exact markers to slice
sections.

| Section | What's there |
|---|---|
| `// SECTION 1:` | Shared utilities: `MotionNoise` (`hash2`, `valueNoise2D`, `loopingNoise`, `envelopePulse`, `perlin3`, `fbm3`). Exposed on `window.MotionNoise`. |
| `// SECTION 2:` | **Glyph** animation. Hershey simplex raw data + `getGlyph` + `buildTextPath`. Now also includes hidden Composer-style Primitive + Effects checkbox params; the actual picker UI is built in Section 4. |
| `// SECTION 3:` | **Vinyl** animation. |
| `// SECTION 3.4:` | **Waves** animation. SVG mode is a two-button choice (Fill / Outline). |
| `// SECTION 3.46:` | **Dots** animation. Default palette is now light gray + red + neutral gray. |
| `// SECTION 3.47:` | **Particles** animation (internal id `gradient`). |
| `// SECTION 3.48:` | **Halftone** animation. Wave effect + cursor attractor added this/prior session; cell coverage allows 0–200%. |
| `// SECTION 3.5:` | **Composer** — segment registry (sources / primitives / effects) + composer animation. Each primitive now clones its `LineMaterial` per item so the line-thickness gradient can vary `linewidth` per anchor. Three new motion effects (`orbit`, `glitch`, `pulse`) live here; Mirror/Kaleidoscope was removed earlier in the session. |
| `// SECTION 4:` | Main app — renderer, page nav, categories grid, `selectAnimation`, `buildControls` (with `hidden`-flagged controls + per-tool `defaultOpenSections`), mouse interactions, exports (JSON / PNG @ 1×/2×/3× / HTML with PAN/ZOOM gating), `buildSegmentPickers` (Composer + Glyph), shared overlay shaders (gradient / pixelate / halftone), and the **Lab module** (~3000 lines, an IIFE near the bottom of Section 4). |

Every Playground animation IIFE ends with `window.MotionAnimations.push({...})`.

---

## Playground architecture model

Same as the previous handoff — single global `window.MotionAnimations`
registry, per-anim `ANIM_STATE[anim.id] = { params, defaults }`,
`INITIAL_DEFAULTS` patches in Section 4, recipe-driven Composer with exposed
`window.MotionSegments`, etc. New this session:

- **Shared overlay system**. Composer / Glyph / Waves / Particles get an
  appended `OVERLAY_CONTROLS` section trio (Gradient / Pixelate / Halftone)
  that drives a render-target chain wrapped around the main loop. Gradient
  mask is shape-only via `bgColor` distance.
- **PNG export** at 1×/2×/3× resolution. Renderer is `alpha: true,
  premultipliedAlpha: false` so transparent PNGs are real. Exports
  temporarily resize the drawing buffer to 2048/4096/6144 px, run the overlay
  chain, dataURL, restore.
- **HTML export** has PAN + ZOOM checkboxes that gate `mousedown`-pan and
  `wheel`-zoom listeners in the runtime template. JS-only export was removed
  from the UI (the function is still used internally by Composer's HTML
  export path).
- **Tool polish**: Composer's name defaults to "The Big O" with cursor
  strength `-0.8`; default-open sections are curated per tool (e.g., Composer
  shows only `Source — Circle` + `Cursor`, everything else collapsed). Defaults
  in `INITIAL_DEFAULTS` got a big refresh for Dots / Waves / Particles /
  Halftone — see the diff.
- **Save to Playground**. Lab's "Save to Create" button writes
  `{id, name, recipe}` to `localStorage["motionkit-saved-tools"]`. They
  render at the bottom of the Playground grid with a yellow `lab` tile;
  click → switches to Lab + `applySavedRecipe`; hover × removes.

---

## Landing page

Vertically centered hero plus a single tagline. The hero renders **211 Line2
instances** along the SVG-star outline (parsed inline at startup via
`window.MotionSegments.parseSvgAsset`). Each Line2 carries its own
`LineMaterial`; per-anchor `linewidth` thickens by up to **40%** with a
cosine-feathered 400px radius around the cursor — pointer position is mapped
to world coords inside the canvas's `mousemove` handler.

---

## Lab page

### Architecture

```js
recipe = {
  subjects: [
    // shape layer
    { type: 'text', content: 'A8C', style: 'fill',
      color: '#3858E9', bgColor: '#F2F2F2',
      subjectScale: 0.9, meshDensity: 97, strokeWidth: 8,
      particleCount, particleSize, cellSize,
      particleMode, particleDotStyle, particleOutlineThickness,
      particleRepulsion, fontFamily,
      gaussianBlur, roundedCorners,
      effects: [], hidden: false, svg: null },
    // effects layer (auto-created on first global motion/pixel click)
    { type: 'effects', label: 'EFFECTS',
      effects: [{id, params}], pixelEffects: [{id, params}], hidden: false },
    // …
  ],
  activeLayer: 0,
  globalMotion: [],   // legacy; new pickers route into the active EFFECTS layer
  globalPixel:  [],
  transparent: false,
};
```

`recipe.subject` is a **non-enumerable getter alias** for
`recipe.subjects[recipe.activeLayer]`.

### Per-layer state (`layerStates[i]`)

Each layer owns its own:

- `group`, `lineMat`, `fillMat`
- `baseAnchors`, `particleMesh`, `workingAnchors`, `dotSize` (Particles /
  Halftone)
- `fillMesh`, `fillBaseVerts`, `fillWorking` (Solid Fill / Outline — both use
  the same vector-mesh pipeline)
- `renderables` (objects to dispose/clear on rerender)

Module globals (`lineMat`, `fillMat`, `baseAnchors`, `particleMesh`,
`renderables`, `group`, `loadedSvg`, `_workingAnchors`, `dotSize`, `fillMesh`,
`fillBaseVerts`, `fillWorking`, `R` for the active recipe in the HTML-export
runtime) are aliases pointing at the active layer's state. `activateLayer(i)`
re-points all of them; `saveActiveLayer()` writes any reassignments back to
the layer state.

### Type × Style matrix (8 renderers)

| | Text | SVG |
|---|---|---|
| Outline | Stroked text → canvas → **marching squares → ShapeGeometry + holes + subdivide → fillMat mesh** | Stroked SVG path → canvas → same marching squares pipeline |
| Solid fill | Filled text → canvas → marching squares → ShapeGeometry + holes + subdivide → fillMat mesh | SVG path points → THREE.Shape (with hole containment) → ShapeGeometry + subdivide → fillMat mesh |
| Particles | Hershey strokes (line) or rasterised mask (fill) → arc-length distribution → InstancedMesh | SVG paths or rasterised mask → arc-length distribution → InstancedMesh |
| Halftone | Rasterised text → Bayer 4×4 dither grid → InstancedMesh | Rasterised SVG fill → Bayer dither |

Outline and Fill share the **`_labVectorFill`** marker and the
`extractMeshBoundary` SVG-export path.

### Solid Fill / Outline — full vector mesh

The big lift this session. Both styles now produce a real triangulated polygon
mesh whose vertices the effect pipeline mutates each frame:

1. **Contour extraction**.
   - Text & SVG-outline: rasterise into a 1024² alpha mask; run
     `marchingSquares(mask,w,h)` (16-case edge-segment table +
     endpoint-matched threading) to get sub-pixel-accurate closed contours.
   - SVG fill: use the asset's path points directly — already polygonal.
2. **Shape assembly**. `contoursToShapes(worldContours)` classifies by signed
   area (`polyArea > 0` = CCW outer, `< 0` = CW hole), assigns each hole to
   the smallest containing outer via `pointInPoly`, and emits
   `THREE.Shape` objects with `shape.holes` populated. For the text path the
   Y-flip from canvas → world inverts winding, so each polygon is reversed
   before classification.
3. **Triangulation**. `new THREE.ShapeGeometry(shapes)`.
4. **Subdivision**. `subdivideGeometry(geom, passes)` runs midpoint splits
   (every triangle → 4 smaller ones) with a shared-edge midpoint cache so
   neighbouring triangles can't crack apart under heavy deformation.
   `meshDensity` 4–100 maps to 0/1/2/3 passes.
5. **Render**. Mesh uses the layer's `fillMat` (color, no texture). Marked
   `_labVectorFill = true`.
6. **Deform**. `writeFillTransforms(t)` seeds `fillWorking[i]` from
   `fillBaseVerts`, runs (local effects) → (cascade from earlier
   effects-layers) → (global motion), writes mutated x/y back to
   `geometry.attributes.position.array` and sets `needsUpdate = true`. Rot
   and scale anchor fields are ignored — they have no meaning on a 2D mesh.
7. **SVG export**. `extractMeshBoundary(geom)` walks the indexed triangle
   list: edges shared by exactly one triangle are boundary edges. Directed
   edges (from each triangle's winding) chain into closed loops. Each loop
   becomes an `M … L … Z` subpath in a single
   `<path d="…" fill-rule="evenodd">`, so the deformation + holes survive
   exactly.

### Effects layers

User-visible model: every motion/pixel effect lives on an EFFECTS layer.
The col-3 pickers no longer write to `recipe.globalMotion` /
`recipe.globalPixel` directly — they target the active layer's `effects` /
`pixelEffects` arrays. Picking a Motion or Pixel effect from col 3 while a
shape layer is active auto-creates an EFFECTS layer at index 0
(`recipe.subjects.unshift`, `layerStates.unshift(makeLayerState())`). The
"+ Effects" button also unshifts so its cascade reaches every layer beneath.

Cascade rule: an EFFECTS layer at index `j` applies its effects to every
non-effects layer at index `i > j` (the layers rendered after it). The
cumulative-cascade flag is rebuilt per tick.

Param panels for selected effects render in **col 2 under the Layer name**
input. The Motion + Pixel pickers live in col 3 with the order-badge
convention (numbered top-to-bottom). The orientation **flips visually** for
params: the latest-added effect shows as `1.` at the top of the list because
it runs last and visually dominates everything below — apply order in `tick`
is unchanged.

Lab-only filter list (`LAB_MOTION_HIDDEN`) hides `rotation-wave` and
`spiral-attractor`. Pixel effects removed entirely from Lab: `blur`,
`vignette`, `posterize`. They're still available in Composer (the motion
ones).

### Six new effects (this session)

Motion (live in `SEGMENTS.effects`, available everywhere):

- **Orbit** — per-anchor rotation around a movable centre with `1/r` speed
  boost for inner anchors.
- **Glitch** — per-anchor jagged offsets quantised to frames; `Chance` picks
  what fraction flickers each frame.
- **Pulse** — radial Gaussian-bell displacement that sweeps outward from a
  centre at `Period`/`Band width`.

Pixel (in `LAB_PIXEL_EFFECTS`, Lab-only):

- **Bloom** — bright-pixel extraction + 7×7 blur + additive blend.
- **Kaleidoscope** — polar fold + intra-segment mirror.
- **Edge detect** — 3×3 Sobel on luminance, mixed between Edge / Fill colors.

### Per-style defaults (new)

`STYLE_DEFAULTS` table applied on every style swap. Only listed keys are
overwritten:

- Solid fill (initial): subjectScale 0.9, meshDensity 97, color `#3858E9`,
  bg `#F2F2F2`.
- Outline: adds strokeWidth 15.
- Particles: subjectScale 0.9, mode `fill`, outline thickness 0.16,
  repulsion 0.13, count 460, size 18.
- Halftone: dot style `outline`, thickness 0.19, repulsion 0.27, particle
  size 8, cell size 10.

### Lab UI specifics

- **Layers panel** now uses SVG icons for up/down/×/eye. Rows are
  HTML5-draggable with `drop-before` / `drop-after` indicators; reorder swaps
  both `recipe.subjects` and `layerStates` and rewrites `activeLayer`.
- **Card heights** are pinned to the stage canvas height every tick (cheap
  bail when size unchanged). Each card scrolls internally.
- **Pause / Restart**: `labPlayback` carries `paused`, `lastT`, `startOffset`.
  Shift-drag on the canvas scrubs `labPlayback.lastT` while paused.
- **Transparent toggle** is a checkbox under Pause/Restart. When on, clear
  color uses alpha 0, overriding `bgColor`.
- **Section/effect collapse** chevron is `▾` open / `^` collapsed (text swap,
  no CSS rotation).
- **EFFECTS layer** rendering is skipped (no group children); `hidden` on an
  EFFECTS layer disables its motion cascade *and* its pixel chain.
- **Font dropdown** seeds from `LAB_FONTS` (System mono + 15 Google Fonts).
  `<link>` is injected once per session.

### Exports

- **JSON snapshot (v2)**: subjects[] + activeLayer + globalMotion +
  globalPixel + transparent. SVG assets inlined as raw XML on the layer's
  subject. v1 (single-subject) format still loadable.
- **SVG**: walks every layer's renderables and emits real SVG elements
  (polyline, polygon, circle, text). For Solid Fill / Outline meshes,
  `extractMeshBoundary` + `<path fill-rule="evenodd">` carries the deformed
  vertex positions.
- **PNG**: oversamples to 2048² (CSS untouched), runs the pixel chain, grabs
  `toDataURL`. Restores renderer state after.
- **HTML**: still active-layer-only (multi-layer + global motion + pixel
  chain in the runtime is the remaining big follow-up).

---

## New gotchas this session

| Trap | Symptom | Fix |
|---|---|---|
| Marching squares on a Y-flipped mask produces inverted winding | Letter holes render as filled, outers vanish | After mapping pixel coords → world Y-up, reverse each polygon before passing to `contoursToShapes`. |
| ShapeGeometry doesn't add interior vertices | Heavy deformation looks angular | Midpoint subdivision with a shared-edge midpoint cache (so neighbouring triangles share new verts). |
| `recipe.subjects.unshift` without also unshifting `layerStates` | Newly inserted EFFECTS layer reads the wrong per-layer state | Always pair `recipe.subjects.unshift(layer)` with `layerStates.unshift(makeLayerState())`. |
| `+ Effects` button pushed at end of stack | EFFECTS layer never reached the shape layer via cascade | Switched to `unshift` so its effects flow downward to every layer beneath. |
| `controls-host-meta` flex-collapsing to 0px in the Playground middle column | Name input visually stacked on top of the Recipe heading | `flex-shrink: 0` on `.controls-host-meta` + `#segment-pickers`. |
| Canvas filter `blur(N) contrast(20)` gave fuzzy "rounded corners" | Visible halo, edges not crisp | Render with `blur(N)` only, then `getImageData` + hard alpha threshold (>127 → 255, else 0). Gaussian blur is now a separate optional pass. |
| Renderer `alpha:false` made transparent PNG impossible | PNG always had a solid bg | Create Playground & Lab renderers with `alpha: true, premultipliedAlpha: false`. Set clear color alpha 0 when Transparent is on. |
| Aliased edges in transparent PNG export | Stair-stepping at viewport size | Oversample: `renderer.setSize(2048,2048,false)` + update Line2 `material.resolution` for the export, then restore. |
| Hidden controls still rendering | Duplicate UI between left panel and middle-column Recipe picker | Added a `hidden: true` flag in `buildControls`. The param still seeds `ANIM_STATE.params`, but no row renders. |
| Composer Circle source + Circle outline primitive was disabled | "Circle outline" greyed out | `SEGMENT_COMPATIBILITY['parametric-circle']` was missing `'circle-outline'`. Added. |
| Two "Enable gradient" checkboxes (thickness gradient + overlay gradient) | Confusing labels | Renamed the thickness one to "Enable thickness gradient". |
| Transparent overlap of identical-color shape layers reads as "both deformed" | User thought EFFECTS was leaking onto the wrong layer | Cascade is correctly scoped (verified by hiding the deformed layer). Documented; offer per-EFFECTS-layer targeting as a future feature if needed. |

Pre-session gotchas (still apply): `<\/script>` escaping, GLSL `half` keyword,
`align-self: start`, exposed segment helpers, `extractMethod` regex for method
shorthand, per-row materials for varying thickness, `HTML_EXPORT_SECTIONS` map
updates.

---

## Decisions log this session

- **Lab Solid Fill is a real triangulated mesh, not a textured plane.** The
  raster-to-grid intermediate (PlaneGeometry + CanvasTexture, ~2 sessions
  ago) was replaced because vector silhouettes survive zoom and SVG export.
- **Marching squares for text contour extraction**. Browsers don't expose
  font path data without an external parser; rasterise → MS → triangulate is
  the most reliable single-file path.
- **EFFECTS layer pickers replace the old global-motion / global-pixel
  arrays.** Saved JSON snapshots still carry the legacy arrays — both are
  read at apply time, but new clicks never write to them.
- **Particle / Halftone Outline dot style** uses `RingGeometry` instead of a
  two-mesh ring approximation.
- **"Save to Create" lives in `localStorage`**, not in JSON files. Each entry
  carries its full snapshot so loading is self-contained.
- **PNG always supersamples to 2048+**. Display-size PNG was visibly stair-
  stepped on transparent backgrounds; the fixed 2k baseline is the simplest
  knob that solves it across all tools.
- **`hidden: true` flag in `buildControls`** chosen over deleting the
  control entry, so params keep seeding `ANIM_STATE` while the row stays
  out of the left panel.
- **Mirror/Kaleidoscope motion effect was removed earlier in the session**
  (the Kaleidoscope *pixel* effect this session is unrelated). Snapshots
  referencing it silently skip — defensive lookup is
  `SEG.effects[id] || LAB_MOTION_EXTRA[id]`.

---

## Open / paused threads

- **HTML export for Lab — still active-layer only.** Multi-layer composition,
  global motion, and pixel post-process in the exported runtime is the
  largest remaining lift (~300–500 LoC). The studio-side rendering already
  works exactly right; this is a runtime-template extension.
- **Per-EFFECTS-layer targeting** ("apply to only layer X, not all below")
  — user mentioned it but didn't ask for it.
- **Effects on Glyph with per-effect param tuning**. Glyph currently has
  Composer-style Primitive + Effects pickers in the middle column, but each
  picked effect runs with the SEGMENTS-default control values. Dynamic
  per-effect prefixed params (`eff0_strength`, `eff0_scale`, …) would give
  full parity with Composer.
- **Save-to-Playground for Lab tools currently navigates back into Lab to
  view/edit them.** A nicer experience would render the saved tool in
  Playground's stage directly, but that requires teaching the Playground
  loop how to host a Lab recipe — non-trivial.
- **Drag-and-drop Layer reorder** works in Lab; could also be added to the
  per-effect blocks in Lab's col 2 (currently they have ↑/↓ buttons only).
- **Distortion card / Poles** — removed entirely. If you want them back, the
  git history shows the IIFEs that were deleted.

---

## Where to start in a fresh chat

1. Read this file.
2. Scan `// SECTION` markers in `index.html` (~30 seconds).
3. Start the dev server and verify (~2 minutes):
   - Landing page loads, hero star animates, lines thicken under cursor.
   - Playground → click each tool card (Composer, Glyph, Vinyl, Waves, Dots,
     Particles, Halftone); each should render with the curated defaults.
   - Lab → "A8C" renders as solid blue on light gray. Switch styles and the
     per-style defaults kick in. Add an EFFECTS layer + Position noise → the
     fill mesh deforms.
4. Ask the user what they want to do next; don't assume.

## Workflow preferences (still applicable)

- **Asks for many questions before big features.** Use `AskUserQuestion` with
  3–4 multi-choice questions per round.
- **Verifies via screenshots, not text reports.** Use `preview_screenshot`
  after each meaningful change.
- **Prefers signed sliders** + `Softness` for falloff curves.
- **"Lock in defaults" requests are common** — patch `INITIAL_DEFAULTS`,
  not segment defaults (which would affect all consumers).
- **Backups before risky work.** When the user says "let's wrap this up",
  they often want a snapshot preserved.
- **`node --check` for syntax verification without browser reload** — works
  on isolated JS but not on `.html`. Eval-in-preview is the substitute.
- **"Drag and drop" usually means the mental model**, not literal DnD UI —
  multi-select with order badges is what they actually want. Lab Layers do
  have real HTML5 drag-and-drop, but that was a specific ask.
- **One feature per session.** When context is tight, wrap and update this
  file rather than pushing through a partial implementation.
- **Terse chat output.** The user explicitly wants minimal narration —
  toolcalls speak for themselves; only summarise what landed at the end of
  the turn.
