# Part 69: Render Targets & Off-Screen Rendering

## บทนำ

Render Targets คือ off-screen framebuffers ที่เราสามารถ render ฉาก Three.js ลงไปแล้วนำผลลัพธ์ไปใช้เป็น texture ใน scene หลัก เทคนิคนี้เป็นพื้นฐานสำคัญของ post-processing effects, reflection rendering, minimap systems และ GPGPU simulations

---

## 69.1 WebGLRenderTarget พื้นฐาน

```javascript
import * as THREE from 'three';

// สร้าง render target พื้นฐาน
const renderTarget = new THREE.WebGLRenderTarget(
  512,  // width
  512,  // height
  {
    // Texture options
    minFilter: THREE.LinearFilter,
    magFilter: THREE.LinearFilter,
    format: THREE.RGBAFormat,
    type: THREE.UnsignedByteType,
    
    // Stencil/Depth
    stencilBuffer: false,
    depthBuffer: true,
    
    // Multisampling (MSAA)
    samples: 4,
    
    // Color space
    colorSpace: THREE.SRGBColorSpace,
    
    // Wrap mode
    wrapS: THREE.ClampToEdgeWrapping,
    wrapT: THREE.ClampToEdgeWrapping,
  }
);

// เข้าถึง texture
const texture = renderTarget.texture;

// Resize render target
renderTarget.setSize(1024, 1024);

// Dispose เมื่อไม่ใช้แล้ว
renderTarget.dispose();
```

### Render ไปยัง Target

```javascript
// Render scene ลงไปใน render target
function renderToTarget(renderer, scene, camera, target) {
  // เก็บค่า current render target
  const currentTarget = renderer.getRenderTarget();
  
  // เปลี่ยนไปยัง target
  renderer.setRenderTarget(target);
  renderer.clear();  // clear ก่อน render
  renderer.render(scene, camera);
  
  // กลับไปยัง default target (screen)
  renderer.setRenderTarget(null);
}
```

---

## 69.2 Render to Texture Demo

