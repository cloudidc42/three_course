# Part 80: AR with WebXR

## บทนำ

Augmented Reality (AR) ผ่าน WebXR ช่วยให้ Three.js แสดง 3D objects ทับบนโลกจริงผ่านกล้อง AR รองรับบน Android Chrome และ Safari iOS 16+ บทนี้ครอบคลุม ARButton, hit testing, anchors, image tracking, light estimation, และ AR apps จริง

---

## 80.1 AR Setup พื้นฐาน

```javascript
import * as THREE from 'three';
import { ARButton } from 'three/examples/jsm/webxr/ARButton.js';

export function createARScene(container) {
  const scene = new THREE.Scene();
  
  const camera = new THREE.PerspectiveCamera(70, container.clientWidth / container.clientHeight, 0.01, 20);
  
  const renderer = new THREE.WebGLRenderer({
    antialias: true,
    alpha: true, // ต้อง alpha: true เพื่อให้เห็นกล้อง
  });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.setPixelRatio(window.devicePixelRatio);
  
  // === Enable WebXR AR ===
  renderer.xr.enabled = true;
  
  container.style.position = 'relative';
  container.appendChild(renderer.domElement);
  
  // === AR Button ===
  // ระบุ features ที่ต้องการ
  const arButton = ARButton.createButton(renderer, {
    requiredFeatures: ['hit-test'],
    optionalFeatures: ['dom-overlay', 'light-estimation'],
    domOverlay: { root: container },
  });
  container.appendChild(arButton);
  
  // === Simple Object to Place ===
  const mesh = new THREE.Mesh(
    new THREE.BoxGeometry(0.1, 0.1, 0.1),
    new THREE.MeshStandardMaterial({ color: '#4ecdc4' })
  );
  mesh.visible = false; // ซ่อนจนกว่าจะวาง
  scene.add(mesh);
  
  // Lighting (AR ต้องการ light estimate หรือ simple ambient)
  const ambientLight = new THREE.AmbientLight(0xffffff, 1);
  scene.add(ambientLight);
  const dirLight = new THREE.DirectionalLight(0xffffff, 1);
  dirLight.position.set(1, 1, 1);
  scene.add(dirLight);
  
  // === Hit Test ===
  let hitTestSource = null;
  let hitTestSourceRequested = false;
  
  renderer.xr.addEventListener('sessionstart', async () => {
    const session = renderer.xr.getSession();
    
    // Request hit test source
    const viewerSpace = await session.requestReferenceSpace('viewer');
    hitTestSource = await session.requestHitTestSource({ space: viewerSpace });
  });
  
  renderer.xr.addEventListener('sessionend', () => {
    hitTestSource = null;
    hitTestSourceRequested = false;
  });
  
  // Reticle (crosshair ที่แสดงจุดวาง)
  const reticle = new THREE.Mesh(
    new THREE.RingGeometry(0.05, 0.07, 32).rotateX(-Math.PI / 2),
    new THREE.MeshBasicMaterial({ color: 'white', side: THREE.DoubleSide })
  );
  reticle.matrixAutoUpdate = false;
  reticle.visible = false;
  scene.add(reticle);
  
  // Tap to place
  const controller = renderer.xr.getController(0);
  controller.addEventListener('select', () => {
    if (reticle.visible) {
      mesh.visible = true;
      mesh.position.setFromMatrixPosition(reticle.matrix);
      mesh.quaternion.setFromRotationMatrix(reticle.matrix);
    }
  });
  scene.add(controller);
  
  // Animation loop
  renderer.setAnimationLoop((timestamp, frame) => {
    if (frame) {
      const referenceSpace = renderer.xr.getReferenceSpace();
      
      // Hit test
      if (hitTestSource) {
        const hitTestResults = frame.getHitTestResults(hitTestSource);
        
        if (hitTestResults.length > 0) {
          const hit = hitTestResults[0];
          const pose = hit.getPose(referenceSpace);
          
          reticle.visible = true;
          reticle.matrix.fromArray(pose.transform.matrix);
        } else {
          reticle.visible = false;
        }
      }
    }
    
    renderer.render(scene, camera);
  });
  
  window.addEventListener('resize', () => {
    camera.aspect = container.clientWidth / container.clientHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(container.clientWidth, container.clientHeight);
  });
  
  return () => {
    renderer.setAnimationLoop(null);
    renderer.dispose();
    arButton.remove();
  };
}
```

