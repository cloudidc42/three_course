# Part 42: Glass, Water & Special Materials ใน Three.js

## บทนำ

Three.js MeshPhysicalMaterial เป็น material ที่ทรงพลังที่สุดสำหรับ physically-based rendering รองรับ:
- **Glass** - การส่องแสงผ่าน (transmission) และการหักเหแสง
- **Water** - คลื่น, Fresnel, reflection
- **Iridescence** - สีสันแบบ thin-film (สีมุก, แมลงทับ)
- **Subsurface Scattering** - แสงกระจายใต้ผิว (ผิวหนัง, หิน)
- **Anisotropy** - การสะท้อนแบบ brushed metal
- **Clearcoat** - เคลือบเงาสำหรับ car paint

## 1. Glass Material (MeshPhysicalMaterial)

```javascript
import * as THREE from 'three';

// ===== Glass Properties =====
const glassMaterial = new THREE.MeshPhysicalMaterial({
  // TRANSMISSION - ความโปร่งแสง (ไม่เหมือน opacity)
  transmission: 1.0,      // 0 = opaque, 1 = fully transmissive
  
  // INDEX OF REFRACTION
  ior: 1.5,               // 1.0=air, 1.33=water, 1.5=glass, 2.42=diamond
  
  // THICKNESS - ความหนา (กระทบต่อ chromatic aberration)
  thickness: 2.0,
  
  // SURFACE QUALITY
  roughness: 0.0,
  metalness: 0.0,
  
  // ATTENUATION (การดูดซับสี ตามความลึก)
  attenuationColor: new THREE.Color(0x88ddff),
  attenuationDistance: 3.0,  // เมตร
  
  // CLEARCOAT
  clearcoat: 1.0,
  clearcoatRoughness: 0.0,
  
  // SPECULAR
  specularIntensity: 1.0,
  specularColor: new THREE.Color(0xffffff),
  
  // ต้องใช้ FrontSide
  side: THREE.FrontSide,
  
  // NOTE: ไม่ใช้ transparent: true กับ transmission
});

// ===== Frosted Glass =====
const frostedGlass = new THREE.MeshPhysicalMaterial({
  transmission: 0.9,
  ior: 1.5,
  roughness: 0.4,         // เพิ่ม roughness = more frosted
  thickness: 0.5,
  side: THREE.FrontSide
});

// ===== Stained Glass (สีกระจก) =====
const stainedGlass = new THREE.MeshPhysicalMaterial({
  transmission: 0.9,
  ior: 1.5,
  color: new THREE.Color(0x8822ff), // สีม่วง
  roughness: 0.0,
  emissive: new THREE.Color(0x440088),
  emissiveIntensity: 0.2,
  side: THREE.FrontSide
});
```

## 2. Water Shader

