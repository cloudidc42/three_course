# Part 18: 3D Model Loaders ใน Three.js

## บทนำ

การโหลดโมเดล 3D จากไฟล์ภายนอกเป็นสิ่งสำคัญในการพัฒนา web 3D Three.js รองรับหลายรูปแบบไฟล์ รูปแบบที่แนะนำที่สุดคือ **GLTF/GLB** เพราะเป็น open standard ที่รองรับ PBR materials, animations, และ skinning

## 1. ประเภทไฟล์ 3D

| Format | Extension | ข้อดี | ข้อเสีย |
|--------|-----------|-------|---------|
| GLTF/GLB | .gltf, .glb | เร็ว, PBR, animations | ใหม่กว่า |
| OBJ | .obj + .mtl | เป็น text, ง่าย | ไม่มี animations |
| FBX | .fbx | รองรับ animations | ขนาดใหญ่ |
| STL | .stl | 3D printing | ไม่มี materials |
| PLY | .ply | Point clouds | จำกัด |

---

## 2. GLTFLoader

GLTF (GL Transmission Format) เป็น "JPEG of 3D" - รูปแบบมาตรฐานสำหรับ web 3D

### การโหลดพื้นฐาน

```javascript
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

const loader = new GLTFLoader();

// โหลด GLTF
loader.load(
    'models/scene.gltf',  // URL
    
    // onLoad callback - สำเร็จ
    (gltf) => {
        console.log('โหลดสำเร็จ!', gltf);
        
        // gltf.scene เป็น THREE.Group ที่มีโมเดลอยู่
        scene.add(gltf.scene);
        
        // gltf.animations - AnimationClips
        console.log('Animations:', gltf.animations);
        
        // gltf.cameras - กล้องจากไฟล์
        console.log('Cameras:', gltf.cameras);
        
        // gltf.asset - metadata
        console.log('Asset:', gltf.asset);
    },
    
    // onProgress callback
    (progress) => {
        const percent = (progress.loaded / progress.total * 100).toFixed(1);
        console.log(`Loading: ${percent}%`);
    },
    
    // onError callback
    (error) => {
        console.error('โหลดล้มเหลว:', error);
    }
);
```

### GLTFLoader + Draco Compression

Draco เป็น codec ของ Google ที่บีบอัด geometry ได้ 10x หรือมากกว่า

```javascript
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js';

// ตั้งค่า DRACOLoader
const dracoLoader = new DRACOLoader();
// path ไปยัง Draco decoder (ต้อง copy ไว้ที่ server)
dracoLoader.setDecoderPath('/draco/');
// หรือใช้ CDN:
dracoLoader.setDecoderPath('https://www.gstatic.com/draco/versioned/decoders/1.5.6/');

// ตั้งค่า GLTFLoader ให้ใช้ Draco
const loader = new GLTFLoader();
loader.setDRACOLoader(dracoLoader);

loader.load('models/compressed.glb', (gltf) => {
    scene.add(gltf.scene);
});
```

### GLB vs GLTF

```
GLTF: ไฟล์ JSON + binary files แยกกัน
├── scene.gltf    (JSON descriptor)
├── mesh.bin      (binary geometry data)
└── texture.png   (textures)

GLB: ไฟล์เดียว รวมทุกอย่าง (แนะนำ)
└── scene.glb     (all in one binary file)
```

```javascript
// โหลด GLB (เหมือนกัน ต่างแค่ extension)
loader.load('models/scene.glb', (gltf) => {
    scene.add(gltf.scene);
});
```

---

## 3. Loading Manager

`THREE.LoadingManager` ช่วยจัดการ loading หลายไฟล์พร้อมกัน

