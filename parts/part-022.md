# Part 22: Custom BufferGeometry ใน Three.js

## บทนำ

`BufferGeometry` คือหัวใจของ Three.js ทุก geometry ใช้ระบบนี้ภายใน การเข้าใจ BufferGeometry จาก scratch จะทำให้สร้าง shapes ที่ต้องการได้ทุกอย่าง ไม่ว่าจะเป็น terrain, custom models, หรือ procedural meshes

---

## 1. BufferGeometry จาก Scratch

```javascript
// สร้าง triangle เดียวจาก scratch
const geometry = new THREE.BufferGeometry();

// Vertices: 3 จุด (x, y, z) × 3 vertices = 9 values
const positions = new Float32Array([
    // vertex 0
    -1, 0, 0,
    // vertex 1
     1, 0, 0,
    // vertex 2
     0, 1, 0
]);

// กำหนด position attribute
// new THREE.BufferAttribute(array, itemSize)
// itemSize = 3 หมายถึง x, y, z 3 ค่าต่อ vertex
geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));

const material = new THREE.MeshBasicMaterial({
    color: 0x00ff88,
    side: THREE.DoubleSide,
    wireframe: false
});

const mesh = new THREE.Mesh(geometry, material);
scene.add(mesh);
```

---

## 2. BufferAttribute Types

| Array Type | Byte/value | ใช้สำหรับ |
|-----------|-----------|---------|
| `Float32Array` | 4 bytes | position, normal, uv, color |
| `Uint16Array` | 2 bytes | index (สูงสุด 65,535 vertices) |
| `Uint32Array` | 4 bytes | index (มากกว่า 65,535 vertices) |
| `Int8Array` | 1 byte | ค่าเล็กๆ -128 to 127 |
| `Uint8Array` | 1 byte | ค่า 0-255 |

```javascript
// Float32Array - ใช้บ่อยที่สุด
const pos = new Float32Array([0, 0, 0,  1, 0, 0,  0, 1, 0]);

// กำหนดขนาดก่อนแล้วค่อยใส่ค่า
const pos2 = new Float32Array(9); // 3 vertices × 3
pos2[0] = 0; pos2[1] = 0; pos2[2] = 0; // vertex 0
pos2[3] = 1; pos2[4] = 0; pos2[5] = 0; // vertex 1
pos2[6] = 0; pos2[7] = 1; pos2[8] = 0; // vertex 2

// Typed array methods
const copy = pos.slice(0);        // copy
const sub  = pos.subarray(3, 6);  // view ไม่ copy
pos.fill(0);                      // ล้างทั้งหมด
pos.set([1, 0, 0], 3);           // ใส่ค่าที่ offset 3
```

---

## 3. Standard Attributes

### Position
```javascript
// position - required, 3 components per vertex
geometry.setAttribute('position', new THREE.BufferAttribute(
    new Float32Array(vertexCount * 3), 3
));
```

### Normal
```javascript
// normal - สำหรับ lighting calculation, 3 components
const normals = new Float32Array([
    0, 0, 1,   // vertex 0 faces +Z
    0, 0, 1,   // vertex 1 faces +Z
    0, 0, 1    // vertex 2 faces +Z
]);
geometry.setAttribute('normal', new THREE.BufferAttribute(normals, 3));

// หรือให้ Three.js คำนวณให้ (ง่ายกว่า)
geometry.computeVertexNormals();
```

### UV Coordinates
```javascript
// uv - texture coordinates, 2 components, ค่า 0.0 - 1.0
const uvs = new Float32Array([
    0, 0,   // vertex 0: ซ้ายล่าง
    1, 0,   // vertex 1: ขวาล่าง
    0.5, 1  // vertex 2: กลางบน
]);
geometry.setAttribute('uv', new THREE.BufferAttribute(uvs, 2));
```

### Color per Vertex
```javascript
// color - vertex colors, 3 components (r, g, b) ค่า 0.0 - 1.0
const colors = new Float32Array([
    1, 0, 0,   // vertex 0: red
    0, 1, 0,   // vertex 1: green
    0, 0, 1    // vertex 2: blue
]);
geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));

// Material ต้องเปิด vertexColors ด้วย
const mat = new THREE.MeshBasicMaterial({ vertexColors: true });
```

---

## 4. Index Buffer

Index buffer ช่วยประหยัด memory เมื่อหลาย triangles ใช้ vertices เดียวกัน