---

## 80.2 AR Furniture Placement

```javascript
// src/demos/arFurniture.js
import * as THREE from 'three';
import { ARButton } from 'three/examples/jsm/webxr/ARButton.js';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

export function createARFurnitureDemo(container) {
  const scene = new THREE.Scene();
  
  const camera = new THREE.PerspectiveCamera(70, container.clientWidth / container.clientHeight, 0.01, 20);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.xr.enabled = true;
  renderer.shadowMap.enabled = true;
  renderer.shadowMap.type = THREE.PCFSoftShadowMap;
  container.style.position = 'relative';
  container.appendChild(renderer.domElement);
  
  // AR Button with hit-test
  const arButton = ARButton.createButton(renderer, {
    requiredFeatures: ['hit-test'],
    optionalFeatures: ['light-estimation', 'dom-overlay'],
    domOverlay: { root: container },
  });
  container.appendChild(arButton);
  
  // Reticle
  const reticle = new THREE.Mesh(
    new THREE.RingGeometry(0.1, 0.13, 32).rotateX(-Math.PI / 2),
    new THREE.MeshBasicMaterial({ color: '#00ff88', side: THREE.DoubleSide, transparent: true, opacity: 0.8 })
  );
  reticle.matrixAutoUpdate = false;
  reticle.visible = false;
  scene.add(reticle);
  
  // Placed objects store
  const placedObjects = [];
  let selectedFurniture = 'chair';
  
  // === Furniture Catalog ===
  const CATALOG = {
    chair: {
      label: '🪑 Chair',
      color: '#8B4513',
      geometry: () => new THREE.BoxGeometry(0.5, 0.8, 0.5),
      scale: 0.3,
    },
    table: {
      label: '🪵 Table',
      color: '#DEB887',
      geometry: () => {
        const g = new THREE.Group();
        const top = new THREE.Mesh(new THREE.BoxGeometry(1, 0.05, 0.6), new THREE.MeshStandardMaterial({ color: '#DEB887' }));
        top.position.y = 0.4;
        g.add(top);
        [[-0.4, -0.2], [0.4, -0.2], [-0.4, 0.2], [0.4, 0.2]].forEach(([x, z]) => {
          const leg = new THREE.Mesh(new THREE.BoxGeometry(0.05, 0.8, 0.05), new THREE.MeshStandardMaterial({ color: '#A0522D' }));
          leg.position.set(x, 0, z);
          g.add(leg);
        });
        return g;
      },
      scale: 0.5,
      isGroup: true,
    },
    lamp: {
      label: '💡 Lamp',
      color: '#FFD700',
      geometry: () => new THREE.ConeGeometry(0.2, 0.4, 16),
      scale: 0.4,
    },
    plant: {
      label: '🌿 Plant',
      color: '#228B22',
      geometry: () => new THREE.SphereGeometry(0.3, 16, 16),
      scale: 0.3,
    },
  };
  
  // === UI Catalog ===
  const uiDiv = document.createElement('div');
  uiDiv.style.cssText = `
    position: absolute; bottom: 80px; left: 50%;
    transform: translateX(-50%);
    display: flex; gap: 10px;
    z-index: 10;
    background: rgba(0,0,0,0.5);
    padding: 10px 20px; border-radius: 30px;
  `;
  
  Object.entries(CATALOG).forEach(([key, item]) => {
    const btn = document.createElement('button');
    btn.textContent = item.label;
    btn.style.cssText = `
      padding: 8px 15px;
      background: ${key === selectedFurniture ? 'rgba(78,205,196,0.4)' : 'rgba(255,255,255,0.1)'};
      border: 1px solid ${key === selectedFurniture ? '#4ecdc4' : 'rgba(255,255,255,0.3)'};
      color: white; border-radius: 20px; cursor: pointer;
      font-size: 0.85em; transition: all 0.2s;
    `;
    btn.addEventListener('click', () => {
      selectedFurniture = key;
      // Update active button style
      uiDiv.querySelectorAll('button').forEach(b => {
        b.style.background = 'rgba(255,255,255,0.1)';
        b.style.borderColor = 'rgba(255,255,255,0.3)';
      });
      btn.style.background = 'rgba(78,205,196,0.4)';
      btn.style.borderColor = '#4ecdc4';
    });
    uiDiv.appendChild(btn);
  });
  
  container.appendChild(uiDiv);
  
  // Clear button
  const clearBtn = document.createElement('button');
  clearBtn.textContent = '🗑️ Clear';
  clearBtn.style.cssText = `
    position: absolute; bottom: 80px; right: 20px;
    z-index: 10;
    background: rgba(255,100,100,0.3); border: 1px solid #ff6b6b;
    color: white; padding: 8px 15px; border-radius: 20px; cursor: pointer;
  `;
  clearBtn.addEventListener('click', () => {
    placedObjects.forEach(obj => scene.remove(obj));
    placedObjects.length = 0;
  });
  container.appendChild(clearBtn);
  
  // Place object
  const controller = renderer.xr.getController(0);
  
  controller.addEventListener('select', () => {
    if (!reticle.visible) return;
    
    const catalog = CATALOG[selectedFurniture];
    let obj;
    
    if (catalog.isGroup) {
      obj = catalog.geometry();
      obj.scale.setScalar(catalog.scale);
    } else {
      obj = new THREE.Mesh(
        catalog.geometry(),
        new THREE.MeshStandardMaterial({ color: catalog.color, roughness: 0.7, metalness: 0.1 })
      );
      obj.scale.setScalar(catalog.scale);
      obj.castShadow = true;
      obj.receiveShadow = true;
    }
    
    obj.position.setFromMatrixPosition(reticle.matrix);
    obj.quaternion.setFromRotationMatrix(reticle.matrix);
    
    scene.add(obj);
    placedObjects.push(obj);
  });
  
  scene.add(controller);
  
  // Lighting
  scene.add(new THREE.AmbientLight(0xffffff, 1));
  const dirLight = new THREE.DirectionalLight(0xffffff, 1);
  dirLight.position.set(1, 3, 1);
  dirLight.castShadow = true;
  scene.add(dirLight);
  
  // Hit test setup
  let hitTestSource = null;
  
  renderer.xr.addEventListener('sessionstart', async () => {
    const session = renderer.xr.getSession();
    const viewerSpace = await session.requestReferenceSpace('viewer');
    hitTestSource = await session.requestHitTestSource({ space: viewerSpace });
  });
  
  renderer.xr.addEventListener('sessionend', () => {
    hitTestSource = null;
  });
  
  // Animation
  renderer.setAnimationLoop((timestamp, frame) => {
    if (frame && hitTestSource) {
      const referenceSpace = renderer.xr.getReferenceSpace();
      const results = frame.getHitTestResults(hitTestSource);
      
      if (results.length > 0) {
        const pose = results[0].getPose(referenceSpace);
        reticle.visible = true;
        reticle.matrix.fromArray(pose.transform.matrix);
        
        // Pulse reticle
        const scale = 1 + Math.sin(Date.now() * 0.005) * 0.1;
        reticle.scale.set(scale, scale, scale);
      } else {
        reticle.visible = false;
      }
    }
    
    renderer.render(scene, camera);
  });
  
  window.addEventListener('resize', () => {
    camera.aspect = container.clientWidth / container.clientHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(container.clientWidth, container.clientHeight);
  });
  
  return () => {
    renderer.setAnimationLoop(null);
    renderer.dispose();
    arButton.remove();
    uiDiv.remove();
    clearBtn.remove();
  };
}
```

