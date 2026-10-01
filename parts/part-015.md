# Part 15: Raycasting & Mouse Interaction ใน Three.js

## บทนำ

Raycasting คือเทคนิคสำคัญในการสร้าง interaction ใน 3D โดยการยิง "ray" (ลำแสง) จากกล้องผ่านตำแหน่ง mouse ไปยัง objects ใน scene ใช้สำหรับ:
- การเลือก (pick/select) วัตถุ
- Hover effects
- Drag and drop ใน 3D
- การยิงในเกม FPS
- การตรวจสอบการชน

## 1. Raycaster Class

```javascript
// สร้าง Raycaster
const raycaster = new THREE.Raycaster();

// กำหนด ray จากกล้องและ mouse
// mouse.x, mouse.y ต้องเป็น Normalized Device Coordinates (-1 ถึง 1)
const mouse = new THREE.Vector2();

// แปลง mouse position เป็น NDC
window.addEventListener('mousemove', (event) => {
    mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
    // Y กลับเพราะ THREE.js ใช้ Y+ ขึ้น แต่ DOM ใช้ Y+ ลง
});

// กำหนด ray จากกล้อง (ใช้ใน render loop)
raycaster.setFromCamera(mouse, camera);

// ตรวจสอบการชนกับ objects
const intersects = raycaster.intersectObjects(objects);
// หรือ
const intersect = raycaster.intersectObject(singleObject);
// recursive = true จะตรวจ children ด้วย
const intersectsRecursive = raycaster.intersectObjects(objects, true);
```

### Intersection Object

```javascript
// intersects เป็น array ของ intersection objects เรียงจากใกล้ไปไกล
if (intersects.length > 0) {
    const hit = intersects[0]; // ใกล้กล้องที่สุด
    
    console.log(hit.distance);   // ระยะจากกล้องถึงจุดชน (float)
    console.log(hit.point);      // Vector3 - จุดที่ชนใน world space
    console.log(hit.face);       // Face - face ที่ชน (สำหรับ Mesh)
    console.log(hit.faceIndex);  // number - index ของ face
    console.log(hit.object);     // Object3D ที่ถูกชน
    console.log(hit.uv);         // Vector2 - UV coordinate ที่จุดชน
    console.log(hit.normal);     // Vector3 - normal ที่จุดชน
    console.log(hit.instanceId); // number - สำหรับ InstancedMesh
}
```

---

## 2. Mouse Coordinates Normalization

```javascript
// สำหรับ Canvas เต็มหน้าจอ
function getMouseNDC(event) {
    return {
        x: (event.clientX / window.innerWidth) * 2 - 1,
        y: -(event.clientY / window.innerHeight) * 2 + 1
    };
}

// สำหรับ Canvas ที่ไม่ได้เต็มหน้าจอ (ต้องใช้ getBoundingClientRect)
function getMouseNDCForCanvas(event, canvas) {
    const rect = canvas.getBoundingClientRect();
    return {
        x: ((event.clientX - rect.left) / rect.width) * 2 - 1,
        y: -((event.clientY - rect.top) / rect.height) * 2 + 1
    };
}

// Touch support
function getTouchNDC(touch) {
    return {
        x: (touch.clientX / window.innerWidth) * 2 - 1,
        y: -(touch.clientY / window.innerHeight) * 2 + 1
    };
}

// ตัวอย่างการใช้งาน
const mouse = new THREE.Vector2();

window.addEventListener('mousemove', (e) => {
    const ndc = getMouseNDC(e);
    mouse.x = ndc.x;
    mouse.y = ndc.y;
});

window.addEventListener('touchmove', (e) => {
    e.preventDefault();
    const ndc = getTouchNDC(e.touches[0]);
    mouse.x = ndc.x;
    mouse.y = ndc.y;
}, { passive: false });
```

---

## 3. Interactive Object Picker Demo

