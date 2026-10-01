# Part 77: Worker Threads & Performance

## บทนำ

JavaScript เป็น single-threaded แต่ Web Workers ช่วยให้ใช้ CPU หลาย core ได้ การ offload งานหนักไปยัง workers ช่วยให้ render loop ไม่กระตุก บทนี้จะครอบคลุม OffscreenCanvas, SharedArrayBuffer, BVH, และ Worker pools

---

## 77.1 OffscreenCanvas Worker

```javascript
// src/workers/renderer.worker.js
let renderer, scene, camera, animId;

self.onmessage = function(e) {
  const { type, data } = e.data;
  
  if (type === 'init') {
    const { canvas, width, height } = data;
    
    // Dynamic import ใน worker
    importScripts('https://cdn.jsdelivr.net/npm/three@0.158.0/build/three.min.js');
    
    renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setSize(width, height);
    renderer.setPixelRatio(1);
    
    scene = new THREE.Scene();
    scene.background = new THREE.Color('#0a0a1a');
    
    camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 100);
    camera.position.set(0, 0, 5);
    
    // Create objects
    const geo = new THREE.BoxGeometry(1, 1, 1);
    const mat = new THREE.MeshStandardMaterial({ color: '#4ecdc4', metalness: 0.5, roughness: 0.3 });
    const mesh = new THREE.Mesh(geo, mat);
    scene.add(mesh);
    scene.add(new THREE.AmbientLight(0xffffff, 1));
    scene.add(new THREE.DirectionalLight(0xffffff, 1).position.set(5, 5, 5));
    
    let t = 0;
    function animate() {
      animId = requestAnimationFrame(animate);
      t += 0.016;
      mesh.rotation.y = t;
      renderer.render(scene, camera);
    }
    animate();
  }
  
  if (type === 'resize') {
    const { width, height } = data;
    renderer?.setSize(width, height);
    if (camera) {
      camera.aspect = width / height;
      camera.updateProjectionMatrix();
    }
  }
  
  if (type === 'dispose') {
    cancelAnimationFrame(animId);
    renderer?.dispose();
  }
};

// --- Main thread ---
// src/demos/offscreenWorker.js
export function createOffscreenDemo(container) {
  const canvas = document.createElement('canvas');
  canvas.width = container.clientWidth;
  canvas.height = container.clientHeight;
  container.appendChild(canvas);
  
  if (!canvas.transferControlToOffscreen) {
    console.warn('OffscreenCanvas not supported');
    return () => {};
  }
  
  const offscreen = canvas.transferControlToOffscreen();
  
  const worker = new Worker(
    new URL('./renderer.worker.js', import.meta.url),
    { type: 'module' }
  );
  
  worker.postMessage(
    { type: 'init', data: { canvas: offscreen, width: canvas.width, height: canvas.height } },
    [offscreen]
  );
  
  window.addEventListener('resize', () => {
    worker.postMessage({
      type: 'resize',
      data: { width: container.clientWidth, height: container.clientHeight }
    });
  });
  
  return () => {
    worker.postMessage({ type: 'dispose' });
    worker.terminate();
  };
}
```

---

## 77.2 Geometry Computation Worker

