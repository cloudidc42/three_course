# Part 36: Morph Targets (Shape Keys) ใน Three.js

## บทนำ

Morph Targets หรือที่รู้จักกันในชื่อ Shape Keys ใน Blender เป็นเทคนิคการทำ Animation ที่ทรงพลังมาก ซึ่งช่วยให้เราสามารถเปลี่ยนรูปร่างของ Geometry ได้อย่างราบรื่น โดยการ interpolate ระหว่างรูปทรงต่างๆ ที่กำหนดไว้ล่วงหน้า

เทคนิคนี้ถูกใช้กันอย่างแพร่หลายในวงการ:
- **Facial Animation** - การแสดงอารมณ์บนใบหน้าตัวละคร
- **Character Morphing** - การเปลี่ยนรูปร่างตัวละคร
- **Blendshapes ใน Game Engine** - ใช้ในการทำ lip sync และ facial expression
- **Soft Body Simulation** - จำลองการเคลื่อนไหวของวัตถุยืดหยุ่น

## 1. ทำความเข้าใจ morphAttributes

ใน Three.js, Morph Targets ถูกเก็บอยู่ใน `geometry.morphAttributes` ซึ่งเป็น dictionary ที่เก็บ BufferAttribute สำหรับแต่ละ attribute ที่ต้องการ morph

```javascript
// โครงสร้างของ morphAttributes
geometry.morphAttributes = {
  position: [
    BufferAttribute, // morph target 0
    BufferAttribute, // morph target 1
    BufferAttribute, // morph target 2
    // ...
  ],
  normal: [
    BufferAttribute, // normals สำหรับ morph target 0
    // ...
  ]
}
```

### การสร้าง Morph Target แบบ Manual

```javascript
import * as THREE from 'three';

// สร้าง Base Geometry
const geometry = new THREE.SphereGeometry(1, 32, 32);

// สร้าง Morph Target - เปลี่ยนเป็นรูปทรงอื่น
const morphPositions = [];

// Copy positions จาก base geometry มาแก้ไข
const basePositions = geometry.attributes.position.array;
const morphPos = new Float32Array(basePositions.length);

for (let i = 0; i < basePositions.length; i += 3) {
  morphPos[i] = basePositions[i] * 1.5;     // ขยาย x
  morphPos[i + 1] = basePositions[i + 1] * 0.5; // บีบ y
  morphPos[i + 2] = basePositions[i + 2] * 1.5; // ขยาย z
}

// เพิ่ม morph target เข้าไปใน geometry
geometry.morphAttributes.position = [
  new THREE.BufferAttribute(morphPos, 3)
];

// สร้าง material ที่รองรับ morph targets
const material = new THREE.MeshStandardMaterial({
  color: 0x4488ff,
  morphTargets: true
});

const mesh = new THREE.Mesh(geometry, material);
scene.add(mesh);

// ควบคุม morph target influence (0 = base shape, 1 = full morph)
mesh.morphTargetInfluences[0] = 0.5; // morph 50%
```

## 2. morphTargetInfluences

`morphTargetInfluences` คือ array ของค่า float ที่ควบคุมความแรงของแต่ละ morph target

```javascript
// mesh.morphTargetInfluences คือ array ขนาดเท่ากับจำนวน morph targets
console.log(mesh.morphTargetInfluences); // [0, 0, 0, ...]

// ตั้งค่า influence สำหรับแต่ละ morph target
mesh.morphTargetInfluences[0] = 1.0; // เปิด morph target 0 เต็มที่
mesh.morphTargetInfluences[1] = 0.5; // เปิด morph target 1 ครึ่งหนึ่ง
mesh.morphTargetInfluences[2] = 0.0; // ปิด morph target 2

// หมายเหตุ: ผลรวมของ influences ไม่จำเป็นต้องเท่ากับ 1
// Three.js จะ blend ทุก morph targets เข้าด้วยกัน
```

### morphTargetDictionary

```javascript
// morphTargetDictionary ช่วยให้เข้าถึง morph target ด้วยชื่อ
// แทนที่จะใช้ index

geometry.morphAttributes.position = [morphPos1, morphPos2];
mesh.morphTargetDictionary = {
  'smile': 0,
  'frown': 1,
  'eyesClosed': 2
};

// เข้าถึงด้วยชื่อ
const smileIndex = mesh.morphTargetDictionary['smile'];
mesh.morphTargetInfluences[smileIndex] = 0.8;
```

## 3. Demo 1: Face Morph

สร้างใบหน้าอย่างง่ายที่สามารถแสดงอารมณ์ต่างๆ ได้