```javascript
const manager = new THREE.LoadingManager();

// Callbacks
manager.onStart = (url, itemsLoaded, itemsTotal) => {
    console.log(`เริ่มโหลด ${url}`);
    console.log(`โหลดแล้ว: ${itemsLoaded} / ${itemsTotal}`);
};

manager.onProgress = (url, itemsLoaded, itemsTotal) => {
    const percent = Math.round(itemsLoaded / itemsTotal * 100);
    console.log(`Progress: ${percent}%`);
    document.getElementById('progress-bar').style.width = percent + '%';
    document.getElementById('progress-text').textContent = `Loading... ${percent}%`;
};

manager.onLoad = () => {
    console.log('โหลดทุกไฟล์เสร็จแล้ว!');
    // ซ่อน loading screen
    document.getElementById('loading-screen').style.display = 'none';
};

manager.onError = (url) => {
    console.error('โหลดล้มเหลว:', url);
};

// ใช้ manager กับ loaders
const gltfLoader = new GLTFLoader(manager);
const textureLoader = new THREE.TextureLoader(manager);
const audioLoader = new THREE.AudioLoader(manager);

// โหลดทุกอย่าง (manager จะติดตามทั้งหมด)
gltfLoader.load('models/character.glb', (gltf) => { /* ... */ });
gltfLoader.load('models/environment.glb', (gltf) => { /* ... */ });
textureLoader.load('textures/ground.jpg', (texture) => { /* ... */ });
```

### Loading Screen UI

```html
<!DOCTYPE html>
<html>
<head>
    <style>
        #loading-screen {
            position: fixed;
            top: 0; left: 0;
            width: 100%; height: 100%;
            background: #0a0a0a;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            z-index: 9999;
            transition: opacity 0.5s;
        }
        
        #loading-screen.hidden {
            opacity: 0;
            pointer-events: none;
        }
        
        .loading-logo {
            font-size: 48px;
            font-family: monospace;
            color: #e94560;
            margin-bottom: 30px;
            letter-spacing: 5px;
        }
        
        .progress-container {
            width: 300px;
            height: 4px;
            background: #333;
            border-radius: 2px;
            overflow: hidden;
        }
        
        .progress-bar {
            height: 100%;
            background: linear-gradient(90deg, #e94560, #4ecdc4);
            width: 0%;
            transition: width 0.3s ease;
        }
        
        .progress-text {
            color: #aaa;
            font-family: monospace;
            margin-top: 10px;
            font-size: 14px;
        }
        
        .loading-spinner {
            width: 40px; height: 40px;
            border: 3px solid #333;
            border-top: 3px solid #e94560;
            border-radius: 50%;
            animation: spin 1s linear infinite;
            margin-bottom: 20px;
        }
        
        @keyframes spin { to { transform: rotate(360deg); } }
    </style>
</head>
<body>

<div id="loading-screen">
    <div class="loading-spinner"></div>
    <div class="loading-logo">THREE.JS</div>
    <div class="progress-container">
        <div class="progress-bar" id="progress-bar"></div>
    </div>
    <div class="progress-text" id="progress-text">Initializing...</div>
</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { GLTFLoader } from 'https://unpkg.com/three@0.158.0/examples/jsm/loaders/GLTFLoader.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x1a1a2e);

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(0, 2, 5);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
document.body.appendChild(renderer.domElement);

// Lighting
scene.add(new THREE.AmbientLight(0xffffff, 0.5));
const dirLight = new THREE.DirectionalLight(0xffffff, 1);
dirLight.position.set(5, 10, 5);
scene.add(dirLight);

// Loading Manager
const manager = new THREE.LoadingManager();
const progressBar = document.getElementById('progress-bar');
const progressText = document.getElementById('progress-text');
const loadingScreen = document.getElementById('loading-screen');

manager.onProgress = (url, loaded, total) => {
    const pct = Math.round(loaded / total * 100);
    progressBar.style.width = pct + '%';
    const filename = url.split('/').pop();
    progressText.textContent = `Loading ${filename}... ${pct}%`;
};

manager.onLoad = () => {
    progressText.textContent = 'Complete!';
    setTimeout(() => {
        loadingScreen.classList.add('hidden');
        setTimeout(() => loadingScreen.remove(), 500);
    }, 500);
};

manager.onError = (url) => {
    progressText.textContent = `Error loading: ${url}`;
    progressText.style.color = '#e94560';
};

// Texture Loader
const textureLoader = new THREE.TextureLoader(manager);
const gltfLoader = new GLTFLoader(manager);

// โหลด textures (ตัวอย่าง)
const floorTexture = textureLoader.load(
    'https://threejs.org/examples/textures/hardwood2_diffuse.jpg',
    (texture) => {
        texture.wrapS = texture.wrapT = THREE.RepeatWrapping;
        texture.repeat.set(4, 4);
    }
);

// พื้น
const floor = new THREE.Mesh(
    new THREE.PlaneGeometry(20, 20),
    new THREE.MeshStandardMaterial({ map: floorTexture })
);
floor.rotation.x = -Math.PI / 2;
scene.add(floor);

// ถ้าไม่มีไฟล์โมเดล สร้าง placeholder
const placeholder = new THREE.Mesh(
    new THREE.BoxGeometry(1, 2, 1),
    new THREE.MeshStandardMaterial({ color: 0xe94560 })
);
placeholder.position.y = 1;
scene.add(placeholder);

// Animation
const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    placeholder.rotation.y = elapsed;
    renderer.render(scene, camera);
}
animate();
</script>
</body>
</html>
```

