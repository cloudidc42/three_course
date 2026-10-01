# Part 63: WebGL Profiling & Debugging

## เครื่องมือ Profiling ที่ควรรู้จัก

### 1. stats.js
```html
<!-- Import stats.js -->
<script src="https://cdn.jsdelivr.net/npm/stats.js@0.17.0/build/stats.min.js"></script>
```

```javascript
import Stats from 'three/examples/jsm/libs/stats.module.js';
// หรือ CDN: https://cdn.jsdelivr.net/npm/stats.js

const stats = new Stats();

// Panel modes:
// 0 = FPS (default)
// 1 = MS (milliseconds per frame)
// 2 = MB (memory usage)
stats.showPanel(0);
document.body.appendChild(stats.dom);

// ใน render loop
function animate() {
    stats.begin();
    // ... render code ...
    stats.end();
    
    requestAnimationFrame(animate);
}
```

### 2. performance.now() Timing

```javascript
// วัดเวลา specific operations
function measureTime(label, fn) {
    const start = performance.now();
    fn();
    const elapsed = performance.now() - start;
    console.log(`${label}: ${elapsed.toFixed(3)}ms`);
    return elapsed;
}

// ใช้งาน
measureTime('BVH Build', () => {
    geo.boundsTree = new MeshBVH(geo);
});

// วัด multiple operations
class Timer {
    constructor() {
        this.marks = {};
        this.measurements = {};
    }
    
    mark(name) {
        this.marks[name] = performance.now();
    }
    
    measure(name, startMark, endMark) {
        const end = endMark ? this.marks[endMark] : performance.now();
        const start = this.marks[startMark] || 0;
        this.measurements[name] = end - start;
        return this.measurements[name];
    }
    
    report() {
        console.table(this.measurements);
    }
}

const timer = new Timer();
timer.mark('start');
buildScene();
timer.mark('scene_done');
buildBVH();
timer.mark('bvh_done');
timer.measure('Scene Build', 'start', 'scene_done');
timer.measure('BVH Build', 'scene_done', 'bvh_done');
timer.report();
```

### 3. renderer.info (ทบทวน)

```javascript
// Detailed stats ที่ renderer.info ให้
function getDetailedStats(renderer) {
    const i = renderer.info;
    return {
        frame: {
            calls: i.render.calls,
            triangles: i.render.triangles,
            points: i.render.points,
            lines: i.render.lines
        },
        memory: {
            geometries: i.memory.geometries,
            textures: i.memory.textures
        },
        programs: i.programs.map(p => ({
            name: p.name,
            usedTimes: p.usedTimes,
            diagnostics: p.diagnostics
        }))
    };
}
```

## WebGL Error Checking

```javascript
// ตรวจสอบ WebGL errors
class GLErrorChecker {
    constructor(renderer) {
        this.gl = renderer.getContext();
        this.enabled = true;
    }
    
    check(label = '') {
        if (!this.enabled) return;
        
        const gl = this.gl;
        const err = gl.getError();
        
        if (err !== gl.NO_ERROR) {
            const errorNames = {
                [gl.INVALID_ENUM]: 'INVALID_ENUM',
                [gl.INVALID_VALUE]: 'INVALID_VALUE',
                [gl.INVALID_OPERATION]: 'INVALID_OPERATION',
                [gl.OUT_OF_MEMORY]: 'OUT_OF_MEMORY',
                [gl.CONTEXT_LOST_WEBGL]: 'CONTEXT_LOST',
                [gl.INVALID_FRAMEBUFFER_OPERATION]: 'INVALID_FRAMEBUFFER_OPERATION'
            };
            
            console.error(`WebGL Error${label ? ' at ' + label : ''}: ${errorNames[err] || err}`);
        }
    }
    
    // Wrap renderer render call
    wrapRender(renderer, scene, camera) {
        renderer.render(scene, camera);
        this.check('render');
    }
}

// Context Lost Handling
renderer.domElement.addEventListener('webglcontextlost', (e) => {
    console.error('WebGL context lost!');
    e.preventDefault();
    // หยุด animation loop
});

renderer.domElement.addEventListener('webglcontextrestored', () => {
    console.log('WebGL context restored');
    // Re-initialize resources
    init();
});
```

