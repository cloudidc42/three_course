# Part 64: React Three Fiber (R3F) Basics

## บทนำ

React Three Fiber (R3F) คือ React renderer สำหรับ Three.js ที่ช่วยให้เราสามารถเขียน Three.js ในรูปแบบ declarative ด้วย JSX แทนที่จะเขียนแบบ imperative ทำให้โค้ดอ่านง่ายขึ้นและผสานกับ React ecosystem ได้อย่างสมบูรณ์

ในบทนี้เราจะเรียนรู้:
- การติดตั้งและตั้งค่า @react-three/fiber
- การใช้ Canvas component
- useFrame hook สำหรับ animation loop
- useThree hook สำหรับเข้าถึง Three.js internals
- การเขียน mesh/geometry/material ในรูปแบบ JSX
- การใช้ ref เพื่อเข้าถึง Three.js objects
- เปรียบเทียบ R3F กับ vanilla Three.js

---

## 64.1 ทำไมต้องใช้ React Three Fiber?

### ปัญหาของ Vanilla Three.js ใน React

เมื่อเราใช้ Three.js แบบดั้งเดิมใน React component จะพบปัญหาหลายอย่าง:

```jsx
// ❌ แบบเก่า - Imperative ใน React
import { useEffect, useRef } from 'react';
import * as THREE from 'three';

function ThreeScene() {
  const mountRef = useRef(null);

  useEffect(() => {
    // ต้องจัดการทุกอย่างเอง
    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(75, 800/600, 0.1, 1000);
    const renderer = new THREE.WebGLRenderer();
    
    renderer.setSize(800, 600);
    mountRef.current.appendChild(renderer.domElement);
    
    const geometry = new THREE.BoxGeometry();
    const material = new THREE.MeshBasicMaterial({ color: 0x00ff00 });
    const cube = new THREE.Mesh(geometry, material);
    scene.add(cube);
    
    camera.position.z = 5;
    
    let animId;
    const animate = () => {
      animId = requestAnimationFrame(animate);
      cube.rotation.x += 0.01;
      cube.rotation.y += 0.01;
      renderer.render(scene, camera);
    };
    animate();
    
    // ต้อง cleanup เอง - ถ้าลืมจะ memory leak!
    return () => {
      cancelAnimationFrame(animId);
      renderer.dispose();
      geometry.dispose();
      material.dispose();
      mountRef.current?.removeChild(renderer.domElement);
    };
  }, []);

  return <div ref={mountRef} />;
}
```

ปัญหาของวิธีนี้:
1. **Verbose มาก** - ต้องเขียนโค้ดเยอะเพื่อทำสิ่งง่ายๆ
2. **Cleanup ยาก** - ต้องระวัง memory leak เองทุกครั้ง
3. **Reactivity ซับซ้อน** - การ sync state กับ Three.js ทำได้ยาก
4. **แยก component ยาก** - โครงสร้าง scene กระจัดกระจาย

### ข้อดีของ R3F

```jsx
// ✅ แบบใหม่ - Declarative ด้วย R3F
import { Canvas, useFrame } from '@react-three/fiber';
import { useRef } from 'react';

function RotatingCube() {
  const meshRef = useRef();
  
  useFrame(() => {
    meshRef.current.rotation.x += 0.01;
    meshRef.current.rotation.y += 0.01;
  });

  return (
    <mesh ref={meshRef}>
      <boxGeometry />
      <meshBasicMaterial color="green" />
    </mesh>
  );
}

function App() {
  return (
    <Canvas>
      <RotatingCube />
    </Canvas>
  );
}
```

ข้อดี:
1. **กระชับ** - โค้ดน้อยกว่ามาก
2. **Declarative** - อ่านแล้วเข้าใจ structure ทันที
3. **Auto cleanup** - R3F จัดการ lifecycle เอง
4. **React state** - ใช้ useState/useEffect ได้ตรงๆ

---

## 64.2 การติดตั้ง

### สร้างโปรเจกต์ใหม่

```bash
# สร้าง React project ด้วย Vite
npm create vite@latest my-r3f-app -- --template react
cd my-r3f-app

# ติดตั้ง dependencies หลัก
npm install @react-three/fiber three

# ติดตั้ง TypeScript types (ถ้าใช้)
npm install -D @types/three

# ติดตั้ง helper libraries ที่มักใช้ร่วม
npm install @react-three/drei

# เริ่ม dev server
npm run dev
```

### โครงสร้างโปรเจกต์

```
my-r3f-app/
├── src/
│   ├── components/
│   │   ├── Scene.jsx        # Three.js scene components
│   │   ├── Cube.jsx         # Individual 3D objects
│   │   └── Controls.jsx     # Camera controls
│   ├── hooks/
│   │   └── useAnimation.js  # Custom animation hooks
│   ├── App.jsx              # Main app with Canvas
│   └── main.jsx
├── public/
│   └── models/              # GLTF/OBJ files
└── package.json
```

### ตรวจสอบ Peer Dependencies

```bash
# ตรวจสอบว่า three version compatible กับ @react-three/fiber
npm ls three
npm ls @react-three/fiber

# ควรได้ประมาณนี้:
# @react-three/fiber@8.x.x
# three@0.160.x
```

---

## 64.3 Canvas Component

Canvas เป็น entry point ของ R3F ที่สร้าง WebGL renderer, scene, และ camera ให้อัตโนมัติ

### Canvas พื้นฐาน

