# Part 84: Deferred Rendering & G-Buffer

## บทนำ: Forward vs Deferred Rendering

### Forward Rendering (แบบปกติ)
```
สำหรับแต่ละ object:
  สำหรับแต่ละ light:
    คำนวณ lighting = O(objects × lights)
```

ปัญหาคือ ถ้ามี 100 objects และ 100 lights จะต้องคำนวณ 10,000 ครั้ง

### Deferred Rendering
```
Pass 1 (Geometry Pass): วาด geometry -> G-Buffer
  - เก็บ albedo, normal, depth, roughness, metalness

Pass 2 (Lighting Pass): คำนวณ lighting จาก G-Buffer
  สำหรับแต่ละ light:
    คำนวณจาก screen-space data = O(lights + pixels)
```

Deferred Rendering ทำให้สามารถมี lights จำนวนมากได้โดยไม่ทำให้ช้า

---

## 1. G-Buffer คืออะไร?

G-Buffer (Geometry Buffer) คือชุดของ textures ที่เก็บข้อมูล geometry ไว้:

```
G-Buffer ประกอบด้วย:
┌─────────────────────────────────────────┐
│  Albedo Buffer:   [R, G, B, A]          │  → สีพื้นผิว
│  Normal Buffer:   [Nx, Ny, Nz, _]       │  → ทิศทาง normal
│  Position Buffer: [X, Y, Z, Depth]      │  → world position
│  Material Buffer: [Roughness, Metalness, AO, _] │
└─────────────────────────────────────────┘
```

---

## 2. G-Buffer ด้วย WebGLMultipleRenderTargets

```javascript
import * as THREE from 'three';

class GBuffer {
  constructor(width, height) {
    this.width = width;
    this.height = height;
    
    // สร้าง MRT (Multiple Render Targets)
    this.renderTarget = new THREE.WebGLMultipleRenderTargets(width, height, 4, {
      minFilter: THREE.NearestFilter,
      magFilter: THREE.NearestFilter,
      type: THREE.HalfFloatType,
      depthBuffer: true,
      stencilBuffer: false
    });
    
    // ตั้งชื่อ textures
    this.renderTarget.texture[0].name = 'diffuse';   // albedo
    this.renderTarget.texture[1].name = 'normal';    // normals
    this.renderTarget.texture[2].name = 'position';  // world pos
    this.renderTarget.texture[3].name = 'material';  // roughness/metalness
    
    // ตั้งค่า format สำหรับแต่ละ buffer
    this.renderTarget.texture[0].format = THREE.RGBAFormat;
    this.renderTarget.texture[1].format = THREE.RGBAFormat;
    this.renderTarget.texture[2].format = THREE.RGBAFormat;
    this.renderTarget.texture[3].format = THREE.RGBAFormat;
    
    // Getters
    this.albedoTexture = this.renderTarget.texture[0];
    this.normalTexture = this.renderTarget.texture[1];
    this.positionTexture = this.renderTarget.texture[2];
    this.materialTexture = this.renderTarget.texture[3];
    this.depthTexture = this.renderTarget.depthTexture;
  }
  
  resize(width, height) {
    this.width = width;
    this.height = height;
    this.renderTarget.setSize(width, height);
  }
  
  dispose() {
    this.renderTarget.dispose();
    this.renderTarget.texture.forEach(t => t.dispose());
  }
}
```

---

## 3. G-Buffer Write Shader (Geometry Pass)

```javascript
// Shader สำหรับ write ลง G-Buffer
const gBufferWriteVert = `
  varying vec3 vWorldPos;
  varying vec3 vNormal;
  varying vec2 vUv;
  varying vec4 vClipPos;
  
  void main() {
    vUv = uv;
    
    // World space normal
    vNormal = normalize((modelMatrix * vec4(normal, 0.0)).xyz);
    
    // World space position
    vec4 worldPos = modelMatrix * vec4(position, 1.0);
    vWorldPos = worldPos.xyz;
    
    vClipPos = projectionMatrix * viewMatrix * worldPos;
    gl_Position = vClipPos;
  }
