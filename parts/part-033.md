# Part 33: Physics Advanced – Soft Bodies, Cloth, Rope & Vehicle

## Soft Bodies ใน Cannon.js / Rapier

cannon-es ไม่มี soft body support ในตัว แต่เราสามารถ **simulate** ได้ด้วย:
- **Spring-Mass System** (สปริงเชื่อมระหว่าง particles)
- ใช้ PointToPointConstraint จำนวนมาก

---

## Cloth Simulation ด้วย Spring-Mass System

### แนวคิด
```
P(0,0) -- P(1,0) -- P(2,0)
  |          |          |
P(0,1) -- P(1,1) -- P(2,1)
  |          |          |
P(0,2) -- P(1,2) -- P(2,2)
```
- แต่ละจุด = CANNON.Body ขนาดเล็ก
- เส้นเชื่อม = CANNON.DistanceConstraint (รักษาระยะ)
- มุมบนสุด 2 จุด = static (ตรึงไว้)

---

### Full Cloth Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Cloth Simulation</title>
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
            padding: 10px;
            border-radius: 8px;
        }
        button {
            display: block;
            margin: 4px 0;
            padding: 6px 14px;
            background: #3a5a8a;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        button:hover { background: #5a7aaa; }
    </style>
</head>
<body>
<div id="ui">
    <b>Cloth Physics</b>
    <button id="wind">Toggle Wind</button>
    <button id="ball">Drop Ball</button>
    <button id="reset">Reset Cloth</button>
    <div id="info">Particles: --</div>
</div>
<script src="https://cdn.jsdelivr.net/npm/cannon-es@0.20.0/dist/cannon-es.js"></script>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';
import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.157.0/examples/jsm/controls/OrbitControls.js';

// === Three.js Setup ===
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x1a1a2e);
const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 200);
camera.position.set(0, 5, 15);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.shadowMap.enabled = true;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.target.set(0, 3, 0);

// Lights
scene.add(new THREE.AmbientLight(0xffffff, 0.4));
const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
dirLight.position.set(5, 15, 5);
dirLight.castShadow = true;
scene.add(dirLight);

// Ground
const groundMesh = new THREE.Mesh(
    new THREE.PlaneGeometry(40, 40),
    new THREE.MeshStandardMaterial({ color: 0x2d3e2d })
);
groundMesh.rotation.x = -Math.PI / 2;
groundMesh.receiveShadow = true;
scene.add(groundMesh);

// === Cannon.js World ===
const world = new CANNON.World({ gravity: new CANNON.Vec3(0, -9.81, 0) });
world.broadphase = new CANNON.SAPBroadphase(world);
world.defaultContactMaterial.friction = 0.3;

// Ground body
const groundBody = new CANNON.Body({
    mass: 0,
    shape: new CANNON.Plane(),
    position: new CANNON.Vec3(0, 0, 0)
});
groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
world.addBody(groundBody);

// === Cloth Parameters ===
const CLOTH_W = 12;        // จำนวน particles แนวนอน
const CLOTH_H = 10;        // จำนวน particles แนวตั้ง
const CLOTH_SIZE_W = 6;    // ขนาดจริง (meters)
const CLOTH_SIZE_H = 5;
const PARTICLE_MASS = 0.1;
const DAMPING = 0.03;

let particles = [];       // CANNON.Body[][]
let clothGeo;
let clothMesh;
let windEnabled = false;

// ตาแหน่ง UV ของแต่ละ vertex ใน BufferGeometry
// ผ้าจะถูก subdivide เป็น CLOTH_W x CLOTH_H grid ของ quads

