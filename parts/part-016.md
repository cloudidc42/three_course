# Part 16: Shadows ใน Three.js

## บทนำ

เงา (Shadows) คือองค์ประกอบสำคัญที่ทำให้ scene 3D ดูสมจริง Three.js รองรับ shadow types หลายแบบ แต่ละแบบมีความสมดุลระหว่างคุณภาพและ performance ต่างกัน

## 1. การเปิดใช้ Shadow

การใช้ shadow ต้องเปิดใช้งานหลายส่วน:

```javascript
// 1. เปิด shadow map บน renderer
renderer.shadowMap.enabled = true;

// 2. กำหนดประเภท shadow map
renderer.shadowMap.type = THREE.PCFSoftShadowMap; // ค่าเริ่มต้นที่แนะนำ

// 3. Light ต้อง castShadow
directionalLight.castShadow = true;
spotLight.castShadow = true;
pointLight.castShadow = true;
// (AmbientLight ไม่มี shadow)

// 4. Object ที่ทอดเงา
mesh.castShadow = true;

// 5. Object ที่รับเงา
floor.receiveShadow = true;
wall.receiveShadow = true;
```

---

## 2. Shadow Map Types

```javascript
// 1. BasicShadowMap - เร็วที่สุด คุณภาพต่ำสุด (ขอบแข็ง)
renderer.shadowMap.type = THREE.BasicShadowMap;

// 2. PCFShadowMap - ค่าเริ่มต้น, ขอบนุ่มขึ้น
renderer.shadowMap.type = THREE.PCFShadowMap;

// 3. PCFSoftShadowMap - นุ่มที่สุดใน PCF family (แนะนำ)
renderer.shadowMap.type = THREE.PCFSoftShadowMap;

// 4. VSMShadowMap - Variance Shadow Map, เร็วกว่า PCF
// แต่อาจมีปัญหา light bleeding
renderer.shadowMap.type = THREE.VSMShadowMap;
```

### เปรียบเทียบ Shadow Types

| Type | คุณภาพ | Performance | ปัญหา |
|------|--------|-------------|-------|
| BasicShadowMap | ต่ำ | ดีที่สุด | ขอบแข็ง, aliasing |
| PCFShadowMap | ปานกลาง | ดี | ขอบนุ่มเล็กน้อย |
| PCFSoftShadowMap | ดี | ปานกลาง | เบลอมากขึ้น |
| VSMShadowMap | ดี | ปานกลาง | Light bleeding |

---

## 3. Shadow Camera Setup

เงาถูกสร้างโดย "shadow camera" ที่ซ่อนอยู่ใน Light ต้องตั้งค่าให้ครอบคลุม area ที่ต้องการ

### DirectionalLight Shadow Camera

```javascript
const dirLight = new THREE.DirectionalLight(0xffffff, 1);
dirLight.position.set(10, 20, 10);
dirLight.castShadow = true;

// shadow camera เป็น OrthographicCamera
const shadowCamera = dirLight.shadow.camera;
shadowCamera.near = 0.5;     // ใกล้สุด
shadowCamera.far = 500;      // ไกลสุด
shadowCamera.left = -50;     // ขอบซ้าย
shadowCamera.right = 50;     // ขอบขวา
shadowCamera.top = 50;       // ขอบบน
shadowCamera.bottom = -50;   // ขอบล่าง

// Shadow map resolution (กำลัง 2 เสมอ)
dirLight.shadow.mapSize.width = 2048;   // Default = 512
dirLight.shadow.mapSize.height = 2048;

// Shadow bias - แก้ปัญหา shadow acne
dirLight.shadow.bias = -0.0001;  // ค่า negative เล็กน้อย
dirLight.shadow.normalBias = 0.02;

// Shadow radius (ความนุ่มของขอบ PCF)
dirLight.shadow.radius = 4;  // สำหรับ PCFSoftShadowMap

scene.add(dirLight);

// CameraHelper - แสดงขอบเขต shadow camera (debug)
const shadowCameraHelper = new THREE.CameraHelper(dirLight.shadow.camera);
scene.add(shadowCameraHelper);
```

