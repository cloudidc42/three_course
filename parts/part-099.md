# Part 99: Loading Screens & Scene Transitions

## บทนำ

Loading screen และ scene transition เป็นส่วนที่ทำให้ application ดูมีคุณภาพและ professional หน้า loading ที่ดีไม่ใช่แค่แสดง progress bar แต่ต้องสื่อสารกับ user ว่ากำลังทำอะไร และทำให้การรอคอยรู้สึกสั้นลง

## LoadingManager Events

Three.js มี `LoadingManager` ที่ track การโหลด assets ทั้งหมด:

```javascript
const manager = new THREE.LoadingManager();

// ก่อนเริ่มโหลดไฟล์แรก
manager.onStart = (url, itemsLoaded, itemsTotal) => {
    console.log(`เริ่มโหลด: ${url}`);
    showLoadingScreen();
};

// เมื่อโหลดแต่ละไฟล์เสร็จ
manager.onProgress = (url, itemsLoaded, itemsTotal) => {
    const progress = itemsLoaded / itemsTotal;
    updateProgressBar(progress);
    updateProgressText(`กำลังโหลด: ${url.split('/').pop()}`);
};

// เมื่อโหลดทุกอย่างเสร็จ
manager.onLoad = () => {
    console.log('โหลดเสร็จทั้งหมด!');
    hideLoadingScreen();
};

// เมื่อเกิด error
manager.onError = (url) => {
    console.error(`โหลดไม่สำเร็จ: ${url}`);
    showErrorMessage(`ไม่สามารถโหลดไฟล์: ${url}`);
};

// ส่ง manager ให้ loaders ทุกตัว
const textureLoader = new THREE.TextureLoader(manager);
const gltfLoader = new GLTFLoader(manager);
const audioLoader = new THREE.AudioLoader(manager);
```

## การคำนวณ Progress

```javascript
class AssetLoader {
    constructor() {
        this.totalItems = 0;
        this.loadedItems = 0;
        this.manager = new THREE.LoadingManager();
        this.assets = {};
        
        this.manager.onProgress = (url, loaded, total) => {
            this.loadedItems = loaded;
            this.totalItems = total;
        };
    }
    
    getProgress() {
        if (this.totalItems === 0) return 0;
        return this.loadedItems / this.totalItems;
    }
    
    async loadAll(manifest) {
        return new Promise((resolve) => {
            this.manager.onLoad = () => resolve(this.assets);
            
            manifest.forEach(item => {
                switch (item.type) {
                    case 'texture':
                        new THREE.TextureLoader(this.manager).load(item.url, (tex) => {
                            this.assets[item.name] = tex;
                        });
                        break;
                    case 'model':
                        // กรณีมี GLTFLoader
                        break;
                    case 'audio':
                        new THREE.AudioLoader(this.manager).load(item.url, (buffer) => {
                            this.assets[item.name] = buffer;
                        });
                        break;
                }
            });
        });
    }
}
```

## Minimum Display Time

UX ที่ดีต้องแสดง loading screen อย่างน้อย 1-2 วินาที แม้โหลดเสร็จเร็ว:

```javascript
class LoadingScreenManager {
    constructor(minDisplayTime = 2000) {
        this.minDisplayTime = minDisplayTime;
        this.startTime = null;
        this.isLoaded = false;
        this.isMinTimeElapsed = false;
        this.hideCallback = null;
    }
    
    start() {
        this.startTime = Date.now();
        setTimeout(() => {
            this.isMinTimeElapsed = true;
            this.tryHide();
        }, this.minDisplayTime);
    }
    
    onLoadComplete(callback) {
        this.hideCallback = callback;
        this.isLoaded = true;
        this.tryHide();
    }
    
    tryHide() {
        if (this.isLoaded && this.isMinTimeElapsed) {
            this.hideCallback && this.hideCallback();
        }
    }
}
```

