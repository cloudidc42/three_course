# Part 67: Three.js กับ Vue.js

## บทนำ

Vue.js เป็น JavaScript framework ที่มีความยืดหยุ่นสูงและใช้งานง่าย การรวม Three.js กับ Vue 3 Composition API ทำให้เราสร้างฉาก 3D ที่ reactive และจัดการได้ง่ายขึ้นมาก ในบทนี้เราจะเรียนรู้วิธีการผสาน Three.js กับ Vue อย่างมีประสิทธิภาพ

---

## 67.1 Setup พื้นฐาน

```bash
# สร้าง Vue 3 project
npm create vite@latest my-vue-three -- --template vue
cd my-vue-three

# ติดตั้ง Three.js
npm install three

# ติดตั้ง optional helpers
npm install @vueuse/core  # Vue composables
npm install pinia          # State management
```

### Vue Component พื้นฐานกับ Three.js

```vue
<!-- src/components/BasicThreeScene.vue -->
<template>
  <div ref="containerRef" class="canvas-container" />
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';
import * as THREE from 'three';

const containerRef = ref(null);

let renderer, scene, camera, cube, animationId;

function init() {
  const container = containerRef.value;
  if (!container) return;

  // Scene
  scene = new THREE.Scene();
  scene.background = new THREE.Color('#1a1a2e');

  // Camera
  camera = new THREE.PerspectiveCamera(
    75,
    container.clientWidth / container.clientHeight,
    0.1,
    1000
  );
  camera.position.z = 5;

  // Renderer
  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  container.appendChild(renderer.domElement);

  // Objects
  const geometry = new THREE.BoxGeometry();
  const material = new THREE.MeshStandardMaterial({
    color: '#4ecdc4',
    metalness: 0.3,
    roughness: 0.4
  });
  cube = new THREE.Mesh(geometry, material);
  scene.add(cube);

  // Lights
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
  scene.add(ambientLight);

  const dirLight = new THREE.DirectionalLight(0xffffff, 1);
  dirLight.position.set(5, 5, 5);
  scene.add(dirLight);

  // Resize handler
  window.addEventListener('resize', handleResize);
  
  // Start animation
  animate();
}

function animate() {
  animationId = requestAnimationFrame(animate);
  cube.rotation.x += 0.01;
  cube.rotation.y += 0.01;
  renderer.render(scene, camera);
}

function handleResize() {
  if (!containerRef.value) return;
  const w = containerRef.value.clientWidth;
  const h = containerRef.value.clientHeight;
  
  camera.aspect = w / h;
  camera.updateProjectionMatrix();
  renderer.setSize(w, h);
}

function cleanup() {
  cancelAnimationFrame(animationId);
  window.removeEventListener('resize', handleResize);
  
  if (cube) {
    cube.geometry.dispose();
    cube.material.dispose();
    scene.remove(cube);
  }
  
  renderer.dispose();
  
  if (containerRef.value && renderer.domElement) {
    containerRef.value.removeChild(renderer.domElement);
  }
}

onMounted(init);
onBeforeUnmount(cleanup);
</script>

<style scoped>
.canvas-container {
  width: 100%;
  height: 100vh;
}
</style>
```

---

## 67.2 Composable useThree

สร้าง composable สำหรับ reuse Three.js logic

