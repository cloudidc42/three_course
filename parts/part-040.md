# Part 40: HDR & Advanced Lighting ใน Three.js

## บทนำ

การทำ Physically Accurate Lighting ต้องเข้าใจ HDR, Tone Mapping, Color Spaces และหน่วยวัดแสงจริง เนื้อหานี้จะพาคุณเข้าสู่โลกของการ render ที่แม่นยำทางฟิสิกส์

## 1. HDR Textures (RGBELoader)

HDR = High Dynamic Range - เก็บค่าแสงได้กว้างกว่า standard 8-bit มาก

```javascript
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js';
import * as THREE from 'three';

// RGBELoader สำหรับ .hdr files
const rgbeLoader = new RGBELoader();

rgbeLoader.load('/textures/studio_small_08_1k.hdr', (hdrTexture) => {
  hdrTexture.mapping = THREE.EquirectangularReflectionMapping;
  
  // PMREMGenerator สำหรับ PBR
  const pmremGenerator = new THREE.PMREMGenerator(renderer);
  const envMap = pmremGenerator.fromEquirectangular(hdrTexture).texture;
  
  scene.environment = envMap;
  scene.background = hdrTexture; // ใช้ HDR โดยตรงเป็น background
  
  hdrTexture.dispose();
  pmremGenerator.dispose();
});

// ===== EXRLoader สำหรับ .exr files =====
import { EXRLoader } from 'three/examples/jsm/loaders/EXRLoader.js';

const exrLoader = new EXRLoader();
exrLoader.load('/textures/environment.exr', (exrTexture) => {
  exrTexture.mapping = THREE.EquirectangularReflectionMapping;
  
  const pmrem = new THREE.PMREMGenerator(renderer);
  const envMap = pmrem.fromEquirectangular(exrTexture).texture;
  scene.environment = envMap;
});
```

## 2. Tone Mapping Deep Dive

Tone Mapping แปลง HDR values เป็น LDR (0-1) สำหรับแสดงผลบนหน้าจอ

```javascript
// ===== Available Tone Mapping =====

// Linear: ไม่มีการ tone map (raw values)
renderer.toneMapping = THREE.NoToneMapping;

// Reinhard: classic tone mapping
renderer.toneMapping = THREE.ReinhardToneMapping;

// Cineon: สีอบอุ่น มีลักษณะ film look
renderer.toneMapping = THREE.CineonToneMapping;

// ACESFilmic: ใช้ใน film/VFX อุตสาหกรรม (แนะนำที่สุด)
renderer.toneMapping = THREE.ACESFilmicToneMapping;

// Custom: ใช้ custom shader
renderer.toneMapping = THREE.CustomToneMapping;

// ===== Exposure =====
renderer.toneMappingExposure = 1.0; // ค่า default
// > 1 = สว่างขึ้น
// < 1 = มืดลง

// ===== ACES Filmic Math =====
// ฟังก์ชัน ACES approximation ที่ใช้ใน Three.js
/*
GLSL code:
vec3 ACESFilmicToneMapping(vec3 x) {
  float a = 2.51;
  float b = 0.03;
  float c = 2.43;
  float d = 0.59;
  float e = 0.14;
  return clamp((x * (a * x + b)) / (x * (c * x + d) + e), 0.0, 1.0);
}
*/

// JavaScript equivalent สำหรับทดสอบ
function acesFilmic(value) {
  const a = 2.51, b = 0.03, c = 2.43, d = 0.59, e = 0.14;
  return Math.max(0, Math.min(1, (value * (a * value + b)) / (value * (c * value + d) + e)));
}

// Test
console.log(acesFilmic(0.0)); // 0 (black)
console.log(acesFilmic(0.5)); // ~0.43 (mid tones)
console.log(acesFilmic(1.0)); // ~0.90 (highlights slightly compressed)
console.log(acesFilmic(5.0)); // ~0.99 (very bright, compressed)
```

## 3. Color Spaces

