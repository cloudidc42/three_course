# Part 21: Particles Part 2 - Advanced ใน Three.js

## บทนำ

ในบทนี้เราจะต่อยอดจาก Part 20 ไปสู่เทคนิค particle ขั้นสูง ได้แก่ Sprite vs Points, particle lifecycle (birth/death), GPU instancing, และการสร้าง effects สวยงาม เช่น ไฟ, ควัน, และเวทมนตร์

## 1. Sprite vs Points

| Feature | Points | Sprite |
|---------|--------|--------|
| Performance | ดีกว่า (1 draw call) | หนักกว่า (หลาย draw calls) |
| ขนาด | uniform ต่อ material | แต่ละ sprite ต่างกันได้ |
| Rotation | ไม่ได้ | ได้ |
| Billboard | อัตโนมัติ | อัตโนมัติ |
| ใช้ทำ | ระบบ particles มากๆ | UI elements, icons |

### Sprite

```javascript
// Sprite - billboard ที่หันหาก camera เสมอ
const spriteMaterial = new THREE.SpriteMaterial({
    map: particleTexture,
    color: 0xffffff,
    transparent: true,
    opacity: 0.8,
    blending: THREE.AdditiveBlending,
    fog: true        // รับผลจาก fog
});

const sprite = new THREE.Sprite(spriteMaterial);
sprite.position.set(5, 2, 0);
sprite.scale.set(2, 2, 1);  // width, height (z ไม่มีผล)
scene.add(sprite);

// Sprite ไม่ต้องการ Billboard shader เพราะ Three.js จัดการให้
// แต่ถ้าต้องการ rotation ต้องใช้ rotation property
sprite.material.rotation = Math.PI / 4;  // หมุน 45°
```

---

## 2. Particle Lifecycle (Birth/Death)

```javascript
// Particle ที่มีชีวิตและตายได้
class ParticleData {
    constructor() {
        this.position = new THREE.Vector3();
        this.velocity = new THREE.Vector3();
        this.life = 0;        // อายุปัจจุบัน
        this.maxLife = 1;     // อายุสูงสุด
        this.alive = false;
        this.size = 1;
        this.color = new THREE.Color();
    }
    
    reset(origin) {
        this.position.copy(origin);
        this.velocity.set(
            (Math.random() - 0.5) * 2,
            Math.random() * 3 + 1,
            (Math.random() - 0.5) * 2
        );
        this.life = 0;
        this.maxLife = 0.5 + Math.random() * 1.5;
        this.alive = true;
        this.size = 0.5 + Math.random() * 1.5;
        this.color.setHSL(Math.random() * 0.1 + 0.05, 1, 0.6); // orange-red
    }
    
    update(delta) {
        if (!this.alive) return;
        
        this.life += delta;
        
        if (this.life >= this.maxLife) {
            this.alive = false;
            return;
        }
        
        // Physics
        this.velocity.y -= 5 * delta; // gravity
        this.velocity.multiplyScalar(1 - 0.5 * delta); // drag
        
        this.position.addScaledVector(this.velocity, delta);
    }
    
    get lifeRatio() {
        return this.life / this.maxLife;
    }
}

// Particle System ที่มี lifecycle
class ParticleSystem {
    constructor(maxParticles = 1000) {
        this.maxParticles = maxParticles;
        this.particles = Array.from({ length: maxParticles }, () => new ParticleData());
        this.aliveCount = 0;
        
        this.geometry = new THREE.BufferGeometry();
        this.positions = new Float32Array(maxParticles * 3);
        this.colors = new Float32Array(maxParticles * 3);
        this.sizes = new Float32Array(maxParticles);
        
        this.geometry.setAttribute('position', new THREE.BufferAttribute(this.positions, 3));
        this.geometry.setAttribute('color', new THREE.BufferAttribute(this.colors, 3));
        this.geometry.setAttribute('size', new THREE.BufferAttribute(this.sizes, 1));
        
        this.material = new THREE.PointsMaterial({
            size: 0.1,
            vertexColors: true,
            transparent: true,
            opacity: 0.8,
            blending: THREE.AdditiveBlending,
            depthWrite: false,
            sizeAttenuation: true
        });
        
        this.points = new THREE.Points(this.geometry, this.material);
        this.emitRate = 50;   // particles per second
        this.emitTimer = 0;
        this.origin = new THREE.Vector3();
    }
    
    emit(position) {
        // หา particle ที่ dead
        for (const particle of this.particles) {
            if (!particle.alive) {
                particle.reset(position || this.origin);
                return particle;
            }
        }
        return null; // เต็มแล้ว
    }
    
    update(delta) {
        // Auto emit
        this.emitTimer += delta;
        const emitCount = Math.floor(this.emitTimer * this.emitRate);
        this.emitTimer %= (1 / this.emitRate);
        
        for (let i = 0; i < emitCount; i++) {
            this.emit();
        }
        
        // อัพเดต particles
        this.aliveCount = 0;
        for (let i = 0; i < this.maxParticles; i++) {
            const p = this.particles[i];
            if (!p.alive) continue;
            
            p.update(delta);
            
            if (p.alive) {
                const ratio = p.lifeRatio;
                const i3 = this.aliveCount * 3;
                
                this.positions[i3] = p.position.x;
                this.positions[i3 + 1] = p.position.y;
                this.positions[i3 + 2] = p.position.z;
                
                // Fade out ตามอายุ
                const fadeColor = p.color.clone().multiplyScalar(1 - ratio);
                this.colors[i3] = fadeColor.r;
                this.colors[i3 + 1] = fadeColor.g;
                this.colors[i3 + 2] = fadeColor.b;
                
                this.sizes[this.aliveCount] = p.size * (1 - ratio * 0.5);
                
                this.aliveCount++;
            }
        }
        
        // อัพเดต draw range (เฉพาะ alive particles)
        this.geometry.setDrawRange(0, this.aliveCount);
        
        // notify GPU
        this.geometry.attributes.position.needsUpdate = true;
        this.geometry.attributes.color.needsUpdate = true;
        this.geometry.attributes.size.needsUpdate = true;
    }
    
    addToScene(scene) {
        scene.add(this.points);
    }
}
```

