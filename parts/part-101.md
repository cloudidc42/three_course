# Part 101: Performance Profiling & Chrome DevTools

## บทนำ

Performance optimization เริ่มต้นที่การ measurement ที่แม่นยำ "อย่าเดา, ให้วัด" — นี่คือหลักการหลักของ performance engineering ในบทนี้เราจะเรียนรู้เครื่องมือทุกชนิดสำหรับการวิเคราะห์และ optimize Three.js applications

## renderer.info — ตัวชี้วัดหลัก

`renderer.info` เป็นจุดเริ่มต้นที่ดีที่สุดสำหรับการ profile Three.js:

```javascript
// ดู stats ทุก frame
function logRendererInfo() {
    const info = renderer.info;
    
    console.log({
        // Draw calls: ยิ่งน้อยยิ่งดี (target < 100 สำหรับ real-time)
        drawCalls: info.render.calls,
        
        // Triangle count
        triangles: info.render.triangles,
        
        // Point/Line counts
        points: info.render.points,
        lines: info.render.lines,
        
        // GPU memory usage
        geometries: info.memory.geometries,
        textures: info.memory.textures,
        
        // Program count (compiled shaders)
        programs: info.programs?.length || 0,
    });
}
```

### ค่าที่ควรตรวจสอบ

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| Draw Calls | < 100 | 100-500 | > 500 |
| Triangles | < 500K | 500K-2M | > 2M |
| Textures | < 50 | 50-100 | > 100 |

```javascript
// Display renderer.info แบบ real-time
class RendererStats {
    constructor(renderer) {
        this.renderer = renderer;
        this.panel = this.createPanel();
    }
    
    createPanel() {
        const div = document.createElement('div');
        div.style.cssText = `
            position: fixed; top: 0; left: 0;
            background: rgba(0,0,0,0.7);
            color: #00ff00;
            font-family: monospace; font-size: 11px;
            padding: 8px 12px; z-index: 9999;
            line-height: 1.5; min-width: 180px;
        `;
        document.body.appendChild(div);
        return div;
    }
    
    update() {
        const info = this.renderer.info;
        const r = info.render;
        const m = info.memory;
        
        const getColor = (val, warn, critical) => {
            if (val > critical) return '#ff4444';
            if (val > warn) return '#ffaa22';
            return '#44ff44';
        };
        
        this.panel.innerHTML = `
            <div>Draw Calls: <span style="color:${getColor(r.calls,100,500)}">${r.calls}</span></div>
            <div>Triangles: <span style="color:${getColor(r.triangles,500000,2000000)}">${(r.triangles/1000).toFixed(1)}K</span></div>
            <div>Textures: <span style="color:${getColor(m.textures,50,100)}">${m.textures}</span></div>
            <div>Geometries: <span style="color:#aaffaa">${m.geometries}</span></div>
            <div>Programs: <span style="color:#aaaaff">${info.programs?.length || 0}</span></div>
        `;
        
        // Reset after reading
        this.renderer.info.reset();
    }
}
```

## Stats.js Integration

Stats.js เป็น library ยอดนิยมสำหรับ FPS/MS/MB monitoring:

```javascript
// Install: npm install stats.js
// หรือใช้ CDN

import Stats from 'https://cdn.jsdelivr.net/npm/stats.js@0.17.0/build/stats.min.js';

const stats = new Stats();
stats.showPanel(0); // 0: FPS, 1: MS, 2: MB
document.body.appendChild(stats.dom);

// ใน animate loop:
function animate() {
    stats.begin(); // เริ่มจับเวลา
    
    requestAnimationFrame(animate);
    renderer.render(scene, camera);
    
    stats.end(); // สิ้นสุดการจับเวลา
}

// เพิ่ม custom panel
const customPanel = stats.addPanel(new Stats.Panel('DrawCalls', '#ff8', '#221'));
stats.showPanel(3); // แสดง custom panel

// อัพเดตใน loop
customPanel.update(renderer.info.render.calls, 200);
```

## Custom Performance Monitor

