# Part 81: WebGPU กับ Three.js

## บทนำ: ยุคใหม่ของ Graphics API

WebGPU คือ Web Graphics API รุ่นใหม่ล่าสุดที่มาแทน WebGL โดยมีการออกแบบที่ทันสมัยกว่า ให้ประสิทธิภาพสูงกว่า และรองรับ Compute Shaders แบบ Native สำหรับ Three.js ตั้งแต่เวอร์ชัน r152+ มีการรองรับ WebGPU ผ่าน `WebGPURenderer` ซึ่งช่วยให้นักพัฒนาสามารถใช้งาน WebGPU ได้โดยไม่ต้องเปลี่ยน API ของ Three.js มากนัก

ในบทนี้เราจะเรียนรู้:
- การตั้งค่า WebGPURenderer
- ความแตกต่างระหว่าง WebGPU และ WebGL
- การใช้งาน Compute Shaders
- ระบบ Material ใหม่ (NodeMaterial, TSL)
- การย้ายจาก WebGL มา WebGPU

---

## 1. WebGPU vs WebGL: ความแตกต่างหลัก

### WebGL (รุ่นเก่า)
```
- ออกแบบมาจาก OpenGL ES 2.0/3.0
- State machine ที่ซับซ้อน
- ไม่มี Compute Shader native
- Single thread rendering
- Driver overhead สูง
- Implicit resource management
```

### WebGPU (รุ่นใหม่)
```
- ออกแบบมาจาก Vulkan/Metal/D3D12
- Explicit resource management
- Compute Shader native support
- Multi-threading friendly
- Lower driver overhead
- Better GPU memory control
- Pipeline objects แบบ immutable
- Shader ใช้ WGSL (WebGPU Shading Language)
```

### ตารางเปรียบเทียบ

| Feature | WebGL | WebGPU |
|---------|-------|--------|
| API ที่ใกล้เคียง | OpenGL ES | Vulkan/Metal/D3D12 |
| Compute Shaders | ต้องใช้ Transform Feedback | Native support |
| Multi-threading | จำกัด | ดีกว่า |
| Shader Language | GLSL | WGSL + GLSL (ผ่าน SPIRV) |
| Resource Management | Implicit | Explicit |
| Performance | ดี | ดีกว่า (สำหรับ complex scenes) |
| Browser Support | ทุก browser | Chrome 113+, Firefox (experimental) |

---

## 2. การตั้งค่า WebGPURenderer

### การติดตั้งและ Import

```javascript
// ต้องใช้ Three.js r152+ หรือใหม่กว่า
// และต้องใช้ module version ไม่ใช่ CDN แบบเก่า

import * as THREE from 'three';
import WebGPURenderer from 'three/addons/renderers/common/Renderer.js';
// หรือ (ขึ้นกับเวอร์ชัน)
import { WebGPURenderer } from 'three/webgpu';
```

### การ Setup พื้นฐาน

```javascript
// ตรวจสอบว่า Browser รองรับ WebGPU หรือไม่
async function checkWebGPUSupport() {
  if (!navigator.gpu) {
    console.log('WebGPU ไม่รองรับใน browser นี้');
    return false;
  }
  
  const adapter = await navigator.gpu.requestAdapter();
  if (!adapter) {
    console.log('ไม่พบ WebGPU adapter');
    return false;
  }
  
  const device = await adapter.requestDevice();
  console.log('WebGPU รองรับ!', device);
  return true;
}

// การสร้าง WebGPURenderer
async function initWebGPU() {
  const canvas = document.getElementById('canvas');
  
  // ตรวจสอบการรองรับ
  const isSupported = await checkWebGPUSupport();
  
  let renderer;
  
  if (isSupported) {
    // ใช้ WebGPURenderer
    renderer = new THREE.WebGPURenderer({
      canvas: canvas,
      antialias: true,
      // WebGPU specific options
      powerPreference: 'high-performance',
    });
    
    console.log('ใช้ WebGPU Renderer');
  } else {
    // Fallback ไป WebGL
    renderer = new THREE.WebGLRenderer({
      canvas: canvas,
      antialias: true,
    });
    
    console.log('Fallback ไป WebGL Renderer');
  }
  
  // ต้อง initialize async
  await renderer.init();
  
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(window.devicePixelRatio);
  
  return renderer;
}
```

