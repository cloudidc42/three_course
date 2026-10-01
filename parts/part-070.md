# Part 70: GPGPU กับ Three.js

## บทนำ

GPGPU (General-Purpose computing on Graphics Processing Units) คือการใช้ GPU สำหรับการคำนวณทั่วไปที่ไม่ใช่แค่การ render รูปภาพ ใน Three.js เราทำ GPGPU ผ่าน fragment shaders และ render targets ซึ่งช่วยให้เราสามารถ simulate particles, fluid, และ physics ด้วยความเร็วสูงมาก

ข้อดีของ GPGPU:
- **Massive Parallelism**: GPU มีหลาย thousands of cores ทำงานพร้อมกัน
- **Memory Bandwidth**: GPU มี bandwidth สูงกว่า CPU มาก
- **Performance**: สามารถ simulate 100,000+ particles ที่ 60fps ได้

---

## 70.1 แนวคิดพื้นฐาน GPGPU

### Data Texture Concept

```
แทนที่จะเก็บ particle data ใน CPU array:
positions[0] = (x, y, z)  // particle 0
positions[1] = (x, y, z)  // particle 1
...

เราเก็บใน texture ที่ GPU อ่านได้:
texture pixel (0,0) = (x, y, z, 0)  // particle 0
texture pixel (1,0) = (x, y, z, 0)  // particle 1
texture pixel (0,1) = (x, y, z, 0)  // particle N
...

จาก N particles → texture ขนาด sqrt(N) x sqrt(N)
เช่น 65536 particles → texture 256x256
```

### Flow ของ GPGPU

```
Initial Data (CPU) → Data Texture
      ↓
[Fragment Shader] reads current state
      ↓
Compute new state (physics, velocity, etc.)
      ↓
Write new state → new Data Texture (render target)
      ↓
Visual Shader reads data texture → render particles
      ↓
swap ping/pong textures, repeat next frame
```

---

## 70.2 GPUComputationRenderer

Three.js มี helper class สำหรับ GPGPU

```javascript
// src/gpgpu/basic.js
import * as THREE from 'three';
import { GPUComputationRenderer } from 'three/examples/jsm/misc/GPUComputationRenderer.js';

// Velocity computation shader
const velocityShader = `
  uniform float uTime;
  uniform float uDelta;
  uniform float uSpeed;
  
  // texturePosition และ textureVelocity จาก previous frame
  // ถูก inject โดย GPUComputationRenderer
  
  void main() {
    vec2 uv = gl_FragCoord.xy / resolution.xy;
    
    vec4 position = texture2D(texturePosition, uv);
    vec4 velocity = texture2D(textureVelocity, uv);
    
    vec3 pos = position.xyz;
    vec3 vel = velocity.xyz;
    
    // Simple gravity
    vel.y -= 9.8 * uDelta * uSpeed;
    
    // Bounce off ground
    if (pos.y < -5.0) {
      vel.y = abs(vel.y) * 0.8;  // restitution
    }
    
    // Drag
    vel *= 0.99;
    
    // Write new velocity
    gl_FragColor = vec4(vel, position.w);
  }
`;

// Position computation shader
const positionShader = `
  uniform float uDelta;
  
  void main() {
    vec2 uv = gl_FragCoord.xy / resolution.xy;
    
    vec4 position = texture2D(texturePosition, uv);
    vec4 velocity = texture2D(textureVelocity, uv);
    
    // Integrate position
    vec3 newPos = position.xyz + velocity.xyz * uDelta;
    
    // Boundary conditions
    if (newPos.y < -5.0) newPos.y = -5.0;
    
    // Write new position
    gl_FragColor = vec4(newPos, position.w);
  }
