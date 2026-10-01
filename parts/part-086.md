# Part 86: 3D Data Visualization

## บทนำ: ทำไมต้องใช้ 3D สำหรับ Data?

2D charts เหมาะสำหรับข้อมูลส่วนใหญ่ แต่ 3D visualization มีข้อได้เปรียบเมื่อ:
- ข้อมูลมี 3 มิติที่สำคัญ (เช่น geographic + value + time)
- ต้องการแสดง network/relationship ที่ซับซ้อน
- ต้องการ immersive storytelling
- ข้อมูลเชิงพื้นที่ (map, terrain)

---

## 1. D3.js + Three.js Integration

### การใช้ D3 สำหรับ Data + Three.js สำหรับ Rendering

```javascript
// D3 ทำหน้าที่: scale, layout, data manipulation
// Three.js ทำหน้าที่: rendering 3D

import * as d3 from 'd3';
import * as THREE from 'three';

// D3 Scales สำหรับ mapping data ไป 3D space
function createScales(data) {
  const xExtent = d3.extent(data, d => d.x);
  const yExtent = d3.extent(data, d => d.y);
  const zExtent = d3.extent(data, d => d.z);
  const valueExtent = d3.extent(data, d => d.value);
  
  return {
    x: d3.scaleLinear().domain(xExtent).range([-10, 10]),
    y: d3.scaleLinear().domain(yExtent).range([0, 10]),
    z: d3.scaleLinear().domain(zExtent).range([-10, 10]),
    color: d3.scaleSequential(d3.interpolateRdYlGn).domain(valueExtent),
    size: d3.scaleSqrt().domain(valueExtent).range([0.1, 1])
  };
}

// แปลง D3 color ไป Three.js color
function d3ColorToThree(d3Color) {
  const { r, g, b } = d3.rgb(d3Color);
  return new THREE.Color(r / 255, g / 255, b / 255);
}
```

---

## 2. 3D Bar Chart

