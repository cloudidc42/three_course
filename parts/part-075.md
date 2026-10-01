# Part 75: SVG & 2D Shape Extrusion

## บทนำ

Three.js รองรับ SVG loading และ 2D Shape extrusion เพื่อสร้าง 3D objects จาก vector paths เหมาะสำหรับ logo to 3D, custom shapes, และ architectural profiles

---

## 75.1 SVGLoader พื้นฐาน

```javascript
import * as THREE from 'three';
import { SVGLoader } from 'three/examples/jsm/loaders/SVGLoader.js';

async function loadSVG(url) {
  const loader = new SVGLoader();
  return new Promise((resolve, reject) => {
    loader.load(url, resolve, undefined, reject);
  });
}

export async function createSVGDemo(container) {
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#f0f0f0');
  
  const camera = new THREE.PerspectiveCamera(45, container.clientWidth / container.clientHeight, 0.1, 1000);
  camera.position.set(0, 0, 200);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  container.appendChild(renderer.domElement);
  
  // Load SVG file
  const loader = new SVGLoader();
  const svgData = loader.parse(`
    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 100 100">
      <circle cx="50" cy="50" r="40" fill="#4ecdc4"/>
      <polygon points="50,15 90,85 10,85" fill="#ff6b6b"/>
      <rect x="25" y="35" width="50" height="30" rx="5" fill="#ffd166"/>
    </svg>
  `);
  
  const group = new THREE.Group();
  
  svgData.paths.forEach((path, pathIndex) => {
    const shapes = SVGLoader.createShapes(path);
    
    shapes.forEach(shape => {
      // Flat 2D (ShapeGeometry)
      const shapeGeo = new THREE.ShapeGeometry(shape);
      const shapeMesh = new THREE.Mesh(
        shapeGeo,
        new THREE.MeshBasicMaterial({
          color: path.color,
          side: THREE.DoubleSide
        })
      );
      group.add(shapeMesh);
    });
  });
  
  // SVG coordinate system fix (Y flipped)
  group.scale.y = -1;
  group.position.set(-50, 50, 0);
  scene.add(group);
  
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    group.rotation.y = clock.getElapsedTime() * 0.3;
    renderer.render(scene, camera);
  }
  
  animate();
  
  return () => { cancelAnimationFrame(animId); renderer.dispose(); };
}
```

---

## 75.2 SVG Extrusion (Flat to 3D)

```javascript
// src/demos/svgExtrude.js
import * as THREE from 'three';
import { SVGLoader } from 'three/examples/jsm/loaders/SVGLoader.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

export function createSVGExtrudeDemo(container) {
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#0a0a1a');
  
  const camera = new THREE.PerspectiveCamera(45, container.clientWidth / container.clientHeight, 0.1, 1000);
  camera.position.set(0, 0, 300);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.shadowMap.enabled = true;
  container.appendChild(renderer.domElement);
  
  const controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  
  // SVG inline data - Thailand outline (simplified)
  const SVG_CONTENT = `
    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 200 300">
      <!-- Simplified star shape as example -->
      <path d="
        M100,20 L120,80 L180,80 L130,120 
        L150,180 L100,145 L50,180 
        L70,120 L20,80 L80,80 Z
      " fill="#4ecdc4"/>
      
      <!-- Inner hole -->
      <path d="
        M100,60 L112,90 L145,90 L120,108
        L130,140 L100,122 L70,140
        L80,108 L55,90 L88,90 Z
      " fill="none"/>
    </svg>
  `;
  
  const loader = new SVGLoader();
  const svgData = loader.parse(SVG_CONTENT);
  
  const group = new THREE.Group();
  
  svgData.paths.forEach((path, i) => {
    const shapes = SVGLoader.createShapes(path);
    
    shapes.forEach(shape => {
      // Extrude 2D shape to 3D
      const extrudeSettings = {
        depth: 20,              // ความลึก
        bevelEnabled: true,
        bevelThickness: 3,
        bevelSize: 2,
        bevelOffset: 0,
        bevelSegments: 8,
        curveSegments: 12,
      };
      
      const extrudedGeo = new THREE.ExtrudeGeometry(shape, extrudeSettings);
      
      const mesh = new THREE.Mesh(
        extrudedGeo,
        new THREE.MeshStandardMaterial({
          color: '#4ecdc4',
          metalness: 0.6,
          roughness: 0.2,
          side: THREE.DoubleSide
        })
      );
      
      group.add(mesh);
    });
  });
  
  // Center and flip Y
  group.scale.set(0.5, -0.5, 0.5);
  const box = new THREE.Box3().setFromObject(group);
  const center = box.getCenter(new THREE.Vector3());
  group.position.sub(center);
  
  scene.add(group);
  
  // Floor
  const floor = new THREE.Mesh(
    new THREE.PlaneGeometry(500, 500),
    new THREE.MeshStandardMaterial({ color: '#111', roughness: 0.8 })
  );
  floor.rotation.x = -Math.PI / 2;
  floor.position.y = -80;
  floor.receiveShadow = true;
  scene.add(floor);
  
  // Lighting
  scene.add(new THREE.AmbientLight(0xffffff, 0.4));
  const sun = new THREE.DirectionalLight(0xffffff, 1.5);
  sun.position.set(100, 200, 100);
  sun.castShadow = true;
  scene.add(sun);
  
  const fillLight = new THREE.PointLight('#4ecdc4', 1, 300);
  fillLight.position.set(-50, 50, 100);
  scene.add(fillLight);
  
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const t = clock.getElapsedTime();
    
    group.rotation.y = t * 0.3;
    group.rotation.x = Math.sin(t * 0.2) * 0.1;
    
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
    controls.dispose();
    renderer.dispose();
  };
}
```

