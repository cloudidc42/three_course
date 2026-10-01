# Part 38: AnimationMixer & Clips ใน Three.js

## บทนำ

Animation System ใน Three.js ประกอบด้วยส่วนสำคัญ 3 ส่วน:

1. **AnimationClip** - เก็บข้อมูล keyframe animation (เหมือน "ไฟล์วิดีโอ")
2. **AnimationMixer** - เล่น animation บน object ที่ต้องการ (เหมือน "เครื่องเล่น")
3. **AnimationAction** - ควบคุมการเล่น clip แต่ละอัน (weight, speed, loop)

ระบบนี้ช่วยให้เราสามารถ:
- เล่น/หยุด/pause animations
- Blend หลาย animations พร้อมกัน
- Cross-fade ระหว่าง animations
- สร้าง complex animation state machines

## 1. AnimationClip

```javascript
import * as THREE from 'three';

// AnimationClip เก็บ tracks หลายๆ อัน
// แต่ละ track ควบคุม property หนึ่งตลอดเวลา

// ===== KeyframeTrack Types =====

// VectorKeyframeTrack: สำหรับ Vector3 (position, scale)
const positionTrack = new THREE.VectorKeyframeTrack(
  'mesh.position',     // propertyPath: ชื่อ object.property
  [0, 1, 2, 3],        // times (seconds)
  [
    0, 0, 0,           // position ที่ t=0
    0, 2, 0,           // position ที่ t=1
    3, 2, 0,           // position ที่ t=2
    3, 0, 0            // position ที่ t=3
  ]
);

// QuaternionKeyframeTrack: สำหรับ rotation (เป็น quaternion)
const rotationTrack = new THREE.QuaternionKeyframeTrack(
  'mesh.quaternion',
  [0, 1, 2],
  [
    0, 0, 0, 1,        // identity rotation ที่ t=0
    0, 0.707, 0, 0.707, // 90° around Y ที่ t=1
    0, 0, 0, 1         // back to identity ที่ t=2
  ]
);

// NumberKeyframeTrack: สำหรับ single number (opacity, intensity, morph)
const opacityTrack = new THREE.NumberKeyframeTrack(
  'material.opacity',
  [0, 0.5, 1],
  [1, 0, 1]
);

// ColorKeyframeTrack: สำหรับสี
const colorTrack = new THREE.ColorKeyframeTrack(
  'material.color',
  [0, 1, 2],
  [
    1, 0, 0,   // red ที่ t=0
    0, 1, 0,   // green ที่ t=1
    0, 0, 1    // blue ที่ t=2
  ]
);

// ===== สร้าง AnimationClip =====
const clip = new THREE.AnimationClip(
  'myAnimation',    // ชื่อ clip
  3,               // duration (-1 = auto detect จาก tracks)
  [positionTrack, rotationTrack, opacityTrack]
);

console.log(`Clip duration: ${clip.duration}s`);
console.log(`Track count: ${clip.tracks.length}`);
```

## 2. AnimationMixer

```javascript
// AnimationMixer ผูกกับ object ที่ต้องการ animate
const mixer = new THREE.AnimationMixer(mesh);

// สร้าง action จาก clip
const action = mixer.clipAction(clip);

// ควบคุม playback
action.play();
action.pause();
action.stop();
action.reset();

// ===== Playback properties =====
action.timeScale = 1;    // ความเร็ว (2 = เร็วกว่า 2x, -1 = play backward)
action.weight = 1;       // น้ำหนักของ animation (0-1)
action.loop = THREE.LoopRepeat;  // LoopRepeat | LoopOnce | LoopPingPong
action.repetitions = Infinity;   // จำนวนรอบ
action.clampWhenFinished = false; // หยุด pose สุดท้ายเมื่อจบ

// ===== Update ใน animation loop =====
const clock = new THREE.Clock();
function animate() {
  requestAnimationFrame(animate);
  
  const delta = clock.getDelta();
  mixer.update(delta);  // สำคัญมาก! ต้อง update ทุก frame
  
  renderer.render(scene, camera);
}
animate();
```

## 3. AnimationAction - Advanced Control