### ตัวอย่างเต็มรูปแบบ: WebGPU Scene Setup

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>WebGPU Three.js Demo</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { overflow: hidden; background: #000; }
    canvas { display: block; }
    #info {
      position: fixed;
      top: 10px;
      left: 10px;
      color: white;
      font-family: monospace;
      background: rgba(0,0,0,0.5);
      padding: 10px;
      border-radius: 5px;
    }
  </style>
</head>
<body>
  <canvas id="canvas"></canvas>
  <div id="info">กำลังโหลด...</div>
  
  <script type="module">
    import * as THREE from 'https://unpkg.com/three@0.160.0/build/three.module.js';
    
    // สำหรับ WebGPU ต้องใช้ import path พิเศษ
    // import { WebGPURenderer } from 'three/webgpu';
    
    const infoEl = document.getElementById('info');
    
    // Feature Detection
    async function detectWebGPU() {
      if (!('gpu' in navigator)) {
        return { supported: false, reason: 'navigator.gpu ไม่มี' };
      }
      
      try {
        const adapter = await navigator.gpu.requestAdapter({
          powerPreference: 'high-performance'
        });
        
        if (!adapter) {
          return { supported: false, reason: 'ไม่พบ GPU adapter' };
        }
        
        // ดูข้อมูล adapter
        const info = adapter.info;
        console.log('GPU Info:', {
          vendor: info.vendor,
          architecture: info.architecture,
          device: info.device,
          description: info.description
        });
        
        // ดู features ที่รองรับ
        const features = [];
        for (const feature of adapter.features) {
          features.push(feature);
        }
        console.log('Supported features:', features);
        
        // ดู limits
        const limits = adapter.limits;
        console.log('Max texture size:', limits.maxTextureDimension2D);
        console.log('Max compute workgroups:', limits.maxComputeWorkgroupsPerDimension);
        
        const device = await adapter.requestDevice({
          requiredFeatures: [],
          requiredLimits: {
            maxStorageBufferBindingSize: limits.maxStorageBufferBindingSize
          }
        });
        
        return { 
          supported: true, 
          adapter,
          device,
          features,
          limits: {
            maxTexture: limits.maxTextureDimension2D,
            maxCompute: limits.maxComputeWorkgroupsPerDimension
          }
        };
      } catch (error) {
        return { supported: false, reason: error.message };
      }
    }
    
    async function main() {
      const canvas = document.getElementById('canvas');
      
      // ตรวจสอบ WebGPU
      const gpuInfo = await detectWebGPU();
      
      if (gpuInfo.supported) {
        infoEl.textContent = `WebGPU: รองรับ | GPU Max Texture: ${gpuInfo.limits.maxTexture}`;
        infoEl.style.color = '#00ff00';
      } else {
        infoEl.textContent = `WebGPU: ไม่รองรับ (${gpuInfo.reason}) | ใช้ WebGL แทน`;
        infoEl.style.color = '#ff8800';
      }
      
      // สร้าง Scene
      const scene = new THREE.Scene();
      const camera = new THREE.PerspectiveCamera(
        75, 
        window.innerWidth / window.innerHeight, 
        0.1, 
        1000
      );
      camera.position.set(0, 0, 5);
      
      // Renderer (WebGL fallback ในตัวอย่างนี้)
      const renderer = new THREE.WebGLRenderer({
        canvas,
        antialias: true
      });
      renderer.setSize(window.innerWidth, window.innerHeight);
      renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
      renderer.shadowMap.enabled = true;
      renderer.shadowMap.type = THREE.PCFSoftShadowMap;
      
      // เพิ่ม Objects
      const geometry = new THREE.TorusKnotGeometry(1, 0.3, 128, 32);
      const material = new THREE.MeshStandardMaterial({
        color: 0x0088ff,
        metalness: 0.8,
        roughness: 0.2,
        envMapIntensity: 1.0
      });
      
      const mesh = new THREE.Mesh(geometry, material);
      scene.add(mesh);
      
      // Lighting
      const ambientLight = new THREE.AmbientLight(0xffffff, 0.3);
      scene.add(ambientLight);
      
      const pointLight = new THREE.PointLight(0xffffff, 2, 20);
      pointLight.position.set(3, 3, 3);
      scene.add(pointLight);
      
      const pointLight2 = new THREE.PointLight(0xff4400, 1.5, 20);
      pointLight2.position.set(-3, -2, 2);
      scene.add(pointLight2);
      
      // Resize handler
      window.addEventListener('resize', () => {
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(window.innerWidth, window.innerHeight);
      });
      
      // Animation Loop
      const clock = new THREE.Clock();
      
      function animate() {
        requestAnimationFrame(animate);
        
        const time = clock.getElapsedTime();
        
        mesh.rotation.x = time * 0.5;
        mesh.rotation.y = time * 0.3;
        
        pointLight.position.x = Math.sin(time) * 3;
        pointLight.position.z = Math.cos(time) * 3;
        
        renderer.render(scene, camera);
      }
      
      animate();
    }
    
    main().catch(console.error);
  </script>
