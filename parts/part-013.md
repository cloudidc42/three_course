# Part 13: Object3D & Transforms ใน Three.js

## บทนำ

`Object3D` เป็น base class ที่สำคัญที่สุดใน Three.js แทบทุกอย่างใน scene ล้วนสืบทอดจาก Object3D ไม่ว่าจะเป็น Mesh, Camera, Light, Group ทำความเข้าใจ Object3D จะช่วยให้ควบคุม 3D objects ได้อย่างมีประสิทธิภาพ

## 1. Object3D Class Hierarchy

```
Object3D
├── Camera
│   ├── PerspectiveCamera
│   └── OrthographicCamera
├── Light
│   ├── DirectionalLight
│   ├── PointLight
│   └── SpotLight
├── Mesh
├── Group
├── Line
├── Points
└── SkinnedMesh
```

ทุก Object3D มีคุณสมบัติพื้นฐานเหล่านี้:

```javascript
const obj = new THREE.Object3D();

// Transform properties
obj.position    // Vector3 - ตำแหน่ง
obj.rotation    // Euler - การหมุน
obj.quaternion  // Quaternion - การหมุนแบบ quaternion
obj.scale       // Vector3 - ขนาด
obj.matrix      // Matrix4 - local transform matrix
obj.matrixWorld // Matrix4 - world transform matrix

// Hierarchy
obj.parent      // Object3D parent
obj.children    // Array<Object3D> children

// Visibility
obj.visible     // boolean
obj.castShadow  // boolean
obj.receiveShadow // boolean

// Info
obj.name        // string
obj.id          // number (auto)
obj.uuid        // string (auto)
obj.userData    // {} - เก็บข้อมูลของเราเอง
```

---

## 2. Vector3 - การจัดการตำแหน่ง

`Vector3` ใช้แทนตำแหน่ง (x, y, z) และทิศทาง

### การสร้างและ basic operations

```javascript
// สร้าง Vector3
const v1 = new THREE.Vector3(1, 2, 3);
const v2 = new THREE.Vector3();    // (0, 0, 0)

// กำหนดค่า
v1.set(5, 10, 15);
v1.x = 5;
v1.y = 10;
v1.z = 15;
v1.setX(5);   // กำหนดแค่แกน x
v1.setY(10);  // กำหนดแค่แกน y

// Copy
v2.copy(v1);           // v2 = v1
const v3 = v1.clone(); // สร้างใหม่

// Arithmetic
v1.add(v2);            // v1 += v2
v1.sub(v2);            // v1 -= v2
v1.multiply(v2);       // v1 *= v2 (element-wise)
v1.multiplyScalar(2);  // v1 *= 2
v1.divideScalar(2);    // v1 /= 2
v1.negate();           // v1 = -v1

// ระวัง: methods ส่วนใหญ่ mutate object และ return this
// เพื่อให้ chain ได้
v1.set(1, 0, 0).multiplyScalar(5).add(v2);
```

### การคำนวณ

```javascript
const v1 = new THREE.Vector3(3, 0, 4);
const v2 = new THREE.Vector3(1, 0, 0);

// ความยาว
console.log(v1.length());           // 5 (sqrt(9+0+16))
console.log(v1.lengthSq());         // 25 (ไม่ sqrt, เร็วกว่า)
console.log(v1.manhattanLength());  // 7 (|3|+|0|+|4|)

// Normalize (ทำให้ length = 1)
const normalized = v1.clone().normalize();
console.log(normalized.length()); // 1

// Dot product (scalar)
const dot = v1.dot(v2); // 3*1 + 0*0 + 4*0 = 3

// Cross product (vector perpendicular to both)
const cross = new THREE.Vector3();
cross.crossVectors(v1, v2); // v1 × v2

// Distance
const dist = v1.distanceTo(v2);      // sqrt distance
const distSq = v1.distanceToSq(v2);  // squared distance (เร็วกว่า)

// Angle
const angle = v1.angleTo(v2);  // radians

// Lerp (Linear interpolation)
const start = new THREE.Vector3(0, 0, 0);
const end = new THREE.Vector3(10, 10, 10);
const middle = start.clone().lerp(end, 0.5); // ครึ่งทาง
// {5, 5, 5}

// ใช้ใน animation
function animate() {
    const delta = clock.getDelta();
    // เลื่อนไปหา target อย่างนุ่มนวล
    mesh.position.lerp(targetPosition, 5 * delta);
}

// Project / Unproject (world <-> screen)
// สำหรับ raycasting และ picking
const worldPos = new THREE.Vector3(5, 0, 0);
worldPos.project(camera); // แปลงเป็น screen space (-1 to 1)
worldPos.unproject(camera); // แปลงกลับเป็น world space
```

