# Part 43: Procedural Terrain Part 1 ใน Three.js

## บทนำ

Procedural Terrain Generation เป็นเทคนิคการสร้างภูมิประเทศด้วยอัลกอริทึม ไม่ต้องสร้างด้วยมือ ใช้ในเกม open world, simulations, และ visualization

หัวใจของ terrain คือ **Heightmap** - ภาพ 2D ที่แต่ละ pixel เก็บความสูง (height) ของจุดนั้น

## 1. Heightmap Concept

```javascript
import * as THREE from 'three';

// ===== Heightmap = 2D array ของค่าความสูง =====

// สร้าง terrain จาก heightmap
function createTerrainFromHeightmap(heightmap, width, depth, maxHeight) {
  const widthSegments = heightmap[0].length - 1;
  const depthSegments = heightmap.length - 1;
  
  const geometry = new THREE.PlaneGeometry(
    width,        // world width
    depth,        // world depth
    widthSegments, // segments X
    depthSegments  // segments Z
  );
  
  // PlaneGeometry ใน XY plane -> rotate เป็น XZ plane
  geometry.rotateX(-Math.PI / 2);
  
  const positions = geometry.attributes.position;
  
  // ตั้งค่า Y (height) สำหรับแต่ละ vertex
  for (let z = 0; z <= depthSegments; z++) {
    for (let x = 0; x <= widthSegments; x++) {
      const vertexIndex = z * (widthSegments + 1) + x;
      const height = heightmap[z][x] * maxHeight;
      positions.setY(vertexIndex, height);
    }
  }
  
  positions.needsUpdate = true;
  geometry.computeVertexNormals(); // สำคัญ! ต้อง recalculate normals
  
  return geometry;
}

// ตัวอย่าง heightmap อย่างง่าย (5x5)
const simpleHeightmap = [
  [0.0, 0.1, 0.2, 0.1, 0.0],
  [0.1, 0.3, 0.5, 0.3, 0.1],
  [0.2, 0.5, 1.0, 0.5, 0.2],
  [0.1, 0.3, 0.5, 0.3, 0.1],
  [0.0, 0.1, 0.2, 0.1, 0.0]
];

const terrainGeo = createTerrainFromHeightmap(simpleHeightmap, 10, 10, 5);
const terrainMat = new THREE.MeshStandardMaterial({ 
  color: 0x44aa44,
  wireframe: false 
});
const terrain = new THREE.Mesh(terrainGeo, terrainMat);
scene.add(terrain);
```

## 2. Perlin Noise สำหรับ Terrain

