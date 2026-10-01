# Part 57: Level of Detail (LOD)

## แนวคิด Level of Detail

LOD (Level of Detail) เป็นเทคนิคที่ใช้ mesh ที่มีรายละเอียดต่างกันตามระยะห่างจากกล้อง วัตถุที่อยู่ใกล้ใช้ mesh polygon สูง วัตถุที่ไกลออกไปใช้ mesh polygon ต่ำ ลด draw cost และ GPU load ได้มาก

```
camera
  │
  ├── 0-10m  → LOD 0 (10,000 polygons)
  ├── 10-50m → LOD 1 (2,500 polygons)
  ├── 50-100m→ LOD 2 (500 polygons)
  └── 100m+  → LOD 3 (billboard sprite)
```

## THREE.LOD Class

```javascript
import * as THREE from 'three';

// สร้าง LOD object
const lod = new THREE.LOD();

// สร้าง meshes หลาย level
const highDetailGeo = new THREE.IcosahedronGeometry(1, 4);   // ~1280 faces
const medDetailGeo = new THREE.IcosahedronGeometry(1, 2);    // ~320 faces
const lowDetailGeo = new THREE.IcosahedronGeometry(1, 1);    // ~80 faces
const veryLowGeo = new THREE.IcosahedronGeometry(1, 0);      // ~20 faces

const material = new THREE.MeshStandardMaterial({ color: 0x4488cc });

const highMesh = new THREE.Mesh(highDetailGeo, material);
const medMesh = new THREE.Mesh(medDetailGeo, material);
const lowMesh = new THREE.Mesh(lowDetailGeo, material);
const veryLowMesh = new THREE.Mesh(veryLowGeo, material);

// addLevel(object, distance)
// distance = ระยะที่เริ่มใช้ level นี้
lod.addLevel(highMesh, 0);       // ใกล้สุด
lod.addLevel(medMesh, 10);
lod.addLevel(lowMesh, 30);
lod.addLevel(veryLowMesh, 60);   // ไกลสุด

lod.position.set(0, 0, 0);
scene.add(lod);

// Update LOD ใน render loop
function animate() {
    requestAnimationFrame(animate);
    
    // LOD.update() รับ camera และเลือก level ที่เหมาะสม
    lod.update(camera);
    
    renderer.render(scene, camera);
}
```

## LOD สำหรับ GLTF Models

```javascript
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

const loader = new GLTFLoader();

// โหลด model หลาย LOD versions
async function loadModelWithLOD(basePath) {
    const lod = new THREE.LOD();
    
    const configs = [
        { file: `${basePath}_lod0.glb`, distance: 0 },
        { file: `${basePath}_lod1.glb`, distance: 15 },
        { file: `${basePath}_lod2.glb`, distance: 50 },
    ];
    
    const promises = configs.map(cfg => 
        new Promise((resolve) => {
            loader.load(cfg.file, (gltf) => {
                resolve({ scene: gltf.scene, distance: cfg.distance });
            });
        })
    );
    
    const results = await Promise.all(promises);
    
    results.forEach(({ scene, distance }) => {
        lod.addLevel(scene, distance);
    });
    
    return lod;
}
```

## Manual LOD System

สำหรับ performance สูงสุด บางครั้งควรทำ LOD เอง:

```javascript
class LODManager {
    constructor(camera) {
        this.camera = camera;
        this.objects = [];
        this.updateInterval = 10; // update ทุก N frames
        this.frameCount = 0;
        this._cameraPos = new THREE.Vector3();
    }
    
    register(position, levels) {
        // levels = [{ mesh, maxDistance }]
        const entry = {
            position,
            levels,
            currentLevel: -1,
            group: new THREE.Group()
        };
        
        levels.forEach(l => entry.group.add(l.mesh));
        entry.group.position.copy(position);
        this.objects.push(entry);
        return entry.group;
    }
    
    update() {
        this.frameCount++;
        if (this.frameCount % this.updateInterval !== 0) return;
        
        this._cameraPos.copy(this.camera.position);
        
        for (const entry of this.objects) {
            const dist = this._cameraPos.distanceTo(entry.position);
            
            let targetLevel = entry.levels.length - 1;
            for (let i = 0; i < entry.levels.length; i++) {
                if (dist < entry.levels[i].maxDistance) {
                    targetLevel = i;
                    break;
                }
            }
            
            if (targetLevel !== entry.currentLevel) {
                entry.currentLevel = targetLevel;
                entry.levels.forEach((l, i) => {
                    l.mesh.visible = (i === targetLevel);
                });
            }
        }
    }
}
```

