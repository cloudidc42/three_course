# Part 25: Fragment Shaders Deep Dive

## บทนำ: Fragment Shader คืออะไร?

Fragment shader (บางครั้งเรียกว่า pixel shader) คือโปรแกรมที่รันสำหรับทุก **fragment** ที่ถูก rasterize มาจาก geometry Fragment คือ "candidate pixel" ที่อาจถูกวาดลงบนหน้าจอ (หรืออาจถูก discard ก็ได้)

```
Vertex Shader Output (gl_Position)
    ↓
Rasterization (GPU สร้าง fragments จาก triangles)
    ↓
Fragment Shader (รันทุก fragment)
    ↓
Depth Test, Stencil Test, Alpha Blending
    ↓
Framebuffer (หน้าจอ)
```

Fragment shader รับ:
- **varyings** จาก vertex shader (ถูก interpolate แล้ว)
- **uniforms** (เหมือนกันทุก fragment)
- **gl_FragCoord** (ตำแหน่งบนหน้าจอ)
- **gl_PointCoord** (เฉพาะ Points, UV ภายใน point)

Fragment shader ส่งออก:
- **gl_FragColor** (สีของ pixel)
- **discard** (ยกเลิก fragment นี้ ไม่วาดอะไร)

---

## Fragment Shader Precision

```glsl
// precision qualifier สำหรับ fragment shader
// (vertex shader มี default เป็น highp)

precision lowp float;    // ~8-bit, ±2 range, 3 decimal digits
precision mediump float; // ~16-bit, ±214 range, 5 decimal digits (ใช้บ่อยสุด)
precision highp float;   // ~32-bit, ±214748 range, 7+ decimal digits

// ใช้ global precision
precision mediump float;

// หรือระบุต่อตัวแปร
lowp vec3 color;
highp vec3 position;
mediump float time;

// แนะนำ:
// - mediump float สำหรับ color, UV
// - highp float สำหรับ world position, depth calculations
```

---

## Color Manipulation

```glsl
precision mediump float;

uniform float uTime;
varying vec2 vUv;

// ฟังก์ชัน: แปลง RGB → HSL
vec3 rgb2hsl(vec3 c) {
    float maxC = max(c.r, max(c.g, c.b));
    float minC = min(c.r, min(c.g, c.b));
    float delta = maxC - minC;
    
    float h = 0.0;
    float s = 0.0;
    float l = (maxC + minC) * 0.5;
    
    if (delta > 0.0) {
        s = delta / (1.0 - abs(2.0 * l - 1.0));
        
        if (maxC == c.r) {
            h = mod((c.g - c.b) / delta, 6.0) / 6.0;
        } else if (maxC == c.g) {
            h = ((c.b - c.r) / delta + 2.0) / 6.0;
        } else {
            h = ((c.r - c.g) / delta + 4.0) / 6.0;
        }
    }
    
    return vec3(h, s, l);
}

// ฟังก์ชัน: HUE ไป RGB helper
float hue2rgb(float p, float q, float t) {
    if (t < 0.0) t += 1.0;
    if (t > 1.0) t -= 1.0;
    if (t < 1.0/6.0) return p + (q - p) * 6.0 * t;
    if (t < 1.0/2.0) return q;
    if (t < 2.0/3.0) return p + (q - p) * (2.0/3.0 - t) * 6.0;
    return p;
}

// ฟังก์ชัน: HSL → RGB
vec3 hsl2rgb(vec3 c) {
    float h = c.x, s = c.y, l = c.z;
    
    if (s == 0.0) return vec3(l);
    
    float q = l < 0.5 ? l * (1.0 + s) : l + s - l * s;
    float p = 2.0 * l - q;
    
    return vec3(
        hue2rgb(p, q, h + 1.0/3.0),
        hue2rgb(p, q, h),
        hue2rgb(p, q, h - 1.0/3.0)
    );
}

// Gamma correction
vec3 linearToSrgb(vec3 c) {
    return pow(c, vec3(1.0/2.2));
}

vec3 srgbToLinear(vec3 c) {
    return pow(c, vec3(2.2));
}

// Brightness, Contrast, Saturation
vec3 adjustColor(vec3 c, float brightness, float contrast, float saturation) {
    // Brightness
    c *= brightness;
    
    // Contrast
    c = (c - 0.5) * contrast + 0.5;
    
    // Saturation (ผ่าน luminance)
    float luminance = dot(c, vec3(0.299, 0.587, 0.114));
    c = mix(vec3(luminance), c, saturation);
    
    return clamp(c, 0.0, 1.0);
}
```

---

