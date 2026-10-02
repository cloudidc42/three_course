# Part 108: Mobile Performance (Advanced)

## บทนำ: ความท้าทายของ Mobile WebGL

Mobile devices มีข้อจำกัดที่แตกต่างจาก desktop อย่างมาก:

| ปัจจัย | Desktop | Mobile |
|--------|---------|--------|
| GPU Tier | High | Low-Mid |
| RAM | 8-32 GB | 2-6 GB |
| CPU Cores | 8-32 | 4-8 (low clock) |
| Thermal Budget | 65-150W | 3-8W |
| Battery | N/A | จำกัด |
| Resolution | แก้ได้ง่าย | แก้ได้ยาก |

การพัฒนา 3D web app สำหรับ mobile ต้องคำนึงถึงสิ่งเหล่านี้ตั้งแต่แรก

---

## 1. GPU Tier Detection

```javascript
// ใช้ detect-gpu library (แนะนำ) หรือ detect เอง

// Method 1: ใช้ detect-gpu
import { getGPUTier } from 'detect-gpu';

async function detectGPUTier() {
    const gpuTier = await getGPUTier();
    
    console.log({
        tier: gpuTier.tier,        // 0=ไม่รองรับ, 1=ต่ำ, 2=กลาง, 3=สูง
        gpu: gpuTier.gpu,          // GPU name
        isMobile: gpuTier.isMobile,
        fps: gpuTier.fps           // ประมาณ fps ที่ GPU ทำได้
    });
    
    return gpuTier.tier;
}

// Method 2: ตรวจสอบด้วยตนเองจาก WebGL info
function estimateGPUTier(renderer) {
    const gl = renderer.getContext();
    const ext = gl.getExtension('WEBGL_debug_renderer_info');
    
    let gpuName = 'unknown';
    if (ext) {
        gpuName = gl.getParameter(ext.UNMASKED_RENDERER_WEBGL).toLowerCase();
    }
    
    // ตรวจสอบ GPU known tiers
    if (gpuName.includes('apple') || gpuName.includes('m1') || gpuName.includes('m2')) {
        return 3; // High tier (Apple Silicon, Apple A15+)
    }
    if (gpuName.includes('adreno 6') || gpuName.includes('mali-g7') || gpuName.includes('mali-g8')) {
        return 2; // Mid tier
    }
    if (gpuName.includes('adreno 5') || gpuName.includes('mali-g5') || gpuName.includes('mali-t')) {
        return 1; // Low tier
    }
    
    // Fallback: ตรวจสอบจาก WebGL capabilities
    const maxTextureSize = gl.getParameter(gl.MAX_TEXTURE_SIZE);
    const maxRenderbufferSize = gl.getParameter(gl.MAX_RENDERBUFFER_SIZE);
    
    if (maxTextureSize >= 16384) return 3;
    if (maxTextureSize >= 8192) return 2;
    return 1;
}

// Method 3: Benchmark-based detection
async function benchmarkGPU(renderer, scene, camera) {
    const FRAMES = 30;
    const startTime = performance.now();
    
    for (let i = 0; i < FRAMES; i++) {
        renderer.render(scene, camera);
        await new Promise(r => requestAnimationFrame(r));
    }
    
    const elapsed = performance.now() - startTime;
    const avgFPS = (FRAMES / elapsed) * 1000;
    
    if (avgFPS >= 50) return 3;
    if (avgFPS >= 30) return 2;
    return 1;
}
```

---

## 2. Adaptive Quality System

