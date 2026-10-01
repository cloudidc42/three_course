# Part 50: Motion Blur & Camera Effects

## บทนำ: ทำไมต้องมี Motion Blur?

Motion Blur หรือ "ความพร่ามัวจากการเคลื่อนไหว" เป็นปรากฏการณ์ที่เกิดขึ้นเมื่อกล้องหรือวัตถุเคลื่อนที่ระหว่างการถ่ายภาพ ในกล้องจริง แสงจะถูกเก็บสะสมตลอดช่วง shutter speed ทำให้วัตถุที่เคลื่อนที่ดูพร่ามัว

ในโลก 3D Graphics เราจำลอง motion blur เพื่อ:
- **เพิ่มความสมจริง** - สมองมนุษย์คาดหวัง blur เมื่อมีการเคลื่อนที่
- **ซ่อน temporal aliasing** - ลดการ "กระตุก" ระหว่าง frames
- **เพิ่ม sense of speed** - blur ทำให้รู้สึกว่าเร็วขึ้น
- **สร้าง cinematic feel** - เอฟเฟกต์แบบภาพยนตร์

## ประเภทของ Motion Blur

### 1. Camera Motion Blur
เกิดจากการเคลื่อนที่ของกล้องเอง ทำให้ทั้งฉากดูพร่ามัว

### 2. Per-Object Motion Blur
เกิดจากการเคลื่อนที่ของวัตถุแต่ละชิ้น ขณะที่กล้องนิ่ง

### 3. Radial/Zoom Blur
เอฟเฟกต์ blur ที่แผ่ออกมาจากจุดศูนย์กลาง ใช้สำหรับความรู้สึก speed

### 4. Temporal Anti-Aliasing (TAA)
ใช้ข้อมูลจาก frame ก่อนหน้าเพื่อลด aliasing และสร้าง subtle motion blur

## Velocity Buffer

Velocity Buffer เป็น render target พิเศษที่เก็บข้อมูลการเคลื่อนที่ของแต่ละ pixel:

```javascript
// การสร้าง Velocity Buffer
class VelocityBuffer {
    constructor(width, height) {
        this.renderTarget = new THREE.WebGLRenderTarget(width, height, {
            minFilter: THREE.LinearFilter,
            magFilter: THREE.LinearFilter,
            format: THREE.RGBAFormat,
            type: THREE.HalfFloatType
        });
        
        // Material พิเศษที่คำนวณ velocity
        this.material = new THREE.ShaderMaterial({
            uniforms: {
                prevMVP: { value: new THREE.Matrix4() }, // previous Model-View-Projection
                currMVP: { value: new THREE.Matrix4() }  // current Model-View-Projection
            },
            vertexShader: `
                uniform mat4 prevMVP;
                uniform mat4 currMVP;
                
                varying vec4 currPos;
                varying vec4 prevPos;
                
                void main() {
                    currPos = currMVP * vec4(position, 1.0);
                    prevPos = prevMVP * vec4(position, 1.0);
                    gl_Position = currPos;
                }
            `,
            fragmentShader: `
                varying vec4 currPos;
                varying vec4 prevPos;
                
                void main() {
                    // Screen-space velocity
                    vec2 curr = (currPos.xy / currPos.w) * 0.5 + 0.5;
                    vec2 prev = (prevPos.xy / prevPos.w) * 0.5 + 0.5;
                    vec2 velocity = curr - prev;
                    
                    // เก็บ velocity ใน R,G channels (scale เพื่อ precision)
                    gl_FragColor = vec4(velocity * 0.5 + 0.5, 0.0, 1.0);
                }
            `
        });
    }
    
    update(renderer, scene, camera) {
        renderer.setRenderTarget(this.renderTarget);
        // Override materials และ render
        // ...
        renderer.setRenderTarget(null);
    }
}
```

## Per-Object Motion Blur Shader

