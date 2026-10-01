# Part 001: Introduction to Three.js - รู้จักกับ WebGL และ Three.js

## สารบัญบทเรียนนี้
- [1. 3D บนเว็บคืออะไร?](#1-3d-บนเว็บคืออะไร)
- [2. WebGL คืออะไร?](#2-webgl-คืออะไร)
- [3. Three.js คืออะไร?](#3-threejs-คืออะไร)
- [4. ประวัติความเป็นมา](#4-ประวัติความเป็นมา)
- [5. ทำไมต้องใช้ Three.js?](#5-ทำไมต้องใช้-threejs)
- [6. การติดตั้งและ Setup](#6-การติดตั้งและ-setup)
- [7. ระบบพิกัด 3 มิติ](#7-ระบบพิกัด-3-มิติ)
- [8. Right-Hand Rule](#8-right-hand-rule)
- [9. หน่วยใน Three.js](#9-หน่วยใน-threejs)
- [10. Browser Support & Performance](#10-browser-support--performance)
- [11. โปรเจกต์แรก: Hello Three.js](#11-โปรเจกต์แรก-hello-threejs)
- [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. 3D บนเว็บคืออะไร?

ก่อนที่เราจะเริ่มเรียน Three.js เราต้องเข้าใจก่อนว่า "3D บนเว็บ" นั้นทำงานอย่างไร

### คอมพิวเตอร์กราฟิกส์พื้นฐาน

ในโลกแห่งความเป็นจริง เราเห็นสิ่งของในสามมิติ แต่หน้าจอคอมพิวเตอร์เป็นแค่สองมิติ  
กระบวนการที่แปลงโลก 3D ให้กลายเป็นภาพ 2D บนหน้าจอเรียกว่า **Rendering (การเรนเดอร์)**

ขั้นตอนหลักของ Rendering มีดังนี้:

1. **Vertex Processing** - ประมวลผลจุดยอด (vertex) ทุกจุดของ 3D object
2. **Rasterization** - แปลงรูปทรง 3D เป็น pixel บนหน้าจอ
3. **Fragment Processing** - กำหนดสีของแต่ละ pixel
4. **Output** - แสดงผลบนหน้าจอ

### GPU คืออะไรและทำไมมันสำคัญ?

**GPU (Graphics Processing Unit)** คือชิปประมวลผลพิเศษที่ถูกออกแบบมาเพื่อคำนวณกราฟิกโดยเฉพาะ  
ต่างจาก CPU ที่มีไม่กี่ core แต่ทรงพลัง GPU มี core หลายพันตัวที่ทำงานพร้อมกัน (parallel processing)

เปรียบเทียบได้ว่า:
- **CPU** = นักคณิตศาสตร์ 4 คนที่เก่งมากๆ
- **GPU** = นักเรียน 4,000 คนที่ทำงานพร้อมกัน

สำหรับการ render 3D ซึ่งต้องคำนวณ pixel หลายล้านตัวพร้อมกัน GPU จึงเหมาะสมกว่ามาก

---

## 2. WebGL คืออะไร?

**WebGL (Web Graphics Library)** คือ JavaScript API ที่ให้ browser สามารถใช้ GPU ในการ render 3D graphics ได้โดยตรง

WebGL ถูกพัฒนาจาก **OpenGL ES 2.0** ซึ่งเป็น standard สำหรับ 3D graphics บนอุปกรณ์ embedded

### WebGL 1.0 vs WebGL 2.0

| Feature | WebGL 1.0 | WebGL 2.0 |
|---------|-----------|-----------|
| ปีที่เปิดตัว | 2011 | 2017 |
| OpenGL base | OpenGL ES 2.0 | OpenGL ES 3.0 |
| Instancing | ไม่มี (ต้องใช้ extension) | มีในตัว |
| Multiple Render Targets | ไม่มี | มี |
| 3D Textures | ไม่มี | มี |
| Transform Feedback | ไม่มี | มี |
| รองรับ browser ปัจจุบัน | ทุก browser | ทุก browser สมัยใหม่ |

Three.js r165+ ใช้ WebGL 2.0 เป็น default

### การเขียน WebGL โดยตรง (Raw WebGL)

นี่คือตัวอย่างโค้ด WebGL ดิบๆ สำหรับการวาดสามเหลี่ยมง่ายๆ:

```javascript
// ตัวอย่างนี้แสดงให้เห็นว่า Raw WebGL ยุ่งยากแค่ไหน
// ไม่ต้องจำ แค่ดูเพื่อเข้าใจว่า Three.js ช่วยเราได้มากแค่ไหน

const canvas = document.getElementById('canvas');
const gl = canvas.getContext('webgl2');

// Vertex Shader Source Code (GLSL)
const vertexShaderSource = `#version 300 es
in vec4 a_position;
void main() {
    gl_Position = a_position;
}`;

// Fragment Shader Source Code (GLSL)
const fragmentShaderSource = `#version 300 es
precision highp float;
out vec4 outColor;
void main() {
    outColor = vec4(1, 0, 0, 1); // สีแดง
}`;

// สร้าง Shader
function createShader(gl, type, source) {
    const shader = gl.createShader(type);
    gl.shaderSource(shader, source);
    gl.compileShader(shader);
    if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
        console.error(gl.getShaderInfoLog(shader));
        gl.deleteShader(shader);
        return null;
    }
    return shader;
}

const vertexShader = createShader(gl, gl.VERTEX_SHADER, vertexShaderSource);
const fragmentShader = createShader(gl, gl.FRAGMENT_SHADER, fragmentShaderSource);

// สร้าง Program
const program = gl.createProgram();
gl.attachShader(program, vertexShader);
gl.attachShader(program, fragmentShader);
gl.linkProgram(program);

// ข้อมูล vertex ของสามเหลี่ยม
const positions = [
    0, 0.5,   // จุดบน
   -0.5, -0.5, // จุดล่างซ้าย
    0.5, -0.5, // จุดล่างขวา
];

// สร้าง Buffer และส่งข้อมูลไปยัง GPU
const positionBuffer = gl.createBuffer();
gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);
gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(positions), gl.STATIC_DRAW);

// สร้าง VAO
const vao = gl.createVertexArray();
gl.bindVertexArray(vao);

const positionAttributeLocation = gl.getAttribLocation(program, 'a_position');
gl.enableVertexAttribArray(positionAttributeLocation);
gl.vertexAttribPointer(positionAttributeLocation, 2, gl.FLOAT, false, 0, 0);

// Render
gl.viewport(0, 0, gl.canvas.width, gl.canvas.height);
gl.clearColor(0, 0, 0, 1);
gl.clear(gl.COLOR_BUFFER_BIT);
gl.useProgram(program);
gl.bindVertexArray(vao);
gl.drawArrays(gl.TRIANGLES, 0, 3);
```

**ว้าว!** โค้ดทั้งหมดนั้นเพื่อวาดแค่สามเหลี่ยมเดียว! และยังไม่มี camera, lighting, shadows หรืออะไรเลย

---

## 3. Three.js คืออะไร?

**Three.js** คือ JavaScript library ระดับสูง (high-level) ที่ห่อหุ้ม WebGL ไว้ ทำให้เราสามารถสร้าง 3D graphics บนเว็บได้ง่ายขึ้นมากโดยไม่ต้องเขียน WebGL โดยตรง

Three.js มีความสามารถหลักดังนี้:
- สร้างและจัดการฉาก 3D (Scene graph)
- จัดการ Geometry, Material, Light, Camera
- โหลด 3D models (GLTF, OBJ, FBX)
- ระบบ Animation
- Post-processing effects
- Shadow mapping
- Physics integration (ผ่าน library อื่น)

### Three.js ทำให้โค้ดง่ายขึ้นแค่ไหน?

โค้ดด้านบน (Raw WebGL) ทำแค่สามเหลี่ยม แต่ด้วย Three.js เราสร้าง cube 3D ที่มีสีสันได้ด้วย:

```javascript
import * as THREE from 'three';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer();

renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const geometry = new THREE.BoxGeometry(1, 1, 1);
const material = new THREE.MeshBasicMaterial({ color: 0xff0000 });
const cube = new THREE.Mesh(geometry, material);
scene.add(cube);

camera.position.z = 5;

function animate() {
    requestAnimationFrame(animate);
    cube.rotation.x += 0.01;
    cube.rotation.y += 0.01;
    renderer.render(scene, camera);
}
animate();
```

เพียง ~15 บรรทัด ได้ cube 3D ที่หมุนได้! นั่นคือพลังของ Three.js

---

## 4. ประวัติความเป็นมา

### Timeline ของ Three.js

**2010** - **Ricardo Cabello** (Mr.doob) เริ่มพัฒนา Three.js เป็น JavaScript library แรกๆ สำหรับ 3D บนเว็บ  
โดยตอนแรกรองรับ Canvas 2D และ SVG renderer ก่อนที่ WebGL จะพร้อมใช้งาน

**2011** - WebGL 1.0 กลายเป็น standard อย่างเป็นทางการ Three.js เพิ่ม WebGLRenderer

**2013** - Three.js กลายเป็น open source project ที่ได้รับความนิยมสูงสุดบน GitHub ในหมวด JavaScript

**2017** - WebGL 2.0 เปิดตัว Three.js เริ่มรองรับ features ใหม่

**2020** - Three.js ย้าย codebase ไปใช้ ES Modules และ TypeScript types

**2022** - เปลี่ยน versioning เป็น r140+ และเริ่มใช้ WebGL 2.0 เป็น default

**2024** - Three.js r165 มี NodeMaterial system ใหม่ที่รองรับ WebGPU ในอนาคต

### ทีมพัฒนา

Three.js เป็น open source project ที่มี contributor มากกว่า 1,500 คนทั่วโลก  
ปัจจุบัน maintain โดย core team ประมาณ 10 คน

---

## 5. ทำไมต้องใช้ Three.js?

### เหตุผลที่ควรเลือก Three.js

**1. Community ขนาดใหญ่**  
มี GitHub stars มากกว่า 100,000+ และมีตัวอย่างอย่างครบครัน

**2. Documentation ดี**  
threejs.org/docs มีเอกสารครบและ examples มากมาย

**3. Ecosystem สมบูรณ์**  
- drei (React Three Fiber) สำหรับ React
- rapier/cannon.js สำหรับ Physics
- postprocessing สำหรับ effects
- troika-three-text สำหรับ text rendering

**4. Production-ready**  
บริษัทใหญ่ๆ ใช้ Three.js จริง เช่น:
- Google (Google Arts & Culture)
- NASA (data visualization)
- Apple (product showcases)
- Spotify (music visualization)

**5. Performance ดี**  
Three.js optimize WebGL calls อย่างชาญฉลาด เช่น batching, caching shaders

### เปรียบเทียบกับ alternatives

| Library | ข้อดี | ข้อเสีย |
|---------|-------|---------|
| **Three.js** | ครบ, community ใหญ่, documentation ดี | API ระดับกลาง-สูง |
| **Babylon.js** | Microsoft, TypeScript native, เหมาะกับ games | หนักกว่า Three.js |
| **PlayCanvas** | Editor GUI มี, เหมาะ game dev | Cloud-based editor |
| **A-Frame** | VR/AR ง่าย, declarative HTML | จำกัดความยืดหยุ่น |
| **Raw WebGL** | Performance สูงสุด, control เต็มที่ | เขียนยากมาก |

---

## 6. การติดตั้งและ Setup

มี 3 วิธีหลักในการใช้ Three.js:

### วิธีที่ 1: CDN (เร็วที่สุดสำหรับทดลอง)

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Three.js CDN</title>
    <style>
        * { margin: 0; padding: 0; }
        body { overflow: hidden; background: #000; }
    </style>
</head>
<body>
    <!-- Import Three.js จาก CDN -->
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
        
        // โค้ดของเราจะอยู่ที่นี่
        console.log('Three.js version:', THREE.REVISION);
    </script>
</body>
</html>
```

### วิธีที่ 2: npm + Vite (แนะนำสำหรับ project จริง)

#### ขั้นตอนการติดตั้ง

```bash
# สร้าง project ใหม่ด้วย Vite
npm create vite@latest my-threejs-project -- --template vanilla

# เข้าไปใน directory
cd my-threejs-project

# ติดตั้ง dependencies
npm install

# ติดตั้ง Three.js
npm install three

# เริ่ม development server
npm run dev
```

#### โครงสร้างโปรเจกต์

```
my-threejs-project/
├── public/
│   └── textures/          # ไฟล์ texture
│       └── placeholder.png
├── src/
│   ├── main.js            # ไฟล์หลัก
│   ├── style.css          # CSS
│   └── utils/
│       └── helpers.js     # helper functions
├── index.html             # HTML หลัก
├── package.json           # dependencies
└── vite.config.js         # Vite configuration
```

#### package.json

```json
{
  "name": "my-threejs-project",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "three": "^0.165.0"
  },
  "devDependencies": {
    "vite": "^5.0.0"
  }
}
```

#### vite.config.js

```javascript
import { defineConfig } from 'vite';

export default defineConfig({
    // กำหนด base URL (สำคัญเมื่อ deploy ไปยัง subdirectory)
    base: '/',
    
    // กำหนด directory สำหรับ static assets
    publicDir: 'public',
    
    // Build options
    build: {
        // output directory
        outDir: 'dist',
        
        // แยก vendor bundle
        rollupOptions: {
            output: {
                manualChunks: {
                    // Three.js จะถูกแยกเป็น chunk ของตัวเอง
                    three: ['three'],
                }
            }
        }
    },
    
    // Server options สำหรับ development
    server: {
        port: 3000,         // port สำหรับ dev server
        open: true,         // เปิด browser อัตโนมัติ
        host: true,         // รองรับ local network
    }
});
```

#### index.html

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My Three.js Project</title>
    <link rel="stylesheet" href="/src/style.css">
</head>
<body>
    <!-- Three.js จะสร้าง canvas เองและ append เข้า body -->
    <!-- หรือเราสามารถกำหนด canvas ไว้ล่วงหน้าได้ -->
    <canvas id="webgl-canvas"></canvas>
    
    <script type="module" src="/src/main.js"></script>
</body>
</html>
```

#### src/style.css

```css
/* Reset CSS */
*, *::before, *::after {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
}

/* ทำให้ canvas เต็มหน้าจอ */
html, body {
    width: 100%;
    height: 100%;
    overflow: hidden;
    background-color: #000;
}

#webgl-canvas {
    display: block;
    width: 100%;
    height: 100%;
}
```

#### src/main.js (ไฟล์หลัก)

```javascript
// นำเข้า Three.js
import * as THREE from 'three';

// ===== SETUP =====

// 1. Scene - เป็นเหมือน "เวที" ที่วางทุกอย่าง
const scene = new THREE.Scene();
scene.background = new THREE.Color('#1a1a2e'); // สีพื้นหลัง

// 2. Camera - มุมมองของเรา
const camera = new THREE.PerspectiveCamera(
    75,                                    // fov (field of view) - มุมมอง 75 องศา
    window.innerWidth / window.innerHeight, // aspect ratio
    0.1,                                   // near plane - ระยะใกล้สุดที่เห็น
    100                                    // far plane - ระยะไกลสุดที่เห็น
);
camera.position.z = 5; // ถอยกล้องออกมา 5 หน่วย

// 3. Renderer - ตัว render ฉาก
const canvas = document.getElementById('webgl-canvas');
const renderer = new THREE.WebGLRenderer({ 
    canvas: canvas,
    antialias: true // เปิด anti-aliasing เพื่อขอบที่เรียบขึ้น
});
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); // รองรับ retina display

// ===== OBJECTS =====

// สร้าง cube
const geometry = new THREE.BoxGeometry(1, 1, 1); // กว้าง, สูง, ลึก
const material = new THREE.MeshBasicMaterial({ color: '#6c63ff' });
const cube = new THREE.Mesh(geometry, material);
scene.add(cube);

// ===== RESIZE HANDLER =====

window.addEventListener('resize', () => {
    // อัปเดต camera aspect ratio
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    
    // อัปเดต renderer size
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
});

// ===== ANIMATION LOOP =====

function animate() {
    // บอก browser ให้เรียก animate() ก่อน repaint ครั้งต่อไป
    requestAnimationFrame(animate);
    
    // หมุน cube
    cube.rotation.x += 0.005;
    cube.rotation.y += 0.01;
    
    // Render ฉาก
    renderer.render(scene, camera);
}

// เริ่ม animation loop
animate();
```

### วิธีที่ 3: แบบ Script Tag (แบบเก่า ไม่แนะนำ)

```html
<!-- ไม่แนะนำ แต่ยังใช้ได้ -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r165/three.min.js"></script>
<script>
    // Three.js จะอยู่ใน global namespace ในชื่อ THREE
    const scene = new THREE.Scene();
    // ...
</script>
```

---

## 7. ระบบพิกัด 3 มิติ

Three.js ใช้ **Right-handed coordinate system** (ระบบพิกัดมือขวา)

```
          Y
          |
          |
          |
          +-------X
         /
        /
       Z (ชี้มาหาเรา)
```

### แกน X, Y, Z หมายความว่าอะไร?

- **แกน X (ซ้าย-ขวา)** → ค่าบวกอยู่ทางขวา ค่าลบอยู่ทางซ้าย
- **แกน Y (บน-ล่าง)** → ค่าบวกอยู่ด้านบน ค่าลบอยู่ด้านล่าง
- **แกน Z (หน้า-หลัง)** → ค่าบวกชี้มาหาเรา (ออกจากหน้าจอ) ค่าลบชี้ออกไป

```javascript
// ตัวอย่างการจัดวาง object ในฉาก
const cube = new THREE.Mesh(geometry, material);

// วาง object ที่ตำแหน่งต่างๆ
cube.position.x = 2;   // เลื่อนไปทางขวา 2 หน่วย
cube.position.y = 1;   // ยกขึ้น 1 หน่วย
cube.position.z = -3;  // เลื่อนออกไปด้านหลัง 3 หน่วย

// หรือกำหนดพร้อมกันด้วย set()
cube.position.set(2, 1, -3);

// หรือใช้ Vector3
cube.position.copy(new THREE.Vector3(2, 1, -3));
```

### ความแตกต่างจากระบบอื่น

> **⚠️ ข้อควรระวัง:** ระบบ coordinate ของ 3D software ต่างกัน!
> - **Three.js**: Y-up (แกน Y ชี้ขึ้น)
> - **Blender**: Z-up (แกน Z ชี้ขึ้น) 
> - **Maya**: Y-up
> - **Unity**: Y-up (แต่ Z ชี้ออกไป ไม่ใช่มาหาเรา)

เมื่อ import model จาก Blender คุณจะเห็นว่า object หมุนไปด้วยเหตุนี้  
GLTFLoader จัดการเรื่องนี้ให้อัตโนมัติ แต่ต้องเข้าใจเพื่อแก้ปัญหาได้

---

## 8. Right-Hand Rule

**กฎมือขวา (Right-Hand Rule)** คือวิธีจำทิศทางการหมุน (rotation) ใน Three.js

วิธีใช้:
1. กำมือขวาให้นิ้วหัวแม่มือชี้ไปตามแกนที่ต้องการ
2. ทิศที่นิ้วอื่นๆ งอไปคือทิศหมุนบวก (positive rotation)

```javascript
const mesh = new THREE.Mesh(geometry, material);

// การหมุนแต่ละแกนใช้หน่วย Radian (ไม่ใช่ degree)
// 1 radian = 180/π ≈ 57.3 degrees

// หมุนรอบแกน X (กดหัว/ยกหัว)
mesh.rotation.x = Math.PI / 4;  // 45 องศา

// หมุนรอบแกน Y (หันซ้าย/ขวา)
mesh.rotation.y = Math.PI / 2;  // 90 องศา

// หมุนรอบแกน Z (เอียงซ้าย/ขวา)
mesh.rotation.z = Math.PI;      // 180 องศา

// Helper function สำหรับแปลง degree เป็น radian
function degToRad(degrees) {
    return degrees * (Math.PI / 180);
}

// หรือใช้ THREE.MathUtils.degToRad()
mesh.rotation.x = THREE.MathUtils.degToRad(45);
```

### ตารางค่า Radian ที่ใช้บ่อย

| Degrees | Radians | สัญลักษณ์ |
|---------|---------|-----------|
| 0° | 0 | 0 |
| 45° | π/4 | ~0.785 |
| 90° | π/2 | ~1.571 |
| 180° | π | ~3.141 |
| 270° | 3π/2 | ~4.712 |
| 360° | 2π | ~6.283 |

---

## 9. หน่วยใน Three.js

Three.js ไม่มีหน่วยตายตัว แต่มีแนวทางปฏิบัติดังนี้:

### หน่วยแบบ Real-world (แนะนำ)

**1 unit = 1 meter** เป็น convention ที่ใช้กันทั่วไป

```javascript
// ตัวอย่างการสร้างห้องขนาด 5x3 เมตร ในสนาม
const room = new THREE.BoxGeometry(5, 3, 4);  // กว้าง 5m, สูง 3m, ลึก 4m

// ตำแหน่งคน (สูง 1.7 เมตร ยืนอยู่บนพื้น)
camera.position.set(0, 1.7, 0);

// โต๊ะสูง 75 ซม.
const table = new THREE.BoxGeometry(1.5, 0.75, 0.8);
```

### เหตุผลที่ควรใช้ real-world units:

1. **Shadow ถูกต้อง** - Shadow map resolution คำนวณจากขนาด
2. **Physics ถูกต้อง** - ถ้าใช้ physics engine จะได้ behavior ที่ถูกต้อง
3. **Post-processing ถูกต้อง** - Depth of field, fog ขึ้นกับระยะทาง
4. **หาจุดบกพร่องง่ายกว่า** - เข้าใจง่ายเมื่อ debug

```javascript
// ตัวอย่าง: ระยะ near/far ของ camera ควรเหมาะสมกับขนาด scene
const camera = new THREE.PerspectiveCamera(
    75,
    window.innerWidth / window.innerHeight,
    0.1,   // near: 10 cm (ใกล้สุด)
    1000   // far: 1 km (ไกลสุด)
);
```

---

## 10. Browser Support & Performance

### Browser Support

| Browser | WebGL 2 | Notes |
|---------|---------|-------|
| Chrome 56+ | ✅ | แนะนำสำหรับ development |
| Firefox 51+ | ✅ | ดีมากสำหรับ WebGL |
| Safari 15+ | ✅ | ต้องการ macOS 12+ |
| Edge 79+ | ✅ | Chromium-based |
| Mobile Chrome | ✅ | |
| Mobile Safari | ✅ | iOS 15+ |

### ตรวจสอบ WebGL Support

```javascript
// ตรวจสอบก่อน render
function checkWebGLSupport() {
    try {
        const canvas = document.createElement('canvas');
        const gl = canvas.getContext('webgl2');
        if (!gl) {
            // ลอง WebGL 1
            const gl1 = canvas.getContext('webgl') || 
                         canvas.getContext('experimental-webgl');
            if (!gl1) {
                return { supported: false, version: 0 };
            }
            return { supported: true, version: 1 };
        }
        return { supported: true, version: 2 };
    } catch(e) {
        return { supported: false, version: 0, error: e };
    }
}

const webglInfo = checkWebGLSupport();
if (!webglInfo.supported) {
    document.body.innerHTML = '<p>Browser ของคุณไม่รองรับ WebGL</p>';
} else {
    console.log('WebGL version:', webglInfo.version);
    // เริ่ม Three.js
}
```

### Performance Basics

#### เทคนิคพื้นฐานเพื่อ performance ดี

```javascript
// 1. ตั้งค่า pixel ratio อย่างถูกต้อง
// อย่าใช้ device pixel ratio สูงเกิน 2 (ไม่คุ้มกับ performance ที่เสียไป)
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));

// 2. เปิด/ปิด features ตามความจำเป็น
renderer.shadowMap.enabled = true;       // เปิดเฉพาะเมื่อต้องการ shadows
renderer.shadowMap.type = THREE.PCFSoftShadowMap; // soft shadows

// 3. ใช้ frustum culling (เปิดโดย default)
mesh.frustumCulled = true;  // ไม่ render object ที่อยู่นอกมุมมอง

// 4. Dispose geometry และ material เมื่อไม่ใช้แล้ว
geometry.dispose();
material.dispose();
renderer.dispose();

// 5. ใช้ Stats.js เพื่อดู FPS
import Stats from 'three/addons/libs/stats.module.js';
const stats = new Stats();
document.body.appendChild(stats.dom);

function animate() {
    requestAnimationFrame(animate);
    stats.begin();
    renderer.render(scene, camera);
    stats.end();
}
```

---

## 11. โปรเจกต์แรก: Hello Three.js

ตอนนี้เราพร้อมแล้วที่จะสร้าง scene แรก! เราจะสร้างฉากที่มี:
- Cube สีม่วงที่หมุนได้
- สีพื้นหลังเข้ม
- Resize handler

### ไฟล์สมบูรณ์ (CDN Version)

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Part 001: Hello Three.js</title>
    <style>
        /* Reset ทุกอย่าง */
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        
        /* ทำให้เต็มหน้าจอและซ่อน scrollbar */
        html, body {
            width: 100%;
            height: 100%;
            overflow: hidden;
        }
        
        /* Canvas จะถูกสร้างโดย Three.js */
        canvas {
            display: block;
        }
        
        /* Info overlay */
        #info {
            position: fixed;
            top: 20px;
            left: 50%;
            transform: translateX(-50%);
            color: white;
            font-family: 'Segoe UI', sans-serif;
            font-size: 14px;
            background: rgba(0, 0, 0, 0.5);
            padding: 10px 20px;
            border-radius: 8px;
            backdrop-filter: blur(4px);
            pointer-events: none; /* ไม่ block mouse events */
        }
    </style>
</head>
<body>
    <div id="info">Part 001: Hello Three.js | Rotating Cube</div>
    
    <!-- Import map บอก browser ว่า 'three' คืออะไร -->
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
        
        // ============================================================
        // STEP 1: สร้าง Scene
        // Scene คือ "ฉาก" หรือ "โลก" ที่เราจะวางของต่างๆ
        // ============================================================
        const scene = new THREE.Scene();
        
        // ตั้งสีพื้นหลัง
        scene.background = new THREE.Color(0x1a1a2e);
        
        // เพิ่ม fog (หมอก) เพื่อ depth effect (optional)
        scene.fog = new THREE.Fog(0x1a1a2e, 10, 50);
        
        // ============================================================
        // STEP 2: สร้าง Camera
        // PerspectiveCamera = กล้องที่ใกล้ใหญ่ไกลเล็ก (เหมือนตามนุษย์)
        // ============================================================
        const camera = new THREE.PerspectiveCamera(
            75,                                     // fov (field of view): 75 องศา
            window.innerWidth / window.innerHeight, // aspect ratio: กว้าง/สูง
            0.1,                                    // near: ระยะใกล้สุดที่ render
            100                                     // far: ระยะไกลสุดที่ render
        );
        
        // ตั้งตำแหน่ง camera
        camera.position.set(2, 1.5, 4);
        
        // บอก camera ให้มองไปที่จุดกลาง (origin)
        camera.lookAt(0, 0, 0);
        
        // ============================================================
        // STEP 3: สร้าง Renderer
        // WebGLRenderer ใช้ GPU ในการ render
        // ============================================================
        const renderer = new THREE.WebGLRenderer({
            antialias: true,   // เปิด anti-aliasing (ขอบเรียบ แต่ช้าลงนิดหน่อย)
            alpha: false,      // ไม่ต้องการ transparent background
        });
        
        // กำหนดขนาด renderer ให้เต็มหน้าจอ
        renderer.setSize(window.innerWidth, window.innerHeight);
        
        // รองรับ retina/high-DPI display (แต่จำกัดไว้ที่ 2 เพื่อ performance)
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        
        // เพิ่ม canvas ของ renderer เข้าไปใน HTML
        document.body.appendChild(renderer.domElement);
        
        // ============================================================
        // STEP 4: สร้าง Object (Cube)
        // Mesh = Geometry + Material
        // ============================================================
        
        // Geometry: รูปร่างของ object
        const geometry = new THREE.BoxGeometry(
            1,   // width: ความกว้าง
            1,   // height: ความสูง
            1    // depth: ความลึก
        );
        
        // Material: วัสดุ/สีของ object
        const material = new THREE.MeshBasicMaterial({
            color: 0x6c63ff,    // สีม่วง
            wireframe: false     // แสดงเป็น solid ไม่ใช่ wireframe
        });
        
        // Mesh: รวม Geometry + Material
        const cube = new THREE.Mesh(geometry, material);
        
        // เพิ่ม cube เข้า scene
        scene.add(cube);
        
        // ============================================================
        // STEP 5: เพิ่ม Grid Helper (ช่วยให้เห็น ground plane)
        // ============================================================
        const gridHelper = new THREE.GridHelper(
            20,      // ขนาดรวม
            20,      // จำนวนช่อง
            0x444444, // สีเส้นกลาง
            0x333333  // สีเส้นทั่วไป
        );
        scene.add(gridHelper);
        
        // ============================================================
        // STEP 6: เพิ่ม Axes Helper (แสดง X Y Z axes)
        // Red = X, Green = Y, Blue = Z
        // ============================================================
        const axesHelper = new THREE.AxesHelper(3);
        scene.add(axesHelper);
        
        // ============================================================
        // STEP 7: Resize Handler
        // เมื่อหน้าต่างเปลี่ยนขนาด ต้องอัปเดต camera และ renderer
        // ============================================================
        window.addEventListener('resize', () => {
            // อัปเดต aspect ratio ของ camera
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix(); // ต้องเรียกหลังจากเปลี่ยน projection parameters
            
            // อัปเดตขนาดของ renderer
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        });
        
        // ============================================================
        // STEP 8: Animation Loop
        // requestAnimationFrame เรียก animate() ก่อน repaint ทุกครั้ง
        // โดยปกติจะเรียก 60 ครั้งต่อวินาที (60 FPS)
        // ============================================================
        
        // ตัวแปรสำหรับเวลา
        const clock = new THREE.Clock();
        
        function animate() {
            // ลงทะเบียนให้เรียก animate() ครั้งต่อไป
            requestAnimationFrame(animate);
            
            // เวลาที่ผ่านไปตั้งแต่ frame ที่แล้ว (seconds)
            const delta = clock.getDelta();
            
            // เวลาที่ผ่านไปทั้งหมด (seconds)
            const elapsed = clock.getElapsedTime();
            
            // หมุน cube
            cube.rotation.y += delta * 0.5; // หมุนรอบแกน Y 0.5 rad/s
            cube.rotation.x = Math.sin(elapsed * 0.5) * 0.3; // แกว่งขึ้นลง
            
            // ทำให้ cube ลอยขึ้นลง
            cube.position.y = Math.sin(elapsed) * 0.3;
            
            // Render ฉาก
            renderer.render(scene, camera);
        }
        
        // เริ่ม loop
        animate();
        
        // ============================================================
        // Debug: แสดง info ใน console
        // ============================================================
        console.log('Three.js version:', THREE.REVISION);
        console.log('Scene:', scene);
        console.log('Camera position:', camera.position);
    </script>
</body>
</html>
```

### ไฟล์สมบูรณ์ (Vite/npm Version)

```javascript
// src/main.js - Complete Version with all explanations

import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import Stats from 'three/addons/libs/stats.module.js';

// ============================================================
// CLASS-BASED APPROACH (แนะนำสำหรับ project ขนาดใหญ่)
// ============================================================

class App {
    constructor() {
        // Properties
        this.scene = null;
        this.camera = null;
        this.renderer = null;
        this.controls = null;
        this.stats = null;
        this.clock = new THREE.Clock();
        
        // Objects
        this.cube = null;
        
        // Bind methods
        this.animate = this.animate.bind(this);
        this.onResize = this.onResize.bind(this);
        
        // Initialize
        this.init();
    }
    
    init() {
        this.setupScene();
        this.setupCamera();
        this.setupRenderer();
        this.setupControls();
        this.setupStats();
        this.addObjects();
        this.addHelpers();
        this.addEventListeners();
        this.animate();
    }
    
    setupScene() {
        this.scene = new THREE.Scene();
        this.scene.background = new THREE.Color('#0d1117');
        
        // Fog
        this.scene.fog = new THREE.FogExp2(0x0d1117, 0.05);
    }
    
    setupCamera() {
        this.camera = new THREE.PerspectiveCamera(
            60,
            window.innerWidth / window.innerHeight,
            0.1,
            200
        );
        this.camera.position.set(3, 2, 5);
    }
    
    setupRenderer() {
        const canvas = document.querySelector('#webgl-canvas');
        
        this.renderer = new THREE.WebGLRenderer({
            canvas,
            antialias: true,
            powerPreference: 'high-performance' // ใช้ GPU ที่แรงที่สุด (สำคัญสำหรับ laptop)
        });
        
        this.renderer.setSize(window.innerWidth, window.innerHeight);
        this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        
        // เปิด shadow map
        this.renderer.shadowMap.enabled = true;
        this.renderer.shadowMap.type = THREE.PCFSoftShadowMap;
        
        // Tone mapping สำหรับ HDR rendering
        this.renderer.toneMapping = THREE.ACESFilmicToneMapping;
        this.renderer.toneMappingExposure = 1.0;
    }
    
    setupControls() {
        this.controls = new OrbitControls(this.camera, this.renderer.domElement);
        this.controls.enableDamping = true; // เพิ่ม inertia
        this.controls.dampingFactor = 0.05;
        this.controls.screenSpacePanning = false;
        this.controls.minDistance = 1;
        this.controls.maxDistance = 50;
        this.controls.maxPolarAngle = Math.PI / 2; // ไม่ให้กล้องลงใต้พื้น
    }
    
    setupStats() {
        this.stats = new Stats();
        this.stats.showPanel(0); // 0: fps, 1: ms, 2: mb
        document.body.appendChild(this.stats.dom);
    }
    
    addObjects() {
        // Main cube
        const geometry = new THREE.BoxGeometry(1, 1, 1);
        const material = new THREE.MeshStandardMaterial({
            color: '#6c63ff',
            metalness: 0.3,
            roughness: 0.4
        });
        
        this.cube = new THREE.Mesh(geometry, material);
        this.cube.castShadow = true;
        this.cube.receiveShadow = true;
        this.scene.add(this.cube);
        
        // Ground plane
        const groundGeometry = new THREE.PlaneGeometry(20, 20);
        const groundMaterial = new THREE.MeshStandardMaterial({
            color: '#1a1a2e',
            metalness: 0.0,
            roughness: 0.8
        });
        const ground = new THREE.Mesh(groundGeometry, groundMaterial);
        ground.rotation.x = -Math.PI / 2;
        ground.position.y = -0.5;
        ground.receiveShadow = true;
        this.scene.add(ground);
        
        // Add lights
        const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
        this.scene.add(ambientLight);
        
        const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
        directionalLight.position.set(5, 5, 5);
        directionalLight.castShadow = true;
        this.scene.add(directionalLight);
    }
    
    addHelpers() {
        const axesHelper = new THREE.AxesHelper(3);
        this.scene.add(axesHelper);
        
        const gridHelper = new THREE.GridHelper(20, 20, '#444', '#333');
        gridHelper.position.y = -0.5;
        this.scene.add(gridHelper);
    }
    
    addEventListeners() {
        window.addEventListener('resize', this.onResize);
    }
    
    onResize() {
        this.camera.aspect = window.innerWidth / window.innerHeight;
        this.camera.updateProjectionMatrix();
        this.renderer.setSize(window.innerWidth, window.innerHeight);
        this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    }
    
    animate() {
        requestAnimationFrame(this.animate);
        
        this.stats.begin();
        
        const delta = this.clock.getDelta();
        const elapsed = this.clock.getElapsedTime();
        
        // Animate cube
        if (this.cube) {
            this.cube.rotation.y += delta * 0.5;
            this.cube.position.y = Math.sin(elapsed) * 0.3;
        }
        
        // Update controls (needed for damping)
        this.controls.update();
        
        // Render
        this.renderer.render(this.scene, this.camera);
        
        this.stats.end();
    }
    
    // Cleanup method
    dispose() {
        window.removeEventListener('resize', this.onResize);
        this.renderer.dispose();
        this.scene.traverse((object) => {
            if (object.geometry) object.geometry.dispose();
            if (object.material) {
                if (Array.isArray(object.material)) {
                    object.material.forEach(m => m.dispose());
                } else {
                    object.material.dispose();
                }
            }
        });
    }
}

// เริ่มต้นแอป
const app = new App();
```

---

## สรุป (Summary)

ใน Part 001 นี้เราเรียนรู้:

1. **3D Graphics Pipeline** - ขั้นตอนการ render จาก 3D ไปยัง 2D
2. **WebGL** - API ระดับต่ำที่ Three.js ใช้
3. **Three.js** คืออะไรและทำไมถึงควรใช้
4. **การติดตั้ง** ทั้งแบบ CDN และ npm + Vite
5. **ระบบพิกัด** X, Y, Z และ Right-Hand Rule
6. **หน่วยใน Three.js** - ใช้ 1 unit = 1 meter
7. **Browser Support** - WebGL 2 รองรับ browser สมัยใหม่ทั้งหมด
8. **Performance Basics** - pixel ratio, dispose, frustum culling

---

## แบบฝึกหัด (Exercises)

### Exercise 1: Setup (ระดับ: ง่าย)
ติดตั้ง Vite project และทำให้ cube หมุนได้ตาม code ด้านบน

### Exercise 2: Customize (ระดับ: ง่าย)
ปรับแต่ง scene โดย:
- เปลี่ยนสีพื้นหลัง
- เปลี่ยนสีของ cube
- เปลี่ยนความเร็วการหมุน
- เพิ่ม cube อีก 2 ลูกในตำแหน่งต่างกัน

### Exercise 3: Exploration (ระดับ: ปานกลาง)
เพิ่ม `THREE.SphereGeometry` และ `THREE.ConeGeometry` เข้าไปในฉาก  
วางแต่ละอันในตำแหน่งต่างกัน และทำให้หมุนด้วยความเร็วต่างกัน

### Exercise 4: Research (ระดับ: ปานกลาง)
ไปที่ [threejs.org/examples](https://threejs.org/examples) และหา 3 ตัวอย่างที่คิดว่า impressive  
อธิบายว่าแต่ละตัวอย่างทำอะไรและใช้เทคนิคอะไร

### Exercise 5: Debug (ระดับ: ยาก)
โค้ดด้านล่างมี bug 3 จุด หาและแก้ให้ถูกต้อง:

```javascript
import * as THREE from 'three';

// Bug 1: ?
const scene = new Scene();

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.z = 5;

const renderer = new THREE.WebGLRenderer();
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// Bug 2: ?
const geometry = new THREE.BoxGeometry();
const material = new THREE.MeshBasicMaterial({ color: 'purple' });
const cube = new THREE.Mesh(geometry, material);

// Bug 3: ?
scene.add(mesh);

function animate() {
    requestAnimationFrame(animate);
    cube.rotation.y += 0.01;
    renderer.render(scene, camera);
}
animate();
```

<details>
<summary>ดูเฉลย (คลิกเพื่อเปิด)</summary>

**Bug 1:** `new Scene()` ต้องเป็น `new THREE.Scene()`  
**Bug 2:** ไม่มี bug ที่นี่ (BoxGeometry ไม่ต้องระบุขนาดก็ได้ จะใช้ default 1x1x1)  
**Bug 3:** `scene.add(mesh)` ต้องเป็น `scene.add(cube)` เพราะตัวแปรชื่อ `cube`

</details>

---

## แหล่งเรียนรู้เพิ่มเติม (Additional Resources)

- [Three.js Official Documentation](https://threejs.org/docs/)
- [Three.js Examples](https://threejs.org/examples/)
- [Three.js on GitHub](https://github.com/mrdoob/three.js)
- [WebGL Fundamentals](https://webglfundamentals.org/)
- [The Book of Shaders](https://thebookofshaders.com/)

---

**ต่อไป:** [Part 002 - Your First 3D Scene](part-002.md)