```javascript
class BarChart3D {
  constructor(scene, data, options = {}) {
    this.scene = scene;
    this.data = data;
    this.options = {
      barWidth: 0.8,
      barSpacing: 1.2,
      maxHeight: 10,
      colorScale: d3.interpolateViridis,
      labelColor: 0xffffff,
      ...options
    };
    
    this.group = new THREE.Group();
    scene.add(this.group);
    this.bars = [];
    this.labels = [];
  }
  
  render() {
    // Clear existing
    while (this.group.children.length) {
      this.group.remove(this.group.children[0]);
    }
    this.bars = [];
    
    const { barWidth, barSpacing, maxHeight } = this.options;
    const maxValue = d3.max(this.data, d => d.value);
    
    // สร้าง Grid floor
    this._createFloor();
    
    // สร้าง Axes
    this._createAxes(maxValue);
    
    // สร้าง bars
    this.data.forEach((d, i) => {
      const height = (d.value / maxValue) * maxHeight;
      const x = i * barSpacing - (this.data.length - 1) * barSpacing / 2;
      
      // Bar geometry
      const geo = new THREE.BoxGeometry(barWidth, height, barWidth);
      geo.translate(0, height / 2, 0);  // origin ที่ bottom
      
      // Color จาก value
      const colorHex = d3.color(this.options.colorScale(d.value / maxValue));
      const color = new THREE.Color(colorHex.r / 255, colorHex.g / 255, colorHex.b / 255);
      
      const mat = new THREE.MeshStandardMaterial({
        color,
        metalness: 0.3,
        roughness: 0.5,
        transparent: true,
        opacity: 0.9
      });
      
      const bar = new THREE.Mesh(geo, mat);
      bar.position.set(x, 0, 0);
      bar.castShadow = true;
      
      // เก็บข้อมูลสำหรับ interaction
      bar.userData = { data: d, index: i };
      
      this.group.add(bar);
      this.bars.push(bar);
      
      // Value label บน top ของ bar
      this._createLabel(d.value.toFixed(1), x, height + 0.3, 0);
      
      // Category label ที่ bottom
      this._createLabel(d.label || `${i}`, x, -0.5, 0);
    });
    
    return this;
  }
  
  _createFloor() {
    const size = this.data.length * this.options.barSpacing + 2;
    const geo = new THREE.PlaneGeometry(size, 8);
    const mat = new THREE.MeshStandardMaterial({
      color: 0x222244,
      roughness: 0.9
    });
    const floor = new THREE.Mesh(geo, mat);
    floor.rotation.x = -Math.PI / 2;
    floor.receiveShadow = true;
    this.group.add(floor);
  }
  
  _createAxes(maxValue) {
    const size = this.data.length * this.options.barSpacing;
    
    // X axis
    const xAxis = new THREE.ArrowHelper(
      new THREE.Vector3(1, 0, 0),
      new THREE.Vector3(-size / 2, 0, 0),
      size + 1,
      0x888888,
      0.5,
      0.3
    );
    this.group.add(xAxis);
    
    // Y axis
    const yAxis = new THREE.ArrowHelper(
      new THREE.Vector3(0, 1, 0),
      new THREE.Vector3(-size / 2, 0, 0),
      this.options.maxHeight + 1,
      0x888888,
      0.5,
      0.3
    );
    this.group.add(yAxis);
    
    // Y axis tick marks
    const ticks = 5;
    for (let i = 0; i <= ticks; i++) {
      const y = (i / ticks) * this.options.maxHeight;
      const value = (i / ticks) * maxValue;
      this._createLabel(value.toFixed(0), -size / 2 - 0.8, y, 0, 0.3);
    }
  }
  
  _createLabel(text, x, y, z, scale = 0.4) {
    // Simple canvas texture label
    const canvas = document.createElement('canvas');
    canvas.width = 256;
    canvas.height = 64;
    const ctx = canvas.getContext('2d');
    
    ctx.fillStyle = 'transparent';
    ctx.clearRect(0, 0, 256, 64);
    
    ctx.font = 'bold 36px Arial';
    ctx.fillStyle = '#ffffff';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText(text, 128, 32);
    
    const texture = new THREE.CanvasTexture(canvas);
    const mat = new THREE.SpriteMaterial({ map: texture, transparent: true });
    const sprite = new THREE.Sprite(mat);
    sprite.position.set(x, y, z);
    sprite.scale.set(scale * 4, scale, 1);
    
    this.group.add(sprite);
    this.labels.push(sprite);
  }
  
  // Animate bars ขึ้นมา
  animateIn(duration = 1.5) {
    this.bars.forEach((bar, i) => {
      const targetScaleY = bar.scale.y;
      bar.scale.y = 0;
      
      const startTime = Date.now() + i * 50;  // stagger
      
      const animate = () => {
        const elapsed = (Date.now() - startTime) / (duration * 1000);
        if (elapsed < 0) { requestAnimationFrame(animate); return; }
        
        const t = Math.min(elapsed, 1);
        const eased = 1 - Math.pow(1 - t, 3);  // ease out cubic
        bar.scale.y = eased * targetScaleY;
        
        if (t < 1) requestAnimationFrame(animate);
      };
      
      requestAnimationFrame(animate);
    });
  }
  
  // Highlight specific bar
  highlight(index) {
    this.bars.forEach((bar, i) => {
      bar.material.emissive.setHex(i === index ? 0x444444 : 0x000000);
    });
  }
  
  // Update data
  updateData(newData) {
    this.data = newData;
    this.render();
    this.animateIn();
  }
}
```

---

## 3. 3D Scatter Plot