## UV-Based Patterns

```glsl
precision mediump float;
varying vec2 vUv;
uniform float uTime;

// Pattern 1: Grid
float grid(vec2 uv, float size) {
    vec2 g = fract(uv * size);
    float lineX = step(0.95, g.x);
    float lineY = step(0.95, g.y);
    return max(lineX, lineY);
}

// Pattern 2: Dots
float dots(vec2 uv, float size, float radius) {
    vec2 g = fract(uv * size) - 0.5;
    return 1.0 - smoothstep(radius - 0.01, radius + 0.01, length(g));
}

// Pattern 3: Checkerboard
float checker(vec2 uv, float size) {
    vec2 g = floor(uv * size);
    return mod(g.x + g.y, 2.0);
}

// Pattern 4: Stripe
float stripe(vec2 uv, float freq, float thickness) {
    float t = fract(uv.x * freq);
    return step(1.0 - thickness, t);
}

// Pattern 5: Hexagonal
float hexagon(vec2 uv) {
    uv = abs(uv);
    return max(uv.x * 0.866025 + uv.y * 0.5, uv.y);
}

float hexGrid(vec2 uv, float size) {
    vec2 s = vec2(1.0, 1.732);
    vec2 p = fract(uv * size / s) * s - s * 0.5;
    vec2 a = mod(p * s, s) - s * 0.5;
    vec2 b = mod(p * s + s * 0.5, s) - s * 0.5;
    return 0.5 - min(hexagon(a), hexagon(b));
}

void main() {
    float t = uTime * 0.1;
    vec2 uv = vUv + vec2(t, 0.0); // เลื่อน UV ตามเวลา
    
    float g = grid(uv, 10.0);
    float d = dots(vUv, 8.0, 0.3);
    float c = checker(vUv, 5.0);
    float hex = hexGrid(vUv - 0.5, 3.0);
    
    // เลือกแสดง pattern
    vec3 gridColor  = vec3(g) * vec3(0.3, 0.7, 1.0);
    vec3 dotColor   = vec3(d) * vec3(1.0, 0.5, 0.2);
    vec3 checkColor = mix(vec3(0.1), vec3(0.9), c);
    vec3 hexColor   = vec3(smoothstep(0.0, 0.05, hex)) * vec3(0.2, 0.8, 0.5);
    
    gl_FragColor = vec4(hexColor, 1.0);
}
```

---

## Noise Patterns

### Random (Hash Function)
```glsl
// Hash function: pseudo-random value จาก input
float hash11(float p) {
    p = fract(p * 0.1031);
    p *= p + 33.33;
    p *= p + p;
    return fract(p);
}

float hash21(vec2 p) {
    vec3 p3 = fract(vec3(p.xyx) * 0.1031);
    p3 += dot(p3, p3.yzx + 33.33);
    return fract((p3.x + p3.y) * p3.z);
}

vec2 hash22(vec2 p) {
    vec3 p3 = fract(vec3(p.xyx) * vec3(0.1031, 0.1030, 0.0973));
    p3 += dot(p3, p3.yzx + 33.33);
    return fract((p3.xx + p3.yz) * p3.zy);
}

vec3 hash33(vec3 p3) {
    p3 = fract(p3 * vec3(0.1031, 0.1030, 0.0973));
    p3 += dot(p3, p3.yxz + 33.33);
    return fract((p3.xxy + p3.yxx) * p3.zyx);
}
```

### Value Noise
```glsl
// Value Noise: random values at grid points, interpolated smoothly
float valueNoise(vec2 p) {
    vec2 i = floor(p);
    vec2 f = fract(p);
    
    // Smooth interpolation curve: 6t^5 - 15t^4 + 10t^3
    vec2 u = f * f * f * (f * (f * 6.0 - 15.0) + 10.0);
    
    // Corner values
    float a = hash21(i);
    float b = hash21(i + vec2(1.0, 0.0));
    float c = hash21(i + vec2(0.0, 1.0));
    float d = hash21(i + vec2(1.0, 1.0));
    
    // Bilinear interpolation
    return mix(mix(a, b, u.x), mix(c, d, u.x), u.y);
}
```

