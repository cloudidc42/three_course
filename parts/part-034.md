# Part 34: Scroll-Based Animation

## บทนำ: Scroll + Three.js

Scroll-based animation คือเทคนิคที่นิยมมากสำหรับ **portfolio** และ **landing page** โดย:
- ตำแหน่ง scroll ควบคุม 3D animation
- Camera เคลื่อนที่ไปตาม scroll
- Object แต่ละ section ปรากฏ/หายไปตาม scroll

---

## แนวทางพื้นฐาน: Window Scroll

```javascript
// วิธีง่ายสุด: ใช้ window.scrollY
let scrollY = 0;
window.addEventListener('scroll', () => {
    scrollY = window.scrollY;
});

function animate() {
    requestAnimationFrame(animate);
    
    // Normalize scroll (0 → 1 ตลอด page)
    const progress = scrollY / (document.body.scrollHeight - window.innerHeight);
    
    // ใช้ progress ขยับ camera
    camera.position.y = -progress * 20;
    
    renderer.render(scene, camera);
}
```

---

## LERP สำหรับ Smooth Scrolling

```javascript
// Linear Interpolation ทำให้การเคลื่อนที่ราบรื่นขึ้น
let currentScrollY = 0;
let targetScrollY = 0;

window.addEventListener('scroll', () => {
    targetScrollY = window.scrollY;
});

function animate() {
    requestAnimationFrame(animate);
    
    // Lerp: currentScrollY ไล่ตาม targetScrollY ช้าๆ
    currentScrollY += (targetScrollY - currentScrollY) * 0.08;
    
    const progress = currentScrollY / (document.body.scrollHeight - window.innerHeight);
    camera.position.y = THREE.MathUtils.lerp(0, -20, progress);
    
    renderer.render(scene, camera);
}
```

---

## Section-Based Scroll

```javascript
// แบ่ง page เป็น sections, แต่ละ section trigger animation ต่างกัน

const sections = document.querySelectorAll('.section');
const sectionCount = sections.length;

function getScrollSection() {
    const sectionHeight = window.innerHeight;
    const scrollIndex = window.scrollY / sectionHeight;
    const sectionIndex = Math.floor(scrollIndex);
    const sectionProgress = scrollIndex % 1;  // 0 → 1 ภายใน section
    
    return {
        index: Math.min(sectionIndex, sectionCount - 1),
        progress: sectionProgress
    };
}

function animate() {
    requestAnimationFrame(animate);
    
    const { index, progress } = getScrollSection();
    
    switch (index) {
        case 0:
            // Section 1: object หมุน
            mesh.rotation.y = progress * Math.PI * 2;
            break;
        case 1:
            // Section 2: camera เลื่อนเข้า
            camera.position.z = THREE.MathUtils.lerp(10, 2, progress);
            break;
        case 2:
            // Section 3: object ขยาย
            mesh.scale.setScalar(THREE.MathUtils.lerp(1, 3, progress));
            break;
    }
    
    renderer.render(scene, camera);
}
```

---

## Camera Path Animation on Scroll

```javascript
// Camera เดินตาม CatmullRom path
const cameraPath = new THREE.CatmullRomCurve3([
    new THREE.Vector3(0,  0, 10),
    new THREE.Vector3(5,  2,  5),
    new THREE.Vector3(0,  5,  0),
    new THREE.Vector3(-5, 2, -5),
    new THREE.Vector3(0,  0, -10)
], false);

// ที่ camera มอง
const lookAtPath = new THREE.CatmullRomCurve3([
    new THREE.Vector3(0, 0, 0),
    new THREE.Vector3(3, 1, 0),
    new THREE.Vector3(0, 2, 0),
    new THREE.Vector3(-3, 1, 0),
    new THREE.Vector3(0, 0, 0)
], false);

let scrollProgress = 0;
let targetProgress = 0;

window.addEventListener('scroll', () => {
    targetProgress = window.scrollY / (document.body.scrollHeight - window.innerHeight);
});

const cameraPos   = new THREE.Vector3();
const lookAtPoint = new THREE.Vector3();

function animate() {
    requestAnimationFrame(animate);
    
    scrollProgress += (targetProgress - scrollProgress) * 0.05;
    
    cameraPath.getPoint(scrollProgress, cameraPos);
    lookAtPath.getPoint(scrollProgress, lookAtPoint);
    
    camera.position.copy(cameraPos);
    camera.lookAt(lookAtPoint);
    
    renderer.render(scene, camera);
}
```