function buildCloth() {
    // ล้าง bodies เก่า
    for (const row of particles) {
        for (const p of row) world.removeBody(p);
    }
    particles = [];

    // สร้าง particles
    for (let row = 0; row < CLOTH_H; row++) {
        particles[row] = [];
        for (let col = 0; col < CLOTH_W; col++) {
            const x = (col / (CLOTH_W - 1) - 0.5) * CLOTH_SIZE_W;
            const y = 8 - (row / (CLOTH_H - 1)) * CLOTH_SIZE_H;
            const z = 0;

            const isFixed = row === 0 && (col === 0 || col === CLOTH_W - 1);
            const body = new CANNON.Body({
                mass: isFixed ? 0 : PARTICLE_MASS,
                position: new CANNON.Vec3(x, y, z),
                shape: new CANNON.Sphere(0.05),
                linearDamping: DAMPING,
                angularDamping: DAMPING
            });
            world.addBody(body);
            particles[row][col] = body;
        }
    }

    // สร้าง constraints
    // Structural springs (horizontal + vertical)
    for (let row = 0; row < CLOTH_H; row++) {
        for (let col = 0; col < CLOTH_W; col++) {
            if (col < CLOTH_W - 1) {
                const c = new CANNON.DistanceConstraint(
                    particles[row][col],
                    particles[row][col + 1]
                );
                world.addConstraint(c);
            }
            if (row < CLOTH_H - 1) {
                const c = new CANNON.DistanceConstraint(
                    particles[row][col],
                    particles[row + 1][col]
                );
                world.addConstraint(c);
            }
        }
    }

    // Shear springs (diagonal)
    for (let row = 0; row < CLOTH_H - 1; row++) {
        for (let col = 0; col < CLOTH_W - 1; col++) {
            const c1 = new CANNON.DistanceConstraint(
                particles[row][col],
                particles[row + 1][col + 1]
            );
            world.addConstraint(c1);
            const c2 = new CANNON.DistanceConstraint(
                particles[row + 1][col],
                particles[row][col + 1]
            );
            world.addConstraint(c2);
        }
    }

    // Bending springs (2 apart)
    for (let row = 0; row < CLOTH_H; row++) {
        for (let col = 0; col < CLOTH_W - 2; col++) {
            const c = new CANNON.DistanceConstraint(
                particles[row][col],
                particles[row][col + 2]
            );
            world.addConstraint(c);
        }
    }
    for (let row = 0; row < CLOTH_H - 2; row++) {
        for (let col = 0; col < CLOTH_W; col++) {
            const c = new CANNON.DistanceConstraint(
                particles[row][col],
                particles[row + 2][col]
            );
            world.addConstraint(c);
        }
    }

    document.getElementById('info').textContent = 
        `Particles: ${CLOTH_W * CLOTH_H} | Constraints: ${world.constraints.length}`;
}

function buildClothMesh() {
    // สร้าง BufferGeometry สำหรับผ้า
    const numVerts = CLOTH_W * CLOTH_H;
    const positions = new Float32Array(numVerts * 3);
    const normals   = new Float32Array(numVerts * 3);
    const uvs       = new Float32Array(numVerts * 2);

    for (let row = 0; row < CLOTH_H; row++) {
        for (let col = 0; col < CLOTH_W; col++) {
            const i = row * CLOTH_W + col;
            uvs[i * 2]     = col / (CLOTH_W - 1);
            uvs[i * 2 + 1] = 1 - row / (CLOTH_H - 1);
        }
    }

    // Indices (triangles)
    const numQuads = (CLOTH_W - 1) * (CLOTH_H - 1);
    const indices = new Uint16Array(numQuads * 6);
    let idx = 0;
    for (let row = 0; row < CLOTH_H - 1; row++) {
        for (let col = 0; col < CLOTH_W - 1; col++) {
            const tl = row * CLOTH_W + col;
            const tr = tl + 1;
            const bl = tl + CLOTH_W;
            const br = bl + 1;
            indices[idx++] = tl; indices[idx++] = bl; indices[idx++] = tr;
            indices[idx++] = tr; indices[idx++] = bl; indices[idx++] = br;
        }
    }

    clothGeo = new THREE.BufferGeometry();
    clothGeo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    clothGeo.setAttribute('normal',   new THREE.BufferAttribute(normals,   3));
    clothGeo.setAttribute('uv',       new THREE.BufferAttribute(uvs,       2));
    clothGeo.setIndex(new THREE.BufferAttribute(indices, 1));

    // Texture
    const canvas = document.createElement('canvas');
    canvas.width = canvas.height = 256;
    const ctx = canvas.getContext('2d');
    ctx.fillStyle = '#cc4444';
    ctx.fillRect(0, 0, 256, 256);
    ctx.strokeStyle = '#ff8888';
    ctx.lineWidth = 4;
    for (let i = 0; i <= 8; i++) {
        ctx.beginPath();
        ctx.moveTo(i * 32, 0);
        ctx.lineTo(i * 32, 256);
        ctx.stroke();
        ctx.beginPath();
        ctx.moveTo(0, i * 32);
        ctx.lineTo(256, i * 32);
        ctx.stroke();
    }
    const tex = new THREE.CanvasTexture(canvas);

    clothMesh = new THREE.Mesh(
        clothGeo,
        new THREE.MeshStandardMaterial({
            map: tex,
            side: THREE.DoubleSide,
            roughness: 0.9
        })
    );
    clothMesh.castShadow = true;
    scene.add(clothMesh);
}

