# Part 68: Three.js กับ TypeScript

## บทนำ

TypeScript ช่วยให้การพัฒนา Three.js มีความปลอดภัยมากขึ้นด้วย static type checking ลด bugs ที่เกิดจาก type errors และทำให้ IDE suggestion ทำงานได้ดียิ่งขึ้น ในบทนี้เราจะเรียนรู้วิธีใช้ TypeScript กับ Three.js อย่างถูกต้องและมีประสิทธิภาพ

---

## 68.1 Setup TypeScript Project

```bash
# สร้าง Vite + TypeScript project
npm create vite@latest my-ts-three -- --template vanilla-ts
cd my-ts-three

# ติดตั้ง Three.js และ types
npm install three
npm install -D @types/three

# ตรวจสอบ tsconfig.json
cat tsconfig.json
```

### tsconfig.json ที่แนะนำ

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "useDefineForClassFields": true,
    "module": "ESNext",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "strict": true,             // เปิด strict mode เสมอ
    "noUnusedLocals": true,     // ไม่อนุญาต unused variables
    "noUnusedParameters": true, // ไม่อนุญาต unused parameters
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,  // Array access อาจเป็น undefined
    "exactOptionalPropertyTypes": true, // strict optional properties
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

---

## 68.2 Type Definitions สำหรับ Three.js

### Import Types

```typescript
// src/types/three-types.ts
import type {
  Scene,
  PerspectiveCamera,
  OrthographicCamera,
  WebGLRenderer,
  Mesh,
  Group,
  Object3D,
  BufferGeometry,
  Material,
  MeshStandardMaterial,
  MeshBasicMaterial,
  Vector3,
  Color,
  Texture,
  Clock,
  AnimationMixer,
  AnimationClip,
} from 'three';

// Re-export ชัดเจน
export type {
  Scene,
  PerspectiveCamera,
  WebGLRenderer,
  Mesh,
  Vector3,
  Color,
};

// Custom types สำหรับโปรเจกต์
export type Camera = PerspectiveCamera | OrthographicCamera;

export interface SceneContext {
  scene: Scene;
  camera: Camera;
  renderer: WebGLRenderer;
  clock: Clock;
}

export interface AnimationContext extends SceneContext {
  mixer?: AnimationMixer;
  clips?: AnimationClip[];
}
```

---

## 68.3 Typed Scene Setup

```typescript
// src/core/scene.ts
import * as THREE from 'three';
import type { SceneContext } from '../types/three-types';

interface SceneOptions {
  width?: number;
  height?: number;
  antialias?: boolean;
  backgroundColor?: string | number;
  shadows?: boolean;
  pixelRatio?: number;
}

export function createScene(
  container: HTMLElement,
  options: SceneOptions = {}
): SceneContext {
  const {
    width = container.clientWidth,
    height = container.clientHeight,
    antialias = true,
    backgroundColor = '#000000',
    shadows = false,
    pixelRatio = Math.min(window.devicePixelRatio, 2),
  } = options;

  // Scene
  const scene = new THREE.Scene();
  scene.background = new THREE.Color(backgroundColor);

  // Camera
  const camera = new THREE.PerspectiveCamera(
    75,
    width / height,
    0.1,
    1000
  );
  camera.position.set(0, 0, 5);

  // Renderer
  const renderer = new THREE.WebGLRenderer({ antialias });
  renderer.setSize(width, height);
  renderer.setPixelRatio(pixelRatio);
  
  if (shadows) {
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
  }
  
  container.appendChild(renderer.domElement);

  // Clock
  const clock = new THREE.Clock();

  return { scene, camera, renderer, clock };
}
```

---

## 68.4 Typed GLTF Loader

