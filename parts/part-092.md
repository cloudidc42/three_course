# Part 92: Deployment & Production

## บทนำ: Production Three.js Apps

การ deploy Three.js app ให้ production-ready ต้องคำนึงถึง:
- **Bundle size** - Three.js เป็น library ขนาดใหญ่
- **Asset delivery** - textures, models, HDRIs
- **Performance** - ต้องรันได้บน hardware หลากหลาย
- **SEO** - 3D sites ยังต้อง crawlable
- **PWA** - offline support

---

## 1. Vite Build Configuration

```bash
# ติดตั้ง
npm create vite@latest my-three-app -- --template vanilla
cd my-three-app
npm install three
npm install -D vite-plugin-glsl @types/three
```

```javascript
// vite.config.js
import { defineConfig } from 'vite';
import glsl from 'vite-plugin-glsl';
import { resolve } from 'path';

export default defineConfig({
  plugins: [
    glsl()  // Import GLSL files as strings
  ],
  
  build: {
    target: 'es2020',  // Modern browsers support
    outDir: 'dist',
    
    rollupOptions: {
      output: {
        // Manual code splitting
        manualChunks: {
          'three-core': ['three'],
          'three-addons': [
            'three/addons/controls/OrbitControls',
            'three/addons/loaders/GLTFLoader',
            'three/addons/loaders/DRACOLoader',
          ],
          'vendor': ['gsap', 'lil-gui']
        }
      }
    },
    
    // Asset optimization
    assetsInlineLimit: 4096,  // Inline files < 4KB
    chunkSizeWarningLimit: 1000,  // 1MB warning
    
    // Source maps for debugging (disable for smaller size)
    sourcemap: false,
    
    // Minification
    minify: 'terser',
    terserOptions: {
      compress: {
        drop_console: true,  // Remove console.log in production
        drop_debugger: true
      }
    }
  },
  
  // Optimize dependencies
  optimizeDeps: {
    include: ['three']
  },
  
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
      '@shaders': resolve(__dirname, 'src/shaders'),
      '@assets': resolve(__dirname, 'public/assets')
    }
  }
});
```

---

## 2. Tree Shaking Three.js

```javascript
// ❌ Bad - imports entire Three.js
import * as THREE from 'three';

// ✅ Good - imports only what you need
import {
  WebGLRenderer,
  Scene,
  PerspectiveCamera,
  Mesh,
  BoxGeometry,
  MeshStandardMaterial,
  DirectionalLight,
  AmbientLight,
  Vector3,
  Color
} from 'three';

// Check bundle size:
// npm run build -- --analyze
```

---

## 3. Asset Optimization Pipeline

```bash
# Texture optimization
# ติดตั้ง tools
npm install -g squoosh-cli

# Compress textures
squoosh-cli --webp '{}' textures/*.png
squoosh-cli --avif '{}' textures/*.jpg

# GLTF optimization
npm install -g @gltf-transform/cli
gltf-transform optimize model.glb model-optimized.glb
gltf-transform draco model.glb model-draco.glb
gltf-transform webp model.glb model-webp.glb  # Convert textures to WebP
```

```javascript
// src/utils/asset-loader.js
// Smart asset loader ที่เลือก format ตาม browser support

class AssetOptimizer {
  static #webpSupported = null;
  static #avifSupported = null;
  
  static async detectSupport() {
    // Test WebP support
    const webpPromise = new Promise(resolve => {
      const img = new Image();
      img.onload = img.onerror = () => resolve(img.height === 1);
      img.src = 'data:image/webp;base64,UklGRiQAAABXRUJQVlA4IBgAAAAwAQCdASoBAAEAAwA0JZQCdAEO/gHOAAA=';
    });
    
    // Test AVIF support
    const avifPromise = new Promise(resolve => {
      const img = new Image();
      img.onload = img.onerror = () => resolve(img.height === 1);
      img.src = 'data:image/avif;base64,AAAAIGZ0eXBhdmlmAAAAAGF2aWZtaWYxbWlhZk1BMUIAAADybWV0YQ==';
    });
    
    [this.#webpSupported, this.#avifSupported] = await Promise.all([webpPromise, avifPromise]);
  }
  
  static getOptimalTextureURL(basePath) {
    if (this.#avifSupported) return basePath.replace(/\.(png|jpg)$/, '.avif');
    if (this.#webpSupported) return basePath.replace(/\.(png|jpg)$/, '.webp');
    return basePath;
  }
}

// Init ก่อนใช้งาน
await AssetOptimizer.detectSupport();
const textureURL = AssetOptimizer.getOptimalTextureURL('textures/wood.jpg');
```

