# Part 85: Game Architecture กับ Three.js

## บทนำ: Three.js เป็น Game Engine

Three.js ไม่ใช่ game engine โดยตรง แต่สามารถใช้เป็น rendering backend สำหรับ game engine ที่สร้างเองได้ ในบทนี้เราจะสร้าง architecture ที่เหมาะสมสำหรับการพัฒนา game

---

## 1. Entity-Component-System (ECS)

ECS เป็น pattern ที่ใช้กันอย่างแพร่หลายในการพัฒนา game เพราะให้ flexibility และ performance ที่ดี

### แนวคิดหลัก

```
Entity  = ID เท่านั้น (เช่น player = entity 1)
Component = ข้อมูล (เช่น Position, Health, Velocity)
System    = Logic (เช่น MovementSystem, RenderSystem)
```

### ตัวอย่าง ECS Framework

```javascript
// ===== Core ECS =====

class EntityManager {
  constructor() {
    this._nextId = 1;
    this._entities = new Set();
    this._components = new Map();  // componentType -> Map<entityId, component>
    this._entityComponents = new Map();  // entityId -> Set<componentType>
    this._toDelete = new Set();
  }
  
  createEntity() {
    const id = this._nextId++;
    this._entities.add(id);
    this._entityComponents.set(id, new Set());
    return id;
  }
  
  destroyEntity(id) {
    this._toDelete.add(id);
  }
  
  _processDeleted() {
    for (const id of this._toDelete) {
      // Remove all components
      const types = this._entityComponents.get(id);
      if (types) {
        for (const type of types) {
          const map = this._components.get(type);
          if (map) map.delete(id);
        }
      }
      this._entityComponents.delete(id);
      this._entities.delete(id);
    }
    this._toDelete.clear();
  }
  
  addComponent(entityId, component) {
    const type = component.constructor;
    
    if (!this._components.has(type)) {
      this._components.set(type, new Map());
    }
    this._components.get(type).set(entityId, component);
    this._entityComponents.get(entityId)?.add(type);
    
    return component;
  }
  
  removeComponent(entityId, componentType) {
    const map = this._components.get(componentType);
    if (map) map.delete(entityId);
    this._entityComponents.get(entityId)?.delete(componentType);
  }
  
  getComponent(entityId, componentType) {
    return this._components.get(componentType)?.get(entityId);
  }
  
  hasComponent(entityId, componentType) {
    return this._components.get(componentType)?.has(entityId) ?? false;
  }
  
  // Query entities ที่มีทุก component ที่ระบุ
  query(...componentTypes) {
    if (componentTypes.length === 0) return Array.from(this._entities);
    
    // หา set ที่เล็กที่สุดก่อน
    let smallest = null;
    let smallestSize = Infinity;
    
    for (const type of componentTypes) {
      const map = this._components.get(type);
      if (!map) return [];
      if (map.size < smallestSize) {
        smallest = map;
        smallestSize = map.size;
      }
    }
    
    const result = [];
    for (const [id] of smallest) {
      let hasAll = true;
      for (const type of componentTypes) {
        if (!this._components.get(type)?.has(id)) {
          hasAll = false;
          break;
        }
      }
      if (hasAll) result.push(id);
    }
    
    return result;
  }
  
  update() {
    this._processDeleted();
  }
}

// ===== Components =====

class TransformComponent {
  constructor(x = 0, y = 0, z = 0) {
    this.position = new THREE.Vector3(x, y, z);
    this.rotation = new THREE.Euler(0, 0, 0);
    this.scale = new THREE.Vector3(1, 1, 1);
    this.matrix = new THREE.Matrix4();
    this.dirty = true;
  }
  
  setPosition(x, y, z) {
    this.position.set(x, y, z);
    this.dirty = true;
    return this;
  }
  
  setRotation(x, y, z) {
    this.rotation.set(x, y, z);
    this.dirty = true;
    return this;
  }
  
  updateMatrix() {
    if (this.dirty) {
      this.matrix.compose(this.position, 
        new THREE.Quaternion().setFromEuler(this.rotation), 
        this.scale);
      this.dirty = false;
    }
    return this.matrix;
  }
}

class VelocityComponent {
  constructor(x = 0, y = 0, z = 0) {
    this.velocity = new THREE.Vector3(x, y, z);
    this.angularVelocity = new THREE.Vector3(0, 0, 0);
    this.damping = 0.99;
  }
}

class HealthComponent {
  constructor(maxHealth = 100) {
    this.maxHealth = maxHealth;
    this.currentHealth = maxHealth;
    this.alive = true;
    this.invincible = false;
    this.invincibleTimer = 0;
  }
  
  get percentage() { return this.currentHealth / this.maxHealth; }
  
  takeDamage(amount) {
    if (this.invincible) return 0;
    const actual = Math.min(amount, this.currentHealth);
    this.currentHealth -= actual;
    if (this.currentHealth <= 0) {
      this.alive = false;
      this.currentHealth = 0;
    }
    return actual;
  }
  
  heal(amount) {
    this.currentHealth = Math.min(this.maxHealth, this.currentHealth + amount);
    if (this.currentHealth > 0) this.alive = true;
    return amount;
  }
}

class RenderComponent {
  constructor(mesh) {
    this.mesh = mesh;
    this.visible = true;
    this.castShadow = true;
    this.receiveShadow = true;
  }
}

class ColliderComponent {
  constructor(type = 'sphere', params = {}) {
    this.type = type;  // 'sphere', 'box', 'capsule'
    this.params = params;
    this.isTrigger = false;
    this.layer = 0;  // collision layer
    this.mask = 0xFFFFFFFF;  // collision mask
  }
}

class TagComponent {
  constructor(...tags) {
    this.tags = new Set(tags);
  }
  
  has(tag) { return this.tags.has(tag); }
  add(tag) { this.tags.add(tag); }
  remove(tag) { this.tags.delete(tag); }
}

class AIComponent {
  constructor() {
    this.state = 'idle';
    this.target = null;
    this.pathNodes = [];
    this.currentPathIndex = 0;
    this.detectionRange = 10;
    this.attackRange = 2;
  }
}

class PlayerInputComponent {
  constructor() {
    this.moveX = 0;
    this.moveZ = 0;
    this.jump = false;
    this.shoot = false;
    this.interact = false;
  }
}

// ===== Systems =====

class System {
  constructor(em) {
    this.em = em;
    this.priority = 0;
  }
  
  update(dt) {}
}

class MovementSystem extends System {
  constructor(em) {
    super(em);
    this.priority = 10;
  }
  
  update(dt) {
    const entities = this.em.query(TransformComponent, VelocityComponent);
    
    for (const id of entities) {
      const transform = this.em.getComponent(id, TransformComponent);
      const velocity = this.em.getComponent(id, VelocityComponent);
      
      // Apply velocity
      transform.position.x += velocity.velocity.x * dt;
      transform.position.y += velocity.velocity.y * dt;
      transform.position.z += velocity.velocity.z * dt;
      
      // Damping
      velocity.velocity.multiplyScalar(velocity.damping);
      
      transform.dirty = true;
    }
  }
}

class PlayerInputSystem extends System {
  constructor(em, inputManager) {
    super(em);
    this.inputManager = inputManager;
    this.priority = 5;
  }
  
  update(dt) {
    const entities = this.em.query(PlayerInputComponent, VelocityComponent);
    
    for (const id of entities) {
      const input = this.em.getComponent(id, PlayerInputComponent);
      const velocity = this.em.getComponent(id, VelocityComponent);
      
      // Update input state
      input.moveX = (this.inputManager.isDown('d') ? 1 : 0) - (this.inputManager.isDown('a') ? 1 : 0);
      input.moveZ = (this.inputManager.isDown('s') ? 1 : 0) - (this.inputManager.isDown('w') ? 1 : 0);
      input.jump = this.inputManager.isJustPressed('space');
      input.shoot = this.inputManager.isDown('lmb');
      
      // Apply to velocity
      const speed = 5;
      velocity.velocity.x = input.moveX * speed;
      velocity.velocity.z = input.moveZ * speed;
      
      if (input.jump) {
        velocity.velocity.y = 8;
      }
    }
  }
}

class RenderSystem extends System {
  constructor(em, scene) {
    super(em);
    this.scene = scene;
    this.priority = 100;
  }
  
  update(dt) {
    const entities = this.em.query(TransformComponent, RenderComponent);
    
    for (const id of entities) {
      const transform = this.em.getComponent(id, TransformComponent);
      const render = this.em.getComponent(id, RenderComponent);
      
      if (!render.mesh) continue;
      
      // Sync Three.js mesh กับ transform component
      const matrix = transform.updateMatrix();
      render.mesh.matrix.copy(matrix);
      render.mesh.matrixWorldNeedsUpdate = true;
      render.mesh.visible = render.visible;
    }
  }
}

class HealthSystem extends System {
  constructor(em, eventSystem) {
    super(em);
    this.eventSystem = eventSystem;
    this.priority = 20;
  }
  
  update(dt) {
    const entities = this.em.query(HealthComponent);
    
    for (const id of entities) {
      const health = this.em.getComponent(id, HealthComponent);
      
      // Update invincibility timer
      if (health.invincible) {
        health.invincibleTimer -= dt;
        if (health.invincibleTimer <= 0) {
          health.invincible = false;
        }
      }
      
      // Check death
      if (!health.alive) {
        this.eventSystem.emit('entity_died', { id });
        this.em.destroyEntity(id);
      }
    }
  }
}

class SimpleAISystem extends System {
  constructor(em) {
    super(em);
    this.priority = 15;
  }
  
  update(dt) {
    const aiEntities = this.em.query(AIComponent, TransformComponent, VelocityComponent);
    const playerEntities = this.em.query(PlayerInputComponent, TransformComponent);
    
    if (playerEntities.length === 0) return;
    
    const playerId = playerEntities[0];
    const playerTransform = this.em.getComponent(playerId, TransformComponent);
    
    for (const id of aiEntities) {
      const ai = this.em.getComponent(id, AIComponent);
      const transform = this.em.getComponent(id, TransformComponent);
      const velocity = this.em.getComponent(id, VelocityComponent);
      
      const distToPlayer = transform.position.distanceTo(playerTransform.position);
      
      // State machine
      switch (ai.state) {
        case 'idle':
          if (distToPlayer < ai.detectionRange) {
            ai.state = 'chase';
            ai.target = playerId;
          }
          break;
          
        case 'chase':
          if (distToPlayer > ai.detectionRange * 1.5) {
            ai.state = 'idle';
            ai.target = null;
          } else if (distToPlayer < ai.attackRange) {
            ai.state = 'attack';
          } else {
            // Move toward player
            const dir = playerTransform.position.clone()
              .sub(transform.position)
              .normalize();
            velocity.velocity.set(dir.x * 3, velocity.velocity.y, dir.z * 3);
          }
          break;
          
        case 'attack':
          if (distToPlayer > ai.attackRange * 1.2) {
            ai.state = 'chase';
          }
          // Attack logic
          break;
      }
    }
  }
}
```

