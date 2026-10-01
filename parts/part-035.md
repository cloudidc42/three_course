# Part 35: GSAP & Advanced Animation Techniques

## บทนำ: GSAP + Three.js

**GSAP (GreenSock Animation Platform)** คือ animation library ที่ทรงพลังที่สุด:
- Timeline-based animation
- Easing functions ครบ (elastic, bounce, back, etc.)
- Stagger effects
- ScrollTrigger plugin
- Morphing between positions

```html
<script src="https://cdn.jsdelivr.net/npm/gsap@3.12.2/dist/gsap.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/gsap@3.12.2/dist/ScrollTrigger.min.js"></script>
```

---

## gsap.to กับ Three.js Properties

```javascript
// GSAP สามารถ animate property ของ Three.js object ได้โดยตรง

// ขยับ position
gsap.to(mesh.position, {
    x: 5,
    y: 2,
    duration: 1.5,
    ease: 'power3.out'
});

// หมุน
gsap.to(mesh.rotation, {
    y: Math.PI * 2,
    duration: 2,
    ease: 'back.inOut(1.7)'
});

// ขยาย
gsap.to(mesh.scale, {
    x: 2, y: 2, z: 2,
    duration: 0.8,
    ease: 'elastic.out(1.2, 0.4)'
});

// เปลี่ยนสี (ต้องใช้ gsap plugin สำหรับ THREE.Color)
gsap.to(mesh.material, {
    opacity: 0,
    duration: 1
});

// Camera
gsap.to(camera.position, {
    x: 5, y: 3, z: 10,
    duration: 2,
    ease: 'power2.inOut',
    onUpdate: () => camera.lookAt(0, 0, 0)
});

// Custom value (ไม่ใช่ property ตรงๆ)
gsap.to({ value: 0 }, {
    value: 1,
    duration: 2,
    onUpdate: function() {
        mesh.material.color.lerpColors(colorA, colorB, this.targets()[0].value);
    }
});
```

---

## GSAP Timeline

```javascript
// Timeline ทำให้ animation หลายชิ้นทำงานต่อเนื่องหรือซ้อนกัน

const tl = gsap.timeline({
    defaults: { ease: 'power3.out', duration: 1 }
});

// Animation ต่อเนื่อง
tl.to(mesh1.position, { y: 3 })
  .to(mesh2.position, { x: 3 }, '-=0.5')   // เริ่มก่อน prev จบ 0.5s
  .to(mesh3.rotation, { y: Math.PI })
  .to(camera.position, { z: 5 }, '+=0.2')   // หน่วง 0.2s หลัง prev

// Named labels
tl.addLabel('phase2')
  .to(mesh1.scale, { x: 2, y: 2, z: 2 }, 'phase2')
  .to(mesh2.scale, { x: 0.5, y: 0.5, z: 0.5 }, 'phase2+=0.3')

// Loop
const loopTl = gsap.timeline({ repeat: -1, yoyo: true });
loopTl.to(mesh.position, { y: 2, duration: 1.5 });

// Pause, play, reverse, restart
tl.pause();
tl.play();
tl.reverse();
tl.restart();
tl.seek(2.5);  // jump to 2.5s
```

---

## Easing Functions ใน GSAP

```javascript
// Power easings: power1 → power4 (soft → sharp)
gsap.to(obj.position, { x: 5, ease: 'power2.out' });

// Special easings
gsap.to(obj.scale, { x: 2, ease: 'elastic.out(1, 0.3)' });   // กระเด้ง
gsap.to(obj.position, { y: 3, ease: 'bounce.out' });           // เด้ง
gsap.to(obj.position, { x: 5, ease: 'back.out(2)' });         // โอเวอร์ชู้ต
gsap.to(obj.rotation, { y: Math.PI, ease: 'expo.inOut' });     // เร่ง-ชะลอรุนแรง

// Custom ease
const customEase = gsap.parseEase('M0,0 C0.25,0 0.25,1 0.5,1 C0.75,1 0.75,0 1,0');
gsap.to(obj.position, { x: 5, ease: customEase });
```

---

## Stagger Effects

```javascript
// Stagger: animate หลาย object โดยหน่วงเวลาทีละนิด

const meshArray = scene.children.filter(c => c.isMesh);

// จาก array
gsap.to(meshArray.map(m => m.position), {
    y: 3,
    duration: 1,
    stagger: 0.1,   // หน่วง 0.1s ต่อ object
    ease: 'power3.out'
});

// Stagger pattern
gsap.to(targets, {
    opacity: 1,
    stagger: {
        amount: 1.5,       // เวลารวมของ stagger
        from: 'center',    // เริ่มจากตรงกลาง (end, edges, random)
        grid: [5, 5],      // ถ้าจัดเป็น grid
        axis: 'x'          // stagger ในแนวแกน x
    }
});
```