```javascript
// src/composables/useThree.js
import { ref, onMounted, onBeforeUnmount, shallowRef } from 'vue';
import * as THREE from 'three';

export function useThree(containerRef, options = {}) {
  const {
    antialias = true,
    shadows = false,
    backgroundColor = '#000000',
    cameraFov = 75,
    cameraNear = 0.1,
    cameraFar = 1000,
    cameraPosition = [0, 0, 5],
  } = options;

  // Use shallowRef เพื่อไม่ให้ Vue reactive ลึกเกินไป (performance)
  const renderer = shallowRef(null);
  const scene = shallowRef(null);
  const camera = shallowRef(null);
  const clock = shallowRef(null);
  
  const isReady = ref(false);
  let animationId = null;
  const callbacks = new Set();

  function addAnimationCallback(fn) {
    callbacks.add(fn);
    return () => callbacks.delete(fn);
  }

  function setupScene() {
    const container = containerRef.value;
    if (!container) return;

    // Scene
    scene.value = new THREE.Scene();
    scene.value.background = new THREE.Color(backgroundColor);

    // Camera
    camera.value = new THREE.PerspectiveCamera(
      cameraFov,
      container.clientWidth / container.clientHeight,
      cameraNear,
      cameraFar
    );
    camera.value.position.set(...cameraPosition);

    // Renderer
    renderer.value = new THREE.WebGLRenderer({ antialias });
    renderer.value.setSize(container.clientWidth, container.clientHeight);
    renderer.value.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    
    if (shadows) {
      renderer.value.shadowMap.enabled = true;
      renderer.value.shadowMap.type = THREE.PCFSoftShadowMap;
    }
    
    container.appendChild(renderer.value.domElement);

    // Clock
    clock.value = new THREE.Clock();

    isReady.value = true;
  }

  function startAnimation() {
    const loop = () => {
      animationId = requestAnimationFrame(loop);
      
      if (!renderer.value || !scene.value || !camera.value) return;
      
      const delta = clock.value.getDelta();
      const elapsed = clock.value.elapsedTime;
      
      // รัน callbacks ทั้งหมด
      callbacks.forEach(fn => fn(delta, elapsed));
      
      renderer.value.render(scene.value, camera.value);
    };
    loop();
  }

  function handleResize() {
    const container = containerRef.value;
    if (!container || !renderer.value || !camera.value) return;
    
    const w = container.clientWidth;
    const h = container.clientHeight;
    
    camera.value.aspect = w / h;
    camera.value.updateProjectionMatrix();
    renderer.value.setSize(w, h);
  }

  function dispose() {
    cancelAnimationFrame(animationId);
    window.removeEventListener('resize', handleResize);
    
    renderer.value?.dispose();
    
    if (containerRef.value && renderer.value?.domElement) {
      containerRef.value.removeChild(renderer.value.domElement);
    }
  }

  onMounted(() => {
    setupScene();
    startAnimation();
    window.addEventListener('resize', handleResize);
  });

  onBeforeUnmount(dispose);

  return {
    renderer,
    scene,
    camera,
    clock,
    isReady,
    addAnimationCallback,
  };
}
```

### ใช้ useThree Composable

```vue
<!-- src/components/ThreeWithComposable.vue -->
<template>
  <div ref="containerRef" class="canvas-container">
    <div v-if="!isReady" class="loading">
      Loading...
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue';
import * as THREE from 'three';
import { useThree } from '@/composables/useThree';

const containerRef = ref(null);

const { scene, isReady, addAnimationCallback } = useThree(containerRef, {
  backgroundColor: '#1a1a2e',
  cameraPosition: [0, 2, 8],
  shadows: true
});

// เพิ่ม objects หลังจาก scene พร้อม
watchEffect(() => {
  if (!isReady.value || !scene.value) return;
  
  // Lights
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
  scene.value.add(ambientLight);
  
  const dirLight = new THREE.DirectionalLight(0xffffff, 1);
  dirLight.position.set(5, 10, 5);
  dirLight.castShadow = true;
  scene.value.add(dirLight);
  
  // Objects
  const geometry = new THREE.BoxGeometry(1, 1, 1);
  const material = new THREE.MeshStandardMaterial({ color: '#4ecdc4' });
  const cube = new THREE.Mesh(geometry, material);
  cube.castShadow = true;
  scene.value.add(cube);
  
  // Animation
  addAnimationCallback((delta) => {
    cube.rotation.x += delta;
    cube.rotation.y += delta * 0.5;
  });
});
</script>
```

---

## 67.3 Vue 3 Reactive Scene

### Reactive Props กับ Three.js

