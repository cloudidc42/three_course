# Part 98: Minimap, Compass & Navigation

## บทนำ

ระบบ minimap และ navigation เป็นส่วนสำคัญของ 3D games และ applications ที่มี world ขนาดใหญ่ ในบทนี้เราจะสร้าง minimap ด้วย WebGLRenderTarget, radar ที่แสดง enemy blips, compass rose ที่ติดตาม camera direction และระบบ waypoints ที่สมบูรณ์

## Minimap ด้วย WebGLRenderTarget

แนวคิดหลัก: ใช้ camera ที่สอง (orthographic, มองลงจากด้านบน) render scene ลง texture แล้วแสดง texture นั้นบน quad ในมุมหนึ่งของหน้าจอ

```javascript
// Minimap camera
const minimapCamera = new THREE.OrthographicCamera(
    -mapSize / 2, mapSize / 2,  // left, right
    mapSize / 2, -mapSize / 2,  // top, bottom
    0.1, 1000
);
minimapCamera.position.set(0, 500, 0);
minimapCamera.rotation.x = -Math.PI / 2;
minimapCamera.lookAt(0, 0, 0);

// Render target
const minimapTarget = new THREE.WebGLRenderTarget(256, 256, {
    minFilter: THREE.LinearFilter,
    magFilter: THREE.LinearFilter,
    format: THREE.RGBAFormat,
});

// Display quad
const minimapPlane = new THREE.Mesh(
    new THREE.PlaneGeometry(150, 150),
    new THREE.MeshBasicMaterial({ map: minimapTarget.texture })
);
// วาง minimap ที่มุมขวาล่าง (HUD scene)
minimapPlane.position.set(HUDWidth / 2 - 90, -HUDHeight / 2 + 90, 0);
hudScene.add(minimapPlane);

// ใน animate loop:
renderer.setRenderTarget(minimapTarget);
renderer.render(mainScene, minimapCamera);
renderer.setRenderTarget(null);
renderer.render(mainScene, mainCamera);
// render HUD
renderer.render(hudScene, hudCamera);
```

## สร้าง Minimap ที่สมบูรณ์

```javascript
class Minimap {
    constructor(renderer, scene, options = {}) {
        this.renderer = renderer;
        this.scene = scene;
        this.options = {
            size: 200,        // pixel size ของ minimap
            worldSize: 500,   // พื้นที่โลกที่แสดง
            position: { x: 0.85, y: 0.15 }, // normalized screen position
            borderColor: '#4488ff',
            backgroundColor: '#0a1a0a',
            ...options,
        };
        
        this.setup();
    }
    
    setup() {
        const { size, worldSize } = this.options;
        
        // Render target
        this.renderTarget = new THREE.WebGLRenderTarget(size, size, {
            minFilter: THREE.LinearFilter,
            magFilter: THREE.LinearFilter,
        });
        
        // Top-down camera
        this.camera = new THREE.OrthographicCamera(
            -worldSize / 2, worldSize / 2,
            worldSize / 2, -worldSize / 2,
            -1000, 1000
        );
        this.camera.position.set(0, 500, 0);
        this.camera.lookAt(0, 0, 0);
        
        // HUD scene for display
        this.hudScene = new THREE.Scene();
        const w = this.renderer.domElement.width;
        const h = this.renderer.domElement.height;
        this.hudCamera = new THREE.OrthographicCamera(-w/2, w/2, h/2, -h/2, -10, 10);
        
        // Create minimap display
        this.createDisplay();
        this.createPlayerMarker();
    }
    
    createDisplay() {
        const { size } = this.options;
        const canvas = document.createElement('canvas');
        canvas.width = size + 4;
        canvas.height = size + 4;
        const ctx = canvas.getContext('2d');
        
        // Border
        ctx.strokeStyle = this.options.borderColor;
        ctx.lineWidth = 2;
        ctx.beginPath();
        ctx.arc(size / 2 + 2, size / 2 + 2, size / 2, 0, Math.PI * 2);
        ctx.stroke();
        
        const borderTex = new THREE.CanvasTexture(canvas);
        const borderMesh = new THREE.Mesh(
            new THREE.PlaneGeometry(size + 4, size + 4),
            new THREE.MeshBasicMaterial({ map: borderTex, transparent: true })
        );
        borderMesh.position.z = 1;
        this.hudScene.add(borderMesh);
        
        // Map display (circular clip via alpha)
        const circleMask = document.createElement('canvas');
        circleMask.width = size;
        circleMask.height = size;
        const mctx = circleMask.getContext('2d');
        mctx.fillStyle = 'white';
        mctx.beginPath();
        mctx.arc(size / 2, size / 2, size / 2 - 2, 0, Math.PI * 2);
        mctx.fill();
        
        this.mapDisplayMesh = new THREE.Mesh(
            new THREE.PlaneGeometry(size, size),
            new THREE.MeshBasicMaterial({
                map: this.renderTarget.texture,
                alphaMap: new THREE.CanvasTexture(circleMask),
                transparent: true,
            })
        );
        this.hudScene.add(this.mapDisplayMesh);
        this.positionDisplay();
    }
    
    positionDisplay() {
        const w = this.renderer.domElement.width;
        const h = this.renderer.domElement.height;
        const { size, position } = this.options;
        const x = (position.x - 0.5) * w;
        const y = -(position.y - 0.5) * h;
        this.mapDisplayMesh.position.set(x, y, 0);
    }
    
    createPlayerMarker() {
        // Triangle marker สำหรับ player
        const canvas = document.createElement('canvas');
        canvas.width = 32; canvas.height = 32;
        const ctx = canvas.getContext('2d');
        ctx.fillStyle = '#44ff88';
        ctx.beginPath();
        ctx.moveTo(16, 2);
        ctx.lineTo(28, 28);
        ctx.lineTo(16, 22);
        ctx.lineTo(4, 28);
        ctx.closePath();
        ctx.fill();
        
        this.playerMarker = new THREE.Mesh(
            new THREE.PlaneGeometry(12, 12),
            new THREE.MeshBasicMaterial({ map: new THREE.CanvasTexture(canvas), transparent: true, depthTest: false })
        );
        this.playerMarker.position.z = 2;
        this.hudScene.add(this.playerMarker);
    }
    
    // อัพเดต minimap camera ให้ follow player
    update(playerPosition, playerRotation) {
        const w = this.renderer.domElement.width;
        const h = this.renderer.domElement.height;
        const { position } = this.options;
        
        // Camera follows player
        this.camera.position.x = playerPosition.x;
        this.camera.position.z = playerPosition.z;
        
        // Render scene to texture
        this.renderer.setRenderTarget(this.renderTarget);
        this.renderer.setClearColor(0x0a1a0a);
        this.renderer.clear();
        this.renderer.render(this.scene, this.camera);
        this.renderer.setRenderTarget(null);
        
        // Update player marker position (center of minimap)
        const mx = (position.x - 0.5) * w;
        const my = -(position.y - 0.5) * h;
        this.playerMarker.position.set(mx, my, 2);
        
        // Rotate marker to match player direction
        if (playerRotation !== undefined) {
            this.playerMarker.rotation.z = -playerRotation;
        }
    }
    
    render() {
        this.renderer.render(this.hudScene, this.hudCamera);
    }
}
```

