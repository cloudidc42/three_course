# Part 17: Fog ใน Three.js

## บทนำ

Fog ในงาน 3D ไม่ใช่แค่ effect สวยงาม แต่ยังช่วยซ่อน far clipping plane ทำให้ scene ดูต่อเนื่อง ไม่มีขอบตัดกะทันหัน และยังช่วย performance โดยทำให้ objects ที่ไกลมองไม่เห็นก็ไม่ต้อง render เต็มที่

## 1. Fog Class - Linear Fog

Fog คือหมอกแบบ linear: ยิ่งไกลยิ่งเข้มขึ้นในอัตราคงที่

```javascript
// สร้าง Fog
scene.fog = new THREE.Fog(
    color,  // สีหมอก
    near,   // ระยะที่หมอกเริ่มปรากฏ
    far     // ระยะที่หมอกเต็ม 100%
);

// ตัวอย่าง
scene.fog = new THREE.Fog(0x87ceeb, 50, 200); // สีฟ้าท้องฟ้า

// กำหนดสี background ให้ตรงกับหมอก (สำคัญมาก!)
scene.background = new THREE.Color(0x87ceeb);

// อัพเดตค่า fog
scene.fog.color.setHex(0xccddee);
scene.fog.near = 10;
scene.fog.far = 100;

// ปิด fog
scene.fog = null;

// ตรวจสอบ
if (scene.fog) {
    console.log('Fog enabled:', scene.fog.near, scene.fog.far);
}
```

---

## 2. FogExp2 Class - Exponential Fog

FogExp2 หมอกแบบ exponential: ความเข้มเพิ่มขึ้นแบบเอกซ์โพเนนเชียล เหมาะสำหรับ outdoor scenes

```javascript
// สร้าง FogExp2
scene.fog = new THREE.FogExp2(
    color,    // สีหมอก
    density   // ความหนาแน่น (0.001 ถึง 0.1)
);

// ตัวอย่าง
scene.fog = new THREE.FogExp2(0x338833, 0.005); // หมอกสีเขียวในป่า

// density = 0.001 - หมอกบางมาก
// density = 0.005 - หมอกปานกลาง
// density = 0.02  - หมอกหนา
// density = 0.05  - หมอกหนาๆ

// อัพเดต
scene.fog.density = 0.01;
scene.fog.color.setHex(0x888888);
```

### Linear vs Exponential

```
Linear (Fog):
ระยะ:    0----near----[gradient]----far--->
ความเข้ม: 0            0→1         1

Exponential (FogExp2):
ระยะ:    0---[เริ่มหนาทันที]--->
ความเข้ม: 0--ค่อยๆเพิ่มแบบ curve-->1
```

---

## 3. Fog กับ Particles

```javascript
// Particles รับผลกระทบจาก Fog โดยอัตโนมัติ (สำหรับ Points)
// แต่ต้องตั้งค่า fog = true ใน PointsMaterial
const particleMat = new THREE.PointsMaterial({
    color: 0xffffff,
    size: 0.1,
    fog: true  // default = true, ให้ particles ได้รับ fog
});

// SpriteMaterial ก็รองรับ fog
const spriteMat = new THREE.SpriteMaterial({
    map: texture,
    fog: true  // default = true
});

// ปิด fog สำหรับ UI elements
const uiMat = new THREE.MeshBasicMaterial({
    color: 0xff0000,
    fog: false  // ไม่ได้รับผลจาก fog
});
```

---

## 4. Demo สมบูรณ์: Atmospheric Fog Scene

