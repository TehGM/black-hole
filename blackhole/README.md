# Black Hole Wallpaper

A single-file WebGL2 black hole simulation built for wallpaper use: gravitational
lensing that bends the light of everything behind the hole (accretion disk arches
over and under it, background stars smear into Einstein rings), a seeded sci-fi
nebula/starfield background, and aggressive performance optimizations.

Open [index.html](index.html) in a browser, or point Wallpaper Engine / Lively
Wallpaper at it as a web wallpaper.

## Controls

Orbit the camera by dragging (mouse or one finger). Zoom with the mouse wheel
or a two-finger pinch on touch screens.

A slim toolbar sits on the right edge (dimmed until hovered):

- 🎲 **Generate new** — random seed with the current settings.
- ⚙ **Settings** — panel with every configurable value.
- ⭱ **Import** — scene code (paste) or JSON file.
- ⭳ **Export** — wallpaper link, scene code (copy), or JSON file.
- ✕ **Hide** — removes all UI (press **H** to bring it back).

## Moving a scene between copies

A **scene code** is the whole configuration as one line of text. Export it,
paste it into another copy under ⭱ Import, and you get the identical scene —
no network needed.

This is the way to configure a copy installed in a wallpaper host. Such a host
loads `index.html` straight off disk with no query string, so a shared link has
nothing to apply; a code pasted into the panel does. Once applied it is
remembered in `localStorage`, along with every later tweak, so the scene comes
back on its own from then on.

Every change is also reflected in the address bar: any value that differs from
the defaults becomes a query parameter, so the URL is always shareable. A link
takes precedence over the remembered local scene, so a link someone shares
always looks the way they saw it. Useful keys: `seed`, `ui=0` (start with hidden UI),
`cam=0` (lock the camera → fastest path), `yaw`/`pitch`/`dist`, `diskColor`
(`seed` or `rrggbb`), `nebulae`, `ambient`, `stars`, `motion`, `scale`, `fps`.
Example:

```
index.html?seed=42&cam=0&ui=0&diskColor=5588ff&nebulae=5&ambient=0.1
```

## Configuration

Everything lives in the `CONFIG` object at the top of `index.html`:

| Key | What it does |
| --- | --- |
| `seed` | Drives the whole look: nebula palette and shape, star layout, disk texture phase. Any 32-bit integer. |
| `cameraControl` | `true`: drag orbits the camera, wheel zooms. `false`: all input is removed and the renderer switches to a cached static path (see below) — same image, far less work. |
| `camera.*` | Initial/fixed orbit position (`yaw`, `pitch`, `distance`), zoom limits, `fov`. |
| `blackHole.*` | Disk radii, rotation speed, brightness, doppler asymmetry, photon-ring glow. |
| `blackHole.diskColor` | `null` = classic orange with a subtle seed tint, `"seed"` = hue fully derived from the seed, or an explicit `"#rrggbb"`. |
| `background.nebulaCount` | Number of distinct wispy nebulae placed on the sky (0–8). The first one is always placed within the default camera's view. |
| `background.nebulaIntensity` | Brightness of those nebulae. |
| `background.ambient` | Faint background haze / galactic band level; `0` = pitch-black sky between the nebulae. |
| `background.starDensity` / `starBrightness` | How many stars and how bright they are. |
| `background.twinkleSpeed` | Star twinkle rate. |
| `background.motion` | Subtle nebula drift. `0` makes the background fully static, which also unlocks the fastest static-camera path. |
| `quality.renderScale` | Internal resolution fraction (default `0.5`); the glow aesthetic upscales cleanly. |
| `quality.maxFPS` | Frame cap (default `30`, `0` = uncapped). |
| `quality.marchSteps` | Ray-march steps for the lensing (read once at load). |

Console API:

```js
BlackHole.setSeed(123)          // new look; omit the argument for a random seed
BlackHole.setSeed()
BlackHole.setCameraControl(false)
```

## How the static mode saves work

With `cameraControl: false` the bent-ray geometry per pixel never changes, so at
startup one multi-target pass bakes into float textures: the lensed
nebula + glow, the star color + twinkle phase, each bent ray's escape direction,
and the disk-plane crossing coordinates. After that, every frame is just a few
texture reads plus the animated disk/twinkle shading — no ray marching.

- `motion: 0` — the nebula is baked too; per-frame work is minimal.
- `motion > 0` — the drifting nebula is re-evaluated each frame from the cached
  ray direction (still no ray marching), so the background stays subtly alive.

The disk, twinkle, and nebula math is shared between both paths, so the image
is identical to the interactive renderer. If float render targets are
unavailable, it silently falls back to the full renderer (same appearance). 