```javascript
// src/demos/renderToTexture.js
import * as THREE from 'three';

export function createRenderToTextureDemo(container) {
  // === Main Scene ===
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#0a0a1a');
  
  const camera = new THREE.PerspectiveCamera(60, container.clientWidth / container.clientHeight, 0.1, 100);
  camera.position.set(0, 2, 8);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  container.appendChild(renderer.domElement);
  
  // === Off-screen Scene (จะ render ลง texture) ===
  const offscreenScene = new THREE.Scene();
  offscreenScene.background = new THREE.Color('#1a1a3a');
  
  const offscreenCamera = new THREE.PerspectiveCamera(60, 1, 0.1, 100);
  offscreenCamera.position.z = 3;
  
  // เพิ่ม objects ใน offscreen scene
  const offObjects = [];
  for (let i = 0; i < 5; i++) {
    const mesh = new THREE.Mesh(
      new THREE.BoxGeometry(0.5, 0.5, 0.5),
      new THREE.MeshStandardMaterial({
        color: new THREE.Color().setHSL(i / 5, 0.8, 0.6),
        metalness: 0.5,
        roughness: 0.3
      })
    );
    mesh.position.set(
      (i - 2) * 0.8,
      Math.sin(i * 1.2) * 0.5,
      (Math.random() - 0.5) * 2
    );
    offscreenScene.add(mesh);
    offObjects.push(mesh);
  }
  
  // Lighting สำหรับ offscreen scene
  offscreenScene.add(new THREE.AmbientLight(0xffffff, 0.5));
  const offLight = new THREE.DirectionalLight(0xffffff, 1);
  offLight.position.set(3, 5, 3);
  offscreenScene.add(offLight);
  
  // === Render Target ===
  const renderTarget = new THREE.WebGLRenderTarget(512, 512, {
    minFilter: THREE.LinearFilter,
    magFilter: THREE.LinearFilter,
    format: THREE.RGBAFormat,
  });
  
  // === Display Screen (ใน main scene) ===
  // Monitor frame
  const frameGeometry = new THREE.BoxGeometry(4.2, 3.2, 0.2);
  const frameMesh = new THREE.Mesh(
    frameGeometry,
    new THREE.MeshStandardMaterial({ color: '#222', metalness: 0.8, roughness: 0.2 })
  );
  frameMesh.position.set(0, 0, -0.1);
  scene.add(frameMesh);
  
  // Screen surface - แสดง render target texture
  const screenGeometry = new THREE.PlaneGeometry(4, 3);
  const screenMaterial = new THREE.MeshBasicMaterial({
    map: renderTarget.texture  // ใช้ texture จาก render target
  });
  const screenMesh = new THREE.Mesh(screenGeometry, screenMaterial);
  scene.add(screenMesh);
  
  // Stand
  const standMesh = new THREE.Mesh(
    new THREE.BoxGeometry(0.3, 2, 0.3),
    new THREE.MeshStandardMaterial({ color: '#333', metalness: 0.8, roughness: 0.3 })
  );
  standMesh.position.set(0, -2.5, 0);
  scene.add(standMesh);
  
  // Main scene lighting
  scene.add(new THREE.AmbientLight(0xffffff, 0.6));
  const mainLight = new THREE.DirectionalLight(0xffffff, 1);
  mainLight.position.set(5, 10, 5);
  scene.add(mainLight);
  
  // === Animation ===
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    
    // Animate offscreen objects
    offObjects.forEach((obj, i) => {
      obj.rotation.x = elapsed * (0.5 + i * 0.1);
      obj.rotation.y = elapsed * (0.3 + i * 0.15);
      obj.position.y = Math.sin(elapsed * 2 + i * 1.2) * 0.5;
    });
    
    // Render offscreen scene ลงไปใน texture
    renderer.setRenderTarget(renderTarget);
    renderer.render(offscreenScene, offscreenCamera);
    renderer.setRenderTarget(null);
    
    // Render main scene
    renderer.render(scene, camera);
  }
  
  animate();
  
  // Handle resize
  window.addEventListener('resize', () => {
    const w = container.clientWidth;
    const h = container.clientHeight;
    camera.aspect = w / h;
    camera.updateProjectionMatrix();
    renderer.setSize(w, h);
  });
  
  return () => {
    cancelAnimationFrame(animId);
    renderTarget.dispose();
    renderer.dispose();
  };
}
```

---

## 69.3 Ping-Pong Technique

Ping-Pong ใช้ render targets สองตัวสลับกันเพื่อ simulate effects ที่ต้องอาศัย previous frame

```javascript
// src/demos/pingPong.js
import * as THREE from 'three';

// Shader สำหรับ fluid simulation แบบง่าย
const fluidVertexShader = `
  varying vec2 vUv;
  void main() {
    vUv = uv;
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
  }
`;

const fluidFragmentShader = `
  uniform sampler2D uPreviousState;
  uniform vec2 uResolution;
  uniform float uTime;
  uniform vec2 uMouse;
  uniform float uMouseActive;
  varying vec2 vUv;

  void main() {
    vec2 texel = 1.0 / uResolution;
    
    // Read neighbors
    vec4 center = texture2D(uPreviousState, vUv);
    vec4 left   = texture2D(uPreviousState, vUv - vec2(texel.x, 0.0));
    vec4 right  = texture2D(uPreviousState, vUv + vec2(texel.x, 0.0));
    vec4 up     = texture2D(uPreviousState, vUv + vec2(0.0, texel.y));
    vec4 down   = texture2D(uPreviousState, vUv - vec2(0.0, texel.y));
    
    // Simple wave equation
    float average = (left.r + right.r + up.r + down.r) * 0.5;
    float newValue = average - center.g;
    newValue *= 0.99; // damping
    
    // Mouse interaction
    float dist = distance(vUv, uMouse);
    if (uMouseActive > 0.5 && dist < 0.03) {
      newValue += 1.0 * (1.0 - dist / 0.03);
    }
    
    gl_FragColor = vec4(newValue, center.r, 0.0, 1.0);
  }
