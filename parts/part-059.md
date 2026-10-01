# Part 59: Geometry Merging & Batching

## แนวคิด Geometry Merging

Geometry Merging รวม mesh หลาย objects เป็น mesh เดียว เพื่อลด draw calls แทนที่จะส่ง draw call แยกกัน 1000 ครั้ง เราส่งครั้งเดียวสำหรับ merged mesh ทั้งหมด

```
ก่อน merge: 1000 Meshes → 1000 draw calls
หลัง merge:  1 Mesh     → 1 draw call
```

**ข้อแตกต่างกับ InstancedMesh:**
- InstancedMesh: geometry เดิม แต่ต่าง transform/color → อัดแน่น, ยืดหยุ่น
- Merging: geometry ต่างกันได้ รวมเป็นหนึ่ง → static เท่านั้น, ไม่ยืดหยุ่น

## BufferGeometryUtils.mergeGeometries

```javascript
import * as THREE from 'three';
import { mergeGeometries } from 'three/examples/jsm/utils/BufferGeometryUtils.js';

// สร้าง geometry หลายชิ้น
const geo1 = new THREE.BoxGeometry(1, 1, 1);
const geo2 = new THREE.SphereGeometry(0.5, 8, 6);
const geo3 = new THREE.CylinderGeometry(0.3, 0.3, 1, 8);

// ย้ายตำแหน่งก่อน merge
const m1 = new THREE.Matrix4().makeTranslation(-2, 0, 0);
const m2 = new THREE.Matrix4().makeTranslation(0, 0, 0);
const m3 = new THREE.Matrix4().makeTranslation(2, 0, 0);

geo1.applyMatrix4(m1);
geo2.applyMatrix4(m2);
geo3.applyMatrix4(m3);

// Merge!
const mergedGeo = mergeGeometries([geo1, geo2, geo3]);
const mergedMesh = new THREE.Mesh(
    mergedGeo,
    new THREE.MeshStandardMaterial({ color: 0xffffff })
);
scene.add(mergedMesh);
```

## Merging กับหลาย Materials (Groups)

```javascript
// Merge geometries แต่เก็บ material groups
const geo1 = new THREE.BoxGeometry(1, 1, 1).applyMatrix4(
    new THREE.Matrix4().makeTranslation(-2, 0, 0)
);
const geo2 = new THREE.SphereGeometry(0.5).applyMatrix4(
    new THREE.Matrix4().makeTranslation(2, 0, 0)
);

// useGroups = true → สร้าง groups สำหรับแต่ละ geometry
const merged = mergeGeometries([geo1, geo2], true);

const materials = [
    new THREE.MeshStandardMaterial({ color: 0xff4444 }), // สำหรับ group 0 (box)
    new THREE.MeshStandardMaterial({ color: 0x4444ff })  // สำหรับ group 1 (sphere)
];

const mesh = new THREE.Mesh(merged, materials);
scene.add(mesh);
```

## Static vs Dynamic Draw Usage

```javascript
// StaticDrawUsage: ข้อมูลไม่เปลี่ยน → GPU เก็บ cache ได้ดี
const staticGeo = new THREE.BufferGeometry();
staticGeo.setAttribute('position', 
    new THREE.BufferAttribute(positions, 3).setUsage(THREE.StaticDrawUsage)
);

// DynamicDrawUsage: ข้อมูลเปลี่ยนบ่อย → GPU optimize สำหรับการ update
const dynamicGeo = new THREE.BufferGeometry();
dynamicGeo.setAttribute('position',
    new THREE.BufferAttribute(positions, 3).setUsage(THREE.DynamicDrawUsage)
);

// StreamDrawUsage: ข้อมูลเปลี่ยนทุก frame → เช่น particle systems
const streamGeo = new THREE.BufferGeometry();
streamGeo.setAttribute('position',
    new THREE.BufferAttribute(positions, 3).setUsage(THREE.StreamDrawUsage)
);
```

## Merge ด้วย Transform Matrix

```javascript
// Utility: merge meshes array เป็น single mesh
function mergeMeshes(meshes, material) {
    const geometries = [];
    
    meshes.forEach(mesh => {
        // Update world matrix ก่อน
        mesh.updateWorldMatrix(true, false);
        
        // Clone geometry และ apply world transform
        const geo = mesh.geometry.clone();
        geo.applyMatrix4(mesh.matrixWorld);
        geometries.push(geo);
    });
    
    const merged = mergeGeometries(geometries);
    
    // Cleanup
    geometries.forEach(g => g.dispose());
    
    return new THREE.Mesh(merged, material);
}

// ใช้งาน
const mergedCity = mergeMeshes(buildingMeshes, buildingMaterial);
scene.add(mergedCity);

// ลบ meshes เดิม
buildingMeshes.forEach(m => {
    scene.remove(m);
    m.geometry.dispose();
});
```