```jsx
// src/App.jsx
import { Canvas } from '@react-three/fiber';

function App() {
  return (
    // Canvas ต้องมี parent ที่กำหนด width/height
    <div style={{ width: '100vw', height: '100vh' }}>
      <Canvas>
        {/* Three.js components ไปที่นี่ */}
        <mesh>
          <boxGeometry />
          <meshBasicMaterial color="hotpink" />
        </mesh>
      </Canvas>
    </div>
  );
}
```

### Canvas Props ที่สำคัญ

```jsx
<Canvas
  // Camera settings
  camera={{ 
    position: [0, 0, 5],  // ตำแหน่ง camera
    fov: 75,               // Field of view
    near: 0.1,             // Near clipping plane
    far: 1000              // Far clipping plane
  }}
  
  // Renderer settings
  gl={{ 
    antialias: true,       // Anti-aliasing
    alpha: true,           // Transparent background
    powerPreference: 'high-performance'
  }}
  
  // Shadows
  shadows                  // เปิด shadow mapping
  shadowMap={{ type: THREE.PCFSoftShadowMap }}
  
  // Performance
  dpr={[1, 2]}            // Pixel ratio (min, max)
  frameloop="demand"       // 'always' | 'demand' | 'never'
  
  // Events
  onCreated={({ gl, scene, camera }) => {
    // เรียกเมื่อ renderer สร้างเสร็จ
    gl.setClearColor('#ffffff');
  }}
  
  // Resize
  resize={{ scroll: false, debounce: { scroll: 50, resize: 0 } }}
>
  {/* children */}
</Canvas>
```

### Canvas กับ Background

```jsx
// App.jsx
import { Canvas } from '@react-three/fiber';

function App() {
  return (
    <div style={{ 
      width: '100vw', 
      height: '100vh', 
      background: '#1a1a2e'  // Background อยู่ที่ CSS
    }}>
      <Canvas
        gl={{ alpha: true }}  // Canvas transparent
        style={{ background: 'transparent' }}
      >
        <Scene />
      </Canvas>
    </div>
  );
}

// หรือตั้ง background ใน Three.js scene
function Scene() {
  const { scene } = useThree();
  
  useEffect(() => {
    scene.background = new THREE.Color('#1a1a2e');
  }, [scene]);
  
  return null;
}
```

---

## 64.4 JSX Three.js Objects

R3F แปลง JSX เป็น Three.js objects โดยอัตโนมัติ

### กฎการแปลงชื่อ

```
Three.js class name  →  JSX tag (lowercase, camelCase)
THREE.BoxGeometry    →  <boxGeometry>
THREE.MeshStandardMaterial  →  <meshStandardMaterial>
THREE.PointLight     →  <pointLight>
THREE.Mesh           →  <mesh>
```

### Geometry Types

```jsx
function GeometryExamples() {
  return (
    <>
      {/* Box */}
      <mesh position={[-4, 0, 0]}>
        <boxGeometry args={[1, 1, 1]} />
        {/* args คือ constructor arguments */}
        <meshStandardMaterial color="royalblue" />
      </mesh>

      {/* Sphere */}
      <mesh position={[-2, 0, 0]}>
        <sphereGeometry args={[0.5, 32, 32]} />
        {/* radius, widthSegments, heightSegments */}
        <meshStandardMaterial color="crimson" />
      </mesh>

      {/* Cylinder */}
      <mesh position={[0, 0, 0]}>
        <cylinderGeometry args={[0.5, 0.5, 1, 32]} />
        {/* radiusTop, radiusBottom, height, radialSegments */}
        <meshStandardMaterial color="limegreen" />
      </mesh>

      {/* Torus */}
      <mesh position={[2, 0, 0]}>
        <torusGeometry args={[0.5, 0.2, 16, 100]} />
        {/* radius, tube, radialSegments, tubularSegments */}
        <meshStandardMaterial color="gold" />
      </mesh>

      {/* Plane */}
      <mesh position={[4, 0, 0]} rotation={[-Math.PI / 2, 0, 0]}>
        <planeGeometry args={[1, 1]} />
        {/* width, height */}
        <meshStandardMaterial color="violet" />
      </mesh>
    </>
  );
}
```

### Material Types

```jsx
function MaterialExamples() {
  return (
    <>
      {/* Basic - ไม่รับแสง */}
      <mesh position={[-3, 0, 0]}>
        <sphereGeometry args={[0.5, 32, 32]} />
        <meshBasicMaterial color="hotpink" wireframe />
      </mesh>

      {/* Lambert - Diffuse lighting */}
      <mesh position={[-1.5, 0, 0]}>
        <sphereGeometry args={[0.5, 32, 32]} />
        <meshLambertMaterial color="orange" />
      </mesh>

      {/* Phong - Specular highlights */}
      <mesh position={[0, 0, 0]}>
        <sphereGeometry args={[0.5, 32, 32]} />
        <meshPhongMaterial 
          color="skyblue" 
          shininess={100}
          specular="white"
        />
      </mesh>

      {/* Standard (PBR) */}
      <mesh position={[1.5, 0, 0]}>
        <sphereGeometry args={[0.5, 32, 32]} />
        <meshStandardMaterial 
          color="silver"
          metalness={0.9}
          roughness={0.1}
        />
      </mesh>

      {/* Physical (Full PBR) */}
      <mesh position={[3, 0, 0]}>
        <sphereGeometry args={[0.5, 32, 32]} />
        <meshPhysicalMaterial 
          color="gold"
          metalness={1}
          roughness={0.1}
          clearcoat={1}
          clearcoatRoughness={0.1}
        />
      </mesh>
    </>
  );
}
```

