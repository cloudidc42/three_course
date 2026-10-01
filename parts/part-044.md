# Part 44: Procedural Terrain Part 2 (Advanced) ใน Three.js

## บทนำ

ต่อจาก Part 43 เราจะเจาะลึกเทคนิค terrain generation ขั้นสูง:
- **Vertex Displacement Shader** - terrain บน GPU
- **Erosion Algorithms** - จำลองการกัดเซาะตามธรรมชาติ
- **Voxel Terrain** (Minecraft-style) - ด้วย Greedy Meshing
- **Marching Cubes** - smooth voxel terrain
- **Cave Generation** - 3D noise สำหรับถ้ำ

## 1. Terrain as Shader (Vertex Displacement ใน GLSL)

```javascript
// ===== Terrain Displacement ใน Vertex Shader =====
// ดีกว่า CPU terrain: GPU parallel, dynamic, ไม่ต้อง upload data

const terrainVertexShader = `
  // Simplex Noise Implementation ใน GLSL
  vec3 mod289(vec3 x) { return x - floor(x * (1.0 / 289.0)) * 289.0; }
  vec2 mod289(vec2 x) { return x - floor(x * (1.0 / 289.0)) * 289.0; }
  vec3 permute(vec3 x) { return mod289(((x*34.0)+1.0)*x); }
  
  float snoise(vec2 v) {
    const vec4 C = vec4(0.211324865405187, 0.366025403784439,
                       -0.577350269189626, 0.024390243902439);
    vec2 i  = floor(v + dot(v, C.yy));
    vec2 x0 = v -   i + dot(i, C.xx);
    vec2 i1;
    i1 = (x0.x > x0.y) ? vec2(1.0, 0.0) : vec2(0.0, 1.0);
    vec4 x12 = x0.xyxy + C.xxzz;
    x12.xy -= i1;
    i = mod289(i);
    vec3 p = permute(permute(i.y + vec3(0.0, i1.y, 1.0)) + i.x + vec3(0.0, i1.x, 1.0));
    vec3 m = max(0.5 - vec3(dot(x0,x0), dot(x12.xy,x12.xy), dot(x12.zw,x12.zw)), 0.0);
    m = m*m; m = m*m;
    vec3 x = 2.0 * fract(p * C.www) - 1.0;
    vec3 h = abs(x) - 0.5;
    vec3 ox = floor(x + 0.5);
    vec3 a0 = x - ox;
    m *= 1.79284291400159 - 0.85373472095314 * (a0*a0 + h*h);
    vec3 g;
    g.x  = a0.x  * x0.x  + h.x  * x0.y;
    g.yz = a0.yz * x12.xz + h.yz * x12.yw;
    return 130.0 * dot(m, g);
  }
  
  // fBm noise
  float fbm(vec2 p, int octaves, float persistence, float lacunarity) {
    float value = 0.0;
    float amplitude = 1.0;
    float frequency = 1.0;
    float maxValue = 0.0;
    
    for (int i = 0; i < 8; i++) {
      if (i >= octaves) break;
      value += snoise(p * frequency) * amplitude;
      maxValue += amplitude;
      amplitude *= persistence;
      frequency *= lacunarity;
    }
    
    return value / maxValue;
  }
  
  uniform float time;
  uniform float maxHeight;
  uniform float scale;
  uniform int octaves;
  uniform float persistence;
  uniform float lacunarity;
  
  varying float vHeight;
  varying vec3 vNormal;
  varying vec3 vWorldPos;
  
  void main() {
    vec2 noiseCoord = position.xz / scale;
    
    float h = fbm(noiseCoord, octaves, persistence, lacunarity);
    h = (h + 1.0) * 0.5; // normalize to 0-1
    h = pow(h, 1.5);
    
    // Island falloff
    float dist = length(position.xz) / 50.0;
    h *= max(0.0, 1.0 - dist * dist);
    
    vec3 displaced = position;
    displaced.y = h * maxHeight;
    
    vHeight = displaced.y;
    vWorldPos = (modelMatrix * vec4(displaced, 1.0)).xyz;
    
    // Approximate normal from nearby samples
    float eps = 0.5;
    float hR = fbm(noiseCoord + vec2(eps/scale, 0.0), octaves, persistence, lacunarity);
    float hU = fbm(noiseCoord + vec2(0.0, eps/scale), octaves, persistence, lacunarity);
    hR = (hR + 1.0) * 0.5; hR = pow(hR, 1.5);
    hU = (hU + 1.0) * 0.5; hU = pow(hU, 1.5);
    
    vec3 right = vec3(eps, (hR - h) * maxHeight, 0.0);
    vec3 up = vec3(0.0, (hU - h) * maxHeight, eps);
    vNormal = normalize(cross(up, right));
    
    gl_Position = projectionMatrix * modelViewMatrix * vec4(displaced, 1.0);
  }
