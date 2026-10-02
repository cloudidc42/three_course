# Part 106: Worker Threads & OffscreenCanvas (Advanced)

## บทนำ: JavaScript Single Thread Problem

JavaScript ทำงานบน Single Thread หมายความว่า ถ้า Main Thread ยุ่งกับการคำนวณ เช่น BVH construction, physics, geometry generation ก็จะทำให้ animation ติดขัด (jank) ได้

```
Single Thread (ปัญหา):
Frame 1: [Physics] [Geometry] [Render] → 33ms = 30fps 😱
Frame 2: [Physics] [Geometry] [Render] → 33ms = 30fps

Multi Thread (แก้ปัญหา):
Main:    [Render]  [Render]  [Render]  → 16ms = 60fps 😊
Worker:  [Physics calc] [Geometry gen]
```

---

## 1. Web Workers: ประเภทและการใช้งาน

### Dedicated Worker (ใช้บ่อยที่สุด)
```javascript
// worker.js - ทำงานบน thread ต่างหาก
self.onmessage = (event) => {
    const { type, data } = event.data;
    
    if (type === 'COMPUTE') {
        // ทำงานหนักๆ
        const result = heavyComputation(data);
        
        // ส่งผลลัพธ์กลับ
        self.postMessage({ type: 'RESULT', data: result });
    }
};

function heavyComputation(data) {
    let sum = 0;
    for (let i = 0; i < 1e8; i++) sum += Math.sin(i * data);
    return sum;
}

// main.js - สร้างและสื่อสารกับ worker
const worker = new Worker('worker.js');

worker.onmessage = (event) => {
    console.log('Result:', event.data);
};

worker.postMessage({ type: 'COMPUTE', data: 0.001 });
```

### Inline Worker (ไม่ต้องมีไฟล์แยก)
```javascript
const workerCode = `
self.onmessage = (e) => {
    const result = e.data * 2;
    self.postMessage(result);
};
`;

const blob = new Blob([workerCode], { type: 'application/javascript' });
const worker = new Worker(URL.createObjectURL(blob));
```

### Module Worker (รองรับ import)
```javascript
// worker.module.js
import { computeSomething } from './compute-module.js';

self.onmessage = async (e) => {
    const result = await computeSomething(e.data);
    self.postMessage(result);
};

// main.js
const worker = new Worker('worker.module.js', { type: 'module' });
```

---

## 2. OffscreenCanvas: Three.js ใน Worker Thread

`OffscreenCanvas` ให้ render 3D graphics ใน worker thread ได้ โดยไม่บล็อก main thread:

```javascript
// main.js
const canvas = document.getElementById('c');

// โอน control ของ canvas ไปให้ worker
const offscreen = canvas.transferControlToOffscreen();

const worker = new Worker('render-worker.js', { type: 'module' });

// ส่ง canvas ไป worker (ต้อง transfer ownership)
worker.postMessage(
    { type: 'INIT', canvas: offscreen, width: canvas.width, height: canvas.height },
    [offscreen] // transferable objects - ย้ายความเป็นเจ้าของ
);

// ส่ง events ไปให้ worker
window.addEventListener('resize', () => {
    worker.postMessage({ 
        type: 'RESIZE', 
        width: window.innerWidth, 
        height: window.innerHeight 
    });
});

document.addEventListener('mousemove', (e) => {
    worker.postMessage({ 
        type: 'MOUSEMOVE', 
        x: e.clientX, 
        y: e.clientY 
    });
});
```

