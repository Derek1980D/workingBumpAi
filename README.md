# Pseudo-3D Relighting Demo

A single-file WebGL page that fakes real-time 3D lighting on a flat 2D image.
Normal maps and ambient-occlusion maps are derived from (or authored
alongside) the source image, then used to shade it per-pixel with a light
you can move around. Two tests live in one page, switchable from a dropdown.

- **Relic Pedestal** — a photographed/rendered object relit by a glowing
  marble you drag around. Pinch with two fingers to zoom/pan; one finger
  still just moves the marble.
- **Conduit** — a procedurally-authored pixel-art panel with a channel loop
  cut into it. A glowing ball rolls the loop on its own, lighting the groove
  as it passes, trailing a fading glow and flame behind it. One finger pans,
  two fingers pinch-zoom.

## Getting this onto GitHub Pages

1. Create a new repo (or open an existing one).
2. Upload the **contents** of this folder to the repo root — `index.html`
   needs to sit at the top level, not inside a subfolder. Dragging the whole
   extracted folder into GitHub's "Add file → Upload files" page in a
   desktop browser preserves the `assets/`/`scripts/` structure in one go.
3. Commit to `main`.
4. In the repo's **Settings → Pages**, set Source to "Deploy from a branch",
   branch `main`, folder `/ (root)`, then save.
5. The demo goes live at `https://<username>.github.io/<repo-name>/`
   within a minute or two.

Nothing else needs configuring — `index.html` has every texture baked in as
base64, so it doesn't reach into `assets/` at runtime. That folder is there
for your own reference/reuse, not because the page depends on it.

## Folder layout

```
pseudo3d-relighting/
├── index.html                    ← the demo — this is what Pages serves
├── README.md
├── assets/
│   ├── diffuse.png                Relic Pedestal's source image
│   ├── height-map.png             its height field
│   ├── normal-map.png             its normal map
│   ├── light-map.png              its ambient occlusion
│   ├── diffuse-conduit.png        Conduit's pixel-art panel (native 128×128)
│   ├── height-map-conduit.png     its authored height field
│   ├── normal-conduit.png         its normal map
│   └── light-map-conduit.png      packed RGBA: AO / ambient-glow mask /
│                                  track-progress / loop-only mask
└── scripts/
    ├── generate-maps.py           regenerate Relic Pedestal's maps from any photo
    └── generate-conduit-scene.py  regenerate Conduit's maps, channel shape as CLI flags
```

## Controls

**Relic Pedestal**
- Drag to move the light, or tap **Orbit** to let it sweep automatically
- **Relief** — exaggerates or flattens the bump effect
- **Glow** — recolors the pulse in the carved lines

**Conduit**
- **Roll** — play/pause the ball's loop
- Same **Relief** and **Glow** controls, which also tint the ball, its trail,
  and the flame

Both scenes: pinch to zoom, drag with two fingers to pan, **Reset** in the
corner returns to the default view.

## How it works

**Relic Pedestal**: grayscale luminance from the photo stands in for a
height map. A Sobel filter over that gives the local slope, which becomes a
per-pixel normal (`normal = normalize(-slope_x, slope_y, 1)`, encoded as RGB).
There's no mesh here, so no UV-unwrapping step — the image *is* the UV space.

**Conduit** is authored differently: rather than deriving height from a
photo, `generate-conduit-scene.py` builds it from signed-distance-field
math — the channel is "distance to a rounded rectangle's outline," carved
into a flat base height with a smooth falloff. The ball's path in the HTML
(`pointOnTrack()`) walks that exact same rounded rectangle, so it always
sits precisely on the channel, not just visually close to it. Textures are
kept at native 128×128 and loaded with `NEAREST` filtering rather than
`LINEAR`, which is what actually produces the pixel-art blockiness (and
keeps the files tiny).

**The trail** reuses that same track parametrization rather than tracking a
history of past ball positions: every pixel near the loop has its own
"nearest point on the track" progress value baked into the texture (0–1
around the loop). Comparing that to the ball's current progress gives an
exact "how long ago was this point passed" value, which drives an
exponential fade — cheaper and more precise than the usual position-history
approach, since the geometry is known up front.

**The flame** is layered noise (`fbm`, four octaves of value noise) scrolled
upward over time and masked to that same fading trail region, colored on a
dark-red-to-white ramp.

Shared logic — the pan/zoom transform, the Lambertian+specular lighting, the
glowing-orb effect used for both the marble and the ball — lives once in a
handful of GLSL string constants that get concatenated into each scene's
shader, rather than being copy-pasted per scene. GLSL has no `#include`, so
this is the standard way around that once more than one or two shaders need
the same logic.

## Regenerating the maps

```
python3 scripts/generate-maps.py your-photo.png output-folder/ --autocrop
python3 scripts/generate-conduit-scene.py output-folder/ --hw 0.68 --hh 0.50 --radius 0.20
```

`--autocrop` strips pure-black phone-screenshot UI bars. If you resize
Conduit's channel with `--hw`/`--hh`/`--radius`, update the matching `TRACK`
constant in `index.html`'s script so the ball keeps following the new shape.
Baking a fresh set of PNGs into a new self-contained `index.html` for either
scene is a quick follow-up — just ask.

## Notes on the repo itself

Everything here is plain text and PNGs — nothing big enough to need Git LFS
(a few MB all in). No build step, no dependencies, no `node_modules`.