`;

export function createGPGPUParticles(container, COUNT = 65536) {
  const SIZE = Math.sqrt(COUNT);  // texture dimension
  
  // === Setup ===
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  container.appendChild(renderer.domElement);
  
  const scene = new THREE.Scene();
  const camera = new THREE.PerspectiveCamera(60, container.clientWidth / container.clientHeight, 0.1, 1000);
  camera.position.set(0, 0, 20);
  
  // === GPUComputationRenderer ===
  const gpuCompute = new GPUComputationRenderer(SIZE, SIZE, renderer);
  
  // ตรวจสอบว่า float textures supported
  if (renderer.capabilities.isWebGL2 === false) {
    const ext = renderer.extensions.get('OES_texture_half_float');
    if (!ext) {
      console.error('Float textures not supported');
      return;
    }
  }
  
  // === Initial Data ===
  const positionTexture = gpuCompute.createTexture();
  const velocityTexture = gpuCompute.createTexture();
  
  const posData = positionTexture.image.data;
  const velData = velocityTexture.image.data;
  
  for (let i = 0; i < SIZE * SIZE; i++) {
    // Initial positions (random sphere)
    const radius = 5 + Math.random() * 5;
    const theta = Math.random() * Math.PI * 2;
    const phi = Math.acos(2 * Math.random() - 1);
    
    posData[i * 4 + 0] = radius * Math.sin(phi) * Math.cos(theta);
    posData[i * 4 + 1] = radius * Math.sin(phi) * Math.sin(theta);
    posData[i * 4 + 2] = radius * Math.cos(phi);
    posData[i * 4 + 3] = Math.random();  // life
    
    // Initial velocities
    velData[i * 4 + 0] = (Math.random() - 0.5) * 2;
    velData[i * 4 + 1] = (Math.random() - 0.5) * 2;
    velData[i * 4 + 2] = (Math.random() - 0.5) * 2;
    velData[i * 4 + 3] = Math.random();  // mass
  }
  
  // === Add Computation Variables ===
  const velocityVar = gpuCompute.addVariable(
    'textureVelocity',
    velocityShader,
    velocityTexture
  );
  
  const positionVar = gpuCompute.addVariable(
    'texturePosition',
    positionShader,
    positionTexture
  );
  
  // Dependencies
  gpuCompute.setVariableDependencies(velocityVar, [velocityVar, positionVar]);
  gpuCompute.setVariableDependencies(positionVar, [velocityVar, positionVar]);
  
  // Uniforms
  velocityVar.material.uniforms['uTime'] = { value: 0 };
  velocityVar.material.uniforms['uDelta'] = { value: 0 };
  velocityVar.material.uniforms['uSpeed'] = { value: 0.1 };
  
  positionVar.material.uniforms['uDelta'] = { value: 0 };
  
  // Initialize
  const error = gpuCompute.init();
  if (error !== null) {
    console.error(error);
    return;
  }
  
  // === Particle Display ===
  // UV coordinates สำหรับ sampling data texture
  const uvs = new Float32Array(COUNT * 2);
  for (let i = 0; i < SIZE; i++) {
    for (let j = 0; j < SIZE; j++) {
      const idx = (i * SIZE + j) * 2;
      uvs[idx + 0] = j / (SIZE - 1);
      uvs[idx + 1] = i / (SIZE - 1);
    }
  }
  
  const geometry = new THREE.BufferGeometry();
  geometry.setAttribute('position', new THREE.BufferAttribute(new Float32Array(COUNT * 3), 3));
  geometry.setAttribute('uv', new THREE.BufferAttribute(uvs, 2));
  
  // Particle shader ที่อ่าน positions จาก data texture
  const particleMaterial = new THREE.ShaderMaterial({
    uniforms: {
      texturePosition: { value: null },
      textureVelocity: { value: null },
      uSize: { value: 3 * renderer.getPixelRatio() },
    },
    vertexShader: `
      uniform sampler2D texturePosition;
      uniform sampler2D textureVelocity;
      uniform float uSize;
      
      attribute vec2 uv;
      
      varying vec3 vVelocity;
      
      void main() {
        // อ่าน position จาก data texture
        vec4 posData = texture2D(texturePosition, uv);
        vec4 velData = texture2D(textureVelocity, uv);
        
        vVelocity = velData.xyz;
        
        vec4 mvPosition = modelViewMatrix * vec4(posData.xyz, 1.0);
        gl_Position = projectionMatrix * mvPosition;
        
        // Size based on speed
        float speed = length(velData.xyz);
        gl_PointSize = uSize * (1.0 + speed * 0.1) * (1.0 / -mvPosition.z);
      }
    `,
    fragmentShader: `
      varying vec3 vVelocity;
      
      void main() {
        // Circular particle
        float d = length(gl_PointCoord - 0.5) * 2.0;
        if (d > 1.0) discard;
        
        // Color based on velocity
        float speed = length(vVelocity);
        vec3 color = mix(
          vec3(0.1, 0.3, 1.0),
          vec3(1.0, 0.5, 0.1),
          clamp(speed / 5.0, 0.0, 1.0)
        );
        
        float alpha = 1.0 - d;
        gl_FragColor = vec4(color, alpha * 0.8);
      }
    `,
    transparent: true,
    depthWrite: false,
    blending: THREE.AdditiveBlending,
  });
  
  const particles = new THREE.Points(geometry, particleMaterial);
  scene.add(particles);
  
  // === Animation ===
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const delta = clock.getDelta();
    const elapsed = clock.elapsedTime;
    
    // Update GPGPU uniforms
    velocityVar.material.uniforms['uTime'].value = elapsed;
    velocityVar.material.uniforms['uDelta'].value = delta;
    positionVar.material.uniforms['uDelta'].value = delta;
    
    // Compute next state
    gpuCompute.compute();
    
    // Update particle material uniforms
    particleMaterial.uniforms['texturePosition'].value = 
      gpuCompute.getCurrentRenderTarget(positionVar).texture;
    particleMaterial.uniforms['textureVelocity'].value = 
      gpuCompute.getCurrentRenderTarget(velocityVar).texture;
    
    renderer.render(scene, camera);
  }
  
  animate();
  
  return () => {
    cancelAnimationFrame(animId);
    geometry.dispose();
    particleMaterial.dispose();
    renderer.dispose();
  };
}
```

