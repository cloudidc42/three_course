# Part 32: Physics with Rapier.js

## บทนำ: ทำไมต้อง Rapier?

**Rapier** คือ physics engine ที่เขียนด้วยภาษา **Rust** แล้ว compile เป็น **WebAssembly** ทำให้:
- **เร็วกว่า** Cannon.js มาก (native speed ผ่าน WASM)
- **Deterministic** - ผลเหมือนกันทุก platform
- **Modern API** - ออกแบบมาใหม่, ใช้งานง่ายกว่า
- **Active development** - maintain อยู่และมี features ใหม่

```bash
npm install @dimforge/rapier3d-compat
```

---

## Setup: @dimforge/rapier3d-compat

```javascript
// ต้อง init WASM ก่อน (async)
import RAPIER from '@dimforge/rapier3d-compat';

async function init() {
    // Init WASM module
    await RAPIER.init();
    
    // สร้าง world
    const gravity = { x: 0.0, y: -9.81, z: 0.0 };
    const world = new RAPIER.World(gravity);
    
    // ...
}

init();
```

หรือใช้ CDN:
```html
<script type="module">
    import('@dimforge/rapier3d-compat').then(RAPIER => {
        RAPIER.init().then(() => {
            startPhysics(RAPIER);
        });
    });
</script>
```

---

## RigidBody Types

```javascript
// 1. Dynamic - ถูก physics ควบคุม (มวล, แรง, collision)
const rigidBodyDesc = RAPIER.RigidBodyDesc.dynamic()
    .setTranslation(x, y, z)
    .setLinvel(0, 5, 0)           // initial velocity
    .setAngvel({ x: 0, y: 1, z: 0 })
    .setLinearDamping(0.1)
    .setAngularDamping(0.1);

const dynamicBody = world.createRigidBody(rigidBodyDesc);

// 2. Fixed - ไม่เคลื่อนที่ (ground, walls)
const fixedDesc = RAPIER.RigidBodyDesc.fixed()
    .setTranslation(0, 0, 0);
const fixedBody = world.createRigidBody(fixedDesc);

// 3. KinematicPositionBased - ขยับตาม position ที่เรากำหนด
const kinematicDesc = RAPIER.RigidBodyDesc.kinematicPositionBased()
    .setTranslation(0, 5, 0);
const kinematicBody = world.createRigidBody(kinematicDesc);

// อัพเดท kinematic position
kinematicBody.setNextKinematicTranslation({ x: 0, y: newY, z: 0 });

// 4. KinematicVelocityBased - ขยับตาม velocity ที่เรากำหนด
const kinVelDesc = RAPIER.RigidBodyDesc.kinematicVelocityBased();
const kinVelBody = world.createRigidBody(kinVelDesc);
kinVelBody.setLinvel({ x: 1, y: 0, z: 0 }, true); // wake up: true
```

---

## Collider Shapes

```javascript
// 1. Ball
const ballCollider = RAPIER.ColliderDesc.ball(radius);

// 2. Cuboid
const cuboidCollider = RAPIER.ColliderDesc.cuboid(halfX, halfY, halfZ);

// 3. Capsule
const capsuleCollider = RAPIER.ColliderDesc.capsule(halfHeight, radius);

// 4. Cylinder
const cylinderCollider = RAPIER.ColliderDesc.cylinder(halfHeight, radius);

// 5. Cone
const coneCollider = RAPIER.ColliderDesc.cone(halfHeight, radius);

// 6. ConvexHull (arbitrary convex shape)
const convexCollider = RAPIER.ColliderDesc.convexHull(pointsFloat32Array);

// 7. TriMesh (arbitrary mesh - static only)
const trimeshCollider = RAPIER.ColliderDesc.trimesh(
    verticesFloat32Array,  // (x,y,z, x,y,z, ...)
    indicesUint32Array     // (i0,i1,i2, ...)
);

// 8. HeightField (terrain)
const heightfieldCollider = RAPIER.ColliderDesc.heightfield(
    nrows, ncols,
    heightData,  // Float32Array
    { x: scaleX, y: scaleY, z: scaleZ }
);

// Attach collider to body
const collider = world.createCollider(ballCollider, rigidBody);

// Properties
const colliderDesc = RAPIER.ColliderDesc.ball(0.5)
    .setFriction(0.7)
    .setRestitution(0.3)
    .setDensity(1.0)
    .setSensor(false);  // sensor = ตรวจจับ collision แต่ไม่ตอบสนองทางฟิสิกส์
```

---

