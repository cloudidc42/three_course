# Part 48: Bloom, Lens Flare & Glow Effects ใน Three.js

## บทนำ

Post-processing effects ทำให้ภาพสวยงามและสมจริงยิ่งขึ้น บทนี้ครอบคลุม:
- **UnrealBloomPass** - Bloom effect แบบ Unreal Engine
- **Selective Bloom** - เลือก object ที่จะ bloom
- **LensflareElement** - Lens flare 3D
- **Glow Shader** - custom two-pass glow
- **Neon Sign Effect** - ป้ายนีออน
- **God Rays / Volumetric Light** - แสงจากท้องฟ้า

## 1. Basic Bloom Setup

```javascript
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
import { ShaderPass } from 'three/addons/postprocessing/ShaderPass.js';

// ===== Basic Bloom =====
function setupBloom(renderer, scene, camera) {
  const composer = new EffectComposer(renderer);
  
  // Step 1: Render scene normally
  const renderPass = new RenderPass(scene, camera);
  composer.addPass(renderPass);
  
  // Step 2: Apply bloom
  const bloomPass = new UnrealBloomPass(
    new THREE.Vector2(window.innerWidth, window.innerHeight),
    1.5,   // strength  - ความเข้มของ bloom (0=none, 3=heavy)
    0.4,   // radius    - ขนาดของ blur (0=sharp, 1=large)
    0.85   // threshold - luminance threshold (0=all glows, 1=only bright)
  );
  
  composer.addPass(bloomPass);
  
  return { composer, bloomPass };
}

// ใช้งาน
const { composer, bloomPass } = setupBloom(renderer, scene, camera);

// ใน render loop
function animate() {
  requestAnimationFrame(animate);
  // ใช้ composer.render() แทน renderer.render()
  composer.render();
}
```

## 2. Selective Bloom (เฉพาะบาง Object)

```javascript
// ===== Selective Bloom =====
// เทคนิค: render bloom layer แยก, ใช้ layers

const ENTIRE_SCENE = 0;
const BLOOM_SCENE = 1;

class SelectiveBloom {
  constructor(renderer, scene, camera) {
    this.renderer = renderer;
    this.scene = scene;
    this.camera = camera;
    this.darkMaterials = {};
    
    this._setup();
  }
  
  _setup() {
    const { width, height } = this.renderer.getSize(new THREE.Vector2());
    
    // Bloom composer - renders only bloom objects
    const renderTarget = new THREE.WebGLRenderTarget(width, height, {
      minFilter: THREE.LinearFilter,
      magFilter: THREE.LinearFilter,
      format: THREE.RGBAFormat,
      encoding: THREE.sRGBEncoding
    });
    
    this.bloomComposer = new EffectComposer(this.renderer, renderTarget);
    this.bloomComposer.renderToScreen = false;
    
    this.bloomComposer.addPass(new RenderPass(this.scene, this.camera));
    
    this.bloomPass = new UnrealBloomPass(
      new THREE.Vector2(width, height), 1.5, 0.4, 0
    );
    this.bloomComposer.addPass(this.bloomPass);
    
    // Final composer - combines bloom + normal scene
    this.finalComposer = new EffectComposer(this.renderer);
    this.finalComposer.addPass(new RenderPass(this.scene, this.camera));
    
    // Mix pass
    const finalPass = new ShaderPass(
      new THREE.ShaderMaterial({
        uniforms: {
          baseTexture: { value: null },
          bloomTexture: { value: this.bloomComposer.renderTarget2.texture }
        },
        vertexShader: `
          varying vec2 vUv;
          void main() {
            vUv = uv;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
          }
        `,
        fragmentShader: `
          uniform sampler2D baseTexture;
          uniform sampler2D bloomTexture;
          varying vec2 vUv;
          void main() {
            gl_FragColor = texture2D(baseTexture, vUv) + vec4(texture2D(bloomTexture, vUv).rgb, 0);
          }
        `,
        defines: {}
      }),
      'baseTexture'
    );
    finalPass.needsSwap = true;
    this.finalComposer.addPass(finalPass);
  }
  
  // Set material สำหรับ object ที่จะ glow
  setBloomLayer(mesh, bloom = true) {
    if (bloom) {
      mesh.layers.enable(BLOOM_SCENE);
    } else {
      mesh.layers.disable(BLOOM_SCENE);
    }
  }
  
  // Darken non-bloom objects
  _darkenNonBloomed() {
    this.scene.traverse(obj => {
      if (obj.isMesh && obj.layers.test(new THREE.Layers()) === false) {
        if (!obj.layers.isEnabled(BLOOM_SCENE)) {
          this.darkMaterials[obj.uuid] = obj.material;
          obj.material = new THREE.MeshBasicMaterial({ color: 0x000000 });
        }
      }
    });
  }
  
  _restoreMaterials() {
    this.scene.traverse(obj => {
      if (obj.isMesh && this.darkMaterials[obj.uuid]) {
        obj.material = this.darkMaterials[obj.uuid];
        delete this.darkMaterials[obj.uuid];
      }
    });
  }
  
  render() {
    // 1. Render bloom objects only
    this.camera.layers.set(BLOOM_SCENE);
    this._darkenNonBloomed();
    this.bloomComposer.render();
    this._restoreMaterials();
    
    // 2. Render everything, composite with bloom
    this.camera.layers.set(ENTIRE_SCENE);
    this.finalComposer.render();
  }
}
```

