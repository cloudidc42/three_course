# Part 54: Toon & Cel Shading

## แนวคิด Toon / Cel Shading

Toon Shading หรือ Cel Shading เป็นเทคนิคการแรเงาที่จำลองลักษณะของการ์ตูนหรือ animation แบบ hand-drawn แทนที่จะใช้การ interpolation แบบ smooth เราจะ quantize (แบ่งชั้น) ค่าแสงออกเป็นขั้นๆ เหมือนกับการแรเงาด้วยมือ

ตัวอย่างเกมที่ใช้ Toon Shading:
- **The Legend of Zelda: Wind Waker** - ผู้บุกเบิก 3D cel shading
- **Borderlands series** - ใช้ outlines และ cel shading
- **Guilty Gear Strive** - cel shading ขั้นสูงใน 3D
- **Dragon Ball FighterZ** - ดูเหมือน 2D anime

## MeshToonMaterial ใน Three.js

```javascript
// วิธีที่ง่ายที่สุดคือใช้ MeshToonMaterial
const toonMaterial = new THREE.MeshToonMaterial({
    color: 0xff6644,
    // gradientMap กำหนดขั้นของการแรเงา
    gradientMap: createGradientTexture([0.0, 0.5, 0.8, 1.0])
});

// สร้าง gradient texture สำหรับ toon shading
function createGradientTexture(steps) {
    const canvas = document.createElement('canvas');
    canvas.width = steps.length * 64;
    canvas.height = 1;
    const ctx = canvas.getContext('2d');
    
    steps.forEach((brightness, i) => {
        ctx.fillStyle = `hsl(0, 0%, ${brightness * 100}%)`;
        ctx.fillRect(i * 64, 0, 64, 1);
    });
    
    const texture = new THREE.CanvasTexture(canvas);
    texture.minFilter = THREE.NearestFilter;
    texture.magFilter = THREE.NearestFilter;
    return texture;
}

// 3-step gradient (shadow, mid, highlight)
const threeStepGradient = createGradientTexture([0.1, 0.5, 1.0]);
// 2-step gradient (classic cel-shading)
const twoStepGradient = createGradientTexture([0.0, 0.7]);
// 4-step gradient (detailed shading)
const fourStepGradient = createGradientTexture([0.05, 0.3, 0.65, 1.0]);
```

## Custom Toon Shader (Quantized Lighting)

```javascript
// Custom Toon Shader พร้อม quantization
const ToonShader = {
    uniforms: {
        color: { value: new THREE.Color(0xff6644) },
        lightDirection: { value: new THREE.Vector3(0.5, 1.0, 0.5).normalize() },
        lightColor: { value: new THREE.Color(0xffffff) },
        ambientColor: { value: new THREE.Color(0x334466) },
        steps: { value: 3 },           // จำนวนขั้นการแรเงา
        specularPower: { value: 32.0 }, // ความแหลมคมของ specular
        specularIntensity: { value: 0.5 },
        rimLightColor: { value: new THREE.Color(0xffffff) },
        rimLightIntensity: { value: 0.5 }
    },
    vertexShader: `
        varying vec3 vNormal;
        varying vec3 vViewPos;
        varying vec3 vWorldPos;
        
        void main() {
            vNormal = normalize(normalMatrix * normal);
            vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
            vViewPos = -mvPosition.xyz;
            vWorldPos = (modelMatrix * vec4(position, 1.0)).xyz;
            gl_Position = projectionMatrix * mvPosition;
        }
    `,
    fragmentShader: `
        uniform vec3 color;
        uniform vec3 lightDirection;
        uniform vec3 lightColor;
        uniform vec3 ambientColor;
        uniform int steps;
        uniform float specularPower;
        uniform float specularIntensity;
        uniform vec3 rimLightColor;
        uniform float rimLightIntensity;
        
        varying vec3 vNormal;
        varying vec3 vViewPos;
        varying vec3 vWorldPos;
        
        // Quantize: แปลง continuous value เป็น discrete steps
        float quantize(float value, int numSteps) {
            float step = 1.0 / float(numSteps);
            return floor(value / step) * step;
        }
        
        void main() {
            vec3 normal = normalize(vNormal);
            vec3 viewDir = normalize(vViewPos);
            vec3 lightDir = normalize(lightDirection);
            
            // Diffuse lighting (quantized)
            float NdotL = max(0.0, dot(normal, lightDir));
            float quantizedDiffuse = quantize(NdotL, steps);
            
            // เพิ่ม threshold เล็กน้อยเพื่อลด noise
            quantizedDiffuse = smoothstep(0.0, 0.02, quantizedDiffuse - 0.0) * quantizedDiffuse;
            
            // Specular (แบบ toon - on/off)
            vec3 halfDir = normalize(lightDir + viewDir);
            float NdotH = max(0.0, dot(normal, halfDir));
            float specular = pow(NdotH, specularPower) * specularIntensity;
            // Quantize specular ให้เป็น on/off
            specular = step(0.5, specular);
            
            // Rim light (เส้นขอบสว่าง)
            float rim = 1.0 - max(0.0, dot(viewDir, normal));
            rim = pow(rim, 3.0);
            rim = step(0.6, rim) * rimLightIntensity;
            
            // Combine
            vec3 finalColor = color * (ambientColor + lightColor * quantizedDiffuse);
            finalColor += specular * lightColor;
            finalColor += rim * rimLightColor;
            
            gl_FragColor = vec4(finalColor, 1.0);
        }
    `
};
```