```vue
<!-- src/components/ReactiveScene.vue -->
<template>
  <div class="scene-wrapper">
    <!-- Controls Panel -->
    <div class="controls-panel">
      <h3>Scene Controls</h3>
      
      <div class="control-group">
        <label>Color</label>
        <input type="color" v-model="config.color" />
      </div>
      
      <div class="control-group">
        <label>Speed: {{ config.speed.toFixed(1) }}</label>
        <input type="range" v-model.number="config.speed" min="0" max="5" step="0.1" />
      </div>
      
      <div class="control-group">
        <label>Scale: {{ config.scale.toFixed(1) }}</label>
        <input type="range" v-model.number="config.scale" min="0.1" max="3" step="0.1" />
      </div>
      
      <div class="control-group">
        <label>
          <input type="checkbox" v-model="config.wireframe" />
          Wireframe
        </label>
      </div>
      
      <div class="control-group">
        <label>Shape</label>
        <select v-model="config.shape">
          <option value="box">Box</option>
          <option value="sphere">Sphere</option>
          <option value="torus">Torus</option>
        </select>
      </div>
    </div>
    
    <!-- Canvas -->
    <div ref="containerRef" class="canvas-container" />
  </div>
</template>

<script setup>
import { ref, reactive, watch, watchEffect, onMounted, onBeforeUnmount } from 'vue';
import * as THREE from 'three';

const containerRef = ref(null);

// Reactive configuration
const config = reactive({
  color: '#4ecdc4',
  speed: 1,
  scale: 1,
  wireframe: false,
  shape: 'box'
});

let renderer, scene, camera, mesh, animationId;

// สร้าง geometry ตาม shape
function createGeometry(shape) {
  switch (shape) {
    case 'sphere': return new THREE.SphereGeometry(0.8, 32, 32);
    case 'torus': return new THREE.TorusGeometry(0.7, 0.3, 16, 100);
    default: return new THREE.BoxGeometry(1, 1, 1);
  }
}

function init() {
  if (!containerRef.value) return;
  const container = containerRef.value;
  
  scene = new THREE.Scene();
  scene.background = new THREE.Color('#1a1a2e');
  
  camera = new THREE.PerspectiveCamera(60, container.clientWidth / container.clientHeight, 0.1, 100);
  camera.position.set(0, 0, 5);
  
  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  container.appendChild(renderer.domElement);
  
  // Lights
  scene.add(new THREE.AmbientLight(0xffffff, 0.5));
  const light = new THREE.DirectionalLight(0xffffff, 1);
  light.position.set(5, 5, 5);
  scene.add(light);
  
  // Initial mesh
  const geometry = createGeometry(config.shape);
  const material = new THREE.MeshStandardMaterial({
    color: config.color,
    wireframe: config.wireframe,
    metalness: 0.3,
    roughness: 0.4
  });
  mesh = new THREE.Mesh(geometry, material);
  scene.add(mesh);
  
  // Start animation
  const clock = new THREE.Clock();
  const animate = () => {
    animationId = requestAnimationFrame(animate);
    const delta = clock.getDelta();
    
    mesh.rotation.x += delta * config.speed;
    mesh.rotation.y += delta * config.speed * 0.7;
    mesh.scale.setScalar(config.scale);
    
    renderer.render(scene, camera);
  };
  animate();
}

// Watch color changes
watch(() => config.color, (newColor) => {
  if (mesh) {
    mesh.material.color.set(newColor);
  }
});

// Watch wireframe changes  
watch(() => config.wireframe, (value) => {
  if (mesh) {
    mesh.material.wireframe = value;
  }
});

// Watch shape changes
watch(() => config.shape, (newShape) => {
  if (mesh && scene) {
    // Dispose old geometry
    mesh.geometry.dispose();
    // Create new geometry
    mesh.geometry = createGeometry(newShape);
  }
});

const handleResize = () => {
  if (!containerRef.value) return;
  const w = containerRef.value.clientWidth;
  const h = containerRef.value.clientHeight;
  camera.aspect = w / h;
  camera.updateProjectionMatrix();
  renderer.setSize(w, h);
};

onMounted(() => {
  init();
  window.addEventListener('resize', handleResize);
});

onBeforeUnmount(() => {
  cancelAnimationFrame(animationId);
  window.removeEventListener('resize', handleResize);
  mesh?.geometry.dispose();
  mesh?.material.dispose();
  renderer?.dispose();
});
</script>

<style scoped>
.scene-wrapper {
  display: flex;
  width: 100vw;
  height: 100vh;
}

.controls-panel {
  width: 250px;
  background: rgba(0, 0, 0, 0.8);
  color: white;
  padding: 20px;
  font-family: sans-serif;
  overflow-y: auto;
  flex-shrink: 0;
}

.controls-panel h3 {
  margin: 0 0 20px;
  color: #4ecdc4;
}

.control-group {
  margin-bottom: 16px;
}

.control-group label {
  display: block;
  margin-bottom: 6px;
  font-size: 0.85em;
  color: #aaa;
}

.control-group input[type="range"] {
  width: 100%;
}

.control-group select {
  width: 100%;
  background: #333;
  color: white;
  border: none;
  padding: 6px;
  border-radius: 4px;
}

.canvas-container {
  flex: 1;
}
</style>
```

---

## 67.4 Pinia Store สำหรับ 3D State

### Scene Store

```javascript
// src/stores/sceneStore.js
import { defineStore } from 'pinia';
import { ref, computed } from 'vue';

export const useSceneStore = defineStore('scene', () => {
  // State
  const objects = ref([]);
  const selectedObjectId = ref(null);
  const cameraPosition = ref([0, 2, 8]);
  const lights = ref({
    ambient: { intensity: 0.5, color: '#ffffff' },
    directional: { intensity: 1, position: [5, 10, 5], castShadow: true }
  });
  const sceneSettings = ref({
    background: '#1a1a2e',
    fog: { enabled: false, color: '#1a1a2e', near: 10, far: 50 }
  });

  // Getters
  const selectedObject = computed(() => 
    objects.value.find(o => o.id === selectedObjectId.value)
  );
  
  const objectCount = computed(() => objects.value.length);

  // Actions
  function addObject(objectData) {
    const id = Date.now().toString();
    objects.value.push({
      id,
      ...objectData,
    });
    return id;
  }

  function removeObject(id) {
    const index = objects.value.findIndex(o => o.id === id);
    if (index !== -1) {
      objects.value.splice(index, 1);
    }
    if (selectedObjectId.value === id) {
      selectedObjectId.value = null;
    }
  }

  function updateObject(id, updates) {
    const obj = objects.value.find(o => o.id === id);
    if (obj) {
      Object.assign(obj, updates);
    }
  }

  function selectObject(id) {
    selectedObjectId.value = id;
  }

  function clearSelection() {
    selectedObjectId.value = null;
  }

  function updateCamera(position) {
    cameraPosition.value = position;
  }

  function updateLight(type, updates) {
    if (lights.value[type]) {
      Object.assign(lights.value[type], updates);
    }
  }

  return {
    objects,
    selectedObjectId,
    selectedObject,
    objectCount,
    cameraPosition,
    lights,
    sceneSettings,
    addObject,
    removeObject,
    updateObject,
    selectObject,
    clearSelection,
    updateCamera,
    updateLight,
  };
});
```

