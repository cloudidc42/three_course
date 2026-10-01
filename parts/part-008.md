# Part 008: Lighting Part 1 - แสงพื้นฐานใน Three.js

## สารบัญบทเรียนนี้
- [1. ทำไมต้องมีแสง](#1-ทำไมต้องมีแสง)
- [2. AmbientLight](#2-ambientlight)
- [3. DirectionalLight](#3-directionallight)
- [4. PointLight](#4-pointlight)
- [5. SpotLight](#5-spotlight)
- [6. HemisphereLight](#6-hemispherelight)
- [7. RectAreaLight](#7-rectarealight)
- [8. Light Helpers](#8-light-helpers)
- [9. Light Performance](#9-light-performance)
- [10. Complete Lighting Demo](#10-complete-lighting-demo)
- [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. ทำไมต้องมีแสง

ใน Three.js materials บางชนิดต้องการแสงจึงจะมองเห็น:

```javascript
// ไม่ต้องการแสง (unlit):
// - MeshBasicMaterial
// - MeshNormalMaterial
// - MeshDepthMaterial
// - MeshMatcapMaterial
// - LineBasicMaterial / LineDashedMaterial
// - PointsMaterial (ถ้าไม่ใช้ PBR)

// ต้องการแสง (lit):
// - MeshLambertMaterial
// - MeshPhongMaterial
// - MeshStandardMaterial    ← PBR
// - MeshPhysicalMaterial    ← PBR advanced
// - MeshToonMaterial

// ถ้าไม่มีแสงในฉาก materials พวกนี้จะเป็นสีดำสนิท
const scene = new THREE.Scene();
// เพิ่ม AmbientLight ขั้นต่ำเสมอ
const ambient = new THREE.AmbientLight(0xffffff, 0.3);
scene.add(ambient);
```

### ประเภทแสงและคุณสมบัติ

| ประเภท | ทิศทาง | เงา | Performance | Use Case |
|--------|--------|-----|-------------|----------|
| AmbientLight | ไม่มี | ไม่ได้ | ดีมาก | fill light ทั่วไป |
| DirectionalLight | ทิศทางเดียว (ขนาน) | ได้ | ดี | แสงอาทิตย์ |
| PointLight | ทุกทิศ (จากจุด) | ได้ (ช้า) | ปานกลาง | หลอดไฟ |
| SpotLight | กรวย | ได้ | ปานกลาง | ไฟสปอตไลต์ |
| HemisphereLight | sky+ground | ไม่ได้ | ดีมาก | outdoor ambient |
| RectAreaLight | สี่เหลี่ยม | ไม่ได้ | ช้า | softbox / หน้าต่าง |

---

## 2. AmbientLight

แสงที่ส่องทุกทิศทางอย่างสม่ำเสมอ ไม่มีทิศทาง ไม่ทำเงา

```javascript
// ========== Basic AmbientLight ==========

const ambientLight = new THREE.AmbientLight(
    0xffffff,   // color
    0.4         // intensity (default 1.0)
);
scene.add(ambientLight);

// เปลี่ยน properties ภายหลัง
ambientLight.color.set('#ffe4b5');   // warm light
ambientLight.intensity = 0.5;

// ========== ใช้ AmbientLight อย่างไร ==========

// ❌ Wrong: ใช้ AmbientLight intensity สูงเดี่ยวๆ
const badAmbient = new THREE.AmbientLight(0xffffff, 1.5);
// ผลลัพธ์: ทุกอย่างดูแบนเหมือน unlit, ไม่มี shading

// ✅ Correct: Ambient น้อยๆ + directional/point lights
const goodAmbient = new THREE.AmbientLight(0xffffff, 0.3);
const mainLight = new THREE.DirectionalLight(0xffffff, 2.0);
scene.add(goodAmbient, mainLight);

// ========== Ambient ตาม Scene ==========

// ในห้อง (indoor): warm light
const indoorAmbient = new THREE.AmbientLight(0xfff0dd, 0.5);

// กลางแจ้งตอนกลางวัน: cool blue sky light
const outdoorAmbient = new THREE.AmbientLight(0xd0e8ff, 0.3);

// กลางคืน: dark blue ambient
const nightAmbient = new THREE.AmbientLight(0x0a0a1a, 0.2);

// ========== Ambient ร่วมกับ Shadow ==========

// AmbientLight ทำให้ส่วนที่เป็นเงาไม่มืดสนิท
// ควรใช้ intensity เบาๆ เพื่อให้เงามีรายละเอียด
// intensity สูงเกินไป = เงาหายไป, scene ดูแบน
```

---

## 3. DirectionalLight

แสงที่ส่องขนานกัน เหมาะสำหรับแสงอาทิตย์

```javascript
// ========== Basic DirectionalLight ==========

const dirLight = new THREE.DirectionalLight(
    0xffffff,   // color
    2.0         // intensity
);

// Position: ทิศทางแสงคือ position → target
// default target คือ (0,0,0)
dirLight.position.set(5, 10, 7);
scene.add(dirLight);

// ========== Light Target ==========

// Target กำหนดทิศทางแสง
const target = new THREE.Object3D();
target.position.set(0, 0, 0);
scene.add(target);

dirLight.target = target;
// แสงจะส่องจาก position → target

// target ต้อง add เข้า scene ด้วย!
// ไม่เช่นนั้น world matrix จะไม่ update

// ========== Directional Light Shadows ==========

dirLight.castShadow = true;

// Shadow camera คือ OrthographicCamera
// (เพราะแสง directional เป็นขนาน)
dirLight.shadow.camera.near = 0.1;
dirLight.shadow.camera.far = 100;

// Shadow frustum (ต้องครอบคลุม objects ทั้งหมด)
const shadowCamSize = 20;
dirLight.shadow.camera.left = -shadowCamSize;
dirLight.shadow.camera.right = shadowCamSize;
dirLight.shadow.camera.top = shadowCamSize;
dirLight.shadow.camera.bottom = -shadowCamSize;

// Shadow map resolution
dirLight.shadow.mapSize.width = 2048;
dirLight.shadow.mapSize.height = 2048;

// Shadow quality
dirLight.shadow.bias = -0.005;      // ลด shadow acne
dirLight.shadow.normalBias = 0.02;  // สำหรับ curved surfaces
dirLight.shadow.radius = 4;         // soft shadow (PCF)

// ========== เปิดใช้ shadow ใน renderer ==========

renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;  // default PCFShadowMap

// Objects ต้อง set castShadow / receiveShadow
mesh.castShadow = true;
mesh.receiveShadow = true;
ground.receiveShadow = true;

// ========== Shadow Map Types ==========

// THREE.BasicShadowMap      - เร็ว, aliased edges
// THREE.PCFShadowMap        - smooth edges (ค่า default)
// THREE.PCFSoftShadowMap    - softer edges (ใช้บ่อยที่สุด)
// THREE.VSMShadowMap        - Variance Shadow Map (soft, มี artifacts)

// ========== Helper: Shadow Camera Helper ==========

import { CameraHelper } from 'three';
const shadowCamHelper = new THREE.CameraHelper(dirLight.shadow.camera);
scene.add(shadowCamHelper);
// แสดง frustum ของ shadow camera (debug เท่านั้น)
// ลบออกเมื่อ production
```

---

## 4. PointLight

แสงที่ส่องออกทุกทิศทางจากจุดเดียว เหมือนหลอดไฟ

```javascript
// ========== Basic PointLight ==========

const pointLight = new THREE.PointLight(
    0xff6633,   // color (สีส้ม)
    5.0,        // intensity
    20,         // distance (0 = ไม่จำกัด)
    2           // decay (2 = physically correct)
);
pointLight.position.set(0, 3, 0);
scene.add(pointLight);

// ========== Distance และ Decay ==========

// distance:
//   0 = ไม่มี cutoff (ส่องไปไกลตลอด)
//   > 0 = ค่อยๆ หายเมื่อถึงระยะ distance

// decay:
//   0 = ไม่มี attenuation (intensity คงที่)
//   1 = linear falloff
//   2 = physically correct (inverse square law)
//      ← ควรใช้เสมอสำหรับ realism

// เมื่อใช้ decay = 2, intensity ต้องสูงขึ้นมาก
// เพราะ intensity = lumens (physical unit)

const physicalLight = new THREE.PointLight(0xffffff, 200, 0, 2);
// intensity 200 lumens, ไม่มี distance limit, physically correct decay

// ========== PointLight Shadows ==========

pointLight.castShadow = true;
pointLight.shadow.camera.near = 0.1;
pointLight.shadow.camera.far = 20;  // ต้องน้อยกว่า light distance

// Shadow map resolution
pointLight.shadow.mapSize.set(1024, 1024);
// ⚠️ PointLight ใช้ cube shadow map (6 faces)
// ดังนั้น 1024² × 6 = หนักกว่า DirectionalLight มาก

pointLight.shadow.bias = -0.001;

// ========== หลาย PointLights ==========

// ⚠️ Shadow casting PointLight ช้ามาก
// หากมีหลายดวง ลดจำนวน shadow casters

const lights = [];
const positions = [
    [-3, 2, -3], [3, 2, -3], [-3, 2, 3], [3, 2, 3]
];
const colors = [0xff4444, 0x4444ff, 0x44ff44, 0xffff44];

positions.forEach((pos, i) => {
    const light = new THREE.PointLight(colors[i], 3, 8, 2);
    light.position.set(...pos);
    // ไม่ castShadow เพื่อ performance
    scene.add(light);
    lights.push(light);
});

// ========== Animated PointLight (เช่น เทียน) ==========

const clock = new THREE.Clock();

function animateCandle(light, deltaTime) {
    const elapsed = clock.getElapsedTime();
    
    // Flickering
    const flicker = 1.0 
        + Math.sin(elapsed * 17.3) * 0.08
        + Math.sin(elapsed * 31.7) * 0.05
        + (Math.random() - 0.5) * 0.03;
    
    light.intensity = 3.0 * flicker;
    
    // Color variation (เหลือง-ส้ม)
    const warm = Math.sin(elapsed * 7.1) * 0.1 + 0.9;
    light.color.setRGB(1.0, warm * 0.4 + 0.2, 0.05);
    
    // Slight position jitter
    light.position.x = light.userData.baseX + (Math.random() - 0.5) * 0.02;
    light.position.y = light.userData.baseY + (Math.random() - 0.5) * 0.02;
}
```

---

## 5. SpotLight

แสงรูปกรวย เหมือนไฟสปอตไลต์หรือไฟฉาย

```javascript
// ========== Basic SpotLight ==========

const spotLight = new THREE.SpotLight(
    0xffffff,   // color
    10,         // intensity
    30,         // distance
    Math.PI/6,  // angle (30 degrees = กรวยแคบ)
    0.3,        // penumbra (0=คม, 1=นุ่ม)
    2           // decay (2 = physical)
);

spotLight.position.set(0, 10, 0);
spotLight.target.position.set(0, 0, 0);  // ชี้ลงพื้น

scene.add(spotLight);
scene.add(spotLight.target);  // ⚠️ ต้อง add target เข้า scene

// ========== Angle และ Penumbra ==========

// angle:
//   กรวยรัศมีในหน่วย radians
//   ค่าสูงสุด = Math.PI/2 (90 degree cone)
//   Math.PI/6 = 30° (narrow spot)
//   Math.PI/4 = 45° (medium)
//   Math.PI/3 = 60° (wide)

// penumbra:
//   0 = edge คมชัดสนิท
//   1 = soft edge เต็มที่
//   0.2-0.5 = ค่าทั่วไป

// ========== SpotLight Shadows ==========

spotLight.castShadow = true;
spotLight.shadow.camera.near = 0.5;
spotLight.shadow.camera.far = 30;  // ควรเท่ากับ distance

// FOV ของ shadow camera ควร = angle ของ spotlight * 2
// Three.js คำนวณอัตโนมัติจาก spot angle
spotLight.shadow.camera.fov = spotLight.angle * 2 * (180 / Math.PI);

spotLight.shadow.mapSize.set(1024, 1024);
spotLight.shadow.bias = -0.001;
spotLight.shadow.radius = 2;

// ========== Moving SpotLight ==========

// ไฟ follow mouse / path
const clock = new THREE.Clock();

function animateSpotlight() {
    const elapsed = clock.getElapsedTime();
    const radius = 5;
    
    // วนรอบ target
    spotLight.position.x = Math.sin(elapsed * 0.5) * radius;
    spotLight.position.z = Math.cos(elapsed * 0.5) * radius;
    spotLight.position.y = 8;
    
    // target คงที่ที่ origin
    // Three.js จะ update direction อัตโนมัติ
}

// ========== SpotLight ชี้ตาม Object ==========

function lookAtObject(spotLight, targetObject) {
    // copy world position ของ object ไปที่ target
    targetObject.getWorldPosition(spotLight.target.position);
}
```

---

## 6. HemisphereLight

แสงที่ mix ระหว่าง sky color (บน) และ ground color (ล่าง)
เหมาะสำหรับ outdoor scene เพราะจำลอง sky/ground ambient

```javascript
// ========== Basic HemisphereLight ==========

const hemiLight = new THREE.HemisphereLight(
    0x87ceeb,   // skyColor (ฟ้าสว่าง)
    0x4a3728,   // groundColor (น้ำตาลดิน)
    0.8         // intensity
);
hemiLight.position.set(0, 20, 0);  // position ไม่สำคัญมาก แค่ใช้กำหนด "ขึ้น"
scene.add(hemiLight);

// ========== ตัวอย่าง: Outdoor Daytime ==========

// Sky
const daySkyColor = new THREE.Color(0x87ceeb);   // light blue
const dayGroundColor = new THREE.Color(0x556b2f); // dark olive (grass)
const dayHemi = new THREE.HemisphereLight(daySkyColor, dayGroundColor, 0.6);

// Sun
const sunLight = new THREE.DirectionalLight(0xfff4e0, 3.0);
sunLight.position.set(50, 100, -50); // high angle

// ========== ตัวอย่าง: Indoor ==========

// Ceiling (warm)
const ceilColor = new THREE.Color(0xffe4b5);    // moccasin
// Floor (dark)
const floorColor = new THREE.Color(0x1a1a1a);   // nearly black
const indoorHemi = new THREE.HemisphereLight(ceilColor, floorColor, 0.4);

// ========== ตัวอย่าง: Sunset ==========

const sunsetHemi = new THREE.HemisphereLight(
    0xff8c00,   // orange sky
    0x4a3728,   // dark ground
    1.0
);

// ========== HemisphereLight ไม่ทำ shadow ==========
// ใช้ร่วมกับ DirectionalLight เสมอเพื่อ shadow
// Hemi = ambient base
// Dir = directional shading + shadow
```

---

## 7. RectAreaLight

แสงสี่เหลี่ยม เหมือน softbox หรือ LED panel
**ใช้ได้กับ MeshStandardMaterial และ MeshPhysicalMaterial เท่านั้น**

```javascript
import { RectAreaLightHelper } from 'three/addons/helpers/RectAreaLightHelper.js';
import { RectAreaLightUniformsLib } from 'three/addons/lights/RectAreaLightUniformsLib.js';

// ⚠️ ต้อง init uniform library ก่อนใช้
RectAreaLightUniformsLib.init();

// ========== Basic RectAreaLight ==========

const rectLight = new THREE.RectAreaLight(
    0xffffff,   // color
    10,         // intensity (lumens/m²)
    4,          // width (meters)
    3           // height (meters)
);

// Position และ look at target
rectLight.position.set(-5, 3, 0);
rectLight.lookAt(0, 0, 0);  // ชี้ไปที่ center

scene.add(rectLight);

// ========== Helper ==========

const rectLightHelper = new RectAreaLightHelper(rectLight);
rectLight.add(rectLightHelper);  // add เป็น child ของ light

// ========== RectAreaLight Use Cases ==========

// 1. Window light
const windowLight = new THREE.RectAreaLight(0xd0e8ff, 8, 2, 3);
windowLight.position.set(-5, 2, 0);
windowLight.rotation.y = Math.PI / 2;  // หน้าชี้เข้าห้อง

// 2. Monitor glow
const monitorLight = new THREE.RectAreaLight(0x4477ff, 5, 1.6, 0.9);
monitorLight.position.set(0, 1.5, 0);
monitorLight.lookAt(0, 1.5, -2); // ชี้ไปทาง viewer

// 3. Neon sign
const neonLight = new THREE.RectAreaLight(0xff0080, 15, 3, 0.2);
neonLight.position.set(0, 4, -5);

// ========== Note: ข้อจำกัด ==========
// - ไม่รองรับ shadow
// - ช้ากว่า light ชนิดอื่น
// - ใช้ได้กับ MeshStandard/PhysicalMaterial เท่านั้น
// - ต้อง RectAreaLightUniformsLib.init() ก่อนเสมอ
```

---

## 8. Light Helpers

Helpers ช่วย visualize ตำแหน่งและทิศทางของแสง

```javascript
// ========== DirectionalLightHelper ==========

const dirHelper = new THREE.DirectionalLightHelper(
    dirLight,  // light
    2          // size (กล่องสี่เหลี่ยมที่แสดง)
);
scene.add(dirHelper);
// แสดงกล่องที่ตำแหน่ง light และเส้นชี้ไปยัง target
dirHelper.update();  // เรียกเมื่อ light เคลื่อนที่

// ========== PointLightHelper ==========

const pointHelper = new THREE.PointLightHelper(
    pointLight,  // light
    0.5          // sphere size
);
scene.add(pointHelper);

// ========== SpotLightHelper ==========

const spotHelper = new THREE.SpotLightHelper(spotLight);
scene.add(spotHelper);
// แสดงกรวยของ spotlight
spotHelper.update();  // เรียกเมื่อ light/target เปลี่ยน

// ========== HemisphereLightHelper ==========

const hemiHelper = new THREE.HemisphereLightHelper(hemiLight, 2);
scene.add(hemiHelper);

// ========== CameraHelper สำหรับ Shadow ==========

const shadowHelper = new THREE.CameraHelper(dirLight.shadow.camera);
scene.add(shadowHelper);
// แสดง frustum ของ shadow camera

// ========== ลบ Helper ตอน Production ==========

const helpers = [];  // เก็บ reference ทั้งหมด

// เพิ่ม helper
function addHelper(helper) {
    scene.add(helper);
    helpers.push(helper);
}

// ลบทั้งหมด
function removeAllHelpers() {
    helpers.forEach(h => {
        scene.remove(h);
        h.dispose?.();
    });
    helpers.length = 0;
}

// Toggle helpers ด้วย keyboard
document.addEventListener('keydown', (e) => {
    if (e.key === 'h') {
        helpers.forEach(h => {
            h.visible = !h.visible;
        });
    }
});

// ========== Update helpers ใน animate loop ==========

function animate() {
    requestAnimationFrame(animate);
    
    // Update helpers ที่ต้องการ
    dirHelper.update();
    spotHelper.update();
    shadowHelper.update();
    
    renderer.render(scene, camera);
}
```

---

## 9. Light Performance

แสงมีผลต่อ performance มาก โดยเฉพาะ shadow

```javascript
// ========== Performance Tips ==========

// 1. จำกัดจำนวน shadow casting lights
// ⚠️ แต่ละ shadow light = extra render pass
// DirectionalLight shadow = 1 extra pass (orthographic)
// SpotLight shadow = 1 extra pass
// PointLight shadow = 6 extra passes (cube map) ← แพงมาก

// ✅ ใช้ shadow casting light น้อยๆ (1-2 ดวง)
// ✅ non-shadow lights สามารถมีได้มากกว่า

// 2. Shadow map size
// ยิ่งใหญ่ = คม แต่ช้าและใช้ VRAM มาก
// 512 = เล็ก (mobile/background objects)
// 1024 = standard
// 2048 = high quality
// 4096 = very high (desktop only)

dirLight.shadow.mapSize.set(1024, 1024);

// 3. Shadow camera frustum ให้เล็กที่สุด
// ยิ่งเล็ก = shadow texel ยิ่งคม (ไม่เปลืองขนาด)
dirLight.shadow.camera.left = -10;  // พอดีกับ scene
dirLight.shadow.camera.right = 10;
dirLight.shadow.camera.top = 10;
dirLight.shadow.camera.bottom = -10;

// 4. Light layers (render ต่างกัน)
// บาง lights อาจไม่จำเป็นต้องกระทบทุก mesh
// ใช้ layers เพื่อ cull

pointLight.layers.set(1);  // layer 1
mesh1.layers.enable(1);    // mesh1 รับแสง layer 1 เท่านั้น
mesh2.layers.disable(1);   // mesh2 ไม่รับแสง layer 1

// 5. castShadow เฉพาะที่จำเป็น
scene.traverse((node) => {
    if (node.isMesh) {
        // เฉพาะ objects ที่มองเห็นในฉากสำคัญ
        node.castShadow = true;
        node.receiveShadow = true;
        
        // Small/background objects: ปิด shadow
        // if (node.userData.smallObject) {
        //     node.castShadow = false;
        // }
    }
});

// 6. Light baking สำหรับ static scene
// แทนที่จะ real-time, bake lighting เข้า lightmap/AO
// ประหยัด performance มาก แต่ต้องทำใน Blender/offline

// ========== Physically Correct Mode ==========

// Three.js r155+ ใช้ physically correct units เสมอ
// intensity = lumens (PointLight/SpotLight)
// intensity = lux (DirectionalLight/AmbientLight)
// decay = 2 (inverse square law)

// typical values:
// ดวงอาทิตย์: DirectionalLight intensity ~100000 lux
// หลอดไฟ 100W: PointLight ~1600 lumens
// เทียน: PointLight ~12 lumens

// ปรับ tone mapping ให้รองรับ HDR values:
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 0.001; // ลด exposure สำหรับ physical units
```

---

## 10. Complete Lighting Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Part 008: Lighting Part 1</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        html, body { width: 100%; height: 100%; overflow: hidden; background: #0d1117; }
        #info {
            position: fixed; top: 10px; left: 10px;
            color: white; font-family: monospace; font-size: 12px;
            background: rgba(0,0,0,0.5); padding: 10px; border-radius: 6px;
        }
    </style>
</head>
<body>
    <div id="info">
        Lighting Demo<br>
        GUI: เลือก light type<br>
        Drag: orbit camera
    </div>
    
    <script type="importmap">
    {
        "imports": {
            "three": "https://cdn.jsdelivr.net/npm/three@0.165.0/build/three.module.js",
            "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.165.0/examples/jsm/"
        }
    }
    </script>
    
    <script type="module">
        import * as THREE from 'three';
        import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
        import { GUI } from 'three/addons/libs/lil-gui.module.min.js';
        import { RectAreaLightHelper } from 'three/addons/helpers/RectAreaLightHelper.js';
        import { RectAreaLightUniformsLib } from 'three/addons/lights/RectAreaLightUniformsLib.js';
        
        // Init RectAreaLight
        RectAreaLightUniformsLib.init();
        
        // ====== RENDERER ======
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(innerWidth, innerHeight);
        renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
        renderer.shadowMap.enabled = true;
        renderer.shadowMap.type = THREE.PCFSoftShadowMap;
        renderer.toneMapping = THREE.ACESFilmicToneMapping;
        renderer.toneMappingExposure = 1.0;
        document.body.appendChild(renderer.domElement);
        
        // ====== SCENE ======
        const scene = new THREE.Scene();
        scene.background = new THREE.Color('#0a0a14');
        scene.fog = new THREE.Fog('#0a0a14', 20, 60);
        
        // ====== CAMERA ======
        const camera = new THREE.PerspectiveCamera(60, innerWidth/innerHeight, 0.1, 100);
        camera.position.set(8, 6, 8);
        
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.target.set(0, 1, 0);
        controls.update();
        
        // ====== GEOMETRY SETUP ======
        
        // Ground
        const groundGeo = new THREE.PlaneGeometry(20, 20, 20, 20);
        const groundMat = new THREE.MeshStandardMaterial({
            color: '#2a2a3a',
            roughness: 0.8,
            metalness: 0.1,
        });
        const ground = new THREE.Mesh(groundGeo, groundMat);
        ground.rotation.x = -Math.PI / 2;
        ground.receiveShadow = true;
        scene.add(ground);
        
        // Create objects array
        const objects = [];
        const positions = [
            [-4, 0, -4], [-4, 0, 0], [-4, 0, 4],
            [ 0, 0, -4], [ 0, 0, 0], [ 0, 0, 4],
            [ 4, 0, -4], [ 4, 0, 0], [ 4, 0, 4],
        ];
        
        const shapes = [
            new THREE.BoxGeometry(1.2, 1.2, 1.2),
            new THREE.SphereGeometry(0.7, 32, 16),
            new THREE.ConeGeometry(0.7, 1.5, 16),
            new THREE.CylinderGeometry(0.5, 0.5, 1.5, 16),
            new THREE.TorusGeometry(0.6, 0.25, 16, 32),
            new THREE.IcosahedronGeometry(0.7, 1),
            new THREE.OctahedronGeometry(0.8, 0),
            new THREE.TetrahedronGeometry(0.9, 0),
            new THREE.TorusKnotGeometry(0.45, 0.18, 100, 16),
        ];
        
        positions.forEach((pos, i) => {
            const mat = new THREE.MeshStandardMaterial({
                color: new THREE.Color().setHSL(i / 9, 0.7, 0.5),
                roughness: 0.3 + i * 0.07,
                metalness: i * 0.1,
            });
            const mesh = new THREE.Mesh(shapes[i], mat);
            mesh.position.set(pos[0], 0.8, pos[2]);
            mesh.castShadow = true;
            mesh.receiveShadow = true;
            scene.add(mesh);
            objects.push(mesh);
        });
        
        // ====== LIGHTS ======
        
        // Storage for all lights and helpers
        const lightsGroup = {
            ambient: null,
            directional: null,
            point: null,
            spot: null,
            hemisphere: null,
            rectArea: null,
        };
        
        const helpersGroup = {};
        
        function clearLights() {
            Object.values(lightsGroup).forEach(light => {
                if (light) {
                    scene.remove(light);
                    if (light.target) scene.remove(light.target);
                }
            });
            Object.values(helpersGroup).forEach(h => {
                if (h) scene.remove(h);
            });
        }
        
        // ====== PRESET SETUPS ======
        
        const presets = {
            
            'Ambient Only': () => {
                const light = new THREE.AmbientLight(0xffffff, 1.5);
                scene.add(light);
                lightsGroup.ambient = light;
            },
            
            'Directional': () => {
                const amb = new THREE.AmbientLight(0xffffff, 0.3);
                const dir = new THREE.DirectionalLight(0xffffff, 3.0);
                dir.position.set(5, 10, 5);
                dir.castShadow = true;
                dir.shadow.mapSize.set(2048, 2048);
                dir.shadow.camera.near = 0.5;
                dir.shadow.camera.far = 50;
                const s = 12;
                dir.shadow.camera.left = -s; dir.shadow.camera.right = s;
                dir.shadow.camera.top = s; dir.shadow.camera.bottom = -s;
                dir.shadow.bias = -0.002;
                dir.shadow.radius = 4;
                
                scene.add(amb, dir);
                lightsGroup.ambient = amb;
                lightsGroup.directional = dir;
                
                const helper = new THREE.DirectionalLightHelper(dir, 1);
                scene.add(helper);
                helpersGroup.dir = helper;
                
                const shadowHelper = new THREE.CameraHelper(dir.shadow.camera);
                scene.add(shadowHelper);
                helpersGroup.shadow = shadowHelper;
            },
            
            'Point Lights': () => {
                const amb = new THREE.AmbientLight(0xffffff, 0.1);
                scene.add(amb);
                lightsGroup.ambient = amb;
                
                const colors = [0xff4444, 0x44ff44, 0x4444ff, 0xffff44, 0xff44ff];
                const ptPositions = [
                    [-5, 4, -5], [5, 4, -5], [-5, 4, 5], [5, 4, 5], [0, 6, 0]
                ];
                
                ptPositions.forEach((pos, i) => {
                    const light = new THREE.PointLight(colors[i], 8, 10, 2);
                    light.position.set(...pos);
                    light.castShadow = i === 0; // เฉพาะ 1 ดวงที่ทำ shadow
                    if (i === 0) {
                        light.shadow.mapSize.set(1024, 1024);
                        light.shadow.camera.far = 10;
                    }
                    scene.add(light);
                    
                    const helper = new THREE.PointLightHelper(light, 0.3);
                    scene.add(helper);
                });
            },
            
            'Spot Light': () => {
                const amb = new THREE.AmbientLight(0xffffff, 0.15);
                const spot = new THREE.SpotLight(0xffffff, 50, 30, Math.PI/6, 0.4, 2);
                spot.position.set(0, 12, 0);
                spot.target.position.set(0, 0, 0);
                spot.castShadow = true;
                spot.shadow.mapSize.set(2048, 2048);
                spot.shadow.bias = -0.001;
                spot.shadow.radius = 3;
                
                scene.add(amb, spot, spot.target);
                lightsGroup.ambient = amb;
                lightsGroup.spot = spot;
                
                const helper = new THREE.SpotLightHelper(spot);
                scene.add(helper);
                helpersGroup.spot = helper;
            },
            
            'Hemisphere + Dir': () => {
                const hemi = new THREE.HemisphereLight(0x87ceeb, 0x4a3728, 0.8);
                const dir = new THREE.DirectionalLight(0xfff4e0, 2.5);
                dir.position.set(10, 15, 10);
                dir.castShadow = true;
                dir.shadow.mapSize.set(2048, 2048);
                dir.shadow.bias = -0.001;
                dir.shadow.radius = 3;
                const s = 12;
                dir.shadow.camera.left = -s; dir.shadow.camera.right = s;
                dir.shadow.camera.top = s; dir.shadow.camera.bottom = -s;
                
                scene.add(hemi, dir);
                lightsGroup.hemisphere = hemi;
                lightsGroup.directional = dir;
                
                const hHelper = new THREE.HemisphereLightHelper(hemi, 2);
                const dHelper = new THREE.DirectionalLightHelper(dir, 1.5);
                scene.add(hHelper, dHelper);
            },
            
            'Rect Area Lights': () => {
                const amb = new THREE.AmbientLight(0xffffff, 0.1);
                scene.add(amb);
                lightsGroup.ambient = amb;
                
                const rectConfigs = [
                    { color: 0xff3333, pos: [-6, 4, 0], lookAt: [0, 2, 0] },
                    { color: 0x3333ff, pos: [6, 4, 0],  lookAt: [0, 2, 0] },
                    { color: 0x33ff33, pos: [0, 6, -6], lookAt: [0, 2, 0] },
                ];
                
                rectConfigs.forEach(cfg => {
                    const light = new THREE.RectAreaLight(cfg.color, 12, 3, 3);
                    light.position.set(...cfg.pos);
                    light.lookAt(...cfg.lookAt);
                    scene.add(light);
                    
                    const helper = new RectAreaLightHelper(light);
                    light.add(helper);
                });
            },
        };
        
        // ====== GUI ======
        const gui = new GUI({ title: 'Lighting' });
        
        const params = {
            preset: 'Hemisphere + Dir',
            showHelpers: true,
            rotateScene: false,
        };
        
        function applyPreset(name) {
            clearLights();
            presets[name]?.();
        }
        
        gui.add(params, 'preset', Object.keys(presets)).onChange(applyPreset);
        gui.add(params, 'showHelpers').onChange(v => {
            Object.values(helpersGroup).forEach(h => {
                if (h) h.visible = v;
            });
        });
        gui.add(params, 'rotateScene');
        
        // Apply default preset
        applyPreset(params.preset);
        
        // ====== ANIMATION ======
        const clock = new THREE.Clock();
        
        function animate() {
            requestAnimationFrame(animate);
            const delta = clock.getDelta();
            const elapsed = clock.getElapsedTime();
            
            // Rotate objects slightly
            objects.forEach((obj, i) => {
                obj.rotation.y += delta * (0.3 + i * 0.05);
                if (params.rotateScene) {
                    obj.rotation.x += delta * 0.2;
                }
            });
            
            // Move spotlight if active
            if (lightsGroup.spot) {
                const angle = elapsed * 0.5;
                lightsGroup.spot.position.x = Math.sin(angle) * 5;
                lightsGroup.spot.position.z = Math.cos(angle) * 5;
                if (helpersGroup.spot) helpersGroup.spot.update();
            }
            
            // Update helpers
            Object.values(helpersGroup).forEach(h => h?.update?.());
            
            controls.update();
            renderer.render(scene, camera);
        }
        
        animate();
        
        // ====== RESIZE ======
        window.addEventListener('resize', () => {
            camera.aspect = innerWidth / innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(innerWidth, innerHeight);
            renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
        });
    </script>
</body>
</html>
```

---

## สรุปแสงทุกชนิด

| ชนิด | สร้างด้วย | Shadow | ใช้กับ Material |
|------|----------|--------|----------------|
| AmbientLight | `new THREE.AmbientLight(color, intensity)` | ✗ | ทุกอย่าง |
| DirectionalLight | `new THREE.DirectionalLight(color, intensity)` | ✓ | ทุกอย่าง |
| PointLight | `new THREE.PointLight(color, intensity, distance, decay)` | ✓ (6x) | ทุกอย่าง |
| SpotLight | `new THREE.SpotLight(color, intensity, distance, angle, penumbra, decay)` | ✓ | ทุกอย่าง |
| HemisphereLight | `new THREE.HemisphereLight(skyColor, groundColor, intensity)` | ✗ | ทุกอย่าง |
| RectAreaLight | `new THREE.RectAreaLight(color, intensity, width, height)` | ✗ | Standard/Physical |

---

## แบบฝึกหัด (Exercises)

### Exercise 1: Three Point Lighting (ง่าย)
สร้าง classic three-point lighting setup:
- Key light: DirectionalLight สว่าง ด้านหน้าซ้าย
- Fill light: DirectionalLight dim ด้านหน้าขวา
- Rim/Back light: DirectionalLight ด้านหลัง

### Exercise 2: Candle Simulation (ปานกลาง)
สร้าง candle flame ด้วย:
- PointLight สีเหลือง-ส้ม
- Flicker animation (random + sine wave)
- Color temperature variation
- CylinderGeometry เป็นเทียน + ConeGeometry เป็น flame

### Exercise 3: Neon Sign (ปานกลาง)
สร้าง neon sign effect ด้วย:
- RectAreaLight หลายดวง (สีต่างกัน)
- EmissiveMap บน sign geometry
- สีส้ม แดง ม่วง หรือ cyan

### Exercise 4: Day/Night Cycle (ยาก)
Animate transition ระหว่าง day และ night:
- DirectionalLight: position animate รอบ scene (sunrise → sunset)
- color: warm ตอนเช้า-เย็น, cool ตอนกลางวัน
- intensity: เปลี่ยนตามมุม
- AmbientLight: เปลี่ยนสีตาม time of day
- Background color: เปลี่ยนตาม time

---

**ก่อนหน้า:** [Part 007 - Textures Part 2](part-007.md)  
**ต่อไป:** [Part 009 - Lighting Part 2 Advanced](part-009.md)
