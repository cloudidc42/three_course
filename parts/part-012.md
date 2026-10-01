# Part 12: Animation Basics ใน Three.js

## บทนำ

Animation เป็นหัวใจของประสบการณ์ 3D ที่ดี ในบทนี้เราจะเรียนรู้พื้นฐานของ Animation ตั้งแต่ requestAnimationFrame loop ไปจนถึงการใช้ library อย่าง GSAP เพื่อสร้าง Animation ที่สวยงามและมีประสิทธิภาพ

## 1. requestAnimationFrame Loop

`requestAnimationFrame` คือ browser API ที่ให้เรารันโค้ดก่อนที่ browser จะ repaint หน้าจอ ทำให้ได้ animation ที่ลื่นไหลที่สุด

### ความแตกต่างจาก setInterval

```javascript
// ❌ วิธีแย่ - setInterval ไม่ sync กับ refresh rate ของหน้าจอ
setInterval(() => {
    // animate
    renderer.render(scene, camera);
}, 1000 / 60); // 60 FPS แบบ hardcode

// ✅ วิธีที่ถูก - requestAnimationFrame sync กับ refresh rate
function animate() {
    requestAnimationFrame(animate); // เรียกตัวเองซ้ำ
    renderer.render(scene, camera);
}
animate(); // เริ่ม loop
```

### ข้อดีของ requestAnimationFrame
1. **Sync กับ monitor refresh rate** - 60Hz, 90Hz, 120Hz, 144Hz ทำงานได้ถูกต้อง
2. **หยุดอัตโนมัติเมื่อ tab ไม่ active** - ประหยัด CPU/GPU
3. **Timing ที่แม่นยำ** - browser จัดการให้
4. **ไม่มี memory leak** ถ้า implement ถูกต้อง

### การหยุดและเริ่ม Animation Loop

```javascript
let animationId = null;

function startAnimation() {
    if (animationId) return; // ไม่ start ซ้ำ
    
    function animate() {
        animationId = requestAnimationFrame(animate);
        renderer.render(scene, camera);
    }
    animate();
}

function stopAnimation() {
    if (animationId) {
        cancelAnimationFrame(animationId);
        animationId = null;
    }
}

// ใช้งาน
startAnimation();
// หยุดชั่วคราว
stopAnimation();
// เริ่มใหม่
startAnimation();
```

---

## 2. Delta Time - หัวใจของ Frame-Rate Independent Animation

**Delta time** คือเวลาที่ผ่านไประหว่าง frame ปัจจุบันและ frame ที่แล้ว (เป็นวินาที)

### ทำไม Delta Time ถึงสำคัญ?

```javascript
// ❌ ปัญหา: ความเร็วขึ้นอยู่กับ frame rate
const mesh = new THREE.Mesh(geo, mat);
function animate() {
    requestAnimationFrame(animate);
    mesh.rotation.y += 0.01; // บน 60fps = 0.6 rad/s
                              // บน 120fps = 1.2 rad/s !! เร็วกว่า 2 เท่า
    renderer.render(scene, camera);
}

// ✅ แก้ไข: ใช้ delta time
let lastTime = 0;
function animate(currentTime) {
    requestAnimationFrame(animate);
    
    const delta = (currentTime - lastTime) / 1000; // แปลงเป็นวินาที
    lastTime = currentTime;
    
    mesh.rotation.y += 1.0 * delta; // 1.0 rad/s ไม่ว่าจะ FPS เท่าไร
    renderer.render(scene, camera);
}
animate(0);
```

### ปัญหากับ Delta Time แบบ Manual

```javascript
// ปัญหา: ถ้า tab ถูกซ่อนแล้วกลับมา delta อาจจะใหญ่มาก
// เช่น 5 วินาที = วัตถุกระโดดไปไกลมาก

function animate(currentTime) {
    requestAnimationFrame(animate);
    
    const rawDelta = (currentTime - lastTime) / 1000;
    // จำกัด delta ไม่ให้เกิน 0.1 วินาที (10 FPS minimum)
    const delta = Math.min(rawDelta, 0.1);
    lastTime = currentTime;
    
    // ใช้ delta ที่ clamped แล้ว
    mesh.rotation.y += 1.0 * delta;
}
```

---

## 3. THREE.Clock - เครื่องมือจับเวลาของ Three.js

`THREE.Clock` เป็น class ที่ Three.js มีให้สำหรับจัดการเวลาอย่างถูกต้อง

