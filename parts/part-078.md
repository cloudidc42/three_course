# Part 78: Mobile Optimization for Three.js

## บทนำ

Mobile devices มี GPU, memory, และ battery จำกัดกว่า desktop การ optimize Three.js สำหรับ mobile เป็นสิ่งสำคัญมาก บทนี้จะครอบคลุม pixelRatio, texture quality, touch controls, adaptive rendering, และ power management

---

## 78.1 Device Detection และ Quality Settings

```javascript
// src/utils/deviceDetector.js
export class DeviceDetector {
  constructor(renderer) {
    this.renderer = renderer;
    this.profile = this._detectProfile();
  }
  
  _detectProfile() {
    const ua = navigator.userAgent;
    const isMobile = /iPhone|iPad|iPod|Android/i.test(ua);
    const isTablet = /iPad|Android(?!.*Mobile)/i.test(ua);
    const isLowEnd = this._isLowEndDevice();
    
    const gl = this.renderer.getContext();
    const maxTexSize = gl.getParameter(gl.MAX_TEXTURE_SIZE);
    const maxVaryings = gl.getParameter(gl.MAX_VARYING_VECTORS);
    
    const tier = isLowEnd ? 'low' : isMobile ? (isTablet ? 'mid' : 'mobile') : 'high';
    
    return {
      isMobile,
      isTablet,
      isLowEnd,
      tier,
      maxTextureSize: maxTexSize,
      pixelRatio: this._getOptimalPixelRatio(tier),
      settings: this._getSettings(tier),
    };
  }
  
  _isLowEndDevice() {
    // ตรวจสอบ device memory (Chrome only)
    const memory = navigator.deviceMemory;
    if (memory && memory <= 2) return true;
    
    // ตรวจสอบ hardware concurrency
    const cores = navigator.hardwareConcurrency;
    if (cores && cores <= 2) return true;
    
    return false;
  }
  
  _getOptimalPixelRatio(tier) {
    const deviceRatio = window.devicePixelRatio || 1;
    
    switch (tier) {
      case 'low': return Math.min(deviceRatio, 1);
      case 'mobile': return Math.min(deviceRatio, 1.5);
      case 'mid': return Math.min(deviceRatio, 2);
      default: return deviceRatio;
    }
  }
  
  _getSettings(tier) {
    const presets = {
      low: {
        shadows: false,
        shadowMapSize: 512,
        antialias: false,
        maxParticles: 5000,
        textureQuality: 0.5,
        postProcessing: false,
        lodBias: 0.5,
        maxLights: 2,
      },
      mobile: {
        shadows: true,
        shadowMapSize: 1024,
        antialias: false,
        maxParticles: 20000,
        textureQuality: 0.75,
        postProcessing: false,
        lodBias: 0.75,
        maxLights: 3,
      },
      mid: {
        shadows: true,
        shadowMapSize: 2048,
        antialias: true,
        maxParticles: 50000,
        textureQuality: 1.0,
        postProcessing: true,
        lodBias: 1.0,
        maxLights: 4,
      },
      high: {
        shadows: true,
        shadowMapSize: 4096,
        antialias: true,
        maxParticles: 200000,
        textureQuality: 1.0,
        postProcessing: true,
        lodBias: 1.0,
        maxLights: 8,
      },
    };
    
    return presets[tier] || presets.high;
  }
  
  applyToRenderer() {
    const { pixelRatio, settings, tier } = this.profile;
    
    this.renderer.setPixelRatio(pixelRatio);
    this.renderer.shadowMap.enabled = settings.shadows;
    
    if (settings.shadows) {
      this.renderer.shadowMap.type = settings.shadowMapSize <= 1024
        ? THREE.BasicShadowMap
        : THREE.PCFSoftShadowMap;
    }
    
    console.log(`[Device] Tier: ${tier}, PixelRatio: ${pixelRatio}, Shadows: ${settings.shadows}`);
    
    return settings;
  }
}
```

---

## 78.2 Touch Controls

