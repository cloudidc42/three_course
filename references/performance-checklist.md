# Three.js Performance Checklist

## Renderer

```
[ ] setPixelRatio(Math.min(devicePixelRatio, 2))
[ ] ใช้ powerPreference: 'high-performance' สำหรับ games
[ ] ปิด antialias บน mobile (devicePixelRatio >= 2 ไม่จำเป็น)
[ ] ปิด shadowMap.enabled ถ้าไม่ใช้ shadow
[ ] ใช้ shadowMap.type = THREE.PCFSoftShadowMap (ดีที่สุด)
[ ] ลด renderer.info.autoReset = false ถ้า track เอง
[ ] ปิด stencil: false, depth: false ถ้าไม่ใช้ (WebGLRenderer option)
```

---

## Geometry

```
[ ] ใช้ BufferGeometry (ไม่ใช้ Geometry เก่า)
[ ] merge static meshes ด้วย BufferGeometryUtils.mergeGeometries()
[ ] ใช้ InstancedMesh สำหรับ objects ซ้ำกัน (>= 20 instances)
[ ] LOD (Level of Detail) สำหรับ objects ไกล
[ ] dispose geometry เมื่อไม่ใช้แล้ว: geo.dispose()
[ ] ลด vertex count: ใช้ Draco/Meshopt compression
[ ] ไม่ใช้ SphereGeometry ที่ segments สูงเกินไป (32x32 พอ)
[ ] ใช้ PlaneGeometry แทน BoxGeometry ถ้า flat
[ ] index geometry ด้วย setIndex() เพื่อ reuse vertices
```

---

## Materials

```
[ ] ใช้ MeshBasicMaterial สำหรับ unlit objects
[ ] ใช้ MeshLambertMaterial แทน MeshStandardMaterial ถ้า PBR ไม่จำเป็น
[ ] share materials ระหว่าง meshes ที่เหมือนกัน
[ ] dispose materials: mat.dispose()
[ ] dispose textures: tex.dispose()
[ ] ลด material.side = THREE.FrontSide (default)
[ ] ปิด material.transparent = false ถ้าไม่ต้องการ
[ ] ตั้ง material.depthWrite = false สำหรับ transparent objects
[ ] ใช้ alphaTest แทน transparent เมื่อเป็นไปได้
[ ] group materials ที่เหมือนกันเพื่อลด draw calls
```

---

## Textures

```
[ ] ใช้ขนาด power-of-two (256, 512, 1024, 2048)
[ ] compress textures: WebP, AVIF, KTX2/Basis
[ ] ตั้ง texture.generateMipmaps = false สำหรับ UI textures
[ ] ใช้ texture atlas รวม textures เล็กๆ
[ ] ลด texture.anisotropy (4 พอ, max 16)
[ ] ตั้ง texture.minFilter = THREE.LinearFilter ถ้าไม่ต้องการ mipmap
[ ] ตั้ง texture.magFilter = THREE.LinearFilter
[ ] dispose unused textures
[ ] ใช้ ColorSpace ที่ถูกต้อง: sRGBColorSpace สำหรับ albedo
[ ] ขนาด texture: 2048x2048 max สำหรับ mobile
```

---

## Lights

```
[ ] ลด number of lights (คำนวณ per light)
[ ] ใช้ AmbientLight + DirectionalLight แทน multiple PointLights
[ ] ปิด castShadow บน lights ที่ไม่จำเป็น
[ ] ลด shadow.mapSize (512 หรือ 1024 สำหรับ mobile)
[ ] ใช้ HemisphereLight แทน AmbientLight + DirectionalLight
[ ] bake lighting ลง texture สำหรับ static scenes
[ ] ตั้ง light.distance สำหรับ PointLight และ SpotLight
[ ] ใช้ rectAreaLight น้อยที่สุด (มันแพง)
```

---

## Draw Calls

```
[ ] ลด draw calls: target < 100 สำหรับ mobile, < 500 desktop
[ ] merge static geometry ด้วย mergeGeometries()
[ ] ใช้ InstancedMesh สำหรับ repeated geometry
[ ] group objects ด้วย material เดียวกัน
[ ] ใช้ multi-material mesh แทนหลาย objects
[ ] frustum culling: เปิด (default): mesh.frustumCulled = true
[ ] custom frustum culling สำหรับ large objects
[ ] ใช้ LOD เพื่อ skip rendering ของ distant objects
```

---

## Render Loop

