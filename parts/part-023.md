# Part 23: GLSL Shaders Introduction

## บทนำ: โลกของ Shaders

Shader คือโปรแกรมขนาดเล็กที่รันบน **GPU (Graphics Processing Unit)** โดยตรง เป็นหัวใจสำคัญของการสร้างภาพ 3D ที่สวยงามและมีประสิทธิภาพสูง ใน Three.js เราสามารถเขียน shader เองได้ด้วยภาษา **GLSL (OpenGL Shading Language)**

### GPU vs CPU: ทำไม GPU ถึงเร็วกว่าสำหรับ Graphics?

```
CPU (Central Processing Unit)
├── Cores: 4-64 cores
├── เหมาะสำหรับ: งานที่ซับซ้อน, sequential tasks
├── Cache: ขนาดใหญ่
└── ตัวอย่าง: Logic, AI, Physics calculations

GPU (Graphics Processing Unit)  
├── Cores: 1,000-10,000+ shader cores
├── เหมาะสำหรับ: Parallel processing, repetitive tasks
├── Cache: ขนาดเล็กแต่เร็วมาก
└── ตัวอย่าง: Rendering pixels, vertex transformations
```

**ทำไม GPU จึงเหมาะกับ Graphics?**

ลองนึกภาพหน้าจอ Full HD (1920x1080) = 2,073,600 pixels  
ทุก pixel ต้องคำนวณสีของตัวเองในทุก frame  
GPU สามารถคำนวณ pixels หลายล้านตัวพร้อมกันได้ในเวลาเดียวกัน

```javascript
// แนวคิด: ถ้าต้องคำนวณ 2 ล้าน pixels
// CPU: ทำทีละ 1 หรือทีละ 8-16 ตัว (ขึ้นอยู่กับ cores)
// GPU: ทำพร้อมกันได้หลายพันตัวต่อครั้ง

// นี่คือเหตุผลว่าทำไม shader programming จึงต้องคิดแบบ "parallel"
// แต่ละ shader ไม่รู้ว่า shader ตัวอื่นทำอะไรอยู่
```

---

## GLSL Language Basics

GLSL เป็นภาษาที่คล้าย C มาก แต่มีข้อจำกัดและความสามารถพิเศษของตัวเอง

### ประเภทข้อมูล (Data Types)

#### Scalar Types (ค่าเดี่ยว)
```glsl
// float - เลขทศนิยม (ใช้บ่อยที่สุด)
float temperature = 3.14;
float negative = -2.5;
float fromInt = float(5); // แปลงจาก int

// int - เลขจำนวนเต็ม
int count = 42;
int negative = -10;

// bool - ค่าบูลีน
bool isVisible = true;
bool isHidden = false;

// uint - unsigned integer (GLSL 3.0+)
uint positiveOnly = 255u;
```

#### Vector Types (เวกเตอร์)
```glsl
// vec2 - 2D vector (x, y)
vec2 uv = vec2(0.5, 0.5);
vec2 position2D = vec2(100.0, 200.0);

// vec3 - 3D vector (x, y, z) หรือ (r, g, b)
vec3 position = vec3(1.0, 2.0, 3.0);
vec3 color = vec3(1.0, 0.0, 0.0); // สีแดง

// vec4 - 4D vector (x, y, z, w) หรือ (r, g, b, a)
vec4 clipPos = vec4(0.0, 0.0, 0.0, 1.0);
vec4 colorAlpha = vec4(1.0, 0.5, 0.0, 1.0); // สีส้มทึบ

// ivec2, ivec3, ivec4 - integer vectors
ivec2 pixelPos = ivec2(100, 200);

// bvec2, bvec3, bvec4 - boolean vectors
bvec3 flags = bvec3(true, false, true);
```