### Lights

```jsx
function LightingSetup() {
  return (
    <>
      {/* Ambient light - แสงสว่างทุกทิศ */}
      <ambientLight intensity={0.5} />

      {/* Directional light - แสงขนาน เหมือนแสงอาทิตย์ */}
      <directionalLight 
        position={[10, 10, 5]} 
        intensity={1}
        castShadow
        shadow-mapSize={[2048, 2048]}
      />

      {/* Point light - แสงจากจุด */}
      <pointLight 
        position={[-5, 5, 0]} 
        intensity={50}
        color="blue"
      />

      {/* Spot light - แสงสปอร์ตไลต์ */}
      <spotLight
        position={[0, 5, 0]}
        angle={0.3}
        penumbra={1}
        intensity={100}
        castShadow
      />

      {/* Hemisphere light - ท้องฟ้า/พื้น */}
      <hemisphereLight 
        skyColor="lightblue" 
        groundColor="brown" 
        intensity={0.5}
      />
    </>
  );
}
```

---

## 64.5 useFrame Hook

useFrame คือ heart ของ R3F animation - เรียกทุก frame (เหมือน requestAnimationFrame)

### useFrame พื้นฐาน

```jsx
import { useFrame } from '@react-three/fiber';
import { useRef } from 'react';

function AnimatedCube() {
  const meshRef = useRef();

  useFrame((state, delta) => {
    // state: { clock, camera, scene, gl, ... }
    // delta: เวลาที่ผ่านไปตั้งแต่ frame ที่แล้ว (วินาที)
    
    // หมุน cube
    meshRef.current.rotation.x += delta;  // frame-rate independent!
    meshRef.current.rotation.y += delta * 0.5;
    
    // ขยับตามเวลา
    const time = state.clock.elapsedTime;
    meshRef.current.position.y = Math.sin(time) * 2;
  });

  return (
    <mesh ref={meshRef}>
      <boxGeometry />
      <meshStandardMaterial color="royalblue" />
    </mesh>
  );
}
```

### useFrame Priority

```jsx
// priority ต่ำกว่า = รันก่อน (default = 0)
useFrame(() => {
  // รัน priority 0
}, 0);

useFrame(() => {
  // รัน priority 1 (หลัง priority 0)
}, 1);

// ใช้สำหรับ: รัน camera controls ก่อน แล้วค่อย render
useFrame(({ gl, scene, camera }) => {
  gl.render(scene, camera);
}, 1);
```

### useFrame กับ State

```jsx
import { useFrame } from '@react-three/fiber';
import { useRef, useState } from 'react';

function InteractiveMesh() {
  const meshRef = useRef();
  const [hovered, setHovered] = useState(false);
  const targetScale = useRef(1);

  // อัพเดต target เมื่อ hover state เปลี่ยน
  useEffect(() => {
    targetScale.current = hovered ? 1.5 : 1;
  }, [hovered]);

  useFrame((state, delta) => {
    if (!meshRef.current) return;
    
    // Smooth interpolation ไปยัง target scale
    meshRef.current.scale.lerp(
      { x: targetScale.current, y: targetScale.current, z: targetScale.current },
      0.1
    );
    
    // หรือใช้ lerp scalar
    meshRef.current.scale.x += (targetScale.current - meshRef.current.scale.x) * 0.1;
  });

  return (
    <mesh
      ref={meshRef}
      onPointerOver={() => setHovered(true)}
      onPointerOut={() => setHovered(false)}
    >
      <boxGeometry />
      <meshStandardMaterial color={hovered ? 'hotpink' : 'royalblue'} />
    </mesh>
  );
}
```

---

## 64.6 useThree Hook

useThree ให้เข้าถึง Three.js internals ได้จากภายใน Canvas

```jsx
import { useThree } from '@react-three/fiber';

function SceneInspector() {
  const {
    scene,      // THREE.Scene
    camera,     // THREE.Camera
    gl,         // THREE.WebGLRenderer
    size,       // { width, height }
    viewport,   // { width, height, factor }
    clock,      // THREE.Clock
    raycaster,  // THREE.Raycaster
    mouse,      // THREE.Vector2 (normalized)
    events,     // Event system
    set,        // Function to update state
    get,        // Function to get current state
    invalidate, // Force re-render (for frameloop="demand")
    advance,    // Advance frame manually
  } = useThree();

  useEffect(() => {
    console.log('Scene size:', size.width, 'x', size.height);
    console.log('Camera FOV:', camera.fov);
    console.log('Renderer capabilities:', gl.capabilities);
  }, [size, camera, gl]);

  return null;
}
```

### ใช้ useThree สำหรับ Responsive Scene

```jsx
function ResponsiveCamera() {
  const { size, camera } = useThree();

  useEffect(() => {
    // ปรับ camera aspect ratio เมื่อ size เปลี่ยน
    if (camera.isPerspectiveCamera) {
      camera.aspect = size.width / size.height;
      camera.updateProjectionMatrix();
    }
  }, [size, camera]);

  return null;
}

function ResponsiveObject() {
  const { viewport } = useThree();

  return (
    <mesh scale={viewport.width / 5}>
      {/* ขนาด object ตาม viewport */}
      <boxGeometry />
      <meshStandardMaterial />
    </mesh>
  );
}
```