```html
<!DOCTYPE html>
<html>
<head>
  <title>Face Morph Demo</title>
  <style>
    body { margin: 0; background: #1a1a2e; }
    canvas { display: block; }
    #controls {
      position: fixed;
      top: 20px;
      right: 20px;
      background: rgba(0,0,0,0.7);
      color: white;
      padding: 20px;
      border-radius: 10px;
      font-family: Arial, sans-serif;
    }
    .slider-group {
      margin: 10px 0;
    }
    label {
      display: block;
      margin-bottom: 5px;
      font-size: 14px;
    }
    input[type="range"] {
      width: 200px;
    }
    .value-display {
      display: inline-block;
      width: 40px;
      text-align: right;
    }
  </style>
</head>
<body>
<div id="controls">
  <h3 style="margin-top:0">Face Morph Controls</h3>
  <div class="slider-group">
    <label>😊 Smile: <span class="value-display" id="smileVal">0.00</span></label>
    <input type="range" id="smile" min="0" max="100" value="0">
  </div>
  <div class="slider-group">
    <label>😢 Sad: <span class="value-display" id="sadVal">0.00</span></label>
    <input type="range" id="sad" min="0" max="100" value="0">
  </div>
  <div class="slider-group">
    <label>😠 Angry: <span class="value-display" id="angryVal">0.00</span></label>
    <input type="range" id="angry" min="0" max="100" value="0">
  </div>
  <div class="slider-group">
    <label>😲 Surprised: <span class="value-display" id="surprisedVal">0.00</span></label>
    <input type="range" id="surprised" min="0" max="100" value="0">
  </div>
  <div class="slider-group">
    <label>😉 Wink: <span class="value-display" id="winkVal">0.00</span></label>
    <input type="range" id="wink" min="0" max="100" value="0">
  </div>
  <button id="animateBtn" style="margin-top:10px; padding:8px 16px; cursor:pointer">
    Auto Animate
  </button>
</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';

// ===== SETUP =====
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x1a1a2e);

const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 0, 4);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
renderer.shadowMap.enabled = true;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

// ===== LIGHTING =====
const ambientLight = new THREE.AmbientLight(0xffffff, 0.4);
scene.add(ambientLight);

const keyLight = new THREE.DirectionalLight(0xffffff, 1.5);
keyLight.position.set(2, 3, 5);
keyLight.castShadow = true;
scene.add(keyLight);

const fillLight = new THREE.DirectionalLight(0x88aaff, 0.5);
fillLight.position.set(-3, 1, 2);
scene.add(fillLight);

// ===== CREATE FACE GEOMETRY =====
function createFaceGeometry() {
  // ใช้ SphereGeometry เป็น base (จำลองหัว)
  const geometry = new THREE.SphereGeometry(1.2, 64, 64);
  
  // ทำให้เป็นรูปหัวคนโดยปรับ y-scale
  const positions = geometry.attributes.position;
  for (let i = 0; i < positions.count; i++) {
    const x = positions.getX(i);
    const y = positions.getY(i);
    const z = positions.getZ(i);
    
    // บีบด้านล่างเล็กน้อยเพื่อให้เป็นรูปหัว
    if (y < 0) {
      positions.setY(i, y * 0.85);
    }
    // นูนด้านหน้าสำหรับหน้าผาก
    if (y > 0.5 && z > 0) {
      positions.setZ(i, z * 1.05);
    }
  }
  positions.needsUpdate = true;
  geometry.computeVertexNormals();
  
  return geometry;
}

// ===== CREATE MORPH TARGET: SMILE =====
function createSmileMorph(geometry) {
  const positions = geometry.attributes.position;
  const morphPos = new Float32Array(positions.array.length);
  
  for (let i = 0; i < positions.count; i++) {
    const x = positions.getX(i);
    const y = positions.getY(i);
    const z = positions.getZ(i);
    
    let dx = 0, dy = 0, dz = 0;
    
    // ยกมุมปากขึ้น (พื้นที่รอบๆ ปาก)
    const mouthDist = Math.sqrt(x * x + (y + 0.3) * (y + 0.3));
    if (mouthDist < 0.6 && z > 0.8) {
      // ยิ้ม: มุมปากยกขึ้น
      const angle = Math.atan2(x, y + 0.3);
      const intensity = Math.max(0, 0.6 - mouthDist) / 0.6;
      dy = Math.abs(Math.sin(angle)) * intensity * 0.15;
      dz = intensity * 0.05;
      
      // ยกแก้มขึ้น
      if (Math.abs(x) > 0.3 && y > -0.5 && y < 0) {
        dy += intensity * 0.1;
        dx = (x > 0 ? 1 : -1) * intensity * 0.05;
      }
    }
    
    morphPos[i * 3] = x + dx;
    morphPos[i * 3 + 1] = y + dy;
    morphPos[i * 3 + 2] = z + dz;
  }
  
  return new THREE.Float32BufferAttribute(morphPos, 3);
}

// ===== CREATE MORPH TARGET: SAD =====
function createSadMorph(geometry) {
  const positions = geometry.attributes.position;
  const morphPos = new Float32Array(positions.array.length);
  
  for (let i = 0; i < positions.count; i++) {
    const x = positions.getX(i);
    const y = positions.getY(i);
    const z = positions.getZ(i);
    
    let dx = 0, dy = 0, dz = 0;
    
    // ดึงมุมปากลง
    const mouthDist = Math.sqrt(x * x + (y + 0.3) * (y + 0.3));
    if (mouthDist < 0.6 && z > 0.8) {
      const angle = Math.atan2(x, y + 0.3);
      const intensity = Math.max(0, 0.6 - mouthDist) / 0.6;
      dy = -Math.abs(Math.sin(angle)) * intensity * 0.2;
    }
    
    // ดึงคิ้วลง (เศร้า)
    if (y > 0.4 && y < 0.8 && Math.abs(x) > 0.2 && z > 0.5) {
      const browIntensity = Math.max(0, 0.2 - Math.abs(y - 0.6)) / 0.2;
      // คิ้วกลางยกขึ้น ขอบนอกลง (สีหน้าเศร้า)
      dy += (Math.abs(x) > 0.5 ? -0.1 : 0.05) * browIntensity;
    }
    
    morphPos[i * 3] = x + dx;
    morphPos[i * 3 + 1] = y + dy;
    morphPos[i * 3 + 2] = z + dz;
  }
  
  return new THREE.Float32BufferAttribute(morphPos, 3);
}

// ===== CREATE MORPH TARGET: ANGRY =====
function createAngryMorph(geometry) {
  const positions = geometry.attributes.position;
  const morphPos = new Float32Array(positions.array.length);
  
  for (let i = 0; i < positions.count; i++) {
    const x = positions.getX(i);
    const y = positions.getY(i);
    const z = positions.getZ(i);
    
    let dx = 0, dy = 0, dz = 0;
    
    // คิ้วขมวด (ด้านกลางลง ด้านนอกขึ้น)
    if (y > 0.35 && y < 0.75 && Math.abs(x) < 0.7 && z > 0.6) {
      const browY = (y - 0.35) / 0.4;
      const browX = Math.abs(x) / 0.7;
      const intensity = browY * (1 - browY) * 4;
      
      // กดด้านในลง ยกด้านนอกขึ้น
      dy += (browX > 0.4 ? 0.05 : -0.1) * intensity;
      // ขมวดเข้าหากัน
      dx += (x > 0 ? -0.05 : 0.05) * intensity;
    }
    
    // ปากกัดฟัน (แน่น)
    const mouthDist = Math.sqrt(x * x + (y + 0.3) * (y + 0.3));
    if (mouthDist < 0.4 && z > 0.9) {
      const intensity = Math.max(0, 0.4 - mouthDist) / 0.4;
      dz += intensity * 0.03;
      dy -= intensity * 0.05;
    }
    
    morphPos[i * 3] = x + dx;
    morphPos[i * 3 + 1] = y + dy;
    morphPos[i * 3 + 2] = z + dz;
  }
  
  return new THREE.Float32BufferAttribute(morphPos, 3);
}

// ===== CREATE MORPH TARGET: SURPRISED =====
function createSurprisedMorph(geometry) {
  const positions = geometry.attributes.position;
  const morphPos = new Float32Array(positions.array.length);
  
  for (let i = 0; i < positions.count; i++) {
    const x = positions.getX(i);
    const y = positions.getY(i);
    const z = positions.getZ(i);
    
    let dx = 0, dy = 0, dz = 0;
    
    // เปิดปากกว้าง
    const mouthDist = Math.sqrt(x * x + (y + 0.35) * (y + 0.35));
    if (mouthDist < 0.35 && z > 0.8) {
      const intensity = Math.max(0, 0.35 - mouthDist) / 0.35;
      // บีบปากเป็นวงกลม
      dx += x * intensity * 0.2;
      dy += (y + 0.35) * intensity * 0.3;
      dz += intensity * 0.05;
    }
    
    // ยกคิ้วขึ้น (ตกใจ)
    if (y > 0.45 && y < 0.85 && Math.abs(x) < 0.6 && z > 0.55) {
      const intensity = Math.max(0, 0.2 - Math.abs(y - 0.65)) / 0.2;
      dy += intensity * 0.15;
    }
    
    morphPos[i * 3] = x + dx;
    morphPos[i * 3 + 1] = y + dy;
    morphPos[i * 3 + 2] = z + dz;
  }
  
  return new THREE.Float32BufferAttribute(morphPos, 3);
}

// ===== CREATE MORPH TARGET: WINK =====
function createWinkMorph(geometry) {
  const positions = geometry.attributes.position;
  const morphPos = new Float32Array(positions.array.length);
  
  for (let i = 0; i < positions.count; i++) {
    const x = positions.getX(i);
    const y = positions.getY(i);
    const z = positions.getZ(i);
    
    let dx = 0, dy = 0, dz = 0;
    
    // หรี่ตาขวา (ด้านซ้ายของ mesh = ตาขวาตัวละคร)
    const rightEyeDist = Math.sqrt((x - 0.35) * (x - 0.35) + (y - 0.25) * (y - 0.25));
    if (rightEyeDist < 0.25 && z > 0.8) {
      const intensity = Math.max(0, 0.25 - rightEyeDist) / 0.25;
      // กดเปลือกตาบนลง
      if (y > 0.25) {
        dy -= intensity * 0.12;
      }
    }
    
    // ยิ้มเล็กน้อย
    const mouthDist = Math.sqrt(x * x + (y + 0.3) * (y + 0.3));
    if (mouthDist < 0.5 && z > 0.85) {
      const intensity = Math.max(0, 0.5 - mouthDist) / 0.5;
      const angle = Math.atan2(x, y + 0.3);
      dy += Math.abs(Math.sin(angle)) * intensity * 0.08;
    }
    
    morphPos[i * 3] = x + dx;
    morphPos[i * 3 + 1] = y + dy;
    morphPos[i * 3 + 2] = z + dz;
  }
  
  return new THREE.Float32BufferAttribute(morphPos, 3);
}

// ===== BUILD THE FACE =====
const faceGeometry = createFaceGeometry();

// เพิ่ม morph targets ทั้งหมด
faceGeometry.morphAttributes.position = [
  createSmileMorph(faceGeometry),
  createSadMorph(faceGeometry),
  createAngryMorph(faceGeometry),
  createSurprisedMorph(faceGeometry),
  createWinkMorph(faceGeometry)
];

const faceMaterial = new THREE.MeshStandardMaterial({
  color: 0xffcba4,  // สีผิว
  roughness: 0.7,
  metalness: 0.0,
  morphTargets: true,
  morphNormals: true
});

const faceMesh = new THREE.Mesh(faceGeometry, faceMaterial);
faceMesh.castShadow = true;
faceMesh.receiveShadow = true;
scene.add(faceMesh);

// morphTargetDictionary สำหรับเข้าถึงด้วยชื่อ
faceMesh.morphTargetDictionary = {
  smile: 0,
  sad: 1,
  angry: 2,
  surprised: 3,
  wink: 4
};

// ===== CONTROLS =====
const sliders = {
  smile: document.getElementById('smile'),
  sad: document.getElementById('sad'),
  angry: document.getElementById('angry'),
  surprised: document.getElementById('surprised'),
  wink: document.getElementById('wink')
};

const displays = {
  smile: document.getElementById('smileVal'),
  sad: document.getElementById('sadVal'),
  angry: document.getElementById('angryVal'),
  surprised: document.getElementById('surprisedVal'),
  wink: document.getElementById('winkVal')
};

function updateMorphFromSlider(name, value) {
  const val = value / 100;
  const index = faceMesh.morphTargetDictionary[name];
  faceMesh.morphTargetInfluences[index] = val;
  displays[name].textContent = val.toFixed(2);
}

Object.entries(sliders).forEach(([name, slider]) => {
  slider.addEventListener('input', (e) => {
    updateMorphFromSlider(name, parseInt(e.target.value));
  });
});

// ===== AUTO ANIMATION =====
let autoAnimate = false;
let animTime = 0;

document.getElementById('animateBtn').addEventListener('click', () => {
  autoAnimate = !autoAnimate;
  document.getElementById('animateBtn').textContent = 
    autoAnimate ? 'Stop Animation' : 'Auto Animate';
});

function updateAutoAnimation(delta) {
  animTime += delta;
  
  // Sequence ของ expressions
  const cycle = animTime % 10;
  
  // Reset all
  Object.keys(faceMesh.morphTargetDictionary).forEach(key => {
    const idx = faceMesh.morphTargetDictionary[key];
    faceMesh.morphTargetInfluences[idx] = 0;
  });
  
  let expr, intensity;
  
  if (cycle < 2) {
    expr = 'smile';
    intensity = Math.min(1, cycle);
  } else if (cycle < 4) {
    expr = 'sad';
    intensity = Math.min(1, cycle - 2);
  } else if (cycle < 6) {
    expr = 'angry';
    intensity = Math.min(1, cycle - 4);
  } else if (cycle < 8) {
    expr = 'surprised';
    intensity = Math.min(1, cycle - 6);
  } else {
    expr = 'wink';
    intensity = Math.min(1, cycle - 8);
  }
  
  const idx = faceMesh.morphTargetDictionary[expr];
  faceMesh.morphTargetInfluences[idx] = Math.sin(intensity * Math.PI / 2);
  
  // อัพเดต sliders
  Object.keys(faceMesh.morphTargetDictionary).forEach(key => {
    const index = faceMesh.morphTargetDictionary[key];
    const val = faceMesh.morphTargetInfluences[index];
    sliders[key].value = Math.round(val * 100);
    displays[key].textContent = val.toFixed(2);
  });
}

// ===== ANIMATION LOOP =====
const clock = new THREE.Clock();

function animate() {
  requestAnimationFrame(animate);
  
  const delta = clock.getDelta();
  
  if (autoAnimate) {
    updateAutoAnimation(delta);
  }
  
  // หมุนใบหน้าเล็กน้อย
  faceMesh.rotation.y = Math.sin(clock.elapsedTime * 0.3) * 0.3;
  
  controls.update();
  renderer.render(scene, camera);
}

animate();

// ===== RESIZE =====
window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});
</script>
</body>
</html>
```

