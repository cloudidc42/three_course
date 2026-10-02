# Part 105: Occlusion Culling (Advanced)

## บทนำ: ทำไม Frustum Culling ไม่พอ

Three.js มี **Frustum Culling** ในตัว - ระบบที่ไม่วาด objects ที่อยู่นอก camera frustum (cone ของการมองเห็น) แต่มีปัญหาสำคัญที่เรียกว่า **Overdraw**:

```
Camera มองจากซ้ายไปขวา:
         [กำแพง A (ใกล้)] [กำแพง B] [กำแพง C (ไกล)]
Camera →     ████████████   ████████   ████████
```

ทั้ง A, B, C อยู่ใน frustum ทั้งหมด แต่จริงๆ เราเห็นแค่ A
GPU ต้องวาด B และ C แม้จะถูก A บังอยู่ทั้งหมด = **Overdraw**

**ปัญหาของ Overdraw:**
- Pixel shader รันทุก pixel ของ B และ C (wasted computation)
- Fragment depth testing ทำงาน แต่ถ้าวาด opaque objects back-to-front ยิ่งแย่
- เป็นปัญหาใหญ่สำหรับ indoor scenes, dense urban environments

---

## 1. Software Occlusion Culling ด้วย BVH

**BVH (Bounding Volume Hierarchy)** เป็นโครงสร้างข้อมูลต้นไม้ที่จัดกลุ่ม objects ตาม spatial position

```
BVH Tree:
              [Root AABB]
             /            \
      [Left AABB]      [Right AABB]
       /       \          /       \
  [Leaf]   [Leaf]    [Leaf]    [Leaf]
```

```javascript
// ติดตั้ง three-mesh-bvh library
// npm install three-mesh-bvh
// หรือใช้ CDN:
// <script src="https://cdn.jsdelivr.net/npm/three-mesh-bvh@0.7.1/build/index.umd.cjs"></script>

import { MeshBVH, CONTAINED, NOT_INTERSECTED, INTERSECTED } from 'three-mesh-bvh';

// สร้าง BVH สำหรับ scene geometry
function buildSceneBVH(scene) {
    const allGeometries = [];
    
    scene.traverse((obj) => {
        if (obj.isMesh) {
            obj.updateWorldMatrix(true, false);
            const geo = obj.geometry.clone();
            geo.applyMatrix4(obj.matrixWorld);
            allGeometries.push({ geo, obj });
        }
    });
    
    return allGeometries;
}

// Frustum + BVH culling
class BVHCuller {
    constructor(camera) {
        this.camera = camera;
        this.frustum = new THREE.Frustum();
        this.projScreenMatrix = new THREE.Matrix4();
    }
    
    update() {
        this.projScreenMatrix.multiplyMatrices(
            this.camera.projectionMatrix,
            this.camera.matrixWorldInverse
        );
        this.frustum.setFromProjectionMatrix(this.projScreenMatrix);
    }
    
    // ตรวจสอบว่า object อยู่ใน frustum หรือไม่
    isVisible(object) {
        if (!object.geometry) return true;
        
        const sphere = new THREE.Sphere();
        object.geometry.computeBoundingSphere();
        sphere.copy(object.geometry.boundingSphere);
        sphere.applyMatrix4(object.matrixWorld);
        
        return this.frustum.intersectsSphere(sphere);
    }
    
    // Cull scene
    cullScene(scene) {
        this.update();
        let hidden = 0, visible = 0;
        
        scene.traverse((obj) => {
            if (obj.isMesh) {
                const wasVisible = obj.visible;
                obj.visible = this.isVisible(obj);
                
                if (obj.visible) visible++;
                else hidden++;
            }
        });
        
        return { visible, hidden };
    }
}
```

---

## 2. Portal-Based Culling สำหรับ Indoor Scenes

Portal culling ทำงานโดยแบ่ง scene เป็น "ห้อง" (cells) ที่เชื่อมกันด้วย "ประตู" (portals)