`;

const displayFragmentShader = `
  uniform sampler2D uState;
  varying vec2 vUv;
  
  void main() {
    vec4 state = texture2D(uState, vUv);
    float height = state.r;
    
    // Color based on height
    vec3 color = mix(
      vec3(0.0, 0.1, 0.3),   // deep water color
      vec3(0.5, 0.8, 1.0),   // surface color
      height * 0.5 + 0.5
    );
    
    // Fresnel-like edge highlight
    float edge = abs(height) * 2.0;
    color += vec3(edge * 0.3);
    
    gl_FragColor = vec4(color, 1.0);
  }
`;

export function createPingPongDemo(container) {
  const SIZE = 512;
  
  // === Setup ===
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  container.appendChild(renderer.domElement);
  
  const scene = new THREE.Scene();
  const camera = new THREE.OrthographicCamera(-1, 1, 1, -1, 0, 1);
  
  // === Ping-Pong Render Targets ===
  const createTarget = () => new THREE.WebGLRenderTarget(SIZE, SIZE, {
    minFilter: THREE.NearestFilter,
    magFilter: THREE.NearestFilter,
    format: THREE.RGBAFormat,
    type: THREE.FloatType,  // Float texture สำหรับ precision
  });
  
  let targetA = createTarget();
  let targetB = createTarget();
  
  // === Simulation Material ===
  const simulationMaterial = new THREE.ShaderMaterial({
    uniforms: {
      uPreviousState: { value: null },
      uResolution: { value: new THREE.Vector2(SIZE, SIZE) },
      uTime: { value: 0 },
      uMouse: { value: new THREE.Vector2(0.5, 0.5) },
      uMouseActive: { value: 0 },
    },
    vertexShader: fluidVertexShader,
    fragmentShader: fluidFragmentShader,
  });
  
  // === Display Material ===
  const displayMaterial = new THREE.ShaderMaterial({
    uniforms: {
      uState: { value: null },
    },
    vertexShader: fluidVertexShader,
    fragmentShader: displayFragmentShader,
  });
  
  // === Full-screen Quad ===
  const quad = new THREE.Mesh(new THREE.PlaneGeometry(2, 2), simulationMaterial);
  scene.add(quad);
  
  // Display scene (แยก scene สำหรับ display)
  const displayScene = new THREE.Scene();
  const displayQuad = new THREE.Mesh(new THREE.PlaneGeometry(2, 2), displayMaterial);
  displayScene.add(displayQuad);
  
  // === Mouse Tracking ===
  const mouse = new THREE.Vector2(0.5, 0.5);
  let isMouseDown = false;
  
  container.addEventListener('mousemove', (e) => {
    const rect = container.getBoundingClientRect();
    mouse.x = (e.clientX - rect.left) / rect.width;
    mouse.y = 1 - (e.clientY - rect.top) / rect.height;
  });
  
  container.addEventListener('mousedown', () => { isMouseDown = true; });
  container.addEventListener('mouseup', () => { isMouseDown = false; });
  
  // === Animation ===
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    
    // Update simulation uniforms
    simulationMaterial.uniforms.uPreviousState.value = targetA.texture;
    simulationMaterial.uniforms.uTime.value = elapsed;
    simulationMaterial.uniforms.uMouse.value.copy(mouse);
    simulationMaterial.uniforms.uMouseActive.value = isMouseDown ? 1 : 0;
    
    // Render simulation ลงไปใน targetB
    quad.material = simulationMaterial;
    renderer.setRenderTarget(targetB);
    renderer.render(scene, camera);
    
    // Display result
    displayMaterial.uniforms.uState.value = targetB.texture;
    displayQuad.material = displayMaterial;
    renderer.setRenderTarget(null);
    renderer.render(displayScene, camera);
    
    // Swap targets (ping-pong!)
    [targetA, targetB] = [targetB, targetA];
  }
  
  animate();
  
  return () => {
    cancelAnimationFrame(animId);
    targetA.dispose();
    targetB.dispose();
    renderer.dispose();
  };
}
```

