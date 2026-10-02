# Part 97: 3D Charts & Graphs

## บทนำ

ในบทนี้เราจะสร้าง chart types ที่ซับซ้อนขึ้น ได้แก่ pie/donut chart, radar chart, treemap, bubble chart, timeline chart และระบบ real-time updating chart นอกจากนี้เรายังจะเรียนรู้ color palette system, chart legend และการ export chart

## Pie/Donut Chart ใน 3D

Pie chart ใน 3D ทำได้โดยใช้ `ExtrudeGeometry` กับ `Shape` ที่สร้างจาก arc:

```javascript
function createPieSlice(startAngle, endAngle, options = {}) {
    const {
        innerRadius = 0,
        outerRadius = 100,
        height = 20,
        color = 0x4488ff,
        explodeOffset = 0,
    } = options;
    
    const shape = new THREE.Shape();
    const segments = 64;
    
    // Outer arc
    shape.moveTo(
        Math.cos(startAngle) * outerRadius,
        Math.sin(startAngle) * outerRadius
    );
    
    for (let i = 0; i <= segments; i++) {
        const angle = startAngle + (endAngle - startAngle) * (i / segments);
        shape.lineTo(
            Math.cos(angle) * outerRadius,
            Math.sin(angle) * outerRadius
        );
    }
    
    // Inner arc (hole for donut)
    if (innerRadius > 0) {
        for (let i = segments; i >= 0; i--) {
            const angle = startAngle + (endAngle - startAngle) * (i / segments);
            shape.lineTo(
                Math.cos(angle) * innerRadius,
                Math.sin(angle) * innerRadius
            );
        }
    } else {
        shape.lineTo(0, 0);
    }
    
    shape.closePath();
    
    const extrudeSettings = {
        depth: height,
        bevelEnabled: true,
        bevelThickness: 1,
        bevelSize: 1,
        bevelSegments: 3,
    };
    
    const geo = new THREE.ExtrudeGeometry(shape, extrudeSettings);
    const mat = new THREE.MeshStandardMaterial({
        color,
        roughness: 0.3,
        metalness: 0.2,
    });
    
    const mesh = new THREE.Mesh(geo, mat);
    mesh.rotation.x = -Math.PI / 2;
    
    // Explode: เลื่อน slice ออกจากศูนย์กลาง
    if (explodeOffset > 0) {
        const midAngle = (startAngle + endAngle) / 2;
        mesh.position.set(
            Math.cos(midAngle) * explodeOffset,
            0,
            Math.sin(midAngle) * explodeOffset
        );
    }
    
    // Metadata
    mesh.userData = { startAngle, endAngle, midAngle: (startAngle + endAngle) / 2, originalPos: mesh.position.clone() };
    
    return mesh;
}
```

## Demo หลัก: Interactive Donut Chart with Explosion

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Interactive 3D Donut Chart</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #0a0a1a; overflow: hidden; font-family: 'Segoe UI', Arial, sans-serif; }
        canvas { display: block; }
        
        #legend {
            position: fixed;
            top: 50%; right: 20px;
            transform: translateY(-50%);
            background: rgba(10,10,30,0.9);
            border: 1px solid rgba(100,100,200,0.3);
            border-radius: 12px;
            padding: 16px;
            color: white;
            min-width: 160px;
            z-index: 10;
        }
        #legend h3 {
            font-size: 13px;
            color: #aabbff;
            margin-bottom: 12px;
            border-bottom: 1px solid rgba(255,255,255,0.1);
            padding-bottom: 8px;
        }
        .legend-item {
            display: flex;
            align-items: center;
            gap: 8px;
            margin: 6px 0;
            cursor: pointer;
            padding: 4px 6px;
            border-radius: 6px;
            transition: background 0.15s;
        }
        .legend-item:hover { background: rgba(255,255,255,0.05); }
        .legend-item.active { background: rgba(100,100,255,0.15); }
        .legend-dot {
            width: 12px; height: 12px;
            border-radius: 50%;
            flex-shrink: 0;
        }
        .legend-name { font-size: 12px; flex: 1; }
        .legend-value { font-size: 11px; color: #ffdd44; }
        .legend-pct { font-size: 10px; color: rgba(255,255,255,0.4); }
        
        #center-label {
            position: fixed;
            top: 50%; left: 50%;
            transform: translate(-50%, -50%);
            text-align: center;
            color: white;
            pointer-events: none;
            transition: opacity 0.3s;
        }
        #center-label .total { font-size: 28px; font-weight: bold; color: #ffdd44; }
        #center-label .label { font-size: 12px; color: rgba(255,255,255,0.5); }
        
        #hover-info {
            position: fixed;
            top: 50%; left: 50%;
            transform: translate(-50%, -50%);
            text-align: center;
            color: white;
            pointer-events: none;
            opacity: 0;
            transition: opacity 0.2s;
        }
        #hover-info .hname { font-size: 16px; font-weight: bold; }
        #hover-info .hvalue { font-size: 24px; color: #ffdd44; font-weight: bold; }
        #hover-info .hpct { font-size: 14px; color: rgba(255,255,255,0.6); }
    </style>