`;

// ใน Three.js
const gpuTerrainMaterial = new THREE.ShaderMaterial({
  uniforms: {
    time: { value: 0 },
    maxHeight: { value: 20 },
    scale: { value: 50 },
    octaves: { value: 6 },
    persistence: { value: 0.5 },
    lacunarity: { value: 2.0 }
  },
  vertexShader: terrainVertexShader,
  fragmentShader: `/* ... terrain color fragment shader ... */`
});
```

## 2. Hydraulic Erosion Algorithm

```javascript
// ===== Hydraulic Erosion =====
// จำลองน้ำไหลบน terrain และกัดเซาะผิว

class HydraulicErosion {
  constructor(heightmap, width, height) {
    this.heightmap = heightmap.slice(); // copy
    this.width = width;
    this.height = height;
  }
  
  // Droplet-based hydraulic erosion
  simulate(numDroplets = 50000, params = {}) {
    const {
      inertia = 0.05,         // droplet inertia (ความเฉื่อย)
      capacity = 4,           // sediment capacity
      deposition = 0.3,       // deposition rate
      erosion = 0.3,          // erosion rate
      evaporation = 0.01,     // water evaporation
      minSlope = 0.01,        // minimum slope
      gravity = 4,            // gravity
      maxSteps = 64           // max steps per droplet
    } = params;
    
    for (let i = 0; i < numDroplets; i++) {
      // Spawn droplet at random position
      let x = Math.random() * (this.width - 2) + 1;
      let y = Math.random() * (this.height - 2) + 1;
      let dx = 0, dy = 0; // direction
      let speed = 1;
      let water = 1;
      let sediment = 0;
      
      for (let step = 0; step < maxSteps; step++) {
        const xi = Math.floor(x);
        const yi = Math.floor(y);
        
        if (xi <= 0 || xi >= this.width - 1 || yi <= 0 || yi >= this.height - 1) break;
        
        // Calculate gradient at current position (bilinear)
        const cellX = x - xi;
        const cellY = y - yi;
        
        const h00 = this.getHeight(xi, yi);
        const h10 = this.getHeight(xi + 1, yi);
        const h01 = this.getHeight(xi, yi + 1);
        const h11 = this.getHeight(xi + 1, yi + 1);
        
        const gradX = (h10 - h00) * (1 - cellY) + (h11 - h01) * cellY;
        const gradY = (h01 - h00) * (1 - cellX) + (h11 - h10) * cellX;
        
        // Update direction with inertia
        dx = dx * inertia - gradX * (1 - inertia);
        dy = dy * inertia - gradY * (1 - inertia);
        
        // Normalize direction
        const len = Math.sqrt(dx * dx + dy * dy);
        if (len <= 0) break;
        dx /= len;
        dy /= len;
        
        // Move droplet
        const newX = x + dx;
        const newY = y + dy;
        
        if (newX < 0 || newX >= this.width - 1 || newY < 0 || newY >= this.height - 1) break;
        
        // Height difference
        const newH = this.getHeightInterp(newX, newY);
        const oldH = this.getHeightInterp(x, y);
        const deltaH = newH - oldH;
        
        // Sediment capacity
        const cap = Math.max(-deltaH, minSlope) * speed * water * capacity;
        
        if (sediment > cap || deltaH > 0) {
          // Deposit sediment
          const depositAmount = deltaH > 0
            ? Math.min(deltaH, sediment)
            : (sediment - cap) * deposition;
          
          sediment -= depositAmount;
          this.depositAt(x, y, depositAmount);
        } else {
          // Erode
          const erodeAmount = Math.min((cap - sediment) * erosion, -deltaH);
          sediment += erodeAmount;
          this.erodeAt(x, y, erodeAmount);
        }
        
        // Update speed and water
        speed = Math.sqrt(Math.max(0, speed * speed + deltaH * gravity));
        water *= (1 - evaporation);
        
        if (water < 0.01) break;
        
        x = newX;
        y = newY;
      }
    }
    
    return this.heightmap;
  }
  
  getHeight(x, y) {
    const idx = Math.floor(y) * this.width + Math.floor(x);
    return this.heightmap[idx] || 0;
  }
  
  getHeightInterp(x, y) {
    const xi = Math.floor(x);
    const yi = Math.floor(y);
    const fx = x - xi;
    const fy = y - yi;
    
    const h00 = this.getHeight(xi, yi);
    const h10 = this.getHeight(xi + 1, yi);
    const h01 = this.getHeight(xi, yi + 1);
    const h11 = this.getHeight(xi + 1, yi + 1);
    
    return h00 * (1 - fx) * (1 - fy) +
           h10 * fx * (1 - fy) +
           h01 * (1 - fx) * fy +
           h11 * fx * fy;
  }
  
  depositAt(x, y, amount) {
    const xi = Math.floor(x);
    const yi = Math.floor(y);
    const fx = x - xi;
    const fy = y - yi;
    
    this.addHeight(xi, yi, amount * (1 - fx) * (1 - fy));
    this.addHeight(xi + 1, yi, amount * fx * (1 - fy));
    this.addHeight(xi, yi + 1, amount * (1 - fx) * fy);
    this.addHeight(xi + 1, yi + 1, amount * fx * fy);
  }
  
  erodeAt(x, y, amount) {
    const radius = 3;
    const xi = Math.floor(x);
    const yi = Math.floor(y);
    
    let totalWeight = 0;
    const weights = [];
    
    for (let ey = yi - radius; ey <= yi + radius; ey++) {
      for (let ex = xi - radius; ex <= xi + radius; ex++) {
        const dist = Math.sqrt((ex - x) ** 2 + (ey - y) ** 2);
        if (dist <= radius) {
          const w = 1 - dist / radius;
          weights.push({ x: ex, y: ey, w });
          totalWeight += w;
        }
      }
    }
    
    weights.forEach(({ x, y, w }) => {
      this.addHeight(x, y, -amount * w / totalWeight);
    });
  }
  
  addHeight(x, y, amount) {
    if (x < 0 || x >= this.width || y < 0 || y >= this.height) return;
    const idx = y * this.width + x;
    this.heightmap[idx] = Math.max(0, this.heightmap[idx] + amount);
  }
}
```