#### Swizzling - การเข้าถึง Components
```glsl
vec4 color = vec4(1.0, 0.5, 0.25, 1.0);

// เข้าถึงด้วย xyzw
float x = color.x; // 1.0
float y = color.y; // 0.5
vec2 xy = color.xy; // vec2(1.0, 0.5)
vec3 xyz = color.xyz; // vec3(1.0, 0.5, 0.25)

// เข้าถึงด้วย rgba (เหมือนกัน แต่ใช้กับ color context)
float r = color.r; // 1.0
float g = color.g; // 0.5
vec3 rgb = color.rgb; // vec3(1.0, 0.5, 0.25)

// เข้าถึงด้วย stpq (ใช้กับ texture coordinates)
float s = color.s; // 1.0
vec2 st = color.st; // vec2(1.0, 0.5)

// Swizzling ที่น่าสนใจ
vec3 reversed = color.zyx; // vec3(0.25, 0.5, 1.0)
vec4 expanded = color.xxyy; // vec4(1.0, 1.0, 0.5, 0.5)
```

#### Matrix Types (เมทริกซ์)
```glsl
// mat2 - 2x2 matrix
mat2 rotation = mat2(
    cos(angle), -sin(angle),
    sin(angle),  cos(angle)
);

// mat3 - 3x3 matrix
mat3 normalMatrix; // ใช้สำหรับ transform normals

// mat4 - 4x4 matrix (ใช้บ่อยที่สุดใน 3D)
mat4 transform; // projection, view, model matrices

// การสร้าง matrix
mat4 identity = mat4(1.0); // Identity matrix
mat4 scale = mat4(
    2.0, 0.0, 0.0, 0.0,
    0.0, 2.0, 0.0, 0.0,
    0.0, 0.0, 2.0, 0.0,
    0.0, 0.0, 0.0, 1.0
);
```

### Variable Qualifiers (ตัวปรับแต่งตัวแปร)

```glsl
// uniform - ค่าที่ส่งจาก JavaScript, เหมือนกันทุก shader instance
uniform float u_time;       // เวลา
uniform vec2 u_resolution;  // ขนาดหน้าจอ
uniform sampler2D u_texture; // texture

// attribute - ข้อมูลต่อ vertex (เฉพาะ vertex shader)
attribute vec3 position;    // ตำแหน่ง vertex
attribute vec3 normal;      // normal vector
attribute vec2 uv;          // texture coordinates
attribute vec3 color;       // vertex color

// varying - ส่งข้อมูลจาก vertex ไป fragment shader
varying vec2 vUv;           // interpolated UV
varying vec3 vNormal;       // interpolated normal
varying vec3 vPosition;     // interpolated position

// const - ค่าคงที่
const float PI = 3.14159265359;
const int MAX_LIGHTS = 8;
```

---

## Built-in Functions

GLSL มี built-in functions จำนวนมากที่เพิ่มประสิทธิภาพ

### Math Functions
```glsl
// Trigonometry
float s = sin(angle);     // sine
float c = cos(angle);     // cosine
float t = tan(angle);     // tangent
float a = asin(x);        // arc sine
float b = acos(x);        // arc cosine
float d = atan(y, x);     // arc tangent (2 args version)

// Power & Exponential
float p = pow(base, exp); // base^exp
float e = exp(x);         // e^x
float l = log(x);         // natural log
float s = sqrt(x);        // square root
float i = inversesqrt(x); // 1/sqrt(x) - เร็วมาก!

// Common
float a = abs(x);         // absolute value |x|
float s = sign(x);        // -1, 0, or 1
float f = floor(x);       // ปัดลง
float c = ceil(x);        // ปัดขึ้น
float r = round(x);       // ปัดใกล้สุด
float fr = fract(x);      // ส่วนทศนิยม: x - floor(x)
float m = mod(x, y);      // remainder: x - y*floor(x/y)
float mn = min(a, b);     // ค่าน้อยกว่า
float mx = max(a, b);     // ค่ามากกว่า
float cl = clamp(x,mn,mx);// จำกัดค่าในช่วง [mn, mx]
```

### Interpolation Functions
```glsl
// mix - Linear interpolation (lerp)
// mix(a, b, t) = a*(1-t) + b*t
float result = mix(0.0, 1.0, 0.5); // = 0.5
vec3 color = mix(red, blue, 0.3);  // 30% blue, 70% red

// step - Threshold function
// step(edge, x) = 0 ถ้า x < edge, 1 ถ้า x >= edge
float s = step(0.5, 0.3); // = 0.0 (0.3 < 0.5)
float s2 = step(0.5, 0.7); // = 1.0 (0.7 >= 0.5)

// smoothstep - Smooth threshold
// smoothstep(edge0, edge1, x)
// = 0 ถ้า x <= edge0
// = 1 ถ้า x >= edge1
// = smooth curve ระหว่าง edge0 ถึง edge1
float ss = smoothstep(0.2, 0.8, 0.5); // smooth S-curve
```

