# Part 91: Testing Three.js Applications

## บทนำ: ทำไมต้อง Test 3D Applications

การ test 3D applications มีความท้าทายพิเศษ:
- WebGL ต้องการ GPU context
- Visual output ยากจะ assert ด้วยโค้ด
- Animation และ physics เป็น stateful
- Async asset loading

เราใช้หลายกลยุทธ์:
1. **Unit tests** - test logic แยก จาก rendering
2. **Headless WebGL** - render ใน Node.js
3. **Visual regression** - screenshot comparison
4. **Performance tests** - FPS และ memory

---

## 1. Setup Vitest + JSDOM

```bash
# ติดตั้ง dependencies
npm install -D vitest jsdom @vitest/coverage-v8
npm install -D canvas  # สำหรับ JSDOM canvas support
```

```javascript
// vitest.config.js
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'jsdom',
    setupFiles: ['./tests/setup.js'],
    globals: true,
    coverage: {
      provider: 'v8',
      include: ['src/**/*.js'],
      exclude: ['src/vendor/**']
    }
  }
});
```

```javascript
// tests/setup.js
import { vi } from 'vitest';

// Mock WebGL context
class MockWebGLRenderingContext {
  canvas = null;
  drawingBufferWidth = 800;
  drawingBufferHeight = 600;
  
  getExtension(name) {
    const extensions = {
      'WEBGL_lose_context': { loseContext: vi.fn(), restoreContext: vi.fn() },
      'OES_texture_float': {},
      'EXT_texture_filter_anisotropic': { MAX_TEXTURE_MAX_ANISOTROPY_EXT: 16 }
    };
    return extensions[name] || null;
  }
  
  getParameter(param) {
    const params = {
      [0x1F02]: 'WebGL 2.0',  // VERSION
      [0x1F00]: 'WebKit WebGL',  // VENDOR
      [0x1F01]: 'WebGL',  // RENDERER
      [0x8B4D]: 16,  // MAX_VERTEX_UNIFORM_VECTORS
      [0x8DFB]: 16,  // MAX_FRAGMENT_UNIFORM_VECTORS
      [0x0D33]: 1024,  // MAX_TEXTURE_SIZE
    };
    return params[param] ?? 0;
  }
  
  // Stub all WebGL methods
  createBuffer() { return {}; }
  createTexture() { return {}; }
  createProgram() { return {}; }
  createShader() { return {}; }
  bindBuffer() {}
  bindTexture() {}
  bufferData() {}
  enable() {}
  disable() {}
  clear() {}
  clearColor() {}
  viewport() {}
  drawArrays() {}
  drawElements() {}
  shaderSource() {}
  compileShader() {}
  attachShader() {}
  linkProgram() {}
  useProgram() {}
  getShaderParameter() { return true; }
  getProgramParameter() { return true; }
  getUniformLocation() { return {}; }
  getAttribLocation() { return 0; }
  uniform1f() {}
  uniform2f() {}
  uniform3f() {}
  uniform4f() {}
  uniformMatrix4fv() {}
  vertexAttribPointer() {}
  enableVertexAttribArray() {}
  texImage2D() {}
  texParameteri() {}
  generateMipmap() {}
  deleteBuffer() {}
  deleteTexture() {}
  deleteProgram() {}
  deleteShader() {}
  getShaderInfoLog() { return ''; }
  getProgramInfoLog() { return ''; }
  createVertexArray() { return {}; }
  bindVertexArray() {}
  deleteVertexArray() {}
  createFramebuffer() { return {}; }
  bindFramebuffer() {}
  framebufferTexture2D() {}
  checkFramebufferStatus() { return 0x8CD5; } // FRAMEBUFFER_COMPLETE
  deleteFramebuffer() {}
  readPixels() {}
  pixelStorei() {}
  scissor() {}
  stencilFunc() {}
  stencilOp() {}
  depthFunc() {}
  depthMask() {}
  blendFunc() {}
  blendEquation() {}
  colorMask() {}
  cullFace() {}
  frontFace() {}
  lineWidth() {}
  polygonOffset() {}
  activeTexture() {}
  createRenderbuffer() { return {}; }
  bindRenderbuffer() {}
  renderbufferStorage() {}
  framebufferRenderbuffer() {}
  deleteRenderbuffer() {}
}

// Mock canvas
const originalGetContext = HTMLCanvasElement.prototype.getContext;
HTMLCanvasElement.prototype.getContext = function(contextType, ...args) {
  if (contextType === 'webgl' || contextType === 'webgl2') {
    const ctx = new MockWebGLRenderingContext();
    ctx.canvas = this;
    return ctx;
  }
  return originalGetContext.call(this, contextType, ...args);
};

// Mock requestAnimationFrame
global.requestAnimationFrame = vi.fn(cb => setTimeout(cb, 16));
global.cancelAnimationFrame = vi.fn(id => clearTimeout(id));

// Mock performance
global.performance = global.performance || {
  now: () => Date.now()
};
```