### SpotLight Shadow Camera

```javascript
const spotLight = new THREE.SpotLight(0xffffff, 1);
spotLight.position.set(5, 10, 5);
spotLight.castShadow = true;
spotLight.angle = Math.PI / 6;  // 30 องศา
spotLight.penumbra = 0.2;        // ขอบนุ่ม

// shadow camera เป็น PerspectiveCamera
spotLight.shadow.camera.near = 0.5;
spotLight.shadow.camera.far = 50;
// FOV = angle * 2 (ตั้งอัตโนมัติ)
spotLight.shadow.camera.fov = 30; // ตรงกับ angle

spotLight.shadow.mapSize.width = 1024;
spotLight.shadow.mapSize.height = 1024;
spotLight.shadow.bias = -0.001;

scene.add(spotLight);

// Target
spotLight.target.position.set(0, 0, 0);
scene.add(spotLight.target);
```

### PointLight Shadow Camera

```javascript
const pointLight = new THREE.PointLight(0xffffff, 1, 100);
pointLight.position.set(0, 5, 0);
pointLight.castShadow = true;

// PointLight ใช้ Cube Map (6 cameras รอบทิศ)
// จึงหนักกว่าประเภทอื่น
pointLight.shadow.camera.near = 0.5;
pointLight.shadow.camera.far = 100;
pointLight.shadow.mapSize.width = 1024;
pointLight.shadow.mapSize.height = 1024;
pointLight.shadow.bias = -0.005;

scene.add(pointLight);
```

---

## 4. Shadow Bias และ Shadow Acne

```javascript
// Shadow Acne = ลายหยักๆ บนพื้นผิวที่รับเงา เกิดจาก self-shadowing
// แก้ด้วย bias (ค่า negative เล็กน้อย)

// ✅ การตั้งค่าที่ดี
dirLight.shadow.bias = -0.0001;      // สำหรับ DirectionalLight
spotLight.shadow.bias = -0.001;      // สำหรับ SpotLight  
pointLight.shadow.bias = -0.005;     // สำหรับ PointLight (ต้องมากกว่า)

// normalBias - เลื่อน shadow ตาม normal (แก้ปัญหากับ curved surfaces)
dirLight.shadow.normalBias = 0.02;

// ⚠️ ถ้า bias มากเกินไป Peter Panning เกิด (เงาลอยออกจากวัตถุ)
// หา balance ที่ดีระหว่าง acne และ peter panning
```

---

## 5. Demo สมบูรณ์: Shadow Showcase