---

## 75.3 Shape API - สร้าง Shapes จาก Code

```javascript
// src/demos/shapesDemo.js
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

// สร้าง heart shape
function createHeartShape() {
  const shape = new THREE.Shape();
  
  const x = 0, y = 0;
  shape.moveTo(x + 5, y + 5);
  shape.bezierCurveTo(x + 5, y + 5, x + 4, y, x, y);
  shape.bezierCurveTo(x - 6, y, x - 6, y + 7, x - 6, y + 7);
  shape.bezierCurveTo(x - 6, y + 11, x - 3, y + 15.4, x + 5, y + 19);
  shape.bezierCurveTo(x + 12, y + 15.4, x + 16, y + 11, x + 16, y + 7);
  shape.bezierCurveTo(x + 16, y + 7, x + 16, y, x + 10, y);
  shape.bezierCurveTo(x + 7, y, x + 5, y + 5, x + 5, y + 5);
  
  return shape;
}

// สร้าง arrow shape
function createArrowShape() {
  const shape = new THREE.Shape();
  shape.moveTo(0, -2);
  shape.lineTo(4, -2);
  shape.lineTo(4, -4);
  shape.lineTo(8, 0);
  shape.lineTo(4, 4);
  shape.lineTo(4, 2);
  shape.lineTo(0, 2);
  shape.closePath();
  return shape;
}

// สร้าง gear shape
function createGearShape(teeth = 8, innerRadius = 3, outerRadius = 5, holeRadius = 1.5) {
  const shape = new THREE.Shape();
  const angleStep = (Math.PI * 2) / teeth;
  
  for (let i = 0; i < teeth; i++) {
    const angle = i * angleStep;
    const innerAngle = angle + angleStep * 0.2;
    const peakAngle = angle + angleStep * 0.5;
    const innerAngle2 = angle + angleStep * 0.8;
    const nextAngle = angle + angleStep;
    
    const p = (r, a) => ({ x: r * Math.cos(a), y: r * Math.sin(a) });
    
    if (i === 0) shape.moveTo(p(innerRadius, angle).x, p(innerRadius, angle).y);
    else shape.lineTo(p(innerRadius, angle).x, p(innerRadius, angle).y);
    
    shape.lineTo(p(outerRadius, innerAngle).x, p(outerRadius, innerAngle).y);
    shape.lineTo(p(outerRadius, innerAngle2).x, p(outerRadius, innerAngle2).y);
    shape.lineTo(p(innerRadius, nextAngle).x, p(innerRadius, nextAngle).y);
  }
  shape.closePath();
  
  // Hole
  const hole = new THREE.Path();
  hole.absarc(0, 0, holeRadius, 0, Math.PI * 2, true);
  shape.holes.push(hole);
  
  return shape;
}

export function createShapesDemo(container) {
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#0a0a1a');
  
  const camera = new THREE.PerspectiveCamera(45, container.clientWidth / container.clientHeight, 0.1, 200);
  camera.position.set(0, 15, 40);
  camera.lookAt(0, 0, 0);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.shadowMap.enabled = true;
  container.appendChild(renderer.domElement);
  
  const controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  
  const shapes = [
    { shape: createHeartShape(), color: '#ff6b6b', position: [-12, 0, 0], label: 'Heart' },
    { shape: createArrowShape(), color: '#4ecdc4', position: [0, 0, 0], label: 'Arrow' },
    { shape: createGearShape(), color: '#ffd166', position: [12, 0, 0], label: 'Gear' },
  ];
  
  const extrudeSettings = {
    depth: 2,
    bevelEnabled: true,
    bevelThickness: 0.3,
    bevelSize: 0.2,
    bevelSegments: 6,
  };
  
  const meshGroup = new THREE.Group();
  
  shapes.forEach(({ shape, color, position, label }) => {
    // 3D extruded
    const mesh = new THREE.Mesh(
      new THREE.ExtrudeGeometry(shape, extrudeSettings),
      new THREE.MeshStandardMaterial({
        color,
        metalness: 0.5,
        roughness: 0.3,
      })
    );
    
    // Center the shape
    mesh.geometry.computeBoundingBox();
    const center = new THREE.Vector3();
    mesh.geometry.boundingBox.getCenter(center);
    mesh.geometry.translate(-center.x, -center.y, -center.z);
    
    mesh.position.set(...position);
    mesh.castShadow = true;
    meshGroup.add(mesh);
  });
  
  scene.add(meshGroup);
  
  // Floor
  const floor = new THREE.Mesh(
    new THREE.PlaneGeometry(60, 60),
    new THREE.MeshStandardMaterial({ color: '#111', roughness: 0.9 })
  );
  floor.rotation.x = -Math.PI / 2;
  floor.position.y = -8;
  floor.receiveShadow = true;
  scene.add(floor);
  
  // Lighting
  scene.add(new THREE.AmbientLight(0xffffff, 0.4));
  const dirLight = new THREE.DirectionalLight(0xffffff, 1.5);
  dirLight.position.set(20, 30, 20);
  dirLight.castShadow = true;
  scene.add(dirLight);
  
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const t = clock.getElapsedTime();
    
    meshGroup.children.forEach((mesh, i) => {
      mesh.rotation.y = t * 0.5 + (i * Math.PI * 2) / 3;
      mesh.position.y = Math.sin(t * 0.7 + i * 1) * 1;
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
    controls.dispose();
    renderer.dispose();
  };
}
```

