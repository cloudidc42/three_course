# Part 55: Outline, Glow & Selection Effects

## แนวคิด Outline Effects

Outline effects ใช้ในหลายบริบท:
- **Game Selection** - แสดงว่า object ใดถูก select
- **UI Feedback** - highlight เมื่อ hover
- **Cel Shading** - เส้นขอบสไตล์การ์ตูน
- **X-Ray Vision** - มองเห็น object ผ่านกำแพง
- **Enemy Detection** - แสดง enemy ที่ซ่อนอยู่

## OutlinePass - ใช้งาน

```javascript
// การตั้งค่า OutlinePass แบบละเอียด
import { OutlinePass } from 'three/examples/jsm/postprocessing/OutlinePass.js';

const outlinePass = new OutlinePass(
    new THREE.Vector2(window.innerWidth, window.innerHeight),
    scene,
    camera
);

// พารามิเตอร์ทั้งหมด
outlinePass.edgeStrength = 3.0;          // ความแรง (ความสว่าง) ของ edge
outlinePass.edgeGlow = 0.0;              // ระยะ glow รอบขอบ (0 = ไม่มี glow)
outlinePass.edgeThickness = 1.0;         // ความหนาของเส้นขอบ (pixels)
outlinePass.pulsePeriod = 0;             // 0 = ไม่กระพริบ, >0 = ระยะเวลา pulse (seconds)
outlinePass.usePatternTexture = false;   // ใช้ pattern texture สำหรับ fill

outlinePass.visibleEdgeColor.set('#ff0000');  // สีของ edge ที่มองเห็น
outlinePass.hiddenEdgeColor.set('#ff0000');   // สีของ edge ที่ถูกบัง (ผ่าน wall)

// กำหนด objects ที่มี outline
outlinePass.selectedObjects = [mesh1, mesh2, mesh3];

composer.addPass(outlinePass);
```

## Multi-Object Outline System

```javascript
// ระบบ outline สำหรับหลาย groups
class OutlineManager {
    constructor(composer, scene, camera, width, height) {
        this.passes = new Map();
        this.composer = composer;
        this.scene = scene;
        this.camera = camera;
        this.width = width;
        this.height = height;
    }
    
    // สร้าง outline group ใหม่
    createGroup(name, options = {}) {
        const pass = new OutlinePass(
            new THREE.Vector2(this.width, this.height),
            this.scene,
            this.camera
        );
        
        pass.edgeStrength = options.strength || 3.0;
        pass.edgeGlow = options.glow || 0.0;
        pass.edgeThickness = options.thickness || 1.0;
        pass.pulsePeriod = options.pulse || 0;
        pass.visibleEdgeColor.set(options.visibleColor || '#ffffff');
        pass.hiddenEdgeColor.set(options.hiddenColor || '#888888');
        pass.selectedObjects = [];
        
        this.passes.set(name, pass);
        this.composer.addPass(pass);
        
        return pass;
    }
    
    addObject(groupName, object) {
        const pass = this.passes.get(groupName);
        if (pass && !pass.selectedObjects.includes(object)) {
            pass.selectedObjects.push(object);
        }
    }
    
    removeObject(groupName, object) {
        const pass = this.passes.get(groupName);
        if (pass) {
            const idx = pass.selectedObjects.indexOf(object);
            if (idx !== -1) pass.selectedObjects.splice(idx, 1);
        }
    }
    
    clearGroup(groupName) {
        const pass = this.passes.get(groupName);
        if (pass) pass.selectedObjects = [];
    }
    
    setGroupVisible(groupName, visible) {
        const pass = this.passes.get(groupName);
        if (pass) pass.enabled = visible;
    }
}

// ใช้งาน
const outlineManager = new OutlineManager(composer, scene, camera, W, H);
outlineManager.createGroup('selection', { visibleColor: '#ffff00', strength: 4, thickness: 2 });
outlineManager.createGroup('hover', { visibleColor: '#00ffff', strength: 2, glow: 0.5 });
outlineManager.createGroup('enemy', { visibleColor: '#ff0000', hiddenColor: '#ff000088', strength: 3 });
```

## Stencil Buffer Outline

วิธีที่ใช้ stencil buffer (ไม่ต้องใช้ post-processing pass):