---

## 80.3 AR Measurement Tool

```javascript
// src/demos/arMeasure.js
import * as THREE from 'three';
import { ARButton } from 'three/examples/jsm/webxr/ARButton.js';

export function createARMeasureDemo(container) {
  const scene = new THREE.Scene();
  
  const camera = new THREE.PerspectiveCamera(70, container.clientWidth / container.clientHeight, 0.01, 20);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.xr.enabled = true;
  container.style.position = 'relative';
  container.appendChild(renderer.domElement);
  
  const arButton = ARButton.createButton(renderer, {
    requiredFeatures: ['hit-test'],
    domOverlay: { root: container },
  });
  container.appendChild(arButton);
  
  // Reticle
  const reticle = new THREE.Mesh(
    new THREE.RingGeometry(0.05, 0.07, 32).rotateX(-Math.PI / 2),
    new THREE.MeshBasicMaterial({ color: '#ffff00', side: THREE.DoubleSide })
  );
  reticle.matrixAutoUpdate = false;
  reticle.visible = false;
  scene.add(reticle);
  
  // Measurement state
  let points = [];
  let measurements = [];
  
  function placeMeasurementPoint() {
    if (!reticle.visible) return;
    
    const pos = new THREE.Vector3();
    pos.setFromMatrixPosition(reticle.matrix);
    
    // Point marker
    const marker = new THREE.Mesh(
      new THREE.SphereGeometry(0.02, 16, 16),
      new THREE.MeshBasicMaterial({ color: '#ff0000' })
    );
    marker.position.copy(pos);
    scene.add(marker);
    
    points.push({ pos, marker });
    
    if (points.length >= 2) {
      // Draw measurement line
      const p1 = points[points.length - 2].pos;
      const p2 = points[points.length - 1].pos;
      
      const lineGeo = new THREE.BufferGeometry().setFromPoints([p1, p2]);
      const line = new THREE.Line(lineGeo, new THREE.LineBasicMaterial({ color: '#ffff00', linewidth: 2 }));
      scene.add(line);
      
      // Calculate distance
      const distance = p1.distanceTo(p2);
      const distanceCm = (distance * 100).toFixed(1);
      
      // Label
      const midPoint = new THREE.Vector3().addVectors(p1, p2).multiplyScalar(0.5);
      midPoint.y += 0.05;
      
      const labelCanvas = document.createElement('canvas');
      labelCanvas.width = 256;
      labelCanvas.height = 64;
      const ctx = labelCanvas.getContext('2d');
      
      ctx.fillStyle = 'rgba(0,0,0,0.8)';
      ctx.roundRect(4, 4, 248, 56, 10);
      ctx.fill();
      
      ctx.fillStyle = '#ffff00';
      ctx.font = 'bold 24px monospace';
      ctx.textAlign = 'center';
      ctx.fillText(`${distanceCm} cm`, 128, 40);
      
      const labelTex = new THREE.CanvasTexture(labelCanvas);
      const label = new THREE.Mesh(
        new THREE.PlaneGeometry(0.2, 0.05),
        new THREE.MeshBasicMaterial({ map: labelTex, transparent: true, depthTest: false })
      );
      label.position.copy(midPoint);
      scene.add(label);
      
      measurements.push({ line, label });
    }
  }
  
  // Control buttons
  const controlDiv = document.createElement('div');
  controlDiv.style.cssText = `
    position: absolute; bottom: 80px; left: 50%;
    transform: translateX(-50%);
    display: flex; gap: 10px; z-index: 10;
  `;
  
  const addBtn = document.createElement('button');
  addBtn.textContent = '📍 Add Point';
  addBtn.style.cssText = `
    padding: 10px 20px;
    background: rgba(255,255,0,0.3); border: 1px solid #ffff00;
    color: white; border-radius: 20px; cursor: pointer;
  `;
  addBtn.addEventListener('click', placeMeasurementPoint);
  
  const clearBtn = document.createElement('button');
  clearBtn.textContent = '🗑️ Clear';
  clearBtn.style.cssText = `
    padding: 10px 20px;
    background: rgba(255,100,100,0.3); border: 1px solid #ff6b6b;
    color: white; border-radius: 20px; cursor: pointer;
  `;
  clearBtn.addEventListener('click', () => {
    points.forEach(p => scene.remove(p.marker));
    measurements.forEach(m => { scene.remove(m.line); scene.remove(m.label); });
    points = [];
    measurements = [];
  });
  
  controlDiv.appendChild(addBtn);
  controlDiv.appendChild(clearBtn);
  container.appendChild(controlDiv);
  
  // Controller tap = add point
  const controller = renderer.xr.getController(0);
  controller.addEventListener('select', placeMeasurementPoint);
  scene.add(controller);
  
  // Lighting
  scene.add(new THREE.AmbientLight(0xffffff, 1));
  
  // Hit test
  let hitTestSource = null;
  
  renderer.xr.addEventListener('sessionstart', async () => {
    const session = renderer.xr.getSession();
    const viewerSpace = await session.requestReferenceSpace('viewer');
    hitTestSource = await session.requestHitTestSource({ space: viewerSpace });
  });
  
  renderer.xr.addEventListener('sessionend', () => {
    hitTestSource = null;
  });
  
  // Make labels face camera
  renderer.setAnimationLoop((timestamp, frame) => {
    if (frame && hitTestSource) {
      const referenceSpace = renderer.xr.getReferenceSpace();
      const results = frame.getHitTestResults(hitTestSource);
      
      if (results.length > 0) {
        reticle.visible = true;
        reticle.matrix.fromArray(results[0].getPose(referenceSpace).transform.matrix);
      } else {
        reticle.visible = false;
      }
    }
    
    // Labels face camera
    measurements.forEach(m => {
      if (m.label) m.label.lookAt(camera.position);
    });
    
    renderer.render(scene, camera);
  });
  
  window.addEventListener('resize', () => {
    camera.aspect = container.clientWidth / container.clientHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(container.clientWidth, container.clientHeight);
  });
  
  return () => {
    renderer.setAnimationLoop(null);
    renderer.dispose();
    arButton.remove();
    controlDiv.remove();
  };
}
```