### ปรับ Camera จากนอก Canvas

```jsx
// ✅ ใช้ useThree ใน component ที่อยู่ใน Canvas
function CameraController({ position }) {
  const { camera } = useThree();
  
  useEffect(() => {
    camera.position.set(...position);
    camera.lookAt(0, 0, 0);
  }, [position, camera]);
  
  return null;
}

// App.jsx
function App() {
  const [cameraPos, setCameraPos] = useState([0, 5, 10]);
  
  return (
    <Canvas>
      <CameraController position={cameraPos} />
      <Scene />
    </Canvas>
  );
}
```

---

## 64.7 Refs กับ Three.js Objects

### การใช้ ref เพื่อเข้าถึง Three.js Object โดยตรง

```jsx
import { useRef, useEffect } from 'react';

function MeshWithRef() {
  const meshRef = useRef();
  const geometryRef = useRef();
  const materialRef = useRef();

  useEffect(() => {
    // เข้าถึง Three.js objects หลัง mount
    if (meshRef.current) {
      console.log(meshRef.current);           // THREE.Mesh
      console.log(geometryRef.current);       // THREE.BoxGeometry
      console.log(materialRef.current);       // THREE.MeshStandardMaterial
      
      // ใช้ Three.js methods โดยตรง
      meshRef.current.lookAt(0, 0, 0);
      materialRef.current.needsUpdate = true;
    }
  }, []);

  return (
    <mesh ref={meshRef}>
      <boxGeometry ref={geometryRef} />
      <meshStandardMaterial ref={materialRef} color="royalblue" />
    </mesh>
  );
}
```

### Imperative Animation ด้วย ref

```jsx
import { useRef, useImperativeHandle, forwardRef } from 'react';
import { useFrame } from '@react-three/fiber';

// Component ที่ expose methods ไปยัง parent
const AnimatedObject = forwardRef(function AnimatedObject(props, ref) {
  const meshRef = useRef();
  const isAnimating = useRef(false);

  // Expose methods ผ่าน ref
  useImperativeHandle(ref, () => ({
    startAnimation: () => { isAnimating.current = true; },
    stopAnimation: () => { isAnimating.current = false; },
    reset: () => {
      meshRef.current.rotation.set(0, 0, 0);
      meshRef.current.position.set(0, 0, 0);
    }
  }));

  useFrame((state, delta) => {
    if (isAnimating.current && meshRef.current) {
      meshRef.current.rotation.y += delta;
    }
  });

  return (
    <mesh ref={meshRef} {...props}>
      <boxGeometry />
      <meshStandardMaterial color="orange" />
    </mesh>
  );
});

// Parent component
function Scene() {
  const objectRef = useRef();

  return (
    <>
      <AnimatedObject ref={objectRef} />
      {/* ควบคุมจากข้างนอก */}
      <Html>
        <button onClick={() => objectRef.current?.startAnimation()}>
          Start
        </button>
      </Html>
    </>
  );
}
```

---

## 64.8 Events ใน R3F

R3F มี built-in pointer events ที่ทำงานเหมือน DOM events

```jsx
function InteractiveObject() {
  const [color, setColor] = useState('royalblue');
  const [active, setActive] = useState(false);

  return (
    <mesh
      // Mouse/Pointer events
      onClick={(e) => {
        e.stopPropagation(); // หยุดไม่ให้ event bubble
        setActive(!active);
        console.log('clicked at:', e.point);  // THREE.Vector3
      }}
      
      onContextMenu={(e) => {
        console.log('right clicked');
      }}
      
      onDoubleClick={(e) => {
        console.log('double clicked');
      }}
      
      onPointerOver={(e) => {
        setColor('hotpink');
        document.body.style.cursor = 'pointer';
      }}
      
      onPointerOut={(e) => {
        setColor('royalblue');
        document.body.style.cursor = 'default';
      }}
      
      onPointerEnter={(e) => {
        // เหมือน onPointerOver แต่ไม่ bubble
      }}
      
      onPointerLeave={(e) => {
        // เหมือน onPointerOut แต่ไม่ bubble
      }}
      
      onPointerMove={(e) => {
        // ขณะเลื่อน pointer บน object
        console.log('position:', e.point);
      }}
      
      onPointerDown={(e) => {
        console.log('pointer down');
      }}
      
      onPointerUp={(e) => {
        console.log('pointer up');
      }}
      
      // Scale เมื่อ active
      scale={active ? 1.5 : 1}
    >
      <boxGeometry />
      <meshStandardMaterial color={color} />
    </mesh>
  );
}
```

### Event Object Properties

```jsx
function DetailedEventHandling() {
  return (
    <mesh
      onClick={(event) => {
        // event properties:
        const {
          object,       // THREE.Object3D ที่ถูกคลิก
          point,        // THREE.Vector3 - จุดที่ถูกคลิกใน world space
          face,         // THREE.Face3 - หน้าที่ถูกคลิก
          faceIndex,    // index ของหน้า
          distance,     // ระยะจาก camera ถึง intersection
          uv,           // THREE.Vector2 - UV coordinates
          ray,          // THREE.Ray
          camera,       // THREE.Camera ที่ใช้
          nativeEvent,  // Native DOM event
          intersections, // Array ของ intersections ทั้งหมด
        } = event;
        
        console.log('Hit point:', point);
        console.log('Object name:', object.name);
        console.log('UV:', uv);
      }}
    >
      <boxGeometry />
      <meshStandardMaterial color="royalblue" />
    </mesh>
  );
}
```

