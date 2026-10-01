# Part 28: Textures in Shaders

## บทนำ: Texture ใน Shader คืออะไร?

Texture ใน shader คือการ sample สีจาก image/buffer ณ ตำแหน่ง UV ที่กำหนด GPU มี texture units ที่ optimize สำหรับการ sample โดยเฉพาะ รวมถึง:
- **Bilinear filtering** - interpolate ระหว่าง pixels
- **Mipmapping** - ใช้ resolution ที่เหมาะสมตามระยะห่าง
- **Anisotropic filtering** - คุณภาพดีกว่าเมื่อ texture เอียง

---

## sampler2D: พื้นฐาน

```glsl
// ประกาศ sampler2D
uniform sampler2D uTexture;

// Sample texture
vec4 color = texture2D(uTexture, vUv);   // GLSL 1.0
vec4 color2 = texture(uTexture, vUv);    // GLSL 3.0 (recommended)

// Components
float r = color.r;   // แดง
float g = color.g;   // เขียว
float b = color.b;   // น้ำเงิน
float a = color.a;   // alpha
vec3 rgb = color.rgb;
```

### JavaScript: ส่ง Texture ไปยัง Shader

```javascript
import * as THREE from 'three';

// สร้าง Texture
const textureLoader = new THREE.TextureLoader();
const texture = textureLoader.load('/textures/earth.jpg', (tex) => {
    // Callback เมื่อ load เสร็จ
    console.log('Texture loaded:', tex.image.width, tex.image.height);
});

// Configure texture
texture.wrapS = THREE.RepeatWrapping;
texture.wrapT = THREE.RepeatWrapping;
texture.repeat.set(2, 2);
texture.minFilter = THREE.LinearMipmapLinearFilter;
texture.magFilter = THREE.LinearFilter;
texture.generateMipmaps = true;
texture.anisotropy = renderer.capabilities.getMaxAnisotropy();

// ส่งให้ ShaderMaterial
const material = new THREE.ShaderMaterial({
    uniforms: {
        uTexture: { value: texture },
        uTime: { value: 0 }
    },
    vertexShader: `...`,
    fragmentShader: `
        uniform sampler2D uTexture;
        varying vec2 vUv;
        
        void main() {
            gl_FragColor = texture2D(uTexture, vUv);
        }
    `
});
```

---

## UV Distortion

```glsl
// UV Distortion เป็นเทคนิคที่ทรงพลังมาก
// เพียงแค่เปลี่ยน UV ก่อน sample ก็ได้ effects มากมาย

uniform float uTime;
uniform sampler2D uTexture;
varying vec2 vUv;

void main() {
    vec2 uv = vUv;
    
    // 1. Simple wave distortion
    uv.x += sin(uv.y * 10.0 + uTime) * 0.01;
    uv.y += cos(uv.x * 10.0 + uTime) * 0.01;
    
    // 2. Radial distortion (fisheye / barrel)
    vec2 centered = uv - 0.5;
    float dist = length(centered);
    float radialFactor = 1.0 + dist * dist * 0.5;
    uv = centered * radialFactor + 0.5;
    
    // 3. Swirl/vortex effect
    vec2 swirl = uv - 0.5;
    float angle = length(swirl) * 5.0 - uTime;
    float s = sin(angle);
    float c = cos(angle);
    uv = vec2(swirl.x * c - swirl.y * s, swirl.x * s + swirl.y * c) + 0.5;
    
    // 4. Chromatic aberration (different UV per channel)
    float aberrationAmount = 0.01;
    vec2 uvR = uv + vec2(aberrationAmount, 0.0);
    vec2 uvG = uv;
    vec2 uvB = uv - vec2(aberrationAmount, 0.0);
    
    float r = texture2D(uTexture, uvR).r;
    float g = texture2D(uTexture, uvG).g;
    float b = texture2D(uTexture, uvB).b;
    
    gl_FragColor = vec4(r, g, b, 1.0);
}
```

---

## Texture Blending

