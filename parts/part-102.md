# Part 102: Draw Call Optimization (Advanced)

## บทนำ: ทำไม Draw Call ถึงสำคัญ

ในการพัฒนา 3D Web Graphics ด้วย Three.js สิ่งที่มักเป็นคอขวดของประสิทธิภาพที่สำคัญที่สุดอย่างหนึ่งคือ **Draw Call** หรือการเรียกคำสั่งวาดกราฟิกไปยัง GPU

Draw Call คือการที่ CPU ส่งคำสั่งไปให้ GPU วาด object หนึ่งชิ้น ปัญหาคือการส่งคำสั่งนี้มีค่าใช้จ่าย (overhead) สูงมากในแง่ของการสื่อสารระหว่าง CPU กับ GPU ยิ่งมี Draw Call มาก CPU ก็ยิ่งต้องทำงานหนักในการส่งคำสั่ง แม้ว่า GPU จะยังว่างอยู่ก็ตาม

ในบทนี้เราจะเรียนรู้:
- Draw Call คืออะไรในระดับ WebGL
- วิธีวัดและ monitor Draw Call
- เทคนิคลด Draw Call 5 วิธีหลัก
- การวัดผลก่อนและหลังการ optimize

---

## 1. Draw Call คืออะไรในระดับ WebGL

เมื่อ Three.js render scene หนึ่งครั้ง มันจะเรียก WebGL API ที่ชื่อว่า `gl.drawElements()` หรือ `gl.drawArrays()` สำหรับแต่ละ Mesh ที่ต้องวาด

```javascript
// สิ่งที่เกิดขึ้นภายใน Three.js renderer (simplified)
// สำหรับแต่ละ mesh ใน scene:
gl.bindVertexArray(mesh.vao);           // ผูก vertex data
gl.useProgram(material.shader);          // เปลี่ยน shader
gl.bindTexture(GL_TEXTURE_2D, texture);  // ผูก texture
gl.uniformMatrix4fv(mvpLocation, matrix); // ส่ง transform matrix
gl.drawElements(GL_TRIANGLES, count, GL_UNSIGNED_INT, 0); // ** DRAW CALL **
```

**ค่าใช้จ่ายของ Draw Call ประกอบด้วย:**
1. **State validation** - GPU driver ต้องตรวจสอบ state ทั้งหมดก่อนวาด
2. **Command submission** - ส่งคำสั่งจาก CPU memory ไป GPU command buffer
3. **Pipeline flush** - GPU ต้อง flush pipeline ระหว่าง state changes
4. **Driver overhead** - JavaScript → WebGL → OpenGL ES driver → GPU

**กฎทั่วไป:**
- Desktop GPU ทนได้ประมาณ 2,000-10,000 draw calls ต่อ frame (60fps)
- Mobile GPU ทนได้น้อยกว่า ประมาณ 200-1,000 draw calls
- แต่ละ draw call เสียเวลา CPU ประมาณ 0.1-1 microsecond

---

## 2. วิธีวัด Draw Call ด้วย renderer.info

Three.js มี built-in performance counters ใน `renderer.info`:

```javascript
// ดู draw call statistics หลังจาก render แต่ละ frame
function animate() {
    requestAnimationFrame(animate);
    renderer.render(scene, camera);
    
    // renderer.info.render จะ reset ทุก frame
    const info = renderer.info.render;
    console.log({
        calls: info.calls,          // จำนวน draw calls
        triangles: info.triangles,  // จำนวน triangles ที่วาด
        points: info.points,        // จำนวน points
        lines: info.lines           // จำนวน lines
    });
    
    // renderer.info.memory แสดงทรัพยากรที่ใช้
    const mem = renderer.info.memory;
    console.log({
        geometries: mem.geometries, // จำนวน geometry objects ใน GPU
        textures: mem.textures      // จำนวน textures ใน GPU
    });
}
```

**หมายเหตุสำคัญ:** `renderer.info` จะ reset ทุก frame เมื่อ `renderer.render()` ถูกเรียก ดังนั้นให้อ่านค่าหลังจาก render เสมอ

---

## 3. กลยุทธ์ที่ 1: InstancedMesh สำหรับ Object ซ้ำๆ

`InstancedMesh` คือวิธีที่มีประสิทธิภาพที่สุดในการวาด objects ที่มี geometry และ material เดียวกัน แต่ต่างกันใน position, rotation, scale