---

## 70.3 Boids Algorithm บน GPU

Boids simulation (flocking behavior) เป็นตัวอย่างคลาสสิกของ GPGPU

```javascript
// src/gpgpu/boids.js
import * as THREE from 'three';
import { GPUComputationRenderer } from 'three/examples/jsm/misc/GPUComputationRenderer.js';

const BOIDS_COUNT = 16384;  // 128x128
const BOIDS_SIZE = 128;

// Boids Velocity Shader
const boidsVelocityShader = `
  uniform float uTime;
  uniform float uDelta;
  
  // Boids parameters
  uniform float uSeparationRadius;  // ระยะ separation
  uniform float uAlignmentRadius;   // ระยะ alignment
  uniform float uCohesionRadius;    // ระยะ cohesion
  uniform float uSeparationForce;
  uniform float uAlignmentForce;
  uniform float uCohesionForce;
  uniform float uMaxSpeed;
  uniform float uMaxForce;
  
  const float PI = 3.1415926535;
  
  void main() {
    vec2 uv = gl_FragCoord.xy / resolution.xy;
    
    vec4 selfPos = texture2D(texturePosition, uv);
    vec4 selfVel = texture2D(textureVelocity, uv);
    
    vec3 pos = selfPos.xyz;
    vec3 vel = selfVel.xyz;
    
    // Boids forces
    vec3 separation = vec3(0.0);
    vec3 alignment = vec3(0.0);
    vec3 cohesion = vec3(0.0);
    
    float sepCount = 0.0;
    float aliCount = 0.0;
    float cohCount = 0.0;
    
    // Sample neighbors (ทุก boid ตรวจสอบ boids อื่น)
    for (float y = 0.0; y < ${BOIDS_SIZE}.0; y++) {
      for (float x = 0.0; x < ${BOIDS_SIZE}.0; x++) {
        vec2 nUv = vec2(x, y) / resolution.xy;
        
        if (nUv == uv) continue;  // ข้ามตัวเอง
        
        vec3 nPos = texture2D(texturePosition, nUv).xyz;
        vec3 nVel = texture2D(textureVelocity, nUv).xyz;
        
        vec3 diff = pos - nPos;
        float dist = length(diff);
        
        // Separation
        if (dist < uSeparationRadius && dist > 0.0) {
          separation += normalize(diff) / dist;
          sepCount++;
        }
        
        // Alignment
        if (dist < uAlignmentRadius) {
          alignment += nVel;
          aliCount++;
        }
        
        // Cohesion
        if (dist < uCohesionRadius) {
          cohesion += nPos;
          cohCount++;
        }
      }
    }
    
    vec3 accel = vec3(0.0);
    
    if (sepCount > 0.0) {
      separation /= sepCount;
      if (length(separation) > 0.0) {
        separation = normalize(separation) * uMaxSpeed - vel;
        separation = clamp(separation, -uMaxForce, uMaxForce);
      }
      accel += separation * uSeparationForce;
    }
    
    if (aliCount > 0.0) {
      alignment /= aliCount;
      if (length(alignment) > 0.0) {
        alignment = normalize(alignment) * uMaxSpeed - vel;
        alignment = clamp(alignment, -uMaxForce, uMaxForce);
      }
      accel += alignment * uAlignmentForce;
    }
    
    if (cohCount > 0.0) {
      cohesion /= cohCount;
      vec3 toCohesion = cohesion - pos;
      if (length(toCohesion) > 0.0) {
        toCohesion = normalize(toCohesion) * uMaxSpeed - vel;
        toCohesion = clamp(toCohesion, -uMaxForce, uMaxForce);
      }
      accel += toCohesion * uCohesionForce;
    }
    
    // Apply acceleration
    vec3 newVel = vel + accel * uDelta;
    
    // Limit speed
    float speed = length(newVel);
    if (speed > uMaxSpeed) {
      newVel = normalize(newVel) * uMaxSpeed;
    }
    
    // Boundary wrapping
    // (handled in position shader)
    
    gl_FragColor = vec4(newVel, selfVel.w);
  }