## 3. Lens Flare

```javascript
import { Lensflare, LensflareElement } from 'three/addons/objects/Lensflare.js';

// ===== Lens Flare Setup =====
function createLensFlare(color = new THREE.Color(0xffffff)) {
  // สร้าง lens flare textures บน canvas
  const tex0 = createFlareTexture(64, 'round');    // ดวงหลัก
  const tex1 = createFlareTexture(32, 'star');     // ดาว
  const tex2 = createFlareTexture(16, 'hexagon');  // hexagon
  const tex3 = createFlareTexture(8,  'ring');     // ring
  
  const lensflare = new Lensflare();
  
  // addElement(texture, size, distance, color)
  // distance: 0 = at source, 1 = opposite side of screen
  lensflare.addElement(new LensflareElement(tex0, 700, 0, color));
  lensflare.addElement(new LensflareElement(tex1, 60,  0.6));
  lensflare.addElement(new LensflareElement(tex2, 70,  0.7));
  lensflare.addElement(new LensflareElement(tex3, 120, 0.9));
  lensflare.addElement(new LensflareElement(tex2, 70,  1.0));
  
  return lensflare;
}

function createFlareTexture(size, type = 'round') {
  const canvas = document.createElement('canvas');
  canvas.width = canvas.height = size;
  const ctx = canvas.getContext('2d');
  const c = size / 2;
  
  if (type === 'round') {
    const g = ctx.createRadialGradient(c, c, 0, c, c, c);
    g.addColorStop(0,   'rgba(255, 255, 255, 1)');
    g.addColorStop(0.1, 'rgba(255, 240, 200, 0.8)');
    g.addColorStop(0.4, 'rgba(255, 200, 100, 0.3)');
    g.addColorStop(1,   'rgba(255, 150, 50, 0)');
    ctx.fillStyle = g;
    ctx.fillRect(0, 0, size, size);
    
  } else if (type === 'star') {
    ctx.strokeStyle = 'rgba(255, 255, 255, 0.8)';
    ctx.lineWidth = 1;
    for (let a = 0; a < 8; a++) {
      const angle = (a / 8) * Math.PI * 2;
      ctx.beginPath();
      ctx.moveTo(c, c);
      ctx.lineTo(c + Math.cos(angle) * c, c + Math.sin(angle) * c);
      ctx.stroke();
    }
    
  } else if (type === 'hexagon') {
    ctx.strokeStyle = 'rgba(200, 220, 255, 0.6)';
    ctx.lineWidth = 2;
    ctx.beginPath();
    for (let i = 0; i < 6; i++) {
      const angle = (i / 6) * Math.PI * 2;
      const x = c + Math.cos(angle) * (c - 2);
      const y = c + Math.sin(angle) * (c - 2);
      i === 0 ? ctx.moveTo(x, y) : ctx.lineTo(x, y);
    }
    ctx.closePath();
    ctx.stroke();
    
  } else if (type === 'ring') {
    ctx.strokeStyle = 'rgba(255, 255, 255, 0.5)';
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.arc(c, c, c - 2, 0, Math.PI * 2);
    ctx.stroke();
  }
  
  return new THREE.CanvasTexture(canvas);
}

// ===== ใช้งาน =====
const sunLight = new THREE.PointLight(0xffffff, 2, 2000);
sunLight.position.set(50, 100, 30);

const lensflare = createLensFlare(new THREE.Color(0xffeedd));
sunLight.add(lensflare);

scene.add(sunLight);
```

