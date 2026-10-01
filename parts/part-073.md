# Part 73: CSS3DRenderer & CSS2DRenderer

## บทนำ

Three.js มี renderer พิเศษ 2 ตัวสำหรับ HTML elements ใน 3D space:
- **CSS3DRenderer** - แปลง HTML elements เป็น 3D objects จริงๆ ใน perspective
- **CSS2DRenderer** - HTML elements ที่ follow 3D position แต่ไม่ distort

ทั้งสองใช้ร่วมกับ WebGLRenderer เพื่อผสม HTML กับ 3D

---

## 73.1 CSS2DRenderer - Labels และ Annotations

```javascript
// src/demos/css2dLabel.js
import * as THREE from 'three';
import { CSS2DRenderer, CSS2DObject } from 'three/examples/jsm/renderers/CSS2DRenderer.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

export function createCSS2DDemo(container) {
  container.style.position = 'relative';
  
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#0a0a1a');
  
  const camera = new THREE.PerspectiveCamera(45, container.clientWidth / container.clientHeight, 0.1, 200);
  camera.position.set(0, 5, 20);
  
  // WebGL Renderer
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.shadowMap.enabled = true;
  container.appendChild(renderer.domElement);
  
  // CSS2D Renderer
  const labelRenderer = new CSS2DRenderer();
  labelRenderer.setSize(container.clientWidth, container.clientHeight);
  labelRenderer.domElement.style.cssText = `
    position: absolute;
    top: 0;
    left: 0;
    pointer-events: none;
  `;
  container.appendChild(labelRenderer.domElement);
  
  const controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  
  // === Solar System ===
  const PLANET_DATA = [
    { name: 'Sun', radius: 2, distance: 0, color: '#FFF200', emissive: '#FF6600', speed: 0, desc: 'Our Star' },
    { name: 'Mercury', radius: 0.3, distance: 4, color: '#b5b5b5', speed: 4.7, desc: '88 days orbit' },
    { name: 'Venus', radius: 0.6, distance: 6, color: '#e8cda0', speed: 3.5, desc: '225 days orbit' },
    { name: 'Earth', radius: 0.65, distance: 8.5, color: '#4fc3f7', speed: 3, desc: '365 days orbit' },
    { name: 'Mars', radius: 0.45, distance: 11, color: '#c1440e', speed: 2.4, desc: '687 days orbit' },
    { name: 'Jupiter', radius: 1.3, distance: 15, color: '#c88b3a', speed: 1.3, desc: '12 year orbit' },
    { name: 'Saturn', radius: 1.1, distance: 20, color: '#e8d5a3', speed: 0.97, desc: '29 year orbit' },
  ];
  
  const orbits = [];
  const planets = [];
  
  PLANET_DATA.forEach(data => {
    // Orbit line
    if (data.distance > 0) {
      const orbitGeo = new THREE.BufferGeometry();
      const points = [];
      for (let i = 0; i <= 64; i++) {
        const angle = (i / 64) * Math.PI * 2;
        points.push(new THREE.Vector3(
          Math.cos(angle) * data.distance,
          0,
          Math.sin(angle) * data.distance
        ));
      }
      orbitGeo.setFromPoints(points);
      const orbit = new THREE.Line(
        orbitGeo,
        new THREE.LineBasicMaterial({ color: 'rgba(255,255,255,0.2)', transparent: true, opacity: 0.2 })
      );
      scene.add(orbit);
    }
    
    // Planet mesh
    const mat = new THREE.MeshStandardMaterial({ color: data.color });
    if (data.emissive) {
      mat.emissive = new THREE.Color(data.emissive);
      mat.emissiveIntensity = 0.5;
    }
    
    const mesh = new THREE.Mesh(
      new THREE.SphereGeometry(data.radius, 32, 32),
      mat
    );
    mesh.castShadow = true;
    mesh.position.x = data.distance;
    scene.add(mesh);
    
    // Saturn rings
    if (data.name === 'Saturn') {
      const ringGeo = new THREE.RingGeometry(data.radius * 1.4, data.radius * 2.2, 64);
      const ringMat = new THREE.MeshBasicMaterial({
        color: '#c8a060',
        side: THREE.DoubleSide,
        transparent: true,
        opacity: 0.7
      });
      const ring = new THREE.Mesh(ringGeo, ringMat);
      ring.rotation.x = Math.PI / 3;
      mesh.add(ring);
    }
    
    // CSS2D Label
    const labelDiv = document.createElement('div');
    labelDiv.className = 'planet-label';
    labelDiv.style.cssText = `
      background: rgba(0,0,0,0.75);
      border: 1px solid ${data.color};
      border-radius: 6px;
      padding: 5px 10px;
      color: white;
      font-family: monospace;
      font-size: 12px;
      white-space: nowrap;
      pointer-events: none;
    `;
    labelDiv.innerHTML = `
      <strong style="color: ${data.color}">${data.name}</strong>
      <br><span style="color: #888">${data.desc}</span>
    `;
    
    const label = new CSS2DObject(labelDiv);
    label.position.set(0, data.radius + 0.5, 0);
    mesh.add(label);
    
    planets.push({ mesh, data, angle: Math.random() * Math.PI * 2 });
  });
  
  // Sun light
  const sunLight = new THREE.PointLight(0xffffff, 2, 100);
  sunLight.position.set(0, 0, 0);
  scene.add(sunLight);
  scene.add(new THREE.AmbientLight(0x111122, 0.5));
  
  // Starfield
  const starPositions = new Float32Array(3000 * 3);
  for (let i = 0; i < 3000; i++) {
    starPositions[i * 3] = (Math.random() - 0.5) * 300;
    starPositions[i * 3 + 1] = (Math.random() - 0.5) * 300;
    starPositions[i * 3 + 2] = (Math.random() - 0.5) * 300;
  }
  const starGeo = new THREE.BufferGeometry();
  starGeo.setAttribute('position', new THREE.BufferAttribute(starPositions, 3));
  scene.add(new THREE.Points(starGeo, new THREE.PointsMaterial({ size: 0.1, color: 'white' })));
  
  // Animation
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    
    // Orbit planets
    planets.forEach(p => {
      if (p.data.distance > 0) {
        p.angle += p.data.speed * 0.005;
        p.mesh.position.x = Math.cos(p.angle) * p.data.distance;
        p.mesh.position.z = Math.sin(p.angle) * p.data.distance;
        p.mesh.rotation.y = elapsed * 0.5;
      }
    });
    
    controls.update();
    renderer.render(scene, camera);
    labelRenderer.render(scene, camera);
  }
  
  animate();
  
  window.addEventListener('resize', () => {
    const w = container.clientWidth;
    const h = container.clientHeight;
    camera.aspect = w / h;
    camera.updateProjectionMatrix();
    renderer.setSize(w, h);
    labelRenderer.setSize(w, h);
  });
  
  return () => {
    cancelAnimationFrame(animId);
    controls.dispose();
    renderer.dispose();
  };
}
```

