# Part 41: Reflections & Refractions ใน Three.js

## บทนำ

Reflections และ Refractions เป็น optical phenomena ที่สำคัญมากสำหรับ realistic rendering:
- **Reflection** - แสงที่สะท้อนกลับจากผิว (กระจก, โลหะ, น้ำ)
- **Refraction** - แสงที่งอผ่านวัตถุโปร่งใส (แก้ว, น้ำ, เพชร)
- **Fresnel Effect** - reflection เพิ่มขึ้นเมื่อมองจากมุมตื้น

## 1. CubeCamera สำหรับ Real-Time Reflections

```javascript
import * as THREE from 'three';

// ===== WebGLCubeRenderTarget =====
const cubeRenderTarget = new THREE.WebGLCubeRenderTarget(256, {
  format: THREE.RGBFormat,
  generateMipmaps: true,
  minFilter: THREE.LinearMipmapLinearFilter
});

// ===== CubeCamera =====
const cubeCamera = new THREE.CubeCamera(
  0.1,   // near
  1000,  // far
  cubeRenderTarget
);
scene.add(cubeCamera);

// Material ที่ใช้ dynamic reflection
const reflectiveMaterial = new THREE.MeshStandardMaterial({
  envMap: cubeRenderTarget.texture,
  envMapIntensity: 1.0,
  roughness: 0.0,
  metalness: 1.0
});

const sphere = new THREE.Mesh(
  new THREE.SphereGeometry(1, 64, 64),
  reflectiveMaterial
);
scene.add(sphere);

// Update ใน render loop
function render() {
  sphere.visible = false; // ป้องกัน self-reflection
  cubeCamera.position.copy(sphere.position);
  cubeCamera.update(renderer, scene);
  sphere.visible = true;
  
  renderer.render(scene, camera);
}
```

## 2. Mirror Formula ใน Shader

```glsl
// Vertex Shader
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

// Fragment Shader
uniform samplerCube envMap;
uniform float roughness;
uniform float metalness;

varying vec3 vNormal;
varying vec3 vViewDir;
varying vec3 vWorldPos;

// Fresnel Schlick approximation
float fresnelSchlick(float cosTheta, float F0) {
  return F0 + (1.0 - F0) * pow(1.0 - cosTheta, 5.0);
}

void main() {
  vec3 N = normalize(vNormal);
  vec3 V = normalize(vViewDir);
  
  // Reflection vector
  vec3 R = reflect(-V, N);
  
  // Fresnel factor
  float F0 = mix(0.04, 1.0, metalness);
  float cosTheta = max(dot(N, V), 0.0);
  float fresnel = fresnelSchlick(cosTheta, F0);
  
  // Sample environment map
  vec3 reflectionColor = textureCube(envMap, R).rgb;
  
  // Diffuse color (attenuated by metalness)
  vec3 diffuse = vec3(1.0) * (1.0 - metalness) * 0.5;
  
  // Final color blend
  vec3 finalColor = mix(diffuse, reflectionColor, fresnel * (1.0 - roughness));
  
  gl_FragColor = vec4(finalColor, 1.0);
}
```

## 3. Refraction with IOR

IOR = Index of Refraction (ดัชนีหักเห)

| Material | IOR |
|----------|-----|
| Air | 1.0 |
| Water | 1.333 |
| Glass | 1.5 |
| Diamond | 2.417 |

```javascript
// ===== MeshPhysicalMaterial สำหรับ Refraction =====
const glassMaterial = new THREE.MeshPhysicalMaterial({
  // Transmission (0 = opaque, 1 = fully transparent)
  transmission: 1.0,
  
  // Index of Refraction
  ior: 1.5, // Glass
  
  // Thickness ของวัตถุ (สำหรับ chromatic aberration)
  thickness: 2.0,
  
  // roughness เปลี่ยน refraction เป็น frosted glass
  roughness: 0.0,
  metalness: 0.0,
  
  // Tint color (การดูดซับสี)
  attenuationColor: new THREE.Color(0x88ddff),
  attenuationDistance: 0.5,
  
  // Clearcoat สำหรับ surface gloss
  clearcoat: 1.0,
  clearcoatRoughness: 0.05,
  
  // Specular highlight
  specularIntensity: 1.0,
  specularColor: new THREE.Color(0xffffff),
  
  // Side ต้องเป็น FrontSide หรือ DoubleSide
  side: THREE.FrontSide,
  transparent: false // ไม่ใช้ transparent เมื่อใช้ transmission
});
```