---

## 2. Game Loop (Fixed Timestep)

```javascript
class GameLoop {
  constructor() {
    this.fixedTimestep = 1 / 60;  // 60 updates per second
    this.maxFrameTime = 0.25;     // ป้องกัน spiral of death
    this.accumulator = 0;
    this.currentTime = 0;
    this.frameId = null;
    
    this.onFixedUpdate = null;  // physics/game logic
    this.onUpdate = null;       // interpolated update
    this.onRender = null;       // render
    
    this.running = false;
    this.frameCount = 0;
    this.fps = 0;
    this._lastFPSTime = 0;
    this._fpsFrameCount = 0;
  }
  
  start() {
    this.running = true;
    this.currentTime = performance.now() / 1000;
    this._lastFPSTime = this.currentTime;
    this.frameId = requestAnimationFrame(this._tick.bind(this));
  }
  
  stop() {
    this.running = false;
    if (this.frameId) {
      cancelAnimationFrame(this.frameId);
      this.frameId = null;
    }
  }
  
  _tick(timestamp) {
    if (!this.running) return;
    
    const newTime = timestamp / 1000;
    let frameTime = Math.min(newTime - this.currentTime, this.maxFrameTime);
    this.currentTime = newTime;
    
    this.accumulator += frameTime;
    
    // Fixed updates (สำหรับ physics)
    while (this.accumulator >= this.fixedTimestep) {
      if (this.onFixedUpdate) {
        this.onFixedUpdate(this.fixedTimestep);
      }
      this.accumulator -= this.fixedTimestep;
    }
    
    // Variable update (สำหรับ animation)
    const alpha = this.accumulator / this.fixedTimestep;  // interpolation factor
    if (this.onUpdate) {
      this.onUpdate(frameTime, alpha);
    }
    
    // Render
    if (this.onRender) {
      this.onRender(alpha);
    }
    
    // FPS counting
    this.frameCount++;
    this._fpsFrameCount++;
    if (newTime - this._lastFPSTime >= 1.0) {
      this.fps = Math.round(this._fpsFrameCount / (newTime - this._lastFPSTime));
      this._fpsFrameCount = 0;
      this._lastFPSTime = newTime;
    }
    
    this.frameId = requestAnimationFrame(this._tick.bind(this));
  }
}
```