### Min, Max, Clamp

```javascript
const v = new THREE.Vector3(5, -3, 8);

// Clamp
v.clamp(
    new THREE.Vector3(-5, -5, -5),  // min
    new THREE.Vector3(5, 5, 5)       // max
); // ผล: (5, -3, 5)

v.clampScalar(-5, 5); // clamp ทุก component

// Min/Max
v.min(new THREE.Vector3(3, 3, 3));  // v = min(v, other) per component
v.max(new THREE.Vector3(0, 0, 0));  // v = max(v, other) per component
```

---

## 3. Euler - การหมุนแบบ Euler

Euler แทนการหมุนด้วยสามมุม: X, Y, Z (pitch, yaw, roll) และลำดับการหมุน (order)

```javascript
// สร้าง Euler
const euler = new THREE.Euler(0, Math.PI / 2, 0);        // หมุน 90° ตามแกน Y
const euler2 = new THREE.Euler(Math.PI / 4, 0, 0, 'XYZ'); // มีกำหนด order

// กำหนดค่า
mesh.rotation.set(Math.PI / 4, 0, 0); // set all at once
mesh.rotation.x = Math.PI / 4;         // หมุนรอบ X 45°
mesh.rotation.y += 0.01;               // หมุนรอบ Y ทีละน้อย

// Euler Order (สำคัญมาก!)
// XYZ: หมุน X ก่อน แล้ว Y แล้ว Z (default)
// YXZ: หมุน Y ก่อน แล้ว X แล้ว Z
// ZXY, ZYX, YZX, XZY

mesh.rotation.order = 'YXZ'; // เปลี่ยน order

// แปลงเป็น Quaternion
const quat = new THREE.Quaternion();
quat.setFromEuler(mesh.rotation);

// แปลงจาก Quaternion
mesh.rotation.setFromQuaternion(quat);
```

### ปัญหาของ Euler: Gimbal Lock

```javascript
// Gimbal Lock เกิดเมื่อแกนหมุนสองแกนตรงกัน
// ทำให้สูญเสีย degree of freedom หนึ่งองศา
// แก้โดยใช้ Quaternion

// ตัวอย่าง Gimbal Lock (order XYZ)
mesh.rotation.x = Math.PI / 2;  // หมุน X 90°
mesh.rotation.y = Math.PI / 4;  // หมุน Y 45°
mesh.rotation.z = Math.PI / 3;  // Z อาจเหมือน X หรือ -X
// ❌ เกิด Gimbal Lock!
```

---

## 4. Quaternion - การหมุนที่ถูกต้อง

Quaternion แก้ปัญหา Gimbal Lock และ interpolate ได้อย่างราบรื่นกว่า Euler

```javascript
// สร้าง Quaternion
const q = new THREE.Quaternion();           // identity (0, 0, 0, 1)
const q2 = new THREE.Quaternion(0, 0, 0, 1); // x, y, z, w

// กำหนดจาก Axis-Angle
// หมุนรอบแกน Y 90°
const axis = new THREE.Vector3(0, 1, 0); // normalized axis
const angle = Math.PI / 2;
q.setFromAxisAngle(axis, angle);

// กำหนดจาก Euler
q.setFromEuler(new THREE.Euler(Math.PI / 4, 0, 0));

// กำหนดจาก rotation matrix
const matrix = new THREE.Matrix4();
matrix.makeRotationY(Math.PI / 4);
q.setFromRotationMatrix(matrix);

// lookAt - หมุนให้หันไปทิศทางที่กำหนด
const from = new THREE.Vector3(0, 0, 0);
const to = new THREE.Vector3(5, 0, 0);
const direction = to.clone().sub(from).normalize();
q.setFromUnitVectors(new THREE.Vector3(0, 0, 1), direction);

// Apply Quaternion to object
mesh.quaternion.copy(q);
mesh.setRotationFromQuaternion(q); // เหมือนกัน

// Multiply Quaternions (รวม rotations)
const q1 = new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(0, 1, 0), Math.PI / 2);
const q3 = new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1, 0, 0), Math.PI / 4);
const combined = q1.clone().multiply(q3); // q1 then q3
mesh.quaternion.copy(combined);
```

