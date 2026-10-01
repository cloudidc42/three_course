# Part 51: SSAO & Ambient Occlusion

## แนวคิด Ambient Occlusion

Ambient Occlusion (AO) เป็นเทคนิคการแรเงาที่คำนวณว่าจุดใดในฉากได้รับ "ambient light" (แสงสิ่งแวดล้อม) มากน้อยเพียงใด บริเวณที่แสงเข้าถึงได้ยาก เช่น มุม รอยต่อ หรือรอยพับ จะมืดกว่า ทำให้ฉากดูมีความลึกและสมจริงมากขึ้น

### ทำไม AO ถึงสำคัญ?

โดยไม่มี AO วัตถุที่ติดกันมักดูเหมือน "ลอยอยู่" ไม่ได้สัมผัสกัน AO แก้ปัญหานี้โดยทำให้พื้นที่สัมผัสหรือมุมแคบๆ มืดลง ทำให้:
- วัตถุดูเหมือนวางอยู่บนพื้นจริงๆ
- เพิ่ม contact shadows แม้ไม่มีแสงตรง
- เพิ่มความลึกและ volume ให้กับฉาก

## Screen Space Ambient Occlusion (SSAO)

SSAO เป็นเทคนิคที่คำนวณ AO ใน screen space โดยใช้ข้อมูล depth buffer และ normal buffer คำนวณ occlusion ในแต่ละ pixel

### หลักการทำงาน
1. สำหรับแต่ละ pixel ใน screen
2. สุ่มตัวอย่างรอบๆ ใน hemisphere ตามทิศ normal
3. เช็คว่า sample แต่ละอันถูกบัง (occluded) ด้วย geometry ใกล้เคียงหรือไม่
4. ค่า occlusion = สัดส่วนของ samples ที่ถูกบัง

```javascript
// การใช้ SSAOPass ใน Three.js
import { SSAOPass } from 'three/examples/jsm/postprocessing/SSAOPass.js';

const ssaoPass = new SSAOPass(scene, camera, width, height);
ssaoPass.kernelRadius = 16;     // รัศมีของ sampling kernel (pixels)
ssaoPass.minDistance = 0.005;   // ระยะขั้นต่ำสำหรับ occlusion
ssaoPass.maxDistance = 0.1;     // ระยะสูงสุดสำหรับ occlusion

composer.addPass(ssaoPass);
```

### พารามิเตอร์ของ SSAOPass

```javascript
// kernelRadius - รัศมีของ hemisphere sample
// ยิ่งมาก ยิ่งเห็นการบัง AO ในระยะไกลกว่า
// แต่ก็ใช้ทรัพยากรมากขึ้น
ssaoPass.kernelRadius = 16;      // default: 8

// kernelSize - จำนวน sample points
// มากขึ้น = คุณภาพดีขึ้น แต่ช้าลง
ssaoPass.kernelSize = 32;        // default: 32

// sigma - blur ของ AO result
ssaoPass.sigma = 4;              // default: 4

// rings - จำนวน rings ใน sampling pattern
ssaoPass.rings = 4;              // default: 4

// minDistance & maxDistance
// ควบคุมช่วงความลึกที่ AO มีผล
ssaoPass.minDistance = 0.001;
ssaoPass.maxDistance = 0.05;

// output mode
// SSAOPass.OUTPUT.Default = render ปกติ + AO
// SSAOPass.OUTPUT.SSAO = เห็นเฉพาะ AO pass
// SSAOPass.OUTPUT.Blur = เห็น blurred AO
// SSAOPass.OUTPUT.Beauty = ไม่มี AO (เพื่อเปรียบเทียบ)
// SSAOPass.OUTPUT.Depth = เห็น depth buffer
// SSAOPass.OUTPUT.Normal = เห็น normal buffer
ssaoPass.output = SSAOPass.OUTPUT.Default;
```