## Radar Minimap (สำหรับ Game)

```javascript
class RadarMinimap {
    constructor(container) {
        this.container = container;
        this.canvas = document.createElement('canvas');
        this.canvas.width = 160;
        this.canvas.height = 160;
        this.canvas.style.cssText = `
            position: absolute;
            bottom: 20px;
            right: 20px;
            border-radius: 50%;
            border: 2px solid rgba(100,255,100,0.4);
            background: rgba(0,20,0,0.7);
        `;
        this.ctx = this.canvas.getContext('2d');
        container.appendChild(this.canvas);
        
        this.blips = []; // [{pos, type, color}]
        this.sweepAngle = 0;
        this.size = 160;
    }
    
    setPlayer(position) {
        this.playerPos = position;
    }
    
    addBlip(position, type = 'enemy') {
        const colors = {
            enemy: '#ff4444',
            ally: '#44ff44',
            objective: '#ffff44',
            item: '#44aaff',
        };
        this.blips.push({ position, type, color: colors[type] });
    }
    
    clearBlips() {
        this.blips = [];
    }
    
    update(dt) {
        this.sweepAngle += dt * 1.5;
        this.render();
    }
    
    render() {
        const ctx = this.ctx;
        const cx = this.size / 2;
        const cy = this.size / 2;
        const r = this.size / 2 - 4;
        
        // Clear
        ctx.clearRect(0, 0, this.size, this.size);
        
        // Background
        ctx.fillStyle = 'rgba(0, 20, 0, 0.8)';
        ctx.beginPath();
        ctx.arc(cx, cy, r, 0, Math.PI * 2);
        ctx.fill();
        
        // Grid circles
        for (let i = 1; i <= 3; i++) {
            ctx.strokeStyle = `rgba(100, 255, 100, ${0.1 + i * 0.05})`;
            ctx.lineWidth = 1;
            ctx.beginPath();
            ctx.arc(cx, cy, r * (i / 3), 0, Math.PI * 2);
            ctx.stroke();
        }
        
        // Cross lines
        ctx.strokeStyle = 'rgba(100, 255, 100, 0.15)';
        ctx.beginPath();
        ctx.moveTo(cx - r, cy); ctx.lineTo(cx + r, cy);
        ctx.moveTo(cx, cy - r); ctx.lineTo(cx, cy + r);
        ctx.stroke();
        
        // Sweep effect
        const sweepGradient = ctx.createConicalGradient
            ? ctx.createConicalGradient(cx, cy, this.sweepAngle)
            : null;
        
        ctx.save();
        ctx.fillStyle = 'rgba(100, 255, 100, 0.08)';
        ctx.beginPath();
        ctx.moveTo(cx, cy);
        ctx.arc(cx, cy, r, this.sweepAngle - 1.5, this.sweepAngle);
        ctx.closePath();
        ctx.fill();
        ctx.restore();
        
        // Sweep line
        ctx.strokeStyle = 'rgba(100, 255, 100, 0.6)';
        ctx.lineWidth = 1.5;
        ctx.beginPath();
        ctx.moveTo(cx, cy);
        ctx.lineTo(
            cx + Math.cos(this.sweepAngle) * r,
            cy + Math.sin(this.sweepAngle) * r
        );
        ctx.stroke();
        
        // Blips
        const playerPos = this.playerPos || new THREE.Vector3();
        const worldToRadar = r / 200; // scale: 200 world units = radius of radar
        
        this.blips.forEach(blip => {
            const dx = (blip.position.x - playerPos.x) * worldToRadar;
            const dz = (blip.position.z - playerPos.z) * worldToRadar;
            const bx = cx + dx;
            const by = cy + dz;
            
            if (Math.sqrt(dx * dx + dz * dz) > r) return; // outside radar
            
            // Blip dot
            ctx.fillStyle = blip.color;
            ctx.shadowColor = blip.color;
            ctx.shadowBlur = 6;
            ctx.beginPath();
            ctx.arc(bx, by, 3, 0, Math.PI * 2);
            ctx.fill();
            ctx.shadowBlur = 0;
        });
        
        // Player dot (center)
        ctx.fillStyle = '#44ff44';
        ctx.shadowColor = '#44ff44';
        ctx.shadowBlur = 8;
        ctx.beginPath();
        ctx.arc(cx, cy, 4, 0, Math.PI * 2);
        ctx.fill();
        ctx.shadowBlur = 0;
        
        // Outer ring mask
        ctx.strokeStyle = 'rgba(100, 255, 100, 0.5)';
        ctx.lineWidth = 2;
        ctx.beginPath();
        ctx.arc(cx, cy, r, 0, Math.PI * 2);
        ctx.stroke();
        
        // N/S/E/W labels
        ctx.fillStyle = 'rgba(100, 255, 100, 0.5)';
        ctx.font = '9px Arial';
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';
        ctx.fillText('N', cx, cy - r + 8);
        ctx.fillText('S', cx, cy + r - 8);
        ctx.fillText('E', cx + r - 8, cy);
        ctx.fillText('W', cx - r + 8, cy);
    }
}
```

