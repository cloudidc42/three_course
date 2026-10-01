# Part 31: Physics with Cannon.js

## บทนำ: Physics Engine คืออะไร?

Physics engine คือซอฟต์แวร์ที่จำลองกฎฟิสิกส์ในโลก 3D เช่น แรงโน้มถ่วง การชนกัน ความเสียดทาน ฯลฯ Three.js เองไม่มี physics ในตัว จึงต้องใช้ library ภายนอก **cannon-es** คือ JavaScript port ของ Cannon.js ที่ maintain อยู่

```bash
npm install cannon-es
```

---

## Architecture: Three.js + Cannon.js

```
Three.js World (Visual)          Cannon.js World (Physics)
├── Scene                        ├── World
│   ├── Mesh (visual)   ←sync→  │   ├── Body (physics)
│   │   ├── position    ←copy←  │   │   ├── position
│   │   └── quaternion  ←copy←  │   │   └── quaternion
│   └── ...                      └── ...
```

**กระบวนการทำงาน:**
1. อัพเดท Physics world (world.step)
2. Copy ตำแหน่งจาก Physics body ไปยัง Three.js mesh
3. Render

---

## World Setup

```javascript
import * as CANNON from 'cannon-es';
import * as THREE from 'three';

// สร้าง Physics World
const world = new CANNON.World({
    gravity: new CANNON.Vec3(0, -9.82, 0)  // โน้มถ่วงโลก
});

// Performance settings
world.broadphase = new CANNON.SAPBroadphase(world);  // ดีกว่า NaiveBroadphase
world.allowSleep = true;  // ให้ bodies "หลับ" เมื่อหยุดนิ่ง (ประหยัด CPU)

// Fixed timestep (60 FPS physics)
const FIXED_TIMESTEP = 1 / 60;
const MAX_SUBSTEPS = 3;

function physicsUpdate(deltaTime) {
    world.step(FIXED_TIMESTEP, deltaTime, MAX_SUBSTEPS);
}
```

---

## Body: Physics Object

```javascript
// สร้าง Body พื้นฐาน
const body = new CANNON.Body({
    mass: 1,            // kg (0 = static/infinite mass)
    position: new CANNON.Vec3(0, 5, 0),
    shape: new CANNON.Sphere(0.5),
    
    // Optional
    velocity: new CANNON.Vec3(2, 0, 0),
    angularVelocity: new CANNON.Vec3(0, 1, 0),
    linearDamping: 0.1,   // ความต้านทานการเคลื่อนที่ (0-1)
    angularDamping: 0.1,  // ความต้านทานการหมุน (0-1)
    material: physicsMaterial  // friction/restitution
});

world.addBody(body);

// Static body (ไม่เคลื่อนที่)
const groundBody = new CANNON.Body({
    mass: 0,  // mass = 0 → static
    shape: new CANNON.Plane()
});
groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
world.addBody(groundBody);

// Kinematic body (เคลื่อนที่ตาม code ไม่ใช่ physics)
const kinematicBody = new CANNON.Body({ mass: 0 });
kinematicBody.type = CANNON.Body.KINEMATIC;
```

---

## Shapes

```javascript
// Sphere
const sphere = new CANNON.Sphere(radius);

// Box
const box = new CANNON.Box(new CANNON.Vec3(halfX, halfY, halfZ));

// Plane (infinite, ใช้สำหรับ ground)
const plane = new CANNON.Plane();

// Cylinder
const cylinder = new CANNON.Cylinder(
    radiusTop,    // ปลายบน
    radiusBottom, // ปลายล่าง
    height,
    numSegments
);

// ConvexPolyhedron (arbitrary convex shape)
const vertices = [/* ... */];
const faces = [/* ... */];
const convex = new CANNON.ConvexPolyhedron({ vertices, faces });

// Trimesh (arbitrary mesh - SLOW, ใช้สำหรับ static เท่านั้น)
const trimesh = new CANNON.Trimesh(
    positions,  // Float32Array
    indices     // Int16Array
);

// Compound Shape (หลาย shapes ใน body เดียว)
const compoundBody = new CANNON.Body({ mass: 1 });
compoundBody.addShape(new CANNON.Box(new CANNON.Vec3(0.5, 0.5, 0.5)));
compoundBody.addShape(
    new CANNON.Sphere(0.3),
    new CANNON.Vec3(0, 1, 0)  // offset
);
```

---

## Material & ContactMaterial

