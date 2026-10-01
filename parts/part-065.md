# Part 65: React Three Fiber Advanced

## บทนำ

หลังจากเรียนรู้พื้นฐาน R3F แล้ว ในบทนี้เราจะเจาะลึกเรื่อง performance optimization, การโหลด assets แบบ async, animation libraries, และ advanced patterns ที่ใช้ในโปรเจกต์จริง

---

## 65.1 Performance Optimization ด้วย useMemo

### ปัญหาการสร้าง Object ซ้ำ

```jsx
// ❌ BAD - สร้าง geometry/material ใหม่ทุก render cycle
function PoorlyOptimized({ count = 1000 }) {
  return Array.from({ length: count }).map((_, i) => (
    <mesh key={i} position={[i, 0, 0]}>
      {/* สร้าง BufferGeometry ใหม่ทุก render! */}
      <boxGeometry args={[0.5, 0.5, 0.5]} />
      <meshStandardMaterial color="blue" />
    </mesh>
  ));
}
```

### แก้ด้วย useMemo

```jsx
import { useMemo, useRef } from 'react';
import { useFrame } from '@react-three/fiber';
import * as THREE from 'three';

// ✅ GOOD - share geometry และ material
function OptimizedInstances({ count = 1000 }) {
  // สร้างครั้งเดียว, share กันทุก instance
  const geometry = useMemo(() => new THREE.BoxGeometry(0.5, 0.5, 0.5), []);
  const material = useMemo(() => new THREE.MeshStandardMaterial({ color: 'blue' }), []);
  
  // สร้าง positions ครั้งเดียว
  const positions = useMemo(() => 
    Array.from({ length: count }, (_, i) => [
      (Math.random() - 0.5) * 20,
      (Math.random() - 0.5) * 20,
      (Math.random() - 0.5) * 20,
    ]), [count]
  );

  return positions.map((position, i) => (
    <mesh key={i} position={position} geometry={geometry} material={material} />
  ));
}
```

### InstancedMesh สำหรับ Maximum Performance

```jsx
import { useRef, useMemo } from 'react';
import { useFrame } from '@react-three/fiber';
import * as THREE from 'three';

function InstancedCubes({ count = 10000 }) {
  const meshRef = useRef();
  
  // Pre-compute positions
  const dummy = useMemo(() => new THREE.Object3D(), []);
  const initialPositions = useMemo(() => 
    Array.from({ length: count }, () => ({
      x: (Math.random() - 0.5) * 50,
      y: (Math.random() - 0.5) * 50,
      z: (Math.random() - 0.5) * 50,
      rotSpeed: Math.random() * 2 - 1
    })), [count]
  );

  useFrame((state) => {
    if (!meshRef.current) return;
    const time = state.clock.elapsedTime;
    
    initialPositions.forEach((pos, i) => {
      // อัพเดตแต่ละ instance
      dummy.position.set(pos.x, pos.y, pos.z);
      dummy.rotation.set(
        time * pos.rotSpeed,
        time * pos.rotSpeed * 0.5,
        0
      );
      dummy.scale.setScalar(Math.sin(time + i * 0.1) * 0.3 + 0.7);
      dummy.updateMatrix();
      
      meshRef.current.setMatrixAt(i, dummy.matrix);
    });
    
    meshRef.current.instanceMatrix.needsUpdate = true;
  });

  return (
    <instancedMesh ref={meshRef} args={[null, null, count]}>
      <boxGeometry args={[0.2, 0.2, 0.2]} />
      <meshStandardMaterial color="#4ecdc4" metalness={0.5} roughness={0.3} />
    </instancedMesh>
  );
}
```

---

## 65.2 useLoader Hook

useLoader ใช้สำหรับโหลด assets ต่างๆ และ cache ผลลัพธ์

### โหลด Texture

```jsx
import { useLoader } from '@react-three/fiber';
import { TextureLoader } from 'three';

function TexturedMesh() {
  // โหลด texture (cached อัตโนมัติ)
  const texture = useLoader(TextureLoader, '/textures/earth.jpg');
  
  return (
    <mesh>
      <sphereGeometry args={[2, 64, 64]} />
      <meshStandardMaterial map={texture} />
    </mesh>
  );
}
```

### โหลด Multiple Textures

```jsx
function PBRMesh() {
  const [
    colorMap,
    normalMap,
    roughnessMap,
    aoMap
  ] = useLoader(TextureLoader, [
    '/textures/rock/color.jpg',
    '/textures/rock/normal.jpg',
    '/textures/rock/roughness.jpg',
    '/textures/rock/ao.jpg',
  ]);

  return (
    <mesh>
      <sphereGeometry args={[1, 64, 64]} />
      <meshStandardMaterial
        map={colorMap}
        normalMap={normalMap}
        roughnessMap={roughnessMap}
        aoMap={aoMap}
      />
    </mesh>
  );
}
```

