# Part 60: Frustum Culling & Occlusion

## แนวคิด Frustum Culling

Frustum Culling คือการข้ามการ render object ที่อยู่นอก camera view frustum เพื่อประหยัด GPU โดย Three.js ทำ frustum culling อัตโนมัติด้วย bounding sphere

```
     /|
    / |  frustum (สามเหลี่ยมจาก camera)
   /  |
  /   |
 /    |   วัตถุนอก frustum → ไม่ render
camera
```

## Frustum Culling ใน Three.js

```javascript
// Three.js ทำ frustum culling อัตโนมัติ
// frustumCulled = true โดย default
mesh.frustumCulled = true; // default

// ปิด frustum culling (เช่น Sky, FullScreenQuad)
skyMesh.frustumCulled = false;

// Bounding sphere ใช้ตัดสินใจ
mesh.geometry.computeBoundingSphere();
mesh.geometry.boundingSphere; // Three.js ใช้ sphere นี้

// Force recompute เมื่อ geometry เปลี่ยน
geo.attributes.position.needsUpdate = true;
geo.computeBoundingSphere();
geo.computeBoundingBox();
```

## Manual Frustum Check

```javascript
// ตรวจสอบว่าวัตถุอยู่ใน frustum หรือไม่
const frustum = new THREE.Frustum();
const projScreenMatrix = new THREE.Matrix4();

function updateFrustum() {
    projScreenMatrix.multiplyMatrices(
        camera.projectionMatrix,
        camera.matrixWorldInverse
    );
    frustum.setFromProjectionMatrix(projScreenMatrix);
}

// ตรวจสอบ point
const point = new THREE.Vector3(10, 5, 0);
if (frustum.containsPoint(point)) {
    console.log('Point is visible!');
}

// ตรวจสอบ Box3
const box = new THREE.Box3(
    new THREE.Vector3(-1, -1, -1),
    new THREE.Vector3(1, 1, 1)
);
if (frustum.intersectsBox(box)) {
    console.log('Box is visible!');
}

// ตรวจสอบ Sphere
const sphere = new THREE.Sphere(new THREE.Vector3(0, 0, 0), 2);
if (frustum.intersectsSphere(sphere)) {
    console.log('Sphere is visible!');
}

// ตรวจสอบ Mesh (ใช้ bounding sphere)
function isMeshVisible(mesh, frustum) {
    if (!mesh.geometry.boundingSphere) {
        mesh.geometry.computeBoundingSphere();
    }
    
    const sphere = mesh.geometry.boundingSphere.clone();
    sphere.applyMatrix4(mesh.matrixWorld);
    
    return frustum.intersectsSphere(sphere);
}
```

## Spatial Partitioning

เพื่อ cull efficiently สำหรับ scene ขนาดใหญ่:

```javascript
// Simple Grid-based Culling
class GridCuller {
    constructor(worldSize, cellSize) {
        this.cellSize = cellSize;
        this.gridSize = Math.ceil(worldSize / cellSize);
        this.cells = new Map();
    }
    
    _cellKey(x, z) {
        const cx = Math.floor(x / this.cellSize);
        const cz = Math.floor(z / this.cellSize);
        return `${cx},${cz}`;
    }
    
    register(object, x, z) {
        const key = this._cellKey(x, z);
        if (!this.cells.has(key)) this.cells.set(key, []);
        this.cells.get(key).push(object);
        object.userData.gridKey = key;
    }
    
    getVisible(frustum) {
        const visible = [];
        
        this.cells.forEach((objects, key) => {
            const [cx, cz] = key.split(',').map(Number);
            
            // สร้าง cell box
            const box = new THREE.Box3(
                new THREE.Vector3(cx * this.cellSize, -10, cz * this.cellSize),
                new THREE.Vector3((cx + 1) * this.cellSize, 50, (cz + 1) * this.cellSize)
            );
            
            if (frustum.intersectsBox(box)) {
                visible.push(...objects);
            }
        });
        
        return visible;
    }
}
```

## BVH (Bounding Volume Hierarchy)

BVH ช่วยทำ raycast และ culling เร็วขึ้นมาก:

```javascript
// ใช้ three-mesh-bvh library
// npm install three-mesh-bvh

import { MeshBVH, acceleratedRaycast } from 'three-mesh-bvh';

// Patch Three.js Mesh เพื่อใช้ BVH raycast
THREE.Mesh.prototype.raycast = acceleratedRaycast;

// สร้าง BVH สำหรับ geometry
const geo = new THREE.TorusKnotGeometry(1, 0.3, 200, 50);
geo.boundsTree = new MeshBVH(geo);

const mesh = new THREE.Mesh(geo, material);
scene.add(mesh);

// Raycast ด้วย BVH (เร็วกว่า default มาก)
const raycaster = new THREE.Raycaster();
const intersections = [];
raycaster.intersectObject(mesh, false, intersections);

// BVH สำหรับ complex terrain (collision detection)
const terrain = new THREE.Mesh(terrainGeo, terrainMat);
terrainGeo.boundsTree = new MeshBVH(terrainGeo);

// Closest point on surface
const { point, face, distance } = geo.boundsTree.closestPointToPoint(
    targetPoint,
    new THREE.Vector3()
);
```

## renderer.info สำหรับ Monitoring

```javascript
// renderer.info ให้ข้อมูล real-time
function logRenderStats() {
    const info = renderer.info;
    
    console.log('=== Render Stats ===');
    console.log('Draw Calls:', info.render.calls);
    console.log('Triangles:', info.render.triangles);
    console.log('Lines:', info.render.lines);
    console.log('Points:', info.render.points);
    console.log('Geometries in GPU:', info.memory.geometries);
    console.log('Textures in GPU:', info.memory.textures);
}

// Reset ต่อ frame (auto)
// info.reset() ถูกเรียกโดย renderer.render()
```