```javascript
const clock = new THREE.Clock();

// Methods หลัก:
// clock.getDelta()         - เวลาที่ผ่านไปตั้งแต่เรียกครั้งล่าสุด (วินาที)
// clock.getElapsedTime()  - เวลาทั้งหมดตั้งแต่เริ่ม Clock (วินาที)
// clock.start()           - เริ่ม/resume clock
// clock.stop()            - หยุด clock
// clock.running           - true ถ้ากำลัง running

function animate() {
    requestAnimationFrame(animate);
    
    const delta = clock.getDelta();           // 0.016 ที่ 60fps
    const elapsed = clock.getElapsedTime();   // เวลาสะสม
    
    // หมุน 1 รอบต่อวินาที
    mesh.rotation.y += Math.PI * 2 * delta;
    
    // ลอยขึ้นลงตามเวลา
    mesh.position.y = Math.sin(elapsed * 2) * 0.5;
    
    renderer.render(scene, camera);
}
```

### ข้อควรระวังกับ getDelta()

```javascript
// ⚠️ getDelta() รีเซ็ตทุกครั้งที่เรียก
const clock = new THREE.Clock();

function animate() {
    const delta = clock.getDelta(); // ถูกต้อง ✅
    
    // ❌ ห้ามเรียก getDelta() หลายครั้งใน frame เดียว!
    // const delta2 = clock.getDelta(); // จะได้ ~0 เพราะเพิ่งเรียกไป
    
    // ✅ ถ้าต้องใช้หลายที่ เก็บใน variable แล้วส่งต่อ
    updatePhysics(delta);
    updateAnimations(delta);
    updateParticles(delta);
}

// ✅ ถ้าต้องการทั้ง delta และ elapsed ใน frame เดียว
function animate() {
    const delta = clock.getDelta();        // เรียกนี้ก่อน
    const elapsed = clock.elapsedTime;    // อ่าน property โดยตรง (ไม่ reset)
    // clock.getElapsedTime() ก็ได้แต่ไม่ reset delta
}
```

---

## 4. GSAP (GreenSock Animation Platform) กับ Three.js

GSAP เป็น JavaScript Animation library ที่ทรงพลังที่สุด ใช้งานกับ Three.js ได้ดีมาก

### การติดตั้ง

```bash
npm install gsap
# หรือ
npm install gsap @types/gsap # สำหรับ TypeScript
```

```html
<!-- CDN -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.2/gsap.min.js"></script>
```

### gsap.to() พื้นฐาน

```javascript
import gsap from 'gsap';

const mesh = new THREE.Mesh(
    new THREE.BoxGeometry(1, 1, 1),
    new THREE.MeshStandardMaterial({ color: 0xff6b6b })
);
scene.add(mesh);

// เคลื่อนที่ไปตำแหน่ง x=5 ใน 2 วินาที
gsap.to(mesh.position, {
    x: 5,
    duration: 2,
    ease: 'power2.inOut'
});

// หมุน 360 องศา
gsap.to(mesh.rotation, {
    y: Math.PI * 2,
    duration: 3,
    ease: 'none',
    repeat: -1 // ทำซ้ำตลอด
});

// ขยาย scale
gsap.to(mesh.scale, {
    x: 2, y: 2, z: 2,
    duration: 1,
    ease: 'elastic.out(1, 0.3)'
});

// เปลี่ยนสี (ต้องใช้ .r .g .b)
gsap.to(mesh.material.color, {
    r: 0, g: 1, b: 0,  // สีเขียว
    duration: 1
});
```

### gsap.from() และ gsap.fromTo()

```javascript
// gsap.from() - เริ่มจากค่านี้ ไปยังค่าปัจจุบัน
gsap.from(mesh.position, {
    y: -5,        // เริ่มจาก y=-5
    duration: 1,  // ไปยัง position ปัจจุบัน
    ease: 'back.out(1.7)'
});

// gsap.fromTo() - กำหนดทั้งต้นและปลาย
gsap.fromTo(mesh.position,
    { y: -5 },      // จาก
    { y: 5, duration: 2, ease: 'power1.inOut', yoyo: true, repeat: -1 }  // ถึง
);
```

### GSAP Easing Functions