```javascript
// โครงสร้างข้อมูลสำหรับ Portal System
class Room {
    constructor(id, bounds) {
        this.id = id;
        this.bounds = bounds; // THREE.Box3
        this.objects = [];    // Objects ในห้องนี้
        this.portals = [];    // Portal connections
    }
    
    addObject(mesh) {
        this.objects.push(mesh);
    }
    
    addPortal(portal) {
        this.portals.push(portal);
    }
}

class Portal {
    constructor(roomA, roomB, vertices) {
        this.roomA = roomA;  // Room object ด้านหนึ่ง
        this.roomB = roomB;  // Room object อีกด้านหนึ่ง
        
        // Portal polygon (quad)
        this.vertices = vertices; // Array of THREE.Vector3 (4 points)
        
        // คำนวณ plane ของ portal
        const v0 = vertices[0], v1 = vertices[1], v2 = vertices[2];
        const edge1 = v1.clone().sub(v0);
        const edge2 = v2.clone().sub(v0);
        this.normal = edge1.cross(edge2).normalize();
        this.plane = new THREE.Plane().setFromNormalAndCoplanarPoint(this.normal, v0);
    }
    
    // ตรวจสอบว่า camera เห็น portal นี้หรือไม่
    isVisibleFrom(cameraPos, frustum) {
        // ตรวจสอบว่า camera อยู่ด้านที่ถูกต้องของ portal plane
        const dot = this.plane.distanceToPoint(cameraPos);
        
        // ถ้า dot < 0 = camera อยู่ด้านหลัง portal (ไม่เห็น)
        if (dot < 0) return false;
        
        // ตรวจสอบว่า portal quad อยู่ใน frustum หรือไม่
        for (const v of this.vertices) {
            if (frustum.containsPoint(v)) return true;
        }
        
        return false;
    }
}

class PortalCuller {
    constructor() {
        this.rooms = new Map();
        this.visibleRooms = new Set();
    }
    
    addRoom(room) {
        this.rooms.set(room.id, room);
    }
    
    // หา room ที่ camera อยู่ใน
    findCameraRoom(cameraPos) {
        for (const [id, room] of this.rooms) {
            if (room.bounds.containsPoint(cameraPos)) {
                return room;
            }
        }
        return null;
    }
    
    // BFS traversal ผ่าน portals
    computeVisibility(camera) {
        this.visibleRooms.clear();
        
        const frustum = new THREE.Frustum();
        const projScreen = new THREE.Matrix4()
            .multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
        frustum.setFromProjectionMatrix(projScreen);
        
        const cameraRoom = this.findCameraRoom(camera.position);
        if (!cameraRoom) return;
        
        // BFS
        const queue = [{ room: cameraRoom, clipFrustum: frustum }];
        this.visibleRooms.add(cameraRoom.id);
        
        while (queue.length > 0) {
            const { room, clipFrustum } = queue.shift();
            
            for (const portal of room.portals) {
                const neighborRoom = portal.roomA.id === room.id 
                    ? portal.roomB 
                    : portal.roomA;
                
                if (this.visibleRooms.has(neighborRoom.id)) continue;
                
                if (portal.isVisibleFrom(camera.position, clipFrustum)) {
                    this.visibleRooms.add(neighborRoom.id);
                    queue.push({ room: neighborRoom, clipFrustum });
                }
            }
        }
        
        // Apply visibility
        for (const [id, room] of this.rooms) {
            const visible = this.visibleRooms.has(id);
            room.objects.forEach(obj => { obj.visible = visible; });
        }
        
        return {
            visibleRooms: this.visibleRooms.size,
            totalRooms: this.rooms.size
        };
    }
}
```

---

## 3. Grid-Based Spatial Subdivision