## Outline Pass

Outline เป็นส่วนสำคัญของ Cel Shading ทำให้ตัวการ์ตูนดูเหมือน hand-drawn:

```javascript
// วิธีที่ 1: OutlinePass ของ Three.js (ง่ายที่สุด)
import { OutlinePass } from 'three/examples/jsm/postprocessing/OutlinePass.js';

const outlinePass = new OutlinePass(
    new THREE.Vector2(window.innerWidth, window.innerHeight),
    scene,
    camera
);

// Configuration
outlinePass.edgeStrength = 3;       // ความแรงของเส้นขอบ
outlinePass.edgeGlow = 0;           // Glow ที่เส้นขอบ
outlinePass.edgeThickness = 1;      // ความหนาของเส้นขอบ
outlinePass.pulsePeriod = 0;        // 0 = ไม่กระพริบ
outlinePass.visibleEdgeColor.set('#000000'); // สีเส้นขอบที่มองเห็น
outlinePass.hiddenEdgeColor.set('#000000');  // สีเส้นขอบที่ถูกบัง

// กำหนดว่า object ใดจะมี outline
outlinePass.selectedObjects = [mesh1, mesh2];

composer.addPass(outlinePass);
```

## Back-Face Expansion Outline

วิธีที่นิยมใช้ในเกม: ขยาย back face ออกมาให้มองเห็น:

```javascript
// Back-face expansion outline
class BackFaceOutline {
    createOutlineMesh(originalMesh, outlineColor = 0x000000, outlineSize = 0.02) {
        // Clone geometry
        const geometry = originalMesh.geometry.clone();
        
        // สร้าง outline material
        const outlineMaterial = new THREE.ShaderMaterial({
            uniforms: {
                outlineColor: { value: new THREE.Color(outlineColor) },
                outlineSize: { value: outlineSize }
            },
            vertexShader: `
                uniform float outlineSize;
                
                void main() {
                    // ขยาย vertex ออกตาม normal direction
                    vec3 newPosition = position + normal * outlineSize;
                    gl_Position = projectionMatrix * modelViewMatrix * vec4(newPosition, 1.0);
                }
            `,
            fragmentShader: `
                uniform vec3 outlineColor;
                void main() {
                    gl_FragColor = vec4(outlineColor, 1.0);
                }
            `,
            side: THREE.BackSide  // สำคัญ! render เฉพาะ back faces
        });
        
        const outlineMesh = new THREE.Mesh(geometry, outlineMaterial);
        outlineMesh.position.copy(originalMesh.position);
        outlineMesh.rotation.copy(originalMesh.rotation);
        outlineMesh.scale.copy(originalMesh.scale);
        
        return outlineMesh;
    }
}
```