### Gradient Noise (Perlin-like)
```glsl
// Gradient Noise: ใช้ gradient vectors แทน random values
vec2 gradients[8];

vec2 gradient(vec2 p) {
    float angle = hash21(p) * 6.28318;
    return vec2(cos(angle), sin(angle));
}

float gradientNoise(vec2 p) {
    vec2 i = floor(p);
    vec2 f = fract(p);
    
    vec2 u = f * f * f * (f * (f * 6.0 - 15.0) + 10.0);
    
    vec2 ga = gradient(i + vec2(0.0, 0.0));
    vec2 gb = gradient(i + vec2(1.0, 0.0));
    vec2 gc = gradient(i + vec2(0.0, 1.0));
    vec2 gd = gradient(i + vec2(1.0, 1.0));
    
    float va = dot(ga, f - vec2(0.0, 0.0));
    float vb = dot(gb, f - vec2(1.0, 0.0));
    float vc = dot(gc, f - vec2(0.0, 1.0));
    float vd = dot(gd, f - vec2(1.0, 1.0));
    
    return mix(mix(va, vb, u.x), mix(vc, vd, u.x), u.y);
}
```

### Voronoi / Cellular Noise
```glsl
// Voronoi: กระจาย feature points, แต่ละ pixel เป็นของ cell ที่ใกล้ที่สุด
vec2 voronoiCellPoint(vec2 cell) {
    return cell + hash22(cell); // jitter center point
}

// ส่งคืน: x = distance ถึง cell border, y = distance ถึง nearest center
vec2 voronoi(vec2 p) {
    vec2 i = floor(p);
    vec2 f = fract(p);
    
    float minDist1 = 1e10; // ใกล้ที่สุด
    float minDist2 = 1e10; // ใกล้อันดับ 2
    vec2 minCell;
    
    for (int y = -2; y <= 2; y++) {
        for (int x = -2; x <= 2; x++) {
            vec2 neighbor = vec2(float(x), float(y));
            vec2 cellPos = voronoiCellPoint(i + neighbor);
            vec2 diff = neighbor + cellPos - f;
            float dist = dot(diff, diff); // squared distance (faster)
            
            if (dist < minDist1) {
                minDist2 = minDist1;
                minDist1 = dist;
                minCell = cellPos;
            } else if (dist < minDist2) {
                minDist2 = dist;
            }
        }
    }
    
    return vec2(sqrt(minDist1), sqrt(minDist2));
}
```

---

## SDF (Signed Distance Fields) Basics

SDF คือฟังก์ชันที่ส่งคืน **ระยะห่าง** จากจุดไปยังขอบของ shape (บวก = นอก shape, ลบ = ใน shape)

```glsl
// SDF พื้นฐาน

// Circle SDF
float sdCircle(vec2 p, float r) {
    return length(p) - r;
}

// Box/Rectangle SDF
float sdBox(vec2 p, vec2 b) {
    vec2 d = abs(p) - b;
    return length(max(d, 0.0)) + min(max(d.x, d.y), 0.0);
}

// Line Segment SDF
float sdSegment(vec2 p, vec2 a, vec2 b) {
    vec2 pa = p - a;
    vec2 ba = b - a;
    float h = clamp(dot(pa, ba) / dot(ba, ba), 0.0, 1.0);
    return length(pa - ba * h);
}

// Rounded Box SDF
float sdRoundedBox(vec2 p, vec2 b, float r) {
    vec2 d = abs(p) - b + r;
    return length(max(d, 0.0)) + min(max(d.x, d.y), 0.0) - r;
}

// Triangle SDF
float sdTriangle(vec2 p, vec2 a, vec2 b, vec2 c) {
    vec2 e0 = b - a, e1 = c - b, e2 = a - c;
    vec2 v0 = p - a, v1 = p - b, v2 = p - c;
    
    vec2 pq0 = v0 - e0 * clamp(dot(v0, e0) / dot(e0, e0), 0.0, 1.0);
    vec2 pq1 = v1 - e1 * clamp(dot(v1, e1) / dot(e1, e1), 0.0, 1.0);
    vec2 pq2 = v2 - e2 * clamp(dot(v2, e2) / dot(e2, e2), 0.0, 1.0);
    
    float s = sign(e0.x * e2.y - e0.y * e2.x);
    vec2 d = min(min(
        vec2(dot(pq0, pq0), s * (v0.x * e0.y - v0.y * e0.x)),
        vec2(dot(pq1, pq1), s * (v1.x * e1.y - v1.y * e1.x))),
        vec2(dot(pq2, pq2), s * (v2.x * e2.y - v2.y * e2.x)));
    
    return -sqrt(d.x) * sign(d.y);
}

// Star SDF
float sdStar(vec2 p, float r, int n, float m) {
    float an = 3.141593 / float(n);
    float en = 3.141593 / m;
    vec2 acs = vec2(cos(an), sin(an));
    vec2 ecs = vec2(cos(en), sin(en));
    
    float bn = mod(atan(p.y, p.x), 2.0 * an) - an;
    p = length(p) * vec2(cos(bn), abs(sin(bn)));
    p -= r * acs;
    p += ecs * clamp(-dot(p, ecs), 0.0, r * acs.y / ecs.y);
    
    return length(p) * sign(p.x);
}
```