## 4. Glass Material Recipe

```javascript
// ===== Perfect Glass =====
function createGlassMaterial(options = {}) {
  return new THREE.MeshPhysicalMaterial({
    // Core glass properties
    transmission: options.transmission ?? 1.0,
    ior: options.ior ?? 1.5,
    thickness: options.thickness ?? 1.0,
    
    // Surface quality
    roughness: options.roughness ?? 0.0,
    metalness: 0.0,
    
    // Color tint
    color: options.color ?? 0xffffff,
    
    // Refraction background blur
    // (requires renderer.transmissionSampler = true)
    
    // Clearcoat for realistic glass surface
    clearcoat: 1.0,
    clearcoatRoughness: 0.0,
    
    // Specular
    specularIntensity: 1.0,
    specularColor: new THREE.Color(0xffffff),
    
    // Attenuation (color shift with depth)
    attenuationColor: options.attenuationColor ?? new THREE.Color(0xffffff),
    attenuationDistance: options.attenuationDistance ?? Infinity,
    
    envMapIntensity: 1.5,
    side: THREE.FrontSide
  });
}

// ===== Glass Presets =====
const glassTypes = {
  clear: createGlassMaterial({ ior: 1.5, roughness: 0.0 }),
  frosted: createGlassMaterial({ ior: 1.5, roughness: 0.4 }),
  green: createGlassMaterial({ 
    ior: 1.52, 
    attenuationColor: new THREE.Color(0x44aa44),
    attenuationDistance: 1.5
  }),
  blue: createGlassMaterial({
    ior: 1.48,
    color: new THREE.Color(0xaaccff),
    attenuationColor: new THREE.Color(0x4488ff),
    attenuationDistance: 2.0
  }),
  diamond: createGlassMaterial({ ior: 2.417, roughness: 0.0 }),
  water: createGlassMaterial({ 
    ior: 1.333, 
    roughness: 0.0,
    attenuationColor: new THREE.Color(0x88ddff),
    attenuationDistance: 5.0
  })
};
```

## 5. Fresnel Effect Math

```javascript
// ===== Fresnel Effect Implementation =====
// Fresnel Effect: reflection เพิ่มขึ้นเมื่อมองจากมุมตื้น

// Schlick Approximation (fast, used in PBR)
function fresnelSchlick(cosTheta, F0) {
  return F0 + (1 - F0) * Math.pow(1 - cosTheta, 5);
}

// Full Fresnel Equations (more accurate)
function fresnelFull(cosTheta, n1, n2) {
  const sinTheta1 = Math.sqrt(1 - cosTheta * cosTheta);
  const sinTheta2 = (n1 / n2) * sinTheta1;
  
  if (sinTheta2 >= 1) return 1; // Total Internal Reflection
  
  const cosTheta2 = Math.sqrt(1 - sinTheta2 * sinTheta2);
  
  // s-polarization
  const rs = (n1 * cosTheta - n2 * cosTheta2) / (n1 * cosTheta + n2 * cosTheta2);
  // p-polarization
  const rp = (n2 * cosTheta - n1 * cosTheta2) / (n2 * cosTheta + n1 * cosTheta2);
  
  return (rs * rs + rp * rp) / 2;
}

// ===== GLSL Fresnel Shader =====
const fresnelShaderMaterial = new THREE.ShaderMaterial({
  uniforms: {
    fresnelBias: { value: 0.1 },
    fresnelScale: { value: 0.5 },
    fresnelPower: { value: 2.0 },
    color1: { value: new THREE.Color(0xffffff) },
    color2: { value: new THREE.Color(0x4488ff) },
    envMap: { value: null }
  },
  
  vertexShader: `
    varying vec3 vNormal;
    varying vec3 vViewDir;
    
    void main() {
      vNormal = normalize(normalMatrix * normal);
      vec4 worldPos = modelMatrix * vec4(position, 1.0);
      vViewDir = normalize(cameraPosition - worldPos.xyz);
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  
  fragmentShader: `
    uniform float fresnelBias;
    uniform float fresnelScale;
    uniform float fresnelPower;
    uniform vec3 color1;
    uniform vec3 color2;
    uniform samplerCube envMap;
    
    varying vec3 vNormal;
    varying vec3 vViewDir;
    
    void main() {
      vec3 N = normalize(vNormal);
      vec3 V = normalize(vViewDir);
      
      // Fresnel factor
      float fresnel = fresnelBias + fresnelScale * pow(1.0 - dot(N, V), fresnelPower);
      fresnel = clamp(fresnel, 0.0, 1.0);
      
      // Reflection
      vec3 R = reflect(-V, N);
      vec3 envColor = textureCube(envMap, R).rgb;
      
      // Blend color based on fresnel
      vec3 finalColor = mix(color1, color2, fresnel);
      finalColor = mix(finalColor, envColor, fresnel);
      
      gl_FragColor = vec4(finalColor, 1.0);
    }
  `
});
```

## 6. Demo 1: Real-Time Reflection Car Scene

```html
<!DOCTYPE html>
<html>
<head>
  <title>Car Reflection Demo</title>
  <style>
    body { margin: 0; background: #111; }
    canvas { display: block; }
    #info {
      position: fixed;
      top: 20px;
      left: 20px;
      color: #aaf;
      font-family: Arial, sans-serif;
      font-size: 14px;
      pointer-events: none;
    }
  </style>
