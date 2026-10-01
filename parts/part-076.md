# Part 76: Loading Optimization

## บทนำ

3D assets มักมีขนาดใหญ่และโหลดช้า การ optimize loading เป็นสิ่งสำคัญมากสำหรับ UX ที่ดี บทนี้จะครอบคลุม DRACOLoader, KTX2, texture compression, progressive loading, และ caching strategies

---

## 76.1 DRACOLoader - Mesh Compression

Draco ลดขนาด geometry 90-95% โดยไม่สูญเสียคุณภาพ

```javascript
import * as THREE from 'three';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js';

// ตั้งค่า DRACOLoader
const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath('https://www.gstatic.com/draco/versioned/decoders/1.5.6/');
dracoLoader.setDecoderConfig({ type: 'js' }); // หรือ 'wasm' สำหรับ performance ดีกว่า
dracoLoader.preload(); // Pre-load decoder

// ใช้กับ GLTFLoader
const gltfLoader = new GLTFLoader();
gltfLoader.setDRACOLoader(dracoLoader);

// Load compressed model
gltfLoader.load('/models/compressed.glb', (gltf) => {
  scene.add(gltf.scene);
}, (progress) => {
  const percent = (progress.loaded / progress.total * 100).toFixed(0);
  console.log(`Loading: ${percent}%`);
});

// สร้าง Draco-compressed GLB ด้วย gltf-pipeline:
// npx gltf-pipeline -i model.glb -o model.draco.glb --draco.compressionLevel 10
```

---

## 76.2 KTX2 Texture Compression

KTX2 ลดขนาด texture และ GPU memory usage

```javascript
import { KTX2Loader } from 'three/examples/jsm/loaders/KTX2Loader.js';

// ตั้งค่า KTX2Loader
const ktx2Loader = new KTX2Loader();
ktx2Loader.setTranscoderPath('https://cdn.jsdelivr.net/npm/three@0.158.0/examples/jsm/libs/basis/');
ktx2Loader.detectSupport(renderer); // detect GPU support

// สร้าง KTX2 texture ด้วย toktx:
// toktx --t2 --encode uastc --uastc_rdo_l 1.0 output.ktx2 input.png

// Load compressed texture
ktx2Loader.load('/textures/diffuse.ktx2', (texture) => {
  texture.colorSpace = THREE.SRGBColorSpace;
  material.map = texture;
  material.needsUpdate = true;
});

// ใช้กับ GLTFLoader
gltfLoader.setKTX2Loader(ktx2Loader.detectSupport(renderer));
```

---

## 76.3 Loading Manager with Progress

```javascript
// src/utils/loadingManager.js
import * as THREE from 'three';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js';
import { TextureLoader } from 'three';

export class AssetLoader {
  constructor(onProgress) {
    this.onProgress = onProgress;
    this.total = 0;
    this.loaded = 0;
    
    // Loading Manager
    this.manager = new THREE.LoadingManager();
    
    this.manager.onStart = (url, loaded, total) => {
      this.total = total;
      this.loaded = loaded;
      this._updateProgress();
    };
    
    this.manager.onProgress = (url, loaded, total) => {
      this.total = total;
      this.loaded = loaded;
      this._updateProgress();
      console.log(`[${loaded}/${total}] ${url}`);
    };
    
    this.manager.onLoad = () => {
      console.log('All assets loaded!');
      this.onProgress?.(1.0, 'Complete');
    };
    
    this.manager.onError = (url) => {
      console.error('Error loading:', url);
    };
    
    // Setup loaders
    const dracoLoader = new DRACOLoader();
    dracoLoader.setDecoderPath('https://www.gstatic.com/draco/versioned/decoders/1.5.6/');
    dracoLoader.preload();
    
    this.gltfLoader = new GLTFLoader(this.manager);
    this.gltfLoader.setDRACOLoader(dracoLoader);
    
    this.textureLoader = new THREE.TextureLoader(this.manager);
    
    // Cache
    this.cache = new Map();
  }
  
  _updateProgress() {
    const progress = this.total > 0 ? this.loaded / this.total : 0;
    const message = `Loading ${this.loaded}/${this.total}`;
    this.onProgress?.(progress, message);
  }
  
  async loadGLTF(url) {
    if (this.cache.has(url)) {
      return this.cache.get(url);
    }
    
    return new Promise((resolve, reject) => {
      this.gltfLoader.load(url, (gltf) => {
        this.cache.set(url, gltf);
        resolve(gltf);
      }, undefined, reject);
    });
  }
  
  async loadTexture(url) {
    if (this.cache.has(url)) {
      return this.cache.get(url);
    }
    
    return new Promise((resolve, reject) => {
      this.textureLoader.load(url, (texture) => {
        this.cache.set(url, texture);
        resolve(texture);
      }, undefined, reject);
    });
  }
  
  async loadAll(assets) {
    const promises = assets.map(asset => {
      if (asset.type === 'gltf') return this.loadGLTF(asset.url);
      if (asset.type === 'texture') return this.loadTexture(asset.url);
      return Promise.reject(new Error(`Unknown type: ${asset.type}`));
    });
    
    return Promise.all(promises);
  }
  
  dispose() {
    this.cache.forEach(asset => {
      if (asset && typeof asset.dispose === 'function') {
        asset.dispose();
      }
    });
    this.cache.clear();
  }
}
```