---

## ScrollTrigger กับ Three.js

```javascript
import { ScrollTrigger } from 'gsap/ScrollTrigger';
gsap.registerPlugin(ScrollTrigger);

// Trigger animation เมื่อ scroll ถึงจุดหนึ่ง
gsap.to(mesh.rotation, {
    y: Math.PI * 2,
    scrollTrigger: {
        trigger: '#section2',    // element ที่ใช้ trigger
        start: 'top center',     // เมื่อ top ของ element ถึง center ของ viewport
        end: 'bottom center',    // จบเมื่อ bottom ถึง center
        scrub: true,             // animation ตาม scroll position (true = smooth)
        markers: false           // debug markers
    }
});

// scrub: true = smooth follow scroll
// scrub: 0.5 = หน่วง 0.5s ตาม scroll
// scrub: 2 = หน่วง 2s

// Pin section
ScrollTrigger.create({
    trigger: '#section3',
    start: 'top top',
    end: '+=500',              // pin ไว้ 500px scroll
    pin: true,
    pinSpacing: true
});

// Callback
gsap.timeline({
    scrollTrigger: {
        trigger: '#section4',
        start: 'top 80%',
        onEnter: () => {
            console.log('Entered section 4');
            playSection4Animation();
        },
        onLeave: () => {
            console.log('Left section 4');
        },
        onEnterBack: () => {
            console.log('Scrolled back into section 4');
        }
    }
})
.from(mesh.position, { y: -5 })
.from(mesh.scale, { x: 0, y: 0, z: 0 }, '-=0.5');
```

---

## Camera Animation Paths

```javascript
// สร้าง cinematic camera path ด้วย CatmullRomCurve3 + GSAP

const path = new THREE.CatmullRomCurve3([
    new THREE.Vector3( 0, 5, 15),
    new THREE.Vector3( 8, 3, 10),
    new THREE.Vector3( 8, 1,  0),
    new THREE.Vector3( 0, 2, -5),
    new THREE.Vector3(-8, 4,  5),
    new THREE.Vector3( 0, 5, 15)   // กลับมาที่เดิม
], true);  // closed loop

// Visualize path (optional)
const pathGeo = new THREE.TubeGeometry(path, 100, 0.02, 8, true);
const pathMesh = new THREE.Mesh(pathGeo, new THREE.MeshBasicMaterial({ color: 0xffaa00, wireframe: true }));
// scene.add(pathMesh);

// Animate camera along path
const progress = { t: 0 };
const pos = new THREE.Vector3();
const lookAhead = new THREE.Vector3();

gsap.to(progress, {
    t: 1,
    duration: 8,
    ease: 'none',
    repeat: -1,
    onUpdate() {
        path.getPoint(progress.t, pos);
        camera.position.copy(pos);
        
        // Look ahead along path
        path.getPoint((progress.t + 0.01) % 1, lookAhead);
        camera.lookAt(lookAhead);
    }
});
```

---

## Object Reveal Animations

```javascript
// Reveal from below (common intro animation)
function revealFromBelow(meshArray, delay = 0) {
    // ตั้งตำแหน่งเริ่มต้น
    meshArray.forEach(mesh => {
        mesh.position.y -= 5;
        mesh.material.opacity = 0;
        mesh.material.transparent = true;
    });
    
    gsap.to(meshArray.map(m => m.position), {
        y: '+=5',
        duration: 1.2,
        stagger: 0.15,
        delay,
        ease: 'power3.out'
    });
    
    gsap.to(meshArray.map(m => m.material), {
        opacity: 1,
        duration: 0.8,
        stagger: 0.15,
        delay,
        ease: 'power2.out'
    });
}

// Explode out animation
function explodeOut(mesh, onComplete) {
    gsap.to(mesh.scale, {
        x: 3, y: 3, z: 3,
        duration: 0.5,
        ease: 'power2.in'
    });
    gsap.to(mesh.material, {
        opacity: 0,
        duration: 0.5,
        ease: 'power2.in',
        onComplete
    });
}

// Elastic pop in
function elasticPop(mesh, delay = 0) {
    mesh.scale.set(0, 0, 0);
    gsap.to(mesh.scale, {
        x: 1, y: 1, z: 1,
        duration: 1.5,
        delay,
        ease: 'elastic.out(1, 0.4)'
    });
}
```

---

## FLIP Technique ใน 3D