## Waypoint Navigation System

```javascript
class WaypointSystem {
    constructor(scene, camera, labelRenderer) {
        this.scene = scene;
        this.camera = camera;
        this.labelRenderer = labelRenderer;
        
        this.waypoints = [];
        this.activePath = null;
        this.pathMesh = null;
    }
    
    addWaypoint(position, data = {}) {
        const waypoint = {
            position: position.clone(),
            data: {
                name: 'Waypoint',
                icon: '📍',
                ...data,
            },
        };
        
        // 3D marker (cylinder)
        const markerGeo = new THREE.CylinderGeometry(0.5, 0, 3, 8);
        const markerMat = new THREE.MeshStandardMaterial({
            color: data.color || 0x4488ff,
            emissive: new THREE.Color(data.color || 0x4488ff).multiplyScalar(0.3),
        });
        const markerMesh = new THREE.Mesh(markerGeo, markerMat);
        markerMesh.position.copy(position);
        markerMesh.position.y += 1.5;
        this.scene.add(markerMesh);
        
        // Pulsing ring
        const ringGeo = new THREE.TorusGeometry(2, 0.15, 8, 32);
        const ringMat = new THREE.MeshBasicMaterial({
            color: data.color || 0x4488ff,
            transparent: true,
            opacity: 0.5,
        });
        const ring = new THREE.Mesh(ringGeo, ringMat);
        ring.rotation.x = Math.PI / 2;
        ring.position.copy(position);
        ring.position.y += 0.1;
        this.scene.add(ring);
        
        // CSS2D label
        const labelEl = document.createElement('div');
        labelEl.style.cssText = `
            color: white;
            font-family: Arial;
            font-size: 12px;
            background: rgba(0,0,0,0.7);
            padding: 3px 8px;
            border-radius: 4px;
            pointer-events: none;
            border-left: 2px solid ${data.color ? '#' + data.color.toString(16) : '#4488ff'};
        `;
        labelEl.innerHTML = `${data.icon || '📍'} ${data.name || 'Waypoint'}`;
        
        const labelObj = new CSS2DObject(labelEl);
        labelObj.position.set(0, 4, 0);
        markerMesh.add(labelObj);
        
        waypoint.mesh = markerMesh;
        waypoint.ring = ring;
        waypoint.labelEl = labelEl;
        
        this.waypoints.push(waypoint);
        return waypoint;
    }
    
    setActivePath(waypointIndices) {
        // ลบ path เดิม
        if (this.pathMesh) {
            this.scene.remove(this.pathMesh);
        }
        
        this.activePath = waypointIndices;
        
        // สร้าง path ใหม่
        const pathPoints = waypointIndices.map(i => {
            const wp = this.waypoints[i];
            return wp.position.clone().add(new THREE.Vector3(0, 0.3, 0));
        });
        
        if (pathPoints.length < 2) return;
        
        // Smooth curve
        const curve = new THREE.CatmullRomCurve3(pathPoints);
        const tubeGeo = new THREE.TubeGeometry(curve, pathPoints.length * 10, 0.15, 8, false);
        const tubeMat = new THREE.MeshBasicMaterial({
            color: 0x4488ff,
            transparent: true,
            opacity: 0.6,
        });
        
        this.pathMesh = new THREE.Mesh(tubeGeo, tubeMat);
        this.scene.add(this.pathMesh);
        
        // Arrow markers along path
        const numArrows = pathPoints.length - 1;
        for (let i = 0; i < numArrows; i++) {
            const t = (i + 0.5) / numArrows;
            const pos = curve.getPoint(t);
            const tangent = curve.getTangent(t);
            
            const arrowGeo = new THREE.ConeGeometry(0.4, 1, 8);
            const arrowMat = new THREE.MeshBasicMaterial({ color: 0x4488ff });
            const arrow = new THREE.Mesh(arrowGeo, arrowMat);
            arrow.position.copy(pos);
            arrow.lookAt(pos.clone().add(tangent));
            arrow.rotateX(Math.PI / 2);
            this.scene.add(arrow);
        }
    }
    
    // ระยะทางถึง waypoint ถัดไป
    getDistanceToNext(playerPosition) {
        if (!this.activePath || this.activePath.length === 0) return null;
        
        const nextWp = this.waypoints[this.activePath[0]];
        return playerPosition.distanceTo(nextWp.position);
    }
    
    // ตรวจสอบว่าถึง waypoint แล้วหรือยัง
    checkReached(playerPosition, threshold = 3) {
        if (!this.activePath || this.activePath.length === 0) return null;
        
        const nextIdx = this.activePath[0];
        const nextWp = this.waypoints[nextIdx];
        const dist = playerPosition.distanceTo(nextWp.position);
        
        if (dist < threshold) {
            this.activePath.shift();
            return nextWp;
        }
        return null;
    }
    
    update(dt) {
        // Animate waypoint rings
        this.waypoints.forEach((wp, i) => {
            const t = performance.now() * 0.002;
            wp.ring.scale.setScalar(1 + Math.sin(t + i) * 0.2);
            wp.ring.material.opacity = 0.3 + Math.sin(t + i) * 0.2;
        });
    }
}
```