---

## 3. State Machine

```javascript
// ===== Game State Machine =====

class StateMachine {
  constructor() {
    this.states = new Map();
    this.currentState = null;
    this.previousState = null;
    this.transitions = new Map();  // state -> Map<event, targetState>
  }
  
  addState(name, state) {
    this.states.set(name, state);
    return this;
  }
  
  addTransition(fromState, event, toState) {
    if (!this.transitions.has(fromState)) {
      this.transitions.set(fromState, new Map());
    }
    this.transitions.get(fromState).set(event, toState);
    return this;
  }
  
  start(stateName) {
    const state = this.states.get(stateName);
    if (!state) throw new Error(`State ${stateName} not found`);
    
    this.currentState = stateName;
    state.onEnter?.(null);
  }
  
  trigger(event) {
    const transitions = this.transitions.get(this.currentState);
    if (!transitions) return false;
    
    const targetState = transitions.get(event);
    if (!targetState) return false;
    
    this._transition(targetState);
    return true;
  }
  
  _transition(newStateName) {
    const currentStateObj = this.states.get(this.currentState);
    const newStateObj = this.states.get(newStateName);
    
    if (!newStateObj) return;
    
    currentStateObj?.onExit?.(newStateName);
    
    this.previousState = this.currentState;
    this.currentState = newStateName;
    
    newStateObj.onEnter?.(this.previousState);
  }
  
  update(dt) {
    const state = this.states.get(this.currentState);
    state?.onUpdate?.(dt);
  }
  
  isIn(stateName) { return this.currentState === stateName; }
}

// Game States
class LoadingState {
  constructor(game) {
    this.game = game;
    this.progress = 0;
  }
  
  onEnter(from) {
    console.log('Entering Loading State');
    document.getElementById('loading').style.display = 'flex';
  }
  
  onUpdate(dt) {
    // Check loading progress
    if (this.game.assetManager.isComplete()) {
      this.game.stateMachine.trigger('load_complete');
    }
  }
  
  onExit(to) {
    document.getElementById('loading').style.display = 'none';
  }
}

class MenuState {
  constructor(game) {
    this.game = game;
  }
  
  onEnter(from) {
    console.log('Entering Menu State');
    document.getElementById('menu').style.display = 'flex';
  }
  
  onUpdate(dt) {}
  
  onExit(to) {
    document.getElementById('menu').style.display = 'none';
  }
}

class PlayingState {
  constructor(game) {
    this.game = game;
    this.paused = false;
  }
  
  onEnter(from) {
    console.log('Entering Playing State');
    this.game.inputManager.lockPointer();
    document.getElementById('hud').style.display = 'block';
  }
  
  onUpdate(dt) {
    if (!this.paused) {
      this.game.world.update(dt);
    }
  }
  
  onExit(to) {
    document.getElementById('hud').style.display = 'none';
  }
}

class PausedState {
  onEnter(from) {
    document.getElementById('pauseMenu').style.display = 'flex';
  }
  
  onUpdate(dt) {}
  
  onExit(to) {
    document.getElementById('pauseMenu').style.display = 'none';
  }
}

class GameOverState {
  constructor(game) { this.game = game; }
  
  onEnter(from) {
    document.getElementById('gameOver').style.display = 'flex';
    this.game.inputManager.unlockPointer();
  }
  
  onUpdate(dt) {}
  
  onExit(to) {
    document.getElementById('gameOver').style.display = 'none';
  }
}
```

