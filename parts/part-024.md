# Part 24: Vertex Shaders Deep Dive

## บทนำ: Vertex Shader คืออะไรและทำอะไรได้บ้าง

Vertex shader คือโปรแกรมที่รันบน GPU สำหรับแต่ละ vertex ใน geometry ของเรา มันมีหน้าที่หลักในการ **transform ตำแหน่ง** ของ vertex จาก object space ไปสู่ clip space แต่นอกจากนั้น เราสามารถใช้มันเพื่อ:

- **Vertex Displacement** - เคลื่อนย้าย vertices เพื่อสร้าง shapes ใหม่
- **Wave Animations** - สร้างคลื่นบนพื้นผิว
- **Morphing** - แปลงรูปร่างจาก shape หนึ่งไปอีก shape หนึ่ง
- **Particle Systems** - จัดการ particles บน GPU
- **gl_PointSize** - กำหนดขนาดของ point sprites

---

## Vertex Shader Attributes

Attributes คือข้อมูลที่มาจาก BufferGeometry ต่อ vertex

### ใน ShaderMaterial (built-in attributes)

```glsl
// Three.js ส่ง attributes เหล่านี้ให้อัตโนมัติ
attribute vec3 position;     // ตำแหน่ง vertex ใน object space
attribute vec3 normal;       // Normal vector (ตั้งฉากกับพื้นผิว)
attribute vec2 uv;           // UV coordinates (texture mapping)
attribute vec2 uv2;          // Second UV set
attribute vec3 color;        // Vertex color (ถ้า geometry.setAttribute('color', ...))
attribute vec4 tangent;      // Tangent vector (สำหรับ normal mapping)
```

### Custom Attributes

```javascript
// เพิ่ม custom attribute ใน JavaScript
const geometry = new THREE.BufferGeometry();
const count = 1000;

// สร้าง custom attribute
const aRandom = new Float32Array(count);
for (let i = 0; i < count; i++) {
    aRandom[i] = Math.random();
}
geometry.setAttribute('aRandom', new THREE.BufferAttribute(aRandom, 1));

// ใน vertex shader ใช้ได้เลย
```

```glsl
// ใน vertex shader
attribute float aRandom;    // custom attribute (scalar)
attribute vec3 aOffset;     // custom attribute (vector3)
attribute vec4 aTransform;  // custom attribute (vector4)
```

---

## Uniforms ใน Vertex Shader

Uniforms ใช้ร่วมกันทั้ง vertex และ fragment shader

```javascript
const material = new THREE.ShaderMaterial({
    vertexShader: `...`,
    fragmentShader: `...`,
    uniforms: {
        // Scalar
        uTime:    { value: 0.0 },
        uScale:   { value: 1.0 },
        
        // Vectors
        uOffset:      { value: new THREE.Vector3(0, 0, 0) },
        uWaveParams:  { value: new THREE.Vector4(1.0, 2.0, 0.5, 1.0) },
        
        // Matrices
        uCustomMatrix: { value: new THREE.Matrix4() },
        
        // Textures (ใน vertex shader ต้องการ vertex texture fetch support)
        uDisplacementMap: { value: texture }
    }
});
```

---

## Varyings: ส่งข้อมูลจาก Vertex ไป Fragment

Varyings เป็นกลไกในการส่งข้อมูลจาก vertex shader ไปยัง fragment shader

```
Vertex A ────────── varying vUv = (0.0, 0.0)
         \
          \  GPU Interpolates!
           \
Vertex B    ── Fragment at (0.5, 0.5) gets vUv = (0.5, 0.5)
           /
          /
Vertex C ────────── varying vUv = (1.0, 1.0)
```

```glsl
// Vertex Shader
varying vec2 vUv;
varying vec3 vNormal;
varying vec3 vWorldPosition;
varying float vElevation;   // custom varying

void main() {
    vUv = uv;
    vNormal = normalize(normalMatrix * normal);
    
    // คำนวณ world position
    vec4 worldPos = modelMatrix * vec4(position, 1.0);
    vWorldPosition = worldPos.xyz;
    
    // Wave displacement
    float elevation = sin(position.x * 5.0 + uTime) * 0.1;
    vElevation = elevation;
    
    // Displace position
    vec3 newPos = position;
    newPos.y += elevation;
    
    gl_Position = projectionMatrix * modelViewMatrix * vec4(newPos, 1.0);
}
```