```glsl
// Blend หลาย textures เข้าด้วยกัน
uniform sampler2D uTexture1;
uniform sampler2D uTexture2;
uniform sampler2D uBlendMask; // mask สำหรับกำหนดสัดส่วน blend
uniform float uBlendFactor;   // global blend factor (0-1)

varying vec2 vUv;
varying float vElevation;    // จาก vertex shader

void main() {
    vec4 tex1 = texture2D(uTexture1, vUv);
    vec4 tex2 = texture2D(uTexture2, vUv);
    
    // Blend mode 1: Linear blend
    vec4 linear = mix(tex1, tex2, uBlendFactor);
    
    // Blend mode 2: Mask-based blend
    float mask = texture2D(uBlendMask, vUv).r;
    vec4 masked = mix(tex1, tex2, mask);
    
    // Blend mode 3: Height-based (terrain)
    float heightBlend = smoothstep(0.3, 0.7, vElevation);
    vec4 heightMixed = mix(tex1, tex2, heightBlend);
    
    // Blend mode 4: Multiply
    vec4 multiply = tex1 * tex2;
    
    // Blend mode 5: Screen
    vec4 screen = 1.0 - (1.0 - tex1) * (1.0 - tex2);
    
    // Blend mode 6: Overlay
    vec4 overlay;
    overlay.rgb = mix(
        2.0 * tex1.rgb * tex2.rgb,
        1.0 - 2.0 * (1.0 - tex1.rgb) * (1.0 - tex2.rgb),
        step(0.5, tex1.rgb)
    );
    overlay.a = 1.0;
    
    // Blend mode 7: Soft Light
    vec4 softLight;
    softLight.rgb = mix(
        2.0 * tex1.rgb * tex2.rgb + tex1.rgb * tex1.rgb * (1.0 - 2.0 * tex2.rgb),
        sqrt(tex1.rgb) * (2.0 * tex2.rgb - 1.0) + 2.0 * tex1.rgb * (1.0 - tex2.rgb),
        step(0.5, tex2.rgb)
    );
    softLight.a = 1.0;
    
    gl_FragColor = heightMixed;
}
```

---

## Normal Mapping ใน Shader

```glsl
// Normal mapping: เพิ่ม surface detail โดยไม่เพิ่ม geometry

uniform sampler2D uNormalMap;
uniform float uNormalScale;

varying vec2 vUv;
varying vec3 vNormal;
varying vec3 vTangent;
varying vec3 vBitangent;
varying vec3 vViewDir;

void main() {
    // Sample normal map (ค่าในช่วง [0,1])
    vec3 normalSample = texture2D(uNormalMap, vUv).xyz;
    
    // Decode: [0,1] → [-1,1]
    vec3 tangentNormal = normalSample * 2.0 - 1.0;
    
    // Scale XY components (normal intensity)
    tangentNormal.xy *= uNormalScale;
    tangentNormal = normalize(tangentNormal);
    
    // TBN Matrix: แปลง tangent space → world space
    // T = tangent, B = bitangent, N = normal
    mat3 TBN = mat3(
        normalize(vTangent),
        normalize(vBitangent),
        normalize(vNormal)
    );
    
    // แปลง normal จาก tangent space → world space
    vec3 worldNormal = normalize(TBN * tangentNormal);
    
    // ใช้ worldNormal สำหรับ lighting
    vec3 lightDir = normalize(vec3(1.0, 2.0, 1.0));
    float diffuse = max(0.0, dot(worldNormal, lightDir));
    
    gl_FragColor = vec4(vec3(0.8) * diffuse, 1.0);
}
```

### Vertex Shader สำหรับ Normal Mapping
```glsl
// ต้องส่ง tangent ผ่าน varying ด้วย
attribute vec4 tangent;  // three.js ส่ง tangent ให้อัตโนมัติถ้า geometry.computeTangents()

varying vec2 vUv;
varying vec3 vNormal;
varying vec3 vTangent;
varying vec3 vBitangent;
varying vec3 vViewDir;

void main() {
    vUv = uv;
    
    // Transform normal
    vec3 worldNormal   = normalize((modelMatrix * vec4(normal, 0.0)).xyz);
    vec3 worldTangent  = normalize((modelMatrix * vec4(tangent.xyz, 0.0)).xyz);
    
    // Gram-Schmidt re-orthogonalization
    worldTangent = normalize(worldTangent - dot(worldTangent, worldNormal) * worldNormal);
    
    // Bitangent: cross product of normal and tangent
    // tangent.w คือ handedness (+1 หรือ -1)
    vec3 worldBitangent = cross(worldNormal, worldTangent) * tangent.w;
    
    vNormal    = worldNormal;
    vTangent   = worldTangent;
    vBitangent = worldBitangent;
    
    vec4 worldPos = modelMatrix * vec4(position, 1.0);
    vViewDir = normalize(cameraPosition - worldPos.xyz);
    
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}
```

