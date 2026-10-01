# Part 47: Fire, Smoke & Particle VFX ใน Three.js

## บทนำ

Particle VFX เป็นหัวใจของ visual effects ในเกม บทนี้ครอบคลุม:
- **Fire System** - ระบบไฟด้วย particles + sprite
- **Smoke** - ควันที่โปร่งแสงพร้อม depth sorting
- **Explosion** - เอฟเฟกต์ระเบิด
- **Magic Effects** - sparkles, energy field
- **Shockwave** - คลื่นกระแทกใน shader

## 1. Particle System Base Class

```javascript
// ===== Generic Particle System =====
class ParticleSystem {
  constructor(scene, maxParticles, config = {}) {
    this.scene = scene;
    this.maxParticles = maxParticles;
    this.particles = [];
    this.config = {
      blending: THREE.AdditiveBlending,
      transparent: true,
      depthWrite: false,
      ...config
    };
    
    this._initGeometry();
    this._initMaterial();
    this._createPoints();
  }
  
  _initGeometry() {
    this.geo = new THREE.BufferGeometry();
    this.positions = new Float32Array(this.maxParticles * 3);
    this.colors = new Float32Array(this.maxParticles * 4); // RGBA
    this.sizes = new Float32Array(this.maxParticles);
    
    this.posAttr = new THREE.BufferAttribute(this.positions, 3);
    this.colAttr = new THREE.BufferAttribute(this.colors, 4);
    this.sizeAttr = new THREE.BufferAttribute(this.sizes, 1);
    
    this.posAttr.setUsage(THREE.DynamicDrawUsage);
    this.colAttr.setUsage(THREE.DynamicDrawUsage);
    this.sizeAttr.setUsage(THREE.DynamicDrawUsage);
    
    this.geo.setAttribute('position', this.posAttr);
    this.geo.setAttribute('color', this.colAttr);
    this.geo.setAttribute('size', this.sizeAttr);
    this.geo.setDrawRange(0, 0);
  }
  
  _initMaterial() {
    this.mat = new THREE.ShaderMaterial({
      uniforms: {
        texture: { value: this._createSprite() },
        time: { value: 0 }
      },
      vertexShader: `
        attribute float size;
        attribute vec4 color;
        varying vec4 vColor;
        
        void main() {
          vColor = color;
          vec4 mv = modelViewMatrix * vec4(position, 1.0);
          gl_PointSize = size * (400.0 / -mv.z);
          gl_Position = projectionMatrix * mv;
        }
      `,
      fragmentShader: `
        uniform sampler2D texture;
        varying vec4 vColor;
        
        void main() {
          vec4 tex = texture2D(texture, gl_PointCoord);
          if (tex.a < 0.01 || vColor.a < 0.01) discard;
          gl_FragColor = vec4(vColor.rgb * tex.rgb, tex.a * vColor.a);
        }
      `,
      blending: this.config.blending,
      transparent: this.config.transparent,
      depthWrite: this.config.depthWrite,
      vertexColors: true
    });
  }
  
  _createSprite() {
    const canvas = document.createElement('canvas');
    canvas.width = canvas.height = 64;
    const ctx = canvas.getContext('2d');
    
    const grad = ctx.createRadialGradient(32, 32, 0, 32, 32, 32);
    grad.addColorStop(0, 'rgba(255,255,255,1)');
    grad.addColorStop(0.4, 'rgba(255,255,255,0.8)');
    grad.addColorStop(1, 'rgba(255,255,255,0)');
    
    ctx.fillStyle = grad;
    ctx.fillRect(0, 0, 64, 64);
    
    return new THREE.CanvasTexture(canvas);
  }
  
  _createPoints() {
    this.points = new THREE.Points(this.geo, this.mat);
    this.scene.add(this.points);
  }
  
  emit(position, config) {
    if (this.particles.length >= this.maxParticles) {
      // Reuse oldest dead particle
      const dead = this.particles.find(p => !p.alive);
      if (!dead) return;
      Object.assign(dead, { alive: true, age: 0, ...config, position: position.clone() });
      return dead;
    }
    
    const particle = {
      alive: true,
      age: 0,
      position: position.clone(),
      velocity: config.velocity || new THREE.Vector3(),
      life: config.life || 1,
      startSize: config.size || 10,
      endSize: config.endSize || 0,
      startColor: config.startColor || new THREE.Color(1, 1, 1),
      endColor: config.endColor || new THREE.Color(0, 0, 0),
      startAlpha: config.startAlpha !== undefined ? config.startAlpha : 1,
      endAlpha: config.endAlpha !== undefined ? config.endAlpha : 0,
      gravity: config.gravity || 0,
      drag: config.drag || 0
    };
    
    this.particles.push(particle);
    return particle;
  }
  
  update(deltaTime) {
    let activeCount = 0;
    
    for (let i = 0; i < this.particles.length; i++) {
      const p = this.particles[i];
      if (!p.alive) continue;
      
      p.age += deltaTime;
      if (p.age >= p.life) {
        p.alive = false;
        continue;
      }
      
      const t = p.age / p.life;
      
      // Physics
      p.velocity.y -= p.gravity * deltaTime;
      p.velocity.multiplyScalar(1 - p.drag * deltaTime);
      p.position.addScaledVector(p.velocity, deltaTime);
      
      // Write to buffers
      const idx = activeCount;
      
      this.positions[idx * 3] = p.position.x;
      this.positions[idx * 3 + 1] = p.position.y;
      this.positions[idx * 3 + 2] = p.position.z;
      
      const color = p.startColor.clone().lerp(p.endColor, t);
      this.colors[idx * 4] = color.r;
      this.colors[idx * 4 + 1] = color.g;
      this.colors[idx * 4 + 2] = color.b;
      this.colors[idx * 4 + 3] = THREE.MathUtils.lerp(p.startAlpha, p.endAlpha, t);
      
      this.sizes[idx] = THREE.MathUtils.lerp(p.startSize, p.endSize, t);
      
      activeCount++;
    }
    
    this.geo.setDrawRange(0, activeCount);
    this.posAttr.needsUpdate = true;
    this.colAttr.needsUpdate = true;
    this.sizeAttr.needsUpdate = true;
    
    // Clean dead particles
    if (this.particles.length > this.maxParticles * 1.5) {
      this.particles = this.particles.filter(p => p.alive);
    }
  }
  
  setPosition(pos) {
    this.points.position.copy(pos);
  }
}
```