</head>
<body>
<div id="info">Car Paint with Real-Time Reflections<br>
Drag to rotate • Scroll to zoom</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';
import { RoomEnvironment } from 'https://unpkg.com/three@0.158.0/examples/jsm/environments/RoomEnvironment.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(50, innerWidth / innerHeight, 0.01, 100);
camera.position.set(0, 2, 8);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(devicePixelRatio);
renderer.shadowMap.enabled = true;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.2;
renderer.outputColorSpace = THREE.SRGBColorSpace;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.minDistance = 3;
controls.maxDistance = 20;

// Environment
const pmrem = new THREE.PMREMGenerator(renderer);
const envMap = pmrem.fromScene(new RoomEnvironment(), 0.04).texture;
scene.environment = envMap;
scene.background = new THREE.Color(0x111122);

// Lights
scene.add(new THREE.AmbientLight(0x223344, 500));
const keyLight = new THREE.DirectionalLight(0xffffff, 5000);
keyLight.position.set(5, 10, 5);
keyLight.castShadow = true;
scene.add(keyLight);

// Ground
const ground = new THREE.Mesh(
  new THREE.PlaneGeometry(30, 30),
  new THREE.MeshStandardMaterial({
    color: 0x222222,
    roughness: 0.3,
    metalness: 0.5
  })
);
ground.rotation.x = -Math.PI / 2;
ground.receiveShadow = true;
scene.add(ground);

// ===== CAR BODY (simplified) =====
// CubeCamera สำหรับ car body reflections
const cubeRenderTarget = new THREE.WebGLCubeRenderTarget(256, {
  generateMipmaps: true,
  minFilter: THREE.LinearMipmapLinearFilter
});
const cubeCamera = new THREE.CubeCamera(0.01, 100, cubeRenderTarget);
scene.add(cubeCamera);

// Car paint material
const carPaintMaterial = new THREE.MeshPhysicalMaterial({
  color: 0xcc2200,  // Red car
  roughness: 0.1,
  metalness: 0.0,
  clearcoat: 1.0,
  clearcoatRoughness: 0.05,
  envMap: cubeRenderTarget.texture,
  envMapIntensity: 1.5
});

// Car body geometry (simplified car shape)
const carGroup = new THREE.Group();

// Main body
const bodyGeo = new THREE.BoxGeometry(4, 1, 2);
const body = new THREE.Mesh(bodyGeo, carPaintMaterial);
body.position.y = 0.7;
body.castShadow = true;
carGroup.add(body);

// Top (cabin)
const cabinGeo = new THREE.BoxGeometry(2, 0.8, 1.8);
const cabin = new THREE.Mesh(cabinGeo, carPaintMaterial);
cabin.position.set(-0.2, 1.6, 0);
cabin.castShadow = true;
carGroup.add(cabin);

// Windows
const windowMat = new THREE.MeshPhysicalMaterial({
  transmission: 0.95,
  ior: 1.5,
  roughness: 0.0,
  color: 0x88aacc,
  opacity: 0.5,
  transparent: true,
  side: THREE.DoubleSide
});

