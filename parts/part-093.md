# Part 93: HTML in 3D ด้วย CSS3DRenderer

## บทนำ

ในบทนี้เราจะเรียนรู้การใช้ `CSS3DRenderer` ซึ่งเป็น renderer พิเศษของ Three.js ที่ช่วยให้เราสามารถนำ HTML elements ทั่วไปมาแสดงผลในพื้นที่ 3D ได้ นี่เป็นเครื่องมือที่ทรงพลังมากสำหรับการสร้าง UI ที่มีความลึกและมีความ interactive

## CSS3DRenderer vs WebGLRenderer

### ความแตกต่างพื้นฐาน

| คุณสมบัติ | CSS3DRenderer | WebGLRenderer |
|-----------|--------------|---------------|
| เทคโนโลยี | CSS3 transform | WebGL/GPU |
| Content | HTML DOM elements | Meshes/Geometries |
| Interactive | ใช้ pointer events HTML ได้เลย | ต้องใช้ raycaster |
| Performance | ดีสำหรับ UI elements | ดีสำหรับ 3D graphics |
| Post-processing | ไม่รองรับโดยตรง | รองรับเต็มที่ |
| Text/Font | HTML/CSS ใดก็ได้ | ต้อง TextGeometry/Texture |

### เมื่อไหรควรใช้ CSS3DRenderer

**ใช้ CSS3DRenderer เมื่อ:**
- ต้องการแสดง HTML content (วิดีโอ, ฟอร์ม, รูปภาพ) ใน 3D space
- ต้องการ UI ที่ interactive โดยไม่ต้องเขียน raycaster
- สร้าง portfolio ที่มีการ์ดแบบ 3D
- สร้าง presentation หรือ data visualization ที่มี HTML

**ใช้ WebGLRenderer เมื่อ:**
- ต้องการ post-processing effects
- ต้องการ physically based rendering
- สร้าง game หรือ simulation ที่ซับซ้อน
- ต้องการ shader effects

## การติดตั้งและ Setup พื้นฐาน

CSS3DRenderer อยู่ใน `three/examples/jsm/renderers/CSS3DRenderer.js`

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CSS3DRenderer - พื้นฐาน</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; }
        #webgl-container, #css3d-container {
            position: absolute;
            top: 0; left: 0;
            width: 100%; height: 100%;
        }
        #css3d-container { pointer-events: none; }
        .card {
            width: 200px;
            height: 150px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            border-radius: 12px;
            color: white;
            display: flex;
            align-items: center;
            justify-content: center;
            font-family: Arial, sans-serif;
            font-size: 18px;
            font-weight: bold;
            cursor: pointer;
            pointer-events: auto;
            transition: transform 0.2s;
            box-shadow: 0 10px 30px rgba(0,0,0,0.5);
        }
        .card:hover { transform: scale(1.05); }
    </style>
</head>
<body>
    <div id="webgl-container"></div>
    <div id="css3d-container"></div>

    <script type="importmap">
    {
        "imports": {
            "three": "https://cdn.jsdelivr.net/npm/three@0.165.0/build/three.module.js",
            "three/examples/jsm/": "https://cdn.jsdelivr.net/npm/three@0.165.0/examples/jsm/"
        }
    }
    </script>
    <script type="module">
        import * as THREE from 'three';
        import { CSS3DRenderer, CSS3DObject } from 'three/examples/jsm/renderers/CSS3DRenderer.js';
        import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

        // Scene setup
        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 1, 5000);
        camera.position.set(0, 0, 800);

        // WebGL Renderer
        const webglRenderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
        webglRenderer.setSize(window.innerWidth, window.innerHeight);
        webglRenderer.setPixelRatio(window.devicePixelRatio);
        document.getElementById('webgl-container').appendChild(webglRenderer.domElement);

        // CSS3D Renderer
        const css3dRenderer = new CSS3DRenderer();
        css3dRenderer.setSize(window.innerWidth, window.innerHeight);
        document.getElementById('css3d-container').appendChild(css3dRenderer.domElement);

        // Controls (ต้องผูกกับ CSS3D container เพราะอยู่บนสุด)
        const controls = new OrbitControls(camera, css3dRenderer.domElement);
        controls.enableDamping = true;

        // สร้าง HTML card
        const cardElement = document.createElement('div');
        cardElement.className = 'card';
        cardElement.textContent = '🎨 CSS3D Card';
        cardElement.addEventListener('click', () => {
            alert('คลิกที่การ์ด!');
        });

        // ห่อ HTML ด้วย CSS3DObject
        const css3dObject = new CSS3DObject(cardElement);
        css3dObject.position.set(0, 0, 0);
        scene.add(css3dObject);

        // Animation loop
        function animate() {
            requestAnimationFrame(animate);
            controls.update();
            webglRenderer.render(scene, camera);
            css3dRenderer.render(scene, camera);
        }
        animate();

        // Resize handler
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            webglRenderer.setSize(window.innerWidth, window.innerHeight);
            css3dRenderer.setSize(window.innerWidth, window.innerHeight);
        });
    </script>
