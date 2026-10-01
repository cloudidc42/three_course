# Part 58: Instanced Mesh

## แนวคิด GPU Instancing

GPU Instancing เป็นเทคนิคที่ render geometry เดิมซ้ำหลายครั้งโดยส่ง instance data (matrix, color) ทั้งหมดไปพร้อมกันใน single draw call แทนที่จะส่ง draw call แยกสำหรับทุก object

```
วิธีปกติ (1000 objects = 1000 draw calls):
CPU → GPU: drawMesh(A)
CPU → GPU: drawMesh(A)
... × 1000

Instancing (1000 objects = 1 draw call):
CPU → GPU: drawInstanced(A, [matrix0, matrix1, ..., matrix999])
```

## THREE.InstancedMesh

```javascript
import * as THREE from 'three';

// InstancedMesh(geometry, material, count)
const geometry = new THREE.BoxGeometry(1, 1, 1);
const material = new THREE.MeshStandardMaterial({ color: 0xffffff });
const COUNT = 10000;

const instancedMesh = new THREE.InstancedMesh(geometry, material, COUNT);
instancedMesh.instanceMatrix.setUsage(THREE.DynamicDrawUsage); // สำหรับ animate
scene.add(instancedMesh);

// ใช้ dummy Object3D เป็น transform helper
const dummy = new THREE.Object3D();

for (let i = 0; i < COUNT; i++) {
    dummy.position.set(
        (Math.random() - 0.5) * 100,
        Math.random() * 10,
        (Math.random() - 0.5) * 100
    );
    dummy.rotation.set(
        Math.random() * Math.PI,
        Math.random() * Math.PI,
        0
    );
    dummy.scale.setScalar(0.5 + Math.random() * 0.5);
    
    dummy.updateMatrix(); // สำคัญ! ต้อง call ก่อน
    instancedMesh.setMatrixAt(i, dummy.matrix);
}

// บอก Three.js ว่า matrix เปลี่ยน
instancedMesh.instanceMatrix.needsUpdate = true;
```

## Per-Instance Color

```javascript
// เปิดใช้ per-instance color
instancedMesh.instanceColor = new THREE.InstancedBufferAttribute(
    new Float32Array(COUNT * 3), 3
);

const color = new THREE.Color();

for (let i = 0; i < COUNT; i++) {
    color.setHSL(i / COUNT, 0.8, 0.5); // gradient สี
    instancedMesh.setColorAt(i, color);
}

instancedMesh.instanceColor.needsUpdate = true;
```

## Getting Instance Data

```javascript
// อ่าน matrix กลับมา
const matrix = new THREE.Matrix4();
instancedMesh.getMatrixAt(5, matrix);

// แยก position/rotation/scale
const position = new THREE.Vector3();
const quaternion = new THREE.Quaternion();
const scale = new THREE.Vector3();
matrix.decompose(position, quaternion, scale);

// อ่าน color กลับมา
const color = new THREE.Color();
instancedMesh.getColorAt(5, color);
```

## Animated Instanced Mesh

```javascript
class AnimatedInstances {
    constructor(count, geometry, material) {
        this.count = count;
        this.mesh = new THREE.InstancedMesh(geometry, material, count);
        this.mesh.instanceMatrix.setUsage(THREE.DynamicDrawUsage);
        
        // Store instance data สำหรับ animation
        this.positions = new Float32Array(count * 3);
        this.velocities = new Float32Array(count * 3);
        this.phases = new Float32Array(count);
        
        this.dummy = new THREE.Object3D();
        this._init();
    }
    
    _init() {
        for (let i = 0; i < this.count; i++) {
            this.positions[i*3] = (Math.random() - 0.5) * 50;
            this.positions[i*3+1] = Math.random() * 5;
            this.positions[i*3+2] = (Math.random() - 0.5) * 50;
            this.phases[i] = Math.random() * Math.PI * 2;
        }
    }
    
    update(time) {
        for (let i = 0; i < this.count; i++) {
            const px = this.positions[i*3];
            const py = this.positions[i*3+1];
            const pz = this.positions[i*3+2];
            
            this.dummy.position.set(
                px,
                py + Math.sin(time * 2 + this.phases[i]) * 0.3,
                pz
            );
            this.dummy.rotation.y = time + this.phases[i];
            this.dummy.updateMatrix();
            this.mesh.setMatrixAt(i, this.dummy.matrix);
        }
        this.mesh.instanceMatrix.needsUpdate = true;
    }
}
```

