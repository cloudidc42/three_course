# Part 66: Drei Library (R3F Helpers)

## บทนำ

`@react-three/drei` คือ collection ของ helper components และ hooks สำหรับ React Three Fiber ที่ช่วยลดโค้ดที่ต้องเขียนซ้ำๆ ในบทนี้เราจะสำรวจ components ที่ใช้บ่อยที่สุดในโปรเจกต์จริง

```bash
npm install @react-three/drei
```

---

## 66.1 Text Components

### Html - HTML ใน 3D Space

```jsx
import { Html } from '@react-three/drei';

function HTMLOverlay() {
  return (
    <mesh position={[0, 0, 0]}>
      <boxGeometry />
      <meshStandardMaterial color="royalblue" />
      
      {/* HTML overlay บน 3D object */}
      <Html
        position={[0, 1.5, 0]}
        center           // center กับ anchor point
        distanceFactor={8}  // scale ตาม distance จาก camera
        occlude          // ซ่อนเมื่อถูก object อื่นบัง
        zIndexRange={[100, 0]}  // z-index range
        transform        // transform ตาม 3D rotation
        sprite           // always face camera
      >
        <div style={{
          background: 'white',
          padding: '10px 15px',
          borderRadius: 8,
          fontSize: '14px',
          whiteSpace: 'nowrap',
          boxShadow: '0 4px 20px rgba(0,0,0,0.3)'
        }}>
          <strong>Hover me!</strong>
          <p>This is HTML in 3D space</p>
        </div>
      </Html>
    </mesh>
  );
}
```

### Text - 2D Text ด้วย troika-three-text

```jsx
import { Text } from '@react-three/drei';

function TextDemo() {
  return (
    <>
      <Text
        position={[0, 2, 0]}
        fontSize={0.5}
        color="white"
        anchorX="center"
        anchorY="middle"
        font="/fonts/Inter-Bold.woff"
        letterSpacing={0.05}
        lineHeight={1.2}
        textAlign="center"
        maxWidth={5}
      >
        Hello Three.js World!
      </Text>
      
      {/* Text ที่มี stroke */}
      <Text
        position={[0, 0, 0]}
        fontSize={1}
        color="white"
        strokeColor="royalblue"
        strokeWidth={0.05}
        outlineColor="black"
        outlineWidth={0.03}
      >
        Outlined Text
      </Text>
      
      {/* Multiline */}
      <Text
        position={[0, -2, 0]}
        fontSize={0.3}
        color="#aaa"
        maxWidth={8}
        textAlign="justify"
      >
        {`This is a longer paragraph that wraps to multiple lines automatically.
Three.js makes 3D text easy!`}
      </Text>
    </>
  );
}
```

### Text3D - 3D Extruded Text

```jsx
import { Text3D, Center } from '@react-three/drei';

function Text3DDemo() {
  return (
    <Center>
      <Text3D
        font="/fonts/helvetiker_regular.typeface.json"
        size={0.8}
        height={0.2}        // Depth of extrusion
        curveSegments={12}
        bevelEnabled
        bevelThickness={0.02}
        bevelSize={0.02}
        bevelOffset={0}
        bevelSegments={5}
      >
        DREI
        <meshNormalMaterial />
      </Text3D>
    </Center>
  );
}
```

---

## 66.2 Environment & Background

### Environment - HDR Environment Map

```jsx
import { Environment } from '@react-three/drei';

function EnvironmentDemo() {
  return (
    <>
      {/* Environment จาก preset */}
      <Environment preset="city" />
      
      {/* หรือจาก HDR file */}
      <Environment files="/hdri/studio.hdr" />
      
      {/* Environment ที่ render เป็น background ด้วย */}
      <Environment preset="sunset" background />
      
      {/* Custom environment ด้วย children */}
      <Environment>
        <Lightformer
          intensity={0.75}
          rotation-x={Math.PI / 2}
          position={[0, 5, -9]}
          scale={[10, 10, 1]}
        />
      </Environment>
    </>
  );
}

// Available presets:
// 'apartment', 'city', 'dawn', 'forest', 'lobby',
// 'night', 'park', 'studio', 'sunset', 'warehouse'
```

### Sky Component

```jsx
import { Sky } from '@react-three/drei';

function SkyDemo() {
  const { sunPosition, turbidity, rayleigh } = useControls({
    sunPosition: [100, 100, 100],
    turbidity: { value: 10, min: 0, max: 20 },
    rayleigh: { value: 3, min: 0, max: 4 },
  });

  return (
    <Sky
      distance={450000}
      sunPosition={sunPosition}
      inclination={0}
      azimuth={0.25}
      turbidity={turbidity}
      rayleigh={rayleigh}
      mieCoefficient={0.005}
      mieDirectionalG={0.8}
    />
  );
}
```