```javascript
// src/controls/TouchControls.js
import * as THREE from 'three';

export class TouchControls {
  constructor(camera, domElement) {
    this.camera = camera;
    this.domElement = domElement;
    
    // State
    this.touches = {};
    this.prevTouches = {};
    this.rotateStart = new THREE.Vector2();
    this.rotateEnd = new THREE.Vector2();
    this.rotateDelta = new THREE.Vector2();
    this.pinchStart = 0;
    this.pinchEnd = 0;
    
    // Camera state
    this.spherical = new THREE.Spherical();
    this.target = new THREE.Vector3();
    this.minDistance = 2;
    this.maxDistance = 50;
    this.rotateSpeed = 0.5;
    this.zoomSpeed = 0.1;
    
    // Initialize from camera
    this.spherical.setFromVector3(camera.position.clone().sub(this.target));
    
    // Bind events
    this._bindEvents();
  }
  
  _bindEvents() {
    const el = this.domElement;
    
    this._onTouchStart = this._onTouchStart.bind(this);
    this._onTouchMove = this._onTouchMove.bind(this);
    this._onTouchEnd = this._onTouchEnd.bind(this);
    
    el.addEventListener('touchstart', this._onTouchStart, { passive: false });
    el.addEventListener('touchmove', this._onTouchMove, { passive: false });
    el.addEventListener('touchend', this._onTouchEnd);
  }
  
  _onTouchStart(e) {
    e.preventDefault();
    
    if (e.touches.length === 1) {
      // Single touch: rotate
      this.rotateStart.set(e.touches[0].clientX, e.touches[0].clientY);
    } else if (e.touches.length === 2) {
      // Double touch: pinch to zoom
      const dx = e.touches[0].clientX - e.touches[1].clientX;
      const dy = e.touches[0].clientY - e.touches[1].clientY;
      this.pinchStart = Math.sqrt(dx * dx + dy * dy);
    }
  }
  
  _onTouchMove(e) {
    e.preventDefault();
    
    if (e.touches.length === 1) {
      this.rotateEnd.set(e.touches[0].clientX, e.touches[0].clientY);
      this.rotateDelta.subVectors(this.rotateEnd, this.rotateStart);
      
      const element = this.domElement;
      this.spherical.theta -= (2 * Math.PI * this.rotateDelta.x) / element.clientWidth * this.rotateSpeed;
      this.spherical.phi -= (2 * Math.PI * this.rotateDelta.y) / element.clientHeight * this.rotateSpeed;
      
      // Clamp phi
      this.spherical.phi = Math.max(0.05, Math.min(Math.PI - 0.05, this.spherical.phi));
      
      this.rotateStart.copy(this.rotateEnd);
    } else if (e.touches.length === 2) {
      const dx = e.touches[0].clientX - e.touches[1].clientX;
      const dy = e.touches[0].clientY - e.touches[1].clientY;
      this.pinchEnd = Math.sqrt(dx * dx + dy * dy);
      
      const scale = this.pinchStart / this.pinchEnd;
      this.spherical.radius = Math.max(
        this.minDistance,
        Math.min(this.maxDistance, this.spherical.radius * (1 + (scale - 1) * this.zoomSpeed))
      );
      
      this.pinchStart = this.pinchEnd;
    }
  }
  
  _onTouchEnd() {
    // Nothing needed
  }
  
  update() {
    // Convert spherical to cartesian
    const position = new THREE.Vector3();
    position.setFromSpherical(this.spherical);
    position.add(this.target);
    this.camera.position.copy(position);
    this.camera.lookAt(this.target);
  }
  
  dispose() {
    const el = this.domElement;
    el.removeEventListener('touchstart', this._onTouchStart);
    el.removeEventListener('touchmove', this._onTouchMove);
    el.removeEventListener('touchend', this._onTouchEnd);
  }
}
```

---

## 78.3 Adaptive Quality System

