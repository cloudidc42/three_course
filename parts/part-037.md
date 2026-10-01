# Part 37: Skinning & Bones (Skeletal Animation) ใน Three.js

## บทนำ

Skeletal Animation หรือที่เรียกว่า Skinning เป็นเทคนิคการทำ Animation ที่ใช้ "โครงกระดูก" (Skeleton) เป็นตัวขับเคลื่อนการเคลื่อนไหวของ Mesh โดย:

- **Skeleton** ประกอบด้วย Bone หลายๆ ชิ้นที่เชื่อมต่อกันแบบ hierarchy
- **Skinning** คือกระบวนการผูก vertices ของ Mesh เข้ากับ Bones
- **Bone Weights** กำหนดว่า vertex แต่ละจุดได้รับอิทธิพลจาก Bone ไหนมากแค่ไหน

เทคนิคนี้ถูกใช้ใน:
- ตัวละครในเกม (Character Animation)
- Robot และ Mechanical objects
- Creature Animation
- Procedural motion

## 1. ทำความเข้าใจ Skeleton และ Bone

```javascript
import * as THREE from 'three';

// ===== สร้าง Bone =====
const bone = new THREE.Bone();
bone.name = 'Spine';
bone.position.set(0, 0, 0);
bone.rotation.x = Math.PI / 4; // หมุน 45 องศา

// ===== สร้าง Skeleton =====
// Skeleton ต้องสร้างจาก array ของ bones ที่เชื่อมต่อกัน
const root = new THREE.Bone();
root.name = 'Root';
root.position.set(0, -1, 0);

const spine = new THREE.Bone();
spine.name = 'Spine';
spine.position.set(0, 1, 0); // offset จาก parent
root.add(spine);             // เชื่อม spine กับ root

const chest = new THREE.Bone();
chest.name = 'Chest';
chest.position.set(0, 1, 0);
spine.add(chest);

const head = new THREE.Bone();
head.name = 'Head';
head.position.set(0, 0.8, 0);
chest.add(head);

// สร้าง Skeleton จาก bones ทั้งหมด (ต้องอยู่ใน flat array)
const bones = [root, spine, chest, head];
const skeleton = new THREE.Skeleton(bones);
```

## 2. skinIndex และ skinWeight Attributes

Skinning ทำงานโดย assign แต่ละ vertex ให้มี weight สำหรับ 4 bones

```javascript
// skinIndex: บอกว่า vertex นี้ได้รับอิทธิพลจาก bone index ไหนบ้าง (สูงสุด 4)
// skinWeight: บอกน้ำหนักของแต่ละ bone (รวมกันควรได้ 1.0)

const geometry = new THREE.BufferGeometry();
const vertexCount = 100;

// Position data
const positions = new Float32Array(vertexCount * 3);
// ... fill positions ...

// Skin indices: 4 bone indices per vertex
const skinIndices = new Float32Array(vertexCount * 4);
// Skin weights: 4 weights per vertex
const skinWeights = new Float32Array(vertexCount * 4);

for (let i = 0; i < vertexCount; i++) {
  const y = positions[i * 3 + 1]; // y position ของ vertex
  
  if (y < 0.5) {
    // อยู่ใน lower half - ได้รับอิทธิพลจาก bone 0 (root)
    skinIndices[i * 4 + 0] = 0; // bone 0
    skinIndices[i * 4 + 1] = 1; // bone 1
    skinIndices[i * 4 + 2] = 0; // padding
    skinIndices[i * 4 + 3] = 0; // padding
    
    skinWeights[i * 4 + 0] = 0.8; // 80% bone 0
    skinWeights[i * 4 + 1] = 0.2; // 20% bone 1
    skinWeights[i * 4 + 2] = 0.0;
    skinWeights[i * 4 + 3] = 0.0;
  } else {
    // อยู่ใน upper half - bone 1 มีอิทธิพลมากกว่า
    skinIndices[i * 4 + 0] = 1;
    skinIndices[i * 4 + 1] = 2;
    skinIndices[i * 4 + 2] = 0;
    skinIndices[i * 4 + 3] = 0;
    
    skinWeights[i * 4 + 0] = 0.7;
    skinWeights[i * 4 + 1] = 0.3;
    skinWeights[i * 4 + 2] = 0.0;
    skinWeights[i * 4 + 3] = 0.0;
  }
}

geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
geometry.setAttribute('skinIndex', new THREE.Uint16BufferAttribute(skinIndices, 4));
geometry.setAttribute('skinWeight', new THREE.BufferAttribute(skinWeights, 4));
```

## 3. SkinnedMesh

```javascript
// SkinnedMesh เป็น Mesh พิเศษที่รองรับ skeletal animation
const material = new THREE.MeshStandardMaterial({
  color: 0x44aaff,
  skinning: true // ต้องเปิด skinning ใน material
});

const skinnedMesh = new THREE.SkinnedMesh(geometry, material);

// Bind skeleton กับ mesh
skinnedMesh.add(skeleton.bones[0]); // เพิ่ม root bone เป็น child
skinnedMesh.bind(skeleton);

// ตรวจสอบ binding matrix
console.log(skeleton.boneInverses); // inverse bind matrices

scene.add(skinnedMesh);
```

## 4. SkeletonHelper