## 4. AnimationClip กับ Morph Targets

Three.js รองรับการสร้าง Animation จาก morph targets โดยใช้ `AnimationClip` ร่วมกับ `NumberKeyframeTrack`

```javascript
import * as THREE from 'three';

// สร้าง AnimationClip สำหรับ morph targets
function createMorphAnimation(mesh) {
  const tracks = [];
  
  // Track สำหรับ smile morph
  const smileTrack = new THREE.NumberKeyframeTrack(
    '.morphTargetInfluences[0]', // path: target ชื่อ property
    [0, 1, 2, 3, 4],             // times (seconds)
    [0, 1, 0, 0.5, 0]            // values (influence ที่แต่ละ keyframe)
  );
  tracks.push(smileTrack);
  
  // Track สำหรับ blink morph (สมมติ index 2 = blink)
  const blinkTrack = new THREE.NumberKeyframeTrack(
    '.morphTargetInfluences[2]',
    [0, 0.1, 0.2, 0.3, 0.4, 2.5, 2.6, 2.7, 2.8, 4],
    [0,   1,   0,   0,   0, 0,   1,   0,   0,   0]
  );
  tracks.push(blinkTrack);
  
  // สร้าง AnimationClip
  const clip = new THREE.AnimationClip('FacialAnimation', 4, tracks);
  
  // สร้าง AnimationMixer
  const mixer = new THREE.AnimationMixer(mesh);
  const action = mixer.clipAction(clip);
  action.play();
  
  return mixer;
}

// ใช้ใน animation loop
const mixer = createMorphAnimation(faceMesh);
const clock = new THREE.Clock();

function animate() {
  requestAnimationFrame(animate);
  
  const delta = clock.getDelta();
  mixer.update(delta);
  
  renderer.render(scene, camera);
}

animate();
```

