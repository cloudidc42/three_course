# Part 52: Screen Space Reflections (SSR)

## แนวคิด Screen Space Reflections

Screen Space Reflections (SSR) เป็นเทคนิคที่คำนวณการสะท้อนแสงโดยใช้ข้อมูลที่มีอยู่แล้วใน screen space (depth buffer, color buffer, normal buffer) เทคนิคนี้ให้การสะท้อนที่สมจริงกว่า environment map ในหลายกรณี แต่มีข้อจำกัดเพราะสามารถสะท้อนเฉพาะสิ่งที่ปรากฏใน screen เท่านั้น

### ข้อดีและข้อเสียของ SSR

**ข้อดี:**
- สะท้อนวัตถุในฉากจริง (ไม่ใช่แค่ environment map)
- Dynamic - อัพเดทตาม frame
- สะท้อนได้เหมือนจริงสำหรับพื้นผิวใกล้ๆ

**ข้อเสีย:**
- สะท้อนได้เฉพาะ visible pixels ใน screen
- Edges ของ screen ไม่มีข้อมูล (fallback ไปยัง env map)
- มีค่าใช้จ่ายสูงด้านประสิทธิภาพ
- Artifacts ที่ขอบ geometry

## หลักการทำงานของ SSR

```
1. Render scene → color buffer + depth buffer + normal buffer
2. สำหรับแต่ละ pixel ที่มี reflective surface:
   a. คำนวณ reflection direction ในจาก camera ray + surface normal
   b. March (ray march) ไปตาม reflected direction ใน screen space
   c. ตรวจสอบว่า ray ชนกับ geometry ที่อยู่ใน depth buffer หรือไม่
   d. ถ้าชน = ใช้สีจาก color buffer ณ จุดนั้น
   e. ถ้าไม่ชน = fallback ไปยัง environment map
3. ทำ blurring ตามค่า roughness
4. Composite ผลลัพธ์
```

## SSRPass ใน Three.js

```javascript
// การใช้ SSRPass
import { SSRPass } from 'three/examples/jsm/postprocessing/SSRPass.js';
import { ReflectorForSSRPass } from 'three/examples/jsm/objects/ReflectorForSSRPass.js';

// สร้าง reflective floor ด้วย ReflectorForSSRPass
const planeGeometry = new THREE.PlaneGeometry(50, 50);
const groundReflector = new ReflectorForSSRPass(planeGeometry, {
    clipBias: 0.0003,
    textureWidth: window.innerWidth,
    textureHeight: window.innerHeight,
    color: 0x888888,
    useDepthTexture: true
});
groundReflector.material.depthWrite = false;
groundReflector.rotation.x = -Math.PI / 2;
scene.add(groundReflector);

// SSRPass configuration
const ssrPass = new SSRPass({
    renderer,
    scene,
    camera,
    width: window.innerWidth,
    height: window.innerHeight,
    groundReflector: groundReflector,
    selects: null  // null = สะท้อนทุกอย่าง, หรือระบุ meshes
});

ssrPass.thickness = 0.018;    // ความหนาของ ray (ควบคุม false positive)
ssrPass.infiniteThick = false; // ใช้ infinite thickness หรือไม่
ssrPass.maxDistance = 0.1;    // ระยะสูงสุดในการ trace
ssrPass.opacity = 0.5;        // ความโปร่งใสของการสะท้อน
ssrPass.blur = true;          // blur การสะท้อน

composer.addPass(ssrPass);
```

## Custom SSR Shader

การสร้าง SSR shader เอง:

```glsl
// SSR Fragment Shader
uniform sampler2D tDiffuse;      // scene color
uniform sampler2D tDepth;        // scene depth
uniform sampler2D tNormal;       // scene normals
uniform sampler2D tEnvironment;  // fallback env map
uniform mat4 projMatrix;
uniform mat4 projMatrixInverse;
uniform mat4 viewMatrix;
uniform mat4 viewMatrixInverse;
uniform vec2 resolution;
uniform float maxSteps;          // จำนวน ray march steps
uniform float stepSize;          // ขนาด step
uniform float thickness;         // ความหนาของ geometry
uniform float maxDistance;       // ระยะสูงสุด
uniform float fresnel;           // Fresnel effect

varying vec2 vUv;

// แปลง depth เป็น view space position
vec3 depthToViewSpace(vec2 uv, float depth) {
    vec4 ndc = vec4(uv * 2.0 - 1.0, depth * 2.0 - 1.0, 1.0);
    vec4 viewPos = projMatrixInverse * ndc;
    return viewPos.xyz / viewPos.w;
}

// แปลง view space เป็น UV + depth
vec3 viewSpaceToScreen(vec3 viewPos) {
    vec4 clipPos = projMatrix * vec4(viewPos, 1.0);
    vec3 ndc = clipPos.xyz / clipPos.w;
    return ndc * 0.5 + 0.5; // xyz = uvw (w = depth)
}

// Binary search สำหรับหาจุดตัดที่ precise
vec2 binarySearch(vec3 startPos, vec3 dir, float startDepth) {
    vec2 bestUV = vec2(-1.0);
    
    for (int i = 0; i < 8; i++) {
        vec3 midPos = startPos + dir * 0.5;
        vec3 screenPos = viewSpaceToScreen(midPos);
        
        if (screenPos.x < 0.0 || screenPos.x > 1.0 ||
            screenPos.y < 0.0 || screenPos.y > 1.0) break;
        
        float sceneDepth = texture2D(tDepth, screenPos.xy).r;
        
        if (screenPos.z > sceneDepth + thickness) {
            startPos = midPos;
        } else {
            dir *= 0.5;
        }
        
        bestUV = screenPos.xy;
    }
    
    return bestUV;
}

void main() {
    vec4 diffuse = texture2D(tDiffuse, vUv);
    float depth = texture2D(tDepth, vUv).r;
    
    // ข้าม sky/background
    if (depth >= 0.9999) {
        gl_FragColor = diffuse;
        return;
    }
    
    vec3 viewPos = depthToViewSpace(vUv, depth);
    vec3 viewNorm = normalize(texture2D(tNormal, vUv).xyz * 2.0 - 1.0);
    
    // คำนวณ reflection direction
    vec3 viewDir = normalize(viewPos);
    vec3 reflDir = reflect(viewDir, viewNorm);
    
    // Ray march
    vec3 currentPos = viewPos + reflDir * stepSize;
    vec4 reflectedColor = vec4(0.0);
    bool hit = false;
    
    for (int i = 0; i < 64; i++) {
        if (float(i) >= maxSteps) break;
        
        vec3 screenPos = viewSpaceToScreen(currentPos);
        
        // ออกนอก screen
        if (screenPos.x < 0.0 || screenPos.x > 1.0 ||
            screenPos.y < 0.0 || screenPos.y > 1.0 ||
            screenPos.z < 0.0 || screenPos.z > 1.0) break;
        
        float sceneDepth = texture2D(tDepth, screenPos.xy).r;
        
        // ตรวจสอบการชน
        if (screenPos.z > sceneDepth && screenPos.z < sceneDepth + thickness) {
            // Hit! ทำ binary search เพื่อความ precise
            vec2 hitUV = binarySearch(currentPos - reflDir * stepSize, reflDir * stepSize, sceneDepth);
            
            if (hitUV.x >= 0.0) {
                reflectedColor = texture2D(tDiffuse, hitUV);
                
                // Fade out ที่ขอบ screen
                vec2 fade = smoothstep(0.0, 0.1, hitUV) * (1.0 - smoothstep(0.9, 1.0, hitUV));
                reflectedColor.a *= fade.x * fade.y;
            }
            
            hit = true;
            break;
        }
        
        currentPos += reflDir * stepSize * (1.0 + float(i) * 0.1);
    }
    
    // Fresnel - มุมมองชัน = สะท้อนมาก
    float fresnelFactor = pow(1.0 - max(0.0, dot(-viewDir, viewNorm)), 5.0);
    float reflectivity = mix(0.02, 1.0, fresnelFactor * fresnel);
    
    // Composite
    if (hit && reflectedColor.a > 0.0) {
        gl_FragColor = mix(diffuse, reflectedColor, reflectivity * reflectedColor.a);
    } else {
        gl_FragColor = diffuse;
    }
}
```