function updateClothGeometry() {
    const pos = clothGeo.attributes.position;
    for (let row = 0; row < CLOTH_H; row++) {
        for (let col = 0; col < CLOTH_W; col++) {
            const i = row * CLOTH_W + col;
            const p = particles[row][col].position;
            pos.setXYZ(i, p.x, p.y, p.z);
        }
    }
    pos.needsUpdate = true;
    clothGeo.computeVertexNormals();
}

// Ball obstacle
let ballBody = null;
let ballMesh = null;

function dropBall() {
    if (ballBody) {
        world.removeBody(ballBody);
        scene.remove(ballMesh);
    }
    const r = 0.8;
    ballMesh = new THREE.Mesh(
        new THREE.SphereGeometry(r, 24, 24),
        new THREE.MeshStandardMaterial({ color: 0x4488ff, metalness: 0.5, roughness: 0.3 })
    );
    ballMesh.castShadow = true;
    scene.add(ballMesh);

    ballBody = new CANNON.Body({
        mass: 5,
        shape: new CANNON.Sphere(r),
        position: new CANNON.Vec3(
            (Math.random() - 0.5) * 4,
            12,
            (Math.random() - 0.5) * 2
        ),
        linearDamping: 0.01
    });
    world.addBody(ballBody);
}

// Init
buildCloth();
buildClothMesh();

// UI
document.getElementById('wind').onclick  = () => { windEnabled = !windEnabled; };
document.getElementById('ball').onclick  = dropBall;
document.getElementById('reset').onclick = () => {
    if (ballBody) { world.removeBody(ballBody); scene.remove(ballMesh); ballBody = null; ballMesh = null; }
    buildCloth();
};