## 5. Demo 2: Shape Shifting Object

สร้าง object ที่ morph ระหว่างรูปทรงเรขาคณิตต่างๆ

```html
<!DOCTYPE html>
<html>
<head>
  <title>Shape Shifting Demo</title>
  <style>
    body { margin: 0; background: #000; overflow: hidden; }
    canvas { display: block; }
    #info {
      position: fixed;
      bottom: 20px;
      left: 50%;
      transform: translateX(-50%);
      color: white;
      font-family: Arial, sans-serif;
      font-size: 16px;
      text-align: center;
      text-shadow: 0 0 10px rgba(100,200,255,0.8);
    }
    #morphProgress {
      position: fixed;
      top: 20px;
      left: 50%;
      transform: translateX(-50%);
      color: white;
      font-family: monospace;
      font-size: 14px;
      background: rgba(0,0,0,0.5);
      padding: 10px 20px;
      border-radius: 5px;
    }
  </style>
</head>
<body>
<div id="morphProgress">Morphing: Sphere → ...</div>
<div id="info">กด SPACE เพื่อหยุด/เล่น | Click เพื่อ morph ถัดไป</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';

// ===== SETUP =====
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x050510);
scene.fog = new THREE.FogExp2(0x050510, 0.05);

const camera = new THREE.PerspectiveCamera(70, innerWidth / innerHeight, 0.1, 100);
camera.position.set(0, 2, 6);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(devicePixelRatio);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.dampingFactor = 0.05;

// ===== LIGHTING =====
scene.add(new THREE.AmbientLight(0x222244, 1));

const pointLight1 = new THREE.PointLight(0x4488ff, 3, 20);
pointLight1.position.set(3, 3, 3);
scene.add(pointLight1);

const pointLight2 = new THREE.PointLight(0xff4488, 3, 20);
pointLight2.position.set(-3, -1, 2);
scene.add(pointLight2);

const pointLight3 = new THREE.PointLight(0x44ff88, 2, 15);
pointLight3.position.set(0, -2, -3);
scene.add(pointLight3);

// ===== ฟังก์ชันสร้าง Geometry ที่มี vertex count เท่ากัน =====
// (สำคัญมาก: morph targets ต้องมี vertex count เท่ากันกับ base geometry)

const SEGMENT_COUNT = 32; // ใช้ resolution เดียวกันทุก geometry

function createUnifiedGeometry() {
  // ใช้ sphere เป็น base (มี vertices จำนวน segment^2 * 2)
  return new THREE.SphereGeometry(1.5, SEGMENT_COUNT, SEGMENT_COUNT);
}

// ฟังก์ชัน sample positions จาก geometry อื่น ที่มี vertex count ต่างกัน
// โดยใช้ UV mapping หรือ parametric approach
function sampleSpherePositions(geometry, targetFunc) {
  const positions = geometry.attributes.position;
  const morphPos = new Float32Array(positions.array.length);
  
  for (let i = 0; i < positions.count; i++) {
    // ดึง normal direction (unit sphere)
    const nx = positions.getX(i);
    const ny = positions.getY(i);
    const nz = positions.getZ(i);
    const len = Math.sqrt(nx*nx + ny*ny + nz*nz);
    const ux = nx/len, uy = ny/len, uz = nz/len;
    
    // map บน parametric surface ใหม่
    const theta = Math.atan2(ux, uz); // longitude
    const phi = Math.acos(Math.max(-1, Math.min(1, uy))); // latitude
    
    const [tx, ty, tz] = targetFunc(theta, phi, ux, uy, uz);
    
    morphPos[i * 3] = tx;
    morphPos[i * 3 + 1] = ty;
    morphPos[i * 3 + 2] = tz;
  }
  
  return new THREE.Float32BufferAttribute(morphPos, 3);
}

// ===== SHAPE FUNCTIONS =====

// Cube shape
function cubeShape(theta, phi, nx, ny, nz) {
  const ax = Math.abs(nx), ay = Math.abs(ny), az = Math.abs(nz);
  const max = Math.max(ax, ay, az);
  const s = 1.2 / max;
  return [nx * s, ny * s, nz * s];
}

// Torus shape (approximation)
function torusShape(theta, phi, nx, ny, nz) {
  const R = 1.2, r = 0.5;
  const u = theta;
  const v = phi * 2 - Math.PI;
  const x = (R + r * Math.cos(v)) * Math.cos(u);
  const y = (R + r * Math.cos(v)) * Math.sin(u);
  const z = r * Math.sin(v);
  return [x * 0.6, z * 0.6, y * 0.6];
}

// Star shape
function starShape(theta, phi, nx, ny, nz) {
  const points = 5;
  const outerR = 1.5, innerR = 0.7;
  const angle = theta;
  const sectorAngle = (Math.PI * 2) / points;
  const normalizedAngle = ((angle % sectorAngle) + sectorAngle) % sectorAngle;
  const halfSector = sectorAngle / 2;
  
  let r;
  if (normalizedAngle < halfSector) {
    r = outerR + (innerR - outerR) * (normalizedAngle / halfSector);
  } else {
    r = innerR + (outerR - innerR) * ((normalizedAngle - halfSector) / halfSector);
  }
  
  const flatness = Math.abs(ny) * 0.8;
  r *= (1 - flatness * 0.5);
  
  return [nx * r, ny * outerR * 0.3, nz * r * Math.sign(nz)];
}

// Diamond shape
function diamondShape(theta, phi, nx, ny, nz) {
  const ax = Math.abs(nx), ay = Math.abs(ny), az = Math.abs(nz);
  const l1 = ax + ay + az; // L1 norm
  const s = 1.5 / l1;
  return [nx * s, ny * s * 1.5, nz * s];
}

// Twisted cylinder
function twistedShape(theta, phi, nx, ny, nz) {
  const twist = ny * Math.PI;
  const cosT = Math.cos(twist), sinT = Math.sin(twist);
  
  const hor = Math.sqrt(nx * nx + nz * nz);
  const angle = Math.atan2(nz, nx) + twist * 0.5;
  
  const r = 1.0;
  const x = r * Math.cos(angle);
  const z = r * Math.sin(angle);
  return [x, ny * 1.5, z];
}

// ===== BUILD MORPH MESH =====
const baseGeometry = createUnifiedGeometry();

// สร้าง morph targets
baseGeometry.morphAttributes.position = [
  sampleSpherePositions(baseGeometry, cubeShape),
  sampleSpherePositions(baseGeometry, torusShape),
  sampleSpherePositions(baseGeometry, starShape),
  sampleSpherePositions(baseGeometry, diamondShape),
  sampleSpherePositions(baseGeometry, twistedShape)
];

const shapeNames = ['Sphere', 'Cube', 'Torus', 'Star', 'Diamond', 'Twisted'];

const morphMaterial = new THREE.MeshStandardMaterial({
  color: 0x2244aa,
  roughness: 0.2,
  metalness: 0.8,
  morphTargets: true,
  morphNormals: true,
  wireframe: false,
  envMapIntensity: 1.0
});

const morphMesh = new THREE.Mesh(baseGeometry, morphMaterial);
morphMesh.castShadow = true;
scene.add(morphMesh);

// ===== MORPH ANIMATION SYSTEM =====
let currentShape = 0; // Sphere
let targetShape = 1;  // Cube
let morphProgress = 0;
let isPlaying = true;
let morphSpeed = 0.5;

const progressDisplay = document.getElementById('morphProgress');

function setMorphInfluences(fromIdx, toIdx, t) {
  // Reset all influences
  for (let i = 0; i < 5; i++) {
    morphMesh.morphTargetInfluences[i] = 0;
  }
  
  // Base shape (index 0 = sphere) is implicit - no influence needed when t=0
  // Other shapes are morph targets (index 1-5 = morph[0]-[4])
  
  const ease = t * t * (3 - 2 * t); // smoothstep
  
  if (fromIdx > 0) {
    // ลดจาก from shape
    morphMesh.morphTargetInfluences[fromIdx - 1] = 1 - ease;
  }
  
  if (toIdx > 0) {
    // เพิ่มไปยัง to shape
    morphMesh.morphTargetInfluences[toIdx - 1] = ease;
  }
}

function advanceToNextShape() {
  currentShape = targetShape;
  targetShape = (targetShape + 1) % shapeNames.length;
  morphProgress = 0;
}

// ===== PARTICLE EFFECT =====
const particleCount = 2000;
const particleGeometry = new THREE.BufferGeometry();
const particlePositions = new Float32Array(particleCount * 3);
const particleColors = new Float32Array(particleCount * 3);

for (let i = 0; i < particleCount; i++) {
  particlePositions[i * 3] = (Math.random() - 0.5) * 20;
  particlePositions[i * 3 + 1] = (Math.random() - 0.5) * 20;
  particlePositions[i * 3 + 2] = (Math.random() - 0.5) * 20;
  
  const hue = Math.random();
  const color = new THREE.Color().setHSL(hue, 1, 0.6);
  particleColors[i * 3] = color.r;
  particleColors[i * 3 + 1] = color.g;
  particleColors[i * 3 + 2] = color.b;
}

particleGeometry.setAttribute('position', new THREE.BufferAttribute(particlePositions, 3));
particleGeometry.setAttribute('color', new THREE.BufferAttribute(particleColors, 3));

const particleMaterial = new THREE.PointsMaterial({
  size: 0.05,
  vertexColors: true,
  transparent: true,
  opacity: 0.6,
  blending: THREE.AdditiveBlending,
  depthWrite: false
});

const particles = new THREE.Points(particleGeometry, particleMaterial);
scene.add(particles);

// ===== INPUT HANDLING =====
document.addEventListener('keydown', (e) => {
  if (e.code === 'Space') {
    isPlaying = !isPlaying;
    e.preventDefault();
  }
});

window.addEventListener('click', () => {
  advanceToNextShape();
});

// ===== ANIMATION LOOP =====
const clock = new THREE.Clock();

function animate() {
  requestAnimationFrame(animate);
  
  const delta = clock.getDelta();
  const elapsed = clock.elapsedTime;
  
  if (isPlaying) {
    morphProgress += delta * morphSpeed;
    
    if (morphProgress >= 1) {
      morphProgress = 1;
      setMorphInfluences(currentShape, targetShape, 1);
      
      // รอสักครู่แล้ว transition ไปรูปทรงถัดไป
      setTimeout(advanceToNextShape, 500);
    }
    
    setMorphInfluences(currentShape, targetShape, morphProgress);
  }
  
  // หมุน
  morphMesh.rotation.y = elapsed * 0.3;
  morphMesh.rotation.x = Math.sin(elapsed * 0.2) * 0.1;
  
  // Animate lights
  pointLight1.position.x = Math.cos(elapsed * 0.7) * 4;
  pointLight1.position.z = Math.sin(elapsed * 0.7) * 4;
  pointLight2.position.x = Math.cos(elapsed * 0.5 + 2) * 4;
  pointLight2.position.z = Math.sin(elapsed * 0.5 + 2) * 4;
  
  // Animate particles
  particles.rotation.y = elapsed * 0.05;
  particles.rotation.x = elapsed * 0.03;
  
  // Update display
  const fromName = shapeNames[currentShape];
  const toName = shapeNames[targetShape];
  const pct = Math.round(morphProgress * 100);
  progressDisplay.textContent = `Morphing: ${fromName} → ${toName} (${pct}%)`;
  
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

## 6. Blendshapes Concept ใน Game Development

Blendshapes คือการนำ morph targets หลายๆ ตัวมา blend กันเพื่อสร้างการแสดงออกที่หลากหลาย

```javascript
// ระบบ Blendshape แบบ FACS (Facial Action Coding System)
class BlendshapeController {
  constructor(mesh) {
    this.mesh = mesh;
    this.blendshapes = {};
    this.setupDictionary();
  }
  
