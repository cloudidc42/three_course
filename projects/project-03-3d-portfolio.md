# Project 03: Interactive 3D Portfolio

## ภาพรวม

Portfolio website แบบ 3D interactive พร้อม:
- Scroll-based camera path
- Floating 3D portfolio items
- Click to expand details
- About / Skills / Projects / Contact sections
- Particle background
- Custom shader background
- Mobile responsive
- Loading screen

---

## โค้ดสมบูรณ์

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta name="description" content="3D Portfolio - Three.js Interactive Experience">
  <title>3D Portfolio</title>
  <style>
    :root {
      --accent: #4488ff;
      --accent2: #44ffcc;
      --bg: #050510;
      --text: #ffffff;
      --text-dim: #8899aa;
    }
    
    * { margin: 0; padding: 0; box-sizing: border-box; }
    
    body {
      background: var(--bg);
      color: var(--text);
      font-family: 'Segoe UI', system-ui, sans-serif;
      overflow-x: hidden;
    }
    
    /* ===== Canvas ===== */
    #canvas-container {
      position: fixed; inset: 0;
      z-index: 0;
    }
    canvas { display: block; }
    
    /* ===== Loading Screen ===== */
    #loading {
      position: fixed; inset: 0;
      background: var(--bg);
      display: flex; flex-direction: column;
      align-items: center; justify-content: center;
      z-index: 1000;
      transition: opacity 1s ease;
    }
    .loading-ring {
      width: 60px; height: 60px;
      border: 3px solid rgba(68,136,255,0.2);
      border-top-color: var(--accent);
      border-radius: 50%;
      animation: spin 1s linear infinite;
    }
    @keyframes spin { to { transform: rotate(360deg); } }
    #loading p { margin-top: 16px; color: var(--text-dim); font-size: 13px; }
    
    /* ===== Navigation ===== */
    #nav {
      position: fixed; top: 0; left: 0; right: 0;
      z-index: 100;
      display: flex; align-items: center; justify-content: space-between;
      padding: 20px 40px;
      background: linear-gradient(to bottom, rgba(5,5,16,0.8), transparent);
    }
    #nav-logo {
      font-size: 20px; font-weight: 700;
      background: linear-gradient(135deg, var(--accent), var(--accent2));
      -webkit-background-clip: text; -webkit-text-fill-color: transparent;
    }
    #nav-links { display: flex; gap: 30px; list-style: none; }
    #nav-links a {
      color: var(--text-dim); text-decoration: none;
      font-size: 13px; transition: color 0.2s;
    }
    #nav-links a:hover { color: var(--text); }
    
    /* ===== Scroll Container ===== */
    #scroll-container {
      position: relative; z-index: 10;
      pointer-events: none;
    }
    
    /* ===== Sections ===== */
    .section {
      min-height: 100vh;
      display: flex; align-items: center;
      padding: 80px 40px;
      pointer-events: auto;
    }
    
    .section-content {
      max-width: 500px;
    }
    
    .section-tag {
      font-size: 11px; letter-spacing: 3px;
      color: var(--accent); text-transform: uppercase;
      margin-bottom: 10px;
    }
    
    .section h2 {
      font-size: clamp(28px, 5vw, 48px);
      line-height: 1.2; margin-bottom: 16px;
      background: linear-gradient(135deg, #fff 60%, var(--accent));
      -webkit-background-clip: text; -webkit-text-fill-color: transparent;
    }
    
    .section p {
      color: var(--text-dim); line-height: 1.7; font-size: 15px;
    }
    
    /* ===== Hero ===== */
    #hero { justify-content: center; text-align: center; }
    #hero .section-content { max-width: 700px; }
    #hero h1 {
      font-size: clamp(36px, 8vw, 72px);
      background: linear-gradient(135deg, #fff 40%, var(--accent), var(--accent2));
      -webkit-background-clip: text; -webkit-text-fill-color: transparent;
      margin-bottom: 16px;
    }
    #hero p { font-size: 18px; color: var(--text-dim); }
    
    .cta-btn {
      display: inline-block;
      margin-top: 30px; padding: 14px 32px;
      background: transparent;
      border: 1px solid var(--accent);
      color: var(--accent); border-radius: 30px;
      cursor: pointer; font-size: 14px;
      letter-spacing: 1px;
      transition: all 0.3s;
      pointer-events: auto;
    }
    .cta-btn:hover {
      background: var(--accent);
      color: white; transform: translateY(-2px);
      box-shadow: 0 8px 30px rgba(68,136,255,0.3);
    }
    
    /* ===== Skills ===== */
    .skills-grid {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(120px, 1fr));
      gap: 10px; margin-top: 20px;
    }
    .skill-chip {
      padding: 8px 12px;
      background: rgba(68,136,255,0.1);
      border: 1px solid rgba(68,136,255,0.2);
      border-radius: 20px; font-size: 12px;
      text-align: center; color: var(--text-dim);
      transition: all 0.2s;
    }
    .skill-chip:hover {
      background: rgba(68,136,255,0.2);
      color: var(--text);
    }
    
    /* ===== Projects Grid ===== */
    #projects { flex-direction: column; align-items: flex-start; }
    #projects .section-content { max-width: 100%; width: 100%; }
    
    .projects-grid {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
      gap: 20px; margin-top: 30px;
    }
    
    .project-card {
      background: rgba(255,255,255,0.03);
      border: 1px solid rgba(255,255,255,0.08);
      border-radius: 12px; padding: 20px;
      cursor: pointer;
      transition: all 0.3s;
    }
    .project-card:hover {
      background: rgba(68,136,255,0.05);
      border-color: rgba(68,136,255,0.3);
      transform: translateY(-4px);
    }
    .project-card h3 { font-size: 16px; margin-bottom: 8px; }
    .project-card p { font-size: 13px; color: var(--text-dim); line-height: 1.6; }
    .project-tags { margin-top: 12px; display: flex; gap: 6px; flex-wrap: wrap; }
    .project-tag {
      padding: 3px 8px; border-radius: 10px;
      font-size: 11px; background: rgba(68,136,255,0.15);
      color: var(--accent);
    }
    
    /* ===== Contact ===== */
    .contact-links {
      display: flex; gap: 15px; margin-top: 20px; flex-wrap: wrap;
    }
    .contact-link {
      padding: 10px 20px;
      background: rgba(255,255,255,0.05);
      border: 1px solid rgba(255,255,255,0.1);
      border-radius: 8px; color: var(--text-dim);
      text-decoration: none; font-size: 13px;
      transition: all 0.2s;
      pointer-events: auto;
    }
    .contact-link:hover {
      background: rgba(68,136,255,0.1);
      color: var(--text);
    }
    
    /* ===== Scroll indicator ===== */
    #scroll-indicator {
      position: fixed; bottom: 30px; left: 50%;
      transform: translateX(-50%);
      color: var(--text-dim); font-size: 12px;
      text-align: center; z-index: 50;
      animation: bounce 2s ease infinite;
      pointer-events: none;
    }
    @keyframes bounce {
      0%, 100% { transform: translateX(-50%) translateY(0); }
      50% { transform: translateX(-50%) translateY(-8px); }
    }
    
    /* ===== 3D Object Label ===== */
    #object-tooltip {
      position: fixed;
      background: rgba(0,0,0,0.8);
      border: 1px solid rgba(68,136,255,0.4);
      color: white; padding: 8px 14px;
      border-radius: 6px; font-size: 13px;
      pointer-events: none; display: none;
      z-index: 200;
      backdrop-filter: blur(5px);
    }
    
    /* Mobile */
    @media (max-width: 768px) {
      #nav-links { display: none; }
      .section { padding: 60px 20px; }
      #nav { padding: 15px 20px; }
    }
  </style>