```javascript
// แบ่ง scene เป็น grid cells และทำ culling ต่อ cell
class GridCuller {
    constructor(worldSize = 200, cellSize = 10) {
        this.worldSize = worldSize;
        this.cellSize = cellSize;
        this.gridSize = Math.ceil(worldSize / cellSize);
        this.cells = new Map(); // key -> Set of objects
    }
    
    // หา cell key จาก world position
    _key(x, z) {
        const cx = Math.floor((x + this.worldSize / 2) / this.cellSize);
        const cz = Math.floor((z + this.worldSize / 2) / this.cellSize);
        return `${cx}_${cz}`;
    }
    
    // เพิ่ม object ลงใน grid
    add(obj) {
        const pos = new THREE.Vector3();
        obj.getWorldPosition(pos);
        const key = this._key(pos.x, pos.z);
        
        if (!this.cells.has(key)) this.cells.set(key, new Set());
        this.cells.get(key).add(obj);
        obj.userData._gridKey = key;
    }
    
    // ลบ object ออกจาก grid
    remove(obj) {
        const key = obj.userData._gridKey;
        if (key && this.cells.has(key)) {
            this.cells.get(key).delete(obj);
        }
    }
    
    // Cull: แสดงเฉพาะ cells ที่อยู่ใน frustum
    cull(camera, renderDistance = 100) {
        const frustum = new THREE.Frustum();
        frustum.setFromProjectionMatrix(
            new THREE.Matrix4().multiplyMatrices(
                camera.projectionMatrix,
                camera.matrixWorldInverse
            )
        );
        
        const camPos = camera.position;
        let visible = 0, hidden = 0;
        
        for (const [key, objects] of this.cells) {
            // Parse cell position
            const [cx, cz] = key.split('_').map(Number);
            const worldX = cx * this.cellSize - this.worldSize / 2 + this.cellSize / 2;
            const worldZ = cz * this.cellSize - this.worldSize / 2 + this.cellSize / 2;
            
            // Distance culling
            const dist = Math.sqrt(
                (worldX - camPos.x) ** 2 + (worldZ - camPos.z) ** 2
            );
            
            // Frustum check สำหรับ cell bounding box
            const box = new THREE.Box3(
                new THREE.Vector3(worldX - this.cellSize/2, -50, worldZ - this.cellSize/2),
                new THREE.Vector3(worldX + this.cellSize/2, 50, worldZ + this.cellSize/2)
            );
            
            const cellVisible = dist < renderDistance && frustum.intersectsBox(box);
            
            objects.forEach(obj => {
                obj.visible = cellVisible;
                if (cellVisible) visible++;
                else hidden++;
            });
        }
        
        return { visible, hidden };
    }
}
```

---

## 4. PVS (Potentially Visible Set) Concept

```javascript
// PVS = Pre-computed Visibility Set
// คำนวณล่วงหน้าว่าจากจุด A สามารถเห็น objects ชุดไหนได้
// เหมาะสำหรับ static indoor scenes

class PVSSystem {
    constructor() {
        this.pvsSets = new Map(); // cellId -> Set of visible cellIds
    }
    
    // Pre-compute PVS (ทำครั้งเดียว - ช้า แต่ runtime เร็วมาก)
    precompute(rooms, raycaster) {
        console.time('PVS Precompute');
        
        for (const [roomId, room] of rooms) {
            const visibleSet = new Set();
            visibleSet.add(roomId); // เห็นตัวเอง
            
            // Sample หลาย positions ภายใน room
            const samples = this._sampleRoomPoints(room, 8);
            
            for (const samplePos of samples) {
                for (const [otherRoomId, otherRoom] of rooms) {
                    if (otherRoomId === roomId) continue;
                    
                    // Test visibility ด้วย raycast
                    const otherSamples = this._sampleRoomPoints(otherRoom, 4);
                    
                    for (const otherPos of otherSamples) {
                        const dir = otherPos.clone().sub(samplePos).normalize();
                        raycaster.set(samplePos, dir);
                        const hits = raycaster.intersectObjects(/* walls */ []);
                        
                        if (hits.length === 0 || hits[0].distance > samplePos.distanceTo(otherPos)) {
                            visibleSet.add(otherRoomId);
                            break;
                        }
                    }
                }
            }
            
            this.pvsSets.set(roomId, visibleSet);
        }
        
        console.timeEnd('PVS Precompute');
    }
    
    _sampleRoomPoints(room, count) {
        const points = [];
        const center = room.bounds.getCenter(new THREE.Vector3());
        const size = room.bounds.getSize(new THREE.Vector3());
        
        for (let i = 0; i < count; i++) {
            points.push(new THREE.Vector3(
                center.x + (Math.random() - 0.5) * size.x * 0.8,
                center.y + 0.5, // Eye level
                center.z + (Math.random() - 0.5) * size.z * 0.8
            ));
        }
        return points;
    }
    
    // Runtime: ใช้ PVS ที่คำนวณไว้แล้ว (เร็วมาก)
    applyVisibility(currentRoomId, rooms) {
        const visibleSet = this.pvsSets.get(currentRoomId);
        if (!visibleSet) return;
        
        for (const [roomId, room] of rooms) {
            const visible = visibleSet.has(roomId);
            room.objects.forEach(obj => { obj.visible = visible; });
        }
    }
}
```

---