---

## 64.9 Scene Setup ใน React

### Complete Scene Component

```jsx
// src/components/Scene.jsx
import { useThree, useFrame } from '@react-three/fiber';
import { useRef } from 'react';
import * as THREE from 'three';

function Floor() {
  return (
    <mesh rotation={[-Math.PI / 2, 0, 0]} position={[0, -2, 0]} receiveShadow>
      <planeGeometry args={[20, 20]} />
      <meshStandardMaterial color="#f0f0f0" />
    </mesh>
  );
}

function Lights() {
  return (
    <>
      <ambientLight intensity={0.5} />
      <directionalLight
        position={[5, 10, 5]}
        intensity={1}
        castShadow
        shadow-mapSize-width={2048}
        shadow-mapSize-height={2048}
        shadow-camera-near={0.1}
        shadow-camera-far={50}
        shadow-camera-left={-10}
        shadow-camera-right={10}
        shadow-camera-top={10}
        shadow-camera-bottom={-10}
      />
    </>
  );
}

function Box({ position, color, speed = 1 }) {
  const ref = useRef();

  useFrame((state, delta) => {
    ref.current.rotation.x += delta * speed;
    ref.current.rotation.y += delta * speed * 0.5;
  });

  return (
    <mesh ref={ref} position={position} castShadow>
      <boxGeometry args={[1, 1, 1]} />
      <meshStandardMaterial color={color} metalness={0.3} roughness={0.4} />
    </mesh>
  );
}

export function Scene() {
  return (
    <>
      <Lights />
      <Floor />
      <Box position={[-2, 0, 0]} color="royalblue" speed={1} />
      <Box position={[0, 0, 0]} color="crimson" speed={1.5} />
      <Box position={[2, 0, 0]} color="limegreen" speed={0.7} />
    </>
  );
}
```

### App.jsx หลัก

```jsx
// src/App.jsx
import { Canvas } from '@react-three/fiber';
import { Scene } from './components/Scene';
import './App.css';

function App() {
  return (
    <div className="canvas-container">
      <Canvas
        shadows
        camera={{ position: [0, 5, 10], fov: 60 }}
        gl={{ antialias: true }}
      >
        <color attach="background" args={['#1a1a2e']} />
        <fog attach="fog" args={['#1a1a2e', 10, 50]} />
        <Scene />
      </Canvas>
    </div>
  );
}

export default App;
```

```css
/* App.css */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

.canvas-container {
  width: 100vw;
  height: 100vh;
}
```

---

## 64.10 R3F vs Vanilla Three.js เปรียบเทียบ

### การสร้าง Scene เดียวกัน

**Vanilla Three.js:**

```javascript
// vanilla.js - ~60 lines สำหรับ basic scene
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color('#1a1a2e');

const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(0, 5, 10);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);

const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
scene.add(ambientLight);

const dirLight = new THREE.DirectionalLight(0xffffff, 1);
dirLight.position.set(5, 10, 5);
dirLight.castShadow = true;
scene.add(dirLight);

const geometry = new THREE.BoxGeometry();
const material = new THREE.MeshStandardMaterial({ color: 'royalblue' });
const cube = new THREE.Mesh(geometry, material);
cube.castShadow = true;
scene.add(cube);

const floorGeo = new THREE.PlaneGeometry(20, 20);
const floorMat = new THREE.MeshStandardMaterial({ color: '#f0f0f0' });
const floor = new THREE.Mesh(floorGeo, floorMat);
floor.rotation.x = -Math.PI / 2;
floor.position.y = -2;
floor.receiveShadow = true;
scene.add(floor);

// Resize handler
window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});

// Animation loop
const clock = new THREE.Clock();
function animate() {
  requestAnimationFrame(animate);
  const delta = clock.getDelta();
  cube.rotation.x += delta;
  cube.rotation.y += delta * 0.5;
  controls.update();
  renderer.render(scene, camera);
}
animate();
```

**R3F เทียบเท่า:**

```jsx
// r3f.jsx - ~35 lines สำหรับ scene เดียวกัน
import { Canvas, useFrame } from '@react-three/fiber';
import { OrbitControls } from '@react-three/drei';
import { useRef } from 'react';

function Cube() {
  const ref = useRef();
  useFrame((_, delta) => {
    ref.current.rotation.x += delta;
    ref.current.rotation.y += delta * 0.5;
  });
  return (
    <mesh ref={ref} castShadow>
      <boxGeometry />
      <meshStandardMaterial color="royalblue" />
    </mesh>
  );
}

export default function App() {
  return (
    <Canvas shadows camera={{ position: [0, 5, 10], fov: 60 }}>
      <color attach="background" args={['#1a1a2e']} />
      <ambientLight intensity={0.5} />
      <directionalLight position={[5, 10, 5]} intensity={1} castShadow />
      <Cube />
      <mesh rotation={[-Math.PI/2, 0, 0]} position={[0,-2,0]} receiveShadow>
        <planeGeometry args={[20, 20]} />
        <meshStandardMaterial color="#f0f0f0" />
      </mesh>
      <OrbitControls />
    </Canvas>
  );
}
```

### เปรียบเทียบข้อดี/ข้อเสีย

