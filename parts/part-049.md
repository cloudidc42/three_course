# Part 49: Depth of Field & Bokeh Effect

## ความรู้เบื้องต้นเกี่ยวกับ Depth of Field

Depth of Field (DoF) หรือ "ระยะชัดลึก" เป็นเอฟเฟกต์ทางการถ่ายภาพที่ทำให้วัตถุบางส่วนในฉากอยู่ในโฟกัส ในขณะที่ส่วนอื่นๆ เบลอหรือพร่ามัว เอฟเฟกต์นี้เกิดขึ้นในกล้องจริงเนื่องจากคุณสมบัติทางกายภาพของเลนส์ แต่ในโลกของกราฟิก 3 มิติ เราต้องจำลองมันขึ้นมาเอง

การใช้ DoF ในเกมและแอปพลิเคชัน 3D ช่วยสร้าง:
- **ความสมจริง** - ทำให้ภาพดูเหมือนถ่ายด้วยกล้องจริง
- **การนำสายตา** - ดึงความสนใจไปยังวัตถุที่ต้องการเน้น
- **ความลึกในฉาก** - ทำให้รู้สึกว่าฉากมีระยะลึกมากขึ้น
- **บรรยากาศ** - สร้างอารมณ์ที่เป็นเอกลักษณ์ของภาพยนตร์

## Circle of Confusion (CoC) คืออะไร?

Circle of Confusion (CoC) หรือ "วงกลมความสับสน" เป็นแนวคิดพื้นฐานของ Depth of Field เมื่อแสงจากจุดหนึ่งผ่านเลนส์ แสงจะโฟกัสที่จุดหนึ่งบนฟิล์มหรือเซนเซอร์ ถ้าวัตถุอยู่ในระยะโฟกัสพอดี แสงจะสร้างจุดเล็กๆ แต่ถ้าวัตถุอยู่นอกระยะโฟกัส แสงจะสร้างวงกลมแทน - นั่นคือ CoC

ขนาดของ CoC บอกเราว่าวัตถุนั้น "เบลอ" แค่ไหน:
- CoC ขนาดเล็ก = อยู่ในโฟกัส (หรือใกล้โฟกัส)
- CoC ขนาดใหญ่ = อยู่นอกโฟกัสมาก (เบลอมาก)

## Bokeh คืออะไร?

Bokeh (โบเก้) มาจากคำญี่ปุ่น "暈け" (boke) แปลว่า "ความพร่ามัว" หรือ "หมอก" ใน context ของการถ่ายภาพ bokeh หมายถึงคุณภาพหรือลักษณะของส่วนที่เบลอในภาพ โดยเฉพาะแสงเล็กๆ ที่กลายเป็นวงกลมหรือรูปทรงต่างๆ

ลักษณะของ bokeh ขึ้นอยู่กับ:
- **รูรับแสง (Aperture)** - รูรับแสงกว้างทำให้ bokeh มาก
- **ระยะโฟกัส (Focus Distance)** - ยิ่งไกลจากระยะโฟกัส ยิ่งเบลอมาก
- **รูปทรงของ diaphragm** - กำหนดรูปร่างของ bokeh balls

## การติดตั้งและ Setup เบื้องต้น

```javascript
// การติดตั้ง dependencies ที่จำเป็น
// npm install three @types/three

import * as THREE from 'three';
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';
import { BokehPass } from 'three/examples/jsm/postprocessing/BokehPass.js';
import { OutputPass } from 'three/examples/jsm/postprocessing/OutputPass.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
```

## BokehPass - การใช้งานหลัก

BokehPass เป็น post-processing pass ที่มาพร้อมกับ Three.js examples ใช้เทคนิค depth of field แบบง่ายๆ

```javascript
// พารามิเตอร์ของ BokehPass
const bokehParams = {
    focus: 1.0,        // ระยะโฟกัส (0-3 หรือมากกว่า)
    aperture: 0.025,   // รูรับแสง (ยิ่งมาก ยิ่งเบลอมาก)
    maxblur: 0.01,     // ขอบเขตสูงสุดของการเบลอ
    width: window.innerWidth,
    height: window.innerHeight
};

const bokehPass = new BokehPass(scene, camera, bokehParams);
composer.addPass(bokehPass);
```

## ตัวอย่างที่ 1: Photography DoF Scene

ฉากจำลองการถ่ายภาพมาโครที่มี depth of field effect:

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Photography Depth of Field</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; font-family: Arial, sans-serif; }
        canvas { display: block; }
        #controls {
            position: fixed;
            top: 20px;
            right: 20px;
            background: rgba(0,0,0,0.8);
            color: white;
            padding: 20px;
            border-radius: 10px;
            width: 250px;
        }
        #controls h3 { margin-bottom: 15px; color: #4fc3f7; }
        .param-group { margin-bottom: 15px; }
        .param-group label { display: block; margin-bottom: 5px; font-size: 13px; }
        .param-group input[type="range"] { width: 100%; }
        .param-value { float: right; color: #4fc3f7; }
        #info {
            position: fixed;
            bottom: 20px;
            left: 20px;
            color: white;
            background: rgba(0,0,0,0.7);
            padding: 10px 15px;
            border-radius: 8px;
            font-size: 14px;
        }
    </style>
