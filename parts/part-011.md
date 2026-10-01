# Part 11: Camera Controls ใน Three.js

## บทนำ

การควบคุมกล้องเป็นหนึ่งในส่วนที่สำคัญที่สุดในการสร้างประสบการณ์ 3D ที่ดี Three.js มี Controls หลายประเภทที่ให้เราใช้งานได้ทันที แต่ละประเภทมีจุดประสงค์และการใช้งานที่แตกต่างกัน

ในบทนี้เราจะเรียนรู้:
- **OrbitControls** - การหมุนรอบวัตถุ (ใช้บ่อยที่สุด)
- **FlyControls** - การบินอิสระในพื้นที่ 3D
- **FirstPersonControls** - มุมมองบุคคลที่หนึ่ง
- **PointerLockControls** - สำหรับเกม FPS
- **TrackballControls** - หมุนได้อิสระ 360 องศา
- **ArcballControls** - ควบคุมแบบ Arcball
- **TransformControls** - จัดการ position/rotation/scale ของวัตถุ

## การติดตั้ง Controls

Controls ทั้งหมดอยู่ใน `three/examples/jsm/controls/` ต้องนำเข้าแยกต่างหาก

```javascript
// นำเข้าแต่ละ Control ที่ต้องการ
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
import { FlyControls } from 'three/examples/jsm/controls/FlyControls.js';
import { FirstPersonControls } from 'three/examples/jsm/controls/FirstPersonControls.js';
import { PointerLockControls } from 'three/examples/jsm/controls/PointerLockControls.js';
import { TrackballControls } from 'three/examples/jsm/controls/TrackballControls.js';
import { ArcballControls } from 'three/examples/jsm/controls/ArcballControls.js';
import { TransformControls } from 'three/examples/jsm/controls/TransformControls.js';
```

---

## 1. OrbitControls

OrbitControls เป็น Control ที่ใช้บ่อยที่สุด ช่วยให้ผู้ใช้สามารถ:
- **Orbit (หมุน)**: คลิกซ้ายแล้วลาก
- **Zoom**: เลื่อนล้อเมาส์
- **Pan (เลื่อน)**: คลิกขวาหรือ Shift + คลิกซ้าย

### การตั้งค่าพื้นฐาน

```javascript
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

// ตั้งค่า Scene
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(5, 5, 5);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
document.body.appendChild(renderer.domElement);

// สร้าง OrbitControls
const controls = new OrbitControls(camera, renderer.domElement);

// animation loop
function animate() {
    requestAnimationFrame(animate);
    controls.update(); // ต้องเรียกทุก frame ถ้าเปิด damping
    renderer.render(scene, camera);
}
animate();
```

### คุณสมบัติสำคัญของ OrbitControls

```javascript
const controls = new OrbitControls(camera, renderer.domElement);

// === Damping (ความหน่วง) ===
// ทำให้การเคลื่อนที่ดูลื่นและเป็นธรรมชาติมากขึ้น
controls.enableDamping = true;      // เปิดใช้ damping
controls.dampingFactor = 0.05;      // ค่า damping (0-1, ยิ่งน้อยยิ่งลื่น)
// หมายเหตุ: ถ้าเปิด damping ต้องเรียก controls.update() ทุก frame

// === ระยะทาง (Zoom Limits) ===
controls.minDistance = 2;           // ระยะใกล้สุดที่ zoom เข้าได้
controls.maxDistance = 50;          // ระยะไกลสุดที่ zoom ออกได้

// === มุมแนวตั้ง (Polar Angle) ===
// กำหนดว่ากล้องสามารถหมุนได้มากแค่ไหนในแนวตั้ง
controls.minPolarAngle = 0;                    // มองจากด้านบนสุด (0 = บน)
controls.maxPolarAngle = Math.PI / 2;          // จำกัดไม่ให้มองต่ำกว่าแนวระนาบ
// controls.maxPolarAngle = Math.PI;           // ไม่จำกัด (ค่าเริ่มต้น)

// === มุมแนวนอน (Azimuth Angle) ===
controls.minAzimuthAngle = -Math.PI / 2;       // จำกัดการหมุนซ้าย
controls.maxAzimuthAngle = Math.PI / 2;        // จำกัดการหมุนขวา
// ถ้าไม่ต้องการจำกัด ไม่ต้องตั้งค่า (ค่าเริ่มต้น = -Infinity ถึง Infinity)

// === Auto Rotate ===
controls.autoRotate = true;                    // หมุนอัตโนมัติ
controls.autoRotateSpeed = 2.0;               // ความเร็ว (องศาต่อวินาที)
// autoRotateSpeed = 1.0 หมายถึง 30 วินาทีต่อรอบ

// === Enable/Disable Features ===
controls.enablePan = true;          // เปิด/ปิดการเลื่อน (panning)
controls.enableZoom = true;         // เปิด/ปิดการซูม
controls.enableRotate = true;       // เปิด/ปิดการหมุน

// === Target (จุดที่กล้องโฟกัส) ===
controls.target.set(0, 0, 0);      // ค่าเริ่มต้น = จุด origin
controls.target.set(0, 2, 0);      // โฟกัสที่ความสูง y=2

// === Pan Speed ===
controls.panSpeed = 1.0;            // ความเร็วการเลื่อน
controls.zoomSpeed = 1.0;           // ความเร็วการซูม
controls.rotateSpeed = 1.0;         // ความเร็วการหมุน

// === Screen Space Panning ===
controls.screenSpacePanning = true;  // เลื่อนตามระนาบหน้าจอ
                                     // false = เลื่อนตามแกน y
```