```javascript
// Stencil buffer outline
// ขั้นตอน:
// 1. Render object ลง stencil buffer
// 2. Render object ที่ขยายออกมา ตรวจสอบ stencil (แสดงเฉพาะส่วนที่ไม่มี object เดิม)

function createStencilOutline(mesh, color = 0xff0000, outlineSize = 0.05) {
    // Object จริง - เขียน stencil
    mesh.material.stencilWrite = true;
    mesh.material.stencilRef = 1;
    mesh.material.stencilFunc = THREE.AlwaysStencilFunc;
    mesh.material.stencilZPassOp = THREE.ReplaceStencilOp;
    mesh.renderOrder = 1;
    
    // Outline - แสดงเฉพาะที่ stencil ไม่มีค่า 1
    const outlineGeo = mesh.geometry.clone();
    const outlineMat = new THREE.MeshBasicMaterial({
        color: color,
        side: THREE.BackSide,
        stencilWrite: true,
        stencilRef: 1,
        stencilFunc: THREE.NotEqualStencilFunc,   // แสดงเฉพาะที่ไม่มี stencil
        stencilFail: THREE.KeepStencilOp,
        stencilZFail: THREE.KeepStencilOp,
        stencilZPass: THREE.KeepStencilOp
    });
    
    const outlineMesh = new THREE.Mesh(outlineGeo, outlineMat);
    outlineMesh.position.copy(mesh.position);
    outlineMesh.rotation.copy(mesh.rotation);
    outlineMesh.scale.setScalar(1 + outlineSize / mesh.geometry.boundingSphere.radius);
    outlineMesh.renderOrder = 0;
    
    return outlineMesh;
}
```

## ตัวอย่างที่ 1: Game Selection Outline

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Game Selection Outline</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; font-family: Arial, sans-serif; }
        canvas { display: block; cursor: crosshair; }
        #ui {
            position: fixed;
            top: 20px;
            left: 20px;
            background: rgba(0,0,0,0.85);
            color: white;
            padding: 15px;
            border-radius: 10px;
            width: 250px;
        }
        h3 { color: #ffcc00; margin-bottom: 12px; }
        .ctrl { margin-bottom: 10px; }
        .ctrl label { display: flex; justify-content: space-between; font-size: 12px; margin-bottom: 3px; }
        .ctrl input[type=range] { width: 100%; }
        .color-row { display: flex; gap: 8px; margin-top: 8px; }
        .color-swatch {
            width: 30px; height: 30px; border-radius: 5px;
            cursor: pointer; border: 2px solid transparent;
            transition: border-color 0.2s;
        }
        .color-swatch.active { border-color: white; }
        #info {
            position: fixed;
            bottom: 20px;
            left: 50%;
            transform: translateX(-50%);
            background: rgba(0,0,0,0.7);
            color: white;
            padding: 8px 20px;
            border-radius: 20px;
            font-size: 14px;
            text-align: center;
        }
        #selected-info {
            margin-top: 10px;
            padding: 8px;
            background: rgba(255,204,0,0.1);
            border: 1px solid #ffcc00;
            border-radius: 5px;
            font-size: 12px;
            color: #ffcc00;
        }
    </style>