```typescript
// src/loaders/gltfLoader.ts
import { GLTFLoader, type GLTF } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js';
import * as THREE from 'three';

// Type สำหรับ GLTF nodes ที่เราต้องการ
interface CharacterGLTF {
  scene: THREE.Group;
  animations: THREE.AnimationClip[];
  cameras: THREE.Camera[];
}

// Type guard สำหรับ GLTF
function isCharacterGLTF(gltf: GLTF): gltf is GLTF & CharacterGLTF {
  return gltf.scene !== undefined && Array.isArray(gltf.animations);
}

// Typed loader function
export async function loadGLTF(url: string): Promise<GLTF> {
  return new Promise((resolve, reject) => {
    const loader = new GLTFLoader();
    
    // Optional: Draco decompression
    const dracoLoader = new DRACOLoader();
    dracoLoader.setDecoderPath('/draco/');
    loader.setDRACOLoader(dracoLoader);
    
    loader.load(
      url,
      (gltf) => resolve(gltf),
      undefined,
      (error) => reject(new Error(`Failed to load GLTF: ${url} - ${error.message}`))
    );
  });
}

// Typed bone access
interface CharacterNodes {
  root: THREE.Bone;
  spine: THREE.Bone;
  head: THREE.Bone;
  leftArm: THREE.Bone;
  rightArm: THREE.Bone;
}

export function extractCharacterNodes(scene: THREE.Group): Partial<CharacterNodes> {
  const nodes: Partial<CharacterNodes> = {};
  
  scene.traverse((child) => {
    if (child instanceof THREE.Bone) {
      switch (child.name) {
        case 'Root':
        case 'root':
        case 'mixamorigHips':
          nodes.root = child;
          break;
        case 'Spine':
        case 'mixamorigSpine':
          nodes.spine = child;
          break;
        case 'Head':
        case 'mixamorigHead':
          nodes.head = child;
          break;
        case 'LeftArm':
        case 'mixamorigLeftArm':
          nodes.leftArm = child;
          break;
        case 'RightArm':
        case 'mixamorigRightArm':
          nodes.rightArm = child;
          break;
      }
    }
  });
  
  return nodes;
}
```

---

## 68.5 Type-safe Shader Uniforms

```typescript
// src/shaders/types.ts
import * as THREE from 'three';

// Interface สำหรับ uniform values
interface UniformValue<T> {
  value: T;
}

// Shader uniforms interface
interface CustomShaderUniforms {
  uTime: UniformValue<number>;
  uResolution: UniformValue<THREE.Vector2>;
  uColor: UniformValue<THREE.Color>;
  uTexture: UniformValue<THREE.Texture | null>;
  uOpacity: UniformValue<number>;
  uNoiseScale: UniformValue<number>;
}

// สร้าง uniforms ด้วย type safety
function createShaderUniforms(): CustomShaderUniforms {
  return {
    uTime: { value: 0 },
    uResolution: { value: new THREE.Vector2(window.innerWidth, window.innerHeight) },
    uColor: { value: new THREE.Color('#4ecdc4') },
    uTexture: { value: null },
    uOpacity: { value: 1.0 },
    uNoiseScale: { value: 1.0 },
  };
}

// Vertex Shader
const vertexShader: string = /* glsl */`
  varying vec2 vUv;
  varying vec3 vNormal;
  varying vec3 vPosition;
  
  uniform float uTime;
  uniform float uNoiseScale;
  
  void main() {
    vUv = uv;
    vNormal = normalize(normalMatrix * normal);
    vPosition = position;
    
    vec3 pos = position;
    pos.y += sin(pos.x * uNoiseScale + uTime) * 0.1;
    
    gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
  }
`;

// Fragment Shader
const fragmentShader: string = /* glsl */`
  uniform float uTime;
  uniform vec2 uResolution;
  uniform vec3 uColor;
  uniform float uOpacity;
  
  varying vec2 vUv;
  varying vec3 vNormal;
  
  void main() {
    float fresnel = dot(vNormal, vec3(0.0, 0.0, 1.0));
    fresnel = pow(1.0 - fresnel, 3.0);
    
    vec3 color = mix(uColor, vec3(1.0), fresnel * 0.5);
    
    gl_FragColor = vec4(color, uOpacity);
  }
