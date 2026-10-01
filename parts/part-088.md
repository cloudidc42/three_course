# Part 88: Product Configurator

## บทนำ: 3D Product Configurator คืออะไร?

Product Configurator คือ web application ที่ให้ลูกค้าสามารถ customize สินค้าแบบ real-time ในรูปแบบ 3D เช่น เลือกสีรถยนต์, ออกแบบรองเท้า, จัดเฟอร์นิเจอร์ ซึ่งช่วยเพิ่ม conversion rate ได้อย่างมีนัยสำคัญ

---

## 1. GLTF Part Swapping

### การ Setup GLTF สำหรับ Configurator

```javascript
import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';

class ProductConfigurator {
  constructor(scene, camera, renderer) {
    this.scene = scene;
    this.camera = camera;
    this.renderer = renderer;
    
    this.model = null;
    this.parts = new Map();  // partName -> mesh
    this.materials = new Map();  // materialName -> material
    this.configurations = {};
    
    this.selectedPart = null;
    this.hoveredPart = null;
    
    this.raycaster = new THREE.Raycaster();
    this.mouse = new THREE.Vector2();
    
    this._setupLoader();
    this._setupInteraction();
  }
  
  _setupLoader() {
    const dracoLoader = new DRACOLoader();
    dracoLoader.setDecoderPath('/libs/draco/');
    
    this.loader = new GLTFLoader();
    this.loader.setDRACOLoader(dracoLoader);
  }
  
  async loadModel(url, options = {}) {
    return new Promise((resolve, reject) => {
      this.loader.load(url, (gltf) => {
        this.model = gltf.scene;
        
        // Extract parts
        this.model.traverse(obj => {
          if (obj.isMesh) {
            const partName = obj.name;
            
            // เก็บ reference ไปที่ mesh
            this.parts.set(partName, obj);
            
            // เก็บ original material
            const originalMat = obj.material;
            if (!this.configurations[partName]) {
              this.configurations[partName] = {
                material: originalMat,
                visible: obj.visible
              };
            }
            
            // เพิ่ม userData สำหรับ interaction
            obj.userData.partName = partName;
            obj.userData.configurable = options.configurableParts 
              ? options.configurableParts.includes(partName)
              : true;
            
            obj.castShadow = true;
            obj.receiveShadow = true;
          }
        });
        
        this.scene.add(this.model);
        resolve(gltf);
      }, undefined, reject);
    });
  }
  
  // เปลี่ยนสีของ part
  setPartColor(partName, color) {
    const part = this.parts.get(partName);
    if (!part) return false;
    
    // Clone material เพื่อไม่ให้ affect other instances
    if (!part.userData.cloned) {
      part.material = part.material.clone();
      part.userData.cloned = true;
    }
    
    part.material.color.set(color);
    this.configurations[partName] = { ...this.configurations[partName], color };
    
    return true;
  }
  
  // เปลี่ยน material ทั้ง part
  setPartMaterial(partName, materialName) {
    const part = this.parts.get(partName);
    const material = this.materials.get(materialName);
    
    if (!part || !material) return false;
    
    part.material = material;
    this.configurations[partName] = { ...this.configurations[partName], material: materialName };
    
    return true;
  }
  
  // Toggle visibility ของ part
  setPartVisibility(partName, visible) {
    const part = this.parts.get(partName);
    if (!part) return false;
    
    part.visible = visible;
    this.configurations[partName] = { ...this.configurations[partName], visible };
    
    return true;
  }
  
  // Swap mesh (เปลี่ยน geometry ทั้ง part)
  async swapMesh(partName, newMeshUrl) {
    const oldPart = this.parts.get(partName);
    if (!oldPart) return false;
    
    const gltf = await new Promise((resolve, reject) => {
      this.loader.load(newMeshUrl, resolve, undefined, reject);
    });
    
    // หา replacement mesh
    let newMesh = null;
    gltf.scene.traverse(obj => {
      if (obj.isMesh && !newMesh) newMesh = obj;
    });
    
    if (!newMesh) return false;
    
    // Replace mesh
    newMesh.position.copy(oldPart.position);
    newMesh.rotation.copy(oldPart.rotation);
    newMesh.scale.copy(oldPart.scale);
    newMesh.material = oldPart.material;
    newMesh.name = partName;
    newMesh.userData = { ...oldPart.userData };
    
    oldPart.parent.add(newMesh);
    oldPart.parent.remove(oldPart);
    
    this.parts.set(partName, newMesh);
    
    return true;
  }
  
  // เพิ่ม material preset
  addMaterialPreset(name, material) {
    this.materials.set(name, material);
    return this;
  }
  
  // Highlight selected/hovered part
  highlightPart(partName, mode = 'select') {
    const part = this.parts.get(partName);
    if (!part) return;
    
    if (!part.userData.cloned) {
      part.material = part.material.clone();
      part.userData.cloned = true;
    }
    
    if (mode === 'select') {
      part.material.emissive.setHex(0x113366);
      part.material.emissiveIntensity = 0.5;
    } else if (mode === 'hover') {
      part.material.emissive.setHex(0x112222);
      part.material.emissiveIntensity = 0.3;
    }
  }
  
  unhighlightPart(partName) {
    const part = this.parts.get(partName);
    if (!part || !part.userData.cloned) return;
    
    part.material.emissiveIntensity = 0;
  }
  
  // Get current configuration
  getConfiguration() {
    return { ...this.configurations };
  }
  
  // Exploded view
  explode(factor = 1.0) {
    this.parts.forEach((part, name) => {
      if (!part.userData.originalPosition) {
        part.userData.originalPosition = part.position.clone();
      }
      
      // Explode ออกจาก center
      const dir = part.userData.originalPosition.clone().normalize();
      const targetPos = part.userData.originalPosition.clone()
        .add(dir.multiplyScalar(factor * 2));
      
      // Animate
      const startPos = part.position.clone();
      let t = 0;
      const start = Date.now();
      
      const animate = () => {
        t = Math.min((Date.now() - start) / 800, 1);
        const ease = 1 - Math.pow(1 - t, 3);
        part.position.lerpVectors(startPos, targetPos, ease);
        if (t < 1) requestAnimationFrame(animate);
      };
      animate();
    });
  }
  
  unexplode() {
    this.parts.forEach((part) => {
      if (!part.userData.originalPosition) return;
      
      const startPos = part.position.clone();
      const targetPos = part.userData.originalPosition;
      let t = 0;
      const start = Date.now();
      
      const animate = () => {
        t = Math.min((Date.now() - start) / 800, 1);
        const ease = 1 - Math.pow(1 - t, 3);
        part.position.lerpVectors(startPos, targetPos, ease);
        if (t < 1) requestAnimationFrame(animate);
      };
      animate();
    });
  }
  
  // Zoom to specific part
  zoomToPart(partName) {
    const part = this.parts.get(partName);
    if (!part) return;
    
    // คำนวณ bounding sphere ของ part
    const box = new THREE.Box3().setFromObject(part);
    const sphere = new THREE.Sphere();
    box.getBoundingSphere(sphere);
    
    // คำนวณ camera position
    const distance = sphere.radius * 3;
    const direction = this.camera.position.clone()
      .sub(sphere.center)
      .normalize()
      .multiplyScalar(distance);
    
    const targetCamPos = sphere.center.clone().add(direction);
    
    // Animate camera
    const startPos = this.camera.position.clone();
    let t = 0;
    const start = Date.now();
    
    const animZoom = () => {
      t = Math.min((Date.now() - start) / 1000, 1);
      const ease = 1 - Math.pow(1 - t, 3);
      this.camera.position.lerpVectors(startPos, targetCamPos, ease);
      if (t < 1) requestAnimationFrame(animZoom);
    };
    animZoom();
  }
  
  _setupInteraction() {
    this.renderer.domElement.addEventListener('mousemove', e => {
      this.mouse.x = (e.clientX / window.innerWidth) * 2 - 1;
      this.mouse.y = -(e.clientY / window.innerHeight) * 2 + 1;
      
      this._updateHover();
    });
    
    this.renderer.domElement.addEventListener('click', () => {
      if (this.hoveredPart) {
        this._selectPart(this.hoveredPart);
      }
    });
  }
  
  _updateHover() {
    this.raycaster.setFromCamera(this.mouse, this.camera);
    
    const meshes = [];
    this.model?.traverse(obj => {
      if (obj.isMesh && obj.userData.configurable) meshes.push(obj);
    });
    
    const hits = this.raycaster.intersectObjects(meshes);
    
    if (this.hoveredPart && this.hoveredPart !== this.selectedPart) {
      this.unhighlightPart(this.hoveredPart);
    }
    
    if (hits.length > 0) {
      this.hoveredPart = hits[0].object.userData.partName;
      if (this.hoveredPart !== this.selectedPart) {
        this.highlightPart(this.hoveredPart, 'hover');
      }
    } else {
      this.hoveredPart = null;
    }
  }
  
  _selectPart(partName) {
    if (this.selectedPart) {
      this.unhighlightPart(this.selectedPart);
    }
    
    this.selectedPart = partName;
    this.highlightPart(partName, 'select');
    
    if (this.onPartSelect) {
      this.onPartSelect(partName, this.parts.get(partName));
    }
  }
}
```