### Slerp (Spherical Linear Interpolation)

```javascript
// Slerp คือการ interpolate ระหว่าง quaternion สองตัว
// ให้การหมุนที่ smooth และ consistent
const startQ = new THREE.Quaternion();
const endQ = new THREE.Quaternion().setFromAxisAngle(
    new THREE.Vector3(0, 1, 0), Math.PI
);

let t = 0;
function animate() {
    requestAnimationFrame(animate);
    const delta = clock.getDelta();
    
    t = Math.min(t + delta * 0.5, 1); // เพิ่ม t ขึ้นช้าๆ
    
    // Slerp ระหว่าง startQ และ endQ
    mesh.quaternion.slerpQuaternions(startQ, endQ, t);
    
    renderer.render(scene, camera);
}

// ใช้ slerp ใน place
mesh.quaternion.slerp(targetQuaternion, 0.1); // 10% toward target ต่อ frame
```

### Quaternion vs Euler Comparison Demo

```javascript
import * as THREE from 'three';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 5, 15);
camera.lookAt(0, 2, 0);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// สร้างวัตถุสองชิ้น
const geo = new THREE.BoxGeometry(2, 0.2, 2);

// วัตถุที่ใช้ Euler
const eulerMesh = new THREE.Mesh(
    geo,
    new THREE.MeshStandardMaterial({ color: 0xe94560 })
);
eulerMesh.position.set(-4, 2, 0);
scene.add(eulerMesh);

// วัตถุที่ใช้ Quaternion
const quatMesh = new THREE.Mesh(
    geo.clone(),
    new THREE.MeshStandardMaterial({ color: 0x4ecdc4 })
);
quatMesh.position.set(4, 2, 0);
scene.add(quatMesh);

// Labels
scene.add(new THREE.AmbientLight(0xffffff, 0.6));
scene.add(new THREE.DirectionalLight(0xffffff, 1).position.set(5, 10, 5) && new THREE.DirectionalLight(0xffffff, 1));

// Start/End rotations
const startEuler = new THREE.Euler(0, 0, 0);
const endEuler = new THREE.Euler(Math.PI, Math.PI, Math.PI);

const startQuat = new THREE.Quaternion().setFromEuler(startEuler);
const endQuat = new THREE.Quaternion().setFromEuler(endEuler);

let t = 0;
let direction = 1;

const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    const delta = clock.getDelta();
    
    // อัพเดต t
    t += delta * 0.3 * direction;
    if (t > 1 || t < 0) direction *= -1;
    t = Math.max(0, Math.min(1, t));
    
    // Euler interpolation (ไม่ smooth)
    eulerMesh.rotation.x = startEuler.x + (endEuler.x - startEuler.x) * t;
    eulerMesh.rotation.y = startEuler.y + (endEuler.y - startEuler.y) * t;
    eulerMesh.rotation.z = startEuler.z + (endEuler.z - startEuler.z) * t;
    
    // Quaternion slerp (smooth)
    quatMesh.quaternion.slerpQuaternions(startQuat, endQuat, t);
    
    renderer.render(scene, camera);
}
animate();
```

---

## 5. Scale

```javascript
// กำหนด scale
mesh.scale.set(2, 2, 2);    // uniform scale
mesh.scale.x = 2;
mesh.scale.y = 0.5;          // squash
mesh.scale.z = 2;

// Uniform scale
mesh.scale.setScalar(2);    // ทุกแกน = 2

// Negative scale = mirror/flip
mesh.scale.x = -1;          // flip ตามแกน X
```

---

## 6. Object3D Methods