```javascript
class ScatterPlot3D {
  constructor(scene, data, options = {}) {
    this.scene = scene;
    this.data = data;
    this.options = {
      pointSize: 0.2,
      colorBy: 'value',
      sizeBy: null,
      ...options
    };
    
    this.group = new THREE.Group();
    scene.add(this.group);
    
    this._createScales();
  }
  
  _createScales() {
    const xExt = d3.extent(this.data, d => d.x);
    const yExt = d3.extent(this.data, d => d.y);
    const zExt = d3.extent(this.data, d => d.z);
    const valExt = d3.extent(this.data, d => d.value || 0);
    
    this.scales = {
      x: d3.scaleLinear().domain(xExt).range([-8, 8]),
      y: d3.scaleLinear().domain(yExt).range([0, 8]),
      z: d3.scaleLinear().domain(zExt).range([-8, 8]),
      color: d3.scaleSequential(d3.interpolateTurbo).domain(valExt),
      size: d3.scaleSqrt().domain(valExt).range([0.05, 0.5])
    };
  }
  
  render() {
    // ใช้ InstancedMesh สำหรับ performance
    const count = this.data.length;
    const geo = new THREE.SphereGeometry(1, 8, 8);
    const mat = new THREE.MeshStandardMaterial({
      roughness: 0.3,
      metalness: 0.5,
      vertexColors: true
    });
    
    const instancedMesh = new THREE.InstancedMesh(geo, mat, count);
    instancedMesh.castShadow = true;
    
    const dummy = new THREE.Object3D();
    const color = new THREE.Color();
    
    this.data.forEach((d, i) => {
      const x = this.scales.x(d.x);
      const y = this.scales.y(d.y);
      const z = this.scales.z(d.z);
      const size = this.options.sizeBy ? 
        this.scales.size(d[this.options.sizeBy]) : 
        this.options.pointSize;
      
      dummy.position.set(x, y, z);
      dummy.scale.setScalar(size);
      dummy.updateMatrix();
      
      instancedMesh.setMatrixAt(i, dummy.matrix);
      
      // Color
      const d3Color = d3.color(this.scales.color(d.value || 0));
      color.setRGB(d3Color.r / 255, d3Color.g / 255, d3Color.b / 255);
      instancedMesh.setColorAt(i, color);
    });
    
    instancedMesh.instanceMatrix.needsUpdate = true;
    instancedMesh.instanceColor.needsUpdate = true;
    
    this.group.add(instancedMesh);
    this.mesh = instancedMesh;
    
    // เพิ่ม grid axes
    this._addGrid();
    
    return this;
  }
  
  _addGrid() {
    const gridHelper = new THREE.GridHelper(16, 16, 0x333355, 0x222244);
    this.group.add(gridHelper);
    
    const axesHelper = new THREE.AxesHelper(9);
    this.group.add(axesHelper);
  }
}
```

---

## 4. World Globe with Data

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title>World Data Globe</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { overflow: hidden; background: #000811; font-family: sans-serif; }
    canvas { display: block; }
    #tooltip {
      position: fixed;
      background: rgba(0,0,20,0.9);
      border: 1px solid #4488ff;
      color: white;
      padding: 8px 12px;
      border-radius: 5px;
      font-size: 13px;
      pointer-events: none;
      display: none;
      z-index: 10;
    }
    #legend {
      position: fixed;
      bottom: 20px;
      left: 20px;
      background: rgba(0,0,20,0.8);
      border: 1px solid rgba(68,136,255,0.3);
      padding: 15px;
      border-radius: 8px;
      color: white;
      font-size: 12px;
    }
    .legend-gradient {
      width: 150px;
      height: 12px;
      border-radius: 3px;
      margin: 5px 0;
    }
    .legend-labels {
      display: flex;
      justify-content: space-between;
    }
    #title {
      position: fixed;
      top: 20px;
      left: 50%;
      transform: translateX(-50%);
      color: #88aaff;
      font-size: 20px;
      font-weight: bold;
    }
  </style>