```javascript
class AdaptiveQualitySystem {
    constructor(renderer, options = {}) {
        this.renderer = renderer;
        this.options = {
            targetFPS: options.targetFPS || 60,
            minFPS: options.minFPS || 25,
            maxPixelRatio: options.maxPixelRatio || 2,
            minPixelRatio: options.minPixelRatio || 0.5,
            checkInterval: options.checkInterval || 2000, // ms
            adjustmentStep: options.adjustmentStep || 0.1
        };
        
        this.currentTier = 2; // เริ่มที่ mid
        this.currentPixelRatio = window.devicePixelRatio;
        this.fpsHistory = [];
        this.lastCheck = performance.now();
        this.locked = false;
        
        this.qualitySettings = {
            3: { // High
                pixelRatioScale: 1.0,
                shadowMapSize: 2048,
                shadowsEnabled: true,
                antialias: true,
                particleCount: 1000,
                drawDistance: 500,
                postProcessing: true,
                anisotropy: 16
            },
            2: { // Mid
                pixelRatioScale: 0.75,
                shadowMapSize: 1024,
                shadowsEnabled: true,
                antialias: false,
                particleCount: 300,
                drawDistance: 200,
                postProcessing: false,
                anisotropy: 4
            },
            1: { // Low
                pixelRatioScale: 0.5,
                shadowMapSize: 512,
                shadowsEnabled: false,
                antialias: false,
                particleCount: 50,
                drawDistance: 100,
                postProcessing: false,
                anisotropy: 1
            },
            0: { // Very Low (fallback)
                pixelRatioScale: 0.4,
                shadowMapSize: 256,
                shadowsEnabled: false,
                antialias: false,
                particleCount: 0,
                drawDistance: 50,
                postProcessing: false,
                anisotropy: 1
            }
        };
        
        this.callbacks = {
            onTierChange: null,
            onQualityUpdate: null
        };
    }
    
    // เรียกทุก frame เพื่อ track FPS
    update(currentFPS) {
        if (this.locked) return;
        
        this.fpsHistory.push(currentFPS);
        if (this.fpsHistory.length > 30) this.fpsHistory.shift();
        
        const now = performance.now();
        if (now - this.lastCheck < this.options.checkInterval) return;
        this.lastCheck = now;
        
        this._adjustQuality();
    }
    
    _adjustQuality() {
        if (this.fpsHistory.length < 10) return;
        
        const avgFPS = this.fpsHistory.reduce((a, b) => a + b, 0) / this.fpsHistory.length;
        const { targetFPS, minFPS } = this.options;
        
        if (avgFPS < minFPS && this.currentTier > 0) {
            // FPS ต่ำเกินไป: ลด quality
            this._setTier(this.currentTier - 1);
            this.fpsHistory = [];
        } else if (avgFPS > targetFPS * 0.95 && this.currentTier < 3) {
            // FPS สูงพอ: ลอง increase quality
            this._setTier(this.currentTier + 1);
            this.fpsHistory = [];
        }
    }
    
    _setTier(tier) {
        const oldTier = this.currentTier;
        this.currentTier = Math.max(0, Math.min(3, tier));
        
        if (oldTier !== this.currentTier) {
            this._applySettings();
            
            if (this.callbacks.onTierChange) {
                this.callbacks.onTierChange(this.currentTier, oldTier);
            }
        }
    }
    
    _applySettings() {
        const settings = this.qualitySettings[this.currentTier];
        
        // Pixel ratio
        const targetPR = window.devicePixelRatio * settings.pixelRatioScale;
        const clampedPR = Math.max(
            this.options.minPixelRatio,
            Math.min(this.options.maxPixelRatio, targetPR)
        );
        this.renderer.setPixelRatio(clampedPR);
        this.currentPixelRatio = clampedPR;
        
        // Shadows
        this.renderer.shadowMap.enabled = settings.shadowsEnabled;
        
        if (this.callbacks.onQualityUpdate) {
            this.callbacks.onQualityUpdate(settings, this.currentTier);
        }
    }
    
    // Force specific tier
    setTier(tier) {
        this.locked = true;
        this._setTier(tier);
    }
    
    // Allow auto-adjustment again
    unlock() {
        this.locked = false;
        this.fpsHistory = [];
        this.lastCheck = performance.now();
    }
    
    getCurrentSettings() {
        return this.qualitySettings[this.currentTier];
    }
    
    getTierName() {
        return ['Very Low', 'Low', 'Medium', 'High'][this.currentTier];
    }
}
```

---

## 3. Power Preference & Battery API Integration

