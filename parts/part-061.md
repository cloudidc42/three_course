# Part 61: Draw Call Optimization

## renderer.info

`renderer.info` คือ tool สำคัญที่สุดสำหรับ diagnose performance:

```javascript
// render.info ให้ข้อมูล per-frame
const info = renderer.info;

// Render stats (reset ทุก frame)
info.render.calls;       // จำนวน draw calls
info.render.triangles;   // จำนวน triangles
info.render.lines;       // จำนวน lines
info.render.points;      // จำนวน points
info.render.frame;       // frame counter

// Memory stats (สะสม)
info.memory.geometries;  // geometries ใน GPU memory
info.memory.textures;    // textures ใน GPU memory

// Programs
info.programs.length;    // shader programs compiled

// Enable เพื่อให้ track ได้
renderer.info.autoReset = true; // default

// Reset manual
renderer.info.reset();
```

## Draw Call คืออะไร

Draw Call = คำสั่งที่ CPU ส่งไป GPU บอกว่า "render geometry นี้ด้วย shader/material นี้"

```
1 Mesh = 1 Draw Call (ปกติ)
1000 Meshes = 1000 Draw Calls → ช้า!

Draw Call Overhead: ~0.1-1ms ต่อ call บน CPU
Target: < 200 draw calls สำหรับ mobile, < 2000 สำหรับ desktop
```

## Material Sharing

```javascript
// BAD: สร้าง material ใหม่ทุกครั้ง
for (let i = 0; i < 1000; i++) {
    const mesh = new THREE.Mesh(geo, new THREE.MeshStandardMaterial({
        color: 0xff0000 // สีเดิม แต่ material ต่างกัน
    }));
}
// → 1000 shader programs!

// GOOD: share material
const sharedMat = new THREE.MeshStandardMaterial({ color: 0xff0000 });
for (let i = 0; i < 1000; i++) {
    const mesh = new THREE.Mesh(geo, sharedMat);
}
// → 1 shader program, 1000 draw calls

// ถ้าต้องการสีต่างกัน → ใช้ vertex colors หรือ InstancedMesh
```

## Geometry Sharing

```javascript
// BAD: สร้าง geometry ใหม่ทุกครั้ง
for (let i = 0; i < 1000; i++) {
    const geo = new THREE.BoxGeometry(1, 1, 1);
    const mesh = new THREE.Mesh(geo, material);
    mesh.position.x = i;
}
// → 1000 geometries ใน GPU memory!

// GOOD: share geometry
const sharedGeo = new THREE.BoxGeometry(1, 1, 1);
for (let i = 0; i < 1000; i++) {
    const mesh = new THREE.Mesh(sharedGeo, material);
    mesh.position.x = i;
}
// → 1 geometry ใน GPU memory
```

## Texture Atlas

รวม texture หลายชิ้นเป็น texture เดียว เพื่อลด texture switching:

```javascript
// Texture Atlas: layout
// +--------+--------+
// | tex0   | tex1   |
// |        |        |
// +--------+--------+
// | tex2   | tex3   |
// |        |        |
// +--------+--------+

class TextureAtlas {
    constructor(size = 1024) {
        this.size = size;
        this.canvas = document.createElement('canvas');
        this.canvas.width = this.canvas.height = size;
        this.ctx = this.canvas.getContext('2d');
        this.regions = {};
        this.cursor = { x: 0, y: 0, rowHeight: 0 };
    }
    
    add(name, image, w, h) {
        // ถ้าไม่พอในแถวนี้ → ขึ้นแถวใหม่
        if (this.cursor.x + w > this.size) {
            this.cursor.x = 0;
            this.cursor.y += this.cursor.rowHeight;
            this.cursor.rowHeight = 0;
        }
        
        this.ctx.drawImage(image, this.cursor.x, this.cursor.y, w, h);
        
        // บันทึก UV region
        this.regions[name] = {
            u: this.cursor.x / this.size,
            v: this.cursor.y / this.size,
            uw: w / this.size,
            vh: h / this.size
        };
        
        this.cursor.x += w;
        this.cursor.rowHeight = Math.max(this.cursor.rowHeight, h);
        
        return this.regions[name];
    }
    
    // แปลง local UV เป็น atlas UV
    getAtlasUV(name, u, v) {
        const r = this.regions[name];
        return {
            u: r.u + u * r.uw,
            v: r.v + v * r.vh
        };
    }
    
    // สร้าง Three.js Texture
    build() {
        const texture = new THREE.CanvasTexture(this.canvas);
        texture.needsUpdate = true;
        return texture;
    }
}

// ใช้งาน
const atlas = new TextureAtlas(1024);
atlas.add('brick', brickImage, 256, 256);
atlas.add('stone', stoneImage, 256, 256);
atlas.add('wood', woodImage, 256, 256);

const atlasTexture = atlas.build();

// UV ใน geometry ต้องปรับให้ชี้ไป region ที่ถูก
function applyAtlasUV(geometry, regionName, atlas) {
    const region = atlas.regions[regionName];
    const uvs = geometry.attributes.uv.array;
    
    for (let i = 0; i < uvs.length; i += 2) {
        uvs[i] = region.u + uvs[i] * region.uw;
        uvs[i+1] = region.v + uvs[i+1] * region.vh;
    }
    
    geometry.attributes.uv.needsUpdate = true;
}
```

