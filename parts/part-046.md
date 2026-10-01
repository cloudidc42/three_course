# Part 46: Ocean & Water Effects ใน Three.js

## บทนำ

การสร้างน้ำและมหาสมุทรที่สมจริงครอบคลุม:
- **Ocean Shader** - คลื่น Gerstner หลายชั้น
- **Water Reflections** - สะท้อนท้องฟ้าและสิ่งแวดล้อม
- **Foam & Wake** - ฟองน้ำและรอยเรือ
- **Underwater Effect** - มุมมองใต้น้ำ
- **Caustics** - แสงหักเหใต้น้ำ

## 1. Gerstner Wave Theory

```javascript
// ===== Gerstner Waves =====
// คลื่นทะเลที่สมจริงกว่า sine waves ธรรมดา
// ทุกจุดบนผิวน้ำเคลื่อนที่แบบวงกลม (circular motion)

// GLSL function:
const gerstnerWaveGLSL = `
struct GerstnerWave {
  vec2 direction;   // normalized wave direction
  float amplitude;  // wave height (meters)
  float steepness;  // sharpness of peaks (0-1)
  float wavelength; // meters per cycle
  float speed;      // meters/second
};

// คำนวณการเคลื่อนที่ Gerstner Wave ที่จุดหนึ่ง
void gerstnerWave(GerstnerWave wave, float time, inout vec3 pos, inout vec3 tangent, inout vec3 binormal) {
  float k = 2.0 * PI / wave.wavelength;       // wave number
  float c = sqrt(9.8 / k);                     // phase speed
  vec2 d = normalize(wave.direction);
  float f = k * (dot(d, pos.xz) - c * time);  // phase
  float a = wave.steepness / k;                // amplitude from steepness
  
  tangent += vec3(
    -d.x * d.x * (wave.steepness * sin(f)),
    d.x * (wave.steepness * cos(f)),
    -d.x * d.y * (wave.steepness * sin(f))
  );
  
  binormal += vec3(
    -d.x * d.y * (wave.steepness * sin(f)),
    d.y * (wave.steepness * cos(f)),
    -d.y * d.y * (wave.steepness * sin(f))
  );
  
  pos += vec3(
    d.x * (a * cos(f)),
    a * sin(f),
    d.y * (a * cos(f))
  );
}
`;
```

## 2. Ocean Shader Material

