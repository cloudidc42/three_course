# Part 90: Scene Transitions & Loading

## บทนำ: Scene Transitions ที่ดี

Transition ระหว่าง scenes เป็นส่วนสำคัญของ UX ใน 3D web apps การทำ transition ที่ดีช่วยให้ผู้ใช้:
- รู้ว่ากำลัง navigate ไปที่ไหน
- ไม่รู้สึก disoriented
- ลดการรอในระหว่างโหลด

---

## 1. Loading Manager

```javascript
class LoadingManager {
  constructor() {
    this.threeManager = new THREE.LoadingManager();
    this.items = new Map();
    this.totalItems = 0;
    this.loadedItems = 0;
    this.onProgress = null;
    this.onComplete = null;
    this.onError = null;
    
    this._setupManager();
  }
  
  _setupManager() {
    this.threeManager.onStart = (url, loaded, total) => {
      this.totalItems = total;
      this.loadedItems = loaded;
      this._updateProgress();
    };
    
    this.threeManager.onProgress = (url, loaded, total) => {
      this.totalItems = total;
      this.loadedItems = loaded;
      this._updateProgress();
    };
    
    this.threeManager.onLoad = () => {
      if (this.onComplete) this.onComplete();
    };
    
    this.threeManager.onError = (url) => {
      console.error(`Error loading: ${url}`);
      if (this.onError) this.onError(url);
    };
  }
  
  _updateProgress() {
    const progress = this.totalItems > 0 ? this.loadedItems / this.totalItems : 0;
    if (this.onProgress) this.onProgress(progress, this.loadedItems, this.totalItems);
  }
  
  // Queue loaders ต่างๆ
  getTextureLoader() {
    return new THREE.TextureLoader(this.threeManager);
  }
  
  getGLTFLoader() {
    const loader = new GLTFLoader(this.threeManager);
    return loader;
  }
  
  getAudioLoader() {
    return new THREE.AudioLoader(this.threeManager);
  }
}
```

---

## 2. Loading Screen with Progress