```javascript
// translateX/Y/Z - เคลื่อนที่ตามแกน local
mesh.translateX(1);    // เคลื่อนที่ไป 1 หน่วยตามแกน X local
mesh.translateY(0.5);
mesh.translateZ(-1);

// ต่างจาก position.x += 1 ตรงที่ translateX คำนึงถึง rotation
// ถ้าวัตถุหมุน 90° ตามแกน Y แล้ว translateX จะเคลื่อนไปแกน Z ของ world

// rotateX/Y/Z - หมุนรอบแกน local
mesh.rotateX(Math.PI / 10);
mesh.rotateY(Math.PI / 6);
mesh.rotateZ(0.1);

// rotateOnAxis - หมุนรอบแกนที่กำหนด (world space)
const axis = new THREE.Vector3(0, 1, 0).normalize();
mesh.rotateOnAxis(axis, Math.PI / 12);

// rotateOnWorldAxis - หมุนรอบแกน world
mesh.rotateOnWorldAxis(new THREE.Vector3(0, 1, 0), 0.01);

// lookAt - หันหน้าไปยังจุดที่กำหนด
mesh.lookAt(new THREE.Vector3(0, 0, 0));
mesh.lookAt(targetMesh.position);

// ข้อควรระวัง: lookAt ใช้ +Z เป็นทิศ "หน้า"
// ถ้าโมเดลหันหน้าทาง -Z ต้องหมุนอีก 180°

// getWorldPosition - ตำแหน่งใน world space (ไม่คำนึง parent)
const worldPos = new THREE.Vector3();
mesh.getWorldPosition(worldPos);

// getWorldQuaternion - rotation ใน world space
const worldQuat = new THREE.Quaternion();
mesh.getWorldQuaternion(worldQuat);

// getWorldDirection - ทิศทาง +Z ใน world space
const worldDir = new THREE.Vector3();
mesh.getWorldDirection(worldDir);

// getWorldScale
const worldScale = new THREE.Vector3();
mesh.getWorldScale(worldScale);
```

---

## 7. Matrix4 - Transform Matrix

Matrix4 แทน transform ทั้งหมด (position + rotation + scale) ด้วย matrix 4x4

```javascript
const matrix = new THREE.Matrix4();

// สร้าง Transform Matrix
matrix.makeTranslation(5, 0, 0);
matrix.makeRotationY(Math.PI / 2);
matrix.makeScale(2, 2, 2);

// รวม transforms (multiply)
const translate = new THREE.Matrix4().makeTranslation(1, 0, 0);
const rotate = new THREE.Matrix4().makeRotationY(Math.PI / 2);
const scale = new THREE.Matrix4().makeScale(2, 1, 1);

// TRS order: Scale, then Rotate, then Translate
const combined = new THREE.Matrix4();
combined.multiplyMatrices(translate, rotate);
combined.multiply(scale);

// Apply matrix to mesh
mesh.matrix.copy(combined);
mesh.matrixAutoUpdate = false; // ปิด auto update เพื่อใช้ matrix เอง

// อ่านค่าจาก matrix
const position = new THREE.Vector3();
const quaternion = new THREE.Quaternion();
const scale2 = new THREE.Vector3();
matrix.decompose(position, quaternion, scale2);

// สร้าง matrix จาก position, rotation, scale
matrix.compose(position, quaternion, scale2);

// Local vs World matrix
// mesh.matrix = local transform (relative to parent)
// mesh.matrixWorld = world transform (absolute)

// updateMatrixWorld - อัพเดต matrixWorld
mesh.updateMatrixWorld(true); // true = recursive

// Transform point จาก local ไป world
const localPoint = new THREE.Vector3(1, 0, 0);
const worldPoint = localPoint.applyMatrix4(mesh.matrixWorld);

// Transform point จาก world ไป local
const invMatrix = mesh.matrixWorld.clone().invert();
const localPoint2 = worldPoint.clone().applyMatrix4(invMatrix);
```

---

## 8. Axes Helper - แสดงแกน

```javascript
// AxesHelper แสดงแกน X (แดง), Y (เขียว), Z (น้ำเงิน)
const axesHelper = new THREE.AxesHelper(5); // ขนาด 5 หน่วย
scene.add(axesHelper);

// เพิ่มให้กับวัตถุแทนที่จะ scene
const objAxes = new THREE.AxesHelper(2);
mesh.add(objAxes); // แสดงแกน local ของ mesh

// Arrow Helper - แสดงทิศทาง
const origin = new THREE.Vector3(0, 0, 0);
const direction = new THREE.Vector3(1, 0, 0).normalize();
const length = 3;
const color = 0xff0000;
const arrowHelper = new THREE.ArrowHelper(direction, origin, length, color);
scene.add(arrowHelper);

// อัพเดตลูกศร
arrowHelper.setDirection(new THREE.Vector3(0, 1, 0));
arrowHelper.setLength(5);
arrowHelper.setColor(0x00ff00);
```