## Render Order

```javascript
// render order ควบคุมลำดับ render
mesh.renderOrder = 0; // default

// Transparent objects ควร render หลัง opaque
opaqueMesh.renderOrder = 0;
transparentMesh.renderOrder = 1;

// UI elements render สุดท้าย
uiMesh.renderOrder = 999;

// Three.js sort transparent objects ด้วย depth automatically
// แต่ render order override การ sort นี้ได้
```

## Batching Strategies

```javascript
// 1. Static Batching: merge static geometry
// ดีสำหรับ environment, buildings, terrain
function staticBatch(objects) {
    const geometries = [];
    objects.forEach(obj => {
        obj.updateWorldMatrix(true, false);
        const geo = obj.geometry.clone().applyMatrix4(obj.matrixWorld);
        geometries.push(geo);
    });
    return mergeGeometries(geometries);
}

// 2. Dynamic Batching: InstancedMesh
// ดีสำหรับ many identical objects (trees, particles)
const instancedMesh = new THREE.InstancedMesh(geo, mat, count);

// 3. Draw Call Batching: sort by material then render
// Three.js ทำ material sorting อัตโนมัติ
// แต่เราควร minimize จำนวน unique materials

// 4. Texture Batching: texture atlas
// รวม textures → ใช้ material เดียวได้

// 5. Shader Batching: ShaderMaterial พร้อม array uniforms
// ส่ง data array แทนการใช้หลาย material
```