```javascript
// SkeletonHelper แสดง skeleton เป็น visual guide
const skeletonHelper = new THREE.SkeletonHelper(skinnedMesh);
scene.add(skeletonHelper);

// ซ่อน/แสดง
skeletonHelper.visible = true;
```

## 5. Demo 1: Custom Skeleton - Human Figure

```html
<!DOCTYPE html>
<html>
<head>
  <title>Custom Skeleton Demo</title>
  <style>
    body { margin: 0; background: #0a0a1a; }
    canvas { display: block; }
    #controls {
      position: fixed;
      top: 20px;
      right: 20px;
      background: rgba(0,0,0,0.8);
      color: #adf;
      padding: 15px;
      border-radius: 8px;
      font-family: monospace;
      font-size: 13px;
      min-width: 200px;
    }
    .bone-control { margin: 6px 0; }
    label { display: block; color: #7af; margin-bottom: 2px; }
    input[type="range"] { width: 150px; }
  </style>
</head>
<body>
<div id="controls">
  <div style="color:#fff;font-size:15px;margin-bottom:10px">Bone Controls</div>
  <div class="bone-control">
    <label>Spine Bend X</label>
    <input type="range" id="spineX" min="-60" max="60" value="0">
  </div>
  <div class="bone-control">
    <label>Spine Twist Y</label>
    <input type="range" id="spineY" min="-90" max="90" value="0">
  </div>
  <div class="bone-control">
    <label>Head Nod X</label>
    <input type="range" id="headX" min="-45" max="45" value="0">
  </div>
  <div class="bone-control">
    <label>Head Turn Y</label>
    <input type="range" id="headY" min="-80" max="80" value="0">
  </div>
  <div class="bone-control">
    <label>Left Arm X</label>
    <input type="range" id="lArmX" min="-150" max="30" value="0">
  </div>
  <div class="bone-control">
    <label>Right Arm X</label>
    <input type="range" id="rArmX" min="-150" max="30" value="0">
  </div>
  <div class="bone-control">
    <label>Left Elbow</label>
    <input type="range" id="lElbow" min="0" max="140" value="0">
  </div>
  <div class="bone-control">
    <label>Right Elbow</label>
    <input type="range" id="rElbow" min="0" max="140" value="0">
  </div>
  <div class="bone-control">
    <label>Left Hip X</label>
    <input type="range" id="lHipX" min="-90" max="90" value="0">
  </div>
  <div class="bone-control">
    <label>Right Hip X</label>
    <input type="range" id="rHipX" min="-90" max="90" value="0">
  </div>
  <button id="resetBtn" style="margin-top:8px;padding:6px 12px;cursor:pointer">Reset Pose</button>
  <button id="walkBtn" style="margin-top:4px;padding:6px 12px;cursor:pointer">Walk Anim</button>
</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';

// ===== SETUP =====
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x0a0a1a);

const camera = new THREE.PerspectiveCamera(55, innerWidth / innerHeight, 0.1, 100);
camera.position.set(0, 3, 8);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(devicePixelRatio);
renderer.shadowMap.enabled = true;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.target.set(0, 2, 0);
controls.enableDamping = true;
controls.update();

// ===== LIGHTING =====
scene.add(new THREE.AmbientLight(0x334466, 1.5));
const dirLight = new THREE.DirectionalLight(0xffffff, 2);
dirLight.position.set(5, 10, 5);
dirLight.castShadow = true;
scene.add(dirLight);
const fillLight = new THREE.DirectionalLight(0x6688ff, 0.5);
fillLight.position.set(-5, 3, -3);
scene.add(fillLight);

// ===== GROUND =====
const groundGeo = new THREE.PlaneGeometry(20, 20);
const groundMat = new THREE.MeshStandardMaterial({ color: 0x222233 });
const ground = new THREE.Mesh(groundGeo, groundMat);
ground.rotation.x = -Math.PI / 2;
ground.receiveShadow = true;
scene.add(ground);

// ===== SKELETON BUILDER =====
class HumanSkeleton {
  constructor() {
    this.bones = {};
    this.boneArray = [];
    this.build();
  }
  
  addBone(name, parent, position) {
    const bone = new THREE.Bone();
    bone.name = name;
    bone.position.copy(position);
    
    if (parent) {
      this.bones[parent].add(bone);
    }
    
    this.bones[name] = bone;
    this.boneArray.push(bone);
    return bone;
  }
  
  build() {
    // Root - ที่ pelvis
    this.addBone('root', null, new THREE.Vector3(0, 0, 0));
    
    // Spine
    this.addBone('spine', 'root', new THREE.Vector3(0, 0.5, 0));
    this.addBone('chest', 'spine', new THREE.Vector3(0, 0.7, 0));
    this.addBone('neck', 'chest', new THREE.Vector3(0, 0.5, 0));
    this.addBone('head', 'neck', new THREE.Vector3(0, 0.3, 0));
    
    // Left Arm
    this.addBone('leftShoulder', 'chest', new THREE.Vector3(-0.5, 0.3, 0));
    this.addBone('leftUpperArm', 'leftShoulder', new THREE.Vector3(-0.5, 0, 0));
    this.addBone('leftForeArm', 'leftUpperArm', new THREE.Vector3(-0.5, 0, 0));
    this.addBone('leftHand', 'leftForeArm', new THREE.Vector3(-0.4, 0, 0));
    
    // Right Arm
    this.addBone('rightShoulder', 'chest', new THREE.Vector3(0.5, 0.3, 0));
    this.addBone('rightUpperArm', 'rightShoulder', new THREE.Vector3(0.5, 0, 0));
    this.addBone('rightForeArm', 'rightUpperArm', new THREE.Vector3(0.5, 0, 0));
    this.addBone('rightHand', 'rightForeArm', new THREE.Vector3(0.4, 0, 0));
    
    // Left Leg
    this.addBone('leftHip', 'root', new THREE.Vector3(-0.25, -0.1, 0));
    this.addBone('leftThigh', 'leftHip', new THREE.Vector3(0, -0.7, 0));
    this.addBone('leftShin', 'leftThigh', new THREE.Vector3(0, -0.7, 0));
    this.addBone('leftFoot', 'leftShin', new THREE.Vector3(0, -0.5, 0));
    
    // Right Leg
    this.addBone('rightHip', 'root', new THREE.Vector3(0.25, -0.1, 0));
    this.addBone('rightThigh', 'rightHip', new THREE.Vector3(0, -0.7, 0));
    this.addBone('rightShin', 'rightThigh', new THREE.Vector3(0, -0.7, 0));
    this.addBone('rightFoot', 'rightShin', new THREE.Vector3(0, -0.5, 0));
  }
  
  getSkeleton() {
    return new THREE.Skeleton(this.boneArray);
  }
}

// ===== SKINNED HUMAN MESH =====
function createHumanMesh(humanSkeleton) {
  // สร้าง cylindrical body parts
  const meshGroup = new THREE.Group();
  meshGroup.position.y = 1.8; // ยกขึ้นจากพื้น
  
  const skinMaterial = new THREE.MeshStandardMaterial({
    color: 0x44aadd,
    roughness: 0.6,
    skinning: true
  });
  
  const darkMaterial = new THREE.MeshStandardMaterial({
    color: 0x224466,
    roughness: 0.7,
    skinning: true
  });
  
  // ฟังก์ชันสร้าง body part mesh อย่างง่าย
  function createBodyPart(radiusTop, radiusBottom, height, segments = 8) {
    return new THREE.CylinderGeometry(radiusTop, radiusBottom, height, segments);
  }
  
  const skeleton = humanSkeleton.getSkeleton();
  const bones = humanSkeleton.bones;
  
  // Body parts
  const bodyParts = [
    // Torso
    { name: 'torso', geo: new THREE.BoxGeometry(0.6, 0.8, 0.3), bone: 'spine', pos: [0, 0.9, 0] },
    { name: 'pelvis', geo: new THREE.BoxGeometry(0.55, 0.4, 0.28), bone: 'root', pos: [0, 0.2, 0] },
    // Head
    { name: 'head', geo: new THREE.SphereGeometry(0.22, 8, 8), bone: 'head', pos: [0, 4.1, 0] },
    // Arms
    { name: 'lUpperArm', geo: createBodyPart(0.08, 0.07, 0.5), bone: 'leftUpperArm', pos: [-0.85, 1.45, 0] },
    { name: 'lForeArm', geo: createBodyPart(0.07, 0.06, 0.45), bone: 'leftForeArm', pos: [-1.35, 1.45, 0] },
    { name: 'rUpperArm', geo: createBodyPart(0.08, 0.07, 0.5), bone: 'rightUpperArm', pos: [0.85, 1.45, 0] },
    { name: 'rForeArm', geo: createBodyPart(0.07, 0.06, 0.45), bone: 'rightForeArm', pos: [1.35, 1.45, 0] },
    // Legs
    { name: 'lThigh', geo: createBodyPart(0.1, 0.09, 0.65), bone: 'leftThigh', pos: [-0.22, -0.35, 0] },
    { name: 'lShin', geo: createBodyPart(0.09, 0.07, 0.6), bone: 'leftShin', pos: [-0.22, -0.95, 0] },
    { name: 'rThigh', geo: createBodyPart(0.1, 0.09, 0.65), bone: 'rightThigh', pos: [0.22, -0.35, 0] },
    { name: 'rShin', geo: createBodyPart(0.09, 0.07, 0.6), bone: 'rightShin', pos: [0.22, -0.95, 0] },
  ];
  
  // สร้าง simple non-skinned meshes (เพื่อแสดงให้เห็นก่อน)
  bodyParts.forEach(part => {
    const mat = part.name === 'head' ? skinMaterial.clone() : darkMaterial.clone();
    mat.color.setHex(part.name.includes('Arm') || part.name.includes('in') ? 0x2255aa : 0x3366bb);
    
    const mesh = new THREE.Mesh(part.geo, mat);
    mesh.position.set(...part.pos);
    mesh.castShadow = true;
    
    // Align cylinders
    if (part.geo.type === 'CylinderGeometry') {
      if (part.name.includes('Arm') || part.name.includes('Fore')) {
        mesh.rotation.z = Math.PI / 2 * (part.name.includes('l') ? 1 : -1);
      }
    }
    
    meshGroup.add(mesh);
    part.meshRef = mesh;
  });
  
  scene.add(meshGroup);
  
  // สร้าง SkeletonHelper สำหรับ visualization
  // เพิ่ม root bone เป็น child ของ group
  const rootBone = bones['root'];
  meshGroup.add(rootBone);
  
  // สร้าง dummy skinned mesh เพื่อให้ SkeletonHelper ทำงานได้
  const dummyGeo = new THREE.BufferGeometry();
  dummyGeo.setAttribute('position', new THREE.BufferAttribute(new Float32Array([0,0,0]), 3));
  const dummyMesh = new THREE.SkinnedMesh(dummyGeo, new THREE.MeshBasicMaterial({ skinning: true }));
  dummyMesh.add(rootBone);
  dummyMesh.bind(skeleton);
  scene.add(dummyMesh);
  
  const skeletonHelper = new THREE.SkeletonHelper(dummyMesh);
  skeletonHelper.material.linewidth = 2;
  scene.add(skeletonHelper);
  
  return {
    meshGroup,
    skeleton,
    bones,
    bodyParts,
    skeletonHelper,
    
    // Method สำหรับ update body parts ตาม bones
    update() {
      bodyParts.forEach(part => {
        const bone = bones[part.bone];
        if (!bone || !part.meshRef) return;
        
        // Get world position ของ bone
        const worldPos = new THREE.Vector3();
        bone.getWorldPosition(worldPos);
        worldPos.y -= meshGroup.position.y;
        
        // Update mesh position
        // (simplified - in real skinning this happens per-vertex on GPU)
      });
    }
  };
}

// ===== BUILD SKELETON =====
const humanSkeleton = new HumanSkeleton();
const humanMesh = createHumanMesh(humanSkeleton);
const bones = humanSkeleton.bones;

// ===== BONE ANIMATION =====
let walkAnimation = false;
let animTime = 0;

const DEFAULT_POSE = {};

function savePose() {
  Object.entries(bones).forEach(([name, bone]) => {
    DEFAULT_POSE[name] = {
      rotation: bone.rotation.clone()
    };
  });
}

function resetPose() {
  Object.entries(DEFAULT_POSE).forEach(([name, pose]) => {
    if (bones[name]) {
      bones[name].rotation.copy(pose.rotation);
    }
  });
  
  // Reset sliders
  document.querySelectorAll('input[type="range"]').forEach(input => {
    input.value = 0;
  });
}

function walkCycle(time) {
  const speed = 2;
  const t = time * speed;
  
  // Legs
  bones['leftThigh'].rotation.x = Math.sin(t) * 0.6;
  bones['leftShin'].rotation.x = Math.max(0, -Math.sin(t) * 0.7);
  bones['rightThigh'].rotation.x = Math.sin(t + Math.PI) * 0.6;
  bones['rightShin'].rotation.x = Math.max(0, -Math.sin(t + Math.PI) * 0.7);
  
  // Feet
  if (bones['leftFoot']) {
    bones['leftFoot'].rotation.x = Math.sin(t) * 0.2;
  }
  if (bones['rightFoot']) {
    bones['rightFoot'].rotation.x = Math.sin(t + Math.PI) * 0.2;
  }
  
  // Arms (opposite to legs)
  bones['leftUpperArm'].rotation.x = Math.sin(t + Math.PI) * 0.4;
  bones['rightUpperArm'].rotation.x = Math.sin(t) * 0.4;
  
  // Spine sway
  bones['spine'].rotation.z = Math.sin(t * 2) * 0.05;
  bones['spine'].rotation.x = -0.05 + Math.sin(t) * 0.03;
}

// ===== SLIDER CONTROLS =====
const DEG = Math.PI / 180;

function setupSlider(id, boneName, axis) {
  const slider = document.getElementById(id);
  if (!slider) return;
  
  slider.addEventListener('input', (e) => {
    if (bones[boneName]) {
      bones[boneName].rotation[axis] = parseInt(e.target.value) * DEG;
    }
  });
}

setupSlider('spineX', 'spine', 'x');
setupSlider('spineY', 'spine', 'y');
setupSlider('headX', 'head', 'x');
setupSlider('headY', 'head', 'y');
setupSlider('lArmX', 'leftUpperArm', 'x');
setupSlider('rArmX', 'rightUpperArm', 'x');
setupSlider('lElbow', 'leftForeArm', 'x');
setupSlider('rElbow', 'rightForeArm', 'x');
setupSlider('lHipX', 'leftThigh', 'x');
setupSlider('rHipX', 'rightThigh', 'x');

document.getElementById('resetBtn').addEventListener('click', resetPose);
document.getElementById('walkBtn').addEventListener('click', () => {
  walkAnimation = !walkAnimation;
  document.getElementById('walkBtn').textContent = 
    walkAnimation ? 'Stop Walk' : 'Walk Anim';
});

// Save initial pose
savePose();

// ===== ANIMATION LOOP =====
const clock = new THREE.Clock();

function animate() {
  requestAnimationFrame(animate);
  
  const delta = clock.getDelta();
  animTime += delta;
  
  if (walkAnimation) {
    walkCycle(animTime);
    
    // Update sliders to reflect animation
    document.getElementById('lHipX').value = 
      Math.round(bones['leftThigh'].rotation.x / DEG);
    document.getElementById('rHipX').value = 
      Math.round(bones['rightThigh'].rotation.x / DEG);
  }
  
  controls.update();
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

## 6. Procedural Skeleton Animation - Creature

```javascript
// ===== PROCEDURAL CREATURE ANIMATION =====
class ProceduralCreature {
  constructor(scene) {
    this.scene = scene;
    this.time = 0;
    this.segments = [];
    this.buildBody();
  }
  
