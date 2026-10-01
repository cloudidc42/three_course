# Part 26: ShaderMaterial vs RawShaderMaterial

## ความแตกต่างพื้นฐาน

เมื่อเขียน custom shaders ใน Three.js เรามีตัวเลือก 2 หลัก:

| Feature | ShaderMaterial | RawShaderMaterial |
|---------|---------------|-------------------|
| Built-in Uniforms | ✅ อัตโนมัติ | ❌ ต้องเพิ่มเอง |
| Built-in Attributes | ✅ อัตโนมัติ | ❌ ต้องประกาศเอง |
| GLSL Version | GLSL 1.00 + Three.js defines | GLSL 1.00 หรือ 3.00 |
| Shader Chunks | ✅ ใช้ได้ | ❌ ไม่รองรับ |
| Preprocessor Defines | ✅ | ✅ |
| Complexity | น้อยกว่า | มากกว่า |
| Control | น้อยกว่า | เต็มที่ |

---

## ShaderMaterial: Built-in Uniforms & Attributes

Three.js inject ข้อมูลเหล่านี้เข้าใน ShaderMaterial อัตโนมัติ:

### Built-in Uniforms
```glsl
// === Transformation Matrices ===
uniform mat4 projectionMatrix;    // Projection (perspective/ortho)
uniform mat4 modelViewMatrix;     // Model × View (combined)
uniform mat4 modelMatrix;         // Model (world transform)
uniform mat4 viewMatrix;          // View (camera inverse)
uniform mat3 normalMatrix;        // For transforming normals = transpose(inverse(modelViewMatrix top-left 3x3))

// === Camera ===
uniform vec3 cameraPosition;     // Camera position ใน world space

// === Fog (ถ้า scene มี fog) ===
uniform float fogDensity;
uniform float fogNear;
uniform float fogFar;
uniform vec3  fogColor;

// === Lights (ถ้า lights: true) ===
uniform vec3 ambientLightColor;
// ... directional/point/spot lights (complex)

// === Other ===
uniform float logDepthBufFC;  // Logarithmic depth buffer
uniform float alphaTest;
```

### Built-in Attributes
```glsl
// === Geometry Attributes ===
attribute vec3 position;    // Vertex position (local space)
attribute vec3 normal;      // Vertex normal
attribute vec2 uv;          // Primary UV
attribute vec2 uv2;         // Secondary UV
attribute vec3 color;       // Vertex color
attribute vec4 tangent;     // Tangent (w=handedness: ±1)

// === Instancing (ถ้าใช้ InstancedMesh) ===
attribute mat4 instanceMatrix;   // Per-instance transform
attribute vec3 instanceColor;    // Per-instance color
```