---

## 69.4 Multiple Render Targets (MRT)

```javascript
// src/demos/mrt.js
import * as THREE from 'three';

// MRT Fragment Shader - เขียนหลาย outputs พร้อมกัน
const mrtFragmentShader = `
  varying vec2 vUv;
  varying vec3 vNormal;
  varying vec3 vPosition;
  
  layout(location = 0) out vec4 gColor;     // Color buffer
  layout(location = 1) out vec4 gNormal;    // Normal buffer
  layout(location = 2) out vec4 gPosition;  // Position buffer
  layout(location = 3) out vec4 gEmissive;  // Emissive/depth buffer
  
  uniform vec3 uColor;
  uniform float uMetalness;
  uniform float uRoughness;
  
  void main() {
    // Write to multiple render targets simultaneously
    gColor = vec4(uColor, 1.0);
    gNormal = vec4(vNormal * 0.5 + 0.5, 1.0);  // Pack to 0-1 range
    gPosition = vec4(vPosition, 1.0);
    gEmissive = vec4(uMetalness, uRoughness, 0.0, 1.0);
  }
`;

export function createMRTDemo(container) {
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  container.appendChild(renderer.domElement);
  
  // ตรวจสอบว่า MRT supported
  if (!renderer.capabilities.isWebGL2) {
    console.warn('WebGL2 required for MRT');
    return;
  }
  
  const scene = new THREE.Scene();
  const camera = new THREE.PerspectiveCamera(60, container.clientWidth / container.clientHeight, 0.1, 100);
  camera.position.z = 5;
  
  // === Multiple Render Targets ===
  const SIZE = 512;
  const mrt = new THREE.WebGLMultipleRenderTargets(SIZE, SIZE, 4);
  
  // ตั้งค่า texture ทั้ง 4
  mrt.texture.forEach((texture, i) => {
    texture.name = ['color', 'normal', 'position', 'emissive'][i];
    texture.minFilter = THREE.NearestFilter;
    texture.magFilter = THREE.NearestFilter;
  });
  
  // === Geometry Pass (G-Buffer) ===
  // Material ที่เขียนไปยัง MRT
  const gBufferMaterial = new THREE.RawShaderMaterial({
    vertexShader: `
      attribute vec3 position;
      attribute vec3 normal;
      attribute vec2 uv;
      
      uniform mat4 modelMatrix;
      uniform mat4 viewMatrix;
      uniform mat4 projectionMatrix;
      uniform mat3 normalMatrix;
      
      varying vec2 vUv;
      varying vec3 vNormal;
      varying vec3 vPosition;
      
      void main() {
        vUv = uv;
        vNormal = normalize(normalMatrix * normal);
        vPosition = (modelMatrix * vec4(position, 1.0)).xyz;
        gl_Position = projectionMatrix * viewMatrix * modelMatrix * vec4(position, 1.0);
      }
    `,
    fragmentShader: mrtFragmentShader,
    uniforms: {
      uColor: { value: new THREE.Color('#4ecdc4') },
      uMetalness: { value: 0.7 },
      uRoughness: { value: 0.2 },
    },
    glslVersion: THREE.GLSL3,
  });
  
  // เพิ่ม objects
  const mesh = new THREE.Mesh(new THREE.SphereGeometry(1.5, 32, 32), gBufferMaterial);
  scene.add(mesh);
  scene.add(new THREE.AmbientLight(0xffffff, 0.3));
  
  // === Display G-Buffer Contents ===
  // แสดงผลแต่ละ buffer ในตาราง 2x2
  const displayScene = new THREE.Scene();
  const displayCamera = new THREE.OrthographicCamera(-1, 1, 1, -1, 0, 1);
  
  // สร้าง grid ของ quads แสดงแต่ละ buffer
  const quadPositions = [
    [-0.5, 0.5],   // top-left: color
    [0.5, 0.5],    // top-right: normal
    [-0.5, -0.5],  // bottom-left: position
    [0.5, -0.5],   // bottom-right: emissive
  ];
  
  quadPositions.forEach((pos, i) => {
    const quad = new THREE.Mesh(
      new THREE.PlaneGeometry(0.9, 0.9),
      new THREE.MeshBasicMaterial({ map: mrt.texture[i] })
    );
    quad.position.set(pos[0], pos[1], 0);
    displayScene.add(quad);
  });
  
  // === Animation ===
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    
    mesh.rotation.x = elapsed * 0.3;
    mesh.rotation.y = elapsed * 0.5;
    
    // Geometry pass - render ลงใน MRT
    renderer.setRenderTarget(mrt);
    renderer.clear();
    renderer.render(scene, camera);
    
    // Display pass - แสดงผล buffers
    renderer.setRenderTarget(null);
    renderer.render(displayScene, displayCamera);
  }
  
  animate();
  
  return () => {
    cancelAnimationFrame(animId);
    mrt.dispose();
    renderer.dispose();
  };
}
```

