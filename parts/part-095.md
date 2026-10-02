# Part 95: Labels, Annotations & HUD

## บทนำ

ระบบ label และ annotation เป็นส่วนสำคัญของ 3D applications ไม่ว่าจะเป็น archviz, game, หรือ data visualization ในบทนี้เราจะเรียนรู้ CSS2DRenderer ซึ่งเหมาะกว่า CSS3DRenderer สำหรับ labels, การตรวจสอบ occlusion, ระบบ annotation และการสร้าง HUD แบบสมบูรณ์

## CSS2DRenderer คืออะไร

`CSS2DRenderer` คือ renderer ที่แปลงตำแหน่ง 3D ของ object เป็น 2D screen position แล้ว render HTML elements ณ ตำแหน่งนั้น

**ข้อดีของ CSS2DRenderer:**
- Labels อยู่ "บนสุด" เสมอ (ไม่ถูก occlude ด้วย geometry)
- ง่ายต่อการสร้าง text labels ที่คมชัด
- รองรับ HTML/CSS เต็มรูปแบบ
- Performance ดีกว่า CSS3DRenderer สำหรับ flat labels

**ข้อเสีย:**
- ไม่มี depth — อยู่บน 2D screen
- ไม่หมุนตาม object
- ต้องจัดการ visibility เอง (ถ้าต้องการซ่อนเมื่อถูกบัง)

## การติดตั้ง CSS2DRenderer

```javascript
import { CSS2DRenderer, CSS2DObject } from 'three/examples/jsm/renderers/CSS2DRenderer.js';

const labelRenderer = new CSS2DRenderer();
labelRenderer.setSize(window.innerWidth, window.innerHeight);
labelRenderer.domElement.style.position = 'absolute';
labelRenderer.domElement.style.top = '0';
labelRenderer.domElement.style.pointerEvents = 'none'; // สำคัญมาก!
document.body.appendChild(labelRenderer.domElement);

// ใน animate loop:
labelRenderer.render(scene, camera);
```

## CSS2DObject พื้นฐาน

```javascript
// สร้าง HTML element
const labelDiv = document.createElement('div');
labelDiv.className = 'label';
labelDiv.textContent = 'Hello World';
labelDiv.style.cssText = `
    background: rgba(0,0,0,0.7);
    color: white;
    padding: 4px 8px;
    border-radius: 4px;
    font-size: 12px;
    white-space: nowrap;
    pointer-events: none;
`;

// ห่อด้วย CSS2DObject
const label = new CSS2DObject(labelDiv);

// ผูกกับ mesh
mesh.add(label);

// หรือวางตำแหน่งเอง
label.position.set(0, 1, 0); // relative to parent
```

## Label Occlusion — การซ่อน Labels เมื่อถูกบัง

CSS2DRenderer ไม่ตรวจสอบ occlusion อัตโนมัติ เราต้องทำเอง:

```javascript
function isOccluded(object, camera, scene) {
    const direction = new THREE.Vector3();
    direction.subVectors(object.position, camera.position);
    const distance = direction.length();
    direction.normalize();
    
    const raycaster = new THREE.Raycaster(camera.position, direction);
    const intersects = raycaster.intersectObjects(scene.children, true);
    
    if (intersects.length === 0) return false;
    
    // ถ้า intersection แรกใกล้กว่า object = occluded
    return intersects[0].distance < distance - 0.1;
}

// ใน animate loop:
function updateLabels() {
    labels.forEach(({ object, labelObj }) => {
        const occluded = isOccluded(object, camera, scene);
        labelObj.element.style.display = occluded ? 'none' : 'block';
        labelObj.element.style.opacity = occluded ? '0' : '1';
    });
}
```

## Distance-Based Label Visibility

แสดง/ซ่อน labels ตามระยะห่างจาก camera:

```javascript
function updateLabelVisibility(labels, camera) {
    labels.forEach(({ object, labelObj, minDist = 0, maxDist = 500 }) => {
        const dist = camera.position.distanceTo(object.position);
        
        if (dist < minDist || dist > maxDist) {
            labelObj.element.style.display = 'none';
            return;
        }
        
        labelObj.element.style.display = 'block';
        
        // Fade ตามระยะ
        const fadeStart = maxDist * 0.7;
        if (dist > fadeStart) {
            const opacity = 1 - (dist - fadeStart) / (maxDist - fadeStart);
            labelObj.element.style.opacity = opacity.toString();
        } else {
            labelObj.element.style.opacity = '1';
        }
        
        // Scale ตามระยะ (optional)
        const scale = Math.max(0.5, 1 - dist / maxDist);
        labelObj.element.style.transform = `scale(${scale})`;
    });
}
```

## Annotation System

ระบบ annotation ที่สมบูรณ์: คลิก object เพื่อแสดง info panel