```javascript
// ===== Simplex/Perlin Noise =====
// Noise ให้ค่า random ที่ smooth และ continuous

// Implementation ของ Perlin Noise
class PerlinNoise {
  constructor(seed = 0) {
    this.permutation = [];
    this.init(seed);
  }
  
  init(seed) {
    // สร้าง permutation table
    for (let i = 0; i < 256; i++) {
      this.permutation[i] = i;
    }
    
    // Shuffle ด้วย seed
    for (let i = 255; i > 0; i--) {
      const j = Math.floor((this.seededRandom(seed + i)) * (i + 1));
      [this.permutation[i], this.permutation[j]] = 
        [this.permutation[j], this.permutation[i]];
    }
    
    // Double array
    this.p = [...this.permutation, ...this.permutation];
  }
  
  seededRandom(seed) {
    const x = Math.sin(seed) * 10000;
    return x - Math.floor(x);
  }
  
  fade(t) {
    return t * t * t * (t * (t * 6 - 15) + 10);
  }
  
  lerp(t, a, b) {
    return a + t * (b - a);
  }
  
  grad(hash, x, y) {
    const h = hash & 3;
    const u = h < 2 ? x : y;
    const v = h < 2 ? y : x;
    return ((h & 1) ? -u : u) + ((h & 2) ? -v : v);
  }
  
  noise2D(x, y) {
    const X = Math.floor(x) & 255;
    const Y = Math.floor(y) & 255;
    
    x -= Math.floor(x);
    y -= Math.floor(y);
    
    const u = this.fade(x);
    const v = this.fade(y);
    
    const a = this.p[X] + Y;
    const aa = this.p[a];
    const ab = this.p[a + 1];
    const b = this.p[X + 1] + Y;
    const ba = this.p[b];
    const bb = this.p[b + 1];
    
    return this.lerp(v,
      this.lerp(u, this.grad(this.p[aa], x, y), this.grad(this.p[ba], x - 1, y)),
      this.lerp(u, this.grad(this.p[ab], x, y - 1), this.grad(this.p[bb], x - 1, y - 1))
    );
  }
  
  // Fractional Brownian Motion (fBm) - layered noise
  fbm(x, y, octaves = 6, persistence = 0.5, lacunarity = 2.0) {
    let value = 0;
    let amplitude = 1;
    let frequency = 1;
    let maxValue = 0;
    
    for (let i = 0; i < octaves; i++) {
      value += this.noise2D(x * frequency, y * frequency) * amplitude;
      maxValue += amplitude;
      amplitude *= persistence;
      frequency *= lacunarity;
    }
    
    return value / maxValue;
  }
}

// ===== Generate Terrain ด้วย Noise =====
function generateNoiseTerrain(width, depth, scale, octaves, persistence, lacunarity, seed) {
  const noise = new PerlinNoise(seed);
  const widthSegs = 128;
  const depthSegs = 128;
  
  const geometry = new THREE.PlaneGeometry(width, depth, widthSegs, depthSegs);
  geometry.rotateX(-Math.PI / 2);
  
  const positions = geometry.attributes.position;
  
  for (let i = 0; i < positions.count; i++) {
    const x = positions.getX(i);
    const z = positions.getZ(i);
    
    // ใช้ fBm สำหรับ natural-looking terrain
    const nx = x / scale;
    const nz = z / scale;
    
    let height = noise.fbm(nx, nz, octaves, persistence, lacunarity);
    
    // Normalize -1..1 ไปเป็น 0..1
    height = (height + 1) * 0.5;
    
    // ใช้ power function เพื่อทำให้ mountains สูงกว่า
    height = Math.pow(height, 1.5);
    
    positions.setY(i, height * 15);
  }
  
  positions.needsUpdate = true;
  geometry.computeVertexNormals();
  
  return geometry;
}
```

## 3. Terrain Material ตาม Height