</head>
<body>
    <canvas id="canvas"></canvas>
    <div id="legend"><h3>📊 ส่วนแบ่งตลาด</h3></div>
    <div id="center-label">
        <div class="total" id="total-val">100%</div>
        <div class="label">รวมทั้งหมด</div>
    </div>
    <div id="hover-info">
        <div class="hname" id="h-name"></div>
        <div class="hvalue" id="h-value"></div>
        <div class="hpct" id="h-pct"></div>
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
        import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

        const W = window.innerWidth, H = window.innerHeight;
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x0a0a1a);
        
        const camera = new THREE.PerspectiveCamera(50, W / H, 1, 2000);
        camera.position.set(0, 220, 280);
        camera.lookAt(0, 0, 0);
        
        const renderer = new THREE.WebGLRenderer({ canvas: document.getElementById('canvas'), antialias: true });
        renderer.setSize(W, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        renderer.shadowMap.enabled = true;
        
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.target.set(0, 0, 0);
        controls.maxPolarAngle = Math.PI * 0.6;
        
        scene.add(new THREE.AmbientLight(0x334466, 1.5));
        const dirLight = new THREE.DirectionalLight(0xffffff, 2);
        dirLight.position.set(200, 400, 200);
        dirLight.castShadow = true;
        scene.add(dirLight);
        const pLight = new THREE.PointLight(0x4466ff, 1, 500);
        pLight.position.set(0, 200, 0);
        scene.add(pLight);
        
        // Data
        const data = [
            { name: 'Product A', value: 35, color: '#4488ff' },
            { name: 'Product B', value: 25, color: '#ff6644' },
            { name: 'Product C', value: 20, color: '#44ff88' },
            { name: 'Product D', value: 12, color: '#ffdd44' },
            { name: 'Others', value: 8, color: '#aa44ff' },
        ];
        
        const total = data.reduce((sum, d) => sum + d.value, 0);
        
        // Create donut slices
        const slices = [];
        const legend = document.getElementById('legend');
        let currentAngle = 0;
        let hoveredSlice = null;
        
        data.forEach((d, i) => {
            const startAngle = currentAngle;
            const endAngle = currentAngle + (d.value / total) * Math.PI * 2;
            currentAngle = endAngle;
            
            const midAngle = (startAngle + endAngle) / 2;
            
            // Create slice geometry
            const shape = new THREE.Shape();
            const outerR = 120, innerR = 60;
            const segs = 64;
            
            shape.moveTo(Math.cos(startAngle) * outerR, Math.sin(startAngle) * outerR);
            for (let j = 0; j <= segs; j++) {
                const a = startAngle + (endAngle - startAngle) * (j / segs);
                shape.lineTo(Math.cos(a) * outerR, Math.sin(a) * outerR);
            }
            for (let j = segs; j >= 0; j--) {
                const a = startAngle + (endAngle - startAngle) * (j / segs);
                shape.lineTo(Math.cos(a) * innerR, Math.sin(a) * innerR);
            }
            shape.closePath();
            
            const extrudeSettings = { depth: 25, bevelEnabled: true, bevelThickness: 1.5, bevelSize: 1, bevelSegments: 3 };
            const geo = new THREE.ExtrudeGeometry(shape, extrudeSettings);
            
            const color = new THREE.Color(d.color);
            const mat = new THREE.MeshStandardMaterial({
                color,
                roughness: 0.25,
                metalness: 0.3,
                emissive: color.clone().multiplyScalar(0.05),
            });
            
            const mesh = new THREE.Mesh(geo, mat);
            mesh.rotation.x = -Math.PI / 2;
            mesh.castShadow = true;
            mesh.receiveShadow = true;
            mesh.userData = {
                index: i,
                name: d.name,
                value: d.value,
                percent: (d.value / total * 100).toFixed(1),
                midAngle,
                color,
                basePosition: new THREE.Vector3(0, 0, 0),
                explodePosition: new THREE.Vector3(
                    Math.cos(midAngle) * 20,
                    5,
                    Math.sin(midAngle) * 20
                ),
                isExploded: false,
                targetY: 0,
                targetExplode: 0,
            };
            
            scene.add(mesh);
            slices.push(mesh);
            
            // Legend item
            const item = document.createElement('div');
            item.className = 'legend-item';
            item.innerHTML = `
                <div class="legend-dot" style="background: ${d.color}"></div>
                <span class="legend-name">${d.name}</span>
                <div>
                    <div class="legend-value">${d.value}M</div>
                    <div class="legend-pct">${(d.value/total*100).toFixed(1)}%</div>
                </div>
            `;
            item.addEventListener('click', () => toggleExplode(mesh, item));
            item.dataset.index = i;
            legend.appendChild(item);
        });
        
        function toggleExplode(slice, legendItem) {
            const ud = slice.userData;
            ud.isExploded = !ud.isExploded;
            
            document.querySelectorAll('.legend-item').forEach(el => el.classList.remove('active'));
            if (ud.isExploded) {
                legendItem.classList.add('active');
            }
        }
        
        // Raycaster
        const raycaster = new THREE.Raycaster();
        const mouse = new THREE.Vector2();
        
        window.addEventListener('mousemove', (e) => {
            mouse.x = (e.clientX / W) * 2 - 1;
            mouse.y = -(e.clientY / H) * 2 + 1;
        });
        
        window.addEventListener('click', () => {
            raycaster.setFromCamera(mouse, camera);
            const hits = raycaster.intersectObjects(slices);
            if (hits.length > 0) {
                const slice = hits[0].object;
                const i = slice.userData.index;
                const item = document.querySelector(`.legend-item[data-index="${i}"]`);
                if (item) toggleExplode(slice, item);
            }
        });
        
        const centerLabel = document.getElementById('center-label');
        const hoverInfo = document.getElementById('hover-info');
        
        function animate() {
            requestAnimationFrame(animate);
            
            // Raycast for hover
            raycaster.setFromCamera(mouse, camera);
            const hits = raycaster.intersectObjects(slices);
            const newHovered = hits.length > 0 ? hits[0].object : null;
            
            if (newHovered !== hoveredSlice) {
                hoveredSlice = newHovered;
                
                if (hoveredSlice) {
                    const ud = hoveredSlice.userData;
                    document.getElementById('h-name').textContent = ud.name;
                    document.getElementById('h-value').textContent = ud.value + 'M';
                    document.getElementById('h-pct').textContent = ud.percent + '%';
                    hoverInfo.style.opacity = '1';
                    centerLabel.style.opacity = '0';
                } else {
                    hoverInfo.style.opacity = '0';
                    centerLabel.style.opacity = '1';
                }
            }
            
            // Animate slices
            slices.forEach(slice => {
                const ud = slice.userData;
                const targetExplode = ud.isExploded ? 1 : 0;
                ud.targetExplode += (targetExplode - ud.targetExplode) * 0.1;
                
                slice.position.x = ud.explodePosition.x * ud.targetExplode;
                slice.position.z = ud.explodePosition.z * ud.targetExplode;
                
                // Hover lift
                const isHovered = slice === hoveredSlice;
                ud.targetY += ((isHovered ? 8 : 0) - ud.targetY) * 0.15;
                slice.position.y = ud.targetY + ud.explodePosition.y * ud.targetExplode;
                
                // Hover scale
                const targetScale = isHovered ? 1.02 : 1;
                slice.scale.lerp(new THREE.Vector3(targetScale, 1, targetScale), 0.1);
            });
            
            controls.update();
            renderer.render(scene, camera);
        }
        animate();
        
        window.addEventListener('resize', () => {
            const w = window.innerWidth, h = window.innerHeight;
            camera.aspect = w / h;
            camera.updateProjectionMatrix();
            renderer.setSize(w, h);
        });
    </script>
</body>
</html>
```

## Radar/Spider Chart 3D

```javascript
function createRadarChart(data, options = {}) {
    // data: [{axis: 'Speed', value: 0.8}, ...]
    const {
        radius = 100,
        height = 20,
        color = 0x4488ff,
        levels = 5,
    } = options;
    
    const n = data.length;
    const angleStep = (Math.PI * 2) / n;
    
    // Background levels
    for (let level = 1; level <= levels; level++) {
        const r = (radius * level) / levels;
        const points = [];
        for (let i = 0; i <= n; i++) {
            const angle = i * angleStep - Math.PI / 2;
            points.push(new THREE.Vector3(Math.cos(angle) * r, 0, Math.sin(angle) * r));
        }
        const geo = new THREE.BufferGeometry().setFromPoints(points);
        const mat = new THREE.LineBasicMaterial({
            color: 0x334466,
            transparent: true,
            opacity: 0.4,
        });
        scene.add(new THREE.Line(geo, mat));
    }
    
    // Axis lines
    for (let i = 0; i < n; i++) {
        const angle = i * angleStep - Math.PI / 2;
        const end = new THREE.Vector3(
            Math.cos(angle) * radius,
            0,
            Math.sin(angle) * radius
        );
        const geo = new THREE.BufferGeometry().setFromPoints([new THREE.Vector3(0, 0, 0), end]);
        scene.add(new THREE.Line(geo, new THREE.LineBasicMaterial({ color: 0x334466, opacity: 0.6, transparent: true })));
    }
    
    // Data polygon
    const dataPoints = data.map((d, i) => {
        const angle = i * angleStep - Math.PI / 2;
        const r = d.value * radius;
        return new THREE.Vector3(Math.cos(angle) * r, 0, Math.sin(angle) * r);
    });
    dataPoints.push(dataPoints[0].clone()); // close polygon
    
    // Line
    const lineGeo = new THREE.BufferGeometry().setFromPoints(dataPoints);
    const lineMat = new THREE.LineBasicMaterial({ color, linewidth: 2 });
    scene.add(new THREE.Line(lineGeo, lineMat));
    
    // Filled shape (extruded)
    const shape = new THREE.Shape();
    dataPoints.slice(0, -1).forEach((p, i) => {
        if (i === 0) shape.moveTo(p.x, p.z);
        else shape.lineTo(p.x, p.z);
    });
    shape.closePath();
    
    const extrudeGeo = new THREE.ExtrudeGeometry(shape, {
        depth: height,
        bevelEnabled: false,
    });
    const extrudeMat = new THREE.MeshStandardMaterial({
        color,
        transparent: true,
        opacity: 0.6,
        side: THREE.DoubleSide,
    });
    const shapeMesh = new THREE.Mesh(extrudeGeo, extrudeMat);
    shapeMesh.rotation.x = -Math.PI / 2;
    shapeMesh.position.y = 0;
    scene.add(shapeMesh);
    
    // Dots at vertices
    dataPoints.slice(0, -1).forEach(p => {
        const dot = new THREE.Mesh(
            new THREE.SphereGeometry(5, 12, 12),
            new THREE.MeshStandardMaterial({ color, emissive: new THREE.Color(color).multiplyScalar(0.5) })
        );
        dot.position.copy(p);
        scene.add(dot);
    });
}
```

## Treemap 3D (Hierarchical Boxes)

```javascript
function createTreemap3D(hierarchy, scene, options = {}) {
    const {
        width = 500,
        depth = 500,
        maxHeight = 100,
        padding = 4,
    } = options;
    
    // Simple treemap layout (squarified)
    function layoutChildren(node, x0, z0, x1, z1) {
        const children = node.children;
        if (!children || children.length === 0) {
            // Leaf node
            const w = x1 - x0 - padding;
            const d = z1 - z0 - padding;
            if (w <= 0 || d <= 0) return;
            
            const h = (node.value / node.parent.value) * maxHeight;
            const geo = new THREE.BoxGeometry(w, h, d);
            const color = node.color || 0x4488ff;
            const mat = new THREE.MeshStandardMaterial({
                color,
                roughness: 0.4,
                metalness: 0.2,
            });
            const mesh = new THREE.Mesh(geo, mat);
            mesh.position.set(
                x0 + w / 2 + padding / 2 - width / 2,
                h / 2,
                z0 + d / 2 + padding / 2 - depth / 2
            );
            mesh.userData = { ...node, isLeaf: true };
            scene.add(mesh);
            return;
        }
        
        const totalValue = children.reduce((s, c) => s + c.value, 0);
        const totalArea = (x1 - x0) * (z1 - z0);
        let cx = x0, cz = z0;
        
        // Horizontal split
        children.forEach(child => {
            const fraction = child.value / totalValue;
            const cx1 = cx + (x1 - x0) * fraction;
            layoutChildren(child, cx + padding, cz + padding, cx1 - padding, z1 - padding);
            cx = cx1;
        });
    }
    
    // Add parent references
    function addParents(node, parent = null) {
        node.parent = parent;
        if (node.children) node.children.forEach(c => addParents(c, node));
    }
    addParents(hierarchy);
    
    layoutChildren(hierarchy, 0, 0, width, depth);
}

// Example usage
const treeData = {
    name: 'root',
    value: 100,
    children: [
        {
            name: 'Tech',
            value: 45,
            children: [
                { name: 'Software', value: 25, color: 0x4488ff },
                { name: 'Hardware', value: 12, color: 0x2266dd },
                { name: 'Services', value: 8, color: 0x1144aa },
            ],
        },
        {
            name: 'Finance',
            value: 35,
            children: [
                { name: 'Banking', value: 20, color: 0xff8844 },
                { name: 'Insurance', value: 15, color: 0xdd6622 },
            ],
        },
        {
            name: 'Healthcare',
            value: 20,
            children: [
                { name: 'Pharma', value: 12, color: 0x44ff88 },
                { name: 'Medical', value: 8, color: 0x22dd66 },
            ],
        },
    ],
};
```

## Real-Time Updating Chart

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Real-Time 3D Chart</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #080818; overflow: hidden; }
        canvas { display: block; }
        #stats {
            position: fixed; top: 16px; left: 16px;
            color: rgba(255,255,255,0.7);
            font-family: 'Courier New', monospace;
            font-size: 13px; z-index: 10;
        }
        .stat { margin: 3px 0; }
        .val { color: #4488ff; }
    </style>
</head>
<body>
    <canvas id="canvas"></canvas>
    <div id="stats">
        <div class="stat">Current: <span class="val" id="s-current">-</span></div>
        <div class="stat">High: <span class="val" id="s-high" style="color:#44ff88">-</span></div>
        <div class="stat">Low: <span class="val" id="s-low" style="color:#ff4444">-</span></div>
        <div class="stat">Change: <span class="val" id="s-change">-</span></div>
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
        import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

        const W = window.innerWidth, H = window.innerHeight;
        const MAX_POINTS = 200;
        
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x080818);
        
        const camera = new THREE.PerspectiveCamera(60, W / H, 1, 5000);
        camera.position.set(0, 150, 400);
        
        const renderer = new THREE.WebGLRenderer({ canvas: document.getElementById('canvas'), antialias: true });
        renderer.setSize(W, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.target.set(0, 80, 0);
        
        scene.add(new THREE.AmbientLight(0x334466, 2));
        const dirLight = new THREE.DirectionalLight(0x8899ff, 2);
        dirLight.position.set(0, 300, 200);
        scene.add(dirLight);
        
        // Grid
        const grid = new THREE.GridHelper(600, 30, 0x112233, 0x0a1a2a);
        scene.add(grid);
        
        // Multiple time series
        const series = [
            { name: 'AAPL', color: new THREE.Color(0x4488ff), values: [], zOffset: 0, basePrice: 180 },
            { name: 'GOOG', color: new THREE.Color(0xff6644), values: [], zOffset: 60, basePrice: 140 },
            { name: 'META', color: new THREE.Color(0x44ff88), values: [], zOffset: 120, basePrice: 360 },
        ];
        
        // Initialize with random data
        series.forEach(s => {
            let price = s.basePrice;
            for (let i = 0; i < MAX_POINTS; i++) {
                price += (Math.random() - 0.5) * 4;
                price = Math.max(s.basePrice * 0.7, Math.min(s.basePrice * 1.3, price));
                s.values.push(price);
            }
        });
        
        // Create line geometries
        series.forEach(s => {
            const positions = new Float32Array(MAX_POINTS * 3);
            s.geo = new THREE.BufferGeometry();
            s.geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
            s.geo.setDrawRange(0, MAX_POINTS);
            
            s.line = new THREE.Line(
                s.geo,
                new THREE.LineBasicMaterial({ color: s.color, linewidth: 2 })
            );
            scene.add(s.line);
            
            // Area (flat plane under line)
            const areaPositions = new Float32Array(MAX_POINTS * 2 * 3);
            s.areaGeo = new THREE.BufferGeometry();
            s.areaGeo.setAttribute('position', new THREE.BufferAttribute(areaPositions, 3));
            s.areaGeo.setDrawRange(0, MAX_POINTS * 2);
            
            const indices = [];
            for (let i = 0; i < MAX_POINTS - 1; i++) {
                indices.push(i * 2, i * 2 + 1, i * 2 + 2);
                indices.push(i * 2 + 1, i * 2 + 3, i * 2 + 2);
            }
            s.areaGeo.setIndex(indices);
            
            const areaMat = new THREE.MeshBasicMaterial({
                color: s.color,
                transparent: true,
                opacity: 0.08,
                side: THREE.DoubleSide,
            });
            const areaMesh = new THREE.Mesh(s.areaGeo, areaMat);
            scene.add(areaMesh);
            s.areaMesh = areaMesh;
        });
        
        function updateGeometry() {
            const xRange = 500;
            const yMin = 50, yMax = 250;
            
            series.forEach(s => {
                const allMin = Math.min(...s.values);
                const allMax = Math.max(...s.values);
                
                const pos = s.geo.attributes.position.array;
                const areaPos = s.areaGeo.attributes.position.array;
                
                s.values.forEach((v, i) => {
                    const x = (i / (MAX_POINTS - 1)) * xRange - xRange / 2;
                    const y = ((v - allMin) / (allMax - allMin)) * (yMax - yMin) + yMin;
                    const z = s.zOffset - 90;
                    
                    pos[i * 3] = x;
                    pos[i * 3 + 1] = y;
                    pos[i * 3 + 2] = z;
                    
                    areaPos[i * 6] = x;
                    areaPos[i * 6 + 1] = y;
                    areaPos[i * 6 + 2] = z;
                    areaPos[i * 6 + 3] = x;
                    areaPos[i * 6 + 4] = 0;
                    areaPos[i * 6 + 5] = z;
                });
                
                s.geo.attributes.position.needsUpdate = true;
                s.areaGeo.attributes.position.needsUpdate = true;
                s.geo.computeBoundingSphere();
                s.areaGeo.computeBoundingSphere();
            });
        }
        
        updateGeometry();
        
        // Real-time update
        let lastUpdate = 0;
        const UPDATE_INTERVAL = 200; // ms
        
        function addDataPoint() {
            series.forEach(s => {
                const last = s.values[s.values.length - 1];
                const newVal = last + (Math.random() - 0.48) * 3;
                const clamped = Math.max(s.basePrice * 0.7, Math.min(s.basePrice * 1.3, newVal));
                s.values.push(clamped);
                s.values.shift();
            });
            
            updateGeometry();
            
            // Update stats for first series
            const s = series[0];
            const current = s.values[s.values.length - 1];
            const prev = s.values[s.values.length - 2];
            const high = Math.max(...s.values);
            const low = Math.min(...s.values);
            const change = ((current - prev) / prev * 100).toFixed(2);
            
            document.getElementById('s-current').textContent = current.toFixed(2);
            document.getElementById('s-high').textContent = high.toFixed(2);
            document.getElementById('s-low').textContent = low.toFixed(2);
            document.getElementById('s-change').textContent = (change > 0 ? '+' : '') + change + '%';
            document.getElementById('s-change').style.color = change >= 0 ? '#44ff88' : '#ff4444';
        }
        
        function animate() {
            requestAnimationFrame(animate);
            
            const now = performance.now();
            if (now - lastUpdate > UPDATE_INTERVAL) {
                addDataPoint();
                lastUpdate = now;
            }
            
            controls.update();
            renderer.render(scene, camera);
        }
        animate();
        
        window.addEventListener('resize', () => {
            const w = window.innerWidth, h = window.innerHeight;
            camera.aspect = w / h;
            camera.updateProjectionMatrix();
            renderer.setSize(w, h);
        });
    </script>
</body>
</html>
```

## Chart Color Palette System

```javascript
class ChartPalette {
    constructor(theme = 'vibrant') {
        this.themes = {
            vibrant: ['#4488ff', '#ff6644', '#44ff88', '#ffdd44', '#aa44ff', '#ff4488', '#44ffdd', '#ff8800'],
            pastel: ['#88aaff', '#ffaa88', '#88ffaa', '#ffeeaa', '#cc88ff', '#ffaacc', '#aaffee', '#ffcc88'],
            monochrome: ['#ffffff', '#cccccc', '#999999', '#666666', '#444444', '#222222', '#888888', '#bbbbbb'],
            cool: ['#4488ff', '#44aaff', '#44ddff', '#44ffee', '#44ffbb', '#4488dd', '#2266aa', '#116688'],
            warm: ['#ffdd44', '#ff8822', '#ff4422', '#ff2244', '#ff44aa', '#ff8844', '#ffbb22', '#dd6622'],
        };
        this.current = this.themes[theme] || this.themes.vibrant;
    }
    
    getColor(index) {
        return new THREE.Color(this.current[index % this.current.length]);
    }
    
    getGradient(t, colorA, colorB) {
        return new THREE.Color(colorA).lerp(new THREE.Color(colorB), t);
    }
    
    getSequential(t) {
        // Blue to Red gradient
        return new THREE.Color(
            Math.max(0, t * 2 - 1),
            Math.max(0, 1 - Math.abs(t * 2 - 1)),
            Math.max(0, 1 - t * 2)
        );
    }
}
```

## Export Chart as Image

```javascript
function exportChartAsImage(renderer, filename = 'chart.png') {
    // ต้อง render ก่อน 1 frame
    renderer.render(scene, camera);
    
    const dataURL = renderer.domElement.toDataURL('image/png');
    
    const link = document.createElement('a');
    link.download = filename;
    link.href = dataURL;
    link.click();
}

// ใน HTML
document.getElementById('export-btn').addEventListener('click', () => {
    exportChartAsImage(renderer, 'my-3d-chart.png');
});
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Budget Donut Chart
สร้าง donut chart ที่แสดงงบประมาณแผนก 6 แผนก มีการ explode เมื่อ hover และแสดงตัวเลขตรงกลาง

### แบบฝึกหัดที่ 2: Skills Radar Chart
สร้าง radar chart ที่แสดง skills ของ developer (Frontend, Backend, DevOps, Database, Design, Testing) เปรียบเทียบ 2 คน

### แบบฝึกหัดที่ 3: Hierarchical Treemap
สร้าง treemap 3D ที่แสดง market cap ของบริษัทใน S&P 500 (จำลอง) จัดกลุ่มเป็น sector

### แบบฝึกหัดที่ 4: Crypto Price Monitor
สร้าง real-time price chart สำหรับ crypto 3 ตัว (จำลองข้อมูล) มี high/low indicators และ export button

## สรุป

Chart types ใน 3D แต่ละแบบมีลักษณะพิเศษ:
- **Pie/Donut**: ExtrudeGeometry + Shape = 3D slices ที่สวยงาม
- **Radar**: Regular polygon + shape fill = spider web visual
- **Treemap**: Hierarchical boxes = nested visualization
- **Real-time**: BufferGeometry.attributes needsUpdate = efficient updates

การออกแบบ color palette ที่ดีและ consistent เป็นสิ่งสำคัญมากสำหรับ chart ที่น่าใช้งาน