```javascript
// JavaScript: เตรียม geometry สำหรับ normal mapping
const geometry = new THREE.SphereGeometry(1, 64, 64);
geometry.computeTangents(); // ← จำเป็น! สร้าง tangent attribute

const normalMap = textureLoader.load('/textures/normal.jpg');

const material = new THREE.ShaderMaterial({
    uniforms: {
        uNormalMap:   { value: normalMap },
        uNormalScale: { value: 1.0 }
    },
    // ...
});
```

---

## Multiple Texture Samplers

```glsl
// ใช้ multiple textures ใน shader เดียว

uniform sampler2D uDiffuse;   // สีพื้นฐาน
uniform sampler2D uNormal;    // Normal map
uniform sampler2D uRoughness; // Roughness map
uniform sampler2D uMetallic;  // Metallic map
uniform sampler2D uAO;        // Ambient Occlusion
uniform sampler2D uEmissive;  // Emissive map

void main() {
    // Sample all maps
    vec4 diffuse   = texture2D(uDiffuse,   vUv);
    vec3 normal    = texture2D(uNormal,    vUv).xyz * 2.0 - 1.0;
    float rough    = texture2D(uRoughness, vUv).g;
    float metallic = texture2D(uMetallic,  vUv).b;
    float ao       = texture2D(uAO,        vUv).r;
    vec3  emissive = texture2D(uEmissive,  vUv).rgb;
    
    // ใช้ค่าเหล่านี้ใน PBR lighting
    // ...
}
```

### GPU Texture Unit Limits
```javascript
// ตรวจสอบจำนวน texture units ที่รองรับ
const maxTextures = renderer.capabilities.maxTextures;
console.log('Max textures:', maxTextures); // ปกติ 16-32

// แต่ละ texture ใช้ 1 texture unit
// Fragment shader: maxTextureImageUnits
// Vertex shader: maxVertexTextureImageUnits (มักน้อยกว่า)
```

---

## Cubemap / Environment Map

```glsl
// samplerCube สำหรับ cubemap
uniform samplerCube uEnvMap;
uniform float uEnvIntensity;

varying vec3 vWorldNormal;
varying vec3 vWorldPos;

void main() {
    vec3 viewDir = normalize(cameraPosition - vWorldPos);
    
    // Reflection direction
    vec3 reflectDir = reflect(-viewDir, normalize(vWorldNormal));
    
    // Refraction direction
    float ior = 1.5; // Index of refraction (glass)
    vec3 refractDir = refract(-viewDir, normalize(vWorldNormal), 1.0 / ior);
    
    // Sample cubemap
    vec4 reflection = textureCube(uEnvMap, reflectDir); // GLSL 1.0
    // หรือ: vec4 reflection = texture(uEnvMap, reflectDir); // GLSL 3.0
    
    vec4 refraction = textureCube(uEnvMap, refractDir);
    
    // Fresnel blend: reflection vs refraction
    float cosTheta = abs(dot(viewDir, normalize(vWorldNormal)));
    float fresnel = pow(1.0 - cosTheta, 3.0);
    
    vec4 envColor = mix(refraction, reflection, fresnel);
    
    gl_FragColor = envColor * uEnvIntensity;
}
```

---

## Render Target as Texture