```javascript
// WebGL power preference hint
const renderer = new THREE.WebGLRenderer({
    canvas,
    // 'high-performance' = GPU ใช้ power เยอะ แต่เร็วกว่า (สำหรับเกม/demo)
    // 'low-power' = GPU ใช้ power น้อย (สำหรับ UI/visualization บน battery)
    // 'default' = browser ตัดสินใจ
    powerPreference: 'high-performance'
});

// Battery API integration
class BatteryAwareRenderer {
    constructor(renderer, qualitySystem) {
        this.renderer = renderer;
        this.qualitySystem = qualitySystem;
        this.battery = null;
        this.isCharging = true;
        this.batteryLevel = 1.0;
        
        this._initBattery();
    }
    
    async _initBattery() {
        if (!navigator.getBattery) {
            console.log('Battery API not supported');
            return;
        }
        
        try {
            this.battery = await navigator.getBattery();
            
            this.isCharging = this.battery.charging;
            this.batteryLevel = this.battery.level;
            
            this.battery.addEventListener('chargingchange', () => {
                this.isCharging = this.battery.charging;
                this._onBatteryChange();
            });
            
            this.battery.addEventListener('levelchange', () => {
                this.batteryLevel = this.battery.level;
                this._onBatteryChange();
            });
            
            this._onBatteryChange();
        } catch (e) {
            console.warn('Battery API error:', e);
        }
    }
    
    _onBatteryChange() {
        const { isCharging, batteryLevel } = this;
        
        if (!isCharging && batteryLevel < 0.15) {
            // Battery low (< 15%) และไม่ได้ชาร์จ: บังคับ Low quality
            this.qualitySystem.setTier(1);
            console.log('Low battery mode: Quality reduced to Low');
        } else if (!isCharging && batteryLevel < 0.30) {
            // Battery medium-low: ลด quality 1 tier
            const currentTier = this.qualitySystem.currentTier;
            this.qualitySystem.setTier(Math.max(1, currentTier - 1));
        } else if (isCharging) {
            // กำลังชาร์จ: unlock quality
            this.qualitySystem.unlock();
        }
    }
    
    getBatteryStatus() {
        return {
            supported: !!this.battery,
            charging: this.isCharging,
            level: Math.round(this.batteryLevel * 100),
            recommendation: this.isCharging ? 'Normal' : 
                this.batteryLevel < 0.15 ? 'Low Power' : 
                this.batteryLevel < 0.30 ? 'Reduced' : 'Normal'
        };
    }
}
```

---

## 4. Thermal Throttling Detection

```javascript
class ThermalThrottleDetector {
    constructor(options = {}) {
        this.sampleWindow = options.sampleWindow || 30; // frames
        this.throttleThreshold = options.throttleThreshold || 0.7; // 70% of baseline
        
        this.baselineFPS = null;
        this.currentFPS = null;
        this.fpsSamples = [];
        this.isThrottled = false;
        this.warmupFrames = 0;
        this.maxWarmup = 60; // 1 second at 60fps
        
        this.onThrottle = null;
        this.onRecover = null;
    }
    
    update(fps) {
        this.warmupFrames++;
        
        // Warmup period: establish baseline
        if (this.warmupFrames < this.maxWarmup) return;
        
        if (!this.baselineFPS) {
            this.fpsSamples.push(fps);
            if (this.fpsSamples.length >= this.sampleWindow) {
                this.baselineFPS = this.fpsSamples.reduce((a, b) => a + b, 0) / this.fpsSamples.length;
                console.log(`Baseline FPS established: ${this.baselineFPS.toFixed(1)}`);
                this.fpsSamples = [];
            }
            return;
        }
        
        // Track current FPS
        this.fpsSamples.push(fps);
        if (this.fpsSamples.length > this.sampleWindow) this.fpsSamples.shift();
        
        const avgFPS = this.fpsSamples.reduce((a, b) => a + b, 0) / this.fpsSamples.length;
        this.currentFPS = avgFPS;
        
        const fpsRatio = avgFPS / this.baselineFPS;
        const wasThrottled = this.isThrottled;
        this.isThrottled = fpsRatio < this.throttleThreshold;
        
        if (this.isThrottled && !wasThrottled && this.onThrottle) {
            this.onThrottle({ currentFPS: avgFPS, baselineFPS: this.baselineFPS, ratio: fpsRatio });
        } else if (!this.isThrottled && wasThrottled && this.onRecover) {
            this.onRecover({ currentFPS: avgFPS, baselineFPS: this.baselineFPS, ratio: fpsRatio });
        }
    }
    
    getStatus() {
        return {
            isThrottled: this.isThrottled,
            baselineFPS: this.baselineFPS ? Math.round(this.baselineFPS) : 'Measuring...',
            currentFPS: this.currentFPS ? Math.round(this.currentFPS) : '-',
            performance: this.baselineFPS && this.currentFPS ? 
                `${Math.round(this.currentFPS / this.baselineFPS * 100)}%` : '-'
        };
    }
}
```

---

## 5. Mobile-Specific Optimizations

