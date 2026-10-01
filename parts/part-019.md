# Part 19: Post-Processing Effects ใน Three.js

## บทนำ

Post-Processing คือ effects ที่ทำหลังจาก render scene แล้ว เหมือนการ filter ภาพ Three.js ใช้ EffectComposer เป็น pipeline สำหรับใส่ effects ต่อๆ กัน

## 1. EffectComposer Setup

```bash
# ติดตั้ง postprocessing (ถ้าใช้ package version เก่า)
npm install three
# Three.js มี post-processing ใน examples/jsm/postprocessing/
```

```javascript
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';

// ตั้งค่า Composer
const composer = new EffectComposer(renderer);

// RenderPass ต้องเป็น pass แรกเสมอ
const renderPass = new RenderPass(scene, camera);
composer.addPass(renderPass);

// แทน renderer.render ด้วย composer.render
function animate() {
    requestAnimationFrame(animate);
    
    // ❌ แทนที่แบบเก่า:
    // renderer.render(scene, camera);
    
    // ✅ ใช้ composer:
    composer.render();
}
```

---

## 2. UnrealBloomPass

Bloom ทำให้แสงสว่างดูเรืองแสงออกมา เหมือนผ่านเลนส์กล้อง

```javascript
import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass.js';

const bloomPass = new UnrealBloomPass(
    new THREE.Vector2(window.innerWidth, window.innerHeight),
    strength,    // ความเข้มของ bloom (0-3)
    radius,      // รัศมี (0-1)
    threshold    // threshold (0-1, สว่างกว่านี้ถึงจะ glow)
);

bloomPass.strength = 1.5;   // เพิ่ม/ลด bloom intensity
bloomPass.radius = 0.4;     // ขนาด bloom spread
bloomPass.threshold = 0.85; // ต้องสว่างกว่า 85% ถึงจะ bloom

composer.addPass(bloomPass);

// อัพเดต resolution เมื่อ resize
window.addEventListener('resize', () => {
    bloomPass.setSize(window.innerWidth, window.innerHeight);
});
```

### Selective Bloom (Bloom เฉพาะบาง Object)

```javascript
// วิธีทำ selective bloom:
// 1. Render ทั้ง scene แบบปกติ
// 2. Render เฉพาะ glowing objects
// 3. Blend สองภาพ

const BLOOM_LAYER = 1;
const bloomLayer = new THREE.Layers();
bloomLayer.set(BLOOM_LAYER);

// Objects ที่ glow ใส่ layer 1
glowingObject.layers.enable(BLOOM_LAYER);

// Bloom pass เฉพาะ layer 1
const bloomComposer = new EffectComposer(renderer);
bloomComposer.renderToScreen = false;
bloomComposer.addPass(new RenderPass(scene, camera));

// Dark materials สำหรับ non-bloom pass
const darkMaterial = new THREE.MeshBasicMaterial({ color: 0x000000 });
const materials = {};

function darkenNonBloomed(obj) {
    if (obj.isMesh && bloomLayer.test(obj.layers) === false) {
        materials[obj.uuid] = obj.material;
        obj.material = darkMaterial;
    }
}

function restoreMaterial(obj) {
    if (materials[obj.uuid]) {
        obj.material = materials[obj.uuid];
        delete materials[obj.uuid];
    }
}

const bloomPass2 = new UnrealBloomPass(
    new THREE.Vector2(window.innerWidth, window.innerHeight),
    1.5, 0.4, 0.85
);
bloomComposer.addPass(bloomPass2);

// Final composer
const finalComposer = new EffectComposer(renderer);
finalComposer.addPass(new RenderPass(scene, camera));

// Custom blend shader
const finalPass = new ShaderPass(
    new THREE.ShaderMaterial({
        uniforms: {
            baseTexture: { value: null },
            bloomTexture: { value: bloomComposer.renderTarget2.texture }
        },
        vertexShader: `
            varying vec2 vUv;
            void main() { vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }
        `,
        fragmentShader: `
            uniform sampler2D baseTexture;
            uniform sampler2D bloomTexture;
            varying vec2 vUv;
            void main() {
                gl_FragColor = texture2D(baseTexture, vUv) + texture2D(bloomTexture, vUv);
            }
        `,
        defines: {}
    }), 'baseTexture'
);
finalPass.needsSwap = true;
finalComposer.addPass(finalPass);

// Render loop
function render() {
    scene.traverse(darkenNonBloomed);
    bloomComposer.render();
    scene.traverse(restoreMaterial);
    finalComposer.render();
}
```

