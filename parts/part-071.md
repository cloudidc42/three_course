# Part 71: Audio ใน Three.js

## บทนำ

Three.js มี audio system ที่สร้างบนบน Web Audio API ทำให้เราสามารถสร้าง spatial audio (เสียง 3D), audio-reactive animations, และ music visualizations ได้ในฉาก 3D

ระบบ audio ของ Three.js ประกอบด้วย:
- **AudioListener** - "หู" ที่ติดกับ camera
- **Audio** - เสียงที่ไม่มีตำแหน่ง (เสียง background music)
- **PositionalAudio** - เสียง 3D ที่มีตำแหน่งในฉาก
- **AudioLoader** - โหลด audio files
- **AudioAnalyser** - วิเคราะห์ frequency data

---

## 71.1 AudioListener และ Audio พื้นฐาน

```javascript
import * as THREE from 'three';

// สร้าง AudioListener และ attach กับ camera
const listener = new THREE.AudioListener();
camera.add(listener);

// สร้าง Audio object (non-positional)
const sound = new THREE.Audio(listener);

// โหลด audio
const audioLoader = new THREE.AudioLoader();
audioLoader.load('/audio/ambient.mp3', (buffer) => {
  sound.setBuffer(buffer);
  sound.setLoop(true);
  sound.setVolume(0.5);
  sound.play();
});

// Controls
sound.play();
sound.pause();
sound.stop();

// Properties
sound.setVolume(0.8);            // 0.0 - 1.0
sound.setLoop(true);             // loop
sound.setPlaybackRate(1.5);      // เร็วขึ้น 1.5x
sound.setDetune(100);            // ปรับ pitch เป็น cents
sound.setLoopStart(0);           // start point สำหรับ loop
sound.setLoopEnd(10);            // end point สำหรับ loop

// State
sound.isPlaying;                 // กำลังเล่นอยู่
sound.duration;                  // ความยาว (วินาที)
```

---

## 71.2 PositionalAudio (3D Audio)

```javascript
// สร้าง PositionalAudio ที่มีตำแหน่งใน 3D space
const positionalAudio = new THREE.PositionalAudio(listener);

// โหลด audio
audioLoader.load('/audio/fire.mp3', (buffer) => {
  positionalAudio.setBuffer(buffer);
  positionalAudio.setRefDistance(1);        // ระยะที่เสียงเต็ม
  positionalAudio.setMaxDistance(100);      // ระยะสูงสุดที่ได้ยิน
  positionalAudio.setRolloffFactor(1);      // อัตราการลดเสียงตามระยะ
  positionalAudio.setDistanceModel('inverse');  // 'linear' | 'inverse' | 'exponential'
  positionalAudio.setDirectionalCone(180, 230, 0.1);  // inner, outer, outer gain
  positionalAudio.setLoop(true);
  positionalAudio.setVolume(1.0);
});

// Attach กับ 3D object
const fireMesh = new THREE.Mesh(
  new THREE.SphereGeometry(0.3),
  new THREE.MeshBasicMaterial({ color: 'orange' })
);
fireMesh.position.set(5, 0, 0);
fireMesh.add(positionalAudio);  // audio ติดกับ mesh
scene.add(fireMesh);
```

---

## 71.3 AudioAnalyser สำหรับ Visualizations

```javascript
// สร้าง analyser สำหรับ FFT analysis
const analyser = new THREE.AudioAnalyser(sound, 2048);  // FFT size

// ใน render loop
function animate() {
  requestAnimationFrame(animate);
  
  // ได้รับ frequency data
  const data = analyser.getFrequencyData();  // Uint8Array (0-255)
  
  // Average frequency
  const average = analyser.getAverageFrequency();
  
  // ใช้ data ใน visualization
  console.log('Average frequency:', average);
  console.log('Bass:', data[0], data[1], data[2]);
  
  renderer.render(scene, camera);
}
```

---

## 71.4 Spatial Audio Demo

