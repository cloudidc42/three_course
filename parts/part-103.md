# Part 103: Geometry Merging (Advanced)

## บทนำ: ทำไมต้อง Merge Geometry

ในบทที่แล้วเราได้เรียนรู้เรื่อง Draw Call Optimization และได้กล่าวถึง `mergeGeometries` ไปบ้างแล้ว บทนี้เราจะเจาะลึกลงไปถึงวิธีการทำงานของ Geometry Merging อย่างละเอียด รวมถึงเทคนิคขั้นสูงและข้อควรระวัง

**Geometry Merging** คือการรวม BufferGeometry หลายชิ้นเป็น geometry ชิ้นเดียว ทำให้ GPU ต้องทำ Draw Call เพียงครั้งเดียวแทนที่จะต้องทำหลายครั้ง

**เมื่อใดควรใช้ Merge:**
- Objects ที่ไม่เคลื่อนไหว (static scene elements)
- Objects ที่ไม่ถูก add/remove บ่อยๆ
- Objects ที่ไม่ต้องการ individual frustum culling
- Background elements เช่น buildings, rocks, trees ระยะไกล

**เมื่อใดไม่ควรใช้ Merge:**
- Objects ที่มีการเคลื่อนไหว (animated)
- Objects ที่ถูก add/remove บ่อยๆ
- Objects ที่ต้องการ per-object culling เพื่อ performance
- Objects ที่ต้องการ per-object interaction (click, hover)

---

## 1. BufferGeometryUtils.mergeGeometries Deep Dive

```javascript
import { mergeGeometries } from 'three/addons/utils/BufferGeometryUtils.js';

// Signature:
// mergeGeometries(geometries: BufferGeometry[], useGroups: boolean = false): BufferGeometry

// พารามิเตอร์:
// - geometries: array ของ BufferGeometry ที่ต้องการ merge
// - useGroups: ถ้า true จะสร้าง geometry groups สำหรับแต่ละ input geometry
//   ทำให้ใช้หลาย material กับ merged geometry ได้

// ข้อกำหนด:
// 1. Geometry ทุกชิ้นต้องมี attributes เหมือนกัน (position, normal, uv ต้องมีครบทุกชิ้น)
// 2. ถ้า geometry ไหนมี attribute พิเศษ (เช่น color) ทุกชิ้นต้องมีด้วย
// 3. Index buffer: ถ้าชิ้นหนึ่งมี index อีกชิ้นต้องมีเหมือนกัน (หรือ non-indexed ทั้งหมด)
```

### การตรวจสอบ Attributes ก่อน Merge

```javascript
function prepareForMerge(geometries) {
    // ตรวจสอบว่าทุก geometry มี attributes เดียวกัน
    const referenceAttributes = Object.keys(geometries[0].attributes);
    
    return geometries.map(geo => {
        // Clone เพื่อไม่ modify ต้นฉบับ
        const cloned = geo.clone();
        
        // เพิ่ม attributes ที่ขาดหายไป
        referenceAttributes.forEach(name => {
            if (!cloned.attributes[name]) {
                const refAttr = geometries[0].attributes[name];
                const count = cloned.attributes.position.count;
                const itemSize = refAttr.itemSize;
                
                // สร้าง default values (zeros)
                const zeros = new Float32Array(count * itemSize);
                cloned.setAttribute(name, new THREE.BufferAttribute(zeros, itemSize));
            }
        });
        
        return cloned;
    });
}
```

---

## 2. Merging with Multiple Materials (Geometry Groups)

```javascript
import { mergeGeometries } from 'three/addons/utils/BufferGeometryUtils.js';

// useGroups = true จะสร้าง DrawRange groups ให้แต่ละ geometry
// ทำให้ mesh ใช้หลาย material ได้

const buildingGeos = [];
const roadGeos = [];
const groundGeos = [];

// สร้าง building geometries
for (let i = 0; i < 50; i++) {
    const geo = new THREE.BoxGeometry(3, 8, 3);
    geo.applyMatrix4(new THREE.Matrix4().setPosition(i * 10 - 250, 4, 0));
    buildingGeos.push(geo);
}

// สร้าง road geometries  
for (let i = 0; i < 20; i++) {
    const geo = new THREE.PlaneGeometry(8, 20);
    geo.rotateX(-Math.PI / 2);
    geo.applyMatrix4(new THREE.Matrix4().setPosition(i * 10 - 250, 0.01, 10));
    roadGeos.push(geo);
}

// Merge แต่ละประเภทแยกกัน
const buildingMerged = mergeGeometries(buildingGeos);
const roadMerged = mergeGeometries(roadGeos);

// Merge ทั้งหมดเป็น geometry เดียว ด้วย useGroups = true
const allGeometries = [buildingMerged, roadMerged];
const cityGeometry = mergeGeometries(allGeometries, true);

// ตอนนี้ cityGeometry.groups จะมี 2 groups
console.log(cityGeometry.groups);
// [
//   { start: 0, count: buildingFaceCount, materialIndex: 0 },
//   { start: buildingFaceCount, count: roadFaceCount, materialIndex: 1 }
// ]

// ใช้ array ของ materials
const materials = [
    new THREE.MeshStandardMaterial({ color: 0x4466aa }),  // buildings
    new THREE.MeshStandardMaterial({ color: 0x333333 })   // roads
];

const cityMesh = new THREE.Mesh(cityGeometry, materials);
scene.add(cityMesh);
// ผลลัพธ์: 2 draw calls แทน 70!
```