</head>
<body>

<div id="loading">
  <div class="loading-ring"></div>
  <p>กำลังโหลด 3D Portfolio...</p>
</div>

<div id="canvas-container">
  <canvas id="canvas"></canvas>
</div>

<!-- Navigation -->
<nav id="nav">
  <div id="nav-logo">Portfolio</div>
  <ul id="nav-links">
    <li><a href="#hero">หน้าแรก</a></li>
    <li><a href="#about">เกี่ยวกับ</a></li>
    <li><a href="#skills">ทักษะ</a></li>
    <li><a href="#projects">ผลงาน</a></li>
    <li><a href="#contact">ติดต่อ</a></li>
  </ul>
</nav>

<!-- Scroll Sections -->
<div id="scroll-container">
  <!-- Hero -->
  <section class="section" id="hero">
    <div class="section-content">
      <h1>สวัสดี ฉันคือ<br>นักพัฒนา 3D</h1>
      <p>สร้างประสบการณ์ 3D บน Web ด้วย Three.js<br>WebGL และ Creative Coding</p>
      <button class="cta-btn" onclick="document.getElementById('projects').scrollIntoView({behavior:'smooth'})">
        ดูผลงาน ↓
      </button>
    </div>
  </section>
  
  <!-- About -->
  <section class="section" id="about" style="justify-content: flex-end;">
    <div class="section-content">
      <div class="section-tag">About Me</div>
      <h2>เกี่ยวกับฉัน</h2>
      <p>นักพัฒนา Web ที่หลงใหลใน 3D Graphics และ Interactive Experiences มีประสบการณ์กว่า 5 ปีในการสร้าง web apps ที่สร้างสรรค์</p>
      <p style="margin-top:12px;">ชอบสำรวจขอบเขตของสิ่งที่เป็นไปได้บน browser โดยใช้ WebGL, Three.js และ Shader programming</p>
    </div>
  </section>
  
  <!-- Skills -->
  <section class="section" id="skills">
    <div class="section-content">
      <div class="section-tag">Skills</div>
      <h2>ทักษะและเทคโนโลยี</h2>
      <div class="skills-grid">
        <div class="skill-chip">Three.js</div>
        <div class="skill-chip">WebGL / GLSL</div>
        <div class="skill-chip">React</div>
        <div class="skill-chip">TypeScript</div>
        <div class="skill-chip">Node.js</div>
        <div class="skill-chip">GSAP</div>
        <div class="skill-chip">Blender</div>
        <div class="skill-chip">WebGPU</div>
        <div class="skill-chip">Vite</div>
        <div class="skill-chip">Figma</div>
      </div>
    </div>
  </section>
  
  <!-- Projects -->
  <section class="section" id="projects">
    <div class="section-content">
      <div class="section-tag">Projects</div>
      <h2>ผลงาน</h2>
      <div class="projects-grid">
        <div class="project-card">
          <h3>🌍 3D Data Globe</h3>
          <p>Visualize ข้อมูลระดับโลกด้วย interactive 3D globe พร้อม flight paths</p>
          <div class="project-tags">
            <span class="project-tag">Three.js</span>
            <span class="project-tag">D3.js</span>
            <span class="project-tag">WebGL</span>
          </div>
        </div>
        <div class="project-card">
          <h3>🏠 Archviz Viewer</h3>
          <p>Real-time architectural visualization พร้อม walkthrough และ PBR materials</p>
          <div class="project-tags">
            <span class="project-tag">GLTF</span>
            <span class="project-tag">HDRI</span>
            <span class="project-tag">React</span>
          </div>
        </div>
        <div class="project-card">
          <h3>🎮 WebGL Game</h3>
          <p>Browser-based 3D game พร้อม ECS architecture และ physics simulation</p>
          <div class="project-tags">
            <span class="project-tag">Three.js</span>
            <span class="project-tag">ECS</span>
            <span class="project-tag">Cannon.js</span>
          </div>
        </div>
        <div class="project-card">
          <h3>✨ Shader Art</h3>
          <p>Generative art ด้วย custom GLSL shaders และ compute shaders</p>
          <div class="project-tags">
            <span class="project-tag">GLSL</span>
            <span class="project-tag">WebGPU</span>
            <span class="project-tag">TSL</span>
          </div>
        </div>
      </div>
    </div>
  </section>
  
  <!-- Contact -->
  <section class="section" id="contact" style="justify-content: flex-end;">
    <div class="section-content">
      <div class="section-tag">Contact</div>
      <h2>ติดต่อ</h2>
      <p>สนใจร่วมงานหรือมีโปรเจกต์ที่น่าสนใจ? ติดต่อฉันได้เลย!</p>
      <div class="contact-links">
        <a class="contact-link" href="mailto:hello@example.com">📧 Email</a>
        <a class="contact-link" href="#">💼 LinkedIn</a>
        <a class="contact-link" href="#">🐙 GitHub</a>
        <a class="contact-link" href="#">🐦 Twitter</a>
      </div>
    </div>
  </section>
