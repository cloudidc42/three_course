# Part 29: Procedural Noise & Patterns

## บทนำ: Procedural Generation คืออะไร?

Procedural generation คือการสร้าง content โดยใช้ mathematical functions แทนการใช้ assets ที่สร้างด้วยมือ ข้อดี:
- **ไม่ต้องใช้ texture** - ประหยัด memory
- **Infinite resolution** - zoom ได้เท่าไหรก็ได้
- **Animatable** - เคลื่อนที่ได้อย่างเป็นธรรมชาติ
- **Parameterizable** - ปรับแต่งได้ง่าย
- **ไม่มี seam** - ไม่มีรอยต่อ

---

## Classic Perlin Noise

Ken Perlin คิดค้น noise function นี้ขึ้นในปี 1983 สำหรับภาพยนตร์ TRON มีคุณสมบัติ:
- **Smooth** - ค่าเปลี่ยนแปลงอย่างต่อเนื่อง
- **Bounded** - ค่าอยู่ในช่วง [-1, 1]
- **Band-limited** - มี frequency ที่กำหนด

```glsl
// Classic Perlin Noise 2D Implementation
vec2 fade(vec2 t) {
    // Smoothstep: 6t^5 - 15t^4 + 10t^3 (Perlin's improvement)
    return t * t * t * (t * (t * 6.0 - 15.0) + 10.0);
}

float grad(float hash, float x, float y) {
    int h = int(hash) & 7;
    float u = h < 4 ? x : y;
    float v = h < 4 ? y : x;
    return ((h & 1) == 0 ? u : -u) + ((h & 2) == 0 ? v : -v);
}

// Permutation table
float perm(float x) {
    // ใช้ sin ทำ hash (ไม่ค่อยสมบูรณ์แต่ใช้ใน GLSL ได้)
    return fract(sin(x * 127.1) * 43758.5453);
}

float perlin2D(vec2 p) {
    vec2 i = floor(p);
    vec2 f = fract(p);
    vec2 u = fade(f);
    
    // Hash corner points
    float a = perm(i.x + perm(i.y));
    float b = perm(i.x + 1.0 + perm(i.y));
    float c = perm(i.x + perm(i.y + 1.0));
    float d = perm(i.x + 1.0 + perm(i.y + 1.0));
    
    // Compute gradients
    float ga = grad(a, f.x, f.y);
    float gb = grad(b, f.x - 1.0, f.y);
    float gc = grad(c, f.x, f.y - 1.0);
    float gd = grad(d, f.x - 1.0, f.y - 1.0);
    
    // Bilinear interpolate
    return mix(mix(ga, gb, u.x), mix(gc, gd, u.x), u.y);
}
```

---

## Simplex Noise

Simplex noise พัฒนาโดย Perlin เพื่อแก้ปัญหา computational complexity ของ classic Perlin:
- **เร็วกว่า** Perlin noise โดยเฉพาะใน high dimensions
- **Fewer artifacts** - ไม่มี grid-aligned artifacts
- **Better gradient distribution**

```glsl
// Simplex Noise 2D
vec3 permute(vec3 x) {
    return mod(((x * 34.0) + 1.0) * x, 289.0);
}

float snoise(vec2 v) {
    const vec4 C = vec4(
        0.211324865405187,  // (3.0-sqrt(3.0))/6.0
        0.366025403784439,  // 0.5*(sqrt(3.0)-1.0)
       -0.577350269189626,  // -1.0 + 2.0 * C.x
        0.024390243902439   // 1.0 / 41.0
    );
    
    // First corner
    vec2 i  = floor(v + dot(v, C.yy));
    vec2 x0 = v - i + dot(i, C.xx);
    
    // Other corners
    vec2 i1;
    i1 = (x0.x > x0.y) ? vec2(1.0, 0.0) : vec2(0.0, 1.0);
    vec4 x12 = x0.xyxy + C.xxzz;
    x12.xy -= i1;
    
    // Permutations
    i = mod(i, 289.0);
    vec3 p = permute(
        permute(i.y + vec3(0.0, i1.y, 1.0)) + i.x + vec3(0.0, i1.x, 1.0)
    );
    
    vec3 m = max(0.5 - vec3(
        dot(x0, x0),
        dot(x12.xy, x12.xy),
        dot(x12.zw, x12.zw)
    ), 0.0);
    m = m * m;
    m = m * m;
    
    // Gradients
    vec3 x = 2.0 * fract(p * C.www) - 1.0;
    vec3 h = abs(x) - 0.5;
    vec3 ox = floor(x + 0.5);
    vec3 a0 = x - ox;
    
    // Normalise gradients
    m *= 1.79284291400159 - 0.85373472095314 * (a0 * a0 + h * h);
    
    // Compute final noise value
    vec3 g;
    g.x  = a0.x  * x0.x  + h.x  * x0.y;
    g.yz = a0.yz * x12.xz + h.yz * x12.yw;
    
    return 130.0 * dot(m, g);
}
```