```javascript
// ตรวจสอบว่าเป็น mobile หรือไม่
function isMobile() {
    return /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent)
        || (navigator.maxTouchPoints > 0 && window.innerWidth < 768);
}

// Mobile-specific Three.js setup
function setupMobileRenderer(canvas) {
    const mobile = isMobile();
    
    const renderer = new THREE.WebGLRenderer({
        canvas,
        antialias: !mobile,           // ปิด antialias บน mobile
        powerPreference: mobile ? 'low-power' : 'high-performance',
        precision: mobile ? 'mediump' : 'highp', // ลด precision บน mobile
        logarithmicDepthBuffer: false // ปิด log depth buffer (ช้า บน mobile)
    });
    
    if (mobile) {
        // ลด pixel ratio สำหรับ mobile
        renderer.setPixelRatio(Math.min(devicePixelRatio, 1.5));
        
        // ปิด features ที่ใช้ไม่ถึงบน mobile
        renderer.shadowMap.enabled = false;
        renderer.toneMapping = THREE.NoToneMapping; // ง่ายกว่า ACES
        
        // ปิด physical lights (ใช้ shader ที่ซับซ้อนกว่า)
        renderer.physicallyCorrectLights = false;
    } else {
        renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
        renderer.shadowMap.enabled = true;
        renderer.toneMapping = THREE.ACESFilmicToneMapping;
    }
    
    return renderer;
}

// Touch-optimized controls
class TouchOptimizedControls {
    constructor(camera, domElement) {
        this.camera = camera;
        this.element = domElement;
        
        this.touches = new Map();
        this.prevPinchDist = 0;
        
        // Smoothing
        this.targetOrbit = { theta: 0, phi: Math.PI / 4 };
        this.currentOrbit = { theta: 0, phi: Math.PI / 4 };
        this.radius = 10;
        this.smoothing = 0.15; // Higher = more responsive, Lower = smoother
        
        this._setupEvents();
    }
    
    _setupEvents() {
        this.element.addEventListener('touchstart', e => this._onTouchStart(e), { passive: false });
        this.element.addEventListener('touchmove', e => this._onTouchMove(e), { passive: false });
        this.element.addEventListener('touchend', e => this._onTouchEnd(e));
    }
    
    _onTouchStart(e) {
        e.preventDefault();
        Array.from(e.changedTouches).forEach(t => {
            this.touches.set(t.identifier, { x: t.clientX, y: t.clientY });
        });
    }
    
    _onTouchMove(e) {
        e.preventDefault();
        
        if (e.touches.length === 1) {
            // Single touch: orbit
            const touch = e.touches[0];
            const prev = this.touches.get(touch.identifier);
            if (!prev) return;
            
            const dx = (touch.clientX - prev.x) * 0.005;
            const dy = (touch.clientY - prev.y) * 0.005;
            
            this.targetOrbit.theta -= dx;
            this.targetOrbit.phi = Math.max(0.1, Math.min(Math.PI / 2, this.targetOrbit.phi + dy));
            
            this.touches.set(touch.identifier, { x: touch.clientX, y: touch.clientY });
            
        } else if (e.touches.length === 2) {
            // Pinch: zoom
            const t1 = e.touches[0], t2 = e.touches[1];
            const dist = Math.sqrt(
                (t1.clientX - t2.clientX) ** 2 + 
                (t1.clientY - t2.clientY) ** 2
            );
            
            if (this.prevPinchDist > 0) {
                const delta = (dist - this.prevPinchDist) * 0.05;
                this.radius = Math.max(2, Math.min(50, this.radius - delta));
            }
            
            this.prevPinchDist = dist;
        }
    }
    
    _onTouchEnd(e) {
        Array.from(e.changedTouches).forEach(t => {
            this.touches.delete(t.identifier);
        });
        if (e.touches.length < 2) this.prevPinchDist = 0;
    }
    
    update() {
        // Smooth interpolation
        this.currentOrbit.theta += (this.targetOrbit.theta - this.currentOrbit.theta) * this.smoothing;
        this.currentOrbit.phi += (this.targetOrbit.phi - this.currentOrbit.phi) * this.smoothing;
        
        this.camera.position.set(
            this.radius * Math.sin(this.currentOrbit.theta) * Math.sin(this.currentOrbit.phi),
            this.radius * Math.cos(this.currentOrbit.phi),
            this.radius * Math.cos(this.currentOrbit.theta) * Math.sin(this.currentOrbit.phi)
        );
        this.camera.lookAt(0, 0, 0);
    }
}
```

---