| Feature | Vanilla Three.js | React Three Fiber |
|---------|-----------------|-------------------|
| Learning curve | ต่ำกว่า (ไม่ต้องรู้ React) | สูงกว่า (ต้องรู้ React + Three.js) |
| Code structure | Imperative | Declarative |
| State management | ยาก | ง่าย (React state) |
| Performance | ดีกว่าเล็กน้อย | เกือบเท่ากัน |
| Ecosystem | Three.js only | React + Three.js |
| Re-renders | ควบคุมเอง | อัตโนมัติ (ต้อง optimize) |
| Testing | ยาก | ง่ายกว่า (React Testing Library) |
| TypeScript | ดี | ดีมาก |

---

## 64.11 โปรเจกต์: Spinning Cube ใน React

```jsx
// src/SpinningCubeDemo.jsx
import { Canvas, useFrame, useThree } from '@react-three/fiber';
import { useRef, useState, useCallback } from 'react';
import * as THREE from 'three';

// Custom hook สำหรับ hover detection
function useHover() {
  const [hovered, setHovered] = useState(false);
  
  const onPointerOver = useCallback(() => setHovered(true), []);
  const onPointerOut = useCallback(() => setHovered(false), []);
  
  return { hovered, onPointerOver, onPointerOut };
}

// Spinning Cube Component
function SpinningCube({ position = [0, 0, 0] }) {
  const meshRef = useRef();
  const { hovered, onPointerOver, onPointerOut } = useHover();
  const [clicked, setClicked] = useState(false);
  
  // Animation
  useFrame((state, delta) => {
    const mesh = meshRef.current;
    if (!mesh) return;
    
    // หมุนเสมอ แต่เร็วกว่าเมื่อ hover
    const speed = hovered ? 3 : 1;
    mesh.rotation.x += delta * speed;
    mesh.rotation.y += delta * speed * 0.7;
    
    // Bobbing animation
    const time = state.clock.elapsedTime;
    mesh.position.y = Math.sin(time * 2) * 0.3;
  });

  return (
    <mesh
      ref={meshRef}
      position={position}
      scale={clicked ? 1.3 : 1}
      onClick={() => setClicked(!clicked)}
      onPointerOver={onPointerOver}
      onPointerOut={onPointerOut}
    >
      <boxGeometry args={[1.5, 1.5, 1.5]} />
      <meshStandardMaterial
        color={clicked ? '#ff6b6b' : hovered ? '#4ecdc4' : '#45b7d1'}
        metalness={0.3}
        roughness={0.4}
        wireframe={false}
      />
    </mesh>
  );
}

// Background Stars
function Stars({ count = 200 }) {
  const positions = new Float32Array(count * 3);
  
  for (let i = 0; i < count; i++) {
    positions[i * 3] = (Math.random() - 0.5) * 50;
    positions[i * 3 + 1] = (Math.random() - 0.5) * 50;
    positions[i * 3 + 2] = (Math.random() - 0.5) * 50;
  }

  return (
    <points>
      <bufferGeometry>
        <bufferAttribute
          attach="attributes-position"
          array={positions}
          count={count}
          itemSize={3}
        />
      </bufferGeometry>
      <pointsMaterial size={0.1} color="white" />
    </points>
  );
}

// HUD แสดง info
function InfoText() {
  return (
    <div style={{
      position: 'absolute',
      top: 20,
      left: 20,
      color: 'white',
      fontFamily: 'monospace',
      fontSize: '14px',
      pointerEvents: 'none'
    }}>
      <p>🎮 Click cube to toggle color</p>
      <p>🖱️ Hover to speed up</p>
    </div>
  );
}

// Main App
export default function SpinningCubeDemo() {
  return (
    <div style={{ width: '100vw', height: '100vh', position: 'relative' }}>
      <InfoText />
      <Canvas camera={{ position: [0, 0, 5], fov: 60 }}>
        <color attach="background" args={['#0a0a1a']} />
        
        {/* Lighting */}
        <ambientLight intensity={0.3} />
        <pointLight position={[5, 5, 5]} intensity={100} color="#4ecdc4" />
        <pointLight position={[-5, -5, -5]} intensity={50} color="#ff6b6b" />
        
        {/* Stars */}
        <Stars count={300} />
        
        {/* Multiple Cubes */}
        <SpinningCube position={[0, 0, 0]} />
        <SpinningCube position={[-3, 0, -2]} />
        <SpinningCube position={[3, 0, -2]} />
      </Canvas>
    </div>
  );
}
```

---

## 64.12 โปรเจกต์: Interactive R3F Scene