## 2. Fire System

```javascript
class FireSystem extends ParticleSystem {
  constructor(scene, intensity = 1) {
    super(scene, 500, {
      blending: THREE.AdditiveBlending,
      depthWrite: false
    });
    
    this.intensity = intensity;
    this.emitRate = 30 * intensity; // particles per second
    this.emitAccum = 0;
    this.position = new THREE.Vector3();
    
    // Override sprite with fire texture
    this.mat.uniforms.texture.value = this._createFireSprite();
  }
  
  _createFireSprite() {
    const canvas = document.createElement('canvas');
    canvas.width = canvas.height = 128;
    const ctx = canvas.getContext('2d');
    
    // Teardrop fire shape
    const grad = ctx.createRadialGradient(64, 80, 0, 64, 64, 50);
    grad.addColorStop(0,   'rgba(255, 255, 200, 1)');
    grad.addColorStop(0.2, 'rgba(255, 200, 50, 0.9)');
    grad.addColorStop(0.5, 'rgba(255, 80, 10, 0.6)');
    grad.addColorStop(0.8, 'rgba(200, 30, 0, 0.2)');
    grad.addColorStop(1,   'rgba(100, 0, 0, 0)');
    
    ctx.fillStyle = grad;
    ctx.beginPath();
    ctx.arc(64, 70, 45, 0, Math.PI * 2);
    ctx.fill();
    
    return new THREE.CanvasTexture(canvas);
  }
  
  _emitParticle() {
    const spread = 0.5 * this.intensity;
    
    this.emit(this.position, {
      velocity: new THREE.Vector3(
        (Math.random() - 0.5) * spread,
        1.5 + Math.random() * 2 * this.intensity,
        (Math.random() - 0.5) * spread
      ),
      life: 0.5 + Math.random() * 0.8,
      size: 15 + Math.random() * 20 * this.intensity,
      endSize: 0,
      startColor: new THREE.Color(1.5, 1.2, 0.3),  // bright yellow-white
      endColor:   new THREE.Color(0.8, 0.1, 0),     // dark red
      startAlpha: 0.9,
      endAlpha: 0,
      gravity: -0.2,
      drag: 0.5
    });
  }
  
  update(deltaTime) {
    // Emit particles
    this.emitAccum += this.emitRate * deltaTime;
    while (this.emitAccum >= 1) {
      this._emitParticle();
      this.emitAccum--;
    }
    
    super.update(deltaTime);
  }
}
```

## 3. Smoke System

