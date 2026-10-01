# Part 82: Node Materials & TSL (Three.js Shading Language)

## บทนำ: ระบบ Material ยุคใหม่

Node Materials หรือ NodeMaterial system เป็นการเปลี่ยนแปลงครั้งใหญ่ของ Three.js ที่เริ่มต้นจาก r150+ ระบบนี้แทนที่การเขียน GLSL โดยตรงด้วย node graph approach ที่คล้ายกับ Shader Editor ใน Blender หรือ Substance Designer

TSL (Three.js Shading Language) เป็น API ระดับ JavaScript ที่ช่วยให้สร้าง shader ได้โดยไม่ต้องเขียน GLSL/WGSL ซึ่งทำให้ material ทำงานได้กับทั้ง WebGL และ WebGPU backend

---

## 1. NodeMaterial Architecture

### โครงสร้างของ Node Graph

```
[Input Nodes]        [Processing Nodes]      [Output Nodes]
  uv()          -->   mul(uv, 5.0)      -->   colorNode
  texture()     -->   mix(a, b, t)      -->   normalNode
  positionLocal -->   sin(x)            -->   roughnessNode
  normalWorld   -->   add(a, b)         -->   metalnessNode
  timerGlobal   -->   tslFn()           -->   emissiveNode
  uniform()     -->   smoothstep()      -->   positionNode
```

### ประเภทของ NodeMaterial

```javascript
// 1. MeshBasicNodeMaterial - ไม่มี lighting
import { MeshBasicNodeMaterial } from 'three/tsl';
const basicMat = new MeshBasicNodeMaterial();
basicMat.colorNode = color(0xff0000);  // สีแดง

// 2. MeshStandardNodeMaterial - PBR material
import { MeshStandardNodeMaterial } from 'three/tsl';
const standardMat = new MeshStandardNodeMaterial();
standardMat.colorNode = color(0x888888);
standardMat.roughnessNode = float(0.5);
standardMat.metalnessNode = float(0.8);

// 3. MeshPhysicalNodeMaterial - ขยายจาก Standard
import { MeshPhysicalNodeMaterial } from 'three/tsl';
const physicalMat = new MeshPhysicalNodeMaterial();
physicalMat.clearcoatNode = float(1.0);
physicalMat.clearcoatRoughnessNode = float(0.1);
physicalMat.iridescenceNode = float(1.0);

// 4. PointsNodeMaterial - สำหรับ Points
import { PointsNodeMaterial } from 'three/tsl';
const pointsMat = new PointsNodeMaterial();
pointsMat.sizeNode = float(5.0);
pointsMat.colorNode = color(0xffffff);

// 5. SpriteNodeMaterial - สำหรับ Sprites
import { SpriteNodeMaterial } from 'three/tsl';
const spriteMat = new SpriteNodeMaterial();
spriteMat.sizeAttenuation = true;

// 6. LineBasicNodeMaterial - สำหรับ Lines
import { LineBasicNodeMaterial } from 'three/tsl';
const lineMat = new LineBasicNodeMaterial();
lineMat.colorNode = color(0x00ff00);
```

---

## 2. Input Nodes: การรับข้อมูลเข้า

### Position และ Normal Nodes

```javascript
import {
  positionLocal,      // position ใน local space (object space)
  positionWorld,      // position ใน world space
  positionView,       // position ใน camera/view space
  positionClip,       // position ใน clip space (NDC)
  normalLocal,        // normal ใน local space
  normalWorld,        // normal ใน world space
  normalView,         // normal ใน view space
  normalFlat,         // flat shading normal
  transformedNormalWorld,  // normal หลัง transform
} from 'three/tsl';

// ตัวอย่าง: แสดงสีตาม world normal
const mat = new MeshBasicNodeMaterial();
// Normal range: -1 ถึง 1  ->  แปลงเป็น 0 ถึง 1 สำหรับสี
const normalColor = add(mul(normalWorld, 0.5), 0.5);
mat.colorNode = vec4(normalColor, 1.0);
```

### UV Nodes

```javascript
import {
  uv,    // UV channel 0 (default)
  uv2,   // UV channel 1
} from 'three/tsl';

// ตัวอย่าง: ขยาย UV 5 เท่า
const scaledUV = mul(uv(), 5.0);

// ตัวอย่าง: animate UV
const animatedUV = add(uv(), vec2(timerGlobal(), 0.0));

// ตัวอย่าง: rotate UV
const rotateUV = tslFn(([uvInput, angle]) => {
  const c = cos(angle);
  const s = sin(angle);
  const pivot = vec2(0.5);
  const centered = sub(uvInput, pivot);
  const rotated = vec2(
    sub(mul(centered.x, c), mul(centered.y, s)),
    add(mul(centered.x, s), mul(centered.y, c))
  );
  return add(rotated, pivot);
});

const rotatedUV = rotateUV(uv(), timerGlobal());
```