```javascript
// src/utils/adaptiveQuality.js
export class AdaptiveQualitySystem {
  constructor(renderer, targetFPS = 60) {
    this.renderer = renderer;
    this.targetFPS = targetFPS;
    this.currentQuality = 1.0; // 0.5 - 1.0
    
    this.fpsHistory = [];
    this.maxHistory = 60;
    this.lastTime = performance.now();
    this.frameCount = 0;
    this.lastFPS = targetFPS;
    
    this.minQuality = 0.3;
    this.maxQuality = 1.0;
    this.adjustRate = 0.05;
    
    // Container size
    this.baseWidth = renderer.domElement.clientWidth;
    this.baseHeight = renderer.domElement.clientHeight;
  }
  
  update() {
    this.frameCount++;
    const now = performance.now();
    const elapsed = now - this.lastTime;
    
    if (elapsed >= 1000) {
      this.lastFPS = (this.frameCount / elapsed) * 1000;
      this.frameCount = 0;
      this.lastTime = now;
      
      this.fpsHistory.push(this.lastFPS);
      if (this.fpsHistory.length > this.maxHistory) {
        this.fpsHistory.shift();
      }
      
      this._adjust();
    }
  }
  
  _adjust() {
    const avgFPS = this.fpsHistory.reduce((a, b) => a + b, 0) / this.fpsHistory.length;
    const ratio = avgFPS / this.targetFPS;
    
    if (ratio < 0.8 && this.currentQuality > this.minQuality) {
      // FPS too low - reduce quality
      this.currentQuality = Math.max(this.minQuality, this.currentQuality - this.adjustRate);
      this._applyQuality();
    } else if (ratio > 1.0 && this.currentQuality < this.maxQuality) {
      // FPS fine - increase quality gradually
      this.currentQuality = Math.min(this.maxQuality, this.currentQuality + this.adjustRate * 0.3);
      this._applyQuality();
    }
  }
  
  _applyQuality() {
    const w = Math.round(this.baseWidth * this.currentQuality);
    const h = Math.round(this.baseHeight * this.currentQuality);
    this.renderer.setSize(w, h, false); // false = don't update style
  }
  
  get stats() {
    return {
      fps: this.lastFPS.toFixed(0),
      quality: (this.currentQuality * 100).toFixed(0) + '%',
    };
  }
}
```

---

## 78.4 Battery API Integration

```javascript
// src/utils/batteryManager.js
export class BatteryManager {
  constructor(onQualityChange) {
    this.onQualityChange = onQualityChange;
    this.battery = null;
    this.charging = true;
    this.level = 1.0;
    
    this._init();
  }
  
  async _init() {
    if (!navigator.getBattery) return;
    
    try {
      this.battery = await navigator.getBattery();
      
      this.battery.addEventListener('chargingchange', () => this._update());
      this.battery.addEventListener('levelchange', () => this._update());
      
      this._update();
    } catch (e) {
      console.log('Battery API not available');
    }
  }
  
  _update() {
    if (!this.battery) return;
    
    this.charging = this.battery.charging;
    this.level = this.battery.level;
    
    let quality = 'high';
    
    if (!this.charging) {
      if (this.level < 0.1) quality = 'low';
      else if (this.level < 0.3) quality = 'medium';
      else quality = 'high';
    }
    
    this.onQualityChange?.(quality, this.level, this.charging);
    
    console.log(`Battery: ${(this.level * 100).toFixed(0)}%, Charging: ${this.charging}, Quality: ${quality}`);
  }
  
  get isLowPower() {
    return !this.charging && this.level < 0.3;
  }
}
```

---

## 78.5 Mobile-Optimized Scene