```javascript
// ประเภท ease หลัก:
// Linear: none
// Smooth: power1, power2, power3, power4
// Bounce: bounce
// Elastic: elastic
// Back (overshoot): back
// Circ: circ
// Expo: expo
// Sine: sine

// แต่ละ ease มี suffix:
// .in    - เริ่มช้า เร็วขึ้น
// .out   - เริ่มเร็ว ช้าลง
// .inOut - ช้า-เร็ว-ช้า

const easeExamples = [
    'none',           // เส้นตรง
    'power1.in',      // Quadratic ease in
    'power2.out',     // Cubic ease out
    'power3.inOut',   // Quartic ease in-out
    'elastic.out(1, 0.3)',   // Elastic bounce
    'back.out(1.7)',         // Overshoot
    'bounce.out',            // Bounce
    'sine.inOut',            // Sine wave
];
```

### gsap.timeline() - เรียง Animation ตามลำดับ

```javascript
// Timeline ช่วยเรียง animation หลายตัวให้เล่นต่อกัน

const tl = gsap.timeline({
    repeat: 0,          // ทำซ้ำกี่ครั้ง (-1 = ตลอด)
    yoyo: false,        // กลับทิศทาง
    paused: false,      // เริ่มเล่นทันที
    onComplete: () => console.log('Timeline จบแล้ว!')
});

// เพิ่ม animation ทีละตัว (เล่นต่อกัน)
tl.to(mesh.position, { x: 5, duration: 1, ease: 'power2.out' })
  .to(mesh.rotation, { y: Math.PI, duration: 0.5 })
  .to(mesh.scale, { x: 2, y: 2, z: 2, duration: 0.5, ease: 'back.out' })
  .to(mesh.position, { y: 3, duration: 0.8 })
  .to(mesh.position, { x: 0, y: 0, duration: 1, ease: 'power2.in' })
  .to(mesh.scale, { x: 1, y: 1, z: 1, duration: 0.3 });

// เพิ่มด้วย offset (เล่นพร้อมกัน หรือ delay)
tl.to(mesh2.position, { x: -5, duration: 1 }, '<')     // เริ่มพร้อมกัน
  .to(mesh3.position, { x: 0, duration: 1 }, '+=0.5')  // delay 0.5 วินาที
  .to(mesh4.rotation, { y: Math.PI }, '-=0.5');          // เริ่ม 0.5 ก่อนจบ

// Control Timeline
tl.play();
tl.pause();
tl.reverse();
tl.restart();
tl.seek(2); // ไปที่ 2 วินาที
tl.timeScale(2); // เล่น 2x speed
```

### Demo สมบูรณ์: GSAP Timeline Animation

