# Part 89: Character Animation System

## บทนำ: Character Animation ใน Three.js

Three.js รองรับ skeletal animation ผ่าน GLTF format ซึ่งเป็น standard ของ industry เราสามารถใช้ AnimationMixer ในการเล่น animations และ blend ระหว่าง animations ต่างๆ

---

## 1. Humanoid Rig Conventions

### GLTF Bone Naming

```
Root
└── Hips (pelvis)
    ├── Spine
    │   ├── Spine1
    │   │   ├── Spine2
    │   │   │   ├── Neck
    │   │   │   │   └── Head
    │   │   │   ├── LeftShoulder
    │   │   │   │   └── LeftArm (upper arm)
    │   │   │   │       └── LeftForeArm
    │   │   │   │           └── LeftHand
    │   │   │   └── RightShoulder
    │   │   │       └── RightArm
    │   │   │           └── RightForeArm
    │   │   │               └── RightHand
    ├── LeftUpLeg (thigh)
    │   └── LeftLeg (shin)
    │       └── LeftFoot
    │           └── LeftToeBase
    └── RightUpLeg
        └── RightLeg
            └── RightFoot
                └── RightToeBase
```

### โหลด GLTF Character

```javascript
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';

async function loadCharacter(url) {
  const loader = new GLTFLoader();
  const gltf = await new Promise((resolve, reject) => {
    loader.load(url, resolve, undefined, reject);
  });
  
  const model = gltf.scene;
  const mixer = new THREE.AnimationMixer(model);
  
  // Extract animations
  const animations = {};
  gltf.animations.forEach(clip => {
    animations[clip.name] = mixer.clipAction(clip);
  });
  
  // Setup shadows
  model.traverse(obj => {
    if (obj.isMesh) {
      obj.castShadow = true;
      obj.receiveShadow = true;
    }
  });
  
  // Find bones
  const bones = {};
  model.traverse(obj => {
    if (obj.isBone || obj.type === 'Bone') {
      bones[obj.name] = obj;
    }
  });
  
  return { model, mixer, animations, bones };
}
```

---

## 2. Animation Blend Tree

```javascript
class AnimationBlendTree {
  constructor(mixer) {
    this.mixer = mixer;
    this.layers = new Map();
    this.baseAction = null;
    
    this._setupLayers();
  }
  
  _setupLayers() {
    // Layer 0: Base locomotion (walk/run/idle)
    this.layers.set('base', {
      actions: new Map(),
      currentAction: null,
      fadeTime: 0.3
    });
    
    // Layer 1: Upper body (additive - shooting, waving)
    this.layers.set('upper', {
      actions: new Map(),
      currentAction: null,
      fadeTime: 0.2,
      mask: null  // จะ setup ใน setupUpperBodyMask
    });
    
    // Layer 2: Face/expression
    this.layers.set('face', {
      actions: new Map(),
      currentAction: null,
      fadeTime: 0.1
    });
  }
  
  // เพิ่ม animation ไปยัง layer
  addAnimation(layerName, name, action, options = {}) {
    const layer = this.layers.get(layerName);
    if (!layer) return;
    
    if (options.loop !== undefined) {
      action.setLoop(options.loop ? THREE.LoopRepeat : THREE.LoopOnce);
    }
    
    if (options.clampWhenFinished !== undefined) {
      action.clampWhenFinished = options.clampWhenFinished;
    }
    
    layer.actions.set(name, action);
  }
  
  // เปลี่ยน animation ใน layer
  crossFade(layerName, toName, duration) {
    const layer = this.layers.get(layerName);
    if (!layer) return;
    
    const toAction = layer.actions.get(toName);
    if (!toAction) return;
    
    const fromAction = layer.currentAction;
    
    if (fromAction === toAction) return;
    
    toAction.reset();
    toAction.setEffectiveTimeScale(1);
    toAction.setEffectiveWeight(1);
    toAction.play();
    
    if (fromAction) {
      fromAction.crossFadeTo(toAction, duration, true);
    }
    
    layer.currentAction = toAction;
  }
  
  // Blend tree สำหรับ locomotion (walk/run ตาม speed)
  blendLocomotion(speed) {
    const layer = this.layers.get('base');
    if (!layer) return;
    
    const idle = layer.actions.get('idle');
    const walk = layer.actions.get('walk');
    const run = layer.actions.get('run');
    
    if (!idle || !walk || !run) return;
    
    // Speed ranges:
    // 0-0.1: idle
    // 0.1-3: walk (blend idle->walk)
    // 3-8: run (blend walk->run)
    
    if (speed < 0.1) {
      // Pure idle
      if (layer.currentAction !== idle) {
        idle.reset().play();
        if (layer.currentAction) layer.currentAction.crossFadeTo(idle, 0.3, true);
        layer.currentAction = idle;
      }
    } else if (speed < 3) {
      // Blend idle -> walk
      const t = (speed - 0.1) / 2.9;
      
      if (!walk.isRunning()) {
        walk.reset().play();
      }
      if (!idle.isRunning()) {
        idle.reset().play();
      }
      
      walk.setEffectiveWeight(t);
      idle.setEffectiveWeight(1 - t);
      run.setEffectiveWeight(0);
    } else {
      // Blend walk -> run
      const t = Math.min((speed - 3) / 5, 1);
      
      if (!run.isRunning()) {
        run.reset().play();
      }
      if (!walk.isRunning()) {
        walk.reset().play();
      }
      
      run.setEffectiveWeight(t);
      walk.setEffectiveWeight(1 - t);
      idle.setEffectiveWeight(0);
    }
  }
}
```

