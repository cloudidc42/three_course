# Part 79: VR Development with WebXR

## บทนำ

WebXR API ช่วยให้ Three.js ทำงานกับ VR headsets ได้ (Meta Quest, HTC Vive, Valve Index ฯลฯ) ผ่าน browser โดยไม่ต้องติดตั้ง native app บทนี้ครอบคลุม WebXRManager, VRButton, controllers, locomotion, และ XR UI

---

## 79.1 Setup WebXR

```javascript
import * as THREE from 'three';
import { VRButton } from 'three/examples/jsm/webxr/VRButton.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

export function createVRScene(container) {
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#87CEEB');
  
  const camera = new THREE.PerspectiveCamera(75, container.clientWidth / container.clientHeight, 0.1, 1000);
  camera.position.set(0, 1.6, 3); // Eye height ~1.6m
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.setPixelRatio(window.devicePixelRatio);
  
  // === Enable WebXR ===
  renderer.xr.enabled = true;
  
  container.appendChild(renderer.domElement);
  
  // === VR Button ===
  // ปุ่มนี้จะแสดงเมื่อ browser รองรับ WebXR VR
  const vrButton = VRButton.createButton(renderer);
  vrButton.style.cssText = `
    position: absolute; bottom: 20px; left: 50%;
    transform: translateX(-50%);
    z-index: 10;
  `;
  container.style.position = 'relative';
  container.appendChild(vrButton);
  
  // === Scene Objects ===
  // Floor
  const floor = new THREE.Mesh(
    new THREE.PlaneGeometry(20, 20),
    new THREE.MeshStandardMaterial({ color: '#228B22', roughness: 0.8 })
  );
  floor.rotation.x = -Math.PI / 2;
  scene.add(floor);
  
  // Grid on floor
  const grid = new THREE.GridHelper(20, 20, '#111', '#333');
  scene.add(grid);
  
  // Objects to interact with
  const objects = [];
  const COLORS = ['#ff6b6b', '#4ecdc4', '#ffd166', '#95e1d3', '#c3a6ff'];
  
  for (let i = 0; i < 5; i++) {
    const mesh = new THREE.Mesh(
      new THREE.BoxGeometry(0.3, 0.3, 0.3),
      new THREE.MeshStandardMaterial({ color: COLORS[i] })
    );
    mesh.position.set(
      (i - 2) * 1.2,
      0.15,
      -2
    );
    mesh.castShadow = true;
    scene.add(mesh);
    objects.push(mesh);
  }
  
  // Sphere target
  const sphere = new THREE.Mesh(
    new THREE.SphereGeometry(0.5, 32, 32),
    new THREE.MeshStandardMaterial({ color: '#4ecdc4', metalness: 0.7, roughness: 0.2 })
  );
  sphere.position.set(0, 2, -3);
  scene.add(sphere);
  
  // Lighting
  const sun = new THREE.DirectionalLight(0xffffff, 1.5);
  sun.position.set(10, 15, 10);
  sun.castShadow = true;
  scene.add(sun);
  scene.add(new THREE.AmbientLight(0xffffff, 0.5));
  
  // Fog
  scene.fog = new THREE.Fog('#87CEEB', 20, 50);
  
  // === Non-VR Controls ===
  const controls = new OrbitControls(camera, renderer.domElement);
  controls.target.set(0, 1, 0);
  controls.enableDamping = true;
  
  // === Animation Loop ===
  // ใช้ setAnimationLoop แทน requestAnimationFrame สำหรับ WebXR
  const clock = new THREE.Clock();
  
  renderer.setAnimationLoop((timestamp, frame) => {
    const t = clock.getElapsedTime();
    
    // Rotate objects
    objects.forEach((obj, i) => {
      obj.rotation.y = t + i * 0.5;
      obj.position.y = 0.15 + Math.sin(t + i) * 0.1;
    });
    
    sphere.rotation.y = t * 0.3;
    
    if (!renderer.xr.isPresenting) {
      controls.update();
    }
    
    renderer.render(scene, camera);
  });
  
  window.addEventListener('resize', () => {
    const w = container.clientWidth;
    const h = container.clientHeight;
    camera.aspect = w / h;
    camera.updateProjectionMatrix();
    renderer.setSize(w, h);
  });
  
  return () => {
    renderer.setAnimationLoop(null);
    controls.dispose();
    renderer.dispose();
    vrButton.remove();
  };
}
```