</body>
</html>
```

---

## 3. NodeMaterial และ TSL (Three.js Shading Language)

### แนวคิดของ Node-based Materials

TSL หรือ Three.js Shading Language เป็นระบบ Material ใหม่ที่ใช้ Node Graph แทนการเขียน GLSL/WGSL โดยตรง มีข้อดีคือ:

- **Cross-renderer**: ทำงานได้ทั้งบน WebGL และ WebGPU
- **Composable**: สามารถ mix และ match nodes ได้ง่าย
- **Type-safe**: มีระบบ type checking
- **Reusable**: สร้าง custom node functions ได้

### TSL Nodes พื้นฐาน

```javascript
// Import TSL nodes
import { 
  MeshStandardNodeMaterial,
  // Position nodes
  positionLocal,
  positionWorld, 
  positionView,
  // Normal nodes
  normalLocal,
  normalWorld,
  normalView,
  // UV nodes
  uv,
  uv2,
  // Color/texture nodes
  color,
  texture,
  // Math nodes
  add,
  sub,
  mul,
  div,
  sin,
  cos,
  abs,
  clamp,
  mix,
  smoothstep,
  step,
  dot,
  cross,
  normalize,
  length,
  // Uniform nodes
  uniform,
  // Time
  timerLocal,
  timerGlobal,
  // Custom function
  tslFn,
  vec2, 
  vec3, 
  vec4,
  float,
  // Special
  varying,
  attribute
} from 'three/addons/nodes/Nodes.js';
// หรือ
import * as TSL from 'three/tsl';
```

### ตัวอย่าง TSL Material พื้นฐาน

```javascript
import * as THREE from 'three';
import { 
  MeshStandardNodeMaterial,
  color,
  uv,
  sin,
  cos,
  mul,
  add,
  uniform,
  timerGlobal,
  vec3,
  mix,
  smoothstep
} from 'three/tsl';

// Material ที่มีสีแบบ gradient animated
function createAnimatedGradientMaterial() {
  const material = new MeshStandardNodeMaterial();
  
  // สร้าง uniform สำหรับควบคุมจาก JavaScript
  const timeUniform = uniform(0.0);
  const colorA = uniform(new THREE.Color(0xff0000));
  const colorB = uniform(new THREE.Color(0x0000ff));
  const speed = uniform(1.0);
  
  // คำนวณ gradient จาก UV coordinates
  const uvNode = uv();  // ค่า UV ของ vertex
  
  // Animated gradient
  const animTime = mul(timerGlobal(), speed);
  const wave = sin(add(mul(uvNode.y, 5.0), animTime));  // sin wave
  
  // Mix ระหว่างสองสี
  const t = smoothstep(0.0, 1.0, add(wave, 0.5));
  const finalColor = mix(colorA, colorB, t);
  
  material.colorNode = finalColor;
  
  // เก็บ reference สำหรับ update
  material.userData.timeUniform = timeUniform;
  material.userData.colorA = colorA;
  material.userData.colorB = colorB;
  material.userData.speed = speed;
  
  return material;
}