### Component ที่ใช้ Store

```vue
<!-- src/components/StoreScene.vue -->
<template>
  <div class="app-layout">
    <!-- Sidebar -->
    <div class="sidebar">
      <h2>Scene Objects</h2>
      
      <!-- Add Object Buttons -->
      <div class="add-buttons">
        <button @click="addBox">+ Box</button>
        <button @click="addSphere">+ Sphere</button>
        <button @click="addTorus">+ Torus</button>
      </div>
      
      <!-- Object List -->
      <div class="object-list">
        <div
          v-for="obj in sceneStore.objects"
          :key="obj.id"
          class="object-item"
          :class="{ selected: sceneStore.selectedObjectId === obj.id }"
          @click="sceneStore.selectObject(obj.id)"
        >
          <span class="obj-name">{{ obj.name }}</span>
          <button @click.stop="sceneStore.removeObject(obj.id)">×</button>
        </div>
      </div>
      
      <!-- Selected Object Properties -->
      <div v-if="sceneStore.selectedObject" class="properties">
        <h3>Properties</h3>
        
        <label>Color</label>
        <input
          type="color"
          :value="sceneStore.selectedObject.color"
          @input="updateSelectedColor"
        />
        
        <label>Scale</label>
        <input
          type="range"
          :value="sceneStore.selectedObject.scale"
          @input="updateSelectedScale"
          min="0.1" max="3" step="0.1"
        />
        
        <label>Rotation Speed</label>
        <input
          type="range"
          :value="sceneStore.selectedObject.rotSpeed"
          @input="updateSelectedSpeed"
          min="0" max="5" step="0.1"
        />
      </div>
    </div>
    
    <!-- Canvas -->
    <div ref="containerRef" class="canvas-area" />
  </div>
</template>

<script setup>
import { ref, watch, onMounted, onBeforeUnmount } from 'vue';
import { useSceneStore } from '@/stores/sceneStore';
import * as THREE from 'three';

const containerRef = ref(null);
const sceneStore = useSceneStore();

let renderer, camera, scene, threeObjects = new Map(), animationId;
const clock = new THREE.Clock();

// Three.js initialization
function init() {
  const container = containerRef.value;
  if (!container) return;
  
  scene = new THREE.Scene();
  scene.background = new THREE.Color(sceneStore.sceneSettings.background);
  
  camera = new THREE.PerspectiveCamera(60, container.clientWidth / container.clientHeight, 0.1, 1000);
  camera.position.set(...sceneStore.cameraPosition);
  
  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.shadowMap.enabled = true;
  container.appendChild(renderer.domElement);
  
  // Default lighting
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
  scene.add(ambientLight);
  
  const dirLight = new THREE.DirectionalLight(0xffffff, 1);
  dirLight.position.set(5, 10, 5);
  scene.add(dirLight);
  
  // Ground
  const ground = new THREE.Mesh(
    new THREE.PlaneGeometry(20, 20),
    new THREE.MeshStandardMaterial({ color: '#333' })
  );
  ground.rotation.x = -Math.PI / 2;
  ground.position.y = -2;
  scene.add(ground);
  
  // Animation loop
  function animate() {
    animationId = requestAnimationFrame(animate);
    const delta = clock.getDelta();
    
    // อัพเดตแต่ละ object
    sceneStore.objects.forEach(objData => {
      const threeMesh = threeObjects.get(objData.id);
      if (threeMesh) {
        threeMesh.rotation.y += delta * (objData.rotSpeed || 1);
        threeMesh.scale.setScalar(objData.scale || 1);
        threeMesh.material.color.set(objData.color || '#4ecdc4');
      }
    });
    
    renderer.render(scene, camera);
  }
  animate();
}

// Watch store changes - sync Three.js objects
watch(() => sceneStore.objects, (newObjects) => {
  if (!scene) return;
  
  // ลบ objects ที่ไม่มีแล้ว
  threeObjects.forEach((mesh, id) => {
    if (!newObjects.find(o => o.id === id)) {
      scene.remove(mesh);
      mesh.geometry.dispose();
      mesh.material.dispose();
      threeObjects.delete(id);
    }
  });
  
  // เพิ่ม objects ใหม่
  newObjects.forEach(objData => {
    if (!threeObjects.has(objData.id)) {
      let geometry;
      switch (objData.type) {
        case 'sphere':
          geometry = new THREE.SphereGeometry(0.7, 32, 32);
          break;
        case 'torus':
          geometry = new THREE.TorusGeometry(0.6, 0.25, 16, 100);
          break;
        default:
          geometry = new THREE.BoxGeometry(1, 1, 1);
      }
      
      const material = new THREE.MeshStandardMaterial({
        color: objData.color,
        metalness: 0.3,
        roughness: 0.4
      });
      
      const mesh = new THREE.Mesh(geometry, material);
      mesh.position.set(
        (Math.random() - 0.5) * 8,
        (Math.random() - 0.5) * 3,
        (Math.random() - 0.5) * 4
      );
      
      scene.add(mesh);
      threeObjects.set(objData.id, mesh);
    }
  });
}, { deep: true });

// Action handlers
let objCounter = 0;

function addBox() {
  sceneStore.addObject({
    name: `Box ${++objCounter}`,
    type: 'box',
    color: `hsl(${Math.random() * 360}, 70%, 60%)`,
    scale: 1,
    rotSpeed: 1
  });
}

function addSphere() {
  sceneStore.addObject({
    name: `Sphere ${++objCounter}`,
    type: 'sphere',
    color: `hsl(${Math.random() * 360}, 70%, 60%)`,
    scale: 1,
    rotSpeed: 0.8
  });
}

function addTorus() {
  sceneStore.addObject({
    name: `Torus ${++objCounter}`,
    type: 'torus',
    color: `hsl(${Math.random() * 360}, 70%, 60%)`,
    scale: 1,
    rotSpeed: 1.2
  });
}

function updateSelectedColor(e) {
  if (sceneStore.selectedObjectId) {
    sceneStore.updateObject(sceneStore.selectedObjectId, { color: e.target.value });
  }
}

function updateSelectedScale(e) {
  if (sceneStore.selectedObjectId) {
    sceneStore.updateObject(sceneStore.selectedObjectId, { scale: parseFloat(e.target.value) });
  }
}

function updateSelectedSpeed(e) {
  if (sceneStore.selectedObjectId) {
    sceneStore.updateObject(sceneStore.selectedObjectId, { rotSpeed: parseFloat(e.target.value) });
  }
}

const handleResize = () => {
  if (!containerRef.value) return;
  const w = containerRef.value.clientWidth;
  const h = containerRef.value.clientHeight;
  camera.aspect = w / h;
  camera.updateProjectionMatrix();
  renderer.setSize(w, h);
};

onMounted(() => {
  init();
  window.addEventListener('resize', handleResize);
});

onBeforeUnmount(() => {
  cancelAnimationFrame(animationId);
  window.removeEventListener('resize', handleResize);
  threeObjects.forEach(mesh => {
    mesh.geometry.dispose();
    mesh.material.dispose();
  });
  renderer?.dispose();
});
</script>

<style scoped>
.app-layout {
  display: flex;
  width: 100vw;
  height: 100vh;
}

.sidebar {
  width: 260px;
  background: #1a1a2e;
  color: white;
  padding: 16px;
  font-family: sans-serif;
  overflow-y: auto;
  border-right: 1px solid #333;
  flex-shrink: 0;
}

.sidebar h2 {
  margin: 0 0 16px;
  font-size: 1em;
  color: #4ecdc4;
}

.add-buttons {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  margin-bottom: 16px;
}

.add-buttons button {
  padding: 6px 12px;
  background: #333;
  border: 1px solid #555;
  color: white;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.85em;
  transition: background 0.2s;
}

.add-buttons button:hover {
  background: #4ecdc4;
  color: black;
}

.object-list {
  margin-bottom: 20px;
}

.object-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 10px;
  background: #2a2a3e;
  border-radius: 6px;
  margin-bottom: 4px;
  cursor: pointer;
  transition: background 0.2s;
  font-size: 0.85em;
}

.object-item:hover {
  background: #3a3a4e;
}

.object-item.selected {
  background: #1a4a4a;
  border: 1px solid #4ecdc4;
}

.object-item button {
  background: none;
  border: none;
  color: #ff6b6b;
  cursor: pointer;
  font-size: 1.2em;
  padding: 0;
  line-height: 1;
}

.properties {
  border-top: 1px solid #333;
  padding-top: 16px;
}

.properties h3 {
  margin: 0 0 12px;
  font-size: 0.9em;
  color: #4ecdc4;
}

.properties label {
  display: block;
  font-size: 0.8em;
  color: #888;
  margin: 10px 0 4px;
}

.properties input[type="range"] {
  width: 100%;
}

.canvas-area {
  flex: 1;
}
</style>
```