`;

const gBufferWriteFrag = `
  layout(location = 0) out vec4 gAlbedo;
  layout(location = 1) out vec4 gNormal;
  layout(location = 2) out vec4 gPosition;
  layout(location = 3) out vec4 gMaterial;
  
  uniform vec3 uColor;
  uniform float uRoughness;
  uniform float uMetalness;
  uniform float uAO;
  uniform sampler2D uAlbedoMap;
  uniform sampler2D uNormalMap;
  uniform sampler2D uRoughnessMap;
  uniform sampler2D uMetalnessMap;
  uniform bool uUseTextures;
  
  varying vec3 vWorldPos;
  varying vec3 vNormal;
  varying vec2 vUv;
  
  void main() {
    // Albedo
    vec3 albedo = uColor;
    if (uUseTextures) {
      albedo *= texture2D(uAlbedoMap, vUv).rgb;
    }
    gAlbedo = vec4(albedo, 1.0);
    
    // Normal (encode -1..1 ไปเป็น 0..1)
    vec3 normal = normalize(vNormal);
    // Normal map tangent space
    if (uUseTextures) {
      // Simplified: ใช้ world normal ตรงๆ
      // ในการใช้งานจริงต้องทำ TBN matrix
    }
    gNormal = vec4(normal * 0.5 + 0.5, 1.0);
    
    // World Position
    gPosition = vec4(vWorldPos, gl_FragCoord.z);  // z = depth
    
    // Material properties
    float roughness = uRoughness;
    float metalness = uMetalness;
    float ao = uAO;
    if (uUseTextures) {
      roughness *= texture2D(uRoughnessMap, vUv).r;
      metalness *= texture2D(uMetalnessMap, vUv).r;
    }
    gMaterial = vec4(roughness, metalness, ao, 1.0);
  }
`;

// สร้าง Material สำหรับ Geometry Pass
function createGBufferMaterial(params = {}) {
  return new THREE.RawShaderMaterial({
    glslVersion: THREE.GLSL3,
    uniforms: {
      uColor: { value: new THREE.Color(params.color || 0xffffff) },
      uRoughness: { value: params.roughness ?? 0.5 },
      uMetalness: { value: params.metalness ?? 0.0 },
      uAO: { value: 1.0 },
      uAlbedoMap: { value: params.albedoMap || null },
      uNormalMap: { value: params.normalMap || null },
      uRoughnessMap: { value: params.roughnessMap || null },
      uMetalnessMap: { value: params.metalnessMap || null },
      uUseTextures: { value: !!params.albedoMap }
    },
    vertexShader: gBufferWriteVert,
    fragmentShader: gBufferWriteFrag
  });
}
```

---

## 4. Deferred Lighting Shader

```javascript
// Deferred Lighting Pass Shader
const deferredLightingVert = `
  out vec2 vUv;
  void main() {
    vUv = uv;
    gl_Position = vec4(position, 1.0);
  }
