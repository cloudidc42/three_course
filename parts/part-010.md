# Part 010: Cameras Deep Dive

## สารบัญบทเรียนนี้
- [1. Camera พื้นฐาน](#1-camera-พื้นฐาน)
- [2. PerspectiveCamera Math](#2-perspectivecamera-math)
- [3. OrthographicCamera](#3-orthographiccamera)
- [4. Camera Movement และ Interpolation](#4-camera-movement-และ-interpolation)
- [5. Multiple Cameras](#5-multiple-cameras)
- [6. Camera Helpers](#6-camera-helpers)
- [7. CubeCamera](#7-cubecamera)
- [8. ArrayCamera](#8-arraycamera)
- [9. Controls](#9-controls)
- [10. Complete Camera Demo](#10-complete-camera-demo)
- [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. Camera พื้นฐาน

Camera ใน Three.js กำหนดว่าเราเห็น scene อย่างไร

```javascript
// Three.js มี Camera หลักสองชนิด:
//
// PerspectiveCamera  → เห็น perspective (สิ่งที่ไกลดูเล็กกว่า)
//                      เหมือนตามนุษย์ และกล้องถ่ายรูป
//
// OrthographicCamera → ไม่มี perspective
//                      ขนาดเท่ากันไม่ว่าจะใกล้ไกล
//                      เหมือน blueprint / 2D game

// ทั้งสองสืบทอดจาก THREE.Camera → THREE.Object3D
// ดังนั้น camera มี position, rotation, scale ปกติ

// กำหนด direction ที่ camera มองด้วย lookAt
camera.lookAt(new THREE.Vector3(0, 0, 0));
// หรือ
camera.lookAt(0, 0, 0);

// camera ที่ "ลืมตา" = ที่ add ใน renderer.render(scene, camera)
```

---

## 2. PerspectiveCamera Math

```javascript
// ========== Constructor ==========

const camera = new THREE.PerspectiveCamera(
    75,             // fov (field of view) ในหน่วย degrees
    width / height, // aspect ratio
    0.1,            // near clipping plane (meters)
    100             // far clipping plane (meters)
);

// ========== FOV (Field of View) ==========

// fov = มุมการมองเห็นในแนวตั้ง (vertical FOV)
// ถ้า aspect ratio เปลี่ยน, horizontal FOV เปลี่ยนตาม
// vertical FOV ไม่เปลี่ยน

// ค่าทั่วไป:
// 45° = telephoto (zoom in, perspective compressed)
// 60° = normal
// 75° = wide (Three.js default)
// 90° = very wide
// 120°+ = fisheye / extreme wide

// ผลของ FOV:
// fov น้อย = zoom in, objects ดูใหญ่, perspective น้อย
// fov มาก = zoom out, objects ดูเล็ก, perspective มาก/บิดเบือน

// ========== Aspect Ratio ==========

// ต้อง update เมื่อ canvas เปลี่ยนขนาด
window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix(); // ⚠️ ต้องเรียกหลัง update properties ใดๆ
    renderer.setSize(window.innerWidth, window.innerHeight);
});

// ========== Near / Far Planes ==========

// Objects นอก [near, far] จะไม่ render
// near ควรมากกว่า 0 (0 = z-fighting)
// far ควรน้อยที่สุดเท่าที่ครอบคลุม scene

// Z-fighting: เกิดเมื่อ near/far ratio ใหญ่เกินไป
// หรือเมื่อสอง surfaces อยู่ใกล้กันมาก

// ❌ Bad ratio (far/near = 1,000,000 → z-precision ต่ำมาก)
const badCamera = new THREE.PerspectiveCamera(75, 1, 0.0001, 100000);

// ✅ Good ratio (far/near ≤ 10,000)
const goodCamera = new THREE.PerspectiveCamera(75, 1, 0.1, 1000);

// ========== Focal Length (alternative to fov) ==========

// Focal length ใน mm เหมือนกล้องถ่ายรูป
// 35mm full frame: standard ~50mm
function setFocalLength(camera, focalLength, filmHeight = 24) {
    // fov = 2 * arctan(filmHeight / (2 * focalLength))
    camera.fov = 2 * Math.atan(filmHeight / (2 * focalLength)) * (180 / Math.PI);
    camera.updateProjectionMatrix();
}

// Preset focal lengths
const focalLengths = {
    'fisheye': 8,
    'ultrawide': 14,
    'wide': 24,
    'normal': 50,
    'portrait': 85,
    'telephoto': 200,
};

setFocalLength(camera, focalLengths.portrait); // 85mm portrait

// ========== Depth of Field Calculation ==========

// Circle of Confusion (CoC)
// Bokeh: ความเบลอของ background/foreground
// Three.js ไม่มี built-in DoF
// ต้องใช้ Post Processing (Part 15)

// ========== Camera Projection Matrix ==========

// updateProjectionMatrix() สร้าง matrix จาก fov, aspect, near, far
// ต้องเรียกหลังเปลี่ยน properties เหล่านี้
camera.fov = 45;
camera.aspect = 1.5;
camera.near = 0.5;
camera.far = 50;
camera.updateProjectionMatrix(); // ← ต้องเรียกทุกครั้ง!
```

---

## 3. OrthographicCamera

```javascript
// ========== Constructor ==========

const aspect = window.innerWidth / window.innerHeight;
const frustumSize = 10; // world units ที่เห็นในแนวตั้ง

const orthoCamera = new THREE.OrthographicCamera(
    -frustumSize * aspect / 2,  // left
     frustumSize * aspect / 2,  // right
     frustumSize / 2,            // top
    -frustumSize / 2,            // bottom
    0.1,                         // near
    100                          // far
);
orthoCamera.position.set(0, 0, 10);
orthoCamera.lookAt(0, 0, 0);

// ========== ปรับขนาดตาม Window ==========

window.addEventListener('resize', () => {
    const newAspect = window.innerWidth / window.innerHeight;
    orthoCamera.left   = -frustumSize * newAspect / 2;
    orthoCamera.right  =  frustumSize * newAspect / 2;
    orthoCamera.top    =  frustumSize / 2;
    orthoCamera.bottom = -frustumSize / 2;
    orthoCamera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});

// ========== Zoom สำหรับ OrthographicCamera ==========

// Orthographic zoom = ลด frustum size
// ไม่ใช่ move camera เข้าหา object

orthoCamera.zoom = 2.0;  // zoom 2x (เห็นพื้นที่น้อยลง)
orthoCamera.updateProjectionMatrix();

// หรือ scale frustum
function setOrthoZoom(camera, zoom, baseSize, aspect) {
    const size = baseSize / zoom;
    camera.left = -size * aspect / 2;
    camera.right = size * aspect / 2;
    camera.top = size / 2;
    camera.bottom = -size / 2;
    camera.updateProjectionMatrix();
}

// ========== Use Cases ==========

// 1. 2D Games (top-down or side view)
const game2dCamera = new THREE.OrthographicCamera(-20, 20, 15, -15, -100, 100);
game2dCamera.position.set(0, 0, 10);
game2dCamera.lookAt(0, 0, 0);

// 2. UI / HUD layer
const hudCamera = new THREE.OrthographicCamera(
    0, window.innerWidth,
    window.innerHeight, 0,
    -10, 10
);

// 3. Isometric view
const isoCamera = new THREE.OrthographicCamera(-15, 15, 10, -10, -50, 50);
isoCamera.position.set(10, 10, 10);
isoCamera.lookAt(0, 0, 0);

// 4. Technical drawing / CAD
const cadCamera = new THREE.OrthographicCamera(-5, 5, 5, -5, 0.1, 50);
cadCamera.position.set(0, 0, 20); // top-down

// ========== Perspective vs Orthographic ==========

// Perspective:  สมจริง, มี depth perception, สำหรับ 3D games/apps
// Orthographic: ไม่บิดเบือน, เหมาะสำหรับ 2D, UI, CAD, isometric
```

---

## 4. Camera Movement และ Interpolation

```javascript
// ========== Basic Movement ==========

// ตรงๆ (immediate)
camera.position.set(5, 3, 8);
camera.lookAt(0, 0, 0);

// ========== LERP (Linear Interpolation) ==========

const targetPosition = new THREE.Vector3(5, 3, 8);
const targetLookAt = new THREE.Vector3(0, 0, 0);
const currentLookAt = new THREE.Vector3();

function animateCameraLERP(delta) {
    // Lerp position
    camera.position.lerp(targetPosition, 1 - Math.pow(0.001, delta));
    // (1 - 0.001^delta) = smooth factor ที่ frame-rate independent
    
    // Lerp lookAt
    currentLookAt.lerp(targetLookAt, 1 - Math.pow(0.001, delta));
    camera.lookAt(currentLookAt);
}

// ========== Smooth Follow Camera ==========

class FollowCamera {
    constructor(camera, target, offset = new THREE.Vector3(0, 3, 6)) {
        this.camera = camera;
        this.target = target;
        this.offset = offset;
        this.currentPos = new THREE.Vector3();
        this.currentLookAt = new THREE.Vector3();
        this.lerpFactor = 5.0;
    }
    
    update(delta) {
        // คำนวณ desired position
        const desiredPos = new THREE.Vector3();
        this.target.getWorldPosition(desiredPos);
        desiredPos.add(this.offset);
        
        // Lerp camera position
        this.currentPos.lerp(desiredPos, this.lerpFactor * delta);
        this.camera.position.copy(this.currentPos);
        
        // Lerp lookAt
        const targetWorldPos = new THREE.Vector3();
        this.target.getWorldPosition(targetWorldPos);
        this.currentLookAt.lerp(targetWorldPos, this.lerpFactor * delta);
        this.camera.lookAt(this.currentLookAt);
    }
}

// ใช้งาน
const followCam = new FollowCamera(camera, playerMesh, new THREE.Vector3(0, 4, 8));
function animate(delta) {
    followCam.update(delta);
}

// ========== Camera Path (Spline) ==========

import { CatmullRomCurve3 } from 'three';

const cameraPath = new THREE.CatmullRomCurve3([
    new THREE.Vector3(-10, 5, 10),
    new THREE.Vector3(0, 8, 15),
    new THREE.Vector3(10, 5, 10),
    new THREE.Vector3(10, 2, -5),
    new THREE.Vector3(0, 4, -10),
    new THREE.Vector3(-10, 5, 10),
], true); // closed loop

const lookAtPath = new THREE.CatmullRomCurve3([
    new THREE.Vector3(0, 0, 0),
    new THREE.Vector3(2, 0, -2),
    new THREE.Vector3(0, 0, 0),
    new THREE.Vector3(-2, 0, 2),
    new THREE.Vector3(0, 0, 0),
    new THREE.Vector3(0, 0, 0),
], true);

let pathTime = 0;

function updateCinematicCamera(delta) {
    pathTime += delta * 0.05; // speed
    if (pathTime > 1) pathTime -= 1;
    
    const pos = cameraPath.getPoint(pathTime);
    const lookAt = lookAtPath.getPoint(pathTime);
    
    camera.position.copy(pos);
    camera.lookAt(lookAt);
}

// ========== Camera Shake ==========

class CameraShake {
    constructor(camera) {
        this.camera = camera;
        this.intensity = 0;
        this.decay = 0.9;
        this.originPos = camera.position.clone();
    }
    
    shake(intensity) {
        this.intensity = intensity;
    }
    
    update(delta) {
        if (this.intensity < 0.001) return;
        
        this.camera.position.x = this.originPos.x + (Math.random() - 0.5) * this.intensity;
        this.camera.position.y = this.originPos.y + (Math.random() - 0.5) * this.intensity;
        this.camera.rotation.z = (Math.random() - 0.5) * this.intensity * 0.1;
        
        this.intensity *= Math.pow(this.decay, delta * 60);
    }
}

// ========== Orbit Camera (manual) ==========

class ManualOrbitCamera {
    constructor(camera, target = new THREE.Vector3(0, 0, 0)) {
        this.camera = camera;
        this.target = target;
        this.spherical = new THREE.Spherical();
        this.spherical.setFromCartesianCoords(
            camera.position.x - target.x,
            camera.position.y - target.y,
            camera.position.z - target.z
        );
    }
    
    rotate(deltaTheta, deltaPhi) {
        this.spherical.theta -= deltaTheta;
        this.spherical.phi = Math.max(0.1, Math.min(Math.PI - 0.1,
            this.spherical.phi - deltaPhi
        ));
        this.updateCamera();
    }
    
    zoom(factor) {
        this.spherical.radius = Math.max(1, 
            Math.min(100, this.spherical.radius * factor)
        );
        this.updateCamera();
    }
    
    updateCamera() {
        const pos = new THREE.Vector3().setFromSpherical(this.spherical);
        this.camera.position.copy(pos).add(this.target);
        this.camera.lookAt(this.target);
    }
}
```

---

## 5. Multiple Cameras

```javascript
// ========== สลับ Camera ==========

let activeCamera = perspCamera;

document.addEventListener('keydown', (e) => {
    if (e.key === '1') activeCamera = perspCamera;
    if (e.key === '2') activeCamera = orthoCamera;
    if (e.key === '3') activeCamera = topCamera;
});

function animate() {
    requestAnimationFrame(animate);
    renderer.render(scene, activeCamera); // ใช้ activeCamera
}

// ========== Split Screen (Viewport Rendering) ==========

const perspCam = new THREE.PerspectiveCamera(60, 0.5 * innerWidth / innerHeight, 0.1, 100);
perspCam.position.set(5, 3, 5);
perspCam.lookAt(0, 0, 0);

const orthoCam = new THREE.OrthographicCamera(-10, 10, 8, -8, 0.1, 100);
orthoCam.position.set(0, 15, 0);
orthoCam.lookAt(0, 0, 0);

function renderSplitScreen() {
    renderer.setScissorTest(true);
    
    // Left: Perspective
    const leftWidth = Math.floor(innerWidth / 2);
    renderer.setViewport(0, 0, leftWidth, innerHeight);
    renderer.setScissor(0, 0, leftWidth, innerHeight);
    perspCam.aspect = leftWidth / innerHeight;
    perspCam.updateProjectionMatrix();
    renderer.render(scene, perspCam);
    
    // Right: Orthographic (top-down)
    const rightWidth = innerWidth - leftWidth;
    renderer.setViewport(leftWidth, 0, rightWidth, innerHeight);
    renderer.setScissor(leftWidth, 0, rightWidth, innerHeight);
    const orthoAspect = rightWidth / innerHeight;
    orthoCam.left = -10 * orthoAspect;
    orthoCam.right = 10 * orthoAspect;
    orthoCam.updateProjectionMatrix();
    renderer.render(scene, orthoCam);
    
    renderer.setScissorTest(false);
}

// ========== Picture-in-Picture (PiP) ==========

function renderPiP(mainCam, pipCam) {
    // Main view (full screen)
    renderer.setScissorTest(false);
    renderer.render(scene, mainCam);
    
    // PiP (bottom right corner)
    const pipW = Math.floor(innerWidth * 0.25);
    const pipH = Math.floor(innerHeight * 0.25);
    const pipX = innerWidth - pipW - 10;
    const pipY = 10;
    
    renderer.setScissorTest(true);
    renderer.setViewport(pipX, pipY, pipW, pipH);
    renderer.setScissor(pipX, pipY, pipW, pipH);
    
    pipCam.aspect = pipW / pipH;
    pipCam.updateProjectionMatrix();
    
    renderer.render(scene, pipCam);
    renderer.setScissorTest(false);
}
```

---

## 6. Camera Helpers

```javascript
// ========== CameraHelper ==========

// แสดง frustum ของ camera
const frustumHelper = new THREE.CameraHelper(perspCamera);
scene.add(frustumHelper);

// update เมื่อ camera เคลื่อนที่
frustumHelper.update();

// ⚠️ CameraHelper จะมองเห็นจาก camera อื่นเท่านั้น
// ถ้าใช้ camera เดียวกันที่ helper แสดง จะเห็นแค่เส้นแปลกๆ

// วิธีใช้: ใช้ 2 cameras
// mainCamera = camera ที่เราควบคุม
// debugCamera = camera อีกอันสำหรับ overview
scene.add(new THREE.CameraHelper(mainCamera));
renderer.render(scene, debugCamera); // เห็น frustum ของ mainCamera

// ========== Axes Helper บน Camera ==========

// เพิ่ม axes เพื่อเห็นทิศทาง
const cameraAxes = new THREE.AxesHelper(0.5);
camera.add(cameraAxes);
scene.add(camera); // camera ต้อง add ใน scene เพื่อ axes จะ inherit transform
```

---

## 7. CubeCamera

```javascript
// CubeCamera render scene ลงใน CubeRenderTarget (6 ทิศทาง)
// ใช้สำหรับ real-time reflection และ environment capture

import * as THREE from 'three';

const cubeRenderTarget = new THREE.WebGLCubeRenderTarget(512, {
    type: THREE.HalfFloatType,   // HDR values
    generateMipmaps: true,
    minFilter: THREE.LinearMipmapLinearFilter,
});

const cubeCamera = new THREE.CubeCamera(0.1, 100, cubeRenderTarget);
scene.add(cubeCamera);

// Material ที่ใช้ cube camera texture
const reflectiveMat = new THREE.MeshStandardMaterial({
    metalness: 1.0,
    roughness: 0.0,
    envMap: cubeRenderTarget.texture,
});

const mirrorBall = new THREE.Mesh(
    new THREE.SphereGeometry(1, 64, 32),
    reflectiveMat
);
scene.add(mirrorBall);
// cubeCamera ควรอยู่ที่ตำแหน่งเดียวกับ mirrorBall
cubeCamera.position.copy(mirrorBall.position);

// Update ใน animate loop
function animate() {
    requestAnimationFrame(animate);
    
    // ซ่อน mirror ball ระหว่าง capture (ไม่งั้นจะเห็นตัวเอง)
    mirrorBall.visible = false;
    cubeCamera.update(renderer, scene);  // render 6 faces
    mirrorBall.visible = true;
    
    renderer.render(scene, camera);
}

// ⚠️ Performance: cubeCamera.update() = render 6x
// ใช้เฉพาะเมื่อจำเป็น หรือทุกๆ N frames
let cubeUpdateInterval = 0;
function animateOptimized() {
    requestAnimationFrame(animateOptimized);
    cubeUpdateInterval++;
    
    if (cubeUpdateInterval % 3 === 0) { // update ทุก 3 frames
        mirrorBall.visible = false;
        cubeCamera.update(renderer, scene);
        mirrorBall.visible = true;
    }
    
    renderer.render(scene, camera);
}
```

---

## 8. ArrayCamera

```javascript
// ArrayCamera ใช้สำหรับ VR/multi-view rendering
// Render หลาย viewports ใน single pass

const cameras = [];
const numViews = 4;

for (let i = 0; i < numViews; i++) {
    const subCamera = new THREE.PerspectiveCamera(45, 0.5, 0.1, 100);
    subCamera.viewport = new THREE.Vector4(
        Math.floor(i % 2) * window.innerWidth / 2,   // x
        Math.floor(i / 2) * window.innerHeight / 2,  // y
        window.innerWidth / 2,                        // width
        window.innerHeight / 2                        // height
    );
    subCamera.position.set(
        (i % 2 - 0.5) * 8,      // left/right
        (Math.floor(i / 2) - 0.5) * 4,  // up/down
        6
    );
    subCamera.lookAt(0, 0, 0);
    cameras.push(subCamera);
}

const arrayCamera = new THREE.ArrayCamera(cameras);

function animate() {
    requestAnimationFrame(animate);
    renderer.render(scene, arrayCamera);
}
```

---

## 9. Controls

Three.js มี built-in controls หลายชนิด

```javascript
// ========== OrbitControls (ที่ใช้บ่อยที่สุด) ==========

import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

const controls = new OrbitControls(camera, renderer.domElement);

// Properties
controls.enableDamping = true;     // smooth inertia
controls.dampingFactor = 0.05;     // ค่า default = 0.05
controls.screenSpacePanning = true; // pan mode (false = pan along XZ)

controls.minDistance = 2;           // zoom in limit
controls.maxDistance = 50;          // zoom out limit
controls.minPolarAngle = 0;         // bottom rotation limit (radians)
controls.maxPolarAngle = Math.PI;   // top rotation limit
controls.minAzimuthAngle = -Infinity; // left limit
controls.maxAzimuthAngle = Infinity;  // right limit

controls.enableZoom = true;
controls.enablePan = true;
controls.enableRotate = true;

controls.autoRotate = true;         // auto rotate
controls.autoRotateSpeed = 2.0;     // degrees/second

controls.target.set(0, 0, 0);       // กำหนด center
controls.update();                  // ต้องเรียกหลัง set target

// ⚠️ ต้องเรียก controls.update() ใน animate loop เมื่อ enableDamping = true
function animate() {
    requestAnimationFrame(animate);
    controls.update(); // สำคัญมาก!
    renderer.render(scene, camera);
}

// ========== FlyControls ==========

import { FlyControls } from 'three/addons/controls/FlyControls.js';

const flyControls = new FlyControls(camera, renderer.domElement);
flyControls.movementSpeed = 10;
flyControls.rollSpeed = Math.PI / 24;
flyControls.autoForward = false;
flyControls.dragToLook = true;
// ใช้ WASD + mouse

function animateFly(delta) {
    flyControls.update(delta); // ต้องการ delta
}

// ========== FirstPersonControls ==========

import { FirstPersonControls } from 'three/addons/controls/FirstPersonControls.js';

const fpControls = new FirstPersonControls(camera, renderer.domElement);
fpControls.movementSpeed = 5;
fpControls.lookSpeed = 0.05;
fpControls.activeLook = true;

// ========== PointerLockControls (FPS) ==========

import { PointerLockControls } from 'three/addons/controls/PointerLockControls.js';

const pointerControls = new PointerLockControls(camera, document.body);

// Lock pointer (เหมือน FPS game)
document.addEventListener('click', () => {
    pointerControls.lock();
});

pointerControls.addEventListener('lock', () => {
    console.log('Pointer locked');
});
pointerControls.addEventListener('unlock', () => {
    console.log('Pointer unlocked');
});

// Move ด้วย WASD
const velocity = new THREE.Vector3();
const direction = new THREE.Vector3();
const keys = {};

document.addEventListener('keydown', e => keys[e.code] = true);
document.addEventListener('keyup', e => keys[e.code] = false);

function updateFPS(delta) {
    const speed = 5;
    velocity.x -= velocity.x * 10 * delta; // friction
    velocity.z -= velocity.z * 10 * delta;
    
    direction.z = (keys['KeyW'] ? -1 : 0) + (keys['KeyS'] ? 1 : 0);
    direction.x = (keys['KeyA'] ? -1 : 0) + (keys['KeyD'] ? 1 : 0);
    direction.normalize();
    
    if (keys['KeyW'] || keys['KeyS']) velocity.z -= direction.z * speed * delta;
    if (keys['KeyA'] || keys['KeyD']) velocity.x -= direction.x * speed * delta;
    
    pointerControls.moveRight(-velocity.x * delta);
    pointerControls.moveForward(-velocity.z * delta);
}
```

---

## 10. Complete Camera Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Part 010: Cameras Deep Dive</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        html, body { width: 100%; height: 100%; overflow: hidden; background: #0d1117; }
        #ui {
            position: fixed; top: 10px; left: 50%; transform: translateX(-50%);
            display: flex; gap: 6px;
        }
        .cam-btn {
            background: rgba(30,30,50,0.9);
            color: #aaa; border: 1px solid #444;
            padding: 6px 14px; border-radius: 6px;
            cursor: pointer; font-family: monospace; font-size: 12px;
            transition: all 0.2s;
        }
        .cam-btn:hover { border-color: #6c63ff; color: white; }
        .cam-btn.active { background: #6c63ff; color: white; border-color: #6c63ff; }
        #info {
            position: fixed; bottom: 10px; left: 10px;
            color: #888; font-family: monospace; font-size: 11px;
        }
        #divider {
            position: fixed; top: 0; bottom: 0;
            left: 50%; width: 1px; background: rgba(255,255,255,0.2);
            display: none;
        }
        #label-left, #label-right {
            position: fixed; top: 40px;
            color: rgba(255,255,255,0.5); font-family: monospace; font-size: 11px;
            display: none;
        }
        #label-left { left: 10px; }
        #label-right { right: 10px; text-align: right; }
    </style>
</head>
<body>
    <div id="ui">
        <button class="cam-btn active" data-mode="orbit">Orbit</button>
        <button class="cam-btn" data-mode="ortho">Orthographic</button>
        <button class="cam-btn" data-mode="split">Split View</button>
        <button class="cam-btn" data-mode="top">Top View</button>
        <button class="cam-btn" data-mode="front">Front View</button>
        <button class="cam-btn" data-mode="cinematic">Cinematic</button>
    </div>
    <div id="divider"></div>
    <div id="label-left">Perspective</div>
    <div id="label-right">Orthographic</div>
    <div id="info">Drag to orbit · Scroll to zoom</div>
    
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
        import { GUI } from 'three/addons/libs/lil-gui.module.min.js';
        
        // ====== RENDERER ======
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(innerWidth, innerHeight);
        renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
        renderer.shadowMap.enabled = true;
        renderer.toneMapping = THREE.ACESFilmicToneMapping;
        renderer.toneMappingExposure = 1.0;
        document.body.appendChild(renderer.domElement);
        
        // ====== SCENE ======
        const scene = new THREE.Scene();
        scene.background = new THREE.Color('#0d1117');
        scene.fog = new THREE.Fog('#0d1117', 30, 80);
        
        // Lights
        const ambLight = new THREE.AmbientLight(0xffffff, 0.3);
        const dirLight = new THREE.DirectionalLight(0xffffff, 3);
        dirLight.position.set(10, 20, 10);
        dirLight.castShadow = true;
        dirLight.shadow.mapSize.set(2048, 2048);
        dirLight.shadow.camera.near = 0.5;
        dirLight.shadow.camera.far = 60;
        const s = 20;
        dirLight.shadow.camera.left = -s; dirLight.shadow.camera.right = s;
        dirLight.shadow.camera.top = s; dirLight.shadow.camera.bottom = -s;
        scene.add(ambLight, dirLight);
        
        // ====== OBJECTS ======
        
        // Ground
        const ground = new THREE.Mesh(
            new THREE.PlaneGeometry(30, 30, 20, 20),
            new THREE.MeshStandardMaterial({ color: '#1a1a2e', roughness: 0.8 })
        );
        ground.rotation.x = -Math.PI / 2;
        ground.receiveShadow = true;
        scene.add(ground);
        
        // Grid
        const grid = new THREE.GridHelper(30, 30, '#333366', '#1a1a33');
        scene.add(grid);
        
        // Objects
        const shapes = [
            { geo: new THREE.BoxGeometry(1.2, 1.2, 1.2), pos: [-4, 0.6, -4], color: '#e74c3c' },
            { geo: new THREE.SphereGeometry(0.8, 32, 16), pos: [0, 0.8, -4], color: '#3498db' },
            { geo: new THREE.ConeGeometry(0.8, 1.6, 16), pos: [4, 0.8, -4], color: '#2ecc71' },
            { geo: new THREE.TorusGeometry(0.7, 0.25, 16, 32), pos: [-4, 1, 0], color: '#f39c12' },
            { geo: new THREE.CylinderGeometry(0.5, 0.5, 1.5, 16), pos: [0, 0.75, 0], color: '#9b59b6' },
            { geo: new THREE.IcosahedronGeometry(0.8, 1), pos: [4, 0.8, 0], color: '#1abc9c' },
            { geo: new THREE.TorusKnotGeometry(0.5, 0.18, 100, 16), pos: [-4, 1, 4], color: '#e67e22' },
            { geo: new THREE.OctahedronGeometry(0.8), pos: [0, 0.8, 4], color: '#ff6b9d' },
            { geo: new THREE.TetrahedronGeometry(0.9), pos: [4, 0.8, 4], color: '#a8e6cf' },
        ];
        
        const meshes = shapes.map(s => {
            const mesh = new THREE.Mesh(
                s.geo,
                new THREE.MeshStandardMaterial({ color: s.color, roughness: 0.3, metalness: 0.2 })
            );
            mesh.position.set(...s.pos);
            mesh.castShadow = true;
            mesh.receiveShadow = true;
            scene.add(mesh);
            return mesh;
        });
        
        // ====== CAMERAS ======
        
        // Main perspective camera
        const perspCamera = new THREE.PerspectiveCamera(60, innerWidth/innerHeight, 0.1, 100);
        perspCamera.position.set(8, 6, 8);
        
        // Orthographic camera
        const aspect = innerWidth / innerHeight;
        const orthoSize = 10;
        const orthoCamera = new THREE.OrthographicCamera(
            -orthoSize * aspect, orthoSize * aspect,
            orthoSize, -orthoSize,
            0.1, 100
        );
        orthoCamera.position.set(8, 6, 8);
        orthoCamera.lookAt(0, 0, 0);
        
        // Top-down camera
        const topCamera = new THREE.OrthographicCamera(
            -15 * aspect, 15 * aspect, 15, -15, 0.1, 100
        );
        topCamera.position.set(0, 20, 0);
        topCamera.lookAt(0, 0, 0);
        
        // Front camera
        const frontCamera = new THREE.OrthographicCamera(
            -12 * aspect, 12 * aspect, 12, -12, 0.1, 100
        );
        frontCamera.position.set(0, 4, 20);
        frontCamera.lookAt(0, 4, 0);
        
        // ====== ORBIT CONTROLS ======
        
        const orbitControls = new OrbitControls(perspCamera, renderer.domElement);
        orbitControls.enableDamping = true;
        orbitControls.target.set(0, 1, 0);
        orbitControls.update();
        
        // ====== CINEMATIC PATH ======
        
        const cinematicPath = new THREE.CatmullRomCurve3([
            new THREE.Vector3(-12, 3, 12),
            new THREE.Vector3(-8, 8, -8),
            new THREE.Vector3(0, 12, -15),
            new THREE.Vector3(12, 6, -10),
            new THREE.Vector3(15, 3, 0),
            new THREE.Vector3(12, 5, 12),
            new THREE.Vector3(0, 8, 14),
            new THREE.Vector3(-12, 3, 12),
        ], false);
        
        // Visualize path
        const pathPoints = cinematicPath.getPoints(100);
        const pathGeo = new THREE.BufferGeometry().setFromPoints(pathPoints);
        const pathLine = new THREE.Line(pathGeo, new THREE.LineBasicMaterial({ color: '#6c63ff', opacity: 0.3, transparent: true }));
        scene.add(pathLine);
        
        let cinematicT = 0;
        let cinematicCamera = new THREE.PerspectiveCamera(50, innerWidth/innerHeight, 0.1, 100);
        
        // ====== MODE MANAGEMENT ======
        
        let currentMode = 'orbit';
        let activeControls = orbitControls;
        
        const divider = document.getElementById('divider');
        const labelLeft = document.getElementById('label-left');
        const labelRight = document.getElementById('label-right');
        
        function setMode(mode) {
            currentMode = mode;
            divider.style.display = 'none';
            labelLeft.style.display = 'none';
            labelRight.style.display = 'none';
            
            if (mode === 'orbit') {
                orbitControls.enabled = true;
            } else {
                orbitControls.enabled = false;
            }
            
            if (mode === 'split') {
                divider.style.display = 'block';
                labelLeft.style.display = 'block';
                labelRight.style.display = 'block';
            }
            
            // Update buttons
            document.querySelectorAll('.cam-btn').forEach(btn => {
                btn.classList.toggle('active', btn.dataset.mode === mode);
            });
        }
        
        document.querySelectorAll('.cam-btn').forEach(btn => {
            btn.addEventListener('click', () => setMode(btn.dataset.mode));
        });
        
        // ====== RENDER FUNCTION ==========
        
        function renderScene() {
            switch (currentMode) {
                case 'orbit':
                    renderer.setScissorTest(false);
                    renderer.setViewport(0, 0, innerWidth, innerHeight);
                    renderer.render(scene, perspCamera);
                    break;
                
                case 'ortho':
                    renderer.setScissorTest(false);
                    renderer.setViewport(0, 0, innerWidth, innerHeight);
                    renderer.render(scene, orthoCamera);
                    break;
                
                case 'split': {
                    const w2 = Math.floor(innerWidth / 2);
                    
                    // Left: Perspective
                    renderer.setScissorTest(true);
                    renderer.setViewport(0, 0, w2, innerHeight);
                    renderer.setScissor(0, 0, w2, innerHeight);
                    perspCamera.aspect = w2 / innerHeight;
                    perspCamera.updateProjectionMatrix();
                    renderer.render(scene, perspCamera);
                    
                    // Right: Orthographic
                    renderer.setViewport(w2, 0, innerWidth - w2, innerHeight);
                    renderer.setScissor(w2, 0, innerWidth - w2, innerHeight);
                    const oAspect = (innerWidth - w2) / innerHeight;
                    orthoCamera.left = -orthoSize * oAspect;
                    orthoCamera.right = orthoSize * oAspect;
                    orthoCamera.updateProjectionMatrix();
                    renderer.render(scene, orthoCamera);
                    
                    renderer.setScissorTest(false);
                    perspCamera.aspect = innerWidth / innerHeight;
                    perspCamera.updateProjectionMatrix();
                    break;
                }
                
                case 'top':
                    renderer.setScissorTest(false);
                    renderer.setViewport(0, 0, innerWidth, innerHeight);
                    const topAspect = innerWidth / innerHeight;
                    topCamera.left = -15 * topAspect;
                    topCamera.right = 15 * topAspect;
                    topCamera.updateProjectionMatrix();
                    renderer.render(scene, topCamera);
                    break;
                
                case 'front':
                    renderer.setScissorTest(false);
                    renderer.setViewport(0, 0, innerWidth, innerHeight);
                    const frontAspect = innerWidth / innerHeight;
                    frontCamera.left = -12 * frontAspect;
                    frontCamera.right = 12 * frontAspect;
                    frontCamera.updateProjectionMatrix();
                    renderer.render(scene, frontCamera);
                    break;
                
                case 'cinematic':
                    renderer.setScissorTest(false);
                    renderer.setViewport(0, 0, innerWidth, innerHeight);
                    renderer.render(scene, cinematicCamera);
                    break;
            }
        }
        
        // ====== ANIMATION ======
        const clock = new THREE.Clock();
        
        function animate() {
            requestAnimationFrame(animate);
            const delta = clock.getDelta();
            const elapsed = clock.getElapsedTime();
            
            // Rotate objects
            meshes.forEach((mesh, i) => {
                mesh.rotation.y += delta * (0.3 + i * 0.04);
            });
            
            // Sync ortho camera with persp
            if (currentMode === 'split' || currentMode === 'ortho') {
                orthoCamera.position.copy(perspCamera.position);
                orthoCamera.quaternion.copy(perspCamera.quaternion);
            }
            
            // Cinematic path
            if (currentMode === 'cinematic') {
                cinematicT += delta * 0.04;
                if (cinematicT > 1) cinematicT -= 1;
                
                const pos = cinematicPath.getPoint(cinematicT);
                const ahead = cinematicPath.getPoint((cinematicT + 0.01) % 1);
                
                cinematicCamera.position.copy(pos);
                cinematicCamera.lookAt(0, 1, 0);
                cinematicCamera.aspect = innerWidth / innerHeight;
                cinematicCamera.updateProjectionMatrix();
            }
            
            if (currentMode === 'orbit') orbitControls.update();
            
            renderScene();
        }
        
        animate();
        
        // ====== RESIZE ======
        window.addEventListener('resize', () => {
            const aspect = innerWidth / innerHeight;
            perspCamera.aspect = aspect;
            perspCamera.updateProjectionMatrix();
            
            orthoCamera.left = -orthoSize * aspect;
            orthoCamera.right = orthoSize * aspect;
            orthoCamera.updateProjectionMatrix();
            
            topCamera.left = -15 * aspect;
            topCamera.right = 15 * aspect;
            topCamera.updateProjectionMatrix();
            
            frontCamera.left = -12 * aspect;
            frontCamera.right = 12 * aspect;
            frontCamera.updateProjectionMatrix();
            
            renderer.setSize(innerWidth, innerHeight);
            renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
        });
        
        // GUI
        const gui = new GUI({ title: 'Camera Settings' });
        const params = {
            fov: 60,
            near: 0.1,
            far: 100,
            orthoSize: 10,
            autoRotate: false,
        };
        
        gui.add(params, 'fov', 20, 120, 1).onChange(v => {
            perspCamera.fov = v;
            perspCamera.updateProjectionMatrix();
        });
        gui.add(params, 'near', 0.01, 5, 0.01).onChange(v => {
            perspCamera.near = v;
            perspCamera.updateProjectionMatrix();
        });
        gui.add(params, 'far', 10, 500, 1).onChange(v => {
            perspCamera.far = v;
            perspCamera.updateProjectionMatrix();
        });
        gui.add(params, 'autoRotate').onChange(v => {
            orbitControls.autoRotate = v;
        });
    </script>
</body>
</html>
```

---

## สรุป Camera Types

| Camera | Constructor | ใช้เมื่อ |
|--------|------------|---------|
| PerspectiveCamera | `new THREE.PerspectiveCamera(fov, aspect, near, far)` | 3D games, apps, realism |
| OrthographicCamera | `new THREE.OrthographicCamera(left, right, top, bottom, near, far)` | 2D, UI, CAD, isometric |
| CubeCamera | `new THREE.CubeCamera(near, far, renderTarget)` | Real-time reflections |
| ArrayCamera | `new THREE.ArrayCamera(cameras[])` | VR, multi-viewport |

---

## แบบฝึกหัด (Exercises)

### Exercise 1: FOV Slider (ง่าย)
สร้าง slider ที่เปลี่ยน FOV ของ PerspectiveCamera จาก 20° ถึง 120°  
แสดงให้เห็นผลของ perspective distortion ที่แตกต่างกัน

### Exercise 2: Isometric Camera (ง่าย)
สร้าง isometric view ด้วย OrthographicCamera:
- Position: (10, 10, 10) และ lookAt (0, 0, 0)
- เพิ่ม objects เป็น city blocks หรือ terrain
- ไม่มี perspective distortion

### Exercise 3: Smooth Camera Transitions (ปานกลาง)
สร้าง 4 camera positions และ LERP ระหว่างกัน:
- Click buttons เพื่อ transition
- Smooth position + lookAt interpolation
- Easing function (ease in/out)

### Exercise 4: Security Camera System (ปานกลาง)
สร้าง CCTV-style scene:
- Main camera (player POV)
- Security cameras 4 ดวงใน scene
- PiP overlay แสดง feed จาก security cameras
- Switch ระหว่าง cameras ด้วย keyboard

### Exercise 5: Dynamic Reflection (ยาก)
สร้าง scene ที่มี mirror ball:
- CubeCamera สำหรับ real-time reflection
- Optimize ด้วยการ update ทุก 2-3 frames
- Animate objects ใน scene ให้เห็น reflection เปลี่ยน

---

**ก่อนหน้า:** [Part 009 - Lighting Part 2](part-009.md)  
**ต่อไป:** [Part 011 - Animation Basics](part-011.md)