```javascript
// Physics Material กำหนด friction และ restitution
const iceMaterial = new CANNON.Material('ice');
const rockMaterial = new CANNON.Material('rock');
const rubberMaterial = new CANNON.Material('rubber');

// ContactMaterial กำหนดพฤติกรรมเมื่อ materials สองอันชนกัน
const iceRockContact = new CANNON.ContactMaterial(
    iceMaterial,
    rockMaterial,
    {
        friction: 0.02,       // ลื่นมาก
        restitution: 0.1      // ไม่เด้ง
    }
);

const rubberRockContact = new CANNON.ContactMaterial(
    rubberMaterial,
    rockMaterial,
    {
        friction: 0.8,        // เสียดทานสูง
        restitution: 0.8      // เด้งมาก
    }
);

world.addContactMaterial(iceRockContact);
world.addContactMaterial(rubberRockContact);

// ใช้ material กับ body
const ball = new CANNON.Body({
    mass: 1,
    shape: new CANNON.Sphere(0.5),
    material: rubberMaterial
});
```

---

## Constraints (Joint)

```javascript
// 1. PointToPoint (Ball Joint) - หมุนได้ทุกแกน
const p2p = new CANNON.PointToPointConstraint(
    bodyA,
    new CANNON.Vec3(0, 1, 0),   // pivot ใน bodyA local space
    bodyB,
    new CANNON.Vec3(0, -1, 0)   // pivot ใน bodyB local space
);
world.addConstraint(p2p);

// 2. HingeConstraint (Revolute) - หมุนได้แกนเดียว
const hinge = new CANNON.HingeConstraint(
    bodyA,
    bodyB,
    {
        pivotA: new CANNON.Vec3(0, 1, 0),
        pivotB: new CANNON.Vec3(0, -1, 0),
        axisA: new CANNON.Vec3(0, 1, 0),    // แกนหมุน
        axisB: new CANNON.Vec3(0, 1, 0)
    }
);
world.addConstraint(hinge);

// Enable motor บน hinge
hinge.enableMotor();
hinge.setMotorSpeed(3.0);  // rad/s

// 3. LockConstraint - ล็อคทุก DOF
const lock = new CANNON.LockConstraint(bodyA, bodyB);
world.addConstraint(lock);

// 4. DistanceConstraint - รักษาระยะห่าง
const dist = new CANNON.DistanceConstraint(bodyA, bodyB, 2.0);  // 2m apart
world.addConstraint(dist);

// 5. ConeTwistConstraint - เหมือน ball joint แต่มี limits
const coneTwist = new CANNON.ConeTwistConstraint(bodyA, bodyB, {
    pivotA: new CANNON.Vec3(0, 1, 0),
    pivotB: new CANNON.Vec3(0, -1, 0),
    axisA: new CANNON.Vec3(0, 1, 0),
    axisB: new CANNON.Vec3(0, 1, 0)
});
world.addConstraint(coneTwist);
```

---

## Synchronization: Physics → Three.js

```javascript
// Pair: physics body + three.js mesh
const objectsToSync = [];

function createPhysicsMesh(
    geometry, physicsShape,
    mass, position, material
) {
    // Three.js mesh
    const mesh = new THREE.Mesh(geometry, material);
    mesh.position.copy(position);
    scene.add(mesh);
    
    // Cannon body
    const body = new CANNON.Body({
        mass,
        shape: physicsShape,
        position: new CANNON.Vec3(position.x, position.y, position.z)
    });
    world.addBody(body);
    
    // Store pair
    objectsToSync.push({ mesh, body });
    
    return { mesh, body };
}

// Sync ใน animation loop
function syncPhysics() {
    for (const { mesh, body } of objectsToSync) {
        // Copy position
        mesh.position.copy(body.position);
        // Copy rotation
        mesh.quaternion.copy(body.quaternion);
    }
}

// Animation loop
const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    
    const deltaTime = clock.getDelta();
    
    // Update physics
    world.step(1 / 60, deltaTime, 3);
    
    // Sync meshes
    syncPhysics();
    
    renderer.render(scene, camera);
}
```

---