## Billboarding

Billboard คือ sprite แบน 2D ที่หันหน้าเข้าหากล้องเสมอ ใช้แทน 3D mesh ในระยะไกล:

```javascript
// Billboard Shader
class BillboardManager {
    createBillboardMaterial(texture) {
        return new THREE.ShaderMaterial({
            uniforms: {
                map: { value: texture },
                opacity: { value: 1.0 }
            },
            vertexShader: `
                // Billboard: ยกเลิก rotation ของ model matrix
                void main() {
                    // Extract position จาก model matrix (column 3)
                    vec3 worldPos = vec3(modelMatrix[3][0], modelMatrix[3][1], modelMatrix[3][2]);
                    
                    // ใช้แค่ camera view projection โดยไม่หมุนตาม model
                    vec4 viewPos = viewMatrix * vec4(worldPos, 1.0);
                    viewPos.xy += position.xy * 0.5; // scale billboard
                    
                    gl_Position = projectionMatrix * viewPos;
                }
            `,
            fragmentShader: `
                uniform sampler2D map;
                uniform float opacity;
                varying vec2 vUv;
                void main() {
                    vec4 color = texture2D(map, vUv);
                    if (color.a < 0.1) discard;
                    gl_FragColor = vec4(color.rgb, color.a * opacity);
                }
            `,
            transparent: true,
            side: THREE.DoubleSide,
            depthWrite: false
        });
    }
    
    createBillboard(texture, width = 1, height = 1) {
        const geo = new THREE.PlaneGeometry(width, height);
        const mat = this.createBillboardMaterial(texture);
        return new THREE.Mesh(geo, mat);
    }
}
```

## ตัวอย่าง: LOD Performance Demo (1000 Objects)

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>LOD Performance Demo</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; font-family: monospace; }
        canvas { display: block; }
        #hud {
            position: fixed;
            top: 20px;
            left: 20px;
            background: rgba(0,0,0,0.75);
            color: #00ff88;
            padding: 16px;
            border-radius: 8px;
            font-size: 13px;
            border: 1px solid #00ff88;
            min-width: 240px;
        }
        #hud h3 { color: #88ffcc; margin-bottom: 10px; }
        .stat { display: flex; justify-content: space-between; margin: 3px 0; }
        .stat span:last-child { color: #ffcc44; }
        #controls {
            position: fixed;
            top: 20px;
            right: 20px;
            background: rgba(0,0,0,0.75);
            color: #ccc;
            padding: 16px;
            border-radius: 8px;
            font-size: 12px;
            border: 1px solid #555;
        }
        .toggle-btn {
            display: block;
            width: 100%;
            padding: 6px 10px;
            margin: 4px 0;
            background: transparent;
            color: #00ff88;
            border: 1px solid #00ff88;
            cursor: pointer;
            border-radius: 4px;
            font-size: 12px;
        }
        .toggle-btn.off { color: #ff4444; border-color: #ff4444; }
        #lodColors {
            margin-top: 10px;
            font-size: 11px;
        }
        .lc { display: flex; align-items: center; gap: 6px; margin: 2px 0; }
        .dot { width: 10px; height: 10px; border-radius: 50%; display: inline-block; }
    </style>