---

## 3. SSAOPass - Ambient Occlusion

SSAO ทำให้บริเวณ corners และ crevices ดูมืดกว่า เพิ่ม depth และ realism

```javascript
import { SSAOPass } from 'three/examples/jsm/postprocessing/SSAOPass.js';

const ssaoPass = new SSAOPass(
    scene,
    camera,
    window.innerWidth,
    window.innerHeight
);

ssaoPass.kernelRadius = 8;     // ขนาด sampling radius
ssaoPass.minDistance = 0.005;  // minimum distance
ssaoPass.maxDistance = 0.1;    // maximum distance

// ปรับได้
ssaoPass.output = SSAOPass.OUTPUT.Default;    // ปกติ
ssaoPass.output = SSAOPass.OUTPUT.SSAO;       // แสดงเฉพาะ AO map
ssaoPass.output = SSAOPass.OUTPUT.Blur;       // แสดง blur
ssaoPass.output = SSAOPass.OUTPUT.Beauty;     // แสดง beauty pass

composer.addPass(ssaoPass);
```

---

## 4. SMAAPass - Anti-aliasing

SMAA (Subpixel Morphological Anti-aliasing) ลด jagged edges

```javascript
import { SMAAPass } from 'three/examples/jsm/postprocessing/SMAAPass.js';

const smaaPass = new SMAAPass(window.innerWidth, window.innerHeight);
composer.addPass(smaaPass);

// เปรียบเทียบ AA methods:
// renderer.antialias = true  - MSAA (ใน renderer constructor)
// SMAAPass                   - SMAA (post-processing)
// TAARenderPass              - TAA (temporal AA)
// FXAAShader                 - FXAA (ShaderPass)
```

---

## 5. FilmPass, GlitchPass, DotScreenPass

```javascript
import { FilmPass } from 'three/examples/jsm/postprocessing/FilmPass.js';
import { GlitchPass } from 'three/examples/jsm/postprocessing/GlitchPass.js';
import { DotScreenPass } from 'three/examples/jsm/postprocessing/DotScreenPass.js';

// Film grain + scanlines
const filmPass = new FilmPass(
    0.35,    // noise intensity (0-1)
    0.025,   // scanline intensity (0-1)
    648,     // scanline count
    false    // grayscale?
);
composer.addPass(filmPass);

// Glitch effect
const glitchPass = new GlitchPass();
glitchPass.goWild = false; // wild mode = glitch ตลอดเวลา
composer.addPass(glitchPass);
// กระตุ้น glitch ชั่วคราว
setTimeout(() => { glitchPass.goWild = true; }, 2000);
setTimeout(() => { glitchPass.goWild = false; }, 2500);

// Dot screen (pop art effect)
const dotScreenPass = new DotScreenPass(
    new THREE.Vector2(0, 0),  // center
    0.5,    // angle (radians)
    0.8     // scale
);
composer.addPass(dotScreenPass);
```

---

## 6. HalftonePass

```javascript
import { HalftonePass } from 'three/examples/jsm/postprocessing/HalftonePass.js';

const halftonePass = new HalftonePass(
    window.innerWidth,
    window.innerHeight,
    {
        shape: 1,           // 1=dot, 2=ellipse, 3=line, 4=square
        radius: 4,          // dot radius
        rotateR: Math.PI / 12,
        rotateG: Math.PI / 12 * 2,
        rotateB: Math.PI / 12 * 3,
        scatter: 0,
        blending: 1,
        blendingMode: 1,
        greyscale: false,
        disable: false
    }
);
composer.addPass(halftonePass);
```

---

## 7. BokehPass - Depth of Field