## Complete Demo: Game Minimap System

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Minimap & Navigation System</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #0a1a0a; overflow: hidden; font-family: Arial, sans-serif; }
        canvas#canvas { display: block; }
        
        #hud {
            position: fixed; top: 0; left: 0;
            width: 100%; height: 100%;
            pointer-events: none;
            z-index: 10;
        }
        
        /* Minimap */
        #minimap-container {
            position: absolute;
            bottom: 20px; right: 20px;
            width: 180px; height: 180px;
        }
        #minimap-canvas {
            position: absolute; top: 0; left: 0;
            width: 180px; height: 180px;
            border-radius: 50%;
            border: 2px solid rgba(100,200,100,0.5);
        }
        #minimap-player {
            position: absolute;
            top: 50%; left: 50%;
            transform: translate(-50%, -50%);
            width: 0; height: 0;
            border-left: 5px solid transparent;
            border-right: 5px solid transparent;
            border-bottom: 10px solid #44ff44;
            filter: drop-shadow(0 0 4px #44ff44);
            transform-origin: center 7px;
        }
        
        /* Compass */
        #compass {
            position: absolute;
            top: 16px; right: 16px;
            width: 80px; height: 80px;
        }
        
        /* Waypoint HUD */
        #waypoint-info {
            position: absolute;
            top: 50%; left: 50%;
            transform: translate(-50%, -120px);
            text-align: center;
            color: #44aaff;
            font-size: 13px;
        }
        
        /* Controls */
        #controls-hint {
            position: absolute;
            bottom: 16px; left: 16px;
            color: rgba(255,255,255,0.4);
            font-size: 11px;
            line-height: 1.7;
        }
        
        /* Radar overlay (top of minimap) */
        #radar-sweep {
            position: absolute; top: 0; left: 0;
            width: 180px; height: 180px;
            border-radius: 50%;
            pointer-events: none;
        }
    </style>