## 4. Neon Sign Effect

```javascript
// ===== Neon Sign =====
class NeonSign {
  constructor(scene, text, color = 0xff0088) {
    this.scene = scene;
    this.color = color;
    this.group = new THREE.Group();
    
    this._createSign(text);
    scene.add(this.group);
  }
  
  _createSign(text) {
    // Tube neon ใช้ CatmullRomCurve3
    const colorObj = new THREE.Color(this.color);
    
    // Material เรืองแสง
    const neonMat = new THREE.MeshStandardMaterial({
      color: colorObj,
      emissive: colorObj,
      emissiveIntensity: 3.0,
      roughness: 0.3,
      metalness: 0.5
    });
    
    // สร้าง tube จาก curve
    const addTube = (points) => {
      const curve = new THREE.CatmullRomCurve3(points);
      const geo = new THREE.TubeGeometry(curve, 20, 0.05, 8, false);
      const mesh = new THREE.Mesh(geo, neonMat);
      this.group.add(mesh);
    };
    
    // ตัวอักษร N แบบ manual
    addTube([
      new THREE.Vector3(-0.3, -0.5, 0),
      new THREE.Vector3(-0.3, 0.5, 0)
    ]);
    addTube([
      new THREE.Vector3(-0.3, 0.5, 0),
      new THREE.Vector3(0.3, -0.5, 0)
    ]);
    addTube([
      new THREE.Vector3(0.3, -0.5, 0),
      new THREE.Vector3(0.3, 0.5, 0)
    ]);
    
    // Point light สำหรับ glow บนพื้นผิวรอบข้าง
    const light = new THREE.PointLight(this.color, 2, 5);
    light.position.set(0, 0, 0.2);
    this.group.add(light);
    
    this.light = light;
    this.neonMat = neonMat;
  }
  
  // Flicker effect
  flicker(time) {
    const flicker = Math.random() > 0.98 ? 0 : 1;
    const pulse = Math.sin(time * 60) * 0.05 + 0.95;
    this.neonMat.emissiveIntensity = flicker * pulse * 3.0;
    this.light.intensity = flicker * pulse * 2.0;
  }
  
  setPosition(x, y, z) {
    this.group.position.set(x, y, z);
  }
}
```

## 5. God Rays (Volumetric Light Shaft)

```javascript
// ===== God Rays (Screen-Space) =====
// เทคนิค: render occluders เป็น silhouette, radial blur จาก sun position

const godRaysShader = {
  uniforms: {
    tDiffuse: { value: null },
    lightPos: { value: new THREE.Vector2(0.5, 0.5) }, // NDC 0-1
    exposure: { value: 0.3 },
    decay: { value: 0.95 },
    density: { value: 0.8 },
    weight: { value: 0.5 },
    samples: { value: 100 }
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
    uniform vec2 lightPos;
    uniform float exposure;
    uniform float decay;
    uniform float density;
    uniform float weight;
    varying vec2 vUv;
    
    const int NUM_SAMPLES = 100;
    
    void main() {
      vec2 texCoord = vUv;
      vec2 deltaTexCoord = texCoord - lightPos;
      deltaTexCoord *= 1.0 / float(NUM_SAMPLES) * density;
      
      vec4 color = texture2D(tDiffuse, texCoord);
      float illuminationDecay = 1.0;
      
      for (int i = 0; i < NUM_SAMPLES; i++) {
        texCoord -= deltaTexCoord;
        vec4 sample = texture2D(tDiffuse, texCoord);
        sample *= illuminationDecay * weight;
        color += sample;
        illuminationDecay *= decay;
      }
      
      gl_FragColor = color * exposure;
    }
  `
};
```

## 6. Complete Demo: Neon City

```html
<!DOCTYPE html>
<html>
<head>
  <title>Neon City - Bloom & VFX</title>
  <style>
    * { margin: 0; padding: 0; }
    body { background: #000; overflow: hidden; }
    canvas { display: block; }
    #controls {
      position: fixed;
      top: 20px;
      right: 20px;
      background: rgba(0,0,20,0.85);
      color: #adf;
      padding: 15px;
      border-radius: 8px;
      font-family: monospace;
      font-size: 13px;
      min-width: 200px;
    }
    #controls label { display: block; margin: 6px 0; }
    #controls input[type=range] { width: 130px; vertical-align: middle; }
  </style>