```javascript
class PerformanceMonitor {
    constructor() {
        this.metrics = {
            fps: [],
            frameTime: [],
            drawCalls: [],
            triangles: [],
            memoryGeo: [],
            memoryTex: [],
        };
        
        this.maxSamples = 120; // 2 seconds at 60fps
        this.lastTime = performance.now();
        this.frameCount = 0;
        this.fpsAccum = 0;
        
        this.createUI();
    }
    
    createUI() {
        this.container = document.createElement('div');
        this.container.style.cssText = `
            position: fixed; top: 50px; left: 0;
            background: rgba(0,0,10,0.85);
            border-right: 1px solid rgba(100,100,255,0.3);
            padding: 12px;
            font-family: 'Courier New', monospace;
            font-size: 11px;
            z-index: 9999;
            min-width: 220px;
        `;
        
        this.rows = {};
        const metrics = [
            { key: 'fps', label: 'FPS', format: v => Math.round(v), good: 55, bad: 30 },
            { key: 'frameTime', label: 'Frame Time', format: v => v.toFixed(2) + 'ms', good: 18, bad: 33 },
            { key: 'drawCalls', label: 'Draw Calls', format: v => Math.round(v), good: 100, bad: 400 },
            { key: 'triangles', label: 'Triangles', format: v => (v/1000).toFixed(0) + 'K', good: 200, bad: 1000 },
            { key: 'memoryTex', label: 'Textures', format: v => Math.round(v), good: 30, bad: 80 },
        ];
        
        metrics.forEach(({ key, label }) => {
            const row = document.createElement('div');
            row.style.cssText = 'display:flex;justify-content:space-between;padding:2px 0;border-bottom:1px solid rgba(255,255,255,0.05)';
            row.innerHTML = `<span style="color:rgba(255,255,255,0.4)">${label}</span><span id="pm-${key}">-</span>`;
            this.container.appendChild(row);
            this.rows[key] = document.getElementById(`pm-${key}`);
        });
        
        // Sparkline canvas
        this.sparkCanvas = document.createElement('canvas');
        this.sparkCanvas.width = 220;
        this.sparkCanvas.height = 40;
        this.sparkCanvas.style.marginTop = '8px';
        this.container.appendChild(this.sparkCanvas);
        
        // Bottleneck indicator
        this.bottleneckEl = document.createElement('div');
        this.bottleneckEl.style.cssText = 'margin-top:8px;font-size:10px;color:#aaa;';
        this.container.appendChild(this.bottleneckEl);
        
        document.body.appendChild(this.container);
    }
    
    record(renderer) {
        const now = performance.now();
        const dt = now - this.lastTime;
        this.lastTime = now;
        
        this.fpsAccum += 1000 / dt;
        this.frameCount++;
        
        if (this.frameCount % 10 === 0) {
            const avgFPS = this.fpsAccum / 10;
            this.fpsAccum = 0;
            
            const info = renderer.info;
            
            const newMetrics = {
                fps: avgFPS,
                frameTime: dt,
                drawCalls: info.render.calls,
                triangles: info.render.triangles,
                memoryTex: info.memory.textures,
                memoryGeo: info.memory.geometries,
            };
            
            // Push and trim
            Object.entries(newMetrics).forEach(([key, value]) => {
                this.metrics[key].push(value);
                if (this.metrics[key].length > this.maxSamples) {
                    this.metrics[key].shift();
                }
            });
            
            this.updateUI(newMetrics);
            this.analyzeBottleneck(newMetrics);
        }
    }
    
    updateUI(current) {
        const getColor = (val, good, bad, reversed = false) => {
            if (reversed) [good, bad] = [bad, good]; // For FPS (higher is better)
            if (val >= good) return '#44ff44';
            if (val >= (good + bad) / 2) return '#ffaa22';
            return '#ff4444';
        };
        
        if (this.rows.fps) {
            this.rows.fps.textContent = Math.round(current.fps);
            this.rows.fps.style.color = getColor(current.fps, 55, 30, true);
        }
        if (this.rows.frameTime) {
            this.rows.frameTime.textContent = current.frameTime.toFixed(1) + 'ms';
            this.rows.frameTime.style.color = getColor(current.frameTime, 18, 33);
        }
        if (this.rows.drawCalls) {
            this.rows.drawCalls.textContent = current.drawCalls;
            this.rows.drawCalls.style.color = getColor(current.drawCalls, 100, 400);
        }
        if (this.rows.triangles) {
            this.rows.triangles.textContent = (current.triangles / 1000).toFixed(0) + 'K';
            this.rows.triangles.style.color = getColor(current.triangles, 200000, 1000000);
        }
        if (this.rows.memoryTex) {
            this.rows.memoryTex.textContent = current.memoryTex;
        }
        
        // Draw FPS sparkline
        this.drawSparkline(this.metrics.fps, 0, 75, '#44ff44');
    }
    
    drawSparkline(data, min, max, color) {
        const ctx = this.sparkCanvas.getContext('2d');
        const w = this.sparkCanvas.width;
        const h = this.sparkCanvas.height;
        
        ctx.clearRect(0, 0, w, h);
        
        // Background
        ctx.fillStyle = 'rgba(0,0,20,0.5)';
        ctx.fillRect(0, 0, w, h);
        
        // Target line (60 FPS)
        const targetY = h - ((60 - min) / (max - min)) * h;
        ctx.strokeStyle = 'rgba(255,255,255,0.1)';
        ctx.beginPath();
        ctx.moveTo(0, targetY);
        ctx.lineTo(w, targetY);
        ctx.stroke();
        
        // Data line
        if (data.length < 2) return;
        
        ctx.strokeStyle = color;
        ctx.lineWidth = 1.5;
        ctx.beginPath();
        
        data.forEach((v, i) => {
            const x = (i / (this.maxSamples - 1)) * w;
            const y = h - ((Math.max(min, Math.min(max, v)) - min) / (max - min)) * h;
            if (i === 0) ctx.moveTo(x, y);
            else ctx.lineTo(x, y);
        });
        ctx.stroke();
        
        // Label
        ctx.fillStyle = 'rgba(255,255,255,0.3)';
        ctx.font = '9px monospace';
        ctx.fillText('FPS', 4, 12);
        ctx.fillText('60', w - 18, targetY - 2);
    }
    
    analyzeBottleneck(metrics) {
        let hint = '';
        
        if (metrics.fps < 30) {
            if (metrics.drawCalls > 200) {
                hint = '⚠️ Draw calls สูง → ใช้ Merge / Instancing';
            } else if (metrics.triangles > 500000) {
                hint = '⚠️ Polygon สูง → ลด geometry หรือใช้ LOD';
            } else {
                hint = '⚠️ อาจเป็น CPU-bound → ตรวจสอบ JavaScript';
            }
        } else if (metrics.frameTime > 25) {
            hint = '📊 Frame time สูง แต่ FPS ยังโอเค';
        } else {
            hint = '✅ Performance ดี';
        }
        
        this.bottleneckEl.textContent = hint;
    }
    
    toggle() {
        this.container.style.display =
            this.container.style.display === 'none' ? 'block' : 'none';
    }
}
```