</head>
<body>
    <canvas id="canvas"></canvas>
    <div id="hud">
        <div id="minimap-container">
            <canvas id="minimap-canvas" width="180" height="180"></canvas>
            <div id="minimap-player"></div>
        </div>
        
        <svg id="compass" viewBox="0 0 80 80">
            <circle cx="40" cy="40" r="36" fill="rgba(0,20,0,0.7)" stroke="rgba(100,200,100,0.4)" stroke-width="1.5"/>
            <g id="compass-needle">
                <polygon points="40,8 43,38 40,34 37,38" fill="#ff4444"/>
                <polygon points="40,72 43,42 40,46 37,42" fill="rgba(200,200,200,0.6)"/>
            </g>
            <text x="40" y="6" text-anchor="middle" fill="#ff5555" font-size="8" font-weight="bold">N</text>
            <text x="40" y="75" text-anchor="middle" fill="white" font-size="7">S</text>
            <text x="73" y="43" text-anchor="middle" fill="white" font-size="7">E</text>
            <text x="7" y="43" text-anchor="middle" fill="white" font-size="7">W</text>
            <circle cx="40" cy="40" r="2" fill="white"/>
        </svg>
        
        <div id="waypoint-info">
            <div id="wp-name"></div>
            <div id="wp-dist"></div>
        </div>
        
        <div id="controls-hint">
            WASD: เดิน | เม้าส์: หมุน<br>
            คลิก: เพิ่ม waypoint ที่ตำแหน่งนั้น
        </div>
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
        
        // === Scenes ===
        const mainScene = new THREE.Scene();
        mainScene.background = new THREE.Color(0x0a1a0a);
        mainScene.fog = new THREE.FogExp2(0x0a1a0a, 0.015);
        
        // === Cameras ===
        const mainCamera = new THREE.PerspectiveCamera(75, W / H, 0.1, 500);
        mainCamera.position.set(0, 3, 0);
        
        const minimapCamera = new THREE.OrthographicCamera(-200, 200, 200, -200, -100, 600);
        minimapCamera.position.set(0, 500, 0);
        minimapCamera.lookAt(0, 0, 0);
        
        // === Renderer ===
        const renderer = new THREE.WebGLRenderer({ canvas: document.getElementById('canvas'), antialias: true });
        renderer.setSize(W, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        renderer.shadowMap.enabled = true;
        renderer.autoClear = false;
        
        const labelRenderer = new CSS2DRenderer();
        labelRenderer.setSize(W, H);
        labelRenderer.domElement.style.cssText = 'position:absolute;top:0;left:0;pointer-events:none;z-index:5;';
        document.body.appendChild(labelRenderer.domElement);
        
        // === Minimap RenderTarget ===
        const minimapTarget = new THREE.WebGLRenderTarget(256, 256, {
            minFilter: THREE.LinearFilter,
            magFilter: THREE.LinearFilter,
        });
        
        // === Lighting ===
        mainScene.add(new THREE.AmbientLight(0x224422, 2));
        const sun = new THREE.DirectionalLight(0xffffff, 1.5);
        sun.position.set(50, 100, 50);
        sun.castShadow = true;
        mainScene.add(sun);
        
        // === World ===
        // Ground
        const ground = new THREE.Mesh(
            new THREE.PlaneGeometry(400, 400, 40, 40),
            new THREE.MeshStandardMaterial({ color: 0x1a3a1a, roughness: 0.9 })
        );
        ground.rotation.x = -Math.PI / 2;
        ground.receiveShadow = true;
        mainScene.add(ground);
        
        // Trees
        const treePositions = [];
        for (let i = 0; i < 80; i++) {
            const x = (Math.random() - 0.5) * 380;
            const z = (Math.random() - 0.5) * 380;
            if (Math.abs(x) < 10 && Math.abs(z) < 10) continue;
            
            const trunk = new THREE.Mesh(
                new THREE.CylinderGeometry(0.3, 0.4, 2, 8),
                new THREE.MeshStandardMaterial({ color: 0x664422 })
            );
            trunk.position.set(x, 1, z);
            trunk.castShadow = true;
            mainScene.add(trunk);
            
            const top = new THREE.Mesh(
                new THREE.ConeGeometry(2, 5, 8),
                new THREE.MeshStandardMaterial({ color: 0x22662a + Math.floor(Math.random() * 0x002200) })
            );
            top.position.set(x, 5, z);
            top.castShadow = true;
            mainScene.add(top);
            treePositions.push({ x, z });
        }
        
        // Buildings
        const buildings = [
            { x: 40, z: -40, w: 12, d: 12, h: 15, color: 0x888899 },
            { x: -50, z: 30, w: 20, d: 10, h: 10, color: 0x997766 },
            { x: 20, z: 50, w: 10, d: 20, h: 20, color: 0x778899 },
            { x: -30, z: -60, w: 15, d: 15, h: 8, color: 0x886644 },
        ];
        
        buildings.forEach(b => {
            const mesh = new THREE.Mesh(
                new THREE.BoxGeometry(b.w, b.h, b.d),
                new THREE.MeshStandardMaterial({ color: b.color })
            );
            mesh.position.set(b.x, b.h / 2, b.z);
            mesh.castShadow = true;
            mesh.receiveShadow = true;
            mainScene.add(mesh);
            
            // Minimap marker (colored dot on ground)
            const marker = new THREE.Mesh(
                new THREE.BoxGeometry(b.w, 0.1, b.d),
                new THREE.MeshBasicMaterial({ color: 0x888899 })
            );
            marker.position.set(b.x, 0.1, b.z);
            mainScene.add(marker);
        });
        
        // === Player ===
        const player = new THREE.Group();
        const playerBody = new THREE.Mesh(
            new THREE.CapsuleGeometry(0.4, 1.2, 8, 12),
            new THREE.MeshStandardMaterial({ color: 0x4488ff })
        );
        playerBody.position.y = 1;
        player.add(playerBody);
        mainScene.add(player);
        
        // Player on minimap
        const playerMinimapDot = new THREE.Mesh(
            new THREE.CircleGeometry(4, 12),
            new THREE.MeshBasicMaterial({ color: 0x44ff44 })
        );
        playerMinimapDot.rotation.x = -Math.PI / 2;
        playerMinimapDot.position.y = 0.2;
        mainScene.add(playerMinimapDot);
        
        // === Waypoints ===
        const waypointData = [
            { x: 40, z: -40, name: '🏢 อาคาร A', color: 0xff8844 },
            { x: -50, z: 30, name: '🏠 บ้านพัก', color: 0xff4444 },
            { x: 20, z: 50, name: '🏛️ อาคาร B', color: 0x4488ff },
            { x: 0, z: -80, name: '⭐ เป้าหมาย', color: 0xffdd44 },
        ];
        
        const waypointMeshes = [];
        waypointData.forEach(wp => {
            // Ground marker
            const ring = new THREE.Mesh(
                new THREE.TorusGeometry(4, 0.3, 8, 32),
                new THREE.MeshBasicMaterial({ color: wp.color })
            );
            ring.rotation.x = Math.PI / 2;
            ring.position.set(wp.x, 0.3, wp.z);
            mainScene.add(ring);
            
            // Spike
            const spike = new THREE.Mesh(
                new THREE.CylinderGeometry(0, 0.5, 8, 8),
                new THREE.MeshStandardMaterial({ color: wp.color, emissive: new THREE.Color(wp.color).multiplyScalar(0.3) })
            );
            spike.position.set(wp.x, 4, wp.z);
            spike.rotation.x = Math.PI;
            mainScene.add(spike);
            
            // Minimap dot
            const dot = new THREE.Mesh(
                new THREE.CircleGeometry(3, 12),
                new THREE.MeshBasicMaterial({ color: wp.color })
            );
            dot.rotation.x = -Math.PI / 2;
            dot.position.set(wp.x, 0.15, wp.z);
            mainScene.add(dot);
            
            // CSS2D label
            const labelEl = document.createElement('div');
            labelEl.style.cssText = `
                color: white; font-size: 11px;
                background: rgba(0,0,0,0.7);
                padding: 2px 6px; border-radius: 3px;
                white-space: nowrap; pointer-events: none;
                border-left: 2px solid #${wp.color.toString(16).padStart(6,'0')};
            `;
            labelEl.textContent = wp.name;
            const labelObj = new CSS2DObject(labelEl);
            labelObj.position.set(0, 9, 0);
            spike.add(labelObj);
            
            waypointMeshes.push({ ...wp, ring, spike });
        });
        
        // === Input ===
        const keys = {};
        let yaw = 0;
        window.addEventListener('keydown', e => { keys[e.code] = true; e.preventDefault(); });
        window.addEventListener('keyup', e => { keys[e.code] = false; });
        
        let isPointerLocked = false;
        document.getElementById('canvas').addEventListener('click', () => {
            document.getElementById('canvas').requestPointerLock();
        });
        document.addEventListener('pointerlockchange', () => {
            isPointerLocked = document.pointerLockElement === document.getElementById('canvas');
        });
        window.addEventListener('mousemove', (e) => {
            if (!isPointerLocked) return;
            yaw -= e.movementX * 0.003;
        });
        
        // === Minimap Canvas 2D ===
        const minimapCtx = document.getElementById('minimap-canvas').getContext('2d');
        const minimapSize = 180;
        const minimapRadius = minimapSize / 2;
        const worldToMap = minimapRadius / 200; // 200 world units per radius
        
        function drawMinimap2D(playerPos, playerYaw) {
            const ctx = minimapCtx;
            ctx.clearRect(0, 0, minimapSize, minimapSize);
            
            ctx.save();
            ctx.beginPath();
            ctx.arc(minimapRadius, minimapRadius, minimapRadius - 1, 0, Math.PI * 2);
            ctx.clip();
            
            // Background
            ctx.fillStyle = '#0a1a0a';
            ctx.fillRect(0, 0, minimapSize, minimapSize);
            
            // Transform to player perspective
            ctx.translate(minimapRadius, minimapRadius);
            ctx.rotate(-playerYaw);
            ctx.translate(-playerPos.x * worldToMap, -playerPos.z * worldToMap);
            
            // Draw trees
            treePositions.forEach(t => {
                ctx.fillStyle = 'rgba(30, 80, 30, 0.7)';
                ctx.beginPath();
                ctx.arc(t.x * worldToMap, t.z * worldToMap, 3, 0, Math.PI * 2);
                ctx.fill();
            });
            
            // Draw buildings
            buildings.forEach(b => {
                ctx.fillStyle = '#556677';
                ctx.fillRect(
                    b.x * worldToMap - (b.w * worldToMap) / 2,
                    b.z * worldToMap - (b.d * worldToMap) / 2,
                    b.w * worldToMap, b.d * worldToMap
                );
            });
            
            // Draw waypoints
            waypointData.forEach(wp => {
                ctx.fillStyle = '#' + wp.color.toString(16).padStart(6, '0');
                ctx.beginPath();
                ctx.arc(wp.x * worldToMap, wp.z * worldToMap, 4, 0, Math.PI * 2);
                ctx.fill();
            });
            
            ctx.restore();
            
            // Player marker (always center, rotated separately)
            document.getElementById('minimap-player').style.transform =
                `translate(-50%, -50%) rotate(0deg)`;
            
            // Range circle
            ctx.strokeStyle = 'rgba(100, 200, 100, 0.3)';
            ctx.lineWidth = 1;
            ctx.beginPath();
            ctx.arc(minimapRadius, minimapRadius, minimapRadius * 0.5, 0, Math.PI * 2);
            ctx.stroke();
            
            // Outer ring
            ctx.strokeStyle = 'rgba(100, 200, 100, 0.6)';
            ctx.lineWidth = 2;
            ctx.beginPath();
            ctx.arc(minimapRadius, minimapRadius, minimapRadius - 1, 0, Math.PI * 2);
            ctx.stroke();
        }
        
        const compassNeedle = document.getElementById('compass-needle');
        
        let lastTime = 0;
        function animate(time) {
            requestAnimationFrame(animate);
            const dt = Math.min((time - lastTime) / 1000, 0.05);
            lastTime = time;
            
            // Player movement
            const speed = 8;
            const forward = new THREE.Vector3(-Math.sin(yaw), 0, -Math.cos(yaw));
            const right = new THREE.Vector3(-Math.cos(yaw), 0, Math.sin(yaw));
            
            if (keys['KeyW']) player.position.addScaledVector(forward, speed * dt);
            if (keys['KeyS']) player.position.addScaledVector(forward, -speed * dt);
            if (keys['KeyA']) player.position.addScaledVector(right, speed * dt);
            if (keys['KeyD']) player.position.addScaledVector(right, -speed * dt);
            if (keys['ArrowLeft']) yaw += 2 * dt;
            if (keys['ArrowRight']) yaw -= 2 * dt;
            
            // Camera follows player
            mainCamera.position.copy(player.position);
            mainCamera.position.y += 1.7;
            mainCamera.rotation.y = yaw;
            
            // Minimap camera follows player
            minimapCamera.position.x = player.position.x;
            minimapCamera.position.z = player.position.z;
            
            // Update player minimap marker
            playerMinimapDot.position.x = player.position.x;
            playerMinimapDot.position.z = player.position.z;
            
            // Animate waypoints
            const t = time * 0.001;
            waypointMeshes.forEach((wp, i) => {
                wp.ring.scale.setScalar(1 + Math.sin(t * 2 + i) * 0.15);
            });
            
            // Draw 2D minimap
            drawMinimap2D(player.position, yaw);
            
            // Update compass
            const angle = yaw * 180 / Math.PI;
            compassNeedle.setAttribute('transform', `rotate(${angle}, 40, 40)`);
            
            // Nearest waypoint info
            let nearest = null, nearestDist = Infinity;
            waypointData.forEach(wp => {
                const dist = Math.sqrt(
                    Math.pow(player.position.x - wp.x, 2) +
                    Math.pow(player.position.z - wp.z, 2)
                );
                if (dist < nearestDist) {
                    nearestDist = dist;
                    nearest = wp;
                }
            });
            if (nearest && nearestDist < 100) {
                document.getElementById('wp-name').textContent = nearest.name;
                document.getElementById('wp-dist').textContent = `${Math.round(nearestDist)}m`;
            } else {
                document.getElementById('wp-name').textContent = '';
                document.getElementById('wp-dist').textContent = '';
            }
            
            // Render
            renderer.clear();
            renderer.render(mainScene, mainCamera);
            labelRenderer.render(mainScene, mainCamera);
        }
        requestAnimationFrame(animate);
        
        window.addEventListener('resize', () => {
            const w = window.innerWidth, h = window.innerHeight;
            mainCamera.aspect = w / h;
            mainCamera.updateProjectionMatrix();
            renderer.setSize(w, h);
            labelRenderer.setSize(w, h);
        });
    </script>
</body>
</html>
```

## Breadcrumb Trail

```javascript
class BreadcrumbTrail {
    constructor(scene, maxPoints = 500) {
        this.scene = scene;
        this.maxPoints = maxPoints;
        this.positions = [];
        this.lastAdded = null;
        this.minDistance = 2; // เพิ่ม point ทุก N units
        
        // Trail line
        const geo = new THREE.BufferGeometry();
        const posArray = new Float32Array(maxPoints * 3);
        geo.setAttribute('position', new THREE.BufferAttribute(posArray, 3));
        geo.setDrawRange(0, 0);
        
        this.geo = geo;
        this.line = new THREE.Line(geo, new THREE.LineBasicMaterial({
            color: 0x4488ff,
            transparent: true,
            opacity: 0.4,
            linewidth: 2,
        }));
        this.line.position.y = 0.1;
        scene.add(this.line);
    }
    
    update(playerPosition) {
        if (this.lastAdded && playerPosition.distanceTo(this.lastAdded) < this.minDistance) return;
        
        this.positions.push(playerPosition.clone());
        if (this.positions.length > this.maxPoints) {
            this.positions.shift();
        }
        
        this.lastAdded = playerPosition.clone();
        
        // Update geometry
        const pos = this.geo.attributes.position.array;
        this.positions.forEach((p, i) => {
            pos[i * 3] = p.x;
            pos[i * 3 + 1] = 0;
            pos[i * 3 + 2] = p.z;
        });
        
        this.geo.attributes.position.needsUpdate = true;
        this.geo.setDrawRange(0, this.positions.length);
        
        // Fade based on age (older = more transparent)
        // สามารถเพิ่ม vertex colors ได้
    }
    
    clear() {
        this.positions = [];
        this.geo.setDrawRange(0, 0);
    }
}
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Advanced Minimap
พัฒนา minimap ให้มีการ zoom in/out ด้วย scroll wheel และมีปุ่ม toggle เพื่อแสดง/ซ่อน labels ของ waypoints

### แบบฝึกหัดที่ 2: GPS Navigation
สร้างระบบ navigation ที่คำนวณ shortest path ระหว่าง waypoints (ใช้ pathfinding อย่างง่าย) และแสดงลูกศรชี้ทิศทางไปหา waypoint ถัดไป

### แบบฝึกหัดที่ 3: Fog of War
เพิ่มระบบ fog of war บน minimap ที่ซ่อนพื้นที่ที่ยังไม่ได้สำรวจ เมื่อ player เดินผ่านพื้นที่ใดก็จะ reveal minimap

### แบบฝึกหัดที่ 4: Multi-Floor Map
สร้าง minimap สำหรับอาคาร multi-floor ที่สามารถสลับชั้นได้ด้วย buttons

## สรุป

Minimap และ navigation systems ต้องการ:
1. **WebGLRenderTarget** สำหรับ render to texture
2. **OrthographicCamera** สำหรับ top-down view
3. **Canvas 2D** สำหรับ radar และ overlay graphics
4. **CSS2DObject** สำหรับ labels ใน world space
5. **Coordinate transformation** สำหรับ world → minimap space

ระบบที่ดีต้องทำงานได้ efficient ใน render loop โดยไม่กระทบ frame rate หลัก