---

## 3. Inverse Kinematics (IK)

### Foot IK (ปรับตำแหน่งเท้าตาม terrain)

```javascript
class FootIK {
  constructor(character, bones) {
    this.character = character;
    
    // Bone references
    this.leftThigh = bones['LeftUpLeg'];
    this.leftShin = bones['LeftLeg'];
    this.leftFoot = bones['LeftFoot'];
    this.rightThigh = bones['RightUpLeg'];
    this.rightShin = bones['RightLeg'];
    this.rightFoot = bones['RightFoot'];
    
    // Target positions สำหรับแต่ละเท้า
    this.leftTarget = new THREE.Vector3();
    this.rightTarget = new THREE.Vector3();
    
    this.raycaster = new THREE.Raycaster();
    
    // IK state
    this.leftFootPos = new THREE.Vector3();
    this.rightFootPos = new THREE.Vector3();
    this.leftFootWeight = 1;
    this.rightFootWeight = 1;
  }
  
  // 2-bone IK (FABRIK simplified)
  solveTwoBoneIK(thigh, shin, foot, targetPos, poleVector) {
    if (!thigh || !shin || !foot) return;
    
    const thighWorldPos = new THREE.Vector3();
    const footWorldPos = new THREE.Vector3();
    
    thigh.getWorldPosition(thighWorldPos);
    
    // คำนวณความยาว bones
    const shinWorldPos = new THREE.Vector3();
    shin.getWorldPosition(shinWorldPos);
    
    const upperLen = thighWorldPos.distanceTo(shinWorldPos);
    const lowerLen = shinWorldPos.distanceTo(footWorldPos);
    const totalLen = upperLen + lowerLen;
    
    // Distance ไปยัง target
    const targetDist = thighWorldPos.distanceTo(targetPos);
    const clampedDist = Math.min(targetDist, totalLen * 0.99);
    
    // Vector จาก thigh ไป target
    const dir = targetPos.clone().sub(thighWorldPos).normalize();
    
    // Calculate middle joint angle (cosine rule)
    const cosAngle = (upperLen * upperLen + clampedDist * clampedDist - lowerLen * lowerLen) 
                     / (2 * upperLen * clampedDist);
    const angle = Math.acos(Math.max(-1, Math.min(1, cosAngle)));
    
    // Middle joint position
    const midPos = thighWorldPos.clone()
      .add(dir.clone().multiplyScalar(Math.cos(angle) * upperLen));
    
    // Apply pole vector (hint for knee/elbow direction)
    if (poleVector) {
      const poleDir = poleVector.clone().sub(thighWorldPos).normalize();
      const perpDir = dir.clone().cross(poleDir).normalize();
      midPos.add(perpDir.multiplyScalar(Math.sin(angle) * upperLen));
    }
    
    // Apply rotations to bones
    const thighUp = midPos.clone().sub(thighWorldPos).normalize();
    const worldQuat = new THREE.Quaternion();
    thigh.getWorldQuaternion(worldQuat);
    
    // Convert to local space and apply
    const localDir = thighUp.clone().applyQuaternion(worldQuat.invert());
    const defaultDir = new THREE.Vector3(0, -1, 0);
    const rotQuat = new THREE.Quaternion().setFromUnitVectors(defaultDir, localDir);
    thigh.quaternion.multiply(rotQuat);
  }
  
  update(scene, deltaTime) {
    // Raycast สำหรับเท้าแต่ละข้าง
    [
      { foot: this.leftFoot, target: this.leftTarget, pos: this.leftFootPos },
      { foot: this.rightFoot, target: this.rightTarget, pos: this.rightFootPos }
    ].forEach(({ foot, target, pos }) => {
      if (!foot) return;
      
      const footWorldPos = new THREE.Vector3();
      foot.getWorldPosition(footWorldPos);
      
      // Raycast ลง
      this.raycaster.set(
        new THREE.Vector3(footWorldPos.x, footWorldPos.y + 1, footWorldPos.z),
        new THREE.Vector3(0, -1, 0)
      );
      
      const hits = this.raycaster.intersectObjects(
        scene.children.filter(o => o.userData.isTerrain),
        true
      );
      
      if (hits.length > 0) {
        target.copy(hits[0].point);
        target.y += 0.05;  // small offset
      } else {
        target.copy(footWorldPos);
      }
      
      // Smooth IK target movement
      pos.lerp(target, deltaTime * 8);
    });
    
    // Apply IK
    if (this.leftThigh && this.leftShin && this.leftFoot) {
      this.solveTwoBoneIK(
        this.leftThigh, 
        this.leftShin, 
        this.leftFoot,
        this.leftFootPos,
        null  // pole vector
      );
    }
    
    if (this.rightThigh && this.rightShin && this.rightFoot) {
      this.solveTwoBoneIK(
        this.rightThigh,
        this.rightShin,
        this.rightFoot,
        this.rightFootPos,
        null
      );
    }
  }
}
```