## ตัวอย่างที่ 1: Scene เปรียบเทียบ With/Without AO

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>SSAO Comparison</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; font-family: Arial, sans-serif; }
        canvas { display: block; }
        #controls {
            position: fixed;
            top: 20px;
            right: 20px;
            background: rgba(0,0,0,0.85);
            color: white;
            padding: 20px;
            border-radius: 10px;
            width: 260px;
        }
        h3 { color: #4fc3f7; margin-bottom: 15px; }
        .ctrl { margin-bottom: 12px; }
        .ctrl label { display: flex; justify-content: space-between; font-size: 12px; margin-bottom: 4px; }
        .ctrl input[type=range] { width: 100%; }
        .btn-group { display: flex; gap: 5px; margin-top: 10px; }
        .btn {
            flex: 1; padding: 8px 5px; border: none; border-radius: 5px;
            cursor: pointer; font-size: 11px; color: white; background: #555;
        }
        .btn.active { background: #4fc3f7; color: #000; }
        #output-modes { margin-top: 10px; }
        .mode-btn {
            display: block; width: 100%; padding: 6px; border: 1px solid #555;
            background: transparent; color: #ccc; text-align: left; cursor: pointer;
            margin-bottom: 3px; border-radius: 4px; font-size: 12px;
        }
        .mode-btn.active { background: #1a1a2e; border-color: #4fc3f7; color: #4fc3f7; }
    </style>
</head>
<body>
    <div id="controls">
        <h3>🌑 SSAO Settings</h3>
        <div class="ctrl">
            <label><span>Kernel Radius</span><span id="krVal">16</span></label>
            <input type="range" id="kernelRadius" min="1" max="32" step="1" value="16">
        </div>
        <div class="ctrl">
            <label><span>Min Distance</span><span id="minVal">0.005</span></label>
            <input type="range" id="minDist" min="0.001" max="0.05" step="0.001" value="0.005">
        </div>
        <div class="ctrl">
            <label><span>Max Distance</span><span id="maxVal">0.100</span></label>
            <input type="range" id="maxDist" min="0.01" max="0.5" step="0.005" value="0.1">
        </div>
        <div id="output-modes">
            <p style="font-size:12px;margin-bottom:5px;color:#aaa">Output Mode:</p>
            <button class="mode-btn active" data-mode="Default">Default (AO Applied)</button>
            <button class="mode-btn" data-mode="SSAO">AO Only</button>
            <button class="mode-btn" data-mode="Beauty">No AO (Beauty)</button>
            <button class="mode-btn" data-mode="Depth">Depth Buffer</button>
            <button class="mode-btn" data-mode="Normal">Normal Buffer</button>
        </div>
        <div style="margin-top:10px;font-size:11px;color:#aaa">
            SPACE = Toggle AO On/Off<br>
            คลิกหมุนฉาก
        </div>
    </div>

    <script type="module">
        import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js';
        import { EffectComposer } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/EffectComposer.js';
        import { RenderPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/RenderPass.js';
        import { SSAOPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/SSAOPass.js';
        import { OutputPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/OutputPass.js';
        import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/controls/OrbitControls.js';

        const W = window.innerWidth, H = window.innerHeight;
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(W, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        renderer.shadowMap.enabled = true;
        document.body.appendChild(renderer.domElement);

        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x222233);

        const camera = new THREE.PerspectiveCamera(60, W/H, 0.1, 100);
        camera.position.set(0, 8, 12);
        camera.lookAt(0, 0, 0);

        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.dampingFactor = 0.05;

        // Lighting (แสงแบบ ambient สุทธิๆ เพื่อเห็น AO ชัด)
        const ambientLight = new THREE.AmbientLight(0xffffff, 1.0);
        scene.add(ambientLight);
        
        const dirLight = new THREE.DirectionalLight(0xffffff, 0.3);
        dirLight.position.set(5, 10, 5);
        scene.add(dirLight);

        // Materials
        const concreteMat = new THREE.MeshStandardMaterial({ color: 0xaaaaaa, roughness: 0.9 });
        const woodMat = new THREE.MeshStandardMaterial({ color: 0x8b6347, roughness: 0.8 });
        const metalMat = new THREE.MeshStandardMaterial({ color: 0x888888, roughness: 0.3, metalness: 0.9 });
        const plasticMat = new THREE.MeshStandardMaterial({ color: 0x2266aa, roughness: 0.5 });

        // สร้าง scene ที่เหมาะสำหรับเห็น AO
        // AO จะเห็นได้ชัดในบริเวณ:
        // - ใต้วัตถุ
        // - มุมห้อง
        // - รอยต่อระหว่างวัตถุ

        // พื้น
        const floorGeo = new THREE.PlaneGeometry(20, 20);
        const floor = new THREE.Mesh(floorGeo, concreteMat);
        floor.rotation.x = -Math.PI / 2;
        scene.add(floor);

        // ผนัง
        const wallMat = new THREE.MeshStandardMaterial({ color: 0xcccccc, roughness: 0.9 });
        const backWall = new THREE.Mesh(new THREE.PlaneGeometry(20, 10), wallMat);
        backWall.position.set(0, 5, -10);
        scene.add(backWall);
        
        const leftWall = new THREE.Mesh(new THREE.PlaneGeometry(20, 10), wallMat);
        leftWall.position.set(-10, 5, 0);
        leftWall.rotation.y = Math.PI / 2;
        scene.add(leftWall);

        // กล่อง stack (เห็น AO ชัดมากใต้กล่อง)
        const boxPositions = [
            { x: -3, y: 0.5, z: -3, w: 2, h: 1, d: 2 },
            { x: -3, y: 1.5, z: -3, w: 1.5, h: 1, d: 1.5 },
            { x: -3, y: 2.5, z: -3, w: 1, h: 1, d: 1 },
            { x: 2, y: 0.75, z: -2, w: 1.5, h: 1.5, d: 1.5 },
            { x: 4, y: 0.5, z: 0, w: 1, h: 1, d: 3 },
            { x: -1, y: 0.5, z: 2, w: 3, h: 1, d: 1 },
        ];
        
        boxPositions.forEach(b => {
            const geo = new THREE.BoxGeometry(b.w, b.h, b.d);
            const mat = Math.random() > 0.5 ? woodMat : concreteMat;
            const mesh = new THREE.Mesh(geo, mat);
            mesh.position.set(b.x, b.y, b.z);
            mesh.castShadow = true;
            mesh.receiveShadow = true;
            scene.add(mesh);
        });

        // ทรงกลม (เห็น contact shadow ใต้ sphere)
        const spherePositions = [
            { x: 0, z: 0, r: 0.8 },
            { x: 3, z: -4, r: 0.5 },
            { x: -4, z: 1, r: 0.6 },
        ];
        spherePositions.forEach(s => {
            const geo = new THREE.SphereGeometry(s.r, 32, 32);
            const mesh = new THREE.Mesh(geo, metalMat);
            mesh.position.set(s.x, s.r, s.z);
            mesh.castShadow = true;
            scene.add(mesh);
        });

        // ท่อและ cylinder (เห็น AO ที่ฐาน)
        for (let i = 0; i < 4; i++) {
            const geo = new THREE.CylinderGeometry(0.3, 0.3, 1.5 + i * 0.5, 16);
            const mesh = new THREE.Mesh(geo, plasticMat.clone());
            mesh.material.color.setHex([0x2266aa, 0xaa6622, 0x22aa66, 0xaa2266][i]);
            mesh.position.set(-6 + i * 2, 0.75 + i * 0.25, -5);
            mesh.castShadow = true;
            scene.add(mesh);
        }

        // Furniture-like objects
        // โต๊ะ
        function createTable(x, z) {
            const g = new THREE.Group();
            const topGeo = new THREE.BoxGeometry(3, 0.1, 1.5);
            const top = new THREE.Mesh(topGeo, woodMat);
            top.position.y = 1.5;
            g.add(top);
            
            const legPositions = [[-1.3, 0, -0.6], [1.3, 0, -0.6], [-1.3, 0, 0.6], [1.3, 0, 0.6]];
            legPositions.forEach(([lx, ly, lz]) => {
                const legGeo = new THREE.BoxGeometry(0.1, 1.5, 0.1);
                const leg = new THREE.Mesh(legGeo, woodMat);
                leg.position.set(lx, 0.75, lz);
                g.add(leg);
            });
            
            g.position.set(x, 0, z);
            scene.add(g);
        }
        createTable(1, -7);
        createTable(-5, -6);

        // Post-processing
        const composer = new EffectComposer(renderer);
        const renderPass = new RenderPass(scene, camera);
        composer.addPass(renderPass);

        const ssaoPass = new SSAOPass(scene, camera, W, H);
        ssaoPass.kernelRadius = 16;
        ssaoPass.minDistance = 0.005;
        ssaoPass.maxDistance = 0.1;
        composer.addPass(ssaoPass);

        composer.addPass(new OutputPass());

        // Controls
        document.getElementById('kernelRadius').addEventListener('input', (e) => {
            const v = parseInt(e.target.value);
            ssaoPass.kernelRadius = v;
            document.getElementById('krVal').textContent = v;
        });
        document.getElementById('minDist').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            ssaoPass.minDistance = v;
            document.getElementById('minVal').textContent = v.toFixed(3);
        });
        document.getElementById('maxDist').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            ssaoPass.maxDistance = v;
            document.getElementById('maxVal').textContent = v.toFixed(3);
        });

        // Output mode buttons
        const outputModes = {
            'Default': SSAOPass.OUTPUT.Default,
            'SSAO': SSAOPass.OUTPUT.SSAO,
            'Beauty': SSAOPass.OUTPUT.Beauty,
            'Depth': SSAOPass.OUTPUT.Depth,
            'Normal': SSAOPass.OUTPUT.Normal
        };
        
        document.querySelectorAll('.mode-btn').forEach(btn => {
            btn.addEventListener('click', () => {
                document.querySelectorAll('.mode-btn').forEach(b => b.classList.remove('active'));
                btn.classList.add('active');
                ssaoPass.output = outputModes[btn.dataset.mode];
            });
        });

        // Toggle AO
        let aoEnabled = true;
        document.addEventListener('keydown', (e) => {
            if (e.key === ' ') {
                aoEnabled = !aoEnabled;
                ssaoPass.enabled = aoEnabled;
            }
        });

        window.addEventListener('resize', () => {
            const w = window.innerWidth, h = window.innerHeight;
            camera.aspect = w / h;
            camera.updateProjectionMatrix();
            renderer.setSize(w, h);
            composer.setSize(w, h);
            ssaoPass.setSize(w, h);
        });

        const clock = new THREE.Clock();
        function animate() {
            requestAnimationFrame(animate);
            controls.update();
            composer.render();
        }
        animate();
    </script>
</body>
</html>
```

## HBAO (Horizon-Based Ambient Occlusion)

HBAO เป็นการพัฒนาต่อจาก SSAO ที่ให้คุณภาพดีกว่าโดยพิจารณาทิศทางของ horizon:

```glsl
// HBAO Shader Fragment (แนวคิดหลัก)
// HBAO ใช้การ march ไปตาม directions แทนการ sample random points
// ทำให้ได้ผลที่ smooth กว่าและ artifact น้อยกว่า

uniform sampler2D tDepth;
uniform sampler2D tNormal;
uniform mat4 projectionMatrix;
uniform mat4 projectionMatrixInverse;
uniform vec2 resolution;
uniform float radius;     // รัศมีของ AO ใน world space
uniform float bias;       // ป้องกัน self-occlusion
uniform float intensity;  // ความแรง AO
uniform int numDirections; // จำนวนทิศทาง (4-8)
uniform int numSteps;      // จำนวน steps ต่อทิศทาง (4-8)

varying vec2 vUv;

// แปลง UV + depth เป็น view space position
vec3 getViewPosition(vec2 uv) {
    float depth = texture2D(tDepth, uv).r;
    // แปลงเป็น NDC แล้วเป็น view space
    vec4 ndc = vec4(uv * 2.0 - 1.0, depth * 2.0 - 1.0, 1.0);
    vec4 viewPos = projectionMatrixInverse * ndc;
    return viewPos.xyz / viewPos.w;
}

void main() {
    vec3 viewPos = getViewPosition(vUv);
    vec3 viewNorm = texture2D(tNormal, vUv).xyz * 2.0 - 1.0;
    
    float occlusion = 0.0;
    float stepSizeUV = radius / (float(numSteps) * length(viewPos));
    
    // วนรอบทิศทางต่างๆ
    for (int d = 0; d < 8; d++) {
        if (d >= numDirections) break;
        
        float angle = float(d) / float(numDirections) * 3.14159 * 2.0;
        vec2 dir = vec2(cos(angle), sin(angle));
        
        float maxHorizonAngle = -3.14159 / 2.0;
        
        // March ไปตาม direction
        for (int s = 1; s <= 8; s++) {
            if (s > numSteps) break;
            
            vec2 sampleUV = vUv + dir * stepSizeUV * float(s);
            if (sampleUV.x < 0.0 || sampleUV.x > 1.0 || 
                sampleUV.y < 0.0 || sampleUV.y > 1.0) break;
            
            vec3 samplePos = getViewPosition(sampleUV);
            vec3 horizonVec = samplePos - viewPos;
            float horizonAngle = atan(horizonVec.z, length(horizonVec.xy));
            
            maxHorizonAngle = max(maxHorizonAngle, horizonAngle);
        }
        
        // AO contribution จาก direction นี้
        float normalAngle = asin(dot(viewNorm, vec3(0.0, 0.0, 1.0)));
        occlusion += clamp((sin(maxHorizonAngle) - sin(normalAngle + bias)) / (1.0 - sin(normalAngle + bias)), 0.0, 1.0);
    }
    
    occlusion = 1.0 - occlusion / float(numDirections) * intensity;
    gl_FragColor = vec4(vec3(occlusion), 1.0);
}
```

## Baked AO (aoMap)

Baked AO เป็นการคำนวณ AO ไว้ล่วงหน้าและเก็บใน texture ซึ่งมีประสิทธิภาพดีกว่า runtime SSAO:

```javascript
// การใช้ aoMap ใน Three.js
const geometry = new THREE.BoxGeometry();
const material = new THREE.MeshStandardMaterial({
    map: diffuseTexture,           // albedo
    aoMap: aoTexture,              // baked AO texture
    aoMapIntensity: 1.0,           // ความแรงของ AO (0-1)
    normalMap: normalTexture,
    roughnessMap: roughnessTexture
});

// สำคัญ: MeshStandardMaterial ใช้ UV2 สำหรับ aoMap
// ต้องตั้งค่า uv2 attribute
geometry.setAttribute('uv2', geometry.attributes.uv);

const mesh = new THREE.Mesh(geometry, material);

// หรือสำหรับ loaded models (GLTF):
gltf.scene.traverse(node => {
    if (node.isMesh) {
        // GLTF มักจะมี uv2 อยู่แล้ว
        if (node.geometry.attributes.uv2 === undefined) {
            node.geometry.setAttribute('uv2', node.geometry.attributes.uv);
        }
        
        if (node.material.aoMap) {
            node.material.aoMapIntensity = 1.0;
        }
    }
});
```

## ตัวอย่างที่ 2: Architecture Visualization with AO

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Architecture Visualization with AO</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #87ceeb; overflow: hidden; font-family: Arial, sans-serif; }
        canvas { display: block; }
        #ui {
            position: fixed;
            top: 20px;
            left: 20px;
            background: rgba(255,255,255,0.9);
            padding: 15px;
            border-radius: 10px;
            width: 200px;
            font-size: 13px;
            color: #333;
        }
        #ui h3 { margin-bottom: 10px; color: #2c3e50; }
        .ctrl { margin-bottom: 8px; }
        .ctrl label { display: flex; justify-content: space-between; margin-bottom: 3px; }
        .toggle { display: flex; align-items: center; gap: 10px; margin-bottom: 8px; }
        .toggle input[type=checkbox] { width: 18px; height: 18px; cursor: pointer; }
        .separator { height: 1px; background: #ddd; margin: 10px 0; }
    </style>
</head>
<body>
    <div id="ui">
        <h3>🏛️ Architecture AO</h3>
        <div class="toggle">
            <input type="checkbox" id="ssaoOn" checked>
            <label for="ssaoOn">Enable SSAO</label>
        </div>
        <div class="ctrl">
            <label><span>AO Radius</span><span id="radVal">8</span></label>
            <input type="range" id="radius" min="1" max="32" step="1" value="8">
        </div>
        <div class="ctrl">
            <label><span>AO Intensity</span><span id="intVal">0.15</span></label>
            <input type="range" id="intensity" min="0.01" max="0.5" step="0.01" value="0.15">
        </div>
        <div class="separator"></div>
        <div class="toggle">
            <input type="checkbox" id="shadowOn" checked>
            <label for="shadowOn">Shadows</label>
        </div>
        <div class="toggle">
            <input type="checkbox" id="wireframe" >
            <label for="wireframe">Wireframe</label>
        </div>
        <div class="separator"></div>
        <div style="font-size:11px;color:#666">
            คลิกลาก = หมุน<br>
            Scroll = Zoom<br>
            คลิกขวาลาก = Pan
        </div>
    </div>

    <script type="module">
        import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js';
        import { EffectComposer } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/EffectComposer.js';
        import { RenderPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/RenderPass.js';
        import { SSAOPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/SSAOPass.js';
        import { OutputPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/OutputPass.js';
        import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/controls/OrbitControls.js';

        const W = window.innerWidth, H = window.innerHeight;
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(W, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        renderer.shadowMap.enabled = true;
        renderer.shadowMap.type = THREE.PCFSoftShadowMap;
        document.body.appendChild(renderer.domElement);

        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0xd4e8f0);
        scene.fog = new THREE.Fog(0xd4e8f0, 30, 100);

        const camera = new THREE.PerspectiveCamera(50, W/H, 0.1, 200);
        camera.position.set(15, 12, 20);
        camera.lookAt(0, 3, 0);

        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.dampingFactor = 0.05;
        controls.target.set(0, 3, 0);

        // Lighting
        scene.add(new THREE.AmbientLight(0xffffff, 1.5));
        const sun = new THREE.DirectionalLight(0xfffde7, 2);
        sun.position.set(20, 30, 10);
        sun.castShadow = true;
        sun.shadow.mapSize.set(4096, 4096);
        sun.shadow.camera.near = 0.5;
        sun.shadow.camera.far = 100;
        sun.shadow.camera.left = -30;
        sun.shadow.camera.right = 30;
        sun.shadow.camera.top = 30;
        sun.shadow.camera.bottom = -30;
        sun.shadow.bias = -0.001;
        scene.add(sun);

        // Materials
        const concreteMat = new THREE.MeshStandardMaterial({ color: 0xd4cfc9, roughness: 0.95 });
        const glassMat = new THREE.MeshStandardMaterial({ color: 0x88aacc, transparent: true, opacity: 0.4, metalness: 0.1, roughness: 0.0 });
        const steelMat = new THREE.MeshStandardMaterial({ color: 0x888888, roughness: 0.3, metalness: 0.9 });
        const woodMat = new THREE.MeshStandardMaterial({ color: 0x8b6347, roughness: 0.9 });
        const stoneMat = new THREE.MeshStandardMaterial({ color: 0x999988, roughness: 1.0 });

        // สร้างอาคาร Modern Architecture
        function buildModernHouse() {
            const house = new THREE.Group();
            
            // ฐาน
            const baseMesh = new THREE.Mesh(new THREE.BoxGeometry(16, 0.5, 12), stoneMat);
            baseMesh.position.y = 0.25;
            baseMesh.receiveShadow = true;
            house.add(baseMesh);
            
            // ผนังหลัก
            const mainWall = new THREE.Mesh(new THREE.BoxGeometry(16, 6, 0.3), concreteMat);
            mainWall.position.set(0, 3.25, -5.85);
            mainWall.castShadow = true;
            mainWall.receiveShadow = true;
            house.add(mainWall);
            
            const backWall = mainWall.clone();
            backWall.position.z = 5.85;
            house.add(backWall);
            
            const leftWall = new THREE.Mesh(new THREE.BoxGeometry(0.3, 6, 12), concreteMat);
            leftWall.position.set(-7.85, 3.25, 0);
            leftWall.castShadow = true;
            leftWall.receiveShadow = true;
            house.add(leftWall);
            
            // หน้าต่าง (glass strips)
            const windowFrameMat = new THREE.MeshStandardMaterial({ color: 0x333333, metalness: 0.5 });
            
            // หน้าต่างใหญ่ด้านหน้า
            for (let i = 0; i < 4; i++) {
                const winFrame = new THREE.Mesh(new THREE.BoxGeometry(3, 4, 0.1), windowFrameMat);
                winFrame.position.set(-4.5 + i * 3, 3.5, 5.9);
                house.add(winFrame);
                
                const win = new THREE.Mesh(new THREE.BoxGeometry(2.7, 3.7, 0.05), glassMat);
                win.position.copy(winFrame.position);
                win.position.z += 0.03;
                house.add(win);
            }
            
            // เพดาน/หลังคา
            const roofMesh = new THREE.Mesh(new THREE.BoxGeometry(16.5, 0.5, 12.5), concreteMat);
            roofMesh.position.y = 6.25;
            roofMesh.castShadow = true;
            roofMesh.receiveShadow = true;
            house.add(roofMesh);
            
            // Overhang (หลังคาที่ยื่นออกมา)
            const overhang = new THREE.Mesh(new THREE.BoxGeometry(18, 0.3, 14), concreteMat);
            overhang.position.y = 7.5;
            overhang.castShadow = true;
            house.add(overhang);
            
            // เสา
            const pillarPositions = [-6, -3, 0, 3, 6];
            pillarPositions.forEach(x => {
                const pillar = new THREE.Mesh(new THREE.CylinderGeometry(0.15, 0.15, 7.5, 8), steelMat);
                pillar.position.set(x, 3.75, 6.2);
                pillar.castShadow = true;
                house.add(pillar);
            });
            
            // บันได
            for (let i = 0; i < 5; i++) {
                const step = new THREE.Mesh(new THREE.BoxGeometry(6, 0.2, 0.4), stoneMat);
                step.position.set(0, 0.5 + i * 0.2 - 0.5, 7 + i * 0.4);
                step.receiveShadow = true;
                house.add(step);
            }
            
            // ราวบันได
            const railMat = new THREE.MeshStandardMaterial({ color: 0x888888, metalness: 0.9 });
            const rail = new THREE.Mesh(new THREE.CylinderGeometry(0.03, 0.03, 1.2, 6), railMat);
            rail.position.set(3.2, 1.2, 8);
            house.add(rail.clone());
            const rail2 = rail.clone();
            rail2.position.x = -3.2;
            house.add(rail2);
            
            // Pool
            const poolEdge = new THREE.Mesh(new THREE.BoxGeometry(6, 0.3, 4), stoneMat);
            poolEdge.position.set(5, 0.15, 0);
            poolEdge.receiveShadow = true;
            house.add(poolEdge);
            
            const waterMat = new THREE.MeshStandardMaterial({ color: 0x1a7fbf, transparent: true, opacity: 0.85, roughness: 0.0, metalness: 0.1 });
            const water = new THREE.Mesh(new THREE.BoxGeometry(5.6, 0.1, 3.6), waterMat);
            water.position.set(5, 0.22, 0);
            house.add(water);
            
            return house;
        }

        // พื้นดิน
        const groundMesh = new THREE.Mesh(
            new THREE.PlaneGeometry(100, 100),
            new THREE.MeshStandardMaterial({ color: 0x4a7c3f, roughness: 0.9 })
        );
        groundMesh.rotation.x = -Math.PI / 2;
        groundMesh.receiveShadow = true;
        scene.add(groundMesh);

        const house = buildModernHouse();
        scene.add(house);

        // ต้นไม้
        function createTree(x, z, height = 4) {
            const g = new THREE.Group();
            const trunk = new THREE.Mesh(
                new THREE.CylinderGeometry(0.1, 0.15, height * 0.4, 8),
                woodMat
            );
            trunk.position.y = height * 0.2;
            trunk.castShadow = true;
            g.add(trunk);
            
            const leaves = new THREE.Mesh(
                new THREE.SphereGeometry(height * 0.25, 10, 8),
                new THREE.MeshStandardMaterial({ color: new THREE.Color(0.1, 0.3 + Math.random()*0.2, 0.05), roughness: 0.9 })
            );
            leaves.position.y = height * 0.55;
            leaves.castShadow = true;
            g.add(leaves);
            
            g.position.set(x, 0, z);
            scene.add(g);
        }

        createTree(-12, -4, 5);
        createTree(-10, 3, 4);
        createTree(12, -6, 6);
        createTree(11, 2, 5);
        createTree(-14, 8, 7);

        // Post-processing
        const composer = new EffectComposer(renderer);
        composer.addPass(new RenderPass(scene, camera));

        const ssaoPass = new SSAOPass(scene, camera, W, H);
        ssaoPass.kernelRadius = 8;
        ssaoPass.minDistance = 0.001;
        ssaoPass.maxDistance = 0.15;
        composer.addPass(ssaoPass);

        composer.addPass(new OutputPass());

        // UI
        document.getElementById('ssaoOn').addEventListener('change', (e) => {
            ssaoPass.enabled = e.target.checked;
        });
        document.getElementById('radius').addEventListener('input', (e) => {
            ssaoPass.kernelRadius = parseInt(e.target.value);
            document.getElementById('radVal').textContent = e.target.value;
        });
        document.getElementById('intensity').addEventListener('input', (e) => {
            ssaoPass.maxDistance = parseFloat(e.target.value);
            document.getElementById('intVal').textContent = e.target.value;
        });
        document.getElementById('shadowOn').addEventListener('change', (e) => {
            renderer.shadowMap.enabled = e.target.checked;
            scene.traverse(node => {
                if (node.isMesh) {
                    node.castShadow = e.target.checked;
                    node.receiveShadow = e.target.checked;
                }
            });
        });
        document.getElementById('wireframe').addEventListener('change', (e) => {
            scene.traverse(node => {
                if (node.isMesh && node.material) {
                    node.material.wireframe = e.target.checked;
                }
            });
        });

        window.addEventListener('resize', () => {
            const w = window.innerWidth, h = window.innerHeight;
            camera.aspect = w / h;
            camera.updateProjectionMatrix();
            renderer.setSize(w, h);
            composer.setSize(w, h);
            ssaoPass.setSize(w, h);
        });

        function animate() {
            requestAnimationFrame(animate);
            controls.update();
            composer.render();
        }
        animate();
    </script>
</body>
</html>
```

## N8AOPass - Alternative ที่คุณภาพสูงกว่า

N8AOPass เป็น third-party SSAO pass ที่ให้คุณภาพดีกว่า SSAOPass มาตรฐาน:

```javascript
// N8AOPass (ต้องติดตั้ง n8ao package)
// npm install n8ao

import { N8AOPass } from 'n8ao';

const n8aoPass = new N8AOPass(scene, camera, width, height);

// Configuration
n8aoPass.configuration.aoRadius = 5;       // รัศมีของ AO
n8aoPass.configuration.distanceFalloff = 1; // การลดลงตามระยะ
n8aoPass.configuration.intensity = 5;       // ความแรง
n8aoPass.configuration.color = new THREE.Color(0, 0, 0); // สีของ AO
n8aoPass.configuration.aoSamples = 16;     // จำนวน samples
n8aoPass.configuration.denoiseSamples = 8; // Denoising samples
n8aoPass.configuration.denoiseRadius = 12; // Denoising radius
n8aoPass.configuration.qualityMode = "Low" / "Medium" / "High" / "Ultra";

composer.addPass(n8aoPass);
```

## การ Optimize AO Performance

```javascript
// เทคนิคการเพิ่มประสิทธิภาพ AO

// 1. ลด resolution ของ SSAO pass
const halfW = Math.floor(window.innerWidth / 2);
const halfH = Math.floor(window.innerHeight / 2);
const ssaoPass = new SSAOPass(scene, camera, halfW, halfH);
// ใช้ blur pass เพื่อ smooth ผล

// 2. ลด kernel size ในการ sample
ssaoPass.kernelSize = 16; // แทนที่จะเป็น 32

// 3. ใช้ baked AO แทน SSAO สำหรับ static geometry
// คำนวณ AO ล่วงหน้าใน Blender หรือ Maya แล้วใส่เป็น texture

// 4. ทำ temporal accumulation (สำหรับ static scenes)
// เก็บผลของ SSAO จาก frames ก่อนหน้าแล้วผสม

// 5. LOD สำหรับ SSAO
// ลด SSAO quality เมื่ออยู่ไกลจากกล้อง
function updateSSAOQuality(cameraHeight) {
    if (cameraHeight > 50) {
        ssaoPass.kernelRadius = 4;
        ssaoPass.kernelSize = 8;
    } else if (cameraHeight > 20) {
        ssaoPass.kernelRadius = 8;
        ssaoPass.kernelSize = 16;
    } else {
        ssaoPass.kernelRadius = 16;
        ssaoPass.kernelSize = 32;
    }
}

// 6. ใช้ SSAO เฉพาะ deferred rendering
// SSAO ทำงานได้ดีกว่าใน deferred rendering pipeline
// เพราะมี normal/depth buffer แยกกันอยู่แล้ว
```

## การรวม AO กับ Lighting Model

```javascript
// AO ควรถูก apply เฉพาะกับ indirect lighting ไม่ใช่ direct lighting
// โดยหลักการที่ถูกต้อง:
//
// finalColor = directLight + indirectLight * ao
//
// ไม่ใช่: finalColor = (directLight + indirectLight) * ao
//
// แต่สำหรับ real-time graphics เรามักใช้การประมาณ:
// finalColor = albedo * (ambientLight * ao + directLight)

// Three.js MeshStandardMaterial จัดการนี้โดยอัตโนมัติ
const material = new THREE.MeshStandardMaterial({
    color: 0xffffff,
    aoMap: aoTexture,
    aoMapIntensity: 1.0  // 0 = ไม่มี AO, 1 = full AO
});

// aoMapIntensity ควบคุมได้ใน runtime
material.aoMapIntensity = 0.8; // ลด intensity เล็กน้อย
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: SSAO Debugger
สร้าง debug tool ที่แสดง:
- SSAO pass ด้านซ้าย
- ผลลัพธ์สุดท้ายด้านขวา
- slider สำหรับ blend ระหว่างสอง views

### แบบฝึกหัดที่ 2: Baked AO Generator
สร้าง tool ง่ายๆ ที่:
- Render scene จากหลายมุม
- สร้าง approximate AO texture
- Apply กลับไปที่ mesh

### แบบฝึกหัดที่ 3: Interior Scene
สร้างห้องภายในที่มี:
- มุมห้อง (เห็น AO ชัดมาก)
- เฟอร์นิเจอร์
- เปรียบเทียบ SSAO กับ Baked AO

### แบบฝึกหัดที่ 4: SSAO Optimization Test
ทดสอบประสิทธิภาพ:
- วัด FPS กับ settings ต่างๆ
- หา sweet spot ระหว่าง quality และ performance
- ทำ LOD system สำหรับ SSAO

---

**สรุปบทที่ 51:**
- AO เป็นเทคนิคสำคัญสำหรับความสมจริงของฉาก
- SSAOPass ใน Three.js ใช้งานง่ายแต่มีข้อจำกัดด้านคุณภาพ
- HBAO ให้คุณภาพดีกว่าแต่ต้องเขียน shader เอง
- Baked AO เหมาะสำหรับ static scenes และมีประสิทธิภาพสูงสุด
- ควรปรับ AO เฉพาะ indirect lighting เพื่อความถูกต้องทางฟิสิกส์

**บทต่อไป:** Screen Space Reflections (SSR)
