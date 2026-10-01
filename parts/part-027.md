# Part 27: Uniforms, Varyings & Advanced GLSL

## Uniforms: ข้อมูลที่ส่งจาก CPU ไป GPU

Uniform คือตัวแปรที่ค่าเหมือนกันสำหรับทุก vertex/fragment ใน draw call เดียวกัน เหมาะสำหรับ:
- เวลา (uTime)
- ขนาดหน้าจอ (uResolution)
- สี (uColor)
- Transformation matrices
- Textures

### Uniform Types ทั้งหมด

```javascript
// Float types
uniforms.uFloat  = { value: 1.0 };
uniforms.uVec2   = { value: new THREE.Vector2(1, 2) };
uniforms.uVec3   = { value: new THREE.Vector3(1, 2, 3) };
uniforms.uVec4   = { value: new THREE.Vector4(1, 2, 3, 4) };

// Integer types  
uniforms.uInt    = { value: 5 };           // int
uniforms.uIVec2  = { value: new THREE.Vector2(1, 2) }; // ivec2

// Boolean
uniforms.uBool   = { value: true };        // bool

// Color (vec3)
uniforms.uColor  = { value: new THREE.Color(0xff0000) }; // vec3

// Matrix
uniforms.uMat3   = { value: new THREE.Matrix3() };
uniforms.uMat4   = { value: new THREE.Matrix4() };

// Textures
uniforms.uTexture     = { value: texture };           // sampler2D
uniforms.uCubeTexture = { value: cubeTexture };       // samplerCube

// Arrays
uniforms.uFloatArray  = { value: [1.0, 2.0, 3.0] };
uniforms.uVec3Array   = { value: [
    new THREE.Vector3(1, 0, 0),
    new THREE.Vector3(0, 1, 0),
    new THREE.Vector3(0, 0, 1)
]};
```

### ประกาศ Uniform ใน GLSL

```glsl
// ต้องตรงกับ type ที่ส่งจาก JavaScript
uniform float    uTime;
uniform vec2     uResolution;
uniform vec3     uColor;
uniform vec4     uParams;
uniform int      uMode;
uniform bool     uEnabled;
uniform mat3     uNormalMatrix;
uniform mat4     uCustomMatrix;
uniform sampler2D uTexture;
uniform samplerCube uEnvMap;

// Arrays
uniform float  uWeights[4];   // float array
uniform vec3   uColors[3];    // vec3 array
uniform mat4   uBones[64];    // bone matrices
```

### Uniform Update ใน JavaScript

```javascript
const material = new THREE.ShaderMaterial({
    uniforms: {
        uTime:   { value: 0.0 },
        uColor:  { value: new THREE.Color(0xff0000) },
        uMouse:  { value: new THREE.Vector2(0, 0) }
    },
    vertexShader: `...`,
    fragmentShader: `...`
});

// Animation loop - update uniforms
const clock = new THREE.Clock();

function animate() {
    requestAnimationFrame(animate);
    
    // Update ด้วย .value
    material.uniforms.uTime.value = clock.getElapsedTime();
    
    // Update Vector2 (ไม่ต้อง new ใหม่ทุก frame!)
    material.uniforms.uMouse.value.set(mouseX, mouseY);
    
    // Update Color
    material.uniforms.uColor.value.setHSL(
        (clock.getElapsedTime() * 0.1) % 1,
        0.8,
        0.5
    );
    
    renderer.render(scene, camera);
}

// Mouse tracking
window.addEventListener('mousemove', (e) => {
    material.uniforms.uMouse.value.x = e.clientX / window.innerWidth;
    material.uniforms.uMouse.value.y = 1.0 - e.clientY / window.innerHeight;
});
```

---

## Precision Qualifiers ในเชิงลึก

