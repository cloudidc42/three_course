# Part 003: Geometries Deep Dive - ทำความรู้จัก Geometry ทุกชนิด

## สารบัญบทเรียนนี้
- [1. BufferGeometry คืออะไร?](#1-buffergeometry-คืออะไร)
- [2. Built-in Geometries ทั้งหมด](#2-built-in-geometries-ทั้งหมด)
- [3. BoxGeometry](#3-boxgeometry)
- [4. SphereGeometry](#4-spheregeometry)
- [5. PlaneGeometry](#5-planegeometry)
- [6. CylinderGeometry & ConeGeometry](#6-cylindergeometry--conegeometry)
- [7. TorusGeometry & TorusKnotGeometry](#7-torusgeometry--torusknotgeometry)
- [8. Polyhedral Geometries](#8-polyhedral-geometries)
- [9. RingGeometry & CircleGeometry](#9-ringgeometry--circlegeometry)
- [10. TubeGeometry & LatheGeometry](#10-tubegeometry--lathegeometry)
- [11. ShapeGeometry & ExtrudeGeometry](#11-shapegeometry--extrudegeometry)
- [12. EdgesGeometry & WireframeGeometry](#12-edgesgeometry--wireframegeometry)
- [13. UV Coordinates](#13-uv-coordinates)
- [14. Gallery Demo สมบูรณ์](#14-gallery-demo-สมบูรณ์)
- [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. BufferGeometry คืออะไร?

ทุก geometry ใน Three.js สืบทอดมาจาก `THREE.BufferGeometry`

### หลักการทำงาน

Geometry ใน Three.js เก็บข้อมูลในรูปแบบ **Buffer** (typed arrays ที่ส่งไปยัง GPU โดยตรง)

```
BufferGeometry
    ├── attributes
    │   ├── position: Float32Array  (x, y, z สำหรับแต่ละ vertex)
    │   ├── normal: Float32Array    (normal vector สำหรับ lighting)
    │   ├── uv: Float32Array        (texture coordinates)
    │   └── color: Float32Array     (vertex colors - optional)
    └── index: Uint16Array/Uint32Array  (indices สำหรับ indexed rendering)
```

### สร้าง Custom BufferGeometry

```javascript
// สร้าง triangle เดี่ยวด้วยมือ
const geometry = new THREE.BufferGeometry();

// Vertex positions (x, y, z สำหรับแต่ละ vertex)
const vertices = new Float32Array([
    // Vertex 1
    0.0,  1.0, 0.0,  // บน
    // Vertex 2
   -1.0, -1.0, 0.0,  // ล่างซ้าย
    // Vertex 3
    1.0, -1.0, 0.0,  // ล่างขวา
]);

// สร้าง BufferAttribute และกำหนดให้ geometry
const positionAttribute = new THREE.BufferAttribute(
    vertices,  // data
    3          // itemSize: 3 values per vertex (x, y, z)
);
geometry.setAttribute('position', positionAttribute);

// Normal vectors (สำหรับ lighting)
const normals = new Float32Array([
    0, 0, 1,  // vertex 1 normal: ชี้ออกมาข้างหน้า
    0, 0, 1,  // vertex 2 normal
    0, 0, 1,  // vertex 3 normal
]);
geometry.setAttribute('normal', new THREE.BufferAttribute(normals, 3));

// UV Coordinates (สำหรับ texture mapping)
const uvs = new Float32Array([
    0.5, 1.0,  // vertex 1 UV
    0.0, 0.0,  // vertex 2 UV
    1.0, 0.0,  // vertex 3 UV
]);
geometry.setAttribute('uv', new THREE.BufferAttribute(uvs, 2));

// สร้าง mesh
const material = new THREE.MeshStandardMaterial({ color: 0x6c63ff, side: THREE.DoubleSide });
const triangle = new THREE.Mesh(geometry, material);
scene.add(triangle);
```

### Indexed vs Non-indexed Geometry

```javascript
// Non-indexed: แต่ละ face เก็บ vertices ของตัวเอง (ซ้ำกัน)
// ใช้ memory มากกว่า แต่ vertex แต่ละตัวสามารถมี UV/Normal ต่างกัน

// Indexed: หลาย face share vertices กัน (ประหยัด memory)
const indexedGeometry = new THREE.BufferGeometry();

const indexedVertices = new Float32Array([
    -1, -1, 0,  // vertex 0
     1, -1, 0,  // vertex 1
     1,  1, 0,  // vertex 2
    -1,  1, 0,  // vertex 3
]);
indexedGeometry.setAttribute('position', new THREE.BufferAttribute(indexedVertices, 3));

// Index: กำหนดว่าแต่ละ triangle ใช้ vertices ไหน
const indices = new Uint16Array([
    0, 1, 2,  // triangle 1: vertex 0, 1, 2
    0, 2, 3,  // triangle 2: vertex 0, 2, 3
]);
indexedGeometry.setIndex(new THREE.BufferAttribute(indices, 1));

// คำนวณ normals อัตโนมัติ
indexedGeometry.computeVertexNormals();
```

### BufferGeometry Utilities

```javascript
const geometry = new THREE.BoxGeometry(1, 1, 1);

// คำนวณ bounding box
geometry.computeBoundingBox();
console.log(geometry.boundingBox);
// { min: Vector3(-0.5, -0.5, -0.5), max: Vector3(0.5, 0.5, 0.5) }

// คำนวณ bounding sphere
geometry.computeBoundingSphere();
console.log(geometry.boundingSphere);
// { center: Vector3(0, 0, 0), radius: 0.866... }

// Center geometry (เลื่อน vertices ให้ center อยู่ที่ origin)
geometry.center();

// Scale geometry
geometry.scale(2, 1, 1); // ขยาย X สองเท่า

// Translate geometry
geometry.translate(0, 0.5, 0); // เลื่อนขึ้น 0.5

// Rotate geometry
geometry.rotateX(Math.PI / 4); // หมุน 45 องศารอบ X

// Dispose (ล้าง memory เมื่อไม่ใช้แล้ว)
geometry.dispose();

// Clone
const geometryCopy = geometry.clone();

// Merge geometries (รวมหลาย geometry เป็นอันเดียว)
// จะเรียนใน Part เรื่อง Performance
```

---

## 2. Built-in Geometries ทั้งหมด

Three.js มี built-in geometries ดังนี้:

| Geometry | Description |
|----------|-------------|
| BoxGeometry | กล่อง/ลูกบาศก์ |
| SphereGeometry | ลูกกลม |
| PlaneGeometry | แผ่นแบน |
| CylinderGeometry | ทรงกระบอก |
| ConeGeometry | ทรงกรวย |
| TorusGeometry | โดนัท |
| TorusKnotGeometry | โดนัทบิด |
| DodecahedronGeometry | ทรง 12 หน้า |
| IcosahedronGeometry | ทรง 20 หน้า |
| OctahedronGeometry | ทรง 8 หน้า |
| TetrahedronGeometry | ทรง 4 หน้า |
| RingGeometry | วงแหวน |
| CircleGeometry | วงกลม |
| ShapeGeometry | รูปทรงจาก Path 2D |
| TubeGeometry | ท่อตาม Curve |
| LatheGeometry | รูปทรงจากการหมุน profile |
| ExtrudeGeometry | รูปทรงจากการ extrude |
| EdgesGeometry | เส้นขอบของ geometry |
| WireframeGeometry | wireframe ของ geometry |

---

## 3. BoxGeometry

```javascript
// BoxGeometry(width, height, depth, widthSeg, heightSeg, depthSeg)
const box = new THREE.BoxGeometry(
    2,    // width: ความกว้าง (X)
    1,    // height: ความสูง (Y)
    1.5,  // depth: ความลึก (Z)
    4,    // widthSegments: แบ่งกี่ส่วนตาม X
    2,    // heightSegments: แบ่งกี่ส่วนตาม Y
    3     // depthSegments: แบ่งกี่ส่วนตาม Z
);

// ตัวอย่างการใช้งาน segments
// Segments มากขึ้น = รายละเอียดมากขึ้น = polygon มากขึ้น
// ใช้เมื่อ:
// 1. ต้องการ displacement map (เขยิบ vertices ตาม texture)
// 2. ต้องการทำ deformation animation
// 3. ต้องการ wireframe ที่มีรายละเอียด

// Simple box (6 faces, 12 triangles)
const simpleBox = new THREE.BoxGeometry(1, 1, 1);
console.log('Simple - vertices:', simpleBox.attributes.position.count); // 24

// Detailed box
const detailBox = new THREE.BoxGeometry(1, 1, 1, 4, 4, 4);
console.log('Detail - vertices:', detailBox.attributes.position.count); // 150
```

---

## 4. SphereGeometry

```javascript
// SphereGeometry(radius, widthSeg, heightSeg, phiStart, phiLength, thetaStart, thetaLength)
const sphere = new THREE.SphereGeometry(
    1,           // radius
    32,          // widthSegments (horizontal): มากขึ้น = กลมขึ้น
    16,          // heightSegments (vertical): มากขึ้น = กลมขึ้น
    0,           // phiStart: เริ่ม angle แนวนอน (default: 0)
    Math.PI * 2, // phiLength: ความกว้าง angle แนวนอน (default: full circle)
    0,           // thetaStart: เริ่ม angle แนวตั้ง (default: 0)
    Math.PI      // thetaLength: ความสูง angle แนวตั้ง (default: full)
);

// ตัวอย่างการสร้าง sphere แบบต่างๆ
// Segment ต่ำ = polygon sphere (เห็น facets)
const lowPoly = new THREE.SphereGeometry(1, 8, 4);

// Segment สูง = เกือบกลมสมบูรณ์
const highPoly = new THREE.SphereGeometry(1, 64, 32);

// Half sphere (hemisphere)
const hemisphere = new THREE.SphereGeometry(1, 32, 16, 0, Math.PI * 2, 0, Math.PI / 2);

// Quarter sphere
const quarter = new THREE.SphereGeometry(1, 32, 16, 0, Math.PI, 0, Math.PI / 2);

// "Pac-man" slice
const pacman = new THREE.SphereGeometry(1, 32, 32, 0, Math.PI * 1.5);
```

---

## 5. PlaneGeometry

```javascript
// PlaneGeometry(width, height, widthSegments, heightSegments)
const plane = new THREE.PlaneGeometry(
    10,   // width
    10,   // height
    20,   // widthSegments
    20    // heightSegments
);

// Default orientation: ชี้ไปทาง Z (vertical)
// เพื่อใช้เป็นพื้น ต้องหมุน 90 องศา:
plane.rotateX(-Math.PI / 2); // หมุน geometry เอง
// หรือ
const planeMesh = new THREE.Mesh(plane, material);
planeMesh.rotation.x = -Math.PI / 2; // หมุน mesh

// PlaneGeometry for terrain (displacement)
const terrainPlane = new THREE.PlaneGeometry(100, 100, 128, 128);
// Segments มากขึ้น = terrain detail มากขึ้นเมื่อใช้ displacementMap
```

---

## 6. CylinderGeometry & ConeGeometry

```javascript
// CylinderGeometry(radiusTop, radiusBottom, height, radialSeg, heightSeg, openEnded, thetaStart, thetaLength)
const cylinder = new THREE.CylinderGeometry(
    0.5,  // radiusTop
    0.5,  // radiusBottom
    2,    // height
    32,   // radialSegments
    4,    // heightSegments
    false // openEnded: false = มีฝาปิด, true = ท่อเปิด
);

// Cylinder แบบต่างๆ
// Cone: radiusTop = 0
const cone = new THREE.CylinderGeometry(0, 1, 2, 32);

// Truncated cone (ทรงกรวยตัด)
const truncCone = new THREE.CylinderGeometry(0.5, 1, 2, 32);

// Open cylinder (ท่อ)
const pipe = new THREE.CylinderGeometry(1, 1, 3, 32, 1, true);

// Hexagon prism (ปริซึมหกเหลี่ยม) - ใช้ radialSegments = 6
const hexPrism = new THREE.CylinderGeometry(1, 1, 1, 6);

// ConeGeometry (wrapper สำหรับ CylinderGeometry ที่ top=0)
const cone2 = new THREE.ConeGeometry(
    1,    // radius
    2,    // height
    32,   // radialSegments
    4,    // heightSegments
    false // openEnded
);

// Slice ของ Cylinder
const slice = new THREE.CylinderGeometry(1, 1, 1, 32, 1, false, 0, Math.PI); // ครึ่งวงกลม
```

---

## 7. TorusGeometry & TorusKnotGeometry

```javascript
// TorusGeometry(radius, tube, radialSegments, tubularSegments, arc)
const torus = new THREE.TorusGeometry(
    1,           // radius: ระยะจากจุดกลางถึงกลางท่อ
    0.4,         // tube: รัศมีของท่อ
    16,          // radialSegments: segments รอบวงแหวน
    100,         // tubularSegments: segments ตลอดท่อ
    Math.PI * 2  // arc: ความยาว arc (default: full circle)
);

// Torus แบบต่างๆ
const thinTorus = new THREE.TorusGeometry(1, 0.1, 16, 100);      // แหวนบาง
const fatTorus = new THREE.TorusGeometry(1, 0.8, 16, 100);        // โดนัทอ้วน
const halfTorus = new THREE.TorusGeometry(1, 0.4, 16, 100, Math.PI); // ครึ่งโดนัท

// TorusKnotGeometry(radius, tube, tubularSeg, radialSeg, p, q)
const torusKnot = new THREE.TorusKnotGeometry(
    0.8,  // radius
    0.35, // tube
    100,  // tubularSegments
    16,   // radialSegments
    2,    // p: windings รอบแกนหลัก
    3     // q: windings รอบ torus
);

// TorusKnot patterns ที่น่าสนใจ
const knot1 = new THREE.TorusKnotGeometry(0.8, 0.35, 200, 16, 2, 3);  // classic
const knot2 = new THREE.TorusKnotGeometry(0.8, 0.35, 200, 16, 3, 5);  // complex
const knot3 = new THREE.TorusKnotGeometry(0.8, 0.35, 200, 16, 4, 3);
const knot4 = new THREE.TorusKnotGeometry(0.8, 0.35, 200, 16, 5, 2);  // star-like
const knot5 = new THREE.TorusKnotGeometry(0.8, 0.35, 200, 16, 2, 5);  // trefoil
```

---

## 8. Polyhedral Geometries

Three.js มี platonic solids ครบทั้งหมด:

```javascript
// TetrahedronGeometry - ทรงสี่หน้า (4 faces, 4 vertices)
const tetra = new THREE.TetrahedronGeometry(
    1,  // radius
    0   // detail: 0=4 faces, 1=16 faces, 2=64 faces
);

// OctahedronGeometry - ทรงแปดหน้า (8 faces, 6 vertices)
const octa = new THREE.OctahedronGeometry(1, 0);

// DodecahedronGeometry - ทรงสิบสองหน้า (12 faces, 20 vertices)
const dodeca = new THREE.DodecahedronGeometry(1, 0);

// IcosahedronGeometry - ทรงยี่สิบหน้า (20 faces, 12 vertices)
const icosa = new THREE.IcosahedronGeometry(1, 0);

// "detail" parameter ทำให้ subdivide faces
// detail=0: icosahedron ดั้งเดิม (20 triangles)
// detail=1: subdivide ทุก triangle เป็น 4 (80 triangles)
// detail=2: 320 triangles
// detail=3: 1280 triangles
// detail สูงขึ้น = กลมขึ้น (แต่ polygon มากขึ้น)

// ใช้ IcosahedronGeometry แทน SphereGeometry ได้
// ข้อดี: face distribution สม่ำเสมอกว่า
// ข้อเสีย: UV mapping ยุ่งยากกว่า
const icosphereDetail3 = new THREE.IcosahedronGeometry(1, 3);

// Low-poly sphere ด้วย IcosahedronGeometry
const lowPolySphere = new THREE.IcosahedronGeometry(1, 1); // 80 faces
```

---

## 9. RingGeometry & CircleGeometry

```javascript
// CircleGeometry(radius, segments, thetaStart, thetaLength)
const circle = new THREE.CircleGeometry(
    1,           // radius
    32,          // segments
    0,           // thetaStart
    Math.PI * 2  // thetaLength
);

// Pie chart slice
const slice = new THREE.CircleGeometry(1, 32, 0, Math.PI / 2); // 1/4 of circle

// RingGeometry(innerRadius, outerRadius, thetaSegments, phiSegments, thetaStart, thetaLength)
const ring = new THREE.RingGeometry(
    0.5,         // innerRadius
    1,           // outerRadius
    32,          // thetaSegments (แนวนอน)
    1,           // phiSegments (แนวรัศมี)
    0,           // thetaStart
    Math.PI * 2  // thetaLength
);

// Ring arc
const arcRing = new THREE.RingGeometry(0.5, 1, 32, 1, 0, Math.PI); // ครึ่งวง

// Thick ring (phiSegments มากขึ้น)
const thickRing = new THREE.RingGeometry(0.3, 1, 32, 4);

// วางแนวนอน
ring.rotateX(-Math.PI / 2);
```

---

## 10. TubeGeometry & LatheGeometry

### TubeGeometry - ท่อตาม Curve

```javascript
// ต้องมี Curve ก่อน
// Curve types: CatmullRomCurve3, CubicBezierCurve3, QuadraticBezierCurve3, LineCurve3

// สร้าง 3D curve ด้วย CatmullRomCurve3 (smooth curve ผ่าน control points)
const path = new THREE.CatmullRomCurve3([
    new THREE.Vector3(-2, 0, 0),
    new THREE.Vector3(-1, 1, 1),
    new THREE.Vector3(0, -1, 2),
    new THREE.Vector3(1, 1, 1),
    new THREE.Vector3(2, 0, 0),
]);

// TubeGeometry(path, tubularSegments, radius, radialSegments, closed)
const tube = new THREE.TubeGeometry(
    path,   // curve
    64,     // tubularSegments: ความละเอียดตามท่อ
    0.2,    // radius: รัศมีท่อ
    8,      // radialSegments: ความละเอียดรอบท่อ
    false   // closed: ปิดท่อหรือไม่
);

// Tube ปิดเป็นวง
const closedCurve = new THREE.CatmullRomCurve3([
    new THREE.Vector3(1, 0, 0),
    new THREE.Vector3(0, 1, 0),
    new THREE.Vector3(-1, 0, 0),
    new THREE.Vector3(0, -1, 0),
], true); // true = closed

const closedTube = new THREE.TubeGeometry(closedCurve, 100, 0.2, 8, true);

// Bezier Curve
const bezierCurve = new THREE.CubicBezierCurve3(
    new THREE.Vector3(-2, 0, 0),  // start
    new THREE.Vector3(-1, 2, 0),  // control 1
    new THREE.Vector3(1, 2, 0),   // control 2
    new THREE.Vector3(2, 0, 0)    // end
);
const bezierTube = new THREE.TubeGeometry(bezierCurve, 64, 0.1, 8, false);
```

### LatheGeometry - รูปทรงจากการหมุน Profile

```javascript
// LatheGeometry สร้าง solid of revolution
// กำหนด 2D profile แล้วหมุนรอบแกน Y

// สร้าง points สำหรับ profile
const points = [];
for (let i = 0; i < 10; i++) {
    const t = i / 9;
    // รูปแบบ vase
    const x = Math.sin(t * Math.PI) * 0.5 + 0.3;
    const y = t * 2 - 1;
    points.push(new THREE.Vector2(x, y));
}

// LatheGeometry(points, segments, phiStart, phiLength)
const vase = new THREE.LatheGeometry(
    points, // profile points (Vector2 array)
    32,     // segments: ความละเอียดการหมุน
    0,      // phiStart: เริ่มที่ 0 radian
    Math.PI * 2 // phiLength: หมุนครบ 360°
);

// Half lathe (ครึ่งวง)
const halfVase = new THREE.LatheGeometry(points, 32, 0, Math.PI);

// สร้าง profile ที่ซับซ้อนกว่า (เช่น โอ่ง, ขวด)
const bottlePoints = [];
// neck
bottlePoints.push(new THREE.Vector2(0.1, 1.5));
bottlePoints.push(new THREE.Vector2(0.15, 1.2));
bottlePoints.push(new THREE.Vector2(0.3, 1.0));
// shoulder
bottlePoints.push(new THREE.Vector2(0.5, 0.8));
// body
bottlePoints.push(new THREE.Vector2(0.6, 0.0));
bottlePoints.push(new THREE.Vector2(0.6, -0.8));
// bottom
bottlePoints.push(new THREE.Vector2(0.4, -1.0));
bottlePoints.push(new THREE.Vector2(0.0, -1.1));

const bottle = new THREE.LatheGeometry(bottlePoints, 32);
```

---

## 11. ShapeGeometry & ExtrudeGeometry

### ShapeGeometry - รูปทรง 2D จาก Path

```javascript
// THREE.Shape ใช้ same API กับ Canvas Path2D
const heartShape = new THREE.Shape();

// สร้างรูปหัวใจ
const x = 0, y = 0;
heartShape.moveTo(x + 0.5, y + 0.5);
heartShape.bezierCurveTo(x + 0.5, y + 0.5, x + 0.4, y, x, y);
heartShape.bezierCurveTo(x - 0.6, y, x - 0.6, y + 0.7, x - 0.6, y + 0.7);
heartShape.bezierCurveTo(x - 0.6, y + 1.1, x - 0.35, y + 1.54, x + 0.5, y + 1.9);
heartShape.bezierCurveTo(x + 1.56, y + 1.54, x + 1.6, y + 1.1, x + 1.6, y + 0.7);
heartShape.bezierCurveTo(x + 1.6, y + 0.7, x + 1.6, y, x + 1.0, y);
heartShape.bezierCurveTo(x + 0.7, y, x + 0.5, y + 0.5, x + 0.5, y + 0.5);

// ShapeGeometry จาก Shape
const heartGeo = new THREE.ShapeGeometry(heartShape);
const heart = new THREE.Mesh(heartGeo, new THREE.MeshBasicMaterial({ 
    color: 0xff0000, 
    side: THREE.DoubleSide 
}));
scene.add(heart);

// Shape ที่มีรู (Holes)
const squareWithHole = new THREE.Shape();
squareWithHole.moveTo(-1, -1);
squareWithHole.lineTo(1, -1);
squareWithHole.lineTo(1, 1);
squareWithHole.lineTo(-1, 1);
squareWithHole.closePath();

// สร้างรู
const hole = new THREE.Path();
hole.moveTo(-0.5, -0.5);
hole.lineTo(0.5, -0.5);
hole.lineTo(0.5, 0.5);
hole.lineTo(-0.5, 0.5);
hole.closePath();

squareWithHole.holes.push(hole); // เพิ่มรู

const squareGeo = new THREE.ShapeGeometry(squareWithHole);

// หลาย shapes พร้อมกัน
const shapes = [heartShape, squareWithHole];
const multiGeo = new THREE.ShapeGeometry(shapes);
```

### ExtrudeGeometry - ดึง 2D Shape ขึ้น 3D

```javascript
// ExtrudeGeometry เปลี่ยน Shape 2D ให้เป็น 3D โดยการ extrude ขึ้น

// สร้าง Shape รูปดาว
function createStarShape(outerRadius, innerRadius, points) {
    const shape = new THREE.Shape();
    for (let i = 0; i < points * 2; i++) {
        const angle = (i * Math.PI) / points - Math.PI / 2;
        const radius = i % 2 === 0 ? outerRadius : innerRadius;
        if (i === 0) {
            shape.moveTo(Math.cos(angle) * radius, Math.sin(angle) * radius);
        } else {
            shape.lineTo(Math.cos(angle) * radius, Math.sin(angle) * radius);
        }
    }
    shape.closePath();
    return shape;
}

const starShape = createStarShape(1, 0.4, 5);

// Extrude settings
const extrudeSettings = {
    depth: 0.5,           // ความลึกที่ extrude
    bevelEnabled: true,   // เปิด bevel (มุมมน)
    bevelSegments: 3,     // ความละเอียดของ bevel
    bevelSize: 0.1,       // ขนาด bevel
    bevelThickness: 0.1,  // ความหนา bevel
    bevelOffset: 0,       // offset
    steps: 1,             // แบ่ง depth กี่ส่วน (ใช้สำหรับ UVs)
    
    // หรือกำหนด extrude path (ดึงตาม curve)
    // extrudePath: someCurve, 
};

const starGeo = new THREE.ExtrudeGeometry(starShape, extrudeSettings);
const star = new THREE.Mesh(
    starGeo, 
    new THREE.MeshStandardMaterial({ color: 0xffd700, metalness: 0.5, roughness: 0.3 })
);
scene.add(star);

// Text เหมือน 3D embossed
const letterShape = new THREE.Shape();
letterShape.moveTo(0, 0);
letterShape.lineTo(0.6, 1);
letterShape.lineTo(1.2, 0);
// ... (ซับซ้อนมาก ใน Part 091 เราจะเรียน TextGeometry)
```

---

## 12. EdgesGeometry & WireframeGeometry

```javascript
// EdgesGeometry - เส้นขอบ (เฉพาะขอบที่ชัดเจน)
const boxGeo = new THREE.BoxGeometry(1, 1, 1);
const edgesGeo = new THREE.EdgesGeometry(boxGeo);
const edgesMesh = new THREE.LineSegments(
    edgesGeo,
    new THREE.LineBasicMaterial({ color: 0xffffff })
);
scene.add(edgesMesh);

// EdgesGeometry กับ threshold angle
// เฉพาะขอบที่ angle ระหว่าง faces > threshold เท่านั้นที่จะแสดง
const sphereGeo = new THREE.SphereGeometry(1, 16, 8);
const edgesGeo2 = new THREE.EdgesGeometry(sphereGeo, 15); // 15 degrees threshold
const edgesMesh2 = new THREE.LineSegments(
    edgesGeo2,
    new THREE.LineBasicMaterial({ color: 0x00ffff })
);

// WireframeGeometry - wireframe ทั้งหมด (รวม internal edges)
const wireframeGeo = new THREE.WireframeGeometry(boxGeo);
const wireframeMesh = new THREE.LineSegments(
    wireframeGeo,
    new THREE.LineBasicMaterial({ color: 0xffffff, transparent: true, opacity: 0.3 })
);

// วิธีที่ง่ายกว่า: wireframe บน Material
const material = new THREE.MeshBasicMaterial({ 
    color: 0x00ff00,
    wireframe: true 
});

// หรือ WireframeGeometry
const wire = new THREE.WireframeGeometry(new THREE.SphereGeometry(1, 32, 16));
const wireLines = new THREE.LineSegments(wire, new THREE.LineBasicMaterial({ color: 0xffffff }));
```

---

## 13. UV Coordinates

UV Coordinates คือพิกัด 2D ที่ map กับ texture  
U = แนวนอน (0 ถึง 1), V = แนวตั้ง (0 ถึง 1)

```javascript
// ดู UV ของ geometry
const geo = new THREE.BoxGeometry(1, 1, 1);
const uvAttribute = geo.attributes.uv;

for (let i = 0; i < uvAttribute.count; i++) {
    const u = uvAttribute.getX(i);
    const v = uvAttribute.getY(i);
    console.log(`UV ${i}: (${u.toFixed(2)}, ${v.toFixed(2)})`);
}

// แก้ไข UV ด้วยมือ
// ตัวอย่าง: ทำให้ texture ขยายเป็น 2x
const uvs = geo.attributes.uv;
for (let i = 0; i < uvs.count; i++) {
    uvs.setXY(i, uvs.getX(i) * 2, uvs.getY(i) * 2);
}
uvs.needsUpdate = true;

// Second UV channel (สำหรับ AO map, Lightmap)
// ส่วนใหญ่ model ที่โหลดมาจาก Blender จะมี UV2 อยู่แล้ว
const uv2 = geo.attributes.uv1; // Three.js ใช้ uv1 สำหรับ UV2

// เพิ่ม UV2 เองถ้าไม่มี
geo.setAttribute('uv1', geo.attributes.uv.clone());
```

---

## 14. Gallery Demo สมบูรณ์

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Part 003: Geometry Gallery</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        html, body { width: 100%; height: 100%; overflow: hidden; background: #000; }
        
        #info {
            position: fixed;
            top: 0;
            left: 0;
            right: 0;
            padding: 15px 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
            background: linear-gradient(to bottom, rgba(0,0,0,0.8), transparent);
            color: white;
            font-family: 'Courier New', monospace;
            font-size: 13px;
            pointer-events: none;
            z-index: 10;
        }
        
        #geo-name {
            position: fixed;
            bottom: 30px;
            left: 50%;
            transform: translateX(-50%);
            background: rgba(108, 99, 255, 0.8);
            color: white;
            padding: 8px 20px;
            border-radius: 20px;
            font-family: 'Courier New', monospace;
            font-size: 14px;
            pointer-events: none;
            z-index: 10;
            backdrop-filter: blur(4px);
        }
        
        #controls {
            position: fixed;
            bottom: 80px;
            left: 50%;
            transform: translateX(-50%);
            display: flex;
            gap: 10px;
            z-index: 10;
        }
        
        button {
            background: rgba(108, 99, 255, 0.5);
            border: 1px solid rgba(108, 99, 255, 0.8);
            color: white;
            padding: 8px 16px;
            border-radius: 8px;
            cursor: pointer;
            font-family: 'Courier New', monospace;
            font-size: 12px;
            backdrop-filter: blur(4px);
        }
        button:hover { background: rgba(108, 99, 255, 0.8); }
    </style>
</head>
<body>
    <div id="info">
        <span>Part 003: Geometry Gallery</span>
        <span>← → เปลี่ยน | W: Wireframe | R: Rotate | Space: Pause</span>
    </div>
    
    <div id="geo-name">BoxGeometry</div>
    
    <div id="controls">
        <button id="prevBtn">← Previous</button>
        <button id="wireBtn">Wireframe</button>
        <button id="nextBtn">Next →</button>
    </div>
    
    <script type="importmap">
    {
        "imports": {
            "three": "https://cdn.jsdelivr.net/npm/three@0.165.0/build/three.module.js",
            "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.165.0/examples/jsm/"
        }
    }
    </script>
    
    <script type="module">
        import * as THREE from 'three';
        import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
        import { GUI } from 'three/addons/libs/lil-gui.module.min.js';
        
        // ========== SETUP ==========
        const scene = new THREE.Scene();
        scene.background = new THREE.Color('#0d1117');
        scene.fog = new THREE.FogExp2('#0d1117', 0.04);
        
        const camera = new THREE.PerspectiveCamera(60, innerWidth/innerHeight, 0.1, 100);
        camera.position.set(0, 1, 5);
        
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(innerWidth, innerHeight);
        renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
        renderer.shadowMap.enabled = true;
        renderer.toneMapping = THREE.ACESFilmicToneMapping;
        renderer.toneMappingExposure = 1.2;
        document.body.appendChild(renderer.domElement);
        
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.dampingFactor = 0.05;
        controls.autoRotate = true;
        controls.autoRotateSpeed = 2;
        
        // ========== LIGHTS ==========
        const ambientLight = new THREE.AmbientLight(0xffffff, 0.4);
        scene.add(ambientLight);
        
        const dirLight = new THREE.DirectionalLight(0xffffff, 2);
        dirLight.position.set(5, 5, 5);
        dirLight.castShadow = true;
        scene.add(dirLight);
        
        const pointLight1 = new THREE.PointLight(0x6c63ff, 3, 20);
        pointLight1.position.set(-5, 3, 0);
        scene.add(pointLight1);
        
        const pointLight2 = new THREE.PointLight(0xff6b6b, 3, 20);
        pointLight2.position.set(5, 3, 0);
        scene.add(pointLight2);
        
        // ========== GEOMETRY DEFINITIONS ==========
        
        const geometries = [
            {
                name: 'BoxGeometry',
                description: 'กล่อง/ลูกบาศก์',
                create: () => new THREE.BoxGeometry(1.5, 1.5, 1.5, 3, 3, 3)
            },
            {
                name: 'SphereGeometry',
                description: 'ลูกกลม',
                create: () => new THREE.SphereGeometry(1, 32, 16)
            },
            {
                name: 'PlaneGeometry',
                description: 'แผ่นแบน',
                create: () => new THREE.PlaneGeometry(2, 2, 8, 8)
            },
            {
                name: 'CylinderGeometry',
                description: 'ทรงกระบอก',
                create: () => new THREE.CylinderGeometry(0.7, 0.7, 2, 32, 4)
            },
            {
                name: 'ConeGeometry',
                description: 'ทรงกรวย',
                create: () => new THREE.ConeGeometry(1, 2, 32)
            },
            {
                name: 'TorusGeometry',
                description: 'โดนัท',
                create: () => new THREE.TorusGeometry(0.8, 0.35, 16, 100)
            },
            {
                name: 'TorusKnotGeometry',
                description: 'โดนัทบิด',
                create: () => new THREE.TorusKnotGeometry(0.7, 0.25, 100, 16, 2, 3)
            },
            {
                name: 'DodecahedronGeometry',
                description: 'ทรงสิบสองหน้า',
                create: () => new THREE.DodecahedronGeometry(1, 0)
            },
            {
                name: 'IcosahedronGeometry',
                description: 'ทรงยี่สิบหน้า',
                create: () => new THREE.IcosahedronGeometry(1, 1)
            },
            {
                name: 'OctahedronGeometry',
                description: 'ทรงแปดหน้า',
                create: () => new THREE.OctahedronGeometry(1, 0)
            },
            {
                name: 'TetrahedronGeometry',
                description: 'ทรงสี่หน้า',
                create: () => new THREE.TetrahedronGeometry(1.2, 0)
            },
            {
                name: 'CircleGeometry',
                description: 'วงกลม',
                create: () => new THREE.CircleGeometry(1.2, 32)
            },
            {
                name: 'RingGeometry',
                description: 'วงแหวน',
                create: () => new THREE.RingGeometry(0.5, 1.2, 32, 4)
            },
            {
                name: 'TubeGeometry',
                description: 'ท่อตาม Curve',
                create: () => {
                    const curve = new THREE.CatmullRomCurve3([
                        new THREE.Vector3(-1.5, 0, 0),
                        new THREE.Vector3(-0.5, 1, 0.5),
                        new THREE.Vector3(0.5, -1, 0.5),
                        new THREE.Vector3(1.5, 0, 0),
                    ]);
                    return new THREE.TubeGeometry(curve, 64, 0.2, 8, false);
                }
            },
            {
                name: 'LatheGeometry',
                description: 'รูปทรงหมุน (แจกัน)',
                create: () => {
                    const points = [];
                    for (let i = 0; i < 10; i++) {
                        points.push(new THREE.Vector2(
                            Math.sin(i * 0.2) * 0.4 + 0.3,
                            (i - 5) * 0.2
                        ));
                    }
                    return new THREE.LatheGeometry(points, 24);
                }
            },
            {
                name: 'ExtrudeGeometry',
                description: 'Shape 2D ถูก Extrude',
                create: () => {
                    const shape = new THREE.Shape();
                    for (let i = 0; i < 5; i++) {
                        const angle = (i / 5) * Math.PI * 2 - Math.PI / 2;
                        const r = i % 2 === 0 ? 1 : 0.4;
                        if (i === 0) shape.moveTo(Math.cos(angle) * r, Math.sin(angle) * r);
                        else shape.lineTo(Math.cos(angle) * r, Math.sin(angle) * r);
                    }
                    shape.closePath();
                    return new THREE.ExtrudeGeometry(shape, {
                        depth: 0.4,
                        bevelEnabled: true,
                        bevelThickness: 0.05,
                        bevelSize: 0.05,
                        bevelSegments: 3
                    });
                }
            },
        ];
        
        // ========== CURRENT GEOMETRY DISPLAY ==========
        let currentIndex = 0;
        let isWireframe = false;
        let currentMesh = null;
        
        const material = new THREE.MeshStandardMaterial({
            color: '#6c63ff',
            metalness: 0.2,
            roughness: 0.4,
            side: THREE.DoubleSide,
        });
        
        const wireframeMaterial = new THREE.MeshBasicMaterial({
            color: '#6c63ff',
            wireframe: true
        });
        
        function showGeometry(index) {
            // ลบ mesh เก่า
            if (currentMesh) {
                scene.remove(currentMesh);
                currentMesh.geometry.dispose();
                currentMesh = null;
            }
            
            const def = geometries[index];
            const geo = def.create();
            
            currentMesh = new THREE.Mesh(
                geo, 
                isWireframe ? wireframeMaterial : material
            );
            currentMesh.castShadow = true;
            scene.add(currentMesh);
            
            document.getElementById('geo-name').textContent = 
                `${def.name} - ${def.description}`;
        }
        
        // แสดง geometry แรก
        showGeometry(0);
        
        // ========== CONTROLS ==========
        document.getElementById('prevBtn').addEventListener('click', () => {
            currentIndex = (currentIndex - 1 + geometries.length) % geometries.length;
            showGeometry(currentIndex);
        });
        
        document.getElementById('nextBtn').addEventListener('click', () => {
            currentIndex = (currentIndex + 1) % geometries.length;
            showGeometry(currentIndex);
        });
        
        document.getElementById('wireBtn').addEventListener('click', () => {
            isWireframe = !isWireframe;
            if (currentMesh) {
                currentMesh.material = isWireframe ? wireframeMaterial : material;
            }
            document.getElementById('wireBtn').textContent = 
                isWireframe ? 'Solid' : 'Wireframe';
        });
        
        // Keyboard controls
        window.addEventListener('keydown', (e) => {
            switch (e.key) {
                case 'ArrowRight':
                    currentIndex = (currentIndex + 1) % geometries.length;
                    showGeometry(currentIndex);
                    break;
                case 'ArrowLeft':
                    currentIndex = (currentIndex - 1 + geometries.length) % geometries.length;
                    showGeometry(currentIndex);
                    break;
                case 'w':
                case 'W':
                    isWireframe = !isWireframe;
                    if (currentMesh) {
                        currentMesh.material = isWireframe ? wireframeMaterial : material;
                    }
                    break;
                case ' ':
                    controls.autoRotate = !controls.autoRotate;
                    e.preventDefault();
                    break;
            }
        });
        
        // ========== GUI ==========
        const gui = new GUI({ title: 'Geometry Settings' });
        const geoFolder = gui.addFolder('Geometry');
        
        const params = {
            geometry: geometries[0].name,
            wireframe: false,
            autoRotate: true,
            color: '#6c63ff',
            metalness: 0.2,
            roughness: 0.4
        };
        
        const geoNames = geometries.map(g => g.name);
        geoFolder.add(params, 'geometry', geoNames).onChange(val => {
            currentIndex = geoNames.indexOf(val);
            showGeometry(currentIndex);
        });
        
        const matFolder = gui.addFolder('Material');
        matFolder.addColor(params, 'color').onChange(val => {
            material.color.set(val);
        });
        matFolder.add(params, 'wireframe').onChange(val => {
            isWireframe = val;
            if (currentMesh) {
                currentMesh.material = isWireframe ? wireframeMaterial : material;
            }
        });
        matFolder.add(params, 'metalness', 0, 1, 0.01).onChange(val => {
            material.metalness = val;
        });
        matFolder.add(params, 'roughness', 0, 1, 0.01).onChange(val => {
            material.roughness = val;
        });
        
        const camFolder = gui.addFolder('Camera');
        camFolder.add(params, 'autoRotate').onChange(val => {
            controls.autoRotate = val;
        });
        
        // ========== GRID ==========
        const gridHelper = new THREE.GridHelper(20, 20, '#444', '#2a2a2a');
        gridHelper.position.y = -2;
        scene.add(gridHelper);
        
        // ========== RESIZE ==========
        window.addEventListener('resize', () => {
            camera.aspect = innerWidth / innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(innerWidth, innerHeight);
            renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
        });
        
        // ========== ANIMATION ==========
        const clock = new THREE.Clock();
        
        function animate() {
            requestAnimationFrame(animate);
            
            const elapsed = clock.getElapsedTime();
            
            // หมุน current mesh ช้าๆ
            if (currentMesh && !controls.autoRotate) {
                currentMesh.rotation.y = elapsed * 0.3;
            }
            
            // Pulse lights
            pointLight1.intensity = 2 + Math.sin(elapsed * 1.5) * 0.5;
            pointLight2.intensity = 2 + Math.cos(elapsed * 1.5) * 0.5;
            
            controls.update();
            renderer.render(scene, camera);
        }
        
        animate();
    </script>
</body>
</html>
```

---

## สรุป Geometry Cheatsheet

| Geometry | Parameters หลัก | ใช้เมื่อ |
|----------|-----------------|---------|
| BoxGeometry | w, h, d, segX, segY, segZ | กล่อง, ห้อง, building |
| SphereGeometry | r, wSeg, hSeg | planet, ball, head |
| PlaneGeometry | w, h, wSeg, hSeg | พื้น, wall, terrain |
| CylinderGeometry | rTop, rBot, h, rSeg | เสา, ถัง, ขวด |
| TorusGeometry | r, tube, rSeg, tSeg | แหวน, donut |
| TorusKnotGeometry | r, tube, tSeg, rSeg, p, q | decoration, abstract |
| TubeGeometry | path, tSeg, r, rSeg | rope, wire, track |
| LatheGeometry | points[], seg | แจกัน, โอ่ง |
| ExtrudeGeometry | shape, settings | logo 3D, text |
| IcosahedronGeometry | r, detail | low-poly sphere |

---

## แบบฝึกหัด (Exercises)

### Exercise 1: Geometry Showcase (ง่าย)
สร้าง grid ของ geometries ต่างๆ ในฉากเดียว (เรียง 4x4)  
แต่ละ geometry มีสีต่างกัน

### Exercise 2: Low-poly Planet (ปานกลาง)
ใช้ `IcosahedronGeometry(2, 2)` สร้าง planet  
เพิ่ม `SphereGeometry` เล็กๆ เป็นดาวเทียมที่โคจรรอบ

### Exercise 3: Vase Creator (ปานกลาง)
ใช้ `LatheGeometry` สร้างแจกันที่ interactive  
ใช้ dat.GUI ให้ผู้ใช้ปรับ control points ของ profile ได้

### Exercise 4: Star Field (ปานกลาง)
สร้าง star field ด้วย `IcosahedronGeometry(0.05, 0)` จำนวน 500 ดวง  
วางแบบสุ่มในพื้นที่ 50x50x50 หน่วย

### Exercise 5: DNA Helix (ยาก)
ใช้ `TubeGeometry` กับ `CatmullRomCurve3` สร้าง DNA double helix  
สองเส้นพันกัน พร้อม sphere เล็กๆ ที่ base pairs

```javascript
// Hint: สูตร helix
function createHelix(radius, height, turns, offset = 0) {
    const points = [];
    const numPoints = turns * 50;
    for (let i = 0; i <= numPoints; i++) {
        const t = i / numPoints;
        const angle = t * turns * Math.PI * 2 + offset;
        points.push(new THREE.Vector3(
            Math.cos(angle) * radius,  // x
            (t - 0.5) * height,        // y
            Math.sin(angle) * radius   // z
        ));
    }
    return new THREE.CatmullRomCurve3(points);
}
```

---

**ก่อนหน้า:** [Part 002 - Your First 3D Scene](part-002.md)  
**ต่อไป:** [Part 004 - Materials Part 1](part-004.md)