---

## Parallax 3D Layers

```javascript
// เลเยอร์ที่ scroll ที่ความเร็วต่างกัน สร้าง depth effect

const layers = [
    { mesh: backgroundMesh, speed: 0.1 },
    { mesh: middleMesh,     speed: 0.4 },
    { mesh: foregroundMesh, speed: 1.0 }
];

window.addEventListener('scroll', () => {
    const scrollY = window.scrollY;
    for (const { mesh, speed } of layers) {
        mesh.position.y = scrollY * speed * -0.01;
    }
});
```

---

## Scroll-Based Morph

```javascript
// Morph vertices ตาม scroll progress
const originalPositions = geometry.attributes.position.array.slice();
const targetPositions   = targetGeometry.attributes.position.array.slice();

window.addEventListener('scroll', () => {
    const t = window.scrollY / maxScroll;
    const pos = geometry.attributes.position;
    
    for (let i = 0; i < pos.count; i++) {
        pos.setX(i, THREE.MathUtils.lerp(originalPositions[i*3],   targetPositions[i*3],   t));
        pos.setY(i, THREE.MathUtils.lerp(originalPositions[i*3+1], targetPositions[i*3+1], t));
        pos.setZ(i, THREE.MathUtils.lerp(originalPositions[i*3+2], targetPositions[i*3+2], t));
    }
    pos.needsUpdate = true;
    geometry.computeVertexNormals();
});
```

---

## Full Scroll Portfolio Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>3D Scroll Portfolio</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        
        body {
            background: #050a12;
            font-family: 'Helvetica Neue', sans-serif;
            color: white;
        }
        
        /* Canvas fixed ทับ background */
        canvas {
            position: fixed;
            top: 0;
            left: 0;
            z-index: 0;
        }
        
        /* Content ทับบน canvas */
        #content {
            position: relative;
            z-index: 1;
            pointer-events: none;
        }
        
        .section {
            height: 100vh;
            display: flex;
            align-items: center;
            justify-content: center;
            padding: 0 10%;
        }
        
        .section-text {
            max-width: 500px;
        }
        
        .section-text.right {
            margin-left: auto;
        }
        
        h1 { font-size: clamp(2rem, 5vw, 4rem); font-weight: 700; line-height: 1.1; }
        h2 { font-size: clamp(1.5rem, 3vw, 2.5rem); font-weight: 600; margin-bottom: 1rem; }
        p  { font-size: 1.1rem; color: rgba(255,255,255,0.7); line-height: 1.7; }
        
        .accent { color: #4af; }
        .divider { width: 60px; height: 3px; background: #4af; margin: 1rem 0; }
        
        #progress-bar {
            position: fixed;
            top: 0;
            left: 0;
            height: 3px;
            background: linear-gradient(90deg, #4af, #f4a);
            z-index: 100;
            transition: none;
        }
    </style>
</head>
<body>
<div id="progress-bar"></div>
<canvas id="c"></canvas>
<div id="content">
    <!-- Section 1: Hero -->
    <section class="section" id="s1">
        <div class="section-text">
            <h1>Creative<br><span class="accent">3D</span> Design</h1>
            <div class="divider"></div>
            <p>ประสบการณ์ 3D แบบ interactive ที่สร้างขึ้นด้วย Three.js และ WebGL</p>
        </div>
    </section>
    <!-- Section 2 -->
    <section class="section" id="s2">
        <div class="section-text right">
            <h2>About</h2>
            <div class="divider"></div>
            <p>เราสร้าง web experience ที่ไม่เหมือนใคร ผสมผสานระหว่าง design และ technology</p>
        </div>
    </section>
    <!-- Section 3 -->
    <section class="section" id="s3">
        <div class="section-text">
            <h2>Services</h2>
            <div class="divider"></div>
            <p>3D Web · Interactive Experiences · Shader Art · Animation</p>
        </div>
    </section>
    <!-- Section 4 -->
    <section class="section" id="s4">
        <div class="section-text right">
            <h2>Contact</h2>
            <div class="divider"></div>
            <p>พร้อมที่จะสร้างสิ่งที่น่าทึ่งด้วยกันไหม?</p>
        </div>
    </section>
</div>

<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';

// === Setup ===
const canvas = document.getElementById('c');
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x050a12);

const camera = new THREE.PerspectiveCamera(75, innerWidth / innerHeight, 0.1, 100);
camera.position.set(0, 0, 5);

const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));