  setupDictionary() {
    // FACS Action Units
    this.blendshapes = {
      // Upper face
      AU1_InnerBrowRaise: 0,
      AU2_OuterBrowRaise: 1,
      AU4_BrowLowerer: 2,
      AU5_UpperLidRaiser: 3,
      AU6_CheekRaiser: 4,
      AU7_LidTightener: 5,
      
      // Lower face
      AU10_UpperLipRaiser: 6,
      AU12_LipCornerPuller: 7,  // Smile
      AU15_LipCornerDepressor: 8, // Frown
      AU17_ChinRaiser: 9,
      AU20_LipStretcher: 10,
      AU25_LipsParted: 11,
      AU26_JawDrop: 12,
      
      // Eyes
      AU43_EyesClosed: 13,
      AU46_Wink: 14,
    };
  }
  
  // ตั้งค่า blendshape ด้วยชื่อ
  set(name, value) {
    const index = this.blendshapes[name];
    if (index !== undefined) {
      this.mesh.morphTargetInfluences[index] = Math.max(0, Math.min(1, value));
    }
  }
  
  // Get current value
  get(name) {
    const index = this.blendshapes[name];
    if (index !== undefined) {
      return this.mesh.morphTargetInfluences[index];
    }
    return 0;
  }
  
  // สร้าง expression จากหลาย blendshapes
  setExpression(expressionDef) {
    // Reset ก่อน
    this.resetAll();
    
    // ตั้งค่าตาม definition
    Object.entries(expressionDef).forEach(([name, value]) => {
      this.set(name, value);
    });
  }
  
