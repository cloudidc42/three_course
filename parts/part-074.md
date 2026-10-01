# Part 74: 3D Text & Typography

## บทนำ

3D Text ใน Three.js มีหลายวิธี ตั้งแต่ built-in TextGeometry ไปจนถึง library ที่ทรงพลังกว่า เช่น troika-three-text ที่รองรับ SDF fonts

---

## 74.1 TextGeometry พื้นฐาน

```javascript
import * as THREE from 'three';
import { FontLoader } from 'three/examples/jsm/loaders/FontLoader.js';
import { TextGeometry } from 'three/examples/jsm/geometries/TextGeometry.js';

export async function createBasicTextDemo(container) {
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#0a0a1a');
  
  const camera = new THREE.PerspectiveCamera(45, container.clientWidth / container.clientHeight, 0.1, 100);
  camera.position.set(0, 2, 10);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  container.appendChild(renderer.domElement);
  
  // Load font
  const fontLoader = new FontLoader();
  const font = await new Promise((resolve, reject) => {
    fontLoader.load(
      'https://threejs.org/examples/fonts/helvetiker_bold.typeface.json',
      resolve,
      undefined,
      reject
    );
  });
  
  // สร้าง TextGeometry
  const textGeo = new TextGeometry('THREE.JS', {
    font,
    size: 1,           // ขนาด font
    height: 0.3,       // ความหนา (depth)
    curveSegments: 12, // ความละเอียดของ curves
    bevelEnabled: true,
    bevelThickness: 0.03,
    bevelSize: 0.02,
    bevelOffset: 0,
    bevelSegments: 5,
  });
  
  // Center the text
  textGeo.computeBoundingBox();
  const center = new THREE.Vector3();
  textGeo.boundingBox.getCenter(center);
  textGeo.translate(-center.x, -center.y, -center.z);
  
  const textMesh = new THREE.Mesh(
    textGeo,
    new THREE.MeshStandardMaterial({
      color: '#4ecdc4',
      metalness: 0.5,
      roughness: 0.3,
    })
  );
  scene.add(textMesh);
  
  // Lighting
  scene.add(new THREE.AmbientLight(0xffffff, 0.4));
  const light1 = new THREE.DirectionalLight(0xffffff, 1);
  light1.position.set(5, 5, 5);
  scene.add(light1);
  
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const t = clock.getElapsedTime();
    textMesh.rotation.y = Math.sin(t * 0.5) * 0.3;
    renderer.render(scene, camera);
  }
  
  animate();
  
  return () => { cancelAnimationFrame(animId); renderer.dispose(); };
}
```

---

## 74.2 Text3D ด้วย Drei (R3F)

```jsx
// src/components/Text3DDemo.jsx
import { Canvas } from '@react-three/fiber';
import { Text3D, Center, OrbitControls, MeshTransmissionMaterial, Float } from '@react-three/drei';
import { useFrame } from '@react-three/fiber';
import { useRef, useState } from 'react';

function FloatingLetter({ char, position, delay = 0 }) {
  const meshRef = useRef();
  const [hovered, setHovered] = useState(false);
  
  useFrame((state) => {
    if (!meshRef.current) return;
    const t = state.clock.elapsedTime + delay;
    meshRef.current.position.y = position[1] + Math.sin(t) * 0.2;
    meshRef.current.rotation.y = Math.sin(t * 0.3) * 0.2;
  });
  
  return (
    <Float speed={1} rotationIntensity={0.5} floatIntensity={0.5}>
      <Text3D
        ref={meshRef}
        font="https://threejs.org/examples/fonts/helvetiker_bold.typeface.json"
        size={1.2}
        height={0.3}
        bevelEnabled
        bevelThickness={0.05}
        bevelSize={0.02}
        position={position}
        onPointerEnter={() => setHovered(true)}
        onPointerLeave={() => setHovered(false)}
      >
        {char}
        <meshStandardMaterial
          color={hovered ? '#ff6b6b' : '#4ecdc4'}
          metalness={0.7}
          roughness={0.2}
          emissive={hovered ? '#ff6b6b' : '#4ecdc4'}
          emissiveIntensity={hovered ? 0.5 : 0.1}
        />
      </Text3D>
    </Float>
  );
}

export default function Text3DDemo() {
  const text = 'THREE';
  
  return (
    <div style={{ width: '100%', height: '500px' }}>
      <Canvas camera={{ position: [0, 0, 12], fov: 45 }}>
        <color attach="background" args={['#050510']} />
        <ambientLight intensity={0.5} />
        <directionalLight position={[10, 10, 10]} intensity={1} />
        <pointLight position={[-5, 5, -5]} color="#ff6b6b" intensity={1} />
        
        <Center>
          <group>
            {text.split('').map((char, i) => (
              <FloatingLetter
                key={i}
                char={char}
                position={[(i - text.length / 2) * 1.5, 0, 0]}
                delay={i * 0.3}
              />
            ))}
          </group>
        </Center>
        
        <OrbitControls enableZoom={false} autoRotate autoRotateSpeed={0.5} />
      </Canvas>
    </div>
  );
}
```

