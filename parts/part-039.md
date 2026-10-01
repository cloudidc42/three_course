# Part 39: Environment Maps & IBL (Image-Based Lighting) ใน Three.js

## บทนำ

Environment Maps และ Image-Based Lighting (IBL) เป็นเทคนิคที่ทำให้ scene ดูสมจริงอย่างมาก โดยใช้ภาพ 360° จากสภาพแวดล้อมจริงเป็น source ของแสง สะท้อน และ ambient lighting

แนวคิดหลัก:
- **Environment Map** - ภาพ 360° ที่ใช้สะท้อนและ lighting
- **IBL** - การใช้ environment map เป็น light source จริงๆ (ผ่าน PMREM)
- **PMREM** - Pre-filtered Mipmapped Radiance Environment Map

## 1. CubeTextureLoader (6 faces)

```javascript
import * as THREE from 'three';

// ===== Cubemap: 6 ภาพสำหรับแต่ละด้าน =====
// ลำดับ: +X, -X, +Y, -Y, +Z, -Z (right, left, top, bottom, front, back)

const loader = new THREE.CubeTextureLoader();
loader.setPath('/textures/cubemap/'); // base path

const cubeTexture = loader.load([
  'px.jpg', // +X (right)
  'nx.jpg', // -X (left)
  'py.jpg', // +Y (top)
  'ny.jpg', // -Y (bottom)
  'pz.jpg', // +Z (front)
  'nz.jpg'  // -Z (back)
]);

// ใช้เป็น background
scene.background = cubeTexture;

// ใช้เป็น environment map (reflections + IBL)
scene.environment = cubeTexture;

// ใช้กับ material โดยตรง
const material = new THREE.MeshStandardMaterial({
  envMap: cubeTexture,
  envMapIntensity: 1.0,
  roughness: 0.0,    // smooth = more reflective
  metalness: 1.0     // metallic = more reflective
});
```

## 2. PMREMGenerator

PMREMGenerator แปลง environment map ให้เหมาะสำหรับ PBR shading

```javascript
import { PMREMGenerator } from 'three';

// สร้าง PMREMGenerator
const pmremGenerator = new THREE.PMREMGenerator(renderer);
pmremGenerator.compileEquirectangularShader();

// ===== จาก CubeTexture =====
const envTexture = pmremGenerator.fromCubemap(cubeTexture).texture;
scene.environment = envTexture;

// ===== จาก Equirectangular Texture =====
const textureLoader = new THREE.TextureLoader();
const equiTexture = textureLoader.load('/textures/env.jpg', (texture) => {
  const envMap = pmremGenerator.fromEquirectangular(texture).texture;
  scene.environment = envMap;
  scene.background = envMap;
  
  texture.dispose();
  pmremGenerator.dispose();
});

// ===== จาก Scene (dynamic) =====
const renderTarget = pmremGenerator.fromScene(
  new THREE.RoomEnvironment(), // built-in room preset
  0.04 // blur amount
);
scene.environment = renderTarget.texture;
```

## 3. EquirectangularReflectionMapping

```javascript
// Equirectangular projection: ภาพ 2D ที่ map เป็น sphere 360°
// ต่างจาก CubeMap ที่ใช้ 6 ภาพ

const loader = new THREE.TextureLoader();
const texture = loader.load('/textures/environment.jpg');
texture.mapping = THREE.EquirectangularReflectionMapping;

// ใช้กับ material
const material = new THREE.MeshStandardMaterial({
  envMap: texture,
  envMapIntensity: 1.5
});

// ใช้เป็น background
scene.background = texture;
```

## 4. RGBELoader สำหรับ HDR Environment

```javascript
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js';

const rgbeLoader = new RGBELoader();

rgbeLoader.load('/textures/studio_small_08_1k.hdr', (hdrTexture) => {
  hdrTexture.mapping = THREE.EquirectangularReflectionMapping;
  
  // Set as environment AND background
  scene.environment = hdrTexture;
  scene.background = hdrTexture;
  
  // Optionally blur the background
  scene.backgroundBlurriness = 0.1;
  scene.backgroundIntensity = 1.0;
  
  console.log('HDR loaded:', hdrTexture.image.width, 'x', hdrTexture.image.height);
});
```