`;

const boidsPositionShader = `
  uniform float uDelta;
  uniform float uBoundSize;
  
  void main() {
    vec2 uv = gl_FragCoord.xy / resolution.xy;
    
    vec4 posData = texture2D(texturePosition, uv);
    vec4 velData = texture2D(textureVelocity, uv);
    
    vec3 newPos = posData.xyz + velData.xyz * uDelta;
    
    // Wrap around boundaries
    float half = uBoundSize * 0.5;
    if (newPos.x > half) newPos.x -= uBoundSize;
    if (newPos.x < -half) newPos.x += uBoundSize;
    if (newPos.y > half) newPos.y -= uBoundSize;
    if (newPos.y < -half) newPos.y += uBoundSize;
    if (newPos.z > half) newPos.z -= uBoundSize;
    if (newPos.z < -half) newPos.z += uBoundSize;
    
    gl_FragColor = vec4(newPos, posData.w);
  }
`;

export function createBoidsDemo(container) {
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  container.appendChild(renderer.domElement);
  
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#050510');
  
  const camera = new THREE.PerspectiveCamera(
    60, container.clientWidth / container.clientHeight, 0.1, 1000
  );
  camera.position.set(0, 0, 50);
  
  // GPGPU setup
  const gpuCompute = new GPUComputationRenderer(BOIDS_SIZE, BOIDS_SIZE, renderer);
  
  // Initial data
  const posTexture = gpuCompute.createTexture();
  const velTexture = gpuCompute.createTexture();
  
  const posData = posTexture.image.data;
  const velData = velTexture.image.data;
  
  const BOUND = 30;
  for (let i = 0; i < BOIDS_SIZE * BOIDS_SIZE; i++) {
    // Random positions in cube
    posData[i * 4 + 0] = (Math.random() - 0.5) * BOUND;
    posData[i * 4 + 1] = (Math.random() - 0.5) * BOUND;
    posData[i * 4 + 2] = (Math.random() - 0.5) * BOUND;
    posData[i * 4 + 3] = 1.0;
    
    // Random initial velocity
    const angle = Math.random() * Math.PI * 2;
    velData[i * 4 + 0] = Math.cos(angle) * 5;
    velData[i * 4 + 1] = (Math.random() - 0.5) * 5;
    velData[i * 4 + 2] = Math.sin(angle) * 5;
    velData[i * 4 + 3] = 1.0;
  }
  
  // Add variables
  const velVar = gpuCompute.addVariable('textureVelocity', boidsVelocityShader, velTexture);
  const posVar = gpuCompute.addVariable('texturePosition', boidsPositionShader, posTexture);
  
  gpuCompute.setVariableDependencies(velVar, [posVar, velVar]);
  gpuCompute.setVariableDependencies(posVar, [posVar, velVar]);
  
  // Velocity uniforms
  const velUniforms = velVar.material.uniforms;
  velUniforms['uTime'] = { value: 0 };
  velUniforms['uDelta'] = { value: 0 };
  velUniforms['uSeparationRadius'] = { value: 2.0 };
  velUniforms['uAlignmentRadius'] = { value: 5.0 };
  velUniforms['uCohesionRadius'] = { value: 8.0 };
  velUniforms['uSeparationForce'] = { value: 1.5 };
  velUniforms['uAlignmentForce'] = { value: 1.0 };
  velUniforms['uCohesionForce'] = { value: 0.8 };
  velUniforms['uMaxSpeed'] = { value: 8.0 };
  velUniforms['uMaxForce'] = { value: 0.3 };
  
  // Position uniforms
  const posUniforms = posVar.material.uniforms;
  posUniforms['uDelta'] = { value: 0 };
  posUniforms['uBoundSize'] = { value: BOUND };
  
  const error = gpuCompute.init();
  if (error) console.error(error);
  
  // Boid display
  const uvs = new Float32Array(BOIDS_COUNT * 2);
  for (let i = 0; i < BOIDS_SIZE; i++) {
    for (let j = 0; j < BOIDS_SIZE; j++) {
      const idx = (i * BOIDS_SIZE + j) * 2;
      uvs[idx] = j / (BOIDS_SIZE - 1);
      uvs[idx + 1] = i / (BOIDS_SIZE - 1);
    }
  }
  
  const geometry = new THREE.BufferGeometry();
  geometry.setAttribute('position', new THREE.BufferAttribute(new Float32Array(BOIDS_COUNT * 3), 3));
  geometry.setAttribute('uv', new THREE.BufferAttribute(uvs, 2));
  
  const material = new THREE.ShaderMaterial({
    uniforms: {
      texturePosition: { value: null },
      textureVelocity: { value: null },
      uSize: { value: 2.0 },
    },
    vertexShader: `
      uniform sampler2D texturePosition;
      uniform sampler2D textureVelocity;
      uniform float uSize;
      
      attribute vec2 uv;
      
      varying vec3 vVelocity;
      
      void main() {
        vec4 posData = texture2D(texturePosition, uv);
        vec4 velData = texture2D(textureVelocity, uv);
        
        vVelocity = velData.xyz;
        
        vec4 mvPosition = modelViewMatrix * vec4(posData.xyz, 1.0);
        gl_Position = projectionMatrix * mvPosition;
        gl_PointSize = uSize * (100.0 / -mvPosition.z);
      }
    `,
    fragmentShader: `
      varying vec3 vVelocity;
      
      void main() {
        float d = length(gl_PointCoord - 0.5) * 2.0;
        if (d > 1.0) discard;
        
        vec3 dir = normalize(vVelocity);
        vec3 color = dir * 0.5 + 0.5;  // Direction as color
        
        gl_FragColor = vec4(color, (1.0 - d) * 0.9);
      }
    `,
    transparent: true,
    depthWrite: false,
    blending: THREE.AdditiveBlending,
  });
  
  const points = new THREE.Points(geometry, material);
  scene.add(points);
  
  // Boundary box
  const boxHelper = new THREE.BoxHelper(
    new THREE.Mesh(new THREE.BoxGeometry(BOUND, BOUND, BOUND)),
    '#4ecdc4'
  );
  scene.add(boxHelper);
  
  // Animation
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const delta = Math.min(clock.getDelta(), 0.1);
    const elapsed = clock.elapsedTime;
    
    velUniforms['uTime'].value = elapsed;
    velUniforms['uDelta'].value = delta;
    posUniforms['uDelta'].value = delta;
    
    gpuCompute.compute();
    
    material.uniforms['texturePosition'].value =
      gpuCompute.getCurrentRenderTarget(posVar).texture;
    material.uniforms['textureVelocity'].value =
      gpuCompute.getCurrentRenderTarget(velVar).texture;
    
    // Orbit camera
    camera.position.x = Math.cos(elapsed * 0.1) * 60;
    camera.position.z = Math.sin(elapsed * 0.1) * 60;
    camera.lookAt(0, 0, 0);
    
    renderer.render(scene, camera);
  }
  
  animate();
  
  return () => {
    cancelAnimationFrame(animId);
    geometry.dispose();
    material.dispose();
    renderer.dispose();
  };
}
```