const frontWindow = new THREE.Mesh(
  new THREE.PlaneGeometry(1.8, 0.7),
  windowMat
);
frontWindow.position.set(0.8, 1.6, 0.91);
carGroup.add(frontWindow);

const rearWindow = new THREE.Mesh(
  new THREE.PlaneGeometry(1.8, 0.7),
  windowMat
);
rearWindow.position.set(-1.0, 1.6, 0.91);
carGroup.add(rearWindow);

// Wheels
const wheelMat = new THREE.MeshStandardMaterial({ color: 0x111111, roughness: 0.8 });
const rimMat = new THREE.MeshStandardMaterial({ color: 0xcccccc, roughness: 0.2, metalness: 1.0 });

function createWheel(x, z) {
  const group = new THREE.Group();
  
  const tire = new THREE.Mesh(
    new THREE.TorusGeometry(0.4, 0.15, 16, 32),
    wheelMat
  );
  tire.rotation.y = Math.PI / 2;
  group.add(tire);
  
  const rim = new THREE.Mesh(
    new THREE.CylinderGeometry(0.3, 0.3, 0.05, 16),
    rimMat
  );
  rim.rotation.z = Math.PI / 2;
  group.add(rim);
  
  group.position.set(x, 0.4, z);
  return group;
}

carGroup.add(createWheel(1.5, 1.1));
carGroup.add(createWheel(1.5, -1.1));
carGroup.add(createWheel(-1.5, 1.1));
carGroup.add(createWheel(-1.5, -1.1));

scene.add(carGroup);

// ===== ENVIRONMENT OBJECTS (for reflections to show) =====
const envObjects = [];
const colors = [0xff4400, 0x4400ff, 0x00ff44, 0xffaa00, 0xff0088];

for (let i = 0; i < 10; i++) {
  const mat = new THREE.MeshStandardMaterial({
    color: colors[i % colors.length],
    roughness: 0.5,
    emissive: new THREE.Color(colors[i % colors.length]).multiplyScalar(0.2)
  });
  
  const obj = new THREE.Mesh(
    i % 2 === 0 ? new THREE.BoxGeometry(1, 2, 1) : new THREE.SphereGeometry(0.8),
    mat
  );
  
  const angle = (i / 10) * Math.PI * 2;
  const radius = 8;
  obj.position.set(
    Math.cos(angle) * radius,
    1,
    Math.sin(angle) * radius
  );
  
  obj.castShadow = true;
  scene.add(obj);
  envObjects.push(obj);
}

// ===== ANIMATION =====
const clock = new THREE.Clock();
let frameCount = 0;