---

## 75.4 TubeGeometry - Path to Tube

```javascript
// src/demos/tubeDemo.js
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

export function createTubeDemo(container) {
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#050510');
  
  const camera = new THREE.PerspectiveCamera(60, container.clientWidth / container.clientHeight, 0.1, 200);
  camera.position.set(0, 10, 30);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  container.appendChild(renderer.domElement);
  
  const controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  
  // DNA double helix
  function createDNAStrand(offset, color) {
    const points = [];
    const TURNS = 4;
    const HEIGHT = 20;
    const RADIUS = 3;
    
    for (let i = 0; i <= 100; i++) {
      const t = i / 100;
      const angle = t * TURNS * Math.PI * 2 + offset;
      points.push(new THREE.Vector3(
        Math.cos(angle) * RADIUS,
        t * HEIGHT - HEIGHT / 2,
        Math.sin(angle) * RADIUS
      ));
    }
    
    const curve = new THREE.CatmullRomCurve3(points);
    const tubeGeo = new THREE.TubeGeometry(curve, 200, 0.2, 8, false);
    
    return new THREE.Mesh(
      tubeGeo,
      new THREE.MeshStandardMaterial({
        color,
        metalness: 0.7,
        roughness: 0.2,
        emissive: color,
        emissiveIntensity: 0.2,
      })
    );
  }
  
  const strand1 = createDNAStrand(0, '#4ecdc4');
  const strand2 = createDNAStrand(Math.PI, '#ff6b6b');
  scene.add(strand1, strand2);
  
  // DNA rungs (base pairs)
  const TURNS = 4;
  const HEIGHT = 20;
  const RADIUS = 3;
  const RUNG_COUNT = 20;
  
  for (let i = 0; i < RUNG_COUNT; i++) {
    const t = i / RUNG_COUNT;
    const angle = t * TURNS * Math.PI * 2;
    const y = t * HEIGHT - HEIGHT / 2;
    
    const p1 = new THREE.Vector3(Math.cos(angle) * RADIUS, y, Math.sin(angle) * RADIUS);
    const p2 = new THREE.Vector3(Math.cos(angle + Math.PI) * RADIUS, y, Math.sin(angle + Math.PI) * RADIUS);
    
    const dir = p2.clone().sub(p1);
    const length = dir.length();
    
    const rung = new THREE.Mesh(
      new THREE.CylinderGeometry(0.1, 0.1, length, 8),
      new THREE.MeshStandardMaterial({ color: '#ffd166', metalness: 0.5, roughness: 0.3 })
    );
    
    rung.position.copy(p1).add(dir.multiplyScalar(0.5));
    rung.lookAt(p2);
    rung.rotateX(Math.PI / 2);
    scene.add(rung);
  }
  
  // === Knot Tube ===
  class TrefoilKnot extends THREE.Curve {
    getPoint(t) {
      const T = t * Math.PI * 2;
      return new THREE.Vector3(
        (2 + Math.cos(3 * T)) * Math.cos(2 * T),
        (2 + Math.cos(3 * T)) * Math.sin(2 * T),
        Math.sin(3 * T)
      ).multiplyScalar(3);
    }
  }
  
  const knotCurve = new TrefoilKnot();
  const knotGeo = new THREE.TubeGeometry(knotCurve, 200, 0.4, 16, true);
  
  // Rainbow material
  const knotColors = new Float32Array(knotGeo.attributes.position.count * 3);
  for (let i = 0; i < knotGeo.attributes.position.count; i++) {
    const color = new THREE.Color().setHSL(i / knotGeo.attributes.position.count, 1, 0.5);
    knotColors[i * 3] = color.r;
    knotColors[i * 3 + 1] = color.g;
    knotColors[i * 3 + 2] = color.b;
  }
  knotGeo.setAttribute('color', new THREE.BufferAttribute(knotColors, 3));
  
  const knotMesh = new THREE.Mesh(
    knotGeo,
    new THREE.MeshStandardMaterial({
      vertexColors: true,
      metalness: 0.5,
      roughness: 0.3,
    })
  );
  knotMesh.position.set(15, 0, 0);
  scene.add(knotMesh);
  
  // Lighting
  scene.add(new THREE.AmbientLight(0xffffff, 0.5));
  const light = new THREE.PointLight(0xffffff, 2, 50);
  light.position.set(10, 10, 10);
  scene.add(light);
  const light2 = new THREE.PointLight('#4ecdc4', 1, 30);
  light2.position.set(-10, -5, 5);
  scene.add(light2);
  
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const t = clock.getElapsedTime();
    
    // Rotate DNA
    strand1.rotation.y = t * 0.3;
    strand2.rotation.y = t * 0.3;
    
    // Rotate knot
    knotMesh.rotation.x = t * 0.3;
    knotMesh.rotation.y = t * 0.2;
    
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
    controls.dispose();
    renderer.dispose();
  };
}
```