```javascript
// render-worker.js
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.165.0/build/three.module.js';

let renderer, scene, camera, controls;

self.onmessage = (e) => {
    const { type, data } = e;
    
    switch (e.data.type) {
        case 'INIT':
            init(e.data.canvas, e.data.width, e.data.height);
            break;
        case 'RESIZE':
            handleResize(e.data.width, e.data.height);
            break;
        case 'MOUSEMOVE':
            handleMouseMove(e.data.x, e.data.y);
            break;
    }
};

function init(canvas, width, height) {
    renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setSize(width, height);
    
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x0a0a1a);
    
    camera = new THREE.PerspectiveCamera(60, width / height, 0.1, 1000);
    camera.position.set(0, 5, 15);
    camera.lookAt(0, 0, 0);
    
    scene.add(new THREE.AmbientLight(0xffffff, 0.5));
    const light = new THREE.DirectionalLight(0xffffff, 1.5);
    light.position.set(10, 20, 10);
    scene.add(light);
    
    // สร้าง scene content
    for (let i = 0; i < 100; i++) {
        const geo = new THREE.BoxGeometry(0.5, 0.5, 0.5);
        const mat = new THREE.MeshStandardMaterial({ 
            color: new THREE.Color().setHSL(Math.random(), 0.8, 0.5) 
        });
        const mesh = new THREE.Mesh(geo, mat);
        mesh.position.set(
            (Math.random() - 0.5) * 20,
            (Math.random() - 0.5) * 10,
            (Math.random() - 0.5) * 20
        );
        scene.add(mesh);
    }
    
    animate();
}

function handleResize(width, height) {
    if (!renderer) return;
    renderer.setSize(width, height);
    camera.aspect = width / height;
    camera.updateProjectionMatrix();
}

let mouseX = 0, mouseY = 0;
function handleMouseMove(x, y) {
    mouseX = x;
    mouseY = y;
}

let time = 0;
function animate() {
    // OffscreenCanvas ใช้ requestAnimationFrame ผ่าน setInterval หรือ setTimeout ใน worker
    // หรือใช้ self.requestAnimationFrame ถ้า browser รองรับ
    
    requestAnimationFrame ? requestAnimationFrame(animate) : setTimeout(animate, 16);
    
    time += 0.01;
    
    // Rotate objects
    scene.traverse(obj => {
        if (obj.isMesh) {
            obj.rotation.x += 0.005;
            obj.rotation.y += 0.008;
        }
    });
    
    // Orbit camera based on mouse
    camera.position.x = Math.sin(time * 0.5) * 20;
    camera.position.z = Math.cos(time * 0.5) * 20;
    camera.lookAt(0, 0, 0);
    
    renderer.render(scene, camera);
}
```

---

## 3. Comlink: RPC สำหรับ Workers

Comlink ทำให้การสื่อสารกับ Worker เหมือน async function calls:

```javascript
// worker-api.js
import { expose } from 'https://cdn.jsdelivr.net/npm/comlink@4.4.1/dist/esm/comlink.mjs';

const api = {
    // Synchronous computation
    generateTerrain(width, depth, scale) {
        const data = new Float32Array(width * depth);
        for (let z = 0; z < depth; z++) {
            for (let x = 0; x < width; x++) {
                const i = z * width + x;
                data[i] = Math.sin(x * scale) * Math.cos(z * scale) * 10;
            }
        }
        return data;
    },
    
    // Async computation with progress
    async buildBVH(geometry, onProgress) {
        // Simulate BVH construction
        const totalSteps = 100;
        for (let i = 0; i < totalSteps; i++) {
            await new Promise(r => setTimeout(r, 10)); // simulate work
            await onProgress(i / totalSteps);
        }
        return { done: true, nodes: 1024 };
    }
};

expose(api);

// main.js
import { wrap } from 'https://cdn.jsdelivr.net/npm/comlink@4.4.1/dist/esm/comlink.mjs';

const worker = new Worker('worker-api.js', { type: 'module' });
const api = wrap(worker);

// ใช้งานแบบ async/await
async function main() {
    // Simple call
    const terrainData = await api.generateTerrain(256, 256, 0.05);
    console.log('Terrain generated:', terrainData.length, 'points');
    
    // Call with progress callback
    const bvh = await api.buildBVH(
        geometry,
        proxy((progress) => {
            console.log(`BVH: ${(progress * 100).toFixed(0)}%`);
        })
    );
}
```

---