```glsl
// Global precision (fragment shader)
precision highp   float;   // IEEE 754 single precision (~7 digits)
precision mediump float;   // ~5 decimal digits
precision lowp    float;   // ~3 decimal digits, range [-2, 2]

precision highp   int;     // [-2^31, 2^31-1]
precision mediump int;     // [-2^15, 2^15-1]
precision lowp    int;     // [-2^8, 2^8-1]

// ต่อตัวแปร
highp   vec3 worldPosition;
mediump vec2 texcoord;
lowp    vec3 vertexColor;

// Precision สำคัญสำหรับ:
// - worldPosition, depth: ใช้ highp
// - color operations: mediump พอ
// - simple colors/alpha: lowp พอ
// - texture coordinates: mediump

// Fragment shader ไม่มี default precision สำหรับ float!
// ต้องประกาศเสมอ
```

---

## Struct ใน GLSL

```glsl
// ประกาศ struct
struct Light {
    vec3 position;
    vec3 color;
    float intensity;
    float radius;
};

struct Material {
    vec3 diffuse;
    vec3 specular;
    float roughness;
    float metallic;
};

// สร้าง instance
Light myLight = Light(
    vec3(5.0, 5.0, 5.0),   // position
    vec3(1.0, 0.9, 0.8),   // color
    2.0,                     // intensity
    10.0                     // radius
);

Material myMat;
myMat.diffuse = vec3(0.8, 0.3, 0.1);
myMat.specular = vec3(0.5);
myMat.roughness = 0.3;
myMat.metallic = 0.0;

// Struct เป็น uniform
uniform Light uLight;      // ส่งจาก JavaScript ผ่าน uniform
uniform Material uMat;

// ใช้ struct ใน function
vec3 computeLight(Light light, Material mat, vec3 normal, vec3 viewDir, vec3 fragPos) {
    vec3 lightDir = normalize(light.position - fragPos);
    float dist = length(light.position - fragPos);
    float attenuation = light.intensity / (1.0 + dist * dist / (light.radius * light.radius));
    
    float diffuse = max(0.0, dot(normal, lightDir));
    vec3 halfVec = normalize(lightDir + viewDir);
    float specular = pow(max(0.0, dot(normal, halfVec)), 1.0 / max(mat.roughness * mat.roughness, 0.001));
    
    return (mat.diffuse * diffuse + mat.specular * specular) * light.color * attenuation;
}
```

### ส่ง Struct จาก JavaScript

```javascript
// ส่ง struct เป็น individual uniforms
uniforms: {
    'uLight.position':  { value: new THREE.Vector3(5, 5, 5) },
    'uLight.color':     { value: new THREE.Color(1, 0.9, 0.8) },
    'uLight.intensity': { value: 2.0 },
    'uLight.radius':    { value: 10.0 },
    
    'uMat.diffuse':   { value: new THREE.Color(0.8, 0.3, 0.1) },
    'uMat.roughness': { value: 0.3 }
}
```

---

## Arrays ใน GLSL

```glsl
// ประกาศ array
float weights[4];           // ยังไม่ initialize
float values[3] = float[3](0.1, 0.5, 0.9);  // initialize พร้อมกัน

// Uniform arrays
uniform float uKernel[9];   // 3x3 convolution kernel
uniform vec3  uPoints[10];  // array of points
uniform mat4  uBones[100];  // skeletal animation bones

// เข้าถึง
float w = weights[0];       // ต้องใช้ constant index (ใน GLSL 1.0)
// ใน GLSL 3.0 ใช้ variable index ได้
// weights[i] // ← ใน for loop ได้

// ใน for loop
for (int i = 0; i < 9; i++) {
    sum += uKernel[i] * sample[i];
}

// ส่ง array จาก JavaScript
uniforms: {
    uKernel: {
        value: [
            1/16, 2/16, 1/16,
            2/16, 4/16, 2/16,
            1/16, 2/16, 1/16
        ]
    }
}
```

---

## Functions ใน GLSL

```glsl
// ประกาศ function ก่อน main()
float myFunction(float x, float y) {
    return x * x + y * y;
}

// Function parameters ด้วย qualifiers
// in: read-only (default)
// out: write-only, ส่งค่ากลับผ่าน parameter
// inout: อ่านและเขียนได้

void swap(inout float a, inout float b) {
    float temp = a;
    a = b;
    b = temp;
}

float safeDiv(in float a, in float b, out bool didDivide) {
    if (abs(b) < 0.0001) {
        didDivide = false;
        return 0.0;
    }
    didDivide = true;
    return a / b;
}

// Function overloading (GLSL รองรับ!)
float add(float a, float b) { return a + b; }
vec2  add(vec2 a, vec2 b)   { return a + b; }
vec3  add(vec3 a, vec3 b)   { return a + b; }

// Recursive functions ไม่รองรับใน GLSL!
// ต้องเขียนแบบ iterative
```