```html
<!DOCTYPE html>
<html>
<head>
    <title>Atmospheric Fog Demo</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { overflow: hidden; font-family: monospace; }
        
        #panel {
            position: absolute;
            top: 10px;
            right: 10px;
            background: rgba(0,0,0,0.8);
            color: white;
            padding: 15px;
            border-radius: 8px;
            min-width: 260px;
            font-size: 13px;
        }
        
        h3 { color: #e94560; margin-bottom: 10px; }
        h4 { color: #4ecdc4; margin: 10px 0 5px; }
        
        .row { display: flex; align-items: center; margin: 5px 0; gap: 6px; }
        .row label { width: 110px; color: #aaa; }
        input[type=range] { width: 100px; }
        .val { color: #ffd700; width: 55px; text-align: right; }
        
        .scene-btns { display: flex; gap: 5px; flex-wrap: wrap; margin: 5px 0; }
        button {
            padding: 6px 10px;
            background: #1a3a5a;
            color: white;
            border: 1px solid #4ecdc4;
            border-radius: 4px;
            cursor: pointer;
            font-size: 11px;
        }
        button.active, button:hover { background: #4ecdc4; color: #000; }
        
        #info {
            position: absolute;
            bottom: 10px;
            left: 10px;
            color: rgba(255,255,255,0.6);
            font-size: 12px;
        }
    </style>
</head>
<body>

<div id="panel">
    <h3>Fog Controls</h3>
    
    <h4>Preset Scenes</h4>
    <div class="scene-btns">
        <button id="btnForest" class="active">🌲 ป่า</button>
        <button id="btnOcean">🌊 ทะเล</button>
        <button id="btnDesert">🏜️ ทะเลทราย</button>
        <button id="btnNight">🌙 กลางคืน</button>
        <button id="btnSnow">❄️ หิมะ</button>
    </div>
    
    <h4>Fog Type</h4>
    <div class="scene-btns">
        <button id="btnLinear" class="active">Linear</button>
        <button id="btnExp2">Exponential</button>
        <button id="btnNone">None</button>
    </div>
    
    <h4>Linear Fog</h4>
    <div class="row">
        <label>Near:</label>
        <input type="range" id="fogNear" min="1" max="200" step="1" value="30">
        <span class="val" id="nearVal">30</span>
    </div>
    <div class="row">
        <label>Far:</label>
        <input type="range" id="fogFar" min="10" max="500" step="5" value="150">
        <span class="val" id="farVal">150</span>
    </div>
    
    <h4>Exponential Fog</h4>
    <div class="row">
        <label>Density:</label>
        <input type="range" id="fogDensity" min="0.001" max="0.05" step="0.001" value="0.005">
        <span class="val" id="densityVal">0.005</span>
    </div>
    
    <h4>Fog Color</h4>
    <div class="row">
        <label>R:</label>
        <input type="range" id="fogR" min="0" max="255" step="1" value="51">
        <span class="val" id="rVal">51</span>
    </div>
    <div class="row">
        <label>G:</label>
        <input type="range" id="fogG" min="0" max="255" step="1" value="136">
        <span class="val" id="gVal">136</span>
    </div>
    <div class="row">
        <label>B:</label>
        <input type="range" id="fogB" min="0" max="255" step="1" value="51">
        <span class="val" id="bVal">51</span>
    </div>
    
    <h4>Animation</h4>
    <div class="row">
        <label>Animated Fog:</label>
        <input type="checkbox" id="animFog">
    </div>
    <div class="row">
        <label>Anim Speed:</label>
        <input type="range" id="animSpeed" min="0.1" max="3" step="0.1" value="1">
        <span class="val" id="speedVal">1.0</span>
    </div>
</div>

<div id="info">OrbitControls: หมุน/ซูม/เลื่อน | เลื่อนเมาส์เพื่อดูผล fog</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';

// Scene
const scene = new THREE.Scene();

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(0, 8, 20);
camera.lookAt(0, 0, 0);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.target.set(0, 2, 0);

// Presets
const presets = {
    forest: {
        fogColor: new THREE.Color(0x336633),
        bgColor: new THREE.Color(0x1a4a1a),
        near: 30, far: 150, density: 0.008,
        ambient: 0x334433, ambientInt: 0.6,
        sun: 0x88bb66, sunInt: 0.8
    },
    ocean: {
        fogColor: new THREE.Color(0x224488),
        bgColor: new THREE.Color(0x112244),
        near: 20, far: 200, density: 0.004,
        ambient: 0x334466, ambientInt: 0.5,
        sun: 0x88aaff, sunInt: 1.0
    },
    desert: {
        fogColor: new THREE.Color(0xddcc88),
        bgColor: new THREE.Color(0xcc9944),
        near: 50, far: 400, density: 0.002,
        ambient: 0xddcc88, ambientInt: 0.8,
        sun: 0xffdd99, sunInt: 2.0
    },
    night: {
        fogColor: new THREE.Color(0x050515),
        bgColor: new THREE.Color(0x010110),
        near: 10, far: 80, density: 0.02,
        ambient: 0x101030, ambientInt: 0.3,
        sun: 0x4455aa, sunInt: 0.2
    },
    snow: {
        fogColor: new THREE.Color(0xddddee),
        bgColor: new THREE.Color(0xccccdd),
        near: 15, far: 100, density: 0.015,
        ambient: 0xaaaacc, ambientInt: 0.7,
        sun: 0xffffff, sunInt: 1.2
    }
};

let currentPreset = presets.forest;
let fogType = 'linear'; // 'linear', 'exp2', 'none'

// Lighting
const ambientLight = new THREE.AmbientLight(0x334433, 0.6);
scene.add(ambientLight);

const sunLight = new THREE.DirectionalLight(0x88bb66, 0.8);
sunLight.position.set(10, 20, 10);
sunLight.castShadow = true;
sunLight.shadow.mapSize.width = 1024;
sunLight.shadow.mapSize.height = 1024;
sunLight.shadow.camera.left = -50;
sunLight.shadow.camera.right = 50;
sunLight.shadow.camera.top = 50;
sunLight.shadow.camera.bottom = -50;
scene.add(sunLight);

// Forest scene elements
function createTree(x, z) {
    const group = new THREE.Group();
    
    // ลำต้น
    const trunk = new THREE.Mesh(
        new THREE.CylinderGeometry(0.2, 0.3, 2, 8),
        new THREE.MeshStandardMaterial({ color: 0x4a3728, roughness: 0.9 })
    );
    trunk.position.y = 1;
    trunk.castShadow = true;
    group.add(trunk);
    
    // ใบไม้ (สามชั้น)
    const leafMat = new THREE.MeshStandardMaterial({ 
        color: 0x2d5a27,
        roughness: 0.8
    });
    
    [{ r: 2, h: 3, y: 3 }, { r: 1.5, h: 2.5, y: 4.5 }, { r: 1, h: 2, y: 5.8 }].forEach(l => {
        const leaves = new THREE.Mesh(
            new THREE.ConeGeometry(l.r, l.h, 8),
            leafMat
        );
        leaves.position.y = l.y;
        leaves.castShadow = true;
        group.add(leaves);
    });
    
    group.position.set(x, 0, z);
    group.rotation.y = Math.random() * Math.PI * 2;
    group.scale.setScalar(0.8 + Math.random() * 0.5);
    scene.add(group);
    return group;
}

// พื้นหญ้า
const floor = new THREE.Mesh(
    new THREE.PlaneGeometry(300, 300),
    new THREE.MeshStandardMaterial({ color: 0x2a5a1a, roughness: 0.9 })
);
floor.rotation.x = -Math.PI / 2;
floor.receiveShadow = true;
scene.add(floor);

// สร้างต้นไม้
const trees = [];
for (let i = 0; i < 100; i++) {
    const angle = Math.random() * Math.PI * 2;
    const dist = 10 + Math.random() * 80;
    trees.push(createTree(
        Math.sin(angle) * dist,
        Math.cos(angle) * dist
    ));
}

// สร้างหิน
for (let i = 0; i < 30; i++) {
    const rock = new THREE.Mesh(
        new THREE.DodecahedronGeometry(0.3 + Math.random() * 0.8),
        new THREE.MeshStandardMaterial({ color: 0x777766, roughness: 0.9 })
    );
    rock.position.set(
        (Math.random() - 0.5) * 100,
        0.2,
        (Math.random() - 0.5) * 100
    );
    rock.rotation.set(Math.random(), Math.random(), Math.random());
    rock.castShadow = true;
    rock.receiveShadow = true;
    scene.add(rock);
}

// หมอก Particles
const fogParticleGeo = new THREE.BufferGeometry();
const particleCount = 2000;
const particlePositions = new Float32Array(particleCount * 3);

for (let i = 0; i < particleCount; i++) {
    particlePositions[i * 3] = (Math.random() - 0.5) * 200;
    particlePositions[i * 3 + 1] = Math.random() * 8;
    particlePositions[i * 3 + 2] = (Math.random() - 0.5) * 200;
}

fogParticleGeo.setAttribute('position', new THREE.BufferAttribute(particlePositions, 3));
const fogParticleMat = new THREE.PointsMaterial({
    color: 0x88aa88,
    size: 0.5,
    transparent: true,
    opacity: 0.3,
    fog: true
});
const fogParticles = new THREE.Points(fogParticleGeo, fogParticleMat);
scene.add(fogParticles);

// Apply preset
function applyPreset(preset) {
    currentPreset = preset;
    
    scene.background = preset.bgColor.clone();
    
    ambientLight.color.setHex(preset.ambient);
    ambientLight.intensity = preset.ambientInt;
    
    sunLight.color.setHex(preset.sun);
    sunLight.intensity = preset.sunInt;
    
    applyFog();
    updateSliders(preset);
}

function applyFog() {
    const r = parseInt(document.getElementById('fogR').value);
    const g = parseInt(document.getElementById('fogG').value);
    const b = parseInt(document.getElementById('fogB').value);
    const color = new THREE.Color(r / 255, g / 255, b / 255);
    
    if (fogType === 'linear') {
        const near = parseFloat(document.getElementById('fogNear').value);
        const far = parseFloat(document.getElementById('fogFar').value);
        scene.fog = new THREE.Fog(color, near, far);
    } else if (fogType === 'exp2') {
        const density = parseFloat(document.getElementById('fogDensity').value);
        scene.fog = new THREE.FogExp2(color, density);
    } else {
        scene.fog = null;
    }
    
    scene.background = color;
}

function updateSliders(preset) {
    document.getElementById('fogNear').value = preset.near;
    document.getElementById('nearVal').textContent = preset.near;
    document.getElementById('fogFar').value = preset.far;
    document.getElementById('farVal').textContent = preset.far;
    document.getElementById('fogDensity').value = preset.density;
    document.getElementById('densityVal').textContent = preset.density;
    
    const c = preset.fogColor;
    document.getElementById('fogR').value = Math.round(c.r * 255);
    document.getElementById('fogG').value = Math.round(c.g * 255);
    document.getElementById('fogB').value = Math.round(c.b * 255);
    document.getElementById('rVal').textContent = Math.round(c.r * 255);
    document.getElementById('gVal').textContent = Math.round(c.g * 255);
    document.getElementById('bVal').textContent = Math.round(c.b * 255);
    
    applyFog();
}

// Initial fog
scene.fog = new THREE.Fog(presets.forest.fogColor, 30, 150);
scene.background = presets.forest.bgColor.clone();

// Event listeners
document.getElementById('btnForest').addEventListener('click', () => {
    applyPreset(presets.forest);
    document.querySelectorAll('.scene-btns button').forEach(b => b.classList.remove('active'));
    document.getElementById('btnForest').classList.add('active');
});
document.getElementById('btnOcean').addEventListener('click', () => {
    applyPreset(presets.ocean);
    document.querySelectorAll('.scene-btns:nth-of-type(1) button').forEach(b => b.classList.remove('active'));
    document.getElementById('btnOcean').classList.add('active');
});
document.getElementById('btnDesert').addEventListener('click', () => {
    applyPreset(presets.desert);
    document.getElementById('btnDesert').classList.add('active');
});
document.getElementById('btnNight').addEventListener('click', () => {
    applyPreset(presets.night);
    document.getElementById('btnNight').classList.add('active');
});
document.getElementById('btnSnow').addEventListener('click', () => {
    applyPreset(presets.snow);
    document.getElementById('btnSnow').classList.add('active');
});

document.getElementById('btnLinear').addEventListener('click', () => {
    fogType = 'linear'; applyFog();
    document.querySelectorAll('#panel .scene-btns:nth-of-type(2) button').forEach(b => b.classList.remove('active'));
    document.getElementById('btnLinear').classList.add('active');
});
document.getElementById('btnExp2').addEventListener('click', () => {
    fogType = 'exp2'; applyFog();
    document.querySelectorAll('#panel .scene-btns:nth-of-type(2) button').forEach(b => b.classList.remove('active'));
    document.getElementById('btnExp2').classList.add('active');
});
document.getElementById('btnNone').addEventListener('click', () => {
    fogType = 'none'; applyFog();
    document.querySelectorAll('#panel .scene-btns:nth-of-type(2) button').forEach(b => b.classList.remove('active'));
    document.getElementById('btnNone').classList.add('active');
});

// Slider events
['fogNear', 'fogFar', 'fogDensity', 'fogR', 'fogG', 'fogB'].forEach(id => {
    document.getElementById(id).addEventListener('input', (e) => {
        const valId = {
            fogNear: 'nearVal', fogFar: 'farVal', fogDensity: 'densityVal',
            fogR: 'rVal', fogG: 'gVal', fogB: 'bVal'
        }[id];
        document.getElementById(valId).textContent = e.target.value;
        applyFog();
    });
});

document.getElementById('animSpeed').addEventListener('input', (e) => {
    document.getElementById('speedVal').textContent = parseFloat(e.target.value).toFixed(1);
});

window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});

// Animation
const clock = new THREE.Clock();
let fogAnimTime = 0;

function animate() {
    requestAnimationFrame(animate);
    const delta = clock.getDelta();
    const elapsed = clock.getElapsedTime();
    
    // Animated fog
    if (document.getElementById('animFog').checked && scene.fog) {
        fogAnimTime += delta * parseFloat(document.getElementById('animSpeed').value);
        
        if (scene.fog.isFog) {
            // Linear fog oscillates
            const baseFar = parseFloat(document.getElementById('fogFar').value);
            scene.fog.far = baseFar + Math.sin(fogAnimTime) * baseFar * 0.3;
        } else if (scene.fog.isFogExp2) {
            // Exp2 fog density oscillates
            const baseDensity = parseFloat(document.getElementById('fogDensity').value);
            scene.fog.density = baseDensity + Math.sin(fogAnimTime * 0.5) * baseDensity * 0.5;
        }
    }
    
    // หมุน fog particles ช้าๆ
    fogParticles.rotation.y += 0.0002;
    
    // ลอย particles
    const positions = fogParticleGeo.attributes.position.array;
    for (let i = 0; i < particleCount; i++) {
        positions[i * 3 + 1] = Math.sin(elapsed * 0.3 + i * 0.1) * 1 + 3;
    }
    fogParticleGeo.attributes.position.needsUpdate = true;
    
    controls.update();
    renderer.render(scene, camera);
}
animate();
</script>
</body>
</html>
```