```javascript
// ===== Fade In/Out =====
function playWithFade(mixer, clip, fadeDuration = 0.5) {
  const action = mixer.clipAction(clip);
  action.reset();
  action.fadeIn(fadeDuration);
  action.play();
  return action;
}

function stopWithFade(action, fadeDuration = 0.5) {
  action.fadeOut(fadeDuration);
  // Remove after fade
  setTimeout(() => {
    action.stop();
  }, fadeDuration * 1000);
}

// ===== CrossFade ระหว่าง animations =====
function crossFadeTo(fromAction, toAction, duration = 0.5) {
  // เริ่ม animation ใหม่
  toAction.reset();
  toAction.play();
  
  // Cross fade
  fromAction.crossFadeTo(toAction, duration, true);
}

// ===== Blend หลาย animations =====
function blendAnimations(mixer, clips, weights) {
  const actions = clips.map(clip => {
    const action = mixer.clipAction(clip);
    action.play();
    return action;
  });
  
  // ตั้งค่า weights
  actions.forEach((action, i) => {
    action.setEffectiveWeight(weights[i]);
  });
  
  return actions;
}

// ===== Time Warping =====
function setPlaybackSpeed(action, speed) {
  action.setEffectiveTimeScale(speed);
}

// ===== จัดการ events =====
const mixer = new THREE.AnimationMixer(mesh);

// Event เมื่อ animation จบ
mixer.addEventListener('finished', (e) => {
  console.log('Animation finished:', e.action._clip.name);
  console.log('Direction:', e.direction); // 1 = forward, -1 = backward
});

// Event เมื่อ loop
mixer.addEventListener('loop', (e) => {
  console.log('Animation looped:', e.action._clip.name);
  console.log('Loop count:', e.loopDelta);
});
```

## 4. KeyframeTrack Interpolation

```javascript
// ===== Interpolation Modes =====
// INTERPOLATION.DISCRETE: ไม่มี interpolation (กระโดด)
// INTERPOLATION.LINEAR: linear interpolation
// INTERPOLATION.SMOOTH: cubic spline (ค่า default สำหรับ quaternion)

const smoothTrack = new THREE.VectorKeyframeTrack(
  'mesh.position',
  [0, 1, 2, 3],
  [0, 0, 0,  0, 2, 0,  3, 2, 0,  3, 0, 0],
  THREE.InterpolateSmooth // cubic interpolation
);

const discreteTrack = new THREE.NumberKeyframeTrack(
  'mesh.visible',
  [0, 1, 2],
  [1, 0, 1],
  THREE.InterpolateDiscrete // jump without interpolation
);

// ===== Subclip =====
// ตัด clip ให้สั้นลง
const shortClip = THREE.AnimationUtils.subclip(
  originalClip, // source clip
  'shortVersion', // ชื่อใหม่
  10, // start frame
  40  // end frame
);

// ===== Optimize =====
const optimizedClip = THREE.AnimationUtils.makeClipAdditive(clip);
```

## 5. Demo 1: GLTF Character Animation System

