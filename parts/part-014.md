# Part 14: Scene Graph & Groups ใน Three.js

## บทนำ

Scene Graph คือโครงสร้างลำดับชั้น (hierarchy) ของ objects ใน scene Three.js ใช้ parent-child relationship ที่ทรงพลัง โดย child จะสืบทอด transform ของ parent ทำให้สร้างระบบที่ซับซ้อนได้ง่าย เช่น ระบบสุริยะ, ตัวละคร, รถยนต์

## 1. Object3D Parent-Child Relationships

เมื่อ object เป็น child ของ parent:
- Position ของ child เป็น **relative** กับ parent
- เมื่อ parent หมุน/ย้าย child จะหมุน/ย้ายตาม
- Scale ของ parent ส่งผลต่อ child ด้วย

```javascript
// สร้าง parent-child relationship
const parent = new THREE.Object3D();
parent.position.set(0, 5, 0);
scene.add(parent);

const child = new THREE.Mesh(
    new THREE.BoxGeometry(1, 1, 1),
    new THREE.MeshStandardMaterial({ color: 0xff0000 })
);
child.position.set(3, 0, 0); // 3 หน่วยจาก parent
parent.add(child);           // child อยู่ที่ world position (3, 5, 0)

// เมื่อหมุน parent
parent.rotation.y = Math.PI / 2;
// child จะหมุนตาม! ตอนนี้อยู่ที่ world position (0, 5, -3)

// เข้าถึง parent/children
console.log(child.parent);     // parent object
console.log(parent.children);  // [child]
console.log(parent.children[0] === child); // true
```

### การ traverse hierarchy

```javascript
// traverse - วนลูปทุก node ใน subtree
scene.traverse((object) => {
    console.log(object.name, object.type);
    
    // ทำ operation กับทุก Mesh
    if (object.isMesh) {
        object.castShadow = true;
        object.receiveShadow = true;
    }
    
    // เปิด/ปิดทุก Light
    if (object.isLight) {
        object.visible = false;
    }
});

// traverseVisible - วนเฉพาะที่ visible
scene.traverseVisible((obj) => {
    // ไม่เข้าถึง invisible subtree
});

// traverseAncestors - ขึ้นไปหา parent
child.traverseAncestors((obj) => {
    console.log('Ancestor:', obj.name);
});
```

### add และ remove

```javascript
// เพิ่ม
scene.add(mesh);
scene.add(mesh1, mesh2, mesh3); // เพิ่มหลายตัวพร้อมกัน
parent.add(child);

// ลบ
scene.remove(mesh);
parent.remove(child);

// ลบและ dispose (คืน memory)
function removeAndDispose(obj, parent) {
    parent.remove(obj);
    
    obj.traverse((child) => {
        if (child.geometry) child.geometry.dispose();
        if (child.material) {
            if (Array.isArray(child.material)) {
                child.material.forEach(m => m.dispose());
            } else {
                child.material.dispose();
            }
        }
    });
}
```

---

## 2. Group Class

`Group` เป็น subclass ของ Object3D ที่ไม่มี geometry ใช้สำหรับจัดกลุ่ม objects

```javascript
// สร้าง Group
const carGroup = new THREE.Group();
carGroup.name = 'car';

// สร้างชิ้นส่วนรถ
const body = new THREE.Mesh(
    new THREE.BoxGeometry(4, 1, 2),
    new THREE.MeshStandardMaterial({ color: 0x3498db })
);
body.position.y = 0.5;
carGroup.add(body);

const roof = new THREE.Mesh(
    new THREE.BoxGeometry(2.5, 0.8, 1.8),
    new THREE.MeshStandardMaterial({ color: 0x2980b9 })
);
roof.position.set(0.5, 1.4, 0);
carGroup.add(roof);

// ล้อรถ
const wheelGeo = new THREE.CylinderGeometry(0.4, 0.4, 0.3, 16);
const wheelMat = new THREE.MeshStandardMaterial({ color: 0x2c3e50 });
const wheelPositions = [
    { x: -1.5, y: 0, z: 1.1 },
    { x: 1.5, y: 0, z: 1.1 },
    { x: -1.5, y: 0, z: -1.1 },
    { x: 1.5, y: 0, z: -1.1 },
];

wheelPositions.forEach(pos => {
    const wheel = new THREE.Mesh(wheelGeo, wheelMat);
    wheel.rotation.z = Math.PI / 2;
    wheel.position.set(pos.x, pos.y, pos.z);
    wheel.name = 'wheel';
    carGroup.add(wheel);
});

// ย้ายรถทั้งคัน
scene.add(carGroup);
carGroup.position.set(0, 0, 0);

// หมุนรถทั้งคัน (ทุกชิ้นส่วนหมุนตาม)
carGroup.rotation.y = Math.PI / 4;
```