---

## 4. OBJLoader + MTLLoader

```javascript
import { OBJLoader } from 'three/examples/jsm/loaders/OBJLoader.js';
import { MTLLoader } from 'three/examples/jsm/loaders/MTLLoader.js';

// ต้องโหลด MTL ก่อน แล้วค่อยโหลด OBJ
const mtlLoader = new MTLLoader();
mtlLoader.setPath('models/');  // path ไปยัง materials

mtlLoader.load('model.mtl', (materials) => {
    materials.preload();  // โหลด textures ล่วงหน้า
    
    const objLoader = new OBJLoader();
    objLoader.setMaterials(materials);
    objLoader.setPath('models/');
    
    objLoader.load('model.obj', (object) => {
        // object เป็น THREE.Group
        object.traverse(child => {
            if (child.isMesh) {
                child.castShadow = true;
                child.receiveShadow = true;
            }
        });
        
        scene.add(object);
        
        // Center และ Scale
        const box = new THREE.Box3().setFromObject(object);
        const center = box.getCenter(new THREE.Vector3());
        const size = box.getSize(new THREE.Vector3());
        
        // Center model
        object.position.sub(center);
        
        // Normalize size
        const maxDim = Math.max(size.x, size.y, size.z);
        object.scale.multiplyScalar(5 / maxDim);
        
    }, undefined, (err) => console.error('OBJ Error:', err));
});
```

---

## 5. FBXLoader

```javascript
import { FBXLoader } from 'three/examples/jsm/loaders/FBXLoader.js';

const loader = new FBXLoader();

loader.load('models/character.fbx', (fbx) => {
    fbx.scale.setScalar(0.01); // FBX มักใหญ่มาก
    
    fbx.traverse(child => {
        if (child.isMesh) {
            child.castShadow = true;
        }
    });
    
    scene.add(fbx);
    
    // Animations
    if (fbx.animations.length > 0) {
        const mixer = new THREE.AnimationMixer(fbx);
        const action = mixer.clipAction(fbx.animations[0]);
        action.play();
        
        // ต้องอัพเดต mixer ใน render loop
        function animate() {
            const delta = clock.getDelta();
            mixer.update(delta);
            renderer.render(scene, camera);
        }
    }
});
```

---

## 6. STLLoader และ PLYLoader

