# Part 007: Textures Part 2 - PBR Texture Maps

## สารบัญบทเรียนนี้
- [1. ทบทวน PBR และ Texture Maps](#1-ทบทวน-pbr-และ-texture-maps)
- [2. Color/Albedo Map](#2-coloralbedo-map)
- [3. Normal Map ลึกขึ้น](#3-normal-map-ลึกขึ้น)
- [4. Roughness Map](#4-roughness-map)
- [5. Metalness Map](#5-metalness-map)
- [6. AO Map (Ambient Occlusion)](#6-ao-map-ambient-occlusion)
- [7. Displacement/Height Map](#7-displacementheight-map)
- [8. Emissive Map](#8-emissive-map)
- [9. Alpha Map](#9-alpha-map)
- [10. Environment Map สำหรับ PBR](#10-environment-map-สำหรับ-pbr)
- [11. Matcap Texture](#11-matcap-texture)
- [12. สร้าง Procedural Texture ด้วย Canvas API](#12-สร้าง-procedural-texture-ด้วย-canvas-api)
- [13. Complete PBR Demo](#13-complete-pbr-demo)
- [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. ทบทวน PBR และ Texture Maps

ใน Part 005 เราเรียน PBR materials ด้วยค่าคงที่  
ใน Part นี้เราจะใช้ **texture maps** เพื่อกำหนด properties เหล่านี้ **ต่อ pixel**

```
PBR Material = Constant Value × Texture Map

ตัวอย่าง:
roughness = 0.7 (constant) × roughnessMap (per-pixel)
metalness = 1.0 (constant) × metalnessMap (per-pixel)
color = WHITE × colorMap (per-pixel)
```

### Map ทั้งหมดใน MeshStandardMaterial

```javascript
const material = new THREE.MeshStandardMaterial({
    // ====== COLOR ======
    color: '#ffffff',          // base color multiplier
    map: colorTexture,         // albedo/color/diffuse map (RGB)
    
    // ====== PBR CORE ======
    metalness: 1.0,            // metalness multiplier (0-1)
    metalnessMap: metalTex,    // metalness per-pixel (R channel)
    roughness: 1.0,            // roughness multiplier (0-1)  
    roughnessMap: roughTex,    // roughness per-pixel (G channel)
    
    // ====== SURFACE DETAIL ======
    normalMap: normalTex,      // normal map (RGB = xyz normal direction)
    normalScale: new THREE.Vector2(1, 1),
    bumpMap: bumpTex,          // height map → normals (alternative)
    bumpScale: 0.1,
    displacementMap: dispTex,  // actual geometry displacement (grayscale)
    displacementScale: 1.0,
    displacementBias: 0,
    
    // ====== OCCLUSION ======
    aoMap: aoTexture,          // ambient occlusion (grayscale)
    aoMapIntensity: 1.0,
    
    // ====== EMISSIVE ======
    emissive: '#000000',       // emissive color multiplier
    emissiveIntensity: 1.0,
    emissiveMap: emissiveTex,  // emissive map (RGB)
    
    // ====== TRANSPARENCY ======
    alphaMap: alphaTex,        // alpha/opacity map (grayscale)
    transparent: true,         // ต้องเปิด transparent
    
    // ====== ENVIRONMENT ======
    envMap: envTexture,        // environment/reflection map
    envMapIntensity: 1.0,
});
```

---

## 2. Color/Albedo Map

Color map คือสีพื้นฐานของ surface ไม่รวม lighting/shadow

```javascript
const loader = new THREE.TextureLoader();
const colorMap = loader.load('/textures/color.jpg');

// ⚠️ Color map ต้องใช้ SRGBColorSpace!
colorMap.colorSpace = THREE.SRGBColorSpace;

// ตั้งค่า wrapping
colorMap.wrapS = colorMap.wrapT = THREE.RepeatWrapping;
colorMap.repeat.set(2, 2);

// anisotropy สำหรับ quality ดีขึ้น
colorMap.anisotropy = renderer.capabilities.getMaxAnisotropy();

const material = new THREE.MeshStandardMaterial({
    map: colorMap,
    // color ที่กำหนดไว้จะ "คูณ" กับ map
    // white (0xffffff) = map ตรงๆ
    // สีอื่น = tint map
    color: '#ffffff',  // ปกติใช้ white
});

// ========== Color Tinting ==========

// ทำให้ texture ออกสีแดงมากขึ้น
const redTintMat = new THREE.MeshStandardMaterial({
    map: colorMap,
    color: '#ff8888',  // tint แดง
});

// ========== ประเภทของ Albedo Map ==========

// 1. Raw Color (ดิบ ไม่มีแสงเงา) - ถูกต้องสำหรับ PBR
// 2. Baked Lighting (มีแสงเงา bake ไว้แล้ว) - ใช้กับ BasicMaterial
// 3. Painted (วาดด้วยมือ) - สำหรับ stylized games

// สำหรับ PBR ควรใช้ raw color เท่านั้น
// แสงเงาควรคำนวณ real-time
```

---

## 3. Normal Map ลึกขึ้น

Normal map เก็บ normal vector direction ในรูปสี RGB  
ทำให้ surface ดูมีรายละเอียดโดยไม่เพิ่ม polygon จริงๆ

```javascript
// ========== Normal Map Encoding ==========

// Tangent Space Normal Map (ที่ใช้ทั่วไป):
// R channel = X (ซ้าย-ขวา)
// G channel = Y (บน-ล่าง)  
// B channel = Z (depth/height)

// สี "ฟ้าอมม่วง" (128,128,255 = 0.5, 0.5, 1.0) = flat surface
// สีอื่น = surface ที่เอียงไปทาง normal direction นั้น

// Object Space Normal Map (ใช้น้อยกว่า):
// Normal เทียบกับ world space ไม่ใช่ tangent space
// มีข้อเสีย: ถ้า mesh หมุน normal map จะผิดทิศ

const normalMap = loader.load('/textures/normal.jpg');
// ไม่ต้องตั้ง colorSpace (default NoColorSpace)

const material = new THREE.MeshStandardMaterial({
    normalMap: normalMap,
    normalMapType: THREE.TangentSpaceNormalMap,  // default
    normalScale: new THREE.Vector2(1, 1),
    // normalScale คือ multiplier สำหรับ X และ Y
    // (1, 1) = full strength
    // (0.5, 0.5) = half
    // (2, 2) = exaggerated
    // (-1, 1) = flip X (ถ้า normal map ดูกลับทิศแนวนอน)
    // (1, -1) = flip Y (ถ้า normal map ดูกลับทิศแนวตั้ง)
});

// ========== Bump Map vs Normal Map ==========

// Bump Map:
// - เป็น grayscale (ขาว = สูง, ดำ = ต่ำ)
// - Three.js คำนวณ normal จาก gradient ของ bump
// - ง่ายกว่าในการสร้าง (แค่ grayscale)
// - คุณภาพต่ำกว่า normal map

const bumpMap = loader.load('/textures/bump.jpg');
const bumpMaterial = new THREE.MeshStandardMaterial({
    bumpMap: bumpMap,
    bumpScale: 0.05,  // ความแรง (หน่วย: world units)
});

// ========== สร้าง Normal Map จาก Height Map ==========

function generateNormalMapFromHeightMap(heightMapData, width, height) {
    const normalData = new Uint8Array(width * height * 4);
    
    for (let y = 0; y < height; y++) {
        for (let x = 0; x < width; x++) {
            // อ่านค่า height รอบๆ pixel
            const getHeight = (px, py) => {
                const cx = Math.max(0, Math.min(width-1, px));
                const cy = Math.max(0, Math.min(height-1, py));
                return heightMapData[(cy * width + cx) * 4] / 255;
            };
            
            const left  = getHeight(x-1, y);
            const right = getHeight(x+1, y);
            const up    = getHeight(x, y-1);
            const down  = getHeight(x, y+1);
            
            // คำนวณ normal จาก cross product
            const strength = 5; // ยิ่งมาก = relief ยิ่งชัด
            const nx = (left - right) * strength;
            const ny = (up - down) * strength;
            const nz = 1.0;
            
            // Normalize
            const len = Math.sqrt(nx*nx + ny*ny + nz*nz);
            
            const idx = (y * width + x) * 4;
            normalData[idx+0] = Math.floor((nx/len * 0.5 + 0.5) * 255);
            normalData[idx+1] = Math.floor((ny/len * 0.5 + 0.5) * 255);
            normalData[idx+2] = Math.floor((nz/len * 0.5 + 0.5) * 255);
            normalData[idx+3] = 255;
        }
    }
    
    return normalData;
}
```

---

## 4. Roughness Map

กำหนด roughness ต่อ pixel สำหรับรายละเอียดที่ซับซ้อน

```javascript
// Roughness Map เป็น grayscale:
// ขาว (1.0) = rough (matte)
// ดำ (0.0) = smooth (shiny)

const roughnessMap = loader.load('/textures/roughness.jpg');

const material = new THREE.MeshStandardMaterial({
    roughness: 1.0,           // multiplier (ควรใช้ 1.0 เพื่อให้ map ทำงานเต็มที่)
    roughnessMap: roughnessMap,
    // ค่าจริงต่อ pixel = roughness * roughnessMap.green
    // (ใช้ G channel)
});

// ========== ตัวอย่างการใช้งาน ==========

// Worn metal: มีบางส่วน shiny บางส่วน rough
// roughnessMap แสดงว่าส่วนไหน wear มากกว่า
const wornMetal = new THREE.MeshStandardMaterial({
    map: metalColorMap,
    roughness: 1.0,
    roughnessMap: roughnessMap,
    metalness: 1.0,
    metalnessMap: metalnessMap,
});

// ========== Combined Roughness + Metalness Map ==========
// บ่อยครั้ง PBR texture packs จะรวมไว้ใน texture เดียว:
// R channel = อาจเป็น ambient occlusion
// G channel = roughness
// B channel = metalness

const ormMap = loader.load('/textures/orm.jpg');
// "ORM" = Occlusion, Roughness, Metalness

const ormMaterial = new THREE.MeshStandardMaterial({
    aoMap: ormMap,          // R channel
    roughnessMap: ormMap,   // G channel
    metalnessMap: ormMap,   // B channel
    roughness: 1.0,
    metalness: 1.0,
    aoMapIntensity: 1.0,
});
```

---

## 5. Metalness Map

กำหนด metalness ต่อ pixel (สำหรับ surfaces ที่มีส่วนผสมโลหะและไม่ใช่โลหะ)

```javascript
// Metalness Map เป็น grayscale:
// ขาว (1.0) = metal
// ดำ (0.0) = non-metal (dielectric)

// Use case: surface ที่มีทั้งโลหะและไม่ใช่โลหะ
// เช่น: handle ไม้กับหัวโลหะ, ผ้าที่มีปุ่มโลหะ

const metalnessMap = loader.load('/textures/metalness.jpg');

const material = new THREE.MeshStandardMaterial({
    metalness: 1.0,           // multiplier
    metalnessMap: metalnessMap,
    // ค่าจริงต่อ pixel = metalness * metalnessMap.blue
    // (ใช้ B channel)
});

// ========== ตัวอย่าง: กุญแจ ==========
// ส่วนที่เป็นโลหะ = metalness=1
// ส่วนที่เป็นพลาสติก = metalness=0

const keyMaterial = new THREE.MeshStandardMaterial({
    map: keyColorMap,
    metalnessMap: keyMetalnessMap,
    roughnessMap: keyRoughnessMap,
    metalness: 1.0,
    roughness: 1.0,
});
```

---

## 6. AO Map (Ambient Occlusion)

AO map เพิ่มความลึกและน้ำหนักให้กับ scene

```javascript
// AO Map เป็น grayscale:
// ขาว (1.0) = แสงเต็ม (ไม่ถูก occlude)
// ดำ (0.0) = ถูกบังแสงสมบูรณ์

// AO Map ต้องใช้ UV channel ที่สอง!
// (uv2 หรือ uv1 ใน Three.js r149+)

const aoMap = loader.load('/textures/ao.jpg');

// เตรียม UV2
geometry.setAttribute('uv1', geometry.attributes.uv.clone());

const material = new THREE.MeshStandardMaterial({
    aoMap: aoMap,
    aoMapIntensity: 1.0,  // 0=ไม่มีผล, 1=full, 2=exaggerated
});

// ========== สำหรับ model ที่โหลดมา ==========

// GLTFLoader จะ import UV2 มาให้เป็น attribute 'uv1' อัตโนมัติ
// แต่ต้องตั้ง aoMap ใน material

import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
const gltfLoader = new GLTFLoader();
gltfLoader.load('/models/scene.glb', (gltf) => {
    gltf.scene.traverse((node) => {
        if (node.isMesh) {
            // กำหนด AO map ให้ทุก mesh
            if (node.material) {
                node.material.aoMap = aoMap;
                node.material.aoMapIntensity = 0.8;
            }
        }
    });
    scene.add(gltf.scene);
});

// ========== Baking AO ใน Blender ==========
// 1. Select mesh
// 2. Add UV unwrap (2nd UV channel)
// 3. Add Image Texture node → ใส่ new image
// 4. Render → Bake → Ambient Occlusion
// 5. Save เป็น PNG
// 6. Use as aoMap ใน Three.js
```

---

## 7. Displacement/Height Map

Displacement map เขยิบ vertices จริงๆ (ต่างจาก normal map ที่แค่หลอกแสง)

```javascript
// Displacement Map เป็น grayscale:
// ขาว = ดันขึ้น
// ดำ = ดันลง (หรือไม่เขยิบขึ้นอยู่กับ bias)

// ⚠️ Displacement Map ต้องการ segments มากพอ
// เพราะมันเขยิบ vertices ที่มีอยู่ (ไม่สร้างใหม่)

const dispMap = loader.load('/textures/displacement.jpg');

// Sphere ที่มี segments มาก
const sphereGeo = new THREE.SphereGeometry(1, 128, 64);

const material = new THREE.MeshStandardMaterial({
    displacementMap: dispMap,
    displacementScale: 0.3,   // ความสูงสูงสุด (world units)
    displacementBias: -0.1,   // offset (ลบค่าออกจาก displacement)
    // ค่า displacement จริง = (map_value * scale) + bias
    
    // ควรมี normalMap ด้วยเพื่อให้แสงดูถูกต้อง
    normalMap: normalMap,
    normalScale: new THREE.Vector2(1, 1),
    
    roughness: 0.8,
    metalness: 0,
});

const sphere = new THREE.Mesh(sphereGeo, material);
scene.add(sphere);

// ========== ตัวอย่าง: Terrain ==========

const terrainGeo = new THREE.PlaneGeometry(20, 20, 256, 256);
const terrainDispMap = loader.load('/textures/heightmap.jpg');

const terrainMat = new THREE.MeshStandardMaterial({
    map: terrainColorMap,
    displacementMap: terrainDispMap,
    displacementScale: 5.0,    // สูง 5 หน่วย (เมตร)
    displacementBias: -2.5,    // center ที่ 0 (ไม่ใช่ 2.5)
    normalMap: terrainNormalMap,
    roughness: 0.9,
    metalness: 0,
});

const terrain = new THREE.Mesh(terrainGeo, terrainMat);
terrain.rotation.x = -Math.PI / 2;
scene.add(terrain);

// ========== Bump vs Normal vs Displacement ==========

// Bump Map:
// ✅ ง่ายสร้าง (grayscale)
// ✅ performance ดี
// ❌ คุณภาพต่ำสุด

// Normal Map:
// ✅ คุณภาพดีมาก
// ✅ performance ดี
// ❌ silhouette ยังเรียบอยู่

// Displacement Map:
// ✅ เปลี่ยน geometry จริงๆ
// ✅ silhouette ถูกต้อง
// ❌ ต้องการ vertex count มาก
// ❌ performance ต่ำที่สุด

// ใช้ทั้ง Normal + Displacement ร่วมกันได้:
// Normal: ให้รายละเอียดเล็ก (micro detail)
// Displacement: ให้รูปร่างหลัก (macro shape)
```

---

## 8. Emissive Map

กำหนดพื้นที่ที่ส่องแสงออกมาเอง

```javascript
// Emissive Map = RGB texture
// ส่วนที่สีขาว = ส่องแสงเต็มที่
// ส่วนที่ดำ = ไม่ส่องแสง

const emissiveMap = loader.load('/textures/emissive.jpg');
emissiveMap.colorSpace = THREE.SRGBColorSpace; // ⚠️ ต้อง SRGBColorSpace

const material = new THREE.MeshStandardMaterial({
    map: colorMap,
    emissiveMap: emissiveMap,
    emissive: '#ffffff',      // emissive color base (คูณกับ emissiveMap)
    emissiveIntensity: 1.0,   // strength
});

// ========== ตัวอย่าง: TV Screen ==========

const tvMaterial = new THREE.MeshStandardMaterial({
    color: '#333333',           // สีหน้าจอ TV
    roughness: 0.8,
    metalness: 0,
    
    emissiveMap: screenContent, // content บนหน้าจอ
    emissive: '#ffffff',
    emissiveIntensity: 2.0,     // > 1 = HDR (ต้องการ tone mapping)
});

// ========== ตัวอย่าง: City at Night ==========

const cityMaterial = new THREE.MeshStandardMaterial({
    map: cityDayMap,            // สีตอนกลางวัน
    roughness: 0.9,
    metalness: 0,
    
    emissiveMap: cityLightsMap, // ไฟตึกและถนน
    emissive: '#ff8844',        // สีอบอุ่นของไฟ
    emissiveIntensity: 3.0,     // สว่างมากสำหรับ bloom effect
});

// ========== Animation: Flicker ==========

let elapsed = 0;
function animate(delta) {
    elapsed += delta;
    
    // flicker effect
    const flicker = 0.8 + Math.sin(elapsed * 47.3) * 0.1 + 
                         Math.sin(elapsed * 23.1) * 0.05 +
                         Math.random() * 0.05;
    material.emissiveIntensity = flicker;
}
```

---

## 9. Alpha Map

กำหนดความโปร่งแสงต่อ pixel

```javascript
// Alpha Map เป็น grayscale:
// ขาว = ทึบสมบูรณ์ (opacity=1)
// ดำ = โปร่งใสสมบูรณ์ (opacity=0)

// ⚠️ ต้องตั้ง transparent: true ใน material

const alphaMap = loader.load('/textures/alpha.jpg');

const material = new THREE.MeshStandardMaterial({
    map: colorMap,
    alphaMap: alphaMap,
    transparent: true,
    
    // Options for transparent rendering:
    depthWrite: false,   // ป้องกัน z-fighting กับ transparent objects อื่น
    // หรือ:
    alphaTest: 0.5,      // hard cutout (เร็วกว่า แต่ edge ไม่เรียบ)
    alphaToCoverage: true, // MSAA-based soft edges (เรียบขึ้น)
});

// ========== Use Cases ==========

// 1. Leaf / Tree
const leafMat = new THREE.MeshStandardMaterial({
    map: leafColorMap,
    alphaMap: leafAlphaMap,
    alphaTest: 0.5,        // cutout, เร็วและเหมาะสำหรับ vegetation
    side: THREE.DoubleSide,
    transparent: false,     // alphaTest ไม่ต้องการ transparent
});

// 2. Chain Link Fence
const fenceMat = new THREE.MeshStandardMaterial({
    map: fenceMap,
    alphaMap: fenceAlphaMap,
    alphaTest: 0.5,
    side: THREE.DoubleSide,
});

// 3. Smoke / Particle
const smokeMat = new THREE.MeshBasicMaterial({
    map: smokeMap,
    alphaMap: smokeAlphaMap,
    transparent: true,
    opacity: 0.5,
    depthWrite: false,
    blending: THREE.NormalBlending,
    side: THREE.DoubleSide,
});

// 4. Circular Sprite
const spriteMat = new THREE.MeshBasicMaterial({
    map: starMap,
    transparent: true,
    depthWrite: false,
    blending: THREE.AdditiveBlending, // สำหรับ particle/star effect
});
```

---

## 10. Environment Map สำหรับ PBR

Environment map ให้ ambient lighting และ reflection ที่ถูกต้องสำหรับ PBR

```javascript
import { RGBELoader } from 'three/addons/loaders/RGBELoader.js';

// ========== Load HDR Environment ==========

const rgbeLoader = new RGBELoader();
const pmremGenerator = new THREE.PMREMGenerator(renderer);
pmremGenerator.compileEquirectangularShader();

async function loadEnvironment(hdrPath) {
    return new Promise((resolve, reject) => {
        rgbeLoader.load(hdrPath, (hdrTexture) => {
            const envMap = pmremGenerator.fromEquirectangular(hdrTexture).texture;
            hdrTexture.dispose();
            pmremGenerator.dispose();
            resolve(envMap);
        }, undefined, reject);
    });
}

// ใช้งาน
const envMap = await loadEnvironment('/textures/hdr/studio.hdr');

// Set environment
scene.environment = envMap;  // ใช้เป็น lighting สำหรับทุก PBR material
scene.background = envMap;   // ใช้เป็น background ด้วย

// หรือกำหนดให้ material โดยตรง
const material = new THREE.MeshStandardMaterial({
    metalness: 1.0,
    roughness: 0.0,
    envMap: envMap,
    envMapIntensity: 2.0,
});

// ========== Background vs Environment ==========

// scene.background = envMap → แสดง HDR เป็น background (เต็มขนาด)
// scene.environment = envMap → ใช้ HDR เป็น light source สำหรับ PBR

// แยกกันได้:
scene.background = new THREE.Color('#1a1a2e'); // พื้นหลังสีทึบ
scene.environment = envMap;                    // แต่ยังใช้ HDR เป็นแสง

// ========== Equirectangular Texture (non-HDR) ==========

const pngEnvLoader = new THREE.TextureLoader();
const pngEnv = pngEnvLoader.load('/textures/env.jpg');
pngEnv.mapping = THREE.EquirectangularReflectionMapping;
pngEnv.colorSpace = THREE.SRGBColorSpace;

scene.environment = pngEnv;  // ใช้ได้ แต่ dynamic range ต่ำกว่า HDR

// ========== CubeTexture Environment ==========

const cubeLoader = new THREE.CubeTextureLoader();
const cubeTexture = cubeLoader.load([
    '/textures/cube/px.jpg', '/textures/cube/nx.jpg',
    '/textures/cube/py.jpg', '/textures/cube/ny.jpg',
    '/textures/cube/pz.jpg', '/textures/cube/nz.jpg',
]);
scene.background = cubeTexture;
scene.environment = cubeTexture;
```

---

## 11. Matcap Texture

MatCap (Material Capture) เป็น all-in-one texture ที่ capture ทั้ง lighting และ material

```javascript
// MatCap texture ดูเหมือนลูกกลมที่ถ่ายรูปไว้ใน lighting condition นั้น
// ทำงานโดย: map normal direction (view space) → UV บน matcap texture

const matcapLoader = new THREE.TextureLoader();
const matcapTexture = matcapLoader.load('/textures/matcap/metal_01.jpg');
matcapTexture.colorSpace = THREE.SRGBColorSpace;

const matcapMat = new THREE.MeshMatcapMaterial({
    matcap: matcapTexture,
    side: THREE.DoubleSide,
});

// ========== สร้าง Matcap ด้วย Canvas ==========

function createMatcapTexture(config = {}) {
    const {
        size = 256,
        baseColor = '#3a3a7a',
        rimColor = '#6c63ff',
        highlightColor = '#ffffff',
        shadowColor = '#000020',
    } = config;
    
    const canvas = document.createElement('canvas');
    canvas.width = canvas.height = size;
    const ctx = canvas.getContext('2d');
    
    // Shadow (bottom left area)
    const shadowGrad = ctx.createRadialGradient(
        size*0.35, size*0.65, 0,
        size*0.5, size*0.5, size*0.5
    );
    shadowGrad.addColorStop(0, shadowColor);
    shadowGrad.addColorStop(1, 'transparent');
    ctx.fillStyle = shadowGrad;
    ctx.beginPath();
    ctx.arc(size/2, size/2, size/2, 0, Math.PI*2);
    ctx.fill();
    
    // Base color
    const baseGrad = ctx.createRadialGradient(
        size*0.5, size*0.5, 0,
        size*0.5, size*0.5, size*0.5
    );
    baseGrad.addColorStop(0, baseColor);
    baseGrad.addColorStop(0.7, baseColor);
    baseGrad.addColorStop(1, shadowColor);
    ctx.fillStyle = baseGrad;
    ctx.globalAlpha = 0.8;
    ctx.beginPath();
    ctx.arc(size/2, size/2, size/2, 0, Math.PI*2);
    ctx.fill();
    ctx.globalAlpha = 1;
    
    // Rim light (edge highlight)
    const rimGrad = ctx.createRadialGradient(
        size*0.15, size*0.85, 0,
        size*0.5, size*0.5, size*0.5
    );
    rimGrad.addColorStop(0, rimColor);
    rimGrad.addColorStop(0.4, 'transparent');
    ctx.fillStyle = rimGrad;
    ctx.globalCompositeOperation = 'screen';
    ctx.beginPath();
    ctx.arc(size/2, size/2, size/2, 0, Math.PI*2);
    ctx.fill();
    ctx.globalCompositeOperation = 'source-over';
    
    // Main highlight (top right)
    const hlGrad = ctx.createRadialGradient(
        size*0.6, size*0.3, 0,
        size*0.6, size*0.3, size*0.25
    );
    hlGrad.addColorStop(0, highlightColor);
    hlGrad.addColorStop(0.3, 'rgba(255,255,255,0.5)');
    hlGrad.addColorStop(1, 'transparent');
    ctx.fillStyle = hlGrad;
    ctx.globalCompositeOperation = 'screen';
    ctx.beginPath();
    ctx.arc(size/2, size/2, size/2, 0, Math.PI*2);
    ctx.fill();
    ctx.globalCompositeOperation = 'source-over';
    
    const texture = new THREE.CanvasTexture(canvas);
    texture.colorSpace = THREE.SRGBColorSpace;
    return texture;
}

// สร้าง matcap หลายแบบ
const purpleMatcap = createMatcapTexture({ baseColor: '#2a1a5e', rimColor: '#6c63ff' });
const goldMatcap = createMatcapTexture({ baseColor: '#4a3500', rimColor: '#ffd700', highlightColor: '#fff8d0' });
const silverMatcap = createMatcapTexture({ baseColor: '#444', rimColor: '#aaa', highlightColor: '#fff' });
```

---

## 12. สร้าง Procedural Texture ด้วย Canvas API

```javascript
// ========== Wood Grain Texture ==========

function createWoodTexture(width = 512, height = 512) {
    const canvas = document.createElement('canvas');
    canvas.width = width;
    canvas.height = height;
    const ctx = canvas.getContext('2d');
    const imageData = ctx.createImageData(width, height);
    
    for (let y = 0; y < height; y++) {
        for (let x = 0; x < width; x++) {
            // Wood ring pattern
            const nx = x / width - 0.5;
            const ny = y / height - 0.5;
            const dist = Math.sqrt(nx*nx + ny*ny);
            
            // Rings
            const rings = 8;
            const ringValue = Math.sin(dist * Math.PI * rings * 2);
            
            // Grain (direction)
            const grain = Math.sin(x * 0.02 + Math.sin(y * 0.01) * 2);
            
            // Combined
            const value = (ringValue * 0.5 + 0.5) * 0.7 + grain * 0.3;
            
            // Wood colors: dark brown to light tan
            const r = Math.floor(101 + value * 80);  // 101-181
            const g = Math.floor(67 + value * 50);   // 67-117
            const b = Math.floor(33 + value * 20);   // 33-53
            
            const idx = (y * width + x) * 4;
            imageData.data[idx+0] = r;
            imageData.data[idx+1] = g;
            imageData.data[idx+2] = b;
            imageData.data[idx+3] = 255;
        }
    }
    
    ctx.putImageData(imageData, 0, 0);
    const texture = new THREE.CanvasTexture(canvas);
    texture.colorSpace = THREE.SRGBColorSpace;
    return texture;
}

// ========== Brick Pattern ==========

function createBrickTexture(width = 512, height = 256) {
    const canvas = document.createElement('canvas');
    canvas.width = width;
    canvas.height = height;
    const ctx = canvas.getContext('2d');
    
    const brickW = 100, brickH = 40;
    const mortarSize = 6;
    
    // Mortar (grout) background
    ctx.fillStyle = '#aaa8a0';
    ctx.fillRect(0, 0, width, height);
    
    const rows = Math.ceil(height / brickH) + 1;
    const cols = Math.ceil(width / brickW) + 1;
    
    for (let row = 0; row < rows; row++) {
        const offset = row % 2 === 0 ? 0 : brickW / 2;
        for (let col = 0; col < cols; col++) {
            const x = col * brickW - offset + mortarSize / 2;
            const y = row * brickH + mortarSize / 2;
            const w = brickW - mortarSize;
            const h = brickH - mortarSize;
            
            // Random brick color variation
            const variation = Math.random() * 30 - 15;
            const r = Math.min(255, Math.max(0, 180 + variation));
            const g = Math.min(255, Math.max(0, 90 + variation * 0.5));
            const b = Math.min(255, Math.max(0, 60 + variation * 0.3));
            
            ctx.fillStyle = `rgb(${Math.floor(r)},${Math.floor(g)},${Math.floor(b)})`;
            ctx.fillRect(x, y, w, h);
            
            // Subtle texture on brick
            ctx.fillStyle = `rgba(0,0,0,${Math.random() * 0.1})`;
            ctx.fillRect(x, y, w, h);
        }
    }
    
    const texture = new THREE.CanvasTexture(canvas);
    texture.colorSpace = THREE.SRGBColorSpace;
    texture.wrapS = texture.wrapT = THREE.RepeatWrapping;
    return texture;
}

// ========== Scratched Metal Normal Map ==========

function createScratchedNormalMap(size = 512, numScratches = 200) {
    const canvas = document.createElement('canvas');
    canvas.width = canvas.height = size;
    const ctx = canvas.getContext('2d');
    
    // Flat normal base (128, 128, 255)
    ctx.fillStyle = `rgb(128, 128, 255)`;
    ctx.fillRect(0, 0, size, size);
    
    // Scratches
    for (let i = 0; i < numScratches; i++) {
        const x = Math.random() * size;
        const y = Math.random() * size;
        const length = 20 + Math.random() * 100;
        const angle = Math.random() * Math.PI;
        const width = 0.5 + Math.random() * 2;
        
        ctx.save();
        ctx.translate(x, y);
        ctx.rotate(angle);
        
        // Scratch: shifted normal
        const scratchNormal = `rgb(${128 + 40}, ${128}, 200)`;
        ctx.strokeStyle = scratchNormal;
        ctx.lineWidth = width;
        ctx.beginPath();
        ctx.moveTo(-length/2, 0);
        ctx.lineTo(length/2, 0);
        ctx.stroke();
        
        ctx.restore();
    }
    
    return new THREE.CanvasTexture(canvas);
}
```

---

## 13. Complete PBR Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Part 007: PBR Texture Maps</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        html, body { width: 100%; height: 100%; overflow: hidden; background: #0d1117; }
    </style>
</head>
<body>
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
        
        // ====== SETUP ======
        const scene = new THREE.Scene();
        scene.background = new THREE.Color('#0d1117');
        
        const camera = new THREE.PerspectiveCamera(60, innerWidth/innerHeight, 0.1, 100);
        camera.position.set(0, 0, 5);
        
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(innerWidth, innerHeight);
        renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
        renderer.toneMapping = THREE.ACESFilmicToneMapping;
        renderer.toneMappingExposure = 1.2;
        document.body.appendChild(renderer.domElement);
        
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.autoRotate = true;
        controls.autoRotateSpeed = 1;
        
        // ====== LIGHTS ======
        const ambientLight = new THREE.AmbientLight(0xffffff, 0.4);
        scene.add(ambientLight);
        
        const dirLight = new THREE.DirectionalLight(0xffffff, 2);
        dirLight.position.set(5, 5, 5);
        scene.add(dirLight);
        
        const rimLight = new THREE.DirectionalLight(0x8080ff, 1);
        rimLight.position.set(-5, -2, -5);
        scene.add(rimLight);
        
        // ====== PROCEDURAL TEXTURES ======
        
        // 1. Checkerboard Color Map
        function createCheckerboard(size, squares, c1, c2) {
            const cv = document.createElement('canvas');
            cv.width = cv.height = size;
            const ctx = cv.getContext('2d');
            const s = size / squares;
            for (let y = 0; y < squares; y++) {
                for (let x = 0; x < squares; x++) {
                    ctx.fillStyle = (x + y) % 2 === 0 ? c1 : c2;
                    ctx.fillRect(x*s, y*s, s, s);
                }
            }
            const t = new THREE.CanvasTexture(cv);
            t.colorSpace = THREE.SRGBColorSpace;
            return t;
        }
        
        // 2. Procedural Normal Map (bumps)
        function createBumpNormal(size, bumpSize, intensity = 1) {
            const cv = document.createElement('canvas');
            cv.width = cv.height = size;
            const ctx = cv.getContext('2d');
            const img = ctx.createImageData(size, size);
            
            for (let y = 0; y < size; y++) {
                for (let x = 0; x < size; x++) {
                    // Bump height from circles
                    let h = 0;
                    const nx = x / size;
                    const ny = y / size;
                    
                    // Multiple bumps
                    for (let b = 0; b < 20; b++) {
                        const bx = (b * 0.618) % 1;
                        const by = (b * 0.381) % 1;
                        const d = Math.sqrt((nx - bx)**2 + (ny - by)**2);
                        if (d < 0.05) {
                            h += Math.cos(d / 0.05 * Math.PI * 0.5);
                        }
                    }
                    
                    // Calculate normal from height gradient
                    const getH = (px, py) => {
                        let hh = 0;
                        const ppx = px / size, ppy = py / size;
                        for (let b = 0; b < 20; b++) {
                            const bx = (b * 0.618) % 1;
                            const by = (b * 0.381) % 1;
                            const d = Math.sqrt((ppx - bx)**2 + (ppy - by)**2);
                            if (d < 0.05) hh += Math.cos(d / 0.05 * Math.PI * 0.5);
                        }
                        return hh;
                    };
                    
                    const left = getH(x-1, y);
                    const right = getH(x+1, y);
                    const up = getH(x, y-1);
                    const down = getH(x, y+1);
                    
                    let nx2 = (left - right) * intensity;
                    let ny2 = (up - down) * intensity;
                    let nz = 1;
                    const len = Math.sqrt(nx2*nx2 + ny2*ny2 + nz*nz);
                    
                    const i = (y * size + x) * 4;
                    img.data[i+0] = Math.floor((nx2/len * 0.5 + 0.5) * 255);
                    img.data[i+1] = Math.floor((ny2/len * 0.5 + 0.5) * 255);
                    img.data[i+2] = Math.floor((nz/len * 0.5 + 0.5) * 255);
                    img.data[i+3] = 255;
                }
            }
            ctx.putImageData(img, 0, 0);
            return new THREE.CanvasTexture(cv);
        }
        
        // 3. Roughness map (varied roughness)
        function createRoughnessMap(size) {
            const cv = document.createElement('canvas');
            cv.width = cv.height = size;
            const ctx = cv.getContext('2d');
            
            // Radial gradient: center smooth, edge rough
            const g = ctx.createRadialGradient(
                size/2, size/2, 0,
                size/2, size/2, size/2
            );
            g.addColorStop(0, '#ffffff');    // rough center
            g.addColorStop(0.5, '#888888');  // medium
            g.addColorStop(1, '#000000');    // smooth edge (for wear)
            
            ctx.fillStyle = g;
            ctx.fillRect(0, 0, size, size);
            
            // Add noise
            const imgData = ctx.getImageData(0, 0, size, size);
            for (let i = 0; i < imgData.data.length; i += 4) {
                const noise = (Math.random() - 0.5) * 40;
                imgData.data[i] = Math.max(0, Math.min(255, imgData.data[i] + noise));
                imgData.data[i+1] = imgData.data[i];
                imgData.data[i+2] = imgData.data[i];
            }
            ctx.putImageData(imgData, 0, 0);
            
            return new THREE.CanvasTexture(cv);
        }
        
        // 4. AO Map
        function createAOMap(size) {
            const cv = document.createElement('canvas');
            cv.width = cv.height = size;
            const ctx = cv.getContext('2d');
            
            ctx.fillStyle = '#ffffff';
            ctx.fillRect(0, 0, size, size);
            
            // Darken edges (typical AO look)
            const g = ctx.createRadialGradient(
                size/2, size/2, size*0.3,
                size/2, size/2, size/2
            );
            g.addColorStop(0, 'rgba(255,255,255,0)');
            g.addColorStop(1, 'rgba(0,0,0,0.7)');
            
            ctx.fillStyle = g;
            ctx.fillRect(0, 0, size, size);
            
            return new THREE.CanvasTexture(cv);
        }
        
        // 5. Emissive Map (grid pattern)
        function createEmissiveGrid(size, gridLines = 8) {
            const cv = document.createElement('canvas');
            cv.width = cv.height = size;
            const ctx = cv.getContext('2d');
            
            ctx.fillStyle = '#000000';
            ctx.fillRect(0, 0, size, size);
            
            ctx.strokeStyle = '#6c63ff';
            ctx.lineWidth = 2;
            ctx.shadowColor = '#6c63ff';
            ctx.shadowBlur = 8;
            
            const step = size / gridLines;
            for (let i = 0; i <= gridLines; i++) {
                ctx.beginPath();
                ctx.moveTo(i * step, 0);
                ctx.lineTo(i * step, size);
                ctx.stroke();
                ctx.beginPath();
                ctx.moveTo(0, i * step);
                ctx.lineTo(size, i * step);
                ctx.stroke();
            }
            
            const t = new THREE.CanvasTexture(cv);
            t.colorSpace = THREE.SRGBColorSpace;
            return t;
        }
        
        // ====== CREATE SPHERE WITH ALL MAPS ======
        
        const sphereGeo = new THREE.SphereGeometry(1.5, 64, 32);
        sphereGeo.setAttribute('uv1', sphereGeo.attributes.uv.clone()); // UV2 for AO
        
        const colorMap = createCheckerboard(512, 8, '#6c63ff', '#1a1a2e');
        const normalMap = createBumpNormal(512, 15, 3);
        const roughnessMap = createRoughnessMap(512);
        const aoMap = createAOMap(512);
        const emissiveMap = createEmissiveGrid(512, 16);
        
        colorMap.wrapS = colorMap.wrapT = THREE.RepeatWrapping;
        normalMap.wrapS = normalMap.wrapT = THREE.RepeatWrapping;
        roughnessMap.wrapS = roughnessMap.wrapT = THREE.RepeatWrapping;
        
        const pbrMaterial = new THREE.MeshStandardMaterial({
            map: colorMap,
            normalMap: normalMap,
            normalScale: new THREE.Vector2(1, 1),
            roughnessMap: roughnessMap,
            roughness: 1.0,
            metalness: 0.3,
            aoMap: aoMap,
            aoMapIntensity: 1.0,
            emissiveMap: emissiveMap,
            emissive: '#ffffff',
            emissiveIntensity: 0.5,
        });
        
        const sphere = new THREE.Mesh(sphereGeo, pbrMaterial);
        scene.add(sphere);
        
        // ====== GUI ======
        const gui = new GUI({ title: 'PBR Maps' });
        
        const params = {
            metalness: 0.3,
            roughness: 1.0,
            normalScale: 1.0,
            aoIntensity: 1.0,
            emissiveIntensity: 0.5,
            showNormals: false,
        };
        
        gui.add(params, 'metalness', 0, 1, 0.01).onChange(v => pbrMaterial.metalness = v);
        gui.add(params, 'roughness', 0, 1, 0.01).onChange(v => pbrMaterial.roughness = v);
        gui.add(params, 'normalScale', 0, 3, 0.01).onChange(v => {
            pbrMaterial.normalScale.set(v, v);
        });
        gui.add(params, 'aoIntensity', 0, 2, 0.01).onChange(v => pbrMaterial.aoMapIntensity = v);
        gui.add(params, 'emissiveIntensity', 0, 3, 0.01).onChange(v => {
            pbrMaterial.emissiveIntensity = v;
        });
        gui.add(params, 'showNormals').onChange(v => {
            sphere.material = v 
                ? new THREE.MeshNormalMaterial() 
                : pbrMaterial;
        });
        
        // ====== ANIMATION ======
        const clock = new THREE.Clock();
        
        function animate() {
            requestAnimationFrame(animate);
            const elapsed = clock.getElapsedTime();
            clock.getDelta();
            
            // Pulse emissive
            pbrMaterial.emissiveIntensity = params.emissiveIntensity * (0.8 + Math.sin(elapsed * 2) * 0.2);
            
            // Rotate lights
            dirLight.position.set(
                Math.sin(elapsed * 0.5) * 5,
                5,
                Math.cos(elapsed * 0.5) * 5
            );
            
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

## สรุป PBR Texture Maps

| Map | Format | Color Space | ใช้ Channel | หน้าที่ |
|-----|--------|-------------|------------|---------|
| Color/Albedo | RGB | SRGBColorSpace | RGB | สีพื้นผิว |
| Normal | RGB | NoColorSpace | RGB=XYZ | รายละเอียด surface |
| Roughness | Grayscale | NoColorSpace | G | ความหยาบ |
| Metalness | Grayscale | NoColorSpace | B | ความเป็นโลหะ |
| AO | Grayscale | NoColorSpace | R | เงา Ambient |
| Displacement | Grayscale | NoColorSpace | R | เขยิบ geometry |
| Emissive | RGB | SRGBColorSpace | RGB | ส่วนที่เรืองแสง |
| Alpha | Grayscale | NoColorSpace | R | ความโปร่งแสง |

---

## แบบฝึกหัด (Exercises)

### Exercise 1: PBR Material Setup (ง่าย)
โหลด PBR texture set จาก [ambientCG.com](https://ambientcg.com/) (ฟรี)  
สร้าง MeshStandardMaterial โดยใช้ทุก map ที่มี

### Exercise 2: Procedural Marble (ปานกลาง)
สร้าง marble texture ด้วย Canvas API:
- สีขาวเป็นพื้น
- เส้นสีเทา/ดำที่วาดแบบ random (veins)
- Normal map ที่สร้างจาก marble texture

### Exercise 3: ORM Texture (ปานกลาง)
สร้าง ORM (Occlusion, Roughness, Metalness) texture เดียว:
- R channel = AO
- G channel = Roughness
- B channel = Metalness
ใช้กับ MeshStandardMaterial โดยใช้ texture เดียวในสาม slots

### Exercise 4: Animated Lava (ยาก)
สร้าง lava material:
- Color map: สีส้ม/แดง
- Emissive map: สีส้ม/เหลืองที่ animated (canvas)
- Displacement map: noise ที่ animate (cube หรือ sphere)
- Emissive intensity สูง (2-5) สำหรับ bloom effect

### Exercise 5: Normal Map Comparison (ปานกลาง)
สร้าง 4 sphere เรียงกัน:
1. No maps (flat)
2. Bump map only
3. Normal map only
4. Both (normal + displacement)
แสดงให้เห็นความแตกต่างชัดเจน

---

**ก่อนหน้า:** [Part 006 - Textures Part 1](part-006.md)  
**ต่อไป:** [Part 008 - Lighting Part 1](part-008.md)