## 4. SharedArrayBuffer + Atomics (Zero-Copy Data Sharing)

```javascript
// SharedArrayBuffer ให้ share memory ระหว่าง threads โดยไม่ต้อง copy data
// ต้องการ COOP/COEP headers:
// Cross-Origin-Opener-Policy: same-origin
// Cross-Origin-Embedder-Policy: require-corp

// Physics Worker ที่ share position data กับ main thread
// physics-worker.js
let sharedPositions;
let objectCount;

self.onmessage = (e) => {
    if (e.data.type === 'INIT') {
        // รับ SharedArrayBuffer
        sharedPositions = new Float32Array(e.data.buffer);
        objectCount = e.data.count;
        runPhysics();
    }
};

function runPhysics() {
    // Update positions ใน shared buffer
    for (let i = 0; i < objectCount; i++) {
        const base = i * 3; // x, y, z
        
        // Simple gravity simulation
        sharedPositions[base + 1] -= 0.01; // y -= gravity
        
        // Bounce on ground
        if (sharedPositions[base + 1] < 0) {
            sharedPositions[base + 1] = 0;
        }
    }
    
    setTimeout(runPhysics, 16); // 60fps physics
}

// main.js
const OBJECT_COUNT = 1000;
const sharedBuffer = new SharedArrayBuffer(OBJECT_COUNT * 3 * 4); // float32 * 3 * count
const sharedPositions = new Float32Array(sharedBuffer);

// Initial positions
for (let i = 0; i < OBJECT_COUNT; i++) {
    const base = i * 3;
    sharedPositions[base] = (Math.random() - 0.5) * 50;      // x
    sharedPositions[base + 1] = Math.random() * 20 + 5;       // y
    sharedPositions[base + 2] = (Math.random() - 0.5) * 50;   // z
}

const worker = new Worker('physics-worker.js');
worker.postMessage({ 
    type: 'INIT', 
    buffer: sharedBuffer,
    count: OBJECT_COUNT 
});

// Main thread reads from shared buffer every frame
function animate() {
    requestAnimationFrame(animate);
    
    // Read positions โดยตรงจาก shared buffer (NO COPY!)
    instancedMesh.instanceMatrix.needsUpdate = true;
    
    const matrix = new THREE.Matrix4();
    for (let i = 0; i < OBJECT_COUNT; i++) {
        const base = i * 3;
        matrix.setPosition(
            sharedPositions[base],
            sharedPositions[base + 1],
            sharedPositions[base + 2]
        );
        instancedMesh.setMatrixAt(i, matrix);
    }
    
    renderer.render(scene, camera);
}
```

---

## 5. Worker Pool Pattern