// การใช้งาน
const geometry = new THREE.SphereGeometry(1, 64, 64);
const material = createAnimatedGradientMaterial();
const mesh = new THREE.Mesh(geometry, material);
scene.add(mesh);

// Update ใน animation loop
function animate() {
  requestAnimationFrame(animate);
  
  // อัพเดท time uniform
  // (timerGlobal จะ update อัตโนมัติ แต่ถ้าใช้ manual time:)
  // material.userData.timeUniform.value = Date.now() / 1000;
  
  renderer.render(scene, camera);
}
```

---

## 4. Compute Shaders กับ WebGPU

Compute Shaders เป็น feature ใหม่ที่ WebGPU รองรับ native ช่วยให้สามารถรัน general-purpose computation บน GPU ได้

### แนวคิด Compute Shader

```
CPU -> ส่งข้อมูล -> GPU Buffer -> Compute Shader -> ผลลัพธ์
                                    |
                                    v
                            [workgroup 1] [workgroup 2] [workgroup 3] ...
                               [thread]    [thread]     [thread]
                               [thread]    [thread]     [thread]
                               [thread]    [thread]     [thread]
```

### Compute Shader Particle System ด้วย TSL

```javascript
import * as THREE from 'three';
import {
  WebGPURenderer,
  StorageBufferNode,
  StorageInstancedBufferAttribute,
  uniform,
  storage,
  instanceIndex,
  float,
  vec3,
  vec4,
  tslFn,
  If,
  Loop,
  uint,
  sqrt,
  dot,
  normalize,
  length,
  abs,
  sin,
  cos,
  add,
  sub,
  mul,
  div,
  mod,
  clamp,
  mix,
  smoothstep
} from 'three/webgpu';

// จำนวน particles
const PARTICLE_COUNT = 100000;

// สร้าง Buffers สำหรับ particle data
function createParticleBuffers() {
  // Position buffer: vec4 (x, y, z, w=lifetime)
  const positionBuffer = new THREE.StorageInstancedBufferAttribute(
    new Float32Array(PARTICLE_COUNT * 4),
    4  // vec4
  );
  
  // Velocity buffer: vec4 (vx, vy, vz, speed)
  const velocityBuffer = new THREE.StorageInstancedBufferAttribute(
    new Float32Array(PARTICLE_COUNT * 4),
    4  // vec4
  );
  
  // สุ่มค่าเริ่มต้น
  for (let i = 0; i < PARTICLE_COUNT; i++) {
    const i4 = i * 4;
    
    // Position สุ่มใน sphere
    const theta = Math.random() * Math.PI * 2;
    const phi = Math.acos(2 * Math.random() - 1);
    const r = Math.cbrt(Math.random()) * 5;
    
    positionBuffer.array[i4 + 0] = r * Math.sin(phi) * Math.cos(theta);
    positionBuffer.array[i4 + 1] = r * Math.sin(phi) * Math.sin(theta);
    positionBuffer.array[i4 + 2] = r * Math.cos(phi);
    positionBuffer.array[i4 + 3] = Math.random();  // lifetime
    
    // Velocity สุ่ม
    velocityBuffer.array[i4 + 0] = (Math.random() - 0.5) * 0.1;
    velocityBuffer.array[i4 + 1] = (Math.random() - 0.5) * 0.1;
    velocityBuffer.array[i4 + 2] = (Math.random() - 0.5) * 0.1;
    velocityBuffer.array[i4 + 3] = 0.5 + Math.random() * 0.5;
  }
  
  return { positionBuffer, velocityBuffer };
}