---

## 70.4 Manual Ping-Pong GPGPU

สำหรับกรณีที่ต้องการ control มากกว่า GPUComputationRenderer

```javascript
// src/gpgpu/manualPingPong.js
import * as THREE from 'three';

class ManualGPGPU {
  constructor(renderer, size) {
    this.renderer = renderer;
    this.size = size;
    this.variables = new Map();
    this.fsScene = new THREE.Scene();
    this.fsCamera = new THREE.OrthographicCamera(-1, 1, 1, -1, 0, 1);
    this.fsQuad = new THREE.Mesh(new THREE.PlaneGeometry(2, 2));
    this.fsScene.add(this.fsQuad);
  }
  
  createDataTexture(fillFn) {
    const texture = new THREE.DataTexture(
      new Float32Array(this.size * this.size * 4),
      this.size,
      this.size,
      THREE.RGBAFormat,
      THREE.FloatType
    );
    
    if (fillFn) {
      for (let i = 0; i < this.size * this.size; i++) {
        fillFn(texture.image.data, i, this.size);
      }
    }
    
    texture.needsUpdate = true;
    return texture;
  }
  
  createRenderTarget() {
    return new THREE.WebGLRenderTarget(this.size, this.size, {
      minFilter: THREE.NearestFilter,
      magFilter: THREE.NearestFilter,
      format: THREE.RGBAFormat,
      type: THREE.FloatType,
      stencilBuffer: false,
    });
  }
  
  addVariable(name, shader, initialTexture) {
    const material = new THREE.ShaderMaterial({
      vertexShader: `
        varying vec2 vUv;
        void main() {
          vUv = uv;
          gl_Position = vec4(position, 1.0);
        }
      `,
      fragmentShader: shader,
      uniforms: {
        resolution: { value: new THREE.Vector2(this.size, this.size) }
      },
    });
    
    const targetA = this.createRenderTarget();
    const targetB = this.createRenderTarget();
    
    // Copy initial texture ไปยัง targetA
    this.fsQuad.material = new THREE.MeshBasicMaterial({ map: initialTexture });
    this.renderer.setRenderTarget(targetA);
    this.renderer.render(this.fsScene, this.fsCamera);
    
    this.variables.set(name, { material, targetA, targetB, name });
    return this.variables.get(name);
  }
  
  compute() {
    this.variables.forEach(variable => {
      this.fsQuad.material = variable.material;
      
      this.renderer.setRenderTarget(variable.targetB);
      this.renderer.render(this.fsScene, this.fsCamera);
      this.renderer.setRenderTarget(null);
      
      // Swap ping-pong
      [variable.targetA, variable.targetB] = [variable.targetB, variable.targetA];
    });
  }
  
  getCurrentTexture(variableName) {
    return this.variables.get(variableName)?.targetA.texture;
  }
  
  dispose() {
    this.variables.forEach(v => {
      v.targetA.dispose();
      v.targetB.dispose();
      v.material.dispose();
    });
  }
}

export { ManualGPGPU };
```