### Stars

```jsx
import { Stars } from '@react-three/drei';

function StarsDemo() {
  return (
    <Stars
      radius={300}         // Sphere radius
      depth={60}           // Depth of star field
      count={5000}         // Number of stars
      factor={7}           // Star size factor
      saturation={0}       // Star color saturation
      fade={true}          // Fade stars at edges
      speed={1}            // Animation speed
    />
  );
}
```

### Cloud

```jsx
import { Cloud, Clouds } from '@react-three/drei';

function CloudsDemo() {
  return (
    <Clouds material={THREE.MeshBasicMaterial}>
      <Cloud
        position={[0, 5, -10]}
        speed={0.4}
        opacity={0.5}
        segments={40}
        bounds={[10, 2, 2]}
        volume={10}
        color="white"
      />
      <Cloud
        position={[-5, 3, -15]}
        speed={0.2}
        opacity={0.3}
        bounds={[8, 2, 2]}
        volume={8}
        color="#ccc"
      />
    </Clouds>
  );
}
```

---

## 66.3 Camera Controls

### CameraControls

```jsx
import { CameraControls } from '@react-three/drei';
import { useRef } from 'react';

function CameraControlsDemo() {
  const cameraControlsRef = useRef();

  const moveTo = async () => {
    await cameraControlsRef.current?.setPosition(5, 5, 5, true);
    await cameraControlsRef.current?.setTarget(0, 0, 0, true);
  };

  return (
    <>
      <CameraControls
        ref={cameraControlsRef}
        minDistance={1}
        maxDistance={100}
        polarAngle={Math.PI / 4}
        minPolarAngle={0}
        maxPolarAngle={Math.PI / 2}
        enabled={true}
        dollySpeed={1}
        truckSpeed={2}
      />
      
      <Html position={[0, 3, 0]}>
        <button onClick={moveTo}>Move Camera</button>
      </Html>
    </>
  );
}
```

### OrbitControls

```jsx
import { OrbitControls } from '@react-three/drei';

function ControlsDemo() {
  return (
    <OrbitControls
      enableZoom={true}
      enablePan={true}
      enableRotate={true}
      autoRotate={true}
      autoRotateSpeed={2}
      minDistance={2}
      maxDistance={20}
      minPolarAngle={0}
      maxPolarAngle={Math.PI / 2}  // ไม่ให้มุมกล้องต่ำกว่า horizon
      target={[0, 0, 0]}
      makeDefault  // ใช้เป็น default controls ของ scene
    />
  );
}
```

### PerspectiveCamera

```jsx
import { PerspectiveCamera } from '@react-three/drei';

function CustomCamera() {
  return (
    <PerspectiveCamera
      makeDefault    // เซ็ตเป็น camera หลัก
      position={[5, 5, 5]}
      fov={60}
      near={0.1}
      far={1000}
    />
  );
}
```

---

## 66.4 Material Helpers

### MeshReflectorMaterial - พื้นสะท้อน

```jsx
import { MeshReflectorMaterial } from '@react-three/drei';

function ReflectiveFloor() {
  return (
    <mesh rotation={[-Math.PI / 2, 0, 0]} position={[0, -1, 0]}>
      <planeGeometry args={[20, 20]} />
      <MeshReflectorMaterial
        blur={[300, 100]}      // Blur (horizontal, vertical)
        resolution={2048}      // Render target resolution
        mixBlur={1}            // Blur mix amount
        mixStrength={80}       // Reflection strength
        roughness={1}
        depthScale={1.2}
        minDepthThreshold={0.4}
        maxDepthThreshold={1.4}
        color="#151515"
        metalness={0.5}
        mirror={0}             // Mirror effect (0-1)
      />
    </mesh>
  );
}
```

### MeshDistortMaterial - Material ที่บิดเบี้ยว

```jsx
import { MeshDistortMaterial } from '@react-three/drei';
import { useRef } from 'react';
import { useFrame } from '@react-three/fiber';

function DistortedSphere() {
  const materialRef = useRef();
  
  useFrame((state) => {
    if (materialRef.current) {
      materialRef.current.distort = Math.sin(state.clock.elapsedTime) * 0.5 + 0.5;
    }
  });

  return (
    <mesh>
      <sphereGeometry args={[1, 64, 64]} />
      <MeshDistortMaterial
        ref={materialRef}
        color="hotpink"
        distort={0.5}      // Distortion amount (0-1)
        speed={2}          // Animation speed
        roughness={0.4}
        metalness={0.1}
      />
    </mesh>
  );
}
```

