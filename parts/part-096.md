# Part 96: Data Visualization ใน 3D

## บทนำ

การนำข้อมูลมาแสดงผลใน 3D เป็นหนึ่งในงานที่น่าสนใจที่สุดของ Three.js การรวม D3.js กับ Three.js ทำให้เราได้รับประโยชน์จากทั้ง data processing ของ D3 และ rendering ของ Three.js ในบทนี้เราจะสร้าง visualizations ต่างๆ ตั้งแต่ bar chart ไปถึง globe ที่มีข้อมูลจริง

## D3.js + Three.js Integration Pattern

D3.js เก่งด้าน data manipulation, scales, และ layouts แต่ไม่ใช้สำหรับ 3D rendering ส่วน Three.js เก่งด้าน 3D rendering แต่ไม่มี built-in data tools วิธีที่ดีที่สุดคือใช้ D3 สำหรับการคำนวณและ Three.js สำหรับการ render

```javascript
// D3 สำหรับ data processing
const data = [30, 80, 45, 60, 25, 90, 55];

const xScale = d3.scaleBand()
    .domain(d3.range(data.length))
    .range([0, 600])
    .padding(0.2);

const yScale = d3.scaleLinear()
    .domain([0, d3.max(data)])
    .range([0, 300]);

const colorScale = d3.scaleSequential(d3.interpolateViridis)
    .domain([0, d3.max(data)]);

// Three.js สำหรับ render
data.forEach((value, i) => {
    const height = yScale(value);
    const width = xScale.bandwidth();
    
    const geo = new THREE.BoxGeometry(width, height, width);
    const color = new THREE.Color(colorScale(value));
    const mat = new THREE.MeshStandardMaterial({ color });
    const bar = new THREE.Mesh(geo, mat);
    
    bar.position.x = xScale(i) + width / 2 - 300;
    bar.position.y = height / 2;
    scene.add(bar);
});
```

## 3D Bar Chart

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>3D Bar Chart</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #0a0a1a; overflow: hidden; font-family: Arial, sans-serif; }
        canvas { display: block; }
        #tooltip {
            position: fixed;
            background: rgba(10,10,30,0.95);
            border: 1px solid rgba(100,120,255,0.5);
            border-radius: 8px;
            padding: 10px 14px;
            color: white;
            font-size: 13px;
            pointer-events: none;
            display: none;
            z-index: 100;
        }
        #tooltip .label { color: #88aaff; font-size: 11px; }
        #tooltip .value { font-size: 18px; font-weight: bold; color: #ffdd44; }
        #controls {
            position: fixed;
            top: 16px; right: 16px;
            display: flex; gap: 8px;
            z-index: 10;
        }
        .btn {
            background: rgba(30,30,60,0.8);
            border: 1px solid rgba(100,100,200,0.4);
            color: white; padding: 6px 14px;
            border-radius: 6px; cursor: pointer;
            font-size: 12px; transition: background 0.2s;
        }
        .btn:hover { background: rgba(50,50,100,0.8); }
    </style>