  // Expression presets
  setHappy() {
    this.setExpression({
      AU6_CheekRaiser: 0.7,
      AU12_LipCornerPuller: 0.9,
      AU25_LipsParted: 0.3
    });
  }
  
  setSad() {
    this.setExpression({
      AU1_InnerBrowRaise: 0.6,
      AU4_BrowLowerer: 0.3,
      AU15_LipCornerDepressor: 0.8,
      AU17_ChinRaiser: 0.4
    });
  }
  
  setSurprised() {
    this.setExpression({
      AU1_InnerBrowRaise: 0.9,
      AU2_OuterBrowRaise: 0.9,
      AU5_UpperLidRaiser: 0.8,
      AU26_JawDrop: 0.7
    });
  }
  
  setAngry() {
    this.setExpression({
      AU4_BrowLowerer: 1.0,
      AU7_LidTightener: 0.7,
      AU10_UpperLipRaiser: 0.5,
      AU20_LipStretcher: 0.4
    });
  }
  
  resetAll() {
    Object.values(this.blendshapes).forEach(index => {
      this.mesh.morphTargetInfluences[index] = 0;
    });
  }
  
  // Smooth transition ระหว่าง expressions
  transitionTo(expressionDef, duration, onComplete) {
    const startValues = {};
    
    // เก็บค่าเริ่มต้น
    Object.keys(this.blendshapes).forEach(name => {
      startValues[name] = this.get(name);
    });
    
    const startTime = performance.now();
    
    const update = () => {
      const elapsed = (performance.now() - startTime) / 1000;
      const t = Math.min(1, elapsed / duration);
      const ease = t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t; // ease in-out
      
      // Interpolate ทุก blendshapes
      Object.keys(this.blendshapes).forEach(name => {
        const from = startValues[name] || 0;
        const to = expressionDef[name] || 0;
        this.set(name, from + (to - from) * ease);
      });
      
      if (t < 1) {
        requestAnimationFrame(update);
      } else {
        if (onComplete) onComplete();
      }
    };
    
    requestAnimationFrame(update);
  }
}