```html
<!DOCTYPE html>
<html>
<head>
    <title>Shadow Types Demo</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #111; overflow: hidden; font-family: monospace; }
        
        #panel {
            position: absolute;
            top: 10px;
            right: 10px;
            background: rgba(0,0,0,0.85);
            color: white;
            padding: 15px;
            border-radius: 8px;
            min-width: 250px;
            font-size: 13px;
        }
        
        h3 { color: #e94560; margin-bottom: 12px; }
        h4 { color: #4ecdc4; margin: 10px 0 5px; }
        
        .row { display: flex; align-items: center; margin: 5px 0; gap: 8px; }
        .row label { width: 120px; color: #aaa; }
        input[type=range] { width: 100px; }
        span.val { color: #ffd700; width: 50px; }
        
        select {
            background: #333;
            color: white;
            border: 1px solid #555;
            padding: 3px 6px;
            border-radius: 3px;
        }
        
        .btn-row { display: flex; gap: 5px; flex-wrap: wrap; margin: 5px 0; }
        button {
            padding: 5px 10px;
            background: #e94560;
            color: white;
            border: none;
            border-radius: 3px;
            cursor: pointer;
            font-size: 11px;
        }
        button:hover { background: #c73652; }
        button.active { background: #4ecdc4; color: #000; }
    </style>
</head>
<body>

<div id="panel">
    <h3>Shadow Controls</h3>
    
    <h4>Shadow Type</h4>
    <div class="btn-row">
        <button id="btnBasic">Basic</button>
        <button id="btnPCF">PCF</button>
        <button id="btnPCFSoft" class="active">PCFSoft</button>
        <button id="btnVSM">VSM</button>
    </div>
    
    <h4>Map Size</h4>
    <div class="btn-row">
        <button id="btn256">256</button>
        <button id="btn512">512</button>
        <button id="btn1024" class="active">1024</button>
        <button id="btn2048">2048</button>
    </div>
    
    <h4>Light Settings</h4>
    <div class="row">
        <label>Bias:</label>
        <input type="range" id="bias" min="-0.01" max="0.01" step="0.0001" value="-0.0001">
        <span class="val" id="biasVal">-0.0001</span>
    </div>
    <div class="row">
        <label>Radius:</label>
        <input type="range" id="radius" min="0" max="20" step="1" value="4">
        <span class="val" id="radiusVal">4</span>
    </div>
    <div class="row">
        <label>Light X:</label>
        <input type="range" id="lightX" min="-20" max="20" step="0.5" value="10">
        <span class="val" id="lightXVal">10</span>
    </div>
    <div class="row">
        <label>Light Y:</label>
        <input type="range" id="lightY" min="1" max="40" step="0.5" value="20">
        <span class="val" id="lightYVal">20</span>
    </div>
    
    <h4>Options</h4>
    <div class="row">
        <label>Show Camera:</label>
        <input type="checkbox" id="showCamera">
    </div>
    <div class="row">
        <label>Animate Light:</label>
        <input type="checkbox" id="animLight" checked>
    </div>
    <div class="row">
        <label>Auto-size:</label>
        <input type="checkbox" id="autoSize" checked>
    </div>
</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';

// Scene
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111122);

const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 500);
camera.position.set(20, 20, 30);
camera.lookAt(0, 0, 0);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

// Ambient light
const ambientLight = new THREE.AmbientLight(0x334466, 0.5);
scene.add(ambientLight);

// Directional Light
const dirLight = new THREE.DirectionalLight(0xffffff, 1.5);
dirLight.position.set(10, 20, 10);
dirLight.castShadow = true;
dirLight.shadow.mapSize.width = 1024;
dirLight.shadow.mapSize.height = 1024;
dirLight.shadow.camera.near = 0.5;
dirLight.shadow.camera.far = 200;
dirLight.shadow.camera.left = -30;
dirLight.shadow.camera.right = 30;
dirLight.shadow.camera.top = 30;
dirLight.shadow.camera.bottom = -30;
dirLight.shadow.bias = -0.0001;
dirLight.shadow.radius = 4;
scene.add(dirLight);

// Light helper
const lightHelper = new THREE.DirectionalLightHelper(dirLight, 3);
scene.add(lightHelper);

// Shadow camera helper
const shadowCamHelper = new THREE.CameraHelper(dirLight.shadow.camera);
shadowCamHelper.visible = false;
scene.add(shadowCamHelper);

// พื้น
const floorGeo = new THREE.PlaneGeometry(60, 60);
const floorMat = new THREE.MeshStandardMaterial({ 
    color: 0x223344, 
    roughness: 0.9,
    metalness: 0.1
});
const floor = new THREE.Mesh(floorGeo, floorMat);
floor.rotation.x = -Math.PI / 2;
floor.receiveShadow = true;
scene.add(floor);

// Grid
scene.add(new THREE.GridHelper(60, 30, 0x334466, 0x223355));

// สร้างวัตถุต่างๆ
function createMesh(geo, color, x, y, z) {
    const mat = new THREE.MeshStandardMaterial({ 
        color, roughness: 0.4, metalness: 0.4 
    });
    const mesh = new THREE.Mesh(geo, mat);
    mesh.position.set(x, y, z);
    mesh.castShadow = true;
    mesh.receiveShadow = true;
    scene.add(mesh);
    return mesh;
}

// กล่อง
const box1 = createMesh(new THREE.BoxGeometry(3, 3, 3), 0xe94560, -8, 1.5, -5);
const box2 = createMesh(new THREE.BoxGeometry(2, 5, 2), 0x4ecdc4, 0, 2.5, -5);
const box3 = createMesh(new THREE.BoxGeometry(4, 2, 4), 0xffd700, 8, 1, -5);

// ทรงกลม
const sphere1 = createMesh(new THREE.SphereGeometry(1.5, 32, 32), 0xe94560, -10, 1.5, 5);
const sphere2 = createMesh(new THREE.SphereGeometry(2, 32, 32), 0x96ceb4, 0, 2, 5);
const sphere3 = createMesh(new THREE.SphereGeometry(1, 32, 32), 0xff6b6b, 10, 1, 5);

// กำแพง
const wall1 = createMesh(new THREE.BoxGeometry(20, 6, 0.5), 0x556677, 0, 3, -12);
const wall2 = createMesh(new THREE.BoxGeometry(0.5, 6, 20), 0x556677, -12, 3, 0);

// หลังคาบางส่วน
const roof = createMesh(new THREE.BoxGeometry(10, 0.3, 10), 0x667788, 0, 6.5, 0);

// GUI Controls
function updateShadowType(type) {
    renderer.shadowMap.type = type;
    // ต้อง refresh materials
    scene.traverse(obj => {
        if (obj.isMesh) {
            obj.material.needsUpdate = true;
        }
    });
    
    document.querySelectorAll('.btn-row button[id^="btn"]').forEach(btn => btn.classList.remove('active'));
}

document.getElementById('btnBasic').addEventListener('click', () => {
    updateShadowType(THREE.BasicShadowMap);
    document.getElementById('btnBasic').classList.add('active');
});
document.getElementById('btnPCF').addEventListener('click', () => {
    updateShadowType(THREE.PCFShadowMap);
    document.getElementById('btnPCF').classList.add('active');
});
document.getElementById('btnPCFSoft').addEventListener('click', () => {
    updateShadowType(THREE.PCFSoftShadowMap);
    document.getElementById('btnPCFSoft').classList.add('active');
});
document.getElementById('btnVSM').addEventListener('click', () => {
    updateShadowType(THREE.VSMShadowMap);
    document.getElementById('btnVSM').classList.add('active');
});

function updateMapSize(size) {
    dirLight.shadow.mapSize.width = size;
    dirLight.shadow.mapSize.height = size;
    dirLight.shadow.map = null; // force recreate
    document.querySelectorAll('#panel .btn-row:nth-of-type(2) button').forEach(btn => btn.classList.remove('active'));
}

document.getElementById('btn256').addEventListener('click', () => {
    updateMapSize(256); document.getElementById('btn256').classList.add('active');
});
document.getElementById('btn512').addEventListener('click', () => {
    updateMapSize(512); document.getElementById('btn512').classList.add('active');
});
document.getElementById('btn1024').addEventListener('click', () => {
    updateMapSize(1024); document.getElementById('btn1024').classList.add('active');
});
document.getElementById('btn2048').addEventListener('click', () => {
    updateMapSize(2048); document.getElementById('btn2048').classList.add('active');
});

// Sliders
document.getElementById('bias').addEventListener('input', (e) => {
    dirLight.shadow.bias = parseFloat(e.target.value);
    document.getElementById('biasVal').textContent = e.target.value;
});

document.getElementById('radius').addEventListener('input', (e) => {
    dirLight.shadow.radius = parseFloat(e.target.value);
    document.getElementById('radiusVal').textContent = e.target.value;
});

document.getElementById('lightX').addEventListener('input', (e) => {
    dirLight.position.x = parseFloat(e.target.value);
    document.getElementById('lightXVal').textContent = e.target.value;
});

document.getElementById('lightY').addEventListener('input', (e) => {
    dirLight.position.y = parseFloat(e.target.value);
    document.getElementById('lightYVal').textContent = e.target.value;
});

document.getElementById('showCamera').addEventListener('change', (e) => {
    shadowCamHelper.visible = e.target.checked;
});

// Resize
window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});

// Animation
const clock = new THREE.Clock();
let angle = 0;

function animate() {
    requestAnimationFrame(animate);
    const delta = clock.getDelta();
    
    const animLight = document.getElementById('animLight').checked;
    if (animLight) {
        angle += delta * 0.5;
        const radius = 20;
        dirLight.position.x = Math.sin(angle) * radius;
        dirLight.position.z = Math.cos(angle) * radius;
        dirLight.position.y = 20;
        
        document.getElementById('lightX').value = dirLight.position.x;
        document.getElementById('lightXVal').textContent = dirLight.position.x.toFixed(1);
    }
    
    // หมุนวัตถุ
    box1.rotation.y += 0.01;
    sphere2.rotation.y += 0.015;
    
    // อัพเดต helpers
    lightHelper.update();
    shadowCamHelper.update();
    
    controls.update();
    renderer.render(scene, camera);
}
animate();
</script>
</body>
</html>
```