---

## Fractal Brownian Motion (fBm)

fBm คือการ stack noise หลาย octaves เข้าด้วยกัน แต่ละ octave มี frequency สูงขึ้น แต่ amplitude ต่ำลง

```glsl
// fBm - Fractal Brownian Motion
float fbm(vec2 p) {
    float value = 0.0;
    float amplitude = 0.5;
    float frequency = 1.0;
    
    // Rotation matrix เพื่อลด grid artifacts
    mat2 rot = mat2(
        cos(0.5), sin(0.5),
       -sin(0.5), cos(0.5)
    );
    
    for (int i = 0; i < 8; i++) {
        value += amplitude * snoise(p * frequency);
        amplitude *= 0.5;   // Lacunarity: ลด amplitude ครึ่งหนึ่งทุก octave
        frequency *= 2.0;   // Gain: เพิ่ม frequency 2 เท่าทุก octave
        p = rot * p;        // Rotate เพื่อลด artifacts
    }
    
    return value;
}

// fBm ที่ปรับแต่งได้
float fbmCustom(vec2 p, int octaves, float lacunarity, float gain) {
    float value = 0.0;
    float amplitude = 1.0;
    float frequency = 1.0;
    float totalAmplitude = 0.0;
    
    for (int i = 0; i < 10; i++) {
        if (i >= octaves) break;
        value += amplitude * snoise(p * frequency);
        totalAmplitude += amplitude;
        amplitude *= gain;
        frequency *= lacunarity;
    }
    
    return value / totalAmplitude; // Normalize
}
```

---

## Domain Warping

Domain Warping คือการใช้ noise ไป warp UV ของ noise อื่น ทำให้ได้ pattern ที่ organic มาก

```glsl
// Domain Warping - Inigo Quilez's technique
float warpedFbm(vec2 p, float t) {
    // Level 1: สร้าง "flow field"
    vec2 q = vec2(
        fbm(p + vec2(0.0, 0.0)),
        fbm(p + vec2(5.2, 1.3))
    );
    
    // Level 2: warp ด้วย q (and add time for animation)
    vec2 r = vec2(
        fbm(p + 4.0 * q + vec2(1.7, 9.2) + t * 0.1),
        fbm(p + 4.0 * q + vec2(8.3, 2.8) + t * 0.15)
    );
    
    // Level 3: final noise with warped coordinates
    return fbm(p + 4.0 * r);
}

// Simple single-level domain warp
vec3 domainWarpedColor(vec2 p, float t) {
    // Warp UV
    float wx = snoise(p * 2.0 + t * 0.2) * 0.5;
    float wy = snoise(p * 2.0 + vec2(3.1, 7.4) + t * 0.15) * 0.5;
    vec2 warpedP = p + vec2(wx, wy);
    
    // Sample with warped UV
    float n = fbm(warpedP * 3.0);
    
    // Cosine palette
    return vec3(0.5) + vec3(0.5) * cos(6.28318 * (vec3(1.0, 1.0, 0.5) * n + vec3(0.0, 0.33, 0.67)));
}
```

---

## Voronoi / Cellular Noise

Voronoi noise แบ่งพื้นที่ออกเป็น cells โดย cell แต่ละ cell เป็นส่วนที่ใกล้ "feature point" มากที่สุด