```html
<!DOCTYPE html>
<html>
<head>
    <title>Object Picker</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #1a1a2e; overflow: hidden; font-family: monospace; }
        
        #tooltip {
            position: absolute;
            background: rgba(0,0,0,0.8);
            color: white;
            padding: 8px 12px;
            border-radius: 4px;
            font-size: 13px;
            pointer-events: none;
            display: none;
            border: 1px solid #e94560;
            z-index: 100;
        }
        
        #selected-info {
            position: absolute;
            bottom: 20px;
            left: 20px;
            background: rgba(0,0,0,0.8);
            color: white;
            padding: 15px;
            border-radius: 8px;
            font-size: 13px;
            min-width: 200px;
        }
        
        .label { color: #4ecdc4; }
        .value { color: #ffd700; }
    </style>
</head>
<body>

<div id="tooltip"></div>
<div id="selected-info">
    <div class="label">Selected: </div>
    <div class="value" id="sel-name">ยังไม่ได้เลือก</div>
    <div class="label">Position: </div>
    <div class="value" id="sel-pos">-</div>
    <div class="label">Hit Point: </div>
    <div class="value" id="hit-point">-</div>
    <div class="label">Distance: </div>
    <div class="value" id="hit-dist">-</div>
</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';

// Scene Setup
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x1a1a2e);

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(0, 8, 15);
camera.lookAt(0, 0, 0);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
renderer.shadowMap.enabled = true;
document.body.appendChild(renderer.domElement);

const orbitControls = new OrbitControls(camera, renderer.domElement);
orbitControls.enableDamping = true;

// Lighting
scene.add(new THREE.AmbientLight(0xffffff, 0.4));
const dirLight = new THREE.DirectionalLight(0xffffff, 1);
dirLight.position.set(5, 10, 5);
dirLight.castShadow = true;
scene.add(dirLight);

// Grid
scene.add(new THREE.GridHelper(30, 30, 0x333355, 0x222244));

// สร้างวัตถุที่ต้องการ interact
const objectsData = [
    { name: 'Red Cube', color: 0xe94560, geo: new THREE.BoxGeometry(2, 2, 2), pos: { x: -5, y: 1, z: 0 } },
    { name: 'Blue Sphere', color: 0x4ecdc4, geo: new THREE.SphereGeometry(1.2, 32, 32), pos: { x: 0, y: 1.2, z: 0 } },
    { name: 'Gold Cone', color: 0xffd700, geo: new THREE.ConeGeometry(1, 2, 32), pos: { x: 5, y: 1, z: 0 } },
    { name: 'Purple Torus', color: 0x9b59b6, geo: new THREE.TorusGeometry(1, 0.4, 16, 100), pos: { x: -5, y: 1, z: -5 } },
    { name: 'Orange Cylinder', color: 0xe67e22, geo: new THREE.CylinderGeometry(0.8, 0.8, 2, 32), pos: { x: 0, y: 1, z: -5 } },
    { name: 'Green Diamond', color: 0x2ecc71, geo: new THREE.OctahedronGeometry(1.2), pos: { x: 5, y: 1.2, z: -5 } },
];

const pickableObjects = [];
const originalMaterials = new Map();
const originalScales = new Map();

objectsData.forEach(data => {
    const mat = new THREE.MeshStandardMaterial({
        color: data.color,
        roughness: 0.3,
        metalness: 0.5
    });
    const mesh = new THREE.Mesh(data.geo, mat);
    mesh.name = data.name;
    mesh.position.set(data.pos.x, data.pos.y, data.pos.z);
    mesh.castShadow = true;
    mesh.receiveShadow = true;
    scene.add(mesh);
    pickableObjects.push(mesh);
    originalMaterials.set(mesh.uuid, mat.clone());
    originalScales.set(mesh.uuid, mesh.scale.clone());
});

// พื้น
const floor = new THREE.Mesh(
    new THREE.PlaneGeometry(30, 30),
    new THREE.MeshStandardMaterial({ color: 0x16213e, roughness: 0.8 })
);
floor.rotation.x = -Math.PI / 2;
floor.receiveShadow = true;
scene.add(floor);

// Raycaster
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();

let hoveredObject = null;
let selectedObject = null;

// Tooltip
const tooltip = document.getElementById('tooltip');
const selName = document.getElementById('sel-name');
const selPos = document.getElementById('sel-pos');
const hitPointEl = document.getElementById('hit-point');
const hitDistEl = document.getElementById('hit-dist');

// Hover effect
function onMouseMove(event) {
    mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
    
    // อัพเดต tooltip position
    tooltip.style.left = (event.clientX + 15) + 'px';
    tooltip.style.top = (event.clientY - 10) + 'px';
    
    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(pickableObjects);
    
    if (intersects.length > 0) {
        const hit = intersects[0];
        const obj = hit.object;
        
        if (hoveredObject !== obj) {
            // Unhover previous
            if (hoveredObject && hoveredObject !== selectedObject) {
                resetHover(hoveredObject);
            }
            
            // Hover new
            if (obj !== selectedObject) {
                hoveredObject = obj;
                applyHover(obj);
            }
        }
        
        // แสดง tooltip
        tooltip.style.display = 'block';
        tooltip.innerHTML = `
            <b>${obj.name}</b><br>
            Distance: ${hit.distance.toFixed(2)}<br>
            UV: (${hit.uv ? hit.uv.x.toFixed(2) : 'N/A'}, ${hit.uv ? hit.uv.y.toFixed(2) : 'N/A'})
        `;
        
        document.body.style.cursor = 'pointer';
    } else {
        if (hoveredObject && hoveredObject !== selectedObject) {
            resetHover(hoveredObject);
            hoveredObject = null;
        }
        tooltip.style.display = 'none';
        document.body.style.cursor = 'default';
    }
}

function applyHover(obj) {
    obj.material.emissive.setHex(0x333333);
    // Scale up slightly
    obj.scale.multiplyScalar(1.1);
}

function resetHover(obj) {
    obj.material.emissive.setHex(0x000000);
    const origScale = originalScales.get(obj.uuid);
    if (origScale) obj.scale.copy(origScale);
}

// Click to select
function onClick(event) {
    mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
    
    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(pickableObjects);
    
    if (intersects.length > 0) {
        const hit = intersects[0];
        const obj = hit.object;
        
        // Deselect previous
        if (selectedObject && selectedObject !== obj) {
            resetHover(selectedObject);
            selectedObject.material.wireframe = false;
        }
        
        // Select new
        selectedObject = obj;
        obj.material.emissive.setHex(0x555555);
        obj.material.wireframe = false;
        
        // อัพเดต info panel
        selName.textContent = obj.name;
        selPos.textContent = `(${obj.position.x.toFixed(1)}, ${obj.position.y.toFixed(1)}, ${obj.position.z.toFixed(1)})`;
        hitPointEl.textContent = `(${hit.point.x.toFixed(2)}, ${hit.point.y.toFixed(2)}, ${hit.point.z.toFixed(2)})`;
        hitDistEl.textContent = hit.distance.toFixed(3);
        
        // Visual feedback - pulse animation
        const origScale = originalScales.get(obj.uuid);
        if (origScale) {
            obj.scale.copy(origScale);
            const targetScale = origScale.clone().multiplyScalar(1.2);
            
            // Simple pulse using TWEEN-like approach
            let t = 0;
            const pulse = setInterval(() => {
                t += 0.1;
                const s = 1 + Math.sin(t * Math.PI) * 0.2;
                if (obj === selectedObject) {
                    obj.scale.setScalar(s);
                }
                if (t >= 1) {
                    clearInterval(pulse);
                    obj.scale.copy(origScale);
                }
            }, 16);
        }
        
    } else {
        // Deselect
        if (selectedObject) {
            resetHover(selectedObject);
            selectedObject = null;
            selName.textContent = 'ยังไม่ได้เลือก';
            selPos.textContent = '-';
            hitPointEl.textContent = '-';
            hitDistEl.textContent = '-';
        }
    }
}

renderer.domElement.addEventListener('mousemove', onMouseMove);
renderer.domElement.addEventListener('click', onClick);

// Touch support
renderer.domElement.addEventListener('touchstart', (e) => {
    e.preventDefault();
    const touch = e.touches[0];
    const clickEvent = { clientX: touch.clientX, clientY: touch.clientY };
    onClick(clickEvent);
}, { passive: false });

// Resize
window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});

// Animation
const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    
    // หมุนวัตถุที่ hover
    if (hoveredObject) {
        hoveredObject.rotation.y += 0.02;
    }
    
    orbitControls.update();
    renderer.render(scene, camera);
}
animate();
</script>
</body>
</html>
```