## Compound Colliders

```javascript
// หลาย colliders ต่อ 1 body
const bodyDesc = RAPIER.RigidBodyDesc.dynamic().setTranslation(0, 5, 0);
const body = world.createRigidBody(bodyDesc);

// Body collider
world.createCollider(
    RAPIER.ColliderDesc.cuboid(0.5, 0.5, 0.5),
    body
);

// Head collider (offset จาก body center)
world.createCollider(
    RAPIER.ColliderDesc.ball(0.4).setTranslation(0, 1.2, 0),
    body
);
```

---

## Physics World Step

```javascript
// Rapier ใช้ fixed timestep ภายใน
// เรียก world.step() ทุก frame

const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    
    // Step physics (Rapier handles substeps internally)
    world.step();
    
    // Sync Three.js meshes
    for (const { mesh, body } of syncObjects) {
        const pos = body.translation();
        const rot = body.rotation();
        
        mesh.position.set(pos.x, pos.y, pos.z);
        mesh.quaternion.set(rot.x, rot.y, rot.z, rot.w);
    }
    
    renderer.render(scene, camera);
}
```

---

## Collision Events

```javascript
// Rapier collision events ผ่าน Event Queue
const eventQueue = new RAPIER.EventQueue(true);

// ตั้ง active collision events บน collider
const colliderDesc = RAPIER.ColliderDesc.ball(0.5)
    .setActiveEvents(RAPIER.ActiveEvents.COLLISION_EVENTS);
world.createCollider(colliderDesc, body);

// ใน animation loop
function animate() {
    world.step(eventQueue);
    
    // ประมวลผล events
    eventQueue.drainCollisionEvents((handle1, handle2, started) => {
        if (started) {
            console.log('Collision started:', handle1, handle2);
            // ทำอะไรบางอย่าง เช่น เล่นเสียง, เปลี่ยนสี
        } else {
            console.log('Collision ended');
        }
    });
    
    // Intersection events (sensor)
    eventQueue.drainIntersectionEvents((handle1, handle2, intersecting) => {
        if (intersecting) {
            console.log('Sensor triggered');
        }
    });
}
```

---

## Character Controller

```javascript
// Rapier มี built-in character controller
const characterController = world.createCharacterController(0.01); // offset

characterController.setUp({ x: 0, y: 1, z: 0 });
characterController.setMaxSlopeClimbAngle(45 * Math.PI / 180);
characterController.setMinSlopeSlideAngle(30 * Math.PI / 180);
characterController.enableAutostep(0.5, 0.2, true);
characterController.enableSnapToGround(0.5);

// ใน animation loop
const desiredMovement = { x: moveX, y: -gravity * delta, z: moveZ };
characterController.computeColliderMovement(
    playerCollider,
    desiredMovement,
    RAPIER.QueryFilterFlags.EXCLUDE_SENSORS,
    undefined
);

const correctedMovement = characterController.computedMovement();
const currentPos = playerBody.translation();

playerBody.setNextKinematicTranslation({
    x: currentPos.x + correctedMovement.x,
    y: currentPos.y + correctedMovement.y,
    z: currentPos.z + correctedMovement.z
});
```

---

## Joints ใน Rapier

```javascript
// 1. Fixed Joint (Weld)
const fixedJoint = RAPIER.JointData.fixed(
    { x: 0, y: 0.5, z: 0 },   // anchor1 (local to body1)
    { x: 0, y: 0, z: 0 },      // frame1
    { x: 0, y: -0.5, z: 0 },  // anchor2
    { x: 0, y: 0, z: 0 }       // frame2
);
world.createImpulseJoint(fixedJoint, body1, body2, true);

// 2. Ball Joint (Spherical)
const ballJoint = RAPIER.JointData.spherical(
    { x: 0, y: 0.5, z: 0 },   // anchor1
    { x: 0, y: -0.5, z: 0 }   // anchor2
);
world.createImpulseJoint(ballJoint, body1, body2, true);

// 3. Revolute Joint (Hinge)
const revJoint = RAPIER.JointData.revolute(
    { x: 0, y: 0.5, z: 0 },   // anchor1
    { x: 0, y: -0.5, z: 0 },  // anchor2
    { x: 0, y: 0, z: 1 }       // axis
);
world.createImpulseJoint(revJoint, body1, body2, true);

// 4. Prismatic Joint (Slider)
const prisJoint = RAPIER.JointData.prismatic(
    { x: 0, y: 0, z: 0 },
    { x: 0, y: 0, z: 0 },
    { x: 0, y: 1, z: 0 }      // slide axis
);
world.createImpulseJoint(prisJoint, body1, body2, true);
```