## DEBUG Mode Helpers

```javascript
// Visual debug helpers
class DebugHelpers {
    constructor(scene) {
        this.scene = scene;
        this.helpers = [];
        this.enabled = true;
    }
    
    addBoundingBox(object, color = 0xffff00) {
        const helper = new THREE.BoxHelper(object, color);
        this.scene.add(helper);
        this.helpers.push(helper);
        return helper;
    }
    
    addAxes(size = 1) {
        const helper = new THREE.AxesHelper(size);
        this.scene.add(helper);
        this.helpers.push(helper);
        return helper;
    }
    
    addCameraHelper(camera) {
        const helper = new THREE.CameraHelper(camera);
        this.scene.add(helper);
        this.helpers.push(helper);
        return helper;
    }
    
    addLightHelper(light) {
        let helper;
        if (light instanceof THREE.DirectionalLight) {
            helper = new THREE.DirectionalLightHelper(light, 5);
        } else if (light instanceof THREE.PointLight) {
            helper = new THREE.PointLightHelper(light, 1);
        } else if (light instanceof THREE.SpotLight) {
            helper = new THREE.SpotLightHelper(light);
        }
        if (helper) {
            this.scene.add(helper);
            this.helpers.push(helper);
        }
        return helper;
    }
    
    addNormalHelper(mesh, size = 0.1, color = 0xff0000) {
        const helper = new THREE.VertexNormalsHelper(mesh, size, color);
        this.scene.add(helper);
        this.helpers.push(helper);
        return helper;
    }
    
    showWireframe(mesh) {
        const wireframe = new THREE.WireframeHelper
            ? new THREE.WireframeHelper(mesh, 0x00ff00)
            : new THREE.Mesh(mesh.geometry, new THREE.MeshBasicMaterial({ wireframe: true, color: 0x00ff00 }));
        this.scene.add(wireframe);
        this.helpers.push(wireframe);
        return wireframe;
    }
    
    toggle() {
        this.enabled = !this.enabled;
        this.helpers.forEach(h => h.visible = this.enabled);
    }
    
    clear() {
        this.helpers.forEach(h => {
            this.scene.remove(h);
            if (h.geometry) h.geometry.dispose();
            if (h.material) h.material.dispose();
        });
        this.helpers = [];
    }
    
    update() {
        // helpers บางตัวต้อง update ทุก frame
        this.helpers.forEach(h => {
            if (h.update) h.update();
        });
    }
}
```

## Draw Call Inspector

```javascript
// ตรวจสอบ draw calls ว่ามาจากไหน
class DrawCallInspector {
    constructor(renderer) {
        this.renderer = renderer;
        this.callStack = [];
    }
    
    // Monkey-patch render function เพื่อ track
    install() {
        const origRender = this.renderer.render.bind(this.renderer);
        const self = this;
        
        this.renderer.render = function(scene, camera) {
            const before = self.renderer.info.render.calls;
            origRender(scene, camera);
            const after = self.renderer.info.render.calls;
            self._lastCallCount = after - before;
        };
    }
    
    // Scan scene หา expensive objects
    scan(scene) {
        const report = [];
        
        scene.traverse(obj => {
            if (obj instanceof THREE.Mesh) {
                const geo = obj.geometry;
                const tri = geo.index 
                    ? geo.index.count / 3 
                    : geo.attributes.position.count / 3;
                
                report.push({
                    name: obj.name || obj.uuid.slice(0,8),
                    triangles: Math.floor(tri),
                    visible: obj.visible,
                    frustumCulled: obj.frustumCulled,
                    renderOrder: obj.renderOrder
                });
            }
        });
        
        // Sort by triangle count
        report.sort((a, b) => b.triangles - a.triangles);
        
        return report;
    }
}
```

## Shader Compilation Profiling

