# Part 87: Architectural Visualization

## บทนำ: Archviz กับ Three.js

Architectural Visualization (Archviz) คือการแสดง building หรือ space ในรูปแบบ 3D เพื่อช่วยลูกค้าหรือผู้ออกแบบเข้าใจ design ก่อนสร้างจริง Three.js เหมาะสำหรับ web-based archviz ที่ต้องการ interactivity

---

## 1. การโหลด GLTF Scene ขนาดใหญ่

### GLTF Loader Optimization

```javascript
import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';
import { MeshoptDecoder } from 'three/addons/libs/meshopt_decoder.module.js';
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js';

function createOptimizedLoader(renderer) {
  const gltfLoader = new GLTFLoader();
  
  // DRACO compression (ลดขนาด geometry)
  const dracoLoader = new DRACOLoader();
  dracoLoader.setDecoderPath('/libs/draco/');
  dracoLoader.preload();
  gltfLoader.setDRACOLoader(dracoLoader);
  
  // Meshopt compression (ลดขนาดและเพิ่มความเร็ว)
  gltfLoader.setMeshoptDecoder(MeshoptDecoder);
  
  // KTX2 textures (GPU compressed textures)
  const ktx2Loader = new KTX2Loader();
  ktx2Loader.setTranscoderPath('/libs/basis/');
  ktx2Loader.detectSupport(renderer);
  gltfLoader.setKTX2Loader(ktx2Loader);
  
  return { gltfLoader, dracoLoader, ktx2Loader };
}

// โหลด scene แบบ progressive
async function loadArchScene(url, onProgress) {
  return new Promise((resolve, reject) => {
    gltfLoader.load(
      url,
      (gltf) => {
        const scene = gltf.scene;
        
        // Process materials
        scene.traverse(obj => {
          if (obj.isMesh) {
            // เปิด shadow
            obj.castShadow = true;
            obj.receiveShadow = true;
            
            // Optimize geometry
            if (obj.geometry) {
              obj.geometry.computeBoundingBox();
              obj.geometry.computeBoundingSphere();
            }
            
            // Handle materials
            if (obj.material) {
              const mats = Array.isArray(obj.material) ? obj.material : [obj.material];
              mats.forEach(mat => {
                mat.envMapIntensity = 1.0;
                
                // สำหรับ glass/transparent materials
                if (mat.name.toLowerCase().includes('glass') || 
                    mat.name.toLowerCase().includes('window')) {
                  mat.transparent = true;
                  mat.opacity = 0.3;
                  mat.roughness = 0.0;
                  mat.metalness = 0.9;
                  mat.envMapIntensity = 2.0;
                }
              });
            }
          }
        });
        
        resolve(gltf);
      },
      (xhr) => {
        if (onProgress) {
          onProgress(xhr.loaded / xhr.total);
        }
      },
      reject
    );
  });
}
```

---

## 2. PBR Materials สำหรับ Architecture

### Material Library สำหรับ Archviz