```glsl
// Fragment Shader
varying vec2 vUv;
varying vec3 vNormal;
varying vec3 vWorldPosition;
varying float vElevation;   // รับค่าที่ interpolate แล้ว

void main() {
    // ใช้ vElevation เพื่อสร้างสีตาม height
    vec3 color = mix(
        vec3(0.0, 0.2, 0.8),  // สีน้ำ
        vec3(0.9, 0.8, 0.5),  // สีทราย
        vElevation * 5.0 + 0.5
    );
    
    gl_FragColor = vec4(color, 1.0);
}
```

---

## Demo 1: Wave Vertex Shader

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Wave Vertex Shader</title>
    <style>
        * { margin: 0; padding: 0; }
        canvas { display: block; }
        #controls {
            position: fixed;
            top: 20px;
            left: 20px;
            color: white;
            font-family: monospace;
            background: rgba(0,0,0,0.7);
            padding: 15px;
            border-radius: 8px;
        }
        label { display: block; margin: 5px 0; }
        input[type="range"] { width: 150px; }
    </style>
</head>
<body>
<div id="controls">
    <h3>Wave Controls</h3>
    <label>Frequency: <input type="range" id="freq" min="1" max="20" value="5" step="0.5"></label>
    <label>Amplitude: <input type="range" id="amp" min="0" max="0.5" value="0.1" step="0.01"></label>
    <label>Speed: <input type="range" id="speed" min="0" max="5" value="1" step="0.1"></label>
    <label>Wave Type:
        <select id="waveType">
            <option value="0">Sine</option>
            <option value="1">Cosine</option>
            <option value="2">Circular</option>
            <option value="3">Diagonal</option>
        </select>
    </label>
</div>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';
import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.157.0/examples/jsm/controls/OrbitControls.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x0a0a2e);
const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 100);
camera.position.set(0, 2, 4);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

// Wave Vertex Shader
const vertexShader = `
    uniform float uTime;
    uniform float uFrequency;
    uniform float uAmplitude;
    uniform float uSpeed;
    uniform int   uWaveType;
    
    varying vec2 vUv;
    varying float vElevation;
    varying vec3 vNormal;
    
    void main() {
        vUv = uv;
        
        vec3 pos = position;
        float elevation = 0.0;
        
        if (uWaveType == 0) {
            // Sine wave แนวนอน
            elevation = sin(pos.x * uFrequency + uTime * uSpeed) * uAmplitude;
        } else if (uWaveType == 1) {
            // Cosine wave แนวนอน + แนวตั้ง
            elevation = cos(pos.x * uFrequency + uTime * uSpeed) * uAmplitude
                      + cos(pos.z * uFrequency * 0.7 + uTime * uSpeed * 0.8) * uAmplitude * 0.5;
        } else if (uWaveType == 2) {
            // Circular wave จากจุดศูนย์กลาง
            float dist = length(pos.xz);
            elevation = sin(dist * uFrequency - uTime * uSpeed) * uAmplitude;
        } else {
            // Diagonal wave
            elevation = sin((pos.x + pos.z) * uFrequency + uTime * uSpeed) * uAmplitude;
        }
        
        // เพิ่ม secondary wave เบาๆ
        elevation += sin(pos.x * uFrequency * 2.0 + uTime * uSpeed * 1.5) * uAmplitude * 0.3;
        elevation += sin(pos.z * uFrequency * 1.5 + uTime * uSpeed * 0.7) * uAmplitude * 0.2;
        
        vElevation = elevation;
        
        // คำนวณ normal แบบ approximate (finite difference)
        float delta = 0.01;
        float e1 = sin((pos.x + delta) * uFrequency + uTime * uSpeed) * uAmplitude;
        float e2 = sin((pos.z + delta) * uFrequency * 1.5 + uTime * uSpeed * 0.7) * uAmplitude;
        vec3 tangent1 = normalize(vec3(delta, e1 - elevation, 0.0));
        vec3 tangent2 = normalize(vec3(0.0, e2 - elevation, delta));
        vec3 approxNormal = normalize(cross(tangent2, tangent1));
        
        vNormal = normalize(normalMatrix * approxNormal);
        
        pos.y += elevation;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
    }
`;