function animate() {
  requestAnimationFrame(animate);
  
  const delta = clock.getDelta();
  const elapsed = clock.elapsedTime;
  
  // หมุน environment objects
  envObjects.forEach((obj, i) => {
    obj.rotation.y = elapsed * 0.5 + i * 0.3;
  });
  
  // หมุนล้อรถ
  carGroup.children.forEach(child => {
    if (child.children.length > 0) {
      child.rotation.x = elapsed * 2;
    }
  });
  
  // Update cube camera (ทุก 2 frames เพื่อ performance)
  if (frameCount % 2 === 0) {
    carGroup.visible = false;
    cubeCamera.position.copy(carGroup.position);
    cubeCamera.update(renderer, scene);
    carGroup.visible = true;
  }
  
  frameCount++;
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

## 7. Demo 2: Glass Material Demo

```html
<!DOCTYPE html>
<html>
<head>
  <title>Glass Material Demo</title>
  <style>
    body { margin: 0; }
    canvas { display: block; }
    #controls {
      position: fixed;
      top: 20px;
      right: 20px;
      background: rgba(0,0,0,0.8);
      color: white;
      padding: 15px;
      border-radius: 8px;
      font-family: Arial, sans-serif;
      font-size: 13px;
      width: 220px;
    }
    label { display: block; margin: 6px 0 2px; }
    input[type="range"] { width: 180px; }
    select { width: 185px; background: #333; color: white; padding: 4px; border: 1px solid #555; }
  </style>
</head>
<body>
<div id="controls">
  <b style="font-size:15px">Glass Properties</b>
  
  <label>Transmission: <span id="transVal">1.0</span></label>
  <input type="range" id="transmission" min="0" max="100" value="100">
  
  <label>IOR: <span id="iorVal">1.50</span></label>
  <input type="range" id="ior" min="100" max="300" value="150">
  
  <label>Thickness: <span id="thickVal">1.0</span></label>
  <input type="range" id="thickness" min="0" max="500" value="100">
  
  <label>Roughness: <span id="roughVal">0.0</span></label>
  <input type="range" id="roughness" min="0" max="100" value="0">
  
  <label>Clearcoat: <span id="ccVal">1.0</span></label>
  <input type="range" id="clearcoat" min="0" max="100" value="100">
  
  <label>Shape:</label>
  <select id="shape">
    <option value="sphere">Sphere</option>
    <option value="box">Box</option>
    <option value="knot">Torus Knot</option>
    <option value="diamond">Diamond</option>
  </select>
  
  <label>Tint:</label>
  <input type="color" id="tint" value="#ffffff">
</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';
import { RoomEnvironment } from 'https://unpkg.com/three@0.158.0/examples/jsm/environments/RoomEnvironment.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.01, 100);
camera.position.set(0, 0, 5);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(devicePixelRatio);
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.0;
renderer.outputColorSpace = THREE.SRGBColorSpace;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

// Environment
const pmrem = new THREE.PMREMGenerator(renderer);
const envMap = pmrem.fromScene(new RoomEnvironment(), 0.04).texture;
scene.environment = envMap;
scene.background = new THREE.Color(0x224455);

// Lighting
scene.add(new THREE.AmbientLight(0xffffff, 500));
const dirLight = new THREE.DirectionalLight(0xffffff, 3000);
dirLight.position.set(5, 10, 5);
scene.add(dirLight);

// Background objects (visible through glass)
const bgColors = [0xff4433, 0x4433ff, 0x33ff44, 0xff33aa, 0xffaa33];
for (let i = 0; i < 15; i++) {
  const mesh = new THREE.Mesh(
    new THREE.SphereGeometry(0.3 + Math.random() * 0.4, 16, 16),
    new THREE.MeshStandardMaterial({ 
      color: bgColors[i % bgColors.length],
      roughness: 0.5
    })
  );
  mesh.position.set(
    (Math.random() - 0.5) * 12,
    (Math.random() - 0.5) * 8,
    -3 - Math.random() * 5
  );
  scene.add(mesh);
}

// Glass material
const glassMat = new THREE.MeshPhysicalMaterial({
  transmission: 1.0,
  ior: 1.5,
  thickness: 1.0,
  roughness: 0.0,
  metalness: 0.0,
  clearcoat: 1.0,
  clearcoatRoughness: 0.0,
  specularIntensity: 1.0,
  specularColor: new THREE.Color(0xffffff),
  envMapIntensity: 1.5
});

let currentMesh = null;

function createShape(type) {
  if (currentMesh) scene.remove(currentMesh);
  
  let geo;
  switch (type) {
    case 'sphere':
      geo = new THREE.SphereGeometry(1.5, 64, 64);
      break;
    case 'box':
      geo = new THREE.BoxGeometry(2, 2, 2, 4, 4, 4);
      break;
    case 'knot':
      geo = new THREE.TorusKnotGeometry(1.0, 0.35, 128, 32);
      break;
    case 'diamond':
      // สร้าง diamond shape
      geo = new THREE.OctahedronGeometry(1.5, 2);
      const pos = geo.attributes.position;
      for (let i = 0; i < pos.count; i++) {
        const y = pos.getY(i);
        const scale = 1 - Math.abs(y) * 0.3;
        pos.setX(i, pos.getX(i) * scale);
        pos.setZ(i, pos.getZ(i) * scale);
        if (y < 0) pos.setY(i, y * 1.5);
      }
      pos.needsUpdate = true;
      geo.computeVertexNormals();
      break;
    default:
      geo = new THREE.SphereGeometry(1.5, 64, 64);
  }
  
  currentMesh = new THREE.Mesh(geo, glassMat);
  scene.add(currentMesh);
}

createShape('sphere');

// Controls
const controls_map = {
  transmission: (v) => { glassMat.transmission = v/100; document.getElementById('transVal').textContent = (v/100).toFixed(2); },
  ior: (v) => { glassMat.ior = v/100; document.getElementById('iorVal').textContent = (v/100).toFixed(2); },
  thickness: (v) => { glassMat.thickness = v/100; document.getElementById('thickVal').textContent = (v/100).toFixed(2); },
  roughness: (v) => { glassMat.roughness = v/100; document.getElementById('roughVal').textContent = (v/100).toFixed(2); },
  clearcoat: (v) => { glassMat.clearcoat = v/100; document.getElementById('ccVal').textContent = (v/100).toFixed(2); },
};

Object.entries(controls_map).forEach(([id, fn]) => {
  document.getElementById(id).addEventListener('input', (e) => fn(parseInt(e.target.value)));
});

document.getElementById('shape').addEventListener('change', (e) => createShape(e.target.value));
document.getElementById('tint').addEventListener('input', (e) => {
  glassMat.color.set(e.target.value);
  glassMat.attenuationColor.set(e.target.value);
});

const clock = new THREE.Clock();
function animate() {
  requestAnimationFrame(animate);
  
  if (currentMesh) {
    currentMesh.rotation.y = clock.elapsedTime * 0.3;
    currentMesh.rotation.x = Math.sin(clock.elapsedTime * 0.2) * 0.1;
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

## 8. Demo 3: Mirror Room

```html
<!DOCTYPE html>
<html>
<head>
  <title>Mirror Room</title>
  <style>
    body { margin: 0; }
    canvas { display: block; }
    #info {
      position: fixed;
      bottom: 20px;
      left: 50%;
      transform: translateX(-50%);
      color: white;
      font-family: Arial, sans-serif;
      text-align: center;
      pointer-events: none;
      text-shadow: 0 2px 4px rgba(0,0,0,0.8);
    }
  </style>
