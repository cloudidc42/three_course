# Part 109: Progressive Enhancement

## บทนำ: หลักการ Progressive Enhancement ใน 3D Web

**Progressive Enhancement** คือแนวคิดในการพัฒนา web ที่เริ่มจากฐานที่ทุกคนสามารถใช้ได้ แล้วค่อยๆ เพิ่มฟีเจอร์สำหรับ browsers/devices ที่รองรับ

```
Layer 1 (ทุกคนเข้าถึงได้):
  → Static image fallback (no WebGL, no JS)
  → HTML/CSS description of the 3D scene

Layer 2 (รองรับ WebGL1):
  → Basic 3D scene, no shadows, basic materials
  → WebGL 1.0 only features

Layer 3 (รองรับ WebGL2):
  → Better materials, shadows, post-processing
  → WebGL 2.0 features (instancing, transform feedback)

Layer 4 (รองรับ WebXR):
  → VR/AR experience
  → WebXR device API

Layer 5 (High-end + Fast internet):
  → Progressive loading, full resolution assets
  → Heavy post-processing effects
```

---

## 1. Feature Detection

```javascript
// ============== WebGL Support Detection ==============
function detectWebGLSupport() {
    const canvas = document.createElement('canvas');
    
    // ลอง WebGL2 ก่อน
    const gl2 = canvas.getContext('webgl2');
    if (gl2) {
        return {
            supported: true,
            version: 2,
            context: gl2,
            features: getWebGL2Features(gl2)
        };
    }
    
    // Fallback WebGL1
    const gl1 = canvas.getContext('webgl') || canvas.getContext('experimental-webgl');
    if (gl1) {
        return {
            supported: true,
            version: 1,
            context: gl1,
            features: getWebGL1Features(gl1)
        };
    }
    
    return { supported: false, version: 0 };
}

function getWebGL2Features(gl) {
    return {
        // WebGL2 specific
        instancing: true, // native in WebGL2
        transformFeedback: true,
        uboSupport: true, // Uniform Buffer Objects
        multipleRenderTargets: true,
        depthTexture: true,
        // Extensions
        anisotropy: getAnisotropy(gl),
        floatTextures: !!gl.getExtension('EXT_color_buffer_float'),
        halfFloatTextures: !!gl.getExtension('EXT_color_buffer_half_float'),
        compressedTextureETC: !!gl.getExtension('WEBGL_compressed_texture_etc'),
        compressedTextureASTC: !!gl.getExtension('WEBGL_compressed_texture_astc'),
        maxTextureSize: gl.getParameter(gl.MAX_TEXTURE_SIZE),
        maxAnisotropy: getMaxAnisotropy(gl)
    };
}

function getWebGL1Features(gl) {
    return {
        instancing: !!gl.getExtension('ANGLE_instanced_arrays'),
        transformFeedback: false,
        uboSupport: false,
        multipleRenderTargets: !!gl.getExtension('WEBGL_draw_buffers'),
        depthTexture: !!gl.getExtension('WEBGL_depth_texture'),
        anisotropy: getAnisotropy(gl),
        floatTextures: !!gl.getExtension('OES_texture_float'),
        compressedTextureETC: false,
        compressedTextureASTC: false,
        maxTextureSize: gl.getParameter(gl.MAX_TEXTURE_SIZE),
        maxAnisotropy: getMaxAnisotropy(gl)
    };
}

function getAnisotropy(gl) {
    return gl.getExtension('EXT_texture_filter_anisotropic') || 
           gl.getExtension('WEBKIT_EXT_texture_filter_anisotropic');
}

function getMaxAnisotropy(gl) {
    const ext = getAnisotropy(gl);
    return ext ? gl.getParameter(ext.MAX_TEXTURE_MAX_ANISOTROPY_EXT) : 1;
}

// ============== Three.js Capabilities Check ==============
function checkThreeCapabilities(renderer) {
    const caps = renderer.capabilities;
    return {
        isWebGL2: caps.isWebGL2,
        maxTextures: caps.maxTextures,
        maxTextureSize: caps.maxTextureSize,
        maxCubemapSize: caps.maxCubemapSize,
        maxAttributes: caps.maxAttributes,
        maxVaryings: caps.maxVaryings,
        maxVertexUniforms: caps.maxVertexUniforms,
        maxFragmentUniforms: caps.maxFragmentUniforms,
        precision: caps.precision,         // 'highp', 'mediump', 'lowp'
        vertexTextures: caps.vertexTextures,
        logarithmicDepthBuffer: caps.logarithmicDepthBuffer
    };
}

// ============== WebXR Detection ==============
async function detectWebXR() {
    if (!navigator.xr) return { supported: false };
    
    const vrSupported = await navigator.xr.isSessionSupported('immersive-vr').catch(() => false);
    const arSupported = await navigator.xr.isSessionSupported('immersive-ar').catch(() => false);
    const inlineSupported = await navigator.xr.isSessionSupported('inline').catch(() => false);
    
    return {
        supported: vrSupported || arSupported || inlineSupported,
        vr: vrSupported,
        ar: arSupported,
        inline: inlineSupported
    };
}

// ============== High DPI Support ==============
function getOptimalPixelRatio() {
    const dpr = window.devicePixelRatio || 1;
    
    // Cap based on device type
    const isHighEnd = dpr >= 3 && navigator.hardwareConcurrency >= 8;
    const isMobile = /Android|iPhone|iPad/i.test(navigator.userAgent);
    
    if (isHighEnd && !isMobile) return Math.min(dpr, 3);
    if (isMobile) return Math.min(dpr, 2);
    return Math.min(dpr, 2);
}
```