### MeshWobbleMaterial - Wobble Effect

```jsx
import { MeshWobbleMaterial } from '@react-three/drei';

function WobblyTorus() {
  return (
    <mesh>
      <torusGeometry args={[1, 0.3, 16, 100]} />
      <MeshWobbleMaterial
        color="aquamarine"
        factor={0.4}    // Wobble intensity
        speed={4}       // Wobble speed
        roughness={0.4}
      />
    </mesh>
  );
}
```

---

## 66.5 Loaders

### useGLTF - โหลด GLTF Models

```jsx
import { useGLTF } from '@react-three/drei';

// Preload ก่อนที่ component จะ mount
useGLTF.preload('/models/helmet.glb');

function Helmet() {
  const { scene, animations, cameras, materials, nodes } = useGLTF('/models/helmet.glb');
  
  return (
    <primitive 
      object={scene} 
      scale={0.5}
      position={[0, 0, 0]}
    />
  );
}
```

### useGLTF กับ Animations

```jsx
import { useGLTF, useAnimations } from '@react-three/drei';
import { useEffect, useRef } from 'react';

function AnimatedCharacter({ action = 'Idle' }) {
  const group = useRef();
  const { scene, animations } = useGLTF('/models/character.glb');
  const { actions, mixer } = useAnimations(animations, group);

  useEffect(() => {
    // หยุด action ก่อนหน้า
    Object.values(actions).forEach(a => a.stop());
    
    // เล่น action ใหม่
    if (actions[action]) {
      actions[action].reset().fadeIn(0.3).play();
    }
    
    return () => {
      if (actions[action]) {
        actions[action].fadeOut(0.3);
      }
    };
  }, [action, actions]);

  return <primitive ref={group} object={scene} />;
}
```

### useTexture

```jsx
import { useTexture } from '@react-three/drei';

function TexturedObject() {
  // โหลด single texture
  const texture = useTexture('/textures/brick.jpg');
  
  // โหลด multiple textures
  const [colorMap, normalMap, roughnessMap] = useTexture([
    '/textures/wood/color.jpg',
    '/textures/wood/normal.jpg',
    '/textures/wood/roughness.jpg',
  ]);

  // โหลด texture map object
  const textureProps = useTexture({
    map: '/textures/metal/color.jpg',
    normalMap: '/textures/metal/normal.jpg',
    metalnessMap: '/textures/metal/metalness.jpg',
  });

  return (
    <mesh>
      <sphereGeometry args={[1, 64, 64]} />
      <meshStandardMaterial {...textureProps} />
    </mesh>
  );
}
```

---

## 66.6 Helper Components

### Float - Floating Animation

```jsx
import { Float } from '@react-three/drei';

function FloatingObjects() {
  return (
    <>
      <Float
        speed={1}              // Float speed
        rotationIntensity={0.5}  // Rotation intensity
        floatIntensity={0.5}   // Float up/down intensity
        floatingRange={[-0.1, 0.1]}  // Y range
      >
        <mesh>
          <boxGeometry />
          <meshStandardMaterial color="hotpink" />
        </mesh>
      </Float>
      
      {/* Float กับ children หลายตัว */}
      <Float speed={2} floatIntensity={1}>
        <group>
          <mesh position={[0, 0, 0]}>
            <sphereGeometry args={[0.5]} />
            <meshStandardMaterial color="gold" />
          </mesh>
          <Text position={[0, 1, 0]} fontSize={0.3}>Hello!</Text>
        </group>
      </Float>
    </>
  );
}
```

### Billboard - Always Face Camera

```jsx
import { Billboard } from '@react-three/drei';

function BillboardDemo() {
  return (
    <>
      {/* Billboard ที่หันหน้าหา camera เสมอ */}
      <Billboard
        follow={true}       // ติดตาม camera
        lockX={false}       // ล็อค rotation X
        lockY={false}       // ล็อค rotation Y  
        lockZ={false}       // ล็อค rotation Z
      >
        <mesh>
          <planeGeometry args={[2, 1]} />
          <meshBasicMaterial color="white" side={THREE.DoubleSide} />
        </mesh>
      </Billboard>
      
      {/* Billboard ที่เป็น sprite (always face camera, scale with distance) */}
      <Sprite position={[3, 0, 0]}>
        <spriteMaterial color="gold" />
      </Sprite>
    </>
  );
}
```

### ContactShadows