```javascript
class ArchvizMaterialLibrary {
  constructor(renderer) {
    this.renderer = renderer;
    this.materials = new Map();
    this.textureLoader = new THREE.TextureLoader();
    this.envMap = null;
  }
  
  setEnvironmentMap(texture) {
    this.envMap = texture;
    this.materials.forEach(mat => {
      mat.envMap = this.envMap;
      mat.needsUpdate = true;
    });
  }
  
  // Concrete material
  createConcrete(options = {}) {
    const mat = new THREE.MeshStandardMaterial({
      color: options.color || 0xcccccc,
      roughness: options.roughness ?? 0.9,
      metalness: 0.0,
      ...options
    });
    
    if (options.normalMap) {
      mat.normalMap = options.normalMap;
      mat.normalScale.set(1, 1);
    }
    
    return mat;
  }
  
  // Glass material
  createGlass(options = {}) {
    return new THREE.MeshPhysicalMaterial({
      color: options.color || 0xffffff,
      metalness: 0.0,
      roughness: 0.0,
      transmission: options.transmission ?? 0.95,
      transparent: true,
      opacity: options.opacity ?? 1.0,
      ior: options.ior ?? 1.5,
      reflectivity: 0.5,
      envMapIntensity: 2.0,
      envMap: this.envMap,
      side: THREE.DoubleSide,
      ...options
    });
  }
  
  // Wood material
  createWood(options = {}) {
    const mat = new THREE.MeshStandardMaterial({
      color: options.color || 0x8B4513,
      roughness: options.roughness ?? 0.7,
      metalness: 0.0,
      ...options
    });
    return mat;
  }
  
  // Metal (steel, aluminum)
  createMetal(options = {}) {
    return new THREE.MeshStandardMaterial({
      color: options.color || 0xcccccc,
      metalness: options.metalness ?? 0.9,
      roughness: options.roughness ?? 0.2,
      envMapIntensity: 2.0,
      envMap: this.envMap,
      ...options
    });
  }
  
  // Stone/Marble
  createStone(options = {}) {
    return new THREE.MeshStandardMaterial({
      color: options.color || 0xe0d0c0,
      roughness: options.roughness ?? 0.4,
      metalness: 0.05,
      ...options
    });
  }
  
  // Painted wall
  createPaintedWall(color = 0xffffff) {
    return new THREE.MeshStandardMaterial({
      color,
      roughness: 0.8,
      metalness: 0.0
    });
  }
  
  // Emissive material สำหรับ lights
  createEmissive(color = 0xffffff, intensity = 1.0) {
    return new THREE.MeshStandardMaterial({
      emissive: new THREE.Color(color),
      emissiveIntensity: intensity,
      roughness: 1.0,
      metalness: 0.0
    });
  }
}
```

---

## 3. HDRI Lighting สำหรับ Archviz

```javascript
import { RGBELoader } from 'three/addons/loaders/RGBELoader.js';
import { PMREMGenerator } from 'three';

class ArchvizLighting {
  constructor(renderer, scene) {
    this.renderer = renderer;
    this.scene = scene;
    this.pmremGenerator = new PMREMGenerator(renderer);
    this.pmremGenerator.compileEquirectangularShader();
    
    this.sunLight = null;
    this.ambientLight = null;
    this.envMapTexture = null;
  }
  
  async loadHDRI(url) {
    const rgbeLoader = new RGBELoader();
    
    return new Promise((resolve) => {
      rgbeLoader.load(url, (texture) => {
        texture.mapping = THREE.EquirectangularReflectionMapping;
        
        // สร้าง environment map จาก HDRI
        const envMap = this.pmremGenerator.fromEquirectangular(texture).texture;
        
        this.scene.environment = envMap;
        this.scene.background = texture;  // หรือ envMap ถ้าต้องการ blurred background
        
        this.envMapTexture = envMap;
        
        texture.dispose();
        
        resolve(envMap);
      });
    });
  }
  
  // ตั้งค่า sun light ตาม time of day
  setSunPosition(timeOfDay, intensity = 1.0) {
    if (!this.sunLight) {
      this.sunLight = new THREE.DirectionalLight(0xfff5e0, intensity);
      this.sunLight.castShadow = true;
      this.sunLight.shadow.mapSize.set(4096, 4096);
      this.sunLight.shadow.camera.near = 0.5;
      this.sunLight.shadow.camera.far = 100;
      this.sunLight.shadow.camera.left = -30;
      this.sunLight.shadow.camera.right = 30;
      this.sunLight.shadow.camera.top = 30;
      this.sunLight.shadow.camera.bottom = -30;
      this.sunLight.shadow.bias = -0.001;
      this.scene.add(this.sunLight);
    }
    
    // timeOfDay: 0-24 hours
    const angle = ((timeOfDay - 6) / 12) * Math.PI;  // 6am = 0, 6pm = PI
    const elevAngle = Math.max(0, Math.sin(angle));
    
    this.sunLight.position.set(
      Math.cos(angle) * 30,
      elevAngle * 30 + 5,
      Math.sin(angle) * 15
    );
    
    // Warm/cool color based on time
    if (timeOfDay < 8 || timeOfDay > 18) {
      // Sunrise/sunset: warm orange
      this.sunLight.color.setHSL(0.08, 0.8, 0.8);
    } else {
      // Midday: white
      this.sunLight.color.setHSL(0.1, 0.1, 1.0);
    }
    
    this.sunLight.intensity = intensity * elevAngle;
    
    return this;
  }
  
  // เพิ่ม fill lights สำหรับ interior
  addInteriorLights(positions, color = 0xffeecc, intensity = 2) {
    positions.forEach(pos => {
      const light = new THREE.PointLight(color, intensity, 8);
      light.position.set(pos.x, pos.y, pos.z);
      light.castShadow = false;  // performance
      this.scene.add(light);
    });
  }
}
```