## 5. Interactive Indoor Scene Demo with Portal Culling

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Portal Culling Demo</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; color: #eee; font-family: monospace; overflow: hidden; }
        canvas { display: block; }
        #ui {
            position: fixed; top: 20px; right: 20px;
            background: rgba(0,0,0,0.85);
            padding: 20px; border-radius: 8px;
            border: 1px solid #0af; min-width: 280px;
        }
        h2 { color: #0af; margin-bottom: 12px; }
        .row { display: flex; justify-content: space-between; margin: 6px 0; font-size: 13px; }
        .val { font-weight: bold; color: #4f4; }
        .bad { color: #f44; }
        #minimap {
            width: 200px; height: 200px;
            border: 1px solid #446;
            margin-top: 10px; position: relative;
        }
        #controls-info { 
            margin-top: 10px; font-size: 11px; color: #888;
            padding: 8px; background: rgba(255,255,255,0.05);
            border-radius: 4px;
        }
        .legend { display: flex; align-items: center; gap: 6px; margin: 3px 0; font-size: 11px; }
        .dot { width: 10px; height: 10px; border-radius: 50%; }
    </style>
</head>
<body>
<canvas id="c"></canvas>
<div id="ui">
    <h2>🔲 Portal Culling Demo</h2>
    
    <div class="row"><span>Draw Calls:</span><span id="m-calls" class="val">-</span></div>
    <div class="row"><span>FPS:</span><span id="m-fps" class="val">-</span></div>
    <div class="row"><span>Visible Rooms:</span><span id="m-vrooms" class="val">-</span></div>
    <div class="row"><span>Total Rooms:</span><span id="m-trooms" class="val">-</span></div>
    <div class="row"><span>Hidden Objects:</span><span id="m-hidden" class="bad">-</span></div>
    <div class="row"><span>Camera Room:</span><span id="m-camroom" class="val">-</span></div>
    <div class="row"><span>Portal Culling:</span><span id="m-culling" class="val">-</span></div>
    
    <canvas id="minimap"></canvas>
    
    <div class="legend">
        <div class="dot" style="background:#0af"></div> Camera
        <div class="dot" style="background:#4f4"></div> Visible Room  
        <div class="dot" style="background:#444"></div> Hidden Room
    </div>
    
    <div id="controls-info">
        WASD: เดินหน้า/ถอยหลัง/ซ้าย/ขวา<br>
        Mouse: หมุนมุมมอง<br>
        Toggle Culling: กด C
    </div>
    
    <button id="btn-toggle" style="margin-top:10px;padding:8px 16px;background:#0af;color:#000;border:none;border-radius:4px;cursor:pointer;width:100%;font-weight:bold;">
        Toggle Culling
    </button>
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
    import { PointerLockControls } from 'three/addons/controls/PointerLockControls.js';
    
    const renderer = new THREE.WebGLRenderer({ canvas: document.getElementById('c'), antialias: true });
    renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
    renderer.shadowMap.enabled = true;
    
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x050510);
    
    const camera = new THREE.PerspectiveCamera(80, 1, 0.1, 200);
    camera.position.set(0, 1.7, 0); // Eye height
    
    // First person controls
    let moveForward = false, moveBackward = false, moveLeft = false, moveRight = false;
    
    document.addEventListener('keydown', e => {
        if (e.key === 'w' || e.key === 'W' || e.key === 'ArrowUp') moveForward = true;
        if (e.key === 's' || e.key === 'S' || e.key === 'ArrowDown') moveBackward = true;
        if (e.key === 'a' || e.key === 'A' || e.key === 'ArrowLeft') moveLeft = true;
        if (e.key === 'd' || e.key === 'D' || e.key === 'ArrowRight') moveRight = true;
        if (e.key === 'c' || e.key === 'C') toggleCulling();
    });
    document.addEventListener('keyup', e => {
        if (e.key === 'w' || e.key === 'W' || e.key === 'ArrowUp') moveForward = false;
        if (e.key === 's' || e.key === 'S' || e.key === 'ArrowDown') moveBackward = false;
        if (e.key === 'a' || e.key === 'A' || e.key === 'ArrowLeft') moveLeft = false;
        if (e.key === 'd' || e.key === 'D' || e.key === 'ArrowRight') moveRight = false;
    });
    
    // Simple FPS camera
    let yaw = 0, pitch = 0;
    const isDragging = { value: false };
    const lastMouse = { x: 0, y: 0 };
    
    renderer.domElement.addEventListener('mousedown', e => { isDragging.value = true; lastMouse.x = e.clientX; lastMouse.y = e.clientY; });
    document.addEventListener('mouseup', () => { isDragging.value = false; });
    document.addEventListener('mousemove', e => {
        if (!isDragging.value) return;
        yaw -= (e.clientX - lastMouse.x) * 0.002;
        pitch -= (e.clientY - lastMouse.y) * 0.002;
        pitch = Math.max(-Math.PI/3, Math.min(Math.PI/3, pitch));
        lastMouse.x = e.clientX; lastMouse.y = e.clientY;
    });
    
    // Lighting
    scene.add(new THREE.AmbientLight(0x223355, 0.5));
    
    // ============== Build Rooms ==============
    const ROOM_COUNT = 6;
    const ROOM_SIZE = 10;
    const WALL_HEIGHT = 3;
    
    // Room layout (2x3 grid)
    const roomPositions = [
        { x: 0, z: 0 }, { x: ROOM_SIZE + 1, z: 0 }, { x: (ROOM_SIZE + 1) * 2, z: 0 },
        { x: 0, z: ROOM_SIZE + 1 }, { x: ROOM_SIZE + 1, z: ROOM_SIZE + 1 }, { x: (ROOM_SIZE + 1) * 2, z: ROOM_SIZE + 1 }
    ];
    
    const roomColors = [0xff4444, 0x44ff44, 0x4444ff, 0xffff44, 0xff44ff, 0x44ffff];
    
    class SimpleRoom {
        constructor(id, cx, cz, color) {
            this.id = id;
            this.cx = cx; this.cz = cz;
            this.color = color;
            this.objects = [];
            this.portals = [];
            this.bounds = new THREE.Box3(
                new THREE.Vector3(cx - ROOM_SIZE/2, 0, cz - ROOM_SIZE/2),
                new THREE.Vector3(cx + ROOM_SIZE/2, WALL_HEIGHT, cz + ROOM_SIZE/2)
            );
        }
        
        addObject(obj) { this.objects.push(obj); }
    }
    
    const rooms = [];
    const roomMeshGroups = [];
    
    const floorMat = new THREE.MeshStandardMaterial({ roughness: 0.8 });
    const ceilingMat = new THREE.MeshStandardMaterial({ color: 0x333333 });
    
    roomPositions.forEach((pos, i) => {
        const cx = pos.x + ROOM_SIZE/2 - (ROOM_SIZE + 1);
        const cz = pos.z + ROOM_SIZE/2 - (ROOM_SIZE + 1);
        const room = new SimpleRoom(i, cx, cz, roomColors[i]);
        
        const group = new THREE.Group();
        
        // Floor
        const floorGeo = new THREE.PlaneGeometry(ROOM_SIZE - 0.1, ROOM_SIZE - 0.1);
        floorGeo.rotateX(-Math.PI/2);
        const floorMesh = new THREE.Mesh(floorGeo, new THREE.MeshStandardMaterial({ color: roomColors[i], roughness: 0.9 }));
        floorMesh.position.set(cx, 0.01, cz);
        floorMesh.receiveShadow = true;
        group.add(floorMesh);
        
        // Ceiling
        const ceilMesh = new THREE.Mesh(
            new THREE.PlaneGeometry(ROOM_SIZE - 0.1, ROOM_SIZE - 0.1),
            ceilingMat
        );
        ceilMesh.rotation.x = Math.PI/2;
        ceilMesh.position.set(cx, WALL_HEIGHT, cz);
        group.add(ceilMesh);
        
        // Walls (with portal openings)
        const wallMat = new THREE.MeshStandardMaterial({ color: 0x445566 });
        
        // North wall
        const nWall = new THREE.Mesh(
            new THREE.BoxGeometry(ROOM_SIZE, WALL_HEIGHT, 0.2),
            wallMat
        );
        nWall.position.set(cx, WALL_HEIGHT/2, cz - ROOM_SIZE/2);
        nWall.castShadow = true;
        group.add(nWall);
        
        // South wall
        const sWall = new THREE.Mesh(
            new THREE.BoxGeometry(ROOM_SIZE, WALL_HEIGHT, 0.2),
            wallMat
        );
        sWall.position.set(cx, WALL_HEIGHT/2, cz + ROOM_SIZE/2);
        sWall.castShadow = true;
        group.add(sWall);
        
        // East wall
        const eWall = new THREE.Mesh(
            new THREE.BoxGeometry(0.2, WALL_HEIGHT, ROOM_SIZE),
            wallMat
        );
        eWall.position.set(cx + ROOM_SIZE/2, WALL_HEIGHT/2, cz);
        eWall.castShadow = true;
        group.add(eWall);
        
        // West wall
        const wWall = new THREE.Mesh(
            new THREE.BoxGeometry(0.2, WALL_HEIGHT, ROOM_SIZE),
            wallMat
        );
        wWall.position.set(cx - ROOM_SIZE/2, WALL_HEIGHT/2, cz);
        wWall.castShadow = true;
        group.add(wWall);
        
        // Furniture (random objects in room)
        const furnitureMat = new THREE.MeshStandardMaterial({ color: roomColors[i], roughness: 0.5 });
        for (let f = 0; f < 5; f++) {
            const fw = 0.5 + Math.random() * 1.5;
            const fh = 0.5 + Math.random() * 2;
            const fd = 0.5 + Math.random() * 1.5;
            const furniture = new THREE.Mesh(new THREE.BoxGeometry(fw, fh, fd), furnitureMat);
            furniture.position.set(
                cx + (Math.random() - 0.5) * (ROOM_SIZE - fw - 2),
                fh / 2,
                cz + (Math.random() - 0.5) * (ROOM_SIZE - fd - 2)
            );
            furniture.castShadow = true;
            furniture.receiveShadow = true;
            group.add(furniture);
        }
        
        // Point light in each room
        const light = new THREE.PointLight(roomColors[i], 1.5, ROOM_SIZE * 1.5);
        light.position.set(cx, WALL_HEIGHT - 0.3, cz);
        light.castShadow = false;
        group.add(light);
        
        scene.add(group);
        
        group.children.forEach(obj => {
            if (obj.isMesh) room.addObject(obj);
        });
        
        rooms.push(room);
        roomMeshGroups.push(group);
    });
    
    // ============== Portal System ==============
    // เชื่อมห้องที่อยู่ติดกัน (openings ในกำแพง)
    // สร้าง portals ระหว่าง adjacent rooms
    
    // ============== Culling System ==============
    let cullingEnabled = true;
    
    function findCameraRoom() {
        const pos = camera.position;
        for (const room of rooms) {
            if (pos.x >= room.cx - ROOM_SIZE/2 && pos.x <= room.cx + ROOM_SIZE/2 &&
                pos.z >= room.cz - ROOM_SIZE/2 && pos.z <= room.cz + ROOM_SIZE/2) {
                return room;
            }
        }
        return null;
    }
    
    function applyFrustumCulling() {
        const frustum = new THREE.Frustum();
        frustum.setFromProjectionMatrix(
            new THREE.Matrix4().multiplyMatrices(
                camera.projectionMatrix, camera.matrixWorldInverse
            )
        );
        
        let visible = 0, hidden = 0;
        
        // Simplified: cull entire rooms based on distance + frustum
        const currentRoom = findCameraRoom();
        
        rooms.forEach((room, i) => {
            const roomCenter = new THREE.Vector3(room.cx, WALL_HEIGHT/2, room.cz);
            const dist = camera.position.distanceTo(roomCenter);
            
            // Show current room and adjacent rooms, cull far ones
            let roomVisible = true;
            if (cullingEnabled) {
                const sphere = new THREE.Sphere(roomCenter, ROOM_SIZE * 0.8);
                roomVisible = dist < ROOM_SIZE * 3 || frustum.intersectsSphere(sphere);
                
                // Always show current room
                if (currentRoom && room.id === currentRoom.id) roomVisible = true;
            }
            
            room.objects.forEach(obj => {
                if (obj.isMesh) {
                    obj.visible = roomVisible;
                    if (roomVisible) visible++;
                    else hidden++;
                }
            });
            
            if (roomVisible) {
                document.getElementById('m-vrooms').textContent;
            }
        });
        
        const visibleRooms = rooms.filter((r, i) => r.objects.some(o => o.visible)).length;
        
        return { visible, hidden, visibleRooms, currentRoom };
    }
    
    function toggleCulling() {
        cullingEnabled = !cullingEnabled;
        document.getElementById('btn-toggle').textContent = 
            cullingEnabled ? 'Culling: ON' : 'Culling: OFF';
        document.getElementById('m-culling').textContent = cullingEnabled ? 'เปิด ✅' : 'ปิด ❌';
    }
    
    document.getElementById('btn-toggle').addEventListener('click', toggleCulling);
    
    // Minimap
    const minimapCanvas = document.getElementById('minimap');
    const mmCtx = minimapCanvas.getContext('2d');
    
    function drawMinimap() {
        mmCtx.fillStyle = '#050510';
        mmCtx.fillRect(0, 0, 200, 200);
        
        const scale = 200 / ((ROOM_SIZE + 1) * 3 + 2);
        const offsetX = 100, offsetZ = 100;
        
        rooms.forEach((room, i) => {
            const rx = room.cx * scale + offsetX;
            const rz = room.cz * scale + offsetZ;
            const rs = ROOM_SIZE * scale;
            
            const anyVisible = room.objects.some(o => o.visible);
            mmCtx.fillStyle = anyVisible ? `#${roomColors[i].toString(16).padStart(6,'0')}44` : '#11111188';
            mmCtx.strokeStyle = anyVisible ? `#${roomColors[i].toString(16).padStart(6,'0')}` : '#333';
            mmCtx.lineWidth = 1;
            mmCtx.fillRect(rx - rs/2, rz - rs/2, rs, rs);
            mmCtx.strokeRect(rx - rs/2, rz - rs/2, rs, rs);
            
            mmCtx.fillStyle = '#fff';
            mmCtx.font = '10px monospace';
            mmCtx.textAlign = 'center';
            mmCtx.fillText(i, rx, rz + 4);
        });
        
        // Camera position
        const cx = camera.position.x * scale + offsetX;
        const cz = camera.position.z * scale + offsetZ;
        mmCtx.fillStyle = '#0af';
        mmCtx.beginPath();
        mmCtx.arc(cx, cz, 4, 0, Math.PI * 2);
        mmCtx.fill();
        
        // Camera direction
        const dir = new THREE.Vector3();
        camera.getWorldDirection(dir);
        mmCtx.strokeStyle = '#0af';
        mmCtx.lineWidth = 2;
        mmCtx.beginPath();
        mmCtx.moveTo(cx, cz);
        mmCtx.lineTo(cx + dir.x * 15, cz + dir.z * 15);
        mmCtx.stroke();
    }
    
    // Movement
    const velocity = new THREE.Vector3();
    const direction = new THREE.Vector3();
    const SPEED = 5;
    let prevTime = performance.now();
    
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
        
        const now = performance.now();
        const dt = Math.min((now - prevTime) / 1000, 0.05);
        prevTime = now;
        
        // Camera rotation from yaw/pitch
        camera.rotation.order = 'YXZ';
        camera.rotation.y = yaw;
        camera.rotation.x = pitch;
        
        // WASD movement
        if (moveForward || moveBackward || moveLeft || moveRight) {
            const moveDir = new THREE.Vector3();
            if (moveForward) moveDir.z -= 1;
            if (moveBackward) moveDir.z += 1;
            if (moveLeft) moveDir.x -= 1;
            if (moveRight) moveDir.x += 1;
            
            moveDir.normalize();
            moveDir.applyEuler(new THREE.Euler(0, yaw, 0));
            
            camera.position.addScaledVector(moveDir, SPEED * dt);
        }
        
        // Keep camera in bounds
        camera.position.y = 1.7;
        
        // Apply culling
        const result = applyFrustumCulling();
        
        renderer.render(scene, camera);
        
        // FPS
        frames++;
        if (now - lastTime >= 500) {
            fps = Math.round(frames / ((now - lastTime) / 1000));
            frames = 0; lastTime = now;
        }
        
        document.getElementById('m-calls').textContent = renderer.info.render.calls;
        document.getElementById('m-fps').textContent = fps;
        document.getElementById('m-vrooms').textContent = result.visibleRooms;
        document.getElementById('m-trooms').textContent = ROOM_COUNT;
        document.getElementById('m-hidden').textContent = result.hidden;
        document.getElementById('m-camroom').textContent = 
            result.currentRoom ? `Room ${result.currentRoom.id}` : 'Outside';
        
        drawMinimap();
    }
    
    animate();