const fragmentShader = `
    precision mediump float;
    
    uniform float uTime;
    
    varying vec2 vUv;
    varying float vElevation;
    varying vec3 vNormal;
    
    void main() {
        // สีตาม elevation
        float normalizedElevation = vElevation * 5.0 + 0.5;
        
        // Deep water (ต่ำ) → Shallow water (สูง)
        vec3 deepColor   = vec3(0.02, 0.05, 0.3);
        vec3 shallowColor = vec3(0.0, 0.6, 0.8);
        vec3 foamColor   = vec3(0.9, 0.95, 1.0);
        
        vec3 waterColor = mix(deepColor, shallowColor, smoothstep(0.0, 0.7, normalizedElevation));
        waterColor = mix(waterColor, foamColor, smoothstep(0.8, 1.0, normalizedElevation));
        
        // Simple lighting
        vec3 lightDir = normalize(vec3(1.0, 2.0, 1.0));
        float diffuse = max(0.0, dot(vNormal, lightDir));
        
        // Specular
        vec3 viewDir = vec3(0.0, 1.0, 0.0); // approximate
        vec3 reflectDir = reflect(-lightDir, vNormal);
        float specular = pow(max(0.0, dot(viewDir, reflectDir)), 32.0);
        
        vec3 color = waterColor * (0.3 + diffuse * 0.7) + vec3(specular * 0.5);
        
        // Transparency effect
        float alpha = 0.85 + specular * 0.15;
        
        gl_FragColor = vec4(color, alpha);
    }
`;

// Create water plane (high resolution สำหรับ wave details)
const geometry = new THREE.PlaneGeometry(8, 8, 256, 256);
geometry.rotateX(-Math.PI / 2);

const uniforms = {
    uTime:       { value: 0 },
    uFrequency:  { value: 5.0 },
    uAmplitude:  { value: 0.1 },
    uSpeed:      { value: 1.0 },
    uWaveType:   { value: 0 }
};

const material = new THREE.ShaderMaterial({
    vertexShader,
    fragmentShader,
    uniforms,
    transparent: true,
    side: THREE.DoubleSide
});

const water = new THREE.Mesh(geometry, material);
scene.add(water);

// Grid helper
const grid = new THREE.GridHelper(8, 8, 0x444444, 0x222222);
grid.position.y = 0.01;
// scene.add(grid); // ซ่อน grid

// UI Controls
document.getElementById('freq').addEventListener('input', (e) => {
    uniforms.uFrequency.value = parseFloat(e.target.value);
});
document.getElementById('amp').addEventListener('input', (e) => {
    uniforms.uAmplitude.value = parseFloat(e.target.value);
});
document.getElementById('speed').addEventListener('input', (e) => {
    uniforms.uSpeed.value = parseFloat(e.target.value);
});
document.getElementById('waveType').addEventListener('change', (e) => {
    uniforms.uWaveType.value = parseInt(e.target.value);
});

const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    uniforms.uTime.value = clock.getElapsedTime();
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

---

## Demo 2: Morphing Sphere

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Morphing Sphere Shader</title>
    <style>
        * { margin: 0; padding: 0; }
        canvas { display: block; }
        #ui {
            position: fixed;
            top: 20px;
            left: 20px;
            color: white;
            font-family: monospace;
            background: rgba(0,0,0,0.7);
            padding: 15px;
            border-radius: 8px;
        }
        button {
            display: block;
            margin: 5px 0;
            padding: 8px 15px;
            background: #2a4a8a;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            width: 100%;
        }
        button:hover { background: #3a6aaa; }
        button.active { background: #4a8aaa; }
    </style>
</head>
<body>
<div id="ui">
    <h3>Morph Shape</h3>
    <button id="btn-sphere" class="active">Sphere</button>
    <button id="btn-cube">Cube</button>
    <button id="btn-cylinder">Cylinder</button>
    <button id="btn-torus">Torus</button>
    <hr style="margin:10px 0">
    <label>Noise Strength:
        <input type="range" id="noiseStr" min="0" max="0.5" value="0.1" step="0.01">
    </label>
    <label>Noise Speed:
        <input type="range" id="noiseSpd" min="0" max="3" value="0.5" step="0.1">
    </label>
</div>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';
import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.157.0/examples/jsm/controls/OrbitControls.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x050510);
const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 100);
camera.position.set(0, 0, 3);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