## ตัวอย่าง: City Builder with Merged Geometry

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>City Builder - Merged Geometry</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; font-family: monospace; }
        canvas { display: block; }
        #hud {
            position: fixed; top: 20px; left: 20px;
            background: rgba(5,10,20,0.9); color: #88ccff;
            padding: 16px; border-radius: 8px;
            border: 1px solid #2255aa; min-width: 220px;
        }
        h3 { color: #aaddff; margin-bottom: 10px; }
        .row { display: flex; justify-content: space-between; margin: 3px 0; font-size: 13px; }
        .val { color: #ffcc44; }
        #panel {
            position: fixed; bottom: 20px; left: 20px;
            background: rgba(5,10,20,0.9); color: #88ccff;
            padding: 16px; border-radius: 8px;
            border: 1px solid #2255aa; width: 300px;
        }
        .mode-btn {
            padding: 6px 12px; background: transparent;
            color: #88ccff; border: 1px solid #2255aa;
            cursor: pointer; border-radius: 4px; margin: 2px;
            font-size: 12px;
        }
        .mode-btn.active { background: rgba(34,85,170,0.4); }
        .ctrl { margin: 8px 0; }
        .ctrl label { display: flex; justify-content: space-between; font-size: 12px; margin-bottom: 3px; }
        .ctrl input { width: 100%; accent-color: #88ccff; }
    </style>
</head>
<body>
    <div id="hud">
        <h3>City Geometry</h3>
        <div class="row"><span>FPS</span><span class="val" id="fps">--</span></div>
        <div class="row"><span>Draw Calls</span><span class="val" id="draws">--</span></div>
        <div class="row"><span>Triangles</span><span class="val" id="tris">--</span></div>
        <div class="row"><span>Buildings</span><span class="val" id="bcount">--</span></div>
        <div class="row"><span>Geometries</span><span class="val" id="gcount">--</span></div>
    </div>
    <div id="panel">
        <div style="margin-bottom:8px">Render Mode:</div>
        <button class="mode-btn active" id="btnSeparate">Separate (No Merge)</button>
        <button class="mode-btn" id="btnMerged">Merged (1 Draw Call)</button>
        <button class="mode-btn" id="btnInstanced">Instanced</button>
        <div style="margin-top:10px;border-top:1px solid #223;padding-top:8px">
        <div class="ctrl">
            <label><span>Building Count</span><span id="bVal">200</span></label>
            <input type="range" id="bCount" min="50" max="2000" step="50" value="200">
        </div>
        </div>
        <button onclick="rebuild()" style="
            width:100%;padding:7px;background:transparent;
            color:#88ccff;border:1px solid #2255aa;cursor:pointer;
            border-radius:4px;margin-top:4px;font-size:12px;">
            Rebuild City
        </button>
    </div>

    <script type="module">
        import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js';
        import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/controls/OrbitControls.js';
        import { mergeGeometries } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/utils/BufferGeometryUtils.js';

        const W = window.innerWidth, H = window.innerHeight;
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(W, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        document.body.appendChild(renderer.domElement);

        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x080c14);
        scene.fog = new THREE.FogExp2(0x080c14, 0.005);

        const camera = new THREE.PerspectiveCamera(70, W/H, 0.5, 500);
        camera.position.set(0, 40, 80);

        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.target.set(0, 10, 0);

        scene.add(new THREE.AmbientLight(0x2244aa, 1.5));
        const moon = new THREE.DirectionalLight(0xaaccff, 2);
        moon.position.set(-50, 80, -50);
        scene.add(moon);
        const warmLight = new THREE.DirectionalLight(0xff8844, 0.5);
        warmLight.position.set(50, 20, 50);
        scene.add(warmLight);

        // Ground
        const ground = new THREE.Mesh(
            new THREE.PlaneGeometry(200, 200),
            new THREE.MeshStandardMaterial({ color: 0x1a1a2e, roughness: 0.95 })
        );
        ground.rotation.x = -Math.PI / 2;
        scene.add(ground);

        let currentMode = 'separate';
        let currentObjects = [];
        let buildingCount = 200;

        // สร้างสีแบบ city
        function getCityColor(type, height) {
            if (type === 'office') {
                return new THREE.Color().setHSL(0.6 + height * 0.05, 0.3, 0.4 + height * 0.1);
            } else if (type === 'residential') {
                return new THREE.Color().setHSL(0.08, 0.3, 0.4);
            } else {
                return new THREE.Color().setHSL(0.0, 0.0, 0.5);
            }
        }

        function generateBuildings(count) {
            const buildings = [];
            const types = ['office', 'residential', 'industrial'];
            const cityRadius = 60;

            for (let i = 0; i < count; i++) {
                const angle = Math.random() * Math.PI * 2;
                const radius = Math.sqrt(Math.random()) * cityRadius;
                const x = Math.cos(angle) * radius;
                const z = Math.sin(angle) * radius;

                const type = types[Math.floor(Math.random() * types.length)];
                const w = 1 + Math.random() * 4;
                const d = 1 + Math.random() * 4;
                let h;

                if (type === 'office') {
                    h = 5 + Math.random() * 25 * (1 - radius / cityRadius);
                } else if (type === 'residential') {
                    h = 3 + Math.random() * 8;
                } else {
                    h = 2 + Math.random() * 5;
                    w += 2; d += 2; // อาคารอุตสาหกรรมกว้างกว่า
                }

                buildings.push({ x, z, w, h, d, type });
            }
            return buildings;
        }

        function buildSeparate(buildings) {
            const meshes = [];
            buildings.forEach(b => {
                const geo = new THREE.BoxGeometry(b.w, b.h, b.d);
                const mat = new THREE.MeshStandardMaterial({
                    color: getCityColor(b.type, b.h / 30),
                    roughness: 0.7, metalness: 0.3
                });
                const mesh = new THREE.Mesh(geo, mat);
                mesh.position.set(b.x, b.h / 2, b.z);
                scene.add(mesh);
                meshes.push(mesh);
            });
            return meshes;
        }

        function buildMerged(buildings) {
            const geometries = [];
            const color = new THREE.Color();
            const colors = [];

            buildings.forEach(b => {
                const geo = new THREE.BoxGeometry(b.w, b.h, b.d);
                geo.translate(b.x, b.h / 2, b.z);

                // Add color attribute
                const count = geo.attributes.position.count;
                const colorArr = new Float32Array(count * 3);
                const c = getCityColor(b.type, b.h / 30);
                for (let i = 0; i < count; i++) {
                    colorArr[i * 3] = c.r;
                    colorArr[i * 3 + 1] = c.g;
                    colorArr[i * 3 + 2] = c.b;
                }
                geo.setAttribute('color', new THREE.BufferAttribute(colorArr, 3));
                geometries.push(geo);
            });

            const merged = mergeGeometries(geometries);
            geometries.forEach(g => g.dispose());

            const mat = new THREE.MeshStandardMaterial({
                vertexColors: true,
                roughness: 0.7,
                metalness: 0.3
            });
            const mesh = new THREE.Mesh(merged, mat);
            scene.add(mesh);
            return [mesh];
        }

        function buildInstanced(buildings) {
            const geo = new THREE.BoxGeometry(1, 1, 1);
            const mat = new THREE.MeshStandardMaterial({ vertexColors: true, roughness: 0.7, metalness: 0.3 });
            const mesh = new THREE.InstancedMesh(geo, mat, buildings.length);

            const dummy = new THREE.Object3D();
            const color = new THREE.Color();

            buildings.forEach((b, i) => {
                dummy.position.set(b.x, b.h / 2, b.z);
                dummy.scale.set(b.w, b.h, b.d);
                dummy.updateMatrix();
                mesh.setMatrixAt(i, dummy.matrix);

                color.copy(getCityColor(b.type, b.h / 30));
                mesh.setColorAt(i, color);
            });

            mesh.instanceMatrix.needsUpdate = true;
            if (mesh.instanceColor) mesh.instanceColor.needsUpdate = true;
            scene.add(mesh);
            return [mesh];
        }

        function clearObjects() {
            currentObjects.forEach(obj => {
                scene.remove(obj);
                if (obj.geometry) obj.geometry.dispose();
                if (obj.material) {
                    if (Array.isArray(obj.material)) obj.material.forEach(m => m.dispose());
                    else obj.material.dispose();
                }
            });
            currentObjects = [];
        }

        let buildings = generateBuildings(buildingCount);

        window.rebuild = function() {
            buildingCount = parseInt(document.getElementById('bCount').value);
            buildings = generateBuildings(buildingCount);
            clearObjects();

            if (currentMode === 'separate') currentObjects = buildSeparate(buildings);
            else if (currentMode === 'merged') currentObjects = buildMerged(buildings);
            else currentObjects = buildInstanced(buildings);

            document.getElementById('bcount').textContent = buildingCount;
        };

        function setMode(mode) {
            currentMode = mode;
            document.querySelectorAll('.mode-btn').forEach(b => b.classList.remove('active'));
            document.getElementById('btn' + mode.charAt(0).toUpperCase() + mode.slice(1)).classList.add('active');
            clearObjects();

            if (mode === 'separate') currentObjects = buildSeparate(buildings);
            else if (mode === 'merged') currentObjects = buildMerged(buildings);
            else currentObjects = buildInstanced(buildings);
        }

        document.getElementById('btnSeparate').addEventListener('click', () => setMode('separate'));
        document.getElementById('btnMerged').addEventListener('click', () => setMode('merged'));
        document.getElementById('btnInstanced').addEventListener('click', () => setMode('instanced'));
        document.getElementById('bCount').addEventListener('input', e => {
            document.getElementById('bVal').textContent = e.target.value;
        });

        // Initial build
        currentObjects = buildSeparate(buildings);
        document.getElementById('bcount').textContent = buildingCount;

        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });

        let fpsFrames = 0, fpsLast = performance.now();
        function animate() {
            requestAnimationFrame(animate);
            controls.update();

            fpsFrames++;
            const now = performance.now();
            if (now - fpsLast >= 500) {
                const fps = Math.round(fpsFrames * 1000 / (now - fpsLast));
                const info = renderer.info;
                document.getElementById('fps').textContent = fps;
                document.getElementById('draws').textContent = info.render.calls;
                document.getElementById('tris').textContent = info.render.triangles.toLocaleString();
                document.getElementById('gcount').textContent = info.memory.geometries;
                fpsFrames = 0;
                fpsLast = now;
            }

            renderer.render(scene, camera);
        }
        animate();
    </script>