</body>
</html>
```

## CSS3DObject - การห่อ HTML Elements

`CSS3DObject` ใช้สำหรับวาง HTML element ใดก็ได้ใน 3D space โดย element จะถูกแปลงด้วย CSS3 transform matrix

### คุณสมบัติของ CSS3DObject

```javascript
const element = document.createElement('div');
const obj = new CSS3DObject(element);

// position, rotation, scale ทำงานเหมือน Object3D ปกติ
obj.position.set(x, y, z);
obj.rotation.set(rx, ry, rz);
obj.scale.set(sx, sy, sz);

// เพิ่มเป็น child ได้เหมือนกัน
parentObject.add(obj);
```

## CSS3DSprite - Always Faces Camera

`CSS3DSprite` คือ CSS3DObject พิเศษที่จะหมุนหา camera เสมอ (billboarding) เหมาะสำหรับ labels และ markers

```javascript
import { CSS3DSprite } from 'three/examples/jsm/renderers/CSS3DRenderer.js';

const labelEl = document.createElement('div');
labelEl.textContent = '📍 Point of Interest';
labelEl.style.cssText = `
    background: rgba(0,0,0,0.8);
    color: white;
    padding: 8px 12px;
    border-radius: 6px;
    font-size: 14px;
    white-space: nowrap;
    pointer-events: none;
`;