</head>
<body>
<div id="info">Mirror Room - Real-Time Reflections<br><small>เดินผ่านห้องกระจก</small></div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';
import { RoomEnvironment } from 'https://unpkg.com/three@0.158.0/examples/jsm/environments/RoomEnvironment.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, innerWidth / innerHeight, 0.01, 100);
camera.position.set(0, 2, 8);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(devicePixelRatio);
renderer.shadowMap.enabled = true;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.outputColorSpace = THREE.SRGBColorSpace;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.target.set(0, 2, 0);

// Environment
const pmrem = new THREE.PMREMGenerator(renderer);
const envMap = pmrem.fromScene(new RoomEnvironment()).texture;
scene.environment = envMap;

// Lights
scene.add(new THREE.AmbientLight(0xffffff, 200));
const pointLight = new THREE.PointLight(0xffffff, 2000, 30, 2);
pointLight.position.set(0, 4, 0);
pointLight.castShadow = true;
scene.add(pointLight);

// ===== ROOM =====
const roomSize = 10;
const roomMat = new THREE.MeshStandardMaterial({ color: 0x334455, roughness: 0.9, side: THREE.BackSide });

const room = new THREE.Mesh(
  new THREE.BoxGeometry(roomSize, 6, roomSize),
  roomMat
);
room.position.y = 3;
scene.add(room);

// Floor
const floor = new THREE.Mesh(
  new THREE.PlaneGeometry(roomSize, roomSize),
  new THREE.MeshStandardMaterial({ color: 0x223344, roughness: 0.3, metalness: 0.5 })
);
floor.rotation.x = -Math.PI / 2;
floor.receiveShadow = true;
scene.add(floor);

// ===== MIRRORS =====
const mirrors = [];

function createMirror(position, rotation, size) {
  // สร้าง CubeCamera สำหรับแต่ละกระจก
  const mirrorTarget = new THREE.WebGLCubeRenderTarget(512, {
    generateMipmaps: true,
    minFilter: THREE.LinearMipmapLinearFilter
  });
  const mirrorCamera = new THREE.CubeCamera(0.01, 100, mirrorTarget);
  scene.add(mirrorCamera);
  
  const mirrorMat = new THREE.MeshStandardMaterial({
    envMap: mirrorTarget.texture,
    roughness: 0.0,
    metalness: 1.0,
    color: 0xffffff,
    envMapIntensity: 1.0
  });
  
  const [w, h] = size;
  const mirror = new THREE.Mesh(
    new THREE.PlaneGeometry(w, h),
    mirrorMat
  );
  
  mirror.position.copy(position);
  if (rotation) mirror.rotation.copy(rotation);
  
  scene.add(mirror);
  
  mirrors.push({ mirror, camera: mirrorCamera });
  return mirror;
}