const vertexShader = `
    uniform float uTime;
    uniform float uMorphTarget; // 0=sphere, 1=cube, 2=cylinder, 3=torus
    uniform float uMorphProgress;
    uniform float uNoiseStrength;
    uniform float uNoiseSpeed;
    
    varying vec2 vUv;
    varying vec3 vNormal;
    varying float vNoise;
    
    // Simple 3D noise
    vec3 mod289(vec3 x) { return x - floor(x * (1.0 / 289.0)) * 289.0; }
    vec4 mod289(vec4 x) { return x - floor(x * (1.0 / 289.0)) * 289.0; }
    vec4 permute(vec4 x) { return mod289(((x * 34.0) + 1.0) * x); }
    vec4 taylorInvSqrt(vec4 r) { return 1.79284291400159 - 0.85373472095314 * r; }
    
    float snoise(vec3 v) {
        const vec2 C = vec2(1.0/6.0, 1.0/3.0);
        const vec4 D = vec4(0.0, 0.5, 1.0, 2.0);
        
        vec3 i  = floor(v + dot(v, C.yyy));
        vec3 x0 = v - i + dot(i, C.xxx);
        
        vec3 g = step(x0.yzx, x0.xyz);
        vec3 l = 1.0 - g;
        vec3 i1 = min(g.xyz, l.zxy);
        vec3 i2 = max(g.xyz, l.zxy);
        
        vec3 x1 = x0 - i1 + C.xxx;
        vec3 x2 = x0 - i2 + C.yyy;
        vec3 x3 = x0 - D.yyy;
        
        i = mod289(i);
        vec4 p = permute(permute(permute(
            i.z + vec4(0.0, i1.z, i2.z, 1.0))
            + i.y + vec4(0.0, i1.y, i2.y, 1.0))
            + i.x + vec4(0.0, i1.x, i2.x, 1.0));
        
        float n_ = 0.142857142857;
        vec3 ns = n_ * D.wyz - D.xzx;
        
        vec4 j = p - 49.0 * floor(p * ns.z * ns.z);
        
        vec4 x_ = floor(j * ns.z);
        vec4 y_ = floor(j - 7.0 * x_);
        
        vec4 x = x_ * ns.x + ns.yyyy;
        vec4 y = y_ * ns.x + ns.yyyy;
        vec4 h = 1.0 - abs(x) - abs(y);
        
        vec4 b0 = vec4(x.xy, y.xy);
        vec4 b1 = vec4(x.zw, y.zw);
        
        vec4 s0 = floor(b0) * 2.0 + 1.0;
        vec4 s1 = floor(b1) * 2.0 + 1.0;
        vec4 sh = -step(h, vec4(0.0));
        
        vec4 a0 = b0.xzyw + s0.xzyw * sh.xxyy;
        vec4 a1 = b1.xzyw + s1.xzyw * sh.zzww;
        
        vec3 p0 = vec3(a0.xy, h.x);
        vec3 p1 = vec3(a0.zw, h.y);
        vec3 p2 = vec3(a1.xy, h.z);
        vec3 p3 = vec3(a1.zw, h.w);
        
        vec4 norm = taylorInvSqrt(vec4(dot(p0,p0), dot(p1,p1), dot(p2,p2), dot(p3,p3)));
        p0 *= norm.x; p1 *= norm.y; p2 *= norm.z; p3 *= norm.w;
        
        vec4 m = max(0.6 - vec4(dot(x0,x0), dot(x1,x1), dot(x2,x2), dot(x3,x3)), 0.0);
        m = m * m;
        return 42.0 * dot(m*m, vec4(dot(p0,x0), dot(p1,x1), dot(p2,x2), dot(p3,x3)));
    }
    
    // Sphere position (r=1)
    vec3 spherePos(vec3 pos) {
        return normalize(pos);
    }
    
    // Cube position (box deformation)
    vec3 cubePos(vec3 pos) {
        // นำ sphere position มา "box" ด้วย max norm
        vec3 n = normalize(pos);
        float maxComp = max(abs(n.x), max(abs(n.y), abs(n.z)));
        return n / maxComp;
    }
    
    // Cylinder position
    vec3 cylinderPos(vec3 pos) {
        vec3 n = normalize(pos);
        float r = length(n.xz);
        float cylinderRadius = 0.7;
        return vec3(n.x / r * cylinderRadius, n.y * 1.5, n.z / r * cylinderRadius);
    }
    
    // Torus position (approximate)
    vec3 torusPos(vec3 pos) {
        vec3 n = normalize(pos);
        float bigR = 0.7;
        float smallR = 0.3;
        float theta = atan(n.z, n.x);
        float phi = atan(n.y, length(n.xz) - bigR);
        return vec3(
            (bigR + smallR * cos(phi)) * cos(theta),
            smallR * sin(phi),
            (bigR + smallR * cos(phi)) * sin(theta)
        );
    }
    
    void main() {
        vUv = uv;
        
        vec3 pos = position;
        
        // คำนวณ target positions
        vec3 sphereP   = spherePos(pos);
        vec3 cubeP     = cubePos(pos);
        vec3 cylinderP = cylinderPos(pos);
        vec3 torusP    = torusPos(pos);
        
        // เลือก morph targets
        vec3 fromPos, toPos;
        if (uMorphTarget < 0.5) {
            fromPos = sphereP;
            toPos = cubeP;
        } else if (uMorphTarget < 1.5) {
            fromPos = cubeP;
            toPos = cylinderP;
        } else if (uMorphTarget < 2.5) {
            fromPos = cylinderP;
            toPos = torusP;
        } else {
            fromPos = torusP;
            toPos = sphereP;
        }
        
        // Smooth interpolation
        float t = smoothstep(0.0, 1.0, uMorphProgress);
        vec3 morphedPos = mix(fromPos, toPos, t);
        
        // Organic noise displacement
        float noiseTime = uTime * uNoiseSpeed;
        float noise = snoise(morphedPos * 2.0 + noiseTime);
        vNoise = noise;
        morphedPos += normalize(morphedPos) * noise * uNoiseStrength;
        
        // Normal approximation
        vNormal = normalize(normalMatrix * normalize(morphedPos));
        
        gl_Position = projectionMatrix * modelViewMatrix * vec4(morphedPos, 1.0);
    }
`;