---

## 5. Animated Fog

```javascript
// Fog ที่เปลี่ยนแปลงตามเวลา
class AnimatedFog {
    constructor(scene, baseColor, baseNear, baseFar) {
        this.scene = scene;
        this.baseColor = new THREE.Color(baseColor);
        this.baseNear = baseNear;
        this.baseFar = baseFar;
        this.time = 0;
        
        this.scene.fog = new THREE.Fog(baseColor, baseNear, baseFar);
        this.scene.background = new THREE.Color(baseColor);
    }
    
    update(delta) {
        this.time += delta;
        
        // Oscillate fog distance
        const fogVariation = Math.sin(this.time * 0.2) * 0.3 + 1;
        this.scene.fog.far = this.baseFar * fogVariation;
        
        // Oscillate color slightly
        const r = this.baseColor.r + Math.sin(this.time * 0.1) * 0.05;
        const g = this.baseColor.g + Math.sin(this.time * 0.15) * 0.05;
        const b = this.baseColor.b + Math.sin(this.time * 0.2) * 0.05;
        
        this.scene.fog.color.setRGB(
            Math.max(0, Math.min(1, r)),
            Math.max(0, Math.min(1, g)),
            Math.max(0, Math.min(1, b))
        );
        this.scene.background.copy(this.scene.fog.color);
    }
}

const animatedFog = new AnimatedFog(scene, 0x334433, 20, 100);

function animate() {
    const delta = clock.getDelta();
    animatedFog.update(delta);
    // ...
}
```