---

## 2. Graceful Degradation Strategy

```javascript
class GracefulDegradationSystem {
    constructor() {
        this.webglInfo = detectWebGLSupport();
        this.capabilities = null;
        this.xrInfo = null;
        this.renderer = null;
    }
    
    async initialize(canvas) {
        if (!this.webglInfo.supported) {
            return this.showNoWebGLFallback(canvas);
        }
        
        // สร้าง renderer ด้วย settings ที่เหมาะสม
        const rendererOptions = this.getRendererOptions();
        
        try {
            this.renderer = new THREE.WebGLRenderer({ 
                canvas, 
                ...rendererOptions 
            });
        } catch (e) {
            console.error('WebGL init failed:', e);
            return this.showNoWebGLFallback(canvas);
        }
        
        this.capabilities = checkThreeCapabilities(this.renderer);
        this.xrInfo = await detectWebXR();
        
        return this.getFeatureSet();
    }
    
    getRendererOptions() {
        const { version, features } = this.webglInfo;
        
        const base = {
            powerPreference: 'high-performance',
            antialias: features?.maxTextureSize >= 4096 // Only if GPU capable
        };
        
        if (version === 1) {
            // WebGL1 limitations
            return {
                ...base,
                antialias: false,
                precision: 'mediump',
                logarithmicDepthBuffer: false
            };
        }
        
        return base;
    }
    
    getFeatureSet() {
        const { version, features } = this.webglInfo;
        const caps = this.capabilities;
        
        return {
            shadows: version >= 2,
            hdr: features?.floatTextures && version >= 2,
            ssao: version >= 2 && features?.multipleRenderTargets,
            taa: version >= 2,
            anisotropy: features?.maxAnisotropy > 1,
            maxAnisotropy: features?.maxAnisotropy || 1,
            webxr: this.xrInfo?.supported || false,
            instancing: features?.instancing || false,
            compressionASTC: features?.compressedTextureASTC || false,
            compressionETC: features?.compressedTextureETC || false
        };
    }
    
    showNoWebGLFallback(canvas) {
        const parent = canvas.parentElement;
        canvas.style.display = 'none';
        
        parent.innerHTML = `
            <div style="
                display: flex; flex-direction: column; align-items: center;
                justify-content: center; height: 100vh;
                background: #050510; color: #eee; font-family: sans-serif;
                text-align: center; padding: 40px;
            ">
                <img src="fallback-preview.jpg" alt="3D Scene Preview" 
                     style="max-width: 80%; border-radius: 8px; margin-bottom: 20px;"
                     onerror="this.style.display='none'">
                <h2 style="color: #f44; margin-bottom: 10px">WebGL ไม่รองรับ</h2>
                <p style="color: #888; max-width: 400px">
                    Browser ของคุณไม่รองรับ WebGL ซึ่งจำเป็นสำหรับการแสดง 3D Graphics
                    กรุณาใช้ Chrome, Firefox, Safari, หรือ Edge รุ่นล่าสุด
                </p>
                <a href="static-preview.html" style="
                    margin-top: 20px; padding: 10px 20px;
                    background: #0af; color: #000; border-radius: 4px;
                    text-decoration: none; font-weight: bold;
                ">ดู Static Preview แทน</a>
            </div>
        `;
        
        return null;
    }
}
```

---

## 3. Reduced Motion Support (prefers-reduced-motion)