```javascript
// ===== Multi-layer terrain material =====
// ใช้ height เพื่อกำหนดสีของ terrain

const terrainShaderMaterial = new THREE.ShaderMaterial({
  uniforms: {
    grassColor: { value: new THREE.Color(0x336622) },
    rockColor: { value: new THREE.Color(0x887766) },
    snowColor: { value: new THREE.Color(0xffffff) },
    sandColor: { value: new THREE.Color(0xddcc88) },
    dirtColor: { value: new THREE.Color(0x8b6914) },
    maxHeight: { value: 15.0 },
    grassHeight: { value: 0.3 },
    rockHeight: { value: 0.6 },
    snowHeight: { value: 0.85 },
    waterLevel: { value: 0.1 },
    sunDir: { value: new THREE.Vector3(0.7, 1, 0.5).normalize() }
  },
  
  vertexShader: `
    varying vec3 vNormal;
    varying vec3 vWorldPos;
    varying float vHeight;
    
    void main() {
      vec4 worldPos = modelMatrix * vec4(position, 1.0);
      vWorldPos = worldPos.xyz;
      vNormal = normalize(normalMatrix * normal);
      vHeight = worldPos.y;
      gl_Position = projectionMatrix * viewMatrix * worldPos;
    }
  `,
  
  fragmentShader: `
    uniform vec3 grassColor;
    uniform vec3 rockColor;
    uniform vec3 snowColor;
    uniform vec3 sandColor;
    uniform vec3 dirtColor;
    uniform float maxHeight;
    uniform float grassHeight;
    uniform float rockHeight;
    uniform float snowHeight;
    uniform float waterLevel;
    uniform vec3 sunDir;
    
    varying vec3 vNormal;
    varying vec3 vWorldPos;
    varying float vHeight;
    
    void main() {
      vec3 N = normalize(vNormal);
      
      // Normalize height to 0-1
      float h = vHeight / maxHeight;
      h = clamp(h, 0.0, 1.0);
      
      // Slope factor (สูงชัน = หิน)
      float slope = 1.0 - N.y; // 0 = flat, 1 = vertical
      
      // Color blending based on height
      vec3 color;
      
      if (h < waterLevel) {
        // Sand (near water)
        color = sandColor;
      } else if (h < grassHeight) {
        // Transition sand -> grass
        float t = smoothstep(waterLevel, grassHeight, h);
        color = mix(sandColor, mix(grassColor, dirtColor, slope), t);
      } else if (h < rockHeight) {
        // Transition grass -> rock
        float t = smoothstep(grassHeight, rockHeight, h);
        // สูงชันกว่า = มีหินมากกว่า
        float slopeMix = smoothstep(0.3, 0.7, slope);
        color = mix(grassColor, rockColor, max(t, slopeMix));
      } else if (h < snowHeight) {
        // Rock
        float slopeMix = smoothstep(0.2, 0.6, slope);
        color = mix(rockColor, dirtColor, slopeMix);
      } else {
        // Snow (top)
        float t = smoothstep(snowHeight, 1.0, h);
        color = mix(rockColor, snowColor, t);
      }
      
      // Lighting
      float diffuse = max(dot(N, sunDir), 0.1);
      color *= diffuse;
      
      // Ambient occlusion approximation
      float ao = 1.0 - slope * 0.3;
      color *= ao;
      
      gl_FragColor = vec4(color, 1.0);
    }
  `
});
```

## 4. Vertex Normals Recalculation

```javascript
// ===== ทำไมต้อง recalculate normals? =====
// เมื่อเราเลื่อน vertex positions ของ PlaneGeometry
// normals เดิมจะไม่ถูกต้องอีกต่อไป

function updateTerrainGeometry(geometry, heightData) {
  const positions = geometry.attributes.position;
  
  // Step 1: Update positions
  for (let i = 0; i < positions.count; i++) {
    positions.setY(i, heightData[i]);
  }
  positions.needsUpdate = true;
  
  // Step 2: Compute new normals
  // Method 1: Three.js built-in (fast, uses adjacent triangles)
  geometry.computeVertexNormals();
  
  // Method 2: Manual normal calculation (more control)
  computeNormalsManual(geometry);
}

function computeNormalsManual(geometry) {
  const positions = geometry.attributes.position;
  const indices = geometry.index;
  
  // Reset normals
  const normals = new Float32Array(positions.count * 3);
  
  if (indices) {
    // Indexed geometry
    for (let i = 0; i < indices.count; i += 3) {
      const a = indices.getX(i);
      const b = indices.getX(i + 1);
      const c = indices.getX(i + 2);
      
      const vA = new THREE.Vector3(
        positions.getX(a), positions.getY(a), positions.getZ(a)
      );
      const vB = new THREE.Vector3(
        positions.getX(b), positions.getY(b), positions.getZ(b)
      );
      const vC = new THREE.Vector3(
        positions.getX(c), positions.getY(c), positions.getZ(c)
      );
      
      // Normal = cross product ของ 2 edges
      const edge1 = vB.clone().sub(vA);
      const edge2 = vC.clone().sub(vA);
      const normal = edge1.cross(edge2).normalize();
      
      // Add to each vertex
      [a, b, c].forEach(idx => {
        normals[idx * 3] += normal.x;
        normals[idx * 3 + 1] += normal.y;
        normals[idx * 3 + 2] += normal.z;
      });
    }
  }
  
  // Normalize accumulated normals
  for (let i = 0; i < positions.count; i++) {
    const n = new THREE.Vector3(
      normals[i * 3], normals[i * 3 + 1], normals[i * 3 + 2]
    ).normalize();
    normals[i * 3] = n.x;
    normals[i * 3 + 1] = n.y;
    normals[i * 3 + 2] = n.z;
  }
  
  geometry.setAttribute('normal', new THREE.BufferAttribute(normals, 3));
}
```

