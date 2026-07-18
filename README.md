#!/usr/bin/env python3
"""
Generate height / normal / AO maps for pseudo-3D relighting from a single image.

Usage:
    python3 generate-maps.py input.png output_dir/ [--autocrop] [--maxdim 720]
"""
import argparse
import os

import numpy as np
from PIL import Image, ImageFilter


def autocrop_ui_bars(img, thresh=10.0):
    """Strip pure-black letterboxing/UI bars by finding the tallest
    contiguous run of rows brighter than `thresh`."""
    arr = np.array(img).astype(np.float32)
    row_brightness = arr.mean(axis=(1, 2))
    mask = row_brightness > thresh

    runs = []
    start = None
    for i, v in enumerate(mask):
        if v and start is None:
            start = i
        if (not v) and start is not None:
            runs.append((start, i))
            start = None
    if start is not None:
        runs.append((start, len(mask)))
    if not runs:
        return img

    top, bottom = max(runs, key=lambda r: r[1] - r[0])
    return img.crop((0, top, img.width, bottom))


def sobel_xy(h):
    kx = np.array([[-1, 0, 1], [-2, 0, 2], [-1, 0, 1]], dtype=np.float32)
    ky = np.array([[-1, -2, -1], [0, 0, 0], [1, 2, 1]], dtype=np.float32)
    padded = np.pad(h, 1, mode='edge')
    hh, ww = h.shape
    gx = np.zeros_like(h)
    gy = np.zeros_like(h)
    for dy in range(3):
        for dx in range(3):
            gx += kx[dy, dx] * padded[dy:dy+hh, dx:dx+ww]
            gy += ky[dy, dx] * padded[dy:dy+hh, dx:dx+ww]
    return gx, gy


def main():
    ap = argparse.ArgumentParser(description=__doc__)
    ap.add_argument('input')
    ap.add_argument('outdir')
    ap.add_argument('--autocrop', action='store_true',
                     help='strip pure-black UI bars (phone screenshots)')
    ap.add_argument('--maxdim', type=int, default=720,
                     help='resize so the longest edge is at most this many px')
    ap.add_argument('--blur', type=float, default=2.0,
                     help='height map denoise blur radius')
    ap.add_argument('--ao-blur', type=float, default=16.0,
                     help='AO neighbourhood blur radius')
    args = ap.parse_args()

    img = Image.open(args.input).convert('RGB')
    if args.autocrop:
        img = autocrop_ui_bars(img)

    w, h = img.size
    scale = args.maxdim / max(w, h)
    if scale < 1:
        img = img.resize((round(w * scale), round(h * scale)), Image.LANCZOS)
    W, H = img.size

    gray = img.convert('L').filter(ImageFilter.GaussianBlur(radius=args.blur))
    height_arr = np.array(gray).astype(np.float32) / 255.0

    gx, gy = sobel_xy(height_arr)
    allmag = np.abs(np.concatenate([gx.ravel(), gy.ravel()]))
    p995 = np.percentile(allmag, 99.5)
    strength = 0.9 / max(p995, 1e-6)

    # nx: image-x (rightward) matches world-x directly.
    # ny: array rows increase downward but world-y is up, hence the sign flip.
    nx = -gx * strength
    ny = gy * strength
    nz = np.ones_like(height_arr)
    vlen = np.sqrt(nx**2 + ny**2 + nz**2)
    nx /= vlen
    ny /= vlen
    nz /= vlen

    normal_rgb = np.stack(
        [(nx * 0.5 + 0.5) * 255, (ny * 0.5 + 0.5) * 255, (nz * 0.5 + 0.5) * 255],
        axis=-1,
    )
    normal_rgb = np.clip(normal_rgb, 0, 255).astype(np.uint8)

    wide = np.array(
        Image.fromarray((height_arr * 255).astype(np.uint8))
        .filter(ImageFilter.GaussianBlur(radius=args.ao_blur))
    ).astype(np.float32) / 255.0
    delta = height_arr - wide
    dscale = 3.2 / (np.percentile(np.abs(delta), 99.5) + 1e-6)
    ao = np.clip(0.55 + delta * dscale * 0.55, 0.15, 1.15)
    ao = ao / ao.max()

    os.makedirs(args.outdir, exist_ok=True)
    img.save(os.path.join(args.outdir, 'diffuse.png'), optimize=True)
    Image.fromarray((height_arr * 255).astype(np.uint8), 'L').save(
        os.path.join(args.outdir, 'height-map.png'), optimize=True)
    Image.fromarray(normal_rgb, 'RGB').save(
        os.path.join(args.outdir, 'normal-map.png'), optimize=True)
    Image.fromarray((ao * 255).astype(np.uint8), 'L').save(
        os.path.join(args.outdir, 'light-map.png'), optimize=True)

    print(f'Wrote diffuse / height-map / normal-map / light-map.png '
          f'to {args.outdir}  ({W}x{H})')


if __name__ == '__main__':
    main()
