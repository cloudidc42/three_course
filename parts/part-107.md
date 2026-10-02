# Part 107: Memory Management & Disposal (Advanced)

## บทนำ: Memory Leak ใน WebGL

WebGL ต่างจาก JavaScript ตรงที่ Garbage Collector ไม่สามารถเก็บ GPU memory ได้โดยอัตโนมัติ เมื่อคุณสร้าง geometry, texture, หรือ material ใน Three.js ระบบจะ allocate memory บน GPU ด้วย

หากคุณไม่ `dispose()` objects เหล่านี้ก่อน remove ออกจาก scene GPU memory จะถูกใช้ไปเรื่อยๆ จนกว่า page จะ reload

```
ปัญหา Memory Leak:
Scene Update:
add Building A → GPU: 10MB
remove Building A → CPU frees, GPU still has 10MB ❌
add Building B → GPU: 20MB  
remove Building B → GPU: 20MB still ❌
...
ผล: GPU memory เต็ม → crash หรือ ช้าลงมากๆ

แก้ด้วย dispose():
remove Building A → building.geometry.dispose() + material.dispose()
→ GPU: 0MB ✅
```

---

## 1. WebGL Resource Lifecycle

```javascript
// สิ่งที่ต้อง dispose มีอะไรบ้าง:

// 1. BufferGeometry
const geo = new THREE.BoxGeometry(1, 1, 1);
// → สร้าง VBO (Vertex Buffer Object) ใน GPU
// Dispose: geo.dispose()

// 2. Material
const mat = new THREE.MeshStandardMaterial({ color: 0xff0000 });
// → สร้าง shader program ใน GPU
// Dispose: mat.dispose()

// 3. Texture
const texture = new THREE.TextureLoader().load('image.jpg');
// → สร้าง texture object ใน GPU VRAM
// Dispose: texture.dispose()

// 4. WebGLRenderTarget
const renderTarget = new THREE.WebGLRenderTarget(512, 512);
// → สร้าง framebuffer + texture ใน GPU
// Dispose: renderTarget.dispose()

// 5. Renderer
const renderer = new THREE.WebGLRenderer();
// → สร้าง WebGL context + GPU resources
// Dispose: renderer.dispose()

// 6. Material uniforms (textures ใน uniforms)
const shaderMat = new THREE.ShaderMaterial({
    uniforms: { myTexture: { value: someTexture } }
});
// ต้อง dispose texture ด้วยตนเองเหมือนกัน
```

---

## 2. Automatic vs Manual Disposal

### Three.js Automatic Disposal (limited)
```javascript
// Three.js ไม่ dispose อัตโนมัติเมื่อ:
scene.remove(mesh); // ❌ ยังคงมี GPU resources

// Three.js dispose อัตโนมัติเมื่อ:
renderer.dispose(); // ✅ dispose WebGL context ทั้งหมด
// แต่นี่หมายถึงปิด renderer ทั้งหมด!

// THREE.Cache จัดการ texture caching:
THREE.Cache.enabled = true; // enable default texture cache
THREE.Cache.get('url'); // ดึง texture จาก cache
THREE.Cache.remove('url'); // ลบออกจาก cache
THREE.Cache.clear(); // ล้าง cache ทั้งหมด
```

### Manual Disposal (best practice)
```javascript
function disposeMesh(mesh) {
    // Dispose geometry
    if (mesh.geometry) {
        mesh.geometry.dispose();
    }
    
    // Dispose material(s)
    if (mesh.material) {
        const materials = Array.isArray(mesh.material) 
            ? mesh.material 
            : [mesh.material];
        
        materials.forEach(mat => {
            // Dispose all textures in material
            Object.values(mat).forEach(value => {
                if (value && value.isTexture) {
                    value.dispose();
                }
            });
            mat.dispose();
        });
    }
    
    // Remove from parent
    if (mesh.parent) {
        mesh.parent.remove(mesh);
    }
}

// ใช้งาน
disposeMesh(oldBuilding);
```

---

## 3. Dispose Helper: Complete Implementation