```html
<!DOCTYPE html>
<html>
<head>
    <title>GSAP Three.js Animation</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #0a0a0a; overflow: hidden; }
        #controls {
            position: absolute;
            bottom: 20px;
            left: 50%;
            transform: translateX(-50%);
            display: flex;
            gap: 10px;
        }
        button {
            padding: 10px 20px;
            background: #e94560;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            font-size: 14px;
        }
        button:hover { background: #c73652; }
    </style>
</head>
<body>
<div id="controls">
    <button id="playBtn">▶ Play</button>
    <button id="pauseBtn">⏸ Pause</button>
    <button id="reverseBtn">↩ Reverse</button>
    <button id="restartBtn">↺ Restart</button>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.2/gsap.min.js"></script>
<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x0a0a0a);

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 3, 10);
camera.lookAt(0, 0, 0);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
document.body.appendChild(renderer.domElement);

// Lighting
scene.add(new THREE.AmbientLight(0xffffff, 0.5));
const pointLight1 = new THREE.PointLight(0xff6b6b, 2, 20);
pointLight1.position.set(5, 5, 5);
scene.add(pointLight1);
const pointLight2 = new THREE.PointLight(0x4ecdc4, 2, 20);
pointLight2.position.set(-5, 5, -5);
scene.add(pointLight2);

// สร้างวัตถุ
const objects = [];
const colors = [0xe94560, 0x4ecdc4, 0xffd700, 0x96ceb4, 0xff6b6b];

for (let i = 0; i < 5; i++) {
    const geo = new THREE.IcosahedronGeometry(0.7, 0);
    const mat = new THREE.MeshStandardMaterial({
        color: colors[i],
        roughness: 0.2,
        metalness: 0.8
    });
    const mesh = new THREE.Mesh(geo, mat);
    mesh.position.x = (i - 2) * 2.5;
    mesh.position.y = 0;
    mesh.scale.setScalar(0); // เริ่มด้วยขนาด 0
    scene.add(mesh);
    objects.push(mesh);
}

// สร้าง GSAP Timeline
const masterTL = gsap.timeline({ repeat: -1, repeatDelay: 1 });

// Phase 1: วัตถุปรากฏทีละตัว (stagger)
masterTL.to(objects.map(o => o.scale), {
    x: 1, y: 1, z: 1,
    duration: 0.8,
    ease: 'back.out(2)',
    stagger: 0.2
});

// Phase 2: กระโดดขึ้น
masterTL.to(objects.map(o => o.position), {
    y: 3,
    duration: 0.6,
    ease: 'power2.out',
    stagger: 0.1
}, '+=0.5');

// Phase 3: ตกลงมา
masterTL.to(objects.map(o => o.position), {
    y: 0,
    duration: 0.4,
    ease: 'bounce.out',
    stagger: 0.1
});

// Phase 4: หมุน 360
masterTL.to(objects.map(o => o.rotation), {
    y: Math.PI * 2,
    duration: 1,
    ease: 'power1.inOut',
    stagger: 0.1
});

// Phase 5: เรียงเป็นวงกลม
objects.forEach((obj, i) => {
    const angle = (i / objects.length) * Math.PI * 2;
    masterTL.to(obj.position, {
        x: Math.cos(angle) * 3,
        z: Math.sin(angle) * 3,
        y: 0,
        duration: 1,
        ease: 'power2.inOut'
    }, '<+=0.1');
});

// Phase 6: หมุนวง
masterTL.to(objects.map(o => o.position), {
    duration: 2,
    ease: 'none',
    onUpdate: function() {
        const t = this.progress();
        objects.forEach((obj, i) => {
            const baseAngle = (i / objects.length) * Math.PI * 2;
            const angle = baseAngle + t * Math.PI * 2;
            obj.position.x = Math.cos(angle) * 3;
            obj.position.z = Math.sin(angle) * 3;
        });
    }
});

// Phase 7: กลับมาจุดเดิม
objects.forEach((obj, i) => {
    masterTL.to(obj.position, {
        x: (i - 2) * 2.5,
        z: 0,
        duration: 1,
        ease: 'power2.inOut'
    }, '<+=0.1');
});

// Phase 8: หายไป
masterTL.to(objects.map(o => o.scale), {
    x: 0, y: 0, z: 0,
    duration: 0.5,
    ease: 'back.in(2)',
    stagger: 0.1
});

// Controls
document.getElementById('playBtn').addEventListener('click', () => masterTL.play());
document.getElementById('pauseBtn').addEventListener('click', () => masterTL.pause());
document.getElementById('reverseBtn').addEventListener('click', () => masterTL.reverse());
document.getElementById('restartBtn').addEventListener('click', () => masterTL.restart());

// Resize
window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});

// Render loop (ไม่ต้องจัดการ animation เอง GSAP ทำให้)
function animate() {
    requestAnimationFrame(animate);
    
    // หมุน objects ช้าๆ ตลอดเวลา
    objects.forEach((obj, i) => {
        obj.rotation.x += 0.005 * (i + 1) * 0.3;
    });
    
    renderer.render(scene, camera);
}
animate();
</script>
</body>
</html>
```

---

## 5. Bouncing Ball Demo พร้อม Physics

