# Part 53: Volumetric Lighting & God Rays

## แนวคิด Volumetric Lighting

Volumetric Lighting หรือที่รู้จักกันในชื่อ "God Rays" หรือ "Crepuscular Rays" เป็นปรากฏการณ์ที่เกิดขึ้นเมื่อแสงเดินทางผ่านตัวกลาง เช่น หมอก ฝุ่น หรือควัน แสงแต่ละลำดูเหมือน "ทึบแสง" และมองเห็นได้ ปรากฏการณ์นี้พบเห็นได้บ่อยในธรรมชาติ เช่น:

- แสงอาทิตย์ส่องผ่านต้นไม้ในป่า
- แสงสาดผ่านหน้าต่างเข้ามาในห้องที่มีฝุ่น
- แสงเทียนในความมืด
- แสงไฟใต้น้ำ

ในการเรนเดอร์ 3D เราจำลอง volumetric lighting โดยใช้เทคนิคต่างๆ ตั้งแต่แบบง่ายจนถึงซับซ้อน

## เทคนิค Volumetric Lighting

### 1. Screen Space Radial Blur (God Rays)
เทคนิคที่ง่ายที่สุด: blur แบบ radial จากตำแหน่งแสงใน screen space

### 2. Depth-Based Light Shafts
ใช้ depth buffer เพื่อ mask ส่วนที่แสงไม่สามารถผ่านได้

### 3. Ray Marching Volumetrics
คำนวณแสงผ่าน volume จริงๆ ด้วย ray marching (แพงมากด้าน GPU)

### 4. Fog + Point Light
ใช้ THREE.FogExp2 ร่วมกับ point lights สร้าง pseudo-volumetric effect

## God Rays ด้วย Screen Space Radial Blur

```javascript
// Shader สำหรับ God Rays
const godRaysShader = {
    uniforms: {
        tDiffuse: { value: null },
        tOccluder: { value: null },   // texture ที่บอกว่าส่วนไหนบัง light
        lightPosition: { value: new THREE.Vector2(0.5, 0.5) }, // ตำแหน่งแสงใน screen space
        exposure: { value: 0.18 },    // ความสว่างของ god rays
        decay: { value: 0.95 },       // การลดลงของแสงตามระยะ
        density: { value: 0.8 },      // ความหนาแน่นของ rays
        weight: { value: 0.4 },       // น้ำหนักของ rays
        numSamples: { value: 80 }     // จำนวน samples
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
        uniform sampler2D tOccluder;
        uniform vec2 lightPosition;
        uniform float exposure;
        uniform float decay;
        uniform float density;
        uniform float weight;
        uniform int numSamples;
        
        varying vec2 vUv;
        
        void main() {
            vec2 texCoord = vUv;
            vec2 deltaTextCoord = texCoord - lightPosition;
            deltaTextCoord *= 1.0 / float(numSamples) * density;
            
            float illuminationDecay = 1.0;
            vec4 accum = vec4(0.0);
            
            for (int i = 0; i < 100; i++) {
                if (i >= numSamples) break;
                texCoord -= deltaTextCoord;
                
                // Clamp to avoid edge artifacts
                vec2 clampedCoord = clamp(texCoord, vec2(0.0), vec2(1.0));
                vec4 sample = texture2D(tOccluder, clampedCoord);
                sample *= illuminationDecay * weight;
                accum += sample;
                illuminationDecay *= decay;
            }
            
            vec4 original = texture2D(tDiffuse, vUv);
            gl_FragColor = original + accum * exposure;
        }
    `
};
```

## Occluder Pass

God rays ต้องการ occluder texture ที่บอกว่าส่วนไหนปล่อยแสงและส่วนไหนบังแสง:

```javascript
// สร้าง occluder pass
class GodRaysOccluderPass {
    constructor(renderer, scene, camera, lightSource, width, height) {
        this.renderer = renderer;
        this.scene = scene;
        this.camera = camera;
        this.lightSource = lightSource;
        
        this.renderTarget = new THREE.WebGLRenderTarget(width / 2, height / 2, {
            minFilter: THREE.LinearFilter,
            magFilter: THREE.LinearFilter
        });
        
        // Material สีดำสำหรับ occluders
        this.occluderMaterial = new THREE.MeshBasicMaterial({ color: 0x000000 });
        
        // Material สีขาวสำหรับ light source
        this.lightMaterial = new THREE.MeshBasicMaterial({ color: 0xffffff });
    }
    
    render(objects, lightMesh) {
        // บันทึก materials เดิม
        const savedMaterials = [];
        this.scene.traverse(obj => {
            if (obj.isMesh) {
                savedMaterials.push({ obj, mat: obj.material });
                obj.material = this.occluderMaterial;
            }
        });
        
        // ตั้งค่า light source ให้ขาว
        if (lightMesh) lightMesh.material = this.lightMaterial;
        
        // Render occluder pass
        this.renderer.setRenderTarget(this.renderTarget);
        this.renderer.render(this.scene, this.camera);
        this.renderer.setRenderTarget(null);
        
        // คืน materials เดิม
        savedMaterials.forEach(({ obj, mat }) => { obj.material = mat; });
    }
    
