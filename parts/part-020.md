# Part 20: Particles Part 1 - Points System ใน Three.js

## บทนำ

Particles หรือระบบอนุภาค เป็นเทคนิคสำคัญในการสร้างเอฟเฟกต์สวยงาม เช่น ดาว, หิมะ, ฝน, ควัน, กาแล็กซี่ Three.js ใช้ `Points` class สำหรับ render particles จำนวนมากในครั้งเดียว

## 1. Points Class พื้นฐาน

```javascript
// Points ต้องการ:
// 1. BufferGeometry ที่มี position attribute
// 2. PointsMaterial

// สร้าง geometry
const geometry = new THREE.BufferGeometry();
const positions = new Float32Array([
    0, 0, 0,   // particle 1
    1, 0, 0,   // particle 2
    0, 1, 0,   // particle 3
]);
geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));

// สร้าง material
const material = new THREE.PointsMaterial({
    size: 0.05,         // ขนาด particle
    color: 0xffffff     // สี
});

// สร้าง Points object
const points = new THREE.Points(geometry, material);
scene.add(points);
```

---

## 2. PointsMaterial Properties

```javascript
const material = new THREE.PointsMaterial({
    // ขนาด
    size: 0.05,                    // ขนาด pixel ของแต่ละ particle
    sizeAttenuation: true,         // ยิ่งไกลยิ่งเล็ก (perspective)
                                   // false = ขนาดคงที่ไม่ว่าจะไกลแค่ไหน
    
    // สี
    color: 0xffffff,               // สี (ถ้าใช้ vertexColors จะถูก override)
    vertexColors: false,           // ใช้ colors attribute จาก geometry
    
    // Transparency
    transparent: true,             // ต้องเปิดสำหรับ opacity และ alphaMap
    opacity: 0.8,                  // ความโปร่งใส
    
    // Texture
    map: particleTexture,          // texture ของ particle
    alphaMap: alphaTexture,        // ใช้ grayscale texture เป็น alpha mask
    alphaTest: 0.001,              // cutoff alpha (ป้องกัน sorting issues)
    
    // Blending
    blending: THREE.AdditiveBlending,  // additive = สว่างขึ้นเมื่อซ้อนกัน
    // THREE.NormalBlending (default)
    // THREE.AdditiveBlending (สำหรับ fire, magic)
    // THREE.SubtractiveBlending
    // THREE.MultiplyBlending
    
    // Depth
    depthWrite: false,             // ปิดเพื่อหลีกเลี่ยง depth sorting issues
    depthTest: true                // เปิด depth test (default)
});
```

---

## 3. BufferGeometry สำหรับ Particles

### Random Particles จำนวนมาก

```javascript
function createParticleSystem(count, spread = 10) {
    const geometry = new THREE.BufferGeometry();
    
    // สร้าง Float32Array สำหรับ positions
    const positions = new Float32Array(count * 3); // x, y, z สำหรับแต่ละ particle
    const colors = new Float32Array(count * 3);    // r, g, b สำหรับแต่ละ particle
    
    const color = new THREE.Color();
    
    for (let i = 0; i < count; i++) {
        const i3 = i * 3;
        
        // Random position ใน sphere
        positions[i3] = (Math.random() - 0.5) * spread;
        positions[i3 + 1] = (Math.random() - 0.5) * spread;
        positions[i3 + 2] = (Math.random() - 0.5) * spread;
        
        // Random color
        color.setHSL(Math.random(), 0.8, 0.6);
        colors[i3] = color.r;
        colors[i3 + 1] = color.g;
        colors[i3 + 2] = color.b;
    }
    
    // กำหนด attributes
    geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));
    
    const material = new THREE.PointsMaterial({
        size: 0.1,
        vertexColors: true,   // ใช้ color จาก attribute
        sizeAttenuation: true,
        transparent: true,
        opacity: 0.8,
        blending: THREE.AdditiveBlending,
        depthWrite: false
    });
    
    return new THREE.Points(geometry, material);
}

const particles = createParticleSystem(10000, 20);
scene.add(particles);
```

---

## 4. Particle Texture