`;

// สร้าง typed ShaderMaterial
function createTypedShaderMaterial(
  overrides: Partial<CustomShaderUniforms> = {}
): THREE.ShaderMaterial {
  const uniforms = createShaderUniforms();
  
  // Apply overrides
  Object.entries(overrides).forEach(([key, value]) => {
    const k = key as keyof CustomShaderUniforms;
    if (uniforms[k] !== undefined) {
      (uniforms[k] as UniformValue<unknown>).value = (value as UniformValue<unknown>).value;
    }
  });
  
  return new THREE.ShaderMaterial({
    uniforms,
    vertexShader,
    fragmentShader,
    transparent: uniforms.uOpacity.value < 1,
  });
}

// Update uniforms ด้วย type safety
function updateShaderUniforms(
  material: THREE.ShaderMaterial,
  updates: Partial<Record<keyof CustomShaderUniforms, unknown>>
): void {
  Object.entries(updates).forEach(([key, value]) => {
    if (material.uniforms[key] !== undefined) {
      material.uniforms[key].value = value;
    }
  });
}

// ใช้งาน
const material = createTypedShaderMaterial({
  uColor: { value: new THREE.Color('hotpink') },
  uOpacity: { value: 0.8 }
});

// Animation loop update
function animate(clock: THREE.Clock, material: THREE.ShaderMaterial): void {
  updateShaderUniforms(material, {
    uTime: clock.getElapsedTime()
  });
}

export { createTypedShaderMaterial, updateShaderUniforms, CustomShaderUniforms };
```

---

## 68.6 Generic Types กับ Three.js

```typescript
// src/utils/sceneUtils.ts
import * as THREE from 'three';

// Generic function สำหรับหา objects ใน scene
function findObjectByName<T extends THREE.Object3D>(
  root: THREE.Object3D,
  name: string
): T | undefined {
  const found = root.getObjectByName(name);
  if (found instanceof THREE.Object3D) {
    return found as T;
  }
  return undefined;
}

// Generic function สำหรับหา objects ทั้งหมดที่ตรงกับ type
function findObjectsByType<T extends THREE.Object3D>(
  root: THREE.Object3D,
  type: new (...args: unknown[]) => T
): T[] {
  const results: T[] = [];
  root.traverse((child) => {
    if (child instanceof type) {
      results.push(child);
    }
  });
  return results;
}

// ใช้งาน
// const mesh = findObjectByName<THREE.Mesh>(scene, 'character');
// const lights = findObjectsByType<THREE.PointLight>(scene, THREE.PointLight);

// Generic event emitter สำหรับ scene events
type EventMap = {
  objectSelected: { object: THREE.Object3D; point: THREE.Vector3 };
  objectHovered: { object: THREE.Object3D | null };
  sceneReady: { scene: THREE.Scene };
  frameUpdate: { delta: number; elapsed: number };
};

type EventCallback<T extends keyof EventMap> = (data: EventMap[T]) => void;

class TypedEventEmitter {
  private listeners: Partial<{
    [K in keyof EventMap]: Set<EventCallback<K>>;
  }> = {};

  on<T extends keyof EventMap>(event: T, callback: EventCallback<T>): void {
    if (!this.listeners[event]) {
      this.listeners[event] = new Set() as Set<EventCallback<T>>;
    }
    (this.listeners[event] as Set<EventCallback<T>>).add(callback);
  }

  off<T extends keyof EventMap>(event: T, callback: EventCallback<T>): void {
    (this.listeners[event] as Set<EventCallback<T>> | undefined)?.delete(callback);
  }

  emit<T extends keyof EventMap>(event: T, data: EventMap[T]): void {
    (this.listeners[event] as Set<EventCallback<T>> | undefined)?.forEach(cb => cb(data));
  }
}
```

---

## 68.7 Discriminated Unions สำหรับ Material Types