`;

const deferredLightingFrag = `
  precision highp float;
  
  // G-Buffer textures
  uniform sampler2D gAlbedo;
  uniform sampler2D gNormal;
  uniform sampler2D gPosition;
  uniform sampler2D gMaterial;
  uniform sampler2D gDepth;
  
  // Camera
  uniform vec3 uCameraPos;
  uniform mat4 uProjectionMatrix;
  uniform mat4 uViewMatrix;
  
  // Lights
  const int MAX_LIGHTS = 64;
  uniform int uNumLights;
  uniform vec3 uLightPositions[MAX_LIGHTS];
  uniform vec3 uLightColors[MAX_LIGHTS];
  uniform float uLightRadii[MAX_LIGHTS];
  uniform float uLightIntensities[MAX_LIGHTS];
  
  // Ambient light
  uniform vec3 uAmbientColor;
  uniform float uAmbientIntensity;
  
  in vec2 vUv;
  out vec4 fragColor;
  
  // PBR Functions
  float distributionGGX(vec3 N, vec3 H, float roughness) {
    float a = roughness * roughness;
    float a2 = a * a;
    float NdotH = max(dot(N, H), 0.0);
    float NdotH2 = NdotH * NdotH;
    float denom = NdotH2 * (a2 - 1.0) + 1.0;
    return a2 / (3.14159 * denom * denom);
  }
  
  float geometrySchlickGGX(float NdotV, float roughness) {
    float r = roughness + 1.0;
    float k = (r * r) / 8.0;
    return NdotV / (NdotV * (1.0 - k) + k);
  }
  
  float geometrySmith(vec3 N, vec3 V, vec3 L, float roughness) {
    float NdotV = max(dot(N, V), 0.0);
    float NdotL = max(dot(N, L), 0.0);
    return geometrySchlickGGX(NdotV, roughness) * geometrySchlickGGX(NdotL, roughness);
  }
  
  vec3 fresnelSchlick(float cosTheta, vec3 F0) {
    return F0 + (1.0 - F0) * pow(clamp(1.0 - cosTheta, 0.0, 1.0), 5.0);
  }
  
  // Calculate point light contribution
  vec3 calculatePointLight(
    int lightIdx,
    vec3 worldPos,
    vec3 N,
    vec3 V,
    vec3 albedo,
    float roughness,
    float metalness
  ) {
    vec3 lightPos = uLightPositions[lightIdx];
    vec3 lightColor = uLightColors[lightIdx];
    float lightRadius = uLightRadii[lightIdx];
    float intensity = uLightIntensities[lightIdx];
    
    vec3 L = lightPos - worldPos;
    float distance = length(L);
    L = normalize(L);
    
    // Distance falloff
    float attenuation = 1.0 - smoothstep(0.0, lightRadius, distance);
    attenuation *= attenuation;
    
    if (attenuation < 0.001) return vec3(0.0);
    
    vec3 H = normalize(V + L);
    float NdotL = max(dot(N, L), 0.0);
    
    // PBR: Cook-Torrance BRDF
    vec3 F0 = mix(vec3(0.04), albedo, metalness);
    
    float NDF = distributionGGX(N, H, roughness);
    float G = geometrySmith(N, V, L, roughness);
    vec3 F = fresnelSchlick(max(dot(H, V), 0.0), F0);
    
    vec3 numerator = NDF * G * F;
    float denominator = 4.0 * max(dot(N, V), 0.0) * NdotL + 0.0001;
    vec3 specular = numerator / denominator;
    
    vec3 kS = F;
    vec3 kD = (1.0 - kS) * (1.0 - metalness);
    
    vec3 radiance = lightColor * intensity * attenuation;
    
    return (kD * albedo / 3.14159 + specular) * radiance * NdotL;
  }
  
  void main() {
    // อ่านจาก G-Buffer
    vec4 albedoData = texture(gAlbedo, vUv);
    vec4 normalData = texture(gNormal, vUv);
    vec4 positionData = texture(gPosition, vUv);
    vec4 materialData = texture(gMaterial, vUv);
    
    // Skip ถ้าไม่มี geometry (background)
    if (albedoData.a < 0.5) {
      fragColor = vec4(0.0, 0.0, 0.1, 1.0);  // background color
      return;
    }
    
    // Decode G-Buffer
    vec3 albedo = albedoData.rgb;
    vec3 N = normalize(normalData.rgb * 2.0 - 1.0);  // decode normal
    vec3 worldPos = positionData.rgb;
    float roughness = materialData.r;
    float metalness = materialData.g;
    float ao = materialData.b;
    
    vec3 V = normalize(uCameraPos - worldPos);
    
    // Accumulate lighting
    vec3 Lo = vec3(0.0);
    
    // Ambient
    vec3 ambient = uAmbientColor * uAmbientIntensity * albedo * ao;
    Lo += ambient;
    
    // Point lights
    for (int i = 0; i < MAX_LIGHTS; i++) {
      if (i >= uNumLights) break;
      Lo += calculatePointLight(i, worldPos, N, V, albedo, roughness, metalness);
    }
    
    // Tone mapping (simple Reinhard)
    Lo = Lo / (Lo + vec3(1.0));
    
    // Gamma correction
    Lo = pow(Lo, vec3(1.0 / 2.2));
    
    fragColor = vec4(Lo, 1.0);
  }
`;
```

---

## 5. G-Buffer Visualization Tool

