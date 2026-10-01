# Part 006: Textures Part 1 - การโหลดและจัดการ Texture

## สารบัญบทเรียนนี้
- [1. Texture คืออะไร?](#1-texture-คืออะไร)
- [2. TextureLoader](#2-textureloader)
- [3. Loading Events & Callbacks](#3-loading-events--callbacks)
- [4. LoadingManager](#4-loadingmanager)
- [5. Texture Wrapping](#5-texture-wrapping)
- [6. Texture Filtering & Mipmaps](#6-texture-filtering--mipmaps)
- [7. Color Space ของ Texture](#7-color-space-ของ-texture)
- [8. Texture Transform (repeat, offset, rotation, center)](#8-texture-transform)
- [9. UV Mapping Concept](#9-uv-mapping-concept)
- [10. Procedural Textures ด้วย Canvas API](#10-procedural-textures-ด้วย-canvas-api)
- [11. CanvasTexture](#11-canvastexture)
- [12. Complete Demo](#12-complete-demo)
- [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. Texture คืออะไร?

**Texture** คือภาพ 2D ที่ถูก "wrap" หรือ "project" ลงบน surface ของ 3D object  
ทำให้ object มีรายละเอียด, สี, และความเป็นธรรมชาติมากขึ้น

### Texture Types (ประเภทของ Texture)

```
Texture Types:
├── Albedo/Color/Diffuse Map  - สีพื้นผิว
├── Normal Map                - ทิศทาง normal สำหรับ lighting detail
├── Roughness Map             - ความหยาบต่อ pixel
├── Metalness Map             - ความเป็นโลหะต่อ pixel
├── AO Map                    - Ambient Occlusion
├── Displacement/Height Map   - เขยิบ geometry จริงๆ
├── Emissive Map              - พื้นที่ที่ส่องแสงออก
├── Alpha/Opacity Map         - ความโปร่งแสง
├── Environment Map           - สภาพแวดล้อมสำหรับ reflection
└── Matcap                    - Material capture (all-in-one)
```

### รูปแบบไฟล์ที่รองรับ

| Format | ข้อดี | ข้อเสีย | ใช้เมื่อ |
|--------|-------|---------|---------|
| JPG/JPEG | ขนาดเล็ก | lossy, ไม่มี alpha | albedo, environment |
| PNG | lossless, มี alpha | ขนาดใหญ่กว่า | alpha maps, normal maps |
| WebP | ขนาดเล็ก lossless/lossy | ต้องตรวจสอบ browser support | ทั่วไป |
| HDR | HDR values | ขนาดใหญ่ | environment lighting |
| EXR | HDR, lossless | ขนาดใหญ่มาก | VFX, high quality |
| KTX2 | GPU compressed | ต้องแปลงก่อน | production (performance) |
| Basis | GPU compressed | ต้องแปลงก่อน | mobile |

### Power of Two (PoT)

```
⚠️ สำคัญมาก!
Texture ควรมีขนาดเป็น Power of Two เพื่อ mipmapping และ performance ดี

ขนาดที่ถูกต้อง: 1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024, 2048, 4096
ขนาดที่ไม่ถูกต้อง: 300, 500, 800, 1000

ไม่จำเป็นต้องเป็นรูปสี่เหลี่ยมจัตุรัส:
- 512x1024 ✅
- 256x512 ✅  
- 1024x512 ✅
- 300x400 ⚠️ (จะทำงานได้แต่ performance ไม่ดี + เส้นขอบ)
```

---

## 2. TextureLoader

`THREE.TextureLoader` คือ class หลักสำหรับโหลด texture จาก URL

```javascript
// ========== Basic Usage ==========

// สร้าง TextureLoader instance
const textureLoader = new THREE.TextureLoader();

// โหลด texture แบบ async (callback)
const texture = textureLoader.load(
    '/textures/color.jpg',  // URL (relative หรือ absolute)
    (texture) => {          // onLoad callback
        console.log('Loaded!', texture);
        console.log('Width:', texture.image.width);
        console.log('Height:', texture.image.height);
    },
    (event) => {            // onProgress callback (ไม่ค่อยทำงานบน images)
        console.log('Loading...', event.loaded / event.total * 100, '%');
    },
    (error) => {            // onError callback
        console.error('Failed to load texture:', error);
    }
);

// ใช้ texture ใน material ทันที (แม้ยังโหลดไม่เสร็จ)
// Three.js จะ use placeholder จนกว่าจะโหลดเสร็จ
const material = new THREE.MeshStandardMaterial({
    map: texture,
});

// ========== Async/Await ด้วย Promise ==========

function loadTexture(url) {
    return new Promise((resolve, reject) => {
        textureLoader.load(url, resolve, undefined, reject);
    });
}

// ใช้งาน
async function init() {
    try {
        const colorTexture = await loadTexture('/textures/color.jpg');
        const normalTexture = await loadTexture('/textures/normal.jpg');
        
        // ตั้งค่า color space
        colorTexture.colorSpace = THREE.SRGBColorSpace;
        
        const material = new THREE.MeshStandardMaterial({
            map: colorTexture,
            normalMap: normalTexture,
        });
        
        // สร้าง mesh...
    } catch (error) {
        console.error('Error loading textures:', error);
    }
}

// โหลดหลาย textures พร้อมกัน (parallel)
async function loadAllTextures() {
    const [color, normal, roughness, metalness, ao] = await Promise.all([
        loadTexture('/textures/color.jpg'),
        loadTexture('/textures/normal.jpg'),
        loadTexture('/textures/roughness.jpg'),
        loadTexture('/textures/metalness.jpg'),
        loadTexture('/textures/ao.jpg'),
    ]);
    
    // ตั้งค่า color space (เฉพาะ color texture)
    color.colorSpace = THREE.SRGBColorSpace;
    
    return { color, normal, roughness, metalness, ao };
}
```

### Texture Properties

```javascript
const texture = textureLoader.load('/textures/example.jpg');

// ดูข้อมูล texture
console.log(texture.image.width, texture.image.height); // dimensions (after load)
console.log(texture.uuid);        // unique id
console.log(texture.version);     // version number

// Flags
texture.needsUpdate = false;       // set true เมื่อ update image data
texture.generateMipmaps = true;   // สร้าง mipmaps อัตโนมัติ

// Dispose เมื่อไม่ใช้แล้ว (ล้าง GPU memory)
texture.dispose();
```

---

## 3. Loading Events & Callbacks

```javascript
// ========== Three Ways to Handle Loading ==========

// Method 1: Callback
const loader = new THREE.TextureLoader();
const tex = loader.load('/image.jpg', 
    function onLoad(texture) {
        console.log('Done:', texture);
    },
    undefined,  // onProgress (ไม่จำเป็น)
    function onError(err) {
        console.error('Error:', err);
    }
);

// Method 2: Promisify
function loadTextureAsync(url) {
    return new Promise((resolve, reject) => {
        new THREE.TextureLoader().load(url, resolve, undefined, reject);
    });
}

// Method 3: ใช้ Library (เช่น three-stdlib)
// ถ้า project ใหญ่อาจใช้ library ช่วยจัดการ

// ========== Error Handling ที่ดี ==========

async function safeLoadTexture(url, fallbackColor = '#ff00ff') {
    try {
        return await loadTextureAsync(url);
    } catch (e) {
        console.warn(`Failed to load texture: ${url}. Using fallback color.`);
        
        // สร้าง fallback texture (สี solid)
        const canvas = document.createElement('canvas');
        canvas.width = 4;
        canvas.height = 4;
        const ctx = canvas.getContext('2d');
        ctx.fillStyle = fallbackColor;
        ctx.fillRect(0, 0, 4, 4);
        
        return new THREE.CanvasTexture(canvas);
    }
}
```

---

## 4. LoadingManager

`THREE.LoadingManager` ช่วยติดตาม progress ของการโหลด assets หลายอย่างพร้อมกัน

```javascript
// ========== Basic LoadingManager ==========

// สร้าง LoadingManager
const loadingManager = new THREE.LoadingManager();

// Callbacks
loadingManager.onStart = function(url, itemsLoaded, itemsTotal) {
    console.log(`Started loading: ${url}`);
    console.log(`${itemsLoaded} / ${itemsTotal}`);
};

loadingManager.onProgress = function(url, itemsLoaded, itemsTotal) {
    const progress = (itemsLoaded / itemsTotal * 100).toFixed(0);
    console.log(`Loading: ${progress}% (${url})`);
    
    // อัปเดต loading bar
    document.querySelector('#progress').style.width = progress + '%';
};

loadingManager.onLoad = function() {
    console.log('All assets loaded!');
    
    // ซ่อน loading screen
    document.querySelector('#loading-screen').style.display = 'none';
    
    // เริ่ม animation
    animate();
};

loadingManager.onError = function(url) {
    console.error(`Failed to load: ${url}`);
};

// ส่ง loadingManager ให้ loaders ต่างๆ
const textureLoader = new THREE.TextureLoader(loadingManager);
// GLTFLoader, OBJLoader ฯลฯ ก็รับ loadingManager เหมือนกัน

// ========== Loading Screen HTML ==========
/*
<div id="loading-screen" style="
    position: fixed;
    inset: 0;
    background: #0d1117;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    z-index: 999;
">
    <p style="color: #fff; margin-bottom: 20px;">Loading...</p>
    <div style="width: 300px; height: 4px; background: #333; border-radius: 2px;">
        <div id="progress" style="
            width: 0%;
            height: 100%;
            background: #6c63ff;
            border-radius: 2px;
            transition: width 0.3s;
        "></div>
    </div>
</div>
*/

// ========== Complete Loading Setup ==========

const manager = new THREE.LoadingManager(
    // onLoad
    () => {
        console.log('Ready!');
        // เริ่ม app
    },
    // onProgress
    (url, loaded, total) => {
        const p = Math.floor(loaded / total * 100);
        document.querySelector('.progress-bar').style.width = p + '%';
    },
    // onError
    (url) => {
        console.error('Error loading:', url);
    }
);

const tLoader = new THREE.TextureLoader(manager);
// GLTFLoader, AudioLoader, etc. จะ share manager เดียวกัน
```

---

## 5. Texture Wrapping

Wrapping กำหนดว่า texture จะ repeat อย่างไรเมื่อ UV coordinates เกิน [0, 1]

```javascript
// Wrapping Types:

// 1. ClampToEdgeWrapping (default)
// UV ที่เกิน 0-1 จะถูก clamp ไว้ที่ edge
// เหมาะสำหรับ: texture ที่ไม่ต้องการ repeat (เช่น face texture)
texture.wrapS = THREE.ClampToEdgeWrapping; // แนวนอน
texture.wrapT = THREE.ClampToEdgeWrapping; // แนวตั้ง

// ผลลัพธ์ที่ UV=1.5: ยังคงเป็นสีที่ขอบ texture
// ผลลัพธ์ที่ UV=-0.5: ยังคงเป็นสีที่ขอบ (0)

// 2. RepeatWrapping
// UV ที่เกิน 0-1 จะ wrap กลับมาเรื่อยๆ
// เหมาะสำหรับ: tileable texture (พื้น, ผนัง, หญ้า)
texture.wrapS = THREE.RepeatWrapping;
texture.wrapT = THREE.RepeatWrapping;

// ผลลัพธ์ที่ UV=1.5: เหมือน UV=0.5
// ผลลัพธ์ที่ UV=2.3: เหมือน UV=0.3

// 3. MirroredRepeatWrapping
// Repeat แบบกลับซ้ายขวา/บนล่างสลับกัน
// ทำให้ขอบ texture ไม่เห็น seam
texture.wrapS = THREE.MirroredRepeatWrapping;
texture.wrapT = THREE.MirroredRepeatWrapping;

// ========== ตัวอย่างการใช้งาน ==========

// พื้นห้อง (repeat)
const floorTexture = textureLoader.load('/textures/floor_tile.jpg');
floorTexture.wrapS = THREE.RepeatWrapping;
floorTexture.wrapT = THREE.RepeatWrapping;
floorTexture.repeat.set(8, 8); // tile 8x8 ครั้ง

const floorMaterial = new THREE.MeshStandardMaterial({
    map: floorTexture,
});

// Face texture (clamp)
const faceTexture = textureLoader.load('/textures/face.jpg');
faceTexture.wrapS = THREE.ClampToEdgeWrapping;
faceTexture.wrapT = THREE.ClampToEdgeWrapping;

// ผนังบริก (mirrored repeat = ไม่เห็น seam)
const brickTexture = textureLoader.load('/textures/brick.jpg');
brickTexture.wrapS = THREE.MirroredRepeatWrapping;
brickTexture.wrapT = THREE.MirroredRepeatWrapping;
brickTexture.repeat.set(4, 2);

// ⚠️ ต้อง set wrapS/wrapT ก่อน set repeat/offset
// มิเช่นนั้น repeat/offset จะไม่มีผล
```

---

## 6. Texture Filtering & Mipmaps

### Filtering คืออะไร?

เมื่อ texture ถูก display ที่ขนาดต่างจากขนาดจริง GPU ต้องเลือกว่าจะคำนวณสีอย่างไร

```javascript
// ========== Minification Filter (texture ดูเล็กกว่า original) ==========
// ใช้เมื่อ object อยู่ไกล / texture ขนาดใหญ่กว่า pixels ที่จะแสดง

texture.minFilter = THREE.NearestFilter;
// เลือก texel ที่ใกล้ที่สุด
// เร็วที่สุด
// ได้ผลแบบ "pixelated" (เหมาะสำหรับ pixel art style)

texture.minFilter = THREE.LinearFilter;
// interpolate ระหว่าง texels ที่ใกล้ที่สุด 4 ตัว
// นวลกว่า NearestFilter
// แต่อาจเห็น moiré pattern เมื่อ texture เล็กมาก

texture.minFilter = THREE.NearestMipmapNearestFilter;
texture.minFilter = THREE.NearestMipmapLinearFilter;
texture.minFilter = THREE.LinearMipmapNearestFilter;
texture.minFilter = THREE.LinearMipmapLinearFilter; // (default, แนะนำ)
// LinearMipmapLinear = trilinear filtering
// ดีที่สุด: เลือก mipmap ที่เหมาะสมและ interpolate

// ========== Magnification Filter (texture ดูใหญ่กว่า original) ==========
// ใช้เมื่อ object อยู่ใกล้ / texture ขนาดเล็กกว่า pixels ที่จะแสดง

texture.magFilter = THREE.NearestFilter; // pixelated (retro style)
texture.magFilter = THREE.LinearFilter;  // smooth (default)

// ========== Mipmaps ==========
// Mipmap = ชุดของ texture ที่ขนาดเล็กลงเรื่อยๆ (1/2 ทุกครั้ง)
// สร้างขึ้นอัตโนมัติโดย Three.js (ถ้า generateMipmaps = true)

// Original: 1024x1024
// Mip 1: 512x512
// Mip 2: 256x256
// ...
// Mip 10: 1x1

// GPU เลือก mipmap ที่เหมาะสมตามขนาดที่แสดง
// ลด aliasing และเพิ่ม performance

texture.generateMipmaps = true;   // default: true
// false: ไม่สร้าง mipmaps (เหมาะสำหรับ RenderTarget, non-PoT textures)

// ========== Anisotropic Filtering ==========
// ลด blurring เมื่อมอง texture ที่มุมเฉียง

const maxAnisotropy = renderer.capabilities.getMaxAnisotropy();
console.log('Max Anisotropy:', maxAnisotropy); // ปกติ: 16

texture.anisotropy = maxAnisotropy; // ดีที่สุด (แต่ใช้ memory มากขึ้น)
// ค่า: 1 (default), 2, 4, 8, 16

// ตัวอย่าง: พื้น/ผนังที่มอง perspective ต้องการ anisotropy สูง
floorTexture.anisotropy = maxAnisotropy;
```

---

## 7. Color Space ของ Texture

```javascript
// ⚠️ สำคัญมาก! ตั้งค่า color space ให้ถูกต้อง

// ========== Rules ==========

// 1. Textures ที่เป็น "color" ที่มองเห็น (ต้องการ sRGB decoding):
//    - albedo/color map
//    - emissive map
//    - environment map (สำหรับ background)
texture.colorSpace = THREE.SRGBColorSpace;

// 2. Textures ที่เป็น "data" (ไม่ต้องการ color conversion):
//    - normal map
//    - roughness map
//    - metalness map
//    - ao map
//    - displacement map
//    - alpha map
texture.colorSpace = THREE.NoColorSpace; // (default)

// ========== ทำไมถึงสำคัญ? ==========

// สีที่เราเห็นบนจอ (sRGB space) ไม่ใช่ค่าที่ GPU คำนวณ
// GPU ทำงานใน linear space
// sRGB → linear: values จะ "ขยาย" ในส่วนมืด

// ถ้าไม่ตั้ง colorSpace:
// - Color textures: จะดูสว่างผิดปกติ (overcorrected)
// - Normal/data textures: จะทำงานผิดพลาด

// Three.js r152+ ตั้ง renderer.outputColorSpace = THREE.SRGBColorSpace เป็น default
// แต่ texture.colorSpace ต้องตั้งเองสำหรับ color textures

// ========== ตัวอย่างสมบูรณ์ ==========

const loader = new THREE.TextureLoader();

// Color textures (ต้อง SRGBColorSpace)
const colorMap = loader.load('/textures/color.jpg');
colorMap.colorSpace = THREE.SRGBColorSpace;

const emissiveMap = loader.load('/textures/emissive.jpg');
emissiveMap.colorSpace = THREE.SRGBColorSpace;

// Data textures (default NoColorSpace ก็ OK)
const normalMap = loader.load('/textures/normal.jpg');
const roughnessMap = loader.load('/textures/roughness.jpg');
const metalnessMap = loader.load('/textures/metalness.jpg');
const aoMap = loader.load('/textures/ao.jpg');
// ไม่ต้องตั้ง colorSpace = NoColorSpace เพราะเป็น default

const material = new THREE.MeshStandardMaterial({
    map: colorMap,
    normalMap: normalMap,
    roughnessMap: roughnessMap,
    metalnessMap: metalnessMap,
    aoMap: aoMap,
    emissiveMap: emissiveMap,
    emissive: '#ffffff',
});
```

---

## 8. Texture Transform

```javascript
// ========== Repeat ==========
// กี่ครั้งที่ texture จะ tile ใน U และ V direction
// ต้องตั้ง wrapS/wrapT = RepeatWrapping หรือ MirroredRepeatWrapping ก่อน

texture.repeat.set(4, 4); // tile 4x4 ครั้ง
texture.repeat.x = 3;     // tile 3 ครั้งแนวนอน
texture.repeat.y = 2;     // tile 2 ครั้งแนวตั้ง

// ========== Offset ==========
// เลื่อน texture (0-1 = เลื่อนหนึ่งรอบ)
// ต้องตั้ง wrapS/wrapT = RepeatWrapping ก่อน

texture.offset.set(0.5, 0.5); // เลื่อนครึ่งหนึ่งในทั้งสองทิศ
texture.offset.x = 0.25;      // เลื่อนแนวนอน 25%

// ========== Rotation ==========
// หมุน texture รอบจุด center (หน่วย: radian)

texture.rotation = Math.PI / 4; // หมุน 45 องศา
texture.rotation = Math.PI;     // หมุน 180 องศา

// ========== Center (จุดหมุน) ==========
// กำหนดจุดที่ rotation หมุนรอบ (UV space)
// (0, 0) = bottom-left corner (default)
// (0.5, 0.5) = center

texture.center.set(0.5, 0.5); // หมุนรอบกลาง (นิยมที่สุด)

// ========== ตัวอย่างการ Animate Texture ==========

// Scrolling texture (เช่น น้ำ, เมฆ, ทางด่วน)
function animate() {
    requestAnimationFrame(animate);
    const delta = clock.getDelta();
    
    // เลื่อน texture ต่อเนื่อง
    waterTexture.offset.x += delta * 0.1;
    waterTexture.offset.y += delta * 0.05;
    
    renderer.render(scene, camera);
}

// หมุน texture
function animate() {
    requestAnimationFrame(animate);
    const delta = clock.getDelta();
    
    spinTexture.rotation += delta * 0.5; // หมุน 0.5 rad/s
    spinTexture.center.set(0.5, 0.5);    // หมุนรอบกลาง
    
    renderer.render(scene, camera);
}
```

---

## 9. UV Mapping Concept

UV Mapping คือการ map พื้นผิว 3D → พื้นที่ 2D ของ texture

```javascript
// ========== UV Coordinates ==========

// U = แนวนอน (0 = ซ้าย, 1 = ขวา)
// V = แนวตั้ง (0 = ล่าง, 1 = บน) ใน Three.js/OpenGL
// (ระวัง: บางซอฟต์แวร์ V เริ่มจากบน)

// ดู UV ของ BoxGeometry
const boxGeo = new THREE.BoxGeometry(1, 1, 1);
const uvAttr = boxGeo.attributes.uv;
console.log('UV count:', uvAttr.count);

for (let i = 0; i < uvAttr.count; i++) {
    console.log(`UV[${i}]: U=${uvAttr.getX(i).toFixed(2)}, V=${uvAttr.getY(i).toFixed(2)}`);
}

// BoxGeometry UV Layout:
// แต่ละ face มี UV ของตัวเอง (0,0)-(1,1)
// ไม่มีการ share UV กัน

// SphereGeometry UV Layout:
// Equirectangular mapping
// U จาก 0 (ซ้าย/longitude -180) ถึง 1 (ขวา/longitude +180)
// V จาก 0 (ล่าง/latitude -90) ถึง 1 (บน/latitude +90)

// ========== Custom UV Mapping ==========

// กรณีที่ต้องการ UV ต่างจาก default
// ตัวอย่าง: PlaneGeometry ที่ต้องการ UV สำหรับ texture atlas

const customPlane = new THREE.PlaneGeometry(2, 2, 1, 1);
const uvs = customPlane.attributes.uv;

// UV ปกติ: (0,0), (1,0), (0,1), (1,1)
// เปลี่ยนให้ครอบแค่ครึ่งขวาของ texture: (0.5,0), (1,0), (0.5,1), (1,1)
uvs.setXY(0, 0.5, 0); // bottom-left vertex
uvs.setXY(1, 1.0, 0); // bottom-right vertex
uvs.setXY(2, 0.5, 1); // top-left vertex
uvs.setXY(3, 1.0, 1); // top-right vertex
uvs.needsUpdate = true;

// ========== UV Unwrapping ==========

// สำหรับ model ที่ซับซ้อน UV ต้อง unwrap ใน 3D software (เช่น Blender)
// Three.js ไม่มี built-in UV unwrapper

// GLTFLoader จะ import UV ที่ unwrap จาก Blender มาให้อัตโนมัติ
```

---

## 10. Procedural Textures ด้วย Canvas API

เราสามารถสร้าง texture จาก Canvas API ได้โดยไม่ต้องใช้ไฟล์ภาพ

```javascript
// ========== Checkerboard Pattern ==========

function createCheckerboard(size = 256, squares = 8, 
                             color1 = '#ffffff', color2 = '#000000') {
    const canvas = document.createElement('canvas');
    canvas.width = size;
    canvas.height = size;
    const ctx = canvas.getContext('2d');
    
    const squareSize = size / squares;
    
    for (let y = 0; y < squares; y++) {
        for (let x = 0; x < squares; x++) {
            // สลับสีตาม (x+y) เป็น even หรือ odd
            ctx.fillStyle = (x + y) % 2 === 0 ? color1 : color2;
            ctx.fillRect(x * squareSize, y * squareSize, squareSize, squareSize);
        }
    }
    
    return new THREE.CanvasTexture(canvas);
}

// ========== Gradient Texture ==========

function createGradientTexture(colors = ['#ff0000', '#00ff00', '#0000ff']) {
    const canvas = document.createElement('canvas');
    canvas.width = 256;
    canvas.height = 1;
    const ctx = canvas.getContext('2d');
    
    const gradient = ctx.createLinearGradient(0, 0, 256, 0);
    colors.forEach((color, i) => {
        gradient.addColorStop(i / (colors.length - 1), color);
    });
    
    ctx.fillStyle = gradient;
    ctx.fillRect(0, 0, 256, 1);
    
    return new THREE.CanvasTexture(canvas);
}

// ========== Perlin Noise Texture (Simple) ==========

function smoothNoise(x, y, seed = 0) {
    // Simple noise function (ไม่ใช่ Perlin จริงๆ แต่ดูเหมือน)
    const val = Math.sin(x * 12.9898 + y * 78.233 + seed) * 43758.5453;
    return val - Math.floor(val);
}

function interpolate(a, b, t) {
    return a * (1 - t) + b * t;
}

function createNoiseTexture(size = 256, scale = 4) {
    const canvas = document.createElement('canvas');
    canvas.width = size;
    canvas.height = size;
    const ctx = canvas.getContext('2d');
    const imageData = ctx.createImageData(size, size);
    
    for (let y = 0; y < size; y++) {
        for (let x = 0; x < size; x++) {
            const nx = x / size * scale;
            const ny = y / size * scale;
            
            // Simple value noise
            const x0 = Math.floor(nx), y0 = Math.floor(ny);
            const x1 = x0 + 1, y1 = y0 + 1;
            const fx = nx - x0, fy = ny - y0;
            
            // Smooth interpolation (smoothstep)
            const ux = fx * fx * (3 - 2 * fx);
            const uy = fy * fy * (3 - 2 * fy);
            
            const n00 = smoothNoise(x0, y0);
            const n10 = smoothNoise(x1, y0);
            const n01 = smoothNoise(x0, y1);
            const n11 = smoothNoise(x1, y1);
            
            const value = interpolate(
                interpolate(n00, n10, ux),
                interpolate(n01, n11, ux),
                uy
            );
            
            const idx = (y * size + x) * 4;
            const v = Math.floor(value * 255);
            imageData.data[idx + 0] = v;    // R
            imageData.data[idx + 1] = v;    // G
            imageData.data[idx + 2] = v;    // B
            imageData.data[idx + 3] = 255;  // A
        }
    }
    
    ctx.putImageData(imageData, 0, 0);
    return new THREE.CanvasTexture(canvas);
}

// ========== UV Test Pattern ==========

function createUVTestPattern(size = 512) {
    const canvas = document.createElement('canvas');
    canvas.width = size;
    canvas.height = size;
    const ctx = canvas.getContext('2d');
    
    // สีพื้น
    ctx.fillStyle = '#1a1a2e';
    ctx.fillRect(0, 0, size, size);
    
    // Grid
    ctx.strokeStyle = 'rgba(255,255,255,0.2)';
    ctx.lineWidth = 1;
    const gridSize = size / 8;
    for (let i = 0; i <= 8; i++) {
        ctx.beginPath();
        ctx.moveTo(i * gridSize, 0);
        ctx.lineTo(i * gridSize, size);
        ctx.stroke();
        ctx.beginPath();
        ctx.moveTo(0, i * gridSize);
        ctx.lineTo(size, i * gridSize);
        ctx.stroke();
    }
    
    // U gradient (horizontal)
    const uGradient = ctx.createLinearGradient(0, 0, size, 0);
    uGradient.addColorStop(0, 'rgba(255,0,0,0.3)');
    uGradient.addColorStop(1, 'rgba(255,0,0,0)');
    ctx.fillStyle = uGradient;
    ctx.fillRect(0, 0, size, size / 2);
    
    // V gradient (vertical)
    const vGradient = ctx.createLinearGradient(0, 0, 0, size);
    vGradient.addColorStop(0, 'rgba(0,255,0,0)');
    vGradient.addColorStop(1, 'rgba(0,255,0,0.3)');
    ctx.fillStyle = vGradient;
    ctx.fillRect(0, size / 2, size, size / 2);
    
    // Center dot
    ctx.fillStyle = '#6c63ff';
    ctx.beginPath();
    ctx.arc(size / 2, size / 2, 10, 0, Math.PI * 2);
    ctx.fill();
    
    // Labels
    ctx.fillStyle = '#ffffff';
    ctx.font = `${size / 16}px monospace`;
    ctx.textAlign = 'left';
    ctx.fillText('U=0,V=0', 10, size - 10);
    ctx.textAlign = 'right';
    ctx.fillText('U=1,V=0', size - 10, size - 10);
    ctx.textAlign = 'left';
    ctx.fillText('U=0,V=1', 10, size / 16 + 5);
    ctx.textAlign = 'right';
    ctx.fillText('U=1,V=1', size - 10, size / 16 + 5);
    
    const texture = new THREE.CanvasTexture(canvas);
    texture.colorSpace = THREE.SRGBColorSpace;
    return texture;
}
```

---

## 11. CanvasTexture

```javascript
// ========== CanvasTexture - Live Update ==========

// CanvasTexture ให้เรา update canvas และ texture จะอัปเดตด้วย

const canvas = document.createElement('canvas');
canvas.width = 512;
canvas.height = 512;
const ctx = canvas.getContext('2d');

// สร้าง CanvasTexture
const dynamicTexture = new THREE.CanvasTexture(canvas);
dynamicTexture.colorSpace = THREE.SRGBColorSpace;

const material = new THREE.MeshBasicMaterial({ map: dynamicTexture });

// อัปเดต canvas ใน animation loop
function drawOnCanvas(elapsed) {
    // Clear
    ctx.fillStyle = '#0d1117';
    ctx.fillRect(0, 0, 512, 512);
    
    // วาด clock
    const cx = 256, cy = 256, r = 200;
    
    // Circle
    ctx.strokeStyle = '#6c63ff';
    ctx.lineWidth = 4;
    ctx.beginPath();
    ctx.arc(cx, cy, r, 0, Math.PI * 2);
    ctx.stroke();
    
    // Second hand
    const seconds = elapsed % 60;
    const secAngle = (seconds / 60) * Math.PI * 2 - Math.PI / 2;
    ctx.strokeStyle = '#ff6b6b';
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.moveTo(cx, cy);
    ctx.lineTo(cx + Math.cos(secAngle) * (r * 0.8), cy + Math.sin(secAngle) * (r * 0.8));
    ctx.stroke();
    
    // Minute hand
    const minutes = (elapsed / 60) % 60;
    const minAngle = (minutes / 60) * Math.PI * 2 - Math.PI / 2;
    ctx.strokeStyle = '#ffffff';
    ctx.lineWidth = 4;
    ctx.beginPath();
    ctx.moveTo(cx, cy);
    ctx.lineTo(cx + Math.cos(minAngle) * (r * 0.65), cy + Math.sin(minAngle) * (r * 0.65));
    ctx.stroke();
    
    // บอก Three.js ว่า texture ต้อง update
    dynamicTexture.needsUpdate = true;
}

function animate() {
    requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    
    drawOnCanvas(elapsed);
    renderer.render(scene, camera);
}
```

---

## 12. Complete Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Part 006: Textures Demo</title>
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
        camera.position.set(0, 2, 8);
        
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(innerWidth, innerHeight);
        renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
        renderer.toneMapping = THREE.ACESFilmicToneMapping;
        document.body.appendChild(renderer.domElement);
        
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        
        // ====== LIGHTS ======
        const ambientLight = new THREE.AmbientLight(0xffffff, 1);
        scene.add(ambientLight);
        
        const dirLight = new THREE.DirectionalLight(0xffffff, 2);
        dirLight.position.set(5, 5, 5);
        scene.add(dirLight);
        
        // ====== PROCEDURAL TEXTURES ======
        
        // Checkerboard
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
        
        // Gradient
        function createGradient(w, h, stops) {
            const cv = document.createElement('canvas');
            cv.width = w; cv.height = h;
            const ctx = cv.getContext('2d');
            const g = ctx.createLinearGradient(0, 0, w, 0);
            stops.forEach(([pos, color]) => g.addColorStop(pos, color));
            ctx.fillStyle = g;
            ctx.fillRect(0, 0, w, h);
            const t = new THREE.CanvasTexture(cv);
            t.colorSpace = THREE.SRGBColorSpace;
            return t;
        }
        
        // Noise
        function createNoiseTexture(size, scale) {
            const cv = document.createElement('canvas');
            cv.width = cv.height = size;
            const ctx = cv.getContext('2d');
            const img = ctx.createImageData(size, size);
            for (let y = 0; y < size; y++) {
                for (let x = 0; x < size; x++) {
                    const v = Math.random() * 255;
                    const i = (y * size + x) * 4;
                    img.data[i] = img.data[i+1] = img.data[i+2] = v;
                    img.data[i+3] = 255;
                }
            }
            ctx.putImageData(img, 0, 0);
            return new THREE.CanvasTexture(cv);
        }
        
        // UV Test
        function createUVTest(size) {
            const cv = document.createElement('canvas');
            cv.width = cv.height = size;
            const ctx = cv.getContext('2d');
            
            // Quadrants
            const colors = ['#ff6b6b', '#ffd93d', '#6bcb77', '#4d96ff'];
            const half = size / 2;
            colors.forEach((c, i) => {
                ctx.fillStyle = c;
                ctx.fillRect((i % 2) * half, Math.floor(i / 2) * half, half, half);
            });
            
            // Grid overlay
            ctx.strokeStyle = 'rgba(255,255,255,0.4)';
            ctx.lineWidth = 1;
            for (let i = 1; i < 8; i++) {
                const p = (i / 8) * size;
                ctx.beginPath();
                ctx.moveTo(p, 0); ctx.lineTo(p, size); ctx.stroke();
                ctx.beginPath();
                ctx.moveTo(0, p); ctx.lineTo(size, p); ctx.stroke();
            }
            
            // Labels
            ctx.fillStyle = '#fff';
            ctx.font = `${size/12}px monospace`;
            ctx.textAlign = 'center';
            ctx.fillText('0,1', half/2, half*0.15);
            ctx.fillText('1,1', half+half/2, half*0.15);
            ctx.fillText('0,0', half/2, half+half*0.85);
            ctx.fillText('1,0', half+half/2, half+half*0.85);
            
            const t = new THREE.CanvasTexture(cv);
            t.colorSpace = THREE.SRGBColorSpace;
            return t;
        }
        
        // Dynamic clock canvas
        const clockCanvas = document.createElement('canvas');
        clockCanvas.width = clockCanvas.height = 512;
        const clockCtx = clockCanvas.getContext('2d');
        const clockTexture = new THREE.CanvasTexture(clockCanvas);
        clockTexture.colorSpace = THREE.SRGBColorSpace;
        
        function drawClock(elapsed) {
            const size = 512, cx = 256, cy = 256, r = 220;
            clockCtx.fillStyle = '#0d1117';
            clockCtx.fillRect(0, 0, size, size);
            
            // Circle
            clockCtx.strokeStyle = '#6c63ff';
            clockCtx.lineWidth = 5;
            clockCtx.beginPath();
            clockCtx.arc(cx, cy, r, 0, Math.PI*2);
            clockCtx.stroke();
            
            // Ticks
            for (let i = 0; i < 60; i++) {
                const a = (i / 60) * Math.PI * 2 - Math.PI / 2;
                const inner = i % 5 === 0 ? r - 25 : r - 12;
                clockCtx.strokeStyle = i % 5 === 0 ? '#6c63ff' : '#444';
                clockCtx.lineWidth = i % 5 === 0 ? 3 : 1;
                clockCtx.beginPath();
                clockCtx.moveTo(cx + Math.cos(a)*inner, cy + Math.sin(a)*inner);
                clockCtx.lineTo(cx + Math.cos(a)*(r-2), cy + Math.sin(a)*(r-2));
                clockCtx.stroke();
            }
            
            // Hands
            const drawHand = (angle, length, color, width) => {
                clockCtx.strokeStyle = color;
                clockCtx.lineWidth = width;
                clockCtx.lineCap = 'round';
                clockCtx.beginPath();
                clockCtx.moveTo(cx, cy);
                clockCtx.lineTo(cx + Math.cos(angle)*length, cy + Math.sin(angle)*length);
                clockCtx.stroke();
            };
            
            const sec = elapsed % 60;
            const min = (elapsed / 60) % 60;
            const hr = (elapsed / 3600) % 12;
            
            drawHand((hr/12)*Math.PI*2 - Math.PI/2, r*0.55, '#ffffff', 6);
            drawHand((min/60)*Math.PI*2 - Math.PI/2, r*0.7, '#cccccc', 4);
            drawHand((sec/60)*Math.PI*2 - Math.PI/2, r*0.85, '#ff6b6b', 2);
            
            // Center dot
            clockCtx.fillStyle = '#6c63ff';
            clockCtx.beginPath();
            clockCtx.arc(cx, cy, 8, 0, Math.PI*2);
            clockCtx.fill();
            
            clockTexture.needsUpdate = true;
        }
        
        // ====== OBJECTS ======
        
        const geo = new THREE.BoxGeometry(2, 2, 2);
        const sphereGeo = new THREE.SphereGeometry(1, 32, 32);
        
        // Row 1: Texture types
        const textureObjects = [
            { geo, mat: new THREE.MeshBasicMaterial({ map: createCheckerboard(256, 8, '#fff', '#000') }), x: -6 },
            { geo, mat: new THREE.MeshBasicMaterial({ map: createCheckerboard(256, 8, '#6c63ff', '#1a1a2e') }), x: -2 },
            { geo, mat: new THREE.MeshBasicMaterial({ map: createUVTest(512) }), x: 2 },
            { geo, mat: new THREE.MeshBasicMaterial({ map: createGradient(256, 1, [[0,'#ff6b6b'],[0.5,'#ffd93d'],[1,'#6bcb77']]) }), x: 6 },
        ];
        
        textureObjects.forEach(({ geo, mat, x }) => {
            const mesh = new THREE.Mesh(geo, mat);
            mesh.position.x = x;
            mesh.position.y = 2;
            scene.add(mesh);
        });
        
        // Clock sphere (bottom)
        const clockMesh = new THREE.Mesh(
            new THREE.SphereGeometry(1.2, 32, 32),
            new THREE.MeshBasicMaterial({ map: clockTexture })
        );
        clockMesh.position.set(0, -1.5, 0);
        scene.add(clockMesh);
        
        // Ground
        const ground = new THREE.Mesh(
            new THREE.PlaneGeometry(30, 30),
            new THREE.MeshBasicMaterial({ 
                map: createCheckerboard(256, 16, '#222', '#1a1a1a'),
                side: THREE.DoubleSide
            })
        );
        ground.rotation.x = -Math.PI / 2;
        ground.position.y = -3;
        scene.add(ground);
        
        // ====== GUI ======
        const gui = new GUI({ title: 'Texture Settings' });
        
        const params = {
            repeatX: 1,
            repeatY: 1,
            offsetX: 0,
            offsetY: 0,
            rotation: 0,
        };
        
        // Apply to first cube's texture
        const firstTex = textureObjects[0].mat.map;
        firstTex.wrapS = firstTex.wrapT = THREE.RepeatWrapping;
        
        gui.add(params, 'repeatX', 0.1, 8, 0.1).onChange(v => {
            firstTex.repeat.x = v;
        }).name('Repeat X (cube 1)');
        
        gui.add(params, 'repeatY', 0.1, 8, 0.1).onChange(v => {
            firstTex.repeat.y = v;
        }).name('Repeat Y (cube 1)');
        
        gui.add(params, 'offsetX', 0, 1, 0.01).onChange(v => {
            firstTex.offset.x = v;
        }).name('Offset X');
        
        gui.add(params, 'offsetY', 0, 1, 0.01).onChange(v => {
            firstTex.offset.y = v;
        }).name('Offset Y');
        
        gui.add(params, 'rotation', 0, Math.PI * 2, 0.01).onChange(v => {
            firstTex.rotation = v;
            firstTex.center.set(0.5, 0.5);
        }).name('Rotation');
        
        // ====== ANIMATION ======
        const clock = new THREE.Clock();
        
        function animate() {
            requestAnimationFrame(animate);
            const elapsed = clock.getElapsedTime();
            const delta = clock.getDelta();
            
            // Rotate cubes
            textureObjects.forEach(({ }, i) => {
                const obj = scene.children.find(c => c.position.x === textureObjects[i].x && c.position.y === 2);
                if (obj) obj.rotation.y += 0.005;
            });
            
            // Draw clock
            drawClock(elapsed);
            
            // Rotate clock sphere
            clockMesh.rotation.y += 0.003;
            
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

## สรุป Texture Cheatsheet

```javascript
// Template สำหรับ setup texture ที่ถูกต้อง

const loader = new THREE.TextureLoader();

function setupColorTexture(url) {
    const t = loader.load(url);
    t.colorSpace = THREE.SRGBColorSpace;        // สำคัญ!
    t.wrapS = t.wrapT = THREE.RepeatWrapping;
    t.repeat.set(1, 1);
    t.anisotropy = renderer.capabilities.getMaxAnisotropy();
    return t;
}

function setupDataTexture(url) {
    const t = loader.load(url);
    // ไม่ต้อง set colorSpace (default NoColorSpace)
    t.wrapS = t.wrapT = THREE.RepeatWrapping;
    return t;
}
```

---

## แบบฝึกหัด (Exercises)

### Exercise 1: Procedural Checkerboard (ง่าย)
สร้าง checkerboard texture ด้วย Canvas API  
ให้ปรับ square count, color1, color2 ได้ผ่าน dat.GUI

### Exercise 2: Scrolling Texture (ง่าย-ปานกลาง)
สร้างแถบถนน (plane) ที่มี texture เลื่อนไปข้างหน้าเรื่อยๆ  
เหมือนรถกำลังวิ่ง

### Exercise 3: Dynamic Canvas Clock (ปานกลาง)
สร้าง sphere ที่มี live clock วาดด้วย Canvas API  
แสดงนาฬิกาที่เดินตาม real-time (ไม่ใช่ elapsed time)  
`new Date().getHours()`, `.getMinutes()`, `.getSeconds()`

### Exercise 4: UV Debugging (ปานกลาง)
สร้าง UV test pattern แล้วนำไปใช้กับ geometry แต่ละชนิด:
- BoxGeometry
- SphereGeometry
- TorusGeometry
- CylinderGeometry
สังเกตว่า UV unwrapping ต่างกันอย่างไร

### Exercise 5: Texture Atlas (ยาก)
สร้าง texture atlas (รูปเดียวที่มีหลาย sub-texture)  
แล้วใช้ UV offset/repeat เพื่อเลือก sub-texture ที่ต้องการ  
ตัวอย่าง: sprite sheet 4x4 ของ emoji หรือ icons

```javascript
// Hint: สมมติ atlas เป็น 4x4 grid
// เลือก sprite ที่ row=1, col=2:
texture.repeat.set(1/4, 1/4);  // tile หนึ่งส่วนสี่
texture.offset.set(2/4, 1/4);  // เลื่อนไปยัง cell ที่ต้องการ
```

---

**ก่อนหน้า:** [Part 005 - Materials Part 2 (PBR)](part-005.md)  
**ต่อไป:** [Part 007 - Textures Part 2 (PBR Texture Maps)](part-007.md)