</head>
<body>
<div id="controls">
  <b>Bloom Controls</b><br><br>
  <label>Strength: <input type="range" id="bStrength" min="0" max="3" step="0.05" value="1.5"><span id="bStrVal">1.5</span></label>
  <label>Radius: <input type="range" id="bRadius" min="0" max="1" step="0.05" value="0.4"><span id="bRadVal">0.4</span></label>
  <label>Threshold: <input type="range" id="bThresh" min="0" max="1" step="0.05" value="0.2"><span id="bThrVal">0.2</span></label>
  <label>Exposure: <input type="range" id="exposure" min="0.1" max="3" step="0.1" value="1"><span id="expVal">1.0</span></label>
  <br><label>Flicker: <input type="checkbox" id="flicker" checked></label>
</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';
import { EffectComposer } from 'https://unpkg.com/three@0.158.0/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'https://unpkg.com/three@0.158.0/examples/jsm/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'https://unpkg.com/three@0.158.0/examples/jsm/postprocessing/UnrealBloomPass.js';
import { OutputPass } from 'https://unpkg.com/three@0.158.0/examples/jsm/postprocessing/OutputPass.js';
import { Lensflare, LensflareElement } from 'https://unpkg.com/three@0.158.0/examples/jsm/objects/Lensflare.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x000008);
scene.fog = new THREE.FogExp2(0x000008, 0.02);

const camera = new THREE.PerspectiveCamera(75, innerWidth/innerHeight, 0.1, 300);
camera.position.set(0, 8, 20);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
renderer.outputColorSpace = THREE.SRGBColorSpace;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.0;
renderer.shadowMap.enabled = true;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.maxPolarAngle = Math.PI * 0.6;

// ===== POST PROCESSING =====
const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));

const bloomPass = new UnrealBloomPass(
  new THREE.Vector2(innerWidth, innerHeight),
  1.5, 0.4, 0.2
);
composer.addPass(bloomPass);
composer.addPass(new OutputPass());

// ===== GROUND =====
const groundMat = new THREE.MeshStandardMaterial({
  color: 0x111122, roughness: 0.8, metalness: 0.2
});
const ground = new THREE.Mesh(new THREE.PlaneGeometry(100, 100), groundMat);
ground.rotateX(-Math.PI / 2);
ground.receiveShadow = true;
scene.add(ground);

// Reflective puddles
for (let i = 0; i < 15; i++) {
  const puddle = new THREE.Mesh(
    new THREE.CircleGeometry(0.5 + Math.random(), 16),
    new THREE.MeshStandardMaterial({
      color: 0x001122, roughness: 0.1, metalness: 0.8, envMapIntensity: 2
    })
  );
  puddle.rotateX(-Math.PI / 2);
  puddle.position.set((Math.random()-0.5)*30, 0.01, (Math.random()-0.5)*30);
  scene.add(puddle);
}

// ===== NEON TUBE HELPER =====
function createNeonTube(points, color, radius = 0.06) {
  const curve = new THREE.CatmullRomCurve3(points);
  const geo = new THREE.TubeGeometry(curve, points.length * 5, radius, 8, false);
  const mat = new THREE.MeshStandardMaterial({
    color, emissive: color, emissiveIntensity: 3, roughness: 0.3
  });
  const mesh = new THREE.Mesh(geo, mat);
  mesh.castShadow = false;
  return { mesh, mat };
}

function addNeonLight(x, y, z, color, intensity = 3, dist = 8) {
  const light = new THREE.PointLight(color, intensity, dist);
  light.position.set(x, y, z);
  scene.add(light);
  return light;
}

const neonObjects = [];

// ===== BUILDINGS =====
const buildingColors = [0xff0088, 0x00ffaa, 0x0088ff, 0xff8800, 0xaa00ff];