```glsl
// Voronoi Noise
struct VoronoiResult {
    float dist1;    // ระยะห่างถึง nearest center
    float dist2;    // ระยะห่างถึง second nearest
    float edge;     // ระยะห่างถึง cell boundary
    vec2  cellId;   // ID ของ cell ที่ใกล้ที่สุด
};

vec2 voronoiPoint(vec2 cell) {
    // Random point ภายใน cell
    return cell + hash22(cell) * 0.8 + 0.1; // jitter 10-90% ของ cell
}

VoronoiResult voronoi(vec2 p) {
    vec2 i = floor(p);
    vec2 f = fract(p);
    
    float d1 = 1e10, d2 = 1e10;
    vec2 nearCell;
    
    // ตรวจสอบ neighbors 5x5 (ป้องกัน edge cases)
    for (int y = -2; y <= 2; y++) {
        for (int x = -2; x <= 2; x++) {
            vec2 n = vec2(float(x), float(y));
            vec2 g = voronoiPoint(i + n);
            vec2 diff = n + g - f;
            float d = dot(diff, diff);
            
            if (d < d1) {
                d2 = d1; d1 = d;
                nearCell = i + n;
            } else if (d < d2) {
                d2 = d;
            }
        }
    }
    
    VoronoiResult result;
    result.dist1 = sqrt(d1);
    result.dist2 = sqrt(d2);
    result.edge  = result.dist2 - result.dist1; // ยิ่งเล็ก ยิ่งอยู่ใกล้ขอบ
    result.cellId = nearCell;
    
    return result;
}

// Animated Voronoi
VoronoiResult animatedVoronoi(vec2 p, float t) {
    vec2 i = floor(p);
    vec2 f = fract(p);
    
    float d1 = 1e10, d2 = 1e10;
    vec2 nearCell;
    
    for (int y = -2; y <= 2; y++) {
        for (int x = -2; x <= 2; x++) {
            vec2 n = vec2(float(x), float(y));
            // Animate: feature points เคลื่อนที่ตามเวลา
            vec2 h = hash22(i + n);
            vec2 anim = sin(h * 6.28318 + t) * 0.4 + 0.5;
            vec2 g = n + anim - f;
            float d = dot(g, g);
            
            if (d < d1) {
                d2 = d1; d1 = d;
                nearCell = i + n;
            } else if (d < d2) {
                d2 = d;
            }
        }
    }
    
    VoronoiResult result;
    result.dist1 = sqrt(d1);
    result.dist2 = sqrt(d2);
    result.edge  = result.dist2 - result.dist1;
    result.cellId = nearCell;
    
    return result;
}
```

---

## Procedural Marble Texture

```glsl
// Marble = fBm + sin distortion

uniform float uTime;
uniform float uMarbleScale;
uniform float uNoiseFreq;
uniform float uTurbulence;

varying vec2 vUv;

float fbm(vec2 p) {
    // ใช้ implementation ด้านบน
    float v = 0.0;
    float a = 0.5;
    for (int i = 0; i < 6; i++) {
        v += a * snoise(p);
        p = p * 2.0 + vec2(3.1, 1.7);
        a *= 0.5;
    }
    return v;
}

vec3 marble(vec2 uv, float t) {
    // Noise-based distortion
    float noise = fbm(uv * uNoiseFreq) * uTurbulence;
    
    // Marble veins: sin function + noise distortion
    float vein = sin(uv.x * uMarbleScale + noise * 10.0) * 0.5 + 0.5;
    
    // Add time animation (slowly flowing)
    vein = sin(uv.x * uMarbleScale + noise * 10.0 + t * 0.2) * 0.5 + 0.5;
    
    // สีหินอ่อน: ขาว, เทา, เทาเข้ม
    vec3 colorLight = vec3(0.95, 0.93, 0.90);  // ขาวนวล
    vec3 colorDark  = vec3(0.3, 0.28, 0.25);   // เทาเข้ม
    vec3 colorMid   = vec3(0.7, 0.65, 0.60);   // เทา
    
    // Two-level marble
    float v1 = smoothstep(0.4, 0.6, vein);
    float v2 = smoothstep(0.0, 0.3, vein) * (1.0 - v1);
    
    vec3 color = colorLight;
    color = mix(color, colorDark, v2 * 0.7);
    color = mix(color, colorMid, v1 * 0.5);
    
    // Add subtle grain
    float grain = snoise(uv * 100.0) * 0.02;
    color += grain;
    
    return clamp(color, 0.0, 1.0);
}
```

---

## Procedural Cloud Shader

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Procedural Clouds</title>
    <style>
        * { margin: 0; padding: 0; }
        canvas { display: block; }
        #ui {
            position: fixed;
            top: 15px;
            right: 15px;
            color: white;
            font-family: monospace;
            background: rgba(0,0,0,0.8);
            padding: 12px;
            border-radius: 8px;
            width: 220px;
        }
        input[type="range"] { width: 100%; }
    </style>