---

## 75.5 Logo to 3D (SVGLoader + ExtrudeGeometry)

```javascript
// src/demos/logo3D.js
import * as THREE from 'three';
import { SVGLoader } from 'three/examples/jsm/loaders/SVGLoader.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

export function createLogo3DDemo(container) {
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#050510');
  
  const camera = new THREE.PerspectiveCamera(45, container.clientWidth / container.clientHeight, 0.1, 1000);
  camera.position.set(0, 0, 300);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.shadowMap.enabled = true;
  container.appendChild(renderer.domElement);
  
  const controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.autoRotate = true;
  controls.autoRotateSpeed = 1;
  
  // Three.js logo SVG (simplified)
  const LOGO_SVG = `
    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 100 100">
      <!-- T shape -->
      <rect x="10" y="10" width="80" height="20" rx="5"/>
      <rect x="40" y="30" width="20" height="60" rx="5"/>
    </svg>
  `;
  
  const loader = new SVGLoader();
  const svgData = loader.parse(LOGO_SVG);
  
  const group = new THREE.Group();
  
  const MATERIALS = [
    new THREE.MeshStandardMaterial({ color: '#4ecdc4', metalness: 0.8, roughness: 0.1 }),
    new THREE.MeshStandardMaterial({ color: '#2a9d8f', metalness: 0.6, roughness: 0.3 }),
  ];
  
  svgData.paths.forEach((path, i) => {
    const shapes = SVGLoader.createShapes(path);
    
    shapes.forEach(shape => {
      const extrudedGeo = new THREE.ExtrudeGeometry(shape, {
        depth: 15,
        bevelEnabled: true,
        bevelThickness: 2,
        bevelSize: 1.5,
        bevelSegments: 8,
      });
      
      const mesh = new THREE.Mesh(extrudedGeo, MATERIALS[i % MATERIALS.length]);
      mesh.castShadow = true;
      group.add(mesh);
    });
  });
  
  // Center and scale
  group.scale.set(1, -1, 1);
  const box = new THREE.Box3().setFromObject(group);
  const center = box.getCenter(new THREE.Vector3());
  group.position.sub(center);
  
  scene.add(group);
  
  // Platform
  const platform = new THREE.Mesh(
    new THREE.CylinderGeometry(80, 80, 5, 64),
    new THREE.MeshStandardMaterial({ color: '#1a1a3a', metalness: 0.5, roughness: 0.5 })
  );
  platform.position.y = -60;
  platform.receiveShadow = true;
  scene.add(platform);
  
  // Lighting
  scene.add(new THREE.AmbientLight(0xffffff, 0.3));
  
  const keyLight = new THREE.DirectionalLight(0xffffff, 2);
  keyLight.position.set(100, 200, 100);
  keyLight.castShadow = true;
  keyLight.shadow.mapSize.width = 2048;
  keyLight.shadow.mapSize.height = 2048;
  scene.add(keyLight);
  
  const fillLight = new THREE.PointLight('#4ecdc4', 1, 500);
  fillLight.position.set(-100, 50, 100);
  scene.add(fillLight);
  
  const rimLight = new THREE.PointLight('#ff6b6b', 0.5, 400);
  rimLight.position.set(0, -50, -150);
  scene.add(rimLight);
  
  // Intro animation
  group.scale.set(0, 0, 0);
  const startTime = Date.now();
  
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    
    // Scale in
    const introProgress = Math.min((Date.now() - startTime) / 1000, 1);
    const introEase = 1 - Math.pow(1 - introProgress, 3);
    const scaleTarget = introEase;
    group.scale.set(scaleTarget, -scaleTarget, scaleTarget);
    
    // Hover
    group.position.y = Math.sin(elapsed * 0.5) * 3;
    
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
    controls.dispose();
    renderer.dispose();
  };
}
```