### โหลด GLTF Model

```jsx
import { useLoader } from '@react-three/fiber';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js';

function GLTFModel({ url }) {
  const gltf = useLoader(GLTFLoader, url, (loader) => {
    // Configure DRACOLoader สำหรับ compressed models
    const dracoLoader = new DRACOLoader();
    dracoLoader.setDecoderPath('/draco/');
    loader.setDRACOLoader(dracoLoader);
  });

  return <primitive object={gltf.scene} />;
}

// ใช้งาน
function App() {
  return (
    <Canvas>
      <Suspense fallback={<LoadingBox />}>
        <GLTFModel url="/models/helmet.glb" />
      </Suspense>
    </Canvas>
  );
}
```

### Custom Loader

```jsx
import { useLoader } from '@react-three/fiber';
import { FileLoader } from 'three';

// Custom loader สำหรับ JSON data
class JSONLoader extends FileLoader {
  load(url, onLoad, onProgress, onError) {
    super.load(url, (data) => {
      onLoad(JSON.parse(data));
    }, onProgress, onError);
  }
}

function CustomDataMesh() {
  const data = useLoader(JSONLoader, '/data/mesh-data.json');
  
  // ใช้ data สร้าง geometry
  const geometry = useMemo(() => {
    const geo = new THREE.BufferGeometry();
    // ... build geometry from data
    return geo;
  }, [data]);

  return <mesh geometry={geometry} />;
}
```

---

## 65.3 Suspense สำหรับ Async Loading

Suspense ใน R3F ทำงานเหมือน React Suspense ปกติ

### Loading States

```jsx
import { Suspense, useState } from 'react';
import { Canvas, useLoader } from '@react-three/fiber';
import { TextureLoader } from 'three';

// Loading placeholder
function LoadingSpinner() {
  const ref = useRef();
  useFrame((_, delta) => {
    if (ref.current) ref.current.rotation.y += delta * 3;
  });
  
  return (
    <mesh ref={ref}>
      <torusGeometry args={[1, 0.2, 16, 100]} />
      <meshBasicMaterial color="#4ecdc4" wireframe />
    </mesh>
  );
}

// Component ที่ต้องโหลด
function HeavyModel() {
  const texture = useLoader(TextureLoader, '/large-texture.jpg');
  const gltf = useLoader(GLTFLoader, '/large-model.glb');
  
  return (
    <group>
      <primitive object={gltf.scene} />
    </group>
  );
}

// App
function App() {
  return (
    <Canvas>
      <Suspense fallback={<LoadingSpinner />}>
        <HeavyModel />
      </Suspense>
    </Canvas>
  );
}
```

### Progress Tracking ด้วย useProgress

```jsx
import { useProgress, Html } from '@react-three/drei';

function Loader() {
  const { progress, active, loaded, total, item, errors } = useProgress();
  
  return (
    <Html center>
      <div style={{
        display: 'flex',
        flexDirection: 'column',
        alignItems: 'center',
        color: 'white',
        fontFamily: 'monospace'
      }}>
        <div style={{
          width: 200,
          height: 4,
          background: '#333',
          borderRadius: 2,
          overflow: 'hidden'
        }}>
          <div style={{
            width: `${progress}%`,
            height: '100%',
            background: '#4ecdc4',
            transition: 'width 0.3s'
          }} />
        </div>
        <p>{Math.round(progress)}% loaded</p>
        {item && <p style={{ fontSize: 10, opacity: 0.7 }}>{item}</p>}
      </div>
    </Html>
  );
}

function App() {
  return (
    <Canvas>
      <Suspense fallback={<Loader />}>
        <Scene />
      </Suspense>
    </Canvas>
  );
}
```

### Lazy Loading ด้วย React.lazy

```jsx
import { lazy, Suspense } from 'react';

// Load component เฉพาะเมื่อต้องการ
const HeavyScene = lazy(() => import('./HeavyScene'));

function App() {
  const [showHeavy, setShowHeavy] = useState(false);
  
  return (
    <Canvas>
      <SimpleScene />
      {showHeavy && (
        <Suspense fallback={null}>
          <HeavyScene />
        </Suspense>
      )}
    </Canvas>
  );
}
```

---

## 65.4 react-spring กับ R3F

react-spring ให้ physics-based animations ที่ smooth มาก

### ติดตั้ง

```bash
npm install @react-spring/three
```

### พื้นฐาน useSpring