---

## 6. Fog สำหรับ Performance

```javascript
// Fog ช่วย performance ด้วยการซ่อน objects ที่ไกล
// แต่ Three.js ไม่ได้ skip rendering ของ fog objects โดยอัตโนมัติ
// ต้อง combine กับ frustum culling

// วิธีที่ 1: ตั้ง camera.far ให้ตรงกับ fog.far
camera.far = scene.fog.far;
camera.updateProjectionMatrix();

// วิธีที่ 2: ใช้ LOD (Level of Detail) กับ fog
// ลด polygon count สำหรับวัตถุไกล
const lod = new THREE.LOD();

const highDetail = new THREE.Mesh(
    new THREE.SphereGeometry(1, 64, 64),  // High poly
    mat
);
const lowDetail = new THREE.Mesh(
    new THREE.SphereGeometry(1, 8, 8),    // Low poly
    mat
);

lod.addLevel(highDetail, 0);    // ใช้ high detail ใกล้กว่า 0 หน่วย
lod.addLevel(lowDetail, 50);    // ใช้ low detail ไกลกว่า 50 หน่วย

scene.add(lod);

// วิธีที่ 3: ปิดการแสดงวัตถุที่อยู่ใน fog เต็มๆ
function updateVisibility() {
    const cameraPos = camera.position;
    scene.traverse(obj => {
        if (obj.isMesh && scene.fog) {
            const dist = obj.position.distanceTo(cameraPos);
            if (scene.fog.isFog) {
                // Linear fog
                obj.visible = dist < scene.fog.far * 1.2;
            } else if (scene.fog.isFogExp2) {
                // Exponential fog
                const opacity = Math.exp(-dist * scene.fog.density);
                obj.visible = opacity > 0.01; // 1% opacity
            }
        }
    });
}
```