```javascript
class AnnotationSystem {
    constructor(scene, camera, renderer, labelRenderer) {
        this.scene = scene;
        this.camera = camera;
        this.renderer = renderer;
        this.labelRenderer = labelRenderer;
        
        this.annotations = new Map(); // object -> annotation data
        this.activeAnnotation = null;
        
        this.raycaster = new THREE.Raycaster();
        this.mouse = new THREE.Vector2();
        
        this.setupEventListeners();
    }
    
    addAnnotation(object, data) {
        // สร้าง marker
        const markerEl = document.createElement('div');
        markerEl.className = 'annotation-marker';
        markerEl.innerHTML = `<div class="marker-dot"></div><div class="marker-ring"></div>`;
        markerEl.addEventListener('click', (e) => {
            e.stopPropagation();
            this.showAnnotation(object);
        });
        
        const markerObj = new CSS2DObject(markerEl);
        markerObj.position.copy(data.offset || new THREE.Vector3(0, 0, 0));
        object.add(markerObj);
        
        // สร้าง info panel (ซ่อนไว้ก่อน)
        const panelEl = document.createElement('div');
        panelEl.className = 'annotation-panel';
        panelEl.style.display = 'none';
        panelEl.innerHTML = `
            <div class="panel-header">
                <span class="panel-title">${data.title}</span>
                <button class="panel-close">✕</button>
            </div>
            <div class="panel-body">${data.content}</div>
        `;
        panelEl.querySelector('.panel-close').addEventListener('click', () => {
            this.hideAnnotation();
        });
        
        const panelObj = new CSS2DObject(panelEl);
        panelObj.position.copy(data.panelOffset || new THREE.Vector3(0, 30, 0));
        object.add(panelObj);
        
        this.annotations.set(object, {
            markerEl, markerObj,
            panelEl, panelObj,
            data,
        });
        
        return this;
    }
    
    showAnnotation(object) {
        // ซ่อน annotation ที่แสดงอยู่
        if (this.activeAnnotation) {
            this.hideAnnotation();
        }
        
        const ann = this.annotations.get(object);
        if (!ann) return;
        
        ann.panelEl.style.display = 'block';
        ann.markerEl.classList.add('active');
        this.activeAnnotation = object;
        
        // Highlight object
        if (object.material) {
            object.material.emissive = new THREE.Color(0x224422);
        }
    }
    
    hideAnnotation() {
        if (!this.activeAnnotation) return;
        
        const ann = this.annotations.get(this.activeAnnotation);
        if (ann) {
            ann.panelEl.style.display = 'none';
            ann.markerEl.classList.remove('active');
            if (this.activeAnnotation.material) {
                this.activeAnnotation.material.emissive = new THREE.Color(0x000000);
            }
        }
        
        this.activeAnnotation = null;
    }
    
    setupEventListeners() {
        window.addEventListener('mousemove', (e) => {
            this.mouse.x = (e.clientX / window.innerWidth) * 2 - 1;
            this.mouse.y = -(e.clientY / window.innerHeight) * 2 + 1;
        });
        
        window.addEventListener('click', (e) => {
            this.raycaster.setFromCamera(this.mouse, this.camera);
            const annotatedObjects = Array.from(this.annotations.keys());
            const hits = this.raycaster.intersectObjects(annotatedObjects, true);
            
            if (hits.length > 0) {
                const clickedObj = hits[0].object;
                const annotatedParent = annotatedObjects.find(o => 
                    o === clickedObj || o.getObjectById(clickedObj.id)
                );
                if (annotatedParent) {
                    this.showAnnotation(annotatedParent);
                }
            } else {
                this.hideAnnotation();
            }
        });
    }
    
    update() {
        // อัพเดต visibility ตาม distance
        this.annotations.forEach((ann, object) => {
            const dist = this.camera.position.distanceTo(object.position);
            const visible = dist < 500;
            ann.markerEl.style.opacity = visible ? '1' : '0';
        });
    }
}
```

## Location Pin System

ระบบ pin แบบ Google Maps ใน 3D:

```javascript
class LocationPinSystem {
    constructor(scene, camera, labelRenderer) {
        this.scene = scene;
        this.camera = camera;
        this.labelRenderer = labelRenderer;
        this.pins = [];
    }
    
    addPin(position, data) {
        const pinEl = document.createElement('div');
        pinEl.className = 'location-pin';
        pinEl.style.cssText = `
            display: flex;
            flex-direction: column;
            align-items: center;
            cursor: pointer;
            pointer-events: auto;
        `;
        
        const category = data.category || 'default';
        const colors = {
            restaurant: '#ff6644',
            hotel: '#4488ff',
            attraction: '#44cc44',
            default: '#aa44ff',
        };
        const color = colors[category] || colors.default;
        
        pinEl.innerHTML = `
            <div class="pin-head" style="
                width: 32px; height: 32px;
                background: ${color};
                border: 3px solid white;
                border-radius: 50% 50% 50% 0;
                transform: rotate(-45deg);
                display: flex; align-items: center; justify-content: center;
                box-shadow: 0 2px 8px rgba(0,0,0,0.5);
                transition: transform 0.2s;
            ">
                <span style="transform: rotate(45deg); font-size: 14px;">${data.icon || '📍'}</span>
            </div>
            <div class="pin-label" style="
                background: rgba(0,0,0,0.8);
                color: white;
                padding: 4px 8px;
                border-radius: 4px;
                font-size: 11px;
                white-space: nowrap;
                margin-top: 4px;
                opacity: 0;
                transition: opacity 0.2s;
            ">${data.name}</div>
        `;
        
        const pinLabel = pinEl.querySelector('.pin-label');
        const pinHead = pinEl.querySelector('.pin-head');
        
        pinEl.addEventListener('mouseenter', () => {
            pinLabel.style.opacity = '1';
            pinHead.style.transform = 'rotate(-45deg) scale(1.3)';
        });
        pinEl.addEventListener('mouseleave', () => {
            pinLabel.style.opacity = '0';
            pinHead.style.transform = 'rotate(-45deg) scale(1)';
        });
        pinEl.addEventListener('click', () => {
            data.onClick && data.onClick(data);
        });
        
        const pinObj = new CSS2DObject(pinEl);
        pinObj.position.copy(position);
        this.scene.add(pinObj);
        
        this.pins.push({ pinObj, pinEl, data, position });
        return pinObj;
    }
    
    update() {
        this.pins.forEach(pin => {
            const dist = this.camera.position.distanceTo(pin.position);
            const visible = dist > 20 && dist < 800;
            pin.pinEl.style.display = visible ? 'flex' : 'none';
            
            // Scale ตามระยะ
            const scale = Math.max(0.5, Math.min(1.5, 200 / dist));
            pin.pinEl.style.transform = `scale(${scale})`;
        });
    }
}
```

## Health Bar Above Character

```javascript
class CharacterHealthBar {
    constructor(character, maxHealth) {
        this.character = character;
        this.maxHealth = maxHealth;
        this._health = maxHealth;
        this.createBar();
    }
    
    createBar() {
        const container = document.createElement('div');
        container.style.cssText = `
            display: flex;
            flex-direction: column;
            align-items: center;
            pointer-events: none;
            gap: 2px;
        `;
        
        this.nameEl = document.createElement('div');
        this.nameEl.style.cssText = `
            color: white;
            font-size: 10px;
            font-family: Arial;
            text-shadow: 0 1px 2px rgba(0,0,0,0.8);
            white-space: nowrap;
        `;
        
        const barContainer = document.createElement('div');
        barContainer.style.cssText = `
            width: 64px; height: 6px;
            background: rgba(0,0,0,0.5);
            border-radius: 3px;
            overflow: hidden;
            border: 1px solid rgba(255,255,255,0.3);
        `;
        
        this.barFill = document.createElement('div');
        this.barFill.style.cssText = `
            height: 100%;
            background: #44ff44;
            border-radius: 3px;
            transition: width 0.3s, background-color 0.3s;
        `;
        this.barFill.style.width = '100%';
        
        barContainer.appendChild(this.barFill);
        container.appendChild(this.nameEl);
        container.appendChild(barContainer);
        
        this.labelObj = new CSS2DObject(container);
        this.labelObj.position.set(0, 2.5, 0); // เหนือหัวตัวละคร
        this.character.add(this.labelObj);
    }
    
    setName(name) {
        this.nameEl.textContent = name;
    }
    
    setHealth(health) {
        this._health = Math.max(0, Math.min(this.maxHealth, health));
        const percent = this._health / this.maxHealth;
        
        this.barFill.style.width = `${percent * 100}%`;
        
        // เปลี่ยนสีตาม health
        if (percent > 0.6) {
            this.barFill.style.background = '#44ff44';
        } else if (percent > 0.3) {
            this.barFill.style.background = '#ffaa22';
        } else {
            this.barFill.style.background = '#ff2222';
        }
    }
    
    damage(amount) {
        this.setHealth(this._health - amount);
        this.showDamageNumber(amount);
    }
    
    showDamageNumber(amount) {
        const dmgEl = document.createElement('div');
        dmgEl.style.cssText = `
            color: #ff4444;
            font-size: 14px;
            font-weight: bold;
            font-family: Arial;
            text-shadow: 0 0 4px rgba(255,0,0,0.8);
            pointer-events: none;
            animation: floatUp 1s ease-out forwards;
        `;
        dmgEl.textContent = `-${amount}`;
        
        const dmgObj = new CSS2DObject(dmgEl);
        dmgObj.position.set(
            (Math.random() - 0.5) * 0.5,
            3 + Math.random() * 0.5,
            0
        );
        this.character.add(dmgObj);
        
        setTimeout(() => {
            this.character.remove(dmgObj);
        }, 1100);
    }
}
```

## Compass Rose