---

## 4. Event System

```javascript
class EventSystem {
  constructor() {
    this.listeners = new Map();
    this.onceListeners = new Map();
    this.queue = [];
    this.processing = false;
  }
  
  on(event, listener) {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event).add(listener);
    return () => this.off(event, listener);  // unsubscribe function
  }
  
  once(event, listener) {
    if (!this.onceListeners.has(event)) {
      this.onceListeners.set(event, new Set());
    }
    this.onceListeners.get(event).add(listener);
  }
  
  off(event, listener) {
    this.listeners.get(event)?.delete(listener);
  }
  
  emit(event, data) {
    // Immediate dispatch
    const listeners = this.listeners.get(event);
    if (listeners) {
      for (const listener of listeners) {
        listener(data);
      }
    }
    
    // One-time listeners
    const once = this.onceListeners.get(event);
    if (once) {
      for (const listener of once) {
        listener(data);
      }
      this.onceListeners.delete(event);
    }
  }
  
  // Queue events สำหรับ process ใน update loop
  queue(event, data) {
    this.queue.push({ event, data });
  }
  
  processQueue() {
    const toProcess = [...this.queue];
    this.queue.length = 0;
    
    for (const { event, data } of toProcess) {
      this.emit(event, data);
    }
  }
}
```

---

