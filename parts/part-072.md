# Part 72: Video & Canvas Textures

## บทนำ

Three.js รองรับการใช้ video และ canvas เป็น texture บน 3D meshes ซึ่งเปิดโอกาสสร้าง interactive displays, video walls, live data visualizations, และ dynamic content ที่น่าสนใจมาก

ในบทนี้เราจะเรียนรู้:
- VideoTexture สำหรับ video บน 3D mesh
- CanvasTexture สำหรับ 2D drawing บน 3D surface
- Dynamic text และ data visualization บน mesh
- OffscreenCanvas สำหรับ performance
- CSS3DRenderer สำหรับ HTML ใน 3D

---

## 72.1 VideoTexture พื้นฐาน

```javascript
import * as THREE from 'three';

// สร้าง video element
const video = document.createElement('video');
video.src = '/video/sample.mp4';
video.loop = true;
video.muted = true;     // ต้อง muted สำหรับ autoplay
video.crossOrigin = 'anonymous';

// รอให้ video ready ก่อน play
video.addEventListener('canplay', () => {
  video.play();
});

// สร้าง VideoTexture
const videoTexture = new THREE.VideoTexture(video);
videoTexture.minFilter = THREE.LinearFilter;
videoTexture.magFilter = THREE.LinearFilter;
videoTexture.format = THREE.RGBAFormat;
videoTexture.colorSpace = THREE.SRGBColorSpace;

// ใช้กับ material
const material = new THREE.MeshBasicMaterial({ map: videoTexture });
const screen = new THREE.Mesh(new THREE.PlaneGeometry(16, 9), material);
scene.add(screen);

// ต้อง update texture ทุก frame ถ้าใช้ needsUpdate
// หรือ VideoTexture จะ auto-update ถ้าเปิด video.readyState > 0
function animate() {
  requestAnimationFrame(animate);
  
  // VideoTexture auto updates เมื่อ video กำลังเล่น
  // แต่บาง browser ต้องการ manual update
  if (video.readyState === video.HAVE_ENOUGH_DATA) {
    videoTexture.needsUpdate = true;
  }
  
  renderer.render(scene, camera);
}
```

---

## 72.2 Video Player บน 3D Wall