```javascript
// ===== Linear vs sRGB =====
// Linear color space: แสง add กันแบบ linear (correct for physics)
// sRGB: ถูกออกแบบมาสำหรับหน้าจอ (gamma ~2.2)

// Three.js ใช้ LinearSRGBColorSpace สำหรับ internal calculations
// ต้อง convert ตอน input และ output

// ===== Output Color Space =====
// สำหรับ renderer output
renderer.outputColorSpace = THREE.SRGBColorSpace; // default ใน Three.js r152+
// เก่ากว่า: renderer.outputEncoding = THREE.sRGBEncoding;

// ===== Texture Color Spaces =====
// Textures ที่เป็น color data (diffuse/albedo) ต้อง decode จาก sRGB
const colorTexture = textureLoader.load('diffuse.jpg');
colorTexture.colorSpace = THREE.SRGBColorSpace; // default สำหรับ jpg/png

// Non-color textures (normal maps, roughness, etc.) ไม่ต้อง decode
const normalMap = textureLoader.load('normal.jpg');
normalMap.colorSpace = THREE.NoColorSpace; // linear, no gamma correction

// ===== Color ใน Materials =====
// ตอนสร้าง Color:
// Three.js Color เก็บ linear values
const red = new THREE.Color(0xff0000); // auto convert จาก sRGB ถ้าตั้งค่าถูก

// Explicit setting
const color = new THREE.Color();
color.setRGB(1.0, 0.0, 0.0, THREE.LinearSRGBColorSpace); // linear
color.setRGB(1.0, 0.0, 0.0, THREE.SRGBColorSpace); // sRGB

// ===== Gamma Correction Formula =====
function linearToSRGB(value) {
  return value <= 0.0031308
    ? value * 12.92
    : 1.055 * Math.pow(value, 1.0 / 2.4) - 0.055;
}

function sRGBToLinear(value) {
  return value <= 0.04045
    ? value / 12.92
    : Math.pow((value + 0.055) / 1.055, 2.4);
}
```

## 4. Physical Light Units

Three.js รองรับหน่วยแสงทางฟิสิกส์ผ่าน `physicallyCorrectLights`

```javascript
// เปิด physically correct lighting
renderer.useLegacyLights = false; // Three.js r152+
// เก่ากว่า: renderer.physicallyCorrectLights = true;

// ===== Light Units =====
// PointLight: candela (cd) = lumens per steradian
const pointLight = new THREE.PointLight(0xffffff);
pointLight.intensity = 100; // 100 candela = หลอดไฟ 100W ประมาณนี้
pointLight.decay = 2; // ต้องเป็น 2 สำหรับ physically correct

// SpotLight: candela (cd)
const spotLight = new THREE.SpotLight(0xffffff);
spotLight.intensity = 200;
spotLight.decay = 2;

// DirectionalLight: lux (lux = lm/m²)
const dirLight = new THREE.DirectionalLight(0xffffff);
dirLight.intensity = 100000; // แสงแดดตอนเที่ยง ~100,000 lux

// AmbientLight: lux
const ambLight = new THREE.AmbientLight(0xffffff);
ambLight.intensity = 0.5; // dim ambient

// ===== Real-world Reference Values =====
const lightValues = {
  candle: 12.56,           // 1 cd * 4π sr = ~12.56 lumens
  60wBulb: 806,            // lumens
  100wBulb: 1600,          // lumens
  officeLighting: 500,     // lux (lm/m²)
  overcastSky: 1000,       // lux
  sunnyDay: 100000,        // lux
  directSunlight: 120000,  // lux
};
```

## 5. Demo 1: Physically Accurate Lighting Scene