## Demo หลัก: Beautiful Animated Loading Screen

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Loading Screen Demo</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #0a0a1a; overflow: hidden; font-family: 'Segoe UI', Arial, sans-serif; }
        
        /* === Loading Screen === */
        #loading-screen {
            position: fixed; top: 0; left: 0;
            width: 100%; height: 100%;
            z-index: 9999;
            display: flex; flex-direction: column;
            align-items: center; justify-content: center;
            background: radial-gradient(ellipse at center, #0d1b4a 0%, #050918 100%);
            transition: opacity 0.8s ease, transform 0.8s ease;
        }
        #loading-screen.hidden {
            opacity: 0;
            pointer-events: none;
        }
        
        /* Logo */
        .loading-logo {
            font-size: 48px;
            margin-bottom: 24px;
            animation: logoFloat 3s ease-in-out infinite;
        }
        @keyframes logoFloat {
            0%, 100% { transform: translateY(0); }
            50% { transform: translateY(-12px); }
        }
        
        .loading-title {
            font-size: 28px;
            font-weight: 700;
            color: white;
            margin-bottom: 8px;
            letter-spacing: 0.1em;
            text-transform: uppercase;
        }
        .loading-subtitle {
            font-size: 13px;
            color: rgba(255,255,255,0.4);
            margin-bottom: 48px;
            letter-spacing: 0.05em;
        }
        
        /* Progress bar */
        .progress-container {
            width: 340px;
            margin-bottom: 20px;
        }
        .progress-bar-bg {
            width: 100%; height: 4px;
            background: rgba(255,255,255,0.1);
            border-radius: 2px;
            overflow: hidden;
        }
        .progress-bar-fill {
            height: 100%;
            background: linear-gradient(to right, #4466ff, #44aaff);
            border-radius: 2px;
            width: 0%;
            transition: width 0.3s ease;
            position: relative;
        }
        .progress-bar-fill::after {
            content: '';
            position: absolute;
            right: 0; top: -2px;
            width: 8px; height: 8px;
            background: white;
            border-radius: 50%;
            box-shadow: 0 0 8px white, 0 0 16px #4488ff;
        }
        
        /* Percentage */
        .progress-percent {
            text-align: right;
            font-size: 24px;
            font-weight: bold;
            color: white;
            margin-top: 12px;
            font-variant-numeric: tabular-nums;
        }
        
        /* Current file */
        .loading-file {
            font-size: 11px;
            color: rgba(255,255,255,0.3);
            margin-bottom: 40px;
            text-align: center;
            max-width: 340px;
            overflow: hidden;
            text-overflow: ellipsis;
            white-space: nowrap;
        }
        
        /* Tips */
        .loading-tips {
            position: absolute;
            bottom: 40px;
            text-align: center;
            width: 100%;
        }
        .tip-label {
            font-size: 11px;
            color: rgba(255,255,255,0.3);
            margin-bottom: 6px;
            text-transform: uppercase;
            letter-spacing: 0.1em;
        }
        .tip-text {
            font-size: 13px;
            color: rgba(255,255,255,0.6);
            max-width: 400px;
            margin: 0 auto;
            line-height: 1.6;
            transition: opacity 0.5s;
        }
        
        /* Spinner rings */
        .spinner-container {
            position: relative;
            width: 80px; height: 80px;
            margin-bottom: 32px;
        }
        .spinner-ring {
            position: absolute;
            border-radius: 50%;
            border: 2px solid transparent;
        }
        .spinner-ring:nth-child(1) {
            width: 80px; height: 80px;
            border-top-color: #4466ff;
            animation: spin 1.5s linear infinite;
        }
        .spinner-ring:nth-child(2) {
            width: 60px; height: 60px;
            top: 10px; left: 10px;
            border-top-color: #44aaff;
            border-right-color: #44aaff;
            animation: spin 1.2s linear infinite reverse;
        }
        .spinner-ring:nth-child(3) {
            width: 40px; height: 40px;
            top: 20px; left: 20px;
            border-top-color: rgba(255,255,255,0.5);
            animation: spin 0.9s linear infinite;
        }
        .spinner-center {
            position: absolute;
            top: 50%; left: 50%;
            transform: translate(-50%, -50%);
            width: 16px; height: 16px;
            background: #4466ff;
            border-radius: 50%;
            box-shadow: 0 0 12px #4466ff;
        }
        @keyframes spin {
            to { transform: rotate(360deg); }
        }
        
        /* Step indicators */
        .loading-steps {
            display: flex;
            gap: 12px;
            margin-bottom: 32px;
        }
        .step {
            display: flex;
            align-items: center;
            gap: 6px;
            font-size: 11px;
            color: rgba(255,255,255,0.3);
            transition: color 0.3s;
        }
        .step.active { color: rgba(100,170,255,0.9); }
        .step.done { color: rgba(100,255,100,0.7); }
        .step-icon {
            width: 16px; height: 16px;
            border-radius: 50%;
            border: 1px solid currentColor;
            display: flex; align-items: center; justify-content: center;
            font-size: 9px;
        }
        .step.done .step-icon {
            background: rgba(100,255,100,0.2);
        }
        
        /* === Main Canvas === */
        #canvas {
            position: fixed; top: 0; left: 0;
            width: 100%; height: 100%;
            opacity: 0;
            transition: opacity 1.2s ease;
        }
        #canvas.visible { opacity: 1; }
    </style>