---

## 9. Demo สมบูรณ์: Transform Visualization

```html
<!DOCTYPE html>
<html>
<head>
    <title>Transform Visualization</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #1a1a2e; overflow: hidden; font-family: monospace; }
        
        #panel {
            position: absolute;
            top: 10px;
            right: 10px;
            background: rgba(0,0,0,0.8);
            color: white;
            padding: 15px;
            border-radius: 8px;
            min-width: 280px;
        }
        
        h3 { color: #e94560; margin-bottom: 10px; }
        h4 { color: #4ecdc4; margin: 10px 0 5px; }
        
        .row {
            display: flex;
            justify-content: space-between;
            margin: 4px 0;
            align-items: center;
        }
        
        .row label { color: #aaa; width: 30px; }
        input[type="range"] { width: 120px; }
        .value { color: #ffd700; width: 50px; text-align: right; }
        
        select {
            background: #333;
            color: white;
            border: 1px solid #555;
            padding: 3px;
            margin-left: 5px;
        }
        
        #info {
            position: absolute;
            bottom: 10px;
            left: 10px;
            color: #aaa;
            font-size: 12px;
        }
    </style>
</head>
<body>
<div id="panel">
    <h3>Transform Controls</h3>
    
    <h4>Position</h4>
    <div class="row">
        <label>X:</label>
        <input type="range" id="posX" min="-5" max="5" step="0.1" value="0">
        <span class="value" id="posXVal">0.0</span>
    </div>
    <div class="row">
        <label>Y:</label>
        <input type="range" id="posY" min="-5" max="5" step="0.1" value="0">
        <span class="value" id="posYVal">0.0</span>
    </div>
    <div class="row">
        <label>Z:</label>
        <input type="range" id="posZ" min="-5" max="5" step="0.1" value="0">
        <span class="value" id="posZVal">0.0</span>
    </div>
    
    <h4>Rotation</h4>
    <div class="row">
        <label>X:</label>
        <input type="range" id="rotX" min="-180" max="180" step="1" value="0">
        <span class="value" id="rotXVal">0°</span>
    </div>
    <div class="row">
        <label>Y:</label>
        <input type="range" id="rotY" min="-180" max="180" step="1" value="0">
        <span class="value" id="rotYVal">0°</span>
    </div>
    <div class="row">
        <label>Z:</label>
        <input type="range" id="rotZ" min="-180" max="180" step="1" value="0">
        <span class="value" id="rotZVal">0°</span>
    </div>
    <div class="row">
        <label>Order:</label>
        <select id="rotOrder">
            <option>XYZ</option>
            <option>YXZ</option>
            <option>ZXY</option>
            <option>ZYX</option>
            <option>YZX</option>
            <option>XZY</option>
        </select>
    </div>
    
    <h4>Scale</h4>
    <div class="row">
        <label>X:</label>
        <input type="range" id="scaleX" min="0.1" max="3" step="0.1" value="1">
        <span class="value" id="scaleXVal">1.0</span>
    </div>
    <div class="row">
        <label>Y:</label>
        <input type="range" id="scaleY" min="0.1" max="3" step="0.1" value="1">
        <span class="value" id="scaleYVal">1.0</span>
    </div>
    <div class="row">
        <label>Z:</label>
        <input type="range" id="scaleZ" min="0.1" max="3" step="0.1" value="1">
        <span class="value" id="scaleZVal">1.0</span>
    </div>
    
    <h4>World Position</h4>
    <div id="worldPos" style="color: #96ceb4; font-size: 12px;"></div>
</div>

<div id="info">R=Reset | แกน: X=แดง Y=เขียว Z=น้ำเงิน</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';

// Scene
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x1a1a2e);

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(8, 6, 8);
camera.lookAt(0, 0, 0);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
document.body.appendChild(renderer.domElement);

const orbitControls = new OrbitControls(camera, renderer.domElement);
orbitControls.enableDamping = true;

// Grid & Axes
scene.add(new THREE.GridHelper(20, 20, 0x333355, 0x222244));
const worldAxes = new THREE.AxesHelper(10);
scene.add(worldAxes);

// Lighting
scene.add(new THREE.AmbientLight(0xffffff, 0.4));
const dirLight = new THREE.DirectionalLight(0xffffff, 1);
dirLight.position.set(5, 10, 5);
scene.add(dirLight);

// สร้าง Parent Group
const parentGroup = new THREE.Group();
parentGroup.position.set(0, 0, 0);
scene.add(parentGroup);

// วัตถุหลัก (Main Object)
const mainGeo = new THREE.BoxGeometry(2, 2, 2);
const mainMat = new THREE.MeshStandardMaterial({ 
    color: 0xe94560,
    roughness: 0.3,
    metalness: 0.7
});
const mainMesh = new THREE.Mesh(mainGeo, mainMat);
parentGroup.add(mainMesh);

// Local Axes บนวัตถุ
const localAxes = new THREE.AxesHelper(3);
mainMesh.add(localAxes);

// Arrow helpers สำหรับแสดงทิศทาง
const arrows = {
    x: new THREE.ArrowHelper(
        new THREE.Vector3(1, 0, 0), 
        new THREE.Vector3(0, 0, 0), 
        3, 0xff0000, 0.4, 0.2
    ),
    y: new THREE.ArrowHelper(
        new THREE.Vector3(0, 1, 0), 
        new THREE.Vector3(0, 0, 0), 
        3, 0x00ff00, 0.4, 0.2
    ),
    z: new THREE.ArrowHelper(
        new THREE.Vector3(0, 0, 1), 
        new THREE.Vector3(0, 0, 0), 
        3, 0x0000ff, 0.4, 0.2
    )
};

// World position indicator
const worldPosSphere = new THREE.Mesh(
    new THREE.SphereGeometry(0.1, 16, 16),
    new THREE.MeshBasicMaterial({ color: 0xffff00 })
);
scene.add(worldPosSphere);

// Sliders
const sliders = {
    posX: { el: document.getElementById('posX'), val: document.getElementById('posXVal') },
    posY: { el: document.getElementById('posY'), val: document.getElementById('posYVal') },
    posZ: { el: document.getElementById('posZ'), val: document.getElementById('posZVal') },
    rotX: { el: document.getElementById('rotX'), val: document.getElementById('rotXVal') },
    rotY: { el: document.getElementById('rotY'), val: document.getElementById('rotYVal') },
    rotZ: { el: document.getElementById('rotZ'), val: document.getElementById('rotZVal') },
    scaleX: { el: document.getElementById('scaleX'), val: document.getElementById('scaleXVal') },
    scaleY: { el: document.getElementById('scaleY'), val: document.getElementById('scaleYVal') },
    scaleZ: { el: document.getElementById('scaleZ'), val: document.getElementById('scaleZVal') },
};

const rotOrderSelect = document.getElementById('rotOrder');
const worldPosDiv = document.getElementById('worldPos');

// Setup arrow update
function updateArrows() {
    const worldQuat = new THREE.Quaternion();
    mainMesh.getWorldQuaternion(worldQuat);
    const worldPos = new THREE.Vector3();
    mainMesh.getWorldPosition(worldPos);
    
    const axes = [
        { arrow: arrows.x, dir: new THREE.Vector3(1, 0, 0) },
        { arrow: arrows.y, dir: new THREE.Vector3(0, 1, 0) },
        { arrow: arrows.z, dir: new THREE.Vector3(0, 0, 1) },
    ];
    
    axes.forEach(({ arrow, dir }) => {
        dir.applyQuaternion(worldQuat);
        arrow.setDirection(dir);
        arrow.position.copy(worldPos);
    });
    
    worldPosSphere.position.copy(worldPos);
    worldPosDiv.textContent = `x:${worldPos.x.toFixed(2)} y:${worldPos.y.toFixed(2)} z:${worldPos.z.toFixed(2)}`;
}

// Add arrows to scene
Object.values(arrows).forEach(arrow => scene.add(arrow));

// Event listeners
function updateMesh() {
    mainMesh.position.set(
        parseFloat(sliders.posX.el.value),
        parseFloat(sliders.posY.el.value),
        parseFloat(sliders.posZ.el.value)
    );
    
    mainMesh.rotation.order = rotOrderSelect.value;
    mainMesh.rotation.set(
        parseFloat(sliders.rotX.el.value) * Math.PI / 180,
        parseFloat(sliders.rotY.el.value) * Math.PI / 180,
        parseFloat(sliders.rotZ.el.value) * Math.PI / 180
    );
    
    mainMesh.scale.set(
        parseFloat(sliders.scaleX.el.value),
        parseFloat(sliders.scaleY.el.value),
        parseFloat(sliders.scaleZ.el.value)
    );
    
    // Update display values
    sliders.posX.val.textContent = parseFloat(sliders.posX.el.value).toFixed(1);
    sliders.posY.val.textContent = parseFloat(sliders.posY.el.value).toFixed(1);
    sliders.posZ.val.textContent = parseFloat(sliders.posZ.el.value).toFixed(1);
    sliders.rotX.val.textContent = sliders.rotX.el.value + '°';
    sliders.rotY.val.textContent = sliders.rotY.el.value + '°';
    sliders.rotZ.val.textContent = sliders.rotZ.el.value + '°';
    sliders.scaleX.val.textContent = parseFloat(sliders.scaleX.el.value).toFixed(1);
    sliders.scaleY.val.textContent = parseFloat(sliders.scaleY.el.value).toFixed(1);
    sliders.scaleZ.val.textContent = parseFloat(sliders.scaleZ.el.value).toFixed(1);
    
    updateArrows();
}

Object.values(sliders).forEach(s => s.el.addEventListener('input', updateMesh));
rotOrderSelect.addEventListener('change', updateMesh);

document.addEventListener('keydown', (e) => {
    if (e.key === 'r' || e.key === 'R') {
        Object.values(sliders).forEach(s => {
            if (s.el.id.startsWith('pos')) s.el.value = 0;
            else if (s.el.id.startsWith('rot')) s.el.value = 0;
            else s.el.value = 1;
        });
        updateMesh();
    }
});

// Resize
window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});

// Animation
function animate() {
    requestAnimationFrame(animate);
    orbitControls.update();
    renderer.render(scene, camera);
}
animate();
</script>
</body>
</html>
```