```javascript
class DisposalManager {
    constructor(renderer) {
        this.renderer = renderer;
        this.disposed = new WeakSet();
    }
    
    // Dispose single object ทุก type
    disposeObject(obj) {
        if (this.disposed.has(obj)) return;
        this.disposed.add(obj);
        
        if (obj.isMesh || obj.isInstancedMesh || obj.isSkinnedMesh) {
            this._disposeMesh(obj);
        } else if (obj.isPoints) {
            this._disposeMesh(obj);
        } else if (obj.isLine || obj.isLineSegments) {
            this._disposeMesh(obj);
        } else if (obj.isSprite) {
            if (obj.material) this._disposeMaterial(obj.material);
        } else if (obj.isGroup || obj.isScene || obj.isObject3D) {
            // ไม่ต้อง dispose group เอง แต่ traverse children
        }
        
        // Dispose children recursively
        obj.children?.forEach(child => this.disposeObject(child));
    }
    
    _disposeMesh(mesh) {
        if (mesh.geometry) {
            this._disposeGeometry(mesh.geometry);
        }
        
        if (mesh.material) {
            const mats = Array.isArray(mesh.material) ? mesh.material : [mesh.material];
            mats.forEach(m => this._disposeMaterial(m));
        }
    }
    
    _disposeGeometry(geo) {
        if (this.disposed.has(geo)) return;
        this.disposed.add(geo);
        geo.dispose();
    }
    
    _disposeMaterial(mat) {
        if (this.disposed.has(mat)) return;
        this.disposed.add(mat);
        
        // Texture properties ที่ต้อง dispose
        const textureProps = [
            'map', 'normalMap', 'roughnessMap', 'metalnessMap',
            'emissiveMap', 'aoMap', 'displacementMap', 'bumpMap',
            'alphaMap', 'envMap', 'lightMap', 'matcap',
            'gradientMap', 'specularMap', 'clearcoatMap',
            'clearcoatRoughnessMap', 'clearcoatNormalMap',
            'transmissionMap', 'thicknessMap', 'sheenColorMap'
        ];
        
        textureProps.forEach(prop => {
            if (mat[prop] && mat[prop].isTexture) {
                this._disposeTexture(mat[prop]);
            }
        });
        
        // ShaderMaterial uniforms
        if (mat.uniforms) {
            Object.values(mat.uniforms).forEach(uniform => {
                if (uniform.value && uniform.value.isTexture) {
                    this._disposeTexture(uniform.value);
                }
                if (uniform.value && uniform.value.isWebGLRenderTarget) {
                    uniform.value.dispose();
                }
            });
        }
        
        mat.dispose();
    }
    
    _disposeTexture(texture) {
        if (this.disposed.has(texture)) return;
        this.disposed.add(texture);
        texture.dispose();
    }
    
    // Dispose entire scene
    disposeScene(scene) {
        // Collect all objects
        const objects = [];
        scene.traverse(obj => objects.push(obj));
        
        // Dispose from leaves to root
        objects.reverse().forEach(obj => this.disposeObject(obj));
        
        // Clear scene
        while (scene.children.length > 0) {
            scene.remove(scene.children[0]);
        }
    }
    
    // Dispose WebGL render targets
    disposeRenderTarget(rt) {
        if (!rt) return;
        
        if (rt.texture) this._disposeTexture(rt.texture);
        if (rt.depthTexture) this._disposeTexture(rt.depthTexture);
        rt.dispose();
    }
    
    // Full cleanup
    fullCleanup(scene) {
        this.disposeScene(scene);
        this.renderer.dispose();
        this.renderer.forceContextLoss(); // Force WebGL context reset
        
        const info = this.renderer.info;
        console.log('After cleanup:', {
            geometries: info.memory.geometries,
            textures: info.memory.textures,
            programs: info.programs?.length || 0
        });
    }
}
```

---

## 4. Memory Leak Patterns และวิธีแก้

### Pattern 1: ลืม dispose เมื่อ remove mesh

```javascript
// ❌ Memory Leak
function removeBuilding(id) {
    const building = buildingMap.get(id);
    scene.remove(building); // GPU memory ยังคงอยู่!
    buildingMap.delete(id);
}

// ✅ ถูกต้อง
function removeBuilding(id) {
    const building = buildingMap.get(id);
    
    // Dispose ก่อน remove
    if (building.geometry) building.geometry.dispose();
    if (building.material) {
        if (Array.isArray(building.material)) {
            building.material.forEach(m => m.dispose());
        } else {
            building.material.dispose();
        }
    }
    
    scene.remove(building);
    buildingMap.delete(id);
}
```

### Pattern 2: RenderTarget ไม่ถูก dispose