```javascript
// src/demos/videoWall.js
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

export function createVideoWallDemo(container) {
  // Setup
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#0a0a1a');
  
  const camera = new THREE.PerspectiveCamera(60, container.clientWidth / container.clientHeight, 0.1, 100);
  camera.position.set(0, 0, 10);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.shadowMap.enabled = true;
  container.appendChild(renderer.domElement);
  
  const controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.minDistance = 3;
  controls.maxDistance = 20;
  
  // === Video Setup ===
  const video = document.createElement('video');
  video.crossOrigin = 'anonymous';
  video.loop = true;
  video.muted = true;
  video.playsInline = true;
  video.preload = 'auto';
  
  // ใช้ sample video URL
  video.src = 'https://commondatastorage.googleapis.com/gtv-videos-bucket/sample/BigBuckBunny.mp4';
  
  const videoTexture = new THREE.VideoTexture(video);
  videoTexture.colorSpace = THREE.SRGBColorSpace;
  
  // === TV Frame ===
  // Outer frame
  const tvFrame = new THREE.Mesh(
    new THREE.BoxGeometry(9.2, 5.4, 0.3),
    new THREE.MeshStandardMaterial({ color: '#1a1a1a', metalness: 0.7, roughness: 0.3 })
  );
  tvFrame.castShadow = true;
  scene.add(tvFrame);
  
  // Screen bezel
  const bezel = new THREE.Mesh(
    new THREE.BoxGeometry(8.8, 5.1, 0.05),
    new THREE.MeshStandardMaterial({ color: '#0a0a0a', metalness: 0.3, roughness: 0.5 })
  );
  bezel.position.z = 0.1;
  scene.add(bezel);
  
  // Screen (video texture)
  const screenMesh = new THREE.Mesh(
    new THREE.PlaneGeometry(8.5, 4.8),
    new THREE.MeshBasicMaterial({ map: videoTexture })
  );
  screenMesh.position.z = 0.13;
  scene.add(screenMesh);
  
  // Power LED
  const led = new THREE.Mesh(
    new THREE.SphereGeometry(0.05, 8, 8),
    new THREE.MeshBasicMaterial({ color: '#00ff00' })
  );
  led.position.set(4.3, -2.4, 0.15);
  scene.add(led);
  
  // Stand
  const stand = new THREE.Group();
  
  const neck = new THREE.Mesh(
    new THREE.BoxGeometry(0.3, 1.5, 0.3),
    new THREE.MeshStandardMaterial({ color: '#333', metalness: 0.8, roughness: 0.2 })
  );
  neck.position.y = -3.2;
  stand.add(neck);
  
  const base = new THREE.Mesh(
    new THREE.BoxGeometry(3, 0.2, 1.5),
    new THREE.MeshStandardMaterial({ color: '#333', metalness: 0.8, roughness: 0.2 })
  );
  base.position.y = -4;
  stand.add(base);
  
  scene.add(stand);
  
  // === Reflection on Floor ===
  const floor = new THREE.Mesh(
    new THREE.PlaneGeometry(30, 30),
    new THREE.MeshStandardMaterial({ color: '#111', roughness: 0.7, metalness: 0.3 })
  );
  floor.rotation.x = -Math.PI / 2;
  floor.position.y = -4.2;
  floor.receiveShadow = true;
  scene.add(floor);
  
  // === Room Walls ===
  const wallMat = new THREE.MeshStandardMaterial({ color: '#2c2c2c', roughness: 0.9 });
  
  // Back wall
  const backWall = new THREE.Mesh(new THREE.PlaneGeometry(30, 15), wallMat);
  backWall.position.z = -8;
  backWall.position.y = 3;
  scene.add(backWall);
  
  // === Lighting ===
  scene.add(new THREE.AmbientLight(0xffffff, 0.3));
  
  const keyLight = new THREE.DirectionalLight(0xffffff, 1);
  keyLight.position.set(5, 10, 5);
  keyLight.castShadow = true;
  scene.add(keyLight);
  
  // Screen glow light
  const screenLight = new THREE.PointLight('#4488ff', 2, 10);
  screenLight.position.set(0, 0, 1);
  scene.add(screenLight);
  
  // === Controls UI ===
  const controlsDiv = document.createElement('div');
  controlsDiv.style.cssText = `
    position: absolute;
    bottom: 20px;
    left: 50%;
    transform: translateX(-50%);
    display: flex;
    gap: 10px;
    background: rgba(0,0,0,0.7);
    padding: 12px 20px;
    border-radius: 30px;
    z-index: 10;
  `;
  
  const buttons = [
    { label: '▶ Play', action: () => video.play() },
    { label: '⏸ Pause', action: () => video.pause() },
    { label: '🔊 Unmute', action: () => { video.muted = false; } },
    { label: '🔇 Mute', action: () => { video.muted = true; } },
  ];
  
  buttons.forEach(btn => {
    const button = document.createElement('button');
    button.textContent = btn.label;
    button.style.cssText = `
      background: rgba(78, 205, 196, 0.2);
      border: 1px solid #4ecdc4;
      color: white;
      padding: 8px 15px;
      border-radius: 20px;
      cursor: pointer;
      font-size: 0.85em;
    `;
    button.addEventListener('click', btn.action);
    controlsDiv.appendChild(button);
  });
  
  container.style.position = 'relative';
  container.appendChild(controlsDiv);
  
  // Auto-play when ready
  video.addEventListener('canplay', () => {
    video.play().catch(() => {
      console.log('Auto-play blocked by browser');
    });
  });
  
  // === Animation ===
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    
    controls.update();
    
    // Screen glow pulse
    screenLight.intensity = 1.5 + Math.sin(elapsed * 2) * 0.3;
    
    // LED blink
    if (video.paused) {
      led.material.color.setHex(Math.sin(elapsed * 5) > 0 ? 0xff6600 : 0x000000);
    } else {
      led.material.color.setHex(0x00ff00);
    }
    
    renderer.render(scene, camera);
  }
  
  animate();
  
  window.addEventListener('resize', () => {
    const w = container.clientWidth;
    const h = container.clientHeight;
    camera.aspect = w / h;
    camera.updateProjectionMatrix();
    renderer.setSize(w, h);
  });
  
  return () => {
    cancelAnimationFrame(animId);
    video.pause();
    videoTexture.dispose();
    renderer.dispose();
    controls.dispose();
    container.removeChild(controlsDiv);
  };
}
```

---

## 72.3 CanvasTexture - Live Clock บน Sphere