---

## If/For/While และข้อจำกัด

### If/Else
```glsl
// ใช้ if/else ได้ แต่ระวัง performance
float result;
if (x > 0.5) {
    result = 1.0;
} else {
    result = 0.0;
}

// ดีกว่า: ใช้ step() แทน
float result2 = step(0.5, x);

// Ternary operator
float result3 = x > 0.5 ? 1.0 : 0.0;

// หลีกเลี่ยง: if ใน tight loops
// ทุก fragment/vertex รัน if แม้ว่า branch จะ predictable
```

### For Loop
```glsl
// For loop - GLSL 1.0: จำนวนรอบต้องรู้ล่วงหน้า
for (int i = 0; i < 10; i++) {
    sum += values[i];
}

// ใช้ const หรือ literal (GLSL 1.0)
const int MAX = 5;
for (int i = 0; i < MAX; i++) { ... }

// ใช้ break/continue ได้
for (int i = 0; i < 10; i++) {
    if (values[i] < 0.0) continue;
    if (values[i] > 1.0) break;
    sum += values[i];
}

// GLSL 3.0: variable loop count ได้
uniform int uCount;
for (int i = 0; i < uCount; i++) { // ← ใน GLSL 3.0
    ...
}
```

### While Loop
```glsl
// While loop (ระวังใช้! อาจทำให้ shader ช้ามาก)
int i = 0;
while (i < 10) {
    sum += values[i];
    i++;
}

// ไม่แนะนำใน fragment shader ที่ไม่รู้จำนวนรอบ
// GPU ไม่ abort loop ถ้า shader ค้าง
```

---

## Discard Keyword

```glsl
// discard: ยกเลิก fragment ทั้งหมด ไม่เขียนลง framebuffer
void main() {
    vec4 texColor = texture2D(uTexture, vUv);
    
    // Alpha test
    if (texColor.a < 0.5) {
        discard; // ทิ้ง pixel โปร่งใส
    }
    
    gl_FragColor = texColor;
}

// ใช้ discard สำหรับ cutout effects
void main() {
    vec2 uv = vUv;
    float mask = texture2D(uMask, uv).r;
    
    if (mask < 0.1) discard; // เจาะรู
    
    gl_FragColor = vec4(color, 1.0);
}

// Circular clip
void main() {
    vec2 centered = vUv * 2.0 - 1.0;
    if (length(centered) > 1.0) discard; // เหลือเฉพาะวงกลม
    
    gl_FragColor = vec4(1.0, 0.5, 0.0, 1.0);
}

// หมายเหตุ: discard ปิดการ optimize เช่น early-z test
// ใช้เฉพาะเมื่อจำเป็น
```

---

