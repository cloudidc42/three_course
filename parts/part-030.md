# Part 30: Animated Shaders

## บทนำ: Animation ใน Shaders

การ animate shaders คือการใช้ `uTime` uniform เพื่อขับเคลื่อน mathematical functions ทำให้เกิดการเคลื่อนไหว ข้อดีหลักคือทุก animation รันบน GPU โดยไม่ต้องอัพเดท geometry เลย

---

## u_time Uniform - หัวใจของ Animation

```javascript
// JavaScript: อัพเดท uTime ทุก frame
const clock = new THREE.Clock();
const uniforms = { uTime: { value: 0 } };

function animate() {
    requestAnimationFrame(animate);
    uniforms.uTime.value = clock.getElapsedTime(); // seconds ตั้งแต่เริ่ม
    renderer.render(scene, camera);
}
```

```glsl
// GLSL: ใช้ uTime สำหรับ animation
uniform float uTime;

void main() {
    float t = uTime;
    
    // Simple oscillation
    float osc = sin(t);              // -1 to 1
    float posOsc = sin(t) * 0.5 + 0.5; // 0 to 1
    
    // Speed control
    float fast = sin(t * 3.0);      // 3x เร็วกว่า
    float slow = sin(t * 0.2);      // 5x ช้ากว่า
    
    // Phase offset
    float phase1 = sin(t + 0.0);
    float phase2 = sin(t + 2.094); // 120° offset
    float phase3 = sin(t + 4.189); // 240° offset
    
    // Triangle wave (ไม่ใช้ abs(sin))
    float tri = abs(sin(t));
    
    // Sawtooth wave
    float saw = fract(t);
    
    // Square wave
    float square = step(0.5, fract(t));
}
```

---

## Wave Animations

```glsl
// Wave Types

// 1. Simple sine wave
float sineWave(float x, float freq, float amp, float speed, float t) {
    return sin(x * freq + t * speed) * amp;
}

// 2. Complex ocean wave
float oceanWave(vec2 p, float t) {
    float w = 0.0;
    
    // Wave 1: ใหญ่ช้า (dominant direction)
    w += sin(p.x * 0.5 + t * 0.8) * 0.15;
    
    // Wave 2: เล็กเร็ว (cross swell)
    w += sin(p.z * 1.0 + t * 1.2) * 0.08;
    
    // Wave 3: ทแยง
    w += sin((p.x + p.z) * 0.7 + t * 0.5) * 0.05;
    
    // Wave 4: microripples (high freq, small amp)
    w += sin(p.x * 3.0 + t * 2.5) * 0.02;
    w += sin(p.z * 2.5 + t * 2.0) * 0.02;
    
    return w;
}

// 3. Standing wave (interference pattern)
float standingWave(vec2 p, float t) {
    return sin(p.x * 3.0 + t) * sin(p.z * 3.0 - t);
}

// 4. Ripple from point
float ripple(vec2 p, vec2 center, float t, float speed) {
    float dist = length(p - center);
    float wave = sin(dist * 10.0 - t * speed);
    float envelope = exp(-dist * 2.0); // ลดลงตามระยะ
    return wave * envelope;
}
```

---

## Pulsing Effects

```glsl
// Pulsing techniques

// 1. Simple pulse
float pulse(float t, float speed) {
    return abs(sin(t * speed));
}

// 2. Soft pulse (smooth in/out)
float softPulse(float t, float speed) {
    float x = sin(t * speed) * 0.5 + 0.5;
    return smoothstep(0.0, 1.0, x) * smoothstep(1.0, 0.0, x) * 4.0;
    // หรือเรียบง่ายกว่า:
    // return sin(t * speed) * 0.5 + 0.5;
}

// 3. Heartbeat pulse
float heartbeat(float t) {
    float beat = fract(t * 1.2); // 1.2 Hz
    float b1 = exp(-beat * 20.0) * 2.0;           // first beat
    float b2 = exp(-(beat - 0.2) * 20.0) * 1.5;  // echo beat
    return (b1 + b2) * exp(-beat * 3.0);
}

// 4. Strobe effect
float strobe(float t, float freq) {
    return step(0.5, fract(t * freq));
}

// 5. Breathing effect (nature-like)
float breathe(float t) {
    // Breathing: inhale fast, pause, exhale slow
    float x = fract(t * 0.3);  // 0.3 breaths per second
    float inhale = smoothstep(0.0, 0.3, x);
    float pause  = smoothstep(0.3, 0.4, x);
    float exhale = 1.0 - smoothstep(0.4, 1.0, x);
    return inhale * pause + exhale * (1.0 - pause);
}
```