```html
<!DOCTYPE html>
<html>
<head>
  <title>Physically Accurate Lighting</title>
  <style>
    body { margin: 0; background: #000; }
    canvas { display: block; }
    #controls {
      position: fixed;
      top: 20px;
      right: 20px;
      background: rgba(0,0,0,0.85);
      color: #ddd;
      padding: 20px;
      border-radius: 10px;
      font-family: monospace;
      font-size: 13px;
      width: 250px;
    }
    h3 { color: #8af; margin: 0 0 10px; }
    .group { margin: 10px 0; border-top: 1px solid #333; padding-top: 8px; }
    label { display: block; margin-bottom: 2px; }
    input[type="range"] { width: 200px; }
    .val { color: #fa8; }
  </style>
</head>
<body>
<div id="controls">
  <h3>Physical Lighting</h3>
  
  <div class="group">
    <b style="color:#ff8">Sun Light</b>
    <label>Intensity (lux): <span class="val" id="sunVal">30000</span></label>
    <input type="range" id="sunIntensity" min="0" max="200000" value="30000">
    <label>Elevation: <span class="val" id="sunElevVal">45</span>°</label>
    <input type="range" id="sunElevation" min="0" max="90" value="45">
    <label>Azimuth: <span class="val" id="sunAziVal">0</span>°</label>
    <input type="range" id="sunAzimuth" min="0" max="360" value="0">
  </div>
  
  <div class="group">
    <b style="color:#8ff">Indoor Light</b>
    <label>Intensity (cd): <span class="val" id="indoorVal">400</span></label>
    <input type="range" id="indoorIntensity" min="0" max="2000" value="400">
  </div>
  
  <div class="group">
    <b style="color:#88f">Tone Mapping</b>
    <select id="toneMapping" style="background:#222;color:#fff;width:200px">
      <option value="none">No Tone Mapping</option>
      <option value="linear">Linear</option>
      <option value="reinhard">Reinhard</option>
      <option value="cineon">Cineon</option>
      <option value="aces" selected>ACES Filmic</option>
    </select>
    <label>Exposure: <span class="val" id="exposureVal">1.0</span></label>
    <input type="range" id="exposure" min="0" max="400" value="100">
  </div>
</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';
import { RoomEnvironment } from 'https://unpkg.com/three@0.158.0/examples/jsm/environments/RoomEnvironment.js';

// ===== SETUP =====
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x87ceeb); // Sky blue

const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 1000);
camera.position.set(5, 3, 8);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(devicePixelRatio);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.0;
renderer.outputColorSpace = THREE.SRGBColorSpace;
renderer.useLegacyLights = false; // Physical lights
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.target.set(0, 0, 0);

// ===== ENVIRONMENT =====
const pmrem = new THREE.PMREMGenerator(renderer);
const envTexture = pmrem.fromScene(new RoomEnvironment(), 0.04).texture;
scene.environment = envTexture;

// ===== LIGHTS =====
// Sun (DirectionalLight with physically correct intensity)
const sunLight = new THREE.DirectionalLight(0xfff8e0, 30000);
sunLight.castShadow = true;
sunLight.shadow.mapSize.set(2048, 2048);
sunLight.shadow.camera.near = 1;
sunLight.shadow.camera.far = 100;
sunLight.shadow.camera.left = -15;
sunLight.shadow.camera.right = 15;
sunLight.shadow.camera.top = 15;
sunLight.shadow.camera.bottom = -15;
sunLight.shadow.bias = -0.001;
scene.add(sunLight);

function updateSunPosition(elevation, azimuth) {
  const elevRad = elevation * Math.PI / 180;
  const aziRad = azimuth * Math.PI / 180;
  
  sunLight.position.set(
    Math.cos(elevRad) * Math.sin(aziRad) * 50,
    Math.sin(elevRad) * 50,
    Math.cos(elevRad) * Math.cos(aziRad) * 50
  );
  
  // Change sky color based on sun elevation
  const dayColor = new THREE.Color(0x87ceeb);
  const sunsetColor = new THREE.Color(0xff8844);
  const nightColor = new THREE.Color(0x001122);
  
  const t = elevation / 90;
  if (t > 0.1) {
    scene.background = dayColor.clone().lerp(sunsetColor, 1 - t);
  } else {
    scene.background = sunsetColor.clone().lerp(nightColor, 1 - t * 10);
  }
  
  // Change sun color at low elevation (golden hour)
  const sunColor = new THREE.Color();
  sunColor.setRGB(1.0, 0.6 + t * 0.4, 0.3 + t * 0.7);
  sunLight.color.copy(sunColor);
}

// Indoor point lights
const indoorLights = [];
const indoorPositions = [
  [-4, 4, -4], [4, 4, -4], [0, 4, 0]
];

indoorPositions.forEach(pos => {
  const light = new THREE.PointLight(0xfff5e0, 400, 20, 2); // candela, decay=2
  light.position.set(...pos);
  light.castShadow = true;
  scene.add(light);
  
  // Visual bulb
  const bulb = new THREE.Mesh(
    new THREE.SphereGeometry(0.15, 16, 16),
    new THREE.MeshStandardMaterial({
      emissive: 0xfff5e0,
      emissiveIntensity: 3,
      color: 0x000000
    })
  );
  bulb.position.copy(light.position);
  scene.add(bulb);
  
  indoorLights.push(light);
});

// ===== SCENE OBJECTS =====
// Ground
const groundMat = new THREE.MeshStandardMaterial({
  color: 0xccbb99,
  roughness: 0.9,
  metalness: 0.0
});
const ground = new THREE.Mesh(new THREE.PlaneGeometry(30, 30), groundMat);
ground.rotation.x = -Math.PI / 2;
ground.position.y = -1;
ground.receiveShadow = true;
scene.add(ground);

// Building (box)
const buildingMat = new THREE.MeshStandardMaterial({ color: 0xddccbb, roughness: 0.8 });
const building = new THREE.Mesh(new THREE.BoxGeometry(8, 5, 8), buildingMat);
building.position.set(0, 1.5, 0);
building.receiveShadow = true;
building.castShadow = true;
scene.add(building);

// PBR Material Showcase
const materialData = [
  { color: 0xffffff, roughness: 0.0, metalness: 1.0, name: 'Chrome' },
  { color: 0xffd700, roughness: 0.1, metalness: 1.0, name: 'Gold' },
  { color: 0x4488ff, roughness: 0.5, metalness: 0.0, name: 'Plastic' },
  { color: 0xffddcc, roughness: 0.7, metalness: 0.0, name: 'Ceramic' },
  { color: 0x8866aa, roughness: 0.2, metalness: 0.8, name: 'Metal' },
];

materialData.forEach((data, i) => {
  const mat = new THREE.MeshStandardMaterial({
    color: data.color,
    roughness: data.roughness,
    metalness: data.metalness
  });
  
  const sphere = new THREE.Mesh(new THREE.SphereGeometry(0.5, 32, 32), mat);
  sphere.position.set(-4 + i * 2, 1, -3);
  sphere.castShadow = true;
  scene.add(sphere);
});

// ===== CONTROLS =====
document.getElementById('sunIntensity').addEventListener('input', (e) => {
  const val = parseInt(e.target.value);
  sunLight.intensity = val;
  document.getElementById('sunVal').textContent = val;
});

let sunElev = 45, sunAzi = 0;

document.getElementById('sunElevation').addEventListener('input', (e) => {
  sunElev = parseInt(e.target.value);
  document.getElementById('sunElevVal').textContent = sunElev;
  updateSunPosition(sunElev, sunAzi);
});

document.getElementById('sunAzimuth').addEventListener('input', (e) => {
  sunAzi = parseInt(e.target.value);
  document.getElementById('sunAziVal').textContent = sunAzi;
  updateSunPosition(sunElev, sunAzi);
});

updateSunPosition(45, 0);

document.getElementById('indoorIntensity').addEventListener('input', (e) => {
  const val = parseInt(e.target.value);
  indoorLights.forEach(l => l.intensity = val);
  document.getElementById('indoorVal').textContent = val;
});

document.getElementById('toneMapping').addEventListener('change', (e) => {
  const mappings = {
    'none': THREE.NoToneMapping,
    'linear': THREE.LinearToneMapping,
    'reinhard': THREE.ReinhardToneMapping,
    'cineon': THREE.CineonToneMapping,
    'aces': THREE.ACESFilmicToneMapping
  };
  renderer.toneMapping = mappings[e.target.value];
});

document.getElementById('exposure').addEventListener('input', (e) => {
  const val = e.target.value / 100;
  renderer.toneMappingExposure = val;
  document.getElementById('exposureVal').textContent = val.toFixed(2);
});

// ===== ANIMATION =====
const clock = new THREE.Clock();
function animate() {
  requestAnimationFrame(animate);
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

## 6. Day/Night Cycle

```javascript
// ===== Complete Day/Night Cycle System =====
class DayNightCycle {
  constructor(scene, renderer) {
    this.scene = scene;
    this.renderer = renderer;
    this.time = 0.25; // 0=midnight, 0.25=sunrise, 0.5=noon, 0.75=sunset, 1=midnight
    this.speed = 0.01; // cycle speed (1 = full day per minute)
    
    this.setupLights();
    this.setupSky();
    this.setupStars();
    this.setupMoon();
  }
  