## Demo: Interactive Uniform Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Interactive Uniform Demo</title>
    <style>
        * { margin: 0; padding: 0; }
        canvas { display: block; }
        #panel {
            position: fixed;
            top: 10px;
            left: 10px;
            color: white;
            font-family: monospace;
            font-size: 13px;
            background: rgba(0,0,0,0.85);
            padding: 15px;
            border-radius: 8px;
            width: 280px;
        }
        .row { display: flex; align-items: center; margin: 6px 0; }
        .row label { width: 120px; flex-shrink: 0; }
        .row input[type="range"] { flex: 1; }
        .row input[type="color"] { width: 50px; height: 24px; }
        .row .val { width: 45px; text-align: right; color: #aaa; }
        select { width: 100%; background: #1a1a2e; color: white; border: 1px solid #444; padding: 4px; }
    </style>
</head>
<body>
<div id="panel">
    <b>Uniform Controls</b>
    <div class="row">
        <label>Shape:</label>
        <select id="shape">
            <option value="0">Circle SDF</option>
            <option value="1">Box SDF</option>
            <option value="2">Star SDF</option>
            <option value="3">Ripple</option>
            <option value="4">Spiral</option>
            <option value="5">Voronoi</option>
        </select>
    </div>
    <div class="row">
        <label>Color A:</label>
        <input type="color" id="colorA" value="#3366ff">
    </div>
    <div class="row">
        <label>Color B:</label>
        <input type="color" id="colorB" value="#ff6633">
    </div>
    <div class="row">
        <label>Scale:</label>
        <input type="range" id="scale" min="0.5" max="5" value="1" step="0.1">
        <span class="val" id="scaleVal">1.0</span>
    </div>
    <div class="row">
        <label>Speed:</label>
        <input type="range" id="speed" min="0" max="5" value="1" step="0.1">
        <span class="val" id="speedVal">1.0</span>
    </div>
    <div class="row">
        <label>Distort:</label>
        <input type="range" id="distort" min="0" max="1" value="0" step="0.01">
        <span class="val" id="distortVal">0.00</span>
    </div>
    <div class="row">
        <label>Glow:</label>
        <input type="range" id="glow" min="0" max="5" value="1" step="0.1">
        <span class="val" id="glowVal">1.0</span>
    </div>
    <div class="row">
        <label>Zoom:</label>
        <input type="range" id="zoom" min="0.1" max="3" value="1" step="0.05">
        <span class="val" id="zoomVal">1.00</span>
    </div>
    <div class="row">
        <label>Mouse:</label>
        <span id="mousePos">(0.50, 0.50)</span>
    </div>
</div>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, innerWidth / innerHeight, 0.1, 10);
camera.position.z = 1;
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
document.body.appendChild(renderer.domElement);

const mouse = new THREE.Vector2(0.5, 0.5);
document.addEventListener('mousemove', (e) => {
    mouse.x = e.clientX / innerWidth;
    mouse.y = 1.0 - e.clientY / innerHeight;
    document.getElementById('mousePos').textContent =
        `(${mouse.x.toFixed(2)}, ${mouse.y.toFixed(2)})`;
});

const fragmentShader = `
    precision highp float;
    
    // All configurable uniforms
    uniform float uTime;
    uniform vec2  uMouse;
    uniform vec2  uResolution;
    uniform int   uShape;
    uniform vec3  uColorA;
    uniform vec3  uColorB;
    uniform float uScale;
    uniform float uSpeed;
    uniform float uDistort;
    uniform float uGlow;
    uniform float uZoom;
    
    varying vec2 vUv;
    
    // --- Utility ---
    float hash(vec2 p) {
        return fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5453);
    }
    
    // --- SDFs ---
    float sdCircle(vec2 p, float r) { return length(p) - r; }
    float sdBox(vec2 p, vec2 b) {
        vec2 d = abs(p) - b;
        return length(max(d, 0.0)) + min(max(d.x, d.y), 0.0);
    }
    float sdStar5(vec2 p, float r) {
        const vec2 k1 = vec2(0.809016994375, -0.587785252192);
        const vec2 k2 = vec2(-k1.x, k1.y);
        p.x = abs(p.x);
        p -= 2.0 * max(dot(k1, p), 0.0) * k1;
        p -= 2.0 * max(dot(k2, p), 0.0) * k2;
        p.x = abs(p.x);
        p.y -= r;
        vec2 ba = vec2(-k1.y, k1.x) * r - vec2(0, 1);
        float h = clamp(dot(p, ba) / dot(ba, ba), 0.0, r);
        return length(p - ba * h) * sign(p.y * ba.x - p.x * ba.y);
    }
    
    void main() {
        vec2 uv = vUv;
        
        // Apply zoom
        uv = (uv - 0.5) / uZoom + 0.5;
        
        // Center
        vec2 p = uv - 0.5;
        p.x *= uResolution.x / uResolution.y; // aspect ratio correct
        
        // Scale
        p *= uScale;
        
        float t = uTime * uSpeed;
        
        // Distortion
        if (uDistort > 0.0) {
            p.x += sin(p.y * 8.0 + t) * uDistort * 0.1;
            p.y += cos(p.x * 8.0 + t * 0.7) * uDistort * 0.1;
        }
        
        // Mouse follow
        vec2 mouseOff = (uMouse - 0.5) * 0.5;
        
        float sdf = 0.0;
        float pattern = 0.0;
        
        if (uShape == 0) {
            // Animated Circle
            float r = 0.3 + sin(t) * 0.1;
            sdf = sdCircle(p + mouseOff, r);
            
        } else if (uShape == 1) {
            // Rotating Box
            float angle = t;
            vec2 rp = vec2(p.x * cos(angle) - p.y * sin(angle),
                           p.x * sin(angle) + p.y * cos(angle));
            sdf = sdBox(rp + mouseOff, vec2(0.25, 0.2));
            
        } else if (uShape == 2) {
            // Spinning Star
            float angle = t * 0.5;
            vec2 rp = vec2(p.x * cos(angle) - p.y * sin(angle),
                           p.x * sin(angle) + p.y * cos(angle));
            sdf = sdStar5(rp + mouseOff, 0.3);
            
        } else if (uShape == 3) {
            // Ripple
            float dist = length(p - mouseOff);
            pattern = sin(dist * 20.0 - t * 3.0) * 0.5 + 0.5;
            sdf = dist - 0.3;
            
        } else if (uShape == 4) {
            // Spiral
            float r = length(p);
            float a = atan(p.y, p.x);
            float spiral = sin(r * 10.0 - a * 3.0 - t * 2.0) * 0.5 + 0.5;
            pattern = spiral;
            sdf = r - 0.5;
            
        } else {
            // Voronoi
            vec2 i = floor(p * 3.0);
            vec2 f = fract(p * 3.0);
            float minDist = 1e10;
            for (int y = -1; y <= 1; y++) {
                for (int x = -1; x <= 1; x++) {
                    vec2 n = vec2(float(x), float(y));
                    vec2 cell = n + vec2(hash(i + n + t * 0.1), hash(i + n + vec2(1.0) + t * 0.1));
                    float d = length(cell - f);
                    minDist = min(minDist, d);
                }
            }
            pattern = minDist;
            sdf = minDist - 0.3;
        }
        
        // Render
        float fill  = smoothstep(0.005, -0.005, sdf);
        float glow_ = exp(-abs(sdf) * (5.0 / uGlow));
        
        float mixFactor = fill;
        if (uShape >= 3) {
            mixFactor = pattern; // pattern-based coloring
        } else {
            mixFactor = fill;
        }
        
        vec3 color = mix(vec3(0.02, 0.02, 0.05), uColorA, fill);
        color += uColorB * glow_ * uGlow * 0.3;
        
        // Time-based hue shift
        float hueShift = t * 0.05;
        vec3 shifted = mix(uColorA, uColorB, sin(hueShift) * 0.5 + 0.5);
        
        if (uShape >= 3) {
            color = mix(uColorA, uColorB, pattern);
        }
        
        gl_FragColor = vec4(color, 1.0);
    }
`;

const geometry = new THREE.PlaneGeometry(2, 2);
const uniforms = {
    uTime:       { value: 0 },
    uMouse:      { value: mouse },
    uResolution: { value: new THREE.Vector2(innerWidth, innerHeight) },
    uShape:      { value: 0 },
    uColorA:     { value: new THREE.Color(0x3366ff) },
    uColorB:     { value: new THREE.Color(0xff6633) },
    uScale:      { value: 1.0 },
    uSpeed:      { value: 1.0 },
    uDistort:    { value: 0.0 },
    uGlow:       { value: 1.0 },
    uZoom:       { value: 1.0 }
};

const material = new THREE.ShaderMaterial({
    vertexShader: `
        varying vec2 vUv;
        void main() { vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }
    `,
    fragmentShader,
    uniforms
});

scene.add(new THREE.Mesh(geometry, material));

// UI
document.getElementById('shape').onchange = (e) => uniforms.uShape.value = parseInt(e.target.value);
document.getElementById('colorA').oninput = (e) => uniforms.uColorA.value.set(e.target.value);
document.getElementById('colorB').oninput = (e) => uniforms.uColorB.value.set(e.target.value);

function makeSlider(id, uniform, digits = 1) {
    document.getElementById(id).oninput = (e) => {
        const v = parseFloat(e.target.value);
        uniforms[uniform].value = v;
        document.getElementById(id + 'Val').textContent = v.toFixed(digits);
    };
}

makeSlider('scale', 'uScale', 1);
makeSlider('speed', 'uSpeed', 1);
makeSlider('distort', 'uDistort', 2);
makeSlider('glow', 'uGlow', 1);
makeSlider('zoom', 'uZoom', 2);

const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    uniforms.uTime.value = clock.getElapsedTime();
    renderer.render(scene, camera);
}
animate();