```javascript
class LoadingScreen {
  constructor() {
    this.element = this._createDOM();
    document.body.appendChild(this.element);
    this.visible = true;
  }
  
  _createDOM() {
    const div = document.createElement('div');
    div.id = 'loading-screen';
    div.innerHTML = `
      <div class="loading-bg"></div>
      <div class="loading-content">
        <div class="loading-logo">⬡</div>
        <div class="loading-title">กำลังโหลด...</div>
        <div class="loading-progress-bar">
          <div class="loading-progress-fill" id="loadProgress"></div>
        </div>
        <div class="loading-percent" id="loadPercent">0%</div>
        <div class="loading-detail" id="loadDetail"></div>
      </div>
    `;
    
    // Inline styles
    div.style.cssText = `
      position: fixed; inset: 0;
      background: #0a0a1a;
      display: flex; align-items: center; justify-content: center;
      z-index: 9999;
      transition: opacity 1s ease;
    `;
    
    const style = document.createElement('style');
    style.textContent = `
      .loading-bg {
        position: absolute; inset: 0;
        background: radial-gradient(ellipse at center, #0d0d2e 0%, #000 100%);
      }
      .loading-content {
        position: relative; z-index: 1;
        text-align: center; color: white;
        width: 300px;
      }
      .loading-logo {
        font-size: 60px; animation: spin 3s linear infinite;
        color: #4488ff;
      }
      @keyframes spin {
        0% { transform: rotate(0deg); }
        100% { transform: rotate(360deg); }
      }
      .loading-title {
        font-family: sans-serif;
        font-size: 18px; margin: 10px 0;
        color: #88aaff;
      }
      .loading-progress-bar {
        width: 100%; height: 4px;
        background: rgba(255,255,255,0.1);
        border-radius: 2px; overflow: hidden;
        margin: 10px 0;
      }
      .loading-progress-fill {
        height: 100%; width: 0%;
        background: linear-gradient(to right, #4488ff, #44ffff);
        border-radius: 2px;
        transition: width 0.3s ease;
      }
      .loading-percent {
        font-size: 14px; color: #aaa;
        font-family: monospace;
      }
      .loading-detail {
        font-size: 11px; color: #666;
        margin-top: 5px; font-family: monospace;
        overflow: hidden; text-overflow: ellipsis;
        white-space: nowrap;
      }
    `;
    document.head.appendChild(style);
    
    return div;
  }
  
  setProgress(progress, detail = '') {
    const fill = document.getElementById('loadProgress');
    const percent = document.getElementById('loadPercent');
    const detailEl = document.getElementById('loadDetail');
    
    if (fill) fill.style.width = `${progress * 100}%`;
    if (percent) percent.textContent = `${Math.round(progress * 100)}%`;
    if (detailEl) detailEl.textContent = detail;
  }
  
  hide(duration = 1000) {
    return new Promise(resolve => {
      this.element.style.opacity = '0';
      setTimeout(() => {
        this.element.style.display = 'none';
        this.visible = false;
        resolve();
      }, duration);
    });
  }
  
  show() {
    this.element.style.display = 'flex';
    this.element.style.opacity = '1';
    this.visible = true;
  }
}
```

---

## 3. Fade Transitions

```javascript
class FadeTransition {
  constructor(renderer) {
    this.renderer = renderer;
    this.fadeOverlay = this._createFadeOverlay();
    this.transitioning = false;
  }
  
  _createFadeOverlay() {
    const overlay = document.createElement('div');
    overlay.style.cssText = `
      position: fixed; inset: 0;
      background: black;
      opacity: 0; pointer-events: none;
      z-index: 1000;
      transition: none;
    `;
    document.body.appendChild(overlay);
    return overlay;
  }
  
  async fadeOut(duration = 500, color = 'black') {
    if (this.transitioning) return;
    this.transitioning = true;
    
    this.fadeOverlay.style.background = color;
    this.fadeOverlay.style.pointerEvents = 'all';
    
    return new Promise(resolve => {
      this.fadeOverlay.style.transition = `opacity ${duration}ms ease`;
      this.fadeOverlay.style.opacity = '1';
      setTimeout(resolve, duration);
    });
  }
  
  async fadeIn(duration = 500) {
    return new Promise(resolve => {
      this.fadeOverlay.style.transition = `opacity ${duration}ms ease`;
      this.fadeOverlay.style.opacity = '0';
      setTimeout(() => {
        this.fadeOverlay.style.pointerEvents = 'none';
        this.transitioning = false;
        resolve();
      }, duration);
    });
  }
  
  // Transition ระหว่าง scenes
  async crossFade(fromScene, toScene, camera, duration = 1000) {
    await this.fadeOut(duration / 2);
    
    // Swap scenes (caller does this)
    // ...
    
    await this.fadeIn(duration / 2);
  }
}
```

---

## 4. Portal Transition (Wormhole Effect)

```javascript
class PortalTransition {
  constructor(scene, renderer, camera) {
    this.scene = scene;
    this.renderer = renderer;
    this.camera = camera;
    
    this.portals = [];
    this.renderTarget = null;
  }
  
  createPortal(position, targetScene, options = {}) {
    const portal = {
      position: new THREE.Vector3(...position),
      targetScene,
      mesh: null,
      renderTarget: new THREE.WebGLRenderTarget(512, 512),
      camera: this.camera.clone(),
      active: false,
      ...options
    };
    
    // สร้าง portal mesh
    const geo = new THREE.RingGeometry(0.8, 1.2, 32);
    const mat = new THREE.MeshBasicMaterial({
      color: 0x4488ff,
      side: THREE.DoubleSide,
      transparent: true,
      opacity: 0.8
    });
    
    const frame = new THREE.Mesh(geo, mat);
    
    // Portal screen (แสดง view ของ target scene)
    const screenGeo = new THREE.CircleGeometry(0.8, 32);
    const screenMat = new THREE.MeshBasicMaterial({
      map: portal.renderTarget.texture,
      transparent: false,
      side: THREE.FrontSide
    });
    
    const screen = new THREE.Mesh(screenGeo, screenMat);
    
    const group = new THREE.Group();
    group.add(frame);
    group.add(screen);
    group.position.copy(portal.position);
    
    portal.mesh = group;
    portal.screen = screen;
    
    this.scene.add(group);
    this.portals.push(portal);
    
    return portal;
  }
  
  // Render portal views (ก่อน render main scene)
  renderPortals() {
    for (const portal of this.portals) {
      if (!portal.targetScene) continue;
      
      // คำนวณ camera position สำหรับ portal
      const cameraToPortal = portal.position.clone().sub(this.camera.position);
      portal.camera.position.copy(this.camera.position);
      portal.camera.quaternion.copy(this.camera.quaternion);
      
      // Render target scene ลงบน render target
      this.renderer.setRenderTarget(portal.renderTarget);
      this.renderer.render(portal.targetScene, portal.camera);
      this.renderer.setRenderTarget(null);
    }
  }
  
  // Check ถ้า player ผ่าน portal
  checkPortalCollision(playerPosition) {
    for (const portal of this.portals) {
      const dist = playerPosition.distanceTo(portal.position);
      if (dist < 1.0) {
        return portal;
      }
    }
    return null;
  }
  
  // Animate portal (ripple effect)
  animate(time) {
    this.portals.forEach(portal => {
      if (!portal.mesh) return;
      
      // Ripple the portal
      portal.mesh.rotation.y = Math.sin(time * 0.5) * 0.1;
      
      // Scale pulse
      const scale = 1 + Math.sin(time * 2) * 0.05;
      portal.mesh.scale.setScalar(scale);
    });
  }
}
```

---

## 5. Scene Streaming

```javascript
class SceneStreamer {
  constructor(renderer, camera) {
    this.renderer = renderer;
    this.camera = camera;
    
    this.chunks = new Map();  // chunkId -> { scene, loaded, loading }
    this.loadingQueue = [];
    this.maxConcurrentLoads = 2;
    this.currentLoads = 0;
    
    this.loadDistance = 50;  // โหลด chunks ที่อยู่ภายใน N units
    this.unloadDistance = 80;  // unload chunks ที่ไกลกว่า N units
  }
  
  // กำหนด chunks layout
  defineChunks(chunkDefs) {
    chunkDefs.forEach(def => {
      this.chunks.set(def.id, {
        id: def.id,
        position: new THREE.Vector3(...def.position),
        url: def.url,
        scene: null,
        loaded: false,
        loading: false,
        visible: false
      });
    });
  }
  
  // Update: โหลด/unload chunks ตาม camera position
  async update(cameraPosition) {
    for (const [id, chunk] of this.chunks) {
      const dist = cameraPosition.distanceTo(chunk.position);
      
      if (dist < this.loadDistance && !chunk.loaded && !chunk.loading) {
        // โหลด chunk
        this._loadChunk(chunk);
      } else if (dist > this.unloadDistance && chunk.loaded) {
        // Unload chunk
        this._unloadChunk(chunk);
      }
      
      // Show/hide ตาม distance
      if (chunk.scene) {
        chunk.scene.visible = dist < this.loadDistance;
      }
    }
  }
  
  async _loadChunk(chunk) {
    if (this.currentLoads >= this.maxConcurrentLoads) {
      this.loadingQueue.push(chunk);
      return;
    }
    
    chunk.loading = true;
    this.currentLoads++;
    
    try {
      const loader = new GLTFLoader();
      const gltf = await new Promise((resolve, reject) => {
        loader.load(chunk.url, resolve, undefined, reject);
      });
      
      chunk.scene = gltf.scene;
      chunk.scene.position.copy(chunk.position);
      chunk.loaded = true;
      
      // Add to main scene
      // (caller needs to add to their scene)
      if (this.onChunkLoaded) {
        this.onChunkLoaded(chunk);
      }
    } catch (err) {
      console.error(`Failed to load chunk ${chunk.id}:`, err);
    } finally {
      chunk.loading = false;
      this.currentLoads--;
      
      // Process queue
      if (this.loadingQueue.length > 0) {
        this._loadChunk(this.loadingQueue.shift());
      }
    }
  }
  
  _unloadChunk(chunk) {
    if (!chunk.scene) return;
    
    // Dispose geometry and materials
    chunk.scene.traverse(obj => {
      if (obj.geometry) obj.geometry.dispose();
      if (obj.material) {
        const mats = Array.isArray(obj.material) ? obj.material : [obj.material];
        mats.forEach(m => {
          if (m.map) m.map.dispose();
          m.dispose();
        });
      }
    });
    
    if (this.onChunkUnloaded) {
      this.onChunkUnloaded(chunk);
    }
    
    chunk.scene = null;
    chunk.loaded = false;
  }
}
```

---

## 6. Cinematic Scene Transition System

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title>Cinematic Scene Transitions</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { overflow: hidden; background: #000; font-family: sans-serif; }
    canvas { display: block; }
    
    #overlay {
      position: fixed; inset: 0;
      background: black;
      opacity: 0;
      pointer-events: none;
      z-index: 100;
    }
    
    #warp-overlay {
      position: fixed; inset: 0;
      pointer-events: none;
      z-index: 99;
      display: none;
    }
    
    #ui {
      position: fixed;
      bottom: 20px;
      left: 50%;
      transform: translateX(-50%);
      display: flex;
      gap: 10px;
      z-index: 200;
    }
    
    .scene-btn {
      padding: 10px 20px;
      background: rgba(255,255,255,0.1);
      border: 1px solid rgba(255,255,255,0.3);
      color: white;
      border-radius: 25px;
      cursor: pointer;
      font-size: 13px;
      backdrop-filter: blur(10px);
      transition: all 0.2s;
    }
    .scene-btn:hover {
      background: rgba(255,255,255,0.2);
    }
    .scene-btn.active {
      background: rgba(68,136,255,0.4);
      border-color: rgba(68,136,255,0.8);
    }
    
    #scene-label {
      position: fixed;
      top: 20px;
      left: 50%;
      transform: translateX(-50%);
      color: white;
      font-size: 24px;
      text-align: center;
      z-index: 200;
      text-shadow: 0 0 20px rgba(100,150,255,0.8);
      opacity: 0;
      transition: opacity 0.5s;
    }
  </style>