// สร้าง Compute Shader สำหรับ update particles
function createComputeShader(positionBuffer, velocityBuffer) {
  // สร้าง storage nodes
  const positionStorage = storage(positionBuffer, 'vec4', PARTICLE_COUNT);
  const velocityStorage = storage(velocityBuffer, 'vec4', PARTICLE_COUNT);
  
  // Uniforms
  const deltaTime = uniform(0.016);
  const attractorPos = uniform(new THREE.Vector3(0, 0, 0));
  const attractorStrength = uniform(1.0);
  const damping = uniform(0.98);
  
  // Compute function
  const computeParticles = tslFn(() => {
    // ดึง index ของ thread นี้
    const idx = instanceIndex;
    
    // อ่านข้อมูลปัจจุบัน
    const pos = positionStorage.element(idx);
    const vel = velocityStorage.element(idx);
    
    // คำนวณ attraction force
    const toAttractor = sub(attractorPos, pos.xyz);
    const dist = length(toAttractor);
    const force = mul(normalize(toAttractor), div(attractorStrength, add(mul(dist, dist), 0.01)));
    
    // อัพเดท velocity
    const newVel = add(vel.xyz, mul(force, deltaTime));
    const dampedVel = mul(newVel, damping);
    
    // อัพเดท position
    const newPos = add(pos.xyz, mul(dampedVel, deltaTime));
    
    // อัพเดท lifetime
    const newLifetime = sub(pos.w, mul(deltaTime, 0.1));
    
    // Reset particle ถ้า lifetime หมด
    If(newLifetime.lessThan(0.0), () => {
      // Reset ไปที่ attractor position พร้อม random offset
      const theta = mul(idx.toFloat(), 2.399963);  // golden angle
      const phi = mul(idx.toFloat(), 1.618);
      
      positionStorage.element(idx).assign(vec4(
        add(attractorPos.x, mul(sin(phi), cos(theta))),
        add(attractorPos.y, cos(phi)),
        add(attractorPos.z, mul(sin(phi), sin(theta))),
        float(1.0)  // reset lifetime
      ));
      
      velocityStorage.element(idx).assign(vec4(0.0, 0.0, 0.0, 1.0));
    }).Else(() => {
      // บันทึกค่าใหม่
      positionStorage.element(idx).assign(vec4(newPos, newLifetime));
      velocityStorage.element(idx).assign(vec4(dampedVel, vel.w));
    });
  })();
  
  // สร้าง compute node
  computeParticles.computeNode = computeParticles;
  
  return {
    compute: computeParticles,
    uniforms: { deltaTime, attractorPos, attractorStrength, damping }
  };
}

// สร้าง Particle Render Material
function createParticleMaterial(positionBuffer) {
  const material = new THREE.SpriteNodeMaterial();
  
  // ใช้ position จาก storage buffer
  const positionStorage = storage(positionBuffer, 'vec4', PARTICLE_COUNT);
  
  // ดึง position ของแต่ละ instance
  const particlePos = positionStorage.element(instanceIndex);
  
  // ตั้งค่า position
  material.positionNode = particlePos.xyz;
  
  // สีตาม lifetime
  const lifetime = particlePos.w;
  const particleColor = mix(
    vec3(1.0, 0.2, 0.0),  // สีแดง (lifetime ต่ำ)
    vec3(0.0, 0.5, 1.0),  // สีน้ำเงิน (lifetime สูง)
    lifetime
  );
  
  material.colorNode = vec4(particleColor, lifetime);
  
  // ขนาด particle
  material.sizeNode = mul(lifetime, 0.05);
  
  return material;
}

// Main setup
async function setupComputeParticles(scene, renderer) {
  const { positionBuffer, velocityBuffer } = createParticleBuffers();
  const { compute, uniforms } = createComputeShader(positionBuffer, velocityBuffer);
  
  // สร้าง particle mesh
  const geometry = new THREE.InstancedBufferGeometry();
  geometry.instanceCount = PARTICLE_COUNT;
  
  // ไม่ต้องมี vertex เพราะใช้ SpriteNode
  geometry.setAttribute('position', new THREE.BufferAttribute(new Float32Array([0, 0, 0]), 3));
  
  const material = createParticleMaterial(positionBuffer);
  const particles = new THREE.Mesh(geometry, material);
  scene.add(particles);
  
  return {
    particles,
    compute,
    uniforms,
    update: (dt) => {
      uniforms.deltaTime.value = dt;
      
      // รัน compute shader
      renderer.computeAsync(compute);
    }
  };
}
```

---

## 5. การ Migrate จาก WebGL มา WebGPU

### Checklist การ Migration

**ขั้นตอนที่ 1: ตรวจสอบ compatibility**
```javascript
// ตรวจสอบว่า features ที่ใช้รองรับ WebGPU หรือไม่
const checkCompatibility = () => {
  const issues = [];
  
  // ShaderMaterial แบบเก่าอาจต้องแก้ไข
  scene.traverse(obj => {
    if (obj.material) {
      if (obj.material instanceof THREE.ShaderMaterial) {
        issues.push(`ShaderMaterial ใน ${obj.name} อาจต้องแปลงเป็น NodeMaterial`);
      }
      if (obj.material instanceof THREE.RawShaderMaterial) {
        issues.push(`RawShaderMaterial ใน ${obj.name} ต้องแปลงเป็น NodeMaterial`);
      }
    }
  });
  
  return issues;
};