```javascript
// Non-indexed: 2 triangles = 6 vertices (มี duplicate)
const posNonIndexed = new Float32Array([
    // Triangle 1
    0, 0, 0,   1, 0, 0,   0, 1, 0,
    // Triangle 2 (0,1,0 และ 1,0,0 ซ้ำ)
    1, 0, 0,   1, 1, 0,   0, 1, 0
]);

// Indexed: 2 triangles = 4 vertices (ไม่ซ้ำ)
const posIndexed = new Float32Array([
    0, 0, 0,   // index 0
    1, 0, 0,   // index 1
    1, 1, 0,   // index 2
    0, 1, 0    // index 3
]);

const indices = new Uint16Array([
    0, 1, 3,   // Triangle 1
    1, 2, 3    // Triangle 2
]);

geometry.setAttribute('position', new THREE.BufferAttribute(posIndexed, 3));
geometry.setIndex(new THREE.BufferAttribute(indices, 1));

// หรือใช้ shorthand
geometry.setIndex([0, 1, 3,  1, 2, 3]);

// ประโยชน์:
// - ประหยัด memory เมื่อ vertices ซ้ำ
// - GPU cache ดีกว่าสำหรับ transform
// - computeVertexNormals ทำงานได้ดีกว่า
```

---

## 5. computeVertexNormals vs Manual Normals

```javascript
// Auto compute - สะดวกแต่อาจไม่ถูกต้องสำหรับ hard edges
geometry.computeVertexNormals();

// Manual normals สำหรับ flat shading
function setFlatNormals(geometry) {
    const pos = geometry.attributes.position;
    const normals = new Float32Array(pos.count * 3);
    const vA = new THREE.Vector3();
    const vB = new THREE.Vector3();
    const vC = new THREE.Vector3();
    const normal = new THREE.Vector3();
    
    for (let i = 0; i < pos.count; i += 3) {
        vA.fromBufferAttribute(pos, i);
        vB.fromBufferAttribute(pos, i + 1);
        vC.fromBufferAttribute(pos, i + 2);
        
        // Cross product ของ edges สองเส้น
        vB.sub(vA);
        vC.sub(vA);
        normal.crossVectors(vB, vC).normalize();
        
        // ใส่ normal เดียวกันทั้ง 3 vertices
        normals[i * 3] = normal.x;
        normals[i * 3 + 1] = normal.y;
        normals[i * 3 + 2] = normal.z;
        normals[(i + 1) * 3] = normal.x;
        normals[(i + 1) * 3 + 1] = normal.y;
        normals[(i + 1) * 3 + 2] = normal.z;
        normals[(i + 2) * 3] = normal.x;
        normals[(i + 2) * 3 + 1] = normal.y;
        normals[(i + 2) * 3 + 2] = normal.z;
    }
    
    geometry.setAttribute('normal', new THREE.BufferAttribute(normals, 3));
}
```

---

## 6. computeBoundingSphere และ computeBoundingBox

```javascript
// จำเป็นต้องคำนวณ bounding volumes สำหรับ frustum culling
geometry.computeBoundingSphere();
geometry.computeBoundingBox();

// อ่านค่า
const sphere = geometry.boundingSphere;
console.log('Center:', sphere.center, 'Radius:', sphere.radius);

const box = geometry.boundingBox;
console.log('Min:', box.min, 'Max:', box.max);

// กำหนดเอง
geometry.boundingSphere = new THREE.Sphere(new THREE.Vector3(0, 0, 0), 10);

// Three.js คำนวณ bounding ใหม่อัตโนมัติเมื่อ render
// แต่ถ้า update positions ใน loop ต้อง call เอง
function updateGeometry() {
    // ... อัพเดต positions ...
    geometry.attributes.position.needsUpdate = true;
    geometry.computeBoundingSphere();  // ← จำเป็นสำหรับ frustum culling
}
```

---

## 7. Procedural Terrain