---

## 3. GPU Instancing สำหรับ Particles

```javascript
// InstancedMesh เหมาะกับ particles ที่ต้องการ rotation และ custom shapes
const count = 5000;
const geo = new THREE.PlaneGeometry(0.1, 0.1); // หรือ CustomGeometry
const mat = new THREE.MeshBasicMaterial({
    map: particleTexture,
    transparent: true,
    blending: THREE.AdditiveBlending,
    depthWrite: false,
    side: THREE.DoubleSide
});

const instancedMesh = new THREE.InstancedMesh(geo, mat, count);
instancedMesh.instanceMatrix.setUsage(THREE.DynamicDrawUsage); // บอก GPU ว่าจะอัพเดตบ่อย

// กำหนด matrices
const matrix = new THREE.Matrix4();
const position = new THREE.Vector3();
const quaternion = new THREE.Quaternion();
const scale = new THREE.Vector3();

for (let i = 0; i < count; i++) {
    position.set(
        (Math.random() - 0.5) * 20,
        Math.random() * 10,
        (Math.random() - 0.5) * 20
    );
    scale.setScalar(0.1 + Math.random() * 0.3);
    quaternion.setFromAxisAngle(new THREE.Vector3(0, 1, 0), Math.random() * Math.PI * 2);
    
    matrix.compose(position, quaternion, scale);
    instancedMesh.setMatrixAt(i, matrix);
    
    // สีแต่ละ instance
    instancedMesh.setColorAt(i, new THREE.Color().setHSL(Math.random(), 1, 0.6));
}

instancedMesh.instanceMatrix.needsUpdate = true;
instancedMesh.instanceColor.needsUpdate = true;
scene.add(instancedMesh);

// อัพเดตใน loop (billboard effect)
function updateBillboards() {
    const cameraQuat = camera.quaternion;
    for (let i = 0; i < count; i++) {
        instancedMesh.getMatrixAt(i, matrix);
        matrix.decompose(position, quaternion, scale);
        
        // Billboard: หันหาก camera เสมอ
        quaternion.copy(cameraQuat);
        
        matrix.compose(position, quaternion, scale);
        instancedMesh.setMatrixAt(i, matrix);
    }
    instancedMesh.instanceMatrix.needsUpdate = true;
}
```