## 5. scene.environment vs scene.background

```javascript
// scene.environment: ใช้สำหรับ PBR lighting (reflections, ambient)
// scene.background: ใช้สำหรับ background ที่เห็น

// สามารถใช้ต่างกันได้:
const envMap = loadHDR('env_lighting.hdr');
const bgMap = loadHDR('background_sky.hdr');

scene.environment = envMap;  // lighting จากภาพนี้
scene.background = bgMap;    // เห็นภาพนี้เป็น background

// หรือใช้สีเป็น background แต่ใช้ HDR เป็น lighting
scene.background = new THREE.Color(0x87ceeb); // sky blue
scene.environment = envMap; // HDR lighting

// เบลอ background
scene.backgroundBlurriness = 0.5; // 0 = sharp, 1 = fully blurred

// Intensity
scene.backgroundIntensity = 1.5; // ทำให้ background สว่างขึ้น
```

## 6. envMapIntensity

```javascript
// ควบคุมความแรงของ reflection/IBL บน material
const material = new THREE.MeshStandardMaterial({
  envMapIntensity: 1.0  // default
});

// เพิ่ม intensity = สว่างขึ้น มีแสงสะท้อนมากขึ้น
material.envMapIntensity = 2.0;

// ลด intensity = มืดลง
material.envMapIntensity = 0.3;

// ทำกับทุก materials ใน scene
scene.traverse(child => {
  if (child.isMesh && child.material) {
    if (Array.isArray(child.material)) {
      child.material.forEach(mat => {
        mat.envMapIntensity = globalIntensity;
      });
    } else {
      child.material.envMapIntensity = globalIntensity;
    }
  }
});
```

## 7. Demo 1: HDR Environment Scene