```typescript
// src/types/materials.ts
import * as THREE from 'three';

// Material type definitions
type BasicMaterialConfig = {
  type: 'basic';
  color: string;
  wireframe?: boolean;
  transparent?: boolean;
  opacity?: number;
};

type StandardMaterialConfig = {
  type: 'standard';
  color: string;
  metalness: number;
  roughness: number;
  emissive?: string;
  emissiveIntensity?: number;
  map?: THREE.Texture;
  normalMap?: THREE.Texture;
};

type PhysicalMaterialConfig = {
  type: 'physical';
  color: string;
  metalness: number;
  roughness: number;
  clearcoat: number;
  clearcoatRoughness: number;
  transmission?: number;
  thickness?: number;
};

type ShaderMaterialConfig = {
  type: 'shader';
  vertexShader: string;
  fragmentShader: string;
  uniforms: Record<string, THREE.IUniform>;
};

// Discriminated union
type MaterialConfig =
  | BasicMaterialConfig
  | StandardMaterialConfig
  | PhysicalMaterialConfig
  | ShaderMaterialConfig;

// Factory function ด้วย type narrowing
function createMaterial(config: MaterialConfig): THREE.Material {
  switch (config.type) {
    case 'basic':
      return new THREE.MeshBasicMaterial({
        color: config.color,
        wireframe: config.wireframe,
        transparent: config.transparent,
        opacity: config.opacity,
      });

    case 'standard':
      return new THREE.MeshStandardMaterial({
        color: config.color,
        metalness: config.metalness,
        roughness: config.roughness,
        emissive: config.emissive,
        emissiveIntensity: config.emissiveIntensity,
        map: config.map,
        normalMap: config.normalMap,
      });

    case 'physical':
      return new THREE.MeshPhysicalMaterial({
        color: config.color,
        metalness: config.metalness,
        roughness: config.roughness,
        clearcoat: config.clearcoat,
        clearcoatRoughness: config.clearcoatRoughness,
        transmission: config.transmission,
        thickness: config.thickness,
      });

    case 'shader':
      return new THREE.ShaderMaterial({
        vertexShader: config.vertexShader,
        fragmentShader: config.fragmentShader,
        uniforms: config.uniforms,
      });
  }
}

// ใช้งาน - TypeScript รู้จัก type ทันที
const configs: MaterialConfig[] = [
  { type: 'basic', color: '#ff0000' },
  { type: 'standard', color: '#00ff00', metalness: 0.5, roughness: 0.3 },
  { type: 'physical', color: '#0000ff', metalness: 0.8, roughness: 0.1, clearcoat: 1, clearcoatRoughness: 0.1 },
];

const materials = configs.map(createMaterial);
```

---

## 68.8 Strict Null Checks กับ Three.js