```javascript
// src/demos/mobileOptimized.js
import * as THREE from 'three';
import { DeviceDetector } from '../utils/deviceDetector.js';
import { TouchControls } from '../controls/TouchControls.js';
import { AdaptiveQualitySystem } from '../utils/adaptiveQuality.js';

export function createMobileDemo(container) {
  // === Setup with device detection ===
  const renderer = new THREE.WebGLRenderer({
    antialias: false,  // OFF by default, device detector จะเปิดถ้า tier สูง
    powerPreference: 'default',  // 'high-performance' หรือ 'low-power'
  });
  renderer.setSize(container.clientWidth, container.clientHeight);
  container.appendChild(renderer.domElement);
  
  const detector = new DeviceDetector(renderer);
  const settings = detector.applyToRenderer();
  
  console.log('Device profile:', detector.profile);
  
  // Scene
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#0a0a1a');
  
  // Fog ลด render distance บน mobile
  if (detector.profile.isMobile) {
    scene.fog = new THREE.Fog('#0a0a1a', 30, 60);
  }
  
  const camera = new THREE.PerspectiveCamera(60, container.clientWidth / container.clientHeight, 0.1, 100);
  camera.position.set(0, 5, 15);
  
  // Touch controls
  const touchControls = new TouchControls(camera, renderer.domElement);
  
  // Adaptive quality
  const adaptiveQuality = new AdaptiveQualitySystem(renderer);
  
  // === Scene Content (optimized per tier) ===
  const OBJECT_COUNT = settings.tier === 'low' ? 20 : settings.tier === 'mobile' ? 50 : 100;
  const objects = [];
  
  // Use instanced mesh for mobile
  const instancedGeo = new THREE.BoxGeometry(0.8, 0.8, 0.8);
  const instancedMat = new THREE.MeshStandardMaterial({ color: '#4ecdc4' });
  const instancedMesh = new THREE.InstancedMesh(instancedGeo, instancedMat, OBJECT_COUNT);
  instancedMesh.castShadow = settings.shadows;
  
  const dummy = new THREE.Object3D();
  
  for (let i = 0; i < OBJECT_COUNT; i++) {
    dummy.position.set(
      (Math.random() - 0.5) * 20,
      (Math.random() - 0.5) * 10,
      (Math.random() - 0.5) * 20
    );
    dummy.rotation.set(
      Math.random() * Math.PI * 2,
      Math.random() * Math.PI * 2,
      Math.random() * Math.PI * 2
    );
    dummy.updateMatrix();
    instancedMesh.setMatrixAt(i, dummy.matrix);
    instancedMesh.setColorAt(i, new THREE.Color().setHSL(Math.random(), 0.8, 0.6));
  }
  instancedMesh.instanceMatrix.needsUpdate = true;
  instancedMesh.instanceColor.needsUpdate = true;
  scene.add(instancedMesh);
  
  // Floor
  const floor = new THREE.Mesh(
    new THREE.PlaneGeometry(50, 50),
    new THREE.MeshStandardMaterial({ color: '#111', roughness: 0.9 })
  );
  floor.rotation.x = -Math.PI / 2;
  floor.position.y = -5;
  floor.receiveShadow = settings.shadows;
  scene.add(floor);
  
  // Minimal lighting for mobile
  scene.add(new THREE.AmbientLight(0xffffff, 0.6));
  
  if (settings.maxLights >= 2) {
    const dirLight = new THREE.DirectionalLight(0xffffff, 1);
    dirLight.position.set(10, 20, 10);
    if (settings.shadows) {
      dirLight.castShadow = true;
      dirLight.shadow.mapSize.width = settings.shadowMapSize;
      dirLight.shadow.mapSize.height = settings.shadowMapSize;
    }
    scene.add(dirLight);
  }
  
  // Stats overlay
  const statsDiv = document.createElement('div');
  statsDiv.style.cssText = `
    position: absolute; bottom: 10px; left: 10px;
    background: rgba(0,0,0,0.7); color: #4ecdc4;
    padding: 8px 12px; border-radius: 8px;
    font-family: monospace; font-size: 11px; z-index: 10;
  `;
  container.style.position = 'relative';
  container.appendChild(statsDiv);
  
  // === Animation ===
  const clock = new THREE.Clock();
  let animId;
  let frameCount = 0;
  let lastStatsTime = 0;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    
    // Skip frames on low-end devices (target 30fps)
    frameCount++;
    if (detector.profile.tier === 'low' && frameCount % 2 !== 0) {
      return;
    }
    
    // Update adaptive quality
    adaptiveQuality.update();
    
    // Update touch controls
    touchControls.update();
    
    // Rotate instanced mesh
    instancedMesh.rotation.y = elapsed * 0.1;
    
    // Update stats
    if (elapsed - lastStatsTime > 1) {
      lastStatsTime = elapsed;
      const { fps, quality } = adaptiveQuality.stats;
      statsDiv.innerHTML = `
        FPS: ${fps}<br>
        Quality: ${quality}<br>
        Tier: ${detector.profile.tier}<br>
        Objects: ${OBJECT_COUNT}
      `;
    }
    
    renderer.render(scene, camera);
  }
  
  animate();
  
  // Handle orientation change
  window.addEventListener('orientationchange', () => {
    setTimeout(() => {
      const w = container.clientWidth;
      const h = container.clientHeight;
      camera.aspect = w / h;
      camera.updateProjectionMatrix();
      renderer.setSize(w, h);
      adaptiveQuality.baseWidth = w;
      adaptiveQuality.baseHeight = h;
    }, 100);
  });
  
  window.addEventListener('resize', () => {
    const w = container.clientWidth;
    const h = container.clientHeight;
    camera.aspect = w / h;
    camera.updateProjectionMatrix();
    renderer.setSize(w, h);
  });
  
  // Page visibility
  document.addEventListener('visibilitychange', () => {
    if (document.hidden) {
      cancelAnimationFrame(animId);
    } else {
      animate();
    }
  });
  
  return () => {
    cancelAnimationFrame(animId);
    touchControls.dispose();
    instancedGeo.dispose();
    instancedMat.dispose();
    renderer.dispose();
    container.removeChild(statsDiv);
  };
}
```