### Vector Functions
```glsl
vec3 a = vec3(1.0, 2.0, 3.0);
vec3 b = vec3(4.0, 5.0, 6.0);

// length - ความยาวของ vector
float len = length(a); // sqrt(1^2 + 2^2 + 3^2)

// dot product - scalar
float d = dot(a, b); // 1*4 + 2*5 + 3*6 = 32

// cross product - vector perpendicular to both
vec3 c = cross(a, b); // vector ตั้งฉากกับ a และ b

// normalize - unit vector (ความยาว = 1)
vec3 n = normalize(a); // a / length(a)

// distance - ระยะห่างระหว่าง 2 จุด
float dist = distance(a, b);

// reflect - สะท้อน vector รอบ normal
vec3 reflected = reflect(incident, normal);

// refract - หักเหแสง
vec3 refracted = refract(incident, normal, eta);
```

---

## Vertex Shader vs Fragment Shader

### Vertex Shader คืออะไร?

```
Input: Vertex data (position, normal, uv, color, ...)
↓
Vertex Shader (รันทุก vertex)
↓
Output: gl_Position (clip space position)
        + varyings (ข้อมูลส่งไป fragment shader)
↓
Rasterization (GPU แปลง triangles เป็น fragments/pixels)
↓
Fragment Shader
```

**Vertex Shader หน้าที่หลัก:**
1. แปลง position จาก object space → clip space
2. ส่งข้อมูลไปยัง fragment shader ผ่าน varyings

```glsl
// ตัวอย่าง Vertex Shader พื้นฐาน
void main() {
    // gl_Position: output ที่ต้องมีเสมอ
    // เป็น position ใน clip space (4D)
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    
    // ส่ง UV ไปให้ fragment shader
    vUv = uv;
}
```

### Fragment Shader คืออะไร?

Fragment shader รันสำหรับทุก **fragment** (เกือบเท่ากับ pixel) ที่ถูก rasterize มาจาก triangles

```
Input: varyings จาก vertex shader (interpolated)
       uniforms
       gl_FragCoord (ตำแหน่งบนหน้าจอ)
↓
Fragment Shader (รันทุก fragment/pixel)
↓
Output: gl_FragColor (สีของ pixel นั้น)
```

```glsl
// ตัวอย่าง Fragment Shader พื้นฐาน
precision mediump float;

varying vec2 vUv;

void main() {
    // gl_FragColor: output สีของ pixel
    // vec4(r, g, b, a) - ค่าระหว่าง 0.0 - 1.0
    gl_FragColor = vec4(1.0, 0.0, 0.0, 1.0); // สีแดงทึบ
}
```

---

## ShaderMaterial ใน Three.js

