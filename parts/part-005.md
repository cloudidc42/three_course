# Part 005: Materials Part 2 - PBR Materials (Physically Based Rendering)

## สารบัญบทเรียนนี้
- [1. PBR คืออะไร?](#1-pbr-คืออะไร)
- [2. MeshStandardMaterial](#2-meshstandardmaterial)
- [3. Metalness & Roughness อธิบาย](#3-metalness--roughness-อธิบาย)
- [4. Normal Map, AO Map, Emissive Map](#4-normal-map-ao-map-emissive-map)
- [5. MeshPhysicalMaterial - PBR ขั้นสูง](#5-meshphysicalmaterial---pbr-ขั้นสูง)
- [6. Clearcoat, Sheen, Transmission](#6-clearcoat-sheen-transmission)
- [7. MeshToonMaterial](#7-meshtoonmaterial)
- [8. Environment Maps กับ PBR](#8-environment-maps-กับ-pbr)
- [9. Tone Mapping](#9-tone-mapping)
- [10. PBR Sphere Comparison Demo](#10-pbr-sphere-comparison-demo)
- [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. PBR คืออะไร?

**PBR (Physically Based Rendering)** คือวิธีการ render ที่พยายามจำลองวิธีที่แสงโต้ตอบกับวัตถุในโลกแห่งความเป็นจริงตามหลักฟิสิกส์

### ทำไม PBR ถึงดีกว่า?

**แบบเก่า (Phong/Lambert):**
- ปรับ specular/shininess ด้วยความรู้สึก
- ผลลัพธ์แตกต่างกันมากตาม lighting setup
- ยากที่จะ match กับโลกจริง

**PBR:**
- ใช้ properties ที่วัดได้จริงในโลกจริง (metalness, roughness)
- ผลลัพธ์สม่ำเสมอในทุก lighting setup
- Artists และ developers ใช้ asset เดิมได้ในทุก engine

### Energy Conservation

กฎสำคัญของ PBR: แสงที่สะท้อน + แสงที่ดูดซึม = แสงที่ตกกระทบ  
(ไม่สามารถสะท้อนแสงได้มากกว่าที่รับมา)

```
แสงตกกระทบ (incident light)
    ↓
    ├── Reflected (สะท้อน) ← กำหนดโดย reflectivity
    └── Absorbed (ดูดซึม)
        ↓
        ├── Scattered (กระจาย = diffuse color)
        └── Absorbed completely (ความร้อน)
```

---

## 2. MeshStandardMaterial

Material PBR หลักของ Three.js  
ใช้ได้ทั่วไปในทุก production project

```javascript
// ⚠️ ต้องมีแสงในฉาก (สำหรับ PBR ที่ถูกต้อง ต้องมี environment map หรือ lights)

const standardMat = new THREE.MeshStandardMaterial({
    // ===== สีหลัก =====
    color: 0xffffff,           // base color / albedo (default: white)
    
    // ===== PBR Core Properties =====
    metalness: 0.0,            // ความเป็นโลหะ (0=non-metal, 1=metal)
    roughness: 1.0,            // ความหยาบ (0=smooth/mirror, 1=rough/matte)
    
    // ===== Texture Maps =====
    map: null,                 // albedo/color texture
    metalnessMap: null,        // metalness per-pixel
    roughnessMap: null,        // roughness per-pixel
    normalMap: null,           // normal map
    normalMapType: THREE.TangentSpaceNormalMap,
    normalScale: new THREE.Vector2(1, 1), // strength ของ normal map
    aoMap: null,               // Ambient Occlusion map
    aoMapIntensity: 1.0,       // strength ของ AO
    displacementMap: null,     // displacement/height map
    displacementScale: 1.0,    // ความแรงของ displacement
    displacementBias: 0,       // offset ของ displacement
    emissiveMap: null,         // emissive texture
    alphaMap: null,            // alpha texture
    envMap: null,              // environment reflection map
    envMapIntensity: 1.0,      // strength ของ env reflection
    bumpMap: null,             // bump map (alternative to normalMap)
    bumpScale: 1,              // strength ของ bump
    
    // ===== Emissive =====
    emissive: 0x000000,        // emissive color (ส่งแสงออกมาเอง)
    emissiveIntensity: 1.0,    // strength ของ emissive
    
    // ===== Transparency =====
    transparent: false,
    opacity: 1.0,
    alphaTest: 0,
    alphaToCoverage: false,
    
    // ===== Display =====
    side: THREE.FrontSide,
    wireframe: false,
    flatShading: false,
    
    // ===== Fog =====
    fog: true,
});

// ========== ตัวอย่างวัตถุในชีวิตจริง ==========

// พลาสติกสีแดง
const redPlastic = new THREE.MeshStandardMaterial({
    color: '#e74c3c',
    metalness: 0.0,   // ไม่ใช่โลหะ
    roughness: 0.5,   // กึ่งมันกึ่งด้าน
});

// เหล็กขัดเงา
const polishedSteel = new THREE.MeshStandardMaterial({
    color: '#aaaaaa',
    metalness: 1.0,   // โลหะสมบูรณ์
    roughness: 0.1,   // เรียบมาก (สะท้อนแสงได้ดี)
});

// เหล็กเก่าขึ้นสนิม
const rustyMetal = new THREE.MeshStandardMaterial({
    color: '#8b4513',
    metalness: 0.7,
    roughness: 0.9,   // หยาบมาก
});

// ยาง/ผ้า
const rubber = new THREE.MeshStandardMaterial({
    color: '#222222',
    metalness: 0.0,
    roughness: 0.9,   // หยาบ ไม่สะท้อน
});

// ทอง
const gold = new THREE.MeshStandardMaterial({
    color: '#ffd700',
    metalness: 1.0,
    roughness: 0.2,
});

// กระจก (เราต้องใช้ MeshPhysicalMaterial สำหรับ refraction)
const glass = new THREE.MeshStandardMaterial({
    color: '#ffffff',
    metalness: 0.0,
    roughness: 0.0,
    transparent: true,
    opacity: 0.3,
    envMapIntensity: 2.0, // เพิ่ม reflection
});
```

---

## 3. Metalness & Roughness อธิบาย

### Metalness (ความเป็นโลหะ)

```javascript
// metalness = 0 (Non-metal / Dielectric)
// - แสงที่ผ่านเข้าจะกระจายออก (diffuse)
// - มีสีของตัวเอง (color มีผล)
// - Reflection เป็นสีขาว (specular ไม่มีสี)
// - ตัวอย่าง: ไม้, หิน, พลาสติก, ผ้า, ดิน

// metalness = 1 (Metal / Conductor)
// - แสงเกือบทั้งหมดจะสะท้อนออก (specular)
// - สีของ reflection คือสีของโลหะนั้น (color มีผล)
// - Diffuse เกือบไม่มี
// - ตัวอย่าง: เหล็ก, ทอง, ทองแดง, อลูมิเนียม

// metalness ระหว่าง 0-1: ไม่ค่อยเป็น realistic
// ในโลกจริง วัตถุส่วนใหญ่เป็น 0 หรือ 1
// ค่ากลางๆ ใช้สำหรับ: dirty metal, painted metal
```

### Roughness (ความหยาบ)

```javascript
// roughness = 0 (Smooth / Mirror-like)
// - สะท้อนแสงชัดเจน (perfect specular reflection)
// - reflection คมชัด
// - ตัวอย่าง: กระจก, น้ำนิ่ง, พื้นผิวขัดเงา

// roughness = 1 (Rough / Matte)
// - กระจาย reflection ไปหลายทิศทาง (diffuse reflection)
// - reflection เบลอหรือไม่มี
// - ตัวอย่าง: คอนกรีต, ดิน, กระดาษ, ผ้า

// ความสัมพันธ์ roughness กับ specular:
// roughness ต่ำ → specular เล็กแต่สว่างและคม
// roughness สูง → specular กว้างและนวล

// ตาราง roughness ของวัตถุต่างๆ:
// กระจก/น้ำ: 0.0
// โลหะขัดเงา: 0.05 - 0.15
// พลาสติกมัน: 0.1 - 0.3
// ไม้ขัดเงา: 0.3 - 0.5
// พลาสติกด้าน: 0.5 - 0.7
// ผ้า/ยาง: 0.7 - 0.9
// คอนกรีต/หิน: 0.8 - 1.0
```

### Metalness/Roughness Map

```javascript
// metalnessMap และ roughnessMap มักเก็บอยู่ใน texture เดียวกัน
// channel B = metalness
// channel G = roughness
// (ประหยัด memory)

const loader = new THREE.TextureLoader();

// โหลด texture เดียวที่มี Metalness + Roughness
const metalRoughMap = loader.load('/textures/metal_roughness.jpg');

const material = new THREE.MeshStandardMaterial({
    color: '#ffffff',
    metalnessMap: metalRoughMap,
    roughnessMap: metalRoughMap,
    metalness: 1.0,   // scale factor
    roughness: 1.0,   // scale factor
    // ค่าจริง = map_value * scalar
});
```

---

## 4. Normal Map, AO Map, Emissive Map

### Normal Map

```javascript
// Normal Map เก็บทิศทาง normal vector ในรูปสี RGB
// R = normal X, G = normal Y, B = normal Z

// ทำให้ surface ดูมีรายละเอียดโดยไม่เพิ่ม polygon
// ✅ ดี: detail มาก, performance ดี
// ❌ ข้อเสีย: ขอบไม่ได้เปลี่ยน (silhouette ยังเรียบอยู่)

const loader = new THREE.TextureLoader();
const normalMap = loader.load('/textures/normal.jpg');

const material = new THREE.MeshStandardMaterial({
    color: '#888888',
    normalMap: normalMap,
    normalScale: new THREE.Vector2(1, 1),
    // normalScale = ความแรงของ normal map
    // (1, 1) = full strength
    // (0.5, 0.5) = half strength
    // (2, 2) = doubled (อาจดูไม่เป็นธรรมชาติ)
    // (-1, 1) = flip horizontal (ถ้า normal map ดูกลับทิศ)
});

// Normal Map Types:
// TangentSpaceNormalMap (default): relative to surface
// ObjectSpaceNormalMap: absolute (น้อยกว่า ใช้น้อยกว่า)

// Bump Map (alternative ที่เก่ากว่า)
// เก็บ height information (grayscale)
// Three.js คำนวณ normal จาก height ให้
// ง่ายกว่าแต่ quality ต่ำกว่า normal map

const bumpMaterial = new THREE.MeshStandardMaterial({
    bumpMap: loader.load('/textures/bump.jpg'),
    bumpScale: 0.1,  // ความแรง
});
```

### Ambient Occlusion (AO) Map

```javascript
// AO Map เก็บว่าจุดไหน "ถูกปิดกั้น" จากแสงรอบทิศ
// สีขาว = แสงเต็ม, สีดำ = เงามืด

// AO ทำให้ corner, crease, cavity ดูมืดและลึกขึ้น
// ทำให้ scene ดูสมจริงและมีน้ำหนักมากขึ้น

const aoMap = loader.load('/textures/ao.jpg');

// ⚠️ AO map ต้องใช้ UV ชุดที่สอง (UV1/UV2)
// Three.js r149+ ใช้ attribute ชื่อ 'uv1'
geometry.setAttribute('uv1', geometry.attributes.uv.clone());

const material = new THREE.MeshStandardMaterial({
    aoMap: aoMap,
    aoMapIntensity: 1.0,  // 0 = ไม่มีผล, 1 = full effect, 2 = exaggerated
});

// ถ้า model จาก Blender มักมี UV2 อยู่แล้ว
// แต่ถ้าไม่มี ต้องสร้างเอง:
mesh.geometry.setAttribute('uv1', mesh.geometry.attributes.uv.clone());
```

### Emissive Map

```javascript
// Emissive Map กำหนดว่าจุดไหนส่งแสงออกมา
// ใช้สำหรับ: หน้าจอ, LED, lava, neon signs

const emissiveMap = loader.load('/textures/emissive.jpg');

const screenMaterial = new THREE.MeshStandardMaterial({
    color: '#000000',
    emissive: '#ffffff',        // base emissive color
    emissiveMap: emissiveMap,   // emissive texture
    emissiveIntensity: 1.5,     // intensity ของ emission
});

// Animated emissive (flicker effect)
function animate() {
    requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    screenMaterial.emissiveIntensity = 0.8 + Math.sin(elapsed * 10) * 0.2;
    renderer.render(scene, camera);
}

// Glowing effect โดยรวม emissive กับ Bloom post-processing
// (จะเรียน Bloom ใน Part 044)
const lava = new THREE.MeshStandardMaterial({
    color: '#ff4400',
    emissive: '#ff2200',
    emissiveIntensity: 2.0,  // HDR value (> 1.0)
    metalness: 0.0,
    roughness: 0.9,
});
```

---

## 5. MeshPhysicalMaterial - PBR ขั้นสูง

`MeshPhysicalMaterial` extends `MeshStandardMaterial`  
เพิ่ม features ขั้นสูง: clearcoat, sheen, transmission, iridescence

```javascript
// ⚠️ MeshPhysicalMaterial ช้ากว่า MeshStandardMaterial
// ใช้เฉพาะเมื่อจำเป็นต้องการ features พิเศษ

const physical = new THREE.MeshPhysicalMaterial({
    // === Properties จาก MeshStandardMaterial ===
    color: '#ffffff',
    metalness: 0,
    roughness: 0,
    
    // === Clearcoat (เคลือบเงา) ===
    clearcoat: 0,              // ความหนาของ clear coat (0-1)
    clearcoatRoughness: 0,     // ความหยาบของ clear coat
    clearcoatMap: null,        // clearcoat intensity map
    clearcoatNormalMap: null,  // normal map สำหรับ clear coat
    clearcoatNormalScale: new THREE.Vector2(1, 1),
    
    // === Sheen (ผ้า/velvet effect) ===
    sheen: 0,                  // ความแรงของ sheen (0-1)
    sheenRoughness: 1,
    sheenColor: '#000000',     // สีของ sheen
    sheenColorMap: null,
    sheenRoughnessMap: null,
    
    // === Transmission (กระจก/แก้ว) ===
    transmission: 0,           // ความโปร่งใส (0=opaque, 1=clear)
    transmissionMap: null,
    
    // === IOR (Index of Refraction) ===
    ior: 1.5,                  // ค่า refraction (น้ำ=1.33, แก้ว=1.5, diamond=2.4)
    
    // === Thickness (สำหรับ transmission) ===
    thickness: 0,              // ความหนาของ material (สำหรับ absorption)
    thicknessMap: null,
    attenuationColor: '#ffffff', // สีที่ถูกดูดซึมภายใน
    attenuationDistance: Infinity, // ระยะที่ attenuate
    
    // === Iridescence (สีรุ้ง/สบู่) ===
    iridescence: 0,
    iridescenceIOR: 1.3,
    iridescenceThicknessRange: [100, 400],
    iridescenceMap: null,
    iridescenceThicknessMap: null,
    
    // === Anisotropy (brushed metal effect) ===
    anisotropy: 0,
    anisotropyRotation: 0,
    anisotropyMap: null,
    
    // === Specular ===
    specularIntensity: 1,      // ความแรงของ specular (non-metal)
    specularColor: '#ffffff',  // สีของ specular
    specularIntensityMap: null,
    specularColorMap: null,
});
```

---

## 6. Clearcoat, Sheen, Transmission

### Clearcoat - เคลือบเงา

```javascript
// Clearcoat จำลอง layer เคลือบใสบน material หลัก
// ใช้สำหรับ: รถยนต์, ไม้ขัดเงา, เล็บทาสี

const carPaint = new THREE.MeshPhysicalMaterial({
    color: '#c0392b',   // สีรถ
    metalness: 0.0,
    roughness: 0.5,
    
    clearcoat: 1.0,     // เต็มที่
    clearcoatRoughness: 0.1,  // เกือบไม่มีความหยาบ
});

// ถ้ามี normalMap สำหรับ paint (orange peel effect)
carPaint.normalMap = loader.load('/textures/car_normal.jpg');
carPaint.normalScale.set(0.5, 0.5);

// clearcoat normal map (scratches บน clearcoat layer)
carPaint.clearcoatNormalMap = loader.load('/textures/scratches_normal.jpg');
carPaint.clearcoatNormalScale.set(0.3, 0.3);
```

### Sheen - ผ้า/Velvet

```javascript
// Sheen จำลอง retroreflective effect ของผ้า
// ขนเล็กๆ ที่ edge ของผ้าสะท้อนแสงกลับมาหาผู้ดู

const velvet = new THREE.MeshPhysicalMaterial({
    color: '#8b0000',   // สีผ้า
    metalness: 0.0,
    roughness: 0.9,
    
    sheen: 1.0,             // เต็มที่
    sheenColor: '#ff0000',  // สีของ sheen highlight
    sheenRoughness: 0.3,    // ความหยาบของ sheen
});

// Satin fabric
const satin = new THREE.MeshPhysicalMaterial({
    color: '#4a90d9',
    metalness: 0.0,
    roughness: 0.2,
    sheen: 0.8,
    sheenColor: '#ffffff',
    sheenRoughness: 0.1,
});
```

### Transmission - กระจก/แก้ว/น้ำ

```javascript
// ⚠️ Transmission ต้องการ WebGLRenderTarget พิเศษ
// และต้องการ transparent material pass ก่อน

// Setup สำหรับ Transmission
renderer.capabilities; // ต้องรองรับ

// Glass sphere
const glassMaterial = new THREE.MeshPhysicalMaterial({
    color: '#ffffff',
    metalness: 0.0,
    roughness: 0.0,
    
    transmission: 1.0,    // โปร่งใสสมบูรณ์
    ior: 1.5,             // index of refraction ของแก้ว
    
    thickness: 0.5,       // ความหนาของแก้ว
    attenuationColor: '#80ff80', // สีเขียวอ่อนเหมือนแก้วจริง
    attenuationDistance: 5.0,
});

// Diamond
const diamond = new THREE.MeshPhysicalMaterial({
    color: '#ffffff',
    metalness: 0.0,
    roughness: 0.0,
    transmission: 1.0,
    ior: 2.4,             // diamond มี IOR สูงมาก
    thickness: 0.5,
    attenuationColor: '#ffffff',
    attenuationDistance: Infinity,
    dispersion: 5.0,       // ความแตกต่างของ IOR ตาม wavelength (สีรุ้ง)
});

// Tinted glass (เช่น สุนข)
const tintedGlass = new THREE.MeshPhysicalMaterial({
    color: '#4444ff',
    metalness: 0.0,
    roughness: 0.0,
    transmission: 0.9,
    ior: 1.5,
    thickness: 0.3,
    attenuationColor: '#0000ff',
    attenuationDistance: 2.0,
});

// Frosted glass
const frostedGlass = new THREE.MeshPhysicalMaterial({
    color: '#ffffff',
    metalness: 0.0,
    roughness: 0.3,         // ความหยาบทำให้ frosted
    transmission: 0.95,
    ior: 1.5,
    thickness: 0.5,
});
```

### Iridescence - สีรุ้ง (สบู่, CD)

```javascript
// Iridescence เกิดจาก thin film interference
// สีเปลี่ยนตามมุมมอง (เหมือนสบู่, ปีกแมลง, CD)

const soapBubble = new THREE.MeshPhysicalMaterial({
    color: '#ffffff',
    metalness: 0.0,
    roughness: 0.0,
    transparent: true,
    opacity: 0.3,
    
    iridescence: 1.0,
    iridescenceIOR: 1.3,
    iridescenceThicknessRange: [100, 800], // nm (ความหนาของ film)
});

// CD/DVD effect
const cdMaterial = new THREE.MeshPhysicalMaterial({
    color: '#888888',
    metalness: 0.5,
    roughness: 0.1,
    
    iridescence: 1.0,
    iridescenceIOR: 1.5,
    iridescenceThicknessRange: [200, 600],
});
```

---

## 7. MeshToonMaterial

จำลองการ shade แบบ cartoon/anime

```javascript
// MeshToonMaterial ใช้ gradient map เพื่อ quantize แสง
// แทนที่จะ smooth gradient ใช้ step ที่ชัดเจน

const toonMat = new THREE.MeshToonMaterial({
    color: '#6c63ff',
    
    // Gradient Map: texture 1D ที่กำหนด "steps" ของแสง
    // ถ้าไม่กำหนด จะใช้ default 3 steps
    gradientMap: null,
    
    map: null,
    normalMap: null,
    normalScale: new THREE.Vector2(1, 1),
    aoMap: null,
    aoMapIntensity: 1.0,
    emissive: '#000000',
    emissiveMap: null,
    emissiveIntensity: 1.0,
    alphaMap: null,
    
    wireframe: false,
    fog: true,
    transparent: false,
    opacity: 1.0,
});

// สร้าง gradient map แบบ 3 steps (dark, mid, bright)
function createToonGradient(steps = 3) {
    const size = 256;
    const canvas = document.createElement('canvas');
    canvas.width = size;
    canvas.height = 1;
    const ctx = canvas.getContext('2d');
    
    const stepSize = size / steps;
    for (let i = 0; i < steps; i++) {
        const brightness = i / (steps - 1);
        ctx.fillStyle = `rgb(${Math.round(brightness * 255)}, ${Math.round(brightness * 255)}, ${Math.round(brightness * 255)})`;
        ctx.fillRect(i * stepSize, 0, stepSize, 1);
    }
    
    const texture = new THREE.CanvasTexture(canvas);
    texture.magFilter = THREE.NearestFilter;
    texture.minFilter = THREE.NearestFilter;
    return texture;
}

// 2 steps (very dramatic)
const gradient2 = createToonGradient(2);
// 3 steps (default toon look)
const gradient3 = createToonGradient(3);
// 5 steps (softer)
const gradient5 = createToonGradient(5);

// Toon material พร้อม outline
// ต้องเพิ่ม outline แยกต่างหาก (invert normals technique)
function createToonWithOutline(color, outlineColor = '#000000', outlineSize = 0.05) {
    const mesh = new THREE.Mesh(
        geometry,
        new THREE.MeshToonMaterial({ color, gradientMap: gradient3 })
    );
    
    // Outline: scale up slightly + invert normals + render with BackSide
    const outlineMesh = new THREE.Mesh(
        geometry.clone(),
        new THREE.MeshBasicMaterial({
            color: outlineColor,
            side: THREE.BackSide, // render ด้านหลัง = outline
        })
    );
    outlineMesh.scale.setScalar(1 + outlineSize);
    mesh.add(outlineMesh);
    
    return mesh;
}
```

---

## 8. Environment Maps กับ PBR

Environment map ทำให้ PBR materials สะท้อน environment รอบๆ ได้

```javascript
import { RGBELoader } from 'three/addons/loaders/RGBELoader.js';
import { PMREMGenerator } from 'three';

// ====== Method 1: ใช้ HDR Environment Map ======

const rgbeLoader = new RGBELoader();
const pmremGenerator = new PMREMGenerator(renderer);
pmremGenerator.compileEquirectangularShader();

// โหลด HDR file
rgbeLoader.load('/textures/hdr/studio.hdr', (hdrTexture) => {
    // แปลง equirectangular HDR → cubemap ที่พร้อมใช้
    const envMap = pmremGenerator.fromEquirectangular(hdrTexture).texture;
    
    // ใช้เป็น background
    scene.background = envMap;
    
    // ใช้เป็น environment light (ทุก PBR material จะสะท้อน)
    scene.environment = envMap;
    
    // cleanup
    hdrTexture.dispose();
    pmremGenerator.dispose();
});

// scene.environment = envMap จะ apply ให้ทุก material ที่มี envMap support
// ไม่ต้องตั้ง material.envMap = envMap ทุกตัว

// ====== Method 2: กำหนด envMap ให้ material โดยตรง ======

const material = new THREE.MeshStandardMaterial({
    color: '#aaaaaa',
    metalness: 1.0,
    roughness: 0.0,
    envMap: envMap,          // กำหนดให้ material โดยตรง
    envMapIntensity: 1.0,    // strength (0=ไม่สะท้อน, 1=ปกติ, 2=สะท้อนเยอะ)
});

// ====== Method 3: CubeCamera (Real-time reflections) ======

// สร้าง CubeCamera สำหรับ real-time reflection
const cubeCamera = new THREE.CubeCamera(0.1, 100, 
    new THREE.WebGLCubeRenderTarget(256, {
        format: THREE.RGBAFormat,
        generateMipmaps: true,
        minFilter: THREE.LinearMipmapLinearFilter,
    })
);
scene.add(cubeCamera);

// Chrome sphere
const chromeMaterial = new THREE.MeshStandardMaterial({
    metalness: 1.0,
    roughness: 0.0,
    envMap: cubeCamera.renderTarget.texture,
});

// Update ใน animation loop
function animate() {
    requestAnimationFrame(animate);
    
    // ซ่อน sphere แล้ว render environment
    chromeSphere.visible = false;
    cubeCamera.position.copy(chromeSphere.position);
    cubeCamera.update(renderer, scene);
    chromeSphere.visible = true;
    
    renderer.render(scene, camera);
}
```

---

## 9. Tone Mapping

Tone Mapping แปลง HDR values → display range (0-1)

```javascript
// ตั้งค่า renderer
renderer.toneMapping = THREE.ACESFilmicToneMapping; // แนะนำ
renderer.toneMappingExposure = 1.0; // exposure

// Tone Mapping Types:
// THREE.NoToneMapping - ไม่ทำอะไร
// THREE.LinearToneMapping - แค่ scale ด้วย exposure
// THREE.ReinhardToneMapping - classic photographic tone mapping
// THREE.CineonToneMapping - เหมือนกล้อง film
// THREE.ACESFilmicToneMapping - ACES standard (industry standard)
// THREE.AgXToneMapping - เหมาะกับสีธรรมชาติ
// THREE.NeutralToneMapping - neutral/minimal

// ตัวอย่างเปรียบเทียบ:
const toneMappingOptions = {
    'No Tone Mapping': THREE.NoToneMapping,
    'Linear': THREE.LinearToneMapping,
    'Reinhard': THREE.ReinhardToneMapping,
    'Cineon': THREE.CineonToneMapping,
    'ACES Filmic': THREE.ACESFilmicToneMapping,
    'AgX': THREE.AgXToneMapping,
    'Neutral': THREE.NeutralToneMapping,
};

// ⚠️ เมื่อเปลี่ยน toneMapping ต้อง update ทุก material
renderer.toneMapping = THREE.ACESFilmicToneMapping;
scene.traverse((object) => {
    if (object.isMesh) {
        object.material.needsUpdate = true;
    }
});

// Color Space
renderer.outputColorSpace = THREE.SRGBColorSpace; // default ใน r152+
// THREE.SRGBColorSpace: output เป็น sRGB (ถูกต้องสำหรับ monitor)
// THREE.LinearSRGBColorSpace: output เป็น linear (ใช้เมื่อ post-process ต้อง linear)
```

---

## 10. PBR Sphere Comparison Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Part 005: PBR Materials</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        html, body { width: 100%; height: 100%; overflow: hidden; background: #000; }
        
        #labels {
            position: fixed;
            bottom: 20px;
            left: 50%;
            transform: translateX(-50%);
            display: flex;
            gap: 20px;
            flex-wrap: wrap;
            justify-content: center;
            pointer-events: none;
        }
        
        .label {
            background: rgba(0,0,0,0.7);
            color: #aaa;
            padding: 4px 10px;
            border-radius: 4px;
            font-family: monospace;
            font-size: 11px;
            border: 1px solid #333;
        }
        .label span { color: #6c63ff; display: block; font-size: 10px; }
    </style>
</head>
<body>
    <div id="labels"></div>
    
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
        import { RGBELoader } from 'three/addons/loaders/RGBELoader.js';
        
        // ====== SETUP ======
        const scene = new THREE.Scene();
        
        const camera = new THREE.PerspectiveCamera(60, innerWidth/innerHeight, 0.1, 100);
        camera.position.set(0, 1, 12);
        
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(innerWidth, innerHeight);
        renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
        renderer.toneMapping = THREE.ACESFilmicToneMapping;
        renderer.toneMappingExposure = 1.2;
        renderer.outputColorSpace = THREE.SRGBColorSpace;
        document.body.appendChild(renderer.domElement);
        
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.target.set(0, 0, 0);
        
        // ====== LIGHTS ======
        const ambientLight = new THREE.AmbientLight(0xffffff, 0.3);
        scene.add(ambientLight);
        
        const dirLight1 = new THREE.DirectionalLight(0xffffff, 2);
        dirLight1.position.set(5, 5, 5);
        scene.add(dirLight1);
        
        const dirLight2 = new THREE.DirectionalLight(0x8080ff, 1);
        dirLight2.position.set(-5, 5, -5);
        scene.add(dirLight2);
        
        // Hemisphere light (sky/ground)
        const hemiLight = new THREE.HemisphereLight(0x8080ff, 0x404020, 0.8);
        scene.add(hemiLight);
        
        // ====== GEOMETRY ======
        const sphereGeo = new THREE.SphereGeometry(0.9, 64, 32);
        
        // ====== PBR MATERIALS COMPARISON ======
        
        // แต่ละ sphere = metalness/roughness ต่างกัน
        const sphereData = [
            // [metalness, roughness, label, description]
            [0.0, 0.0, 'Plastic Glossy', 'metal:0 rough:0'],
            [0.0, 0.25, 'Plastic Semi', 'metal:0 rough:0.25'],
            [0.0, 0.5, 'Plastic Matte', 'metal:0 rough:0.5'],
            [0.0, 0.75, 'Rubber', 'metal:0 rough:0.75'],
            [0.0, 1.0, 'Chalk/Concrete', 'metal:0 rough:1'],
            [0.5, 0.5, 'Dirty Metal', 'metal:0.5 rough:0.5'],
            [1.0, 0.0, 'Mirror Metal', 'metal:1 rough:0'],
            [1.0, 0.25, 'Polished Steel', 'metal:1 rough:0.25'],
            [1.0, 0.5, 'Brushed Metal', 'metal:1 rough:0.5'],
            [1.0, 0.75, 'Oxidized Metal', 'metal:1 rough:0.75'],
            [1.0, 1.0, 'Rusty Metal', 'metal:1 rough:1'],
        ];
        
        const materials = [];
        const sphereMeshes = [];
        
        const cols = 5;
        const spacing = 2.3;
        const startX = -((sphereData.length - 1) / 2) * spacing / 2;
        
        // สร้างไม่กี่แถว
        sphereData.forEach((data, i) => {
            const [metalness, roughness, label, desc] = data;
            
            const mat = new THREE.MeshStandardMaterial({
                color: '#b0b0b0',
                metalness,
                roughness,
            });
            materials.push(mat);
            
            const mesh = new THREE.Mesh(sphereGeo, mat);
            const row = Math.floor(i / cols);
            const col = i % cols;
            mesh.position.x = (col - (cols - 1) / 2) * spacing;
            mesh.position.y = -row * spacing;
            mesh.castShadow = true;
            scene.add(mesh);
            sphereMeshes.push(mesh);
        });
        
        // Center camera
        camera.position.y = -((Math.floor(sphereData.length / cols) - 1) * spacing) / 2;
        controls.target.y = camera.position.y;
        
        // ====== SPECIAL MATERIALS ======
        
        // Physical Material Row
        const physicalData = [
            {
                name: 'Clearcoat',
                mat: new THREE.MeshPhysicalMaterial({
                    color: '#c0392b',
                    metalness: 0,
                    roughness: 0.5,
                    clearcoat: 1.0,
                    clearcoatRoughness: 0.05,
                })
            },
            {
                name: 'Glass',
                mat: new THREE.MeshPhysicalMaterial({
                    color: '#ffffff',
                    metalness: 0,
                    roughness: 0,
                    transmission: 1.0,
                    ior: 1.5,
                    thickness: 0.5,
                })
            },
            {
                name: 'Velvet',
                mat: new THREE.MeshPhysicalMaterial({
                    color: '#8b0000',
                    metalness: 0,
                    roughness: 0.9,
                    sheen: 1.0,
                    sheenColor: '#ff4444',
                    sheenRoughness: 0.2,
                })
            },
            {
                name: 'Iridescent',
                mat: new THREE.MeshPhysicalMaterial({
                    color: '#888888',
                    metalness: 0.5,
                    roughness: 0.1,
                    iridescence: 1.0,
                    iridescenceIOR: 1.5,
                    iridescenceThicknessRange: [100, 500],
                })
            },
            {
                name: 'Toon',
                mat: new THREE.MeshToonMaterial({
                    color: '#6c63ff',
                })
            },
        ];
        
        const physicalY = -((Math.floor(sphereData.length / cols)) * spacing) - spacing * 0.5;
        
        physicalData.forEach((data, i) => {
            const mesh = new THREE.Mesh(sphereGeo, data.mat);
            mesh.position.x = (i - (physicalData.length - 1) / 2) * spacing;
            mesh.position.y = physicalY;
            mesh.castShadow = true;
            scene.add(mesh);
        });
        
        // ====== BACKGROUND ======
        scene.background = new THREE.Color('#111111');
        
        // ====== GUI ======
        const gui = new GUI({ title: 'PBR Settings' });
        
        const params = {
            baseColor: '#b0b0b0',
            toneMapping: 'ACES Filmic',
            exposure: 1.2,
        };
        
        gui.addColor(params, 'baseColor').onChange(v => {
            materials.forEach(m => m.color.set(v));
        });
        
        gui.add(params, 'toneMapping', Object.keys({
            'No Tone Mapping': THREE.NoToneMapping,
            'Linear': THREE.LinearToneMapping,
            'Reinhard': THREE.ReinhardToneMapping,
            'Cineon': THREE.CineonToneMapping,
            'ACES Filmic': THREE.ACESFilmicToneMapping,
        })).onChange(v => {
            const map = {
                'No Tone Mapping': THREE.NoToneMapping,
                'Linear': THREE.LinearToneMapping,
                'Reinhard': THREE.ReinhardToneMapping,
                'Cineon': THREE.CineonToneMapping,
                'ACES Filmic': THREE.ACESFilmicToneMapping,
            };
            renderer.toneMapping = map[v];
            scene.traverse(obj => {
                if (obj.material) obj.material.needsUpdate = true;
            });
        });
        
        gui.add(params, 'exposure', 0, 3, 0.01).onChange(v => {
            renderer.toneMappingExposure = v;
        });
        
        // ====== ANIMATION ======
        const clock = new THREE.Clock();
        
        function animate() {
            requestAnimationFrame(animate);
            const elapsed = clock.getElapsedTime();
            
            // Rotate spheres
            sphereMeshes.forEach(mesh => {
                mesh.rotation.y += 0.003;
            });
            
            // Animate lights
            dirLight1.position.x = Math.sin(elapsed * 0.3) * 8;
            dirLight1.position.z = Math.cos(elapsed * 0.3) * 8;
            
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

## สรุป PBR Properties

| Property | Range | ผลลัพธ์ |
|----------|-------|---------|
| metalness 0 | 0 | Non-metal (plastic, wood, stone) |
| metalness 1 | 1 | Metal (steel, gold, copper) |
| roughness 0 | 0 | Mirror-like |
| roughness 1 | 1 | Completely matte |
| clearcoat 1 | 0-1 | Lacquer/varnish layer |
| transmission 1 | 0-1 | Glass/transparent |
| sheen 1 | 0-1 | Fabric/velvet |
| iridescence 1 | 0-1 | Soap bubble/CD |

---

## แบบฝึกหัด (Exercises)

### Exercise 1: Material Ball (ง่าย)
สร้าง sphere ที่สามารถปรับ metalness และ roughness ได้แบบ real-time  
ใช้ dat.GUI ให้ปรับค่า 0-1 แบบ slider

### Exercise 2: Material Library (ปานกลาง)
สร้าง "material library" ที่มี preset ต่างๆ:
- Gold, Silver, Copper, Rubber, Plastic, Wood, Stone
แต่ละ preset มีค่า metalness, roughness, color ที่เหมาะสม

### Exercise 3: PBR Scene (ปานกลาง)
สร้าง scene ที่มีวัตถุหลายชิ้น:
- โต๊ะไม้ (wood material)
- แก้วน้ำ (glass - MeshPhysicalMaterial transmission)
- กล่องโลหะ (metal - metalness 1)
- หนังสือ (plastic matte)
พร้อมแสงที่เหมาะสม

### Exercise 4: Car Paint (ยาก)
สร้าง sphere ที่มี car paint effect:
- Base coat: สีรถ
- Clear coat: เคลือบเงา
- ให้ปรับสีรถ, ความเงาของ clearcoat
- เพิ่ม environment map สำหรับ reflection

### Exercise 5: Glass Scene (ยาก)
สร้าง scene ที่มีวัตถุแก้วหลายชิ้น (sphere, box, torus)  
ด้วย MeshPhysicalMaterial + transmission + ior  
ทดลองค่า ior ต่างๆ (1.0 อากาศ, 1.33 น้ำ, 1.5 แก้ว, 2.4 diamond)

---

**ก่อนหน้า:** [Part 004 - Materials Part 1](part-004.md)  
**ต่อไป:** [Part 006 - Textures Part 1](part-006.md)