```jsx
// src/InteractiveScene.jsx
import { Canvas, useFrame, useThree } from '@react-three/fiber';
import { useRef, useState, useMemo } from 'react';
import * as THREE from 'three';

// Component สำหรับแต่ละ object ใน scene
function InteractiveObject({ 
  position, 
  color, 
  shape = 'box',
  onSelect,
  isSelected 
}) {
  const meshRef = useRef();
  const [hovered, setHovered] = useState(false);
  
  useFrame((state, delta) => {
    if (!meshRef.current) return;
    
    // หมุนเมื่อ selected
    if (isSelected) {
      meshRef.current.rotation.y += delta * 2;
    }
    
    // Float animation
    const time = state.clock.elapsedTime;
    const id = position[0]; // ใช้ position เป็น unique id สำหรับ offset
    meshRef.current.position.y = position[1] + Math.sin(time + id) * 0.2;
    
    // Scale เมื่อ hover หรือ selected
    const targetScale = isSelected ? 1.4 : hovered ? 1.2 : 1;
    meshRef.current.scale.lerp(
      new THREE.Vector3(targetScale, targetScale, targetScale),
      0.1
    );
  });

  // สร้าง geometry ตาม shape prop
  const geometryNode = useMemo(() => {
    switch(shape) {
      case 'sphere':
        return <sphereGeometry args={[0.6, 32, 32]} />;
      case 'torus':
        return <torusGeometry args={[0.5, 0.2, 16, 100]} />;
      case 'octahedron':
        return <octahedronGeometry args={[0.7]} />;
      default:
        return <boxGeometry args={[1, 1, 1]} />;
    }
  }, [shape]);

  return (
    <mesh
      ref={meshRef}
      position={position}
      onClick={(e) => {
        e.stopPropagation();
        onSelect();
      }}
      onPointerOver={() => {
        setHovered(true);
        document.body.style.cursor = 'pointer';
      }}
      onPointerOut={() => {
        setHovered(false);
        document.body.style.cursor = 'default';
      }}
    >
      {geometryNode}
      <meshStandardMaterial
        color={isSelected ? 'gold' : hovered ? '#ff6b6b' : color}
        emissive={isSelected ? 'gold' : 'black'}
        emissiveIntensity={isSelected ? 0.3 : 0}
        metalness={0.3}
        roughness={0.4}
      />
    </mesh>
  );
}

// Grid ของ objects
function ObjectGrid({ selectedId, onSelect }) {
  const objects = useMemo(() => [
    { id: 0, position: [-3, 0, 0], color: '#e74c3c', shape: 'box' },
    { id: 1, position: [-1.5, 0, 0], color: '#3498db', shape: 'sphere' },
    { id: 2, position: [0, 0, 0], color: '#2ecc71', shape: 'torus' },
    { id: 3, position: [1.5, 0, 0], color: '#f39c12', shape: 'octahedron' },
    { id: 4, position: [3, 0, 0], color: '#9b59b6', shape: 'box' },
  ], []);

  return (
    <>
      {objects.map(obj => (
        <InteractiveObject
          key={obj.id}
          position={obj.position}
          color={obj.color}
          shape={obj.shape}
          isSelected={selectedId === obj.id}
          onSelect={() => onSelect(obj.id)}
        />
      ))}
    </>
  );
}

// Ground Plane
function Ground() {
  return (
    <mesh rotation={[-Math.PI / 2, 0, 0]} position={[0, -1.5, 0]} receiveShadow>
      <planeGeometry args={[20, 20]} />
      <meshStandardMaterial color="#1a1a2e" />
    </mesh>
  );
}

// Ambient Particles
function Particles({ count = 100 }) {
  const ref = useRef();
  
  const positions = useMemo(() => {
    const arr = new Float32Array(count * 3);
    for (let i = 0; i < count; i++) {
      arr[i * 3] = (Math.random() - 0.5) * 20;
      arr[i * 3 + 1] = (Math.random() - 0.5) * 10;
      arr[i * 3 + 2] = (Math.random() - 0.5) * 20;
    }
    return arr;
  }, [count]);

  useFrame((state) => {
    if (!ref.current) return;
    ref.current.rotation.y = state.clock.elapsedTime * 0.05;
  });

  return (
    <points ref={ref}>
      <bufferGeometry>
        <bufferAttribute
          attach="attributes-position"
          array={positions}
          count={count}
          itemSize={3}
        />
      </bufferGeometry>
      <pointsMaterial size={0.05} color="#4ecdc4" transparent opacity={0.6} />
    </points>
  );
}

// Info Panel (HTML overlay)
function InfoPanel({ selectedId }) {
  const objects = [
    { name: 'Red Box', description: 'Simple cube geometry' },
    { name: 'Blue Sphere', description: 'Smooth sphere with 32 segments' },
    { name: 'Green Torus', description: 'Donut-shaped geometry' },
    { name: 'Orange Octahedron', description: '8-sided polyhedron' },
    { name: 'Purple Box', description: 'Another cube, but purple!' },
  ];

  return (
    <div style={{
      position: 'absolute',
      bottom: 20,
      left: '50%',
      transform: 'translateX(-50%)',
      background: 'rgba(0,0,0,0.7)',
      color: 'white',
      padding: '15px 25px',
      borderRadius: 10,
      fontFamily: 'sans-serif',
      textAlign: 'center',
      minWidth: 200,
      backdropFilter: 'blur(10px)'
    }}>
      {selectedId !== null ? (
        <>
          <h3 style={{ margin: 0, color: 'gold' }}>
            {objects[selectedId]?.name}
          </h3>
          <p style={{ margin: '5px 0 0' }}>
            {objects[selectedId]?.description}
          </p>
        </>
      ) : (
        <p style={{ margin: 0 }}>Click an object to select it</p>
      )}
    </div>
  );
}

// Main Interactive Scene
export default function InteractiveScene() {
  const [selectedId, setSelectedId] = useState(null);

  const handleSelect = (id) => {
    setSelectedId(prev => prev === id ? null : id);
  };

  return (
    <div style={{ width: '100vw', height: '100vh', position: 'relative' }}>
      <InfoPanel selectedId={selectedId} />
      
      <Canvas 
        shadows 
        camera={{ position: [0, 3, 8], fov: 60 }}
        onClick={() => setSelectedId(null)}
      >
        <color attach="background" args={['#0a0a1a']} />
        <fog attach="fog" args={['#0a0a1a', 15, 40]} />
        
        {/* Lighting */}
        <ambientLight intensity={0.4} />
        <directionalLight position={[5, 10, 5]} intensity={1} castShadow />
        <pointLight position={[0, 5, 0]} intensity={50} color="#4ecdc4" />
        
        {/* Scene Content */}
        <ObjectGrid selectedId={selectedId} onSelect={handleSelect} />
        <Ground />
        <Particles count={150} />
      </Canvas>
    </div>
  );
}
```