```javascript
// สร้าง terrain mesh จาก heightmap
function createTerrain(width, depth, segments, heightFn) {
    const wSegs = segments;
    const dSegs = segments;
    
    const vertexCount = (wSegs + 1) * (dSegs + 1);
    const indexCount = wSegs * dSegs * 6;
    
    const positions = new Float32Array(vertexCount * 3);
    const normals = new Float32Array(vertexCount * 3);
    const uvs = new Float32Array(vertexCount * 2);
    const indices = new Uint32Array(indexCount);
    
    // สร้าง vertices
    let vIdx = 0, uvIdx = 0;
    for (let z = 0; z <= dSegs; z++) {
        for (let x = 0; x <= wSegs; x++) {
            const nx = x / wSegs;  // 0..1
            const nz = z / dSegs;  // 0..1
            
            const px = (nx - 0.5) * width;
            const pz = (nz - 0.5) * depth;
            const py = heightFn(px, pz);
            
            positions[vIdx] = px;
            positions[vIdx + 1] = py;
            positions[vIdx + 2] = pz;
            
            uvs[uvIdx] = nx;
            uvs[uvIdx + 1] = 1 - nz;  // flip V
            
            vIdx += 3;
            uvIdx += 2;
        }
    }
    
    // สร้าง indices (2 triangles per quad)
    let iIdx = 0;
    for (let z = 0; z < dSegs; z++) {
        for (let x = 0; x < wSegs; x++) {
            const a = z * (wSegs + 1) + x;
            const b = a + 1;
            const c = a + (wSegs + 1);
            const d = c + 1;
            
            // Triangle 1
            indices[iIdx++] = a;
            indices[iIdx++] = c;
            indices[iIdx++] = b;
            
            // Triangle 2
            indices[iIdx++] = b;
            indices[iIdx++] = c;
            indices[iIdx++] = d;
        }
    }
    
    const geometry = new THREE.BufferGeometry();
    geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    geometry.setAttribute('uv', new THREE.BufferAttribute(uvs, 2));
    geometry.setIndex(new THREE.BufferAttribute(indices, 1));
    geometry.computeVertexNormals();
    geometry.computeBoundingSphere();
    
    return geometry;
}

// Simplex noise height function (ใช้ library จริงควรใช้ simplex-noise)
function hills(x, z) {
    return Math.sin(x * 0.3) * Math.cos(z * 0.3) * 3
         + Math.sin(x * 0.7 + 1) * Math.cos(z * 0.5) * 1.5
         + Math.sin(x * 1.5) * Math.cos(z * 1.2) * 0.5;
}

const terrainGeo = createTerrain(40, 40, 64, hills);
const terrainMat = new THREE.MeshStandardMaterial({
    color: 0x4a7c4e,
    roughness: 0.9,
    metalness: 0,
    wireframe: false
});

const terrain = new THREE.Mesh(terrainGeo, terrainMat);
scene.add(terrain);
```

---

## 8. Heart Shape Geometry

```javascript
// สร้าง heart shape geometry
function createHeartGeometry(size = 1, depth = 0.3) {
    const shape = new THREE.Shape();
    
    // Heart curve using bezier
    const s = size;
    shape.moveTo(0, -s * 0.3);
    
    // ซ้ายล่าง
    shape.bezierCurveTo(-s * 0.5, -s * 0.8, -s, -s * 0.3, -s, s * 0.1);
    
    // ซ้ายบน (โค้ง)
    shape.bezierCurveTo(-s, s * 0.6, -s * 0.5, s * 0.9, 0, s * 0.5);
    
    // ขวาบน (โค้ง)
    shape.bezierCurveTo(s * 0.5, s * 0.9, s, s * 0.6, s, s * 0.1);
    
    // ขวาล่าง
    shape.bezierCurveTo(s, -s * 0.3, s * 0.5, -s * 0.8, 0, -s * 0.3);
    
    const extrudeSettings = {
        depth: depth,
        bevelEnabled: true,
        bevelThickness: 0.05,
        bevelSize: 0.05,
        bevelSegments: 3
    };
    
    return new THREE.ExtrudeGeometry(shape, extrudeSettings);
}

const heartGeo = createHeartGeometry(1, 0.2);
heartGeo.center(); // center geometry

const heartMat = new THREE.MeshStandardMaterial({
    color: 0xff1177,
    roughness: 0.2,
    metalness: 0.3
});

const heart = new THREE.Mesh(heartGeo, heartMat);
scene.add(heart);
```

---

## 9. Custom Geometry: Wave Plane