</head>
<body>
    <div id="controls">
        <h3>🎥 Depth of Field Controls</h3>
        <div class="param-group">
            <label>Focus Distance: <span class="param-value" id="focusVal">1.0</span></label>
            <input type="range" id="focus" min="0.1" max="5.0" step="0.01" value="1.0">
        </div>
        <div class="param-group">
            <label>Aperture: <span class="param-value" id="apertureVal">0.025</span></label>
            <input type="range" id="aperture" min="0.0001" max="0.1" step="0.0001" value="0.025">
        </div>
        <div class="param-group">
            <label>Max Blur: <span class="param-value" id="maxblurVal">0.010</span></label>
            <input type="range" id="maxblur" min="0.0" max="0.05" step="0.001" value="0.01">
        </div>
    </div>
    <div id="info">กดลูกศรซ้าย/ขวา เพื่อเปลี่ยนโฟกัสอัตโนมัติ</div>

    <script type="module">
        import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js';
        import { EffectComposer } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/EffectComposer.js';
        import { RenderPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/RenderPass.js';
        import { BokehPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/BokehPass.js';
        import { OutputPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/OutputPass.js';
        import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/controls/OrbitControls.js';

        // Setup
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.setPixelRatio(window.devicePixelRatio);
        renderer.shadowMap.enabled = true;
        renderer.shadowMap.type = THREE.PCFSoftShadowMap;
        document.body.appendChild(renderer.domElement);

        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x1a1a2e);
        scene.fog = new THREE.Fog(0x1a1a2e, 5, 25);

        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
        camera.position.set(0, 1, 5);

        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.dampingFactor = 0.05;

        // สร้างแสง
        const ambientLight = new THREE.AmbientLight(0x404040, 0.5);
        scene.add(ambientLight);

        const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
        directionalLight.position.set(5, 10, 5);
        directionalLight.castShadow = true;
        directionalLight.shadow.mapSize.width = 2048;
        directionalLight.shadow.mapSize.height = 2048;
        scene.add(directionalLight);

        // สร้างพื้น
        const groundGeo = new THREE.PlaneGeometry(30, 30);
        const groundMat = new THREE.MeshStandardMaterial({
            color: 0x2d5a27,
            roughness: 0.9,
            metalness: 0.0
        });
        const ground = new THREE.Mesh(groundGeo, groundMat);
        ground.rotation.x = -Math.PI / 2;
        ground.receiveShadow = true;
        scene.add(ground);

        // สร้างดอกไม้แถวหนึ่ง (Objects ที่ระยะต่างๆ)
        const flowers = [];
        const flowerColors = [0xff6b6b, 0xffd93d, 0x6bcb77, 0x4d96ff, 0xff9f43, 0xee5a24];
        
        for (let i = 0; i < 12; i++) {
            const group = new THREE.Group();
            
            // ก้านดอก
            const stemGeo = new THREE.CylinderGeometry(0.02, 0.02, 0.8, 8);
            const stemMat = new THREE.MeshStandardMaterial({ color: 0x2d5a27 });
            const stem = new THREE.Mesh(stemGeo, stemMat);
            stem.position.y = 0.4;
            stem.castShadow = true;
            group.add(stem);
            
            // กลีบดอก
            const petalColor = flowerColors[i % flowerColors.length];
            for (let p = 0; p < 6; p++) {
                const petalGeo = new THREE.SphereGeometry(0.15, 8, 8);
                petalGeo.scale(1, 0.3, 0.5);
                const petalMat = new THREE.MeshStandardMaterial({ color: petalColor });
                const petal = new THREE.Mesh(petalGeo, petalMat);
                const angle = (p / 6) * Math.PI * 2;
                petal.position.set(Math.cos(angle) * 0.2, 0.82, Math.sin(angle) * 0.2);
                petal.rotation.y = -angle;
                petal.castShadow = true;
                group.add(petal);
            }
            
            // ดอกกลาง
            const centerGeo = new THREE.SphereGeometry(0.08, 8, 8);
            const centerMat = new THREE.MeshStandardMaterial({ color: 0xffd700 });
            const center = new THREE.Mesh(centerGeo, centerMat);
            center.position.y = 0.85;
            group.add(center);
            
            // จัดตำแหน่ง
            group.position.set(
                (i - 5.5) * 0.8,
                0,
                (Math.random() - 0.5) * 0.3
            );
            
            scene.add(group);
            flowers.push(group);
        }

        // หญ้า
        for (let i = 0; i < 200; i++) {
            const grassGeo = new THREE.PlaneGeometry(0.1, 0.4);
            const grassMat = new THREE.MeshStandardMaterial({
                color: new THREE.Color(0.1, 0.4 + Math.random() * 0.2, 0.1),
                side: THREE.DoubleSide
            });
            const grass = new THREE.Mesh(grassGeo, grassMat);
            grass.position.set(
                (Math.random() - 0.5) * 20,
                0.2,
                (Math.random() - 0.5) * 20
            );
            grass.rotation.y = Math.random() * Math.PI;
            scene.add(grass);
        }

        // Post-processing
        const composer = new EffectComposer(renderer);
        const renderPass = new RenderPass(scene, camera);
        composer.addPass(renderPass);

        const bokehParams = {
            focus: 1.0,
            aperture: 0.025,
            maxblur: 0.01,
            width: window.innerWidth,
            height: window.innerHeight
        };

        const bokehPass = new BokehPass(scene, camera, bokehParams);
        composer.addPass(bokehPass);

        const outputPass = new OutputPass();
        composer.addPass(outputPass);

        // UI Controls
        function updateControl(id, paramName, displayId, digits = 3) {
            const slider = document.getElementById(id);
            const display = document.getElementById(displayId);
            slider.addEventListener('input', () => {
                const val = parseFloat(slider.value);
                bokehPass.uniforms[paramName].value = val;
                display.textContent = val.toFixed(digits);
            });
        }

        updateControl('focus', 'focus', 'focusVal', 2);
        updateControl('aperture', 'aperture', 'apertureVal', 4);
        updateControl('maxblur', 'maxblur', 'maxblurVal', 3);

        // Auto focus animation
        let autoFocus = false;
        let focusTarget = 0;
        
        document.addEventListener('keydown', (e) => {
            if (e.key === 'ArrowLeft') {
                focusTarget = Math.max(0, focusTarget - 1);
                animateFocusToFlower(focusTarget);
            } else if (e.key === 'ArrowRight') {
                focusTarget = Math.min(flowers.length - 1, focusTarget + 1);
                animateFocusToFlower(focusTarget);
            }
        });

        function animateFocusToFlower(index) {
            const flower = flowers[index];
            const distance = camera.position.distanceTo(flower.position);
            // อัพเดท focus slider
            const focusSlider = document.getElementById('focus');
            focusSlider.value = distance;
            document.getElementById('focusVal').textContent = distance.toFixed(2);
            bokehPass.uniforms['focus'].value = distance;
        }

        // Resize
        window.addEventListener('resize', () => {
            const w = window.innerWidth;
            const h = window.innerHeight;
            camera.aspect = w / h;
            camera.updateProjectionMatrix();
            renderer.setSize(w, h);
            composer.setSize(w, h);
        });

        // Animation loop
        const clock = new THREE.Clock();
        function animate() {
            requestAnimationFrame(animate);
            const time = clock.getElapsedTime();
            
            // Sway flowers slightly
            flowers.forEach((flower, i) => {
                flower.rotation.z = Math.sin(time * 0.5 + i * 0.5) * 0.05;
            });
            
            controls.update();
            composer.render();
        }
        animate();
    </script>
</body>
</html>
```

## ความเข้าใจเรื่อง Focus Distance และ Blur Radius

### Focus Distance
ระยะโฟกัส (Focus Distance) คือระยะห่างจากกล้องที่วัตถุจะอยู่ในโฟกัสชัดที่สุด ใน BokehPass ค่านี้คือ `focus`:

```javascript
// การคำนวณ focus distance จากตำแหน่ง camera และ target
function calculateFocusDistance(camera, targetObject) {
    const cameraPos = camera.position;
    const targetPos = targetObject.position;
    const distance = cameraPos.distanceTo(targetPos);
    return distance;
}

// การใช้ raycasting เพื่อหา focus point อัตโนมัติ
function autoFocusWithRaycast(camera, scene, mouse) {
    const raycaster = new THREE.Raycaster();
    raycaster.setFromCamera(mouse, camera);
    
    const intersects = raycaster.intersectObjects(scene.children, true);
    if (intersects.length > 0) {
        const focusDistance = intersects[0].distance;
        bokehPass.uniforms['focus'].value = focusDistance;
    }
}
```

### Aperture (รูรับแสง)
รูรับแสงควบคุมขนาดของ CoC ยิ่ง aperture มาก ยิ่งเบลอมาก:

```javascript
// F-stop to aperture conversion (ประมาณ)
// F/1.4 = aperture สูง (bokeh มาก)
// F/16 = aperture ต่ำ (bokeh น้อย)
function fStopToAperture(fStop) {
    // Scale สำหรับ Three.js BokehPass
    return 0.1 / fStop;
}

bokehPass.uniforms['aperture'].value = fStopToAperture(2.8); // F/2.8
```

## DoFPass แบบ Custom - การสร้างเอง

เมื่อต้องการควบคุม DoF มากขึ้น สามารถสร้าง custom pass ได้:

```javascript
// Custom Depth of Field using Render Targets
class CustomDoFPass {
    constructor(renderer, scene, camera, width, height) {
        this.renderer = renderer;
        this.scene = scene;
        this.camera = camera;
        
        // Render targets
        this.colorTarget = new THREE.WebGLRenderTarget(width, height, {
            minFilter: THREE.LinearFilter,
            magFilter: THREE.LinearFilter,
            format: THREE.RGBAFormat,
            type: THREE.HalfFloatType
        });
        
        this.depthTarget = new THREE.WebGLRenderTarget(width, height, {
            minFilter: THREE.LinearFilter,
            magFilter: THREE.LinearFilter,
            format: THREE.RGBAFormat,
            type: THREE.UnsignedByteType
        });
        this.depthTarget.depthTexture = new THREE.DepthTexture(width, height);
        this.depthTarget.depthTexture.format = THREE.DepthFormat;
        this.depthTarget.depthTexture.type = THREE.UnsignedIntType;
        
        // DoF shader
        this.dofMaterial = new THREE.ShaderMaterial({
            uniforms: {
                tColor: { value: null },
                tDepth: { value: null },
                focus: { value: 1.0 },
                aperture: { value: 0.025 },
                maxBlur: { value: 0.01 },
                nearClip: { value: camera.near },
                farClip: { value: camera.far },
                texelSize: { value: new THREE.Vector2(1/width, 1/height) }
            },
            vertexShader: `
                varying vec2 vUv;
                void main() {
                    vUv = uv;
                    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
                }
            `,
            fragmentShader: `
                uniform sampler2D tColor;
                uniform sampler2D tDepth;
                uniform float focus;
                uniform float aperture;
                uniform float maxBlur;
                uniform float nearClip;
                uniform float farClip;
                uniform vec2 texelSize;
                
                varying vec2 vUv;
                
                // แปลง depth buffer value เป็น linear depth
                float linearizeDepth(float depth) {
                    float z = depth * 2.0 - 1.0;
                    return (2.0 * nearClip * farClip) / (farClip + nearClip - z * (farClip - nearClip));
                }
                
                // คำนวณ CoC (Circle of Confusion)
                float calculateCoC(float depth, float focusDist, float aperture) {
                    float coc = abs(depth - focusDist) / focusDist;
                    return coc * aperture;
                }
                
                void main() {
                    vec4 color = texture2D(tColor, vUv);
                    float depth = texture2D(tDepth, vUv).r;
                    float linearDepth = linearizeDepth(depth);
                    
                    float coc = calculateCoC(linearDepth, focus, aperture);
                    coc = clamp(coc, 0.0, maxBlur);
                    
                    // Blur โดยใช้ circle sampling
                    vec4 blurred = vec4(0.0);
                    float totalWeight = 0.0;
                    
                    // 16-sample Poisson disk
                    vec2 poissonDisk[16];
                    poissonDisk[0] = vec2(-0.94201624, -0.39906216);
                    poissonDisk[1] = vec2(0.94558609, -0.76890725);
                    poissonDisk[2] = vec2(-0.094184101, -0.92938870);
                    poissonDisk[3] = vec2(0.34495938, 0.29387760);
                    poissonDisk[4] = vec2(-0.91588581, 0.45771432);
                    poissonDisk[5] = vec2(-0.81544232, -0.87912464);
                    poissonDisk[6] = vec2(-0.38277543, 0.27676845);
                    poissonDisk[7] = vec2(0.97484398, 0.75648379);
                    poissonDisk[8] = vec2(0.44323325, -0.97511554);
                    poissonDisk[9] = vec2(0.53742981, -0.47373420);
                    poissonDisk[10] = vec2(-0.26496911, -0.41893023);
                    poissonDisk[11] = vec2(0.79197514, 0.19090188);
                    poissonDisk[12] = vec2(-0.24188840, 0.99706507);
                    poissonDisk[13] = vec2(-0.81409955, 0.91437590);
                    poissonDisk[14] = vec2(0.19984126, 0.78641367);
                    poissonDisk[15] = vec2(0.14383161, -0.14100790);
                    
                    for (int i = 0; i < 16; i++) {
                        vec2 offset = poissonDisk[i] * coc;
                        vec2 sampleUv = vUv + offset;
                        
                        // Clamp to texture bounds
                        sampleUv = clamp(sampleUv, vec2(0.0), vec2(1.0));
                        
                        float sampleDepth = texture2D(tDepth, sampleUv).r;
                        float sampleLinear = linearizeDepth(sampleDepth);
                        float sampleCoc = calculateCoC(sampleLinear, focus, aperture);
                        
                        // ป้องกัน foreground bleeding
                        float weight = sampleCoc >= coc ? 1.0 : sampleCoc / coc;
                        
                        blurred += texture2D(tColor, sampleUv) * weight;
                        totalWeight += weight;
                    }
                    
                    blurred /= totalWeight;
                    
                    // Blend ระหว่าง sharp และ blurred ตาม CoC
                    float blendFactor = clamp(coc / maxBlur, 0.0, 1.0);
                    gl_FragColor = mix(color, blurred, blendFactor);
                }
            `
        });
    }
    
    render() {
        // Render scene to color target
        this.renderer.setRenderTarget(this.depthTarget);
        this.renderer.render(this.scene, this.camera);
        
        // Apply DoF
        this.dofMaterial.uniforms.tColor.value = this.depthTarget.texture;
        this.dofMaterial.uniforms.tDepth.value = this.depthTarget.depthTexture;
        
        // Render quad
        this.renderer.setRenderTarget(null);
    }
    
    dispose() {
        this.colorTarget.dispose();
        this.depthTarget.dispose();
        this.dofMaterial.dispose();
    }
}
```

## Tilt-Shift Effect

Tilt-shift เป็น DoF effect พิเศษที่สร้าง gradient blur จากบนลงล่าง ทำให้ภาพดูเหมือนถ่ายภาพย่อส่วน:

```javascript
// Tilt-Shift Shader
const tiltShiftMaterial = new THREE.ShaderMaterial({
    uniforms: {
        tDiffuse: { value: null },
        focus: { value: 0.5 },     // ตำแหน่งกลางของ sharp area (0-1)
        blurRadius: { value: 7.0 }, // ขนาด blur
        gradientStrength: { value: 0.4 } // ความแรงของ gradient
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
        uniform float focus;
        uniform float blurRadius;
        uniform float gradientStrength;
        
        varying vec2 vUv;
        
        vec4 gaussianBlur(sampler2D tex, vec2 uv, vec2 direction, float radius) {
            vec4 result = vec4(0.0);
            float total = 0.0;
            
            for (float i = -radius; i <= radius; i++) {
                float weight = exp(-0.5 * (i / radius) * (i / radius));
                vec2 offset = direction * i / vec2(float(textureSize(tex, 0)));
                result += texture2D(tex, uv + offset) * weight;
                total += weight;
            }
            return result / total;
        }
        
        void main() {
            // คำนวณระยะห่างจาก focus center
            float dist = abs(vUv.y - focus);
            
            // Blur amount ตาม distance
            float blurAmount = smoothstep(0.0, gradientStrength, dist);
            float currentBlur = blurAmount * blurRadius;
            
            // Horizontal blur
            vec4 blurredH = gaussianBlur(tDiffuse, vUv, vec2(1.0, 0.0), currentBlur);
            // Vertical blur (ใช้ 2-pass blur สำหรับประสิทธิภาพดีกว่า)
            vec4 blurredV = gaussianBlur(tDiffuse, vUv, vec2(0.0, 1.0), currentBlur);
            vec4 blurred = mix(blurredH, blurredV, 0.5);
            
            vec4 sharp = texture2D(tDiffuse, vUv);
            gl_FragColor = mix(sharp, blurred, blurAmount);
        }
    `
});
```

## ตัวอย่างที่ 2: Miniature World Effect (ผลกระทบโลกจิ๋ว)

ผสานเทคนิค tilt-shift เพื่อสร้างเอฟเฟกต์โลกจิ๋ว ทำให้ฉากขนาดใหญ่ดูเหมือนโมเดลขนาดเล็ก:

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Miniature World Effect</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; font-family: Arial, sans-serif; }
        canvas { display: block; }
        #controls {
            position: fixed;
            top: 20px;
            left: 20px;
            background: rgba(0,0,0,0.85);
            color: white;
            padding: 20px;
            border-radius: 10px;
            width: 280px;
        }
        #controls h3 { margin-bottom: 15px; color: #ff6b6b; font-size: 16px; }
        .ctrl { margin-bottom: 12px; }
        .ctrl label { display: flex; justify-content: space-between; font-size: 12px; margin-bottom: 4px; }
        .ctrl input { width: 100%; }
        .toggle-btn {
            display: block;
            width: 100%;
            padding: 8px;
            background: #ff6b6b;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            margin-top: 10px;
            font-size: 13px;
        }
    </style>
</head>
<body>
    <div id="controls">
        <h3>🏙️ Miniature World Effect</h3>
        <div class="ctrl">
            <label><span>Focus Point Y</span><span id="fpVal">0.50</span></label>
            <input type="range" id="focusPoint" min="0" max="1" step="0.01" value="0.5">
        </div>
        <div class="ctrl">
            <label><span>Blur Strength</span><span id="bsVal">8.0</span></label>
            <input type="range" id="blurStrength" min="0" max="20" step="0.5" value="8">
        </div>
        <div class="ctrl">
            <label><span>Focus Band Width</span><span id="fbwVal">0.15</span></label>
            <input type="range" id="focusBand" min="0.01" max="0.5" step="0.01" value="0.15">
        </div>
        <div class="ctrl">
            <label><span>Saturation Boost</span><span id="satVal">1.5</span></label>
            <input type="range" id="saturation" min="1.0" max="3.0" step="0.1" value="1.5">
        </div>
        <button class="toggle-btn" id="toggleEffect">Toggle Miniature Effect</button>
        <button class="toggle-btn" id="toggleRotate" style="background:#4fc3f7">Toggle Auto-Rotate</button>
    </div>

    <script type="module">
        import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js';
        import { EffectComposer } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/EffectComposer.js';
        import { RenderPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/RenderPass.js';
        import { ShaderPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/ShaderPass.js';
        import { OutputPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/OutputPass.js';

        const W = window.innerWidth, H = window.innerHeight;
        
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(W, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        renderer.shadowMap.enabled = true;
        document.body.appendChild(renderer.domElement);

        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x87ceeb);
        scene.fog = new THREE.Fog(0x87ceeb, 20, 80);

        // กล้องมุมสูง เหมือนมองจากด้านบน (มุมเหมาะสำหรับ tilt-shift)
        const camera = new THREE.PerspectiveCamera(50, W/H, 0.1, 200);
        camera.position.set(0, 20, 25);
        camera.lookAt(0, 0, 0);

        // Lighting
        const sunLight = new THREE.DirectionalLight(0xfffde7, 1.5);
        sunLight.position.set(10, 20, 10);
        sunLight.castShadow = true;
        sunLight.shadow.mapSize.set(2048, 2048);
        sunLight.shadow.camera.near = 0.5;
        sunLight.shadow.camera.far = 100;
        sunLight.shadow.camera.left = -30;
        sunLight.shadow.camera.right = 30;
        sunLight.shadow.camera.top = 30;
        sunLight.shadow.camera.bottom = -30;
        scene.add(sunLight);
        scene.add(new THREE.AmbientLight(0x87ceeb, 0.5));

        // สร้างเมือง (City)
        function createCity() {
            const cityGroup = new THREE.Group();
            
            // พื้นถนน
            const roadMat = new THREE.MeshStandardMaterial({ color: 0x555555 });
            const sidewalkMat = new THREE.MeshStandardMaterial({ color: 0x999999 });
            
            // Grid ของถนน
            for (let x = -15; x <= 15; x += 5) {
                const roadV = new THREE.Mesh(
                    new THREE.BoxGeometry(1, 0.05, 30),
                    roadMat
                );
                roadV.position.set(x, 0.02, 0);
                roadV.receiveShadow = true;
                cityGroup.add(roadV);
            }
            for (let z = -15; z <= 15; z += 5) {
                const roadH = new THREE.Mesh(
                    new THREE.BoxGeometry(30, 0.05, 1),
                    roadMat
                );
                roadH.position.set(0, 0.02, z);
                roadH.receiveShadow = true;
                cityGroup.add(roadH);
            }
            
            // สร้างอาคาร
            const buildingColors = [
                0x795548, 0x607d8b, 0x455a64, 0x37474f,
                0x546e7a, 0x4e342e, 0x6d4c41
            ];
            
            for (let x = -12; x <= 12; x += 5) {
                for (let z = -12; z <= 12; z += 5) {
                    // หลีกเลี่ยงตำแหน่งบนถนน
                    if (Math.abs(x) % 5 === 0 || Math.abs(z) % 5 === 0) {
                        const numBuildings = 1 + Math.floor(Math.random() * 3);
                        for (let b = 0; b < numBuildings; b++) {
                            const w = 1 + Math.random() * 2;
                            const d = 1 + Math.random() * 2;
                            const h = 1 + Math.random() * 8;
                            const color = buildingColors[Math.floor(Math.random() * buildingColors.length)];
                            
                            const bGeo = new THREE.BoxGeometry(w, h, d);
                            const bMat = new THREE.MeshStandardMaterial({ color });
                            const building = new THREE.Mesh(bGeo, bMat);
                            building.position.set(
                                x + (Math.random() - 0.5) * 2,
                                h / 2,
                                z + (Math.random() - 0.5) * 2
                            );
                            building.castShadow = true;
                            building.receiveShadow = true;
                            cityGroup.add(building);
                            
                            // Windows
                            for (let wy = 0; wy < h; wy += 0.6) {
                                for (let wx = 0; wx < 3; wx++) {
                                    if (Math.random() > 0.4) {
                                        const winGeo = new THREE.BoxGeometry(0.2, 0.2, 0.01);
                                        const winMat = new THREE.MeshStandardMaterial({
                                            color: Math.random() > 0.3 ? 0xffff88 : 0x333333,
                                            emissive: Math.random() > 0.3 ? 0xffff44 : 0x000000,
                                            emissiveIntensity: 0.3
                                        });
                                        const win = new THREE.Mesh(winGeo, winMat);
                                        win.position.set(
                                            building.position.x + (wx - 1) * 0.5,
                                            wy + 0.3,
                                            building.position.z + d / 2 + 0.01
                                        );
                                        cityGroup.add(win);
                                    }
                                }
                            }
                        }
                    }
                }
            }
            
            // ต้นไม้
            for (let i = 0; i < 40; i++) {
                const treeGroup = new THREE.Group();
                const trunkGeo = new THREE.CylinderGeometry(0.05, 0.08, 0.5, 6);
                const trunkMat = new THREE.MeshStandardMaterial({ color: 0x5d4037 });
                const trunk = new THREE.Mesh(trunkGeo, trunkMat);
                trunk.position.y = 0.25;
                treeGroup.add(trunk);
                
                const leavesGeo = new THREE.SphereGeometry(0.3, 8, 6);
                const leavesMat = new THREE.MeshStandardMaterial({
                    color: new THREE.Color(
                        0.1 + Math.random() * 0.1,
                        0.4 + Math.random() * 0.3,
                        0.1
                    )
                });
                const leaves = new THREE.Mesh(leavesGeo, leavesMat);
                leaves.position.y = 0.7;
                leaves.castShadow = true;
                treeGroup.add(leaves);
                
                treeGroup.position.set(
                    (Math.random() - 0.5) * 25,
                    0,
                    (Math.random() - 0.5) * 25
                );
                cityGroup.add(treeGroup);
            }
            
            // รถ
            for (let i = 0; i < 20; i++) {
                const carGroup = new THREE.Group();
                const bodyGeo = new THREE.BoxGeometry(0.6, 0.2, 0.35);
                const carColors = [0xff4444, 0x4444ff, 0x44ff44, 0xffff44, 0xff44ff];
                const bodyMat = new THREE.MeshStandardMaterial({
                    color: carColors[Math.floor(Math.random() * carColors.length)]
                });
                const body = new THREE.Mesh(bodyGeo, bodyMat);
                body.castShadow = true;
                carGroup.add(body);
                
                // ตำแหน่งบนถนน
                const onXRoad = Math.random() > 0.5;
                if (onXRoad) {
                    carGroup.position.set(
                        (Math.random() - 0.5) * 28,
                        0.12,
                        Math.round((Math.random() - 0.5) * 6) * 5
                    );
                } else {
                    carGroup.position.set(
                        Math.round((Math.random() - 0.5) * 6) * 5,
                        0.12,
                        (Math.random() - 0.5) * 28
                    );
                    carGroup.rotation.y = Math.PI / 2;
                }
                cityGroup.add(carGroup);
            }
            
            // พื้นสีเขียว
            const groundGeo = new THREE.PlaneGeometry(60, 60);
            const groundMat = new THREE.MeshStandardMaterial({ color: 0x4caf50 });
            const ground = new THREE.Mesh(groundGeo, groundMat);
            ground.rotation.x = -Math.PI / 2;
            ground.receiveShadow = true;
            cityGroup.add(ground);
            
            return cityGroup;
        }

        const city = createCity();
        scene.add(city);

        // Tilt-Shift Shader
        const TiltShiftShader = {
            uniforms: {
                tDiffuse: { value: null },
                focusPoint: { value: 0.5 },
                blurStrength: { value: 8.0 },
                focusBand: { value: 0.15 },
                saturation: { value: 1.5 },
                enabled: { value: 1 }
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
                uniform float focusPoint;
                uniform float blurStrength;
                uniform float focusBand;
                uniform float saturation;
                uniform int enabled;
                
                varying vec2 vUv;
                
                vec3 adjustSaturation(vec3 color, float sat) {
                    float grey = dot(color, vec3(0.299, 0.587, 0.114));
                    return mix(vec3(grey), color, sat);
                }
                
                vec4 blur(sampler2D tex, vec2 uv, float amount) {
                    if (amount < 0.5) return texture2D(tex, uv);
                    
                    vec4 sum = vec4(0.0);
                    float count = 0.0;
                    float step = 1.0 / float(textureSize(tex, 0).y);
                    
                    for (float i = -4.0; i <= 4.0; i += 1.0) {
                        for (float j = -4.0; j <= 4.0; j += 1.0) {
                            float w = exp(-(i*i + j*j) / (2.0 * amount));
                            sum += texture2D(tex, uv + vec2(j, i) * step * amount) * w;
                            count += w;
                        }
                    }
                    return sum / count;
                }
                
                void main() {
                    vec4 color = texture2D(tDiffuse, vUv);
                    
                    if (enabled == 0) {
                        gl_FragColor = color;
                        return;
                    }
                    
                    float dist = abs(vUv.y - focusPoint);
                    float blurMask = smoothstep(0.0, focusBand, dist - focusBand * 0.5);
                    float blurAmt = blurMask * blurStrength;
                    
                    vec4 blurred = blur(tDiffuse, vUv, blurAmt);
                    vec4 result = mix(color, blurred, blurMask);
                    
                    // Saturation boost (ทำให้ดูเหมือนของเล่น)
                    result.rgb = adjustSaturation(result.rgb, saturation);
                    
                    // เพิ่ม warmth เล็กน้อย
                    result.rgb *= vec3(1.05, 1.02, 0.98);
                    
                    gl_FragColor = result;
                }
            `
        };

        // Post-processing
        const composer = new EffectComposer(renderer);
        composer.addPass(new RenderPass(scene, camera));
        
        const tiltShiftPass = new ShaderPass(TiltShiftShader);
        composer.addPass(tiltShiftPass);
        composer.addPass(new OutputPass());

        // UI Controls
        const params = { enabled: 1, autoRotate: true };
        
        document.getElementById('focusPoint').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            tiltShiftPass.uniforms.focusPoint.value = v;
            document.getElementById('fpVal').textContent = v.toFixed(2);
        });
        document.getElementById('blurStrength').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            tiltShiftPass.uniforms.blurStrength.value = v;
            document.getElementById('bsVal').textContent = v.toFixed(1);
        });
        document.getElementById('focusBand').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            tiltShiftPass.uniforms.focusBand.value = v;
            document.getElementById('fbwVal').textContent = v.toFixed(2);
        });
        document.getElementById('saturation').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            tiltShiftPass.uniforms.saturation.value = v;
            document.getElementById('satVal').textContent = v.toFixed(1);
        });
        document.getElementById('toggleEffect').addEventListener('click', () => {
            params.enabled = 1 - params.enabled;
            tiltShiftPass.uniforms.enabled.value = params.enabled;
        });
        document.getElementById('toggleRotate').addEventListener('click', () => {
            params.autoRotate = !params.autoRotate;
        });

        window.addEventListener('resize', () => {
            const w = window.innerWidth, h = window.innerHeight;
            camera.aspect = w / h;
            camera.updateProjectionMatrix();
            renderer.setSize(w, h);
            composer.setSize(w, h);
        });

        const clock = new THREE.Clock();
        function animate() {
            requestAnimationFrame(animate);
            const t = clock.getElapsedTime();
            
            if (params.autoRotate) {
                camera.position.x = Math.sin(t * 0.1) * 30;
                camera.position.z = Math.cos(t * 0.1) * 30;
                camera.position.y = 20;
                camera.lookAt(0, 0, 0);
            }
            
            composer.render();
        }
        animate();
    </script>
</body>
</html>
```

## การตั้งค่า BokehPass Parameters

การปรับค่าต่างๆ ของ BokehPass อย่างละเอียด:

```javascript
// ตัวอย่างการตั้งค่าสำหรับสถานการณ์ต่างๆ

// Portrait Photography (ถ่ายภาพบุคคล)
const portraitDOF = {
    focus: 2.5,        // โฟกัสที่ระยะ 2.5 หน่วย
    aperture: 0.05,    // รูรับแสงกว้าง (F/2 equivalent)
    maxblur: 0.02      // blur ปานกลาง
};

// Macro Photography (ถ่ายภาพมาโคร)
const macroDOF = {
    focus: 0.5,        // โฟกัสใกล้มาก
    aperture: 0.08,    // รูรับแสงกว้างมาก
    maxblur: 0.025     // blur มาก
};

// Landscape Photography (ถ่ายภาพทิวทัศน์)
const landscapeDOF = {
    focus: 10.0,       // โฟกัสไกล
    aperture: 0.005,   // รูรับแสงแคบ (F/11 equivalent)
    maxblur: 0.005     // blur น้อย
};

// Cinematic (สไตล์ภาพยนตร์)
const cinematicDOF = {
    focus: 3.0,
    aperture: 0.035,
    maxblur: 0.015
};

// ฟังก์ชันสำหรับ smooth transition ระหว่าง DOF settings
function transitionDOF(bokehPass, from, to, duration, onComplete) {
    const startTime = performance.now();
    
    function update() {
        const elapsed = (performance.now() - startTime) / 1000;
        const t = Math.min(elapsed / duration, 1.0);
        const ease = t * t * (3 - 2 * t); // smoothstep
        
        bokehPass.uniforms.focus.value = from.focus + (to.focus - from.focus) * ease;
        bokehPass.uniforms.aperture.value = from.aperture + (to.aperture - from.aperture) * ease;
        bokehPass.uniforms.maxblur.value = from.maxblur + (to.maxblur - from.maxblur) * ease;
        
        if (t < 1.0) {
            requestAnimationFrame(update);
        } else if (onComplete) {
            onComplete();
        }
    }
    
    requestAnimationFrame(update);
}

// ใช้งาน:
transitionDOF(bokehPass, portraitDOF, macroDOF, 1.5, () => {
    console.log('Transition complete!');
});
```

## การ Focus Tracking แบบอัตโนมัติ

```javascript
// ระบบ Auto-Focus ที่ติดตามวัตถุ
class AutoFocusSystem {
    constructor(camera, composer, bokehPass) {
        this.camera = camera;
        this.bokehPass = bokehPass;
        this.raycaster = new THREE.Raycaster();
        this.targetFocus = 1.0;
        this.currentFocus = 1.0;
        this.focusSpeed = 3.0; // speed ของ autofocus
        this.enabled = true;
        
        // ศูนย์กลางของหน้าจอ
        this.center = new THREE.Vector2(0, 0);
    }
    
    update(scene, deltaTime) {
        if (!this.enabled) return;
        
        // Raycast จากกลางจอ
        this.raycaster.setFromCamera(this.center, this.camera);
        const intersects = this.raycaster.intersectObjects(scene.children, true);
        
        if (intersects.length > 0) {
            this.targetFocus = intersects[0].distance;
        }
        
        // Smooth interpolation ไปยัง target focus
        const diff = this.targetFocus - this.currentFocus;
        this.currentFocus += diff * this.focusSpeed * deltaTime;
        this.bokehPass.uniforms.focus.value = this.currentFocus;
    }
    
    setTarget(object) {
        const distance = this.camera.position.distanceTo(object.position);
        this.targetFocus = distance;
    }
    
    setEnabled(enabled) {
        this.enabled = enabled;
    }
}

// การใช้งาน
const autoFocus = new AutoFocusSystem(camera, composer, bokehPass);

// ใน animation loop
function animate() {
    requestAnimationFrame(animate);
    const delta = clock.getDelta();
    autoFocus.update(scene, delta);
    composer.render();
}
```

## สรุปเปรียบเทียบเทคนิค DoF

| เทคนิค | คุณภาพ | ประสิทธิภาพ | ความยืดหยุ่น |
|--------|---------|-------------|--------------|
| BokehPass (built-in) | ปานกลาง | ดี | จำกัด |
| Custom DoF (Poisson) | ดี | ปานกลาง | สูง |
| Tilt-Shift | เฉพาะเจาะจง | ดีมาก | ปานกลาง |
| CoC-based (Full) | ดีมาก | ต่ำ | สูงมาก |

## แนวทางการ Optimize

```javascript
// เทคนิคการปรับปรุงประสิทธิภาพ DoF

// 1. ลด resolution ของ blur pass
const halfSizeTarget = new THREE.WebGLRenderTarget(
    Math.floor(width / 2), 
    Math.floor(height / 2)
);

// 2. Separate Near/Far blur
// - วัตถุที่ใกล้กว่า focus = foreground blur (มักจะใส่ blur น้อยกว่า)
// - วัตถุที่ไกลกว่า focus = background blur

// 3. Temporal accumulation สำหรับ softer bokeh
// (ใช้ TAA pass ก่อน bokeh pass)

// 4. เช็ค focus range ก่อน apply blur
// ถ้า CoC น้อยกว่า threshold = ไม่ต้อง blur
const COC_THRESHOLD = 0.001;
if (coc > COC_THRESHOLD) {
    // apply blur
}

// 5. ใช้ Compute Shader (WebGPU) สำหรับ scatter-based bokeh
// (รองรับบน WebGPU backends เท่านั้น)
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Focus Puller Animation
สร้าง animation ที่ค่อยๆ เปลี่ยน focus distance จากวัตถุชิ้นหนึ่งไปยังอีกชิ้นหนึ่ง (เหมือนใน ภาพยนตร์ที่มีการ "pull focus")

**เงื่อนไข:**
- มีวัตถุ 3 ชิ้นเรียงห่างกันตามแนวลึก
- กด 1, 2, 3 เพื่อ focus ไปยังวัตถุแต่ละชิ้น
- การเปลี่ยน focus ต้อง smooth (ใช้ lerp หรือ spring)

### แบบฝึกหัดที่ 2: Bokeh Shape Customization
แก้ไข fragment shader ของ custom DoF เพื่อสร้าง bokeh ในรูปทรงต่างๆ:
- วงกลม (เริ่มต้น)
- หกเหลี่ยม (hexagonal aperture)
- หัวใจ

### แบบฝึกหัดที่ 3: Depth-Based Color Grading
รวม DoF เข้ากับ color grading:
- วัตถุที่อยู่ใน focus = สีปกติ
- วัตถุที่ out of focus = desaturated หรือเปลี่ยน hue เล็กน้อย

### แบบฝึกหัดที่ 4: Interactive Miniature World
ขยาย miniature world demo:
- เพิ่ม animation ให้รถวิ่งไปตามถนน
- เพิ่มแสงไฟกลางคืน (emissive materials)
- เพิ่มปุ่ม Day/Night toggle

---

**ความรู้ที่ได้รับในบทนี้:**
- หลักการทำงานของ Depth of Field และ Bokeh ในกล้องจริงและในกราฟิก 3D
- การใช้ BokehPass ของ Three.js ด้วยพารามิเตอร์ต่างๆ
- การสร้าง Custom DoF Pass ด้วย render targets และ Poisson disk sampling
- การสร้าง Tilt-Shift effect สำหรับ miniature world
- การทำ auto-focus ด้วย raycasting
- เทคนิคการ optimize DoF effects

**บทต่อไป:** Motion Blur & Camera Effects - เราจะเรียนรู้การสร้างเอฟเฟกต์การเคลื่อนไหวที่สมจริงด้วย motion blur, radial blur, camera shake และอื่นๆ