const issues = checkCompatibility();
if (issues.length > 0) {
  console.warn('พบ compatibility issues:', issues);
}
```

**ขั้นตอนที่ 2: แปลง ShaderMaterial เป็น NodeMaterial**
```javascript
// แบบเก่า (WebGL ShaderMaterial)
const oldMaterial = new THREE.ShaderMaterial({
  uniforms: {
    uTime: { value: 0 },
    uColor: { value: new THREE.Color(0xff0000) }
  },
  vertexShader: `
    uniform float uTime;
    void main() {
      vec3 pos = position;
      pos.y += sin(pos.x * 5.0 + uTime) * 0.1;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
    }
  `,
  fragmentShader: `
    uniform vec3 uColor;
    void main() {
      gl_FragColor = vec4(uColor, 1.0);
    }
  `
});

// แบบใหม่ (WebGPU NodeMaterial)
import { 
  MeshBasicNodeMaterial, 
  positionLocal, 
  uniform, 
  sin, 
  mul, 
  add,
  timerGlobal,
  vec4,
  vec3,
  float,
  tslFn
} from 'three/tsl';

function createNodeMaterial() {
  const material = new MeshBasicNodeMaterial();
  
  const uTime = timerGlobal();  // automatic time
  const uColor = uniform(new THREE.Color(0xff0000));
  
  // Custom position node
  const customPosition = tslFn(() => {
    const pos = positionLocal.toVar();
    const wave = sin(add(mul(pos.x, 5.0), uTime));
    pos.y.addAssign(mul(wave, 0.1));
    return pos;
  });
  
  material.positionNode = customPosition();
  material.colorNode = vec4(uColor, 1.0);
  
  return material;
}
```

**ขั้นตอนที่ 3: อัพเดท Render Target**
```javascript
// แบบเก่า
const oldRenderTarget = new THREE.WebGLRenderTarget(width, height, {
  format: THREE.RGBAFormat,
  type: THREE.FloatType
});

// แบบใหม่ (ใช้ร่วมกันได้กับทั้ง WebGL และ WebGPU)
const newRenderTarget = new THREE.RenderTarget(width, height, {
  colorSpace: THREE.LinearSRGBColorSpace,
  type: THREE.FloatType
});
// หรือใช้ WebGLRenderTarget ซึ่งยังทำงานได้
```

---

## 6. WebGPU Feature Detection แบบสมบูรณ์

```javascript
class WebGPUDetector {
  constructor() {
    this.capabilities = {
      webgpu: false,
      computeShaders: false,
      multipleRenderTargets: false,
      maxTextureSize: 0,
      maxComputeWorkgroups: 0,
      features: []
    };
  }
  
  async detect() {
    // ตรวจสอบ WebGPU API
    if (!('gpu' in navigator)) {
      this.capabilities.webgpu = false;
      return this.capabilities;
    }
    
    try {
      // ขอ adapter
      const adapter = await navigator.gpu.requestAdapter({
        powerPreference: 'high-performance'
      });
      
      if (!adapter) {
        this.capabilities.webgpu = false;
        return this.capabilities;
      }
      
      // รวบรวม features
      const features = [];
      for (const feature of adapter.features) {
        features.push(feature);
      }
      
      const limits = adapter.limits;
      
      this.capabilities = {
        webgpu: true,
        computeShaders: true,  // WebGPU รองรับ compute เสมอ
        maxTextureSize: limits.maxTextureDimension2D,
        maxComputeWorkgroups: limits.maxComputeWorkgroupsPerDimension,
        maxComputeWorkgroupSize: {
          x: limits.maxComputeWorkgroupSizeX,
          y: limits.maxComputeWorkgroupSizeY,
          z: limits.maxComputeWorkgroupSizeZ
        },
        maxComputeInvocations: limits.maxComputeInvocationsPerWorkgroup,
        maxStorageBufferSize: limits.maxStorageBufferBindingSize,
        features: features,
        timestamp: features.includes('timestamp-query'),
        indirectDraw: true,  // WebGPU รองรับ
        multipleRenderTargets: true  // WebGPU รองรับ MRT
      };
      
    } catch (error) {
      this.capabilities.webgpu = false;
      this.capabilities.error = error.message;
    }
    
    return this.capabilities;
  }
  