---

## 74.3 troika-three-text (SDF Text)

troika-three-text ใช้ Signed Distance Field เพื่อ text คมชัดทุก zoom level

```javascript
// npm install troika-three-text
import { Text } from 'troika-three-text';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

export function createTroikaTextDemo(container) {
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#0a0a1a');
  
  const camera = new THREE.PerspectiveCamera(45, container.clientWidth / container.clientHeight, 0.1, 100);
  camera.position.set(0, 0, 15);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  container.appendChild(renderer.domElement);
  
  const controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  
  // === troika Text Object ===
  const myText = new Text();
  scene.add(myText);
  
  // ตั้งค่า text properties
  myText.text = 'Hello\nWorld\n3D!';
  myText.font = 'https://fonts.gstatic.com/s/inter/v13/UcCO3FwrK3iLTeHuS_fvQtMwCp50KnMw2boKoduKmMEVuLyfAZ9hiJ-Ek-_EeA.woff';
  myText.fontSize = 2;
  myText.color = 0x4ecdc4;
  myText.anchorX = 'center';
  myText.anchorY = 'middle';
  myText.textAlign = 'center';
  
  // Outline
  myText.outlineWidth = 0.05;
  myText.outlineColor = 0xff6b6b;
  myText.outlineOpacity = 0.8;
  
  // Stroke
  myText.strokeWidth = 0.02;
  myText.strokeColor = 0xffd166;
  
  // Letter spacing & line height
  myText.letterSpacing = 0.05;
  myText.lineHeight = 1.3;
  
  // Material override
  myText.material = new THREE.MeshStandardMaterial({
    color: 0x4ecdc4,
    metalness: 0.3,
    roughness: 0.5,
  });
  
  // ต้อง sync หลัง set properties
  myText.sync();
  
  // === Multiple Text Objects ===
  const texts = [
    { text: 'WebGL', position: [-5, 3, 0], color: 0xff6b6b, size: 1 },
    { text: 'Three.js', position: [0, 0, 0], color: 0x4ecdc4, size: 1.5 },
    { text: 'JavaScript', position: [5, -3, -2], color: 0xffd166, size: 0.8 },
  ];
  
  const textObjects = [];
  
  texts.forEach(t => {
    const textObj = new Text();
    textObj.text = t.text;
    textObj.fontSize = t.size;
    textObj.color = t.color;
    textObj.anchorX = 'center';
    textObj.anchorY = 'middle';
    textObj.position.set(...t.position);
    textObj.sync();
    scene.add(textObj);
    textObjects.push(textObj);
  });
  
  // Lighting
  scene.add(new THREE.AmbientLight(0xffffff, 0.5));
  scene.add(new THREE.DirectionalLight(0xffffff, 1).position.set(5, 10, 5) && new THREE.DirectionalLight(0xffffff, 1));
  
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const t = clock.getElapsedTime();
    
    // Animate texts
    textObjects.forEach((obj, i) => {
      obj.rotation.y = Math.sin(t * 0.3 + i * Math.PI / 3) * 0.3;
      obj.position.y = texts[i].position[1] + Math.sin(t * 0.5 + i) * 0.3;
    });
    
    controls.update();
    renderer.render(scene, camera);
  }
  
  animate();
  
  return () => {
    cancelAnimationFrame(animId);
    // cleanup troika text objects
    myText.dispose();
    textObjects.forEach(t => t.dispose());
    controls.dispose();
    renderer.dispose();
  };
}
```

---

## 74.4 Text Along Path