const fragmentShader = `
    precision mediump float;
    
    uniform float uTime;
    
    varying vec2 vUv;
    varying vec3 vNormal;
    varying float vNoise;
    
    void main() {
        // Fresnel-like edge glow
        vec3 viewDir = vec3(0.0, 0.0, 1.0);
        float fresnel = 1.0 - abs(dot(vNormal, viewDir));
        fresnel = pow(fresnel, 2.0);
        
        // Color based on noise + time
        float t = uTime * 0.2;
        vec3 col1 = vec3(0.5 + sin(t) * 0.5, 0.2, 0.8);
        vec3 col2 = vec3(0.1, 0.5 + cos(t * 1.3) * 0.5, 0.7);
        
        float noiseNorm = vNoise * 0.5 + 0.5;
        vec3 baseColor = mix(col1, col2, noiseNorm);
        
        // Edge glow
        vec3 glowColor = vec3(0.3, 0.7, 1.0);
        vec3 color = mix(baseColor, glowColor, fresnel * 0.5);
        
        // Lighting
        vec3 lightDir = normalize(vec3(1.0, 2.0, 3.0));
        float diffuse = max(0.0, dot(vNormal, lightDir));
        color *= 0.3 + diffuse * 0.7;
        color += glowColor * fresnel * 0.3;
        
        gl_FragColor = vec4(color, 1.0);
    }
`;

const geometry = new THREE.SphereGeometry(1, 128, 128);
const uniforms = {
    uTime:          { value: 0 },
    uMorphTarget:   { value: 0 },
    uMorphProgress: { value: 0 },
    uNoiseStrength: { value: 0.1 },
    uNoiseSpeed:    { value: 0.5 }
};

const material = new THREE.ShaderMaterial({
    vertexShader,
    fragmentShader,
    uniforms,
    side: THREE.DoubleSide
});

scene.add(new THREE.Mesh(geometry, material));

// Morphing state
let currentShape = 0;
let targetShape = 0;
let isMorphing = false;
const morphDuration = 1.0; // seconds
let morphStartTime = 0;

function startMorph(shapeIndex) {
    if (isMorphing) return;
    if (shapeIndex === currentShape) return;
    
    targetShape = shapeIndex;
    isMorphing = true;
    morphStartTime = clock.getElapsedTime();
    
    // Update morph target direction
    uniforms.uMorphTarget.value = currentShape;
    uniforms.uMorphProgress.value = 0;
}

// Button controls
const buttons = document.querySelectorAll('button');
document.getElementById('btn-sphere').onclick = () => { startMorph(0); setActive('btn-sphere'); };
document.getElementById('btn-cube').onclick = () => { startMorph(1); setActive('btn-cube'); };
document.getElementById('btn-cylinder').onclick = () => { startMorph(2); setActive('btn-cylinder'); };
document.getElementById('btn-torus').onclick = () => { startMorph(3); setActive('btn-torus'); };

function setActive(id) {
    buttons.forEach(b => b.classList.remove('active'));
    document.getElementById(id).classList.add('active');
}