    get texture() {
        return this.renderTarget.texture;
    }
}
```

## ตัวอย่างที่ 1: Forest God Rays Scene

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Forest God Rays</title>
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
            width: 250px;
        }
        h3 { color: #90ee90; margin-bottom: 15px; }
        .ctrl { margin-bottom: 10px; }
        .ctrl label { display: flex; justify-content: space-between; font-size: 12px; margin-bottom: 3px; }
        .ctrl input { width: 100%; }
        #timeLabel {
            position: fixed;
            bottom: 20px;
            left: 50%;
            transform: translateX(-50%);
            color: white;
            background: rgba(0,0,0,0.6);
            padding: 8px 20px;
            border-radius: 20px;
            font-size: 14px;
        }
    </style>
</head>
<body>
    <div id="controls">
        <h3>🌲 Forest God Rays</h3>
        <div class="ctrl">
            <label><span>Ray Exposure</span><span id="expVal">0.18</span></label>
            <input type="range" id="exposure" min="0.01" max="0.5" step="0.01" value="0.18">
        </div>
        <div class="ctrl">
            <label><span>Ray Decay</span><span id="decVal">0.95</span></label>
            <input type="range" id="decay" min="0.8" max="0.99" step="0.005" value="0.95">
        </div>
        <div class="ctrl">
            <label><span>Ray Density</span><span id="denVal">0.8</span></label>
            <input type="range" id="density" min="0.1" max="1.5" step="0.05" value="0.8">
        </div>
        <div class="ctrl">
            <label><span>Time of Day</span><span id="todVal">Morning</span></label>
            <input type="range" id="timeOfDay" min="0" max="1" step="0.01" value="0.25">
        </div>
        <div class="ctrl">
            <label><span>Fog Density</span><span id="fogVal">0.03</span></label>
            <input type="range" id="fogDensity" min="0.005" max="0.1" step="0.005" value="0.03">
        </div>
    </div>
    <div id="timeLabel">🌅 ช่วงเช้าในป่า</div>

    <script type="module">
        import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js';
        import { EffectComposer } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/EffectComposer.js';
        import { RenderPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/RenderPass.js';
        import { ShaderPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/ShaderPass.js';
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
        scene.background = new THREE.Color(0xd4a57a);
        scene.fog = new THREE.FogExp2(0xd4a57a, 0.03);

        const camera = new THREE.PerspectiveCamera(60, W/H, 0.1, 200);
        camera.position.set(0, 2, 10);
        camera.lookAt(0, 3, -10);

        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.dampingFactor = 0.05;
        controls.target.set(0, 3, -10);

        // Sun light
        const sunLight = new THREE.DirectionalLight(0xffe066, 3);
        sunLight.position.set(-8, 20, -10);
        sunLight.castShadow = true;
        sunLight.shadow.mapSize.set(4096, 4096);
        sunLight.shadow.camera.near = 0.5;
        sunLight.shadow.camera.far = 200;
        sunLight.shadow.camera.left = -30;
        sunLight.shadow.camera.right = 30;
        sunLight.shadow.camera.top = 40;
        sunLight.shadow.camera.bottom = -30;
        sunLight.shadow.bias = -0.001;
        scene.add(sunLight);
        scene.add(new THREE.AmbientLight(0x334455, 0.5));

        // Sun mesh (visible orb)
        const sunMesh = new THREE.Mesh(
            new THREE.SphereGeometry(0.5, 16, 16),
            new THREE.MeshBasicMaterial({ color: 0xffffff })
        );
        sunMesh.position.copy(sunLight.position.clone().normalize().multiplyScalar(80));
        scene.add(sunMesh);

        // พื้นป่า
        const groundMat = new THREE.MeshStandardMaterial({ color: 0x2d4a1e, roughness: 0.95 });
        const ground = new THREE.Mesh(new THREE.PlaneGeometry(100, 100), groundMat);
        ground.rotation.x = -Math.PI / 2;
        ground.receiveShadow = true;
        scene.add(ground);

        // สร้างต้นไม้
        function createTree(x, z, height, trunkH) {
            const g = new THREE.Group();
            
            // ลำต้น
            const trunkMat = new THREE.MeshStandardMaterial({ color: 0x4a2e0e, roughness: 0.9 });
            const trunk = new THREE.Mesh(
                new THREE.CylinderGeometry(0.15, 0.25, trunkH, 8),
                trunkMat
            );
            trunk.position.y = trunkH / 2;
            trunk.castShadow = true;
            trunk.receiveShadow = true;
            g.add(trunk);
            
            // ใบ - หลายชั้น
            const leafColors = [0x1a4a0e, 0x2a5e1a, 0x1e3a0e];
            const leafLayers = Math.floor(height / 2);
            for (let i = 0; i < leafLayers; i++) {
                const r = (height - i) * 0.3;
                const h = height * 0.4;
                const leavesGeo = new THREE.ConeGeometry(r, h, 8);
                const leavesMat = new THREE.MeshStandardMaterial({
                    color: leafColors[i % 3],
                    roughness: 0.9
                });
                const leaves = new THREE.Mesh(leavesGeo, leavesMat);
                leaves.position.y = trunkH + h / 2 + i * (height * 0.15);
                leaves.castShadow = true;
                leaves.receiveShadow = true;
                g.add(leaves);
            }
            
            g.position.set(x, 0, z);
            scene.add(g);
            return g;
        }

        // แถวต้นไม้
        const trees = [];
        const treePositions = [
            [-5, -5, 12, 4], [-3, -15, 10, 3.5], [4, -8, 14, 5],
            [7, -20, 11, 4], [-8, -25, 13, 4.5], [2, -30, 9, 3],
            [-6, -12, 15, 5.5], [5, -35, 12, 4], [-4, -40, 11, 3.5],
            [8, -2, 10, 4], [-9, 0, 13, 5], [6, -42, 14, 6],
            [-2, -50, 12, 4.5], [3, -18, 16, 5], [-7, -28, 11, 4]
        ];
        
        treePositions.forEach(([x, z, h, trunkH]) => {
            trees.push(createTree(x, z, h, trunkH));
        });

        // ใบไม้ร่วง (particles)
        const leafCount = 500;
        const leafPositions = new Float32Array(leafCount * 3);
        const leafRotations = new Float32Array(leafCount);
        const leafVelocities = new Float32Array(leafCount * 3);
        
        for (let i = 0; i < leafCount; i++) {
            leafPositions[i * 3] = (Math.random() - 0.5) * 40;
            leafPositions[i * 3 + 1] = Math.random() * 15;
            leafPositions[i * 3 + 2] = -Math.random() * 50;
            leafVelocities[i * 3] = (Math.random() - 0.5) * 0.3;
            leafVelocities[i * 3 + 1] = -(0.5 + Math.random() * 0.5);
            leafVelocities[i * 3 + 2] = (Math.random() - 0.5) * 0.2;
        }
        
        const leafGeo = new THREE.BufferGeometry();
        leafGeo.setAttribute('position', new THREE.BufferAttribute(leafPositions, 3));
        const leafMat = new THREE.PointsMaterial({
            color: 0x88aa44,
            size: 0.1,
            transparent: true,
            opacity: 0.7
        });
        const leafParticles = new THREE.Points(leafGeo, leafMat);
        scene.add(leafParticles);

        // ฝุ่นในแสง (dust motes)
        const dustCount = 1000;
        const dustPositions = new Float32Array(dustCount * 3);
        for (let i = 0; i < dustCount; i++) {
            dustPositions[i * 3] = (Math.random() - 0.5) * 20;
            dustPositions[i * 3 + 1] = Math.random() * 10;
            dustPositions[i * 3 + 2] = -Math.random() * 20;
        }
        const dustGeo = new THREE.BufferGeometry();
        dustGeo.setAttribute('position', new THREE.BufferAttribute(dustPositions, 3));
        const dustMat = new THREE.PointsMaterial({
            color: 0xffe090,
            size: 0.05,
            transparent: true,
            opacity: 0.4
        });
        scene.add(new THREE.Points(dustGeo, dustMat));

        // Post-processing
        const occluderTarget = new THREE.WebGLRenderTarget(W / 2, H / 2);
        const godRaysTarget = new THREE.WebGLRenderTarget(W, H);

        // Occluder material
        const occluderMat = new THREE.MeshBasicMaterial({ color: 0x000000, side: THREE.FrontSide });
        const lightMat = new THREE.MeshBasicMaterial({ color: 0xffffff });

        // God Rays shader
        const GodRaysShader = {
            uniforms: {
                tDiffuse: { value: null },
                tOccluder: { value: null },
                lightPosition: { value: new THREE.Vector2(0.5, 0.8) },
                exposure: { value: 0.18 },
                decay: { value: 0.95 },
                density: { value: 0.8 },
                weight: { value: 0.4 }
            },
            vertexShader: `varying vec2 vUv; void main() { vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
            fragmentShader: `
                uniform sampler2D tDiffuse;
                uniform sampler2D tOccluder;
                uniform vec2 lightPosition;
                uniform float exposure;
                uniform float decay;
                uniform float density;
                uniform float weight;
                
                varying vec2 vUv;
                
                void main() {
                    const int NUM_SAMPLES = 80;
                    vec2 texCoord = vUv;
                    vec2 delta = (texCoord - lightPosition) / float(NUM_SAMPLES) * density;
                    float illuminationDecay = 1.0;
                    vec4 accum = vec4(0.0);
                    
                    for (int i = 0; i < NUM_SAMPLES; i++) {
                        texCoord -= delta;
                        vec2 clamped = clamp(texCoord, vec2(0.0), vec2(1.0));
                        vec4 s = texture2D(tOccluder, clamped);
                        s *= illuminationDecay * weight;
                        accum += s;
                        illuminationDecay *= decay;
                    }
                    
                    vec4 base = texture2D(tDiffuse, vUv);
                    gl_FragColor = base + accum * exposure;
                }
            `
        };

        const composer = new EffectComposer(renderer);
        const renderPass = new RenderPass(scene, camera);
        composer.addPass(renderPass);

        const godRaysPass = new ShaderPass(GodRaysShader);
        composer.addPass(godRaysPass);
        composer.addPass(new OutputPass());

        // Helper: คำนวณ screen position ของแสง
        function getLightScreenPosition(light, camera, width, height) {
            const v = light.position.clone().normalize().multiplyScalar(80);
            v.project(camera);
            return new THREE.Vector2((v.x + 1) / 2, (v.y + 1) / 2);
        }

        // Time of day
        const timeColors = {
            night: { sky: 0x050510, sunColor: 0xffa040, ambColor: 0x112244, fogColor: 0x050510, sunIntensity: 0 },
            sunrise: { sky: 0xff8844, sunColor: 0xff9933, ambColor: 0x442233, fogColor: 0xff8844, sunIntensity: 2 },
            morning: { sky: 0xd4a57a, sunColor: 0xffe066, ambColor: 0x334455, fogColor: 0xd4a57a, sunIntensity: 3 },
            noon: { sky: 0x87ceeb, sunColor: 0xffffff, ambColor: 0x445566, fogColor: 0xaaccee, sunIntensity: 4 },
            afternoon: { sky: 0xd4996b, sunColor: 0xffcc44, ambColor: 0x554433, fogColor: 0xd4996b, sunIntensity: 2.5 },
            sunset: { sky: 0xff6633, sunColor: 0xff8844, ambColor: 0x443322, fogColor: 0xff6633, sunIntensity: 2 }
        };

        const timeKeys = Object.keys(timeColors);
        const todLabel = document.getElementById('timeLabel');
        const todNames = ['🌃 กลางคืน', '🌅 พระอาทิตย์ขึ้น', '🌅 ช่วงเช้า', '☀️ เที่ยง', '🌆 บ่าย', '🌇 พระอาทิตย์ตก'];

        function setTimeOfDay(t) {
            const idx = Math.floor(t * (timeKeys.length - 1));
            const frac = (t * (timeKeys.length - 1)) - idx;
            const c1 = timeColors[timeKeys[Math.min(idx, timeKeys.length-1)]];
            const c2 = timeColors[timeKeys[Math.min(idx+1, timeKeys.length-1)]];
            
            // Interpolate
            const skyColor = new THREE.Color(c1.sky).lerp(new THREE.Color(c2.sky), frac);
            scene.background = skyColor;
            scene.fog.color = skyColor;
            sunLight.color.set(c1.sunColor).lerp(new THREE.Color(c2.sunColor), frac);
            sunLight.intensity = c1.sunIntensity + (c2.sunIntensity - c1.sunIntensity) * frac;
            
            // ย้ายตำแหน่งดวงอาทิตย์
            const angle = t * Math.PI;
            sunLight.position.set(
                Math.cos(angle) * 20,
                Math.sin(angle) * 30,
                -10
            );
            sunMesh.position.copy(sunLight.position.clone().normalize().multiplyScalar(80));
            
            todLabel.textContent = todNames[Math.round(t * (todNames.length - 1))];
        }

        // UI
        document.getElementById('exposure').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            godRaysPass.uniforms.exposure.value = v;
            document.getElementById('expVal').textContent = v.toFixed(2);
        });
        document.getElementById('decay').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            godRaysPass.uniforms.decay.value = v;
            document.getElementById('decVal').textContent = v.toFixed(3);
        });
        document.getElementById('density').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            godRaysPass.uniforms.density.value = v;
            document.getElementById('denVal').textContent = v.toFixed(2);
        });
        document.getElementById('timeOfDay').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            setTimeOfDay(v);
            document.getElementById('todVal').textContent = todNames[Math.round(v * (todNames.length - 1))].replace(/^\S+\s/, '');
        });
        document.getElementById('fogDensity').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            scene.fog.density = v;
            document.getElementById('fogVal').textContent = v.toFixed(3);
        });

        window.addEventListener('resize', () => {
            const w = window.innerWidth, h = window.innerHeight;
            camera.aspect = w / h;
            camera.updateProjectionMatrix();
            renderer.setSize(w, h);
            composer.setSize(w, h);
        });

        // Render occluder scene
        const blackMat = new THREE.MeshBasicMaterial({ color: 0x000000 });
        const whiteMat = new THREE.MeshBasicMaterial({ color: 0xffffff });

        function renderOccluder() {
            // Override materials to black
            const savedMats = [];
            scene.traverse(obj => {
                if (obj.isMesh) {
                    savedMats.push({ obj, mat: obj.material });
                    obj.material = blackMat;
                }
            });
            
            // Sun = white
            sunMesh.material = whiteMat;
            
            renderer.setRenderTarget(occluderTarget);
            renderer.render(scene, camera);
            renderer.setRenderTarget(null);
            
            // Restore
            savedMats.forEach(({ obj, mat }) => { obj.material = mat; });
        }

        const clock = new THREE.Clock();
        function animate() {
            requestAnimationFrame(animate);
            const dt = clock.getDelta();
            const time = clock.getElapsedTime();

            // Update leaves
            const pos = leafGeo.attributes.position.array;
            for (let i = 0; i < leafCount; i++) {
                pos[i * 3] += leafVelocities[i * 3] * dt + Math.sin(time + i) * 0.01;
                pos[i * 3 + 1] += leafVelocities[i * 3 + 1] * dt;
                pos[i * 3 + 2] += leafVelocities[i * 3 + 2] * dt;
                
                if (pos[i * 3 + 1] < 0) {
                    pos[i * 3 + 1] = 12;
                    pos[i * 3] = (Math.random() - 0.5) * 40;
                    pos[i * 3 + 2] = -Math.random() * 50;
                }
            }
            leafGeo.attributes.position.needsUpdate = true;

            // Render occluder
            renderOccluder();
            
            // Update light screen position
            const lightScreenPos = getLightScreenPosition(sunLight, camera, W, H);
            godRaysPass.uniforms.lightPosition.value.copy(lightScreenPos);
            godRaysPass.uniforms.tOccluder.value = occluderTarget.texture;
            
            controls.update();
            composer.render();
        }

        setTimeOfDay(0.25);
        animate();
    </script>