## 3. Voxel Terrain (Minecraft-Style)

```javascript
// ===== Voxel System =====

const BLOCK_TYPES = {
  AIR: 0,
  GRASS: 1,
  DIRT: 2,
  STONE: 3,
  SAND: 4,
  WATER: 5,
  WOOD: 6,
  LEAVES: 7
};

class VoxelWorld {
  constructor(width, height, depth) {
    this.width = width;
    this.height = height;
    this.depth = depth;
    this.blocks = new Uint8Array(width * height * depth);
  }
  
  getBlock(x, y, z) {
    if (x < 0 || x >= this.width || y < 0 || y >= this.height || z < 0 || z >= this.depth) {
      return BLOCK_TYPES.AIR;
    }
    return this.blocks[y * this.width * this.depth + z * this.width + x];
  }
  
  setBlock(x, y, z, type) {
    if (x < 0 || x >= this.width || y < 0 || y >= this.height || z < 0 || z >= this.depth) return;
    this.blocks[y * this.width * this.depth + z * this.width + x] = type;
  }
  
  // Generate terrain
  generate(noise, seaLevel = 8) {
    for (let z = 0; z < this.depth; z++) {
      for (let x = 0; x < this.width; x++) {
        const h = noise.fbm(x * 0.05, z * 0.05, 4, 0.5, 2.0);
        const surfaceY = Math.floor((h * 0.5 + 0.5) * (this.height * 0.7));
        
        for (let y = 0; y < this.height; y++) {
          let type = BLOCK_TYPES.AIR;
          
          if (y === 0) {
            type = BLOCK_TYPES.STONE;
          } else if (y < surfaceY - 3) {
            type = BLOCK_TYPES.STONE;
          } else if (y < surfaceY) {
            type = BLOCK_TYPES.DIRT;
          } else if (y === surfaceY) {
            type = surfaceY <= seaLevel ? BLOCK_TYPES.SAND : BLOCK_TYPES.GRASS;
          } else if (y <= seaLevel) {
            type = BLOCK_TYPES.WATER;
          }
          
          this.setBlock(x, y, z, type);
        }
      }
    }
  }
}

// ===== Greedy Meshing =====
// รวม faces ที่ติดกันและมี type เดียวกันเข้าด้วยกัน
// ลด polygon count อย่างมาก

class GreedyMesher {
  constructor(world) {
    this.world = world;
  }
  
  buildMesh() {
    const vertices = [];
    const normals = [];
    const uvs = [];
    const colors = [];
    const indices = [];
    let vertexCount = 0;
    
    const blockColors = {
      [BLOCK_TYPES.GRASS]: [0.2, 0.6, 0.15],
      [BLOCK_TYPES.DIRT]:  [0.45, 0.3, 0.1],
      [BLOCK_TYPES.STONE]: [0.5, 0.5, 0.5],
      [BLOCK_TYPES.SAND]:  [0.85, 0.8, 0.55],
      [BLOCK_TYPES.WATER]: [0.1, 0.5, 0.8],
      [BLOCK_TYPES.WOOD]:  [0.4, 0.25, 0.1],
      [BLOCK_TYPES.LEAVES]:[0.1, 0.4, 0.1],
    };
    
    const FACES = [
      { dir: [0, 1, 0], corners: [[0,1,0],[1,1,0],[1,1,1],[0,1,1]], normal: [0,1,0] }, // top
      { dir: [0,-1, 0], corners: [[0,0,1],[1,0,1],[1,0,0],[0,0,0]], normal: [0,-1,0] }, // bottom
      { dir: [ 1, 0, 0], corners: [[1,0,1],[1,1,1],[1,1,0],[1,0,0]], normal: [1,0,0] }, // right
      { dir: [-1, 0, 0], corners: [[0,0,0],[0,1,0],[0,1,1],[0,0,1]], normal: [-1,0,0] }, // left
      { dir: [0, 0, 1], corners: [[0,0,1],[1,0,1],[1,1,1],[0,1,1]], normal: [0,0,1] }, // front
      { dir: [0, 0,-1], corners: [[1,0,0],[0,0,0],[0,1,0],[1,1,0]], normal: [0,0,-1] }, // back
    ];
    
    const { width, height, depth } = this.world;
    
    for (let y = 0; y < height; y++) {
      for (let z = 0; z < depth; z++) {
        for (let x = 0; x < width; x++) {
          const block = this.world.getBlock(x, y, z);
          if (block === BLOCK_TYPES.AIR) continue;
          
          const color = blockColors[block] || [1, 0, 1];
          
          FACES.forEach(face => {
            const nx = x + face.dir[0];
            const ny = y + face.dir[1];
            const nz = z + face.dir[2];
            
            const neighbor = this.world.getBlock(nx, ny, nz);
            
            // Show face only if neighbor is air or water (and block is not water)
            const showFace = neighbor === BLOCK_TYPES.AIR || 
                           (neighbor === BLOCK_TYPES.WATER && block !== BLOCK_TYPES.WATER);
            
            if (!showFace) return;
            
            // Add face
            const startVertex = vertexCount;
            
            face.corners.forEach(([cx, cy, cz]) => {
              vertices.push(x + cx, y + cy, z + cz);
              normals.push(...face.normal);
              uvs.push(cx, cz);
              
              // Top face lighter, bottom darker
              const shade = face.dir[1] === 1 ? 1.0 : (face.dir[1] === -1 ? 0.5 : 0.8);
              colors.push(color[0] * shade, color[1] * shade, color[2] * shade);
            });
            
            // Two triangles
            indices.push(
              startVertex, startVertex + 1, startVertex + 2,
              startVertex, startVertex + 2, startVertex + 3
            );
            
            vertexCount += 4;
          });
        }
      }
    }
    
    // สร้าง BufferGeometry
    const geo = new THREE.BufferGeometry();
    geo.setAttribute('position', new THREE.Float32BufferAttribute(vertices, 3));
    geo.setAttribute('normal', new THREE.Float32BufferAttribute(normals, 3));
    geo.setAttribute('uv', new THREE.Float32BufferAttribute(uvs, 2));
    geo.setAttribute('color', new THREE.Float32BufferAttribute(colors, 3));
    geo.setIndex(indices);
    geo.computeBoundingBox();
    
    return geo;
  }
}
```