```javascript
class SmokeSystem extends ParticleSystem {
  constructor(scene) {
    super(scene, 200, {
      blending: THREE.NormalBlending,
      depthWrite: false,
      transparent: true
    });
    
    this.emitRate = 5;
    this.emitAccum = 0;
    this.position = new THREE.Vector3();
    
    this.mat.uniforms.texture.value = this._createSmokeSprite();
  }
  
  _createSmokeSprite() {
    const canvas = document.createElement('canvas');
    canvas.width = canvas.height = 128;
    const ctx = canvas.getContext('2d');
    
    // Soft cloud sprite
    const grad = ctx.createRadialGradient(64, 64, 0, 64, 64, 60);
    grad.addColorStop(0,   'rgba(180, 180, 180, 0.6)');
    grad.addColorStop(0.4, 'rgba(150, 150, 150, 0.3)');
    grad.addColorStop(0.7, 'rgba(120, 120, 120, 0.1)');
    grad.addColorStop(1,   'rgba(100, 100, 100, 0)');
    
    ctx.fillStyle = grad;
    ctx.fillRect(0, 0, 128, 128);
    
    return new THREE.CanvasTexture(canvas);
  }
  
  _emitParticle() {
    this.emit(this.position, {
      velocity: new THREE.Vector3(
        (Math.random() - 0.5) * 0.3,
        0.3 + Math.random() * 0.4,
        (Math.random() - 0.5) * 0.3
      ),
      life: 3 + Math.random() * 2,
      size: 20,
      endSize: 80,
      startColor: new THREE.Color(0.3, 0.3, 0.3),
      endColor:   new THREE.Color(0.6, 0.6, 0.6),
      startAlpha: 0.5,
      endAlpha: 0,
      gravity: -0.05,
      drag: 0.3
    });
  }
  
  update(deltaTime) {
    this.emitAccum += this.emitRate * deltaTime;
    while (this.emitAccum >= 1) {
      this._emitParticle();
      this.emitAccum--;
    }
    super.update(deltaTime);
  }
}
```

## 4. Explosion Effect

```javascript
class ExplosionEffect {
  constructor(scene) {
    this.scene = scene;
    this.active = false;
  }
  
  trigger(position, scale = 1) {
    this.active = true;
    
    // === Fire ball ===
    const fireball = new ParticleSystem(this.scene, 300, {
      blending: THREE.AdditiveBlending, depthWrite: false
    });
    fireball.setPosition(position);
    
    for (let i = 0; i < 200; i++) {
      const dir = new THREE.Vector3(
        Math.random() - 0.5,
        Math.random() - 0.5,
        Math.random() - 0.5
      ).normalize();
      
      fireball.emit(new THREE.Vector3(), {
        velocity: dir.multiplyScalar(5 + Math.random() * 8 * scale),
        life: 0.5 + Math.random() * 0.8,
        size: 20 + Math.random() * 30 * scale,
        endSize: 0,
        startColor: new THREE.Color(2.0, 1.5, 0.5),
        endColor:   new THREE.Color(0.5, 0.05, 0),
        startAlpha: 1, endAlpha: 0,
        gravity: 1, drag: 2
      });
    }
    
    // === Smoke ===
    const smoke = new ParticleSystem(this.scene, 100, {
      blending: THREE.NormalBlending, depthWrite: false
    });
    smoke.setPosition(position);
    
    for (let i = 0; i < 50; i++) {
      const dir = new THREE.Vector3(
        (Math.random() - 0.5) * 0.5,
        1, 
        (Math.random() - 0.5) * 0.5
      );
      
      smoke.emit(new THREE.Vector3(), {
        velocity: dir.multiplyScalar(2 + Math.random() * 3),
        life: 3 + Math.random() * 2,
        size: 30 * scale, endSize: 100 * scale,
        startColor: new THREE.Color(0.2, 0.2, 0.2),
        endColor:   new THREE.Color(0.5, 0.5, 0.5),
        startAlpha: 0.8, endAlpha: 0,
        gravity: -0.1, drag: 0.3
      });
    }
    
    // === Debris ===
    const debrisMeshes = [];
    const debrisMat = new THREE.MeshStandardMaterial({ color: 0x444444, roughness: 1 });
    
    for (let i = 0; i < 20; i++) {
      const debris = new THREE.Mesh(
        new THREE.TetrahedronGeometry(0.1 + Math.random() * 0.3 * scale, 0),
        debrisMat
      );
      debris.position.copy(position);
      debris.castShadow = true;
      
      const vel = new THREE.Vector3(
        (Math.random() - 0.5) * 15,
        5 + Math.random() * 15,
        (Math.random() - 0.5) * 15
      ).multiplyScalar(scale);
      
      debris.userData.vel = vel;
      debris.userData.rotVel = new THREE.Vector3(
        Math.random() * 5, Math.random() * 5, Math.random() * 5
      );
      debris.userData.life = 3;
      debris.userData.age = 0;
      this.scene.add(debris);
      debrisMeshes.push(debris);
    }
    
    // Shockwave ring
    const shockwave = this._createShockwave(position, scale);
    
    // Lifetime management
    let elapsed = 0;
    const systems = [fireball, smoke];
    
    const updateExplosion = (dt) => {
      elapsed += dt;
      
      systems.forEach(s => s.update(dt));
      
      // Update debris
      for (let i = debrisMeshes.length - 1; i >= 0; i--) {
        const d = debrisMeshes[i];
        d.userData.age += dt;
        
        if (d.userData.age > d.userData.life) {
          this.scene.remove(d);
          debrisMeshes.splice(i, 1);
          continue;
        }
        
        d.userData.vel.y -= 9.8 * dt;
        d.position.addScaledVector(d.userData.vel, dt);
        d.rotation.x += d.userData.rotVel.x * dt;
        d.rotation.y += d.userData.rotVel.y * dt;
        
        // Bounce
        if (d.position.y < 0) {
          d.position.y = 0;
          d.userData.vel.y *= -0.3;
          d.userData.vel.x *= 0.8;
          d.userData.vel.z *= 0.8;
        }
      }
      
      // Shockwave
      if (shockwave) {
        const st = elapsed * 3;
        shockwave.scale.setScalar(1 + st * 5 * scale);
        shockwave.material.opacity = Math.max(0, 1 - st);
        if (st > 1) {
          this.scene.remove(shockwave);
        }
      }
      
      return elapsed < 5; // continue for 5 seconds
    };
    
    return updateExplosion;
  }
  
  _createShockwave(position, scale) {
    const geo = new THREE.RingGeometry(0.1, 0.5, 32);
    geo.rotateX(-Math.PI / 2);
    
    const mat = new THREE.MeshBasicMaterial({
      color: 0xffaa44,
      transparent: true,
      opacity: 0.8,
      side: THREE.DoubleSide,
      depthWrite: false,
      blending: THREE.AdditiveBlending
    });
    
    const ring = new THREE.Mesh(geo, mat);
    ring.position.copy(position);
    this.scene.add(ring);
    return ring;
  }
}
```