```javascript
// แบบเดิม: 1000 mesh = 1000 draw calls
const boxes = [];
for (let i = 0; i < 1000; i++) {
    const mesh = new THREE.Mesh(geometry, material);
    mesh.position.set(Math.random() * 100, Math.random() * 100, Math.random() * 100);
    scene.add(mesh);
    boxes.push(mesh);
}
// ผลลัพธ์: ~1000 draw calls

// แบบใหม่: 1000 instances = 1 draw call
const instancedMesh = new THREE.InstancedMesh(geometry, material, 1000);

const matrix = new THREE.Matrix4();
const position = new THREE.Vector3();
const quaternion = new THREE.Quaternion();
const scale = new THREE.Vector3(1, 1, 1);

for (let i = 0; i < 1000; i++) {
    position.set(
        Math.random() * 100 - 50,
        Math.random() * 100 - 50,
        Math.random() * 100 - 50
    );
    matrix.compose(position, quaternion, scale);
    instancedMesh.setMatrixAt(i, matrix);
}

instancedMesh.instanceMatrix.needsUpdate = true;
scene.add(instancedMesh);
// ผลลัพธ์: 1 draw call สำหรับ 1000 instances!
```

### การใช้ Instance Colors

```javascript
// เพิ่ม per-instance color
instancedMesh.instanceColor = new THREE.InstancedBufferAttribute(
    new Float32Array(1000 * 3), 3
);

const color = new THREE.Color();
for (let i = 0; i < 1000; i++) {
    color.setHSL(Math.random(), 0.7, 0.5);
    instancedMesh.setColorAt(i, color);
}
instancedMesh.instanceColor.needsUpdate = true;
```

---

## 4. กลยุทธ์ที่ 2: mergeGeometries สำหรับ Static Scene

สำหรับ objects ที่ไม่เคลื่อนไหว การ merge geometry เป็น geometry เดียวเป็นวิธีที่มีประสิทธิภาพมาก

```javascript
import * as THREE from 'three';
import { mergeGeometries } from 'three/addons/utils/BufferGeometryUtils.js';

// สร้าง geometries หลายชิ้นที่ต้องการ merge
const geometries = [];

for (let i = 0; i < 500; i++) {
    const geo = new THREE.BoxGeometry(1, 1, 1);
    
    // ต้อง apply transform ก่อน merge (ไม่งั้นทุกชิ้นจะอยู่ที่ origin)
    const matrix = new THREE.Matrix4();
    matrix.setPosition(
        Math.random() * 100 - 50,
        Math.random() * 10,
        Math.random() * 100 - 50
    );
    geo.applyMatrix4(matrix);
    geometries.push(geo);
}

// Merge ทุก geometry เป็น geometry เดียว
const mergedGeometry = mergeGeometries(geometries);

// ใช้ material เดียว
const material = new THREE.MeshStandardMaterial({ color: 0x888888 });
const mergedMesh = new THREE.Mesh(mergedGeometry, material);

// Dispose geometries ชิ้นเล็กๆ ที่ไม่ใช้แล้ว
geometries.forEach(geo => geo.dispose());

scene.add(mergedMesh);
// ผลลัพธ์: 1 draw call แทนที่จะเป็น 500!
```

### การ Merge ด้วย Multiple Materials

```javascript
// ถ้าต้องการใช้หลาย material กับ merged geometry
const geometries = [];
const materials = [];

// group 1: buildings (red)
for (let i = 0; i < 50; i++) {
    const geo = new THREE.BoxGeometry(2, 4, 2);
    geo.applyMatrix4(new THREE.Matrix4().setPosition(
        i * 5, 2, 0
    ));
    geometries.push(geo);
}
materials.push(new THREE.MeshStandardMaterial({ color: 0xff4444 }));

// group 2: roads (gray)
for (let i = 0; i < 50; i++) {
    const geo = new THREE.PlaneGeometry(4, 100);
    geo.rotateX(-Math.PI / 2);
    geo.applyMatrix4(new THREE.Matrix4().setPosition(i * 5 + 2, 0, 0));
    geometries.push(geo);
}
materials.push(new THREE.MeshStandardMaterial({ color: 0x555555 }));

// mergeGeometries ด้วย useGroups = true จะสร้าง groups สำหรับแต่ละ material
const merged = mergeGeometries(geometries, false); // false = don't use groups
// ถ้าต้องการ groups:
// const merged = mergeGeometries(geometries, true);

const mesh = new THREE.Mesh(merged, materials);
scene.add(mesh);
// ผลลัพธ์: 2 draw calls แทน 100!
```

---

## 5. กลยุทธ์ที่ 3: Texture Atlas เพื่อหลีกเลี่ยง Material Switches

เมื่อ Three.js เปลี่ยน material ระหว่างการวาด มันต้องทำ state change ซึ่งมีค่าใช้จ่ายสูง Texture Atlas คือการรวม textures หลายอันไว้ในรูปเดียวกัน

