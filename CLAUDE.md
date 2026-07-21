# Working on these wallpapers

This CLAUDE file was generated using Claude itself, based on chat history.

A GitHub Pages repo of self-contained animated web wallpapers, one folder per
wallpaper (`blackhole/`, `neonscape/`), each a single `index.html` with optional
assets if required. Root `index.html` is a gallery.

Nothing here assumes a particular rendering library or host. Wallpapers may use
raw WebGL, a library, or plain canvas/CSS, and must work as an ordinary static
page opened in a browser. Whatever a given wallpaper already uses, match it —
don't introduce a dependency to solve a problem the existing approach covers.

## Performance is the top priority

These run **continuously, in the background, on someone else's desktop**, while
they work or game. A wallpaper that looks marginally better but costs another few
percent of GPU is a worse wallpaper. CPU, GPU and RAM cost outrank visual
ambition — when they conflict, cost wins.

- Budget every feature and **measure it**, in milliseconds per frame at a real
  resolution. Features that come out inside noise are the ones worth keeping.
- Prefer folding work into a pass that already runs over adding a draw call or a
  full-screen quad. Extra fill is usually the expensive part.
- Compile features out rather than branching per pixel — a shader permutation
  behind a `#define` means a control at 0 costs literally nothing, not "a cheap
  early-out per pixel". Anything that can be disabled should cost nothing when it is.
- Precompute anything that can't change. If a mode makes per-pixel geometry
  static, bake it once at startup and reduce each frame to texture reads.
- Expose the knobs that buy the most: internal render scale (well below 1.0 is
  usually fine for glow-heavy looks), a frame cap, and quality/step counts.
- Idle should be cheap. Don't render when nothing changed.
- Watch RAM too: textures and buffers persist for the machine's entire uptime.

## Host compatibility

Wallpapers should run in a plain browser, but hosts that embed them can be far
stricter. The one target measured so far — Wallpaper Engine 2.8.42, rendering
off-screen in CEF 146 — **instantly hard-crashes** on two DOM features, with
`STATUS_ILLEGAL_INSTRUCTION` (0xC000001D) and no crash dump:

- **`<select>`** — any one that generates a layout box. `appearance:none`,
  `visibility:hidden`, `opacity:0`, `size=3` all still crash; only `display:none`
  survives, so styling or hiding one is not a workaround. Build a div dropdown.
- **`window.prompt` / `alert` / `confirm`** — any native modal. Easy to hit
  indirectly; a `catch` falling back to `prompt()` takes the wallpaper down.

Verified safe there: `input` range/number/color/checkbox (including the native
color chooser), `<input type=file>`, `<datalist>`, div dropdowns,
`backdrop-filter`, `accent-color`, `requestFullscreen`, blob downloads,
`navigator.clipboard`, `document.execCommand('copy')`.

Treat this as evidence that embedded hosts break in ways normal browsers don't —
prefer plain, widely-supported DOM, and verify anything exotic in the actual host.

## GPU portability — assume the user's hardware is stricter than yours

Found because the user's GPU rendered black boxes where mine rendered fine. Don't
rely on local rendering to validate a shader.

- **Never `pow(x, y)` with a possibly-negative base** — undefined, NaN on some
  drivers. Use exp/multiply falloffs.
- **Additive-blended transparent quads misrender** on some hardware; Points and
  Lines are fine. Additive is also only safe on dark backgrounds — over bright
  areas it clips every channel and washes out to white.
- **Size `discard` thresholds in sRGB, not linear.** The canvas is sRGB-encoded
  and its toe expands darks enormously — alpha 0.004 still renders ~29/255 and
  leaves a visible hard ring. Judge edges by per-pixel sRGB code-value steps, not
  by linear alpha.
- A falloff that must fade to nothing needs **compact support** — an exponential
  never reaches zero, so window it.
- **Don't float a mesh a fraction above another surface** to layer it — at a few
  hundred units it z-fights into stripes. Fold it into the underlying surface's
  shader instead.
- Share one snippet and the *same uniform objects* between things that must agree.
  Duplicated math drifts.

## Perspective, where a scene has a horizon

Anything meant to look right near the horizon must be computed in **screen** terms
(`gl_FragCoord`, or a camera-height/distance angle), never raw world depth —
perspective crams hundreds of world units into the last few pixel rows.

- A depth-based band near the horizon is invisible; use angular extents.
- Keep detail variation lateral. Appreciable depth-frequency becomes transverse
  banding no matter how well anti-aliased.
- At grazing angles, don't perturb the ray you sample a gradient with — a ripple
  bends it far more than its own elevation, sweeping a huge arc per pixel, and the
  gradient boils into stripes. Sample off the flat ray, modulate *brightness*.
- Physical Fresnel makes reflections vanish exactly where the viewer looks; raise
  the floor above physical.

## Verifying changes

`.claude/launch.json` serves the repo root via `npx http-server` on port 8471
(config name `wallpapers`).

The Claude Browser pane reports `document.hidden=true`, so rAF never fires and
screenshots time out. Give each wallpaper a debug hook that single-steps frames
synchronously and sample pixels instead of screenshotting. Traps that produced
completely bogus measurements:

- **Confirm your freeze actually renders.** Disabling animation to hold a scene
  still can hit a render early-out, so every subsequent step is a no-op and you
  sample a stale buffer. Freeze by passing a fixed timestamp instead.
- Single-stepping advances only ~16.7 ms per call; time-dependent effects need
  hundreds of calls.
- Sample **off-axis**. Scene centre often has a feature sitting exactly on it.
- Prefer `gl.readPixels` (origin bottom-left, flip y) over `drawImage`, which can
  grab a stale frame.
- Project a world position and assert its ndc range rather than assuming where it
  lands on screen.
- The pane's console buffer is tab-lifetime cumulative — old shader errors persist
  across reloads, so don't trust a stale error as current.

If a wallpaper's script is a module, its internals aren't global — the debug hook
is the only way in.

To test against a live embedded host, Wallpaper Engine's `config.json` sets
`--remote-debugging-port=8887 --remote-allow-origins=*`, so the running wallpaper
is drivable over CDP at `http://127.0.0.1:8887/json/list` (Node 24 has a global
`WebSocket`, no deps). Point the target at a local http-server copy, mutate, probe
`Runtime.evaluate`; a crash surfaces as `Inspector.targetCrashed` or a socket
close. Ablating one feature at a time off a known-crashing page pinpoints a cause
in minutes. Restore the user's wallpaper by navigating back to its original
`file://` URL when finished.

## Conventions and preferences

- **No save-migration code at this stage** — no version fields, no legacy-key
  shims. The user has been explicit about this; don't add it defensively.
- **Ship dependencies with the wallpaper** wherever that can legally be done for
  free — vendor the file into the wallpaper's own folder rather than pointing at a
  CDN, and include the upstream licence text next to it (MIT, BSD, Apache-2.0 and
  similar all permit this and only require keeping the notice). A wallpaper runs
  on a desktop that may be offline, behind a filter, or embedded in a host with no
  network at all, and a CDN that disappears breaks it years later. If a dependency
  can't be redistributed for free, say so and pick a different one.
- Pin the exact version, and note in the file where a refresh comes from.
- Size things in real proportions before reaching for glow. Correct proportions
  and subtle surface shading sell a form better than any effect.
- **Measure, don't assert.** Claims about performance and appearance in this
  project have repeatedly been wrong until probed.
- The user commits manually. Don't create commits unless asked.
