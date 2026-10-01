# Three.js Resources & References

## Official Documentation

- **Three.js Docs**: https://threejs.org/docs/
- **Three.js Examples**: https://threejs.org/examples/
- **Three.js GitHub**: https://github.com/mrdoob/three.js
- **Three.js r3f (React Three Fiber)**: https://docs.pmnd.rs/react-three-fiber
- **Three.js Changelog**: https://github.com/mrdoob/three.js/blob/dev/CHANGELOG.md

---

## Learning Resources

### Thai / ภาษาไทย
- คอร์สนี้ (parts 001-092)

### English Courses
- **Bruno Simon's Three.js Journey**: https://threejs-journey.com (paid, best course)
- **Discover three.js**: https://discoverthreejs.com (free e-book)
- **The Book of Shaders**: https://thebookofshaders.com (GLSL focused)
- **Inigo Quilez**: https://iquilezles.org/articles/ (advanced math/shaders)

### YouTube Channels
- **Yuri Artiukh** (@akella): Creative coding, shaders
- **Suboptimal Engineer**: Three.js tutorials
- **Visionary 3D**: Three.js for beginners

### Blogs & Articles
- **codrops**: https://tympanus.net/codrops/tag/threejs/
- **pmnd.rs**: https://pmnd.rs (React Three Fiber ecosystem)
- **glsl.app**: Online GLSL editor

---

## Free 3D Assets

### Models (GLTF/GLB)
- **Khronos Sample Models**: https://github.com/KhronosGroup/glTF-Sample-Models
- **Sketchfab** (free tier): https://sketchfab.com/features/free-3d-models
- **Poly Haven**: https://polyhaven.com/models (CC0)
- **Quaternius**: https://quaternius.com (CC0 game assets)
- **Kenney**: https://kenney.nl/assets (CC0)
- **Google Poly Archive**: https://poly.pizza

### Textures
- **Poly Haven Textures**: https://polyhaven.com/textures (CC0, 4K)
- **ambientCG**: https://ambientcg.com (CC0, PBR)
- **Texture.ninja**: https://texture.ninja (various)
- **3dtextures.me**: https://3dtextures.me (CC0)
- **Freepbr**: https://freepbr.com

### HDRI
- **Poly Haven HDRI**: https://polyhaven.com/hdris (CC0, up to 16K)
- **HDRI Haven** (now Poly Haven): same URL
- **iHDRI**: https://www.ihdri.com

### Fonts (for 3D Text)
- **Google Fonts**: https://fonts.google.com
- **FontAwesome**: for icon fonts
- **Facetype.js**: https://gero3.github.io/facetype.js/ (convert TTF to JSON for Three.js)

---

## Useful Libraries

### Physics
- **Cannon.js**: https://schteppe.github.io/cannon.js/ (classic)
- **Cannon-es**: https://github.com/pmndrs/cannon-es (maintained fork)
- **Rapier.js**: https://rapier.rs (Rust/WASM, very fast)
- **Ammo.js**: https://github.com/kripken/ammo.js (Bullet physics port)

### Post-Processing
- **pmndrs/postprocessing**: https://github.com/pmndrs/postprocessing
- **Three.js PostProcessing**: built-in examples/jsm/postprocessing/

### Animation
- **GSAP**: https://greensock.com/gsap/ (industry standard)
- **Framer Motion**: https://www.framer.com/motion/ (React)
- **anime.js**: https://animejs.com (lightweight)

### UI & Controls
- **lil-gui**: https://lil-gui.georgealways.com (debug GUI)
- **dat.GUI**: https://github.com/dataarts/dat.gui (classic)
- **tweakpane**: https://tweakpane.github.io (modern GUI)

### Utilities
- **three-mesh-bvh**: https://github.com/gkjohnson/three-mesh-bvh (fast raycasting)
- **three-stdlib**: https://github.com/pmndrs/three-stdlib (maintained addons)
- **three-custom-shader-material**: CSM for extending built-in materials
- **troika-three-text**: https://github.com/protectwise/troika (3D text)