const sprite = new CSS3DSprite(labelEl);
sprite.position.set(100, 50, 0);
scene.add(sprite);
```

## การรวม CSS3DRenderer + WebGLRenderer ใน Scene เดียวกัน

นี่คือ pattern ที่ใช้บ่อยที่สุด — ให้ renderers สองตัวทำงานร่วมกัน

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Combined Renderers</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #111; overflow: hidden; }
        
        /* สำคัญ: ทั้งสอง container ต้อง stack ทับกัน */
        #webgl-container, #css3d-container {
            position: absolute;
            top: 0; left: 0;
            width: 100%; height: 100%;
        }
        
        /* CSS3D อยู่บน WebGL เสมอ */
        #css3d-container { pointer-events: none; }
        
        .info-panel {
            width: 160px;
            background: rgba(20, 20, 40, 0.9);
            border: 1px solid rgba(100, 100, 255, 0.5);
            border-radius: 8px;
            padding: 12px;
            color: #adf;
            font-family: 'Courier New', monospace;
            font-size: 12px;
            pointer-events: auto;
            backdrop-filter: blur(4px);
        }
        .info-panel h3 {
            color: #88f;
            margin-bottom: 8px;
            font-size: 14px;
        }
        .info-panel .value {
            color: #ff8;
        }
    </style>
</head>
<body>
    <div id="webgl-container"></div>
    <div id="css3d-container"></div>

    <script type="importmap">
    {
        "imports": {
            "three": "https://cdn.jsdelivr.net/npm/three@0.165.0/build/three.module.js",
            "three/examples/jsm/": "https://cdn.jsdelivr.net/npm/three@0.165.0/examples/jsm/"
        }
    }
    </script>
    <script type="module">
        import * as THREE from 'three';
        import { CSS3DRenderer, CSS3DObject, CSS3DSprite } from 'three/examples/jsm/renderers/CSS3DRenderer.js';
        import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

        const W = window.innerWidth, H = window.innerHeight;

        // Scene
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x111122);
        
        // Camera
        const camera = new THREE.PerspectiveCamera(60, W / H, 1, 10000);
        camera.position.set(0, 300, 800);

        // ===== WebGL Renderer =====
        const webglRenderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
        webglRenderer.setSize(W, H);
        webglRenderer.shadowMap.enabled = true;
        document.getElementById('webgl-container').appendChild(webglRenderer.domElement);

        // ===== CSS3D Renderer =====
        const css3dRenderer = new CSS3DRenderer();
        css3dRenderer.setSize(W, H);
        document.getElementById('css3d-container').appendChild(css3dRenderer.domElement);

        // Controls
        const controls = new OrbitControls(camera, css3dRenderer.domElement);
        controls.enableDamping = true;
        controls.target.set(0, 0, 0);

        // ===== WebGL Objects =====
        // Grid
        const grid = new THREE.GridHelper(1000, 20, 0x334455, 0x223344);
        scene.add(grid);

        // สร้าง 3D objects
        const objects = [];
        const geometries = [
            new THREE.BoxGeometry(80, 80, 80),
            new THREE.SphereGeometry(45, 32, 32),
            new THREE.CylinderGeometry(0, 45, 80, 6),
            new THREE.TorusGeometry(35, 15, 16, 32),
        ];
        const materials = [
            new THREE.MeshStandardMaterial({ color: 0x4466ff, roughness: 0.3, metalness: 0.7 }),
            new THREE.MeshStandardMaterial({ color: 0xff4488, roughness: 0.3, metalness: 0.7 }),
            new THREE.MeshStandardMaterial({ color: 0x44ff88, roughness: 0.3, metalness: 0.7 }),
            new THREE.MeshStandardMaterial({ color: 0xffaa22, roughness: 0.3, metalness: 0.7 }),
        ];

        const positions = [
            new THREE.Vector3(-250, 40, 0),
            new THREE.Vector3(-80, 45, 0),
            new THREE.Vector3(80, 40, 0),
            new THREE.Vector3(250, 35, 0),
        ];

        const names = ['Box', 'Sphere', 'Pyramid', 'Torus'];
        const infos = [
            { vertices: 8, faces: 12 },
            { vertices: 961, faces: 1920 },
            { vertices: 7, faces: 12 },
            { vertices: 576, faces: 1152 },
        ];

        // Lighting
        const ambient = new THREE.AmbientLight(0x334466, 0.6);
        scene.add(ambient);
        const dirLight = new THREE.DirectionalLight(0xffffff, 1.5);
        dirLight.position.set(300, 500, 200);
        dirLight.castShadow = true;
        scene.add(dirLight);

        geometries.forEach((geo, i) => {
            const mesh = new THREE.Mesh(geo, materials[i]);
            mesh.position.copy(positions[i]);
            mesh.castShadow = true;
            mesh.receiveShadow = true;
            scene.add(mesh);
            objects.push(mesh);

            // สร้าง info panel สำหรับแต่ละ object
            const panelEl = document.createElement('div');
            panelEl.className = 'info-panel';
            panelEl.innerHTML = `
                <h3>📦 ${names[i]}</h3>
                <div>Vertices: <span class="value">${infos[i].vertices}</span></div>
                <div>Faces: <span class="value">${infos[i].faces}</span></div>
                <div>Position:</div>
                <div>X: <span class="value">${positions[i].x}</span></div>
                <div>Y: <span class="value">${positions[i].y}</span></div>
            `;

            const css3dObj = new CSS3DObject(panelEl);
            css3dObj.position.set(
                positions[i].x,
                positions[i].y + 120,
                positions[i].z
            );
            css3dObj.scale.set(0.3, 0.3, 0.3); // scale ลงเพื่อให้พอดีกับ scene
            scene.add(css3dObj);
        });

        // Animation
        function animate() {
            requestAnimationFrame(animate);
            
            const t = performance.now() * 0.001;
            objects.forEach((obj, i) => {
                obj.rotation.y = t * (0.3 + i * 0.1);
                obj.rotation.x = t * 0.2;
            });

            controls.update();
            webglRenderer.render(scene, camera);
            css3dRenderer.render(scene, camera);
        }
        animate();

        window.addEventListener('resize', () => {
            const w = window.innerWidth, h = window.innerHeight;
            camera.aspect = w / h;
            camera.updateProjectionMatrix();
            webglRenderer.setSize(w, h);
            css3dRenderer.setSize(w, h);
        });
    </script>
</body>
</html>
```

## Z-Fighting ระหว่าง CSS และ WebGL

ปัญหา z-fighting เกิดขึ้นเมื่อ CSS3D elements และ WebGL objects อยู่ใน depth เดียวกัน

### วิธีแก้ไข

```javascript
// วิธีที่ 1: ใช้ renderer.domElement.style.zIndex
webglRenderer.domElement.style.zIndex = '1';
css3dRenderer.domElement.style.zIndex = '2';

// วิธีที่ 2: ปรับ camera near/far ให้เหมาะสม
camera = new THREE.PerspectiveCamera(75, aspect, 1, 10000);

// วิธีที่ 3: เพิ่ม offset ให้ CSS3D objects
css3dObject.position.z += 1; // เลื่อนมาข้างหน้าเล็กน้อย

// วิธีที่ 4: ใช้ alpha: true ใน WebGL renderer
const renderer = new THREE.WebGLRenderer({ alpha: true });
renderer.setClearColor(0x000000, 0); // transparent background
```

## Demo หลัก: 3D Flip Card Portfolio