---

## 4. Fire Effect Demo

```javascript
// Fire Particle System
class FireSystem {
    constructor(scene, position) {
        this.scene = scene;
        this.position = position.clone();
        this.particles = [];
        this.maxParticles = 500;
        
        // Geometry
        this.geo = new THREE.BufferGeometry();
        this.posArray = new Float32Array(this.maxParticles * 3);
        this.colArray = new Float32Array(this.maxParticles * 3);
        this.sizeArray = new Float32Array(this.maxParticles);
        
        this.geo.setAttribute('position', new THREE.BufferAttribute(this.posArray, 3));
        this.geo.setAttribute('color', new THREE.BufferAttribute(this.colArray, 3));
        this.geo.setAttribute('size', new THREE.BufferAttribute(this.sizeArray, 1));
        
        // Fire texture
        this.texture = this.createFireTexture();
        
        this.mat = new THREE.PointsMaterial({
            size: 0.5,
            map: this.texture,
            vertexColors: true,
            transparent: true,
            blending: THREE.AdditiveBlending,
            depthWrite: false,
            sizeAttenuation: true
        });
        
        this.points = new THREE.Points(this.geo, this.mat);
        scene.add(this.points);
        
        // Fire light
        this.light = new THREE.PointLight(0xff4400, 2, 8);
        this.light.position.copy(position);
        this.light.position.y += 1;
        scene.add(this.light);
        
        this.emitTimer = 0;
    }
    
    createFireTexture() {
        const canvas = document.createElement('canvas');
        canvas.width = canvas.height = 64;
        const ctx = canvas.getContext('2d');
        
        const gradient = ctx.createRadialGradient(32, 32, 0, 32, 42, 32);
        gradient.addColorStop(0, 'rgba(255, 255, 255, 1)');
        gradient.addColorStop(0.2, 'rgba(255, 200, 100, 0.8)');
        gradient.addColorStop(0.5, 'rgba(255, 100, 0, 0.4)');
        gradient.addColorStop(1, 'rgba(0, 0, 0, 0)');
        
        ctx.fillStyle = gradient;
        ctx.fillRect(0, 0, 64, 64);
        
        return new THREE.CanvasTexture(canvas);
    }
    
    emit() {
        if (this.particles.length >= this.maxParticles) return;
        
        this.particles.push({
            position: this.position.clone().add(new THREE.Vector3(
                (Math.random() - 0.5) * 0.5,
                0,
                (Math.random() - 0.5) * 0.5
            )),
            velocity: new THREE.Vector3(
                (Math.random() - 0.5) * 0.5,
                1 + Math.random() * 2,
                (Math.random() - 0.5) * 0.5
            ),
            life: 0,
            maxLife: 0.5 + Math.random() * 0.8,
            size: 0.5 + Math.random() * 0.5
        });
    }
    
    update(delta) {
        // Emit
        this.emitTimer += delta;
        const emitRate = 60;
        while (this.emitTimer > 1 / emitRate) {
            this.emit();
            this.emitTimer -= 1 / emitRate;
        }
        
        // Update alive particles
        const alive = [];
        for (const p of this.particles) {
            p.life += delta;
            if (p.life >= p.maxLife) continue; // ตาย
            
            p.velocity.x *= (1 - 2 * delta);
            p.velocity.z *= (1 - 2 * delta);
            p.velocity.y += 0.1 * delta; // ลอยขึ้น
            
            p.position.addScaledVector(p.velocity, delta);
            alive.push(p);
        }
        this.particles = alive;
        
        // Update buffers
        const count = Math.min(alive.length, this.maxParticles);
        for (let i = 0; i < count; i++) {
            const p = alive[i];
            const ratio = p.life / p.maxLife;
            const i3 = i * 3;
            
            this.posArray[i3] = p.position.x;
            this.posArray[i3 + 1] = p.position.y;
            this.posArray[i3 + 2] = p.position.z;
            
            // Fire color: white -> yellow -> orange -> red
            if (ratio < 0.3) {
                // white to yellow
                const t = ratio / 0.3;
                this.colArray[i3] = 1;
                this.colArray[i3 + 1] = 1 - t * 0.3;
                this.colArray[i3 + 2] = 1 - t;
            } else if (ratio < 0.6) {
                // yellow to orange
                const t = (ratio - 0.3) / 0.3;
                this.colArray[i3] = 1;
                this.colArray[i3 + 1] = 0.7 - t * 0.4;
                this.colArray[i3 + 2] = 0;
            } else {
                // orange to red+fade
                const t = (ratio - 0.6) / 0.4;
                this.colArray[i3] = 1 - t * 0.3;
                this.colArray[i3 + 1] = 0.3 - t * 0.3;
                this.colArray[i3 + 2] = 0;
            }
            
            this.sizeArray[i] = p.size * (1 - ratio * 0.5);
        }
        
        this.geo.setDrawRange(0, count);
        this.geo.attributes.position.needsUpdate = true;
        this.geo.attributes.color.needsUpdate = true;
        this.geo.attributes.size.needsUpdate = true;
        
        // Flicker light
        this.light.intensity = 1.5 + Math.sin(Date.now() * 0.01) * 0.5;
    }
    
    dispose() {
        this.scene.remove(this.points);
        this.scene.remove(this.light);
        this.geo.dispose();
        this.mat.dispose();
        this.texture.dispose();
    }
}
```