</head>
<body>
  <canvas id="canvas"></canvas>
  <div id="overlay"></div>
  <canvas id="warp-overlay"></canvas>
  <div id="scene-label" id="sceneLabel">Scene 1</div>
  <div id="ui">
    <button class="scene-btn active" id="scene1Btn">Scene 1: Space</button>
    <button class="scene-btn" id="scene2Btn">Scene 2: Ocean</button>
    <button class="scene-btn" id="scene3Btn">Scene 3: Forest</button>
  </div>
  
  <script type="module">
    import * as THREE from 'https://cdn.skypack.dev/three@0.160.0';
    import { OrbitControls } from 'https://cdn.skypack.dev/three@0.160.0/examples/jsm/controls/OrbitControls.js';
    
    const canvas = document.getElementById('canvas');
    const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.outputColorSpace = THREE.SRGBColorSpace;
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    
    const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 200);
    camera.position.set(0, 2, 8);
    
    const controls = new OrbitControls(camera, canvas);
    controls.enableDamping = true;
    
    // ===== Scene Definitions =====
    
    // Scene 1: Space
    function createSpaceScene() {
      const scene = new THREE.Scene();
      scene.background = new THREE.Color(0x000814);
      
      // Stars
      const starGeo = new THREE.BufferGeometry();
      const starCount = 5000;
      const positions = new Float32Array(starCount * 3);
      for (let i = 0; i < starCount * 3; i++) {
        positions[i] = (Math.random() - 0.5) * 200;
      }
      starGeo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
      const stars = new THREE.Points(starGeo, new THREE.PointsMaterial({ color: 0xffffff, size: 0.1 }));
      scene.add(stars);
      
      // Planet
      const planet = new THREE.Mesh(
        new THREE.SphereGeometry(2, 32, 32),
        new THREE.MeshStandardMaterial({ color: 0x3366cc, roughness: 0.5, metalness: 0.3 })
      );
      scene.add(planet);
      
      // Ring
      const ring = new THREE.Mesh(
        new THREE.RingGeometry(3, 5, 64),
        new THREE.MeshBasicMaterial({ color: 0xaaaa88, transparent: true, opacity: 0.5, side: THREE.DoubleSide })
      );
      ring.rotation.x = Math.PI / 6;
      scene.add(ring);
      
      // Lighting
      scene.add(new THREE.AmbientLight(0x112244, 0.5));
      const sun = new THREE.PointLight(0xffffee, 2, 100);
      sun.position.set(20, 10, 20);
      scene.add(sun);
      
      return { scene, objects: { planet, ring, stars } };
    }
    
    // Scene 2: Ocean
    function createOceanScene() {
      const scene = new THREE.Scene();
      scene.background = new THREE.Color(0x0066aa);
      scene.fog = new THREE.FogExp2(0x0066aa, 0.03);
      
      // Water
      const waterGeo = new THREE.PlaneGeometry(100, 100, 50, 50);
      const waterMat = new THREE.MeshStandardMaterial({
        color: 0x0088cc,
        roughness: 0.1,
        metalness: 0.5,
        transparent: true,
        opacity: 0.9,
        side: THREE.DoubleSide
      });
      const water = new THREE.Mesh(waterGeo, waterMat);
      water.rotation.x = -Math.PI / 2;
      scene.add(water);
      
      // Waves (vertex shader)
      const waterGeo2 = new THREE.PlaneGeometry(100, 100, 100, 100);
      const waterMat2 = new THREE.MeshStandardMaterial({ color: 0x0099cc, roughness: 0.1 });
      const water2 = new THREE.Mesh(waterGeo2, waterMat2);
      water2.rotation.x = -Math.PI / 2;
      water2.position.y = 0.1;
      scene.add(water2);
      
      // Buoys/floating objects
      for (let i = 0; i < 5; i++) {
        const buoy = new THREE.Mesh(
          new THREE.CylinderGeometry(0.2, 0.2, 1, 8),
          new THREE.MeshStandardMaterial({ color: 0xff4400 })
        );
        buoy.position.set((Math.random() - 0.5) * 20, 0.5, (Math.random() - 0.5) * 20);
        scene.add(buoy);
      }
      
      // Sky
      scene.add(new THREE.AmbientLight(0x88aaff, 0.8));
      const sunLight = new THREE.DirectionalLight(0xffffcc, 2);
      sunLight.position.set(5, 10, 5);
      scene.add(sunLight);
      
      return { scene, objects: { water, water2 } };
    }
    
    // Scene 3: Forest
    function createForestScene() {
      const scene = new THREE.Scene();
      scene.background = new THREE.Color(0x2d5a27);
      scene.fog = new THREE.FogExp2(0x2d5a27, 0.03);
      
      // Ground
      const ground = new THREE.Mesh(
        new THREE.PlaneGeometry(60, 60),
        new THREE.MeshStandardMaterial({ color: 0x3a6b35, roughness: 0.9 })
      );
      ground.rotation.x = -Math.PI / 2;
      scene.add(ground);
      
      // Trees
      for (let i = 0; i < 30; i++) {
        const treeGroup = new THREE.Group();
        
        const trunk = new THREE.Mesh(
          new THREE.CylinderGeometry(0.15, 0.2, 3, 6),
          new THREE.MeshStandardMaterial({ color: 0x4a3728 })
        );
        trunk.position.y = 1.5;
        treeGroup.add(trunk);
        
        const layers = 3;
        for (let l = 0; l < layers; l++) {
          const top = new THREE.Mesh(
            new THREE.ConeGeometry(1.5 - l * 0.3, 2, 6),
            new THREE.MeshStandardMaterial({ color: 0x2d6e27 })
          );
          top.position.y = 3 + l * 1.2;
          treeGroup.add(top);
        }
        
        treeGroup.position.set(
          (Math.random() - 0.5) * 50,
          0,
          (Math.random() - 0.5) * 50
        );
        treeGroup.rotation.y = Math.random() * Math.PI * 2;
        
        scene.add(treeGroup);
      }
      
      // Lighting
      scene.add(new THREE.AmbientLight(0x2d5a27, 0.8));
      const sunLight = new THREE.DirectionalLight(0x88ff66, 1.5);
      sunLight.position.set(3, 10, 3);
      scene.add(sunLight);
      
      return { scene, objects: {} };
    }
    
    // สร้าง scenes ทั้งหมด
    const scenes = [
      { name: 'Scene 1: Space', ...createSpaceScene() },
      { name: 'Scene 2: Ocean', ...createOceanScene() },
      { name: 'Scene 3: Forest', ...createForestScene() }
    ];
    
    let currentSceneIndex = 0;
    let transitioning = false;
    
    const overlay = document.getElementById('overlay');
    const sceneLabel = document.getElementById('scene-label');
    
    // Fade transition
    async function transitionToScene(newIndex, transitionType = 'fade') {
      if (transitioning || newIndex === currentSceneIndex) return;
      transitioning = true;
      
      // Update buttons
      document.querySelectorAll('.scene-btn').forEach((b, i) => {
        b.classList.toggle('active', i === newIndex);
      });
      
      // Show scene name
      sceneLabel.textContent = scenes[newIndex].name;
      
      switch (transitionType) {
        case 'fade':
          await fadeTransition(newIndex);
          break;
        case 'warp':
          await warpTransition(newIndex);
          break;
        default:
          await fadeTransition(newIndex);
      }
      
      // Show label briefly
      sceneLabel.style.opacity = '1';
      setTimeout(() => {
        sceneLabel.style.opacity = '0';
      }, 2000);
      
      currentSceneIndex = newIndex;
      transitioning = false;
    }
    
    // Fade transition
    function fadeTransition(newIndex) {
      return new Promise(resolve => {
        overlay.style.transition = 'opacity 0.7s ease';
        overlay.style.opacity = '1';
        
        setTimeout(() => {
          currentSceneIndex = newIndex;
          
          overlay.style.opacity = '0';
          setTimeout(() => {
            resolve();
          }, 700);
        }, 700);
      });
    }
    
    // Warp transition (tunnel effect)
    function warpTransition(newIndex) {
      return new Promise(resolve => {
        const warpCanvas = document.getElementById('warp-overlay');
        warpCanvas.width = window.innerWidth;
        warpCanvas.height = window.innerHeight;
        warpCanvas.style.display = 'block';
        
        const ctx = warpCanvas.getContext('2d');
        let progress = 0;
        
        const animWarp = () => {
          progress += 0.02;
          
          ctx.clearRect(0, 0, warpCanvas.width, warpCanvas.height);
          
          const cx = warpCanvas.width / 2;
          const cy = warpCanvas.height / 2;
          const maxR = Math.sqrt(cx * cx + cy * cy);
          
          // Draw tunnel rings
          for (let r = maxR * progress; r > 0; r -= 20) {
            const opacity = 1 - r / maxR;
            ctx.beginPath();
            ctx.arc(cx, cy, r, 0, Math.PI * 2);
            ctx.strokeStyle = `rgba(68, 136, 255, ${opacity * 0.8})`;
            ctx.lineWidth = 3;
            ctx.stroke();
          }
          
          // Dark overlay grows from center
          const grad = ctx.createRadialGradient(cx, cy, 0, cx, cy, maxR * progress);
          grad.addColorStop(0, 'rgba(0,0,20,0.9)');
          grad.addColorStop(1, 'rgba(0,0,20,0)');
          ctx.fillStyle = grad;
          ctx.fillRect(0, 0, warpCanvas.width, warpCanvas.height);
          
          if (progress < 0.5) {
            requestAnimationFrame(animWarp);
          } else if (progress >= 0.5 && progress < 0.51) {
            // Switch scene at midpoint
            currentSceneIndex = newIndex;
            requestAnimationFrame(animWarp);
          } else if (progress < 1.0) {
            requestAnimationFrame(animWarp);
          } else {
            warpCanvas.style.display = 'none';
            resolve();
          }
        };
        
        animWarp();
      });
    }
    
    // Button handlers
    document.getElementById('scene1Btn').onclick = () => transitionToScene(0, 'fade');
    document.getElementById('scene2Btn').onclick = () => transitionToScene(1, 'warp');
    document.getElementById('scene3Btn').onclick = () => transitionToScene(2, 'fade');
    
    // Resize
    window.addEventListener('resize', () => {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    });
    
    // Animation
    const clock = new THREE.Clock();
    function animate() {
      requestAnimationFrame(animate);
      
      const time = clock.getElapsedTime();
      const delta = clock.getDelta();
      
      // Animate current scene
      const { scene: currentScene, objects } = scenes[currentSceneIndex];
      
      if (objects.planet) {
        objects.planet.rotation.y = time * 0.1;
        objects.ring.rotation.z = time * 0.05;
        objects.stars.rotation.y = time * 0.01;
      }
      
      if (objects.water) {
        const positions = objects.water2.geometry.attributes.position;
        for (let i = 0; i < positions.count; i++) {
          const x = positions.getX(i);
          const z = positions.getZ(i);
          positions.setY(i, Math.sin(x * 0.3 + time) * 0.3 + Math.sin(z * 0.5 + time * 1.3) * 0.2);
        }
        positions.needsUpdate = true;
      }
      
      controls.update();
      renderer.render(currentScene, camera);
    }
    
    animate();
  </script>