---

## 4. Look-At IK (Head/Eye)

```javascript
class LookAtIK {
  constructor(bones, options = {}) {
    this.head = bones['Head'];
    this.neck = bones['Neck'];
    this.eyes = {
      left: bones['LeftEye'],
      right: bones['RightEye']
    };
    
    this.options = {
      headWeight: 0.7,
      eyeWeight: 1.0,
      maxHeadAngle: Math.PI / 3,
      maxEyeAngle: Math.PI / 4,
      smoothSpeed: 3.0,
      ...options
    };
    
    this.target = new THREE.Vector3();
    this.smoothTarget = new THREE.Vector3();
  }
  
  setTarget(target) {
    this.target.copy(target);
  }
  
  update(deltaTime) {
    // Smooth the target
    this.smoothTarget.lerp(this.target, deltaTime * this.options.smoothSpeed);
    
    // Head look-at
    if (this.head) {
      const headWorldPos = new THREE.Vector3();
      this.head.getWorldPosition(headWorldPos);
      
      const dirToTarget = this.smoothTarget.clone().sub(headWorldPos).normalize();
      
      // Clamp angle
      const forwardDir = new THREE.Vector3(0, 0, 1)
        .applyQuaternion(this.head.getWorldQuaternion(new THREE.Quaternion()));
      
      const angle = Math.acos(Math.max(-1, Math.min(1, dirToTarget.dot(forwardDir))));
      const clampedAngle = Math.min(angle, this.options.maxHeadAngle);
      
      if (clampedAngle > 0.01) {
        const rotAxis = forwardDir.clone().cross(dirToTarget).normalize();
        const targetQuat = new THREE.Quaternion()
          .setFromAxisAngle(rotAxis, clampedAngle * this.options.headWeight);
        
        this.head.quaternion.multiply(targetQuat);
      }
    }
    
    // Eye look-at (simpler)
    ['left', 'right'].forEach(side => {
      const eye = this.eyes[side];
      if (!eye) return;
      
      const eyeWorldPos = new THREE.Vector3();
      eye.getWorldPosition(eyeWorldPos);
      
      const dirToTarget = this.smoothTarget.clone().sub(eyeWorldPos).normalize();
      
      // Apply rotation
      const targetQuat = new THREE.Quaternion();
      const defaultDir = new THREE.Vector3(0, 0, 1)
        .applyQuaternion(eye.getWorldQuaternion(new THREE.Quaternion()));
      
      targetQuat.setFromUnitVectors(defaultDir, dirToTarget);
      eye.quaternion.multiply(targetQuat);
    });
  }
}
```

---

## 5. Procedural Animation (Spring Bones)