### Demo สมบูรณ์: OrbitControls พร้อม GUI

```html
<!DOCTYPE html>
<html>
<head>
    <title>OrbitControls Demo</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #1a1a2e; overflow: hidden; }
        #info {
            position: absolute;
            top: 10px;
            left: 10px;
            color: white;
            font-family: monospace;
            background: rgba(0,0,0,0.5);
            padding: 10px;
            border-radius: 5px;
        }
        #controls-panel {
            position: absolute;
            top: 10px;
            right: 10px;
            color: white;
            font-family: monospace;
            background: rgba(0,0,0,0.7);
            padding: 15px;
            border-radius: 5px;
            min-width: 200px;
        }
        .control-row {
            display: flex;
            justify-content: space-between;
            margin: 5px 0;
            align-items: center;
        }
        input[type="range"] { width: 100px; }
        input[type="checkbox"] { cursor: pointer; }
    </style>
</head>
<body>
<div id="info">
    <h3>OrbitControls Demo</h3>
    <p>🖱️ ซ้าย: หมุน | ขวา: เลื่อน | เลื่อนล้อ: ซูม</p>
</div>

<div id="controls-panel">
    <h3>Settings</h3>
    <div class="control-row">
        <label>Damping:</label>
        <input type="checkbox" id="damping" checked>
    </div>
    <div class="control-row">
        <label>DampingFactor:</label>
        <input type="range" id="dampingFactor" min="0.01" max="0.5" step="0.01" value="0.05">
        <span id="dampingValue">0.05</span>
    </div>
    <div class="control-row">
        <label>AutoRotate:</label>
        <input type="checkbox" id="autoRotate">
    </div>
    <div class="control-row">
        <label>RotateSpeed:</label>
        <input type="range" id="rotateSpeed" min="0.1" max="5" step="0.1" value="2">
        <span id="rotateValue">2.0</span>
    </div>
    <div class="control-row">
        <label>MinDist:</label>
        <input type="range" id="minDist" min="1" max="10" step="0.5" value="2">
        <span id="minDistValue">2</span>
    </div>
    <div class="control-row">
        <label>MaxDist:</label>
        <input type="range" id="maxDist" min="10" max="100" step="5" value="20">
        <span id="maxDistValue">20</span>
    </div>
    <div class="control-row">
        <label>EnablePan:</label>
        <input type="checkbox" id="enablePan" checked>
    </div>
    <div class="control-row">
        <label>EnableZoom:</label>
        <input type="checkbox" id="enableZoom" checked>
    </div>
</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';

// Scene setup
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x1a1a2e);

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(8, 6, 8);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
renderer.shadowMap.enabled = true;
document.body.appendChild(renderer.domElement);

// OrbitControls
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.dampingFactor = 0.05;
controls.minDistance = 2;
controls.maxDistance = 20;
controls.target.set(0, 1, 0);
controls.update();

// Lighting
const ambientLight = new THREE.AmbientLight(0x404040, 0.5);
scene.add(ambientLight);

const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
directionalLight.position.set(5, 10, 5);
directionalLight.castShadow = true;
scene.add(directionalLight);

// สร้างวัตถุใน Scene
// พื้น
const floorGeometry = new THREE.PlaneGeometry(20, 20);
const floorMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x16213e,
    roughness: 0.8,
    metalness: 0.2
});
const floor = new THREE.Mesh(floorGeometry, floorMaterial);
floor.rotation.x = -Math.PI / 2;
floor.receiveShadow = true;
scene.add(floor);

// Grid Helper
const gridHelper = new THREE.GridHelper(20, 20, 0x0f3460, 0x0f3460);
scene.add(gridHelper);

// วัตถุหลาย ๆ ชิ้น
const materials = [
    new THREE.MeshStandardMaterial({ color: 0xe94560, roughness: 0.3, metalness: 0.7 }),
    new THREE.MeshStandardMaterial({ color: 0x0f3460, roughness: 0.5, metalness: 0.5 }),
    new THREE.MeshStandardMaterial({ color: 0x16213e, roughness: 0.7, metalness: 0.3 }),
    new THREE.MeshStandardMaterial({ color: 0x533483, roughness: 0.2, metalness: 0.8 }),
];

const geometries = [
    new THREE.BoxGeometry(1.5, 1.5, 1.5),
    new THREE.SphereGeometry(0.8, 32, 32),
    new THREE.ConeGeometry(0.7, 1.5, 32),
    new THREE.TorusGeometry(0.7, 0.3, 16, 100),
];

const positions = [
    { x: -3, y: 0.75, z: -3 },
    { x: 3, y: 0.8, z: -3 },
    { x: 3, y: 0.75, z: 3 },
    { x: -3, y: 0.7, z: 3 },
];

const objects = positions.map((pos, i) => {
    const mesh = new THREE.Mesh(geometries[i], materials[i]);
    mesh.position.set(pos.x, pos.y, pos.z);
    mesh.castShadow = true;
    scene.add(mesh);
    return mesh;
});

// กล่องตรงกลาง
const centerBox = new THREE.Mesh(
    new THREE.BoxGeometry(1, 2, 1),
    new THREE.MeshStandardMaterial({ color: 0xffd700, roughness: 0.2, metalness: 0.9 })
);
centerBox.position.y = 1;
centerBox.castShadow = true;
scene.add(centerBox);

// Axes Helper
const axesHelper = new THREE.AxesHelper(3);
scene.add(axesHelper);

// GUI Controls
const dampingCheckbox = document.getElementById('damping');
const dampingFactorSlider = document.getElementById('dampingFactor');
const dampingValueSpan = document.getElementById('dampingValue');
const autoRotateCheckbox = document.getElementById('autoRotate');
const rotateSpeedSlider = document.getElementById('rotateSpeed');
const rotateValueSpan = document.getElementById('rotateValue');
const minDistSlider = document.getElementById('minDist');
const minDistValueSpan = document.getElementById('minDistValue');
const maxDistSlider = document.getElementById('maxDist');
const maxDistValueSpan = document.getElementById('maxDistValue');
const enablePanCheckbox = document.getElementById('enablePan');
const enableZoomCheckbox = document.getElementById('enableZoom');

dampingCheckbox.addEventListener('change', () => {
    controls.enableDamping = dampingCheckbox.checked;
});

dampingFactorSlider.addEventListener('input', () => {
    controls.dampingFactor = parseFloat(dampingFactorSlider.value);
    dampingValueSpan.textContent = dampingFactorSlider.value;
});

autoRotateCheckbox.addEventListener('change', () => {
    controls.autoRotate = autoRotateCheckbox.checked;
});

rotateSpeedSlider.addEventListener('input', () => {
    controls.autoRotateSpeed = parseFloat(rotateSpeedSlider.value);
    rotateValueSpan.textContent = rotateSpeedSlider.value;
});

minDistSlider.addEventListener('input', () => {
    controls.minDistance = parseFloat(minDistSlider.value);
    minDistValueSpan.textContent = minDistSlider.value;
});

maxDistSlider.addEventListener('input', () => {
    controls.maxDistance = parseFloat(maxDistSlider.value);
    maxDistValueSpan.textContent = maxDistSlider.value;
});

enablePanCheckbox.addEventListener('change', () => {
    controls.enablePan = enablePanCheckbox.checked;
});

enableZoomCheckbox.addEventListener('change', () => {
    controls.enableZoom = enableZoomCheckbox.checked;
});

// Resize
window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});

// Animation Loop
const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    
    // หมุนวัตถุ
    objects.forEach((obj, i) => {
        obj.rotation.x = elapsed * 0.5 + i;
        obj.rotation.y = elapsed * 0.3 + i * 0.5;
    });
    centerBox.rotation.y = elapsed;
    
    controls.update();
    renderer.render(scene, camera);
}
animate();
</script>
</body>
</html>
```