### Color และ Texture Nodes

```javascript
import {
  color,     // สร้าง color constant
  texture,   // texture sampler
  cubeTexture,  // cube texture
} from 'three/tsl';

// ตัวอย่าง: load และใช้ texture
const loader = new THREE.TextureLoader();
const myTexture = loader.load('/textures/diffuse.jpg');
myTexture.colorSpace = THREE.SRGBColorSpace;

// ใช้ texture ใน node material
const texNode = texture(myTexture);

// ตัวอย่าง: texture lookup ด้วย custom UV
const scaledTex = texture(myTexture, mul(uv(), 2.0));

// ตัวอย่าง: mix สองสี
const blendedColor = mix(
  color(0xff0000),  // แดง
  color(0x0000ff),  // น้ำเงิน
  0.5               // 50% blend
);
```

### Uniform Nodes (ควบคุมจาก JavaScript)

```javascript
import { uniform } from 'three/tsl';

// สร้าง uniforms ต่างๆ
const uTime = uniform(0.0);           // float
const uColor = uniform(new THREE.Color(0xff0000));  // color
const uVec2 = uniform(new THREE.Vector2(1, 1));     // vec2
const uVec3 = uniform(new THREE.Vector3(0, 1, 0)); // vec3
const uMatrix = uniform(new THREE.Matrix4());       // mat4
const uBool = uniform(false);          // bool
const uInt = uniform(0);              // int

// อัพเดทจาก JavaScript
uTime.value = Date.now() / 1000;
uColor.value.set(0x00ff00);
uVec2.value.x = Math.sin(t);

// ใช้ใน material
const mat = new MeshStandardNodeMaterial();
mat.colorNode = vec4(uColor, 1.0);
```

### Time Nodes

```javascript
import {
  timerLocal,   // timer ของ renderer (reset เมื่อ renderer ถูก create)
  timerGlobal,  // timer แบบ global (ไม่ reset)
  timerDelta,   // delta time ของ frame นี้
} from 'three/tsl';

// ตัวอย่าง: animation ตาม time
const animated = sin(mul(timerGlobal(), 2.0));
```

---

## 3. Math Nodes: การคำนวณ

### Arithmetic Operations

```javascript
import {
  add,        // a + b
  sub,        // a - b
  mul,        // a * b
  div,        // a / b
  mod,        // a % b
  pow,        // a ^ b
  sqrt,       // √a
  abs,        // |a|
  sign,       // sign(a)
  negate,     // -a
  oneMinus,   // 1 - a
  reciprocal, // 1 / a
} from 'three/tsl';

// ตัวอย่าง
const result = add(mul(a, b), div(c, 2.0));
```

### Trigonometric Functions

```javascript
import {
  sin,   // sin(a)
  cos,   // cos(a)
  tan,   // tan(a)
  asin,  // asin(a)
  acos,  // acos(a)
  atan,  // atan(a) or atan2(y, x)
} from 'three/tsl';

// ตัวอย่าง: circular motion
const x = mul(cos(timerGlobal()), 3.0);
const y = mul(sin(timerGlobal()), 3.0);
```

### Interpolation และ Clamping

```javascript
import {
  clamp,      // clamp(x, min, max)
  saturate,   // clamp(x, 0, 1)
  mix,        // mix(a, b, t)
  smoothstep, // smoothstep(edge0, edge1, x)
  step,       // step(edge, x)
  lerp,       // alias for mix
} from 'three/tsl';

// ตัวอย่าง: smooth mask
const mask = smoothstep(0.3, 0.7, uv().x);
const finalColor = mix(color(0xff0000), color(0x0000ff), mask);
```

### Vector Operations

```javascript
import {
  dot,        // dot product
  cross,      // cross product
  normalize,  // normalize vector
  length,     // vector length
  distance,   // distance ระหว่างสองจุด
  reflect,    // reflection
  refract,    // refraction
  faceForward,
} from 'three/tsl';

// ตัวอย่าง: rim lighting
const viewDir = normalize(sub(cameraPosition, positionWorld));
const rim = oneMinus(saturate(dot(normalWorld, viewDir)));
const rimLight = mul(pow(rim, 3.0), color(0x00aaff));
```