---

## Continuous Collision Detection (CCD)

```javascript
// CCD ป้องกัน fast-moving objects ทะลุผ่านกัน (tunneling)
const fastBulletDesc = RAPIER.RigidBodyDesc.dynamic()
    .setTranslation(0, 0, 0)
    .setCcdEnabled(true);  // เปิด CCD สำหรับ body นี้

const bulletBody = world.createRigidBody(fastBulletDesc);

// ยิงกระสุน
bulletBody.setLinvel({ x: 0, y: 0, z: -100 }, true); // 100 m/s!

// โดยไม่มี CCD: กระสุนอาจทะลุผ่าน wall บางๆ ได้
// ด้วย CCD: Rapier จะตรวจสอบ motion ตลอดทาง
```

---

## Demo: Complete Rapier Scene

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Rapier Physics Demo</title>
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
            margin: 4px 0;
            padding: 8px 15px;
            background: #2a5a4a;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        button:hover { background: #3a7a6a; }
        #loading {
            position: fixed;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            color: white;
            font-family: monospace;
            font-size: 20px;
        }
    </style>
</head>
<body>
<div id="loading">Loading Rapier WASM...</div>
<div id="ui" style="display:none">
    <b>Rapier Physics</b>
    <button id="addBall">Add Ball</button>
    <button id="addBox">Add Box</button>
    <button id="addCapsule">Add Capsule</button>
    <button id="impulse">Impulse All!</button>
    <button id="reset">Reset</button>
    <div id="stats">Bodies: 0 | FPS: --</div>
</div>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';
import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.157.0/examples/jsm/controls/OrbitControls.js';

// Load Rapier
let RAPIER;
try {
    const module = await import('https://cdn.skypack.dev/@dimforge/rapier3d-compat');
    RAPIER = module.default;
    await RAPIER.init();
} catch (e) {
    // Fallback: show error
    document.getElementById('loading').textContent = 'Failed to load Rapier. Please use npm instead.';
    throw e;
}

document.getElementById('loading').style.display = 'none';
document.getElementById('ui').style.display = 'block';

// === Three.js ===
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x0a1a2e);
scene.fog = new THREE.Fog(0x0a1a2e, 30, 80);

const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 200);
camera.position.set(0, 10, 20);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.target.set(0, 3, 0);

// Lights
scene.add(new THREE.AmbientLight(0xffffff, 0.5));
const dirLight = new THREE.DirectionalLight(0xffffff, 1);
dirLight.position.set(10, 20, 10);
dirLight.castShadow = true;
dirLight.shadow.mapSize.set(2048, 2048);
scene.add(dirLight);

// === Rapier World ===
const world = new RAPIER.World({ x: 0, y: -9.81, z: 0 });
const eventQueue = new RAPIER.EventQueue(true);

const syncObjects = [];
const bodyMeshMap = new Map(); // body handle → mesh

// === Ground ===
const groundMesh = new THREE.Mesh(
    new THREE.BoxGeometry(30, 0.5, 30),
    new THREE.MeshStandardMaterial({ color: 0x2d4e3d })
);
groundMesh.position.y = -0.25;
groundMesh.receiveShadow = true;
scene.add(groundMesh);
scene.add(new THREE.GridHelper(30, 30, 0x336644, 0x224433));

const groundBody = world.createRigidBody(
    RAPIER.RigidBodyDesc.fixed().setTranslation(0, -0.25, 0)
);
world.createCollider(RAPIER.ColliderDesc.cuboid(15, 0.25, 15), groundBody);

// Walls
function addWall(tx, ty, tz, hw, hh, hd) {
    const wallMesh = new THREE.Mesh(
        new THREE.BoxGeometry(hw * 2, hh * 2, hd * 2),
        new THREE.MeshStandardMaterial({ color: 0x2d3e5d, transparent: true, opacity: 0.3 })
    );
    wallMesh.position.set(tx, ty, tz);
    scene.add(wallMesh);
    
    const wb = world.createRigidBody(RAPIER.RigidBodyDesc.fixed().setTranslation(tx, ty, tz));
    world.createCollider(RAPIER.ColliderDesc.cuboid(hw, hh, hd), wb);
}
addWall(0, 5, -12, 15, 6, 0.3);
addWall(0, 5, 12, 15, 6, 0.3);
addWall(-12, 5, 0, 0.3, 6, 12);
addWall(12, 5, 0, 0.3, 6, 12);