## 5. LOD Terrain System

```javascript
// ===== Level of Detail สำหรับ Terrain =====
class TerrainLOD {
  constructor(scene, noise) {
    this.scene = scene;
    this.noise = noise;
    this.chunks = new Map();
    this.camera = null;
    
    this.chunkSize = 32; // world units ต่อ chunk
    this.levels = [
      { distance: 50, segments: 64 },   // LOD 0: high detail
      { distance: 150, segments: 32 },  // LOD 1: medium
      { distance: 300, segments: 16 },  // LOD 2: low
      { distance: 600, segments: 8 },   // LOD 3: very low
    ];
  }
  
  getChunkKey(cx, cz) {
    return `${cx},${cz}`;
  }
  
  createChunk(cx, cz, lod) {
    const segments = this.levels[lod].segments;
    const geo = new THREE.PlaneGeometry(
      this.chunkSize, this.chunkSize, segments, segments
    );
    geo.rotateX(-Math.PI / 2);
    
    const positions = geo.attributes.position;
    const worldX = cx * this.chunkSize;
    const worldZ = cz * this.chunkSize;
    
    for (let i = 0; i < positions.count; i++) {
      const x = positions.getX(i) + worldX;
      const z = positions.getZ(i) + worldZ;
      const height = this.noise.fbm(x * 0.01, z * 0.01, 6, 0.5, 2.0) * 20;
      positions.setY(i, height);
    }
    
    positions.needsUpdate = true;
    geo.computeVertexNormals();
    
    const mesh = new THREE.Mesh(geo, terrainShaderMaterial.clone());
    mesh.position.set(worldX, 0, worldZ);
    this.scene.add(mesh);
    
    return { mesh, lod, cx, cz };
  }
  
  removeChunk(key) {
    const chunk = this.chunks.get(key);
    if (chunk) {
      this.scene.remove(chunk.mesh);
      chunk.mesh.geometry.dispose();
      this.chunks.delete(key);
    }
  }
  
  update(camera) {
    const camX = camera.position.x;
    const camZ = camera.position.z;
    
    // คำนวณช่วง chunks ที่ควรจะ visible
    const viewRange = this.levels[this.levels.length - 1].distance;
    const chunkRange = Math.ceil(viewRange / this.chunkSize);
    
    const centerCX = Math.floor(camX / this.chunkSize);
    const centerCZ = Math.floor(camZ / this.chunkSize);
    
    // Mark old chunks for removal
    const toRemove = new Set(this.chunks.keys());
    
    // Add/update chunks in range
    for (let dz = -chunkRange; dz <= chunkRange; dz++) {
      for (let dx = -chunkRange; dx <= chunkRange; dx++) {
        const cx = centerCX + dx;
        const cz = centerCZ + dz;
        const key = this.getChunkKey(cx, cz);
        
        // ระยะห่างจาก center
        const worldDist = Math.sqrt(
          (dx * this.chunkSize) ** 2 + (dz * this.chunkSize) ** 2
        );
        
        // กำหนด LOD ตามระยะ
        let lod = this.levels.length - 1;
        for (let l = 0; l < this.levels.length; l++) {
          if (worldDist < this.levels[l].distance) {
            lod = l;
            break;
          }
        }
        
        if (worldDist > viewRange) continue;
        
        toRemove.delete(key);
        
        const existing = this.chunks.get(key);
        if (!existing || existing.lod !== lod) {
          if (existing) this.removeChunk(key);
          const chunk = this.createChunk(cx, cz, lod);
          this.chunks.set(key, chunk);
        }
      }
    }
    
    // Remove out-of-range chunks
    toRemove.forEach(key => this.removeChunk(key));
  }
}
```