```javascript
// สร้าง particle texture ด้วย Canvas
function createCircleTexture(size = 64, color = 'white', blurAmount = 0.3) {
    const canvas = document.createElement('canvas');
    canvas.width = canvas.height = size;
    const ctx = canvas.getContext('2d');
    
    const center = size / 2;
    const radius = size / 2;
    
    // Radial gradient
    const gradient = ctx.createRadialGradient(center, center, 0, center, center, radius);
    gradient.addColorStop(0, `${color}`);
    gradient.addColorStop(blurAmount, `${color}`);
    gradient.addColorStop(1, 'rgba(0,0,0,0)');
    
    ctx.fillStyle = gradient;
    ctx.fillRect(0, 0, size, size);
    
    const texture = new THREE.CanvasTexture(canvas);
    return texture;
}

// Star texture
function createStarTexture() {
    const canvas = document.createElement('canvas');
    canvas.width = canvas.height = 64;
    const ctx = canvas.getContext('2d');
    
    ctx.fillStyle = 'black';
    ctx.fillRect(0, 0, 64, 64);
    
    ctx.beginPath();
    const points = 5;
    const outerR = 28;
    const innerR = 12;
    
    for (let i = 0; i < points * 2; i++) {
        const angle = (i * Math.PI) / points - Math.PI / 2;
        const r = i % 2 === 0 ? outerR : innerR;
        const x = 32 + Math.cos(angle) * r;
        const y = 32 + Math.sin(angle) * r;
        i === 0 ? ctx.moveTo(x, y) : ctx.lineTo(x, y);
    }
    
    ctx.closePath();
    
    // Glow
    ctx.shadowBlur = 15;
    ctx.shadowColor = 'rgba(255, 255, 200, 1)';
    ctx.fillStyle = 'rgba(255, 255, 200, 1)';
    ctx.fill();
    
    return new THREE.CanvasTexture(canvas);
}

// ใช้ texture
const particleTexture = createCircleTexture(64, 'white', 0.3);
const material = new THREE.PointsMaterial({
    size: 0.2,
    map: particleTexture,
    transparent: true,
    alphaTest: 0.001,
    depthWrite: false,
    blending: THREE.AdditiveBlending
});
```

---

## 5. Starfield Demo สมบูรณ์