// Animation
const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    const delta = Math.min(clock.getDelta(), 0.05);

    // Wind force
    if (windEnabled) {
        const t = Date.now() * 0.001;
        const windStrength = Math.sin(t * 1.3) * 3 + Math.sin(t * 2.7) * 1.5;
        for (const row of particles) {
            for (const p of row) {
                if (p.mass > 0) {
                    p.applyForce(
                        new CANNON.Vec3(
                            windStrength,
                            Math.sin(t * 3) * 0.5,
                            Math.cos(t * 0.8) * windStrength * 0.3
                        )
                    );
                }
            }
        }
    }

    world.step(1 / 60, delta, 3);
    updateClothGeometry();

    if (ballBody && ballMesh) {
        ballMesh.position.copy(ballBody.position);
        ballMesh.quaternion.copy(ballBody.quaternion);
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

## Rope Simulation

```javascript
class PhysicsRope {
    constructor(world, scene, segments = 15, length = 5) {
        this.world = world;
        this.segments = segments;
        this.bodies = [];
        this.constraints = [];

        const segLen = length / segments;
        const radius = 0.06;

        // Create bodies
        for (let i = 0; i <= segments; i++) {
            const y = 8 - i * segLen;
            const body = new CANNON.Body({
                mass: i === 0 ? 0 : 0.05,   // first = fixed
                shape: new CANNON.Sphere(radius),
                position: new CANNON.Vec3(0, y, 0),
                linearDamping: 0.05,
                angularDamping: 0.8
            });
            world.addBody(body);
            this.bodies.push(body);
        }

        // Connect with P2P constraints
        for (let i = 0; i < segments; i++) {
            const a = this.bodies[i];
            const b = this.bodies[i + 1];
            const half = segLen / 2;

            const c = new CANNON.PointToPointConstraint(
                a, new CANNON.Vec3(0, -half, 0),
                b, new CANNON.Vec3(0,  half, 0)
            );
            world.addConstraint(c);
            this.constraints.push(c);
        }

        // Three.js visual (tube along rope)
        this.curve = new THREE.CatmullRomCurve3(
            this.bodies.map(b => new THREE.Vector3(b.position.x, b.position.y, b.position.z))
        );
        this.geometry = new THREE.TubeGeometry(this.curve, segments * 4, 0.06, 8, false);
        this.mesh = new THREE.Mesh(
            this.geometry,
            new THREE.MeshStandardMaterial({ color: 0xc07830, roughness: 0.8 })
        );
        this.mesh.castShadow = true;
        scene.add(this.mesh);
    }

    update() {
        // Update curve points
        const pts = this.bodies.map(b =>
            new THREE.Vector3(b.position.x, b.position.y, b.position.z)
        );
        this.curve.points = pts;

        // Rebuild tube geometry (expensive but simple)
        this.geometry.dispose();
        this.geometry = new THREE.TubeGeometry(this.curve, this.segments * 4, 0.06, 8, false);
        this.mesh.geometry = this.geometry;
    }

    // Attach a weight to the end
    attachWeight(mass = 2) {
        const lastBody = this.bodies[this.bodies.length - 1];
        lastBody.mass = mass;
        lastBody.updateMassProperties();
    }

    destroy() {
        for (const c of this.constraints) this.world.removeConstraint(c);
        for (const b of this.bodies)      this.world.removeBody(b);
    }
}
```

---

## Wrecking Ball Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Wrecking Ball</title>
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
            padding: 10px;
            border-radius: 8px;
        }
        button {
            display: block;
            margin: 4px 0;
            padding: 6px 14px;
            background: #5a3a2a;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        button:hover { background: #7a5a4a; }
    </style>
</head>
<body>
<div id="ui">
    <b>Wrecking Ball</b>
    <button id="swing">Swing!</button>
    <button id="build">Rebuild Tower</button>
    <div id="info">Click "Swing!" to start</div>
</div>
<script src="https://cdn.jsdelivr.net/npm/cannon-es@0.20.0/dist/cannon-es.js"></script>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';
import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.157.0/examples/jsm/controls/OrbitControls.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x1a1a2e);
const camera = new THREE.PerspectiveCamera(65, innerWidth / innerHeight, 0.1, 200);
camera.position.set(0, 8, 22);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.shadowMap.enabled = true;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.target.set(0, 4, 0);

scene.add(new THREE.AmbientLight(0xffffff, 0.5));
const dirLight = new THREE.DirectionalLight(0xffffff, 1);
dirLight.position.set(10, 20, 5);
dirLight.castShadow = true;
dirLight.shadow.camera.far = 50;
dirLight.shadow.mapSize.set(2048, 2048);
scene.add(dirLight);

const world = new CANNON.World({ gravity: new CANNON.Vec3(0, -15, 0) });
world.broadphase = new CANNON.SAPBroadphase(world);
world.defaultContactMaterial.friction = 0.4;

// Ground
const groundMesh = new THREE.Mesh(
    new THREE.PlaneGeometry(50, 50),
    new THREE.MeshStandardMaterial({ color: 0x2d4e2d })
);
groundMesh.rotation.x = -Math.PI / 2;
groundMesh.receiveShadow = true;
scene.add(groundMesh);
scene.add(new THREE.GridHelper(50, 50, 0x2d6e3d, 0x1d4e2d));