```html
<!DOCTYPE html>
<html>
<head>
  <title>GLTF Animation System</title>
  <style>
    body { margin: 0; background: #1a1a2e; font-family: Arial; }
    canvas { display: block; }
    #ui {
      position: fixed;
      bottom: 20px;
      left: 50%;
      transform: translateX(-50%);
      display: flex;
      gap: 10px;
      flex-wrap: wrap;
      justify-content: center;
    }
    .anim-btn {
      padding: 10px 20px;
      border: 2px solid #4488ff;
      background: rgba(0,0,50,0.8);
      color: #88aaff;
      cursor: pointer;
      border-radius: 6px;
      font-size: 14px;
      transition: all 0.2s;
    }
    .anim-btn:hover, .anim-btn.active {
      background: #4488ff;
      color: white;
    }
    #status {
      position: fixed;
      top: 20px;
      left: 50%;
      transform: translateX(-50%);
      color: #8af;
      font-size: 16px;
      text-shadow: 0 0 10px rgba(100,150,255,0.8);
    }
  </style>
</head>
<body>
<div id="status">Loading...</div>
<div id="ui">
  <button class="anim-btn active" data-anim="idle">Idle</button>
  <button class="anim-btn" data-anim="walk">Walk</button>
  <button class="anim-btn" data-anim="run">Run</button>
  <button class="anim-btn" data-anim="jump">Jump</button>
  <button class="anim-btn" data-anim="wave">Wave</button>
</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';
import { GLTFLoader } from 'https://unpkg.com/three@0.158.0/examples/jsm/loaders/GLTFLoader.js';

// ===== SETUP =====
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x1a1a2e);
scene.fog = new THREE.Fog(0x1a1a2e, 10, 30);

const camera = new THREE.PerspectiveCamera(50, innerWidth / innerHeight, 0.1, 100);
camera.position.set(0, 2, 6);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(devicePixelRatio);
renderer.shadowMap.enabled = true;
renderer.outputColorSpace = THREE.SRGBColorSpace;
document.body.appendChild(renderer.domElement);

const orbitControls = new OrbitControls(camera, renderer.domElement);
orbitControls.target.set(0, 1, 0);
orbitControls.enableDamping = true;

// Lighting
scene.add(new THREE.AmbientLight(0xffffff, 0.5));
const dirLight = new THREE.DirectionalLight(0xffffff, 2);
dirLight.position.set(5, 10, 5);
dirLight.castShadow = true;
scene.add(dirLight);
const fillLight = new THREE.DirectionalLight(0x8899ff, 0.5);
fillLight.position.set(-5, 3, -3);
scene.add(fillLight);

// Ground
const groundMesh = new THREE.Mesh(
  new THREE.PlaneGeometry(20, 20),
  new THREE.MeshStandardMaterial({ color: 0x222233, roughness: 0.8 })
);
groundMesh.rotation.x = -Math.PI / 2;
groundMesh.receiveShadow = true;
scene.add(groundMesh);

// Grid
const grid = new THREE.GridHelper(20, 20, 0x334455, 0x223344);
scene.add(grid);

// ===== ANIMATION CONTROLLER =====
class AnimationController {
  constructor() {
    this.mixer = null;
    this.actions = {};
    this.currentAction = null;
    this.fadeDuration = 0.3;
  }
  
  init(model, animations) {
    this.mixer = new THREE.AnimationMixer(model);
    
    animations.forEach(clip => {
      const action = this.mixer.clipAction(clip);
      this.actions[clip.name.toLowerCase()] = action;
      
      // ตั้งค่า default properties
      action.setEffectiveWeight(0);
      action.play();
    });
    
    console.log('Available animations:', Object.keys(this.actions));
  }
  
  playAnimation(name, fadeTime) {
    const duration = fadeTime !== undefined ? fadeTime : this.fadeDuration;
    const nextAction = this.actions[name.toLowerCase()];
    
    if (!nextAction) {
      console.warn(`Animation "${name}" not found`);
      return;
    }
    
    if (this.currentAction === nextAction) return;
    
    if (this.currentAction) {
      this.currentAction.fadeOut(duration);
    }
    
    nextAction.reset();
    nextAction.fadeIn(duration);
    this.currentAction = nextAction;
    
    document.getElementById('status').textContent = `Playing: ${name}`;
  }
  
  update(delta) {
    if (this.mixer) {
      this.mixer.update(delta);
    }
  }
}

const animController = new AnimationController();

// ===== LOAD GLTF =====
// ใช้ Mixamo character (ถ้าไม่มี ให้สร้าง procedural character)
const loader = new GLTFLoader();

// เนื่องจากอาจไม่มีไฟล์ จึง fallback ไปสร้าง procedural character
function createProceduralCharacter() {
  const group = new THREE.Group();
  
  const mat = new THREE.MeshStandardMaterial({ color: 0x4488cc, roughness: 0.6 });
  const darkMat = new THREE.MeshStandardMaterial({ color: 0x223355, roughness: 0.7 });
  
  // Body
  const body = new THREE.Mesh(new THREE.BoxGeometry(0.5, 0.7, 0.25), mat);
  body.position.y = 1.1;
  body.name = 'body';
  body.castShadow = true;
  group.add(body);
  
  // Head
  const head = new THREE.Mesh(new THREE.SphereGeometry(0.2, 12, 12), mat);
  head.position.y = 1.75;
  head.name = 'head';
  head.castShadow = true;
  group.add(head);
  
  // Arms
  const lArm = new THREE.Mesh(new THREE.BoxGeometry(0.12, 0.6, 0.12), darkMat);
  lArm.position.set(-0.36, 1.1, 0);
  lArm.name = 'leftArm';
  lArm.castShadow = true;
  group.add(lArm);
  
  const rArm = new THREE.Mesh(new THREE.BoxGeometry(0.12, 0.6, 0.12), darkMat);
  rArm.position.set(0.36, 1.1, 0);
  rArm.name = 'rightArm';
  rArm.castShadow = true;
  group.add(rArm);
  
  // Legs
  const lLeg = new THREE.Mesh(new THREE.BoxGeometry(0.15, 0.7, 0.15), darkMat);
  lLeg.position.set(-0.15, 0.45, 0);
  lLeg.name = 'leftLeg';
  lLeg.castShadow = true;
  group.add(lLeg);
  
  const rLeg = new THREE.Mesh(new THREE.BoxGeometry(0.15, 0.7, 0.15), darkMat);
  rLeg.position.set(0.15, 0.45, 0);
  rLeg.name = 'rightLeg';
  rLeg.castShadow = true;
  group.add(rLeg);
  
  return group;
}

// สร้าง animations สำหรับ procedural character
function createProceduralAnimations() {
  const DEG = Math.PI / 180;
  
  // ===== IDLE =====
  const idleBreath = new THREE.VectorKeyframeTrack(
    'body.scale',
    [0, 1, 2],
    [1, 1, 1,  1, 1.02, 1,  1, 1, 1]
  );
  const idleHeadNod = new THREE.QuaternionKeyframeTrack(
    'head.quaternion',
    [0, 1.5, 3],
    [
      ...new THREE.Quaternion().toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), -5*DEG).toArray(),
      ...new THREE.Quaternion().toArray()
    ]
  );
  const idleClip = new THREE.AnimationClip('idle', 3, [idleBreath, idleHeadNod]);
  
  // ===== WALK =====
  // Left leg swing
  const walkLLeg = new THREE.QuaternionKeyframeTrack(
    'leftLeg.quaternion',
    [0, 0.25, 0.5, 0.75, 1],
    [
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), 30*DEG).toArray(),
      ...new THREE.Quaternion().toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), -30*DEG).toArray(),
      ...new THREE.Quaternion().toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), 30*DEG).toArray(),
    ]
  );
  
  // Right leg swing (opposite phase)
  const walkRLeg = new THREE.QuaternionKeyframeTrack(
    'rightLeg.quaternion',
    [0, 0.25, 0.5, 0.75, 1],
    [
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), -30*DEG).toArray(),
      ...new THREE.Quaternion().toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), 30*DEG).toArray(),
      ...new THREE.Quaternion().toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), -30*DEG).toArray(),
    ]
  );
  
  // Arms swing (opposite to legs)
  const walkLArm = new THREE.QuaternionKeyframeTrack(
    'leftArm.quaternion',
    [0, 0.25, 0.5, 0.75, 1],
    [
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), -25*DEG).toArray(),
      ...new THREE.Quaternion().toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), 25*DEG).toArray(),
      ...new THREE.Quaternion().toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), -25*DEG).toArray(),
    ]
  );
  
  const walkRArm = new THREE.QuaternionKeyframeTrack(
    'rightArm.quaternion',
    [0, 0.25, 0.5, 0.75, 1],
    [
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), 25*DEG).toArray(),
      ...new THREE.Quaternion().toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), -25*DEG).toArray(),
      ...new THREE.Quaternion().toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), 25*DEG).toArray(),
    ]
  );
  
  // Body bob
  const walkBodyPos = new THREE.VectorKeyframeTrack(
    'body.position',
    [0, 0.25, 0.5, 0.75, 1],
    [
      0, 1.1, 0,
      0, 1.13, 0,
      0, 1.1, 0,
      0, 1.13, 0,
      0, 1.1, 0
    ]
  );
  
  const walkClip = new THREE.AnimationClip('walk', 1, [walkLLeg, walkRLeg, walkLArm, walkRArm, walkBodyPos]);
  
  // ===== RUN (faster, more exaggerated) =====
  const runLLeg = new THREE.QuaternionKeyframeTrack(
    'leftLeg.quaternion',
    [0, 0.15, 0.3, 0.45, 0.6],
    [
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), 50*DEG).toArray(),
      ...new THREE.Quaternion().toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), -50*DEG).toArray(),
      ...new THREE.Quaternion().toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), 50*DEG).toArray(),
    ]
  );
  
  const runRLeg = new THREE.QuaternionKeyframeTrack(
    'rightLeg.quaternion',
    [0, 0.15, 0.3, 0.45, 0.6],
    [
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), -50*DEG).toArray(),
      ...new THREE.Quaternion().toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), 50*DEG).toArray(),
      ...new THREE.Quaternion().toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), -50*DEG).toArray(),
    ]
  );
  
  const runBodyTilt = new THREE.QuaternionKeyframeTrack(
    'body.quaternion',
    [0, 0.6],
    [
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), -15*DEG).toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), -15*DEG).toArray(),
    ]
  );
  
  const runClip = new THREE.AnimationClip('run', 0.6, [runLLeg, runRLeg, runBodyTilt]);
  
  // ===== JUMP =====
  const jumpBodyPos = new THREE.VectorKeyframeTrack(
    'body.position',
    [0, 0.3, 0.6, 1.0, 1.2],
    [
      0, 1.1, 0,   // crouch
      0, 0.9, 0,   // full crouch
      0, 2.2, 0,   // peak
      0, 1.1, 0,   // land
      0, 1.1, 0    // recover
    ]
  );
  
  const jumpLArm = new THREE.QuaternionKeyframeTrack(
    'leftArm.quaternion',
    [0, 0.3, 0.5, 0.8, 1.2],
    [
      ...new THREE.Quaternion().toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), -80*DEG).toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), -160*DEG).toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), -30*DEG).toArray(),
      ...new THREE.Quaternion().toArray(),
    ]
  );
  
  const jumpRArm = new THREE.QuaternionKeyframeTrack(
    'rightArm.quaternion',
    [0, 0.3, 0.5, 0.8, 1.2],
    [
      ...new THREE.Quaternion().toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), -80*DEG).toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), -160*DEG).toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), -30*DEG).toArray(),
      ...new THREE.Quaternion().toArray(),
    ]
  );
  
  const jumpClip = new THREE.AnimationClip('jump', 1.2, [jumpBodyPos, jumpLArm, jumpRArm]);
  
  // ===== WAVE =====
  const waveLArm = new THREE.QuaternionKeyframeTrack(
    'leftArm.quaternion',
    [0, 0.3, 0.6, 0.9, 1.2, 1.5, 1.8],
    [
      ...new THREE.Quaternion().toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(0,0,1), 90*DEG).toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(0,0,1), 110*DEG).toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(0,0,1), 90*DEG).toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(0,0,1), 110*DEG).toArray(),
      ...new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(0,0,1), 90*DEG).toArray(),
      ...new THREE.Quaternion().toArray(),
    ]
  );
  
  const waveClip = new THREE.AnimationClip('wave', 1.8, [waveLArm]);
  
  return [idleClip, walkClip, runClip, jumpClip, waveClip];
}

// ===== CREATE AND SETUP CHARACTER =====
const character = createProceduralCharacter();
scene.add(character);

const animations = createProceduralAnimations();

// สร้าง mixer บน parent object
const mixer = new THREE.AnimationMixer(character);
const actions = {};

animations.forEach(clip => {
  const action = mixer.clipAction(clip, character);
  
  // ต้องหา nested object
  // mixer จะ find objects ด้วย name matching
  clip.tracks.forEach(track => {
    const [objectName] = track.name.split('.');
    const target = character.getObjectByName(objectName);
    if (target) {
      const localAction = mixer.clipAction(clip, target.parent);
    }
  });
  
  actions[clip.name] = action;
  action.loop = clip.name === 'jump' ? THREE.LoopOnce : THREE.LoopRepeat;
  action.clampWhenFinished = clip.name === 'jump';
});

// เล่น idle ตั้งแต่ต้น
if (actions['idle']) {
  actions['idle'].play();
}

document.getElementById('status').textContent = 'Playing: idle';

let currentActionName = 'idle';

// ===== BUTTON CONTROLS =====
document.querySelectorAll('.anim-btn').forEach(btn => {
  btn.addEventListener('click', () => {
    const animName = btn.dataset.anim;
    
    // Update button states
    document.querySelectorAll('.anim-btn').forEach(b => b.classList.remove('active'));
    btn.classList.add('active');
    
    // Transition
    const current = actions[currentActionName];
    const next = actions[animName];
    
    if (current && next && current !== next) {
      current.fadeOut(0.3);
      
      next.reset();
      next.fadeIn(0.3);
      next.play();
      
      currentActionName = animName;
      document.getElementById('status').textContent = `Playing: ${animName}`;
    }
  });
});

// ===== ANIMATION LOOP =====
const clock = new THREE.Clock();

function animate() {
  requestAnimationFrame(animate);
  
  const delta = clock.getDelta();
  mixer.update(delta);
  
  orbitControls.update();
  renderer.render(scene, camera);
}

animate();

window.addEventListener('resize', () => {
  camera.aspect = innerWidth / innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(innerWidth, innerHeight);
});
</script>
</body>
</html>
```