## ตัวอย่าง: 1 Million Instances Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>1M Instanced Mesh</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; font-family: monospace; }
        canvas { display: block; }
        #hud {
            position: fixed; top: 20px; left: 20px;
            background: rgba(0,0,0,0.8); color: #00ffcc;
            padding: 16px; border-radius: 8px;
            border: 1px solid #00ffcc; min-width: 220px;
        }
        h3 { color: #00ffcc; margin-bottom: 10px; }
        .row { display: flex; justify-content: space-between; margin: 3px 0; font-size: 13px; }
        .val { color: #ffcc44; }
        #slider-panel {
            position: fixed; bottom: 20px; left: 20px;
            background: rgba(0,0,0,0.8); color: #ccc;
            padding: 16px; border-radius: 8px; border: 1px solid #555;
            width: 300px;
        }
        .ctrl { margin: 8px 0; }
        .ctrl label { display: flex; justify-content: space-between; font-size: 12px; margin-bottom: 3px; }
        .ctrl input { width: 100%; accent-color: #00ffcc; }
    </style>
</head>
<body>
    <div id="hud">
        <h3>Instanced Mesh</h3>
        <div class="row"><span>FPS</span><span class="val" id="fps">--</span></div>
        <div class="row"><span>Draw Calls</span><span class="val" id="draws">--</span></div>
        <div class="row"><span>Triangles</span><span class="val" id="tris">--</span></div>
        <div class="row"><span>Instances</span><span class="val" id="count">--</span></div>
        <div class="row"><span>Frustum Culled</span><span class="val" id="culled">auto</span></div>
    </div>
    <div id="slider-panel">
        <div class="ctrl">
            <label><span>Count</span><span id="cntVal">50000</span></label>
            <input type="range" id="countSlider" min="1000" max="1000000" step="1000" value="50000">
        </div>
        <div class="ctrl">
            <label><span>Animation Speed</span><span id="spdVal">1.0</span></label>
            <input type="range" id="speedSlider" min="0" max="5" step="0.1" value="1.0">
        </div>
        <div class="ctrl">
            <label><span>Spread</span><span id="sprVal">100</span></label>
            <input type="range" id="spreadSlider" min="10" max="500" step="10" value="100">
        </div>
        <button onclick="rebuildMesh()" style="
            width:100%;padding:8px;background:transparent;
            color:#00ffcc;border:1px solid #00ffcc;cursor:pointer;
            border-radius:4px;margin-top:6px;font-size:13px;">
            Rebuild Mesh
        </button>
    </div>

    <script type="module">
        import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js';
        import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/controls/OrbitControls.js';

        const W = window.innerWidth, H = window.innerHeight;
        const renderer = new THREE.WebGLRenderer({ antialias: false });
        renderer.setSize(W, H);
        renderer.setPixelRatio(1);
        document.body.appendChild(renderer.domElement);

        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x050510);

        const camera = new THREE.PerspectiveCamera(70, W/H, 0.1, 2000);
        camera.position.set(0, 50, 150);

        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;

        scene.add(new THREE.AmbientLight(0x334466, 1.0));
        const dirLight = new THREE.DirectionalLight(0xffffff, 1.5);
        dirLight.position.set(100, 200, 100);
        scene.add(dirLight);
        scene.add(new THREE.DirectionalLight(0x4466ff, 0.5).position.set(-100, 50, -100));

        const geo = new THREE.BoxGeometry(0.5, 0.5, 0.5);
        const mat = new THREE.MeshStandardMaterial({ vertexColors: true, roughness: 0.6, metalness: 0.4 });

        let instancedMesh = null;
        let phases = null;
        let basePositions = null;
        let animSpeed = 1.0;
        let spread = 100;
        let count = 50000;

        function rebuildMesh() {
            count = parseInt(document.getElementById('countSlider').value);
            document.getElementById('cntVal').textContent = count.toLocaleString();
            document.getElementById('count').textContent = count.toLocaleString();

            if (instancedMesh) {
                scene.remove(instancedMesh);
                instancedMesh.geometry.dispose();
                // Note: don't dispose shared geometry/material
            }

            instancedMesh = new THREE.InstancedMesh(geo, mat, count);
            instancedMesh.instanceMatrix.setUsage(THREE.DynamicDrawUsage);

            phases = new Float32Array(count);
            basePositions = new Float32Array(count * 3);

            const color = new THREE.Color();

            const dummy = new THREE.Object3D();

            for (let i = 0; i < count; i++) {
                const x = (Math.random() - 0.5) * spread;
                const y = Math.random() * 20;
                const z = (Math.random() - 0.5) * spread;

                basePositions[i*3] = x;
                basePositions[i*3+1] = y;
                basePositions[i*3+2] = z;
                phases[i] = Math.random() * Math.PI * 2;

                dummy.position.set(x, y, z);
                dummy.rotation.set(Math.random() * Math.PI, Math.random() * Math.PI, 0);
                dummy.updateMatrix();
                instancedMesh.setMatrixAt(i, dummy.matrix);

                // Per-instance color
                color.setHSL((i / count + Math.random() * 0.1) % 1, 0.7, 0.5);
                instancedMesh.setColorAt(i, color);
            }

            instancedMesh.instanceMatrix.needsUpdate = true;
            if (instancedMesh.instanceColor) instancedMesh.instanceColor.needsUpdate = true;

            scene.add(instancedMesh);
        }

        window.rebuildMesh = rebuildMesh;
        rebuildMesh();

        document.getElementById('countSlider').addEventListener('input', e => {
            document.getElementById('cntVal').textContent = parseInt(e.target.value).toLocaleString();
        });
        document.getElementById('speedSlider').addEventListener('input', e => {
            animSpeed = parseFloat(e.target.value);
            document.getElementById('spdVal').textContent = animSpeed.toFixed(1);
        });
        document.getElementById('spreadSlider').addEventListener('input', e => {
            spread = parseInt(e.target.value);
            document.getElementById('sprVal').textContent = spread;
        });

        const dummy = new THREE.Object3D();
        let fpsFrames = 0, fpsLast = performance.now();

        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });

        const clock = new THREE.Clock();
        function animate() {
            requestAnimationFrame(animate);
            const t = clock.getElapsedTime() * animSpeed;

            if (instancedMesh && phases) {
                for (let i = 0; i < count; i++) {
                    dummy.position.set(
                        basePositions[i*3],
                        basePositions[i*3+1] + Math.sin(t + phases[i]) * 0.5,
                        basePositions[i*3+2]
                    );
                    dummy.rotation.y = t * 0.5 + phases[i];
                    dummy.rotation.x = Math.sin(t * 0.3 + phases[i]) * 0.5;
                    dummy.updateMatrix();
                    instancedMesh.setMatrixAt(i, dummy.matrix);
                }
                instancedMesh.instanceMatrix.needsUpdate = true;
            }

            controls.update();

            fpsFrames++;
            const now = performance.now();
            if (now - fpsLast >= 500) {
                const fps = Math.round(fpsFrames * 1000 / (now - fpsLast));
                document.getElementById('fps').textContent = fps;
                document.getElementById('draws').textContent = renderer.info.render.calls;
                document.getElementById('tris').textContent = renderer.info.render.triangles.toLocaleString();
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

## Animated Grass Field

```javascript
// Grass blade ที่ sway ตามลม
class GrassField {
    constructor(count, area) {
        this.count = count;
        
        // สร้าง grass blade geometry
        const geo = new THREE.PlaneGeometry(0.1, 0.5, 1, 3);
        geo.translate(0, 0.25, 0); // pivot ที่ base
        
        // Custom grass shader
        const mat = new THREE.ShaderMaterial({
            uniforms: {
                time: { value: 0 },
                windDirection: { value: new THREE.Vector2(1, 0.3) },
                windStrength: { value: 0.3 }
            },
            vertexShader: `
                uniform float time;
                uniform vec2 windDirection;
                uniform float windStrength;
                
                attribute vec3 instanceOffset;
                attribute float instancePhase;
                
                varying vec2 vUv;
                varying float vHeight;
                
                void main() {
                    vUv = uv;
                    vHeight = uv.y;
                    
                    // Wind sway ตาม vertex height
                    float sway = sin(time * 2.0 + instancePhase) * windStrength * vHeight;
                    
                    vec3 pos = position;
                    pos.x += windDirection.x * sway;
                    pos.z += windDirection.y * sway;
                    
                    // Place at instance position
                    pos += instanceOffset;
                    
                    gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
                }
            `,
            fragmentShader: `
                varying vec2 vUv;
                varying float vHeight;
                
                void main() {
                    vec3 baseColor = vec3(0.1, 0.4, 0.05);
                    vec3 tipColor = vec3(0.5, 0.8, 0.1);
                    vec3 color = mix(baseColor, tipColor, vHeight);
                    
                    if (vUv.x < 0.02 || vUv.x > 0.98) discard;
                    
                    gl_FragColor = vec4(color, 1.0);
                }
            `,
            side: THREE.DoubleSide
        });
        
        this.mesh = new THREE.InstancedMesh(geo, mat, count);
        
        const offsets = new Float32Array(count * 3);
        const phaseArr = new Float32Array(count);
        
        for (let i = 0; i < count; i++) {
            offsets[i*3] = (Math.random() - 0.5) * area;
            offsets[i*3+1] = 0;
            offsets[i*3+2] = (Math.random() - 0.5) * area;
            phaseArr[i] = Math.random() * Math.PI * 2;
        }
        
        geo.setAttribute('instanceOffset', new THREE.InstancedBufferAttribute(offsets, 3));
        geo.setAttribute('instancePhase', new THREE.InstancedBufferAttribute(phaseArr, 1));
        this.mat = mat;
    }
    
    update(time) {
        this.mat.uniforms.time.value = time;
    }
}
```

## Crowd Simulation

```javascript
// Simple crowd simulation ด้วย InstancedMesh
class CrowdSimulation {
    constructor(count) {
        this.count = count;
        this.positions = new Float32Array(count * 3);
        this.velocities = new Float32Array(count * 3);
        this.targets = new Float32Array(count * 3);
        
        // Initialize ตำแหน่ง random
        for (let i = 0; i < count; i++) {
            this.positions[i*3] = (Math.random() - 0.5) * 50;
            this.positions[i*3+2] = (Math.random() - 0.5) * 50;
            this._setRandomTarget(i);
        }
        
        const geo = new THREE.CapsuleGeometry(0.3, 1, 4, 8);
        const mat = new THREE.MeshStandardMaterial({ color: 0x4488cc });
        this.mesh = new THREE.InstancedMesh(geo, mat, count);
        this.mesh.instanceMatrix.setUsage(THREE.DynamicDrawUsage);
        this.dummy = new THREE.Object3D();
    }
    
    _setRandomTarget(i) {
        this.targets[i*3] = (Math.random() - 0.5) * 50;
        this.targets[i*3+2] = (Math.random() - 0.5) * 50;
    }
    
    update(dt) {
        const speed = 5;
        const turnSpeed = 3;
        
        for (let i = 0; i < this.count; i++) {
            const px = this.positions[i*3], pz = this.positions[i*3+2];
            const tx = this.targets[i*3], tz = this.targets[i*3+2];
            
            const dx = tx - px, dz = tz - pz;
            const dist = Math.sqrt(dx*dx + dz*dz);
            
            if (dist < 1.0) {
                this._setRandomTarget(i);
            } else {
                const nx = dx / dist, nz = dz / dist;
                this.positions[i*3] += nx * speed * dt;
                this.positions[i*3+2] += nz * speed * dt;
                
                this.dummy.position.set(px, 0.8, pz);
                this.dummy.rotation.y = Math.atan2(nx, nz);
                this.dummy.updateMatrix();
                this.mesh.setMatrixAt(i, this.dummy.matrix);
            }
        }
        this.mesh.instanceMatrix.needsUpdate = true;
    }
}
```

## Frustum Culling กับ InstancedMesh

```javascript
// InstancedMesh ไม่มี per-instance frustum culling built-in
// ต้องทำเอง

class CulledInstances {
    constructor(geometry, material, positions) {
        this.allPositions = positions;
        this.geometry = geometry;
        this.material = material;
        
        this.mesh = new THREE.InstancedMesh(geometry, material, positions.length);
        this.mesh.frustumCulled = false; // ปิด mesh-level culling
        
        this._frustum = new THREE.Frustum();
        this._projScreenMatrix = new THREE.Matrix4();
        this._sphere = new THREE.Sphere();
        this.dummy = new THREE.Object3D();
    }
    
    update(camera) {
        // คำนวณ frustum
        this._projScreenMatrix.multiplyMatrices(
            camera.projectionMatrix,
            camera.matrixWorldInverse
        );
        this._frustum.setFromProjectionMatrix(this._projScreenMatrix);
        
        let visibleCount = 0;
        
        for (const pos of this.allPositions) {
            this._sphere.center.copy(pos);
            this._sphere.radius = 1.0;
            
            if (this._frustum.intersectsSphere(this._sphere)) {
                this.dummy.position.copy(pos);
                this.dummy.updateMatrix();
                this.mesh.setMatrixAt(visibleCount++, this.dummy.matrix);
            }
        }
        
        this.mesh.count = visibleCount;
        this.mesh.instanceMatrix.needsUpdate = true;
    }
}
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Instanced Trees
สร้างป่า 100,000 ต้นไม้:
- LOD: high poly ใกล้, low poly ไกล, billboard ไกลมาก
- Random rotation + scale ต่อ instance
- Seasonal color change ผ่าน instance color

### แบบฝึกหัดที่ 2: Space Particles
สร้างกาแล็กซี 500,000 stars:
- Orbit animation แต่ละดาว
- Color gradient ตาม orbit radius
- Interactive: click เพื่อ select star

### แบบฝึกหัดที่ 3: Minecraft Voxel
สร้าง voxel world ด้วย InstancedMesh:
- Chunk system (16x16x16 blocks)
- Only render visible faces
- Block types ด้วย instance color

### แบบฝึกหัดที่ 4: Boid Simulation
สร้าง flocking behavior:
- Separation, Alignment, Cohesion
- 5,000 boids ด้วย InstancedMesh
- หมุนตาม velocity direction

---

**สรุปบทที่ 58:**
- InstancedMesh render ซ้ำ geometry ด้วย single draw call
- setMatrixAt/getMatrixAt ใช้ Object3D dummy เป็น helper
- setColorAt/getColorAt ควบคุม per-instance color
- instanceMatrix.needsUpdate = true ทุกครั้งที่เปลี่ยน matrix
- DynamicDrawUsage สำหรับ animation, StaticDrawUsage สำหรับ static
- InstancedBufferAttribute ใส่ custom per-instance data ได้

**บทต่อไป:** Geometry Merging & Batching