</head>
<body>
  <canvas id="canvas"></canvas>
  <div id="tooltip"></div>
  <div id="legend">
    <div>Population Density</div>
    <div class="legend-gradient" id="gradient"></div>
    <div class="legend-labels">
      <span>Low</span>
      <span>Medium</span>
      <span>High</span>
    </div>
  </div>
  <div id="title">World Population Globe</div>

  <script type="module">
    import * as THREE from 'https://cdn.skypack.dev/three@0.160.0';
    import { OrbitControls } from 'https://cdn.skypack.dev/three@0.160.0/examples/jsm/controls/OrbitControls.js';
    
    const canvas = document.getElementById('canvas');
    const tooltip = document.getElementById('tooltip');
    
    const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.outputColorSpace = THREE.SRGBColorSpace;
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.toneMappingExposure = 1.2;
    
    const scene = new THREE.Scene();
    
    const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
    camera.position.set(0, 0, 4);
    
    const controls = new OrbitControls(camera, canvas);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
    controls.rotateSpeed = 0.3;
    controls.minDistance = 2;
    controls.maxDistance = 8;
    controls.autoRotate = true;
    controls.autoRotateSpeed = 0.3;
    
    // === Globe ===
    const globeRadius = 1.5;
    
    // Earth sphere
    const earthGeo = new THREE.SphereGeometry(globeRadius, 64, 64);
    const textureLoader = new THREE.TextureLoader();
    
    // Globe material
    const earthMat = new THREE.MeshStandardMaterial({
      color: 0x1a3a6a,
      metalness: 0.0,
      roughness: 0.8
    });
    
    const earth = new THREE.Mesh(earthGeo, earthMat);
    scene.add(earth);
    
    // Atmosphere glow
    const atmGeo = new THREE.SphereGeometry(globeRadius * 1.02, 64, 64);
    const atmMat = new THREE.MeshBasicMaterial({
      color: 0x4466ff,
      transparent: true,
      opacity: 0.08,
      side: THREE.BackSide
    });
    scene.add(new THREE.Mesh(atmGeo, atmMat));
    
    // Star background
    const starGeo = new THREE.BufferGeometry();
    const starCount = 3000;
    const starPositions = new Float32Array(starCount * 3);
    for (let i = 0; i < starCount * 3; i++) {
      starPositions[i] = (Math.random() - 0.5) * 100;
    }
    starGeo.setAttribute('position', new THREE.BufferAttribute(starPositions, 3));
    const starMat = new THREE.PointsMaterial({ color: 0xffffff, size: 0.05 });
    scene.add(new THREE.Points(starGeo, starMat));
    
    // Convert lat/lon to 3D position
    function latLonToVector3(lat, lon, radius) {
      const phi = (90 - lat) * (Math.PI / 180);
      const theta = (lon + 180) * (Math.PI / 180);
      return new THREE.Vector3(
        -radius * Math.sin(phi) * Math.cos(theta),
        radius * Math.cos(phi),
        radius * Math.sin(phi) * Math.sin(theta)
      );
    }
    
    // === Sample data: major cities with population ===
    const cityData = [
      { name: 'Tokyo', lat: 35.68, lon: 139.69, population: 37.4, country: 'Japan' },
      { name: 'Delhi', lat: 28.68, lon: 77.21, population: 32.9, country: 'India' },
      { name: 'Shanghai', lat: 31.23, lon: 121.47, population: 27.1, country: 'China' },
      { name: 'São Paulo', lat: -23.55, lon: -46.63, population: 22.4, country: 'Brazil' },
      { name: 'Mexico City', lat: 19.43, lon: -99.13, population: 22.0, country: 'Mexico' },
      { name: 'Cairo', lat: 30.06, lon: 31.25, population: 21.3, country: 'Egypt' },
      { name: 'Mumbai', lat: 19.08, lon: 72.88, population: 20.7, country: 'India' },
      { name: 'Beijing', lat: 39.93, lon: 116.40, population: 20.4, country: 'China' },
      { name: 'Dhaka', lat: 23.72, lon: 90.41, population: 19.6, country: 'Bangladesh' },
      { name: 'Osaka', lat: 34.69, lon: 135.50, population: 19.3, country: 'Japan' },
      { name: 'New York', lat: 40.71, lon: -74.01, population: 18.8, country: 'USA' },
      { name: 'Karachi', lat: 24.86, lon: 67.01, population: 16.9, country: 'Pakistan' },
      { name: 'Buenos Aires', lat: -34.60, lon: -58.38, population: 15.3, country: 'Argentina' },
      { name: 'Istanbul', lat: 41.01, lon: 28.95, population: 15.2, country: 'Turkey' },
      { name: 'Kolkata', lat: 22.57, lon: 88.36, population: 14.9, country: 'India' },
      { name: 'Manila', lat: 14.60, lon: 120.98, population: 14.2, country: 'Philippines' },
      { name: 'Lagos', lat: 6.45, lon: 3.40, population: 14.0, country: 'Nigeria' },
      { name: 'Rio de Janeiro', lat: -22.91, lon: -43.17, population: 13.7, country: 'Brazil' },
      { name: 'Tianjin', lat: 39.13, lon: 117.20, population: 13.6, country: 'China' },
      { name: 'Kinshasa', lat: -4.32, lon: 15.32, population: 13.5, country: 'DRC' },
      { name: 'Guangzhou', lat: 23.13, lon: 113.27, population: 13.1, country: 'China' },
      { name: 'Los Angeles', lat: 34.05, lon: -118.24, population: 12.5, country: 'USA' },
      { name: 'Moscow', lat: 55.75, lon: 37.62, population: 12.5, country: 'Russia' },
      { name: 'Shenzhen', lat: 22.54, lon: 114.05, population: 12.4, country: 'China' },
      { name: 'Lahore', lat: 31.55, lon: 74.35, population: 12.2, country: 'Pakistan' },
      { name: 'Bangalore', lat: 12.97, lon: 77.59, population: 11.9, country: 'India' },
      { name: 'Jakarta', lat: -6.21, lon: 106.85, population: 11.1, country: 'Indonesia' },
      { name: 'London', lat: 51.51, lon: -0.13, population: 9.1, country: 'UK' },
      { name: 'Paris', lat: 48.86, lon: 2.35, population: 11.0, country: 'France' },
      { name: 'Bangkok', lat: 13.75, lon: 100.50, population: 10.7, country: 'Thailand' }
    ];
    
    const maxPop = Math.max(...cityData.map(d => d.population));
    
    // Color interpolation
    function popColor(population) {
      const t = population / maxPop;
      if (t < 0.33) {
        return new THREE.Color().setHSL(0.33, 1, 0.3 + t * 0.5);  // green
      } else if (t < 0.66) {
        return new THREE.Color().setHSL(0.15, 1, 0.4 + t * 0.3);  // yellow-orange
      } else {
        return new THREE.Color().setHSL(0, 1, 0.4 + t * 0.2);     // red
      }
    }
    
    // Create markers สำหรับแต่ละเมือง
    const markers = [];
    const markerGroup = new THREE.Group();
    earth.add(markerGroup);
    
    cityData.forEach(city => {
      const pos = latLonToVector3(city.lat, city.lon, globeRadius);
      
      // Base dot
      const dotSize = 0.02 + (city.population / maxPop) * 0.05;
      const dotGeo = new THREE.SphereGeometry(dotSize, 8, 8);
      const dotColor = popColor(city.population);
      const dotMat = new THREE.MeshBasicMaterial({ color: dotColor });
      const dot = new THREE.Mesh(dotGeo, dotMat);
      dot.position.copy(pos);
      markerGroup.add(dot);
      
      // Spike (bar ขึ้นจาก surface)
      const spikeHeight = (city.population / maxPop) * 0.5;
      const spikeGeo = new THREE.CylinderGeometry(dotSize * 0.5, dotSize * 0.3, spikeHeight, 4);
      const spikeMat = new THREE.MeshBasicMaterial({ 
        color: dotColor, 
        transparent: true, 
        opacity: 0.8 
      });
      const spike = new THREE.Mesh(spikeGeo, spikeMat);
      
      const spikeDir = pos.clone().normalize();
      spike.position.copy(pos).addScaledVector(spikeDir, spikeHeight / 2);
      spike.quaternion.setFromUnitVectors(new THREE.Vector3(0, 1, 0), spikeDir);
      markerGroup.add(spike);
      
      // Glow ring
      const ringGeo = new THREE.RingGeometry(dotSize * 1.5, dotSize * 2.5, 16);
      const ringMat = new THREE.MeshBasicMaterial({ 
        color: dotColor, 
        transparent: true, 
        opacity: 0.4,
        side: THREE.DoubleSide
      });
      const ring = new THREE.Mesh(ringGeo, ringMat);
      ring.position.copy(pos).multiplyScalar(1.001);
      ring.lookAt(pos.clone().multiplyScalar(2));
      markerGroup.add(ring);
      
      markers.push({ 
        dot, spike, ring, 
        data: city, 
        pos: pos.clone() 
      });
    });
    
    // Flight paths (arc ระหว่างเมือง)
    function createArc(from, to, height = 0.5) {
      const points = [];
      const segments = 50;
      
      for (let i = 0; i <= segments; i++) {
        const t = i / segments;
        
        // Interpolate ระหว่าง 2 points บน sphere
        const point = from.clone().lerp(to, t).normalize();
        
        // เพิ่ม height สำหรับ arc
        const arcHeight = Math.sin(t * Math.PI) * height;
        point.multiplyScalar(globeRadius + arcHeight);
        
        points.push(point);
      }
      
      return new THREE.CatmullRomCurve3(points);
    }
    
    // สร้าง flight paths
    const routePairs = [
      ['Tokyo', 'Los Angeles'],
      ['London', 'New York'],
      ['Moscow', 'Beijing'],
      ['Mumbai', 'London'],
      ['São Paulo', 'New York'],
      ['Dubai', 'Singapore'],  // (ไม่มีใน list แต่ใช้เป็น example)
      ['Tokyo', 'Bangkok'],
      ['Paris', 'Cairo']
    ];
    
    // เพิ่มเมืองที่ขาด
    const allCities = { ...Object.fromEntries(cityData.map(d => [d.name, d])) };
    
    const routeGroup = new THREE.Group();
    earth.add(routeGroup);
    
    routePairs.forEach(([from, to]) => {
      const cityFrom = cityData.find(d => d.name === from);
      const cityTo = cityData.find(d => d.name === to);
      
      if (!cityFrom || !cityTo) return;
      
      const fromPos = latLonToVector3(cityFrom.lat, cityFrom.lon, globeRadius);
      const toPos = latLonToVector3(cityTo.lat, cityTo.lon, globeRadius);
      
      const curve = createArc(fromPos, toPos, 0.4);
      const points = curve.getPoints(100);
      
      const geo = new THREE.BufferGeometry().setFromPoints(points);
      const mat = new THREE.LineBasicMaterial({ 
        color: 0x4488ff, 
        transparent: true, 
        opacity: 0.4 
      });
      
      const line = new THREE.Line(geo, mat);
      routeGroup.add(line);
      
      // Animated dot บน route
      const dotGeo = new THREE.SphereGeometry(0.015, 8, 8);
      const dotMat = new THREE.MeshBasicMaterial({ color: 0x88bbff });
      const dot = new THREE.Mesh(dotGeo, dotMat);
      
      dot.userData.curve = curve;
      dot.userData.progress = Math.random();
      dot.userData.speed = 0.1 + Math.random() * 0.1;
      
      earth.add(dot);
    });
    
    // Lighting
    scene.add(new THREE.AmbientLight(0x223355, 0.5));
    
    const sunLight = new THREE.DirectionalLight(0xfff5e0, 2);
    sunLight.position.set(5, 3, 5);
    scene.add(sunLight);
    
    const fillLight = new THREE.DirectionalLight(0x224488, 0.3);
    fillLight.position.set(-5, -2, -3);
    scene.add(fillLight);
    
    // Raycaster สำหรับ hover
    const raycaster = new THREE.Raycaster();
    const mouse = new THREE.Vector2();
    
    canvas.addEventListener('mousemove', e => {
      mouse.x = (e.clientX / window.innerWidth) * 2 - 1;
      mouse.y = -(e.clientY / window.innerHeight) * 2 + 1;
      
      raycaster.setFromCamera(mouse, camera);
      
      const dotMeshes = markers.map(m => m.dot);
      const intersects = raycaster.intersectObjects(dotMeshes);
      
      if (intersects.length > 0) {
        const marker = markers.find(m => m.dot === intersects[0].object);
        if (marker) {
          tooltip.style.display = 'block';
          tooltip.style.left = `${e.clientX + 15}px`;
          tooltip.style.top = `${e.clientY - 30}px`;
          tooltip.innerHTML = `
            <strong>${marker.data.name}</strong><br>
            ${marker.data.country}<br>
            Population: ${marker.data.population.toFixed(1)}M
          `;
        }
      } else {
        tooltip.style.display = 'none';
      }
    });
    
    // Legend gradient
    const gradCanvas = document.createElement('canvas');
    gradCanvas.width = 150;
    gradCanvas.height = 12;
    const gradCtx = gradCanvas.getContext('2d');
    const grad = gradCtx.createLinearGradient(0, 0, 150, 0);
    grad.addColorStop(0, '#1a5c1a');
    grad.addColorStop(0.5, '#ccaa00');
    grad.addColorStop(1, '#cc2200');
    gradCtx.fillStyle = grad;
    gradCtx.fillRect(0, 0, 150, 12);
    document.getElementById('gradient').style.background = 
      `linear-gradient(to right, #1a5c1a, #ccaa00, #cc2200)`;
    
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
      
      const time = clock.getElapsedTime();
      const delta = clock.getDelta();
      
      // Update animated dots on routes
      earth.children.forEach(child => {
        if (child.userData.curve) {
          child.userData.progress = (child.userData.progress + child.userData.speed * delta) % 1;
          const pos = child.userData.curve.getPoint(child.userData.progress);
          child.position.copy(pos);
        }
      });
      
      // Pulse markers
      markers.forEach(({ ring, data }, i) => {
        const t = (time * 2 + i * 0.3) % 1;
        ring.material.opacity = 0.2 * (1 - t);
        const scale = 1 + t * 2;
        ring.scale.setScalar(scale);
      });
      
      controls.update();
      renderer.render(scene, camera);
    }
    
    animate();
  </script>