---

## 3. getObjectByName และ getObjectById

```javascript
// ตั้งชื่อ objects
const redBox = new THREE.Mesh(geo, redMat);
redBox.name = 'red-box';

const blueBox = new THREE.Mesh(geo, blueMat);
blueBox.name = 'blue-box';

scene.add(redBox, blueBox);

// ค้นหาด้วยชื่อ
const found = scene.getObjectByName('red-box');
console.log(found === redBox); // true

// ค้นหาใน subtree
const carWheel = carGroup.getObjectByName('front-left-wheel');

// ค้นหาด้วย ID (อัตโนมัติ เพิ่มขึ้นทีละ 1)
const byId = scene.getObjectById(redBox.id);

// ค้นหาด้วย property (ใช้ callback)
function getObjectByProperty(parent, prop, value) {
    let found = null;
    parent.traverse((obj) => {
        if (obj[prop] === value && !found) {
            found = obj;
        }
    });
    return found;
}

// ค้นหาทุก object ที่ตรง condition
function getObjectsByFilter(parent, filter) {
    const results = [];
    parent.traverse((obj) => {
        if (filter(obj)) results.push(obj);
    });
    return results;
}

// ใช้งาน
const allMeshes = getObjectsByFilter(scene, obj => obj.isMesh);
const allLights = getObjectsByFilter(scene, obj => obj.isLight);
const redObjects = getObjectsByFilter(scene, obj => 
    obj.isMesh && obj.material && obj.material.color && 
    obj.material.color.equals(new THREE.Color(0xff0000))
);
```

---

## 4. Clone

```javascript
// clone() - สร้าง deep copy ของ object
const original = new THREE.Mesh(
    new THREE.BoxGeometry(1, 1, 1),
    new THREE.MeshStandardMaterial({ color: 0xff0000 })
);
original.name = 'original';

// ตื้น clone (share geometry และ material)
const clone = original.clone();
clone.position.x = 3; // ย้ายเฉพาะ clone

// ปรับสี clone โดยไม่กระทบ original
// ต้อง clone material ด้วย!
clone.material = original.material.clone();
clone.material.color.setHex(0x0000ff);

// Deep clone (clone ทุกอย่าง รวม children)
const group = new THREE.Group();
group.add(mesh1, mesh2, mesh3);

const groupClone = group.clone(true); // true = recursive
groupClone.position.x = 10;
scene.add(groupClone);

// Clone แต่ share geometry (ประหยัด memory)
function cloneWithSharedGeometry(source) {
    const clone = source.clone();
    clone.geometry = source.geometry; // ใช้ geometry เดิม
    clone.material = source.material.clone(); // clone material
    return clone;
}
```

---

## 5. Layers

Layers ใช้ควบคุมว่า object ไหนจะถูก render ด้วย camera ไหน

```javascript
// Three.js มี 32 layers (0-31)
// Default = layer 0

// กำหนด layer
mesh.layers.set(1);        // ใส่ layer 1 อย่างเดียว
mesh.layers.enable(2);     // เพิ่ม layer 2 (ยังคง layer เดิม)
mesh.layers.disable(0);    // เอา layer 0 ออก
mesh.layers.toggle(3);     // toggle layer 3
mesh.layers.enableAll();   // ทุก layer
mesh.layers.disableAll();  // ไม่มี layer

// Camera เห็นเฉพาะ layer ที่ตรงกัน
camera.layers.set(0);      // camera default = layer 0
camera.layers.enable(1);   // camera เห็น layer 0 และ 1

// ตัวอย่าง: แยก UI elements จาก scene
const uiGroup = new THREE.Group();
uiGroup.layers.set(1); // UI ใน layer 1
scene.add(uiGroup);

// Camera หลัก
const mainCamera = new THREE.PerspectiveCamera(75, aspect, 0.1, 1000);
mainCamera.layers.enable(0); // เห็น scene ปกติ

// UI Camera
const uiCamera = new THREE.OrthographicCamera(-1, 1, 1, -1, 0, 1);
uiCamera.layers.enable(1); // เห็นแค่ UI layer

// Render สองรอบ
renderer.render(scene, mainCamera);
renderer.autoClear = false; // ไม่ clear ระหว่าง render
renderer.render(scene, uiCamera);
renderer.autoClear = true;
```