---

## 2. Unit Tests: Core Logic

```javascript
// tests/unit/math.test.js
import { describe, it, expect } from 'vitest';
import * as THREE from 'three';

describe('THREE.js Math Utilities', () => {
  describe('Vector3', () => {
    it('should add vectors correctly', () => {
      const a = new THREE.Vector3(1, 2, 3);
      const b = new THREE.Vector3(4, 5, 6);
      a.add(b);
      expect(a.x).toBe(5);
      expect(a.y).toBe(7);
      expect(a.z).toBe(9);
    });
    
    it('should calculate distance', () => {
      const a = new THREE.Vector3(0, 0, 0);
      const b = new THREE.Vector3(3, 4, 0);
      expect(a.distanceTo(b)).toBe(5);
    });
    
    it('should normalize correctly', () => {
      const v = new THREE.Vector3(3, 0, 0);
      v.normalize();
      expect(v.x).toBeCloseTo(1);
      expect(v.length()).toBeCloseTo(1);
    });
  });
  
  describe('Box3', () => {
    it('should detect intersection', () => {
      const box1 = new THREE.Box3(
        new THREE.Vector3(-1, -1, -1),
        new THREE.Vector3(1, 1, 1)
      );
      const box2 = new THREE.Box3(
        new THREE.Vector3(0, 0, 0),
        new THREE.Vector3(2, 2, 2)
      );
      const box3 = new THREE.Box3(
        new THREE.Vector3(5, 5, 5),
        new THREE.Vector3(7, 7, 7)
      );
      
      expect(box1.intersectsBox(box2)).toBe(true);
      expect(box1.intersectsBox(box3)).toBe(false);
    });
  });
  
  describe('Raycaster', () => {
    it('should intersect sphere', () => {
      const raycaster = new THREE.Raycaster(
        new THREE.Vector3(0, 0, 5),
        new THREE.Vector3(0, 0, -1)
      );
      
      const sphere = new THREE.Mesh(
        new THREE.SphereGeometry(1),
        new THREE.MeshBasicMaterial()
      );
      
      const hits = raycaster.intersectObject(sphere);
      expect(hits.length).toBe(2); // Enter and exit
    });
    
    it('should miss when no intersection', () => {
      const raycaster = new THREE.Raycaster(
        new THREE.Vector3(10, 0, 5),
        new THREE.Vector3(0, 0, -1)
      );
      
      const sphere = new THREE.Mesh(
        new THREE.SphereGeometry(1),
        new THREE.MeshBasicMaterial()
      );
      
      const hits = raycaster.intersectObject(sphere);
      expect(hits.length).toBe(0);
    });
  });
});
```

---

## 3. Unit Tests: Game Logic

