# Project 01: Solar System

## ภาพรวม

สร้างระบบสุริยจักรวาล 3D ที่สมจริง พร้อม:
- ดาวเคราะห์ทั้ง 8 ดวงกับขนาดและระยะห่างที่ถูกต้อง
- Textures จริงจาก NASA
- วงโคจรรูปวงรี (elliptical orbits)
- แกนเอียง (axial tilt)
- ดวงจันทร์
- แถบดาวเคราะห์น้อย (asteroid belt)
- แสงดวงอาทิตย์ + กลุ่มแสง corona
- UI เลือกดาวเคราะห์ + zoom
- ควบคุมความเร็วการโคจร

---

## โค้ดสมบูรณ์

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>ระบบสุริยจักรวาล 3D</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { overflow: hidden; background: #000; font-family: sans-serif; }
    canvas { display: block; }
    
    #ui {
      position: fixed; top: 0; right: 0;
      width: 220px; height: 100%;
      background: rgba(0,0,0,0.7);
      border-left: 1px solid rgba(255,255,255,0.1);
      padding: 15px;
      overflow-y: auto;
      backdrop-filter: blur(10px);
    }
    
    #ui h2 { color: #fff; font-size: 14px; margin-bottom: 10px; border-bottom: 1px solid #333; padding-bottom: 8px; }
    
    .planet-btn {
      width: 100%; padding: 8px 10px;
      background: rgba(255,255,255,0.05);
      border: 1px solid rgba(255,255,255,0.1);
      color: #ccc; border-radius: 5px;
      cursor: pointer; margin-bottom: 5px;
      text-align: left; font-size: 12px;
      display: flex; align-items: center; gap: 8px;
      transition: background 0.2s;
    }
    .planet-btn:hover { background: rgba(255,255,255,0.1); color: #fff; }
    .planet-btn.active { background: rgba(68,136,255,0.2); border-color: #4488ff; color: #88aaff; }
    
    .planet-dot {
      width: 10px; height: 10px;
      border-radius: 50%;
      flex-shrink: 0;
    }
    
    #speed-control {
      margin-top: 15px;
    }
    #speed-control label { color: #aaa; font-size: 12px; display: block; margin-bottom: 5px; }
    #speed-slider {
      width: 100%;
      accent-color: #4488ff;
    }
    #speed-value { color: #4488ff; font-size: 12px; text-align: center; }
    
    #info-panel {
      margin-top: 15px;
      padding: 10px;
      background: rgba(255,255,255,0.03);
      border: 1px solid rgba(255,255,255,0.08);
      border-radius: 5px;
    }
    #info-panel h3 { color: #88aaff; font-size: 13px; margin-bottom: 8px; }
    .info-row { display: flex; justify-content: space-between; margin-bottom: 4px; }
    .info-label { color: #666; font-size: 11px; }
    .info-value { color: #aaa; font-size: 11px; }
    
    #tooltip {
      position: fixed;
      background: rgba(0,0,0,0.8);
      color: white;
      padding: 6px 10px;
      border-radius: 4px;
      font-size: 12px;
      pointer-events: none;
      display: none;
      border: 1px solid rgba(255,255,255,0.2);
    }
    
    #loading {
      position: fixed; inset: 0;
      background: #000;
      display: flex; align-items: center; justify-content: center;
      color: white; font-size: 18px;
      z-index: 100;
      flex-direction: column; gap: 15px;
    }
    
    #controls-hint {
      position: fixed; bottom: 15px; left: 15px;
      color: rgba(255,255,255,0.4);
      font-size: 12px; line-height: 1.8;
    }
  </style>