```javascript
// src/demos/canvasTexture.js
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

export function createClockSphereDemo(container) {
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#0a0a2a');
  
  const camera = new THREE.PerspectiveCamera(60, container.clientWidth / container.clientHeight, 0.1, 100);
  camera.position.set(0, 0, 5);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  container.appendChild(renderer.domElement);
  
  const controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.autoRotate = true;
  controls.autoRotateSpeed = 0.5;
  
  // === Canvas Setup ===
  const CANVAS_SIZE = 1024;
  const canvas = document.createElement('canvas');
  canvas.width = CANVAS_SIZE;
  canvas.height = CANVAS_SIZE;
  const ctx = canvas.getContext('2d');
  
  // === Draw Clock Function ===
  function drawClock(ctx, size) {
    const cx = size / 2;
    const cy = size / 2;
    const r = size * 0.45;
    
    const now = new Date();
    const hours = now.getHours() % 12;
    const minutes = now.getMinutes();
    const seconds = now.getSeconds();
    const ms = now.getMilliseconds();
    
    // Clear
    ctx.clearRect(0, 0, size, size);
    
    // Background
    const bgGrad = ctx.createRadialGradient(cx, cy, 0, cx, cy, r);
    bgGrad.addColorStop(0, '#1a1a3a');
    bgGrad.addColorStop(1, '#0a0a1a');
    ctx.fillStyle = bgGrad;
    ctx.beginPath();
    ctx.arc(cx, cy, r + 20, 0, Math.PI * 2);
    ctx.fill();
    
    // Outer ring
    ctx.strokeStyle = '#4ecdc4';
    ctx.lineWidth = 8;
    ctx.beginPath();
    ctx.arc(cx, cy, r, 0, Math.PI * 2);
    ctx.stroke();
    
    // Hour marks
    for (let i = 0; i < 12; i++) {
      const angle = (i / 12) * Math.PI * 2 - Math.PI / 2;
      const x1 = cx + Math.cos(angle) * (r - 15);
      const y1 = cy + Math.sin(angle) * (r - 15);
      const x2 = cx + Math.cos(angle) * (r - 35);
      const y2 = cy + Math.sin(angle) * (r - 35);
      
      ctx.strokeStyle = i % 3 === 0 ? '#4ecdc4' : 'rgba(78,205,196,0.5)';
      ctx.lineWidth = i % 3 === 0 ? 6 : 3;
      ctx.beginPath();
      ctx.moveTo(x1, y1);
      ctx.lineTo(x2, y2);
      ctx.stroke();
    }
    
    // Numbers
    ctx.fillStyle = '#ffffff';
    ctx.font = `bold ${size * 0.05}px monospace`;
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    
    for (let i = 1; i <= 12; i++) {
      const angle = (i / 12) * Math.PI * 2 - Math.PI / 2;
      const x = cx + Math.cos(angle) * (r - 65);
      const y = cy + Math.sin(angle) * (r - 65);
      ctx.fillText(i.toString(), x, y);
    }
    
    // Hour hand
    const hourAngle = ((hours + minutes / 60 + seconds / 3600) / 12) * Math.PI * 2 - Math.PI / 2;
    drawHand(ctx, cx, cy, hourAngle, r * 0.55, 10, '#ffffff');
    
    // Minute hand
    const minuteAngle = ((minutes + seconds / 60) / 60) * Math.PI * 2 - Math.PI / 2;
    drawHand(ctx, cx, cy, minuteAngle, r * 0.78, 6, '#4ecdc4');
    
    // Second hand (smooth)
    const secondAngle = ((seconds + ms / 1000) / 60) * Math.PI * 2 - Math.PI / 2;
    drawHand(ctx, cx, cy, secondAngle, r * 0.88, 3, '#ff6b6b');
    
    // Center dot
    ctx.fillStyle = '#4ecdc4';
    ctx.beginPath();
    ctx.arc(cx, cy, 12, 0, Math.PI * 2);
    ctx.fill();
    
    ctx.fillStyle = '#ffffff';
    ctx.beginPath();
    ctx.arc(cx, cy, 6, 0, Math.PI * 2);
    ctx.fill();
    
    // Digital time
    const timeStr = `${now.getHours().toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
    ctx.fillStyle = 'rgba(78,205,196,0.8)';
    ctx.font = `${size * 0.04}px monospace`;
    ctx.fillText(timeStr, cx, cy + r * 0.35);
    
    // Date
    const dateStr = now.toLocaleDateString('en-US', { weekday: 'short', month: 'short', day: 'numeric' });
    ctx.fillStyle = 'rgba(255,255,255,0.5)';
    ctx.font = `${size * 0.03}px sans-serif`;
    ctx.fillText(dateStr, cx, cy + r * 0.45);
  }
  
  function drawHand(ctx, cx, cy, angle, length, width, color) {
    ctx.strokeStyle = color;
    ctx.lineWidth = width;
    ctx.lineCap = 'round';
    ctx.beginPath();
    ctx.moveTo(
      cx - Math.cos(angle) * length * 0.2,
      cy - Math.sin(angle) * length * 0.2
    );
    ctx.lineTo(
      cx + Math.cos(angle) * length,
      cy + Math.sin(angle) * length
    );
    ctx.stroke();
  }
  
  // === CanvasTexture ===
  const canvasTexture = new THREE.CanvasTexture(canvas);
  canvasTexture.colorSpace = THREE.SRGBColorSpace;
  
  // === Sphere ===
  const sphere = new THREE.Mesh(
    new THREE.SphereGeometry(2, 64, 64),
    new THREE.MeshStandardMaterial({
      map: canvasTexture,
      metalness: 0.1,
      roughness: 0.8,
    })
  );
  scene.add(sphere);
  
  // Glow rings
  for (let i = 1; i <= 3; i++) {
    const ring = new THREE.Mesh(
      new THREE.TorusGeometry(2.1 + i * 0.1, 0.02, 8, 64),
      new THREE.MeshBasicMaterial({ 
        color: '#4ecdc4', 
        transparent: true, 
        opacity: 0.3 / i 
      })
    );
    ring.rotation.x = Math.PI / 2 * (i % 2 === 0 ? 1 : 0);
    scene.add(ring);
  }
  
  // Lighting
  scene.add(new THREE.AmbientLight(0xffffff, 0.5));
  const light = new THREE.DirectionalLight(0xffffff, 1);
  light.position.set(5, 10, 5);
  scene.add(light);
  
  // Stars
  const starCount = 2000;
  const starPositions = new Float32Array(starCount * 3);
  for (let i = 0; i < starCount; i++) {
    starPositions[i * 3] = (Math.random() - 0.5) * 100;
    starPositions[i * 3 + 1] = (Math.random() - 0.5) * 100;
    starPositions[i * 3 + 2] = (Math.random() - 0.5) * 100;
  }
  const starGeo = new THREE.BufferGeometry();
  starGeo.setAttribute('position', new THREE.BufferAttribute(starPositions, 3));
  scene.add(new THREE.Points(starGeo, new THREE.PointsMaterial({ size: 0.05, color: 'white' })));
  
  // === Animation ===
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    
    // Redraw clock canvas ทุก frame
    drawClock(ctx, CANVAS_SIZE);
    canvasTexture.needsUpdate = true;
    
    controls.update();
    renderer.render(scene, camera);
  }
  
  animate();
  
  window.addEventListener('resize', () => {
    const w = container.clientWidth;
    const h = container.clientHeight;
    camera.aspect = w / h;
    camera.updateProjectionMatrix();
    renderer.setSize(w, h);
  });
  
  return () => {
    cancelAnimationFrame(animId);
    canvasTexture.dispose();
    renderer.dispose();
    controls.dispose();
  };
}
```

---

## 72.4 Live Data Visualization บน Mesh

```javascript
// src/demos/dataVisualization.js
import * as THREE from 'three';