```javascript
// tests/unit/game-logic.test.js
import { describe, it, expect, vi, beforeEach } from 'vitest';

// ===== Entity Component System Tests =====
class Component {}

class TransformComponent extends Component {
  constructor(x = 0, y = 0, z = 0) {
    super();
    this.position = { x, y, z };
    this.rotation = { x: 0, y: 0, z: 0 };
  }
}

class HealthComponent extends Component {
  constructor(hp = 100) {
    super();
    this.current = hp;
    this.max = hp;
    this.dead = false;
  }
  
  damage(amount) {
    this.current = Math.max(0, this.current - amount);
    if (this.current === 0) this.dead = true;
  }
  
  heal(amount) {
    this.current = Math.min(this.max, this.current + amount);
    this.dead = false;
  }
}

class EntityManager {
  constructor() {
    this.entities = new Map();
    this.nextId = 1;
  }
  
  createEntity(components = {}) {
    const id = this.nextId++;
    this.entities.set(id, components);
    return id;
  }
  
  getComponent(entityId, ComponentClass) {
    const entity = this.entities.get(entityId);
    if (!entity) return null;
    return entity[ComponentClass.name] || null;
  }
  
  addComponent(entityId, component) {
    const entity = this.entities.get(entityId);
    if (entity) {
      entity[component.constructor.name] = component;
    }
  }
  
  query(...ComponentClasses) {
    const results = [];
    for (const [id, components] of this.entities) {
      if (ComponentClasses.every(CC => components[CC.name])) {
        results.push(id);
      }
    }
    return results;
  }
  
  destroyEntity(id) {
    this.entities.delete(id);
  }
}

describe('EntityManager', () => {
  let em;
  
  beforeEach(() => {
    em = new EntityManager();
  });
  
  it('should create entities with unique IDs', () => {
    const id1 = em.createEntity();
    const id2 = em.createEntity();
    expect(id1).not.toBe(id2);
  });
  
  it('should query entities with specific components', () => {
    const transform = new TransformComponent(1, 2, 3);
    const health = new HealthComponent(100);
    
    em.createEntity({
      TransformComponent: transform,
      HealthComponent: health
    });
    em.createEntity({
      TransformComponent: new TransformComponent()
    });
    
    const withBoth = em.query(TransformComponent, HealthComponent);
    const withTransform = em.query(TransformComponent);
    
    expect(withBoth.length).toBe(1);
    expect(withTransform.length).toBe(2);
  });
  
  it('should add components to entities', () => {
    const id = em.createEntity({ TransformComponent: new TransformComponent() });
    
    expect(em.getComponent(id, HealthComponent)).toBeNull();
    
    em.addComponent(id, new HealthComponent(50));
    
    const health = em.getComponent(id, HealthComponent);
    expect(health).not.toBeNull();
    expect(health.current).toBe(50);
  });
  
  it('should destroy entities', () => {
    const id = em.createEntity({ TransformComponent: new TransformComponent() });
    
    expect(em.entities.has(id)).toBe(true);
    em.destroyEntity(id);
    expect(em.entities.has(id)).toBe(false);
  });
});

describe('HealthComponent', () => {
  it('should take damage correctly', () => {
    const hp = new HealthComponent(100);
    hp.damage(30);
    expect(hp.current).toBe(70);
    expect(hp.dead).toBe(false);
  });
  
  it('should die at 0 HP', () => {
    const hp = new HealthComponent(100);
    hp.damage(150);
    expect(hp.current).toBe(0);
    expect(hp.dead).toBe(true);
  });
  
  it('should heal correctly', () => {
    const hp = new HealthComponent(100);
    hp.damage(50);
    hp.heal(30);
    expect(hp.current).toBe(80);
  });
  
  it('should not heal over max', () => {
    const hp = new HealthComponent(100);
    hp.heal(200);
    expect(hp.current).toBe(100);
  });
});
```

---

## 4. Tests: StateMachine