---

## 2. Screenshot และ Share Feature

```javascript
class ConfiguratorScreenshot {
  constructor(renderer, scene, camera) {
    this.renderer = renderer;
    this.scene = scene;
    this.camera = camera;
  }
  
  // Screenshot ขณะนั้น
  takeScreenshot(width = 1920, height = 1080) {
    // Save state
    const originalSize = new THREE.Vector2();
    this.renderer.getSize(originalSize);
    const originalPixelRatio = this.renderer.getPixelRatio();
    
    // Render ที่ resolution สูง
    this.renderer.setSize(width, height);
    this.renderer.setPixelRatio(1);
    this.renderer.render(this.scene, this.camera);
    
    // Get image data
    const dataURL = this.renderer.domElement.toDataURL('image/png');
    
    // Restore
    this.renderer.setSize(originalSize.x, originalSize.y);
    this.renderer.setPixelRatio(originalPixelRatio);
    
    return dataURL;
  }
  
  // Download screenshot
  download(filename = 'product-config.png') {
    const dataURL = this.takeScreenshot();
    
    const link = document.createElement('a');
    link.download = filename;
    link.href = dataURL;
    link.click();
  }
  
  // Share via Web Share API
  async share(title = 'My Custom Product', text = 'Check out my configuration!') {
    if (!navigator.share) {
      // Fallback: copy link to clipboard
      await navigator.clipboard.writeText(window.location.href);
      alert('Link copied to clipboard!');
      return;
    }
    
    // Convert to blob
    const dataURL = this.takeScreenshot(1200, 900);
    const blob = await (await fetch(dataURL)).blob();
    const file = new File([blob], 'config.png', { type: 'image/png' });
    
    try {
      await navigator.share({
        title,
        text,
        url: window.location.href,
        files: [file]
      });
    } catch (e) {
      console.log('Share cancelled or not supported');
    }
  }
  
  // สร้าง URL ที่มี configuration encoded
  generateShareURL(configuration) {
    const params = new URLSearchParams();
    params.set('config', btoa(JSON.stringify(configuration)));
    
    const url = new URL(window.location.href);
    url.search = params.toString();
    
    return url.toString();
  }
  
  // โหลด configuration จาก URL
  loadFromURL() {
    const params = new URLSearchParams(window.location.search);
    const configParam = params.get('config');
    
    if (!configParam) return null;
    
    try {
      return JSON.parse(atob(configParam));
    } catch (e) {
      return null;
    }
  }
}
```