const groundBody = new CANNON.Body({ mass: 0, shape: new CANNON.Plane() });
groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
world.addBody(groundBody);

// Crane arm (static kinematic bar)
const armMesh = new THREE.Mesh(
    new THREE.BoxGeometry(0.4, 0.4, 20),
    new THREE.MeshStandardMaterial({ color: 0xffcc44, roughness: 0.3 })
);
armMesh.position.set(0, 15, -5);
armMesh.castShadow = true;
scene.add(armMesh);

const armBody = new CANNON.Body({
    mass: 0,
    shape: new CANNON.Box(new CANNON.Vec3(0.2, 0.2, 10)),
    position: new CANNON.Vec3(0, 15, -5)
});
world.addBody(armBody);

// Crane pillar
const pillarMesh = new THREE.Mesh(
    new THREE.BoxGeometry(0.6, 15, 0.6),
    new THREE.MeshStandardMaterial({ color: 0xffcc44 })
);
pillarMesh.position.set(0, 7.5, -14.8);
scene.add(pillarMesh);

// Wrecking ball
const BALL_RADIUS = 1.0;
const ballMesh = new THREE.Mesh(
    new THREE.SphereGeometry(BALL_RADIUS, 32, 32),
    new THREE.MeshStandardMaterial({ color: 0x333333, metalness: 0.9, roughness: 0.1 })
);
ballMesh.castShadow = true;
scene.add(ballMesh);

const ballBody = new CANNON.Body({
    mass: 50,
    shape: new CANNON.Sphere(BALL_RADIUS),
    position: new CANNON.Vec3(0, 5, 5),
    linearDamping: 0.01,
    angularDamping: 0.5
});
world.addBody(ballBody);

// Attach ball to arm via DistanceConstraint
const ropeLength = 10;
const ropeConstraint = new CANNON.DistanceConstraint(armBody, ballBody, ropeLength);
world.addConstraint(ropeConstraint);

// Visual rope (line)
const ropeMat = new THREE.LineBasicMaterial({ color: 0x888888, linewidth: 2 });
const ropePts = new THREE.BufferGeometry().setFromPoints([
    new THREE.Vector3(0, 15, -5),
    new THREE.Vector3(0,  5,  5)
]);
const ropeLine = new THREE.Line(ropePts, ropeMat);
scene.add(ropeLine);

// Tower blocks
const towerObjects = [];

function buildTower() {
    // Remove old
    for (const { mesh, body } of towerObjects) {
        scene.remove(mesh);
        world.removeBody(body);
    }
    towerObjects.length = 0;

    const cols = 3, rows = 8;
    const bw = 1.2, bh = 0.8, bd = 1.2;

    for (let r = 0; r < rows; r++) {
        for (let c = 0; c < cols; c++) {
            const x = (c - (cols - 1) / 2) * (bw + 0.05) + 5;
            const y = bh / 2 + r * (bh + 0.02);
            const z = 0;

            const mesh = new THREE.Mesh(
                new THREE.BoxGeometry(bw, bh, bd),
                new THREE.MeshStandardMaterial({
                    color: new THREE.Color().setHSL(r / rows, 0.7, 0.5)
                })
            );
            mesh.position.set(x, y, z);
            mesh.castShadow = true;
            scene.add(mesh);

            const body = new CANNON.Body({
                mass: 2,
                shape: new CANNON.Box(new CANNON.Vec3(bw/2, bh/2, bd/2)),
                position: new CANNON.Vec3(x, y, z)
            });
            world.addBody(body);

            towerObjects.push({ mesh, body });
        }
    }
    document.getElementById('info').textContent = `Blocks: ${towerObjects.length}`;
}

// Swing ball
function swing() {
    ballBody.velocity.set(-15, 5, 0);
    ballBody.angularVelocity.set(0, 0, 0);
}

buildTower();
document.getElementById('swing').onclick = swing;
document.getElementById('build').onclick = buildTower;

// Sync rope line
function updateRope() {
    const pts = ropeLine.geometry.attributes.position;
    pts.setXYZ(0, 0, 15, -5);
    pts.setXYZ(1, ballBody.position.x, ballBody.position.y, ballBody.position.z);
    pts.needsUpdate = true;
}