</body>
</html>
```

---

## 5. 3D Network Graph (Force-Directed)

```javascript
class NetworkGraph3D {
  constructor(scene, nodes, edges, options = {}) {
    this.scene = scene;
    this.nodes = nodes;
    this.edges = edges;
    this.options = {
      nodeSize: 0.2,
      linkLength: 3,
      linkStrength: 0.1,
      repulsion: 2,
      iterations: 100,
      colorBy: 'group',
      ...options
    };
    
    this.group = new THREE.Group();
    scene.add(this.group);
    
    this.nodeMeshes = [];
    this.edgeMeshes = [];
    this.positions = [];
    this.velocities = [];
    
    this._initPositions();
    this._simulate();
    this._render();
  }
  
  _initPositions() {
    // สุ่ม positions เริ่มต้น
    this.nodes.forEach((node, i) => {
      this.positions.push(new THREE.Vector3(
        (Math.random() - 0.5) * 10,
        (Math.random() - 0.5) * 10,
        (Math.random() - 0.5) * 10
      ));
      this.velocities.push(new THREE.Vector3(0, 0, 0));
    });
  }
  
  _simulate() {
    const { linkLength, linkStrength, repulsion, iterations } = this.options;
    const dt = 0.1;
    
    for (let iter = 0; iter < iterations; iter++) {
      // Reset forces
      const forces = this.nodes.map(() => new THREE.Vector3(0, 0, 0));
      
      // Repulsion between all node pairs
      for (let i = 0; i < this.nodes.length; i++) {
        for (let j = i + 1; j < this.nodes.length; j++) {
          const diff = this.positions[i].clone().sub(this.positions[j]);
          const dist = Math.max(diff.length(), 0.1);
          const force = repulsion / (dist * dist);
          const dir = diff.normalize().multiplyScalar(force);
          
          forces[i].add(dir);
          forces[j].sub(dir);
        }
      }
      
      // Attraction along edges
      this.edges.forEach(({ source, target }) => {
        const diff = this.positions[target].clone().sub(this.positions[source]);
        const dist = diff.length();
        const force = (dist - linkLength) * linkStrength;
        const dir = diff.normalize().multiplyScalar(force);
        
        forces[source].add(dir);
        forces[target].sub(dir);
      });
      
      // Update positions
      const damping = 0.9;
      for (let i = 0; i < this.nodes.length; i++) {
        this.velocities[i].add(forces[i].multiplyScalar(dt));
        this.velocities[i].multiplyScalar(damping);
        this.positions[i].add(this.velocities[i].clone().multiplyScalar(dt));
      }
    }
  }
  