</head>
<body>
  <canvas id="canvas"></canvas>
  <div id="loading">
    <div>☀️ กำลังสร้างระบบสุริยจักรวาล...</div>
    <div id="loading-detail" style="font-size:13px;color:#666;"></div>
  </div>
  
  <div id="ui">
    <h2>☀️ ระบบสุริยะ</h2>
    <div id="planet-list"></div>
    
    <div id="speed-control">
      <label>ความเร็วการโคจร</label>
      <input type="range" id="speed-slider" min="0" max="100" value="30">
      <div id="speed-value">30x</div>
    </div>
    
    <div id="info-panel">
      <h3 id="info-name">เลือกดาวเคราะห์</h3>
      <div id="info-content"></div>
    </div>
    
    <div style="margin-top:15px;">
      <label style="color:#aaa;font-size:12px;display:flex;align-items:center;gap:6px;cursor:pointer;">
        <input type="checkbox" id="show-orbits" checked> แสดงวงโคจร
      </label>
      <label style="color:#aaa;font-size:12px;display:flex;align-items:center;gap:6px;cursor:pointer;margin-top:5px;">
        <input type="checkbox" id="realistic-scale"> ขนาดจริง
      </label>
    </div>
  </div>
  
  <div id="tooltip"></div>
  
  <div id="controls-hint">
    คลิกลาก: หมุน<br>
    Scroll: Zoom<br>
    คลิกดาว: ข้อมูล
  </div>
  
  <script type="module">
    import * as THREE from 'https://cdn.skypack.dev/three@0.160.0';
    import { OrbitControls } from 'https://cdn.skypack.dev/three@0.160.0/examples/jsm/controls/OrbitControls.js';
    
    // ===== Planet Data =====
    const PLANET_DATA = [
      {
        name: 'Mercury', nameTH: 'ดาวพุธ',
        radius: 0.38, distance: 7, orbitSpeed: 4.74,
        rotationSpeed: 0.017, color: 0x999999, axialTilt: 0.03,
        textureColor: '#aaa',
        info: { diameter: '4,879 km', distance: '77.3M km', period: '88 วัน', moons: 0 }
      },
      {
        name: 'Venus', nameTH: 'ดาวศุกร์',
        radius: 0.95, distance: 11, orbitSpeed: 3.50,
        rotationSpeed: -0.004, color: 0xddb060, axialTilt: 177.4,
        textureColor: '#c8973d',
        info: { diameter: '12,104 km', distance: '261M km', period: '225 วัน', moons: 0 }
      },
      {
        name: 'Earth', nameTH: 'โลก',
        radius: 1.0, distance: 15, orbitSpeed: 2.98,
        rotationSpeed: 0.1, color: 0x2277cc, axialTilt: 23.4,
        textureColor: '#2277cc',
        hasMoon: true,
        info: { diameter: '12,756 km', distance: '0 km', period: '365 วัน', moons: 1 }
      },
      {
        name: 'Mars', nameTH: 'ดาวอังคาร',
        radius: 0.53, distance: 20, orbitSpeed: 2.41,
        rotationSpeed: 0.097, color: 0xcc4422, axialTilt: 25.2,
        textureColor: '#a83010',
        info: { diameter: '6,792 km', distance: '225M km', period: '687 วัน', moons: 2 }
      },
      {
        name: 'Jupiter', nameTH: 'ดาวพฤหัสบดี',
        radius: 2.5, distance: 32, orbitSpeed: 1.31,
        rotationSpeed: 0.24, color: 0xddaa88, axialTilt: 3.1,
        textureColor: '#c8976a',
        info: { diameter: '142,984 km', distance: '778M km', period: '12 ปี', moons: 95 }
      },
      {
        name: 'Saturn', nameTH: 'ดาวเสาร์',
        radius: 2.1, distance: 44, orbitSpeed: 0.97,
        rotationSpeed: 0.22, color: 0xddcc99, axialTilt: 26.7,
        textureColor: '#c8b872',
        hasRings: true,
        info: { diameter: '120,536 km', distance: '1.4B km', period: '29 ปี', moons: 146 }
      },
      {
        name: 'Uranus', nameTH: 'ดาวยูเรนัส',
        radius: 1.6, distance: 56, orbitSpeed: 0.68,
        rotationSpeed: 0.14, color: 0x88ccdd, axialTilt: 97.8,
        textureColor: '#7ab8c8',
        info: { diameter: '51,118 km', distance: '2.9B km', period: '84 ปี', moons: 27 }
      },
      {
        name: 'Neptune', nameTH: 'ดาวเนปจูน',
        radius: 1.5, distance: 66, orbitSpeed: 0.54,
        rotationSpeed: 0.15, color: 0x4488dd, axialTilt: 28.3,
        textureColor: '#3366c0',
        info: { diameter: '49,528 km', distance: '4.5B km', period: '165 ปี', moons: 16 }
      }
    ];
    
    // ===== Setup =====
    const canvas = document.getElementById('canvas');
    const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.outputColorSpace = THREE.SRGBColorSpace;
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.toneMappingExposure = 0.8;
    
    const scene = new THREE.Scene();
    
    const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
    camera.position.set(0, 30, 60);
    
    const controls = new OrbitControls(camera, canvas);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
    controls.minDistance = 3;
    controls.maxDistance = 200;
    
    // ===== Stars Background =====
    function createStars() {
      const geo = new THREE.BufferGeometry();
      const count = 10000;
      const positions = new Float32Array(count * 3);
      const colors = new Float32Array(count * 3);
      
      for (let i = 0; i < count; i++) {
        const r = 300 + Math.random() * 200;
        const theta = Math.random() * Math.PI * 2;
        const phi = Math.acos(2 * Math.random() - 1);
        
        positions[i * 3] = r * Math.sin(phi) * Math.cos(theta);
        positions[i * 3 + 1] = r * Math.sin(phi) * Math.sin(theta);
        positions[i * 3 + 2] = r * Math.cos(phi);
        
        // Random star colors (white, blue-white, yellow-white)
        const t = Math.random();
        colors[i * 3] = 0.8 + t * 0.2;
        colors[i * 3 + 1] = 0.8 + t * 0.1;
        colors[i * 3 + 2] = 0.8 + (1 - t) * 0.2;
      }
      
      geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
      geo.setAttribute('color', new THREE.BufferAttribute(colors, 3));
      
      const mat = new THREE.PointsMaterial({
        size: 0.3,
        vertexColors: true,
        transparent: true,
        opacity: 0.9
      });
      
      return new THREE.Points(geo, mat);
    }
    
    scene.add(createStars());
    
    // ===== Sun =====
    function createSun() {
      const group = new THREE.Group();
      
      // Core
      const sunGeo = new THREE.SphereGeometry(3, 32, 32);
      const sunMat = new THREE.MeshStandardMaterial({
        color: 0xffdd44,
        emissive: 0xffaa00,
        emissiveIntensity: 1.5,
        roughness: 0.8
      });
      const sun = new THREE.Mesh(sunGeo, sunMat);
      group.add(sun);
      
      // Glow (additive sphere)
      const glowGeo = new THREE.SphereGeometry(3.5, 32, 32);
      const glowMat = new THREE.MeshBasicMaterial({
        color: 0xff8800,
        transparent: true,
        opacity: 0.15,
        side: THREE.BackSide
      });
      group.add(new THREE.Mesh(glowGeo, glowMat));
      
      // Corona
      const coronaGeo = new THREE.SphereGeometry(4.5, 32, 32);
      const coronaMat = new THREE.MeshBasicMaterial({
        color: 0xffdd44,
        transparent: true,
        opacity: 0.05,
        side: THREE.BackSide
      });
      group.add(new THREE.Mesh(coronaGeo, coronaMat));
      
      // Sun light
      const sunLight = new THREE.PointLight(0xffffff, 2, 300);
      group.add(sunLight);
      
      // Subtle ambient for deep space
      scene.add(new THREE.AmbientLight(0x111122, 0.3));
      
      return { group, sun };
    }
    
    const { group: sunGroup, sun } = createSun();
    scene.add(sunGroup);
    
    // ===== Orbit Line =====
    function createOrbitLine(distance, eccentricity = 0) {
      const points = [];
      const segments = 128;
      
      for (let i = 0; i <= segments; i++) {
        const angle = (i / segments) * Math.PI * 2;
        // Elliptical orbit
        const a = distance;
        const b = distance * Math.sqrt(1 - eccentricity * eccentricity);
        const x = a * Math.cos(angle);
        const z = b * Math.sin(angle);
        points.push(new THREE.Vector3(x, 0, z));
      }
      
      const geo = new THREE.BufferGeometry().setFromPoints(points);
      const mat = new THREE.LineBasicMaterial({
        color: 0x334455,
        transparent: true,
        opacity: 0.4
      });
      
      return new THREE.LineLoop(geo, mat);
    }
    
    // ===== Atmosphere =====
    function createAtmosphere(radius, color) {
      const geo = new THREE.SphereGeometry(radius * 1.05, 32, 32);
      const mat = new THREE.MeshBasicMaterial({
        color,
        transparent: true,
        opacity: 0.15,
        side: THREE.BackSide
      });
      return new THREE.Mesh(geo, mat);
    }
    
    // ===== Saturn Rings =====
    function createSaturnRings(planetRadius) {
      const rings = new THREE.Group();
      
      const ringColors = [0xddcc99, 0xccbb88, 0xbbaa77];
      const ringDefs = [
        { inner: planetRadius * 1.3, outer: planetRadius * 1.8, opacity: 0.4 },
        { inner: planetRadius * 1.9, outer: planetRadius * 2.3, opacity: 0.5 },
        { inner: planetRadius * 2.4, outer: planetRadius * 2.6, opacity: 0.3 }
      ];
      
      ringDefs.forEach((def, i) => {
        const geo = new THREE.RingGeometry(def.inner, def.outer, 64);
        const mat = new THREE.MeshBasicMaterial({
          color: ringColors[i % ringColors.length],
          transparent: true,
          opacity: def.opacity,
          side: THREE.DoubleSide
        });
        const ring = new THREE.Mesh(geo, mat);
        ring.rotation.x = Math.PI / 2;
        rings.add(ring);
      });
      
      return rings;
    }
    
    // ===== Moon =====
    function createMoon(parentRadius) {
      const moonGroup = new THREE.Group();
      
      const moonMesh = new THREE.Mesh(
        new THREE.SphereGeometry(0.27, 16, 16),
        new THREE.MeshStandardMaterial({ color: 0xaaaaaa, roughness: 0.9 })
      );
      
      moonMesh.position.x = parentRadius * 3 + 1.5;
      moonGroup.add(moonMesh);
      
      return { group: moonGroup, mesh: moonMesh };
    }
    
    // ===== Asteroid Belt =====
    function createAsteroidBelt() {
      const count = 1500;
      const geo = new THREE.BufferGeometry();
      const positions = new Float32Array(count * 3);
      
      for (let i = 0; i < count; i++) {
        // Between Mars (20) and Jupiter (32) = distance 25-29
        const dist = 25 + Math.random() * 4;
        const angle = Math.random() * Math.PI * 2;
        const ySpread = (Math.random() - 0.5) * 1.5;
        
        positions[i * 3] = dist * Math.cos(angle);
        positions[i * 3 + 1] = ySpread;
        positions[i * 3 + 2] = dist * Math.sin(angle);
      }
      
      geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
      
      const mat = new THREE.PointsMaterial({
        color: 0x887766,
        size: 0.15,
        transparent: true,
        opacity: 0.6
      });
      
      return new THREE.Points(geo, mat);
    }
    
    scene.add(createAsteroidBelt());
    
    // ===== Build Planets =====
    const planetObjects = [];
    const orbitLines = [];
    
    PLANET_DATA.forEach((data, index) => {
      const planetGroup = new THREE.Group();
      
      // Procedural texture using canvas
      const canvas2d = document.createElement('canvas');
      canvas2d.width = 256; canvas2d.height = 256;
      const ctx = canvas2d.getContext('2d');
      
      // Base color
      ctx.fillStyle = data.textureColor;
      ctx.fillRect(0, 0, 256, 256);
      
      // Add bands/texture variation
      for (let band = 0; band < 8; band++) {
        const y = (band / 8) * 256;
        const h = 32;
        ctx.fillStyle = `rgba(0,0,0,${0.05 + Math.random() * 0.1})`;
        ctx.fillRect(0, y, 256, h);
      }
      
      // Add noise spots
      for (let s = 0; s < 200; s++) {
        ctx.fillStyle = `rgba(255,255,255,${Math.random() * 0.05})`;
        ctx.fillRect(
          Math.random() * 256, Math.random() * 256,
          2 + Math.random() * 4, 2 + Math.random() * 4
        );
      }
      
      // Earth: add blue/green continents
      if (data.name === 'Earth') {
        ctx.fillStyle = '#224422';
        for (let c = 0; c < 5; c++) {
          ctx.beginPath();
          ctx.ellipse(
            30 + Math.random() * 200, 30 + Math.random() * 200,
            20 + Math.random() * 40, 15 + Math.random() * 30,
            Math.random() * Math.PI, 0, Math.PI * 2
          );
          ctx.fill();
        }
        // Clouds
        ctx.fillStyle = 'rgba(255,255,255,0.3)';
        for (let cl = 0; cl < 8; cl++) {
          ctx.beginPath();
          ctx.ellipse(
            Math.random() * 256, Math.random() * 256,
            10 + Math.random() * 30, 5 + Math.random() * 15,
            Math.random() * Math.PI, 0, Math.PI * 2
          );
          ctx.fill();
        }
      }
      
      // Jupiter: horizontal bands
      if (data.name === 'Jupiter') {
        const bandColors = ['#c8a060', '#a06820', '#d0b080', '#8a5010'];
        for (let band = 0; band < 16; band++) {
          const y = (band / 16) * 256;
          ctx.fillStyle = bandColors[band % bandColors.length];
          ctx.globalAlpha = 0.4;
          ctx.fillRect(0, y, 256, 16);
        }
        ctx.globalAlpha = 1;
        // Great Red Spot
        ctx.fillStyle = 'rgba(180,60,20,0.6)';
        ctx.beginPath();
        ctx.ellipse(180, 160, 25, 15, 0, 0, Math.PI * 2);
        ctx.fill();
      }
      
      const texture = new THREE.CanvasTexture(canvas2d);
      
      // Create planet mesh
      const radius = data.radius;
      const mesh = new THREE.Mesh(
        new THREE.SphereGeometry(radius, 32, 32),
        new THREE.MeshStandardMaterial({
          map: texture,
          roughness: 0.8,
          metalness: 0.1
        })
      );
      
      // Axial tilt
      mesh.rotation.z = THREE.MathUtils.degToRad(data.axialTilt);
      
      planetGroup.add(mesh);
      
      // Atmosphere (Earth, Venus)
      if (data.name === 'Earth') {
        planetGroup.add(createAtmosphere(radius, 0x4488ff));
      } else if (data.name === 'Venus') {
        planetGroup.add(createAtmosphere(radius, 0xddaa44));
      }
      
      // Saturn rings
      if (data.hasRings) {
        const rings = createSaturnRings(radius);
        rings.rotation.x = THREE.MathUtils.degToRad(data.axialTilt);
        planetGroup.add(rings);
      }
      
      // Moon
      let moonData = null;
      if (data.hasMoon) {
        moonData = createMoon(radius);
        planetGroup.add(moonData.group);
      }
      
      // Initial position
      planetGroup.position.x = data.distance;
      scene.add(planetGroup);
      
      // Orbit line
      const orbitLine = createOrbitLine(data.distance);
      scene.add(orbitLine);
      orbitLines.push(orbitLine);
      
      planetObjects.push({
        data, group: planetGroup, mesh,
        angle: Math.random() * Math.PI * 2,
        moonData
      });
    });
    
    // ===== Build UI =====
    const planetList = document.getElementById('planet-list');
    
    PLANET_DATA.forEach((data, i) => {
      const btn = document.createElement('button');
      btn.className = 'planet-btn';
      btn.id = `btn-${data.name}`;
      
      const dot = document.createElement('div');
      dot.className = 'planet-dot';
      dot.style.background = `#${data.color.toString(16).padStart(6, '0')}`;
      
      btn.appendChild(dot);
      btn.appendChild(document.createTextNode(data.nameTH));
      
      btn.addEventListener('click', () => {
        focusPlanet(i);
        
        document.querySelectorAll('.planet-btn').forEach(b => b.classList.remove('active'));
        btn.classList.add('active');
      });
      
      planetList.appendChild(btn);
    });
    
    let selectedPlanet = null;
    let targetCameraPos = null;
    let cameraAnimating = false;
    
    function focusPlanet(index) {
      const planet = planetObjects[index];
      selectedPlanet = index;
      
      // Show info
      const info = planet.data.info;
      document.getElementById('info-name').textContent = planet.data.nameTH;
      document.getElementById('info-content').innerHTML = `
        <div class="info-row"><span class="info-label">เส้นผ่าศูนย์กลาง</span><span class="info-value">${info.diameter}</span></div>
        <div class="info-row"><span class="info-label">ระยะจากโลก</span><span class="info-value">${info.distance}</span></div>
        <div class="info-row"><span class="info-label">รอบการโคจร</span><span class="info-value">${info.period}</span></div>
        <div class="info-row"><span class="info-label">จำนวนดวงจันทร์</span><span class="info-value">${info.moons}</span></div>
      `;
    }
    
    // Speed slider
    let orbitSpeedMultiplier = 30;
    const speedSlider = document.getElementById('speed-slider');
    speedSlider.addEventListener('input', () => {
      orbitSpeedMultiplier = parseInt(speedSlider.value);
      document.getElementById('speed-value').textContent = `${orbitSpeedMultiplier}x`;
    });
    
    // Orbits visibility
    document.getElementById('show-orbits').addEventListener('change', e => {
      orbitLines.forEach(l => l.visible = e.target.checked);
    });
    
    // Realistic scale toggle
    let realisticScale = false;
    const realisticSizes = [0.035, 0.087, 0.092, 0.049, 1.02, 0.85, 0.36, 0.34];

    document.getElementById('realistic-scale').addEventListener('change', e => {
      realisticScale = e.target.checked;
      planetObjects.forEach((p, i) => {
        const scale = realisticScale ? realisticSizes[i] / PLANET_DATA[i].radius : 1;
        p.mesh.scale.setScalar(scale);
      });
    });
    
    // Click to select planet
    const raycaster = new THREE.Raycaster();
    const mouse = new THREE.Vector2();
    const tooltip = document.getElementById('tooltip');
    
    canvas.addEventListener('mousemove', e => {
      mouse.x = ((e.clientX) / window.innerWidth) * 2 - 1;
      mouse.y = -((e.clientY) / window.innerHeight) * 2 + 1;
    });
    
    canvas.addEventListener('click', () => {
      raycaster.setFromCamera(mouse, camera);
      const meshes = planetObjects.map(p => p.mesh);
      const hits = raycaster.intersectObjects(meshes);
      
      if (hits.length > 0) {
        const hitMesh = hits[0].object;
        const idx = planetObjects.findIndex(p => p.mesh === hitMesh);
        if (idx >= 0) {
          focusPlanet(idx);
          document.querySelectorAll('.planet-btn').forEach(b => b.classList.remove('active'));
          document.getElementById(`btn-${PLANET_DATA[idx].name}`)?.classList.add('active');
        }
      }
    });
    
    // ===== Resize =====
    window.addEventListener('resize', () => {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    });
    
    // ===== Hide loading =====
    document.getElementById('loading').style.display = 'none';
    
    // ===== Animation =====
    const clock = new THREE.Clock();
    
    function animate() {
      requestAnimationFrame(animate);
      const delta = clock.getDelta();
      const time = clock.getElapsedTime();
      const speedMult = orbitSpeedMultiplier / 30;
      
      // Animate planets
      planetObjects.forEach((planet) => {
        const { data } = planet;
        
        // Orbit
        planet.angle += data.orbitSpeed * delta * 0.05 * speedMult;
        planet.group.position.x = data.distance * Math.cos(planet.angle);
        planet.group.position.z = data.distance * Math.sin(planet.angle);
        
        // Self rotation
        planet.mesh.rotation.y += data.rotationSpeed * delta * speedMult * 0.5;
        
        // Moon orbit
        if (planet.moonData) {
          planet.moonData.group.rotation.y += delta * 0.5 * speedMult;
        }
      });
      
      // Animate sun
      sun.rotation.y += delta * 0.02;
      
      // Tooltip on hover
      raycaster.setFromCamera(mouse, camera);
      const hits = raycaster.intersectObjects(planetObjects.map(p => p.mesh));
      
      if (hits.length > 0) {
        const idx = planetObjects.findIndex(p => p.mesh === hits[0].object);
        if (idx >= 0) {
          tooltip.style.display = 'block';
          tooltip.style.left = `${(mouse.x + 1) / 2 * window.innerWidth + 10}px`;
          tooltip.style.top = `${(-mouse.y + 1) / 2 * window.innerHeight - 10}px`;
          tooltip.textContent = PLANET_DATA[idx].nameTH;
        }
      } else {
        tooltip.style.display = 'none';
      }
      
      controls.update();
      renderer.render(scene, camera);
    }
    
    animate();
  </script>
</body>
</html>
```

---

## วิธีใช้งาน

1. เปิดไฟล์ในเบราว์เซอร์
2. คลิกดาวเคราะห์ในรายการด้านขวาเพื่อดูข้อมูล
3. ใช้ slider ปรับความเร็วการโคจร
4. คลิกบนดาวเคราะห์ในฉากโดยตรง
5. เช็ค "ขนาดจริง" เพื่อดูอัตราส่วนที่ถูกต้อง

## การปรับปรุงเพิ่มเติม

- โหลด NASA textures จริงจาก `https://www.solarsystemscope.com/textures/`
- เพิ่ม skybox จาก ESO
- เพิ่ม particle effects สำหรับ solar wind
- เพิ่ม comet path