  // เลือก renderer ที่เหมาะสม
  async getBestRenderer(canvas, options = {}) {
    const caps = await this.detect();
    
    if (caps.webgpu) {
      console.log('ใช้ WebGPU Renderer');
      console.log('Max Texture Size:', caps.maxTextureSize);
      console.log('Max Compute Workgroups:', caps.maxComputeWorkgroups);
      
      // Return WebGPURenderer (ต้อง import จาก three/webgpu)
      // const renderer = new WebGPURenderer({ canvas, ...options });
      // await renderer.init();
      // return { renderer, type: 'webgpu', capabilities: caps };
      
      return { type: 'webgpu', capabilities: caps };
    } else {
      console.log('Fallback ไป WebGL Renderer');
      const renderer = new THREE.WebGLRenderer({ canvas, antialias: true, ...options });
      return { renderer, type: 'webgl', capabilities: caps };
    }
  }
  
  getReport() {
    const caps = this.capabilities;
    return `
=== WebGPU Capabilities Report ===
WebGPU Supported: ${caps.webgpu}
Compute Shaders: ${caps.computeShaders}
Max Texture Size: ${caps.maxTextureSize}
Max Compute Workgroups: ${caps.maxComputeWorkgroups}
Features: ${caps.features.join(', ')}
=================================
    `.trim();
  }
}

// การใช้งาน
const detector = new WebGPUDetector();
const caps = await detector.detect();
console.log(detector.getReport());
```

---

## 7. TSL Material ตัวอย่างเต็มรูปแบบ

### TSL Gradient Material

```javascript
import * as THREE from 'three';
import {
  MeshStandardNodeMaterial,
  uv,
  positionWorld,
  normalWorld,
  color,
  uniform,
  timerGlobal,
  sin,
  cos,
  mul,
  add,
  sub,
  mix,
  smoothstep,
  vec3,
  vec4,
  float,
  abs,
  clamp,
  dot,
  normalize,
  tslFn
} from 'three/tsl';

// 1. Material ที่มี gradient สวยงาม
function createGradientMaterial(colorTop, colorBottom) {
  const mat = new MeshStandardNodeMaterial();
  
  const cTop = uniform(new THREE.Color(colorTop));
  const cBottom = uniform(new THREE.Color(colorBottom));
  
  // ใช้ world position Y สำหรับ gradient
  const worldY = positionWorld.y;
  const t = clamp(add(mul(worldY, 0.5), 0.5), 0.0, 1.0);
  
  mat.colorNode = vec4(mix(cBottom, cTop, t), 1.0);
  mat.roughnessNode = float(0.4);
  mat.metalnessNode = float(0.1);
  
  return mat;
}

// 2. Material ที่มี animated patterns
function createAnimatedPatternMaterial() {
  const mat = new MeshStandardNodeMaterial();
  
  const timeNode = timerGlobal();
  const uvNode = uv();
  
  // Checker pattern
  const checkerFn = tslFn(([uvInput]) => {
    const scaledUV = mul(uvInput, 10.0);
    const cx = abs(sin(mul(scaledUV.x, 3.14159)));
    const cy = abs(sin(mul(scaledUV.y, 3.14159)));
    return smoothstep(0.4, 0.5, mul(cx, cy));
  });
  
  const checker = checkerFn(uvNode);
  
  // Animated color
  const animColor = vec3(
    add(0.5, mul(0.5, sin(add(timeNode, mul(uvNode.x, 5.0))))),
    add(0.5, mul(0.5, sin(add(timeNode, mul(uvNode.y, 5.0), 2.0)))),
    add(0.5, mul(0.5, cos(add(timeNode, 4.0))))
  );
  
  mat.colorNode = vec4(mix(vec3(0.1), animColor, checker), 1.0);
  mat.roughnessNode = mix(float(0.1), float(0.9), checker);
  mat.metalnessNode = float(0.5);
  
  return mat;
}