```javascript
import { STLLoader } from 'three/examples/jsm/loaders/STLLoader.js';
import { PLYLoader } from 'three/examples/jsm/loaders/PLYLoader.js';

// STL - ใช้สำหรับ 3D printing / engineering models
const stlLoader = new STLLoader();
stlLoader.load('models/part.stl', (geometry) => {
    // STL โหลดเป็น BufferGeometry โดยตรง
    const material = new THREE.MeshPhongMaterial({
        color: 0x888888,
        shininess: 100
    });
    
    const mesh = new THREE.Mesh(geometry, material);
    
    // Compute normals ถ้าไม่มี
    geometry.computeVertexNormals();
    
    // Center
    geometry.computeBoundingBox();
    geometry.center();
    
    scene.add(mesh);
});

// PLY - Point clouds หรือ detailed meshes
const plyLoader = new PLYLoader();
plyLoader.load('models/cloud.ply', (geometry) => {
    geometry.computeVertexNormals();
    
    // ถ้ามี vertex colors
    if (geometry.attributes.color) {
        const material = new THREE.MeshStandardMaterial({
            vertexColors: true
        });
        scene.add(new THREE.Mesh(geometry, material));
    } else {
        // Point cloud
        const material = new THREE.PointsMaterial({
            size: 0.01,
            vertexColors: geometry.attributes.color !== undefined
        });
        scene.add(new THREE.Points(geometry, material));
    }
});
```

---

## 7. KTX2Loader - Compressed Textures

```javascript
import { KTX2Loader } from 'three/examples/jsm/loaders/KTX2Loader.js';

const ktx2Loader = new KTX2Loader();
ktx2Loader.setTranscoderPath('https://unpkg.com/three@0.158.0/examples/jsm/libs/basis/');
ktx2Loader.detectSupport(renderer);

// ใช้กับ GLTFLoader
gltfLoader.setKTX2Loader(ktx2Loader);

// หรือโหลดโดยตรง
ktx2Loader.load('textures/diffuse.ktx2', (texture) => {
    material.map = texture;
    material.needsUpdate = true;
});
```

---

## 8. Demo สมบูรณ์: Model Viewer