```javascript
// Wave plane ที่ animated ด้วย vertex positions
class WavePlane {
    constructor(width, height, widthSegs, heightSegs) {
        this.widthSegs = widthSegs;
        this.heightSegs = heightSegs;
        
        this.geometry = new THREE.PlaneGeometry(width, height, widthSegs, heightSegs);
        this.geometry.rotateX(-Math.PI / 2);
        
        // เก็บ original positions
        const pos = this.geometry.attributes.position;
        this.originalY = new Float32Array(pos.count);
        for (let i = 0; i < pos.count; i++) {
            this.originalY[i] = pos.getY(i);
        }
        
        this.material = new THREE.MeshStandardMaterial({
            color: 0x0077be,
            roughness: 0.1,
            metalness: 0.6,
            side: THREE.DoubleSide
        });
        
        this.mesh = new THREE.Mesh(this.geometry, this.material);
    }
    
    update(elapsed) {
        const pos = this.geometry.attributes.position;
        
        for (let i = 0; i < pos.count; i++) {
            const x = pos.getX(i);
            const z = pos.getZ(i);
            
            // Sine wave combination
            const wave = Math.sin(x * 0.5 + elapsed * 2) * 0.5
                       + Math.sin(z * 0.3 + elapsed * 1.5) * 0.3
                       + Math.sin((x + z) * 0.2 + elapsed) * 0.2;
            
            pos.setY(i, this.originalY[i] + wave);
        }
        
        pos.needsUpdate = true;
        this.geometry.computeVertexNormals(); // recompute normals ทุก frame
    }
}
```

---

## 10. mergeGeometries

```javascript
// รวมหลาย geometries เป็นอันเดียวเพื่อลด draw calls
import { mergeGeometries } from 'https://unpkg.com/three@0.158.0/examples/jsm/utils/BufferGeometryUtils.js';

// สร้าง geometries หลายอัน
const geos = [];
for (let i = 0; i < 100; i++) {
    const geo = new THREE.BoxGeometry(0.5, 0.5, 0.5);
    // เลื่อน geometry position ก่อน merge (ไม่ใช่ mesh position)
    geo.translate(
        (Math.random() - 0.5) * 20,
        Math.random() * 5,
        (Math.random() - 0.5) * 20
    );
    geos.push(geo);
}

// Merge เป็น geometry เดียว
const merged = mergeGeometries(geos);

const mergedMesh = new THREE.Mesh(
    merged,
    new THREE.MeshStandardMaterial({ color: 0x88aaff })
);
scene.add(mergedMesh);

// ล้าง source geometries
geos.forEach(g => g.dispose());

// ข้อควรระวัง:
// - merge แล้วแยกไม่ได้ (ต้องการข้อมูลเพิ่มถ้าอยากแยก)
// - ใช้ groups ถ้าต้องการหลาย materials
// - InstancedMesh ดีกว่าถ้า geometry เหมือนกัน
```

---

## 11. Geometry Groups (Multiple Materials)

```javascript
// สร้าง geometry ที่ใช้หลาย materials
const geo = new THREE.BufferGeometry();

const positions = new Float32Array([
    // Top face (group 0)
    -1, 1, -1,   1, 1, -1,   1, 1,  1,
    -1, 1, -1,   1, 1,  1,  -1, 1,  1,
    // Bottom face (group 1)
    -1,-1,  1,   1,-1,  1,   1,-1, -1,
    -1,-1,  1,   1,-1, -1,  -1,-1, -1
]);

geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));

// กำหนด groups: addGroup(start, count, materialIndex)
geo.addGroup(0, 6, 0);   // 6 indices สำหรับ material 0
geo.addGroup(6, 6, 1);   // 6 indices สำหรับ material 1

geo.computeVertexNormals();

// ต้องใช้ array of materials
const mesh = new THREE.Mesh(geo, [
    new THREE.MeshStandardMaterial({ color: 0xff4444 }),  // material index 0
    new THREE.MeshStandardMaterial({ color: 0x4444ff })   // material index 1
]);
scene.add(mesh);
```

---

## 12. Complete Demo: Custom BufferGeometry Showcase