```javascript
// สร้าง render target
const renderTarget = new THREE.WebGLRenderTarget(512, 512, {
    minFilter: THREE.LinearFilter,
    magFilter: THREE.LinearFilter,
    format: THREE.RGBAFormat
});

// Render scene ลงใน render target
renderer.setRenderTarget(renderTarget);
renderer.render(rtScene, rtCamera);
renderer.setRenderTarget(null); // กลับมา render ปกติ

// ใช้ texture จาก render target
const material = new THREE.ShaderMaterial({
    uniforms: {
        uRenderTarget: { value: renderTarget.texture }
    },
    fragmentShader: `
        uniform sampler2D uRenderTarget;
        varying vec2 vUv;
        
        void main() {
            vec4 color = texture2D(uRenderTarget, vUv);
            // Post-processing effects...
            gl_FragColor = color;
        }
    `
});

// Double buffering (ping-pong)
const pingpong = [
    new THREE.WebGLRenderTarget(512, 512),
    new THREE.WebGLRenderTarget(512, 512)
];

let pingpongIndex = 0;

function processPingPong() {
    const read = pingpong[pingpongIndex];
    const write = pingpong[1 - pingpongIndex];
    
    material.uniforms.uPrevFrame.value = read.texture;
    renderer.setRenderTarget(write);
    renderer.render(processScene, camera);
    renderer.setRenderTarget(null);
    
    pingpongIndex = 1 - pingpongIndex;
}
```

---

## Demo: UV Warp Shader

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>UV Warp Shader</title>
    <style>
        * { margin: 0; padding: 0; }
        canvas { display: block; }
        #ui {
            position: fixed;
            top: 15px;
            left: 15px;
            color: white;
            font-family: monospace;
            background: rgba(0,0,0,0.8);
            padding: 12px;
            border-radius: 8px;
        }
        select, input { width: 160px; display: block; margin: 4px 0; }
    </style>
</head>
<body>
<div id="ui">
    <b>UV Warp Demo</b>
    Effect:
    <select id="effect">
        <option value="0">None</option>
        <option value="1">Sine Wave</option>
        <option value="2">Swirl/Vortex</option>
        <option value="3">Fisheye</option>
        <option value="4">Chromatic Aberration</option>
        <option value="5">Noise Warp</option>
        <option value="6">Ripple</option>
        <option value="7">Kaleidoscope</option>
    </select>
    Intensity: <input type="range" id="intensity" min="0" max="2" value="0.5" step="0.01">
    Speed: <input type="range" id="speed" min="0" max="3" value="1" step="0.1">
</div>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, innerWidth / innerHeight, 0.1, 10);
camera.position.z = 1;
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
document.body.appendChild(renderer.domElement);

// สร้าง procedural texture ใน canvas
const canvas = document.createElement('canvas');
canvas.width = canvas.height = 512;
const ctx = canvas.getContext('2d');

// วาด grid pattern
ctx.fillStyle = '#1a1a2e';
ctx.fillRect(0, 0, 512, 512);
for (let i = 0; i < 8; i++) {
    for (let j = 0; j < 8; j++) {
        const x = i * 64, y = j * 64;
        const hue = (i + j) / 14 * 360;
        ctx.fillStyle = `hsl(${hue}, 70%, 50%)`;
        ctx.fillRect(x + 4, y + 4, 56, 56);
    }
}

// เพิ่ม text
ctx.fillStyle = 'white';
ctx.font = 'bold 40px monospace';
ctx.textAlign = 'center';
ctx.fillText('UV WARP', 256, 256);
ctx.font = '20px monospace';
ctx.fillText('Three.js Shaders', 256, 295);

const texture = new THREE.CanvasTexture(canvas);