---

## 3. StaticDrawUsage Hint

การกำหนด `DrawUsage` บอก GPU ว่า buffer จะถูก update บ่อยแค่ไหน ช่วยให้ GPU จัดการ memory ได้อย่างมีประสิทธิภาพ:

```javascript
import * as THREE from 'three';

// ค่าที่มีให้เลือก:
// THREE.StaticDrawUsage  - buffer ไม่เปลี่ยนแปลง (default ใน three.js ส่วนใหญ่)
// THREE.DynamicDrawUsage - buffer เปลี่ยนแปลงบ่อย
// THREE.StreamDrawUsage  - buffer เปลี่ยนแปลงทุก frame

// หลังจาก merge geometry ให้ set StaticDrawUsage เพื่อ optimize memory
const mergedGeo = mergeGeometries(geos);

mergedGeo.attributes.position.usage = THREE.StaticDrawUsage;
mergedGeo.attributes.normal.usage = THREE.StaticDrawUsage;
mergedGeo.attributes.uv.usage = THREE.StaticDrawUsage;
if (mergedGeo.index) {
    mergedGeo.index.usage = THREE.StaticDrawUsage;
}

// สำหรับ geometry ที่จะ update:
dynamicGeo.attributes.position.usage = THREE.DynamicDrawUsage;

// หลังจาก upload ไป GPU แล้ว เราสามารถ free RAM ได้ด้วย:
// (three.js จะ upload ไปที่ GPU ในการ render ครั้งแรก)
function uploadAndFreeRAM(geometry) {
    // ต้อง render ก่อนสักครั้งเพื่อ upload
    // จากนั้น:
    geometry.dispose(); // ลบ CPU copy
    // GPU ยังคง retain ข้อมูลไว้
}
```

---

## 4. Handling Per-Instance Data: Vertex Colors

เมื่อ merge geometry เราสูญเสียความสามารถในการกำหนดค่าต่างกันสำหรับแต่ละ object แต่สามารถ encode ข้อมูลเหล่านั้นเป็น vertex attributes ได้:

```javascript
function mergeWithColors(geometries, colorsPerGeo) {
    // เพิ่ม color attribute ให้แต่ละ geometry ก่อน merge
    const coloredGeos = geometries.map((geo, geoIndex) => {
        const cloned = geo.clone();
        const count = cloned.attributes.position.count;
        const colors = new Float32Array(count * 3);
        const color = colorsPerGeo[geoIndex];
        
        // Fill color สำหรับทุก vertex ใน geometry นี้
        for (let i = 0; i < count; i++) {
            colors[i * 3] = color.r;
            colors[i * 3 + 1] = color.g;
            colors[i * 3 + 2] = color.b;
        }
        
        cloned.setAttribute('color', new THREE.BufferAttribute(colors, 3));
        return cloned;
    });
    
    const merged = mergeGeometries(coloredGeos);
    coloredGeos.forEach(g => g.dispose());
    
    return merged;
}

// ใช้งาน
const geos = [];
const colors = [];

for (let i = 0; i < 100; i++) {
    const geo = new THREE.BoxGeometry(1, 2, 1);
    geo.applyMatrix4(new THREE.Matrix4().setPosition(i * 2, 1, 0));
    geos.push(geo);
    
    const c = new THREE.Color();
    c.setHSL(i / 100, 0.7, 0.5);
    colors.push(c);
}

const mergedWithColors = mergeWithColors(geos, colors);

// Material ต้องใช้ vertexColors
const mat = new THREE.MeshStandardMaterial({ vertexColors: true });
const mesh = new THREE.Mesh(mergedWithColors, mat);
scene.add(mesh);
```

---

## 5. Handling Per-Instance Data: UV Offset

สำหรับ texture atlas - แต่ละ object ใช้ส่วนต่างกันของ texture:

```javascript
function mergeWithUVOffset(geometries, uvOffsets) {
    // uvOffsets: array of { u, v, uSize, vSize } (atlas coordinates)
    
    const offsettedGeos = geometries.map((geo, i) => {
        const cloned = geo.clone();
        const uv = cloned.attributes.uv;
        const offset = uvOffsets[i];
        
        for (let v = 0; v < uv.count; v++) {
            // Remap UV เข้า atlas tile
            const origU = uv.getX(v);
            const origV = uv.getY(v);
            uv.setX(v, origU * offset.uSize + offset.u);
            uv.setY(v, origV * offset.vSize + offset.v);
        }
        uv.needsUpdate = true;
        
        return cloned;
    });
    
    const merged = mergeGeometries(offsettedGeos);
    offsettedGeos.forEach(g => g.dispose());
    return merged;
}

// Atlas layout (2x2 grid)
const atlasUVs = [
    { u: 0.0, v: 0.0, uSize: 0.5, vSize: 0.5 },  // tile 0 (top-left)
    { u: 0.5, v: 0.0, uSize: 0.5, vSize: 0.5 },  // tile 1 (top-right)
    { u: 0.0, v: 0.5, uSize: 0.5, vSize: 0.5 },  // tile 2 (bottom-left)
    { u: 0.5, v: 0.5, uSize: 0.5, vSize: 0.5 }   // tile 3 (bottom-right)
];
```