---

## 4. Interactive Floor Plan

```javascript
class FloorPlanViewer {
  constructor(scene, camera, renderer, rooms) {
    this.scene = scene;
    this.camera = camera;
    this.renderer = renderer;
    this.rooms = rooms;
    
    this.selected = null;
    this.highlighted = null;
    
    this.group = new THREE.Group();
    scene.add(this.group);
    
    this.raycaster = new THREE.Raycaster();
    this.mouse = new THREE.Vector2();
    
    this._buildFloorPlan();
    this._setupInteraction();
  }
  
  _buildFloorPlan() {
    this.rooms.forEach(room => {
      // สร้าง room mesh จาก polygon
      const shape = new THREE.Shape();
      room.vertices.forEach((v, i) => {
        if (i === 0) shape.moveTo(v.x, v.y);
        else shape.lineTo(v.x, v.y);
      });
      shape.closePath();
      
      // Extrude สร้างเป็น 3D
      const extrudeSettings = {
        depth: 3,  // ความสูงของห้อง
        bevelEnabled: false
      };
      
      const geo = new THREE.ExtrudeGeometry(shape, extrudeSettings);
      const mat = new THREE.MeshStandardMaterial({
        color: room.color || 0xddddff,
        transparent: true,
        opacity: 0.7,
        roughness: 0.8
      });
      
      const mesh = new THREE.Mesh(geo, mat);
      mesh.rotation.x = -Math.PI / 2;
      mesh.userData = { room, type: 'room' };
      mesh.castShadow = true;
      mesh.receiveShadow = true;
      
      this.group.add(mesh);
      room.mesh = mesh;
      
      // Floor
      const floorGeo = new THREE.ShapeGeometry(shape);
      const floorMat = new THREE.MeshStandardMaterial({
        color: room.floorColor || 0xf0ead6,
        roughness: 0.9
      });
      const floor = new THREE.Mesh(floorGeo, floorMat);
      floor.rotation.x = -Math.PI / 2;
      floor.position.y = 0.01;
      floor.receiveShadow = true;
      this.group.add(floor);
      
      // Room label
      const label = this._createRoomLabel(room.name, room.center);
      this.group.add(label);
      
      // Outline
      const outlineGeo = new THREE.EdgesGeometry(geo);
      const outlineMat = new THREE.LineBasicMaterial({ color: 0x333333 });
      const outline = new THREE.LineSegments(outlineGeo, outlineMat);
      outline.rotation.x = -Math.PI / 2;
      this.group.add(outline);
    });
    
    // สร้าง walls
    this._buildWalls();
  }
  
  _buildWalls() {
    // สร้าง walls ตาม room boundaries (simplified)
    const wallMat = new THREE.MeshStandardMaterial({
      color: 0xfafafa,
      roughness: 0.8
    });
    
    this.rooms.forEach(room => {
      const verts = room.vertices;
      for (let i = 0; i < verts.length; i++) {
        const a = verts[i];
        const b = verts[(i + 1) % verts.length];
        
        // สร้าง wall panel
        const length = Math.sqrt((b.x - a.x) ** 2 + (b.y - a.y) ** 2);
        const midX = (a.x + b.x) / 2;
        const midY = (a.y + b.y) / 2;
        const angle = Math.atan2(b.y - a.y, b.x - a.x);
        
        const wallGeo = new THREE.BoxGeometry(length, 3, 0.15);
        const wall = new THREE.Mesh(wallGeo, wallMat);
        
        wall.position.set(midX, 1.5, midY);
        wall.rotation.y = -angle;
        wall.castShadow = true;
        wall.receiveShadow = true;
        
        this.group.add(wall);
      }
    });
  }
  
  _createRoomLabel(name, center) {
    const canvas = document.createElement('canvas');
    canvas.width = 512;
    canvas.height = 128;
    const ctx = canvas.getContext('2d');
    
    ctx.fillStyle = 'rgba(0,0,0,0.7)';
    ctx.roundRect(0, 0, 512, 128, 10);
    ctx.fill();
    
    ctx.font = 'bold 42px Arial';
    ctx.fillStyle = '#ffffff';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText(name, 256, 64);
    
    const texture = new THREE.CanvasTexture(canvas);
    const sprite = new THREE.Sprite(new THREE.SpriteMaterial({ 
      map: texture, 
      transparent: true 
    }));
    
    sprite.position.set(center.x, 2, center.y);
    sprite.scale.set(3, 0.75, 1);
    
    return sprite;
  }
  
  _setupInteraction() {
    const canvas = this.renderer.domElement;
    
    canvas.addEventListener('mousemove', e => {
      this.mouse.x = (e.clientX / window.innerWidth) * 2 - 1;
      this.mouse.y = -(e.clientY / window.innerHeight) * 2 + 1;
      
      this._updateHover();
    });
    
    canvas.addEventListener('click', () => {
      if (this.highlighted) {
        this._selectRoom(this.highlighted);
      }
    });
  }
  
  _updateHover() {
    this.raycaster.setFromCamera(this.mouse, this.camera);
    
    const meshes = this.rooms.map(r => r.mesh).filter(Boolean);
    const intersects = this.raycaster.intersectObjects(meshes);
    
    // Reset highlight
    if (this.highlighted && this.highlighted !== this.selected) {
      this.highlighted.mesh.material.opacity = 0.7;
      this.highlighted.mesh.material.emissive.setHex(0x000000);
    }
    
    if (intersects.length > 0) {
      const room = intersects[0].object.userData.room;
      
      if (room !== this.highlighted) {
        this.highlighted = room;
        
        if (room !== this.selected) {
          room.mesh.material.opacity = 0.85;
          room.mesh.material.emissive.setHex(0x222244);
        }
      }
    } else {
      this.highlighted = null;
    }
  }
  
  _selectRoom(room) {
    // Deselect previous
    if (this.selected) {
      this.selected.mesh.material.opacity = 0.7;
      this.selected.mesh.material.emissive.setHex(0x000000);
    }
    
    this.selected = room;
    room.mesh.material.opacity = 0.95;
    room.mesh.material.emissive.setHex(0x114488);
    
    // Show room info
    if (this.onRoomSelect) {
      this.onRoomSelect(room);
    }
    
    // Animate camera ไปที่ room
    this._flyToRoom(room);
  }
  
  _flyToRoom(room) {
    const targetPos = new THREE.Vector3(room.center.x, 5, room.center.y + 5);
    const targetLookAt = new THREE.Vector3(room.center.x, 0, room.center.y);
    
    // GSAP animation (ถ้ามี) หรือใช้ lerp
    const startPos = this.camera.position.clone();
    const startTarget = new THREE.Vector3();
    
    let t = 0;
    const duration = 1.0;
    const startTime = Date.now();
    
    const animateFly = () => {
      t = Math.min((Date.now() - startTime) / (duration * 1000), 1);
      const ease = t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t;
      
      this.camera.position.lerpVectors(startPos, targetPos, ease);
      
      if (t < 1) requestAnimationFrame(animateFly);
    };
    
    requestAnimationFrame(animateFly);
  }
}
```