// สร้าง canvas ที่แสดง real-time data
function createDataCanvas(width, height) {
  const canvas = document.createElement('canvas');
  canvas.width = width;
  canvas.height = height;
  const ctx = canvas.getContext('2d');
  
  const history = new Array(width).fill(0);
  let dataIndex = 0;
  
  return {
    canvas,
    ctx,
    update(value) {
      history[dataIndex % width] = value;
      dataIndex++;
      
      // Clear
      ctx.fillStyle = '#0a0a1a';
      ctx.fillRect(0, 0, width, height);
      
      // Grid
      ctx.strokeStyle = 'rgba(78,205,196,0.1)';
      ctx.lineWidth = 1;
      for (let y = 0; y < height; y += height / 8) {
        ctx.beginPath();
        ctx.moveTo(0, y);
        ctx.lineTo(width, y);
        ctx.stroke();
      }
      
      // Data line
      ctx.strokeStyle = '#4ecdc4';
      ctx.lineWidth = 2;
      ctx.beginPath();
      
      for (let x = 0; x < width; x++) {
        const idx = (dataIndex + x) % width;
        const val = history[idx] || 0;
        const y = height - (val / 100) * height * 0.8 - height * 0.1;
        
        if (x === 0) ctx.moveTo(x, y);
        else ctx.lineTo(x, y);
      }
      ctx.stroke();
      
      // Gradient fill under line
      const gradient = ctx.createLinearGradient(0, 0, 0, height);
      gradient.addColorStop(0, 'rgba(78,205,196,0.4)');
      gradient.addColorStop(1, 'rgba(78,205,196,0)');
      ctx.fillStyle = gradient;
      ctx.lineTo(width, height);
      ctx.lineTo(0, height);
      ctx.closePath();
      ctx.fill();
      
      // Current value label
      ctx.fillStyle = '#ffffff';
      ctx.font = 'bold 16px monospace';
      ctx.textAlign = 'right';
      ctx.fillText(`${value.toFixed(1)}%`, width - 10, 25);
      
      // Title
      ctx.fillStyle = '#4ecdc4';
      ctx.textAlign = 'left';
      ctx.font = '14px monospace';
      ctx.fillText('CPU USAGE', 10, 20);
    }
  };
}