```html
<!DOCTYPE html>
<html>
<head>
    <title>Starfield Demo</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; }
        
        #controls {
            position: absolute;
            top: 10px;
            right: 10px;
            background: rgba(0,0,0,0.8);
            color: white;
            padding: 15px;
            border-radius: 8px;
            font-family: monospace;
            font-size: 12px;
            min-width: 220px;
        }
        h3 { color: #e94560; margin-bottom: 10px; }
        h4 { color: #4ecdc4; margin: 8px 0 4px; }
        .row { display: flex; justify-content: space-between; align-items: center; margin: 4px 0; }
        input[type=range] { width: 100px; }
        .val { color: #ffd700; width: 45px; text-align: right; }
    </style>
</head>
<body>
<div id="controls">
    <h3>Starfield</h3>
    <h4>Stars</h4>
    <div class="row">
        <label>Size:</label>
        <input type="range" id="starSize" min="0.01" max="0.5" step="0.01" value="0.1">
        <span class="val" id="starSizeVal">0.1</span>
    </div>
    <div class="row">
        <label>Opacity:</label>
        <input type="range" id="starOpacity" min="0" max="1" step="0.05" value="0.8">
        <span class="val" id="starOpacityVal">0.8</span>
    </div>
    <div class="row">
        <label>Speed:</label>
        <input type="range" id="warpSpeed" min="0" max="5" step="0.1" value="0">
        <span class="val" id="warpSpeedVal">0</span>
    </div>
    
    <h4>Colors</h4>
    <div class="row">
        <label>Colorful:</label>
        <input type="checkbox" id="colorful" checked>
    </div>
    
    <h4>Layers</h4>
    <div class="row">
        <label>Near Stars:</label>
        <input type="checkbox" id="nearStars" checked>
    </div>
    <div class="row">
        <label>Far Stars:</label>
        <input type="checkbox" id="farStars" checked>
    </div>
    <div class="row">
        <label>Nebula:</label>
        <input type="checkbox" id="nebula" checked>
    </div>
</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';

// Scene
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 2000);
camera.position.z = 5;

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
document.body.appendChild(renderer.domElement);

// สร้าง particle texture
function createGlowTexture() {
    const canvas = document.createElement('canvas');
    canvas.width = canvas.height = 64;
    const ctx = canvas.getContext('2d');
    
    const gradient = ctx.createRadialGradient(32, 32, 0, 32, 32, 32);
    gradient.addColorStop(0, 'rgba(255, 255, 255, 1)');
    gradient.addColorStop(0.1, 'rgba(255, 255, 255, 0.8)');
    gradient.addColorStop(0.4, 'rgba(255, 255, 255, 0.3)');
    gradient.addColorStop(1, 'rgba(0, 0, 0, 0)');
    
    ctx.fillStyle = gradient;
    ctx.fillRect(0, 0, 64, 64);
    
    return new THREE.CanvasTexture(canvas);
}

const starTexture = createGlowTexture();

// สร้าง Stars layers
function createStarLayer(count, spread, minSize, maxSize, colorful = true) {
    const geo = new THREE.BufferGeometry();
    const positions = new Float32Array(count * 3);
    const colors = new Float32Array(count * 3);
    const sizes = new Float32Array(count);
    
    const starColors = colorful ? [
        new THREE.Color(0xaabbff), // น้ำเงินเย็น
        new THREE.Color(0xffffff), // ขาว
        new THREE.Color(0xffddaa), // ส้มอุ่น
        new THREE.Color(0xffaaaa), // แดง
        new THREE.Color(0xaaffaa), // เขียว
    ] : [new THREE.Color(0xffffff)];
    
    for (let i = 0; i < count; i++) {
        const i3 = i * 3;
        
        // Spherical distribution
        const theta = Math.random() * Math.PI * 2;
        const phi = Math.acos(2 * Math.random() - 1);
        const radius = spread * (0.5 + Math.random() * 0.5);
        
        positions[i3] = radius * Math.sin(phi) * Math.cos(theta);
        positions[i3 + 1] = radius * Math.sin(phi) * Math.sin(theta);
        positions[i3 + 2] = radius * Math.cos(phi);
        
        // สุ่มสี
        const c = starColors[Math.floor(Math.random() * starColors.length)];
        colors[i3] = c.r;
        colors[i3 + 1] = c.g;
        colors[i3 + 2] = c.b;
        
        // สุ่มขนาด
        sizes[i] = minSize + Math.random() * (maxSize - minSize);
    }
    
    geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    geo.setAttribute('color', new THREE.BufferAttribute(colors, 3));
    geo.setAttribute('size', new THREE.BufferAttribute(sizes, 1));
    
    const mat = new THREE.PointsMaterial({
        size: 0.1,
        map: starTexture,
        vertexColors: colorful,
        transparent: true,
        opacity: 0.9,
        depthWrite: false,
        blending: THREE.AdditiveBlending,
        sizeAttenuation: true
    });
    
    return new THREE.Points(geo, mat);
}

// สร้างหลาย layers
const nearStars = createStarLayer(2000, 200, 0.08, 0.2, true);
const farStars = createStarLayer(8000, 1000, 0.02, 0.08, false);
scene.add(nearStars, farStars);

// Nebula (หมอกสีสัน)
function createNebula() {
    const geo = new THREE.BufferGeometry();
    const count = 3000;
    const positions = new Float32Array(count * 3);
    const colors = new Float32Array(count * 3);
    
    const nebulaColors = [
        new THREE.Color(0x4400aa),  // ม่วง
        new THREE.Color(0x002288),  // น้ำเงิน
        new THREE.Color(0xaa0044),  // แดง
        new THREE.Color(0x004422),  // เขียว
    ];
    
    for (let i = 0; i < count; i++) {
        const i3 = i * 3;
        const spread = 300;
        const x = (Math.random() - 0.5) * spread;
        const y = (Math.random() - 0.5) * spread * 0.3; // แบนในแนวนอน
        const z = (Math.random() - 0.5) * spread;
        
        // Cluster ไปที่ศูนย์กลาง
        const dist = Math.sqrt(x * x + y * y + z * z);
        const factor = Math.exp(-dist / 100);
        
        positions[i3] = x;
        positions[i3 + 1] = y;
        positions[i3 + 2] = z;
        
        const c = nebulaColors[Math.floor(Math.random() * nebulaColors.length)];
        colors[i3] = c.r;
        colors[i3 + 1] = c.g;
        colors[i3 + 2] = c.b;
    }
    
    geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    geo.setAttribute('color', new THREE.BufferAttribute(colors, 3));
    
    return new THREE.Points(geo, new THREE.PointsMaterial({
        size: 2,
        vertexColors: true,
        transparent: true,
        opacity: 0.15,
        depthWrite: false,
        blending: THREE.AdditiveBlending,
        sizeAttenuation: true
    }));
}

const nebula = createNebula();
scene.add(nebula);

// Controls
document.getElementById('starSize').addEventListener('input', (e) => {
    const v = parseFloat(e.target.value);
    nearStars.material.size = v;
    document.getElementById('starSizeVal').textContent = v.toFixed(2);
});
document.getElementById('starOpacity').addEventListener('input', (e) => {
    const v = parseFloat(e.target.value);
    nearStars.material.opacity = v;
    farStars.material.opacity = v * 0.6;
    document.getElementById('starOpacityVal').textContent = v.toFixed(2);
});
document.getElementById('warpSpeed').addEventListener('input', (e) => {
    document.getElementById('warpSpeedVal').textContent = parseFloat(e.target.value).toFixed(1);
});
document.getElementById('nearStars').addEventListener('change', (e) => {
    nearStars.visible = e.target.checked;
});
document.getElementById('farStars').addEventListener('change', (e) => {
    farStars.visible = e.target.checked;
});
document.getElementById('nebula').addEventListener('change', (e) => {
    nebula.visible = e.target.checked;
});
document.getElementById('colorful').addEventListener('change', (e) => {
    nearStars.material.vertexColors = e.target.checked;
    nearStars.material.needsUpdate = true;
});

// Resize
window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});

// Mouse control
let mouseX = 0, mouseY = 0;
document.addEventListener('mousemove', (e) => {
    mouseX = (e.clientX / window.innerWidth - 0.5) * 2;
    mouseY = -(e.clientY / window.innerHeight - 0.5) * 2;
});

// Animation
const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    const delta = clock.getDelta();
    const elapsed = clock.getElapsedTime();
    
    const warpSpeed = parseFloat(document.getElementById('warpSpeed').value);
    
    // Warp speed - เคลื่อนที่ไปข้างหน้า
    if (warpSpeed > 0) {
        camera.position.z -= warpSpeed * delta * 50;
        if (camera.position.z < -1000) camera.position.z = 5;
    }
    
    // หมุนช้าๆ ตาม mouse
    camera.rotation.y += (mouseX * 0.001 - camera.rotation.y) * 0.05;
    camera.rotation.x += (mouseY * 0.0005 - camera.rotation.x) * 0.05;
    
    // หมุน near stars ช้าๆ
    nearStars.rotation.y = elapsed * 0.01;
    nebula.rotation.y = elapsed * 0.005;
    
    renderer.render(scene, camera);
}
animate();
</script>
</body>
</html>
```