</script>
</body>
</html>
```

---

## 6. Hi-Z Occlusion Culling (Concept)

```javascript
// Hi-Z (Hierarchical Z-buffer) culling
// - render depth buffer ในความละเอียดเต็ม
// - สร้าง mipmap ของ depth buffer (ค่าสูงสุดในแต่ละ tile)
// - test AABB ของ objects กับ Hi-Z mip levels

class HiZOcclusionCuller {
    constructor(renderer, width, height) {
        this.renderer = renderer;
        
        // Depth render target
        this.depthTarget = new THREE.WebGLRenderTarget(width, height, {
            format: THREE.RGBAFormat,
            type: THREE.UnsignedByteType,
            depthBuffer: true,
            stencilBuffer: false
        });
        
        // Hi-Z mip levels
        this.mipLevels = [];
        let w = width / 2, h = height / 2;
        while (w >= 1 && h >= 1) {
            this.mipLevels.push(new THREE.WebGLRenderTarget(w, h));
            w = Math.floor(w / 2);
            h = Math.floor(h / 2);
        }
    }
    
    // Step 1: Render depth prepass
    renderDepthPrepass(scene, camera) {
        const prevMat = new Map();
        
        // Replace all materials with depth material
        scene.traverse(obj => {
            if (obj.isMesh) {
                prevMat.set(obj, obj.material);
                obj.material = new THREE.MeshDepthMaterial();
            }
        });
        
        this.renderer.setRenderTarget(this.depthTarget);
        this.renderer.render(scene, camera);
        this.renderer.setRenderTarget(null);
        
        // Restore materials
        scene.traverse(obj => {
            if (obj.isMesh && prevMat.has(obj)) {
                obj.material = prevMat.get(obj);
            }
        });
    }
    