</head>
<body>
<div id="ui">
    <b>Cloud Controls</b>
    <label>Coverage: <input type="range" id="coverage" min="0" max="1" value="0.5" step="0.01"></label>
    <label>Density: <input type="range" id="density" min="0" max="5" value="2" step="0.1"></label>
    <label>Speed: <input type="range" id="speed" min="0" max="2" value="0.3" step="0.01"></label>
    <label>Scale: <input type="range" id="scale" min="1" max="10" value="3" step="0.1"></label>
    <label>Octaves: <input type="range" id="octaves" min="1" max="8" value="6" step="1"></label>
    <label>Sky Color A: <input type="color" id="skyA" value="#1a3a6e"></label>
    <label>Sky Color B: <input type="color" id="skyB" value="#4a7abf"></label>
</div>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, innerWidth / innerHeight, 0.1, 10);
camera.position.z = 1;
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
document.body.appendChild(renderer.domElement);

const fragmentShader = `
    precision highp float;
    
    uniform float uTime;
    uniform float uCoverage;
    uniform float uDensity;
    uniform float uSpeed;
    uniform float uScale;
    uniform int   uOctaves;
    uniform vec3  uSkyColorA;
    uniform vec3  uSkyColorB;
    
    varying vec2 vUv;
    
    // === Noise Library ===
    vec3 permute(vec3 x) { return mod(((x * 34.0) + 1.0) * x, 289.0); }
    
    float snoise(vec2 v) {
        const vec4 C = vec4(0.211324865405187, 0.366025403784439,
                           -0.577350269189626, 0.024390243902439);
        vec2 i  = floor(v + dot(v, C.yy));
        vec2 x0 = v - i + dot(i, C.xx);
        vec2 i1 = (x0.x > x0.y) ? vec2(1.0, 0.0) : vec2(0.0, 1.0);
        vec4 x12 = x0.xyxy + C.xxzz;
        x12.xy -= i1;
        i = mod(i, 289.0);
        vec3 p = permute(permute(i.y + vec3(0.0, i1.y, 1.0)) + i.x + vec3(0.0, i1.x, 1.0));
        vec3 m = max(0.5 - vec3(dot(x0,x0), dot(x12.xy,x12.xy), dot(x12.zw,x12.zw)), 0.0);
        m = m * m; m = m * m;
        vec3 x = 2.0 * fract(p * C.www) - 1.0;
        vec3 h = abs(x) - 0.5;
        vec3 ox = floor(x + 0.5);
        vec3 a0 = x - ox;
        m *= 1.79284291400159 - 0.85373472095314 * (a0*a0 + h*h);
        vec3 g;
        g.x  = a0.x  * x0.x  + h.x  * x0.y;
        g.yz = a0.yz * x12.xz + h.yz * x12.yw;
        return 130.0 * dot(m, g);
    }
    
    // FBM
    float fbm(vec2 p, int oct) {
        float v = 0.0;
        float a = 0.5;
        float f = 1.0;
        mat2 rot = mat2(0.8, -0.6, 0.6, 0.8);
        for (int i = 0; i < 8; i++) {
            if (i >= oct) break;
            v += a * snoise(p * f);
            a *= 0.5;
            f *= 2.0;
            p = rot * p;
        }
        return v;
    }
    
    // Cloud density function
    float cloudDensity(vec2 uv, float t) {
        // Move clouds with time
        vec2 p = uv;
        p.x += t * uSpeed;
        
        // Domain warping สำหรับ organic shape
        vec2 warp = vec2(
            fbm(p * 0.5 + vec2(1.7, 9.2) + t * 0.05, 4),
            fbm(p * 0.5 + vec2(8.3, 2.8) + t * 0.04, 4)
        );
        p += warp * 0.5;
        
        // Main cloud shape
        float cloud = fbm(p * uScale, uOctaves);
        
        // Coverage threshold
        cloud = cloud - (1.0 - uCoverage);
        cloud = max(0.0, cloud);
        cloud = pow(cloud, uDensity);
        
        return cloud;
    }
    
    // Lighting function สำหรับ clouds
    vec3 cloudLighting(vec2 uv, float density, float t) {
        // Fake light direction
        vec2 lightDir = normalize(vec2(1.0, 0.5));
        
        // Sample density ในทิศ light
        float densityLight = cloudDensity(uv - lightDir * 0.02, t);
        
        // Beer's law: light absorption
        float absorption = exp(-densityLight * uDensity * 2.0);
        
        // Cloud color
        vec3 cloudColor   = vec3(1.0, 0.98, 0.95);  // สีขาวอมเหลืองเล็กน้อย
        vec3 shadowColor  = vec3(0.5, 0.55, 0.7);    // เงาอมน้ำเงิน
        
        return mix(shadowColor, cloudColor, absorption);
    }
    
    void main() {
        vec2 uv = vUv;
        // Correct aspect ratio
        float aspect = float(${innerWidth}) / float(${innerHeight});
        uv.x = (uv.x - 0.5) * aspect + 0.5;
        
        float t = uTime;
        
        // Sky gradient
        vec3 sky = mix(uSkyColorA, uSkyColorB, pow(vUv.y, 0.5));
        
        // Clouds
        float density = cloudDensity(uv, t);
        vec3 cloudColor = cloudLighting(uv, density, t);
        
        // ฟ้าวันขอบฟ้า (horizon haze)
        float horizonFog = 1.0 - smoothstep(0.3, 0.7, vUv.y);
        vec3 horizonColor = vec3(0.7, 0.8, 0.9);
        sky = mix(sky, horizonColor, horizonFog * 0.5);
        
        // รวม sky + clouds
        vec3 color = mix(sky, cloudColor, density);
        
        // Sun glow
        vec2 sunPos = vec2(0.7, 0.8);
        float sunDist = distance(uv, sunPos);
        float sunGlow = exp(-sunDist * sunDist * 20.0);
        color += vec3(1.0, 0.9, 0.5) * sunGlow * 0.5;
        color += vec3(1.0, 0.7, 0.3) * exp(-sunDist * sunDist * 3.0) * 0.15;
        
        // Tone mapping
        color = color / (color + 0.3);
        
        gl_FragColor = vec4(color, 1.0);
    }
`;