---

## 4. Drag and Drop 3D Objects

```javascript
// Drag and Drop ใน 3D
class DragController {
    constructor(camera, scene, renderer) {
        this.camera = camera;
        this.scene = scene;
        this.renderer = renderer;
        
        this.raycaster = new THREE.Raycaster();
        this.mouse = new THREE.Vector2();
        this.plane = new THREE.Plane();  // plane สำหรับ drag บน
        this.planeNormal = new THREE.Vector3(0, 1, 0); // Y-up plane
        this.planeIntersect = new THREE.Vector3();
        
        this.dragObject = null;
        this.offset = new THREE.Vector3();
        this.draggableObjects = [];
        
        this.bindEvents();
    }
    
    addObject(obj) {
        this.draggableObjects.push(obj);
    }
    
    bindEvents() {
        const canvas = this.renderer.domElement;
        canvas.addEventListener('mousedown', this.onMouseDown.bind(this));
        canvas.addEventListener('mousemove', this.onMouseMove.bind(this));
        canvas.addEventListener('mouseup', this.onMouseUp.bind(this));
    }
    
    getMouseNDC(event) {
        this.mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
        this.mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
    }
    
    onMouseDown(event) {
        this.getMouseNDC(event);
        this.raycaster.setFromCamera(this.mouse, this.camera);
        
        const intersects = this.raycaster.intersectObjects(this.draggableObjects);
        
        if (intersects.length > 0) {
            this.dragObject = intersects[0].object;
            
            // ตั้ง drag plane ที่ความสูงของ object
            this.plane.setFromNormalAndCoplanarPoint(
                this.planeNormal,
                this.dragObject.position
            );
            
            // คำนวณ offset
            if (this.raycaster.ray.intersectPlane(this.plane, this.planeIntersect)) {
                this.offset.copy(this.planeIntersect).sub(this.dragObject.position);
            }
            
            document.body.style.cursor = 'grabbing';
        }
    }
    
    onMouseMove(event) {
        if (!this.dragObject) return;
        
        this.getMouseNDC(event);
        this.raycaster.setFromCamera(this.mouse, this.camera);
        
        if (this.raycaster.ray.intersectPlane(this.plane, this.planeIntersect)) {
            this.dragObject.position.copy(
                this.planeIntersect.sub(this.offset)
            );
        }
    }
    
    onMouseUp() {
        if (this.dragObject) {
            document.body.style.cursor = 'default';
            this.dragObject = null;
        }
    }
    
    dispose() {
        const canvas = this.renderer.domElement;
        canvas.removeEventListener('mousedown', this.onMouseDown.bind(this));
        canvas.removeEventListener('mousemove', this.onMouseMove.bind(this));
        canvas.removeEventListener('mouseup', this.onMouseUp.bind(this));
    }
}

// ใช้งาน
const dragController = new DragController(camera, scene, renderer);
dragController.addObject(cube);
dragController.addObject(sphere);
```

