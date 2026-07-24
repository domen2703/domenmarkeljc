# podium.global — hero→next-section "through the logo" transition

**Status:** `DONE_BASELINE_VERIFIED` (mechanism). Target locked, wiring verified numerically.
**Route:** runtime object trace — `gl.getParameter(CURRENT_PROGRAM)` + `getShaderSource()` + live uniform sampling across scroll positions.

## Target lock

| Fact | Value | Truth |
|---|---|---|
| Surface | single `<canvas>` webgl2, fullscreen, `1620×1928` backing @dpr2 | SOURCE |
| Runtime | Three.js (bundled — `__THREE__` present, no global `THREE`) | SOURCE |
| Scroll driver | Lenis + GSAP | SOURCE |
| Transition owner | the WebGL program itself + mesh Z translation | SOURCE |

## Discriminating probe (the decisive one)

Sampled at scrollY 0 → 1735 (viewport 964):

- `canvasTransform`, `wrapTransform`, `wrapClip` = **`none` at every position** → CSS/DOM scaling **eliminated**
- `uScrollProgress` 0 → 0.2998 → 0.8999 → 1.0
- `modelViewMatrix[14]` (translate Z) −5 → −3.501 → −0.5005 → 0

**Exact wiring (verified at 4 sample points):**

```
meshZ = -5.0 * (1.0 - uScrollProgress)
```

The plane physically travels from z=−5 to z=0 and passes *through* the camera. Projection is a real
perspective matrix (not orthographic), so apparent growth is true perspective, not a linear scale.
This is the single most important fact: it is why it reads as flying *through* an opening rather than
watching a logo scale up.

## Mechanism (fragment shader structure — not reproduced verbatim)

Declared helpers: `hash`, `noise`, `barrelPincushion`, `sdCircle`, `luminance`, sRGB transfer fns.

1. **Logo is an SDF sampled from a texture**, not geometry — `uBackgroundPodiumShapeTexture` (512×512).
   The mask is `smoothstep()` over that SDF, so the silhouette edge stays perfectly sharp at *any*
   magnification. A rasterised/DOM-scaled logo cannot do this — it softens and pixelates as it grows.
2. **Shape morph:** the texture SDF blends toward `sdCircle` via `uShapeReveal`, with an `edgeGuard`
   (`smoothstep(0.0, 0.05, edgeDist)`) so the blend never tears the silhouette at the boundary.
3. **Scroll-driven detail falloff:** `fnoise = pow(1.0 - uScrollProgress, 4.0) * noise(uv*1000) * NOISE_INTENSITY`
   — grain fades on a **quartic** curve, so the surface visibly cleans up as it rushes past.
4. **Intra-shape parallax:** `podiumUv.y += (1.0 - uScrollProgress) * (1.0 - resolutionRatio) * 0.3`
   — the texture drifts vertically inside the shape during the transition.
5. **Barrel/pincushion distortion** at intensity `6.0` warps UV space so the plane reads as a curved
   lens (a portal), not a flat card.
6. **Mouse trail** (`uMouseTexture`) is multiplied by that same scroll-faded noise, so pointer
   interactivity naturally hands off to the transition.
7. **Pulse:** `uvScale = uUvScale - (uPulseReveal * uPulseMult)` — a breathing scale on the UVs.

## Constants observed at rest

`uUvScale 1.4` · `uBarrelIntensity 6.0` · `uNoiseIntensity 0.6` · `uFluidIntensity 0.6`
`uPulseMult 0.2` · `uShapeReveal 1` · `uPulseReveal 1` · `uTextureShapeSize [512,512]`

## What this means for our port

We reimplement the **technique**, with our own mark, palette and copy — no assets, shader text,
branding or content copied.

| Ours (before) | Podium technique | Why it matters |
|---|---|---|
| CSS `transform: scale()` to 16× on a DOM wrapper | mesh translated on Z through a perspective camera | true perspective ⇒ reads as travelling *through*, not zooming |
| logo = rasterised 2D-canvas texture | logo = SDF ⇒ `smoothstep` mask | edge stays sharp at any scale |
| linear-ish grow, constant grain | grain fades `pow(1-p, 4)` | surface "cleans up" into the transition |
| flat plane | barrel distortion 6.0 | portal/lens feel |