```typescript
// src/core/renderer.ts
import * as THREE from 'three';

class TypeSafeRenderer {
  private renderer: THREE.WebGLRenderer | null = null;
  private scene: THREE.Scene | null = null;
  private camera: THREE.PerspectiveCamera | null = null;
  private animationId: number | null = null;
  private container: HTMLElement | null = null;

  // Type-safe initialization
  initialize(container: HTMLElement): void {
    this.container = container;
    
    this.scene = new THREE.Scene();
    
    this.camera = new THREE.PerspectiveCamera(
      75,
      container.clientWidth / container.clientHeight,
      0.1,
      1000
    );
    this.camera.position.z = 5;
    
    this.renderer = new THREE.WebGLRenderer({ antialias: true });
    this.renderer.setSize(container.clientWidth, container.clientHeight);
    container.appendChild(this.renderer.domElement);
  }

  // Null-safe getter
  getScene(): THREE.Scene {
    if (!this.scene) {
      throw new Error('Renderer not initialized. Call initialize() first.');
    }
    return this.scene;
  }

  getCamera(): THREE.PerspectiveCamera {
    if (!this.camera) {
      throw new Error('Renderer not initialized. Call initialize() first.');
    }
    return this.camera;
  }

  // Optional chaining สำหรับ safe access
  getRendererInfo(): string {
    return this.renderer?.info.render.calls.toString() ?? 'Not initialized';
  }

  // Safe add object
  addObject(object: THREE.Object3D): boolean {
    if (!this.scene) return false;
    this.scene.add(object);
    return true;
  }

  // Safe resize
  resize(width?: number, height?: number): void {
    if (!this.renderer || !this.camera || !this.container) return;
    
    const w = width ?? this.container.clientWidth;
    const h = height ?? this.container.clientHeight;
    
    this.camera.aspect = w / h;
    this.camera.updateProjectionMatrix();
    this.renderer.setSize(w, h);
  }

  // Safe render
  render(): void {
    if (!this.renderer || !this.scene || !this.camera) return;
    this.renderer.render(this.scene, this.camera);
  }

  // Type-safe dispose
  dispose(): void {
    if (this.animationId !== null) {
      cancelAnimationFrame(this.animationId);
      this.animationId = null;
    }

    if (this.renderer) {
      this.renderer.dispose();
      
      if (this.container && this.renderer.domElement.parentNode === this.container) {
        this.container.removeChild(this.renderer.domElement);
      }
      
      this.renderer = null;
    }

    this.scene = null;
    this.camera = null;
    this.container = null;
  }
}
```

---

## 68.9 โปรเจกต์: Complete TypeScript Three.js Project