### ตัวอย่าง ShaderMaterial ที่ใช้ built-ins ทั้งหมด
```javascript
const material = new THREE.ShaderMaterial({
    vertexShader: `
        // ไม่ต้องประกาศ projectionMatrix, modelViewMatrix, position, uv
        // Three.js inject เข้ามาให้อัตโนมัติ
        
        varying vec2 vUv;
        varying vec3 vWorldPos;
        
        void main() {
            vUv = uv;
            // World position จาก modelMatrix
            vWorldPos = (modelMatrix * vec4(position, 1.0)).xyz;
            
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
    `,
    fragmentShader: `
        precision mediump float;
        
        uniform vec3 cameraPosition;
        
        varying vec2 vUv;
        varying vec3 vWorldPos;
        
        void main() {
            // ระยะห่างจาก camera
            float dist = distance(vWorldPos, cameraPosition);
            float fog = 1.0 - exp(-dist * 0.1);
            
            vec3 color = vec3(vUv, 0.5);
            color = mix(color, vec3(0.8, 0.9, 1.0), fog);
            
            gl_FragColor = vec4(color, 1.0);
        }
    `
});
```

---

## RawShaderMaterial: Everything Manual

RawShaderMaterial ให้ control เต็มที่ แต่ต้องทำทุกอย่างเอง:

```javascript
const rawMaterial = new THREE.RawShaderMaterial({
    vertexShader: `
        // ต้องประกาศทุกอย่างเอง!
        
        // GLSL version (optional แต่แนะนำ)
        // #version 300 es  // ← ถ้าต้องการ GLSL ES 3.0
        
        // Precision
        precision mediump float;
        
        // Uniforms: ต้องประกาศเองทั้งหมด
        uniform mat4 projectionMatrix;
        uniform mat4 modelViewMatrix;
        
        // Attributes: ต้องประกาศเองทั้งหมด
        attribute vec3 position;
        attribute vec2 uv;
        
        // Varyings
        varying vec2 vUv;
        
        void main() {
            vUv = uv;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
    `,
    fragmentShader: `
        precision mediump float;
        
        varying vec2 vUv;
        
        void main() {
            gl_FragColor = vec4(vUv, 0.5, 1.0);
        }
    `,
    // uniforms ต้องมีทุกอย่างที่ประกาศไว้ใน shader
    uniforms: {
        projectionMatrix:  { value: new THREE.Matrix4() },
        modelViewMatrix:   { value: new THREE.Matrix4() }
    }
});
```

> **หมายเหตุ**: ใน RawShaderMaterial ถ้าประกาศ `projectionMatrix` และ `modelViewMatrix` ใน `uniforms`, Three.js จะ update ค่าเหล่านี้ให้อัตโนมัติ แต่ถ้าไม่ประกาศ จะไม่ได้ค่า

---

## GLSL Preprocessor Defines

```javascript
const material = new THREE.ShaderMaterial({
    defines: {
        USE_COLOR: true,        // boolean define
        MAX_LIGHTS: 4,          // number define
        QUALITY: 'HIGH',        // string define
        PI: Math.PI             // float define
    },
    vertexShader: `...`,
    fragmentShader: `...`
});
```

```glsl
// ใน shader code
#ifdef USE_COLOR
    // code นี้รันเฉพาะเมื่อ USE_COLOR เป็น true
    attribute vec3 color;
    varying vec3 vColor;
#endif

// ใช้ค่าตัวเลข
for (int i = 0; i < MAX_LIGHTS; i++) {
    // ...
}

void main() {
    float pi = PI; // ใช้ค่า PI ที่ define ไว้
    
    #ifdef USE_COLOR
        vColor = color;
    #endif
    
    // ...
}
```

### ตัวอย่าง Conditional Features
```glsl
// Preprocessor conditionals
#ifdef FEATURE_A
    // ...
#endif

#ifndef FEATURE_B
    // code ถ้าไม่มี FEATURE_B
#endif

#if MAX_LIGHTS > 2
    // code สำหรับ lights มากกว่า 2
#elif MAX_LIGHTS == 1
    // ...
#else
    // ...
#endif

// Three.js built-in defines (อัตโนมัติ)
#ifdef USE_MAP
    // texture map enabled
#endif

#ifdef USE_NORMALMAP
    // normal map enabled
#endif

#ifdef USE_FOG
    // fog enabled
#endif
```

---

## Shader Chunks (Three.js Shader Library)

Three.js มี shader library ที่เรียกว่า "shader chunks" ซึ่งเป็น reusable code ที่ใช้ใน built-in materials

### วิธีใช้ Shader Chunks ใน ShaderMaterial

```javascript
// ดู Three.js ShaderChunk library
import * as THREE from 'three';
console.log(Object.keys(THREE.ShaderChunk));
// ['alphamap_fragment', 'alphamap_pars_fragment', 'alphatest_fragment', ...]

// Include ใน shader ด้วย #include
const fragmentShader = `
    // Include common utilities
    #include <common>
    
    // Include fog support
    #include <fog_pars_fragment>
    
    varying vec2 vUv;
    
    void main() {
        vec3 color = vec3(vUv, 0.5);
        gl_FragColor = vec4(color, 1.0);
        
        // Apply fog
        #include <fog_fragment>
    }
`;
```

### สำคัญ: Shader Chunk อยู่ใน `THREE.ShaderChunk`
```javascript
// เข้าถึง chunk content
const commonChunk = THREE.ShaderChunk['common'];
const fogChunk = THREE.ShaderChunk['fog_pars_fragment'];

// แสดงเนื้อหาของ chunk
console.log(commonChunk);
```

### ตัวอย่าง Common Chunks ที่ใช้บ่อย

```glsl
// common: utilities เช่น PI, EPSILON, saturate, etc.
#include <common>

// uv_pars_vertex: UV handling
#include <uv_pars_vertex>
// → เพิ่ม varying vec2 vUv; และ vUv = uv; อัตโนมัติ

// normal_pars_vertex: Normal handling
#include <normal_pars_vertex>

// fog_pars_vertex / fog_pars_fragment: Fog support
#include <fog_pars_vertex>
// ใน fragment:
#include <fog_pars_fragment>

// shadowmap_pars_vertex / shadowmap_pars_fragment: Shadows
#include <shadowmap_pars_vertex>

// dithering_fragment: Dithering for banding reduction
#include <dithering_fragment>

// tonemapping_fragment: Tone mapping
#include <tonemapping_fragment>

// colorspace_fragment: Color space conversion
#include <colorspace_fragment>
```

---

## Custom Shader Includes

เราสามารถเพิ่ม custom chunks เข้าไปใน Three.js shader library:

```javascript
// เพิ่ม custom chunk
THREE.ShaderChunk['myUtils'] = `
    // Custom utility functions
    vec3 gammaCorrect(vec3 color) {
        return pow(color, vec3(1.0/2.2));
    }
    
    float luminance(vec3 color) {
        return dot(color, vec3(0.299, 0.587, 0.114));
    }
    
    vec3 toneMap(vec3 color, float exposure) {
        color *= exposure;
        return color / (color + vec3(1.0)); // Reinhard tone mapping
    }
`;

// ใช้ใน shader
const fragmentShader = `
    #include <myUtils>
    
    void main() {
        vec3 rawColor = vec3(2.0, 1.5, 0.5); // HDR color
        vec3 mapped = toneMap(rawColor, 1.0);
        vec3 corrected = gammaCorrect(mapped);
        gl_FragColor = vec4(corrected, 1.0);
    }
`;
```

### Shader Injection (Modify Built-in Materials)
```javascript
// แก้ไข built-in material shader โดยใช้ onBeforeCompile
const material = new THREE.MeshStandardMaterial({ color: 0xff0000 });

material.onBeforeCompile = (shader) => {
    // เพิ่ม uniform
    shader.uniforms.uTime = { value: 0 };
    
    // แก้ vertex shader
    shader.vertexShader = shader.vertexShader.replace(
        '#include <begin_vertex>',
        `
        #include <begin_vertex>
        
        // เพิ่ม wave displacement
        transformed.y += sin(position.x * 5.0 + uTime) * 0.1;
        `
    );
    
    // บันทึก uniform reference ไว้ update ทีหลัง
    material.userData.shader = shader;
};

// Update uniform ใน animation loop
function animate() {
    if (material.userData.shader) {
        material.userData.shader.uniforms.uTime.value = clock.getElapsedTime();
    }
}
```

---

## Demo: Complete Comparison

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>ShaderMaterial vs RawShaderMaterial</title>
    <style>
        * { margin: 0; padding: 0; }
        canvas { display: block; }
        #ui {
            position: fixed;
            top: 20px;
            left: 20px;
            color: white;
            font-family: monospace;
            background: rgba(0,0,0,0.8);
            padding: 15px;
            border-radius: 8px;
        }
        .label {
            display: inline-block;
            padding: 3px 8px;
            border-radius: 3px;
            font-size: 12px;
            margin: 3px;
        }
        .sm { background: #2a5aaa; }
        .rsm { background: #aa2a2a; }
        .ms { background: #2aaa2a; }
    </style>
</head>
<body>
<div id="ui">
    <div>Material Comparison</div>
    <br>
    <span class="label sm">Left: ShaderMaterial</span><br>
    <span class="label rsm">Mid: RawShaderMaterial</span><br>
    <span class="label ms">Right: MeshStandard + onBeforeCompile</span><br>
    <br>
    <div id="fps">FPS: --</div>
</div>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';
import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.157.0/examples/jsm/controls/OrbitControls.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111122);
const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 100);
camera.position.set(0, 2, 6);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

// Add Lights
const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
scene.add(ambientLight);
const dirLight = new THREE.DirectionalLight(0xffffff, 1.0);
dirLight.position.set(5, 5, 5);
scene.add(dirLight);

// === 1. ShaderMaterial ===
// Three.js inject projectionMatrix, modelViewMatrix, position, normal, uv automatically
const shaderMaterial = new THREE.ShaderMaterial({
    uniforms: {
        uTime:  { value: 0 },
        uColor: { value: new THREE.Color(0x4488ff) }
    },
    vertexShader: `
        // ไม่ต้องประกาศ projectionMatrix, modelViewMatrix, position, normal, uv
        uniform float uTime;
        varying vec2 vUv;
        varying vec3 vNormal;
        varying vec3 vWorldPos;
        
        void main() {
            vUv = uv;
            vNormal = normalize(normalMatrix * normal);
            
            vec3 pos = position;
            // Wave effect
            pos.y += sin(pos.x * 3.0 + uTime * 2.0) * 0.15;
            pos.y += sin(pos.z * 2.0 + uTime * 1.5) * 0.1;
            
            vec4 worldPos = modelMatrix * vec4(pos, 1.0);
            vWorldPos = worldPos.xyz;
            
            gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
        }
    `,
    fragmentShader: `
        precision mediump float;
        
        uniform float uTime;
        uniform vec3 uColor;
        
        varying vec2 vUv;
        varying vec3 vNormal;
        varying vec3 vWorldPos;
        
        void main() {
            // Fresnel
            vec3 viewDir = normalize(cameraPosition - vWorldPos);
            float fresnel = pow(1.0 - abs(dot(vNormal, viewDir)), 3.0);
            
            // Diffuse lighting
            vec3 lightDir = normalize(vec3(1.0, 2.0, 1.0));
            float diffuse = max(0.0, dot(vNormal, lightDir));
            
            vec3 color = uColor * (0.3 + diffuse * 0.7);
            color = mix(color, vec3(0.5, 0.8, 1.0), fresnel * 0.5);
            
            gl_FragColor = vec4(color, 1.0);
        }
    `
});

// === 2. RawShaderMaterial ===
// ต้องประกาศ EVERYTHING เอง
const rawShaderMaterial = new THREE.RawShaderMaterial({
    uniforms: {
        // ต้องประกาศ matrices ที่ใช้ด้วย!
        projectionMatrix: { value: camera.projectionMatrix },
        modelViewMatrix:  { value: new THREE.Matrix4() },
        modelMatrix:      { value: new THREE.Matrix4() },
        normalMatrix:     { value: new THREE.Matrix3() },
        cameraPosition:   { value: camera.position },
        uTime:            { value: 0 },
        uColor:           { value: new THREE.Color(0xff4444) }
    },
    vertexShader: `
        precision mediump float;
        
        // ต้องประกาศ uniforms เอง
        uniform mat4 projectionMatrix;
        uniform mat4 modelViewMatrix;
        uniform mat4 modelMatrix;
        uniform mat3 normalMatrix;
        uniform float uTime;
        
        // ต้องประกาศ attributes เอง
        attribute vec3 position;
        attribute vec3 normal;
        attribute vec2 uv;
        
        // Varyings
        varying vec2 vUv;
        varying vec3 vNormal;
        varying vec3 vWorldPos;
        
        void main() {
            vUv = uv;
            vNormal = normalize(normalMatrix * normal);
            
            vec3 pos = position;
            pos.y += cos(pos.x * 3.0 + uTime * 2.5) * 0.15;
            pos.y += cos(pos.z * 2.5 + uTime * 1.8) * 0.1;
            
            vec4 worldPos = modelMatrix * vec4(pos, 1.0);
            vWorldPos = worldPos.xyz;
            
            gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
        }
    `,
    fragmentShader: `
        precision mediump float;
        
        // ต้องประกาศ uniforms เอง
        uniform vec3 cameraPosition;
        uniform vec3 uColor;
        
        // Varyings
        varying vec2 vUv;
        varying vec3 vNormal;
        varying vec3 vWorldPos;
        
        void main() {
            vec3 viewDir = normalize(cameraPosition - vWorldPos);
            float fresnel = pow(1.0 - abs(dot(vNormal, viewDir)), 3.0);
            
            vec3 lightDir = normalize(vec3(1.0, 2.0, 1.0));
            float diffuse = max(0.0, dot(vNormal, lightDir));
            
            vec3 color = uColor * (0.3 + diffuse * 0.7);
            color = mix(color, vec3(1.0, 0.8, 0.5), fresnel * 0.5);
            
            gl_FragColor = vec4(color, 1.0);
        }
    `
});

// === 3. MeshStandardMaterial + onBeforeCompile ===
const standardMaterial = new THREE.MeshStandardMaterial({
    color: 0x44cc44,
    roughness: 0.3,
    metalness: 0.5
});

standardMaterial.onBeforeCompile = (shader) => {
    shader.uniforms.uTime = { value: 0 };
    
    shader.vertexShader = `
        uniform float uTime;
    ` + shader.vertexShader;
    
    shader.vertexShader = shader.vertexShader.replace(
        '#include <begin_vertex>',
        `
        #include <begin_vertex>
        transformed.y += sin(position.x * 3.0 + uTime * 3.0) * 0.15;
        transformed.y += sin(position.z * 2.0 + uTime * 2.0) * 0.1;
        `
    );
    
    standardMaterial.userData.shader = shader;
};

// สร้าง 3 sphere
const geometry = new THREE.SphereGeometry(0.8, 64, 64);

const meshSM  = new THREE.Mesh(geometry, shaderMaterial);
const meshRSM = new THREE.Mesh(geometry, rawShaderMaterial);
const meshMS  = new THREE.Mesh(geometry, standardMaterial);

meshSM.position.x  = -2.5;
meshRSM.position.x = 0;
meshMS.position.x  = 2.5;

scene.add(meshSM, meshRSM, meshMS);

// Labels using canvas texture
function makeLabel(text, color) {
    const canvas = document.createElement('canvas');
    canvas.width = 512;
    canvas.height = 64;
    const ctx = canvas.getContext('2d');
    ctx.fillStyle = color;
    ctx.fillRect(0, 0, 512, 64);
    ctx.fillStyle = 'white';
    ctx.font = '24px monospace';
    ctx.textAlign = 'center';
    ctx.fillText(text, 256, 40);
    
    const texture = new THREE.CanvasTexture(canvas);
    const mat = new THREE.MeshBasicMaterial({ map: texture, transparent: true });
    const geo = new THREE.PlaneGeometry(2, 0.25);
    return new THREE.Mesh(geo, mat);
}

const label1 = makeLabel('ShaderMaterial', '#2a5aaa');
const label2 = makeLabel('RawShaderMaterial', '#aa2a2a');
const label3 = makeLabel('MeshStandard + hook', '#2aaa2a');

label1.position.set(-2.5, -1.2, 0);
label2.position.set(0, -1.2, 0);
label3.position.set(2.5, -1.2, 0);

scene.add(label1, label2, label3);

// FPS counter
let lastTime = performance.now();
let frames = 0;

const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    
    const elapsed = clock.getElapsedTime();
    
    // Update ShaderMaterial uniform
    shaderMaterial.uniforms.uTime.value = elapsed;
    
    // Update RawShaderMaterial uniforms
    rawShaderMaterial.uniforms.uTime.value = elapsed;
    rawShaderMaterial.uniforms.projectionMatrix.value.copy(camera.projectionMatrix);
    meshRSM.updateMatrixWorld();
    rawShaderMaterial.uniforms.modelViewMatrix.value.multiplyMatrices(
        camera.matrixWorldInverse, meshRSM.matrixWorld
    );
    rawShaderMaterial.uniforms.modelMatrix.value.copy(meshRSM.matrixWorld);
    rawShaderMaterial.uniforms.normalMatrix.value.getNormalMatrix(meshRSM.matrixWorld);
    rawShaderMaterial.uniforms.cameraPosition.value.copy(camera.position);
    
    // Update MeshStandard shader
    if (standardMaterial.userData.shader) {
        standardMaterial.userData.shader.uniforms.uTime.value = elapsed;
    }
    
    // FPS
    frames++;
    const now = performance.now();
    if (now - lastTime > 500) {
        document.getElementById('fps').textContent = `FPS: ${Math.round(frames * 1000 / (now - lastTime))}`;
        frames = 0;
        lastTime = now;
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

## Shader Includes Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Custom Shader Includes</title>
    <style>
        * { margin: 0; padding: 0; }
        canvas { display: block; }
        pre {
            position: fixed;
            top: 10px;
            right: 10px;
            color: #aaa;
            font-size: 11px;
            background: rgba(0,0,0,0.7);
            padding: 10px;
            border-radius: 5px;
            max-width: 300px;
        }
    </style>
</head>
<body>
<pre>Custom Shader Includes Demo
THREE.ShaderChunk['myLighting'] = ...
THREE.ShaderChunk['myNoise'] = ...
ใช้ #include ใน shader</pre>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';
import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.157.0/examples/jsm/controls/OrbitControls.js';

// === เพิ่ม Custom Shader Chunks ===
THREE.ShaderChunk['myLighting'] = `
    // Custom Blinn-Phong lighting
    vec3 blinnPhong(vec3 normal, vec3 viewDir, vec3 lightDir, vec3 lightColor, 
                    vec3 diffuseColor, vec3 specularColor, float shininess) {
        float diff = max(0.0, dot(normal, lightDir));
        vec3 halfDir = normalize(lightDir + viewDir);
        float spec = pow(max(0.0, dot(normal, halfDir)), shininess);
        
        return diffuseColor * lightColor * diff + specularColor * lightColor * spec;
    }
    
    // Fresnel reflectance (Schlick approximation)
    float fresnelSchlick(float cosTheta, float F0) {
        return F0 + (1.0 - F0) * pow(1.0 - cosTheta, 5.0);
    }
    
    // Rim lighting
    float rimLight(vec3 normal, vec3 viewDir, float power) {
        return pow(1.0 - abs(dot(normal, viewDir)), power);
    }
`;

THREE.ShaderChunk['myNoise'] = `
    // Value noise
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
    
    float fbm(vec2 p, int octaves) {
        float value = 0.0;
        float amplitude = 0.5;
        float frequency = 1.0;
        for (int i = 0; i < 8; i++) {
            if (i >= octaves) break;
            value += amplitude * valueNoise(p * frequency);
            amplitude *= 0.5;
            frequency *= 2.0;
        }
        return value;
    }
`;

THREE.ShaderChunk['myColorUtils'] = `
    const float PI = 3.14159265359;
    const float TWO_PI = 6.28318530718;
    
    // Cosine color palette
    vec3 palette(float t, vec3 a, vec3 b, vec3 c, vec3 d) {
        return a + b * cos(TWO_PI * (c * t + d));
    }
    
    // Standard palette presets
    vec3 coolWarmPalette(float t) {
        return palette(t, 
            vec3(0.5, 0.5, 0.5), 
            vec3(0.5, 0.5, 0.5),
            vec3(1.0, 1.0, 1.0),
            vec3(0.0, 0.33, 0.67)
        );
    }
    
    vec3 firePalette(float t) {
        return palette(t,
            vec3(0.5, 0.5, 0.0),
            vec3(0.5, 0.5, 0.1),
            vec3(1.0, 0.8, 0.3),
            vec3(0.0, 0.2, 0.5)
        );
    }
    
    // Tone mapping
    vec3 reinhardToneMap(vec3 color, float exposure) {
        color *= exposure;
        return color / (color + vec3(1.0));
    }
`;

// Scene setup
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x050515);
const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 100);
camera.position.set(0, 1, 4);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

// Shader using custom chunks
const vertexShader = `
    #include <myNoise>
    
    uniform float uTime;
    
    varying vec2 vUv;
    varying vec3 vNormal;
    varying vec3 vWorldPos;
    varying float vNoiseSample;
    
    void main() {
        vUv = uv;
        
        // Sample noise for displacement
        float noiseVal = fbm(position.xz * 2.0 + uTime * 0.3, 4);
        vNoiseSample = noiseVal;
        
        vec3 pos = position;
        pos.y += noiseVal * 0.3;
        
        vNormal = normalize(normalMatrix * normal);
        
        vec4 worldPos = modelMatrix * vec4(pos, 1.0);
        vWorldPos = worldPos.xyz;
        
        gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
    }
`;

const fragmentShader = `
    precision mediump float;
    
    // Include ทุก chunk ที่เพิ่มไว้
    #include <myLighting>
    #include <myColorUtils>
    #include <myNoise>
    
    uniform float uTime;
    
    varying vec2 vUv;
    varying vec3 vNormal;
    varying vec3 vWorldPos;
    varying float vNoiseSample;
    
    void main() {
        // ใช้ custom lighting function
        vec3 viewDir  = normalize(cameraPosition - vWorldPos);
        vec3 lightDir = normalize(vec3(2.0, 3.0, 2.0));
        
        // ใช้ custom color palette
        vec3 baseColor = firePalette(vNoiseSample + uTime * 0.1);
        
        // ใช้ custom lighting
        vec3 lit = blinnPhong(
            vNormal, viewDir, lightDir,
            vec3(1.0), baseColor, vec3(1.0), 64.0
        );
        
        // เพิ่ม rim lighting
        float rim = rimLight(vNormal, viewDir, 3.0);
        lit += rim * vec3(0.3, 0.5, 1.0) * 0.5;
        
        // Fresnel
        float cosAngle = abs(dot(vNormal, viewDir));
        float fres = fresnelSchlick(cosAngle, 0.1);
        lit = mix(lit, vec3(0.5, 0.7, 1.0), fres * 0.3);
        
        // Tone mapping
        vec3 finalColor = reinhardToneMap(lit, 1.5);
        
        gl_FragColor = vec4(finalColor, 1.0);
    }
`;

const geometry = new THREE.SphereGeometry(1.2, 128, 128);
const material = new THREE.ShaderMaterial({
    vertexShader,
    fragmentShader,
    uniforms: {
        uTime: { value: 0 }
    }
});

scene.add(new THREE.Mesh(geometry, material));

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
});
</script>
</body>
</html>
```

---

## เมื่อไหรควรใช้อะไร?

### ใช้ ShaderMaterial เมื่อ:
- ต้องการ setup ง่ายและเร็ว
- ต้องการใช้ built-in Three.js features (lights, fog, shadows)
- สร้าง prototype หรือเรียนรู้

### ใช้ RawShaderMaterial เมื่อ:
- ต้องการ GLSL ES 3.0 features
- ต้องการ control เต็มที่บน shader code
- ต้องการ optimize performance สูงสุด
- เขียน shader สำหรับ non-standard use cases

### ใช้ onBeforeCompile เมื่อ:
- ต้องการ modify built-in material (MeshStandard, MeshPhong, etc.)
- ต้องการเพิ่ม custom behavior โดยยังคง features เดิม (shadows, lights, etc.)
- ต้องการ compatibility กับ Three.js materials system

---

## GLSL ES 3.0 ใน RawShaderMaterial

```javascript
const rawMaterial = new THREE.RawShaderMaterial({
    glslVersion: THREE.GLSL3, // เปิดใช้ GLSL ES 3.0
    vertexShader: `
        #version 300 es
        
        // GLSL 3.0 features
        in vec3 position;    // แทน attribute
        in vec2 uv;          // แทน attribute
        
        out vec2 vUv;        // แทน varying
        
        uniform mat4 projectionMatrix;
        uniform mat4 modelViewMatrix;
        
        void main() {
            vUv = uv;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
    `,
    fragmentShader: `
        #version 300 es
        precision mediump float;
        
        // GLSL 3.0: ระบุ output variable
        out vec4 outColor;   // แทน gl_FragColor
        
        in vec2 vUv;         // แทน varying
        
        void main() {
            outColor = vec4(vUv, 0.5, 1.0);
        }
    `
});
```

---

## สรุป

```
ShaderMaterial
├── Three.js inject uniforms/attributes อัตโนมัติ
├── รองรับ shader chunks (#include)
├── ง่ายกว่าสำหรับ standard 3D rendering
└── ใช้ GLSL ES 1.0

RawShaderMaterial
├── ต้องประกาศ/ส่ง uniforms และ attributes เองทุกอย่าง
├── ควบคุมได้เต็มที่
├── รองรับ GLSL ES 3.0 (in/out แทน attribute/varying/gl_FragColor)
└── เหมาะสำหรับ advanced use cases

onBeforeCompile
├── Modify built-in materials
├── ยังคงได้ features ทั้งหมดของ built-in material
└── เหมาะสำหรับ add-ons ไม่ใช่ full custom shaders
```

---

## แบบฝึกหัด

### ระดับ 1
1. สร้าง ShaderMaterial ที่ใช้ built-in `cameraPosition` uniform
2. สร้าง RawShaderMaterial ที่ทำงานเหมือน ShaderMaterial แต่ประกาศทุกอย่างเอง
3. เพิ่ม custom shader chunk ที่มี utility functions

### ระดับ 2
4. ใช้ `onBeforeCompile` เพิ่ม wave animation ให้ MeshStandardMaterial
5. สร้าง preprocessor defines system ที่เปิด/ปิด features ได้
6. สร้าง shader library ส่วนตัว (>5 chunks)

### ระดับ 3
7. แปลง ShaderMaterial เป็น RawShaderMaterial ที่ใช้ GLSL ES 3.0
8. สร้าง material system ที่ใช้ #include จาก custom shader library
9. Implement shadow support ใน custom shader โดยใช้ Three.js shadow chunks

### Challenge
10. สร้าง PBR (Physically Based Rendering) shader จาก scratch โดยใช้:
    - Lambertian diffuse
    - Cook-Torrance specular (GGX distribution)
    - Fresnel-Schlick approximation
    - Geometric attenuation
    - Ambient occlusion term
    - Multiple light sources