---

## 10. lookAt ในทางปฏิบัติ

```javascript
// Camera lookAt
camera.position.set(5, 5, 5);
camera.lookAt(0, 0, 0);         // มองไปที่ origin
camera.lookAt(mesh.position);   // มองไปที่ mesh

// Mesh lookAt (ใช้ +Z เป็นหน้า)
const enemy = new THREE.Mesh(enemyGeo, enemyMat);
const player = new THREE.Mesh(playerGeo, playerMat);

// ศัตรูหันไปหาผู้เล่น
function update() {
    enemy.lookAt(player.position);
}

// ถ้าโมเดลหัน -Z เป็น default ต้องเพิ่ม 180°
function lookAtWithFlip(obj, target) {
    obj.lookAt(target);
    obj.rotateY(Math.PI);
}

// Billboard effect - วัตถุหันไปหากล้องเสมอ
function makeBillboard(mesh, camera) {
    const cameraPos = new THREE.Vector3();
    camera.getWorldPosition(cameraPos);
    mesh.lookAt(cameraPos);
}
```

---

## แบบฝึกหัด

1. **Vector3 Calculator**: สร้าง UI คำนวณ Vector operations (dot, cross, lerp)
2. **Quaternion Rotation**: สร้าง animation หมุนวัตถุรอบแกน custom ด้วย quaternion
3. **Transform Chain**: สร้าง parent-child 3 ระดับ แล้วดู world position
4. **lookAt Tracker**: สร้างลูกศรที่ชี้ไปหา cursor ตลอดเวลา
5. **Matrix Transform**: เขียน function ที่รับ TRS แล้วสร้าง Matrix4 แล้ว apply กับวัตถุ

---

## สรุป

- `Object3D` เป็น base class ของทุกอย่างใน Three.js
- `Vector3` ใช้แทนตำแหน่งและทิศทาง มี methods มากมาย
- `Euler` ง่าย แต่มีปัญหา Gimbal Lock
- `Quaternion` แก้ Gimbal Lock และ slerp ได้ smooth กว่า
- `Matrix4` รวม position + rotation + scale ไว้ด้วยกัน
- `lookAt()` ทำให้วัตถุหันไปยังเป้าหมาย
- `getWorldPosition()` หาตำแหน่งจริงใน world space