---

## 5. Magic Spell Effect

```javascript
// Magic spell particle system ที่มี complex behaviors
class MagicSpell {
    constructor(scene, startPos, endPos, color = 0x8844ff) {
        this.scene = scene;
        this.start = startPos.clone();
        this.end = endPos.clone();
        this.color = new THREE.Color(color);
        this.progress = 0;
        this.active = true;
        
        this.particles = [];
        this.maxParticles = 300;
        
        // Geometry
        this.geo = new THREE.BufferGeometry();
        this.posArray = new Float32Array(this.maxParticles * 3);
        this.colArray = new Float32Array(this.maxParticles * 3);
        
        this.geo.setAttribute('position', new THREE.BufferAttribute(this.posArray, 3));
        this.geo.setAttribute('color', new THREE.BufferAttribute(this.colArray, 3));
        
        const texture = this.createSpellTexture(color);
        this.mat = new THREE.PointsMaterial({
            size: 0.15,
            map: texture,
            vertexColors: true,
            transparent: true,
            blending: THREE.AdditiveBlending,
            depthWrite: false
        });
        
        this.points = new THREE.Points(this.geo, this.mat);
        scene.add(this.points);
    }
    
    createSpellTexture(color) {
        const canvas = document.createElement('canvas');
        canvas.width = canvas.height = 64;
        const ctx = canvas.getContext('2d');
        
        const hex = '#' + new THREE.Color(color).getHexString();
        const gradient = ctx.createRadialGradient(32, 32, 0, 32, 32, 32);
        gradient.addColorStop(0, 'white');
        gradient.addColorStop(0.2, hex);
        gradient.addColorStop(1, 'rgba(0,0,0,0)');
        
        ctx.fillStyle = gradient;
        ctx.fillRect(0, 0, 64, 64);
        
        return new THREE.CanvasTexture(canvas);
    }
    
    spawnParticle(position) {
        if (this.particles.length >= this.maxParticles) return;
        
        const spread = 0.3;
        this.particles.push({
            position: position.clone().add(new THREE.Vector3(
                (Math.random() - 0.5) * spread,
                (Math.random() - 0.5) * spread,
                (Math.random() - 0.5) * spread
            )),
            velocity: new THREE.Vector3(
                (Math.random() - 0.5) * 1.5,
                (Math.random() - 0.5) * 1.5 + 0.5,
                (Math.random() - 0.5) * 1.5
            ),
            life: 0,
            maxLife: 0.5 + Math.random() * 0.5,
            swirl: Math.random() * Math.PI * 2,
            swirlSpeed: (Math.random() - 0.5) * 5
        });
    }
    
    update(delta) {
        if (!this.active) return;
        
        // Progress ตาม path
        this.progress += delta * 2;
        
        // Spawn particles ตามเส้นทาง
        const spellPos = this.start.clone().lerp(this.end, Math.min(this.progress, 1));
        for (let i = 0; i < 5; i++) {
            this.spawnParticle(spellPos);
        }
        
        // เพิ่ม spiral effect
        const spiralAngle = this.progress * Math.PI * 8;
        const spiralRadius = 0.3 * (1 - Math.min(this.progress, 1));
        const spiralOffset = new THREE.Vector3(
            Math.cos(spiralAngle) * spiralRadius,
            0,
            Math.sin(spiralAngle) * spiralRadius
        );
        this.spawnParticle(spellPos.clone().add(spiralOffset));
        
        if (this.progress >= 1) {
            // Impact!
            for (let i = 0; i < 50; i++) {
                this.particles.push({
                    position: this.end.clone(),
                    velocity: new THREE.Vector3(
                        (Math.random() - 0.5) * 5,
                        (Math.random() - 0.5) * 5,
                        (Math.random() - 0.5) * 5
                    ),
                    life: 0,
                    maxLife: 0.5 + Math.random(),
                    swirl: 0, swirlSpeed: 0
                });
            }
            this.active = false;
        }
        
        // Update particles
        const alive = [];
        for (const p of this.particles) {
            p.life += delta;
            if (p.life >= p.maxLife) continue;
            
            p.velocity.multiplyScalar(1 - 2 * delta);
            p.position.addScaledVector(p.velocity, delta);
            
            alive.push(p);
        }
        this.particles = alive;
        
        const count = Math.min(alive.length, this.maxParticles);
        for (let i = 0; i < count; i++) {
            const p = alive[i];
            const ratio = p.life / p.maxLife;
            const i3 = i * 3;
            
            this.posArray[i3] = p.position.x;
            this.posArray[i3 + 1] = p.position.y;
            this.posArray[i3 + 2] = p.position.z;
            
            // Color fade
            const fade = 1 - ratio;
            this.colArray[i3] = this.color.r * fade;
            this.colArray[i3 + 1] = this.color.g * fade;
            this.colArray[i3 + 2] = this.color.b * fade;
        }
        
        this.geo.setDrawRange(0, count);
        this.geo.attributes.position.needsUpdate = true;
        this.geo.attributes.color.needsUpdate = true;
    }
    
    dispose() {
        this.scene.remove(this.points);
        this.geo.dispose();
        this.mat.dispose();
    }
    
    get isDone() {
        return !this.active && this.particles.length === 0;
    }
}
```