## 6. Complete Mobile Performance Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <meta name="mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="theme-color" content="#050510">
    <title>Mobile Performance Demo</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; touch-action: none; }
        body { background: #050510; color: #eee; font-family: system-ui, sans-serif; overflow: hidden; }
        canvas { display: block; width: 100vw; height: 100vh; }
        #hud {
            position: fixed; top: env(safe-area-inset-top, 20px); left: 50%;
            transform: translateX(-50%);
            background: rgba(0,0,15,0.85);
            padding: 12px 20px; border-radius: 20px;
            border: 1px solid rgba(0,170,255,0.3);
            display: flex; gap: 20px; align-items: center;
            backdrop-filter: blur(10px);
            -webkit-backdrop-filter: blur(10px);
            z-index: 10;
        }
        .stat { text-align: center; }
        .stat-val { font-size: 18px; font-weight: 700; color: #0af; }
        .stat-label { font-size: 10px; color: #888; text-transform: uppercase; }
        #quality-panel {
            position: fixed; bottom: env(safe-area-inset-bottom, 20px); left: 20px;
            background: rgba(0,0,15,0.85);
            padding: 12px 16px; border-radius: 12px;
            border: 1px solid rgba(0,170,255,0.3);
            backdrop-filter: blur(10px);
            -webkit-backdrop-filter: blur(10px);
            z-index: 10; min-width: 200px;
        }
        .qrow { display: flex; justify-content: space-between; margin: 3px 0; font-size: 12px; }
        .qlabel { color: #888; }
        .qval { color: #eee; font-weight: 600; }
        .qval.green { color: #4f4; }
        .qval.yellow { color: #fa0; }
        .qval.red { color: #f44; }
        .tier-badge {
            display: inline-block; padding: 2px 8px;
            border-radius: 10px; font-size: 11px; font-weight: 700;
            margin-left: 8px;
        }
        .tier-3 { background: rgba(0,255,0,0.2); color: #0f0; }
        .tier-2 { background: rgba(255,220,0,0.2); color: #ffd700; }
        .tier-1 { background: rgba(255,100,0,0.2); color: #f60; }
        .tier-0 { background: rgba(255,0,0,0.2); color: #f00; }
        #controls-row {
            position: fixed; bottom: env(safe-area-inset-bottom, 20px); right: 20px;
            display: flex; flex-direction: column; gap: 8px; z-index: 10;
        }
        .ctrl-btn {
            padding: 10px 14px; background: rgba(0,170,255,0.15);
            color: #0af; border: 1px solid rgba(0,170,255,0.4);
            border-radius: 8px; cursor: pointer; font-size: 12px; font-weight: 600;
            backdrop-filter: blur(10px); -webkit-backdrop-filter: blur(10px);
        }
    </style>
</head>
<body>
<canvas id="c"></canvas>

<div id="hud">
    <div class="stat"><div class="stat-val" id="h-fps">-</div><div class="stat-label">FPS</div></div>
    <div class="stat"><div class="stat-val" id="h-calls">-</div><div class="stat-label">Draw Calls</div></div>
    <div class="stat"><div class="stat-val" id="h-pr">-</div><div class="stat-label">Pixel Ratio</div></div>
    <div class="stat"><div class="stat-val" id="h-tier">-</div><div class="stat-label">GPU Tier</div></div>
</div>

<div id="quality-panel">
    <div style="font-size:12px;color:#0af;font-weight:700;margin-bottom:8px">
        Quality: <span id="q-tier-name">-</span>
        <span id="q-tier-badge" class="tier-badge tier-2">MID</span>
    </div>
    <div class="qrow"><span class="qlabel">Pixel Ratio:</span><span id="q-pr" class="qval">-</span></div>
    <div class="qrow"><span class="qlabel">Shadows:</span><span id="q-shadow" class="qval">-</span></div>
    <div class="qrow"><span class="qlabel">Particles:</span><span id="q-particles" class="qval">-</span></div>
    <div class="qrow"><span class="qlabel">Draw Distance:</span><span id="q-dist" class="qval">-</span></div>
    <div class="qrow"><span class="qlabel">Battery:</span><span id="q-battery" class="qval">-</span></div>
    <div class="qrow"><span class="qlabel">Throttled:</span><span id="q-thermal" class="qval green">ปกติ</span></div>
</div>

<div id="controls-row">
    <button class="ctrl-btn" onclick="setQualityTier(3)">High</button>
    <button class="ctrl-btn" onclick="setQualityTier(2)">Mid</button>
    <button class="ctrl-btn" onclick="setQualityTier(1)">Low</button>
    <button class="ctrl-btn" onclick="toggleAuto()">Auto: ON</button>
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
    
    const canvas = document.getElementById('c');
    const isMob = /Android|webOS|iPhone|iPad|iPod/i.test(navigator.userAgent) || 
                  (navigator.maxTouchPoints > 0);
    
    // Renderer
    const renderer = new THREE.WebGLRenderer({
        canvas,
        antialias: !isMob,
        powerPreference: 'high-performance',
        precision: isMob ? 'mediump' : 'highp'
    });
    renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
    renderer.shadowMap.enabled = !isMob;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x050510);
    scene.fog = new THREE.FogExp2(0x050510, 0.008);
    
    const camera = new THREE.PerspectiveCamera(60, 1, 0.1, 500);
    camera.position.set(20, 15, 30);
    camera.lookAt(0, 0, 0);
    
    // Lighting
    scene.add(new THREE.AmbientLight(0x224488, 1));
    const sun = new THREE.DirectionalLight(0xffeedd, 2);
    sun.position.set(50, 80, 30);
    if (!isMob) {
        sun.castShadow = true;
        sun.shadow.mapSize.set(2048, 2048);
        sun.shadow.camera.near = 0.5;
        sun.shadow.camera.far = 200;
        sun.shadow.camera.left = -60;
        sun.shadow.camera.right = 60;
        sun.shadow.camera.top = 60;
        sun.shadow.camera.bottom = -60;
    }
    scene.add(sun);
    
    // Ground
    const groundMat = new THREE.MeshStandardMaterial({ color: 0x1a2a1a, roughness: 1 });
    const ground = new THREE.Mesh(new THREE.PlaneGeometry(200, 200), groundMat);
    ground.rotation.x = -Math.PI / 2;
    if (!isMob) ground.receiveShadow = true;
    scene.add(ground);
    
    // Quality-dependent objects
    let particleSystems = [];
    let buildings = [];
    
    function setupScene(tier) {
        // Clear previous dynamic objects
        particleSystems.forEach(p => { p.geometry.dispose(); p.material.dispose(); scene.remove(p); });
        buildings.forEach(b => { if (b.geometry) { b.geometry.dispose(); b.material.dispose(); } scene.remove(b); });
        particleSystems = [];
        buildings = [];
        
        const settings = qualitySettings[tier];
        
        // Buildings (draw distance aware)
        const geo = new THREE.BoxGeometry(3, 8, 3);
        const mat = new THREE.MeshStandardMaterial({ color: 0x334455, roughness: 0.8 });
        
        for (let i = 0; i < 150; i++) {
            const mesh = new THREE.Mesh(geo, mat);
            const x = (Math.random() - 0.5) * settings.drawDistance;
            const z = (Math.random() - 0.5) * settings.drawDistance;
            const h = 3 + Math.random() * 12;
            mesh.position.set(x, h/2, z);
            mesh.scale.set(1, h/8, 1);
            if (!isMob) mesh.castShadow = true;
            scene.add(mesh);
            buildings.push(mesh);
        }
        
        // Particles
        if (settings.particleCount > 0) {
            const positions = new Float32Array(settings.particleCount * 3);
            for (let i = 0; i < settings.particleCount * 3; i++) {
                positions[i] = (Math.random() - 0.5) * 100;
            }
            const pgeo = new THREE.BufferGeometry();
            pgeo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
            const pmat = new THREE.PointsMaterial({ color: 0x4488ff, size: 0.3, sizeAttenuation: true });
            const particles = new THREE.Points(pgeo, pmat);
            scene.add(particles);
            particleSystems.push(particles);
        }
        
        // Scene fog based on draw distance
        scene.fog = new THREE.Fog(0x050510, settings.drawDistance * 0.6, settings.drawDistance);
    }
    
    // Quality settings
    const qualitySettings = {
        3: { pixelRatioScale: 1.0, shadowsEnabled: true, particleCount: 1000, drawDistance: 300 },
        2: { pixelRatioScale: 0.75, shadowsEnabled: !isMob, particleCount: 200, drawDistance: 150 },
        1: { pixelRatioScale: 0.5, shadowsEnabled: false, particleCount: 50, drawDistance: 80 },
        0: { pixelRatioScale: 0.35, shadowsEnabled: false, particleCount: 0, drawDistance: 40 }
    };
    
    const tierNames = ['Very Low', 'Low', 'Medium', 'High'];
    let currentTier = 2;
    let autoQuality = true;
    
    window.setQualityTier = (tier) => {
        currentTier = tier;
        autoQuality = false;
        document.getElementById('q-tier-name').textContent = tierNames[tier];
        document.getElementById('q-tier-badge').textContent = tierNames[tier].toUpperCase();
        document.getElementById('q-tier-badge').className = `tier-badge tier-${tier}`;
        
        const s = qualitySettings[tier];
        renderer.setPixelRatio(Math.min(devicePixelRatio * s.pixelRatioScale, 2));
        renderer.shadowMap.enabled = s.shadowsEnabled;
        
        document.getElementById('q-pr').textContent = renderer.getPixelRatio().toFixed(2);
        document.getElementById('q-shadow').textContent = s.shadowsEnabled ? 'เปิด ✅' : 'ปิด ❌';
        document.getElementById('q-particles').textContent = s.particleCount;
        document.getElementById('q-dist').textContent = s.drawDistance;
        
        setupScene(tier);
    };
    
    window.toggleAuto = () => {
        autoQuality = !autoQuality;
        const btn = document.getElementById('controls-row').querySelector('button:last-child');
        btn.textContent = `Auto: ${autoQuality ? 'ON' : 'OFF'}`;
    };
    
    // Initialize
    setupScene(currentTier);
    
    // Touch controls
    let touchStart = { x: 0, y: 0 };
    let orbit = { theta: 0.5, phi: 0.8, radius: 40 };
    let prevPinchDist = 0;
    
    canvas.addEventListener('touchstart', e => {
        const t = e.touches[0];
        touchStart = { x: t.clientX, y: t.clientY };
        if (e.touches.length === 2) {
            prevPinchDist = Math.hypot(
                e.touches[0].clientX - e.touches[1].clientX,
                e.touches[0].clientY - e.touches[1].clientY
            );
        }
    }, { passive: true });
    
    canvas.addEventListener('touchmove', e => {
        if (e.touches.length === 1) {
            const t = e.touches[0];
            const dx = (t.clientX - touchStart.x) * 0.005;
            const dy = (t.clientY - touchStart.y) * 0.005;
            orbit.theta -= dx;
            orbit.phi = Math.max(0.2, Math.min(Math.PI/2 - 0.1, orbit.phi - dy));
            touchStart = { x: t.clientX, y: t.clientY };
        } else if (e.touches.length === 2) {
            const dist = Math.hypot(
                e.touches[0].clientX - e.touches[1].clientX,
                e.touches[0].clientY - e.touches[1].clientY
            );
            orbit.radius = Math.max(5, Math.min(80, orbit.radius * (prevPinchDist / dist)));
            prevPinchDist = dist;
        }
    }, { passive: true });
    
    // Mouse orbit
    let mouseDown = false;
    let lastMouse = { x: 0, y: 0 };
    canvas.addEventListener('mousedown', e => { mouseDown = true; lastMouse = { x: e.clientX, y: e.clientY }; });
    document.addEventListener('mouseup', () => { mouseDown = false; });
    document.addEventListener('mousemove', e => {
        if (!mouseDown) return;
        orbit.theta -= (e.clientX - lastMouse.x) * 0.005;
        orbit.phi = Math.max(0.2, Math.min(Math.PI/2 - 0.1, orbit.phi - (e.clientY - lastMouse.y) * 0.005));
        lastMouse = { x: e.clientX, y: e.clientY };
    });
    canvas.addEventListener('wheel', e => {
        orbit.radius = Math.max(5, Math.min(80, orbit.radius + e.deltaY * 0.05));
    });
    
    // Battery API
    async function initBattery() {
        if (!navigator.getBattery) return;
        try {
            const battery = await navigator.getBattery();
            function updateBattery() {
                const pct = Math.round(battery.level * 100);
                const charging = battery.charging;
                document.getElementById('q-battery').textContent = 
                    `${pct}% ${charging ? '⚡' : '🔋'}`;
                document.getElementById('q-battery').className = 
                    `qval ${pct < 20 && !charging ? 'red' : pct < 40 && !charging ? 'yellow' : 'green'}`;
            }
            battery.addEventListener('levelchange', updateBattery);
            battery.addEventListener('chargingchange', updateBattery);
            updateBattery();
        } catch (e) {}
    }
    initBattery();
    
    // FPS tracking for auto quality
    let frameCount = 0, lastFPSTime = performance.now(), fps = 0;
    const fpsHistory = [];
    let autoCheckTimer = 0;
    
    function resize() {
        renderer.setSize(innerWidth, innerHeight);
        camera.aspect = innerWidth / innerHeight;
        camera.updateProjectionMatrix();
    }
    window.addEventListener('resize', resize);
    resize();
    
    let time = 0;
    
    function animate() {
        requestAnimationFrame(animate);
        time += 0.01;
        
        // Update camera
        camera.position.set(
            orbit.radius * Math.sin(orbit.theta) * Math.sin(orbit.phi),
            orbit.radius * Math.cos(orbit.phi),
            orbit.radius * Math.cos(orbit.theta) * Math.sin(orbit.phi)
        );
        camera.lookAt(0, 0, 0);
        
        // Animate particles
        particleSystems.forEach(p => { p.rotation.y += 0.001; });
        
        renderer.render(scene, camera);
        
        // FPS
        frameCount++;
        const now = performance.now();
        if (now - lastFPSTime >= 500) {
            fps = Math.round(frameCount / ((now - lastFPSTime) / 1000));
            frameCount = 0;
            lastFPSTime = now;
            
            fpsHistory.push(fps);
            if (fpsHistory.length > 20) fpsHistory.shift();
        }
        
        // Auto quality
        autoCheckTimer++;
        if (autoQuality && autoCheckTimer >= 120 && fpsHistory.length >= 10) {
            autoCheckTimer = 0;
            const avg = fpsHistory.reduce((a, b) => a + b, 0) / fpsHistory.length;
            if (avg < 25 && currentTier > 0) {
                setQualityTier(currentTier - 1);
            } else if (avg > 55 && currentTier < 3) {
                setQualityTier(currentTier + 1);
            }
        }
        
        // Update UI
        document.getElementById('h-fps').textContent = fps;
        document.getElementById('h-calls').textContent = renderer.info.render.calls;
        document.getElementById('h-pr').textContent = renderer.getPixelRatio().toFixed(1);
        document.getElementById('h-tier').textContent = tierNames[currentTier];
    }
    
    animate();
</script>
</body>
</html>
```

---

## 7. PWA Setup สำหรับ Three.js Apps

```json
// manifest.json
{
    "name": "3D Viewer App",
    "short_name": "3D Viewer",
    "description": "Interactive 3D scene viewer",
    "start_url": "/",
    "display": "standalone",
    "orientation": "any",
    "background_color": "#050510",
    "theme_color": "#050510",
    "icons": [
        {
            "src": "/icon-192.png",
            "sizes": "192x192",
            "type": "image/png",
            "purpose": "maskable"
        },
        {
            "src": "/icon-512.png",
            "sizes": "512x512",
            "type": "image/png"
        }
    ],
    "categories": ["games", "entertainment"]
}
```

```javascript
// service-worker.js สำหรับ cache Three.js
const CACHE_NAME = 'three-app-v1';
const PRECACHE = [
    '/',
    '/index.html',
    'https://cdn.jsdelivr.net/npm/three@0.165.0/build/three.module.js',
    '/models/scene.glb',
    '/textures/atlas.jpg'
];

self.addEventListener('install', (e) => {
    e.waitUntil(
        caches.open(CACHE_NAME).then(cache => cache.addAll(PRECACHE))
    );
});

self.addEventListener('fetch', (e) => {
    e.respondWith(
        caches.match(e.request).then(cached => 
            cached || fetch(e.request).then(response => {
                // Cache GLB models and textures
                if (e.request.url.match(/\.(glb|gltf|jpg|png|ktx2)$/)) {
                    const clone = response.clone();
                    caches.open(CACHE_NAME).then(cache => cache.put(e.request, clone));
                }
                return response;
            })
        )
    );
});
```

---

## แบบฝึกหัด

1. **ง่าย**: สร้าง function `detectGPUTier()` ที่ detect tier จาก WebGL info และ benchmark simple scene
2. **กลาง**: Implement `AdaptiveQualitySystem` ที่ auto-adjust pixel ratio เมื่อ FPS ต่ำกว่า 30fps
3. **ยาก**: สร้าง `MobileOptimizedScene` class ที่ auto-configure shadows, antialias, particle count ตาม device tier
4. **Challenge**: สร้าง PWA 3D viewer ที่ cache assets, รองรับ offline, และ install ได้บน homescreen

---

## อ้างอิง

- [detect-gpu library](https://github.com/TimvanScherpenzeel/detect-gpu)
- [Battery Status API](https://developer.mozilla.org/en-US/docs/Web/API/Battery_Status_API)
- [PWA for Games](https://web.dev/progressively-enhance-your-pwa/)
- [WebGL Mobile Performance](https://developer.chrome.com/docs/devtools/evaluate-performance/)