</body>
</html>
```

## Merging กับ Per-Vertex Data

```javascript
// การเพิ่ม custom attributes ก่อน merge
function mergeWithCustomData(meshesWithData) {
    const geos = meshesWithData.map(({ mesh, data }) => {
        const geo = mesh.geometry.clone();
        geo.applyMatrix4(mesh.matrixWorld);
        
        // เพิ่ม custom attribute
        const customArr = new Float32Array(geo.attributes.position.count);
        customArr.fill(data.value);
        geo.setAttribute('customData', new THREE.BufferAttribute(customArr, 1));
        
        return geo;
    });
    
    return mergeGeometries(geos);
}
```

## Batch Rendering Pattern

```javascript
// Batch meshes ตาม material เพื่อลด state changes
class RenderBatch {
    constructor() {
        this.batches = new Map(); // material → geometries[]
    }
    
    add(geometry, material, matrix) {
        const matId = material.uuid;
        if (!this.batches.has(matId)) {
            this.batches.set(matId, { material, geometries: [] });
        }
        
        const geo = geometry.clone();
        geo.applyMatrix4(matrix);
        this.batches.get(matId).geometries.push(geo);
    }
    
    build(scene) {
        const meshes = [];
        
        this.batches.forEach(({ material, geometries }) => {
            const merged = mergeGeometries(geometries);
            geometries.forEach(g => g.dispose());
            
            const mesh = new THREE.Mesh(merged, material);
            scene.add(mesh);
            meshes.push(mesh);
        });
        
        this.batches.clear();
        return meshes;
    }
}
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Road Network
สร้างถนนด้วย merged geometry:
- ถนน straight segments
- Intersections
- ทางแยก T และ X