  setupLights() {
    // Sun
    this.sunLight = new THREE.DirectionalLight(0xffffff, 0);
    this.sunLight.castShadow = true;
    this.scene.add(this.sunLight);
    
    // Moon
    this.moonLight = new THREE.DirectionalLight(0x8899cc, 0);
    this.scene.add(this.moonLight);
    
    // Ambient (sky light)
    this.ambientLight = new THREE.AmbientLight(0x334466, 0);
    this.scene.add(this.ambientLight);
    
    // Hemisphere light
    this.hemiLight = new THREE.HemisphereLight(0x8899bb, 0x443322, 0);
    this.scene.add(this.hemiLight);
  }
  
  setupSky() {
    // Sky colors for different times
    this.skyColors = {
      midnight: new THREE.Color(0x000511),
      dawn: new THREE.Color(0x1a0a2a),
      sunrise: new THREE.Color(0xff6633),
      morning: new THREE.Color(0x87ceeb),
      noon: new THREE.Color(0x4488dd),
      afternoon: new THREE.Color(0x6699cc),
      sunset: new THREE.Color(0xff5500),
      dusk: new THREE.Color(0x220a33),
    };
    
    this.scene.background = new THREE.Color(0x000511);
  }
  
  setupStars() {
    const starCount = 5000;
    const positions = new Float32Array(starCount * 3);
    
    for (let i = 0; i < starCount; i++) {
      const theta = Math.random() * Math.PI * 2;
      const phi = Math.acos(2 * Math.random() - 1);
      const r = 500;
      
      positions[i * 3] = r * Math.sin(phi) * Math.cos(theta);
      positions[i * 3 + 1] = r * Math.sin(phi) * Math.sin(theta);
      positions[i * 3 + 2] = r * Math.cos(phi);
    }
    
    const starGeo = new THREE.BufferGeometry();
    starGeo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    
    this.stars = new THREE.Points(
      starGeo,
      new THREE.PointsMaterial({
        color: 0xffffff,
        size: 0.5,
        transparent: true,
        opacity: 1.0
      })
    );
    
    this.scene.add(this.stars);
  }
  