```javascript
// src/demos/textOnPath.js
import * as THREE from 'three';
import { FontLoader } from 'three/examples/jsm/loaders/FontLoader.js';
import { TextGeometry } from 'three/examples/jsm/geometries/TextGeometry.js';

export async function createTextOnPathDemo(container) {
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#0a0a1a');
  
  const camera = new THREE.PerspectiveCamera(45, container.clientWidth / container.clientHeight, 0.1, 100);
  camera.position.set(0, 5, 15);
  camera.lookAt(0, 0, 0);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  container.appendChild(renderer.domElement);
  
  const fontLoader = new FontLoader();
  const font = await new Promise(resolve => {
    fontLoader.load('https://threejs.org/examples/fonts/helvetiker_regular.typeface.json', resolve);
  });
  
  // Define path
  const path = new THREE.CatmullRomCurve3([
    new THREE.Vector3(-8, 0, 0),
    new THREE.Vector3(-4, 3, -2),
    new THREE.Vector3(0, 0, -4),
    new THREE.Vector3(4, -3, -2),
    new THREE.Vector3(8, 0, 0),
  ]);
  
  // Visualize path
  const pathPoints = path.getPoints(100);
  const pathLine = new THREE.Line(
    new THREE.BufferGeometry().setFromPoints(pathPoints),
    new THREE.LineBasicMaterial({ color: '#333', transparent: true, opacity: 0.3 })
  );
  scene.add(pathLine);
  
  // Place text along path
  const TEXT = 'THREE.JS IS AWESOME • ';
  const CHAR_SPACING = 0.7;
  const totalLength = TEXT.length * CHAR_SPACING;
  
  const charMeshes = [];
  
  TEXT.split('').forEach((char, i) => {
    if (char === ' ') return;
    
    const charGeo = new TextGeometry(char, {
      font,
      size: 0.5,
      height: 0.1,
      bevelEnabled: true,
      bevelThickness: 0.02,
      bevelSize: 0.01,
    });
    
    charGeo.computeBoundingBox();
    const center = new THREE.Vector3();
    charGeo.boundingBox.getCenter(center);
    charGeo.translate(-center.x, -center.y, 0);
    
    const hue = (i / TEXT.length) * 360;
    const mesh = new THREE.Mesh(
      charGeo,
      new THREE.MeshStandardMaterial({
        color: new THREE.Color(`hsl(${hue}, 80%, 60%)`),
        metalness: 0.5,
        roughness: 0.3,
      })
    );
    
    scene.add(mesh);
    charMeshes.push({ mesh, t: i / TEXT.length });
  });
  
  // Lighting
  scene.add(new THREE.AmbientLight(0xffffff, 0.5));
  const dirLight = new THREE.DirectionalLight(0xffffff, 1);
  dirLight.position.set(5, 10, 5);
  scene.add(dirLight);
  
  const clock = new THREE.Clock();
  let animId;
  const dummy = new THREE.Object3D();
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    
    const speed = 0.1;
    const offset = (elapsed * speed) % 1;
    
    charMeshes.forEach(({ mesh, t }) => {
      const pathT = (t + offset) % 1;
      
      // Get position on path
      const pos = path.getPoint(pathT);
      
      // Get tangent for orientation
      const tangent = path.getTangent(pathT);
      
      mesh.position.copy(pos);
      
      // Align to tangent
      dummy.position.copy(pos);
      dummy.lookAt(pos.clone().add(tangent));
      mesh.rotation.copy(dummy.rotation);
      
      // Slight y-axis tilt to face outward
      mesh.rotation.z = Math.PI / 2;
    });
    
    // Rotate camera slowly
    camera.position.x = Math.sin(elapsed * 0.1) * 15;
    camera.position.z = Math.cos(elapsed * 0.1) * 15;
    camera.lookAt(0, 0, 0);
    
    renderer.render(scene, camera);
  }
  
  animate();
  
  return () => {
    cancelAnimationFrame(animId);
    renderer.dispose();
  };
}
```

---

## 74.5 Particle Text Morphing