### SDF Operations (Boolean)
```glsl
// Union: รวม shapes (min = ใกล้ขอบน้อยสุด)
float sdfUnion(float d1, float d2) {
    return min(d1, d2);
}

// Intersection: จุดที่อยู่ใน shapes ทั้งสอง
float sdfIntersection(float d1, float d2) {
    return max(d1, d2);
}

// Difference: ตัดส่วนหนึ่งออกจากอีกส่วน
float sdfDifference(float d1, float d2) {
    return max(d1, -d2);
}

// Smooth Union: รวมแบบ smooth edges
float sdfSmoothUnion(float d1, float d2, float k) {
    float h = clamp(0.5 + 0.5 * (d2 - d1) / k, 0.0, 1.0);
    return mix(d2, d1, h) - k * h * (1.0 - h);
}

// ตัวอย่างการใช้
float shape1 = sdCircle(p - vec2(0.3, 0.0), 0.3);
float shape2 = sdBox(p + vec2(0.3, 0.0), vec2(0.2, 0.2));

float combined  = sdfUnion(shape1, shape2);
float intersect = sdfIntersection(shape1, shape2);
float diff      = sdfDifference(shape1, shape2);
float smooth    = sdfSmoothUnion(shape1, shape2, 0.1);
```

### Rendering SDF
```glsl
// แปลง SDF distance เป็น pixel color

// วิธี 1: Hard edge
float hardEdge = step(0.0, -sdf); // 1.0 ถ้าอยู่ใน shape, 0.0 ถ้าอยู่นอก

// วิธี 2: Anti-aliased edge
float aaEdge = smoothstep(0.0, 0.01, -sdf);

// วิธี 3: Glow effect
float glow = exp(-abs(sdf) * 20.0);

// วิธี 4: Outline
float outline = smoothstep(0.01, 0.0, abs(sdf) - 0.02);

// วิธี 5: Fill + Outline
float fill    = smoothstep(0.0, 0.01, -sdf);
float border  = smoothstep(0.04, 0.03, abs(sdf));
float shape   = fill + border * (1.0 - fill); // outline บน fill
```

---