  buildBody() {
    const numSegments = 12;
    const segmentLength = 0.4;
    
    // สร้าง spine chain
    for (let i = 0; i < numSegments; i++) {
      const geo = new THREE.SphereGeometry(
        0.15 * (1 - i / numSegments * 0.5), 
        8, 8
      );
      const mat = new THREE.MeshStandardMaterial({
        color: new THREE.Color().setHSL(0.3 - i * 0.02, 0.8, 0.4),
        roughness: 0.7
      });
      
      const mesh = new THREE.Mesh(geo, mat);
      mesh.castShadow = true;
      this.scene.add(mesh);
      
      this.segments.push({
        mesh,
        position: new THREE.Vector3(0, 0.5, i * segmentLength),
        targetPos: new THREE.Vector3(),
        velocity: new THREE.Vector3()
      });
    }
    
    // เพิ่ม legs (4 ขา)
    this.legs = [];
    const legPositions = [2, 5]; // segment indices สำหรับขา
    
    legPositions.forEach(segIdx => {
      // ขาซ้ายและขาขวา
      [-1, 1].forEach(side => {
        const upperLeg = new THREE.Mesh(
          new THREE.CylinderGeometry(0.04, 0.05, 0.5),
          new THREE.MeshStandardMaterial({ color: 0x225533 })
        );
        const lowerLeg = new THREE.Mesh(
          new THREE.CylinderGeometry(0.03, 0.04, 0.4),
          new THREE.MeshStandardMaterial({ color: 0x335544 })
        );
        
        this.scene.add(upperLeg);
        this.scene.add(lowerLeg);
        
        this.legs.push({
          upperLeg, lowerLeg,
          segIdx,
          side,
          footPos: new THREE.Vector3(),
          targetFootPos: new THREE.Vector3()
        });
      });
    });
  }
  