</head>
<body>
    <canvas id="canvas"></canvas>
    <div id="tooltip">
        <div class="label" id="tt-label">Month</div>
        <div class="value" id="tt-value">0</div>
    </div>
    <div id="controls">
        <button class="btn" id="btn-animate">🔄 Animate Data</button>
        <button class="btn" id="btn-reset">↺ Reset</button>
    </div>

    <script type="importmap">
    {
        "imports": {
            "three": "https://cdn.jsdelivr.net/npm/three@0.165.0/build/three.module.js",
            "three/examples/jsm/": "https://cdn.jsdelivr.net/npm/three@0.165.0/examples/jsm/",
            "d3": "https://cdn.jsdelivr.net/npm/d3@7/+esm"
        }
    }
    </script>
    <script type="module">
        import * as THREE from 'three';
        import * as d3 from 'd3';
        import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
        import { CSS2DRenderer, CSS2DObject } from 'three/examples/jsm/renderers/CSS2DRenderer.js';

        const W = window.innerWidth, H = window.innerHeight;
        
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x0a0a1a);
        scene.add(new THREE.GridHelper(800, 20, 0x1a1a3a, 0x111130));
        
        const camera = new THREE.PerspectiveCamera(55, W / H, 1, 3000);
        camera.position.set(0, 250, 500);
        
        const renderer = new THREE.WebGLRenderer({ canvas: document.getElementById('canvas'), antialias: true });
        renderer.setSize(W, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        renderer.shadowMap.enabled = true;
        
        const labelRenderer = new CSS2DRenderer();
        labelRenderer.setSize(W, H);
        labelRenderer.domElement.style.cssText = 'position:absolute;top:0;left:0;pointer-events:none;';
        document.body.appendChild(labelRenderer.domElement);
        
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.target.set(0, 100, 0);
        
        scene.add(new THREE.AmbientLight(0x334466, 1.5));
        const dirLight = new THREE.DirectionalLight(0xffffff, 2);
        dirLight.position.set(300, 500, 200);
        dirLight.castShadow = true;
        scene.add(dirLight);
        
        // Data
        const months = ['ม.ค.', 'ก.พ.', 'มี.ค.', 'เม.ย.', 'พ.ค.', 'มิ.ย.', 'ก.ค.', 'ส.ค.', 'ก.ย.', 'ต.ค.', 'พ.ย.', 'ธ.ค.'];
        let salesData = [45000, 52000, 48000, 70000, 65000, 58000, 72000, 81000, 67000, 75000, 89000, 95000];
        
        const barWidth = 45;
        const barSpacing = 60;
        const totalWidth = months.length * barSpacing;
        
        // Scales
        const yScale = d3.scaleLinear()
            .domain([0, d3.max(salesData)])
            .range([0, 300]);
        
        const colorScale = d3.scaleSequential()
            .domain([0, d3.max(salesData)])
            .interpolator(d3.interpolateRdYlBu);
        
        const bars = [];
        const labelObjects = [];
        
        months.forEach((month, i) => {
            const x = i * barSpacing - totalWidth / 2 + barSpacing / 2;
            const value = salesData[i];
            const height = yScale(value);
            const colorHex = colorScale(value);
            const color = new THREE.Color(colorHex);
            
            // Bar mesh
            const geo = new THREE.BoxGeometry(barWidth, height, barWidth);
            const mat = new THREE.MeshStandardMaterial({
                color,
                roughness: 0.3,
                metalness: 0.2,
                emissive: color.clone().multiplyScalar(0.1),
            });
            const bar = new THREE.Mesh(geo, mat);
            bar.position.set(x, height / 2, 0);
            bar.castShadow = true;
            bar.receiveShadow = true;
            bar.userData = { index: i, value, month, originalHeight: height, targetHeight: height };
            scene.add(bar);
            bars.push(bar);
            
            // Month label (CSS2D)
            const labelEl = document.createElement('div');
            labelEl.textContent = month;
            labelEl.style.cssText = `
                color: rgba(150,170,255,0.7);
                font-size: 11px;
                font-family: Arial;
                white-space: nowrap;
            `;
            const labelObj = new CSS2DObject(labelEl);
            labelObj.position.set(x, -15, 0);
            scene.add(labelObj);
            labelObjects.push(labelObj);
            
            // Value label on top
            const valueEl = document.createElement('div');
            valueEl.textContent = `฿${(value/1000).toFixed(0)}k`;
            valueEl.style.cssText = `
                color: rgba(255,220,100,0.8);
                font-size: 10px;
                font-family: Arial;
                white-space: nowrap;
            `;
            const valueObj = new CSS2DObject(valueEl);
            valueObj.position.set(x, height + 8, 0);
            scene.add(valueObj);
            bar.userData.valueLabel = valueObj;
        });
        
        // Platform
        const platform = new THREE.Mesh(
            new THREE.BoxGeometry(totalWidth + 80, 5, barWidth + 40),
            new THREE.MeshStandardMaterial({ color: 0x1a1a3a, roughness: 0.8 })
        );
        platform.position.y = -2.5;
        platform.receiveShadow = true;
        scene.add(platform);
        
        // Raycaster for hover
        const raycaster = new THREE.Raycaster();
        const mouse = new THREE.Vector2();
        const tooltip = document.getElementById('tooltip');
        let hoveredBar = null;
        
        window.addEventListener('mousemove', (e) => {
            mouse.x = (e.clientX / W) * 2 - 1;
            mouse.y = -(e.clientY / H) * 2 + 1;
            
            tooltip.style.left = (e.clientX + 15) + 'px';
            tooltip.style.top = (e.clientY - 10) + 'px';
        });
        
        // Animate to new data
        function animateToData(newData) {
            const newMax = d3.max(newData);
            bars.forEach((bar, i) => {
                const newHeight = yScale(newData[i]);
                const newColorHex = colorScale(newData[i]);
                bar.userData.targetHeight = newHeight;
                bar.userData.targetColor = new THREE.Color(newColorHex);
                bar.userData.value = newData[i];
            });
        }
        
        document.getElementById('btn-animate').addEventListener('click', () => {
            const newData = months.map(() => Math.floor(30000 + Math.random() * 80000));
            animateToData(newData);
        });
        
        document.getElementById('btn-reset').addEventListener('click', () => {
            animateToData(salesData);
        });
        
        function animate() {
            requestAnimationFrame(animate);
            
            // Raycast hover
            raycaster.setFromCamera(mouse, camera);
            const hits = raycaster.intersectObjects(bars);
            
            if (hits.length > 0) {
                const bar = hits[0].object;
                if (bar !== hoveredBar) {
                    if (hoveredBar) hoveredBar.scale.z = 1;
                    hoveredBar = bar;
                    bar.scale.z = 1.1;
                    document.getElementById('tt-label').textContent = bar.userData.month;
                    document.getElementById('tt-value').textContent = `฿${bar.userData.value.toLocaleString()}`;
                    tooltip.style.display = 'block';
                }
            } else if (hoveredBar) {
                hoveredBar.scale.z = 1;
                hoveredBar = null;
                tooltip.style.display = 'none';
            }
            
            // Animate bar heights
            bars.forEach(bar => {
                if (bar.userData.targetHeight !== undefined) {
                    const curr = bar.scale.y * bar.geometry.parameters.height;
                    const target = bar.userData.targetHeight;
                    const newHeight = curr + (target - curr) * 0.08;
                    const factor = newHeight / bar.geometry.parameters.height;
                    bar.scale.y = factor;
                    bar.position.y = newHeight / 2;
                    
                    // Update value label position
                    if (bar.userData.valueLabel) {
                        bar.userData.valueLabel.position.y = newHeight + 8;
                    }
                    
                    if (bar.userData.targetColor) {
                        bar.material.color.lerp(bar.userData.targetColor, 0.05);
                    }
                }
            });
            
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

## 3D Scatter Plot

```javascript
function createScatterPlot(data, scene) {
    // data: [{x, y, z, category, label, value}]
    
    const categories = [...new Set(data.map(d => d.category))];
    const colors = d3.schemeTableau10;
    const colorMap = Object.fromEntries(categories.map((c, i) => [c, colors[i % colors.length]]));
    
    const xExtent = d3.extent(data, d => d.x);
    const yExtent = d3.extent(data, d => d.y);
    const zExtent = d3.extent(data, d => d.z);
    const valueExtent = d3.extent(data, d => d.value || 1);
    
    const xScale = d3.scaleLinear().domain(xExtent).range([-250, 250]);
    const yScale = d3.scaleLinear().domain(yExtent).range([0, 300]);
    const zScale = d3.scaleLinear().domain(zExtent).range([-200, 200]);
    const sizeScale = d3.scaleSqrt().domain(valueExtent).range([3, 20]);
    
    const points = [];
    data.forEach(d => {
        const radius = sizeScale(d.value || 1);
        const geo = new THREE.SphereGeometry(radius, 16, 16);
        const color = new THREE.Color(colorMap[d.category]);
        const mat = new THREE.MeshStandardMaterial({
            color,
            roughness: 0.3,
            metalness: 0.5,
            transparent: true,
            opacity: 0.85,
        });
        const mesh = new THREE.Mesh(geo, mat);
        mesh.position.set(
            xScale(d.x),
            yScale(d.y),
            zScale(d.z)
        );
        mesh.userData = { data: d, category: d.category };
        scene.add(mesh);
        points.push(mesh);
    });
    
    // Axes
    const axisLines = [
        { start: new THREE.Vector3(-250, 0, 0), end: new THREE.Vector3(250, 0, 0), color: 0xff4444 }, // X
        { start: new THREE.Vector3(0, 0, 0), end: new THREE.Vector3(0, 300, 0), color: 0x44ff44 }, // Y
        { start: new THREE.Vector3(0, 0, -200), end: new THREE.Vector3(0, 0, 200), color: 0x4444ff }, // Z
    ];
    
    axisLines.forEach(({ start, end, color }) => {
        const points = [start, end];
        const geo = new THREE.BufferGeometry().setFromPoints(points);
        const mat = new THREE.LineBasicMaterial({ color });
        scene.add(new THREE.Line(geo, mat));
    });
    
    return points;
}
```

## 3D Line Chart

```javascript
function createLineChart(timeSeriesData, scene) {
    // timeSeriesData: [{time, value, label}][]  (multiple series)
    
    const allValues = timeSeriesData.flat().map(d => d.value);
    const xScale = d3.scaleLinear()
        .domain([0, d3.max(timeSeriesData, s => s.length) - 1])
        .range([-300, 300]);
    const yScale = d3.scaleLinear()
        .domain(d3.extent(allValues))
        .range([0, 200]);
    
    const seriesColors = [0x4488ff, 0xff6644, 0x44ff88, 0xffdd44];
    
    timeSeriesData.forEach((series, si) => {
        const points = series.map((d, i) =>
            new THREE.Vector3(xScale(i), yScale(d.value), si * 40 - (timeSeriesData.length - 1) * 20)
        );
        
        // Line
        const curve = new THREE.CatmullRomCurve3(points);
        const linePoints = curve.getPoints(series.length * 4);
        const geo = new THREE.BufferGeometry().setFromPoints(linePoints);
        const mat = new THREE.LineBasicMaterial({ color: seriesColors[si], linewidth: 2 });
        scene.add(new THREE.Line(geo, mat));
        
        // Area fill
        const areaPoints = [];
        linePoints.forEach(p => {
            areaPoints.push(p.clone());
            areaPoints.push(new THREE.Vector3(p.x, 0, p.z));
        });
        
        // Sphere at data points
        series.forEach((d, i) => {
            const sphere = new THREE.Mesh(
                new THREE.SphereGeometry(4, 12, 12),
                new THREE.MeshStandardMaterial({ color: seriesColors[si] })
            );
            sphere.position.copy(points[i]);
            sphere.userData = { value: d.value, label: d.label, time: d.time };
            scene.add(sphere);
        });
    });
}
```

## Force-Directed Network Graph (3D)

```javascript
class NetworkGraph3D {
    constructor(scene) {
        this.scene = scene;
        this.nodes = [];
        this.edges = [];
        this.meshes = new Map();
        this.lineMeshes = [];
    }
    
    addNode(id, data = {}) {
        const radius = data.size || 10;
        const geo = new THREE.SphereGeometry(radius, 20, 20);
        const mat = new THREE.MeshStandardMaterial({
            color: data.color || 0x4488ff,
            roughness: 0.3,
            metalness: 0.5,
        });
        const mesh = new THREE.Mesh(geo, mat);
        
        // Random initial position
        mesh.position.set(
            (Math.random() - 0.5) * 400,
            (Math.random() - 0.5) * 400,
            (Math.random() - 0.5) * 400
        );
        mesh.userData = { id, velocity: new THREE.Vector3(), ...data };
        
        this.scene.add(mesh);
        this.nodes.push(mesh);
        this.meshes.set(id, mesh);
        return mesh;
    }
    
    addEdge(fromId, toId, weight = 1) {
        const from = this.meshes.get(fromId);
        const to = this.meshes.get(toId);
        if (!from || !to) return;
        
        const points = [from.position, to.position];
        const geo = new THREE.BufferGeometry().setFromPoints(points);
        const mat = new THREE.LineBasicMaterial({
            color: 0x334466,
            transparent: true,
            opacity: 0.5 * weight,
        });
        const line = new THREE.Line(geo, mat);
        this.scene.add(line);
        this.edges.push({ from, to, line, weight });
        this.lineMeshes.push(line);
    }
    
    // Force-directed layout step
    tick() {
        const repulsion = 5000;
        const attraction = 0.01;
        const damping = 0.85;
        
        // Repulsion between all nodes
        for (let i = 0; i < this.nodes.length; i++) {
            for (let j = i + 1; j < this.nodes.length; j++) {
                const ni = this.nodes[i];
                const nj = this.nodes[j];
                
                const diff = new THREE.Vector3().subVectors(ni.position, nj.position);
                const dist = Math.max(diff.length(), 10);
                const force = repulsion / (dist * dist);
                diff.normalize().multiplyScalar(force);
                
                ni.userData.velocity.add(diff);
                nj.userData.velocity.sub(diff);
            }
        }
        
        // Attraction along edges
        this.edges.forEach(({ from, to, weight }) => {
            const diff = new THREE.Vector3().subVectors(to.position, from.position);
            const dist = diff.length();
            const force = attraction * dist * weight;
            diff.normalize().multiplyScalar(force);
            
            from.userData.velocity.add(diff);
            to.userData.velocity.sub(diff);
        });
        
        // Update positions
        this.nodes.forEach(node => {
            node.userData.velocity.multiplyScalar(damping);
            node.position.add(node.userData.velocity);
            
            // Boundary
            const limit = 300;
            node.position.clampLength(0, limit);
        });
        
        // Update edge lines
        this.edges.forEach(({ from, to, line }) => {
            const pos = line.geometry.attributes.position;
            pos.setXYZ(0, from.position.x, from.position.y, from.position.z);
            pos.setXYZ(1, to.position.x, to.position.y, to.position.z);
            pos.needsUpdate = true;
        });
    }
}
```

## World Globe with Data

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>3D World Globe with Data</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000010; overflow: hidden; }
        canvas { display: block; }
        #info {
            position: fixed; bottom: 20px; left: 50%;
            transform: translateX(-50%);
            color: rgba(255,255,255,0.5);
            font-family: Arial; font-size: 13px;
            pointer-events: none; z-index: 10;
        }
    </style>