for (let i = 0; i < 20; i++) {
  const w = 1.5 + Math.random() * 2.5;
  const h = 3 + Math.random() * 12;
  const d = 1.5 + Math.random() * 2.5;
  const x = (Math.random() - 0.5) * 50;
  const z = -5 - Math.random() * 30;
  
  // Building body
  const building = new THREE.Mesh(
    new THREE.BoxGeometry(w, h, d),
    new THREE.MeshStandardMaterial({ color: 0x112233, roughness: 0.8 })
  );
  building.position.set(x, h/2, z);
  building.castShadow = true;
  building.receiveShadow = true;
  scene.add(building);
  
  // Windows (emissive)
  const winColor = new THREE.Color().setHSL(Math.random(), 0.8, 0.7);
  const winMat = new THREE.MeshStandardMaterial({
    color: winColor, emissive: winColor, emissiveIntensity: 0.5 + Math.random() * 1
  });
  
  for (let wy = 1; wy < h - 1; wy += 1.2) {
    for (let wx = 0; wx < 2; wx++) {
      const win = new THREE.Mesh(new THREE.BoxGeometry(0.3, 0.4, 0.05), winMat);
      win.position.set(x + (wx - 0.5) * w * 0.5, wy, z + d/2 + 0.01);
      scene.add(win);
      neonObjects.push({ mesh: win, mat: winMat, baseIntensity: winMat.emissiveIntensity });
    }
  }
  
  // Neon sign on building
  if (Math.random() > 0.5) {
    const nColor = buildingColors[Math.floor(Math.random() * buildingColors.length)];
    const yPos = h * 0.7 + h/2;
    
    // Horizontal bar
    const bar = createNeonTube([
      new THREE.Vector3(x - w*0.3, yPos, z + d/2 + 0.2),
      new THREE.Vector3(x + w*0.3, yPos, z + d/2 + 0.2)
    ], nColor);
    scene.add(bar.mesh);
    neonObjects.push({ mesh: bar.mesh, mat: bar.mat, baseIntensity: 3 });
    
    addNeonLight(x, yPos, z + d/2 + 0.5, nColor, 2, 6);
  }
}

// ===== NEON SIGNS =====
function createLetterO(cx, cy, cz, color) {
  const pts = [];
  for (let i = 0; i <= 16; i++) {
    const a = (i / 16) * Math.PI * 2;
    pts.push(new THREE.Vector3(cx + Math.cos(a) * 0.5, cy + Math.sin(a) * 0.5, cz));
  }
  const tube = createNeonTube(pts, color, 0.05);
  scene.add(tube.mesh);
  neonObjects.push({ mesh: tube.mesh, mat: tube.mat, baseIntensity: 3 });
  addNeonLight(cx, cy, cz + 0.3, color, 2, 5);
}

function createLetterN(cx, cy, cz, color) {
  const left = createNeonTube([
    new THREE.Vector3(cx - 0.3, cy - 0.6, cz),
    new THREE.Vector3(cx - 0.3, cy + 0.6, cz)
  ], color, 0.05);
  const diag = createNeonTube([
    new THREE.Vector3(cx - 0.3, cy + 0.6, cz),
    new THREE.Vector3(cx + 0.3, cy - 0.6, cz)
  ], color, 0.05);
  const right = createNeonTube([
    new THREE.Vector3(cx + 0.3, cy - 0.6, cz),
    new THREE.Vector3(cx + 0.3, cy + 0.6, cz)
  ], color, 0.05);
  
  [left, diag, right].forEach(t => {
    scene.add(t.mesh);
    neonObjects.push({ mesh: t.mesh, mat: t.mat, baseIntensity: 3 });
  });
  addNeonLight(cx, cy, cz + 0.3, color, 1.5, 4);
}

// "NEON" sign
createLetterN(-3, 4, 5, 0xff00ff);
createLetterO(-1.5, 4, 5, 0xff00ff);
createLetterO(0, 4, 5, 0xff00ff);
createLetterN(1.5, 4, 5, 0xff00ff);

// Board behind sign
const board = new THREE.Mesh(
  new THREE.BoxGeometry(6, 2.5, 0.1),
  new THREE.MeshStandardMaterial({ color: 0x110011, roughness: 0.9 })
);
board.position.set(-0.75, 4, 4.9);
scene.add(board);
addNeonLight(-0.75, 4, 5.5, 0xff00ff, 3, 10);