  update(delta) {
    this.time += delta;
    
    // Head follows a sine wave path
    const headPos = this.segments[0].position;
    headPos.x = Math.sin(this.time * 0.8) * 2;
    headPos.z = -this.time * 0.5; // เดินไปข้างหน้า
    headPos.y = 0.5 + Math.sin(this.time * 2) * 0.1;
    
    // Chain following (FABRIK-like)
    for (let i = 1; i < this.segments.length; i++) {
      const seg = this.segments[i];
      const prevSeg = this.segments[i - 1];
      
      const diff = seg.position.clone().sub(prevSeg.position);
      const dist = diff.length();
      
      if (dist > 0.4) {
        // ดึง segment ให้ follow segment ก่อนหน้า
        diff.normalize().multiplyScalar(0.4);
        seg.position.copy(prevSeg.position).add(diff);
      }
    }
    
    // อัพเดต visual positions
    this.segments.forEach((seg, i) => {
      seg.mesh.position.copy(seg.position);
      
      // หมุนเพื่อให้หันไปทิศทางการเคลื่อนที่
      if (i < this.segments.length - 1) {
        const next = this.segments[i + 1];
        const dir = seg.position.clone().sub(next.position);
        seg.mesh.lookAt(seg.position.clone().add(dir));
      }
    });
    
    // อัพเดต legs
    this.legs.forEach((leg, i) => {
      const bodyPos = this.segments[leg.segIdx].position.clone();
      
      // คำนวณตำแหน่งเท้าที่ต้องการ
      const legOffset = new THREE.Vector3(leg.side * 0.5, -0.4, 0);
      
      // หมุน offset ตามทิศทางตัว
      if (leg.segIdx > 0) {
        const bodyDir = this.segments[0].position.clone()
          .sub(this.segments[this.segments.length - 1].position)
          .normalize();
        const angle = Math.atan2(bodyDir.x, bodyDir.z);
        legOffset.applyAxisAngle(new THREE.Vector3(0, 1, 0), angle);
      }
      
      const desiredFootPos = bodyPos.clone().add(legOffset);
      desiredFootPos.y = 0; // ติดพื้น
      
      // Foot stepping (เท้าเหยียบตาม pattern)
      const stepPhase = this.time * 2 + i * Math.PI / 2;
      if (Math.sin(stepPhase) > 0.7) {
        leg.footPos.lerp(desiredFootPos, 0.3);
        leg.footPos.y = Math.max(0, Math.sin(stepPhase) * 0.2);
      } else {
        leg.footPos.lerp(desiredFootPos, 0.05);
        leg.footPos.y = 0;
      }
      
      // IK for leg (2-bone IK)
      const hip = bodyPos;
      const foot = leg.footPos;
      
      const L1 = 0.5, L2 = 0.4;
      const diff = foot.clone().sub(hip);
      const dist = Math.min(diff.length(), L1 + L2 - 0.01);
      
      // Calculate knee position
      const cosA = (L1 * L1 + dist * dist - L2 * L2) / (2 * L1 * dist);
      const A = Math.acos(Math.max(-1, Math.min(1, cosA)));
      
      const dirToFoot = diff.normalize();
      const kneeOffset = new THREE.Vector3(-leg.side * 0.2, 0.1, 0);
      const perp = dirToFoot.clone().cross(kneeOffset).normalize();
      
      const kneePos = hip.clone()
        .add(dirToFoot.clone().multiplyScalar(Math.cos(A) * L1))
        .add(perp.multiplyScalar(Math.sin(A) * L1));
      
      // อัพเดต leg meshes
      this.updateLimbMesh(leg.upperLeg, hip, kneePos);
      this.updateLimbMesh(leg.lowerLeg, kneePos, foot);
    });
  }
  