นี่คือ demo ที่สมบูรณ์แบบที่สุดในบทนี้ — portfolio ที่ใช้ flip card animation

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>3D Flip Card Portfolio</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { 
            background: radial-gradient(ellipse at center, #1a1a2e 0%, #0d0d1a 100%);
            overflow: hidden; 
            font-family: 'Segoe UI', sans-serif;
        }
        #webgl-container, #css3d-container {
            position: absolute; top: 0; left: 0;
            width: 100%; height: 100%;
        }
        #css3d-container { pointer-events: none; }
        
        .flip-card {
            width: 220px;
            height: 300px;
            perspective: 600px;
            cursor: pointer;
            pointer-events: auto;
        }
        .flip-card-inner {
            width: 100%; height: 100%;
            position: relative;
            transform-style: preserve-3d;
            transition: transform 0.7s cubic-bezier(0.4, 0, 0.2, 1);
        }
        .flip-card.flipped .flip-card-inner {
            transform: rotateY(180deg);
        }
        .flip-card-front, .flip-card-back {
            position: absolute;
            width: 100%; height: 100%;
            backface-visibility: hidden;
            border-radius: 16px;
            overflow: hidden;
        }
        .flip-card-front {
            background: linear-gradient(135deg, var(--color1), var(--color2));
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            padding: 20px;
            color: white;
        }
        .flip-card-front .icon {
            font-size: 48px;
            margin-bottom: 16px;
        }
        .flip-card-front h2 {
            font-size: 20px;
            text-align: center;
            margin-bottom: 8px;
        }
        .flip-card-front p {
            font-size: 12px;
            text-align: center;
            opacity: 0.8;
        }
        .flip-card-back {
            background: rgba(15, 15, 30, 0.95);
            transform: rotateY(180deg);
            border: 1px solid rgba(255,255,255,0.1);
            padding: 20px;
            display: flex;
            flex-direction: column;
            color: white;
            backdrop-filter: blur(10px);
        }
        .flip-card-back h3 {
            font-size: 16px;
            margin-bottom: 12px;
            color: var(--accent);
        }
        .flip-card-back .desc {
            font-size: 12px;
            line-height: 1.6;
            opacity: 0.8;
            flex: 1;
        }
        .flip-card-back .tags {
            display: flex;
            flex-wrap: wrap;
            gap: 4px;
            margin-top: 12px;
        }
        .flip-card-back .tag {
            background: rgba(255,255,255,0.1);
            border-radius: 20px;
            padding: 3px 8px;
            font-size: 10px;
        }
        .flip-card-back .btn {
            margin-top: 12px;
            background: var(--accent);
            color: white;
            border: none;
            border-radius: 8px;
            padding: 8px 16px;
            font-size: 12px;
            cursor: pointer;
            width: 100%;
            transition: opacity 0.2s;
        }
        .flip-card-back .btn:hover { opacity: 0.8; }
        
        .hint {
            position: fixed;
            bottom: 20px;
            left: 50%;
            transform: translateX(-50%);
            color: rgba(255,255,255,0.5);
            font-size: 14px;
            pointer-events: none;
            z-index: 100;
        }
    </style>
