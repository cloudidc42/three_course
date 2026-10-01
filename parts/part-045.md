# Part 45: Sky, Clouds & Atmosphere ใน Three.js

## บทนำ

การสร้างท้องฟ้าที่สมจริงในเกมและ 3D visualization เป็นสิ่งสำคัญ บทนี้ครอบคลุม:
- **Sky Shader** - จำลอง Rayleigh & Mie scattering
- **Day/Night Cycle** - ตำแหน่งดวงอาทิตย์ที่สมจริง
- **Procedural Clouds** - เมฆจาก noise
- **Stars & Moon** - ระบบดาวและดวงจันทร์
- **Atmospheric Fog** - หมอกบรรยากาศ

## 1. Physics ของท้องฟ้า

```javascript
// ===== Rayleigh & Mie Scattering =====
// Rayleigh - แสงกระจายจากโมเลกุลขนาดเล็ก → ทำให้ท้องฟ้าสีฟ้า
// Mie     - แสงกระจายจากอนุภาคขนาดใหญ่ → corona รอบดวงอาทิตย์

const skyVertexShader = `
  varying vec3 vWorldPosition;
  
  void main() {
    vec4 worldPos = modelMatrix * vec4(position, 1.0);
    vWorldPosition = worldPos.xyz;
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
  }
`;

const skyFragmentShader = `
  uniform vec3 sunPosition;    // normalized sun direction
  uniform float turbidity;     // ความขุ่นของบรรยากาศ (2-20)
  uniform float rayleigh;      // Rayleigh coefficient (0-4)
  uniform float mieCoefficient;// Mie coefficient (0-0.1)
  uniform float mieDirectionalG; // Mie directional (0.7-0.999)
  
  varying vec3 vWorldPosition;
  
  // Constants
  const vec3 up = vec3(0.0, 1.0, 0.0);
  const float e = 2.71828182845904523536;
  const float pi = 3.141592653589793238462643383;
  
  // Atmospheric scattering coefficients
  const float n = 1.0003;       // refractive index of air
  const float N = 2.545E25;     // molecular number density
  const float rayleighZenithLength = 8.4E3;
  const float mieZenithLength = 1.25E3;
  const float sunAngularDiameterCos = 0.999956676946;
  
  vec3 totalRayleigh(vec3 lambda) {
    return (8.0 * pow(pi, 3.0) * pow(pow(n, 2.0) - 1.0, 2.0) * (6.0 + 3.0 * 0.035))
           / (3.0 * N * pow(lambda, vec3(4.0)) * (6.0 - 7.0 * 0.035));
  }
  
  float rayleighPhase(float cosTheta) {
    return (3.0 / (16.0 * pi)) * (1.0 + pow(cosTheta, 2.0));
  }
  
  float miePhase(float g, float cosTheta) {
    float g2 = pow(g, 2.0);
    float k = 3.0 / (8.0 * pi) * (1.0 - g2) / (2.0 + g2);
    return k * (1.0 + pow(cosTheta, 2.0)) / pow(1.0 + g2 - 2.0 * g * cosTheta, 1.5);
  }
  
  void main() {
    vec3 direction = normalize(vWorldPosition);
    
    // Sun angle
    float sunTheta = acos(clamp(sunPosition.y, -1.0, 1.0));
    float cosTheta = dot(direction, sunPosition);
    
    // Optical depth (เส้นทางที่แสงผ่าน)
    float zenithAngle = acos(max(0.0, dot(up, direction)));
    float inv = 1.0 / (cos(zenithAngle) + 0.15 * pow(93.885 - (zenithAngle * 180.0/pi), -1.253));
    float sR = rayleighZenithLength * inv;
    float sM = mieZenithLength * inv;
    
    // Rayleigh coefficients for visible wavelengths (RGB)
    vec3 betaR = totalRayleigh(vec3(680E-9, 550E-9, 450E-9)) * rayleigh;
    
    // Mie coefficient
    vec3 betaM = vec3(mieCoefficient) * 1.0E-4 * turbidity * 0.434 * 
                 vec3(1.0 / pow(680E-9, 2.0), 1.0 / pow(550E-9, 2.0), 1.0 / pow(450E-9, 2.0));
    
    // Transmittance
    vec3 Fex = exp(-(betaR * sR + betaM * sM));
    
    // in-scattering
    float cosTheta2 = dot(direction, sunPosition);
    float rPhase = rayleighPhase(cosTheta2 * 0.5 + 0.5);
    vec3 betaRTheta = betaR * rPhase;
    
    float mPhase = miePhase(mieDirectionalG, cosTheta2);
    vec3 betaMTheta = betaM * mPhase;
    
    vec3 Lin = pow(3E22 * ((betaRTheta + betaMTheta) / (betaR + betaM)) * 
               (1.0 - Fex), vec3(1.5));
    Lin *= mix(vec3(1.0), 
               pow(3E22 * ((betaRTheta + betaMTheta) / (betaR + betaM)) * Fex, vec3(0.5)),
               clamp(pow(1.0 - dot(up, sunPosition), 5.0), 0.0, 1.0));
    
    // Nighttime sky color
    vec3 L0 = vec3(0.1) * Fex;
    
    // Sun disk
    float sundisk = smoothstep(sunAngularDiameterCos, sunAngularDiameterCos + 0.00002, cosTheta2);
    L0 += (1000.0 * Fex) * sundisk;
    
    // Tone map
    vec3 texColor = (Lin + L0) * 0.04 + vec3(0.0, 0.0003, 0.00075);
    vec3 curr = vec3(1.0) - exp(-1.0 * texColor);
    vec3 retColor = pow(curr, vec3(1.0 / 2.2));
    
    gl_FragColor = vec4(retColor, 1.0);
  }