```javascript
// FLIP = First, Last, Invert, Play
// เทคนิคการทำ transition ที่ smooth ระหว่างสอง states

class FLIP3D {
    constructor(mesh) {
        this.mesh = mesh;
        this.savedState = null;
    }
    
    // บันทึก state ปัจจุบัน
    saveFirst() {
        this.savedState = {
            position: this.mesh.position.clone(),
            rotation: this.mesh.rotation.clone(),
            scale:    this.mesh.scale.clone()
        };
    }
    
    // เปลี่ยนไปยัง "last" state แล้ว animate กลับ
    playTo(targetPos, targetRot, targetScale, duration = 1) {
        if (!this.savedState) return;
        
        // จำ first state
        const first = this.savedState;
        
        // ตั้ง position เป็น "last" state ทันที
        this.mesh.position.copy(targetPos);
        this.mesh.rotation.copy(targetRot);
        this.mesh.scale.copy(targetScale);
        
        // "Invert" = กลับไป first ทันที
        this.mesh.position.copy(first.position);
        this.mesh.rotation.copy(first.rotation);
        this.mesh.scale.copy(first.scale);
        
        // "Play" = animate ไปยัง last
        gsap.to(this.mesh.position, {
            x: targetPos.x, y: targetPos.y, z: targetPos.z,
            duration, ease: 'power3.inOut'
        });
        gsap.to(this.mesh.rotation, {
            x: targetRot.x, y: targetRot.y, z: targetRot.z,
            duration, ease: 'power3.inOut'
        });
        gsap.to(this.mesh.scale, {
            x: targetScale.x, y: targetScale.y, z: targetScale.z,
            duration, ease: 'power3.inOut'
        });
    }
}
```

---

## Cinematic Intro Animation

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Cinematic Intro</title>
    <style>
        * { margin: 0; padding: 0; }
        body { background: #000; overflow: hidden; }
        canvas { display: block; }
        
        #overlay {
            position: fixed;
            inset: 0;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            z-index: 10;
            pointer-events: none;
        }
        
        #title {
            font-family: 'Helvetica Neue', sans-serif;
            font-size: clamp(3rem, 8vw, 8rem);
            font-weight: 900;
            color: white;
            letter-spacing: 0.05em;
            opacity: 0;
            transform: translateY(50px);
        }
        
        #subtitle {
            font-family: 'Helvetica Neue', sans-serif;
            font-size: clamp(1rem, 2vw, 1.5rem);
            color: rgba(255,255,255,0.6);
            letter-spacing: 0.3em;
            text-transform: uppercase;
            margin-top: 1rem;
            opacity: 0;
        }
        
        #vignette {
            position: fixed;
            inset: 0;
            background: radial-gradient(ellipse at center, transparent 40%, rgba(0,0,0,0.8) 100%);
            pointer-events: none;
            z-index: 5;
        }
        
        #bars {
            position: fixed;
            inset: 0;
            pointer-events: none;
            z-index: 8;
        }
        #bar-top {
            position: absolute;
            top: 0; left: 0; right: 0;
            height: 12vh;
            background: #000;
        }
        #bar-bottom {
            position: absolute;
            bottom: 0; left: 0; right: 0;
            height: 12vh;
            background: #000;
        }
        
        #play-btn {
            position: fixed;
            bottom: 30px;
            right: 30px;
            padding: 12px 24px;
            background: rgba(255,255,255,0.1);
            border: 1px solid rgba(255,255,255,0.3);
            color: white;
            font-family: sans-serif;
            border-radius: 4px;
            cursor: pointer;
            z-index: 20;
        }
        #play-btn:hover { background: rgba(255,255,255,0.2); }
    </style>
</head>
<body>
<div id="bars">
    <div id="bar-top"></div>
    <div id="bar-bottom"></div>
</div>
<div id="vignette"></div>
<div id="overlay">
    <div id="title">UNIVERSE</div>
    <div id="subtitle">A Three.js Experience</div>
</div>
<button id="play-btn">▶ Replay</button>

<script src="https://cdn.jsdelivr.net/npm/gsap@3.12.2/dist/gsap.min.js"></script>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';

// === Three.js ===
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 1000);
camera.position.set(0, 0, 50);

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: false });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
document.body.insertBefore(renderer.domElement, document.body.firstChild);
renderer.domElement.style.position = 'fixed';
renderer.domElement.style.top = '0';
renderer.domElement.style.left = '0';

scene.background = new THREE.Color(0x000005);
scene.fog = new THREE.FogExp2(0x000005, 0.015);