```html
<!DOCTYPE html>
<html>
<head>
  <title>HDR Environment Demo</title>
  <style>
    body { margin: 0; background: #000; }
    canvas { display: block; }
    #controls {
      position: fixed;
      top: 20px;
      right: 20px;
      background: rgba(0,0,0,0.7);
      color: white;
      padding: 20px;
      border-radius: 10px;
      font-family: Arial, sans-serif;
      font-size: 14px;
      min-width: 220px;
    }
    .control-row { margin: 10px 0; }
    label { display: block; margin-bottom: 3px; }
    input[type="range"] { width: 180px; }
    select { width: 185px; padding: 4px; background: #333; color: #fff; border: 1px solid #555; }
  </style>
</head>
<body>
<div id="controls">
  <div style="font-size:16px;font-weight:bold;margin-bottom:12px">Environment Controls</div>
  
  <div class="control-row">
    <label>Roughness: <span id="roughnessVal">0.0</span></label>
    <input type="range" id="roughness" min="0" max="100" value="0">
  </div>
  
  <div class="control-row">
    <label>Metalness: <span id="metalnessVal">1.0</span></label>
    <input type="range" id="metalness" min="0" max="100" value="100">
  </div>
  
  <div class="control-row">
    <label>Env Intensity: <span id="envIntVal">1.0</span></label>
    <input type="range" id="envInt" min="0" max="300" value="100">
  </div>
  
  <div class="control-row">
    <label>Background Blur: <span id="blurVal">0.0</span></label>
    <input type="range" id="blur" min="0" max="100" value="0">
  </div>
  
  <div class="control-row">
    <label>Object:</label>
    <select id="objectType">
      <option value="sphere">Sphere</option>
      <option value="box">Box</option>
      <option value="torus">Torus</option>
      <option value="knot">Torus Knot</option>
    </select>
  </div>
  
  <div class="control-row">
    <label>Color:</label>
    <input type="color" id="color" value="#ffffff" style="width:40px">
  </div>
</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';
import { RGBELoader } from 'https://unpkg.com/three@0.158.0/examples/jsm/loaders/RGBELoader.js';
import { RoomEnvironment } from 'https://unpkg.com/three@0.158.0/examples/jsm/environments/RoomEnvironment.js';

// ===== SETUP =====
const scene = new THREE.Scene();

const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 100);
camera.position.set(0, 1, 5);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(devicePixelRatio);
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.0;
renderer.outputColorSpace = THREE.SRGBColorSpace;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.target.set(0, 0, 0);

// ===== PMREM GENERATOR =====
const pmremGenerator = new THREE.PMREMGenerator(renderer);
pmremGenerator.compileEquirectangularShader();

// ===== SETUP ENVIRONMENT WITH ROOM ENVIRONMENT (FALLBACK) =====
// ใช้ RoomEnvironment เมื่อไม่มี HDR file
const roomEnv = pmremGenerator.fromScene(new RoomEnvironment(), 0.04);
scene.environment = roomEnv.texture;
scene.background = roomEnv.texture;

// ===== ลอง load HDR (ถ้ามี) =====
// สร้าง procedural HDR-like environment แทน
function createProceduralEnvironment() {
  // สร้าง gradient sky texture
  const size = 512;
  const data = new Uint8Array(size * size * 4);
  
  for (let y = 0; y < size; y++) {
    for (let x = 0; x < size; x++) {
      const i = (y * size + x) * 4;
      const t = y / size;
      
      // Sky gradient: blue to light blue
      const r = Math.floor(30 + t * 150);
      const g = Math.floor(60 + t * 160);
      const b = Math.floor(120 + t * 135);
      
      // Add sun glow in corner
      const sunX = size * 0.7, sunY = size * 0.3;
      const sunDist = Math.sqrt((x - sunX) ** 2 + (y - sunY) ** 2);
      const sunGlow = Math.max(0, 1 - sunDist / 100);
      
      data[i] = Math.min(255, r + sunGlow * 200);
      data[i + 1] = Math.min(255, g + sunGlow * 150);
      data[i + 2] = Math.min(255, b + sunGlow * 50);
      data[i + 3] = 255;
    }
  }
  
  const texture = new THREE.DataTexture(data, size, size);
  texture.mapping = THREE.EquirectangularReflectionMapping;
  texture.needsUpdate = true;
  
  return texture;
}

// ===== MAIN OBJECT =====
let currentMesh = null;
const material = new THREE.MeshStandardMaterial({
  color: 0xffffff,
  roughness: 0.0,
  metalness: 1.0,
  envMapIntensity: 1.0
});

function createObject(type) {
  if (currentMesh) scene.remove(currentMesh);
  
  let geo;
  switch (type) {
    case 'sphere':
      geo = new THREE.SphereGeometry(1.2, 64, 64);
      break;
    case 'box':
      geo = new THREE.BoxGeometry(1.8, 1.8, 1.8, 4, 4, 4);
      break;
    case 'torus':
      geo = new THREE.TorusGeometry(1.0, 0.4, 32, 64);
      break;
    case 'knot':
      geo = new THREE.TorusKnotGeometry(0.8, 0.3, 128, 32);
      break;
    default:
      geo = new THREE.SphereGeometry(1.2, 64, 64);
  }
  
  currentMesh = new THREE.Mesh(geo, material);
  scene.add(currentMesh);
}

createObject('sphere');

// ===== ADD SOME OBJECTS AROUND =====
const ballPositions = [
  [-3, 0, 0], [3, 0, 0], [0, 2, 0], [0, -2, 0],
  [-2, 1.5, -2], [2, -1.5, -2]
];

ballPositions.forEach(([x, y, z], i) => {
  const mat = new THREE.MeshStandardMaterial({
    color: new THREE.Color().setHSL(i / ballPositions.length, 0.8, 0.5),
    roughness: i * 0.15,
    metalness: 1.0 - i * 0.1,
    envMapIntensity: 1.0
  });
  
  const ball = new THREE.Mesh(
    new THREE.SphereGeometry(0.4, 32, 32),
    mat
  );
  ball.position.set(x, y, z);
  scene.add(ball);
});

// ===== CONTROLS =====
document.getElementById('roughness').addEventListener('input', (e) => {
  const val = e.target.value / 100;
  material.roughness = val;
  document.getElementById('roughnessVal').textContent = val.toFixed(2);
});

document.getElementById('metalness').addEventListener('input', (e) => {
  const val = e.target.value / 100;
  material.metalness = val;
  document.getElementById('metalnessVal').textContent = val.toFixed(2);
});

document.getElementById('envInt').addEventListener('input', (e) => {
  const val = e.target.value / 100;
  material.envMapIntensity = val;
  document.getElementById('envIntVal').textContent = val.toFixed(2);
  
  // Apply to all materials
  scene.traverse(child => {
    if (child.isMesh && child.material !== material) {
      child.material.envMapIntensity = val;
    }
  });
});

document.getElementById('blur').addEventListener('input', (e) => {
  const val = e.target.value / 100;
  scene.backgroundBlurriness = val;
  document.getElementById('blurVal').textContent = val.toFixed(2);
});

document.getElementById('objectType').addEventListener('change', (e) => {
  createObject(e.target.value);
});

document.getElementById('color').addEventListener('input', (e) => {
  material.color.set(e.target.value);
});

// ===== ANIMATION =====
const clock = new THREE.Clock();

function animate() {
  requestAnimationFrame(animate);
  
  const elapsed = clock.getElapsedTime();
  
  if (currentMesh) {
    currentMesh.rotation.y = elapsed * 0.3;
    currentMesh.rotation.x = Math.sin(elapsed * 0.2) * 0.1;
  }
  
  controls.update();
  renderer.render(scene, camera);
}

animate();

window.addEventListener('resize', () => {
  camera.aspect = innerWidth / innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(innerWidth, innerHeight);
});
</script>
</body>
</html>
```