---

## 4. tslFn(): สร้าง Custom Node Functions

### โครงสร้างของ tslFn

```javascript
import { tslFn } from 'three/tsl';

// รูปแบบพื้นฐาน
const myFunction = tslFn(([arg1, arg2, ...]) => {
  // ทำ operations ต่างๆ
  const result = add(arg1, arg2);
  return result;
});

// การใช้งาน
const output = myFunction(input1, input2);
```

### ตัวอย่าง: TSL Gradient Material เต็มรูปแบบ

```javascript
import * as THREE from 'three';
import {
  MeshStandardNodeMaterial,
  uv,
  positionWorld,
  normalWorld,
  color,
  uniform,
  timerGlobal,
  sin,
  cos,
  mul,
  add,
  sub,
  mix,
  smoothstep,
  clamp,
  vec2,
  vec3,
  vec4,
  float,
  abs,
  dot,
  normalize,
  oneMinus,
  saturate,
  pow,
  tslFn
} from 'three/tsl';

// --- Custom Noise Function ---
const hash2 = tslFn(([p]) => {
  const x = dot(p, vec2(127.1, 311.7));
  const y = dot(p, vec2(269.5, 183.3));
  return fract(mul(sin(vec2(x, y)), 43758.5453123));
});

const noise2D = tslFn(([p]) => {
  const i = floor(p);
  const f = fract(p);
  
  const u = mul(f, mul(f, sub(3.0, mul(2.0, f))));  // smoothstep
  
  const a = hash2(i);
  const b = hash2(add(i, vec2(1.0, 0.0)));
  const c = hash2(add(i, vec2(0.0, 1.0)));
  const d = hash2(add(i, vec2(1.0, 1.0)));
  
  return mix(
    mix(a.x, b.x, u.x),
    mix(c.x, d.x, u.x),
    u.y
  );
});

// --- Gradient Material with Noise ---
function createAdvancedGradientMaterial() {
  const mat = new MeshStandardNodeMaterial();
  
  // Uniforms
  const timeU = timerGlobal();
  const colorA = uniform(new THREE.Color(0x001133));
  const colorB = uniform(new THREE.Color(0x0066ff));
  const colorC = uniform(new THREE.Color(0x00ffcc));
  const noiseScale = uniform(3.0);
  const noiseSpeed = uniform(0.5);
  const rimStrength = uniform(2.0);
  
  // UV ที่ scale แล้ว
  const scaledUV = mul(uv(), noiseScale);
  
  // Animated noise
  const animNoise = noise2D(add(scaledUV, mul(timeU, noiseSpeed)));
  
  // Gradient ตาม Y position
  const heightGrad = clamp(add(mul(positionWorld.y, 0.5), 0.5), 0.0, 1.0);
  
  // รวม noise กับ gradient
  const combined = add(mul(heightGrad, 0.7), mul(animNoise, 0.3));
  
  // สีหลัก: 3-way gradient
  const col1 = mix(colorA, colorB, saturate(mul(combined, 2.0)));
  const col2 = mix(colorB, colorC, saturate(sub(mul(combined, 2.0), 1.0)));
  const baseColor = mix(col1, col2, step(0.5, combined));
  
  // Rim lighting effect
  const viewDir = normalize(sub(cameraPosition, positionWorld));
  const rim = pow(oneMinus(saturate(dot(normalWorld, viewDir))), 3.0);
  const rimColor = mul(colorC, mul(rim, rimStrength));
  
  // Final color
  const finalColor = add(baseColor, rimColor);
  
  mat.colorNode = vec4(finalColor, 1.0);
  mat.roughnessNode = float(0.3);
  mat.metalnessNode = float(0.7);
  mat.emissiveNode = mul(rimColor, 0.3);
  
  return mat;
}
```

---

## 5. ตัวอย่าง TSL Animated Shader เต็มรูปแบบ