```javascript
// src/demos/spatialAudio.js
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

export function createSpatialAudioDemo(container) {
  // === Setup ===
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#1a1a2e');
  
  const camera = new THREE.PerspectiveCamera(60, container.clientWidth / container.clientHeight, 0.1, 1000);
  camera.position.set(0, 5, 15);
  camera.lookAt(0, 0, 0);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.shadowMap.enabled = true;
  container.appendChild(renderer.domElement);
  
  const controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.target.set(0, 0, 0);
  
  // === Audio Setup ===
  const listener = new THREE.AudioListener();
  camera.add(listener);
  
  const audioLoader = new THREE.AudioLoader();
  
  // === Sound Sources (ตัวอย่างทำแบบ oscillator แทน file) ===
  const soundSources = [];
  
  function createOscillatorSource(frequency, position, color) {
    const positionalAudio = new THREE.PositionalAudio(listener);
    
    // สร้าง oscillator ด้วย Web Audio API โดยตรง
    const oscillator = listener.context.createOscillator();
    oscillator.type = 'sine';
    oscillator.frequency.setValueAtTime(frequency, listener.context.currentTime);
    oscillator.start();
    
    // Connect ผ่าน gain
    const gainNode = listener.context.createGain();
    gainNode.gain.value = 0.3;
    oscillator.connect(gainNode);
    
    // Set audio node
    positionalAudio.setNodeSource(gainNode);
    positionalAudio.setRefDistance(3);
    positionalAudio.setMaxDistance(20);
    positionalAudio.setRolloffFactor(1);
    
    // Create visual mesh
    const mesh = new THREE.Mesh(
      new THREE.SphereGeometry(0.5, 16, 16),
      new THREE.MeshStandardMaterial({
        color: color,
        emissive: color,
        emissiveIntensity: 0.3,
      })
    );
    mesh.position.set(...position);
    mesh.add(positionalAudio);
    scene.add(mesh);
    
    return { mesh, positionalAudio, oscillator, gainNode, color };
  }
  
  // สร้าง sound sources ตำแหน่งต่างๆ
  const sources = [
    { freq: 220, pos: [-6, 1, 0], color: '#ff6b6b' },
    { freq: 330, pos: [0, 1, -6], color: '#4ecdc4' },
    { freq: 440, pos: [6, 1, 0], color: '#f7dc6f' },
    { freq: 550, pos: [0, 1, 6], color: '#a29bfe' },
  ];
  
  sources.forEach(s => {
    soundSources.push(createOscillatorSource(s.freq, s.pos, s.color));
  });
  
  // === Range Indicators ===
  function createRangeCircle(radius, color) {
    const geometry = new THREE.RingGeometry(radius - 0.05, radius + 0.05, 64);
    const material = new THREE.MeshBasicMaterial({
      color,
      side: THREE.DoubleSide,
      transparent: true,
      opacity: 0.3
    });
    const ring = new THREE.Mesh(geometry, material);
    ring.rotation.x = -Math.PI / 2;
    return ring;
  }
  
  soundSources.forEach(src => {
    // refDistance circle
    const refCircle = createRangeCircle(3, src.color);
    src.mesh.add(refCircle);
    refCircle.position.y = -1;
  });
  
  // === Floor ===
  const floor = new THREE.Mesh(
    new THREE.PlaneGeometry(30, 30),
    new THREE.MeshStandardMaterial({ color: '#2c3e50', roughness: 0.8 })
  );
  floor.rotation.x = -Math.PI / 2;
  floor.receiveShadow = true;
  scene.add(floor);
  
  // === Listener Position Indicator ===
  const listenerMarker = new THREE.Mesh(
    new THREE.ConeGeometry(0.3, 0.6, 8),
    new THREE.MeshBasicMaterial({ color: 'white' })
  );
  listenerMarker.rotation.x = -Math.PI / 2;
  scene.add(listenerMarker);
  
  // === Lighting ===
  scene.add(new THREE.AmbientLight(0xffffff, 0.4));
  const dirLight = new THREE.DirectionalLight(0xffffff, 1);
  dirLight.position.set(5, 10, 5);
  dirLight.castShadow = true;
  scene.add(dirLight);
  
  // Point lights matching sound sources
  soundSources.forEach(src => {
    const light = new THREE.PointLight(src.color, 1, 15);
    light.position.copy(src.mesh.position);
    scene.add(light);
  });
  
  // === Pulse Animation ===
  function pulseMesh(mesh, baseScale, time, frequency) {
    const pulse = Math.sin(time * frequency * 2 * Math.PI) * 0.2 + 1;
    mesh.scale.setScalar(baseScale * pulse);
  }
  
  // === Animation ===
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const delta = clock.getDelta();
    const elapsed = clock.elapsedTime;
    
    controls.update();
    
    // Update listener marker position
    listenerMarker.position.copy(camera.position);
    listenerMarker.position.y = 0;
    
    // Pulse sound source meshes
    soundSources.forEach((src, i) => {
      pulseMesh(src.mesh, 1, elapsed, sources[i].freq / 440);
    });
    
    renderer.render(scene, camera);
  }
  
  animate();
  
  // Resize
  window.addEventListener('resize', () => {
    const w = container.clientWidth;
    const h = container.clientHeight;
    camera.aspect = w / h;
    camera.updateProjectionMatrix();
    renderer.setSize(w, h);
  });
  
  return () => {
    cancelAnimationFrame(animId);
    soundSources.forEach(src => {
      src.oscillator.stop();
      src.oscillator.disconnect();
      src.gainNode.disconnect();
    });
    listener.context.close();
    controls.dispose();
    renderer.dispose();
  };
}
```