## 4. Marching Cubes Algorithm

```javascript
// ===== Marching Cubes สำหรับ Smooth Terrain =====
// แปลง voxel density field เป็น smooth mesh

// Lookup tables for marching cubes
const MC_EDGE_TABLE = [
  0x0, 0x109, 0x203, 0x30a, 0x406, 0x50f, 0x605, 0x70c,
  // ... (256 entries - ตัดทอนสำหรับ clarity)
];

class MarchingCubes {
  constructor(field, width, height, depth, isoLevel = 0.5) {
    this.field = field;
    this.width = width;
    this.height = height;
    this.depth = depth;
    this.isoLevel = isoLevel;
  }
  
  // ค่า density ที่จุด (x, y, z)
  getDensity(x, y, z) {
    if (x < 0 || x >= this.width || y < 0 || y >= this.height || z < 0 || z >= this.depth) {
      return 0;
    }
    return this.field[z * this.width * this.height + y * this.width + x];
  }
  
  // Interpolate vertex position บน edge
  interpolateVertex(p1, p2, d1, d2) {
    if (Math.abs(d1 - d2) < 0.0001) return p1;
    const t = (this.isoLevel - d1) / (d2 - d1);
    return {
      x: p1.x + t * (p2.x - p1.x),
      y: p1.y + t * (p2.y - p1.y),
      z: p1.z + t * (p2.z - p1.z)
    };
  }
  
  // Triangulate single cube
  triangulateCube(cx, cy, cz) {
    const vertices = [
      { x: cx,   y: cy,   z: cz   },
      { x: cx+1, y: cy,   z: cz   },
      { x: cx+1, y: cy+1, z: cz   },
      { x: cx,   y: cy+1, z: cz   },
      { x: cx,   y: cy,   z: cz+1 },
      { x: cx+1, y: cy,   z: cz+1 },
      { x: cx+1, y: cy+1, z: cz+1 },
      { x: cx,   y: cy+1, z: cz+1 }
    ];
    
    const densities = vertices.map(v => this.getDensity(v.x, v.y, v.z));
    
    // Cube configuration index
    let cubeIndex = 0;
    densities.forEach((d, i) => {
      if (d >= this.isoLevel) cubeIndex |= (1 << i);
    });
    
    if (cubeIndex === 0 || cubeIndex === 255) return [];
    
    // Get edge intersections
    const edgeVertices = new Array(12).fill(null);
    
    const edgePairs = [
      [0,1], [1,2], [2,3], [3,0],
      [4,5], [5,6], [6,7], [7,4],
      [0,4], [1,5], [2,6], [3,7]
    ];
    
    // Simplified edge table (ใช้ full table ใน production)
    edgePairs.forEach(([a, b], i) => {
      const d1 = densities[a], d2 = densities[b];
      if ((d1 < this.isoLevel) !== (d2 < this.isoLevel)) {
        edgeVertices[i] = this.interpolateVertex(vertices[a], vertices[b], d1, d2);
      }
    });
    
    return edgeVertices.filter(v => v !== null);
  }
  
  // Generate full mesh
  generateMesh() {
    const allVertices = [];
    
    for (let z = 0; z < this.depth - 1; z++) {
      for (let y = 0; y < this.height - 1; y++) {
        for (let x = 0; x < this.width - 1; x++) {
          const verts = this.triangulateCube(x, y, z);
          allVertices.push(...verts);
        }
      }
    }
    
    if (allVertices.length === 0) return null;
    
    const positions = new Float32Array(allVertices.length * 3);
    allVertices.forEach((v, i) => {
      positions[i * 3] = v.x;
      positions[i * 3 + 1] = v.y;
      positions[i * 3 + 2] = v.z;
    });
    
    const geo = new THREE.BufferGeometry();
    geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    geo.computeVertexNormals();
    
    return geo;
  }
}

// ===== สร้าง density field จาก noise =====
function createDensityField(width, height, depth, noise) {
  const field = new Float32Array(width * height * depth);
  
  for (let z = 0; z < depth; z++) {
    for (let y = 0; y < height; y++) {
      for (let x = 0; x < width; x++) {
        const idx = z * width * height + y * width + x;
        
        // 3D noise
        const density3D = noise.noise2D(x * 0.1, z * 0.1) * 0.5 +
                         noise.noise2D(y * 0.1 + 100, x * 0.05) * 0.3;
        
        // Gravity (higher y = less dense)
        const surfaceH = 10 + noise.fbm(x * 0.05, z * 0.05, 4, 0.5, 2) * 8;
        const gravityBias = (surfaceH - y) / 3;
        
        field[idx] = density3D + gravityBias * 0.5;
      }
    }
  }
  
  return field;
}
```

