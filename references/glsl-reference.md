# GLSL Quick Reference

## Types

```glsl
// Scalar
float  f = 1.0;
int    i = 1;
uint   u = 1u;
bool   b = true;

// Vector
vec2   v2 = vec2(1.0, 2.0);
vec3   v3 = vec3(1.0, 2.0, 3.0);
vec4   v4 = vec4(1.0, 2.0, 3.0, 1.0);

ivec2, ivec3, ivec4  // int vectors
bvec2, bvec3, bvec4  // bool vectors

// Matrix
mat2, mat3, mat4
mat4 m = mat4(1.0);  // Identity

// Sampler
sampler2D     tex;
samplerCube   cubemap;
sampler3D     vol;
```

---

## Swizzling

```glsl
vec4 v = vec4(1.0, 2.0, 3.0, 4.0);

v.x == v.r == v.s == 1.0
v.y == v.g == v.t == 2.0
v.z == v.b == v.p == 3.0
v.w == v.a == v.q == 4.0

v.xy     // vec2(1.0, 2.0)
v.rgb    // vec3(1.0, 2.0, 3.0)
v.zyx    // vec3(3.0, 2.0, 1.0)  -- can reorder
v.xxyy   // vec4(1.0, 1.0, 2.0, 2.0)

vec3 c = v.bgr;   // Swizzle channels
v.yz = vec2(5.0, 6.0);  // Can set via swizzle
```

---

## Built-in Math Functions

```glsl
// Trig (radians)
sin(x)  cos(x)  tan(x)
asin(x) acos(x) atan(y, x)  // atan2
sinh(x) cosh(x) tanh(x)

// Power / Exp
pow(x, y)     // x^y
exp(x)        // e^x
exp2(x)       // 2^x
log(x)        // ln(x)
log2(x)       // log base 2
sqrt(x)       // square root
inversesqrt(x)// 1/sqrt(x)

// Absolute / Sign
abs(x)        // |x|
sign(x)       // -1, 0, or 1
floor(x)      // round down
ceil(x)       // round up
round(x)      // nearest integer
fract(x)      // fractional part: x - floor(x)
mod(x, y)     // modulo: x - y*floor(x/y)
modf(x, i)    // splits into integer+fraction
trunc(x)      // toward zero

// Min / Max / Clamp
min(x, y)
max(x, y)
clamp(x, min, max)  // clamp(x, 0.0, 1.0)

// Interpolation
mix(x, y, a)        // x*(1-a) + y*a  (linear lerp)
smoothstep(e0, e1, x)  // smooth Hermite
step(edge, x)          // 0.0 or 1.0

// Vector operations
length(v)             // |v|
distance(a, b)        // |a - b|
dot(a, b)             // a · b
cross(a, b)           // a × b  (vec3 only)
normalize(v)          // v / |v|
reflect(i, n)         // reflect incident i around normal n
refract(i, n, eta)    // refraction vector
faceforward(n,i,nref) // return n if dot(nref,i)<0, else -n

// Matrix
transpose(m)
inverse(m)
determinant(m)
matrixCompMult(a, b)  // Component-wise multiply (NOT matrix mult)
outerProduct(c, r)    // outer product

// Comparisons (component-wise)
lessThan(a, b)           // bvec
lessThanEqual(a, b)      // bvec
greaterThan(a, b)        // bvec
greaterThanEqual(a, b)   // bvec
equal(a, b)              // bvec
notEqual(a, b)           // bvec
any(bvec)               // bool
all(bvec)               // bool
not(bvec)               // bvec
```

---

## Texture Sampling

```glsl
// Vertex shader
uniform sampler2D uMap;
varying vec2 vUv;

// Fragment shader
vec4  color = texture2D(uMap, vUv);
vec4  color = texture(uMap, vUv);      // GLSL3

// Mipmapping control
vec4 c = texture(uMap, vUv, bias);     // lod bias
vec4 c = textureLod(uMap, vUv, lod);  // explicit lod
vec4 c = textureGrad(uMap, uv, dPdx, dPdy);

// Cube map
uniform samplerCube uCube;
vec3  dir = normalize(vPosition);
vec4  c = texture(uCube, dir);

// 3D texture
uniform sampler3D uVol;
vec4  c = texture(uVol, vec3(uv, depth));
```

---

## Vertex Shader Inputs/Outputs

```glsl
// Built-in outputs (vertex shader)
gl_Position    // clip space position (required)
gl_PointSize   // point size for Points

// Built-in inputs (vertex shader)
gl_VertexID
gl_InstanceID

// Three.js auto-provided attributes
attribute vec3 position;
attribute vec3 normal;
attribute vec2 uv;
attribute vec3 color;
attribute vec2 uv2;
attribute vec4 tangent;
attribute vec3 binormal;

// Three.js auto-provided uniforms
uniform mat4 modelMatrix;          // Object local -> world
uniform mat4 modelViewMatrix;      // Object local -> view
uniform mat4 projectionMatrix;     // View -> clip
uniform mat4 viewMatrix;           // World -> view
uniform mat3 normalMatrix;         // Normal transform
uniform vec3 cameraPosition;
```

---

## Fragment Shader Inputs/Outputs

```glsl
// GLSL1 output
gl_FragColor = vec4(r, g, b, a);

// GLSL3 output (layout)
layout(location = 0) out vec4 fragColor;
layout(location = 1) out vec4 gNormal;    // MRT

// Built-in inputs
gl_FragCoord   // vec4, window-space position
gl_FrontFacing // bool, front face?
gl_PointCoord  // vec2, for point sprites

// Derivative functions
dFdx(x)   // derivative in x screen direction
dFdy(x)   // derivative in y screen direction
fwidth(x) // abs(dFdx(x)) + abs(dFdy(x))
```

---