```html
<!DOCTYPE html>
<html>
<head>
    <title>Custom BufferGeometry Demo</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #0a0a1a; overflow: hidden; font-family: monospace; }
        
        #ui {
            position: absolute;
            top: 10px;
            right: 10px;
            background: rgba(0,0,0,0.8);
            color: white;
            padding: 15px;
            border-radius: 8px;
            width: 220px;
        }
        
        h3 { color: #4ecdc4; margin-bottom: 8px; font-size: 14px; }
        
        .section {
            margin-bottom: 12px;
            border-bottom: 1px solid #333;
            padding-bottom: 8px;
        }
        
        label { display: block; font-size: 11px; color: #aaa; margin: 4px 0 2px; }
        
        input[type=range] { width: 100%; margin-bottom: 4px; }
        
        .row { display: flex; gap: 6px; margin: 4px 0; }
        
        .btn {
            flex: 1;
            padding: 6px;
            background: #1a2a3a;
            color: white;
            border: 1px solid #4ecdc4;
            border-radius: 4px;
            cursor: pointer;
            font-family: monospace;
            font-size: 11px;
        }
        .btn:hover { background: #4ecdc4; color: #000; }
        .btn.active { background: #e94560; border-color: #e94560; }
        
        #info {
            position: absolute;
            bottom: 10px;
            left: 10px;
            color: #4ecdc4;
            font-size: 12px;
        }
    </style>
</head>
<body>

<div id="ui">
    <h3>Custom Geometry</h3>
    
    <div class="section">
        <div class="row">
            <button class="btn active" id="btnTriangle">Triangle</button>
            <button class="btn" id="btnTerrain">Terrain</button>
        </div>
        <div class="row">
            <button class="btn" id="btnHeart">Heart</button>
            <button class="btn" id="btnWave">Wave</button>
        </div>
    </div>
    
    <div class="section" id="terrainControls" style="display:none;">
        <label>Segments: <span id="segVal">32</span></label>
        <input type="range" id="segments" min="4" max="128" value="32" step="4">
        <label>Height Scale: <span id="hVal">3.0</span></label>
        <input type="range" id="heightScale" min="0" max="10" value="3" step="0.1">
        <label>Wireframe</label>
        <input type="checkbox" id="wireframe"> เปิด
    </div>
    
    <div class="section">
        <label>Rotation Speed</label>
        <input type="range" id="rotSpeed" min="0" max="2" value="0.5" step="0.1">
    </div>
    
    <div class="section">
        <button class="btn" id="btnNormals">Show Normals</button>
    </div>
</div>

<div id="info">Vertices: 0 | Triangles: 0</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';
import { VertexNormalsHelper } from 'https://unpkg.com/three@0.158.0/examples/jsm/helpers/VertexNormalsHelper.js';
import { mergeGeometries } from 'https://unpkg.com/three@0.158.0/examples/jsm/utils/BufferGeometryUtils.js';

// Setup
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x0a0a1a);
scene.add(new THREE.GridHelper(20, 20, 0x222244, 0x111133));

const camera = new THREE.PerspectiveCamera(75, innerWidth / innerHeight, 0.1, 1000);
camera.position.set(0, 10, 20);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(devicePixelRatio);
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

// Lighting
scene.add(new THREE.AmbientLight(0x334466, 1));
const dirLight = new THREE.DirectionalLight(0xffffff, 2);
dirLight.position.set(10, 20, 10);
scene.add(dirLight);
const dirLight2 = new THREE.DirectionalLight(0x4499ff, 1);
dirLight2.position.set(-10, 5, -10);
scene.add(dirLight2);

// State
let currentMesh = null;
let normalsHelper = null;
let rotSpeed = 0.5;
let showNormals = false;
let currentType = 'triangle';
let terrainParams = { segments: 32, heightScale: 3 };

// ========== GEOMETRY CREATORS ==========

function createTriangle() {
    const geo = new THREE.BufferGeometry();
    
    const positions = new Float32Array([
        -2, 0, 0,
         2, 0, 0,
         0, 3, 0
    ]);
    
    const uvs = new Float32Array([
        0, 0,
        1, 0,
        0.5, 1
    ]);
    
    const colors = new Float32Array([
        1, 0.2, 0.2,  // red
        0.2, 1, 0.2,  // green
        0.2, 0.2, 1   // blue
    ]);
    
    geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    geo.setAttribute('uv', new THREE.BufferAttribute(uvs, 2));
    geo.setAttribute('color', new THREE.BufferAttribute(colors, 3));
    geo.computeVertexNormals();
    geo.computeBoundingSphere();
    
    return geo;
}

function createTerrain(segs, heightScale) {
    const size = 16;
    const vertexCount = (segs + 1) * (segs + 1);
    
    const positions = new Float32Array(vertexCount * 3);
    const uvs = new Float32Array(vertexCount * 2);
    const indices = new Uint32Array(segs * segs * 6);
    
    let vIdx = 0, uvIdx = 0;
    for (let z = 0; z <= segs; z++) {
        for (let x = 0; x <= segs; x++) {
            const nx = x / segs;
            const nz = z / segs;
            const px = (nx - 0.5) * size;
            const pz = (nz - 0.5) * size;
            
            // Combined waves
            const py = (
                Math.sin(px * 0.4) * Math.cos(pz * 0.4) +
                Math.sin(px * 0.9 + 1) * Math.cos(pz * 0.6) * 0.5 +
                Math.sin(px * 1.8) * Math.cos(pz * 1.4) * 0.25
            ) * heightScale;
            
            positions[vIdx] = px;
            positions[vIdx + 1] = py;
            positions[vIdx + 2] = pz;
            
            uvs[uvIdx] = nx;
            uvs[uvIdx + 1] = 1 - nz;
            
            vIdx += 3;
            uvIdx += 2;
        }
    }
    
    let iIdx = 0;
    for (let z = 0; z < segs; z++) {
        for (let x = 0; x < segs; x++) {
            const a = z * (segs + 1) + x;
            const b = a + 1;
            const c = a + segs + 1;
            const d = c + 1;
            indices[iIdx++] = a; indices[iIdx++] = c; indices[iIdx++] = b;
            indices[iIdx++] = b; indices[iIdx++] = c; indices[iIdx++] = d;
        }
    }
    
    const geo = new THREE.BufferGeometry();
    geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    geo.setAttribute('uv', new THREE.BufferAttribute(uvs, 2));
    geo.setIndex(new THREE.BufferAttribute(indices, 1));
    geo.computeVertexNormals();
    geo.computeBoundingSphere();
    
    return geo;
}

function createHeart() {
    const shape = new THREE.Shape();
    const s = 2;
    
    shape.moveTo(0, -s * 0.3);
    shape.bezierCurveTo(-s * 0.5, -s * 0.8, -s, -s * 0.3, -s, s * 0.1);
    shape.bezierCurveTo(-s, s * 0.6, -s * 0.5, s * 0.9, 0, s * 0.5);
    shape.bezierCurveTo(s * 0.5, s * 0.9, s, s * 0.6, s, s * 0.1);
    shape.bezierCurveTo(s, -s * 0.3, s * 0.5, -s * 0.8, 0, -s * 0.3);
    
    const extrudeSettings = {
        depth: 0.5,
        bevelEnabled: true,
        bevelThickness: 0.1,
        bevelSize: 0.1,
        bevelSegments: 4
    };
    
    const geo = new THREE.ExtrudeGeometry(shape, extrudeSettings);
    geo.center();
    geo.computeVertexNormals();
    geo.computeBoundingSphere();
    return geo;
}

// ========== MESH MANAGEMENT ==========

function getMaterial(type) {
    switch(type) {
        case 'triangle':
            return new THREE.MeshStandardMaterial({ vertexColors: true, side: THREE.DoubleSide, roughness: 0.3, metalness: 0.5 });
        case 'terrain':
            return new THREE.MeshStandardMaterial({
                color: 0x4a7c4e,
                roughness: 0.8,
                metalness: 0,
                wireframe: document.getElementById('wireframe').checked
            });
        case 'heart':
            return new THREE.MeshStandardMaterial({ color: 0xff1177, roughness: 0.2, metalness: 0.4 });
        case 'wave':
            return new THREE.MeshStandardMaterial({ color: 0x0077be, roughness: 0.1, metalness: 0.7, side: THREE.DoubleSide });
        default:
            return new THREE.MeshStandardMaterial({ color: 0x88aaff });
    }
}

function switchTo(type) {
    currentType = type;
    
    // ลบ mesh เดิม
    if (currentMesh) {
        if (normalsHelper) { scene.remove(normalsHelper); normalsHelper = null; }
        scene.remove(currentMesh);
        currentMesh.geometry.dispose();
        currentMesh.material.dispose();
        currentMesh = null;
    }
    
    let geo;
    const mat = getMaterial(type);
    
    switch(type) {
        case 'triangle': geo = createTriangle(); break;
        case 'terrain':  geo = createTerrain(terrainParams.segments, terrainParams.heightScale); break;
        case 'heart':    geo = createHeart(); break;
        case 'wave':
            geo = new THREE.PlaneGeometry(16, 16, 64, 64);
            geo.rotateX(-Math.PI / 2);
            break;
    }
    
    currentMesh = new THREE.Mesh(geo, mat);
    scene.add(currentMesh);
    
    if (showNormals) addNormalsHelper();
    updateInfo();
    
    document.getElementById('terrainControls').style.display = type === 'terrain' ? 'block' : 'none';
}

function addNormalsHelper() {
    if (normalsHelper) { scene.remove(normalsHelper); normalsHelper = null; }
    if (currentMesh) {
        normalsHelper = new VertexNormalsHelper(currentMesh, 0.3, 0x00ff00);
        scene.add(normalsHelper);
    }
}

function updateInfo() {
    if (!currentMesh) return;
    const geo = currentMesh.geometry;
    const pos = geo.attributes.position;
    const idx = geo.index;
    const tris = idx ? idx.count / 3 : pos.count / 3;
    document.getElementById('info').textContent = 
        `Vertices: ${pos.count.toLocaleString()} | Triangles: ${Math.floor(tris).toLocaleString()}`;
}

// ========== UI EVENTS ==========

['triangle', 'terrain', 'heart', 'wave'].forEach(type => {
    document.getElementById(`btn${type.charAt(0).toUpperCase() + type.slice(1)}`).addEventListener('click', () => {
        document.querySelectorAll('.btn').forEach(b => {
            if (['btnTriangle', 'btnTerrain', 'btnHeart', 'btnWave'].includes(b.id)) b.classList.remove('active');
        });
        document.getElementById(`btn${type.charAt(0).toUpperCase() + type.slice(1)}`).classList.add('active');
        switchTo(type);
    });
});

document.getElementById('segments').addEventListener('input', (e) => {
    terrainParams.segments = parseInt(e.target.value);
    document.getElementById('segVal').textContent = terrainParams.segments;
    if (currentType === 'terrain') switchTo('terrain');
});

document.getElementById('heightScale').addEventListener('input', (e) => {
    terrainParams.heightScale = parseFloat(e.target.value);
    document.getElementById('hVal').textContent = terrainParams.heightScale.toFixed(1);
    if (currentType === 'terrain') switchTo('terrain');
});

document.getElementById('wireframe').addEventListener('change', (e) => {
    if (currentMesh) currentMesh.material.wireframe = e.target.checked;
});

document.getElementById('rotSpeed').addEventListener('input', (e) => {
    rotSpeed = parseFloat(e.target.value);
});

document.getElementById('btnNormals').addEventListener('click', () => {
    showNormals = !showNormals;
    document.getElementById('btnNormals').textContent = showNormals ? 'Hide Normals' : 'Show Normals';
    if (showNormals) {
        addNormalsHelper();
    } else if (normalsHelper) {
        scene.remove(normalsHelper);
        normalsHelper = null;
    }
});

window.addEventListener('resize', () => {
    camera.aspect = innerWidth / innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(innerWidth, innerHeight);
});

// ========== ANIMATE ==========
const clock = new THREE.Clock();

switchTo('triangle');

function animate() {
    requestAnimationFrame(animate);
    const delta = clock.getDelta();
    const elapsed = clock.getElapsedTime();
    
    if (currentMesh) {
        currentMesh.rotation.y += rotSpeed * delta;
        
        // Wave animation
        if (currentType === 'wave') {
            const pos = currentMesh.geometry.attributes.position;
            for (let i = 0; i < pos.count; i++) {
                const x = pos.getX(i);
                const z = pos.getZ(i);
                const y = Math.sin(x * 0.5 + elapsed * 2) * 0.5
                        + Math.sin(z * 0.3 + elapsed * 1.5) * 0.3;
                pos.setY(i, y);
            }
            pos.needsUpdate = true;
            currentMesh.geometry.computeVertexNormals();
            if (normalsHelper) normalsHelper.update();
        }
    }
    
    controls.update();
    renderer.render(scene, camera);
}
animate();
</script>
</body>
</html>
```