```jsx
import { useSpring, animated } from '@react-spring/three';

function SpringMesh() {
  const [active, setActive] = useState(false);
  
  const springs = useSpring({
    scale: active ? 2 : 1,
    color: active ? 'hotpink' : 'royalblue',
    position: active ? [0, 2, 0] : [0, 0, 0],
    config: {
      mass: 1,
      tension: 280,
      friction: 60,
    }
  });

  return (
    <animated.mesh
      scale={springs.scale}
      position={springs.position}
      onClick={() => setActive(!active)}
    >
      <boxGeometry />
      <animated.meshStandardMaterial color={springs.color} />
    </animated.mesh>
  );
}
```

### Spring Chain Animation

```jsx
import { useSprings, animated } from '@react-spring/three';

function WaveAnimation() {
  const count = 10;
  const [trigger, setTrigger] = useState(false);
  
  const springs = useSprings(
    count,
    Array.from({ length: count }, (_, i) => ({
      position: trigger ? [i * 1.5 - 7.5, 2, 0] : [i * 1.5 - 7.5, 0, 0],
      delay: i * 100,
      config: { mass: 1, tension: 300, friction: 30 }
    }))
  );

  return (
    <>
      {springs.map((props, i) => (
        <animated.mesh key={i} position={props.position}>
          <boxGeometry args={[0.8, 0.8, 0.8]} />
          <meshStandardMaterial color={`hsl(${i * 36}, 70%, 60%)`} />
        </animated.mesh>
      ))}
      
      <Html>
        <button onClick={() => setTrigger(!trigger)}>Wave!</button>
      </Html>
    </>
  );
}
```

### useSpring กับ Rotation

```jsx
import { useSpring, animated } from '@react-spring/three';

function FlippingCard() {
  const [flipped, setFlipped] = useState(false);
  
  const { rotation } = useSpring({
    rotation: flipped ? [0, Math.PI, 0] : [0, 0, 0],
    config: { mass: 2, tension: 200, friction: 40 }
  });

  return (
    <animated.group rotation={rotation} onClick={() => setFlipped(!flipped)}>
      {/* Front face */}
      <mesh position={[0, 0, 0.01]}>
        <planeGeometry args={[2, 3]} />
        <meshStandardMaterial color="#4ecdc4" />
      </mesh>
      
      {/* Back face */}
      <mesh position={[0, 0, -0.01]} rotation={[0, Math.PI, 0]}>
        <planeGeometry args={[2, 3]} />
        <meshStandardMaterial color="#ff6b6b" />
      </mesh>
    </animated.group>
  );
}
```

---

## 65.5 useControls (Leva)

Leva เป็น GUI สำหรับ debug และ prototype

```bash
npm install leva
```

### พื้นฐาน useControls

```jsx
import { useControls } from 'leva';

function ControlledMesh() {
  const {
    color,
    metalness,
    roughness,
    speed,
    wireframe,
    positionY
  } = useControls({
    color: '#4ecdc4',
    metalness: { value: 0.5, min: 0, max: 1, step: 0.01 },
    roughness: { value: 0.3, min: 0, max: 1, step: 0.01 },
    speed: { value: 1, min: 0, max: 5 },
    wireframe: false,
    positionY: { value: 0, min: -5, max: 5 }
  });

  const ref = useRef();
  useFrame((_, delta) => {
    if (ref.current) ref.current.rotation.y += delta * speed;
  });

  return (
    <mesh ref={ref} position-y={positionY}>
      <boxGeometry />
      <meshStandardMaterial
        color={color}
        metalness={metalness}
        roughness={roughness}
        wireframe={wireframe}
      />
    </mesh>
  );
}
```

### Grouped Controls

```jsx
import { useControls, folder } from 'leva';

function AdvancedControls() {
  const controls = useControls({
    'Geometry': folder({
      width: { value: 1, min: 0.1, max: 5 },
      height: { value: 1, min: 0.1, max: 5 },
      depth: { value: 1, min: 0.1, max: 5 },
    }),
    'Material': folder({
      color: '#4ecdc4',
      metalness: { value: 0.5, min: 0, max: 1 },
      roughness: { value: 0.5, min: 0, max: 1 },
      emissive: '#000000',
      emissiveIntensity: { value: 0, min: 0, max: 3 },
    }),
    'Transform': folder({
      posX: { value: 0, min: -10, max: 10 },
      posY: { value: 0, min: -10, max: 10 },
      rotSpeed: { value: 1, min: 0, max: 5 },
    }),
    'Debug': folder({
      wireframe: false,
      normals: false,
      showAxes: false,
    }, { collapsed: true })
  });

  return (
    <mesh
      position={[controls.posX, controls.posY, 0]}
      rotation-y={Date.now() * 0.001 * controls.rotSpeed}
    >
      <boxGeometry args={[controls.width, controls.height, controls.depth]} />
      <meshStandardMaterial
        color={controls.color}
        metalness={controls.metalness}
        roughness={controls.roughness}
        emissive={controls.emissive}
        emissiveIntensity={controls.emissiveIntensity}
        wireframe={controls.wireframe}
      />
    </mesh>
  );
}
```