---

## 6. Merging with Different Transforms (applyMatrix4)

ขั้นตอนสำคัญในการ merge คือการ apply transform ของแต่ละ mesh ลงใน geometry ก่อน:

```javascript
function mergeSceneObjects(objects) {
    const geometries = [];
    
    objects.forEach(obj => {
        // Update world matrix ก่อน
        obj.updateMatrixWorld(true);
        
        const geo = obj.geometry.clone();
        
        // Apply world matrix ลงใน geometry
        // ทำให้ vertex positions อยู่ใน world space
        geo.applyMatrix4(obj.matrixWorld);
        
        geometries.push(geo);
    });
    
    const merged = mergeGeometries(geometries);
    geometries.forEach(g => g.dispose());
    
    return merged;
}

// ตัวอย่าง: merge objects จาก scene
const staticObjects = [];
scene.traverse(obj => {
    if (obj.isMesh && obj.userData.isStatic) {
        staticObjects.push(obj);
    }
});

const mergedGeo = mergeSceneObjects(staticObjects);
const mergedMesh = new THREE.Mesh(
    mergedGeo,
    new THREE.MeshStandardMaterial({ color: 0x888888 })
);

// ลบ original objects
staticObjects.forEach(obj => scene.remove(obj));

// เพิ่ม merged mesh เดียว
scene.add(mergedMesh);
```

---

## 7. Memory Layout ของ Merged Geometry

```javascript
// หลังจาก merge, geometry จะมี memory layout แบบนี้:
// [geo1_v0, geo1_v1, ..., geo1_vN, geo2_v0, geo2_v1, ..., geo2_vM, ...]

// ตรวจสอบ memory layout
function analyzeGeometry(geo) {
    const report = {};
    
    for (const [name, attr] of Object.entries(geo.attributes)) {
        const byteLength = attr.array.byteLength;
        report[name] = {
            count: attr.count,
            itemSize: attr.itemSize,
            byteLength,
            type: attr.array.constructor.name
        };
    }
    
    if (geo.index) {
        report['index'] = {
            count: geo.index.count,
            byteLength: geo.index.array.byteLength,
            type: geo.index.array.constructor.name
        };
    }
    
    const totalBytes = Object.values(report).reduce((sum, a) => sum + a.byteLength, 0);
    report['_total_MB'] = (totalBytes / 1024 / 1024).toFixed(2);
    
    return report;
}

// ใช้งาน
const geos = [];
for (let i = 0; i < 1000; i++) {
    geos.push(new THREE.SphereGeometry(1, 8, 8));
}
const merged = mergeGeometries(geos);
console.log(analyzeGeometry(merged));
// { position: { count: N, ... }, normal: { ... }, _total_MB: '...' }
```

---

## 8. Updating Merged Geometry (DynamicDrawUsage + updateRange)

บางครั้งเราต้องการ update บางส่วนของ merged geometry โดยไม่ upload ทั้งหมด:

```javascript
class DynamicMergedGeometry {
    constructor(maxVertices) {
        // Pre-allocate buffers
        this.maxVertices = maxVertices;
        this.usedVertices = 0;
        
        this.positions = new Float32Array(maxVertices * 3);
        this.normals = new Float32Array(maxVertices * 3);
        this.uvs = new Float32Array(maxVertices * 2);
        
        // สร้าง geometry ด้วย DynamicDrawUsage
        this.geometry = new THREE.BufferGeometry();
        
        const posAttr = new THREE.BufferAttribute(this.positions, 3);
        posAttr.usage = THREE.DynamicDrawUsage;
        this.geometry.setAttribute('position', posAttr);
        
        const normalAttr = new THREE.BufferAttribute(this.normals, 3);
        normalAttr.usage = THREE.DynamicDrawUsage;
        this.geometry.setAttribute('normal', normalAttr);
        
        const uvAttr = new THREE.BufferAttribute(this.uvs, 2);
        uvAttr.usage = THREE.DynamicDrawUsage;
        this.geometry.setAttribute('uv', uvAttr);
        
        this.slotMap = new Map(); // slotId -> { start, count }
        this.nextSlotId = 0;
    }
    
    addGeometry(geo) {
        const count = geo.attributes.position.count;
        const start = this.usedVertices;
        
        if (start + count > this.maxVertices) {
            throw new Error('Exceeded max vertices!');
        }
        
        // Copy vertex data
        const srcPos = geo.attributes.position.array;
        this.positions.set(srcPos, start * 3);
        
        if (geo.attributes.normal) {
            const srcNorm = geo.attributes.normal.array;
            this.normals.set(srcNorm, start * 3);
        }
        
        if (geo.attributes.uv) {
            const srcUV = geo.attributes.uv.array;
            this.uvs.set(srcUV, start * 2);
        }
        
        this.usedVertices += count;
        
        // Mark update range
        const posAttr = this.geometry.attributes.position;
        posAttr.updateRange.offset = start * 3;
        posAttr.updateRange.count = count * 3;
        posAttr.needsUpdate = true;
        
        // Update draw range
        this.geometry.setDrawRange(0, this.usedVertices);
        
        const slotId = this.nextSlotId++;
        this.slotMap.set(slotId, { start, count });
        
        return slotId;
    }
    
    updateSlot(slotId, geo) {
        const slot = this.slotMap.get(slotId);
        if (!slot) throw new Error(`Invalid slot: ${slotId}`);
        
        const srcPos = geo.attributes.position.array;
        this.positions.set(srcPos, slot.start * 3);
        
        const posAttr = this.geometry.attributes.position;
        posAttr.updateRange.offset = slot.start * 3;
        posAttr.updateRange.count = slot.count * 3;
        posAttr.needsUpdate = true;
    }
}

// ใช้งาน
const dynamicGeo = new DynamicMergedGeometry(100000);
const mesh = new THREE.Mesh(
    dynamicGeo.geometry,
    new THREE.MeshStandardMaterial()
);
scene.add(mesh);

// เพิ่ม geometry ทีละชิ้น
const boxGeo = new THREE.BoxGeometry(1, 1, 1);
const slotId = dynamicGeo.addGeometry(boxGeo);

// Update ภายหลัง (เช่น เปลี่ยน position)
const updatedGeo = boxGeo.clone();
updatedGeo.applyMatrix4(new THREE.Matrix4().setPosition(5, 0, 0));
dynamicGeo.updateSlot(slotId, updatedGeo);
```

---