// 4 mirrors on walls
createMirror(
  new THREE.Vector3(0, 3, -4.9),
  new THREE.Euler(0, 0, 0),
  [8, 5]
);

createMirror(
  new THREE.Vector3(0, 3, 4.9),
  new THREE.Euler(0, Math.PI, 0),
  [8, 5]
);

createMirror(
  new THREE.Vector3(-4.9, 3, 0),
  new THREE.Euler(0, Math.PI/2, 0),
  [8, 5]
);

createMirror(
  new THREE.Vector3(4.9, 3, 0),
  new THREE.Euler(0, -Math.PI/2, 0),
  [8, 5]
);

// Center object
const centerObj = new THREE.Mesh(
  new THREE.TorusKnotGeometry(0.7, 0.25, 64, 16),
  new THREE.MeshPhysicalMaterial({
    color: 0xffd700,
    roughness: 0.1,
    metalness: 1.0,
    clearcoat: 1.0,
    envMapIntensity: 2.0
  })
);
centerObj.position.y = 2;
centerObj.castShadow = true;
scene.add(centerObj);

// Column lights
for (let i = 0; i < 4; i++) {
  const angle = (i / 4) * Math.PI * 2 + Math.PI/4;
  const light = new THREE.PointLight(
    new THREE.Color().setHSL(i/4, 1, 0.5),
    500, 8, 2
  );
  light.position.set(
    Math.cos(angle) * 3,
    1,
    Math.sin(angle) * 3
  );
  scene.add(light);
  
  const bulb = new THREE.Mesh(
    new THREE.SphereGeometry(0.1, 8, 8),
    new THREE.MeshStandardMaterial({
      emissive: new THREE.Color().setHSL(i/4, 1, 0.5),
      emissiveIntensity: 5,
      color: 0x000000
    })
  );
  bulb.position.copy(light.position);
  scene.add(bulb);
}

const clock = new THREE.Clock();
let frame = 0;