---

## 6. Vertex Colors Animation

```javascript
// Animate vertex colors เพื่อเอฟเฟกต์ที่สวยงาม
class ColorAnimatedParticles {
    constructor(count) {
        this.count = count;
        this.geometry = new THREE.BufferGeometry();
        
        const positions = new Float32Array(count * 3);
        const colors = new Float32Array(count * 3);
        const phases = new Float32Array(count); // phase offset
        
        for (let i = 0; i < count; i++) {
            const i3 = i * 3;
            
            // Random position บน sphere surface
            const phi = Math.acos(2 * Math.random() - 1);
            const theta = Math.random() * Math.PI * 2;
            const r = 5;
            
            positions[i3] = r * Math.sin(phi) * Math.cos(theta);
            positions[i3 + 1] = r * Math.sin(phi) * Math.sin(theta);
            positions[i3 + 2] = r * Math.cos(phi);
            
            // Initial color
            colors[i3] = 1;
            colors[i3 + 1] = 1;
            colors[i3 + 2] = 1;
            
            phases[i] = Math.random() * Math.PI * 2;
        }
        
        this.geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
        this.geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));
        this.phases = phases;
        
        this.material = new THREE.PointsMaterial({
            size: 0.05,
            vertexColors: true,
            transparent: true,
            blending: THREE.AdditiveBlending,
            depthWrite: false
        });
        
        this.points = new THREE.Points(this.geometry, this.material);
    }
    
    update(elapsed) {
        const colors = this.geometry.attributes.color.array;
        
        for (let i = 0; i < this.count; i++) {
            const i3 = i * 3;
            const phase = this.phases[i];
            const t = elapsed + phase;
            
            // Rainbow cycle
            const hue = (t * 0.1 + phase * 0.159) % 1;
            const c = new THREE.Color().setHSL(hue, 1, 0.6);
            
            colors[i3] = c.r;
            colors[i3 + 1] = c.g;
            colors[i3 + 2] = c.b;
        }
        
        this.geometry.attributes.color.needsUpdate = true;
        this.points.rotation.y = elapsed * 0.1;
    }
}
```

