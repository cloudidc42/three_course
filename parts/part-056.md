# Part 56: Advanced Normal & Displacement Mapping

## แนวคิด Normal Mapping

Normal Mapping เป็นเทคนิคที่ใช้ texture เพื่อจำลองรายละเอียดบนพื้นผิวโดยไม่ต้องเพิ่ม polygon ใช้สีใน texture แทนทิศทาง normal ของแต่ละจุด

### Tangent Space vs Object Space Normals

**Tangent Space Normal Map** (สีน้ำเงินม่วง):
- เป็น standard สำหรับ game assets
- Normal ถูก encode relative ต่อ surface
- ใช้งานร่วมกับ UV ได้ (tiling, mirroring)
- ดู re-use ได้กับ mesh ที่หมุน

**Object Space Normal Map** (สีสดใส):
- Normal encode ใน object local space
- ทำให้ texture มีสีสว่างชัดเจน
- ไม่รองรับ tiling
- เหมาะสำหรับ character faces, unique assets

```javascript
// การใช้ Normal Map ใน Three.js
const material = new THREE.MeshStandardMaterial({
    color: 0xffffff,
    map: albedoTexture,
    normalMap: normalTexture,
    normalMapType: THREE.TangentSpaceNormalMap, // default - tangent space
    // หรือ THREE.ObjectSpaceNormalMap
    normalScale: new THREE.Vector2(1, 1) // ควบคุมความแรง (x=horizontal, y=vertical)
});

// normalScale เปลี่ยนได้ตาม runtime
material.normalScale.set(2.0, 2.0); // เพิ่ม normal strength
material.normalScale.set(0.3, 0.3); // ลด normal strength

// Normal scale ที่ต่างกันในแต่ละ axis ทำให้เกิด anisotropic effect
material.normalScale.set(1.0, 0.3);  // stretch แนวตั้ง
```

## Displacement Map (Vertex Displacement)

Displacement map เปลี่ยนตำแหน่งของ vertex จริงๆ ต่างจาก normal map ที่เปลี่ยนแค่ lighting:

```javascript
// Displacement map ใน MeshStandardMaterial
const material = new THREE.MeshStandardMaterial({
    displacementMap: displacementTexture,
    displacementScale: 0.5,    // ขนาดของ displacement
    displacementBias: -0.25    // offset ของ displacement
});

// Geometry ต้องมี subdivisions เพียงพอ
const geometry = new THREE.PlaneGeometry(10, 10, 512, 512); // 512x512 subdivisions
const mesh = new THREE.Mesh(geometry, material);
```

## Custom Displacement Shader

```glsl
// Vertex shader พร้อม custom displacement
uniform sampler2D displacementMap;
uniform float displacementScale;
uniform float displacementBias;
uniform float time;

varying vec3 vNormal;
varying vec2 vUv;
varying vec3 vWorldPos;

void main() {
    vUv = uv;
    
    // Sample displacement
    float displacement = texture2D(displacementMap, uv).r;
    displacement = displacement * displacementScale + displacementBias;
    
    // Displace along normal
    vec3 displaced = position + normal * displacement;
    
    vNormal = normalize(normalMatrix * normal);
    vWorldPos = (modelMatrix * vec4(displaced, 1.0)).xyz;
    
    gl_Position = projectionMatrix * modelViewMatrix * vec4(displaced, 1.0);
}
```

## Parallax Occlusion Mapping (POM)

POM เป็นเทคนิคที่จำลอง depth บน flat surface โดยใช้เฉพาะ fragment shader ให้ความรู้สึก 3D มากกว่า normal map:

```javascript
// POM Shader
const pomShader = {
    uniforms: {
        map: { value: null },
        normalMap: { value: null },
        heightMap: { value: null },
        heightScale: { value: 0.1 },
        minLayers: { value: 8.0 },
        maxLayers: { value: 32.0 },
        lightPosition: { value: new THREE.Vector3(1, 1, 1) }
    },
    vertexShader: `
        attribute vec4 tangent;
        
        varying vec3 vTangentViewPos;
        varying vec3 vTangentFragPos;
        varying vec3 vTangentLightPos;
        varying vec2 vUv;
        
        uniform vec3 lightPosition;
        
        void main() {
            vUv = uv;
            
            vec3 T = normalize((modelMatrix * vec4(tangent.xyz, 0.0)).xyz);
            vec3 N = normalize((modelMatrix * vec4(normal, 0.0)).xyz);
            T = normalize(T - dot(T, N) * N);
            vec3 B = cross(N, T) * tangent.w;
            
            mat3 TBN = transpose(mat3(T, B, N));
            
            vec3 worldPos = (modelMatrix * vec4(position, 1.0)).xyz;
            vec3 cameraPos = (inverse(viewMatrix) * vec4(0, 0, 0, 1)).xyz;
            
            vTangentViewPos = TBN * cameraPos;
            vTangentFragPos = TBN * worldPos;
            vTangentLightPos = TBN * lightPosition;
        }
    `,
    fragmentShader: `
        uniform sampler2D map;
        uniform sampler2D normalMap;
        uniform sampler2D heightMap;
        uniform float heightScale;
        uniform float minLayers;
        uniform float maxLayers;
        
        varying vec3 vTangentViewPos;
        varying vec3 vTangentFragPos;
        varying vec3 vTangentLightPos;
        varying vec2 vUv;
        
        // Parallax Occlusion Mapping
        vec2 parallaxOcclusionMapping(vec2 texCoords, vec3 viewDir) {
            // จำนวน layers มากขึ้นเมื่อมุม grazing
            float numLayers = mix(maxLayers, minLayers, max(dot(vec3(0.0, 0.0, 1.0), viewDir), 0.0));
            float layerDepth = 1.0 / numLayers;
            float currentLayerDepth = 0.0;
            
            // Amount to shift UV per layer
            vec2 P = viewDir.xy * heightScale;
            vec2 deltaTexCoords = P / numLayers;
            
            vec2 currentTexCoords = texCoords;
            float currentDepthMapValue = texture2D(heightMap, currentTexCoords).r;
            
            // Ray march
            while (currentLayerDepth < currentDepthMapValue) {
                currentTexCoords -= deltaTexCoords;
                currentDepthMapValue = texture2D(heightMap, currentTexCoords).r;
                currentLayerDepth += layerDepth;
            }
            
            // Interpolate ระหว่าง layers สุดท้าย (Occlusion Mapping)
            vec2 prevTexCoords = currentTexCoords + deltaTexCoords;
            float afterDepth = currentDepthMapValue - currentLayerDepth;
            float beforeDepth = texture2D(heightMap, prevTexCoords).r - currentLayerDepth + layerDepth;
            float weight = afterDepth / (afterDepth - beforeDepth);
            
            return mix(currentTexCoords, prevTexCoords, weight);
        }
        
        void main() {
            vec3 viewDir = normalize(vTangentViewPos - vTangentFragPos);
            
            // Apply POM
            vec2 texCoords = parallaxOcclusionMapping(vUv, viewDir);
            
            // Discard ถ้า UV อยู่นอก [0,1]
            if (texCoords.x > 1.0 || texCoords.y > 1.0 || texCoords.x < 0.0 || texCoords.y < 0.0) {
                discard;
            }
            
            // Sample textures ด้วย parallax-corrected UV
            vec3 albedo = texture2D(map, texCoords).rgb;
            vec3 normal = texture2D(normalMap, texCoords).xyz * 2.0 - 1.0;
            
            // Lighting
            vec3 lightDir = normalize(vTangentLightPos - vTangentFragPos);
            float diff = max(dot(lightDir, normal), 0.0);
            
            vec3 ambient = 0.1 * albedo;
            vec3 diffuse = diff * albedo;
            
            // Specular
            vec3 halfDir = normalize(lightDir + viewDir);
            float spec = pow(max(dot(normal, halfDir), 0.0), 32.0);
            vec3 specular = vec3(0.2) * spec;
            
            gl_FragColor = vec4(ambient + diffuse + specular, 1.0);
        }
    `
};
```