---

## 70.5 Fluid Simulation (Navier-Stokes)

```javascript
// src/gpgpu/fluid.js - Simplified Navier-Stokes
import * as THREE from 'three';
import { GPUComputationRenderer } from 'three/examples/jsm/misc/GPUComputationRenderer.js';

const FLUID_SIZE = 256;

// Advection shader - เลื่อน fluid ตาม velocity field
const advectionShader = `
  uniform sampler2D uVelocity;
  uniform float uDelta;
  uniform float uDissipation;
  
  void main() {
    vec2 uv = gl_FragCoord.xy / resolution.xy;
    vec2 texel = 1.0 / resolution.xy;
    
    // Semi-Lagrangian advection
    vec2 vel = texture2D(uVelocity, uv).xy;
    vec2 prevPos = uv - vel * uDelta * texel;
    
    // Bilinear interpolation
    vec4 result = texture2D(textureAdvect, prevPos);
    
    gl_FragColor = result * uDissipation;
  }
`;

// Divergence shader
const divergenceShader = `
  uniform sampler2D uVelocity;
  
  void main() {
    vec2 uv = gl_FragCoord.xy / resolution.xy;
    vec2 texel = 1.0 / resolution.xy;
    
    float left  = texture2D(uVelocity, uv - vec2(texel.x, 0.0)).x;
    float right = texture2D(uVelocity, uv + vec2(texel.x, 0.0)).x;
    float bottom= texture2D(uVelocity, uv - vec2(0.0, texel.y)).y;
    float top   = texture2D(uVelocity, uv + vec2(0.0, texel.y)).y;
    
    float div = 0.5 * (right - left + top - bottom);
    gl_FragColor = vec4(div, 0.0, 0.0, 1.0);
  }
`;

// Pressure Jacobi iteration
const pressureShader = `
  uniform sampler2D uDivergence;
  
  void main() {
    vec2 uv = gl_FragCoord.xy / resolution.xy;
    vec2 texel = 1.0 / resolution.xy;
    
    float left  = texture2D(texturePressure, uv - vec2(texel.x, 0.0)).x;
    float right = texture2D(texturePressure, uv + vec2(texel.x, 0.0)).x;
    float bottom= texture2D(texturePressure, uv - vec2(0.0, texel.y)).x;
    float top   = texture2D(texturePressure, uv + vec2(0.0, texel.y)).x;
    
    float div = texture2D(uDivergence, uv).x;
    
    float pressure = (left + right + bottom + top - div) * 0.25;
    gl_FragColor = vec4(pressure, 0.0, 0.0, 1.0);
  }