```html
<!DOCTYPE html>
<html>
<head>
    <title>GLTF Model Viewer</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #1a1a2e; overflow: hidden; font-family: monospace; }
        
        #loading {
            position: fixed;
            inset: 0;
            background: rgba(0,0,0,0.9);
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            color: white;
            z-index: 100;
            transition: opacity 0.5s;
        }
        
        #loading.hidden { opacity: 0; pointer-events: none; }
        
        #progress-wrap {
            width: 300px;
            height: 6px;
            background: #333;
            border-radius: 3px;
            margin: 15px 0;
            overflow: hidden;
        }
        
        #progress-fill {
            height: 100%;
            background: linear-gradient(90deg, #e94560, #ffd700);
            width: 0%;
            transition: width 0.2s;
        }
        
        #info {
            position: absolute;
            bottom: 15px;
            left: 50%;
            transform: translateX(-50%);
            color: rgba(255,255,255,0.5);
            font-size: 12px;
            text-align: center;
        }
        
        #model-info {
            position: absolute;
            top: 10px;
            left: 10px;
            background: rgba(0,0,0,0.7);
            color: white;
            padding: 12px;
            border-radius: 6px;
            font-size: 12px;
            display: none;
        }
        
        #controls-panel {
            position: absolute;
            top: 10px;
            right: 10px;
            background: rgba(0,0,0,0.7);
            color: white;
            padding: 12px;
            border-radius: 6px;
            font-size: 12px;
        }
        
        button {
            display: block;
            width: 100%;
            padding: 8px;
            margin: 4px 0;
            background: #e94560;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            font-size: 12px;
            font-family: monospace;
        }
        button:hover { background: #c73652; }
        
        .toggle-btn.active { background: #4ecdc4; color: #000; }
    </style>
</head>
<body>

<div id="loading">
    <div style="font-size: 24px; color: #e94560; margin-bottom: 10px;">Loading...</div>
    <div id="progress-wrap"><div id="progress-fill"></div></div>
    <div id="loading-text" style="color: #aaa; font-size: 13px;">Preparing...</div>
</div>

<div id="model-info">
    <div id="mesh-count">Meshes: -</div>
    <div id="vert-count">Vertices: -</div>
    <div id="anim-count">Animations: -</div>
</div>

<div id="controls-panel">
    <button id="btnWireframe" class="toggle-btn">Wireframe</button>
    <button id="btnSkeleton" class="toggle-btn">Skeleton</button>
    <button id="btnBBox" class="toggle-btn">BBox</button>
    <button id="btnAxes" class="toggle-btn">Axes</button>
    <hr style="border-color: #444; margin: 8px 0;">
    <div style="color: #aaa; margin-bottom: 4px;">Animations:</div>
    <div id="anim-buttons"></div>
</div>

<div id="info">OrbitControls: Drag=Rotate | Scroll=Zoom | Right-click=Pan</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';
import { GLTFLoader } from 'https://unpkg.com/three@0.158.0/examples/jsm/loaders/GLTFLoader.js';

// Scene
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x1a1a2e);

const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.01, 1000);
camera.position.set(0, 2, 5);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
renderer.outputColorSpace = THREE.SRGBColorSpace;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1;
document.body.appendChild(renderer.domElement);

// Controls
const orbitControls = new OrbitControls(camera, renderer.domElement);
orbitControls.enableDamping = true;
orbitControls.dampingFactor = 0.05;

// Lighting
const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
scene.add(ambientLight);

const dirLight1 = new THREE.DirectionalLight(0xffffff, 1);
dirLight1.position.set(5, 10, 7);
dirLight1.castShadow = true;
dirLight1.shadow.mapSize.width = 1024;
dirLight1.shadow.mapSize.height = 1024;
scene.add(dirLight1);

const dirLight2 = new THREE.DirectionalLight(0x4488ff, 0.5);
dirLight2.position.set(-5, 5, -5);
scene.add(dirLight2);

const hemiLight = new THREE.HemisphereLight(0xffeebb, 0x080820, 0.3);
scene.add(hemiLight);

// Grid
const grid = new THREE.GridHelper(20, 20, 0x334455, 0x223344);
scene.add(grid);

// Axes
const axesHelper = new THREE.AxesHelper(3);
axesHelper.visible = false;
scene.add(axesHelper);

// Loading Manager
const progressFill = document.getElementById('progress-fill');
const loadingText = document.getElementById('loading-text');
const loadingDiv = document.getElementById('loading');

const manager = new THREE.LoadingManager();
manager.onProgress = (url, loaded, total) => {
    const pct = Math.round(loaded / total * 100);
    progressFill.style.width = pct + '%';
    loadingText.textContent = url.split('/').pop();
};
manager.onLoad = () => {
    setTimeout(() => {
        loadingDiv.classList.add('hidden');
        document.getElementById('model-info').style.display = 'block';
    }, 300);
};

const loader = new GLTFLoader(manager);

// ตัวแปร global
let mixer = null;
let currentModel = null;
const wireframeMeshes = [];
let boxHelper = null;
let skeletonHelper = null;

function loadModel(url) {
    // ลบโมเดลเก่า
    if (currentModel) {
        scene.remove(currentModel);
        currentModel.traverse(obj => {
            if (obj.geometry) obj.geometry.dispose();
            if (obj.material) {
                const mats = Array.isArray(obj.material) ? obj.material : [obj.material];
                mats.forEach(m => m.dispose());
            }
        });
    }
    if (mixer) { mixer.stopAllAction(); mixer = null; }
    if (boxHelper) { scene.remove(boxHelper); boxHelper = null; }
    if (skeletonHelper) { scene.remove(skeletonHelper); skeletonHelper = null; }
    
    // Show loading
    progressFill.style.width = '0%';
    loadingDiv.classList.remove('hidden');
    
    loader.load(url, (gltf) => {
        const model = gltf.scene;
        currentModel = model;
        
        // Auto scale and center
        const box = new THREE.Box3().setFromObject(model);
        const size = box.getSize(new THREE.Vector3());
        const center = box.getCenter(new THREE.Vector3());
        
        const maxDim = Math.max(size.x, size.y, size.z);
        model.scale.multiplyScalar(4 / maxDim);
        
        box.setFromObject(model);
        box.getCenter(center);
        model.position.sub(center);
        model.position.y += size.y * 4 / maxDim / 2;
        
        // Shadows
        model.traverse(child => {
            if (child.isMesh) {
                child.castShadow = true;
                child.receiveShadow = true;
            }
        });
        
        scene.add(model);
        
        // Update orbit target
        orbitControls.target.set(0, size.y * 4 / maxDim / 2, 0);
        camera.position.set(0, size.y * 4 / maxDim, maxDim * 4);
        orbitControls.update();
        
        // Model info
        let meshCount = 0, vertCount = 0;
        model.traverse(obj => {
            if (obj.isMesh) {
                meshCount++;
                vertCount += obj.geometry.attributes.position.count;
            }
        });
        
        document.getElementById('mesh-count').textContent = `Meshes: ${meshCount}`;
        document.getElementById('vert-count').textContent = `Vertices: ${vertCount.toLocaleString()}`;
        document.getElementById('anim-count').textContent = `Animations: ${gltf.animations.length}`;
        
        // Animations
        if (gltf.animations.length > 0) {
            mixer = new THREE.AnimationMixer(model);
            const animBtns = document.getElementById('anim-buttons');
            animBtns.innerHTML = '';
            
            gltf.animations.forEach((clip, i) => {
                const btn = document.createElement('button');
                btn.textContent = clip.name || `Anim ${i}`;
                btn.style.background = '#333';
                btn.addEventListener('click', () => {
                    mixer.stopAllAction();
                    const action = mixer.clipAction(clip);
                    action.play();
                    document.querySelectorAll('#anim-buttons button').forEach(b => {
                        b.style.background = '#333';
                    });
                    btn.style.background = '#4ecdc4';
                    btn.style.color = '#000';
                });
                animBtns.appendChild(btn);
            });
            
            // Auto-play first animation
            mixer.clipAction(gltf.animations[0]).play();
        }
        
        // Skeleton
        model.traverse(child => {
            if (child.isSkinnedMesh) {
                skeletonHelper = new THREE.SkeletonHelper(child);
                skeletonHelper.visible = false;
                scene.add(skeletonHelper);
            }
        });
        
    }, undefined, (err) => {
        console.error('Load error:', err);
        loadingText.textContent = 'Error loading model';
        loadingText.style.color = '#e94560';
    });
}

// Default model
loadModel('https://threejs.org/examples/models/gltf/Flamingo.glb');

// Control buttons
let wireframeOn = false;
document.getElementById('btnWireframe').addEventListener('click', (e) => {
    wireframeOn = !wireframeOn;
    e.target.classList.toggle('active', wireframeOn);
    if (currentModel) {
        currentModel.traverse(obj => {
            if (obj.isMesh) {
                const mats = Array.isArray(obj.material) ? obj.material : [obj.material];
                mats.forEach(m => m.wireframe = wireframeOn);
            }
        });
    }
});

document.getElementById('btnSkeleton').addEventListener('click', (e) => {
    if (skeletonHelper) {
        skeletonHelper.visible = !skeletonHelper.visible;
        e.target.classList.toggle('active', skeletonHelper.visible);
    }
});

document.getElementById('btnBBox').addEventListener('click', (e) => {
    if (currentModel) {
        if (boxHelper) {
            scene.remove(boxHelper);
            boxHelper = null;
            e.target.classList.remove('active');
        } else {
            boxHelper = new THREE.BoxHelper(currentModel, 0x00ff00);
            scene.add(boxHelper);
            e.target.classList.add('active');
        }
    }
});

document.getElementById('btnAxes').addEventListener('click', (e) => {
    axesHelper.visible = !axesHelper.visible;
    e.target.classList.toggle('active', axesHelper.visible);
});

window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});

// Animation
const clock = new THREE.Clock();
function animate() {
    requestAnimationFrame(animate);
    const delta = clock.getDelta();
    
    if (mixer) mixer.update(delta);
    if (boxHelper && currentModel) boxHelper.update();
    
    orbitControls.update();
    renderer.render(scene, camera);
}
animate();
</script>
</body>
</html>
```