```jsx
import { ContactShadows } from '@react-three/drei';

function ShadowDemo() {
  return (
    <>
      <mesh position={[0, 1, 0]} castShadow>
        <boxGeometry />
        <meshStandardMaterial color="royalblue" />
      </mesh>
      
      {/* Contact shadows บน ground */}
      <ContactShadows
        position={[0, 0, 0]}
        opacity={0.75}
        scale={10}
        blur={2.5}
        far={4}
        resolution={256}
        color="#000000"
        frames={1}   // 1 = render ครั้งเดียว (static), Infinity = animate
      />
    </>
  );
}
```

### SoftShadows

```jsx
import { SoftShadows } from '@react-three/drei';

function SoftShadowScene() {
  return (
    <>
      {/* เปิด soft shadows สำหรับ whole scene */}
      <SoftShadows
        size={25}    // Penumbra size
        focus={0}    // Shadow sharpness at focus
        samples={10} // Sample count (higher = better quality, slower)
      />
      
      <directionalLight
        position={[5, 10, 5]}
        intensity={1}
        castShadow
      />
      
      <mesh castShadow>
        <boxGeometry />
        <meshStandardMaterial />
      </mesh>
      
      <mesh receiveShadow rotation={[-Math.PI/2, 0, 0]}>
        <planeGeometry args={[20, 20]} />
        <shadowMaterial opacity={0.3} />
      </mesh>
    </>
  );
}
```

---

## 66.7 Instances & Performance

### Instances - การสร้าง Instances แบบ Declarative

```jsx
import { Instances, Instance } from '@react-three/drei';

function DeclarativeInstances() {
  const positions = useMemo(() =>
    Array.from({ length: 500 }, () => ({
      position: [
        (Math.random() - 0.5) * 20,
        (Math.random() - 0.5) * 20,
        (Math.random() - 0.5) * 20,
      ],
      rotation: [Math.random() * Math.PI, Math.random() * Math.PI, 0],
      scale: Math.random() * 0.5 + 0.5,
      color: `hsl(${Math.random() * 360}, 70%, 60%)`
    })), []
  );

  return (
    <Instances
      limit={1000}     // Maximum instances
      range={500}      // Visible instances
    >
      <boxGeometry args={[0.3, 0.3, 0.3]} />
      <meshStandardMaterial />
      
      {positions.map((props, i) => (
        <Instance
          key={i}
          position={props.position}
          rotation={props.rotation}
          scale={props.scale}
          color={props.color}
        />
      ))}
    </Instances>
  );
}
```

### Merged - รวม Geometries สำหรับ Performance

```jsx
import { Merged } from '@react-three/drei';
import { useMemo } from 'react';
import * as THREE from 'three';

function MergedGeometries() {
  const meshes = useMemo(() => ({
    Box: new THREE.Mesh(new THREE.BoxGeometry()),
    Sphere: new THREE.Mesh(new THREE.SphereGeometry(0.5, 16, 16)),
    Torus: new THREE.Mesh(new THREE.TorusGeometry(0.5, 0.2, 8, 32)),
  }), []);

  return (
    <Merged meshes={meshes}>
      {({ Box, Sphere, Torus }) => (
        <>
          {Array.from({ length: 100 }, (_, i) => (
            <group key={i} position={[Math.random() * 20 - 10, 0, Math.random() * 20 - 10]}>
              <Box position={[-2, 0, 0]} color={`hsl(${i * 3.6}, 70%, 60%)`} />
              <Sphere position={[0, 0, 0]} color={`hsl(${i * 3.6 + 120}, 70%, 60%)`} />
              <Torus position={[2, 0, 0]} color={`hsl(${i * 3.6 + 240}, 70%, 60%)`} />
            </group>
          ))}
        </>
      )}
    </Merged>
  );
}
```

---

## 66.8 Sparkles & Effects

### Sparkles

```jsx
import { Sparkles } from '@react-three/drei';

function SparkleEffect() {
  return (
    <>
      {/* Sparkles รอบ mesh */}
      <mesh>
        <sphereGeometry args={[1, 32, 32]} />
        <meshStandardMaterial color="gold" metalness={1} roughness={0} />
        
        <Sparkles
          count={50}
          scale={3}      // Area size
          size={2}       // Sparkle size
          speed={0.4}    // Animation speed
          opacity={0.7}
          color="gold"
          noise={0.5}    // Movement noise
        />
      </mesh>
    </>
  );
}
```

---

## 66.9 โปรเจกต์: Drei Showcase Demo