---

## 69.5 Minimap Render Target

```javascript
// src/demos/minimap.js
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

export function createMinimapDemo(container) {
  // === Main Setup ===
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#1a1a2e');
  
  const mainCamera = new THREE.PerspectiveCamera(
    60,
    container.clientWidth / container.clientHeight,
    0.1,
    1000
  );
  mainCamera.position.set(5, 3, 8);
  
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  container.appendChild(renderer.domElement);
  
  const controls = new OrbitControls(mainCamera, renderer.domElement);
  controls.enableDamping = true;
  
  // === Minimap Camera (top-down) ===
  const minimapCamera = new THREE.OrthographicCamera(-15, 15, 15, -15, 0.1, 100);
  minimapCamera.position.set(0, 50, 0);
  minimapCamera.lookAt(0, 0, 0);
  minimapCamera.up.set(0, 0, -1);  // Correct up direction for top-down
  
  // === Minimap Render Target ===
  const minimapTarget = new THREE.WebGLRenderTarget(256, 256, {
    minFilter: THREE.LinearFilter,
    magFilter: THREE.LinearFilter,
  });
  
  // === Scene Objects ===
  // Floor
  const floor = new THREE.Mesh(
    new THREE.PlaneGeometry(30, 30),
    new THREE.MeshStandardMaterial({ color: '#333' })
  );
  floor.rotation.x = -Math.PI / 2;
  scene.add(floor);
  
  // Buildings/Objects
  const buildingColors = ['#e74c3c', '#3498db', '#2ecc71', '#f39c12', '#9b59b6'];
  for (let i = 0; i < 20; i++) {
    const height = 1 + Math.random() * 4;
    const building = new THREE.Mesh(
      new THREE.BoxGeometry(1 + Math.random(), height, 1 + Math.random()),
      new THREE.MeshStandardMaterial({
        color: buildingColors[Math.floor(Math.random() * buildingColors.length)]
      })
    );
    building.position.set(
      (Math.random() - 0.5) * 25,
      height / 2,
      (Math.random() - 0.5) * 25
    );
    scene.add(building);
  }
  
  // Player marker (แสดงตำแหน่ง camera ใน minimap)
  const playerDot = new THREE.Mesh(
    new THREE.SphereGeometry(0.3, 8, 8),
    new THREE.MeshBasicMaterial({ color: '#ff0000' })
  );
  playerDot.position.y = 10;  // สูงพอให้เห็นจาก minimap camera
  scene.add(playerDot);
  
  // Lighting
  scene.add(new THREE.AmbientLight(0xffffff, 0.5));
  const light = new THREE.DirectionalLight(0xffffff, 1);
  light.position.set(10, 20, 10);
  scene.add(light);
  
  // === Minimap Display (overlay ใน main scene) ===
  // สร้าง separate scene สำหรับ minimap UI
  const uiScene = new THREE.Scene();
  const uiCamera = new THREE.OrthographicCamera(-1, 1, 1, -1, 0, 1);
  
  // Minimap frame
  const minimapFrame = new THREE.Mesh(
    new THREE.PlaneGeometry(0.38, 0.38),
    new THREE.MeshBasicMaterial({ color: '#4ecdc4' })
  );
  minimapFrame.position.set(0.82, -0.62, 0);
  uiScene.add(minimapFrame);
  
  // Minimap display
  const minimapDisplay = new THREE.Mesh(
    new THREE.PlaneGeometry(0.35, 0.35),
    new THREE.MeshBasicMaterial({ map: minimapTarget.texture })
  );
  minimapDisplay.position.set(0.82, -0.62, 0.001);
  uiScene.add(minimapDisplay);
  
  // === Animation ===
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    
    controls.update();
    
    // อัพเดต player dot ตาม main camera position
    playerDot.position.x = mainCamera.position.x;
    playerDot.position.z = mainCamera.position.z;
    
    // Render ลงใน minimap target
    renderer.setRenderTarget(minimapTarget);
    renderer.render(scene, minimapCamera);
    renderer.setRenderTarget(null);
    
    // Render main scene
    renderer.render(scene, mainCamera);
    
    // Render minimap overlay (on top)
    renderer.autoClear = false;  // ไม่ clear ก่อน render overlay
    renderer.render(uiScene, uiCamera);
    renderer.autoClear = true;
  }
  
  animate();
  
  // Resize handler
  window.addEventListener('resize', () => {
    const w = container.clientWidth;
    const h = container.clientHeight;
    mainCamera.aspect = w / h;
    mainCamera.updateProjectionMatrix();
    renderer.setSize(w, h);
  });
  
  return () => {
    cancelAnimationFrame(animId);
    minimapTarget.dispose();
    renderer.dispose();
    controls.dispose();
  };
}
```