---

## 73.2 CSS3DRenderer - 3D HTML Cards

```javascript
// src/demos/css3dCards.js
import * as THREE from 'three';
import { CSS3DRenderer, CSS3DObject, CSS3DSprite } from 'three/examples/jsm/renderers/CSS3DRenderer.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

export function createCSS3DCardsDemo(container) {
  container.style.position = 'relative';
  container.style.overflow = 'hidden';
  
  // Scenes
  const scene = new THREE.Scene();
  const cssScene = new THREE.Scene();
  
  const camera = new THREE.PerspectiveCamera(40, container.clientWidth / container.clientHeight, 1, 10000);
  camera.position.set(0, 200, 1200);
  
  // WebGL Renderer (background stars)
  const webglRenderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
  webglRenderer.setSize(container.clientWidth, container.clientHeight);
  webglRenderer.setClearColor(0x000000, 1);
  container.appendChild(webglRenderer.domElement);
  
  // CSS3D Renderer
  const cssRenderer = new CSS3DRenderer();
  cssRenderer.setSize(container.clientWidth, container.clientHeight);
  cssRenderer.domElement.style.cssText = `
    position: absolute;
    top: 0; left: 0;
  `;
  container.appendChild(cssRenderer.domElement);
  
  // Controls
  const controls = new OrbitControls(camera, cssRenderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
  controls.minDistance = 300;
  controls.maxDistance = 3000;
  
  // === Starfield (WebGL) ===
  const starCount = 3000;
  const starPos = new Float32Array(starCount * 3);
  for (let i = 0; i < starCount; i++) {
    starPos[i * 3] = (Math.random() - 0.5) * 8000;
    starPos[i * 3 + 1] = (Math.random() - 0.5) * 8000;
    starPos[i * 3 + 2] = (Math.random() - 0.5) * 8000;
  }
  const starGeo = new THREE.BufferGeometry();
  starGeo.setAttribute('position', new THREE.BufferAttribute(starPos, 3));
  scene.add(new THREE.Points(starGeo, new THREE.PointsMaterial({ size: 2, color: 'white', transparent: true, opacity: 0.6 })));
  
  // === CSS3DObject Cards ===
  const PROJECTS = [
    {
      name: 'E-Commerce Platform',
      tech: 'React + Node.js',
      year: '2024',
      color: '#4ecdc4',
      desc: 'Full-stack shopping platform with real-time inventory',
      metrics: { users: '50K', revenue: '$2M', uptime: '99.9%' }
    },
    {
      name: '3D Game Engine',
      tech: 'Three.js + WebGL',
      year: '2024',
      color: '#ff6b6b',
      desc: 'Browser-based 3D game engine with physics',
      metrics: { fps: '60', objects: '10K', draw: '< 5ms' }
    },
    {
      name: 'ML Dashboard',
      tech: 'Python + React',
      year: '2023',
      color: '#ffd166',
      desc: 'Real-time ML model monitoring and visualization',
      metrics: { models: '150', accuracy: '94%', latency: '12ms' }
    },
    {
      name: 'Mobile App',
      tech: 'React Native',
      year: '2023',
      color: '#a8edea',
      desc: 'Cross-platform fitness tracking application',
      metrics: { downloads: '100K', rating: '4.8', sessions: '2M' }
    },
    {
      name: 'Cloud Infrastructure',
      tech: 'Kubernetes + Terraform',
      year: '2024',
      color: '#c3a6ff',
      desc: 'Auto-scaling microservices infrastructure',
      metrics: { services: '45', regions: '5', cost: '-60%' }
    },
    {
      name: 'Data Pipeline',
      tech: 'Spark + Kafka',
      year: '2023',
      color: '#95e1d3',
      desc: 'Real-time data processing pipeline',
      metrics: { events: '1B/day', lag: '< 50ms', uptime: '99.5%' }
    },
  ];
  
  function createProjectCard(project) {
    const div = document.createElement('div');
    div.style.cssText = `
      width: 280px;
      height: 200px;
      background: linear-gradient(135deg, rgba(10,10,30,0.95), rgba(20,20,50,0.95));
      border: 1px solid ${project.color}44;
      border-radius: 16px;
      padding: 24px;
      color: white;
      font-family: 'Courier New', monospace;
      cursor: pointer;
      transition: border-color 0.3s;
      box-shadow: 0 0 30px ${project.color}22;
      position: relative;
      overflow: hidden;
    `;
    
    div.innerHTML = `
      <div style="
        position: absolute; top: 0; right: 0;
        width: 80px; height: 80px;
        background: radial-gradient(circle, ${project.color}22, transparent);
        border-radius: 0 16px 0 100%;
      "></div>
      
      <div style="display: flex; justify-content: space-between; align-items: flex-start; margin-bottom: 12px">
        <div>
          <div style="font-size: 0.8em; color: ${project.color}; letter-spacing: 2px; text-transform: uppercase">${project.tech}</div>
          <div style="font-size: 1.1em; font-weight: bold; color: white; margin-top: 4px">${project.name}</div>
        </div>
        <div style="font-size: 0.75em; color: #666; background: rgba(255,255,255,0.05); padding: 3px 8px; border-radius: 10px">${project.year}</div>
      </div>
      
      <div style="font-size: 0.75em; color: #aaa; margin-bottom: 16px; line-height: 1.5">${project.desc}</div>
      
      <div style="display: flex; gap: 10px;">
        ${Object.entries(project.metrics).map(([k, v]) => `
          <div style="flex: 1; text-align: center; background: rgba(255,255,255,0.05); border-radius: 8px; padding: 6px">
            <div style="font-size: 1em; font-weight: bold; color: ${project.color}">${v}</div>
            <div style="font-size: 0.65em; color: #666; text-transform: uppercase">${k}</div>
          </div>
        `).join('')}
      </div>
    `;
    
    div.addEventListener('mouseenter', () => {
      div.style.borderColor = project.color + 'aa';
      div.style.transform = 'scale(1.02)';
    });
    div.addEventListener('mouseleave', () => {
      div.style.borderColor = project.color + '44';
      div.style.transform = 'scale(1)';
    });
    
    return div;
  }
  
  // === Create objects in circular arrangement ===
  const cssObjects = [];
  
  PROJECTS.forEach((project, i) => {
    const el = createProjectCard(project);
    const cssObj = new CSS3DObject(el);
    
    // Circular arrangement
    const angle = (i / PROJECTS.length) * Math.PI * 2;
    const radius = 700;
    
    cssObj.position.x = Math.sin(angle) * radius;
    cssObj.position.y = (Math.random() - 0.5) * 200;
    cssObj.position.z = Math.cos(angle) * radius;
    
    // Face outward
    cssObj.rotation.y = -angle;
    
    cssScene.add(cssObj);
    cssObjects.push(cssObj);
  });
  
  // === WebGL planes behind cards (for depth occlusion) ===
  const planeMat = new THREE.MeshBasicMaterial({
    color: 0x000000,
    opacity: 0,
    transparent: true,
    side: THREE.DoubleSide
  });
  
  PROJECTS.forEach((_, i) => {
    const plane = new THREE.Mesh(
      new THREE.PlaneGeometry(300, 220),
      planeMat.clone()
    );
    plane.position.copy(cssObjects[i].position);
    plane.rotation.copy(cssObjects[i].rotation);
    scene.add(plane);
  });
  
  // === Layout Toggle ===
  const layoutTargets = {
    circle: PROJECTS.map((_, i) => {
      const angle = (i / PROJECTS.length) * Math.PI * 2;
      return {
        position: new THREE.Vector3(Math.sin(angle) * 700, 0, Math.cos(angle) * 700),
        rotation: new THREE.Euler(0, -angle, 0)
      };
    }),
    grid: PROJECTS.map((_, i) => ({
      position: new THREE.Vector3((i % 3 - 1) * 400, (Math.floor(i / 3) - 0.5) * -300, 0),
      rotation: new THREE.Euler(0, 0, 0)
    })),
    spiral: PROJECTS.map((_, i) => {
      const t = i / PROJECTS.length;
      const angle = t * Math.PI * 4;
      return {
        position: new THREE.Vector3(Math.sin(angle) * (300 + i * 50), (i - PROJECTS.length / 2) * 80, Math.cos(angle) * (300 + i * 50)),
        rotation: new THREE.Euler(0, -angle, 0)
      };
    }),
  };
  
  let currentLayout = 'circle';
  let transitioning = false;
  
  function transitionTo(layoutName) {
    if (transitioning) return;
    transitioning = true;
    currentLayout = layoutName;
    
    const targets = layoutTargets[layoutName];
    const duration = 1000;
    const start = performance.now();
    
    const startPositions = cssObjects.map(obj => obj.position.clone());
    const startRotations = cssObjects.map(obj => obj.rotation.clone());
    
    function update() {
      const now = performance.now();
      const progress = Math.min((now - start) / duration, 1);
      const ease = 1 - Math.pow(1 - progress, 3);
      
      cssObjects.forEach((obj, i) => {
        obj.position.lerpVectors(startPositions[i], targets[i].position, ease);
        obj.rotation.y = startRotations[i].y + (targets[i].rotation.y - startRotations[i].y) * ease;
      });
      
      if (progress < 1) requestAnimationFrame(update);
      else transitioning = false;
    }
    
    requestAnimationFrame(update);
  }
  
  // Layout buttons
  const btnsDiv = document.createElement('div');
  btnsDiv.style.cssText = `
    position: absolute;
    bottom: 30px;
    left: 50%;
    transform: translateX(-50%);
    display: flex;
    gap: 10px;
    z-index: 100;
  `;
  
  ['circle', 'grid', 'spiral'].forEach(layout => {
    const btn = document.createElement('button');
    btn.textContent = layout.charAt(0).toUpperCase() + layout.slice(1);
    btn.style.cssText = `
      padding: 10px 20px;
      background: rgba(0,0,0,0.8);
      border: 1px solid #4ecdc4;
      color: #4ecdc4;
      border-radius: 8px;
      cursor: pointer;
      font-family: monospace;
      letter-spacing: 1px;
      z-index: 200;
      pointer-events: all;
    `;
    btn.addEventListener('click', () => transitionTo(layout));
    btnsDiv.appendChild(btn);
  });
  
  container.appendChild(btnsDiv);
  
  // === Animation ===
  let animId;
  const clock = new THREE.Clock();
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    
    controls.update();
    
    // Gentle floating
    cssObjects.forEach((obj, i) => {
      const baseY = layoutTargets[currentLayout][i].position.y;
      obj.position.y = baseY + Math.sin(elapsed * 0.5 + i * 0.8) * 10;
    });
    
    webglRenderer.render(scene, camera);
    cssRenderer.render(cssScene, camera);
  }
  
  animate();
  
  window.addEventListener('resize', () => {
    const w = container.clientWidth;
    const h = container.clientHeight;
    camera.aspect = w / h;
    camera.updateProjectionMatrix();
    webglRenderer.setSize(w, h);
    cssRenderer.setSize(w, h);
  });
  
  return () => {
    cancelAnimationFrame(animId);
    controls.dispose();
    webglRenderer.dispose();
    container.removeChild(btnsDiv);
  };
}
```