`;

// Gradient subtraction - project velocity
const gradientSubtractShader = `
  uniform sampler2D uPressure;
  
  void main() {
    vec2 uv = gl_FragCoord.xy / resolution.xy;
    vec2 texel = 1.0 / resolution.xy;
    
    vec2 vel = texture2D(textureVelocity, uv).xy;
    
    float left  = texture2D(uPressure, uv - vec2(texel.x, 0.0)).x;
    float right = texture2D(uPressure, uv + vec2(texel.x, 0.0)).x;
    float bottom= texture2D(uPressure, uv - vec2(0.0, texel.y)).x;
    float top   = texture2D(uPressure, uv + vec2(0.0, texel.y)).x;
    
    vec2 grad = vec2(right - left, top - bottom) * 0.5;
    vel -= grad;
    
    gl_FragColor = vec4(vel, 0.0, 1.0);
  }
`;

// Add external force (mouse input)
const addForceShader = `
  uniform vec2 uPoint;
  uniform vec2 uForce;
  uniform float uRadius;
  
  void main() {
    vec2 uv = gl_FragCoord.xy / resolution.xy;
    
    vec2 vel = texture2D(textureVelocity, uv).xy;
    
    float dist = distance(uv, uPoint);
    float influence = exp(-dist * dist / (2.0 * uRadius * uRadius));
    
    vel += uForce * influence;
    
    gl_FragColor = vec4(vel, 0.0, 1.0);
  }
`;

// Visualization shader
const visualizeShader = `
  uniform sampler2D uDye;
  uniform sampler2D uVelocity;
  
  void main() {
    vec2 uv = gl_FragCoord.xy / resolution.xy;
    
    vec4 dye = texture2D(uDye, uv);
    vec2 vel = texture2D(uVelocity, uv).xy;
    
    float speed = length(vel);
    
    // Color: dye + velocity direction
    vec3 color = dye.rgb;
    
    // Add velocity visualization
    vec3 velColor = normalize(vec3(vel + 0.5, 0.5));
    color = mix(color, velColor, 0.3);
    
    gl_FragColor = vec4(color, 1.0);
  }