## 8. Cube Render Targets สำหรับ Dynamic Environment

```javascript
// ===== Dynamic Reflections ด้วย CubeCamera =====
// CubeCamera render scene จาก 6 ทิศทางเพื่อสร้าง real-time reflection

import { CubeCamera, WebGLCubeRenderTarget } from 'three';

// สร้าง render target
const cubeRenderTarget = new THREE.WebGLCubeRenderTarget(256, {
  format: THREE.RGBFormat,
  generateMipmaps: true,
  minFilter: THREE.LinearMipmapLinearFilter
});

// สร้าง CubeCamera
const cubeCamera = new THREE.CubeCamera(0.1, 100, cubeRenderTarget);
scene.add(cubeCamera);

// Material ที่ใช้ dynamic reflection
const chromeMaterial = new THREE.MeshStandardMaterial({
  envMap: cubeRenderTarget.texture,
  envMapIntensity: 1.0,
  roughness: 0.0,
  metalness: 1.0
});

const reflectiveSphere = new THREE.Mesh(
  new THREE.SphereGeometry(1, 64, 64),
  chromeMaterial
);
scene.add(reflectiveSphere);

// ===== Update ใน animation loop =====
function animate() {
  requestAnimationFrame(animate);
  
  // ซ่อน sphere ขณะ render environment (ป้องกัน self-reflection)
  reflectiveSphere.visible = false;
  
  // Update position ของ CubeCamera ให้อยู่ที่ตำแหน่งเดียวกับ sphere
  cubeCamera.position.copy(reflectiveSphere.position);
  
  // Render environment ทั้ง 6 ด้าน
  cubeCamera.update(renderer, scene);
  
  // แสดง sphere อีกครั้ง
  reflectiveSphere.visible = true;
  
  renderer.render(scene, camera);
}
```

## 9. Demo 2: Dynamic Reflection Demo