```javascript
import { BokehPass } from 'three/examples/jsm/postprocessing/BokehPass.js';

const bokehPass = new BokehPass(scene, camera, {
    focus: 5.0,      // focal distance
    aperture: 0.025, // aperture (ขนาด blur)
    maxblur: 0.01    // max blur amount
});
composer.addPass(bokehPass);

// อัพเดต focus ระหว่าง runtime
bokehPass.uniforms['focus'].value = focalDistance;
bokehPass.uniforms['aperture'].value = aperture;

// ให้ focus ตาม mouse (depth picking)
raycaster.setFromCamera(mouse, camera);
const intersects = raycaster.intersectObjects(focusableObjects);
if (intersects.length > 0) {
    const dist = intersects[0].distance;
    bokehPass.uniforms['focus'].value = dist;
}
```

---

## 8. OutlinePass

```javascript
import { OutlinePass } from 'three/examples/jsm/postprocessing/OutlinePass.js';

const outlinePass = new OutlinePass(
    new THREE.Vector2(window.innerWidth, window.innerHeight),
    scene,
    camera
);

outlinePass.edgeStrength = 3.0;        // ความเข้ม outline
outlinePass.edgeGlow = 0.0;            // glow ของขอบ
outlinePass.edgeThickness = 1.0;       // ความหนา
outlinePass.pulsePeriod = 0;           // ความเร็ว pulse (0 = ไม่ pulse)
outlinePass.visibleEdgeColor.set('#ff0000');    // สีขอบ visible
outlinePass.hiddenEdgeColor.set('#190a05');     // สีขอบ hidden

composer.addPass(outlinePass);

// กำหนดวัตถุที่ outline
outlinePass.selectedObjects = [mesh1, mesh2];

// อัพเดตเมื่อ select
function selectObject(obj) {
    outlinePass.selectedObjects = obj ? [obj] : [];
}
```

---

## 9. ShaderPass - Custom Effects