// Colors
const colors = [0xff5555, 0x55ff55, 0x5555ff, 0xffff55, 0xff55ff, 0x55ffff, 0xff9955];
let colorIdx = 0;

function addPhysicsBall(position = new THREE.Vector3(0, 10, 0)) {
    const r = 0.3 + Math.random() * 0.4;
    const color = colors[colorIdx++ % colors.length];
    
    const mesh = new THREE.Mesh(
        new THREE.SphereGeometry(r, 20, 20),
        new THREE.MeshStandardMaterial({ color, roughness: 0.3, metalness: 0.2 })
    );
    mesh.castShadow = true;
    scene.add(mesh);
    
    const rbDesc = RAPIER.RigidBodyDesc.dynamic()
        .setTranslation(
            position.x + (Math.random() - 0.5) * 3,
            position.y,
            position.z + (Math.random() - 0.5) * 3
        )
        .setLinvel(
            (Math.random() - 0.5) * 5,
            3,
            (Math.random() - 0.5) * 5
        );
    
    const body = world.createRigidBody(rbDesc);
    const collDesc = RAPIER.ColliderDesc.ball(r)
        .setFriction(0.5)
        .setRestitution(0.4)
        .setActiveEvents(RAPIER.ActiveEvents.COLLISION_EVENTS);
    world.createCollider(collDesc, body);
    
    syncObjects.push({ mesh, body });
    bodyMeshMap.set(body.handle, mesh);
    updateStats();
}

function addPhysicsBox(position = new THREE.Vector3(0, 10, 0)) {
    const s = 0.3 + Math.random() * 0.5;
    const color = colors[colorIdx++ % colors.length];
    
    const mesh = new THREE.Mesh(
        new THREE.BoxGeometry(s, s, s),
        new THREE.MeshStandardMaterial({ color, roughness: 0.5, metalness: 0.1 })
    );
    mesh.castShadow = true;
    scene.add(mesh);
    
    const h = s / 2;
    const rbDesc = RAPIER.RigidBodyDesc.dynamic()
        .setTranslation(
            position.x + (Math.random() - 0.5) * 3,
            position.y,
            position.z + (Math.random() - 0.5) * 3
        )
        .setAngvel({ x: Math.random(), y: Math.random(), z: Math.random() });
    
    const body = world.createRigidBody(rbDesc);
    world.createCollider(
        RAPIER.ColliderDesc.cuboid(h, h, h).setFriction(0.7).setRestitution(0.2),
        body
    );
    
    syncObjects.push({ mesh, body });
    bodyMeshMap.set(body.handle, mesh);
    updateStats();
}

function addPhysicsCapsule(position = new THREE.Vector3(0, 10, 0)) {
    const h = 0.5, r = 0.2;
    const color = colors[colorIdx++ % colors.length];
    
    const mesh = new THREE.Mesh(
        new THREE.CapsuleGeometry(r, h, 8, 16),
        new THREE.MeshStandardMaterial({ color, roughness: 0.4 })
    );
    mesh.castShadow = true;
    scene.add(mesh);
    
    const rbDesc = RAPIER.RigidBodyDesc.dynamic()
        .setTranslation(
            position.x + (Math.random() - 0.5) * 3,
            position.y,
            position.z + (Math.random() - 0.5) * 3
        );
    
    const body = world.createRigidBody(rbDesc);
    world.createCollider(
        RAPIER.ColliderDesc.capsule(h / 2, r).setFriction(0.5).setRestitution(0.3),
        body
    );
    
    syncObjects.push({ mesh, body });
    bodyMeshMap.set(body.handle, mesh);
    updateStats();
}

// Init with objects
for (let i = 0; i < 5; i++) addPhysicsBall(new THREE.Vector3(0, 8 + i, 0));
for (let i = 0; i < 5; i++) addPhysicsBox(new THREE.Vector3(2, 8 + i, 0));
for (let i = 0; i < 5; i++) addPhysicsCapsule(new THREE.Vector3(-2, 8 + i, 0));

// Impulse all
function impulseAll() {
    for (const { body } of syncObjects) {
        body.applyImpulse(
            {
                x: (Math.random() - 0.5) * 20,
                y: 10 + Math.random() * 5,
                z: (Math.random() - 0.5) * 20
            },
            true
        );
    }
}