</body>
</html>
```

## Volumetric Spotlight Effect

```javascript
// Volumetric Spotlight ด้วย Cone Mesh + Custom Shader
class VolumetricSpotlight {
    constructor(position, target, color, angle = Math.PI / 8, height = 10) {
        this.group = new THREE.Group();
        
        // Actual spotlight
        this.spotlight = new THREE.SpotLight(color, 2, height * 2, angle);
        this.spotlight.position.copy(position);
        this.spotlight.castShadow = true;
        this.group.add(this.spotlight);
        
        const targetObj = new THREE.Object3D();
        targetObj.position.copy(target);
        this.group.add(targetObj);
        this.spotlight.target = targetObj;
        
        // Volumetric cone
        const radius = Math.tan(angle) * height;
        const coneGeo = new THREE.ConeGeometry(radius, height, 32, 1, true);
        coneGeo.translate(0, -height / 2, 0);
        
        const coneMat = new THREE.ShaderMaterial({
            uniforms: {
                color: { value: new THREE.Color(color) },
                intensity: { value: 0.3 },
                time: { value: 0.0 },
                dustEnabled: { value: 1 }
            },
            vertexShader: `
                varying vec3 vPos;
                varying vec2 vUv;
                void main() {
                    vPos = position;
                    vUv = uv;
                    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
                }
            `,
            fragmentShader: `
                uniform vec3 color;
                uniform float intensity;
                uniform float time;
                uniform int dustEnabled;
                
                varying vec3 vPos;
                varying vec2 vUv;
                
                float hash(vec3 p) {
                    p = fract(p * 0.3183099 + 0.1);
                    p *= 17.0;
                    return fract(p.x * p.y * p.z * (p.x + p.y + p.z));
                }
                
                float noise(vec3 p) {
                    vec3 i = floor(p);
                    vec3 f = fract(p);
                    f = f * f * (3.0 - 2.0 * f);
                    return mix(mix(mix(hash(i), hash(i + vec3(1,0,0)), f.x),
                                  mix(hash(i + vec3(0,1,0)), hash(i + vec3(1,1,0)), f.x), f.y),
                               mix(mix(hash(i + vec3(0,0,1)), hash(i + vec3(1,0,1)), f.x),
                                  mix(hash(i + vec3(0,1,1)), hash(i + vec3(1,1,1)), f.x), f.y), f.z);
                }
                
                void main() {
                    // Gradient จากบนลงล่าง
                    float gradient = 1.0 - vUv.y;
                    
                    // Radial fade (ขอบ cone)
                    float radialFade = 1.0 - pow(abs(vUv.x * 2.0 - 1.0), 2.0);
                    
                    float alpha = gradient * radialFade * intensity;
                    
                    // เพิ่ม dust effect
                    if (dustEnabled == 1) {
                        float dust = noise(vPos * 5.0 + vec3(0, time * 0.5, 0));
                        dust = pow(dust, 3.0) * 0.5;
                        alpha += dust * gradient * 0.3;
                    }
                    
                    alpha = clamp(alpha, 0.0, 0.8);
                    gl_FragColor = vec4(color, alpha);
                }
            `,
            transparent: true,
            side: THREE.FrontSide,
            depthWrite: false,
            blending: THREE.AdditiveBlending
        });
        
        this.coneMesh = new THREE.Mesh(coneGeo, coneMat);
        this.coneMesh.position.copy(position);
        
        // หมุน cone ให้ชี้ไปยัง target
        const dir = new THREE.Vector3().subVectors(target, position).normalize();
        const quaternion = new THREE.Quaternion().setFromUnitVectors(new THREE.Vector3(0, -1, 0), dir);
        this.coneMesh.quaternion.copy(quaternion);
        
        this.group.add(this.coneMesh);
    }
    