</head>
<body>
    <div id="webgl-container"></div>
    <div id="css3d-container"></div>
    <div class="hint">คลิกที่การ์ดเพื่อดูรายละเอียด | ลากเพื่อหมุน</div>

    <script type="importmap">
    {
        "imports": {
            "three": "https://cdn.jsdelivr.net/npm/three@0.165.0/build/three.module.js",
            "three/examples/jsm/": "https://cdn.jsdelivr.net/npm/three@0.165.0/examples/jsm/"
        }
    }
    </script>
    <script type="module">
        import * as THREE from 'three';
        import { CSS3DRenderer, CSS3DObject } from 'three/examples/jsm/renderers/CSS3DRenderer.js';
        import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

        const W = window.innerWidth, H = window.innerHeight;

        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(60, W / H, 10, 5000);
        camera.position.set(0, 200, 1000);

        // WebGL renderer (สำหรับพื้นหลังและ effects)
        const webglRenderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
        webglRenderer.setSize(W, H);
        webglRenderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        document.getElementById('webgl-container').appendChild(webglRenderer.domElement);

        // CSS3D renderer
        const css3dRenderer = new CSS3DRenderer();
        css3dRenderer.setSize(W, H);
        document.getElementById('css3d-container').appendChild(css3dRenderer.domElement);

        // Controls
        const controls = new OrbitControls(camera, css3dRenderer.domElement);
        controls.enableDamping = true;
        controls.dampingFactor = 0.05;
        controls.minDistance = 200;
        controls.maxDistance = 2000;
        controls.maxPolarAngle = Math.PI * 0.7;

        // Lighting
        scene.add(new THREE.AmbientLight(0x334466, 0.5));
        const point1 = new THREE.PointLight(0x4466ff, 2, 2000);
        point1.position.set(-500, 500, 500);
        scene.add(point1);
        const point2 = new THREE.PointLight(0xff6644, 2, 2000);
        point2.position.set(500, 300, -500);
        scene.add(point2);

        // Particle background
        const particleGeo = new THREE.BufferGeometry();
        const particleCount = 2000;
        const positions = new Float32Array(particleCount * 3);
        for (let i = 0; i < particleCount * 3; i++) {
            positions[i] = (Math.random() - 0.5) * 3000;
        }
        particleGeo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
        const particleMat = new THREE.PointsMaterial({ color: 0x4466aa, size: 2, transparent: true, opacity: 0.6 });
        scene.add(new THREE.Points(particleGeo, particleMat));

        // Portfolio data
        const projects = [
            {
                icon: '🎮',
                title: 'Space Shooter',
                subtitle: 'เกม 2D Action',
                color1: '#667eea',
                color2: '#764ba2',
                accent: '#a78bfa',
                desc: 'เกม Shoot\'em Up แบบ classic สร้างด้วย Canvas API มีระบบ particle effects และ boss battles ที่น่าตื่นเต้น',
                tags: ['JavaScript', 'Canvas', 'Game Dev'],
            },
            {
                icon: '🌏',
                title: 'Globe Viz',
                subtitle: 'Data Visualization',
                color1: '#11998e',
                color2: '#38ef7d',
                accent: '#4ade80',
                desc: 'แสดงข้อมูล flight paths ทั่วโลกบน 3D globe แบบ real-time ใช้ Three.js และ D3.js ในการ render',
                tags: ['Three.js', 'D3.js', 'WebGL'],
            },
            {
                icon: '🤖',
                title: 'AI Chat',
                subtitle: 'Machine Learning App',
                color1: '#f093fb',
                color2: '#f5576c',
                accent: '#fb7185',
                desc: 'Chatbot ที่ใช้ Claude API สร้างบทสนทนาที่เป็นธรรมชาติ มีระบบ context memory และ multi-turn conversation',
                tags: ['React', 'Claude API', 'Node.js'],
            },
            {
                icon: '📊',
                title: 'Dashboard Pro',
                subtitle: 'Analytics Platform',
                color1: '#4facfe',
                color2: '#00f2fe',
                accent: '#38bdf8',
                desc: 'Real-time analytics dashboard สำหรับ e-commerce มีกราฟแบบ interactive และ data export หลายรูปแบบ',
                tags: ['Vue.js', 'Chart.js', 'WebSocket'],
            },
            {
                icon: '🏗️',
                title: 'ArchViz Studio',
                subtitle: '3D Architecture',
                color1: '#fa709a',
                color2: '#fee140',
                accent: '#fbbf24',
                desc: 'เครื่องมือ visualization สำหรับ architect ที่ใช้ PBR materials และ real-time ray tracing สำหรับ preview',
                tags: ['Three.js', 'GLSL', 'PBR'],
            },
            {
                icon: '🎵',
                title: 'Music Viz',
                subtitle: 'Audio Visualization',
                color1: '#a18cd1',
                color2: '#fbc2eb',
                accent: '#c084fc',
                desc: 'Visualizer ที่ใช้ Web Audio API + Three.js สร้าง particles ที่เต้นตามเพลง มี frequency analysis แบบ real-time',
                tags: ['Web Audio', 'Three.js', 'GLSL'],
            },
        ];

        // สร้าง flip cards และวางใน circle
        const radius = 400;
        projects.forEach((project, i) => {
            const angle = (i / projects.length) * Math.PI * 2;
            const x = Math.sin(angle) * radius;
            const z = Math.cos(angle) * radius;

            // สร้าง HTML card
            const cardEl = document.createElement('div');
            cardEl.className = 'flip-card';
            cardEl.style.setProperty('--color1', project.color1);
            cardEl.style.setProperty('--color2', project.color2);
            cardEl.style.setProperty('--accent', project.accent);
            
            cardEl.innerHTML = `
                <div class="flip-card-inner">
                    <div class="flip-card-front">
                        <div class="icon">${project.icon}</div>
                        <h2>${project.title}</h2>
                        <p>${project.subtitle}</p>
                    </div>
                    <div class="flip-card-back">
                        <h3>${project.title}</h3>
                        <div class="desc">${project.desc}</div>
                        <div class="tags">
                            ${project.tags.map(t => `<span class="tag">${t}</span>`).join('')}
                        </div>
                        <button class="btn">ดูโปรเจค →</button>
                    </div>
                </div>
            `;

            cardEl.addEventListener('click', () => {
                cardEl.classList.toggle('flipped');
            });

            const css3dObj = new CSS3DObject(cardEl);
            css3dObj.position.set(x, 0, z);
            // หมุนให้หันมาตรงกลาง
            css3dObj.lookAt(0, 0, 0);
            // scale ลงให้เหมาะสมกับ scene
            css3dObj.scale.set(0.5, 0.5, 0.5);
            scene.add(css3dObj);
        });

        // Floor plane (WebGL)
        const floorGeo = new THREE.CircleGeometry(600, 64);
        const floorMat = new THREE.MeshStandardMaterial({
            color: 0x111122,
            roughness: 0.8,
            metalness: 0.2,
        });
        const floor = new THREE.Mesh(floorGeo, floorMat);
        floor.rotation.x = -Math.PI / 2;
        floor.position.y = -160;
        scene.add(floor);

        // Center sphere
        const centerGeo = new THREE.IcosahedronGeometry(50, 2);
        const centerMat = new THREE.MeshStandardMaterial({
            color: 0x8888ff,
            roughness: 0.1,
            metalness: 0.9,
            wireframe: false,
        });
        const centerSphere = new THREE.Mesh(centerGeo, centerMat);
        centerSphere.position.y = 20;
        scene.add(centerSphere);

        // Animate
        let autoRotate = true;
        controls.addEventListener('start', () => { autoRotate = false; });

        function animate() {
            requestAnimationFrame(animate);
            
            const t = performance.now() * 0.001;
            
            if (autoRotate) {
                camera.position.x = Math.sin(t * 0.2) * 1000;
                camera.position.z = Math.cos(t * 0.2) * 1000;
                camera.position.y = 300 + Math.sin(t * 0.1) * 100;
                camera.lookAt(0, 0, 0);
            }

            centerSphere.rotation.y = t * 0.5;
            centerSphere.rotation.x = t * 0.3;
            
            controls.update();
            webglRenderer.render(scene, camera);
            css3dRenderer.render(scene, camera);
        }
        animate();

        window.addEventListener('resize', () => {
            const w = window.innerWidth, h = window.innerHeight;
            camera.aspect = w / h;
            camera.updateProjectionMatrix();
            webglRenderer.setSize(w, h);
            css3dRenderer.setSize(w, h);
        });
    </script>