```javascript
// Shader สำหรับ per-object motion blur
const motionBlurShader = {
    uniforms: {
        tDiffuse: { value: null },
        tVelocity: { value: null },
        samples: { value: 8 },
        scale: { value: 1.0 }
    },
    vertexShader: `
        varying vec2 vUv;
        void main() {
            vUv = uv;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
    `,
    fragmentShader: `
        uniform sampler2D tDiffuse;
        uniform sampler2D tVelocity;
        uniform int samples;
        uniform float scale;
        
        varying vec2 vUv;
        
        void main() {
            vec2 velocity = (texture2D(tVelocity, vUv).rg - 0.5) * 2.0 * scale;
            
            vec4 color = vec4(0.0);
            float totalWeight = 0.0;
            
            for (int i = 0; i < 16; i++) {
                if (i >= samples) break;
                float t = float(i) / float(samples - 1) - 0.5;
                vec2 offset = velocity * t;
                vec2 sampleUv = vUv + offset;
                
                // Clamp to texture bounds
                if (sampleUv.x >= 0.0 && sampleUv.x <= 1.0 &&
                    sampleUv.y >= 0.0 && sampleUv.y <= 1.0) {
                    color += texture2D(tDiffuse, sampleUv);
                    totalWeight += 1.0;
                }
            }
            
            if (totalWeight > 0.0) color /= totalWeight;
            gl_FragColor = color;
        }
    `
};
```

## Radial Blur (Speed Lines)

Radial blur แผ่ออกจากจุดศูนย์กลางเหมือน speed lines ในการ์ตูน:

```javascript
// Radial Blur / Speed Lines Shader
const radialBlurShader = {
    uniforms: {
        tDiffuse: { value: null },
        center: { value: new THREE.Vector2(0.5, 0.5) }, // จุดศูนย์กลาง
        strength: { value: 0.1 },  // ความแรงของ blur
        samples: { value: 20 }     // จำนวน samples
    },
    vertexShader: `
        varying vec2 vUv;
        void main() {
            vUv = uv;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
    `,
    fragmentShader: `
        uniform sampler2D tDiffuse;
        uniform vec2 center;
        uniform float strength;
        uniform int samples;
        
        varying vec2 vUv;
        
        void main() {
            vec2 dir = vUv - center;
            float dist = length(dir);
            
            vec4 color = vec4(0.0);
            float total = 0.0;
            
            for (int i = 0; i < 30; i++) {
                if (i >= samples) break;
                float t = float(i) / float(samples);
                float scale = 1.0 - strength * t;
                vec2 sampleUv = center + dir * scale;
                
                // Gaussian weight
                float weight = exp(-4.0 * t);
                color += texture2D(tDiffuse, clamp(sampleUv, vec2(0.0), vec2(1.0))) * weight;
                total += weight;
            }
            
            gl_FragColor = color / total;
        }
    `
};
```

## Zoom Blur

Zoom blur ให้ความรู้สึกของการเคลื่อนที่เข้าหาหรือออกจากกล้อง:

```javascript
// Zoom Blur Shader
const zoomBlurShader = {
    uniforms: {
        tDiffuse: { value: null },
        center: { value: new THREE.Vector2(0.5, 0.5) },
        strength: { value: 0.05 },
        samples: { value: 15 }
    },
    vertexShader: `
        varying vec2 vUv;
        void main() {
            vUv = uv;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
    `,
    fragmentShader: `
        uniform sampler2D tDiffuse;
        uniform vec2 center;
        uniform float strength;
        uniform int samples;
        
        varying vec2 vUv;
        
        void main() {
            vec4 color = vec4(0.0);
            float totalWeight = 0.0;
            
            for (int i = 0; i < 20; i++) {
                if (i >= samples) break;
                float scale = 1.0 + strength * float(i) / float(samples);
                vec2 sampleUv = center + (vUv - center) * scale;
                
                float weight = 1.0 / (1.0 + float(i));
                color += texture2D(tDiffuse, clamp(sampleUv, vec2(0.0), vec2(1.0))) * weight;
                totalWeight += weight;
            }
            
            gl_FragColor = color / totalWeight;
        }
    `
};
```

## Camera Shake Effect

```javascript
// Camera Shake System
class CameraShake {
    constructor(camera) {
        this.camera = camera;
        this.originalPosition = camera.position.clone();
        this.originalRotation = camera.rotation.clone();
        
        this.intensity = 0.0;
        this.duration = 0.0;
        this.elapsed = 0.0;
        this.frequency = 10.0;
        this.enabled = false;
        
        // Noise สำหรับ shake
        this.noiseX = this._createNoise();
        this.noiseY = this._createNoise();
        this.noiseZ = this._createNoise();
    }
    
    _createNoise() {
        // Simple perlin-like noise array
        const n = new Float32Array(256);
        for (let i = 0; i < 256; i++) {
            n[i] = Math.random() * 2 - 1;
        }
        return n;
    }
    
    _smoothNoise(t, noiseArray) {
        const idx = Math.floor(t) % 256;
        const frac = t - Math.floor(t);
        const a = noiseArray[idx];
        const b = noiseArray[(idx + 1) % 256];
        // Smooth interpolation
        const smooth = frac * frac * (3 - 2 * frac);
        return a + (b - a) * smooth;
    }
    
    shake(intensity = 0.1, duration = 0.5, frequency = 10.0) {
        this.intensity = intensity;
        this.duration = duration;
        this.elapsed = 0.0;
        this.frequency = frequency;
        this.enabled = true;
    }
    
    // Trauma system - shake ที่ลดลงเมื่อเวลาผ่านไป
    addTrauma(amount) {
        this.trauma = Math.min(1.0, (this.trauma || 0) + amount);
        this.enabled = true;
    }
    
    update(deltaTime) {
        if (!this.enabled) return;
        
        this.elapsed += deltaTime;
        
        // Trauma-based shake (ลดลงเรื่อยๆ)
        if (this.trauma !== undefined && this.trauma > 0) {
            const shake = this.trauma * this.trauma;
            const t = this.elapsed * this.frequency;
            
            const maxAngle = 0.05;
            const maxOffset = 0.05;
            
            this.camera.rotation.x = this.originalRotation.x + 
                maxAngle * shake * this._smoothNoise(t, this.noiseX);
            this.camera.rotation.y = this.originalRotation.y + 
                maxAngle * shake * this._smoothNoise(t + 100, this.noiseY);
            this.camera.position.x = this.originalPosition.x + 
                maxOffset * shake * this._smoothNoise(t + 200, this.noiseZ);
            
            this.trauma = Math.max(0, this.trauma - deltaTime * 0.5);
            if (this.trauma <= 0) {
                this.reset();
            }
        } else {
            // Duration-based shake
            if (this.elapsed >= this.duration) {
                this.reset();
                return;
            }
            
            const progress = this.elapsed / this.duration;
            const currentIntensity = this.intensity * (1 - progress);
            const t = this.elapsed * this.frequency;
            
            this.camera.position.x = this.originalPosition.x + 
                currentIntensity * this._smoothNoise(t, this.noiseX);
            this.camera.position.y = this.originalPosition.y + 
                currentIntensity * this._smoothNoise(t + 50, this.noiseY);
            this.camera.rotation.z = this.originalRotation.z + 
                currentIntensity * 0.1 * this._smoothNoise(t + 100, this.noiseZ);
        }
    }
    
    reset() {
        this.camera.position.copy(this.originalPosition);
        this.camera.rotation.copy(this.originalRotation);
        this.enabled = false;
        this.trauma = 0;
    }
    
    setOriginalTransform() {
        this.originalPosition.copy(this.camera.position);
        this.originalRotation.copy(this.camera.rotation);
    }
}
```

## Chromatic Aberration

เอฟเฟกต์ที่ทำให้ช่องสีแต่ละช่องถูก offset ต่างกัน:

```javascript
// Chromatic Aberration Shader
const chromaticAberrationShader = {
    uniforms: {
        tDiffuse: { value: null },
        offset: { value: 0.005 },          // ปริมาณการแยก
        radialMask: { value: true },        // ใช้ mask แบบ radial
        maskRadius: { value: 0.4 },         // รัศมีของ mask
        maskSoftness: { value: 0.3 }        // ความนุ่มนวลของ mask
    },
    vertexShader: `
        varying vec2 vUv;
        void main() {
            vUv = uv;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
    `,
    fragmentShader: `
        uniform sampler2D tDiffuse;
        uniform float offset;
        uniform bool radialMask;
        uniform float maskRadius;
        uniform float maskSoftness;
        
        varying vec2 vUv;
        
        void main() {
            vec2 center = vec2(0.5);
            vec2 dir = vUv - center;
            
            float mask = 1.0;
            if (radialMask) {
                float dist = length(dir);
                mask = smoothstep(maskRadius, maskRadius + maskSoftness, dist);
            }
            
            float r = texture2D(tDiffuse, vUv + dir * offset * mask).r;
            float g = texture2D(tDiffuse, vUv).g;
            float b = texture2D(tDiffuse, vUv - dir * offset * mask).b;
            
            gl_FragColor = vec4(r, g, b, 1.0);
        }
    `
};
```

## Vignette Effect

```javascript
// Vignette Shader
const vignetteShader = {
    uniforms: {
        tDiffuse: { value: null },
        offset: { value: 0.95 },   // เริ่มต้นของ vignette
        darkness: { value: 1.6 },  // ความเข้มของขอบ
        color: { value: new THREE.Color(0, 0, 0) } // สีของ vignette
    },
    vertexShader: `
        varying vec2 vUv;
        void main() {
            vUv = uv;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
    `,
    fragmentShader: `
        uniform sampler2D tDiffuse;
        uniform float offset;
        uniform float darkness;
        uniform vec3 color;
        
        varying vec2 vUv;
        
        void main() {
            vec4 texel = texture2D(tDiffuse, vUv);
            
            // วิธีง่ายๆ
            vec2 uv = (vUv - 0.5) * 2.0;
            float dist = length(uv);
            float vignette = smoothstep(offset, offset - 0.3, dist);
            
            // Apply vignette
            texel.rgb = mix(color, texel.rgb, vignette);
            
            gl_FragColor = texel;
        }
    `
};
```

## ตัวอย่างที่ 1: Racing Game Speed Effect

ฉากเกมรถแข่งที่มี motion blur, radial blur, และ camera shake:

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Racing Game Speed Effects</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; font-family: Arial, sans-serif; }
        canvas { display: block; }
        #hud {
            position: fixed;
            bottom: 0;
            left: 0;
            right: 0;
            padding: 20px;
            display: flex;
            justify-content: space-between;
            align-items: flex-end;
        }
        #speedometer {
            background: rgba(0,0,0,0.7);
            color: white;
            padding: 15px 25px;
            border-radius: 10px;
            text-align: center;
        }
        #speed-num { font-size: 48px; font-weight: bold; color: #ff6b35; }
        #speed-unit { font-size: 14px; color: #aaa; }
        #controls-info {
            background: rgba(0,0,0,0.7);
            color: white;
            padding: 10px 15px;
            border-radius: 8px;
            font-size: 13px;
            line-height: 1.8;
        }
        #effects-panel {
            position: fixed;
            top: 20px;
            right: 20px;
            background: rgba(0,0,0,0.8);
            color: white;
            padding: 15px;
            border-radius: 10px;
            width: 200px;
        }
        .effect-toggle {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-bottom: 8px;
            font-size: 13px;
        }
        .toggle-switch {
            position: relative;
            width: 40px;
            height: 20px;
        }
        .toggle-switch input { display: none; }
        .slider-sw {
            position: absolute;
            cursor: pointer;
            top: 0; left: 0; right: 0; bottom: 0;
            background: #555;
            border-radius: 10px;
            transition: 0.3s;
        }
        .slider-sw:before {
            content: '';
            position: absolute;
            width: 14px;
            height: 14px;
            left: 3px;
            bottom: 3px;
            background: white;
            border-radius: 50%;
            transition: 0.3s;
        }
        input:checked + .slider-sw { background: #ff6b35; }
        input:checked + .slider-sw:before { transform: translateX(20px); }
    </style>
</head>
<body>
    <div id="hud">
        <div id="speedometer">
            <div id="speed-num">0</div>
            <div id="speed-unit">KM/H</div>
        </div>
        <div id="controls-info">
            ↑↓ เพิ่ม/ลดความเร็ว<br>
            ←→ เลี้ยว<br>
            SPACE = เบรก<br>
            R = ปรับตำแหน่ง
        </div>
    </div>
    <div id="effects-panel">
        <h4 style="margin-bottom:10px;color:#ff6b35">Effects</h4>
        <div class="effect-toggle">
            <span>Radial Blur</span>
            <label class="toggle-switch">
                <input type="checkbox" id="radialBlurToggle" checked>
                <div class="slider-sw"></div>
            </label>
        </div>
        <div class="effect-toggle">
            <span>Camera Shake</span>
            <label class="toggle-switch">
                <input type="checkbox" id="shakeToggle" checked>
                <div class="slider-sw"></div>
            </label>
        </div>
        <div class="effect-toggle">
            <span>Chromatic AB</span>
            <label class="toggle-switch">
                <input type="checkbox" id="caToggle" checked>
                <div class="slider-sw"></div>
            </label>
        </div>
        <div class="effect-toggle">
            <span>Vignette</span>
            <label class="toggle-switch">
                <input type="checkbox" id="vigToggle" checked>
                <div class="slider-sw"></div>
            </label>
        </div>
        <div class="effect-toggle">
            <span>Speed Lines</span>
            <label class="toggle-switch">
                <input type="checkbox" id="speedLinesToggle" checked>
                <div class="slider-sw"></div>
            </label>
        </div>
    </div>

    <script type="module">
        import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js';
        import { EffectComposer } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/EffectComposer.js';
        import { RenderPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/RenderPass.js';
        import { ShaderPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/ShaderPass.js';
        import { OutputPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/OutputPass.js';
        import { UnrealBloomPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/UnrealBloomPass.js';

        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        renderer.shadowMap.enabled = true;
        document.body.appendChild(renderer.domElement);

        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x0a0a1a);
        scene.fog = new THREE.Fog(0x0a0a1a, 30, 200);

        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 500);

        // Lighting
        scene.add(new THREE.AmbientLight(0x111133, 1));
        const moonLight = new THREE.DirectionalLight(0x8888cc, 0.5);
        moonLight.position.set(10, 20, 10);
        moonLight.castShadow = true;
        scene.add(moonLight);

        // สร้างถนน
        function createRaceTrack() {
            const trackGroup = new THREE.Group();
            
            // พื้นถนน - ยาวมาก
            const trackLength = 500;
            const trackWidth = 12;
            
            const roadGeo = new THREE.PlaneGeometry(trackWidth, trackLength, 1, 100);
            const roadMat = new THREE.MeshStandardMaterial({
                color: 0x222222,
                roughness: 0.9,
                metalness: 0.1
            });
            const road = new THREE.Mesh(roadGeo, roadMat);
            road.rotation.x = -Math.PI / 2;
            road.position.z = -trackLength / 2 + 10;
            road.receiveShadow = true;
            trackGroup.add(road);
            
            // เส้นกลาง (dashed)
            for (let i = 0; i < 100; i++) {
                const lineGeo = new THREE.PlaneGeometry(0.3, 3);
                const lineMat = new THREE.MeshStandardMaterial({ color: 0xffff00 });
                const line = new THREE.Mesh(lineGeo, lineMat);
                line.rotation.x = -Math.PI / 2;
                line.position.set(0, 0.01, -i * 5);
                trackGroup.add(line);
            }
            
            // กำแพงถนน
            const wallMat = new THREE.MeshStandardMaterial({ color: 0x444444 });
            for (let side of [-1, 1]) {
                const wallGeo = new THREE.BoxGeometry(0.5, 1, trackLength);
                const wall = new THREE.Mesh(wallGeo, wallMat);
                wall.position.set(side * (trackWidth / 2 + 0.25), 0.5, -trackLength / 2 + 10);
                wall.castShadow = true;
                wall.receiveShadow = true;
                trackGroup.add(wall);
            }
            
            // ป้าย
            for (let i = 0; i < 20; i++) {
                const poleGeo = new THREE.CylinderGeometry(0.05, 0.05, 3, 6);
                const poleMat = new THREE.MeshStandardMaterial({ color: 0x888888 });
                const pole = new THREE.Mesh(poleGeo, poleMat);
                pole.position.set(
                    (Math.random() > 0.5 ? -1 : 1) * 8,
                    1.5,
                    -i * 25
                );
                pole.castShadow = true;
                trackGroup.add(pole);
            }
            
            // พื้นรอบถนน
            const sideGeo = new THREE.PlaneGeometry(200, trackLength);
            const sideMat = new THREE.MeshStandardMaterial({ color: 0x1a3a1a });
            const side = new THREE.Mesh(sideGeo, sideMat);
            side.rotation.x = -Math.PI / 2;
            side.position.set(0, -0.01, -trackLength / 2 + 10);
            side.receiveShadow = true;
            trackGroup.add(side);
            
            // Street lights
            for (let i = 0; i < 30; i++) {
                const lightGeo = new THREE.CylinderGeometry(0.04, 0.04, 4, 6);
                const lightMat = new THREE.MeshStandardMaterial({ color: 0x888888 });
                const lightPole = new THREE.Mesh(lightGeo, lightMat);
                lightPole.position.set((i % 2 === 0 ? -1 : 1) * 7, 2, -i * 16);
                trackGroup.add(lightPole);
                
                const bulbGeo = new THREE.SphereGeometry(0.15, 8, 8);
                const bulbMat = new THREE.MeshStandardMaterial({
                    color: 0xffee88,
                    emissive: 0xffee44,
                    emissiveIntensity: 1
                });
                const bulb = new THREE.Mesh(bulbGeo, bulbMat);
                bulb.position.set((i % 2 === 0 ? -1 : 1) * 7, 4.2, -i * 16);
                trackGroup.add(bulb);
                
                const pointLight = new THREE.PointLight(0xffee44, 0.5, 15);
                pointLight.position.copy(bulb.position);
                trackGroup.add(pointLight);
            }
            
            return trackGroup;
        }

        // รถ
        function createCar() {
            const carGroup = new THREE.Group();
            
            const bodyMat = new THREE.MeshStandardMaterial({ color: 0xff3300, roughness: 0.3, metalness: 0.6 });
            const glassMat = new THREE.MeshStandardMaterial({ color: 0x88aaff, transparent: true, opacity: 0.6 });
            const wheelMat = new THREE.MeshStandardMaterial({ color: 0x111111, roughness: 0.9 });
            const rimMat = new THREE.MeshStandardMaterial({ color: 0x888888, metalness: 0.9 });
            
            // Body
            const bodyGeo = new THREE.BoxGeometry(1.8, 0.5, 4);
            const body = new THREE.Mesh(bodyGeo, bodyMat);
            body.position.y = 0.35;
            body.castShadow = true;
            carGroup.add(body);
            
            // Cabin
            const cabinGeo = new THREE.BoxGeometry(1.5, 0.45, 2);
            const cabin = new THREE.Mesh(cabinGeo, bodyMat);
            cabin.position.set(0, 0.85, -0.2);
            cabin.castShadow = true;
            carGroup.add(cabin);
            
            // Windshield
            const windshieldGeo = new THREE.BoxGeometry(1.4, 0.4, 0.05);
            const windshield = new THREE.Mesh(windshieldGeo, glassMat);
            windshield.position.set(0, 0.85, 0.8);
            windshield.rotation.x = 0.3;
            carGroup.add(windshield);
            
            // Wheels
            const wheelPositions = [
                [-0.95, 0.2, 1.2], [0.95, 0.2, 1.2],
                [-0.95, 0.2, -1.2], [0.95, 0.2, -1.2]
            ];
            const wheels = [];
            wheelPositions.forEach(pos => {
                const wheelGeo = new THREE.CylinderGeometry(0.35, 0.35, 0.25, 12);
                const wheel = new THREE.Mesh(wheelGeo, wheelMat);
                wheel.rotation.z = Math.PI / 2;
                wheel.position.set(...pos);
                wheel.castShadow = true;
                carGroup.add(wheel);
                wheels.push(wheel);
                
                const rimGeo = new THREE.CylinderGeometry(0.2, 0.2, 0.27, 8);
                const rim = new THREE.Mesh(rimGeo, rimMat);
                rim.rotation.z = Math.PI / 2;
                rim.position.set(...pos);
                carGroup.add(rim);
            });
            
            // Headlights
            const headlightGeo = new THREE.CylinderGeometry(0.15, 0.15, 0.05, 12);
            const headlightMat = new THREE.MeshStandardMaterial({ emissive: 0xffffff, emissiveIntensity: 2 });
            [-0.5, 0.5].forEach(x => {
                const hl = new THREE.Mesh(headlightGeo, headlightMat);
                hl.rotation.x = Math.PI / 2;
                hl.position.set(x, 0.4, 2.02);
                carGroup.add(hl);
                
                const spotlight = new THREE.SpotLight(0xffffee, 2, 30, Math.PI / 8);
                spotlight.position.set(x, 0.4, 2);
                spotlight.target.position.set(x, -0.5, 10);
                carGroup.add(spotlight);
                carGroup.add(spotlight.target);
            });
            
            return { group: carGroup, wheels };
        }

        const track = createRaceTrack();
        scene.add(track);
        
        const { group: car, wheels } = createCar();
        car.position.set(0, 0, 5);
        scene.add(car);

        // Car physics state
        const carState = {
            speed: 0,
            maxSpeed: 50,
            acceleration: 15,
            braking: 30,
            friction: 5,
            turnSpeed: 1.5,
            position: new THREE.Vector3(0, 0, 5),
            rotation: 0,
            lateralVelocity: 0
        };

        // Camera state
        const cameraState = {
            offset: new THREE.Vector3(0, 2.5, 7),
            smoothing: 0.08
        };

        // Input
        const keys = {};
        document.addEventListener('keydown', e => { keys[e.key] = true; });
        document.addEventListener('keyup', e => { keys[e.key] = false; });

        // Camera Shake
        let shakeIntensity = 0;
        let shakeTime = 0;

        // Post-processing
        const composer = new EffectComposer(renderer);
        composer.addPass(new RenderPass(scene, camera));

        // Radial blur
        const RadialBlurShader = {
            uniforms: {
                tDiffuse: { value: null },
                center: { value: new THREE.Vector2(0.5, 0.5) },
                strength: { value: 0.0 },
                samples: { value: 16 }
            },
            vertexShader: `varying vec2 vUv; void main() { vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
            fragmentShader: `
                uniform sampler2D tDiffuse;
                uniform vec2 center;
                uniform float strength;
                uniform int samples;
                varying vec2 vUv;
                void main() {
                    if (strength < 0.001) { gl_FragColor = texture2D(tDiffuse, vUv); return; }
                    vec2 dir = center - vUv;
                    vec4 color = vec4(0.0);
                    float total = 0.0;
                    for (int i = 0; i < 20; i++) {
                        if (i >= samples) break;
                        float t = float(i) / float(samples);
                        float w = exp(-3.0 * t);
                        color += texture2D(tDiffuse, vUv + dir * t * strength) * w;
                        total += w;
                    }
                    gl_FragColor = color / total;
                }
            `
        };
        const radialBlurPass = new ShaderPass(RadialBlurShader);
        composer.addPass(radialBlurPass);

        // Chromatic aberration
        const CARShader = {
            uniforms: {
                tDiffuse: { value: null },
                offset: { value: 0.0 }
            },
            vertexShader: `varying vec2 vUv; void main() { vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
            fragmentShader: `
                uniform sampler2D tDiffuse;
                uniform float offset;
                varying vec2 vUv;
                void main() {
                    vec2 center = vec2(0.5);
                    vec2 dir = normalize(vUv - center) * offset;
                    float r = texture2D(tDiffuse, vUv + dir).r;
                    float g = texture2D(tDiffuse, vUv).g;
                    float b = texture2D(tDiffuse, vUv - dir).b;
                    gl_FragColor = vec4(r, g, b, 1.0);
                }
            `
        };
        const caPass = new ShaderPass(CARShader);
        composer.addPass(caPass);

        // Vignette
        const VigShader = {
            uniforms: {
                tDiffuse: { value: null },
                intensity: { value: 0.5 }
            },
            vertexShader: `varying vec2 vUv; void main() { vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
            fragmentShader: `
                uniform sampler2D tDiffuse;
                uniform float intensity;
                varying vec2 vUv;
                void main() {
                    vec4 color = texture2D(tDiffuse, vUv);
                    vec2 uv = vUv * 2.0 - 1.0;
                    float vig = 1.0 - dot(uv, uv) * intensity;
                    color.rgb *= clamp(vig, 0.0, 1.0);
                    gl_FragColor = color;
                }
            `
        };
        const vigPass = new ShaderPass(VigShader);
        composer.addPass(vigPass);
        composer.addPass(new OutputPass());

        // Effect toggles
        const effects = {
            radialBlur: true,
            shake: true,
            ca: true,
            vignette: true,
            speedLines: true
        };

        ['radialBlurToggle', 'shakeToggle', 'caToggle', 'vigToggle', 'speedLinesToggle'].forEach((id, i) => {
            const keys2 = ['radialBlur', 'shake', 'ca', 'vignette', 'speedLines'];
            document.getElementById(id).addEventListener('change', (e) => {
                effects[keys2[i]] = e.target.checked;
            });
        });

        // Speed lines (2D canvas overlay)
        const speedCanvas = document.createElement('canvas');
        speedCanvas.style.cssText = 'position:fixed;top:0;left:0;pointer-events:none;mix-blend-mode:screen;';
        speedCanvas.width = window.innerWidth;
        speedCanvas.height = window.innerHeight;
        document.body.appendChild(speedCanvas);
        const speedCtx = speedCanvas.getContext('2d');

        function drawSpeedLines(speed) {
            speedCtx.clearRect(0, 0, speedCanvas.width, speedCanvas.height);
            if (!effects.speedLines || speed < 20) return;
            
            const cx = speedCanvas.width / 2;
            const cy = speedCanvas.height / 2;
            const alpha = Math.min(1, (speed - 20) / 30) * 0.4;
            const numLines = Math.floor(speed * 0.5);
            
            speedCtx.strokeStyle = `rgba(255, 255, 255, ${alpha})`;
            
            for (let i = 0; i < numLines; i++) {
                const angle = (i / numLines) * Math.PI * 2 + Date.now() * 0.001;
                const r1 = 100 + Math.random() * 50;
                const r2 = r1 + 50 + Math.random() * 200 * (speed / 50);
                
                speedCtx.beginPath();
                speedCtx.lineWidth = Math.random() * 2;
                speedCtx.moveTo(cx + Math.cos(angle) * r1, cy + Math.sin(angle) * r1);
                speedCtx.lineTo(cx + Math.cos(angle) * r2, cy + Math.sin(angle) * r2);
                speedCtx.stroke();
            }
        }

        window.addEventListener('resize', () => {
            const w = window.innerWidth, h = window.innerHeight;
            camera.aspect = w / h;
            camera.updateProjectionMatrix();
            renderer.setSize(w, h);
            composer.setSize(w, h);
            speedCanvas.width = w;
            speedCanvas.height = h;
        });

        const clock = new THREE.Clock();
        let displaySpeed = 0;
        
        function animate() {
            requestAnimationFrame(animate);
            const dt = clock.getDelta();
            const time = clock.getElapsedTime();

            // Car controls
            if (keys['ArrowUp'] || keys['w']) {
                carState.speed = Math.min(carState.maxSpeed, carState.speed + carState.acceleration * dt);
            } else if (keys['ArrowDown'] || keys['s']) {
                carState.speed = Math.max(0, carState.speed - carState.braking * dt);
            } else {
                carState.speed = Math.max(0, carState.speed - carState.friction * dt);
            }
            
            if (keys[' ']) { // SPACE = brake
                carState.speed = Math.max(0, carState.speed - carState.braking * 2 * dt);
            }
            
            if (keys['r'] || keys['R']) {
                carState.position.set(0, 0, 5);
                carState.rotation = 0;
                carState.speed = 0;
            }
            
            const turnAmount = (carState.speed / carState.maxSpeed) * carState.turnSpeed * dt;
            if (keys['ArrowLeft'] || keys['a']) carState.rotation += turnAmount;
            if (keys['ArrowRight'] || keys['d']) carState.rotation -= turnAmount;
            
            // Move car
            carState.position.x += Math.sin(carState.rotation) * carState.speed * dt;
            carState.position.z -= Math.cos(carState.rotation) * carState.speed * dt;
            
            // Keep on track (loose constraint)
            carState.position.x = Math.max(-5.5, Math.min(5.5, carState.position.x));
            
            car.position.copy(carState.position);
            car.rotation.y = carState.rotation;
            
            // Wheel spin
            const spinSpeed = carState.speed * 0.2;
            wheels.forEach(w => { w.rotation.x += spinSpeed * dt; });
            
            // Camera follow
            const desiredCamOffset = new THREE.Vector3(
                -Math.sin(carState.rotation) * cameraState.offset.z,
                cameraState.offset.y,
                -Math.cos(carState.rotation) * cameraState.offset.z
            );
            const desiredCamPos = carState.position.clone().add(desiredCamOffset);
            camera.position.lerp(desiredCamPos, cameraState.smoothing);
            
            // Camera shake
            if (effects.shake && carState.speed > 30) {
                shakeIntensity = (carState.speed - 30) / 20 * 0.03;
                camera.position.x += (Math.random() - 0.5) * shakeIntensity;
                camera.position.y += (Math.random() - 0.5) * shakeIntensity;
            }
            
            camera.lookAt(carState.position.x, carState.position.y + 0.5, carState.position.z);
            
            // Speed-based effects
            const speedRatio = carState.speed / carState.maxSpeed;
            
            // Radial blur
            if (effects.radialBlur) {
                radialBlurPass.uniforms.strength.value = speedRatio * 0.08;
            } else {
                radialBlurPass.uniforms.strength.value = 0;
            }
            
            // Chromatic aberration
            if (effects.ca) {
                caPass.uniforms.offset.value = speedRatio * 0.006;
            } else {
                caPass.uniforms.offset.value = 0;
            }
            
            // Vignette (stronger at high speed)
            if (effects.vignette) {
                vigPass.uniforms.intensity.value = 0.3 + speedRatio * 0.4;
            } else {
                vigPass.uniforms.intensity.value = 0;
            }
            
            // Update speedometer
            displaySpeed = Math.round(carState.speed * 2); // scale to km/h
            document.getElementById('speed-num').textContent = displaySpeed;
            
            // Speed lines
            drawSpeedLines(displaySpeed);
            
            composer.render();
        }
        animate();
    </script>
</body>
</html>
```

## Temporal Anti-Aliasing (TAA)

TAA ใช้ข้อมูลจาก frame ก่อนหน้าเพื่อลด aliasing:

```javascript
// TAA Pass แบบง่าย
class SimpleTAAPass {
    constructor(renderer, width, height) {
        this.renderer = renderer;
        
        // สองตำแหน่งสลับกัน
        this.targets = [
            new THREE.WebGLRenderTarget(width, height, { type: THREE.HalfFloatType }),
            new THREE.WebGLRenderTarget(width, height, { type: THREE.HalfFloatType })
        ];
        
        this.currentTarget = 0;
        this.frameCount = 0;
        
        this.blendMaterial = new THREE.ShaderMaterial({
            uniforms: {
                tCurrent: { value: null },
                tPrevious: { value: null },
                blendFactor: { value: 0.1 }, // ผสม 10% ของ frame ใหม่
                jitterStrength: { value: 1.0 }
            },
            vertexShader: `
                varying vec2 vUv;
                void main() { vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }
            `,
            fragmentShader: `
                uniform sampler2D tCurrent;
                uniform sampler2D tPrevious;
                uniform float blendFactor;
                
                varying vec2 vUv;
                
                // Tonemap for better blending in HDR
                vec3 tonemap(vec3 c) { return c / (1.0 + c); }
                vec3 tonemapInverse(vec3 c) { return c / (1.0 - c); }
                
                void main() {
                    vec3 curr = tonemap(texture2D(tCurrent, vUv).rgb);
                    vec3 prev = tonemap(texture2D(tPrevious, vUv).rgb);
                    
                    // Clip previous color to current neighborhood (reduce ghosting)
                    vec3 minColor = min(curr, prev);
                    vec3 maxColor = max(curr, prev);
                    prev = clamp(prev, minColor, maxColor);
                    
                    vec3 result = mix(prev, curr, blendFactor);
                    gl_FragColor = vec4(tonemapInverse(result), 1.0);
                }
            `
        });
        
        // Jitter offsets (Halton sequence)
        this.jitterSequence = this._generateHalton(16);
        this.jitterIndex = 0;
    }
    
    _generateHalton(count) {
        const result = [];
        for (let i = 0; i < count; i++) {
            result.push(new THREE.Vector2(
                this._halton(i + 1, 2),
                this._halton(i + 1, 3)
            ));
        }
        return result;
    }
    
    _halton(index, base) {
        let f = 1.0, r = 0.0;
        while (index > 0) {
            f /= base;
            r += f * (index % base);
            index = Math.floor(index / base);
        }
        return r;
    }
    
    getJitter(width, height) {
        const j = this.jitterSequence[this.jitterIndex % this.jitterSequence.length];
        this.jitterIndex++;
        return new THREE.Vector2(
            (j.x - 0.5) / width,
            (j.y - 0.5) / height
        );
    }
    
    dispose() {
        this.targets[0].dispose();
        this.targets[1].dispose();
        this.blendMaterial.dispose();
    }
}
```

## ตัวอย่างที่ 2: Camera Shake Demo (เชิงเทคนิค)

```javascript
// Camera Shake แบบ Trauma System
class TraumaCameraShake {
    constructor(camera) {
        this.camera = camera;
        this.trauma = 0;           // 0-1
        this.maxTrauma = 1.0;
        this.traumaDecay = 0.8;    // per second
        
        this.maxYaw = 0.1;         // radians
        this.maxPitch = 0.05;
        this.maxRoll = 0.02;
        this.maxOffsetX = 0.1;
        this.maxOffsetY = 0.05;
        
        this.seed = Math.random() * 1000;
        this.basePosition = new THREE.Vector3();
        this.baseRotation = new THREE.Euler();
    }
    
    // ใช้ simplex noise หรือ value noise
    _noise(t) {
        // Simple smooth noise
        const i = Math.floor(t);
        const f = t - i;
        const u = f * f * (3 - 2 * f);
        const a = Math.sin(i * 127.1 + this.seed) * 43758.5453 % 1;
        const b = Math.sin((i + 1) * 127.1 + this.seed) * 43758.5453 % 1;
        return a + (b - a) * u;
    }
    
    addTrauma(amount) {
        this.trauma = Math.min(this.maxTrauma, this.trauma + amount);
    }
    
    update(deltaTime, time) {
        if (this.trauma <= 0) {
            this.camera.position.copy(this.basePosition);
            this.camera.rotation.copy(this.baseRotation);
            return;
        }
        
        const shake = this.trauma * this.trauma; // Quadratic for better feel
        const freq = 15.0;
        const t = time * freq;
        
        // Apply noise-based displacement
        this.camera.position.x = this.basePosition.x + 
            this.maxOffsetX * shake * (this._noise(t) * 2 - 1);
        this.camera.position.y = this.basePosition.y + 
            this.maxOffsetY * shake * (this._noise(t + 100) * 2 - 1);
        
        this.camera.rotation.x = this.baseRotation.x + 
            this.maxPitch * shake * (this._noise(t + 200) * 2 - 1);
        this.camera.rotation.y = this.baseRotation.y + 
            this.maxYaw * shake * (this._noise(t + 300) * 2 - 1);
        this.camera.rotation.z = this.baseRotation.z + 
            this.maxRoll * shake * (this._noise(t + 400) * 2 - 1);
        
        // Decay trauma
        this.trauma = Math.max(0, this.trauma - this.traumaDecay * deltaTime);
    }
    
    setBaseTransform(pos, rot) {
        this.basePosition.copy(pos);
        this.baseRotation.copy(rot);
    }
}
```

## Cinematic Effects Combo

การรวม effects ทั้งหมดเข้าด้วยกันสำหรับ cinematic experience:

```javascript
// Cinematic Effects Manager
class CinematicFX {
    constructor(composer, scene, camera) {
        this.composer = composer;
        this.scene = scene;
        this.camera = camera;
        
        this.passes = {};
        this.params = {
            motionBlurStrength: 0.0,
            chromaticAberration: 0.0,
            vignetteIntensity: 0.3,
            bloomStrength: 0.2,
            filmGrainStrength: 0.0,
            scanlines: false
        };
        
        this._setupPasses();
    }
    
    _setupPasses() {
        // Film grain shader
        const filmGrainShader = {
            uniforms: {
                tDiffuse: { value: null },
                time: { value: 0.0 },
                amount: { value: 0.0 }
            },
            vertexShader: `varying vec2 vUv; void main() { vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
            fragmentShader: `
                uniform sampler2D tDiffuse;
                uniform float time;
                uniform float amount;
                varying vec2 vUv;
                
                float rand(vec2 co) {
                    return fract(sin(dot(co, vec2(12.9898, 78.233))) * 43758.5453);
                }
                
                void main() {
                    vec4 color = texture2D(tDiffuse, vUv);
                    float noise = rand(vUv + time) * 2.0 - 1.0;
                    color.rgb += noise * amount;
                    gl_FragColor = color;
                }
            `
        };
        this.passes.filmGrain = new ShaderPass(filmGrainShader);
        this.composer.addPass(this.passes.filmGrain);
    }
    
    // Preset: Action scene
    setActionPreset() {
        this.params.motionBlurStrength = 0.06;
        this.params.chromaticAberration = 0.004;
        this.params.vignetteIntensity = 0.5;
        this.params.bloomStrength = 0.4;
        this.params.filmGrainStrength = 0.02;
    }
    
    // Preset: Horror
    setHorrorPreset() {
        this.params.chromaticAberration = 0.008;
        this.params.vignetteIntensity = 0.8;
        this.params.filmGrainStrength = 0.05;
    }
    
    // Preset: Dreamy
    setDreamyPreset() {
        this.params.motionBlurStrength = 0.02;
        this.params.bloomStrength = 0.8;
        this.params.vignetteIntensity = 0.2;
    }
    
    update(time) {
        if (this.passes.filmGrain) {
            this.passes.filmGrain.uniforms.time.value = time;
            this.passes.filmGrain.uniforms.amount.value = this.params.filmGrainStrength;
        }
    }
}
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Velocity Buffer Motion Blur
สร้าง motion blur ที่แท้จริงโดยใช้ velocity buffer:
- Render ฉากปกติ
- Render velocity ของแต่ละ pixel
- Apply motion blur โดยใช้ velocity

### แบบฝึกหัดที่ 2: เกมยิงปืนพร้อม Camera Shake
สร้าง demo เกม FPS ง่ายๆ ที่:
- กดคลิกซ้าย = ยิง → camera shake เล็กน้อย
- ถูกยิง → camera shake แรง
- วิ่ง = subtle motion blur

### แบบฝึกหัดที่ 3: Cinematic Package
สร้าง effects package ที่มี presets ต่างๆ:
- Night Vision (green tint + grain)
- Noir (black & white + vignette + grain)
- Retro (scanlines + aberration + vignette)
- Underwater (color shift + distortion)

### แบบฝึกหัดที่ 4: Slow Motion Effect
สร้าง "bullet time" effect:
- ลด time scale ลง
- เพิ่ม motion blur เมื่อ time scale ต่ำ
- Desaturate สีนิดหน่อย
- เพิ่ม film grain

---

**สรุปบทที่ 50:**
- Motion blur สร้างความสมจริงและ sense of speed
- Velocity buffer เป็น technique ที่ถูกต้องสำหรับ per-pixel motion blur
- Camera shake ด้วย trauma system ให้ผลที่ natural กว่า random shake
- Chromatic aberration, vignette, และ film grain รวมกันสร้าง cinematic feel
- ควร toggle effects ได้เพื่อ accessibility (บางคนเวียนหัวกับ motion effects)

**บทต่อไป:** SSAO & Ambient Occlusion