  _render() {
    // Color palette สำหรับ groups
    const groupColors = [
      0xff4444, 0x44ff44, 0x4444ff, 0xffff44, 0xff44ff,
      0x44ffff, 0xff8844, 0x8844ff, 0x44ff88
    ];
    
    const groups = [...new Set(this.nodes.map(n => n.group || 0))];
    const colorMap = new Map(groups.map((g, i) => [g, groupColors[i % groupColors.length]]));
    
    // Nodes
    this.nodes.forEach((node, i) => {
      const size = this.options.nodeSize * (1 + (node.size || 0) * 0.5);
      const geo = new THREE.SphereGeometry(size, 16, 16);
      const color = colorMap.get(node.group || 0) || 0xffffff;
      const mat = new THREE.MeshStandardMaterial({
        color,
        metalness: 0.5,
        roughness: 0.3,
        emissive: color,
        emissiveIntensity: 0.2
      });
      
      const mesh = new THREE.Mesh(geo, mat);
      mesh.position.copy(this.positions[i]);
      mesh.userData = { node, index: i };
      
      // Point light บน node
      if (node.highlight) {
        const light = new THREE.PointLight(color, 0.3, 3);
        mesh.add(light);
      }
      
      this.group.add(mesh);
      this.nodeMeshes.push(mesh);
    });
    
    // Edges
    this.edges.forEach(({ source, target, weight = 1 }) => {
      const start = this.positions[source];
      const end = this.positions[target];
      
      const mid = start.clone().lerp(end, 0.5);
      const length = start.distanceTo(end);
      const direction = end.clone().sub(start).normalize();
      
      const geo = new THREE.CylinderGeometry(
        0.02 * weight, 
        0.02 * weight, 
        length, 
        4
      );
      geo.translate(0, length / 2, 0);
      
      const mat = new THREE.MeshBasicMaterial({
        color: 0x336688,
        transparent: true,
        opacity: 0.4
      });
      
      const edge = new THREE.Mesh(geo, mat);
      edge.position.copy(start);
      edge.quaternion.setFromUnitVectors(new THREE.Vector3(0, 1, 0), direction);
      
      this.group.add(edge);
      this.edgeMeshes.push(edge);
    });
    
    // Labels
    this.nodes.forEach((node, i) => {
      if (node.label) {
        // ใช้ canvas texture สำหรับ label
        const canvas = document.createElement('canvas');
        canvas.width = 256;
        canvas.height = 64;
        const ctx = canvas.getContext('2d');
        ctx.font = 'bold 28px Arial';
        ctx.fillStyle = 'white';
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';
        ctx.fillText(node.label, 128, 32);
        
        const texture = new THREE.CanvasTexture(canvas);
        const sprite = new THREE.Sprite(new THREE.SpriteMaterial({ map: texture, transparent: true }));
        sprite.position.copy(this.positions[i]).add(new THREE.Vector3(0, this.options.nodeSize + 0.2, 0));
        sprite.scale.set(1.5, 0.4, 1);
        
        this.group.add(sprite);
      }
    });
  }
  