```javascript
class GBufferVisualizer {
  constructor(gbuffer) {
    this.gbuffer = gbuffer;
    this.mode = 'final';  // 'final', 'albedo', 'normal', 'position', 'material'
    
    this.setupVisualizationQuad();
  }
  
  setupVisualizationQuad() {
    this.quad = new THREE.Mesh(
      new THREE.PlaneGeometry(2, 2),
      new THREE.ShaderMaterial({
        uniforms: {
          tDiffuse: { value: null },
          uMode: { value: 0 }
        },
        vertexShader: `
          varying vec2 vUv;
          void main() { vUv = uv; gl_Position = vec4(position, 1.0); }
        `,
        fragmentShader: `
          uniform sampler2D tDiffuse;
          uniform int uMode;
          varying vec2 vUv;
          
          void main() {
            vec4 tex = texture2D(tDiffuse, vUv);
            
            if (uMode == 1) {
              // Normal visualization
              gl_FragColor = vec4(tex.rgb * 0.5 + 0.5, 1.0);
            } else if (uMode == 2) {
              // Depth visualization
              float depth = tex.r;
              gl_FragColor = vec4(vec3(depth), 1.0);
            } else {
              gl_FragColor = vec4(tex.rgb, 1.0);
            }
          }
        `
      })
    );
  }
  
  renderDebug(renderer, camera) {
    // แสดง 4 quadrants
    // ต้องการ orthographic camera
    const w = this.gbuffer.width;
    const h = this.gbuffer.height;
    const hw = w / 2;
    const hh = h / 2;
    
    // Viewport สำหรับแต่ละ quadrant
    const viewports = [
      { x: 0, y: hh, w: hw, h: hh, tex: this.gbuffer.albedoTexture, label: 'Albedo' },
      { x: hw, y: hh, w: hw, h: hh, tex: this.gbuffer.normalTexture, label: 'Normal' },
      { x: 0, y: 0, w: hw, h: hh, tex: this.gbuffer.positionTexture, label: 'Position' },
      { x: hw, y: 0, w: hw, h: hh, tex: this.gbuffer.materialTexture, label: 'Material' }
    ];
    
    const orthoScene = new THREE.Scene();
    const orthoCam = new THREE.OrthographicCamera(-1, 1, 1, -1, 0, 1);
    orthoScene.add(this.quad);
    
    for (const vp of viewports) {
      renderer.setViewport(vp.x, vp.y, vp.w, vp.h);
      this.quad.material.uniforms.tDiffuse.value = vp.tex;
      renderer.render(orthoScene, orthoCam);
    }
    
    // Reset viewport
    renderer.setViewport(0, 0, w, h);
  }
}
```

---