## 5. Input Manager

```javascript
class InputManager {
  constructor() {
    this.keys = new Map();
    this.keysJustPressed = new Set();
    this.keysJustReleased = new Set();
    
    this.mouseButtons = new Map();
    this.mouseButtonsJustPressed = new Set();
    
    this.mouseDelta = new THREE.Vector2();
    this.mousePosition = new THREE.Vector2();
    
    this.isPointerLocked = false;
    
    this._setupListeners();
  }
  
  _setupListeners() {
    document.addEventListener('keydown', e => {
      const key = e.code.toLowerCase();
      if (!this.keys.get(key)) {
        this.keysJustPressed.add(key);
      }
      this.keys.set(key, true);
    });
    
    document.addEventListener('keyup', e => {
      const key = e.code.toLowerCase();
      this.keys.set(key, false);
      this.keysJustReleased.add(key);
    });
    
    document.addEventListener('mousedown', e => {
      const btn = `mouse${e.button}`;
      if (!this.mouseButtons.get(btn)) {
        this.mouseButtonsJustPressed.add(btn);
      }
      this.mouseButtons.set(btn, true);
    });
    
    document.addEventListener('mouseup', e => {
      this.mouseButtons.set(`mouse${e.button}`, false);
    });
    
    document.addEventListener('mousemove', e => {
      if (this.isPointerLocked) {
        this.mouseDelta.x += e.movementX;
        this.mouseDelta.y += e.movementY;
      } else {
        this.mousePosition.set(
          (e.clientX / window.innerWidth) * 2 - 1,
          -(e.clientY / window.innerHeight) * 2 + 1
        );
      }
    });
    
    document.addEventListener('pointerlockchange', () => {
      this.isPointerLocked = !!document.pointerLockElement;
    });
  }
  
  isDown(key) { return this.keys.get(key) || false; }
  isMouseDown(btn) { return this.mouseButtons.get(`mouse${btn}`) || false; }
  isJustPressed(key) { return this.keysJustPressed.has(key); }
  isJustReleased(key) { return this.keysJustReleased.has(key); }
  isMouseJustPressed(btn) { return this.mouseButtonsJustPressed.has(`mouse${btn}`); }
  
  lockPointer() {
    document.body.requestPointerLock();
  }
  
  unlockPointer() {
    document.exitPointerLock();
  }
  
  // ต้องเรียก endFrame() ที่ท้าย loop ทุก frame
  endFrame() {
    this.keysJustPressed.clear();
    this.keysJustReleased.clear();
    this.mouseButtonsJustPressed.clear();
    this.mouseDelta.set(0, 0);
  }
}
```

---

## 6. Asset Manager

```javascript
class AssetManager {
  constructor() {
    this.assets = new Map();
    this.loaders = new Map();
    this.loadQueue = [];
    this.totalToLoad = 0;
    this.loaded = 0;
    this.errors = [];
    this.complete = false;
    
    // Setup loaders
    const gltfLoader = new THREE.GLTFLoader();
    const textureLoader = new THREE.TextureLoader();
    const audioLoader = new THREE.AudioLoader();
    
    this.loaders.set('gltf', gltfLoader);
    this.loaders.set('texture', textureLoader);
    this.loaders.set('audio', audioLoader);
  }
  
  // Queue asset สำหรับ load
  queue(type, key, url, options = {}) {
    this.loadQueue.push({ type, key, url, options });
    return this;  // chainable
  }
  
  // เริ่ม load ทุกอย่างใน queue
  loadAll() {
    return new Promise((resolve, reject) => {
      if (this.loadQueue.length === 0) {
        this.complete = true;
        resolve(this.assets);
        return;
      }
      
      this.totalToLoad = this.loadQueue.length;
      this.loaded = 0;
      
      const promises = this.loadQueue.map(({ type, key, url, options }) => {
        return this._loadAsset(type, key, url, options)
          .then(asset => {
            this.assets.set(key, asset);
            this.loaded++;
            this._onProgress(key, this.loaded / this.totalToLoad);
          })
          .catch(err => {
            this.errors.push({ key, url, error: err });
            this.loaded++;
            console.error(`Error loading ${key}:`, err);
          });
      });
      
      Promise.all(promises).then(() => {
        this.complete = true;
        this.loadQueue.length = 0;
        resolve(this.assets);
      });
    });
  }
  
  _loadAsset(type, key, url, options) {
    return new Promise((resolve, reject) => {
      const loader = this.loaders.get(type);
      if (!loader) {
        reject(new Error(`No loader for type: ${type}`));
        return;
      }
      
      loader.load(url, resolve, null, reject);
    });
  }
  
  _onProgress(key, progress) {
    if (this.onProgress) {
      this.onProgress(key, progress, this.loaded, this.totalToLoad);
    }
  }
  
  get(key) { return this.assets.get(key); }
  isComplete() { return this.complete; }
  getProgress() { return this.totalToLoad > 0 ? this.loaded / this.totalToLoad : 1; }
}
```