---

## 73.3 CSS3DSprite - Billboard Labels

```javascript
// CSS3DSprite จะหมุนหา camera เสมอ เหมือน billboard
import { CSS3DSprite } from 'three/examples/jsm/renderers/CSS3DRenderer.js';

function createAnnotation(text, position, color = '#4ecdc4') {
  const div = document.createElement('div');
  div.style.cssText = `
    background: rgba(0,0,0,0.8);
    border: 1px solid ${color};
    border-radius: 4px;
    padding: 4px 10px;
    color: white;
    font-size: 12px;
    font-family: monospace;
    white-space: nowrap;
    pointer-events: none;
    text-shadow: 0 0 5px ${color};
  `;
  div.textContent = text;
  
  // CSS3DSprite จะหมุนหา camera อัตโนมัติ
  const sprite = new CSS3DSprite(div);
  sprite.position.copy(position);
  sprite.scale.set(0.1, 0.1, 0.1);  // Scale down สำหรับ 3D scene
  
  return sprite;
}
```

---

## 73.4 Mixing HTML iframe ใน 3D

```javascript
// src/demos/iframeIn3D.js
import * as THREE from 'three';
import { CSS3DObject } from 'three/examples/jsm/renderers/CSS3DRenderer.js';
import { CSS3DRenderer } from 'three/examples/jsm/renderers/CSS3DRenderer.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

export function createIframeDemo(container) {
  container.style.position = 'relative';
  
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#05050f');
  
  const cssScene = new THREE.Scene();
  
  const camera = new THREE.PerspectiveCamera(40, container.clientWidth / container.clientHeight, 1, 5000);
  camera.position.set(0, 0, 1500);
  
  const webglRenderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
  webglRenderer.setSize(container.clientWidth, container.clientHeight);
  webglRenderer.setClearColor(0x000000, 0);
  container.appendChild(webglRenderer.domElement);
  
  const cssRenderer = new CSS3DRenderer();
  cssRenderer.setSize(container.clientWidth, container.clientHeight);
  cssRenderer.domElement.style.position = 'absolute';
  cssRenderer.domElement.style.top = '0';
  cssRenderer.domElement.style.left = '0';
  container.appendChild(cssRenderer.domElement);
  
  const controls = new OrbitControls(camera, cssRenderer.domElement);
  controls.enableDamping = true;
  controls.rotateSpeed = 0.5;
  controls.minDistance = 500;
  controls.maxDistance = 3000;
  
  // === Embedded iframe ===
  const iframe = document.createElement('iframe');
  iframe.src = 'https://threejs.org';
  iframe.style.cssText = `
    width: 800px;
    height: 500px;
    border: none;
    border-radius: 8px;
    background: white;
  `;
  
  const iframeObj = new CSS3DObject(iframe);
  iframeObj.scale.set(0.8, 0.8, 0.8);
  iframeObj.position.set(0, 0, 0);
  cssScene.add(iframeObj);
  
  // WebGL plane to receive shadows/depth from iframe area
  const iframePlane = new THREE.Mesh(
    new THREE.PlaneGeometry(700, 430),
    new THREE.MeshBasicMaterial({ color: 0x000000, opacity: 0, transparent: true })
  );
  scene.add(iframePlane);
  
  // Decorative WebGL frame
  const frameMat = new THREE.MeshStandardMaterial({ color: '#4ecdc4', metalness: 0.8, roughness: 0.2 });
  
  const frameTop = new THREE.Mesh(new THREE.BoxGeometry(720, 10, 5), frameMat);
  frameTop.position.y = 225;
  scene.add(frameTop);
  
  const frameBottom = new THREE.Mesh(new THREE.BoxGeometry(720, 10, 5), frameMat);
  frameBottom.position.y = -225;
  scene.add(frameBottom);
  
  const frameLeft = new THREE.Mesh(new THREE.BoxGeometry(10, 450, 5), frameMat);
  frameLeft.position.x = -360;
  scene.add(frameLeft);
  
  const frameRight = new THREE.Mesh(new THREE.BoxGeometry(10, 450, 5), frameMat);
  frameRight.position.x = 360;
  scene.add(frameRight);
  
  // Corner decorations
  const cornerGeo = new THREE.BoxGeometry(20, 20, 10);
  const corners = [
    new THREE.Vector3(-360, 225, 0),
    new THREE.Vector3(360, 225, 0),
    new THREE.Vector3(-360, -225, 0),
    new THREE.Vector3(360, -225, 0),
  ];
  corners.forEach(pos => {
    const corner = new THREE.Mesh(cornerGeo, new THREE.MeshStandardMaterial({ color: '#ff6b6b', metalness: 0.9 }));
    corner.position.copy(pos);
    scene.add(corner);
  });
  
  // Ambient particles
  const particleCount = 200;
  const particlePos = new Float32Array(particleCount * 3);
  for (let i = 0; i < particleCount; i++) {
    particlePos[i * 3] = (Math.random() - 0.5) * 3000;
    particlePos[i * 3 + 1] = (Math.random() - 0.5) * 3000;
    particlePos[i * 3 + 2] = (Math.random() - 0.5) * 1000;
  }
  const particleGeo = new THREE.BufferGeometry();
  particleGeo.setAttribute('position', new THREE.BufferAttribute(particlePos, 3));
  scene.add(new THREE.Points(particleGeo, new THREE.PointsMaterial({ size: 3, color: '#4ecdc4', transparent: true, opacity: 0.4 })));
  
  // Lighting
  scene.add(new THREE.AmbientLight(0xffffff, 0.3));
  const light = new THREE.DirectionalLight(0x4ecdc4, 1);
  light.position.set(5, 5, 5);
  scene.add(light);
  
  // Animation
  let animId;
  const clock = new THREE.Clock();
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    
    controls.update();
    
    // Gentle tilt
    iframeObj.rotation.y = Math.sin(elapsed * 0.1) * 0.05;
    iframePlane.rotation.copy(iframeObj.rotation);
    
    webglRenderer.render(scene, camera);
    cssRenderer.render(cssScene, camera);
  }
  
  animate();
  
  window.addEventListener('resize', () => {
    const w = container.clientWidth;
    const h = container.clientHeight;
    camera.aspect = w / h;
    camera.updateProjectionMatrix();
    webglRenderer.setSize(w, h);
    cssRenderer.setSize(w, h);
  });
  
  return () => {
    cancelAnimationFrame(animId);
    controls.dispose();
    webglRenderer.dispose();
  };
}
```