---

## 7. Complete Demo: Particles Showcase

```html
<!DOCTYPE html>
<html>
<head>
    <title>Advanced Particles Demo</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; font-family: monospace; }
        
        #panel {
            position: absolute;
            top: 10px;
            right: 10px;
            background: rgba(0,0,0,0.85);
            color: white;
            padding: 15px;
            border-radius: 8px;
            font-size: 12px;
            width: 200px;
        }
        
        h3 { color: #e94560; margin-bottom: 10px; }
        
        .btn {
            display: block;
            width: 100%;
            padding: 8px;
            margin: 4px 0;
            background: #1a2a3a;
            color: white;
            border: 1px solid #4ecdc4;
            border-radius: 4px;
            cursor: pointer;
            font-family: monospace;
            text-align: left;
        }
        .btn:hover { background: #4ecdc4; color: #000; }
        .btn.active { background: #e94560; border-color: #e94560; }
        
        #stats {
            position: absolute;
            bottom: 10px;
            left: 10px;
            color: #4ecdc4;
            font-size: 12px;
        }
    </style>
</head>
<body>

<div id="panel">
    <h3>Particle Effects</h3>
    <button class="btn active" id="btnFire">🔥 Fire</button>
    <button class="btn" id="btnSmoke">💨 Smoke</button>
    <button class="btn" id="btnMagic">✨ Magic</button>
    <button class="btn" id="btnExplosion">💥 Explosion</button>
    <button class="btn" id="btnRain">🌧️ Rain</button>
    <button class="btn" id="btnSnow">❄️ Snow</button>
    
    <div style="margin-top: 10px; color: #aaa;">
        คลิกใน scene เพื่อสร้าง effect
    </div>
</div>

<div id="stats">Particles: 0</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';

// Scene
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x050510);
scene.fog = new THREE.FogExp2(0x050510, 0.01);

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(0, 8, 20);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

// Lighting
scene.add(new THREE.AmbientLight(0x111133, 0.5));

// Floor
const floor = new THREE.Mesh(
    new THREE.PlaneGeometry(50, 50),
    new THREE.MeshStandardMaterial({ color: 0x111133, roughness: 0.9 })
);
floor.rotation.x = -Math.PI / 2;
scene.add(floor);
scene.add(new THREE.GridHelper(50, 50, 0x222244, 0x111133));

// Active effect type
let activeEffect = 'fire';
document.querySelectorAll('.btn').forEach(btn => {
    btn.addEventListener('click', () => {
        document.querySelectorAll('.btn').forEach(b => b.classList.remove('active'));
        btn.classList.add('active');
        activeEffect = btn.id.replace('btn', '').toLowerCase();
    });
});

// Particle texture
function createGlowTexture(color = 'white', innerOpacity = 1) {
    const canvas = document.createElement('canvas');
    canvas.width = canvas.height = 64;
    const ctx = canvas.getContext('2d');
    const g = ctx.createRadialGradient(32, 32, 0, 32, 32, 32);
    g.addColorStop(0, `rgba(255,255,255,${innerOpacity})`);
    g.addColorStop(0.15, color);
    g.addColorStop(1, 'rgba(0,0,0,0)');
    ctx.fillStyle = g;
    ctx.fillRect(0, 0, 64, 64);
    return new THREE.CanvasTexture(canvas);
}

// Generic particle effect function
function createEffect(type, origin) {
    const systems = [];
    const count = 500;
    
    let configs;
    switch(type) {
        case 'fire':
            configs = { count: 300, color: 0xff4400, size: 0.3, gravity: -0.5, blending: THREE.AdditiveBlending };
            break;
        case 'smoke':
            configs = { count: 100, color: 0x888888, size: 1, gravity: -0.2, blending: THREE.NormalBlending };
            break;
        case 'magic':
            configs = { count: 200, color: 0x8844ff, size: 0.2, gravity: 0, blending: THREE.AdditiveBlending };
            break;
        case 'explosion':
            configs = { count: 500, color: 0xff8800, size: 0.4, gravity: 2, blending: THREE.AdditiveBlending };
            break;
        case 'rain':
            configs = { count: 1000, color: 0x4499ff, size: 0.05, gravity: 10, blending: THREE.NormalBlending };
            break;
        case 'snow':
            configs = { count: 300, color: 0xffffff, size: 0.1, gravity: 0.5, blending: THREE.NormalBlending };
            break;
        default:
            configs = { count: 200, color: 0xffffff, size: 0.2, gravity: 1, blending: THREE.AdditiveBlending };
    }
    
    const geo = new THREE.BufferGeometry();
    const particles = [];
    const maxP = configs.count;
    const posArr = new Float32Array(maxP * 3);
    const colArr = new Float32Array(maxP * 3);
    
    geo.setAttribute('position', new THREE.BufferAttribute(posArr, 3));
    geo.setAttribute('color', new THREE.BufferAttribute(colArr, 3));
    
    const color = new THREE.Color(configs.color);
    const tex = createGlowTexture('#' + color.getHexString());
    
    const mat = new THREE.PointsMaterial({
        size: configs.size,
        map: tex,
        vertexColors: true,
        transparent: true,
        blending: configs.blending,
        depthWrite: false,
        opacity: type === 'smoke' ? 0.3 : 0.8
    });
    
    const pts = new THREE.Points(geo, mat);
    scene.add(pts);
    
    // Spawn particles
    for (let i = 0; i < maxP; i++) {
        const spread = type === 'explosion' ? 0 : 0.5;
        particles.push({
            pos: origin.clone().add(new THREE.Vector3(
                (Math.random() - 0.5) * spread,
                0,
                (Math.random() - 0.5) * spread
            )),
            vel: new THREE.Vector3(
                (Math.random() - 0.5) * (type === 'explosion' ? 8 : 2),
                type === 'rain' ? -5 : (1 + Math.random() * 3),
                (Math.random() - 0.5) * (type === 'explosion' ? 8 : 2)
            ),
            life: 0,
            maxLife: type === 'explosion' ? 0.3 + Math.random() * 0.7 : 1 + Math.random() * 2,
            delay: type === 'explosion' ? 0 : Math.random() * 2
        });
    }
    
    return {
        update(delta) {
            let alive = 0;
            for (const p of particles) {
                p.delay -= delta;
                if (p.delay > 0) continue;
                
                p.life += delta;
                if (p.life > p.maxLife) continue;
                
                p.vel.y -= configs.gravity * delta;
                if (type !== 'magic') p.vel.multiplyScalar(1 - 0.5 * delta);
                p.pos.addScaledVector(p.vel, delta);
                
                // Rain restart
                if (type === 'rain' && p.pos.y < 0) {
                    p.pos.set(origin.x + (Math.random() - 0.5) * 20, 10, origin.z + (Math.random() - 0.5) * 20);
                    p.life = 0;
                }
                
                const i3 = alive * 3;
                posArr[i3] = p.pos.x;
                posArr[i3 + 1] = p.pos.y;
                posArr[i3 + 2] = p.pos.z;
                
                const ratio = p.life / p.maxLife;
                const fade = 1 - ratio;
                colArr[i3] = color.r * fade;
                colArr[i3 + 1] = color.g * fade;
                colArr[i3 + 2] = color.b * fade;
                
                alive++;
            }
            
            geo.setDrawRange(0, alive);
            geo.attributes.position.needsUpdate = true;
            geo.attributes.color.needsUpdate = true;
            
            document.getElementById('stats').textContent = `Particles: ${alive}`;
            
            return alive > 0 || type === 'rain' || type === 'snow';
        },
        dispose() {
            scene.remove(pts);
            geo.dispose();
            mat.dispose();
            tex.dispose();
        }
    };
}

// Active effects
let activeEffects = [];

// Click to create effect
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();

renderer.domElement.addEventListener('click', (e) => {
    mouse.x = (e.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(e.clientY / window.innerHeight) * 2 + 1;
    
    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObject(floor);
    
    if (intersects.length > 0) {
        // ลบ effect เก่า (ยกเว้น rain/snow)
        if (activeEffect !== 'rain' && activeEffect !== 'snow') {
            activeEffects.forEach(e => e.dispose());
            activeEffects = [];
        }
        
        const effect = createEffect(activeEffect, intersects[0].point);
        activeEffects.push(effect);
    }
});

window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});

// Animation
const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    const delta = Math.min(clock.getDelta(), 0.05);
    
    activeEffects = activeEffects.filter(e => {
        const alive = e.update(delta);
        if (!alive) { e.dispose(); return false; }
        return true;
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

## 8. Particle System Class Design

```javascript
// Base class สำหรับ particle systems
class BaseParticleSystem {
    constructor(maxCount, scene) {
        this.maxCount = maxCount;
        this.scene = scene;
        this.aliveCount = 0;
        
        this.posBuffer = new Float32Array(maxCount * 3);
        this.colBuffer = new Float32Array(maxCount * 3);
        
        this.geo = new THREE.BufferGeometry();
        this.geo.setAttribute('position', new THREE.BufferAttribute(this.posBuffer, 3));
        this.geo.setAttribute('color', new THREE.BufferAttribute(this.colBuffer, 3));
        
        this.mat = this.createMaterial();
        this.mesh = new THREE.Points(this.geo, this.mat);
        scene.add(this.mesh);
    }
    