---

## 5. Advanced Raycasting Techniques

### Raycasting กับ InstancedMesh

```javascript
const instancedMesh = new THREE.InstancedMesh(geo, mat, 100);

// ตั้งค่า instance matrices...
instancedMesh.instanceMatrix.needsUpdate = true;

// Raycast ทำงานกับ InstancedMesh ได้ตรงๆ
raycaster.setFromCamera(mouse, camera);
const intersects = raycaster.intersectObject(instancedMesh);

if (intersects.length > 0) {
    const instanceId = intersects[0].instanceId;
    console.log('Instance ID:', instanceId);
    
    // ดึง matrix ของ instance ที่ถูกชน
    const matrix = new THREE.Matrix4();
    instancedMesh.getMatrixAt(instanceId, matrix);
    
    // เปลี่ยนสี instance (ต้องใช้ setColorAt)
    instancedMesh.setColorAt(instanceId, new THREE.Color(0xff0000));
    instancedMesh.instanceColor.needsUpdate = true;
}
```

### Raycasting ในแนวตั้ง (สำหรับ ground detection)

```javascript
// ตรวจจับพื้นดิน - ยิง ray ลงจากตำแหน่งวัตถุ
function checkGround(object, groundMesh) {
    const position = object.position.clone();
    position.y += 1; // เริ่มจากสูงขึ้นเล็กน้อย
    
    const rayDirection = new THREE.Vector3(0, -1, 0); // ลงแนวตั้ง
    
    const raycaster = new THREE.Raycaster(position, rayDirection);
    const intersects = raycaster.intersectObject(groundMesh);
    
    if (intersects.length > 0) {
        return intersects[0].point.y; // ความสูงของพื้น
    }
    return null;
}

// ใช้ใน update loop
function updatePhysics(delta) {
    const groundY = checkGround(player, ground);
    if (groundY !== null && player.position.y <= groundY + 0.5) {
        player.position.y = groundY + 0.5;
        velocity.y = 0;
        canJump = true;
    }
}
```