---

## 5. Walkthrough Mode (First Person)

```javascript
class WalkthroughController {
  constructor(camera, scene, options = {}) {
    this.camera = camera;
    this.scene = scene;
    this.options = {
      height: 1.7,  // ความสูงสายตา (meter)
      speed: 3.0,
      mouseSensitivity: 0.002,
      ...options
    };
    
    this.euler = new THREE.Euler(0, 0, 0, 'YXZ');
    this.velocity = new THREE.Vector3();
    this.keys = {};
    this.isLocked = false;
    
    this.raycaster = new THREE.Raycaster(
      new THREE.Vector3(), 
      new THREE.Vector3(0, -1, 0), 
      0, 
      options.height + 0.5
    );
    
    this._setupControls();
  }
  
  _setupControls() {
    document.addEventListener('keydown', e => { this.keys[e.code] = true; });
    document.addEventListener('keyup', e => { this.keys[e.code] = false; });
    
    document.addEventListener('pointerlockchange', () => {
      this.isLocked = !!document.pointerLockElement;
    });
    
    document.addEventListener('mousemove', e => {
      if (!this.isLocked) return;
      
      this.euler.setFromQuaternion(this.camera.quaternion);
      this.euler.y -= e.movementX * this.options.mouseSensitivity;
      this.euler.x -= e.movementY * this.options.mouseSensitivity;
      
      // Clamp vertical rotation
      this.euler.x = Math.max(-Math.PI / 3, Math.min(Math.PI / 3, this.euler.x));
      
      this.camera.quaternion.setFromEuler(this.euler);
    });
    
    document.addEventListener('click', () => {
      if (!this.isLocked) {
        document.body.requestPointerLock();
      }
    });
    
    // Touch controls (mobile)
    this._setupTouchControls();
  }
  
  _setupTouchControls() {
    let touchStartX = 0;
    let touchStartY = 0;
    
    document.addEventListener('touchstart', e => {
      touchStartX = e.touches[0].clientX;
      touchStartY = e.touches[0].clientY;
    });
    
    document.addEventListener('touchmove', e => {
      const dx = e.touches[0].clientX - touchStartX;
      const dy = e.touches[0].clientY - touchStartY;
      
      this.euler.setFromQuaternion(this.camera.quaternion);
      this.euler.y -= dx * this.options.mouseSensitivity * 0.5;
      this.euler.x -= dy * this.options.mouseSensitivity * 0.5;
      this.euler.x = Math.max(-Math.PI / 3, Math.min(Math.PI / 3, this.euler.x));
      this.camera.quaternion.setFromEuler(this.euler);
      
      touchStartX = e.touches[0].clientX;
      touchStartY = e.touches[0].clientY;
      
      e.preventDefault();
    }, { passive: false });
  }
  
  update(dt, collidables = []) {
    if (!this.isLocked) return;
    
    const { speed } = this.options;
    
    // Movement direction
    const forward = new THREE.Vector3();
    const right = new THREE.Vector3();
    
    this.camera.getWorldDirection(forward);
    forward.y = 0;
    forward.normalize();
    
    right.crossVectors(forward, new THREE.Vector3(0, 1, 0));
    
    // Apply input
    const moveDir = new THREE.Vector3();
    
    if (this.keys['KeyW'] || this.keys['ArrowUp']) moveDir.add(forward);
    if (this.keys['KeyS'] || this.keys['ArrowDown']) moveDir.sub(forward);
    if (this.keys['KeyA'] || this.keys['ArrowLeft']) moveDir.sub(right);
    if (this.keys['KeyD'] || this.keys['ArrowRight']) moveDir.add(right);
    
    if (moveDir.length() > 0) {
      moveDir.normalize().multiplyScalar(speed * dt);
    }
    
    // Collision detection
    const newPos = this.camera.position.clone().add(moveDir);
    
    if (collidables.length > 0) {
      // Check collision
      const directions = [
        forward.clone(),
        forward.clone().negate(),
        right.clone(),
        right.clone().negate()
      ];
      
      directions.forEach(dir => {
        this.raycaster.ray.origin.copy(this.camera.position);
        this.raycaster.ray.direction.copy(dir);
        this.raycaster.ray.direction.y = 0;
        this.raycaster.far = 0.5;
        
        const hits = this.raycaster.intersectObjects(collidables, true);
        if (hits.length > 0) {
          // Push back
          const push = dir.multiplyScalar(-(0.5 - hits[0].distance));
          newPos.add(push);
        }
      });
    }
    
    // Floor collision
    this.raycaster.ray.origin.copy(newPos);
    this.raycaster.ray.origin.y += 1;
    this.raycaster.ray.direction.set(0, -1, 0);
    this.raycaster.far = this.options.height + 0.5;
    
    const floorHits = this.raycaster.intersectObjects(collidables, true);
    
    if (floorHits.length > 0) {
      newPos.y = floorHits[0].point.y + this.options.height;
    }
    
    this.camera.position.copy(newPos);
  }
}
```