---

## 9. GLTF Animations

```javascript
// การจัดการ Animations จาก GLTF
loader.load('model.glb', (gltf) => {
    const model = gltf.scene;
    scene.add(model);
    
    // AnimationMixer จัดการ animations
    const mixer = new THREE.AnimationMixer(model);
    
    // ดู animations ที่มี
    console.log('Animations:', gltf.animations.map(a => a.name));
    
    // เล่น animation ชื่อ "Walk"
    const walkClip = THREE.AnimationClip.findByName(gltf.animations, 'Walk');
    if (walkClip) {
        const walkAction = mixer.clipAction(walkClip);
        walkAction.play();
        
        // Loop mode
        walkAction.setLoop(THREE.LoopRepeat, Infinity); // ซ้ำตลอด
        // walkAction.setLoop(THREE.LoopOnce, 1);      // เล่นครั้งเดียว
        // walkAction.setLoop(THREE.LoopPingPong, 5);  // ไป-กลับ 5 ครั้ง
        
        walkAction.clampWhenFinished = true;  // หยุดที่ frame สุดท้าย
        walkAction.timeScale = 1.0;            // ความเร็ว (1 = ปกติ, 2 = เร็ว x2)
    }
    
    // Blend animations
    const runClip = THREE.AnimationClip.findByName(gltf.animations, 'Run');
    if (runClip) {
        const runAction = mixer.clipAction(runClip);
        runAction.weight = 0;  // เริ่มที่ 0
        runAction.play();
        
        // Crossfade จาก walk ไป run ใน 0.5 วินาที
        function blendToRun() {
            walkAction.crossFadeTo(runAction, 0.5, true);
        }
    }
    
    // อัพเดตใน loop
    const clock = new THREE.Clock();
    function animate() {
        const delta = clock.getDelta();
        mixer.update(delta);  // สำคัญมาก!
        renderer.render(scene, camera);
    }
    
    // Events
    mixer.addEventListener('finished', (e) => {
        console.log('Animation finished:', e.action.getClip().name);
    });
    
    mixer.addEventListener('loop', (e) => {
        console.log('Animation looped:', e.action.getClip().name);
    });
});
```