  updateLimbMesh(limb, start, end) {
    const mid = start.clone().add(end).multiplyScalar(0.5);
    limb.position.copy(mid);
    
    const dir = end.clone().sub(start);
    const length = dir.length();
    
    limb.scale.y = length;
    limb.lookAt(end);
    limb.rotateX(Math.PI / 2);
  }
}
```

## 7. IK (Inverse Kinematics) Concept

```javascript
// ===== 2-Bone IK (Arm/Leg) =====
class TwoBoneIK {
  constructor(boneA, boneB, boneC) {
    this.upper = boneA;   // shoulder / hip
    this.lower = boneB;   // elbow / knee
    this.end = boneC;     // hand / foot (end effector target)
    
    // Pre-calculate bone lengths
    this.upperLength = boneB.position.length();
    this.lowerLength = boneC.position.length();
  }
  
  solve(targetPos, polePos) {
    // root bone world position
    const rootPos = new THREE.Vector3();
    this.upper.getWorldPosition(rootPos);
    
    // Vector from root to target
    const toTarget = targetPos.clone().sub(rootPos);
    const dist = Math.min(
      toTarget.length(), 
      this.upperLength + this.lowerLength - 0.001
    );
    
    // Clamp target distance
    toTarget.normalize().multiplyScalar(dist);
    const clampedTarget = rootPos.clone().add(toTarget);
    
    // Calculate angles using law of cosines
    const L1 = this.upperLength;
    const L2 = this.lowerLength;
    const d = dist;
    
    const cosUpper = (L1 * L1 + d * d - L2 * L2) / (2 * L1 * d);
    const cosLower = (L1 * L1 + L2 * L2 - d * d) / (2 * L1 * L2);
    
    const upperAngle = Math.acos(Math.max(-1, Math.min(1, cosUpper)));
    const lowerAngle = Math.acos(Math.max(-1, Math.min(1, cosLower)));
    
    // Calculate pole vector direction (สำหรับ knee/elbow orientation)
    const poleDir = polePos 
      ? polePos.clone().sub(rootPos).normalize() 
      : new THREE.Vector3(0, 1, 0);
    
    // Apply rotations to bones
    const toTargetDir = toTarget.clone().normalize();
    const poleRight = toTargetDir.clone().cross(poleDir).normalize();
    const actualPole = poleRight.clone().cross(toTargetDir).normalize();
    
    // Upper bone rotation
    const upperQ = new THREE.Quaternion();
    const upperMatrix = new THREE.Matrix4();
    upperMatrix.lookAt(
      new THREE.Vector3(),
      toTargetDir,
      actualPole
    );
    upperQ.setFromRotationMatrix(upperMatrix);
    
    // Apply pole angle rotation
    const poleRot = new THREE.Quaternion();
    poleRot.setFromAxisAngle(toTargetDir, 0);
    upperQ.multiply(poleRot);
    
    // Rotate by upper angle
    const upperBendQ = new THREE.Quaternion();
    upperBendQ.setFromAxisAngle(poleRight, -upperAngle);
    upperQ.multiply(upperBendQ);
    
    this.upper.quaternion.copy(upperQ);
    
    // Lower bone bends down by remaining angle
    const lowerQ = new THREE.Quaternion();
    lowerQ.setFromAxisAngle(poleRight, -(Math.PI - lowerAngle));
    this.lower.quaternion.copy(lowerQ);
  }
}