---

## Demo 1: Animated Wave Ocean Shader

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Animated Wave Ocean</title>
    <style>
        * { margin: 0; padding: 0; }
        canvas { display: block; }
        #ui {
            position: fixed;
            bottom: 20px;
            left: 20px;
            color: white;
            font-family: monospace;
            background: rgba(0,0,0,0.7);
            padding: 12px;
            border-radius: 8px;
        }
        input { width: 120px; }
    </style>
</head>
<body>
<div id="ui">
    <b>Ocean Controls</b><br>
    Wave Height: <input type="range" id="wh" min="0" max="0.5" value="0.1" step="0.01"><br>
    Wave Speed: <input type="range" id="ws" min="0" max="3" value="1" step="0.1"><br>
    Choppiness: <input type="range" id="ch" min="0" max="1" value="0.3" step="0.01"><br>
    Fog: <input type="range" id="fog" min="0" max="1" value="0.3" step="0.01">
</div>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';
import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.157.0/examples/jsm/controls/OrbitControls.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x87ceeb);
const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 200);
camera.position.set(0, 3, 10);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.target.set(0, 0, 0);

const vertexShader = `
    uniform float uTime;
    uniform float uWaveHeight;
    uniform float uWaveSpeed;
    uniform float uChoppiness;
    
    varying vec3 vWorldPos;
    varying vec3 vNormal;
    varying float vHeight;
    varying vec2 vUv;
    
    float wave(vec2 p, float amp, float freq, float speed, vec2 dir) {
        return amp * sin(dot(dir, p) * freq + uTime * speed);
    }
    
    float getHeight(vec2 p) {
        float h = 0.0;
        float t = uTime * uWaveSpeed;
        
        // Gerstner waves (ocean waves)
        h += wave(p, uWaveHeight, 0.5, 0.8, normalize(vec2(1.0, 0.0)));
        h += wave(p, uWaveHeight * 0.7, 0.8, 1.2, normalize(vec2(0.8, 0.6)));
        h += wave(p, uWaveHeight * 0.5, 1.2, 1.5, normalize(vec2(-0.3, 1.0)));
        h += wave(p, uWaveHeight * 0.3, 2.0, 2.0, normalize(vec2(0.5, -0.5)));
        h += wave(p, uWaveHeight * 0.1, 4.0, 3.0, normalize(vec2(1.0, 0.3)));
        
        // Choppiness (sharper peaks)
        if (uChoppiness > 0.0) {
            h += abs(sin(p.x * 2.0 + uTime * uWaveSpeed)) * uChoppiness * uWaveHeight;
        }
        
        return h;
    }
    
    void main() {
        vUv = uv;
        
        vec3 pos = position;
        float height = getHeight(pos.xz);
        pos.y = height;
        vHeight = height;
        
        // Compute normal
        float d = 0.1;
        float h1 = getHeight(pos.xz + vec2(d, 0));
        float h2 = getHeight(pos.xz - vec2(d, 0));
        float h3 = getHeight(pos.xz + vec2(0, d));
        float h4 = getHeight(pos.xz - vec2(0, d));
        
        vec3 norm = normalize(vec3(h2 - h1, 2.0 * d, h4 - h3));
        vNormal = normalize(normalMatrix * norm);
        
        vec4 worldPos = modelMatrix * vec4(pos, 1.0);
        vWorldPos = worldPos.xyz;
        
        gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
    }
`;