```javascript
class WorkerPool {
    constructor(workerUrl, poolSize = navigator.hardwareConcurrency || 4) {
        this.poolSize = poolSize;
        this.workers = [];
        this.queue = [];
        this.available = [];
        
        // สร้าง workers
        for (let i = 0; i < poolSize; i++) {
            const worker = new Worker(workerUrl, { type: 'module' });
            worker.onmessage = (e) => this._onWorkerMessage(i, e);
            worker.onerror = (e) => this._onWorkerError(i, e);
            this.workers.push(worker);
            this.available.push(i);
        }
        
        this.pending = new Map(); // workerId -> { resolve, reject }
    }
    
    // ส่งงานไป worker pool
    async exec(data, transferables = []) {
        return new Promise((resolve, reject) => {
            const job = { data, transferables, resolve, reject };
            
            if (this.available.length > 0) {
                this._dispatch(this.available.shift(), job);
            } else {
                this.queue.push(job);
            }
        });
    }
    
    _dispatch(workerId, job) {
        this.pending.set(workerId, { resolve: job.resolve, reject: job.reject });
        this.workers[workerId].postMessage(job.data, job.transferables);
    }
    
    _onWorkerMessage(workerId, event) {
        const { resolve } = this.pending.get(workerId);
        this.pending.delete(workerId);
        resolve(event.data);
        
        // Process next job
        if (this.queue.length > 0) {
            this._dispatch(workerId, this.queue.shift());
        } else {
            this.available.push(workerId);
        }
    }
    
    _onWorkerError(workerId, error) {
        const pending = this.pending.get(workerId);
        if (pending) {
            pending.reject(error);
            this.pending.delete(workerId);
        }
        
        if (this.queue.length > 0) {
            this._dispatch(workerId, this.queue.shift());
        } else {
            this.available.push(workerId);
        }
    }
    
    // ยกเลิก workers ทั้งหมด
    terminate() {
        this.workers.forEach(w => w.terminate());
        this.workers = [];
        this.queue = [];
        this.available = [];
    }
    
    // สถานะ pool
    get stats() {
        return {
            total: this.poolSize,
            available: this.available.length,
            busy: this.poolSize - this.available.length,
            queued: this.queue.length
        };
    }
}

// ใช้งาน Worker Pool
const pool = new WorkerPool('geometry-worker.js', 4);

// ส่งงาน parallel
async function generateMeshes(count) {
    const promises = [];
    
    for (let i = 0; i < count; i++) {
        promises.push(pool.exec({ 
            type: 'GENERATE_TERRAIN',
            seed: i,
            resolution: 64
        }));
    }
    
    // รอผลลัพธ์ทั้งหมด
    const results = await Promise.all(promises);
    console.log(`Generated ${results.length} meshes in parallel!`);
    return results;
}
```

---