</head>
<body>
    <canvas id="canvas"></canvas>
    
    <div id="loading-screen">
        <!-- Spinner -->
        <div class="spinner-container">
            <div class="spinner-ring"></div>
            <div class="spinner-ring"></div>
            <div class="spinner-ring"></div>
            <div class="spinner-center"></div>
        </div>
        
        <div class="loading-title">Three.js Universe</div>
        <div class="loading-subtitle">กำลังเตรียม 3D Experience</div>
        
        <!-- Steps -->
        <div class="loading-steps">
            <div class="step" id="step-textures">
                <div class="step-icon">1</div>
                <span>Textures</span>
            </div>
            <div class="step" id="step-models">
                <div class="step-icon">2</div>
                <span>Models</span>
            </div>
            <div class="step" id="step-shaders">
                <div class="step-icon">3</div>
                <span>Shaders</span>
            </div>
            <div class="step" id="step-scene">
                <div class="step-icon">4</div>
                <span>Scene</span>
            </div>
        </div>
        
        <!-- Progress -->
        <div class="progress-container">
            <div class="progress-bar-bg">
                <div class="progress-bar-fill" id="progress-fill"></div>
            </div>
            <div class="progress-percent" id="progress-percent">0%</div>
        </div>
        
        <div class="loading-file" id="loading-file">กำลังเริ่มต้น...</div>
        
        <!-- Tips -->
        <div class="loading-tips">
            <div class="tip-label">💡 เคล็ดลับ</div>
            <div class="tip-text" id="tip-text">
                Three.js รองรับ WebGL 2.0 ที่ให้ประสิทธิภาพสูงกว่า WebGL 1.0 มาก
            </div>
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

        const tips = [
            'Three.js รองรับ WebGL 2.0 ที่ให้ประสิทธิภาพสูงกว่า WebGL 1.0 มาก',
            'ใช้ BufferGeometry แทน Geometry เสมอ เพราะเร็วกว่ามาก',
            'LOD (Level of Detail) ช่วยลด polygon count สำหรับ objects ที่อยู่ไกล',
            'Instanced Mesh เหมาะสำหรับ objects ที่ซ้ำกันจำนวนมาก',
            'OrbitControls ไม่ควรใช้ใน production โดยไม่ปรับแต่ง',
        ];
        
        let tipIndex = 0;
        const tipEl = document.getElementById('tip-text');
        
        // Rotate tips
        setInterval(() => {
            tipEl.style.opacity = '0';
            setTimeout(() => {
                tipIndex = (tipIndex + 1) % tips.length;
                tipEl.textContent = tips[tipIndex];
                tipEl.style.opacity = '1';
            }, 500);
        }, 4000);
        
        // Loading steps
        const steps = ['textures', 'models', 'shaders', 'scene'];
        let currentStep = 0;
        
        function setStep(stepIndex) {
            steps.forEach((s, i) => {
                const el = document.getElementById(`step-${s}`);
                if (i < stepIndex) {
                    el.className = 'step done';
                    el.querySelector('.step-icon').textContent = '✓';
                } else if (i === stepIndex) {
                    el.className = 'step active';
                    el.querySelector('.step-icon').textContent = (i + 1).toString();
                } else {
                    el.className = 'step';
                    el.querySelector('.step-icon').textContent = (i + 1).toString();
                }
            });
        }
        
        // Simulated loading progress
        let progress = 0;
        let targetProgress = 0;
        const progressFill = document.getElementById('progress-fill');
        const progressPercent = document.getElementById('progress-percent');
        const loadingFile = document.getElementById('loading-file');
        
        const loadingSteps = [
            { target: 0.25, step: 0, file: 'กำลังโหลด textures...' },
            { target: 0.55, step: 1, file: 'กำลังโหลด 3D models...' },
            { target: 0.80, step: 2, file: 'กำลังคอมไพล์ shaders...' },
            { target: 0.95, step: 3, file: 'กำลังสร้าง scene...' },
            { target: 1.00, step: 3, file: 'เตรียมพร้อมแล้ว!' },
        ];
        
        let stepIdx = 0;
        let loaded = false;
        
        function simulateLoading() {
            if (stepIdx >= loadingSteps.length) return;
            
            const curr = loadingSteps[stepIdx];
            targetProgress = curr.target;
            setStep(curr.step);
            loadingFile.textContent = curr.file;
            stepIdx++;
            
            if (stepIdx < loadingSteps.length) {
                setTimeout(simulateLoading, 800 + Math.random() * 400);
            } else {
                // โหลดเสร็จ
                setTimeout(() => {
                    loaded = true;
                    finishLoading();
                }, 500);
            }
        }
        
        setTimeout(simulateLoading, 500);
        
        // Smooth progress animation
        function updateProgress() {
            progress += (targetProgress - progress) * 0.08;
            progressFill.style.width = `${progress * 100}%`;
            progressPercent.textContent = `${Math.round(progress * 100)}%`;
            if (!loaded) requestAnimationFrame(updateProgress);
        }
        updateProgress();
        
        // Three.js scene setup
        const W = window.innerWidth, H = window.innerHeight;
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x050918);
        
        const camera = new THREE.PerspectiveCamera(75, W / H, 0.1, 1000);
        camera.position.set(0, 10, 30);
        
        const renderer = new THREE.WebGLRenderer({
            canvas: document.getElementById('canvas'),
            antialias: true,
        });
        renderer.setSize(W, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        
        // Build scene immediately in background
        buildScene();
        
        function buildScene() {
            scene.add(new THREE.AmbientLight(0x334466, 1.5));
            const dirLight = new THREE.DirectionalLight(0x8899ff, 2);
            dirLight.position.set(10, 20, 10);
            scene.add(dirLight);
            
            // Star field
            const starsGeo = new THREE.BufferGeometry();
            const starPos = new Float32Array(3000 * 3);
            for (let i = 0; i < starPos.length; i++) {
                starPos[i] = (Math.random() - 0.5) * 500;
            }
            starsGeo.setAttribute('position', new THREE.BufferAttribute(starPos, 3));
            scene.add(new THREE.Points(starsGeo, new THREE.PointsMaterial({ color: 0xffffff, size: 0.3, sizeAttenuation: true })));
            
            // Planet
            const planet = new THREE.Mesh(
                new THREE.SphereGeometry(8, 64, 64),
                new THREE.MeshStandardMaterial({
                    color: 0x4466aa,
                    roughness: 0.7,
                    metalness: 0.1,
                })
            );
            scene.add(planet);
            
            // Rings
            const ringGeo = new THREE.TorusGeometry(14, 1.5, 4, 64);
            const ringMat = new THREE.MeshStandardMaterial({
                color: 0x8899aa,
                roughness: 0.5,
                transparent: true,
                opacity: 0.6,
            });
            const ring = new THREE.Mesh(ringGeo, ringMat);
            ring.rotation.x = Math.PI / 3;
            scene.add(ring);
            
            // Moons
            for (let i = 0; i < 4; i++) {
                const moon = new THREE.Mesh(
                    new THREE.SphereGeometry(1 + Math.random(), 16, 16),
                    new THREE.MeshStandardMaterial({ color: 0x999988 })
                );
                const angle = (i / 4) * Math.PI * 2;
                const dist = 15 + Math.random() * 5;
                moon.position.set(Math.cos(angle) * dist, (Math.random() - 0.5) * 3, Math.sin(angle) * dist);
                moon.userData = { angle, dist, speed: 0.3 + Math.random() * 0.3, orbitY: (Math.random() - 0.5) * 0.5 };
                scene.add(moon);
            }
            
            const controls = new OrbitControls(camera, renderer.domElement);
            controls.enableDamping = true;
            controls.autoRotate = true;
            controls.autoRotateSpeed = 0.5;
            
            let t = 0;
            function animateScene() {
                requestAnimationFrame(animateScene);
                t = performance.now() * 0.001;
                planet.rotation.y = t * 0.1;
                ring.rotation.z = t * 0.05;
                controls.update();
                renderer.render(scene, camera);
            }
            animateScene();
        }
        
        function finishLoading() {
            const loadingScreen = document.getElementById('loading-screen');
            const canvas = document.getElementById('canvas');
            
            // Minimum display time already handled by simulation
            setTimeout(() => {
                loadingScreen.classList.add('hidden');
                canvas.classList.add('visible');
                
                // Remove from DOM after transition
                setTimeout(() => {
                    loadingScreen.style.display = 'none';
                }, 1000);
            }, 600);
        }
        
        window.addEventListener('resize', () => {
            const w = window.innerWidth, h = window.innerHeight;
            camera.aspect = w / h;
            camera.updateProjectionMatrix();
            renderer.setSize(w, h);
        });
    </script>
</body>
</html>
```

## Scene Transitions ด้วย 3D Effect

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>3D Scene Transitions</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; }
        canvas { display: block; }
        
        #transition-overlay {
            position: fixed; top: 0; left: 0;
            width: 100%; height: 100%;
            pointer-events: none;
            z-index: 50;
        }
        
        /* Circle wipe transition */
        .circle-wipe {
            position: absolute; top: 0; left: 0;
            width: 100%; height: 100%;
            background: #0a0a1a;
            clip-path: circle(150% at 50% 50%);
            transition: clip-path 0.8s cubic-bezier(0.77, 0, 0.18, 1);
        }
        .circle-wipe.closing {
            clip-path: circle(0% at 50% 50%);
        }
        .circle-wipe.opening {
            clip-path: circle(150% at 50% 50%);
        }
        
        /* Pixel dissolve */
        .pixel-overlay {
            position: absolute; top: 0; left: 0;
            width: 100%; height: 100%;
            background: #0a0a1a;
            opacity: 0;
            transition: opacity 0.5s;
        }
        .pixel-overlay.active { opacity: 1; }
        
        /* Scene selector */
        #scene-selector {
            position: fixed; bottom: 20px; left: 50%;
            transform: translateX(-50%);
            display: flex; gap: 8px; z-index: 100;
        }
        .scene-btn {
            background: rgba(30,30,60,0.9);
            border: 1px solid rgba(100,100,200,0.4);
            color: white; padding: 8px 18px;
            border-radius: 20px; cursor: pointer;
            font-size: 12px; transition: all 0.2s;
        }
        .scene-btn:hover { background: rgba(60,60,120,0.9); }
        .scene-btn.active { background: rgba(68,102,255,0.6); border-color: #4466ff; }
    </style>
</head>
<body>
    <canvas id="canvas"></canvas>
    <div id="transition-overlay">
        <div class="circle-wipe" id="wipe"></div>
    </div>
    <div id="scene-selector">
        <button class="scene-btn active" data-scene="0">🌌 Space</button>
        <button class="scene-btn" data-scene="1">🌊 Ocean</button>
        <button class="scene-btn" data-scene="2">🏔️ Mountain</button>
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

        const W = window.innerWidth, H = window.innerHeight;
        
        let scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(75, W / H, 0.1, 1000);
        camera.position.set(0, 5, 20);
        
        const renderer = new THREE.WebGLRenderer({ canvas: document.getElementById('canvas'), antialias: true });
        renderer.setSize(W, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        
        // Scene builders
        function buildSpaceScene() {
            const s = new THREE.Scene();
            s.background = new THREE.Color(0x000010);
            
            s.add(new THREE.AmbientLight(0x334466, 1));
            s.add(new THREE.PointLight(0xffffff, 2, 500));
            
            // Stars
            const geo = new THREE.BufferGeometry();
            const pos = new Float32Array(5000 * 3);
            for (let i = 0; i < pos.length; i++) pos[i] = (Math.random() - 0.5) * 500;
            geo.setAttribute('position', new THREE.BufferAttribute(pos, 3));
            s.add(new THREE.Points(geo, new THREE.PointsMaterial({ color: 0xffffff, size: 0.5 })));
            
            // Planet
            const planet = new THREE.Mesh(
                new THREE.SphereGeometry(5, 32, 32),
                new THREE.MeshStandardMaterial({ color: 0x4466ff, roughness: 0.6 })
            );
            s.add(planet);
            
            s._update = (t) => { planet.rotation.y = t * 0.2; };
            return s;
        }
        
        function buildOceanScene() {
            const s = new THREE.Scene();
            s.background = new THREE.Color(0x001122);
            s.fog = new THREE.FogExp2(0x001122, 0.02);
            
            s.add(new THREE.AmbientLight(0x224466, 2));
            const sunLight = new THREE.DirectionalLight(0x88aaff, 1.5);
            sunLight.position.set(50, 80, 30);
            s.add(sunLight);
            
            // Ocean plane
            const oceanGeo = new THREE.PlaneGeometry(100, 100, 50, 50);
            const oceanMat = new THREE.MeshStandardMaterial({
                color: 0x006688,
                roughness: 0.1,
                metalness: 0.8,
                transparent: true,
                opacity: 0.9,
            });
            const ocean = new THREE.Mesh(oceanGeo, oceanMat);
            ocean.rotation.x = -Math.PI / 2;
            s.add(ocean);
            
            // Bubbles
            for (let i = 0; i < 50; i++) {
                const bubble = new THREE.Mesh(
                    new THREE.SphereGeometry(0.1 + Math.random() * 0.4, 8, 8),
                    new THREE.MeshBasicMaterial({ color: 0xaaddff, transparent: true, opacity: 0.3 })
                );
                bubble.position.set(
                    (Math.random() - 0.5) * 40,
                    Math.random() * 15,
                    (Math.random() - 0.5) * 40
                );
                bubble.userData.speed = 0.5 + Math.random() * 1.5;
                bubble.userData.startY = bubble.position.y;
                s.add(bubble);
            }
            
            s._update = (t) => {
                // Wave ocean
                const pos = oceanGeo.attributes.position;
                for (let i = 0; i < pos.count; i++) {
                    const x = pos.getX(i);
                    const z = pos.getZ(i);
                    pos.setY(i, Math.sin(x * 0.3 + t * 1.5) * 0.5 + Math.cos(z * 0.4 + t) * 0.3);
                }
                pos.needsUpdate = true;
                oceanGeo.computeVertexNormals();
                
                // Float bubbles
                s.children.forEach(child => {
                    if (child.userData.speed) {
                        child.position.y += child.userData.speed * 0.02;
                        if (child.position.y > 20) child.position.y = child.userData.startY;
                    }
                });
            };
            return s;
        }
        
        function buildMountainScene() {
            const s = new THREE.Scene();
            s.background = new THREE.Color(0x334455);
            s.fog = new THREE.FogExp2(0x334455, 0.01);
            
            s.add(new THREE.AmbientLight(0x445566, 1.5));
            const sun = new THREE.DirectionalLight(0xffeedd, 2);
            sun.position.set(-50, 80, -30);
            s.add(sun);
            
            // Terrain
            const size = 100;
            const segs = 60;
            const terrGeo = new THREE.PlaneGeometry(size, size, segs, segs);
            const pos = terrGeo.attributes.position;
            for (let i = 0; i < pos.count; i++) {
                const x = pos.getX(i);
                const z = pos.getY(i);
                const noise = Math.sin(x * 0.3) * Math.cos(z * 0.25) * 8
                    + Math.sin(x * 0.1 + z * 0.15) * 15
                    + Math.random() * 1.5;
                pos.setZ(i, noise);
            }
            pos.needsUpdate = true;
            terrGeo.computeVertexNormals();
            
            const terrMat = new THREE.MeshStandardMaterial({
                color: 0x445533,
                roughness: 0.9,
            });
            const terrain = new THREE.Mesh(terrGeo, terrMat);
            terrain.rotation.x = -Math.PI / 2;
            terrain.position.y = -5;
            s.add(terrain);
            
            // Snow caps
            const snowGeo = new THREE.PlaneGeometry(size, size, segs, segs);
            const snowPos = snowGeo.attributes.position;
            for (let i = 0; i < pos.count; i++) {
                snowPos.setXYZ(i, pos.getX(i), pos.getY(i), Math.max(0, pos.getZ(i) - 10));
            }
            snowPos.needsUpdate = true;
            const snowMat = new THREE.MeshStandardMaterial({ color: 0xeeeeff, roughness: 0.5 });
            const snow = new THREE.Mesh(snowGeo, snowMat);
            snow.rotation.x = -Math.PI / 2;
            snow.position.y = 4.5;
            s.add(snow);
            
            s._update = (t) => {};
            return s;
        }
        
        // Scene management
        const sceneBuilders = [buildSpaceScene, buildOceanScene, buildMountainScene];
        let currentSceneIndex = 0;
        let scenes = [];
        let isTransitioning = false;
        
        // Preload all scenes
        sceneBuilders.forEach((builder, i) => {
            scenes.push(builder());
        });
        
        scene = scenes[0];
        
        // Camera positions per scene
        const cameraPositions = [
            new THREE.Vector3(0, 5, 25),
            new THREE.Vector3(0, 15, 30),
            new THREE.Vector3(0, 20, 40),
        ];
        
        // Transition function
        async function transitionToScene(index) {
            if (isTransitioning || index === currentSceneIndex) return;
            isTransitioning = true;
            
            const wipe = document.getElementById('wipe');
            
            // Phase 1: Wipe in (cover screen)
            wipe.style.transition = 'clip-path 0.5s cubic-bezier(0.77, 0, 0.18, 1)';
            wipe.style.clipPath = 'circle(0% at 50% 50%)';
            
            await new Promise(r => setTimeout(r, 500));
            
            // Phase 2: Switch scene
            currentSceneIndex = index;
            scene = scenes[index];
            
            const targetPos = cameraPositions[index];
            camera.position.copy(targetPos);
            controls.target.set(0, 0, 0);
            controls.update();
            
            // Render one frame of new scene
            renderer.render(scene, camera);
            
            await new Promise(r => setTimeout(r, 100));
            
            // Phase 3: Wipe out (reveal)
            wipe.style.transition = 'clip-path 0.7s cubic-bezier(0.77, 0, 0.18, 1)';
            wipe.style.clipPath = 'circle(150% at 50% 50%)';
            
            await new Promise(r => setTimeout(r, 700));
            
            isTransitioning = false;
        }
        
        // Buttons
        document.querySelectorAll('.scene-btn').forEach(btn => {
            btn.addEventListener('click', () => {
                document.querySelectorAll('.scene-btn').forEach(b => b.classList.remove('active'));
                btn.classList.add('active');
                transitionToScene(parseInt(btn.dataset.scene));
            });
        });
        
        let t = 0;
        function animate() {
            requestAnimationFrame(animate);
            t = performance.now() * 0.001;
            
            if (scene._update) scene._update(t);
            
            controls.update();
            renderer.render(scene, camera);
        }
        animate();
        
        window.addEventListener('resize', () => {
            const w = window.innerWidth, h = window.innerHeight;
            camera.aspect = w / h;
            camera.updateProjectionMatrix();
            renderer.setSize(w, h);
        });
    </script>
</body>
</html>
```

## Loading Bar Shader Animation

```glsl
/* Fragment Shader สำหรับ animated progress bar */
uniform float uProgress;
uniform float uTime;

void main() {
    vec2 uv = vUv;
    
    // Base color
    vec3 emptyColor = vec3(0.1, 0.1, 0.2);
    vec3 fillColor = vec3(0.2, 0.5, 1.0);
    
    // Progress
    float filled = step(uv.x, uProgress);
    vec3 color = mix(emptyColor, fillColor, filled);
    
    // Shimmer effect on fill
    if (filled > 0.5) {
        float shimmer = sin(uv.x * 20.0 - uTime * 3.0) * 0.5 + 0.5;
        color += vec3(shimmer * 0.15);
    }
    
    // Edge glow
    float edgeDist = abs(uv.x - uProgress);
    if (edgeDist < 0.02) {
        float glow = 1.0 - edgeDist / 0.02;
        color += vec3(0.3, 0.6, 1.0) * glow;
    }
    
    gl_FragColor = vec4(color, 1.0);
}
```

```javascript
// Three.js usage
const progressBarMat = new THREE.ShaderMaterial({
    uniforms: {
        uProgress: { value: 0 },
        uTime: { value: 0 },
    },
    vertexShader: `
        varying vec2 vUv;
        void main() {
            vUv = uv;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
    `,
    fragmentShader: `/* shader above */`,
});

// Update progress
manager.onProgress = (url, loaded, total) => {
    progressBarMat.uniforms.uProgress.value = loaded / total;
};

// Update time
function animate() {
    progressBarMat.uniforms.uTime.value = performance.now() * 0.001;
}
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Asset Preloader
สร้าง preloader จริงที่โหลด textures จาก URL เปิดเผย (เช่น Three.js examples) แสดง progress แบบ real-time

### แบบฝึกหัดที่ 2: Staggered Reveal
สร้าง scene ที่ objects ค่อยๆ fade in ทีละตัวหลัง loading เสร็จ (staggered animation)

### แบบฝึกหัดที่ 3: Transition Gallery
สร้าง gallery 3 ห้องที่สามารถเดินทาง transit ระหว่างกันได้ด้วย transition effect แตกต่างกัน (slide, zoom, fade)

### แบบฝึกหัดที่ 4: Error Handling
พัฒนา loading system ที่จัดการ error อย่างสง่างาม แสดง retry button เมื่อโหลดล้มเหลว

## สรุป

Loading screen ที่ดีต้องมี:
1. **Visual feedback** ที่ชัดเจน (progress bar, percentage)
2. **Minimum display time** เพื่อหลีกเลี่ยง flash
3. **Tips/messages** เพื่อ entertain ผู้ใช้ขณะรอ
4. **Smooth transition** เมื่อโหลดเสร็จ
5. **Error handling** ที่ graceful

Scene transitions ที่ดีช่วยให้ UX ดูลื่นไหลและ professional มากขึ้น