const fragmentShader = `
    precision mediump float;
    
    uniform float uTime;
    uniform float uFog;
    
    varying vec3 vWorldPos;
    varying vec3 vNormal;
    varying float vHeight;
    varying vec2 vUv;
    
    void main() {
        vec3 N = normalize(vNormal);
        vec3 viewDir = normalize(cameraPosition - vWorldPos);
        
        // Fresnel
        float cosTheta = abs(dot(N, viewDir));
        float fresnel = pow(1.0 - cosTheta, 4.0);
        fresnel = clamp(fresnel * 2.0, 0.0, 1.0);
        
        // Water colors
        vec3 deepColor    = vec3(0.01, 0.1, 0.3);
        vec3 shallowColor = vec3(0.05, 0.4, 0.6);
        vec3 skyColor     = vec3(0.5, 0.7, 0.9);
        vec3 sunColor     = vec3(1.0, 0.9, 0.7);
        
        // Depth-based color
        float depth = smoothstep(0.0, 0.5, vHeight + 0.1);
        vec3 waterColor = mix(deepColor, shallowColor, depth);
        
        // Reflection (sky + sun)
        vec3 reflectDir = reflect(-viewDir, N);
        float sunDot = max(0.0, dot(reflectDir, normalize(vec3(1.0, 2.0, 1.0))));
        float sunGlare = pow(sunDot, 256.0) * 5.0;
        
        vec3 reflectedColor = skyColor + sunColor * sunGlare;
        
        // Mix water + reflection via fresnel
        vec3 color = mix(waterColor, reflectedColor, fresnel);
        
        // Foam on crests
        float foam = smoothstep(0.05, 0.1, vHeight) * 0.5;
        color = mix(color, vec3(1.0), foam);
        
        // Underwater caustics pattern
        float caustics = sin(vWorldPos.x * 5.0 + uTime) * sin(vWorldPos.z * 5.0 + uTime * 0.7);
        caustics = caustics * 0.5 + 0.5;
        caustics = pow(caustics, 4.0);
        color += caustics * 0.05 * vec3(0.5, 0.8, 1.0);
        
        // Fog
        float dist = length(vWorldPos - cameraPosition);
        float fogFactor = exp(-dist * uFog * 0.05);
        vec3 fogColor = vec3(0.8, 0.9, 1.0);
        color = mix(fogColor, color, fogFactor);
        
        gl_FragColor = vec4(color, 0.9);
    }
`;

const geometry = new THREE.PlaneGeometry(60, 60, 256, 256);
geometry.rotateX(-Math.PI / 2);

const uniforms = {
    uTime:       { value: 0 },
    uWaveHeight: { value: 0.1 },
    uWaveSpeed:  { value: 1.0 },
    uChoppiness: { value: 0.3 },
    uFog:        { value: 0.3 }
};

const material = new THREE.ShaderMaterial({
    vertexShader, fragmentShader, uniforms,
    transparent: true, side: THREE.FrontSide
});

scene.add(new THREE.Mesh(geometry, material));

// Sky sphere
const sky = new THREE.Mesh(
    new THREE.SphereGeometry(150, 16, 16),
    new THREE.MeshBasicMaterial({ color: 0x87ceeb, side: THREE.BackSide })
);
scene.add(sky);

document.getElementById('wh').oninput = e => uniforms.uWaveHeight.value = parseFloat(e.target.value);
document.getElementById('ws').oninput = e => uniforms.uWaveSpeed.value  = parseFloat(e.target.value);
document.getElementById('ch').oninput = e => uniforms.uChoppiness.value = parseFloat(e.target.value);
document.getElementById('fog').oninput = e => uniforms.uFog.value       = parseFloat(e.target.value);

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

## Demo 2: Particle Flow Shader

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Particle Flow Shader</title>
    <style>* { margin: 0; padding: 0; } canvas { display: block; }</style>
</head>
<body>
<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js';
import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.157.0/examples/jsm/controls/OrbitControls.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x000005);
const camera = new THREE.PerspectiveCamera(75, innerWidth / innerHeight, 0.1, 100);
camera.position.set(0, 0, 8);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
document.body.appendChild(renderer.domElement);
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

const vertexShader = `
    attribute float aPhase;
    attribute float aSpeed;
    attribute float aRadius;
    attribute float aLife;
    attribute vec3  aColor;
    
    uniform float uTime;
    uniform float uPixelRatio;
    
    varying vec3  vColor;
    varying float vAlpha;
    varying float vLife;
    
    // Flow field - curl noise-like
    vec3 flowField(vec3 p, float t) {
        float s1 = sin(p.x * 1.5 + t * 0.5);
        float c1 = cos(p.y * 1.2 + t * 0.4);
        float s2 = sin(p.z * 1.0 + t * 0.6);
        
        return vec3(
            s1 * c1 * 0.5,
            sin(p.x * 0.8 + t * 0.3) * cos(p.z * 0.7) * 0.5,
            c1 * s2 * 0.5
        );
    }
    
    void main() {
        vColor = aColor;
        vLife  = aLife;
        
        float t = uTime * aSpeed + aPhase * 6.28318;
        
        // Animate position
        vec3 pos = position;
        
        // Orbit motion
        float orbitAngle = t;
        float orbitRadius = aRadius;
        
        // Flow field displacement
        vec3 flow = flowField(pos * 0.5, uTime * 0.2);
        pos += flow * 0.8;
        
        // Circular motion around center
        float cy = pos.y;
        float cx = cos(orbitAngle) * orbitRadius;
        float cz = sin(orbitAngle) * orbitRadius;
        
        pos.x += cx * 0.3;
        pos.z += cz * 0.3;
        
        // Life-based alpha
        vAlpha = sin(aLife * 3.14159) * (0.4 + aSpeed * 0.3);
        
        // Project
        vec4 mvPos = modelViewMatrix * vec4(pos, 1.0);
        gl_PointSize = 3.0 * uPixelRatio * (5.0 / -mvPos.z);
        gl_Position  = projectionMatrix * mvPos;
    }
`;