</div>

<div id="scroll-indicator">↓ เลื่อนลง</div>
<div id="object-tooltip"></div>

<script type="module">
  import * as THREE from 'https://cdn.skypack.dev/three@0.160.0';
  import { OrbitControls } from 'https://cdn.skypack.dev/three@0.160.0/examples/jsm/controls/OrbitControls.js';
  
  // ===== Setup =====
  const canvas = document.getElementById('canvas');
  const renderer = new THREE.WebGLRenderer({ canvas, antialias: true, alpha: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.outputColorSpace = THREE.SRGBColorSpace;
  renderer.toneMapping = THREE.ACESFilmicToneMapping;
  renderer.toneMappingExposure = 1.0;
  
  const scene = new THREE.Scene();
  
  const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 200);
  camera.position.set(0, 0, 12);
  
  // ===== Background Gradient Shader =====
  const bgUniforms = {
    uTime: { value: 0 },
    uScrollY: { value: 0 },
    uResolution: { value: new THREE.Vector2(window.innerWidth, window.innerHeight) }
  };
  
  const bgMaterial = new THREE.ShaderMaterial({
    uniforms: bgUniforms,
    vertexShader: `
      varying vec2 vUv;
      void main() {
        vUv = uv;
        gl_Position = vec4(position.xy, 1.0, 1.0);
      }
    `,
    fragmentShader: `
      uniform float uTime;
      uniform float uScrollY;
      uniform vec2 uResolution;
      varying vec2 vUv;
      
      // Simplex-like hash
      float hash(vec2 p) {
        p = fract(p * vec2(127.1, 311.7));
        p += dot(p, p + 19.19);
        return fract(p.x * p.y);
      }
      
      float noise(vec2 p) {
        vec2 i = floor(p);
        vec2 f = fract(p);
        f = f * f * (3.0 - 2.0 * f);
        return mix(
          mix(hash(i), hash(i + vec2(1,0)), f.x),
          mix(hash(i + vec2(0,1)), hash(i + vec2(1,1)), f.x),
          f.y
        );
      }
      
      void main() {
        vec2 uv = vUv;
        float scroll = uScrollY * 0.3;
        
        // Nebula base
        vec3 col = vec3(0.02, 0.02, 0.08);
        
        // Noise layers
        float n1 = noise(uv * 3.0 + vec2(uTime * 0.05, scroll * 0.1));
        float n2 = noise(uv * 6.0 - vec2(uTime * 0.03, -scroll * 0.15));
        float n3 = noise(uv * 12.0 + vec2(uTime * 0.02));
        
        float nebula = n1 * 0.5 + n2 * 0.3 + n3 * 0.2;
        
        // Color variation
        vec3 c1 = vec3(0.05, 0.1, 0.3);  // Blue
        vec3 c2 = vec3(0.1, 0.05, 0.25); // Purple
        vec3 c3 = vec3(0.0, 0.15, 0.2);  // Teal
        
        col = mix(col, c1, nebula * 0.6);
        col = mix(col, c2, noise(uv * 4.0 + 5.3) * 0.3);
        col = mix(col, c3, noise(uv * 5.0 + uTime * 0.02) * 0.2);
        
        // Edge vignette
        vec2 center = uv - 0.5;
        float vign = 1.0 - dot(center, center) * 1.5;
        col *= vign;
        
        gl_FragColor = vec4(col, 1.0);
      }
    `,
    depthTest: false,
    depthWrite: false
  });
  
  const bgMesh = new THREE.Mesh(new THREE.PlaneGeometry(2, 2), bgMaterial);
  bgMesh.renderOrder = -1;
  scene.add(bgMesh);
  
  // ===== Particles =====
  function createParticles() {
    const count = 3000;
    const geo = new THREE.BufferGeometry();
    const positions = new Float32Array(count * 3);
    const colors = new Float32Array(count * 3);
    const sizes = new Float32Array(count);
    
    for (let i = 0; i < count; i++) {
      positions[i * 3] = (Math.random() - 0.5) * 50;
      positions[i * 3 + 1] = (Math.random() - 0.5) * 50;
      positions[i * 3 + 2] = (Math.random() - 0.5) * 30;
      
      const t = Math.random();
      colors[i * 3] = 0.3 + t * 0.5;
      colors[i * 3 + 1] = 0.5 + t * 0.3;
      colors[i * 3 + 2] = 0.8 + t * 0.2;
      
      sizes[i] = Math.random() * 2 + 0.5;
    }
    
    geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    geo.setAttribute('color', new THREE.BufferAttribute(colors, 3));
    geo.setAttribute('size', new THREE.BufferAttribute(sizes, 1));
    
    const mat = new THREE.PointsMaterial({
      size: 0.05,
      vertexColors: true,
      transparent: true,
      opacity: 0.6,
      sizeAttenuation: true
    });
    
    return new THREE.Points(geo, mat);
  }
  
  const particles = createParticles();
  scene.add(particles);
  
  // ===== 3D Floating Objects =====
  const floatingObjects = [];
  
  const objectDefs = [
    { geometry: new THREE.IcosahedronGeometry(0.6, 1), color: 0x4488ff, label: 'Three.js', pos: [4, 2, -2] },
    { geometry: new THREE.TorusGeometry(0.6, 0.2, 8, 20), color: 0x44ffcc, label: 'WebGL', pos: [-5, 1, -3] },
    { geometry: new THREE.OctahedronGeometry(0.7), color: 0xff8844, label: 'GLSL', pos: [3, -3, -1] },
    { geometry: new THREE.TetrahedronGeometry(0.7), color: 0x88ff44, label: 'React', pos: [-4, -2, -2] },
    { geometry: new THREE.DodecahedronGeometry(0.6), color: 0xdd44ff, label: 'Node.js', pos: [6, -1, -4] },
    { geometry: new THREE.TorusKnotGeometry(0.4, 0.15, 60, 8), color: 0xff4488, label: 'GSAP', pos: [-6, 3, -3] }
  ];
  
  const tooltip = document.getElementById('object-tooltip');
  
  objectDefs.forEach(def => {
    const mat = new THREE.MeshStandardMaterial({
      color: def.color,
      metalness: 0.7,
      roughness: 0.2,
      emissive: new THREE.Color(def.color).multiplyScalar(0.1)
    });
    
    const mesh = new THREE.Mesh(def.geometry, mat);
    mesh.position.set(...def.pos);
    mesh.userData.label = def.label;
    mesh.userData.baseY = def.pos[1];
    mesh.userData.phaseOffset = Math.random() * Math.PI * 2;
    
    scene.add(mesh);
    floatingObjects.push(mesh);
    
    // Wireframe overlay
    const wireMat = new THREE.MeshBasicMaterial({
      color: def.color,
      wireframe: true,
      transparent: true,
      opacity: 0.15
    });
    const wireMesh = new THREE.Mesh(def.geometry, wireMat);
    wireMesh.scale.setScalar(1.02);
    mesh.add(wireMesh);
  });
  
  // ===== Lighting =====
  scene.add(new THREE.AmbientLight(0x334455, 0.5));
  
  const mainLight = new THREE.DirectionalLight(0x8899ff, 1.5);
  mainLight.position.set(5, 10, 5);
  scene.add(mainLight);
  
  const fillLight = new THREE.PointLight(0x44ffcc, 1, 30);
  fillLight.position.set(-8, 0, 0);
  scene.add(fillLight);
  
  const backLight = new THREE.PointLight(0xff4488, 0.5, 20);
  backLight.position.set(0, -5, -10);
  scene.add(backLight);
  
  // ===== Scroll Camera Path =====
  // Camera follows a spline path as user scrolls
  const cameraPath = [
    new THREE.Vector3(0, 0, 12),   // Hero
    new THREE.Vector3(-3, 1, 10),  // About
    new THREE.Vector3(3, -1, 10),  // Skills
    new THREE.Vector3(0, 0, 8),    // Projects
    new THREE.Vector3(3, 1, 10)    // Contact
  ];
  
  const cameraSpline = new THREE.CatmullRomCurve3(cameraPath);
  
  const lookAtPath = [
    new THREE.Vector3(0, 0, 0),
    new THREE.Vector3(-1, 0, 0),
    new THREE.Vector3(1, 0, 0),
    new THREE.Vector3(0, 0, 0),
    new THREE.Vector3(1, 0, 0)
  ];
  const lookAtSpline = new THREE.CatmullRomCurve3(lookAtPath);
  
  let scrollProgress = 0;
  let targetScrollProgress = 0;
  
  // ===== Raycasting for hover =====
  const raycaster = new THREE.Raycaster();
  const mouse = new THREE.Vector2();
  let hoveredObject = null;
  
  window.addEventListener('mousemove', e => {
    mouse.x = (e.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(e.clientY / window.innerHeight) * 2 + 1;
    
    // Update tooltip position
    tooltip.style.left = `${e.clientX + 12}px`;
    tooltip.style.top = `${e.clientY - 10}px`;
  });
  
  // ===== Scroll handler =====
  const totalScrollHeight = document.body.scrollHeight - window.innerHeight;
  
  window.addEventListener('scroll', () => {
    const progress = window.scrollY / totalScrollHeight;
    targetScrollProgress = Math.max(0, Math.min(1, progress));
    bgUniforms.uScrollY.value = window.scrollY;
    
    // Hide scroll indicator after first scroll
    if (window.scrollY > 50) {
      document.getElementById('scroll-indicator').style.opacity = '0';
    }
  });
  
  // ===== Click on 3D objects =====
  canvas.addEventListener('click', () => {
    if (hoveredObject) {
      // Animate the clicked object
      const obj = hoveredObject;
      const origScale = obj.scale.x;
      
      // Pulse animation
      let t = 0;
      const animClick = () => {
        t += 0.1;
        obj.scale.setScalar(origScale + Math.sin(t * 5) * 0.2 * Math.exp(-t));
        if (t < 2) requestAnimationFrame(animClick);
        else obj.scale.setScalar(origScale);
      };
      animClick();
    }
  });
  
  // ===== Resize =====
  window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
    bgUniforms.uResolution.value.set(window.innerWidth, window.innerHeight);
  });
  
  // ===== Hide loading =====
  setTimeout(() => {
    const loading = document.getElementById('loading');
    loading.style.opacity = '0';
    setTimeout(() => loading.style.display = 'none', 1000);
  }, 800);
  
  // ===== Animation =====
  const clock = new THREE.Clock();
  
  // Smooth scroll progress
  const lerpedProgress = { value: 0 };
  
  function animate() {
    requestAnimationFrame(animate);
    
    const time = clock.getElapsedTime();
    const delta = clock.getDelta();
    
    // Smooth scroll
    lerpedProgress.value += (targetScrollProgress - lerpedProgress.value) * 0.05;
    
    // Camera along spline
    const camPos = cameraSpline.getPoint(lerpedProgress.value);
    camera.position.lerp(camPos, 0.08);
    
    const lookAtPos = lookAtSpline.getPoint(lerpedProgress.value);
    camera.lookAt(lookAtPos);
    
    // Floating objects animation
    floatingObjects.forEach(obj => {
      const phase = obj.userData.phaseOffset;
      obj.position.y = obj.userData.baseY + Math.sin(time * 0.7 + phase) * 0.3;
      obj.rotation.x = time * 0.2 + phase;
      obj.rotation.y = time * 0.3;
    });
    
    // Particle drift
    particles.rotation.y = time * 0.01;
    particles.rotation.x = time * 0.005;
    
    // Background shader
    bgUniforms.uTime.value = time;
    
    // Light animation
    fillLight.position.x = Math.sin(time * 0.5) * 10;
    fillLight.position.y = Math.cos(time * 0.3) * 5;
    
    // Hover detection
    raycaster.setFromCamera(mouse, camera);
    const hits = raycaster.intersectObjects(floatingObjects);
    
    if (hits.length > 0) {
      const newHovered = hits[0].object;
      
      if (newHovered !== hoveredObject) {
        // Unhover old
        if (hoveredObject) {
          hoveredObject.scale.setScalar(1);
        }
        
        hoveredObject = newHovered;
        hoveredObject.scale.setScalar(1.1);
        
        tooltip.style.display = 'block';
        tooltip.textContent = hoveredObject.userData.label || '';
      }
    } else {
      if (hoveredObject) {
        hoveredObject.scale.setScalar(1);
        hoveredObject = null;
      }
      tooltip.style.display = 'none';
    }
    
    renderer.render(scene, camera);
  }
  
  animate();
  
  // ===== Nav Smooth Scroll =====
  document.querySelectorAll('#nav-links a').forEach(link => {
    link.addEventListener('click', e => {
      e.preventDefault();
      const target = document.querySelector(link.getAttribute('href'));
      if (target) {
        target.scrollIntoView({ behavior: 'smooth' });
      }
    });
  });
</script>
</body>
</html>
```

---

## Features

- **Background Shader**: Animated nebula GLSL shader
- **Particle System**: 3000 particles drifting in space
- **Scroll Camera Path**: CatmullRom spline ตาม scroll
- **Floating 3D Objects**: Interactive geometry แต่ละ skill
- **Hover Effects**: Scale up + tooltip บน 3D objects
- **Click Animation**: Pulse effect เมื่อ click object
- **Responsive**: รองรับ mobile

## การปรับปรุงเพิ่มเติม

- เพิ่ม GSAP scroll animations
- ใช้ real project screenshots เป็น texture บน 3D frames
- เพิ่ม contact form ที่ functional
- Deploy ไป Vercel/Netlify