```javascript
class CompassRose3D {
    constructor(scene, camera) {
        this.scene = scene;
        this.camera = camera;
        this.createCompass();
    }
    
    createCompass() {
        const container = document.createElement('div');
        container.style.cssText = `
            width: 80px; height: 80px;
            position: relative;
        `;
        
        this.svg = document.createElementNS('http://www.w3.org/2000/svg', 'svg');
        this.svg.setAttribute('viewBox', '0 0 80 80');
        this.svg.style.cssText = 'width: 80px; height: 80px;';
        
        this.svg.innerHTML = `
            <circle cx="40" cy="40" r="38" fill="rgba(0,0,20,0.7)" stroke="rgba(100,120,255,0.5)" stroke-width="1"/>
            <g id="compass-needle">
                <!-- North arrow (red) -->
                <polygon points="40,8 44,40 40,36 36,40" fill="#ff3333"/>
                <!-- South arrow (white) -->
                <polygon points="40,72 44,40 40,44 36,40" fill="rgba(255,255,255,0.6)"/>
            </g>
            <text x="40" y="6" text-anchor="middle" fill="#ff6666" font-size="8" font-family="Arial" font-weight="bold">N</text>
            <text x="40" y="78" text-anchor="middle" fill="white" font-size="8" font-family="Arial">S</text>
            <text x="74" y="43" text-anchor="middle" fill="white" font-size="8" font-family="Arial">E</text>
            <text x="6" y="43" text-anchor="middle" fill="white" font-size="8" font-family="Arial">W</text>
            <circle cx="40" cy="40" r="3" fill="rgba(255,255,255,0.8)"/>
        `;
        
        container.appendChild(this.svg);
        this.needle = this.svg.getElementById('compass-needle');
        
        // วางตำแหน่งที่มุมขวาบน (ใช้ CSS2D หรือ CSS สตาติก)
        container.style.position = 'fixed';
        container.style.top = '16px';
        container.style.right = '16px';
        container.style.zIndex = '100';
        container.style.pointerEvents = 'none';
        document.body.appendChild(container);
        
        this.container = container;
    }
    
    update() {
        // คำนวณทิศทาง camera
        const dir = new THREE.Vector3();
        this.camera.getWorldDirection(dir);
        
        // มุมบน XZ plane
        const angle = Math.atan2(dir.x, dir.z);
        
        // หมุน needle ตามทิศทาง
        this.needle.setAttribute('transform', `rotate(${angle * 180 / Math.PI}, 40, 40)`);
    }
    
    destroy() {
        document.body.removeChild(this.container);
    }
}
```