  setupMoon() {
    this.moonMesh = new THREE.Mesh(
      new THREE.SphereGeometry(5, 32, 32),
      new THREE.MeshStandardMaterial({
        color: 0xddddcc,
        roughness: 0.9,
        emissive: 0x223333,
        emissiveIntensity: 0.3
      })
    );
    this.scene.add(this.moonMesh);
  }
  
  // คำนวณ sun/moon position จาก time
  getSunPosition(t) {
    const angle = (t - 0.25) * Math.PI * 2; // noon at t=0.5
    return new THREE.Vector3(
      Math.cos(angle) * 500,
      Math.sin(angle) * 500,
      100
    );
  }
  
  getMoonPosition(t) {
    const angle = (t + 0.25) * Math.PI * 2; // opposite to sun
    return new THREE.Vector3(
      Math.cos(angle) * 500,
      Math.sin(angle) * 500,
      -100
    );
  }
  
  // Get sky color for current time
  getSkyColor(t) {
    const colors = this.skyColors;
    
    if (t < 0.2) { // midnight to dawn
      return colors.midnight.clone().lerp(colors.dawn, t / 0.2);
    } else if (t < 0.27) { // dawn to sunrise
      return colors.dawn.clone().lerp(colors.sunrise, (t - 0.2) / 0.07);
    } else if (t < 0.35) { // sunrise to morning
      return colors.sunrise.clone().lerp(colors.morning, (t - 0.27) / 0.08);
    } else if (t < 0.45) { // morning to noon
      return colors.morning.clone().lerp(colors.noon, (t - 0.35) / 0.1);
    } else if (t < 0.55) { // noon
      return colors.noon.clone();
    } else if (t < 0.65) { // afternoon
      return colors.noon.clone().lerp(colors.afternoon, (t - 0.55) / 0.1);
    } else if (t < 0.73) { // sunset
      return colors.afternoon.clone().lerp(colors.sunset, (t - 0.65) / 0.08);
    } else if (t < 0.80) { // dusk
      return colors.sunset.clone().lerp(colors.dusk, (t - 0.73) / 0.07);
    } else { // night
      return colors.dusk.clone().lerp(colors.midnight, (t - 0.8) / 0.2);
    }
  }
  
  update(delta) {
    this.time = (this.time + delta * this.speed * 0.1) % 1;
    
    const t = this.time;
    const sunPos = this.getSunPosition(t);
    const moonPos = this.getMoonPosition(t);
    
    // Sun above horizon?
    const sunAbove = sunPos.y > 0;
    const moonAbove = moonPos.y > 0;
    
    // Update light positions
    this.sunLight.position.copy(sunPos);
    this.moonLight.position.copy(moonPos);
    
    // Sun intensity (0 at horizon, max at noon)
    const sunAngle = Math.max(0, sunPos.y / 500);
    this.sunLight.intensity = sunAbove ? sunAngle * 100000 : 0;
    
    // Sun color (white at noon, orange at sunrise/sunset)
    const sunAngleNorm = sunAngle;
    const sunColor = new THREE.Color();
    sunColor.setRGB(1.0, 0.5 + sunAngleNorm * 0.5, sunAngleNorm);
    this.sunLight.color.copy(sunColor);
    
    // Moon intensity
    const moonAngle = Math.max(0, moonPos.y / 500);
    this.moonLight.intensity = moonAbove ? moonAngle * 2 : 0;
    
    // Ambient light
    this.ambientLight.intensity = sunAbove ? sunAngle * 5000 : moonAngle * 50;
    
    // Hemisphere
    this.hemiLight.intensity = sunAbove ? sunAngle * 3 : 0.1;
    
    // Sky color
    this.scene.background = this.getSkyColor(t);
    
    // Stars visibility (visible at night)
    this.stars.material.opacity = Math.max(0, 1 - sunAngle * 5);
    
    // Moon position
    this.moonMesh.position.copy(moonPos.clone().normalize().multiplyScalar(200));
    this.moonMesh.lookAt(0, 0, 0);
    
    // Exposure adjust for night/day
    const targetExposure = sunAbove ? 0.8 + sunAngle : 1.5;
    this.renderer.toneMappingExposure = THREE.MathUtils.lerp(
      this.renderer.toneMappingExposure, 
      targetExposure, 
      0.02
    );
    
    return {
      timeOfDay: t,
      hour: Math.floor(t * 24),
      sunHeight: sunPos.y
    };
  }
}
```

## 7. Studio Lighting Setup

```javascript
// ===== Professional 3-Point Studio Lighting =====
class StudioLighting {
  constructor(scene) {
    this.scene = scene;
    this.lights = {};
    this.setup();
  }
  