---

## 80.4 AR Light Estimation

```javascript
// Light estimation ปรับ lighting ให้เข้ากับสภาพแสงจริง
async function setupLightEstimation(renderer, scene, ambientLight, dirLight) {
  const session = renderer.xr.getSession();
  
  // Check support
  if (!session.requestLightProbe) {
    console.log('Light estimation not supported');
    return;
  }
  
  try {
    const lightProbe = await session.requestLightProbe();
    
    renderer.setAnimationLoop((timestamp, frame) => {
      if (!frame) return;
      
      // Get lighting estimate
      const estimate = frame.getLightEstimate(lightProbe);
      
      if (estimate) {
        // Primary light direction
        const direction = estimate.primaryLightDirection;
        dirLight.position.set(
          -direction.x,
          -direction.y,
          -direction.z
        );
        
        // Primary light intensity
        const intensity = estimate.primaryLightIntensity;
        dirLight.intensity = (intensity.x + intensity.y + intensity.z) / 3;
        
        // Ambient spherical harmonics
        const sh = estimate.sphericalHarmonicsCoefficients;
        if (sh && sh.length >= 9) {
          // Simplified: use DC component (first 3 = red, green, blue)
          ambientLight.color.setRGB(
            Math.max(0, sh[0]) * 0.5,
            Math.max(0, sh[1]) * 0.5,
            Math.max(0, sh[2]) * 0.5
          );
        }
      }
    });
  } catch (e) {
    console.log('Light estimation error:', e);
  }
}
```