// Stars
const starCount = 5000;
const starGeo = new THREE.BufferGeometry();
const starPos = new Float32Array(starCount * 3);
for (let i = 0; i < starCount * 3; i++) {
    starPos[i] = (Math.random() - 0.5) * 200;
}
starGeo.setAttribute('position', new THREE.BufferAttribute(starPos, 3));
const stars = new THREE.Points(starGeo,
    new THREE.PointsMaterial({ color: 0xffffff, size: 0.15, sizeAttenuation: true })
);
scene.add(stars);

// Central object
const mainObj = new THREE.Mesh(
    new THREE.IcosahedronGeometry(3, 4),
    new THREE.MeshStandardMaterial({
        color: 0x2244aa,
        roughness: 0.1, metalness: 0.9,
        emissive: 0x001133, emissiveIntensity: 0.5
    })
);
scene.add(mainObj);

// Rings
const rings = [];
for (let i = 0; i < 3; i++) {
    const r = new THREE.Mesh(
        new THREE.TorusGeometry(5 + i * 2, 0.05, 8, 80),
        new THREE.MeshBasicMaterial({ color: new THREE.Color().setHSL(i * 0.15 + 0.6, 0.8, 0.7) })
    );
    r.rotation.x = Math.PI / 2 + i * 0.2;
    r.rotation.y = i * 0.4;
    scene.add(r);
    rings.push(r);
}

// Lights
scene.add(new THREE.AmbientLight(0x112244, 1));
const blueLight = new THREE.PointLight(0x4488ff, 3, 30);
blueLight.position.set(0, 5, 5);
scene.add(blueLight);
const purpleLight = new THREE.PointLight(0x8844ff, 2, 25);
purpleLight.position.set(-5, -5, 5);
scene.add(purpleLight);

// === Cinematic Sequence ===
function playIntro() {
    const tl = gsap.timeline();
    
    // Start: bars covering top/bottom (cinematic letterbox)
    gsap.set('#bar-top',    { height: '12vh' });
    gsap.set('#bar-bottom', { height: '12vh' });
    gsap.set('#title',    { opacity: 0, y: 50 });
    gsap.set('#subtitle', { opacity: 0, y: 20 });
    
    // Camera starts far back
    camera.position.set(0, 0, 80);
    mainObj.scale.set(0, 0, 0);
    rings.forEach(r => { r.scale.set(0, 0, 0); });
    
    // 0.0s - Fade in bars (cinematic open)
    tl.to(['#bar-top', '#bar-bottom'], {
        height: '8vh',
        duration: 1.5,
        ease: 'power2.out'
    });
    
    // 0.5s - Camera flies in
    tl.to(camera.position, {
        z: 18,
        duration: 3.5,
        ease: 'power3.inOut'
    }, '-=1');
    
    // 1.5s - Main object appears
    tl.to(mainObj.scale, {
        x: 1, y: 1, z: 1,
        duration: 2,
        ease: 'elastic.out(1, 0.5)'
    }, '-=2.5');
    
    // 2.5s - Rings appear
    tl.to(rings.map(r => r.scale), {
        x: 1, y: 1, z: 1,
        duration: 1.5,
        stagger: 0.2,
        ease: 'back.out(1.5)'
    }, '-=1.5');
    
    // 4.0s - Title appears
    tl.to('#title', {
        opacity: 1,
        y: 0,
        duration: 1.2,
        ease: 'power3.out'
    }, '-=0.5');
    
    // 4.5s - Subtitle fades in
    tl.to('#subtitle', {
        opacity: 1,
        y: 0,
        duration: 1,
        ease: 'power2.out'
    }, '-=0.8');
    
    // 5.5s - Cinematic bars open fully
    tl.to(['#bar-top', '#bar-bottom'], {
        height: '0vh',
        duration: 1,
        ease: 'power2.out'
    }, '-=0.3');
    
    return tl;
}

playIntro();
document.getElementById('play-btn').onclick = () => playIntro();

// === Animation Loop ===
const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    const t = clock.getElapsedTime();
    
    mainObj.rotation.y = t * 0.2;
    mainObj.rotation.x = t * 0.1;
    
    rings[0].rotation.z = t * 0.3;
    rings[1].rotation.x = t * 0.25 + 0.5;
    rings[2].rotation.y = t * 0.2 + 1;
    
    stars.rotation.y = t * 0.01;
    blueLight.position.x   = Math.sin(t * 0.5) * 8;
    blueLight.position.y   = Math.cos(t * 0.3) * 5;
    purpleLight.position.x = Math.cos(t * 0.4) * -8;
    
    camera.lookAt(0, 0, 0);
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