  setup() {
    // Key Light - หลักที่สว่างที่สุด
    this.lights.key = new THREE.DirectionalLight(0xfff8f0, 2000); // warm white
    this.lights.key.position.set(3, 5, 3);
    this.lights.key.castShadow = true;
    this.lights.key.shadow.mapSize.set(2048, 2048);
    this.scene.add(this.lights.key);
    
    // Fill Light - ลดเงาด้านที่มืด
    this.lights.fill = new THREE.DirectionalLight(0xf0f8ff, 500); // cool white
    this.lights.fill.position.set(-3, 2, 2);
    this.scene.add(this.lights.fill);
    
    // Back/Rim Light - แยก subject จาก background
    this.lights.back = new THREE.DirectionalLight(0xffffff, 1000);
    this.lights.back.position.set(0, 3, -5);
    this.scene.add(this.lights.back);
    
    // Ambient (very low)
    this.lights.ambient = new THREE.AmbientLight(0x111122, 50);
    this.scene.add(this.lights.ambient);
    
    // Optional: Area light (RectAreaLight)
    this.setupAreaLights();
  }
  
  setupAreaLights() {
    // RectAreaLight สำหรับ soft box effect
    const { RectAreaLight } = THREE;
    
    this.lights.softbox1 = new RectAreaLight(0xfff5e0, 500, 3, 3);
    this.lights.softbox1.position.set(4, 4, 2);
    this.lights.softbox1.lookAt(0, 0, 0);
    this.scene.add(this.lights.softbox1);
    
    this.lights.softbox2 = new RectAreaLight(0xe0f0ff, 200, 2, 4);
    this.lights.softbox2.position.set(-3, 3, 1);
    this.lights.softbox2.lookAt(0, 0, 0);
    this.scene.add(this.lights.softbox2);
    
    // Visual helpers สำหรับ development
    // import { RectAreaLightHelper } from 'three/examples/jsm/helpers/RectAreaLightHelper.js';
    // const helper = new RectAreaLightHelper(this.lights.softbox1);
    // this.scene.add(helper);
  }
  
  // Preset: Product Photography
  setProductMode() {
    this.lights.key.intensity = 3000;
    this.lights.key.color.set(0xffffff);
    this.lights.fill.intensity = 1000;
    this.lights.back.intensity = 1500;
    this.lights.ambient.intensity = 100;
  }
  
  // Preset: Portrait
  setPortraitMode() {
    this.lights.key.intensity = 2000;
    this.lights.key.color.set(0xfff5e0); // warm
    this.lights.fill.intensity = 300;
    this.lights.fill.color.set(0xe0f0ff); // cool fill
    this.lights.back.intensity = 800;
    this.lights.ambient.intensity = 50;
  }
  
  // Preset: Dramatic
  setDramaticMode() {
    this.lights.key.intensity = 5000;
    this.lights.key.color.set(0xff8844);
    this.lights.fill.intensity = 100; // very low fill = deep shadows
    this.lights.back.intensity = 300;
    this.lights.ambient.intensity = 20;
  }
}
```

## 8. Demo 2: Day/Night Cycle Scene

```html
<!DOCTYPE html>
<html>
<head>
  <title>Day/Night Cycle</title>
  <style>
    body { margin: 0; overflow: hidden; }
    canvas { display: block; }
    #timeDisplay {
      position: fixed;
      top: 20px;
      left: 50%;
      transform: translateX(-50%);
      color: white;
      font-family: monospace;
      font-size: 20px;
      text-shadow: 0 2px 10px rgba(0,0,0,0.8);
      pointer-events: none;
    }
    #controls {
      position: fixed;
      bottom: 20px;
      left: 50%;
      transform: translateX(-50%);
      display: flex;
      gap: 10px;
      align-items: center;
    }
    button {
      padding: 8px 20px;
      background: rgba(0,0,0,0.7);
      color: white;
      border: 1px solid #888;
      border-radius: 4px;
      cursor: pointer;
    }
    input[type="range"] { width: 300px; }
  </style>
</head>
<body>
<div id="timeDisplay">🕐 12:00</div>
<div id="controls">
  <button id="toggleBtn">⏸ Pause</button>
  <button id="speedBtn">Speed: 1x</button>
  <input type="range" id="timeSlider" min="0" max="1000" value="500">
  <span id="speedDisplay" style="color:white;font-family:monospace">1x</span>
</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';
import { RoomEnvironment } from 'https://unpkg.com/three@0.158.0/examples/jsm/environments/RoomEnvironment.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 2000);
camera.position.set(10, 5, 15);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(devicePixelRatio);
renderer.shadowMap.enabled = true;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.0;
renderer.outputColorSpace = THREE.SRGBColorSpace;
renderer.useLegacyLights = false;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.target.set(0, 2, 0);

// ===== DAY/NIGHT CYCLE =====
let dayTime = 0.5; // noon
let isPlaying = true;
let speed = 1;