```javascript
const material = new THREE.ShaderMaterial({
    // Vertex Shader code (string)
    vertexShader: `
        void main() {
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
    `,
    // Fragment Shader code (string)
    fragmentShader: `
        void main() {
            gl_FragColor = vec4(1.0, 0.0, 0.0, 1.0);
        }
    `,
    // Uniforms: ข้อมูลที่ส่งจาก JavaScript
    uniforms: {
        uColor: { value: new THREE.Color(0xff0000) },
        uTime: { value: 0.0 }
    }
});
```

### ShaderMaterial Built-in Uniforms

Three.js ส่ง uniforms เหล่านี้ให้อัตโนมัติเมื่อใช้ `ShaderMaterial`:

```glsl
// Matrices
uniform mat4 projectionMatrix;   // Projection matrix
uniform mat4 modelViewMatrix;    // Model * View matrix (combined)
uniform mat4 modelMatrix;        // Model matrix (world transform)
uniform mat4 viewMatrix;         // View matrix (camera)
uniform mat3 normalMatrix;       // Normal transform matrix

// Attributes (ต้องประกาศเอง)
attribute vec3 position;    // Vertex position
attribute vec3 normal;      // Vertex normal
attribute vec2 uv;          // UV coordinates
attribute vec3 color;       // Vertex color (ถ้า geometry มี)
```

---

## Coordinate Systems ใน Shaders

เข้าใจ coordinate systems เป็นสิ่งจำเป็นสำหรับ shader programming

```
Object Space (Local Space)
    ↓  × modelMatrix
World Space
    ↓  × viewMatrix  
Camera Space (View Space)
    ↓  × projectionMatrix
Clip Space
    ↓  Perspective Division (÷ w)
NDC (Normalized Device Coordinates) [-1, 1]
    ↓  Viewport Transform
Screen Space (Pixels)
```

```glsl
// ใน Vertex Shader:

// Object Space position
vec3 objectPos = position;  // local coordinates

// World Space position
vec4 worldPos = modelMatrix * vec4(position, 1.0);

// Camera Space position  
vec4 cameraPos = viewMatrix * worldPos;
// หรือ
vec4 cameraPos2 = modelViewMatrix * vec4(position, 1.0);

// Clip Space position (ส่งเป็น output)
gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);

// NDC: GPU จะหาร x,y,z ด้วย w ให้อัตโนมัติ
```

---

## Demo 1: First Custom Shader

สร้าง shader แรกที่แสดงสีพื้นฐาน

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>First Custom Shader</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        canvas { display: block; }
    </style>
</head>
<body>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';

// Scene Setup
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
document.body.appendChild(renderer.domElement);
camera.position.z = 2;

// Vertex Shader
const vertexShader = `
    // ShaderMaterial ให้ built-in uniforms เหล่านี้อัตโนมัติ:
    // uniform mat4 projectionMatrix;
    // uniform mat4 modelViewMatrix;
    // attribute vec3 position;
    // attribute vec2 uv;
    
    // Varying: ส่งข้อมูลไป Fragment Shader
    varying vec2 vUv;
    varying vec3 vPosition;
    
    void main() {
        // บันทึก UV ไว้ใช้ใน fragment shader
        vUv = uv;
        
        // บันทึก position (object space) ไว้ใช้ใน fragment shader
        vPosition = position;
        
        // Transform position ไป clip space
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
`;

// Fragment Shader
const fragmentShader = `
    // precision บอก GPU ว่าให้ใช้ความแม่นยำระดับไหน
    precision mediump float;
    
    // รับ varyings จาก vertex shader
    varying vec2 vUv;
    varying vec3 vPosition;
    
    void main() {
        // ใช้ UV coordinates เป็นสี (u=red, v=green)
        vec3 color = vec3(vUv.x, vUv.y, 0.5);
        
        // gl_FragColor = vec4(r, g, b, alpha)
        gl_FragColor = vec4(color, 1.0);
    }
`;

// Create Mesh
const geometry = new THREE.PlaneGeometry(2, 2);
const material = new THREE.ShaderMaterial({
    vertexShader,
    fragmentShader
});

const mesh = new THREE.Mesh(geometry, material);
scene.add(mesh);

// Animation Loop
function animate() {
    requestAnimationFrame(animate);
    renderer.render(scene, camera);
}
animate();

// Resize Handler
window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});
</script>
</body>
</html>
```

---

## Demo 2: Color Gradient Shader

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Color Gradient Shader</title>
    <style>
        * { margin: 0; padding: 0; }
        canvas { display: block; }
        #info {
            position: fixed;
            top: 20px;
            left: 20px;
            color: white;
            font-family: monospace;
            font-size: 14px;
            background: rgba(0,0,0,0.7);
            padding: 10px;
            border-radius: 5px;
        }
    </style>
</head>
<body>
<div id="info">Color Gradient Shader Demo<br>ลาก mouse เพื่อเปลี่ยนสี</div>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, innerWidth / innerHeight, 0.1, 100);
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
document.body.appendChild(renderer.domElement);
camera.position.z = 1.5;

// Mouse position tracking
const mouse = new THREE.Vector2(0.5, 0.5);
document.addEventListener('mousemove', (e) => {
    mouse.x = e.clientX / innerWidth;
    mouse.y = 1.0 - (e.clientY / innerHeight); // invert Y
});

// Gradient Shader
const vertexShader = `
    varying vec2 vUv;
    void main() {
        vUv = uv;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
`;

const fragmentShader = `
    precision mediump float;
    
    uniform vec2 uMouse;    // Mouse position (0-1)
    uniform float uTime;     // Time
    
    varying vec2 vUv;
    
    // Function: สร้าง gradient แบบ radial
    float radialGradient(vec2 uv, vec2 center, float radius) {
        float dist = distance(uv, center);
        return 1.0 - smoothstep(0.0, radius, dist);
    }
    
    void main() {
        // สี 3 จุด
        vec3 colorA = vec3(0.8, 0.1, 0.9);  // ม่วง
        vec3 colorB = vec3(0.1, 0.4, 0.9);  // น้ำเงิน
        vec3 colorC = vec3(0.9, 0.3, 0.1);  // ส้ม
        
        // Gradient แนวนอน
        float t1 = vUv.x;
        vec3 horizontal = mix(colorA, colorB, t1);
        
        // Gradient แนวตั้ง
        float t2 = vUv.y;
        vec3 vertical = mix(colorC, horizontal, t2);
        
        // Radial gradient จาก mouse position
        float radial = radialGradient(vUv, uMouse, 0.4);
        
        // รวมกัน
        vec3 finalColor = mix(vertical, vec3(1.0), radial * 0.3);
        
        // เพิ่ม vignette (มืดขอบ)
        float vignette = radialGradient(vUv, vec2(0.5, 0.5), 0.7);
        finalColor *= 0.3 + vignette * 0.7;
        
        gl_FragColor = vec4(finalColor, 1.0);
    }
`;

const geometry = new THREE.PlaneGeometry(3, 3);
const material = new THREE.ShaderMaterial({
    vertexShader,
    fragmentShader,
    uniforms: {
        uMouse: { value: mouse },
        uTime: { value: 0.0 }
    }
});

scene.add(new THREE.Mesh(geometry, material));

const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    material.uniforms.uTime.value = clock.getElapsedTime();
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

---

## Demo 3: Moving/Animated Shader

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Moving Shader Demo</title>
    <style>
        * { margin: 0; padding: 0; }
        canvas { display: block; }
    </style>
</head>
<body>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x000000);
const camera = new THREE.PerspectiveCamera(75, innerWidth / innerHeight, 0.1, 100);
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
document.body.appendChild(renderer.domElement);
camera.position.z = 2;

const vertexShader = `
    varying vec2 vUv;
    varying vec3 vPosition;
    
    void main() {
        vUv = uv;
        vPosition = position;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
`;

const fragmentShader = `
    precision highp float;
    
    uniform float uTime;
    uniform vec2 uResolution;
    
    varying vec2 vUv;
    
    // ฟังก์ชัน: Hash function สำหรับ pseudo-random
    float hash(vec2 p) {
        p = fract(p * vec2(234.34, 435.345));
        p += dot(p, p + 34.23);
        return fract(p.x * p.y);
    }
    
    // ฟังก์ชัน: สร้าง wave pattern
    float wave(vec2 uv, float time, float freq, float speed, float amplitude) {
        return sin(uv.x * freq + time * speed) * amplitude;
    }
    
    void main() {
        vec2 uv = vUv;
        float t = uTime;
        
        // --- Wave Pattern ---
        
        // Wave 1: ใหญ่ช้า
        float w1 = wave(uv, t, 8.0, 0.5, 0.1);
        
        // Wave 2: เล็กเร็ว
        float w2 = wave(uv, t, 20.0, 1.5, 0.05);
        
        // Wave 3: แนวทแยง
        float w3 = sin(uv.x * 10.0 + uv.y * 10.0 + t * 0.8) * 0.05;
        
        // รวม waves
        float waves = w1 + w2 + w3;
        
        // สร้าง "band" จาก wave
        float band = sin((uv.y + waves) * 20.0 + t * 0.5) * 0.5 + 0.5;
        
        // --- Color Palette ---
        // palette technique: cosine color palette
        // color = a + b * cos(2π(c*t + d))
        vec3 a = vec3(0.5, 0.5, 0.5);
        vec3 b = vec3(0.5, 0.5, 0.5);
        vec3 c = vec3(1.0, 1.0, 1.0);
        vec3 d = vec3(0.00, 0.33, 0.67);
        
        float palIndex = band + t * 0.1;
        vec3 color = a + b * cos(6.28318 * (c * palIndex + d));
        
        // เพิ่มความสว่างตาม wave intensity
        float brightness = 0.3 + abs(sin(t * 0.5)) * 0.7;
        color *= brightness;
        
        // Glow effect
        float glow = smoothstep(0.45, 0.5, band) - smoothstep(0.5, 0.55, band);
        color += glow * vec3(1.0, 0.8, 0.5) * 2.0;
        
        gl_FragColor = vec4(color, 1.0);
    }
`;

const geometry = new THREE.PlaneGeometry(4, 4);
const material = new THREE.ShaderMaterial({
    vertexShader,
    fragmentShader,
    uniforms: {
        uTime: { value: 0.0 },
        uResolution: { value: new THREE.Vector2(innerWidth, innerHeight) }
    }
});

scene.add(new THREE.Mesh(geometry, material));

const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    material.uniforms.uTime.value = clock.getElapsedTime();
    renderer.render(scene, camera);
}
animate();

window.addEventListener('resize', () => {
    camera.aspect = innerWidth / innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(innerWidth, innerHeight);
    material.uniforms.uResolution.value.set(innerWidth, innerHeight);
});
</script>
</body>
</html>
```

---

## การใช้ GLSL กับ Sphere

```javascript
// ตัวอย่างการใช้ shader กับ 3D sphere
const sphereGeometry = new THREE.SphereGeometry(1, 64, 64);

const sphereVertexShader = `
    varying vec2 vUv;
    varying vec3 vNormal;
    varying vec3 vWorldPosition;
    
    void main() {
        vUv = uv;
        vNormal = normalize(normalMatrix * normal);
        
        // คำนวณ world position
        vec4 worldPos = modelMatrix * vec4(position, 1.0);
        vWorldPosition = worldPos.xyz;
        
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
`;

const sphereFragmentShader = `
    precision mediump float;
    
    uniform float uTime;
    uniform vec3 uLightDir;
    
    varying vec2 vUv;
    varying vec3 vNormal;
    varying vec3 vWorldPosition;
    
    void main() {
        // Simple Diffuse Lighting
        vec3 lightDir = normalize(uLightDir);
        float diffuse = max(0.0, dot(vNormal, lightDir));
        
        // Base color จาก UV
        vec3 baseColor = vec3(0.2 + vUv.x * 0.8, 0.4, 0.6 + vUv.y * 0.4);
        
        // Animated stripe pattern
        float stripes = sin(vUv.x * 20.0 + uTime) * 0.5 + 0.5;
        baseColor = mix(baseColor, vec3(1.0, 0.8, 0.2), stripes * 0.3);
        
        // Apply lighting
        vec3 ambient = baseColor * 0.3;
        vec3 diffuseColor = baseColor * diffuse * 0.7;
        
        // Specular (Blinn-Phong)
        vec3 viewDir = normalize(cameraPosition - vWorldPosition);
        vec3 halfDir = normalize(lightDir + viewDir);
        float specular = pow(max(0.0, dot(vNormal, halfDir)), 32.0);
        
        vec3 finalColor = ambient + diffuseColor + vec3(specular * 0.5);
        
        gl_FragColor = vec4(finalColor, 1.0);
    }
`;

const sphereMaterial = new THREE.ShaderMaterial({
    vertexShader: sphereVertexShader,
    fragmentShader: sphereFragmentShader,
    uniforms: {
        uTime: { value: 0.0 },
        uLightDir: { value: new THREE.Vector3(1.0, 1.0, 1.0) }
    }
});
```

---

## gl_Position และ gl_FragColor ในเชิงลึก

### gl_Position
```glsl
// gl_Position คือ vec4 (x, y, z, w)
// x, y: ตำแหน่งบนหน้าจอใน NDC (-1 ถึง 1)
// z: depth (0 ถึง 1 หลัง perspective divide)
// w: perspective divide factor

// Standard transform
gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);

// ปรับตำแหน่งด้วยตัวเอง (เลื่อน)
vec4 pos = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
pos.x += 0.5; // เลื่อนไปทางขวา (clip space)
gl_Position = pos;

// ปรับขนาด
gl_Position = projectionMatrix * modelViewMatrix * vec4(position * 2.0, 1.0);
```

### gl_FragColor
```glsl
// gl_FragColor คือ vec4 (r, g, b, a)
// ค่าระหว่าง 0.0 - 1.0

// สีพื้นฐาน
gl_FragColor = vec4(1.0, 0.0, 0.0, 1.0);    // แดง
gl_FragColor = vec4(0.0, 1.0, 0.0, 1.0);    // เขียว
gl_FragColor = vec4(0.0, 0.0, 1.0, 1.0);    // น้ำเงิน
gl_FragColor = vec4(1.0, 1.0, 1.0, 1.0);    // ขาว
gl_FragColor = vec4(0.0, 0.0, 0.0, 1.0);    // ดำ

// Transparent
gl_FragColor = vec4(1.0, 0.0, 0.0, 0.5);    // แดงกึ่งโปร่งใส

// ใช้ vec3 แล้วเพิ่ม alpha
vec3 color = vec3(0.5, 0.2, 0.8);
gl_FragColor = vec4(color, 1.0);
```

---

## Debug Techniques สำหรับ Shaders

```glsl
// เทคนิค debug: ใช้สีแสดงค่าตัวแปร

// Debug float value (map 0-1 to grayscale)
float someValue = 0.7;
gl_FragColor = vec4(vec3(someValue), 1.0);

// Debug vec2 (map xy to rg)
vec2 someVec2 = vUv;
gl_FragColor = vec4(someVec2, 0.0, 1.0);

// Debug vec3 directly as color
vec3 someVec3 = vNormal * 0.5 + 0.5; // remap [-1,1] to [0,1]
gl_FragColor = vec4(someVec3, 1.0);

// Debug: แสดงเฉพาะ channel เดียว
float onlyRed = someValue;
gl_FragColor = vec4(onlyRed, 0.0, 0.0, 1.0);

// Debug: ค่า negative ด้วย abs
float maybeNegative = sin(vUv.x * 10.0);
gl_FragColor = vec4(abs(maybeNegative)); // แสดงแค่ขนาด

// Checkerboard debug pattern
float checker = mod(floor(vUv.x * 10.0) + floor(vUv.y * 10.0), 2.0);
gl_FragColor = vec4(vec3(checker), 1.0);
```

---

## GLSL Precision Qualifiers

```glsl
// Fragment shader ต้องประกาศ precision
precision highp float;   // ความแม่นยำสูงสุด (~7 decimal digits)
precision mediump float; // กลาง (~5 decimal digits) - พอสำหรับส่วนใหญ่
precision lowp float;    // ต่ำ (~3 decimal digits) - เร็วที่สุด แต่ไม่แม่น

// ใช้กับตัวแปรเดี่ยวก็ได้
lowp float colorComponent;
mediump vec2 uvCoord;
highp mat4 transformMatrix;

// Vertex shader: default highp float
// Fragment shader: ไม่มี default ต้องประกาศเอง

// แนะนำ: ใช้ mediump float สำหรับ fragment shader ทั่วไป
// ใช้ highp สำหรับ position calculations ที่ต้องการความแม่นยำ
```

---

## Common GLSL Patterns

### Pattern 1: UV-based Color
```glsl
void main() {
    // สร้างสีจาก UV coordinates
    vec3 color = vec3(vUv.x, vUv.y, sin(vUv.x * 3.14));
    gl_FragColor = vec4(color, 1.0);
}
```

### Pattern 2: Distance-based Effect
```glsl
void main() {
    // วงกลมตรงกลาง
    float dist = distance(vUv, vec2(0.5));
    float circle = 1.0 - smoothstep(0.0, 0.3, dist);
    gl_FragColor = vec4(circle, circle * 0.5, 0.0, 1.0);
}
```

### Pattern 3: Animated Gradient
```glsl
uniform float uTime;

void main() {
    float t = uTime * 0.5;
    vec3 colA = vec3(sin(t) * 0.5 + 0.5, 0.2, cos(t) * 0.5 + 0.5);
    vec3 colB = vec3(0.1, sin(t + 2.0) * 0.5 + 0.5, 0.9);
    vec3 color = mix(colA, colB, vUv.x);
    gl_FragColor = vec4(color, 1.0);
}
```

### Pattern 4: Tiled Pattern
```glsl
void main() {
    // สร้าง grid ด้วย fract
    vec2 tiledUv = fract(vUv * 5.0); // 5x5 grid
    vec2 centered = tiledUv - 0.5;   // center ใน tile
    
    // วงกลมใน tile
    float circle = 1.0 - smoothstep(0.2, 0.25, length(centered));
    
    gl_FragColor = vec4(circle, circle * 0.8, circle * 0.4, 1.0);
}
```

---

## สรุปสิ่งที่ต้องรู้เกี่ยวกับ GLSL

| หัวข้อ | Vertex Shader | Fragment Shader |
|--------|--------------|-----------------|
| รันต่อ | Vertex | Fragment (pixel) |
| Output | gl_Position | gl_FragColor |
| Input | attributes, uniforms | varyings, uniforms |
| ส่งข้อมูล | varyings → | → varyings |
| ความถี่ | น้อยกว่า (vertices) | มากกว่า (pixels) |

### สิ่งสำคัญที่ต้องจำ

1. **GLSL เป็น statically typed** - ต้องประกาศ type ทุกตัวแปร
2. **int ≠ float** - `1` ≠ `1.0` ต้องระวัง
3. **ไม่มี recursion** - ไม่สามารถเรียกฟังก์ชันตัวเองได้
4. **Loop ต้องรู้จำนวนรอบล่วงหน้า** - ใน GLSL เก่า loop count ต้องเป็น constant
5. **Vectors ทำ math แบบ component-wise** - `vec3(1,2,3) * vec3(4,5,6)` = `vec3(4,10,18)`
6. **Swizzling ใช้ได้กับ vector ทุกประเภท**
7. **Precision matter** - ใช้ `mediump` เป็น default ใน fragment shader

---

## แบบฝึกหัด (Exercises)

### ระดับ 1: พื้นฐาน
1. สร้าง shader ที่แสดงสีขาวล้วน จากนั้นเปลี่ยนเป็นสีน้ำเงิน
2. สร้าง shader ที่ใช้ UV.x เป็น red channel และ UV.y เป็น green channel
3. เพิ่ม uniform `uColor` (vec3) แล้วส่งสีจาก JavaScript

### ระดับ 2: กลาง
4. สร้าง shader ที่วาดวงกลมตรงกลาง plane โดยใช้ distance()
5. สร้าง horizontal gradient จากสีแดงไปเขียว
6. เพิ่ม animation โดยใช้ `uTime` ให้วงกลมเต้นเป็นจังหวะ

### ระดับ 3: ขั้นสูง
7. สร้าง checkerboard pattern ที่ animated (เคลื่อนที่ตามเวลา)
8. สร้าง cosine color palette animation
9. สร้าง sphere ที่มี custom lighting shader

### Challenge
10. สร้าง shader art ที่แสดงภาพ abstract สวยงาม โดยใช้แค่ math functions (ไม่ใช้ textures)
    - ใช้อย่างน้อย: sin, cos, smoothstep, mix
    - ต้องมี animation (ขยับตามเวลา)
    - ต้องมีสีหลายสี

---

## References

- [The Book of Shaders](https://thebookofshaders.com/) - เรียน GLSL เชิงศิลปะ
- [Shadertoy](https://www.shadertoy.com/) - ตัวอย่าง shader art มากมาย
- [GLSL Reference](https://www.khronos.org/files/opengles_shading_language.pdf)
- [Three.js ShaderMaterial docs](https://threejs.org/docs/#api/en/materials/ShaderMaterial)