```javascript
// ✅ Render เมื่อมีการเปลี่ยนแปลงเท่านั้น (static scenes)
let needsRender = true;

controls.addEventListener('change', () => { needsRender = true; });

function animate() {
  requestAnimationFrame(animate);
  if (!needsRender) return;
  needsRender = false;
  renderer.render(scene, camera);
}

// ✅ ลด pixel ratio เมื่อ FPS ต่ำ
let currentDpr = window.devicePixelRatio;
function adaptiveResolution(fps) {
  if (fps < 30 && currentDpr > 1) {
    currentDpr = Math.max(1, currentDpr - 0.25);
    renderer.setPixelRatio(currentDpr);
  } else if (fps > 55 && currentDpr < window.devicePixelRatio) {
    currentDpr = Math.min(window.devicePixelRatio, currentDpr + 0.25);
    renderer.setPixelRatio(currentDpr);
  }
}
```

---

## Memory Management

```javascript
// Dispose เมื่อ remove object
function disposeObject(obj) {
  obj.traverse(child => {
    if (child.geometry) {
      child.geometry.dispose();
    }
    if (child.material) {
      const mats = Array.isArray(child.material)
        ? child.material : [child.material];
      mats.forEach(mat => {
        // Dispose all textures
        Object.values(mat).forEach(v => {
          if (v && v.isTexture) v.dispose();
        });
        mat.dispose();
      });
    }
  });
}

// Track memory leaks
function checkMemory() {
  const info = renderer.info;
  console.log('Geometries:', info.memory.geometries);
  console.log('Textures:', info.memory.textures);
  console.log('Draw calls:', info.render.calls);
}
```

---

## Raycasting

```javascript
// ✅ จำกัด objects ที่ raycast
const hitObjects = importantObjects; // เฉพาะ objects ที่ต้องการ
raycaster.intersectObjects(hitObjects, false); // false = ไม่ recursive

// ✅ ใช้ BVH for complex meshes
// npm install three-mesh-bvh
import { MeshBVH, acceleratedRaycast } from 'three-mesh-bvh';
THREE.Mesh.prototype.raycast = acceleratedRaycast;
geo.boundsTree = new MeshBVH(geo);

// ✅ Throttle raycasting
let lastRaycast = 0;
function onMouseMove() {
  const now = performance.now();
  if (now - lastRaycast < 50) return; // Max 20x/sec
  lastRaycast = now;
  // do raycast
}
```

---

## Mobile-Specific

```javascript
// ✅ ตรวจสอบ mobile
const isMobile = /iPhone|iPad|Android/i.test(navigator.userAgent);

const renderer = new THREE.WebGLRenderer({
  antialias: !isMobile,  // ปิด AA บน mobile
  powerPreference: isMobile ? 'low-power' : 'high-performance'
});

// ✅ ลด quality บน mobile
if (isMobile) {
  renderer.setPixelRatio(1);
  shadowMap.mapSize.set(512, 512);
  // ลด particle count
  // ใช้ simpler materials
}

// ✅ ตรวจสอบ WebGL capabilities
const caps = renderer.capabilities;
if (!caps.isWebGL2) {
  // Fallback for WebGL1
}
if (caps.maxTextures < 16) {
  // ลด texture usage
}
```

---

## Performance Targets

```
Desktop High-end:  60 fps, <500 draw calls, <200MB VRAM
Desktop Mid-range: 60 fps, <200 draw calls, <100MB VRAM
Mobile High-end:   60 fps, <100 draw calls, <50MB VRAM
Mobile Mid-range:  30 fps, <50  draw calls, <30MB VRAM

Frame budget (16.67ms @ 60fps):
  CPU:  5-8ms  (JS + Three.js overhead)
  GPU:  8-10ms (rendering)
  Overhead: 1-2ms

Tools for profiling:
  - renderer.info (draw calls, triangles, memory)
  - Chrome DevTools Performance tab
  - Chrome GPU tab (chrome://gpu)
  - Spector.js (WebGL debugger extension)
  - stats.js (FPS counter)
```

---

## Asset Loading Performance

```javascript
// ✅ Preload critical assets
const preloadPromises = [
  loadTexture('albedo.webp'),
  loadTexture('normal.webp'),
  loadGLTF('main-model.glb')
];
await Promise.all(preloadPromises);

// ✅ Lazy load non-critical assets
async function lazyLoadSection(section) {
  if (!section.loaded) {
    section.texture = await loadTexture(section.texturePath);
    section.loaded = true;
  }
}

// ✅ Reuse loaded assets
const textureCache = new Map();
async function loadTextureOnce(url) {
  if (textureCache.has(url)) return textureCache.get(url);
  const tex = await new THREE.TextureLoader().loadAsync(url);
  textureCache.set(url, tex);
  return tex;
}

// ✅ Use compressed formats
// KTX2/Basis (GPU-native compression)
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js';
const ktx2Loader = new KTX2Loader();
ktx2Loader.setTranscoderPath('basis/');
ktx2Loader.detectSupport(renderer);
```