```javascript
// src/demos/particleText.js
import * as THREE from 'three';
import { FontLoader } from 'three/examples/jsm/loaders/FontLoader.js';
import { TextGeometry } from 'three/examples/jsm/geometries/TextGeometry.js';

export async function createParticleTextDemo(container) {
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#050510');
  
  const camera = new THREE.PerspectiveCamera(45, container.clientWidth / container.clientHeight, 0.1, 100);
  camera.position.set(0, 0, 15);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  container.appendChild(renderer.domElement);
  
  // Load font
  const fontLoader = new FontLoader();
  const font = await new Promise(resolve => {
    fontLoader.load('https://threejs.org/examples/fonts/helvetiker_bold.typeface.json', resolve);
  });
  
  const PARTICLE_COUNT = 5000;
  
  // สร้าง particle positions จาก text
  function getTextPositions(text, count) {
    const textGeo = new TextGeometry(text, {
      font,
      size: 2,
      height: 0.1,
    });
    
    textGeo.computeBoundingBox();
    const center = new THREE.Vector3();
    textGeo.boundingBox.getCenter(center);
    
    const positions = textGeo.attributes.position;
    const textPositions = [];
    
    for (let i = 0; i < positions.count; i++) {
      textPositions.push(new THREE.Vector3(
        positions.getX(i) - center.x,
        positions.getY(i) - center.y,
        positions.getZ(i) - center.z
      ));
    }
    
    // Sample or repeat to match count
    const result = new Float32Array(count * 3);
    for (let i = 0; i < count; i++) {
      const src = textPositions[i % textPositions.length];
      result[i * 3] = src.x + (Math.random() - 0.5) * 0.1;
      result[i * 3 + 1] = src.y + (Math.random() - 0.5) * 0.1;
      result[i * 3 + 2] = src.z + (Math.random() - 0.5) * 0.1;
    }
    
    textGeo.dispose();
    return result;
  }
  
  // Target positions สำหรับ different words
  const WORDS = ['HELLO', 'THREE', 'WORLD'];
  const wordPositions = await Promise.all(WORDS.map(w => getTextPositions(w, PARTICLE_COUNT)));
  
  // Current floating positions
  const floatingPositions = new Float32Array(PARTICLE_COUNT * 3);
  for (let i = 0; i < PARTICLE_COUNT; i++) {
    floatingPositions[i * 3] = (Math.random() - 0.5) * 20;
    floatingPositions[i * 3 + 1] = (Math.random() - 0.5) * 10;
    floatingPositions[i * 3 + 2] = (Math.random() - 0.5) * 5;
  }
  
  // Current positions (lerp target)
  const currentPositions = floatingPositions.slice();
  const targetPositions = wordPositions[0].slice();
  
  // Particle system
  const geometry = new THREE.BufferGeometry();
  geometry.setAttribute('position', new THREE.BufferAttribute(currentPositions, 3));
  
  const material = new THREE.PointsMaterial({
    size: 0.05,
    color: '#4ecdc4',
    transparent: true,
    opacity: 0.8,
    sizeAttenuation: true,
  });
  
  const particles = new THREE.Points(geometry, material);
  scene.add(particles);
  
  // Morphing state
  let currentWordIndex = 0;
  let morphProgress = 1;
  let morphing = false;
  const morphFrom = new Float32Array(PARTICLE_COUNT * 3);
  const MORPH_DURATION = 1.5;
  let morphStartTime = 0;
  
  function startMorph(toIndex) {
    if (morphing) return;
    morphing = true;
    currentWordIndex = toIndex;
    morphFrom.set(currentPositions);
    morphProgress = 0;
    morphStartTime = clock.getElapsedTime();
  }
  
  // Auto cycle words
  let wordTimer = 0;
  
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    const delta = clock.getDelta();
    
    wordTimer += delta;
    
    // Auto-change word every 3 seconds
    if (wordTimer > 3 && !morphing) {
      wordTimer = 0;
      const nextWord = (currentWordIndex + 1) % WORDS.length;
      startMorph(nextWord);
    }
    
    // Update morph
    if (morphing) {
      const morphElapsed = elapsed - morphStartTime;
      morphProgress = Math.min(morphElapsed / MORPH_DURATION, 1);
      
      // Ease in-out cubic
      const ease = morphProgress < 0.5
        ? 4 * morphProgress * morphProgress * morphProgress
        : 1 - Math.pow(-2 * morphProgress + 2, 3) / 2;
      
      const target = wordPositions[currentWordIndex];
      
      for (let i = 0; i < PARTICLE_COUNT; i++) {
        currentPositions[i * 3] = morphFrom[i * 3] + (target[i * 3] - morphFrom[i * 3]) * ease;
        currentPositions[i * 3 + 1] = morphFrom[i * 3 + 1] + (target[i * 3 + 1] - morphFrom[i * 3 + 1]) * ease;
        currentPositions[i * 3 + 2] = morphFrom[i * 3 + 2] + (target[i * 3 + 2] - morphFrom[i * 3 + 2]) * ease;
      }
      
      geometry.attributes.position.needsUpdate = true;
      
      if (morphProgress >= 1) morphing = false;
    } else {
      // Gentle scatter in formed state
      const target = wordPositions[currentWordIndex];
      for (let i = 0; i < PARTICLE_COUNT; i++) {
        const t = elapsed * 0.5 + i * 0.001;
        currentPositions[i * 3] = target[i * 3] + Math.sin(t) * 0.02;
        currentPositions[i * 3 + 1] = target[i * 3 + 1] + Math.cos(t * 1.3) * 0.02;
        currentPositions[i * 3 + 2] = target[i * 3 + 2] + Math.sin(t * 0.7) * 0.02;
      }
      geometry.attributes.position.needsUpdate = true;
    }
    
    // Color based on position
    material.color.setHSL((elapsed * 0.05) % 1, 0.8, 0.6);
    
    renderer.render(scene, camera);
  }
  
  animate();
  
  return () => {
    cancelAnimationFrame(animId);
    geometry.dispose();
    material.dispose();
    renderer.dispose();
  };
}
```