---

## 2. FlyControls

FlyControls ให้ความรู้สึกเหมือนกำลังบินในพื้นที่ 3D สามารถเคลื่อนที่ไปในทุกทิศทางรวมถึงขึ้น-ลง

### วิธีการควบคุม
- **W/S** หรือ **ลูกศรขึ้น/ลง**: เดินหน้า/ถอยหลัง
- **A/D** หรือ **ลูกศรซ้าย/ขวา**: เลื่อนซ้าย/ขวา
- **R/F**: ขึ้น/ลง
- **เมาส์**: หมุนมุมมอง
- **Q**: หมุน roll

```javascript
import { FlyControls } from 'three/examples/jsm/controls/FlyControls.js';

const controls = new FlyControls(camera, renderer.domElement);

// ความเร็วการเคลื่อนที่
controls.movementSpeed = 10;        // ความเร็วการเคลื่อนที่
controls.rollSpeed = Math.PI / 24;  // ความเร็วการ roll
controls.autoForward = false;       // บินไปข้างหน้าอัตโนมัติ
controls.dragToLook = true;         // ต้องกดเมาส์ค้างเพื่อมองรอบ

// ต้องเรียก update ใน animation loop พร้อม delta time!
const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    const delta = clock.getDelta(); // delta time สำคัญมาก
    controls.update(delta);         // ต้องส่ง delta เสมอ
    renderer.render(scene, camera);
}
```