---

## 6. Shadow Performance Tips

```javascript
// ✅ ใช้ mapSize ที่เหมาะสม
// เล็กก็พอถ้าไม่ต้องการ pixel perfect
dirLight.shadow.mapSize.width = 1024;  // มักพอสำหรับ outdoor
dirLight.shadow.mapSize.height = 1024;

// 2048 หรือ 4096 สำหรับ close-up interior

// ✅ จำกัด shadow camera frustum ให้แน่น
// ยิ่งแน่น = resolution ดีขึ้น
function fitShadowCameraToScene(light, objects) {
    const box = new THREE.Box3();
    objects.forEach(obj => {
        if (obj.castShadow || obj.receiveShadow) {
            box.expandByObject(obj);
        }
    });
    
    const size = new THREE.Vector3();
    box.getSize(size);
    const maxDim = Math.max(size.x, size.z);
    
    light.shadow.camera.left = -maxDim / 2;
    light.shadow.camera.right = maxDim / 2;
    light.shadow.camera.top = maxDim / 2;
    light.shadow.camera.bottom = -maxDim / 2;
    light.shadow.camera.updateProjectionMatrix();
}

// ✅ ปิด shadow ของวัตถุที่ไม่จำเป็น
smallDetail.castShadow = false;  // วัตถุเล็กไม่ต้องทอดเงา

// ✅ ใช้ castShadow เฉพาะสิ่งจำเป็น
// ไม่ต้องเปิดทุก mesh
importedModel.traverse(obj => {
    if (obj.isMesh) {
        obj.castShadow = true;
        obj.receiveShadow = true;
    }
});

// ✅ ลดจำนวน Shadow Lights
// PointLight shadow หนักมาก (6 render passes)
// ใช้ DirectionalLight หรือ SpotLight แทน

// ✅ Shadow frustum culling
dirLight.shadow.camera.near = 1;    // อย่าตั้งน้อยเกิน
dirLight.shadow.camera.far = 100;   // อย่าตั้งมากเกิน
```