### แบบฝึกหัดที่ 2: Dungeon Level
สร้าง dungeon ด้วย merging:
- Floor tiles merged
- Wall segments merged ต่างกัน
- Ceiling merged แยก
- วัดผลเทียบกับ separate meshes

### แบบฝึกหัดที่ 3: Voxel Terrain
สร้าง Minecraft-style terrain:
- Merge เฉพาะ visible faces
- Chunk system (merge ต่อ chunk)
- Remerge เมื่อ chunk เปลี่ยน

### แบบฝึกหัดที่ 4: Benchmark Tool
สร้าง benchmark ที่:
- ทดสอบ separate vs merged vs instanced
- วัด FPS, draw calls, memory
- แสดง comparison chart

---

**สรุปบทที่ 59:**
- mergeGeometries รวม geometry หลายชิ้นเป็นหนึ่ง draw call
- useGroups = true รักษา material assignments
- ต้อง applyMatrix4 ก่อน merge เพื่อ bake world transform
- StaticDrawUsage เหมาะกับ static merged geometry
- Merged mesh ไม่ยืดหยุ่น แต่ draw call ต่ำมาก
- InstancedMesh ยืดหยุ่นกว่า แต่ต้องใช้ geometry เดิม

**บทต่อไป:** Frustum Culling & Occlusion