```javascript
import * as THREE from 'three';

// Scene setup
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x1a1a2e);

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 5, 15);
camera.lookAt(0, 0, 0);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.shadowMap.enabled = true;
document.body.appendChild(renderer.domElement);

// Lighting
scene.add(new THREE.AmbientLight(0xffffff, 0.3));
const dirLight = new THREE.DirectionalLight(0xffffff, 1);
dirLight.position.set(5, 20, 5);
dirLight.castShadow = true;
scene.add(dirLight);

// พื้น
const floor = new THREE.Mesh(
    new THREE.PlaneGeometry(20, 20),
    new THREE.MeshStandardMaterial({ color: 0x16213e })
);
floor.rotation.x = -Math.PI / 2;
floor.receiveShadow = true;
scene.add(floor);

// Ball class
class Ball {
    constructor(x, color, radius = 0.5) {
        this.mesh = new THREE.Mesh(
            new THREE.SphereGeometry(radius, 32, 32),
            new THREE.MeshStandardMaterial({ 
                color, 
                roughness: 0.3, 
                metalness: 0.2 
            })
        );
        this.mesh.position.set(x, 8, 0);
        this.mesh.castShadow = true;
        scene.add(this.mesh);
        
        this.radius = radius;
        this.velocity = 0;        // ความเร็วแนวตั้ง
        this.gravity = -20;       // แรงโน้มถ่วง
        this.bounce = 0.7 + Math.random() * 0.2;  // ค่า bounce (0-1)
        this.groundY = radius;    // ความสูงจากพื้น
        
        // Shadow blob
        this.shadow = new THREE.Mesh(
            new THREE.CircleGeometry(radius, 32),
            new THREE.MeshBasicMaterial({ color: 0x000000, opacity: 0.3, transparent: true })
        );
        this.shadow.rotation.x = -Math.PI / 2;
        this.shadow.position.y = 0.01;
        this.shadow.position.x = x;
        scene.add(this.shadow);
    }
    
    update(delta) {
        // ใช้ physics: v = v0 + a*t
        this.velocity += this.gravity * delta;
        this.mesh.position.y += this.velocity * delta;
        
        // Bounce เมื่อถึงพื้น
        if (this.mesh.position.y <= this.groundY) {
            this.mesh.position.y = this.groundY;
            this.velocity = -this.velocity * this.bounce; // ลดความเร็วลง
            
            // ถ้าความเร็วน้อยมาก หยุดเลย
            if (Math.abs(this.velocity) < 0.5) {
                this.velocity = 0;
            }
        }
        
        // อัพเดต shadow
        const height = this.mesh.position.y - this.groundY;
        const shadowScale = 1 - height * 0.05;
        this.shadow.scale.setScalar(Math.max(0.1, shadowScale));
        this.shadow.material.opacity = 0.3 * shadowScale;
        this.shadow.position.x = this.mesh.position.x;
        this.shadow.position.z = this.mesh.position.z;
        
        // Squash and stretch
        const squash = Math.max(0.5, 1 - Math.abs(this.velocity) * 0.02);
        this.mesh.scale.y = squash;
        this.mesh.scale.x = 1 / Math.sqrt(squash);
        this.mesh.scale.z = 1 / Math.sqrt(squash);
    }
    
    reset() {
        this.mesh.position.y = 8;
        this.velocity = 0;
        this.mesh.scale.set(1, 1, 1);
    }
}

// สร้างลูกบอลหลายสี
const ballColors = [0xe94560, 0x4ecdc4, 0xffd700, 0x96ceb4, 0xff9ff3];
const balls = ballColors.map((color, i) => 
    new Ball((i - 2) * 3, color)
);

// Reset button
document.addEventListener('keydown', (e) => {
    if (e.key === 'r' || e.key === 'R') {
        balls.forEach(ball => ball.reset());
    }
});

// Animation
const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    const delta = Math.min(clock.getDelta(), 0.05); // cap delta
    
    balls.forEach(ball => ball.update(delta));
    
    renderer.render(scene, camera);
}
animate();
```

---

## 6. GSAP ScrollTrigger กับ Three.js

ScrollTrigger ทำให้ animation ของ Three.js ตอบสนองต่อการ scroll ของหน้าเว็บ