```html
<!DOCTYPE html>
<html>
<head>
  <title>Dynamic Reflections</title>
  <style>
    body { margin: 0; background: #0a0a20; }
    canvas { display: block; }
    #info {
      position: fixed;
      top: 20px;
      left: 50%;
      transform: translateX(-50%);
      color: #8af;
      font-family: Arial, sans-serif;
      text-align: center;
    }
  </style>
</head>
<body>
<div id="info">Chrome Ball - Real-Time Reflections<br>
<small>วัตถุรอบๆ สะท้อนลงบนลูกบอล</small></div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';
import { RoomEnvironment } from 'https://unpkg.com/three@0.158.0/examples/jsm/environments/RoomEnvironment.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 100);
camera.position.set(0, 3, 8);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(devicePixelRatio);
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.outputColorSpace = THREE.SRGBColorSpace;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

// Setup base environment
const pmrem = new THREE.PMREMGenerator(renderer);
const envTexture = pmrem.fromScene(new RoomEnvironment(), 0.04).texture;
scene.environment = envTexture;
scene.background = new THREE.Color(0x0a0a20);

// Lighting
scene.add(new THREE.AmbientLight(0x334466, 1));
const dirLight = new THREE.DirectionalLight(0xffffff, 2);
dirLight.position.set(5, 10, 5);
scene.add(dirLight);

// ===== DYNAMIC CUBE CAMERA =====
const cubeRenderTarget = new THREE.WebGLCubeRenderTarget(256, {
  generateMipmaps: true,
  minFilter: THREE.LinearMipmapLinearFilter
});

const cubeCamera = new THREE.CubeCamera(0.1, 100, cubeRenderTarget);
scene.add(cubeCamera);

// Chrome sphere with dynamic reflection
const chromeMat = new THREE.MeshStandardMaterial({
  envMap: cubeRenderTarget.texture,
  roughness: 0.0,
  metalness: 1.0,
  color: 0xffffff
});

const chromeSphere = new THREE.Mesh(
  new THREE.SphereGeometry(1.2, 64, 64),
  chromeMat
);
scene.add(chromeSphere);

// ===== COLORFUL ORBITING OBJECTS =====
const orbitObjects = [];
const numOrbiters = 12;

for (let i = 0; i < numOrbiters; i++) {
  const hue = i / numOrbiters;
  const mat = new THREE.MeshStandardMaterial({
    color: new THREE.Color().setHSL(hue, 1.0, 0.5),
    roughness: 0.4,
    metalness: 0.3,
    emissive: new THREE.Color().setHSL(hue, 1.0, 0.2)
  });
  
  const size = 0.2 + Math.random() * 0.3;
  let geo;
  
  switch (Math.floor(i / 4)) {
    case 0: geo = new THREE.BoxGeometry(size, size, size); break;
    case 1: geo = new THREE.SphereGeometry(size * 0.5, 16, 16); break;
    default: geo = new THREE.ConeGeometry(size * 0.4, size, 8); break;
  }
  
  const obj = new THREE.Mesh(geo, mat);
  obj.userData = {
    orbitRadius: 3 + Math.random() * 2,
    orbitSpeed: 0.3 + Math.random() * 0.7,
    orbitHeight: (Math.random() - 0.5) * 3,
    phase: (i / numOrbiters) * Math.PI * 2
  };
  
  scene.add(obj);
  orbitObjects.push(obj);
}

// ===== GROUND =====
const ground = new THREE.Mesh(
  new THREE.PlaneGeometry(20, 20),
  new THREE.MeshStandardMaterial({
    color: 0x112233,
    roughness: 0.8,
    metalness: 0.1
  })
);
ground.rotation.x = -Math.PI / 2;
ground.position.y = -2;
scene.add(ground);

// ===== ANIMATION =====
const clock = new THREE.Clock();

function animate() {
  requestAnimationFrame(animate);
  
  const elapsed = clock.getElapsedTime();
  
  // Animate orbiting objects
  orbitObjects.forEach(obj => {
    const d = obj.userData;
    const angle = elapsed * d.orbitSpeed + d.phase;
    obj.position.set(
      Math.cos(angle) * d.orbitRadius,
      d.orbitHeight,
      Math.sin(angle) * d.orbitRadius
    );
    obj.rotation.x = elapsed * 0.5;
    obj.rotation.y = elapsed * 0.3;
  });
  
  // Update dynamic reflection
  chromeSphere.visible = false;
  cubeCamera.update(renderer, scene);
  chromeSphere.visible = true;
  
  controls.update();
  renderer.render(scene, camera);
}

animate();

window.addEventListener('resize', () => {
  camera.aspect = innerWidth / innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(innerWidth, innerHeight);
});
</script>
</body>
</html>
```

## 10. SphericalHarmonics3 (Light Probes)

