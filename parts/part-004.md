# Part 004: Materials Part 1 - วัสดุพื้นฐาน (Basic Materials)

## สารบัญบทเรียนนี้
- [1. Material คืออะไร?](#1-material-คืออะไร)
- [2. Material Properties พื้นฐาน](#2-material-properties-พื้นฐาน)
- [3. Color Formats ใน Three.js](#3-color-formats-ใน-threejs)
- [4. MeshBasicMaterial](#4-meshbasicmaterial)
- [5. MeshNormalMaterial](#5-meshnormalmaterial)
- [6. MeshDepthMaterial](#6-meshdepthmaterial)
- [7. MeshMatcapMaterial](#7-meshmatchcapmaterial)
- [8. MeshLambertMaterial](#8-meshlambertmaterial)
- [9. MeshPhongMaterial](#9-meshphongmaterial)
- [10. Material Side Options](#10-material-side-options)
- [11. Blending Modes](#11-blending-modes)
- [12. Complete Demo with dat.GUI](#12-complete-demo-with-datgui)
- [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. Material คืออะไร?

**Material** กำหนดว่า surface ของ object จะดูเป็นอย่างไร ซึ่งรวมถึง:
- สี (color)
- การสะท้อนแสง (reflectivity)
- ความโปร่งแสง (transparency)
- การรับ/ทอดเงา (shadows)
- texture mapping
- ลักษณะพิเศษต่างๆ

### Material ใน Three.js มีหลายระดับ

```
ระดับความซับซ้อนจากน้อยไปมาก:
1. MeshBasicMaterial      - ไม่มีแสง เร็วที่สุด
2. MeshNormalMaterial     - แสดง normal vectors เป็นสี
3. MeshDepthMaterial      - แสดงระยะห่างจาก camera
4. MeshMatcapMaterial     - ใช้ matcap texture
5. MeshLambertMaterial    - แสงแบบ Lambertian (diffuse only)
6. MeshPhongMaterial      - แสงแบบ Phong (diffuse + specular)
7. MeshStandardMaterial   - PBR (Part 005)
8. MeshPhysicalMaterial   - PBR ขั้นสูง (Part 005)
9. ShaderMaterial         - Custom shader (Part 052)
10. RawShaderMaterial     - Raw shader (Part 059)
```

---

## 2. Material Properties พื้นฐาน

Properties เหล่านี้มีอยู่ใน **ทุก Material** ใน Three.js:

```javascript
const material = new THREE.MeshBasicMaterial();

// ========== Visibility ==========

material.visible = true;
// true: แสดง object
// false: ซ่อน object (ยังใช้ memory อยู่ แต่ไม่ render)
// เทียบเท่ากับ mesh.visible = false

material.opacity = 1.0;
// 0.0: โปร่งใสสมบูรณ์
// 1.0: ทึบสมบูรณ์
// ต้องตั้ง transparent = true ก่อน

material.transparent = false;
// false: ไม่โปร่งแสง (opacity = 1.0 เสมอ)
// true: เปิดการโปร่งแสง (ใช้ opacity ได้)

material.alphaTest = 0;
// fragment ที่มี alpha < alphaTest จะไม่ render
// ใช้สำหรับ hard-cutout transparency (เช่น ต้นไม้, ใบหญ้า)
// ค่า 0.5 หมายความว่า alpha < 0.5 จะหายไป

material.alphaToCoverage = false;
// ใช้ MSAA alpha coverage (anti-aliased cutout transparency)

// ========== Depth & Stencil ==========

material.depthTest = true;
// true: ทดสอบว่า fragment นี้อยู่หน้าหรือหลัง fragments อื่น
// false: ไม่สนใจ depth (เช่น HUD elements)

material.depthWrite = true;
// true: เขียนค่า depth ลง depth buffer
// false: ไม่เขียน (ใช้สำหรับ transparent objects)

material.depthFunc = THREE.LessEqualDepth;
// เงื่อนไขการ depth test
// THREE.AlwaysDepth, THREE.NeverDepth, THREE.LessDepth,
// THREE.LessEqualDepth (default), THREE.GreaterDepth, etc.

// ========== Sides ==========

material.side = THREE.FrontSide;
// THREE.FrontSide:  render ด้านหน้า (front face) เท่านั้น (default)
// THREE.BackSide:   render ด้านหลัง (back face) เท่านั้น
// THREE.DoubleSide: render ทั้งสองด้าน

material.shadowSide = null;
// กำหนด side สำหรับ shadow rendering
// null: ใช้ค่า side เดียวกับ material

// ========== Color ==========

material.color = new THREE.Color(0xffffff);
// สีหลักของ material (ไม่ใช่ทุก material มี property นี้)

material.vertexColors = false;
// false: ใช้ material.color
// true: ใช้สีจาก geometry.attributes.color

// ========== Misc ==========

material.wireframe = false;
// แสดงเป็น wireframe

material.fog = true;
// true: ได้รับผลจาก scene.fog
// false: ไม่ได้รับผลจาก fog

material.flatShading = false;
// true: shading แบบ flat (แต่ละ polygon มีสีเดียวกัน)
// false: smooth shading (interpolate across polygon)

// ========== Clipping ==========

material.clippingPlanes = [];
// กำหนด clipping planes พิเศษสำหรับ material นี้

material.clipIntersection = false;
// false: clip ทุก plane
// true: clip เฉพาะส่วนที่อยู่นอกทุก plane

material.clipShadows = false;
// true: clipping planes มีผลกับ shadows ด้วย

// ========== Miscellaneous ==========

material.precision = null;
// กำหนด precision ของ shader
// null: ใช้ renderer default
// 'highp' | 'mediump' | 'lowp'

material.toneMapped = true;
// true: ใช้ renderer tone mapping
// false: ไม่ใช้ (เหมาะสำหรับ emissive objects ที่ต้องการสีสว่างมาก)

material.dithering = false;
// เปิด dithering เพื่อลด color banding

// ========== Update ==========

material.needsUpdate = false;
// set = true เมื่อเปลี่ยน properties ที่ต้องการ recompile shader
// เช่น เปลี่ยน transparent, side, fog, vertexColors

// Dispose (cleanup)
material.dispose();
```

---

## 3. Color Formats ใน Three.js

```javascript
// ============================================================
// THREE.Color - Class หลักสำหรับสีใน Three.js
// ============================================================

// สร้าง Color object
const color = new THREE.Color();

// ========== การกำหนดสีด้วยวิธีต่างๆ ==========

// 1. Hex number (ที่ใช้บ่อยที่สุด)
const red = new THREE.Color(0xff0000);
const green = new THREE.Color(0x00ff00);
const blue = new THREE.Color(0x0000ff);

// 2. Hex string
const redStr = new THREE.Color('#ff0000');
const shortHex = new THREE.Color('#f00'); // shorthand

// 3. CSS color name
const white = new THREE.Color('white');
const black = new THREE.Color('black');
const rebeccapurple = new THREE.Color('rebeccapurple');

// 4. RGB string
const rgbColor = new THREE.Color('rgb(255, 128, 0)');

// 5. HSL string  
const hslColor = new THREE.Color('hsl(120, 100%, 50%)');

// 6. RGB values (0-1)
const rgbNorm = new THREE.Color(1, 0.5, 0); // orange

// ========== การเปลี่ยนสี ==========

const c = new THREE.Color();

c.set(0xff0000);          // hex number
c.set('#ff0000');         // hex string
c.set('red');             // css name

c.setHex(0xff0000);       // hex number
c.setStyle('#ff0000');    // css string
c.setRGB(1, 0, 0);        // RGB (0-1)
c.setHSL(0, 1, 0.5);      // HSL (0-1 each)

// ========== การอ่านสี ==========

console.log(c.r, c.g, c.b);    // RGB components (0-1)
console.log(c.getHex());        // hex number: 16711680
console.log(c.getHexString());  // hex string: 'ff0000'
console.log(c.getStyle());      // css string: 'rgb(255,0,0)'

const hsl = {};
c.getHSL(hsl);
console.log(hsl.h, hsl.s, hsl.l);

// ========== Color Math ==========

const c1 = new THREE.Color(0xff0000); // แดง
const c2 = new THREE.Color(0x0000ff); // น้ำเงิน

// ผสมสี
const mixed = c1.clone().lerp(c2, 0.5); // ม่วง (50%)

// Multiply
c1.multiply(c2); // คูณสีกัน

// Add
c1.add(c2);

// ========== Color Space ==========

// Three.js r152+ ใช้ sRGB color space โดย default
// สีที่กำหนดด้วย hex/css จะเป็น sRGB

// เมื่อใช้กับ material.color หรือ scene.background
// Three.js จะแปลง sRGB → Linear ให้อัตโนมัติในกรณีส่วนใหญ่

// ========== ตัวอย่างในการใช้งานจริง ==========

// Random color
function randomColor() {
    return new THREE.Color(Math.random(), Math.random(), Math.random());
}

// Color from temperature (0=blue, 1=red)
function temperatureColor(t) {
    const cold = new THREE.Color(0x4444ff);
    const hot = new THREE.Color(0xff4444);
    return cold.clone().lerp(hot, t);
}

// Rainbow
function rainbowColor(t) {
    const color = new THREE.Color();
    color.setHSL(t, 1, 0.5);
    return color;
}
```

---

## 4. MeshBasicMaterial

Material ที่ง่ายที่สุด - **ไม่รับผลจากแสง**

```javascript
// ใช้เมื่อ:
// - Debug: เห็น shape ชัดเจน ไม่มีแสงรบกวน
// - Background objects
// - HUD/UI elements
// - Objects ที่ต้องการสีคงที่

const basic = new THREE.MeshBasicMaterial({
    // === สี ===
    color: 0x6c63ff,
    
    // === Texture ===
    map: null,              // Color texture (albedo)
    
    // === Transparency ===
    transparent: false,
    opacity: 1.0,
    alphaMap: null,         // Alpha texture
    
    // === Display ===
    wireframe: false,
    wireframeLinewidth: 1,
    wireframeLinecap: 'round',  // 'butt', 'round', 'square'
    wireframeLinejoin: 'round', // 'round', 'bevel', 'miter'
    
    // === Sides ===
    side: THREE.FrontSide,
    
    // === Special ===
    vertexColors: false,    // ใช้สีจาก vertex attributes
    fog: true,              // ได้รับผล fog
});

// ตัวอย่างการใช้ MeshBasicMaterial กับ texture
const loader = new THREE.TextureLoader();
const texture = loader.load('/textures/color.jpg');

const materialWithTexture = new THREE.MeshBasicMaterial({
    map: texture,
    side: THREE.DoubleSide
});

// MeshBasicMaterial เป็น default ของ Helpers
// AxesHelper, GridHelper ใช้ LineBasicMaterial
// แต่สำหรับ debug mesh ใช้ MeshBasicMaterial ดีที่สุด

// ========== Vertex Colors ==========
// สร้าง geometry ที่มีสีแต่ละ vertex

const geometry = new THREE.BufferGeometry();
const positions = new Float32Array([
    0,  1, 0,   // top
   -1, -1, 0,   // bottom-left
    1, -1, 0,   // bottom-right
]);

// สี RGB (0-1) สำหรับแต่ละ vertex
const colors = new Float32Array([
    1, 0, 0,  // แดง
    0, 1, 0,  // เขียว
    0, 0, 1,  // น้ำเงิน
]);

geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));

const vertexColorMat = new THREE.MeshBasicMaterial({ vertexColors: true });
const colorTriangle = new THREE.Mesh(geometry, vertexColorMat);
```

---

## 5. MeshNormalMaterial

แสดง **normal vectors** เป็นสี (มีประโยชน์สำหรับ debug และ visual effects)

```javascript
// Normal direction → สี:
// +X (ขวา)  = แดง
// -X (ซ้าย) = ฟ้าเข้ม
// +Y (บน)   = เขียว
// -Y (ล่าง) = ม่วง
// +Z (หน้า) = ฟ้าอมเขียว
// -Z (หลัง) = น้ำตาล

const normalMat = new THREE.MeshNormalMaterial({
    // Properties พิเศษของ NormalMaterial
    flatShading: false,    // true = เห็น facets ชัด
    wireframe: false,
    opacity: 1.0,
    transparent: false,
});

// MeshNormalMaterial ไม่ได้รับผลจากแสง
// สีขึ้นกับ orientation ของ face เทียบกับ camera

// ⚠️ ข้อสังเกต:
// เมื่อหมุน mesh สีจะเปลี่ยนตาม orientation ของ faces
// เพราะ normals ถูก transform ไปด้วย

// ใช้สำหรับ:
// 1. Debug normal vectors
// 2. สร้าง normal map (baking)
// 3. Visual effect แบบ iridescent/holographic

// ตัวอย่าง: สร้าง normal material sphere
const sphere = new THREE.Mesh(
    new THREE.SphereGeometry(1, 32, 16),
    new THREE.MeshNormalMaterial({ flatShading: true })
);
```

---

## 6. MeshDepthMaterial

แสดง **ระยะห่างจาก camera** เป็น shade ของสีขาว-ดำ  
ใกล้ = ขาว, ไกล = ดำ

```javascript
// MeshDepthMaterial ใช้สำหรับ:
// 1. Debug depth buffer
// 2. Post-processing (Depth of Field, SSAO)
// 3. Visual effects

const depthMat = new THREE.MeshDepthMaterial({
    depthPacking: THREE.BasicDepthPacking,
    // THREE.BasicDepthPacking: depth เป็น linear (0-1)
    // THREE.RGBADepthPacking: encode depth เป็น RGBA (ความแม่นยำสูงกว่า)
    
    displacementMap: null,    // displacement texture
    displacementScale: 1,
    displacementBias: 0,
    
    wireframe: false,
    opacity: 1.0,
    transparent: false,
});

// Near/Far ของ camera มีผลกับ depth range
// สิ่งที่อยู่ที่ near plane จะขาว
// สิ่งที่อยู่ที่ far plane จะดำ

// ตัวอย่าง: render depth pass
const depthRenderTarget = new THREE.WebGLRenderTarget(
    window.innerWidth, 
    window.innerHeight
);
renderer.setRenderTarget(depthRenderTarget);
scene.overrideMaterial = new THREE.MeshDepthMaterial();
renderer.render(scene, camera);
scene.overrideMaterial = null;
renderer.setRenderTarget(null);
// depthRenderTarget.texture ตอนนี้มี depth image

// ⚠️ depthPacking: THREE.RGBADepthPacking ให้ precision สูงกว่า
// เพราะแทนที่จะใช้ 8 bits (1 channel)
// ใช้ 32 bits (4 channels: RGBA)
```

---

## 7. MeshMatcapMaterial

ใช้ **MatCap texture** (Material Capture) ในการ shade  
MatCap คือ sphere texture ที่ "capture" lighting condition ไว้

```javascript
// MatCap texture ดูเหมือนลูกกลมที่ถ่ายรูปไว้
// Three.js จะ map texture นี้ตาม view-space normals

// โหลด matcap texture
const textureLoader = new THREE.TextureLoader();
const matcapTexture = textureLoader.load('/textures/matcap/matcap-01.png');
matcapTexture.colorSpace = THREE.SRGBColorSpace;

const matcapMat = new THREE.MeshMatcapMaterial({
    matcap: matcapTexture,
    
    // Properties เพิ่มเติม
    map: null,           // albedo map (คูณกับ matcap)
    normalMap: null,     // normal map
    normalMapType: THREE.TangentSpaceNormalMap,
    bumpMap: null,
    bumpScale: 1,
    displacementMap: null,
    displacementScale: 1,
    alphaMap: null,
    
    flatShading: false,
    wireframe: false,
    transparent: false,
    opacity: 1.0,
    side: THREE.FrontSide,
});

// ข้อดีของ MatCap:
// 1. ให้ visual ที่สวยงามโดยไม่ต้องมีแสงในฉาก
// 2. Performance ดีมาก (ไม่คำนวณ lighting)
// 3. ดูเหมือนมี lighting แต่ speed เหมือน MeshBasicMaterial

// ข้อเสีย:
// 1. ไม่ responsive ต่อแสงในฉาก (เพราะ capture ไว้แล้ว)
// 2. ดูไม่ถูกต้องเมื่อมอง object จากมุมต่างๆ มาก

// แหล่ง MatCap:
// https://github.com/nidorx/matcaps
// https://www.kchapelier.com/matcap-studio/

// สร้าง matcap ด้วย Canvas API
function createProceduralMatcap() {
    const size = 256;
    const canvas = document.createElement('canvas');
    canvas.width = size;
    canvas.height = size;
    const ctx = canvas.getContext('2d');
    
    // วาด gradient sphere
    const gradient = ctx.createRadialGradient(
        size * 0.35, size * 0.35, 0,  // highlight center
        size * 0.5, size * 0.5, size * 0.5  // sphere edge
    );
    gradient.addColorStop(0, '#ffffff');
    gradient.addColorStop(0.3, '#6c63ff');
    gradient.addColorStop(0.7, '#1a1a2e');
    gradient.addColorStop(1, '#000000');
    
    // วาดวงกลม
    ctx.beginPath();
    ctx.arc(size/2, size/2, size/2, 0, Math.PI * 2);
    ctx.fillStyle = gradient;
    ctx.fill();
    
    const texture = new THREE.CanvasTexture(canvas);
    texture.colorSpace = THREE.SRGBColorSpace;
    return texture;
}
```

---

## 8. MeshLambertMaterial

Material ที่ **รับผลจากแสง** (diffuse shading)  
ใช้ Lambertian reflectance model - แสงสะท้อนเท่ากันในทุกทิศทาง

```javascript
// ⚠️ ต้องมีแสง (Light) ในฉากด้วย!
// ถ้าไม่มีแสง object จะดำ

// สำหรับ AmbientLight + DirectionalLight:
const ambientLight = new THREE.AmbientLight(0xffffff, 0.4);
const dirLight = new THREE.DirectionalLight(0xffffff, 1);
scene.add(ambientLight, dirLight);

const lambertMat = new THREE.MeshLambertMaterial({
    // === สี ===
    color: 0x6c63ff,        // diffuse color
    emissive: 0x000000,     // emissive color (ไม่ขึ้นกับแสง)
    emissiveIntensity: 1.0, // ความเข้มของ emissive
    
    // === Texture Maps ===
    map: null,              // diffuse/albedo texture
    emissiveMap: null,      // emissive texture
    alphaMap: null,
    envMap: null,           // environment map (reflection)
    
    // === Reflection ===
    reflectivity: 1,        // ความสะท้อนของ envMap
    combine: THREE.MultiplyOperation, // วิธีผสม envMap กับ color
    // THREE.MultiplyOperation (default)
    // THREE.MixOperation (ผสมตาม reflectivity)
    // THREE.AddOperation (บวกกัน)
    
    // === Display ===
    wireframe: false,
    flatShading: false,
    
    // === Transparency ===
    transparent: false,
    opacity: 1.0,
    
    // === Sides ===
    side: THREE.FrontSide,
    
    fog: true,
});

// Performance:
// MeshLambertMaterial เร็วกว่า MeshPhongMaterial
// เพราะคำนวณแสงต่อ vertex (vertex shading)
// แทนที่จะคำนวณต่อ fragment (pixel)

// ใช้เมื่อ:
// - ต้องการแสงแต่ performance สำคัญ
// - Objects ที่ไม่ต้องการ specular highlights
// - Mobile games

// ❌ ไม่ดี สำหรับ:
// - Objects ที่ต้องการ shiny surface (ใช้ Phong แทน)
// - High-quality renders (ใช้ Standard แทน)

// Emissive: สีที่วัตถุส่งแสงออกมา (ไม่ได้รับผลจาก lighting)
const glowingMat = new THREE.MeshLambertMaterial({
    color: 0x000000,       // surface สีดำ
    emissive: 0x6c63ff,    // ส่งแสงสีม่วง
    emissiveIntensity: 1.0,
});

// Animation emissive intensity
function animate() {
    requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    glowingMat.emissiveIntensity = 0.5 + Math.sin(elapsed) * 0.5; // 0-1
    renderer.render(scene, camera);
}
```

---

## 9. MeshPhongMaterial

เพิ่ม **specular highlights** บน top ของ Lambert  
ใช้ Phong reflection model

```javascript
// Phong Model = Ambient + Diffuse + Specular
// ดูดีกว่า Lambert แต่ช้ากว่า

const phongMat = new THREE.MeshPhongMaterial({
    // === สีหลัก ===
    color: 0x6c63ff,        // diffuse color
    emissive: 0x000000,     // emissive
    emissiveIntensity: 1.0,
    
    // === Specular (สีเงาสว่าง) ===
    specular: 0xffffff,     // สีของ specular highlight
    shininess: 30,          // ความคมของ specular (0-1000+)
    // shininess สูง = specular เล็กแต่สว่างมาก (shiny)
    // shininess ต่ำ = specular ใหญ่และนวล (matte)
    
    // === Texture Maps ===
    map: null,
    normalMap: null,
    normalMapType: THREE.TangentSpaceNormalMap,
    normalScale: new THREE.Vector2(1, 1),
    bumpMap: null,
    bumpScale: 1,
    displacementMap: null,
    displacementScale: 1,
    displacementBias: 0,
    specularMap: null,      // กำหนด specular strength per-pixel
    alphaMap: null,
    envMap: null,
    emissiveMap: null,
    
    // === Display ===
    flatShading: false,
    wireframe: false,
    
    // === Misc ===
    fog: true,
    transparent: false,
    opacity: 1.0,
    side: THREE.FrontSide,
});

// ========== Shininess กับผลลัพธ์ ==========

// Rubber/Plastic (low specular)
const rubber = new THREE.MeshPhongMaterial({
    color: 0x222222,
    specular: 0x333333,
    shininess: 5,
});

// Plastic (medium specular)
const plastic = new THREE.MeshPhongMaterial({
    color: 0x6c63ff,
    specular: 0xffffff,
    shininess: 50,
});

// Polished metal (high specular)
const polished = new THREE.MeshPhongMaterial({
    color: 0xaaaaaa,
    specular: 0xffffff,
    shininess: 200,
});

// Glass-like
const glass = new THREE.MeshPhongMaterial({
    color: 0xffffff,
    specular: 0xffffff,
    shininess: 500,
    transparent: true,
    opacity: 0.3,
});

// ========== Specular Map ==========
// กำหนด shininess per-pixel

const loader = new THREE.TextureLoader();
const specMap = loader.load('/textures/specular.jpg');

const carMat = new THREE.MeshPhongMaterial({
    map: loader.load('/textures/car_color.jpg'),
    specularMap: specMap,  // สีขาว = shiny, สีดำ = matte
    shininess: 80,
});

// ========== เปรียบเทียบ Lambert vs Phong ==========

// Lambert: การคำนวณแสงทำที่ vertex level
// - เร็วกว่า (vertex shader)
// - Specular ไม่ถูกต้อง (interpolated)

// Phong: การคำนวณแสงทำที่ fragment level
// - ช้ากว่าเล็กน้อย (fragment shader)
// - Specular ถูกต้องและสวยงาม
```

---

## 10. Material Side Options

```javascript
// ทำความเข้าใจ Front/Back/Double Side

// Default: FrontSide
// Three.js จะ render เฉพาะ "ด้านหน้า" ของ polygon
// ด้านหน้า = ด้านที่ normal vector ชี้ออก
// ด้านหลัง = ด้านใน (เมื่อมองเข้าไปข้างใน)

// ตัวอย่าง: กล่องปกติ
const box = new THREE.Mesh(
    new THREE.BoxGeometry(1, 1, 1),
    new THREE.MeshBasicMaterial({
        color: 0x6c63ff,
        side: THREE.FrontSide  // default
    })
);
// เมื่อมองจากนอก: เห็น surface
// เมื่อมองจากใน (กล้องเข้าไปข้างใน): ไม่เห็น surface!

// BackSide: render ด้านหลัง
const invertedBox = new THREE.Mesh(
    new THREE.BoxGeometry(1, 1, 1),
    new THREE.MeshBasicMaterial({
        color: 0xff6b6b,
        side: THREE.BackSide
    })
);
// ใช้สำหรับ: skybox, environment room (กล้องอยู่ข้างใน)

// DoubleSide: render ทั้งสองด้าน
const plane = new THREE.Mesh(
    new THREE.PlaneGeometry(2, 2),
    new THREE.MeshBasicMaterial({
        color: 0x6bcb77,
        side: THREE.DoubleSide
    })
);
// ใช้สำหรับ: plane, leaves, flags, paper

// ⚠️ DoubleSide ช้ากว่า FrontSide ประมาณ 2x
// เพราะ renderer ต้อง render ทุก face สองครั้ง

// ========== Practical Examples ==========

// Skybox (camera inside sphere)
const skybox = new THREE.Mesh(
    new THREE.SphereGeometry(50, 32, 32),
    new THREE.MeshBasicMaterial({
        map: skyTexture,
        side: THREE.BackSide  // render ด้านใน
    })
);

// Leaf / Grass blade
const leaf = new THREE.Mesh(
    new THREE.PlaneGeometry(0.2, 0.5),
    new THREE.MeshLambertMaterial({
        map: leafTexture,
        alphaTest: 0.5,       // cut-out transparency
        side: THREE.DoubleSide // เห็นได้ทั้งสองด้าน
    })
);

// Window glass (transparent, both sides)
const window = new THREE.Mesh(
    new THREE.PlaneGeometry(2, 2),
    new THREE.MeshPhongMaterial({
        color: 0xffffff,
        transparent: true,
        opacity: 0.3,
        side: THREE.DoubleSide
    })
);
```

---

## 11. Blending Modes

```javascript
// Blending กำหนดวิธีที่ material ผสมกับ background

const material = new THREE.MeshBasicMaterial({
    color: 0xffffff,
    transparent: true,
    opacity: 0.5,
    
    // กำหนด blending mode
    blending: THREE.NormalBlending,
    
    // Custom blend factors
    blendEquation: THREE.AddEquation,
    blendSrc: THREE.SrcAlphaFactor,
    blendDst: THREE.OneMinusSrcAlphaFactor,
});

// ========== Blending Modes ==========

// Normal (default): alpha blending ปกติ
const normalBlend = new THREE.MeshBasicMaterial({
    map: particleTexture,
    transparent: true,
    blending: THREE.NormalBlending,
});

// Additive: บวกสี (ทำให้สว่างขึ้น)
// ดีสำหรับ: fire, light beams, stars, explosions
const additiveBlend = new THREE.MeshBasicMaterial({
    color: 0xff8800,
    transparent: true,
    opacity: 0.8,
    blending: THREE.AdditiveBlending,
    depthWrite: false,  // สำคัญ! transparent objects ไม่ควร write depth
});

// Subtractive: ลบสี (ทำให้มืดลง)
const subtractBlend = new THREE.MeshBasicMaterial({
    blending: THREE.SubtractiveBlending,
});

// Multiply: คูณสี
const multiplyBlend = new THREE.MeshBasicMaterial({
    blending: THREE.MultiplyBlending,
});

// Custom Blending
const customBlend = new THREE.MeshBasicMaterial({
    transparent: true,
    blending: THREE.CustomBlending,
    blendEquation: THREE.AddEquation,
    blendSrc: THREE.SrcAlphaFactor,
    blendDst: THREE.OneMinusSrcAlphaFactor,
    // ผลลัพธ์ = src.rgb * src.alpha + dst.rgb * (1 - src.alpha)
    // = Normal blending
});

// ========== Use Cases ==========

// Fire particle
const fireMat = new THREE.MeshBasicMaterial({
    map: fireTexture,
    transparent: true,
    blending: THREE.AdditiveBlending,
    depthWrite: false,  // ไม่เขียน depth
    side: THREE.DoubleSide,
});

// Lens flare
const flareMat = new THREE.MeshBasicMaterial({
    map: flareTexture,
    transparent: true,
    blending: THREE.AdditiveBlending,
    depthTest: false,   // แสดงเสมอ ไม่ว่าจะมีอะไรอยู่หน้า
    depthWrite: false,
});
```

---

## 12. Complete Demo with dat.GUI

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Part 004: Materials Demo</title>
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
        renderer.shadowMap.enabled = true;
        renderer.toneMapping = THREE.ACESFilmicToneMapping;
        document.body.appendChild(renderer.domElement);
        
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.target.set(0, 0, 0);
        
        // ====== LIGHTS ======
        const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
        scene.add(ambientLight);
        
        const dirLight = new THREE.DirectionalLight(0xffffff, 2);
        dirLight.position.set(5, 5, 5);
        dirLight.castShadow = true;
        scene.add(dirLight);
        
        const pointLight = new THREE.PointLight(0xff6b6b, 3, 20);
        pointLight.position.set(-4, 3, 0);
        scene.add(pointLight);
        
        const pointLight2 = new THREE.PointLight(0x6c63ff, 3, 20);
        pointLight2.position.set(4, 3, 0);
        scene.add(pointLight2);
        
        // ====== SPHERE GEOMETRY ======
        const sphereGeo = new THREE.SphereGeometry(1, 64, 32);
        
        // ====== MATERIALS ======
        const materials = {
            basic: new THREE.MeshBasicMaterial({ color: '#6c63ff' }),
            normal: new THREE.MeshNormalMaterial({ flatShading: false }),
            depth: new THREE.MeshDepthMaterial(),
            lambert: new THREE.MeshLambertMaterial({ color: '#6c63ff', emissive: '#000' }),
            phong: new THREE.MeshPhongMaterial({ 
                color: '#6c63ff',
                specular: '#ffffff',
                shininess: 100 
            }),
        };
        
        // ====== SPHERES GRID ======
        const materialNames = Object.keys(materials);
        const meshes = {};
        
        materialNames.forEach((name, i) => {
            const mesh = new THREE.Mesh(sphereGeo, materials[name]);
            const angle = (i / materialNames.length) * Math.PI * 2;
            const radius = 3;
            mesh.position.x = Math.cos(angle) * radius;
            mesh.position.z = Math.sin(angle) * radius;
            mesh.castShadow = true;
            mesh.name = name;
            scene.add(mesh);
            meshes[name] = mesh;
        });
        
        // Labels (ถ้ามี CSS2DRenderer จะดีกว่า แต่ใช้ console แทน)
        console.log('Materials:', materialNames.join(', '));
        
        // ====== GROUND ======
        const ground = new THREE.Mesh(
            new THREE.PlaneGeometry(20, 20),
            new THREE.MeshPhongMaterial({ color: '#1a1a2e', shininess: 0 })
        );
        ground.rotation.x = -Math.PI / 2;
        ground.position.y = -1.2;
        ground.receiveShadow = true;
        scene.add(ground);
        
        // Grid
        const grid = new THREE.GridHelper(20, 20, '#333', '#222');
        grid.position.y = -1.19;
        scene.add(grid);
        
        // ====== GUI ======
        const gui = new GUI({ title: 'Materials' });
        
        // Lambert controls
        const lambertFolder = gui.addFolder('MeshLambertMaterial');
        const lambertParams = {
            color: '#6c63ff',
            emissive: '#000000',
            emissiveIntensity: 1.0,
            wireframe: false,
            flatShading: false,
        };
        lambertFolder.addColor(lambertParams, 'color').onChange(v => materials.lambert.color.set(v));
        lambertFolder.addColor(lambertParams, 'emissive').onChange(v => materials.lambert.emissive.set(v));
        lambertFolder.add(lambertParams, 'emissiveIntensity', 0, 3, 0.01).onChange(v => {
            materials.lambert.emissiveIntensity = v;
        });
        lambertFolder.add(lambertParams, 'wireframe').onChange(v => {
            materials.lambert.wireframe = v;
        });
        lambertFolder.add(lambertParams, 'flatShading').onChange(v => {
            materials.lambert.flatShading = v;
            materials.lambert.needsUpdate = true;
        });
        
        // Phong controls
        const phongFolder = gui.addFolder('MeshPhongMaterial');
        const phongParams = {
            color: '#6c63ff',
            specular: '#ffffff',
            shininess: 100,
            emissive: '#000000',
            flatShading: false,
        };
        phongFolder.addColor(phongParams, 'color').onChange(v => materials.phong.color.set(v));
        phongFolder.addColor(phongParams, 'specular').onChange(v => materials.phong.specular.set(v));
        phongFolder.add(phongParams, 'shininess', 0, 300, 1).onChange(v => {
            materials.phong.shininess = v;
        });
        phongFolder.addColor(phongParams, 'emissive').onChange(v => materials.phong.emissive.set(v));
        phongFolder.add(phongParams, 'flatShading').onChange(v => {
            materials.phong.flatShading = v;
            materials.phong.needsUpdate = true;
        });
        
        // Lights
        const lightFolder = gui.addFolder('Lights');
        const lightParams = {
            ambientIntensity: 0.5,
            dirIntensity: 2,
            pointIntensity: 3,
        };
        lightFolder.add(lightParams, 'ambientIntensity', 0, 2, 0.01).onChange(v => {
            ambientLight.intensity = v;
        });
        lightFolder.add(lightParams, 'dirIntensity', 0, 5, 0.01).onChange(v => {
            dirLight.intensity = v;
        });
        lightFolder.add(lightParams, 'pointIntensity', 0, 10, 0.01).onChange(v => {
            pointLight.intensity = v;
            pointLight2.intensity = v;
        });
        
        // Normal Material
        const normalFolder = gui.addFolder('MeshNormalMaterial');
        const normalParams = { flatShading: false, wireframe: false };
        normalFolder.add(normalParams, 'flatShading').onChange(v => {
            materials.normal.flatShading = v;
            materials.normal.needsUpdate = true;
        });
        normalFolder.add(normalParams, 'wireframe').onChange(v => {
            materials.normal.wireframe = v;
        });
        
        // ====== ANIMATION ======
        const clock = new THREE.Clock();
        
        function animate() {
            requestAnimationFrame(animate);
            
            const elapsed = clock.getElapsedTime();
            const delta = clock.getDelta();
            
            // หมุน spheres
            Object.values(meshes).forEach((mesh, i) => {
                mesh.rotation.y += 0.005;
            });
            
            // Animate lights
            pointLight.position.x = Math.sin(elapsed * 0.7) * 5;
            pointLight.position.z = Math.cos(elapsed * 0.7) * 5;
            pointLight2.position.x = Math.sin(elapsed * 0.7 + Math.PI) * 5;
            pointLight2.position.z = Math.cos(elapsed * 0.7 + Math.PI) * 5;
            
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
        
        // ====== INFO ======
        console.log('%c Part 004: Materials Demo', 'font-size: 16px; color: #6c63ff;');
        console.log('Spheres (left to right): Basic, Normal, Depth, Lambert, Phong');
    </script>
</body>
</html>
```

---

## สรุปตาราง Material เปรียบเทียบ

| Material | แสง | Specular | Performance | ใช้เมื่อ |
|----------|-----|---------|-------------|---------|
| MeshBasicMaterial | ❌ | ❌ | ⚡⚡⚡⚡⚡ | Debug, HUD, UI |
| MeshNormalMaterial | ❌ | ❌ | ⚡⚡⚡⚡⚡ | Debug normals |
| MeshDepthMaterial | ❌ | ❌ | ⚡⚡⚡⚡⚡ | Debug depth |
| MeshMatcapMaterial | ❌ (fake) | ❌ | ⚡⚡⚡⚡ | Quick visual |
| MeshLambertMaterial | ✅ | ❌ | ⚡⚡⚡⚡ | Matte objects |
| MeshPhongMaterial | ✅ | ✅ | ⚡⚡⚡ | Shiny objects |
| MeshStandardMaterial | ✅ | ✅ | ⚡⚡ | PBR general |
| MeshPhysicalMaterial | ✅ | ✅ | ⚡ | PBR advanced |

---

## แบบฝึกหัด (Exercises)

### Exercise 1: Material Showcase (ง่าย)
สร้าง sphere แต่ละอันด้วย material แต่ละชนิด วางเรียงกัน  
เพิ่มป้าย label (ใช้ div overlay) บอกชื่อ material แต่ละอัน

### Exercise 2: Phong Parameters (ง่าย-ปานกลาง)
สร้าง sphere หนึ่งลูกพร้อม `MeshPhongMaterial`  
ใช้ dat.GUI ให้ปรับ:
- color, specular, shininess, emissive
- ambient light intensity, directional light intensity
- flat shading toggle

### Exercise 3: Neon Effect (ปานกลาง)
ใช้ `MeshPhongMaterial` + emissive สร้างวัตถุที่ดูเหมือนเรืองแสง (neon)  
Hint: color = สีหลัก, emissive = สีแสงที่ส่งออกมา

### Exercise 4: Color Animation (ปานกลาง)
ทำ animation ที่เปลี่ยนสีของ material ตามเวลา  
เช่น rainbow effect โดยเปลี่ยน hue ไปเรื่อยๆ

```javascript
// Hint:
function animate() {
    requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    // เปลี่ยน hue ตามเวลา
    material.color.setHSL(elapsed * 0.1 % 1, 1, 0.5);
    renderer.render(scene, camera);
}
```

### Exercise 5: Material Switch (ยาก)
สร้าง sphere เดี่ยว พร้อม UI buttons ที่สลับ material ได้  
เมื่อสลับ material ทำ transition animation ด้วย opacity fade  
(fade out แล้ว เปลี่ยน material แล้ว fade in)

---

**ก่อนหน้า:** [Part 003 - Geometries Deep Dive](part-003.md)  
**ต่อไป:** [Part 005 - Materials Part 2 (PBR)](part-005.md)