## ตัวอย่าง: Stone Wall POM Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>POM Stone Wall Demo</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; font-family: Arial, sans-serif; }
        canvas { display: block; }
        #controls {
            position: fixed;
            top: 20px;
            right: 20px;
            background: rgba(20,15,10,0.9);
            color: #d4a970;
            padding: 20px;
            border-radius: 10px;
            width: 260px;
            border: 1px solid #6b4c30;
        }
        h3 { color: #d4a970; margin-bottom: 15px; }
        .ctrl { margin-bottom: 10px; }
        .ctrl label { display: flex; justify-content: space-between; font-size: 12px; margin-bottom: 3px; }
        .ctrl input { width: 100%; accent-color: #d4a970; }
        .mode-btn {
            width: 100%; padding: 7px; background: transparent;
            color: #d4a970; border: 1px solid #6b4c30; cursor: pointer;
            margin-bottom: 4px; font-size: 12px; border-radius: 4px;
        }
        .mode-btn.active { background: rgba(212,169,112,0.2); }
    </style>
</head>
<body>
    <div id="controls">
        <h3>🪨 Stone Wall POM</h3>
        <div class="ctrl">
            <label><span>Height Scale</span><span id="hsVal">0.08</span></label>
            <input type="range" id="heightScale" min="0" max="0.3" step="0.005" value="0.08">
        </div>
        <div class="ctrl">
            <label><span>Min Layers</span><span id="mlVal">8</span></label>
            <input type="range" id="minLayers" min="4" max="16" step="1" value="8">
        </div>
        <div class="ctrl">
            <label><span>Max Layers</span><span id="mxlVal">32</span></label>
            <input type="range" id="maxLayers" min="16" max="64" step="4" value="32">
        </div>
        <div class="ctrl">
            <label><span>Normal Scale</span><span id="nsVal">1.0</span></label>
            <input type="range" id="normalScale" min="0" max="3" step="0.1" value="1.0">
        </div>
        <div class="ctrl">
            <label><span>Light Rotation</span><span id="lrVal">0°</span></label>
            <input type="range" id="lightRot" min="-180" max="180" step="1" value="0">
        </div>
        <p style="font-size:11px;color:#8b6545;margin-top:10px">
            Mapping Method:
        </p>
        <button class="mode-btn" data-mode="normal">Normal Map Only</button>
        <button class="mode-btn active" data-mode="pom">+ Parallax Occlusion</button>
        <button class="mode-btn" data-mode="displacement">+ Displacement</button>
        <p style="font-size:11px;color:#8b6545;margin-top:10px">
            ลากเมาส์เพื่อดู parallax effect<br>
            (มองจากมุมเฉียงเห็นชัดที่สุด)
        </p>
    </div>

    <script type="module">
        import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js';
        import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/controls/OrbitControls.js';

        const W = window.innerWidth, H = window.innerHeight;
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(W, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        document.body.appendChild(renderer.domElement);

        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x111008);

        const camera = new THREE.PerspectiveCamera(60, W/H, 0.1, 100);
        camera.position.set(0, 0, 4);
        camera.lookAt(0, 0, 0);

        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.enablePan = false;

        // สร้าง procedural stone wall texture
        function generateStoneTextures() {
            const size = 512;
            
            // Albedo (diffuse color)
            const albedoCanvas = document.createElement('canvas');
            albedoCanvas.width = albedoCanvas.height = size;
            const actx = albedoCanvas.getContext('2d');
            
            // Height map
            const heightCanvas = document.createElement('canvas');
            heightCanvas.width = heightCanvas.height = size;
            const hctx = heightCanvas.getContext('2d');
            
            // Normal map
            const normalCanvas = document.createElement('canvas');
            normalCanvas.width = normalCanvas.height = size;
            const nctx = normalCanvas.getContext('2d');
            
            const stoneData = actx.createImageData(size, size);
            const heightData = hctx.createImageData(size, size);
            const normalData = nctx.createImageData(size, size);
            
            const heights = new Float32Array(size * size);
            
            // Simple noise function
            function noise(x, y, scale = 1) {
                const ix = Math.floor(x * scale);
                const iy = Math.floor(y * scale);
                const fx = x * scale - ix;
                const fy = y * scale - iy;
                
                const h00 = Math.sin(ix * 127.1 + iy * 311.7) * 43758.5453;
                const h10 = Math.sin((ix+1) * 127.1 + iy * 311.7) * 43758.5453;
                const h01 = Math.sin(ix * 127.1 + (iy+1) * 311.7) * 43758.5453;
                const h11 = Math.sin((ix+1) * 127.1 + (iy+1) * 311.7) * 43758.5453;
                
                const ux = fx * fx * (3 - 2 * fx);
                const uy = fy * fy * (3 - 2 * fy);
                
                return ((h00 % 1 + 1) % 1) + 
                    (((h10 % 1 + 1) % 1) - ((h00 % 1 + 1) % 1)) * ux +
                    (((h01 % 1 + 1) % 1) - ((h00 % 1 + 1) % 1)) * uy +
                    (((h11 % 1 + 1) % 1) - ((h10 % 1 + 1) % 1) - ((h01 % 1 + 1) % 1) + ((h00 % 1 + 1) % 1)) * ux * uy;
            }
            
            // Generate stone pattern
            const brickWidth = 80, brickHeight = 40;
            
            for (let y = 0; y < size; y++) {
                for (let x = 0; x < size; x++) {
                    const idx = (y * size + x) * 4;
                    
                    // Determine which brick
                    const row = Math.floor(y / brickHeight);
                    const offset = row % 2 === 0 ? 0 : brickWidth / 2;
                    const col = Math.floor((x + offset) / brickWidth);
                    
                    // Distance to mortar (nearest grid line)
                    const bx = ((x + offset) % brickWidth);
                    const by = (y % brickHeight);
                    const mortarDist = Math.min(
                        Math.min(bx, brickWidth - bx),
                        Math.min(by, brickHeight - by)
                    );
                    
                    const isMortar = mortarDist < 3;
                    
                    // Height
                    let h;
                    if (isMortar) {
                        h = 0.3 + noise(x / size, y / size, 8) * 0.1;
                    } else {
                        // Stone surface with detail
                        h = 0.5 + noise(x / size * 4, y / size * 4, 2) * 0.3 +
                            noise(x / size, y / size, 8) * 0.15;
                        // เพิ่ม edge rounding
                        const edgeFactor = Math.min(mortarDist / 5, 1);
                        h *= edgeFactor;
                    }
                    
                    heights[y * size + x] = h;
                    
                    // Albedo color
                    let r, g, b;
                    if (isMortar) {
                        const v = 120 + noise(x / size, y / size, 16) * 30;
                        r = v; g = v - 5; b = v - 10;
                    } else {
                        const stoneType = noise(col, row, 0.1); // ชนิดหิน
                        if (stoneType < 0.33) {
                            r = 150 + noise(x/size, y/size, 6) * 40;
                            g = 120 + noise(x/size+1, y/size, 6) * 30;
                            b = 90 + noise(x/size, y/size+1, 6) * 20;
                        } else if (stoneType < 0.66) {
                            r = 130 + noise(x/size, y/size, 6) * 35;
                            g = 125 + noise(x/size+1, y/size, 6) * 30;
                            b = 115 + noise(x/size, y/size+1, 6) * 25;
                        } else {
                            r = 140 + noise(x/size, y/size, 6) * 25;
                            g = 135 + noise(x/size+1, y/size, 6) * 25;
                            b = 130 + noise(x/size, y/size+1, 6) * 20;
                        }
                        r = r * (0.7 + h * 0.5);
                        g = g * (0.7 + h * 0.5);
                        b = b * (0.7 + h * 0.5);
                    }
                    
                    stoneData.data[idx] = Math.min(255, r);
                    stoneData.data[idx + 1] = Math.min(255, g);
                    stoneData.data[idx + 2] = Math.min(255, b);
                    stoneData.data[idx + 3] = 255;
                    
                    // Height map
                    const hv = h * 255;
                    heightData.data[idx] = hv;
                    heightData.data[idx + 1] = hv;
                    heightData.data[idx + 2] = hv;
                    heightData.data[idx + 3] = 255;
                }
            }
            
            // Compute normals from height
            for (let y = 0; y < size; y++) {
                for (let x = 0; x < size; x++) {
                    const idx = (y * size + x) * 4;
                    
                    const xl = heights[y * size + Math.max(0, x - 1)];
                    const xr = heights[y * size + Math.min(size - 1, x + 1)];
                    const yd = heights[Math.max(0, y - 1) * size + x];
                    const yu = heights[Math.min(size - 1, y + 1) * size + x];
                    
                    const nx = (xl - xr) * 5;
                    const ny = (yd - yu) * 5;
                    const nz = 1.0;
                    
                    const len = Math.sqrt(nx*nx + ny*ny + nz*nz);
                    
                    normalData.data[idx] = Math.floor((nx / len * 0.5 + 0.5) * 255);
                    normalData.data[idx + 1] = Math.floor((ny / len * 0.5 + 0.5) * 255);
                    normalData.data[idx + 2] = Math.floor((nz / len * 0.5 + 0.5) * 255);
                    normalData.data[idx + 3] = 255;
                }
            }
            
            actx.putImageData(stoneData, 0, 0);
            hctx.putImageData(heightData, 0, 0);
            nctx.putImageData(normalData, 0, 0);
            
            const makeTexture = (canvas, repeat = 2) => {
                const tex = new THREE.CanvasTexture(canvas);
                tex.wrapS = tex.wrapT = THREE.RepeatWrapping;
                tex.repeat.set(repeat, repeat);
                return tex;
            };
            
            return {
                albedo: makeTexture(albedoCanvas),
                height: makeTexture(heightCanvas),
                normal: makeTexture(normalCanvas)
            };
        }

        console.log('Generating stone textures...');
        const stoneTextures = generateStoneTextures();

        // POM Shader Material
        const pomUniforms = {
            map: { value: stoneTextures.albedo },
            normalMap: { value: stoneTextures.normal },
            heightMap: { value: stoneTextures.height },
            heightScale: { value: 0.08 },
            minLayers: { value: 8.0 },
            maxLayers: { value: 32.0 },
            normalScale: { value: 1.0 },
            lightPos: { value: new THREE.Vector3(2, 2, 3) },
            viewPos: { value: camera.position }
        };

        const pomMaterial = new THREE.ShaderMaterial({
            uniforms: pomUniforms,
            vertexShader: `
                attribute vec4 tangent;
                
                uniform vec3 lightPos;
                uniform vec3 viewPos;
                
                varying vec3 vTangentLightPos;
                varying vec3 vTangentViewPos;
                varying vec3 vTangentFragPos;
                varying vec2 vUv;
                
                void main() {
                    vUv = uv;
                    
                    vec3 worldPos = (modelMatrix * vec4(position, 1.0)).xyz;
                    vTangentFragPos = worldPos;
                    
                    vec3 T = normalize(mat3(modelMatrix) * tangent.xyz);
                    vec3 N = normalize(mat3(modelMatrix) * normal);
                    T = normalize(T - dot(T, N) * N);
                    vec3 B = cross(N, T) * tangent.w;
                    mat3 TBN = transpose(mat3(T, B, N));
                    
                    vTangentLightPos = TBN * lightPos;
                    vTangentViewPos = TBN * viewPos;
                    vTangentFragPos = TBN * worldPos;
                    
                    gl_Position = projectionMatrix * viewMatrix * modelMatrix * vec4(position, 1.0);
                }
            `,
            fragmentShader: `
                uniform sampler2D map;
                uniform sampler2D normalMap;
                uniform sampler2D heightMap;
                uniform float heightScale;
                uniform float minLayers;
                uniform float maxLayers;
                uniform float normalScale;
                
                varying vec3 vTangentLightPos;
                varying vec3 vTangentViewPos;
                varying vec3 vTangentFragPos;
                varying vec2 vUv;
                
                vec2 pomMapping(vec2 uv, vec3 viewDir) {
                    float numLayers = mix(maxLayers, minLayers, abs(dot(vec3(0.0, 0.0, 1.0), viewDir)));
                    float layerDepth = 1.0 / numLayers;
                    float currentDepth = 0.0;
                    
                    vec2 P = viewDir.xy / viewDir.z * heightScale;
                    vec2 delta = P / numLayers;
                    
                    vec2 currentUV = uv;
                    float currentHeight = 1.0 - texture2D(heightMap, currentUV).r;
                    
                    for (float i = 0.0; i < 64.0; i++) {
                        if (i >= numLayers || currentDepth >= currentHeight) break;
                        currentUV -= delta;
                        currentHeight = 1.0 - texture2D(heightMap, currentUV).r;
                        currentDepth += layerDepth;
                    }
                    
                    // Refinement
                    vec2 prevUV = currentUV + delta;
                    float nextHeight = currentHeight - currentDepth;
                    float prevHeight = 1.0 - texture2D(heightMap, prevUV).r - (currentDepth - layerDepth);
                    float weight = nextHeight / (nextHeight - prevHeight);
                    return mix(currentUV, prevUV, weight);
                }
                
                void main() {
                    vec3 viewDir = normalize(vTangentViewPos - vTangentFragPos);
                    vec2 mappedUV = pomMapping(vUv, viewDir);
                    
                    // Discard outside UV
                    if (mappedUV.x > 1.0 || mappedUV.y > 1.0 || mappedUV.x < 0.0 || mappedUV.y < 0.0) discard;
                    
                    vec3 albedo = texture2D(map, mappedUV).rgb;
                    
                    vec3 norm = texture2D(normalMap, mappedUV).rgb * 2.0 - 1.0;
                    norm.xy *= normalScale;
                    norm = normalize(norm);
                    
                    vec3 lightDir = normalize(vTangentLightPos - vTangentFragPos);
                    
                    // Ambient
                    vec3 ambient = 0.15 * albedo;
                    
                    // Diffuse
                    float diff = max(dot(norm, lightDir), 0.0);
                    vec3 diffuse = diff * albedo * vec3(1.0, 0.95, 0.85);
                    
                    // Specular (Blinn-Phong)
                    vec3 halfDir = normalize(lightDir + viewDir);
                    float spec = pow(max(dot(norm, halfDir), 0.0), 64.0);
                    vec3 specular = spec * vec3(0.15);
                    
                    // Shadow term (simple)
                    float shadow = smoothstep(0.0, 0.2, diff);
                    
                    gl_FragColor = vec4(ambient + (diffuse + specular) * shadow, 1.0);
                }
            `
        });

        // Stone wall plane
        const planeGeo = new THREE.PlaneGeometry(4, 4, 1, 1);
        // เพิ่ม tangent attribute
        planeGeo.computeTangents();
        const wallMesh = new THREE.Mesh(planeGeo, pomMaterial);
        scene.add(wallMesh);

        // Light helper
        const lightHelper = new THREE.Mesh(
            new THREE.SphereGeometry(0.05),
            new THREE.MeshBasicMaterial({ color: 0xffffff })
        );
        lightHelper.position.set(2, 2, 3);
        scene.add(lightHelper);

        // UI Controls
        document.getElementById('heightScale').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            pomUniforms.heightScale.value = v;
            document.getElementById('hsVal').textContent = v.toFixed(3);
        });
        document.getElementById('minLayers').addEventListener('input', (e) => {
            pomUniforms.minLayers.value = parseFloat(e.target.value);
            document.getElementById('mlVal').textContent = e.target.value;
        });
        document.getElementById('maxLayers').addEventListener('input', (e) => {
            pomUniforms.maxLayers.value = parseFloat(e.target.value);
            document.getElementById('mxlVal').textContent = e.target.value;
        });
        document.getElementById('normalScale').addEventListener('input', (e) => {
            pomUniforms.normalScale.value = parseFloat(e.target.value);
            document.getElementById('nsVal').textContent = e.target.value;
        });
        document.getElementById('lightRot').addEventListener('input', (e) => {
            const angle = parseFloat(e.target.value) * Math.PI / 180;
            lightHelper.position.set(Math.sin(angle) * 3, 1.5, Math.cos(angle) * 3);
            pomUniforms.lightPos.value.copy(lightHelper.position);
            document.getElementById('lrVal').textContent = `${e.target.value}°`;
        });

        document.querySelectorAll('.mode-btn').forEach(btn => {
            btn.addEventListener('click', () => {
                document.querySelectorAll('.mode-btn').forEach(b => b.classList.remove('active'));
                btn.classList.add('active');
                if (btn.dataset.mode === 'normal') {
                    pomUniforms.heightScale.value = 0;
                    document.getElementById('heightScale').value = 0;
                    document.getElementById('hsVal').textContent = '0.000';
                } else if (btn.dataset.mode === 'pom') {
                    pomUniforms.heightScale.value = 0.08;
                    document.getElementById('heightScale').value = 0.08;
                    document.getElementById('hsVal').textContent = '0.080';
                } else if (btn.dataset.mode === 'displacement') {
                    pomUniforms.heightScale.value = 0.15;
                    document.getElementById('heightScale').value = 0.15;
                    document.getElementById('hsVal').textContent = '0.150';
                }
            });
        });

        window.addEventListener('resize', () => {
            const w = window.innerWidth, h = window.innerHeight;
            camera.aspect = w / h; camera.updateProjectionMatrix();
            renderer.setSize(w, h);
        });

        const clock = new THREE.Clock();
        function animate() {
            requestAnimationFrame(animate);
            pomUniforms.viewPos.value.copy(camera.position);
            controls.update();
            renderer.render(scene, camera);
        }
        animate();
    </script>