// การใช้งาน BlendshapeController
const controller = new BlendshapeController(characterMesh);

// เปลี่ยน expression แบบ smooth
controller.transitionTo({
  AU12_LipCornerPuller: 1.0,
  AU6_CheekRaiser: 0.8
}, 0.5, () => {
  console.log('Transition complete!');
});
```

## 7. dat.GUI Integration สำหรับ Morph Control

```javascript
import * as dat from 'https://unpkg.com/dat.gui@0.7.9/build/dat.gui.module.js';

function createMorphGUI(mesh) {
  const gui = new dat.GUI({ width: 280 });
  
  // สร้าง control object
  const controls = {};
  
  if (mesh.morphTargetDictionary) {
    // ใช้ dictionary ถ้ามี
    const folder = gui.addFolder('Morph Targets');
    
    Object.entries(mesh.morphTargetDictionary).forEach(([name, index]) => {
      controls[name] = mesh.morphTargetInfluences[index] || 0;
      
      folder.add(controls, name, 0, 1, 0.01)
        .name(name)
        .onChange((value) => {
          mesh.morphTargetInfluences[index] = value;
        });
    });
    
    folder.open();
  } else {
    // ใช้ index ถ้าไม่มี dictionary
    const numTargets = mesh.morphTargetInfluences
      ? mesh.morphTargetInfluences.length
      : 0;
    
    const folder = gui.addFolder(`Morph Targets (${numTargets})`);
    
    for (let i = 0; i < numTargets; i++) {
      controls[`target_${i}`] = 0;
      
      folder.add(controls, `target_${i}`, 0, 1, 0.01)
        .name(`Target ${i}`)
        .onChange((value) => {
          mesh.morphTargetInfluences[i] = value;
        });
    }
    
    folder.open();
  }
  
  // เพิ่ม preset buttons
  const presets = {
    resetAll: () => {
      mesh.morphTargetInfluences.fill(0);
      Object.keys(controls).forEach(key => {
        controls[key] = 0;
      });
      gui.updateDisplay();
    },
    randomize: () => {
      mesh.morphTargetInfluences.forEach((_, i) => {
        const val = Math.random();
        mesh.morphTargetInfluences[i] = val;
        controls[`target_${i}`] = val;
      });
      gui.updateDisplay();
    }
  };
  
  gui.add(presets, 'resetAll').name('Reset All');
  gui.add(presets, 'randomize').name('Randomize');
  
  return gui;
}
```

## 8. createMorphTarget Helper Function

```javascript
// Utility function สำหรับสร้าง morph targets อย่างง่าย
class MorphTargetBuilder {
  constructor(baseGeometry) {
    this.baseGeometry = baseGeometry;
    this.morphTargets = [];
    this.names = {};
  }
  
  // เพิ่ม morph target จาก callback function
  addTarget(name, transformFn) {
    const positions = this.baseGeometry.attributes.position;
    const morphPos = new Float32Array(positions.array.length);
    
    for (let i = 0; i < positions.count; i++) {
      const v = new THREE.Vector3(
        positions.getX(i),
        positions.getY(i),
        positions.getZ(i)
      );
      
      // Apply transform
      const newV = transformFn(v.clone(), i);
      
      morphPos[i * 3] = newV.x;
      morphPos[i * 3 + 1] = newV.y;
      morphPos[i * 3 + 2] = newV.z;
    }
    
    const index = this.morphTargets.length;
    this.morphTargets.push(new THREE.Float32BufferAttribute(morphPos, 3));
    this.names[name] = index;
    
    return this;
  }
  
  // เพิ่ม morph target จาก geometry อื่น (ต้องมี vertex count เท่ากัน)
  addTargetFromGeometry(name, targetGeometry) {
    const sourcePos = this.baseGeometry.attributes.position;
    const targetPos = targetGeometry.attributes.position;
    
    if (sourcePos.count !== targetPos.count) {
      console.error(`Vertex count mismatch: ${sourcePos.count} vs ${targetPos.count}`);
      return this;
    }
    
    const morphPos = new Float32Array(targetPos.array);
    const index = this.morphTargets.length;
    this.morphTargets.push(new THREE.Float32BufferAttribute(morphPos, 3));
    this.names[name] = index;
    
    return this;
  }
  
  // Apply morph targets กลับไปยัง geometry
  build() {
    this.baseGeometry.morphAttributes.position = this.morphTargets;
    return this.baseGeometry;
  }
  
  // สร้าง mesh พร้อม morph support
  createMesh(material) {
    this.build();
    
    const mesh = new THREE.Mesh(this.baseGeometry, material);
    mesh.morphTargetDictionary = { ...this.names };
    mesh.morphTargetInfluences = new Array(this.morphTargets.length).fill(0);
    
    return mesh;
  }
}

// ตัวอย่างการใช้งาน
const builder = new MorphTargetBuilder(new THREE.BoxGeometry(1, 1, 1, 4, 4, 4));

builder
  .addTarget('puff', (v) => {
    // ทำให้นูนออกด้านนอก
    const len = v.length();
    return v.normalize().multiplyScalar(len * 1.3 + 0.2);
  })
  .addTarget('spike', (v) => {
    // สร้าง spike effect
    const len = v.length();
    const spike = Math.pow(len, 0.3) * 2;
    return v.normalize().multiplyScalar(spike);
  })
  .addTarget('flatten', (v) => {
    // แบนราบ
    return new THREE.Vector3(v.x * 1.5, v.y * 0.1, v.z * 1.5);
  });

