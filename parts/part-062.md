# Part 62: Memory Management & Disposal

## ทำไม Memory Management ถึงสำคัญ

WebGL เก็บ resources ใน GPU memory: geometries, textures, shader programs สิ่งเหล่านี้ไม่ถูก garbage collect โดย JavaScript engine โดยอัตโนมัติ ต้อง dispose() เอง

```
JavaScript GC ทำได้:
- ลบ reference → memory JS ถูก free

JavaScript GC ทำไม่ได้:
- ลบ GPU buffer (VBO)
- ลบ GPU texture
- ลบ compiled shader

ถ้าไม่ dispose() → GPU memory leak!
```

## dispose() Pattern

```javascript
// 1. Geometry
geometry.dispose();
// → ลบ VBO (Vertex Buffer Object) ใน GPU

// 2. Material
material.dispose();
// → ลบ compiled shader program

// 3. Texture
texture.dispose();
// → ลบ texture ใน GPU

// 4. RenderTarget
renderTarget.dispose();
// → ลบ framebuffer + color/depth textures

// 5. ทำ complete cleanup สำหรับ Mesh
function disposeMesh(mesh) {
    if (mesh.geometry) mesh.geometry.dispose();
    
    if (mesh.material) {
        const materials = Array.isArray(mesh.material) ? mesh.material : [mesh.material];
        materials.forEach(mat => {
            // Dispose map textures
            ['map', 'normalMap', 'roughnessMap', 'metalnessMap', 'aoMap',
             'emissiveMap', 'envMap', 'alphaMap', 'bumpMap', 'displacementMap',
             'lightMap', 'specularMap'].forEach(key => {
                if (mat[key]) mat[key].dispose();
            });
            mat.dispose();
        });
    }
}
```

## Scene Clearing

```javascript
// Clear scene อย่างถูกต้อง
function clearScene(scene) {
    const toDispose = [];
    
    scene.traverse(obj => {
        if (obj.geometry) toDispose.push(obj.geometry);
        if (obj.material) {
            const mats = Array.isArray(obj.material) ? obj.material : [obj.material];
            mats.forEach(mat => {
                Object.keys(mat).forEach(key => {
                    if (mat[key] instanceof THREE.Texture) {
                        mat[key].dispose();
                    }
                });
                mat.dispose();
            });
        }
    });
    
    toDispose.forEach(g => g.dispose());
    
    // ลบ children ทั้งหมด
    while (scene.children.length > 0) {
        scene.remove(scene.children[0]);
    }
}

// ลบ renderer เมื่อเสร็จใช้งาน
renderer.dispose();
```

## Memory Leak Detection

```javascript
class MemoryLeakDetector {
    constructor(renderer) {
        this.renderer = renderer;
        this.baseline = null;
        this.history = [];
    }
    
    capture() {
        return {
            geometries: this.renderer.info.memory.geometries,
            textures: this.renderer.info.memory.textures,
            programs: this.renderer.info.programs.length,
            timestamp: Date.now()
        };
    }
    
    setBaseline() {
        this.baseline = this.capture();
        console.log('Baseline set:', this.baseline);
    }
    
    check() {
        const current = this.capture();
        this.history.push(current);
        
        if (!this.baseline) return current;
        
        const delta = {
            geometries: current.geometries - this.baseline.geometries,
            textures: current.textures - this.baseline.textures,
            programs: current.programs - this.baseline.programs
        };
        
        if (delta.geometries > 0 || delta.textures > 0 || delta.programs > 0) {
            console.warn('⚠️ Possible Memory Leak Detected!');
            console.warn('Geometry leak:', delta.geometries);
            console.warn('Texture leak:', delta.textures);
            console.warn('Program leak:', delta.programs);
        }
        
        return { current, delta };
    }
    
    // หา leaky functions
    async testFunction(fn, iterations = 10) {
        const before = this.capture();
        
        for (let i = 0; i < iterations; i++) {
            fn();
        }
        
        // Wait for GC
        await new Promise(r => setTimeout(r, 100));
        
        const after = this.capture();
        const delta = {
            geometries: after.geometries - before.geometries,
            textures: after.textures - before.textures
        };
        
        if (delta.geometries > 0 || delta.textures > 0) {
            console.error('🚨 Memory Leak in function!', delta);
            return false;
        }
        
        console.log('✅ No leak detected');
        return true;
    }
}
```