```html
<!DOCTYPE html>
<html>
<head>
    <style>
        body { height: 500vh; background: #0a0a0a; }
        canvas { position: fixed; top: 0; left: 0; }
        .section {
            position: relative;
            height: 100vh;
            display: flex;
            align-items: center;
            justify-content: center;
            color: white;
            font-size: 48px;
            font-family: sans-serif;
            z-index: 10;
            pointer-events: none;
        }
    </style>
</head>
<body>
<div class="section" style="color: #e94560;">เริ่มต้น</div>
<div class="section" style="color: #4ecdc4;">หมุน</div>
<div class="section" style="color: #ffd700;">ขยาย</div>
<div class="section" style="color: #96ceb4;">เคลื่อนที่</div>
<div class="section" style="color: #ff9ff3;">จบ</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.2/gsap.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.2/ScrollTrigger.min.js"></script>
<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';

gsap.registerPlugin(ScrollTrigger);

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.z = 5;

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setClearColor(0x000000, 0); // transparent
document.body.insertBefore(renderer.domElement, document.body.firstChild);

// Lighting
scene.add(new THREE.AmbientLight(0xffffff, 0.5));
const pointLight = new THREE.PointLight(0xffffff, 2, 20);
pointLight.position.set(5, 5, 5);
scene.add(pointLight);

// วัตถุหลัก
const geo = new THREE.IcosahedronGeometry(1.5, 1);
const mat = new THREE.MeshStandardMaterial({
    color: 0xe94560,
    roughness: 0.2,
    metalness: 0.8,
    wireframe: false
});
const mesh = new THREE.Mesh(geo, mat);
scene.add(mesh);

// Wireframe overlay
const wireMat = new THREE.MeshBasicMaterial({ 
    color: 0xffffff, wireframe: true, opacity: 0.1, transparent: true 
});
const wireMesh = new THREE.Mesh(geo, wireMat);
scene.add(wireMesh);

// ScrollTrigger animations
// Section 1 -> 2: เปลี่ยนสี + หมุน
gsap.to(mat.color, {
    r: 78/255, g: 205/255, b: 196/255,  // #4ecdc4
    scrollTrigger: {
        trigger: '.section:nth-child(2)',
        start: 'top center',
        end: 'bottom center',
        scrub: true  // sync กับ scroll
    }
});

gsap.to(mesh.rotation, {
    y: Math.PI * 2,
    x: Math.PI,
    scrollTrigger: {
        trigger: '.section:nth-child(2)',
        start: 'top bottom',
        end: 'bottom top',
        scrub: 1  // 1 วินาที lag
    }
});

// Section 2 -> 3: ขยาย
gsap.to(mesh.scale, {
    x: 2, y: 2, z: 2,
    scrollTrigger: {
        trigger: '.section:nth-child(3)',
        start: 'top center',
        end: 'center center',
        scrub: true
    }
});

gsap.to(mat.color, {
    r: 1, g: 215/255, b: 0,  // gold
    scrollTrigger: {
        trigger: '.section:nth-child(3)',
        start: 'top center',
        end: 'center center',
        scrub: true
    }
});

// Section 3 -> 4: เคลื่อนที่
gsap.to(mesh.position, {
    x: 3,
    scrollTrigger: {
        trigger: '.section:nth-child(4)',
        start: 'top center',
        end: 'center center',
        scrub: true
    }
});

// Section 4 -> 5: กลับมา + wireframe
gsap.to(mesh.position, {
    x: 0,
    scrollTrigger: {
        trigger: '.section:nth-child(5)',
        start: 'top center',
        end: 'center center',
        scrub: true
    }
});

gsap.to(mesh.scale, {
    x: 1, y: 1, z: 1,
    scrollTrigger: {
        trigger: '.section:nth-child(5)',
        start: 'top center',
        end: 'center center',
        scrub: true
    }
});

gsap.to(wireMat, {
    opacity: 0.5,
    scrollTrigger: {
        trigger: '.section:nth-child(5)',
        start: 'top center',
        end: 'bottom center',
        scrub: true
    }
});

// Camera movement
gsap.to(camera.position, {
    z: 3,
    scrollTrigger: {
        trigger: 'body',
        start: 'top top',
        end: 'bottom bottom',
        scrub: 2
    }
});

window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});

const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    const delta = clock.getDelta();
    
    // หมุนช้าๆ ตลอดเวลา
    mesh.rotation.y += 0.2 * delta;
    wireMesh.rotation.copy(mesh.rotation);
    
    renderer.render(scene, camera);
}
animate();
</script>
</body>
</html>
```

---

## 7. Frame Rate Independence

### การทดสอบ Frame Rate Independence

```javascript
// ทดสอบว่า animation เป็น frame-rate independent จริงไหม
class FrameRateTest {
    constructor() {
        this.frames = 0;
        this.lastTime = 0;
        this.fps = 60;
        this.fpsHistory = [];
    }
    
    update(currentTime) {
        this.frames++;
        const delta = currentTime - this.lastTime;
        
        if (delta >= 1000) {
            this.fps = Math.round(this.frames * 1000 / delta);
            this.fpsHistory.push(this.fps);
            if (this.fpsHistory.length > 60) this.fpsHistory.shift();
            this.frames = 0;
            this.lastTime = currentTime;
        }
    }
    
    getAverageFPS() {
        return this.fpsHistory.reduce((a, b) => a + b, 0) / this.fpsHistory.length;
    }
}

// FPS Limiter - จำกัด FPS เพื่อประหยัดพลังงาน
class FPSLimiter {
    constructor(targetFPS = 60) {
        this.targetFPS = targetFPS;
        this.interval = 1000 / targetFPS;
        this.lastTime = 0;
    }
    
    shouldRender(currentTime) {
        if (currentTime - this.lastTime >= this.interval) {
            this.lastTime = currentTime;
            return true;
        }
        return false;
    }
}

const limiter = new FPSLimiter(30); // จำกัดที่ 30 FPS

function animate(time) {
    requestAnimationFrame(animate);
    
    if (!limiter.shouldRender(time)) return; // ข้าม frame
    
    renderer.render(scene, camera);
}
```