// Street lights
for (let x = -20; x <= 20; x += 8) {
  const pole = new THREE.Mesh(
    new THREE.CylinderGeometry(0.1, 0.1, 8, 8),
    new THREE.MeshStandardMaterial({ color: 0x333344 })
  );
  pole.position.set(x, 4, 2);
  scene.add(pole);
  
  const arm = new THREE.Mesh(
    new THREE.CylinderGeometry(0.06, 0.06, 2, 6),
    new THREE.MeshStandardMaterial({ color: 0x333344 })
  );
  arm.rotation.z = Math.PI / 2;
  arm.position.set(x + 1, 8, 2);
  scene.add(arm);
  
  const bulb = new THREE.Mesh(
    new THREE.SphereGeometry(0.2, 8, 8),
    new THREE.MeshStandardMaterial({ color: 0xffee88, emissive: 0xffee88, emissiveIntensity: 3 })
  );
  bulb.position.set(x + 2, 8, 2);
  scene.add(bulb);
  
  const streetLight = new THREE.PointLight(0xffee88, 2, 12);
  streetLight.position.set(x + 2, 7.5, 2);
  streetLight.castShadow = true;
  scene.add(streetLight);
  
  // Lens flare on street lights
  const flareColor = new THREE.Color(0xffee88);
  const flare = new Lensflare();
  const flareTex = createFlareTex();
  flare.addElement(new LensflareElement(flareTex, 100, 0, flareColor));
  flare.addElement(new LensflareElement(flareTex, 20, 0.3, flareColor));
  flare.addElement(new LensflareElement(flareTex, 30, 0.6, flareColor));
  streetLight.add(flare);
}

function createFlareTex() {
  const canvas = document.createElement('canvas');
  canvas.width = canvas.height = 64;
  const ctx = canvas.getContext('2d');
  const g = ctx.createRadialGradient(32, 32, 0, 32, 32, 32);
  g.addColorStop(0, 'rgba(255,255,255,1)');
  g.addColorStop(0.2, 'rgba(255,240,180,0.8)');
  g.addColorStop(1, 'rgba(0,0,0,0)');
  ctx.fillStyle = g;
  ctx.fillRect(0, 0, 64, 64);
  return new THREE.CanvasTexture(canvas);
}

// Ambient
scene.add(new THREE.AmbientLight(0x112244, 1));

// ===== RAIN =====
const rainCount = 2000;
const rainGeo = new THREE.BufferGeometry();
const rainPos = new Float32Array(rainCount * 3);
for (let i = 0; i < rainCount; i++) {
  rainPos[i*3] = (Math.random()-0.5)*60;
  rainPos[i*3+1] = Math.random()*30;
  rainPos[i*3+2] = (Math.random()-0.5)*60;
}
rainGeo.setAttribute('position', new THREE.Float32BufferAttribute(rainPos, 3));
const rainMat = new THREE.PointsMaterial({ color: 0x5599bb, size: 0.05, transparent: true, opacity: 0.4 });
const rain = new THREE.Points(rainGeo, rainMat);
scene.add(rain);

// ===== CONTROLS UI =====
const bStrength = document.getElementById('bStrength');
const bRadius = document.getElementById('bRadius');
const bThresh = document.getElementById('bThresh');
const exposureSlider = document.getElementById('exposure');
const flickerCheck = document.getElementById('flicker');

bStrength.addEventListener('input', e => {
  bloomPass.strength = parseFloat(e.target.value);
  document.getElementById('bStrVal').textContent = parseFloat(e.target.value).toFixed(2);
});
bRadius.addEventListener('input', e => {
  bloomPass.radius = parseFloat(e.target.value);
  document.getElementById('bRadVal').textContent = parseFloat(e.target.value).toFixed(2);
});
bThresh.addEventListener('input', e => {
  bloomPass.threshold = parseFloat(e.target.value);
  document.getElementById('bThrVal').textContent = parseFloat(e.target.value).toFixed(2);
});
exposureSlider.addEventListener('input', e => {
  renderer.toneMappingExposure = parseFloat(e.target.value);
  document.getElementById('expVal').textContent = parseFloat(e.target.value).toFixed(1);
});