---

## 6. renderOrder และ visible

```javascript
// renderOrder - กำหนดลำดับการ render (ค่าน้อย = render ก่อน)
// ค่าเริ่มต้น = 0

// ใช้สำหรับ transparency ordering
const backgroundMesh = new THREE.Mesh(geo, mat);
backgroundMesh.renderOrder = 0;

const transparentMesh = new THREE.Mesh(geo, transparentMat);
transparentMesh.renderOrder = 1; // render หลัง background

// visible - เปิด/ปิดการแสดงผล
mesh.visible = false;  // ซ่อน (ยังอยู่ใน scene)
mesh.visible = true;   // แสดง

// visible = false ปิด subtree ทั้งหมด
group.visible = false; // ซ่อนทุก child ใน group

// ต่างจาก remove: visible = false ยังคงอยู่ใน scene
// remove ทำให้หายออกจาก hierarchy
```

---

## 7. Object3D.userData

```javascript
// userData คือ {} เปล่าสำหรับเก็บข้อมูล custom
const enemy = new THREE.Mesh(enemyGeo, enemyMat);

enemy.userData = {
    health: 100,
    maxHealth: 100,
    speed: 5,
    type: 'zombie',
    isAlive: true,
    reward: 50,
    lastAttack: 0
};

// เข้าถึง userData
console.log(enemy.userData.health);

// อัพเดต userData
enemy.userData.health -= 20;
enemy.userData.lastAttack = Date.now();

// เช็คสถานะ
if (enemy.userData.health <= 0) {
    enemy.userData.isAlive = false;
    scene.remove(enemy);
}

// ค้นหา enemies ทั้งหมด
const enemies = [];
scene.traverse((obj) => {
    if (obj.userData.type === 'zombie') {
        enemies.push(obj);
    }
});
```

---

## 8. Demo สมบูรณ์: Solar System