// ===== FABRIK IK (สำหรับ chain หลาย joint) =====
class FABRIKSolver {
  constructor(bones) {
    this.bones = bones;
    this.lengths = [];
    
    // คำนวณความยาวระหว่างแต่ละ bone
    for (let i = 0; i < bones.length - 1; i++) {
      const diff = bones[i + 1].position.length();
      this.lengths.push(diff);
    }
    
    this.totalLength = this.lengths.reduce((a, b) => a + b, 0);
  }
  
  // positions: world space positions ของ joints
  solve(positions, target, iterations = 10) {
    const n = positions.length;
    const origin = positions[0].clone();
    
    // ตรวจสอบว่า target ไกลเกินไปหรือไม่
    const distToTarget = positions[0].distanceTo(target);
    
    if (distToTarget > this.totalLength) {
      // Stretch ไปยัง target
      for (let i = 0; i < n - 1; i++) {
        const r = target.distanceTo(positions[i]);
        const lambda = this.lengths[i] / r;
        positions[i + 1].lerpVectors(positions[i], target, lambda);
      }
      return positions;
    }
    
    for (let iter = 0; iter < iterations; iter++) {
      // Forward pass: เริ่มจาก end effector
      positions[n - 1].copy(target);
      
      for (let i = n - 2; i >= 0; i--) {
        const r = positions[i + 1].distanceTo(positions[i]);
        const lambda = this.lengths[i] / r;
        positions[i].lerpVectors(positions[i + 1], positions[i], lambda);
      }
      
      // Backward pass: เริ่มจาก root
      positions[0].copy(origin);
      
      for (let i = 0; i < n - 1; i++) {
        const r = positions[i + 1].distanceTo(positions[i]);
        const lambda = this.lengths[i] / r;
        positions[i + 1].lerpVectors(positions[i], positions[i + 1], lambda);
      }
    }
    
    return positions;
  }
}
```

## 8. Demo 3: IK Arm Demo

```html
<!DOCTYPE html>
<html>
<head>
  <title>IK Arm Demo</title>
  <style>
    body { margin: 0; background: #111; }
    canvas { display: block; }
    #info {
      position: fixed;
      bottom: 20px;
      left: 50%;
      transform: translateX(-50%);
      color: #adf;
      font-family: Arial, sans-serif;
      text-align: center;
      pointer-events: none;
    }
  </style>