## Precision

```glsl
// Precision qualifiers
precision highp float;    // ~24 bits, full range
precision mediump float;  // ~16 bits
precision lowp float;     // ~8 bits

// Typically:
// Vertex: highp
// Fragment: mediump (mobile default) or highp
```

---

## Control Flow

```glsl
// If
if (x > 0.5) {
  col = vec3(1.0);
} else {
  col = vec3(0.0);
}

// For loop
for (int i = 0; i < 10; i++) {
  sum += float(i);
}

// While (use carefully - GPU loops can be slow)
while (condition) { ... }

// Discard (fragment only) - discards fragment
if (alpha < 0.01) discard;

// Break / Continue
for (int i = 0; i < 10; i++) {
  if (i == 5) break;
  if (i == 3) continue;
}

// Ternary
float v = condition ? 1.0 : 0.0;
```

---

## Functions

```glsl
// Basic function
float myFunc(float x, float y) {
  return x * y;
}

// Void function (modifies inout)
void transform(inout vec3 pos) {
  pos.y += 1.0;
}

// Multiple outputs via out
void splitColor(vec3 col, out float r, out float g, out float b) {
  r = col.r; g = col.g; b = col.b;
}

// Parameter qualifiers
float calc(in float a, out float b, inout float c) {
  b = a * 2.0;
  c += 1.0;
  return a + b + c;
}
```

---

## Common Patterns

```glsl
// RGB to Grayscale
float gray = dot(color.rgb, vec3(0.299, 0.587, 0.114));

// Gamma correction
vec3 toLinear(vec3 c) { return pow(c, vec3(2.2)); }
vec3 toGamma(vec3 c)  { return pow(c, vec3(1.0/2.2)); }

// Hash / Pseudo-random
float hash(vec2 p) {
  p = fract(p * vec2(127.1, 311.7));
  p += dot(p, p + 19.19);
  return fract(p.x * p.y);
}

float random(vec2 st) {
  return fract(sin(dot(st, vec2(12.9898, 78.233))) * 43758.5453);
}

// Value Noise 2D
float noise(vec2 p) {
  vec2 i = floor(p);
  vec2 f = fract(p);
  f = f * f * (3.0 - 2.0 * f); // Smoothstep
  return mix(
    mix(hash(i),             hash(i + vec2(1,0)), f.x),
    mix(hash(i + vec2(0,1)), hash(i + vec2(1,1)), f.x),
    f.y
  );
}

// Fractal Brownian Motion (fbm)
float fbm(vec2 p) {
  float v = 0.0, a = 0.5;
  for (int i = 0; i < 6; i++) {
    v += a * noise(p);
    p = p * 2.0 + vec2(3.1, 1.7);
    a *= 0.5;
  }
  return v;
}

// SDF sphere
float sdSphere(vec3 p, float r) {
  return length(p) - r;
}

// SDF box
float sdBox(vec3 p, vec3 b) {
  vec3 q = abs(p) - b;
  return length(max(q, 0.0)) + min(max(q.x, max(q.y, q.z)), 0.0);
}

// Normal from SDF
vec3 calcNormal(vec3 p) {
  vec2 e = vec2(0.001, 0.0);
  return normalize(vec3(
    sdSphere(p + e.xyy, 1.0) - sdSphere(p - e.xyy, 1.0),
    sdSphere(p + e.yxy, 1.0) - sdSphere(p - e.yxy, 1.0),
    sdSphere(p + e.yyx, 1.0) - sdSphere(p - e.yyx, 1.0)
  ));
}

// Rotate 2D
vec2 rotate2D(vec2 v, float angle) {
  float s = sin(angle), c = cos(angle);
  return mat2(c, -s, s, c) * v;
}

// UV Polar
vec2 toPolar(vec2 uv) {
  uv -= 0.5;
  return vec2(length(uv), atan(uv.y, uv.x));
}

// Remap range
float remap(float val, float inMin, float inMax, float outMin, float outMax) {
  return outMin + (val - inMin) / (inMax - inMin) * (outMax - outMin);
}

// Wireframe from barycentric
float wireframe(vec3 bary, float thickness) {
  vec3 d = fwidth(bary);
  vec3 s = smoothstep(d * (thickness - 0.5), d * (thickness + 0.5), bary);
  return 1.0 - min(s.x, min(s.y, s.z));
}
```

---

## GLSL3 (WebGL2) Specifics

```glsl
#version 300 es
precision highp float;

// In/Out instead of attribute/varying
in vec3 position;     // vertex input
out vec2 vUv;         // varying out

// in fragment:
in vec2 vUv;          // varying in
out vec4 fragColor;   // output

// Multiple render targets
layout(location = 0) out vec4 gAlbedo;
layout(location = 1) out vec4 gNormal;

// Integer textures
uniform isampler2D intTex;
ivec4 v = texture(intTex, uv);
```

---

## Three.js Shader Chunks

```glsl
// Include Three.js chunks
#include <common>
#include <packing>        // packNormalToRGB, unpackRGBToNormal
#include <fog_pars_fragment>
#include <fog_fragment>
#include <lights_pars_begin>
#include <shadowmap_pars_fragment>
#include <shadow_pars_fragment>

// Useful Three.js functions (from <common>)
float linearToRelativeLuminance(vec3 color);
vec4 LinearTosRGB(vec4 value);
vec4 sRGBToLinear(vec4 value);
float luminance(vec3 rgb);
```

---

## Precision Tips

```
highp: 32-bit float (full IEEE 754 single)
mediump: 16-bit float (range: -65504 to 65504, precision 1/1024)
lowp: 10-bit fixed (range: -2 to 2, precision 1/256)

- Position calculations: highp
- Color: mediump or lowp
- UV coords: mediump
- Mobile performance: use mediump when possible
```