// 3. Holographic material
function createHolographicMaterial() {
  const mat = new MeshStandardNodeMaterial();
  mat.side = THREE.DoubleSide;
  mat.transparent = true;
  
  const timeNode = timerGlobal();
  const uvNode = uv();
  const normalNode = normalWorld;
  
  // Fresnel effect
  // const viewDir = cameraPosition.sub(positionWorld).normalize();
  // const fresnel = sub(1.0, abs(dot(normalNode, viewDir)));
  
  // Scan lines
  const scanLine = abs(sin(add(
    mul(positionWorld.y, 20.0),
    mul(timeNode, 2.0)
  )));
  
  // Holographic colors
  const holoColor = vec3(
    add(0.3, mul(0.7, sin(add(mul(uvNode.x, 10.0), timeNode)))),
    add(0.5, mul(0.5, sin(add(mul(uvNode.y, 8.0), mul(timeNode, 1.3))))),
    add(0.7, mul(0.3, cos(timeNode)))
  );
  
  const alpha = smoothstep(0.3, 1.0, scanLine);
  
  mat.colorNode = vec4(holoColor, mul(alpha, 0.7));
  mat.emissiveNode = mul(holoColor, 0.5);
  
  return mat;
}

// การใช้งาน
const scene = new THREE.Scene();

// Object 1: Gradient
const geo1 = new THREE.SphereGeometry(1, 64, 64);
const mat1 = createGradientMaterial(0x00ffff, 0xff0088);
const mesh1 = new THREE.Mesh(geo1, mat1);
mesh1.position.x = -3;
scene.add(mesh1);

// Object 2: Animated Pattern
const geo2 = new THREE.BoxGeometry(1.5, 1.5, 1.5, 4, 4, 4);
const mat2 = createAnimatedPatternMaterial();
const mesh2 = new THREE.Mesh(geo2, mat2);
scene.add(mesh2);

// Object 3: Holographic
const geo3 = new THREE.TorusGeometry(1, 0.3, 32, 128);
const mat3 = createHolographicMaterial();
const mesh3 = new THREE.Mesh(geo3, mat3);
mesh3.position.x = 3;
scene.add(mesh3);
```

---

## 8. สรุปและขั้นตอนถัดไป

### สิ่งที่เรียนรู้ในบทนี้:
1. **WebGPU vs WebGL**: WebGPU มี overhead ต่ำกว่า รองรับ compute shaders และมีการจัดการ resource แบบ explicit
2. **WebGPURenderer**: การ setup แบบ async ที่มี feature detection
3. **TSL (Three.js Shading Language)**: ระบบ node-based ที่ทำงานได้กับทั้ง WebGL และ WebGPU
4. **Compute Shaders**: การใช้งาน GPU compute สำหรับ simulation
5. **Migration**: การย้าย ShaderMaterial ไปเป็น NodeMaterial

### Browser Support (ณ ปัจจุบัน):
- **Chrome 113+**: รองรับเต็มที่
- **Edge 113+**: รองรับเต็มที่
- **Firefox**: Nightly เท่านั้น (ต้องเปิด flag)
- **Safari**: มีแผนรองรับ
- **Mobile**: Chrome on Android 113+

### เคล็ดลับ:
```javascript
// ใช้ isAvailable เพื่อตรวจสอบ WebGPU support
import { WebGPURenderer } from 'three/webgpu';

const isWebGPUAvailable = await WebGPURenderer.isAvailable();

const renderer = new WebGPURenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);

// ต้อง init ก่อนใช้งาน
await renderer.init();
```

ในบทถัดไป (Part 82) เราจะเจาะลึก Node Materials และ TSL มากยิ่งขึ้น รวมถึงการสร้าง custom shader nodes และการทำ PBR material ที่ซับซ้อน