---

## 71.5 Music Visualizer 3D

```javascript
// src/demos/musicVisualizer.js
import * as THREE from 'three';

const visualizerFragmentShader = `
  uniform float uTime;
  uniform float[64] uFrequencies;
  varying vec2 vUv;
  
  void main() {
    // ค้นหา frequency bin ที่ใกล้ที่สุด
    int bin = int(vUv.x * 64.0);
    float freq = uFrequencies[bin] / 255.0;
    
    // วาดแท่ง equalizer
    float bar = vUv.y < freq ? 1.0 : 0.0;
    
    vec3 color = mix(
      vec3(0.1, 0.3, 1.0),
      vec3(1.0, 0.1, 0.3),
      vUv.x
    );
    
    gl_FragColor = vec4(color * bar, bar);
  }
`;

export function createMusicVisualizer(container) {
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#000000');
  
  const camera = new THREE.PerspectiveCamera(60, container.clientWidth / container.clientHeight, 0.1, 1000);
  camera.position.set(0, 5, 15);
  camera.lookAt(0, 0, 0);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  container.appendChild(renderer.domElement);
  
  // Audio
  const listener = new THREE.AudioListener();
  camera.add(listener);
  
  const sound = new THREE.Audio(listener);
  const analyser = new THREE.AudioAnalyser(sound, 256);
  
  // === Equalizer Bars (3D) ===
  const BAR_COUNT = 64;
  const bars = [];
  const barGeometry = new THREE.BoxGeometry(0.3, 1, 0.3);
  
  for (let i = 0; i < BAR_COUNT; i++) {
    const hue = i / BAR_COUNT;
    const material = new THREE.MeshStandardMaterial({
      color: new THREE.Color().setHSL(hue, 1, 0.5),
      emissive: new THREE.Color().setHSL(hue, 1, 0.3),
      emissiveIntensity: 0.5,
      metalness: 0.5,
      roughness: 0.3,
    });
    
    const bar = new THREE.Mesh(barGeometry, material);
    
    // วางในรูปครึ่งวงกลม
    const angle = (i / BAR_COUNT) * Math.PI;
    const radius = 8;
    bar.position.x = Math.cos(angle) * radius;
    bar.position.z = -Math.sin(angle) * radius;
    
    scene.add(bar);
    bars.push(bar);
  }
  
  // === Circular Bars (Second ring) ===
  const outerBars = [];
  for (let i = 0; i < BAR_COUNT; i++) {
    const hue = (i / BAR_COUNT + 0.5) % 1;
    const material = new THREE.MeshStandardMaterial({
      color: new THREE.Color().setHSL(hue, 0.8, 0.6),
      emissive: new THREE.Color().setHSL(hue, 0.8, 0.3),
      emissiveIntensity: 0.3,
    });
    
    const bar = new THREE.Mesh(
      new THREE.BoxGeometry(0.2, 1, 0.2),
      material
    );
    
    // วงนอก
    const angle = (i / BAR_COUNT) * Math.PI * 2;
    const radius = 12;
    bar.position.x = Math.cos(angle) * radius;
    bar.position.z = Math.sin(angle) * radius;
    
    scene.add(bar);
    outerBars.push(bar);
  }
  
  // === Center Sphere ===
  const centerSphere = new THREE.Mesh(
    new THREE.SphereGeometry(2, 32, 32),
    new THREE.MeshStandardMaterial({
      color: '#4ecdc4',
      metalness: 0.8,
      roughness: 0.2,
    })
  );
  scene.add(centerSphere);
  
  // === Particles ===
  const particleCount = 5000;
  const particleGeo = new THREE.BufferGeometry();
  const positions = new Float32Array(particleCount * 3);
  
  for (let i = 0; i < particleCount; i++) {
    const angle = Math.random() * Math.PI * 2;
    const radius = 15 + Math.random() * 10;
    positions[i * 3] = Math.cos(angle) * radius;
    positions[i * 3 + 1] = (Math.random() - 0.5) * 20;
    positions[i * 3 + 2] = Math.sin(angle) * radius;
  }
  
  particleGeo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
  const particles = new THREE.Points(
    particleGeo,
    new THREE.PointsMaterial({ size: 0.05, color: 'white', transparent: true, opacity: 0.5 })
  );
  scene.add(particles);
  
  // === Lighting ===
  scene.add(new THREE.AmbientLight(0xffffff, 0.3));
  const pointLight1 = new THREE.PointLight('#4ecdc4', 2, 30);
  pointLight1.position.set(0, 10, 0);
  scene.add(pointLight1);
  
  const pointLight2 = new THREE.PointLight('#ff6b6b', 2, 30);
  pointLight2.position.set(0, -5, 0);
  scene.add(pointLight2);
  
  // === Audio Upload UI ===
  const uploadDiv = document.createElement('div');
  uploadDiv.style.cssText = `
    position: absolute;
    top: 20px;
    left: 50%;
    transform: translateX(-50%);
    text-align: center;
    color: white;
    font-family: sans-serif;
    z-index: 10;
  `;
  uploadDiv.innerHTML = `
    <div style="background: rgba(0,0,0,0.7); padding: 15px 25px; border-radius: 10px; border: 1px solid #4ecdc4;">
      <p style="margin: 0 0 10px; color: #4ecdc4">🎵 Music Visualizer</p>
      <input type="file" id="audioFile" accept="audio/*" style="display: none">
      <button onclick="document.getElementById('audioFile').click()" 
              style="background: #4ecdc4; border: none; padding: 8px 16px; border-radius: 6px; cursor: pointer; color: black; font-weight: bold">
        Upload Audio File
      </button>
    </div>
  `;
  container.style.position = 'relative';
  container.appendChild(uploadDiv);
  
  document.getElementById('audioFile')?.addEventListener('change', (e) => {
    const file = e.target.files[0];
    if (!file) return;
    
    const url = URL.createObjectURL(file);
    const audioLoader = new THREE.AudioLoader();
    audioLoader.load(url, (buffer) => {
      if (sound.isPlaying) sound.stop();
      sound.setBuffer(buffer);
      sound.setLoop(true);
      sound.setVolume(0.5);
      sound.play();
    });
  });
  
  // === Animation ===
  const clock = new THREE.Clock();
  let animId;
  const targetScales = new Array(BAR_COUNT).fill(1);
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const delta = clock.getDelta();
    const elapsed = clock.elapsedTime;
    
    // Get frequency data
    const freqData = analyser.getFrequencyData();
    const avgFreq = analyser.getAverageFrequency();
    
    // Update bars
    bars.forEach((bar, i) => {
      const freq = freqData[i] / 255;
      targetScales[i] = 0.1 + freq * 10;
      bar.scale.y += (targetScales[i] - bar.scale.y) * 0.3;
      bar.position.y = bar.scale.y / 2;
      
      // Update emissive intensity
      (bar.material).emissiveIntensity = freq * 2;
    });
    
    // Update outer bars (use different frequency range)
    outerBars.forEach((bar, i) => {
      const binIdx = Math.floor(i * (freqData.length / BAR_COUNT));
      const freq = freqData[binIdx] / 255;
      bar.scale.y += (0.1 + freq * 8 - bar.scale.y) * 0.2;
      bar.position.y = bar.scale.y / 2;
    });
    
    // Center sphere pulsing
    const bassPower = (freqData[0] + freqData[1] + freqData[2]) / (3 * 255);
    centerSphere.scale.setScalar(1 + bassPower * 0.5);
    
    // Rotate camera
    camera.position.x = Math.sin(elapsed * 0.1) * 18;
    camera.position.z = Math.cos(elapsed * 0.1) * 18;
    camera.position.y = 5 + Math.sin(elapsed * 0.05) * 3;
    camera.lookAt(0, 0, 0);
    
    // Particles floating
    particles.rotation.y = elapsed * 0.05;
    
    // Light pulsing
    pointLight1.intensity = 1 + bassPower * 5;
    pointLight2.intensity = 1 + (avgFreq / 255) * 3;
    
    renderer.render(scene, camera);
  }
  
  animate();
  
  // Resize
  window.addEventListener('resize', () => {
    const w = container.clientWidth;
    const h = container.clientHeight;
    camera.aspect = w / h;
    camera.updateProjectionMatrix();
    renderer.setSize(w, h);
  });
  
  return () => {
    cancelAnimationFrame(animId);
    if (sound.isPlaying) sound.stop();
    renderer.dispose();
    container.removeChild(uploadDiv);
  };
}
```