</head>
<body>
    <div id="hud">
        <h3>LOD Performance</h3>
        <div class="stat"><span>FPS:</span><span id="fps">--</span></div>
        <div class="stat"><span>Draw Calls:</span><span id="draws">--</span></div>
        <div class="stat"><span>Triangles:</span><span id="tris">--</span></div>
        <div class="stat"><span>Objects:</span><span id="objs">1000</span></div>
        <div id="lodColors" style="margin-top:10px;border-top:1px solid #333;padding-top:8px;">
            <div class="lc"><span class="dot" style="background:#ff4466"></span> LOD 0 (near, high)</div>
            <div class="lc"><span class="dot" style="background:#ffaa22"></span> LOD 1 (mid)</div>
            <div class="lc"><span class="dot" style="background:#44ff88"></span> LOD 2 (far)</div>
            <div class="lc"><span class="dot" style="background:#4488ff"></span> LOD 3 (very far)</div>
            <div class="lc"><span class="dot" style="background:#aaaaaa"></span> Billboard</div>
        </div>
    </div>
    <div id="controls">
        <div style="margin-bottom:8px;font-size:13px;color:#fff">Settings</div>
        <button class="toggle-btn" id="lodToggle">LOD: ON</button>
        <button class="toggle-btn" id="colorToggle">Color by LOD: ON</button>
        <div style="margin-top:8px;font-size:11px;color:#777">
            WASD = Move<br>
            Mouse Drag = Look<br>
            Scroll = Speed
        </div>
    </div>

    <script type="module">
        import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js';

        const W = window.innerWidth, H = window.innerHeight;
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(W, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        document.body.appendChild(renderer.domElement);

        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x0a0a14);
        scene.fog = new THREE.Fog(0x0a0a14, 80, 300);

        const camera = new THREE.PerspectiveCamera(70, W / H, 0.1, 500);
        camera.position.set(0, 5, 0);

        // Ground
        const groundGeo = new THREE.PlaneGeometry(500, 500, 50, 50);
        const groundMat = new THREE.MeshStandardMaterial({
            color: 0x1a1a2e,
            roughness: 0.9, metalness: 0.0,
            wireframe: false
        });
        const ground = new THREE.Mesh(groundGeo, groundMat);
        ground.rotation.x = -Math.PI / 2;
        scene.add(ground);

        // Grid helper
        const grid = new THREE.GridHelper(500, 100, 0x223344, 0x112233);
        scene.add(grid);

        // Lights
        scene.add(new THREE.AmbientLight(0x334455, 0.8));
        const sun = new THREE.DirectionalLight(0xffffff, 1.5);
        sun.position.set(50, 80, 50);
        scene.add(sun);

        // LOD Geometries (4 levels)
        const geo0 = new THREE.IcosahedronGeometry(0.5, 4);  // ~1280 tri
        const geo1 = new THREE.IcosahedronGeometry(0.5, 2);  // ~320 tri
        const geo2 = new THREE.IcosahedronGeometry(0.5, 1);  // ~80 tri
        const geo3 = new THREE.IcosahedronGeometry(0.5, 0);  // ~20 tri

        // Billboard geometry
        const geoB = new THREE.PlaneGeometry(1, 1);

        const lodColors = [0xff4466, 0xffaa22, 0x44ff88, 0x4488ff, 0xaaaaaa];

        let useLOD = true;
        let colorByLOD = true;

        const materials = {
            lod: lodColors.map(c => new THREE.MeshStandardMaterial({
                color: c,
                roughness: 0.7, metalness: 0.2
            })),
            single: new THREE.MeshStandardMaterial({ color: 0xff4466, roughness: 0.7, metalness: 0.2 }),
            billboard: new THREE.MeshBasicMaterial({ color: 0xaaaaaa, side: THREE.DoubleSide })
        };

        const lodObjects = [];

        function createLODObject(x, z) {
            const lod = new THREE.LOD();

            const m0 = new THREE.Mesh(geo0, materials.lod[0]);
            const m1 = new THREE.Mesh(geo1, materials.lod[1]);
            const m2 = new THREE.Mesh(geo2, materials.lod[2]);
            const m3 = new THREE.Mesh(geo3, materials.lod[3]);
            const mb = new THREE.Mesh(geoB, materials.billboard);

            lod.addLevel(m0, 0);
            lod.addLevel(m1, 15);
            lod.addLevel(m2, 40);
            lod.addLevel(m3, 70);
            lod.addLevel(mb, 120);

            lod.position.set(x, 0.5, z);
            return lod;
        }

        function createSingleDetailObject(x, z) {
            const m = new THREE.Mesh(geo0, materials.single);
            m.position.set(x, 0.5, z);
            return m;
        }

        // สร้าง 1000 objects แบบกระจาย
        const SPREAD = 200;
        for (let i = 0; i < 1000; i++) {
            const x = (Math.random() - 0.5) * SPREAD;
            const z = (Math.random() - 0.5) * SPREAD;
            const obj = createLODObject(x, z);
            lodObjects.push(obj);
            scene.add(obj);
        }

        // LOD toggle
        document.getElementById('lodToggle').addEventListener('click', (e) => {
            useLOD = !useLOD;
            e.target.textContent = `LOD: ${useLOD ? 'ON' : 'OFF'}`;
            e.target.classList.toggle('off', !useLOD);

            lodObjects.forEach(lod => {
                if (!useLOD) {
                    // Force LOD 0 only
                    lod.levels.forEach((l, i) => {
                        l.object.visible = i === 0;
                    });
                }
            });
        });

        document.getElementById('colorToggle').addEventListener('click', (e) => {
            colorByLOD = !colorByLOD;
            e.target.textContent = `Color by LOD: ${colorByLOD ? 'ON' : 'OFF'}`;
            e.target.classList.toggle('off', !colorByLOD);
        });

        // FPS & Stats
        let lastTime = performance.now();
        let frameCount = 0;
        let fps = 0;

        // FPS camera controls (simple WASD)
        const keys = {};
        let moveSpeed = 0.3;
        document.addEventListener('keydown', e => keys[e.code] = true);
        document.addEventListener('keyup', e => keys[e.code] = false);

        let yaw = 0, pitch = 0;
        let isDragging = false;
        let lastMX = 0, lastMY = 0;

        renderer.domElement.addEventListener('mousedown', e => {
            isDragging = true; lastMX = e.clientX; lastMY = e.clientY;
        });
        renderer.domElement.addEventListener('mouseup', () => isDragging = false);
        renderer.domElement.addEventListener('mousemove', e => {
            if (!isDragging) return;
            yaw -= (e.clientX - lastMX) * 0.003;
            pitch -= (e.clientY - lastMY) * 0.003;
            pitch = Math.max(-1.2, Math.min(1.2, pitch));
            lastMX = e.clientX; lastMY = e.clientY;
            camera.rotation.order = 'YXZ';
            camera.rotation.y = yaw;
            camera.rotation.x = pitch;
        });
        renderer.domElement.addEventListener('wheel', e => {
            moveSpeed = Math.max(0.05, moveSpeed - e.deltaY * 0.001);
        });

        const _moveDir = new THREE.Vector3();
        const _forward = new THREE.Vector3();
        const _right = new THREE.Vector3();

        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });

        const clock = new THREE.Clock();

        function animate() {
            requestAnimationFrame(animate);
            const dt = clock.getDelta();

            // Camera movement
            camera.getWorldDirection(_forward);
            _right.crossVectors(_forward, camera.up).normalize();
            if (keys['KeyW']) camera.position.addScaledVector(_forward, moveSpeed);
            if (keys['KeyS']) camera.position.addScaledVector(_forward, -moveSpeed);
            if (keys['KeyA']) camera.position.addScaledVector(_right, -moveSpeed);
            if (keys['KeyD']) camera.position.addScaledVector(_right, moveSpeed);
            camera.position.y = Math.max(1, camera.position.y);

            // Update LODs
            if (useLOD) {
                lodObjects.forEach(lod => lod.update(camera));
            }

            // FPS counter
            frameCount++;
            const now = performance.now();
            if (now - lastTime >= 500) {
                fps = Math.round(frameCount * 1000 / (now - lastTime));
                frameCount = 0;
                lastTime = now;

                const info = renderer.info;
                document.getElementById('fps').textContent = fps;
                document.getElementById('draws').textContent = info.render.calls;
                document.getElementById('tris').textContent = info.render.triangles.toLocaleString();
            }

            renderer.render(scene, camera);
        }
        animate();
    </script>