// Sun
const sunLight = new THREE.DirectionalLight(0xffffff, 30000);
sunLight.castShadow = true;
sunLight.shadow.mapSize.set(2048, 2048);
sunLight.shadow.camera.near = 0.5;
sunLight.shadow.camera.far = 500;
sunLight.shadow.camera.left = -30;
sunLight.shadow.camera.right = 30;
sunLight.shadow.camera.top = 30;
sunLight.shadow.camera.bottom = -30;
scene.add(sunLight);

// Moon
const moonLight = new THREE.DirectionalLight(0x8899cc, 1.5);
scene.add(moonLight);

// Hemisphere
const hemiLight = new THREE.HemisphereLight(0x88aaff, 0x443322, 2);
scene.add(hemiLight);

// Stars
const starGeo = new THREE.BufferGeometry();
const starPositions = [];
for (let i = 0; i < 3000; i++) {
  const theta = Math.random() * Math.PI * 2;
  const phi = Math.acos(2 * Math.random() - 1);
  const r = 800 + Math.random() * 200;
  starPositions.push(
    r * Math.sin(phi) * Math.cos(theta),
    Math.abs(r * Math.cos(phi)),
    r * Math.sin(phi) * Math.sin(theta)
  );
}
starGeo.setAttribute('position', new THREE.Float32BufferAttribute(starPositions, 3));
const stars = new THREE.Points(starGeo, new THREE.PointsMaterial({
  color: 0xffffff, size: 1.5, transparent: true, opacity: 0
}));
scene.add(stars);

// ===== SCENE OBJECTS =====
// Ground
const ground = new THREE.Mesh(
  new THREE.PlaneGeometry(60, 60),
  new THREE.MeshStandardMaterial({ color: 0x44aa44, roughness: 0.9 })
);
ground.rotation.x = -Math.PI / 2;
ground.receiveShadow = true;
scene.add(ground);

// Houses
function createHouse(x, z, color) {
  const group = new THREE.Group();
  
  const body = new THREE.Mesh(
    new THREE.BoxGeometry(3, 2, 3),
    new THREE.MeshStandardMaterial({ color, roughness: 0.8 })
  );
  body.position.y = 1;
  body.castShadow = true;
  body.receiveShadow = true;
  group.add(body);
  
  const roof = new THREE.Mesh(
    new THREE.ConeGeometry(2.5, 1.5, 4),
    new THREE.MeshStandardMaterial({ color: 0x882222, roughness: 0.7 })
  );
  roof.position.y = 2.75;
  roof.rotation.y = Math.PI / 4;
  roof.castShadow = true;
  group.add(roof);
  
  // Window (emissive - glows at night)
  const window = new THREE.Mesh(
    new THREE.PlaneGeometry(0.6, 0.6),
    new THREE.MeshStandardMaterial({
      color: 0xffee99,
      emissive: 0xffee99,
      emissiveIntensity: 0,
      side: THREE.DoubleSide
    })
  );
  window.position.set(0, 1, 1.51);
  group.add(window);
  group.userData.window = window;
  
  group.position.set(x, 0, z);
  return group;
}

const houses = [
  createHouse(-6, -5, 0xddccbb),
  createHouse(5, -3, 0xccbbaa),
  createHouse(-3, 4, 0xbbccdd),
  createHouse(7, 5, 0xddbbcc),
];

houses.forEach(h => scene.add(h));

// Trees
function createTree(x, z) {
  const group = new THREE.Group();
  
  const trunk = new THREE.Mesh(
    new THREE.CylinderGeometry(0.1, 0.15, 1.5),
    new THREE.MeshStandardMaterial({ color: 0x553311, roughness: 0.9 })
  );
  trunk.position.y = 0.75;
  trunk.castShadow = true;
  group.add(trunk);
  
  for (let i = 0; i < 3; i++) {
    const cone = new THREE.Mesh(
      new THREE.ConeGeometry(1.2 - i * 0.2, 1.5, 8),
      new THREE.MeshStandardMaterial({ color: 0x226633, roughness: 0.8 })
    );
    cone.position.y = 1.5 + i * 0.8;
    cone.castShadow = true;
    group.add(cone);
  }
  
  group.position.set(x, 0, z);
  return group;
}

const treePositions = [[-8,2],[3,8],[8,-7],[-5,-8],[0,6],[-10,-3]];
treePositions.forEach(([x,z]) => scene.add(createTree(x, z)));