export function createDataVizDemo(container) {
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#050510');
  
  const camera = new THREE.PerspectiveCamera(60, container.clientWidth / container.clientHeight, 0.1, 100);
  camera.position.set(0, 0, 8);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  container.appendChild(renderer.domElement);
  
  // === Dashboard Panels ===
  const panels = [];
  
  const panelConfigs = [
    { title: 'CPU', width: 4, height: 2, position: [-3, 1.5, 0] },
    { title: 'Memory', width: 4, height: 2, position: [3, 1.5, 0] },
    { title: 'Network', width: 8.5, height: 2, position: [0, -1, 0] },
  ];
  
  panelConfigs.forEach(config => {
    const CANVAS_W = 512;
    const CANVAS_H = 256;
    
    const { canvas, ctx, update } = createDataCanvas(CANVAS_W, CANVAS_H);
    const texture = new THREE.CanvasTexture(canvas);
    
    // Panel frame
    const frame = new THREE.Mesh(
      new THREE.BoxGeometry(config.width + 0.2, config.height + 0.2, 0.05),
      new THREE.MeshStandardMaterial({ color: '#1a1a3a', metalness: 0.5, roughness: 0.5 })
    );
    frame.position.set(...config.position);
    scene.add(frame);
    
    // Panel screen
    const screen = new THREE.Mesh(
      new THREE.PlaneGeometry(config.width, config.height),
      new THREE.MeshBasicMaterial({ map: texture })
    );
    screen.position.set(config.position[0], config.position[1], 0.03);
    scene.add(screen);
    
    panels.push({ update, texture });
  });
  
  // === Lighting ===
  scene.add(new THREE.AmbientLight(0xffffff, 0.3));
  const light = new THREE.DirectionalLight(0xffffff, 0.8);
  light.position.set(5, 5, 5);
  scene.add(light);
  
  // Blue ambient from screen
  const screenLight = new THREE.PointLight('#0044ff', 1, 15);
  screenLight.position.set(0, 0, 1);
  scene.add(screenLight);
  
  // === Simulate Data ===
  let time = 0;
  const cpuHistory = [];
  const memHistory = [];
  const netHistory = [];
  
  // === Animation ===
  const clock = new THREE.Clock();
  let animId;
  let updateCounter = 0;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const delta = clock.getDelta();
    time += delta;
    updateCounter += delta;
    
    // Update data every 100ms
    if (updateCounter > 0.1) {
      updateCounter = 0;
      
      // Simulate CPU usage
      const cpuValue = 30 + Math.sin(time * 0.5) * 20 + Math.random() * 15;
      panels[0].update(cpuValue);
      panels[0].texture.needsUpdate = true;
      
      // Simulate Memory usage
      const memValue = 60 + Math.sin(time * 0.3) * 10 + Math.random() * 5;
      panels[1].update(memValue);
      panels[1].texture.needsUpdate = true;
      
      // Simulate Network
      const netValue = Math.abs(Math.sin(time * 2)) * 80 + Math.random() * 20;
      panels[2].update(netValue);
      panels[2].texture.needsUpdate = true;
    }
    
    // Gentle camera sway
    camera.position.x = Math.sin(time * 0.1) * 0.5;
    camera.position.y = Math.cos(time * 0.15) * 0.2;
    camera.lookAt(0, 0, 0);
    
    renderer.render(scene, camera);
  }
  
  animate();
  
  window.addEventListener('resize', () => {
    const w = container.clientWidth;
    const h = container.clientHeight;
    camera.aspect = w / h;
    camera.updateProjectionMatrix();
    renderer.setSize(w, h);
  });
  
  return () => {
    cancelAnimationFrame(animId);
    panels.forEach(p => p.texture.dispose());
    renderer.dispose();
  };
}
```

---

## 72.5 CSS3DRenderer

CSS3DRenderer ให้ render HTML elements ใน 3D space

```javascript
// src/demos/css3d.js
import * as THREE from 'three';
import { CSS3DRenderer, CSS3DObject } from 'three/examples/jsm/renderers/CSS3DRenderer.js';
import { TrackballControls } from 'three/examples/jsm/controls/TrackballControls.js';