// Reset
function reset() {
    for (const { mesh, body } of syncObjects) {
        scene.remove(mesh);
        world.removeRigidBody(body);
    }
    syncObjects.length = 0;
    bodyMeshMap.clear();
    colorIdx = 0;
    
    for (let i = 0; i < 5; i++) addPhysicsBall(new THREE.Vector3(0, 8 + i, 0));
    for (let i = 0; i < 5; i++) addPhysicsBox(new THREE.Vector3(2, 8 + i, 0));
    for (let i = 0; i < 5; i++) addPhysicsCapsule(new THREE.Vector3(-2, 8 + i, 0));
}

// UI events
document.getElementById('addBall').onclick    = () => addPhysicsBall(new THREE.Vector3(0, 12, 0));
document.getElementById('addBox').onclick     = () => addPhysicsBox(new THREE.Vector3(0, 12, 0));
document.getElementById('addCapsule').onclick = () => addPhysicsCapsule(new THREE.Vector3(0, 12, 0));
document.getElementById('impulse').onclick    = impulseAll;
document.getElementById('reset').onclick      = reset;

// FPS tracking
let frameCount = 0;
let lastTime = performance.now();

function updateStats() {
    document.getElementById('stats').textContent =
        `Bodies: ${syncObjects.length} | FPS: --`;
}