---

## 65.6 R3F Events ขั้นสูง

### Raycasting และ Intersection Details

```jsx
function DetailedRaycast() {
  const [info, setInfo] = useState(null);

  return (
    <>
      <mesh
        onPointerMove={(e) => {
          // รายละเอียดของ intersection
          setInfo({
            point: e.point.toArray().map(v => v.toFixed(2)),
            face: e.faceIndex,
            uv: e.uv ? [e.uv.x.toFixed(2), e.uv.y.toFixed(2)] : null,
            distance: e.distance.toFixed(2),
            normal: e.face?.normal.toArray().map(v => v.toFixed(2))
          });
        }}
        onPointerLeave={() => setInfo(null)}
      >
        <sphereGeometry args={[2, 64, 64]} />
        <meshNormalMaterial />
      </mesh>
      
      {info && (
        <Html>
          <pre style={{ color: 'white', fontSize: 12 }}>
            {JSON.stringify(info, null, 2)}
          </pre>
        </Html>
      )}
    </>
  );
}
```

### Event Propagation

```jsx
function NestedObjects() {
  return (
    {/* Parent */}
    <mesh
      onClick={(e) => {
        // ถ้าไม่ stopPropagation จะ bubble ขึ้นมาที่ parent
        console.log('parent clicked');
      }}
    >
      <boxGeometry args={[3, 3, 3]} />
      <meshStandardMaterial transparent opacity={0.3} />
      
      {/* Child - overlapping */}
      <mesh
        position={[0, 0, 1.5]}
        onClick={(e) => {
          e.stopPropagation(); // หยุด bubble
          console.log('child clicked only');
        }}
      >
        <sphereGeometry args={[0.5]} />
        <meshStandardMaterial color="red" />
      </mesh>
    </mesh>
  );
}
```

### Global Pointer Events

```jsx
function GlobalPointerTracker() {
  const [mousePos, setMousePos] = useState({ x: 0, y: 0 });
  
  // ใช้ Canvas events ระดับ global
  return (
    <Canvas
      onPointerMove={(e) => {
        setMousePos({ x: e.clientX, y: e.clientY });
      }}
    >
      <FollowCursor mousePos={mousePos} />
    </Canvas>
  );
}

function FollowCursor({ mousePos }) {
  const { viewport } = useThree();
  const ref = useRef();
  
  useFrame(({ mouse }) => {
    if (!ref.current) return;
    // mouse เป็น normalized coordinates (-1 to 1)
    ref.current.position.x = mouse.x * viewport.width / 2;
    ref.current.position.y = mouse.y * viewport.height / 2;
  });
  
  return (
    <mesh ref={ref}>
      <sphereGeometry args={[0.3]} />
      <meshBasicMaterial color="yellow" />
    </mesh>
  );
}
```

---

## 65.7 Portals ใน R3F

Portals ให้เราเรนเดอร์ Three.js objects ไปยัง scene อื่น

```jsx
import { createPortal, useFrame } from '@react-three/fiber';
import { useMemo, useRef } from 'react';
import * as THREE from 'three';

function PortalDemo() {
  // สร้าง virtual scene สำหรับ render to texture
  const virtualScene = useMemo(() => new THREE.Scene(), []);
  const virtualCamera = useMemo(() => 
    new THREE.PerspectiveCamera(50, 1, 0.1, 100), []);
  
  const renderTarget = useMemo(() => 
    new THREE.WebGLRenderTarget(512, 512), []);
  
  const portalRef = useRef();

  useFrame(({ gl, scene, camera }) => {
    // Render virtual scene ไปยัง render target
    gl.setRenderTarget(renderTarget);
    gl.render(virtualScene, virtualCamera);
    gl.setRenderTarget(null);
  }, 1); // Priority 1 = render ก่อน main scene

  return (
    <>
      {/* Render virtual scene ผ่าน portal */}
      {createPortal(
        <group>
          {/* นี่คือ content ใน virtual scene */}
          <mesh rotation={[0, Date.now() * 0.001, 0]}>
            <torusGeometry args={[1, 0.3, 16, 100]} />
            <meshNormalMaterial />
          </mesh>
          <ambientLight />
          <pointLight position={[3, 3, 3]} />
        </group>,
        virtualScene
      )}
      
      {/* Main scene - แสดง render target บน mesh */}
      <mesh ref={portalRef}>
        <planeGeometry args={[3, 3]} />
        <meshBasicMaterial map={renderTarget.texture} />
      </mesh>
    </>
  );
}
```