const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    const delta = Math.min(clock.getDelta(), 0.05);

    world.step(1 / 60, delta, 3);

    // Sync ball
    ballMesh.position.copy(ballBody.position);
    ballMesh.quaternion.copy(ballBody.quaternion);
    updateRope();

    // Sync blocks
    for (const { mesh, body } of towerObjects) {
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

## RaycastVehicle (Car Physics)

```javascript
// RaycastVehicle ใน cannon-es
function buildCar(world, scene) {
    const chassisMesh = new THREE.Mesh(
        new THREE.BoxGeometry(2, 0.5, 4),
        new THREE.MeshStandardMaterial({ color: 0xff4444 })
    );
    scene.add(chassisMesh);

    // Chassis body
    const chassisBody = new CANNON.Body({
        mass: 150,
        shape: new CANNON.Box(new CANNON.Vec3(1, 0.25, 2)),
        position: new CANNON.Vec3(0, 2, 0),
        angularDamping: 0.4
    });
    world.addBody(chassisBody);

    // RaycastVehicle
    const vehicle = new CANNON.RaycastVehicle({
        chassisBody,
        indexRightAxis: 0,
        indexUpAxis: 1,
        indexForwardAxis: 2
    });

    // Wheel options
    const wheelOptions = {
        radius: 0.4,
        directionLocal: new CANNON.Vec3(0, -1, 0),   // suspension direction
        suspensionStiffness: 30,
        suspensionRestLength: 0.3,
        frictionSlip: 1.4,
        dampingRelaxation: 2.3,
        dampingCompression: 4.4,
        maxSuspensionForce: 100000,
        rollInfluence: 0.01,
        axleLocal: new CANNON.Vec3(-1, 0, 0),
        chassisConnectionPointLocal: new CANNON.Vec3()
    };

    // 4 wheels: FR, FL, RR, RL
    const wheelPositions = [
        new CANNON.Vec3( 1.0, 0, -1.5),   // FR
        new CANNON.Vec3(-1.0, 0, -1.5),   // FL
        new CANNON.Vec3( 1.0, 0,  1.5),   // RR
        new CANNON.Vec3(-1.0, 0,  1.5)    // RL
    ];

    const wheelMeshes = [];
    for (const pos of wheelPositions) {
        wheelOptions.chassisConnectionPointLocal.copy(pos);
        vehicle.addWheel({ ...wheelOptions });

        const wheelMesh = new THREE.Mesh(
            new THREE.CylinderGeometry(0.4, 0.4, 0.3, 24),
            new THREE.MeshStandardMaterial({ color: 0x222222 })
        );
        wheelMesh.rotation.z = Math.PI / 2;
        scene.add(wheelMesh);
        wheelMeshes.push(wheelMesh);
    }

    vehicle.addToWorld(world);

    // Input state
    const keys = { w: false, s: false, a: false, d: false, space: false };
    const MAX_STEER = 0.5;
    const MAX_FORCE = 1500;
    const BRAKE_FORCE = 30;

    document.addEventListener('keydown', e => {
        if (e.key === 'w' || e.key === 'ArrowUp')    keys.w = true;
        if (e.key === 's' || e.key === 'ArrowDown')  keys.s = true;
        if (e.key === 'a' || e.key === 'ArrowLeft')  keys.a = true;
        if (e.key === 'd' || e.key === 'ArrowRight') keys.d = true;
        if (e.key === ' ') keys.space = true;
    });
    document.addEventListener('keyup', e => {
        if (e.key === 'w' || e.key === 'ArrowUp')    keys.w = false;
        if (e.key === 's' || e.key === 'ArrowDown')  keys.s = false;
        if (e.key === 'a' || e.key === 'ArrowLeft')  keys.a = false;
        if (e.key === 'd' || e.key === 'ArrowRight') keys.d = false;
        if (e.key === ' ') keys.space = false;
    });

    return {
        chassisBody, chassisMesh, vehicle, wheelMeshes,
        
        update() {
            // Input processing
            const engineForce = keys.w ? -MAX_FORCE : keys.s ? MAX_FORCE : 0;
            const steerValue   = keys.a ? MAX_STEER  : keys.d ? -MAX_STEER : 0;
            const brakeForce   = keys.space ? BRAKE_FORCE : 0;

            // Apply to wheels
            vehicle.applyEngineForce(engineForce, 2); // RR
            vehicle.applyEngineForce(engineForce, 3); // RL

            vehicle.setSteeringValue(steerValue, 0);  // FR
            vehicle.setSteeringValue(steerValue, 1);  // FL

            vehicle.setBrake(brakeForce, 0); vehicle.setBrake(brakeForce, 1);
            vehicle.setBrake(brakeForce, 2); vehicle.setBrake(brakeForce, 3);

            // Sync chassis
            chassisMesh.position.copy(chassisBody.position);
            chassisMesh.quaternion.copy(chassisBody.quaternion);

            // Sync wheels
            for (let i = 0; i < vehicle.wheelInfos.length; i++) {
                vehicle.updateWheelTransform(i);
                const t = vehicle.wheelInfos[i].worldTransform;
                wheelMeshes[i].position.copy(t.position);
                wheelMeshes[i].quaternion.copy(t.quaternion);
            }
        }
    };
}
```

---

## Physics Debug Renderer

```javascript
// CannonDebugger สำหรับ visualize physics shapes
// npm install cannon-es-debugger

import CannonDebugger from 'cannon-es-debugger';

const cannonDebugger = new CannonDebugger(scene, world, {
    color: 0x00ff00,   // debug wireframe color
    scale: 1.0
});

// ใน animation loop
function animate() {
    requestAnimationFrame(animate);
    world.step(1/60);
    cannonDebugger.update();  // อัพเดท debug visuals
    renderer.render(scene, camera);
}
```

---

## Convex Decomposition

```javascript
// สำหรับ concave mesh → ต้องแบ่งเป็น convex pieces
// ใช้ VHACD หรือ built-in ConvexPolyhedron สำหรับส่วนๆ

import { computeConvexPolyhedron } from 'cannon-es';

function concaveMeshToPhysics(threeGeometry, body) {
    // Extract vertices
    const posAttr = threeGeometry.attributes.position;
    const verts = [];
    for (let i = 0; i < posAttr.count; i++) {
        verts.push(new CANNON.Vec3(
            posAttr.getX(i),
            posAttr.getY(i),
            posAttr.getZ(i)
        ));
    }

    // Extract faces
    const faces = [];
    if (threeGeometry.index) {
        const idx = threeGeometry.index;
        for (let i = 0; i < idx.count; i += 3) {
            faces.push([idx.getX(i), idx.getX(i+1), idx.getX(i+2)]);
        }
    }

    const shape = new CANNON.ConvexPolyhedron({ vertices: verts, faces });
    body.addShape(shape);
}
```

---

## แบบฝึกหัด

### ระดับ 1
1. สร้าง cloth ที่มีสีต่างๆ และทดลองกับ wind
2. สร้าง rope ที่มี weight ห้อยอยู่ และแกว่งได้
3. สร้าง car ง่ายๆ ที่ขับได้ด้วย WASD

### ระดับ 2
4. เพิ่ม cloth ให้มีหลาย pin points (ไม่ใช่แค่มุม)
5. สร้าง rope bridge ที่เดินข้ามได้
6. Car with camera follow (third-person)

### ระดับ 3
7. Cloth ที่ถูก ball ชนแล้วเด้งออก
8. Crane game: เครนที่ยกของได้ด้วย DistanceConstraint
9. Racing game: track + lap counting + timer

### Challenge
10. Physics Playground ครบ:
    - Cloth, rope, car ในฉากเดียว
    - Interactive: คลิกเพื่อยิง cannon ball
    - Car ชนได้ทั้ง cloth และ rope
    - Score/timer system
    - Soft-reset vs Hard-reset