---

## 80.5 AR Image Tracking

```javascript
// src/demos/arImageTracking.js
import * as THREE from 'three';
import { ARButton } from 'three/examples/jsm/webxr/ARButton.js';

export async function createARImageTrackingDemo(container) {
  const scene = new THREE.Scene();
  const camera = new THREE.PerspectiveCamera(70, container.clientWidth / container.clientHeight, 0.01, 20);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.xr.enabled = true;
  container.style.position = 'relative';
  container.appendChild(renderer.domElement);
  
  // === Image Targets ===
  // ต้องโหลด image ที่จะ track ก่อน
  async function loadImageBitmap(url) {
    const response = await fetch(url);
    const blob = await response.blob();
    return createImageBitmap(blob);
  }
  
  // สร้าง target image สมมติ (ใน production ใช้ real image)
  const targetCanvas = document.createElement('canvas');
  targetCanvas.width = 256;
  targetCanvas.height = 256;
  const tCtx = targetCanvas.getContext('2d');
  
  // สร้าง distinctive QR-like pattern
  tCtx.fillStyle = 'white';
  tCtx.fillRect(0, 0, 256, 256);
  tCtx.fillStyle = 'black';
  
  // Corners
  [[0,0],[192,0],[0,192],[192,192]].forEach(([x,y]) => {
    tCtx.fillRect(x + 16, y + 16, 48, 48);
    tCtx.fillStyle = 'white';
    tCtx.fillRect(x + 24, y + 24, 32, 32);
    tCtx.fillStyle = 'black';
    tCtx.fillRect(x + 32, y + 32, 16, 16);
  });
  
  // Center cross
  tCtx.fillRect(112, 64, 32, 128);
  tCtx.fillRect(64, 112, 128, 32);
  
  const targetBitmap = await createImageBitmap(targetCanvas);
  
  // AR Button with image tracking
  const arButton = ARButton.createButton(renderer, {
    requiredFeatures: ['image-tracking'],
    trackedImages: [
      {
        image: targetBitmap,
        widthInMeters: 0.2, // 20cm physical size
      }
    ],
  });
  container.appendChild(arButton);
  
  // Objects to show on tracked image
  const trackedObjects = new Map();
  
  function createTrackedContent(id) {
    const group = new THREE.Group();
    
    // Info panel above image
    const panelCanvas = document.createElement('canvas');
    panelCanvas.width = 512;
    panelCanvas.height = 256;
    const pCtx = panelCanvas.getContext('2d');
    
    pCtx.fillStyle = 'rgba(10,10,30,0.9)';
    pCtx.roundRect(8, 8, 496, 240, 15);
    pCtx.fill();
    pCtx.strokeStyle = '#4ecdc4';
    pCtx.lineWidth = 3;
    pCtx.roundRect(8, 8, 496, 240, 15);
    pCtx.stroke();
    
    pCtx.fillStyle = '#4ecdc4';
    pCtx.font = 'bold 28px monospace';
    pCtx.textAlign = 'center';
    pCtx.fillText('AR Content Active!', 256, 80);
    
    pCtx.fillStyle = 'white';
    pCtx.font = '18px monospace';
    pCtx.fillText(`Image ID: ${id}`, 256, 130);
    pCtx.fillStyle = '#888';
    pCtx.font = '14px monospace';
    pCtx.fillText('Three.js AR Image Tracking', 256, 175);
    
    const panelTex = new THREE.CanvasTexture(panelCanvas);
    const panel = new THREE.Mesh(
      new THREE.PlaneGeometry(0.3, 0.15),
      new THREE.MeshBasicMaterial({ map: panelTex, transparent: true })
    );
    panel.position.set(0, 0.15, 0);
    panel.rotation.x = -0.3;
    group.add(panel);
    
    // 3D spinning object above image
    const obj = new THREE.Mesh(
      new THREE.TorusKnotGeometry(0.04, 0.015, 64, 8),
      new THREE.MeshStandardMaterial({ color: '#4ecdc4', metalness: 0.8, roughness: 0.2, emissive: '#4ecdc4', emissiveIntensity: 0.3 })
    );
    obj.position.set(0, 0.08, 0);
    group.add(obj);
    
    group.userData.spinObj = obj;
    group.userData.panel = panel;
    group.userData.panelTex = panelTex;
    
    scene.add(group);
    trackedObjects.set(id, group);
    
    return group;
  }
  
  // Lighting
  scene.add(new THREE.AmbientLight(0xffffff, 1));
  scene.add(new THREE.DirectionalLight(0xffffff, 1).position.set(1, 3, 1));
  
  const clock = new THREE.Clock();
  
  renderer.setAnimationLoop((timestamp, frame) => {
    const t = clock.getElapsedTime();
    
    if (frame) {
      const results = frame.getImageTrackingResults?.() || [];
      
      results.forEach(result => {
        const { imageIndex, trackingState, measuredWidthInMeters } = result;
        
        // Get or create tracked content
        if (!trackedObjects.has(imageIndex)) {
          createTrackedContent(imageIndex);
        }
        
        const group = trackedObjects.get(imageIndex);
        
        if (trackingState === 'tracked' || trackingState === 'emulated') {
          const pose = result.getPose(renderer.xr.getReferenceSpace());
          
          if (pose) {
            group.visible = true;
            group.matrix.fromArray(pose.transform.matrix);
            group.matrix.decompose(group.position, group.quaternion, group.scale);
          }
        } else {
          group.visible = false;
        }
        
        // Animate spin object
        if (group.userData.spinObj) {
          group.userData.spinObj.rotation.x = t * 2;
          group.userData.spinObj.rotation.y = t * 1.5;
        }
      });
    }
    
    renderer.render(scene, camera);
  });
  
  window.addEventListener('resize', () => {
    camera.aspect = container.clientWidth / container.clientHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(container.clientWidth, container.clientHeight);
  });
  
  return () => {
    renderer.setAnimationLoop(null);
    renderer.dispose();
    arButton.remove();
  };
}
```