document.getElementById('noiseStr').oninput = (e) => {
    uniforms.uNoiseStrength.value = parseFloat(e.target.value);
};
document.getElementById('noiseSpd').oninput = (e) => {
    uniforms.uNoiseSpeed.value = parseFloat(e.target.value);
};

const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    
    const elapsed = clock.getElapsedTime();
    uniforms.uTime.value = elapsed;
    
    // Handle morphing
    if (isMorphing) {
        const progress = (elapsed - morphStartTime) / morphDuration;
        uniforms.uMorphProgress.value = Math.min(progress, 1.0);
        
        if (progress >= 1.0) {
            isMorphing = false;
            currentShape = targetShape;
        }
    }
    
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

---

## Demo 3: Vertex Color & gl_PointSize (Particle System)

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Vertex Color & Point Particles</title>
    <style>
        * { margin: 0; padding: 0; }
        canvas { display: block; background: #000; }
    </style>
</head>
<body>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';
import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.157.0/examples/jsm/controls/OrbitControls.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 100);
camera.position.set(0, 0, 5);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

// Particle Vertex Shader
const vertexShader = `
    uniform float uTime;
    uniform float uPixelRatio;
    uniform float uSize;
    
    attribute float aScale;    // ขนาดของ particle แต่ละตัว
    attribute float aOffset;   // offset เวลา (phase)
    attribute vec3  aColor;    // สีของ particle
    
    varying vec3 vColor;
    varying float vAlpha;
    
    void main() {
        vColor = aColor;
        
        vec3 pos = position;
        
        // Animate: orbit around original position
        float angle = uTime * 0.5 + aOffset * 6.28318;
        float radius = 0.3 + sin(aOffset * 3.14159 * 2.0 + uTime) * 0.2;
        
        pos.x += cos(angle) * radius * 0.5;
        pos.y += sin(angle * 1.3) * 0.3;
        pos.z += sin(angle) * radius * 0.5;
        
        // Pulsing alpha
        vAlpha = 0.5 + sin(uTime * 2.0 + aOffset * 6.28) * 0.5;
        
        // Transform
        vec4 mvPosition = modelViewMatrix * vec4(pos, 1.0);
        
        // gl_PointSize: ขนาดของ point ใน pixels
        // ยิ่งอยู่ไกล ยิ่งเล็ก (perspective)
        gl_PointSize = uSize * aScale * uPixelRatio * (300.0 / -mvPosition.z);
        
        gl_Position = projectionMatrix * mvPosition;
    }
`;

// Particle Fragment Shader
const fragmentShader = `
    precision mediump float;
    
    varying vec3 vColor;
    varying float vAlpha;
    
    void main() {
        // gl_PointCoord: UV coordinates ใน point (0-1)
        // ศูนย์กลาง point อยู่ที่ (0.5, 0.5)
        vec2 uv = gl_PointCoord - 0.5;
        
        // วงกลม
        float dist = length(uv);
        float alpha = smoothstep(0.5, 0.3, dist);
        
        // Soft glow
        float glow = exp(-dist * dist * 10.0);
        
        vec3 finalColor = vColor + glow * 0.5;
        float finalAlpha = alpha * vAlpha;
        
        if (finalAlpha < 0.01) discard; // ทิ้ง pixel ที่โปร่งใสมาก
        
        gl_FragColor = vec4(finalColor, finalAlpha);
    }
`;

// สร้าง particles
const count = 5000;
const positions = new Float32Array(count * 3);
const aScale = new Float32Array(count);
const aOffset = new Float32Array(count);
const aColor = new Float32Array(count * 3);

// Palettes
const palettes = [
    [1.0, 0.3, 0.1],  // ส้ม
    [0.1, 0.5, 1.0],  // น้ำเงิน
    [0.5, 1.0, 0.3],  // เขียว
    [1.0, 0.2, 0.8],  // ชมพู
    [1.0, 1.0, 0.2],  // เหลือง
];