```javascript
class MotionPreferences {
    constructor() {
        this.reducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
        
        // Listen for changes
        window.matchMedia('(prefers-reduced-motion: reduce)')
            .addEventListener('change', (e) => {
                this.reducedMotion = e.matches;
                this.onPreferenceChange?.(this.reducedMotion);
            });
    }
    
    // Adapter: ปรับ animation settings ตาม preference
    getAnimationSettings() {
        if (this.reducedMotion) {
            return {
                enableAutoRotate: false,        // ปิด auto-rotate
                particleAnimation: false,       // ปิด particle movement
                cameraShake: false,             // ปิด camera shake
                transitionDuration: 0,          // ปิด transitions
                loopAnimations: false,          // ปิด loop animations
                blinkEffects: false,            // ปิด blinking effects
                motionBlur: false               // ปิด motion blur
            };
        }
        
        return {
            enableAutoRotate: true,
            particleAnimation: true,
            cameraShake: true,
            transitionDuration: 300,
            loopAnimations: true,
            blinkEffects: true,
            motionBlur: true
        };
    }
    
    // Clamp animation duration
    clampDuration(ms) {
        return this.reducedMotion ? 0 : ms;
    }
    
    // Clamp rotation speed
    clampRotationSpeed(speed) {
        return this.reducedMotion ? 0 : speed;
    }
}

// CSS Media Query ใน Three.js
const motionPrefs = new MotionPreferences();
const animSettings = motionPrefs.getAnimationSettings();

// ใช้กับ OrbitControls
controls.autoRotate = animSettings.enableAutoRotate;
controls.autoRotateSpeed = animSettings.enableAutoRotate ? 0.5 : 0;

// ใช้กับ animations
if (mixer && !animSettings.loopAnimations) {
    mixer.timeScale = 0; // หยุด animation
}
```

---

## 4. Color Scheme Support (prefers-color-scheme)

```javascript
class ColorSchemeAdapter {
    constructor(scene, lights) {
        this.scene = scene;
        this.lights = lights;
        this.isDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
        
        window.matchMedia('(prefers-color-scheme: dark)')
            .addEventListener('change', (e) => {
                this.isDark = e.matches;
                this.applyColorScheme();
            });
        
        this.schemes = {
            dark: {
                background: new THREE.Color(0x050510),
                fog: new THREE.Color(0x050510),
                ambientLight: 0.4,
                skyColor: new THREE.Color(0x0a1a3a),
                groundColor: new THREE.Color(0x0a0a0a)
            },
            light: {
                background: new THREE.Color(0xd0e8ff),
                fog: new THREE.Color(0xd0e8ff),
                ambientLight: 0.8,
                skyColor: new THREE.Color(0x87ceeb),
                groundColor: new THREE.Color(0x8b7355)
            }
        };
    }
    
    applyColorScheme() {
        const scheme = this.isDark ? this.schemes.dark : this.schemes.light;
        
        // Background
        this.scene.background = scheme.background;
        if (this.scene.fog) this.scene.fog.color = scheme.fog;
        
        // Lights
        this.lights.ambient.intensity = scheme.ambientLight;
        
        // Optional: Update materials
        this.scene.traverse(obj => {
            if (obj.isMesh && obj.userData.adaptsToColorScheme) {
                const darkMat = obj.userData.darkMaterial;
                const lightMat = obj.userData.lightMaterial;
                if (darkMat && lightMat) {
                    obj.material = this.isDark ? darkMat : lightMat;
                }
            }
        });
        
        console.log(`Color scheme changed to: ${this.isDark ? 'dark' : 'light'}`);
    }
}
```

---

## 5. Progressive Loading