</head>
<body>
    <canvas id="canvas"></canvas>
    <div id="info">Drag to rotate | Scroll to zoom</div>

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
        scene.background = new THREE.Color(0x000010);
        
        const camera = new THREE.PerspectiveCamera(45, W / H, 0.1, 1000);
        camera.position.z = 3;
        
        const renderer = new THREE.WebGLRenderer({ canvas: document.getElementById('canvas'), antialias: true });
        renderer.setSize(W, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.minDistance = 1.5;
        controls.maxDistance = 8;
        
        // Stars
        const starsGeo = new THREE.BufferGeometry();
        const starPositions = new Float32Array(10000 * 3);
        for (let i = 0; i < starPositions.length; i += 3) {
            const theta = Math.random() * Math.PI * 2;
            const phi = Math.acos(2 * Math.random() - 1);
            const r = 100 + Math.random() * 200;
            starPositions[i] = r * Math.sin(phi) * Math.cos(theta);
            starPositions[i+1] = r * Math.sin(phi) * Math.sin(theta);
            starPositions[i+2] = r * Math.cos(phi);
        }
        starsGeo.setAttribute('position', new THREE.BufferAttribute(starPositions, 3));
        scene.add(new THREE.Points(starsGeo, new THREE.PointsMaterial({ color: 0xffffff, size: 0.3, sizeAttenuation: false })));
        
        // Globe
        const globeGeo = new THREE.SphereGeometry(1, 64, 64);
        
        // Create globe texture procedurally
        const canvas2d = document.createElement('canvas');
        canvas2d.width = 1024; canvas2d.height = 512;
        const ctx = canvas2d.getContext('2d');
        
        // Ocean
        ctx.fillStyle = '#0a2040';
        ctx.fillRect(0, 0, 1024, 512);
        
        // Rough continent outlines (simplified)
        ctx.fillStyle = '#1a4a2a';
        // North America
        ctx.fillRect(100, 80, 180, 200);
        // South America
        ctx.fillRect(150, 280, 120, 160);
        // Europe
        ctx.fillRect(420, 60, 100, 130);
        // Africa
        ctx.fillRect(420, 190, 120, 220);
        // Asia
        ctx.fillRect(520, 50, 280, 220);
        // Australia
        ctx.fillRect(700, 310, 100, 80);
        
        const globeTexture = new THREE.CanvasTexture(canvas2d);
        const globeMat = new THREE.MeshPhongMaterial({
            map: globeTexture,
            specular: new THREE.Color(0x224488),
            shininess: 25,
        });
        const globe = new THREE.Mesh(globeGeo, globeMat);
        scene.add(globe);
        
        // Atmosphere
        const atmGeo = new THREE.SphereGeometry(1.02, 64, 64);
        const atmMat = new THREE.MeshPhongMaterial({
            color: 0x4488ff,
            transparent: true,
            opacity: 0.1,
            side: THREE.FrontSide,
        });
        scene.add(new THREE.Mesh(atmGeo, atmMat));
        
        // Lighting
        scene.add(new THREE.AmbientLight(0x111133, 2));
        const sunLight = new THREE.DirectionalLight(0xfff5dd, 3);
        sunLight.position.set(5, 3, 5);
        scene.add(sunLight);
        
        // Convert lat/lon to 3D position
        function latLonToVec3(lat, lon, radius = 1.01) {
            const phi = (90 - lat) * (Math.PI / 180);
            const theta = (lon + 180) * (Math.PI / 180);
            return new THREE.Vector3(
                -radius * Math.sin(phi) * Math.cos(theta),
                radius * Math.cos(phi),
                radius * Math.sin(phi) * Math.sin(theta)
            );
        }
        
        // City markers
        const cities = [
            { name: 'New York', lat: 40.7, lon: -74.0, value: 8.3 },
            { name: 'London', lat: 51.5, lon: -0.1, value: 9.0 },
            { name: 'Tokyo', lat: 35.7, lon: 139.7, value: 13.9 },
            { name: 'Bangkok', lat: 13.7, lon: 100.5, value: 10.5 },
            { name: 'Sydney', lat: -33.9, lon: 151.2, value: 5.3 },
            { name: 'Dubai', lat: 25.2, lon: 55.3, value: 3.3 },
            { name: 'Paris', lat: 48.9, lon: 2.3, value: 2.1 },
            { name: 'Mumbai', lat: 19.1, lon: 72.9, value: 20.7 },
        ];
        
        cities.forEach(city => {
            const pos = latLonToVec3(city.lat, city.lon);
            
            // Marker spike
            const height = city.value * 0.01;
            const spikeGeo = new THREE.CylinderGeometry(0.003, 0.003, height, 8);
            const spikeMat = new THREE.MeshBasicMaterial({ color: 0xff4444 });
            const spike = new THREE.Mesh(spikeGeo, spikeMat);
            
            // Orient spike radially
            spike.position.copy(pos).multiplyScalar(1 + height / 2);
            spike.lookAt(pos.clone().multiplyScalar(10));
            spike.rotateX(Math.PI / 2);
            scene.add(spike);
            
            // Dot at base
            const dotGeo = new THREE.SphereGeometry(0.008, 8, 8);
            const dot = new THREE.Mesh(dotGeo, new THREE.MeshBasicMaterial({ color: 0xff6644 }));
            dot.position.copy(pos);
            scene.add(dot);
        });
        
        // Flight arcs
        const routes = [
            { from: { lat: 13.7, lon: 100.5 }, to: { lat: 35.7, lon: 139.7 } }, // BKK-TKY
            { from: { lat: 13.7, lon: 100.5 }, to: { lat: 51.5, lon: -0.1 } }, // BKK-LDN
            { from: { lat: 40.7, lon: -74.0 }, to: { lat: 51.5, lon: -0.1 } }, // NYC-LDN
            { from: { lat: 35.7, lon: 139.7 }, to: { lat: 40.7, lon: -74.0 } }, // TKY-NYC
        ];
        
        routes.forEach(route => {
            const start = latLonToVec3(route.from.lat, route.from.lon);
            const end = latLonToVec3(route.to.lat, route.to.lon);
            
            // Create arc via control point above midpoint
            const mid = start.clone().add(end).multiplyScalar(0.5).normalize().multiplyScalar(1.4);
            
            const curve = new THREE.QuadraticBezierCurve3(start, mid, end);
            const points = curve.getPoints(50);
            
            const arcGeo = new THREE.BufferGeometry().setFromPoints(points);
            const arcMat = new THREE.LineBasicMaterial({
                color: 0x4488ff,
                transparent: true,
                opacity: 0.6,
            });
            scene.add(new THREE.Line(arcGeo, arcMat));
            
            // Animated dot along arc
            const dotGeo = new THREE.SphereGeometry(0.01, 8, 8);
            const dot = new THREE.Mesh(dotGeo, new THREE.MeshBasicMaterial({ color: 0xffffff }));
            scene.add(dot);
            
            route.curve = curve;
            route.dot = dot;
            route.t = Math.random();
        });
        
        function animate() {
            requestAnimationFrame(animate);
            const t = performance.now() * 0.001;
            
            globe.rotation.y = t * 0.05;
            
            // Move flight dots along arcs
            routes.forEach(route => {
                route.t = (route.t + 0.001) % 1;
                const pos = route.curve.getPoint(route.t);
                route.dot.position.copy(pos);
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

## Animated Data Transitions

```javascript
class DataTransitionSystem {
    constructor(bars) {
        this.bars = bars;
        this.isAnimating = false;
    }
    
    transitionTo(newData, duration = 1000) {
        if (this.isAnimating) return;
        this.isAnimating = true;
        
        const startTime = performance.now();
        const startValues = this.bars.map(bar => bar.position.y * 2); // current heights
        const easeInOut = t => t < 0.5 ? 2 * t * t : 1 - 2 * (1 - t) * (1 - t);
        
        const animate = () => {
            const elapsed = performance.now() - startTime;
            const progress = Math.min(elapsed / duration, 1);
            const eased = easeInOut(progress);
            
            this.bars.forEach((bar, i) => {
                const startH = startValues[i];
                const targetH = newData[i];
                const currentH = startH + (targetH - startH) * eased;
                
                bar.scale.y = currentH / bar.geometry.parameters.height;
                bar.position.y = currentH / 2;
            });
            
            if (progress < 1) {
                requestAnimationFrame(animate);
            } else {
                this.isAnimating = false;
            }
        };
        
        requestAnimationFrame(animate);
    }
    
    // Staggered animation (bars animate one by one)
    staggerTransition(newData, staggerDelay = 50) {
        this.bars.forEach((bar, i) => {
            setTimeout(() => {
                this.animateSingleBar(bar, newData[i], 500);
            }, i * staggerDelay);
        });
    }
    
    animateSingleBar(bar, targetHeight, duration) {
        const startHeight = bar.scale.y * bar.geometry.parameters.height;
        const startTime = performance.now();
        
        const animate = () => {
            const elapsed = performance.now() - startTime;
            const progress = Math.min(elapsed / duration, 1);
            const eased = 1 - Math.pow(1 - progress, 3); // ease out cubic
            
            const currentH = startHeight + (targetHeight - startHeight) * eased;
            bar.scale.y = currentH / bar.geometry.parameters.height;
            bar.position.y = currentH / 2;
            
            if (progress < 1) requestAnimationFrame(animate);
        };
        
        requestAnimationFrame(animate);
    }
}
```

## 3D Heatmap บน Surface

```javascript
function createHeatmap(data, rows, cols, scene) {
    // data: Float32Array ขนาด rows * cols
    const width = 300, depth = 300;
    const cellW = width / cols;
    const cellD = depth / rows;
    
    const minVal = Math.min(...data);
    const maxVal = Math.max(...data);
    
    const colorScale = (v) => {
        const t = (v - minVal) / (maxVal - minVal);
        // Cold (blue) to hot (red)
        return new THREE.Color(t, 0, 1 - t);
    };
    
    for (let row = 0; row < rows; row++) {
        for (let col = 0; col < cols; col++) {
            const value = data[row * cols + col];
            const height = ((value - minVal) / (maxVal - minVal)) * 50;
            
            const geo = new THREE.BoxGeometry(cellW - 1, height + 1, cellD - 1);
            const mat = new THREE.MeshStandardMaterial({
                color: colorScale(value),
                roughness: 0.4,
                metalness: 0.2,
            });
            
            const cell = new THREE.Mesh(geo, mat);
            cell.position.set(
                col * cellW - width / 2 + cellW / 2,
                height / 2,
                row * cellD - depth / 2 + cellD / 2
            );
            cell.userData = { row, col, value };
            scene.add(cell);
        }
    }
}

// Usage
const heatData = new Float32Array(20 * 20);
for (let i = 0; i < heatData.length; i++) {
    const row = Math.floor(i / 20);
    const col = i % 20;
    heatData[i] = Math.sin(row * 0.5) * Math.cos(col * 0.5) + Math.random() * 0.2;
}
createHeatmap(heatData, 20, 20, scene);
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Sales Dashboard 3D
สร้าง dashboard ที่มีข้อมูลยอดขาย 3 ปีล่าสุด แสดงเป็น grouped 3D bar chart (แต่ละปีเป็นสีต่างกัน) มี animation เมื่อเปลี่ยนปีที่แสดง

### แบบฝึกหัดที่ 2: Social Network Graph
สร้าง network graph 3D ที่มี nodes 20 ตัว แต่ละ node มีขนาดต่างกันตาม connections สี node แสดง community และมี force-directed layout

### แบบฝึกหัดที่ 3: COVID Data Globe
สร้าง globe ที่แสดงข้อมูลสถิติประเทศต่างๆ เป็น spikes ความสูง spikes ต้องแปรตามข้อมูล มี tooltip เมื่อ hover

### แบบฝึกหัดที่ 4: Stock Market Visualization
สร้าง 3D visualization ของหุ้น 5 ตัว แสดง price history เป็น line chart ใน 3D space สามารถเลือก timeframe ได้ (1D, 1W, 1M)

## สรุป

การ visualize ข้อมูลใน 3D ต้องการความเข้าใจใน:
1. **D3.js scales** สำหรับ mapping data → visual dimensions
2. **BufferGeometry** สำหรับ custom shapes ที่ efficient
3. **Canvas Texture** สำหรับ labels และ legends
4. **Animation** สำหรับ data transitions
5. **Raycasting** สำหรับ interactive tooltips

Three.js + D3.js เป็น combination ที่ทรงพลังมากสำหรับ data visualization ใน 3D