---

## 7. Fog สีจาก Skybox

```javascript
// ถ้าใช้ skybox ต้องตั้ง fog color ให้ blend กับ horizon
// ปกติ fog จะไม่ match กับ skybox ถ้าใช้ CubeMap

// วิธีแก้: ใช้ Gradient Background แทน
function createGradientBackground(topColor, bottomColor) {
    const canvas = document.createElement('canvas');
    canvas.width = 2;
    canvas.height = 512;
    
    const ctx = canvas.getContext('2d');
    const gradient = ctx.createLinearGradient(0, 0, 0, 512);
    gradient.addColorStop(0, topColor);
    gradient.addColorStop(1, bottomColor);
    ctx.fillStyle = gradient;
    ctx.fillRect(0, 0, 2, 512);
    
    const texture = new THREE.CanvasTexture(canvas);
    texture.mapping = THREE.EquirectangularReflectionMapping;
    
    return texture;
}

// fog color = bottomColor ของ gradient
const topColor = '#87ceeb';    // ท้องฟ้า
const bottomColor = '#336633'; // ขอบฟ้า/พื้นดิน
scene.background = createGradientBackground(topColor, bottomColor);
scene.fog = new THREE.Fog(new THREE.Color(bottomColor), 50, 200);

// วิธีที่ 2: ใช้ shader สำหรับ gradient sky + fog
const skyGeo = new THREE.SphereGeometry(500, 32, 32);
const skyMat = new THREE.ShaderMaterial({
    vertexShader: `
        varying vec3 vWorldPosition;
        void main() {
            vec4 worldPos = modelMatrix * vec4(position, 1.0);
            vWorldPosition = worldPos.xyz;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
    `,
    fragmentShader: `
        uniform vec3 topColor;
        uniform vec3 bottomColor;
        uniform float offset;
        uniform float exponent;
        varying vec3 vWorldPosition;
        
        void main() {
            float h = normalize(vWorldPosition + offset).y;
            gl_FragColor = vec4(mix(bottomColor, topColor, max(pow(max(h, 0.0), exponent), 0.0)), 1.0);
        }
    `,
    uniforms: {
        topColor: { value: new THREE.Color(0x0077ff) },
        bottomColor: { value: new THREE.Color(0xffffff) },
        offset: { value: 400 },
        exponent: { value: 0.6 }
    },
    side: THREE.BackSide
});

const sky = new THREE.Mesh(skyGeo, skyMat);
scene.add(sky);
```

---

## 8. Ocean Fog Demo

```javascript
// ใต้น้ำ fog effect
function setupUnderwaterFog(scene, renderer) {
    scene.fog = new THREE.FogExp2(0x004466, 0.03);
    scene.background = new THREE.Color(0x001133);
    
    // Blue point light (แสงจากผิวน้ำ)
    const waterLight = new THREE.PointLight(0x0088ff, 2, 50);
    waterLight.position.set(0, 20, 0);
    scene.add(waterLight);
    
    // เพิ่ม ambient สีน้ำเงิน
    const ambientLight = new THREE.AmbientLight(0x001a33, 0.8);
    scene.add(ambientLight);
    
    // ปรับ clear color
    renderer.setClearColor(0x001133);
    
    // Caustics effect (แสงระยิบระยับในน้ำ)
    const causticLight = new THREE.SpotLight(0x44aaff, 1, 30, Math.PI / 4, 0.5);
    causticLight.position.set(0, 20, 0);
    scene.add(causticLight);
    
    return { waterLight, causticLight };
}

// อัพเดต caustics ในลูป
function updateCaustics(elapsed, causticLight) {
    causticLight.position.x = Math.sin(elapsed * 0.5) * 5;
    causticLight.position.z = Math.cos(elapsed * 0.3) * 5;
    causticLight.intensity = 0.5 + Math.sin(elapsed * 2) * 0.3;
}
```

---

## แบบฝึกหัด

1. **Weather System**: สร้างระบบสภาพอากาศที่สลับระหว่าง sunny, foggy, stormy
2. **Distance Fading**: สร้าง objects ที่ค่อยๆ fade เมื่อไกลออกไป
3. **Fog of War**: ใช้ fog ซ่อน area ที่ยังไม่ explore ในเกม
4. **Depth Effect**: ใช้ fog สร้าง depth cue ใน scene ห้อง
5. **Animated Mist**: สร้าง particle system ที่เคลื่อนที่พร้อม fog

---

## สรุป

- `scene.fog = new THREE.Fog(color, near, far)` - Linear fog
- `scene.fog = new THREE.FogExp2(color, density)` - Exponential fog
- ต้องตั้ง `scene.background` ให้ตรงกับสี fog เสมอ
- `Fog` เหมาะกับ indoor/corridor, `FogExp2` เหมาะกับ outdoor
- Materials มี `fog` property สำหรับเปิด/ปิดผลกระทบของ fog
- Fog ช่วย performance โดยซ่อน far objects
- Combine fog กับ `camera.far` เพื่อ performance ที่ดีขึ้น
