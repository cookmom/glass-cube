# Glass Cube Clock — Technical Notes

**Project:** Dichroic glass prism cube with animated clock-hand dispersion rays  
**Stack:** Three.js r169, vanilla JS, served via Python HTTP server  
**Started:** 2026-02-24  
**Author:** chef + Chris (lookdev) — Seven Heavens Studio

---

## Concept

A glass cube sitting on a 18% grey studio floor, lit as a precision prism. Light enters from a focused penlight, disperses through the glass into spectral rays, and the rays animate as clock hands (hour, minute, second) sweeping the floor. The cube acts as both optical instrument and clock face.

---

## Core Architecture

### FBO Dichroic Glass Shader
The cube uses a custom `ShaderMaterial` with a Frame Buffer Object (FBO) render pass — NOT `MeshPhysicalMaterial` transmission.

**Why FBO over MeshPhysicalMaterial:**
- MeshPhysicalMaterial transmission shows the screen-space background behind the glass, which in a dark scene = dark glass. The cube appeared darker than the floor.
- FBO technique renders the scene *without* the cube first, captures that as a texture, then the glass shader samples it with per-channel IOR offsets to create physically-correct chromatic dispersion.
- Per-channel IOR (R=1.14, G=1.18, B=1.23) creates true chromatic dispersion — red refracts less, blue more. This is the physically accurate dichroic glass effect.

**FBO Loop:**
```javascript
cubeMesh.visible = false;
renderer.setRenderTarget(fboRT);
renderer.render(scene, camera);     // Scene without cube → FBO texture
renderer.setRenderTarget(null);
cubeMesh.visible = true;
cubeMat.uniforms.uScene.value = fboRT.texture;
renderer.render(scene, camera);     // Main render with glass shader
```

**When to use MeshPhysicalMaterial instead:**
- When the scene has a bright background / rich environment (HDRI)
- When you want physically accurate caustics and don't need custom IOR per channel
- For quick prototyping — `transmission: 1.0, ior: 1.65, thickness: 0.9` is fast

### Aspect Ratio Correction in FBO Shader
**Problem discovered:** When viewport is non-square (portrait phone), the FBO shader was applying equal UV offsets in X and Y. But 1 UV unit in Y covers more pixels than in X on a portrait screen — so the refraction stretched vertically, making the cube appear horizontally squeezed.

**Fix:** Pass `uAspect = W/H` as a uniform, correct the refraction offset:
```glsl
vec2 abXY = vec2(uAb / uAspect, uAb);
float R = texture2D(uScene, clamp(uv + rR.xy * abXY, 0.001, 0.999)).r;
```

### Thin-Film Dichroic Iridescence
A diagonal face perturbation creates the dichroic "rainbow interference" effect:
```glsl
float diagF = exp(-abs(vLocalPos.x + vLocalPos.y) * 7.0) * uDich;
vec3  dn    = normalize(mix(n, normalize(vec3(1.0, 1.0, 0.0)), diagF));
vec3  irid  = thinFilm(cosT, uTime);  // Animated thin-film BRDF
```

---

## Lighting Setup (Chris — Three-Point Glass Rig)

Glass lighting fundamentals learned the hard way:

> **Dark scenes kill glass.** The FBO captures dark content → glass refracts dark → cube appears darker than the floor. The fix was a medium grey background (0x7a8090) so the glass always has visible content to refract.

### The Three Lights

| Light | Type | Position | Purpose |
|-------|------|----------|---------|
| KEY | SpotLight (narrow, 0.07 rad) | (-4.3, 0.55, -0.4) | Main specular streak, drives dispersion rays |
| BACK | SpotLight (wide, 0.70 rad) | (3.0, 3.0, -5.5) | Illuminates scene behind glass → cube reads lighter than floor |
| RIM | SpotLight | (-1.5, 5.5, -3.0) | Catches top edges of cube |
| FILL | RectAreaLight (10×10) | (0, 6.5, 0.5) | Even floor illumination, studio softbox feel |
| AMBIENT | AmbientLight | — | 0.15 intensity, prevents crushed shadows |