---

## 74.6 Word Cloud ใน 3D

```javascript
// src/demos/wordCloud3D.js
import * as THREE from 'three';
import { FontLoader } from 'three/examples/jsm/loaders/FontLoader.js';
import { TextGeometry } from 'three/examples/jsm/geometries/TextGeometry.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

export async function createWordCloudDemo(container) {
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#050510');
  
  const camera = new THREE.PerspectiveCamera(45, container.clientWidth / container.clientHeight, 0.1, 200);
  camera.position.set(0, 0, 30);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  container.appendChild(renderer.domElement);
  
  const controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.autoRotate = true;
  controls.autoRotateSpeed = 0.5;
  
  const fontLoader = new FontLoader();
  const font = await new Promise(resolve => {
    fontLoader.load('https://threejs.org/examples/fonts/helvetiker_bold.typeface.json', resolve);
  });
  
  const WORDS = [
    { text: 'JavaScript', weight: 5 },
    { text: 'WebGL', weight: 4 },
    { text: 'Three.js', weight: 5 },
    { text: 'React', weight: 4 },
    { text: 'Node.js', weight: 3 },
    { text: 'TypeScript', weight: 4 },
    { text: 'Canvas', weight: 2 },
    { text: 'GLSL', weight: 3 },
    { text: 'Shader', weight: 3 },
    { text: 'Animation', weight: 3 },
    { text: 'Physics', weight: 2 },
    { text: 'Particles', weight: 3 },
    { text: 'VR', weight: 2 },
    { text: 'AR', weight: 2 },
    { text: 'GPU', weight: 2 },
    { text: 'Performance', weight: 2 },
  ];
  
  const wordMeshes = [];
  const SPHERE_RADIUS = 12;
  
  WORDS.forEach((wordData, index) => {
    const size = 0.3 + wordData.weight * 0.15;
    
    const geo = new TextGeometry(wordData.text, {
      font,
      size,
      height: size * 0.2,
      bevelEnabled: true,
      bevelThickness: size * 0.02,
      bevelSize: size * 0.01,
    });
    
    geo.computeBoundingBox();
    const center = new THREE.Vector3();
    geo.boundingBox.getCenter(center);
    geo.translate(-center.x, -center.y, -center.z);
    
    const hue = (index / WORDS.length) * 360;
    const mesh = new THREE.Mesh(
      geo,
      new THREE.MeshStandardMaterial({
        color: new THREE.Color(`hsl(${hue}, 80%, 65%)`),
        metalness: 0.3,
        roughness: 0.4,
        transparent: true,
        opacity: 0.85,
      })
    );
    
    // Fibonacci sphere distribution
    const golden = (1 + Math.sqrt(5)) / 2;
    const phi = Math.acos(1 - (2 * (index + 0.5)) / WORDS.length);
    const theta = 2 * Math.PI * index / golden;
    
    const x = SPHERE_RADIUS * Math.sin(phi) * Math.cos(theta);
    const y = SPHERE_RADIUS * Math.sin(phi) * Math.sin(theta);
    const z = SPHERE_RADIUS * Math.cos(phi);
    
    mesh.position.set(x, y, z);
    
    // Face center
    mesh.lookAt(0, 0, 0);
    mesh.rotateY(Math.PI);
    
    scene.add(mesh);
    wordMeshes.push({ mesh, originalPos: new THREE.Vector3(x, y, z), hue });
  });
  
  // Center sphere
  scene.add(new THREE.Mesh(
    new THREE.SphereGeometry(1, 32, 32),
    new THREE.MeshStandardMaterial({ color: '#4ecdc4', metalness: 0.8, roughness: 0.1, emissive: '#4ecdc4', emissiveIntensity: 0.2 })
  ));
  
  // Lighting
  scene.add(new THREE.AmbientLight(0xffffff, 0.6));
  const light = new THREE.DirectionalLight(0xffffff, 1);
  light.position.set(10, 10, 10);
  scene.add(light);
  
  // Mouse interaction
  const raycaster = new THREE.Raycaster();
  const mouse = new THREE.Vector2();
  let hoveredMesh = null;
  
  container.addEventListener('mousemove', (e) => {
    const rect = container.getBoundingClientRect();
    mouse.x = ((e.clientX - rect.left) / rect.width) * 2 - 1;
    mouse.y = -((e.clientY - rect.top) / rect.height) * 2 + 1;
  });
  
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    
    // Raycast for hover
    raycaster.setFromCamera(mouse, camera);
    const meshes = wordMeshes.map(w => w.mesh);
    const intersects = raycaster.intersectObjects(meshes);
    
    const newHovered = intersects.length > 0 ? intersects[0].object : null;
    
    if (newHovered !== hoveredMesh) {
      if (hoveredMesh) {
        hoveredMesh.material.emissiveIntensity = 0;
        hoveredMesh.material.opacity = 0.85;
      }
      if (newHovered) {
        newHovered.material.emissive = new THREE.Color(newHovered.material.color);
        newHovered.material.emissiveIntensity = 0.5;
        newHovered.material.opacity = 1;
        container.style.cursor = 'pointer';
      } else {
        container.style.cursor = 'default';
      }
      hoveredMesh = newHovered;
    }
    
    // Gentle breathing animation
    wordMeshes.forEach(({ mesh, originalPos, hue }, i) => {
      const breathe = 1 + Math.sin(elapsed * 0.5 + i * 0.3) * 0.02;
      mesh.position.copy(originalPos).multiplyScalar(breathe);
      
      // Subtle color shift
      mesh.material.color.setHSL(
        ((hue / 360 + elapsed * 0.02) % 1),
        0.8,
        0.65
      );
    });
    
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
    wordMeshes.forEach(w => { w.mesh.geometry.dispose(); w.mesh.material.dispose(); });
    controls.dispose();
    renderer.dispose();
  };
}
```