```javascript
// tests/unit/state-machine.test.js
import { describe, it, expect, vi, beforeEach } from 'vitest';

class StateMachine {
  constructor() {
    this.states = new Map();
    this.transitions = new Map();
    this.currentState = null;
    this.listeners = {};
  }
  
  addState(name, callbacks = {}) {
    this.states.set(name, callbacks);
    return this;
  }
  
  addTransition(from, trigger, to, condition = null) {
    if (!this.transitions.has(from)) {
      this.transitions.set(from, []);
    }
    this.transitions.get(from).push({ trigger, to, condition });
    return this;
  }
  
  start(stateName) {
    this.currentState = stateName;
    const state = this.states.get(stateName);
    if (state?.onEnter) state.onEnter();
  }
  
  trigger(triggerName, data = null) {
    if (!this.currentState) return false;
    
    const trans = this.transitions.get(this.currentState) || [];
    const match = trans.find(t => t.trigger === triggerName);
    
    if (!match) return false;
    if (match.condition && !match.condition(data)) return false;
    
    const fromState = this.states.get(this.currentState);
    if (fromState?.onExit) fromState.onExit();
    
    this.currentState = match.to;
    
    const toState = this.states.get(this.currentState);
    if (toState?.onEnter) toState.onEnter(data);
    
    return true;
  }
  
  is(stateName) {
    return this.currentState === stateName;
  }
}

describe('StateMachine', () => {
  let sm;
  const onEnterMenu = vi.fn();
  const onExitMenu = vi.fn();
  const onEnterPlaying = vi.fn();
  
  beforeEach(() => {
    vi.clearAllMocks();
    
    sm = new StateMachine();
    sm.addState('menu', { onEnter: onEnterMenu, onExit: onExitMenu })
      .addState('playing', { onEnter: onEnterPlaying })
      .addState('paused', {})
      .addState('gameover', {})
      .addTransition('menu', 'start', 'playing')
      .addTransition('playing', 'pause', 'paused')
      .addTransition('paused', 'resume', 'playing')
      .addTransition('playing', 'die', 'gameover');
  });
  
  it('should start in correct state', () => {
    sm.start('menu');
    expect(sm.currentState).toBe('menu');
    expect(onEnterMenu).toHaveBeenCalledOnce();
  });
  
  it('should transition on trigger', () => {
    sm.start('menu');
    const result = sm.trigger('start');
    expect(result).toBe(true);
    expect(sm.currentState).toBe('playing');
    expect(onExitMenu).toHaveBeenCalledOnce();
    expect(onEnterPlaying).toHaveBeenCalledOnce();
  });
  
  it('should ignore invalid triggers', () => {
    sm.start('menu');
    const result = sm.trigger('pause');  // Invalid from menu
    expect(result).toBe(false);
    expect(sm.currentState).toBe('menu');
  });
  
  it('should handle conditional transitions', () => {
    sm.addTransition('playing', 'pickup', 'playing', (data) => data?.type === 'health');
    sm.start('menu');
    sm.trigger('start');
    
    const resultFail = sm.trigger('pickup', { type: 'ammo' });
    expect(resultFail).toBe(false);
    
    const resultPass = sm.trigger('pickup', { type: 'health' });
    expect(resultPass).toBe(true);
  });
  
  it('should check current state with is()', () => {
    sm.start('menu');
    expect(sm.is('menu')).toBe(true);
    expect(sm.is('playing')).toBe(false);
    
    sm.trigger('start');
    expect(sm.is('menu')).toBe(false);
    expect(sm.is('playing')).toBe(true);
  });
});
```

---

## 5. Tests: Scene Management

