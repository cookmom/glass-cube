# VFX Supervisor Brief — Glass Cube Prism Scene

## Mission
Build a standalone Three.js HTML file (`index.html`) that renders a photorealistic glass prism cube on a flat grey surface, matching the reference image.

## Reference Description
- Glass crystal cube in "diamond pose": rotated 45° on Y axis, ~35° tilted on X axis, resting on its lower vertex
- Camera: 28° above horizontal, slightly left of cube center, looking down at cube
- Single focused white light from lower-left, low elevation (~20°), like a penlight
- Three spectral dispersion ray fans emanating from cube:
  1. BLUE-VIOLET: shoots upper-left at ~45° elevation
  2. CYAN-TEAL: shoots upper-right at ~35° elevation  
  3. RED-ORANGE-YELLOW spectrum: fans out lower-right onto the floor surface
- Dark/near-black environment, zero ambient — only the cube, rays, and lit floor patches visible
- Ground: flat matte grey surface (only visible where colored light hits it)
- Background: very dark charcoal/black

## Scene Requirements

### Geometry (Bob)
- BoxGeometry(1, 1, 1) for the cube
- Cube rotation: eulerX = 0.35 rad, eulerY = Math.PI/4 (45°), eulerZ = 0.1 rad
- Ground plane: PlaneGeometry(20, 20) rotated -90° on X
- Camera: PerspectiveCamera(50°), position (1.5, 1.2, 2.8), lookAt(0, 0.3, 0)

### Glass Material (Chris)
Use THREE.MeshPhysicalMaterial with:
```javascript
{
  color: 0xffffff,
  transmission: 1.0,
  opacity: 1.0,
  metalness: 0.0,
  roughness: 0.0,
  ior: 1.6,
  thickness: 0.8,
  envMapIntensity: 1.5,
  clearcoat: 1.0,
  clearcoatRoughness: 0.0,
  transparent: true,
  side: THREE.FrontSide
}
```

### Ground Material (Chris)
```javascript
{
  color: 0x404040,  // medium grey
  roughness: 0.9,
  metalness: 0.0
}
```

### Lighting (Chris)
1. Main key light: SpotLight(0xffffff, 8.0), position(-3, 0.5, -1.5), target at cube, penumbra 0.3, angle 0.15 — this is the "penlight"
2. Rim light: SpotLight(0x8888ff, 1.5), position(2, 3, -2) — subtle cool rim
3. NO ambient light (AmbientLight intensity 0) — pure darkness except key

### Dispersion Ray System (Chris + Shader)
This is the hero element. Build custom ray beams using PlaneGeometry + ShaderMaterial with AdditiveBlending.

Create a `createDispersionRay(color1, color2, direction, length, width, opacity)` function.

Each ray is a flat plane (PlaneGeometry(width, length)) with a custom shader:
```glsl
// Vertex shader
varying vec2 vUv;
void main() {
  vUv = uv;
  gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}

// Fragment shader — rainbow gradient ray
uniform vec3 color1;
uniform vec3 color2;
uniform float opacity;
varying vec2 vUv;

void main() {
  // Gradient along length (0=near cube, 1=far end)
  vec3 col = mix(color1, color2, vUv.y);
  // Gaussian falloff across width
  float alpha = exp(-pow((vUv.x - 0.5) * 4.0, 2.0));
  // Fade near tip and far end
  alpha *= smoothstep(0.0, 0.15, vUv.y) * smoothstep(1.0, 0.6, vUv.y);
  gl_FragColor = vec4(col * opacity, alpha * opacity);
}
```

Ray configuration (position rays from cube center, pivot at near end):
1. **Blue-violet ray**: colors #3333ff → #8800cc, length 3.0, width 0.4, rotated to shoot upper-left (~315° azimuth, 40° up)
2. **Cyan ray**: colors #00ffee → #0088ff, length 2.5, width 0.35, rotated to shoot upper-right (~30° azimuth, 35° up)
3. **Red-orange-yellow fan** (3 sub-rays on floor surface):
   - Red: #ff2200 → #ff0000, length 2.8, width 0.3, on ground, azimuth ~150°
   - Orange: #ff8800 → #ff4400, length 2.5, width 0.3, on ground, azimuth ~135°  
   - Yellow: #ffee00 → #ffaa00, length 2.2, width 0.28, on ground, azimuth ~120°
   - These 3 rays lay FLAT on the ground (rotation so they project onto XZ plane)

All rays use THREE.AdditiveBlending, depthWrite: false.

### Colored Floor Caustics
Add soft colored PointLights hovering just above the floor at the ray landing spots:
- Blue: PointLight(0x2200ff, 2.0, 3.0) at (-1.5, 0.05, -0.5)
- Cyan: PointLight(0x00eeff, 1.5, 2.5) at (1.2, 0.05, -0.8)
- Red: PointLight(0xff3300, 1.5, 2.0) at (0.8, 0.05, 1.0)
- Orange: PointLight(0xff8800, 1.2, 2.0) at (0.5, 0.05, 1.2)
- Yellow: PointLight(0xffcc00, 1.0, 1.8) at (0.2, 0.05, 1.3)

### Renderer Config
```javascript
renderer.setPixelRatio(window.devicePixelRatio)
renderer.setSize(800, 800)
renderer.toneMapping = THREE.ACESFilmicToneMapping
renderer.toneMappingExposure = 1.2
renderer.shadowMap.enabled = true
renderer.shadowMap.type = THREE.PCFSoftShadowMap
```

### Environment Map
Use PMREMGenerator with a simple dark environment (RoomEnvironment or just a dark scene) for the glass reflections.

### Three.js Import
Use importmap with three.js from CDN:
```html
<script type="importmap">
{
  "imports": {
    "three": "https://cdn.jsdelivr.net/npm/three@0.169.0/build/three.module.js",
    "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.169.0/examples/jsm/"
  }
}
</script>
```

## Final Output
- Single `index.html` file
- Canvas fills full viewport (800x800 centered or full page)
- Black background
- Subtle slow rotation animation (cube rotates slowly on Y axis, rays follow)
- Quality: PHOTOREALISTIC lookdev — VFX supervisor will reject if it looks cheap

## Completion
When done, run:
openclaw system event --text "Glass cube scene built — ready for lookdev review" --mode now