const fragmentShader = `
    precision highp float;
    
    uniform sampler2D uTexture;
    uniform float uTime;
    uniform int   uEffect;
    uniform float uIntensity;
    uniform float uSpeed;
    uniform vec2  uResolution;
    
    varying vec2 vUv;
    
    // Hash noise
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
    
    void main() {
        vec2 uv = vUv;
        float t = uTime * uSpeed;
        float intensity = uIntensity;
        
        if (uEffect == 0) {
            // No warp
            
        } else if (uEffect == 1) {
            // Sine Wave
            uv.x += sin(uv.y * 10.0 + t) * intensity * 0.1;
            uv.y += sin(uv.x * 10.0 + t * 0.7) * intensity * 0.1;
            
        } else if (uEffect == 2) {
            // Swirl/Vortex
            vec2 centered = uv - 0.5;
            float dist = length(centered);
            float angle = dist * intensity * 10.0 - t;
            float s = sin(angle);
            float c = cos(angle);
            uv = vec2(
                centered.x * c - centered.y * s,
                centered.x * s + centered.y * c
            ) + 0.5;
            
        } else if (uEffect == 3) {
            // Fisheye/Barrel Distortion
            vec2 centered = uv - 0.5;
            float dist = length(centered);
            float barrel = 1.0 + dist * dist * intensity * 2.0;
            uv = centered * barrel + 0.5;
            
        } else if (uEffect == 4) {
            // Chromatic Aberration (วาดแยก channel)
            float amount = intensity * 0.02;
            vec2 dir = normalize(uv - 0.5);
            
            float r = texture2D(uTexture, uv + dir * amount).r;
            float g = texture2D(uTexture, uv).g;
            float b = texture2D(uTexture, uv - dir * amount).b;
            
            gl_FragColor = vec4(r, g, b, 1.0);
            return;
            
        } else if (uEffect == 5) {
            // Noise Warp
            float nx = valueNoise(uv * 3.0 + vec2(t, 0.0));
            float ny = valueNoise(uv * 3.0 + vec2(0.0, t));
            uv.x += (nx - 0.5) * intensity * 0.2;
            uv.y += (ny - 0.5) * intensity * 0.2;
            
        } else if (uEffect == 6) {
            // Ripple from center
            vec2 centered = uv - 0.5;
            float dist = length(centered);
            float ripple = sin(dist * 30.0 - t * 3.0) * intensity * 0.03;
            uv += normalize(centered) * ripple;
            
        } else {
            // Kaleidoscope
            vec2 centered = uv - 0.5;
            float angle = atan(centered.y, centered.x);
            float dist2 = length(centered);
            float k = 6.0; // จำนวนซ้ำ
            angle = mod(angle, 3.14159 * 2.0 / k);
            angle = abs(angle - 3.14159 / k);
            uv = vec2(cos(angle), sin(angle)) * dist2 + 0.5;
            uv.x += t * intensity * 0.05; // rotate slowly
        }
        
        // Clamp UV
        vec4 color = texture2D(uTexture, clamp(uv, 0.0, 1.0));
        
        // Edge darkening สำหรับ effects ที่ stretch
        float edgeDarken = 1.0;
        if (uv.x < 0.0 || uv.x > 1.0 || uv.y < 0.0 || uv.y > 1.0) {
            edgeDarken = 0.0;
        }
        
        gl_FragColor = vec4(color.rgb * edgeDarken, 1.0);
    }
`;