```javascript
// tests/unit/scene-manager.test.js
import { describe, it, expect, vi, beforeEach } from 'vitest';
import * as THREE from 'three';

describe('THREE.Scene Management', () => {
  it('should add and remove objects', () => {
    const scene = new THREE.Scene();
    
    const cube = new THREE.Mesh(
      new THREE.BoxGeometry(1, 1, 1),
      new THREE.MeshBasicMaterial({ color: 0xff0000 })
    );
    cube.name = 'TestCube';
    
    scene.add(cube);
    expect(scene.children.length).toBe(1);
    
    scene.remove(cube);
    expect(scene.children.length).toBe(0);
  });
  
  it('should find objects by name', () => {
    const scene = new THREE.Scene();
    
    const obj1 = new THREE.Mesh(new THREE.BoxGeometry(), new THREE.MeshBasicMaterial());
    obj1.name = 'Player';
    
    const obj2 = new THREE.Mesh(new THREE.BoxGeometry(), new THREE.MeshBasicMaterial());
    obj2.name = 'Enemy';
    
    scene.add(obj1, obj2);
    
    const found = scene.getObjectByName('Player');
    expect(found).toBe(obj1);
    expect(found.name).toBe('Player');
  });
  
  it('should traverse all children', () => {
    const scene = new THREE.Scene();
    const group = new THREE.Group();
    
    for (let i = 0; i < 5; i++) {
      group.add(new THREE.Mesh(new THREE.BoxGeometry(), new THREE.MeshBasicMaterial()));
    }
    scene.add(group);
    
    let count = 0;
    scene.traverse(() => count++);
    
    // scene + group + 5 meshes
    expect(count).toBe(7);
  });
  
  it('should dispose geometry and materials', () => {
    const geo = new THREE.BoxGeometry(1, 1, 1);
    const mat = new THREE.MeshBasicMaterial({ color: 0xff0000 });
    
    const disposeSpy = vi.spyOn(geo, 'dispose');
    const matDisposeSpy = vi.spyOn(mat, 'dispose');
    
    geo.dispose();
    mat.dispose();
    
    expect(disposeSpy).toHaveBeenCalledOnce();
    expect(matDisposeSpy).toHaveBeenCalledOnce();
  });
});

describe('Animation System', () => {
  it('should update animation mixer', () => {
    const obj = new THREE.Object3D();
    const mixer = new THREE.AnimationMixer(obj);
    
    // Create a simple animation clip
    const track = new THREE.VectorKeyframeTrack(
      '.position',
      [0, 1],
      [0, 0, 0, 1, 0, 0]
    );
    const clip = new THREE.AnimationClip('test', 1, [track]);
    
    const action = mixer.clipAction(clip);
    action.play();
    
    // Should not throw when updating
    expect(() => mixer.update(0.016)).not.toThrow();
  });
});
```

---

## 6. Performance Testing

```javascript
// tests/performance/render-perf.test.js
import { describe, it, expect } from 'vitest';
import * as THREE from 'three';

describe('Performance Tests', () => {
  it('should create 1000 objects in < 100ms', () => {
    const scene = new THREE.Scene();
    const geometry = new THREE.BoxGeometry(1, 1, 1);
    const material = new THREE.MeshBasicMaterial({ color: 0x00ff00 });
    
    const start = performance.now();
    
    for (let i = 0; i < 1000; i++) {
      const mesh = new THREE.Mesh(geometry, material);
      mesh.position.set(
        Math.random() * 100,
        Math.random() * 100,
        Math.random() * 100
      );
      scene.add(mesh);
    }
    
    const end = performance.now();
    const duration = end - start;
    
    console.log(`Created 1000 objects in ${duration.toFixed(2)}ms`);
    expect(duration).toBeLessThan(100);
    
    // Cleanup
    scene.clear();
    geometry.dispose();
    material.dispose();
  });
  
  it('should update 1000 object transforms in < 16ms', () => {
    const objects = [];
    const geo = new THREE.BoxGeometry();
    const mat = new THREE.MeshBasicMaterial();
    
    for (let i = 0; i < 1000; i++) {
      const mesh = new THREE.Mesh(geo, mat);
      objects.push(mesh);
    }
    
    const start = performance.now();
    
    for (const obj of objects) {
      obj.position.x += 0.1;
      obj.rotation.y += 0.01;
      obj.updateMatrix();
    }
    
    const end = performance.now();
    const duration = end - start;
    
    console.log(`Updated 1000 transforms in ${duration.toFixed(2)}ms`);
    expect(duration).toBeLessThan(16);
    
    geo.dispose();
    mat.dispose();
  });
  
  it('should raycasting 100 objects in < 10ms', () => {
    const scene = new THREE.Scene();
    const geo = new THREE.BoxGeometry(1, 1, 1);
    const mat = new THREE.MeshBasicMaterial();
    const objects = [];
    
    for (let i = 0; i < 100; i++) {
      const mesh = new THREE.Mesh(geo, mat);
      mesh.position.set(
        (Math.random() - 0.5) * 20,
        (Math.random() - 0.5) * 20,
        (Math.random() - 0.5) * 20
      );
      scene.add(mesh);
      objects.push(mesh);
    }
    
    const raycaster = new THREE.Raycaster(
      new THREE.Vector3(0, 0, 50),
      new THREE.Vector3(0, 0, -1)
    );
    
    const start = performance.now();
    raycaster.intersectObjects(objects);
    const end = performance.now();
    
    const duration = end - start;
    console.log(`Raycast against 100 objects: ${duration.toFixed(2)}ms`);
    expect(duration).toBeLessThan(10);
    
    geo.dispose();
    mat.dispose();
  });
});
```