  // อัพเดท positions แบบ real-time (สำหรับ live graph)
  tick(dt) {
    const { linkLength, linkStrength, repulsion } = this.options;
    const forces = this.nodes.map(() => new THREE.Vector3(0, 0, 0));
    
    // Repulsion
    for (let i = 0; i < this.nodes.length; i++) {
      for (let j = i + 1; j < this.nodes.length; j++) {
        const diff = this.positions[i].clone().sub(this.positions[j]);
        const dist = Math.max(diff.length(), 0.1);
        const force = repulsion / (dist * dist);
        const dir = diff.normalize().multiplyScalar(force);
        forces[i].add(dir);
        forces[j].sub(dir);
      }
    }
    
    // Attraction
    this.edges.forEach(({ source, target }) => {
      const diff = this.positions[target].clone().sub(this.positions[source]);
      const dist = diff.length();
      const force = (dist - linkLength) * linkStrength;
      const dir = diff.normalize().multiplyScalar(force);
      forces[source].add(dir);
      forces[target].sub(dir);
    });
    
    // Update
    for (let i = 0; i < this.nodes.length; i++) {
      this.velocities[i].add(forces[i].multiplyScalar(dt));
      this.velocities[i].multiplyScalar(0.85);
      this.positions[i].add(this.velocities[i].clone().multiplyScalar(dt));
      
      if (this.nodeMeshes[i]) {
        this.nodeMeshes[i].position.copy(this.positions[i]);
      }
    }
    
    // Update edge positions
    this.edgeMeshes.forEach((mesh, idx) => {
      const edge = this.edges[idx];
      const start = this.positions[edge.source];
      const end = this.positions[edge.target];
      const direction = end.clone().sub(start).normalize();
      const length = start.distanceTo(end);
      
      mesh.position.copy(start);
      mesh.quaternion.setFromUnitVectors(new THREE.Vector3(0, 1, 0), direction);
      mesh.scale.y = length;
    });
  }
}
```

---

## 6. สรุป 3D Data Visualization

### เมื่อไหร่ควรใช้ 3D vs 2D?

| ประเภทข้อมูล | แนะนำ |
|------------|-------|
| Time series | 2D line chart |
| Comparison | 2D bar chart |
| Distribution | 2D histogram |
| Geographic | 3D Globe หรือ 2D map |
| Network | 3D force-directed |
| Multi-dimensional | 3D scatter plot |
| Spatial simulation | 3D เสมอ |

### Performance Tips:
- ใช้ InstancedMesh สำหรับ data points จำนวนมาก
- LOD สำหรับ markers บน globe
- Web Workers สำหรับ force simulation
- Virtual scrolling สำหรับ data updates

ในบทถัดไป (Part 87) เราจะเรียนรู้เกี่ยวกับ Architectural Visualization