</body>
</html>
```

## Pointer Events และการจัดการ Interaction

การจัดการ pointer events ใน CSS3D มีความละเอียดอ่อน:

```javascript
// CSS3D container ควร pointer-events: none เสมอ
// แล้วเปิด pointer-events เฉพาะ elements ที่ต้องการ interaction

css3dContainer.style.pointerEvents = 'none';

// สำหรับ element ที่ต้องการ interaction
myElement.style.pointerEvents = 'auto';

// WebGL controls ต้องผูกกับ CSS3D container (อยู่ข้างบน)
const controls = new OrbitControls(camera, css3dRenderer.domElement);

// แต่ถ้า element บางตัว capture events ไป controls จะไม่ทำงาน
// แก้ไขด้วยการใช้ stopPropagation อย่างระมัดระวัง
myElement.addEventListener('mousedown', (e) => {
    e.stopPropagation(); // ป้องกัน controls ทำงาน
});
```

## CSS3D กับ WebGL Post-Processing

CSS3DRenderer ไม่รองรับ EffectComposer โดยตรง แต่มี workaround:

```javascript
// Workaround: render WebGL ก่อน แล้ว CSS3D ทับ
// WebGL ใช้ EffectComposer ตามปกติ
// CSS3D render เป็น overlay

// ลำดับการ render:
function animate() {
    requestAnimationFrame(animate);
    
    // 1. Render WebGL ด้วย post-processing
    composer.render();
    
    // 2. Render CSS3D ทับ (ไม่มี post-processing)
    css3dRenderer.render(scene, camera);
}