---

## 4. PWA Setup

```javascript
// vite.config.js - เพิ่ม PWA plugin
import { VitePWA } from 'vite-plugin-pwa';

export default defineConfig({
  plugins: [
    glsl(),
    VitePWA({
      registerType: 'autoUpdate',
      workbox: {
        // Cache Three.js assets
        globPatterns: ['**/*.{js,css,html,png,jpg,webp,avif,glb,gltf,hdr,exr}'],
        
        // Cache strategy สำหรับ assets ขนาดใหญ่
        runtimeCaching: [
          {
            urlPattern: /\.(?:glb|gltf)$/,
            handler: 'CacheFirst',
            options: {
              cacheName: 'three-models',
              expiration: {
                maxEntries: 20,
                maxAgeSeconds: 30 * 24 * 60 * 60  // 30 days
              }
            }
          },
          {
            urlPattern: /\.(?:hdr|exr)$/,
            handler: 'CacheFirst',
            options: {
              cacheName: 'three-hdri',
              expiration: {
                maxEntries: 5,
                maxAgeSeconds: 60 * 24 * 60 * 60  // 60 days
              }
            }
          },
          {
            urlPattern: /\.(?:png|jpg|webp|avif)$/,
            handler: 'CacheFirst',
            options: {
              cacheName: 'three-textures',
              expiration: {
                maxEntries: 50,
                maxAgeSeconds: 30 * 24 * 60 * 60  // 30 days
              }
            }
          }
        ]
      },
      
      manifest: {
        name: 'My Three.js App',
        short_name: '3D App',
        description: 'Interactive 3D Experience',
        theme_color: '#0a0a1a',
        background_color: '#0a0a1a',
        display: 'standalone',
        icons: [
          { src: 'icons/icon-192.png', sizes: '192x192', type: 'image/png' },
          { src: 'icons/icon-512.png', sizes: '512x512', type: 'image/png' }
        ]
      }
    })
  ]
});
```

---

## 5. CDN Deployment (Vercel/Netlify)

```json
// vercel.json
{
  "version": 2,
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "X-Content-Type-Options",
          "value": "nosniff"
        },
        {
          "key": "X-Frame-Options",
          "value": "DENY"
        },
        {
          "key": "X-XSS-Protection",
          "value": "1; mode=block"
        },
        {
          "key": "Content-Security-Policy",
          "value": "default-src 'self'; script-src 'self' 'unsafe-eval'; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob:; worker-src blob:; connect-src 'self';"
        }
      ]
    },
    {
      "source": "/assets/(.*)",
      "headers": [
        {
          "key": "Cache-Control",
          "value": "public, max-age=31536000, immutable"
        }
      ]
    }
  ]
}
```

```toml
# netlify.toml
[build]
  command = "npm run build"
  publish = "dist"

[[headers]]
  for = "/*"
  [headers.values]
    X-Frame-Options = "DENY"
    X-XSS-Protection = "1; mode=block"
    Content-Security-Policy = "default-src 'self'; script-src 'self' 'unsafe-eval'; worker-src blob:; img-src 'self' data: blob:;"

[[headers]]
  for = "/assets/*"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"

# Redirect for SPA routing
[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

---

## 6. Performance Monitoring

```javascript
// src/utils/performance-monitor.js
class PerformanceMonitor {
  constructor() {
    this.stats = {
      fps: 0,
      frameTime: 0,
      memory: 0,
      drawCalls: 0,
      triangles: 0
    };
    
    this.frameCount = 0;
    this.lastTime = performance.now();
    this.fpsHistory = [];
    
    // สร้าง display element
    this._createDisplay();
  }
  
  _createDisplay() {
    this.display = document.createElement('div');
    this.display.style.cssText = `
      position: fixed; top: 10px; left: 10px;
      background: rgba(0,0,0,0.7);
      color: #0f0; font: 12px monospace;
      padding: 8px 12px;
      border-radius: 4px;
      pointer-events: none;
      z-index: 9999;
      line-height: 1.6;
    `;
    document.body.appendChild(this.display);
  }
  