---

## 78.6 Mobile-Specific Tips

```javascript
// Mobile Best Practices

// 1. ลด pixel ratio
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));

// 2. ใช้ power preference
const renderer = new THREE.WebGLRenderer({
  canvas: canvas,
  powerPreference: 'low-power', // ประหยัดไฟ
  // powerPreference: 'high-performance', // ประสิทธิภาพสูง
});

// 3. ปิด features ที่ไม่จำเป็น
renderer.shadowMap.enabled = false; // ปิด shadows บน low-end

// 4. ลด geometry complexity
const sphere = new THREE.SphereGeometry(1, 16, 16); // 16 แทน 32

// 5. ใช้ MeshLambertMaterial แทน MeshStandardMaterial
const mat = new THREE.MeshLambertMaterial({ color: '#4ecdc4' }); // ถูกกว่า

// 6. Texture size
const texture = textureLoader.load('texture.jpg');
// ขนาด texture ไม่เกิน 1024x1024 สำหรับ mobile

// 7. ลด shadow map size
dirLight.shadow.mapSize.width = 512;
dirLight.shadow.mapSize.height = 512;

// 8. ใช้ frustum culling
mesh.frustumCulled = true; // default=true, ตรวจสอบว่าเปิดอยู่

// 9. Merge static geometries
import { mergeGeometries } from 'three/examples/jsm/utils/BufferGeometryUtils.js';
const merged = mergeGeometries([geo1, geo2, geo3]);

// 10. Page visibility API - หยุดเมื่อ tab ซ่อน
document.addEventListener('visibilitychange', () => {
  if (document.hidden) renderer.setAnimationLoop(null);
  else renderer.setAnimationLoop(animate);
});
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Auto Quality System
สร้าง system ที่:
- Benchmark device เมื่อเริ่ม
- เลือก quality preset อัตโนมัติ
- User override settings
- Save preference

### แบบฝึกหัดที่ 2: Mobile Portfolio
สร้าง portfolio ที่:
- Touch-friendly navigation
- Swipe gestures
- Pinch-to-zoom product viewer
- Fast load time

### แบบฝึกหัดที่ 3: Responsive 3D Scene
สร้าง scene ที่:
- ปรับ complexity ตาม viewport
- Portrait/landscape layout
- Touch, mouse, gamepad support
- Battery-aware

### แบบฝึกหัดที่ 4: Mobile AR Groundwork
เตรียม scene สำหรับ AR:
- Camera permissions
- Gyroscope-based view
- Touch raycast
- Anchored objects

---

## สรุป

ในบทนี้เราเรียนรู้:
1. **Device Detection** - ตรวจสอบ device tier
2. **Touch Controls** - Rotate/zoom ด้วย touch
3. **Adaptive Quality** - ปรับ quality ตาม FPS
4. **Battery API** - ประหยัดพลังงาน
5. **Mobile Best Practices** - Tips ทั้งหมด
6. **Instancing** - วิธี optimize บน mobile

บทถัดไปเราจะเรียน **VR Development with WebXR**