</body>
</html>
```

## Terrain LOD System

```javascript
// QuadTree Terrain LOD
class TerrainLOD {
    constructor(scene, size, maxDepth) {
        this.scene = scene;
        this.size = size;
        this.maxDepth = maxDepth;
        this.chunks = new Map();
    }
    
    createChunkGeometry(x, z, width, depth) {
        const resolution = Math.max(4, 32 >> depth); // ลด resolution ตาม depth
        const geo = new THREE.PlaneGeometry(width, width, resolution, resolution);
        geo.rotateX(-Math.PI / 2);
        geo.translate(x, 0, z);
        
        // Apply height function
        const pos = geo.attributes.position.array;
        for (let i = 0; i < pos.length; i += 3) {
            const wx = pos[i], wz = pos[i + 2];
            pos[i + 1] = this.heightAt(wx, wz);
        }
        geo.computeVertexNormals();
        return geo;
    }
    
    heightAt(x, z) {
        // Simplex noise terrain
        return Math.sin(x * 0.1) * Math.cos(z * 0.1) * 5;
    }
    
    update(cameraPos) {
        // สร้าง/ลบ chunk ตาม camera distance
        // Implementation ขึ้นกับ quadtree structure
    }
}
```

## LOD สำหรับ Particles

```javascript
// Particle LOD: ใช้ Points ต่างขนาดตามระยะ
class ParticleLOD {
    constructor(count) {
        const positions = new Float32Array(count * 3);
        
        for (let i = 0; i < count; i++) {
            positions[i * 3] = (Math.random() - 0.5) * 100;
            positions[i * 3 + 1] = Math.random() * 10;
            positions[i * 3 + 2] = (Math.random() - 0.5) * 100;
        }
        
        const geo = new THREE.BufferGeometry();
        geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
        
        this.points = new THREE.Points(geo, new THREE.PointsMaterial({
            size: 0.2,
            sizeAttenuation: true, // ขนาดเล็กลงตามระยะ (built-in LOD)
            color: 0xffffff,
            transparent: true,
            opacity: 0.7
        }));
    }
}
```

## SimplifyModifier

Three.js มี SimplifyModifier สำหรับลด polygon count:

```javascript
import { SimplifyModifier } from 'three/examples/jsm/modifiers/SimplifyModifier.js';