const fragmentShader = `
    precision mediump float;
    
    varying vec3  vColor;
    varying float vAlpha;
    varying float vLife;
    
    void main() {
        vec2 uv = gl_PointCoord - 0.5;
        float dist = length(uv);
        
        // Circular point
        float circle = 1.0 - smoothstep(0.3, 0.5, dist);
        
        // Core glow
        float core = exp(-dist * dist * 20.0);
        
        vec3 color = vColor + core * 0.5;
        float alpha = circle * vAlpha * core;
        
        if (alpha < 0.01) discard;
        
        gl_FragColor = vec4(color, alpha);
    }
`;

// Particle system
const count = 15000;
const positions = new Float32Array(count * 3);
const aPhase  = new Float32Array(count);
const aSpeed  = new Float32Array(count);
const aRadius = new Float32Array(count);
const aLife   = new Float32Array(count);
const aColor  = new Float32Array(count * 3);

// Color palettes
const palettes = [
    [0.3, 0.7, 1.0],  // น้ำเงิน
    [0.2, 1.0, 0.5],  // เขียว
    [1.0, 0.4, 0.2],  // ส้ม
    [0.8, 0.2, 1.0],  // ม่วง
    [1.0, 0.9, 0.2],  // เหลือง
];

for (let i = 0; i < count; i++) {
    const i3 = i * 3;
    
    // Distribute particles in sphere
    const theta = Math.random() * Math.PI * 2;
    const phi   = Math.acos(2 * Math.random() - 1);
    const r     = 1.5 + Math.random() * 1.5;
    
    positions[i3]     = r * Math.sin(phi) * Math.cos(theta);
    positions[i3 + 1] = r * Math.sin(phi) * Math.sin(theta);
    positions[i3 + 2] = r * Math.cos(phi);
    
    aPhase[i]  = Math.random();
    aSpeed[i]  = 0.3 + Math.random() * 0.7;
    aRadius[i] = 0.1 + Math.random() * 0.3;
    aLife[i]   = Math.random();
    
    const pal = palettes[Math.floor(Math.random() * palettes.length)];
    const bright = 0.6 + Math.random() * 0.4;
    aColor[i3]     = pal[0] * bright;
    aColor[i3 + 1] = pal[1] * bright;
    aColor[i3 + 2] = pal[2] * bright;
}

const geometry = new THREE.BufferGeometry();
geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
geometry.setAttribute('aPhase',   new THREE.BufferAttribute(aPhase, 1));
geometry.setAttribute('aSpeed',   new THREE.BufferAttribute(aSpeed, 1));
geometry.setAttribute('aRadius',  new THREE.BufferAttribute(aRadius, 1));
geometry.setAttribute('aLife',    new THREE.BufferAttribute(aLife, 1));
geometry.setAttribute('aColor',   new THREE.BufferAttribute(aColor, 3));

const material = new THREE.ShaderMaterial({
    vertexShader, fragmentShader,
    uniforms: {
        uTime:       { value: 0 },
        uPixelRatio: { value: Math.min(devicePixelRatio, 2) }
    },
    transparent: true,
    blending: THREE.AdditiveBlending,
    depthWrite: false
});

scene.add(new THREE.Points(geometry, material));

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

## Demo 3: Morphing Logo Shader (SDF-based)

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Morphing Logo Shader</title>
    <style>* { margin: 0; padding: 0; } canvas { display: block; }</style>