    // Step 2: Build Hi-Z mipmap (simplified - would need custom shader)
    buildHiZMipmaps() {
        // ใน production: ใช้ custom compute shader หรือ render pass
        // เพื่อ build mip hierarchy ของ depth buffer
        // แต่ละ mip = maximum depth ของ pixels ใน 2x2 block
        console.log('Hi-Z mipmaps built');
    }
    
    // Step 3: Test object against Hi-Z
    testVisibility(object, camera) {
        // Project AABB ไปที่ screen space
        // Sample Hi-Z ที่ appropriate mip level
        // ถ้า max depth ของ AABB < depth ใน Hi-Z = occluded
        
        // Simplified version:
        object.geometry.computeBoundingBox();
        const box = object.geometry.boundingBox.clone();
        box.applyMatrix4(object.matrixWorld);
        
        // Convert to screen space
        const frustum = new THREE.Frustum();
        frustum.setFromProjectionMatrix(
            new THREE.Matrix4().multiplyMatrices(
                camera.projectionMatrix, camera.matrixWorldInverse
            )
        );
        
        return frustum.intersectsBox(box);
    }
}
```

---

## 7. Front-to-Back Rendering (Paint Order Optimization)

```javascript
// วาด opaque objects front-to-back เพื่อ early depth rejection
function sortFrontToBack(scene, camera) {
    const cameraPos = camera.position;
    const objects = [];
    
    scene.traverse(obj => {
        if (obj.isMesh && !obj.material.transparent) {
            const dist = obj.position.distanceTo(cameraPos);
            objects.push({ obj, dist });
        }
    });
    
    // Sort by distance (ascending = front to back)
    objects.sort((a, b) => a.dist - b.dist);
    
    // Assign render order
    objects.forEach(({ obj }, i) => {
        obj.renderOrder = i;
    });
}