---

## 6. Complete Archviz Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title>Archviz Demo - Interactive Apartment</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { overflow: hidden; background: #1a1a2e; font-family: 'Segoe UI', sans-serif; }
    canvas { display: block; }
    
    #ui {
      position: fixed;
      bottom: 20px;
      left: 50%;
      transform: translateX(-50%);
      display: flex;
      gap: 10px;
      background: rgba(0,0,0,0.7);
      padding: 12px;
      border-radius: 30px;
      border: 1px solid rgba(255,255,255,0.1);
    }
    
    .view-btn {
      padding: 8px 20px;
      background: rgba(255,255,255,0.1);
      border: 1px solid rgba(255,255,255,0.2);
      color: white;
      border-radius: 20px;
      cursor: pointer;
      font-size: 13px;
      transition: all 0.2s;
    }
    .view-btn:hover, .view-btn.active {
      background: rgba(68,136,255,0.5);
      border-color: rgba(68,136,255,0.8);
    }
    
    #info {
      position: fixed;
      top: 20px;
      left: 20px;
      color: white;
      font-size: 14px;
      background: rgba(0,0,0,0.6);
      padding: 10px 15px;
      border-radius: 8px;
    }
    
    #room-info {
      position: fixed;
      top: 20px;
      right: 20px;
      color: white;
      width: 200px;
      background: rgba(0,0,30,0.8);
      border: 1px solid rgba(68,136,255,0.3);
      padding: 15px;
      border-radius: 8px;
      display: none;
    }
    .room-info-title { font-size: 16px; font-weight: bold; margin-bottom: 8px; color: #88aaff; }
    .room-info-detail { font-size: 12px; color: #aaa; line-height: 1.6; }
    
    #crosshair {
      position: fixed;
      top: 50%; left: 50%;
      transform: translate(-50%, -50%);
      width: 16px; height: 16px;
      display: none;
      pointer-events: none;
    }
    #crosshair::before, #crosshair::after {
      content: '';
      position: absolute;
      background: rgba(255,255,255,0.7);
    }
    #crosshair::before { width: 2px; height: 16px; left: 7px; top: 0; }
    #crosshair::after { height: 2px; width: 16px; top: 7px; left: 0; }
  </style>