---

## 79.2 XR Controllers

```javascript
// src/demos/vrControllers.js
import * as THREE from 'three';
import { VRButton } from 'three/examples/jsm/webxr/VRButton.js';
import { XRControllerModelFactory } from 'three/examples/jsm/webxr/XRControllerModelFactory.js';

export function createVRControllersDemo(container) {
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#1a1a2e');
  
  const camera = new THREE.PerspectiveCamera(75, container.clientWidth / container.clientHeight, 0.1, 100);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.xr.enabled = true;
  renderer.shadowMap.enabled = true;
  container.appendChild(renderer.domElement);
  
  const vrButton = VRButton.createButton(renderer);
  container.style.position = 'relative';
  container.appendChild(vrButton);
  
  // === Player Group (moves with VR camera rig) ===
  const player = new THREE.Group();
  scene.add(player);
  player.add(camera);
  player.position.set(0, 0, 3);
  
  // === Controllers ===
  const controllerModelFactory = new XRControllerModelFactory();
  
  function setupController(index) {
    // Controller grip (where model attaches)
    const controllerGrip = renderer.xr.getControllerGrip(index);
    const model = controllerModelFactory.createControllerModel(controllerGrip);
    controllerGrip.add(model);
    player.add(controllerGrip);
    
    // Controller (ray origin)
    const controller = renderer.xr.getController(index);
    player.add(controller);
    
    // Ray visual
    const rayGeometry = new THREE.BufferGeometry();
    rayGeometry.setFromPoints([
      new THREE.Vector3(0, 0, 0),
      new THREE.Vector3(0, 0, -1)
    ]);
    
    const ray = new THREE.Line(
      rayGeometry,
      new THREE.LineBasicMaterial({ color: '#4ecdc4', transparent: true, opacity: 0.5 })
    );
    ray.scale.z = 5;
    controller.add(ray);
    
    // Dot at end of ray
    const dot = new THREE.Mesh(
      new THREE.SphereGeometry(0.01, 8, 8),
      new THREE.MeshBasicMaterial({ color: '#4ecdc4' })
    );
    dot.position.z = -5;
    controller.add(dot);
    
    // Controller events
    controller.addEventListener('selectstart', () => {
      controller.userData.selecting = true;
      ray.material.color.set('#ff6b6b');
      
      // Raycast for interaction
      const tempMatrix = new THREE.Matrix4();
      tempMatrix.identity().extractRotation(controller.matrixWorld);
      
      const raycaster = new THREE.Raycaster();
      raycaster.ray.origin.setFromMatrixPosition(controller.matrixWorld);
      raycaster.ray.direction.set(0, 0, -1).applyMatrix4(tempMatrix);
      
      const intersects = raycaster.intersectObjects(interactables);
      if (intersects.length > 0) {
        const obj = intersects[0].object;
        controller.userData.grabbed = obj;
        obj.userData.originalParent = obj.parent;
        controller.attach(obj); // Re-parent to controller
      }
    });
    
    controller.addEventListener('selectend', () => {
      controller.userData.selecting = false;
      ray.material.color.set('#4ecdc4');
      
      // Release grabbed object
      if (controller.userData.grabbed) {
        const obj = controller.userData.grabbed;
        const parent = obj.userData.originalParent || scene;
        parent.attach(obj); // Return to original parent in world space
        delete controller.userData.grabbed;
        delete obj.userData.originalParent;
      }
    });
    
    controller.addEventListener('squeezestart', () => {
      // Squeeze = grab all nearby
      console.log('Squeeze pressed');
    });
    
    return { controller, controllerGrip };
  }
  
  const { controller: ctrl0 } = setupController(0);
  const { controller: ctrl1 } = setupController(1);
  
  // === Interactable Objects ===
  const interactables = [];
  
  for (let i = 0; i < 8; i++) {
    const types = [
      new THREE.BoxGeometry(0.2, 0.2, 0.2),
      new THREE.SphereGeometry(0.1, 16, 16),
      new THREE.ConeGeometry(0.1, 0.3, 16),
    ];
    
    const mesh = new THREE.Mesh(
      types[i % types.length],
      new THREE.MeshStandardMaterial({
        color: new THREE.Color().setHSL(i / 8, 0.8, 0.6),
        metalness: 0.3,
        roughness: 0.5,
      })
    );
    
    mesh.position.set(
      (Math.random() - 0.5) * 3,
      0.5 + Math.random() * 1.5,
      -1.5 - Math.random() * 1.5
    );
    mesh.castShadow = true;
    scene.add(mesh);
    interactables.push(mesh);
  }
  
  // Floor
  const floor = new THREE.Mesh(
    new THREE.PlaneGeometry(10, 10),
    new THREE.MeshStandardMaterial({ color: '#2a2a4a', roughness: 0.9 })
  );
  floor.rotation.x = -Math.PI / 2;
  floor.receiveShadow = true;
  scene.add(floor);
  
  // Walls
  const wallMat = new THREE.MeshStandardMaterial({ color: '#1a1a3a', roughness: 0.8 });
  const walls = [
    { pos: [0, 2, -5], rot: [0, 0, 0], size: [10, 4] },
    { pos: [-5, 2, 0], rot: [0, Math.PI/2, 0], size: [10, 4] },
    { pos: [5, 2, 0], rot: [0, -Math.PI/2, 0], size: [10, 4] },
  ];
  walls.forEach(w => {
    const wall = new THREE.Mesh(new THREE.PlaneGeometry(...w.size), wallMat);
    wall.position.set(...w.pos);
    wall.rotation.set(...w.rot);
    wall.receiveShadow = true;
    scene.add(wall);
  });
  
  // Lighting
  scene.add(new THREE.AmbientLight(0xffffff, 0.4));
  const light = new THREE.DirectionalLight(0xffffff, 1);
  light.position.set(3, 5, 3);
  light.castShadow = true;
  scene.add(light);
  
  // Ceiling lights
  for (let x = -2; x <= 2; x += 2) {
    const pointLight = new THREE.PointLight('#4466ff', 0.5, 5);
    pointLight.position.set(x, 3.5, -2);
    scene.add(pointLight);
  }
  
  // === Locomotion (Teleport) ===
  const teleportMarker = new THREE.Mesh(
    new THREE.RingGeometry(0.15, 0.2, 32),
    new THREE.MeshBasicMaterial({ color: '#4ecdc4', side: THREE.DoubleSide })
  );
  teleportMarker.rotation.x = -Math.PI / 2;
  teleportMarker.visible = false;
  scene.add(teleportMarker);
  
  // === Animation Loop ===
  const clock = new THREE.Clock();
  
  renderer.setAnimationLoop((timestamp, frame) => {
    const t = clock.getElapsedTime();
    
    // Gentle floating of objects
    interactables.forEach((obj, i) => {
      if (!obj.parent !== scene) return; // Don't animate grabbed objects
      obj.rotation.y = t * 0.5 + i * 0.5;
    });
    
    renderer.render(scene, camera);
  });
  
  window.addEventListener('resize', () => {
    const w = container.clientWidth;
    const h = container.clientHeight;
    camera.aspect = w / h;
    camera.updateProjectionMatrix();
    renderer.setSize(w, h);
  });
  
  return () => {
    renderer.setAnimationLoop(null);
    renderer.dispose();
    vrButton.remove();
  };
}
```