---

## 69.6 Post-Processing ด้วย Render Targets

```javascript
// src/demos/postProcessing.js
import * as THREE from 'three';

// Blur shader (horizontal pass)
const blurVertexShader = `
  varying vec2 vUv;
  void main() {
    vUv = uv;
    gl_Position = vec4(position, 1.0);
  }
`;

const horizontalBlurShader = `
  uniform sampler2D tDiffuse;
  uniform float h;
  varying vec2 vUv;
  
  void main() {
    vec4 sum = vec4(0.0);
    sum += texture2D(tDiffuse, vec2(vUv.x - 4.0 * h, vUv.y)) * 0.051;
    sum += texture2D(tDiffuse, vec2(vUv.x - 3.0 * h, vUv.y)) * 0.0918;
    sum += texture2D(tDiffuse, vec2(vUv.x - 2.0 * h, vUv.y)) * 0.1227;
    sum += texture2D(tDiffuse, vec2(vUv.x - 1.0 * h, vUv.y)) * 0.1633;
    sum += texture2D(tDiffuse, vec2(vUv.x, vUv.y)) * 0.1633;
    sum += texture2D(tDiffuse, vec2(vUv.x + 1.0 * h, vUv.y)) * 0.1633;
    sum += texture2D(tDiffuse, vec2(vUv.x + 2.0 * h, vUv.y)) * 0.1227;
    sum += texture2D(tDiffuse, vec2(vUv.x + 3.0 * h, vUv.y)) * 0.0918;
    sum += texture2D(tDiffuse, vec2(vUv.x + 4.0 * h, vUv.y)) * 0.051;
    gl_FragColor = sum;
  }