```typescript
// src/main.ts
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

// === Type Definitions ===

interface SceneConfig {
  backgroundColor: string;
  fog?: {
    color: string;
    near: number;
    far: number;
  };
}

interface ObjectDefinition {
  id: string;
  type: 'box' | 'sphere' | 'torus' | 'cylinder';
  position: [number, number, number];
  color: string;
  metalness: number;
  roughness: number;
  rotationSpeed: [number, number, number];
  castShadow?: boolean;
}

interface AppState {
  isInitialized: boolean;
  isAnimating: boolean;
  selectedObjectId: string | null;
}

// === Scene Manager ===

class SceneManager {
  private renderer!: THREE.WebGLRenderer;
  private scene!: THREE.Scene;
  private camera!: THREE.PerspectiveCamera;
  private controls!: OrbitControls;
  private clock: THREE.Clock;
  private objects: Map<string, THREE.Mesh>;
  private state: AppState;
  private animationId: number | null = null;
  private raycaster: THREE.Raycaster;
  private pointer: THREE.Vector2;

  constructor() {
    this.clock = new THREE.Clock();
    this.objects = new Map();
    this.raycaster = new THREE.Raycaster();
    this.pointer = new THREE.Vector2();
    this.state = {
      isInitialized: false,
      isAnimating: false,
      selectedObjectId: null,
    };
  }

  initialize(container: HTMLElement, config: SceneConfig): void {
    // Scene setup
    this.scene = new THREE.Scene();
    this.scene.background = new THREE.Color(config.backgroundColor);
    
    if (config.fog) {
      this.scene.fog = new THREE.Fog(
        config.fog.color,
        config.fog.near,
        config.fog.far
      );
    }

    // Camera
    this.camera = new THREE.PerspectiveCamera(
      60,
      container.clientWidth / container.clientHeight,
      0.1,
      1000
    );
    this.camera.position.set(5, 5, 10);

    // Renderer
    this.renderer = new THREE.WebGLRenderer({ antialias: true });
    this.renderer.setSize(container.clientWidth, container.clientHeight);
    this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    this.renderer.shadowMap.enabled = true;
    this.renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    this.renderer.toneMapping = THREE.ACESFilmicToneMapping;
    this.renderer.toneMappingExposure = 1;
    container.appendChild(this.renderer.domElement);

    // Controls
    this.controls = new OrbitControls(this.camera, this.renderer.domElement);
    this.controls.enableDamping = true;
    this.controls.dampingFactor = 0.05;
    this.controls.minDistance = 3;
    this.controls.maxDistance = 50;

    // Lighting
    this.setupLighting();

    // Ground
    this.setupGround();

    // Events
    this.renderer.domElement.addEventListener('click', this.handleClick.bind(this));
    window.addEventListener('resize', this.handleResize.bind(this));

    this.state.isInitialized = true;
  }

  private setupLighting(): void {
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.4);
    this.scene.add(ambientLight);

    const dirLight = new THREE.DirectionalLight(0xffffff, 1.5);
    dirLight.position.set(10, 20, 10);
    dirLight.castShadow = true;
    dirLight.shadow.mapSize.width = 2048;
    dirLight.shadow.mapSize.height = 2048;
    dirLight.shadow.camera.near = 0.1;
    dirLight.shadow.camera.far = 100;
    dirLight.shadow.camera.left = -20;
    dirLight.shadow.camera.right = 20;
    dirLight.shadow.camera.top = 20;
    dirLight.shadow.camera.bottom = -20;
    this.scene.add(dirLight);

    const fillLight = new THREE.PointLight(0x4ecdc4, 1, 30);
    fillLight.position.set(-10, 5, -10);
    this.scene.add(fillLight);
  }

  private setupGround(): void {
    const ground = new THREE.Mesh(
      new THREE.PlaneGeometry(40, 40),
      new THREE.MeshStandardMaterial({ color: '#333', roughness: 0.8 })
    );
    ground.rotation.x = -Math.PI / 2;
    ground.position.y = -2;
    ground.receiveShadow = true;
    this.scene.add(ground);
  }

  private createGeometry(type: ObjectDefinition['type']): THREE.BufferGeometry {
    switch (type) {
      case 'box':      return new THREE.BoxGeometry(1, 1, 1);
      case 'sphere':   return new THREE.SphereGeometry(0.6, 32, 32);
      case 'torus':    return new THREE.TorusGeometry(0.5, 0.2, 16, 100);
      case 'cylinder': return new THREE.CylinderGeometry(0.5, 0.5, 1, 32);
    }
  }

  addObject(definition: ObjectDefinition): void {
    const geometry = this.createGeometry(definition.type);
    const material = new THREE.MeshStandardMaterial({
      color: definition.color,
      metalness: definition.metalness,
      roughness: definition.roughness,
    });

    const mesh = new THREE.Mesh(geometry, material);
    mesh.position.set(...definition.position);
    mesh.castShadow = definition.castShadow ?? true;
    mesh.name = definition.id;
    mesh.userData = { definition };

    this.scene.add(mesh);
    this.objects.set(definition.id, mesh);
  }

  removeObject(id: string): boolean {
    const mesh = this.objects.get(id);
    if (!mesh) return false;

    this.scene.remove(mesh);
    
    if (mesh.geometry) mesh.geometry.dispose();
    if (Array.isArray(mesh.material)) {
      mesh.material.forEach(m => m.dispose());
    } else {
      mesh.material.dispose();
    }
    
    this.objects.delete(id);
    
    if (this.state.selectedObjectId === id) {
      this.state.selectedObjectId = null;
    }
    
    return true;
  }

  private handleClick(event: MouseEvent): void {
    if (!this.renderer || !this.camera) return;

    const rect = this.renderer.domElement.getBoundingClientRect();
    this.pointer.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
    this.pointer.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;

    this.raycaster.setFromCamera(this.pointer, this.camera);
    
    const meshes = Array.from(this.objects.values());
    const intersects = this.raycaster.intersectObjects(meshes);

    if (intersects.length > 0) {
      const first = intersects[0];
      if (first.object instanceof THREE.Mesh) {
        this.state.selectedObjectId = first.object.name;
        console.log('Selected:', first.object.name, 'at', first.point);
      }
    } else {
      this.state.selectedObjectId = null;
    }
  }

  private handleResize(): void {
    if (!this.renderer || !this.camera) return;
    
    const container = this.renderer.domElement.parentElement;
    if (!container) return;

    const w = container.clientWidth;
    const h = container.clientHeight;
    
    this.camera.aspect = w / h;
    this.camera.updateProjectionMatrix();
    this.renderer.setSize(w, h);
  }

  startAnimation(): void {
    if (this.state.isAnimating) return;
    this.state.isAnimating = true;
    
    const animate = (): void => {
      this.animationId = requestAnimationFrame(animate);
      
      const delta = this.clock.getDelta();
      const elapsed = this.clock.elapsedTime;
      
      // Update objects
      this.objects.forEach((mesh, id) => {
        const def = mesh.userData['definition'] as ObjectDefinition | undefined;
        if (def?.rotationSpeed) {
          mesh.rotation.x += def.rotationSpeed[0] * delta;
          mesh.rotation.y += def.rotationSpeed[1] * delta;
          mesh.rotation.z += def.rotationSpeed[2] * delta;
        }
        
        // Highlight selected
        if (id === this.state.selectedObjectId) {
          const mat = mesh.material as THREE.MeshStandardMaterial;
          mat.emissive.setHSL(elapsed * 0.5 % 1, 1, 0.2);
        } else {
          const mat = mesh.material as THREE.MeshStandardMaterial;
          mat.emissive.setScalar(0);
        }
      });
      
      this.controls.update();
      this.renderer.render(this.scene, this.camera);
    };
    
    animate();
  }

  stopAnimation(): void {
    if (this.animationId !== null) {
      cancelAnimationFrame(this.animationId);
      this.animationId = null;
    }
    this.state.isAnimating = false;
  }

  dispose(): void {
    this.stopAnimation();
    window.removeEventListener('resize', this.handleResize.bind(this));
    
    // Dispose all objects
    this.objects.forEach((_, id) => this.removeObject(id));
    
    this.controls.dispose();
    this.renderer.dispose();
  }

  getState(): Readonly<AppState> {
    return { ...this.state };
  }
}

// === Main Application ===

const objectDefinitions: ObjectDefinition[] = [
  {
    id: 'box-1',
    type: 'box',
    position: [-3, 0, 0],
    color: '#e74c3c',
    metalness: 0.3,
    roughness: 0.4,
    rotationSpeed: [0.5, 1, 0],
    castShadow: true,
  },
  {
    id: 'sphere-1',
    type: 'sphere',
    position: [0, 0, 0],
    color: '#3498db',
    metalness: 0.8,
    roughness: 0.2,
    rotationSpeed: [0, 0.8, 0.2],
    castShadow: true,
  },
  {
    id: 'torus-1',
    type: 'torus',
    position: [3, 0, 0],
    color: '#2ecc71',
    metalness: 0.5,
    roughness: 0.5,
    rotationSpeed: [1, 0.5, 0],
    castShadow: true,
  },
];

// Initialize
const container = document.getElementById('app');
if (!container) {
  throw new Error('Container element not found');
}

const sceneManager = new SceneManager();
sceneManager.initialize(container, {
  backgroundColor: '#1a1a2e',
  fog: { color: '#1a1a2e', near: 20, far: 60 },
});

objectDefinitions.forEach(def => sceneManager.addObject(def));
sceneManager.startAnimation();

// Cleanup on page unload
window.addEventListener('beforeunload', () => {
  sceneManager.dispose();
});
```