### Demo สมบูรณ์: FlyControls

```javascript
import * as THREE from 'three';
import { FlyControls } from 'three/examples/jsm/controls/FlyControls.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x000011);
scene.fog = new THREE.FogExp2(0x000011, 0.002);

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 2000);
camera.position.set(0, 50, 200);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// FlyControls
const controls = new FlyControls(camera, renderer.domElement);
controls.movementSpeed = 30;
controls.rollSpeed = Math.PI / 12;
controls.dragToLook = true;

// สร้างโลก
const ambientLight = new THREE.AmbientLight(0x222244);
scene.add(ambientLight);

// ดาว
const starGeometry = new THREE.BufferGeometry();
const starCount = 5000;
const starPositions = new Float32Array(starCount * 3);
for (let i = 0; i < starCount * 3; i++) {
    starPositions[i] = (Math.random() - 0.5) * 2000;
}
starGeometry.setAttribute('position', new THREE.BufferAttribute(starPositions, 3));
const starMaterial = new THREE.PointsMaterial({ color: 0xffffff, size: 0.5 });
scene.add(new THREE.Points(starGeometry, starMaterial));

// ดาวเคราะห์
const planets = [];
const planetColors = [0xff6b6b, 0x4ecdc4, 0x45b7d1, 0x96ceb4, 0xffeaa7];
for (let i = 0; i < 20; i++) {
    const geo = new THREE.SphereGeometry(Math.random() * 8 + 2, 32, 32);
    const mat = new THREE.MeshPhongMaterial({ 
        color: planetColors[Math.floor(Math.random() * planetColors.length)],
        shininess: 30
    });
    const planet = new THREE.Mesh(geo, mat);
    planet.position.set(
        (Math.random() - 0.5) * 800,
        (Math.random() - 0.5) * 400,
        (Math.random() - 0.5) * 800
    );
    scene.add(planet);
    
    // เพิ่มแสงให้แต่ละดาว
    const light = new THREE.PointLight(
        planetColors[Math.floor(Math.random() * planetColors.length)],
        0.5, 100
    );
    light.position.copy(planet.position);
    scene.add(light);
    planets.push(planet);
}

// สร้างดาว (sun)
const sunGeo = new THREE.SphereGeometry(20, 32, 32);
const sunMat = new THREE.MeshBasicMaterial({ color: 0xffdd44 });
const sun = new THREE.Mesh(sunGeo, sunMat);
scene.add(sun);
const sunLight = new THREE.PointLight(0xffdd44, 2, 1000);
scene.add(sunLight);

const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    const delta = clock.getDelta();
    
    // หมุนดาวเคราะห์
    planets.forEach((planet, i) => {
        planet.rotation.y += 0.01 * (i % 3 + 1);
    });
    
    controls.update(delta);
    renderer.render(scene, camera);
}
animate();
```

---

## 3. PointerLockControls (FPS Controls)

PointerLockControls ใช้สำหรับเกม First-Person Shooter โดยจะล็อกเมาส์ไว้กับหน้าจอและใช้การเคลื่อนไหวของเมาส์เพื่อหมุนกล้อง