</head>
<body>
    <div id="ui">
        <h3>🎮 Selection System</h3>
        <div class="ctrl">
            <label><span>Edge Strength</span><span id="esVal">4</span></label>
            <input type="range" id="edgeStrength" min="1" max="10" step="0.5" value="4">
        </div>
        <div class="ctrl">
            <label><span>Edge Thickness</span><span id="etVal">2</span></label>
            <input type="range" id="edgeThick" min="1" max="5" step="0.5" value="2">
        </div>
        <div class="ctrl">
            <label><span>Glow</span><span id="glowVal">0.3</span></label>
            <input type="range" id="glow" min="0" max="2" step="0.1" value="0.3">
        </div>
        <div class="ctrl">
            <label>Outline Color:</label>
            <div class="color-row">
                <div class="color-swatch active" style="background:#ffcc00" data-col="#ffcc00"></div>
                <div class="color-swatch" style="background:#00ffcc" data-col="#00ffcc"></div>
                <div class="color-swatch" style="background:#ff4444" data-col="#ff4444"></div>
                <div class="color-swatch" style="background:#4488ff" data-col="#4488ff"></div>
                <div class="color-swatch" style="background:#ff44ff" data-col="#ff44ff"></div>
            </div>
        </div>
        <div id="selected-info">คลิก object เพื่อ select<br>Shift+คลิก = multi-select</div>
    </div>
    <div id="info">คลิก object เพื่อ select | Shift = เพิ่ม/ลด | ESC = deselect ทั้งหมด</div>

    <script type="module">
        import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js';
        import { EffectComposer } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/EffectComposer.js';
        import { RenderPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/RenderPass.js';
        import { OutlinePass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/OutlinePass.js';
        import { OutputPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/OutputPass.js';
        import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/controls/OrbitControls.js';

        const W = window.innerWidth, H = window.innerHeight;
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(W, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        renderer.shadowMap.enabled = true;
        document.body.appendChild(renderer.domElement);

        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x1a1a2e);

        const camera = new THREE.PerspectiveCamera(60, W/H, 0.1, 100);
        camera.position.set(0, 8, 14);
        camera.lookAt(0, 0, 0);

        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;

        // Lighting
        scene.add(new THREE.AmbientLight(0x334466, 1));
        const dirLight = new THREE.DirectionalLight(0xffffff, 2);
        dirLight.position.set(10, 15, 10);
        dirLight.castShadow = true;
        dirLight.shadow.mapSize.set(2048, 2048);
        scene.add(dirLight);

        // Ground
        const ground = new THREE.Mesh(
            new THREE.PlaneGeometry(30, 30),
            new THREE.MeshStandardMaterial({ color: 0x223344, roughness: 0.9 })
        );
        ground.rotation.x = -Math.PI / 2;
        ground.receiveShadow = true;
        scene.add(ground);

        // Grid lines on ground
        const gridHelper = new THREE.GridHelper(30, 15, 0x334455, 0x334455);
        scene.add(gridHelper);

        // Selectables - units/objects
        const units = [];
        const unitTypes = [
            { geo: new THREE.BoxGeometry(1.5, 1.5, 1.5), color: 0x4488cc, name: 'Cube' },
            { geo: new THREE.SphereGeometry(0.9, 32, 32), color: 0xcc4444, name: 'Sphere' },
            { geo: new THREE.CylinderGeometry(0.6, 0.8, 1.8, 8), color: 0x44cc44, name: 'Cylinder' },
            { geo: new THREE.ConeGeometry(0.8, 1.8, 8), color: 0xcccc44, name: 'Cone' },
            { geo: new THREE.TorusGeometry(0.7, 0.25, 16, 32), color: 0xcc44cc, name: 'Torus' },
            { geo: new THREE.OctahedronGeometry(1.0), color: 0x44cccc, name: 'Octa' },
        ];

        for (let row = 0; row < 3; row++) {
            for (let col = 0; col < 4; col++) {
                const typeIdx = (row * 4 + col) % unitTypes.length;
                const type = unitTypes[typeIdx];
                
                const mat = new THREE.MeshStandardMaterial({
                    color: type.color,
                    roughness: 0.4,
                    metalness: 0.3
                });
                
                const mesh = new THREE.Mesh(type.geo, mat);
                mesh.position.set(col * 4 - 6, 0.9, row * 4 - 4);
                mesh.castShadow = true;
                mesh.receiveShadow = true;
                mesh.userData = {
                    name: `${type.name} ${row * 4 + col + 1}`,
                    type: type.name,
                    selected: false,
                    hovered: false
                };
                scene.add(mesh);
                units.push(mesh);
            }
        }

        // Post-processing
        const composer = new EffectComposer(renderer);
        composer.addPass(new RenderPass(scene, camera));

        // Selection outline (yellow)
        const selectionOutline = new OutlinePass(new THREE.Vector2(W, H), scene, camera);
        selectionOutline.edgeStrength = 4;
        selectionOutline.edgeThickness = 2;
        selectionOutline.edgeGlow = 0.3;
        selectionOutline.pulsePeriod = 2; // กระพริบทุก 2 วินาที
        selectionOutline.visibleEdgeColor.set('#ffcc00');
        selectionOutline.hiddenEdgeColor.set('#aa8800');
        selectionOutline.selectedObjects = [];
        composer.addPass(selectionOutline);

        // Hover outline (cyan)
        const hoverOutline = new OutlinePass(new THREE.Vector2(W, H), scene, camera);
        hoverOutline.edgeStrength = 2;
        hoverOutline.edgeThickness = 1;
        hoverOutline.edgeGlow = 0;
        hoverOutline.visibleEdgeColor.set('#00ffff');
        hoverOutline.hiddenEdgeColor.set('#00aaaa');
        hoverOutline.selectedObjects = [];
        composer.addPass(hoverOutline);

        composer.addPass(new OutputPass());

        // Raycasting
        const raycaster = new THREE.Raycaster();
        const mouse = new THREE.Vector2();
        let hoveredObject = null;
        const selectedObjects = new Set();

        renderer.domElement.addEventListener('mousemove', (e) => {
            mouse.x = (e.clientX / W) * 2 - 1;
            mouse.y = -(e.clientY / H) * 2 + 1;
        });

        renderer.domElement.addEventListener('click', (e) => {
            raycaster.setFromCamera(mouse, camera);
            const hits = raycaster.intersectObjects(units);
            
            if (hits.length > 0) {
                const obj = hits[0].object;
                
                if (e.shiftKey) {
                    // Multi-select
                    if (selectedObjects.has(obj)) {
                        selectedObjects.delete(obj);
                    } else {
                        selectedObjects.add(obj);
                    }
                } else {
                    // Single select
                    selectedObjects.clear();
                    selectedObjects.add(obj);
                }
                
                updateOutlines();
                updateInfo();
            } else if (!e.shiftKey) {
                selectedObjects.clear();
                updateOutlines();
                updateInfo();
            }
        });

        document.addEventListener('keydown', (e) => {
            if (e.key === 'Escape') {
                selectedObjects.clear();
                updateOutlines();
                updateInfo();
            }
        });

        function updateOutlines() {
            selectionOutline.selectedObjects = [...selectedObjects];
        }

        function updateInfo() {
            const info = document.getElementById('selected-info');
            if (selectedObjects.size === 0) {
                info.innerHTML = 'คลิก object เพื่อ select<br>Shift+คลิก = multi-select';
            } else {
                const names = [...selectedObjects].map(o => o.userData.name).join(', ');
                info.innerHTML = `<strong>Selected:</strong> ${selectedObjects.size} object(s)<br><small>${names}</small>`;
            }
        }

        // UI
        document.getElementById('edgeStrength').addEventListener('input', (e) => {
            selectionOutline.edgeStrength = parseFloat(e.target.value);
            document.getElementById('esVal').textContent = e.target.value;
        });
        document.getElementById('edgeThick').addEventListener('input', (e) => {
            selectionOutline.edgeThickness = parseFloat(e.target.value);
            document.getElementById('etVal').textContent = e.target.value;
        });
        document.getElementById('glow').addEventListener('input', (e) => {
            selectionOutline.edgeGlow = parseFloat(e.target.value);
            document.getElementById('glowVal').textContent = e.target.value;
        });
        document.querySelectorAll('.color-swatch').forEach(sw => {
            sw.addEventListener('click', () => {
                document.querySelectorAll('.color-swatch').forEach(s => s.classList.remove('active'));
                sw.classList.add('active');
                selectionOutline.visibleEdgeColor.set(sw.dataset.col);
            });
        });

        window.addEventListener('resize', () => {
            const w = window.innerWidth, h = window.innerHeight;
            camera.aspect = w / h; camera.updateProjectionMatrix();
            renderer.setSize(w, h); composer.setSize(w, h);
        });

        const clock = new THREE.Clock();
        function animate() {
            requestAnimationFrame(animate);
            const dt = clock.getDelta();
            const time = clock.getElapsedTime();
            
            // Hover detection
            raycaster.setFromCamera(mouse, camera);
            const hits = raycaster.intersectObjects(units);
            
            const newHovered = hits.length > 0 ? hits[0].object : null;
            if (newHovered !== hoveredObject) {
                hoveredObject = newHovered;
                hoverOutline.selectedObjects = hoveredObject && !selectedObjects.has(hoveredObject) 
                    ? [hoveredObject] : [];
            }
            
            // Animate unselected units
            units.forEach((unit, i) => {
                if (!selectedObjects.has(unit)) {
                    unit.position.y = 0.9 + Math.sin(time * 0.5 + i * 0.5) * 0.1;
                } else {
                    unit.position.y = 0.9 + Math.sin(time * 2) * 0.15; // bounce when selected
                }
            });

            controls.update();
            composer.render();
        }
        animate();
    </script>
</body>
</html>
```

## X-Ray Scanner Effect

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>X-Ray Scanner Effect</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; font-family: Arial, sans-serif; }
        canvas { display: block; }
        #ui {
            position: fixed;
            top: 20px;
            right: 20px;
            background: rgba(0,20,0,0.9);
            color: #00ff88;
            padding: 15px;
            border-radius: 10px;
            width: 230px;
            border: 1px solid #00ff88;
        }
        h3 { color: #00ff88; margin-bottom: 12px; font-size: 14px; }
        .ctrl { margin-bottom: 8px; font-size: 12px; }
        .ctrl label { display: flex; justify-content: space-between; margin-bottom: 3px; }
        .ctrl input { width: 100%; accent-color: #00ff88; }
        .mode-btn {
            display: block; width: 100%; padding: 7px;
            background: transparent; color: #00ff88;
            border: 1px solid #00ff88; cursor: pointer;
            text-align: center; margin-bottom: 5px; font-size: 12px;
            border-radius: 4px;
        }
        .mode-btn.active { background: #00ff8830; }
    </style>
</head>
<body>
    <div id="ui">
        <h3>☢️ X-RAY SCANNER</h3>
        <div class="ctrl">
            <label><span>X-Ray Opacity</span><span id="xrOpacity">0.3</span></label>
            <input type="range" id="xropacity" min="0" max="0.8" step="0.01" value="0.3">
        </div>
        <div class="ctrl">
            <label><span>Scan Line Speed</span><span id="scanSpeed">1.0</span></label>
            <input type="range" id="scanspeed" min="0" max="3" step="0.1" value="1.0">
        </div>
        <div class="ctrl">
            <label><span>Edge Glow</span><span id="xrGlow">0.5</span></label>
            <input type="range" id="xrglow" min="0" max="2" step="0.1" value="0.5">
        </div>
        <p style="font-size:11px;color:#007744;margin:8px 0">Mode:</p>
        <button class="mode-btn active" data-mode="full">Full X-Ray</button>
        <button class="mode-btn" data-mode="outline">Outline Only</button>
        <button class="mode-btn" data-mode="hidden">Hidden Objects</button>
    </div>

    <script type="module">
        import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js';
        import { EffectComposer } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/EffectComposer.js';
        import { RenderPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/RenderPass.js';
        import { OutlinePass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/OutlinePass.js';
        import { ShaderPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/ShaderPass.js';
        import { OutputPass } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/postprocessing/OutputPass.js';
        import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/controls/OrbitControls.js';

        const W = window.innerWidth, H = window.innerHeight;
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(W, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        document.body.appendChild(renderer.domElement);

        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x0a0a0a);

        const camera = new THREE.PerspectiveCamera(60, W/H, 0.1, 100);
        camera.position.set(0, 5, 10);
        camera.lookAt(0, 0, 0);
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;

        scene.add(new THREE.AmbientLight(0x002200, 1));
        const dirLight = new THREE.DirectionalLight(0x00ff00, 0.5);
        dirLight.position.set(5, 10, 5);
        scene.add(dirLight);

        // Container box (transparent)
        const containerMat = new THREE.MeshStandardMaterial({
            color: 0x334433,
            transparent: true,
            opacity: 0.2,
            side: THREE.DoubleSide
        });
        const container = new THREE.Mesh(new THREE.BoxGeometry(8, 6, 8), containerMat);
        container.position.y = 3;
        scene.add(container);

        // Hidden objects inside
        const hiddenObjects = [];
        const hiddenGeos = [
            new THREE.SphereGeometry(0.8, 32, 32),
            new THREE.BoxGeometry(1.2, 1.2, 1.2),
            new THREE.CylinderGeometry(0.5, 0.5, 1.5, 16),
            new THREE.TorusGeometry(0.6, 0.2, 16, 32),
            new THREE.ConeGeometry(0.7, 1.5, 8)
        ];
        const hiddenColors = [0xff4444, 0x4444ff, 0x44ff44, 0xff44ff, 0xffff44];
        
        hiddenGeos.forEach((geo, i) => {
            const mesh = new THREE.Mesh(geo, new THREE.MeshStandardMaterial({
                color: hiddenColors[i],
                emissive: hiddenColors[i],
                emissiveIntensity: 0.3
            }));
            mesh.position.set(
                (Math.random() - 0.5) * 6,
                1 + Math.random() * 4,
                (Math.random() - 0.5) * 6
            );
            mesh.rotation.set(Math.random(), Math.random(), Math.random());
            scene.add(mesh);
            hiddenObjects.push(mesh);
        });

        // Post-processing
        const composer = new EffectComposer(renderer);
        composer.addPass(new RenderPass(scene, camera));

        const xrayOutline = new OutlinePass(new THREE.Vector2(W, H), scene, camera);
        xrayOutline.edgeStrength = 4;
        xrayOutline.edgeThickness = 2;
        xrayOutline.edgeGlow = 0.5;
        xrayOutline.visibleEdgeColor.set('#00ff88');
        xrayOutline.hiddenEdgeColor.set('#00aa55'); // เห็นผ่าน container
        xrayOutline.selectedObjects = hiddenObjects;
        composer.addPass(xrayOutline);

        // Scan line effect
        const ScanlineShader = {
            uniforms: {
                tDiffuse: { value: null },
                time: { value: 0 },
                scanSpeed: { value: 1.0 },
                tintColor: { value: new THREE.Color(0, 0.3, 0.1) }
            },
            vertexShader: `varying vec2 vUv; void main() { vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
            fragmentShader: `
                uniform sampler2D tDiffuse;
                uniform float time;
                uniform float scanSpeed;
                uniform vec3 tintColor;
                varying vec2 vUv;
                
                void main() {
                    vec4 color = texture2D(tDiffuse, vUv);
                    
                    // Scanlines
                    float line = sin(vUv.y * 300.0) * 0.02;
                    color.rgb += line;
                    
                    // Scan effect (moving line)
                    float scanY = mod(time * scanSpeed, 1.0);
                    float scanLine = exp(-pow((vUv.y - scanY) / 0.05, 2.0)) * 0.15;
                    color.rgb += vec3(0.0, scanLine, scanLine * 0.5);
                    
                    // Green tint
                    color.rgb = mix(color.rgb, color.rgb * vec3(0.7, 1.3, 0.9), 0.3);
                    
                    // Vignette
                    vec2 uv2 = vUv * 2.0 - 1.0;
                    float vig = 1.0 - dot(uv2, uv2) * 0.4;
                    color.rgb *= vig;
                    
                    gl_FragColor = color;
                }
            `
        };
        const scanPass = new ShaderPass(ScanlineShader);
        composer.addPass(scanPass);
        composer.addPass(new OutputPass());

        // UI Controls
        document.getElementById('xropacity').addEventListener('input', (e) => {
            const v = parseFloat(e.target.value);
            hiddenObjects.forEach(o => { o.material.opacity = v; o.material.transparent = v < 1; });
            document.getElementById('xrOpacity').textContent = v.toFixed(2);
        });
        document.getElementById('scanspeed').addEventListener('input', (e) => {
            scanPass.uniforms.scanSpeed.value = parseFloat(e.target.value);
            document.getElementById('scanSpeed').textContent = e.target.value;
        });
        document.getElementById('xrglow').addEventListener('input', (e) => {
            xrayOutline.edgeGlow = parseFloat(e.target.value);
            document.getElementById('xrGlow').textContent = e.target.value;
        });
        document.querySelectorAll('.mode-btn').forEach(btn => {
            btn.addEventListener('click', () => {
                document.querySelectorAll('.mode-btn').forEach(b => b.classList.remove('active'));
                btn.classList.add('active');
                
                if (btn.dataset.mode === 'full') {
                    xrayOutline.hiddenEdgeColor.set('#00aa55');
                    containerMat.opacity = 0.2;
                } else if (btn.dataset.mode === 'outline') {
                    xrayOutline.hiddenEdgeColor.set('#00ff88');
                    containerMat.opacity = 0.05;
                } else if (btn.dataset.mode === 'hidden') {
                    xrayOutline.hiddenEdgeColor.set('#ff4400');
                    containerMat.opacity = 0.3;
                }
            });
        });

        window.addEventListener('resize', () => {
            const w = window.innerWidth, h = window.innerHeight;
            camera.aspect = w / h; camera.updateProjectionMatrix();
            renderer.setSize(w, h); composer.setSize(w, h);
        });

        const clock = new THREE.Clock();
        function animate() {
            requestAnimationFrame(animate);
            const t = clock.getElapsedTime();
            scanPass.uniforms.time.value = t;
            hiddenObjects.forEach((obj, i) => {
                obj.rotation.y = t * 0.3 + i;
                obj.position.y = 1 + Math.sin(t * 0.5 + i) * 0.3 + i * 0.8;
            });
            controls.update();
            composer.render();
        }
        animate();
    </script>
</body>
</html>
```

## Heat Distortion Outline

```javascript
// Heat Distortion Outline Shader
const heatDistortionShader = {
    uniforms: {
        tDiffuse: { value: null },
        time: { value: 0 },
        outlineColor: { value: new THREE.Color(1, 0.5, 0) },
        intensity: { value: 0.02 },
        frequency: { value: 10.0 }
    },
    vertexShader: `varying vec2 vUv; void main() { vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
    fragmentShader: `
        uniform sampler2D tDiffuse;
        uniform float time;
        uniform vec3 outlineColor;
        uniform float intensity;
        uniform float frequency;
        
        varying vec2 vUv;
        
        float rand(vec2 co) {
            return fract(sin(dot(co, vec2(12.9898, 78.233))) * 43758.5453);
        }
        
        void main() {
            // Distort UV near bright areas (outline)
            float edge = 0.0;
            float step = 0.003;
            for (int dx = -1; dx <= 1; dx++) {
                for (int dy = -1; dy <= 1; dy++) {
                    if (dx == 0 && dy == 0) continue;
                    vec2 offset = vec2(float(dx), float(dy)) * step;
                    vec4 neighbor = texture2D(tDiffuse, vUv + offset);
                    vec4 center = texture2D(tDiffuse, vUv);
                    edge += length(neighbor.rgb - center.rgb);
                }
            }
            edge = clamp(edge, 0.0, 1.0);
            
            // Heat distortion near edges
            vec2 distortion = vec2(
                sin(vUv.y * frequency + time * 3.0) * intensity,
                cos(vUv.x * frequency + time * 2.0) * intensity
            );
            
            vec4 distorted = texture2D(tDiffuse, vUv + distortion * edge);
            
            // Add heat shimmer color
            vec3 heat = mix(distorted.rgb, outlineColor, edge * 0.5);
            
            gl_FragColor = vec4(heat, 1.0);
        }
    `
};
```

## Pulse Glow Effect

```javascript
// Pulse glow ที่ใช้กับ OutlinePass
class PulseGlowManager {
    constructor(outlinePass) {
        this.outlinePass = outlinePass;
        this.baseStrength = 3.0;
        this.pulseAmplitude = 1.5;
        this.pulseSpeed = 2.0;
    }
    
    update(time) {
        const pulse = Math.sin(time * this.pulseSpeed * Math.PI * 2) * 0.5 + 0.5;
        this.outlinePass.edgeStrength = this.baseStrength + pulse * this.pulseAmplitude;
        this.outlinePass.edgeGlow = pulse * 0.5;
    }
    
    setWarning(active) {
        if (active) {
            this.outlinePass.visibleEdgeColor.set('#ff0000');
            this.pulseSpeed = 4.0;
        } else {
            this.outlinePass.visibleEdgeColor.set('#ffcc00');
            this.pulseSpeed = 1.0;
        }
    }
}
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: RTS Game Selection
สร้าง RTS-style unit selection:
- คลิกลาก = selection box (rectangle drag select)
- CTRL = add to selection
- ESC = deselect
- Selected units แสดง health bar เหนือหัว

### แบบฝึกหัดที่ 2: Night Vision Goggle Effect
สร้าง night vision effect:
- Green tint + scanlines
- Fuzzy edges (noise)
- Objects ที่ถูก outline สว่างกว่า

### แบบฝึกหัดที่ 3: Multi-team Outline
สร้าง outline ที่ต่างสีตาม team:
- Team A: สีน้ำเงิน (ally)
- Team B: สีแดง (enemy)
- Neutral: สีเหลือง
- แต่ละ team มี pulse pattern ต่างกัน

### แบบฝึกหัดที่ 4: Interactive X-Ray
สร้าง interactive x-ray ที่:
- กดปุ่มค้างไว้ = เห็น hidden objects ผ่าน walls
- X-ray area ตามตำแหน่ง mouse (spotlight-style)
- Animation เมื่อ "scan" เจอ object

---

**สรุปบทที่ 55:**
- OutlinePass ยืดหยุ่นสูง ควบคุม strength, thickness, glow, และ color
- Multi-pass outline ทำให้แต่ละ group มีสีต่างกัน
- Back-face expansion ทำงานได้กับทุก mesh แต่มี artifact กับ concave shapes
- Stencil buffer ให้ control ที่ precise กว่าสำหรับ complex scenarios
- Pulse/animated outlines เพิ่ม feedback ที่ดีในเกม

**บทต่อไป:** Advanced Normal & Displacement Mapping