---

## 65.8 Canvas Resize Handling

```jsx
import { Canvas, useThree } from '@react-three/fiber';

// Auto resize - R3F handle ให้เอง
function AutoResizeScene() {
  const { size } = useThree();
  
  useEffect(() => {
    console.log('Canvas resized to:', size.width, 'x', size.height);
  }, [size]);
  
  return null;
}

// Custom resize behavior
function App() {
  const [canvasSize, setCanvasSize] = useState({ width: 800, height: 600 });

  useEffect(() => {
    const handleResize = () => {
      setCanvasSize({
        width: window.innerWidth,
        height: window.innerHeight
      });
    };
    
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return (
    <Canvas
      style={{ width: canvasSize.width, height: canvasSize.height }}
      camera={{ aspect: canvasSize.width / canvasSize.height }}
    >
      <AutoResizeScene />
    </Canvas>
  );
}
```

---

## 65.9 โปรเจกต์: Complete R3F Product Showcase

```jsx
// src/ProductShowcase.jsx
import { Canvas, useFrame, useLoader, useThree } from '@react-three/fiber';
import { useSpring, animated } from '@react-spring/three';
import { Suspense, useRef, useState, useMemo } from 'react';
import { useControls } from 'leva';
import { TextureLoader, MathUtils } from 'three';

// Product Display Component
function ProductModel({ isHovered, isSelected, onClick }) {
  const meshRef = useRef();
  const time = useRef(0);

  const { scale } = useSpring({
    scale: isSelected ? 1.3 : isHovered ? 1.1 : 1,
    config: { mass: 1, tension: 300, friction: 30 }
  });

  const { rotationY } = useSpring({
    rotationY: isSelected ? Math.PI * 2 : 0,
    config: { mass: 2, tension: 100, friction: 30 }
  });

  useFrame((state, delta) => {
    if (!meshRef.current) return;
    time.current += delta;
    
    // Gentle floating animation
    meshRef.current.position.y = Math.sin(time.current) * 0.1;
    
    // Slow auto-rotation when not selected
    if (!isSelected) {
      meshRef.current.rotation.y += delta * 0.3;
    }
  });

  return (
    <animated.mesh
      ref={meshRef}
      scale={scale}
      rotation-y={rotationY}
      onClick={onClick}
    >
      <boxGeometry args={[1.5, 2, 0.3]} />
      <meshStandardMaterial
        color={isSelected ? '#4ecdc4' : isHovered ? '#45b7d1' : '#2c3e50'}
        metalness={0.8}
        roughness={0.2}
        emissive={isSelected ? '#4ecdc4' : '#000000'}
        emissiveIntensity={isSelected ? 0.2 : 0}
      />
    </animated.mesh>
  );
}

// Camera Animation
function CameraController({ target }) {
  const { camera } = useThree();
  const targetPosition = useMemo(() => {
    const positions = [
      [0, 0, 8],
      [-5, 2, 6],
      [5, 2, 6],
      [0, 5, 5]
    ];
    return positions[target] || positions[0];
  }, [target]);

  useFrame((state, delta) => {
    camera.position.x = MathUtils.lerp(camera.position.x, targetPosition[0], delta * 2);
    camera.position.y = MathUtils.lerp(camera.position.y, targetPosition[1], delta * 2);
    camera.position.z = MathUtils.lerp(camera.position.z, targetPosition[2], delta * 2);
    camera.lookAt(0, 0, 0);
  });

  return null;
}

// Environment Setup
function Environment() {
  return (
    <>
      <ambientLight intensity={0.4} />
      <directionalLight position={[5, 10, 5]} intensity={1.5} castShadow />
      <pointLight position={[-5, 5, -5]} intensity={50} color="#4ecdc4" />
      <pointLight position={[5, -5, 5]} intensity={30} color="#ff6b6b" />
    </>
  );
}

// Ground Reflection
function ReflectiveFloor() {
  return (
    <mesh rotation={[-Math.PI / 2, 0, 0]} position={[0, -2, 0]} receiveShadow>
      <planeGeometry args={[30, 30]} />
      <meshStandardMaterial
        color="#1a1a2e"
        metalness={0.5}
        roughness={0.3}
      />
    </mesh>
  );
}

// Product Info Panel
function ProductInfo({ product, onClose }) {
  if (!product) return null;
  
  return (
    <div style={{
      position: 'absolute',
      right: 20,
      top: '50%',
      transform: 'translateY(-50%)',
      background: 'rgba(0, 0, 0, 0.85)',
      backdropFilter: 'blur(15px)',
      color: 'white',
      padding: '30px',
      borderRadius: '16px',
      width: '280px',
      border: '1px solid rgba(78, 205, 196, 0.3)',
      fontFamily: 'sans-serif'
    }}>
      <button
        onClick={onClose}
        style={{
          position: 'absolute',
          top: 15,
          right: 15,
          background: 'none',
          border: 'none',
          color: 'white',
          fontSize: '20px',
          cursor: 'pointer'
        }}
      >×</button>
      
      <h2 style={{ 
        margin: '0 0 8px', 
        color: '#4ecdc4',
        fontSize: '1.4em'
      }}>
        {product.name}
      </h2>
      
      <div style={{
        background: '#4ecdc4',
        color: '#000',
        display: 'inline-block',
        padding: '4px 12px',
        borderRadius: '20px',
        fontSize: '0.85em',
        marginBottom: '15px'
      }}>
        {product.category}
      </div>
      
      <p style={{ 
        color: '#aaa', 
        lineHeight: 1.6,
        fontSize: '0.9em',
        marginBottom: '20px'
      }}>
        {product.description}
      </p>
      
      <div style={{
        display: 'flex',
        justifyContent: 'space-between',
        alignItems: 'center',
        marginBottom: '20px'
      }}>
        <span style={{ fontSize: '1.5em', fontWeight: 'bold' }}>
          ${product.price}
        </span>
        <div style={{ color: '#ffd700' }}>
          {'★'.repeat(Math.floor(product.rating))} 
          {product.rating % 1 > 0 ? '½' : ''}
          <span style={{ color: '#666', fontSize: '0.9em' }}>
            ({product.reviews})
          </span>
        </div>
      </div>
      
      <button style={{
        width: '100%',
        padding: '12px',
        background: 'linear-gradient(135deg, #4ecdc4, #45b7d1)',
        border: 'none',
        borderRadius: '8px',
        color: 'white',
        fontSize: '1em',
        fontWeight: 'bold',
        cursor: 'pointer',
        transition: 'opacity 0.2s'
      }}>
        Add to Cart
      </button>
    </div>
  );
}

// Products Data
const PRODUCTS = [
  {
    id: 0,
    name: 'Quantum Speaker',
    category: 'Audio',
    description: 'Premium wireless speaker with 360° sound and 24-hour battery life. Perfect for home and outdoor use.',
    price: 299,
    rating: 4.5,
    reviews: 128
  },
  {
    id: 1,
    name: 'Nexus Watch',
    category: 'Wearable',
    description: 'Smart watch with health monitoring, GPS tracking, and 7-day battery. Water-resistant to 50m.',
    price: 449,
    rating: 4,
    reviews: 86
  },
  {
    id: 2,
    name: 'Vision Lens',
    category: 'Camera',
    description: 'Professional camera lens with f/1.4 aperture. Captures stunning detail in any lighting condition.',
    price: 799,
    rating: 5,
    reviews: 204
  }
];

// Product Item in Scene
function ProductItem({ product, position, onSelect, selectedId }) {
  const [hovered, setHovered] = useState(false);
  const isSelected = selectedId === product.id;

  return (
    <group position={position}>
      <ProductModel
        isHovered={hovered}
        isSelected={isSelected}
        onClick={(e) => {
          e.stopPropagation();
          onSelect(product.id);
        }}
      />
      
      {/* Product Label */}
      <mesh position={[0, -1.8, 0]}>
        <planeGeometry args={[1.6, 0.4]} />
        <meshBasicMaterial color={isSelected ? '#4ecdc4' : '#333'} />
      </mesh>
    </group>
  );
}

// Main App
export default function ProductShowcase() {
  const [selectedId, setSelectedId] = useState(null);
  const [cameraView, setCameraView] = useState(0);
  
  const selectedProduct = PRODUCTS.find(p => p.id === selectedId);

  const handleSelect = (id) => {
    if (selectedId === id) {
      setSelectedId(null);
      setCameraView(0);
    } else {
      setSelectedId(id);
      setCameraView(id + 1);
    }
  };

  return (
    <div style={{ 
      width: '100vw', 
      height: '100vh', 
      position: 'relative',
      background: '#0a0a1a'
    }}>
      
      {/* Header */}
      <div style={{
        position: 'absolute',
        top: 20,
        left: '50%',
        transform: 'translateX(-50%)',
        color: 'white',
        fontFamily: 'sans-serif',
        textAlign: 'center',
        zIndex: 10
      }}>
        <h1 style={{ 
          margin: 0, 
          fontSize: '1.8em',
          background: 'linear-gradient(135deg, #4ecdc4, #45b7d1)',
          WebkitBackgroundClip: 'text',
          WebkitTextFillColor: 'transparent'
        }}>
          NEXUS STORE
        </h1>
        <p style={{ margin: '5px 0 0', color: '#666', fontSize: '0.9em' }}>
          Click a product to explore
        </p>
      </div>
      
      {/* Camera View Buttons */}
      <div style={{
        position: 'absolute',
        top: 20,
        right: 20,
        display: 'flex',
        gap: 8,
        zIndex: 10
      }}>
        {['Front', 'Left', 'Right', 'Top'].map((label, i) => (
          <button
            key={i}
            onClick={() => setCameraView(i)}
            style={{
              padding: '6px 12px',
              background: cameraView === i ? '#4ecdc4' : 'rgba(255,255,255,0.1)',
              border: 'none',
              borderRadius: 6,
              color: 'white',
              cursor: 'pointer',
              fontSize: '0.8em'
            }}
          >
            {label}
          </button>
        ))}
      </div>

      {/* 3D Canvas */}
      <Canvas shadows camera={{ position: [0, 0, 8], fov: 60 }}>
        <CameraController target={cameraView} />
        <Environment />
        <ReflectiveFloor />
        
        {PRODUCTS.map((product, i) => (
          <ProductItem
            key={product.id}
            product={product}
            position={[(i - 1) * 3.5, 0, 0]}
            onSelect={handleSelect}
            selectedId={selectedId}
          />
        ))}

        {/* Deselect on background click */}
        <mesh 
          position={[0, 0, -5]}
          onClick={() => { setSelectedId(null); setCameraView(0); }}
        >
          <planeGeometry args={[50, 50]} />
          <meshBasicMaterial visible={false} />
        </mesh>
      </Canvas>
      
      {/* Product Info Panel */}
      <ProductInfo
        product={selectedProduct}
        onClose={() => { setSelectedId(null); setCameraView(0); }}
      />
    </div>
  );
}
```

