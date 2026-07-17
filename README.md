# Pseudo-3D Relighting Demo

A single-file WebGL page that fakes real-time 3D lighting on a flat 2D image.
A normal map and an ambient-occlusion map are derived from the image itself,
then used to shade it per-pixel with a light you can move around.

## Files

| File | What it is |
|---|---|
| `pseudo3d-lighting-demo.html` | The demo. Self-contained — the three textures below are baked in as base64, no external files or server needed. |
| `diffuse.png` | The source image. |
| `height-map.png` | Grayscale elevation guess (brightness → height). |
| `normal-map.png` | Per-pixel surface direction, derived from the height map. This is what the shader reads for lighting. |
| `light-map.png` | Ambient occlusion — darkens grooves and undercuts. |
| `generate-maps.py` | Regenerates the four PNGs above from any new image. |

## Running it

Open `pseudo3d-lighting-demo.html` in a browser — double-click it, no build
step, no dependencies, no server. Works fine hosted on GitHub Pages too if
you want a shareable link.

**Controls**
- Drag on the image to move the light, or tap **Orbit** to let it sweep automatically
- **Relief** — exaggerates or flattens the bump effect
- **Glow** — recolors the faint pulse in the carved lines

## How it works

1. **Height map** — the image's grayscale luminance, lightly blurred to cut noise.
2. **Normal map** — a Sobel filter over the height map gives the local slope
   in x and y; `normal = normalize(-slope_x, slope_y, 1)` turns that into a
   per-pixel surface direction, encoded as RGB (`0..255` ↔ `-1..1`).
3. **AO map** — wherever the height map sits below its own local blurred
   average (grooves, undercuts), it's darkened. Cheap ambient occlusion.
4. **Shader** — a flat textured quad. Each fragment samples its normal,
   computes `dot(normal, lightDirection)` for diffuse shading plus a small
   Blinn-Phong specular highlight, multiplies in the AO map, and adds a
   faint emissive pulse wherever the diffuse texture is dark (the rune grooves).

There's no mesh here, so there's no UV-unwrapping step — the image *is* the
UV space, sampled `0..1` directly.

## Using it on a different image

```
python3 generate-maps.py your-image.png output-folder/ --autocrop
```

- `--autocrop` strips pure-black letterboxing/UI bars — useful for phone
  screenshots like the "Generation Details" one this was built from. Leave
  it off for a clean render with no chrome.
- `--maxdim 720` (default) caps the longest edge, keeping file size sane.
- Works best on fairly monochrome, evenly-lit source images — heavy colour
  variation gets misread as fake elevation.

The script writes the same four PNGs. Baking a fresh set into a new
self-contained HTML like this one is a quick follow-up if you want it —
just ask.

## Putting this in a git repo

Works as-is — plain text and PNGs, nothing big enough to need LFS (~2.8 MB
all in).

```
git init
git add .
git commit -m "pseudo-3D relighting demo"
git remote add origin <your-repo-url>
git push -u origin main
```

Worth knowing for later: this opens straight from disk *because* the
textures are embedded as base64. If you ever split them back into separate
files loaded via `fetch`/`<img>`, some browsers block that under `file://`
for WebGL texture uploads — you'd need to actually serve it (GitHub Pages,
or `python3 -m http.server` locally) rather than double-click it.