```javascript
// สร้าง Texture Atlas ด้วย Canvas API
function createTextureAtlas(textureURLs, atlasSize = 2048) {
    const canvas = document.createElement('canvas');
    canvas.width = atlasSize;
    canvas.height = atlasSize;
    const ctx = canvas.getContext('2d');
    
    const tileSize = atlasSize / Math.ceil(Math.sqrt(textureURLs.length));
    const tilesPerRow = Math.floor(atlasSize / tileSize);
    
    const uvMap = {};
    
    const loadPromises = textureURLs.map((url, index) => {
        return new Promise((resolve) => {
            const img = new Image();
            img.onload = () => {
                const col = index % tilesPerRow;
                const row = Math.floor(index / tilesPerRow);
                const x = col * tileSize;
                const y = row * tileSize;
                
                ctx.drawImage(img, x, y, tileSize, tileSize);
                
                // เก็บ UV coordinates ของ tile นี้
                uvMap[url] = {
                    u: x / atlasSize,
                    v: y / atlasSize,
                    uSize: tileSize / atlasSize,
                    vSize: tileSize / atlasSize
                };
                resolve();
            };
            img.src = url;
        });
    });
    
    return Promise.all(loadPromises).then(() => {
        const texture = new THREE.CanvasTexture(canvas);
        return { texture, uvMap };
    });
}

// ใช้งาน Atlas
async function buildScene() {
    const { texture, uvMap } = await createTextureAtlas([
        'building1.jpg', 'building2.jpg', 'road.jpg', 'grass.jpg'
    ]);
    
    const material = new THREE.MeshStandardMaterial({ map: texture });
    
    // Remap UV coordinates ของ mesh แต่ละชิ้น
    function remapUV(geometry, textureURL) {
        const uv = uvMap[textureURL];
        const uvAttribute = geometry.attributes.uv;
        
        for (let i = 0; i < uvAttribute.count; i++) {
            uvAttribute.setX(i, uvAttribute.getX(i) * uv.uSize + uv.u);
            uvAttribute.setY(i, uvAttribute.getY(i) * uv.vSize + uv.v);
        }
        uvAttribute.needsUpdate = true;
    }
    
    // สร้าง meshes ด้วย material เดียวกัน (atlas)
    const buildingGeo = new THREE.BoxGeometry(2, 4, 2);
    remapUV(buildingGeo, 'building1.jpg');
    
    const roadGeo = new THREE.PlaneGeometry(10, 100);
    remapUV(roadGeo, 'road.jpg');
    
    // ทั้งสองใช้ material เดียวกัน = ไม่มี material switch!
    scene.add(new THREE.Mesh(buildingGeo, material));
    scene.add(new THREE.Mesh(roadGeo, material));
}
```

---

## 6. กลยุทธ์ที่ 4: Batching by Material

Three.js วาด objects ที่ใช้ material เดียวกันต่อเนื่องกัน การจัด batch ตาม material ช่วยลด state changes:

```javascript
// เรียงลำดับ children ตาม material UUID เพื่อลด state changes
function sortByMaterial(scene) {
    const groups = new Map();
    
    scene.traverse((obj) => {
        if (obj.isMesh) {
            const matId = obj.material.uuid;
            if (!groups.has(matId)) groups.set(matId, []);
            groups.get(matId).push(obj);
        }
    });
    
    // ลบทุก mesh และ add กลับในลำดับที่ sort แล้ว
    const meshes = [];
    groups.forEach((group) => meshes.push(...group));
    
    // จัด render order
    meshes.forEach((mesh, i) => {
        mesh.renderOrder = i;
    });
}

// หรือใช้ Custom Sort Function
renderer.setRenderListSorting((a, b) => {
    // sort by material first
    if (a.material.id !== b.material.id) {
        return a.material.id - b.material.id;
    }
    // then by distance (front to back for opaque)
    return a.z - b.z;
});
```

---

## 7. กลยุทธ์ที่ 5: LOD (Level of Detail)

LOD ลดจำนวน vertices ที่ต้องประมวลผลตามระยะห่างจาก camera:

```javascript
function createLODTree(position) {
    const lod = new THREE.LOD();
    
    // High detail (ใกล้ camera)
    const highGeo = new THREE.CylinderGeometry(1, 2, 5, 16, 4);
    const highMesh = new THREE.Mesh(highGeo, material);
    lod.addLevel(highMesh, 0); // แสดงเมื่อระยะ < x (x จะคำนวณจาก level ถัดไป)
    
    // Medium detail
    const medGeo = new THREE.CylinderGeometry(1, 2, 5, 8, 2);
    const medMesh = new THREE.Mesh(medGeo, material);
    lod.addLevel(medMesh, 30); // แสดงเมื่อระยะ >= 30
    
    // Low detail (ไกล camera)
    const lowGeo = new THREE.CylinderGeometry(1, 2, 5, 4, 1);
    const lowMesh = new THREE.Mesh(lowGeo, material);
    lod.addLevel(lowMesh, 80); // แสดงเมื่อระยะ >= 80
    
    // Invisible (ไกลมาก)
    lod.addLevel(new THREE.Object3D(), 200); // ไม่แสดง
    
    lod.position.copy(position);
    return lod;
}

// อย่าลืม update LOD ทุก frame
function animate() {
    requestAnimationFrame(animate);
    scene.traverse((obj) => {
        if (obj.isLOD) obj.update(camera);
    });
    renderer.render(scene, camera);
}
```