```javascript
// ❌ Leak - สร้าง render target ใหม่ทุกครั้งที่ resize
window.addEventListener('resize', () => {
    ssaoRenderTarget = new THREE.WebGLRenderTarget(innerWidth, innerHeight);
    // เดิมไม่ได้ dispose!
});

// ✅ ถูกต้อง
window.addEventListener('resize', () => {
    if (ssaoRenderTarget) ssaoRenderTarget.dispose(); // Dispose เดิมก่อน
    ssaoRenderTarget = new THREE.WebGLRenderTarget(innerWidth, innerHeight);
});
```

### Pattern 3: Event Listeners ที่ reference objects

```javascript
// ❌ Leak - event listener ป้องกัน GC
class GameEntity {
    constructor() {
        this.mesh = new THREE.Mesh(geo, mat);
        // Window event listener ที่ reference this
        window.addEventListener('resize', this.onResize = () => {
            this.mesh.scale.set(innerWidth/100, innerHeight/100, 1);
        });
    }
}

const entity = new GameEntity();
// entity ถูก remove แต่ window event listener ยังอ้างถึง entity.mesh อยู่!

// ✅ ถูกต้อง
class GameEntity {
    constructor() {
        this.mesh = new THREE.Mesh(geo, mat);
        this._resizeHandler = () => {
            this.mesh.scale.set(innerWidth/100, innerHeight/100, 1);
        };
        window.addEventListener('resize', this._resizeHandler);
    }
    
    destroy() {
        // Remove event listener ก่อน
        window.removeEventListener('resize', this._resizeHandler);
        
        // Dispose mesh
        this.mesh.geometry.dispose();
        this.mesh.material.dispose();
        
        this.mesh = null;
    }
}
```

### Pattern 4: Cloned Materials ที่ต้อง dispose แยก

```javascript
// ❌ Leak - clone material แต่ไม่ dispose
const sharedMat = new THREE.MeshStandardMaterial({ color: 0xff0000 });
const meshes = [];

for (let i = 0; i < 100; i++) {
    const mat = sharedMat.clone(); // clone สร้าง material ใหม่!
    mat.color.setHex(Math.random() * 0xffffff);
    meshes.push(new THREE.Mesh(geo, mat));
}

// เมื่อ dispose:
meshes.forEach(m => scene.remove(m)); // ❌ materials ไม่ถูก dispose

// ✅ ถูกต้อง
meshes.forEach(m => {
    m.material.dispose(); // ต้อง dispose clone ด้วย
    scene.remove(m);
});
sharedMat.dispose();
```

---

## 5. Object Pooling สำหรับ Frequently Created Objects

```javascript
class ObjectPool {
    constructor(factory, reset, initialSize = 20) {
        this.factory = factory;   // สร้าง object ใหม่
        this.reset = reset;       // reset object ก่อนนำกลับมาใช้
        this.pool = [];
        this.active = new Set();
        
        // Pre-populate pool
        for (let i = 0; i < initialSize; i++) {
            this.pool.push(factory());
        }
    }
    
    // ดึง object จาก pool
    get() {
        let obj;
        
        if (this.pool.length > 0) {
            obj = this.pool.pop();
        } else {
            // Pool หมด: สร้างใหม่
            obj = this.factory();
            console.warn('Pool exhausted, creating new object');
        }
        
        this.reset(obj);
        this.active.add(obj);
        return obj;
    }
    
    // คืน object กลับ pool
    release(obj) {
        if (!this.active.has(obj)) return;
        this.active.delete(obj);
        this.pool.push(obj);
    }
    
    // คืนทุก active object
    releaseAll() {
        this.active.forEach(obj => {
            this.pool.push(obj);
        });
        this.active.clear();
    }
    
    // Dispose pool ทั้งหมด
    dispose() {
        [...this.pool, ...this.active].forEach(obj => {
            if (obj.geometry) obj.geometry.dispose();
            if (obj.material) obj.material.dispose();
        });
        this.pool = [];
        this.active.clear();
    }
    
    get stats() {
        return {
            pooled: this.pool.length,
            active: this.active.size,
            total: this.pool.length + this.active.size
        };
    }
}

// ตัวอย่าง: Bullet Pool
const geo = new THREE.SphereGeometry(0.1, 6, 4);
const mat = new THREE.MeshBasicMaterial({ color: 0xffff00 });

const bulletPool = new ObjectPool(
    // Factory: สร้าง bullet mesh ใหม่
    () => {
        const bullet = new THREE.Mesh(geo.clone(), mat.clone());
        bullet.visible = false;
        scene.add(bullet);
        return bullet;
    },
    // Reset: reset state ก่อนใช้
    (bullet) => {
        bullet.visible = true;
        bullet.userData.velocity = new THREE.Vector3();
        bullet.userData.age = 0;
    }
);

// Fire bullet
function fireBullet(from, direction) {
    const bullet = bulletPool.get();
    bullet.position.copy(from);
    bullet.userData.velocity.copy(direction).multiplyScalar(30);
    return bullet;
}

// Update bullets
function updateBullets(dt) {
    bulletPool.active.forEach(bullet => {
        bullet.position.addScaledVector(bullet.userData.velocity, dt);
        bullet.userData.age += dt;
        
        if (bullet.userData.age > 3) { // 3 second lifetime
            bullet.visible = false;
            bulletPool.release(bullet);
        }
    });
}
```