// ===== UPDATE FUNCTION =====
function updateDayNight(t) {
  // Sun position (t=0.5 = noon)
  const sunAngle = (t - 0.5) * Math.PI * 2;
  const sunX = Math.cos(sunAngle) * 500;
  const sunY = Math.sin(sunAngle) * 500;
  sunLight.position.set(sunX, sunY, 200);
  
  const moonAngle = sunAngle + Math.PI;
  moonLight.position.set(
    Math.cos(moonAngle) * 500,
    Math.sin(moonAngle) * 500,
    -200
  );
  
  // Normalize sun height (-1 to 1)
  const sunHeight = sunY / 500;
  const sunAbove = sunHeight > 0;
  const sunIntensityFactor = Math.max(0, sunHeight);
  
  // Sun light
  sunLight.intensity = sunAbove ? sunIntensityFactor * 80000 : 0;
  
  const sunColorTemp = sunIntensityFactor;
  sunLight.color.setRGB(1.0, 0.6 + sunColorTemp * 0.4, 0.2 + sunColorTemp * 0.8);
  
  // Moon
  const moonHeight = Math.max(0, -sunHeight);
  moonLight.intensity = moonHeight * 3;
  
  // Hemisphere
  hemiLight.intensity = sunAbove ? sunIntensityFactor * 3 : 0.1;
  hemiLight.color.setRGB(0.5 + sunIntensityFactor * 0.3, 0.6 + sunIntensityFactor * 0.4, 0.7 + sunIntensityFactor * 0.3);
  
  // Sky color
  const nightSky = new THREE.Color(0x000511);
  const daySky = new THREE.Color(0x4488dd);
  const sunriseSky = new THREE.Color(0xff6633);
  
  let skyColor;
  if (sunIntensityFactor > 0.2) {
    skyColor = daySky;
  } else if (sunIntensityFactor > 0) {
    skyColor = sunriseSky.clone().lerp(daySky, sunIntensityFactor / 0.2);
  } else if (sunHeight > -0.1) {
    skyColor = nightSky.clone().lerp(sunriseSky, (sunHeight + 0.1) / 0.1);
  } else {
    skyColor = nightSky;
  }
  scene.background = skyColor;
  
  // Stars
  stars.material.opacity = Math.max(0, -sunHeight * 3);
  
  // Window glow (night only)
  const windowGlow = Math.max(0, -sunHeight * 2);
  houses.forEach(h => {
    h.userData.window.material.emissiveIntensity = windowGlow * 3;
  });
  
  // Exposure
  renderer.toneMappingExposure = sunAbove 
    ? 0.8 + sunIntensityFactor * 0.2 
    : 1.5 + moonHeight * 0.5;
  
  // Time string
  const hour = Math.floor(t * 24);
  const minute = Math.floor((t * 24 - hour) * 60);
  document.getElementById('timeDisplay').textContent = 
    `🕐 ${String(hour).padStart(2, '0')}:${String(minute).padStart(2, '0')}`;
}

// ===== INPUT =====
let userDragging = false;

document.getElementById('timeSlider').addEventListener('mousedown', () => userDragging = true);
document.getElementById('timeSlider').addEventListener('mouseup', () => userDragging = false);
document.getElementById('timeSlider').addEventListener('input', (e) => {
  dayTime = e.target.value / 1000;
});

document.getElementById('toggleBtn').addEventListener('click', (e) => {
  isPlaying = !isPlaying;
  e.target.textContent = isPlaying ? '⏸ Pause' : '▶ Play';
});

const speeds = [0.5, 1, 2, 5, 10];
let speedIndex = 1;
document.getElementById('speedBtn').addEventListener('click', () => {
  speedIndex = (speedIndex + 1) % speeds.length;
  speed = speeds[speedIndex];
  document.getElementById('speedBtn').textContent = `Speed: ${speed}x`;
});

// ===== ANIMATION =====
const clock = new THREE.Clock();

function animate() {
  requestAnimationFrame(animate);
  
  const delta = clock.getDelta();
  
  if (isPlaying && !userDragging) {
    dayTime = (dayTime + delta * 0.01 * speed) % 1;
    document.getElementById('timeSlider').value = dayTime * 1000;
  }
  
  updateDayNight(dayTime);
  
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

Advanced Lighting ใน Three.js ต้องการเข้าใจ:

1. **HDR** - High Dynamic Range สำหรับ accurate lighting
2. **Tone Mapping** - แปลง HDR เป็น LDR ที่สวยงาม
3. **Color Spaces** - Linear vs sRGB สำหรับ accurate colors
4. **Physical Units** - Candela, Lux, Lumen สำหรับ realistic lights
5. **Day/Night Cycle** - Dynamic lighting ตามเวลา

---

## แบบฝึกหัด

1. **Easy**: ทดลอง tone mapping แต่ละแบบบน metallic sphere และสังเกตความต่าง
2. **Medium**: สร้าง day/night cycle ที่มี realistic sky, stars, และ window lights
3. **Hard**: สร้าง studio lighting rig ด้วย 3-point lighting ที่ถูกต้องตาม physical units
4. **Challenge**: Implement custom tone mapping shader ที่ให้ look แตกต่างจาก built-in options