    // Override ใน subclass
    createMaterial() {
        return new THREE.PointsMaterial({
            size: 0.1, vertexColors: true,
            transparent: true, blending: THREE.AdditiveBlending,
            depthWrite: false
        });
    }
    
    update(delta) {
        this.aliveCount = 0;
        this.onUpdate(delta);
        this.geo.setDrawRange(0, this.aliveCount);
        this.geo.attributes.position.needsUpdate = true;
        this.geo.attributes.color.needsUpdate = true;
    }
    
    onUpdate(delta) { /* override */ }
    
    addParticle(x, y, z, r, g, b) {
        if (this.aliveCount >= this.maxCount) return;
        const i3 = this.aliveCount * 3;
        this.posBuffer[i3] = x;
        this.posBuffer[i3 + 1] = y;
        this.posBuffer[i3 + 2] = z;
        this.colBuffer[i3] = r;
        this.colBuffer[i3 + 1] = g;
        this.colBuffer[i3 + 2] = b;
        this.aliveCount++;
    }
    
    dispose() {
        this.scene.remove(this.mesh);
        this.geo.dispose();
        this.mat.dispose();
    }
}
```

---

## แบบฝึกหัด

1. **Smoke Simulation**: สร้างควันที่สมจริงพร้อม turbulence
2. **Fireworks**: ระเบิดพลุไฟที่ขึ้นไปแล้วแตกออก
3. **Water Fountain**: น้ำพุที่กระดอนตาม physics
4. **Trail Effect**: ทิ้งร่องรอย particles ตามการเคลื่อนที่ของวัตถุ
5. **Vortex Particles**: Particles ที่วนเป็นก้นหอย

---

## สรุป

- **Points** เหมาะกับ particles มาก เพราะ 1 draw call
- **Sprite** เหมาะกับ particles น้อยที่ต้องการ rotation
- Particle lifecycle (birth/death) ทำได้ด้วย Array filter
- `setDrawRange` ประหยัด GPU สำหรับ dynamic particle count
- `needsUpdate = true` บอก GPU ว่า buffer เปลี่ยนแล้ว
- `AdditiveBlending` สร้าง glow effect สวยงาม
- `depthWrite: false` ป้องกัน z-fighting
- Performance: update buffer ทุก frame หนักกว่า static particles