---

## 6. Memory Snapshot และ Monitoring

```javascript
class MemoryMonitor {
    constructor(renderer) {
        this.renderer = renderer;
        this.snapshots = [];
        this.maxSnapshots = 100;
    }
    
    takeSnapshot(label = '') {
        const memory = this.renderer.info.memory;
        const render = this.renderer.info.render;
        
        const snapshot = {
            label,
            timestamp: performance.now(),
            geometries: memory.geometries,
            textures: memory.textures,
            programs: this.renderer.info.programs?.length || 0,
            drawCalls: render.calls,
            triangles: render.triangles
        };
        
        this.snapshots.push(snapshot);
        if (this.snapshots.length > this.maxSnapshots) {
            this.snapshots.shift();
        }
        
        return snapshot;
    }
    
    // เปรียบเทียบสอง snapshot
    compare(snapshotA, snapshotB) {
        return {
            geometriesDelta: snapshotB.geometries - snapshotA.geometries,
            texturesDelta: snapshotB.textures - snapshotA.textures,
            programsDelta: snapshotB.programs - snapshotA.programs,
            timeDelta: snapshotB.timestamp - snapshotA.timestamp
        };
    }
    
    // ตรวจหา memory leak
    detectLeak(threshold = 5) {
        if (this.snapshots.length < 2) return null;
        
        const first = this.snapshots[0];
        const last = this.snapshots[this.snapshots.length - 1];
        const diff = this.compare(first, last);
        
        const leaks = [];
        
        if (diff.geometriesDelta > threshold) {
            leaks.push(`⚠️ Geometries increased by ${diff.geometriesDelta}`);
        }
        if (diff.texturesDelta > threshold) {
            leaks.push(`⚠️ Textures increased by ${diff.texturesDelta}`);
        }
        if (diff.programsDelta > threshold) {
            leaks.push(`⚠️ Shader programs increased by ${diff.programsDelta}`);
        }
        
        return leaks.length > 0 ? leaks : null;
    }
    
    report() {
        const current = this.snapshots[this.snapshots.length - 1];
        const leaks = this.detectLeak();
        
        return {
            current,
            hasLeak: !!leaks,
            leaks,
            history: this.snapshots.slice(-10)
        };
    }
}
```

---