```javascript
class ProgressiveSceneLoader {
    constructor(scene, camera, renderer) {
        this.scene = scene;
        this.camera = camera;
        this.renderer = renderer;
        this.loadedStages = new Set();
    }
    
    // Stage 1: Show basic scene immediately (< 100ms)
    async loadBasicScene() {
        // สร้าง placeholder geometry ทันที
        const groundGeo = new THREE.PlaneGeometry(100, 100);
        const groundMat = new THREE.MeshBasicMaterial({ color: 0x1a2a1a });
        const ground = new THREE.Mesh(groundGeo, groundMat);
        ground.rotation.x = -Math.PI / 2;
        this.scene.add(ground);
        
        // Basic lighting
        const ambient = new THREE.AmbientLight(0xffffff, 0.5);
        this.scene.add(ambient);
        
        // Placeholder boxes แทน buildings จริง
        for (let i = 0; i < 20; i++) {
            const geo = new THREE.BoxGeometry(3, 8, 3);
            const mat = new THREE.MeshBasicMaterial({ color: 0x334455 });
            const mesh = new THREE.Mesh(geo, mat);
            mesh.position.set(
                (Math.random() - 0.5) * 80,
                4,
                (Math.random() - 0.5) * 80
            );
            mesh.userData.isPlaceholder = true;
            this.scene.add(mesh);
        }
        
        this.loadedStages.add('basic');
        this.onStageLoaded?.('basic', 'ฉากพื้นฐาน');
        
        // Render ทันที
        this.renderer.render(this.scene, this.camera);
    }
    
    // Stage 2: Load actual 3D models (async)
    async loadModels() {
        const loader = new THREE.GLTFLoader();
        
        // Load low-res models ก่อน
        const lowResURL = '/models/city-low.glb';
        
        try {
            const gltf = await loader.loadAsync(lowResURL, (progress) => {
                const pct = (progress.loaded / progress.total * 100).toFixed(0);
                this.onProgress?.('models-low', pct);
            });
            
            // Replace placeholders
            const placeholders = [];
            this.scene.traverse(obj => {
                if (obj.userData.isPlaceholder) placeholders.push(obj);
            });
            placeholders.forEach(p => this.scene.remove(p));
            
            this.scene.add(gltf.scene);
            this.loadedStages.add('models-low');
            this.onStageLoaded?.('models-low', 'โมเดลต่ำ');
        } catch (e) {
            console.warn('Low-res model failed:', e);
        }
        
        // Then load high-res
        if (this.shouldLoadHighRes()) {
            this.loadHighResModels();
        }
    }
    
    // Stage 3: Load high-res assets (background)
    async loadHighResModels() {
        // Progressive texture loading
        const textureLoader = new THREE.TextureLoader();
        
        const textures = [
            { url: '/textures/city-diffuse-4k.jpg', target: 'diffuse' },
            { url: '/textures/city-normal-4k.jpg', target: 'normal' },
        ];
        
        for (const { url, target } of textures) {
            try {
                const texture = await textureLoader.loadAsync(url);
                
                // Replace existing texture
                this.scene.traverse(obj => {
                    if (obj.isMesh && obj.material.userData?.[target]) {
                        const oldTex = obj.material[target];
                        obj.material[target] = texture;
                        if (oldTex) oldTex.dispose();
                        obj.material.needsUpdate = true;
                    }
                });
                
                this.onStageLoaded?.(target, `texture-${target}`);
            } catch (e) {
                console.warn(`Failed to load ${target} texture:`, e);
            }
        }
    }
    
    shouldLoadHighRes() {
        // เช็ค bandwidth
        const connection = navigator.connection;
        if (connection) {
            const slowConnections = ['slow-2g', '2g', '3g'];
            if (slowConnections.includes(connection.effectiveType)) {
                return false;
            }
        }
        
        // เช็ค GPU tier
        return true;
    }
}
```

---

## 6. Accessibility ใน 3D Scenes

```javascript
// ARIA Labels สำหรับ 3D content
function setup3DAccessibility(container, scene) {
    // Container role
    container.setAttribute('role', 'application');
    container.setAttribute('aria-label', '3D Interactive Scene - ฉาก 3D แบบ Interactive');
    container.setAttribute('aria-describedby', 'scene-description');
    
    // Live region สำหรับ screen readers
    const liveRegion = document.createElement('div');
    liveRegion.id = 'scene-live';
    liveRegion.setAttribute('aria-live', 'polite');
    liveRegion.setAttribute('aria-atomic', 'true');
    liveRegion.style.cssText = 'position:absolute;width:1px;height:1px;overflow:hidden;clip:rect(0,0,0,0)';
    document.body.appendChild(liveRegion);
    
    // Static description
    const description = document.createElement('div');
    description.id = 'scene-description';
    description.style.cssText = 'position:absolute;width:1px;height:1px;overflow:hidden;clip:rect(0,0,0,0)';
    description.textContent = '3D scene แสดงภูมิทัศน์เมือง ประกอบด้วยอาคาร ถนน และสวนสาธารณะ สามารถหมุนโดยใช้เมาส์หรือ touch ซูมด้วย scroll หรือ pinch';
    document.body.appendChild(description);
    
    // Announce scene changes
    function announceChange(message) {
        liveRegion.textContent = message;
    }
    
    // Keyboard navigation
    function handleKeyboard(event) {
        const STEP = 5;
        const orbitStep = 0.1;
        
        switch (event.key) {
            case 'ArrowLeft':
                announceChange('หมุนฉากไปทางซ้าย');
                break;
            case 'ArrowRight':
                announceChange('หมุนฉากไปทางขวา');
                break;
            case 'ArrowUp':
                announceChange('ขยับกล้องขึ้น');
                break;
            case 'ArrowDown':
                announceChange('ขยับกล้องลง');
                break;
            case '+':
            case '=':
                announceChange('ซูมเข้า');
                break;
            case '-':
                announceChange('ซูมออก');
                break;
            case 'r':
            case 'R':
                announceChange('รีเซ็ตกล้องกลับตำแหน่งเริ่มต้น');
                break;
            case 'Tab':
                // Navigate to next interactive element
                focusNextElement();
                break;
        }
    }
    
    container.addEventListener('keydown', handleKeyboard);
    container.setAttribute('tabindex', '0'); // Make focusable
    
    return { announceChange };
}

// Interactive elements ใน 3D scene
class AccessibleObject3D {
    constructor(mesh, options) {
        this.mesh = mesh;
        this.label = options.label || 'Object';
        this.description = options.description || '';
        this.interactive = options.interactive || false;
        
        // Create screen-reader accessible element
        this.srElement = document.createElement('button');
        this.srElement.style.cssText = 'position:absolute;width:1px;height:1px;overflow:hidden;clip:rect(0,0,0,0)';
        this.srElement.textContent = this.label;
        this.srElement.setAttribute('aria-label', `${this.label}: ${this.description}`);
        
        if (this.interactive) {
            this.srElement.addEventListener('click', () => {
                this.mesh.dispatchEvent({ type: 'select' });
            });
        }
        
        document.body.appendChild(this.srElement);
    }
    
    // อัปเดตตำแหน่งของ SR element ให้ตรงกับ 3D position
    updateScreenPosition(camera, renderer) {
        const position = new THREE.Vector3();
        this.mesh.getWorldPosition(position);
        position.project(camera);
        
        const x = (position.x * 0.5 + 0.5) * renderer.domElement.clientWidth;
        const y = (-position.y * 0.5 + 0.5) * renderer.domElement.clientHeight;
        
        this.srElement.style.left = `${x}px`;
        this.srElement.style.top = `${y}px`;
    }
    
    destroy() {
        this.srElement.remove();
    }
}
```