---

## 68.10 Type-safe BufferAttribute

```typescript
// src/geometry/customGeometry.ts
import * as THREE from 'three';

// Typed BufferGeometry builder
class TypedGeometryBuilder {
  private positions: number[] = [];
  private normals: number[] = [];
  private uvs: number[] = [];
  private indices: number[] = [];

  addVertex(
    position: [number, number, number],
    normal: [number, number, number],
    uv: [number, number]
  ): number {
    const index = this.positions.length / 3;
    this.positions.push(...position);
    this.normals.push(...normal);
    this.uvs.push(...uv);
    return index;
  }

  addTriangle(a: number, b: number, c: number): void {
    this.indices.push(a, b, c);
  }

  build(): THREE.BufferGeometry {
    const geometry = new THREE.BufferGeometry();
    
    geometry.setAttribute(
      'position',
      new THREE.Float32BufferAttribute(this.positions, 3)
    );
    
    geometry.setAttribute(
      'normal',
      new THREE.Float32BufferAttribute(this.normals, 3)
    );
    
    geometry.setAttribute(
      'uv',
      new THREE.Float32BufferAttribute(this.uvs, 2)
    );
    
    geometry.setIndex(this.indices);
    geometry.computeVertexNormals();
    
    return geometry;
  }
}

// สร้าง custom geometry ด้วย type safety
function createDiamond(): THREE.BufferGeometry {
  const builder = new TypedGeometryBuilder();
  
  // Top vertex
  const top = builder.addVertex([0, 1, 0], [0, 1, 0], [0.5, 1]);
  
  // Middle ring
  const ring: number[] = [];
  const sides = 6;
  for (let i = 0; i < sides; i++) {
    const angle = (i / sides) * Math.PI * 2;
    const x = Math.cos(angle) * 0.7;
    const z = Math.sin(angle) * 0.7;
    const normal: [number, number, number] = [Math.cos(angle), 0.3, Math.sin(angle)];
    const uv: [number, number] = [i / sides, 0.5];
    ring.push(builder.addVertex([x, 0, z], normal, uv));
  }
  
  // Bottom vertex
  const bottom = builder.addVertex([0, -1, 0], [0, -1, 0], [0.5, 0]);
  
  // Top faces
  for (let i = 0; i < sides; i++) {
    const next = (i + 1) % sides;
    const curr = ring[i];
    const nextV = ring[next];
    if (curr !== undefined && nextV !== undefined) {
      builder.addTriangle(top, curr, nextV);
    }
  }
  
  // Bottom faces
  for (let i = 0; i < sides; i++) {
    const next = (i + 1) % sides;
    const curr = ring[i];
    const nextV = ring[next];
    if (curr !== undefined && nextV !== undefined) {
      builder.addTriangle(bottom, nextV, curr);
    }
  }
  
  return builder.build();
}

export { TypedGeometryBuilder, createDiamond };
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Type-safe Scene Builder
สร้าง builder pattern สำหรับ Three.js scene ที่:
- Fluent API: `new SceneBuilder().addBox().addSphere().build()`
- Strongly typed parameters
- Automatic cleanup on dispose

### แบบฝึกหัดที่ 2: Typed Animation System
สร้าง animation system ที่:
- State machine ด้วย discriminated union
- Type-safe transition conditions
- Typed animation clips

### แบบฝึกหัดที่ 3: GLTF Type Extractor
สร้าง utility ที่:
- Extract typed nodes จาก GLTF
- Validate bone hierarchy
- Type-safe animation controls

### แบบฝึกหัดที่ 4: Shader Hot Reload
สร้าง shader system ที่:
- Type-safe uniform updates
- Hot reload ใน development
- Validation ของ uniform types

### แบบฝึกหัดที่ 5: Physics Integration
สร้าง physics wrapper ที่:
- Typed rigid body definitions
- Type-safe collision callbacks
- Sync กับ Three.js objects

---

## สรุป

ในบทนี้เราเรียนรู้:
1. **TypeScript Setup** - tsconfig.json ที่เหมาะสมสำหรับ Three.js
2. **Type Definitions** - Import และ define types
3. **Typed GLTF** - โหลดและ access GLTF อย่าง type-safe
4. **Shader Uniforms** - Interface สำหรับ uniform values
5. **Generics** - Generic functions สำหรับ scene utilities
6. **Discriminated Unions** - Material type system
7. **Strict Null Checks** - Null-safe Three.js patterns

บทถัดไปเราจะเรียน **Render Targets & Off-Screen Rendering**