---

## 6. Galaxy Generator Demo

```javascript
import * as THREE from 'three';

class GalaxyGenerator {
    constructor(scene) {
        this.scene = scene;
        this.geometry = null;
        this.material = null;
        this.points = null;
        
        this.params = {
            count: 50000,
            size: 0.01,
            radius: 10,
            branches: 5,
            spin: 1,
            randomness: 0.3,
            randomnessPower: 3,
            insideColor: '#ff6030',
            outsideColor: '#1b3984'
        };
    }
    
    generate() {
        // Dispose เก่า
        if (this.points) {
            this.geometry.dispose();
            this.material.dispose();
            this.scene.remove(this.points);
        }
        
        this.geometry = new THREE.BufferGeometry();
        
        const positions = new Float32Array(this.params.count * 3);
        const colors = new Float32Array(this.params.count * 3);
        
        const colorInside = new THREE.Color(this.params.insideColor);
        const colorOutside = new THREE.Color(this.params.outsideColor);
        
        for (let i = 0; i < this.params.count; i++) {
            const i3 = i * 3;
            
            // ระยะจากศูนย์กลาง
            const radius = Math.random() * this.params.radius;
            
            // มุมของ branch
            const branchAngle = (i % this.params.branches) / this.params.branches * Math.PI * 2;
            
            // spin angle ตามระยะ
            const spinAngle = radius * this.params.spin;
            
            // Randomness
            const randomX = Math.pow(Math.random(), this.params.randomnessPower) * 
                            (Math.random() < 0.5 ? 1 : -1) * this.params.randomness * radius;
            const randomY = Math.pow(Math.random(), this.params.randomnessPower) * 
                            (Math.random() < 0.5 ? 1 : -1) * this.params.randomness * radius;
            const randomZ = Math.pow(Math.random(), this.params.randomnessPower) * 
                            (Math.random() < 0.5 ? 1 : -1) * this.params.randomness * radius;
            
            // Position บน arm ของ galaxy
            positions[i3] = Math.cos(branchAngle + spinAngle) * radius + randomX;
            positions[i3 + 1] = randomY;
            positions[i3 + 2] = Math.sin(branchAngle + spinAngle) * radius + randomZ;
            
            // Color interpolation จากใน → นอก
            const mixedColor = colorInside.clone();
            mixedColor.lerp(colorOutside, radius / this.params.radius);
            
            colors[i3] = mixedColor.r;
            colors[i3 + 1] = mixedColor.g;
            colors[i3 + 2] = mixedColor.b;
        }
        
        this.geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
        this.geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));
        
        this.material = new THREE.PointsMaterial({
            size: this.params.size,
            sizeAttenuation: true,
            depthWrite: false,
            blending: THREE.AdditiveBlending,
            vertexColors: true
        });
        
        this.points = new THREE.Points(this.geometry, this.material);
        this.scene.add(this.points);
        
        return this.points;
    }
    
    update(params) {
        Object.assign(this.params, params);
        this.generate();
    }
    
    animate(elapsed) {
        if (this.points) {
            this.points.rotation.y = elapsed * 0.05;
        }
    }
}

// ใช้งาน
const galaxy = new GalaxyGenerator(scene);
galaxy.generate();

function animate() {
    const elapsed = clock.getElapsedTime();
    galaxy.animate(elapsed);
    renderer.render(scene, camera);
}
```