// ข้อจำกัด: CSS3D elements จะไม่มี bloom, blur, etc.
// ถ้าต้องการ effect ให้ใช้ CSS filter แทน
element.style.filter = 'blur(2px) brightness(1.5)';
```

## Demo เพิ่มเติม: Info Panels ผูกกับ 3D Objects

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>3D Info Panels</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #0a0a1a; overflow: hidden; }
        #webgl-container, #css3d-container {
            position: absolute; top: 0; left: 0;
            width: 100%; height: 100%;
        }
        #css3d-container { pointer-events: none; }
        
        .planet-info {
            width: 140px;
            background: rgba(10, 20, 40, 0.9);
            border: 1px solid rgba(100, 160, 255, 0.4);
            border-radius: 10px;
            padding: 10px;
            color: white;
            font-family: Arial, sans-serif;
            font-size: 11px;
            pointer-events: auto;
            transform: scale(1);
            transition: transform 0.2s, border-color 0.2s;
        }
        .planet-info:hover {
            transform: scale(1.1);
            border-color: rgba(100, 160, 255, 0.8);
        }
        .planet-info .name {
            font-size: 14px;
            font-weight: bold;
            color: #88aaff;
            margin-bottom: 6px;
        }
        .planet-info .stat {
            display: flex;
            justify-content: space-between;
            margin: 2px 0;
        }
        .planet-info .stat-val { color: #ffcc44; }
        .line {
            display: flex;
            align-items: center;
            gap: 4px;
            margin: 4px 0;
        }
    </style>
</head>
<body>
    <div id="webgl-container"></div>
    <div id="css3d-container"></div>

    <script type="importmap">
    {
        "imports": {
            "three": "https://cdn.jsdelivr.net/npm/three@0.165.0/build/three.module.js",
            "three/examples/jsm/": "https://cdn.jsdelivr.net/npm/three@0.165.0/examples/jsm/"
        }
    }
    </script>
    <script type="module">
        import * as THREE from 'three';
        import { CSS3DRenderer, CSS3DObject } from 'three/examples/jsm/renderers/CSS3DRenderer.js';
        import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 1, 10000);
        camera.position.set(0, 400, 1200);

        const webglRenderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
        webglRenderer.setSize(window.innerWidth, window.innerHeight);
        document.getElementById('webgl-container').appendChild(webglRenderer.domElement);

        const css3dRenderer = new CSS3DRenderer();
        css3dRenderer.setSize(window.innerWidth, window.innerHeight);
        document.getElementById('css3d-container').appendChild(css3dRenderer.domElement);

        const controls = new OrbitControls(camera, css3dRenderer.domElement);
        controls.enableDamping = true;

        scene.add(new THREE.AmbientLight(0x334466, 1));
        const sun = new THREE.PointLight(0xfff5dd, 3, 3000);
        scene.add(sun);

        const planets = [
            { name: 'Mercury', radius: 25, distance: 180, color: 0xaaaaaa, speed: 1.6, mass: '3.3×10²³', temp: '430°C' },
            { name: 'Venus', radius: 40, distance: 280, color: 0xffcc88, speed: 1.1, mass: '4.9×10²⁴', temp: '465°C' },
            { name: 'Earth', radius: 45, distance: 400, color: 0x4488ff, speed: 0.8, mass: '5.9×10²⁴', temp: '15°C' },
            { name: 'Mars', radius: 30, distance: 530, color: 0xff6644, speed: 0.6, mass: '6.4×10²³', temp: '-60°C' },
        ];

        // Sun (WebGL)
        const sunMesh = new THREE.Mesh(
            new THREE.SphereGeometry(60, 32, 32),
            new THREE.MeshBasicMaterial({ color: 0xffdd55 })
        );
        scene.add(sunMesh);

        const planetObjects = [];
        planets.forEach(p => {
            const mesh = new THREE.Mesh(
                new THREE.SphereGeometry(p.radius, 32, 32),
                new THREE.MeshStandardMaterial({ color: p.color, roughness: 0.7 })
            );
            scene.add(mesh);

            // วง orbit
            const orbit = new THREE.Line(
                new THREE.BufferGeometry().setFromPoints(
                    Array.from({ length: 65 }, (_, i) => {
                        const a = (i / 64) * Math.PI * 2;
                        return new THREE.Vector3(Math.cos(a) * p.distance, 0, Math.sin(a) * p.distance);
                    })
                ),
                new THREE.LineBasicMaterial({ color: 0x334455, transparent: true, opacity: 0.4 })
            );
            scene.add(orbit);

            // Info panel (CSS3D)
            const infoEl = document.createElement('div');
            infoEl.className = 'planet-info';
            infoEl.innerHTML = `
                <div class="name">🪐 ${p.name}</div>
                <div class="stat"><span>Mass</span><span class="stat-val">${p.mass}</span></div>
                <div class="stat"><span>Temp</span><span class="stat-val">${p.temp}</span></div>
                <div class="stat"><span>Distance</span><span class="stat-val">${p.distance} AU</span></div>
            `;

            const infoObj = new CSS3DObject(infoEl);
            infoObj.scale.set(0.35, 0.35, 0.35);
            scene.add(infoObj);

            planetObjects.push({ mesh, infoObj, ...p });
        });

        function animate() {
            requestAnimationFrame(animate);
            const t = performance.now() * 0.001;

            planetObjects.forEach(p => {
                const angle = t * p.speed * 0.3;
                p.mesh.position.x = Math.cos(angle) * p.distance;
                p.mesh.position.z = Math.sin(angle) * p.distance;
                p.mesh.rotation.y = t;

                // Info panel ตามดาวเคราะห์
                p.infoObj.position.set(
                    p.mesh.position.x,
                    p.radius + 60,
                    p.mesh.position.z
                );
            });

            controls.update();
            webglRenderer.render(scene, camera);
            css3dRenderer.render(scene, camera);
        }
        animate();

        window.addEventListener('resize', () => {
            const w = window.innerWidth, h = window.innerHeight;
            camera.aspect = w / h;
            camera.updateProjectionMatrix();
            webglRenderer.setSize(w, h);
            css3dRenderer.setSize(w, h);
        });
    </script>
</body>
</html>
```