---

## 7. Object Pool

```javascript
class ObjectPool {
  constructor(factory, initialSize = 10) {
    this.factory = factory;
    this.available = [];
    this.inUse = new Set();
    
    // Pre-populate
    for (let i = 0; i < initialSize; i++) {
      this.available.push(this._create());
    }
  }
  
  _create() {
    const obj = this.factory();
    obj._pooled = true;
    return obj;
  }
  
  get() {
    let obj;
    
    if (this.available.length > 0) {
      obj = this.available.pop();
    } else {
      // Expand pool
      obj = this._create();
    }
    
    this.inUse.add(obj);
    obj._active = true;
    return obj;
  }
  
  release(obj) {
    if (!this.inUse.has(obj)) return;
    
    obj._active = false;
    this.inUse.delete(obj);
    this.available.push(obj);
    
    // Reset object state
    obj.onRelease?.();
  }
  
  releaseAll() {
    for (const obj of this.inUse) {
      this.release(obj);
    }
  }
  
  get size() { return this.available.length + this.inUse.size; }
  get activeCount() { return this.inUse.size; }
}

// ตัวอย่าง: Bullet pool
class BulletPool extends ObjectPool {
  constructor(scene) {
    const geo = new THREE.SphereGeometry(0.1, 8, 8);
    const mat = new THREE.MeshBasicMaterial({ color: 0xffff00 });
    
    super(() => {
      const mesh = new THREE.Mesh(geo, mat);
      mesh.visible = false;
      
      const bullet = {
        mesh,
        velocity: new THREE.Vector3(),
        lifetime: 0,
        maxLifetime: 3,
        active: false,
        
        fire(position, direction, speed) {
          this.mesh.position.copy(position);
          this.velocity.copy(direction).multiplyScalar(speed);
          this.lifetime = 0;
          this.mesh.visible = true;
          this.active = true;
        },
        
        update(dt, pool) {
          if (!this.active) return;
          
          this.lifetime += dt;
          this.mesh.position.addScaledVector(this.velocity, dt);
          
          if (this.lifetime >= this.maxLifetime) {
            pool.release(this);
          }
        },
        
        onRelease() {
          this.mesh.visible = false;
          this.active = false;
        }
      };
      
      scene.add(mesh);
      return bullet;
    }, 50);
    
    this.scene = scene;
  }
  
  fireBullet(position, direction, speed = 20) {
    const bullet = this.get();
    bullet.fire(position, direction, speed);
    return bullet;
  }
  
  update(dt) {
    for (const bullet of this.inUse) {
      bullet.update(dt, this);
    }
  }
}
```

---

## 8. Complete Game Template

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title>Simple Game - Three.js ECS</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { overflow: hidden; background: #000; }
    canvas { display: block; }
    
    /* HUD */
    #hud {
      position: fixed;
      top: 0; left: 0;
      width: 100%; height: 100%;
      pointer-events: none;
      display: none;
    }
    .hp-bar {
      position: absolute;
      bottom: 20px; left: 20px;
      width: 200px; height: 20px;
      background: rgba(0,0,0,0.5);
      border: 2px solid #555;
      border-radius: 10px;
      overflow: hidden;
    }
    .hp-fill {
      height: 100%;
      background: linear-gradient(to right, #f00, #0f0);
      transition: width 0.3s;
    }
    #score {
      position: absolute;
      top: 20px; right: 20px;
      color: white;
      font-family: monospace;
      font-size: 24px;
    }
    
    /* Menus */
    .screen {
      position: fixed;
      top: 0; left: 0;
      width: 100%; height: 100%;
      background: rgba(0,0,0,0.8);
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      color: white;
    }
    .screen h1 { font-size: 48px; margin-bottom: 30px; font-family: monospace; }
    .menu-btn {
      padding: 12px 40px;
      font-size: 18px;
      background: #334;
      border: 2px solid #668;
      color: white;
      cursor: pointer;
      border-radius: 5px;
      margin: 5px;
      font-family: monospace;
    }
    .menu-btn:hover { background: #446; }
    #loading .progress-bar {
      width: 300px; height: 10px;
      background: #333; border-radius: 5px;
      overflow: hidden; margin-top: 10px;
    }
    #loading .progress-fill {
      height: 100%;
      background: #88aaff;
      transition: width 0.3s;
    }
    #crosshair {
      position: fixed;
      top: 50%; left: 50%;
      transform: translate(-50%, -50%);
      width: 20px; height: 20px;
      display: none;
      pointer-events: none;
    }
    #crosshair::before, #crosshair::after {
      content: '';
      position: absolute;
      background: rgba(255,255,255,0.8);
    }
    #crosshair::before { width: 2px; height: 100%; left: 50%; transform: translateX(-50%); }
    #crosshair::after { height: 2px; width: 100%; top: 50%; transform: translateY(-50%); }
  </style>