## 6. Complete Deferred Rendering Example

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title>Deferred Rendering Demo - Many Lights</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { overflow: hidden; background: #000; font-family: monospace; }
    canvas { display: block; }
    #info {
      position: fixed;
      top: 10px;
      left: 10px;
      background: rgba(0,0,0,0.7);
      color: #0f0;
      padding: 10px;
      border-radius: 5px;
      font-size: 12px;
    }
    #controls {
      position: fixed;
      top: 10px;
      right: 10px;
      background: rgba(0,0,0,0.7);
      color: white;
      padding: 10px;
      border-radius: 5px;
      font-size: 12px;
    }
    button {
      display: block;
      width: 100%;
      padding: 5px;
      margin: 3px 0;
      background: #334;
      border: 1px solid #556;
      color: white;
      cursor: pointer;
      border-radius: 3px;
    }
    button.active { background: #446; border-color: #88f; }
  </style>
</head>
<body>
  <canvas id="canvas"></canvas>
  <div id="info">
    <div id="fps">FPS: --</div>
    <div id="lightCount">Lights: --</div>
    <div id="renderMode">Mode: Forward</div>
  </div>
  <div id="controls">
    <div style="margin-bottom: 5px; font-weight: bold;">Render Mode</div>
    <button id="btnForward" class="active">Forward Rendering</button>
    <button id="btnDeferred">Deferred Rendering</button>
    <hr style="border-color: #334; margin: 8px 0">
    <div style="margin-bottom: 5px; font-weight: bold;">Light Count</div>
    <button id="btn10">10 Lights</button>
    <button id="btn50">50 Lights</button>
    <button id="btn100" class="active">100 Lights</button>
    <hr style="border-color: #334; margin: 8px 0">
    <div style="margin-bottom: 5px; font-weight: bold;">G-Buffer View</div>
    <button id="btnFinal" class="active">Final</button>
    <button id="btnAlbedo">Albedo</button>
    <button id="btnNormal">Normal</button>
    <button id="btnPosition">Position</button>
    <button id="btnMaterial">Material</button>
  </div>

  <script type="module">
    import * as THREE from 'https://cdn.skypack.dev/three@0.160.0';
    import { OrbitControls } from 'https://cdn.skypack.dev/three@0.160.0/examples/jsm/controls/OrbitControls.js';
    
    const canvas = document.getElementById('canvas');
    const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.outputColorSpace = THREE.SRGBColorSpace;
    
    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
    camera.position.set(0, 8, 15);
    
    const controls = new OrbitControls(camera, canvas);
    controls.enableDamping = true;
    controls.target.set(0, 0, 0);
    
    // สร้าง scene objects
    function buildScene(scene) {
      // Floor
      const floorGeo = new THREE.PlaneGeometry(40, 40, 20, 20);
      const floorMat = new THREE.MeshStandardMaterial({ 
        color: 0x333344, roughness: 0.8, metalness: 0.0 
      });
      const floor = new THREE.Mesh(floorGeo, floorMat);
      floor.rotation.x = -Math.PI / 2;
      floor.receiveShadow = true;
      scene.add(floor);
      
      // สร้าง pillars
      const pillarColors = [0x884422, 0x228844, 0x224488, 0x884488, 0x448844];
      for (let i = 0; i < 20; i++) {
        const angle = (i / 20) * Math.PI * 2;
        const r = 6;
        
        const pillarGeo = new THREE.BoxGeometry(0.8, 4, 0.8);
        const pillarMat = new THREE.MeshStandardMaterial({
          color: pillarColors[i % pillarColors.length],
          roughness: 0.5,
          metalness: 0.3
        });
        const pillar = new THREE.Mesh(pillarGeo, pillarMat);
        pillar.position.set(Math.cos(angle) * r, 2, Math.sin(angle) * r);
        pillar.castShadow = true;
        pillar.receiveShadow = true;
        scene.add(pillar);
      }
      
      // Central object
      const centerGeo = new THREE.IcosahedronGeometry(1.5, 2);
      const centerMat = new THREE.MeshStandardMaterial({
        color: 0xffffff,
        roughness: 0.1,
        metalness: 0.9
      });
      const center = new THREE.Mesh(centerGeo, centerMat);
      center.position.y = 2;
      center.castShadow = true;
      scene.add(center);
      
      return { center };
    }
    
    // สร้าง lights
    function createLights(scene, count) {
      const lights = [];
      const hues = [];
      
      for (let i = 0; i < count; i++) {
        const hue = (i / count);
        hues.push(hue);
        
        const color = new THREE.Color().setHSL(hue, 1.0, 0.5);
        const light = new THREE.PointLight(color, 2, 8);
        
        const angle = (i / count) * Math.PI * 2;
        const r = 3 + Math.random() * 5;
        light.position.set(
          Math.cos(angle) * r,
          0.5 + Math.random() * 3,
          Math.sin(angle) * r
        );
        
        lights.push({ light, angle, r, speed: 0.2 + Math.random() * 0.5, hue, y: light.position.y });
        scene.add(light);
        
        // Visual indicator
        const indicatorGeo = new THREE.SphereGeometry(0.1, 8, 8);
        const indicatorMat = new THREE.MeshBasicMaterial({ color });
        const indicator = new THREE.Mesh(indicatorGeo, indicatorMat);
        light.add(indicator);
      }
      
      return lights;
    }
    
    const { center } = buildScene(scene);
    scene.add(new THREE.AmbientLight(0x112233, 0.2));
    
    let lights = createLights(scene, 100);
    let numLights = 100;
    
    // FPS counter
    let frameCount = 0;
    let lastTime = performance.now();
    let fps = 0;
    
    // Resize
    window.addEventListener('resize', () => {
      const w = window.innerWidth, h = window.innerHeight;
      renderer.setSize(w, h);
      camera.aspect = w / h;
      camera.updateProjectionMatrix();
    });
    
    // Light count buttons
    function setLightCount(count) {
      // Remove old lights
      lights.forEach(({ light }) => {
        scene.remove(light);
        light.dispose?.();
      });
      
      lights = createLights(scene, count);
      numLights = count;
      
      document.querySelectorAll('[id^="btn"]:not([id^="btnForward"]):not([id^="btnDeferred"]):not([id^="btnFinal"]):not([id^="btnAlbedo"]):not([id^="btnNormal"]):not([id^="btnPosition"]):not([id^="btnMaterial"])').forEach(b => b.classList.remove('active'));
    }
    
    document.getElementById('btn10').onclick = () => { setLightCount(10); document.getElementById('btn10').classList.add('active'); };
    document.getElementById('btn50').onclick = () => { setLightCount(50); document.getElementById('btn50').classList.add('active'); };
    document.getElementById('btn100').onclick = () => { setLightCount(100); document.getElementById('btn100').classList.add('active'); };
    
    // Animation
    const clock = new THREE.Clock();
    
    function animate() {
      requestAnimationFrame(animate);
      
      const time = clock.getElapsedTime();
      const delta = clock.getDelta();
      
      // FPS
      frameCount++;
      const now = performance.now();
      if (now - lastTime >= 1000) {
        fps = Math.round(frameCount * 1000 / (now - lastTime));
        document.getElementById('fps').textContent = `FPS: ${fps}`;
        document.getElementById('lightCount').textContent = `Lights: ${numLights}`;
        frameCount = 0;
        lastTime = now;
      }
      
      // Animate lights
      lights.forEach(({ light, angle, r, speed, hue, y }, i) => {
        const a = angle + time * speed;
        light.position.x = Math.cos(a) * r;
        light.position.z = Math.sin(a) * r;
        light.position.y = y + Math.sin(time * 2 + i) * 0.5;
        
        // Animate color
        const newHue = (hue + time * 0.05) % 1;
        light.color.setHSL(newHue, 1.0, 0.5);
      });
      
      center.rotation.y = time * 0.5;
      center.rotation.x = Math.sin(time * 0.3) * 0.3;
      
      controls.update();
      renderer.render(scene, camera);
    }
    
    animate();
  </script>
</body>
</html>
```

---

## 7. Tiled Deferred Rendering Concept

```javascript
// แนวคิด Tiled Deferred Rendering
// แบ่ง screen เป็น tiles แล้วหา lights ที่ส่งผลต่อแต่ละ tile

class TiledDeferredRenderer {
  constructor(renderer, tileSize = 32) {
    this.renderer = renderer;
    this.tileSize = tileSize;
  }
  
  // คำนวณว่า light ส่งผลต่อ tile ไหนบ้าง
  computeLightTiles(lights, camera) {
    const width = this.renderer.domElement.width;
    const height = this.renderer.domElement.height;
    const tilesX = Math.ceil(width / this.tileSize);
    const tilesY = Math.ceil(height / this.tileSize);
    
    // แต่ละ tile เก็บ list ของ lights
    const tileLights = Array.from({ length: tilesX * tilesY }, () => []);
    
    // Project light sphere ลงบน screen
    for (let i = 0; i < lights.length; i++) {
      const light = lights[i];
      const lightPosClip = light.position.clone().project(camera);
      
      // Convert to pixel coordinates
      const screenX = (lightPosClip.x + 1) * 0.5 * width;
      const screenY = (1 - lightPosClip.y) * 0.5 * height;
      
      // คำนวณรัศมีบน screen
      const screenRadius = this.projectRadius(light.position, light.distance, camera);
      
      // หา tiles ที่ overlaps กับ light sphere
      const tileMinX = Math.floor(Math.max(0, screenX - screenRadius) / this.tileSize);
      const tileMaxX = Math.ceil(Math.min(width, screenX + screenRadius) / this.tileSize);
      const tileMinY = Math.floor(Math.max(0, screenY - screenRadius) / this.tileSize);
      const tileMaxY = Math.ceil(Math.min(height, screenY + screenRadius) / this.tileSize);
      
      for (let ty = tileMinY; ty < tileMaxY; ty++) {
        for (let tx = tileMinX; tx < tileMaxX; tx++) {
          const tileIdx = ty * tilesX + tx;
          if (tileIdx < tileLights.length) {
            tileLights[tileIdx].push(i);
          }
        }
      }
    }
    
    return { tileLights, tilesX, tilesY };
  }
  
  projectRadius(position, radius, camera) {
    // Project radius onto screen space (simplified)
    const distance = position.distanceTo(camera.position);
    const fov = camera.fov * Math.PI / 180;
    const height = this.renderer.domElement.height;
    return (radius / distance) * height / Math.tan(fov / 2) * 0.5;
  }
}
```

---

## 8. Forward vs Deferred เปรียบเทียบ

| Aspect | Forward | Deferred |
|--------|---------|----------|
| Lights จำนวนมาก | ช้า (O(O×L)) | เร็วกว่า (O(L+P)) |
| Transparency | ง่าย | ยาก (ต้องทำ forward pass แยก) |
| MSAA | ได้ผล | ไม่ทำงาน (ต้องใช้ SMAA/FXAA) |
| Memory | น้อย | มาก (G-Buffer) |
| Mobile | เหมาะสม | ไม่เหมาะ (bandwidth สูง) |
| Custom Shading | ง่าย | ซับซ้อน |
| Shadow | ปกติ | ต้องทำแยก |

### เมื่อไหร่ควรใช้ Deferred?
- มี point lights มากกว่า 10+ ที่เคลื่อนที่
- PC/console target (bandwidth ไม่จำกัด)
- Complex lighting environments

### เมื่อไหร่ควรใช้ Forward?
- Mobile games
- ต้องการ transparency
- Lights น้อย
- Simple shading

ในบทถัดไป (Part 85) เราจะเรียนรู้เกี่ยวกับ Game Architecture ด้วย Three.js