```javascript
// src/workers/geometry.worker.js
// Heavy geometry computation ใน worker thread

self.onmessage = function(e) {
  const { type, id, data } = e.data;
  
  if (type === 'generateTerrain') {
    const result = generateTerrain(data);
    self.postMessage({ type: 'result', id, data: result }, [
      result.positions.buffer,
      result.normals.buffer,
      result.indices.buffer,
    ]);
  }
  
  if (type === 'computeNormals') {
    const normals = computeSmoothNormals(data.positions, data.indices);
    self.postMessage({ type: 'result', id, data: normals }, [normals.buffer]);
  }
  
  if (type === 'simplifyMesh') {
    const simplified = simplifyMesh(data.positions, data.indices, data.targetRatio);
    self.postMessage({ type: 'result', id, data: simplified });
  }
};

function generateTerrain({ width, depth, segments, seed }) {
  const w = segments + 1;
  const d = segments + 1;
  const positions = new Float32Array(w * d * 3);
  const normals = new Float32Array(w * d * 3);
  const indices = new Uint32Array(segments * segments * 6);
  
  // Simple noise function
  function noise(x, z, s = seed) {
    const hash = Math.sin(x * 12.9898 + z * 78.233 + s) * 43758.5453;
    return hash - Math.floor(hash);
  }
  
  function fbm(x, z) {
    let val = 0, amp = 1, freq = 1, total = 0;
    for (let i = 0; i < 6; i++) {
      val += noise(x * freq, z * freq) * amp;
      total += amp;
      amp *= 0.5;
      freq *= 2.0;
    }
    return val / total;
  }
  
  // Generate heightmap
  let idx = 0;
  for (let i = 0; i < d; i++) {
    for (let j = 0; j < w; j++) {
      const x = (j / segments - 0.5) * width;
      const z = (i / segments - 0.5) * depth;
      const y = fbm(j / segments * 3, i / segments * 3) * 5 - 1;
      
      positions[idx * 3] = x;
      positions[idx * 3 + 1] = y;
      positions[idx * 3 + 2] = z;
      idx++;
    }
  }
  
  // Generate indices
  let triIdx = 0;
  for (let i = 0; i < segments; i++) {
    for (let j = 0; j < segments; j++) {
      const a = i * w + j;
      const b = i * w + j + 1;
      const c = (i + 1) * w + j;
      const d2 = (i + 1) * w + j + 1;
      
      indices[triIdx++] = a;
      indices[triIdx++] = c;
      indices[triIdx++] = b;
      indices[triIdx++] = b;
      indices[triIdx++] = c;
      indices[triIdx++] = d2;
    }
  }
  
  // Compute normals
  for (let i = 0; i < w * d; i++) {
    normals[i * 3 + 1] = 1; // Default up
  }
  
  return { positions, normals, indices };
}
```

---

## 77.3 Worker Pool

```javascript
// src/utils/workerPool.js
export class WorkerPool {
  constructor(workerUrl, poolSize = 4) {
    this.workers = [];
    this.queue = [];
    this.pendingTasks = new Map();
    this.taskIdCounter = 0;
    
    for (let i = 0; i < poolSize; i++) {
      const worker = new Worker(workerUrl, { type: 'module' });
      worker.idle = true;
      
      worker.onmessage = (e) => {
        const { id, data, error } = e.data;
        worker.idle = true;
        
        const task = this.pendingTasks.get(id);
        if (task) {
          this.pendingTasks.delete(id);
          if (error) task.reject(new Error(error));
          else task.resolve(data);
        }
        
        this._processQueue();
      };
      
      worker.onerror = (e) => {
        console.error('Worker error:', e);
        worker.idle = true;
        this._processQueue();
      };
      
      this.workers.push(worker);
    }
  }
  
  run(type, data, transferables = []) {
    return new Promise((resolve, reject) => {
      const id = this.taskIdCounter++;
      
      this.queue.push({ id, type, data, transferables, resolve, reject });
      this.pendingTasks.set(id, { resolve, reject });
      
      this._processQueue();
    });
  }
  
  _processQueue() {
    if (this.queue.length === 0) return;
    
    const idleWorker = this.workers.find(w => w.idle);
    if (!idleWorker) return;
    
    const task = this.queue.shift();
    idleWorker.idle = false;
    
    idleWorker.postMessage(
      { type: task.type, id: task.id, data: task.data },
      task.transferables
    );
  }
  
  terminate() {
    this.workers.forEach(w => w.terminate());
    this.workers = [];
  }
  
  get stats() {
    return {
      idle: this.workers.filter(w => w.idle).length,
      busy: this.workers.filter(w => !w.idle).length,
      queued: this.queue.length,
    };
  }
}
```

---

## 77.4 three-mesh-bvh สำหรับ Raycast เร็วขึ้น