</head>
<body>
  <canvas id="canvas"></canvas>
  
  <div id="info">
    <div id="viewLabel">View: Overview</div>
    <div style="font-size: 11px; color: #aaa; margin-top: 4px;">Click to explore</div>
  </div>
  
  <div id="room-info">
    <div class="room-info-title" id="roomName">Room</div>
    <div class="room-info-detail" id="roomDetails"></div>
  </div>
  
  <div id="ui">
    <button class="view-btn active" id="viewOverview">Overview</button>
    <button class="view-btn" id="viewFloorPlan">Floor Plan</button>
    <button class="view-btn" id="viewWalkthrough">Walkthrough</button>
    <button class="view-btn" id="viewExterior">Exterior</button>
  </div>
  
  <div id="crosshair"></div>

  <script type="module">
    import * as THREE from 'https://cdn.skypack.dev/three@0.160.0';
    import { OrbitControls } from 'https://cdn.skypack.dev/three@0.160.0/examples/jsm/controls/OrbitControls.js';
    
    const canvas = document.getElementById('canvas');
    const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    renderer.outputColorSpace = THREE.SRGBColorSpace;
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.toneMappingExposure = 1.2;
    
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x87ceeb);  // sky blue
    scene.fog = new THREE.Fog(0x87ceeb, 30, 80);
    
    const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 200);
    camera.position.set(15, 10, 15);
    
    const controls = new OrbitControls(camera, canvas);
    controls.enableDamping = true;
    controls.target.set(0, 2, 0);
    controls.minDistance = 2;
    controls.maxDistance = 50;
    
    // === Lighting ===
    const ambientLight = new THREE.AmbientLight(0x88aaff, 0.5);
    scene.add(ambientLight);
    
    const sunLight = new THREE.DirectionalLight(0xffeecc, 3);
    sunLight.position.set(20, 30, 20);
    sunLight.castShadow = true;
    sunLight.shadow.mapSize.set(2048, 2048);
    sunLight.shadow.camera.near = 0.5;
    sunLight.shadow.camera.far = 100;
    sunLight.shadow.camera.left = -25;
    sunLight.shadow.camera.right = 25;
    sunLight.shadow.camera.top = 25;
    sunLight.shadow.camera.bottom = -25;
    sunLight.shadow.bias = -0.001;
    scene.add(sunLight);
    
    // Hemisphere light สำหรับ sky/ground lighting
    const hemiLight = new THREE.HemisphereLight(0x87ceeb, 0x4a7c59, 0.5);
    scene.add(hemiLight);
    
    // === Build Apartment Building ===
    function createBox(w, h, d, color, x, y, z, options = {}) {
      const geo = new THREE.BoxGeometry(w, h, d);
      const mat = new THREE.MeshStandardMaterial({ color, ...options });
      const mesh = new THREE.Mesh(geo, mat);
      mesh.position.set(x, y, z);
      mesh.castShadow = true;
      mesh.receiveShadow = true;
      return mesh;
    }
    
    const buildingGroup = new THREE.Group();
    scene.add(buildingGroup);
    
    // Ground
    const ground = createBox(60, 0.5, 60, 0x4a7c59, 0, -0.25, 0, { roughness: 0.9 });
    scene.add(ground);
    
    // Pathway
    const path = createBox(3, 0.1, 20, 0xccbbaa, 0, 0.05, 10, { roughness: 0.8 });
    scene.add(path);
    
    // Main building - floors
    const floorCount = 4;
    const floorHeight = 3;
    const buildingWidth = 12;
    const buildingDepth = 8;
    
    for (let f = 0; f < floorCount; f++) {
      const y = f * floorHeight + floorHeight / 2;
      
      // Main structure
      const floor = createBox(buildingWidth, floorHeight, buildingDepth, 0xf0e8d8, 0, y, 0, {
        roughness: 0.7
      });
      buildingGroup.add(floor);
      
      // Windows (front)
      for (let i = 0; i < 3; i++) {
        const wx = (i - 1) * 4;
        
        // Window frame
        const windowFrame = createBox(2, 2, 0.2, 0x888888, wx, y, buildingDepth / 2 + 0.1, {
          metalness: 0.5, roughness: 0.3
        });
        buildingGroup.add(windowFrame);
        
        // Glass
        const glass = new THREE.Mesh(
          new THREE.BoxGeometry(1.6, 1.6, 0.05),
          new THREE.MeshPhysicalMaterial({
            color: 0x88bbff,
            transparent: true,
            opacity: 0.5,
            roughness: 0.0,
            metalness: 0.0,
            transmission: 0.5
          })
        );
        glass.position.set(wx, y, buildingDepth / 2 + 0.15);
        buildingGroup.add(glass);
      }
      
      // Balcony
      const balcony = createBox(buildingWidth + 1, 0.2, 1.5, 0xddccbb, 0, y - floorHeight / 2, buildingDepth / 2 + 0.75);
      buildingGroup.add(balcony);
      
      // Balcony railing
      const railing = createBox(buildingWidth, 0.8, 0.05, 0x888888, 0, y - floorHeight / 2 + 0.4, buildingDepth / 2 + 1.45, { metalness: 0.8 });
      buildingGroup.add(railing);
    }
    
    // Roof
    const roof = createBox(buildingWidth + 0.5, 0.3, buildingDepth + 0.5, 0xccbbaa, 0, floorCount * floorHeight + 0.15, 0);
    buildingGroup.add(roof);
    
    // Door
    const door = createBox(1.5, 2.5, 0.2, 0x4a3728, 0, 1.25, buildingDepth / 2 + 0.1, { roughness: 0.7 });
    buildingGroup.add(door);
    
    // Trees
    function createTree(x, z) {
      const trunk = createBox(0.3, 2, 0.3, 0x4a3728, x, 1, z);
      const top = new THREE.Mesh(
        new THREE.SphereGeometry(1.5, 8, 8),
        new THREE.MeshStandardMaterial({ color: 0x2d5a27, roughness: 0.9 })
      );
      top.position.set(x, 3, z);
      top.castShadow = true;
      
      scene.add(trunk);
      scene.add(top);
    }
    
    createTree(-8, 8);
    createTree(8, 8);
    createTree(-9, -3);
    createTree(9, -3);
    
    // Interior lights
    for (let f = 0; f < floorCount; f++) {
      const y = f * floorHeight + floorHeight / 2;
      const light = new THREE.PointLight(0xffeecc, 0.5, 6);
      light.position.set(0, y, 0);
      buildingGroup.add(light);
    }
    
    // Room info data
    const roomData = {
      'living': { name: 'ห้องนั่งเล่น', details: 'พื้นที่: 25 ตร.ม.\nเฟอร์นิเจอร์: โซฟา, ทีวี, โต๊ะกาแฟ' },
      'bedroom': { name: 'ห้องนอน', details: 'พื้นที่: 18 ตร.ม.\nเฟอร์นิเจอร์: เตียง King, ตู้เสื้อผ้า' },
      'kitchen': { name: 'ห้องครัว', details: 'พื้นที่: 12 ตร.ม.\nเครื่องใช้: เตาแก๊ส, ตู้เย็น, เครื่องล้างจาน' },
      'bathroom': { name: 'ห้องน้ำ', details: 'พื้นที่: 6 ตร.ม.\nสิ่งอำนวย: อ่างอาบน้ำ, ฝักบัว' }
    };
    
    // View controls
    let currentView = 'overview';
    const viewPositions = {
      overview: { pos: [15, 10, 15], target: [0, 2, 0] },
      floorPlan: { pos: [0, 30, 0], target: [0, 0, 0] },
      exterior: { pos: [20, 8, 20], target: [0, 6, 0] }
    };
    
    function setView(view) {
      currentView = view;
      document.querySelectorAll('.view-btn').forEach(b => b.classList.remove('active'));
      document.getElementById(`view${view.charAt(0).toUpperCase() + view.slice(1)}`).classList.add('active');
      document.getElementById('viewLabel').textContent = `View: ${view}`;
      
      if (viewPositions[view]) {
        const vp = viewPositions[view];
        controls.enabled = true;
        document.getElementById('crosshair').style.display = 'none';
        
        // Animate camera
        const startPos = camera.position.clone();
        const endPos = new THREE.Vector3(...vp.pos);
        const startTarget = controls.target.clone();
        const endTarget = new THREE.Vector3(...vp.target);
        
        let t = 0;
        const duration = 1500;
        const start = Date.now();
        
        const animMove = () => {
          t = Math.min((Date.now() - start) / duration, 1);
          const ease = 1 - Math.pow(1 - t, 3);
          camera.position.lerpVectors(startPos, endPos, ease);
          controls.target.lerpVectors(startTarget, endTarget, ease);
          if (t < 1) requestAnimationFrame(animMove);
        };
        animMove();
      }
    }
    
    document.getElementById('viewOverview').onclick = () => setView('overview');
    document.getElementById('viewFloorPlan').onclick = () => setView('floorPlan');
    document.getElementById('viewExterior').onclick = () => setView('exterior');
    document.getElementById('viewWalkthrough').onclick = () => {
      document.querySelectorAll('.view-btn').forEach(b => b.classList.remove('active'));
      document.getElementById('viewWalkthrough').classList.add('active');
      document.getElementById('viewLabel').textContent = 'View: Walkthrough';
      document.getElementById('crosshair').style.display = 'block';
      // In real implementation, switch to FPS controls here
    };
    
    // Animation
    const clock = new THREE.Clock();
    
    function animate() {
      requestAnimationFrame(animate);
      
      const time = clock.getElapsedTime();
      
      // Animate sun
      sunLight.position.x = Math.cos(time * 0.05) * 30;
      sunLight.position.z = Math.sin(time * 0.05) * 20;
      
      controls.update();
      renderer.render(scene, camera);
    }
    
    window.addEventListener('resize', () => {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    });
    
    animate();
  </script>
</body>
</html>
```

---

## 7. สรุป Architectural Visualization

### เทคนิคที่ใช้:
- **PBR Materials**: ให้ความสมจริง
- **HDRI Lighting**: แสงธรรมชาติ
- **Shadow Maps**: ให้ depth และ realism
- **LOD**: ปรับ detail ตามระยะทาง
- **Interactivity**: ให้ client explore ได้
- **Walkthrough**: สร้าง immersive experience

### Performance Optimization สำหรับ Archviz:
```javascript
// 1. ใช้ instancing สำหรับ repeated objects (furniture)
// 2. Merge static geometry
const merged = BufferGeometryUtils.mergeGeometries(staticGeos);

// 3. ปิด shadow สำหรับ small objects
smallObjects.forEach(obj => {
  obj.castShadow = false;
  obj.receiveShadow = false;
});

// 4. ใช้ compressed textures (KTX2/ASTC)
// 5. Frustum culling (Three.js ทำให้อัตโนมัติ)
// 6. Reduce shadow map size สำหรับ distant lights
```

ในบทถัดไป (Part 88) เราจะเรียนรู้เกี่ยวกับ Product Configurator