```html
<!DOCTYPE html>
<html>
<head>
    <title>Solar System Demo</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; }
        #info {
            position: absolute;
            top: 10px;
            left: 10px;
            color: white;
            font-family: monospace;
            font-size: 13px;
            background: rgba(0,0,0,0.7);
            padding: 10px;
            border-radius: 5px;
        }
        #speed {
            position: absolute;
            top: 10px;
            right: 10px;
            background: rgba(0,0,0,0.7);
            color: white;
            padding: 10px;
            border-radius: 5px;
            font-family: monospace;
        }
        input[type=range] { width: 150px; }
    </style>
</head>
<body>
<div id="info">
    <b>🌞 Solar System</b><br>
    ☀️ Sun (Center)<br>
    🌍 Earth orbits Sun<br>
    🌕 Moon orbits Earth<br>
    🪐 Saturn orbits Sun<br>
    <br>
    OrbitControls: หมุน/ซูม/เลื่อน
</div>
<div id="speed">
    Speed: <input type="range" id="speedSlider" min="0" max="10" step="0.1" value="1">
    <span id="speedValue">1.0x</span>
</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';

// Scene
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x000011);

const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 5000);
camera.position.set(0, 80, 200);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.dampingFactor = 0.05;
controls.minDistance = 10;
controls.maxDistance = 1000;

// ดาว background
const starGeo = new THREE.BufferGeometry();
const starPositions = new Float32Array(10000 * 3);
for (let i = 0; i < starPositions.length; i++) {
    starPositions[i] = (Math.random() - 0.5) * 4000;
}
starGeo.setAttribute('position', new THREE.BufferAttribute(starPositions, 3));
scene.add(new THREE.Points(starGeo, new THREE.PointsMaterial({ color: 0xffffff, size: 0.3 })));

// ฟังก์ชันสร้างวัตถุทรงกลม
function createSphere(radius, color, emissive = false) {
    const geo = new THREE.SphereGeometry(radius, 64, 64);
    const mat = emissive 
        ? new THREE.MeshBasicMaterial({ color })  // ไม่ต้องการ light
        : new THREE.MeshPhongMaterial({ 
            color, 
            shininess: 30,
            specular: 0x444444
          });
    return new THREE.Mesh(geo, mat);
}

// ฟังก์ชันสร้างวงโคจร
function createOrbit(radius) {
    const geo = new THREE.RingGeometry(radius - 0.1, radius + 0.1, 128);
    const mat = new THREE.MeshBasicMaterial({ 
        color: 0x444444, 
        side: THREE.DoubleSide,
        transparent: true,
        opacity: 0.3
    });
    const orbit = new THREE.Mesh(geo, mat);
    orbit.rotation.x = Math.PI / 2;
    return orbit;
}

// === ดวงอาทิตย์ ===
const sun = createSphere(15, 0xffdd44, true);
sun.name = 'Sun';
sun.userData = { type: 'star', temp: 5778 };
scene.add(sun);

// แสงจากดวงอาทิตย์
const sunLight = new THREE.PointLight(0xffffff, 2, 2000);
sun.add(sunLight);

// Glow ของดวงอาทิตย์
const sunGlow = createSphere(16, 0xffdd44, true);
sunGlow.material = new THREE.MeshBasicMaterial({
    color: 0xff8800,
    transparent: true,
    opacity: 0.3
});
sun.add(sunGlow);

// === ระบบดาวเคราะห์ ===
const planets = [
    // name, radius, color, orbitRadius, orbitSpeed, selfRotation, info
    { name: 'Mercury', radius: 3, color: 0x888888, orbit: 30, speed: 4.7, rot: 0.01, moons: [] },
    { name: 'Venus', radius: 4, color: 0xe8c07d, orbit: 50, speed: 3.5, rot: 0.005, moons: [] },
    { 
        name: 'Earth', radius: 5, color: 0x2233ff, orbit: 75, speed: 2.9, rot: 0.02,
        moons: [
            { name: 'Moon', radius: 1.5, color: 0xaaaaaa, orbit: 12, speed: 13 }
        ]
    },
    { name: 'Mars', radius: 3.5, color: 0xdd4400, orbit: 100, speed: 2.4, rot: 0.019, moons: [] },
    { name: 'Jupiter', radius: 10, color: 0xddaa66, orbit: 150, speed: 1.3, rot: 0.045, moons: [] },
    { 
        name: 'Saturn', radius: 9, color: 0xddcc88, orbit: 200, speed: 0.97, rot: 0.038,
        moons: [
            { name: 'Titan', radius: 2, color: 0xffaa55, orbit: 18, speed: 8 }
        ],
        rings: true
    },
    { name: 'Uranus', radius: 7, color: 0x77cccc, orbit: 250, speed: 0.68, rot: 0.03, moons: [] },
    { name: 'Neptune', radius: 7, color: 0x4455ff, orbit: 300, speed: 0.54, rot: 0.032, moons: [] },
];

const planetObjects = [];

planets.forEach(data => {
    // สร้าง orbit pivot (center of rotation)
    const orbitPivot = new THREE.Object3D();
    orbitPivot.name = `${data.name}_orbit_pivot`;
    scene.add(orbitPivot);
    
    // วงโคจร visual
    const orbitLine = createOrbit(data.orbit);
    scene.add(orbitLine);
    
    // สร้างดาวเคราะห์
    const planet = createSphere(data.radius, data.color);
    planet.name = data.name;
    planet.position.x = data.orbit;
    planet.userData = { 
        type: 'planet', 
        orbitRadius: data.orbit,
        selfRotSpeed: data.rot
    };
    
    // เพิ่ม rings สำหรับ Saturn
    if (data.rings) {
        const ringGeo = new THREE.RingGeometry(data.radius + 2, data.radius + 8, 64);
        const ringMat = new THREE.MeshBasicMaterial({
            color: 0xddcc88,
            side: THREE.DoubleSide,
            transparent: true,
            opacity: 0.6
        });
        const ring = new THREE.Mesh(ringGeo, ringMat);
        ring.rotation.x = Math.PI / 3;
        planet.add(ring);
    }
    
    // ดวงจันทร์
    data.moons.forEach(moonData => {
        const moonPivot = new THREE.Object3D();
        moonPivot.name = `${moonData.name}_pivot`;
        planet.add(moonPivot);
        
        const moon = createSphere(moonData.radius, moonData.color);
        moon.name = moonData.name;
        moon.position.x = moonData.orbit;
        moon.userData = { 
            type: 'moon',
            orbitRadius: moonData.orbit,
            orbitSpeed: moonData.speed
        };
        moonPivot.add(moon);
        
        // วงโคจรดวงจันทร์
        const moonOrbit = createOrbit(moonData.orbit);
        planet.add(moonOrbit);
        
        planetObjects.push({
            pivot: moonPivot,
            mesh: moon,
            speed: moonData.speed * 0.005,
            selfRotSpeed: 0.01
        });
    });
    
    orbitPivot.add(planet);
    
    planetObjects.push({
        pivot: orbitPivot,
        mesh: planet,
        speed: data.speed * 0.002,
        selfRotSpeed: data.rot
    });
});

// Labels (Sprites)
function createLabel(text, position) {
    const canvas = document.createElement('canvas');
    canvas.width = 256;
    canvas.height = 64;
    const ctx = canvas.getContext('2d');
    ctx.fillStyle = 'rgba(0,0,0,0)';
    ctx.fillRect(0, 0, 256, 64);
    ctx.fillStyle = 'white';
    ctx.font = '20px monospace';
    ctx.textAlign = 'center';
    ctx.fillText(text, 128, 40);
    
    const texture = new THREE.CanvasTexture(canvas);
    const mat = new THREE.SpriteMaterial({ map: texture, transparent: true });
    const sprite = new THREE.Sprite(mat);
    sprite.scale.set(20, 5, 1);
    sprite.position.copy(position);
    sprite.position.y += 20;
    return sprite;
}

// Speed control
let simulationSpeed = 1;
document.getElementById('speedSlider').addEventListener('input', (e) => {
    simulationSpeed = parseFloat(e.target.value);
    document.getElementById('speedValue').textContent = simulationSpeed.toFixed(1) + 'x';
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
    const delta = clock.getDelta() * simulationSpeed;
    
    // หมุนดวงอาทิตย์
    sun.rotation.y += 0.005 * simulationSpeed;
    
    // อัพเดตดาวเคราะห์และดวงจันทร์
    planetObjects.forEach(obj => {
        obj.pivot.rotation.y += obj.speed * simulationSpeed;
        obj.mesh.rotation.y += obj.selfRotSpeed * simulationSpeed;
    });
    
    controls.update();
    renderer.render(scene, camera);
}
animate();
</script>
</body>
</html>
```

