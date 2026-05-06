# WebGL Motion Studies

A small browser-based exploration tool for motion graphics. Built as a single HTML file using Three.js. Two animations included:

- **Glyph** — animated shapes (square, circle, triangle) traveling along letterform paths from the Hershey simplex stroke font. Each character gets independent shape distribution, with rotation envelopes, looping noise, and a per-character wave toggle.
- **Vinyl** — text rendered as concentric ring grooves, with each ring sliced into arc segments that rotate around screen center. Includes Perlin-noise-driven per-segment rotation and a radial gradient mask that scales the overall effect strength.

## Live demo

👉 **[Try it in your browser](https://nick-a8c.github.io/webgl-motion-studies/)**

## Preview

<!-- Replace with actual screenshot or GIF -->
![Preview](preview.png)

<!--
Suggested capture:
- 1280×800 viewport
- Glyph default (text "A8C", red on light gray) animating
- ~3 second GIF or a still PNG
-->

## Run

It's a single static HTML file. Open it any way you like:

```bash
# Just open in a browser:
open index.html

# Or serve it (some browsers behave better with HTTP than file://):
python3 -m http.server 8000
# then visit http://localhost:8000
```

No build step, no `npm install`. Three.js and the Line2 modules are loaded from CDNs at runtime.

## Hosting

The repo is set up to work with **GitHub Pages** out of the box (no config files needed — `index.html` at the root just works).

To enable it on a fresh GitHub repo:

1. Push this repo to GitHub
2. Go to your repo's **Settings** → **Pages** (in the sidebar)
3. Under **Source**, choose **Deploy from a branch**
4. Pick `main` branch, `/ (root)` folder, then click **Save**
5. Wait ~30 seconds; your site will be live at `https://<your-username>.github.io/<repo-name>/`

Update the [Live demo](#live-demo) link above with your URL once it's live.

If you'd rather host elsewhere (Netlify, Vercel, your own server), just upload `index.html` — there's nothing else to deploy.

## Controls

Each animation has its own panel of sliders, color pickers, toggles, and a text input. Hover over any control to see its label; the value updates live with no apply step. The **Reset** button restores per-animation defaults; **Pause** and **Restart** affect the time clock. There's a **Light/Dark mode** toggle at the bottom — choice persists in localStorage.

The text input supports the pipe character `|` for line breaks (max 3 lines).

## Architecture (brief)

```
index.html
├── SECTION 1: Shared utilities (noise functions: value, looping, Perlin, FBM)
├── SECTION 2: Glyph animation (Hershey font + path-following shapes)
├── SECTION 3: Vinyl animation (text mask + concentric rings + Perlin)
└── SECTION 4: Main app (renderer, animation registry, UI plumbing)
```

Animations register themselves via `window.MotionAnimations.push({...})`. Each animation declares:

- `id`, `label` — registry identifiers
- `bg`, `metaColor` — initial canvas background and meta-text color (canvas bg becomes a control in the panel)
- `controls` — schema-driven UI: sliders, text inputs, color pickers, toggles
- `build(group)` — called once when the animation is selected; sets up Three.js objects on the provided group
- `update(t, params, state)` — called every animation frame

State persistence: each animation's parameters live in their own object and survive across animation switches. Only the explicit Reset button restores defaults.

## Dependencies

- [Three.js r128](https://threejs.org/) — core 3D library
- Line2 / LineMaterial / LineGeometry from Three.js examples (variable-thickness lines for Glyph)
- Hershey simplex Roman font data (public domain, embedded inline; ~7KB)

All loaded from CDN. No local dependencies.

## Browser support

Targets modern evergreen browsers with WebGL. Tested on Chrome, Safari, Firefox.

## Credits

- Hershey simplex Roman font by Allen V. Hershey, 1967 (public domain)
- Three.js by mrdoob and contributors

## License

All rights reserved. No license is granted for use, modification, or redistribution.