---

## 7. Complete Feature Detection Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Progressive Enhancement Demo</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        
        /* CSS Custom Properties for color scheme */
        :root {
            --bg: #050510;
            --surface: rgba(0,0,20,0.9);
            --text: #eee;
            --text-muted: #888;
            --accent: #0af;
            --success: #4f4;
            --warn: #fa0;
            --danger: #f44;
        }
        
        @media (prefers-color-scheme: light) {
            :root {
                --bg: #f0f4ff;
                --surface: rgba(240,244,255,0.95);
                --text: #111;
                --text-muted: #666;
                --accent: #0077cc;
                --success: #080;
                --warn: #a60;
                --danger: #c00;
            }
        }
        
        body {
            background: var(--bg);
            color: var(--text);
            font-family: system-ui, -apple-system, sans-serif;
            overflow: hidden;
        }
        
        /* Reduced motion */
        @media (prefers-reduced-motion: reduce) {
            *, *::before, *::after {
                animation-duration: 0.001ms !important;
                transition-duration: 0.001ms !important;
            }
        }
        
        canvas { display: block; width: 100vw; height: 100vh; }
        
        #feature-panel {
            position: fixed; top: 20px; right: 20px;
            background: var(--surface);
            padding: 20px; border-radius: 12px;
            border: 1px solid rgba(128,128,255,0.2);
            backdrop-filter: blur(12px);
            -webkit-backdrop-filter: blur(12px);
            width: 300px; z-index: 10;
        }
        
        h2 { color: var(--accent); margin-bottom: 12px; font-size: 14px; }
        
        .feature-item {
            display: flex; justify-content: space-between;
            padding: 5px 0; border-bottom: 1px solid rgba(128,128,255,0.1);
            font-size: 12px;
        }
        
        .feature-name { color: var(--text-muted); }
        .support-yes { color: var(--success); }
        .support-no { color: var(--danger); }
        .support-partial { color: var(--warn); }
        
        .section-title {
            color: var(--warn);
            font-size: 11px;
            margin: 12px 0 4px;
            text-transform: uppercase;
            font-weight: 700;
        }
        
        #progress-overlay {
            position: fixed; inset: 0;
            background: var(--bg);
            display: flex; flex-direction: column;
            align-items: center; justify-content: center;
            z-index: 100; transition: opacity 0.5s;
        }
        
        .progress-bar {
            width: 300px; height: 4px;
            background: rgba(128,128,255,0.2);
            border-radius: 2px; margin: 10px 0;
        }
        
        .progress-fill {
            height: 100%; border-radius: 2px;
            background: var(--accent);
            transition: width 0.3s;
        }
        
        #stage-text { color: var(--text-muted); font-size: 12px; }
        
        /* No-JS fallback */
        .no-js-fallback { display: none; }
    </style>
    
    <!-- No-JS fallback -->
    <noscript>
        <style>.no-js-fallback { display: block; }</style>
    </noscript>