## 5. Cave Generation

```javascript
// ===== Cave Generation ด้วย 3D Noise =====
class CaveGenerator {
  constructor(noise) {
    this.noise = noise;
  }
  
  // Density function สำหรับ cave
  getDensity(x, y, z) {
    // Base terrain density (solid below, air above)
    const surfaceH = 15 + this.noise.fbm(x * 0.02, z * 0.02, 4, 0.5, 2) * 10;
    let density = surfaceH - y;
    
    // 3D cave noise (ลดค่าสำหรับสร้างถ้ำ)
    const cave1 = Math.abs(this.noise.noise2D(x * 0.08, y * 0.08 + 1000)) - 0.1;
    const cave2 = Math.abs(this.noise.noise2D(z * 0.08 + 500, y * 0.12 + 2000)) - 0.1;
    const caveIntensity = Math.min(cave1, cave2);
    
    // Worm caves (longer, more natural)
    const wormX = this.noise.fbm(x * 0.05, z * 0.05, 2, 0.5, 2) * 2 + x * 0.05;
    const wormY = y * 0.08;
    const wormDist = Math.abs(this.noise.noise2D(wormX, wormY + 3000));
    const wormCave = 0.3 - wormDist;
    
    // Subtract caves from terrain
    density -= Math.max(0, -caveIntensity * 20);
    density -= Math.max(0, wormCave * 15);
    
    return density;
  }
  
  generateTerrain(width, height, depth) {
    const field = new Float32Array(width * height * depth);
    
    for (let z = 0; z < depth; z++) {
      for (let y = 0; y < height; y++) {
        for (let x = 0; x < width; x++) {
          field[z * width * height + y * width + x] = this.getDensity(x, y, z);
        }
      }
    }
    
    return field;
  }
}
```