```javascript
import { LightProbe, LightProbeGenerator } from 'three';
import { LightProbeGenerator } from 'three/examples/jsm/lights/LightProbeGenerator.js';

// ===== Light Probe =====
// Light Probe เก็บ ambient lighting information ด้วย Spherical Harmonics
// ใช้ memory น้อยกว่า CubeMap แต่ให้ low-frequency lighting

// สร้างจาก CubeTexture
const lightProbe = new THREE.LightProbe();
scene.add(lightProbe);

// Generate จาก environment map
LightProbeGenerator.fromCubeTexture(cubeTexture).then(probe => {
  lightProbe.copy(probe);
  lightProbe.intensity = 2.0;
});

// ===== SphericalHarmonics3 โดยตรง =====
const sh = new THREE.SphericalHarmonics3();

// ตั้งค่า coefficients โดยตรง (9 coefficients)
sh.coefficients[0].set(0.5, 0.5, 0.7); // L0 (ambient)
sh.coefficients[1].set(0.2, 0.2, 0.1); // L1
// ... up to L8

const manualProbe = new THREE.LightProbe(sh, 1.5);
scene.add(manualProbe);

// Generate SH from cubemap ด้วย CPU
function generateSHFromCubeTexture(cubeTexture) {
  const sh = new THREE.SphericalHarmonics3();
  
  // Sample directions on unit sphere
  const numSamples = 1000;
  
  for (let i = 0; i < numSamples; i++) {
    // Fibonacci sphere distribution
    const goldenRatio = (1 + Math.sqrt(5)) / 2;
    const theta = Math.acos(1 - 2 * (i / numSamples));
    const phi = 2 * Math.PI * i / goldenRatio;
    
    const dir = new THREE.Vector3(
      Math.sin(theta) * Math.cos(phi),
      Math.sin(theta) * Math.sin(phi),
      Math.cos(theta)
    );
    
    // Sample color from cube texture at this direction
    // (simplified - real implementation samples the texture)
    const color = new THREE.Color(0.5, 0.6, 0.8);
    
    // Add contribution to SH
    sh.addAmbient(color);
  }
  
  return sh;
}
```

## 11. Environment Map จาก 3D Scene

```javascript
// สร้าง environment map จาก scene ปัจจุบัน
class DynamicEnvironmentMapper {
  constructor(renderer, resolution = 512) {
    this.renderer = renderer;
    this.resolution = resolution;
    this.cubeRenderTarget = new THREE.WebGLCubeRenderTarget(resolution, {
      format: THREE.RGBFormat,
      generateMipmaps: true,
      minFilter: THREE.LinearMipmapLinearFilter
    });
    this.cubeCamera = new THREE.CubeCamera(0.1, 1000, this.cubeRenderTarget);
  }
  
  capture(scene, position) {
    this.cubeCamera.position.copy(position);
    this.cubeCamera.update(this.renderer, scene);
    return this.cubeRenderTarget.texture;
  }
  
  // PMREM version for PBR
  capturePBR(scene, position, pmremGenerator) {
    const texture = this.capture(scene, position);
    return pmremGenerator.fromCubemap(texture).texture;
  }
  
  // Capture static screenshot ของ environment
  captureStatic(scene, position) {
    const texture = this.capture(scene, position);
    
    // Convert to equirectangular format ถ้าต้องการ
    return texture;
  }
}
```

## สรุป

Environment Maps และ IBL เป็น foundation ของ PBR rendering:

1. **CubeTextureLoader** - 6 ภาพสำหรับ cubemap
2. **PMREMGenerator** - แปลง environment เป็น PBR-ready format
3. **RGBELoader** - โหลด HDR .hdr files
4. **CubeCamera** - real-time dynamic reflections
5. **LightProbe/SphericalHarmonics** - efficient ambient lighting

### Performance Tips
- ใช้ static environment เมื่อไม่ต้องการ real-time reflections
- ลด CubeCamera resolution (128-256 พอ)
- Update CubeCamera ทุกๆ N frames ไม่ต้องทุก frame
- ใช้ LightProbe แทน CubeCamera เมื่อไม่ต้องการ accurate reflections

---

## แบบฝึกหัด

1. **Easy**: โหลด equirectangular environment texture และใช้เป็น background + IBL
2. **Medium**: สร้างฉาก showcase ที่มี objects หลายชนิด (metal, plastic, glass) บน HDR environment
3. **Hard**: สร้าง dynamic reflection system ที่อัพเดต CubeCamera เฉพาะเมื่อมี object เคลื่อนที่
4. **Challenge**: สร้าง interior scene ที่มี Light Probes หลายจุดเพื่อให้ lighting ถูกต้องในทุกส่วนของห้อง