## ตัวอย่าง: Optimization Workshop Dashboard

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Draw Call Optimization Dashboard</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #08080f; overflow: hidden; font-family: 'Courier New', monospace; color: #ccc; }
        canvas { display: block; }
        #dashboard {
            position: fixed; top: 0; right: 0;
            width: 320px; height: 100vh;
            background: rgba(5,8,15,0.95);
            border-left: 1px solid #2244aa;
            padding: 16px;
            overflow-y: auto;
        }
        h2 { color: #4488ff; margin-bottom: 12px; font-size: 14px; border-bottom: 1px solid #2244aa; padding-bottom: 8px; }
        h3 { color: #88aaff; margin: 12px 0 6px; font-size: 12px; }
        .metric { 
            display: flex; justify-content: space-between;
            padding: 4px 8px; margin: 2px 0;
            background: rgba(255,255,255,0.03);
            border-radius: 3px;
            font-size: 12px;
        }
        .metric .label { color: #8899aa; }
        .metric .value { color: #ffcc44; font-weight: bold; }
        .metric.good .value { color: #44ff88; }
        .metric.warn .value { color: #ffaa22; }
        .metric.bad .value { color: #ff4444; }
        .bar-container { background: #111; border-radius: 3px; height: 8px; margin: 3px 0; overflow: hidden; }
        .bar { height: 100%; border-radius: 3px; transition: width 0.3s; }
        .bar.green { background: #44ff88; }
        .bar.yellow { background: #ffaa22; }
        .bar.red { background: #ff4444; }
        .mode-row { display: flex; gap: 4px; flex-wrap: wrap; margin: 6px 0; }
        .mode-btn {
            flex: 1; padding: 5px; font-size: 11px;
            background: transparent; color: #4488ff;
            border: 1px solid #2244aa; cursor: pointer;
            border-radius: 3px; min-width: 60px;
        }
        .mode-btn.active { background: rgba(34,68,170,0.4); color: #aaddff; }
        .history { height: 60px; background: #111; border-radius: 4px; position: relative; overflow: hidden; }
        .history canvas { width: 100%; height: 100%; }
        .tip { font-size: 11px; color: #556; padding: 4px 8px; line-height: 1.5; }
    </style>
</head>
<body>
    <div id="dashboard">
        <h2>⚡ Draw Call Optimizer</h2>
        
        <h3>📊 Real-time Stats</h3>
        <div class="metric" id="metFPS">
            <span class="label">FPS</span>
            <span class="value" id="valFPS">--</span>
        </div>
        <div class="bar-container"><div class="bar green" id="barFPS" style="width:0%"></div></div>
        
        <div class="metric" id="metDraws">
            <span class="label">Draw Calls</span>
            <span class="value" id="valDraws">--</span>
        </div>
        <div class="bar-container"><div class="bar" id="barDraws" style="width:0%"></div></div>
        
        <div class="metric">
            <span class="label">Triangles</span>
            <span class="value" id="valTris">--</span>
        </div>
        <div class="metric">
            <span class="label">GPU Geometries</span>
            <span class="value" id="valGeos">--</span>
        </div>
        <div class="metric">
            <span class="label">GPU Textures</span>
            <span class="value" id="valTexs">--</span>
        </div>
        <div class="metric">
            <span class="label">Objects in Scene</span>
            <span class="value" id="valObjs">--</span>
        </div>
        
        <h3>🔬 Optimization Mode</h3>
        <div class="mode-row">
            <button class="mode-btn active" data-mode="baseline">Baseline</button>
            <button class="mode-btn" data-mode="shared-mat">Shared Mat</button>
            <button class="mode-btn" data-mode="shared-geo">Shared Geo</button>
            <button class="mode-btn" data-mode="instanced">Instanced</button>
            <button class="mode-btn" data-mode="merged">Merged</button>
        </div>
        
        <div class="metric">
            <span class="label">Mode Description</span>
        </div>
        <div class="tip" id="modeTip">
            Baseline: สร้าง material + geometry ใหม่ทุก object
        </div>
        
        <h3>📈 FPS History</h3>
        <div class="history">
            <canvas id="historyCanvas" width="280" height="60"></canvas>
        </div>
        
        <h3>💡 Tips</h3>
        <div id="tips" class="tip">
            Loading...
        </div>
    </div>

    <script type="module">
        import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js';
        import { mergeGeometries } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/utils/BufferGeometryUtils.js';
        import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/controls/OrbitControls.js';

        const DW = window.innerWidth - 320, H = window.innerHeight;
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(DW, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        renderer.domElement.style.position = 'fixed';
        renderer.domElement.style.left = '0';
        document.body.appendChild(renderer.domElement);

        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x080810);

        const camera = new THREE.PerspectiveCamera(70, DW/H, 0.5, 500);
        camera.position.set(0, 30, 60);

        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;

        scene.add(new THREE.AmbientLight(0x334466, 1));
        const dir = new THREE.DirectionalLight(0xffffff, 1.5);
        dir.position.set(30, 60, 30);
        scene.add(dir);

        const ground = new THREE.Mesh(
            new THREE.PlaneGeometry(200, 200),
            new THREE.MeshStandardMaterial({ color: 0x111122 })
        );
        ground.rotation.x = -Math.PI / 2;
        scene.add(ground);

        const COUNT = 500;
        const SPREAD = 80;
        let currentObjects = [];

        const modeDescriptions = {
            'baseline': 'Baseline: สร้าง material + geometry ใหม่ทุก object',
            'shared-mat': 'Shared Material: geometry ใหม่, material เดิม',
            'shared-geo': 'Shared Geometry: geometry เดิม, material เดิม',
            'instanced': 'InstancedMesh: 1 draw call สำหรับทุก object',
            'merged': 'Merged Geometry: 1 draw call, static'
        };

        const modeTips = {
            'baseline': '⚠️ การสร้าง Material ใหม่ทุกครั้งสร้าง shader program ใหม่ทำให้ช้ามาก',
            'shared-mat': '✅ การ share material ลด shader compilation แต่ยังมีหลาย draw calls',
            'shared-geo': '✅ share geometry ลด GPU memory, share material ลด state changes',
            'instanced': '🚀 1 draw call! เหมาะกับ objects เยอะที่ใช้ geometry เดิม',
            'merged': '🚀 1 draw call! เหมาะกับ static geometry ที่ไม่ต้องเคลื่อนที่'
        };

        function clearObjects() {
            currentObjects.forEach(obj => {
                scene.remove(obj);
                if (obj.geometry && !obj.geometry._shared) obj.geometry.dispose();
                if (obj.material) {
                    if (!obj.material._shared) {
                        if (Array.isArray(obj.material)) obj.material.forEach(m => m.dispose());
                        else obj.material.dispose();
                    }
                }
            });
            currentObjects = [];
        }

        function buildBaseline() {
            for (let i = 0; i < COUNT; i++) {
                const geo = new THREE.BoxGeometry(1, 1 + Math.random() * 3, 1);
                const mat = new THREE.MeshStandardMaterial({
                    color: new THREE.Color().setHSL(Math.random(), 0.6, 0.5)
                });
                const mesh = new THREE.Mesh(geo, mat);
                mesh.position.set(
                    (Math.random() - 0.5) * SPREAD,
                    mesh.geometry.parameters.height / 2,
                    (Math.random() - 0.5) * SPREAD
                );
                scene.add(mesh);
                currentObjects.push(mesh);
            }
        }

        function buildSharedMat() {
            const mat = new THREE.MeshStandardMaterial({ color: 0x4488cc });
            mat._shared = true;
            for (let i = 0; i < COUNT; i++) {
                const h = 1 + Math.random() * 3;
                const geo = new THREE.BoxGeometry(1, h, 1);
                const mesh = new THREE.Mesh(geo, mat);
                mesh.position.set((Math.random()-0.5)*SPREAD, h/2, (Math.random()-0.5)*SPREAD);
                scene.add(mesh);
                currentObjects.push(mesh);
            }
        }

        function buildSharedGeo() {
            const geo = new THREE.BoxGeometry(1, 2, 1);
            geo._shared = true;
            const mat = new THREE.MeshStandardMaterial({ color: 0x44cc88, vertexColors: false });
            mat._shared = true;
            for (let i = 0; i < COUNT; i++) {
                const mesh = new THREE.Mesh(geo, mat);
                mesh.position.set((Math.random()-0.5)*SPREAD, 1, (Math.random()-0.5)*SPREAD);
                mesh.scale.y = 0.5 + Math.random() * 2;
                scene.add(mesh);
                currentObjects.push(mesh);
            }
        }

        function buildInstanced() {
            const geo = new THREE.BoxGeometry(1, 1, 1);
            const mat = new THREE.MeshStandardMaterial({ vertexColors: true });
            const mesh = new THREE.InstancedMesh(geo, mat, COUNT);
            const dummy = new THREE.Object3D();
            const color = new THREE.Color();
            for (let i = 0; i < COUNT; i++) {
                const h = 1 + Math.random() * 3;
                dummy.position.set((Math.random()-0.5)*SPREAD, h/2, (Math.random()-0.5)*SPREAD);
                dummy.scale.y = h;
                dummy.updateMatrix();
                mesh.setMatrixAt(i, dummy.matrix);
                color.setHSL(i / COUNT, 0.6, 0.5);
                mesh.setColorAt(i, color);
            }
            mesh.instanceMatrix.needsUpdate = true;
            if (mesh.instanceColor) mesh.instanceColor.needsUpdate = true;
            scene.add(mesh);
            currentObjects.push(mesh);
        }

        function buildMerged() {
            const geos = [];
            const color = new THREE.Color();
            for (let i = 0; i < COUNT; i++) {
                const h = 1 + Math.random() * 3;
                const geo = new THREE.BoxGeometry(1, h, 1);
                const x = (Math.random()-0.5)*SPREAD;
                const z = (Math.random()-0.5)*SPREAD;
                geo.translate(x, h/2, z);

                const cnt = geo.attributes.position.count;
                const colors = new Float32Array(cnt * 3);
                color.setHSL(i / COUNT, 0.6, 0.5);
                for (let j = 0; j < cnt; j++) {
                    colors[j*3] = color.r; colors[j*3+1] = color.g; colors[j*3+2] = color.b;
                }
                geo.setAttribute('color', new THREE.BufferAttribute(colors, 3));
                geos.push(geo);
            }
            const merged = mergeGeometries(geos);
            geos.forEach(g => g.dispose());
            const mesh = new THREE.Mesh(merged, new THREE.MeshStandardMaterial({ vertexColors: true }));
            scene.add(mesh);
            currentObjects.push(mesh);
        }

        let currentMode = 'baseline';
        buildBaseline();

        document.querySelectorAll('.mode-btn').forEach(btn => {
            btn.addEventListener('click', () => {
                document.querySelectorAll('.mode-btn').forEach(b => b.classList.remove('active'));
                btn.classList.add('active');
                currentMode = btn.dataset.mode;
                document.getElementById('modeTip').textContent = modeDescriptions[currentMode];
                document.getElementById('tips').textContent = modeTips[currentMode];
                clearObjects();
                if (currentMode === 'baseline') buildBaseline();
                else if (currentMode === 'shared-mat') buildSharedMat();
                else if (currentMode === 'shared-geo') buildSharedGeo();
                else if (currentMode === 'instanced') buildInstanced();
                else if (currentMode === 'merged') buildMerged();
            });
        });

        document.getElementById('tips').textContent = modeTips['baseline'];

        // FPS History chart
        const histCanvas = document.getElementById('historyCanvas');
        const histCtx = histCanvas.getContext('2d');
        const history = new Array(100).fill(60);

        function drawHistory() {
            histCtx.clearRect(0, 0, 280, 60);
            histCtx.strokeStyle = '#44ff88';
            histCtx.lineWidth = 1.5;
            histCtx.beginPath();
            history.forEach((v, i) => {
                const x = i * 2.8;
                const y = 60 - Math.min(v / 120 * 60, 60);
                i === 0 ? histCtx.moveTo(x, y) : histCtx.lineTo(x, y);
            });
            histCtx.stroke();

            // 60fps line
            histCtx.strokeStyle = '#2244aa';
            histCtx.lineWidth = 1;
            histCtx.setLineDash([4, 4]);
            histCtx.beginPath();
            histCtx.moveTo(0, 30); histCtx.lineTo(280, 30);
            histCtx.stroke();
            histCtx.setLineDash([]);
        }

        function classify(val, good, warn) {
            if (val <= good) return 'good';
            if (val <= warn) return 'warn';
            return 'bad';
        }

        let fpsFrames = 0, fpsLast = performance.now();

        window.addEventListener('resize', () => {
            const dw = window.innerWidth - 320;
            camera.aspect = dw / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(dw, window.innerHeight);
        });

        function animate() {
            requestAnimationFrame(animate);
            controls.update();

            fpsFrames++;
            const now = performance.now();
            if (now - fpsLast >= 500) {
                const fps = Math.round(fpsFrames * 1000 / (now - fpsLast));
                const info = renderer.info;
                const draws = info.render.calls;

                history.push(fps); history.shift();
                drawHistory();

                document.getElementById('valFPS').textContent = fps;
                document.getElementById('valDraws').textContent = draws;
                document.getElementById('valTris').textContent = info.render.triangles.toLocaleString();
                document.getElementById('valGeos').textContent = info.memory.geometries;
                document.getElementById('valTexs').textContent = info.memory.textures;
                document.getElementById('valObjs').textContent = scene.children.length;

                // Classify
                const fpsEl = document.getElementById('metFPS');
                fpsEl.className = 'metric ' + (fps >= 55 ? 'good' : fps >= 30 ? 'warn' : 'bad');
                const drEl = document.getElementById('metDraws');
                drEl.className = 'metric ' + classify(draws, 100, 500);

                // Bars
                document.getElementById('barFPS').style.width = Math.min(fps / 120 * 100, 100) + '%';
                document.getElementById('barFPS').className = 'bar ' + (fps >= 55 ? 'green' : fps >= 30 ? 'yellow' : 'red');
                document.getElementById('barDraws').style.width = Math.min(draws / 1000 * 100, 100) + '%';
                document.getElementById('barDraws').className = 'bar ' + classify(draws, 100, 500).replace('good','green').replace('warn','yellow').replace('bad','red');

                fpsFrames = 0; fpsLast = now;
            }

            renderer.render(scene, camera);
        }
        animate();
    </script>
</body>
</html>
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Texture Atlas Tool
สร้าง tool ที่:
- รับ image files หลายไฟล์
- Pack เป็น atlas อัตโนมัติ
- Output atlas image + UV data JSON

### แบบฝึกหัดที่ 2: Material Optimizer
สร้าง class ที่:
- scan scene หาซ้ำ materials
- Merge identical materials
- รายงาน จำนวน programs ที่ลดลง

### แบบฝึกหัดที่ 3: Draw Call Budget
สร้าง real-time budget system:
- กำหนด budget (200 draw calls)
- เตือนเมื่อเกิน
- แสดง top offenders

### แบบฝึกหัดที่ 4: Batch Manager
สร้าง BatchManager ที่:
- รับ objects ใหม่ทุก frame
- Sort ตาม material
- Merge static objects อัตโนมัติ

---

**สรุปบทที่ 61:**
- renderer.info.render.calls เป็น metric หลักสำหรับ performance
- Material sharing ลด shader state changes
- Geometry sharing ลด GPU memory usage
- Texture atlas ลด texture binding
- InstancedMesh เหมาะกับ many identical objects
- Merge geometry เหมาะกับ static complex environments
- Target: mobile < 200 draws, desktop < 2000 draws

**บทต่อไป:** Memory Management & Disposal