```javascript
// --- Lava / Magma Material ---
function createLavaMaterial() {
  const mat = new MeshStandardNodeMaterial();
  
  const timeU = timerGlobal();
  const lavaColor1 = uniform(new THREE.Color(0xff2200));   // deep red
  const lavaColor2 = uniform(new THREE.Color(0xff8800));   // orange
  const lavaColor3 = uniform(new THREE.Color(0xffff00));   // yellow hot
  const crackColor = uniform(new THREE.Color(0x1a0500));   // dark cracks
  const speed = uniform(0.3);
  const scale = uniform(4.0);
  
  const animUV = add(mul(uv(), scale), mul(timeU, speed));
  
  // Layered noise
  const n1 = noise2D(animUV);
  const n2 = noise2D(mul(add(animUV, 10.0), 2.0));
  const n3 = noise2D(mul(sub(animUV, 5.0), 4.0));
  
  const fbm = add(mul(n1, 0.5), add(mul(n2, 0.25), mul(n3, 0.125)));
  
  // Crack pattern: ค่า fbm ต่ำ = รอยแตก
  const crack = smoothstep(0.0, 0.3, fbm);
  
  // Hot spots: ค่า fbm สูง = จุดร้อน
  const hot = smoothstep(0.6, 1.0, fbm);
  
  // สีตาม temperature
  const coolLava = mix(crackColor, lavaColor1, crack);
  const warmLava = mix(lavaColor1, lavaColor2, smoothstep(0.3, 0.6, fbm));
  const hotLava = mix(warmLava, lavaColor3, hot);
  
  const finalColor = mix(coolLava, hotLava, smoothstep(0.2, 0.5, fbm));
  
  // Emissive สำหรับ glow effect
  const emissive = mul(hotLava, mul(hot, 2.0));
  
  mat.colorNode = vec4(finalColor, 1.0);
  mat.emissiveNode = emissive;
  mat.roughnessNode = mix(float(0.9), float(0.1), hot);
  mat.metalnessNode = float(0.0);
  
  return mat;
}

// --- Water Surface Material ---
function createWaterMaterial() {
  const mat = new MeshStandardNodeMaterial();
  mat.transparent = true;
  mat.side = THREE.FrontSide;
  
  const timeU = timerGlobal();
  const waterColor = uniform(new THREE.Color(0x006994));
  const foamColor = uniform(new THREE.Color(0xffffff));
  const waveHeight = uniform(0.1);
  const waveSpeed = uniform(1.5);
  const waveFreq = uniform(5.0);
  
  // Wave animation ด้วย normal map technique
  const uv0 = add(uv(), mul(vec2(timeU, 0.0), 0.1));
  const uv1 = add(uv(), mul(vec2(0.0, timeU), 0.15));
  
  // Multi-wave normal
  const wave1 = sin(add(mul(uv0.x, waveFreq), mul(timeU, waveSpeed)));
  const wave2 = sin(add(mul(uv1.y, mul(waveFreq, 1.3)), mul(timeU, mul(waveSpeed, 0.7))));
  const combinedWave = mul(add(wave1, wave2), 0.5);
  
  // Vertex displacement (positionNode)
  const displacement = vec3(0.0, mul(combinedWave, waveHeight), 0.0);
  mat.positionNode = add(positionLocal, displacement);
  
  // Normal perturbation
  const normalOffset = mul(combinedWave, 0.3);
  const perturbedNormal = normalize(add(normalWorld, vec3(normalOffset, 0.0, normalOffset)));
  mat.normalNode = perturbedNormal;
  
  // Fresnel for water transparency
  const viewDir = normalize(sub(cameraPosition, positionWorld));
  const fresnel = pow(oneMinus(saturate(dot(perturbedNormal, viewDir))), 4.0);
  
  // Foam at wave peaks
  const foam = smoothstep(0.6, 1.0, add(combinedWave, 0.5));
  
  // Final color
  const baseColor = mix(waterColor, foamColor, foam);
  const alpha = mix(float(0.7), float(0.95), fresnel);
  
  mat.colorNode = vec4(baseColor, alpha);
  mat.roughnessNode = float(0.1);
  mat.metalnessNode = float(0.0);
  mat.envMapIntensity = 1.5;
  
  return mat;
}
```

---

## 6. TSL PBR Override: การ Override PBR Properties