```javascript
function createOceanMaterial(envTexture) {
  const oceanVertexShader = `
    #define PI 3.14159265358979
    
    uniform float time;
    uniform vec4 wave0; // xy=dir, z=steepness, w=wavelength
    uniform vec4 wave1;
    uniform vec4 wave2;
    uniform vec4 wave3;
    
    varying vec3 vWorldPos;
    varying vec3 vNormal;
    varying vec2 vUv;
    varying float vFoam;
    
    vec3 gerstner(vec4 wave, float t, vec3 pos) {
      float k = 2.0 * PI / wave.w;
      float c = sqrt(9.8 / k);
      vec2 d = normalize(wave.xy);
      float f = k * (dot(d, pos.xz) - c * t);
      float a = wave.z / k;
      
      return vec3(
        d.x * a * cos(f),
        a * sin(f),
        d.y * a * cos(f)
      );
    }
    
    void main() {
      vec3 pos = position;
      vec3 tangent = vec3(1.0, 0.0, 0.0);
      vec3 binormal = vec3(0.0, 0.0, 1.0);
      vFoam = 0.0;
      
      // Apply 4 Gerstner waves
      vec4 waves[4];
      waves[0] = wave0; waves[1] = wave1;
      waves[2] = wave2; waves[3] = wave3;
      
      for (int i = 0; i < 4; i++) {
        float k = 2.0 * PI / waves[i].w;
        float c = sqrt(9.8 / k);
        vec2 d = normalize(waves[i].xy);
        float f = k * (dot(d, pos.xz) - c * time);
        float a = waves[i].z / k;
        float s = waves[i].z;
        
        pos.x += d.x * a * cos(f);
        pos.y += a * sin(f);
        pos.z += d.y * a * cos(f);
        
        tangent.x += -d.x * d.x * s * sin(f);
        tangent.y += d.x * s * cos(f);
        tangent.z += -d.x * d.y * s * sin(f);
        
        binormal.x += -d.x * d.y * s * sin(f);
        binormal.y += d.y * s * cos(f);
        binormal.z += -d.y * d.y * s * sin(f);
        
        // Foam at wave crests
        vFoam += max(0.0, (sin(f) - 0.7)) * 3.0;
      }
      
      vNormal = normalize(cross(binormal, tangent));
      vWorldPos = (modelMatrix * vec4(pos, 1.0)).xyz;
      vUv = uv;
      
      gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
    }
  `;

  const oceanFragmentShader = `
    uniform vec3 cameraPos;
    uniform samplerCube envMap;
    uniform sampler2D normalMap;
    uniform float time;
    uniform vec3 shallowColor;  // สีน้ำตื้น
    uniform vec3 deepColor;     // สีน้ำลึก
    
    varying vec3 vWorldPos;
    varying vec3 vNormal;
    varying vec2 vUv;
    varying float vFoam;
    
    float fresnelSchlick(float cosTheta, float F0) {
      return F0 + (1.0 - F0) * pow(1.0 - cosTheta, 5.0);
    }
    
    void main() {
      vec3 N = normalize(vNormal);
      vec3 V = normalize(cameraPos - vWorldPos);
      
      // Normal map detail (animated scrolling)
      vec2 uv1 = vUv * 4.0 + vec2(time * 0.03, time * 0.02);
      vec2 uv2 = vUv * 6.0 + vec2(-time * 0.02, time * 0.04);
      
      vec3 n1 = texture2D(normalMap, uv1).rgb * 2.0 - 1.0;
      vec3 n2 = texture2D(normalMap, uv2).rgb * 2.0 - 1.0;
      vec3 detailNormal = normalize(n1 + n2 + vec3(0, 2, 0));
      
      N = normalize(mix(N, detailNormal, 0.3));
      
      // Reflection
      vec3 R = reflect(-V, N);
      vec3 envColor = textureCube(envMap, R).rgb;
      
      // Fresnel
      float fresnel = fresnelSchlick(max(0.0, dot(V, N)), 0.02);
      
      // Water color depth
      float depth = 1.0 - exp(-2.0 * max(0.0, -vWorldPos.y + 1.0));
      vec3 waterColor = mix(shallowColor, deepColor, depth * 0.5);
      
      // Specular highlight
      vec3 L = normalize(vec3(0.5, 1.0, 0.3)); // sun direction
      float spec = pow(max(0.0, dot(reflect(-L, N), V)), 256.0) * 3.0;
      
      // Foam
      float foam = smoothstep(0.3, 0.6, vFoam);
      vec3 foamColor = vec3(0.95, 0.97, 1.0);
      
      // Final color
      vec3 color = mix(waterColor, envColor, fresnel * 0.6);
      color += vec3(spec) * vec3(1.0, 0.98, 0.9);
      color = mix(color, foamColor, foam);
      
      // Subsurface scattering (แสงที่ส่องผ่านยอดคลื่น)
      float sss = pow(max(0.0, dot(L, -N)), 3.0) * 0.2;
      color += vec3(0.2, 0.7, 0.8) * sss;
      
      gl_FragColor = vec4(color, 0.92);
    }
  `;

  return new THREE.ShaderMaterial({
    uniforms: {
      time: { value: 0 },
      wave0: { value: new THREE.Vector4(1, 0.2, 0.3, 12) },
      wave1: { value: new THREE.Vector4(0.8, 0.6, 0.25, 8) },
      wave2: { value: new THREE.Vector4(-0.5, 0.8, 0.2, 5) },
      wave3: { value: new THREE.Vector4(0.2, -1, 0.15, 3) },
      cameraPos: { value: new THREE.Vector3() },
      envMap: { value: envTexture },
      normalMap: { value: createWaterNormalMap() },
      shallowColor: { value: new THREE.Color(0x00c8c8) },
      deepColor: { value: new THREE.Color(0x004060) }
    },
    vertexShader: oceanVertexShader,
    fragmentShader: oceanFragmentShader,
    transparent: true,
    side: THREE.FrontSide
  });
}

function createWaterNormalMap() {
  const canvas = document.createElement('canvas');
  canvas.width = canvas.height = 512;
  const ctx = canvas.getContext('2d');
  
  // Generate normal map from noise
  const imageData = ctx.createImageData(512, 512);
  for (let y = 0; y < 512; y++) {
    for (let x = 0; x < 512; x++) {
      const nx = Math.sin(x * 0.05) * 127 + 128;
      const ny = Math.sin(y * 0.05 + 1) * 127 + 128;
      const nz = 255;
      const idx = (y * 512 + x) * 4;
      imageData.data[idx] = nx;
      imageData.data[idx+1] = ny;
      imageData.data[idx+2] = nz;
      imageData.data[idx+3] = 255;
    }
  }
  ctx.putImageData(imageData, 0, 0);
  
  const tex = new THREE.CanvasTexture(canvas);
  tex.wrapS = tex.wrapT = THREE.RepeatWrapping;
  return tex;
}
```