## Product Reveal Animation

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Product Reveal</title>
    <style>
        * { margin: 0; padding: 0; }
        body { background: #111; overflow: hidden; font-family: 'Helvetica Neue', sans-serif; }
        canvas { position: fixed; top: 0; left: 0; z-index: 0; }
        
        #ui {
            position: fixed;
            inset: 0;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: flex-end;
            padding-bottom: 60px;
            z-index: 10;
            pointer-events: none;
        }
        
        #product-name {
            font-size: 3rem;
            font-weight: 900;
            color: white;
            letter-spacing: 0.1em;
            opacity: 0;
            transform: translateY(30px);
        }
        
        #product-tagline {
            font-size: 1rem;
            color: rgba(255,255,255,0.5);
            letter-spacing: 0.3em;
            text-transform: uppercase;
            margin-top: 8px;
            opacity: 0;
        }
        
        #cta {
            margin-top: 30px;
            padding: 14px 40px;
            background: transparent;
            border: 1.5px solid rgba(255,255,255,0.5);
            color: white;
            font-size: 0.9rem;
            letter-spacing: 0.2em;
            text-transform: uppercase;
            cursor: pointer;
            pointer-events: auto;
            opacity: 0;
            transition: background 0.3s, border-color 0.3s;
        }
        #cta:hover { background: rgba(255,255,255,0.1); border-color: white; }
        
        #color-picker {
            position: fixed;
            bottom: 20px;
            right: 20px;
            display: flex;
            gap: 10px;
            z-index: 10;
        }
        .color-dot {
            width: 24px;
            height: 24px;
            border-radius: 50%;
            cursor: pointer;
            border: 2px solid transparent;
            transition: border-color 0.2s, transform 0.2s;
        }
        .color-dot:hover, .color-dot.active { border-color: white; transform: scale(1.2); }
    </style>
</head>
<body>
<div id="ui">
    <div id="product-name">NEXUS X1</div>
    <div id="product-tagline">Reimagined · Perfected · Elevated</div>
    <button id="cta">Explore Now</button>
</div>
<div id="color-picker">
    <div class="color-dot active" style="background:#4488ff" data-color="#4488ff"></div>
    <div class="color-dot" style="background:#ff4466" data-color="#ff4466"></div>
    <div class="color-dot" style="background:#44ff88" data-color="#44ff88"></div>
    <div class="color-dot" style="background:#ffcc44" data-color="#ffcc44"></div>
    <div class="color-dot" style="background:#cccccc" data-color="#cccccc"></div>
</div>

<script src="https://cdn.jsdelivr.net/npm/gsap@3.12.2/dist/gsap.min.js"></script>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x0a0a0f);
const camera = new THREE.PerspectiveCamera(50, innerWidth / innerHeight, 0.1, 100);
camera.position.set(0, 0, 6);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.2;
document.body.appendChild(renderer.domElement);

// Lights
scene.add(new THREE.AmbientLight(0xffffff, 0.3));
const keyLight = new THREE.DirectionalLight(0xffffff, 2);
keyLight.position.set(5, 5, 5);
scene.add(keyLight);
const fillLight = new THREE.DirectionalLight(0x8844ff, 1);
fillLight.position.set(-5, 0, 2);
scene.add(fillLight);
const rimLight = new THREE.DirectionalLight(0x4488ff, 1.5);
rimLight.position.set(0, -5, -5);
scene.add(rimLight);

// Product object (complex geometry)
const product = new THREE.Group();
scene.add(product);

// Main body
const bodyMat = new THREE.MeshStandardMaterial({
    color: 0x4488ff,
    roughness: 0.1,
    metalness: 0.9
});
const bodyMesh = new THREE.Mesh(
    new THREE.CylinderGeometry(0.8, 1.0, 2.2, 32, 1, false),
    bodyMat
);
product.add(bodyMesh);

// Top cap
product.add(Object.assign(new THREE.Mesh(
    new THREE.SphereGeometry(0.8, 32, 16, 0, Math.PI * 2, 0, Math.PI / 2),
    bodyMat.clone()
), { position: new THREE.Vector3(0, 1.1, 0) }));

// Bottom cap
const bottomCap = new THREE.Mesh(
    new THREE.CylinderGeometry(1.0, 1.0, 0.1, 32),
    new THREE.MeshStandardMaterial({ color: 0x222222, roughness: 0.5, metalness: 0.8 })
);
bottomCap.position.y = -1.15;
product.add(bottomCap);

// Accent ring
const accentRing = new THREE.Mesh(
    new THREE.TorusGeometry(0.85, 0.04, 8, 32),
    new THREE.MeshStandardMaterial({ color: 0xffffff, roughness: 0.2, metalness: 1 })
);
accentRing.position.y = 0;
accentRing.rotation.x = Math.PI / 2;
product.add(accentRing);