```javascript
// npm install three-mesh-bvh
import * as THREE from 'three';
import { MeshBVH, acceleratedRaycast, computeBoundsTree, disposeBoundsTree } from 'three-mesh-bvh';

// Extend Three.js geometry prototype
THREE.BufferGeometry.prototype.computeBoundsTree = computeBoundsTree;
THREE.BufferGeometry.prototype.disposeBoundsTree = disposeBoundsTree;
THREE.Mesh.prototype.raycast = acceleratedRaycast;

export function createBVHDemo(container) {
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#0a0a1a');
  
  const camera = new THREE.PerspectiveCamera(45, container.clientWidth / container.clientHeight, 0.1, 1000);
  camera.position.set(0, 5, 20);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  container.appendChild(renderer.domElement);
  
  // สร้าง complex mesh สำหรับ raycast testing
  const OBJECT_COUNT = 1000;
  const meshes = [];
  
  for (let i = 0; i < OBJECT_COUNT; i++) {
    const geo = new THREE.SphereGeometry(0.3, 16, 16);
    
    // BVH computation (ทำครั้งเดียว)
    geo.computeBoundsTree();
    
    const mesh = new THREE.Mesh(
      geo,
      new THREE.MeshStandardMaterial({ color: '#4ecdc4' })
    );
    
    mesh.position.set(
      (Math.random() - 0.5) * 30,
      (Math.random() - 0.5) * 15,
      (Math.random() - 0.5) * 30
    );
    
    scene.add(mesh);
    meshes.push(mesh);
  }
  
  // Raycaster
  const raycaster = new THREE.Raycaster();
  const mouse = new THREE.Vector2();
  
  // Stats
  const statsDiv = document.createElement('div');
  statsDiv.style.cssText = `
    position: absolute; top: 10px; left: 10px;
    background: rgba(0,0,0,0.7); color: #4ecdc4;
    padding: 10px; border-radius: 8px; font-family: monospace;
    font-size: 12px; z-index: 10;
  `;
  container.style.position = 'relative';
  container.appendChild(statsDiv);
  
  container.addEventListener('mousemove', (e) => {
    const rect = container.getBoundingClientRect();
    mouse.x = ((e.clientX - rect.left) / rect.width) * 2 - 1;
    mouse.y = -((e.clientY - rect.top) / rect.height) * 2 + 1;
  });
  
  let lastColor = null;
  
  // Lighting
  scene.add(new THREE.AmbientLight(0xffffff, 0.5));
  scene.add(new THREE.DirectionalLight(0xffffff, 1).position.set(10, 10, 10));
  
  const clock = new THREE.Clock();
  let animId;
  let frameCount = 0;
  let lastFPSTime = 0;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    const delta = clock.getDelta();
    
    frameCount++;
    if (elapsed - lastFPSTime > 1) {
      const fps = (frameCount / (elapsed - lastFPSTime)).toFixed(0);
      lastFPSTime = elapsed;
      frameCount = 0;
      statsDiv.textContent = `FPS: ${fps}\nObjects: ${OBJECT_COUNT}\nBVH: ON`;
    }
    
    // Rotate objects slowly
    meshes.forEach((mesh, i) => {
      mesh.rotation.y += delta * 0.3;
    });
    
    // Raycast
    raycaster.setFromCamera(mouse, camera);
    const t0 = performance.now();
    const intersects = raycaster.intersectObjects(meshes);
    const raycastTime = (performance.now() - t0).toFixed(2);
    
    // Highlight hovered
    if (lastColor) {
      lastColor.mesh.material.color.set('#4ecdc4');
      lastColor.mesh.material.emissiveIntensity = 0;
    }
    
    if (intersects.length > 0) {
      const hit = intersects[0].object;
      hit.material.color.set('#ff6b6b');
      hit.material.emissive = new THREE.Color('#ff6b6b');
      hit.material.emissiveIntensity = 0.5;
      lastColor = { mesh: hit };
    }
    
    renderer.render(scene, camera);
  }
  
  animate();
  
  return () => {
    cancelAnimationFrame(animId);
    meshes.forEach(m => {
      m.geometry.disposeBoundsTree();
      m.geometry.dispose();
      m.material.dispose();
    });
    renderer.dispose();
    container.removeChild(statsDiv);
  };
}
```