### Raycasting สำหรับ Wall Detection

```javascript
// ตรวจจับกำแพงในทิศทางการเดิน
function checkWall(position, direction, walls) {
    const raycaster = new THREE.Raycaster(
        position.clone(),
        direction.clone().normalize()
    );
    raycaster.far = 1.0; // ระยะตรวจสอบ
    
    const intersects = raycaster.intersectObjects(walls, true);
    return intersects.length > 0;
}
```

---

## 6. Hover Highlight Effect

```javascript
// Outline effect ด้วยการ scale up
class HoverHighlight {
    constructor(scene) {
        this.scene = scene;
        this.outlineObjects = new Map();
    }
    
    addOutline(mesh) {
        // สร้าง outline mesh (scale up เล็กน้อย + render backface)
        const outlineGeo = mesh.geometry.clone();
        const outlineMat = new THREE.MeshBasicMaterial({
            color: 0xffffff,
            side: THREE.BackSide // render เฉพาะด้านหลัง
        });
        const outline = new THREE.Mesh(outlineGeo, outlineMat);
        outline.scale.multiplyScalar(1.1);
        outline.visible = false;
        
        mesh.add(outline); // เป็น child ของ mesh
        this.outlineObjects.set(mesh.uuid, outline);
    }
    
    highlight(mesh) {
        const outline = this.outlineObjects.get(mesh.uuid);
        if (outline) outline.visible = true;
    }
    
    unhighlight(mesh) {
        const outline = this.outlineObjects.get(mesh.uuid);
        if (outline) outline.visible = false;
    }
}

// ใช้งาน
const hoverHighlight = new HoverHighlight(scene);
pickableObjects.forEach(obj => hoverHighlight.addOutline(obj));

function onMouseMove(event) {
    // ...raycast...
    if (intersects.length > 0) {
        const obj = intersects[0].object;
        if (lastHovered !== obj) {
            if (lastHovered) hoverHighlight.unhighlight(lastHovered);
            hoverHighlight.highlight(obj);
            lastHovered = obj;
        }
    } else {
        if (lastHovered) {
            hoverHighlight.unhighlight(lastHovered);
            lastHovered = null;
        }
    }
}
```