</head>
<body>
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
    uniform vec2  uResolution;
    
    varying vec2 vUv;
    
    // === SDF Shapes ===
    float sdCircle(vec2 p, float r) { return length(p) - r; }
    
    float sdBox(vec2 p, vec2 b) {
        vec2 d = abs(p) - b;
        return length(max(d, 0.0)) + min(max(d.x, d.y), 0.0);
    }
    
    float sdTriangle(vec2 p, float r) {
        const float k = sqrt(3.0);
        p.x = abs(p.x) - r;
        p.y = p.y + r / k;
        if (p.x + k * p.y > 0.0)
            p = vec2(p.x - k * p.y, -k * p.x - p.y) / 2.0;
        p.x -= clamp(p.x, -2.0 * r, 0.0);
        return -length(p) * sign(p.y);
    }
    
    float sdRing(vec2 p, float r, float t) {
        return abs(length(p) - r) - t;
    }
    
    float sdStar(vec2 p, float r) {
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
    
    // Smooth min (organic blending)
    float smin(float a, float b, float k) {
        float h = max(k - abs(a - b), 0.0) / k;
        return min(a, b) - h * h * k * 0.25;
    }
    
    // สร้าง SDF จากเวลา (morphing)
    float getMorphedSDF(vec2 p, float t) {
        float cycle = mod(t * 0.3, 5.0);
        
        float s0 = sdCircle(p, 0.28);
        float s1 = sdBox(p, vec2(0.22, 0.22));
        float s2 = sdTriangle(p, 0.28);
        float s3 = sdRing(p, 0.24, 0.05);
        float s4 = sdStar(p, 0.3);
        
        float sdf;
        float progress = fract(cycle);
        progress = smoothstep(0.2, 0.8, progress);
        
        if (cycle < 1.0)      sdf = mix(s0, s1, progress);
        else if (cycle < 2.0) sdf = mix(s1, s2, progress);
        else if (cycle < 3.0) sdf = mix(s2, s3, progress);
        else if (cycle < 4.0) sdf = mix(s3, s4, progress);
        else                   sdf = mix(s4, s0, progress);
        
        return sdf;
    }
    
    // Cosine palette
    vec3 palette(float t) {
        return vec3(0.5) + vec3(0.5) * cos(6.28318 * (
            vec3(1.0, 1.0, 0.5) * t + vec3(0.0, 0.33, 0.67)
        ));
    }
    
    void main() {
        // Center and correct aspect
        vec2 aspect = vec2(uResolution.x / uResolution.y, 1.0);
        vec2 p = (vUv - 0.5) * aspect;
        
        float t = uTime;
        
        // Main shape
        float sdf = getMorphedSDF(p, t);
        
        // Rotation
        float angle = t * 0.2;
        vec2 rp = vec2(
            p.x * cos(angle) - p.y * sin(angle),
            p.x * sin(angle) + p.y * cos(angle)
        );
        
        // Distance rings
        float rings = sin(sdf * 30.0 - t * 2.0) * 0.5 + 0.5;
        
        // Fill
        float fill = smoothstep(0.005, -0.005, sdf);
        
        // Outline layers
        float outline1 = smoothstep(0.015, 0.01, abs(sdf));
        float outline2 = smoothstep(0.03, 0.025, abs(sdf - 0.03));
        float outline3 = smoothstep(0.05, 0.045, abs(sdf - 0.06));
        
        // Glow
        float glow = exp(-abs(sdf) * 15.0);
        
        // Color
        vec3 baseColor = palette(t * 0.1 + sdf * 2.0);
        vec3 glowColor = palette(t * 0.1 + 0.5);
        
        vec3 color = vec3(0.02, 0.02, 0.05); // background
        
        // Apply layers
        color = mix(color, baseColor * rings, fill);
        color += glowColor * glow * 0.4;
        color = mix(color, vec3(1.0), outline1 * 0.8);
        color = mix(color, vec3(0.5, 0.8, 1.0), outline2 * 0.6);
        color = mix(color, vec3(0.3, 0.5, 0.8), outline3 * 0.4);
        
        // Vignette
        float vignette = 1.0 - length(p) * 1.2;
        color *= vignette;
        
        gl_FragColor = vec4(color, 1.0);
    }
`;

const geometry = new THREE.PlaneGeometry(2, 2);
const uniforms = {
    uTime:       { value: 0 },
    uResolution: { value: new THREE.Vector2(innerWidth, innerHeight) }
};

const material = new THREE.ShaderMaterial({
    vertexShader: `varying vec2 vUv; void main() { vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
    fragmentShader, uniforms
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

## Rotating Patterns

```glsl
// 1. Rotating spiral
vec3 rotatingSpiral(vec2 p, float t) {
    float r = length(p);
    float a = atan(p.y, p.x);
    float spiral = sin(r * 10.0 - a * 3.0 + t * 2.0);
    return vec3(spiral * 0.5 + 0.5);
}

// 2. Clock-like rotation
vec3 clockPattern(vec2 p, float t) {
    float a = atan(p.y, p.x);
    float r = length(p);
    
    // Seconds hand
    float sec = mod(t, 60.0) / 60.0 * 6.28318;
    float secHand = smoothstep(0.005, 0.0, abs(a - sec)) * step(r, 0.4);
    
    // Minutes hand
    float min = mod(t / 60.0, 60.0) / 60.0 * 6.28318;
    float minHand = smoothstep(0.007, 0.0, abs(a - min)) * step(r, 0.3);
    
    return vec3(secHand * 0.8 + minHand * 0.5 + 0.05);
}

// 3. Lissajous curve
vec2 lissajous(float t, float a, float b, float delta) {
    return vec2(sin(a * t + delta), sin(b * t));
}

// สร้าง Lissajous animation
float lissajousPattern(vec2 p, float t) {
    float thickness = 0.015;
    float result = 0.0;
    
    // วาด Lissajous curve เป็น line
    for (int i = 0; i < 200; i++) {
        float u = float(i) / 200.0 * 6.28318 * 2.0;
        vec2 lp = lissajous(u, 3.0, 4.0, t);
        float dist = length(p - lp * 0.4);
        result = max(result, smoothstep(thickness, 0.0, dist));
    }
    
    return result;
}
```

---

## Shader-based Fireworks

```glsl
// Fireworks simulation ใน fragment shader

// Firework burst
float burst(vec2 p, vec2 center, float t, int numRays, float speed) {
    vec2 d = p - center;
    float angle = atan(d.y, d.x);
    float dist = length(d);
    
    float result = 0.0;
    for (int i = 0; i < 16; i++) {
        if (i >= numRays) break;
        
        float rayAngle = float(i) * 6.28318 / float(numRays);
        float angleDiff = abs(mod(angle - rayAngle + 3.14159, 6.28318) - 3.14159);
        float rayWidth = 0.05;
        float onRay = smoothstep(rayWidth, 0.0, angleDiff);
        
        float expectedDist = t * speed;
        float trailFade = exp(-abs(dist - expectedDist) * 5.0);
        
        result = max(result, onRay * trailFade * exp(-t * 0.5));
    }
    
    return result;
}

void main() {
    vec2 p = vUv * 2.0 - 1.0;
    
    float t = uTime;
    
    // หลาย fireworks
    float fw = 0.0;
    
    // Firework 1
    float t1 = mod(t * 0.4, 3.0);
    fw += burst(p, vec2(0.2, 0.3), t1, 12, 0.8);
    
    // Firework 2
    float t2 = mod(t * 0.4 + 1.0, 3.0);
    fw += burst(p, vec2(-0.3, 0.1), t2, 8, 1.0);
    
    // Firework 3
    float t3 = mod(t * 0.4 + 2.0, 3.0);
    fw += burst(p, vec2(0.0, 0.5), t3, 16, 0.6);
    
    // Color palette per firework
    vec3 c1 = vec3(1.0, 0.4, 0.2);
    vec3 c2 = vec3(0.3, 0.8, 1.0);
    vec3 c3 = vec3(1.0, 0.9, 0.2);
    
    vec3 color = fw * mix(c1, c2, sin(t) * 0.5 + 0.5);
    
    gl_FragColor = vec4(color, 1.0);
}
```

---

## แบบฝึกหัด

### ระดับ 1
1. สร้าง uniform uTime และทำให้สีเปลี่ยนตามเวลา
2. สร้าง sine wave animation บน plane (vertex shader)
3. สร้าง pulsing circle ใน fragment shader

### ระดับ 2
4. สร้าง ocean wave ด้วย Gerstner waves
5. สร้าง Lissajous curve animation
6. สร้าง heartbeat effect

### ระดับ 3
7. สร้าง particle system ด้วย flow field
8. สร้าง morphing SDF animation (5 shapes)
9. สร้าง fireworks display

### Challenge
10. สร้าง "Shader Clock" - นาฬิกาจาก scratch ด้วย shader:
    - วงแหวนนาฬิกา
    - เข็มวินาที, นาที, ชั่วโมง (แม่นยำตามเวลาจริง)
    - Number markers ด้วย SDF
    - Animated glow effects
    - Tick marks
    - Background pattern