---

## 7. Indoor Scene with Multiple Shadows

```javascript
// Scene สำหรับห้องในบ้าน
function createIndoorScene() {
    // พื้น
    const floor = new THREE.Mesh(
        new THREE.PlaneGeometry(20, 15),
        new THREE.MeshStandardMaterial({ color: 0x8B6914, roughness: 0.9 })
    );
    floor.rotation.x = -Math.PI / 2;
    floor.receiveShadow = true;
    scene.add(floor);
    
    // เพดาน
    const ceiling = new THREE.Mesh(
        new THREE.PlaneGeometry(20, 15),
        new THREE.MeshStandardMaterial({ color: 0xffffff })
    );
    ceiling.rotation.x = Math.PI / 2;
    ceiling.position.y = 5;
    scene.add(ceiling);
    
    // กำแพง
    const wallMat = new THREE.MeshStandardMaterial({ color: 0xffe0cc });
    const walls = [
        { geo: new THREE.BoxGeometry(20, 5, 0.2), pos: [0, 2.5, -7.5] },
        { geo: new THREE.BoxGeometry(20, 5, 0.2), pos: [0, 2.5, 7.5] },
        { geo: new THREE.BoxGeometry(0.2, 5, 15), pos: [-10, 2.5, 0] },
        { geo: new THREE.BoxGeometry(0.2, 5, 15), pos: [10, 2.5, 0] },
    ];
    
    walls.forEach(w => {
        const wall = new THREE.Mesh(w.geo, wallMat);
        wall.position.set(...w.pos);
        wall.receiveShadow = true;
        wall.castShadow = true;
        scene.add(wall);
    });
    
    // โต๊ะ
    const table = new THREE.Group();
    const tableTop = new THREE.Mesh(
        new THREE.BoxGeometry(4, 0.2, 2),
        new THREE.MeshStandardMaterial({ color: 0x8B4513, roughness: 0.7 })
    );
    tableTop.position.y = 2;
    tableTop.castShadow = true;
    tableTop.receiveShadow = true;
    table.add(tableTop);
    
    // ขาโต๊ะ
    const legGeo = new THREE.BoxGeometry(0.1, 2, 0.1);
    const legMat = new THREE.MeshStandardMaterial({ color: 0x6B3410 });
    [[-1.8, 1, -0.8], [1.8, 1, -0.8], [-1.8, 1, 0.8], [1.8, 1, 0.8]].forEach(([x, y, z]) => {
        const leg = new THREE.Mesh(legGeo, legMat);
        leg.position.set(x, y, z);
        leg.castShadow = true;
        table.add(leg);
    });
    
    table.position.set(0, 0, -2);
    scene.add(table);
    
    // โคมไฟบนโต๊ะ (PointLight)
    const lampBase = new THREE.Mesh(
        new THREE.CylinderGeometry(0.05, 0.15, 0.5, 16),
        new THREE.MeshStandardMaterial({ color: 0x888888 })
    );
    lampBase.position.set(-2, 2.35, -2);
    lampBase.castShadow = true;
    scene.add(lampBase);
    
    const lampLight = new THREE.PointLight(0xffddaa, 1.5, 15);
    lampLight.position.set(-2, 3.5, -2);
    lampLight.castShadow = true;
    lampLight.shadow.mapSize.width = 512;
    lampLight.shadow.mapSize.height = 512;
    lampLight.shadow.bias = -0.005;
    scene.add(lampLight);
    
    // หน้าต่าง (directional light จากนอก)
    const windowLight = new THREE.DirectionalLight(0x88aaff, 0.8);
    windowLight.position.set(15, 8, 0);
    windowLight.target.position.set(0, 0, 0);
    windowLight.castShadow = true;
    windowLight.shadow.mapSize.width = 1024;
    windowLight.shadow.mapSize.height = 1024;
    windowLight.shadow.camera.near = 1;
    windowLight.shadow.camera.far = 50;
    windowLight.shadow.camera.left = -20;
    windowLight.shadow.camera.right = 20;
    windowLight.shadow.camera.top = 10;
    windowLight.shadow.camera.bottom = -10;
    windowLight.shadow.bias = -0.001;
    scene.add(windowLight);
    scene.add(windowLight.target);
}
```