---

## 7. Touch Support สมบูรณ์

```javascript
class InputHandler {
    constructor(renderer, camera) {
        this.renderer = renderer;
        this.camera = camera;
        this.pointer = new THREE.Vector2();
        this.raycaster = new THREE.Raycaster();
        
        this.onPointerMove = this.onPointerMove.bind(this);
        this.onPointerDown = this.onPointerDown.bind(this);
        this.onPointerUp = this.onPointerUp.bind(this);
        
        const canvas = renderer.domElement;
        canvas.addEventListener('pointermove', this.onPointerMove);
        canvas.addEventListener('pointerdown', this.onPointerDown);
        canvas.addEventListener('pointerup', this.onPointerUp);
    }
    
    updatePointer(event) {
        // Pointer Events รองรับทั้ง mouse, touch, stylus
        const rect = this.renderer.domElement.getBoundingClientRect();
        this.pointer.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
        this.pointer.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
    }
    
    onPointerMove(event) {
        this.updatePointer(event);
        this.raycaster.setFromCamera(this.pointer, this.camera);
        
        // Dispatch custom event
        this.renderer.domElement.dispatchEvent(new CustomEvent('raycast-move', {
            detail: {
                raycaster: this.raycaster,
                pointer: this.pointer.clone(),
                originalEvent: event
            }
        }));
    }
    
    onPointerDown(event) {
        this.updatePointer(event);
        this.raycaster.setFromCamera(this.pointer, this.camera);
        
        this.renderer.domElement.dispatchEvent(new CustomEvent('raycast-down', {
            detail: { raycaster: this.raycaster, pointer: this.pointer.clone() }
        }));
    }
    
    onPointerUp(event) {
        this.renderer.domElement.dispatchEvent(new CustomEvent('raycast-up', {
            detail: { originalEvent: event }
        }));
    }
    
    dispose() {
        const canvas = this.renderer.domElement;
        canvas.removeEventListener('pointermove', this.onPointerMove);
        canvas.removeEventListener('pointerdown', this.onPointerDown);
        canvas.removeEventListener('pointerup', this.onPointerUp);
    }
}

// ใช้งาน
const inputHandler = new InputHandler(renderer, camera);

renderer.domElement.addEventListener('raycast-move', (event) => {
    const { raycaster } = event.detail;
    const intersects = raycaster.intersectObjects(pickableObjects);
    // handle hover...
});

renderer.domElement.addEventListener('raycast-down', (event) => {
    const { raycaster } = event.detail;
    const intersects = raycaster.intersectObjects(pickableObjects);
    // handle click/tap...
});
```

---

## 8. Context Menu ใน 3D

```javascript
// Right-click context menu สำหรับ 3D objects
let contextMenuObject = null;

renderer.domElement.addEventListener('contextmenu', (event) => {
    event.preventDefault();
    
    const mouse = {
        x: (event.clientX / window.innerWidth) * 2 - 1,
        y: -(event.clientY / window.innerHeight) * 2 + 1
    };
    
    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(pickableObjects);
    
    if (intersects.length > 0) {
        contextMenuObject = intersects[0].object;
        showContextMenu(event.clientX, event.clientY, contextMenuObject);
    } else {
        hideContextMenu();
    }
});

function showContextMenu(x, y, object) {
    const menu = document.getElementById('context-menu');
    menu.style.left = x + 'px';
    menu.style.top = y + 'px';
    menu.style.display = 'block';
    
    // ตั้งค่าเมนูตาม object type
    document.getElementById('ctx-name').textContent = object.name;
    document.getElementById('ctx-delete').onclick = () => {
        scene.remove(contextMenuObject);
        hideContextMenu();
    };
    document.getElementById('ctx-color').onclick = () => {
        contextMenuObject.material.color.setHex(Math.random() * 0xffffff);
        hideContextMenu();
    };
}

function hideContextMenu() {
    document.getElementById('context-menu').style.display = 'none';
    contextMenuObject = null;
}

// ปิด context menu เมื่อคลิกที่อื่น
document.addEventListener('click', hideContextMenu);
```