// Start hidden
product.scale.set(0, 0, 0);
product.rotation.y = -Math.PI;

// === Reveal Animation ===
const tl = gsap.timeline({ delay: 0.5 });

// Object reveal
tl.to(product.scale, {
    x: 1, y: 1, z: 1,
    duration: 1.5,
    ease: 'elastic.out(1, 0.5)'
}, 0);

tl.to(product.rotation, {
    y: 0,
    duration: 2,
    ease: 'power3.out'
}, 0);

// Camera zoom in
tl.to(camera.position, {
    z: 5,
    duration: 2,
    ease: 'power2.inOut'
}, 0.3);

// Text reveals
tl.to('#product-name', {
    opacity: 1, y: 0,
    duration: 0.8,
    ease: 'power3.out'
}, 1.2);

tl.to('#product-tagline', {
    opacity: 1,
    duration: 0.6,
    ease: 'power2.out'
}, 1.6);

tl.to('#cta', {
    opacity: 1,
    duration: 0.5,
    ease: 'power2.out'
}, 2.0);

// Color picker
document.querySelectorAll('.color-dot').forEach(dot => {
    dot.addEventListener('click', () => {
        document.querySelectorAll('.color-dot').forEach(d => d.classList.remove('active'));
        dot.classList.add('active');
        const newColor = new THREE.Color(dot.dataset.color);
        
        gsap.to({}, {
            duration: 0.8,
            onUpdate() {
                const t = this.progress();
                bodyMat.color.lerpColors(bodyMat.color.clone(), newColor, t);
                product.children.forEach(c => {
                    if (c.material === bodyMat) return;
                    if (c.material?.color && c !== bottomCap && c !== accentRing) {
                        c.material.color.lerpColors(c.material.color.clone(), newColor, t);
                    }
                });
            }
        });
    });
});

// Auto-rotate
let userInteracting = false;
renderer.domElement.addEventListener('pointerdown', () => { userInteracting = true; });
window.addEventListener('pointerup', () => { userInteracting = false; });

let autoRotateSpeed = 0.3;
let dragStart = null;

renderer.domElement.addEventListener('pointerdown', e => {
    dragStart = { x: e.clientX, rotY: product.rotation.y };
});
renderer.domElement.addEventListener('pointermove', e => {
    if (dragStart) {
        const dx = (e.clientX - dragStart.x) * 0.01;
        product.rotation.y = dragStart.rotY + dx;
    }
});
renderer.domElement.addEventListener('pointerup', () => { dragStart = null; });

const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    const t = clock.getElapsedTime();
    
    if (!dragStart) {
        product.rotation.y += 0.003;
    }
    
    product.position.y = Math.sin(t * 0.8) * 0.1;
    accentRing.material.emissive.setHSL((t * 0.1) % 1, 0.5, 0.3);
    
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

## Choreographed Scene Transitions

```javascript
// ระบบ Scene Transition แบบ choreographed

class SceneTransitionManager {
    constructor(renderer, camera) {
        this.renderer = renderer;
        this.camera = camera;
        this.currentScene = null;
        this.scenes = new Map();
        this.isTransitioning = false;
    }
    
    addScene(name, scene, setupFn) {
        this.scenes.set(name, { scene, setup: setupFn, initialized: false });
    }
    
    async transitionTo(name, transitionType = 'fade') {
        if (this.isTransitioning) return;
        this.isTransitioning = true;
        
        const targetData = this.scenes.get(name);
        if (!targetData) { this.isTransitioning = false; return; }
        
        // Initialize on first visit
        if (!targetData.initialized) {
            targetData.setup(targetData.scene);
            targetData.initialized = true;
        }
        
        switch (transitionType) {
            case 'fade':
                await this._fadeTransition(targetData.scene);
                break;
            case 'zoom':
                await this._zoomTransition(targetData.scene);
                break;
            case 'slide':
                await this._slideTransition(targetData.scene);
                break;
        }
        
        this.currentScene = targetData.scene;
        this.isTransitioning = false;
    }
    
    _fadeTransition(nextScene) {
        return new Promise(resolve => {
            // Fade overlay in
            const overlay = { opacity: 0 };
            
            gsap.to(overlay, {
                opacity: 1,
                duration: 0.5,
                onUpdate: () => {
                    // Draw dark overlay on canvas
                    const ctx = overlayCanvas.getContext('2d');
                    ctx.fillStyle = `rgba(0,0,0,${overlay.opacity})`;
                    ctx.fillRect(0, 0, overlayCanvas.width, overlayCanvas.height);
                },
                onComplete: () => {
                    // Switch scene
                    this.renderer.render(nextScene, this.camera);
                    
                    // Fade overlay out
                    gsap.to(overlay, {
                        opacity: 0,
                        duration: 0.5,
                        onUpdate: () => {
                            const ctx = overlayCanvas.getContext('2d');
                            ctx.clearRect(0, 0, overlayCanvas.width, overlayCanvas.height);
                            ctx.fillStyle = `rgba(0,0,0,${overlay.opacity})`;
                            ctx.fillRect(0, 0, overlayCanvas.width, overlayCanvas.height);
                        },
                        onComplete: resolve
                    });
                }
            });
        });
    }
    
    _zoomTransition(nextScene) {
        return new Promise(resolve => {
            const savedFov = this.camera.fov;
            gsap.to(this.camera, {
                fov: 120,
                duration: 0.4,
                ease: 'power2.in',
                onUpdate: () => this.camera.updateProjectionMatrix(),
                onComplete: () => {
                    // Switch
                    this.currentScene = nextScene;
                    
                    gsap.to(this.camera, {
                        fov: savedFov,
                        duration: 0.6,
                        ease: 'power3.out',
                        onUpdate: () => this.camera.updateProjectionMatrix(),
                        onComplete: resolve
                    });
                }
            });
        });
    }
}
```