## 9. Interactive Merge Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Geometry Merging Demo</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #0a0a1a; color: #eee; font-family: 'Courier New', monospace; overflow: hidden; }
        #ui {
            position: fixed; top: 20px; left: 20px;
            background: rgba(0,0,20,0.9);
            padding: 20px; border-radius: 8px;
            border: 1px solid #0af;
            width: 340px; z-index: 10;
        }
        h2 { color: #0af; margin-bottom: 12px; font-size: 15px; }
        .section { margin: 12px 0; padding: 10px; background: rgba(255,255,255,0.05); border-radius: 4px; }
        .section h3 { color: #fa0; font-size: 12px; margin-bottom: 8px; }
        .row { display: flex; justify-content: space-between; margin: 4px 0; font-size: 12px; }
        .val-bad { color: #f44; font-weight: bold; }
        .val-good { color: #4f4; font-weight: bold; }
        .val-info { color: #fff; }
        .btn-row { display: flex; gap: 8px; margin: 10px 0; }
        button {
            flex: 1; padding: 8px; background: #1a1a3a;
            color: #eee; border: 1px solid #446;
            border-radius: 4px; cursor: pointer; font-size: 11px;
        }
        button.active { background: #0af; color: #000; border-color: #0af; }
        button:hover:not(.active) { border-color: #0af; }
        .progress-bar {
            height: 6px; background: #333; border-radius: 3px; margin-top: 4px;
        }
        .progress-fill { height: 100%; border-radius: 3px; transition: width 0.3s; }
    </style>
</head>
<body>
<canvas id="c"></canvas>
<div id="ui">
    <h2>📐 Geometry Merging Demo</h2>
    
    <div class="section">
        <h3>โหมดการแสดงผล</h3>
        <div class="btn-row">
            <button id="btn-sep" class="active">แยก Objects</button>
            <button id="btn-merged">Merged</button>
            <button id="btn-groups">Multi-Material</button>
        </div>
    </div>
    
    <div class="section">
        <h3>ประสิทธิภาพ</h3>
        <div class="row">
            <span>Draw Calls:</span>
            <span id="m-calls" class="val-bad">-</span>
        </div>
        <div class="progress-bar">
            <div id="calls-bar" class="progress-fill" style="background:#f44;width:100%"></div>
        </div>
        <div class="row"><span>Triangles:</span><span id="m-tris" class="val-info">-</span></div>
        <div class="row"><span>FPS:</span><span id="m-fps" class="val-info">-</span></div>
        <div class="row"><span>Geometry Objects:</span><span id="m-geos" class="val-info">-</span></div>
    </div>
    
    <div class="section">
        <h3>ข้อมูล Geometry</h3>
        <div class="row"><span>Object Count:</span><span id="m-objcount" class="val-info">-</span></div>
        <div class="row"><span>Merge Mode:</span><span id="m-mode" class="val-info">-</span></div>
        <div class="row"><span>Total Vertices:</span><span id="m-verts" class="val-info">-</span></div>
        <div class="row"><span>Memory (MB):</span><span id="m-mem" class="val-info">-</span></div>
    </div>
</div>

<script type="importmap">
{
    "imports": {
        "three": "https://cdn.jsdelivr.net/npm/three@0.165.0/build/three.module.js",
        "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.165.0/examples/jsm/"
    }
}
</script>
<script type="module">
    import * as THREE from 'three';
    import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
    import { mergeGeometries } from 'three/addons/utils/BufferGeometryUtils.js';
    
    const OBJ_COUNT = 800;
    
    const renderer = new THREE.WebGLRenderer({ canvas: document.getElementById('c'), antialias: true });
    renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
    
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x05050f);
    scene.fog = new THREE.FogExp2(0x05050f, 0.006);
    
    const camera = new THREE.PerspectiveCamera(60, 1, 0.1, 500);
    camera.position.set(0, 40, 100);
    
    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.autoRotate = true;
    controls.autoRotateSpeed = 0.3;
    
    scene.add(new THREE.AmbientLight(0xffffff, 0.4));
    const sun = new THREE.DirectionalLight(0xffeecc, 2);
    sun.position.set(100, 150, 50);
    scene.add(sun);
    
    // Ground plane
    const groundMesh = new THREE.Mesh(
        new THREE.PlaneGeometry(400, 400, 1, 1),
        new THREE.MeshStandardMaterial({ color: 0x0a1505 })
    );
    groundMesh.rotation.x = -Math.PI / 2;
    scene.add(groundMesh);
    
    // Pre-generate object data
    const objectData = Array.from({ length: OBJ_COUNT }, (_, i) => ({
        x: (Math.random() - 0.5) * 180,
        y: Math.random() * 0.5,
        z: (Math.random() - 0.5) * 180,
        w: 0.5 + Math.random() * 2,
        h: 2 + Math.random() * 8,
        d: 0.5 + Math.random() * 2,
        type: i % 3, // 0=building, 1=cylinder, 2=sphere
        color: new THREE.Color().setHSL(Math.random(), 0.6, 0.4)
    }));
    
    function clearDynamic() {
        const toRemove = [];
        scene.traverse(obj => {
            if (obj !== groundMesh && (obj.isMesh || obj.isInstancedMesh || obj.isGroup)) {
                toRemove.push(obj);
            }
        });
        toRemove.forEach(obj => {
            if (obj.geometry) obj.geometry.dispose();
            if (obj.material) {
                if (Array.isArray(obj.material)) obj.material.forEach(m => m.dispose());
                else obj.material.dispose();
            }
            scene.remove(obj);
        });
    }
    
    function makeGeo(d) {
        if (d.type === 0) return new THREE.BoxGeometry(d.w, d.h, d.d);
        if (d.type === 1) return new THREE.CylinderGeometry(d.w / 2, d.w / 2, d.h, 6);
        return new THREE.SphereGeometry(d.w / 2, 6, 4);
    }
    
    let currentMode = 'sep';
    let totalVerts = 0;
    let totalMem = 0;
    
    function buildSeparate() {
        clearDynamic();
        const mat = new THREE.MeshStandardMaterial({ vertexColors: true });
        totalVerts = 0;
        
        objectData.forEach(d => {
            const geo = makeGeo(d);
            totalVerts += geo.attributes.position.count;
            
            // Vertex colors
            const count = geo.attributes.position.count;
            const colors = new Float32Array(count * 3);
            for (let i = 0; i < count; i++) {
                colors[i * 3] = d.color.r;
                colors[i * 3 + 1] = d.color.g;
                colors[i * 3 + 2] = d.color.b;
            }
            geo.setAttribute('color', new THREE.BufferAttribute(colors, 3));
            
            const mesh = new THREE.Mesh(geo, mat);
            mesh.position.set(d.x, d.y + d.h / 2, d.z);
            scene.add(mesh);
        });
        
        totalMem = (totalVerts * (3 + 3 + 2) * 4 / 1024 / 1024);
        document.getElementById('m-mode').textContent = 'แยก (1 mesh = 1 draw call)';
        document.getElementById('m-objcount').textContent = OBJ_COUNT;
        document.getElementById('m-verts').textContent = totalVerts.toLocaleString();
        document.getElementById('m-mem').textContent = totalMem.toFixed(2);
    }
    
    function buildMerged() {
        clearDynamic();
        const geos = [];
        totalVerts = 0;
        
        objectData.forEach(d => {
            const geo = makeGeo(d);
            
            // Apply transform
            const mat4 = new THREE.Matrix4();
            mat4.setPosition(d.x, d.y + d.h / 2, d.z);
            geo.applyMatrix4(mat4);
            
            // Vertex colors
            const count = geo.attributes.position.count;
            totalVerts += count;
            const colors = new Float32Array(count * 3);
            for (let i = 0; i < count; i++) {
                colors[i * 3] = d.color.r;
                colors[i * 3 + 1] = d.color.g;
                colors[i * 3 + 2] = d.color.b;
            }
            geo.setAttribute('color', new THREE.BufferAttribute(colors, 3));
            geos.push(geo);
        });
        
        const merged = mergeGeometries(geos);
        geos.forEach(g => g.dispose());
        
        // Set StaticDrawUsage
        merged.attributes.position.usage = THREE.StaticDrawUsage;
        merged.attributes.normal.usage = THREE.StaticDrawUsage;
        merged.attributes.color.usage = THREE.StaticDrawUsage;
        
        const mat = new THREE.MeshStandardMaterial({ vertexColors: true });
        scene.add(new THREE.Mesh(merged, mat));
        
        totalMem = (totalVerts * (3 + 3 + 2) * 4 / 1024 / 1024);
        document.getElementById('m-mode').textContent = 'Merged (1 draw call)';
        document.getElementById('m-objcount').textContent = `${OBJ_COUNT} → 1 mesh`;
        document.getElementById('m-verts').textContent = totalVerts.toLocaleString();
        document.getElementById('m-mem').textContent = totalMem.toFixed(2);
    }
    
    function buildGroups() {
        clearDynamic();
        
        const types = [0, 1, 2];
        const typeGeos = types.map(() => []);
        const typeMaterials = [
            new THREE.MeshStandardMaterial({ color: 0x4488ff }),
            new THREE.MeshStandardMaterial({ color: 0xff8844 }),
            new THREE.MeshStandardMaterial({ color: 0x44ff88 })
        ];
        
        totalVerts = 0;
        
        objectData.forEach(d => {
            const geo = makeGeo(d);
            const mat4 = new THREE.Matrix4();
            mat4.setPosition(d.x, d.y + d.h / 2, d.z);
            geo.applyMatrix4(mat4);
            totalVerts += geo.attributes.position.count;
            typeGeos[d.type].push(geo);
        });
        
        // Merge each type separately
        const mergedByType = types.map(t => {
            if (typeGeos[t].length === 0) return null;
            const merged = mergeGeometries(typeGeos[t]);
            typeGeos[t].forEach(g => g.dispose());
            return merged;
        }).filter(Boolean);
        
        // Final merge with groups
        const finalGeo = mergeGeometries(mergedByType, true);
        mergedByType.forEach(g => g.dispose());
        
        const mesh = new THREE.Mesh(finalGeo, typeMaterials);
        scene.add(mesh);
        
        totalMem = (totalVerts * (3 + 3 + 2) * 4 / 1024 / 1024);
        document.getElementById('m-mode').textContent = `Multi-material (${types.length} draw calls)`;
        document.getElementById('m-objcount').textContent = `${OBJ_COUNT} → 1 mesh (${types.length} groups)`;
        document.getElementById('m-verts').textContent = totalVerts.toLocaleString();
        document.getElementById('m-mem').textContent = totalMem.toFixed(2);
    }
    
    buildSeparate();
    
    function setMode(mode) {
        currentMode = mode;
        document.querySelectorAll('button').forEach(b => b.classList.remove('active'));
        document.getElementById(`btn-${mode}`).classList.add('active');
        
        if (mode === 'sep') buildSeparate();
        else if (mode === 'merged') buildMerged();
        else if (mode === 'groups') buildGroups();
    }
    
    document.getElementById('btn-sep').addEventListener('click', () => setMode('sep'));
    document.getElementById('btn-merged').addEventListener('click', () => setMode('merged'));
    document.getElementById('btn-groups').addEventListener('click', () => setMode('groups'));
    
    let lastTime = performance.now(), frames = 0, fps = 0;
    
    function resize() {
        renderer.setSize(innerWidth, innerHeight);
        camera.aspect = innerWidth / innerHeight;
        camera.updateProjectionMatrix();
    }
    window.addEventListener('resize', resize);
    resize();
    
    function animate() {
        requestAnimationFrame(animate);
        controls.update();
        renderer.render(scene, camera);
        
        frames++;
        const now = performance.now();
        if (now - lastTime >= 500) {
            fps = Math.round(frames / ((now - lastTime) / 1000));
            frames = 0;
            lastTime = now;
        }
        
        const calls = renderer.info.render.calls;
        const maxCalls = OBJ_COUNT;
        const callsPercent = Math.min((calls / maxCalls) * 100, 100);
        
        document.getElementById('m-calls').textContent = calls;
        document.getElementById('m-calls').className = calls > 100 ? 'val-bad' : 'val-good';
        document.getElementById('calls-bar').style.width = callsPercent + '%';
        document.getElementById('calls-bar').style.background = calls > 100 ? '#f44' : '#4f4';
        document.getElementById('m-tris').textContent = renderer.info.render.triangles.toLocaleString();
        document.getElementById('m-fps').textContent = fps;
        document.getElementById('m-geos').textContent = renderer.info.memory.geometries;
    }
    
    animate();
</script>
</body>
</html>
```

---

## 10. City Block Builder Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>City Block Builder - Merged</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; }
        canvas { display: block; }
        #hud {
            position: fixed; bottom: 20px; left: 50%; transform: translateX(-50%);
            background: rgba(0,0,0,0.8); padding: 15px 30px;
            border-radius: 30px; border: 1px solid #0af;
            display: flex; gap: 30px; color: #eee; font-family: monospace; font-size: 13px;
        }
        .stat { text-align: center; }
        .stat-val { font-size: 20px; font-weight: bold; color: #0af; }
    </style>
</head>
<body>
<canvas id="c"></canvas>
<div id="hud">
    <div class="stat"><div class="stat-val" id="h-calls">-</div><div>Draw Calls</div></div>
    <div class="stat"><div class="stat-val" id="h-fps">-</div><div>FPS</div></div>
    <div class="stat"><div class="stat-val" id="h-tris">-</div><div>Triangles</div></div>
    <div class="stat"><div class="stat-val" id="h-blocks">-</div><div>City Blocks</div></div>
</div>

<script type="importmap">
{
    "imports": {
        "three": "https://cdn.jsdelivr.net/npm/three@0.165.0/build/three.module.js",
        "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.165.0/examples/jsm/"
    }
}
</script>
<script type="module">
    import * as THREE from 'three';
    import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
    import { mergeGeometries } from 'three/addons/utils/BufferGeometryUtils.js';
    
    const renderer = new THREE.WebGLRenderer({ canvas: document.getElementById('c'), antialias: true });
    renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x0a1528);
    scene.fog = new THREE.FogExp2(0x0a1528, 0.004);
    
    const camera = new THREE.PerspectiveCamera(50, 1, 0.5, 1000);
    camera.position.set(0, 80, 150);
    
    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.autoRotate = true;
    controls.autoRotateSpeed = 0.2;
    controls.maxPolarAngle = Math.PI / 2.1;
    
    const ambientLight = new THREE.AmbientLight(0x223355, 0.8);
    scene.add(ambientLight);
    
    const sunLight = new THREE.DirectionalLight(0xfff0cc, 2);
    sunLight.position.set(100, 200, 100);
    sunLight.castShadow = true;
    sunLight.shadow.mapSize.set(2048, 2048);
    sunLight.shadow.camera.near = 0.5;
    sunLight.shadow.camera.far = 500;
    sunLight.shadow.camera.left = -200;
    sunLight.shadow.camera.right = 200;
    sunLight.shadow.camera.top = 200;
    sunLight.shadow.camera.bottom = -200;
    scene.add(sunLight);
    
    // Accent light
    const blueLight = new THREE.PointLight(0x4488ff, 2, 100);
    blueLight.position.set(-50, 20, -50);
    scene.add(blueLight);
    
    // Build city using merged geometry
    const GRID_SIZE = 10; // 10x10 = 100 city blocks
    const BLOCK_SIZE = 20;
    const STREET_WIDTH = 5;
    
    let totalBlocks = 0;
    
    // Separate arrays for different material types
    const buildingGeos = [];
    const streetGeos = [];
    const sidewalkGeos = [];
    
    for (let gx = -GRID_SIZE / 2; gx < GRID_SIZE / 2; gx++) {
        for (let gz = -GRID_SIZE / 2; gz < GRID_SIZE / 2; gz++) {
            const blockX = gx * (BLOCK_SIZE + STREET_WIDTH);
            const blockZ = gz * (BLOCK_SIZE + STREET_WIDTH);
            totalBlocks++;
            
            // Buildings in block
            const buildingsPerBlock = 2 + Math.floor(Math.random() * 4);
            for (let b = 0; b < buildingsPerBlock; b++) {
                const bw = 3 + Math.random() * 6;
                const bh = 5 + Math.random() * 25;
                const bd = 3 + Math.random() * 6;
                const bx = blockX + (Math.random() - 0.5) * (BLOCK_SIZE - bw);
                const bz = blockZ + (Math.random() - 0.5) * (BLOCK_SIZE - bd);
                
                const geo = new THREE.BoxGeometry(bw, bh, bd);
                geo.applyMatrix4(new THREE.Matrix4().setPosition(bx, bh / 2, bz));
                buildingGeos.push(geo);
            }
            
            // Sidewalk
            const swGeo = new THREE.BoxGeometry(BLOCK_SIZE, 0.2, BLOCK_SIZE);
            swGeo.applyMatrix4(new THREE.Matrix4().setPosition(blockX, 0.1, blockZ));
            sidewalkGeos.push(swGeo);
        }
    }
    
    // Streets (grid)
    for (let i = -GRID_SIZE / 2; i <= GRID_SIZE / 2; i++) {
        const pos = i * (BLOCK_SIZE + STREET_WIDTH);
        const length = GRID_SIZE * (BLOCK_SIZE + STREET_WIDTH) + STREET_WIDTH;
        
        // Horizontal street
        const hStreet = new THREE.PlaneGeometry(length, STREET_WIDTH);
        hStreet.rotateX(-Math.PI / 2);
        hStreet.applyMatrix4(new THREE.Matrix4().setPosition(0, 0.05, pos));
        streetGeos.push(hStreet);
        
        // Vertical street
        const vStreet = new THREE.PlaneGeometry(STREET_WIDTH, length);
        vStreet.rotateX(-Math.PI / 2);
        vStreet.applyMatrix4(new THREE.Matrix4().setPosition(pos, 0.05, 0));
        streetGeos.push(vStreet);
    }
    
    // Merge and add to scene
    const buildingMat = new THREE.MeshStandardMaterial({ color: 0x3355aa, roughness: 0.8, metalness: 0.2 });
    const streetMat = new THREE.MeshStandardMaterial({ color: 0x1a1a2a, roughness: 1.0 });
    const sidewalkMat = new THREE.MeshStandardMaterial({ color: 0x888899, roughness: 0.9 });
    
    if (buildingGeos.length) {
        const merged = mergeGeometries(buildingGeos);
        buildingGeos.forEach(g => g.dispose());
        const mesh = new THREE.Mesh(merged, buildingMat);
        mesh.castShadow = true;
        mesh.receiveShadow = true;
        scene.add(mesh);
    }
    
    if (streetGeos.length) {
        const merged = mergeGeometries(streetGeos);
        streetGeos.forEach(g => g.dispose());
        const mesh = new THREE.Mesh(merged, streetMat);
        mesh.receiveShadow = true;
        scene.add(mesh);
    }
    
    if (sidewalkGeos.length) {
        const merged = mergeGeometries(sidewalkGeos);
        sidewalkGeos.forEach(g => g.dispose());
        const mesh = new THREE.Mesh(merged, sidewalkMat);
        mesh.receiveShadow = true;
        scene.add(mesh);
    }
    
    // Ground
    const ground = new THREE.Mesh(
        new THREE.PlaneGeometry(1000, 1000),
        new THREE.MeshStandardMaterial({ color: 0x0a0a0f })
    );
    ground.rotation.x = -Math.PI / 2;
    ground.receiveShadow = true;
    scene.add(ground);
    
    let lastTime = performance.now(), frames = 0, fps = 0;
    
    function resize() {
        renderer.setSize(innerWidth, innerHeight);
        camera.aspect = innerWidth / innerHeight;
        camera.updateProjectionMatrix();
    }
    window.addEventListener('resize', resize);
    resize();
    
    function animate() {
        requestAnimationFrame(animate);
        controls.update();
        renderer.render(scene, camera);
        
        frames++;
        const now = performance.now();
        if (now - lastTime >= 500) {
            fps = Math.round(frames / ((now - lastTime) / 1000));
            frames = 0;
            lastTime = now;
        }
        
        document.getElementById('h-calls').textContent = renderer.info.render.calls;
        document.getElementById('h-fps').textContent = fps;
        document.getElementById('h-tris').textContent = 
            (renderer.info.render.triangles / 1000).toFixed(0) + 'K';
        document.getElementById('h-blocks').textContent = totalBlocks;
    }
    
    animate();
</script>
</body>
</html>
```

---

## 11. เมื่อ NOT ควร Merge: กรณีที่ควรหลีกเลี่ยง

```javascript
// ❌ อย่า merge objects ที่มี animation
const walkingCharacter = new THREE.Mesh(skinnedGeo, mat);
// SkinnedMesh ต้องการ bone transforms แยกกัน

// ❌ อย่า merge objects ที่ถูก add/remove บ่อยๆ
function spawnBullet() {
    const geo = new THREE.SphereGeometry(0.1);
    const mesh = new THREE.Mesh(geo, bulletMat);
    scene.add(mesh);
    
    // ถ้า merge ทุกครั้งที่ spawn bullet จะช้ากว่า!
    // ให้ใช้ InstancedMesh หรือ Object Pool แทน
}

// ❌ อย่า merge objects ที่ต้องการ individual raycasting
const interactiveObjects = buildings.map(b => new THREE.Mesh(b, mat));
// ถ้า merge แล้ว raycast จะไม่รู้ว่า hit building ไหน

// ✅ ใช้ Merge สำหรับ:
// - Background scenery
// - Static terrain
// - Distant objects (low-poly)
// - Objects ที่ไม่มีการ interact

// ✅ ใช้ InstancedMesh สำหรับ:
// - Objects เดียวกันหลายชิ้น (trees, grass)
// - Objects ที่ต้องการ animate transforms แยกกัน
// - Objects ที่ถูก add/remove บ่อย
```

---

## แบบฝึกหัด

1. **ง่าย**: สร้าง terrain จาก PlaneGeometry 100 ชิ้น merge เป็น 1 ชิ้น และเปรียบเทียบ draw calls
2. **กลาง**: สร้าง city block ที่ใช้ multi-material merged geometry (buildings=1 material, roads=1 material) ทำให้ใช้แค่ 2 draw calls สำหรับ 200+ objects
3. **ยาก**: สร้าง `DynamicMergedGeometry` class ที่รองรับการ add/update/remove objects แบบ efficient โดยใช้ `updateRange`
4. **Challenge**: Implement Frustum Culling สำหรับ merged geometry โดยแบ่ง merged geometry เป็น chunks ตาม spatial position

---

## อ้างอิง

- [Three.js BufferGeometryUtils](https://threejs.org/docs/#examples/en/utils/BufferGeometryUtils)
- [WebGL Buffer Usage Hints](https://registry.khronos.org/OpenGL-Refpages/es3.0/html/glBufferData.xhtml)
- [Static Draw vs Dynamic Draw](https://developer.mozilla.org/en-US/docs/Web/API/WebGLRenderingContext/bufferData)