## ตัวอย่างที่ 1: Wet Floor SSR Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Wet Floor SSR Demo</title>
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
        #controls h3 { color: #4fc3f7; margin-bottom: 15px; }
        .ctrl { margin-bottom: 12px; }
        .ctrl label { display: flex; justify-content: space-between; font-size: 12px; margin-bottom: 4px; }
        .ctrl input[type=range] { width: 100%; }
        .btn { 
            display: block; width: 100%; padding: 8px; 
            background: #1a5276; color: white; border: none;
            border-radius: 5px; cursor: pointer; margin-bottom: 5px; font-size: 13px;
        }
        .btn:hover { background: #2471a3; }
        .badge {
            display: inline-block; padding: 2px 8px; border-radius: 10px;
            font-size: 11px; font-weight: bold;
        }
    </style>
</head>
<body>
    <div id="controls">
        <h3>💧 Wet Floor SSR</h3>
        <div class="ctrl">
            <label><span>Reflection Opacity</span><span id="opacityVal">0.5</span></label>
            <input type="range" id="opacity" min="0" max="1" step="0.01" value="0.5">
        </div>
        <div class="ctrl">
            <label><span>Thickness</span><span id="thickVal">0.018</span></label>
            <input type="range" id="thick" min="0.001" max="0.1" step="0.001" value="0.018">
        </div>
        <div class="ctrl">
            <label><span>Max Distance</span><span id="distVal">0.10</span></label>
            <input type="range" id="dist" min="0.01" max="0.5" step="0.01" value="0.1">
        </div>
        <div class="ctrl">
            <label><span>Wetness</span><span id="wetVal">1.0</span></label>
            <input type="range" id="wetness" min="0" max="1" step="0.01" value="1">
        </div>
        <button class="btn" id="toggleSSR">Toggle SSR On/Off</button>
        <button class="btn" id="toggleRain">Toggle Rain</button>
        <div style="margin-top:10px;font-size:11px;color:#aaa">
            * SSRPass ต้องการ Three.js examples<br>
            * ตัวอย่างนี้ใช้ ReflectorPass แทน
        </div>
    </div>

    <script type="module">
        import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js';
        import { EffectComposer } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/EffectComposer.js';
        import { RenderPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/RenderPass.js';
        import { OutputPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/OutputPass.js';
        import { Reflector } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/objects/Reflector.js';
        import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/controls/OrbitControls.js';
        import { UnrealBloomPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/UnrealBloomPass.js';

        const W = window.innerWidth, H = window.innerHeight;
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(W, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        renderer.shadowMap.enabled = true;
        renderer.shadowMap.type = THREE.PCFSoftShadowMap;
        document.body.appendChild(renderer.domElement);

        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x0a0a15);
        scene.fog = new THREE.FogExp2(0x0a0a15, 0.02);

        const camera = new THREE.PerspectiveCamera(60, W/H, 0.1, 200);
        camera.position.set(0, 5, 15);
        camera.lookAt(0, 0, 0);

        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.dampingFactor = 0.05;
        controls.maxPolarAngle = Math.PI / 2.2;

        // Lighting - night scene
        scene.add(new THREE.AmbientLight(0x112233, 0.5));
        
        const neonColors = [0xff0088, 0x0088ff, 0x00ff88, 0xff8800];
        const neonLights = [];
        
        neonColors.forEach((color, i) => {
            const light = new THREE.PointLight(color, 3, 15);
            light.position.set(Math.sin(i / 4 * Math.PI * 2) * 8, 4, Math.cos(i / 4 * Math.PI * 2) * 8);
            light.castShadow = true;
            light.shadow.mapSize.set(512, 512);
            scene.add(light);
            neonLights.push(light);
        });

        // Reflective wet floor
        const floorGeometry = new THREE.PlaneGeometry(50, 50);
        
        // Reflector (กระจกสะท้อน)
        const reflector = new Reflector(floorGeometry, {
            clipBias: 0.003,
            textureWidth: W,
            textureHeight: H,
            color: new THREE.Color(0x445566)
        });
        reflector.rotation.x = -Math.PI / 2;
        reflector.position.y = -0.01;
        scene.add(reflector);

        // Wet overlay (ทำให้ดูเปียก)
        const wetMaterial = new THREE.MeshStandardMaterial({
            color: 0x222233,
            roughness: 0.0,
            metalness: 0.1,
            transparent: true,
            opacity: 0.5,
            side: THREE.FrontSide
        });
        const wetFloor = new THREE.Mesh(floorGeometry, wetMaterial);
        wetFloor.rotation.x = -Math.PI / 2;
        wetFloor.position.y = 0.0;
        scene.add(wetFloor);

        // Ripple effect on wet floor
        const rippleCanvas = document.createElement('canvas');
        rippleCanvas.width = rippleCanvas.height = 512;
        const rippleCtx = rippleCanvas.getContext('2d');
        const rippleTexture = new THREE.CanvasTexture(rippleCanvas);
        wetMaterial.normalMap = rippleTexture;

        function updateRipples(time) {
            rippleCtx.fillStyle = '#8080ff'; // neutral normal
            rippleCtx.fillRect(0, 0, 512, 512);
            
            // วาด ripples ต่างๆ
            for (let i = 0; i < 8; i++) {
                const x = 100 + i * 50;
                const y = 100 + (i % 3) * 80;
                const r = ((time * 30 + i * 100) % 200);
                const alpha = 1.0 - r / 200;
                
                const gradient = rippleCtx.createRadialGradient(x, y, 0, x, y, r);
                gradient.addColorStop(0, `rgba(128, 128, 255, 0)`);
                gradient.addColorStop(0.8, `rgba(100, 100, 220, ${alpha * 0.3})`);
                gradient.addColorStop(1, `rgba(128, 128, 255, 0)`);
                
                rippleCtx.fillStyle = gradient;
                rippleCtx.fillRect(0, 0, 512, 512);
            }
            
            rippleTexture.needsUpdate = true;
        }

        // Buildings และ props สำหรับการสะท้อน
        function createNeonBuilding(x, z, h, color) {
            const group = new THREE.Group();
            
            const bodyMat = new THREE.MeshStandardMaterial({ color: 0x111122, roughness: 0.8 });
            const body = new THREE.Mesh(new THREE.BoxGeometry(3, h, 3), bodyMat);
            body.position.y = h / 2;
            body.castShadow = true;
            group.add(body);
            
            // Neon sign
            const neonMat = new THREE.MeshStandardMaterial({
                color: color,
                emissive: color,
                emissiveIntensity: 2
            });
            const neon = new THREE.Mesh(new THREE.BoxGeometry(2.5, 0.3, 0.1), neonMat);
            neon.position.set(0, h * 0.7, 1.55);
            group.add(neon);
            
            // นีออนแนวตั้ง
            const neonV = new THREE.Mesh(new THREE.BoxGeometry(0.1, h * 0.3, 0.1), neonMat);
            neonV.position.set(1.2, h * 0.85, 1.55);
            group.add(neonV);
            
            // Windows
            const winMat = new THREE.MeshStandardMaterial({
                color: 0xffffcc,
                emissive: 0xffff88,
                emissiveIntensity: Math.random() * 0.5
            });
            for (let wy = 0; wy < h - 1; wy += 1.5) {
                for (let wx = -0.8; wx <= 0.8; wx += 0.8) {
                    if (Math.random() > 0.3) {
                        const win = new THREE.Mesh(new THREE.BoxGeometry(0.4, 0.5, 0.05), winMat.clone());
                        win.material.emissiveIntensity = Math.random() > 0.3 ? 0.3 : 0;
                        win.position.set(wx, 0.5 + wy, 1.53);
                        group.add(win);
                    }
                }
            }
            
            group.position.set(x, 0, z);
            scene.add(group);
            return group;
        }

        createNeonBuilding(-8, -3, 8, 0xff0088);
        createNeonBuilding(-5, -2, 12, 0x0088ff);
        createNeonBuilding(5, -4, 9, 0x00ff88);
        createNeonBuilding(8, -2, 14, 0xff8800);
        createNeonBuilding(-10, 2, 6, 0xff00ff);
        createNeonBuilding(10, 3, 10, 0x00ffff);

        // Puddles (แอ่งน้ำ)
        function createPuddle(x, z, r) {
            const puddleGeo = new THREE.CircleGeometry(r, 32);
            const puddleReflector = new Reflector(puddleGeo, {
                clipBias: 0.003,
                textureWidth: W / 2,
                textureHeight: H / 2,
                color: new THREE.Color(0x334455)
            });
            puddleReflector.rotation.x = -Math.PI / 2;
            puddleReflector.position.set(x, 0.005, z);
            scene.add(puddleReflector);
        }
        
        createPuddle(-3, 5, 2);
        createPuddle(4, 3, 1.5);
        createPuddle(-6, 6, 1);
        createPuddle(2, 8, 2.5);

        // ฝน (particles)
        let rainEnabled = true;
        const rainCount = 3000;
        const rainPositions = new Float32Array(rainCount * 3);
        const rainVelocities = new Float32Array(rainCount);
        
        for (let i = 0; i < rainCount; i++) {
            rainPositions[i * 3] = (Math.random() - 0.5) * 40;
            rainPositions[i * 3 + 1] = Math.random() * 20;
            rainPositions[i * 3 + 2] = (Math.random() - 0.5) * 40;
            rainVelocities[i] = 5 + Math.random() * 5;
        }
        
        const rainGeo = new THREE.BufferGeometry();
        rainGeo.setAttribute('position', new THREE.BufferAttribute(rainPositions, 3));
        
        const rainMat = new THREE.PointsMaterial({
            color: 0xaaccff,
            size: 0.05,
            transparent: true,
            opacity: 0.6
        });
        
        const rain = new THREE.Points(rainGeo, rainMat);
        scene.add(rain);

        // Post-processing
        const composer = new EffectComposer(renderer);
        composer.addPass(new RenderPass(scene, camera));
        
        const bloomPass = new UnrealBloomPass(
            new THREE.Vector2(W, H),
            0.5,  // strength
            0.4,  // radius
            0.85  // threshold
        );
        composer.addPass(bloomPass);
        composer.addPass(new OutputPass());

        // UI Controls
        document.getElementById('opacity').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            wetMaterial.opacity = v;
            document.getElementById('opacityVal').textContent = v.toFixed(2);
        });
        document.getElementById('wetness').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            wetMaterial.roughness = 1.0 - v;
            document.getElementById('wetVal').textContent = v.toFixed(2);
        });
        document.getElementById('toggleSSR').addEventListener('click', () => {
            reflector.visible = !reflector.visible;
            wetFloor.visible = !wetFloor.visible;
        });
        document.getElementById('toggleRain').addEventListener('click', () => {
            rainEnabled = !rainEnabled;
            rain.visible = rainEnabled;
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
            const dt = clock.getDelta();
            const time = clock.getElapsedTime();
            
            // Animate neon lights
            neonLights.forEach((light, i) => {
                light.intensity = 2 + Math.sin(time * 2 + i) * 0.5;
            });
            
            // Update rain
            if (rainEnabled) {
                const pos = rainGeo.attributes.position.array;
                for (let i = 0; i < rainCount; i++) {
                    pos[i * 3 + 1] -= rainVelocities[i] * dt;
                    if (pos[i * 3 + 1] < 0) {
                        pos[i * 3 + 1] = 20;
                        pos[i * 3] = (Math.random() - 0.5) * 40;
                        pos[i * 3 + 2] = (Math.random() - 0.5) * 40;
                    }
                }
                rainGeo.attributes.position.needsUpdate = true;
            }
            
            // Update ripples
            updateRipples(time);
            
            controls.update();
            composer.render();
        }
        animate();
    </script>
</body>
</html>
```

## ตัวอย่างที่ 2: Metallic Surface Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Metallic Surface SSR</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #111; overflow: hidden; font-family: Arial, sans-serif; }
        canvas { display: block; }
        #panel {
            position: fixed;
            top: 20px;
            right: 20px;
            background: rgba(10,10,20,0.9);
            color: white;
            padding: 20px;
            border-radius: 10px;
            width: 260px;
        }
        h3 { color: #c0a060; margin-bottom: 15px; }
        .ctrl { margin-bottom: 10px; }
        .ctrl label { display: flex; justify-content: space-between; font-size: 12px; margin-bottom: 3px; }
        .ctrl input { width: 100%; }
        .material-btns { display: grid; grid-template-columns: 1fr 1fr; gap: 5px; margin-top: 8px; }
        .mat-btn {
            padding: 8px; border: 1px solid #333; background: #1a1a2e;
            color: #ccc; cursor: pointer; border-radius: 5px; font-size: 11px;
            text-align: center;
        }
        .mat-btn:hover, .mat-btn.active { background: #2a2a4e; border-color: #c0a060; color: #c0a060; }
    </style>
</head>
<body>
    <div id="panel">
        <h3>🔮 Metallic Surface</h3>
        <div class="ctrl">
            <label><span>Metalness</span><span id="metVal">0.9</span></label>
            <input type="range" id="metalness" min="0" max="1" step="0.01" value="0.9">
        </div>
        <div class="ctrl">
            <label><span>Roughness</span><span id="roughVal">0.1</span></label>
            <input type="range" id="roughness" min="0" max="1" step="0.01" value="0.1">
        </div>
        <div class="ctrl">
            <label><span>Env Map Intensity</span><span id="envVal">1.0</span></label>
            <input type="range" id="envIntensity" min="0" max="3" step="0.1" value="1.0">
        </div>
        <p style="font-size:12px;color:#888;margin:10px 0 5px">Preset Materials:</p>
        <div class="material-btns">
            <button class="mat-btn active" data-m="gold">Gold</button>
            <button class="mat-btn" data-m="chrome">Chrome</button>
            <button class="mat-btn" data-m="copper">Copper</button>
            <button class="mat-btn" data-m="brushed">Brushed Steel</button>
            <button class="mat-btn" data-m="anodized">Anodized</button>
            <button class="mat-btn" data-m="plastic">Glossy Plastic</button>
        </div>
    </div>

    <script type="module">
        import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js';
        import { EffectComposer } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/EffectComposer.js';
        import { RenderPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/RenderPass.js';
        import { OutputPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/OutputPass.js';
        import { Reflector } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/objects/Reflector.js';
        import { UnrealBloomPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/UnrealBloomPass.js';
        import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/controls/OrbitControls.js';
        import { RGBELoader } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/loaders/RGBELoader.js';

        const W = window.innerWidth, H = window.innerHeight;
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(W, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        renderer.toneMapping = THREE.ACESFilmicToneMapping;
        renderer.toneMappingExposure = 1.0;
        document.body.appendChild(renderer.domElement);

        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(50, W/H, 0.1, 100);
        camera.position.set(0, 3, 8);
        camera.lookAt(0, 1, 0);

        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.target.set(0, 1, 0);

        // ใช้ procedural environment map แทน HDRI
        const pmremGenerator = new THREE.PMREMGenerator(renderer);
        
        // สร้าง environment ด้วย gradient
        const envScene = new THREE.Scene();
        const envGeo = new THREE.SphereGeometry(50, 32, 32);
        const envMat = new THREE.ShaderMaterial({
            side: THREE.BackSide,
            uniforms: {
                topColor: { value: new THREE.Color(0x2255aa) },
                bottomColor: { value: new THREE.Color(0xffffff) },
            },
            vertexShader: `
                varying vec3 vPos;
                void main() {
                    vPos = position;
                    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
                }
            `,
            fragmentShader: `
                uniform vec3 topColor;
                uniform vec3 bottomColor;
                varying vec3 vPos;
                void main() {
                    float t = normalize(vPos).y * 0.5 + 0.5;
                    gl_FragColor = vec4(mix(bottomColor, topColor, t), 1.0);
                }
            `
        });
        envScene.add(new THREE.Mesh(envGeo, envMat));
        const envTexture = pmremGenerator.fromScene(envScene).texture;
        scene.environment = envTexture;
        scene.background = envTexture;

        // Lighting
        scene.add(new THREE.AmbientLight(0xffffff, 0.3));
        const keyLight = new THREE.DirectionalLight(0xffffff, 2);
        keyLight.position.set(5, 8, 5);
        keyLight.castShadow = true;
        keyLight.shadow.mapSize.set(2048, 2048);
        scene.add(keyLight);
        const fillLight = new THREE.DirectionalLight(0x4488ff, 0.5);
        fillLight.position.set(-5, 3, -5);
        scene.add(fillLight);

        // Materials
        const materials = {
            gold: { color: 0xffcc44, metalness: 0.9, roughness: 0.1 },
            chrome: { color: 0xffffff, metalness: 1.0, roughness: 0.0 },
            copper: { color: 0xd4642a, metalness: 0.9, roughness: 0.2 },
            brushed: { color: 0x888888, metalness: 0.8, roughness: 0.4 },
            anodized: { color: 0x2266dd, metalness: 0.7, roughness: 0.15 },
            plastic: { color: 0xcc3333, metalness: 0.0, roughness: 0.05 }
        };

        const mainMaterial = new THREE.MeshStandardMaterial({
            color: materials.gold.color,
            metalness: materials.gold.metalness,
            roughness: materials.gold.roughness,
            envMap: envTexture,
            envMapIntensity: 1.0
        });

        // Objects
        const objects = [];
        
        // Main sphere
        const sphere = new THREE.Mesh(
            new THREE.SphereGeometry(1.2, 64, 64),
            mainMaterial
        );
        sphere.position.set(0, 1.5, 0);
        sphere.castShadow = true;
        scene.add(sphere);
        objects.push(sphere);

        // Torus
        const torus = new THREE.Mesh(
            new THREE.TorusGeometry(0.8, 0.2, 32, 64),
            mainMaterial
        );
        torus.position.set(3, 1.2, 0);
        torus.rotation.x = Math.PI / 4;
        torus.castShadow = true;
        scene.add(torus);
        objects.push(torus);

        // Box
        const box = new THREE.Mesh(
            new THREE.BoxGeometry(1.5, 1.5, 1.5),
            mainMaterial
        );
        box.position.set(-3, 1.2, 0);
        box.rotation.y = Math.PI / 4;
        box.castShadow = true;
        scene.add(box);
        objects.push(box);

        // Reflective floor
        const floorReflector = new Reflector(
            new THREE.PlaneGeometry(30, 30),
            {
                clipBias: 0.003,
                textureWidth: W,
                textureHeight: H,
                color: new THREE.Color(0x333344)
            }
        );
        floorReflector.rotation.x = -Math.PI / 2;
        floorReflector.position.y = 0;
        scene.add(floorReflector);

        // Shiny floor overlay
        const floorMat = new THREE.MeshStandardMaterial({
            color: 0x111122,
            metalness: 0.5,
            roughness: 0.0,
            transparent: true,
            opacity: 0.6,
            envMap: envTexture
        });
        const floor = new THREE.Mesh(new THREE.PlaneGeometry(30, 30), floorMat);
        floor.rotation.x = -Math.PI / 2;
        floor.position.y = 0.005;
        floor.receiveShadow = true;
        scene.add(floor);

        // Post-processing
        const composer = new EffectComposer(renderer);
        composer.addPass(new RenderPass(scene, camera));
        
        const bloom = new UnrealBloomPass(new THREE.Vector2(W, H), 0.3, 0.4, 0.85);
        composer.addPass(bloom);
        composer.addPass(new OutputPass());

        // UI
        document.getElementById('metalness').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            mainMaterial.metalness = v;
            document.getElementById('metVal').textContent = v.toFixed(2);
        });
        document.getElementById('roughness').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            mainMaterial.roughness = v;
            document.getElementById('roughVal').textContent = v.toFixed(2);
        });
        document.getElementById('envIntensity').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            mainMaterial.envMapIntensity = v;
            document.getElementById('envVal').textContent = v.toFixed(1);
        });

        document.querySelectorAll('.mat-btn').forEach(btn => {
            btn.addEventListener('click', () => {
                document.querySelectorAll('.mat-btn').forEach(b => b.classList.remove('active'));
                btn.classList.add('active');
                const m = materials[btn.dataset.m];
                mainMaterial.color.setHex(m.color);
                mainMaterial.metalness = m.metalness;
                mainMaterial.roughness = m.roughness;
                document.getElementById('metalness').value = m.metalness;
                document.getElementById('roughness').value = m.roughness;
                document.getElementById('metVal').textContent = m.metalness.toFixed(2);
                document.getElementById('roughVal').textContent = m.roughness.toFixed(2);
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
            
            sphere.rotation.y = t * 0.3;
            torus.rotation.z = t * 0.5;
            box.rotation.y = t * 0.4;
            
            controls.update();
            composer.render();
        }
        animate();
    </script>
</body>
</html>
```

## SSR Limitations และ Workarounds

### ข้อจำกัดหลักของ SSR:

```javascript
// 1. Missing information at screen edges
// วัตถุที่อยู่นอก viewport จะไม่ถูกสะท้อน
// แก้ไข: ใช้ environment map เป็น fallback
//
// Blend ระหว่าง SSR และ env map
const blendedColor = mix(envMapColor, ssrColor, ssrConfidence);

// 2. Self-reflection artifacts
// วัตถุอาจสะท้อนตัวเอง
// แก้ไข: ตรวจสอบว่า reflected ray ชนกับวัตถุเดิมหรือไม่
if (hitObjectID == currentObjectID) {
    // ใช้ env map แทน
}

// 3. Performance
// SSR ใช้ทรัพยากรสูง
// แก้ไข: ลด resolution ของ SSR
// ลด maxSteps
// ใช้ temporal accumulation

// 4. Roughness-based blur
// วัตถุที่ roughness สูงควรมี blurry reflection
// แก้ไข: blur SSR output ตาม roughness value
function blurSSRByRoughness(ssrTexture, roughness, blurRadius) {
    const blurAmount = roughness * blurRadius;
    return gaussianBlur(ssrTexture, blurAmount);
}
```

## การรวม SSR กับ Environment Maps

```javascript
// Hybrid Reflection System
// ผสม SSR และ env map อย่างชาญฉลาด

const hybridReflectionShader = `
uniform sampler2D tSSR;       // SSR result
uniform sampler2D tEnvMap;    // environment map
uniform sampler2D tSSRMask;   // ความน่าเชื่อถือของ SSR (0-1)
uniform float envMapWeight;   // น้ำหนักของ env map

varying vec2 vUv;

void main() {
    vec4 ssr = texture2D(tSSR, vUv);
    vec4 env = texture2D(tEnvMap, vUv);
    float confidence = texture2D(tSSRMask, vUv).r;
    
    // ใช้ SSR เมื่อ confidence สูง
    // ใช้ env map เมื่อ SSR ไม่มีข้อมูล (ขอบ screen)
    float ssrWeight = confidence * (1.0 - envMapWeight);
    float envWeight = 1.0 - ssrWeight;
    
    // Fade SSR ที่ขอบ screen
    vec2 edgeFade = smoothstep(0.0, 0.1, vUv) * (1.0 - smoothstep(0.9, 1.0, vUv));
    float edgeMask = edgeFade.x * edgeFade.y;
    ssrWeight *= edgeMask;
    
    gl_FragColor = mix(env, ssr, ssrWeight);
}
`;
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Puddle Reflection System
สร้างระบบ puddle ที่:
- สร้าง puddle shapes แบบ random (ไม่ใช่ circle)
- มี ripple animation
- Reflection เปลี่ยนตาม camera angle
- Fade ออกเมื่อ "dry"

### แบบฝึกหัดที่ 2: Mirror Room
สร้างห้องที่มีผนังเป็นกระจก:
- ใช้ Reflector สำหรับผนังแต่ละด้าน
- จำกัด recursive reflections (ป้องกัน infinite loop)
- เพิ่ม distortion เล็กน้อย

### แบบฝึกหัดที่ 3: Car Showroom
สร้าง showroom รถที่มี:
- พื้น reflective
- รถ metallic
- Spotlight lighting
- เปรียบเทียบ env map กับ Reflector

### แบบฝึกหัดที่ 4: Roughness-Blurred Reflection
ทดลองสร้าง blurred reflection ตาม roughness:
- roughness 0 = sharp reflection
- roughness 0.5 = slightly blurred
- roughness 1.0 = no visible reflection

---

**สรุปบทที่ 52:**
- SSR ให้การสะท้อนที่สมจริงกว่า environment map แต่จำกัดเฉพาะ screen space
- Three.js SSRPass เหมาะสำหรับพื้นแนวนอน
- Reflector class ใช้งานง่ายกว่าสำหรับกรณีพิเศษ
- ควรผสม SSR กับ env map เพื่อแก้ปัญหาขอบ screen
- Roughness ควรส่งผลต่อความชัดของ reflection

**บทต่อไป:** Volumetric Lighting & God Rays