</body>
</html>
```

---

## 7. Route-Based 3D Scenes (SPA Integration)

```javascript
// การ integrate กับ SPA router (Vue Router, React Router, etc.)
class ThreeJSRouteManager {
  constructor(renderer) {
    this.renderer = renderer;
    this.scenes = new Map();
    this.currentRoute = null;
    this.transition = new FadeTransition(renderer);
    
    // Listen to browser history
    window.addEventListener('popstate', () => {
      this.handleRoute(window.location.pathname);
    });
  }
  
  // Register route กับ scene
  registerRoute(path, sceneConfig) {
    this.scenes.set(path, sceneConfig);
  }
  
  async navigate(path) {
    if (path === this.currentRoute) return;
    
    // Update URL
    window.history.pushState({}, '', path);
    
    await this.handleRoute(path);
  }
  
  async handleRoute(path) {
    const config = this.scenes.get(path);
    if (!config) return;
    
    // Transition out
    await this.transition.fadeOut(500);
    
    // Load scene if not loaded
    if (!config.scene) {
      config.scene = await config.load();
    }
    
    this.currentRoute = path;
    
    // Transition in
    await this.transition.fadeIn(500);
    
    // Notify
    if (config.onEnter) {
      config.onEnter();
    }
  }
  
  getCurrentScene() {
    const config = this.scenes.get(this.currentRoute);
    return config?.scene;
  }
}