window.addEventListener('resize', () => {
    camera.aspect = innerWidth / innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(innerWidth, innerHeight);
    uniforms.uResolution.value.set(innerWidth, innerHeight);
});
</script>
</body>
</html>
```

---

## Demo: Complex GLSL Functions

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Complex GLSL Functions</title>
    <style>
        * { margin: 0; padding: 0; }
        canvas { display: block; }
    </style>
</head>
<body>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';
import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.157.0/examples/jsm/controls/OrbitControls.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x020210);
const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 100);
camera.position.set(0, 0, 3);
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
document.body.appendChild(renderer.domElement);
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

const fragmentShader = `
    precision highp float;
    
    uniform float uTime;
    uniform vec2  uResolution;
    
    varying vec2 vUv;
    
    // === ฟังก์ชัน Library ===
    
    // --- Color Manipulation ---
    vec3 hsv2rgb(vec3 c) {
        vec4 K = vec4(1.0, 2.0/3.0, 1.0/3.0, 3.0);
        vec3 p = abs(fract(c.xxx + K.xyz) * 6.0 - K.www);
        return c.z * mix(K.xxx, clamp(p - K.xxx, 0.0, 1.0), c.y);
    }
    
    // Cosine palette - สวยงามมาก
    vec3 pal(float t, vec3 a, vec3 b, vec3 c, vec3 d) {
        return a + b * cos(6.28318 * (c * t + d));
    }
    
    // Spectral color (rainbow)
    vec3 spectral(float t) {
        return pal(t,
            vec3(0.5, 0.5, 0.5),
            vec3(0.5, 0.5, 0.5),
            vec3(1.0, 1.0, 1.0),
            vec3(0.0, 0.33, 0.67)
        );
    }
    
    // --- Math Utilities ---
    
    // Smooth minimum (blending)
    float smin(float a, float b, float k) {
        float h = max(k - abs(a - b), 0.0) / k;
        return min(a, b) - h * h * k * (1.0/4.0);
    }
    
    // Smooth maximum
    float smax(float a, float b, float k) {
        return -smin(-a, -b, k);
    }
    
    // Remap value จาก range [a, b] ไป [c, d]
    float remap(float value, float a, float b, float c, float d) {
        return c + (d - c) * (value - a) / (b - a);
    }
    
    // Easing functions
    float easeInQuad(float t) { return t * t; }
    float easeOutQuad(float t) { return t * (2.0 - t); }
    float easeInOutQuad(float t) {
        return t < 0.5 ? 2.0 * t * t : -1.0 + (4.0 - 2.0 * t) * t;
    }
    float easeElastic(float t) {
        return sin(-13.0 * (t + 1.0) * 1.5707963) * pow(2.0, -10.0 * t) + 1.0;
    }
    
    // --- 2D Rotation ---
    vec2 rotate2D(vec2 p, float angle) {
        float s = sin(angle);
        float c = cos(angle);
        return vec2(p.x * c - p.y * s, p.x * s + p.y * c);
    }
    
    // --- Tiling ---
    vec2 tile(vec2 p, float size) {
        return fract(p / size) * size;
    }
    
    vec2 tileMirror(vec2 p, float size) {
        p /= size;
        p = abs(fract(p * 0.5) * 2.0 - 1.0);
        return p * size;
    }
    
    // --- Noise Functions ---
    float hash21(vec2 p) {
        vec3 p3 = fract(vec3(p.xyx) * 0.1031);
        p3 += dot(p3, p3.yzx + 33.33);
        return fract((p3.x + p3.y) * p3.z);
    }
    
    float valueNoise(vec2 p) {
        vec2 i = floor(p);
        vec2 f = fract(p);
        vec2 u = f * f * (3.0 - 2.0 * f);
        return mix(
            mix(hash21(i), hash21(i + vec2(1,0)), u.x),
            mix(hash21(i + vec2(0,1)), hash21(i + vec2(1,1)), u.x),
            u.y
        );
    }
    
    // FBM (Fractional Brownian Motion)
    float fbm(vec2 p) {
        float value = 0.0;
        float amplitude = 0.5;
        float frequency = 1.0;
        mat2 rot = mat2(0.8, -0.6, 0.6, 0.8); // rotation
        
        for (int i = 0; i < 6; i++) {
            value += amplitude * valueNoise(p * frequency);
            amplitude *= 0.5;
            frequency *= 2.0;
            p = rot * p; // เพิ่ม rotation ทุก octave
        }
        return value;
    }
    
    // Domain Warping - fbm of fbm!
    float warpedFbm(vec2 p) {
        // First warp
        vec2 q = vec2(
            fbm(p + vec2(0.0, 0.0)),
            fbm(p + vec2(5.2, 1.3))
        );
        
        // Second warp
        vec2 r = vec2(
            fbm(p + 4.0 * q + vec2(1.7, 9.2)),
            fbm(p + 4.0 * q + vec2(8.3, 2.8))
        );
        
        return fbm(p + 4.0 * r);
    }
    
    void main() {
        vec2 uv = vUv;
        vec2 aspect = vec2(uResolution.x / uResolution.y, 1.0);
        vec2 p = (uv - 0.5) * aspect;
        
        float t = uTime * 0.2;
        
        // Animated warping
        vec2 warpedP = p + vec2(t * 0.1, t * 0.07);
        float n = warpedFbm(warpedP * 2.0);
        
        // Map noise to color palette
        vec3 color1 = spectral(n + t * 0.1);
        
        // เพิ่ม detail layer
        float detail = fbm(p * 8.0 + t * 0.3) * 0.5;
        
        // เพิ่ม circular glow
        float dist = length(p);
        float glow = exp(-dist * dist * 5.0);
        
        // รวมทุกอย่าง
        vec3 finalColor = color1;
        finalColor = mix(finalColor, vec3(1.0), detail * 0.15);
        finalColor += spectral(n + 0.5) * glow * 0.5;
        
        // Edge vignette
        float vignette = 1.0 - smoothstep(0.3, 0.8, length(p));
        finalColor *= vignette;
        
        // Tone mapping
        finalColor = finalColor / (finalColor + 0.5);
        
        gl_FragColor = vec4(finalColor, 1.0);
    }