## 6. Demo: Voxel World

```html
<!DOCTYPE html>
<html>
<head>
  <title>Voxel World</title>
  <style>
    body { margin: 0; background: #1a3a5a; }
    canvas { display: block; }
    #info {
      position: fixed;
      top: 20px;
      left: 50%;
      transform: translateX(-50%);
      color: #adf;
      font-family: Arial, sans-serif;
      text-align: center;
      pointer-events: none;
    }
    #stats {
      position: fixed;
      bottom: 20px;
      left: 20px;
      color: #8af;
      font-family: monospace;
      font-size: 13px;
    }
  </style>
</head>
<body>
<div id="info">Voxel World - Greedy Meshing Demo</div>
<div id="stats">Vertices: 0 | Faces: 0</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x1a3a5a);
scene.fog = new THREE.FogExp2(0x1a3a5a, 0.02);

const camera = new THREE.PerspectiveCamera(70, innerWidth/innerHeight, 0.1, 200);
camera.position.set(20, 20, 30);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(devicePixelRatio);
renderer.shadowMap.enabled = true;
renderer.outputColorSpace = THREE.SRGBColorSpace;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.target.set(10, 8, 10);

// Lighting
scene.add(new THREE.AmbientLight(0x334466, 3));
const sun = new THREE.DirectionalLight(0xfff8e0, 4);
sun.position.set(30, 50, 20);
sun.castShadow = true;
sun.shadow.mapSize.set(2048, 2048);
sun.shadow.camera.left = -40;
sun.shadow.camera.right = 40;
sun.shadow.camera.top = 40;
sun.shadow.camera.bottom = -40;
scene.add(sun);

// ===== SIMPLE NOISE =====
class Noise {
  constructor(seed = 0) {
    this.seed = seed;
  }
  hash(n) {
    return (Math.sin(n + this.seed * 127.3) * 43758.5453) % 1;
  }
  smooth(t) { return t * t * (3 - 2 * t); }
  
  noise2D(x, y) {
    const ix = Math.floor(x), iy = Math.floor(y);
    const fx = x - ix, fy = y - iy;
    const ux = this.smooth(fx), uy = this.smooth(fy);
    const a = Math.abs(this.hash(ix + iy * 57));
    const b = Math.abs(this.hash(ix + 1 + iy * 57));
    const c = Math.abs(this.hash(ix + (iy + 1) * 57));
    const d = Math.abs(this.hash(ix + 1 + (iy + 1) * 57));
    return a + (b-a)*ux + (c-a)*uy + (a-b-c+d)*ux*uy;
  }
  
  fbm(x, y, octaves = 4) {
    let v = 0, a = 0.5, f = 1;
    for (let i = 0; i < octaves; i++) {
      v += this.noise2D(x * f, y * f) * a;
      a *= 0.5; f *= 2;
    }
    return v;
  }
}

// ===== VOXEL TYPES =====
const TYPES = { AIR:0, GRASS:1, DIRT:2, STONE:3, SAND:4, WATER:5 };
const TYPE_COLORS = {
  [TYPES.GRASS]: [0.22, 0.58, 0.16],
  [TYPES.DIRT]:  [0.42, 0.27, 0.10],
  [TYPES.STONE]: [0.50, 0.48, 0.45],
  [TYPES.SAND]:  [0.83, 0.78, 0.52],
  [TYPES.WATER]: [0.10, 0.48, 0.78],
};

// ===== WORLD =====
const WORLD_W = 32, WORLD_H = 24, WORLD_D = 32;
const SEA_LEVEL = 8;
const noise = new Noise(Math.floor(Math.random() * 1000));

// สร้าง voxel data
const voxels = new Uint8Array(WORLD_W * WORLD_H * WORLD_D);

function getVoxel(x, y, z) {
  if (x < 0 || x >= WORLD_W || y < 0 || y >= WORLD_H || z < 0 || z >= WORLD_D) return TYPES.AIR;
  return voxels[y * WORLD_W * WORLD_D + z * WORLD_W + x];
}

function setVoxel(x, y, z, type) {
  if (x < 0 || x >= WORLD_W || y < 0 || y >= WORLD_H || z < 0 || z >= WORLD_D) return;
  voxels[y * WORLD_W * WORLD_D + z * WORLD_W + x] = type;
}

// Generate world
for (let z = 0; z < WORLD_D; z++) {
  for (let x = 0; x < WORLD_W; x++) {
    const h = Math.floor(SEA_LEVEL + noise.fbm(x * 0.08, z * 0.08) * 10);
    
    for (let y = 0; y < WORLD_H; y++) {
      let type = TYPES.AIR;
      if (y === 0) type = TYPES.STONE;
      else if (y < h - 3) type = TYPES.STONE;
      else if (y < h) type = TYPES.DIRT;
      else if (y === h) type = h <= SEA_LEVEL + 1 ? TYPES.SAND : TYPES.GRASS;
      else if (y <= SEA_LEVEL) type = TYPES.WATER;
      setVoxel(x, y, z, type);
    }
  }
}

// ===== BUILD MESH =====
function buildMesh() {
  const positions = [], normals = [], colors = [], indices = [];
  let vi = 0;
  
  const FACES = [
    { n:[0,1,0],  c:[[0,1,0],[1,1,0],[1,1,1],[0,1,1]], shade:1.0 },
    { n:[0,-1,0], c:[[0,0,1],[1,0,1],[1,0,0],[0,0,0]], shade:0.5 },
    { n:[1,0,0],  c:[[1,0,1],[1,1,1],[1,1,0],[1,0,0]], shade:0.8 },
    { n:[-1,0,0], c:[[0,0,0],[0,1,0],[0,1,1],[0,0,1]], shade:0.7 },
    { n:[0,0,1],  c:[[0,0,1],[1,0,1],[1,1,1],[0,1,1]], shade:0.9 },
    { n:[0,0,-1], c:[[1,0,0],[0,0,0],[0,1,0],[1,1,0]], shade:0.85 },
  ];
  
  for (let y = 0; y < WORLD_H; y++) {
    for (let z = 0; z < WORLD_D; z++) {
      for (let x = 0; x < WORLD_W; x++) {
        const block = getVoxel(x, y, z);
        if (block === TYPES.AIR) continue;
        
        const col = TYPE_COLORS[block] || [1, 0, 1];
        
        FACES.forEach(face => {
          const [nx, ny, nz] = face.n;
          const neighbor = getVoxel(x + nx, y + ny, z + nz);
          
          if (neighbor !== TYPES.AIR && 
              !(neighbor === TYPES.WATER && block !== TYPES.WATER)) return;
          
          face.c.forEach(([cx, cy, cz]) => {
            positions.push(x + cx, y + cy, z + cz);
            normals.push(...face.n);
            const s = face.shade;
            colors.push(col[0] * s, col[1] * s, col[2] * s);
          });
          
          indices.push(vi, vi+1, vi+2, vi, vi+2, vi+3);
          vi += 4;
        });
      }
    }
  }
  
  const geo = new THREE.BufferGeometry();
  geo.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));
  geo.setAttribute('normal', new THREE.Float32BufferAttribute(normals, 3));
  geo.setAttribute('color', new THREE.Float32BufferAttribute(colors, 3));
  geo.setIndex(indices);
  
  document.getElementById('stats').textContent = 
    `Vertices: ${positions.length/3} | Faces: ${indices.length/3}`;
  
  return geo;
}

const geo = buildMesh();
const mat = new THREE.MeshLambertMaterial({ vertexColors: true });
const voxelMesh = new THREE.Mesh(geo, mat);
voxelMesh.castShadow = true;
voxelMesh.receiveShadow = true;
scene.add(voxelMesh);

// Clouds
for (let i = 0; i < 10; i++) {
  const cloud = new THREE.Mesh(
    new THREE.BoxGeometry(4 + Math.random() * 6, 1.5, 3 + Math.random() * 4),
    new THREE.MeshStandardMaterial({ color: 0xffffff, transparent: true, opacity: 0.8 })
  );
  cloud.position.set(
    Math.random() * 32, 20 + Math.random() * 4, Math.random() * 32
  );
  cloud.userData.speed = 0.3 + Math.random() * 0.5;
  scene.add(cloud);
}

// Sky dome
const skyGeo = new THREE.SphereGeometry(150, 16, 16);
const skyMat = new THREE.MeshBasicMaterial({ 
  color: 0x5588cc, 
  side: THREE.BackSide 
});
scene.add(new THREE.Mesh(skyGeo, skyMat));

const clock = new THREE.Clock();

function animate() {
  requestAnimationFrame(animate);
  const elapsed = clock.elapsedTime;
  
  // Animate clouds
  scene.children.forEach(obj => {
    if (obj.userData.speed) {
      obj.position.x += obj.userData.speed * 0.01;
      if (obj.position.x > 50) obj.position.x = -10;
    }
  });
  
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

Advanced Terrain Techniques:

1. **GPU Displacement** - vertex shader สำหรับ dynamic terrain
2. **Hydraulic Erosion** - สมจริงยิ่งขึ้นด้วย simulation
3. **Voxel + Greedy Meshing** - Minecraft-style ที่ optimize
4. **Marching Cubes** - smooth terrain จาก density field
5. **Cave Generation** - 3D noise สำหรับ underground

### Performance Notes
- Greedy Meshing ลด face count ได้ 10-100x
- Chunk-based loading เพื่อ memory efficiency
- LOD system สำหรับ distant terrain

---

## แบบฝึกหัด

1. **Easy**: สร้าง voxel world อย่างง่ายด้วย Greedy Meshing
2. **Medium**: เพิ่ม hydraulic erosion 1000 droplets บน noise terrain
3. **Hard**: สร้าง Marching Cubes implementation สำหรับ smooth caves
4. **Challenge**: สร้าง infinite procedural world ที่โหลด/unload chunks ตาม player position