---

## Interactive Timeline

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Interactive Timeline</title>
    <style>
        * { margin: 0; padding: 0; }
        body { background: #0a0a12; font-family: sans-serif; overflow: hidden; }
        canvas { position: fixed; top: 0; left: 0; }
        
        #controls {
            position: fixed;
            bottom: 30px;
            left: 50%;
            transform: translateX(-50%);
            display: flex;
            align-items: center;
            gap: 15px;
            background: rgba(0,0,0,0.6);
            padding: 15px 25px;
            border-radius: 50px;
            z-index: 10;
        }
        
        button {
            width: 40px;
            height: 40px;
            background: rgba(255,255,255,0.1);
            border: 1px solid rgba(255,255,255,0.3);
            color: white;
            border-radius: 50%;
            cursor: pointer;
            font-size: 1rem;
            display: flex;
            align-items: center;
            justify-content: center;
        }
        button:hover { background: rgba(255,255,255,0.2); }
        
        #scrubber {
            width: 300px;
            appearance: none;
            height: 4px;
            background: rgba(255,255,255,0.2);
            border-radius: 2px;
            outline: none;
            cursor: pointer;
        }
        #scrubber::-webkit-slider-thumb {
            appearance: none;
            width: 16px;
            height: 16px;
            background: white;
            border-radius: 50%;
        }
        
        #time-label {
            color: rgba(255,255,255,0.6);
            font-size: 0.9rem;
            min-width: 50px;
            text-align: center;
        }
        
        #chapter-label {
            position: fixed;
            top: 25px;
            left: 50%;
            transform: translateX(-50%);
            color: white;
            font-size: 1.5rem;
            font-weight: 700;
            letter-spacing: 0.1em;
            z-index: 10;
        }
    </style>
</head>
<body>
<div id="chapter-label">Chapter 1: Birth</div>
<div id="controls">
    <button id="rewind">⏮</button>
    <button id="playPause">⏸</button>
    <button id="fastForward">⏭</button>
    <input type="range" id="scrubber" min="0" max="100" value="0" step="0.1">
    <div id="time-label">0.0s</div>
</div>

<script src="https://cdn.jsdelivr.net/npm/gsap@3.12.2/dist/gsap.min.js"></script>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x060610);
const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 100);
camera.position.set(0, 0, 12);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
document.body.appendChild(renderer.domElement);

scene.add(new THREE.AmbientLight(0x334466, 1));
const pLight = new THREE.PointLight(0x4488ff, 3, 20);
scene.add(pLight);
const pLight2 = new THREE.PointLight(0xff4488, 2, 20);
pLight2.position.set(-5, 5, 0);
scene.add(pLight2);

// Objects
const sphere = new THREE.Mesh(
    new THREE.SphereGeometry(1.5, 32, 32),
    new THREE.MeshStandardMaterial({ color: 0x4488ff, roughness: 0.2, metalness: 0.8 })
);
scene.add(sphere);

const ring = new THREE.Mesh(
    new THREE.TorusGeometry(3, 0.05, 8, 60),
    new THREE.MeshBasicMaterial({ color: 0x88aaff, transparent: true, opacity: 0 })
);
ring.rotation.x = Math.PI / 2;
scene.add(ring);