export function createCSS3DDemo(container) {
  container.style.position = 'relative';
  
  // === 3D Scene (WebGL) ===
  const scene = new THREE.Scene();
  const camera = new THREE.PerspectiveCamera(40, container.clientWidth / container.clientHeight, 1, 10000);
  camera.position.z = 3000;
  
  const webglRenderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
  webglRenderer.setSize(container.clientWidth, container.clientHeight);
  webglRenderer.setClearColor(0x000000, 0);
  container.appendChild(webglRenderer.domElement);
  
  // === CSS3D Scene ===
  const cssScene = new THREE.Scene();
  
  const cssRenderer = new CSS3DRenderer();
  cssRenderer.setSize(container.clientWidth, container.clientHeight);
  cssRenderer.domElement.style.cssText = `
    position: absolute;
    top: 0;
    left: 0;
    pointer-events: none;
  `;
  container.appendChild(cssRenderer.domElement);
  
  // === Controls (on CSS renderer for pointer events) ===
  cssRenderer.domElement.style.pointerEvents = 'auto';
  const controls = new TrackballControls(camera, cssRenderer.domElement);
  controls.rotateSpeed = 0.5;
  controls.minDistance = 500;
  controls.maxDistance = 5000;
  
  // === Create HTML Cards as CSS3DObjects ===
  const CARD_DATA = [
    { title: 'React', tech: 'Frontend', color: '#61dafb', code: 'JSX + Hooks' },
    { title: 'Three.js', tech: '3D', color: '#049EF4', code: 'WebGL API' },
    { title: 'Node.js', tech: 'Backend', color: '#8CC84B', code: 'V8 Engine' },
    { title: 'TypeScript', tech: 'Language', color: '#3178C6', code: 'Type Safety' },
    { title: 'PostgreSQL', tech: 'Database', color: '#336791', code: 'SQL + JSON' },
    { title: 'Docker', tech: 'DevOps', color: '#2496ED', code: 'Containers' },
    { title: 'GraphQL', tech: 'API', color: '#E535AB', code: 'Query Lang' },
    { title: 'Python', tech: 'ML/Data', color: '#FFD43B', code: 'Batteries Inc' },
    { title: 'Redis', tech: 'Cache', color: '#DC382D', code: 'In-Memory' },
  ];
  
  const objects = [];
  const targets = { sphere: [], helix: [], grid: [] };
  
  CARD_DATA.forEach((data, index) => {
    // Create HTML element
    const element = document.createElement('div');
    element.style.cssText = `
      width: 120px;
      height: 160px;
      background: linear-gradient(135deg, ${data.color}22, ${data.color}44);
      border: 1px solid ${data.color}88;
      border-radius: 12px;
      padding: 16px;
      color: white;
      font-family: monospace;
      cursor: pointer;
      transition: transform 0.2s;
      backdrop-filter: blur(10px);
    `;
    element.innerHTML = `
      <div style="font-size: 1.3em; font-weight: bold; color: ${data.color}; margin-bottom: 8px">${data.title}</div>
      <div style="font-size: 0.75em; color: #aaa; margin-bottom: 12px">${data.tech}</div>
      <div style="font-size: 0.7em; background: rgba(0,0,0,0.3); padding: 6px 8px; border-radius: 6px; color: #ddd">${data.code}</div>
    `;
    
    element.addEventListener('mouseenter', () => {
      element.style.transform = 'scale(1.05)';
    });
    element.addEventListener('mouseleave', () => {
      element.style.transform = 'scale(1)';
    });
    
    // Create CSS3DObject
    const object = new CSS3DObject(element);
    object.position.set(Math.random() * 4000 - 2000, Math.random() * 4000 - 2000, Math.random() * 4000 - 2000);
    cssScene.add(object);
    objects.push(object);
    
    // Sphere target positions
    const phi = Math.acos(-1 + (2 * index) / CARD_DATA.length);
    const theta = Math.sqrt(CARD_DATA.length * Math.PI) * phi;
    const sphereTarget = new THREE.Object3D();
    sphereTarget.position.setFromSphericalCoords(800, phi, theta);
    targets.sphere.push(sphereTarget);
    
    // Grid target positions
    const gridTarget = new THREE.Object3D();
    gridTarget.position.set(
      (index % 3) * 300 - 300,
      -(Math.floor(index / 3)) * 300 + 300,
      0
    );
    targets.grid.push(gridTarget);
  });
  
  // === Layout buttons ===
  const layoutDiv = document.createElement('div');
  layoutDiv.style.cssText = `
    position: absolute;
    bottom: 30px;
    left: 50%;
    transform: translateX(-50%);
    display: flex;
    gap: 10px;
    z-index: 20;
  `;
  
  function transformToTarget(targetArr) {
    targetArr.forEach((target, i) => {
      if (!objects[i]) return;
      objects[i].position.copy(target.position);
      objects[i].rotation.copy(target.rotation);
    });
  }
  
  ['Sphere', 'Grid'].forEach(layout => {
    const btn = document.createElement('button');
    btn.textContent = layout;
    btn.style.cssText = `
      padding: 10px 20px;
      background: rgba(78,205,196,0.2);
      border: 1px solid #4ecdc4;
      color: white;
      border-radius: 8px;
      cursor: pointer;
      font-family: monospace;
    `;
    btn.addEventListener('click', () => {
      transformToTarget(targets[layout.toLowerCase()]);
    });
    layoutDiv.appendChild(btn);
  });
  
  container.appendChild(layoutDiv);
  
  // Initialize to sphere layout
  transformToTarget(targets.sphere);
  
  // === WebGL Background ===
  const starCount = 5000;
  const starPositions = new Float32Array(starCount * 3);
  for (let i = 0; i < starCount; i++) {
    starPositions[i * 3] = (Math.random() - 0.5) * 10000;
    starPositions[i * 3 + 1] = (Math.random() - 0.5) * 10000;
    starPositions[i * 3 + 2] = (Math.random() - 0.5) * 10000;
  }
  const starGeo = new THREE.BufferGeometry();
  starGeo.setAttribute('position', new THREE.BufferAttribute(starPositions, 3));
  scene.add(new THREE.Points(starGeo, new THREE.PointsMaterial({ size: 3, color: 'white', transparent: true, opacity: 0.5 })));
  
  // === Animation ===
  let animId;
  const clock = new THREE.Clock();
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    
    controls.update();
    
    // Gentle floating of individual cards
    objects.forEach((obj, i) => {
      obj.position.y += Math.sin(elapsed * 0.5 + i * 0.3) * 0.2;
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
    container.removeChild(layoutDiv);
  };
}
```

---

## 72.6 OffscreenCanvas สำหรับ Performance

```javascript
// src/demos/offscreenCanvas.js

// Worker thread สำหรับ heavy canvas operations
const workerCode = `
  let canvas;
  let ctx;
  let frame = 0;
  
  self.onmessage = function(e) {
    if (e.data.type === 'init') {
      canvas = e.data.canvas;
      ctx = canvas.getContext('2d');
      draw();
    }
  };
  
  function draw() {
    const w = canvas.width;
    const h = canvas.height;
    const t = frame / 60;
    frame++;
    
    // Clear
    ctx.clearRect(0, 0, w, h);
    
    // Background
    ctx.fillStyle = '#0a0a1a';
    ctx.fillRect(0, 0, w, h);
    
    // Draw complex mandelbrot set
    const maxIter = 100;
    const zoom = 1 + Math.sin(t * 0.1) * 0.5;
    const centerX = -0.5 + Math.sin(t * 0.05) * 0.3;
    const centerY = Math.sin(t * 0.07) * 0.3;
    
    for (let py = 0; py < h; py += 2) {
      for (let px = 0; px < w; px += 2) {
        const x0 = (px / w - 0.5) * 3.5 / zoom + centerX;
        const y0 = (py / h - 0.5) * 2 / zoom + centerY;
        
        let x = 0, y = 0, iter = 0;
        
        while (x*x + y*y <= 4 && iter < maxIter) {
          const xtemp = x*x - y*y + x0;
          y = 2*x*y + y0;
          x = xtemp;
          iter++;
        }
        
        if (iter < maxIter) {
          const hue = (iter / maxIter) * 360;
          ctx.fillStyle = 'hsl(' + hue + ', 100%, 50%)';
          ctx.fillRect(px, py, 2, 2);
        }
      }
    }
    
    setTimeout(() => self.requestAnimationFrame(draw), 33);
  }
`;

export async function createOffscreenDemo(container) {
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#000');
  
  const camera = new THREE.PerspectiveCamera(60, container.clientWidth / container.clientHeight, 0.1, 100);
  camera.position.z = 5;
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  container.appendChild(renderer.domElement);
  
  // Check OffscreenCanvas support
  if (typeof OffscreenCanvas !== 'undefined') {
    const offscreen = new OffscreenCanvas(512, 512);
    const blob = new Blob([workerCode], { type: 'application/javascript' });
    const worker = new Worker(URL.createObjectURL(blob));
    
    // Transfer canvas to worker
    worker.postMessage({ type: 'init', canvas: offscreen }, [offscreen]);
    
    // Note: OffscreenCanvas ไม่สามารถ transfer กลับมาเป็น texture ได้โดยตรงใน Three.js
    // ต้องใช้ ImageBitmap
  }
  
  // Fallback: ใช้ regular canvas
  const canvas = document.createElement('canvas');
  canvas.width = 512;
  canvas.height = 512;
  const ctx = canvas.getContext('2d');
  
  const texture = new THREE.CanvasTexture(canvas);
  
  const mesh = new THREE.Mesh(
    new THREE.SphereGeometry(2, 64, 64),
    new THREE.MeshBasicMaterial({ map: texture })
  );
  scene.add(mesh);
  
  // Draw function
  let frame = 0;
  function drawFrame() {
    frame++;
    const t = frame / 60;
    const SIZE = 512;
    
    ctx.fillStyle = '#0a0a1a';
    ctx.fillRect(0, 0, SIZE, SIZE);
    
    // Simplified pattern
    for (let i = 0; i < 20; i++) {
      const x = (Math.sin(t + i) * 0.5 + 0.5) * SIZE;
      const y = (Math.cos(t * 1.3 + i) * 0.5 + 0.5) * SIZE;
      const r = 30 + Math.sin(t * 2 + i) * 20;
      const hue = (i / 20) * 360 + t * 30;
      
      ctx.fillStyle = `hsla(${hue}, 100%, 60%, 0.3)`;
      ctx.beginPath();
      ctx.arc(x, y, r, 0, Math.PI * 2);
      ctx.fill();
    }
    
    texture.needsUpdate = true;
  }
  
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    
    drawFrame();
    
    mesh.rotation.y = clock.getElapsedTime() * 0.2;
    mesh.rotation.x = Math.sin(clock.getElapsedTime() * 0.1) * 0.2;
    
    renderer.render(scene, camera);
  }
  
  animate();
  
  return () => {
    cancelAnimationFrame(animId);
    texture.dispose();
    renderer.dispose();
  };
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Multi-Screen Video Wall
สร้าง video wall ที่มี:
- 9 screens (3x3 grid)
- แต่ละ screen เล่น video segments ต่างกัน
- Sync หรือ offset timing
- Interactive: คลิกเพื่อ fullscreen

### แบบฝึกหัดที่ 2: Live Dashboard Sphere
สร้าง sphere ที่แสดง real-time metrics:
- อุณหภูมิ, pressure, humidity (simulated)
- Charts หลายประเภทบน sphere
- Auto-rotate เพื่อดูข้อมูลทั้งหมด

### แบบฝึกหัดที่ 3: CSS3D Portfolio
สร้าง 3D portfolio ด้วย CSS3D:
- Project cards ใน 3D space
- Layout transitions (sphere/grid/helix)
- Interactive details panel
- Responsive design

### แบบฝึกหัดที่ 4: Video Kaleidoscope
สร้าง video effect:
- วิดีโอ mapped ด้วย custom UV
- Mirror/kaleidoscope effect บน sphere
- Real-time color grading ด้วย GLSL
- User controls

### แบบฝึกหัดที่ 5: Dynamic Texture Atlas
สร้าง system ที่:
- อัพเดต texture atlas แบบ dynamic
- แต่ละ tile แสดงข้อมูลต่างกัน
- Smooth transitions ระหว่าง states
- Performance optimized

---

## สรุป

ในบทนี้เราเรียนรู้:
1. **VideoTexture** - Video บน 3D mesh
2. **CanvasTexture** - 2D drawing บน 3D surface
3. **Live Data Viz** - Real-time charts บน mesh
4. **CSS3DRenderer** - HTML elements ใน 3D space
5. **OffscreenCanvas** - Performance optimization

บทถัดไปเราจะเรียน **CSS3DRenderer & CSS2DRenderer แบบเจาะลึก**