`;

const verticalBlurShader = `
  uniform sampler2D tDiffuse;
  uniform float v;
  varying vec2 vUv;
  
  void main() {
    vec4 sum = vec4(0.0);
    sum += texture2D(tDiffuse, vec2(vUv.x, vUv.y - 4.0 * v)) * 0.051;
    sum += texture2D(tDiffuse, vec2(vUv.x, vUv.y - 3.0 * v)) * 0.0918;
    sum += texture2D(tDiffuse, vec2(vUv.x, vUv.y - 2.0 * v)) * 0.1227;
    sum += texture2D(tDiffuse, vec2(vUv.x, vUv.y - 1.0 * v)) * 0.1633;
    sum += texture2D(tDiffuse, vec2(vUv.x, vUv.y)) * 0.1633;
    sum += texture2D(tDiffuse, vec2(vUv.x, vUv.y + 1.0 * v)) * 0.1633;
    sum += texture2D(tDiffuse, vec2(vUv.x, vUv.y + 2.0 * v)) * 0.1227;
    sum += texture2D(tDiffuse, vec2(vUv.x, vUv.y + 3.0 * v)) * 0.0918;
    sum += texture2D(tDiffuse, vec2(vUv.x, vUv.y + 4.0 * v)) * 0.051;
    gl_FragColor = sum;
  }
`;

// Composite (bloom) shader
const compositeShader = `
  uniform sampler2D tDiffuse;
  uniform sampler2D tBloom;
  uniform float uBloomStrength;
  varying vec2 vUv;
  
  void main() {
    vec4 original = texture2D(tDiffuse, vUv);
    vec4 bloom = texture2D(tBloom, vUv);
    gl_FragColor = original + bloom * uBloomStrength;
  }