## ตัวอย่างที่ 1: Complete Anime Renderer

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Anime Renderer</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; font-family: Arial, sans-serif; }
        canvas { display: block; }
        #controls {
            position: fixed;
            top: 20px;
            right: 20px;
            background: rgba(255,240,250,0.95);
            color: #333;
            padding: 20px;
            border-radius: 12px;
            width: 260px;
            box-shadow: 0 4px 20px rgba(255,100,150,0.3);
        }
        h3 { color: #ff6699; margin-bottom: 15px; font-size: 16px; }
        .ctrl { margin-bottom: 10px; }
        .ctrl label { display: flex; justify-content: space-between; font-size: 12px; margin-bottom: 3px; }
        .ctrl input[type=range] { width: 100%; accent-color: #ff6699; }
        .section-title { color: #ff6699; font-weight: bold; font-size: 12px; margin: 10px 0 5px; }
        .color-btn {
            display: inline-block; width: 30px; height: 30px;
            border: 2px solid #ccc; border-radius: 50%; cursor: pointer;
            margin: 2px; transition: transform 0.2s;
        }
        .color-btn:hover { transform: scale(1.2); }
        .color-btn.active { border-color: #333; transform: scale(1.1); }
    </style>
</head>
<body>
    <div id="controls">
        <h3>🌸 Anime Renderer</h3>
        <div class="section-title">Toon Shading</div>
        <div class="ctrl">
            <label><span>Shade Steps</span><span id="stepsVal">3</span></label>
            <input type="range" id="steps" min="1" max="8" step="1" value="3">
        </div>
        <div class="ctrl">
            <label><span>Outline Width</span><span id="outlineVal">0.02</span></label>
            <input type="range" id="outlineWidth" min="0" max="0.05" step="0.002" value="0.02">
        </div>
        <div class="ctrl">
            <label><span>Rim Light</span><span id="rimVal">0.5</span></label>
            <input type="range" id="rimLight" min="0" max="1" step="0.05" value="0.5">
        </div>
        <div class="ctrl">
            <label><span>Specular Power</span><span id="specVal">32</span></label>
            <input type="range" id="specular" min="1" max="128" step="1" value="32">
        </div>
        <div class="section-title">Character Colors</div>
        <div id="colorPicker">
            <div class="color-btn active" style="background:#ff6644" data-color="0xff6644"></div>
            <div class="color-btn" style="background:#44aaff" data-color="0x44aaff"></div>
            <div class="color-btn" style="background:#44ff88" data-color="0x44ff88"></div>
            <div class="color-btn" style="background:#ff44aa" data-color="0xff44aa"></div>
            <div class="color-btn" style="background:#ffcc44" data-color="0xffcc44"></div>
            <div class="color-btn" style="background:#aa44ff" data-color="0xaa44ff"></div>
        </div>
        <div class="section-title">Style</div>
        <div class="ctrl">
            <label><span>Shadow Darkness</span><span id="shadowDarkVal">0.1</span></label>
            <input type="range" id="shadowDark" min="0" max="0.5" step="0.01" value="0.1">
        </div>
    </div>

    <script type="module">
        import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js';
        import { EffectComposer } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/EffectComposer.js';
        import { RenderPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/RenderPass.js';
        import { OutlinePass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/OutlinePass.js';
        import { OutputPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/OutputPass.js';
        import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/controls/OrbitControls.js';

        const W = window.innerWidth, H = window.innerHeight;
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(W, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        renderer.shadowMap.enabled = true;
        document.body.appendChild(renderer.domElement);

        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x87ceeb);

        const camera = new THREE.PerspectiveCamera(60, W/H, 0.1, 100);
        camera.position.set(0, 3, 8);
        camera.lookAt(0, 1.5, 0);

        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.target.set(0, 1.5, 0);

        // Lighting
        scene.add(new THREE.AmbientLight(0x334466, 1));
        const sunLight = new THREE.DirectionalLight(0xfff5cc, 2);
        sunLight.position.set(5, 10, 5);
        sunLight.castShadow = true;
        sunLight.shadow.mapSize.set(2048, 2048);
        scene.add(sunLight);

        // Toon Shader
        const toonUniforms = {
            color: { value: new THREE.Color(0xff6644) },
            lightDirection: { value: new THREE.Vector3(5, 10, 5).normalize() },
            lightColor: { value: new THREE.Color(0xfff5cc) },
            ambientColor: { value: new THREE.Color(0x334466) },
            steps: { value: 3.0 },
            specularPower: { value: 32.0 },
            specularIntensity: { value: 0.5 },
            rimLightColor: { value: new THREE.Color(0x88aaff) },
            rimLightIntensity: { value: 0.5 },
            shadowDarkness: { value: 0.1 }
        };

        function createToonMaterial(color) {
            return new THREE.ShaderMaterial({
                uniforms: {
                    ...JSON.parse(JSON.stringify(toonUniforms)),
                    color: { value: new THREE.Color(color) }
                },
                vertexShader: `
                    varying vec3 vNormal;
                    varying vec3 vViewPos;
                    void main() {
                        vNormal = normalize(normalMatrix * normal);
                        vec4 mvPos = modelViewMatrix * vec4(position, 1.0);
                        vViewPos = -mvPos.xyz;
                        gl_Position = projectionMatrix * mvPos;
                    }
                `,
                fragmentShader: `
                    uniform vec3 color;
                    uniform vec3 lightDirection;
                    uniform vec3 lightColor;
                    uniform vec3 ambientColor;
                    uniform float steps;
                    uniform float specularPower;
                    uniform float specularIntensity;
                    uniform vec3 rimLightColor;
                    uniform float rimLightIntensity;
                    uniform float shadowDarkness;
                    
                    varying vec3 vNormal;
                    varying vec3 vViewPos;
                    
                    float quantize(float val, float n) {
                        return floor(val * n) / n;
                    }
                    
                    void main() {
                        vec3 normal = normalize(vNormal);
                        vec3 viewDir = normalize(vViewPos);
                        vec3 lightDir = normalize(lightDirection);
                        
                        float NdotL = dot(normal, lightDir) * 0.5 + 0.5;
                        float toonDiff = quantize(NdotL, steps);
                        toonDiff = mix(shadowDarkness, 1.0, toonDiff);
                        
                        vec3 halfDir = normalize(lightDir + viewDir);
                        float spec = pow(max(0.0, dot(normal, halfDir)), specularPower);
                        float toonSpec = step(0.5, spec) * specularIntensity;
                        
                        float rim = 1.0 - max(0.0, dot(viewDir, normal));
                        float toonRim = step(0.65, pow(rim, 3.0)) * rimLightIntensity;
                        
                        vec3 final = color * (ambientColor * 0.3 + lightColor * toonDiff);
                        final += toonSpec * lightColor;
                        final += toonRim * rimLightColor;
                        
                        gl_FragColor = vec4(final, 1.0);
                    }
                `
            });
        }

        // สร้างตัวละคร anime แบบ stylized
        const characterGroup = new THREE.Group();
        const outlineObjects = [];

        // Body parts with materials
        const bodyMat = createToonMaterial(0xff6644);
        const skinMat = createToonMaterial(0xffe0c0);
        const hairMat = createToonMaterial(0x442211);
        const clothesMat = createToonMaterial(0x4488cc);
        const eyesMat = new THREE.MeshStandardMaterial({ color: 0x224488, roughness: 0.3 });

        function addWithOutline(mesh, scene) {
            scene.add(mesh);
            outlineObjects.push(mesh);
            
            // Back-face outline
            const outlineGeo = mesh.geometry.clone();
            const outlineMat = new THREE.ShaderMaterial({
                uniforms: {
                    outlineColor: { value: new THREE.Color(0x000000) },
                    outlineSize: { value: 0.02 }
                },
                vertexShader: `
                    uniform float outlineSize;
                    void main() {
                        vec3 newPos = position + normal * outlineSize;
                        gl_Position = projectionMatrix * modelViewMatrix * vec4(newPos, 1.0);
                    }
                `,
                fragmentShader: `
                    uniform vec3 outlineColor;
                    void main() { gl_FragColor = vec4(outlineColor, 1.0); }
                `,
                side: THREE.BackSide
            });
            const outlineMesh = new THREE.Mesh(outlineGeo, outlineMat);
            outlineMesh.position.copy(mesh.position);
            outlineMesh.rotation.copy(mesh.rotation);
            outlineMesh.scale.copy(mesh.scale);
            scene.add(outlineMesh);
            
            return outlineMesh;
        }

        // Body
        const body = new THREE.Mesh(new THREE.BoxGeometry(1.2, 1.5, 0.8), clothesMat);
        body.position.y = 1.75;
        body.castShadow = true;
        addWithOutline(body, characterGroup);

        // Head
        const head = new THREE.Mesh(new THREE.BoxGeometry(1.0, 1.0, 0.9), skinMat);
        head.position.y = 3.0;
        head.castShadow = true;
        addWithOutline(head, characterGroup);

        // Hair
        const hair = new THREE.Mesh(new THREE.BoxGeometry(1.1, 0.6, 1.0), hairMat);
        hair.position.set(0, 3.4, 0);
        hair.castShadow = true;
        addWithOutline(hair, characterGroup);

        // Eyes
        const leftEye = new THREE.Mesh(new THREE.BoxGeometry(0.2, 0.25, 0.1), eyesMat);
        leftEye.position.set(-0.22, 3.0, 0.46);
        characterGroup.add(leftEye);
        
        const rightEye = leftEye.clone();
        rightEye.position.x = 0.22;
        characterGroup.add(rightEye);

        // Arms
        [-0.85, 0.85].forEach(x => {
            const arm = new THREE.Mesh(new THREE.BoxGeometry(0.35, 1.2, 0.35), clothesMat);
            arm.position.set(x, 1.75, 0);
            arm.castShadow = true;
            addWithOutline(arm, characterGroup);
        });

        // Legs
        [-0.35, 0.35].forEach(x => {
            const leg = new THREE.Mesh(new THREE.BoxGeometry(0.4, 1.2, 0.4), hairMat);
            leg.position.set(x, 0.6, 0);
            leg.castShadow = true;
            addWithOutline(leg, characterGroup);
        });

        scene.add(characterGroup);

        // Environment
        const floorMesh = new THREE.Mesh(
            new THREE.PlaneGeometry(20, 20),
            new THREE.MeshToonMaterial({ color: 0x88cc66 })
        );
        floorMesh.rotation.x = -Math.PI / 2;
        floorMesh.receiveShadow = true;
        scene.add(floorMesh);

        // Clouds
        for (let i = 0; i < 5; i++) {
            const cloud = new THREE.Group();
            [0, 0.5, -0.5, 0.8, -0.8].forEach((ox, j) => {
                const c = new THREE.Mesh(
                    new THREE.SphereGeometry(0.5 + Math.random() * 0.3, 8, 6),
                    new THREE.MeshToonMaterial({ color: 0xffffff })
                );
                c.position.set(ox, j * 0.1, 0);
                cloud.add(c);
            });
            cloud.position.set((i - 2) * 6, 7, -10 + i * 2);
            scene.add(cloud);
        }

        // Collect all outline meshes for outlinePass
        const allToonMeshes = [];
        characterGroup.traverse(obj => {
            if (obj.isMesh) allToonMeshes.push(obj);
        });

        // Post-processing
        const composer = new EffectComposer(renderer);
        composer.addPass(new RenderPass(scene, camera));
        
        const outlinePass = new OutlinePass(new THREE.Vector2(W, H), scene, camera);
        outlinePass.edgeStrength = 3;
        outlinePass.edgeThickness = 1;
        outlinePass.edgeGlow = 0;
        outlinePass.visibleEdgeColor.set('#000000');
        outlinePass.hiddenEdgeColor.set('#000000');
        outlinePass.selectedObjects = outlineObjects;
        composer.addPass(outlinePass);
        
        composer.addPass(new OutputPass());

        // Collect all toon materials for update
        const allToonMaterials = [];
        characterGroup.traverse(obj => {
            if (obj.isMesh && obj.material.uniforms) allToonMaterials.push(obj.material);
        });

        // UI Controls
        document.getElementById('steps').addEventListener('input', (e) => {
            const v = parseInt(e.target.value);
            allToonMaterials.forEach(m => { if (m.uniforms.steps) m.uniforms.steps.value = v; });
            document.getElementById('stepsVal').textContent = v;
        });
        
        document.getElementById('outlineWidth').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            document.getElementById('outlineVal').textContent = v.toFixed(3);
            // Update back-face outline size
            characterGroup.traverse(obj => {
                if (obj.isMesh && obj.material.uniforms && obj.material.uniforms.outlineSize) {
                    obj.material.uniforms.outlineSize.value = v;
                }
            });
        });
        
        document.getElementById('rimLight').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            allToonMaterials.forEach(m => { if (m.uniforms.rimLightIntensity) m.uniforms.rimLightIntensity.value = v; });
            document.getElementById('rimVal').textContent = v.toFixed(2);
        });
        
        document.getElementById('specular').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            allToonMaterials.forEach(m => { if (m.uniforms.specularPower) m.uniforms.specularPower.value = v; });
            document.getElementById('specVal').textContent = v;
        });

        document.getElementById('shadowDark').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            allToonMaterials.forEach(m => { if (m.uniforms.shadowDarkness) m.uniforms.shadowDarkness.value = v; });
            document.getElementById('shadowDarkVal').textContent = v.toFixed(2);
        });

        document.querySelectorAll('.color-btn').forEach(btn => {
            btn.addEventListener('click', () => {
                document.querySelectorAll('.color-btn').forEach(b => b.classList.remove('active'));
                btn.classList.add('active');
                const color = new THREE.Color(parseInt(btn.dataset.color));
                // Change body and clothes color
                [body.material, ...characterGroup.children
                    .filter(c => c.material && c.material.uniforms && c.material.uniforms.color)]
                    .forEach(m => { if (m && m.uniforms && m.uniforms.color) m.uniforms.color.value.copy(color); });
            });
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
            
            characterGroup.rotation.y = Math.sin(t * 0.3) * 0.5;
            
            controls.update();
            composer.render();
        }
        animate();
    </script>
</body>
</html>
```

## Hatching Shader

Hatching สร้างลักษณะการแรเงาแบบ pen-and-ink:

```glsl
// Hatching Fragment Shader
uniform sampler2D hatchTex0; // light hatching
uniform sampler2D hatchTex1; // medium hatching
uniform sampler2D hatchTex2; // dark hatching
uniform vec3 lightDir;

varying vec3 vNormal;
varying vec2 vUv;

void main() {
    float intensity = dot(normalize(vNormal), normalize(lightDir));
    intensity = clamp(intensity * 0.5 + 0.5, 0.0, 1.0);
    
    vec3 hatch;
    
    if (intensity > 0.75) {
        // Very bright - no hatching
        hatch = vec3(1.0);
    } else if (intensity > 0.5) {
        // Light hatching
        vec3 h0 = texture2D(hatchTex0, vUv * 6.0).rgb;
        hatch = mix(h0, vec3(1.0), (intensity - 0.5) / 0.25);
    } else if (intensity > 0.25) {
        // Medium hatching
        vec3 h0 = texture2D(hatchTex0, vUv * 6.0).rgb;
        vec3 h1 = texture2D(hatchTex1, vUv * 6.0).rgb;
        hatch = mix(h1, h0, (intensity - 0.25) / 0.25);
    } else {
        // Dark hatching
        vec3 h1 = texture2D(hatchTex1, vUv * 6.0).rgb;
        vec3 h2 = texture2D(hatchTex2, vUv * 6.0).rgb;
        hatch = mix(h2, h1, intensity / 0.25);
    }
    
    gl_FragColor = vec4(hatch, 1.0);
}
```

## Comic Book Post-Process Stack

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Comic Book Effect</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; font-family: Arial, sans-serif; }
        canvas { display: block; }
        #panel {
            position: fixed;
            top: 20px;
            left: 20px;
            background: rgba(255,255,240,0.95);
            color: #222;
            padding: 20px;
            border-radius: 8px;
            width: 240px;
            border: 3px solid #000;
            box-shadow: 5px 5px 0 #000;
            font-family: 'Comic Sans MS', cursive;
        }
        h3 { color: #cc0000; margin-bottom: 12px; text-shadow: 2px 2px 0 #000; }
        .ctrl { margin-bottom: 8px; font-size: 12px; }
        .ctrl label { display: flex; justify-content: space-between; margin-bottom: 2px; }
        .toggle-btn {
            display: block; width: 100%; padding: 8px;
            background: #ffee00; color: #000; border: 2px solid #000;
            cursor: pointer; font-family: inherit; font-weight: bold;
            font-size: 13px; margin-top: 5px;
            box-shadow: 3px 3px 0 #000;
        }
        .toggle-btn:active { transform: translate(2px, 2px); box-shadow: 1px 1px 0 #000; }
    </style>
</head>
<body>
    <div id="panel">
        <h3>💥 COMIC BOOK!</h3>
        <div class="ctrl">
            <label><span>Halftone Size</span><span id="htVal">3.0</span></label>
            <input type="range" id="halftoneSize" min="1" max="10" step="0.5" value="3">
        </div>
        <div class="ctrl">
            <label><span>Edge Darkness</span><span id="edgeVal">0.5</span></label>
            <input type="range" id="edgeDark" min="0" max="1" step="0.05" value="0.5">
        </div>
        <div class="ctrl">
            <label><span>Color Quantize</span><span id="cqVal">4</span></label>
            <input type="range" id="colorQuant" min="2" max="16" step="1" value="4">
        </div>
        <button class="toggle-btn" id="toggleHalftone">HALFTONE: ON</button>
        <button class="toggle-btn" id="toggleEdge">EDGE LINES: ON</button>
    </div>

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
        document.body.appendChild(renderer.domElement);

        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0xfff8e7);

        const camera = new THREE.PerspectiveCamera(60, W/H, 0.1, 100);
        camera.position.set(0, 4, 10);
        camera.lookAt(0, 1, 0);
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;

        scene.add(new THREE.AmbientLight(0xffffff, 1));
        const dirLight = new THREE.DirectionalLight(0xffffff, 2);
        dirLight.position.set(5, 10, 5);
        dirLight.castShadow = true;
        scene.add(dirLight);

        // Simple scene
        const ground = new THREE.Mesh(
            new THREE.PlaneGeometry(20, 20),
            new THREE.MeshToonMaterial({ color: 0x88cc44 })
        );
        ground.rotation.x = -Math.PI / 2;
        scene.add(ground);

        const shapes = [
            { geo: new THREE.BoxGeometry(2, 2, 2), pos: [0, 1, 0], color: 0xff4444 },
            { geo: new THREE.SphereGeometry(1, 32, 32), pos: [-4, 1, -2], color: 0x4444ff },
            { geo: new THREE.ConeGeometry(1, 2, 16), pos: [4, 1, -2], color: 0x44ff44 },
            { geo: new THREE.TorusGeometry(1, 0.3, 16, 32), pos: [0, 2, -5], color: 0xff8800 },
        ];
        
        shapes.forEach(s => {
            const mesh = new THREE.Mesh(s.geo, new THREE.MeshToonMaterial({ color: s.color }));
            mesh.position.set(...s.pos);
            mesh.castShadow = true;
            scene.add(mesh);
        });

        // Comic Book Shader
        const ComicShader = {
            uniforms: {
                tDiffuse: { value: null },
                resolution: { value: new THREE.Vector2(W, H) },
                halftoneSize: { value: 3.0 },
                edgeDarkness: { value: 0.5 },
                colorQuantize: { value: 4.0 },
                halftoneEnabled: { value: 1 },
                edgeEnabled: { value: 1 }
            },
            vertexShader: `
                varying vec2 vUv;
                void main() { vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }
            `,
            fragmentShader: `
                uniform sampler2D tDiffuse;
                uniform vec2 resolution;
                uniform float halftoneSize;
                uniform float edgeDarkness;
                uniform float colorQuantize;
                uniform int halftoneEnabled;
                uniform int edgeEnabled;
                
                varying vec2 vUv;
                
                // Quantize color
                vec3 quantizeColor(vec3 color, float steps) {
                    return floor(color * steps) / steps;
                }
                
                // Simple halftone pattern
                float halftone(vec2 uv, float size) {
                    vec2 pos = mod(uv * resolution / size, 1.0) - 0.5;
                    return length(pos);
                }
                
                // Sobel edge detection
                float edgeDetect(sampler2D tex, vec2 uv, vec2 texelSize) {
                    float tl = length(texture2D(tex, uv + texelSize * vec2(-1, -1)).rgb);
                    float t  = length(texture2D(tex, uv + texelSize * vec2( 0, -1)).rgb);
                    float tr = length(texture2D(tex, uv + texelSize * vec2( 1, -1)).rgb);
                    float l  = length(texture2D(tex, uv + texelSize * vec2(-1,  0)).rgb);
                    float r  = length(texture2D(tex, uv + texelSize * vec2( 1,  0)).rgb);
                    float bl = length(texture2D(tex, uv + texelSize * vec2(-1,  1)).rgb);
                    float b  = length(texture2D(tex, uv + texelSize * vec2( 0,  1)).rgb);
                    float br = length(texture2D(tex, uv + texelSize * vec2( 1,  1)).rgb);
                    
                    float Gx = -tl - 2.0*l - bl + tr + 2.0*r + br;
                    float Gy = -tl - 2.0*t - tr + bl + 2.0*b + br;
                    return sqrt(Gx*Gx + Gy*Gy);
                }
                
                void main() {
                    vec2 texelSize = 1.0 / resolution;
                    vec4 color = texture2D(tDiffuse, vUv);
                    
                    // Quantize colors
                    color.rgb = quantizeColor(color.rgb, colorQuantize);
                    
                    // Halftone pattern
                    if (halftoneEnabled == 1) {
                        float pattern = halftone(vUv, halftoneSize);
                        float luma = dot(color.rgb, vec3(0.299, 0.587, 0.114));
                        float threshold = 0.3 + luma * 0.4;
                        float dot = step(threshold, pattern);
                        color.rgb = mix(color.rgb * 0.7, color.rgb * 1.1, dot);
                    }
                    
                    // Edge lines
                    if (edgeEnabled == 1) {
                        float edge = edgeDetect(tDiffuse, vUv, texelSize * 1.5);
                        edge = step(0.15, edge);
                        color.rgb = mix(color.rgb, vec3(0.0), edge * edgeDarkness);
                    }
                    
                    // Slight paper texture
                    float paper = fract(sin(dot(vUv, vec2(127.1, 311.7))) * 43758.5) * 0.03;
                    color.rgb += paper;
                    
                    gl_FragColor = color;
                }
            `
        };

        const composer = new EffectComposer(renderer);
        composer.addPass(new RenderPass(scene, camera));
        const comicPass = new ShaderPass(ComicShader);
        composer.addPass(comicPass);
        composer.addPass(new OutputPass());

        // UI
        document.getElementById('halftoneSize').addEventListener('input', (e) => {
            comicPass.uniforms.halftoneSize.value = parseFloat(e.target.value);
            document.getElementById('htVal').textContent = e.target.value;
        });
        document.getElementById('edgeDark').addEventListener('input', (e) => {
            comicPass.uniforms.edgeDarkness.value = parseFloat(e.target.value);
            document.getElementById('edgeVal').textContent = e.target.value;
        });
        document.getElementById('colorQuant').addEventListener('input', (e) => {
            comicPass.uniforms.colorQuantize.value = parseFloat(e.target.value);
            document.getElementById('cqVal').textContent = e.target.value;
        });
        
        let halftoneOn = true, edgeOn = true;
        document.getElementById('toggleHalftone').addEventListener('click', function() {
            halftoneOn = !halftoneOn;
            comicPass.uniforms.halftoneEnabled.value = halftoneOn ? 1 : 0;
            this.textContent = `HALFTONE: ${halftoneOn ? 'ON' : 'OFF'}`;
        });
        document.getElementById('toggleEdge').addEventListener('click', function() {
            edgeOn = !edgeOn;
            comicPass.uniforms.edgeEnabled.value = edgeOn ? 1 : 0;
            this.textContent = `EDGE LINES: ${edgeOn ? 'ON' : 'OFF'}`;
        });

        window.addEventListener('resize', () => {
            const w = window.innerWidth, h = window.innerHeight;
            camera.aspect = w / h; camera.updateProjectionMatrix();
            renderer.setSize(w, h); composer.setSize(w, h);
            comicPass.uniforms.resolution.value.set(w, h);
        });

        const clock = new THREE.Clock();
        function animate() {
            requestAnimationFrame(animate);
            const t = clock.getElapsedTime();
            shapes.forEach((s, i) => {
                if (scene.children[i + 2]) {
                    scene.children[i + 2].rotation.y = t * 0.3 + i;
                }
            });
            controls.update();
            composer.render();
        }
        animate();
    </script>
</body>
</html>
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Dragon Ball Style
สร้าง character ที่ดูเหมือน Dragon Ball:
- Strong rim light สีส้ม/เหลือง
- 2-step toon shading
- Thick black outlines

### แบบฝึกหัดที่ 2: Watercolor Effect
สร้าง watercolor style:
- Soft gradient ระหว่างสี
- Paper texture ใน background
- Color bleed ที่ขอบ

### แบบฝึกหัดที่ 3: Ink Drawing
สร้าง black-and-white ink drawing:
- ใช้ edge detection สำหรับ lines
- Hatching สำหรับ shadows
- Ink splatter effects

### แบบฝึกหัดที่ 4: Anime Background (Paint Style)
สร้าง background ในสไตล์ Studio Ghibli:
- Soft color gradients
- Simplified shapes
- Atmospheric depth

---

**สรุปบทที่ 54:**
- MeshToonMaterial เป็นทางเลือกที่ง่ายสำหรับ basic toon shading
- Custom shader ให้การควบคุมมากกว่า ทั้ง diffuse steps, specular, rim light
- Back-face expansion เป็น technique outline ที่ทำงานได้กับทุก mesh
- OutlinePass ทำงานแบบ post-process มีประสิทธิภาพสูง
- Comic book effect ผสม halftone + edge detection + color quantization

**บทต่อไป:** Outline, Glow & Selection Effects
