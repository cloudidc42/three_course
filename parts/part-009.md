# Part 009: Lighting Part 2 - Advanced Lighting

## สารบัญบทเรียนนี้
- [1. Environment Lighting ด้วย HDR](#1-environment-lighting-ด้วย-hdr)
- [2. PMREMGenerator และ Equirectangular](#2-pmremgenerator-และ-equirectangular)
- [3. Light Baking Concept](#3-light-baking-concept)
- [4. LightProbe](#4-lightprobe)
- [5. Physically Correct Lighting Units](#5-physically-correct-lighting-units)
- [6. Tone Mapping](#6-tone-mapping)
- [7. Color Management](#7-color-management)
- [8. HDR Environment Demo](#8-hdr-environment-demo)
- [9. Tone Mapping Comparison Demo](#9-tone-mapping-comparison-demo)
- [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. Environment Lighting ด้วย HDR

HDR (High Dynamic Range) environment map ให้ real-world ambient lighting ที่สมจริงที่สุด  
แสงมาจากทุกทิศทาง ครอบคลุม sky, horizon, และ ground

```javascript
// ทำไม HDR ดีกว่าแค่ AmbientLight?
// 
// AmbientLight:      แสงสม่ำเสมอทุกทิศ ไม่มีทิศทาง
// HemisphereLight:   แยก sky/ground แต่ยังเรียบ
// HDR Environment:   reflections, specular highlights ที่ถูกต้อง
//                    directional ambient จาก sky/cloud/horizon
//                    สีถูกต้องตาม real-world photography

// HDR File:
// - เก็บ floating point values (ไม่จำกัด 0-255)
// - dynamic range สูง: ดวงอาทิตย์ vs เงา
// - format: .hdr (Radiance RGBE), .exr (OpenEXR)

// แหล่ง HDR ฟรี:
// - https://polyhaven.com/ (CC0, คุณภาพสูง)
// - https://hdrihaven.com/
// - https://ambientcg.com/
```

---

## 2. PMREMGenerator และ Equirectangular

**PMREM** = Pre-filtered Mipmapped Radiance Environment Map  
ใช้สำหรับแปลง HDR equirectangular → TexureCube ที่เหมาะสำหรับ PBR

```javascript
import * as THREE from 'three';
import { RGBELoader } from 'three/addons/loaders/RGBELoader.js';
import { EXRLoader } from 'three/addons/loaders/EXRLoader.js';

// ========== Setup ==========

const renderer = new THREE.WebGLRenderer({ antialias: true });
const pmremGenerator = new THREE.PMREMGenerator(renderer);
pmremGenerator.compileEquirectangularShader();

// ========== Load .hdr ==========

const rgbeLoader = new RGBELoader();

function loadHDR(path) {
    return new Promise((resolve, reject) => {
        rgbeLoader.load(
            path,
            (texture) => {
                // แปลง equirectangular HDR → PMREM cube map
                const envMap = pmremGenerator.fromEquirectangular(texture).texture;
                texture.dispose();
                pmremGenerator.dispose();
                resolve(envMap);
            },
            (progress) => {
                console.log(`Loading: ${(progress.loaded / progress.total * 100).toFixed(0)}%`);
            },
            reject
        );
    });
}

// ใช้งาน
const envMap = await loadHDR('/textures/hdr/studio_small_09_1k.hdr');
scene.environment = envMap;  // ambient lighting สำหรับ PBR materials
scene.background = envMap;   // แสดงเป็น background

// ========== Load .exr ==========

const exrLoader = new EXRLoader();

function loadEXR(path) {
    return new Promise((resolve, reject) => {
        exrLoader.load(path, (texture) => {
            const envMap = pmremGenerator.fromEquirectangular(texture).texture;
            texture.dispose();
            resolve(envMap);
        }, undefined, reject);
    });
}

// ========== Environment ≠ Background ==========

// แยกกันได้:
const envMap2 = await loadHDR('/hdr/studio.hdr');
scene.environment = envMap2;                     // ใช้เป็นแสง
scene.background = new THREE.Color('#1a1a2e');   // พื้นหลังสีทึบ

// หรือ blur background แต่ยังใช้เป็น environment
scene.backgroundBlurriness = 0.5;  // 0=คม, 1=เบลอมาก
scene.backgroundIntensity = 0.8;   // ความสว่าง background

// ========== envMapIntensity ต่อ material ==========

const mat = new THREE.MeshStandardMaterial({
    metalness: 1.0,
    roughness: 0.0,
    envMapIntensity: 2.0,  // เพิ่ม reflection intensity
});

// Override สำหรับทุก material ใน scene
scene.traverse((node) => {
    if (node.isMesh && node.material) {
        node.material.envMapIntensity = 1.5;
    }
});

// ========== PMREM จาก CubeCamera (Dynamic Reflection) ==========

// สำหรับ reflection ที่ update real-time (เช่น car paint บน track)
const cubeRenderTarget = new THREE.WebGLCubeRenderTarget(256, {
    type: THREE.HalfFloatType,
});
const cubeCamera = new THREE.CubeCamera(0.1, 100, cubeRenderTarget);
scene.add(cubeCamera);

function updateCubeCamera(time) {
    // ซ่อน reflective object ระหว่าง capture
    reflectiveMesh.visible = false;
    cubeCamera.update(renderer, scene);
    reflectiveMesh.visible = true;
    
    // ใช้ render target เป็น env map
    reflectiveMat.envMap = cubeRenderTarget.texture;
}

// ⚠️ cubeCamera.update() = render 6 ครั้ง = ช้ามาก
// ใช้ทุกๆ 2-3 frames หรือเฉพาะเมื่อ scene เปลี่ยน
```

---

## 3. Light Baking Concept

Light baking = คำนวณ lighting ล่วงหน้า แล้วเก็บในรูปแบบ texture (lightmap)

```javascript
// Baking คืออะไร?
// แทนที่ real-time lighting computation
// เราคำนวณทุกอย่างในโปรแกรมอื่น (Blender)
// แล้วเก็บผลไว้ใน texture
// Three.js แค่ sample texture นั้น ← เร็วมาก

// Workflow:
// 1. Model ใน Blender
// 2. UV Unwrap (unique UVs, ไม่ซ้ำกัน)
// 3. Bake: Combined/Diffuse/Shadow
// 4. Export texture เป็น PNG/JPG
// 5. ใน Three.js: ใช้เป็น lightMap

// ========== ใช้ Lightmap ใน Three.js ==========

const lightMap = textureLoader.load('/textures/baked_lighting.jpg');
// lightmap ใช้ UV2!
geometry.setAttribute('uv1', geometry.attributes.uv.clone());

const material = new THREE.MeshStandardMaterial({
    map: colorMap,           // สี
    lightMap: lightMap,      // baked lighting
    lightMapIntensity: 1.0,
    
    // ปิด real-time lighting เพราะ bake แล้ว
    // (ใช้ MeshBasicMaterial ถ้าไม่ต้องการ any real-time light)
});

// หรือใช้ MeshBasicMaterial สำหรับ fully baked
const bakedMat = new THREE.MeshBasicMaterial({
    map: bakedColorAndLight,  // baked all-in-one
});

// ========== เมื่อไหร่ควร bake ==========

// ✅ ควร bake:
// - Static scene (ไม่ขยับ)
// - Complex lighting (หลาย lights, GI)
// - Mobile / low-end devices
// - Large scenes (game levels)

// ❌ ไม่ควร bake:
// - Moving objects
// - Dynamic time of day
// - Interactive objects
// - Characters

// ========== Lightmap + Real-time ==========

// Hybrid approach: bake สำหรับ static, real-time สำหรับ dynamic
const staticMat = new THREE.MeshStandardMaterial({
    map: colorMap,
    lightMap: bakedLightMap,  // static ambient + indirect
    lightMapIntensity: 0.8,
    // เพิ่ม real-time directional สำหรับ direct sun
    // scene.add(dirLight) ← ยังใช้ได้
});
```

---

## 4. LightProbe

LightProbe เก็บ ambient lighting ในรูปแบบ Spherical Harmonics (SH)  
เบากว่า environment map มาก

```javascript
import { LightProbeGenerator } from 'three/addons/lights/LightProbeGenerator.js';

// ========== สร้าง LightProbe จาก CubeTexture ==========

const cubeTextureLoader = new THREE.CubeTextureLoader();
const cubeTexture = cubeTextureLoader.load([
    '/cube/px.jpg', '/cube/nx.jpg',
    '/cube/py.jpg', '/cube/ny.jpg',
    '/cube/pz.jpg', '/cube/nz.jpg',
]);

const lightProbe = LightProbeGenerator.fromCubeTexture(cubeTexture);
scene.add(lightProbe);

// ========== สร้าง LightProbe จาก HDR ==========

// LightProbeGenerator ยังไม่รองรับ HDR โดยตรงใน r165
// แต่สามารถทำได้ผ่าน CubeRenderTarget

// 1. Render scene ลง cube render target
const cubeRT = new THREE.WebGLCubeRenderTarget(256);
const cubeCamera2 = new THREE.CubeCamera(0.1, 100, cubeRT);
scene.add(cubeCamera2);
cubeCamera2.update(renderer, scene);

// 2. สร้าง probe จาก cube texture
const probe = LightProbeGenerator.fromCubeRenderTarget(renderer, cubeRT);
scene.add(probe);

// ========== Properties ==========

lightProbe.intensity = 1.0;  // ปรับความสว่าง

// LightProbe ใช้ SH (Spherical Harmonics)
// - ไม่มี hard shadows
// - ให้ ambient ที่มีทิศทางมากกว่า AmbientLight
// - เบากว่า full environment map มาก
// - เหมาะสำหรับ mobile / optimization
```

---

## 5. Physically Correct Lighting Units

Three.js r155+ ใช้ SI units สำหรับ lighting

```javascript
// ========== Light Intensity Units ==========

// PointLight / SpotLight:
//   intensity = luminous intensity (candela, cd)
//   หรือ luminous flux ถ้าใช้ "power" mode
//   
// DirectionalLight:
//   intensity = illuminance (lux, lx)
//
// AmbientLight / HemisphereLight:
//   intensity = irradiance (W/m²)

// ค่าอ้างอิง Real World:
// ดวงอาทิตย์กลางวัน:    ~100,000 lux
// วัน흐ม:                ~1,000-10,000 lux
// ออฟฟิศ:                ~300-500 lux
// ห้องนั่งเล่น:          ~50-150 lux
// กลางคืน (full moon):   ~0.1-0.3 lux
// หลอดไฟ 60W:           ~800 lumens
// หลอดไฟ LED 10W:       ~800-1000 lumens
// เทียน:                 ~12 lumens

// ========== ตั้งค่า Physically Correct ==========

// Three.js r155+ เปิด physically correct โดย default
// ก่อน r155: useLegacyLights = false ต้องตั้งเอง

renderer.useLegacyLights = false; // ใน older versions

// ========== ตัวอย่าง Physically Correct Scene ==========

// Outdoor daylight
const sun = new THREE.DirectionalLight(0xfff4e0, 100000);  // 100k lux
sun.position.set(100, 200, 100);
scene.add(sun);

// Sky ambient
const sky = new THREE.AmbientLight(0x87ceeb, 20000);  // ท้องฟ้า

// ⚠️ Values สูงมากต้องใช้ tone mapping
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 0.00001;  // ลด exposure ลงมาก

// ========== Practical Values (non-physical) ==========

// ใช้ค่าที่ต่ำกว่า และปรับ tone mapping exposure:
const practicalSun = new THREE.DirectionalLight(0xfff4e0, 3.0);
renderer.toneMappingExposure = 1.0;

// อ่านเพิ่มเติม: https://threejs.org/docs/#api/en/renderers/WebGLRenderer.useLegacyLights
```

---

## 6. Tone Mapping

Tone mapping แปลง HDR values ลงมาเป็น LDR (0-1) สำหรับ screen

```javascript
// ========== ชนิด Tone Mapping ใน Three.js ==========

// 1. THREE.NoToneMapping (default ก่อน r152)
//    ไม่ทำ tone mapping, clamp ที่ 1.0
//    สี > 1.0 จะ clip

// 2. THREE.LinearToneMapping
//    เพิ่ม/ลด exposure เท่านั้น, ไม่มี curve
//    exposure = toneMappingExposure

// 3. THREE.ReinhardToneMapping
//    สูตร: x / (1 + x)
//    นุ่มนวล, preserves highlight detail
//    สีจะดูอ่อนกว่า

// 4. THREE.CineonToneMapping
//    จำลอง film stock (Kodak Cineon)
//    contrast สูง, ใช้ใน film industry

// 5. THREE.ACESFilmicToneMapping ← ที่นิยมที่สุด
//    Academy Color Encoding System
//    ใช้ใน feature films และ games
//    contrast และ saturation ดี

// 6. THREE.AgXToneMapping (r157+)
//    AgX tone mapping ของ Blender
//    handles very bright colors ดีกว่า ACES

// 7. THREE.NeutralToneMapping (r165+)
//    neutral / balanced
//    preserves hue ได้ดี

// ========== ตั้งค่า ==========

renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.0;  // ปรับความสว่างรวม

// toneMappingExposure:
// 1.0 = normal
// 0.5 = ลดความสว่าง 50%
// 2.0 = เพิ่มความสว่าง 2x

// ========== เปลี่ยน Tone Mapping หลัง init ==========

// เมื่อเปลี่ยน, materials ต้อง update
renderer.toneMapping = THREE.ReinhardToneMapping;
scene.traverse((node) => {
    if (node.isMesh) {
        node.material.needsUpdate = true;
    }
});

// ========== Exposure Compensation ==========

// ถ้าใช้ environment map สว่างหรือมืดเกินไป:
renderer.toneMappingExposure = 0.7;  // dark/moody
// หรือ
renderer.toneMappingExposure = 1.5;  // bright/airy

// ========== เปรียบเทียบ Tone Mappers ==========
//
// NoToneMapping:       ดี สำหรับ UI / unlit scenes
// LinearToneMapping:   เหมาะ debug และ reference
// ReinhardToneMapping: soft, cinematic
// CineonToneMapping:   warm, filmic
// ACESFilmicToneMapping: standard for games/film ← ใช้บ่อยที่สุด
// AgXToneMapping:      modern alternative to ACES
// NeutralToneMapping:  balanced, hue-preserving
```

---

## 7. Color Management

```javascript
// ========== Color Space ใน Three.js ==========

// Three.js r152+ ใช้ Linear workflow
// - Texture: load ใน sRGB, Three.js แปลงเป็น linear อัตโนมัติ
// - Lighting: คำนวณใน linear space
// - Output: แปลงกลับเป็น sRGB สำหรับ screen

// ตั้งค่า output color space
renderer.outputColorSpace = THREE.SRGBColorSpace;  // default

// ========== Texture Color Space ==========

// Color textures (albedo, emissive):
colorTexture.colorSpace = THREE.SRGBColorSpace;

// Data textures (normal, roughness, metalness, AO):
normalTexture.colorSpace = THREE.NoColorSpace;  // linear

// HDR textures (loaded via RGBELoader/EXRLoader):
// Handled automatically, no need to set colorSpace

// ========== Color Input ==========

// ค่าสีที่กำหนดใน Three.js ควรเป็น sRGB:
material.color.set('#ff8800');        // hex string - sRGB
material.color.setHex(0xff8800);      // hex number - sRGB
material.color.setRGB(1, 0.53, 0);   // ⚠️ linear values!
material.color.setHSL(0.09, 1, 0.5); // sRGB

// Three.js Color class จัดการ conversion ให้

// ========== Color Space Check ==========

// Debug: ตรวจสอบว่า texture ใช้ color space ถูกต้อง
function checkTextureColorSpace(texture, expectedSpace) {
    if (texture.colorSpace !== expectedSpace) {
        console.warn(
            `Texture color space mismatch: ` +
            `expected ${expectedSpace}, got ${texture.colorSpace}`
        );
    }
}

// ========== Linear Workflow Summary ==========
//
// Input (textures/colors)  → sRGB
// Three.js internal        → linear
// Lighting calculation     → linear
// Output (screen)          → sRGB (via outputColorSpace)
//
// Three.js r152+ ทำทุกอย่างให้อัตโนมัติ
// ถ้าตั้ง colorSpace ของ texture ถูกต้อง
```

---

## 8. HDR Environment Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Part 009: HDR Environment Lighting</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        html, body { width: 100%; height: 100%; overflow: hidden; background: #000; }
        #loading {
            position: fixed; top: 50%; left: 50%;
            transform: translate(-50%, -50%);
            color: white; font-family: monospace; font-size: 16px;
            text-align: center;
        }
        #loading-bar {
            width: 200px; height: 4px;
            background: #333; margin-top: 10px; border-radius: 2px;
        }
        #loading-fill {
            height: 100%; background: #6c63ff; border-radius: 2px;
            width: 0%; transition: width 0.1s;
        }
    </style>
</head>
<body>
    <div id="loading">
        <div>กำลังโหลด...</div>
        <div id="loading-bar"><div id="loading-fill"></div></div>
    </div>
    
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
        
        // ====== RENDERER ======
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(innerWidth, innerHeight);
        renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
        renderer.toneMapping = THREE.ACESFilmicToneMapping;
        renderer.toneMappingExposure = 1.0;
        renderer.outputColorSpace = THREE.SRGBColorSpace;
        document.body.appendChild(renderer.domElement);
        
        // ====== SCENE ======
        const scene = new THREE.Scene();
        
        // ====== CAMERA ======
        const camera = new THREE.PerspectiveCamera(60, innerWidth/innerHeight, 0.1, 100);
        camera.position.set(0, 2, 8);
        
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.target.set(0, 0.5, 0);
        controls.update();
        
        // ====== PMREM GENERATOR ======
        const pmremGenerator = new THREE.PMREMGenerator(renderer);
        pmremGenerator.compileEquirectangularShader();
        
        // ====== สร้าง Procedural HDR Environment (ไม่ต้อง file) ======
        
        function createProceduralHDR(type) {
            const size = 512;
            const data = new Float32Array(size * size * 4);
            
            if (type === 'studio') {
                // Soft studio lighting
                for (let y = 0; y < size; y++) {
                    for (let x = 0; x < size; x++) {
                        const u = x / size;
                        const v = y / size;
                        const theta = u * Math.PI * 2;  // azimuth
                        const phi = v * Math.PI;          // elevation
                        
                        // Sky: light grey
                        let r = 0.6, g = 0.6, b = 0.7;
                        
                        // Top hemisphere (top light - softbox)
                        if (phi < Math.PI * 0.3) {
                            const blend = 1 - phi / (Math.PI * 0.3);
                            r = r + blend * 1.5;
                            g = g + blend * 1.5;
                            b = b + blend * 2.0;
                        }
                        
                        // Key light (bright spot)
                        const keyTheta = Math.PI * 0.2; // 36 degrees
                        const keyPhi = Math.PI * 0.3;    // 54 degrees
                        const angDist = Math.sqrt(
                            Math.pow(theta - keyTheta, 2) + 
                            Math.pow(phi - keyPhi, 2)
                        );
                        if (angDist < 0.3) {
                            const spot = Math.pow(1 - angDist / 0.3, 3);
                            r += spot * 5;
                            g += spot * 5;
                            b += spot * 5.5;
                        }
                        
                        // Ground: dark
                        if (phi > Math.PI * 0.55) {
                            const blend = (phi - Math.PI * 0.55) / (Math.PI * 0.45);
                            r = r * (1 - blend) + 0.1 * blend;
                            g = g * (1 - blend) + 0.1 * blend;
                            b = b * (1 - blend) + 0.15 * blend;
                        }
                        
                        const i = (y * size + x) * 4;
                        data[i+0] = r;
                        data[i+1] = g;
                        data[i+2] = b;
                        data[i+3] = 1;
                    }
                }
            } else if (type === 'sunset') {
                for (let y = 0; y < size; y++) {
                    for (let x = 0; x < size; x++) {
                        const u = x / size;
                        const v = y / size;
                        const phi = v * Math.PI;
                        
                        let r, g, b;
                        
                        if (phi < Math.PI * 0.5) {
                            // Upper sky: deep blue → orange
                            const t = phi / (Math.PI * 0.5);
                            r = 0.1 + t * 1.5;
                            g = 0.05 + t * 0.5;
                            b = 0.5 - t * 0.4;
                        } else {
                            // Horizon → ground
                            const t = (phi - Math.PI * 0.5) / (Math.PI * 0.5);
                            r = 1.5 - t * 1.3;
                            g = 0.4 - t * 0.35;
                            b = 0.1 - t * 0.05;
                        }
                        
                        // Sun disk
                        const theta = u * Math.PI * 2;
                        const sunTheta = Math.PI * 1.7;
                        const sunPhi = Math.PI * 0.48;
                        const sunDist = Math.sqrt(
                            Math.pow(theta - sunTheta, 2) * 0.5 + 
                            Math.pow(phi - sunPhi, 2) * 0.5
                        );
                        if (sunDist < 0.08) {
                            const sun = Math.pow(1 - sunDist / 0.08, 2);
                            r += sun * 8;
                            g += sun * 4;
                            b += sun * 1;
                        }
                        
                        const i = (y * size + x) * 4;
                        data[i+0] = Math.max(0, r);
                        data[i+1] = Math.max(0, g);
                        data[i+2] = Math.max(0, b);
                        data[i+3] = 1;
                    }
                }
            } else if (type === 'night') {
                for (let y = 0; y < size; y++) {
                    for (let x = 0; x < size; x++) {
                        const u = x / size;
                        const v = y / size;
                        const phi = v * Math.PI;
                        
                        // Dark blue sky
                        const skyBlend = Math.max(0, 1 - phi / (Math.PI * 0.5));
                        const r = 0.01 + skyBlend * 0.02;
                        const g = 0.01 + skyBlend * 0.03;
                        const b = 0.05 + skyBlend * 0.1;
                        
                        const i = (y * size + x) * 4;
                        data[i+0] = r;
                        data[i+1] = g;
                        data[i+2] = b;
                        data[i+3] = 1;
                    }
                }
                // Add stars (random bright points)
                for (let s = 0; s < 300; s++) {
                    const sx = Math.floor(Math.random() * size);
                    const sy = Math.floor(Math.random() * size * 0.5);
                    const brightness = 0.5 + Math.random() * 2;
                    const si = (sy * size + sx) * 4;
                    data[si+0] += brightness;
                    data[si+1] += brightness;
                    data[si+2] += brightness * 1.2;
                }
            }
            
            const texture = new THREE.DataTexture(
                data, size, size,
                THREE.RGBAFormat,
                THREE.FloatType
            );
            texture.mapping = THREE.EquirectangularReflectionMapping;
            texture.needsUpdate = true;
            return texture;
        }
        
        // ====== OBJECTS ======
        
        // Ground
        const groundMat = new THREE.MeshStandardMaterial({
            color: '#404055',
            roughness: 0.3,
            metalness: 0.1,
        });
        const ground = new THREE.Mesh(
            new THREE.PlaneGeometry(20, 20),
            groundMat
        );
        ground.rotation.x = -Math.PI / 2;
        ground.receiveShadow = true;
        scene.add(ground);
        
        // Sphere grid (different materials)
        const spheres = [];
        const materials = [
            { metalness: 0, roughness: 0.0, color: '#ffffff', label: 'Dielectric Smooth' },
            { metalness: 0, roughness: 0.5, color: '#ffffff', label: 'Dielectric Mid' },
            { metalness: 0, roughness: 1.0, color: '#ffffff', label: 'Dielectric Rough' },
            { metalness: 1, roughness: 0.0, color: '#c0c0c0', label: 'Metal Smooth' },
            { metalness: 1, roughness: 0.3, color: '#c87533', label: 'Copper' },
            { metalness: 1, roughness: 0.8, color: '#b8860b', label: 'Gold Rough' },
        ];
        
        materials.forEach((props, i) => {
            const mat = new THREE.MeshStandardMaterial({
                color: props.color,
                metalness: props.metalness,
                roughness: props.roughness,
            });
            const sphere = new THREE.Mesh(
                new THREE.SphereGeometry(0.7, 64, 32),
                mat
            );
            const x = (i % 3) * 2.5 - 2.5;
            const z = Math.floor(i / 3) * 2.5 - 1.25;
            sphere.position.set(x, 0.8, z);
            sphere.castShadow = true;
            sphere.receiveShadow = true;
            scene.add(sphere);
            spheres.push(sphere);
        });
        
        // ====== GUI ======
        const gui = new GUI({ title: 'Environment Lighting' });
        
        const params = {
            envType: 'studio',
            envIntensity: 1.0,
            exposure: 1.0,
            toneMapping: 'ACESFilmic',
            backgroundBlur: 0,
            showBackground: true,
        };
        
        const toneMappingMap = {
            'None': THREE.NoToneMapping,
            'Linear': THREE.LinearToneMapping,
            'Reinhard': THREE.ReinhardToneMapping,
            'Cineon': THREE.CineonToneMapping,
            'ACESFilmic': THREE.ACESFilmicToneMapping,
        };
        
        function updateEnv() {
            const envTex = createProceduralHDR(params.envType);
            const envMap = pmremGenerator.fromEquirectangular(envTex).texture;
            envTex.dispose();
            
            scene.environment = envMap;
            if (params.showBackground) {
                scene.background = envMap;
            } else {
                scene.background = new THREE.Color('#0d1117');
            }
            scene.backgroundBlurriness = params.backgroundBlur;
            
            scene.traverse(node => {
                if (node.isMesh && node.material) {
                    node.material.envMapIntensity = params.envIntensity;
                    node.material.needsUpdate = true;
                }
            });
        }
        
        gui.add(params, 'envType', ['studio', 'sunset', 'night']).onChange(updateEnv);
        gui.add(params, 'envIntensity', 0, 5, 0.1).onChange(updateEnv);
        gui.add(params, 'exposure', 0.1, 3, 0.05).onChange(v => {
            renderer.toneMappingExposure = v;
        });
        gui.add(params, 'toneMapping', Object.keys(toneMappingMap)).onChange(v => {
            renderer.toneMapping = toneMappingMap[v];
            scene.traverse(n => {
                if (n.isMesh) n.material.needsUpdate = true;
            });
        });
        gui.add(params, 'backgroundBlur', 0, 1, 0.01).onChange(v => {
            scene.backgroundBlurriness = v;
        });
        gui.add(params, 'showBackground').onChange(updateEnv);
        
        // Apply initial
        updateEnv();
        
        // Remove loading overlay
        document.getElementById('loading').style.display = 'none';
        
        // ====== ANIMATION ======
        const clock = new THREE.Clock();
        
        function animate() {
            requestAnimationFrame(animate);
            const delta = clock.getDelta();
            
            spheres.forEach((s, i) => {
                s.rotation.y += delta * 0.3;
            });
            
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

## 9. Tone Mapping Comparison Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Tone Mapping Comparison</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        html, body { width: 100%; height: 100%; overflow: hidden; background: #000; }
        #labels {
            position: fixed; bottom: 20px; left: 0; right: 0;
            display: flex; justify-content: space-around;
            pointer-events: none;
        }
        .label {
            color: white; font-family: monospace; font-size: 11px;
            background: rgba(0,0,0,0.6); padding: 4px 8px; border-radius: 4px;
            text-align: center;
        }
    </style>
</head>
<body>
    <div id="labels">
        <div class="label">No TM</div>
        <div class="label">Reinhard</div>
        <div class="label">Cineon</div>
        <div class="label">ACES Filmic</div>
    </div>
    
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
        
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(innerWidth, innerHeight);
        renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
        renderer.outputColorSpace = THREE.SRGBColorSpace;
        document.body.appendChild(renderer.domElement);
        
        // ====== สร้าง 4 Scenes ในแนวนอน ==========
        // ใช้ viewport/scissor rendering

        const scenes = [];
        const cameras = [];
        const tonemappers = [
            THREE.NoToneMapping,
            THREE.ReinhardToneMapping,
            THREE.CineonToneMapping,
            THREE.ACESFilmicToneMapping,
        ];
        
        // สร้างของเหมือนกัน 4 ชุด
        tonemappers.forEach((tm, i) => {
            const scene = new THREE.Scene();
            scene.background = new THREE.Color('#0a0a14');
            
            const camera = new THREE.PerspectiveCamera(50, (innerWidth/4) / innerHeight, 0.1, 50);
            camera.position.set(0, 0, 4);
            
            // Lights (bright HDR values)
            const amb = new THREE.AmbientLight(0xffffff, 0.3);
            const dir = new THREE.DirectionalLight(0xffffff, 5); // HDR value
            dir.position.set(2, 4, 3);
            
            const rim = new THREE.DirectionalLight(0x8866ff, 2);
            rim.position.set(-3, -1, -2);
            
            scene.add(amb, dir, rim);
            
            // Objects
            const group = new THREE.Group();
            const mat = new THREE.MeshStandardMaterial({
                color: '#cc8833',
                metalness: 0.8,
                roughness: 0.1,
            });
            
            // Sphere
            const sphere = new THREE.Mesh(new THREE.SphereGeometry(0.8, 32, 16), mat.clone());
            sphere.position.set(-1.2, 0, 0);
            group.add(sphere);
            
            // Torus
            const torus = new THREE.Mesh(
                new THREE.TorusGeometry(0.5, 0.2, 16, 48),
                new THREE.MeshStandardMaterial({ color: '#3388cc', metalness: 0.9, roughness: 0.1 })
            );
            torus.position.set(1.2, 0, 0);
            group.add(torus);
            
            scene.add(group);
            scene.userData.group = group;
            scene.userData.toneMapping = tm;
            
            scenes.push(scene);
            cameras.push(camera);
        });
        
        const clock = new THREE.Clock();
        
        function animate() {
            requestAnimationFrame(animate);
            const delta = clock.getDelta();
            const elapsed = clock.getElapsedTime();
            
            const w4 = Math.floor(innerWidth / 4);
            renderer.setScissorTest(true);
            
            scenes.forEach((scene, i) => {
                const left = i * w4;
                const width = i < 3 ? w4 : innerWidth - i * w4;
                
                renderer.setViewport(left, 0, width, innerHeight);
                renderer.setScissor(left, 0, width, innerHeight);
                renderer.toneMapping = scene.userData.toneMapping;
                
                // Update aspect
                cameras[i].aspect = width / innerHeight;
                cameras[i].updateProjectionMatrix();
                
                // Animate
                const group = scene.userData.group;
                group.rotation.y = elapsed * 0.5;
                
                renderer.render(scene, cameras[i]);
            });
        }
        
        animate();
        
        window.addEventListener('resize', () => {
            renderer.setSize(innerWidth, innerHeight);
        });
    </script>
</body>
</html>
```

---

## สรุป Advanced Lighting

| เทคนิค | เมื่อใช้ | Performance |
|--------|---------|-------------|
| HDR Environment | PBR materials, reflections | ปานกลาง |
| PMREMGenerator | แปลง equirectangular → cube | setup ครั้งเดียว |
| Light Baking | Static scenes | ดีมาก (real-time) |
| LightProbe | Ambient ที่มีทิศทาง เบากว่า env | ดี |
| Physically Correct Units | Realism, photographic accuracy | เหมือนเดิม |
| Tone Mapping | HDR → LDR display | ฟรี (post process) |

---

## แบบฝึกหัด (Exercises)

### Exercise 1: สลับ Environment (ง่าย)
สร้าง scene กับ 3 buttons เพื่อสลับระหว่าง:
- Warm studio
- Outdoor day
- Night
โดยแต่ละ environment เปลี่ยนสีแสงและ HDR ที่สอดคล้องกัน

### Exercise 2: Exposure Control (ง่าย)
สร้าง slider ที่เปลี่ยน `renderer.toneMappingExposure` จาก 0.1 ถึง 5.0
แสดงให้เห็นความแตกต่างของ tone mapping ที่ exposure ต่างกัน

### Exercise 3: Day-Night + Tone Mapping (ปานกลาง)
ต่อยอด Exercise 1 (Part 008):
- เพิ่ม procedural HDR environment ที่ animate ตามเวลา
- Tone mapping เปลี่ยนตาม time of day:
  - กลางวัน: ACES Filmic, exposure 1.0
  - พระอาทิตย์ตก: Cineon, exposure 1.2
  - กลางคืน: Reinhard, exposure 2.0

### Exercise 4: Product Showcase (ยาก)
สร้าง product showcase:
- Procedural studio HDR environment
- Object ที่น่าสนใจ (เช่น แหวนหรือนาฬิกา)
- RectAreaLight 3 ดวง (3-point lighting)
- lil-gui สำหรับปรับ exposure, tone mapping, background blur
- OrbitControls ที่ smooth

---

**ก่อนหน้า:** [Part 008 - Lighting Part 1](part-008.md)  
**ต่อไป:** [Part 010 - Cameras Deep Dive](part-010.md)