## 6. Walk/Run/Jump Blend Tree

```javascript
// ===== LOCOMOTION BLEND TREE =====
class LocomotionBlendTree {
  constructor(mixer) {
    this.mixer = mixer;
    this.actions = {};
    this.speed = 0; // 0 = idle, 1 = walk, 2 = run
    this.isJumping = false;
  }
  
  addAction(name, clip, options = {}) {
    const action = this.mixer.clipAction(clip);
    action.play();
    action.setEffectiveWeight(0);
    
    if (options.loop === false) {
      action.loop = THREE.LoopOnce;
      action.clampWhenFinished = true;
    }
    
    this.actions[name] = action;
    return this;
  }
  
  // อัพเดต blend weights ตาม speed
  update(speed) {
    this.speed = speed;
    
    if (this.isJumping) return;
    
    // Blend ระหว่าง idle, walk, run
    let idleWeight = 0;
    let walkWeight = 0;
    let runWeight = 0;
    
    if (speed <= 0) {
      idleWeight = 1;
    } else if (speed <= 1) {
      // Blend idle -> walk
      walkWeight = speed;
      idleWeight = 1 - speed;
    } else if (speed <= 2) {
      // Blend walk -> run
      const t = speed - 1;
      runWeight = t;
      walkWeight = 1 - t;
    } else {
      runWeight = 1;
    }
    
    this.setWeight('idle', idleWeight);
    this.setWeight('walk', walkWeight);
    this.setWeight('run', runWeight);
  }
  
  setWeight(name, weight) {
    if (this.actions[name]) {
      this.actions[name].setEffectiveWeight(weight);
    }
  }
  
  jump() {
    if (this.isJumping) return;
    
    this.isJumping = true;
    
    // Reduce all locomotion weights
    ['idle', 'walk', 'run'].forEach(name => this.setWeight(name, 0.3));
    
    const jumpAction = this.actions['jump'];
    if (jumpAction) {
      jumpAction.reset();
      jumpAction.setEffectiveWeight(1);
      
      // เมื่อ jump จบ กลับไป locomotion
      const onFinished = (e) => {
        if (e.action === jumpAction) {
          this.isJumping = false;
          jumpAction.setEffectiveWeight(0);
          this.update(this.speed); // Restore locomotion weights
          this.mixer.removeEventListener('finished', onFinished);
        }
      };
      
      this.mixer.addEventListener('finished', onFinished);
    }
  }
  
  // Additive animation (overlay บน top ของ base animation)
  playAdditive(name, weight = 1) {
    const action = this.actions[name];
    if (!action) return;
    
    // Additive blending
    action.blendMode = THREE.AdditiveAnimationBlendMode;
    action.reset();
    action.setEffectiveWeight(weight);
    action.play();
  }
}
```