`;

const geometry = new THREE.PlaneGeometry(2, 2);
const uniforms = {
    uTime:       { value: 0 },
    uResolution: { value: new THREE.Vector2(innerWidth, innerHeight) }
};

const material = new THREE.ShaderMaterial({
    vertexShader: `
        varying vec2 vUv;
        void main() { vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }
    `,
    fragmentShader,
    uniforms
});

scene.add(new THREE.Mesh(geometry, material));

const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    uniforms.uTime.value = clock.getElapsedTime();
    renderer.render(scene, camera);
}
animate();

window.addEventListener('resize', () => {
    camera.aspect = innerWidth / innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(innerWidth, innerHeight);
    uniforms.uResolution.value.set(innerWidth, innerHeight);
});
</script>
</body>
</html>
```

---

## Early Return Pattern ใน GLSL

```glsl
// GLSL ไม่มี early return ใน main()
// แต่ทำได้ใน functions!

vec3 computeColor(vec2 uv) {
    // Early return ใน function ได้
    if (uv.x < 0.0 || uv.x > 1.0) return vec3(0.0); // out of bounds
    if (uv.y < 0.0 || uv.y > 1.0) return vec3(0.0); // out of bounds
    
    // Normal computation
    return vec3(uv, 0.5);
}

void main() {
    // ใน main ต้องใช้ if/else หรือ discard แทน early return
    vec3 color = computeColor(vUv);
    
    if (shouldDiscard) {
        discard; // ← ทำงานเหมือน early return แต่ทิ้ง fragment
        // code หลัง discard ไม่รัน
    }
    
    gl_FragColor = vec4(color, 1.0);
    // code หลัง gl_FragColor = ... ยังคงรัน!
}
```