for (let i = 0; i < count; i++) {
    const i3 = i * 3;
    
    // กระจาย particles เป็นทรงกลม
    const r = 1.5 + Math.random() * 0.5;
    const theta = Math.random() * Math.PI * 2;
    const phi = Math.acos(2 * Math.random() - 1);
    
    positions[i3]     = r * Math.sin(phi) * Math.cos(theta);
    positions[i3 + 1] = r * Math.sin(phi) * Math.sin(theta);
    positions[i3 + 2] = r * Math.cos(phi);
    
    aScale[i] = 0.5 + Math.random() * 1.5;
    aOffset[i] = Math.random();
    
    // เลือกสีจาก palette
    const palette = palettes[Math.floor(Math.random() * palettes.length)];
    aColor[i3]     = palette[0] * (0.7 + Math.random() * 0.3);
    aColor[i3 + 1] = palette[1] * (0.7 + Math.random() * 0.3);
    aColor[i3 + 2] = palette[2] * (0.7 + Math.random() * 0.3);
}

const geometry = new THREE.BufferGeometry();
geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
geometry.setAttribute('aScale', new THREE.BufferAttribute(aScale, 1));
geometry.setAttribute('aOffset', new THREE.BufferAttribute(aOffset, 1));
geometry.setAttribute('aColor', new THREE.BufferAttribute(aColor, 3));

const material = new THREE.ShaderMaterial({
    vertexShader,
    fragmentShader,
    uniforms: {
        uTime:       { value: 0 },
        uPixelRatio: { value: Math.min(devicePixelRatio, 2) },
        uSize:       { value: 30.0 }
    },
    transparent: true,
    blending: THREE.AdditiveBlending, // Additive blending = เรืองแสง!
    depthWrite: false                  // ปิด depth write สำหรับ particles
});

const particles = new THREE.Points(geometry, material);
scene.add(particles);

const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    material.uniforms.uTime.value = clock.getElapsedTime();
    controls.update();
    renderer.render(scene, camera);
}
animate();

window.addEventListener('resize', () => {
    camera.aspect = innerWidth / innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(innerWidth, innerHeight);
    material.uniforms.uPixelRatio.value = Math.min(devicePixelRatio, 2);
});
</script>
</body>
</html>
```

---

## Vertex Displacement Techniques

### Technique 1: Height Map Displacement
```glsl
uniform sampler2D uHeightMap;
uniform float uDisplacementScale;

varying vec2 vUv;

void main() {
    vUv = uv;
    
    // อ่านค่าจาก height map texture
    float height = texture2D(uHeightMap, uv).r;
    
    // เลื่อน vertex ตาม normal direction
    vec3 displacedPos = position + normal * height * uDisplacementScale;
    
    gl_Position = projectionMatrix * modelViewMatrix * vec4(displacedPos, 1.0);
}
```

### Technique 2: Procedural Displacement
```glsl
uniform float uTime;
uniform float uNoiseFreq;
uniform float uNoiseAmp;

void main() {
    vec3 pos = position;
    
    // Procedural displacement โดยไม่ต้องใช้ texture
    float disp = sin(pos.x * uNoiseFreq + uTime) * 
                 cos(pos.y * uNoiseFreq * 0.7 + uTime * 0.8) * 
                 uNoiseAmp;
    
    pos += normal * disp;
    
    gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
}
```

### Technique 3: Explosion Effect
```glsl
uniform float uProgress; // 0 = ปกติ, 1 = exploded

void main() {
    vec3 pos = position;
    
    // Direction ออกจากจุดกำเนิด
    vec3 dir = normalize(pos);
    
    // Explode ออกไปตาม direction
    float t = smoothstep(0.0, 1.0, uProgress);
    pos = mix(pos, pos + dir * 3.0, t);
    
    // Rotate ขณะ explode
    float angle = t * 6.28318;
    float s = sin(angle);
    float c = cos(angle);
    
    // Apply y-rotation
    float newX = pos.x * c - pos.z * s;
    float newZ = pos.x * s + pos.z * c;
    pos.x = newX;
    pos.z = newZ;
    
    gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
}
```

---

## gl_PointSize ในเชิงลึก

```glsl
// gl_PointSize กำหนดขนาดของ point ใน pixels
// ใช้ได้เฉพาะเมื่อ draw mode เป็น POINTS (THREE.Points)

void main() {
    // ขนาดคงที่
    gl_PointSize = 10.0;
    
    // ขนาดขึ้นกับระยะห่าง (perspective correct)
    vec4 mvPos = modelViewMatrix * vec4(position, 1.0);
    gl_PointSize = 50.0 / -mvPos.z;
    
    // ขนาดขึ้นกับ attribute
    gl_PointSize = aSize * (pixelRatio * 200.0 / -mvPos.z);
    
    // ขนาดขึ้นกับเวลา (pulsing)
    float pulse = 0.8 + sin(uTime * 3.0 + aOffset * 6.28) * 0.2;
    gl_PointSize = aSize * pulse;
    
    gl_Position = projectionMatrix * mvPos;
}
```

---

## Normal Transformation

```glsl
// Normal ต้องใช้ normalMatrix (ไม่ใช่ modelMatrix!) เพื่อ transform
// normalMatrix = transpose(inverse(modelViewMatrix)) สำหรับ top-left 3x3

