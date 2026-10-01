# Part 002: Your First 3D Scene - ฉากสามมิติแรกของคุณ

## สารบัญบทเรียนนี้
- [1. Three.js Core Concepts ทบทวน](#1-threejs-core-concepts-ทบทวน)
- [2. Scene - เวทีของเรา](#2-scene---เวทีของเรา)
- [3. PerspectiveCamera - กล้องถ่ายรูป](#3-perspectivecamera---กล้องถ่ายรูป)
- [4. WebGLRenderer - ตัว render](#4-webglrenderer---ตัว-render)
- [5. Mesh = Geometry + Material](#5-mesh--geometry--material)
- [6. BoxGeometry พื้นฐาน](#6-boxgeometry-พื้นฐาน)
- [7. MeshBasicMaterial](#7-meshbasicmaterial)
- [8. Canvas Element](#8-canvas-element)
- [9. Render Loop](#9-render-loop)
- [10. Window Resize Handler](#10-window-resize-handler)
- [11. ฉากสมบูรณ์พร้อม Comments](#11-ฉากสมบูรณ์พร้อม-comments)
- [12. Variations และการทดลอง](#12-variations-และการทดลอง)
- [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. Three.js Core Concepts ทบทวน

ก่อนจะเริ่ม เรามาทำความเข้าใจ "สามสิ่งหลัก" ของทุก Three.js project:

```
Scene + Camera + Renderer = Three.js Application
  เวที    กล้อง    ตัวฉาย        แอปพลิเคชัน
```

เปรียบเทียบกับการถ่ายหนัง:
- **Scene** = ฉากและ set ที่ตั้งไว้ (พร้อมนักแสดง, ไฟ, พื้นหลัง)
- **Camera** = กล้องถ่ายทำ (มุม, ระยะ, ขนาด)
- **Renderer** = โรงฉาย/จอฉาย (ทำให้เราเห็น)

```javascript
// โครงสร้าง minimal ของ Three.js
const scene = new THREE.Scene();           // 1. สร้างเวที
const camera = new THREE.PerspectiveCamera(...); // 2. วางกล้อง
const renderer = new THREE.WebGLRenderer(); // 3. เตรียมจอ

// เพิ่ม object เข้า scene
scene.add(someObject);

// Render loop
function animate() {
    requestAnimationFrame(animate);
    renderer.render(scene, camera); // render scene ผ่าน camera
}
animate();
```

---

## 2. Scene - เวทีของเรา

`THREE.Scene` คือ container หลักที่เก็บทุกอย่างใน 3D world ของเรา

### การสร้าง Scene

```javascript
const scene = new THREE.Scene();
```

### Properties ที่สำคัญของ Scene

```javascript
// 1. Background - สีพื้นหลัง
scene.background = new THREE.Color(0x000000);        // สีดำ
scene.background = new THREE.Color('#1a1a2e');        // Hex string
scene.background = new THREE.Color('rgb(26,26,46)'); // RGB string
scene.background = new THREE.Color('hsl(240, 28%, 14%)'); // HSL string
// หรือ null (transparent)
scene.background = null;

// 2. Fog - หมอก
// Linear Fog: เริ่มหมอกที่ near หนามากที่ far
scene.fog = new THREE.Fog(
    0x000000,  // สีหมอก
    10,        // near: เริ่มมีหมอกที่ระยะ 10
    100        // far: หนาสุดที่ระยะ 100
);

// Exponential Fog: หนาขึ้นแบบ exponential
scene.fog = new THREE.FogExp2(
    0x000000, // สีหมอก
    0.02      // density: ความหนาแน่น (ค่าน้อย = หมอกน้อย)
);

// 3. Environment - แสงรอบทิศ (จาก HDR texture)
// เราจะเรียนเรื่องนี้ใน Part 009

// 4. Scene Graph - hierarchy ของ objects
console.log(scene.children); // ดู objects ทั้งหมดใน scene

// เพิ่ม/ลบ objects
scene.add(mesh);
scene.remove(mesh);

// ใช้ traverse เพื่อ iterate ผ่านทุก object
scene.traverse((object) => {
    console.log(object.name, object.type);
});

// หา object ด้วยชื่อ
mesh.name = 'myCube';
const foundMesh = scene.getObjectByName('myCube');

// หา object ด้วย id
const foundById = scene.getObjectById(mesh.id);
```

### ตัวอย่าง Scene ที่มีการตั้งค่าครบ

```javascript
function createScene() {
    const scene = new THREE.Scene();
    
    // Background gradient (ต้องทำผ่าน CSS หรือ texture)
    scene.background = new THREE.Color('#0d1117');
    
    // Fog เพื่อความลึก
    scene.fog = new THREE.FogExp2('#0d1117', 0.015);
    
    return scene;
}
```

---

## 3. PerspectiveCamera - กล้องถ่ายรูป

`THREE.PerspectiveCamera` คือกล้องที่จำลองการมองของมนุษย์  
object ที่อยู่ไกลจะดูเล็กกว่า object ที่อยู่ใกล้ (perspective projection)

### Constructor Parameters

```javascript
const camera = new THREE.PerspectiveCamera(
    fov,    // Field of View - มุมมองกว้างของกล้อง (degrees)
    aspect, // Aspect Ratio - อัตราส่วนกว้าง:สูง
    near,   // Near Clipping Plane - ระยะใกล้สุดที่ render
    far     // Far Clipping Plane - ระยะไกลสุดที่ render
);
```

### ทำความเข้าใจ FOV (Field of View)

```
        Camera
           |
     ------+------  <- ขอบ frustum (left/right based on FOV)
    /       |       \
   /        |        \
  /         |         \
 /          |          \
(far plane - สิ่งที่อยู่ไกลกว่านี้จะไม่ render)
```

FOV ยิ่งกว้าง:
- เห็นพื้นที่มากขึ้น
- perspective distortion มากขึ้น (เหมือนกล้อง fisheye)
- ดูคล้ายกล้อง wide-angle

FOV ยิ่งแคบ:
- เห็นพื้นที่น้อยลง
- perspective distortion น้อยลง
- ดูคล้าย telephoto / zoom lens

```javascript
// FOV ต่างๆ กับ use cases
// 15-30°: telephoto / cinematic zoom shots
// 45°: ใกล้เคียงกับตามนุษย์ธรรมดา
// 60-75°: มาตรฐานสำหรับ games ทั่วไป
// 90-120°: first-person games, VR
// 120°+: fisheye effect

// สำหรับ architectural visualization: ~45°
const archCamera = new THREE.PerspectiveCamera(45, aspect, 0.1, 1000);

// สำหรับ first-person game: ~90°
const fpCamera = new THREE.PerspectiveCamera(90, aspect, 0.01, 100);
```

### Aspect Ratio

```javascript
// Aspect ratio ต้องตรงกับขนาดของ renderer เสมอ
// ถ้า renderer 800x600 → aspect = 800/600 = 1.333...
// ถ้า renderer fullscreen → aspect = window.innerWidth / window.innerHeight

camera.aspect = window.innerWidth / window.innerHeight;
camera.updateProjectionMatrix(); // ต้องเรียกทุกครั้งที่เปลี่ยน aspect
```

### Near และ Far Clipping Planes

```javascript
// Near: ระยะต่ำสุดจาก camera ที่จะ render
// Far: ระยะสูงสุดจาก camera ที่จะ render
// Object ที่อยู่นอกช่วงนี้จะถูก "clip" ออก

// ⚠️ ข้อควรระวัง: near/far ratio มีผลต่อ depth buffer precision
// ถ้า ratio ใหญ่มาก (เช่น near=0.001, far=100000) จะเกิด "z-fighting"

// ❌ ไม่ดี: ratio ใหญ่เกินไป
const badCamera = new THREE.PerspectiveCamera(75, aspect, 0.001, 100000);

// ✅ ดี: ratio สมเหตุสมผล
const goodCamera = new THREE.PerspectiveCamera(75, aspect, 0.1, 1000);

// กฎทั่วไป: near/far ratio ไม่ควรเกิน 10,000
// near = far / 10000 เป็นอย่างต่ำ
```

### การควบคุม Camera

```javascript
// Position
camera.position.set(x, y, z);
camera.position.x = 5;
camera.position.y = 3;
camera.position.z = 10;

// Rotation (ไม่แนะนำโดยตรง ควรใช้ lookAt)
camera.rotation.y = Math.PI / 4;

// lookAt - บอก camera ให้มองไปที่จุดนี้
camera.lookAt(0, 0, 0);            // มองที่ origin
camera.lookAt(new THREE.Vector3(1, 0, 0)); // มองที่จุด (1,0,0)
camera.lookAt(targetMesh.position); // มองที่ mesh

// up vector - กำหนดทิศขึ้น (default: Y-up)
camera.up.set(0, 1, 0); // default

// หา direction ที่ camera มอง
const direction = new THREE.Vector3();
camera.getWorldDirection(direction);
console.log('Camera looking at:', direction);
```

---

## 4. WebGLRenderer - ตัว Render

`THREE.WebGLRenderer` คือส่วนที่แปลง scene + camera ให้เป็นภาพบน canvas

### Constructor Options

```javascript
const renderer = new THREE.WebGLRenderer({
    // canvas: กำหนด canvas element ที่ต้องการใช้
    // ถ้าไม่กำหนด จะสร้างใหม่
    canvas: document.querySelector('#canvas'),
    
    // antialias: เปิด multi-sample anti-aliasing (MSAA)
    // ทำให้ขอบ object เรียบขึ้น แต่ใช้ memory มากขึ้น
    antialias: true,
    
    // alpha: เปิด transparent background
    // ใช้เมื่อต้องการให้ WebGL render บน background อื่น (เช่น video, image)
    alpha: true,
    
    // logarithmicDepthBuffer: แก้ปัญหา z-fighting ในฉากขนาดใหญ่มาก
    // แต่ทำให้ช้าลงเล็กน้อย
    logarithmicDepthBuffer: false,
    
    // preserveDrawingBuffer: เก็บ buffer ไว้หลัง render
    // ใช้เมื่อต้องการ screenshot (canvas.toDataURL())
    preserveDrawingBuffer: false,
    
    // powerPreference: บอก browser ว่าต้องการ GPU แบบไหน
    // 'default' | 'high-performance' | 'low-power'
    powerPreference: 'high-performance',
    
    // failIfMajorPerformanceCaveat: throw error ถ้า performance จะแย่มาก
    failIfMajorPerformanceCaveat: false,
    
    // precision: ความละเอียดของ fragment shader
    // 'highp' | 'mediump' | 'lowp'
    precision: 'highp',
    
    // stencil: เปิด stencil buffer (ใช้สำหรับ stencil effects)
    stencil: true,
    
    // depth: เปิด depth buffer
    depth: true,
});
```

### การตั้งค่าหลังสร้าง Renderer

```javascript
// ขนาด
renderer.setSize(window.innerWidth, window.innerHeight);
// setSize(width, height, updateStyle)
// updateStyle: อัปเดต CSS ของ canvas ด้วยหรือไม่ (default: true)

// Pixel Ratio (สำหรับ retina display)
// devicePixelRatio: 1 = normal, 2 = retina, 3+ = high-end retina
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));

// Shadow Map
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
// THREE.BasicShadowMap (เร็วสุดแต่คุณภาพต่ำสุด)
// THREE.PCFShadowMap (default, ดี)
// THREE.PCFSoftShadowMap (นุ่มกว่า แต่ช้ากว่า)
// THREE.VSMShadowMap (Variance Shadow Map, ดีที่สุดแต่ช้า)

// Tone Mapping
renderer.toneMapping = THREE.ACESFilmicToneMapping;
// THREE.NoToneMapping (default)
// THREE.LinearToneMapping
// THREE.ReinhardToneMapping
// THREE.CineonToneMapping
// THREE.ACESFilmicToneMapping (แนะนำ)

// Exposure (ความสว่างรวม)
renderer.toneMappingExposure = 1.0; // 1.0 = ปกติ

// Color Space Output
renderer.outputColorSpace = THREE.SRGBColorSpace; // default ใน r152+

// Clear Color (สีที่ใช้ clear ก่อน render ใหม่)
renderer.setClearColor(0x000000, 1); // สี, alpha
// หรือ
renderer.setClearColor(new THREE.Color(0x000000), 0); // alpha=0 = transparent

// ดูข้อมูล WebGL
const info = renderer.info;
console.log('Draw calls:', info.render.calls);
console.log('Triangles:', info.render.triangles);
console.log('Geometries:', info.memory.geometries);
console.log('Textures:', info.memory.textures);
```

### การ Render

```javascript
// render scene ปกติ
renderer.render(scene, camera);

// render ไปยัง RenderTarget (เราจะเรียนใน Part 071)
renderer.setRenderTarget(myRenderTarget);
renderer.render(scene, camera);
renderer.setRenderTarget(null); // reset กลับมา screen

// clear ด้วยมือ (ปกติ renderer จะ clear ให้อัตโนมัติ)
renderer.clear(); // clear ทั้งหมด
renderer.clearColor(); // clear แค่ color buffer
renderer.clearDepth(); // clear แค่ depth buffer
renderer.clearStencil(); // clear แค่ stencil buffer
```

---

## 5. Mesh = Geometry + Material

`THREE.Mesh` คือ 3D object ที่มองเห็นได้ ประกอบจาก:

```
Mesh
 ├── Geometry (รูปร่าง - โครงสร้าง vertices)
 └── Material (วัสดุ - สี, พื้นผิว, การสะท้อนแสง)
```

```javascript
// สร้าง Mesh
const geometry = new THREE.BoxGeometry(1, 1, 1);
const material = new THREE.MeshBasicMaterial({ color: 0xff0000 });
const mesh = new THREE.Mesh(geometry, material);

// Mesh เป็น Object3D จึงมี properties เหมือนกัน
mesh.position.set(0, 0, 0);
mesh.rotation.set(0, 0, 0);
mesh.scale.set(1, 1, 1);
mesh.visible = true;
mesh.name = 'myMesh';

// Shadows
mesh.castShadow = true;    // ทอดเงา
mesh.receiveShadow = true; // รับเงา

// เพิ่มเข้า scene
scene.add(mesh);

// Multi-material (แต่ละ face ใช้ material ต่างกัน)
const materials = [
    new THREE.MeshBasicMaterial({ color: 0xff0000 }), // ขวา
    new THREE.MeshBasicMaterial({ color: 0x00ff00 }), // ซ้าย
    new THREE.MeshBasicMaterial({ color: 0x0000ff }), // บน
    new THREE.MeshBasicMaterial({ color: 0xffff00 }), // ล่าง
    new THREE.MeshBasicMaterial({ color: 0xff00ff }), // หน้า
    new THREE.MeshBasicMaterial({ color: 0x00ffff }), // หลัง
];
const colorfulCube = new THREE.Mesh(
    new THREE.BoxGeometry(1, 1, 1),
    materials  // ส่ง array ของ materials
);
```

---

## 6. BoxGeometry พื้นฐาน

`THREE.BoxGeometry` คือ geometry รูปกล่อง/ลูกบาศก์

```javascript
// Syntax
const geometry = new THREE.BoxGeometry(
    width,          // ความกว้าง (default: 1)
    height,         // ความสูง (default: 1)
    depth,          // ความลึก (default: 1)
    widthSegments,  // จำนวนส่วนย่อยแกน X (default: 1)
    heightSegments, // จำนวนส่วนย่อยแกน Y (default: 1)
    depthSegments   // จำนวนส่วนย่อยแกน Z (default: 1)
);

// ตัวอย่าง
const cube1x1x1 = new THREE.BoxGeometry();           // 1x1x1 default
const cube2x2x2 = new THREE.BoxGeometry(2, 2, 2);    // 2x2x2
const box = new THREE.BoxGeometry(3, 1, 2);           // กว้าง 3, สูง 1, ลึก 2

// ใช้ segments มากขึ้น (เพื่อ displacement map หรือ deformation)
const detailedBox = new THREE.BoxGeometry(1, 1, 1, 10, 10, 10);
// แต่ triangle count เพิ่มมาก!

// ดู geometry info
console.log('Vertices:', geometry.attributes.position.count);
console.log('Triangle count:', geometry.index.count / 3);

// ดู bounding box
geometry.computeBoundingBox();
const bbox = geometry.boundingBox;
console.log('Min:', bbox.min);
console.log('Max:', bbox.max);
console.log('Size:', bbox.getSize(new THREE.Vector3()));
```

### Geometry Properties

```javascript
// attributes - ข้อมูลต่างๆ ของ vertices
geometry.attributes.position; // ตำแหน่ง vertex (vec3)
geometry.attributes.normal;   // normal vector (vec3)
geometry.attributes.uv;       // UV coordinates (vec2)

// การเข้าถึงข้อมูล
const positions = geometry.attributes.position;
for (let i = 0; i < positions.count; i++) {
    console.log(`Vertex ${i}:`, 
        positions.getX(i), 
        positions.getY(i), 
        positions.getZ(i)
    );
}

// การแก้ไขข้อมูล
positions.setXYZ(0, 2, 0, 0); // แก้ไข vertex ที่ index 0
positions.needsUpdate = true;  // บอก Three.js ว่าข้อมูลเปลี่ยน
```

---

## 7. MeshBasicMaterial

`THREE.MeshBasicMaterial` คือ material ที่ง่ายที่สุด  
**ไม่ได้รับผลจากแสง** - แสดงสีตรงๆ ไม่สนใจ lighting

```javascript
const material = new THREE.MeshBasicMaterial({
    // ===== สี =====
    color: 0xff0000,          // สีแดง (hex number)
    color: '#ff0000',         // hex string
    color: 'red',             // color name
    color: 'rgb(255,0,0)',    // RGB
    
    // ===== Transparency =====
    transparent: false,       // เปิด transparency (ต้องเปิดก่อนจึงจะใช้ opacity ได้)
    opacity: 1.0,             // ความโปร่งแสง (0.0 = โปร่งใส, 1.0 = ทึบ)
    alphaTest: 0,             // ค่า alpha ต่ำกว่านี้จะไม่ render (0-1)
    
    // ===== Sides =====
    side: THREE.FrontSide,    // default: render แค่ด้านหน้า
    // THREE.FrontSide  = ด้านหน้าเท่านั้น
    // THREE.BackSide   = ด้านหลังเท่านั้น
    // THREE.DoubleSide = ทั้งสองด้าน (ช้าลง 2x)
    
    // ===== Wireframe =====
    wireframe: false,         // แสดงเป็น wireframe
    wireframeLinewidth: 1,    // ความหนาของ wireframe (browser support จำกัด)
    
    // ===== Texture =====
    map: null,                // texture map หลัก (จะเรียนใน Part 006)
    
    // ===== Visibility =====
    visible: true,            // แสดง/ซ่อน
    depthTest: true,          // ทดสอบ depth (ปกติเปิดไว้)
    depthWrite: true,         // เขียน depth buffer (ปิดสำหรับ transparent objects)
    
    // ===== Blending =====
    blending: THREE.NormalBlending, // วิธีผสมสี
    // THREE.NoBlending
    // THREE.NormalBlending   (default)
    // THREE.AdditiveBlending (เหมาะกับไฟ, กลีบดอกไม้เรือง)
    // THREE.SubtractiveBlending
    // THREE.MultiplyBlending
    
    // ===== Vertex Colors =====
    vertexColors: false,      // ใช้สีจาก geometry vertices
});

// การเปลี่ยนแปลงหลังสร้าง
material.color.set(0x00ff00); // เปลี่ยนสี
material.color.setHex(0x00ff00);
material.color.setRGB(0, 1, 0);
material.color.setHSL(0.333, 1, 0.5);
material.opacity = 0.5;
material.needsUpdate = true; // บอก Three.js เมื่อเปลี่ยน properties บางอย่าง
```

---

## 8. Canvas Element

Three.js ทำงานกับ HTML `<canvas>` element

### วิธีจัดการ Canvas

```javascript
// วิธีที่ 1: ให้ Three.js สร้าง canvas เอง (แล้ว append เข้า body)
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// วิธีที่ 2: ใช้ canvas ที่มีอยู่ใน HTML
// HTML: <canvas id="myCanvas"></canvas>
const canvas = document.getElementById('myCanvas');
const renderer = new THREE.WebGLRenderer({ 
    canvas: canvas,
    antialias: true 
});
renderer.setSize(window.innerWidth, window.innerHeight);

// วิธีที่ 3: ใช้ query selector
const canvas = document.querySelector('canvas');
const renderer = new THREE.WebGLRenderer({ canvas });

// ดึง canvas จาก renderer
const canvasElement = renderer.domElement;

// ปรับ style
renderer.domElement.style.position = 'fixed';
renderer.domElement.style.top = '0';
renderer.domElement.style.left = '0';
renderer.domElement.style.zIndex = '-1'; // วางไว้ด้านหลัง HTML content

// Screenshot
function takeScreenshot() {
    // ต้องสร้าง renderer ด้วย preserveDrawingBuffer: true
    const dataURL = renderer.domElement.toDataURL('image/png');
    const link = document.createElement('a');
    link.download = 'screenshot.png';
    link.href = dataURL;
    link.click();
}
```

---

## 9. Render Loop

Animation loop คือ "หัวใจ" ของทุก Three.js application

### requestAnimationFrame

```javascript
// requestAnimationFrame เป็น Web API ที่เรียก callback
// ก่อนที่ browser จะ repaint หน้าจอครั้งต่อไป
// โดยปกติ = 60 ครั้ง/วินาที (60 fps)
// แต่อาจมากกว่าบน high refresh rate monitor (120hz, 144hz)

function animate() {
    requestAnimationFrame(animate); // ลงทะเบียนให้เรียกครั้งต่อไป
    renderer.render(scene, camera);
}
animate(); // เริ่ม loop

// ⚠️ อย่าใช้ setInterval หรือ setTimeout สำหรับ render loop
// เพราะ requestAnimationFrame จะหยุดทำงานเมื่อ tab ไม่ active
// ทำให้ประหยัด battery บน mobile
```

### Delta Time - ทำ animation ให้ smooth ในทุก FPS

```javascript
// ปัญหา: ถ้าใช้ค่าคงที่ animation จะเร็วบนเครื่องที่มี FPS สูง
// ❌ ไม่ดี: animation จะเร็วบน 144hz monitor
cube.rotation.y += 0.01; // ไม่ขึ้นกับเวลา

// ✅ ดี: ใช้ delta time ทำให้ animation speed เท่ากันทุก FPS
const clock = new THREE.Clock();

function animate() {
    requestAnimationFrame(animate);
    
    const delta = clock.getDelta(); // เวลาที่ผ่านไปตั้งแต่ frame ที่แล้ว (วินาที)
    
    // @ 60fps: delta ≈ 0.0167
    // @ 30fps: delta ≈ 0.0333
    // @ 120fps: delta ≈ 0.0083
    
    cube.rotation.y += 1 * delta; // หมุน 1 radian/second เสมอ
    
    renderer.render(scene, camera);
}
animate();
```

### THREE.Clock

```javascript
const clock = new THREE.Clock();

// getDelta() - เวลาตั้งแต่ call getDelta() ครั้งล่าสุด
const delta = clock.getDelta();

// getElapsedTime() - เวลาทั้งหมดตั้งแต่ start
const elapsed = clock.getElapsedTime();

// autoStart: เริ่มนับอัตโนมัติเมื่อ getDelta() ถูกเรียกครั้งแรก
const clock2 = new THREE.Clock(false); // ไม่ auto start
clock2.start(); // เริ่มเองด้วยมือ
clock2.stop();  // หยุด
clock2.running; // boolean: กำลัง run หรือไม่

// ⚠️ ข้อควรระวัง: เรียก getDelta() แค่ครั้งเดียวต่อ frame
// ถ้าเรียกหลายครั้ง จะได้ค่า 0 ในครั้งที่ 2+
function animate() {
    requestAnimationFrame(animate);
    
    const delta = clock.getDelta(); // เรียกครั้งเดียวตรงนี้
    const elapsed = clock.getElapsedTime();
    
    // ใช้ delta และ elapsed ตลอด frame นี้
    mesh1.rotation.y += delta * 0.5;
    mesh2.position.x = Math.sin(elapsed);
    
    renderer.render(scene, camera);
}
```

### Performance ของ Animation Loop

```javascript
// วิธีหยุด animation loop
let animationId;

function startAnimation() {
    function animate() {
        animationId = requestAnimationFrame(animate);
        renderer.render(scene, camera);
    }
    animate();
}

function stopAnimation() {
    cancelAnimationFrame(animationId);
}

// หยุดเมื่อ tab ไม่ active (ประหยัด battery)
document.addEventListener('visibilitychange', () => {
    if (document.hidden) {
        stopAnimation();
    } else {
        startAnimation();
    }
});
```

---

## 10. Window Resize Handler

เมื่อหน้าต่าง browser เปลี่ยนขนาด เราต้องอัปเดต:
1. Camera aspect ratio
2. Renderer size
3. Renderer pixel ratio

```javascript
// Resize handler พื้นฐาน
function handleResize() {
    // 1. อัปเดต aspect ratio
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix(); // ⚠️ ต้องเรียกทุกครั้งหลังเปลี่ยน projection params
    
    // 2. อัปเดตขนาด renderer
    renderer.setSize(window.innerWidth, window.innerHeight);
    
    // 3. อัปเดต pixel ratio (เผื่อย้าย monitor)
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
}

window.addEventListener('resize', handleResize);

// ============================================================
// Debounced Resize (ประหยัด performance)
// ============================================================
let resizeTimeout;
window.addEventListener('resize', () => {
    clearTimeout(resizeTimeout);
    resizeTimeout = setTimeout(handleResize, 100); // รอ 100ms หลัง resize หยุด
});

// ============================================================
// ResizeObserver (สำหรับ canvas ที่ไม่ใช่ fullscreen)
// ============================================================
const canvas = renderer.domElement;
const resizeObserver = new ResizeObserver((entries) => {
    for (const entry of entries) {
        const { width, height } = entry.contentRect;
        camera.aspect = width / height;
        camera.updateProjectionMatrix();
        renderer.setSize(width, height);
    }
});
resizeObserver.observe(canvas);
```

---

## 11. ฉากสมบูรณ์พร้อม Comments

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Part 002: First 3D Scene</title>
    <style>
        /* ========== RESET & BASE STYLES ========== */
        *, *::before, *::after {
            box-sizing: border-box;
            margin: 0;
            padding: 0;
        }
        
        html, body {
            width: 100%;
            height: 100%;
            overflow: hidden;
            background: #000; /* fallback color */
        }
        
        /* Three.js จะ append canvas เข้า body */
        canvas {
            display: block;
            width: 100% !important;
            height: 100% !important;
        }
        
        /* ========== UI OVERLAY ========== */
        #ui {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            pointer-events: none; /* ไม่บัง mouse events */
            z-index: 10;
            padding: 20px;
            display: flex;
            justify-content: space-between;
            align-items: flex-start;
        }
        
        .panel {
            background: rgba(0, 0, 0, 0.6);
            backdrop-filter: blur(8px);
            border: 1px solid rgba(255, 255, 255, 0.1);
            border-radius: 12px;
            padding: 12px 16px;
            color: white;
            font-family: 'Courier New', monospace;
            font-size: 12px;
            line-height: 1.6;
        }
        
        .panel h3 {
            color: #6c63ff;
            font-size: 14px;
            margin-bottom: 8px;
        }
        
        .stat { color: #aaa; }
        .stat span { color: #6c63ff; font-weight: bold; }
    </style>
</head>
<body>

    <!-- UI Overlay -->
    <div id="ui">
        <!-- Info Panel -->
        <div class="panel">
            <h3>Part 002: First 3D Scene</h3>
            <div class="stat">FPS: <span id="fps">0</span></div>
            <div class="stat">Objects: <span id="objects">0</span></div>
            <div class="stat">Triangles: <span id="triangles">0</span></div>
            <div class="stat">Draw Calls: <span id="drawcalls">0</span></div>
        </div>
        
        <!-- Controls Panel -->
        <div class="panel">
            <h3>Controls</h3>
            <div class="stat">Mouse: Orbit</div>
            <div class="stat">Scroll: Zoom</div>
            <div class="stat">Right Click: Pan</div>
        </div>
    </div>
    
    <!-- Import Map -->
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
        
        // ================================================================
        // PART 1: SCENE SETUP
        // ================================================================
        
        // สร้าง Scene
        const scene = new THREE.Scene();
        scene.background = new THREE.Color('#0d1117'); // GitHub-dark background
        
        // เพิ่ม fog สำหรับ depth
        scene.fog = new THREE.Fog('#0d1117', 15, 40);
        
        // ================================================================
        // PART 2: CAMERA SETUP
        // ================================================================
        
        // สร้าง PerspectiveCamera
        const camera = new THREE.PerspectiveCamera(
            60,                                     // fov: 60 องศา
            window.innerWidth / window.innerHeight, // aspect
            0.1,                                    // near
            100                                     // far
        );
        
        // วางตำแหน่ง camera
        camera.position.set(3, 2, 5);
        
        // ================================================================
        // PART 3: RENDERER SETUP
        // ================================================================
        
        // สร้าง Renderer
        const renderer = new THREE.WebGLRenderer({
            antialias: true,
            powerPreference: 'high-performance'
        });
        
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        
        // เปิด shadows
        renderer.shadowMap.enabled = true;
        renderer.shadowMap.type = THREE.PCFSoftShadowMap;
        
        // Tone mapping
        renderer.toneMapping = THREE.ACESFilmicToneMapping;
        renderer.toneMappingExposure = 1.2;
        
        // เพิ่ม canvas เข้า DOM
        document.body.appendChild(renderer.domElement);
        
        // ================================================================
        // PART 4: ORBIT CONTROLS
        // ================================================================
        
        // OrbitControls ให้เราหมุน, zoom, pan ด้วย mouse
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;    // damping (smooth movement)
        controls.dampingFactor = 0.05;
        controls.enableZoom = true;
        controls.enablePan = true;
        controls.minDistance = 1;
        controls.maxDistance = 30;
        controls.maxPolarAngle = Math.PI * 0.85; // ไม่ให้กล้องลงต่ำเกินไป
        controls.target.set(0, 0.5, 0); // จุดที่กล้องมองไปหา
        
        // ================================================================
        // PART 5: LIGHTS
        // ================================================================
        
        // Ambient Light - แสงรอบทิศ (ไม่มีทิศทาง)
        const ambientLight = new THREE.AmbientLight(0xffffff, 0.4);
        scene.add(ambientLight);
        
        // Directional Light - แสงทิศทางเดียว (เหมือนดวงอาทิตย์)
        const dirLight = new THREE.DirectionalLight(0xffffff, 1.5);
        dirLight.position.set(5, 8, 5);
        dirLight.castShadow = true;
        dirLight.shadow.mapSize.width = 2048;
        dirLight.shadow.mapSize.height = 2048;
        dirLight.shadow.camera.near = 0.5;
        dirLight.shadow.camera.far = 30;
        scene.add(dirLight);
        
        // Point Light - แสงจากจุด (เหมือนหลอดไฟ)
        const pointLight = new THREE.PointLight(0x6c63ff, 2, 10);
        pointLight.position.set(-3, 2, 0);
        scene.add(pointLight);
        
        // ================================================================
        // PART 6: CREATE OBJECTS
        // ================================================================
        
        // --- Main Cube ---
        const cubeGeo = new THREE.BoxGeometry(1, 1, 1);
        const cubeMat = new THREE.MeshStandardMaterial({
            color: '#6c63ff',
            metalness: 0.3,
            roughness: 0.4,
        });
        const cube = new THREE.Mesh(cubeGeo, cubeMat);
        cube.position.set(0, 0.5, 0);
        cube.castShadow = true;
        cube.receiveShadow = true;
        cube.name = 'MainCube';
        scene.add(cube);
        
        // --- Wireframe Cube (แสดงโครงสร้าง) ---
        const wireGeo = new THREE.BoxGeometry(1.01, 1.01, 1.01);
        const wireMat = new THREE.MeshBasicMaterial({
            color: '#fff',
            wireframe: true,
            transparent: true,
            opacity: 0.1
        });
        const wireframe = new THREE.Mesh(wireGeo, wireMat);
        cube.add(wireframe); // เพิ่มเป็น child ของ cube
        
        // --- Small Orbiting Spheres ---
        const sphereGroup = new THREE.Group();
        scene.add(sphereGroup);
        
        const sphereColors = ['#ff6b6b', '#ffd93d', '#6bcb77', '#4d96ff'];
        sphereColors.forEach((color, i) => {
            const sGeo = new THREE.SphereGeometry(0.15, 16, 16);
            const sMat = new THREE.MeshStandardMaterial({ color, roughness: 0.3 });
            const sphere = new THREE.Mesh(sGeo, sMat);
            
            // วางในวงกลม
            const angle = (i / sphereColors.length) * Math.PI * 2;
            sphere.position.x = Math.cos(angle) * 2;
            sphere.position.z = Math.sin(angle) * 2;
            sphere.position.y = 0.5;
            sphere.castShadow = true;
            
            sphereGroup.add(sphere);
        });
        
        // --- Ground Plane ---
        const groundGeo = new THREE.PlaneGeometry(30, 30);
        const groundMat = new THREE.MeshStandardMaterial({
            color: '#1a1a2e',
            roughness: 0.9,
            metalness: 0.0
        });
        const ground = new THREE.Mesh(groundGeo, groundMat);
        ground.rotation.x = -Math.PI / 2; // หมุน 90° เพื่อวางนอน
        ground.receiveShadow = true;
        scene.add(ground);
        
        // ================================================================
        // PART 7: HELPERS
        // ================================================================
        
        // Axes Helper: X=แดง, Y=เขียว, Z=น้ำเงิน
        const axesHelper = new THREE.AxesHelper(3);
        scene.add(axesHelper);
        
        // Grid Helper
        const gridHelper = new THREE.GridHelper(20, 20, '#444', '#2a2a2a');
        scene.add(gridHelper);
        
        // DirectionalLight Helper (แสดงทิศทางแสง)
        const dirLightHelper = new THREE.DirectionalLightHelper(dirLight, 2, '#ffff00');
        scene.add(dirLightHelper);
        
        // PointLight Helper (แสดงตำแหน่งไฟ)
        const pointLightHelper = new THREE.PointLightHelper(pointLight, 0.3, '#6c63ff');
        scene.add(pointLightHelper);
        
        // ================================================================
        // PART 8: KEYBOARD CONTROLS (Toggle Helpers)
        // ================================================================
        
        window.addEventListener('keydown', (e) => {
            switch (e.key.toLowerCase()) {
                case 'h': // toggle helpers
                    axesHelper.visible = !axesHelper.visible;
                    gridHelper.visible = !gridHelper.visible;
                    dirLightHelper.visible = !dirLightHelper.visible;
                    pointLightHelper.visible = !pointLightHelper.visible;
                    break;
                case 'w': // toggle wireframe
                    wireframe.visible = !wireframe.visible;
                    break;
                case 'f': // toggle fog
                    scene.fog = scene.fog ? null : new THREE.Fog('#0d1117', 15, 40);
                    break;
            }
        });
        
        // ================================================================
        // PART 9: RESIZE HANDLER
        // ================================================================
        
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        });
        
        // ================================================================
        // PART 10: ANIMATION LOOP
        // ================================================================
        
        const clock = new THREE.Clock();
        
        // FPS counter
        let frameCount = 0;
        let lastTime = performance.now();
        let fps = 0;
        
        function updateUI() {
            const now = performance.now();
            frameCount++;
            if (now - lastTime >= 1000) {
                fps = frameCount;
                frameCount = 0;
                lastTime = now;
                
                document.getElementById('fps').textContent = fps;
                document.getElementById('objects').textContent = scene.children.length;
                document.getElementById('triangles').textContent = 
                    renderer.info.render.triangles.toLocaleString();
                document.getElementById('drawcalls').textContent = 
                    renderer.info.render.calls;
            }
        }
        
        function animate() {
            requestAnimationFrame(animate);
            
            const delta = clock.getDelta();
            const elapsed = clock.getElapsedTime();
            
            // หมุน main cube
            cube.rotation.y += delta * 0.8;
            
            // ทำให้ cube ลอย
            cube.position.y = 0.5 + Math.sin(elapsed * 1.5) * 0.2;
            
            // หมุน sphere group (orbit)
            sphereGroup.rotation.y += delta * 0.5;
            
            // pulse point light
            pointLight.intensity = 2 + Math.sin(elapsed * 3) * 0.5;
            
            // อัปเดต controls
            controls.update();
            
            // อัปเดต UI
            updateUI();
            
            // Render
            renderer.render(scene, camera);
        }
        
        animate();
        
        // ================================================================
        // ข้อความต้อนรับ
        // ================================================================
        console.log('%c🎉 Three.js Scene Ready!', 'font-size: 16px; color: #6c63ff;');
        console.log('Press H: Toggle Helpers');
        console.log('Press W: Toggle Wireframe');
        console.log('Press F: Toggle Fog');
    </script>
</body>
</html>
```

---

## 12. Variations และการทดลอง

### Variation 1: Orthographic Camera

```javascript
// OrthographicCamera ไม่มี perspective distortion
// ดูเหมือน 2D (เหมาะสำหรับ CAD, isometric games)
const aspect = window.innerWidth / window.innerHeight;
const frustumSize = 10;

const orthoCamera = new THREE.OrthographicCamera(
    frustumSize * aspect / -2, // left
    frustumSize * aspect / 2,  // right
    frustumSize / 2,           // top
    frustumSize / -2,          // bottom
    0.1,                       // near
    100                        // far
);
orthoCamera.position.set(5, 5, 5);
orthoCamera.lookAt(0, 0, 0);
```

### Variation 2: Multiple Renders

```javascript
// Render ไปยัง 2 viewport (split screen)
function renderSplitScreen() {
    const width = window.innerWidth / 2;
    const height = window.innerHeight;
    
    // Left viewport: Perspective Camera
    renderer.setScissor(0, 0, width, height);
    renderer.setViewport(0, 0, width, height);
    renderer.setScissorTest(true);
    camera.aspect = width / height;
    camera.updateProjectionMatrix();
    renderer.render(scene, camera);
    
    // Right viewport: Orthographic Camera
    renderer.setScissor(width, 0, width, height);
    renderer.setViewport(width, 0, width, height);
    orthoCamera.left = frustumSize * (width / height) / -2;
    orthoCamera.right = frustumSize * (width / height) / 2;
    orthoCamera.updateProjectionMatrix();
    renderer.render(scene, orthoCamera);
}
```

### Variation 3: Canvas Texture Background

```javascript
// สร้าง gradient background ด้วย Canvas API
function createGradientTexture() {
    const canvas = document.createElement('canvas');
    canvas.width = 2;
    canvas.height = 512;
    
    const ctx = canvas.getContext('2d');
    const gradient = ctx.createLinearGradient(0, 0, 0, 512);
    gradient.addColorStop(0, '#0d1117');
    gradient.addColorStop(0.5, '#1a1a2e');
    gradient.addColorStop(1, '#16213e');
    
    ctx.fillStyle = gradient;
    ctx.fillRect(0, 0, 2, 512);
    
    const texture = new THREE.CanvasTexture(canvas);
    return texture;
}

scene.background = createGradientTexture();
```

---

## สรุป (Summary)

ใน Part 002 นี้เราได้เรียนรู้:

| Component | สิ่งที่เรียน |
|-----------|------------|
| **Scene** | Container หลัก, background, fog |
| **PerspectiveCamera** | fov, aspect, near/far, position, lookAt |
| **WebGLRenderer** | options, size, pixelRatio, shadows, toneMapping |
| **Mesh** | Geometry + Material = Object |
| **BoxGeometry** | parameters, segments, attributes |
| **MeshBasicMaterial** | color, transparent, opacity, side, wireframe |
| **Canvas** | วิธีใช้ canvas element |
| **Render Loop** | requestAnimationFrame, delta time, Clock |
| **Resize Handler** | อัปเดต camera + renderer เมื่อ resize |

---

## แบบฝึกหัด (Exercises)

### Exercise 1: Basic Scene (ง่าย)
สร้าง scene ที่มี:
- Background สีน้ำเงินเข้ม
- Cube สีเหลืองตรงกลาง
- Sphere สีแดงทางซ้าย
- Cone สีเขียวทางขวา
- Grid helper บนพื้น

### Exercise 2: Animation (ง่าย-ปานกลาง)
ทำให้:
- Cube หมุนรอบแกน Y ที่ความเร็ว 1 radian/second
- Sphere เคลื่อนที่ขึ้นลงแบบ sine wave
- Cone หมุนรอบ cube (orbit)

### Exercise 3: Interactivity (ปานกลาง)
เพิ่ม keyboard controls:
- Space: หยุด/เล่น animation
- R: Reset positions
- C: เปลี่ยนสีแบบสุ่ม

```javascript
// Hint:
let isPlaying = true;

window.addEventListener('keydown', (e) => {
    if (e.code === 'Space') {
        isPlaying = !isPlaying;
        e.preventDefault(); // ป้องกัน scroll
    }
});

function animate() {
    requestAnimationFrame(animate);
    
    if (isPlaying) {
        const delta = clock.getDelta();
        // ... animate
    } else {
        clock.getDelta(); // ยังต้อง call เพื่อ delta ไม่สะสม
    }
    
    renderer.render(scene, camera);
}
```

### Exercise 4: Performance Monitor (ปานกลาง)
เพิ่ม panel ที่แสดง:
- FPS
- Draw calls
- Triangle count
- Memory usage (renderer.info.memory.geometries, renderer.info.memory.textures)
อัปเดตทุก 1 วินาที

### Exercise 5: Screenshot Feature (ยาก)
เพิ่มปุ่ม "Save Screenshot" ที่:
1. หยุด animation ชั่วคราว
2. Render ด้วย resolution 2x
3. Download เป็น PNG
4. เล่น animation ต่อ

```javascript
// Hint:
// ต้องสร้าง renderer ใหม่ด้วย preserveDrawingBuffer: true
// หรือใช้ renderer.render() แล้ว toDataURL() ทันที
```

---

**ก่อนหน้า:** [Part 001 - Introduction to Three.js](part-001.md)  
**ต่อไป:** [Part 003 - Geometries Deep Dive](part-003.md)