---

## 80.6 AR Quick Tips

```javascript
// AR Best Practices

// 1. alpha: true เสมอสำหรับ AR
const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });

// 2. Transparent scene background
scene.background = null; // ต้อง null ไม่ใช่ color

// 3. ใช้ setAnimationLoop แทน requestAnimationFrame
renderer.setAnimationLoop(animate);

// 4. อ่าน pose จาก frame เสมอ (ไม่ใช่จาก object โดยตรง)
const pose = frame.getPose(referenceSpace);

// 5. Hit test ต้องการ HTTPS
// localhost ใช้ได้ แต่ production ต้องการ HTTPS

// 6. ลด draw calls (instancing)
const instancedMesh = new THREE.InstancedMesh(geo, mat, count);

// 7. ใช้ shadow catcher สำหรับ realistic shadows
const shadowCatcher = new THREE.Mesh(
  new THREE.PlaneGeometry(10, 10).rotateX(-Math.PI / 2),
  new THREE.ShadowMaterial({ opacity: 0.3 })
);
shadowCatcher.receiveShadow = true;
scene.add(shadowCatcher);

// 8. Check WebXR support
if (navigator.xr) {
  const supported = await navigator.xr.isSessionSupported('immersive-ar');
  if (supported) {
    // AR available
  }
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: AR Business Card
สร้าง AR business card:
- Image tracking บน business card
- 3D contact info
- Social media links
- Portfolio preview

### แบบฝึกหัดที่ 2: AR Game
สร้าง game ที่:
- Spawn enemies บน surfaces
- Shoot ด้วย tap
- Score system
- Lives และ levels

### แบบฝึกหัดที่ 3: AR Product Viewer
สร้าง product viewer:
- Scale adjustment
- Color variants
- 360-degree view
- Share button

### แบบฝึกหัดที่ 4: AR Navigation
สร้าง indoor navigation:
- Path arrows on floor
- Room labels
- Distance indicators
- Turn-by-turn

---

## สรุปบท

ในบทนี้เราเรียนรู้:
1. **AR Setup** - ARButton, alpha renderer
2. **Hit Testing** - ตรวจจับ surfaces
3. **Furniture Placement** - AR app จริง
4. **Measurement** - วัดระยะในโลกจริง
5. **Light Estimation** - ปรับแสงตาม environment
6. **Image Tracking** - Track printed images

---

## สรุป Course ทั้งหมด (Part 064-080)

เราได้เรียนรู้ครบ Three.js Ecosystem:

**Framework Integration:**
- Part 064-065: React Three Fiber (R3F)
- Part 066: Drei Library
- Part 067: Vue.js + Three.js
- Part 068: TypeScript + Three.js

**Advanced Rendering:**
- Part 069: Render Targets & Off-Screen
- Part 070: GPGPU Computing
- Part 071: Audio System
- Part 072: Video & Canvas Textures
- Part 073: CSS3D/CSS2D Renderers
- Part 074: 3D Text & Typography
- Part 075: SVG & Shape Extrusion

**Optimization:**
- Part 076: Loading Optimization
- Part 077: Worker Threads
- Part 078: Mobile Optimization

**XR Development:**
- Part 079: VR with WebXR
- Part 080: AR with WebXR

**ยินดีด้วย! คุณเรียนจบ Three.js Course แล้ว!** 🎉

Three.js เป็น ecosystem ที่กว้างมาก แนะนำให้:
1. สร้าง projects จริงๆ
2. ดู Three.js examples
3. ร่วม Three.js community
4. ทดลอง shaders ใน Shadertoy
5. ติดตาม update ของ Three.js