---

## 74.7 Typewriter Effect

```javascript
// src/demos/typewriter.js
import * as THREE from 'three';
import { FontLoader } from 'three/examples/jsm/loaders/FontLoader.js';
import { TextGeometry } from 'three/examples/jsm/geometries/TextGeometry.js';

export async function createTypewriterDemo(container) {
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#050510');
  
  const camera = new THREE.PerspectiveCamera(45, container.clientWidth / container.clientHeight, 0.1, 100);
  camera.position.set(0, 0, 12);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  container.appendChild(renderer.domElement);
  
  const fontLoader = new FontLoader();
  const font = await new Promise(resolve => {
    fontLoader.load('https://threejs.org/examples/fonts/helvetiker_regular.typeface.json', resolve);
  });
  
  const LINES = [
    '> Loading Three.js...',
    '> Scene initialized',
    '> Camera: PerspectiveCamera',
    '> Renderer: WebGLRenderer',
    '> Status: READY',
  ];
  
  let currentLineIndex = 0;
  let currentCharIndex = 0;
  let displayText = '';
  let lastCharTime = 0;
  const CHAR_DELAY = 0.05;
  
  // Text mesh group
  const textGroup = new THREE.Group();
  scene.add(textGroup);
  
  let currentTextMesh = null;
  
  function buildTextMesh(text) {
    if (currentTextMesh) {
      currentTextMesh.geometry.dispose();
      textGroup.remove(currentTextMesh);
    }
    
    if (!text) return;
    
    const geo = new TextGeometry(text, {
      font,
      size: 0.4,
      height: 0.05,
    });
    
    geo.computeBoundingBox();
    const bb = geo.boundingBox;
    
    currentTextMesh = new THREE.Mesh(
      geo,
      new THREE.MeshBasicMaterial({ color: '#00ff88' })
    );
    
    // Align left
    const lineY = -(currentLineIndex % LINES.length) * 0.7;
    currentTextMesh.position.set(-6, lineY + 1.5, 0);
    textGroup.add(currentTextMesh);
  }
  
  // Cursor mesh
  const cursorMesh = new THREE.Mesh(
    new THREE.BoxGeometry(0.02, 0.4, 0.05),
    new THREE.MeshBasicMaterial({ color: '#00ff88' })
  );
  scene.add(cursorMesh);
  
  // Grid background
  const gridHelper = new THREE.GridHelper(20, 20, '#111133', '#111133');
  gridHelper.rotation.x = Math.PI / 2;
  gridHelper.position.z = -1;
  scene.add(gridHelper);
  
  // Scanlines effect (using line meshes)
  for (let y = -5; y < 5; y += 0.4) {
    const scanLine = new THREE.Mesh(
      new THREE.PlaneGeometry(14, 0.01),
      new THREE.MeshBasicMaterial({ color: '#00ff88', transparent: true, opacity: 0.03 })
    );
    scanLine.position.set(0, y, -0.5);
    scene.add(scanLine);
  }
  
  // Track all completed lines
  const completedLineMeshes = [];
  
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    
    // Typewriter effect
    if (currentLineIndex < LINES.length) {
      const currentLine = LINES[currentLineIndex];
      
      if (elapsed - lastCharTime > CHAR_DELAY) {
        lastCharTime = elapsed;
        
        if (currentCharIndex < currentLine.length) {
          displayText = currentLine.substring(0, currentCharIndex + 1);
          currentCharIndex++;
          buildTextMesh(displayText);
        } else {
          // Line complete - save it and move to next
          setTimeout(() => {
            if (currentTextMesh) {
              completedLineMeshes.push(currentTextMesh);
              currentTextMesh = null;
            }
            displayText = '';
            currentCharIndex = 0;
            currentLineIndex++;
          }, 500);
        }
      }
    }
    
    // Position cursor after last char
    if (currentTextMesh) {
      const bb = currentTextMesh.geometry.boundingBox;
      if (bb) {
        cursorMesh.position.set(
          currentTextMesh.position.x + bb.max.x + 0.05,
          currentTextMesh.position.y + 0.2,
          0
        );
      }
    }
    
    // Cursor blink
    cursorMesh.visible = Math.sin(elapsed * 6) > 0;
    
    // Scroll text up as new lines added
    textGroup.position.y = Math.min(currentLineIndex * 0.7, 3);
    
    renderer.render(scene, camera);
  }
  
  animate();
  
  return () => {
    cancelAnimationFrame(animId);
    renderer.dispose();
  };
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: 3D Logo Animation
สร้าง logo ที่:
- Text ประกอบด้วย particles
- Assemble/disassemble animation
- Metallic material พร้อม environment map
- Camera orbit รอบ logo

### แบบฝึกหัดที่ 2: 3D Score Counter
สร้าง score counter ที่:
- Number flip animation
- Particle burst เมื่อ score เพิ่ม
- Multiple text layers (depth)
- Glow effect

### แบบฝึกหัดที่ 3: Interactive Word Art
สร้าง word art ที่:
- Words react to mouse
- Physics-based collision
- Color themes
- Export as image

### แบบฝึกหัดที่ 4: Multi-language Billboard
สร้าง billboard ที่:
- แสดง text หลายภาษา
- Smooth transition ระหว่างภาษา
- RTL support
- Auto-scale

---

## สรุป

ในบทนี้เราเรียนรู้:
1. **TextGeometry** - Built-in 3D text
2. **Text3D (Drei)** - ง่ายกว่าใน React
3. **troika-three-text** - SDF text คมชัดทุก zoom
4. **Text on Path** - Text ตาม curve
5. **Particle Text** - Morphing particles
6. **Word Cloud** - Fibonacci distribution
7. **Typewriter Effect** - Character-by-character

บทถัดไปเราจะเรียน **SVG & 2D Shape Extrusion**