---

## 8. Dynamic Shadows

```javascript
// เงาที่เปลี่ยนแปลงตามเวลา (เช่น วัฏจักรกลางวัน-กลางคืน)
class DayNightCycle {
    constructor(scene) {
        this.scene = scene;
        this.time = 0; // 0 = เที่ยงคืน, 0.5 = เที่ยงวัน
        
        this.sunLight = new THREE.DirectionalLight(0xfff5e0, 1.5);
        this.sunLight.castShadow = true;
        this.sunLight.shadow.mapSize.width = 2048;
        this.sunLight.shadow.mapSize.height = 2048;
        this.sunLight.shadow.camera.left = -50;
        this.sunLight.shadow.camera.right = 50;
        this.sunLight.shadow.camera.top = 50;
        this.sunLight.shadow.camera.bottom = -50;
        this.sunLight.shadow.bias = -0.0001;
        scene.add(this.sunLight);
        scene.add(this.sunLight.target);
        
        this.ambientLight = new THREE.AmbientLight(0x404060, 0.2);
        scene.add(this.ambientLight);
        
        this.moonLight = new THREE.DirectionalLight(0x4488ff, 0.1);
        scene.add(this.moonLight);
    }
    
    update(delta, speed = 0.01) {
        this.time += delta * speed;
        if (this.time > 1) this.time -= 1;
        
        const angle = this.time * Math.PI * 2;
        const radius = 100;
        
        // ตำแหน่งดวงอาทิตย์
        this.sunLight.position.set(
            Math.sin(angle) * radius,
            Math.cos(angle) * radius,
            0
        );
        
        // กลางวัน/กลางคืน
        const sunHeight = Math.cos(angle);
        const isDaytime = sunHeight > 0;
        
        // ปรับความเข้มของแสง
        const sunIntensity = Math.max(0, sunHeight) * 1.5;
        this.sunLight.intensity = sunIntensity;
        
        // ปรับสีแสงตามเวลา
        if (sunHeight > 0.2) {
            // กลางวัน - ขาว
            this.sunLight.color.setHex(0xfff5e0);
        } else if (sunHeight > 0) {
            // พระอาทิตย์ตก - ส้ม
            this.sunLight.color.setHex(0xff8844);
        } else {
            // กลางคืน
            this.sunLight.intensity = 0;
        }
        
        // แสงจันทร์
        this.moonLight.position.set(
            -Math.sin(angle) * radius,
            -Math.cos(angle) * radius,
            0
        );
        const moonHeight = -Math.cos(angle);
        this.moonLight.intensity = Math.max(0, moonHeight) * 0.15;
        
        // Ambient (ท้องฟ้า)
        const ambientIntensity = 0.1 + Math.max(0, sunHeight) * 0.4;
        this.ambientLight.intensity = ambientIntensity;
        this.ambientLight.color.lerpColors(
            new THREE.Color(0x080820),  // กลางคืน
            new THREE.Color(0x4060ff),  // กลางวัน
            Math.max(0, sunHeight)
        );
        
        // sky color
        const skyColor = new THREE.Color().lerpColors(
            new THREE.Color(0x000011),
            new THREE.Color(0x87ceeb),
            Math.max(0, sunHeight)
        );
        this.scene.background = skyColor;
    }
}
```