`;
```

## 2. Sky Class

```javascript
class PhysicalSky {
  constructor(scene) {
    this.scene = scene;
    this.sunPosition = new THREE.Vector3(0, 1, 0);
    this.elevation = 45; // degrees above horizon
    this.azimuth = 180;  // degrees (0=North, 90=East)
    
    this._createSkyMesh();
    this._createSun();
  }
  
  _createSkyMesh() {
    const skyGeo = new THREE.SphereGeometry(450, 32, 15);
    
    this.skyMaterial = new THREE.ShaderMaterial({
      name: 'SkyShader',
      fragmentShader: skyFragmentShader,
      vertexShader: skyVertexShader,
      uniforms: {
        turbidity: { value: 10 },
        rayleigh: { value: 3 },
        mieCoefficient: { value: 0.005 },
        mieDirectionalG: { value: 0.7 },
        sunPosition: { value: new THREE.Vector3() }
      },
      side: THREE.BackSide,
      depthWrite: false
    });
    
    this.skyMesh = new THREE.Mesh(skyGeo, this.skyMaterial);
    this.scene.add(this.skyMesh);
  }
  
  _createSun() {
    const sunGeo = new THREE.SphereGeometry(2, 16, 16);
    const sunMat = new THREE.MeshBasicMaterial({
      color: 0xfffae0,
      fog: false
    });
    this.sunSphere = new THREE.Mesh(sunGeo, sunMat);
    this.scene.add(this.sunSphere);
  }
  
  setTime(hours, minutes = 0) {
    // hours: 0-24
    const timeOfDay = (hours + minutes / 60) / 24; // 0-1
    
    // Sun elevation: ขึ้นตอนเช้า ตกตอนเย็น
    const dayFraction = Math.sin(timeOfDay * Math.PI * 2 - Math.PI / 2);
    this.elevation = dayFraction * 90; // -90 to 90
    
    // Azimuth: หมุนรอบจากตะวันออกไปตะวันตก
    this.azimuth = (timeOfDay * 360) % 360;
    
    this.updateSun();
  }
  
  updateSun() {
    const phi = THREE.MathUtils.degToRad(90 - this.elevation);
    const theta = THREE.MathUtils.degToRad(this.azimuth);
    
    this.sunPosition.setFromSphericalCoords(1, phi, theta);
    this.skyMaterial.uniforms.sunPosition.value.copy(this.sunPosition);
    
    // Move sun sphere
    this.sunSphere.position.copy(this.sunPosition).multiplyScalar(400);
    this.sunSphere.visible = this.elevation > -5;
    
    // Update scene lighting
    if (this.sun) {
      this.sun.position.copy(this.sunPosition).multiplyScalar(100);
      const intensity = Math.max(0, this.sunPosition.y) * 5;
      this.sun.intensity = intensity;
      
      // Sunrise/sunset color
      const t = Math.max(0, this.sunPosition.y);
      this.sun.color.setHSL(0.1 * (1 - t) + 0.15, 0.8, 0.5 + t * 0.5);
    }
  }
  
  attachLight(directionalLight) {
    this.sun = directionalLight;
    this.updateSun();
  }
  
  setTurbidity(value) {
    this.skyMaterial.uniforms.turbidity.value = value;
  }
  
  setRayleigh(value) {
    this.skyMaterial.uniforms.rayleigh.value = value;
  }
}
```

## 3. Procedural Cloud System