    update(time) {
        this.coneMesh.material.uniforms.time.value = time;
    }
    
    get mesh() { return this.group; }
}
```

## ตัวอย่างที่ 2: Dungeon Torch Volumetric Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Dungeon Torch Volumetric</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; font-family: Arial, sans-serif; }
        canvas { display: block; }
        #ui {
            position: fixed;
            bottom: 20px;
            left: 50%;
            transform: translateX(-50%);
            background: rgba(0,0,0,0.8);
            color: #cc9933;
            padding: 10px 20px;
            border-radius: 20px;
            font-size: 13px;
            text-align: center;
        }
        #controls {
            position: fixed;
            top: 20px;
            left: 20px;
            background: rgba(0,0,0,0.8);
            color: white;
            padding: 15px;
            border-radius: 10px;
            width: 220px;
        }
        h3 { color: #cc9933; margin-bottom: 12px; }
        .ctrl { margin-bottom: 8px; }
        .ctrl label { display: flex; justify-content: space-between; font-size: 12px; margin-bottom: 3px; }
        .ctrl input { width: 100%; }
    </style>
</head>
<body>
    <div id="controls">
        <h3>🔥 Dungeon Torch</h3>
        <div class="ctrl">
            <label><span>Flame Intensity</span><span id="fiVal">2.0</span></label>
            <input type="range" id="flameInt" min="0.5" max="5" step="0.1" value="2">
        </div>
        <div class="ctrl">
            <label><span>Smoke Density</span><span id="sdVal">0.3</span></label>
            <input type="range" id="smokeDen" min="0" max="1" step="0.05" value="0.3">
        </div>
        <div class="ctrl">
            <label><span>Flicker Speed</span><span id="fsVal">3.0</span></label>
            <input type="range" id="flickerSpd" min="0.5" max="10" step="0.5" value="3">
        </div>
        <div class="ctrl">
            <label><span>Volumetric Cone</span><span id="vcVal">0.3</span></label>
            <input type="range" id="volCone" min="0" max="1" step="0.05" value="0.3">
        </div>
    </div>
    <div id="ui">WASD = เดิน | Mouse = หมุนมุมมอง | ลองเดินเข้าไปใกล้คบเพลิง</div>

    <script type="module">
        import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js';
        import { EffectComposer } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/EffectComposer.js';
        import { RenderPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/RenderPass.js';
        import { UnrealBloomPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/UnrealBloomPass.js';
        import { OutputPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/OutputPass.js';

        const W = window.innerWidth, H = window.innerHeight;
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(W, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        renderer.shadowMap.enabled = true;
        renderer.shadowMap.type = THREE.PCFSoftShadowMap;
        document.body.appendChild(renderer.domElement);

        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x000005);
        scene.fog = new THREE.Fog(0x000005, 5, 25);

        const camera = new THREE.PerspectiveCamera(75, W/H, 0.1, 100);
        camera.position.set(0, 1.6, 5);

        // FPS-like controls
        const keys = {};
        let isLocked = false;
        let yaw = 0, pitch = 0;
        
        document.addEventListener('click', () => {
            if (!isLocked) renderer.domElement.requestPointerLock();
        });
        document.addEventListener('pointerlockchange', () => {
            isLocked = document.pointerLockElement === renderer.domElement;
        });
        document.addEventListener('mousemove', (e) => {
            if (!isLocked) return;
            yaw -= e.movementX * 0.002;
            pitch = Math.max(-Math.PI/3, Math.min(Math.PI/3, pitch - e.movementY * 0.002));
        });
        document.addEventListener('keydown', e => keys[e.key.toLowerCase()] = true);
        document.addEventListener('keyup', e => keys[e.key.toLowerCase()] = false);

        // Dungeon walls
        const stoneMat = new THREE.MeshStandardMaterial({ color: 0x222222, roughness: 0.95 });
        
        // Floor
        const floor = new THREE.Mesh(new THREE.PlaneGeometry(20, 30), stoneMat);
        floor.rotation.x = -Math.PI / 2;
        floor.receiveShadow = true;
        scene.add(floor);
        
        // Ceiling
        const ceiling = new THREE.Mesh(new THREE.PlaneGeometry(20, 30), stoneMat);
        ceiling.rotation.x = Math.PI / 2;
        ceiling.position.y = 4;
        scene.add(ceiling);
        
        // Walls
        const wallMat = new THREE.MeshStandardMaterial({ color: 0x1a1a1a, roughness: 0.95 });
        [-5, 5].forEach(x => {
            const wall = new THREE.Mesh(new THREE.BoxGeometry(0.3, 4, 30), wallMat);
            wall.position.set(x, 2, 0);
            wall.receiveShadow = true;
            wall.castShadow = true;
            scene.add(wall);
        });
        
        // End wall
        const endWall = new THREE.Mesh(new THREE.BoxGeometry(10, 4, 0.3), wallMat);
        endWall.position.set(0, 2, -14);
        scene.add(endWall);
        
        // Pillars
        const pillarMat = new THREE.MeshStandardMaterial({ color: 0x333333, roughness: 0.9 });
        for (let i = 0; i < 4; i++) {
            [-3, 3].forEach(x => {
                const pillar = new THREE.Mesh(new THREE.BoxGeometry(0.8, 4, 0.8), pillarMat);
                pillar.position.set(x, 2, -i * 4);
                pillar.castShadow = true;
                pillar.receiveShadow = true;
                scene.add(pillar);
            });
        }

        // Torch system
        const torches = [];
        const torchPositions = [
            [-4, 2.2, 0], [4, 2.2, 0],
            [-4, 2.2, -5], [4, 2.2, -5],
            [-4, 2.2, -10], [4, 2.2, -10]
        ];

        function createTorch(x, y, z) {
            const g = new THREE.Group();
            
            // Torch stick
            const stickMat = new THREE.MeshStandardMaterial({ color: 0x5d3a1a, roughness: 0.9 });
            const stick = new THREE.Mesh(new THREE.CylinderGeometry(0.04, 0.04, 0.5, 6), stickMat);
            stick.castShadow = true;
            g.add(stick);
            
            // Bracket on wall
            const bracketMat = new THREE.MeshStandardMaterial({ color: 0x333333, metalness: 0.6 });
            const bracket = new THREE.Mesh(new THREE.BoxGeometry(0.15, 0.08, 0.15), bracketMat);
            bracket.position.set(0, 0, 0);
            g.add(bracket);
            
            // Flame particles
            const flameCount = 30;
            const flamePos = new Float32Array(flameCount * 3);
            const flameVel = [];
            for (let i = 0; i < flameCount; i++) {
                flamePos[i * 3] = (Math.random() - 0.5) * 0.1;
                flamePos[i * 3 + 1] = 0.25 + Math.random() * 0.1;
                flamePos[i * 3 + 2] = (Math.random() - 0.5) * 0.1;
                flameVel.push({
                    x: (Math.random() - 0.5) * 0.3,
                    y: 0.5 + Math.random() * 0.5,
                    life: Math.random()
                });
            }
            const flameGeo = new THREE.BufferGeometry();
            flameGeo.setAttribute('position', new THREE.BufferAttribute(flamePos, 3));
            const flameMat = new THREE.PointsMaterial({
                color: 0xff6600,
                size: 0.12,
                transparent: true,
                opacity: 0.9,
                blending: THREE.AdditiveBlending,
                depthWrite: false
            });
            const flame = new THREE.Points(flameGeo, flameMat);
            g.add(flame);
            
            // Point light
            const light = new THREE.PointLight(0xff9933, 2, 8);
            light.position.y = 0.3;
            light.castShadow = true;
            light.shadow.mapSize.set(256, 256);
            g.add(light);
            
            // Volumetric cone
            const coneGeo = new THREE.ConeGeometry(1.5, 3, 16, 1, true);
            const coneMat = new THREE.ShaderMaterial({
                uniforms: {
                    time: { value: 0 },
                    intensity: { value: 0.3 }
                },
                vertexShader: `
                    varying vec2 vUv;
                    varying vec3 vPos;
                    void main() {
                        vUv = uv;
                        vPos = position;
                        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
                    }
                `,
                fragmentShader: `
                    uniform float time;
                    uniform float intensity;
                    varying vec2 vUv;
                    varying vec3 vPos;
                    void main() {
                        float radial = 1.0 - length(vec2(vUv.x - 0.5, 0) * 2.0);
                        float vertical = 1.0 - vUv.y;
                        float alpha = radial * vertical * intensity;
                        // noise
                        float n = sin(vPos.x * 10.0 + time * 2.0) * sin(vPos.z * 8.0 - time) * 0.1;
                        alpha += n * vertical;
                        gl_FragColor = vec4(1.0, 0.6, 0.2, clamp(alpha, 0.0, 0.6));
                    }
                `,
                transparent: true,
                side: THREE.DoubleSide,
                depthWrite: false,
                blending: THREE.AdditiveBlending
            });
            const cone = new THREE.Mesh(coneGeo, coneMat);
            cone.position.y = 0.3 - 1.5;
            g.add(cone);
            
            g.position.set(x, y, z);
            
            // ถ้าอยู่ข้างซ้าย ชี้ไปทางขวา
            if (x < 0) g.rotation.z = Math.PI / 6;
            else g.rotation.z = -Math.PI / 6;
            
            scene.add(g);
            return { group: g, flame, flameGeo, flameVel, flamePos, light, cone };
        }

        torchPositions.forEach(([x, y, z]) => {
            torches.push(createTorch(x, y, z));
        });

        // Ambient (very dark)
        scene.add(new THREE.AmbientLight(0x110a05, 0.3));

        // Post-processing
        const composer = new EffectComposer(renderer);
        composer.addPass(new RenderPass(scene, camera));
        const bloom = new UnrealBloomPass(new THREE.Vector2(W, H), 0.8, 0.6, 0.6);
        composer.addPass(bloom);
        composer.addPass(new OutputPass());

        // UI
        document.getElementById('flameInt').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            torches.forEach(t => t.light.intensity = v);
            document.getElementById('fiVal').textContent = v.toFixed(1);
        });
        document.getElementById('smokeDen').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            document.getElementById('sdVal').textContent = v.toFixed(2);
            scene.fog.near = 5 - v * 4;
        });
        document.getElementById('volCone').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            torches.forEach(t => t.cone.material.uniforms.intensity.value = v);
            document.getElementById('vcVal').textContent = v.toFixed(2);
        });

        window.addEventListener('resize', () => {
            const w = window.innerWidth, h = window.innerHeight;
            camera.aspect = w / h;
            camera.updateProjectionMatrix();
            renderer.setSize(w, h);
            composer.setSize(w, h);
        });

        const clock = new THREE.Clock();
        const playerPos = new THREE.Vector3(0, 1.6, 5);
        let flickerSpeed = 3.0;

        document.getElementById('flickerSpd').addEventListener('input', (e) => {
            flickerSpeed = parseFloat(e.target.value);
            document.getElementById('fsVal').textContent = flickerSpeed.toFixed(1);
        });

        function animate() {
            requestAnimationFrame(animate);
            const dt = clock.getDelta();
            const time = clock.getElapsedTime();

            // Player movement
            const moveSpeed = 4.0;
            const forward = new THREE.Vector3(-Math.sin(yaw), 0, -Math.cos(yaw));
            const right = new THREE.Vector3(Math.cos(yaw), 0, -Math.sin(yaw));
            
            if (keys['w']) playerPos.addScaledVector(forward, moveSpeed * dt);
            if (keys['s']) playerPos.addScaledVector(forward, -moveSpeed * dt);
            if (keys['a']) playerPos.addScaledVector(right, -moveSpeed * dt);
            if (keys['d']) playerPos.addScaledVector(right, moveSpeed * dt);
            
            playerPos.x = Math.max(-4.5, Math.min(4.5, playerPos.x));
            playerPos.z = Math.max(-13, Math.min(6, playerPos.z));
            
            camera.position.copy(playerPos);
            camera.rotation.order = 'YXZ';
            camera.rotation.y = yaw;
            camera.rotation.x = pitch;

            // Update torches
            torches.forEach((torch, ti) => {
                // Flame flicker
                const flicker = Math.sin(time * flickerSpeed + ti * 2.3) * 0.3 +
                                Math.sin(time * flickerSpeed * 1.7 + ti) * 0.2 + 1.0;
                torch.light.intensity = parseFloat(document.getElementById('flameInt').value) * flicker;
                torch.cone.material.uniforms.time.value = time;
                
                // Update flame particles
                const pos = torch.flamePos;
                for (let i = 0; i < 30; i++) {
                    torch.flameVel[i].life += dt * 2;
                    pos[i * 3] += torch.flameVel[i].x * dt;
                    pos[i * 3 + 1] += torch.flameVel[i].y * dt;
                    pos[i * 3 + 2] += (Math.random() - 0.5) * 0.02;
                    
                    if (torch.flameVel[i].life > 1) {
                        torch.flameVel[i].life = 0;
                        pos[i * 3] = (Math.random() - 0.5) * 0.1;
                        pos[i * 3 + 1] = 0.25;
                        pos[i * 3 + 2] = (Math.random() - 0.5) * 0.1;
                        torch.flameVel[i].x = (Math.random() - 0.5) * 0.3;
                        torch.flameVel[i].y = 0.5 + Math.random() * 0.5;
                    }
                    
                    // Color based on height (ส่วนล่าง = แดง, ส่วนบน = เหลือง)
                    const t2 = torch.flameVel[i].life;
                    torch.flame.material.color.setHSL(0.08 - t2 * 0.05, 1, 0.5 + t2 * 0.2);
                    torch.flame.material.opacity = 0.9 - t2 * 0.5;
                }
                torch.flameGeo.attributes.position.needsUpdate = true;
            });

            composer.render();
        }
        animate();
    </script>
</body>
</html>
```