---

## 13. Dynamic Geometry Update Pattern

```javascript
// Pattern สำหรับ geometry ที่เปลี่ยนทุก frame
class DynamicGeometry {
    constructor(maxVertices) {
        this.maxVertices = maxVertices;
        this.geometry = new THREE.BufferGeometry();
        
        // ใช้ DynamicDrawUsage บอก GPU ว่าจะอัพเดตบ่อย
        const posAttr = new THREE.BufferAttribute(
            new Float32Array(maxVertices * 3), 3
        );
        posAttr.setUsage(THREE.DynamicDrawUsage);
        
        const normalAttr = new THREE.BufferAttribute(
            new Float32Array(maxVertices * 3), 3
        );
        normalAttr.setUsage(THREE.DynamicDrawUsage);
        
        this.geometry.setAttribute('position', posAttr);
        this.geometry.setAttribute('normal', normalAttr);
        
        // setDrawRange เพื่อ render เฉพาะส่วนที่ใช้
        this.geometry.setDrawRange(0, 0);
        this.currentCount = 0;
    }
    
    reset() {
        this.currentCount = 0;
    }
    
    addVertex(x, y, z, nx = 0, ny = 1, nz = 0) {
        if (this.currentCount >= this.maxVertices) return;
        
        const pos = this.geometry.attributes.position;
        const nor = this.geometry.attributes.normal;
        
        pos.setXYZ(this.currentCount, x, y, z);
        nor.setXYZ(this.currentCount, nx, ny, nz);
        
        this.currentCount++;
    }
    
    flush() {
        this.geometry.setDrawRange(0, this.currentCount);
        this.geometry.attributes.position.needsUpdate = true;
        this.geometry.attributes.normal.needsUpdate = true;
        this.geometry.computeBoundingSphere();
    }
}
```