### การทำงาน
- เมื่อ lock เมาส์แล้ว cursor จะหายไป
- การเคลื่อนเมาส์จะหมุนกล้อง 360 องศา
- กด Escape เพื่อ unlock

```javascript
import { PointerLockControls } from 'three/examples/jsm/controls/PointerLockControls.js';

const controls = new PointerLockControls(camera, document.body);

// เปิด Pointer Lock เมื่อคลิก
document.addEventListener('click', () => {
    controls.lock();
});

// Events
controls.addEventListener('lock', () => {
    console.log('Mouse locked! เกมเริ่มแล้ว');
    // ซ่อน overlay menu
    document.getElementById('menu').style.display = 'none';
});

controls.addEventListener('unlock', () => {
    console.log('Mouse unlocked! เกมหยุดชั่วคราว');
    // แสดง overlay menu
    document.getElementById('menu').style.display = 'flex';
});
```

### Demo สมบูรณ์: FPS Game Demo

```html
<!DOCTYPE html>
<html>
<head>
    <title>FPS Demo</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; font-family: monospace; }
        
        #crosshair {
            position: absolute;
            top: 50%; left: 50%;
            transform: translate(-50%, -50%);
            color: white;
            font-size: 20px;
            pointer-events: none;
            z-index: 100;
        }
        
        #menu {
            position: absolute;
            top: 0; left: 0;
            width: 100%; height: 100%;
            background: rgba(0, 0, 0, 0.8);
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            color: white;
            z-index: 200;
        }
        
        #menu h1 { font-size: 48px; margin-bottom: 20px; color: #e94560; }
        #menu p { font-size: 18px; margin-bottom: 10px; color: #aaa; }
        
        #startBtn {
            margin-top: 20px;
            padding: 15px 30px;
            font-size: 18px;
            background: #e94560;
            color: white;
            border: none;
            cursor: pointer;
            border-radius: 5px;
        }
        #startBtn:hover { background: #c73652; }
        
        #hud {
            position: absolute;
            bottom: 20px;
            left: 20px;
            color: white;
            font-size: 14px;
            display: none;
        }
        
        #score {
            position: absolute;
            top: 20px;
            right: 20px;
            color: #e94560;
            font-size: 24px;
            font-weight: bold;
            display: none;
        }
    </style>
</head>
<body>

<div id="menu">
    <h1>FPS THREE.JS</h1>
    <p>W/A/S/D - เดิน</p>
    <p>Space - กระโดด</p>
    <p>เมาส์ - มอง</p>
    <p>คลิก - ยิง</p>
    <button id="startBtn">เริ่มเกม</button>
</div>

<div id="crosshair">+</div>

<div id="hud">
    <p>WASD = เคลื่อนที่ | SPACE = กระโดด | ESC = หยุด</p>
</div>

<div id="score">คะแนน: <span id="scoreValue">0</span></div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { PointerLockControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/PointerLockControls.js';

// Scene
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x87ceeb);
scene.fog = new THREE.Fog(0x87ceeb, 10, 100);

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.y = 1.8; // ความสูงของตัวละคร

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);

// PointerLockControls
const controls = new PointerLockControls(camera, document.body);

document.getElementById('startBtn').addEventListener('click', () => {
    controls.lock();
});

controls.addEventListener('lock', () => {
    document.getElementById('menu').style.display = 'none';
    document.getElementById('hud').style.display = 'block';
    document.getElementById('score').style.display = 'block';
});

controls.addEventListener('unlock', () => {
    document.getElementById('menu').style.display = 'flex';
    document.getElementById('hud').style.display = 'none';
});

// Physics variables
const velocity = new THREE.Vector3();
const direction = new THREE.Vector3();
let canJump = false;
let isOnGround = true;
const GRAVITY = -30;
const JUMP_FORCE = 10;
const MOVE_SPEED = 15;

// Keys
const keys = {
    w: false, a: false, s: false, d: false, space: false
};

document.addEventListener('keydown', (e) => {
    switch(e.code) {
        case 'KeyW': keys.w = true; break;
        case 'KeyA': keys.a = true; break;
        case 'KeyS': keys.s = true; break;
        case 'KeyD': keys.d = true; break;
        case 'Space': 
            if (canJump) {
                velocity.y += JUMP_FORCE;
                canJump = false;
                isOnGround = false;
            }
            break;
    }
});

document.addEventListener('keyup', (e) => {
    switch(e.code) {
        case 'KeyW': keys.w = false; break;
        case 'KeyA': keys.a = false; break;
        case 'KeyS': keys.s = false; break;
        case 'KeyD': keys.d = false; break;
    }
});

// Lighting
const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
scene.add(ambientLight);

const sunLight = new THREE.DirectionalLight(0xffffff, 1);
sunLight.position.set(50, 100, 50);
sunLight.castShadow = true;
sunLight.shadow.mapSize.width = 2048;
sunLight.shadow.mapSize.height = 2048;
sunLight.shadow.camera.near = 0.5;
sunLight.shadow.camera.far = 500;
sunLight.shadow.camera.left = -100;
sunLight.shadow.camera.right = 100;
sunLight.shadow.camera.top = 100;
sunLight.shadow.camera.bottom = -100;
scene.add(sunLight);

// พื้น
const groundGeo = new THREE.PlaneGeometry(200, 200);
const groundMat = new THREE.MeshLambertMaterial({ color: 0x4a7c4f });
const ground = new THREE.Mesh(groundGeo, groundMat);
ground.rotation.x = -Math.PI / 2;
ground.receiveShadow = true;
scene.add(ground);

// สร้างเป้า (targets)
const targets = [];
let score = 0;

function createTarget() {
    const geo = new THREE.SphereGeometry(0.5, 16, 16);
    const mat = new THREE.MeshLambertMaterial({ color: 0xe94560 });
    const target = new THREE.Mesh(geo, mat);
    target.position.set(
        (Math.random() - 0.5) * 40,
        Math.random() * 5 + 1,
        (Math.random() - 0.5) * 40 - 10
    );
    target.castShadow = true;
    scene.add(target);
    targets.push(target);
    return target;
}

// สร้างเป้าเริ่มต้น
for (let i = 0; i < 15; i++) {
    createTarget();
}

// สร้างกำแพงและสิ่งกีดขวาง
function createBox(w, h, d, x, y, z, color = 0x8B4513) {
    const geo = new THREE.BoxGeometry(w, h, d);
    const mat = new THREE.MeshLambertMaterial({ color });
    const box = new THREE.Mesh(geo, mat);
    box.position.set(x, y, z);
    box.castShadow = true;
    box.receiveShadow = true;
    scene.add(box);
    return box;
}

createBox(10, 4, 1, 0, 2, -15, 0x8B6914);
createBox(10, 4, 1, -15, 2, -10, 0x6B4914);
createBox(10, 4, 1, 15, 2, -10, 0x7B5914);
createBox(1, 4, 10, -5, 2, -5, 0x8B7914);
createBox(1, 4, 10, 5, 2, -5, 0x9B8914);

// Shooting
const raycaster = new THREE.Raycaster();
const center = new THREE.Vector2(0, 0);

document.addEventListener('click', () => {
    if (!controls.isLocked) return;
    
    raycaster.setFromCamera(center, camera);
    const intersects = raycaster.intersectObjects(targets);
    
    if (intersects.length > 0) {
        const hit = intersects[0].object;
        const index = targets.indexOf(hit);
        if (index !== -1) {
            // ผล hit
            scene.remove(hit);
            targets.splice(index, 1);
            score += 10;
            document.getElementById('scoreValue').textContent = score;
            
            // flash effect
            hit.material.color.setHex(0xffff00);
            
            // สร้างเป้าใหม่
            setTimeout(() => createTarget(), 1000);
        }
    }
    
    // Muzzle flash effect - short flash
    const muzzleLight = new THREE.PointLight(0xffaa00, 5, 5);
    camera.add(muzzleLight);
    muzzleLight.position.set(0, 0, -1);
    scene.add(camera);
    setTimeout(() => { camera.remove(muzzleLight); }, 100);
});

// Resize
window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});

// Animation
const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    const delta = clock.getDelta();
    
    if (controls.isLocked) {
        // Gravity
        velocity.y += GRAVITY * delta;
        
        // Movement direction
        direction.z = Number(keys.w) - Number(keys.s);
        direction.x = Number(keys.d) - Number(keys.a);
        direction.normalize();
        
        if (keys.w || keys.s) {
            velocity.z -= direction.z * MOVE_SPEED * delta;
        }
        if (keys.a || keys.d) {
            velocity.x -= direction.x * MOVE_SPEED * delta;
        }
        
        // Damping
        velocity.x -= velocity.x * 10 * delta;
        velocity.z -= velocity.z * 10 * delta;
        
        controls.moveRight(-velocity.x * delta);
        controls.moveForward(-velocity.z * delta);
        camera.position.y += velocity.y * delta;
        
        // Ground check
        if (camera.position.y < 1.8) {
            velocity.y = 0;
            camera.position.y = 1.8;
            canJump = true;
        }
        
        // หมุนเป้า
        targets.forEach((target, i) => {
            target.rotation.y += 0.02;
            target.position.y = Math.sin(Date.now() * 0.001 + i) * 0.5 + 2;
        });
    }
    
    renderer.render(scene, camera);
}
animate();
</script>
</body>
</html>
```