---

## 67.5 Vue Router กับ 3D Transitions

```javascript
// src/router/index.js
import { createRouter, createWebHistory } from 'vue-router';

const routes = [
  { path: '/', component: () => import('@/views/HomeView.vue') },
  { path: '/gallery', component: () => import('@/views/GalleryView.vue') },
  { path: '/about', component: () => import('@/views/AboutView.vue') },
];

export const router = createRouter({
  history: createWebHistory(),
  routes,
});
```

```vue
<!-- src/App.vue - กับ 3D Transition -->
<template>
  <div class="app">
    <!-- Navigation -->
    <nav>
      <router-link to="/" @click="navigate('/')">Home</router-link>
      <router-link to="/gallery" @click="navigate('/gallery')">Gallery</router-link>
      <router-link to="/about" @click="navigate('/about')">About</router-link>
    </nav>
    
    <!-- 3D Transition Canvas (อยู่เบื้องหลัง) -->
    <div ref="transitionCanvas" class="transition-canvas" />
    
    <!-- Page Content -->
    <Transition name="page" @enter="onEnter" @leave="onLeave">
      <RouterView />
    </Transition>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import { useRouter } from 'vue-router';
import * as THREE from 'three';

const transitionCanvas = ref(null);
const isTransitioning = ref(false);

let renderer, scene, camera, particles, animationId;

// Setup transition scene
function setupTransition() {
  const container = transitionCanvas.value;
  if (!container) return;
  
  scene = new THREE.Scene();
  camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 100);
  camera.position.z = 5;
  
  renderer = new THREE.WebGLRenderer({ alpha: true, antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setClearColor(0x000000, 0);
  container.appendChild(renderer.domElement);
  
  // Particles
  const positions = new Float32Array(3000 * 3);
  for (let i = 0; i < 3000; i++) {
    positions[i * 3] = (Math.random() - 0.5) * 20;
    positions[i * 3 + 1] = (Math.random() - 0.5) * 20;
    positions[i * 3 + 2] = (Math.random() - 0.5) * 20;
  }
  
  const geo = new THREE.BufferGeometry();
  geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
  
  particles = new THREE.Points(
    geo,
    new THREE.PointsMaterial({ 
      size: 0.05, 
      color: '#4ecdc4', 
      transparent: true, 
      opacity: 0 
    })
  );
  scene.add(particles);
  
  const clock = new THREE.Clock();
  const animate = () => {
    animationId = requestAnimationFrame(animate);
    const delta = clock.getDelta();
    if (isTransitioning.value && particles) {
      particles.rotation.y += delta * 2;
    }
    renderer.render(scene, camera);
  };
  animate();
}

// Trigger transition effect
function triggerTransition(duration = 600) {
  if (!particles) return;
  
  isTransitioning.value = true;
  
  // Fade in particles
  const startTime = Date.now();
  const fadeIn = () => {
    const t = (Date.now() - startTime) / (duration / 2);
    if (particles && t < 1) {
      particles.material.opacity = t * 0.8;
      requestAnimationFrame(fadeIn);
    } else if (particles) {
      particles.material.opacity = 0.8;
      // Fade out
      const fadeOutStart = Date.now();
      const fadeOut = () => {
        const t2 = (Date.now() - fadeOutStart) / (duration / 2);
        if (particles && t2 < 1) {
          particles.material.opacity = (1 - t2) * 0.8;
          requestAnimationFrame(fadeOut);
        } else if (particles) {
          particles.material.opacity = 0;
          isTransitioning.value = false;
        }
      };
      fadeOut();
    }
  };
  fadeIn();
}

// Hooks สำหรับ router transition
const navigate = () => {
  triggerTransition();
};

onMounted(() => {
  setupTransition();
  window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  });
});
</script>

<style>
.app {
  position: relative;
  width: 100vw;
  height: 100vh;
}

.transition-canvas {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 100;
}

nav {
  position: fixed;
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 200;
  display: flex;
  gap: 20px;
}

nav a {
  color: white;
  text-decoration: none;
  padding: 8px 16px;
  background: rgba(0,0,0,0.5);
  border-radius: 8px;
}

.page-enter-active,
.page-leave-active {
  transition: opacity 0.3s;
}

.page-enter-from,
.page-leave-to {
  opacity: 0;
}
</style>
```