// Animation
function animate() {
    requestAnimationFrame(animate);
    
    // Rapier step
    world.step(eventQueue);
    
    // Process collision events
    eventQueue.drainCollisionEvents((h1, h2, started) => {
        if (started) {
            // Flash color on collision (optional effect)
            const mesh = bodyMeshMap.get(h1) || bodyMeshMap.get(h2);
            if (mesh) {
                const originalColor = mesh.material.color.clone();
                mesh.material.color.set(0xffffff);
                setTimeout(() => mesh.material.color.copy(originalColor), 100);
            }
        }
    });
    
    // Sync meshes
    for (const { mesh, body } of syncObjects) {
        const pos = body.translation();
        const rot = body.rotation();
        mesh.position.set(pos.x, pos.y, pos.z);
        mesh.quaternion.set(rot.x, rot.y, rot.z, rot.w);
    }
    
    // FPS counter
    frameCount++;
    const now = performance.now();
    if (now - lastTime > 500) {
        const fps = Math.round(frameCount * 1000 / (now - lastTime));
        document.getElementById('stats').textContent =
            `Bodies: ${syncObjects.length} | FPS: ${fps}`;
        frameCount = 0;
        lastTime = now;
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

## Demo: Character Movement with Physics

```javascript
// Character Controller ที่สมบูรณ์กว่า
class PhysicsCharacter {
    constructor(rapier, world, scene, startPos = { x: 0, y: 2, z: 0 }) {
        this.rapier = rapier;
        this.world = world;
        
        // Kinematic body
        const bodyDesc = rapier.RigidBodyDesc
            .kinematicPositionBased()
            .setTranslation(startPos.x, startPos.y, startPos.z);
        
        this.body = world.createRigidBody(bodyDesc);
        
        // Capsule collider สำหรับ character
        const colliderDesc = rapier.ColliderDesc
            .capsule(0.5, 0.3)
            .setFriction(0.0)  // ไม่มี friction บน capsule
            .setActiveEvents(rapier.ActiveEvents.COLLISION_EVENTS);
        
        this.collider = world.createCollider(colliderDesc, this.body);
        
        // Character controller
        this.controller = world.createCharacterController(0.02);
        this.controller.setUp({ x: 0, y: 1, z: 0 });
        this.controller.setMaxSlopeClimbAngle(45 * Math.PI / 180);
        this.controller.setMinSlopeSlideAngle(30 * Math.PI / 180);
        this.controller.enableAutostep(0.3, 0.1, true);
        this.controller.enableSnapToGround(0.3);
        
        // State
        this.velocity = { x: 0, y: 0, z: 0 };
        this.isOnGround = false;
        this.jumpCooldown = 0;
        
        // Visual
        const geo = new THREE.CapsuleGeometry(0.3, 1.0, 8, 16);
        const mat = new THREE.MeshStandardMaterial({ color: 0x4488ff });
        this.mesh = new THREE.Mesh(geo, mat);
        this.mesh.castShadow = true;
        scene.add(this.mesh);
    }
    
    update(delta, input) {
        const SPEED = 5;
        const JUMP_FORCE = 8;
        const GRAVITY = -20;
        
        // Horizontal movement
        const moveX = (input.right - input.left) * SPEED;
        const moveZ = (input.backward - input.forward) * SPEED;
        
        // Apply gravity
        this.velocity.y += GRAVITY * delta;
        
        // Jump
        if (input.jump && this.isOnGround && this.jumpCooldown <= 0) {
            this.velocity.y = JUMP_FORCE;
            this.jumpCooldown = 0.3;
        }
        
        this.jumpCooldown -= delta;
        
        // Desired movement
        const desiredMovement = {
            x: moveX * delta,
            y: this.velocity.y * delta,
            z: moveZ * delta
        };
        
        // Let character controller handle collision
        this.controller.computeColliderMovement(
            this.collider,
            desiredMovement
        );
        
        const corrected = this.controller.computedMovement();
        const currentPos = this.body.translation();
        
        // Check if on ground (y movement was corrected)
        this.isOnGround = this.controller.computedGrounded();
        
        if (this.isOnGround) {
            this.velocity.y = Math.max(0, this.velocity.y);
        }
        
        // Apply movement
        this.body.setNextKinematicTranslation({
            x: currentPos.x + corrected.x,
            y: currentPos.y + corrected.y,
            z: currentPos.z + corrected.z
        });
        
        // Sync mesh
        const pos = this.body.translation();
        this.mesh.position.set(pos.x, pos.y, pos.z);
        
        return { x: pos.x, y: pos.y, z: pos.z };
    }
    
    destroy() {
        this.world.removeCharacterController(this.controller);
        this.world.removeRigidBody(this.body);
    }
}
```

---

## Stacking Objects Demo

```javascript
// สร้าง tower of blocks
async function buildTower(world, scene, RAPIER, floors = 5, blocksPerFloor = 4) {
    const syncObjects = [];
    
    for (let floor = 0; floor < floors; floor++) {
        const y = 0.5 + floor * 1.0;
        const offsetAngle = (floor % 2) * (Math.PI / blocksPerFloor);
        
        for (let b = 0; b < blocksPerFloor; b++) {
            const angle = (b / blocksPerFloor) * Math.PI * 2 + offsetAngle;
            const r = 1.5;
            const x = Math.cos(angle) * r;
            const z = Math.sin(angle) * r;
            
            // Three.js
            const mesh = new THREE.Mesh(
                new THREE.BoxGeometry(0.8, 0.8, 0.8),
                new THREE.MeshStandardMaterial({
                    color: new THREE.Color().setHSL(floor / floors, 0.7, 0.5)
                })
            );
            mesh.position.set(x, y, z);
            mesh.castShadow = true;
            scene.add(mesh);
            
            // Rapier
            const bodyDesc = RAPIER.RigidBodyDesc.dynamic()
                .setTranslation(x, y, z);
            const body = world.createRigidBody(bodyDesc);
            world.createCollider(
                RAPIER.ColliderDesc.cuboid(0.4, 0.4, 0.4)
                    .setFriction(0.8)
                    .setRestitution(0.1),
                body
            );
            
            syncObjects.push({ mesh, body });
        }
    }
    
    return syncObjects;
}
```

---

## Comparison: Cannon.js vs Rapier

| Feature | Cannon.js | Rapier |
|---------|-----------|--------|
| Language | JavaScript | Rust → WASM |
| Speed | ปานกลาง | เร็วกว่ามาก |
| Deterministic | ไม่ | ใช่ |
| Setup | ง่าย | ต้อง await init |
| API | ดี | ดีกว่า (modern) |
| CCD | ไม่มี | มี |
| Character Ctrl | ไม่มี (ต้องทำเอง) | มี built-in |
| File Size | เล็ก | ใหญ่กว่า (WASM) |
| Community | ใหญ่กว่า | กำลังเติบโต |

---

## แบบฝึกหัด

### ระดับ 1
1. Setup Rapier และสร้าง sphere ตกลงบน ground
2. เพิ่ม boxes และ capsules ที่กองบน ground
3. ทดลอง friction และ restitution ต่างๆ

### ระดับ 2
4. สร้าง Rapier character controller ขั้นพื้นฐาน
5. สร้าง domino effect ด้วย Rapier
6. ทดสอบ CCD ด้วย fast-moving projectile

### ระดับ 3
7. สร้าง revolute joint pendulum
8. สร้าง compound collider (body + head + arms)
9. Implement collision sound effects ผ่าน collision events

### Challenge
10. สร้าง Physics Playground:
    - Character controller ที่เดินได้
    - Jump, crouch, sprint
    - Pushable objects
    - Trampolines (high restitution)
    - Sticky surfaces (high friction)
    - Conveyor belts (kinematic bodies)
    - Breakable walls (collider removal on impact)
    - Score system