```javascript
// ===== Water Normal Map Animation =====
// ใช้ 2 normal maps สลับกันเพื่อสร้าง water effect

const waterMaterial = new THREE.MeshStandardMaterial({
  color: 0x1188cc,
  roughness: 0.1,
  metalness: 0.0,
  transparent: true,
  opacity: 0.85,
  side: THREE.FrontSide
});

// Animated normal maps
const normalMap1 = textureLoader.load('/textures/water_normal_1.jpg');
const normalMap2 = textureLoader.load('/textures/water_normal_2.jpg');

normalMap1.wrapS = normalMap1.wrapT = THREE.RepeatWrapping;
normalMap2.wrapS = normalMap2.wrapT = THREE.RepeatWrapping;
normalMap1.repeat.set(4, 4);
normalMap2.repeat.set(4, 4);

// Custom water shader
const waterShaderMaterial = new THREE.ShaderMaterial({
  uniforms: {
    time: { value: 0 },
    normalMap1: { value: normalMap1 },
    normalMap2: { value: normalMap2 },
    waterColor: { value: new THREE.Color(0x1166aa) },
    deepColor: { value: new THREE.Color(0x003366) },
    lightDir: { value: new THREE.Vector3(1, 2, 1).normalize() },
    envMap: { value: null }
  },
  
  vertexShader: `
    uniform float time;
    varying vec2 vUv;
    varying vec3 vNormal;
    varying vec3 vViewDir;
    varying vec3 vWorldPos;
    
    void main() {
      vUv = uv;
      
      // Wave displacement
      vec3 pos = position;
      float wave1 = sin(pos.x * 2.0 + time * 1.5) * 0.1;
      float wave2 = sin(pos.y * 1.5 + time * 0.8) * 0.08;
      float wave3 = cos(pos.x * 0.5 + pos.y * 0.7 + time) * 0.05;
      pos.z += wave1 + wave2 + wave3;
      
      vec4 worldPos = modelMatrix * vec4(pos, 1.0);
      vWorldPos = worldPos.xyz;
      vViewDir = normalize(cameraPosition - worldPos.xyz);
      vNormal = normalize(normalMatrix * normal);
      
      gl_Position = projectionMatrix * viewMatrix * worldPos;
    }
  `,
  
  fragmentShader: `
    uniform float time;
    uniform sampler2D normalMap1;
    uniform sampler2D normalMap2;
    uniform vec3 waterColor;
    uniform vec3 deepColor;
    uniform vec3 lightDir;
    
    varying vec2 vUv;
    varying vec3 vNormal;
    varying vec3 vViewDir;
    varying vec3 vWorldPos;
    
    void main() {
      // Animated UV for normal maps
      vec2 uv1 = vUv * 4.0 + vec2(time * 0.03, time * 0.02);
      vec2 uv2 = vUv * 6.0 + vec2(-time * 0.02, time * 0.04);
      
      // Sample normal maps
      vec3 n1 = texture2D(normalMap1, uv1).rgb * 2.0 - 1.0;
      vec3 n2 = texture2D(normalMap2, uv2).rgb * 2.0 - 1.0;
      
      // Blend normals
      vec3 N = normalize(n1 + n2 + vNormal);
      vec3 V = normalize(vViewDir);
      vec3 L = normalize(lightDir);
      
      // Fresnel
      float fresnel = pow(1.0 - max(dot(N, V), 0.0), 3.0);
      
      // Specular
      vec3 H = normalize(L + V);
      float specular = pow(max(dot(N, H), 0.0), 128.0);
      
      // Depth-based color
      float depth = 0.5; // would be from depth buffer
      vec3 waterCol = mix(waterColor, deepColor, depth);
      
      // Combine
      vec3 finalColor = waterCol * 0.4 + specular * 1.5;
      finalColor = mix(finalColor, vec3(0.8, 0.9, 1.0), fresnel * 0.3);
      
      gl_FragColor = vec4(finalColor, 0.85 + fresnel * 0.15);
    }
  `,
  
  transparent: true,
  side: THREE.FrontSide
});
```

## 3. Iridescence (Thin Film)

```javascript
// ===== Iridescence - สีสันแบบ soap bubble / beetle / pearl =====
const iridescenceMaterial = new THREE.MeshPhysicalMaterial({
  color: 0xffffff,
  roughness: 0.1,
  metalness: 0.0,
  
  // Iridescence
  iridescence: 1.0,          // ความแรงของ iridescence
  iridescenceIOR: 1.8,        // IOR ของ thin film
  iridescenceThicknessRange: [100, 400], // ช่วงความหนา (nm)
  
  // Optional: use map for thickness variation
  // iridescenceThicknessMap: texture
});

// ===== Soap Bubble =====
const soapBubble = new THREE.MeshPhysicalMaterial({
  color: 0xffffff,
  roughness: 0.0,
  metalness: 0.0,
  transmission: 0.95,
  ior: 1.4,
  thickness: 0.1,
  iridescence: 1.0,
  iridescenceIOR: 1.3,
  iridescenceThicknessRange: [50, 250],
  side: THREE.DoubleSide,
  transparent: true,
  opacity: 0.3
});

// ===== Beetle Shell / Oil Slick =====
const oilSlick = new THREE.MeshPhysicalMaterial({
  color: 0x111111,
  roughness: 0.05,
  metalness: 1.0,
  iridescence: 1.0,
  iridescenceIOR: 2.0,
  iridescenceThicknessRange: [200, 800]
});
```

## 4. Subsurface Scattering (SSS)