---

## 65.10 โปรเจกต์: R3F Character กับ Animations

```jsx
// src/CharacterDemo.jsx
import { Canvas, useFrame } from '@react-three/fiber';
import { useRef, useState, useMemo } from 'react';
import { useSpring, animated } from '@react-spring/three';
import * as THREE from 'three';

// Character ที่สร้างจาก primitive shapes
function Character({ action }) {
  const groupRef = useRef();
  const leftArmRef = useRef();
  const rightArmRef = useRef();
  const leftLegRef = useRef();
  const rightLegRef = useRef();
  
  const { bodyY } = useSpring({
    bodyY: action === 'jump' ? 2 : 0,
    config: { mass: 1, tension: 300, friction: 20 }
  });

  useFrame((state) => {
    const t = state.clock.elapsedTime;
    const speed = action === 'run' ? 8 : action === 'walk' ? 3 : 0;
    
    if (speed > 0) {
      // Arm swing
      if (leftArmRef.current) {
        leftArmRef.current.rotation.x = Math.sin(t * speed) * 0.5;
      }
      if (rightArmRef.current) {
        rightArmRef.current.rotation.x = -Math.sin(t * speed) * 0.5;
      }
      
      // Leg swing
      if (leftLegRef.current) {
        leftLegRef.current.rotation.x = -Math.sin(t * speed) * 0.5;
      }
      if (rightLegRef.current) {
        rightLegRef.current.rotation.x = Math.sin(t * speed) * 0.5;
      }
    } else if (action === 'idle') {
      // Idle breathing animation
      if (groupRef.current) {
        groupRef.current.scale.y = 1 + Math.sin(t * 2) * 0.02;
      }
    }
  });

  return (
    <animated.group ref={groupRef} position-y={bodyY}>
      {/* Head */}
      <mesh position={[0, 1.4, 0]}>
        <boxGeometry args={[0.5, 0.5, 0.5]} />
        <meshStandardMaterial color="#ffd700" />
      </mesh>
      
      {/* Body */}
      <mesh position={[0, 0.5, 0]}>
        <boxGeometry args={[0.6, 0.8, 0.3]} />
        <meshStandardMaterial color="#3498db" />
      </mesh>
      
      {/* Left Arm */}
      <group ref={leftArmRef} position={[-0.5, 0.8, 0]}>
        <mesh position={[0, -0.3, 0]}>
          <boxGeometry args={[0.2, 0.6, 0.2]} />
          <meshStandardMaterial color="#ffd700" />
        </mesh>
      </group>
      
      {/* Right Arm */}
      <group ref={rightArmRef} position={[0.5, 0.8, 0]}>
        <mesh position={[0, -0.3, 0]}>
          <boxGeometry args={[0.2, 0.6, 0.2]} />
          <meshStandardMaterial color="#ffd700" />
        </mesh>
      </group>
      
      {/* Left Leg */}
      <group ref={leftLegRef} position={[-0.2, 0.1, 0]}>
        <mesh position={[0, -0.3, 0]}>
          <boxGeometry args={[0.2, 0.6, 0.2]} />
          <meshStandardMaterial color="#2c3e50" />
        </mesh>
      </group>
      
      {/* Right Leg */}
      <group ref={rightLegRef} position={[0.2, 0.1, 0]}>
        <mesh position={[0, -0.3, 0]}>
          <boxGeometry args={[0.2, 0.6, 0.2]} />
          <meshStandardMaterial color="#2c3e50" />
        </mesh>
      </group>
    </animated.group>
  );
}

// Main Character Demo
export default function CharacterDemo() {
  const [action, setAction] = useState('idle');
  
  return (
    <div style={{ width: '100vw', height: '100vh', position: 'relative' }}>
      
      {/* Controls */}
      <div style={{
        position: 'absolute',
        bottom: 30,
        left: '50%',
        transform: 'translateX(-50%)',
        display: 'flex',
        gap: 10,
        zIndex: 10
      }}>
        {['idle', 'walk', 'run', 'jump'].map(a => (
          <button
            key={a}
            onClick={() => setAction(a)}
            style={{
              padding: '10px 20px',
              background: action === a ? '#4ecdc4' : '#333',
              border: 'none',
              borderRadius: 8,
              color: 'white',
              cursor: 'pointer',
              textTransform: 'capitalize',
              fontWeight: 'bold'
            }}
          >
            {a}
          </button>
        ))}
      </div>
      
      <Canvas shadows camera={{ position: [0, 2, 5], fov: 60 }}>
        <color attach="background" args={['#87ceeb']} />
        <ambientLight intensity={0.6} />
        <directionalLight position={[5, 10, 5]} intensity={1} castShadow />
        
        <Character action={action} />
        
        {/* Ground */}
        <mesh rotation={[-Math.PI / 2, 0, 0]} position={[0, -0.5, 0]} receiveShadow>
          <planeGeometry args={[20, 20]} />
          <meshStandardMaterial color="#8fbc8f" />
        </mesh>
      </Canvas>
    </div>
  );
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: InstancedMesh Optimization
สร้าง particle system ด้วย instancedMesh จำนวน 50,000 particles:
- แต่ละ particle มีสีและขนาดต่างกัน
- Animate ด้วย GPU-friendly method
- เปรียบเทียบ FPS ระหว่าง instanced vs non-instanced

### แบบฝึกหัดที่ 2: Asset Loading Pipeline
สร้าง loading system ที่:
- โหลด GLTF model และ textures พร้อมกัน
- แสดง progress bar ด้วย useProgress
- มี error handling สำหรับ failed loads
- Cache ไว้ใน sessionStorage

### แบบฝึกหัดที่ 3: Spring Animation Showcase
สร้าง animation showcase ที่มี:
- Bounce effect เมื่อคลิก
- Chain reaction animation (domino effect)
- Morphing shape animation
- Spring-based camera follow

### แบบฝึกหัดที่ 4: Product Gallery
สร้าง product gallery ที่สมบูรณ์:
- 6 products พร้อม info
- Smooth camera transitions
- Shopping cart (เพิ่ม/ลบสินค้า)
- Filter by category ด้วย animation

### แบบฝึกหัดที่ 5: Character Customizer
สร้าง character customizer:
- เปลี่ยนสีร่างกาย/เสื้อผ้า
- ปรับขนาดแต่ละส่วน
- บันทึก/โหลด character config
- Share URL ด้วย query params

---

## สรุป

ในบทนี้เราเรียนรู้:
1. **Performance Optimization** - useMemo, instancedMesh, frameloop
2. **useLoader** - โหลด textures, models, และ custom data
3. **Suspense** - จัดการ async loading ด้วย React patterns
4. **react-spring** - Physics-based animations ที่ smooth
5. **Leva Controls** - GUI สำหรับ debug และ prototype
6. **Advanced Events** - Raycasting details, event propagation
7. **Portals** - Render to texture ใน R3F

บทถัดไปเราจะเรียน **Drei Library** ซึ่งเป็น collection ของ R3F helpers ที่ทรงพลัง