```javascript
// การ Override PBR properties แบบซับซ้อน
function createAdvancedPBRMaterial(texturePaths) {
  const mat = new MeshStandardNodeMaterial();
  
  const loader = new THREE.TextureLoader();
  
  // โหลด textures
  const albedoMap = loader.load(texturePaths.albedo);
  albedoMap.colorSpace = THREE.SRGBColorSpace;
  
  const normalMap = loader.load(texturePaths.normal);
  const roughnessMap = loader.load(texturePaths.roughness);
  const metalnessMap = loader.load(texturePaths.metalness);
  const aoMap = loader.load(texturePaths.ao);
  
  // Uniform controls
  const timeU = timerGlobal();
  const wetness = uniform(0.0);       // 0 = dry, 1 = wet
  const damage = uniform(0.0);        // 0 = new, 1 = damaged
  const emissiveStrength = uniform(0.0);
  const uvTiling = uniform(1.0);
  
  // Scale UV
  const tiledUV = mul(uv(), uvTiling);
  
  // --- Color Node ---
  const baseAlbedo = texture(albedoMap, tiledUV).rgb;
  
  // เปื้อนและเสียหาย
  const damageNoise = noise2D(mul(tiledUV, 10.0));
  const damagePattern = smoothstep(
    sub(0.6, mul(damage, 0.5)), 
    add(0.6, mul(damage, 0.2)), 
    damageNoise
  );
  
  // สีความชื้น (wet darkens color)
  const dryColor = baseAlbedo;
  const wetColor = mul(baseAlbedo, 0.6);  // wet = darker
  const damagedColor = mix(baseAlbedo, vec3(0.3, 0.15, 0.1), damagePattern);
  
  const moistColor = mix(dryColor, wetColor, wetness);
  const finalAlbedo = mix(moistColor, damagedColor, damage);
  
  mat.colorNode = vec4(finalAlbedo, 1.0);
  
  // --- Roughness Node ---
  const baseRoughness = texture(roughnessMap, tiledUV).r;
  
  // Wet surface = smoother (lower roughness)
  const wetRoughness = mix(baseRoughness, mul(baseRoughness, 0.3), wetness);
  
  // Damaged = rougher
  const damagedRoughness = mix(wetRoughness, float(0.95), mul(damagePattern, damage));
  
  mat.roughnessNode = damagedRoughness;
  
  // --- Metalness Node ---
  const baseMetalness = texture(metalnessMap, tiledUV).r;
  
  // Rust on damaged metal
  const rustMetalness = mix(baseMetalness, float(0.0), mul(damagePattern, damage));
  
  mat.metalnessNode = rustMetalness;
  
  // --- Normal Node ---
  const baseNormal = texture(normalMap, tiledUV).rgb;
  
  // Flatten normals when wet (puddles)
  const normalStrength = mix(float(1.0), float(0.3), wetness);
  const adjustedNormal = mix(vec3(0.5, 0.5, 1.0), baseNormal, normalStrength);
  
  mat.normalNode = adjustedNormal;
  
  // --- AO Node ---
  mat.aoNode = texture(aoMap, tiledUV).r;
  
  // --- Emissive Node (optional) ---
  const glowColor = uniform(new THREE.Color(0x00ffff));
  mat.emissiveNode = mul(vec3(glowColor), emissiveStrength);
  
  // Return material with controls
  return {
    material: mat,
    controls: {
      setWetness: (v) => { wetness.value = v; },
      setDamage: (v) => { damage.value = v; },
      setEmissive: (v) => { emissiveStrength.value = v; },
      setTiling: (v) => { uvTiling.value = v; },
    }
  };
}

// ตัวอย่างการใช้งาน
const { material, controls } = createAdvancedPBRMaterial({
  albedo: '/textures/metal_albedo.jpg',
  normal: '/textures/metal_normal.jpg',
  roughness: '/textures/metal_roughness.jpg',
  metalness: '/textures/metal_metalness.jpg',
  ao: '/textures/metal_ao.jpg'
});

// Control wetness ด้วย UI
document.getElementById('wetSlider').addEventListener('input', (e) => {
  controls.setWetness(parseFloat(e.target.value));
});
```

---

## 7. Node Graph Concept: การเชื่อม Nodes

### ตัวอย่าง Node Graph ที่ซับซ้อน