---

## 7. Particle Count และ Performance

```javascript
// Particle count guideline:
// Mobile:    1,000 - 10,000
// Desktop:  10,000 - 100,000
// High-end: 100,000 - 1,000,000+

// ทดสอบ performance
let fpsHistory = [];
let lastTime = performance.now();
let frameCount = 0;

function checkFPS() {
    const now = performance.now();
    const delta = now - lastTime;
    frameCount++;
    
    if (delta >= 1000) {
        const fps = Math.round(frameCount * 1000 / delta);
        fpsHistory.push(fps);
        if (fpsHistory.length > 5) fpsHistory.shift();
        const avgFPS = fpsHistory.reduce((a, b) => a + b, 0) / fpsHistory.length;
        
        // ปรับ particle count ตาม FPS
        if (avgFPS < 30) {
            reduceParticles();
        } else if (avgFPS > 55) {
            increaseParticles();
        }
        
        frameCount = 0;
        lastTime = now;
    }
}

// Dynamic particle count
function createAdaptiveParticleSystem(maxCount = 100000) {
    let currentCount = 10000;
    
    function rebuild(count) {
        // Re-create geometry with new count
        currentCount = count;
        // ...
    }
    
    return {
        reduce: () => rebuild(Math.max(1000, currentCount * 0.8)),
        increase: () => rebuild(Math.min(maxCount, currentCount * 1.2))
    };
}
```

---

## 8. Float32Array Tips