// Lights
scene.add(new THREE.AmbientLight(0xffffff, 0.3));
const pointLight1 = new THREE.PointLight(0x4488ff, 2, 20);
pointLight1.position.set(5, 5, 5);
scene.add(pointLight1);
const pointLight2 = new THREE.PointLight(0xff4488, 2, 20);
pointLight2.position.set(-5, -5, -5);
scene.add(pointLight2);

// === Objects per section ===
// Section 1: Icosahedron
const geo1 = new THREE.IcosahedronGeometry(1.5, 2);
const mat1 = new THREE.MeshStandardMaterial({
    color: 0x4488ff, wireframe: false,
    roughness: 0.2, metalness: 0.8
});
const obj1 = new THREE.Mesh(geo1, mat1);
obj1.position.set(3, 0, 0);
scene.add(obj1);

// Section 2: Torus
const obj2 = new THREE.Mesh(
    new THREE.TorusKnotGeometry(1.2, 0.3, 100, 16),
    new THREE.MeshStandardMaterial({ color: 0xff4488, roughness: 0.1, metalness: 0.9 })
);
obj2.position.set(-3, -8, 0);
scene.add(obj2);

// Section 3: Octahedron cluster
const obj3Group = new THREE.Group();
obj3Group.position.set(3, -16, 0);
scene.add(obj3Group);
for (let i = 0; i < 6; i++) {
    const o = new THREE.Mesh(
        new THREE.OctahedronGeometry(0.5),
        new THREE.MeshStandardMaterial({
            color: new THREE.Color().setHSL(i / 6, 0.8, 0.5),
            roughness: 0.2, metalness: 0.7
        })
    );
    const angle = (i / 6) * Math.PI * 2;
    o.position.set(Math.cos(angle) * 1.5, Math.sin(angle) * 1.5, 0);
    obj3Group.add(o);
}

// Section 4: Sphere with particles
const obj4 = new THREE.Mesh(
    new THREE.SphereGeometry(1.2, 32, 32),
    new THREE.MeshStandardMaterial({
        color: 0x44ffaa, roughness: 0.1, metalness: 0.8, wireframe: true
    })
);
obj4.position.set(-3, -24, 0);
scene.add(obj4);

// Particles background
const pCount = 2000;
const pGeo = new THREE.BufferGeometry();
const pPos = new Float32Array(pCount * 3);
for (let i = 0; i < pCount * 3; i++) {
    pPos[i] = (Math.random() - 0.5) * 50;
}
pGeo.setAttribute('position', new THREE.BufferAttribute(pPos, 3));
const particles = new THREE.Points(
    pGeo,
    new THREE.PointsMaterial({ color: 0xffffff, size: 0.05, sizeAttenuation: true })
);
scene.add(particles);

// === Scroll Logic ===
let scrollY = 0;
let currentScrollY = 0;
let sectionCount = 4;
const sectionHeight = window.innerHeight;

window.addEventListener('scroll', () => {
    scrollY = window.scrollY;
    const maxScroll = document.body.scrollHeight - window.innerHeight;
    document.getElementById('progress-bar').style.width = (scrollY / maxScroll * 100) + '%';
});