function animate() {
  requestAnimationFrame(animate);
  
  const elapsed = clock.elapsedTime;
  
  // หมุน center object
  centerObj.rotation.y = elapsed * 0.5;
  centerObj.rotation.x = elapsed * 0.3;
  
  // Update mirrors (สลับกันทีละอัน เพื่อ performance)
  const mirrorToUpdate = mirrors[frame % mirrors.length];
  mirrorToUpdate.mirror.visible = false;
  mirrorToUpdate.camera.position.copy(mirrorToUpdate.mirror.position);
  mirrorToUpdate.camera.update(renderer, scene);
  mirrorToUpdate.mirror.visible = true;
  
  frame++;
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

## 9. Water Surface Reflection

```javascript
// ===== Water Reflection =====
// ใช้ mirror plane technique

class WaterSurface {
  constructor(scene, renderer, camera) {
    this.scene = scene;
    this.renderer = renderer;
    this.camera = camera;
    
    this.reflectionCamera = this.camera.clone();
    this.reflectionTarget = new THREE.WebGLRenderTarget(
      window.innerWidth / 2, // half resolution
      window.innerHeight / 2,
      {
        format: THREE.RGBFormat,
        generateMipmaps: false,
        minFilter: THREE.LinearFilter
      }
    );
    
    this.buildMesh();
    this.buildMaterial();
  }
  
  buildMesh() {
    this.mesh = new THREE.Mesh(
      new THREE.PlaneGeometry(100, 100, 64, 64),
      null // จะตั้งค่า material ทีหลัง
    );
    this.mesh.rotation.x = -Math.PI / 2;
    this.mesh.position.y = 0;
    this.scene.add(this.mesh);
  }
  
  buildMaterial() {
    this.material = new THREE.ShaderMaterial({
      uniforms: {
        reflectionTexture: { value: this.reflectionTarget.texture },
        normalMap: { value: null },
        time: { value: 0 },
        waterColor: { value: new THREE.Color(0x1155aa) },
        resolution: { 
          value: new THREE.Vector2(window.innerWidth, window.innerHeight) 
        }
      },
      
      vertexShader: `
        uniform float time;
        varying vec2 vUv;
        varying vec4 vReflectCoord;
        varying vec3 vNormal;
        varying vec3 vViewDir;
        
        mat4 biasMatrix = mat4(
          0.5, 0.0, 0.0, 0.0,
          0.0, 0.5, 0.0, 0.0,
          0.0, 0.0, 0.5, 0.0,
          0.5, 0.5, 0.5, 1.0
        );
        
        void main() {
          vUv = uv;
          
          // Small wave displacement
          vec3 pos = position;
          pos.z += sin(pos.x * 2.0 + time) * 0.05;
          pos.z += sin(pos.y * 1.5 + time * 0.8) * 0.05;
          
          vec4 worldPos = modelMatrix * vec4(pos, 1.0);
          vViewDir = normalize(cameraPosition - worldPos.xyz);
          
          // Normal in world space
          vNormal = normalize(normalMatrix * normal);
          
          // Screen-space projection for reflection
          vReflectCoord = biasMatrix * projectionMatrix * viewMatrix * worldPos;
          
          gl_Position = projectionMatrix * viewMatrix * worldPos;
        }
      `,
      
      fragmentShader: `
        uniform sampler2D reflectionTexture;
        uniform float time;
        uniform vec3 waterColor;
        
        varying vec2 vUv;
        varying vec4 vReflectCoord;
        varying vec3 vNormal;
        varying vec3 vViewDir;
        
        void main() {
          vec3 N = normalize(vNormal);
          vec3 V = normalize(vViewDir);
          
          // Animated distortion for ripples
          vec2 distortion = vec2(
            sin(vUv.x * 15.0 + time * 1.5) * 0.02,
            cos(vUv.y * 15.0 + time * 1.2) * 0.02
          );
          
          // Sample reflection texture
          vec2 reflectUv = (vReflectCoord.xy / vReflectCoord.w) + distortion;
          reflectUv = clamp(reflectUv, 0.01, 0.99);
          vec3 reflection = texture2D(reflectionTexture, reflectUv).rgb;
          
          // Fresnel
          float fresnel = pow(1.0 - max(dot(N, V), 0.0), 3.0);
          
          // Blend water color and reflection
          vec3 finalColor = mix(waterColor * 0.3, reflection, 0.3 + fresnel * 0.7);
          
          gl_FragColor = vec4(finalColor, 0.9);
        }
      `,
      
      transparent: true,
      side: THREE.FrontSide
    });
    
    this.mesh.material = this.material;
  }
  
  update(deltaTime) {
    this.material.uniforms.time.value += deltaTime;
    
    // Render reflection
    this.mesh.visible = false;
    
    // Flip camera below water plane
    const camera = this.camera;
    const reflectCamera = this.reflectionCamera;
    
    reflectCamera.copy(camera);
    
    // Mirror camera below y=0
    reflectCamera.position.y = -camera.position.y;
    reflectCamera.rotation.x = -camera.rotation.x;
    reflectCamera.rotation.z = camera.rotation.z;
    
    reflectCamera.updateMatrixWorld();
    reflectCamera.updateProjectionMatrix();
    
    this.renderer.setRenderTarget(this.reflectionTarget);
    this.renderer.render(this.scene, reflectCamera);
    this.renderer.setRenderTarget(null);
    
    this.mesh.visible = true;
  }
}
```

## สรุป

Reflections & Refractions เป็น key techniques สำหรับ photorealistic rendering:

1. **CubeCamera** - real-time cube reflections
2. **MeshPhysicalMaterial** - built-in glass/refraction support
3. **Fresnel Effect** - physically correct reflection angle dependency
4. **Water Reflections** - plane reflection rendering

### Performance Considerations
- ลด CubeCamera resolution (128-256 เพียงพอ)
- Update mirrors สลับกัน (ไม่พร้อมกันทุก frame)
- ใช้ static environment maps เมื่อไม่จำเป็น
- Screen-space reflections เร็วกว่า CubeCamera แต่มีข้อจำกัด

---

## แบบฝึกหัด

1. **Easy**: สร้าง chrome sphere ที่สะท้อน environment โดยใช้ CubeCamera
2. **Medium**: สร้าง glass showcase box ที่มี refraction และ Fresnel effect
3. **Hard**: สร้าง mirror room ที่ bouncing reflections ดูสมจริง
4. **Challenge**: สร้าง realistic car paint material ที่มี clearcoat, metallic paint flakes, และ dynamic reflections