// เรียกทุก frame หรือเมื่อ camera/objects เปลี่ยน
function animate() {
    requestAnimationFrame(animate);
    sortFrontToBack(scene, camera);
    renderer.render(scene, camera);
}
```

---

## แบบฝึกหัด

1. **ง่าย**: สร้าง grid-based culler ที่แบ่ง scene เป็น 10x10 cells และซ่อน cells ที่อยู่นอก frustum + ระยะ render distance
2. **กลาง**: สร้าง indoor scene 4 ห้อง ด้วย portal culling อย่างง่าย (แสดงเฉพาะห้องปัจจุบันและห้องที่มี portal เปิดอยู่)
3. **ยาก**: Implement depth-based occlusion culling บน CPU โดย render depth buffer แบบ low-resolution แล้ว test objects ต่อมัน
4. **Challenge**: สร้าง PVS system ที่ precompute visibility ระหว่าง grid cells ด้วย raycast sampling

---

## อ้างอิง

- [three-mesh-bvh library](https://github.com/gkjohnson/three-mesh-bvh)
- [Portal Rendering Overview](https://en.wikipedia.org/wiki/Portal_rendering)
- [Hierarchical Z-Buffer Culling](https://www.gameenginebook.com/)
- [Potentially Visible Sets (PVS)](https://developer.valvesoftware.com/wiki/Potentially_Visible_Set)