varying vec3 vNormal;

void main() {
    // ถูกต้อง: ใช้ normalMatrix
    vNormal = normalize(normalMatrix * normal);
    
    // ผิด: normal จะบิดเบี้ยวถ้า model มี non-uniform scale
    // vNormal = normalize((modelViewMatrix * vec4(normal, 0.0)).xyz);
    
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}
```

---

## Advanced: Instance Vertex Shader

```javascript
// Instanced mesh + custom vertex shader
const count = 100;
const geometry = new THREE.BoxGeometry(0.1, 0.1, 0.1);

// สร้าง custom InstancedBufferGeometry
const instancedGeometry = geometry.clone();
const instancePositions = new Float32Array(count * 3);
const instanceColors = new Float32Array(count * 3);

for (let i = 0; i < count; i++) {
    instancePositions[i * 3]     = (Math.random() - 0.5) * 10;
    instancePositions[i * 3 + 1] = (Math.random() - 0.5) * 10;
    instancePositions[i * 3 + 2] = (Math.random() - 0.5) * 10;
    
    instanceColors[i * 3]     = Math.random();
    instanceColors[i * 3 + 1] = Math.random();
    instanceColors[i * 3 + 2] = Math.random();
}

instancedGeometry.setAttribute(
    'instancePosition', 
    new THREE.InstancedBufferAttribute(instancePositions, 3)
);
instancedGeometry.setAttribute(
    'instanceColor', 
    new THREE.InstancedBufferAttribute(instanceColors, 3)
);
```

```glsl
// Instance Vertex Shader
attribute vec3 instancePosition;
attribute vec3 instanceColor;

varying vec3 vInstanceColor;

void main() {
    vInstanceColor = instanceColor;
    
    // ย้าย vertex ตาม instance position
    vec3 worldPos = position + instancePosition;
    
    gl_Position = projectionMatrix * modelViewMatrix * vec4(worldPos, 1.0);
}
```

---

## สรุป Vertex Shader Best Practices

```
1. Minimize ตำแหน่งที่คำนวณใน vertex shader → ใช้ varyings ส่งไป fragment
2. ใช้ normalMatrix สำหรับ transform normals (ไม่ใช่ modelMatrix)
3. gl_PointSize ต้องไม่เกิน maxPointSize ของ GPU (ตรวจด้วย renderer.capabilities)
4. Custom attributes ต้องตั้งชื่อขึ้นต้นด้วย "a" (convention) เช่น aOffset, aScale
5. Varyings ควรตั้งชื่อขึ้นต้นด้วย "v" เช่น vUv, vNormal
6. Uniforms ตั้งชื่อขึ้นต้นด้วย "u" เช่น uTime, uColor
```

---

## แบบฝึกหัด

### ระดับ 1: พื้นฐาน
1. สร้าง plane ที่ vertices เคลื่อนขึ้นลงแบบ sine wave ตามแกน X
2. สร้าง sphere ที่ขยาย/หดตามเวลา โดยใช้ uniform `uScale`
3. เพิ่ม varying `vHeight` ส่งความสูงของ vertex ไป fragment shader

### ระดับ 2: กลาง
4. สร้าง terrain โดยใช้ procedural noise ใน vertex shader
5. สร้าง particle system (Points) ที่ particles โคจรรอบจุดกำเนิด
6. ทำ vertex color gradient โดยสีเปลี่ยนตาม world position Y

### ระดับ 3: ขั้นสูง
7. สร้าง morphing animation ระหว่าง sphere กับ cube โดยใช้ uniform `uProgress`
8. สร้าง explosion effect ที่ pieces กระจายออกไปพร้อม rotation
9. สร้าง instanced particle system ที่แต่ละ instance มีขนาด, สี, speed ต่างกัน

### Challenge
10. สร้าง "Black Hole" simulation โดย:
    - Particles โคจรรอบจุดศูนย์กลาง
    - Vertices ถูกดึงเข้าหาจุดกำเนิด
    - ยิ่งใกล้ศูนย์กลาง ยิ่งเร็วขึ้น
    - สีเปลี่ยนตามความเร็ว