---

## 64.13 Declarative vs Imperative เปรียบเทียบเชิงลึก

### Declarative (R3F)

```jsx
// โค้ดบอกว่า "อยากได้อะไร" ไม่ใช่ "ทำอะไรยังไง"
function Scene() {
  const [size, setSize] = useState(1);
  
  return (
    // React จัดการ re-render และ update ให้เอง
    <mesh scale={size}>
      <boxGeometry />
      <meshStandardMaterial color={size > 1.5 ? 'red' : 'blue'} />
    </mesh>
  );
}
```

### Imperative (Vanilla)

```javascript
// โค้ดบอกว่า "ทำอะไรยังไง" step by step
const mesh = new THREE.Mesh(geometry, material);
scene.add(mesh);

// ต้องจัดการ update เอง
function updateSize(newSize) {
  mesh.scale.set(newSize, newSize, newSize);
  if (newSize > 1.5) {
    material.color.set('red');
  } else {
    material.color.set('blue');
  }
}
```

### เมื่อไหร่ควรใช้อะไร

**ใช้ R3F เมื่อ:**
- โปรเจกต์ใช้ React อยู่แล้ว
- ต้องการ sync 3D scene กับ UI state
- ทีมมี React experience
- ต้องการ rapid development

**ใช้ Vanilla Three.js เมื่อ:**
- ไม่ใช้ React
- ต้องการ maximum performance
- โปรเจกต์เล็กๆ ไม่ซับซ้อน
- ต้องการ control ทุกอย่างเอง

---

## 64.14 Performance Considerations

### ปัญหา Re-render

```jsx
// ❌ BAD - สร้าง geometry ใหม่ทุก render
function BadComponent() {
  return (
    <mesh>
      <boxGeometry args={[new Date().getTime()]} />  {/* สร้างใหม่ทุกครั้ง! */}
    </mesh>
  );
}

// ✅ GOOD - ใช้ useMemo
function GoodComponent() {
  const geometry = useMemo(() => new THREE.BoxGeometry(1, 1, 1), []);
  
  return (
    <mesh geometry={geometry}>
      <meshStandardMaterial />
    </mesh>
  );
}
```

### frameloop="demand"

```jsx
// ใช้เมื่อ scene ไม่เคลื่อนไหวตลอดเวลา
function StaticScene() {
  const { invalidate } = useThree();
  
  const handleChange = () => {
    invalidate();  // บอก R3F ให้ render frame ถัดไป
  };

  return (
    <mesh onClick={handleChange}>
      <boxGeometry />
      <meshStandardMaterial />
    </mesh>
  );
}

function App() {
  return (
    <Canvas frameloop="demand">  {/* render เฉพาะเมื่อจำเป็น */}
      <StaticScene />
    </Canvas>
  );
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Basic R3F Setup
สร้าง React app ด้วย R3F ที่มี:
- Canvas พร้อม custom camera
- Sphere ที่หมุนรอบตัวเอง
- ChangeColor เมื่อ hover
- แสดง coordinate ปัจจุบันบน HUD

### แบบฝึกหัดที่ 2: Multiple Objects
สร้าง scene ที่มี:
- Box, Sphere, Cylinder ในตำแหน่งต่างกัน
- แต่ละ object มีสีและความเร็วหมุนต่างกัน
- เมื่อคลิก object ให้แสดง info panel
- Background ที่เปลี่ยนสีตาม object ที่ selected

### แบบฝึกหัดที่ 3: State Sync
สร้าง UI panel ที่:
- มี slider ควบคุม rotation speed
- มี color picker สำหรับเปลี่ยนสี material
- มี checkbox toggle wireframe
- มี button reset ตำแหน่ง
- ทุก UI element sync กับ 3D scene แบบ real-time

### แบบฝึกหัดที่ 4: Camera Animation
สร้าง camera tour ที่:
- เดินทางผ่าน waypoints ต่างๆ
- Smooth transition ระหว่าง camera positions
- มี play/pause/next/prev controls
- แสดง minimap บอกตำแหน่ง camera

### แบบฝึกหัดที่ 5: Particle System
สร้าง particle system ใน R3F:
- 1000 particles ในรูป sphere
- Animate ด้วย useFrame
- Interactive: คลิกสร้าง explosion effect
- ใช้ useMemo สำหรับ geometry และ material

---

## สรุป

ในบทนี้เราเรียนรู้:

1. **R3F คืออะไร** - React renderer สำหรับ Three.js ที่ทำให้เขียน 3D แบบ declarative
2. **Canvas Component** - Entry point ที่ตั้งค่า renderer, scene, camera
3. **JSX Syntax** - วิธีเขียน Three.js objects ใน JSX
4. **useFrame** - Animation loop ที่ทำงานทุก frame
5. **useThree** - เข้าถึง Three.js internals
6. **Events** - Pointer events บน 3D objects
7. **R3F vs Vanilla** - เมื่อไหร่ควรใช้อะไร

บทถัดไปเราจะเรียน **React Three Fiber Advanced** ซึ่งรวมถึง performance optimization, useLoader, Suspense, animation libraries และอื่นๆ