---

## 14. Performance Tips

```javascript
// 1. DynamicDrawUsage สำหรับ buffers ที่เปลี่ยนบ่อย
bufferAttribute.setUsage(THREE.DynamicDrawUsage);   // อัพเดตบ่อย
bufferAttribute.setUsage(THREE.StaticDrawUsage);    // สร้างครั้งเดียว (default)
bufferAttribute.setUsage(THREE.StreamDrawUsage);    // เปลี่ยนทุก frame

// 2. updateRange สำหรับอัพเดตบางส่วน
attribute.updateRange.offset = startIndex * 3;  // byte offset
attribute.updateRange.count = numElements * 3;  // จำนวน elements
attribute.needsUpdate = true;

// 3. Frustum culling
mesh.frustumCulled = true;  // default true
geometry.computeBoundingSphere(); // ต้องมีเพื่อ frustum culling ทำงาน

// 4. Indexed geometry ดีกว่า non-indexed
// - GPU vertex cache hit rate สูงกว่า
// - ใช้ memory น้อยกว่าเมื่อ vertices ซ้ำ

// 5. Dispose เมื่อไม่ใช้
geometry.dispose();
material.dispose();
texture.dispose();

// 6. Merge static geometries
const merged = mergeGeometries(staticGeos);
// ลด draw calls จาก N เป็น 1

// 7. LOD สำหรับ terrain ไกล
const lod = new THREE.LOD();
lod.addLevel(highDetailMesh, 0);
lod.addLevel(medDetailMesh, 50);
lod.addLevel(lowDetailMesh, 200);
scene.add(lod);
```