// Mouse parallax
const mouse = { x: 0, y: 0 };
window.addEventListener('mousemove', e => {
    mouse.x = (e.clientX / innerWidth  - 0.5) * 2;
    mouse.y = (e.clientY / innerHeight - 0.5) * 2;
});

const clock = new THREE.Clock();

function animate() {
    requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    
    // Smooth scroll
    currentScrollY += (scrollY - currentScrollY) * 0.08;
    
    // Camera Y: scroll down = camera moves down
    camera.position.y = -currentScrollY / sectionHeight * 8 * (sectionCount > 1 ? 1 : 0);
    
    // Mouse parallax on camera
    camera.position.x += (mouse.x * 0.3 - camera.position.x) * 0.05;
    
    // Per-section scroll progress (0 → 1)
    const scrollIndex    = currentScrollY / sectionHeight;
    const sectionIndex   = Math.floor(scrollIndex);
    const sectionProg    = scrollIndex % 1;
    
    // --- Section 1 animations ---
    obj1.rotation.y = elapsed * 0.5;
    obj1.rotation.x = elapsed * 0.3;
    if (sectionIndex === 0) {
        obj1.scale.setScalar(THREE.MathUtils.lerp(0.5, 1.2, sectionProg));
        mat1.wireframe = sectionProg > 0.5;
    }
    
    // --- Section 2 animations ---
    obj2.rotation.x = elapsed * 0.3;
    obj2.rotation.y = elapsed * 0.6;
    if (sectionIndex === 1) {
        obj2.rotation.z = sectionProg * Math.PI;
    }
    
    // --- Section 3 animations ---
    obj3Group.rotation.z = elapsed * 0.4;
    obj3Group.children.forEach((o, i) => {
        o.rotation.y = elapsed * (1 + i * 0.1);
        if (sectionIndex === 2) {
            const t = sectionProg;
            const baseAngle = (i / 6) * Math.PI * 2;
            const r = 1.5 + Math.sin(t * Math.PI) * 1.5;
            o.position.x = Math.cos(baseAngle + t * Math.PI) * r;
            o.position.y = Math.sin(baseAngle + t * Math.PI) * r;
        }
    });
    
    // --- Section 4 animations ---
    obj4.rotation.y = elapsed * 0.8;
    if (sectionIndex === 3) {
        obj4.scale.setScalar(THREE.MathUtils.lerp(1, 2, sectionProg));
    }
    
    // Particles slow rotation
    particles.rotation.y = elapsed * 0.02;
    
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

## Locomotive Scroll Integration

```javascript
// Locomotive Scroll ให้ smooth scroll ที่ดีกว่า native
// npm install locomotive-scroll

import LocomotiveScroll from 'locomotive-scroll';

const scroll = new LocomotiveScroll({
    el: document.querySelector('[data-scroll-container]'),
    smooth: true,
    multiplier: 1.0,
    lerp: 0.06
});

// Listen to scroll event
scroll.on('scroll', ({ scroll }) => {
    const progress = scroll.y / (document.body.scrollHeight - window.innerHeight);
    // ใช้ progress กับ Three.js
    updateThreeJS(progress);
});

// Scroll section trigger
scroll.on('call', (func, way, obj) => {
    if (func === 'enterSection') {
        // obj เป็น element ที่ trigger
        const sectionId = obj.el.dataset.section;
        triggerSectionAnimation(sectionId, way === 'enter');
    }
});
```

---

## Intersection Observer สำหรับ Section Trigger

```javascript
// ไม่ต้องพึ่ง library: ใช้ IntersectionObserver built-in

const observer = new IntersectionObserver(
    (entries) => {
        for (const entry of entries) {
            const sectionId = entry.target.id;
            if (entry.isIntersecting) {
                activateSection(sectionId);
            } else {
                deactivateSection(sectionId);
            }
        }
    },
    {
        threshold: 0.5,    // 50% of section visible
        rootMargin: '0px'
    }
);

document.querySelectorAll('.section').forEach(s => observer.observe(s));

function activateSection(id) {
    console.log(`Section ${id} activated`);
    // เริ่ม animation สำหรับ section นี้
}
```

---

## Multi-Section 3D Experience

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Multi-Section 3D</title>
    <style>
        * { margin: 0; padding: 0; }
        canvas { position: fixed; top: 0; left: 0; z-index: 0; }
        #page { position: relative; z-index: 1; pointer-events: none; }
        .section {
            height: 100vh;
            display: flex;
            align-items: center;
            padding: 0 8%;
            color: white;
            font-family: sans-serif;
        }
        .text-block { max-width: 400px; }
        .text-block.right { margin-left: auto; }
        h2 { font-size: 3rem; font-weight: 900; margin-bottom: 10px; }
        p  { font-size: 1.1rem; opacity: 0.7; line-height: 1.7; }
        .tag { display: inline-block; padding: 4px 14px; background: rgba(255,255,255,0.1);
               border-radius: 20px; font-size: 0.85rem; margin-top: 12px; }
        
        /* Scroll indicator */
        #scroll-hint {
            position: fixed;
            bottom: 30px;
            left: 50%;
            transform: translateX(-50%);
            color: rgba(255,255,255,0.5);
            font-size: 0.85rem;
            font-family: sans-serif;
            animation: bounce 2s infinite;
            z-index: 10;
        }
        @keyframes bounce {
            0%, 100% { transform: translateX(-50%) translateY(0); }
            50% { transform: translateX(-50%) translateY(-10px); }
        }
    </style>