// การใช้งาน
const router = new ThreeJSRouteManager(renderer);

router.registerRoute('/', {
  load: async () => {
    const scene = new THREE.Scene();
    // Setup home scene
    return scene;
  },
  onEnter: () => console.log('Home entered')
});

router.registerRoute('/about', {
  load: async () => {
    const scene = new THREE.Scene();
    // Setup about scene
    return scene;
  }
});

// Navigation
document.querySelectorAll('a[data-3d-route]').forEach(link => {
  link.addEventListener('click', e => {
    e.preventDefault();
    router.navigate(link.href);
  });
});

// Animation loop
function animate() {
  requestAnimationFrame(animate);
  const scene = router.getCurrentScene();
  if (scene) renderer.render(scene, camera);
}
animate();
```

---

## 8. สรุป Scene Transitions

### Pattern ที่แนะนำ:
1. **Fade**: เรียบง่าย เหมาะสำหรับ portfolio/presentation
2. **Warp/Tunnel**: Dramatic เหมาะสำหรับ game/immersive
3. **Crossfade**: Smooth blend ระหว่าง 2 renders
4. **Portal**: Interactive สำหรับ world exploration

### Performance Tips:
```javascript
// 1. Dispose scene resources เมื่อไม่ใช้
function disposeScene(scene) {
  scene.traverse(obj => {
    if (obj.geometry) obj.geometry.dispose();
    if (obj.material) {
      const mats = Array.isArray(obj.material) ? obj.material : [obj.material];
      mats.forEach(m => {
        Object.values(m).forEach(val => {
          if (val && val.isTexture) val.dispose();
        });
        m.dispose();
      });
    }
  });
}

// 2. Keep pool ของ objects ที่ใช้บ่อย
// 3. Preload next scene ระหว่าง transition
// 4. ใช้ loading screen เพื่อซ่อน pop-in
```

ในบทถัดไป (Part 91) เราจะเรียนรู้เกี่ยวกับ Testing Three.js Applications