---

## 9. Scene Traversal Examples

```javascript
// เปลี่ยน material ทุก mesh
function setAllMaterial(parent, material) {
    parent.traverse(obj => {
        if (obj.isMesh) {
            obj.material = material;
        }
    });
}

// เปิด shadow ทุก object
function enableAllShadows(parent) {
    parent.traverse(obj => {
        if (obj.isMesh) {
            obj.castShadow = true;
            obj.receiveShadow = true;
        }
    });
}

// นับจำนวน mesh ทั้งหมด
function countMeshes(parent) {
    let count = 0;
    parent.traverse(obj => {
        if (obj.isMesh) count++;
    });
    return count;
}

// หา bounding box ของทั้ง group
function getGroupBoundingBox(group) {
    const box = new THREE.Box3();
    group.traverse(obj => {
        if (obj.isMesh) {
            obj.geometry.computeBoundingBox();
            const meshBox = obj.geometry.boundingBox.clone();
            meshBox.applyMatrix4(obj.matrixWorld);
            box.union(meshBox);
        }
    });
    return box;
}

// Center group at origin
function centerGroup(group) {
    const box = getGroupBoundingBox(group);
    const center = new THREE.Vector3();
    box.getCenter(center);
    group.position.sub(center);
}

// ลบ dispose ทุกอย่าง
function disposeHierarchy(node) {
    node.traverse(obj => {
        if (obj.geometry) obj.geometry.dispose();
        if (obj.material) {
            if (Array.isArray(obj.material)) {
                obj.material.forEach(m => {
                    if (m.map) m.map.dispose();
                    m.dispose();
                });
            } else {
                if (obj.material.map) obj.material.map.dispose();
                obj.material.dispose();
            }
        }
    });
}
```