</head>
<body>
    <!-- No-JS fallback -->
    <div class="no-js-fallback" style="
        display: none; position: fixed; inset: 0;
        background: #050510; color: #eee;
        display: flex; align-items: center; justify-content: center;
        flex-direction: column; gap: 20px; text-align: center; padding: 40px;
    ">
        <img src="fallback.jpg" alt="3D Scene" style="max-width:80%;border-radius:8px" 
             onerror="this.style.display='none'">
        <h2 style="color:#f44">JavaScript ปิดอยู่</h2>
        <p style="color:#888">กรุณาเปิด JavaScript เพื่อดู 3D Scene แบบ Interactive</p>
    </div>
    
    <!-- Progress overlay -->
    <div id="progress-overlay">
        <div style="color: var(--accent); font-size: 18px; font-weight: 700; margin-bottom: 8px">
            🔍 ตรวจสอบระบบ...
        </div>
        <div class="progress-bar"><div id="prog-fill" class="progress-fill" style="width: 0%"></div></div>
        <div id="stage-text">กำลังตรวจสอบ WebGL support...</div>
    </div>
    
    <canvas id="c" role="application" aria-label="3D Scene" tabindex="0"></canvas>
    
    <div id="feature-panel" style="display:none">
        <h2>🔬 Feature Detection</h2>
        
        <div class="section-title">WebGL</div>
        <div class="feature-item"><span class="feature-name">WebGL Version</span><span id="f-webgl">-</span></div>
        <div class="feature-item"><span class="feature-name">Float Textures</span><span id="f-float">-</span></div>
        <div class="feature-item"><span class="feature-name">Instancing</span><span id="f-inst">-</span></div>
        <div class="feature-item"><span class="feature-name">Max Texture Size</span><span id="f-texsize">-</span></div>
        <div class="feature-item"><span class="feature-name">Max Anisotropy</span><span id="f-aniso">-</span></div>
        
        <div class="section-title">Browser</div>
        <div class="feature-item"><span class="feature-name">WebXR (VR/AR)</span><span id="f-xr">-</span></div>
        <div class="feature-item"><span class="feature-name">OffscreenCanvas</span><span id="f-offscreen">-</span></div>
        <div class="feature-item"><span class="feature-name">Shared Array Buffer</span><span id="f-sab">-</span></div>
        <div class="feature-item"><span class="feature-name">Battery API</span><span id="f-battery">-</span></div>
        
        <div class="section-title">User Preferences</div>
        <div class="feature-item"><span class="feature-name">Color Scheme</span><span id="f-scheme">-</span></div>
        <div class="feature-item"><span class="feature-name">Reduced Motion</span><span id="f-motion">-</span></div>
        <div class="feature-item"><span class="feature-name">High Contrast</span><span id="f-contrast">-</span></div>
        
        <div class="section-title">Performance</div>
        <div class="feature-item"><span class="feature-name">Hardware Concurrency</span><span id="f-cores">-</span></div>
        <div class="feature-item"><span class="feature-name">Device Memory</span><span id="f-memory">-</span></div>
        <div class="feature-item"><span class="feature-name">Network</span><span id="f-network">-</span></div>
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
    
    const overlay = document.getElementById('progress-overlay');
    const progFill = document.getElementById('prog-fill');
    const stageText = document.getElementById('stage-text');
    
    function setProgress(pct, msg) {
        progFill.style.width = pct + '%';
        stageText.textContent = msg;
    }
    
    function featureSpan(supported, trueText = 'รองรับ ✅', falseText = 'ไม่รองรับ ❌', partialText = null) {
        const cls = supported === true ? 'support-yes' : 
                    supported === false ? 'support-no' : 'support-partial';
        const text = supported === true ? trueText : 
                     supported === false ? falseText : (partialText || trueText);
        return `<span class="${cls}">${text}</span>`;
    }
    
    // ============== Feature Detection ==============
    setProgress(10, 'ตรวจสอบ WebGL support...');
    
    const canvas = document.getElementById('c');
    const testCanvas = document.createElement('canvas');
    const gl2 = testCanvas.getContext('webgl2');
    const gl1 = !gl2 && (testCanvas.getContext('webgl') || testCanvas.getContext('experimental-webgl'));
    const webglVersion = gl2 ? 2 : (gl1 ? 1 : 0);
    
    if (webglVersion === 0) {
        overlay.innerHTML = `
            <div style="text-align:center;padding:40px;color:#eee">
                <div style="font-size:40px;margin-bottom:20px">❌</div>
                <h2 style="color:#f44">WebGL ไม่รองรับ</h2>
                <p style="color:#888;margin-top:10px">Browser ของคุณไม่รองรับ WebGL</p>
            </div>`;
        return;
    }
    
    const gl = gl2 || gl1;
    const extAniso = gl.getExtension('EXT_texture_filter_anisotropic') ||
                     gl.getExtension('WEBKIT_EXT_texture_filter_anisotropic');
    const maxAniso = extAniso ? gl.getParameter(extAniso.MAX_TEXTURE_MAX_ANISOTROPY_EXT) : 1;
    const maxTexSize = gl.getParameter(gl.MAX_TEXTURE_SIZE);
    const hasFloat = !!(webglVersion === 2 ? gl.getExtension('EXT_color_buffer_float') : gl.getExtension('OES_texture_float'));
    const hasInstancing = webglVersion === 2 || !!gl.getExtension('ANGLE_instanced_arrays');
    
    setProgress(30, 'ตรวจสอบ browser features...');
    
    const hasOffscreen = typeof OffscreenCanvas !== 'undefined';
    const hasSAB = typeof SharedArrayBuffer !== 'undefined';
    const hasBattery = 'getBattery' in navigator;
    
    const prefersColorScheme = window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light';
    const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
    const prefersHighContrast = window.matchMedia('(prefers-contrast: high)').matches;
    
    const cores = navigator.hardwareConcurrency || 'unknown';
    const memory = navigator.deviceMemory ? `${navigator.deviceMemory} GB` : 'unknown';
    const network = navigator.connection?.effectiveType || 'unknown';
    
    setProgress(50, 'ตรวจสอบ WebXR...');
    
    let hasVR = false, hasAR = false;
    if (navigator.xr) {
        hasVR = await navigator.xr.isSessionSupported('immersive-vr').catch(() => false);
        hasAR = await navigator.xr.isSessionSupported('immersive-ar').catch(() => false);
    }
    
    setProgress(70, 'สร้าง Three.js renderer...');
    
    // Create renderer with detected capabilities
    const renderer = new THREE.WebGLRenderer({
        canvas,
        antialias: webglVersion === 2 && maxTexSize >= 4096,
        powerPreference: 'high-performance'
    });
    renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
    renderer.shadowMap.enabled = webglVersion >= 2;
    renderer.toneMapping = webglVersion >= 2 ? THREE.ACESFilmicToneMapping : THREE.NoToneMapping;
    
    const scene = new THREE.Scene();
    
    // Color scheme
    const bgColor = prefersColorScheme === 'dark' ? 0x050510 : 0xc8dcff;
    scene.background = new THREE.Color(bgColor);
    scene.fog = new THREE.FogExp2(bgColor, 0.01);
    
    const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 300);
    camera.position.set(0, 10, 25);
    
    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.autoRotate = !prefersReducedMotion; // Respect reduced motion
    controls.autoRotateSpeed = prefersReducedMotion ? 0 : 0.5;
    
    // Lights
    const ambientIntensity = prefersColorScheme === 'dark' ? 0.5 : 0.8;
    scene.add(new THREE.AmbientLight(0xffffff, ambientIntensity));
    const sun = new THREE.DirectionalLight(0xffeedd, 1.5);
    sun.position.set(30, 50, 20);
    if (webglVersion >= 2) {
        sun.castShadow = true;
        sun.shadow.mapSize.set(1024, 1024);
    }
    scene.add(sun);
    
    // Scene content
    const groundMat = new THREE.MeshStandardMaterial({ 
        color: prefersColorScheme === 'dark' ? 0x1a2a1a : 0x4a7a4a 
    });
    const ground = new THREE.Mesh(new THREE.PlaneGeometry(200, 200), groundMat);
    ground.rotation.x = -Math.PI / 2;
    if (webglVersion >= 2) ground.receiveShadow = true;
    scene.add(ground);
    
    // Buildings
    const buildingColors = prefersHighContrast 
        ? [0xff0000, 0x0000ff, 0x00ff00]  // High contrast
        : [0x334455, 0x445566, 0x556677];  // Normal
    
    for (let i = 0; i < 50; i++) {
        const h = 4 + Math.random() * 12;
        const geo = new THREE.BoxGeometry(3, h, 3);
        const mat = new THREE.MeshStandardMaterial({
            color: buildingColors[i % buildingColors.length]
        });
        const mesh = new THREE.Mesh(geo, mat);
        mesh.position.set((Math.random()-0.5)*80, h/2, (Math.random()-0.5)*80);
        if (webglVersion >= 2) mesh.castShadow = true;
        scene.add(mesh);
    }
    
    setProgress(90, 'เตรียมการแสดงผล...');
    
    // Update feature panel
    document.getElementById('f-webgl').innerHTML = featureSpan(true, `WebGL ${webglVersion} ✅`);
    document.getElementById('f-float').innerHTML = featureSpan(hasFloat);
    document.getElementById('f-inst').innerHTML = featureSpan(hasInstancing);
    document.getElementById('f-texsize').innerHTML = `<span class="support-yes">${maxTexSize}px</span>`;
    document.getElementById('f-aniso').innerHTML = `<span class="${maxAniso > 1 ? 'support-yes' : 'support-no'}">x${maxAniso}</span>`;
    document.getElementById('f-xr').innerHTML = featureSpan(hasVR || hasAR, 
        `${hasVR ? 'VR' : ''}${hasVR && hasAR ? '/' : ''}${hasAR ? 'AR' : ''} ✅`, 'ไม่รองรับ');
    document.getElementById('f-offscreen').innerHTML = featureSpan(hasOffscreen);
    document.getElementById('f-sab').innerHTML = featureSpan(hasSAB);
    document.getElementById('f-battery').innerHTML = featureSpan(hasBattery);
    document.getElementById('f-scheme').innerHTML = `<span class="support-yes">${prefersColorScheme}</span>`;
    document.getElementById('f-motion').innerHTML = featureSpan(!prefersReducedMotion, 'เปิด ✅', 'reduced ⚠️');
    document.getElementById('f-contrast').innerHTML = featureSpan(prefersHighContrast, 'high ⚠️', 'normal ✅');
    document.getElementById('f-cores').innerHTML = `<span class="support-yes">${cores} cores</span>`;
    document.getElementById('f-memory').innerHTML = `<span class="support-yes">${memory}</span>`;
    document.getElementById('f-network').innerHTML = `<span class="${network === '4g' ? 'support-yes' : network === '3g' ? 'support-partial' : 'support-no'}">${network}</span>`;
    
    // Show panel
    document.getElementById('feature-panel').style.display = 'block';
    
    setProgress(100, 'เสร็จสิ้น!');
    
    // Hide overlay
    setTimeout(() => {
        overlay.style.opacity = '0';
        overlay.style.pointerEvents = 'none';
        setTimeout(() => overlay.style.display = 'none', prefersReducedMotion ? 0 : 500);
    }, prefersReducedMotion ? 0 : 300);
    
    // Listen for color scheme changes
    window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', (e) => {
        const newBg = e.matches ? 0x050510 : 0xc8dcff;
        scene.background = new THREE.Color(newBg);
        scene.fog = new THREE.FogExp2(newBg, 0.01);
    });
    
    // Listen for reduced motion changes
    window.matchMedia('(prefers-reduced-motion: reduce)').addEventListener('change', (e) => {
        controls.autoRotate = !e.matches;
    });
    
    function resize() {
        renderer.setSize(innerWidth, innerHeight);
        camera.aspect = innerWidth / innerHeight;
        camera.updateProjectionMatrix();
    }
    window.addEventListener('resize', resize);
    resize();
    
    let lastTime = performance.now(), frames = 0;
    
    function animate() {
        requestAnimationFrame(animate);
        controls.update();
        renderer.render(scene, camera);
    }
    
    animate();