**Why RectAreaLight for the fill:** PointLight overhead fill created a visible falloff ring on the floor. RectAreaLight gives even, gradient-free illumination — like a real studio softbox. Requires `RectAreaLightUniformsLib.init()` before use (commonly forgotten).

**Backlight is the most critical light for glass:** Without it, the scene behind the cube is dark and the FBO captures nothing useful. The backlight illuminates the scene the glass is refracting through — this is what makes the cube read as glass rather than a dark void.

---

## Tone Mapping

**Current: `THREE.AgXToneMapping`** (switched from ACESFilmic)

| Mapper | Character | Glass/Metal |
|--------|-----------|-------------|
| ACESFilmic | High contrast, filmic | Orange-shifts highlights ❌ |
| **AgX** | Scene-referred, neutral | No hue shift ✅ Recommended |
| Neutral | Flat, commercial | Safe fallback |

**Why AgX:** ACES orange-shifts bright glass and metal highlights — visible on the Fresnel edge glow. AgX compresses HDR values into display range while preserving hue constancy. Added in Three.js r167. Consistent with Blender 4.0 defaults.

`renderer.toneMappingExposure = 1.2` — slight lift to show the grey floor correctly.

---

## Background & Fog

**Background:** `0x7a8090` (medium grey-blue studio)  
This isn't just aesthetic — it's functional. The FBO captures this colour through the glass faces. A brighter background = brighter-reading glass.

**Scene fog:** `THREE.FogExp2(0x7a8090, 0.038)`  
Exponential falloff. Matches the background colour so the floor fades seamlessly into the background — no visible ground plane edge. Avoids the need for a sky/backdrop mesh.

**Ground fog layer:** Custom `ShaderMaterial` plane at y=0.018, 80×80 units.  
Radial gradient + gentle breathing animation (`0.88 + 0.12 * sin(t * 0.6)`).  
Normal blending (not additive) — additive would blow out the floor.

---

## Clock Hand Animation

### Design
Three clock hands using real gear ratios (1:12:144), sped up to be visually readable:

| Hand | Ray | Speed | Revolution |
|------|-----|-------|-----------|
| Hour | Violet floor ray | `2π/720` rad/s | ~12 min |
| Minute | Blue floor ray | `2π/60` rad/s | 60s |
| Second | Green floor ray | `2π/5` rad/s | 5s |

### Lengths — Golden Ratio
Stepped down from second hand (7.6) by φ=1.618:
- Second: 7.6 / Minute: 4.7 / Hour: 2.9

### Key Learning — Elevated vs Floor Rays
Early versions used elevated beams (tilted up in space) for minute/hour hands. Problem: elevated beams go **edge-on to the camera** as they sweep around — they foreshorten and nearly disappear, making it look like they don't complete a full revolution. Solution: all hands are floor rays. Only floor rays are visible for a complete 360°.

### Rotation Direction
`r.mesh.rotation.y = r.initY - t * r.omega`  
Negative sign = clockwise when viewed from above (right-hand rule in Three.js: positive Y rotation is counter-clockwise from above).

---

## Dispersion Rays (Additive Geometry)

Spectral rays are `PlaneGeometry` meshes with `AdditiveBlending` — not real light, not particles. Each ray is a `crossBeam` (two perpendicular planes) for a volumetric feel.

```javascript
// Cross-beam = two planes at 90° to each other
function crossBeam(c1, c2, w, len, op) { ... }
```

Floor rays: rotated flat (`rotation.x = Math.PI/2`) with `position.y = 0.008` (just above floor).  
All rays are children of `prismGroup` so they sweep with the clock animation.

---

## Floor

- `PlaneGeometry(500, 500)` — large enough to never show an edge at any camera angle
- `MeshStandardMaterial({ color: 0x767676 })` — 18% grey (correct photographic reference grey)
- Camera far clip = 1000 to prevent clipping