const puffyBox = builder.createMesh(
  new THREE.MeshStandardMaterial({ color: 0xff6644, morphTargets: true })
);

scene.add(puffyBox);

// ควบคุม
puffyBox.morphTargetInfluences[0] = 0.5; // half puffy
```

## 9. Performance Tips สำหรับ Morph Targets

```javascript
// 1. จำกัดจำนวน morph targets
// Three.js รองรับสูงสุด 8 morph targets ใน standard shader
// ถ้าต้องการมากกว่า ใช้ custom shader

// 2. ใช้ morphNormals เมื่อจำเป็น
const material = new THREE.MeshStandardMaterial({
  morphTargets: true,
  morphNormals: true // เพิ่ม performance overhead แต่ให้ lighting ที่ถูกต้อง
});

// 3. LOD สำหรับ morph targets
class MorphLOD {
  constructor() {
    this.levels = [];
    this.currentLevel = 0;
  }
  
  addLevel(mesh, distance) {
    this.levels.push({ mesh, distance });
    this.levels.sort((a, b) => a.distance - b.distance);
  }
  
  update(camera, position) {
    const dist = camera.position.distanceTo(position);
    
    for (let i = this.levels.length - 1; i >= 0; i--) {
      if (dist >= this.levels[i].distance) {
        if (this.currentLevel !== i) {
          // Hide current, show new
          if (this.levels[this.currentLevel]) {
            this.levels[this.currentLevel].mesh.visible = false;
          }
          this.levels[i].mesh.visible = true;
          
          // Copy morph influences
          if (i < this.levels.length - 1) {
            const fromMesh = this.levels[this.currentLevel].mesh;
            const toMesh = this.levels[i].mesh;
            
            fromMesh.morphTargetInfluences.forEach((val, idx) => {
              if (toMesh.morphTargetInfluences[idx] !== undefined) {
                toMesh.morphTargetInfluences[idx] = val;
              }
            });
          }
          
          this.currentLevel = i;
        }
        break;
      }
    }
  }
}

// 4. Sparse morph targets (เปลี่ยนเฉพาะ vertices ที่จำเป็น)
function createSparseMorphTarget(geometry, affectedVertices, transformFn) {
  const positions = geometry.attributes.position;
  // Copy base positions
  const morphPos = new Float32Array(positions.array);
  
  // แก้ไขเฉพาะ vertices ที่ต้องการ
  affectedVertices.forEach(i => {
    const v = new THREE.Vector3(
      positions.getX(i),
      positions.getY(i),
      positions.getZ(i)
    );
    
    const newV = transformFn(v);
    morphPos[i * 3] = newV.x;
    morphPos[i * 3 + 1] = newV.y;
    morphPos[i * 3 + 2] = newV.z;
  });
  
  return new THREE.Float32BufferAttribute(morphPos, 3);
}
```

## 10. GLTF Morph Targets

```javascript
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

const loader = new GLTFLoader();

loader.load('/models/character.glb', (gltf) => {
  const character = gltf.scene;
  scene.add(character);
  
  // หา mesh ที่มี morph targets
  character.traverse((child) => {
    if (child.isMesh && child.morphTargetInfluences) {
      console.log('Found mesh with morph targets:', child.name);
      console.log('Morph targets:', child.morphTargetDictionary);
      console.log('Influences:', child.morphTargetInfluences);
      
      // ตั้งค่า morph target ด้วยชื่อ
      if (child.morphTargetDictionary['smile'] !== undefined) {
        const smileIdx = child.morphTargetDictionary['smile'];
        child.morphTargetInfluences[smileIdx] = 0.8;
      }
    }
  });
  
  // ถ้ามี animations ที่ใช้ morph targets
  if (gltf.animations.length > 0) {
    const mixer = new THREE.AnimationMixer(character);
    
    gltf.animations.forEach(clip => {
      console.log(`Animation: ${clip.name}, Duration: ${clip.duration}s`);
      
      // ตรวจสอบว่า clip มี morph target tracks หรือไม่
      clip.tracks.forEach(track => {
        if (track.name.includes('morphTargetInfluences')) {
          console.log('  Morph track:', track.name);
        }
      });
      
      const action = mixer.clipAction(clip);
      action.play();
    });
    
    // Update mixer ใน animation loop
    const clock = new THREE.Clock();
    function animate() {
      requestAnimationFrame(animate);
      mixer.update(clock.getDelta());
      renderer.render(scene, camera);
    }
    animate();
  }
});
```

## สรุป

Morph Targets เป็นเทคนิคที่ทรงพลังสำหรับ:
- **Facial Animation** ในการสร้างตัวละคร
- **Smooth Shape Transitions** ระหว่างรูปทรงต่างๆ
- **Blendshapes** สำหรับ lip sync และ expression
- **Procedural Animation** ที่ควบคุมด้วย code

### ข้อดีของ Morph Targets
1. **GPU-efficient** - การคำนวณทำบน GPU
2. **Smooth interpolation** - ได้ผลลัพธ์ที่นุ่มนวล
3. **Multiple simultaneous morphs** - blend หลาย shape พร้อมกัน
4. **Works with AnimationClip** - ผสานกับระบบ animation ของ Three.js

### ข้อจำกัด
1. ต้องการ vertex count เท่ากัน
2. จำกัด 8 targets ใน standard shader
3. Memory usage สูงเมื่อมีหลาย targets

---

## แบบฝึกหัด

1. **Easy**: สร้าง sphere ที่ morph เป็น cube โดยใช้ morphAttributes และควบคุมด้วย slider
2. **Medium**: สร้างระบบ facial expression ที่มี 5 อารมณ์ต่างๆ โดยใช้ BlendshapeController
3. **Hard**: สร้าง morph animation sequence ที่ใช้ AnimationClip เพื่อเล่น facial animation แบบ automatic พร้อม blink และ lip sync
4. **Challenge**: โหลด GLTF character ที่มี morph targets และสร้าง GUI สำหรับควบคุม blendshapes ทั้งหมด พร้อม smooth transition ระหว่าง expression presets