</head>
<body>
<canvas id="c"></canvas>
<div id="scroll-hint">↓ Scroll</div>
<div id="page">
    <div class="section">
        <div class="text-block">
            <h2>Geometry</h2>
            <p>สำรวจโลกของ 3D geometry ที่ซับซ้อนและสวยงาม</p>
            <span class="tag">Three.js</span>
        </div>
    </div>
    <div class="section">
        <div class="text-block right">
            <h2>Shaders</h2>
            <p>GLSL shaders สร้าง visual effects ที่ไม่มีขีดจำกัด</p>
            <span class="tag">GLSL</span>
        </div>
    </div>
    <div class="section">
        <div class="text-block">
            <h2>Physics</h2>
            <p>จำลองโลกกายภาพด้วย physics engine</p>
            <span class="tag">Cannon.js</span>
        </div>
    </div>
    <div class="section">
        <div class="text-block right">
            <h2>Animation</h2>
            <p>ชีวิตใน 3D ด้วย scroll และ GSAP animation</p>
            <span class="tag">GSAP</span>
        </div>
    </div>
</div>

<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';

const canvas = document.getElementById('c');
const W = innerWidth, H = innerHeight;

const renderer = new THREE.WebGLRenderer({ canvas, antialias: true, alpha: false });
renderer.setSize(W, H);
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x08091a);

const camera = new THREE.PerspectiveCamera(70, W / H, 0.1, 100);
camera.position.z = 6;

// Lights
scene.add(new THREE.AmbientLight(0xffffff, 0.4));
const light1 = new THREE.DirectionalLight(0x6699ff, 1);
light1.position.set(5, 5, 5);
scene.add(light1);
const light2 = new THREE.PointLight(0xff6644, 2, 15);
light2.position.set(-5, 0, 0);
scene.add(light2);

// 4 objects for 4 sections
function makeSection(geo, color, x, yOffset) {
    const mesh = new THREE.Mesh(
        geo,
        new THREE.MeshStandardMaterial({ color, roughness: 0.3, metalness: 0.7 })
    );
    mesh.position.set(x, yOffset, 0);
    scene.add(mesh);
    return mesh;
}

const SECTION_Y_STEP = 6;
const meshes = [
    makeSection(new THREE.TorusKnotGeometry(1, 0.35, 100, 16),   0x4488ff,  2.5,  0),
    makeSection(new THREE.IcosahedronGeometry(1.3, 1),           0xff4488, -2.5, -SECTION_Y_STEP),
    makeSection(new THREE.DodecahedronGeometry(1.3),             0x44ff88,  2.5, -SECTION_Y_STEP * 2),
    makeSection(new THREE.OctahedronGeometry(1.4),               0xffaa22, -2.5, -SECTION_Y_STEP * 3)
];