## 3. Boat Wake & Ripples

```javascript
class BoatWakeSystem {
  constructor(scene, waterLevel = 0) {
    this.scene = scene;
    this.waterLevel = waterLevel;
    this.wakeParticles = [];
    this.rippleRings = [];
    
    this._createRippleMaterial();
  }
  
  _createRippleMaterial() {
    this.rippleMat = new THREE.ShaderMaterial({
      uniforms: {
        time: { value: 0 },
        color: { value: new THREE.Color(0.9, 0.95, 1.0) }
      },
      vertexShader: `
        uniform float time;
        varying float vProgress;
        varying vec2 vUv;
        
        void main() {
          vUv = uv;
          vProgress = mod(time, 2.0) / 2.0; // 0-1 ใน 2 วินาที
          gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
      `,
      fragmentShader: `
        uniform vec3 color;
        varying float vProgress;
        varying vec2 vUv;
        
        void main() {
          vec2 center = vUv - 0.5;
          float dist = length(center) * 2.0;
          
          // Expanding ring
          float ring = abs(dist - vProgress) / 0.05;
          float alpha = (1.0 - smoothstep(0.0, 1.0, ring)) * (1.0 - vProgress);
          
          if (alpha < 0.01) discard;
          gl_FragColor = vec4(color, alpha * 0.7);
        }
      `,
      transparent: true,
      depthWrite: false,
      side: THREE.DoubleSide
    });
  }
  
  // เพิ่ม ripple ที่ตำแหน่งที่กำหนด
  addRipple(x, z) {
    const geo = new THREE.PlaneGeometry(10, 10);
    geo.rotateX(-Math.PI / 2);
    
    const mat = this.rippleMat.clone();
    mat.uniforms.time = { value: 0 };
    
    const mesh = new THREE.Mesh(geo, mat);
    mesh.position.set(x, this.waterLevel + 0.01, z);
    this.scene.add(mesh);
    
    const ripple = { mesh, age: 0, maxAge: 2 };
    this.rippleRings.push(ripple);
  }
  
  // Wake trail ด้านหลังเรือ
  addWakePoint(x, z) {
    const geo = new THREE.PlaneGeometry(2, 2);
    geo.rotateX(-Math.PI / 2);
    
    const mat = new THREE.MeshBasicMaterial({
      color: 0xffffff,
      transparent: true,
      opacity: 0.6,
      depthWrite: false
    });
    
    const mesh = new THREE.Mesh(geo, mat);
    mesh.position.set(x, this.waterLevel + 0.01, z);
    this.scene.add(mesh);
    
    this.wakeParticles.push({ mesh, age: 0, maxAge: 3 });
  }
  
  update(deltaTime) {
    // Update ripples
    for (let i = this.rippleRings.length - 1; i >= 0; i--) {
      const r = this.rippleRings[i];
      r.age += deltaTime;
      r.mesh.material.uniforms.time.value = r.age;
      
      if (r.age >= r.maxAge) {
        this.scene.remove(r.mesh);
        this.rippleRings.splice(i, 1);
      }
    }
    
    // Update wake particles
    for (let i = this.wakeParticles.length - 1; i >= 0; i--) {
      const w = this.wakeParticles[i];
      w.age += deltaTime;
      const t = w.age / w.maxAge;
      w.mesh.material.opacity = (1 - t) * 0.6;
      
      // Expand
      const s = 1 + t * 4;
      w.mesh.scale.set(s, 1, s * 0.3 + 1);
      
      if (w.age >= w.maxAge) {
        this.scene.remove(w.mesh);
        this.wakeParticles.splice(i, 1);
      }
    }
  }
}
```

## 4. Underwater Effect

```javascript
class UnderwaterEffect {
  constructor(renderer, scene, camera) {
    this.renderer = renderer;
    this.scene = scene;
    this.camera = camera;
    this.isUnderwater = false;
    this.waterLevel = 0;
    
    this._createCaustics();
    this._createFog();
  }
  
  _createCaustics() {
    // Caustics texture (แสงหักเหใต้น้ำ)
    const canvas = document.createElement('canvas');
    canvas.width = canvas.height = 256;
    const ctx = canvas.getContext('2d');
    
    // Generate caustic pattern
    const drawCaustics = (time = 0) => {
      ctx.fillStyle = '#003355';
      ctx.fillRect(0, 0, 256, 256);
      
      ctx.strokeStyle = 'rgba(100, 200, 255, 0.3)';
      ctx.lineWidth = 1;
      
      for (let i = 0; i < 30; i++) {
        const x = (Math.sin(i * 2.3 + time) + 1) * 128;
        const y = (Math.cos(i * 1.7 + time * 0.7) + 1) * 128;
        const r = 5 + Math.abs(Math.sin(i + time)) * 20;
        
        ctx.beginPath();
        ctx.arc(x, y, r, 0, Math.PI * 2);
        ctx.stroke();
      }
    };
    
    drawCaustics();
    
    this.causticTexture = new THREE.CanvasTexture(canvas);
    this.causticCanvas = canvas;
    this.causticCtx = ctx;
    this.causticDrawFn = drawCaustics;
    
    // Caustic projector (floor light)
    this.causticLight = new THREE.SpotLight(0x4488ff, 2, 50, Math.PI / 4, 0.5);
    this.causticLight.position.set(0, 5, 0);
    this.causticLight.map = this.causticTexture;
    this.causticLight.visible = false;
    this.scene.add(this.causticLight);
  }
  
  _createFog() {
    this.originalFog = this.scene.fog;
    this.underwaterFog = new THREE.FogExp2(0x004466, 0.08);
  }
  
  check() {
    const cameraY = this.camera.position.y;
    
    if (cameraY < this.waterLevel && !this.isUnderwater) {
      this.enterWater();
    } else if (cameraY >= this.waterLevel && this.isUnderwater) {
      this.exitWater();
    }
  }
  
  enterWater() {
    this.isUnderwater = true;
    this.scene.fog = this.underwaterFog;
    this.causticLight.visible = true;
    this.renderer.setClearColor(0x004466, 1);
    console.log('Entered water');
  }
  
  exitWater() {
    this.isUnderwater = false;
    this.scene.fog = this.originalFog;
    this.causticLight.visible = false;
    this.renderer.setClearColor(0x87ceeb, 1);
  }
  
  update(time) {
    if (this.isUnderwater) {
      // Animate caustics
      this.causticDrawFn(time * 0.5);
      this.causticTexture.needsUpdate = true;
      
      // Undulate fog density (เหมือน underwater shimmer)
      this.underwaterFog.density = 0.07 + Math.sin(time * 0.5) * 0.01;
    }
    
    this.check();
  }
}
```

## 5. Complete Ocean Demo

```html
<!DOCTYPE html>
<html>
<head>
  <title>Ocean Scene</title>
  <style>
    * { margin: 0; padding: 0; }
    body { background: #001122; overflow: hidden; }
    canvas { display: block; }
    #ui {
      position: fixed;
      top: 20px;
      right: 20px;
      background: rgba(0,20,40,0.8);
      color: #8df;
      padding: 15px;
      border-radius: 8px;
      font-family: monospace;
      font-size: 13px;
    }
    #ui label { display: block; margin: 5px 0; }
    #ui input[type=range] { width: 140px; vertical-align: middle; }
    #boatInfo {
      position: fixed;
      bottom: 20px;
      left: 50%;
      transform: translateX(-50%);
      color: #8df;
      font-family: Arial, sans-serif;
      text-align: center;
    }
  </style>
</head>
<body>
<div id="ui">
  <b>Ocean Controls</b><br><br>
  <label>Wave Height: <input type="range" id="waveHeight" min="0.1" max="3" step="0.1" value="0.8"></label>
  <label>Wave Speed: <input type="range" id="waveSpeed" min="0.1" max="3" step="0.1" value="1"></label>
  <label>Turbidity: <input type="range" id="turbidity" min="0" max="1" step="0.05" value="0.3"></label>
  <label>Wind X: <input type="range" id="windX" min="-1" max="1" step="0.1" value="0.7"></label>
  <label>Wind Z: <input type="range" id="windZ" min="-1" max="1" step="0.1" value="0.3"></label>
</div>
<div id="boatInfo">WASD to move boat | Scroll to zoom</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';
import { RoomEnvironment } from 'https://unpkg.com/three@0.158.0/examples/jsm/environments/RoomEnvironment.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(70, innerWidth/innerHeight, 0.1, 500);
camera.position.set(0, 6, 25);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
renderer.outputColorSpace = THREE.SRGBColorSpace;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 0.6;
document.body.appendChild(renderer.domElement);

// Environment
const pmremGen = new THREE.PMREMGenerator(renderer);
const env = pmremGen.fromScene(new RoomEnvironment()).texture;
scene.environment = env;

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.target.set(0, 0, 0);
controls.maxPolarAngle = Math.PI * 0.65;

// ===== SKY =====
const skyGeo = new THREE.SphereGeometry(400, 32, 16);
const skyMat = new THREE.ShaderMaterial({
  uniforms: {
    topColor: { value: new THREE.Color(0x0044aa) },
    bottomColor: { value: new THREE.Color(0x88bbdd) },
    sunDir: { value: new THREE.Vector3(0.5, 0.7, 0.2).normalize() }
  },
  vertexShader: `
    varying vec3 vDir;
    void main() {
      vDir = normalize(position);
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    uniform vec3 topColor, bottomColor, sunDir;
    varying vec3 vDir;
    void main() {
      float t = clamp(dot(vDir, vec3(0,1,0)) * 2.0 + 0.5, 0.0, 1.0);
      vec3 sky = mix(bottomColor, topColor, t);
      
      // Sun
      float sun = pow(max(0.0, dot(vDir, sunDir)), 64.0) * 2.0;
      sky += vec3(sun * 1.2, sun * 0.9, sun * 0.5);
      
      gl_FragColor = vec4(sky, 1.0);
    }
  `,
  side: THREE.BackSide, depthWrite: false
});
scene.add(new THREE.Mesh(skyGeo, skyMat));

// ===== OCEAN =====
const OCEAN_SIZE = 200;
const OCEAN_SEGS = 128;

const oceanGeo = new THREE.PlaneGeometry(OCEAN_SIZE, OCEAN_SIZE, OCEAN_SEGS, OCEAN_SEGS);
oceanGeo.rotateX(-Math.PI / 2);

const oceanMat = new THREE.ShaderMaterial({
  uniforms: {
    time: { value: 0 },
    waveHeight: { value: 0.8 },
    waveSpeed: { value: 1.0 },
    turbidity: { value: 0.3 },
    windX: { value: 0.7 },
    windZ: { value: 0.3 },
    cameraPos: { value: new THREE.Vector3() },
    envMap: { value: env },
    sunDir: { value: new THREE.Vector3(0.5, 0.7, 0.2).normalize() }
  },
  vertexShader: `
    #define PI 3.14159265
    
    uniform float time;
    uniform float waveHeight;
    uniform float waveSpeed;
    uniform float windX;
    uniform float windZ;
    
    varying vec3 vWorldPos;
    varying vec3 vNormal;
    varying float vFoam;
    
    float hash(vec2 p) { return fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5); }
    
    float wave(vec2 pos, vec2 dir, float wavelength, float steepness, float t) {
      float k = 2.0 * PI / wavelength;
      float c = sqrt(9.8 / k);
      float f = k * (dot(dir, pos) - c * t * waveSpeed);
      return steepness / k * sin(f);
    }
    
    float waveNX(vec2 pos, vec2 dir, float wavelength, float steepness, float t) {
      float k = 2.0 * PI / wavelength;
      float c = sqrt(9.8 / k);
      float f = k * (dot(dir, pos) - c * t * waveSpeed);
      return dir.x * steepness * cos(f);
    }
    
    float waveNZ(vec2 pos, vec2 dir, float wavelength, float steepness, float t) {
      float k = 2.0 * PI / wavelength;
      float c = sqrt(9.8 / k);
      float f = k * (dot(dir, pos) - c * t * waveSpeed);
      return dir.y * steepness * cos(f);
    }
    
    void main() {
      vec3 pos = position;
      vec2 w = normalize(vec2(windX, windZ));
      
      float h = 0.0;
      float nx = 0.0, nz = 0.0;
      
      // 4 wave layers
      vec2[4] dirs;
      float[4] wls, steeps;
      dirs[0] = w; wls[0] = 18.0; steeps[0] = 0.4;
      dirs[1] = normalize(w + vec2(0.4, -0.3)); wls[1] = 10.0; steeps[1] = 0.3;
      dirs[2] = normalize(w - vec2(0.3, 0.5)); wls[2] = 6.0; steeps[2] = 0.25;
      dirs[3] = normalize(-w + vec2(0.2, 0.8)); wls[3] = 3.5; steeps[3] = 0.2;
      
      for (int i = 0; i < 4; i++) {
        h += wave(pos.xz, dirs[i], wls[i], steeps[i], time) * waveHeight;
        nx += waveNX(pos.xz, dirs[i], wls[i], steeps[i], time) * waveHeight;
        nz += waveNZ(pos.xz, dirs[i], wls[i], steeps[i], time) * waveHeight;
      }
      
      pos.y = h;
      vNormal = normalize(vec3(-nx, 1.0, -nz));
      vFoam = clamp(h / waveHeight, 0.0, 1.0);
      vWorldPos = (modelMatrix * vec4(pos, 1.0)).xyz;
      
      gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
    }
  `,
  fragmentShader: `
    uniform vec3 cameraPos;
    uniform samplerCube envMap;
    uniform vec3 sunDir;
    uniform float turbidity;
    
    varying vec3 vWorldPos;
    varying vec3 vNormal;
    varying float vFoam;
    
    void main() {
      vec3 N = normalize(vNormal);
      vec3 V = normalize(cameraPos - vWorldPos);
      vec3 R = reflect(-V, N);
      
      // Fresnel
      float fresnel = 0.02 + 0.98 * pow(1.0 - max(0.0, dot(V, N)), 5.0);
      
      // Reflection
      vec3 envColor = textureCube(envMap, R).rgb;
      
      // Water color
      vec3 shallow = vec3(0.1, 0.55, 0.55);
      vec3 deep = vec3(0.01, 0.08, 0.2);
      float depth = clamp(turbidity, 0.0, 1.0);
      vec3 waterColor = mix(shallow, deep, depth);
      
      // Specular sun
      vec3 H = normalize(sunDir + V);
      float spec = pow(max(0.0, dot(N, H)), 512.0) * 3.0;
      
      // Subsurface scatter at wave crests
      float sss = pow(max(0.0, dot(sunDir, -N)), 2.0) * 0.3;
      waterColor += vec3(0.1, 0.6, 0.8) * sss;
      
      vec3 color = mix(waterColor, envColor, fresnel * 0.7);
      color += vec3(1.0, 0.98, 0.9) * spec;
      
      // Foam at crests
      float foam = smoothstep(0.7, 0.9, vFoam);
      color = mix(color, vec3(0.97, 0.98, 1.0), foam * 0.5);
      
      gl_FragColor = vec4(color, 0.93);
    }
  `,
  transparent: true
});

const ocean = new THREE.Mesh(oceanGeo, oceanMat);
scene.add(ocean);

// ===== BOAT =====
const boatGroup = new THREE.Group();
scene.add(boatGroup);

// Hull
const hullGeo = new THREE.BoxGeometry(3, 1, 7);
const hullMat = new THREE.MeshStandardMaterial({ color: 0xcc4422, roughness: 0.6 });
const hull = new THREE.Mesh(hullGeo, hullMat);
hull.castShadow = true;
boatGroup.add(hull);

// Deck
const deckGeo = new THREE.BoxGeometry(2.5, 0.3, 6);
const deckMat = new THREE.MeshStandardMaterial({ color: 0xc8a06a, roughness: 0.8 });
const deck = new THREE.Mesh(deckGeo, deckMat);
deck.position.y = 0.65;
boatGroup.add(deck);

// Mast
const mastGeo = new THREE.CylinderGeometry(0.05, 0.08, 8, 8);
const mastMat = new THREE.MeshStandardMaterial({ color: 0x8a6a3a });
const mast = new THREE.Mesh(mastGeo, mastMat);
mast.position.set(0, 4.5, 0);
boatGroup.add(mast);

// Sail
const sailGeo = new THREE.PlaneGeometry(3, 5);
const sailMat = new THREE.MeshStandardMaterial({ 
  color: 0xfffae0, side: THREE.DoubleSide, roughness: 0.9
});
const sail = new THREE.Mesh(sailGeo, sailMat);
sail.position.set(1.5, 4.5, 0);
sail.rotation.y = 0.3;
boatGroup.add(sail);

boatGroup.position.set(5, 0, 0);

// ===== ISLANDS =====
function createIsland(x, z, size) {
  const g = new THREE.Group();
  g.position.set(x, -0.5, z);

  // Base
  const base = new THREE.Mesh(
    new THREE.CylinderGeometry(size * 0.8, size * 1.2, 3, 16),
    new THREE.MeshStandardMaterial({ color: 0xd4c080, roughness: 1 })
  );
  g.add(base);

  // Top dirt
  const top = new THREE.Mesh(
    new THREE.SphereGeometry(size * 0.9, 16, 8, 0, Math.PI * 2, 0, Math.PI / 2),
    new THREE.MeshStandardMaterial({ color: 0x3a7a3a, roughness: 0.9 })
  );
  top.position.y = 1;
  g.add(top);

  // Palm trees
  for (let i = 0; i < 3; i++) {
    const px = (Math.random() - 0.5) * size * 1.2;
    const pz = (Math.random() - 0.5) * size * 1.2;
    const py = 1.5 + Math.random();

    const trunkH = 4 + Math.random() * 3;
    const trunk = new THREE.Mesh(
      new THREE.CylinderGeometry(0.1, 0.2, trunkH, 6),
      new THREE.MeshStandardMaterial({ color: 0x8a6030 })
    );
    trunk.position.set(px, py + trunkH/2, pz);
    trunk.rotation.z = (Math.random() - 0.5) * 0.4;
    g.add(trunk);

    const leaves = new THREE.Mesh(
      new THREE.ConeGeometry(2.5, 1.5, 8),
      new THREE.MeshStandardMaterial({ color: 0x2a8a2a, roughness: 0.8 })
    );
    leaves.position.set(px, py + trunkH + 0.5, pz);
    g.add(leaves);
  }

  scene.add(g);
}

createIsland(-40, -30, 8);
createIsland(50, 20, 5);
createIsland(-20, 50, 10);

// Lighting
const sun = new THREE.DirectionalLight(0xfff8e0, 4);
sun.position.set(50, 70, 20);
sun.castShadow = true;
sun.shadow.mapSize.set(2048, 2048);
scene.add(sun);
scene.add(new THREE.AmbientLight(0x334466, 1.5));

// Fog
scene.fog = new THREE.FogExp2(0x88aacc, 0.005);

// ===== BOAT CONTROLS =====
const keys = {};
window.addEventListener('keydown', e => { keys[e.key.toLowerCase()] = true; });
window.addEventListener('keyup', e => { keys[e.key.toLowerCase()] = false; });

// ===== UI CONTROLS =====
document.getElementById('waveHeight').addEventListener('input', e => {
  oceanMat.uniforms.waveHeight.value = parseFloat(e.target.value);
});
document.getElementById('waveSpeed').addEventListener('input', e => {
  oceanMat.uniforms.waveSpeed.value = parseFloat(e.target.value);
});
document.getElementById('turbidity').addEventListener('input', e => {
  oceanMat.uniforms.turbidity.value = parseFloat(e.target.value);
});
document.getElementById('windX').addEventListener('input', e => {
  oceanMat.uniforms.windX.value = parseFloat(e.target.value);
});
document.getElementById('windZ').addEventListener('input', e => {
  oceanMat.uniforms.windZ.value = parseFloat(e.target.value);
});

// Wake system
const wakePositions = [];
let lastWakeTime = 0;

const clock = new THREE.Clock();

function getWaveHeight(x, z, time, wh, ws, wx, wz) {
  const w = Math.sqrt(wx * wx + wz * wz);
  const dx = wx / w, dz = wz / w;
  let h = 0;
  const waves = [
    [dx, dz, 18, 0.4],
    [dx + 0.4, dz - 0.3, 10, 0.3],
  ];
  for (const [ddx, ddz, wl, st] of waves) {
    const k = 2 * Math.PI / wl;
    const c = Math.sqrt(9.8 / k) * ws;
    const f = k * (ddx * x + ddz * z - c * time);
    h += (st / k) * Math.sin(f) * wh;
  }
  return h;
}

function animate() {
  requestAnimationFrame(animate);
  const dt = clock.getDelta();
  const elapsed = clock.elapsedTime;

  oceanMat.uniforms.time.value = elapsed;
  oceanMat.uniforms.cameraPos.value.copy(camera.position);

  // Boat movement
  const spd = 5;
  if (keys['w'] || keys['arrowup']) {
    boatGroup.position.x += Math.sin(boatGroup.rotation.y) * spd * dt;
    boatGroup.position.z += Math.cos(boatGroup.rotation.y) * spd * dt;
  }
  if (keys['s'] || keys['arrowdown']) {
    boatGroup.position.x -= Math.sin(boatGroup.rotation.y) * spd * dt;
    boatGroup.position.z -= Math.cos(boatGroup.rotation.y) * spd * dt;
  }
  if (keys['a'] || keys['arrowleft']) boatGroup.rotation.y += 1 * dt;
  if (keys['d'] || keys['arrowright']) boatGroup.rotation.y -= 1 * dt;

  // Boat on waves
  const bx = boatGroup.position.x, bz = boatGroup.position.z;
  const wh = oceanMat.uniforms.waveHeight.value;
  const ws = oceanMat.uniforms.waveSpeed.value;
  const wx = oceanMat.uniforms.windX.value;
  const wz = oceanMat.uniforms.windZ.value;

  const boatH = getWaveHeight(bx, bz, elapsed, wh, ws, wx, wz);
  boatGroup.position.y = boatH + 0.5;
  boatGroup.rotation.x = Math.sin(elapsed * 0.8) * 0.05 * wh;
  boatGroup.rotation.z = Math.cos(elapsed * 0.6) * 0.04 * wh;

  // Wake trail
  if (elapsed - lastWakeTime > 0.3 && (keys['w'] || keys['arrowup'])) {
    wakePositions.unshift({ x: bx, z: bz, age: 0 });
    if (wakePositions.length > 20) wakePositions.pop();
    lastWakeTime = elapsed;
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

## สรุป

ระบบน้ำสมบูรณ์:
1. **Gerstner Waves** - คลื่นที่สมจริงบน GPU
2. **Reflections** - สะท้อนผ่าน env map
3. **Fresnel Effect** - ความโปร่งใสตามมุมมอง
4. **Foam** - ฟองที่ยอดคลื่น
5. **Boat Physics** - เรือที่ลอยตาม wave height

---

## แบบฝึกหัด

1. **Easy**: สร้าง plane กับ ShaderMaterial ที่ทำ Gerstner wave
2. **Medium**: เพิ่ม underwater fog เมื่อกล้องต่ำกว่า y=0
3. **Hard**: ใช้ CubeCamera สำหรับ real-time water reflection
4. **Challenge**: เพิ่ม caustics projection บนพื้นใต้น้ำ