```javascript
// ===== TOON SHADING ด้วย TSL =====
function createToonMaterial(steps = 4) {
  const mat = new MeshStandardNodeMaterial();
  
  const mainColor = uniform(new THREE.Color(0x4488ff));
  const shadowColor = uniform(new THREE.Color(0x112244));
  const numSteps = uniform(float(steps));
  
  // คำนวณ diffuse lighting แบบ toon
  const toonDiffuse = tslFn(([normal, lightDir, shadowCol, mainCol, steps]) => {
    // Diffuse dot product
    const NdotL = dot(normal, lightDir);
    
    // Quantize ให้เป็น steps
    const quantized = div(
      floor(mul(add(NdotL, 1.0), mul(steps, 0.5))),
      steps
    );
    
    // Mix ระหว่าง shadow กับ main color
    return mix(shadowCol, mainCol, saturate(quantized));
  });
  
  // Override lighting calculation
  // (This is simplified - actual toon needs custom lighting)
  const N = normalWorld;
  const L = normalize(vec3(1.0, 1.0, 0.0));  // simplified light direction
  
  const toonColor = toonDiffuse(N, L, shadowColor, mainColor, numSteps);
  
  // Outline: rim ที่เป็น solid black
  const viewDir = normalize(sub(cameraPosition, positionWorld));
  const outline = step(0.3, oneMinus(dot(N, viewDir)));
  
  const finalColor = mix(toonColor, vec3(0.0), outline);
  
  mat.colorNode = vec4(finalColor, 1.0);
  mat.roughnessNode = float(1.0);
  mat.metalnessNode = float(0.0);
  
  return mat;
}

// ===== IRIDESCENT / HOLOGRAPHIC Material =====
function createIridescentMaterial() {
  const mat = new MeshStandardNodeMaterial();
  mat.side = THREE.DoubleSide;
  
  const timeU = timerGlobal();
  const iridescenceStrength = uniform(1.0);
  const iridescenceFreq = uniform(8.0);
  
  // Compute view-dependent color shift
  const viewDir = normalize(sub(cameraPosition, positionWorld));
  const NdotV = dot(normalWorld, viewDir);
  
  // Map to hue
  const hue = add(mul(NdotV, iridescenceFreq), mul(timeU, 0.5));
  
  // HSV to RGB
  const iriColor = tslFn(([h]) => {
    const r = abs(sub(mul(h, 6.0), 3.0));
    const g = sub(2.0, abs(sub(mul(h, 6.0), 2.0)));
    const b = sub(2.0, abs(sub(mul(h, 6.0), 4.0)));
    return clamp(sub(vec3(r, g, b), 1.0), 0.0, 1.0);
  });
  
  const iridColor = iriColor(fract(hue));
  
  // Base silver/chrome look
  const baseColor = vec3(0.8, 0.8, 0.8);
  
  // Mix base กับ iridescent
  const finalColor = mix(
    baseColor, 
    iridColor, 
    mul(iridescenceStrength, pow(oneMinus(NdotV), 2.0))
  );
  
  mat.colorNode = vec4(finalColor, 1.0);
  mat.roughnessNode = float(0.05);
  mat.metalnessNode = float(1.0);
  
  return mat;
}
```

---