```javascript
// ===== SSS ใน Three.js =====
// MeshPhysicalMaterial ไม่มี built-in SSS
// ต้องใช้ Custom Shader หรือ approximation

// Method 1: Cheap SSS approximation ด้วย color + emissive
function createSkinMaterial(skinColor) {
  return new THREE.MeshStandardMaterial({
    color: skinColor,
    roughness: 0.7,
    metalness: 0.0,
    
    // SSS approximation: ทำให้ผิวดู translucent เมื่อแสงส่องจากด้านหลัง
    // (ใช้ร่วมกับ custom shader จะดีกว่า)
  });
}

// Method 2: Custom SSS Shader
const sssMaterial = new THREE.ShaderMaterial({
  uniforms: {
    diffuseColor: { value: new THREE.Color(0xffccaa) },
    sssColor: { value: new THREE.Color(0xff8866) },
    sssStrength: { value: 0.5 },
    lightDir: { value: new THREE.Vector3(1, 1, 1).normalize() },
    lightColor: { value: new THREE.Color(0xffffff) },
    sssDistortion: { value: 0.2 },
    sssPower: { value: 8.0 }
  },
  
  vertexShader: `
    varying vec3 vNormal;
    varying vec3 vViewDir;
    varying vec3 vWorldPos;
    
    void main() {
      vec4 worldPos = modelMatrix * vec4(position, 1.0);
      vWorldPos = worldPos.xyz;
      vNormal = normalize(normalMatrix * normal);
      vViewDir = normalize(cameraPosition - worldPos.xyz);
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  
  fragmentShader: `
    uniform vec3 diffuseColor;
    uniform vec3 sssColor;
    uniform float sssStrength;
    uniform vec3 lightDir;
    uniform vec3 lightColor;
    uniform float sssDistortion;
    uniform float sssPower;
    
    varying vec3 vNormal;
    varying vec3 vViewDir;
    varying vec3 vWorldPos;
    
    void main() {
      vec3 N = normalize(vNormal);
      vec3 V = normalize(vViewDir);
      vec3 L = normalize(lightDir);
      
      // Diffuse
      float NdotL = max(dot(N, L), 0.0);
      vec3 diffuse = diffuseColor * lightColor * NdotL;
      
      // Subsurface Scattering approximation
      // Light wraps around to the dark side
      vec3 sssLight = L + N * sssDistortion;
      float sssDot = pow(max(dot(V, -sssLight), 0.0), sssPower);
      float sss = sssDot * sssStrength;
      
      // Translucency: light seen from behind
      float backlight = pow(max(dot(-L, V), 0.0), 2.0) * sssStrength * 0.5;
      
      vec3 finalColor = diffuse + (sssColor * sss) + (sssColor * backlight);
      
      // Ambient
      finalColor += diffuseColor * 0.1;
      
      gl_FragColor = vec4(finalColor, 1.0);
    }
  `
});
```

## 5. Anisotropy สำหรับ Brushed Metal

```javascript
// ===== Anisotropy - Brushed Metal =====
const brushedMetalMaterial = new THREE.MeshPhysicalMaterial({
  color: 0xcccccc,
  roughness: 0.3,
  metalness: 1.0,
  
  // Anisotropy: specular highlight ยาวในทิศทางเดียว
  anisotropy: 1.0,           // ความแรง (0 = isotropic, 1 = full anisotropic)
  anisotropyRotation: 0.0,   // ทิศทาง (radians)
  
  // Optional: anisotropy direction map
  // anisotropyMap: texture  // r = anisotropy strength, g = rotation
});

// ===== Circular Brushed Metal =====
const circularBrushedMetal = new THREE.MeshPhysicalMaterial({
  color: 0xaaaaaa,
  roughness: 0.4,
  metalness: 1.0,
  anisotropy: 0.8,
  // ใช้ anisotropy map สำหรับ circular pattern
});