  update(renderer) {
    const now = performance.now();
    const delta = now - this.lastTime;
    this.lastTime = now;
    
    this.frameCount++;
    this.stats.frameTime = delta;
    this.stats.fps = Math.round(1000 / delta);
    
    // Track FPS history
    this.fpsHistory.push(this.stats.fps);
    if (this.fpsHistory.length > 60) this.fpsHistory.shift();
    
    // Renderer stats
    if (renderer) {
      const info = renderer.info;
      this.stats.drawCalls = info.render.calls;
      this.stats.triangles = info.render.triangles;
      
      // Memory (if available)
      if (performance.memory) {
        this.stats.memory = Math.round(performance.memory.usedJSHeapSize / 1024 / 1024);
      }
    }
    
    this._render();
  }
  
  _render() {
    const minFPS = Math.min(...this.fpsHistory);
    const avgFPS = Math.round(this.fpsHistory.reduce((a, b) => a + b, 0) / this.fpsHistory.length);
    
    const fpsColor = this.stats.fps >= 55 ? '#0f0' : this.stats.fps >= 30 ? '#ff0' : '#f00';
    
    this.display.innerHTML = `
      <span style="color:${fpsColor}">FPS: ${this.stats.fps}</span> | Avg: ${avgFPS} | Min: ${minFPS}<br>
      Frame: ${this.stats.frameTime.toFixed(1)}ms<br>
      Draws: ${this.stats.drawCalls} | Tris: ${(this.stats.triangles / 1000).toFixed(1)}k<br>
      ${this.stats.memory ? `Memory: ${this.stats.memory}MB` : ''}
    `;
  }
  
  hide() {
    this.display.style.display = 'none';
  }
  
  show() {
    this.display.style.display = 'block';
  }
}
```

---

## 7. Error Handling & Recovery

```javascript
// src/utils/error-handler.js
class ThreeJSErrorHandler {
  constructor(renderer, scene, camera) {
    this.renderer = renderer;
    this.scene = scene;
    this.camera = camera;
    
    this._setupContextLossHandling();
    this._setupGlobalErrorHandler();
  }
  
  _setupContextLossHandling() {
    const canvas = this.renderer.domElement;
    
    canvas.addEventListener('webglcontextlost', (event) => {
      console.warn('WebGL context lost');
      event.preventDefault();
      
      this._showContextLostUI();
    });
    
    canvas.addEventListener('webglcontextrestored', () => {
      console.log('WebGL context restored');
      
      // Re-upload all textures, buffers etc.
      this._restoreContext();
    });
  }
  
  _showContextLostUI() {
    const overlay = document.createElement('div');
    overlay.id = 'context-lost-overlay';
    overlay.style.cssText = `
      position: fixed; inset: 0;
      background: rgba(0,0,0,0.9);
      color: white;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      font-family: sans-serif;
      z-index: 9999;
    `;
    overlay.innerHTML = `
      <h2>GPU Context Lost</h2>
      <p>กรุณาโหลดหน้าใหม่</p>
      <button onclick="location.reload()" style="
        padding: 10px 20px;
        background: #4488ff;
        border: none;
        color: white;
        border-radius: 5px;
        cursor: pointer;
        margin-top: 10px;
        font-size: 16px;
      ">โหลดใหม่</button>
    `;
    document.body.appendChild(overlay);
  }
  
  _restoreContext() {
    // Remove overlay
    const overlay = document.getElementById('context-lost-overlay');
    if (overlay) overlay.remove();
    
    // Re-init renderer settings
    this.renderer.outputColorSpace = THREE.SRGBColorSpace;
    this.renderer.toneMapping = THREE.ACESFilmicToneMapping;
    
    // Dispose and reload textures
    this.scene.traverse(obj => {
      if (obj.material) {
        const mats = Array.isArray(obj.material) ? obj.material : [obj.material];
        mats.forEach(m => m.needsUpdate = true);
      }
    });
    
    console.log('Context restored successfully');
  }
  