## CPU-Bound vs GPU-Bound

ทำความเข้าใจว่า bottleneck อยู่ที่ไหน:

```javascript
class BoundnessDetector {
    constructor() {
        this.cpuTimes = [];
        this.frameTimes = [];
        this.samples = 60;
    }
    
    startFrame() {
        this._frameStart = performance.now();
    }
    
    startCPU() {
        this._cpuStart = performance.now();
    }
    
    endCPU() {
        this.cpuTimes.push(performance.now() - this._cpuStart);
        if (this.cpuTimes.length > this.samples) this.cpuTimes.shift();
    }
    
    endFrame() {
        this.frameTimes.push(performance.now() - this._frameStart);
        if (this.frameTimes.length > this.samples) this.frameTimes.shift();
    }
    
    analyze() {
        if (this.frameTimes.length < 10) return 'gathering data...';
        
        const avgFrame = this.frameTimes.reduce((s, v) => s + v) / this.frameTimes.length;
        const avgCPU = this.cpuTimes.reduce((s, v) => s + v) / this.cpuTimes.length;
        
        const cpuRatio = avgCPU / avgFrame;
        
        if (cpuRatio > 0.8) {
            return `CPU-bound (${(cpuRatio * 100).toFixed(0)}% CPU time)\n→ ลด JavaScript computation`;
        } else if (avgFrame > 16.67) {
            return `GPU-bound (CPU ${(cpuRatio * 100).toFixed(0)}%)\n→ ลด draw calls หรือ polygons`;
        } else {
            return `ไม่มี bottleneck (${avgFrame.toFixed(1)}ms/frame)`;
        }
    }
}
```

## performance.mark/measure

```javascript
// Custom timing สำหรับ specific operations
function profiledUpdate() {
    performance.mark('update-start');
    
    // Physics
    performance.mark('physics-start');
    updatePhysics();
    performance.mark('physics-end');
    performance.measure('physics', 'physics-start', 'physics-end');
    
    // Animation
    performance.mark('animation-start');
    updateAnimations();
    performance.mark('animation-end');
    performance.measure('animation', 'animation-start', 'animation-end');
    
    performance.mark('update-end');
    performance.measure('total-update', 'update-start', 'update-end');
}

// อ่านผล
function getTimings() {
    const measures = performance.getEntriesByType('measure');
    const result = {};
    
    measures.forEach(m => {
        if (!result[m.name]) result[m.name] = [];
        result[m.name].push(m.duration);
    });
    
    // Average last 60 frames
    Object.entries(result).forEach(([name, times]) => {
        const recent = times.slice(-60);
        const avg = recent.reduce((s, v) => s + v) / recent.length;
        console.log(`${name}: ${avg.toFixed(2)}ms`);
    });
    
    performance.clearMeasures();
    performance.clearMarks();
}
```

## Chrome DevTools Performance Tab

```javascript
// เพิ่ม label ให้ profiler เห็นชัด
function profileFrame() {
    console.profile('Frame');
    
    // ...game logic...
    
    console.profileEnd('Frame');
}

// ใช้ performance.mark ให้ DevTools เห็น
performance.mark('Three.js Render Start');
renderer.render(scene, camera);
performance.mark('Three.js Render End');
performance.measure('Three.js Frame', 'Three.js Render Start', 'Three.js Render End');
```