---

## 8. Transparent Object Sorting

Objects ที่มี transparency ต้องวาด back-to-front (painter's algorithm) เพื่อให้ blending ถูกต้อง:

```javascript
// Three.js sort transparent objects โดยอัตโนมัติ
// แต่เราสามารถ optimize เพิ่มเติมได้

// กำหนด render order ด้วยตนเอง
solidMesh.renderOrder = 0;
transparentMesh.renderOrder = 1;

// ปิด depthWrite สำหรับ transparent objects เพื่อประสิทธิภาพ
transparentMaterial.transparent = true;
transparentMaterial.depthWrite = false; // ไม่เขียน depth buffer

// แยก opaque และ transparent ให้ชัดเจน
// Three.js จัดการ render list แยกกัน
opaqueMaterial.transparent = false;   // render pass แรก (front-to-back)
transparentMaterial.transparent = true; // render pass ที่สอง (back-to-front)
```

---

## 9. การวัดผล Before/After

```javascript
class DrawCallBenchmark {
    constructor(renderer) {
        this.renderer = renderer;
        this.samples = [];
        this.maxSamples = 60;
    }
    
    measure() {
        const info = this.renderer.info.render;
        const sample = {
            calls: info.calls,
            triangles: info.triangles,
            timestamp: performance.now()
        };
        
        this.samples.push(sample);
        if (this.samples.length > this.maxSamples) {
            this.samples.shift();
        }
        
        return sample;
    }
    
    getAverage() {
        if (this.samples.length === 0) return null;
        
        const avg = this.samples.reduce((acc, s) => ({
            calls: acc.calls + s.calls,
            triangles: acc.triangles + s.triangles
        }), { calls: 0, triangles: 0 });
        
        return {
            calls: avg.calls / this.samples.length,
            triangles: avg.triangles / this.samples.length
        };
    }
}
```

---

## 10. Interactive Demo: Draw Call Benchmark

ตัวอย่างด้านล่างนี้เป็น interactive demo ที่แสดงให้เห็นความแตกต่างของ draw calls ระหว่างการใช้เทคนิคต่างๆ:

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Draw Call Optimization Benchmark</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #0a0a0a; color: #fff; font-family: monospace; overflow: hidden; }
        #canvas-container { position: absolute; top: 0; left: 0; width: 100%; height: 100%; }
        #ui {
            position: absolute;
            top: 20px;
            left: 20px;
            background: rgba(0,0,0,0.8);
            padding: 20px;
            border-radius: 8px;
            border: 1px solid #333;
            min-width: 320px;
            z-index: 10;
        }
        h2 { color: #0af; margin-bottom: 15px; font-size: 14px; }
        .metric { margin: 8px 0; display: flex; justify-content: space-between; }
        .metric-label { color: #888; }
        .metric-value { color: #0f0; font-weight: bold; }
        .btn-group { display: flex; flex-wrap: wrap; gap: 8px; margin: 15px 0; }
        button {
            padding: 8px 14px;
            background: #1a1a2e;
            color: #fff;
            border: 1px solid #444;
            border-radius: 4px;
            cursor: pointer;
            font-size: 12px;
            transition: all 0.2s;
        }
        button:hover { background: #0af; color: #000; }
        button.active { background: #0af; color: #000; border-color: #0af; }
        .improvement { color: #0f0; font-size: 12px; margin-top: 10px; padding: 8px; background: rgba(0,255,0,0.1); border-radius: 4px; }
        .slider-group { margin: 10px 0; }
        .slider-group label { display: block; margin-bottom: 5px; color: #aaa; font-size: 12px; }
        input[type=range] { width: 100%; accent-color: #0af; }
    </style>
</head>
<body>
    <div id="canvas-container">
        <canvas id="c"></canvas>
    </div>
    <div id="ui">
        <h2>🎯 Draw Call Benchmark</h2>
        
        <div class="slider-group">
            <label>จำนวน Objects: <span id="count-label">500</span></label>
            <input type="range" id="count-slider" min="100" max="5000" step="100" value="500">
        </div>
        
        <div class="btn-group">
            <button id="btn-naive" class="active">Naive (Individual Mesh)</button>
            <button id="btn-instanced">InstancedMesh</button>
            <button id="btn-merged">Merged Geometry</button>
            <button id="btn-lod">LOD Demo</button>
        </div>
        
        <div class="metric">
            <span class="metric-label">Draw Calls:</span>
            <span class="metric-value" id="m-calls">-</span>
        </div>
        <div class="metric">
            <span class="metric-label">Triangles:</span>
            <span class="metric-value" id="m-triangles">-</span>
        </div>
        <div class="metric">
            <span class="metric-label">FPS:</span>
            <span class="metric-value" id="m-fps">-</span>
        </div>
        <div class="metric">
            <span class="metric-label">Frame Time (ms):</span>
            <span class="metric-value" id="m-frametime">-</span>
        </div>
        <div class="metric">
            <span class="metric-label">GPU Memory (Geo):</span>
            <span class="metric-value" id="m-memory">-</span>
        </div>
        
        <div id="improvement-info" class="improvement" style="display:none"></div>
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

        // Setup
        const canvas = document.getElementById('c');
        const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
        renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
        renderer.shadowMap.enabled = true;
        
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x0a0a1a);
        scene.fog = new THREE.Fog(0x0a0a1a, 50, 200);
        
        const camera = new THREE.PerspectiveCamera(60, 1, 0.1, 1000);
        camera.position.set(0, 30, 80);
        
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.dampingFactor = 0.05;
        
        // Lighting
        const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
        scene.add(ambientLight);
        
        const directLight = new THREE.DirectionalLight(0xffffff, 1.5);
        directLight.position.set(50, 100, 50);
        directLight.castShadow = true;
        scene.add(directLight);
        
        // Ground
        const groundGeo = new THREE.PlaneGeometry(300, 300);
        const groundMat = new THREE.MeshStandardMaterial({ color: 0x1a2a1a });
        const ground = new THREE.Mesh(groundGeo, groundMat);
        ground.rotation.x = -Math.PI / 2;
        ground.receiveShadow = true;
        scene.add(ground);
        
        // State
        let currentMode = 'naive';
        let objectCount = 500;
        let naiveBaseCalls = 0;
        
        // Colors for variety
        const colors = [0xff4444, 0x44ff44, 0x4444ff, 0xffff44, 0xff44ff, 0x44ffff];
        
        // Clear scene objects (keep ground and lights)
        function clearObjects() {
            const toRemove = [];
            scene.traverse(obj => {
                if (obj.isMesh && obj !== ground) toRemove.push(obj);
                if (obj.isInstancedMesh) toRemove.push(obj);
                if (obj.isLOD) toRemove.push(obj);
                if (obj.isGroup) toRemove.push(obj);
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
        
        // Mode 1: Naive - individual meshes
        function buildNaive(count) {
            clearObjects();
            
            const mat = new THREE.MeshStandardMaterial({ color: 0x888888 });
            
            for (let i = 0; i < count; i++) {
                const size = 0.5 + Math.random() * 2;
                const geo = new THREE.BoxGeometry(size, size * 2, size);
                const mesh = new THREE.Mesh(geo, mat);
                mesh.position.set(
                    (Math.random() - 0.5) * 150,
                    size,
                    (Math.random() - 0.5) * 150
                );
                mesh.rotation.y = Math.random() * Math.PI * 2;
                mesh.castShadow = true;
                scene.add(mesh);
            }
            
            document.getElementById('improvement-info').style.display = 'none';
        }
        
        // Mode 2: InstancedMesh
        function buildInstanced(count) {
            clearObjects();
            
            const geo = new THREE.BoxGeometry(1, 2, 1);
            const mat = new THREE.MeshStandardMaterial();
            const mesh = new THREE.InstancedMesh(geo, mat, count);
            mesh.castShadow = true;
            
            const matrix = new THREE.Matrix4();
            const position = new THREE.Vector3();
            const quaternion = new THREE.Quaternion();
            const scale = new THREE.Vector3();
            const color = new THREE.Color();
            
            for (let i = 0; i < count; i++) {
                const s = 0.5 + Math.random() * 1.5;
                position.set(
                    (Math.random() - 0.5) * 150,
                    s,
                    (Math.random() - 0.5) * 150
                );
                quaternion.setFromEuler(new THREE.Euler(0, Math.random() * Math.PI * 2, 0));
                scale.set(s, s, s);
                matrix.compose(position, quaternion, scale);
                mesh.setMatrixAt(i, matrix);
                
                color.setHSL(Math.random(), 0.7, 0.5);
                mesh.setColorAt(i, color);
            }
            
            mesh.instanceMatrix.needsUpdate = true;
            if (mesh.instanceColor) mesh.instanceColor.needsUpdate = true;
            scene.add(mesh);
            
            // Show improvement
            const reduction = Math.round((1 - 1/count) * 100);
            document.getElementById('improvement-info').style.display = 'block';
            document.getElementById('improvement-info').innerHTML = 
                `✅ InstancedMesh ลด draw calls จาก ~${count} เหลือ 1<br>ประหยัด: ${reduction}%`;
        }
        
        // Mode 3: Merged Geometry
        function buildMerged(count) {
            clearObjects();
            
            const geometries = [];
            
            for (let i = 0; i < count; i++) {
                const s = 0.5 + Math.random() * 2;
                const geo = new THREE.BoxGeometry(s, s * 2, s);
                
                const matrix = new THREE.Matrix4();
                matrix.makeTranslation(
                    (Math.random() - 0.5) * 150,
                    s,
                    (Math.random() - 0.5) * 150
                );
                matrix.multiply(new THREE.Matrix4().makeRotationY(Math.random() * Math.PI * 2));
                geo.applyMatrix4(matrix);
                geometries.push(geo);
            }
            
            const merged = mergeGeometries(geometries);
            geometries.forEach(g => g.dispose());
            
            const mat = new THREE.MeshStandardMaterial({ color: 0x7788aa });
            const mesh = new THREE.Mesh(merged, mat);
            mesh.castShadow = true;
            scene.add(mesh);
            
            document.getElementById('improvement-info').style.display = 'block';
            document.getElementById('improvement-info').innerHTML = 
                `✅ Merged Geometry ลด draw calls จาก ~${count} เหลือ 1<br>⚠️ แต่ไม่สามารถ update transforms แยกกันได้`;
        }
        
        // Mode 4: LOD
        function buildLOD(count) {
            clearObjects();
            
            const matHigh = new THREE.MeshStandardMaterial({ color: 0x44ff88 });
            const matMed = new THREE.MeshStandardMaterial({ color: 0x44bb55 });
            const matLow = new THREE.MeshStandardMaterial({ color: 0x228833 });
            
            for (let i = 0; i < count; i++) {
                const lod = new THREE.LOD();
                
                const geoHigh = new THREE.CylinderGeometry(0.5, 0.8, 3, 12);
                const geoMed = new THREE.CylinderGeometry(0.5, 0.8, 3, 6);
                const geoLow = new THREE.CylinderGeometry(0.5, 0.8, 3, 4);
                
                lod.addLevel(new THREE.Mesh(geoHigh, matHigh), 0);
                lod.addLevel(new THREE.Mesh(geoMed, matMed), 20);
                lod.addLevel(new THREE.Mesh(geoLow, matLow), 50);
                lod.addLevel(new THREE.Object3D(), 120);
                
                lod.position.set(
                    (Math.random() - 0.5) * 150,
                    1.5,
                    (Math.random() - 0.5) * 150
                );
                scene.add(lod);
            }
            
            document.getElementById('improvement-info').style.display = 'block';
            document.getElementById('improvement-info').innerHTML = 
                `✅ LOD ลด vertex count ตามระยะห่าง<br>Draw calls ลดเมื่อ objects ไกลออกไป (Invisible)`;
        }
        
        // Initial build
        buildNaive(objectCount);
        
        // UI Events
        document.getElementById('count-slider').addEventListener('input', (e) => {
            objectCount = parseInt(e.target.value);
            document.getElementById('count-label').textContent = objectCount;
            
            if (currentMode === 'naive') buildNaive(objectCount);
            else if (currentMode === 'instanced') buildInstanced(objectCount);
            else if (currentMode === 'merged') buildMerged(objectCount);
            else if (currentMode === 'lod') buildLOD(objectCount);
        });
        
        function setMode(mode) {
            currentMode = mode;
            document.querySelectorAll('button').forEach(b => b.classList.remove('active'));
            document.getElementById(`btn-${mode}`).classList.add('active');
            
            if (mode === 'naive') buildNaive(objectCount);
            else if (mode === 'instanced') buildInstanced(objectCount);
            else if (mode === 'merged') buildMerged(objectCount);
            else if (mode === 'lod') buildLOD(objectCount);
        }
        
        document.getElementById('btn-naive').addEventListener('click', () => setMode('naive'));
        document.getElementById('btn-instanced').addEventListener('click', () => setMode('instanced'));
        document.getElementById('btn-merged').addEventListener('click', () => setMode('merged'));
        document.getElementById('btn-lod').addEventListener('click', () => setMode('lod'));
        
        // FPS tracking
        let lastTime = performance.now();
        let frameCount = 0;
        let fps = 0;
        
        // Resize handler
        function resize() {
            const w = window.innerWidth;
            const h = window.innerHeight;
            renderer.setSize(w, h);
            camera.aspect = w / h;
            camera.updateProjectionMatrix();
        }
        window.addEventListener('resize', resize);
        resize();
        
        // Animate
        function animate() {
            requestAnimationFrame(animate);
            
            // Update LOD
            if (currentMode === 'lod') {
                scene.traverse(obj => {
                    if (obj.isLOD) obj.update(camera);
                });
            }
            
            controls.update();
            renderer.render(scene, camera);
            
            // FPS calculation
            frameCount++;
            const now = performance.now();
            const elapsed = now - lastTime;
            if (elapsed >= 500) {
                fps = Math.round(frameCount / (elapsed / 1000));
                frameCount = 0;
                lastTime = now;
            }
            
            // Update metrics
            const info = renderer.info.render;
            document.getElementById('m-calls').textContent = info.calls;
            document.getElementById('m-triangles').textContent = info.triangles.toLocaleString();
            document.getElementById('m-fps').textContent = fps;
            document.getElementById('m-frametime').textContent = (1000/fps).toFixed(1);
            document.getElementById('m-memory').textContent = renderer.info.memory.geometries;
        }
        
        animate();
    </script>
</body>
</html>
```

---

## 11. City Scene Optimization Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>City Scene Optimization</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; color: #fff; font-family: sans-serif; }
        canvas { display: block; }
        #stats {
            position: fixed; top: 20px; right: 20px;
            background: rgba(0,0,0,0.85);
            padding: 15px; border-radius: 8px;
            border: 1px solid #0af;
            min-width: 250px;
        }
        .stat-row { display: flex; justify-content: space-between; margin: 4px 0; font-size: 13px; }
        .before { color: #f44; }
        .after { color: #4f4; }
        h3 { color: #0af; margin-bottom: 10px; }
        #toggle { margin-top: 10px; padding: 8px 16px; background: #0af; color: #000; border: none; border-radius: 4px; cursor: pointer; width: 100%; font-weight: bold; }
    </style>
</head>
<body>
<canvas id="c"></canvas>
<div id="stats">
    <h3>City Scene Metrics</h3>
    <div class="stat-row"><span>Mode:</span><span id="s-mode">Unoptimized</span></div>
    <div class="stat-row"><span>Draw Calls:</span><span id="s-calls">-</span></div>
    <div class="stat-row"><span>Triangles:</span><span id="s-tris">-</span></div>
    <div class="stat-row"><span>FPS:</span><span id="s-fps">-</span></div>
    <div class="stat-row"><span>Buildings:</span><span id="s-buildings">-</span></div>
    <div class="stat-row"><span>Draw Call Saved:</span><span id="s-saved" class="after">-</span></div>
    <button id="toggle">Switch to Optimized</button>
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
    
    const BUILDING_COUNT = 300;
    
    const renderer = new THREE.WebGLRenderer({ canvas: document.getElementById('c'), antialias: true });
    renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
    renderer.shadowMap.enabled = false; // disable for fair comparison
    
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x0a1a2a);
    scene.fog = new THREE.Fog(0x0a1a2a, 100, 300);
    
    const camera = new THREE.PerspectiveCamera(60, 1, 0.1, 500);
    camera.position.set(0, 60, 150);
    
    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.autoRotate = true;
    controls.autoRotateSpeed = 0.5;
    
    scene.add(new THREE.AmbientLight(0xffffff, 0.6));
    const sun = new THREE.DirectionalLight(0xfff5e0, 1.5);
    sun.position.set(100, 200, 100);
    scene.add(sun);
    
    // Ground
    const ground = new THREE.Mesh(
        new THREE.PlaneGeometry(500, 500),
        new THREE.MeshStandardMaterial({ color: 0x1a1a2a })
    );
    ground.rotation.x = -Math.PI / 2;
    scene.add(ground);
    
    let isOptimized = false;
    let savedCalls = 0;
    
    // Generate building data
    const buildingData = [];
    for (let i = 0; i < BUILDING_COUNT; i++) {
        const w = 2 + Math.random() * 5;
        const h = 5 + Math.random() * 30;
        const d = 2 + Math.random() * 5;
        const x = (Math.random() - 0.5) * 200;
        const z = (Math.random() - 0.5) * 200;
        buildingData.push({ w, h, d, x, z });
    }
    
    function clearScene() {
        const toRemove = [];
        scene.traverse(obj => {
            if (obj.isMesh && obj !== ground) toRemove.push(obj);
            if (obj.isInstancedMesh) toRemove.push(obj);
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
    
    function buildUnoptimized() {
        clearScene();
        const mat = new THREE.MeshStandardMaterial({ color: 0x334455 });
        buildingData.forEach(b => {
            const geo = new THREE.BoxGeometry(b.w, b.h, b.d);
            const mesh = new THREE.Mesh(geo, mat);
            mesh.position.set(b.x, b.h / 2, b.z);
            scene.add(mesh);
        });
        savedCalls = 0;
    }
    
    function buildOptimized() {
        clearScene();
        
        // Strategy 1: InstancedMesh for similar-sized buildings
        const smallGeo = new THREE.BoxGeometry(2, 1, 2);
        const tallGeo = new THREE.BoxGeometry(3, 1, 3);
        const mat = new THREE.MeshStandardMaterial({ color: 0x334455 });
        
        const smallBuildings = buildingData.filter(b => b.h < 15);
        const tallBuildings = buildingData.filter(b => b.h >= 15);
        
        // Small buildings as instances
        const smallMesh = new THREE.InstancedMesh(smallGeo, mat, smallBuildings.length);
        const matrix = new THREE.Matrix4();
        smallBuildings.forEach((b, i) => {
            matrix.makeScale(b.w / 2, b.h, b.d / 2);
            matrix.setPosition(b.x, b.h / 2, b.z);
            smallMesh.setMatrixAt(i, matrix);
        });
        smallMesh.instanceMatrix.needsUpdate = true;
        scene.add(smallMesh);
        
        // Tall buildings merged
        const tallGeos = tallBuildings.map(b => {
            const geo = new THREE.BoxGeometry(b.w, b.h, b.d);
            geo.applyMatrix4(new THREE.Matrix4().makeTranslation(b.x, b.h / 2, b.z));
            return geo;
        });
        if (tallGeos.length > 0) {
            const merged = mergeGeometries(tallGeos);
            tallGeos.forEach(g => g.dispose());
            scene.add(new THREE.Mesh(merged, new THREE.MeshStandardMaterial({ color: 0x445566 })));
        }
        
        savedCalls = BUILDING_COUNT - 3; // ~3 draw calls now
    }
    
    buildUnoptimized();
    
    document.getElementById('toggle').addEventListener('click', () => {
        isOptimized = !isOptimized;
        if (isOptimized) {
            buildOptimized();
            document.getElementById('toggle').textContent = 'Switch to Unoptimized';
            document.getElementById('s-mode').textContent = 'Optimized ✅';
        } else {
            buildUnoptimized();
            document.getElementById('toggle').textContent = 'Switch to Optimized';
            document.getElementById('s-mode').textContent = 'Unoptimized ❌';
        }
    });
    
    let lastTime = performance.now();
    let frames = 0;
    let fps = 0;
    
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
        
        document.getElementById('s-calls').textContent = renderer.info.render.calls;
        document.getElementById('s-tris').textContent = renderer.info.render.triangles.toLocaleString();
        document.getElementById('s-fps').textContent = fps;
        document.getElementById('s-buildings').textContent = BUILDING_COUNT;
        document.getElementById('s-saved').textContent = 
            savedCalls > 0 ? `~${savedCalls} calls saved (${Math.round(savedCalls/BUILDING_COUNT*100)}%)` : '-';
    }
    
    animate();
</script>
</body>
</html>
```

---

## 12. Material Sorting และ Render Queue

```javascript
// ใช้ renderOrder เพื่อควบคุมลำดับการวาด
// ค่าน้อย = วาดก่อน (default: 0)

// Opaque objects ควรวาดก่อน (front to back)
skybox.renderOrder = -1;         // วาดก่อนสุด
opaqueBuildings.renderOrder = 0; // วาดปกติ
transparentGlass.renderOrder = 1; // วาดทีหลัง (transparent)
particleEffects.renderOrder = 2;  // วาดหลังสุด

// Custom sort function สำหรับ render list
renderer.setRenderListSorting(
    (a, b) => {
        // Sort opaque objects front-to-back (ลด overdraw)
        if (!a.material.transparent && !b.material.transparent) {
            return a.z - b.z; // front to back
        }
        // Sort transparent objects back-to-front (painter's algorithm)
        if (a.material.transparent && b.material.transparent) {
            return b.z - a.z; // back to front
        }
        // Opaque before transparent
        return a.material.transparent ? 1 : -1;
    }
);
```

---

## 13. สรุปและเปรียบเทียบเทคนิค

| เทคนิค | Draw Call ลด | ข้อดี | ข้อเสีย |
|--------|-------------|-------|---------|
| InstancedMesh | N → 1 | เร็วมาก, animate แต่ละ instance ได้ | ต้องใช้ geometry/material เดียวกัน |
| Merged Geometry | N → 1 | Static scenes, ง่าย | ไม่สามารถ update แยกกันได้ |
| Texture Atlas | ลด material switches | ใช้ texture ต่างกันได้ | ซับซ้อนในการ setup |
| LOD | Variable | ลด vertex count | ต้องสร้างหลาย detail levels |
| Material Batching | ลด state changes | ไม่ต้อง rebuild scene | ผลไม่เยอะเท่าวิธีอื่น |

---

## แบบฝึกหัด

1. **ง่าย**: สร้าง scene ที่มี 1000 cubes โดยใช้ `InstancedMesh` และแสดงจำนวน draw calls
2. **กลาง**: เปรียบเทียบ FPS ระหว่าง naive vs instanced vs merged สำหรับ 500 objects พร้อม graph
3. **ยาก**: สร้าง city scene ที่ใช้ทั้ง InstancedMesh (buildings ซ้ำๆ), merged geometry (ground/roads), และ LOD (trees) และแสดง draw calls ก่อนและหลัง optimize
4. **Challenge**: implement automatic LOD system ที่สร้าง LOD levels โดยอัตโนมัติจาก high-detail geometry โดยใช้ SimplifyModifier

---

## อ้างอิง

- [Three.js WebGLRenderer.info](https://threejs.org/docs/#api/en/renderers/WebGLRenderer.info)
- [Three.js InstancedMesh](https://threejs.org/docs/#api/en/objects/InstancedMesh)
- [BufferGeometryUtils](https://threejs.org/docs/#examples/en/utils/BufferGeometryUtils)
- [WebGL Draw Call Best Practices](https://developer.chrome.com/docs/devtools/rendering/performance/)
- [Khronos WebGL Optimization Guide](https://www.khronos.org/webgl/)