```javascript
// Spring simulation สำหรับ hair, clothes, tail
class SpringBone {
  constructor(bone, options = {}) {
    this.bone = bone;
    this.options = {
      stiffness: 5.0,      // ความแข็ง (สูง = bounce มาก)
      damping: 2.0,        // ความหน่วง (สูง = หยุดเร็ว)
      gravity: -0.1,       // gravity
      windStrength: 0.0,   // แรงลม
      ...options
    };
    
    // Current state
    this.position = new THREE.Vector3();
    this.velocity = new THREE.Vector3();
    
    // Target (resting) position
    this.restPosition = new THREE.Vector3();
    bone.getWorldPosition(this.restPosition);
  }
  
  update(deltaTime, windDir = null) {
    const worldPos = new THREE.Vector3();
    this.bone.getWorldPosition(worldPos);
    
    // Spring force ดึงกลับ rest position
    const displacement = this.restPosition.clone().sub(this.position);
    const springForce = displacement.multiplyScalar(this.options.stiffness);
    
    // Damping force
    const dampForce = this.velocity.clone().multiplyScalar(-this.options.damping);
    
    // Gravity
    const gravForce = new THREE.Vector3(0, this.options.gravity, 0);
    
    // Wind
    let windForce = new THREE.Vector3();
    if (windDir && this.options.windStrength > 0) {
      windForce = windDir.clone().multiplyScalar(this.options.windStrength);
    }
    
    // Total force
    const totalForce = springForce.add(dampForce).add(gravForce).add(windForce);
    
    // Update velocity
    this.velocity.addScaledVector(totalForce, deltaTime);
    
    // Update position
    this.position.addScaledVector(this.velocity, deltaTime);
    
    // Keep on sphere radius from parent
    const parent = this.bone.parent;
    if (parent) {
      const parentWorldPos = new THREE.Vector3();
      parent.getWorldPosition(parentWorldPos);
      
      const boneLength = this.restPosition.distanceTo(parentWorldPos);
      const dirFromParent = this.position.clone().sub(parentWorldPos).normalize();
      this.position.copy(parentWorldPos).addScaledVector(dirFromParent, boneLength);
    }
    
    // Apply position to bone
    this.bone.getWorldPosition(worldPos);
    const offset = this.position.clone().sub(worldPos);
    this.bone.position.add(this.bone.parent 
      ? offset.applyMatrix4(this.bone.parent.matrixWorld.invert()) 
      : offset
    );
  }
}

class SpringBoneSystem {
  constructor() {
    this.springs = [];
    this.windDirection = new THREE.Vector3(1, 0, 0);
    this.windStrength = 0;
    this.windTime = 0;
  }
  
  addBone(bone, options) {
    const spring = new SpringBone(bone, options);
    this.springs.push(spring);
    return spring;
  }
  
  // เพิ่ม spring chain (เช่น hair strands)
  addBoneChain(rootBone, options) {
    const bones = [];
    let current = rootBone;
    
    while (current) {
      const spring = this.addBone(current, {
        ...options,
        // bones ที่อยู่ไกลออกไปมีความ flexible มากกว่า
        stiffness: options.stiffness * (1 - bones.length * 0.1),
        damping: options.damping * (1 + bones.length * 0.1)
      });
      bones.push(spring);
      current = current.children[0];
    }
    
    return bones;
  }
  
  update(deltaTime) {
    // Animate wind
    this.windTime += deltaTime;
    const windX = Math.sin(this.windTime * 0.5) * 0.3 + Math.sin(this.windTime * 1.3) * 0.1;
    const windZ = Math.cos(this.windTime * 0.7) * 0.2;
    this.windDirection.set(windX, 0, windZ).normalize();
    
    this.springs.forEach(spring => {
      spring.options.windStrength = this.windStrength;
      spring.update(deltaTime, this.windDirection);
    });
  }
}
```

---

## 6. Complete Character Controller

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title>Character Controller with IK</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { overflow: hidden; background: #1a1a2e; font-family: monospace; }
    canvas { display: block; }
    #controls {
      position: fixed;
      bottom: 20px;
      left: 50%;
      transform: translateX(-50%);
      background: rgba(0,0,0,0.6);
      color: white;
      padding: 10px 20px;
      border-radius: 5px;
      font-size: 12px;
      text-align: center;
    }
    #info {
      position: fixed;
      top: 10px;
      left: 10px;
      background: rgba(0,0,0,0.6);
      color: #0f0;
      padding: 10px;
      border-radius: 5px;
      font-size: 12px;
    }
    #anim-state {
      position: fixed;
      top: 10px;
      right: 10px;
      background: rgba(0,0,0,0.6);
      color: #88aaff;
      padding: 10px;
      border-radius: 5px;
      font-size: 12px;
    }
  </style>