const geometry = new THREE.PlaneGeometry(2, 2);
const uniforms = {
    uTexture:    { value: texture },
    uTime:       { value: 0 },
    uEffect:     { value: 0 },
    uIntensity:  { value: 0.5 },
    uSpeed:      { value: 1.0 },
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

document.getElementById('effect').onchange = (e) => uniforms.uEffect.value = parseInt(e.target.value);
document.getElementById('intensity').oninput = (e) => uniforms.uIntensity.value = parseFloat(e.target.value);
document.getElementById('speed').oninput = (e) => uniforms.uSpeed.value = parseFloat(e.target.value);

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
});
</script>
</body>
</html>
```

---

## Demo: Multi-Texture Blend Shader

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Multi-Texture Blend</title>
    <style>
        * { margin: 0; padding: 0; }
        canvas { display: block; }
        #ui {
            position: fixed;
            bottom: 20px;
            left: 50%;
            transform: translateX(-50%);
            color: white;
            font-family: monospace;
            background: rgba(0,0,0,0.8);
            padding: 12px 20px;
            border-radius: 8px;
            display: flex;
            gap: 20px;
            align-items: center;
        }
    </style>
</head>
<body>
<div id="ui">
    Blend Mode:
    <select id="blendMode">
        <option value="0">Normal</option>
        <option value="1">Height-based</option>
        <option value="2">Multiply</option>
        <option value="3">Screen</option>
        <option value="4">Overlay</option>
        <option value="5">Animated Mix</option>
    </select>
    Mix Factor: <input type="range" id="mix" min="0" max="1" value="0.5" step="0.01">
    Speed: <input type="range" id="speed" min="0" max="3" value="1" step="0.1">
</div>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';
import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.157.0/examples/jsm/controls/OrbitControls.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111122);
const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 100);
camera.position.set(0, 0, 2.5);
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
document.body.appendChild(renderer.domElement);
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

// สร้าง procedural textures
function createTexture(drawFn, size = 256) {
    const canvas = document.createElement('canvas');
    canvas.width = canvas.height = size;
    const ctx = canvas.getContext('2d');
    drawFn(ctx, size);
    return new THREE.CanvasTexture(canvas);
}

// Texture 1: Grid pattern
const tex1 = createTexture((ctx, s) => {
    ctx.fillStyle = '#2244aa';
    ctx.fillRect(0, 0, s, s);
    for (let i = 0; i < 4; i++) {
        for (let j = 0; j < 4; j++) {
            ctx.fillStyle = `hsl(${i * 30 + j * 20 + 200}, 70%, 60%)`;
            ctx.fillRect(i * s/4 + 4, j * s/4 + 4, s/4 - 8, s/4 - 8);
        }
    }
});

// Texture 2: Gradient dots
const tex2 = createTexture((ctx, s) => {
    const grd = ctx.createRadialGradient(s/2, s/2, 0, s/2, s/2, s/2);
    grd.addColorStop(0, '#ff8833');
    grd.addColorStop(1, '#aa2255');
    ctx.fillStyle = grd;
    ctx.fillRect(0, 0, s, s);
    ctx.fillStyle = 'rgba(255,255,255,0.3)';
    for (let i = 0; i < 50; i++) {
        ctx.beginPath();
        ctx.arc(Math.random() * s, Math.random() * s, 3 + Math.random() * 8, 0, Math.PI * 2);
        ctx.fill();
    }
});

// Noise texture สำหรับ mask
const tex3 = createTexture((ctx, s) => {
    for (let y = 0; y < s; y++) {
        for (let x = 0; x < s; x++) {
            const v = Math.floor(Math.random() * 255);
            ctx.fillStyle = `rgb(${v},${v},${v})`;
            ctx.fillRect(x, y, 1, 1);
        }
    }
});

const fragmentShader = `
    precision mediump float;
    
    uniform sampler2D uTex1;
    uniform sampler2D uTex2;
    uniform sampler2D uMask;
    uniform float uTime;
    uniform int   uBlendMode;
    uniform float uMixFactor;
    uniform float uSpeed;
    
    varying vec2 vUv;
    varying float vElevation;
    
    // ฟังก์ชัน Blend Modes
    vec3 blendNormal(vec3 base, vec3 blend, float opacity) {
        return mix(base, blend, opacity);
    }
    
    vec3 blendMultiply(vec3 base, vec3 blend) {
        return base * blend;
    }
    
    vec3 blendScreen(vec3 base, vec3 blend) {
        return 1.0 - (1.0 - base) * (1.0 - blend);
    }
    
    vec3 blendOverlay(vec3 base, vec3 blend) {
        return mix(
            2.0 * base * blend,
            1.0 - 2.0 * (1.0 - base) * (1.0 - blend),
            step(0.5, base)
        );
    }
    
    void main() {
        vec2 uv = vUv;
        
        vec4 c1 = texture2D(uTex1, uv);
        vec4 c2 = texture2D(uTex2, uv);
        float mask = texture2D(uMask, uv).r;
        
        vec3 result;
        float t = uTime * uSpeed;
        
        if (uBlendMode == 0) {
            // Normal blend with mix factor
            result = blendNormal(c1.rgb, c2.rgb, uMixFactor);
            
        } else if (uBlendMode == 1) {
            // Height-based blend (animate vElevation)
            float heightFactor = sin(uv.x * 5.0 + t) * 0.5 + 0.5;
            result = mix(c1.rgb, c2.rgb, heightFactor);
            
        } else if (uBlendMode == 2) {
            // Multiply
            vec3 mult = blendMultiply(c1.rgb, c2.rgb);
            result = mix(c1.rgb, mult, uMixFactor);
            
        } else if (uBlendMode == 3) {
            // Screen
            vec3 screen = blendScreen(c1.rgb, c2.rgb);
            result = mix(c1.rgb, screen, uMixFactor);
            
        } else if (uBlendMode == 4) {
            // Overlay
            vec3 overlay = blendOverlay(c1.rgb, c2.rgb);
            result = mix(c1.rgb, overlay, uMixFactor);
            
        } else {
            // Animated mask blend
            float animMask = mask + sin(uv.x * 8.0 + t) * 0.3 + sin(uv.y * 6.0 + t * 0.7) * 0.3;
            animMask = clamp(animMask, 0.0, 1.0);
            result = mix(c1.rgb, c2.rgb, animMask);
        }
        
        gl_FragColor = vec4(result, 1.0);
    }