---

## 15. Memory Layout ของ BufferGeometry

```
Float32Array สำหรับ 3 vertices:
Index:  0    1    2    3    4    5    6    7    8
Value: [x0]  [y0]  [z0]  [x1]  [y1]  [z1]  [x2]  [y2]  [z2]
        ←── vertex 0 ──→  ←── vertex 1 ──→  ←── vertex 2 ──→

itemSize = 3 หมายถึงแต่ละ vertex ใช้ 3 ค่า

Memory: 3 vertices × 3 components × 4 bytes = 36 bytes

สำหรับ 1 million vertices:
position:  1M × 3 × 4 = 12 MB
normal:    1M × 3 × 4 = 12 MB
uv:        1M × 2 × 4 = 8 MB
color:     1M × 3 × 4 = 12 MB
index:     1M × 4     = 4 MB (Uint32Array)
รวม: ~48 MB
```

---

## แบบฝึกหัด

1. **Custom Sphere**: สร้าง sphere geometry เอง โดยใช้ spherical coordinates (phi, theta)
2. **Torus**: สร้าง torus (donut) จาก scratch ด้วย trigonometry
3. **Animated Cloth**: Grid plane ที่จำลอง cloth physics ด้วย spring forces
4. **Voxel Terrain**: สร้าง minecraft-style terrain จาก merged cube geometries
5. **Morph Targets**: สร้าง geometry สองแบบแล้วทำ morph animation

---

## สรุป

- `BufferGeometry` ใช้ `Float32Array` สำหรับ vertex data ทำให้ GPU-friendly
- Standard attributes: `position` (required), `normal`, `uv`, `color`
- Index buffer ประหยัด memory เมื่อ vertices ซ้ำ (indexed geometry)
- `computeVertexNormals()` คำนวณ normals อัตโนมัติ (เหมาะกับ smooth shading)
- `computeBoundingSphere()` จำเป็นสำหรับ frustum culling
- `needsUpdate = true` แจ้ง GPU เมื่อ buffer เปลี่ยน
- `setDrawRange(start, count)` ควบคุมจำนวน vertices ที่ render
- `DynamicDrawUsage` ใช้เมื่ออัพเดต buffer บ่อยๆ
- `mergeGeometries()` รวม static meshes เพื่อลด draw calls
- ทุก built-in geometry ของ Three.js ใช้ `BufferGeometry` ภายใน