---

## แบบฝึกหัด

1. **Shadow Quality Compare**: สร้าง split-view แสดงเงา 4 ประเภทพร้อมกัน
2. **Shadow Cascade**: ใช้ shadow cameras หลายตัวที่ระยะต่างกัน
3. **Soft Shadows**: ทดลอง bias/radius ต่างๆ เพื่อได้เงาที่ดีที่สุด
4. **Day-Night Cycle**: พัฒนา DayNightCycle ให้สมบูรณ์พร้อม UI
5. **Indoor Lighting**: สร้างห้องที่มีหลาย light sources พร้อม shadows

---

## สรุป

- เปิด `renderer.shadowMap.enabled = true` และตั้ง `type`
- `PCFSoftShadowMap` เป็นตัวเลือกที่ดีสำหรับใช้งานทั่วไป
- Light, object ต้องตั้ง `castShadow = true`
- วัตถุที่รับเงาต้องตั้ง `receiveShadow = true`
- ตั้งค่า shadow camera ให้แน่นเพื่อ resolution ที่ดี
- `bias = -0.0001` แก้ Shadow Acne
- ใช้ `CameraHelper` เพื่อ debug shadow frustum
- `mapSize` ที่ใหญ่ = คุณภาพดีขึ้น แต่ใช้ memory มากขึ้น