`;

const geometry = new THREE.SphereGeometry(1, 128, 128);
const uniforms = {
    uTex1:      { value: tex1 },
    uTex2:      { value: tex2 },
    uMask:      { value: tex3 },
    uTime:      { value: 0 },
    uBlendMode: { value: 0 },
    uMixFactor: { value: 0.5 },
    uSpeed:     { value: 1.0 }
};

const material = new THREE.ShaderMaterial({
    vertexShader: `
        varying vec2 vUv;
        varying float vElevation;
        void main() {
            vUv = uv;
            vElevation = position.y;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
    `,
    fragmentShader,
    uniforms
});

scene.add(new THREE.Mesh(geometry, material));

document.getElementById('blendMode').onchange = (e) => uniforms.uBlendMode.value = parseInt(e.target.value);
document.getElementById('mix').oninput = (e) => uniforms.uMixFactor.value = parseFloat(e.target.value);
document.getElementById('speed').oninput = (e) => uniforms.uSpeed.value = parseFloat(e.target.value);

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

## Video Texture ใน Shader

```javascript
// Video texture
const video = document.createElement('video');
video.src = '/video/clip.mp4';
video.loop = true;
video.muted = true;
video.play();

const videoTexture = new THREE.VideoTexture(video);
videoTexture.minFilter = THREE.LinearFilter;
videoTexture.magFilter = THREE.LinearFilter;
videoTexture.format = THREE.RGBFormat;

const material = new THREE.ShaderMaterial({
    uniforms: {
        uVideo: { value: videoTexture },
        uTime:  { value: 0 }
    },
    fragmentShader: `
        uniform sampler2D uVideo;
        uniform float uTime;
        varying vec2 vUv;
        
        void main() {
            // Scanline effect
            float scanline = sin(vUv.y * 400.0 + uTime * 5.0) * 0.05;
            
            // Chromatic aberration
            float r = texture2D(uVideo, vUv + vec2(0.002, 0.0)).r;
            float g = texture2D(uVideo, vUv).g;
            float b = texture2D(uVideo, vUv - vec2(0.002, 0.0)).b;
            
            vec3 color = vec3(r, g, b) * (1.0 + scanline);
            
            // Vignette
            vec2 uv2 = vUv * 2.0 - 1.0;
            float vignette = 1.0 - dot(uv2, uv2) * 0.5;
            
            gl_FragColor = vec4(color * vignette, 1.0);
        }
    `
});

// Update video texture ทุก frame
function animate() {
    requestAnimationFrame(animate);
    
    if (video.readyState >= video.HAVE_CURRENT_DATA) {
        videoTexture.needsUpdate = true; // ← สำคัญ! บอก Three.js ให้ upload frame ใหม่
    }
    
    renderer.render(scene, camera);
}
```

---

## แบบฝึกหัด

### ระดับ 1
1. โหลด texture และแสดงบน plane ด้วย custom shader
2. เพิ่ม UV animation ให้ texture เลื่อนตามเวลา
3. ใช้ texture เป็น mask สำหรับ discard

### ระดับ 2
4. Implement 3 blend modes (multiply, screen, overlay)
5. สร้าง chromatic aberration effect
6. ทำ normal mapping บน sphere

### ระดับ 3
7. สร้าง render target และใช้เป็น texture ใน shader
8. Implement ripple effect บน texture
9. สร้าง environment reflection โดยใช้ cubemap

### Challenge
10. สร้าง Water Surface Effect:
    - Normal map animation (2 normal maps เลื่อนต่างทิศ)
    - Environment reflection
    - Fresnel blending
    - Underwater distortion
    - Foam บน high wave peaks
    - Caustics pattern