```jsx
// src/DreiShowcase.jsx
import { Canvas, useFrame } from '@react-three/fiber';
import {
  Float, Text, Environment, Stars, Cloud, Clouds,
  ContactShadows, MeshDistortMaterial, MeshWobbleMaterial,
  Billboard, Sparkles, OrbitControls, PerspectiveCamera,
  Html, Center
} from '@react-three/drei';
import { useRef, useState, Suspense } from 'react';
import * as THREE from 'three';

// Hero Text
function HeroText() {
  return (
    <Float speed={1} floatIntensity={0.3} rotationIntensity={0.2}>
      <Center>
        <Text
          fontSize={1.5}
          color="white"
          anchorX="center"
          anchorY="middle"
          position={[0, 4, 0]}
        >
          DREI SHOWCASE
          <meshStandardMaterial
            emissive="white"
            emissiveIntensity={0.2}
          />
        </Text>
      </Center>
    </Float>
  );
}

// Distorted Hero Sphere
function HeroSphere() {
  const meshRef = useRef();
  
  return (
    <Float speed={1.5} floatIntensity={0.5}>
      <mesh ref={meshRef} position={[0, 0, 0]}>
        <sphereGeometry args={[1.5, 64, 64]} />
        <MeshDistortMaterial
          color="#4ecdc4"
          distort={0.5}
          speed={2}
          roughness={0}
          metalness={0.5}
        />
        <Sparkles
          count={80}
          scale={4}
          size={3}
          speed={0.3}
          color="cyan"
          opacity={0.5}
        />
      </mesh>
    </Float>
  );
}

// Info Labels
function InfoLabel({ position, title, description }) {
  const [hovered, setHovered] = useState(false);
  
  return (
    <group position={position}>
      <Billboard>
        <Html distanceFactor={6}>
          <div style={{
            background: 'rgba(0,0,0,0.85)',
            color: 'white',
            padding: '12px 16px',
            borderRadius: '10px',
            fontSize: '12px',
            width: '160px',
            border: `1px solid ${hovered ? '#4ecdc4' : 'rgba(255,255,255,0.1)'}`,
            transition: 'border-color 0.3s',
            backdropFilter: 'blur(10px)',
            cursor: 'default'
          }}
          onMouseEnter={() => setHovered(true)}
          onMouseLeave={() => setHovered(false)}
          >
            <div style={{ 
              color: '#4ecdc4', 
              fontWeight: 'bold',
              marginBottom: '6px',
              fontSize: '13px'
            }}>
              {title}
            </div>
            <div style={{ color: '#aaa', lineHeight: 1.5 }}>
              {description}
            </div>
          </div>
        </Html>
      </Billboard>
      
      {/* Indicator dot */}
      <mesh>
        <sphereGeometry args={[0.05]} />
        <meshBasicMaterial color={hovered ? '#4ecdc4' : 'white'} />
      </mesh>
    </group>
  );
}

// Wobble Torus Display
function WobbleTorus({ position }) {
  return (
    <Float speed={2} floatIntensity={0.4} position={position}>
      <mesh>
        <torusGeometry args={[0.8, 0.3, 16, 100]} />
        <MeshWobbleMaterial
          color="#ff6b6b"
          factor={0.5}
          speed={3}
          roughness={0.2}
          metalness={0.3}
        />
      </mesh>
    </Float>
  );
}

// Animated Star Points
function AnimatedStars() {
  const starsRef = useRef();
  
  useFrame((state) => {
    if (starsRef.current) {
      starsRef.current.rotation.y = state.clock.elapsedTime * 0.05;
    }
  });

  return (
    <group ref={starsRef}>
      <Stars
        radius={100}
        depth={50}
        count={3000}
        factor={4}
        saturation={0}
        fade
        speed={1}
      />
    </group>
  );
}

// Cloud Layer
function CloudLayer() {
  return (
    <Clouds>
      <Cloud
        position={[0, 8, -20]}
        speed={0.2}
        opacity={0.3}
        bounds={[15, 2, 2]}
        volume={10}
        color="white"
        segments={40}
      />
      <Cloud
        position={[-10, 6, -25]}
        speed={0.15}
        opacity={0.2}
        bounds={[10, 2, 2]}
        volume={8}
        color="#e0e0ff"
      />
    </Clouds>
  );
}

// Display Pedestal
function Pedestal({ position, children, color = '#2c3e50' }) {
  return (
    <group position={position}>
      {/* Stand */}
      <mesh position={[0, -1.5, 0]} castShadow receiveShadow>
        <cylinderGeometry args={[0.5, 0.7, 1, 32]} />
        <meshStandardMaterial color={color} metalness={0.5} roughness={0.3} />
      </mesh>
      
      {/* Platform */}
      <mesh position={[0, -1, 0]} castShadow receiveShadow>
        <cylinderGeometry args={[0.8, 0.8, 0.1, 32]} />
        <meshStandardMaterial color={color} metalness={0.8} roughness={0.1} />
      </mesh>
      
      {/* Content */}
      {children}
    </group>
  );
}

// Main Showcase
export default function DreiShowcase() {
  const [activeSection, setActiveSection] = useState('all');
  
  const sections = ['all', 'materials', 'effects', 'text'];

  return (
    <div style={{ width: '100vw', height: '100vh', position: 'relative' }}>
      
      {/* Navigation */}
      <div style={{
        position: 'absolute',
        top: 20,
        left: '50%',
        transform: 'translateX(-50%)',
        display: 'flex',
        gap: 8,
        zIndex: 10,
        background: 'rgba(0,0,0,0.5)',
        padding: '8px',
        borderRadius: 12,
        backdropFilter: 'blur(10px)'
      }}>
        {sections.map(s => (
          <button
            key={s}
            onClick={() => setActiveSection(s)}
            style={{
              padding: '8px 18px',
              background: activeSection === s 
                ? 'linear-gradient(135deg, #4ecdc4, #45b7d1)' 
                : 'transparent',
              border: 'none',
              borderRadius: 8,
              color: 'white',
              cursor: 'pointer',
              textTransform: 'capitalize',
              fontFamily: 'sans-serif',
              fontWeight: 'bold',
              fontSize: '0.85em'
            }}
          >
            {s}
          </button>
        ))}
      </div>

      <Canvas shadows camera={{ position: [0, 2, 12], fov: 60 }}>
        {/* Background */}
        <color attach="background" args={['#050510']} />
        
        {/* Stars */}
        <AnimatedStars />
        
        {/* Clouds */}
        <Suspense fallback={null}>
          <CloudLayer />
        </Suspense>
        
        {/* Environment Lighting */}
        <ambientLight intensity={0.3} />
        <directionalLight position={[5, 10, 5]} intensity={1} castShadow />
        <pointLight position={[-5, 5, -5]} intensity={50} color="#4ecdc4" />
        <pointLight position={[5, 5, 5]} intensity={30} color="#ff6b6b" />
        
        {/* Hero Section */}
        <HeroText />
        <HeroSphere />
        
        {/* Material Pedestals */}
        <Pedestal position={[-5, -1, 0]} color="#1a1a3a">
          <WobbleTorus position={[0, 1, 0]} />
          <InfoLabel
            position={[0, 2.5, 0]}
            title="MeshWobbleMaterial"
            description="Dynamic wobble distortion effect"
          />
        </Pedestal>
        
        <Pedestal position={[5, -1, 0]} color="#1a3a1a">
          <Float speed={1.5} floatIntensity={0.4} position={[0, 0.5, 0]}>
            <mesh>
              <boxGeometry args={[1, 1, 1]} />
              <MeshDistortMaterial
                color="#2ecc71"
                distort={0.6}
                speed={1.5}
                roughness={0}
                metalness={0.8}
              />
            </mesh>
          </Float>
          <InfoLabel
            position={[0, 2.5, 0]}
            title="MeshDistortMaterial"
            description="Vertex distortion with noise"
          />
        </Pedestal>
        
        {/* Contact Shadows */}
        <ContactShadows
          position={[0, -2, 0]}
          opacity={0.6}
          scale={15}
          blur={2}
          far={4}
          resolution={256}
          frames={1}
        />
        
        {/* Ground */}
        <mesh rotation={[-Math.PI / 2, 0, 0]} position={[0, -2, 0]} receiveShadow>
          <planeGeometry args={[30, 30]} />
          <meshStandardMaterial color="#0a0a1a" />
        </mesh>
        
        {/* Camera Controls */}
        <OrbitControls
          enableZoom={true}
          enablePan={false}
          autoRotate={true}
          autoRotateSpeed={0.5}
          maxPolarAngle={Math.PI / 2}
          minDistance={5}
          maxDistance={25}
        />
      </Canvas>
    </div>
  );
}
```