---

## 9. Raycaster Properties

```javascript
const raycaster = new THREE.Raycaster();

// กำหนด near/far ของ ray
raycaster.near = 0;      // ค่าเริ่มต้น 0
raycaster.far = Infinity; // ค่าเริ่มต้น Infinity
// จำกัดระยะ:
raycaster.far = 10;       // ตรวจเฉพาะในระยะ 10 หน่วย

// Threshold สำหรับ Points และ Line
raycaster.params.Points.threshold = 0.1;  // ระยะตรวจสำหรับ Points
raycaster.params.Line.threshold = 0.1;    // ระยะตรวจสำหรับ Line

// setFromCamera - ใช้กับ Perspective/Orthographic Camera
raycaster.setFromCamera(mouse, perspectiveCamera);
raycaster.setFromCamera(mouse, orthographicCamera);

// set - กำหนด ray โดยตรง
const origin = new THREE.Vector3(0, 10, 0);
const direction = new THREE.Vector3(0, -1, 0).normalize();
raycaster.set(origin, direction);
```

---

## 10. Performance Tips

```javascript
// ✅ จำกัด objects ที่ตรวจสอบ
// แทนที่จะ intersect ทุก object ใน scene
// ✅ เก็บ list ของ pickable objects แยกต่างหาก
const pickableObjects = [];
// เพิ่มเฉพาะที่ต้องการ pick
pickableObjects.push(interactiveObject);

// ✅ ใช้ Layers เพื่อ filter
// กำหนด objects ที่ pickable ไว้ใน layer เดียวกัน
raycaster.layers.set(1); // ตรวจเฉพาะ layer 1
object.layers.set(1);

// ✅ ลด frequency ของ raycast
let lastRaycastTime = 0;
const RAYCAST_INTERVAL = 50; // ms

function onMouseMove(event) {
    const now = Date.now();
    if (now - lastRaycastTime < RAYCAST_INTERVAL) return;
    lastRaycastTime = now;
    
    // do raycast...
}

// ✅ ใช้ BVH (Bounding Volume Hierarchy) สำหรับ complex geometries
// npm install three-mesh-bvh
import { computeBoundsTree, disposeBoundsTree, acceleratedRaycast } from 'three-mesh-bvh';
THREE.BufferGeometry.prototype.computeBoundsTree = computeBoundsTree;
THREE.BufferGeometry.prototype.disposeBoundsTree = disposeBoundsTree;
THREE.Mesh.prototype.raycast = acceleratedRaycast;

// แล้วเรียก:
mesh.geometry.computeBoundsTree(); // เร็วขึ้นมากสำหรับ complex mesh
```

---

## แบบฝึกหัด

1. **Color Picker**: คลิก object แล้วเปิด color picker เปลี่ยนสีได้
2. **3D Inspector**: แสดงข้อมูล geometry/material เมื่อ hover
3. **Multi-Select**: กด Shift + คลิก เพื่อเลือกหลาย objects พร้อมกัน
4. **Snap to Grid**: ระหว่าง drag ให้ snap ไปยัง grid เป็น 0.5 หน่วย
5. **Measurement Tool**: คลิกสองจุด แล้ว display ระยะห่าง

---

## สรุป

- `Raycaster.setFromCamera(mouse, camera)` คือวิธีมาตรฐาน
- Mouse coordinates ต้องแปลงเป็น NDC (-1 ถึง 1) ก่อน
- `intersects[0]` คือ object ที่ใกล้กล้องที่สุด
- Intersection มีข้อมูล: `point`, `face`, `distance`, `uv`, `object`
- ใช้ Pointer Events แทน Mouse/Touch Events สำหรับ cross-platform
- จำกัด objects ที่ตรวจสอบเพื่อ performance
- `instanceId` ใช้สำหรับ InstancedMesh