---

## 73.5 CSS2D Label System สำหรับ 3D Map

```javascript
// src/demos/labelSystem.js
import * as THREE from 'three';
import { CSS2DRenderer, CSS2DObject } from 'three/examples/jsm/renderers/CSS2DRenderer.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

class LabelSystem {
  constructor(scene, camera, container) {
    this.scene = scene;
    this.camera = camera;
    this.labels = [];
    
    this.renderer = new CSS2DRenderer();
    this.renderer.setSize(container.clientWidth, container.clientHeight);
    this.renderer.domElement.style.cssText = `
      position: absolute;
      top: 0; left: 0;
      pointer-events: none;
    `;
    container.appendChild(this.renderer.domElement);
  }
  
  addLabel(text, position, options = {}) {
    const {
      color = '#ffffff',
      background = 'rgba(0,0,0,0.7)',
      border = 'rgba(255,255,255,0.3)',
      icon = null,
      permanent = false,
      offset = new THREE.Vector3(0, 0.5, 0)
    } = options;
    
    const div = document.createElement('div');
    div.style.cssText = `
      background: ${background};
      border: 1px solid ${border};
      border-radius: 6px;
      padding: 4px 10px;
      color: ${color};
      font-family: monospace;
      font-size: 11px;
      white-space: nowrap;
      pointer-events: none;
      transform: translateX(-50%);
    `;
    
    if (icon) {
      div.innerHTML = `${icon} ${text}`;
    } else {
      div.textContent = text;
    }
    
    // Connector line (CSS pseudo-element)
    const connector = document.createElement('div');
    connector.style.cssText = `
      position: absolute;
      top: 100%;
      left: 50%;
      transform: translateX(-50%);
      width: 1px;
      height: 20px;
      background: ${border};
    `;
    div.appendChild(connector);
    
    const label = new CSS2DObject(div);
    label.position.copy(position).add(offset);
    this.scene.add(label);
    
    const entry = { label, div, position, permanent };
    this.labels.push(entry);
    
    return label;
  }
  
  update() {
    // Dynamic visibility based on distance
    this.labels.forEach(entry => {
      if (entry.permanent) return;
      
      const distance = entry.label.position.distanceTo(this.camera.position);
      const maxDistance = 20;
      const opacity = Math.max(0, 1 - (distance - maxDistance * 0.5) / maxDistance);
      entry.div.style.opacity = opacity.toFixed(2);
    });
    
    this.renderer.render(this.scene, this.camera);
  }
  
  resize(width, height) {
    this.renderer.setSize(width, height);
  }
  
  removeAll() {
    this.labels.forEach(entry => {
      this.scene.remove(entry.label);
    });
    this.labels = [];
  }
}

export function createMapLabelDemo(container) {
  container.style.position = 'relative';
  
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#0a1520');
  scene.fog = new THREE.Fog('#0a1520', 30, 80);
  
  const camera = new THREE.PerspectiveCamera(50, container.clientWidth / container.clientHeight, 0.1, 200);
  camera.position.set(0, 15, 25);
  camera.lookAt(0, 0, 0);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.shadowMap.enabled = true;
  container.appendChild(renderer.domElement);
  
  const controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.maxPolarAngle = Math.PI / 2.2;
  controls.target.set(0, 0, 0);
  
  // Label system
  const labelSystem = new LabelSystem(scene, camera, container);
  
  // === City Map ===
  const BUILDINGS = [
    { name: 'Central Tower', pos: [0, 0], height: 8, size: 1.5, color: '#4ecdc4', type: '🏢' },
    { name: 'Tech Hub', pos: [-3, 2], height: 5, size: 1.2, color: '#ff6b6b', type: '💻' },
    { name: 'Shopping Mall', pos: [4, -1], height: 3, size: 2.5, color: '#ffd166', type: '🛍️' },
    { name: 'University', pos: [-5, -3], height: 4, size: 2, color: '#95e1d3', type: '🎓' },
    { name: 'Hospital', pos: [3, 4], height: 4, size: 1.8, color: '#c3a6ff', type: '🏥' },
    { name: 'Park', pos: [-2, -5], height: 0.5, size: 3, color: '#6bcb77', type: '🌳' },
    { name: 'Station', pos: [6, 3], height: 2, size: 1.5, color: '#a8edea', type: '🚉' },
    { name: 'Museum', pos: [-7, 1], height: 3,5, size: 1.8, color: '#ffb347', type: '🏛️' },
  ];
  
  // Ground
  const ground = new THREE.Mesh(
    new THREE.PlaneGeometry(40, 40),
    new THREE.MeshStandardMaterial({ color: '#1a2a3a', roughness: 0.9 })
  );
  ground.rotation.x = -Math.PI / 2;
  ground.receiveShadow = true;
  scene.add(ground);
  
  // Roads
  const roadMat = new THREE.MeshStandardMaterial({ color: '#2a3a4a', roughness: 1 });
  [-15, -7, 0, 7, 15].forEach(x => {
    const road = new THREE.Mesh(new THREE.PlaneGeometry(0.8, 40), roadMat);
    road.rotation.x = -Math.PI / 2;
    road.position.set(x, 0.01, 0);
    scene.add(road);
  });
  [-15, -7, 0, 7, 15].forEach(z => {
    const road = new THREE.Mesh(new THREE.PlaneGeometry(40, 0.8), roadMat);
    road.rotation.x = -Math.PI / 2;
    road.position.set(0, 0.01, z);
    scene.add(road);
  });
  
  // Buildings with labels
  BUILDINGS.forEach(building => {
    const h = building.height;
    const s = building.size;
    
    const mesh = new THREE.Mesh(
      new THREE.BoxGeometry(s, h, s),
      new THREE.MeshStandardMaterial({ color: building.color, metalness: 0.3, roughness: 0.7 })
    );
    mesh.position.set(building.pos[0], h / 2, building.pos[1]);
    mesh.castShadow = true;
    mesh.receiveShadow = true;
    scene.add(mesh);
    
    // Add label
    labelSystem.addLabel(
      `${building.type} ${building.name}`,
      new THREE.Vector3(building.pos[0], h, building.pos[1]),
      {
        color: building.color,
        background: 'rgba(5,10,20,0.9)',
        border: building.color + '88',
        offset: new THREE.Vector3(0, 1, 0)
      }
    );
    
    // Windows
    const windowMat = new THREE.MeshBasicMaterial({ color: '#ffffaa', transparent: true, opacity: 0.6 });
    for (let floor = 1; floor < Math.floor(h); floor++) {
      for (let side = 0; side < 4; side++) {
        const angle = (side / 4) * Math.PI * 2;
        const wx = new THREE.Mesh(new THREE.PlaneGeometry(0.2, 0.25), windowMat);
        wx.position.set(
          building.pos[0] + Math.sin(angle) * (s / 2 + 0.01),
          floor * 1,
          building.pos[1] + Math.cos(angle) * (s / 2 + 0.01)
        );
        wx.rotation.y = angle;
        scene.add(wx);
      }
    }
  });
  
  // Lighting
  const sun = new THREE.DirectionalLight(0xffeedd, 1.5);
  sun.position.set(10, 20, 10);
  sun.castShadow = true;
  sun.shadow.mapSize.width = 2048;
  sun.shadow.mapSize.height = 2048;
  sun.shadow.camera.near = 0.5;
  sun.shadow.camera.far = 100;
  sun.shadow.camera.left = -25;
  sun.shadow.camera.right = 25;
  sun.shadow.camera.top = 25;
  sun.shadow.camera.bottom = -25;
  scene.add(sun);
  scene.add(new THREE.AmbientLight(0x4466aa, 0.5));
  
  // Night atmosphere lights
  BUILDINGS.forEach(b => {
    const light = new THREE.PointLight(b.color, 0.5, 5);
    light.position.set(b.pos[0], b.height, b.pos[1]);
    scene.add(light);
  });
  
  // Animation
  let animId;
  const clock = new THREE.Clock();
  
  function animate() {
    animId = requestAnimationFrame(animate);
    
    controls.update();
    labelSystem.update();
    renderer.render(scene, camera);
  }
  
  animate();
  
  window.addEventListener('resize', () => {
    const w = container.clientWidth;
    const h = container.clientHeight;
    camera.aspect = w / h;
    camera.updateProjectionMatrix();
    renderer.setSize(w, h);
    labelSystem.resize(w, h);
  });
  
  return () => {
    cancelAnimationFrame(animId);
    controls.dispose();
    renderer.dispose();
    labelSystem.removeAll();
  };
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Interactive 3D Resume
สร้าง resume ใน 3D space:
- CSS3D cards สำหรับแต่ละส่วน
- Animated transitions
- Click-to-expand details
- Timeline layout

### แบบฝึกหัดที่ 2: Real-time Dashboard
สร้าง dashboard ที่:
- CSS2D labels อัพเดต real-time
- ชี้ไปยัง 3D data points
- Animated connector lines
- Filterable by category

### แบบฝึกหัดที่ 3: 3D Product Catalog
สร้าง catalog ด้วย:
- CSS3D cards ด้านหน้า
- 3D models ด้านหลัง
- Flip animation เมื่อ hover
- Shopping cart ใน 3D

### แบบฝึกหัดที่ 4: Annotated 3D Architecture
สร้าง building walkthrough:
- CSS2D annotations สำหรับ rooms
- Distance-based visibility
- Tooltip on hover
- Floor plan minimap

---

## สรุป

ในบทนี้เราเรียนรู้:
1. **CSS2DRenderer** - HTML labels ที่ follow 3D position
2. **CSS3DRenderer** - HTML elements ใน 3D perspective
3. **CSS3DSprite** - Billboard HTML elements
4. **LabelSystem** - Reusable label management
5. **iframe in 3D** - Embedding web content in 3D

บทถัดไปเราจะเรียน **3D Text & Typography**