// Wireframe overlays
const wireMeshes = meshes.map(m => {
    const w = new THREE.Mesh(
        m.geometry,
        new THREE.MeshBasicMaterial({ color: 0xffffff, wireframe: true, transparent: true, opacity: 0.1 })
    );
    w.position.copy(m.position);
    scene.add(w);
    return w;
});

// Particles
const pCount = 3000;
const pPos = new Float32Array(pCount * 3);
for (let i = 0; i < pCount * 3; i++) pPos[i] = (Math.random() - 0.5) * 30;
const pGeo = new THREE.BufferGeometry();
pGeo.setAttribute('position', new THREE.BufferAttribute(pPos, 3));
scene.add(new THREE.Points(pGeo, new THREE.PointsMaterial({ color: 0x888888, size: 0.04, sizeAttenuation: true })));

// Scroll handling
let scrollY = 0;
let smoothScrollY = 0;
window.addEventListener('scroll', () => { scrollY = window.scrollY; });

// Hide scroll hint after first scroll
window.addEventListener('scroll', () => {
    document.getElementById('scroll-hint').style.opacity = '0';
}, { once: true });

// Mouse
const mouse = { x: 0, y: 0 };
window.addEventListener('mousemove', e => {
    mouse.x = (e.clientX / innerWidth  - 0.5);
    mouse.y = (e.clientY / innerHeight - 0.5);
});

const clock = new THREE.Clock();

function animate() {
    requestAnimationFrame(animate);
    const t = clock.getElapsedTime();

    // Smooth scroll
    smoothScrollY += (scrollY - smoothScrollY) * 0.07;
    const totalHeight = document.body.scrollHeight - innerHeight;
    const progress = Math.min(smoothScrollY / totalHeight, 1);

    // Camera follows scroll (Y axis)
    const targetCamY = -(progress * SECTION_Y_STEP * (meshes.length - 1));
    camera.position.y += (targetCamY - camera.position.y) * 0.07;
    camera.position.x += (mouse.x * 0.5 - camera.position.x) * 0.05;

    // Per-section index
    const scrollIndex = smoothScrollY / innerHeight;
    const sIdx = Math.floor(scrollIndex);
    const sProg = scrollIndex - sIdx;

    // Animate each mesh
    meshes.forEach((m, i) => {
        m.rotation.x = t * 0.2 + i * 0.5;
        m.rotation.y = t * 0.3 + i * 0.7;
        wireMeshes[i].rotation.copy(m.rotation);
        
        // Scale based on visibility
        const dist = Math.abs(i - scrollIndex);
        const scale = THREE.MathUtils.clamp(1.5 - dist * 0.8, 0.3, 1.5);
        m.scale.setScalar(scale);
        wireMeshes[i].scale.setScalar(scale * 1.02);
        
        // Emissive glow
        const glow = Math.max(0, 1 - dist);
        m.material.emissive.set(m.material.color).multiplyScalar(glow * 0.2);
    });

    // Light follows camera with offset
    light2.position.set(-5 + mouse.x * 2, camera.position.y, 2);

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

## แบบฝึกหัด

### ระดับ 1
1. สร้าง scroll-triggered camera path ง่ายๆ
2. Scroll ขยับ object rotation
3. Parallax layers 3 ระดับ

### ระดับ 2
4. 4-section portfolio พร้อม text
5. Camera path ที่ซับซ้อน (spiral path)
6. Scroll-morphing geometry (sphere → box → torus)

### ระดับ 3
7. Particle explosion เมื่อ scroll ถึง section
8. Smooth camera transition ระหว่าง sections
9. Mobile touch support สำหรับ scroll

### Challenge
10. สร้าง Full Portfolio Website:
    - Hero section พร้อม animated 3D title
    - Projects section: แต่ละ project มี 3D card
    - Skills section: 3D bar chart animated บน scroll
    - Contact section: 3D form พร้อม physics
    - Progress indicator
    - Mobile responsive