## Object Pooling

```javascript
// Pool ป้องกันการ create/destroy บ่อยๆ
class GeometryPool {
    constructor(geometryFactory, initialSize = 10) {
        this.factory = geometryFactory;
        this.available = [];
        this.inUse = new Set();
        
        // Pre-allocate
        for (let i = 0; i < initialSize; i++) {
            this.available.push(this.factory());
        }
    }
    
    acquire() {
        let geo;
        if (this.available.length > 0) {
            geo = this.available.pop();
        } else {
            console.warn('Pool exhausted, creating new geometry');
            geo = this.factory();
        }
        this.inUse.add(geo);
        return geo;
    }
    
    release(geo) {
        if (!this.inUse.has(geo)) {
            console.warn('Geometry not from this pool!');
            return;
        }
        this.inUse.delete(geo);
        this.available.push(geo);
    }
    
    dispose() {
        this.available.forEach(g => g.dispose());
        this.inUse.forEach(g => g.dispose());
        this.available = [];
        this.inUse.clear();
    }
    
    get stats() {
        return {
            available: this.available.length,
            inUse: this.inUse.size,
            total: this.available.length + this.inUse.size
        };
    }
}

// Mesh Pool
class MeshPool {
    constructor(geometry, material, initialSize = 20) {
        this.geometry = geometry;
        this.material = material;
        this.available = [];
        this.inUse = new Set();
        
        for (let i = 0; i < initialSize; i++) {
            this._createMesh();
        }
    }
    
    _createMesh() {
        const mesh = new THREE.Mesh(this.geometry, this.material);
        mesh.visible = false;
        this.available.push(mesh);
        return mesh;
    }
    
    acquire(scene) {
        let mesh;
        if (this.available.length > 0) {
            mesh = this.available.pop();
        } else {
            mesh = this._createMesh();
        }
        mesh.visible = true;
        this.inUse.add(mesh);
        if (scene) scene.add(mesh);
        return mesh;
    }
    
    release(mesh, scene) {
        if (!this.inUse.has(mesh)) return;
        this.inUse.delete(mesh);
        mesh.visible = false;
        mesh.position.set(0, -1000, 0);
        if (scene) scene.remove(mesh);
        this.available.push(mesh);
    }
    
    releaseAll(scene) {
        this.inUse.forEach(mesh => this.release(mesh, scene));
    }
}
```