---

## 4. TransformControls

TransformControls ใช้สำหรับจัดการวัตถุในพื้นที่ 3D คล้ายกับ Blender หรือ Unity Editor สามารถ:
- **Translate (T)**: ย้ายวัตถุ
- **Rotate (R)**: หมุนวัตถุ
- **Scale (S)**: ขนาดวัตถุ

```javascript
import { TransformControls } from 'three/examples/jsm/controls/TransformControls.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

const orbitControls = new OrbitControls(camera, renderer.domElement);
const transformControls = new TransformControls(camera, renderer.domElement);

// สำคัญ: ปิด OrbitControls เมื่อใช้ TransformControls
transformControls.addEventListener('dragging-changed', (event) => {
    orbitControls.enabled = !event.value;
});

// กำหนด mode
transformControls.setMode('translate');  // 'translate', 'rotate', 'scale'
transformControls.setSpace('world');     // 'world', 'local'
transformControls.setSize(1.0);         // ขนาดของ gizmo

// แนบกับวัตถุ
const box = new THREE.Mesh(
    new THREE.BoxGeometry(2, 2, 2),
    new THREE.MeshStandardMaterial({ color: 0x00ff88 })
);
scene.add(box);
transformControls.attach(box);
scene.add(transformControls);

// Keyboard shortcuts
window.addEventListener('keydown', (e) => {
    switch(e.key) {
        case 't': transformControls.setMode('translate'); break;
        case 'r': transformControls.setMode('rotate'); break;
        case 's': transformControls.setMode('scale'); break;
        case 'Escape': transformControls.detach(); break;
    }
});

// Events
transformControls.addEventListener('change', () => {
    console.log('Object changed:', box.position, box.rotation, box.scale);
    renderer.render(scene, camera);
});

transformControls.addEventListener('objectChange', () => {
    // เรียกทุกครั้งที่วัตถุเปลี่ยน
    updateUI();
});
```