## ตัวอย่าง: Spatial Partitioning + BVH Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Frustum Culling & Spatial Partitioning</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; font-family: monospace; }
        canvas { display: block; }
        #hud {
            position: fixed; top: 20px; left: 20px;
            background: rgba(0,0,0,0.85); color: #00ff88;
            padding: 16px; border-radius: 8px;
            border: 1px solid #00ff88; min-width: 230px;
        }
        h3 { color: #88ffcc; margin-bottom: 10px; }
        .row { display: flex; justify-content: space-between; margin: 4px 0; font-size: 13px; }
        .val { color: #ffcc44; }
        .highlight { color: #ff4466; }
        #panel {
            position: fixed; top: 20px; right: 20px;
            background: rgba(0,0,0,0.85); color: #ccc;
            padding: 16px; border-radius: 8px;
            border: 1px solid #444; width: 200px;
        }
        .toggle-btn {
            width: 100%; padding: 6px; background: transparent;
            color: #00ff88; border: 1px solid #00ff88;
            cursor: pointer; border-radius: 4px;
            margin: 3px 0; font-size: 12px;
        }
        .toggle-btn.off { color: #ff4466; border-color: #ff4466; }
    </style>
</head>
<body>
    <div id="hud">
        <h3>Culling Stats</h3>
        <div class="row"><span>FPS</span><span class="val" id="fps">--</span></div>
        <div class="row"><span>Draw Calls</span><span class="val" id="draws">--</span></div>
        <div class="row"><span>Triangles</span><span class="val" id="tris">--</span></div>
        <div class="row"><span>Total Objects</span><span class="val" id="total">--</span></div>
        <div class="row"><span>Visible</span><span class="val highlight" id="visible">--</span></div>
        <div class="row"><span>Culled</span><span class="val" id="culled">--</span></div>
        <div class="row"><span>Culling Mode</span><span class="val" id="mode">Auto</span></div>
    </div>
    <div id="panel">
        <div style="margin-bottom:8px;font-size:13px;color:#fff">Controls</div>
        <button class="toggle-btn" id="btnAuto">Three.js Auto Culling</button>
        <button class="toggle-btn" id="btnManual">Manual Frustum Check</button>
        <button class="toggle-btn" id="btnNone">No Culling</button>
        <hr style="border-color:#333;margin:8px 0">
        <button class="toggle-btn" id="btnShowFrustum">Show Frustum: OFF</button>
        <button class="toggle-btn" id="btnRotate">Auto Rotate: ON</button>
        <div style="margin-top:8px;font-size:11px;color:#777">
            WASD = Move<br>Mouse = Look
        </div>
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
        scene.background = new THREE.Color(0x040810);
        scene.fog = new THREE.Fog(0x040810, 100, 300);

        const camera = new THREE.PerspectiveCamera(75, W/H, 0.5, 400);
        camera.position.set(0, 10, 50);

        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;

        scene.add(new THREE.AmbientLight(0x334466, 1.0));
        const dirLight = new THREE.DirectionalLight(0xffffff, 1.5);
        dirLight.position.set(30, 60, 30);
        scene.add(dirLight);

        // Ground
        const gnd = new THREE.Mesh(
            new THREE.PlaneGeometry(300, 300, 30, 30),
            new THREE.MeshStandardMaterial({ color: 0x111824, wireframe: false })
        );
        gnd.rotation.x = -Math.PI / 2;
        scene.add(gnd);

        // สร้าง objects ใน grid
        const TOTAL = 3000;
        const SPREAD = 200;
        const objects = [];
        const positions = [];

        const geo = new THREE.IcosahedronGeometry(0.5, 1);
        const matVisible = new THREE.MeshStandardMaterial({ color: 0x00ff88, roughness: 0.5 });
        const matCulled = new THREE.MeshStandardMaterial({ color: 0x334455, roughness: 0.9 });

        for (let i = 0; i < TOTAL; i++) {
            const x = (Math.random() - 0.5) * SPREAD;
            const y = Math.random() * 5 + 0.5;
            const z = (Math.random() - 0.5) * SPREAD;

            const mesh = new THREE.Mesh(geo, matVisible.clone());
            mesh.position.set(x, y, z);
            mesh.userData.baseColor = new THREE.Color().setHSL(
                0.3 + Math.random() * 0.2, 0.7, 0.5
            );
            scene.add(mesh);
            objects.push(mesh);
            positions.push(new THREE.Vector3(x, y, z));
        }

        // Frustum checker
        const frustum = new THREE.Frustum();
        const projScreenMat = new THREE.Matrix4();
        const _sphere = new THREE.Sphere();

        let cullingMode = 'auto'; // auto | manual | none
        let autoRotate = true;

        // Frustum visualizer
        const frustumHelper = new THREE.CameraHelper(camera);
        frustumHelper.visible = false;
        scene.add(frustumHelper);

        let showFrustumViz = false;

        // UI
        const modes = ['btnAuto', 'btnManual', 'btnNone'];
        document.getElementById('btnAuto').addEventListener('click', () => {
            cullingMode = 'auto';
            document.getElementById('mode').textContent = 'Auto';
            objects.forEach(o => { o.frustumCulled = true; o.visible = true; });
        });
        document.getElementById('btnManual').addEventListener('click', () => {
            cullingMode = 'manual';
            document.getElementById('mode').textContent = 'Manual';
            objects.forEach(o => { o.frustumCulled = false; });
        });
        document.getElementById('btnNone').addEventListener('click', () => {
            cullingMode = 'none';
            document.getElementById('mode').textContent = 'None';
            objects.forEach(o => { o.frustumCulled = false; o.visible = true; });
        });

        document.getElementById('btnShowFrustum').addEventListener('click', (e) => {
            showFrustumViz = !showFrustumViz;
            frustumHelper.visible = showFrustumViz;
            e.target.textContent = `Show Frustum: ${showFrustumViz ? 'ON' : 'OFF'}`;
        });
        document.getElementById('btnRotate').addEventListener('click', (e) => {
            autoRotate = !autoRotate;
            controls.autoRotate = autoRotate;
            e.target.textContent = `Auto Rotate: ${autoRotate ? 'ON' : 'OFF'}`;
        });

        controls.autoRotate = true;
        controls.autoRotateSpeed = 0.5;

        document.getElementById('total').textContent = TOTAL;

        let fpsFrames = 0, fpsLast = performance.now();
        let visibleCount = TOTAL;

        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });

        const clock = new THREE.Clock();
        function animate() {
            requestAnimationFrame(animate);
            const t = clock.getElapsedTime();

            controls.update();

            if (cullingMode === 'manual') {
                // Manual frustum culling
                projScreenMat.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
                frustum.setFromProjectionMatrix(projScreenMat);

                visibleCount = 0;
                objects.forEach((obj, i) => {
                    if (!obj.geometry.boundingSphere) obj.geometry.computeBoundingSphere();
                    _sphere.copy(obj.geometry.boundingSphere);
                    _sphere.applyMatrix4(obj.matrixWorld);

                    const isVisible = frustum.intersectsSphere(_sphere);
                    obj.visible = isVisible;
                    if (isVisible) {
                        visibleCount++;
                        obj.material.color.copy(obj.userData.baseColor);
                    }
                });
            } else if (cullingMode === 'auto') {
                // Auto: Three.js จะทำ culling เอง
                // เราแค่นับ visible objects หลัง render
                objects.forEach(o => o.visible = true);
                visibleCount = TOTAL; // Three.js culls internally
            } else {
                visibleCount = TOTAL;
            }

            if (frustumHelper.visible) frustumHelper.update();

            // FPS
            fpsFrames++;
            const now = performance.now();
            if (now - fpsLast >= 500) {
                const fps = Math.round(fpsFrames * 1000 / (now - fpsLast));
                document.getElementById('fps').textContent = fps;
                document.getElementById('draws').textContent = renderer.info.render.calls;
                document.getElementById('tris').textContent = renderer.info.render.triangles.toLocaleString();
                document.getElementById('visible').textContent = cullingMode === 'manual'
                    ? visibleCount
                    : renderer.info.render.calls - 2; // approx
                document.getElementById('culled').textContent = cullingMode === 'manual'
                    ? (TOTAL - visibleCount)
                    : 'auto';
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

## Occlusion Culling (Concept)

```javascript
// Hardware Occlusion Query (WebGL extension)
// ใช้ OcclusionQuery ของ WebGL สำหรับ GPU-side occlusion test

class OcclusionCuller {
    constructor(renderer) {
        this.renderer = renderer;
        this.gl = renderer.getContext();
        this.ext = this.gl.getExtension('EXT_disjoint_timer_query_webgl2');
    }
    
    // วิธีง่าย: render proxy geometry (ขนาดเล็ก)
    // ถ้า proxy ไม่ผ่าน depth test → วัตถุ occluded → ไม่ render
    
    // วิธีที่ใช้ใน production:
    // 1. Render occluders (terrain, buildings) ลงใน depth buffer
    // 2. Test วัตถุแต่ละชิ้นกับ depth buffer
    // 3. ถ้า occluded → skip draw call
}
```

## Portal Culling

```javascript
// Portal culling สำหรับ indoor scenes
class Portal {
    constructor(roomA, roomB, geometry) {
        this.roomA = roomA;
        this.roomB = roomB;
        this.geometry = geometry; // quad ที่เป็นประตู
    }
    
    isVisible(camera) {
        // ตรวจว่ากล้องอยู่ด้าน A หรือ B
        // ถ้าหน้าต่างประตู visible → render room ที่ต่อไป
    }
}

class Room {
    constructor(objects, portals) {
        this.objects = objects;
        this.portals = portals;
    }
    
    render(renderer, camera, scene, depth = 0) {
        if (depth > 3) return; // ป้องกัน infinite recursion
        
        this.objects.forEach(o => o.visible = true);
        
        for (const portal of this.portals) {
            if (portal.isVisible(camera)) {
                portal.getConnectedRoom(this).render(renderer, camera, scene, depth + 1);
            }
        }
    }
}
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Dynamic Culling System
สร้างระบบ culling สำหรับ city:
- Grid-based spatial partitioning
- Cell visibility ตาม frustum
- Debug visualization แสดง cells

### แบบฝึกหัดที่ 2: BVH Raycast
ติดตั้ง three-mesh-bvh และ:
- Load complex model (สร้าง TorusKnot ที่มี polygon สูง)
- เปรียบเทียบ default raycast vs BVH
- แสดง time difference

### แบบฝึกหัดที่ 3: View Distance System
สร้าง LOD + Culling ร่วมกัน:
- Objects ไกล → ไม่ render เลย
- Objects กลาง → low LOD
- Objects ใกล้ → high LOD

### แบบฝึกหัดที่ 4: Occlusion Volumes
สร้าง simple occlusion culling:
- กำหนด occluder boxes (buildings)
- Test objects กับ occluder ด้วย ray test
- ซ่อน objects ที่ blocked

---

**สรุปบทที่ 60:**
- Three.js ทำ frustum culling อัตโนมัติด้วย bounding sphere
- frustumCulled = false ปิด culling สำหรับ object นั้นๆ
- THREE.Frustum + frustum.setFromProjectionMatrix() ทำ manual check
- Grid partitioning ช่วย cull เป็น cell แทน object ทีละตัว
- three-mesh-bvh เพิ่มความเร็ว raycast อย่างมากสำหรับ complex geometry
- renderer.info.render.calls ติดตาม draw calls ได้ real-time

**บทต่อไป:** Draw Call Optimization