---

## 79.3 VR UI Panel

```javascript
// src/demos/vrUI.js
import * as THREE from 'three';
import { VRButton } from 'three/examples/jsm/webxr/VRButton.js';

// สร้าง UI panel ด้วย canvas texture ใน VR
function createVRPanel(text, color = '#4ecdc4') {
  const canvas = document.createElement('canvas');
  canvas.width = 512;
  canvas.height = 256;
  const ctx = canvas.getContext('2d');
  
  // Background
  ctx.fillStyle = 'rgba(10, 10, 30, 0.95)';
  ctx.roundRect(0, 0, 512, 256, 20);
  ctx.fill();
  
  // Border
  ctx.strokeStyle = color;
  ctx.lineWidth = 3;
  ctx.roundRect(5, 5, 502, 246, 17);
  ctx.stroke();
  
  // Text
  ctx.fillStyle = color;
  ctx.font = 'bold 24px monospace';
  ctx.textAlign = 'center';
  ctx.fillText(text, 256, 80);
  
  ctx.fillStyle = 'white';
  ctx.font = '16px monospace';
  ctx.fillText('WebXR VR Panel', 256, 130);
  ctx.fillStyle = '#888';
  ctx.font = '14px monospace';
  ctx.fillText('Interactive in VR space', 256, 165);
  
  const texture = new THREE.CanvasTexture(canvas);
  
  const panel = new THREE.Mesh(
    new THREE.PlaneGeometry(0.5, 0.25),
    new THREE.MeshBasicMaterial({ map: texture, transparent: true })
  );
  
  return { panel, canvas, ctx, texture };
}

export function createVRUIDemo(container) {
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#0a0a1a');
  
  const camera = new THREE.PerspectiveCamera(75, container.clientWidth / container.clientHeight, 0.1, 100);
  camera.position.set(0, 1.6, 0);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.xr.enabled = true;
  container.appendChild(renderer.domElement);
  
  const vrButton = VRButton.createButton(renderer);
  container.style.position = 'relative';
  container.appendChild(vrButton);
  
  // Main VR panel
  const { panel: mainPanel, canvas, ctx, texture } = createVRPanel('MAIN MENU', '#4ecdc4');
  mainPanel.position.set(0, 1.5, -1.5);
  scene.add(mainPanel);
  
  // Smaller info panels
  const panelPositions = [
    { pos: [-0.7, 1.6, -1.2], rot: [0, 0.3, 0], text: 'Settings', color: '#ff6b6b' },
    { pos: [0.7, 1.6, -1.2], rot: [0, -0.3, 0], text: 'Help', color: '#ffd166' },
    { pos: [0, 1.0, -1.5], rot: [0.2, 0, 0], text: 'World', color: '#95e1d3' },
  ];
  
  panelPositions.forEach(({ pos, rot, text, color }) => {
    const { panel } = createVRPanel(text, color);
    panel.position.set(...pos);
    panel.rotation.set(...rot);
    panel.scale.set(0.6, 0.6, 0.6);
    scene.add(panel);
  });
  
  // Floor grid
  const grid = new THREE.GridHelper(10, 20, '#222', '#222');
  scene.add(grid);
  
  // Ambient particles for atmosphere
  const particleCount = 1000;
  const pPositions = new Float32Array(particleCount * 3);
  for (let i = 0; i < particleCount; i++) {
    pPositions[i * 3] = (Math.random() - 0.5) * 10;
    pPositions[i * 3 + 1] = Math.random() * 3;
    pPositions[i * 3 + 2] = (Math.random() - 0.5) * 10;
  }
  const pGeo = new THREE.BufferGeometry();
  pGeo.setAttribute('position', new THREE.BufferAttribute(pPositions, 3));
  scene.add(new THREE.Points(pGeo, new THREE.PointsMaterial({ size: 0.02, color: '#4ecdc4', transparent: true, opacity: 0.5 })));
  
  // Lighting
  scene.add(new THREE.AmbientLight(0xffffff, 0.4));
  const light = new THREE.PointLight('#4ecdc4', 1, 5);
  light.position.set(0, 2, -1);
  scene.add(light);
  
  // Animation - update panel content
  const clock = new THREE.Clock();
  let updateTimer = 0;
  
  renderer.setAnimationLoop((timestamp) => {
    const elapsed = clock.getElapsedTime();
    const delta = clock.getDelta();
    
    updateTimer += delta;
    
    // Update panel clock
    if (updateTimer > 1) {
      updateTimer = 0;
      
      ctx.fillStyle = 'rgba(10, 10, 30, 0.95)';
      ctx.clearRect(0, 110, 512, 100);
      ctx.fillStyle = 'white';
      ctx.font = '20px monospace';
      ctx.textAlign = 'center';
      ctx.fillText(new Date().toLocaleTimeString(), 256, 140);
      texture.needsUpdate = true;
    }
    
    // Floating animation for panels
    mainPanel.position.y = 1.5 + Math.sin(elapsed * 0.5) * 0.02;
    
    renderer.render(scene, camera);
  });
  
  window.addEventListener('resize', () => {
    const w = container.clientWidth;
    const h = container.clientHeight;
    camera.aspect = w / h;
    camera.updateProjectionMatrix();
    renderer.setSize(w, h);
  });
  
  return () => {
    renderer.setAnimationLoop(null);
    texture.dispose();
    renderer.dispose();
    vrButton.remove();
  };
}
```