## Fog + Point Light Volumetric Effect

```javascript
// เทคนิคง่ายๆ: ใช้ fog กับ point lights
// เมื่อ point light อยู่ในหมอก แสงจะดู volumetric

// FogExp2 ให้ผลที่ดูเป็นธรรมชาติกว่า Fog ธรรมดา
scene.fog = new THREE.FogExp2(0x000011, 0.05);

// Point lights ใน fog
const torchLight = new THREE.PointLight(0xff8822, 3, 8);
torchLight.position.set(0, 2, 0);
scene.add(torchLight);

// เพิ่ม lens flare effect (ไม่ใช่ standard THREE แต่เป็นแนวคิด)
// ทำด้วย canvas texture ที่ overlay บน screen

// Sprite สำหรับ glow ที่แสง
const glowSprite = new THREE.Sprite(
    new THREE.SpriteMaterial({
        map: glowTexture,
        color: 0xff8822,
        transparent: true,
        blending: THREE.AdditiveBlending,
        depthWrite: false
    })
);
glowSprite.position.copy(torchLight.position);
glowSprite.scale.set(2, 2, 1);
scene.add(glowSprite);
```

## Performance Tips สำหรับ Volumetric Effects

```javascript
// 1. ลด god ray samples ตาม quality setting
const qualityLevels = {
    low: { samples: 20, resolution: 0.5 },
    medium: { samples: 50, resolution: 0.75 },
    high: { samples: 100, resolution: 1.0 }
};

// 2. ทำ god rays เฉพาะเมื่อแสงอยู่ใน frustum
function isSunVisible(sunPosition, camera) {
    const frustum = new THREE.Frustum();
    const matrix = new THREE.Matrix4().multiplyMatrices(
        camera.projectionMatrix,
        camera.matrixWorldInverse
    );
    frustum.setFromProjectionMatrix(matrix);
    return frustum.containsPoint(sunPosition);
}

// 3. ใช้ temporal accumulation
// สำหรับ volumetric effects ที่มั่นคง สามารถ accumulate ผลจาก frame ก่อนๆ

// 4. Precompute volumetric lighting สำหรับ static scenes
// Bake volumetric lighting เป็น lightmap

// 5. ใช้ billboard particles แทน ray marching
// สำหรับ smoke/fog ใช้ particle system ที่ render เป็น alpha-blended sprites
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Stained Glass God Rays
สร้าง church interior ที่มี stained glass windows:
- ทำสีของ god rays ตาม glass color
- เปลี่ยนมุมแสงตามเวลา (เช้า/เย็น)

### แบบฝึกหัดที่ 2: Underwater Volumetric Light
สร้างฉากใต้น้ำ:
- Light shafts จากผิวน้ำ
- Caustic patterns บนพื้น
- Blue-green tint ตามความลึก

### แบบฝึกหัดที่ 3: Stage Spotlight
สร้าง concert stage:
- Moving spotlights
- Volumetric cones ของแต่ละ spotlight
- Haze effect (หมอกบนเวที)

### แบบฝึกหัดที่ 4: Neon Volumetric
สร้าง cyberpunk alley:
- Neon signs ที่มี glow
- Rain + neon reflections
- Volumetric fog + neon colored light

---

**สรุปบทที่ 53:**
- God rays ด้วย radial blur เป็นเทคนิคที่ง่ายและมีประสิทธิภาพสำหรับ screen space
- Occluder pass จำเป็นสำหรับ god rays ที่ถูกต้อง
- Volumetric cone meshes ด้วย additive blending สร้าง spotlight effect ที่ดูดี
- Fog + point lights เป็น pseudo-volumetric ที่ประหยัด GPU
- Performance เป็นสิ่งสำคัญ - ลด samples หรือใช้ temporal accumulation

**บทต่อไป:** Toon & Cel Shading