// ===== Hair Material (Kajiya-Kay) =====
// Anisotropy ใช้กับ hair rendering ด้วย
const hairMaterial = new THREE.ShaderMaterial({
  uniforms: {
    hairColor: { value: new THREE.Color(0x4a2800) },
    specularColor: { value: new THREE.Color(0xfff8f0) },
    lightDir: { value: new THREE.Vector3(1, 1, 0).normalize() },
    specularShift: { value: 0.1 }
  },
  
  vertexShader: `
    varying vec3 vTangent;
    varying vec3 vViewDir;
    
    void main() {
      vec4 worldPos = modelMatrix * vec4(position, 1.0);
      vTangent = normalize(normalMatrix * tangent.xyz);
      vViewDir = normalize(cameraPosition - worldPos.xyz);
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  
  fragmentShader: `
    uniform vec3 hairColor;
    uniform vec3 specularColor;
    uniform vec3 lightDir;
    uniform float specularShift;
    
    varying vec3 vTangent;
    varying vec3 vViewDir;
    
    // Kajiya-Kay hair shading
    float kajiyaKay(vec3 T, vec3 L, float shift) {
      vec3 Ts = normalize(T + shift * vec3(0.0, 1.0, 0.0));
      return sqrt(1.0 - pow(dot(Ts, normalize(L)), 2.0));
    }
    
    void main() {
      vec3 T = normalize(vTangent);
      vec3 L = normalize(lightDir);
      vec3 V = normalize(vViewDir);
      
      // Diffuse (wrapped around)
      float diffuse = 0.5 + 0.5 * dot(L, T);
      
      // Specular (Kajiya-Kay)
      float spec = pow(kajiyaKay(T, L, specularShift), 16.0);
      
      vec3 finalColor = hairColor * diffuse + specularColor * spec;
      
      gl_FragColor = vec4(finalColor, 1.0);
    }
  `
});
```

## 6. Clearcoat สำหรับ Car Paint

```javascript
// ===== Car Paint with Clearcoat =====
const carPaintMaterial = new THREE.MeshPhysicalMaterial({
  // Base paint color
  color: 0xcc2200,
  roughness: 0.4,
  metalness: 0.0,
  
  // Clearcoat (เคลือบใส)
  clearcoat: 1.0,
  clearcoatRoughness: 0.05,  // 0 = mirror, 0.1+ = slight haze
  
  // Normal map สำหรับ base paint (paint flakes)
  // normalMap: flakeNormalMap,
  // normalScale: new THREE.Vector2(0.3, 0.3),
  
  // Clearcoat normal map (สำหรับ surface imperfections)
  // clearcoatNormalMap: clearcoatNormalMap,
  // clearcoatNormalScale: new THREE.Vector2(0.15, 0.15),
  
  envMapIntensity: 1.0
});

// ===== Metallic Flake Car Paint (Custom Shader) =====
const metallicFlakeMaterial = new THREE.ShaderMaterial({
  uniforms: {
    baseColor: { value: new THREE.Color(0x882200) },
    flakeColor: { value: new THREE.Color(0xffffff) },
    flakeScale: { value: 50.0 },
    flakeIntensity: { value: 0.3 },
    clearcoatIntensity: { value: 0.5 },
    envMap: { value: null }
  },
  
  vertexShader: `
    varying vec3 vNormal;
    varying vec3 vViewDir;
    varying vec3 vWorldPos;
    varying vec2 vUv;
    
    void main() {
      vUv = uv;
      vec4 worldPos = modelMatrix * vec4(position, 1.0);
      vWorldPos = worldPos.xyz;
      vNormal = normalize(normalMatrix * normal);
      vViewDir = normalize(cameraPosition - worldPos.xyz);
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  
  fragmentShader: `
    uniform vec3 baseColor;
    uniform vec3 flakeColor;
    uniform float flakeScale;
    uniform float flakeIntensity;
    uniform float clearcoatIntensity;
    
    varying vec3 vNormal;
    varying vec3 vViewDir;
    varying vec3 vWorldPos;
    varying vec2 vUv;
    
    // Simple hash function for procedural flakes
    float hash(vec2 p) {
      p = fract(p * vec2(234.34, 435.345));
      p += dot(p, p + 34.23);
      return fract(p.x * p.y);
    }
    
    void main() {
      vec3 N = normalize(vNormal);
      vec3 V = normalize(vViewDir);
      
      // Procedural metallic flakes
      vec2 flakeUV = vUv * flakeScale;
      float flakeNoise = hash(floor(flakeUV));
      
      // Random flake orientation
      vec3 flakeNormal = N;
      if (flakeNoise > 0.8) {
        float angle = flakeNoise * 6.28;
        flakeNormal = normalize(N + vec3(cos(angle), 0.0, sin(angle)) * 0.3);
      }
      
      // Flake reflection
      vec3 flakeReflect = reflect(-V, flakeNormal);
      float flakeSpecular = pow(max(dot(flakeReflect, V), 0.0), 64.0);
      
      // Fresnel for clearcoat
      float fresnel = pow(1.0 - max(dot(N, V), 0.0), 4.0);
      
      // Base color
      vec3 finalColor = baseColor;
      
      // Add flakes
      finalColor = mix(finalColor, flakeColor, flakeNoise > 0.8 ? flakeIntensity : 0.0);
      
      // Clearcoat reflection
      finalColor += vec3(0.9, 0.95, 1.0) * fresnel * clearcoatIntensity;
      finalColor += flakeSpecular * 0.3;
      
      gl_FragColor = vec4(finalColor, 1.0);
    }
  `
});
```

## 7. Demo: Realistic Glass Scene

```html
<!DOCTYPE html>
<html>
<head>
  <title>Glass & Special Materials</title>
  <style>
    body { margin: 0; background: #111; }
    canvas { display: block; }
    #controls {
      position: fixed;
      top: 10px;
      right: 10px;
      background: rgba(0,0,0,0.85);
      color: #ddd;
      padding: 15px;
      border-radius: 8px;
      font-family: Arial, sans-serif;
      font-size: 12px;
      width: 200px;
    }
    .row { margin: 6px 0; }
    label { display: block; margin-bottom: 2px; color: #aaf; }
    input[type="range"] { width: 170px; }
    select { width: 175px; background: #333; color: #fff; border: 1px solid #555; padding: 3px; }
    .val { color: #fa8; font-size: 11px; }
    h4 { color: #8af; margin: 0 0 8px; }
  </style>
</head>
<body>
<div id="controls">
  <h4>Material Properties</h4>
  <div class="row">
    <label>Transmission: <span class="val" id="transVal">1.00</span></label>
    <input type="range" id="trans" min="0" max="100" value="100">
  </div>
  <div class="row">
    <label>IOR: <span class="val" id="iorVal">1.50</span></label>
    <input type="range" id="ior" min="100" max="300" value="150">
  </div>
  <div class="row">
    <label>Roughness: <span class="val" id="roughVal">0.00</span></label>
    <input type="range" id="rough" min="0" max="100" value="0">
  </div>
  <div class="row">
    <label>Iridescence: <span class="val" id="iriVal">0.00</span></label>
    <input type="range" id="iri" min="0" max="100" value="0">
  </div>
  <div class="row">
    <label>Clearcoat: <span class="val" id="ccVal">1.00</span></label>
    <input type="range" id="cc" min="0" max="100" value="100">
  </div>
  <div class="row">
    <label>Anisotropy: <span class="val" id="aniVal">0.00</span></label>
    <input type="range" id="ani" min="0" max="100" value="0">
  </div>
  <div class="row">
    <label>Preset:</label>
    <select id="preset">
      <option value="glass">Clear Glass</option>
      <option value="frosted">Frosted Glass</option>
      <option value="soap">Soap Bubble</option>
      <option value="diamond">Diamond</option>
      <option value="car">Car Paint</option>
      <option value="brushed">Brushed Metal</option>
      <option value="pearl">Pearl/Iridescent</option>
    </select>
  </div>
</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';
import { RoomEnvironment } from 'https://unpkg.com/three@0.158.0/examples/jsm/environments/RoomEnvironment.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.01, 100);
camera.position.set(0, 1, 7);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(devicePixelRatio);
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.2;
renderer.outputColorSpace = THREE.SRGBColorSpace;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

// Environment
const pmrem = new THREE.PMREMGenerator(renderer);
const envMap = pmrem.fromScene(new RoomEnvironment(), 0.04).texture;
scene.environment = envMap;
scene.background = new THREE.Color(0x112233);

// Lights
scene.add(new THREE.AmbientLight(0xffffff, 300));
const dirLight = new THREE.DirectionalLight(0xffffff, 2000);
dirLight.position.set(5, 10, 5);
dirLight.castShadow = true;
scene.add(dirLight);

// Colorful background objects
const bgColors = [0xff3300, 0x3300ff, 0x00ff33, 0xff9900, 0xff0099, 0x00ffcc];
for (let i = 0; i < 20; i++) {
  const obj = new THREE.Mesh(
    new THREE.SphereGeometry(0.3 + Math.random() * 0.3, 16, 16),
    new THREE.MeshStandardMaterial({
      color: bgColors[i % bgColors.length],
      roughness: 0.5
    })
  );
  obj.position.set(
    (Math.random() - 0.5) * 14,
    (Math.random() - 0.5) * 8,
    -4 - Math.random() * 8
  );
  scene.add(obj);
}

// Main glass object
const glassMat = new THREE.MeshPhysicalMaterial({
  transmission: 1.0,
  ior: 1.5,
  roughness: 0.0,
  metalness: 0.0,
  thickness: 1.5,
  clearcoat: 1.0,
  clearcoatRoughness: 0.0,
  specularIntensity: 1.0,
  iridescence: 0.0,
  anisotropy: 0.0,
  color: 0xffffff,
  side: THREE.FrontSide
});

const mainGeo = new THREE.TorusKnotGeometry(1.0, 0.4, 128, 32);
const mainMesh = new THREE.Mesh(mainGeo, glassMat);
mainMesh.castShadow = true;
scene.add(mainMesh);

// Ground
const ground = new THREE.Mesh(
  new THREE.PlaneGeometry(20, 20),
  new THREE.MeshStandardMaterial({ color: 0x223344, roughness: 0.8 })
);
ground.rotation.x = -Math.PI / 2;
ground.position.y = -2;
ground.receiveShadow = true;
scene.add(ground);

// Presets
const presets = {
  glass: {
    transmission: 1.0, ior: 1.5, roughness: 0.0, metalness: 0.0,
    clearcoat: 1.0, clearcoatRoughness: 0.0, iridescence: 0.0,
    anisotropy: 0.0, color: 0xffffff
  },
  frosted: {
    transmission: 0.9, ior: 1.5, roughness: 0.4, metalness: 0.0,
    clearcoat: 0.5, clearcoatRoughness: 0.3, iridescence: 0.0,
    anisotropy: 0.0, color: 0xeeeeff
  },
  soap: {
    transmission: 0.9, ior: 1.35, roughness: 0.0, metalness: 0.0,
    clearcoat: 1.0, clearcoatRoughness: 0.0, iridescence: 1.0,
    anisotropy: 0.0, color: 0xffffff
  },
  diamond: {
    transmission: 0.95, ior: 2.42, roughness: 0.0, metalness: 0.0,
    clearcoat: 1.0, clearcoatRoughness: 0.0, iridescence: 0.0,
    anisotropy: 0.0, color: 0xffffff
  },
  car: {
    transmission: 0.0, ior: 1.5, roughness: 0.1, metalness: 0.0,
    clearcoat: 1.0, clearcoatRoughness: 0.05, iridescence: 0.0,
    anisotropy: 0.0, color: 0xcc2200
  },
  brushed: {
    transmission: 0.0, ior: 1.5, roughness: 0.3, metalness: 1.0,
    clearcoat: 0.0, clearcoatRoughness: 0.0, iridescence: 0.0,
    anisotropy: 0.9, color: 0xbbbbcc
  },
  pearl: {
    transmission: 0.0, ior: 1.5, roughness: 0.1, metalness: 0.0,
    clearcoat: 1.0, clearcoatRoughness: 0.05, iridescence: 1.0,
    anisotropy: 0.0, color: 0xf0e8e0
  }
};

function applyPreset(name) {
  const p = presets[name];
  if (!p) return;
  
  glassMat.transmission = p.transmission;
  glassMat.ior = p.ior;
  glassMat.roughness = p.roughness;
  glassMat.metalness = p.metalness;
  glassMat.clearcoat = p.clearcoat;
  glassMat.clearcoatRoughness = p.clearcoatRoughness;
  glassMat.iridescence = p.iridescence;
  glassMat.anisotropy = p.anisotropy;
  glassMat.color.setHex(p.color);
  
  // Update sliders
  document.getElementById('trans').value = p.transmission * 100;
  document.getElementById('ior').value = p.ior * 100;
  document.getElementById('rough').value = p.roughness * 100;
  document.getElementById('iri').value = p.iridescence * 100;
  document.getElementById('cc').value = p.clearcoat * 100;
  document.getElementById('ani').value = p.anisotropy * 100;
  
  document.getElementById('transVal').textContent = p.transmission.toFixed(2);
  document.getElementById('iorVal').textContent = p.ior.toFixed(2);
  document.getElementById('roughVal').textContent = p.roughness.toFixed(2);
  document.getElementById('iriVal').textContent = p.iridescence.toFixed(2);
  document.getElementById('ccVal').textContent = p.clearcoat.toFixed(2);
  document.getElementById('aniVal').textContent = p.anisotropy.toFixed(2);
}

// Slider events
const sliderMap = {
  trans: (v) => { glassMat.transmission = v/100; document.getElementById('transVal').textContent = (v/100).toFixed(2); },
  ior: (v) => { glassMat.ior = v/100; document.getElementById('iorVal').textContent = (v/100).toFixed(2); },
  rough: (v) => { glassMat.roughness = v/100; document.getElementById('roughVal').textContent = (v/100).toFixed(2); },
  iri: (v) => { glassMat.iridescence = v/100; document.getElementById('iriVal').textContent = (v/100).toFixed(2); },
  cc: (v) => { glassMat.clearcoat = v/100; document.getElementById('ccVal').textContent = (v/100).toFixed(2); },
  ani: (v) => { glassMat.anisotropy = v/100; document.getElementById('aniVal').textContent = (v/100).toFixed(2); },
};

Object.entries(sliderMap).forEach(([id, fn]) => {
  document.getElementById(id).addEventListener('input', (e) => fn(parseInt(e.target.value)));
});

document.getElementById('preset').addEventListener('change', (e) => applyPreset(e.target.value));

const clock = new THREE.Clock();
function animate() {
  requestAnimationFrame(animate);
  mainMesh.rotation.y = clock.elapsedTime * 0.3;
  mainMesh.rotation.x = Math.sin(clock.elapsedTime * 0.2) * 0.2;
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

## 8. Demo: Animated Water Surface

```html
<!DOCTYPE html>
<html>
<head>
  <title>Animated Water</title>
  <style>
    body { margin: 0; }
    canvas { display: block; }
  </style>
</head>
<body>
<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';
import { RoomEnvironment } from 'https://unpkg.com/three@0.158.0/examples/jsm/environments/RoomEnvironment.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(60, innerWidth/innerHeight, 0.1, 1000);
camera.position.set(0, 10, 20);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(devicePixelRatio);
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.shadowMap.enabled = true;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.target.set(0, 0, 0);
controls.enableDamping = true;

// Environment
const pmrem = new THREE.PMREMGenerator(renderer);
const envMap = pmrem.fromScene(new RoomEnvironment()).texture;
scene.environment = envMap;
scene.background = new THREE.Color(0x557799);

// Sun light
const sun = new THREE.DirectionalLight(0xfff8e0, 3000);
sun.position.set(10, 20, 5);
sun.castShadow = true;
scene.add(sun);
scene.add(new THREE.AmbientLight(0x4466aa, 200));

// ===== WATER SHADER =====
const waterMat = new THREE.ShaderMaterial({
  uniforms: {
    time: { value: 0 },
    sunDir: { value: new THREE.Vector3(0.5, 1, 0.3).normalize() },
    sunColor: { value: new THREE.Color(0xfff8e0) },
    waterColor: { value: new THREE.Color(0x0a4a6a) },
    deepColor: { value: new THREE.Color(0x020a1a) },
    shallowColor: { value: new THREE.Color(0x1a9a8a) },
    fogColor: { value: new THREE.Color(0x557799) },
    resolution: { value: new THREE.Vector2(innerWidth, innerHeight) }
  },
  
  vertexShader: `
    uniform float time;
    
    varying vec3 vNormal;
    varying vec3 vViewDir;
    varying vec3 vWorldPos;
    varying vec2 vUv;
    
    // Gerstner Wave
    vec3 gerstnerWave(vec3 pos, vec2 dir, float amplitude, float wavelength, float speed, float steepness) {
      float k = 2.0 * 3.14159 / wavelength;
      float c = sqrt(9.8 / k);
      vec2 d = normalize(dir);
      float f = k * (dot(d, pos.xz) - c * time * speed);
      
      vec3 displacement;
      displacement.x = steepness * amplitude * d.x * cos(f);
      displacement.y = amplitude * sin(f);
      displacement.z = steepness * amplitude * d.y * cos(f);
      return displacement;
    }
    
    void main() {
      vUv = uv;
      vec3 pos = position;
      
      // Multiple Gerstner waves
      pos += gerstnerWave(pos, vec2(1.0, 0.7), 0.3, 8.0, 1.0, 0.5);
      pos += gerstnerWave(pos, vec2(-0.5, 1.0), 0.15, 5.0, 1.2, 0.5);
      pos += gerstnerWave(pos, vec2(0.3, -0.8), 0.1, 3.0, 1.5, 0.5);
      pos += gerstnerWave(pos, vec2(0.8, 0.2), 0.05, 2.0, 2.0, 0.3);
      
      vec4 worldPos = modelMatrix * vec4(pos, 1.0);
      vWorldPos = worldPos.xyz;
      
      // Approximate normal from waves
      vec3 eps = vec3(0.01, 0, 0);
      vec3 posX = position + eps.xyz;
      vec3 posZ = position + eps.zyx;
      
      posX += gerstnerWave(posX, vec2(1.0, 0.7), 0.3, 8.0, 1.0, 0.5);
      posZ += gerstnerWave(posZ, vec2(1.0, 0.7), 0.3, 8.0, 1.0, 0.5);
      
      vec3 tangentX = normalize(posX - pos);
      vec3 tangentZ = normalize(posZ - pos);
      vNormal = normalize(cross(tangentZ, tangentX));
      
      vViewDir = normalize(cameraPosition - worldPos.xyz);
      
      gl_Position = projectionMatrix * viewMatrix * worldPos;
    }
  `,
  
  fragmentShader: `
    uniform float time;
    uniform vec3 sunDir;
    uniform vec3 sunColor;
    uniform vec3 waterColor;
    uniform vec3 deepColor;
    uniform vec3 shallowColor;
    uniform vec3 fogColor;
    
    varying vec3 vNormal;
    varying vec3 vViewDir;
    varying vec3 vWorldPos;
    varying vec2 vUv;
    
    void main() {
      vec3 N = normalize(vNormal);
      vec3 V = normalize(vViewDir);
      vec3 L = normalize(sunDir);
      
      // Fresnel
      float cosTheta = max(dot(N, V), 0.0);
      float fresnel = pow(1.0 - cosTheta, 3.0) * 0.7 + 0.1;
      
      // Specular highlight (sun reflection)
      vec3 H = normalize(L + V);
      float specular = pow(max(dot(N, H), 0.0), 256.0) * 3.0;
      
      // Water color blending (depth simulation)
      float depth = smoothstep(0.0, 1.0, abs(vWorldPos.y) * 0.5);
      vec3 wColor = mix(shallowColor, deepColor, depth);
      
      // Small ripple details
      float ripple = sin(vUv.x * 40.0 + time * 3.0) * sin(vUv.y * 40.0 + time * 2.5) * 0.02;
      
      // Combine
      vec3 finalColor = wColor * (0.3 + 0.2 * max(dot(N, L), 0.0));
      finalColor += sunColor * specular;
      finalColor = mix(finalColor, vec3(0.8, 0.9, 1.0), fresnel);
      
      // Distance fog
      float dist = length(cameraPosition - vWorldPos);
      float fog = clamp(dist / 60.0, 0.0, 1.0);
      finalColor = mix(finalColor, fogColor, fog * fog);
      
      gl_FragColor = vec4(finalColor, 0.9 + fresnel * 0.1);
    }
  `,
  
  transparent: true,
  side: THREE.FrontSide
});

// Water mesh
const water = new THREE.Mesh(
  new THREE.PlaneGeometry(80, 80, 128, 128),
  waterMat
);
water.rotation.x = -Math.PI / 2;
scene.add(water);

// Island/objects above water
const island = new THREE.Mesh(
  new THREE.ConeGeometry(3, 2, 8),
  new THREE.MeshStandardMaterial({ color: 0x886633, roughness: 0.9 })
);
island.position.set(-5, -0.5, -5);
island.castShadow = true;
scene.add(island);

const palm = new THREE.Mesh(
  new THREE.CylinderGeometry(0.1, 0.2, 4),
  new THREE.MeshStandardMaterial({ color: 0x664422, roughness: 0.9 })
);
palm.position.set(-5, 2, -5);
scene.add(palm);

// Boat
const boat = new THREE.Group();
const hull = new THREE.Mesh(
  new THREE.BoxGeometry(2, 0.5, 1),
  new THREE.MeshStandardMaterial({ color: 0x886644, roughness: 0.6 })
);
boat.add(hull);
boat.position.set(4, 0.2, 3);
scene.add(boat);

const clock = new THREE.Clock();
function animate() {
  requestAnimationFrame(animate);
  
  const elapsed = clock.elapsedTime;
  waterMat.uniforms.time.value = elapsed;
  
  // Boat bob on waves
  boat.position.y = 0.2 + Math.sin(elapsed * 1.5) * 0.15;
  boat.rotation.z = Math.sin(elapsed * 1.2) * 0.05;
  
  controls.update();
  renderer.render(scene, camera);
}

animate();

window.addEventListener('resize', () => {
  camera.aspect = innerWidth / innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(innerWidth, innerHeight);
  waterMat.uniforms.resolution.value.set(innerWidth, innerHeight);
});
</script>
</body>
</html>
```

## สรุป

Special Materials ใน Three.js:

1. **MeshPhysicalMaterial** - glass, clearcoat, anisotropy, iridescence
2. **Custom Shader** - water, SSS, hair
3. **Transmission** - แสงผ่านวัตถุโปร่ง
4. **Iridescence** - thin film effects (soap, pearl)
5. **Anisotropy** - directional highlights (brushed metal)

---

## แบบฝึกหัด

1. **Easy**: สร้าง glass ornament collection ที่มี glass shapes หลายรูปทรง
2. **Medium**: สร้าง water scene ที่มี Gerstner waves และ Fresnel reflection
3. **Hard**: สร้าง SSS skin material สำหรับ character lighting
4. **Challenge**: สร้าง car paint system ที่มี metallic flakes, clearcoat, และ anisotropic specular