---

## 8. Animation Best Practices

```javascript
// ✅ Best Practice 1: ใช้ delta time เสมอ
function animate() {
    const delta = clock.getDelta();
    mesh.rotation.y += angularSpeed * delta;
}

// ✅ Best Practice 2: Cache expensive calculations
const vec3 = new THREE.Vector3(); // สร้างครั้งเดียว
function animate() {
    vec3.set(1, 0, 0);  // ใช้ซ้ำ ไม่สร้างใหม่
}

// ✅ Best Practice 3: ใช้ Object Pool สำหรับ particles
class ObjectPool {
    constructor(factory, size) {
        this.pool = Array.from({ length: size }, factory);
        this.index = 0;
    }
    
    get() {
        const obj = this.pool[this.index];
        this.index = (this.index + 1) % this.pool.length;
        return obj;
    }
}

// ✅ Best Practice 4: Throttle heavy operations
let frameCount = 0;
function animate() {
    frameCount++;
    
    // ทำ expensive operation ทุก 10 frames
    if (frameCount % 10 === 0) {
        updateAI();
        checkCollisions();
    }
    
    // ทำ critical operation ทุก frame
    renderer.render(scene, camera);
}

// ✅ Best Practice 5: ใช้ Performance.now() สำหรับ profiling
function profiledAnimate() {
    const start = performance.now();
    
    updateScene();
    renderer.render(scene, camera);
    
    const end = performance.now();
    if (end - start > 16) { // เกิน 16ms = ต่ำกว่า 60fps
        console.warn(`Slow frame: ${(end - start).toFixed(2)}ms`);
    }
}
```

---

## 9. Tween.js พื้นฐาน

```javascript
// ติดตั้ง: npm install @tweenjs/tween.js
import TWEEN from '@tweenjs/tween.js';

const coords = { x: 0, y: 0 };

// สร้าง Tween
const tween = new TWEEN.Tween(coords)
    .to({ x: 300, y: 200 }, 2000)  // target, duration(ms)
    .easing(TWEEN.Easing.Quadratic.Out)
    .onUpdate(() => {
        mesh.position.x = coords.x * 0.01;
        mesh.position.y = coords.y * 0.01;
    })
    .onComplete(() => {
        console.log('Tween เสร็จแล้ว!');
    })
    .start();

// อย่าลืมเรียก TWEEN.update() ใน loop
function animate(time) {
    requestAnimationFrame(animate);
    TWEEN.update(time);  // สำคัญ!
    renderer.render(scene, camera);
}
```

---

## แบบฝึกหัด

1. **Ball Simulation**: เพิ่มฟิสิกส์ให้ลูกบอลกระดอนโดยใช้ delta time ให้ถูกต้อง
2. **GSAP Morphing**: ใช้ GSAP animate การเปลี่ยนสีและ scale ของวัตถุ 10 ตัวพร้อมกัน
3. **ScrollTrigger Scene**: สร้างหน้าเว็บที่มี 3D animation ตอบสนองต่อ scroll
4. **FPS Monitor**: สร้าง FPS counter แสดงบน DOM และวาดกราฟ FPS
5. **Tween Chain**: ใช้ GSAP timeline สร้าง animation 5 ขั้นตอนต่อเนื่องกัน

---

## สรุป

- ใช้ `requestAnimationFrame` แทน `setInterval` เสมอ
- **Delta time** คือกุญแจของ frame-rate independence
- `THREE.Clock.getDelta()` ให้ค่า delta ที่ถูกต้องและ clamp แล้ว
- **GSAP** เหมาะสำหรับ complex timeline animations และ easing
- **ScrollTrigger** เชื่อม 3D animation กับ scroll position
- Cache objects ที่สร้างบ่อย เช่น Vector3, Quaternion เพื่อประสิทธิภาพ