---

## 75.6 LatheGeometry สำหรับ Rotational Objects

```javascript
// LatheGeometry สร้าง 3D object จาก 2D profile ที่ revolve รอบแกน Y
function createLatheDemoObjects(scene) {
  // Wine glass profile
  const winePoints = [];
  for (let i = 0; i < 30; i++) {
    const t = i / 29;
    const y = t * 10 - 5;
    let x;
    
    if (t < 0.3) {
      // Base
      x = 2.5 - t * 5;
    } else if (t < 0.5) {
      // Stem
      x = 0.2;
    } else if (t < 0.7) {
      // Transition
      x = 0.2 + (t - 0.5) * 5;
    } else {
      // Bowl
      x = 1.2 + Math.sin((t - 0.7) / 0.3 * Math.PI / 2) * 1.5;
    }
    
    winePoints.push(new THREE.Vector2(Math.abs(x), y));
  }
  
  const wineGlass = new THREE.Mesh(
    new THREE.LatheGeometry(winePoints, 32),
    new THREE.MeshPhysicalMaterial({
      color: '#aaffee',
      metalness: 0,
      roughness: 0,
      transmission: 0.9,
      thickness: 0.5,
      transparent: true,
      opacity: 0.3,
    })
  );
  wineGlass.position.x = -8;
  scene.add(wineGlass);
  
  // Vase profile
  const vasePoints = [];
  for (let i = 0; i < 25; i++) {
    const t = i / 24;
    const y = t * 8;
    const x = 2 * Math.sin(t * Math.PI) + 0.5 + (1 - t) * 0.5;
    vasePoints.push(new THREE.Vector2(x, y));
  }
  
  const vase = new THREE.Mesh(
    new THREE.LatheGeometry(vasePoints, 64),
    new THREE.MeshStandardMaterial({
      color: '#c88b3a',
      metalness: 0.3,
      roughness: 0.7,
    })
  );
  vase.position.x = 0;
  scene.add(vase);
  
  // Bowl profile
  const bowlPoints = [];
  for (let i = 0; i < 20; i++) {
    const t = i / 19;
    const angle = t * Math.PI / 2;
    bowlPoints.push(new THREE.Vector2(
      Math.sin(angle) * 4,
      -Math.cos(angle) * 3 + 2
    ));
  }
  
  const bowl = new THREE.Mesh(
    new THREE.LatheGeometry(bowlPoints, 48),
    new THREE.MeshStandardMaterial({
      color: '#4ecdc4',
      metalness: 0.6,
      roughness: 0.2,
    })
  );
  bowl.position.x = 8;
  scene.add(bowl);
  
  return { wineGlass, vase, bowl };
}

export function createLatheDemo(container) {
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#0a0a1a');
  
  const camera = new THREE.PerspectiveCamera(45, container.clientWidth / container.clientHeight, 0.1, 100);
  camera.position.set(0, 5, 25);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  container.appendChild(renderer.domElement);
  
  const controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  
  const objects = createLatheDemoObjects(scene);
  
  // Floor
  const floor = new THREE.Mesh(
    new THREE.PlaneGeometry(40, 40),
    new THREE.MeshStandardMaterial({ color: '#111', roughness: 0.8 })
  );
  floor.rotation.x = -Math.PI / 2;
  floor.position.y = -1;
  scene.add(floor);
  
  // Lighting
  scene.add(new THREE.AmbientLight(0xffffff, 0.4));
  const dirLight = new THREE.DirectionalLight(0xffffff, 1.5);
  dirLight.position.set(10, 20, 10);
  scene.add(dirLight);
  const fillLight = new THREE.PointLight('#4488ff', 1, 30);
  fillLight.position.set(-10, 5, 5);
  scene.add(fillLight);
  
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const t = clock.getElapsedTime();
    
    Object.values(objects).forEach((obj, i) => {
      obj.rotation.y = t * 0.3 + (i * Math.PI * 2) / 3;
    });
    
    controls.update();
    renderer.render(scene, camera);
  }
  
  animate();
  
  return () => {
    cancelAnimationFrame(animId);
    controls.dispose();
    renderer.dispose();
  };
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: 3D Country Map
สร้าง map ที่:
- Load SVG country outline
- Extrude เป็น 3D
- Color by region
- Hover for info

### แบบฝึกหัดที่ 2: Corporate Logo 3D
แปลง logo เป็น 3D:
- Multiple colors
- Metallic finish
- Spotlight showcase
- Rotation animation

### แบบฝึกหัดที่ 3: Shape Editor
สร้าง interactive editor:
- Click เพื่อเพิ่ม path points
- Real-time 3D preview
- Extrude depth slider
- Export as GLB

### แบบฝึกหัดที่ 4: Architectural Profile
สร้าง building profile:
- LatheGeometry สำหรับ columns
- ExtrudeGeometry สำหรับ facades
- Combine เป็น building
- Day/night mode

---

## สรุป

ในบทนี้เราเรียนรู้:
1. **SVGLoader** - Load SVG paths
2. **ExtrudeGeometry** - 2D shapes to 3D
3. **Shape API** - สร้าง shapes จาก code
4. **TubeGeometry** - Path เป็น tube
5. **LatheGeometry** - Profile revolve
6. **Logo to 3D** - Complete workflow

บทถัดไปเราจะเรียน **Loading Optimization**