## 7. Custom Keyframe Animation

```javascript
// ===== สร้าง complex animation จาก scratch =====
function createComplexAnimation(mesh) {
  const DEG = Math.PI / 180;
  const tracks = [];
  
  // ===== Position track: เดินเป็นวงกลม =====
  const circlePoints = [];
  const circleTime = [];
  const radius = 3;
  const steps = 20;
  
  for (let i = 0; i <= steps; i++) {
    const t = i / steps;
    const angle = t * Math.PI * 2;
    circleTime.push(t * 4); // 4 วินาที
    circlePoints.push(
      Math.cos(angle) * radius,
      0,
      Math.sin(angle) * radius
    );
  }
  
  const posTrack = new THREE.VectorKeyframeTrack(
    '.position',
    circleTime,
    circlePoints
  );
  tracks.push(posTrack);
  
  // ===== Rotation track: หันตามทิศทางการเดิน =====
  const rotTime = [];
  const rotValues = [];
  
  for (let i = 0; i <= steps; i++) {
    const t = i / steps;
    const angle = t * Math.PI * 2;
    
    rotTime.push(t * 4);
    
    // หันหน้าไปทิศทางการเดิน (tangent ของวงกลม)
    const q = new THREE.Quaternion();
    q.setFromAxisAngle(new THREE.Vector3(0, 1, 0), -angle + Math.PI / 2);
    rotValues.push(...q.toArray());
  }
  
  const rotTrack = new THREE.QuaternionKeyframeTrack(
    '.quaternion',
    rotTime,
    rotValues
  );
  tracks.push(rotTrack);
  
  // ===== Scale track: pulsing effect =====
  const scaleTrack = new THREE.VectorKeyframeTrack(
    '.scale',
    [0, 0.5, 1, 1.5, 2, 2.5, 3, 3.5, 4],
    [
      1, 1, 1,
      1.1, 1.3, 1.1,
      1, 1, 1,
      0.9, 0.7, 0.9,
      1, 1, 1,
      1.1, 1.3, 1.1,
      1, 1, 1,
      0.9, 0.7, 0.9,
      1, 1, 1,
    ]
  );
  tracks.push(scaleTrack);
  
  // ===== Material opacity track =====
  if (mesh.material) {
    mesh.material.transparent = true;
    
    const opacityTrack = new THREE.NumberKeyframeTrack(
      'material.opacity',
      [0, 1, 2, 3, 4],
      [1, 0.5, 1, 0.5, 1]
    );
    tracks.push(opacityTrack);
  }
  
  return new THREE.AnimationClip('complex', 4, tracks);
}

// ===== ANIMATION LAYER SYSTEM =====
class AnimationLayer {
  constructor(mixer, weight = 1) {
    this.mixer = mixer;
    this.weight = weight;
    this.actions = new Map();
    this.currentAction = null;
  }
  
  addClip(clip) {
    const action = this.mixer.clipAction(clip);
    action.setEffectiveWeight(0);
    action.play();
    this.actions.set(clip.name, action);
    return this;
  }
  
  play(clipName, transitionTime = 0.3) {
    const newAction = this.actions.get(clipName);
    if (!newAction) return;
    
    if (this.currentAction) {
      this.currentAction.fadeOut(transitionTime);
    }
    
    newAction.reset();
    newAction.fadeIn(transitionTime);
    newAction.setEffectiveWeight(this.weight);
    this.currentAction = newAction;
  }
  
  setLayerWeight(weight) {
    this.weight = weight;
    if (this.currentAction) {
      this.currentAction.setEffectiveWeight(weight);
    }
  }
}
```