## Demo 1: Pattern Gallery Shader

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Pattern Gallery Shader</title>
    <style>
        * { margin: 0; padding: 0; }
        canvas { display: block; }
        #controls {
            position: fixed;
            top: 20px;
            right: 20px;
            color: white;
            font-family: monospace;
            background: rgba(0,0,0,0.8);
            padding: 15px;
            border-radius: 8px;
        }
        button {
            display: block;
            margin: 4px 0;
            padding: 6px 12px;
            background: #2a2a4a;
            color: white;
            border: 1px solid #4a4a8a;
            border-radius: 4px;
            cursor: pointer;
            width: 100%;
        }
        button.active { background: #4a4aaa; }
    </style>
</head>
<body>
<div id="controls">
    <div>Pattern Gallery</div>
    <hr>
    <button id="p0" class="active">Grid</button>
    <button id="p1">Dots</button>
    <button id="p2">Checker</button>
    <button id="p3">Hex</button>
    <button id="p4">Voronoi</button>
    <button id="p5">Value Noise</button>
    <button id="p6">Gradient Noise</button>
    <button id="p7">Animated</button>
    <hr>
    <label>Scale: <input type="range" id="scale" min="1" max="20" value="5"></label>
    <label>Animate: <input type="checkbox" id="animate"></label>
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
    precision mediump float;
    
    uniform float uTime;
    uniform int   uPattern;
    uniform float uScale;
    uniform bool  uAnimate;
    
    varying vec2 vUv;
    
    // --- Hash Functions ---
    float hash11(float p) {
        p = fract(p * 0.1031);
        p *= p + 33.33;
        p *= p + p;
        return fract(p);
    }
    
    float hash21(vec2 p) {
        vec3 p3 = fract(vec3(p.xyx) * 0.1031);
        p3 += dot(p3, p3.yzx + 33.33);
        return fract((p3.x + p3.y) * p3.z);
    }
    
    vec2 hash22(vec2 p) {
        vec3 p3 = fract(vec3(p.xyx) * vec3(0.1031, 0.1030, 0.0973));
        p3 += dot(p3, p3.yzx + 33.33);
        return fract((p3.xx + p3.yz) * p3.zy);
    }
    
    // --- Value Noise ---
    float valueNoise(vec2 p) {
        vec2 i = floor(p);
        vec2 f = fract(p);
        vec2 u = f * f * f * (f * (f * 6.0 - 15.0) + 10.0);
        
        float a = hash21(i);
        float b = hash21(i + vec2(1.0, 0.0));
        float c = hash21(i + vec2(0.0, 1.0));
        float d = hash21(i + vec2(1.0, 1.0));
        
        return mix(mix(a, b, u.x), mix(c, d, u.x), u.y);
    }
    
    // --- Gradient Noise ---
    vec2 gradVec(vec2 p) {
        float angle = hash21(p) * 6.28318;
        return vec2(cos(angle), sin(angle));
    }
    
    float gradNoise(vec2 p) {
        vec2 i = floor(p);
        vec2 f = fract(p);
        vec2 u = f * f * f * (f * (f * 6.0 - 15.0) + 10.0);
        
        float va = dot(gradVec(i + vec2(0,0)), f - vec2(0,0));
        float vb = dot(gradVec(i + vec2(1,0)), f - vec2(1,0));
        float vc = dot(gradVec(i + vec2(0,1)), f - vec2(0,1));
        float vd = dot(gradVec(i + vec2(1,1)), f - vec2(1,1));
        
        return mix(mix(va, vb, u.x), mix(vc, vd, u.x), u.y) * 0.5 + 0.5;
    }
    
    // --- Voronoi ---
    vec2 voronoi(vec2 p) {
        vec2 i = floor(p);
        vec2 f = fract(p);
        float d1 = 1e10, d2 = 1e10;
        
        for (int y = -2; y <= 2; y++) {
            for (int x = -2; x <= 2; x++) {
                vec2 n = vec2(float(x), float(y));
                vec2 g = hash22(i + n);
                vec2 r = n + g * 0.5 + 0.5 - f;
                float d = dot(r, r);
                if (d < d1) { d2 = d1; d1 = d; }
                else if (d < d2) { d2 = d; }
            }
        }
        return vec2(sqrt(d1), sqrt(d2));
    }
    
    // --- Hexagonal Grid ---
    float hexSDF(vec2 p) {
        p = abs(p);
        return max(p.x * 0.866025 + p.y * 0.5, p.y);
    }
    
    float hexGrid(vec2 p, float s) {
        vec2 r = vec2(1.0, 1.732) * s;
        vec2 h = r * 0.5;
        vec2 a = mod(p, r) - h;
        vec2 b = mod(p + h, r) - h;
        float da = hexSDF(a);
        float db = hexSDF(b);
        float hex = 0.5 - min(da, db);
        return smoothstep(0.0, 0.02, hex);
    }
    
    void main() {
        vec2 uv = vUv;
        if (uAnimate) uv.x += uTime * 0.05;
        
        float s = uScale;
        vec2 scaledUv = uv * s;
        
        vec3 color = vec3(0.0);
        
        if (uPattern == 0) {
            // Grid
            vec2 g = fract(scaledUv);
            float lx = step(0.95, g.x);
            float ly = step(0.95, g.y);
            float grid = max(lx, ly);
            color = vec3(grid) * vec3(0.2, 0.6, 1.0) + vec3(0.05, 0.05, 0.1);
            
        } else if (uPattern == 1) {
            // Dots
            vec2 g = fract(scaledUv) - 0.5;
            float d = length(g);
            float dot_ = 1.0 - smoothstep(0.25, 0.3, d);
            color = dot_ * vec3(1.0, 0.5, 0.2) + vec3(0.05, 0.05, 0.05);
            
        } else if (uPattern == 2) {
            // Checker
            vec2 g = floor(scaledUv);
            float c = mod(g.x + g.y, 2.0);
            color = mix(vec3(0.1, 0.1, 0.2), vec3(0.9, 0.9, 1.0), c);
            
        } else if (uPattern == 3) {
            // Hex Grid
            float h = hexGrid(scaledUv, 0.5);
            color = h * vec3(0.2, 0.8, 0.5) + vec3(0.02, 0.1, 0.05);
            
        } else if (uPattern == 4) {
            // Voronoi
            vec2 v = voronoi(scaledUv);
            float edge = smoothstep(0.05, 0.0, v.y - v.x);
            float cell = v.x;
            color = mix(vec3(cell * 0.5, cell * 0.3, cell), vec3(1.0, 0.8, 0.2), edge);
            
        } else if (uPattern == 5) {
            // Value Noise
            float n = valueNoise(scaledUv);
            vec3 c1 = vec3(0.1, 0.2, 0.5);
            vec3 c2 = vec3(0.8, 0.6, 0.2);
            color = mix(c1, c2, n);
            
        } else if (uPattern == 6) {
            // Gradient Noise
            float n = gradNoise(scaledUv);
            // fBm: stack multiple octaves
            float fbm = n;
            fbm += gradNoise(scaledUv * 2.0) * 0.5;
            fbm += gradNoise(scaledUv * 4.0) * 0.25;
            fbm /= 1.75;
            
            vec3 c1 = vec3(0.4, 0.1, 0.6);
            vec3 c2 = vec3(0.9, 0.9, 0.4);
            color = mix(c1, c2, fbm);
            
        } else {
            // Animated composite
            float t = uTime;
            float v1 = valueNoise(scaledUv + t * 0.1);
            float v2 = gradNoise(scaledUv * 1.5 - t * 0.05);
            
            float n = v1 * 0.6 + v2 * 0.4;
            
            // Cosine palette
            vec3 a = vec3(0.5);
            vec3 b = vec3(0.5);
            vec3 c = vec3(1.0, 1.0, 0.5);
            vec3 d = vec3(0.0, 0.33, 0.67);
            color = a + b * cos(6.28318 * (c * n + d + t * 0.1));
        }
        
        gl_FragColor = vec4(color, 1.0);
    }
`;

const geometry = new THREE.PlaneGeometry(2, 2);
const uniforms = {
    uTime:    { value: 0 },
    uPattern: { value: 0 },
    uScale:   { value: 5.0 },
    uAnimate: { value: false }
};

const material = new THREE.ShaderMaterial({
    vertexShader: `
        varying vec2 vUv;
        void main() {
            vUv = uv;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
    `,
    fragmentShader,
    uniforms
});

scene.add(new THREE.Mesh(geometry, material));

// Controls
for (let i = 0; i < 8; i++) {
    document.getElementById('p' + i)?.addEventListener('click', () => {
        uniforms.uPattern.value = i;
        document.querySelectorAll('button').forEach(b => b.classList.remove('active'));
        document.getElementById('p' + i).classList.add('active');
    });
}

document.getElementById('scale').addEventListener('input', (e) => {
    uniforms.uScale.value = parseFloat(e.target.value);
});
document.getElementById('animate').addEventListener('change', (e) => {
    uniforms.uAnimate.value = e.target.checked;
});

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

## Demo 2: SDF Shapes Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>SDF Shapes Demo</title>
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
            max-width: 250px;
        }
        select, input { width: 100%; margin: 3px 0; }
    </style>
</head>
<body>
<div id="ui">
    <b>SDF Shapes</b><br><br>
    Shape: <select id="shape">
        <option value="0">Circle</option>
        <option value="1">Box</option>
        <option value="2">Rounded Box</option>
        <option value="3">Triangle</option>
        <option value="4">Star (5)</option>
        <option value="5">Cross</option>
        <option value="6">Ring</option>
        <option value="7">Smooth Union</option>
        <option value="8">Difference</option>
    </select>
    <br>
    Render Mode:
    <select id="mode">
        <option value="0">Fill</option>
        <option value="1">Outline</option>
        <option value="2">Glow</option>
        <option value="3">Distance Field</option>
        <option value="4">Normal Map</option>
    </select>
    <br>
    Size: <input type="range" id="size" min="0.05" max="0.5" value="0.3" step="0.01">
    <br>
    Animate: <input type="checkbox" id="anim" checked>
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
    uniform int   uShape;
    uniform int   uMode;
    uniform float uSize;
    uniform bool  uAnimate;
    
    varying vec2 vUv;
    
    // --- SDF Functions ---
    float sdCircle(vec2 p, float r) {
        return length(p) - r;
    }
    
    float sdBox(vec2 p, vec2 b) {
        vec2 d = abs(p) - b;
        return length(max(d, 0.0)) + min(max(d.x, d.y), 0.0);
    }
    
    float sdRoundedBox(vec2 p, vec2 b, float r) {
        vec2 d = abs(p) - b + r;
        return length(max(d, 0.0)) + min(max(d.x, d.y), 0.0) - r;
    }
    
    float sdTriangle(vec2 p) {
        const float k = sqrt(3.0);
        p.x = abs(p.x) - 1.0;
        p.y = p.y + 1.0/k;
        if (p.x + k * p.y > 0.0) p = vec2(p.x - k * p.y, -k * p.x - p.y) / 2.0;
        p.x -= clamp(p.x, -2.0, 0.0);
        return -length(p) * sign(p.y);
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
    
    float sdCross(vec2 p, vec2 b, float r) {
        p = abs(p); p = (p.y > p.x) ? p.yx : p.xy;
        vec2 q = p - b;
        float k = max(q.y, q.x);
        vec2 w = (k > 0.0) ? q : vec2(b.y - p.x, -k);
        return sign(k) * length(max(w, 0.0)) + r;
    }
    
    float sdRing(vec2 p, float r, float thickness) {
        return abs(length(p) - r) - thickness;
    }
    
    float sdSmoothUnion(float d1, float d2, float k) {
        float h = clamp(0.5 + 0.5 * (d2 - d1) / k, 0.0, 1.0);
        return mix(d2, d1, h) - k * h * (1.0 - h);
    }
    
    // คำนวณ SDF
    float getSDF(vec2 p, float size, float t) {
        float s = size;
        float d = 0.0;
        
        if (uShape == 0) {
            d = sdCircle(p, s);
            
        } else if (uShape == 1) {
            d = sdBox(p, vec2(s, s * 0.7));
            
        } else if (uShape == 2) {
            d = sdRoundedBox(p, vec2(s * 0.9, s * 0.6), s * 0.2);
            
        } else if (uShape == 3) {
            d = sdTriangle(p / s) * s;
            
        } else if (uShape == 4) {
            d = sdStar5(p, s);
            
        } else if (uShape == 5) {
            d = sdCross(p, vec2(s * 0.8, s * 0.25), 0.0);
            
        } else if (uShape == 6) {
            d = sdRing(p, s, s * 0.15);
            
        } else if (uShape == 7) {
            // Animated smooth union ของ 2 circles
            float offset = uAnimate ? sin(t) * s * 0.8 : s * 0.5;
            float d1 = sdCircle(p - vec2(offset, 0.0), s * 0.7);
            float d2 = sdCircle(p + vec2(offset, 0.0), s * 0.7);
            d = sdSmoothUnion(d1, d2, s * 0.5);
            
        } else {
            // Difference: circle - moving box
            float offset = uAnimate ? sin(t * 1.5) * s : s * 0.5;
            float d1 = sdCircle(p, s);
            float d2 = sdBox(p - vec2(offset, 0.0), vec2(s * 0.5, s * 1.5));
            d = max(d1, -d2); // difference
        }
        
        return d;
    }
    
    void main() {
        // Center UV: (-1, 1) range
        vec2 uv = (vUv - 0.5) * 2.0;
        // Correct aspect ratio
        uv.x *= float(${innerWidth}) / float(${innerHeight});
        
        float t = uTime;
        float s = uSize;
        
        // Animate rotation
        vec2 p = uv;
        if (uAnimate && uShape <= 5) {
            float angle = t * 0.5;
            p = vec2(
                p.x * cos(angle) - p.y * sin(angle),
                p.x * sin(angle) + p.y * cos(angle)
            );
        }
        
        float sdf = getSDF(p, s, t);
        
        vec3 color = vec3(0.0);
        float alpha = 1.0;
        
        if (uMode == 0) {
            // Fill
            float fill = smoothstep(0.005, -0.005, sdf);
            color = fill * vec3(0.3, 0.7, 1.0);
            color += (1.0 - fill) * vec3(0.05, 0.05, 0.1);
            
        } else if (uMode == 1) {
            // Outline
            float outline = smoothstep(0.015, 0.005, abs(sdf));
            color = outline * vec3(1.0, 0.8, 0.3);
            color += vec3(0.05, 0.05, 0.1);
            
        } else if (uMode == 2) {
            // Glow
            float fill = smoothstep(0.005, -0.005, sdf);
            float glow1 = exp(-abs(sdf) * 15.0);
            float glow2 = exp(-abs(sdf) * 30.0);
            color = fill * vec3(0.1, 0.4, 1.0);
            color += glow1 * vec3(0.2, 0.5, 1.0) * 0.5;
            color += glow2 * vec3(0.5, 0.7, 1.0) * 0.8;
            
        } else if (uMode == 3) {
            // Distance Field Visualization
            float d = sdf;
            float rings = sin(d * 30.0) * 0.5 + 0.5;
            float fill = smoothstep(0.005, -0.005, d);
            
            vec3 outside = mix(vec3(0.05, 0.1, 0.3), vec3(0.3, 0.5, 0.8), rings);
            vec3 inside  = mix(vec3(0.3, 0.05, 0.05), vec3(0.8, 0.3, 0.3), rings);
            
            color = mix(outside, inside, fill);
            
        } else {
            // Normal Map from SDF
            float e = 0.001;
            vec3 n = normalize(vec3(
                getSDF(p + vec2(e, 0), s, t) - getSDF(p - vec2(e, 0), s, t),
                getSDF(p + vec2(0, e), s, t) - getSDF(p - vec2(0, e), s, t),
                e * 100.0
            ));
            
            float fill = smoothstep(0.005, -0.005, sdf);
            
            // Lighting from normal
            vec3 lightDir = normalize(vec3(0.5, 0.8, 1.0));
            float diffuse = max(0.0, dot(n, lightDir));
            float specular = pow(diffuse, 32.0);
            
            color = fill * (vec3(0.3, 0.5, 0.8) * diffuse + specular);
        }
        
        gl_FragColor = vec4(color, 1.0);
    }
`;

const geometry = new THREE.PlaneGeometry(2, 2);
const uniforms = {
    uTime:   { value: 0 },
    uShape:  { value: 0 },
    uMode:   { value: 0 },
    uSize:   { value: 0.3 },
    uAnimate:{ value: true }
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

document.getElementById('shape').onchange = (e) => uniforms.uShape.value = parseInt(e.target.value);
document.getElementById('mode').onchange = (e) => uniforms.uMode.value = parseInt(e.target.value);
document.getElementById('size').oninput = (e) => uniforms.uSize.value = parseFloat(e.target.value);
document.getElementById('anim').onchange = (e) => uniforms.uAnimate.value = e.target.checked;

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

## gl_FragDepth

```glsl
// gl_FragDepth ให้เขียน depth ของ fragment เองแทน default
// ปกติ depth จะถูกคำนวณจาก gl_Position.z/w อัตโนมัติ

void main() {
    // ตัวอย่าง: เขียน depth ที่ flat กว่า actual geometry
    float customDepth = gl_FragCoord.z * 0.5; // แบนครึ่งหนึ่ง
    gl_FragDepth = customDepth;
    
    gl_FragColor = vec4(1.0, 0.0, 0.0, 1.0);
    
    // WARNING: การใช้ gl_FragDepth ทำให้ GPU ปิด early fragment test
    // (ไม่ discard fragments ก่อนรัน shader) → ช้าลง
    // ใช้เฉพาะเมื่อจำเป็น
}
```

---

## Fragment Shader Performance Tips

```glsl
// 1. ใช้ mediump float ถ้าไม่ต้องการ highp
precision mediump float;

// 2. Avoid branching (if/else) เมื่อทำได้
// แทนที่จะ:
float result;
if (x > 0.5) {
    result = 1.0;
} else {
    result = 0.0;
}

// ใช้ step:
float result2 = step(0.5, x);

// 3. Avoid expensive operations
// ไม่ดี: pow(x, 2.0)
// ดีกว่า: x * x

// 4. ใช้ built-in functions ที่ GPU optimize ไว้
float invSqrt = inversesqrt(x); // เร็วกว่า 1.0/sqrt(x)

// 5. Minimize texture samples ใน fragment shader
// Cache ผลลัพธ์ ไม่ sample ซ้ำ
vec4 texColor = texture2D(uTex, vUv); // sample ครั้งเดียว
// ใช้ texColor หลายครั้งได้เลย

// 6. discard keyword ทำให้ fragment ไม่ถูก render
// ใช้สำหรับ transparency patterns
void main() {
    float alpha = texture2D(uTex, vUv).a;
    if (alpha < 0.1) discard; // ทิ้ง pixel โปร่งใส
    gl_FragColor = vec4(color, alpha);
}
```

---

## แบบฝึกหัด

### ระดับ 1
1. เขียน fragment shader ที่แสดง red circle ตรงกลาง plane
2. ทำให้วงกลมมี outline สีทอง
3. เพิ่ม glow effect ให้กับวงกลม

### ระดับ 2
4. สร้าง checkerboard pattern ที่มีขนาด tile ปรับได้ผ่าน uniform
5. สร้าง Voronoi pattern
6. สร้าง SDF star 5 แฉก

### ระดับ 3
7. Combine SDF shapes ด้วย boolean operations (union, intersect, difference)
8. สร้าง animated SDF ที่ shapes เคลื่อนที่และ smooth union กัน
9. สร้าง Normal map จาก SDF สำหรับ lighting effect

### Challenge
10. สร้าง procedural planet surface โดย:
    - ใช้ fBm noise เป็น height map
    - ใช้ height ในการกำหนดสี (น้ำ/ทราย/หิน/หิมะ)
    - เพิ่ม specular reflection บนน้ำ
    - เพิ่ม cloud layer ด้วย noise ที่ต่างกัน