const geometry = new THREE.PlaneGeometry(2, 2);
const uniforms = {
    uTime:       { value: 0 },
    uCoverage:   { value: 0.5 },
    uDensity:    { value: 2.0 },
    uSpeed:      { value: 0.3 },
    uScale:      { value: 3.0 },
    uOctaves:    { value: 6 },
    uSkyColorA:  { value: new THREE.Color('#1a3a6e') },
    uSkyColorB:  { value: new THREE.Color('#4a7abf') }
};

const material = new THREE.ShaderMaterial({
    vertexShader: `varying vec2 vUv; void main() { vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
    fragmentShader, uniforms
});

scene.add(new THREE.Mesh(geometry, material));

document.getElementById('coverage').oninput = (e) => uniforms.uCoverage.value = parseFloat(e.target.value);
document.getElementById('density').oninput  = (e) => uniforms.uDensity.value  = parseFloat(e.target.value);
document.getElementById('speed').oninput    = (e) => uniforms.uSpeed.value    = parseFloat(e.target.value);
document.getElementById('scale').oninput    = (e) => uniforms.uScale.value    = parseFloat(e.target.value);
document.getElementById('octaves').oninput  = (e) => uniforms.uOctaves.value  = parseInt(e.target.value);
document.getElementById('skyA').oninput     = (e) => uniforms.uSkyColorA.value.set(e.target.value);
document.getElementById('skyB').oninput     = (e) => uniforms.uSkyColorB.value.set(e.target.value);

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

## Procedural Terrain Noise

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Procedural Terrain</title>
    <style>* { margin: 0; padding: 0; } canvas { display: block; }</style>
</head>
<body>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';
import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.157.0/examples/jsm/controls/OrbitControls.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x87ceeb);
scene.fog = new THREE.FogExp2(0x87ceeb, 0.02);

const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 500);
camera.position.set(0, 8, 20);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.target.set(0, 2, 0);

// Terrain vertex shader
const vertexShader = `
    uniform float uTime;
    uniform float uTerrainScale;
    uniform float uHeightScale;
    
    varying vec3 vWorldPos;
    varying float vHeight;
    varying vec3 vNormal;
    varying vec2 vUv;
    
    // Simplex noise
    vec3 permute(vec3 x) { return mod(((x * 34.0) + 1.0) * x, 289.0); }
    
    float snoise(vec2 v) {
        const vec4 C = vec4(0.211324865405187, 0.366025403784439,
                           -0.577350269189626, 0.024390243902439);
        vec2 i  = floor(v + dot(v, C.yy));
        vec2 x0 = v - i + dot(i, C.xx);
        vec2 i1 = (x0.x > x0.y) ? vec2(1.0, 0.0) : vec2(0.0, 1.0);
        vec4 x12 = x0.xyxy + C.xxzz;
        x12.xy -= i1;
        i = mod(i, 289.0);
        vec3 p = permute(permute(i.y + vec3(0.0, i1.y, 1.0)) + i.x + vec3(0.0, i1.x, 1.0));
        vec3 m = max(0.5 - vec3(dot(x0,x0), dot(x12.xy,x12.xy), dot(x12.zw,x12.zw)), 0.0);
        m = m * m * m * m;
        vec3 x = 2.0 * fract(p * C.www) - 1.0;
        vec3 h = abs(x) - 0.5;
        vec3 a0 = x - floor(x + 0.5);
        m *= 1.79284291400159 - 0.85373472095314 * (a0*a0 + h*h);
        vec3 g;
        g.x = a0.x * x0.x + h.x * x0.y;
        g.yz = a0.yz * x12.xz + h.yz * x12.yw;
        return 130.0 * dot(m, g);
    }
    
    float fbm(vec2 p) {
        float v = 0.0, a = 0.5, f = 1.0;
        mat2 rot = mat2(0.8, -0.6, 0.6, 0.8);
        for (int i = 0; i < 8; i++) {
            v += a * snoise(p * f);
            a *= 0.5; f *= 2.0;
            p = rot * p;
        }
        return v;
    }
    
    // Terrain height function
    float getHeight(vec2 p) {
        float h = fbm(p * uTerrainScale * 0.5);
        // เพิ่ม sharp ridges
        h += fbm(p * uTerrainScale * 1.0) * 0.5;
        h = abs(h); // ridges
        h = pow(h, 1.5);
        return h * uHeightScale;
    }
    
    void main() {
        vUv = uv;
        vec3 pos = position;
        
        // Calculate terrain height
        float height = getHeight(pos.xz);
        pos.y = height;
        
        vHeight = height;
        
        // Calculate normal using finite difference
        float delta = 0.5;
        float hL = getHeight(pos.xz + vec2(-delta, 0));
        float hR = getHeight(pos.xz + vec2(delta, 0));
        float hD = getHeight(pos.xz + vec2(0, -delta));
        float hU = getHeight(pos.xz + vec2(0, delta));
        
        vec3 normal = normalize(vec3(hL - hR, 2.0 * delta, hD - hU));
        vNormal = normalize(normalMatrix * normal);
        
        vec4 worldPos = modelMatrix * vec4(pos, 1.0);
        vWorldPos = worldPos.xyz;
        
        gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
    }
`;

const fragmentShader = `
    precision mediump float;
    
    uniform float uTime;
    
    varying vec3 vWorldPos;
    varying float vHeight;
    varying vec3 vNormal;
    varying vec2 vUv;
    
    void main() {
        float h = vHeight;
        
        // Terrain zones based on height
        vec3 deepWater   = vec3(0.0, 0.1, 0.4);
        vec3 shallowWater= vec3(0.1, 0.4, 0.7);
        vec3 sand        = vec3(0.8, 0.7, 0.4);
        vec3 grass       = vec3(0.2, 0.6, 0.15);
        vec3 darkGrass   = vec3(0.15, 0.45, 0.1);
        vec3 rock        = vec3(0.5, 0.45, 0.4);
        vec3 darkRock    = vec3(0.35, 0.3, 0.28);
        vec3 snow        = vec3(0.95, 0.95, 1.0);
        
        vec3 color;
        
        // Deep water
        if (h < 0.5) {
            color = mix(deepWater, shallowWater, h / 0.5);
        }
        // Beach/Sand
        else if (h < 0.8) {
            color = mix(shallowWater, sand, (h - 0.5) / 0.3);
        }
        // Grass
        else if (h < 2.0) {
            color = mix(sand, grass, smoothstep(0.8, 1.2, h));
        }
        // Dark grass to rock
        else if (h < 3.5) {
            color = mix(darkGrass, rock, smoothstep(2.0, 3.5, h));
        }
        // Rock to dark rock
        else if (h < 5.0) {
            color = mix(rock, darkRock, smoothstep(3.5, 5.0, h));
        }
        // Snow
        else {
            color = mix(darkRock, snow, smoothstep(5.0, 6.0, h));
        }
        
        // Slope-based rock replacement
        float slope = 1.0 - dot(vNormal, vec3(0, 1, 0));
        if (slope > 0.4 && h > 1.0) {
            color = mix(color, rock, smoothstep(0.4, 0.7, slope));
        }
        
        // Lighting
        vec3 lightDir = normalize(vec3(1.0, 2.0, 1.0));
        float diffuse = max(0.0, dot(vNormal, lightDir));
        float ambient = 0.2;
        
        // Shadow in valleys
        float shadow = smoothstep(0.0, 0.3, diffuse);
        
        color *= (ambient + diffuse * 0.8) * shadow;
        
        // Atmospheric fog (built into scene fog)
        
        gl_FragColor = vec4(color, 1.0);
    }
`;

// Large plane สำหรับ terrain
const geometry = new THREE.PlaneGeometry(100, 100, 512, 512);
geometry.rotateX(-Math.PI / 2);

const uniforms = {
    uTime:         { value: 0 },
    uTerrainScale: { value: 0.1 },
    uHeightScale:  { value: 8.0 }
};

const material = new THREE.ShaderMaterial({
    vertexShader,
    fragmentShader,
    uniforms
});

scene.add(new THREE.Mesh(geometry, material));

// Sky sphere
const skyGeom = new THREE.SphereGeometry(400, 16, 16);
const skyMat = new THREE.MeshBasicMaterial({ color: 0x87ceeb, side: THREE.BackSide });
scene.add(new THREE.Mesh(skyGeom, skyMat));

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

## Noise Types Comparison

```
นประเภท    | ลักษณะ                    | ใช้ทำ
-----------|--------------------------|----------------------------
Value Noise| ราบเรียบ, grid artifacts  | heightmaps ทั่วไป
Perlin     | ราบเรียบ, gradient-based  | clouds, fire, terrain
Simplex    | เร็วกว่า Perlin, ดีกว่า   | ทุกอย่างที่ Perlin ทำได้
Voronoi    | แบ่งเป็น cells           | cracked earth, cells
fBm        | Perlin หลาย octaves       | terrain, clouds, marble
Turbulence | abs(Perlin) × octaves     | fire, explosion
Domain Warp| fBm(fBm(p))              | ของเหลว, ควัน, ก้อนเมฆ
```

---

## Periodic Noise

```glsl
// Periodic noise: tile แบบไม่มีรอยต่อ
float periodicNoise(vec2 p, float period) {
    // ใช้ sine/cosine ทำให้ tiling ราบเรียบ
    vec2 q = 2.0 * 3.14159 * p / period;
    
    // Sample 4 octaves ที่ tiling
    return sin(q.x) * cos(q.y) * 0.5 +
           sin(q.x * 2.0) * cos(q.y * 2.0) * 0.25 +
           sin(q.x * 4.0) * cos(q.y * 4.0) * 0.125;
}

// หรือใช้ hash ที่ periodic
float tileNoise(vec2 p, float period) {
    vec2 i = mod(floor(p), period);
    vec2 f = fract(p);
    vec2 u = f * f * (3.0 - 2.0 * f);
    
    // hash ด้วย periodic index
    float a = hash21(mod(i, period));
    float b = hash21(mod(i + vec2(1, 0), period));
    float c = hash21(mod(i + vec2(0, 1), period));
    float d = hash21(mod(i + vec2(1, 1), period));
    
    return mix(mix(a, b, u.x), mix(c, d, u.x), u.y);
}
```

---

## แบบฝึกหัด

### ระดับ 1
1. Implement value noise และแสดงเป็น grayscale image
2. สร้าง fBm ด้วย 4 octaves
3. สร้าง animated noise โดยเพิ่ม time ใน noise input

### ระดับ 2
4. Implement Voronoi noise พร้อม cell coloring
5. สร้าง marble texture จาก fBm + sin
6. สร้าง single-level domain warping

### ระดับ 3
7. สร้าง procedural terrain ใน vertex shader
8. Implement 2-level domain warping (warpedFbm)
9. สร้าง animated fire shader

### Challenge
10. สร้าง Procedural Planet Generator:
    - Terrain generation ด้วย domain-warped fBm
    - Ocean floor / land / mountain zones
    - Cloud layer animation
    - Atmospheric scattering (gradient near horizon)
    - Specular on water
    - Snow caps
    - Procedural crater patterns