---

## 5. TrackballControls

TrackballControls คล้ายกับ OrbitControls แต่ไม่มี "pole" ทำให้สามารถหมุนได้อิสระ 360 องศาในทุกทิศทาง

```javascript
import { TrackballControls } from 'three/examples/jsm/controls/TrackballControls.js';

const controls = new TrackballControls(camera, renderer.domElement);

controls.rotateSpeed = 1.0;        // ความเร็วหมุน
controls.zoomSpeed = 1.2;          // ความเร็วซูม
controls.panSpeed = 0.8;           // ความเร็วเลื่อน
controls.noZoom = false;           // ปิดซูม?
controls.noPan = false;            // ปิดเลื่อน?
controls.staticMoving = true;      // true = หยุดทันที, false = มี damping
controls.dynamicDampingFactor = 0.3;  // ค่า damping (ถ้า staticMoving = false)

// ต้องเรียก update ใน animation loop
function animate() {
    requestAnimationFrame(animate);
    controls.update();
    renderer.render(scene, camera);
}
```

---

## 6. ArcballControls

ArcballControls ใช้เทคนิค Arcball สำหรับการหมุน ให้ความรู้สึกเป็นธรรมชาติมากกว่า ออกแบบมาสำหรับการดูโมเดล 3D

```javascript
import { ArcballControls } from 'three/examples/jsm/controls/ArcballControls.js';

const controls = new ArcballControls(camera, renderer.domElement, scene);

controls.setGizmosVisible(true);    // แสดง gizmos
controls.dampingFactor = 25;        // ค่า damping
controls.maxDistance = 100;
controls.minDistance = 1;

// ต้องเรียก update ใน loop
function animate() {
    requestAnimationFrame(animate);
    controls.update();
    renderer.render(scene, camera);
}
```

---

## 7. การเลือก Controls ที่เหมาะสม

| Use Case | Controls ที่แนะนำ |
|----------|-----------------|
| การดูโมเดล 3D ทั่วไป | OrbitControls |
| เกม FPS | PointerLockControls |
| สำรวจพื้นที่ขนาดใหญ่ | FlyControls |
| Editor / Tool | TransformControls + OrbitControls |
| Simulation / Scientific | TrackballControls |
| Product Viewer | ArcballControls |