---

## Responsive Design

Canvas fills the full viewport on all devices:
```javascript
const W = window.innerWidth;
const H = window.innerHeight;
renderer.setSize(W, H);
camera.aspect = W / H;
```

Resize handler updates camera, FBO, and `uAspect` uniform.

### Pixel Budget
Hard cap at 2560px on the longest axis — performance consistent across all targets:

| Device | CSS | DPR | Physical |
|--------|-----|-----|---------|
| iPhone 15 Pro | 393×852 | 2.0 | 786×1704 |
| MacBook Pro 14" | 1512×982 | 1.69 | 2560×1663 |
| UHD 4K | 3840×2160 | 0.67 | 2560×1440 |

---

## Caustic Floor Lights

7 `PointLight`s just above the floor (y=0.06) in spectral colours (violet → red).  
These are faked caustics — not real light simulation. They work because the floor is `MeshStandardMaterial` which responds to point lights, giving the same quality "lit dot" appearance.

---

## Known Issues / Future Improvements

### Glass Quality
- [ ] **True chromatic dispersion on glass body** — current FBO refraction is good but the glass faces don't show the spectral splitting you'd see on real glass. Investigate `dispersion` property (Three.js r167+) in a hybrid approach.
- [ ] **Caustics should react to hand position** — the floor caustic dots are static, not linked to where the hands are pointing.
- [ ] **Glass interior** — no internal reflections yet. Real thick glass would show trapped bounces inside.
- [ ] **envMapIntensity** — if we switch to HDRI environment, `envMapIntensity` on a supplemental MeshPhysicalMaterial layer could add Fresnel edge quality.

### Lighting
- [ ] **Try a studio HDRI** instead of RectAreaLight for more realistic environment reflections on the glass.
- [ ] **Bloom/glow post-processing** — `UnrealBloomPass` threshold ~0.7 on a `HalfFloatType` render target would make the dispersion rays glow more physically. Need EffectComposer.
- [ ] **SSAO** — subtle ambient occlusion under the cube would ground it better than the current setup.

### Clock
- [ ] **Real time** — currently uses elapsed time. Should sync to actual system clock (`new Date()`) for a true clock.
- [ ] **Tick animation** — second hand could tick rather than sweep smoothly.
- [ ] **12 o'clock marker** — some minimal visual indicator of the clock face orientation.

### Technical
- [ ] **Cloudflare tunnel is ephemeral** — URL changes every restart. Need a persistent hosting solution (GitHub Pages or similar).
- [ ] **PWA manifest** — add `manifest.json` and `apple-mobile-web-app-capable` meta tags so "Add to Home Screen" gives true full-screen on iOS.
- [ ] **AgX tone mapping** — verify Three.js version is r167+ (we're on r169, confirmed).

---

## File Structure

```
/var/lib/cookmom-workspace/glass-cube/
├── index.html          — main scene (close camera, FOV 48°)
├── wide.html           — wide shot (camera 0,6.8,8.5 / FOV 78°) — BUILT FROM index.html
└── TECHNICAL-NOTES.md  — this file
```

**wide.html is always a derivative of index.html.** Rebuild with:
```bash
cp index.html wide.html
sed -i "s/camera.position.set(1.9, 1.5, 2.8)/camera.position.set(0, 6.8, 8.5)/" wide.html
sed -i "s/new THREE.PerspectiveCamera(48/new THREE.PerspectiveCamera(78/" wide.html
```

---

## Reference Material

- `dichroic-cube-research.md` — original FBO technique research, full shader code, IOR values
- `skills/lookdev-artist/SKILL.md` — glass material parameters, FBO vs transmission guide
- `skills/lighting-designer/SKILL.md` — glass lighting rig, RectAreaLight setup, backlight technique
- Both skills updated 2026-02-24 with state-of-the-art glass + color science sections