</body>
</html>
```

## Steep Parallax Mapping

Steep parallax ใช้ลำดับขั้นมากขึ้นสำหรับ depth สูง:

```glsl
// Steep Parallax Mapping - ให้ผลดีกว่า basic parallax
vec2 steepParallaxMapping(vec2 uv, vec3 viewDir, float numLayers, float heightScale) {
    float layerDepth = 1.0 / numLayers;
    float currentDepth = 0.0;
    
    vec2 P = viewDir.xy * heightScale;
    vec2 deltaUV = P / numLayers;
    
    vec2 currentUV = uv;
    float sampledDepth = texture2D(heightMap, currentUV).r;
    
    while (currentDepth < sampledDepth) {
        currentUV -= deltaUV;
        sampledDepth = texture2D(heightMap, currentUV).r;
        currentDepth += layerDepth;
    }
    
    return currentUV;
}

// Relief Mapping - ที่แม่นยำที่สุด
vec2 reliefMapping(vec2 uv, vec3 viewDir, float heightScale) {
    // Step 1: Steep parallax
    vec2 P = viewDir.xy * heightScale;
    int numSteps = 32;
    float stepSize = 1.0 / float(numSteps);
    float depth = 0.0;
    
    vec2 currentUV = uv;
    float sampledDepth = 1.0;
    vec2 delta = P / float(numSteps);
    
    while (depth < sampledDepth) {
        currentUV -= delta;
        sampledDepth = 1.0 - texture2D(heightMap, currentUV).r;
        depth += stepSize;
    }
    
    // Step 2: Binary search refinement
    vec2 prevUV = currentUV + delta;
    for (int i = 0; i < 5; i++) {
        vec2 midUV = (currentUV + prevUV) * 0.5;
        float midDepth = 1.0 - texture2D(heightMap, midUV).r;
        float midSample = depth - stepSize * 0.5;
        
        if (midSample < midDepth) {
            prevUV = midUV;
        } else {
            currentUV = midUV;
        }
        
        delta *= 0.5;
    }
    
    return (currentUV + prevUV) * 0.5;
}
```

## Detail Normal Maps

```javascript
// การผสม detail normal maps หลายชั้น
const detailNormalShader = `
uniform sampler2D normalMap1;    // macro normals (low frequency)
uniform sampler2D normalMap2;    // detail normals (high frequency)
uniform float detailScale;       // tiling ของ detail map
uniform float detailStrength;    // ความแรงของ detail

varying vec2 vUv;

// Whiteout blending สำหรับ normal maps
vec3 blendNormals(vec3 n1, vec3 n2) {
    n1 += vec3(0.0, 0.0, 1.0);
    n2 *= vec3(-1.0, -1.0, 1.0);
    return n1 * dot(n1, n2) / n1.z - n2;
}

void main() {
    vec3 n1 = texture2D(normalMap1, vUv).xyz * 2.0 - 1.0;
    vec3 n2 = texture2D(normalMap2, vUv * detailScale).xyz * 2.0 - 1.0;
    n2.xy *= detailStrength;
    
    vec3 blended = normalize(blendNormals(n1, n2));
    gl_FragColor = vec4(blended * 0.5 + 0.5, 1.0);
}
`;
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Material Library
สร้าง library ของวัสดุที่ใช้ POM:
- หินอ่อน (marble)
- พื้นไม้ (wood planks)
- หิน cobblestone
- โลหะ brushed