`;

export function createFluidSimulation(container) {
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  container.appendChild(renderer.domElement);
  
  // Full-screen display
  const displayScene = new THREE.Scene();
  const displayCamera = new THREE.OrthographicCamera(-1, 1, 1, -1, 0, 1);
  const displayMat = new THREE.ShaderMaterial({
    uniforms: {
      uDye: { value: null },
      uVelocity: { value: null },
      resolution: { value: new THREE.Vector2(FLUID_SIZE, FLUID_SIZE) }
    },
    vertexShader: `
      varying vec2 vUv;
      void main() { vUv = uv; gl_Position = vec4(position, 1.0); }
    `,
    fragmentShader: visualizeShader,
  });
  displayScene.add(new THREE.Mesh(new THREE.PlaneGeometry(2, 2), displayMat));
  
  // GPUComputationRenderer setup
  const gpuCompute = new GPUComputationRenderer(FLUID_SIZE, FLUID_SIZE, renderer);
  
  // Initialize textures
  const velocityTexture = gpuCompute.createTexture();
  const pressureTexture = gpuCompute.createTexture();
  const dyeTexture = gpuCompute.createTexture();
  
  // Initialize dye with some color
  const dyeData = dyeTexture.image.data;
  for (let i = 0; i < FLUID_SIZE * FLUID_SIZE; i++) {
    const x = (i % FLUID_SIZE) / FLUID_SIZE;
    const y = Math.floor(i / FLUID_SIZE) / FLUID_SIZE;
    const d = Math.sqrt((x - 0.5) ** 2 + (y - 0.5) ** 2);
    dyeData[i * 4] = d < 0.1 ? 1.0 : 0.0;
    dyeData[i * 4 + 1] = d < 0.15 && d > 0.1 ? 1.0 : 0.0;
    dyeData[i * 4 + 2] = d < 0.05 ? 1.0 : 0.0;
    dyeData[i * 4 + 3] = 1.0;
  }
  
  const velVar = gpuCompute.addVariable('textureVelocity', addForceShader, velocityTexture);
  velVar.material.uniforms['uPoint'] = { value: new THREE.Vector2(0.5, 0.5) };
  velVar.material.uniforms['uForce'] = { value: new THREE.Vector2(0, 0) };
  velVar.material.uniforms['uRadius'] = { value: 0.01 };
  
  const pressureVar = gpuCompute.addVariable('texturePressure', pressureShader, pressureTexture);
  pressureVar.material.uniforms['uDivergence'] = { value: null };
  
  const dyeVar = gpuCompute.addVariable('textureAdvect', advectionShader, dyeTexture);
  dyeVar.material.uniforms['uVelocity'] = { value: null };
  dyeVar.material.uniforms['uDelta'] = { value: 0.016 };
  dyeVar.material.uniforms['uDissipation'] = { value: 0.995 };
  
  gpuCompute.setVariableDependencies(velVar, [velVar]);
  gpuCompute.setVariableDependencies(pressureVar, [pressureVar]);
  gpuCompute.setVariableDependencies(dyeVar, [dyeVar]);
  
  const initError = gpuCompute.init();
  if (initError) console.error(initError);
  
  // Mouse interaction
  const mouse = new THREE.Vector2(0.5, 0.5);
  const lastMouse = new THREE.Vector2(0.5, 0.5);
  
  container.addEventListener('mousemove', (e) => {
    const rect = container.getBoundingClientRect();
    mouse.x = (e.clientX - rect.left) / rect.width;
    mouse.y = 1 - (e.clientY - rect.top) / rect.height;
  });
  
  // Animation
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const delta = clock.getDelta();
    
    // Apply mouse force
    const force = new THREE.Vector2(mouse.x - lastMouse.x, mouse.y - lastMouse.y);
    force.multiplyScalar(1000);
    force.clampLength(0, 10);
    
    velVar.material.uniforms['uPoint'].value.copy(mouse);
    velVar.material.uniforms['uForce'].value.copy(force);
    
    lastMouse.copy(mouse);
    
    // Update uniforms
    dyeVar.material.uniforms['uDelta'].value = delta;
    dyeVar.material.uniforms['uVelocity'].value = 
      gpuCompute.getCurrentRenderTarget(velVar).texture;
    
    gpuCompute.compute();
    
    // Display
    displayMat.uniforms['uDye'].value = 
      gpuCompute.getCurrentRenderTarget(dyeVar).texture;
    displayMat.uniforms['uVelocity'].value = 
      gpuCompute.getCurrentRenderTarget(velVar).texture;
    
    renderer.setRenderTarget(null);
    renderer.render(displayScene, displayCamera);
  }
  
  animate();
  
  return () => {
    cancelAnimationFrame(animId);
    renderer.dispose();
  };
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: 100K Particle Fountain
สร้าง particle fountain ด้วย GPGPU:
- 131,072 particles (512x256 texture)
- Gravity + wind forces
- Color gradient ตาม velocity
- Interactive mouse repulsion

### แบบฝึกหัดที่ 2: Galaxy Simulation
สร้าง galaxy simulation:
- 32,768 stars ด้วย GPGPU
- Gravitational attraction
- Spiral arm formation
- Multiple galaxies interaction

### แบบฝึกหัดที่ 3: Cloth Simulation
สร้าง cloth simulation:
- Grid ของ particles
- Spring constraints (stretch, shear, bend)
- Wind force
- Collision detection กับ sphere

### แบบฝึกหัดที่ 4: Reaction-Diffusion
สร้าง reaction-diffusion simulation (Gray-Scott):
- 2 chemical species A, B
- Diffusion + reaction rates
- Different patterns (spots, stripes, waves)
- Color visualization

### แบบฝึกหัดที่ 5: Optimized Boids
ปรับ Boids simulation ให้เร็วขึ้น:
- Spatial hashing ใน texture
- Reduce neighbor search radius
- LOD (Level of Detail)
- ทำให้ได้ 100K boids ที่ 60fps

---

## สรุป

ในบทนี้เราเรียนรู้:
1. **GPGPU Concept** - ใช้ GPU สำหรับ general computation
2. **Data Textures** - เก็บ simulation data ใน textures
3. **GPUComputationRenderer** - Three.js helper สำหรับ GPGPU
4. **Boids** - Flocking algorithm บน GPU
5. **Manual Ping-Pong** - Custom GPGPU implementation
6. **Fluid Simulation** - Navier-Stokes บน GPU

บทถัดไปเราจะเรียน **Audio ใน Three.js**