---

## 67.6 Vue 3 Product Viewer Component

```vue
<!-- src/components/ProductViewer.vue -->
<template>
  <div class="product-viewer">
    <!-- 3D View -->
    <div ref="viewerRef" class="viewer-canvas" />
    
    <!-- Overlay Controls -->
    <div class="viewer-controls">
      <button class="btn-icon" @click="resetCamera" title="Reset View">
        ↺
      </button>
      <button class="btn-icon" @click="toggleAutoRotate" title="Auto Rotate">
        {{ autoRotate ? '⏸' : '▶' }}
      </button>
      <button class="btn-icon" @click="toggleWireframe" title="Wireframe">
        ⬡
      </button>
    </div>
    
    <!-- Color Swatches -->
    <div class="color-swatches">
      <button
        v-for="color in availableColors"
        :key="color.id"
        class="swatch"
        :style="{ background: color.hex }"
        :class="{ active: currentColor === color.id }"
        @click="setColor(color)"
        :title="color.name"
      />
    </div>
    
    <!-- Product Info -->
    <div class="product-info">
      <h2>{{ productName }}</h2>
      <p>{{ productDescription }}</p>
    </div>
  </div>
</template>

<script setup>
import { ref, watch, onMounted, onBeforeUnmount, defineProps, defineEmits } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

const props = defineProps({
  productName: { type: String, default: 'Product Name' },
  productDescription: { type: String, default: '' },
});

const emit = defineEmits(['colorChange']);

// State
const viewerRef = ref(null);
const autoRotate = ref(true);
const currentColor = ref('blue');
const availableColors = ref([
  { id: 'blue', name: 'Ocean Blue', hex: '#4ecdc4' },
  { id: 'red', name: 'Sunset Red', hex: '#ff6b6b' },
  { id: 'gold', name: 'Solar Gold', hex: '#ffd700' },
  { id: 'purple', name: 'Cosmic Purple', hex: '#9b59b6' },
  { id: 'green', name: 'Forest Green', hex: '#2ecc71' },
]);

// Three.js
let renderer, scene, camera, controls, productMesh, animationId;

function createProductGeometry() {
  // Complex product shape (as example: rounded box using LatheGeometry)
  const points = [
    new THREE.Vector2(0.3, 0),
    new THREE.Vector2(0.5, 0.2),
    new THREE.Vector2(0.6, 0.5),
    new THREE.Vector2(0.6, 1.5),
    new THREE.Vector2(0.5, 1.8),
    new THREE.Vector2(0.3, 2.0),
  ];
  return new THREE.LatheGeometry(points, 32);
}

function init() {
  const container = viewerRef.value;
  if (!container) return;
  
  // Scene
  scene = new THREE.Scene();
  scene.background = new THREE.Color('#f0f0f0');
  
  // Camera
  camera = new THREE.PerspectiveCamera(
    45,
    container.clientWidth / container.clientHeight,
    0.1,
    100
  );
  camera.position.set(3, 2, 4);
  
  // Renderer
  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.shadowMap.enabled = true;
  renderer.shadowMap.type = THREE.PCFSoftShadowMap;
  renderer.toneMapping = THREE.ACESFilmicToneMapping;
  renderer.toneMappingExposure = 1;
  container.appendChild(renderer.domElement);
  
  // Controls
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
  controls.autoRotate = autoRotate.value;
  controls.autoRotateSpeed = 2;
  controls.minDistance = 2;
  controls.maxDistance = 10;
  
  // Lights
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
  scene.add(ambientLight);
  
  const dirLight1 = new THREE.DirectionalLight(0xffffff, 1);
  dirLight1.position.set(5, 10, 5);
  dirLight1.castShadow = true;
  scene.add(dirLight1);
  
  const dirLight2 = new THREE.DirectionalLight(0x8080ff, 0.3);
  dirLight2.position.set(-5, -5, -5);
  scene.add(dirLight2);
  
  // Product
  const geometry = createProductGeometry();
  const material = new THREE.MeshStandardMaterial({
    color: '#4ecdc4',
    metalness: 0.7,
    roughness: 0.2,
  });
  productMesh = new THREE.Mesh(geometry, material);
  productMesh.castShadow = true;
  productMesh.position.y = -1;
  scene.add(productMesh);
  
  // Ground shadow catcher
  const groundMesh = new THREE.Mesh(
    new THREE.CircleGeometry(3, 64),
    new THREE.ShadowMaterial({ opacity: 0.3 })
  );
  groundMesh.rotation.x = -Math.PI / 2;
  groundMesh.position.y = -1;
  groundMesh.receiveShadow = true;
  scene.add(groundMesh);
  
  // Grid
  const grid = new THREE.GridHelper(6, 12, '#ccc', '#eee');
  grid.position.y = -1;
  scene.add(grid);
  
  // Animation
  const clock = new THREE.Clock();
  const animate = () => {
    animationId = requestAnimationFrame(animate);
    const delta = clock.getDelta();
    controls.update();
    renderer.render(scene, camera);
  };
  animate();
  
  // Resize
  window.addEventListener('resize', handleResize);
}

function handleResize() {
  if (!viewerRef.value) return;
  const w = viewerRef.value.clientWidth;
  const h = viewerRef.value.clientHeight;
  camera.aspect = w / h;
  camera.updateProjectionMatrix();
  renderer.setSize(w, h);
}

function resetCamera() {
  if (!camera || !controls) return;
  camera.position.set(3, 2, 4);
  controls.target.set(0, 0, 0);
  controls.update();
}

function toggleAutoRotate() {
  autoRotate.value = !autoRotate.value;
  if (controls) {
    controls.autoRotate = autoRotate.value;
  }
}

function toggleWireframe() {
  if (productMesh) {
    productMesh.material.wireframe = !productMesh.material.wireframe;
  }
}

function setColor(color) {
  currentColor.value = color.id;
  if (productMesh) {
    productMesh.material.color.set(color.hex);
  }
  emit('colorChange', color);
}

onMounted(init);
onBeforeUnmount(() => {
  cancelAnimationFrame(animationId);
  window.removeEventListener('resize', handleResize);
  productMesh?.geometry.dispose();
  productMesh?.material.dispose();
  renderer?.dispose();
  controls?.dispose();
});
</script>

<style scoped>
.product-viewer {
  position: relative;
  width: 100%;
  height: 100%;
  min-height: 400px;
}

.viewer-canvas {
  width: 100%;
  height: 100%;
}

.viewer-controls {
  position: absolute;
  top: 12px;
  right: 12px;
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.btn-icon {
  width: 36px;
  height: 36px;
  background: rgba(255, 255, 255, 0.9);
  border: none;
  border-radius: 50%;
  cursor: pointer;
  font-size: 1.1em;
  box-shadow: 0 2px 8px rgba(0,0,0,0.15);
  display: flex;
  align-items: center;
  justify-content: center;
  transition: transform 0.2s;
}

.btn-icon:hover {
  transform: scale(1.1);
}

.color-swatches {
  position: absolute;
  bottom: 80px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 8px;
  background: rgba(255,255,255,0.9);
  padding: 8px 12px;
  border-radius: 30px;
  box-shadow: 0 4px 15px rgba(0,0,0,0.1);
}

.swatch {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  border: 2px solid transparent;
  cursor: pointer;
  transition: transform 0.2s, border-color 0.2s;
}

.swatch:hover {
  transform: scale(1.2);
}

.swatch.active {
  border-color: white;
  box-shadow: 0 0 0 2px #333;
  transform: scale(1.1);
}

.product-info {
  position: absolute;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
  text-align: center;
  font-family: sans-serif;
}

.product-info h2 {
  margin: 0 0 4px;
  font-size: 1.1em;
  color: #333;
}

.product-info p {
  margin: 0;
  font-size: 0.85em;
  color: #666;
}
</style>
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Vue Composable
สร้าง composable `useScene` ที่:
- จัดการ lifecycle Three.js ทั้งหมด
- Support multiple render targets
- Event system (on/off)
- Debug mode ด้วย stats panel

### แบบฝึกหัดที่ 2: Pinia + Three.js
สร้าง 3D scene editor ด้วย Pinia ที่:
- เพิ่ม/ลบ/แก้ไข objects
- Undo/Redo operations
- Save/Load scene เป็น JSON
- Share scene ผ่าน URL

### แบบฝึกหัดที่ 3: Vue Router Transitions
สร้าง multi-page 3D site ที่:
- แต่ละ page มี unique 3D scene
- Transition effects ระหว่าง pages
- Shared 3D background
- Preserve camera state

### แบบฝึกหัดที่ 4: Product Configurator
สร้าง product configurator Vue app ที่:
- เลือก model/shape
- เลือกสี/material
- เพิ่ม accessories
- บันทึก config
- Share ผ่าน URL

### แบบฝึกหัดที่ 5: Data Visualization
สร้าง 3D data visualization ที่:
- โหลด data จาก API
- สร้าง 3D bar chart/scatter plot
- Interactive (hover/click แสดง details)
- Real-time update ด้วย Vue reactivity

---

## สรุป

ในบทนี้เราเรียนรู้:
1. **Vue + Three.js Basics** - Setup พื้นฐานด้วย Composition API
2. **useThree Composable** - Reusable Three.js logic
3. **Reactive Scene** - Sync Vue state กับ Three.js
4. **Pinia Store** - State management สำหรับ 3D scene
5. **Vue Router** - 3D transitions ระหว่าง pages
6. **Product Viewer** - Complete component example

บทถัดไปเราจะเรียน **Three.js กับ TypeScript**