### แบบฝึกหัดที่ 2: Terrain POM
สร้าง terrain แบบ flat mesh ที่ใช้ POM:
- ดูเหมือนมีความสูงต่ำ
- ควบคุม height scale ตาม zoom level
- Mix หลาย terrain types

### แบบฝึกหัดที่ 3: Normal Map Baking Tool
สร้าง tool อย่างง่ายที่:
- รับ height map เป็น input
- คำนวณ normal map
- Export เป็น PNG

### แบบฝึกหัดที่ 4: Animated Displacement
สร้าง sphere ที่:
- Noise-based displacement animate ตามเวลา
- Deform เมื่อคลิก
- Spring back animation

---

**สรุปบทที่ 56:**
- Normal mapping เพิ่ม fake surface detail โดยไม่เพิ่ม geometry
- Tangent space normals ยืดหยุ่นกว่าสำหรับ reusable assets
- Displacement map เปลี่ยน geometry จริงๆ ต้องการ subdivisions มาก
- Parallax Occlusion Mapping ให้ความลึก ที่สมจริงมากกว่า normal map โดยใช้ fragment shader
- Relief mapping เป็น POM ที่แม่นยำที่สุดแต่แพงที่สุด
- Detail normal maps ผสมกัน ด้วย whiteout blending ให้ผลดีสุด

**บทต่อไป:** Level of Detail (LOD)