## 5. Magic Particle Effects

```javascript
// ===== Magic Sparkles =====
class MagicSparkles extends ParticleSystem {
  constructor(scene, color = new THREE.Color(0.5, 0.3, 1.0)) {
    super(scene, 200, { blending: THREE.AdditiveBlending, depthWrite: false });
    this.color = color;
    this.emitRate = 20;
    this.emitAccum = 0;
    this.origin = new THREE.Vector3();
    this.radius = 1;
  }
  
  _emitSpark() {
    // สุ่มตำแหน่งบน sphere
    const angle = Math.random() * Math.PI * 2;
    const elevation = (Math.random() - 0.5) * Math.PI;
    const pos = new THREE.Vector3(
      Math.cos(angle) * Math.cos(elevation) * this.radius,
      Math.sin(elevation) * this.radius,
      Math.sin(angle) * Math.cos(elevation) * this.radius
    );
    
    const inward = pos.clone().negate().normalize();
    const tangent = new THREE.Vector3(
      -Math.sin(angle), 0, Math.cos(angle)
    );
    
    const vel = inward.multiplyScalar(Math.random() * 0.5)
                       .add(tangent.multiplyScalar((Math.random() - 0.5) * 2));
    vel.y += 0.5;
    
    this.emit(pos.add(this.origin), {
      velocity: vel,
      life: 0.3 + Math.random() * 0.7,
      size: 3 + Math.random() * 8,
      endSize: 0,
      startColor: this.color.clone().multiplyScalar(2),
      endColor: this.color.clone().multiplyScalar(0.3),
      startAlpha: 1, endAlpha: 0,
      gravity: 0.3, drag: 1.5
    });
  }
  
  update(dt) {
    this.emitAccum += this.emitRate * dt;
    while (this.emitAccum >= 1) { this._emitSpark(); this.emitAccum--; }
    super.update(dt);
  }
}

// ===== Energy Shield =====
class EnergyShield {
  constructor(scene, radius = 3) {
    this.scene = scene;
    this.radius = radius;
    this._create();
  }
  
  _create() {
    const geo = new THREE.SphereGeometry(this.radius, 32, 32);
    this.mesh = new THREE.Mesh(geo, new THREE.ShaderMaterial({
      uniforms: {
        time: { value: 0 },
        hitPos: { value: new THREE.Vector3(0, 0, 1) },
        hitStrength: { value: 0.0 },
        color: { value: new THREE.Color(0.2, 0.6, 1.0) }
      },
      vertexShader: `
        varying vec3 vNormal;
        varying vec3 vPos;
        void main() {
          vNormal = normalize(normalMatrix * normal);
          vPos = position;
          gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
      `,
      fragmentShader: `
        uniform float time;
        uniform vec3 hitPos;
        uniform float hitStrength;
        uniform vec3 color;
        varying vec3 vNormal;
        varying vec3 vPos;
        
        float hash(vec2 p) { return fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5); }
        
        void main() {
          // Fresnel rim glow
          vec3 viewDir = normalize(cameraPosition - vPos);
          float fresnel = pow(1.0 - abs(dot(viewDir, vNormal)), 3.0);
          
          // Hexagonal grid
          vec2 uv = vec2(atan(vPos.z, vPos.x), acos(vPos.y / length(vPos)));
          uv *= 5.0;
          vec2 grid = fract(uv) - 0.5;
          float hex = length(grid);
          float hexLine = smoothstep(0.45, 0.4, hex);
          
          // Hit ripple
          float hitDist = length(vPos - hitPos);
          float ripple = sin(hitDist * 10.0 - time * 15.0) * exp(-hitDist * 2.0) * hitStrength;
          
          // Animate
          float pulse = sin(time * 2.0) * 0.1 + 0.9;
          
          float alpha = (fresnel * 0.6 + hexLine * 0.3 + ripple) * pulse;
          vec3 finalColor = color + vec3(ripple * 0.5);
          
          if (alpha < 0.01) discard;
          gl_FragColor = vec4(finalColor, alpha * 0.7);
        }
      `,
      transparent: true,
      depthWrite: false,
      side: THREE.DoubleSide,
      blending: THREE.AdditiveBlending
    }));
    
    this.scene.add(this.mesh);
  }
  
  hit(direction) {
    const pos = direction.clone().normalize().multiplyScalar(this.radius);
    this.mesh.material.uniforms.hitPos.value.copy(pos);
    this.mesh.material.uniforms.hitStrength.value = 1.0;
  }
  
  update(time, dt) {
    this.mesh.material.uniforms.time.value = time;
    const hs = this.mesh.material.uniforms.hitStrength;
    hs.value = Math.max(0, hs.value - dt * 2);
  }
}
```