---

## 66.10 โปรเจกต์: Full Scene Using Drei Components

```jsx
// src/DreiFullScene.jsx
import { Canvas, useFrame } from '@react-three/fiber';
import {
  Environment, Float, Text, MeshReflectorMaterial,
  ContactShadows, Stars, OrbitControls, useGLTF,
  Sparkles, Billboard, Html, SoftShadows, useProgress,
  Loader
} from '@react-three/drei';
import { Suspense, useRef, useState } from 'react';
import * as THREE from 'three';

// Loading Screen
function LoadingScreen() {
  const { progress } = useProgress();
  return (
    <Html center>
      <div style={{
        display: 'flex',
        flexDirection: 'column',
        alignItems: 'center',
        gap: 16,
        color: 'white',
        fontFamily: 'sans-serif'
      }}>
        <div style={{
          width: 60,
          height: 60,
          border: '3px solid rgba(255,255,255,0.1)',
          borderTop: '3px solid #4ecdc4',
          borderRadius: '50%',
          animation: 'spin 1s linear infinite'
        }} />
        <div>Loading {Math.round(progress)}%</div>
      </div>
    </Html>
  );
}

// Product Item
function ProductItem({ position, color, name, price }) {
  const [hovered, setHovered] = useState(false);
  const ref = useRef();
  
  return (
    <Float 
      position={position} 
      speed={1 + Math.random()} 
      floatIntensity={0.3}
      rotationIntensity={0.2}
    >
      <group
        onPointerOver={() => { 
          setHovered(true); 
          document.body.style.cursor = 'pointer';
        }}
        onPointerOut={() => { 
          setHovered(false);
          document.body.style.cursor = 'default';
        }}
      >
        {/* Product Mesh */}
        <mesh ref={ref} castShadow>
          <icosahedronGeometry args={[0.7, 1]} />
          <meshStandardMaterial
            color={color}
            metalness={0.8}
            roughness={0.2}
            emissive={hovered ? color : 'black'}
            emissiveIntensity={hovered ? 0.3 : 0}
          />
        </mesh>
        
        {/* Sparkles สำหรับ hovered item */}
        {hovered && (
          <Sparkles
            count={20}
            scale={2}
            size={3}
            speed={0.5}
            color={color}
          />
        )}
        
        {/* Product Label */}
        <Billboard>
          <Html distanceFactor={8} position={[0, 1.2, 0]}>
            <div style={{
              background: hovered ? 'rgba(0,0,0,0.9)' : 'rgba(0,0,0,0.6)',
              color: 'white',
              padding: '6px 12px',
              borderRadius: 6,
              fontSize: '12px',
              textAlign: 'center',
              whiteSpace: 'nowrap',
              transition: 'background 0.3s',
              border: `1px solid ${hovered ? color : 'transparent'}`
            }}>
              <div style={{ fontWeight: 'bold' }}>{name}</div>
              <div style={{ color: '#4ecdc4', marginTop: 2 }}>${price}</div>
            </div>
          </Html>
        </Billboard>
      </group>
    </Float>
  );
}

// Main Full Scene
export default function DreiFullScene() {
  const products = [
    { id: 0, position: [-4, 0.5, 0], color: '#e74c3c', name: 'Aurora Ring', price: '299' },
    { id: 1, position: [-2, 0.5, 0], color: '#3498db', name: 'Nebula Gem', price: '149' },
    { id: 2, position: [0, 0.5, 0], color: '#2ecc71', name: 'Prism Stone', price: '199' },
    { id: 3, position: [2, 0.5, 0], color: '#f39c12', name: 'Solar Orb', price: '349' },
    { id: 4, position: [4, 0.5, 0], color: '#9b59b6', name: 'Cosmic Gem', price: '449' },
  ];

  return (
    <div style={{ width: '100vw', height: '100vh' }}>
      <Canvas shadows camera={{ position: [0, 3, 10], fov: 60 }}>
        {/* Background */}
        <color attach="background" args={['#050510']} />
        
        {/* Stars background */}
        <Stars radius={100} depth={50} count={5000} factor={4} saturation={0} fade />
        
        {/* Soft Shadows */}
        <SoftShadows size={20} samples={8} />
        
        {/* Lighting */}
        <ambientLight intensity={0.3} />
        <directionalLight position={[5, 10, 5]} intensity={1} castShadow />
        <pointLight position={[-10, 5, 0]} intensity={100} color="#4ecdc4" />
        
        {/* Environment for reflections */}
        <Suspense fallback={null}>
          <Environment preset="night" />
        </Suspense>
        
        {/* Hero Title */}
        <Float speed={0.5} floatIntensity={0.1} position={[0, 4, 0]}>
          <Text
            fontSize={1.2}
            color="white"
            anchorX="center"
            letterSpacing={0.1}
          >
            COSMIC GEMS
          </Text>
        </Float>
        
        <Float speed={0.5} floatIntensity={0.1} position={[0, 3, 0]}>
          <Text
            fontSize={0.35}
            color="#4ecdc4"
            anchorX="center"
            letterSpacing={0.2}
          >
            EXCLUSIVE COLLECTION 2024
          </Text>
        </Float>
        
        {/* Product Items */}
        <Suspense fallback={<LoadingScreen />}>
          {products.map(p => (
            <ProductItem key={p.id} {...p} />
          ))}
        </Suspense>
        
        {/* Contact Shadows */}
        <ContactShadows
          position={[0, -0.5, 0]}
          opacity={0.7}
          scale={15}
          blur={2}
          frames={Infinity}
        />
        
        {/* Reflective Floor */}
        <mesh rotation={[-Math.PI / 2, 0, 0]} position={[0, -0.5, 0]}>
          <planeGeometry args={[30, 30]} />
          <MeshReflectorMaterial
            blur={[300, 100]}
            resolution={2048}
            mixBlur={1}
            mixStrength={40}
            roughness={0.8}
            depthScale={1.2}
            minDepthThreshold={0.4}
            maxDepthThreshold={1.4}
            color="#050510"
            metalness={0.5}
          />
        </mesh>
        
        {/* Camera Controls */}
        <OrbitControls
          enableZoom={true}
          enablePan={false}
          autoRotate={true}
          autoRotateSpeed={0.3}
          maxPolarAngle={Math.PI / 2.2}
          minDistance={5}
          maxDistance={20}
          target={[0, 0.5, 0]}
        />
      </Canvas>
      
      {/* Loading overlay */}
      <Loader />
    </div>
  );
}
```