### Data Visualization
- **D3.js**: https://d3js.org
- **Observable Plot**: https://observablehq.com/@observablehq/plot

### Shader Tools
- **glslify**: https://github.com/glslify/glslify (GLSL modules)
- **vite-plugin-glsl**: Vite plugin for GLSL imports
- **lygia**: https://lygia.xyz (GLSL shader library)

---

## Online Editors & Playgrounds

- **Three.js Editor**: https://threejs.org/editor/ (visual editor)
- **Shadertoy**: https://www.shadertoy.com (GLSL shaders)
- **GLSL Sandbox**: http://glslsandbox.com
- **glsl.app**: https://glsl.app (modern GLSL editor)
- **CodeSandbox**: https://codesandbox.io (full project)
- **StackBlitz**: https://stackblitz.com

---

## Community

- **Three.js Forum**: https://discourse.threejs.org (official)
- **Three.js Discord**: https://discord.gg/threejs
- **r/threejs**: https://reddit.com/r/threejs
- **Twitter/X**: #threejs

---

## Tools

### Performance
- **Spector.js** (browser extension): WebGL frame debugger
- **Chrome DevTools**: Performance, Memory, GPU
- **stats.js**: https://github.com/mrdoob/stats.js (FPS counter)
- **Lighthouse**: Performance audit

### Asset Processing
- **gltf-transform**: https://gltf-transform.donmccurdy.com/cli.html
- **glTF Validator**: https://github.khronos.org/glTF-Validator/
- **Squoosh**: https://squoosh.app (texture optimization)
- **Blender**: https://www.blender.org (3D modeling + exporter)

### Build Tools
- **Vite**: https://vitejs.dev (recommended)
- **Webpack**: https://webpack.js.org
- **Rollup**: https://rollupjs.org

---

## Texture Sources by Category

| Category | Site | License | Quality |
|----------|------|---------|---------|
| PBR Materials | polyhaven.com | CC0 | 4K-8K |
| PBR Materials | ambientcg.com | CC0 | 2K-4K |
| HDRI | polyhaven.com | CC0 | 1K-16K |
| Game textures | kenney.nl | CC0 | 1K |
| General | texture.ninja | Various | 2K |

---

## Three.js Version History (Key Features)

| Version | Key Feature |
|---------|-------------|
| r152+ | WebGPURenderer (beta) |
| r155+ | TSL (Three.js Shading Language) |
| r149+ | PMREMGenerator improvements |
| r145+ | WebGL2 as default |
| r140+ | MeshPhysicalMaterial improvements |
| r135+ | KTX2 compressed texture support |
| r128+ | WebXR improvements |
| r122+ | WebGPU research renderer |

---

## Recommended Setup (2025)

```bash
# Modern Three.js project
npm create vite@latest my-3d-app -- --template vanilla
cd my-3d-app
npm install three
npm install -D vite-plugin-glsl

# Optional but useful
npm install @pmndrs/postprocessing  # Post-processing
npm install gsap                    # Animation
npm install lil-gui                 # Debug GUI
npm install three-mesh-bvh          # Fast raycasting
npm install cannon-es               # Physics
```

```javascript
// Minimal working setup
import * as THREE from 'three';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, innerWidth/innerHeight, 0.1, 100);
const renderer = new THREE.WebGLRenderer({ antialias: true });

renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
renderer.outputColorSpace = THREE.SRGBColorSpace;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
document.body.appendChild(renderer.domElement);

camera.position.z = 5;

const mesh = new THREE.Mesh(
  new THREE.BoxGeometry(),
  new THREE.MeshStandardMaterial({ color: 0x4488ff })
);
scene.add(mesh);
scene.add(new THREE.AmbientLight(0xffffff, 0.5));
scene.add(Object.assign(new THREE.DirectionalLight(0xffffff, 1), { position: { x: 5, y: 10, z: 5 } }));

window.addEventListener('resize', () => {
  camera.aspect = innerWidth / innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(innerWidth, innerHeight);
});

(function animate() {
  requestAnimationFrame(animate);
  mesh.rotation.y += 0.01;
  renderer.render(scene, camera);
})();
```