`;

export function createBloomDemo(container) {
  const W = container.clientWidth;
  const H = container.clientHeight;
  
  // Renderer
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(W, H);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  container.appendChild(renderer.domElement);
  
  // Scene
  const scene = new THREE.Scene();
  scene.background = new THREE.Color('#000000');
  
  const camera = new THREE.PerspectiveCamera(60, W / H, 0.1, 100);
  camera.position.z = 5;
  
  // Glowing objects
  const objects = [];
  for (let i = 0; i < 8; i++) {
    const obj = new THREE.Mesh(
      new THREE.SphereGeometry(0.4, 32, 32),
      new THREE.MeshStandardMaterial({
        color: new THREE.Color().setHSL(i / 8, 1, 0.5),
        emissive: new THREE.Color().setHSL(i / 8, 1, 0.5),
        emissiveIntensity: 2,
      })
    );
    const angle = (i / 8) * Math.PI * 2;
    obj.position.set(Math.cos(angle) * 2.5, Math.sin(angle) * 2.5, 0);
    scene.add(obj);
    objects.push(obj);
  }
  
  // === Render Targets ===
  const sceneTarget = new THREE.WebGLRenderTarget(W, H);
  const blurTargetH = new THREE.WebGLRenderTarget(W / 4, H / 4);
  const blurTargetV = new THREE.WebGLRenderTarget(W / 4, H / 4);
  
  // Full-screen quad scene
  const fsScene = new THREE.Scene();
  const fsCamera = new THREE.OrthographicCamera(-1, 1, 1, -1, 0, 1);
  const fsQuad = new THREE.Mesh(new THREE.PlaneGeometry(2, 2));
  fsScene.add(fsQuad);
  
  // Materials
  const hBlurMat = new THREE.ShaderMaterial({
    uniforms: { tDiffuse: { value: null }, h: { value: 1 / (W / 4) } },
    vertexShader: blurVertexShader,
    fragmentShader: horizontalBlurShader,
  });
  
  const vBlurMat = new THREE.ShaderMaterial({
    uniforms: { tDiffuse: { value: null }, v: { value: 1 / (H / 4) } },
    vertexShader: blurVertexShader,
    fragmentShader: verticalBlurShader,
  });
  
  const compositeMat = new THREE.ShaderMaterial({
    uniforms: {
      tDiffuse: { value: null },
      tBloom: { value: null },
      uBloomStrength: { value: 1.5 },
    },
    vertexShader: blurVertexShader,
    fragmentShader: compositeShader,
  });
  
  // Animation
  const clock = new THREE.Clock();
  let animId;
  
  function animate() {
    animId = requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();
    
    objects.forEach((obj, i) => {
      const angle = (i / 8) * Math.PI * 2 + elapsed * 0.5;
      obj.position.set(Math.cos(angle) * 2.5, Math.sin(angle) * 2.5, 0);
    });
    
    // Pass 1: Render scene ลงใน sceneTarget
    renderer.setRenderTarget(sceneTarget);
    renderer.render(scene, camera);
    
    // Pass 2: Horizontal blur
    hBlurMat.uniforms.tDiffuse.value = sceneTarget.texture;
    fsQuad.material = hBlurMat;
    renderer.setRenderTarget(blurTargetH);
    renderer.render(fsScene, fsCamera);
    
    // Pass 3: Vertical blur
    vBlurMat.uniforms.tDiffuse.value = blurTargetH.texture;
    fsQuad.material = vBlurMat;
    renderer.setRenderTarget(blurTargetV);
    renderer.render(fsScene, fsCamera);
    
    // Pass 4: Composite (original + bloom)
    compositeMat.uniforms.tDiffuse.value = sceneTarget.texture;
    compositeMat.uniforms.tBloom.value = blurTargetV.texture;
    fsQuad.material = compositeMat;
    renderer.setRenderTarget(null);
    renderer.render(fsScene, fsCamera);
  }
  
  animate();
  
  return () => {
    cancelAnimationFrame(animId);
    [sceneTarget, blurTargetH, blurTargetV].forEach(t => t.dispose());
    renderer.dispose();
  };
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Security Camera System
สร้าง security camera system ที่:
- มี 4 cameras แต่ละมุม
- แสดง feeds ทั้ง 4 บน monitor
- Real-time update ทุก frame
- เพิ่ม scanline effect ให้ดูเหมือนกล้อง CCTV

### แบบฝึกหัดที่ 2: Water Reflection
สร้าง water reflection ด้วย render target:
- Top-down camera render เป็น texture
- Apply ลงบน water plane
- Add distortion ด้วย normal map
- Animate ripples

### แบบฝึกหัดที่ 3: Fluid Simulation
ปรับปรุง ping-pong fluid simulation ให้มี:
- Multi-pass simulation (velocity + pressure)
- Color injection บนพื้นที่ต่างกัน
- Interactive obstacles
- Export เป็น video

### แบบฝึกหัดที่ 4: Deferred Rendering
สร้าง deferred rendering pipeline ที่มี:
- G-Buffer: color, normal, position, roughness/metalness
- Lighting pass ที่คำนวณ SSAO
- Final composite pass

### แบบฝึกหัดที่ 5: Shadow Map Visualization
สร้าง debug tool ที่แสดง:
- Shadow map texture จาก lights
- Cascade shadow maps
- Visualize depth buffer

---

## สรุป

ในบทนี้เราเรียนรู้:
1. **WebGLRenderTarget** - สร้างและใช้ off-screen render targets
2. **Render to Texture** - Render scene ลงไปใน texture
3. **Ping-Pong** - Technique สำหรับ simulations
4. **MRT** - Multiple render targets สำหรับ deferred rendering
5. **Minimap** - Render target สำหรับ overview
6. **Post-Processing** - Custom bloom ด้วย render targets

บทถัดไปเราจะเรียน **GPGPU with Three.js**