---

## 10. Model Optimization

```javascript
// Optimize loaded model
function optimizeModel(model) {
    model.traverse(obj => {
        if (obj.isMesh) {
            // Merge vertices
            obj.geometry = mergeVertices(obj.geometry);
            
            // Compute normals ถ้าไม่มี
            if (!obj.geometry.attributes.normal) {
                obj.geometry.computeVertexNormals();
            }
            
            // Enable frustum culling (default true)
            obj.frustumCulled = true;
            
            // ลด draw calls ด้วย material merging
            obj.material.side = THREE.FrontSide; // อย่าใช้ DoubleSide ถ้าไม่จำเป็น
            
            // Shadow optimization
            obj.castShadow = true;
            obj.receiveShadow = true;
        }
    });
}

// Cache models ที่ใช้ซ้ำ
const modelCache = new Map();

async function loadModelCached(url) {
    if (modelCache.has(url)) {
        return modelCache.get(url).clone();
    }
    
    return new Promise((resolve, reject) => {
        loader.load(url, (gltf) => {
            modelCache.set(url, gltf.scene);
            resolve(gltf.scene.clone());
        }, undefined, reject);
    });
}
```

---

## แบบฝึกหัด

1. **Model Gallery**: สร้างหน้า gallery ที่แสดงโมเดล GLTF หลายตัวให้เลือก
2. **Animation Blender**: สร้าง UI สลับ animations และปรับ weight ได้
3. **LOD System**: โหลดโมเดลหลาย LOD levels และสลับตามระยะ
4. **Multi-format Viewer**: รองรับทั้ง GLTF, OBJ, FBX ในตัวเดียว
5. **Progress Bar**: สร้าง loading screen ที่สวยงามพร้อม progress bar

---

## สรุป

- **GLTF/GLB** คือ standard format ที่แนะนำ รองรับ PBR, animations, skinning
- **Draco** compression ลดขนาดได้มากถึง 10x
- **LoadingManager** จัดการ multiple files และแสดง progress
- **AnimationMixer** จัดการ animations จาก GLTF/FBX
- ต้อง auto-scale และ center โมเดลที่โหลดมาเสมอ
- Dispose geometry/material เมื่อไม่ใช้แล้ว
- Cache โมเดลที่ใช้ซ้ำเพื่อ performance