</head>
<body>
  <canvas id="canvas"></canvas>
  <div id="controls">WASD: เดิน | Shift: วิ่ง | Space: กระโดด | Click: Look-At | T: Tpose</div>
  <div id="info">
    <div>Speed: <span id="speed">0</span></div>
    <div>State: <span id="state">idle</span></div>
    <div>Spring: <span id="spring">on</span></div>
  </div>
  <div id="anim-state">Animation Blend Tree</div>
  
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
    scene.background = new THREE.Color(0x1a1a2e);
    scene.fog = new THREE.Fog(0x1a1a2e, 20, 50);
    
    const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 100);
    camera.position.set(0, 3, 6);
    
    const controls = new OrbitControls(camera, canvas);
    controls.enableDamping = true;
    controls.target.set(0, 1, 0);
    
    // Lighting
    scene.add(new THREE.AmbientLight(0x334466, 0.8));
    
    const dirLight = new THREE.DirectionalLight(0xffffff, 2);
    dirLight.position.set(5, 10, 5);
    dirLight.castShadow = true;
    dirLight.shadow.mapSize.set(2048, 2048);
    scene.add(dirLight);
    
    const fillLight = new THREE.DirectionalLight(0x4466ff, 0.5);
    fillLight.position.set(-5, 3, -3);
    scene.add(fillLight);
    
    // Environment
    const ground = new THREE.Mesh(
      new THREE.PlaneGeometry(30, 30),
      new THREE.MeshStandardMaterial({ color: 0x223344, roughness: 0.9 })
    );
    ground.rotation.x = -Math.PI / 2;
    ground.receiveShadow = true;
    scene.add(ground);
    
    // Grid overlay
    const grid = new THREE.GridHelper(30, 30, 0x334455, 0x223344);
    scene.add(grid);
    
    // ===== Procedural Character =====
    class ProceduralCharacter {
      constructor(scene) {
        this.scene = scene;
        this.group = new THREE.Group();
        scene.add(this.group);
        
        this.position = new THREE.Vector3(0, 0, 0);
        this.velocity = new THREE.Vector3();
        this.rotation = 0;
        this.targetRotation = 0;
        
        this.speed = 0;
        this.isGrounded = true;
        this.jumpVelocity = 0;
        
        this.animState = 'idle';
        this.animTime = 0;
        
        this.spring = { enabled: true };
        
        this._buildCharacter();
        this._buildSpringBones();
      }
      
      _buildCharacter() {
        const mat = (color, rough = 0.5, metal = 0) => 
          new THREE.MeshStandardMaterial({ color, roughness: rough, metalness: metal });
        
        // Body
        this.body = new THREE.Mesh(
          new THREE.CapsuleGeometry(0.25, 0.6, 4, 8),
          mat(0x4488ff, 0.4, 0.2)
        );
        this.body.position.y = 1.3;
        this.body.castShadow = true;
        this.group.add(this.body);
        
        // Head
        this.head = new THREE.Mesh(
          new THREE.SphereGeometry(0.2, 16, 16),
          mat(0xffcc99, 0.5)
        );
        this.head.position.y = 1.85;
        this.head.castShadow = true;
        this.group.add(this.head);
        
        // Eyes
        const eyeMat = mat(0x222222);
        this.leftEye = new THREE.Mesh(new THREE.SphereGeometry(0.03), eyeMat);
        this.leftEye.position.set(-0.07, 1.88, 0.17);
        this.rightEye = new THREE.Mesh(new THREE.SphereGeometry(0.03), eyeMat);
        this.rightEye.position.set(0.07, 1.88, 0.17);
        this.group.add(this.leftEye, this.rightEye);
        
        // Arms
        this.leftArm = new THREE.Group();
        this.leftArm.position.set(-0.35, 1.5, 0);
        
        this.leftUpperArm = new THREE.Mesh(
          new THREE.CapsuleGeometry(0.07, 0.3, 4, 6),
          mat(0x4488ff, 0.4)
        );
        this.leftUpperArm.position.y = -0.2;
        this.leftArm.add(this.leftUpperArm);
        
        this.leftForeArm = new THREE.Group();
        this.leftForeArm.position.y = -0.45;
        
        const leftForeArmMesh = new THREE.Mesh(
          new THREE.CapsuleGeometry(0.06, 0.25, 4, 6),
          mat(0xffcc99, 0.5)
        );
        leftForeArmMesh.position.y = -0.18;
        this.leftForeArm.add(leftForeArmMesh);
        this.leftArm.add(this.leftForeArm);
        
        this.group.add(this.leftArm);
        
        // Right arm (mirror)
        this.rightArm = this.leftArm.clone();
        this.rightArm.position.x = 0.35;
        this.group.add(this.rightArm);
        
        // Legs
        this.leftLeg = new THREE.Group();
        this.leftLeg.position.set(-0.15, 1.0, 0);
        
        this.leftThigh = new THREE.Mesh(
          new THREE.CapsuleGeometry(0.09, 0.35, 4, 6),
          mat(0x336699, 0.5)
        );
        this.leftThigh.position.y = -0.25;
        this.leftLeg.add(this.leftThigh);
        
        this.leftShin = new THREE.Group();
        this.leftShin.position.y = -0.55;
        
        const leftShinMesh = new THREE.Mesh(
          new THREE.CapsuleGeometry(0.07, 0.32, 4, 6),
          mat(0x336699, 0.5)
        );
        leftShinMesh.position.y = -0.22;
        this.leftShin.add(leftShinMesh);
        
        // Foot
        const leftFootMesh = new THREE.Mesh(
          new THREE.BoxGeometry(0.1, 0.06, 0.22),
          mat(0x222222, 0.7)
        );
        leftFootMesh.position.set(0, -0.47, 0.05);
        this.leftShin.add(leftFootMesh);
        
        this.leftLeg.add(this.leftShin);
        this.group.add(this.leftLeg);
        
        this.rightLeg = this.leftLeg.clone();
        this.rightLeg.position.x = 0.15;
        this.group.add(this.rightLeg);
        
        // Hair (for spring simulation)
        this.hair = [];
        for (let i = 0; i < 5; i++) {
          const strand = new THREE.Mesh(
            new THREE.CapsuleGeometry(0.015, 0.1 + i * 0.03, 4, 4),
            mat(0x332211, 0.9)
          );
          const angle = (i / 5) * Math.PI * 2;
          strand.position.set(
            Math.cos(angle) * 0.1,
            2.1 + i * 0.02,
            Math.sin(angle) * 0.1
          );
          this.group.add(strand);
          this.hair.push({
            mesh: strand,
            basePos: strand.position.clone(),
            vel: new THREE.Vector3(),
            springPos: strand.position.clone()
          });
        }
      }
      
      _buildSpringBones() {
        // Spring simulation สำหรับ hair
        this.springTime = 0;
      }
      
      update(dt, keys) {
        // Input
        let moveX = (keys['KeyD'] ? 1 : 0) - (keys['KeyA'] ? 1 : 0);
        let moveZ = (keys['KeyS'] ? 1 : 0) - (keys['KeyW'] ? 1 : 0);
        const isRunning = keys['ShiftLeft'] || keys['ShiftRight'];
        
        const moveLen = Math.sqrt(moveX * moveX + moveZ * moveZ);
        if (moveLen > 0) {
          moveX /= moveLen;
          moveZ /= moveLen;
        }
        
        // Speed
        const maxSpeed = isRunning ? 7 : 3;
        const targetSpeed = moveLen * maxSpeed;
        this.speed = THREE.MathUtils.lerp(this.speed, targetSpeed, dt * 8);
        
        // Movement
        if (moveLen > 0) {
          this.targetRotation = Math.atan2(moveX, moveZ) + Math.PI;
        }
        this.rotation = THREE.MathUtils.lerp(this.rotation, this.targetRotation, dt * 10);
        
        this.velocity.x = Math.sin(this.rotation + Math.PI) * this.speed;
        this.velocity.z = Math.cos(this.rotation + Math.PI) * this.speed;
        
        // Jump
        if (keys['Space'] && this.isGrounded) {
          this.jumpVelocity = 8;
          this.isGrounded = false;
          keys['Space'] = false;
        }
        
        // Apply gravity
        if (!this.isGrounded) {
          this.jumpVelocity -= 20 * dt;
        }
        
        this.position.x += this.velocity.x * dt;
        this.position.z += this.velocity.z * dt;
        this.position.y += this.jumpVelocity * dt;
        
        // Ground collision
        if (this.position.y <= 0) {
          this.position.y = 0;
          this.jumpVelocity = 0;
          this.isGrounded = true;
        }
        
        // Apply to group
        this.group.position.copy(this.position);
        this.group.rotation.y = this.rotation;
        
        // Animate
        this._animateCharacter(dt);
        
        // Spring bones
        if (this.spring.enabled) {
          this._updateSprings(dt);
        }
        
        // Update state UI
        let state = 'idle';
        if (!this.isGrounded) state = 'jump';
        else if (this.speed > 5) state = 'run';
        else if (this.speed > 0.5) state = 'walk';
        
        this.animState = state;
      }
      
      _animateCharacter(dt) {
        this.animTime += dt;
        const t = this.animTime;
        
        const speedFactor = this.speed / 3;
        const cycleSpeed = this.speed > 5 ? 3 : 2;
        
        if (this.speed > 0.5) {
          // Walk/run animation
          const legSwing = Math.sin(t * cycleSpeed * Math.PI) * 0.5 * speedFactor;
          const armSwing = Math.sin(t * cycleSpeed * Math.PI) * 0.4 * speedFactor;
          
          this.leftLeg.rotation.x = legSwing;
          this.rightLeg.rotation.x = -legSwing;
          
          this.leftArm.rotation.x = -armSwing;
          this.rightArm.rotation.x = armSwing;
          
          // Body bounce
          const bounce = Math.abs(Math.sin(t * cycleSpeed * Math.PI)) * 0.05 * speedFactor;
          this.body.position.y = 1.3 + bounce;
          this.head.position.y = 1.85 + bounce;
        } else {
          // Idle animation
          const idle = Math.sin(t * 1.5) * 0.03;
          this.body.position.y = 1.3 + idle;
          this.head.position.y = 1.85 + idle;
          
          this.leftArm.rotation.x = Math.sin(t * 0.8) * 0.1;
          this.rightArm.rotation.x = -Math.sin(t * 0.8) * 0.1;
          
          this.leftLeg.rotation.x = 0;
          this.rightLeg.rotation.x = 0;
        }
        
        // Jump animation
        if (!this.isGrounded) {
          this.leftLeg.rotation.x = -0.5;
          this.rightLeg.rotation.x = -0.5;
          this.leftArm.rotation.x = -0.8;
          this.rightArm.rotation.x = -0.8;
        }
      }
      
      _updateSprings(dt) {
        this.springTime += dt;
        
        this.hair.forEach((strand, i) => {
          // Spring physics
          const targetPos = strand.basePos.clone();
          targetPos.applyMatrix4(this.group.matrixWorld);
          
          // Add movement inertia
          const vel = this.velocity.clone().multiplyScalar(-0.3);
          const wind = new THREE.Vector3(
            Math.sin(this.springTime * 2 + i) * 0.1,
            0,
            Math.cos(this.springTime * 1.5 + i) * 0.08
          );
          
          const spring = targetPos.add(vel).add(wind);
          
          strand.springPos.lerp(spring, dt * 5);
          strand.mesh.position.copy(strand.springPos).sub(this.group.position);
        });
      }
      
      // Look-at target
      lookAt(target) {
        const localTarget = target.clone().sub(this.group.position);
        localTarget.y = 0;
        
        if (localTarget.length() > 0.1) {
          const angle = Math.atan2(localTarget.x, localTarget.z);
          this.targetRotation = angle;
        }
        
        // Head look
        const headTarget = target.clone();
        const headPos = new THREE.Vector3();
        this.head.getWorldPosition(headPos);
        
        const headToTarget = headTarget.sub(headPos).normalize();
        const headAngle = Math.atan2(headToTarget.x, headToTarget.z);
        
        this.head.rotation.y = THREE.MathUtils.clamp(
          headAngle - this.rotation - Math.PI,
          -Math.PI / 3,
          Math.PI / 3
        );
      }
    }
    
    // Character instance
    const character = new ProceduralCharacter(scene);
    
    // Input
    const keys = {};
    window.addEventListener('keydown', e => keys[e.code] = true);
    window.addEventListener('keyup', e => keys[e.code] = false);
    
    // Click look-at
    const raycaster = new THREE.Raycaster();
    const mouse = new THREE.Vector2();
    let lookAtTarget = null;
    
    canvas.addEventListener('click', e => {
      mouse.x = (e.clientX / window.innerWidth) * 2 - 1;
      mouse.y = -(e.clientY / window.innerHeight) * 2 + 1;
      
      raycaster.setFromCamera(mouse, camera);
      const hits = raycaster.intersectObject(ground);
      
      if (hits.length > 0) {
        lookAtTarget = hits[0].point.clone();
        lookAtTarget.y = 1.5;
        
        // Visual marker
        const marker = new THREE.Mesh(
          new THREE.SphereGeometry(0.1),
          new THREE.MeshBasicMaterial({ color: 0xff8800 })
        );
        marker.position.copy(hits[0].point);
        marker.position.y = 0.1;
        scene.add(marker);
        setTimeout(() => scene.remove(marker), 1000);
      }
    });
    
    // Resize
    window.addEventListener('resize', () => {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    });
    
    // Animation loop
    const clock = new THREE.Clock();
    function animate() {
      requestAnimationFrame(animate);
      
      const dt = Math.min(clock.getDelta(), 0.05);
      
      character.update(dt, keys);
      
      if (lookAtTarget) {
        character.lookAt(lookAtTarget);
      }
      
      // Camera follow
      const targetCamPos = new THREE.Vector3(
        character.position.x, 
        character.position.y + 3, 
        character.position.z + 5
      );
      camera.position.lerp(targetCamPos, dt * 2);
      controls.target.lerp(
        new THREE.Vector3(character.position.x, character.position.y + 1, character.position.z),
        dt * 4
      );
      
      // Update UI
      document.getElementById('speed').textContent = character.speed.toFixed(1);
      document.getElementById('state').textContent = character.animState;
      
      controls.update();
      renderer.render(scene, camera);
    }
    animate();
  </script>