</head>
<body>
<div id="info">เลื่อน mouse เพื่อควบคุม IK Target | กด SPACE สำหรับ auto mode</div>

<script type="module">
import * as THREE from 'https://unpkg.com/three@0.158.0/build/three.module.js';
import { OrbitControls } from 'https://unpkg.com/three@0.158.0/examples/jsm/controls/OrbitControls.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111122);
scene.add(new THREE.AmbientLight(0x334455, 2));
const dirLight = new THREE.DirectionalLight(0xffffff, 2);
dirLight.position.set(5, 10, 5);
scene.add(dirLight);

const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 100);
camera.position.set(0, 3, 8);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.target.set(0, 2, 0);

// ===== IK ARM SETUP =====
// Bone structure: Shoulder -> Elbow -> Wrist -> End

const shoulderPos = new THREE.Vector3(-2, 3, 0);
const elbowPos = new THREE.Vector3(-1, 2, 0);
const wristPos = new THREE.Vector3(0, 2, 0);
const handPos = new THREE.Vector3(1, 2, 0);

// จุดแสดง joint
function createJointSphere(radius, color) {
  const mesh = new THREE.Mesh(
    new THREE.SphereGeometry(radius, 12, 12),
    new THREE.MeshStandardMaterial({ color, emissive: color, emissiveIntensity: 0.3 })
  );
  scene.add(mesh);
  return mesh;
}

// จุดแสดง bone (cylinder ระหว่าง joints)
function createBoneMesh(color) {
  const geo = new THREE.CylinderGeometry(0.06, 0.06, 1, 8);
  const mat = new THREE.MeshStandardMaterial({ color });
  const mesh = new THREE.Mesh(geo, mat);
  scene.add(mesh);
  return mesh;
}

function updateBone(bone, start, end) {
  const mid = start.clone().add(end).multiplyScalar(0.5);
  bone.position.copy(mid);
  
  const dir = end.clone().sub(start);
  const len = dir.length();
  bone.scale.y = len;
  
  if (len > 0.001) {
    bone.lookAt(end);
    bone.rotateX(Math.PI / 2);
  }
}

const shoulderJoint = createJointSphere(0.12, 0xff8800);
const elbowJoint = createJointSphere(0.1, 0xffaa00);
const wristJoint = createJointSphere(0.1, 0xffcc00);
const handJoint = createJointSphere(0.08, 0xffee00);

const upperBone = createBoneMesh(0x6699cc);
const lowerBone = createBoneMesh(0x4477aa);
const handBone = createBoneMesh(0x2255aa);

// Target indicator
const targetMesh = new THREE.Mesh(
  new THREE.SphereGeometry(0.15, 12, 12),
  new THREE.MeshStandardMaterial({ 
    color: 0xff4444, 
    emissive: 0xff4444, 
    emissiveIntensity: 0.5,
    wireframe: true
  })
);
scene.add(targetMesh);

// Pole target (knee/elbow direction)
const poleMesh = new THREE.Mesh(
  new THREE.OctahedronGeometry(0.1),
  new THREE.MeshStandardMaterial({ color: 0x44ff44, emissive: 0x44ff44, emissiveIntensity: 0.5 })
);
poleMesh.position.set(-1, 4, 1);
scene.add(poleMesh);

// ===== FABRIK SOLVER =====
class SimpleFABRIK {
  constructor(positions, lengths) {
    this.positions = positions.map(p => p.clone());
    this.lengths = lengths;
    this.root = positions[0].clone();
  }
  
  solve(target, iterations = 10) {
    const n = this.positions.length;
    const totalLen = this.lengths.reduce((a, b) => a + b, 0);
    
    const distToTarget = this.root.distanceTo(target);
    
    if (distToTarget > totalLen) {
      // Over-extended: stretch straight
      const dir = target.clone().sub(this.root).normalize();
      for (let i = 1; i < n; i++) {
        const prev = this.positions[i - 1];
        this.positions[i] = prev.clone().add(dir.clone().multiplyScalar(this.lengths[i - 1]));
      }
      return;
    }
    
    for (let iter = 0; iter < iterations; iter++) {
      // Forward
      this.positions[n - 1].copy(target);
      for (let i = n - 2; i >= 0; i--) {
        const dir = this.positions[i].clone().sub(this.positions[i + 1]).normalize();
        this.positions[i].copy(this.positions[i + 1]).add(dir.multiplyScalar(this.lengths[i]));
      }
      
      // Backward
      this.positions[0].copy(this.root);
      for (let i = 0; i < n - 1; i++) {
        const dir = this.positions[i + 1].clone().sub(this.positions[i]).normalize();
        this.positions[i + 1].copy(this.positions[i]).add(dir.multiplyScalar(this.lengths[i]));
      }
    }
  }
}