## Best Practices และข้อควรระวัง

### Performance Tips

```javascript
// 1. จำกัดจำนวน CSS3D objects
// CSS3D ใช้ DOM elements — มากเกินไปทำให้ช้า
// แนะนำไม่เกิน 50-100 elements

// 2. ใช้ will-change สำหรับ animated elements
element.style.willChange = 'transform';

// 3. หลีกเลี่ยง complex CSS ใน CSS3D objects
// opacity, filter, border-radius ราคาแพง GPU

// 4. ปิด pointer-events สำหรับ elements ที่ไม่ต้องการ
element.style.pointerEvents = 'none';

// 5. Scale มีผลต่อ text rendering
// scale เล็กเกินไปทำให้ตัวหนังสือเบลอ
// แก้ได้ด้วยการ scale HTML ให้ใหญ่แล้ว scale CSS3D ลง
const el = document.createElement('div');
el.style.width = '400px'; // HTML ใหญ่
el.style.fontSize = '32px'; // font ใหญ่
const obj = new CSS3DObject(el);
obj.scale.set(0.25, 0.25, 0.25); // แล้วค่อย scale ลง
```

### Common Pitfalls

```javascript
// ❌ ผิด: ไม่ set pointer-events บน container
css3dContainer.style.pointerEvents = 'auto'; // blocks WebGL interaction

// ✓ ถูก: container เป็น none, เปิดเฉพาะ elements ที่ต้องการ
css3dContainer.style.pointerEvents = 'none';
interactiveElement.style.pointerEvents = 'auto';

// ❌ ผิด: render แค่ CSS3D โดยไม่มี WebGL clear
// ทำให้เกิด ghosting
css3dRenderer.render(scene, camera); // ไม่มี webgl render

// ✓ ถูก: render ทั้งคู่ทุก frame
webglRenderer.render(scene, camera);
css3dRenderer.render(scene, camera);

// ❌ ผิด: ใช้ camera ต่างตัวกัน
// CSS3D และ WebGL ต้องใช้ camera เดียวกัน!
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Product Showcase
สร้าง 3D product showcase ที่มีการ์ด 4 ใบ วางเป็น grid 2x2 แต่ละการ์ดแสดงรูปสินค้า (ใช้ emoji แทน), ชื่อ, ราคา และปุ่ม "เพิ่มลงตะกร้า" เมื่อคลิกให้แสดง animation

### แบบฝึกหัดที่ 2: Interactive Timeline
สร้าง timeline แนวตั้งใน 3D space ที่มี event cards เชื่อมด้วยเส้น (WebGL Line) แต่ละ card เป็น HTML ที่คลิกได้เพื่อดูรายละเอียด

### แบบฝึกหัดที่ 3: Mixed Scene
สร้าง scene ที่มีทั้ง WebGL objects (terrain, lighting) และ CSS3D elements (สถิติ, labels) ที่ทำงานร่วมกันอย่างลงตัว

### แบบฝึกหัดที่ 4: Video in 3D
ใช้ CSS3DObject ห่อ `<video>` element แสดงวิดีโอ YouTube embed ใน 3D space ที่สามารถ interact ได้ปกติ

## สรุป

`CSS3DRenderer` เป็นเครื่องมือที่ทรงพลังสำหรับ:
- การสร้าง interactive UI ใน 3D space
- การแสดง HTML content (วิดีโอ, ฟอร์ม, rich text) ใน 3D
- การผสมผสาน web content กับ 3D graphics

กุญแจสำคัญคือการจัดการ pointer events ให้ถูกต้อง และการ render ทั้งสอง renderers ทุก animation frame ด้วย camera เดียวกัน

ในบทถัดไปเราจะเรียนรู้การสร้าง 3D UI Components library ที่สมบูรณ์