---

## 66.11 Custom Drei-like Hooks

### useScroll-like Animation

```jsx
import { useRef, useEffect, useState } from 'react';
import { useFrame } from '@react-three/fiber';

// Custom hook สำหรับ scroll-based animation ใน R3F
function useScrollProgress() {
  const scrollY = useRef(0);
  const target = useRef(0);
  const current = useRef(0);

  useEffect(() => {
    const handleScroll = () => {
      const maxScroll = document.documentElement.scrollHeight - window.innerHeight;
      scrollY.current = window.scrollY / maxScroll;
    };
    window.addEventListener('scroll', handleScroll);
    return () => window.removeEventListener('scroll', handleScroll);
  }, []);

  useFrame(() => {
    current.current += (scrollY.current - current.current) * 0.1;
  });

  return current;
}

// Component ที่ animate ตาม scroll
function ScrollAnimatedObject() {
  const ref = useRef();
  const progress = useScrollProgress();

  useFrame(() => {
    if (!ref.current) return;
    ref.current.rotation.y = progress.current * Math.PI * 4;
    ref.current.position.y = Math.sin(progress.current * Math.PI) * 3;
    ref.current.scale.setScalar(1 + progress.current * 0.5);
  });

  return (
    <mesh ref={ref}>
      <boxGeometry />
      <meshStandardMaterial color="royalblue" />
    </mesh>
  );
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Sky & Environment
สร้าง outdoor scene ที่มี:
- Sky component ที่ปรับได้ (time of day slider)
- Stars ที่ปรากฏเมื่อตกกลางคืน
- Clouds ที่เคลื่อนที่
- Environment lighting ที่เปลี่ยนตามเวลา

### แบบฝึกหัดที่ 2: Showroom
สร้าง product showroom ที่มี:
- MeshReflectorMaterial พื้น
- Float effects ทุก product
- ContactShadows ที่สวยงาม
- Billboard labels ทุก product
- OrbitControls พร้อม smooth damping

### แบบฝึกหัดที่ 3: Material Gallery
สร้าง gallery สำหรับทดสอบ materials:
- MeshDistortMaterial, MeshWobbleMaterial
- Environment reflections
- Controls สำหรับปรับ parameters
- Animation preview

### แบบฝึกหัดที่ 4: Animated Text Scene
สร้าง text scene ที่มี:
- Text3D title ด้วย bevel
- Text labels ด้วย Float
- HTML overlays ด้วย Html component
- Billboard text ที่หันหา camera

### แบบฝึกหัดที่ 5: Particle Constellation
สร้าง particle constellation ด้วย:
- Sparkles components หลายกลุ่ม
- Stars background
- Float effects
- Color-changing particles ตาม time

---

## สรุป

ในบทนี้เราเรียนรู้ Drei components สำคัญ:
1. **Text Components** - Html, Text, Text3D สำหรับ text ใน 3D
2. **Environment** - Sky, Stars, Cloud, Environment
3. **Camera Controls** - OrbitControls, CameraControls, PerspectiveCamera
4. **Materials** - MeshReflectorMaterial, MeshDistortMaterial, MeshWobbleMaterial
5. **Loaders** - useGLTF, useTexture, useAnimations
6. **Effects** - Float, Billboard, ContactShadows, SoftShadows, Sparkles
7. **Performance** - Instances, Merged

บทถัดไปเราจะเรียน **Three.js กับ Vue.js**