// สร้าง FABRIK chain: Shoulder, Elbow, Wrist, Hand
const chainPositions = [
  shoulderPos.clone(),
  elbowPos.clone(),
  wristPos.clone(),
  handPos.clone()
];

const chainLengths = [1.0, 1.0, 0.5];
const fabrik = new SimpleFABRIK(chainPositions, chainLengths);

// ===== MOUSE TRACKING =====
const mouse = new THREE.Vector2();
const raycaster = new THREE.Raycaster();
const targetPlane = new THREE.Plane(new THREE.Vector3(0, 0, 1), 0);
const targetPoint = new THREE.Vector3();

window.addEventListener('mousemove', (e) => {
  mouse.set(
    (e.clientX / innerWidth) * 2 - 1,
    -(e.clientY / innerHeight) * 2 + 1
  );
  
  raycaster.setFromCamera(mouse, camera);
  raycaster.ray.intersectPlane(targetPlane, targetPoint);
});

let autoMode = false;
let autoTime = 0;

document.addEventListener('keydown', (e) => {
  if (e.code === 'Space') {
    autoMode = !autoMode;
    e.preventDefault();
  }
});

// ===== ANIMATION =====
const clock = new THREE.Clock();

function animate() {
  requestAnimationFrame(animate);
  
  const delta = clock.getDelta();
  autoTime += delta;
  
  // Update target position
  let target;
  if (autoMode) {
    target = new THREE.Vector3(
      Math.sin(autoTime * 1.5) * 2,
      2 + Math.cos(autoTime * 0.7) * 1.5,
      Math.sin(autoTime * 0.5) * 1
    );
  } else {
    target = targetPoint.length() > 0 
      ? targetPoint.clone() 
      : new THREE.Vector3(1, 2, 0);
  }
  
  targetMesh.position.copy(target);
  
  // Solve IK
  fabrik.root.copy(shoulderPos);
  fabrik.solve(target);
  
  // อัพเดต visual
  shoulderJoint.position.copy(fabrik.positions[0]);
  elbowJoint.position.copy(fabrik.positions[1]);
  wristJoint.position.copy(fabrik.positions[2]);
  handJoint.position.copy(fabrik.positions[3]);
  
  updateBone(upperBone, fabrik.positions[0], fabrik.positions[1]);
  updateBone(lowerBone, fabrik.positions[1], fabrik.positions[2]);
  updateBone(handBone, fabrik.positions[2], fabrik.positions[3]);
  
  controls.update();
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

## 9. Binding Pose และ Bone Inverse Matrices

```javascript
// Binding Pose คือ pose เริ่มต้นที่ skeleton อยู่ในตอน bind
// สำคัญมากสำหรับ skinning ที่ถูกต้อง

// เมื่อเรียก skeleton.bind() หรือ skinnedMesh.bind(skeleton)
// Three.js จะ calculate inverse bind matrices อัตโนมัติ

const skeleton = new THREE.Skeleton(bones);

// Manual binding พร้อม custom bind matrices
const bindMatrices = bones.map(bone => {
  const matrix = new THREE.Matrix4();
  bone.updateWorldMatrix(true, false);
  matrix.copy(bone.matrixWorld);
  return matrix;
});

skeleton.boneInverses = bindMatrices.map(m => m.clone().invert());

// skinnedMesh.bind(skeleton, bindMatrix)
// bindMatrix คือ inverse of root transform
const bindMatrix = new THREE.Matrix4();
skinnedMesh.bind(skeleton, bindMatrix);

// เมื่อ bone เคลื่อนที่:
// finalPosition = boneMatrix * boneInverse * originalPosition
// (แปลง original position เป็น bone space, แล้วแปลงกลับ world space ด้วย bone matrix ปัจจุบัน)
```

## สรุป

Skeletal Animation ใน Three.js ประกอบด้วย:

1. **Skeleton** - โครงสร้าง hierarchy ของ bones
2. **SkinnedMesh** - mesh ที่ผูกกับ skeleton
3. **skinIndex/skinWeight** - กำหนดว่า vertex ได้รับอิทธิพลจาก bone ไหน
4. **IK Solvers** - คำนวณมุม bone จาก target position

---

## แบบฝึกหัด

1. **Easy**: สร้าง skeleton แบบง่ายมี 3 bones และสร้าง simple pendulum animation
2. **Medium**: สร้าง 2-bone IK arm ที่ follow mouse cursor
3. **Hard**: สร้าง procedural spider ที่มี 8 ขาซึ่งใช้ IK ทั้งหมด ให้เดินบนพื้นผิวขรุขระ
4. **Challenge**: สร้าง full biped character ด้วย SkinnedMesh จริงๆ ที่มี skinning weights และทำ walk cycle animation