const particles = [];
const pGeo = new THREE.BufferGeometry();
const pPos = new Float32Array(500 * 3);
for (let i = 0; i < 500 * 3; i++) pPos[i] = (Math.random() - 0.5) * 20;
pGeo.setAttribute('position', new THREE.BufferAttribute(pPos, 3));
const particleSystem = new THREE.Points(pGeo, new THREE.PointsMaterial({ color: 0xffffff, size: 0.05, transparent: true, opacity: 0 }));
scene.add(particleSystem);

// === Master Timeline ===
const masterTl = gsap.timeline({ paused: true });
const DURATION = 10;

const chapters = [
    { time: 0, name: 'Chapter 1: Birth' },
    { time: 3, name: 'Chapter 2: Growth' },
    { time: 6, name: 'Chapter 3: Expansion' },
    { time: 9, name: 'Chapter 4: Transcendence' }
];

// Chapter 1: Birth (0 → 3s)
masterTl
    .from(sphere.scale, { x: 0, y: 0, z: 0, duration: 1.5, ease: 'elastic.out(1, 0.5)' }, 0)
    .to(sphere.material, { emissiveIntensity: 0.5, duration: 1.5 }, 0)

// Chapter 2: Growth (3 → 6s)
    .to(sphere.scale, { x: 1.5, y: 1.5, z: 1.5, duration: 1, ease: 'power2.out' }, 3)
    .to(ring.material, { opacity: 1, duration: 1 }, 3)
    .to(ring.scale, { x: 1.5, y: 1.5, z: 1.5, duration: 1.5, ease: 'power2.out' }, 3.2)

// Chapter 3: Expansion (6 → 9s)
    .to(camera.position, { z: 18, duration: 2, ease: 'power2.inOut' }, 6)
    .to(particleSystem.material, { opacity: 0.6, duration: 1 }, 6)
    .to(sphere.rotation, { y: Math.PI * 4, duration: 3, ease: 'none' }, 6)

// Chapter 4: Transcendence (9 → 10s)
    .to(sphere.material, { opacity: 0, duration: 0.5 }, 9.2)
    .to(ring.material, { opacity: 0, duration: 0.5 }, 9.2)
    .to(particleSystem.material, { opacity: 0, duration: 0.8 }, 9)
    .to(camera.position, { z: 0, duration: 0.8, ease: 'power3.in' }, 9.2);

// === Controls ===
let isPlaying = true;
masterTl.play();

function updateChapterLabel(time) {
    let current = chapters[0];
    for (const ch of chapters) {
        if (time >= ch.time) current = ch;
    }
    document.getElementById('chapter-label').textContent = current.name;
}

document.getElementById('playPause').addEventListener('click', () => {
    if (isPlaying) {
        masterTl.pause();
        document.getElementById('playPause').textContent = '▶';
    } else {
        masterTl.resume();
        document.getElementById('playPause').textContent = '⏸';
    }
    isPlaying = !isPlaying;
});

document.getElementById('rewind').onclick = () => { masterTl.seek(0); masterTl.play(); };
document.getElementById('fastForward').onclick = () => { masterTl.seek(DURATION); };

const scrubber = document.getElementById('scrubber');
scrubber.addEventListener('input', () => {
    masterTl.seek(parseFloat(scrubber.value) / 100 * DURATION);
});

// Render loop
function animate() {
    requestAnimationFrame(animate);
    
    // Update scrubber
    const t = masterTl.time();
    scrubber.value = (t / DURATION) * 100;
    document.getElementById('time-label').textContent = t.toFixed(1) + 's';
    updateChapterLabel(t);
    
    // Ambient animation
    pLight.position.set(
        Math.sin(t * 0.5) * 5,
        Math.cos(t * 0.3) * 3,
        3
    );
    particleSystem.rotation.y = t * 0.05;
    
    camera.lookAt(0, 0, 0);
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
1. สร้าง GSAP timeline ที่ animate 3 objects ทีละชิ้น
2. ใช้ `gsap.to` กับ ScrollTrigger สำหรับ scroll animation
3. ทดลอง easing types ต่างๆ: elastic, bounce, back, expo

### ระดับ 2
4. Stagger animation สำหรับ particle grid
5. Camera fly-through path ด้วย GSAP
6. Product reveal พร้อม color switcher

### ระดับ 3
7. Cinematic scene intro พร้อม letterbox bars
8. Scene transition system (fade, zoom, slide)
9. Interactive timeline พร้อม scrubber

### Challenge
10. สร้าง Full Animated Landing Page:
    - Cinematic intro (5-10 วินาที)
    - Scroll-triggered sections (4 sections)
    - Product showcase section (drag to rotate)
    - About section พร้อม skill bars animated
    - Contact section พร้อม floating particles
    - Smooth transitions ทุก interaction
    - Mobile-responsive layout
    - Performance: 60fps บน mid-range phone