---

## 71.6 Footstep Audio กับ Distance

```javascript
// src/demos/footstepAudio.js
import * as THREE from 'three';

export function createFootstepDemo(container) {
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#2c3e50');
  
  const camera = new THREE.PerspectiveCamera(60, container.clientWidth / container.clientHeight, 0.1, 1000);
  camera.position.set(0, 5, 20);
  camera.lookAt(0, 0, 0);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.shadowMap.enabled = true;
  container.appendChild(renderer.domElement);
  
  // === Audio ===
  const listener = new THREE.AudioListener();
  camera.add(listener);
  
  const audioLoader = new THREE.AudioLoader();
  
  // === Walking Character ===
  const characterGroup = new THREE.Group();
  
  // Character body
  const body = new THREE.Mesh(
    new THREE.BoxGeometry(0.6, 1.0, 0.4),
    new THREE.MeshStandardMaterial({ color: '#e74c3c' })
  );
  body.position.y = 0.5;
  characterGroup.add(body);
  
  // Head
  const head = new THREE.Mesh(
    new THREE.BoxGeometry(0.4, 0.4, 0.4),
    new THREE.MeshStandardMaterial({ color: '#ffd700' })
  );
  head.position.y = 1.2;
  characterGroup.add(head);
  
  // Legs
  const legMat = new THREE.MeshStandardMaterial({ color: '#2c3e50' });
  const leftLeg = new THREE.Group();
  leftLeg.position.set(-0.15, 0, 0);
  const leftLegMesh = new THREE.Mesh(new THREE.BoxGeometry(0.2, 0.6, 0.2), legMat);
  leftLegMesh.position.y = -0.3;
  leftLeg.add(leftLegMesh);
  characterGroup.add(leftLeg);
  
  const rightLeg = new THREE.Group();
  rightLeg.position.set(0.15, 0, 0);
  const rightLegMesh = new THREE.Mesh(new THREE.BoxGeometry(0.2, 0.6, 0.2), legMat);
  rightLegMesh.position.y = -0.3;
  rightLeg.add(rightLegMesh);
  characterGroup.add(rightLeg);
  
  scene.add(characterGroup);
  
  // === Footstep Sound ===
  // สร้าง footstep sound ด้วย oscillator (แทนไฟล์)
  function createFootstepSound() {
    const posAudio = new THREE.PositionalAudio(listener);
    
    const playFootstep = () => {
      const ctx = listener.context;
      const osc = ctx.createOscillator();
      const gain = ctx.createGain();
      const filter = ctx.createBiquadFilter();
      
      // Low-pass filter สำหรับเสียงก้าว
      filter.type = 'lowpass';
      filter.frequency.value = 300;
      filter.Q.value = 0.5;
      
      osc.type = 'square';
      osc.frequency.setValueAtTime(100, ctx.currentTime);
      osc.frequency.exponentialRampToValueAtTime(40, ctx.currentTime + 0.1);
      
      gain.gain.setValueAtTime(0.3, ctx.currentTime);
      gain.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + 0.15);
      
      osc.connect(filter);
      filter.connect(gain);
      
      posAudio.setNodeSource(gain);
      posAudio.setRefDistance(2);
      posAudio.setMaxDistance(30);
      
      osc.start(ctx.currentTime);
      osc.stop(ctx.currentTime + 0.15);
    };
    
    return { posAudio, playFootstep };
  }
  
  const { posAudio, playFootstep } = createFootstepSound();
  characterGroup.add(posAudio);
  
  // === Path ===
  const pathPoints = [
    new THREE.Vector3(-10, 0, 0),
    new THREE.Vector3(-5, 0, 8),
    new THREE.Vector3(5, 0, 5),
    new THREE.Vector3(10, 0, -5),
    new THREE.Vector3(5, 0, -10),
    new THREE.Vector3(-5, 0, -8),
    new THREE.Vector3(-10, 0, 0),
  ];
  
  const curve = new THREE.CatmullRomCurve3(pathPoints, true);
  
  // Visualize path
  const pathGeo = new THREE.TubeGeometry(curve, 100, 0.05, 8, true);
  const pathMesh = new THREE.Mesh(
    pathGeo,
    new THREE.MeshBasicMaterial({ color: '#4ecdc4', transparent: true, opacity: 0.3 })
  );
  scene.add(pathMesh);
  
  // === Environment ===
  const floor = new THREE.Mesh(
    new THREE.PlaneGeometry(40, 40),
    new THREE.MeshStandardMaterial({ color: '#8fbc8f', roughness: 0.9 })
  );
  floor.rotation.x = -Math.PI / 2;
  floor.position.y = -0.3;
  floor.receiveShadow = true;
  scene.add(floor);
  
  // Trees
  for (let i = 0; i < 20; i++) {
    const tree = new THREE.Group();
    
    const trunk = new THREE.Mesh(
      new THREE.CylinderGeometry(0.1, 0.15, 1, 8),
      new THREE.MeshStandardMaterial({ color: '#8b4513' })
    );
    trunk.position.y = 0.5;
    tree.add(trunk);
    
    const crown = new THREE.Mesh(
      new THREE.ConeGeometry(0.8, 2, 8),
      new THREE.MeshStandardMaterial({ color: '#228b22' })
    );
    crown.position.y = 2;
    tree.add(crown);
    
    tree.position.set(
      (Math.random() - 0.5) * 35,
      -0.3,
      (Math.random() - 0.5) * 35
    );
    
    // ไม่วางบน path
    const distFromCenter = tree.position.length();
    if (distFromCenter > 15 || distFromCenter < 5) {
      scene.add(tree);
    }
  }
  
  // Lighting
  scene.add(new THREE.AmbientLight(0xffffff, 0.6));
  const sunLight = new THREE.DirectionalLight(0xffffff, 1);
  sunLight.position.set(10, 20, 5);
  sunLight.castShadow = true;
  scene.add(sunLight);
  
  // === Animation ===
  const clock = new THREE.Clock();
  let animId;
  let progress = 0;
  let lastStepProgress = 0;
  const STEP_INTERVAL = 0.04;  // ทุกๆ 4% ของ path = ก้าวหนึ่งก้าว
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const delta = clock.getDelta();
    const elapsed = clock.elapsedTime;
    
    // Move character along path
    progress = (progress + delta * 0.03) % 1;
    
    const currentPos = curve.getPointAt(progress);
    const nextPos = curve.getPointAt((progress + 0.01) % 1);
    
    characterGroup.position.copy(currentPos);
    characterGroup.lookAt(nextPos);
    
    // Leg animation
    leftLeg.rotation.x = Math.sin(elapsed * 8) * 0.4;
    rightLeg.rotation.x = -Math.sin(elapsed * 8) * 0.4;
    
    // Footstep sound trigger
    if (progress - lastStepProgress > STEP_INTERVAL || 
        (progress < STEP_INTERVAL && lastStepProgress > 1 - STEP_INTERVAL)) {
      playFootstep();
      lastStepProgress = progress;
    }
    
    // Camera follow character
    const camOffset = new THREE.Vector3(0, 5, 15);
    camera.position.lerp(
      characterGroup.position.clone().add(camOffset),
      0.05
    );
    camera.lookAt(characterGroup.position);
    
    renderer.render(scene, camera);
  }
  
  animate();
  
  // Resize
  window.addEventListener('resize', () => {
    const w = container.clientWidth;
    const h = container.clientHeight;
    camera.aspect = w / h;
    camera.updateProjectionMatrix();
    renderer.setSize(w, h);
  });
  
  return () => {
    cancelAnimationFrame(animId);
    listener.context.close();
    renderer.dispose();
  };
}
```