```javascript
import { ShaderPass } from 'three/examples/jsm/postprocessing/ShaderPass.js';

// Custom shader effect
const customShader = {
    uniforms: {
        tDiffuse: { value: null },     // input texture (อัตโนมัติ)
        time: { value: 0 },
        intensity: { value: 1.0 }
    },
    vertexShader: `
        varying vec2 vUv;
        void main() {
            vUv = uv;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
    `,
    fragmentShader: `
        uniform sampler2D tDiffuse;
        uniform float time;
        uniform float intensity;
        varying vec2 vUv;
        
        void main() {
            vec2 uv = vUv;
            
            // Chromatic aberration
            float amount = 0.003 * intensity;
            vec4 r = texture2D(tDiffuse, vec2(uv.x + amount, uv.y));
            vec4 g = texture2D(tDiffuse, uv);
            vec4 b = texture2D(tDiffuse, vec2(uv.x - amount, uv.y));
            
            gl_FragColor = vec4(r.r, g.g, b.b, 1.0);
        }
    `
};

const customPass = new ShaderPass(customShader);
customPass.uniforms['intensity'].value = 2.0;
composer.addPass(customPass);

// Update uniform ใน loop
customPass.uniforms['time'].value = clock.getElapsedTime();
```

### Vignette Effect

```javascript
import { ShaderPass } from 'three/examples/jsm/postprocessing/ShaderPass.js';
import { VignetteShader } from 'three/examples/jsm/shaders/VignetteShader.js';

const vignettePass = new ShaderPass(VignetteShader);
vignettePass.uniforms['offset'].value = 0.95;  // ขอบ
vignettePass.uniforms['darkness'].value = 1.6; // ความมืด
composer.addPass(vignettePass);
```

---

## 10. Demo สมบูรณ์: Post-Processing Showcase

```html
<!DOCTYPE html>
<html>
<head>
    <title>Post-Processing Demo</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; font-family: monospace; }
        
        #effects-panel {
            position: absolute;
            top: 10px;
            right: 10px;
            background: rgba(0,0,0,0.85);
            color: white;
            padding: 15px;
            border-radius: 8px;
            width: 220px;
            font-size: 12px;
        }
        
        h3 { color: #e94560; margin-bottom: 10px; }
        h4 { color: #4ecdc4; margin: 8px 0 4px; font-size: 11px; }
        
        .effect-row {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin: 3px 0;
            padding: 3px 0;
            border-bottom: 1px solid #222;
        }
        .effect-row input[type=checkbox] { cursor: pointer; }
        .effect-row input[type=range] { width: 80px; }
        .effect-label { color: #aaa; }
    </style>
</head>
<body>
<div id="effects-panel">
    <h3>Post-Processing</h3>
    
    <h4>Bloom</h4>
    <div class="effect-row">
        <span class="effect-label">Enable</span>
        <input type="checkbox" id="bloomEnabled" checked>
    </div>
    <div class="effect-row">
        <span class="effect-label">Strength</span>
        <input type="range" id="bloomStrength" min="0" max="3" step="0.05" value="1.5">
    </div>
    <div class="effect-row">
        <span class="effect-label">Radius</span>
        <input type="range" id="bloomRadius" min="0" max="1" step="0.01" value="0.4">
    </div>
    <div class="effect-row">
        <span class="effect-label">Threshold</span>
        <input type="range" id="bloomThreshold" min="0" max="1" step="0.01" value="0.85">
    </div>
    
    <h4>Film</h4>
    <div class="effect-row">
        <span class="effect-label">Enable</span>
        <input type="checkbox" id="filmEnabled">
    </div>
    
    <h4>Glitch</h4>
    <div class="effect-row">
        <span class="effect-label">Enable</span>
        <input type="checkbox" id="glitchEnabled">
    </div>
    <div class="effect-row">
        <span class="effect-label">Wild</span>
        <input type="checkbox" id="glitchWild">
    </div>
    
    <h4>Dot Screen</h4>
    <div class="effect-row">
        <span class="effect-label">Enable</span>
        <input type="checkbox" id="dotEnabled">
    </div>
    
    <h4>SMAA</h4>
    <div class="effect-row">
        <span class="effect-label">Enable</span>
        <input type="checkbox" id="smaaEnabled" checked>
    </div>
    
    <h4>Outline (hover)</h4>
    <div class="effect-row">
        <span class="effect-label">Enable</span>
        <input type="checkbox" id="outlineEnabled" checked>
    </div>
</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';
import { EffectComposer } from 'https://unpkg.com/three@0.158.0/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'https://unpkg.com/three@0.158.0/examples/jsm/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'https://unpkg.com/three@0.158.0/examples/jsm/postprocessing/UnrealBloomPass.js';
import { FilmPass } from 'https://unpkg.com/three@0.158.0/examples/jsm/postprocessing/FilmPass.js';
import { GlitchPass } from 'https://unpkg.com/three@0.158.0/examples/jsm/postprocessing/GlitchPass.js';
import { DotScreenPass } from 'https://unpkg.com/three@0.158.0/examples/jsm/postprocessing/DotScreenPass.js';
import { SMAAPass } from 'https://unpkg.com/three@0.158.0/examples/jsm/postprocessing/SMAAPass.js';
import { OutlinePass } from 'https://unpkg.com/three@0.158.0/examples/jsm/postprocessing/OutlinePass.js';

// Scene
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x000011);

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 5, 12);

const renderer = new THREE.WebGLRenderer({ antialias: false }); // ปิด native AA (ใช้ SMAA แทน)
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

// Lighting
scene.add(new THREE.AmbientLight(0x111133, 0.5));

const pointLight1 = new THREE.PointLight(0xff4444, 2, 30);
pointLight1.position.set(5, 5, 5);
scene.add(pointLight1);

const pointLight2 = new THREE.PointLight(0x4444ff, 2, 30);
pointLight2.position.set(-5, 5, -5);
scene.add(pointLight2);

const pointLight3 = new THREE.PointLight(0x44ff44, 1.5, 20);
pointLight3.position.set(0, 8, 0);
scene.add(pointLight3);

// สร้างวัตถุ emissive (จะ glow)
const emissiveObjects = [];
const normalObjects = [];

const colors = [0xff0044, 0x00ffaa, 0xffaa00, 0x0088ff, 0xff00ff];
colors.forEach((color, i) => {
    const angle = (i / colors.length) * Math.PI * 2;
    const geo = new THREE.IcosahedronGeometry(0.8, 0);
    const mat = new THREE.MeshStandardMaterial({
        color: color,
        emissive: color,
        emissiveIntensity: 1.5,
        roughness: 0.1,
        metalness: 0.8
    });
    const mesh = new THREE.Mesh(geo, mat);
    mesh.position.set(Math.cos(angle) * 5, 0, Math.sin(angle) * 5);
    scene.add(mesh);
    emissiveObjects.push(mesh);
});

// วัตถุทั่วไป (ไม่ glow)
for (let i = 0; i < 20; i++) {
    const geo = new THREE.BoxGeometry(0.5, 0.5, 0.5);
    const mat = new THREE.MeshStandardMaterial({
        color: 0x444466,
        roughness: 0.7,
        metalness: 0.3
    });
    const mesh = new THREE.Mesh(geo, mat);
    mesh.position.set(
        (Math.random() - 0.5) * 15,
        (Math.random() - 0.5) * 5,
        (Math.random() - 0.5) * 15
    );
    mesh.rotation.set(Math.random(), Math.random(), Math.random());
    scene.add(mesh);
    normalObjects.push(mesh);
}

// Central glowing sphere
const centerGeo = new THREE.SphereGeometry(1.5, 64, 64);
const centerMat = new THREE.MeshStandardMaterial({
    color: 0xffffff,
    emissive: 0xffffff,
    emissiveIntensity: 2,
    roughness: 0.1,
    metalness: 0.5
});
const centerSphere = new THREE.Mesh(centerGeo, centerMat);
scene.add(centerSphere);
emissiveObjects.push(centerSphere);

// Grid
scene.add(new THREE.GridHelper(30, 30, 0x112233, 0x0a1a2a));

// === Post-Processing Setup ===
const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));

// Bloom
const bloomPass = new UnrealBloomPass(
    new THREE.Vector2(window.innerWidth, window.innerHeight),
    1.5, 0.4, 0.85
);
composer.addPass(bloomPass);

// Film
const filmPass = new FilmPass(0.35, 0.025, 648, false);
filmPass.enabled = false;
composer.addPass(filmPass);

// Glitch
const glitchPass = new GlitchPass();
glitchPass.enabled = false;
composer.addPass(glitchPass);

// Dot Screen
const dotPass = new DotScreenPass(new THREE.Vector2(0, 0), 0.5, 0.8);
dotPass.enabled = false;
composer.addPass(dotPass);

// Outline
const outlinePass = new OutlinePass(
    new THREE.Vector2(window.innerWidth, window.innerHeight),
    scene, camera
);
outlinePass.edgeStrength = 3;
outlinePass.edgeThickness = 1;
outlinePass.visibleEdgeColor.set('#ffffff');
composer.addPass(outlinePass);

// SMAA (ต้องเป็น pass สุดท้าย)
const smaaPass = new SMAAPass(window.innerWidth, window.innerHeight);
composer.addPass(smaaPass);

// GUI Controls
document.getElementById('bloomEnabled').addEventListener('change', (e) => {
    bloomPass.enabled = e.target.checked;
});
document.getElementById('bloomStrength').addEventListener('input', (e) => {
    bloomPass.strength = parseFloat(e.target.value);
});
document.getElementById('bloomRadius').addEventListener('input', (e) => {
    bloomPass.radius = parseFloat(e.target.value);
});
document.getElementById('bloomThreshold').addEventListener('input', (e) => {
    bloomPass.threshold = parseFloat(e.target.value);
});
document.getElementById('filmEnabled').addEventListener('change', (e) => {
    filmPass.enabled = e.target.checked;
});
document.getElementById('glitchEnabled').addEventListener('change', (e) => {
    glitchPass.enabled = e.target.checked;
});
document.getElementById('glitchWild').addEventListener('change', (e) => {
    glitchPass.goWild = e.target.checked;
});
document.getElementById('dotEnabled').addEventListener('change', (e) => {
    dotPass.enabled = e.target.checked;
});
document.getElementById('smaaEnabled').addEventListener('change', (e) => {
    smaaPass.enabled = e.target.checked;
});
document.getElementById('outlineEnabled').addEventListener('change', (e) => {
    outlinePass.enabled = e.target.checked;
});

// Raycaster สำหรับ hover outline
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();
let allObjects = [...emissiveObjects, ...normalObjects];

renderer.domElement.addEventListener('mousemove', (e) => {
    mouse.x = (e.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(e.clientY / window.innerHeight) * 2 + 1;
    
    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(allObjects);
    
    if (document.getElementById('outlineEnabled').checked) {
        if (intersects.length > 0) {
            outlinePass.selectedObjects = [intersects[0].object];
        } else {
            outlinePass.selectedObjects = [];
        }
    }
});

// Resize
window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
    composer.setSize(window.innerWidth, window.innerHeight);
    bloomPass.setSize(window.innerWidth, window.innerHeight);
});

// Animation
const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    const delta = clock.getDelta();
    
    // หมุนวัตถุ emissive
    emissiveObjects.forEach((obj, i) => {
        if (obj !== centerSphere) {
            const angle = elapsed * 0.5 + (i / emissiveObjects.length) * Math.PI * 2;
            const radius = 5;
            obj.position.x = Math.cos(angle) * radius;
            obj.position.z = Math.sin(angle) * radius;
            obj.position.y = Math.sin(elapsed + i) * 1.5;
            obj.rotation.y += 0.02;
        }
    });
    
    // กระพริบ center sphere
    centerMat.emissiveIntensity = 1.5 + Math.sin(elapsed * 3) * 0.5;
    
    controls.update();
    composer.render();
}
animate();
</script>
</body>
</html>
```

---

## 11. Pass Ordering

ลำดับ passes มีความสำคัญมาก:

```javascript
const composer = new EffectComposer(renderer);

// 1. RenderPass - ต้องเป็นแรกเสมอ
composer.addPass(new RenderPass(scene, camera));

// 2. SSAOPass - ต้องก่อน bloom เพราะเปลี่ยน luminance
composer.addPass(ssaoPass);

// 3. UnrealBloomPass
composer.addPass(bloomPass);

// 4. BokehPass / DOF
composer.addPass(bokehPass);

// 5. OutlinePass
composer.addPass(outlinePass);

// 6. FilmPass, GlitchPass, DotScreenPass - stylistic effects
composer.addPass(filmPass);
// หรือ
composer.addPass(glitchPass);

// 7. ShaderPass (custom)
composer.addPass(vignettePass);
composer.addPass(colorGradingPass);

// 8. SMAAPass/FXAAPass - AA ต้องเป็นสุดท้ายเสมอ
composer.addPass(smaaPass);
```

---

## 12. Performance

```javascript
// ✅ ปิด pass ที่ไม่ใช้
filmPass.enabled = false; // ดีกว่าลบออก (ยังอยู่ใน pipeline แต่ skip)

// ✅ ลด render target size
const composer = new EffectComposer(renderer);
// ใช้ half resolution สำหรับ blur effects
const halfResTarget = new THREE.WebGLRenderTarget(
    window.innerWidth / 2,
    window.innerHeight / 2
);

// ✅ ลด passes ที่ใช้
// SSAO + Bloom + DOF ร่วมกันหนักมาก
// เลือกเฉพาะที่จำเป็น

// ✅ ใช้ pixelRatio ที่เหมาะสม
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
// ไม่ต้องตั้ง devicePixelRatio เต็มๆ บน high-DPI screens

// ✅ อัพเดต composer size เมื่อ resize
window.addEventListener('resize', () => {
    composer.setSize(window.innerWidth, window.innerHeight);
});
```

---

## แบบฝึกหัด

1. **Sci-Fi Scene**: ใช้ Bloom + custom shader สร้าง holographic display
2. **Horror Effect**: รวม Film grain + vignette + desaturation
3. **Retro Arcade**: ใช้ DotScreen + chromatic aberration
4. **Focus Puller**: Bokeh DOF ที่ focus เปลี่ยนตาม hover object
5. **Glitch Art**: สร้าง glitch effect ที่ triggered ตามเหตุการณ์ในเกม

---

## สรุป

- `EffectComposer` เป็น pipeline รัน passes ตามลำดับ
- `RenderPass` ต้องเป็น pass แรก, AA pass ต้องเป็นสุดท้าย
- `UnrealBloomPass` ให้ glow effect สวยงาม
- `OutlinePass` ใช้ highlight ที่ hover/select
- `ShaderPass` ใช้ custom GLSL effects
- ลำดับ passes มีผลต่อผลลัพธ์มาก
- `pass.enabled = false` ดีกว่า remove เพราะกลับมาเปิดได้