---

## 76.4 Loading Screen Component

```javascript
// src/ui/loadingScreen.js
export class LoadingScreen {
  constructor(container) {
    this.container = container;
    this.overlay = null;
    this.progressBar = null;
    this.statusText = null;
    this._create();
  }
  
  _create() {
    this.overlay = document.createElement('div');
    this.overlay.style.cssText = `
      position: absolute;
      top: 0; left: 0; right: 0; bottom: 0;
      background: #050510;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      z-index: 100;
      transition: opacity 0.5s;
    `;
    
    // Logo/Title
    const logo = document.createElement('div');
    logo.style.cssText = `
      font-family: monospace;
      font-size: 2em;
      font-weight: bold;
      color: #4ecdc4;
      margin-bottom: 40px;
      letter-spacing: 0.2em;
      text-shadow: 0 0 20px #4ecdc4;
    `;
    logo.textContent = 'THREE.JS';
    
    // Animated spinner
    const spinner = document.createElement('div');
    spinner.style.cssText = `
      width: 60px;
      height: 60px;
      border: 3px solid rgba(78,205,196,0.2);
      border-top: 3px solid #4ecdc4;
      border-radius: 50%;
      animation: spin 1s linear infinite;
      margin-bottom: 30px;
    `;
    
    // Inject spinner animation
    const style = document.createElement('style');
    style.textContent = `@keyframes spin { to { transform: rotate(360deg); } }`;
    document.head.appendChild(style);
    
    // Progress bar
    const progressContainer = document.createElement('div');
    progressContainer.style.cssText = `
      width: 300px;
      height: 4px;
      background: rgba(78,205,196,0.2);
      border-radius: 2px;
      overflow: hidden;
      margin-bottom: 15px;
    `;
    
    this.progressBar = document.createElement('div');
    this.progressBar.style.cssText = `
      height: 100%;
      background: linear-gradient(90deg, #4ecdc4, #95e1d3);
      border-radius: 2px;
      width: 0%;
      transition: width 0.3s ease;
    `;
    progressContainer.appendChild(this.progressBar);
    
    // Status text
    this.statusText = document.createElement('div');
    this.statusText.style.cssText = `
      font-family: monospace;
      font-size: 0.75em;
      color: #4ecdc4;
      opacity: 0.7;
    `;
    this.statusText.textContent = 'Initializing...';
    
    this.overlay.appendChild(logo);
    this.overlay.appendChild(spinner);
    this.overlay.appendChild(progressContainer);
    this.overlay.appendChild(this.statusText);
    
    this.container.style.position = 'relative';
    this.container.appendChild(this.overlay);
  }
  
  update(progress, message) {
    if (this.progressBar) {
      this.progressBar.style.width = `${progress * 100}%`;
    }
    if (this.statusText && message) {
      this.statusText.textContent = message;
    }
  }
  
  hide() {
    if (this.overlay) {
      this.overlay.style.opacity = '0';
      setTimeout(() => {
        this.overlay?.remove();
        this.overlay = null;
      }, 500);
    }
  }
}
```

---

## 76.5 Progressive Loading (LOD)

```javascript
// src/utils/lod.js
import * as THREE from 'three';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

export class ProgressiveMeshLoader {
  constructor(scene) {
    this.scene = scene;
    this.loader = new GLTFLoader();
    this.lodObjects = new Map();
  }
  
  // โหลด LOD levels ตามระยะห่าง
  async loadWithLOD(modelConfigs, position) {
    const lod = new THREE.LOD();
    lod.position.copy(position);
    this.scene.add(lod);
    
    for (const config of modelConfigs) {
      // โหลด placeholder ก่อน
      const placeholder = new THREE.Mesh(
        new THREE.BoxGeometry(1, 1, 1),
        new THREE.MeshBasicMaterial({ wireframe: true, color: '#4ecdc4' })
      );
      lod.addLevel(placeholder, config.distance);
      
      // โหลด model จริงๆ
      this.loader.load(config.url, (gltf) => {
        // Replace placeholder
        const levelIndex = lod.levels.findIndex(l => l.object === placeholder);
        if (levelIndex !== -1) {
          lod.levels[levelIndex].object = gltf.scene;
          placeholder.geometry.dispose();
          placeholder.material.dispose();
        }
      });
    }
    
    return lod;
  }
}

// ใช้งาน:
// const loader = new ProgressiveMeshLoader(scene);
// loader.loadWithLOD([
//   { url: '/models/high.glb', distance: 0 },
//   { url: '/models/mid.glb', distance: 50 },
//   { url: '/models/low.glb', distance: 200 },
// ], position);
```

---

## 76.6 Texture Optimization

```javascript
// src/utils/textureOptimizer.js
import * as THREE from 'three';

export class TextureOptimizer {
  constructor(renderer) {
    this.renderer = renderer;
    this.maxAnisotropy = renderer.capabilities.getMaxAnisotropy();
  }
  
  // กำหนด texture settings ที่เหมาะสม
  optimize(texture, options = {}) {
    const {
      colorSpace = THREE.SRGBColorSpace,
      minFilter = THREE.LinearMipmapLinearFilter,
      magFilter = THREE.LinearFilter,
      anisotropy = this.maxAnisotropy,
      generateMipmaps = true,
      flipY = false,
    } = options;
    
    texture.colorSpace = colorSpace;
    texture.minFilter = minFilter;
    texture.magFilter = magFilter;
    texture.anisotropyLevel = Math.min(anisotropy, this.maxAnisotropy);
    texture.generateMipmaps = generateMipmaps;
    texture.flipY = flipY;
    
    return texture;
  }
  
  // Resize texture สำหรับ mobile
  resizeForDevice(texture, maxSize = 1024) {
    const isMobile = /iPhone|iPad|iPod|Android/i.test(navigator.userAgent);
    if (isMobile) {
      // Force lower resolution
      const targetSize = Math.min(maxSize, 512);
      if (texture.image && texture.image.width > targetSize) {
        const canvas = document.createElement('canvas');
        canvas.width = targetSize;
        canvas.height = targetSize;
        const ctx = canvas.getContext('2d');
        ctx.drawImage(texture.image, 0, 0, targetSize, targetSize);
        texture.image = canvas;
        texture.needsUpdate = true;
      }
    }
    return texture;
  }
  
  // เลือก format ตาม device
  getBestFormat() {
    const gl = this.renderer.getContext();
    const ext = {
      astc: gl.getExtension('WEBGL_compressed_texture_astc'),
      etc2: gl.getExtension('WEBGL_compressed_texture_etc'),
      s3tc: gl.getExtension('WEBGL_compressed_texture_s3tc'),
      pvrtc: gl.getExtension('WEBGL_compressed_texture_pvrtc'),
    };
    
    if (ext.astc) return 'ASTC (Mobile)';
    if (ext.etc2) return 'ETC2 (Android)';
    if (ext.s3tc) return 'S3TC (Desktop)';
    if (ext.pvrtc) return 'PVRTC (iOS old)';
    return 'None (uncompressed)';
  }
}
```

---

## 76.7 Complete Loading Demo

```javascript
// src/demos/loadingDemo.js
import * as THREE from 'three';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
import { LoadingScreen } from '../ui/loadingScreen.js';
import { AssetLoader } from '../utils/loadingManager.js';

export async function createLoadingDemo(container) {
  // Show loading screen
  const loadingScreen = new LoadingScreen(container);
  
  // Setup scene
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#0a0a1a');
  
  const camera = new THREE.PerspectiveCamera(45, container.clientWidth / container.clientHeight, 0.1, 100);
  camera.position.set(0, 2, 8);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.shadowMap.enabled = true;
  renderer.shadowMap.type = THREE.PCFSoftShadowMap;
  container.appendChild(renderer.domElement);
  
  const controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  
  // Asset loader with progress callback
  const assetLoader = new AssetLoader((progress, message) => {
    loadingScreen.update(progress, message);
  });
  
  // Simulate loading multiple assets
  const assets = [
    { type: 'texture', url: 'https://threejs.org/examples/textures/uv_grid_opengl.jpg' },
    { type: 'texture', url: 'https://threejs.org/examples/textures/brick_diffuse.jpg' },
  ];
  
  // Build scene (no network delay in this demo - using procedural)
  loadingScreen.update(0.2, 'Building scene...');
  await new Promise(resolve => setTimeout(resolve, 300));
  
  // === Scene Content ===
  loadingScreen.update(0.5, 'Loading textures...');
  
  let textures;
  try {
    textures = await assetLoader.loadAll(assets);
  } catch (e) {
    console.warn('Failed to load some textures, using fallbacks');
    textures = [null, null];
  }
  
  loadingScreen.update(0.8, 'Setting up materials...');
  await new Promise(resolve => setTimeout(resolve, 200));
  
  // Objects
  const objects = [];
  const colors = ['#4ecdc4', '#ff6b6b', '#ffd166', '#95e1d3', '#c3a6ff'];
  
  for (let i = 0; i < 15; i++) {
    const geometries = [
      new THREE.BoxGeometry(1, 1, 1),
      new THREE.SphereGeometry(0.6, 32, 32),
      new THREE.TorusGeometry(0.5, 0.2, 16, 64),
      new THREE.ConeGeometry(0.5, 1, 32),
      new THREE.OctahedronGeometry(0.7),
    ];
    
    const geo = geometries[i % geometries.length];
    const mat = new THREE.MeshStandardMaterial({
      color: colors[i % colors.length],
      metalness: 0.3 + Math.random() * 0.5,
      roughness: 0.1 + Math.random() * 0.5,
      map: textures[i % textures.length] || null,
    });
    
    const mesh = new THREE.Mesh(geo, mat);
    mesh.position.set(
      (Math.random() - 0.5) * 15,
      (Math.random() - 0.5) * 8,
      (Math.random() - 0.5) * 10
    );
    mesh.castShadow = true;
    mesh.receiveShadow = true;
    scene.add(mesh);
    objects.push(mesh);
  }
  
  // Floor
  const floor = new THREE.Mesh(
    new THREE.PlaneGeometry(30, 30),
    new THREE.MeshStandardMaterial({ color: '#111', roughness: 0.9 })
  );
  floor.rotation.x = -Math.PI / 2;
  floor.position.y = -4;
  floor.receiveShadow = true;
  scene.add(floor);
  
  // Lighting
  scene.add(new THREE.AmbientLight(0xffffff, 0.4));
  const dirLight = new THREE.DirectionalLight(0xffffff, 1.5);
  dirLight.position.set(5, 10, 5);
  dirLight.castShadow = true;
  scene.add(dirLight);
  
  loadingScreen.update(1.0, 'Ready!');
  await new Promise(resolve => setTimeout(resolve, 500));
  
  // Hide loading screen
  loadingScreen.hide();
  
  // === Animation ===
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const t = clock.getElapsedTime();
    
    objects.forEach((obj, i) => {
      obj.rotation.x = t * (0.2 + i * 0.01);
      obj.rotation.y = t * (0.3 + i * 0.01);
      obj.position.y = Math.sin(t * 0.5 + i) * 0.3 + (Math.random() - 0.5) * 0.001;
    });
    
    controls.update();
    renderer.render(scene, camera);
  }
  
  animate();
  
  window.addEventListener('resize', () => {
    const w = container.clientWidth;
    const h = container.clientHeight;
    camera.aspect = w / h;
    camera.updateProjectionMatrix();
    renderer.setSize(w, h);
  });
  
  return () => {
    cancelAnimationFrame(animId);
    assetLoader.dispose();
    controls.dispose();
    renderer.dispose();
  };
}
```

---

## 76.8 GLTF Optimization Checklist

```javascript
// Optimization tips
const GLTF_OPTIMIZATION_CHECKLIST = {
  geometry: [
    'ใช้ Draco compression (--draco.compressionLevel 7-10)',
    'Merge meshes ที่ไม่เคลื่อนที่',
    'ลด polygon count สำหรับ objects ที่ไกล',
    'Remove unused morph targets',
    'ลบ vertex attributes ที่ไม่ใช้',
  ],
  textures: [
    'ใช้ KTX2/Basis Universal compression',
    'Power of 2 dimensions (512, 1024, 2048)',
    'Pack textures (ao/rough/metal in RGB channels)',
    'Mipmap generation',
    'ลด resolution สำหรับ secondary textures',
  ],
  scene: [
    'Instance repeated objects',
    'LOD levels สำหรับ complex objects',
    'Lazy loading (โหลดเมื่อจำเป็น)',
    'Frustum culling',
    'Occlusion culling สำหรับ large scenes',
  ],
  loading: [
    'CDN for static assets',
    'Preload critical assets',
    'Service Worker caching',
    'HTTP/2 for parallel loading',
    'Gzip/Brotli compression',
  ],
};

// gltf-pipeline commands:
// Draco: npx gltf-pipeline -i in.glb -o out.glb --draco.compressionLevel 10
// Optimize: npx gltf-pipeline -i in.gltf -o out.glb --binary
// KTX2: npm install @gltf-transform/cli && npx gltf-transform optimize in.glb out.glb
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Loading Progress UI
สร้าง loading screen ที่:
- 3D spinner ด้วย Three.js
- Real-time progress bar
- Asset thumbnails ก่อน load
- Smooth fade in เมื่อพร้อม

### แบบฝึกหัดที่ 2: Asset Cache System
สร้าง cache ที่:
- IndexedDB สำหรับ persistent cache
- LRU eviction policy
- Cache versioning
- Background preload

### แบบฝึกหัดที่ 3: Adaptive Loading
สร้าง system ที่:
- Detect network speed
- โหลด LOD ตาม bandwidth
- เปลี่ยน quality runtime
- Report loading metrics

### แบบฝึกหัดที่ 4: Streaming Geometry
สร้าง terrain ที่:
- Chunk-based loading
- Load เมื่อ camera เข้าใกล้
- Unload เมื่อ camera ออก
- Smooth LOD transitions

---

## สรุป

ในบทนี้เราเรียนรู้:
1. **DRACOLoader** - Geometry compression 90-95%
2. **KTX2Loader** - GPU-native texture compression
3. **LoadingManager** - Track loading progress
4. **Loading Screen** - UX ที่ดีระหว่างโหลด
5. **LOD System** - Progressive detail
6. **Texture Optimization** - Format, size, filtering
7. **Optimization Checklist** - Best practices

บทถัดไปเราจะเรียน **Worker Threads & Performance**