---

## 7. Complete Test Suite Setup

```bash
# package.json scripts
{
  "scripts": {
    "test": "vitest",
    "test:ui": "vitest --ui",
    "test:coverage": "vitest --coverage",
    "test:run": "vitest run",
    "test:perf": "vitest run tests/performance"
  }
}
```

```javascript
// tests/helpers/three-helpers.js
// Reusable test helpers

export function createTestRenderer(width = 100, height = 100) {
  const canvas = document.createElement('canvas');
  canvas.width = width;
  canvas.height = height;
  
  // Use mock context set up in setup.js
  const renderer = new THREE.WebGLRenderer({ canvas });
  renderer.setSize(width, height);
  return renderer;
}

export function createTestScene() {
  const scene = new THREE.Scene();
  const camera = new THREE.PerspectiveCamera(75, 1, 0.1, 100);
  camera.position.z = 5;
  
  return { scene, camera };
}

export function waitForAnimation(frames = 1) {
  return new Promise(resolve => {
    let count = 0;
    const tick = () => {
      count++;
      if (count >= frames) resolve();
      else requestAnimationFrame(tick);
    };
    requestAnimationFrame(tick);
  });
}

export function createMockGLTF(options = {}) {
  const scene = new THREE.Scene();
  
  if (options.addCube) {
    scene.add(new THREE.Mesh(
      new THREE.BoxGeometry(),
      new THREE.MeshStandardMaterial()
    ));
  }
  
  return {
    scene,
    animations: options.animations || [],
    cameras: options.cameras || [],
    asset: { generator: 'Test', version: '2.0' },
    parser: { json: {} }
  };
}
```

---

## 8. CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/test.yml
name: Three.js Tests

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run unit tests
        run: npm run test:run
      
      - name: Generate coverage
        run: npm run test:coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/coverage-final.json

  visual-regression:
    runs-on: ubuntu-latest
    needs: unit-tests
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - name: Install Playwright
        run: npx playwright install --with-deps chromium
      
      - name: Run visual tests
        run: npx playwright test tests/visual/
      
      - name: Upload screenshots
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: visual-test-failures
          path: test-results/
```

---

## 9. สรุป Testing Strategy

```
Unit Tests:         Logic, Math, Game Systems
Integration Tests:  Scene Setup, Asset Loading
Visual Tests:       Screenshot Comparisons
Performance Tests:  Frame Budget, Memory
E2E Tests:          User Flows
```

**เครื่องมือที่ใช้:**
- Vitest - unit tests รวดเร็ว
- Playwright - browser automation + visual regression
- GitHub Actions - CI/CD pipeline
- Codecov - coverage reporting

ในบทถัดไป (Part 92) เราจะ deploy Three.js app ไป production