```javascript
// Float32Array เหมาะกับ WebGL เพราะ GPU ใช้ 32-bit float
// ประหยัด memory กว่า Float64Array (8 bytes vs 4 bytes)

const count = 100000;
const positions = new Float32Array(count * 3);

// กำหนดค่าแบบ loop
for (let i = 0; i < count; i++) {
    const i3 = i * 3;
    positions[i3 + 0] = Math.random() * 20 - 10; // x
    positions[i3 + 1] = Math.random() * 20 - 10; // y
    positions[i3 + 2] = Math.random() * 20 - 10; // z
}

// อัพเดตค่า (ระหว่าง runtime)
function updateParticles(elapsed) {
    for (let i = 0; i < count; i++) {
        const i3 = i * 3;
        positions[i3 + 1] = Math.sin(elapsed + i * 0.1) * 5; // animate Y
    }
    
    // สำคัญ! บอก three.js ว่า data เปลี่ยนแล้ว
    geometry.attributes.position.needsUpdate = true;
}

// อ่านค่า
const x = positions[i * 3 + 0];
const y = positions[i * 3 + 1];
const z = positions[i * 3 + 2];

// Memory calculation
// 100,000 particles × 3 components × 4 bytes = 1.2 MB for positions only
// + colors: 1.2 MB
// Total: ~2.4 MB per 100k particles
```

---

## 9. Custom Shader สำหรับ Particles

```javascript
// ใช้ ShaderMaterial สำหรับ custom particle effects
const geometry = new THREE.BufferGeometry();
// ... set positions, colors...

// Add custom attributes
const sizes = new Float32Array(count);
const life = new Float32Array(count);
for (let i = 0; i < count; i++) {
    sizes[i] = Math.random();
    life[i] = Math.random();
}
geometry.setAttribute('aSize', new THREE.BufferAttribute(sizes, 1));
geometry.setAttribute('aLife', new THREE.BufferAttribute(life, 1));

const shaderMaterial = new THREE.ShaderMaterial({
    uniforms: {
        time: { value: 0 },
        pointTexture: { value: particleTexture }
    },
    
    vertexShader: `
        attribute float aSize;
        attribute float aLife;
        
        uniform float time;
        
        varying float vLife;
        
        void main() {
            vLife = aLife;
            
            vec3 pos = position;
            pos.y += sin(time * 2.0 + aLife * 6.28) * 0.5;
            
            vec4 mvPosition = modelViewMatrix * vec4(pos, 1.0);
            gl_Position = projectionMatrix * mvPosition;
            
            // size attenuation
            gl_PointSize = aSize * 300.0 / -mvPosition.z;
        }
    `,
    
    fragmentShader: `
        uniform sampler2D pointTexture;
        varying float vLife;
        
        void main() {
            vec4 color = texture2D(pointTexture, gl_PointCoord);
            
            // Fade based on life
            color.a *= vLife;
            
            gl_FragColor = color;
        }
    `,
    
    transparent: true,
    blending: THREE.AdditiveBlending,
    depthWrite: false
});

const points = new THREE.Points(geometry, shaderMaterial);
scene.add(points);

// อัพเดตใน loop
shaderMaterial.uniforms.time.value = elapsed;
```

---

## แบบฝึกหัด

1. **Interactive Starfield**: คลิกเพื่อสร้างกลุ่มดาว
2. **Galaxy Parameter**: สร้าง GUI ปรับ galaxy parameters แบบ Real-time
3. **Particle Morphing**: Morph particles จากรูป sphere เป็น cube
4. **Rain/Snow**: สร้างฝนหรือหิมะที่ตกลงมาอย่างสมจริง
5. **Music Visualizer**: Particles ที่ตอบสนองต่อ audio data

---

## สรุป

- `Points` + `PointsMaterial` + `BufferGeometry` คือสูตรสำเร็จ
- `Float32Array` เป็น typed array ที่ GPU ชอบ
- `needsUpdate = true` ต้องเรียกเมื่ออัพเดต attribute
- `AdditiveBlending` + `depthWrite: false` ให้ glow effect ที่สวย
- `sizeAttenuation` ทำให้ particles ไกลดูเล็กลงตาม perspective
- `vertexColors: true` ให้แต่ละ particle มีสีของตัวเอง
- `alphaTest` หรือ `alphaMap` ลด aliasing ของ particle texture
- ตรวจ FPS และ limit particle count ให้เหมาะสมกับ device