---

## 77.5 SharedArrayBuffer สำหรับ Particles

```javascript
// src/workers/particles.worker.js
// Worker ที่อัพเดต particle positions ผ่าน SharedArrayBuffer

let positions, velocities, running = false;

self.onmessage = function(e) {
  const { type, data } = e.data;
  
  if (type === 'init') {
    positions = new Float32Array(data.positionBuffer);
    velocities = new Float32Array(data.velocityBuffer);
    running = true;
    updateLoop();
  }
  
  if (type === 'stop') {
    running = false;
  }
};

function updateLoop() {
  if (!running) return;
  
  const count = positions.length / 3;
  const GRAVITY = -0.001;
  const BOUNDS = 10;
  
  for (let i = 0; i < count; i++) {
    const pi = i * 3;
    
    // Apply gravity
    velocities[pi + 1] += GRAVITY;
    
    // Update position
    positions[pi] += velocities[pi];
    positions[pi + 1] += velocities[pi + 1];
    positions[pi + 2] += velocities[pi + 2];
    
    // Bounce off bounds
    if (Math.abs(positions[pi]) > BOUNDS) velocities[pi] *= -0.9;
    if (positions[pi + 1] < -BOUNDS) {
      positions[pi + 1] = -BOUNDS;
      velocities[pi + 1] = Math.abs(velocities[pi + 1]) * 0.8;
    }
    if (Math.abs(positions[pi + 2]) > BOUNDS) velocities[pi + 2] *= -0.9;
  }
  
  // Post update notification (no data, just signal)
  self.postMessage({ type: 'updated' });
  
  setTimeout(updateLoop, 16); // ~60fps
}

// --- Main Thread ---
// src/demos/sharedBufferDemo.js
import * as THREE from 'three';

export function createSharedBufferDemo(container) {
  if (!crossOriginIsolated) {
    // SharedArrayBuffer ต้องการ COOP/COEP headers
    const warning = document.createElement('div');
    warning.textContent = 'SharedArrayBuffer requires COOP/COEP headers. Running without worker.';
    warning.style.cssText = `
      position: absolute; top: 10px; left: 10px;
      background: rgba(255,165,0,0.8); color: white;
      padding: 10px; border-radius: 8px; font-family: monospace;
      font-size: 12px; z-index: 10;
    `;
    container.style.position = 'relative';
    container.appendChild(warning);
  }
  
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#050510');
  
  const camera = new THREE.PerspectiveCamera(60, container.clientWidth / container.clientHeight, 0.1, 100);
  camera.position.set(0, 5, 20);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  container.appendChild(renderer.domElement);
  
  const PARTICLE_COUNT = 50000;
  
  // ใช้ SharedArrayBuffer ถ้ามี
  let positionBuffer, velocityBuffer;
  let useShared = crossOriginIsolated;
  
  if (useShared) {
    positionBuffer = new SharedArrayBuffer(PARTICLE_COUNT * 3 * 4);
    velocityBuffer = new SharedArrayBuffer(PARTICLE_COUNT * 3 * 4);
  } else {
    positionBuffer = new ArrayBuffer(PARTICLE_COUNT * 3 * 4);
    velocityBuffer = new ArrayBuffer(PARTICLE_COUNT * 3 * 4);
  }
  
  const positions = new Float32Array(positionBuffer);
  const velocities = new Float32Array(velocityBuffer);
  
  // Initialize
  for (let i = 0; i < PARTICLE_COUNT; i++) {
    positions[i * 3] = (Math.random() - 0.5) * 20;
    positions[i * 3 + 1] = Math.random() * 15 + 2;
    positions[i * 3 + 2] = (Math.random() - 0.5) * 20;
    
    velocities[i * 3] = (Math.random() - 0.5) * 0.1;
    velocities[i * 3 + 1] = -Math.random() * 0.05;
    velocities[i * 3 + 2] = (Math.random() - 0.5) * 0.1;
  }
  
  // Particle geometry
  const geometry = new THREE.BufferGeometry();
  geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
  
  const particles = new THREE.Points(
    geometry,
    new THREE.PointsMaterial({
      size: 0.05,
      color: '#4ecdc4',
      transparent: true,
      opacity: 0.6,
      sizeAttenuation: true,
    })
  );
  scene.add(particles);
  
  // Worker (ถ้ามี SharedArrayBuffer)
  let worker = null;
  if (useShared) {
    const workerCode = `
      let positions, velocities, running = false;
      self.onmessage = function(e) {
        if (e.data.type === 'init') {
          positions = new Float32Array(e.data.positionBuffer);
          velocities = new Float32Array(e.data.velocityBuffer);
          running = true;
          loop();
        }
        if (e.data.type === 'stop') running = false;
      };
      function loop() {
        if (!running) return;
        const count = positions.length / 3;
        for (let i = 0; i < count; i++) {
          const p = i * 3;
          velocities[p+1] -= 0.001;
          positions[p] += velocities[p];
          positions[p+1] += velocities[p+1];
          positions[p+2] += velocities[p+2];
          if (Math.abs(positions[p]) > 10) velocities[p] *= -0.9;
          if (positions[p+1] < -5) { positions[p+1] = -5; velocities[p+1] = Math.abs(velocities[p+1]) * 0.8; }
          if (Math.abs(positions[p+2]) > 10) velocities[p+2] *= -0.9;
        }
        setTimeout(loop, 16);
      }
    `;
    
    const blob = new Blob([workerCode], { type: 'application/javascript' });
    worker = new Worker(URL.createObjectURL(blob));
    worker.postMessage({ type: 'init', positionBuffer, velocityBuffer });
  }
  
  // Lighting
  scene.add(new THREE.AmbientLight(0xffffff, 0.3));
  
  const clock = new THREE.Clock();
  let animId;
  const GRAVITY = -0.001;
  const BOUNDS = 10;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    
    // Update particles in main thread if no worker
    if (!useShared) {
      for (let i = 0; i < PARTICLE_COUNT; i++) {
        const p = i * 3;
        velocities[p + 1] += GRAVITY;
        positions[p] += velocities[p];
        positions[p + 1] += velocities[p + 1];
        positions[p + 2] += velocities[p + 2];
        
        if (Math.abs(positions[p]) > BOUNDS) velocities[p] *= -0.9;
        if (positions[p + 1] < -BOUNDS) {
          positions[p + 1] = -BOUNDS;
          velocities[p + 1] = Math.abs(velocities[p + 1]) * 0.8;
        }
        if (Math.abs(positions[p + 2]) > BOUNDS) velocities[p + 2] *= -0.9;
      }
    }
    
    geometry.attributes.position.needsUpdate = true;
    
    const t = clock.getElapsedTime();
    camera.position.x = Math.sin(t * 0.1) * 20;
    camera.position.z = Math.cos(t * 0.1) * 20;
    camera.lookAt(0, 0, 0);
    
    renderer.render(scene, camera);
  }
  
  animate();
  
  return () => {
    cancelAnimationFrame(animId);
    worker?.postMessage({ type: 'stop' });
    worker?.terminate();
    geometry.dispose();
    renderer.dispose();
  };
}
```