---

## 8. การรวม OrbitControls กับ TransformControls

ตัวอย่างการสร้าง 3D Editor อย่างง่าย

```javascript
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
import { TransformControls } from 'three/examples/jsm/controls/TransformControls.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(5, 5, 5);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// Orbit Controls สำหรับ navigate
const orbit = new OrbitControls(camera, renderer.domElement);
orbit.enableDamping = true;
orbit.dampingFactor = 0.05;

// Transform Controls สำหรับ edit
const transform = new TransformControls(camera, renderer.domElement);
scene.add(transform);

// ปิด orbit เมื่อกำลัง drag transform
transform.addEventListener('dragging-changed', (e) => {
    orbit.enabled = !e.value;
});

// สร้างวัตถุ
const objects = [];
function addObject(type) {
    let geo;
    switch(type) {
        case 'box': geo = new THREE.BoxGeometry(1, 1, 1); break;
        case 'sphere': geo = new THREE.SphereGeometry(0.5, 32, 32); break;
        case 'cylinder': geo = new THREE.CylinderGeometry(0.5, 0.5, 1, 32); break;
    }
    const mat = new THREE.MeshStandardMaterial({ 
        color: Math.random() * 0xffffff 
    });
    const mesh = new THREE.Mesh(geo, mat);
    mesh.position.set(
        (Math.random() - 0.5) * 5,
        0.5,
        (Math.random() - 0.5) * 5
    );
    scene.add(mesh);
    objects.push(mesh);
    return mesh;
}

// Raycaster สำหรับ select
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();
let selectedObject = null;

renderer.domElement.addEventListener('click', (e) => {
    if (transform.dragging) return;
    
    mouse.x = (e.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(e.clientY / window.innerHeight) * 2 + 1;
    
    raycaster.setFromCamera(mouse, camera);
    const hits = raycaster.intersectObjects(objects);
    
    if (hits.length > 0) {
        selectedObject = hits[0].object;
        transform.attach(selectedObject);
        // Highlight selected
        objects.forEach(o => o.material.emissive.setHex(0x000000));
        selectedObject.material.emissive.setHex(0x333333);
    } else {
        if (selectedObject) {
            selectedObject.material.emissive.setHex(0x000000);
            selectedObject = null;
            transform.detach();
        }
    }
});

// Keyboard shortcuts
window.addEventListener('keydown', (e) => {
    switch(e.key.toLowerCase()) {
        case 'g': transform.setMode('translate'); break;
        case 'r': transform.setMode('rotate'); break;
        case 's': transform.setMode('scale'); break;
        case 'delete':
            if (selectedObject) {
                scene.remove(selectedObject);
                objects.splice(objects.indexOf(selectedObject), 1);
                transform.detach();
                selectedObject = null;
            }
            break;
    }
});

// Lighting
scene.add(new THREE.AmbientLight(0xffffff, 0.6));
const dirLight = new THREE.DirectionalLight(0xffffff, 1);
dirLight.position.set(5, 10, 5);
scene.add(dirLight);

// Grid
scene.add(new THREE.GridHelper(20, 20));
scene.add(new THREE.AxesHelper(5));

// สร้างวัตถุตัวอย่าง
addObject('box');
addObject('sphere');
addObject('cylinder');

// Animation Loop
const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    orbit.update();
    renderer.render(scene, camera);
}
animate();
```

---

## แบบฝึกหัด

1. **ปรับแต่ง OrbitControls**: สร้าง scene ที่มีวัตถุหมุนอัตโนมัติและจำกัดมุมการมอง
2. **FPS Mini-game**: เพิ่มระบบนับคะแนนและ timer ให้กับ FPS Demo
3. **3D Object Editor**: พัฒนา editor ที่สามารถเพิ่มวัตถุ เลือก และย้ายได้
4. **Camera Switch**: สร้างปุ่มสลับระหว่าง OrbitControls และ FlyControls
5. **Smooth Target**: ทำให้ OrbitControls.target เลื่อนไปยังตำแหน่งใหม่อย่างนุ่มนวล

---

## สรุป

- **OrbitControls** เป็นตัวเลือกหลักสำหรับการดูวัตถุ 3D
- **PointerLockControls** เหมาะสำหรับเกม FPS
- **TransformControls** ใช้คู่กับ OrbitControls สำหรับ Editor
- ทุก Controls ที่ใช้ delta time ต้องส่ง `delta` เข้าไปใน `update()`
- การรวม Controls หลายตัวต้องระวังการ conflict