## 8. ตัวอย่างเต็มรูปแบบ: TSL Shader Gallery

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title>TSL Shader Gallery</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { overflow: hidden; background: #111; font-family: 'Kanit', sans-serif; }
    canvas { display: block; }
    #ui {
      position: fixed;
      top: 20px;
      right: 20px;
      background: rgba(0,0,0,0.7);
      border: 1px solid rgba(255,255,255,0.2);
      border-radius: 8px;
      padding: 15px;
      color: white;
      width: 220px;
    }
    h3 { font-size: 14px; margin-bottom: 10px; color: #88aaff; }
    .control { margin: 8px 0; }
    label { font-size: 12px; display: block; margin-bottom: 3px; }
    input[type="range"] { width: 100%; }
    .shader-select {
      width: 100%;
      background: #333;
      color: white;
      border: 1px solid #555;
      padding: 5px;
      border-radius: 4px;
      margin-bottom: 10px;
    }
  </style>
</head>
<body>
  <canvas id="c"></canvas>
  <div id="ui">
    <h3>TSL Shader Gallery</h3>
    <div class="control">
      <label>เลือก Shader:</label>
      <select class="shader-select" id="shaderSelect">
        <option value="gradient">Animated Gradient</option>
        <option value="lava">Lava</option>
        <option value="holographic">Holographic</option>
        <option value="iridescent">Iridescent</option>
        <option value="toon">Toon Shading</option>
      </select>
    </div>
    <div class="control">
      <label>Speed: <span id="speedVal">1.0</span></label>
      <input type="range" id="speed" min="0" max="3" step="0.1" value="1">
    </div>
    <div class="control">
      <label>Intensity: <span id="intensityVal">1.0</span></label>
      <input type="range" id="intensity" min="0" max="2" step="0.1" value="1">
    </div>
    <div class="control">
      <label>Scale: <span id="scaleVal">1.0</span></label>
      <input type="range" id="scale" min="0.1" max="10" step="0.1" value="1">
    </div>
  </div>

  <script type="module">
    import * as THREE from 'https://cdn.skypack.dev/three@0.160.0';
    
    // NOTE: ตัวอย่างนี้ใช้ ShaderMaterial แทน TSL เพราะ CDN ไม่รองรับ
    // ในการใช้งานจริงให้ import จาก three/tsl ใน build environment
    
    const canvas = document.getElementById('c');
    const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.toneMappingExposure = 1.0;
    renderer.outputColorSpace = THREE.SRGBColorSpace;
    
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x111111);
    
    const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
    camera.position.set(0, 0, 4);
    
    // Lighting
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.3);
    scene.add(ambientLight);
    
    const dirLight = new THREE.DirectionalLight(0xffffff, 2);
    dirLight.position.set(3, 5, 3);
    scene.add(dirLight);
    
    const pointLight1 = new THREE.PointLight(0xff4400, 3, 20);
    pointLight1.position.set(-3, 2, 2);
    scene.add(pointLight1);
    
    const pointLight2 = new THREE.PointLight(0x0044ff, 3, 20);
    pointLight2.position.set(3, -2, 2);
    scene.add(pointLight2);
    
    // เนื้อ geometry เพื่อแสดง shader
    const geo = new THREE.SphereGeometry(1.5, 128, 128);
    
    // === Shader definitions ===
    const shaders = {
      gradient: {
        uniforms: {
          uTime: { value: 0 },
          uSpeed: { value: 1 },
          uScale: { value: 3 },
          uColorA: { value: new THREE.Vector3(0, 0.5, 1) },
          uColorB: { value: new THREE.Vector3(1, 0.2, 0.8) }
        },
        vertexShader: `
          varying vec2 vUv;
          varying vec3 vNormal;
          varying vec3 vWorldPos;
          void main() {
            vUv = uv;
            vNormal = normalize(normalMatrix * normal);
            vWorldPos = (modelMatrix * vec4(position, 1.0)).xyz;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
          }
        `,
        fragmentShader: `
          uniform float uTime;
          uniform float uSpeed;
          uniform float uScale;
          uniform vec3 uColorA;
          uniform vec3 uColorB;
          varying vec2 vUv;
          varying vec3 vNormal;
          varying vec3 vWorldPos;
          
          // Simple noise
          float hash(vec2 p) {
            return fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5453);
          }
          float noise(vec2 p) {
            vec2 i = floor(p), f = fract(p);
            vec2 u = f * f * (3.0 - 2.0 * f);
            return mix(
              mix(hash(i), hash(i + vec2(1,0)), u.x),
              mix(hash(i + vec2(0,1)), hash(i + vec2(1,1)), u.x),
              u.y
            );
          }
          
          void main() {
            vec2 scaledUv = vUv * uScale;
            float t = uTime * uSpeed;
            
            float n1 = noise(scaledUv + vec2(t * 0.3, t * 0.2));
            float n2 = noise(scaledUv * 2.0 - vec2(t * 0.5, 0.0));
            float n = n1 * 0.6 + n2 * 0.4;
            
            vec3 col = mix(uColorA, uColorB, n);
            
            // Rim light
            vec3 viewDir = normalize(cameraPosition - vWorldPos);
            float rim = pow(1.0 - max(dot(vNormal, viewDir), 0.0), 3.0);
            col += rim * 0.5;
            
            gl_FragColor = vec4(col, 1.0);
          }
        `
      },
      lava: {
        uniforms: {
          uTime: { value: 0 },
          uSpeed: { value: 0.3 },
          uScale: { value: 4 },
          uColorA: { value: new THREE.Vector3(1, 0.13, 0) },
          uColorB: { value: new THREE.Vector3(1, 0.53, 0) }
        },
        vertexShader: `
          varying vec2 vUv;
          varying vec3 vNormal;
          varying vec3 vWorldPos;
          void main() {
            vUv = uv;
            vNormal = normalize(normalMatrix * normal);
            vWorldPos = (modelMatrix * vec4(position, 1.0)).xyz;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
          }
        `,
        fragmentShader: `
          uniform float uTime;
          uniform float uSpeed;
          uniform float uScale;
          uniform vec3 uColorA;
          uniform vec3 uColorB;
          varying vec2 vUv;
          varying vec3 vNormal;
          varying vec3 vWorldPos;
          
          float hash(vec2 p) { return fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5453); }
          float noise(vec2 p) {
            vec2 i = floor(p), f = fract(p);
            vec2 u = f * f * (3.0 - 2.0 * f);
            return mix(mix(hash(i),hash(i+vec2(1,0)),u.x),mix(hash(i+vec2(0,1)),hash(i+vec2(1,1)),u.x),u.y);
          }
          float fbm(vec2 p) {
            float v = 0.0; float a = 0.5;
            for(int i = 0; i < 5; i++) { v += a * noise(p); p = p * 2.0 + 1.7; a *= 0.5; }
            return v;
          }
          
          void main() {
            vec2 uv = vUv * uScale + uTime * uSpeed;
            float n = fbm(uv);
            
            vec3 hotColor = vec3(1.0, 1.0, 0.0);
            vec3 col = n < 0.3 ? vec3(0.05, 0.01, 0.0) : 
                       n < 0.6 ? mix(uColorA, uColorB, (n - 0.3) / 0.3) : 
                       mix(uColorB, hotColor, (n - 0.6) / 0.4);
            
            float emissive = smoothstep(0.5, 1.0, n);
            gl_FragColor = vec4(col + emissive * hotColor * 0.5, 1.0);
          }
        `
      }
    };
    
    // สร้าง mesh
    let currentShader = 'gradient';
    let mesh = new THREE.Mesh(geo, new THREE.ShaderMaterial(shaders.gradient));
    scene.add(mesh);
    
    // เปลี่ยน shader
    document.getElementById('shaderSelect').addEventListener('change', (e) => {
      const shaderName = e.target.value;
      if (shaders[shaderName]) {
        mesh.material.dispose();
        mesh.material = new THREE.ShaderMaterial(shaders[shaderName]);
        currentShader = shaderName;
      }
    });
    
    // Controls
    const speedEl = document.getElementById('speedVal');
    const intensityEl = document.getElementById('intensityVal');
    const scaleEl = document.getElementById('scaleVal');
    
    document.getElementById('speed').addEventListener('input', (e) => {
      const v = parseFloat(e.target.value);
      speedEl.textContent = v.toFixed(1);
      if (mesh.material.uniforms.uSpeed) mesh.material.uniforms.uSpeed.value = v;
    });
    
    document.getElementById('scale').addEventListener('input', (e) => {
      const v = parseFloat(e.target.value);
      scaleEl.textContent = v.toFixed(1);
      if (mesh.material.uniforms.uScale) mesh.material.uniforms.uScale.value = v;
    });
    
    // Window resize
    window.addEventListener('resize', () => {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    });
    
    // Animation
    const clock = new THREE.Clock();
    function animate() {
      requestAnimationFrame(animate);
      
      const time = clock.getElapsedTime();
      
      // Update time uniform
      if (mesh.material.uniforms && mesh.material.uniforms.uTime) {
        mesh.material.uniforms.uTime.value = time;
      }
      
      mesh.rotation.y = time * 0.3;
      mesh.rotation.x = Math.sin(time * 0.2) * 0.2;
      
      // Animate lights
      pointLight1.position.x = Math.sin(time * 0.7) * 3;
      pointLight1.position.y = Math.cos(time * 0.5) * 2;
      pointLight2.position.x = Math.cos(time * 0.8) * 3;
      pointLight2.position.y = Math.sin(time * 0.6) * 2;
      
      renderer.render(scene, camera);
    }
    
    animate();
  </script>
</body>
</html>
```

---

## 9. สรุปและ Best Practices

### เมื่อไหร่ควรใช้ TSL/NodeMaterial?
- เมื่อต้องการ shader ที่ทำงานบน WebGPU ในอนาคต
- เมื่อต้องการ reuse shader logic ระหว่าง materials
- เมื่อต้องการ cross-platform compatibility

### เมื่อไหร่ยังใช้ ShaderMaterial ได้?
- WebGL-only projects ที่มี GLSL code พร้อมแล้ว
- Performance critical applications ที่ต้องการควบคุม shader อย่างเต็มที่
- Legacy projects

### TSL Node Naming Convention

```javascript
// ตัวแปร ใช้ camelCase
const myColor = color(0xff0000);
const timeValue = timerGlobal();

// Function ใช้ PascalCase หรือ camelCase
const MyCustomFn = tslFn(([input]) => {
  return input;
});

// Uniform ใช้ 'u' prefix
const uSpeed = uniform(1.0);
const uColor = uniform(new THREE.Color());
```

ในบทถัดไป (Part 83) เราจะเรียนรู้เกี่ยวกับ Advanced Post-Processing ด้วย pmndrs/postprocessing library