## 8. GLTF Animations - Full Implementation

```javascript
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

class CharacterAnimationSystem {
  constructor(scene) {
    this.scene = scene;
    this.model = null;
    this.mixer = null;
    this.actions = {};
    this.currentAction = null;
    this.clock = new THREE.Clock();
  }
  
  async load(url) {
    return new Promise((resolve, reject) => {
      const loader = new GLTFLoader();
      
      loader.load(url, 
        (gltf) => {
          this.model = gltf.scene;
          this.scene.add(this.model);
          
          // Setup mixer
          this.mixer = new THREE.AnimationMixer(this.model);
          
          // Setup all animations
          gltf.animations.forEach(clip => {
            this.addAnimation(clip);
          });
          
          console.log('Loaded animations:', Object.keys(this.actions));
          resolve(this);
        },
        (progress) => {
          const pct = (progress.loaded / progress.total * 100).toFixed(1);
          console.log(`Loading: ${pct}%`);
        },
        reject
      );
    });
  }
  
  addAnimation(clip) {
    const action = this.mixer.clipAction(clip);
    
    // ตั้งค่า loop ตามชื่อ
    if (clip.name.toLowerCase().includes('jump') ||
        clip.name.toLowerCase().includes('hit') ||
        clip.name.toLowerCase().includes('die')) {
      action.loop = THREE.LoopOnce;
      action.clampWhenFinished = true;
    }
    
    this.actions[clip.name] = action;
  }
  
  transition(toName, duration = 0.3) {
    const toAction = this.actions[toName];
    if (!toAction) {
      console.warn(`Animation "${toName}" not found`);
      return;
    }
    
    if (this.currentAction === toAction) return;
    
    // Stop current with fadeOut
    if (this.currentAction) {
      this.currentAction.fadeOut(duration);
    }
    
    // Start new with fadeIn
    toAction.reset();
    toAction.setEffectiveWeight(1);
    toAction.fadeIn(duration);
    toAction.play();
    
    this.currentAction = toAction;
    return toAction;
  }
  
  // Blend ระหว่าง 2 animations ด้วย weight
  blend(nameA, nameB, blendFactor) {
    const actionA = this.actions[nameA];
    const actionB = this.actions[nameB];
    
    if (!actionA || !actionB) return;
    
    actionA.play();
    actionB.play();
    
    actionA.setEffectiveWeight(1 - blendFactor);
    actionB.setEffectiveWeight(blendFactor);
  }
  
  // Sync time scales สำหรับ seamless transition
  syncTimescale(fromName, toName) {
    const fromAction = this.actions[fromName];
    const toAction = this.actions[toName];
    
    if (!fromAction || !toAction) return;
    
    // ทำให้ duration ของทั้งสองตรงกัน
    const ratio = fromAction._clip.duration / toAction._clip.duration;
    toAction.setEffectiveTimeScale(ratio);
  }
  
  update() {
    const delta = this.clock.getDelta();
    if (this.mixer) {
      this.mixer.update(delta);
    }
  }
  
  // Pause/Resume
  pause() {
    if (this.mixer) this.mixer.timeScale = 0;
  }
  
  resume() {
    if (this.mixer) this.mixer.timeScale = 1;
  }
  
  // Get normalized time (0-1)
  getNormalizedTime(actionName) {
    const action = this.actions[actionName];
    if (!action) return 0;
    
    const clip = action.getClip();
    return (action.time % clip.duration) / clip.duration;
  }
}
```

## สรุป

Animation System ใน Three.js ประกอบด้วย:

1. **AnimationClip** - เก็บ KeyframeTracks สำหรับ properties ต่างๆ
2. **AnimationMixer** - จัดการ playback และ blending
3. **AnimationAction** - ควบคุม clip แต่ละอัน (weight, time, speed)

### Best Practices
- ใช้ `fadeIn/fadeOut` สำหรับ smooth transitions
- Normalize weights ให้รวมกันเป็น 1 เมื่อ blending
- Cache `mixer.clipAction()` เพราะมัน expensive
- Update mixer ทุก frame ด้วย `mixer.update(delta)`

---

## แบบฝึกหัด

1. **Easy**: สร้าง bouncing ball animation ด้วย VectorKeyframeTrack และ NumberKeyframeTrack
2. **Medium**: สร้าง Locomotion system ที่ blend ระหว่าง idle/walk/run ตาม keyboard input
3. **Hard**: โหลด GLTF model พร้อม animations และสร้าง Animation State Machine แบบ full
4. **Challenge**: สร้าง Blend Tree system ที่รองรับ 2D blending (เช่น direction + speed) พร้อม smooth transition