---

## 77.6 Performance Monitoring

```javascript
// src/utils/perfMonitor.js
export class PerformanceMonitor {
  constructor(container) {
    this.frames = [];
    this.maxFrames = 120;
    
    this.panel = document.createElement('div');
    this.panel.style.cssText = `
      position: absolute; top: 10px; right: 10px;
      background: rgba(0,0,0,0.85);
      border: 1px solid #333;
      border-radius: 8px;
      padding: 10px 15px;
      font-family: 'Courier New', monospace;
      font-size: 11px;
      color: #4ecdc4;
      min-width: 150px;
      z-index: 100;
    `;
    
    container.style.position = 'relative';
    container.appendChild(this.panel);
    
    this.lastTime = performance.now();
    this.frameCount = 0;
    this.fps = 0;
    this.fpsHistory = new Array(60).fill(60);
    
    // Canvas for mini graph
    this.graph = document.createElement('canvas');
    this.graph.width = 120;
    this.graph.height = 40;
    this.graph.style.cssText = 'display: block; margin-top: 8px;';
    this.panel.appendChild(this.graph);
    this.graphCtx = this.graph.getContext('2d');
  }
  
  update(renderer) {
    this.frameCount++;
    const now = performance.now();
    const delta = now - this.lastTime;
    
    if (delta >= 1000) {
      this.fps = (this.frameCount / delta) * 1000;
      this.frameCount = 0;
      this.lastTime = now;
      
      this.fpsHistory.push(this.fps);
      this.fpsHistory.shift();
    }
    
    const info = renderer?.info;
    
    this.panel.innerHTML = `
      <div style="color: ${this.fps < 30 ? '#ff6b6b' : this.fps < 50 ? '#ffd166' : '#4ecdc4'}">
        FPS: ${this.fps.toFixed(0)}
      </div>
      <div style="color: #888">
        Draw: ${info?.render.calls || 0}<br>
        Tri: ${((info?.render.triangles || 0) / 1000).toFixed(1)}K<br>
        Geo: ${info?.memory.geometries || 0}<br>
        Tex: ${info?.memory.textures || 0}
      </div>
    `;
    
    this.panel.appendChild(this.graph);
    
    // Draw FPS graph
    const ctx = this.graphCtx;
    ctx.clearRect(0, 0, 120, 40);
    ctx.fillStyle = 'rgba(0,0,0,0.5)';
    ctx.fillRect(0, 0, 120, 40);
    
    ctx.strokeStyle = '#4ecdc4';
    ctx.lineWidth = 1;
    ctx.beginPath();
    
    this.fpsHistory.forEach((fps, i) => {
      const x = i * (120 / this.fpsHistory.length);
      const y = 40 - (fps / 60) * 38;
      if (i === 0) ctx.moveTo(x, y);
      else ctx.lineTo(x, y);
    });
    
    ctx.stroke();
    
    // Target line at 60fps
    ctx.strokeStyle = 'rgba(255,255,255,0.2)';
    ctx.beginPath();
    ctx.moveTo(0, 1);
    ctx.lineTo(120, 1);
    ctx.stroke();
  }
  
  remove() {
    this.panel.remove();
  }
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Terrain Generator Worker
สร้าง terrain generator ที่:
- Perlin noise ใน worker
- Chunk-based สำหรับ streaming
- LOD per chunk
- Real-time editing

### แบบฝึกหัดที่ 2: Physics Worker
สร้าง physics simulation ที่:
- Simple particle physics ใน worker
- SharedArrayBuffer transfer
- 100K+ particles
- Collision detection

### แบบฝึกหัดที่ 3: Asset Processing Pipeline
สร้าง pipeline ที่:
- Load GLTF ใน worker
- Process/optimize geometry
- Transfer back to main thread
- Progress reporting

### แบบฝึกหัดที่ 4: Performance Dashboard
สร้าง dashboard ที่:
- FPS, frame time graphs
- Memory usage
- GPU metrics
- Alert เมื่อ performance ตก

---

## สรุป

ในบทนี้เราเรียนรู้:
1. **OffscreenCanvas** - Render ใน worker thread
2. **Geometry Workers** - Heavy computation offload
3. **Worker Pool** - จัดการหลาย workers
4. **three-mesh-bvh** - Fast raycast
5. **SharedArrayBuffer** - Zero-copy data sharing
6. **Performance Monitor** - Real-time metrics

บทถัดไปเราจะเรียน **Mobile Optimization**