## Demo: Complete Physics Scene

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Cannon.js Physics Demo</title>
    <style>
        * { margin: 0; padding: 0; }
        canvas { display: block; }
        #ui {
            position: fixed;
            top: 15px;
            left: 15px;
            color: white;
            font-family: monospace;
            background: rgba(0,0,0,0.7);
            padding: 12px;
            border-radius: 8px;
        }
        button {
            display: block;
            margin: 5px 0;
            padding: 8px 15px;
            background: #2a4a8a;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            width: 100%;
        }
        button:hover { background: #3a6aaa; }
    </style>
</head>
<body>
<div id="ui">
    <b>Physics Demo</b>
    <button id="addSphere">Add Sphere</button>
    <button id="addBox">Add Box</button>
    <button id="explosion">Explosion!</button>
    <button id="reset">Reset</button>
    <div id="info">Objects: 0</div>
</div>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';
import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.157.0/examples/jsm/controls/OrbitControls.js';
import * as CANNON from 'https://cdn.jsdelivr.net/npm/cannon-es@0.20.0/dist/cannon-es.js';

// === Three.js Setup ===
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x1a1a2e);

const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 100);
camera.position.set(0, 8, 15);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.target.set(0, 3, 0);

// Lighting
const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
scene.add(ambientLight);

const dirLight = new THREE.DirectionalLight(0xffffff, 1.0);
dirLight.position.set(10, 20, 10);
dirLight.castShadow = true;
dirLight.shadow.mapSize.set(2048, 2048);
dirLight.shadow.camera.near = 0.1;
dirLight.shadow.camera.far = 100;
dirLight.shadow.camera.top = 20;
dirLight.shadow.camera.right = 20;
dirLight.shadow.camera.bottom = -20;
dirLight.shadow.camera.left = -20;
scene.add(dirLight);

// === Cannon.js Setup ===
const world = new CANNON.World({
    gravity: new CANNON.Vec3(0, -9.82, 0)
});
world.broadphase = new CANNON.SAPBroadphase(world);
world.allowSleep = true;

// Materials
const defaultMaterial = new CANNON.Material('default');
const groundContactMaterial = new CANNON.ContactMaterial(
    defaultMaterial,
    defaultMaterial,
    { friction: 0.4, restitution: 0.3 }
);
world.addContactMaterial(groundContactMaterial);
world.defaultContactMaterial = groundContactMaterial;

// === Objects to Sync ===
const syncObjects = [];

// === Ground ===
// Three.js ground
const groundGeo = new THREE.PlaneGeometry(30, 30);
const groundMat = new THREE.MeshStandardMaterial({
    color: 0x2d2d4e,
    roughness: 0.8,
    metalness: 0.1
});
const groundMesh = new THREE.Mesh(groundGeo, groundMat);
groundMesh.rotateX(-Math.PI / 2);
groundMesh.receiveShadow = true;
scene.add(groundMesh);

// Grid
scene.add(new THREE.GridHelper(30, 30, 0x444466, 0x333355));

// Cannon ground
const groundBody = new CANNON.Body({ mass: 0 });
groundBody.addShape(new CANNON.Plane());
groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
world.addBody(groundBody);

// === Walls ===
function addWall(pos, size) {
    const wallGeo = new THREE.BoxGeometry(...size);
    const wallMat = new THREE.MeshStandardMaterial({ color: 0x3a3a5e, transparent: true, opacity: 0.3 });
    const wallMesh = new THREE.Mesh(wallGeo, wallMat);
    wallMesh.position.set(...pos);
    wallMesh.receiveShadow = true;
    scene.add(wallMesh);
    
    const wallBody = new CANNON.Body({ mass: 0 });
    wallBody.addShape(new CANNON.Box(new CANNON.Vec3(...size.map(v => v / 2))));
    wallBody.position.set(...pos);
    world.addBody(wallBody);
}

addWall([0, 3, -10], [30, 6, 0.5]);
addWall([0, 3, 10], [30, 6, 0.5]);
addWall([-10, 3, 0], [0.5, 6, 20]);
addWall([10, 3, 0], [0.5, 6, 20]);

// === Dynamic Objects ===
const colors = [0xff4444, 0x44ff44, 0x4444ff, 0xffff44, 0xff44ff, 0x44ffff, 0xff8844];
let colorIndex = 0;

function addSphere(position) {
    const radius = 0.3 + Math.random() * 0.3;
    const color = colors[colorIndex++ % colors.length];
    
    // Three.js
    const geo = new THREE.SphereGeometry(radius, 16, 16);
    const mat = new THREE.MeshStandardMaterial({ color, roughness: 0.3, metalness: 0.1 });
    const mesh = new THREE.Mesh(geo, mat);
    mesh.castShadow = true;
    mesh.receiveShadow = true;
    scene.add(mesh);
    
    // Cannon
    const body = new CANNON.Body({
        mass: 1,
        shape: new CANNON.Sphere(radius),
        position: new CANNON.Vec3(
            position.x + (Math.random() - 0.5) * 2,
            position.y,
            position.z + (Math.random() - 0.5) * 2
        )
    });
    body.velocity.set(
        (Math.random() - 0.5) * 5,
        2,
        (Math.random() - 0.5) * 5
    );
    world.addBody(body);
    
    syncObjects.push({ mesh, body });
    updateInfo();
    
    return { mesh, body };
}