</head>
<body>
  <canvas id="canvas"></canvas>
  
  <div id="loading" class="screen">
    <h1>กำลังโหลด...</h1>
    <div class="progress-bar"><div class="progress-fill" id="loadProgress" style="width: 0%"></div></div>
    <div id="loadText" style="margin-top: 10px; font-size: 14px;">0%</div>
  </div>
  
  <div id="menu" class="screen" style="display: none;">
    <h1 style="color: #88aaff;">Simple 3D Game</h1>
    <p style="margin-bottom: 20px; color: #aaa;">ใช้ WASD เดิน, คลิกซ้ายเพื่อเล่น</p>
    <button class="menu-btn" id="startBtn">เริ่มเกม</button>
  </div>
  
  <div id="pauseMenu" class="screen" style="display: none;">
    <h1>พัก</h1>
    <button class="menu-btn" id="resumeBtn">เล่นต่อ</button>
    <button class="menu-btn" id="menuBtn">เมนูหลัก</button>
  </div>
  
  <div id="gameOver" class="screen" style="display: none;">
    <h1 style="color: #f44;">Game Over</h1>
    <div id="finalScore" style="font-size: 24px; margin-bottom: 20px;">Score: 0</div>
    <button class="menu-btn" id="restartBtn">เล่นอีกครั้ง</button>
  </div>
  
  <div id="hud">
    <div class="hp-bar"><div class="hp-fill" id="hpFill"></div></div>
    <div id="score">Score: 0</div>
  </div>
  <div id="crosshair"></div>

  <script type="module">
    import * as THREE from 'https://cdn.skypack.dev/three@0.160.0';
    import { OrbitControls } from 'https://cdn.skypack.dev/three@0.160.0/examples/jsm/controls/OrbitControls.js';
    
    // === Simplified Game Setup ===
    const canvas = document.getElementById('canvas');
    const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.shadowMap.enabled = true;
    renderer.outputColorSpace = THREE.SRGBColorSpace;
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x1a1a2e);
    scene.fog = new THREE.Fog(0x1a1a2e, 20, 60);
    
    const camera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.1, 100);
    camera.position.set(0, 10, 15);
    
    const controls = new OrbitControls(camera, canvas);
    controls.enableDamping = true;
    
    // Lighting
    scene.add(new THREE.AmbientLight(0x222244, 1));
    const dirLight = new THREE.DirectionalLight(0xffffff, 2);
    dirLight.position.set(10, 20, 10);
    dirLight.castShadow = true;
    scene.add(dirLight);
    
    // Floor
    const floor = new THREE.Mesh(
      new THREE.PlaneGeometry(50, 50),
      new THREE.MeshStandardMaterial({ color: 0x222233, roughness: 0.9 })
    );
    floor.rotation.x = -Math.PI / 2;
    floor.receiveShadow = true;
    scene.add(floor);
    
    // Player
    const playerGeo = new THREE.CapsuleGeometry(0.4, 1.2, 4, 8);
    const playerMat = new THREE.MeshStandardMaterial({ color: 0x4488ff });
    const player = new THREE.Mesh(playerGeo, playerMat);
    player.position.y = 1;
    player.castShadow = true;
    scene.add(player);
    
    // Enemies
    const enemies = [];
    let score = 0;
    
    function spawnEnemy() {
      const angle = Math.random() * Math.PI * 2;
      const r = 10 + Math.random() * 10;
      
      const geo = new THREE.BoxGeometry(0.8, 1.2, 0.8);
      const mat = new THREE.MeshStandardMaterial({ color: 0xff3300 });
      const mesh = new THREE.Mesh(geo, mat);
      mesh.position.set(Math.cos(angle) * r, 0.6, Math.sin(angle) * r);
      mesh.castShadow = true;
      scene.add(mesh);
      
      enemies.push({ mesh, health: 3, speed: 1.5 + Math.random() });
    }
    
    for (let i = 0; i < 10; i++) spawnEnemy();
    
    // Game state
    let gameState = 'menu';
    let playerHealth = 100;
    
    // Input
    const keys = {};
    window.addEventListener('keydown', e => keys[e.code] = true);
    window.addEventListener('keyup', e => keys[e.code] = false);
    
    // Game start/end
    document.getElementById('startBtn').onclick = () => {
      gameState = 'playing';
      document.getElementById('menu').style.display = 'none';
      document.getElementById('hud').style.display = 'block';
      document.getElementById('crosshair').style.display = 'block';
    };
    
    document.getElementById('restartBtn').onclick = () => {
      // Reset
      playerHealth = 100;
      score = 0;
      document.getElementById('gameOver').style.display = 'none';
      document.getElementById('hud').style.display = 'block';
      gameState = 'playing';
    };
    
    // Hide loading quickly since we have no assets to load
    document.getElementById('loading').style.display = 'none';
    document.getElementById('menu').style.display = 'flex';
    
    // Animation
    const clock = new THREE.Clock();
    const tmpVec = new THREE.Vector3();
    
    function animate() {
      requestAnimationFrame(animate);
      const dt = Math.min(clock.getDelta(), 0.05);
      
      if (gameState === 'playing') {
        // Player movement
        const speed = 5;
        if (keys['KeyW']) player.position.z -= speed * dt;
        if (keys['KeyS']) player.position.z += speed * dt;
        if (keys['KeyA']) player.position.x -= speed * dt;
        if (keys['KeyD']) player.position.x += speed * dt;
        
        // Camera follow player
        camera.position.lerp(
          new THREE.Vector3(player.position.x, player.position.y + 8, player.position.z + 12),
          dt * 3
        );
        controls.target.lerp(player.position, dt * 3);
        
        // Update enemies
        for (let i = enemies.length - 1; i >= 0; i--) {
          const e = enemies[i];
          
          // Move toward player
          tmpVec.copy(player.position).sub(e.mesh.position).normalize();
          e.mesh.position.addScaledVector(tmpVec, e.speed * dt);
          e.mesh.lookAt(player.position);
          
          // Attack player
          if (e.mesh.position.distanceTo(player.position) < 1.2) {
            playerHealth -= 10 * dt;
            e.mesh.material.color.setHex(0xff8800);
          } else {
            e.mesh.material.color.setHex(0xff3300);
          }
          
          // Bobbing
          e.mesh.position.y = 0.6 + Math.sin(clock.elapsedTime * 3 + i) * 0.1;
        }
        
        // Update HUD
        document.getElementById('hpFill').style.width = `${Math.max(0, playerHealth)}%`;
        document.getElementById('score').textContent = `Score: ${score}`;
        
        // Check game over
        if (playerHealth <= 0) {
          gameState = 'gameover';
          document.getElementById('hud').style.display = 'none';
          document.getElementById('crosshair').style.display = 'none';
          document.getElementById('gameOver').style.display = 'flex';
          document.getElementById('finalScore').textContent = `Score: ${score}`;
        }
        
        // Spawn enemies over time
        if (Math.random() < 0.005 && enemies.length < 20) {
          spawnEnemy();
        }
      }
      
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

## 9. สรุป Game Architecture

### สิ่งที่ได้เรียนรู้:
1. **ECS Pattern**: แยก Entity, Component, System ออกจากกัน
2. **Fixed Timestep**: ทำให้ physics deterministic
3. **State Machine**: จัดการ game states อย่างมีระเบียบ
4. **Event System**: สื่อสารระหว่าง systems
5. **Input Manager**: จัดการ input อย่างเป็นระบบ
6. **Asset Manager**: จัดการการโหลด assets
7. **Object Pool**: ลด GC pressure

### Best Practices:
- ใช้ fixed timestep สำหรับ physics เสมอ
- Query components ให้น้อยครั้งที่สุดใน hot loops
- Pool objects ที่ create/destroy บ่อย
- แยก rendering state จาก game state

ในบทถัดไป (Part 86) เราจะเรียนรู้เกี่ยวกับ 3D Data Visualization