</body>
</html>
```

---

## 7. Animation Retargeting

```javascript
class AnimationRetargeter {
  constructor(sourceRig, targetRig) {
    this.sourceRig = sourceRig;  // เช่น Mixamo rig
    this.targetRig = targetRig;  // rig ของเรา
    
    // Bone mapping: source name -> target name
    this.boneMapping = new Map();
    this.offsets = new Map();
  }
  
  // Setup auto mapping โดยดูจาก bone names
  autoMap() {
    const commonMappings = {
      'mixamorig:Hips': 'Hips',
      'mixamorig:Spine': 'Spine',
      'mixamorig:Spine1': 'Spine1',
      'mixamorig:Spine2': 'Spine2',
      'mixamorig:Neck': 'Neck',
      'mixamorig:Head': 'Head',
      'mixamorig:LeftUpLeg': 'LeftUpLeg',
      'mixamorig:LeftLeg': 'LeftLeg',
      'mixamorig:LeftFoot': 'LeftFoot',
      'mixamorig:RightUpLeg': 'RightUpLeg',
      'mixamorig:RightLeg': 'RightLeg',
      'mixamorig:RightFoot': 'RightFoot',
      'mixamorig:LeftArm': 'LeftArm',
      'mixamorig:LeftForeArm': 'LeftForeArm',
      'mixamorig:LeftHand': 'LeftHand',
      'mixamorig:RightArm': 'RightArm',
      'mixamorig:RightForeArm': 'RightForeArm',
      'mixamorig:RightHand': 'RightHand'
    };
    
    for (const [src, tgt] of Object.entries(commonMappings)) {
      this.boneMapping.set(src, tgt);
    }
  }
  