// ===== ANIMATION =====
const clock = new THREE.Clock();

function animate() {
  requestAnimationFrame(animate);
  const dt = clock.getDelta();
  const elapsed = clock.elapsedTime;

  // Animate rain
  const rp = rain.geometry.attributes.position;
  for (let i = 0; i < rainCount; i++) {
    rp.array[i*3+1] -= 8 * dt;
    if (rp.array[i*3+1] < 0) rp.array[i*3+1] = 30;
  }
  rp.needsUpdate = true;
  rain.position.x = Math.sin(elapsed * 0.3) * 0.5;

  // Neon flicker
  if (flickerCheck.checked) {
    neonObjects.forEach((obj, i) => {
      const flicker = Math.random() > 0.995 ? 0 : 1;
      const pulse = Math.sin(elapsed * 3 + i * 0.7) * 0.1 + 0.9;
      if (obj.mat.emissiveIntensity !== undefined) {
        obj.mat.emissiveIntensity = flicker * pulse * obj.baseIntensity;
      }
    });
  }

  controls.update();
  composer.render();
}

animate();

window.addEventListener('resize', () => {
  camera.aspect = innerWidth/innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(innerWidth, innerHeight);
  composer.setSize(innerWidth, innerHeight);
  bloomPass.setSize(innerWidth, innerHeight);
});
</script>
</body>
</html>
```

## 7. Bloom Parameters Guide

```javascript
// ===== ค่า Bloom สำหรับ Scenarios ต่างๆ =====

const bloomPresets = {
  // สำหรับ SciFi / Cyberpunk
  cyberpunk: {
    strength: 2.0,    // bloom เยอะ
    radius: 0.5,      // blur กว้าง
    threshold: 0.1,   // glows ทุกอย่างที่สว่างพอ
    exposure: 1.2
  },
  
  // สำหรับ Realistic Scene กลางวัน
  realistic: {
    strength: 0.3,    // bloom น้อย
    radius: 0.3,      // blur แคบ
    threshold: 0.9,   // เฉพาะสิ่งที่สว่างมากๆ
    exposure: 0.8
  },
  
  // สำหรับ Space Scene
  space: {
    strength: 1.5,
    radius: 0.8,      // blur ใหญ่สำหรับดาว
    threshold: 0.3,
    exposure: 1.5
  },
  
  // สำหรับ Neon/Night Scene
  neon: {
    strength: 1.8,
    radius: 0.6,
    threshold: 0.2,   // สีสดทุกอันจะ bloom
    exposure: 0.7     // exposure ต่ำเพื่อให้เห็น contrast
  },
  
  // สำหรับ Subtle Film Look
  film: {
    strength: 0.4,
    radius: 0.2,
    threshold: 0.8,
    exposure: 1.0
  }
};

// Apply preset
function applyBloomPreset(bloomPass, renderer, preset) {
  bloomPass.strength  = preset.strength;
  bloomPass.radius    = preset.radius;
  bloomPass.threshold = preset.threshold;
  renderer.toneMappingExposure = preset.exposure;
}
```

## สรุป

Post-Processing Toolkit:
1. **UnrealBloomPass** - bloom effect ที่ดีที่สุดใน Three.js
2. **Selective Bloom** - bloom เฉพาะบาง object ด้วย layers
3. **LensflareElement** - lens flare 3D แบบสมจริง
4. **Neon Effect** - emissive material + PointLight + bloom
5. **God Rays** - Screen-space radial blur จาก light source
6. **Bloom Presets** - ตั้งค่าสำหรับ scenarios ต่างๆ

### Best Practices
- ใช้ `ACESFilmicToneMapping` เสมอเมื่อมี bloom
- `emissiveIntensity > 1` ทำให้ object bloom ได้
- ปรับ `threshold` ต่ำสำหรับ neon, สูงสำหรับ realistic
- `OutputPass` ต่อท้ายเสมอเพื่อ color space correction

---

## แบบฝึกหัด

1. **Easy**: เพิ่ม bloom ให้ scene ปัจจุบันด้วย EffectComposer
2. **Medium**: สร้าง neon sign ด้วย TubeGeometry + emissive material
3. **Hard**: Selective bloom ด้วย object layers
4. **Challenge**: God rays จาก directional light ด้วย screen-space shader