---

## 3. Complete Shoe Configurator

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title>3D Shoe Configurator</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { overflow: hidden; background: #f5f5f5; font-family: 'Segoe UI', sans-serif; }
    canvas { display: block; }
    
    #ui {
      position: fixed;
      right: 0;
      top: 0;
      bottom: 0;
      width: 320px;
      background: white;
      box-shadow: -3px 0 15px rgba(0,0,0,0.1);
      overflow-y: auto;
      z-index: 10;
    }
    
    #brand {
      padding: 20px;
      background: #1a1a1a;
      color: white;
      font-size: 20px;
      font-weight: bold;
    }
    
    .section {
      padding: 15px 20px;
      border-bottom: 1px solid #eee;
    }
    .section-title {
      font-size: 11px;
      font-weight: bold;
      text-transform: uppercase;
      letter-spacing: 1px;
      color: #999;
      margin-bottom: 12px;
    }
    
    .part-selector {
      display: flex;
      flex-wrap: wrap;
      gap: 8px;
      margin-bottom: 15px;
    }
    .part-btn {
      padding: 6px 14px;
      border: 1px solid #ddd;
      border-radius: 20px;
      background: white;
      font-size: 12px;
      cursor: pointer;
      transition: all 0.2s;
    }
    .part-btn:hover { background: #f0f0f0; }
    .part-btn.active {
      background: #1a1a1a;
      color: white;
      border-color: #1a1a1a;
    }
    
    .color-grid {
      display: grid;
      grid-template-columns: repeat(8, 1fr);
      gap: 6px;
    }
    .color-swatch {
      width: 100%;
      aspect-ratio: 1;
      border-radius: 50%;
      cursor: pointer;
      border: 2px solid transparent;
      transition: all 0.2s;
    }
    .color-swatch:hover { transform: scale(1.15); }
    .color-swatch.selected { border-color: #1a1a1a; transform: scale(1.1); }
    
    .material-grid {
      display: grid;
      grid-template-columns: repeat(3, 1fr);
      gap: 8px;
    }
    .material-card {
      border: 2px solid #eee;
      border-radius: 8px;
      padding: 8px;
      cursor: pointer;
      text-align: center;
      transition: all 0.2s;
    }
    .material-card:hover { border-color: #aaa; }
    .material-card.active { border-color: #1a1a1a; }
    .material-preview {
      width: 100%;
      height: 50px;
      border-radius: 4px;
      margin-bottom: 4px;
    }
    .material-name { font-size: 10px; }
    
    .price-bar {
      padding: 15px 20px;
      background: #1a1a1a;
      color: white;
      display: flex;
      justify-content: space-between;
      align-items: center;
    }
    .price { font-size: 24px; font-weight: bold; }
    .add-to-cart {
      background: #ff5500;
      color: white;
      border: none;
      padding: 10px 20px;
      border-radius: 20px;
      font-size: 13px;
      cursor: pointer;
      font-weight: bold;
    }
    .add-to-cart:hover { background: #ee4400; }
    
    .action-buttons {
      display: flex;
      gap: 10px;
      padding: 10px 20px;
    }
    .action-btn {
      flex: 1;
      padding: 8px;
      border: 1px solid #ddd;
      border-radius: 5px;
      background: white;
      font-size: 11px;
      cursor: pointer;
      text-align: center;
    }
    .action-btn:hover { background: #f0f0f0; }
    
    #canvas-container {
      position: fixed;
      top: 0;
      left: 0;
      right: 320px;
      bottom: 0;
    }
    
    canvas { width: 100% !important; height: 100% !important; }
    
    #part-tooltip {
      position: fixed;
      background: rgba(0,0,0,0.8);
      color: white;
      padding: 5px 10px;
      border-radius: 4px;
      font-size: 12px;
      pointer-events: none;
      display: none;
      z-index: 100;
    }
  </style>
</head>
<body>
  <div id="canvas-container">
    <canvas id="canvas"></canvas>
  </div>
  <div id="part-tooltip"></div>
  
  <div id="ui">
    <div id="brand">SNEAKER STUDIO 3D</div>
    
    <!-- Part Selection -->
    <div class="section">
      <div class="section-title">Select Part to Customize</div>
      <div class="part-selector" id="partSelector">
        <button class="part-btn active" data-part="upper">Upper</button>
        <button class="part-btn" data-part="sole">Sole</button>
        <button class="part-btn" data-part="midsole">Midsole</button>
        <button class="part-btn" data-part="laces">Laces</button>
        <button class="part-btn" data-part="tongue">Tongue</button>
        <button class="part-btn" data-part="heel">Heel Tab</button>
      </div>
    </div>
    
    <!-- Color Selection -->
    <div class="section">
      <div class="section-title">Color</div>
      <div class="color-grid" id="colorGrid"></div>
    </div>
    
    <!-- Material Selection -->
    <div class="section">
      <div class="section-title">Material</div>
      <div class="material-grid" id="materialGrid">
        <div class="material-card active" data-material="leather">
          <div class="material-preview" style="background: linear-gradient(135deg, #c8a87a, #a07850);"></div>
          <div class="material-name">Leather</div>
        </div>
        <div class="material-card" data-material="suede">
          <div class="material-preview" style="background: linear-gradient(135deg, #b09070, #907060);"></div>
          <div class="material-name">Suede</div>
        </div>
        <div class="material-card" data-material="canvas">
          <div class="material-preview" style="background: repeating-linear-gradient(45deg, #ddd, #ddd 2px, #eee 2px, #eee 8px);"></div>
          <div class="material-name">Canvas</div>
        </div>
        <div class="material-card" data-material="mesh">
          <div class="material-preview" style="background: repeating-linear-gradient(0deg, rgba(0,0,0,0.1) 0, rgba(0,0,0,0.1) 1px, transparent 1px, transparent 4px), repeating-linear-gradient(90deg, rgba(0,0,0,0.1) 0, rgba(0,0,0,0.1) 1px, transparent 1px, transparent 4px), #e0e0e0;"></div>
          <div class="material-name">Mesh</div>
        </div>
        <div class="material-card" data-material="rubber">
          <div class="material-preview" style="background: #333;"></div>
          <div class="material-name">Rubber</div>
        </div>
        <div class="material-card" data-material="foam">
          <div class="material-preview" style="background: linear-gradient(135deg, #f0f0f0, #e0e0e0);"></div>
          <div class="material-name">Foam</div>
        </div>
      </div>
    </div>
    
    <!-- View Options -->
    <div class="section">
      <div class="section-title">View Options</div>
      <div class="action-buttons">
        <button class="action-btn" id="explodeBtn">Exploded View</button>
        <button class="action-btn" id="resetBtn">Reset View</button>
      </div>
    </div>
    
    <!-- Price & Actions -->
    <div class="price-bar">
      <div>
        <div style="font-size: 11px; color: #aaa;">Total Price</div>
        <div class="price" id="price">฿4,590</div>
      </div>
      <button class="add-to-cart">Add to Cart</button>
    </div>
    
    <div class="action-buttons">
      <button class="action-btn" id="screenshotBtn">📸 Screenshot</button>
      <button class="action-btn" id="shareBtn">🔗 Share</button>
    </div>
  </div>

  <script type="module">
    import * as THREE from 'https://cdn.skypack.dev/three@0.160.0';
    import { OrbitControls } from 'https://cdn.skypack.dev/three@0.160.0/examples/jsm/controls/OrbitControls.js';
    
    const container = document.getElementById('canvas-container');
    const canvas = document.getElementById('canvas');
    const tooltip = document.getElementById('part-tooltip');
    
    const W = window.innerWidth - 320;
    const H = window.innerHeight;
    
    const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setSize(W, H);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    renderer.outputColorSpace = THREE.SRGBColorSpace;
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.toneMappingExposure = 1.5;
    
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0xf5f5f5);
    
    const camera = new THREE.PerspectiveCamera(45, W / H, 0.01, 50);
    camera.position.set(0, 0.5, 3);
    
    const controls = new OrbitControls(camera, canvas);
    controls.enableDamping = true;
    controls.target.set(0, 0.3, 0);
    controls.minDistance = 1;
    controls.maxDistance = 5;
    controls.maxPolarAngle = Math.PI * 0.85;
    controls.autoRotate = true;
    controls.autoRotateSpeed = 1;
    
    // Lighting
    scene.add(new THREE.AmbientLight(0xffffff, 0.5));
    
    const light1 = new THREE.DirectionalLight(0xffffff, 2);
    light1.position.set(5, 8, 5);
    light1.castShadow = true;
    light1.shadow.mapSize.set(1024, 1024);
    scene.add(light1);
    
    const light2 = new THREE.DirectionalLight(0x8899ff, 0.5);
    light2.position.set(-5, 3, -5);
    scene.add(light2);
    
    const light3 = new THREE.PointLight(0xff8844, 0.5, 8);
    light3.position.set(0, 4, 0);
    scene.add(light3);
    
    // Ground
    const ground = new THREE.Mesh(
      new THREE.CircleGeometry(2, 64),
      new THREE.MeshStandardMaterial({ color: 0xeeeeee, roughness: 0.9 })
    );
    ground.rotation.x = -Math.PI / 2;
    ground.receiveShadow = true;
    scene.add(ground);
    
    // ===== สร้าง Shoe (procedural) =====
    // ในการใช้งานจริงจะใช้ GLTF แต่สำหรับ demo ใช้ procedural shapes
    
    const shoeGroup = new THREE.Group();
    scene.add(shoeGroup);
    
    // Color options
    const colorOptions = [
      '#ffffff', '#000000', '#ff4444', '#4444ff', '#44bb44',
      '#ffaa00', '#ff44aa', '#44aaff', '#aaaaaa', '#553311',
      '#226644', '#662244', '#334488', '#886622', '#445566', '#aa4400'
    ];
    
    // Material configurations
    const materialConfigs = {
      leather: { roughness: 0.4, metalness: 0.0, sheen: 0.0 },
      suede: { roughness: 0.9, metalness: 0.0 },
      canvas: { roughness: 0.8, metalness: 0.0 },
      mesh: { roughness: 0.6, metalness: 0.0, transparent: true, opacity: 0.9 },
      rubber: { roughness: 0.7, metalness: 0.0 },
      foam: { roughness: 0.95, metalness: 0.0 }
    };
    
    // Part configuration
    const partConfig = {
      upper: { color: '#ffffff', material: 'leather' },
      sole: { color: '#333333', material: 'rubber' },
      midsole: { color: '#f0f0f0', material: 'foam' },
      laces: { color: '#ffffff', material: 'canvas' },
      tongue: { color: '#ffffff', material: 'leather' },
      heel: { color: '#333333', material: 'rubber' }
    };
    
    // Price table
    const materialPrices = {
      leather: 500, suede: 400, canvas: 200, mesh: 300, rubber: 150, foam: 100
    };
    
    const basePrices = {
      upper: 2000, sole: 800, midsole: 500, laces: 100, tongue: 400, heel: 200
    };
    
    // สร้าง shoe parts (simplified shapes)
    function createShoePart(name, geometry, position, color = '#ffffff', matConfig = materialConfigs.leather) {
      const material = new THREE.MeshStandardMaterial({
        color: new THREE.Color(color),
        ...matConfig
      });
      
      const mesh = new THREE.Mesh(geometry, material);
      mesh.position.copy(position);
      mesh.castShadow = true;
      mesh.receiveShadow = true;
      mesh.name = name;
      mesh.userData.partName = name;
      mesh.userData.configurable = true;
      
      return mesh;
    }
    
    // Upper
    const upperGeo = new THREE.SphereGeometry(0.6, 32, 32);
    upperGeo.scale(1.8, 0.6, 1.0);
    const upper = createShoePart('upper', upperGeo, new THREE.Vector3(0, 0.4, 0));
    shoeGroup.add(upper);
    
    // Tongue
    const tongueGeo = new THREE.BoxGeometry(0.3, 0.5, 0.05);
    const tongue = createShoePart('tongue', tongueGeo, new THREE.Vector3(0, 0.65, 0.3));
    shoeGroup.add(tongue);
    
    // Midsole
    const midsoleGeo = new THREE.BoxGeometry(1.8, 0.12, 0.75);
    midsoleGeo.translate(0, 0, -0.02);
    const midsole = createShoePart('midsole', midsoleGeo, new THREE.Vector3(0, 0.08, 0), '#f0f0f0', materialConfigs.foam);
    shoeGroup.add(midsole);
    
    // Sole
    const soleShape = new THREE.Shape();
    soleShape.moveTo(-0.9, -0.35);
    soleShape.quadraticCurveTo(-1.05, 0, -0.9, 0.35);
    soleShape.lineTo(0.7, 0.38);
    soleShape.quadraticCurveTo(1.0, 0.35, 1.0, 0);
    soleShape.quadraticCurveTo(1.0, -0.35, 0.7, -0.38);
    soleShape.closePath();
    
    const soleGeo = new THREE.ExtrudeGeometry(soleShape, { depth: 0.08, bevelEnabled: false });
    soleGeo.rotateX(-Math.PI / 2);
    const sole = createShoePart('sole', soleGeo, new THREE.Vector3(0, 0, 0), '#333333', materialConfigs.rubber);
    shoeGroup.add(sole);
    
    // Heel
    const heelGeo = new THREE.BoxGeometry(0.4, 0.15, 0.6);
    const heel = createShoePart('heel', heelGeo, new THREE.Vector3(-0.75, 0.2, 0), '#333333', materialConfigs.rubber);
    shoeGroup.add(heel);
    
    // Laces
    const lacesGeo = new THREE.CylinderGeometry(0.01, 0.01, 1.5, 4);
    lacesGeo.rotateX(Math.PI / 2);
    lacesGeo.rotateY(Math.PI / 3);
    const laces = createShoePart('laces', lacesGeo, new THREE.Vector3(0, 0.65, 0.2), '#ffffff', materialConfigs.canvas);
    shoeGroup.add(laces);
    
    shoeGroup.rotation.y = -Math.PI / 6;
    
    // ===== UI Controls =====
    
    let selectedPart = 'upper';
    let isExploded = false;
    
    // Build color swatches
    const colorGrid = document.getElementById('colorGrid');
    colorOptions.forEach(color => {
      const swatch = document.createElement('div');
      swatch.className = 'color-swatch';
      swatch.style.backgroundColor = color;
      swatch.style.border = '2px solid ' + (color === '#ffffff' ? '#ddd' : 'transparent');
      
      swatch.addEventListener('click', () => {
        // Update 3D model
        const part = shoeGroup.getObjectByName(selectedPart);
        if (part) {
          part.material = part.material.clone();
          part.material.color.set(color);
        }
        
        partConfig[selectedPart].color = color;
        
        // Update UI
        document.querySelectorAll('.color-swatch').forEach(s => s.classList.remove('selected'));
        swatch.classList.add('selected');
        
        updatePrice();
      });
      
      colorGrid.appendChild(swatch);
    });
    
    // Part selection
    document.getElementById('partSelector').addEventListener('click', e => {
      if (e.target.classList.contains('part-btn')) {
        selectedPart = e.target.dataset.part;
        document.querySelectorAll('.part-btn').forEach(b => b.classList.remove('active'));
        e.target.classList.add('active');
        
        // Update color swatches to show current color of selected part
        const currentColor = partConfig[selectedPart]?.color || '#ffffff';
        document.querySelectorAll('.color-swatch').forEach(s => {
          s.classList.toggle('selected', s.style.backgroundColor === currentColor);
        });
        
        // Zoom to part
        const part = shoeGroup.getObjectByName(selectedPart);
        if (part) {
          // ไม่ zoom ใน demo นี้ แต่ highlight
          shoeGroup.children.forEach(child => {
            child.material.emissiveIntensity = child.name === selectedPart ? 0.3 : 0;
            if (!child.material.emissive) child.material.emissive = new THREE.Color(0x000000);
            if (child.name === selectedPart) {
              child.material.emissive.setHex(0x224488);
            } else {
              child.material.emissive.setHex(0x000000);
            }
          });
        }
        
        controls.autoRotate = false;
      }
    });
    
    // Material selection
    document.getElementById('materialGrid').addEventListener('click', e => {
      const card = e.target.closest('.material-card');
      if (!card) return;
      
      const materialName = card.dataset.material;
      const matConfig = materialConfigs[materialName];
      
      if (matConfig) {
        const part = shoeGroup.getObjectByName(selectedPart);
        if (part) {
          const currentColor = part.material.color.getHex();
          part.material = new THREE.MeshStandardMaterial({
            color: currentColor,
            ...matConfig
          });
        }
        
        partConfig[selectedPart].material = materialName;
        
        document.querySelectorAll('.material-card').forEach(c => c.classList.remove('active'));
        card.classList.add('active');
        
        updatePrice();
      }
    });
    
    // Explode button
    document.getElementById('explodeBtn').addEventListener('click', () => {
      isExploded = !isExploded;
      
      const positions = {
        upper: [0, 0.4, 0],
        tongue: [0, 0.65, 0.3],
        midsole: [0, 0.08, 0],
        sole: [0, 0, 0],
        heel: [-0.75, 0.2, 0],
        laces: [0, 0.65, 0.2]
      };
      
      const explodeOffset = {
        upper: [0, 0.8, 0],
        tongue: [0.5, 1.2, 0.5],
        midsole: [0, 0.3, 0],
        sole: [0, -0.5, 0],
        heel: [-1.5, 0, 0],
        laces: [0.5, 1.5, 0]
      };
      
      shoeGroup.children.forEach((child) => {
        const name = child.name;
        if (!name || !positions[name]) return;
        
        const basePos = positions[name];
        const offset = explodeOffset[name] || [0, 0, 0];
        const target = isExploded
          ? [basePos[0] + offset[0], basePos[1] + offset[1], basePos[2] + offset[2]]
          : basePos;
        
        const start = child.position.clone();
        const end = new THREE.Vector3(...target);
        let t = 0;
        const startTime = Date.now();
        
        const animExplode = () => {
          t = Math.min((Date.now() - startTime) / 600, 1);
          const ease = 1 - Math.pow(1 - t, 3);
          child.position.lerpVectors(start, end, ease);
          if (t < 1) requestAnimationFrame(animExplode);
        };
        animExplode();
      });
      
      document.getElementById('explodeBtn').textContent = isExploded ? 'Collapse View' : 'Exploded View';
    });
    
    // Reset
    document.getElementById('resetBtn').addEventListener('click', () => {
      camera.position.set(0, 0.5, 3);
      controls.target.set(0, 0.3, 0);
      controls.autoRotate = true;
    });
    
    // Screenshot
    document.getElementById('screenshotBtn').addEventListener('click', () => {
      renderer.render(scene, camera);
      const dataURL = renderer.domElement.toDataURL('image/png');
      const link = document.createElement('a');
      link.download = 'my-shoe.png';
      link.href = dataURL;
      link.click();
    });
    
    // Share
    document.getElementById('shareBtn').addEventListener('click', () => {
      const config = JSON.stringify(partConfig);
      const encoded = btoa(config);
      const url = `${window.location.origin}${window.location.pathname}?config=${encoded}`;
      
      if (navigator.clipboard) {
        navigator.clipboard.writeText(url).then(() => {
          document.getElementById('shareBtn').textContent = '✓ Copied!';
          setTimeout(() => {
            document.getElementById('shareBtn').textContent = '🔗 Share';
          }, 2000);
        });
      }
    });
    
    // Price calculation
    function updatePrice() {
      let total = 2590;  // base price
      
      Object.entries(partConfig).forEach(([part, config]) => {
        const matPrice = materialPrices[config.material] || 200;
        total += matPrice * 0.1;
      });
      
      document.getElementById('price').textContent = `฿${Math.round(total).toLocaleString()}`;
    }
    
    // Hover tooltip
    const raycaster = new THREE.Raycaster();
    const mouse = new THREE.Vector2();
    
    canvas.addEventListener('mousemove', e => {
      mouse.x = (e.clientX / W) * 2 - 1;
      mouse.y = -(e.clientY / H) * 2 + 1;
      
      raycaster.setFromCamera(mouse, camera);
      const hits = raycaster.intersectObjects(shoeGroup.children);
      
      if (hits.length > 0) {
        const part = hits[0].object.name;
        tooltip.style.display = 'block';
        tooltip.style.left = `${e.clientX + 15}px`;
        tooltip.style.top = `${e.clientY - 30}px`;
        tooltip.textContent = part.charAt(0).toUpperCase() + part.slice(1);
      } else {
        tooltip.style.display = 'none';
      }
    });
    
    // Resize
    window.addEventListener('resize', () => {
      const W = window.innerWidth - 320;
      const H = window.innerHeight;
      camera.aspect = W / H;
      camera.updateProjectionMatrix();
      renderer.setSize(W, H);
    });
    
    // Animation
    const clock = new THREE.Clock();
    function animate() {
      requestAnimationFrame(animate);
      controls.update();
      renderer.render(scene, camera);
    }
    animate();
  </script>
</body>
</html>
```

---

## 4. สรุป Product Configurator

### Key Features:
1. **Part Selection**: Click เพื่อเลือก part ที่ต้องการ customize
2. **Color Picker**: เปลี่ยนสีแบบ real-time
3. **Material Swap**: เปลี่ยน material ทั้ง PBR properties
4. **Exploded View**: แสดง parts แยกออกจากกัน
5. **Screenshot/Share**: บันทึกหรือแชร์ configuration
6. **Price Calculator**: คำนวณราคาตาม options

### Performance Tips:
```javascript
// 1. Clone material ก็ต่อเมื่อจำเป็น
if (!mesh.userData.materialCloned) {
  mesh.material = mesh.material.clone();
  mesh.userData.materialCloned = true;
}

// 2. ใช้ dispose() เมื่อเปลี่ยน material
oldMaterial.dispose();

// 3. Preload materials ทั้งหมดก่อน
// 4. ใช้ texture atlas แทน individual textures
// 5. Environment map แบบ pre-baked
```

ในบทถัดไป (Part 89) เราจะเรียนรู้เกี่ยวกับ Character Animation System