function addBox(position) {
    const size = 0.3 + Math.random() * 0.4;
    const color = colors[colorIndex++ % colors.length];
    
    // Three.js
    const geo = new THREE.BoxGeometry(size, size, size);
    const mat = new THREE.MeshStandardMaterial({ color, roughness: 0.5, metalness: 0.2 });
    const mesh = new THREE.Mesh(geo, mat);
    mesh.castShadow = true;
    mesh.receiveShadow = true;
    scene.add(mesh);
    
    // Cannon
    const halfSize = size / 2;
    const body = new CANNON.Body({
        mass: 1,
        shape: new CANNON.Box(new CANNON.Vec3(halfSize, halfSize, halfSize)),
        position: new CANNON.Vec3(
            position.x + (Math.random() - 0.5) * 2,
            position.y,
            position.z + (Math.random() - 0.5) * 2
        )
    });
    body.angularVelocity.set(
        Math.random() * 2,
        Math.random() * 2,
        Math.random() * 2
    );
    world.addBody(body);
    
    syncObjects.push({ mesh, body });
    updateInfo();
    
    return { mesh, body };
}

// เพิ่ม objects เริ่มต้น
for (let i = 0; i < 5; i++) {
    addSphere(new THREE.Vector3(0, 5 + i * 1.5, 0));
}
for (let i = 0; i < 5; i++) {
    addBox(new THREE.Vector3(0, 5 + i * 1.5, 0));
}

// Explosion effect
function doExplosion() {
    const center = new CANNON.Vec3(0, 0, 0);
    const force = 20;
    
    for (const { body } of syncObjects) {
        const dir = new CANNON.Vec3().copy(body.position).vsub(center);
        dir.normalize();
        dir.scale(force, dir);
        body.applyImpulse(dir, body.position);
        body.wakeUp();
    }
}

// Reset
function reset() {
    for (const { mesh, body } of syncObjects) {
        scene.remove(mesh);
        world.removeBody(body);
    }
    syncObjects.length = 0;
    colorIndex = 0;
    
    for (let i = 0; i < 5; i++) {
        addSphere(new THREE.Vector3(0, 5 + i * 1.5, 0));
    }
    for (let i = 0; i < 5; i++) {
        addBox(new THREE.Vector3(0, 5 + i * 1.5, 0));
    }
}

function updateInfo() {
    document.getElementById('info').textContent = `Objects: ${syncObjects.length}`;
}

// UI
document.getElementById('addSphere').onclick = () => addSphere(new THREE.Vector3(0, 10, 0));
document.getElementById('addBox').onclick    = () => addBox(new THREE.Vector3(0, 10, 0));
document.getElementById('explosion').onclick = doExplosion;
document.getElementById('reset').onclick     = reset;

// Animation
const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    
    const delta = clock.getDelta();
    
    // Physics step
    world.step(1 / 60, delta, 3);
    
    // Sync
    for (const { mesh, body } of syncObjects) {
        mesh.position.copy(body.position);
        mesh.quaternion.copy(body.quaternion);
    }
    
    controls.update();
    renderer.render(scene, camera);
}
animate();