## 6. Biomes Concept

```javascript
// ===== Biomes =====
// ใช้ Temperature + Moisture ในการกำหนด biome

class BiomeSystem {
  constructor() {
    this.temperatureNoise = new PerlinNoise(42);
    this.moistureNoise = new PerlinNoise(137);
  }
  
  getBiome(x, z) {
    const temp = this.temperatureNoise.fbm(x * 0.002, z * 0.002, 4, 0.5, 2) * 0.5 + 0.5;
    const moisture = this.moistureNoise.fbm(x * 0.003, z * 0.003, 4, 0.5, 2) * 0.5 + 0.5;
    
    // Whittaker biome classification
    if (temp < 0.2) {
      return moisture < 0.5 ? 'tundra' : 'snow';
    } else if (temp < 0.5) {
      if (moisture < 0.3) return 'steppe';
      if (moisture < 0.6) return 'forest';
      return 'taiga';
    } else if (temp < 0.8) {
      if (moisture < 0.3) return 'savanna';
      if (moisture < 0.6) return 'temperate_forest';
      return 'temperate_rainforest';
    } else {
      if (moisture < 0.2) return 'desert';
      if (moisture < 0.5) return 'tropical_forest';
      return 'tropical_rainforest';
    }
  }
  
  getBiomeColor(biome) {
    const colors = {
      tundra: new THREE.Color(0x8899aa),
      snow: new THREE.Color(0xeeeeff),
      steppe: new THREE.Color(0xaaaa55),
      forest: new THREE.Color(0x227733),
      taiga: new THREE.Color(0x336644),
      savanna: new THREE.Color(0xcc9944),
      temperate_forest: new THREE.Color(0x338822),
      temperate_rainforest: new THREE.Color(0x225522),
      desert: new THREE.Color(0xddcc88),
      tropical_forest: new THREE.Color(0x226611),
      tropical_rainforest: new THREE.Color(0x116622)
    };
    return colors[biome] || new THREE.Color(0x448822);
  }
}
```

## 7. Demo: Interactive Terrain Generator