---

## 10. Dispose - คืน Memory

```javascript
// เมื่อลบ object ออกจาก scene ต้อง dispose ทรัพยากรด้วย
function removeObject(mesh) {
    // ลบจาก scene
    scene.remove(mesh);
    
    // Dispose geometry
    if (mesh.geometry) {
        mesh.geometry.dispose();
    }
    
    // Dispose material และ textures
    if (mesh.material) {
        const materials = Array.isArray(mesh.material) 
            ? mesh.material 
            : [mesh.material];
        
        materials.forEach(mat => {
            // Dispose textures
            Object.values(mat).forEach(value => {
                if (value instanceof THREE.Texture) {
                    value.dispose();
                }
            });
            mat.dispose();
        });
    }
}

// Dispose ทั้ง scene
function disposeScene(scene) {
    while (scene.children.length > 0) {
        const child = scene.children[0];
        disposeHierarchy(child);
        scene.remove(child);
    }
}

// Dispose renderer
function cleanup() {
    disposeScene(scene);
    renderer.dispose();
    renderer.domElement.remove();
}
```

---

## 11. Instance Mesh - Render หลาย Object ด้วย 1 Draw Call

```javascript
// InstancedMesh - เหมาะสำหรับ object เหมือนกันหลาย ๆ ตัว เช่น ต้นไม้, คน, หิน
const count = 1000;
const geo = new THREE.BoxGeometry(0.5, 0.5, 0.5);
const mat = new THREE.MeshStandardMaterial({ color: 0x00ff88 });
const instancedMesh = new THREE.InstancedMesh(geo, mat, count);

const matrix = new THREE.Matrix4();
const position = new THREE.Vector3();
const quaternion = new THREE.Quaternion();
const scale = new THREE.Vector3(1, 1, 1);

// กำหนด transform ของแต่ละ instance
for (let i = 0; i < count; i++) {
    position.set(
        (Math.random() - 0.5) * 100,
        Math.random() * 5,
        (Math.random() - 0.5) * 100
    );
    
    quaternion.setFromAxisAngle(
        new THREE.Vector3(0, 1, 0),
        Math.random() * Math.PI * 2
    );
    
    matrix.compose(position, quaternion, scale);
    instancedMesh.setMatrixAt(i, matrix);
}

instancedMesh.instanceMatrix.needsUpdate = true;
scene.add(instancedMesh);

// อัพเดต instance ระหว่าง runtime
function updateInstance(index, newPosition) {
    instancedMesh.getMatrixAt(index, matrix);
    matrix.decompose(position, quaternion, scale);
    position.copy(newPosition);
    matrix.compose(position, quaternion, scale);
    instancedMesh.setMatrixAt(index, matrix);
    instancedMesh.instanceMatrix.needsUpdate = true; // สำคัญ!
}
```

---

## แบบฝึกหัด

1. **Robot Arm**: สร้างแขนหุ่นยนต์ด้วย Groups ที่มี shoulder, elbow, wrist, fingers
2. **Scene Manager**: สร้าง class จัดการ scene graph มี add, remove, findByName, findByType
3. **Level System**: ใช้ Groups สร้างระบบ level ที่ซ่อน/แสดงได้
4. **Tree Generator**: สร้างต้นไม้ procedural ด้วย Groups และ Children
5. **Instance Forest**: ใช้ InstancedMesh สร้างป่าไม้ 10,000 ต้น

---

## สรุป

- **Scene Graph** ใช้ parent-child hierarchy สำหรับจัดการ objects
- **Group** เป็น container สำหรับรวม objects ที่เกี่ยวข้องกัน
- **traverse** วนลูปทุก node ในลำดับชั้น
- **getObjectByName/ById** ค้นหา object ใน scene
- **clone** สร้าง copy ของ object (share geometry แต่ clone material)
- **dispose** คืน GPU memory เมื่อไม่ใช้งาน
- **InstancedMesh** render หลาย object ด้วย 1 draw call
- **layers** ควบคุม visibility ต่อ camera
- **userData** เก็บข้อมูล custom ของ object