window.addEventListener('resize', () => {
    camera.aspect = innerWidth / innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(innerWidth, innerHeight);
});
</script>
</body>
</html>
```

---

## Demo: Domino Effect

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Domino Effect</title>
    <style>* { margin: 0; padding: 0; } canvas { display: block; }</style>
</head>
<body>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';
import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.157.0/examples/jsm/controls/OrbitControls.js';
import * as CANNON from 'https://cdn.jsdelivr.net/npm/cannon-es@0.20.0/dist/cannon-es.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x1a1a2e);
scene.fog = new THREE.Fog(0x1a1a2e, 20, 60);

const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 100);
camera.position.set(0, 6, 20);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.shadowMap.enabled = true;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.target.set(0, 2, 0);

// Lights
scene.add(new THREE.AmbientLight(0xffffff, 0.4));
const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
dirLight.position.set(5, 15, 5);
dirLight.castShadow = true;
dirLight.shadow.mapSize.set(1024, 1024);
scene.add(dirLight);

// Physics
const world = new CANNON.World({ gravity: new CANNON.Vec3(0, -9.82, 0) });
world.broadphase = new CANNON.SAPBroadphase(world);
world.allowSleep = true;

const syncObjects = [];

// Ground
const groundMesh = new THREE.Mesh(
    new THREE.PlaneGeometry(40, 40),
    new THREE.MeshStandardMaterial({ color: 0x2d2d4e })
);
groundMesh.rotateX(-Math.PI / 2);
groundMesh.receiveShadow = true;
scene.add(groundMesh);
scene.add(new THREE.GridHelper(40, 40, 0x333355, 0x222244));

const groundBody = new CANNON.Body({ mass: 0 });
groundBody.addShape(new CANNON.Plane());
groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
world.addBody(groundBody);

// Domino dimensions
const DOMINO_W = 0.3;
const DOMINO_H = 1.2;
const DOMINO_D = 0.1;

const dominoGeo = new THREE.BoxGeometry(DOMINO_W, DOMINO_H, DOMINO_D);
const dominoColors = [0xff4444, 0xff8844, 0xffff44, 0x44ff44, 0x44ffff, 0x4444ff, 0xff44ff];

function createDomino(x, z, colorIdx = 0) {
    const mesh = new THREE.Mesh(
        dominoGeo,
        new THREE.MeshStandardMaterial({ color: dominoColors[colorIdx % dominoColors.length] })
    );
    mesh.position.set(x, DOMINO_H / 2, z);
    mesh.castShadow = true;
    mesh.receiveShadow = true;
    scene.add(mesh);
    
    const body = new CANNON.Body({
        mass: 1,
        shape: new CANNON.Box(new CANNON.Vec3(DOMINO_W/2, DOMINO_H/2, DOMINO_D/2)),
        position: new CANNON.Vec3(x, DOMINO_H / 2, z),
        sleepSpeedLimit: 0.1,
        sleepTimeLimit: 1.0
    });
    body.linearDamping = 0.1;
    body.angularDamping = 0.1;
    world.addBody(body);
    
    syncObjects.push({ mesh, body });
    return { mesh, body };
}

// สร้าง domino ในแนวตรง
for (let i = 0; i < 30; i++) {
    createDomino(-14 + i, 0, i);
}

// สร้าง domino โค้ง
for (let i = 0; i < 20; i++) {
    const angle = (i / 20) * Math.PI;
    const r = 8;
    createDomino(
        Math.cos(angle) * r,
        Math.sin(angle) * r + 5,
        30 + i
    );
}

// Push first domino
function startDomino() {
    const firstBody = syncObjects[0].body;
    firstBody.wakeUp();
    firstBody.applyImpulse(
        new CANNON.Vec3(0, 0, -3),  // push ไปข้างหน้า
        firstBody.position
    );
}

setTimeout(startDomino, 1000);

// Click to start
renderer.domElement.addEventListener('click', startDomino);

const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    const delta = clock.getDelta();
    world.step(1/60, delta, 3);
    
    for (const { mesh, body } of syncObjects) {
        mesh.position.copy(body.position);
        mesh.quaternion.copy(body.quaternion);
    }
    
    controls.update();
    renderer.render(scene, camera);
}
animate();

window.addEventListener('resize', () => {
    camera.aspect = innerWidth / innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(innerWidth, innerHeight);
});
</script>
</body>
</html>
```

---

## Sleep/Wake System

```javascript
// ตั้งค่า sleep parameters
world.sleepSpeedLimit = 0.1;     // body จะ sleep ถ้าเร็วน้อยกว่านี้
world.sleepTimeLimit = 1.0;      // ต้องช้ากว่า speedLimit นาน 1 วินาทีก่อน sleep

// per-body sleep settings
body.sleepSpeedLimit = 0.2;
body.sleepTimeLimit = 0.5;

// ตรวจสอบ sleep state
if (body.sleepState === CANNON.Body.SLEEPING) {
    console.log('Body is asleep');
} else if (body.sleepState === CANNON.Body.AWAKE) {
    console.log('Body is awake');
}

// Wake up body
body.wakeUp();

// ฟัง events
body.addEventListener('sleep', () => {
    console.log('Body went to sleep');
});
body.addEventListener('wakeup', () => {
    console.log('Body woke up');
});
```

---

## Collision Events