## 6. Complete VFX Demo

```html
<!DOCTYPE html>
<html>
<head>
  <title>Fire & Particle VFX</title>
  <style>
    * { margin: 0; padding: 0; }
    body { background: #0a0a0a; overflow: hidden; }
    canvas { display: block; }
    #controls {
      position: fixed;
      bottom: 20px;
      left: 50%;
      transform: translateX(-50%);
      display: flex;
      gap: 10px;
      flex-wrap: wrap;
      justify-content: center;
    }
    button {
      background: rgba(40,40,80,0.9);
      color: #88aaff;
      border: 1px solid #446;
      padding: 8px 16px;
      border-radius: 4px;
      cursor: pointer;
      font-family: Arial, sans-serif;
      font-size: 14px;
    }
    button:hover { background: rgba(60,60,120,0.9); }
    #info {
      position: fixed;
      top: 20px;
      left: 50%;
      transform: translateX(-50%);
      color: #88aaff;
      font-family: Arial, sans-serif;
      font-size: 16px;
      pointer-events: none;
    }
  </style>
</head>
<body>
<div id="info">Particle VFX Demo - Click to interact</div>
<div id="controls">
  <button id="addFire">Add Fire</button>
  <button id="explode">Explosion</button>
  <button id="addSmoke">Add Smoke</button>
  <button id="addMagic">Magic Sparkles</button>
  <button id="clearAll">Clear All</button>
</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x050508);
const camera = new THREE.PerspectiveCamera(75, innerWidth/innerHeight, 0.1, 500);
camera.position.set(0, 5, 15);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
renderer.outputColorSpace = THREE.SRGBColorSpace;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.5;
renderer.shadowMap.enabled = true;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

// Ground
const ground = new THREE.Mesh(
  new THREE.PlaneGeometry(40, 40),
  new THREE.MeshStandardMaterial({ color: 0x222222, roughness: 0.9 })
);
ground.rotateX(-Math.PI / 2);
ground.receiveShadow = true;
scene.add(ground);

// Grid
scene.add(new THREE.GridHelper(40, 20, 0x333333, 0x222222));

// Lights
scene.add(new THREE.AmbientLight(0x111122, 2));
const pLight = new THREE.PointLight(0x443311, 3, 20);
pLight.position.set(0, 5, 0);
pLight.castShadow = true;
scene.add(pLight);

// ===== PARTICLE SYSTEM BASE =====
function createSprite(type = 'circle') {
  const canvas = document.createElement('canvas');
  canvas.width = canvas.height = 64;
  const ctx = canvas.getContext('2d');
  
  if (type === 'fire') {
    const g = ctx.createRadialGradient(32, 40, 0, 32, 32, 32);
    g.addColorStop(0, 'rgba(255,255,200,1)');
    g.addColorStop(0.3, 'rgba(255,150,30,0.8)');
    g.addColorStop(0.7, 'rgba(255,30,0,0.3)');
    g.addColorStop(1, 'rgba(0,0,0,0)');
    ctx.fillStyle = g; ctx.fillRect(0,0,64,64);
  } else if (type === 'smoke') {
    const g = ctx.createRadialGradient(32,32,0,32,32,32);
    g.addColorStop(0, 'rgba(180,180,180,0.5)');
    g.addColorStop(1, 'rgba(100,100,100,0)');
    ctx.fillStyle = g; ctx.fillRect(0,0,64,64);
  } else {
    const g = ctx.createRadialGradient(32,32,0,32,32,32);
    g.addColorStop(0, 'rgba(255,255,255,1)');
    g.addColorStop(0.5, 'rgba(255,255,255,0.5)');
    g.addColorStop(1, 'rgba(0,0,0,0)');
    ctx.fillStyle = g; ctx.fillRect(0,0,64,64);
  }
  
  return new THREE.CanvasTexture(canvas);
}

// Generic particle system class
class PS {
  constructor(maxP, sprite, blend = THREE.AdditiveBlending) {
    this.maxP = maxP;
    this.ps = [];
    this.positions = new Float32Array(maxP * 3);
    this.colors = new Float32Array(maxP * 4);
    this.sizes = new Float32Array(maxP);
    
    const geo = new THREE.BufferGeometry();
    this.pa = new THREE.BufferAttribute(this.positions, 3);
    this.ca = new THREE.BufferAttribute(this.colors, 4);
    this.sa = new THREE.BufferAttribute(this.sizes, 1);
    this.pa.setUsage(THREE.DynamicDrawUsage);
    this.ca.setUsage(THREE.DynamicDrawUsage);
    this.sa.setUsage(THREE.DynamicDrawUsage);
    geo.setAttribute('position', this.pa);
    geo.setAttribute('color', this.ca);
    geo.setAttribute('size', this.sa);
    geo.setDrawRange(0, 0);
    
    const mat = new THREE.ShaderMaterial({
      uniforms: { tex: { value: sprite } },
      vertexShader: `
        attribute float size; attribute vec4 color; varying vec4 vC;
        void main() {
          vC = color;
          vec4 mv = modelViewMatrix * vec4(position, 1.0);
          gl_PointSize = size * (300.0 / -mv.z);
          gl_Position = projectionMatrix * mv;
        }
      `,
      fragmentShader: `
        uniform sampler2D tex; varying vec4 vC;
        void main() {
          vec4 t = texture2D(tex, gl_PointCoord);
          if (t.a * vC.a < 0.01) discard;
          gl_FragColor = vec4(vC.rgb * t.rgb, t.a * vC.a);
        }
      `,
      blending: blend, transparent: true, depthWrite: false, vertexColors: true
    });
    
    this.points = new THREE.Points(geo, mat);
    this.geo = geo;
    scene.add(this.points);
  }
  
  emit(x, y, z, vx, vy, vz, life, size, eSize, sr, sg, sb, er, eg, eb, sa, ea, grav = 0, drag = 0) {
    this.ps.push({ x,y,z, vx,vy,vz, age:0, life, size, eSize, sr,sg,sb, er,eg,eb, sa, ea, grav, drag });
  }
  
  update(dt) {
    let n = 0;
    for (let i = this.ps.length-1; i >= 0; i--) {
      const p = this.ps[i];
      p.age += dt;
      if (p.age >= p.life) { this.ps.splice(i, 1); continue; }
      const t = p.age / p.life;
      p.vy -= p.grav * dt;
      p.vx *= 1 - p.drag * dt;
      p.vy *= 1 - p.drag * dt;
      p.vz *= 1 - p.drag * dt;
      p.x += p.vx * dt; p.y += p.vy * dt; p.z += p.vz * dt;
      this.positions[n*3] = p.x; this.positions[n*3+1] = p.y; this.positions[n*3+2] = p.z;
      this.colors[n*4]   = p.sr + (p.er - p.sr)*t;
      this.colors[n*4+1] = p.sg + (p.eg - p.sg)*t;
      this.colors[n*4+2] = p.sb + (p.eb - p.sb)*t;
      this.colors[n*4+3] = p.sa + (p.ea - p.sa)*t;
      this.sizes[n] = p.size + (p.eSize - p.size)*t;
      n++;
    }
    this.geo.setDrawRange(0, n);
    this.pa.needsUpdate = true; this.ca.needsUpdate = true; this.sa.needsUpdate = true;
  }
  
  remove() { scene.remove(this.points); }
}

// Systems
const fireSystems = [];
const smokeSystems = [];
const magicSystems = [];
const explosionUpdaters = [];

// Fire emitter
function createFire(x, z) {
  const fs = new PS(500, createSprite('fire'), THREE.AdditiveBlending);
  const ss = new PS(200, createSprite('smoke'), THREE.NormalBlending);
  fs.emitPos = {x, z};
  ss.emitPos = {x: x, z: z};
  
  const light = new THREE.PointLight(0xff4400, 4, 8);
  light.position.set(x, 1.5, z);
  scene.add(light);
  fs.light = light;
  
  fireSystems.push(fs);
  smokeSystems.push(ss);
}

function createExplosion(x, z) {
  const fs = new PS(500, createSprite('fire'), THREE.AdditiveBlending);
  const ss = new PS(200, createSprite('smoke'), THREE.NormalBlending);
  const sp = new PS(300, createSprite('circle'), THREE.AdditiveBlending);
  
  for (let i = 0; i < 300; i++) {
    const dir = new THREE.Vector3(Math.random()-0.5, Math.random()-0.5, Math.random()-0.5).normalize();
    const spd = 3 + Math.random() * 12;
    fs.emit(x, 0, z, dir.x*spd, dir.y*spd, dir.z*spd, 0.5+Math.random()*0.8, 25, 0, 2,1.2,0.3, 0.5,0.05,0, 1,0, 1, 2);
  }
  for (let i = 0; i < 80; i++) {
    const a = Math.random()*Math.PI*2;
    ss.emit(x, 0, z, Math.cos(a)*1, 2+Math.random()*2, Math.sin(a)*1, 3+Math.random()*2, 30, 100, 0.3,0.3,0.3, 0.6,0.6,0.6, 0.7,0, 0.05, 0.3);
  }
  for (let i = 0; i < 200; i++) {
    const dir = new THREE.Vector3(Math.random()-0.5, Math.random()-0.5, Math.random()-0.5).normalize();
    sp.emit(x, 0, z, dir.x*15, dir.y*15, dir.z*15, 0.3+Math.random()*0.4, 8, 0, 2,1.5,0.5, 1,0.3,0, 1,0, 2, 1);
  }
  
  const flash = new THREE.PointLight(0xff8800, 50, 30);
  flash.position.set(x, 1, z);
  scene.add(flash);
  
  // Shockwave
  const swGeo = new THREE.RingGeometry(0.1, 0.5, 32);
  swGeo.rotateX(-Math.PI/2);
  const swMat = new THREE.MeshBasicMaterial({
    color: 0xffaa22, transparent: true, opacity: 1,
    side: THREE.DoubleSide, depthWrite: false, blending: THREE.AdditiveBlending
  });
  const sw = new THREE.Mesh(swGeo, swMat);
  sw.position.set(x, 0.1, z);
  scene.add(sw);
  
  let age = 0;
  const updater = (dt) => {
    age += dt;
    fs.update(dt); ss.update(dt); sp.update(dt);
    flash.intensity = Math.max(0, 50 - age * 100);
    sw.scale.setScalar(1 + age * 20);
    swMat.opacity = Math.max(0, 1 - age * 3);
    if (age > 1) scene.remove(sw);
    if (age > 5) {
      fs.remove(); ss.remove(); sp.remove();
      scene.remove(flash);
    }
    return age < 5;
  };
  explosionUpdaters.push(updater);
}

function createMagic(x, z) {
  const ms = new PS(300, createSprite('circle'), THREE.AdditiveBlending);
  ms.origin = {x, z};
  ms.radius = 2;
  ms.age = 0;
  magicSystems.push(ms);
}

// Button handlers
document.getElementById('addFire').addEventListener('click', () => {
  const x = (Math.random() - 0.5) * 10, z = (Math.random() - 0.5) * 10;
  createFire(x, z);
});

document.getElementById('explode').addEventListener('click', () => {
  const x = (Math.random() - 0.5) * 10, z = (Math.random() - 0.5) * 10;
  createExplosion(x, z);
});

document.getElementById('addSmoke').addEventListener('click', () => {
  const x = (Math.random() - 0.5) * 10, z = (Math.random() - 0.5) * 10;
  const ss = new PS(200, createSprite('smoke'), THREE.NormalBlending);
  ss.emitPos = {x, z};
  ss.isSmoke = true;
  smokeSystems.push(ss);
});

document.getElementById('addMagic').addEventListener('click', () => {
  const x = (Math.random() - 0.5) * 10, z = (Math.random() - 0.5) * 10;
  createMagic(x, z);
});

document.getElementById('clearAll').addEventListener('click', () => {
  [...fireSystems, ...smokeSystems, ...magicSystems].forEach(s => s.remove());
  fireSystems.length = 0; smokeSystems.length = 0; magicSystems.length = 0;
  explosionUpdaters.length = 0;
});

const clock = new THREE.Clock();
let emitAccum = { fire: 0, smoke: 0, magic: 0 };

function animate() {
  requestAnimationFrame(animate);
  const dt = clock.getDelta();
  const t = clock.elapsedTime;

  // Fire emission
  emitAccum.fire += dt;
  while (emitAccum.fire >= 1/30) {
    fireSystems.forEach(fs => {
      const {x, z} = fs.emitPos;
      const spread = 0.4;
      fs.emit(
        x + (Math.random()-0.5)*spread, 0, z + (Math.random()-0.5)*spread,
        (Math.random()-0.5)*0.5, 2+Math.random()*3, (Math.random()-0.5)*0.5,
        0.5+Math.random()*0.7, 20+Math.random()*15, 0,
        2,1.5,0.3, 0.8,0.05,0, 0.9,0, 0.3, 0.5
      );
      if (fs.light) {
        fs.light.intensity = 3 + Math.sin(t * 8 + x) * 1;
      }
    });
    emitAccum.fire -= 1/30;
  }
  
  // Smoke emission
  emitAccum.smoke += dt;
  while (emitAccum.smoke >= 1/5) {
    smokeSystems.forEach(ss => {
      if (!ss.emitPos) return;
      const {x, z} = ss.emitPos;
      ss.emit(x, ss.isSmoke ? 0 : 1.5, z, (Math.random()-0.5)*0.3, 0.3+Math.random()*0.4, (Math.random()-0.5)*0.3, 4+Math.random()*2, 20, 80, 0.3,0.3,0.3, 0.6,0.6,0.6, 0.4,0, 0.1, 0.2);
    });
    emitAccum.smoke -= 1/5;
  }
  
  // Magic emission
  emitAccum.magic += dt;
  while (emitAccum.magic >= 1/20) {
    magicSystems.forEach(ms => {
      ms.age += 1/20;
      const angle = ms.age * 5;
      const px = ms.origin.x + Math.cos(angle) * ms.radius;
      const pz = ms.origin.z + Math.sin(angle) * ms.radius;
      const py = 0.5 + Math.sin(ms.age * 3) * 0.5;
      ms.emit(px, py, pz, (Math.random()-0.5)*0.5, 0.5+Math.random(), (Math.random()-0.5)*0.5, 0.5+Math.random()*0.5, 5+Math.random()*8, 0, 1.5,0.5,3, 0.5,0.1,1, 1,0, 0.5, 2);
    });
    emitAccum.magic -= 1/20;
  }
  
  // Update all
  [...fireSystems, ...smokeSystems, ...magicSystems].forEach(s => s.update(dt));
  
  // Explosion updaters
  for (let i = explosionUpdaters.length-1; i >= 0; i--) {
    const cont = explosionUpdaters[i](dt);
    if (!cont) explosionUpdaters.splice(i, 1);
  }

  controls.update();
  renderer.render(scene, camera);
}

animate();

window.addEventListener('resize', () => {
  camera.aspect = innerWidth/innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(innerWidth, innerHeight);
});
</script>
</body>
</html>
```

## สรุป

Particle VFX Toolkit:
1. **ParticleSystem Base** - reusable system ที่ extend ได้
2. **Fire** - Additive blending, yellow→red gradient
3. **Smoke** - Normal blending, expand over time
4. **Explosion** - Combined: fireball + smoke + debris + shockwave
5. **Magic** - Orbit pattern + color variance
6. **Energy Shield** - Custom GLSL shader

---

## แบบฝึกหัด

1. **Easy**: สร้าง simple fire particle system ด้วย additive blending
2. **Medium**: เพิ่ม depth sorting ให้ smoke particles (camera distance)
3. **Hard**: GPU particle system ด้วย Transform Feedback (WebGL2)
4. **Challenge**: Fire simulation ด้วย velocity field บน texture