  // Retarget animation clip
  retargetClip(sourceClip, sourceBones, targetBones) {
    const tracks = [];
    
    for (const track of sourceClip.tracks) {
      // Parse bone name from track
      const parts = track.name.split('.');
      const boneName = parts[0];
      const property = parts[1];
      
      // Map to target bone
      const targetBoneName = this.boneMapping.get(boneName);
      if (!targetBoneName) continue;
      
      const targetBone = targetBones[targetBoneName];
      if (!targetBone) continue;
      
      // Create new track for target
      const newTrack = track.clone();
      newTrack.name = `${targetBoneName}.${property}`;
      
      // Apply offset if any
      if (property === 'quaternion' && this.offsets.has(boneName)) {
        const offset = this.offsets.get(boneName);
        // Apply rotation offset to each keyframe
        for (let i = 0; i < newTrack.values.length; i += 4) {
          const q = new THREE.Quaternion(
            newTrack.values[i],
            newTrack.values[i + 1],
            newTrack.values[i + 2],
            newTrack.values[i + 3]
          );
          q.multiply(offset);
          newTrack.values[i] = q.x;
          newTrack.values[i + 1] = q.y;
          newTrack.values[i + 2] = q.z;
          newTrack.values[i + 3] = q.w;
        }
      }
      
      tracks.push(newTrack);
    }
    
    return new THREE.AnimationClip(sourceClip.name, sourceClip.duration, tracks);
  }
}
```

---

## 8. สรุป Character Animation

### สิ่งที่เรียนรู้:
1. **Blend Tree**: การ blend ระหว่าง animations ตาม speed
2. **Foot IK**: ปรับตำแหน่งเท้าตาม terrain
3. **Look-At IK**: ให้ character มอง target
4. **Spring Bones**: Procedural animation สำหรับ cloth/hair
5. **Animation Retargeting**: การ apply animations จาก rig หนึ่งไปอีก rig

### Resources ที่แนะนำ:
- **Mixamo**: Free character animations จาก Adobe
- **ReadyPlayerMe**: Web-based character creation
- **VRM Format**: Humanoid character standard สำหรับ web

ในบทถัดไป (Part 90) เราจะเรียนรู้เกี่ยวกับ Scene Transitions & Loading