```javascript
// ฟัง collision events
world.addEventListener('beginContact', (event) => {
    const { bodyA, bodyB } = event;
    console.log('Collision between', bodyA.id, 'and', bodyB.id);
    
    // ตัวอย่าง: เล่นเสียงเมื่อ sphere ชน ground
    const velocity = bodyA.velocity.length();
    if (velocity > 2) {
        playSound('impact', velocity / 10);
    }
});

world.addEventListener('endContact', (event) => {
    const { bodyA, bodyB } = event;
    console.log('Contact ended');
});

// Per-body collision
bodyA.addEventListener('collide', (event) => {
    const { body, contact } = event;
    // contact.getImpactVelocityAlongNormal() - ความเร็วชน
    const impactVelocity = contact.getImpactVelocityAlongNormal();
    console.log(`Collision at velocity: ${impactVelocity}`);
});
```

---

## Vehicle Physics Intro

```javascript
// RaycastVehicle - ใช้ raycast สำหรับ suspension
const vehicle = new CANNON.RaycastVehicle({
    chassisBody: chassisBody
});

// เพิ่ม wheels
const wheelOptions = {
    radius: 0.5,
    directionLocal: new CANNON.Vec3(0, -1, 0),
    suspensionStiffness: 30,
    suspensionRestLength: 0.3,
    frictionSlip: 5,
    dampingRelaxation: 2.3,
    dampingCompression: 4.4,
    maxSuspensionForce: 100000,
    rollInfluence: 0.01,
    axleLocal: new CANNON.Vec3(0, 0, 1),
    chassisConnectionPointLocal: new CANNON.Vec3(0, 0, 0),
    maxSuspensionTravel: 0.3,
    customSlidingRotationalSpeed: -30,
    useCustomSlidingRotationalSpeed: true
};

// FL wheel
wheelOptions.chassisConnectionPointLocal.set(-1, 0.1, 1.5);
vehicle.addWheel(wheelOptions);

// FR wheel
wheelOptions.chassisConnectionPointLocal.set(1, 0.1, 1.5);
vehicle.addWheel(wheelOptions);

// RL wheel
wheelOptions.chassisConnectionPointLocal.set(-1, 0.1, -1.5);
vehicle.addWheel(wheelOptions);

// RR wheel
wheelOptions.chassisConnectionPointLocal.set(1, 0.1, -1.5);
vehicle.addWheel(wheelOptions);

vehicle.addToWorld(world);

// Control
const maxSteerVal = 0.5;
const maxForce = 1000;
const brakeForce = 1000000;

window.addEventListener('keydown', (e) => {
    switch (e.key) {
        case 'w': vehicle.applyEngineForce(-maxForce, 2); vehicle.applyEngineForce(-maxForce, 3); break;
        case 's': vehicle.applyEngineForce(maxForce, 2);  vehicle.applyEngineForce(maxForce, 3);  break;
        case 'a': vehicle.setSteeringValue(maxSteerVal, 0); vehicle.setSteeringValue(maxSteerVal, 1); break;
        case 'd': vehicle.setSteeringValue(-maxSteerVal, 0); vehicle.setSteeringValue(-maxSteerVal, 1); break;
    }
});

window.addEventListener('keyup', (e) => {
    switch (e.key) {
        case 'w': case 's': vehicle.applyEngineForce(0, 2); vehicle.applyEngineForce(0, 3); break;
        case 'a': case 'd': vehicle.setSteeringValue(0, 0); vehicle.setSteeringValue(0, 1); break;
    }
});
```

---

## แบบฝึกหัด

### ระดับ 1
1. สร้าง scene ที่มี sphere ตกลงบน plane
2. เพิ่ม boxes หลายๆ อัน ให้ชนกัน
3. ปรับ restitution ให้ลูกบอลเด้ง (0.9)

### ระดับ 2
4. สร้าง domino effect (อย่างน้อย 20 โดมิโน)
5. สร้าง constraints chain (chain link simulation)
6. เพิ่ม collision sound effects (Web Audio API)

### ระดับ 3
7. สร้าง ragdoll พื้นฐาน (body parts connected with constraints)
8. สร้าง vehicle ที่ขับได้ด้วย WASD
9. สร้าง wrecking ball (hinge + pendulum)

### Challenge
10. สร้าง "Tower of Blocks" game:
    - Stack blocks โดยกดปุ่ม
    - Block เลื่อนไปมาแบบสุ่ม
    - ต้องวางให้ตรง (timing-based)
    - Physics จัดการ blocks ที่ตกออกไป
    - Score system
    - Particle effects เมื่อ block ตก