## ตัวอย่าง: Memory Leak Detector + Object Pool Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Memory Management Demo</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #0a0a0a; overflow: hidden; font-family: 'Courier New', monospace; }
        canvas { display: block; }
        #panel {
            position: fixed; top: 0; right: 0;
            width: 300px; height: 100vh;
            background: rgba(5,5,8,0.97);
            border-left: 1px solid #333;
            padding: 14px;
            overflow-y: auto;
        }
        h2 { color: #ff6644; margin-bottom: 10px; font-size: 13px; border-bottom: 1px solid #333; padding-bottom: 6px; }
        h3 { color: #ff9966; margin: 10px 0 6px; font-size: 12px; }
        .stat { display: flex; justify-content: space-between; padding: 3px 6px; font-size: 12px; margin: 2px 0; background: rgba(255,255,255,0.03); border-radius: 3px; }
        .stat .l { color: #888; }
        .stat .v { color: #ffcc44; }
        .stat.leak .v { color: #ff4444; font-weight: bold; }
        .stat.ok .v { color: #44ff88; }
        .btn {
            width: 100%; padding: 6px; margin: 3px 0;
            background: transparent; cursor: pointer;
            border-radius: 4px; font-size: 12px; font-family: inherit;
        }
        .btn.danger { color: #ff4444; border: 1px solid #ff4444; }
        .btn.safe { color: #44ff88; border: 1px solid #44ff88; }
        .btn.info { color: #4488ff; border: 1px solid #4488ff; }
        #log {
            background: #111; border: 1px solid #333;
            border-radius: 4px; padding: 6px;
            height: 180px; overflow-y: auto;
            font-size: 11px; line-height: 1.5;
        }
        .log-ok { color: #44ff88; }
        .log-warn { color: #ffcc44; }
        .log-err { color: #ff4444; }
        .pool-bar { background: #222; border-radius: 3px; height: 12px; margin: 3px 0; display: flex; overflow: hidden; }
        .pool-used { background: #ff6644; transition: width 0.3s; }
        .pool-free { background: #446644; transition: width 0.3s; }
    </style>
</head>
<body>
    <div id="panel">
        <h2>🧠 Memory Management</h2>

        <h3>GPU Memory</h3>
        <div class="stat" id="statGeos"><span class="l">Geometries</span><span class="v" id="vGeos">0</span></div>
        <div class="stat" id="statTexs"><span class="l">Textures</span><span class="v" id="vTexs">0</span></div>
        <div class="stat" id="statProgs"><span class="l">Programs</span><span class="v" id="vProgs">0</span></div>

        <h3>Object Pool</h3>
        <div class="stat"><span class="l">In Use</span><span class="v" id="vPoolUsed">0</span></div>
        <div class="stat"><span class="l">Available</span><span class="v" id="vPoolFree">0</span></div>
        <div class="pool-bar">
            <div class="pool-used" id="barUsed" style="width:0%"></div>
            <div class="pool-free" id="barFree" style="width:100%"></div>
        </div>

        <h3>Tests</h3>
        <button class="btn danger" id="btnLeak">
            🚨 Create 50 (NO dispose)
        </button>
        <button class="btn safe" id="btnSafe">
            ✅ Create 50 (with dispose)
        </button>
        <button class="btn info" id="btnPool">
            🔄 Pool: Spawn 20 Objects
        </button>
        <button class="btn info" id="btnRelease">
            ♻️ Pool: Release All
        </button>
        <button class="btn danger" id="btnClear">
            🗑️ Clear Scene
        </button>

        <h3>Log</h3>
        <div id="log"></div>
    </div>

    <script type="module">
        import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js';
        import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/controls/OrbitControls.js';

        const DW = window.innerWidth - 300, H = window.innerHeight;
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(DW, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        renderer.domElement.style.position = 'fixed';
        document.body.appendChild(renderer.domElement);

        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x0a0a10);
        const camera = new THREE.PerspectiveCamera(70, DW/H, 0.5, 200);
        camera.position.set(0, 10, 30);
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;

        scene.add(new THREE.AmbientLight(0x334466, 1.0));
        scene.add(Object.assign(new THREE.DirectionalLight(0xffffff, 1.5), { position: { x:30,y:50,z:30 } }));
        scene.add(new THREE.GridHelper(60, 30, 0x223344, 0x1a2233));

        const logEl = document.getElementById('log');
        function log(msg, type = 'ok') {
            const div = document.createElement('div');
            div.className = `log-${type}`;
            div.textContent = `[${new Date().toLocaleTimeString()}] ${msg}`;
            logEl.appendChild(div);
            logEl.scrollTop = logEl.scrollHeight;
        }

        const leakyMeshes = [];
        const safeMeshes = [];

        // Leaky creation (no dispose)
        document.getElementById('btnLeak').addEventListener('click', () => {
            const before = { g: renderer.info.memory.geometries, t: renderer.info.memory.textures };
            for (let i = 0; i < 50; i++) {
                const geo = new THREE.SphereGeometry(0.3, 8, 6);
                const tex = new THREE.DataTexture(new Uint8Array([200, 50, 50, 255]), 1, 1);
                tex.needsUpdate = true;
                const mat = new THREE.MeshStandardMaterial({ map: tex });
                const mesh = new THREE.Mesh(geo, mat);
                mesh.position.set((Math.random()-0.5)*20, 1, (Math.random()-0.5)*20);
                scene.add(mesh);
                leakyMeshes.push(mesh);
            }
            const after = { g: renderer.info.memory.geometries, t: renderer.info.memory.textures };
            log(`Added 50 meshes (no dispose). Geo: +${after.g - before.g}, Tex: +${after.t - before.t}`, 'err');
        });

        // Safe creation (with dispose)
        document.getElementById('btnSafe').addEventListener('click', () => {
            const before = { g: renderer.info.memory.geometries, t: renderer.info.memory.textures };

            // Dispose previously safe meshes first
            safeMeshes.forEach(m => {
                scene.remove(m);
                m.geometry.dispose();
                if (m.material.map) m.material.map.dispose();
                m.material.dispose();
            });
            safeMeshes.length = 0;

            for (let i = 0; i < 50; i++) {
                const geo = new THREE.BoxGeometry(0.5, 0.5, 0.5);
                const mat = new THREE.MeshStandardMaterial({ color: 0x44cc88 });
                const mesh = new THREE.Mesh(geo, mat);
                mesh.position.set((Math.random()-0.5)*20, 0.5, (Math.random()-0.5)*20);
                scene.add(mesh);
                safeMeshes.push(mesh);
            }

            const after = { g: renderer.info.memory.geometries, t: renderer.info.memory.textures };
            log(`Added 50 meshes (safe). Geo: +${after.g - before.g}`, 'ok');
        });

        // Object Pool
        const poolGeo = new THREE.CapsuleGeometry(0.3, 0.8, 4, 8);
        const poolMat = new THREE.MeshStandardMaterial({ color: 0x4488ff, roughness: 0.5 });

        class Pool {
            constructor(size) {
                this.free = [];
                this.used = new Set();
                for (let i = 0; i < size; i++) {
                    const m = new THREE.Mesh(poolGeo, poolMat);
                    m.visible = false;
                    scene.add(m);
                    this.free.push(m);
                }
            }
            acquire() {
                if (!this.free.length) { log('Pool empty!', 'warn'); return null; }
                const m = this.free.pop();
                m.visible = true;
                this.used.add(m);
                return m;
            }
            release(m) {
                this.used.delete(m);
                m.visible = false;
                m.position.set(0, -100, 0);
                this.free.push(m);
            }
            releaseAll() { [...this.used].forEach(m => this.release(m)); }
            get stats() { return { used: this.used.size, free: this.free.length, total: this.used.size + this.free.length }; }
        }

        const pool = new Pool(30);

        document.getElementById('btnPool').addEventListener('click', () => {
            for (let i = 0; i < 20; i++) {
                const m = pool.acquire();
                if (m) m.position.set((Math.random()-0.5)*20, 1, (Math.random()-0.5)*20);
            }
            const s = pool.stats;
            log(`Pool: ${s.used} used, ${s.free} free`, 'ok');
        });

        document.getElementById('btnRelease').addEventListener('click', () => {
            pool.releaseAll();
            log(`Pool: all released. ${pool.stats.free} available`, 'ok');
        });

        document.getElementById('btnClear').addEventListener('click', () => {
            const before = { g: renderer.info.memory.geometries, t: renderer.info.memory.textures };

            leakyMeshes.forEach(m => {
                scene.remove(m);
                m.geometry.dispose();
                if (m.material.map) m.material.map.dispose();
                m.material.dispose();
            });
            leakyMeshes.length = 0;

            safeMeshes.forEach(m => {
                scene.remove(m);
                m.geometry.dispose();
                m.material.dispose();
            });
            safeMeshes.length = 0;

            pool.releaseAll();

            const after = { g: renderer.info.memory.geometries, t: renderer.info.memory.textures };
            log(`Cleared. Geo freed: ${before.g - after.g}, Tex freed: ${before.t - after.t}`, 'ok');
        });

        // Update stats
        function updateStats() {
            const info = renderer.info;
            document.getElementById('vGeos').textContent = info.memory.geometries;
            document.getElementById('vTexs').textContent = info.memory.textures;
            document.getElementById('vProgs').textContent = info.programs.length;

            const ps = pool.stats;
            document.getElementById('vPoolUsed').textContent = ps.used;
            document.getElementById('vPoolFree').textContent = ps.free;
            const pct = ps.total > 0 ? (ps.used / ps.total * 100) : 0;
            document.getElementById('barUsed').style.width = pct + '%';
            document.getElementById('barFree').style.width = (100 - pct) + '%';
        }

        window.addEventListener('resize', () => {
            const dw = window.innerWidth - 300;
            camera.aspect = dw / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(dw, window.innerHeight);
        });

        let lastStats = 0;
        function animate() {
            requestAnimationFrame(animate);
            controls.update();
            if (performance.now() - lastStats > 1000) {
                updateStats();
                lastStats = performance.now();
            }
            renderer.render(scene, camera);
        }
        animate();
    </script>
</body>
</html>
```

## Texture Disposal ที่ถูกต้อง

```javascript
// ลบ texture ทุกชนิดใน material
function disposeMaterialTextures(material) {
    const textureProps = [
        'map', 'normalMap', 'roughnessMap', 'metalnessMap',
        'aoMap', 'emissiveMap', 'envMap', 'alphaMap',
        'bumpMap', 'displacementMap', 'lightMap', 'specularMap',
        'clearcoatMap', 'clearcoatNormalMap', 'clearcoatRoughnessMap',
        'transmissionMap', 'thicknessMap', 'sheenColorMap',
        'sheenRoughnessMap', 'iridescenceMap', 'iridescenceThicknessMap',
        'gradientMap', 'anisotropyMap', 'envMapRotation'
    ];
    
    textureProps.forEach(prop => {
        if (material[prop] instanceof THREE.Texture) {
            material[prop].dispose();
        }
    });
}

// ลบ EffectComposer passes
function disposeComposer(composer) {
    composer.passes.forEach(pass => {
        if (pass.fsQuad) pass.fsQuad.dispose();
        if (pass.material) pass.material.dispose();
    });
    composer.dispose();
}
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Memory Profiler
สร้าง profiler ที่:
- Track geometry/texture count over time
- Graph แสดงการเพิ่มของ memory
- Alert เมื่อ rate สูงเกินปกติ

### แบบฝึกหัดที่ 2: Scene Loader
สร้าง scene loader ที่:
- Load scene ใหม่
- Dispose scene เก่าสมบูรณ์
- ตรวจสอบว่า memory กลับมาเท่าเดิม

### แบบฝึกหัดที่ 3: Particle Pool
สร้าง particle system ด้วย object pool:
- Pool size คงที่ 1000
- Spawn/kill particles ด้วย pool
- ไม่มี geometry/material allocations ใน runtime

### แบบฝึกหัดที่ 4: Auto Cleanup System
สร้าง system ที่:
- Track ทุก object ที่เพิ่มเข้า scene
- Auto dispose เมื่อ remove
- รายงาน leaks

---

**สรุปบทที่ 62:**
- dispose() ต้องเรียกใน geometry, material, texture, renderTarget ทุกชนิด
- JavaScript GC ไม่ทำ GPU cleanup อัตโนมัติ
- Object Pool ป้องกัน allocation/deallocation runtime
- renderer.info.memory ติดตาม GPU resource count
- Material textures ต้อง dispose แยกก่อน dispose material
- EffectComposer มี renderTarget ของตัวเองที่ต้อง dispose

**บทต่อไป:** WebGL Profiling & Debugging