---

## 79.4 VR Locomotion - Teleportation

```javascript
// src/demos/vrTeleport.js
import * as THREE from 'three';
import { VRButton } from 'three/examples/jsm/webxr/VRButton.js';
import { XRControllerModelFactory } from 'three/examples/jsm/webxr/XRControllerModelFactory.js';

export function createVRTeleportDemo(container) {
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#87ceeb');
  
  const camera = new THREE.PerspectiveCamera(75, container.clientWidth / container.clientHeight, 0.1, 200);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.xr.enabled = true;
  renderer.shadowMap.enabled = true;
  container.appendChild(renderer.domElement);
  
  const vrButton = VRButton.createButton(renderer);
  container.style.position = 'relative';
  container.appendChild(vrButton);
  
  // Player rig
  const player = new THREE.Group();
  scene.add(player);
  player.add(camera);
  player.position.set(0, 0, 5);
  
  // === Teleport System ===
  const TELEPORT_COLOR = '#00ff88';
  const INVALID_COLOR = '#ff4444';
  
  // Arc preview
  const arcPoints = new Array(30).fill(null).map(() => new THREE.Vector3());
  const arcGeometry = new THREE.BufferGeometry();
  arcGeometry.setFromPoints(arcPoints);
  const arcLine = new THREE.Line(
    arcGeometry,
    new THREE.LineBasicMaterial({ color: TELEPORT_COLOR, transparent: true, opacity: 0.8 })
  );
  arcLine.visible = false;
  scene.add(arcLine);
  
  // Landing marker
  const landingMarker = new THREE.Group();
  
  const ring = new THREE.Mesh(
    new THREE.TorusGeometry(0.5, 0.03, 8, 32),
    new THREE.MeshBasicMaterial({ color: TELEPORT_COLOR })
  );
  ring.rotation.x = -Math.PI / 2;
  landingMarker.add(ring);
  
  const innerRing = new THREE.Mesh(
    new THREE.TorusGeometry(0.2, 0.02, 8, 32),
    new THREE.MeshBasicMaterial({ color: TELEPORT_COLOR })
  );
  innerRing.rotation.x = -Math.PI / 2;
  landingMarker.add(innerRing);
  
  landingMarker.visible = false;
  scene.add(landingMarker);
  
  // Controllers
  const modelFactory = new XRControllerModelFactory();
  
  function createController(index) {
    const grip = renderer.xr.getControllerGrip(index);
    grip.add(modelFactory.createControllerModel(grip));
    player.add(grip);
    
    const controller = renderer.xr.getController(index);
    player.add(controller);
    
    let validTarget = null;
    
    controller.addEventListener('selectstart', () => {
      if (validTarget) {
        // Teleport!
        const offset = camera.position.clone();
        offset.y = 0;
        player.position.copy(validTarget).sub(offset);
        
        // Flash effect
        flashOverlay();
      }
    });
    
    return { controller, grip };
  }
  
  const { controller: ctrl0 } = createController(0);
  
  // Flash overlay for teleport
  const overlayCanvas = document.createElement('canvas');
  overlayCanvas.width = 2;
  overlayCanvas.height = 2;
  const overlayCtx = overlayCanvas.getContext('2d');
  overlayCtx.fillStyle = 'white';
  overlayCtx.fillRect(0, 0, 2, 2);
  
  const overlayTex = new THREE.CanvasTexture(overlayCanvas);
  const overlay = new THREE.Mesh(
    new THREE.PlaneGeometry(2, 2),
    new THREE.MeshBasicMaterial({ map: overlayTex, transparent: true, opacity: 0, depthTest: false })
  );
  overlay.position.z = -0.5;
  camera.add(overlay);
  
  let flashProgress = 0;
  let isFlashing = false;
  
  function flashOverlay() {
    flashProgress = 0;
    isFlashing = true;
  }
  
  // === World ===
  // Large terrain
  const terrain = new THREE.Mesh(
    new THREE.PlaneGeometry(50, 50, 64, 64),
    new THREE.MeshStandardMaterial({ color: '#4a7c59', roughness: 0.9 })
  );
  terrain.rotation.x = -Math.PI / 2;
  terrain.receiveShadow = true;
  scene.add(terrain);
  
  // Teleport spots
  const spots = [
    { pos: [0, 0, 0], color: '#4ecdc4' },
    { pos: [-5, 0, -8], color: '#ff6b6b' },
    { pos: [5, 0, -8], color: '#ffd166' },
    { pos: [-10, 0, 0], color: '#95e1d3' },
    { pos: [10, 0, 0], color: '#c3a6ff' },
  ];
  
  spots.forEach(({ pos, color }) => {
    const spot = new THREE.Mesh(
      new THREE.CylinderGeometry(1, 1, 0.01, 32),
      new THREE.MeshBasicMaterial({ color, transparent: true, opacity: 0.4 })
    );
    spot.position.set(...pos);
    scene.add(spot);
    
    // Pillar
    const pillar = new THREE.Mesh(
      new THREE.BoxGeometry(0.2, 2, 0.2),
      new THREE.MeshStandardMaterial({ color, metalness: 0.5, roughness: 0.3 })
    );
    pillar.position.set(pos[0], 1, pos[2]);
    scene.add(pillar);
  });
  
  // Lighting
  const sun = new THREE.DirectionalLight(0xffeedd, 1.5);
  sun.position.set(20, 30, 20);
  sun.castShadow = true;
  scene.add(sun);
  scene.add(new THREE.AmbientLight(0x88aaff, 0.5));
  scene.fog = new THREE.Fog('#87ceeb', 30, 100);
  
  // Raycaster for arc calculation
  const raycaster = new THREE.Raycaster();
  const tempMatrix = new THREE.Matrix4();
  
  function calculateArc(controller) {
    tempMatrix.identity().extractRotation(controller.matrixWorld);
    
    const origin = new THREE.Vector3();
    origin.setFromMatrixPosition(controller.matrixWorld);
    
    const direction = new THREE.Vector3(0, 0, -1).applyMatrix4(tempMatrix);
    
    // Parabolic arc
    const velocity = direction.clone().multiplyScalar(10);
    const gravity = new THREE.Vector3(0, -9.8, 0);
    const dt = 0.1;
    let pos = origin.clone();
    let vel = velocity.clone();
    let valid = false;
    let landingPos = null;
    
    for (let i = 0; i < arcPoints.length; i++) {
      arcPoints[i].copy(pos);
      vel.addScaledVector(gravity, dt);
      pos.addScaledVector(vel, dt);
      
      if (pos.y <= 0) {
        // Hit ground
        arcPoints[i].y = 0;
        valid = true;
        landingPos = arcPoints[i].clone();
        
        for (let j = i + 1; j < arcPoints.length; j++) {
          arcPoints[j].copy(landingPos);
        }
        break;
      }
    }
    
    arcGeometry.setFromPoints(arcPoints);
    arcGeometry.computeBoundingSphere();
    
    return { valid, landingPos };
  }
  
  // Animation
  const clock = new THREE.Clock();
  
  renderer.setAnimationLoop((timestamp, frame) => {
    const elapsed = clock.getElapsedTime();
    const delta = clock.getDelta();
    
    // Check if in VR and squeeze held for teleport arc
    if (renderer.xr.isPresenting) {
      const { valid, landingPos } = calculateArc(ctrl0);
      
      arcLine.visible = true;
      landingMarker.visible = valid;
      
      if (valid && landingPos) {
        landingMarker.position.copy(landingPos);
        landingMarker.position.y = 0.01;
        landingMarker.rotation.y = elapsed;
        arcLine.material.color.set(TELEPORT_COLOR);
      } else {
        arcLine.material.color.set(INVALID_COLOR);
      }
    } else {
      arcLine.visible = false;
      landingMarker.visible = false;
    }
    
    // Flash animation
    if (isFlashing) {
      flashProgress += delta * 4;
      if (flashProgress > 1) {
        isFlashing = false;
        overlay.material.opacity = 0;
      } else {
        overlay.material.opacity = Math.sin(flashProgress * Math.PI);
      }
    }
    
    renderer.render(scene, camera);
  });
  
  window.addEventListener('resize', () => {
    const w = container.clientWidth;
    const h = container.clientHeight;
    camera.aspect = w / h;
    camera.updateProjectionMatrix();
    renderer.setSize(w, h);
  });
  
  return () => {
    renderer.setAnimationLoop(null);
    renderer.dispose();
    vrButton.remove();
  };
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: VR Art Gallery
สร้าง gallery ที่:
- ห้องหลายห้องเชื่อมกัน
- Teleport ระหว่างห้อง
- Paintings on walls
- Controller pointing สำหรับ info

### แบบฝึกหัดที่ 2: VR Physics Sandbox
สร้าง sandbox ที่:
- Grab และ throw objects
- Physics simulation
- Multiple rooms
- Score system

### แบบฝึกหัดที่ 3: VR Dashboard
สร้าง dashboard ที่:
- 3D charts รอบตัว
- Controller point-and-click
- Real-time data
- Spatial audio

### แบบฝึกหัดที่ 4: VR Meditation Space
สร้าง space ที่:
- Particle effects ambient
- Spatial audio (birds, water)
- Comfortable movement
- Breathing guide UI

---

## สรุป

ในบทนี้เราเรียนรู้:
1. **WebXR Setup** - renderer.xr.enabled, VRButton
2. **XR Controllers** - Controller models, events
3. **Object Interaction** - Grab, release ใน VR
4. **VR UI** - Canvas texture panels
5. **Teleportation** - Parabolic arc locomotion
6. **Player Rig** - Camera + controllers group

บทถัดไปเราจะเรียน **AR with WebXR** (บทสุดท้ายของ course!)