---

## Varying Precision & Interpolation

```glsl
// Vertex Shader
varying highp   vec3 vWorldPosition;  // สำคัญ: ต้องการความแม่นยำสูง
varying mediump vec2 vUv;             // พอแล้วสำหรับ UV
varying lowp    vec3 vColor;          // เพียงพอสำหรับ color

// Fragment Shader
// ต้องประกาศ precision เดียวกับใน vertex shader!
varying highp   vec3 vWorldPosition;
varying mediump vec2 vUv;
varying lowp    vec3 vColor;
```

---

## Uniform Buffer Objects (UBO) - สำหรับ Three.js ขั้นสูง

```javascript
// Three.js ยังไม่รองรับ UBO โดยตรง
// แต่เราใช้ WebGL2 ได้ถ้าต้องการ

// วิธีใช้ UBO ผ่าน WebGL2
const renderer = new THREE.WebGLRenderer({ antialias: true });
const gl = renderer.getContext(); // WebGL2 context

// สร้าง UBO
const uboData = new Float32Array([
    // Common params
    0.0,  // time
    1280, // resolution.x
    720,  // resolution.y
    0.0   // padding
]);

const ubo = gl.createBuffer();
gl.bindBuffer(gl.UNIFORM_BUFFER, ubo);
gl.bufferData(gl.UNIFORM_BUFFER, uboData, gl.DYNAMIC_DRAW);

// Update ทุก frame
function updateUBO(time) {
    uboData[0] = time;
    gl.bindBuffer(gl.UNIFORM_BUFFER, ubo);
    gl.bufferSubData(gl.UNIFORM_BUFFER, 0, uboData);
}
```