  _setupGlobalErrorHandler() {
    window.addEventListener('error', (e) => {
      if (e.message.includes('WebGL') || e.message.includes('THREE')) {
        console.error('Three.js error:', e.message);
        // Send to error tracking service (Sentry, etc.)
      }
    });
    
    window.addEventListener('unhandledrejection', (e) => {
      if (e.reason?.message?.includes('WebGL')) {
        console.error('WebGL Promise rejection:', e.reason);
      }
    });
  }
}
```

---

## 8. Production Checklist HTML Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta name="description" content="Three.js Production App - Interactive 3D Experience">
  
  <!-- Open Graph for sharing -->
  <meta property="og:title" content="Three.js App">
  <meta property="og:description" content="Interactive 3D Experience">
  <meta property="og:image" content="/og-image.jpg">
  
  <title>Three.js Production App</title>
  
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { overflow: hidden; background: #0a0a1a; }
    canvas { display: block; }
    
    #loading {
      position: fixed; inset: 0;
      background: #0a0a1a;
      display: flex; flex-direction: column;
      align-items: center; justify-content: center;
      z-index: 100; transition: opacity 1s;
      color: white; font-family: sans-serif;
    }
    
    #loading-bar-bg {
      width: 200px; height: 3px;
      background: rgba(255,255,255,0.2);
      border-radius: 2px; margin-top: 15px;
    }
    
    #loading-bar {
      height: 100%; width: 0%;
      background: linear-gradient(to right, #4488ff, #44ffff);
      border-radius: 2px;
      transition: width 0.3s ease;
    }
    
    #stats-panel {
      position: fixed; top: 10px; left: 10px;
      background: rgba(0,0,0,0.7);
      color: #0f0; font: 12px monospace;
      padding: 8px; border-radius: 4px;
      line-height: 1.6;
    }
    
    #controls-hint {
      position: fixed; bottom: 15px; left: 50%;
      transform: translateX(-50%);
      color: rgba(255,255,255,0.5);
      font: 13px sans-serif;
    }
    
    /* WebGL not supported message */
    #no-webgl {
      position: fixed; inset: 0;
      background: #000; color: white;
      display: none; align-items: center;
      justify-content: center; text-align: center;
      font-family: sans-serif; padding: 20px;
    }
  </style>
</head>
<body>
  <canvas id="canvas"></canvas>
  
  <div id="loading">
    <div style="font-size: 18px; color: #88aaff;">กำลังโหลด...</div>
    <div id="loading-bar-bg">
      <div id="loading-bar"></div>
    </div>
    <div id="loading-percent" style="font-size: 12px; color: #666; margin-top: 8px;">0%</div>
  </div>
  
  <div id="stats-panel"></div>
  
  <div id="controls-hint">คลิกลาก: หมุน | Scroll: Zoom</div>
  
  <div id="no-webgl">
    <div>
      <h2>WebGL ไม่รองรับ</h2>
      <p>กรุณาอัพเดท browser หรือเปิดใช้งาน hardware acceleration</p>
    </div>
  </div>
  
  <script type="module">
    import * as THREE from 'https://cdn.skypack.dev/three@0.160.0';
    import { OrbitControls } from 'https://cdn.skypack.dev/three@0.160.0/examples/jsm/controls/OrbitControls.js';
    
    // ===== Feature Detection =====
    function checkWebGLSupport() {
      try {
        const canvas = document.createElement('canvas');
        return !!(canvas.getContext('webgl2') || canvas.getContext('webgl'));
      } catch (e) {
        return false;
      }
    }
    
    if (!checkWebGLSupport()) {
      document.getElementById('no-webgl').style.display = 'flex';
      document.getElementById('loading').style.display = 'none';
      throw new Error('WebGL not supported');
    }
    
    // ===== Renderer =====
    const canvas = document.getElementById('canvas');
    const renderer = new THREE.WebGLRenderer({
      canvas,
      antialias: window.devicePixelRatio < 2,  // MSAA สำหรับ low-DPI เท่านั้น
      powerPreference: 'high-performance'
    });
    
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.outputColorSpace = THREE.SRGBColorSpace;
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.toneMappingExposure = 1.2;
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    
    // ===== Scene =====
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x0a0a1a);
    scene.fog = new THREE.FogExp2(0x0a0a1a, 0.015);
    
    const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 100);
    camera.position.set(0, 3, 10);
    
    const controls = new OrbitControls(camera, canvas);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
    controls.maxDistance = 30;
    controls.minDistance = 2;
    
    // ===== Loading Manager =====
    const loadingManager = new THREE.LoadingManager();
    const loadingBar = document.getElementById('loading-bar');
    const loadingPercent = document.getElementById('loading-percent');
    
    loadingManager.onProgress = (url, loaded, total) => {
      const progress = loaded / total;
      loadingBar.style.width = `${progress * 100}%`;
      loadingPercent.textContent = `${Math.round(progress * 100)}%`;
    };
    
    loadingManager.onLoad = () => {
      setTimeout(hideLoading, 300);
    };
    
    async function hideLoading() {
      const el = document.getElementById('loading');
      el.style.opacity = '0';
      setTimeout(() => el.style.display = 'none', 1000);
    }
    
    // ===== Scene Content =====
    // สร้าง showcase scene สำหรับ demo
    const textureLoader = new THREE.TextureLoader(loadingManager);
    
    // Grid
    const grid = new THREE.GridHelper(30, 30, 0x333344, 0x222233);
    scene.add(grid);
    
    // Central showcase - floating objects
    const objects = [];
    const geo = new THREE.IcosahedronGeometry(1, 1);
    
    for (let i = 0; i < 8; i++) {
      const mat = new THREE.MeshStandardMaterial({
        color: new THREE.Color().setHSL(i / 8, 0.8, 0.5),
        metalness: 0.5,
        roughness: 0.3,
        emissive: new THREE.Color().setHSL(i / 8, 0.5, 0.1)
      });
      
      const mesh = new THREE.Mesh(geo, mat);
      const angle = (i / 8) * Math.PI * 2;
      const radius = 5;
      
      mesh.position.set(
        Math.cos(angle) * radius,
        Math.sin(i * 0.7) * 1.5 + 2,
        Math.sin(angle) * radius
      );
      
      mesh.castShadow = true;
      scene.add(mesh);
      objects.push({ mesh, angle: angle, speed: 0.2 + Math.random() * 0.3 });
    }
    
    // Ground plane
    const groundMat = new THREE.MeshStandardMaterial({ color: 0x222233, roughness: 0.9 });
    const ground = new THREE.Mesh(new THREE.PlaneGeometry(30, 30), groundMat);
    ground.rotation.x = -Math.PI / 2;
    ground.receiveShadow = true;
    scene.add(ground);
    
    // Particles
    const particleGeo = new THREE.BufferGeometry();
    const count = 2000;
    const pos = new Float32Array(count * 3);
    for (let i = 0; i < count * 3; i++) {
      pos[i] = (Math.random() - 0.5) * 40;
    }
    particleGeo.setAttribute('position', new THREE.BufferAttribute(pos, 3));
    const particles = new THREE.Points(
      particleGeo,
      new THREE.PointsMaterial({ color: 0x4488ff, size: 0.05, transparent: true, opacity: 0.6 })
    );
    scene.add(particles);
    
    // Lighting
    scene.add(new THREE.AmbientLight(0x112244, 0.5));
    
    const mainLight = new THREE.DirectionalLight(0xffffff, 2);
    mainLight.position.set(5, 10, 5);
    mainLight.castShadow = true;
    mainLight.shadow.mapSize.set(2048, 2048);
    mainLight.shadow.camera.near = 0.5;
    mainLight.shadow.camera.far = 50;
    mainLight.shadow.camera.left = -15;
    mainLight.shadow.camera.right = 15;
    mainLight.shadow.camera.top = 15;
    mainLight.shadow.camera.bottom = -15;
    scene.add(mainLight);
    
    const fillLight = new THREE.PointLight(0x4488ff, 1, 20);
    fillLight.position.set(-5, 3, -5);
    scene.add(fillLight);
    
    // Simulate loading
    let loadProgress = 0;
    const simulateLoad = setInterval(() => {
      loadProgress += 0.05;
      loadingBar.style.width = `${Math.min(loadProgress * 100, 95)}%`;
      loadingPercent.textContent = `${Math.round(Math.min(loadProgress * 100, 95))}%`;
      
      if (loadProgress >= 1) {
        clearInterval(simulateLoad);
        loadingBar.style.width = '100%';
        loadingPercent.textContent = '100%';
        setTimeout(hideLoading, 300);
      }
    }, 50);
    
    // ===== Performance Monitor =====
    const statsPanel = document.getElementById('stats-panel');
    let frameCount = 0;
    let lastFPSTime = performance.now();
    let fps = 0;
    
    // ===== Resize =====
    window.addEventListener('resize', () => {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    });
    
    // ===== Animation =====
    const clock = new THREE.Clock();
    let frame = 0;
    
    function animate() {
      requestAnimationFrame(animate);
      frame++;
      
      const time = clock.getElapsedTime();
      const delta = clock.getDelta();
      
      // Animate objects
      objects.forEach((item, i) => {
        item.angle += item.speed * delta;
        const r = 5;
        item.mesh.position.x = Math.cos(item.angle) * r;
        item.mesh.position.z = Math.sin(item.angle) * r;
        item.mesh.position.y = Math.sin(time * 0.5 + i) * 1.5 + 2;
        item.mesh.rotation.x = time * 0.5;
        item.mesh.rotation.y = time * 0.3;
      });
      
      // Rotate particles
      particles.rotation.y = time * 0.02;
      
      // FPS counter
      frameCount++;
      const now = performance.now();
      if (now - lastFPSTime >= 1000) {
        fps = frameCount;
        frameCount = 0;
        lastFPSTime = now;
      }
      
      // Stats
      if (frame % 10 === 0) {
        const info = renderer.info;
        const fpsColor = fps >= 55 ? '#0f0' : fps >= 30 ? '#ff0' : '#f00';
        statsPanel.innerHTML = `
          <span style="color:${fpsColor}">FPS: ${fps}</span><br>
          Draws: ${info.render.calls}<br>
          Tris: ${(info.render.triangles / 1000).toFixed(1)}k<br>
          Objs: ${scene.children.length}
        `;
      }
      
      controls.update();
      renderer.render(scene, camera);
    }
    
    animate();
    
    // ===== Context Loss Recovery =====
    canvas.addEventListener('webglcontextlost', e => {
      e.preventDefault();
      alert('WebGL context lost. กรุณาโหลดหน้าใหม่');
    });
  </script>
</body>
</html>
```