```javascript
// วัดเวลา shader compilation
function profileShaderCompilation(renderer, material) {
    const start = performance.now();
    
    // Force shader compile โดยสร้าง dummy render
    const dummyGeo = new THREE.PlaneGeometry(0.001, 0.001);
    const dummy = new THREE.Mesh(dummyGeo, material);
    const dummyScene = new THREE.Scene();
    dummyScene.add(dummy);
    const dummyCamera = new THREE.PerspectiveCamera();
    renderer.compile(dummyScene, dummyCamera);
    
    const elapsed = performance.now() - start;
    
    // Cleanup
    dummyGeo.dispose();
    
    return elapsed;
}
```

## ตัวอย่าง: Performance Monitoring Dashboard

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Performance Dashboard</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #070710; overflow: hidden; font-family: 'Courier New', monospace; }
        canvas { display: block; }
        #dash {
            position: fixed; top: 0; right: 0;
            width: 340px; height: 100vh;
            background: rgba(3,5,12,0.97);
            border-left: 1px solid #1a3366;
            padding: 14px;
            overflow-y: auto;
        }
        .section { margin-bottom: 14px; }
        h2 { color: #4488ff; font-size: 13px; border-bottom: 1px solid #1a3366; padding-bottom: 6px; margin-bottom: 8px; }
        .row { display: flex; justify-content: space-between; padding: 3px 6px; font-size: 12px; margin: 2px 0; background: rgba(255,255,255,0.02); }
        .row .l { color: #6688aa; }
        .row .v { color: #ffcc44; }
        .graph { background: #0a0a14; border: 1px solid #1a3366; border-radius: 4px; margin: 6px 0; overflow: hidden; }
        .graph canvas { display: block; }
        .btn-row { display: flex; gap: 4px; flex-wrap: wrap; }
        .btn {
            flex: 1; padding: 5px; font-size: 11px;
            background: transparent; cursor: pointer;
            border-radius: 3px; font-family: inherit;
            min-width: 60px;
        }
        .btn.blue { color: #4488ff; border: 1px solid #2255aa; }
        .btn.green { color: #44ff88; border: 1px solid #228844; }
        .btn.red { color: #ff4444; border: 1px solid #882222; }
        .btn.active { background: rgba(34,85,170,0.3); }
        #profLog {
            background: #060a14; border: 1px solid #1a3366;
            border-radius: 4px; padding: 6px;
            height: 120px; overflow-y: auto;
            font-size: 11px; line-height: 1.6;
        }
        .l-info { color: #4488ff; }
        .l-good { color: #44ff88; }
        .l-warn { color: #ffcc44; }
        .l-bad { color: #ff4444; }
        .badge { display: inline-block; padding: 1px 6px; border-radius: 3px; font-size: 10px; margin-left: 4px; }
        .badge.g { background: rgba(68,255,136,0.2); color: #44ff88; }
        .badge.y { background: rgba(255,204,68,0.2); color: #ffcc44; }
        .badge.r { background: rgba(255,68,68,0.2); color: #ff4444; }
    </style>
</head>
<body>
    <div id="dash">
        <div class="section">
            <h2>📊 Real-time Metrics</h2>
            <div class="row"><span class="l">FPS</span><span class="v" id="mFPS">--</span></div>
            <div class="row"><span class="l">Frame Time (ms)</span><span class="v" id="mFT">--</span></div>
            <div class="row"><span class="l">Draw Calls</span><span class="v" id="mDC">--</span></div>
            <div class="row"><span class="l">Triangles</span><span class="v" id="mTri">--</span></div>
            <div class="row"><span class="l">GPU Geometries</span><span class="v" id="mGeo">--</span></div>
            <div class="row"><span class="l">GPU Textures</span><span class="v" id="mTex">--</span></div>
            <div class="row"><span class="l">Shader Programs</span><span class="v" id="mProg">--</span></div>
        </div>

        <div class="section">
            <h2>📈 FPS Graph</h2>
            <div class="graph">
                <canvas id="fpsGraph" width="310" height="60"></canvas>
            </div>
            <h2>📈 Draw Calls Graph</h2>
            <div class="graph">
                <canvas id="dcGraph" width="310" height="60"></canvas>
            </div>
        </div>

        <div class="section">
            <h2>🔧 Scene Controls</h2>
            <div class="btn-row">
                <button class="btn blue active" id="bAdd">+ Add Objects</button>
                <button class="btn red" id="bRemove">- Remove All</button>
                <button class="btn green" id="bDebug">Debug Helpers</button>
            </div>
            <div style="margin-top:6px">
                <div class="row"><span class="l">Objects</span><span class="v" id="mObjs">0</span></div>
            </div>
        </div>

        <div class="section">
            <h2>🔍 Profiler Log</h2>
            <div id="profLog"></div>
        </div>
    </div>

    <script type="module">
        import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js';
        import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/controls/OrbitControls.js';

        const DW = window.innerWidth - 340, H = window.innerHeight;
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(DW, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        renderer.domElement.style.position = 'fixed';
        document.body.appendChild(renderer.domElement);

        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x070710);
        const camera = new THREE.PerspectiveCamera(70, DW/H, 0.5, 300);
        camera.position.set(0, 20, 50);
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.autoRotate = true;
        controls.autoRotateSpeed = 0.5;

        scene.add(new THREE.AmbientLight(0x334466, 1));
        const sun = new THREE.DirectionalLight(0xffffff, 2);
        sun.position.set(30, 60, 30);
        scene.add(sun);
        scene.add(new THREE.GridHelper(100, 40, 0x1a2233, 0x111822));

        const objects = [];
        const helpers = [];

        const geos = [
            new THREE.BoxGeometry(1, 1, 1),
            new THREE.SphereGeometry(0.5, 8, 6),
            new THREE.ConeGeometry(0.5, 1, 8),
            new THREE.CylinderGeometry(0.3, 0.3, 1, 8),
            new THREE.TorusGeometry(0.4, 0.15, 8, 12)
        ];
        const sharedMat = new THREE.MeshStandardMaterial({ color: 0x4488cc, roughness: 0.5, metalness: 0.4 });

        const profLog = document.getElementById('profLog');
        function plog(msg, type = 'info') {
            const d = document.createElement('div');
            d.className = `l-${type}`;
            d.innerHTML = `<span style="color:#556">${new Date().toLocaleTimeString()}</span> ${msg}`;
            profLog.appendChild(d);
            profLog.scrollTop = profLog.scrollHeight;
        }

        let debugMode = false;

        document.getElementById('bAdd').addEventListener('click', () => {
            const t0 = performance.now();
            const count = 50;
            for (let i = 0; i < count; i++) {
                const geo = geos[Math.floor(Math.random() * geos.length)];
                const mesh = new THREE.Mesh(geo, sharedMat);
                mesh.position.set(
                    (Math.random()-0.5)*40, 0.5 + Math.random()*4, (Math.random()-0.5)*40
                );
                mesh.rotation.set(Math.random()*Math.PI, Math.random()*Math.PI, 0);
                scene.add(mesh);
                objects.push(mesh);
            }
            const t1 = performance.now();
            document.getElementById('mObjs').textContent = objects.length;
            plog(`Added ${count} objects in ${(t1-t0).toFixed(2)}ms`, 'good');
        });

        document.getElementById('bRemove').addEventListener('click', () => {
            const n = objects.length;
            objects.forEach(m => scene.remove(m));
            objects.length = 0;
            document.getElementById('mObjs').textContent = 0;
            plog(`Removed ${n} objects (geometries shared — not disposed)`, 'warn');
        });

        document.getElementById('bDebug').addEventListener('click', (e) => {
            debugMode = !debugMode;
            e.target.textContent = `Debug: ${debugMode ? 'ON' : 'OFF'}`;
            e.target.classList.toggle('active', debugMode);

            helpers.forEach(h => { scene.remove(h); });
            helpers.length = 0;

            if (debugMode) {
                // Axes helper
                const axes = new THREE.AxesHelper(5);
                scene.add(axes); helpers.push(axes);

                // Light helper
                const lh = new THREE.DirectionalLightHelper(sun, 5, 0xffff00);
                scene.add(lh); helpers.push(lh);

                // Camera frustum
                const ch = new THREE.CameraHelper(camera);
                scene.add(ch); helpers.push(ch);

                plog('Debug helpers added: Axes, Light, Frustum', 'info');
            } else {
                plog('Debug helpers removed', 'info');
            }
        });

        // Graph rendering
        const fpsGraphCanvas = document.getElementById('fpsGraph');
        const dcGraphCanvas = document.getElementById('dcGraph');
        const fpsCtx = fpsGraphCanvas.getContext('2d');
        const dcCtx = dcGraphCanvas.getContext('2d');
        const fpsData = new Array(100).fill(60);
        const dcData = new Array(100).fill(0);

        function drawGraph(ctx, data, maxVal, color, label, w = 310, h = 60) {
            ctx.clearRect(0, 0, w, h);

            // Grid
            ctx.strokeStyle = '#1a2233';
            ctx.lineWidth = 1;
            ctx.setLineDash([2, 4]);
            for (let i = 0; i < 4; i++) {
                const y = (h / 4) * i;
                ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(w, y); ctx.stroke();
            }
            ctx.setLineDash([]);

            // Line
            ctx.strokeStyle = color;
            ctx.lineWidth = 1.5;
            ctx.beginPath();
            data.forEach((v, i) => {
                const x = i * (w / data.length);
                const y = h - Math.min(v / maxVal * h, h);
                i === 0 ? ctx.moveTo(x, y) : ctx.lineTo(x, y);
            });
            ctx.stroke();

            // Fill
            ctx.fillStyle = color + '22';
            ctx.beginPath();
            data.forEach((v, i) => {
                const x = i * (w / data.length);
                const y = h - Math.min(v / maxVal * h, h);
                i === 0 ? ctx.moveTo(x, y) : ctx.lineTo(x, y);
            });
            ctx.lineTo(w, h); ctx.lineTo(0, h);
            ctx.fill();

            // Label
            ctx.fillStyle = color;
            ctx.font = '10px monospace';
            ctx.fillText(`${label}: ${data[data.length-1]}`, 4, 12);
        }

        let fpsLast = performance.now(), fpsFrames = 0;
        let lastFrameTime = performance.now();

        window.addEventListener('resize', () => {
            const dw = window.innerWidth - 340;
            camera.aspect = dw / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(dw, window.innerHeight);
        });

        function animate() {
            requestAnimationFrame(animate);
            controls.update();

            if (debugMode) helpers.forEach(h => h.update && h.update());

            fpsFrames++;
            const now = performance.now();
            const ft = now - lastFrameTime;
            lastFrameTime = now;

            if (now - fpsLast >= 500) {
                const fps = Math.round(fpsFrames * 1000 / (now - fpsLast));
                const info = renderer.info;

                fpsData.push(fps); fpsData.shift();
                dcData.push(info.render.calls); dcData.shift();

                drawGraph(fpsCtx, fpsData, 120, '#44ff88', 'FPS');
                drawGraph(dcCtx, dcData, Math.max(500, Math.max(...dcData) * 1.2), '#4488ff', 'Draws');

                document.getElementById('mFPS').textContent = fps;
                document.getElementById('mFT').textContent = ft.toFixed(2);
                document.getElementById('mDC').textContent = info.render.calls;
                document.getElementById('mTri').textContent = info.render.triangles.toLocaleString();
                document.getElementById('mGeo').textContent = info.memory.geometries;
                document.getElementById('mTex').textContent = info.memory.textures;
                document.getElementById('mProg').textContent = info.programs.length;

                fpsFrames = 0; fpsLast = now;
            }

            renderer.render(scene, camera);
        }
        animate();
        plog('Dashboard initialized. Click "Add Objects" to start.', 'info');
    </script>
</body>
</html>
```

## Spector.js Integration

```javascript
// Spector.js: WebGL frame capture tool
// ติดตั้ง: npm install spectorjs
// หรือ Chrome Extension: "Spector.js"

// Programmatic capture
import { Spector } from 'spectorjs';

const spector = new Spector();

// Capture ใน production
function captureFrame() {
    spector.startCapture(renderer.domElement, 100);
    renderer.render(scene, camera);
    spector.stopCapture();
    
    spector.onCapture.add((capture) => {
        console.log('Frame captured:', capture);
        spector.displayUI(); // เปิด UI ดู capture
    });
}

// Capture เมื่อกด keyboard
document.addEventListener('keydown', (e) => {
    if (e.key === 'F12') captureFrame();
});
```

## Chrome DevTools GPU Profiling

```javascript
// เปิดใช้ Chrome Tracing เพื่อดู GPU timeline
// 1. เปิด Chrome DevTools
// 2. ไปที่ Performance tab
// 3. เปิด "Show advanced rendering statistics" (gear icon)
// 4. Record แล้วดู GPU track

// หรือใช้ chrome://tracing สำหรับ advanced profiling

// GPU Timing ด้วย EXT_disjoint_timer_query_webgl2
function setupGPUTiming(renderer) {
    const gl = renderer.getContext();
    const ext = gl.getExtension('EXT_disjoint_timer_query_webgl2');
    
    if (!ext) {
        console.warn('GPU timing not available');
        return null;
    }
    
    return {
        begin() {
            const query = gl.createQuery();
            gl.beginQuery(ext.TIME_ELAPSED_EXT, query);
            return query;
        },
        end(query) {
            gl.endQuery(ext.TIME_ELAPSED_EXT);
            return query;
        },
        result(query) {
            const available = gl.getQueryParameter(query, gl.QUERY_RESULT_AVAILABLE);
            if (available) {
                const timeNs = gl.getQueryParameter(query, gl.QUERY_RESULT);
                gl.deleteQuery(query);
                return timeNs / 1e6; // convert to ms
            }
            return null;
        }
    };
}
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Custom Stats Panel
สร้าง stats panel ที่ใช้งานจริง:
- FPS + Frame time
- Draw calls vs Budget
- GPU memory ที่ใช้
- Alert เมื่อ performance ต่ำ

### แบบฝึกหัดที่ 2: Shader Hot Reload
สร้าง dev tool ที่:
- Monitor shader file changes
- Recompile shader ใหม่ auto
- Show compilation errors ใน overlay

### แบบฝึกหัดที่ 3: Scene Inspector
สร้าง tree view ของ scene:
- แสดง hierarchy
- Click เพื่อ select object
- Highlight selected ด้วย bounding box

### แบบฝึกหัดที่ 4: Performance Report
สร้าง report generator ที่:
- Run scene เป็นเวลา 10 วินาที
- เก็บ metrics ทุก frame
- สร้าง HTML report พร้อม graphs

---

**สรุปบทที่ 63:**
- stats.js ให้ FPS/MS/MB panel ง่ายๆ
- performance.now() วัดเวลา operations ได้แม่น
- renderer.info เป็น built-in profiler สำหรับ WebGL
- THREE.BoxHelper, AxesHelper, CameraHelper ช่วย debug visual
- Spector.js จับภาพ WebGL frame ดู draw commands ทีละ call
- Chrome DevTools Performance tab ดู CPU/GPU timeline

---

## สรุป Part 49-63: Advanced Three.js

ตลอด 15 บทที่ผ่านมาเราได้เรียนรู้:

| บท | หัวข้อ | เทคนิคหลัก |
|----|--------|------------|
| 49 | Depth of Field | BokehPass, CoC, tilt-shift |
| 50 | Motion Blur | VelocityBuffer, radial blur, camera shake |
| 51 | SSAO | SSAOPass, HBAO, baked AO |
| 52 | SSR | SSRPass, Reflector, ray marching |
| 53 | Volumetric Lighting | God rays, radial blur, spotlight cone |
| 54 | Toon Shading | MeshToonMaterial, quantize(), halftone |
| 55 | Outlines | OutlinePass, stencil buffer, X-ray |
| 56 | Normal/POM | Tangent space, parallax occlusion |
| 57 | LOD | THREE.LOD, billboarding, terrain LOD |
| 58 | Instancing | InstancedMesh, 1M objects, crowd sim |
| 59 | Merging | mergeGeometries, city builder, batching |
| 60 | Frustum Culling | Manual frustum, BVH, spatial partition |
| 61 | Draw Calls | renderer.info, atlas, optimization |
| 62 | Memory | dispose(), object pool, leak detection |
| 63 | Profiling | stats.js, DevTools, GPU timing |

**ขั้นต่อไป:** Physics Integration (Cannon.js / Rapier), Advanced Shaders, and Scene Graph Architecture