---

## สรุป Best Practices

```
1. ตั้งชื่อ uniforms: u prefix (uTime, uColor, uResolution)
2. ตั้งชื่อ varyings: v prefix (vUv, vNormal, vWorldPos)
3. ตั้งชื่อ attributes: a prefix (aPosition, aOffset)
4. ใช้ precision ให้เหมาะสม: position=highp, color=mediump, flags=lowp
5. อัพเดท uniform แค่เมื่อมีการเปลี่ยนแปลง (ไม่ต้อง set ทุก frame ถ้าไม่เปลี่ยน)
6. ใช้ .copy() / .set() แทนสร้าง object ใหม่ทุก frame
7. Group uniforms ที่เปลี่ยนพร้อมกันใน vec4 เพื่อลด draw calls
8. ใช้ struct สำหรับ related data
9. Avoid dynamic branching (if ใน fragment shader)
10. ใช้ step/smoothstep/clamp แทน if เมื่อทำได้
```

---

## แบบฝึกหัด

### ระดับ 1
1. สร้าง uniform `uColor` (vec3) แล้ว animate ผ่าน HSL color space
2. สร้าง struct `Light` และส่งค่าจาก JavaScript
3. ใช้ `discard` สร้าง circular clip สำหรับ PlaneGeometry

### ระดับ 2
4. Implement `fbm()` function ด้วย 6 octaves
5. สร้าง `easeInOut()` function ใน GLSL
6. ใช้ `inout` parameter สร้าง function ที่แก้ค่าใน place

### ระดับ 3
7. สร้าง multi-light system โดยใช้ struct array (8 lights)
8. Implement domain warping บน sphere
9. สร้าง real-time color grading system

### Challenge
10. สร้าง procedural animated background ที่:
    - ใช้ domain-warped fBm เป็น base
    - Cosine palette animation ตาม noise value
    - Animated rotation per octave
    - Mouse interaction warps the pattern
    - Bloom effect บน bright areas