---

## 9. Deployment Checklist

```
PRE-DEPLOYMENT:
  [ ] npm run build สำเร็จ ไม่มี errors
  [ ] Bundle size < 500KB (gzipped)
  [ ] Textures optimized (WebP/AVIF)
  [ ] Models compressed (Draco/Meshopt)
  [ ] Tree shaking เปิดใช้งาน
  [ ] Source maps ปิด (production)
  [ ] Console.log ถูก strip ออก

SECURITY:
  [ ] Content-Security-Policy headers
  [ ] CORS configured สำหรับ asset CDN
  [ ] HTTPS only
  [ ] X-Frame-Options: DENY

PERFORMANCE:
  [ ] Cache headers สำหรับ static assets (immutable)
  [ ] Gzip/Brotli compression บน server
  [ ] CDN สำหรับ large assets
  [ ] Lazy loading สำหรับ scenes ที่ไม่ใช้ทันที

TESTING:
  [ ] ทดสอบบน iPhone Safari
  [ ] ทดสอบบน Android Chrome
  [ ] ทดสอบบน Firefox
  [ ] ทดสอบด้วย hardware เก่า
  [ ] WebGL context loss recovery

PWA (ถ้าใช้):
  [ ] manifest.json ครบถ้วน
  [ ] Service worker registered
  [ ] Offline fallback page
  [ ] Install prompt

MONITORING:
  [ ] Error tracking (Sentry)
  [ ] Performance monitoring
  [ ] Analytics
```

---

## 10. สรุปคอร์ส

ตลอด 92 parts เราได้เรียนรู้:

**พื้นฐาน**: Scene, Camera, Renderer, Geometry, Material, Light, Shadow, Texture, Animation

**Advanced Rendering**: PBR, Custom Shaders (GLSL), Post-Processing, Deferred Rendering, WebGPU, TSL

**Systems**: ECS Game Architecture, Asset Loading, Scene Management, Character Animation, IK

**Applications**: Data Visualization, Archviz, Product Configurators, FPS Games, Portfolios

**Production**: Testing, Performance Optimization, PWA, Deployment, CI/CD

**Three.js คือ library ที่ทรงพลัง** - ขอให้ทดลองสร้าง projects ของตัวเอง!