## 7. Memory Leak Detector Interactive Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Memory Leak Detector</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #050510; color: #eee; font-family: 'Courier New', monospace; overflow: hidden; }
        canvas { display: block; }
        #ui {
            position: fixed; top: 0; left: 0; width: 340px; height: 100vh;
            background: rgba(0,0,15,0.95);
            padding: 20px; border-right: 1px solid #224;
            overflow-y: auto; z-index: 10;
        }
        h2 { color: #0af; margin-bottom: 12px; font-size: 14px; }
        .section { margin: 12px 0; padding: 10px; background: rgba(255,255,255,0.04); border-radius: 4px; }
        .section h3 { color: #fa0; font-size: 11px; margin-bottom: 8px; text-transform: uppercase; }
        .row { display: flex; justify-content: space-between; margin: 4px 0; font-size: 12px; }
        .label { color: #888; }
        .val { color: #eee; font-weight: bold; }
        .ok { color: #4f4; }
        .warn { color: #fa0; }
        .error { color: #f44; }
        .btn-group { display: flex; gap: 4px; flex-wrap: wrap; margin: 8px 0; }
        button {
            padding: 6px 10px; background: #112233; color: #acd;
            border: 1px solid #336; border-radius: 3px; cursor: pointer; font-size: 11px;
        }
        button:hover { background: #224455; }
        button.danger { border-color: #633; color: #f88; }
        button.danger:hover { background: #332211; }
        #leak-chart {
            width: 100%; height: 80px; margin-top: 8px;
            background: rgba(0,0,0,0.3); border-radius: 4px;
        }
        #log {
            height: 150px; overflow-y: auto; margin-top: 8px;
            font-size: 10px; color: #666; line-height: 1.5;
            background: rgba(0,0,0,0.3); padding: 6px; border-radius: 4px;
        }
        .log-ok { color: #4f4; }
        .log-warn { color: #fa0; }
        .log-err { color: #f44; }
    </style>
</head>
<body>
<canvas id="c" style="margin-left: 340px;"></canvas>
<div id="ui">
    <h2>🔍 Memory Leak Detector</h2>
    
    <div class="section">
        <h3>GPU Memory (renderer.info)</h3>
        <div class="row"><span class="label">Geometries:</span><span id="m-geo" class="val">-</span></div>
        <div class="row"><span class="label">Textures:</span><span id="m-tex" class="val">-</span></div>
        <div class="row"><span class="label">Shader Programs:</span><span id="m-prog" class="val">-</span></div>
        <div class="row"><span class="label">Draw Calls:</span><span id="m-calls" class="val">-</span></div>
        <div class="row"><span class="label">Triangles:</span><span id="m-tris" class="val">-</span></div>
    </div>
    
    <div class="section">
        <h3>ทดสอบ Memory Leak</h3>
        <div class="btn-group">
            <button id="btn-leak" class="danger">สร้าง Leak</button>
            <button id="btn-clean">สร้างถูกต้อง</button>
            <button id="btn-clear-scene">Clear Scene</button>
        </div>
        <div class="row"><span class="label">Objects ที่ไม่ถูก dispose:</span></div>
        <div class="row"><span id="m-leak-count" class="error">0</span><span class="label">leaked objects</span></div>
    </div>
    
    <div class="section">
        <h3>Object Pool</h3>
        <div class="btn-group">
            <button id="btn-pool-fire">Fire Bullet (Pool)</button>
            <button id="btn-naive-fire">Fire Bullet (Naive)</button>
        </div>
        <div class="row"><span class="label">Pool Objects:</span><span id="m-pool" class="ok">-</span></div>
        <div class="row"><span class="label">Active:</span><span id="m-active" class="warn">-</span></div>
        <div class="row"><span class="label">Meshes in Scene:</span><span id="m-scene" class="val">-</span></div>
    </div>
    
    <div class="section">
        <h3>Memory Trend</h3>
        <canvas id="leak-chart" width="300" height="80"></canvas>
    </div>
    
    <div id="log"></div>
</div>

<script type="importmap">
{
    "imports": {
        "three": "https://cdn.jsdelivr.net/npm/three@0.165.0/build/three.module.js",
        "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.165.0/examples/jsm/"
    }
}
</script>
<script type="module">
    import * as THREE from 'three';
    import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
    
    const W = window.innerWidth - 340;
    const H = window.innerHeight;
    
    const renderer = new THREE.WebGLRenderer({ canvas: document.getElementById('c'), antialias: true });
    renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
    renderer.setSize(W, H);
    
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x050510);
    
    const camera = new THREE.PerspectiveCamera(60, W / H, 0.1, 200);
    camera.position.set(0, 10, 30);
    
    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    
    scene.add(new THREE.AmbientLight(0xffffff, 0.5));
    const sun = new THREE.DirectionalLight(0xffffff, 1);
    sun.position.set(10, 20, 10);
    scene.add(sun);
    
    const ground = new THREE.Mesh(
        new THREE.PlaneGeometry(100, 100),
        new THREE.MeshStandardMaterial({ color: 0x1a2a1a })
    );
    ground.rotation.x = -Math.PI / 2;
    scene.add(ground);
    
    // Log system
    const logDiv = document.getElementById('log');
    function addLog(msg, type = '') {
        const div = document.createElement('div');
        div.className = type ? `log-${type}` : '';
        div.textContent = `[${new Date().toLocaleTimeString()}] ${msg}`;
        logDiv.prepend(div);
        if (logDiv.children.length > 30) logDiv.lastChild.remove();
    }
    
    // Memory trend data
    const trendData = { geo: [], tex: [], prog: [] };
    const MAX_TREND = 60;
    
    function drawTrend() {
        const canvas = document.getElementById('leak-chart');
        const ctx = canvas.getContext('2d');
        const W = canvas.width, H = canvas.height;
        ctx.clearRect(0, 0, W, H);
        
        const datasets = [
            { data: trendData.geo, color: '#4af' },
            { data: trendData.tex, color: '#fa4' },
            { data: trendData.prog, color: '#f4a' }
        ];
        
        datasets.forEach(ds => {
            if (ds.data.length < 2) return;
            const max = Math.max(...ds.data, 1);
            ctx.strokeStyle = ds.color;
            ctx.lineWidth = 1.5;
            ctx.beginPath();
            ds.data.forEach((val, i) => {
                const x = (i / (MAX_TREND - 1)) * W;
                const y = H - (val / max) * H * 0.9 - 4;
                i === 0 ? ctx.moveTo(x, y) : ctx.lineTo(x, y);
            });
            ctx.stroke();
        });
        
        // Labels
        ctx.fillStyle = '#4af';
        ctx.font = '9px monospace';
        ctx.fillText(`Geo: ${trendData.geo.slice(-1)[0] || 0}`, 4, 12);
        ctx.fillStyle = '#fa4';
        ctx.fillText(`Tex: ${trendData.tex.slice(-1)[0] || 0}`, 50, 12);
        ctx.fillStyle = '#f4a';
        ctx.fillText(`Prog: ${trendData.prog.slice(-1)[0] || 0}`, 96, 12);
    }
    
    // ============== Leak Demo ==============
    let leakCount = 0;
    
    document.getElementById('btn-leak').addEventListener('click', () => {
        // สร้าง objects โดยไม่ dispose เมื่อไม่ใช้
        for (let i = 0; i < 10; i++) {
            const geo = new THREE.BoxGeometry(
                0.5 + Math.random(),
                0.5 + Math.random(),
                0.5 + Math.random()
            );
            const tex = new THREE.DataTexture(
                new Uint8Array([Math.random()*255, Math.random()*255, Math.random()*255, 255]),
                1, 1
            );
            tex.needsUpdate = true;
            
            const mat = new THREE.MeshStandardMaterial({ map: tex });
            const mesh = new THREE.Mesh(geo, mat);
            mesh.position.set(
                (Math.random() - 0.5) * 40,
                Math.random() * 5 + 1,
                (Math.random() - 0.5) * 40
            );
            scene.add(mesh);
            
            // ❌ จำลอง "leak" โดยไม่บันทึก reference
            // เราเพิ่ม mesh เข้า scene แต่ไม่มี reference จะ dispose
        }
        leakCount += 10;
        addLog(`สร้าง 10 objects (ไม่มี reference สำหรับ dispose)`, 'warn');
    });
    
    const cleanMeshes = [];
    document.getElementById('btn-clean').addEventListener('click', () => {
        // สร้าง objects โดย track reference
        for (let i = 0; i < 10; i++) {
            const geo = new THREE.BoxGeometry(
                0.5 + Math.random(),
                0.5 + Math.random(),
                0.5 + Math.random()
            );
            const mat = new THREE.MeshStandardMaterial({ 
                color: new THREE.Color().setHSL(Math.random(), 0.8, 0.5) 
            });
            const mesh = new THREE.Mesh(geo, mat);
            mesh.position.set(
                (Math.random() - 0.5) * 40,
                Math.random() * 5 + 1,
                (Math.random() - 0.5) * 40
            );
            scene.add(mesh);
            cleanMeshes.push(mesh);
        }
        addLog(`สร้าง 10 objects (tracked สำหรับ dispose)`, 'ok');
    });
    
    document.getElementById('btn-clear-scene').addEventListener('click', () => {
        // Dispose only tracked meshes properly
        cleanMeshes.forEach(mesh => {
            if (mesh.geometry) mesh.geometry.dispose();
            if (mesh.material) {
                if (mesh.material.map) mesh.material.map.dispose();
                mesh.material.dispose();
            }
            scene.remove(mesh);
        });
        const cleared = cleanMeshes.length;
        cleanMeshes.length = 0;
        
        // Remove leaked meshes (can't properly dispose without reference to their resources)
        const toRemove = [];
        scene.traverse(obj => {
            if (obj.isMesh && obj !== ground) toRemove.push(obj);
        });
        toRemove.forEach(obj => scene.remove(obj));
        
        leakCount = 0;
        addLog(`Cleared ${cleared} tracked + leaked objects`, 'ok');
    });
    
    // ============== Object Pool Demo ==============
    const bulletGeo = new THREE.SphereGeometry(0.15, 6, 4);
    const bulletMat = new THREE.MeshBasicMaterial({ color: 0xffff00 });
    
    // Pool implementation
    const bulletPool = [];
    const activeBullets = [];
    
    function getBullet() {
        let bullet;
        if (bulletPool.length > 0) {
            bullet = bulletPool.pop();
        } else {
            bullet = new THREE.Mesh(bulletGeo, bulletMat);
            scene.add(bullet);
        }
        bullet.visible = true;
        bullet.userData.age = 0;
        bullet.userData.velocity = new THREE.Vector3(
            (Math.random() - 0.5) * 20,
            5 + Math.random() * 10,
            (Math.random() - 0.5) * 20
        );
        activeBullets.push(bullet);
        return bullet;
    }
    
    function releaseBullet(bullet) {
        bullet.visible = false;
        const idx = activeBullets.indexOf(bullet);
        if (idx >= 0) activeBullets.splice(idx, 1);
        bulletPool.push(bullet);
    }
    
    document.getElementById('btn-pool-fire').addEventListener('click', () => {
        const bullet = getBullet();
        bullet.position.set(0, 2, 0);
        addLog(`Pool fire. Pool: ${bulletPool.length}, Active: ${activeBullets.length}`, 'ok');
    });
    
    const naiveBullets = [];
    document.getElementById('btn-naive-fire').addEventListener('click', () => {
        // ❌ สร้าง Mesh ใหม่ทุกครั้ง
        const geo = new THREE.SphereGeometry(0.15, 6, 4);
        const mat = new THREE.MeshBasicMaterial({ color: 0xff8800 });
        const bullet = new THREE.Mesh(geo, mat);
        bullet.position.set(0, 2, 0);
        bullet.userData.velocity = new THREE.Vector3(
            (Math.random() - 0.5) * 20,
            5 + Math.random() * 10,
            (Math.random() - 0.5) * 20
        );
        bullet.userData.age = 0;
        bullet.userData.isNaive = true;
        scene.add(bullet);
        naiveBullets.push(bullet);
        addLog(`Naive fire. New geo+mat created. Total naive: ${naiveBullets.length}`, 'warn');
    });
    
    // Update bullets
    let prevTime = performance.now();
    
    function updateBullets(dt) {
        // Pool bullets
        for (let i = activeBullets.length - 1; i >= 0; i--) {
            const b = activeBullets[i];
            b.position.addScaledVector(b.userData.velocity, dt);
            b.userData.velocity.y -= 9.8 * dt;
            b.userData.age += dt;
            
            if (b.userData.age > 3 || b.position.y < -5) {
                releaseBullet(b);
            }
        }
        
        // Naive bullets
        for (let i = naiveBullets.length - 1; i >= 0; i--) {
            const b = naiveBullets[i];
            b.position.addScaledVector(b.userData.velocity, dt);
            b.userData.velocity.y -= 9.8 * dt;
            b.userData.age += dt;
            
            if (b.userData.age > 3 || b.position.y < -5) {
                // ❌ ไม่ dispose เมื่อหมดอายุ
                scene.remove(b);
                naiveBullets.splice(i, 1);
                // geo และ mat ยังคงอยู่ใน GPU!
            }
        }
    }
    
    // Count all meshes in scene
    function countSceneMeshes() {
        let count = 0;
        scene.traverse(obj => { if (obj.isMesh) count++; });
        return count;
    }
    
    let lastTime = performance.now(), frames = 0, fps = 0;
    let trendInterval = setInterval(() => {
        const info = renderer.info;
        trendData.geo.push(info.memory.geometries);
        trendData.tex.push(info.memory.textures);
        trendData.prog.push(info.programs?.length || 0);
        if (trendData.geo.length > MAX_TREND) {
            trendData.geo.shift(); trendData.tex.shift(); trendData.prog.shift();
        }
        drawTrend();
    }, 200);
    
    function resize() {
        const W = window.innerWidth - 340;
        renderer.setSize(W, window.innerHeight);
        camera.aspect = W / window.innerHeight;
        camera.updateProjectionMatrix();
    }
    window.addEventListener('resize', resize);
    
    function animate() {
        requestAnimationFrame(animate);
        
        const now = performance.now();
        const dt = Math.min((now - prevTime) / 1000, 0.05);
        prevTime = now;
        
        updateBullets(dt);
        controls.update();
        renderer.render(scene, camera);
        
        frames++;
        if (now - lastTime >= 500) {
            fps = Math.round(frames / ((now - lastTime) / 1000));
            frames = 0; lastTime = now;
        }
        
        const info = renderer.info;
        document.getElementById('m-geo').textContent = info.memory.geometries;
        document.getElementById('m-tex').textContent = info.memory.textures;
        document.getElementById('m-prog').textContent = info.programs?.length || 0;
        document.getElementById('m-calls').textContent = info.render.calls;
        document.getElementById('m-tris').textContent = info.render.triangles.toLocaleString();
        document.getElementById('m-leak-count').textContent = leakCount;
        document.getElementById('m-pool').textContent = bulletPool.length;
        document.getElementById('m-active').textContent = activeBullets.length;
        document.getElementById('m-scene').textContent = countSceneMeshes();
    }
    
    animate();
</script>
</body>
</html>
```

---

## 8. Full Scene Teardown Utility

```javascript
// Utility สำหรับทำลาย Three.js scene อย่างสมบูรณ์
class SceneTeardown {
    static dispose(renderer, scene) {
        // 1. Traverse and collect all resources
        const resources = {
            geometries: new Set(),
            materials: new Set(),
            textures: new Set(),
            renderTargets: new Set()
        };
        
        scene.traverse((object) => {
            if (object.isMesh || object.isLine || object.isPoints || object.isSprite) {
                if (object.geometry) resources.geometries.add(object.geometry);
                
                const mats = Array.isArray(object.material) 
                    ? object.material 
                    : [object.material];
                
                mats.forEach(mat => {
                    if (!mat) return;
                    resources.materials.add(mat);
                    
                    // Collect textures from material properties
                    const texProps = Object.keys(mat).filter(k => mat[k]?.isTexture);
                    texProps.forEach(k => resources.textures.add(mat[k]));
                    
                    // Collect from uniforms (ShaderMaterial)
                    if (mat.uniforms) {
                        Object.values(mat.uniforms).forEach(u => {
                            if (u.value?.isTexture) resources.textures.add(u.value);
                            if (u.value?.isWebGLRenderTarget) resources.renderTargets.add(u.value);
                        });
                    }
                });
            }
        });
        
        // 2. Dispose all resources
        console.log(`Disposing: ${resources.geometries.size} geos, ${resources.materials.size} mats, ${resources.textures.size} textures`);
        
        resources.textures.forEach(t => t.dispose());
        resources.materials.forEach(m => m.dispose());
        resources.geometries.forEach(g => g.dispose());
        resources.renderTargets.forEach(rt => rt.dispose());
        
        // 3. Clear scene
        while (scene.children.length > 0) {
            scene.remove(scene.children[0]);
        }
        
        // 4. Optional: dispose renderer
        // renderer.dispose();
        
        console.log('Scene teardown complete');
        
        return {
            geometries: resources.geometries.size,
            materials: resources.materials.size,
            textures: resources.textures.size
        };
    }
}

// ใช้งาน
const stats = SceneTeardown.dispose(renderer, scene);
console.log(`Freed: ${stats.geometries} geometries, ${stats.materials} materials, ${stats.textures} textures`);
```

---

## แบบฝึกหัด

1. **ง่าย**: เขียน `disposeMesh(mesh)` function ที่ dispose geometry, material, และ textures ทั้งหมด
2. **กลาง**: สร้าง object pool สำหรับ particle system ที่ spawn/despawn 1000 particles ต่อวินาที
3. **ยาก**: สร้าง `MemoryMonitor` class ที่ track GPU memory over time และแสดง graph พร้อม alert เมื่อ detect leak
4. **Challenge**: Implement full scene manager ที่มี load/unload chunks ของ world geometry พร้อม proper disposal และ memory budgeting

---

## อ้างอิง

- [Three.js Dispose Guide](https://threejs.org/docs/#manual/en/introduction/How-to-dispose-of-objects)
- [WebGL Memory Management](https://developer.mozilla.org/en-US/docs/Web/API/WebGL_API/WebGL_best_practices)
- [JavaScript WeakRef](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakRef)
- [FinalizationRegistry](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/FinalizationRegistry)