const modifier = new SimplifyModifier();

// ลด geometry ลง 50%
const originalGeo = new THREE.TorusKnotGeometry(1, 0.3, 64, 16);
const vertexCount = originalGeo.attributes.position.count;
const targetCount = Math.floor(vertexCount * 0.5);

const simplifiedGeo = modifier.modify(originalGeo, targetCount);

// สร้าง LOD levels
const lod = new THREE.LOD();
lod.addLevel(new THREE.Mesh(originalGeo, material), 0);
lod.addLevel(new THREE.Mesh(simplifiedGeo, material), 20);
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: City LOD
สร้างเมืองจำลอง 500 buildings:
- LOD 0: detailed box + windows texture
- LOD 1: simple box
- LOD 2: billboard sprite
- วัด FPS ก่อนหลัง LOD

### แบบฝึกหัดที่ 2: Terrain LOD
สร้าง infinite terrain โดยใช้:
- QuadTree ระดับ 5
- HeightMap procedural
- ลด resolution ตามระยะ

### แบบฝึกหัดที่ 3: LOD Hysteresis
ป้องกัน LOD popping โดย:
- ใช้ hysteresis band (threshold เข้า ≠ threshold ออก)
- Smooth transition ระหว่าง levels
- Fade บน billboard ก่อน swap

### แบบฝึกหัดที่ 4: Dynamic LOD Budget
สร้างระบบที่:
- กำหนด triangle budget (เช่น 500,000)
- ปรับ LOD thresholds อัตโนมัติ
- รักษา FPS ที่ 60

---

**สรุปบทที่ 57:**
- THREE.LOD จัดการ level switching อัตโนมัติตาม camera distance
- addLevel(mesh, distance) กำหนด threshold แต่ละ level
- Billboard/imposters ใช้แทน mesh ในระยะไกล
- SimplifyModifier ลด polygon count ได้ง่าย
- Terrain LOD ใช้ QuadTree เลือก chunk ที่เหมาะสม
- ปรับ updateInterval ของ LOD เพื่อลด CPU overhead

**บทต่อไป:** Instanced Mesh