## 6. Complete OffscreenCanvas Three.js Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>OffscreenCanvas Three.js Demo</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; color: #eee; font-family: monospace; overflow: hidden; }
        canvas { display: block; width: 100vw; height: 100vh; }
        #ui {
            position: fixed; top: 20px; right: 20px;
            background: rgba(0,0,0,0.85);
            padding: 20px; border-radius: 8px;
            border: 1px solid #0af; min-width: 280px;
            z-index: 10; pointer-events: none;
        }
        h2 { color: #0af; margin-bottom: 10px; }
        .row { display: flex; justify-content: space-between; margin: 5px 0; font-size: 13px; }
        .val { color: #4f4; font-weight: bold; }
        #log { 
            margin-top: 10px; font-size: 11px; color: #888;
            max-height: 100px; overflow-y: auto;
            padding: 5px; background: rgba(255,255,255,0.03);
            border-radius: 4px;
        }
        #thread-indicator {
            margin-top: 8px; padding: 6px 12px;
            border-radius: 4px; font-size: 12px; text-align: center;
        }
        .worker-thread { background: rgba(0,255,100,0.2); border: 1px solid #0f4; color: #0f4; }
        .main-thread { background: rgba(255,100,0,0.2); border: 1px solid #f40; color: #f40; }
    </style>
</head>
<body>
<canvas id="c"></canvas>
<div id="ui">
    <h2>🧵 Worker Thread Demo</h2>
    <div id="thread-indicator" class="main-thread">Loading...</div>
    
    <div class="row"><span>Render Thread:</span><span id="m-thread" class="val">-</span></div>
    <div class="row"><span>Main Thread FPS:</span><span id="m-fps" class="val">-</span></div>
    <div class="row"><span>Worker Message:</span><span id="m-worker" class="val">-</span></div>
    <div class="row"><span>Objects:</span><span id="m-objects" class="val">-</span></div>
    <div class="row"><span>OffscreenCanvas:</span><span id="m-offscreen" class="val">-</span></div>
    
    <div id="log">Initializing...</div>
</div>

<script>
    const logEl = document.getElementById('log');
    const logs = [];
    
    function log(msg) {
        logs.push(`[${new Date().toLocaleTimeString()}] ${msg}`);
        if (logs.length > 8) logs.shift();
        logEl.textContent = logs.join('\n');
    }
    
    function updateUI(data) {
        document.getElementById('m-thread').textContent = data.thread;
        document.getElementById('m-worker').textContent = data.workerFps + ' fps';
        document.getElementById('m-objects').textContent = data.objects;
        document.getElementById('m-offscreen').textContent = 
            typeof OffscreenCanvas !== 'undefined' ? 'รองรับ ✅' : 'ไม่รองรับ ❌';
    }
    
    const canvas = document.getElementById('c');
    const offscreenSupported = typeof OffscreenCanvas !== 'undefined' && 
                               typeof canvas.transferControlToOffscreen === 'function';
    
    if (offscreenSupported) {
        document.getElementById('thread-indicator').textContent = '✅ Rendering ใน Worker Thread';
        document.getElementById('thread-indicator').className = 'worker-thread';
        log('OffscreenCanvas supported! Starting worker...');
        
        // Worker code as blob
        const workerCode = `
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.165.0/build/three.module.js';

let renderer, scene, camera;
let frameCount = 0, lastFpsTime = 0, fps = 0;
const OBJECT_COUNT = 500;

self.onmessage = (e) => {
    const msg = e.data;
    switch (msg.type) {
        case 'INIT':
            init(msg.canvas, msg.width, msg.height);
            break;
        case 'RESIZE':
            if (renderer) {
                renderer.setSize(msg.width, msg.height);
                camera.aspect = msg.width / msg.height;
                camera.updateProjectionMatrix();
            }
            break;
    }
};

function init(canvas, width, height) {
    renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setSize(width, height);
    renderer.setPixelRatio(Math.min(self.devicePixelRatio || 1, 2));
    
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x050515);
    scene.fog = new THREE.FogExp2(0x050515, 0.02);
    
    camera = new THREE.PerspectiveCamera(60, width / height, 0.1, 500);
    camera.position.set(0, 0, 30);
    
    // Lights
    scene.add(new THREE.AmbientLight(0x334466, 0.8));
    const light1 = new THREE.DirectionalLight(0xffffff, 1.5);
    light1.position.set(10, 20, 10);
    scene.add(light1);
    const light2 = new THREE.PointLight(0x4488ff, 2, 50);
    light2.position.set(-15, 5, 10);
    scene.add(light2);
    
    // Create objects
    const geo = new THREE.IcosahedronGeometry(0.5, 1);
    const mat = new THREE.MeshStandardMaterial({ metalness: 0.3, roughness: 0.5 });
    const mesh = new THREE.InstancedMesh(geo, mat, OBJECT_COUNT);
    
    const matrix = new THREE.Matrix4();
    const color = new THREE.Color();
    const positions = [];
    const velocities = [];
    
    for (let i = 0; i < OBJECT_COUNT; i++) {
        const px = (Math.random() - 0.5) * 60;
        const py = (Math.random() - 0.5) * 40;
        const pz = (Math.random() - 0.5) * 60;
        positions.push(px, py, pz);
        velocities.push(
            (Math.random() - 0.5) * 0.1,
            (Math.random() - 0.5) * 0.1,
            (Math.random() - 0.5) * 0.1
        );
        
        matrix.setPosition(px, py, pz);
        mesh.setMatrixAt(i, matrix);
        
        color.setHSL(i / OBJECT_COUNT, 0.8, 0.5);
        mesh.setColorAt(i, color);
    }
    mesh.instanceMatrix.needsUpdate = true;
    if (mesh.instanceColor) mesh.instanceColor.needsUpdate = true;
    
    scene.add(mesh);
    
    // Animate
    function loop(time) {
        requestAnimationFrame(loop);
        
        // Update physics
        for (let i = 0; i < OBJECT_COUNT; i++) {
            const base = i * 3;
            positions[base] += velocities[base];
            positions[base + 1] += velocities[base + 1];
            positions[base + 2] += velocities[base + 2];
            
            // Bounce off walls
            if (Math.abs(positions[base]) > 30) velocities[base] *= -1;
            if (Math.abs(positions[base + 1]) > 20) velocities[base + 1] *= -1;
            if (Math.abs(positions[base + 2]) > 30) velocities[base + 2] *= -1;
            
            matrix.setPosition(positions[base], positions[base + 1], positions[base + 2]);
            mesh.setMatrixAt(i, matrix);
        }
        mesh.instanceMatrix.needsUpdate = true;
        
        // Rotate camera
        camera.position.x = Math.sin(time * 0.0003) * 40;
        camera.position.z = Math.cos(time * 0.0003) * 40;
        camera.position.y = Math.sin(time * 0.0002) * 20;
        camera.lookAt(0, 0, 0);
        
        renderer.render(scene, camera);
        
        // FPS
        frameCount++;
        if (time - lastFpsTime >= 1000) {
            fps = frameCount;
            frameCount = 0;
            lastFpsTime = time;
            
            self.postMessage({ 
                type: 'STATS',
                workerFps: fps,
                objects: OBJECT_COUNT,
                thread: 'Worker Thread'
            });
        }
    }
    
    requestAnimationFrame(loop);
    self.postMessage({ type: 'READY' });
}
`;
        
        const blob = new Blob([workerCode], { type: 'application/javascript' });
        const workerURL = URL.createObjectURL(blob);
        const worker = new Worker(workerURL, { type: 'module' });
        
        const offscreen = canvas.transferControlToOffscreen();
        
        worker.onmessage = (e) => {
            if (e.data.type === 'READY') {
                log('Worker ready! Three.js running in worker thread.');
            } else if (e.data.type === 'STATS') {
                updateUI(e.data);
            }
        };
        
        worker.onerror = (e) => {
            log(`Worker error: ${e.message}`);
            // Fallback to main thread
            fallbackToMain();
        };
        
        worker.postMessage(
            { type: 'INIT', canvas: offscreen, width: innerWidth, height: innerHeight },
            [offscreen]
        );
        
        window.addEventListener('resize', () => {
            worker.postMessage({ type: 'RESIZE', width: innerWidth, height: innerHeight });
        });
        
    } else {
        log('OffscreenCanvas not supported - using main thread');
        document.getElementById('thread-indicator').textContent = '⚠️ Fallback: Main Thread Rendering';
        document.getElementById('m-offscreen').textContent = 'ไม่รองรับ ❌';
        fallbackToMain();
    }
    
    function fallbackToMain() {
        // Fallback: render on main thread
        import('https://cdn.jsdelivr.net/npm/three@0.165.0/build/three.module.js').then(THREE => {
            const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
            renderer.setSize(innerWidth, innerHeight);
            
            const scene = new THREE.Scene();
            scene.background = new THREE.Color(0x050515);
            
            const camera = new THREE.PerspectiveCamera(60, innerWidth/innerHeight, 0.1, 500);
            camera.position.set(0, 0, 30);
            
            scene.add(new THREE.AmbientLight(0xffffff, 0.5));
            scene.add(new THREE.DirectionalLight(0xffffff, 1.5));
            
            const geo = new THREE.IcosahedronGeometry(0.5, 1);
            const mat = new THREE.MeshStandardMaterial();
            const mesh = new THREE.InstancedMesh(geo, mat, 200);
            scene.add(mesh);
            
            let t = 0;
            const m = new THREE.Matrix4();
            function animate() {
                requestAnimationFrame(animate);
                t += 0.01;
                for (let i = 0; i < 200; i++) {
                    m.setPosition(
                        Math.sin(i * 0.5 + t) * 15,
                        Math.cos(i * 0.3 + t) * 10,
                        Math.sin(i * 0.7 + t * 0.5) * 15
                    );
                    mesh.setMatrixAt(i, m);
                }
                mesh.instanceMatrix.needsUpdate = true;
                camera.position.x = Math.sin(t * 0.3) * 30;
                camera.position.z = Math.cos(t * 0.3) * 30;
                camera.lookAt(0, 0, 0);
                renderer.render(scene, camera);
                updateUI({ thread: 'Main Thread', workerFps: 60, objects: 200 });
            }
            animate();
        });
    }
    
    // FPS counter for main thread
    let mainFrames = 0, mainLastTime = performance.now(), mainFps = 0;
    function countMainFPS() {
        mainFrames++;
        const now = performance.now();
        if (now - mainLastTime >= 1000) {
            mainFps = mainFrames;
            mainFrames = 0;
            mainLastTime = now;
            document.getElementById('m-fps').textContent = mainFps;
        }
        requestAnimationFrame(countMainFPS);
    }
    countMainFPS();
</script>
</body>
</html>
```

---

## 7. Geometry Generation Worker with Progress

```javascript
// geometry-worker.js
self.onmessage = async (e) => {
    const { type, id, params } = e.data;
    
    if (type === 'GENERATE_TERRAIN') {
        const { width, depth, scale, octaves } = params;
        
        // Report progress
        const report = (progress, stage) => {
            self.postMessage({ type: 'PROGRESS', id, progress, stage });
        };
        
        report(0, 'Allocating buffers');
        
        const positions = new Float32Array(width * depth * 3);
        const normals = new Float32Array(width * depth * 3);
        const uvs = new Float32Array(width * depth * 2);
        const indices = new Uint32Array((width - 1) * (depth - 1) * 6);
        
        report(0.1, 'Generating heightmap');
        
        // Generate heightmap (Simplex noise simulation)
        const heights = new Float32Array(width * depth);
        for (let z = 0; z < depth; z++) {
            for (let x = 0; x < width; x++) {
                let h = 0;
                let freq = scale;
                let amp = 1;
                
                for (let o = 0; o < octaves; o++) {
                    h += Math.sin(x * freq) * Math.cos(z * freq) * amp;
                    freq *= 2;
                    amp *= 0.5;
                }
                
                heights[z * width + x] = h;
            }
            
            if (z % 10 === 0) report(0.1 + (z / depth) * 0.4, 'Heightmap');
        }
        
        report(0.5, 'Building vertex buffers');
        
        // Build vertex buffers
        for (let z = 0; z < depth; z++) {
            for (let x = 0; x < width; x++) {
                const i = z * width + x;
                const base3 = i * 3;
                const base2 = i * 2;
                
                positions[base3] = x - width / 2;
                positions[base3 + 1] = heights[i];
                positions[base3 + 2] = z - depth / 2;
                
                uvs[base2] = x / (width - 1);
                uvs[base2 + 1] = z / (depth - 1);
            }
        }
        
        report(0.7, 'Computing normals');
        
        // Compute normals (simplified)
        for (let z = 1; z < depth - 1; z++) {
            for (let x = 1; x < width - 1; x++) {
                const i = z * width + x;
                const hL = heights[i - 1];
                const hR = heights[i + 1];
                const hD = heights[(z - 1) * width + x];
                const hU = heights[(z + 1) * width + x];
                
                const base3 = i * 3;
                const nx = hL - hR;
                const ny = 2.0;
                const nz = hD - hU;
                const len = Math.sqrt(nx*nx + ny*ny + nz*nz);
                normals[base3] = nx / len;
                normals[base3 + 1] = ny / len;
                normals[base3 + 2] = nz / len;
            }
        }
        
        report(0.85, 'Building index buffer');
        
        // Build indices
        let idx = 0;
        for (let z = 0; z < depth - 1; z++) {
            for (let x = 0; x < width - 1; x++) {
                const a = z * width + x;
                const b = z * width + x + 1;
                const c = (z + 1) * width + x;
                const d = (z + 1) * width + x + 1;
                
                indices[idx++] = a;
                indices[idx++] = b;
                indices[idx++] = c;
                indices[idx++] = b;
                indices[idx++] = d;
                indices[idx++] = c;
            }
        }
        
        report(1.0, 'Complete');
        
        // Transfer buffers back (zero-copy with Transferable)
        self.postMessage(
            { 
                type: 'TERRAIN_READY', 
                id,
                positions: positions.buffer,
                normals: normals.buffer,
                uvs: uvs.buffer,
                indices: indices.buffer,
                width,
                depth
            },
            [positions.buffer, normals.buffer, uvs.buffer, indices.buffer]
        );
    }
};

// main.js: ใช้งาน geometry worker
const worker = new Worker('geometry-worker.js');

worker.onmessage = (e) => {
    const { type, id } = e.data;
    
    if (type === 'PROGRESS') {
        updateProgressBar(id, e.data.progress, e.data.stage);
    } else if (type === 'TERRAIN_READY') {
        // สร้าง BufferGeometry จาก transferred buffers
        const geo = new THREE.BufferGeometry();
        geo.setAttribute('position', new THREE.BufferAttribute(new Float32Array(e.data.positions), 3));
        geo.setAttribute('normal', new THREE.BufferAttribute(new Float32Array(e.data.normals), 3));
        geo.setAttribute('uv', new THREE.BufferAttribute(new Float32Array(e.data.uvs), 2));
        geo.setIndex(new THREE.BufferAttribute(new Uint32Array(e.data.indices), 1));
        
        const mesh = new THREE.Mesh(geo, new THREE.MeshStandardMaterial({ 
            color: 0x226633, 
            wireframe: false 
        }));
        scene.add(mesh);
    }
};

worker.postMessage({
    type: 'GENERATE_TERRAIN',
    id: 'terrain-1',
    params: { width: 256, depth: 256, scale: 0.05, octaves: 4 }
});
```

---

## 8. Worker Performance Tradeoffs

```javascript
// ⚠️ Serialization Cost: postMessage ต้อง serialize/deserialize data
// ใช้ Transferable Objects เพื่อ zero-copy transfer

// ❌ ช้า (copy)
worker.postMessage({ positions: bigFloat32Array });

// ✅ เร็ว (transfer - no copy)
worker.postMessage({ positions: bigFloat32Array.buffer }, [bigFloat32Array.buffer]);
// หลังจาก transfer, bigFloat32Array.buffer จะไม่ accessible ใน main thread อีกต่อไป!

// ✅ เร็วที่สุด (shared memory)
const shared = new SharedArrayBuffer(size);
worker.postMessage({ shared }); // ส่ง reference (ไม่ต้อง transfer)
// ทั้ง main และ worker ใช้ buffer เดียวกัน!

// Benchmarking worker overhead:
async function benchmarkWorker() {
    const data = new Float32Array(1_000_000).fill(1.0);
    
    const startCopy = performance.now();
    await new Promise(r => {
        worker.onmessage = r;
        worker.postMessage({ data: data }); // Copy (slow)
    });
    console.log('Copy time:', performance.now() - startCopy, 'ms');
    
    const startTransfer = performance.now();
    const dataCopy = data.slice(); // Need new buffer since we transferred
    await new Promise(r => {
        worker.onmessage = r;
        worker.postMessage({ data: dataCopy.buffer }, [dataCopy.buffer]); // Transfer (fast)
    });
    console.log('Transfer time:', performance.now() - startTransfer, 'ms');
}
```

---

## แบบฝึกหัด

1. **ง่าย**: สร้าง dedicated worker ที่คำนวณ Fibonacci แบบ blocking แล้ว postMessage ผลลัพธ์กลับ เปรียบเทียบ UI responsiveness
2. **กลาง**: ใช้ OffscreenCanvas render Three.js scene ใน worker thread โดยมี FPS counter บน main thread
3. **ยาก**: สร้าง Worker Pool สำหรับ generate terrain geometry แบบ parallel แล้ว display progress bars
4. **Challenge**: Implement physics simulation ใน worker ด้วย SharedArrayBuffer ให้ instances อัปเดต position โดยตรงโดยไม่ต้อง copy data

---

## อ้างอิง

- [MDN Web Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API)
- [OffscreenCanvas API](https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas)
- [SharedArrayBuffer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer)
- [Comlink Library](https://github.com/GoogleChromeLabs/comlink)
- [COOP/COEP Headers for SharedArrayBuffer](https://developer.chrome.com/blog/enabling-shared-array-buffer/)