## Demo หลัก: Performance Monitoring Dashboard

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Performance Profiling Dashboard</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #050510; overflow: hidden; font-family: 'Courier New', monospace; }
        canvas#canvas { display: block; }
        
        #perf-dashboard {
            position: fixed; top: 0; right: 0;
            width: 300px; height: 100%;
            background: rgba(5, 5, 20, 0.95);
            border-left: 1px solid rgba(100,100,255,0.2);
            padding: 12px;
            overflow-y: auto;
            z-index: 100;
            display: flex;
            flex-direction: column;
            gap: 12px;
        }
        
        .panel-title {
            font-size: 11px;
            color: rgba(100,150,255,0.7);
            text-transform: uppercase;
            letter-spacing: 0.1em;
            border-bottom: 1px solid rgba(100,100,255,0.2);
            padding-bottom: 6px;
            margin-bottom: 6px;
        }
        
        .metric-row {
            display: flex;
            justify-content: space-between;
            font-size: 12px;
            padding: 3px 0;
            color: rgba(255,255,255,0.5);
        }
        .metric-row .val { font-weight: bold; }
        .val.good { color: #44ff44; }
        .val.warn { color: #ffaa22; }
        .val.bad { color: #ff4444; }
        
        .sparkline-wrapper {
            background: rgba(0,0,30,0.5);
            border-radius: 4px;
            overflow: hidden;
        }
        .sparkline-label {
            font-size: 9px;
            color: rgba(255,255,255,0.3);
            padding: 3px 6px 0;
        }
        
        .bottleneck-box {
            background: rgba(20,20,50,0.7);
            border: 1px solid rgba(100,100,200,0.3);
            border-radius: 6px;
            padding: 10px;
            font-size: 11px;
            color: rgba(200,220,255,0.7);
            line-height: 1.6;
        }
        
        .test-controls {
            display: flex;
            flex-direction: column;
            gap: 6px;
        }
        .test-btn {
            background: rgba(30,30,60,0.8);
            border: 1px solid rgba(100,100,200,0.4);
            color: white;
            padding: 6px 10px;
            border-radius: 4px;
            cursor: pointer;
            font-size: 11px;
            font-family: inherit;
            text-align: left;
            transition: background 0.15s;
        }
        .test-btn:hover { background: rgba(50,50,100,0.8); }
        .test-btn.active { border-color: #ff4444; color: #ff8888; }
        
        #scene-info {
            font-size: 11px;
            color: rgba(255,255,255,0.4);
            line-height: 1.7;
        }
        
        .warning-log {
            max-height: 120px;
            overflow-y: auto;
            font-size: 10px;
            line-height: 1.6;
        }
        .warning-log .warn-item {
            color: #ffaa44;
            padding: 2px 0;
            border-bottom: 1px solid rgba(255,255,255,0.05);
        }
        .warning-log .warn-item.critical { color: #ff4444; }
    </style>
</head>
<body>
    <canvas id="canvas"></canvas>
    
    <div id="perf-dashboard">
        <div>
            <div class="panel-title">📊 Render Stats</div>
            <div id="render-stats"></div>
        </div>
        
        <div>
            <div class="panel-title">📈 FPS Graph</div>
            <div class="sparkline-wrapper">
                <div class="sparkline-label">FPS (target: 60)</div>
                <canvas id="fps-graph" width="276" height="50"></canvas>
            </div>
        </div>
        
        <div>
            <div class="panel-title">⏱️ Frame Budget</div>
            <div class="sparkline-wrapper">
                <div class="sparkline-label">Frame Time ms (budget: 16.67)</div>
                <canvas id="frame-graph" width="276" height="40"></canvas>
            </div>
        </div>
        
        <div>
            <div class="panel-title">🔍 Bottleneck Analysis</div>
            <div class="bottleneck-box" id="bottleneck-box">กำลังรวบรวมข้อมูล...</div>
        </div>
        
        <div>
            <div class="panel-title">⚠️ Warning Log</div>
            <div class="warning-log" id="warning-log"></div>
        </div>
        
        <div>
            <div class="panel-title">🧪 Stress Tests</div>
            <div class="test-controls">
                <button class="test-btn" id="btn-normal">✅ Normal Scene</button>
                <button class="test-btn" id="btn-many-draws">🔴 Many Draw Calls (1000 meshes)</button>
                <button class="test-btn" id="btn-high-poly">🔴 High Polygon (10M triangles)</button>
                <button class="test-btn" id="btn-many-lights">🟡 Many Lights (20 point lights)</button>
                <button class="test-btn" id="btn-heavy-js">🟡 Heavy JS (physics sim)</button>
            </div>
        </div>
        
        <div>
            <div class="panel-title">ℹ️ Scene Info</div>
            <div id="scene-info"></div>
        </div>
    </div>

    <script type="importmap">
    {
        "imports": {
            "three": "https://cdn.jsdelivr.net/npm/three@0.165.0/build/three.module.js",
            "three/examples/jsm/": "https://cdn.jsdelivr.net/npm/three@0.165.0/examples/jsm/"
        }
    }
    </script>
    <script type="module">
        import * as THREE from 'three';
        import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

        const PANEL_WIDTH = 300;
        const W = window.innerWidth - PANEL_WIDTH;
        const H = window.innerHeight;
        
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x050510);
        
        const camera = new THREE.PerspectiveCamera(75, W / H, 0.1, 1000);
        camera.position.set(0, 30, 60);
        
        const renderer = new THREE.WebGLRenderer({
            canvas: document.getElementById('canvas'),
            antialias: true,
        });
        renderer.setSize(W, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.target.set(0, 10, 0);
        
        // Lighting
        scene.add(new THREE.AmbientLight(0x334466, 1.5));
        const mainLight = new THREE.DirectionalLight(0xffffff, 2);
        mainLight.position.set(30, 50, 20);
        scene.add(mainLight);
        
        // === Current scene meshes ===
        let currentTestMeshes = [];
        let heavyJSActive = false;
        let manyLightsActive = false;
        const extraLights = [];
        
        // Normal scene
        function buildNormalScene() {
            clearTestMeshes();
            
            const ground = new THREE.Mesh(
                new THREE.PlaneGeometry(100, 100),
                new THREE.MeshStandardMaterial({ color: 0x223344 })
            );
            ground.rotation.x = -Math.PI / 2;
            scene.add(ground);
            currentTestMeshes.push(ground);
            
            for (let i = 0; i < 20; i++) {
                const mesh = new THREE.Mesh(
                    new THREE.BoxGeometry(3, 3, 3),
                    new THREE.MeshStandardMaterial({ color: new THREE.Color().setHSL(i / 20, 0.7, 0.5) })
                );
                mesh.position.set((Math.random() - 0.5) * 60, 1.5, (Math.random() - 0.5) * 60);
                scene.add(mesh);
                currentTestMeshes.push(mesh);
            }
        }
        
        function buildManyDrawsScene() {
            clearTestMeshes();
            
            for (let i = 0; i < 1000; i++) {
                const mesh = new THREE.Mesh(
                    new THREE.BoxGeometry(1, 1, 1),
                    new THREE.MeshStandardMaterial({ color: 0x4488ff })
                );
                mesh.position.set(
                    (Math.random() - 0.5) * 100,
                    Math.random() * 20,
                    (Math.random() - 0.5) * 100
                );
                scene.add(mesh);
                currentTestMeshes.push(mesh);
            }
        }
        
        function buildHighPolyScene() {
            clearTestMeshes();
            
            // High poly sphere
            const mesh = new THREE.Mesh(
                new THREE.SphereGeometry(15, 256, 256), // huge subdivision
                new THREE.MeshStandardMaterial({ color: 0x4488ff, wireframe: false })
            );
            mesh.position.y = 15;
            scene.add(mesh);
            currentTestMeshes.push(mesh);
        }
        
        function buildManyLightsScene() {
            clearTestMeshes();
            buildNormalScene();
            
            for (let i = 0; i < 20; i++) {
                const light = new THREE.PointLight(
                    new THREE.Color().setHSL(i / 20, 1, 0.5),
                    2, 30
                );
                light.position.set(
                    Math.sin(i / 20 * Math.PI * 2) * 25,
                    5 + Math.random() * 10,
                    Math.cos(i / 20 * Math.PI * 2) * 25
                );
                scene.add(light);
                extraLights.push(light);
            }
            manyLightsActive = true;
        }
        
        function clearTestMeshes() {
            currentTestMeshes.forEach(m => {
                scene.remove(m);
                m.geometry?.dispose();
                if (Array.isArray(m.material)) {
                    m.material.forEach(mat => mat.dispose());
                } else {
                    m.material?.dispose();
                }
            });
            currentTestMeshes = [];
            
            extraLights.forEach(l => scene.remove(l));
            extraLights.length = 0;
            manyLightsActive = false;
            heavyJSActive = false;
        }
        
        buildNormalScene();
        
        // Button handlers
        document.getElementById('btn-normal').addEventListener('click', () => {
            document.querySelectorAll('.test-btn').forEach(b => b.classList.remove('active'));
            buildNormalScene();
        });
        document.getElementById('btn-many-draws').addEventListener('click', (e) => {
            document.querySelectorAll('.test-btn').forEach(b => b.classList.remove('active'));
            e.target.classList.add('active');
            buildManyDrawsScene();
        });
        document.getElementById('btn-high-poly').addEventListener('click', (e) => {
            document.querySelectorAll('.test-btn').forEach(b => b.classList.remove('active'));
            e.target.classList.add('active');
            buildHighPolyScene();
        });
        document.getElementById('btn-many-lights').addEventListener('click', (e) => {
            document.querySelectorAll('.test-btn').forEach(b => b.classList.remove('active'));
            e.target.classList.add('active');
            buildManyLightsScene();
        });
        document.getElementById('btn-heavy-js').addEventListener('click', (e) => {
            document.querySelectorAll('.test-btn').forEach(b => b.classList.remove('active'));
            e.target.classList.add('active');
            buildNormalScene();
            heavyJSActive = true;
        });
        
        // === Performance tracking ===
        const fpsHistory = [];
        const frameHistory = [];
        const MAX_HISTORY = 100;
        
        let lastTime = performance.now();
        let frameCount = 0;
        let fps = 60;
        
        const fpsCtx = document.getElementById('fps-graph').getContext('2d');
        const frameCtx = document.getElementById('frame-graph').getContext('2d');
        
        const warningLog = document.getElementById('warning-log');
        const warnings = [];
        
        function addWarning(msg, isCritical = false) {
            const time = new Date().toLocaleTimeString();
            warnings.unshift({ time, msg, isCritical });
            if (warnings.length > 20) warnings.pop();
            
            warningLog.innerHTML = warnings.slice(0, 10).map(w =>
                `<div class="warn-item ${w.isCritical ? 'critical' : ''}">[${w.time}] ${w.msg}</div>`
            ).join('');
        }
        
        function drawSparklineGraph(ctx, data, w, h, min, max, color, budget = null) {
            ctx.clearRect(0, 0, w, h);
            ctx.fillStyle = 'rgba(0,0,20,0.5)';
            ctx.fillRect(0, 0, w, h);
            
            if (budget !== null) {
                const budgetY = h - ((budget - min) / (max - min)) * h;
                ctx.strokeStyle = 'rgba(255,255,100,0.2)';
                ctx.lineWidth = 1;
                ctx.setLineDash([3, 3]);
                ctx.beginPath();
                ctx.moveTo(0, budgetY);
                ctx.lineTo(w, budgetY);
                ctx.stroke();
                ctx.setLineDash([]);
            }
            
            if (data.length < 2) return;
            
            ctx.strokeStyle = color;
            ctx.lineWidth = 1.5;
            ctx.beginPath();
            data.forEach((v, i) => {
                const x = (i / (MAX_HISTORY - 1)) * w;
                const y = h - ((Math.min(Math.max(v, min), max) - min) / (max - min)) * h;
                if (i === 0) ctx.moveTo(x, y);
                else ctx.lineTo(x, y);
            });
            ctx.stroke();
        }
        
        function updateDashboard(renderer) {
            const info = renderer.info;
            const r = info.render;
            const m = info.memory;
            
            // Color helper
            const col = (v, good, bad, reverse = false) => {
                if (reverse) { // higher is better
                    if (v >= good) return 'good';
                    if (v >= bad) return 'warn';
                    return 'bad';
                } else { // lower is better
                    if (v <= good) return 'good';
                    if (v <= bad) return 'warn';
                    return 'bad';
                }
            };
            
            document.getElementById('render-stats').innerHTML = `
                <div class="metric-row"><span>FPS</span><span class="val ${col(fps, 55, 30, true)}">${Math.round(fps)}</span></div>
                <div class="metric-row"><span>Frame Time</span><span class="val ${col(frameHistory[frameHistory.length-1]||0, 16.7, 33.3)}">${(frameHistory[frameHistory.length-1]||0).toFixed(1)}ms</span></div>
                <div class="metric-row"><span>Draw Calls</span><span class="val ${col(r.calls, 100, 400)}">${r.calls}</span></div>
                <div class="metric-row"><span>Triangles</span><span class="val ${col(r.triangles, 200000, 1000000)}">${(r.triangles/1000).toFixed(0)}K</span></div>
                <div class="metric-row"><span>Points</span><span class="val">${r.points}</span></div>
                <div class="metric-row"><span>Lines</span><span class="val">${r.lines}</span></div>
                <div class="metric-row"><span>Geometries (GPU)</span><span class="val">${m.geometries}</span></div>
                <div class="metric-row"><span>Textures (GPU)</span><span class="val ${col(m.textures, 30, 80)}">${m.textures}</span></div>
                <div class="metric-row"><span>Shader Programs</span><span class="val">${info.programs?.length || 0}</span></div>
            `;
            
            // Bottleneck analysis
            let analysis = '';
            if (fps < 20) {
                analysis = '🔴 CRITICAL: FPS < 20\n';
                if (r.calls > 300) analysis += '→ Draw calls สูงมาก (' + r.calls + ')\n';
                if (r.triangles > 2000000) analysis += '→ Triangles สูงมาก\n';
                if (heavyJSActive) analysis += '→ JavaScript heavy computation\n';
            } else if (fps < 40) {
                analysis = '🟡 WARNING: FPS < 40\n';
                if (r.calls > 150) analysis += '→ ลด Draw calls\n';
                if (r.triangles > 500000) analysis += '→ ลด polygon count\n';
            } else if (fps < 55) {
                analysis = '🟡 OK: FPS ต่ำกว่า target เล็กน้อย\n';
            } else {
                analysis = '🟢 EXCELLENT: Performance ดี\n';
                analysis += `→ ${Math.round(fps)} FPS / ${r.calls} draw calls`;
            }
            
            document.getElementById('bottleneck-box').textContent = analysis;
            
            // Warnings
            if (r.calls > 400 && frameCount % 60 === 0) {
                addWarning(`Draw calls: ${r.calls} (สูงเกินไป)`, r.calls > 600);
            }
            if (fps < 25 && frameCount % 60 === 0) {
                addWarning(`FPS: ${Math.round(fps)} (ต่ำมาก)`, true);
            }
            if (m.textures > 80 && frameCount % 300 === 0) {
                addWarning(`Textures: ${m.textures} (อาจ leak)`, false);
            }
            
            // Scene info
            document.getElementById('scene-info').innerHTML = `
                Objects: ${scene.children.length}<br>
                Camera: (${camera.position.x.toFixed(1)}, ${camera.position.y.toFixed(1)}, ${camera.position.z.toFixed(1)})<br>
                Near/Far: ${camera.near}/${camera.far}<br>
                FOV: ${camera.fov}°
            `;
        }
        
        function heavyJSWork() {
            // Simulate heavy physics computation
            let sum = 0;
            for (let i = 0; i < 500000; i++) {
                sum += Math.sqrt(i) * Math.sin(i * 0.001);
            }
            return sum;
        }
        
        let t = 0;
        function animate() {
            requestAnimationFrame(animate);
            
            const now = performance.now();
            const dt = now - lastTime;
            lastTime = now;
            frameCount++;
            
            // FPS calculation
            fps = 0.9 * fps + 0.1 * (1000 / dt);
            fpsHistory.push(fps);
            frameHistory.push(dt);
            if (fpsHistory.length > MAX_HISTORY) fpsHistory.shift();
            if (frameHistory.length > MAX_HISTORY) frameHistory.shift();
            
            t = now * 0.001;
            
            // Optional heavy JS
            if (heavyJSActive && frameCount % 2 === 0) {
                heavyJSWork();
            }
            
            // Animate objects
            currentTestMeshes.forEach((mesh, i) => {
                if (mesh.isMesh && mesh.geometry.parameters?.width === 3) {
                    mesh.rotation.y = t * (0.3 + i * 0.02);
                }
            });
            
            // Animate extra lights
            extraLights.forEach((light, i) => {
                const a = t * 0.5 + (i / extraLights.length) * Math.PI * 2;
                light.position.x = Math.sin(a) * 25;
                light.position.z = Math.cos(a) * 25;
            });
            
            // Update graphs every N frames
            if (frameCount % 3 === 0) {
                drawSparklineGraph(fpsCtx, fpsHistory, 276, 50, 0, 75, '#44ff44', 60);
                drawSparklineGraph(frameCtx, frameHistory, 276, 40, 0, 50, '#4488ff', 16.67);
                updateDashboard(renderer);
                renderer.info.reset();
            }
            
            controls.update();
            renderer.render(scene, camera);
        }
        animate();
        
        window.addEventListener('resize', () => {
            const w = window.innerWidth - PANEL_WIDTH;
            const h = window.innerHeight;
            camera.aspect = w / h;
            camera.updateProjectionMatrix();
            renderer.setSize(w, h);
        });
    </script>
</body>
</html>
```

## Chrome DevTools GPU Timeline

```javascript
// เคล็ดลับสำหรับ Chrome DevTools Performance tab

// 1. ใช้ requestAnimationFrame เสมอ (ไม่ใช้ setTimeout/setInterval)
// Chrome จะแสดงใน "Animation Frame Fired" events

// 2. Label WebGL operations
renderer.domElement.setAttribute('label', 'Three.js Canvas');

// 3. ใช้ performance.mark สำหรับ custom timing
performance.mark('ThreeJS:SceneUpdate:Start');
updateScene();
performance.mark('ThreeJS:SceneUpdate:End');
performance.measure('ThreeJS:SceneUpdate', 'ThreeJS:SceneUpdate:Start', 'ThreeJS:SceneUpdate:End');

// 4. Force GPU sync เพื่อ accurate GPU timing (ระวัง performance hit)
function getGPUTime(renderer) {
    renderer.render(scene, camera);
    const gl = renderer.getContext();
    gl.finish(); // block จนกว่า GPU เสร็จ
    return performance.now();
}
```

## Memory Leak Detection

```javascript
class MemoryTracker {
    constructor(renderer) {
        this.renderer = renderer;
        this.snapshots = [];
        this.interval = null;
    }
    
    startTracking(intervalMs = 5000) {
        this.interval = setInterval(() => {
            this.takeSnapshot();
        }, intervalMs);
    }
    
    takeSnapshot() {
        const info = this.renderer.info;
        const snapshot = {
            time: Date.now(),
            geometries: info.memory.geometries,
            textures: info.memory.textures,
            programs: info.programs?.length || 0,
            jsHeap: performance.memory?.usedJSHeapSize || 0,
        };
        
        this.snapshots.push(snapshot);
        
        // Check for leaks (growing memory)
        if (this.snapshots.length > 5) {
            const old = this.snapshots[this.snapshots.length - 5];
            const curr = snapshot;
            
            if (curr.textures > old.textures + 5) {
                console.warn(`⚠️ Texture leak: ${old.textures} → ${curr.textures}`);
            }
            if (curr.geometries > old.geometries + 10) {
                console.warn(`⚠️ Geometry leak: ${old.geometries} → ${curr.geometries}`);
            }
        }
        
        return snapshot;
    }
    
    stopTracking() {
        clearInterval(this.interval);
    }
    
    // ลบ objects ที่ไม่ใช้แล้ว
    disposeObject(object) {
        object.traverse(child => {
            if (child.isMesh) {
                child.geometry?.dispose();
                if (child.material) {
                    if (Array.isArray(child.material)) {
                        child.material.forEach(m => this.disposeMaterial(m));
                    } else {
                        this.disposeMaterial(child.material);
                    }
                }
            }
        });
    }
    
    disposeMaterial(material) {
        Object.values(material).forEach(value => {
            if (value && value.isTexture) {
                value.dispose();
            }
        });
        material.dispose();
    }
}
```

## Optimization Checklist

```javascript
/*
 * Performance Optimization Checklist
 * ===================================
 *
 * 1. GEOMETRY
 *    ✓ ใช้ BufferGeometry (ไม่ใช้ Geometry เก่า)
 *    ✓ Merge static geometries (BufferGeometryUtils.mergeGeometries)
 *    ✓ ใช้ InstancedMesh สำหรับ repeated objects
 *    ✓ ใช้ LOD สำหรับ objects ที่อยู่ไกล
 *    ✓ Dispose geometry ที่ไม่ใช้แล้ว
 *
 * 2. MATERIALS
 *    ✓ ใช้ MeshBasicMaterial เมื่อไม่ต้องการ lighting
 *    ✓ Share materials ระหว่าง meshes (ถ้า color เหมือนกัน)
 *    ✓ Dispose material และ textures ที่ไม่ใช้
 *    ✓ ใช้ texture atlas แทน many small textures
 *
 * 3. DRAW CALLS
 *    ✓ Merge meshes ที่ใช้ material เดียวกัน
 *    ✓ ใช้ frustum culling (เปิดโดย default ใน Three.js)
 *    ✓ ปิด object.visible = false แทนการลบออก
 *    ✓ ใช้ renderer.info.render.calls เพื่อ monitor
 *
 * 4. TEXTURES
 *    ✓ ใช้ power-of-2 dimensions (256, 512, 1024...)
 *    ✓ Compress textures (GPU-compressed: DDS, KTX2)
 *    ✓ ใช้ mipmap
 *    ✓ texture.generateMipmaps = false สำหรับ render targets
 *
 * 5. RENDERING
 *    ✓ renderer.setPixelRatio(Math.min(devicePixelRatio, 2))
 *    ✓ Shadow map ขนาดเล็กพอ (512-2048)
 *    ✓ ปิด shadow สำหรับ objects ที่ไกล
 *    ✓ ใช้ renderer.autoClear = false เมื่อ render multiple passes
 *
 * 6. JAVASCRIPT
 *    ✓ หลีกเลี่ยง new Vector3() ใน animation loop
 *    ✓ Pre-allocate objects, ใช้ object pooling
 *    ✓ ใช้ typed arrays (Float32Array) แทน regular arrays
 *    ✓ Web Workers สำหรับ heavy computation
 */
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Before/After Comparison
สร้าง demo ที่แสดง scene ก่อนและหลัง optimization ด้วย side-by-side comparison มี counter แสดง draw calls ของทั้งสอง

### แบบฝึกหัดที่ 2: Memory Leak Finder
สร้าง scene ที่มี memory leak (เพิ่ม objects โดยไม่ dispose) และ tool ที่ตรวจพบ leak แล้วแสดงใน console

### แบบฝึกหัดที่ 3: LOD Profiling
เปรียบเทียบ performance ของ scene ที่มี LOD vs ไม่มี LOD แสดงผลด้วย graphs

### แบบฝึกหัดที่ 4: Custom Profiler
สร้าง profiler ที่วัดเวลา specific operations (physics, animation, AI) และแสดงเป็น flame chart อย่างง่าย

## สรุปท้ายคอร์ส

ยินดีด้วยที่เรียนจบ **101 Parts** ของ Three.js Course!

### สิ่งที่เรียนรู้ใน Parts 93-101:

- **Part 93**: CSS3DRenderer — HTML ใน 3D space
- **Part 94**: 3D UI Components — buttons, sliders, panels
- **Part 95**: Labels & Annotations — CSS2DRenderer, HUD
- **Part 96**: Data Visualization — D3.js + Three.js, charts
- **Part 97**: 3D Charts — pie, radar, treemap, real-time
- **Part 98**: Minimap & Navigation — WebGLRenderTarget, waypoints
- **Part 99**: Loading Screens — LoadingManager, transitions
- **Part 100**: Progress Indicators — health bars, achievements
- **Part 101**: Performance Profiling — DevTools, optimization

### Tips สุดท้าย

1. **วัดก่อน optimize** — อย่า premature optimize
2. **renderer.info เป็นเพื่อนที่ดี** — ใช้บ่อยๆ
3. **Dispose เสมอ** — memory leak คือศัตรูตัวร้าย
4. **profile บน target device** — mobile ต่างจาก desktop มาก
5. **ใช้ Chrome DevTools Performance** — flame graph บอกได้ทุกอย่าง

Happy coding! 🚀