```html
<!DOCTYPE html>
<html>
<head>
  <title>Procedural Terrain Generator</title>
  <style>
    body { margin: 0; background: #1a2a3a; overflow: hidden; }
    canvas { display: block; }
    #controls {
      position: fixed;
      top: 20px;
      right: 20px;
      background: rgba(0,0,0,0.85);
      color: #ddd;
      padding: 20px;
      border-radius: 10px;
      font-family: Arial, sans-serif;
      font-size: 13px;
      width: 250px;
    }
    h3 { color: #8af; margin: 0 0 12px; }
    .group { margin: 10px 0; }
    label { display: block; margin-bottom: 3px; color: #8af; }
    input[type="range"] { width: 210px; }
    .val { color: #fa8; font-size: 11px; }
    button { 
      width: 100%;
      margin: 4px 0;
      padding: 8px;
      background: #224466;
      color: #adf;
      border: 1px solid #446688;
      border-radius: 4px;
      cursor: pointer;
      font-size: 13px;
    }
    button:hover { background: #335577; }
    #toggleWire {
      background: #442244;
      border-color: #664466;
      color: #faa;
    }
  </style>
</head>
<body>
<div id="controls">
  <h3>Terrain Generator</h3>
  
  <div class="group">
    <label>Scale: <span class="val" id="scaleVal">50</span></label>
    <input type="range" id="scale" min="5" max="200" value="50">
  </div>
  
  <div class="group">
    <label>Octaves: <span class="val" id="octavesVal">6</span></label>
    <input type="range" id="octaves" min="1" max="10" value="6">
  </div>
  
  <div class="group">
    <label>Persistence: <span class="val" id="persVal">0.50</span></label>
    <input type="range" id="persistence" min="0" max="100" value="50">
  </div>
  
  <div class="group">
    <label>Lacunarity: <span class="val" id="lacuVal">2.00</span></label>
    <input type="range" id="lacunarity" min="100" max="400" value="200">
  </div>
  
  <div class="group">
    <label>Max Height: <span class="val" id="heightVal">15</span></label>
    <input type="range" id="maxHeight" min="1" max="50" value="15">
  </div>
  
  <div class="group">
    <label>Resolution: <span class="val" id="resVal">128</span></label>
    <input type="range" id="resolution" min="16" max="256" value="128" step="16">
  </div>
  
  <div class="group">
    <label>Seed: <span class="val" id="seedVal">42</span></label>
    <input type="range" id="seed" min="0" max="999" value="42">
  </div>
  
  <button id="generateBtn">Generate Terrain</button>
  <button id="randomBtn">Random Seed</button>
  <button id="toggleWire">Toggle Wireframe</button>
  <button id="exportBtn">Export Settings</button>
</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';

// ===== SETUP =====
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x1a2a3a);
scene.fog = new THREE.FogExp2(0x1a2a3a, 0.01);

const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 1000);
camera.position.set(0, 30, 60);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(devicePixelRatio);
renderer.shadowMap.enabled = true;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.outputColorSpace = THREE.SRGBColorSpace;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.target.set(0, 0, 0);
controls.maxPolarAngle = Math.PI / 2;

// Lighting
const ambientLight = new THREE.AmbientLight(0x334466, 2);
scene.add(ambientLight);

const sunLight = new THREE.DirectionalLight(0xfff8e0, 3);
sunLight.position.set(100, 150, 100);
sunLight.castShadow = true;
sunLight.shadow.mapSize.set(2048, 2048);
sunLight.shadow.camera.near = 1;
sunLight.shadow.camera.far = 500;
sunLight.shadow.camera.left = -100;
sunLight.shadow.camera.right = 100;
sunLight.shadow.camera.top = 100;
sunLight.shadow.camera.bottom = -100;
scene.add(sunLight);

const fillLight = new THREE.DirectionalLight(0x8899cc, 1);
fillLight.position.set(-100, 50, -100);
scene.add(fillLight);

// ===== PERLIN NOISE =====
class SimpleNoise {
  constructor(seed = 0) {
    this.seed = seed;
  }
  
  seededRandom(n) {
    n = Math.sin(n + this.seed) * 43758.5453123;
    return n - Math.floor(n);
  }
  
  smoothStep(t) {
    return t * t * (3 - 2 * t);
  }
  
  noise2D(x, y) {
    const ix = Math.floor(x);
    const iy = Math.floor(y);
    const fx = x - ix;
    const fy = y - iy;
    
    const ux = this.smoothStep(fx);
    const uy = this.smoothStep(fy);
    
    const a = this.seededRandom(ix + iy * 57);
    const b = this.seededRandom(ix + 1 + iy * 57);
    const c = this.seededRandom(ix + (iy + 1) * 57);
    const d = this.seededRandom(ix + 1 + (iy + 1) * 57);
    
    return (a + (b - a) * ux + (c - a) * uy + (a - b - c + d) * ux * uy) * 2 - 1;
  }
  
  fbm(x, y, octaves, persistence, lacunarity) {
    let value = 0, amplitude = 1, frequency = 1, maxValue = 0;
    for (let i = 0; i < octaves; i++) {
      value += this.noise2D(x * frequency, y * frequency) * amplitude;
      maxValue += amplitude;
      amplitude *= persistence;
      frequency *= lacunarity;
    }
    return value / maxValue;
  }
}

// ===== TERRAIN =====
let terrainMesh = null;
let wireframeOn = false;

const terrainShader = new THREE.ShaderMaterial({
  uniforms: {
    maxHeight: { value: 15 },
    waterLevel: { value: 1.5 },
    sunDir: { value: new THREE.Vector3(0.7, 1, 0.5).normalize() }
  },
  vertexShader: `
    varying vec3 vWorldPos;
    varying vec3 vNormal;
    
    void main() {
      vec4 worldPos = modelMatrix * vec4(position, 1.0);
      vWorldPos = worldPos.xyz;
      vNormal = normalize(normalMatrix * normal);
      gl_Position = projectionMatrix * viewMatrix * worldPos;
    }
  `,
  fragmentShader: `
    uniform float maxHeight;
    uniform float waterLevel;
    uniform vec3 sunDir;
    
    varying vec3 vWorldPos;
    varying vec3 vNormal;
    
    vec3 colorAt(float h, float slope) {
      if (h < waterLevel * 0.7) return vec3(0.2, 0.5, 0.7); // water
      if (h < waterLevel) return vec3(0.7, 0.65, 0.5); // beach/sand
      
      float t = (h - waterLevel) / (maxHeight - waterLevel);
      float slopeFactor = smoothstep(0.3, 0.7, slope);
      
      vec3 grass = vec3(0.2, 0.55, 0.15);
      vec3 dirt = vec3(0.45, 0.3, 0.15);
      vec3 rock = vec3(0.55, 0.5, 0.45);
      vec3 snow = vec3(0.95, 0.95, 1.0);
      
      if (t < 0.3) return mix(mix(grass, dirt, slopeFactor), mix(grass, dirt, slopeFactor), t / 0.3);
      if (t < 0.6) return mix(mix(grass, dirt, slopeFactor), rock, (t - 0.3) / 0.3);
      if (t < 0.85) return mix(rock, mix(rock, snow, slopeFactor * 0.5), (t - 0.6) / 0.25);
      return mix(rock, snow, (t - 0.85) / 0.15);
    }
    
    void main() {
      vec3 N = normalize(vNormal);
      float slope = 1.0 - N.y;
      
      vec3 baseColor = colorAt(vWorldPos.y, slope);
      
      float diffuse = max(dot(N, sunDir), 0.0) * 0.8 + 0.2;
      vec3 finalColor = baseColor * diffuse;
      
      gl_FragColor = vec4(finalColor, 1.0);
    }
  `,
  side: THREE.FrontSide
});

function generateTerrain() {
  if (terrainMesh) {
    scene.remove(terrainMesh);
    terrainMesh.geometry.dispose();
  }
  
  const scale = parseFloat(document.getElementById('scale').value);
  const octaves = parseInt(document.getElementById('octaves').value);
  const persistence = parseFloat(document.getElementById('persistence').value) / 100;
  const lacunarity = parseFloat(document.getElementById('lacunarity').value) / 100;
  const maxHeight = parseFloat(document.getElementById('maxHeight').value);
  const resolution = parseInt(document.getElementById('resolution').value);
  const seed = parseInt(document.getElementById('seed').value);
  
  terrainShader.uniforms.maxHeight.value = maxHeight;
  terrainShader.uniforms.waterLevel.value = maxHeight * 0.15;
  
  const noise = new SimpleNoise(seed);
  
  const size = 100;
  const geo = new THREE.PlaneGeometry(size, size, resolution, resolution);
  geo.rotateX(-Math.PI / 2);
  
  const positions = geo.attributes.position;
  
  for (let i = 0; i < positions.count; i++) {
    const x = positions.getX(i);
    const z = positions.getZ(i);
    
    let h = noise.fbm(x / scale, z / scale, octaves, persistence, lacunarity);
    h = (h + 1) * 0.5;
    h = Math.pow(h, 1.5);
    
    // Island falloff
    const dist = Math.sqrt(x * x + z * z) / (size * 0.5);
    const falloff = Math.max(0, 1 - dist * dist);
    h *= falloff;
    
    positions.setY(i, h * maxHeight);
  }
  
  positions.needsUpdate = true;
  geo.computeVertexNormals();
  
  terrainMesh = new THREE.Mesh(geo, wireframeOn ? terrainShader.clone() : terrainShader);
  terrainMesh.castShadow = true;
  terrainMesh.receiveShadow = true;
  
  if (wireframeOn) {
    const wireMat = new THREE.WireframeGeometry(geo);
    const lines = new THREE.LineSegments(wireMat, new THREE.LineBasicMaterial({ color: 0x445566 }));
    terrainMesh.add(lines);
  }
  
  scene.add(terrainMesh);
  
  // Water plane
  if (!scene.getObjectByName('water')) {
    const water = new THREE.Mesh(
      new THREE.PlaneGeometry(size, size),
      new THREE.MeshStandardMaterial({
        color: 0x1166aa,
        transparent: true,
        opacity: 0.7,
        roughness: 0.1
      })
    );
    water.rotation.x = -Math.PI / 2;
    water.position.y = maxHeight * 0.15;
    water.name = 'water';
    scene.add(water);
  } else {
    scene.getObjectByName('water').position.y = maxHeight * 0.15;
  }
}

// ===== CONTROLS =====
const sliders = ['scale', 'octaves', 'persistence', 'lacunarity', 'maxHeight', 'resolution', 'seed'];
sliders.forEach(id => {
  const el = document.getElementById(id);
  el.addEventListener('input', (e) => {
    let val = e.target.value;
    if (id === 'persistence') val = (val / 100).toFixed(2);
    else if (id === 'lacunarity') val = (val / 100).toFixed(2);
    document.getElementById(id + 'Val').textContent = val;
  });
});

document.getElementById('generateBtn').addEventListener('click', generateTerrain);

document.getElementById('randomBtn').addEventListener('click', () => {
  const seed = Math.floor(Math.random() * 1000);
  document.getElementById('seed').value = seed;
  document.getElementById('seedVal').textContent = seed;
  generateTerrain();
});

document.getElementById('toggleWire').addEventListener('click', () => {
  wireframeOn = !wireframeOn;
  terrainShader.wireframe = wireframeOn;
  document.getElementById('toggleWire').textContent = 
    wireframeOn ? 'Solid Mode' : 'Toggle Wireframe';
});

document.getElementById('exportBtn').addEventListener('click', () => {
  const settings = {
    scale: document.getElementById('scale').value,
    octaves: document.getElementById('octaves').value,
    persistence: document.getElementById('persistence').value,
    lacunarity: document.getElementById('lacunarity').value,
    maxHeight: document.getElementById('maxHeight').value,
    resolution: document.getElementById('resolution').value,
    seed: document.getElementById('seed').value
  };
  console.log('Terrain Settings:', JSON.stringify(settings, null, 2));
  alert('Settings logged to console!');
});

// Generate initial terrain
generateTerrain();

// ===== ANIMATION =====
const clock = new THREE.Clock();
function animate() {
  requestAnimationFrame(animate);
  
  const elapsed = clock.elapsedTime;
  
  // Animate water
  const water = scene.getObjectByName('water');
  if (water) {
    water.position.y = terrainShader.uniforms.maxHeight.value * 0.15 +
      Math.sin(elapsed * 0.5) * 0.05;
  }
  
  controls.update();
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

## สรุป

Procedural Terrain Generation ใช้:

1. **Heightmap** - 2D array ของ height values
2. **Perlin/Simplex Noise** - smooth random terrain
3. **fBm** - layered noise สำหรับ natural detail
4. **Vertex Normal Recalculation** - สำหรับ correct lighting
5. **Height-based Materials** - grass/rock/snow ตามความสูง
6. **LOD** - ประหยัด polygons สำหรับ distant terrain
7. **Biomes** - ประเภทภูมิประเทศตาม temperature/moisture

---

## แบบฝึกหัด

1. **Easy**: สร้าง terrain จาก noise ด้วย vertex displacement บน PlaneGeometry
2. **Medium**: เพิ่ม height-based material (sand/grass/rock/snow) ด้วย custom shader
3. **Hard**: สร้าง chunked terrain ที่โหลดตาม player position พร้อม LOD
4. **Challenge**: สร้าง biome system ที่ใช้ temperature และ moisture map กำหนดสีและต้นไม้