</script>
</body>
</html>
```

---

## 8. Performance Budget

```javascript
// Performance Budget: กำหนด limits สำหรับแต่ละ tier
const performanceBudget = {
    high: {
        drawCalls: 200,
        triangles: 500_000,
        textureMB: 256,
        shaderComplexity: 'high',
        targetFPS: 60,
        maxFrameTime: 16.7
    },
    medium: {
        drawCalls: 100,
        triangles: 200_000,
        textureMB: 128,
        shaderComplexity: 'medium',
        targetFPS: 60,
        maxFrameTime: 16.7
    },
    low: {
        drawCalls: 50,
        triangles: 50_000,
        textureMB: 64,
        shaderComplexity: 'low',
        targetFPS: 30,
        maxFrameTime: 33.3
    }
};

function checkPerformanceBudget(renderer, tier) {
    const budget = performanceBudget[tier];
    const info = renderer.info.render;
    const violations = [];
    
    if (info.calls > budget.drawCalls) {
        violations.push(`Draw calls: ${info.calls} > ${budget.drawCalls}`);
    }
    if (info.triangles > budget.triangles) {
        violations.push(`Triangles: ${info.triangles.toLocaleString()} > ${budget.triangles.toLocaleString()}`);
    }
    
    return { budget, violations, passing: violations.length === 0 };
}
```

---

## แบบฝึกหัด

1. **ง่าย**: สร้าง `detectWebGLSupport()` function ที่ return version, max texture size, และ anisotropy support
2. **กลาง**: Implement `prefers-color-scheme` support ใน Three.js scene ที่เปลี่ยน background, fog, และ light colors
3. **ยาก**: สร้าง `ProgressiveSceneLoader` ที่ load scene เป็น 3 stages: placeholder → low-res models → high-res textures
4. **Challenge**: สร้าง fully accessible 3D scene ที่มี ARIA labels, keyboard navigation, screen reader announcements, และ respects reduced motion

---

## อ้างอิง

- [WebGL Best Practices](https://developer.mozilla.org/en-US/docs/Web/API/WebGL_API/WebGL_best_practices)
- [prefers-reduced-motion](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion)
- [prefers-color-scheme](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-color-scheme)
- [Web Accessibility for 3D](https://www.w3.org/TR/wai-aria-1.2/)
- [Progressive Enhancement](https://developer.mozilla.org/en-US/docs/Glossary/Progressive_Enhancement)