## Demo หลัก: Complete Annotation System สำหรับ ArchViz

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ArchViz Annotation System</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #1a1a2e; overflow: hidden; font-family: 'Segoe UI', Arial, sans-serif; }
        canvas { display: block; }
        
        #label-renderer { position: absolute; top: 0; left: 0; pointer-events: none; }
        
        /* Annotation Markers */
        .ann-marker {
            width: 24px; height: 24px;
            display: flex; align-items: center; justify-content: center;
            cursor: pointer;
            pointer-events: auto;
            transition: transform 0.2s;
        }
        .ann-marker .dot {
            width: 12px; height: 12px;
            background: #4488ff;
            border: 2px solid white;
            border-radius: 50%;
            box-shadow: 0 0 0 0 rgba(68,136,255,0.7);
            animation: pulse 2s infinite;
        }
        .ann-marker.active .dot { background: #ff8844; }
        .ann-marker:hover { transform: scale(1.3); }
        
        @keyframes pulse {
            0% { box-shadow: 0 0 0 0 rgba(68,136,255,0.7); }
            70% { box-shadow: 0 0 0 8px rgba(68,136,255,0); }
            100% { box-shadow: 0 0 0 0 rgba(68,136,255,0); }
        }
        
        /* Annotation Panel */
        .ann-panel {
            width: 220px;
            background: rgba(10, 10, 30, 0.95);
            border: 1px solid rgba(68, 136, 255, 0.4);
            border-radius: 10px;
            overflow: hidden;
            box-shadow: 0 8px 32px rgba(0,0,0,0.6);
            pointer-events: auto;
            transform: translateX(20px);
            transition: opacity 0.2s, transform 0.2s;
        }
        .ann-panel.hidden { opacity: 0; transform: translateX(30px); pointer-events: none; }
        
        .ann-panel-header {
            background: rgba(68, 136, 255, 0.2);
            padding: 10px 14px;
            display: flex;
            justify-content: space-between;
            align-items: center;
            border-bottom: 1px solid rgba(68, 136, 255, 0.2);
        }
        .ann-panel-title {
            color: #88aaff;
            font-weight: bold;
            font-size: 13px;
        }
        .ann-panel-close {
            background: none; border: none;
            color: rgba(255,255,255,0.5);
            cursor: pointer; font-size: 16px;
            padding: 0 4px;
            pointer-events: auto;
        }
        .ann-panel-close:hover { color: white; }
        .ann-panel-body {
            padding: 12px 14px;
            color: rgba(255,255,255,0.75);
            font-size: 12px;
            line-height: 1.7;
        }
        .ann-panel-row {
            display: flex;
            justify-content: space-between;
            margin: 4px 0;
            padding: 3px 0;
            border-bottom: 1px solid rgba(255,255,255,0.05);
        }
        .ann-panel-row .label { color: rgba(255,255,255,0.4); font-size: 11px; }
        .ann-panel-row .value { color: #aaccff; font-size: 11px; }
        
        /* HUD */
        #hud {
            position: fixed;
            top: 0; left: 0;
            width: 100%; height: 100%;
            pointer-events: none;
            z-index: 50;
        }
        .hud-info {
            position: absolute;
            bottom: 16px; left: 16px;
            color: rgba(255,255,255,0.5);
            font-size: 13px;
        }
        .hud-compass {
            position: absolute;
            top: 16px; right: 16px;
            width: 72px; height: 72px;
        }
        
        /* Distance labels */
        .dist-label {
            background: rgba(0,0,0,0.6);
            color: rgba(255,255,255,0.7);
            padding: 2px 6px;
            border-radius: 3px;
            font-size: 10px;
            white-space: nowrap;
            pointer-events: none;
        }
    </style>
</head>
<body>
    <canvas id="canvas"></canvas>
    <div id="hud">
        <div class="hud-info">คลิกที่ ● สีน้ำเงินเพื่อดูข้อมูล | ลากเพื่อหมุน</div>
        <svg class="hud-compass" id="compass" viewBox="0 0 72 72">
            <circle cx="36" cy="36" r="34" fill="rgba(0,0,20,0.7)" stroke="rgba(100,120,255,0.4)" stroke-width="1"/>
            <g id="needle">
                <polygon points="36,6 39,36 36,30 33,36" fill="#ff3333"/>
                <polygon points="36,66 39,36 36,42 33,36" fill="rgba(255,255,255,0.5)"/>
            </g>
            <text x="36" y="4" text-anchor="middle" fill="#ff6666" font-size="7" font-family="Arial" font-weight="bold">N</text>
            <text x="36" y="72" text-anchor="middle" fill="white" font-size="7" font-family="Arial">S</text>
            <text x="68" y="39" text-anchor="middle" fill="white" font-size="7" font-family="Arial">E</text>
            <text x="4" y="39" text-anchor="middle" fill="white" font-size="7" font-family="Arial">W</text>
        </svg>
    </div>

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
        import { CSS2DRenderer, CSS2DObject } from 'three/examples/jsm/renderers/CSS2DRenderer.js';
        import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

        const W = window.innerWidth, H = window.innerHeight;
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x1a1a2e);
        scene.fog = new THREE.FogExp2(0x1a1a2e, 0.003);
        
        const camera = new THREE.PerspectiveCamera(60, W / H, 0.1, 1000);
        camera.position.set(15, 12, 25);
        
        const renderer = new THREE.WebGLRenderer({ canvas: document.getElementById('canvas'), antialias: true });
        renderer.setSize(W, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        renderer.shadowMap.enabled = true;
        renderer.shadowMap.type = THREE.PCFSoftShadowMap;
        
        // CSS2D Renderer
        const labelRenderer = new CSS2DRenderer();
        labelRenderer.setSize(W, H);
        labelRenderer.domElement.id = 'label-renderer';
        document.body.appendChild(labelRenderer.domElement);
        
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.target.set(0, 3, 0);
        controls.maxPolarAngle = Math.PI * 0.85;
        
        // Lighting
        scene.add(new THREE.AmbientLight(0x334466, 1.2));
        const sun = new THREE.DirectionalLight(0xfff8e7, 2.5);
        sun.position.set(30, 50, 20);
        sun.castShadow = true;
        sun.shadow.mapSize.setScalar(2048);
        sun.shadow.camera.near = 0.5;
        sun.shadow.camera.far = 200;
        sun.shadow.camera.left = -40;
        sun.shadow.camera.right = 40;
        sun.shadow.camera.top = 40;
        sun.shadow.camera.bottom = -40;
        scene.add(sun);
        
        // Ground
        const ground = new THREE.Mesh(
            new THREE.PlaneGeometry(100, 100),
            new THREE.MeshStandardMaterial({ color: 0x2a4a2a, roughness: 0.9 })
        );
        ground.rotation.x = -Math.PI / 2;
        ground.receiveShadow = true;
        scene.add(ground);
        
        // Buildings
        const buildings = [
            { pos: [-8, 0, -5], size: [5, 8, 5], color: 0x8899aa, name: 'อาคาร A', data: { ชั้น: 4, พื้นที่: '2,400 ตร.ม.', วัสดุ: 'คอนกรีต', ปีสร้าง: 2019 } },
            { pos: [3, 0, -4], size: [6, 12, 6], color: 0x998877, name: 'อาคาร B', data: { ชั้น: 6, พื้นที่: '4,320 ตร.ม.', วัสดุ: 'เหล็ก+กระจก', ปีสร้าง: 2021 } },
            { pos: [12, 0, -3], size: [4, 6, 4], color: 0x7788aa, name: 'โรงรถ', data: { ชั้น: 3, พื้นที่: '960 ตร.ม.', วัสดุ: 'คอนกรีต', ปีสร้าง: 2020 } },
            { pos: [-3, 0, 6], size: [8, 4, 5], color: 0xaa8877, name: 'ห้องประชุม', data: { ชั้น: 2, พื้นที่: '1,600 ตร.ม.', วัสดุ: 'อิฐ', ปีสร้าง: 2018 } },
            { pos: [8, 0, 7], size: [4, 3, 6], color: 0x889966, name: 'โรงอาหาร', data: { ชั้น: 1, พื้นที่: '720 ตร.ม.', วัสดุ: 'เหล็ก', ปีสร้าง: 2022 } },
        ];
        
        const annotationObjects = [];
        const raycaster = new THREE.Raycaster();
        const mouse = new THREE.Vector2();
        let activeAnn = null;
        
        buildings.forEach(b => {
            const mesh = new THREE.Mesh(
                new THREE.BoxGeometry(...b.size),
                new THREE.MeshStandardMaterial({ color: b.color, roughness: 0.6 })
            );
            mesh.position.set(b.pos[0], b.size[1] / 2, b.pos[2]);
            mesh.castShadow = true;
            mesh.receiveShadow = true;
            scene.add(mesh);
            
            // Annotation marker
            const markerEl = document.createElement('div');
            markerEl.className = 'ann-marker';
            markerEl.innerHTML = `<div class="dot"></div>`;
            
            const markerObj = new CSS2DObject(markerEl);
            markerObj.position.set(0, b.size[1] / 2 + 1, 0);
            mesh.add(markerObj);
            
            // Panel
            const panelEl = document.createElement('div');
            panelEl.className = 'ann-panel hidden';
            
            const rows = Object.entries(b.data).map(([k, v]) =>
                `<div class="ann-panel-row"><span class="label">${k}</span><span class="value">${v}</span></div>`
            ).join('');
            
            panelEl.innerHTML = `
                <div class="ann-panel-header">
                    <span class="ann-panel-title">🏢 ${b.name}</span>
                    <button class="ann-panel-close">✕</button>
                </div>
                <div class="ann-panel-body">${rows}</div>
            `;
            
            const panelObj = new CSS2DObject(panelEl);
            panelObj.position.set(b.size[0] / 2 + 1, b.size[1] / 2, 0);
            mesh.add(panelObj);
            
            panelEl.querySelector('.ann-panel-close').addEventListener('click', (e) => {
                e.stopPropagation();
                hideAnnotation();
            });
            
            markerEl.addEventListener('click', (e) => {
                e.stopPropagation();
                showAnnotation(mesh, markerEl, panelEl);
            });
            
            annotationObjects.push({ mesh, markerEl, panelEl });
        });
        
        function showAnnotation(mesh, markerEl, panelEl) {
            hideAnnotation();
            markerEl.classList.add('active');
            panelEl.classList.remove('hidden');
            activeAnn = { mesh, markerEl, panelEl };
            
            // Highlight
            mesh.material.emissive = new THREE.Color(0x112233);
        }
        
        function hideAnnotation() {
            if (!activeAnn) return;
            activeAnn.markerEl.classList.remove('active');
            activeAnn.panelEl.classList.add('hidden');
            activeAnn.mesh.material.emissive = new THREE.Color(0x000000);
            activeAnn = null;
        }
        
        window.addEventListener('mousemove', (e) => {
            mouse.x = (e.clientX / W) * 2 - 1;
            mouse.y = -(e.clientY / H) * 2 + 1;
        });
        
        window.addEventListener('click', () => {
            raycaster.setFromCamera(mouse, camera);
            const meshes = annotationObjects.map(a => a.mesh);
            const hits = raycaster.intersectObjects(meshes);
            
            if (hits.length > 0) {
                const hitMesh = hits[0].object;
                const ann = annotationObjects.find(a => a.mesh === hitMesh);
                if (ann) showAnnotation(hitMesh, ann.markerEl, ann.panelEl);
            } else {
                hideAnnotation();
            }
        });
        
        // Compass
        const compassNeedle = document.getElementById('needle');
        
        function animate() {
            requestAnimationFrame(animate);
            
            // Update compass
            const dir = new THREE.Vector3();
            camera.getWorldDirection(dir);
            const angle = Math.atan2(dir.x, dir.z) * 180 / Math.PI;
            compassNeedle.setAttribute('transform', `rotate(${angle}, 36, 36)`);
            
            controls.update();
            renderer.render(scene, camera);
            labelRenderer.render(scene, camera);
        }
        animate();
        
        window.addEventListener('resize', () => {
            const w = window.innerWidth, h = window.innerHeight;
            camera.aspect = w / h;
            camera.updateProjectionMatrix();
            renderer.setSize(w, h);
            labelRenderer.setSize(w, h);
        });
    </script>
</body>
</html>
```

## Game HUD สมบูรณ์พร้อม Labels

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Game HUD with CSS2D Labels</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; }
        canvas { display: block; }
        
        #label-container { position: absolute; top: 0; left: 0; pointer-events: none; }
        
        .enemy-name {
            background: rgba(0,0,0,0.5);
            color: #ff8888;
            font-size: 10px;
            font-family: Arial;
            padding: 1px 5px;
            border-radius: 3px;
            white-space: nowrap;
        }
        .health-bar-container {
            display: flex;
            flex-direction: column;
            align-items: center;
            gap: 1px;
        }
        .health-bar-bg {
            width: 50px; height: 4px;
            background: rgba(0,0,0,0.5);
            border-radius: 2px;
            overflow: hidden;
        }
        .health-bar-fill {
            height: 100%;
            background: #44ff44;
            border-radius: 2px;
            transition: width 0.3s, background 0.3s;
        }
        
        /* Fixed HUD */
        #fixed-hud {
            position: fixed; top: 0; left: 0;
            width: 100%; height: 100%;
            pointer-events: none;
            z-index: 10;
        }
        .hud-bar { position: absolute; bottom: 20px; }
        .hud-bar.health { left: 20px; }
        .hud-bar.mana { left: 200px; }
        .hud-bar-inner {
            width: 160px; height: 18px;
            background: rgba(0,0,0,0.5);
            border-radius: 9px;
            overflow: hidden;
            border: 1px solid rgba(255,255,255,0.2);
        }
        .hud-bar-fill {
            height: 100%;
            border-radius: 9px;
            transition: width 0.5s;
        }
        .hud-bar.health .hud-bar-fill { background: linear-gradient(to right, #ff2222, #ff6644); }
        .hud-bar.mana .hud-bar-fill { background: linear-gradient(to right, #2244ff, #44aaff); }
        .hud-bar-label {
            font-size: 11px;
            color: rgba(255,255,255,0.6);
            margin-bottom: 2px;
        }
        
        .minimap {
            position: absolute;
            bottom: 20px; right: 20px;
            width: 120px; height: 120px;
            border-radius: 50%;
            background: rgba(0,20,0,0.8);
            border: 2px solid rgba(100,255,100,0.4);
            overflow: hidden;
        }
    </style>
</head>
<body>
    <canvas id="canvas"></canvas>
    <div id="fixed-hud">
        <div class="hud-bar health">
            <div class="hud-bar-label">❤️ Health</div>
            <div class="hud-bar-inner">
                <div class="hud-bar-fill" id="health-fill" style="width: 80%"></div>
            </div>
        </div>
        <div class="hud-bar mana">
            <div class="hud-bar-label">🔷 Mana</div>
            <div class="hud-bar-inner">
                <div class="hud-bar-fill" id="mana-fill" style="width: 60%"></div>
            </div>
        </div>
        <div class="minimap" id="minimap"></div>
    </div>

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
        import { CSS2DRenderer, CSS2DObject } from 'three/examples/jsm/renderers/CSS2DRenderer.js';
        
        const W = window.innerWidth, H = window.innerHeight;
        
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x0a1a0a);
        scene.fog = new THREE.FogExp2(0x0a1a0a, 0.02);
        
        const camera = new THREE.PerspectiveCamera(75, W / H, 0.1, 500);
        camera.position.set(0, 5, 15);
        
        const renderer = new THREE.WebGLRenderer({ canvas: document.getElementById('canvas'), antialias: true });
        renderer.setSize(W, H);
        renderer.shadowMap.enabled = true;
        
        const labelRenderer = new CSS2DRenderer();
        labelRenderer.setSize(W, H);
        labelRenderer.domElement.id = 'label-container';
        document.body.appendChild(labelRenderer.domElement);
        
        // Lighting
        scene.add(new THREE.AmbientLight(0x224422, 2));
        const dirLight = new THREE.DirectionalLight(0xffffff, 1.5);
        dirLight.position.set(10, 20, 5);
        dirLight.castShadow = true;
        scene.add(dirLight);
        
        // Ground
        const ground = new THREE.Mesh(
            new THREE.PlaneGeometry(100, 100),
            new THREE.MeshStandardMaterial({ color: 0x1a3a1a, roughness: 1 })
        );
        ground.rotation.x = -Math.PI / 2;
        ground.receiveShadow = true;
        scene.add(ground);
        
        // Player
        const player = new THREE.Mesh(
            new THREE.CapsuleGeometry(0.4, 1.2, 8, 12),
            new THREE.MeshStandardMaterial({ color: 0x4488ff })
        );
        player.position.y = 1;
        player.castShadow = true;
        scene.add(player);
        
        // Enemies with CSS2D health bars
        const enemyData = [
            { pos: [-5, 0, -8], name: 'Goblin', hp: 60, maxHp: 60, color: 0x44aa44 },
            { pos: [6, 0, -6], name: 'Orc', hp: 100, maxHp: 100, color: 0x886644 },
            { pos: [-3, 0, -14], name: 'Dragon', hp: 200, maxHp: 200, color: 0xaa2222 },
        ];
        
        const enemies = [];
        enemyData.forEach(e => {
            const mesh = new THREE.Mesh(
                new THREE.CapsuleGeometry(0.5, 1, 8, 12),
                new THREE.MeshStandardMaterial({ color: e.color })
            );
            mesh.position.set(e.pos[0], 1, e.pos[2]);
            mesh.castShadow = true;
            scene.add(mesh);
            
            // Health bar CSS2D
            const container = document.createElement('div');
            container.className = 'health-bar-container';
            
            const nameEl = document.createElement('div');
            nameEl.className = 'enemy-name';
            nameEl.textContent = e.name;
            
            const barBg = document.createElement('div');
            barBg.className = 'health-bar-bg';
            
            const barFill = document.createElement('div');
            barFill.className = 'health-bar-fill';
            barFill.style.width = `${(e.hp / e.maxHp) * 100}%`;
            
            barBg.appendChild(barFill);
            container.appendChild(nameEl);
            container.appendChild(barBg);
            
            const labelObj = new CSS2DObject(container);
            labelObj.position.set(0, 2.5, 0);
            mesh.add(labelObj);
            
            enemies.push({ mesh, barFill, ...e });
        });
        
        // Keyboard
        const keys = {};
        window.addEventListener('keydown', e => keys[e.code] = true);
        window.addEventListener('keyup', e => keys[e.code] = false);
        
        let health = 80, mana = 60;
        
        function animate() {
            requestAnimationFrame(animate);
            const t = performance.now() * 0.001;
            
            // Player movement
            const speed = 0.08;
            if (keys['KeyW']) player.position.z -= speed;
            if (keys['KeyS']) player.position.z += speed;
            if (keys['KeyA']) player.position.x -= speed;
            if (keys['KeyD']) player.position.x += speed;
            
            // Camera follow
            camera.position.x += (player.position.x - camera.position.x + 0) * 0.05;
            camera.position.z += (player.position.z - camera.position.z + 15) * 0.05;
            camera.lookAt(player.position);
            
            // Animate enemies
            enemies.forEach((e, i) => {
                e.mesh.rotation.y = t + i * 1.3;
                // Simulate hp changes
                const hp = e.maxHp * (0.5 + 0.5 * Math.abs(Math.sin(t * 0.2 + i)));
                e.barFill.style.width = `${(hp / e.maxHp) * 100}%`;
                if (hp / e.maxHp < 0.3) e.barFill.style.background = '#ff2222';
                else if (hp / e.maxHp < 0.6) e.barFill.style.background = '#ffaa22';
                else e.barFill.style.background = '#44ff44';
            });
            
            // Update HUD
            document.getElementById('health-fill').style.width = `${health + Math.sin(t * 0.3) * 10}%`;
            document.getElementById('mana-fill').style.width = `${mana + Math.sin(t * 0.5 + 1) * 15}%`;
            
            renderer.render(scene, camera);
            labelRenderer.render(scene, camera);
        }
        animate();
        
        window.addEventListener('resize', () => {
            const w = window.innerWidth, h = window.innerHeight;
            camera.aspect = w / h;
            camera.updateProjectionMatrix();
            renderer.setSize(w, h);
            labelRenderer.setSize(w, h);
        });
    </script>
</body>
</html>
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Museum Guide System
สร้างระบบ annotation สำหรับพิพิธภัณฑ์ 3D ที่มีรูปปั้น 5 ชิ้น แต่ละชิ้นมี marker และ panel ที่แสดงข้อมูลศิลปิน, ปีสร้าง, วัสดุ และคำอธิบาย

### แบบฝึกหัดที่ 2: Location Pin System
สร้าง map 3D ที่มี location pins 10 จุด แต่ละ pin มี category ต่างกัน (ร้านอาหาร, โรงแรม, ห้างสรรพสินค้า) pins ต้องมี animation pulse และ label แสดงชื่อเมื่อ hover

### แบบฝึกหัดที่ 3: RPG Health Bars
สร้าง scene ที่มีตัวละคร 5 ตัว แต่ละตัวมี health bar, ชื่อ และ damage number pop-up เมื่อ HP ลด สร้าง auto-attack system ง่ายๆ

### แบบฝึกหัดที่ 4: Distance Labels
สร้างระบบ distance measurement ใน 3D space ที่แสดง label บอกระยะห่างระหว่าง objects ต่างๆ กับ player ตัวเลขต้องอัพเดตแบบ real-time

## สรุป

CSS2DRenderer เหมาะสำหรับ:
- World-space labels ที่อยู่ตำแหน่ง 3D แต่แสดงเป็น HTML overlay
- Annotation systems ที่มี rich HTML content
- Health bars และ status indicators เหนือตัวละคร
- Location pins และ markers

กุญแจสำคัญคือการตั้ง `pointerEvents: none` บน labelRenderer และเปิดเฉพาะ elements ที่ต้องการ interaction เพื่อไม่ให้รบกวน OrbitControls