```javascript
class CloudSystem {
  constructor(scene) {
    this.scene = scene;
    this.clouds = [];
    this.cloudMeshes = new THREE.Group();
    scene.add(this.cloudMeshes);
  }
  
  // Cloud shader ใช้ noise
  _createCloudMaterial() {
    return new THREE.ShaderMaterial({
      uniforms: {
        time: { value: 0 },
        color: { value: new THREE.Color(1, 1, 1) },
        opacity: { value: 0.85 }
      },
      vertexShader: `
        varying vec2 vUv;
        varying vec3 vPos;
        void main() {
          vUv = uv;
          vPos = position;
          gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
      `,
      fragmentShader: `
        uniform float time;
        uniform vec3 color;
        uniform float opacity;
        varying vec2 vUv;
        varying vec3 vPos;
        
        float hash(vec2 p) {
          return fract(sin(dot(p, vec2(127.3, 311.7))) * 43758.5453);
        }
        
        float noise(vec2 p) {
          vec2 i = floor(p);
          vec2 f = fract(p);
          vec2 u = f * f * (3.0 - 2.0 * f);
          return mix(mix(hash(i), hash(i + vec2(1,0)), u.x),
                     mix(hash(i + vec2(0,1)), hash(i + vec2(1,1)), u.x), u.y);
        }
        
        float fbm(vec2 p) {
          float v = 0.0, a = 0.5;
          for (int i = 0; i < 5; i++) {
            v += noise(p) * a;
            p *= 2.1; a *= 0.5;
          }
          return v;
        }
        
        void main() {
          vec2 uv = vUv - 0.5;
          
          // Radial fade at edges
          float radial = 1.0 - smoothstep(0.3, 0.5, length(uv));
          
          // Animated noise
          vec2 noiseUv = vUv * 3.0 + vec2(time * 0.01, 0.0);
          float cloud = fbm(noiseUv);
          cloud = smoothstep(0.4, 0.7, cloud);
          
          float alpha = cloud * radial * opacity;
          if (alpha < 0.01) discard;
          
          gl_FragColor = vec4(color * (0.8 + cloud * 0.2), alpha);
        }
      `,
      transparent: true,
      depthWrite: false,
      side: THREE.DoubleSide
    });
  }
  
  // สร้างเมฆเดี่ยว
  createCloud(x, y, z, scale = 1) {
    const group = new THREE.Group();
    group.position.set(x, y, z);
    
    const mat = this._createCloudMaterial();
    
    // หลาย billboard puff ซ้อนกัน
    const puffCount = 5 + Math.floor(Math.random() * 8);
    for (let i = 0; i < puffCount; i++) {
      const size = (2 + Math.random() * 4) * scale;
      const geo = new THREE.PlaneGeometry(size, size * 0.6);
      const mesh = new THREE.Mesh(geo, mat.clone());
      
      mesh.position.set(
        (Math.random() - 0.5) * 6 * scale,
        (Math.random() - 0.5) * 1.5 * scale,
        (Math.random() - 0.5) * 3 * scale
      );
      mesh.rotation.y = Math.random() * Math.PI;
      group.add(mesh);
    }
    
    this.clouds.push({ group, speed: 0.5 + Math.random() });
    this.cloudMeshes.add(group);
    return group;
  }
  
  generateClouds(count = 20) {
    for (let i = 0; i < count; i++) {
      this.createCloud(
        (Math.random() - 0.5) * 400,
        50 + Math.random() * 40,
        (Math.random() - 0.5) * 400,
        1 + Math.random() * 2
      );
    }
  }
  
  // Billboard: หันหน้าหา camera เสมอ
  update(camera, time) {
    this.cloudMeshes.children.forEach((group, i) => {
      const cloud = this.clouds[i];
      if (!cloud) return;
      
      // เคลื่อนที่ตามลม
      group.position.x += cloud.speed * 0.05;
      if (group.position.x > 200) group.position.x = -200;
      
      // Billboard children
      group.children.forEach(mesh => {
        if (mesh.material.uniforms) {
          mesh.material.uniforms.time.value = time;
        }
      });
      
      // Face camera
      group.lookAt(camera.position.x, group.position.y, camera.position.z);
    });
  }
}
```

## 4. Star System

```javascript
class StarField {
  constructor(scene, count = 5000) {
    this.scene = scene;
    this._create(count);
  }
  
  _create(count) {
    const positions = new Float32Array(count * 3);
    const colors = new Float32Array(count * 3);
    const sizes = new Float32Array(count);
    
    // Star colors: O-type (blue), B (blue-white), A (white), 
    //              F (yellow-white), G (yellow, sun), K (orange), M (red)
    const starColors = [
      [0.6, 0.7, 1.0],  // blue
      [0.8, 0.85, 1.0], // blue-white
      [1.0, 1.0, 1.0],  // white
      [1.0, 0.95, 0.8], // yellow-white
      [1.0, 0.9, 0.6],  // yellow
      [1.0, 0.7, 0.4],  // orange
      [1.0, 0.4, 0.3],  // red
    ];
    
    for (let i = 0; i < count; i++) {
      // สุ่มจุดบน sphere
      const theta = Math.acos(2 * Math.random() - 1);
      const phi = 2 * Math.PI * Math.random();
      const r = 400;
      
      positions[i * 3] = r * Math.sin(theta) * Math.cos(phi);
      positions[i * 3 + 1] = r * Math.sin(theta) * Math.sin(phi);
      positions[i * 3 + 2] = r * Math.cos(theta);
      
      const col = starColors[Math.floor(Math.random() * starColors.length)];
      colors[i * 3] = col[0];
      colors[i * 3 + 1] = col[1];
      colors[i * 3 + 2] = col[2];
      
      sizes[i] = 0.5 + Math.random() * 2;
    }
    
    const geo = new THREE.BufferGeometry();
    geo.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));
    geo.setAttribute('color', new THREE.Float32BufferAttribute(colors, 3));
    geo.setAttribute('size', new THREE.Float32BufferAttribute(sizes, 1));
    
    const mat = new THREE.ShaderMaterial({
      uniforms: {
        time: { value: 0 },
        opacity: { value: 0 }
      },
      vertexShader: `
        attribute float size;
        attribute vec3 color;
        uniform float time;
        varying vec3 vColor;
        varying float vTwinkle;
        
        float hash(float n) { return fract(sin(n) * 43758.5453); }
        
        void main() {
          vColor = color;
          
          // Twinkle
          float twinkle = sin(time * 2.0 + hash(float(gl_VertexID)) * 100.0) * 0.3 + 0.7;
          vTwinkle = twinkle;
          
          gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
          gl_PointSize = size * twinkle;
        }
      `,
      fragmentShader: `
        uniform float opacity;
        varying vec3 vColor;
        varying float vTwinkle;
        
        void main() {
          vec2 uv = gl_PointCoord - 0.5;
          float d = length(uv);
          float star = 1.0 - smoothstep(0.0, 0.5, d);
          
          if (star < 0.01) discard;
          
          gl_FragColor = vec4(vColor * vTwinkle, star * opacity);
        }
      `,
      transparent: true,
      depthWrite: false,
      blending: THREE.AdditiveBlending,
      vertexColors: true
    });
    
    this.points = new THREE.Points(geo, mat);
    this.scene.add(this.points);
    this.material = mat;
  }
  
  // visibility: 0 = 낮, 1 = กลางคืน
  setVisibility(skyElevation) {
    const opacity = Math.max(0, -skyElevation / 0.3);
    this.material.uniforms.opacity.value = Math.min(1, opacity);
  }
  
  update(time) {
    this.material.uniforms.time.value = time;
    // Slow rotation
    this.points.rotation.y = time * 0.0001;
  }
}
```

## 5. Day/Night Cycle Controller

```javascript
class DayNightCycle {
  constructor(scene, renderer) {
    this.scene = scene;
    this.renderer = renderer;
    this.time = 12; // เริ่มที่เที่ยงวัน (0-24)
    this.timeSpeed = 0.5; // ชั่วโมงต่อวินาที
    
    this._setup();
  }
  
  _setup() {
    // Physical Sky
    this.sky = new PhysicalSky(this.scene);
    
    // Sun light
    this.sunLight = new THREE.DirectionalLight(0xfff8e0, 4);
    this.sunLight.castShadow = true;
    this.sunLight.shadow.mapSize.set(2048, 2048);
    this.sunLight.shadow.camera.near = 1;
    this.sunLight.shadow.camera.far = 500;
    this.sunLight.shadow.camera.left = -100;
    this.sunLight.shadow.camera.right = 100;
    this.sunLight.shadow.camera.top = 100;
    this.sunLight.shadow.camera.bottom = -100;
    this.scene.add(this.sunLight);
    this.sky.attachLight(this.sunLight);
    
    // Moon light (สีเย็นกว่า)
    this.moonLight = new THREE.DirectionalLight(0x8899cc, 0.3);
    this.scene.add(this.moonLight);
    
    // Ambient light
    this.ambientLight = new THREE.AmbientLight(0x334466, 0.5);
    this.scene.add(this.ambientLight);
    
    // Moon sprite
    const moonTex = this._createMoonTexture();
    const moonMat = new THREE.SpriteMaterial({ map: moonTex });
    this.moonSprite = new THREE.Sprite(moonMat);
    this.moonSprite.scale.set(15, 15, 1);
    this.scene.add(this.moonSprite);
    
    // Stars
    this.stars = new StarField(this.scene, 3000);
    
    // Clouds
    this.clouds = new CloudSystem(this.scene);
    this.clouds.generateClouds(15);
  }
  
  _createMoonTexture() {
    const canvas = document.createElement('canvas');
    canvas.width = canvas.height = 128;
    const ctx = canvas.getContext('2d');
    
    const grad = ctx.createRadialGradient(64, 64, 0, 64, 64, 64);
    grad.addColorStop(0, 'rgba(230,230,200,1)');
    grad.addColorStop(0.7, 'rgba(200,200,180,0.8)');
    grad.addColorStop(1, 'rgba(180,180,160,0)');
    ctx.fillStyle = grad;
    ctx.fillRect(0, 0, 128, 128);
    
    // Moon craters
    ctx.fillStyle = 'rgba(150,150,130,0.4)';
    [[40, 45, 8], [75, 60, 6], [55, 80, 10], [85, 35, 5]].forEach(([x, y, r]) => {
      ctx.beginPath();
      ctx.arc(x, y, r, 0, Math.PI * 2);
      ctx.fill();
    });
    
    return new THREE.CanvasTexture(canvas);
  }
  
  update(deltaTime, camera) {
    this.time = (this.time + deltaTime * this.timeSpeed) % 24;
    
    // Update sky
    this.sky.setTime(this.time);
    
    // Moon position (ตรงข้ามดวงอาทิตย์)
    const sunPos = this.sky.sunPosition;
    this.moonLight.position.set(-sunPos.x, -sunPos.y, -sunPos.z).normalize().multiplyScalar(100);
    this.moonSprite.position.copy(this.moonLight.position).multiplyScalar(4);
    
    // Stars visibility
    this.stars.setVisibility(sunPos.y);
    this.stars.update(this.time * 100);
    
    // Ambient light color by time of day
    const t = Math.max(0, sunPos.y);
    this.ambientLight.color.setHSL(0.6, 0.3, 0.05 + t * 0.15);
    this.ambientLight.intensity = 0.3 + t * 1.5;
    
    // Fog color
    const nightColor = new THREE.Color(0x050510);
    const dayColor = new THREE.Color(0x8ab8d8);
    const fogColor = nightColor.clone().lerp(dayColor, t);
    this.scene.fog?.color?.copy(fogColor);
    this.scene.background = fogColor;
    
    // Clouds
    this.clouds.update(camera, this.time * 100);
  }
}
```

## 6. Complete Demo: Day/Night Cycle Scene

```html
<!DOCTYPE html>
<html>
<head>
  <title>Day/Night Cycle</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { background: #000; overflow: hidden; }
    canvas { display: block; }
    #ui {
      position: fixed;
      bottom: 20px;
      left: 50%;
      transform: translateX(-50%);
      background: rgba(0,0,0,0.5);
      color: #ddf;
      padding: 15px 25px;
      border-radius: 8px;
      font-family: Arial, sans-serif;
      text-align: center;
      min-width: 320px;
    }
    #ui label { display: block; margin: 6px 0; font-size: 14px; }
    #ui input[type=range] { width: 200px; vertical-align: middle; }
    #timeDisplay { font-size: 22px; font-weight: bold; color: #ffd; margin-bottom: 10px; }
  </style>
</head>
<body>
<div id="ui">
  <div id="timeDisplay">12:00</div>
  <label>Time: <input type="range" id="timeSlider" min="0" max="24" step="0.1" value="12"></label>
  <label>Speed: <input type="range" id="speedSlider" min="0" max="5" step="0.1" value="0.5"> <span id="speedVal">0.5</span>x</label>
  <label>Turbidity: <input type="range" id="turbSlider" min="1" max="20" step="0.1" value="10"> <span id="turbVal">10</span></label>
  <label>Auto: <input type="checkbox" id="autoPlay" checked></label>
</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, innerWidth/innerHeight, 0.1, 1000);
camera.position.set(0, 5, 20);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
renderer.outputColorSpace = THREE.SRGBColorSpace;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 0.5;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.maxPolarAngle = Math.PI * 0.6;

// ===== SIMPLE NOISE =====
function hash21(p) {
  return Math.abs(Math.sin(p[0] * 127.3 + p[1] * 311.7) * 43758.5453 % 1);
}

function smoothNoise(x, y) {
  const ix = Math.floor(x), iy = Math.floor(y);
  const fx = x - ix, fy = y - iy;
  const ux = fx*fx*(3-2*fx), uy = fy*fy*(3-2*fy);
  const a = hash21([ix, iy]);
  const b = hash21([ix+1, iy]);
  const c = hash21([ix, iy+1]);
  const d = hash21([ix+1, iy+1]);
  return a + (b-a)*ux + (c-a)*uy + (a-b-c+d)*ux*uy;
}

function fbm(x, y) {
  let v = 0, a = 0.5, f = 1;
  for (let i = 0; i < 5; i++) { v += smoothNoise(x*f,y*f)*a; a*=0.5; f*=2; }
  return v;
}

// ===== TERRAIN =====
const terrainGeo = new THREE.PlaneGeometry(200, 200, 100, 100);
terrainGeo.rotateX(-Math.PI / 2);

const pos = terrainGeo.attributes.position;
for (let i = 0; i < pos.count; i++) {
  const x = pos.getX(i), z = pos.getZ(i);
  const h = fbm(x * 0.02, z * 0.02) * 10 + smoothNoise(x * 0.05, z * 0.05) * 3;
  pos.setY(i, h);
}
terrainGeo.computeVertexNormals();

const terrainMat = new THREE.MeshStandardMaterial({
  color: 0x3a7a3a, roughness: 0.9, metalness: 0
});
const terrain = new THREE.Mesh(terrainGeo, terrainMat);
terrain.receiveShadow = true;
scene.add(terrain);

// Trees
function addTree(x, z) {
  const h = fbm(x * 0.02, z * 0.02) * 10 + smoothNoise(x * 0.05, z * 0.05) * 3;
  const g = new THREE.Group();
  g.position.set(x, h, z);

  const trunk = new THREE.Mesh(
    new THREE.CylinderGeometry(0.2, 0.3, 3, 6),
    new THREE.MeshStandardMaterial({ color: 0x4a2f0a })
  );
  trunk.position.y = 1.5;
  trunk.castShadow = true;
  g.add(trunk);

  for (let i = 0; i < 3; i++) {
    const cone = new THREE.Mesh(
      new THREE.ConeGeometry(1.5 - i * 0.3, 2, 7),
      new THREE.MeshStandardMaterial({ color: 0x1a5a1a })
    );
    cone.position.y = 3.5 + i * 1.2;
    cone.castShadow = true;
    g.add(cone);
  }
  scene.add(g);
}

for (let i = 0; i < 60; i++) {
  addTree((Math.random() - 0.5) * 80, (Math.random() - 0.5) * 80);
}

// Rocks
for (let i = 0; i < 30; i++) {
  const x = (Math.random() - 0.5) * 100;
  const z = (Math.random() - 0.5) * 100;
  const h = fbm(x * 0.02, z * 0.02) * 10 + smoothNoise(x * 0.05, z * 0.05) * 3;
  const rock = new THREE.Mesh(
    new THREE.DodecahedronGeometry(0.5 + Math.random() * 1.5, 0),
    new THREE.MeshStandardMaterial({ color: 0x888880, roughness: 1 })
  );
  rock.position.set(x, h + 0.3, z);
  rock.rotation.set(Math.random(), Math.random(), Math.random());
  rock.castShadow = true;
  rock.receiveShadow = true;
  scene.add(rock);
}

// ===== SKY SYSTEM =====
// Simplified sky using gradient shader
const skyGeo = new THREE.SphereGeometry(450, 32, 15);
const skyMat = new THREE.ShaderMaterial({
  uniforms: {
    sunDir: { value: new THREE.Vector3(0, 1, 0) },
    sunColor: { value: new THREE.Color(1, 0.9, 0.6) },
    skyTop: { value: new THREE.Color(0.1, 0.3, 0.7) },
    skyHorizon: { value: new THREE.Color(0.7, 0.85, 1.0) },
    nightColor: { value: new THREE.Color(0.02, 0.03, 0.1) },
    timeT: { value: 1.0 }
  },
  vertexShader: `
    varying vec3 vDir;
    void main() {
      vDir = normalize(position);
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    uniform vec3 sunDir;
    uniform vec3 sunColor;
    uniform vec3 skyTop;
    uniform vec3 skyHorizon;
    uniform vec3 nightColor;
    uniform float timeT;
    varying vec3 vDir;
    
    void main() {
      float up = dot(vDir, vec3(0,1,0));
      
      // Sky gradient (day)
      vec3 daySky = mix(skyHorizon, skyTop, max(0.0, up));
      
      // Night sky
      vec3 night = nightColor;
      
      // Sun glow
      float sunAngle = dot(vDir, sunDir);
      float sunDisk = smoothstep(0.998, 0.9999, sunAngle);
      float sunGlow = pow(max(0.0, sunAngle), 32.0) * 0.4;
      
      // Sunrise/sunset color at horizon
      float horizonFactor = pow(max(0.0, 1.0 - abs(up)), 3.0);
      float sunsetFactor = pow(max(0.0, sunDir.y * 2.0 + 0.5), 1.5);
      vec3 sunsetColor = vec3(1.0, 0.4, 0.1) * horizonFactor * (1.0 - sunsetFactor);
      
      // Blend day/night
      vec3 sky = mix(night, daySky, timeT);
      sky += sunGlow * sunColor * timeT;
      sky += sunsetColor * 0.5;
      sky += sunDisk * 3.0 * sunColor;
      
      gl_FragColor = vec4(sky, 1.0);
    }
  `,
  side: THREE.BackSide,
  depthWrite: false
});
const skyMesh = new THREE.Mesh(skyGeo, skyMat);
scene.add(skyMesh);

// Sun
const sunMesh = new THREE.Mesh(
  new THREE.SphereGeometry(3, 16, 16),
  new THREE.MeshBasicMaterial({ color: 0xfffae0 })
);
scene.add(sunMesh);

// Stars
const starsGeo = new THREE.BufferGeometry();
const starPos = new Float32Array(3000 * 3);
const starSizes = new Float32Array(3000);
for (let i = 0; i < 3000; i++) {
  const theta = Math.acos(2 * Math.random() - 1);
  const phi = 2 * Math.PI * Math.random();
  const r = 420;
  starPos[i*3] = r * Math.sin(theta) * Math.cos(phi);
  starPos[i*3+1] = r * Math.sin(theta) * Math.sin(phi);
  starPos[i*3+2] = r * Math.cos(theta);
  starSizes[i] = 0.5 + Math.random() * 2;
}
starsGeo.setAttribute('position', new THREE.Float32BufferAttribute(starPos, 3));
starsGeo.setAttribute('size', new THREE.Float32BufferAttribute(starSizes, 1));
const starsMat = new THREE.ShaderMaterial({
  uniforms: { opacity: { value: 0 }, time: { value: 0 } },
  vertexShader: `
    attribute float size;
    uniform float time;
    void main() {
      float twinkle = sin(time * 2.0 + float(gl_VertexID) * 1.37) * 0.2 + 0.8;
      gl_PointSize = size * twinkle;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    uniform float opacity;
    void main() {
      float d = length(gl_PointCoord - 0.5);
      float s = 1.0 - smoothstep(0.0, 0.5, d);
      if (s < 0.01) discard;
      gl_FragColor = vec4(vec3(0.9, 0.95, 1.0), s * opacity);
    }
  `,
  transparent: true, depthWrite: false, blending: THREE.AdditiveBlending
});
const stars = new THREE.Points(starsGeo, starsMat);
scene.add(stars);

// Lights
const sunLight = new THREE.DirectionalLight(0xfffae0, 4);
sunLight.castShadow = true;
sunLight.shadow.mapSize.set(2048, 2048);
sunLight.shadow.camera.left = -80;
sunLight.shadow.camera.right = 80;
sunLight.shadow.camera.top = 80;
sunLight.shadow.camera.bottom = -80;
scene.add(sunLight);
const ambLight = new THREE.AmbientLight(0x334466, 1);
scene.add(ambLight);

// Fog
scene.fog = new THREE.Fog(0x8ab8d8, 80, 250);

// ===== CLOUDS =====
const cloudGroup = new THREE.Group();
scene.add(cloudGroup);

for (let i = 0; i < 20; i++) {
  const cloud = new THREE.Group();
  cloud.position.set(
    (Math.random() - 0.5) * 300,
    40 + Math.random() * 30,
    (Math.random() - 0.5) * 300
  );
  cloud.userData.speed = 0.2 + Math.random() * 0.5;

  for (let j = 0; j < 4; j++) {
    const puff = new THREE.Mesh(
      new THREE.SphereGeometry(3 + Math.random() * 5, 8, 6),
      new THREE.MeshStandardMaterial({ color: 0xffffff, transparent: true, opacity: 0.85 })
    );
    puff.position.set((j-2)*4 + Math.random()*3, Math.random()*2, Math.random()*3);
    cloud.add(puff);
  }
  cloudGroup.add(cloud);
}

// ===== ANIMATION =====
let currentTime = 12;
let autoPlay = true;
let timeSpeed = 0.5;
let turbidity = 10;

const timeSlider = document.getElementById('timeSlider');
const speedSlider = document.getElementById('speedSlider');
const turbSlider = document.getElementById('turbSlider');
const autoCheckbox = document.getElementById('autoPlay');
const timeDisplay = document.getElementById('timeDisplay');

timeSlider.addEventListener('input', () => { currentTime = parseFloat(timeSlider.value); });
speedSlider.addEventListener('input', () => {
  timeSpeed = parseFloat(speedSlider.value);
  document.getElementById('speedVal').textContent = timeSpeed.toFixed(1);
});
turbSlider.addEventListener('input', () => {
  turbidity = parseFloat(turbSlider.value);
  document.getElementById('turbVal').textContent = turbidity.toFixed(1);
});
autoCheckbox.addEventListener('change', () => { autoPlay = autoCheckbox.checked; });

function updateSky(timeOfDay) {
  // Sun position
  const fraction = timeOfDay / 24;
  const sunPhi = fraction * Math.PI * 2 - Math.PI / 2;
  const sunX = Math.cos(sunPhi) * 400;
  const sunY = Math.sin(sunPhi) * 400;
  const sunZ = 0;

  sunMesh.position.set(sunX, sunY, sunZ);
  sunLight.position.set(sunX, sunY, sunZ);
  sunLight.visible = sunY > -30;

  const sunDir = new THREE.Vector3(sunX, sunY, sunZ).normalize();
  skyMat.uniforms.sunDir.value.copy(sunDir);

  const t = Math.max(0, Math.min(1, (sunY / 400 + 0.1) / 0.2));
  skyMat.uniforms.timeT.value = t;

  // Sunrise/sunset colors
  const elevation = sunY / 400;
  if (elevation > 0.05) {
    sunLight.color.set(0xfff8e0);
    sunLight.intensity = elevation * 5;
    terrainMat.color.setRGB(0.22, 0.48, 0.22);
    ambLight.color.set(0x446688);
    ambLight.intensity = 0.5 + elevation;
  } else if (elevation > -0.1) {
    // Golden hour
    sunLight.color.setRGB(1, 0.5, 0.1);
    sunLight.intensity = Math.max(0, elevation * 15);
    terrainMat.color.setRGB(0.5, 0.35, 0.15);
    ambLight.color.set(0x664422);
    ambLight.intensity = 0.3;
  } else {
    sunLight.intensity = 0;
    terrainMat.color.setRGB(0.05, 0.06, 0.08);
    ambLight.color.set(0x223355);
    ambLight.intensity = 0.15;
  }

  // Stars
  starsMat.uniforms.opacity.value = Math.max(0, Math.min(1, (-elevation - 0.05) / 0.15));

  // Fog
  const fogColor = new THREE.Color().setHSL(0.6, 0.3, 0.05 + t * 0.3);
  scene.fog.color.copy(fogColor);
  renderer.setClearColor(fogColor);
}

const clock = new THREE.Clock();

function animate() {
  requestAnimationFrame(animate);
  const dt = clock.getDelta();
  const elapsed = clock.elapsedTime;

  if (autoPlay) {
    currentTime = (currentTime + dt * timeSpeed / 6) % 24;
    timeSlider.value = currentTime;
  }

  const hh = Math.floor(currentTime);
  const mm = Math.floor((currentTime - hh) * 60);
  timeDisplay.textContent = `${String(hh).padStart(2,'0')}:${String(mm).padStart(2,'0')}`;

  updateSky(currentTime);

  // Clouds
  cloudGroup.children.forEach(cloud => {
    cloud.position.x += cloud.userData.speed * 0.05;
    if (cloud.position.x > 200) cloud.position.x = -200;
  });

  starsMat.uniforms.time.value = elapsed;
  stars.rotation.y = elapsed * 0.00003;

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

ระบบท้องฟ้าที่สมบูรณ์:
1. **Physical Sky** - Rayleigh/Mie scattering shader
2. **Clouds** - Billboard-based procedural clouds
3. **Stars** - Particle system พร้อม twinkle effect
4. **Day/Night Cycle** - ควบคุมเวลาด้วย slider
5. **Dynamic Lighting** - ปรับสีและความเข้มตามเวลา

---

## แบบฝึกหัด

1. **Easy**: สร้าง sky box ด้วยสีเปลี่ยนตาม time of day
2. **Medium**: เพิ่ม lens flare รอบดวงอาทิตย์
3. **Hard**: Implement Preetham sky model แบบเต็ม
4. **Challenge**: Volume cloud ด้วย raymarching GLSL