---

## 71.7 Web Audio API Integration

```javascript
// src/audio/webAudioHelpers.js

// สร้าง reverb effect
async function createReverb(audioContext, seconds = 2.5) {
  const sampleRate = audioContext.sampleRate;
  const length = sampleRate * seconds;
  const impulse = audioContext.createBuffer(2, length, sampleRate);
  
  for (let channel = 0; channel < 2; channel++) {
    const channelData = impulse.getChannelData(channel);
    for (let i = 0; i < length; i++) {
      channelData[i] = (Math.random() * 2 - 1) * Math.pow(1 - i / length, 2);
    }
  }
  
  const convolver = audioContext.createConvolver();
  convolver.buffer = impulse;
  return convolver;
}

// สร้าง equalizer
function createEqualizer(audioContext) {
  const filters = [];
  const frequencies = [60, 170, 350, 1000, 3500, 10000];
  
  frequencies.forEach(freq => {
    const filter = audioContext.createBiquadFilter();
    filter.type = freq === 60 ? 'lowshelf' : 
                  freq === 10000 ? 'highshelf' : 'peaking';
    filter.frequency.value = freq;
    filter.gain.value = 0;
    filter.Q.value = 0.8;
    filters.push(filter);
  });
  
  // Chain filters
  for (let i = 0; i < filters.length - 1; i++) {
    filters[i].connect(filters[i + 1]);
  }
  
  return {
    input: filters[0],
    output: filters[filters.length - 1],
    filters,
    setGain(index, gain) {
      if (filters[index]) {
        filters[index].gain.value = gain;
      }
    }
  };
}

// Audio-reactive parameter mapper
function createAudioMapper(analyser) {
  return {
    getBass() {
      const data = analyser.getFrequencyData();
      return data.slice(0, 8).reduce((sum, v) => sum + v, 0) / (8 * 255);
    },
    getMid() {
      const data = analyser.getFrequencyData();
      const mid = data.slice(16, 64);
      return mid.reduce((sum, v) => sum + v, 0) / (48 * 255);
    },
    getTreble() {
      const data = analyser.getFrequencyData();
      const treble = data.slice(64);
      return treble.reduce((sum, v) => sum + v, 0) / (treble.length * 255);
    },
    getAverageDB() {
      return analyser.getAverageFrequency() / 255;
    }
  };
}

export { createReverb, createEqualizer, createAudioMapper };
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Spatial Audio Explorer
สร้าง interactive spatial audio demo:
- 8 sound sources ในตำแหน่งต่างๆ
- ฟังเสียงจะเปลี่ยนเมื่อเดินเข้าใกล้
- Visualize distance attenuation
- Directional cone visualization

### แบบฝึกหัดที่ 2: Music Visualizer Upgrade
ปรับปรุง music visualizer:
- 360° particle ring ตอบสนอง bass
- Waveform visualization ตรงกลาง
- Color changing ตาม frequency
- Camera shake ตาม beat

### แบบฝึกหัดที่ 3: 3D Soundscape
สร้าง interactive soundscape:
- Forest sounds (birds, wind, water)
- เดินได้ใน scene
- เสียงเปลี่ยนตามตำแหน่ง
- Day/night cycle กับเสียงที่ต่างกัน

### แบบฝึกหัดที่ 4: Drum Machine Visual
สร้าง drum machine ที่:
- มี 8 tracks
- แต่ละ beat สร้าง 3D explosion
- Analyser แสดง waveform
- BPM sync กับ animation

### แบบฝึกหัดที่ 5: Audio Tour
สร้าง virtual audio tour:
- Move through waypoints ด้วย animation
- Audio narration ที่ fade in/out ตามตำแหน่ง
- Ambient sounds ที่เปลี่ยนตาม zone
- Background music ที่ blend กัน

---

## สรุป

ในบทนี้เราเรียนรู้:
1. **AudioListener** - Setup audio system ใน Three.js
2. **Audio vs PositionalAudio** - Ambient vs 3D spatial audio
3. **AudioLoader** - โหลด audio files
4. **AudioAnalyser** - FFT data สำหรับ visualizations
5. **Music Visualizer** - Equalizer bars reactive ต่อ music
6. **Footstep System** - Procedural audio กับ character movement
7. **Web Audio API** - Reverb, EQ, และ audio effects

บทถัดไปเราจะเรียน **Video & Canvas Textures**
