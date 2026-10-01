# Part 83: Advanced Post-Processing (Three.js v2 PPF)

## บทนำ: Post-Processing คืออะไร?

Post-Processing หรือ Post-Effects คือการใช้ effect ต่างๆ กับภาพที่ render แล้ว เหมือนกับการใช้ filter ใน Photoshop แต่ทำแบบ real-time บน GPU

Three.js มี post-processing ในตัว แต่มี library ที่ดีกว่าคือ **pmndrs/postprocessing** ซึ่งมี:
- Performance ดีกว่า (ใช้ single pass เมื่อเป็นไปได้)
- Effects จำนวนมาก
- การสร้าง Custom Effects ง่ายกว่า

---

## 1. การติดตั้ง postprocessing

```bash
npm install postprocessing
# หรือ
yarn add postprocessing
```

### การ Import

```javascript
import { 
  EffectComposer,
  EffectPass,
  RenderPass,
  Effect,
  BloomEffect,
  SSAOEffect,
  SMAAEffect,
  ToneMappingEffect,
  ChromaticAberrationEffect,
  GlitchEffect,
  NoiseEffect,
  VignetteEffect,
  DotScreenEffect,
  PixelationEffect,
  SepiaEffect,
  HueSaturationEffect,
  BrightnessContrastEffect,
  ColorDepthEffect,
  GridEffect,
  ScanlineEffect,
  ShockWaveEffect,
  SelectiveBloomEffect
} from 'postprocessing';
```

---

## 2. EffectComposer Setup พื้นฐาน

```javascript
import * as THREE from 'three';
import {
  EffectComposer,
  EffectPass,
  RenderPass,
  BloomEffect,
  ToneMappingEffect,
  SMAAEffect,
  VignetteEffect
} from 'postprocessing';

// สร้าง renderer ปกติ
const renderer = new THREE.WebGLRenderer({
  canvas: document.getElementById('canvas'),
  antialias: false,  // ปิด built-in AA เพราะใช้ SMAA แทน
  powerPreference: 'high-performance'
});
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;

// scene, camera setup ปกติ
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);

// ===== สร้าง EffectComposer =====
const composer = new EffectComposer(renderer, {
  frameBufferType: THREE.HalfFloatType,  // ใช้ HDR buffer
  multisampling: 0  // ปิด MSAA (ใช้ SMAA แทน)
});

// Pass 1: Render scene ปกติ
const renderPass = new RenderPass(scene, camera);
composer.addPass(renderPass);

// Pass 2: Effects (รวมหลาย effects ใน EffectPass เดียวเพื่อ performance)
const bloomEffect = new BloomEffect({
  luminanceThreshold: 0.6,
  luminanceSmoothing: 0.4,
  intensity: 2.0,
  radius: 0.8
});

const toneMappingEffect = new ToneMappingEffect({
  mode: ToneMappingMode.ACES_FILMIC,
  resolution: 256,
  whitePoint: 4.0,
  middleGrey: 0.6,
  minLuminance: 0.01
});

const vignetteEffect = new VignetteEffect({
  offset: 0.4,
  darkness: 0.5
});

const smaaEffect = new SMAAEffect();

// รวม effects ใน single pass
const effectPass = new EffectPass(camera, 
  bloomEffect,
  toneMappingEffect,
  vignetteEffect,
  smaaEffect
);
composer.addPass(effectPass);

// Animation loop ใช้ composer.render() แทน renderer.render()
function animate() {
  requestAnimationFrame(animate);
  composer.render();  // ไม่ใช้ renderer.render() โดยตรง
}
animate();

// Resize
window.addEventListener('resize', () => {
  const w = window.innerWidth;
  const h = window.innerHeight;
  renderer.setSize(w, h);
  composer.setSize(w, h);
  camera.aspect = w / h;
  camera.updateProjectionMatrix();
});
```

---

## 3. Effect ต่างๆ

### BloomEffect

```javascript
import { BloomEffect, KernelSize } from 'postprocessing';

// Bloom settings
const bloom = new BloomEffect({
  // Threshold: เฉพาะ pixel ที่สว่างกว่านี้จะมี bloom
  luminanceThreshold: 0.6,
  // ความนุ่มนวลของ transition
  luminanceSmoothing: 0.4,
  // ความแรงของ bloom
  intensity: 2.0,
  // รัศมีของ bloom
  radius: 0.8,
  // ขนาด kernel สำหรับ blur
  kernelSize: KernelSize.LARGE,
  // ความละเอียดของ bloom (ต่ำ = เร็ว)
  resolutionScale: 0.5
});

// Dynamic control
bloom.luminanceMaterial.threshold = 0.7;
bloom.intensity = 3.0;
```

### SSAOEffect (Screen Space Ambient Occlusion)

```javascript
import { SSAOEffect, BlendFunction } from 'postprocessing';

const ssao = new SSAOEffect(camera, scene.normalMap, {
  // จำนวน samples (สูง = คุณภาพดี, ช้า)
  samples: 16,
  // รัศมีการมอง
  rings: 4,
  distanceThreshold: 0.02,
  distanceFalloff: 0.0025,
  rangeThreshold: 0.005,
  rangeFalloff: 0.01,
  // ความแรงของ AO
  luminanceInfluence: 0.7,
  radius: 18.25,
  scale: 1.0,
  bias: 0.05,
  // Blend mode
  blendFunction: BlendFunction.MULTIPLY
});
```

### ChromaticAberrationEffect

```javascript
import { ChromaticAberrationEffect } from 'postprocessing';

// Chromatic aberration (สีแยก)
const chromaticAberration = new ChromaticAberrationEffect({
  offset: new THREE.Vector2(0.002, 0.002),
  radialModulation: true,  // แรงขึ้นที่ขอบ
  modulationOffset: 0.4    // จุดเริ่มต้นของ radial effect
});

// Dynamic: ทำให้แรงขึ้นเมื่อเคลื่อนไหว
let velocity = 0;
window.addEventListener('mousemove', (e) => {
  velocity = Math.sqrt(e.movementX ** 2 + e.movementY ** 2) * 0.001;
  chromaticAberration.offset.set(velocity * 0.5, velocity * 0.5);
});
```

### GlitchEffect

```javascript
import { GlitchEffect, GlitchMode } from 'postprocessing';

const glitch = new GlitchEffect({
  // ช่วงเวลาระหว่าง glitch (วินาที)
  delay: new THREE.Vector2(1.5, 3.5),
  // ความนานของ glitch
  duration: new THREE.Vector2(0.6, 1.0),
  // ความแรงของ displacement
  strength: new THREE.Vector2(0.3, 1.0),
  // Perturbation แบบ permanent หรือ sporadic
  mode: GlitchMode.SPORADIC,
  // Columns ของ noise
  columns: 0.05,
  // Custom perturbation map
  // perturbationMap: yourTexture
});

// Force glitch
glitch.mode = GlitchMode.CONSTANT_WILD;  // glitch ตลอดเวลา
setTimeout(() => {
  glitch.mode = GlitchMode.SPORADIC;  // กลับเป็นปกติ
}, 2000);
```

### ToneMappingEffect

```javascript
import { ToneMappingEffect, ToneMappingMode } from 'postprocessing';

const toneMapping = new ToneMappingEffect({
  // เลือก mode:
  // NONE, LINEAR, REINHARD, REINHARD2, REINHARD2_ADAPTIVE
  // UNCHARTED2, OPTIMIZED_CINEON, ACES_FILMIC, NEUTRAL
  mode: ToneMappingMode.ACES_FILMIC,
  // สำหรับ adaptive tone mapping
  resolution: 256,
  whitePoint: 4.0,
  middleGrey: 0.6,
  minLuminance: 0.01,
  averageLuminance: 1.0,
  // Adaptation speed
  adaptationRate: 1.0,
  // เพิ่ม exposure bias
  exposureBias: 0.0
});
```

---

## 4. Custom Effect Class

การสร้าง Custom Effect ต้องสืบทอดจาก `Effect` class

```javascript
import { Effect, BlendFunction } from 'postprocessing';
import * as THREE from 'three';

// ===== Film Grain Effect =====
const filmGrainShader = `
  // uniforms
  uniform float uTime;
  uniform float uStrength;
  uniform bool uAnimated;
  
  // noise function
  float random(vec2 p) {
    return fract(sin(dot(p, vec2(12.9898, 78.233))) * 43758.5453);
  }
  
  // grain function
  float grain(vec2 uv, float strength) {
    // ใช้ time เพื่อทำให้ grain เคลื่อนไหว
    float t = uAnimated ? floor(uTime * 24.0) / 24.0 : 0.0;
    
    // Tiled noise
    vec2 p = uv * 1000.0 + t * 100.0;
    float noise = random(floor(p)) * 0.5 + random(floor(p + 0.5)) * 0.5;
    
    return (noise - 0.5) * strength;
  }
  
  void mainImage(const in vec4 inputColor, const in vec2 uv, out vec4 outputColor) {
    float g = grain(uv, uStrength);
    
    // Apply grain: ลดแสง บวก ลดแสง-บวก noise
    vec3 col = inputColor.rgb + g;
    
    outputColor = vec4(col, inputColor.a);
  }
`;

class FilmGrainEffect extends Effect {
  constructor({
    strength = 0.05,
    animated = true,
    blendFunction = BlendFunction.NORMAL
  } = {}) {
    super('FilmGrainEffect', filmGrainShader, {
      blendFunction,
      uniforms: new Map([
        ['uTime', new THREE.Uniform(0)],
        ['uStrength', new THREE.Uniform(strength)],
        ['uAnimated', new THREE.Uniform(animated)]
      ])
    });
    
    this._strength = strength;
    this._animated = animated;
  }
  
  // ต้อง update time ใน render loop
  update(renderer, inputBuffer, deltaTime) {
    this.uniforms.get('uTime').value += deltaTime;
  }
  
  get strength() { return this._strength; }
  set strength(v) {
    this._strength = v;
    this.uniforms.get('uStrength').value = v;
  }
  
  get animated() { return this._animated; }
  set animated(v) {
    this._animated = v;
    this.uniforms.get('uAnimated').value = v;
  }
}

// ===== Old Film Effect (รวม Sepia + Vignette + Grain + Scanlines) =====
const oldFilmShader = `
  uniform float uTime;
  uniform float uGrainStrength;
  uniform float uScanlineStrength;
  uniform float uScanlineCount;
  uniform float uVignetteStrength;
  uniform float uSepiaStrength;
  uniform float uFlicker;
  
  float random(vec2 p) {
    return fract(sin(dot(p, vec2(12.9898, 78.233))) * 43758.5453);
  }
  
  vec3 sepia(vec3 col) {
    return vec3(
      dot(col, vec3(0.393, 0.769, 0.189)),
      dot(col, vec3(0.349, 0.686, 0.168)),
      dot(col, vec3(0.272, 0.534, 0.131))
    );
  }
  
  void mainImage(const in vec4 inputColor, const in vec2 uv, out vec4 outputColor) {
    vec3 col = inputColor.rgb;
    
    // Sepia
    col = mix(col, sepia(col), uSepiaStrength);
    
    // Flicker
    float flicker = 1.0 - uFlicker * random(vec2(floor(uTime * 30.0), 0.0)) * 0.1;
    col *= flicker;
    
    // Grain
    float t = floor(uTime * 18.0);
    float grain = random(uv * 1000.0 + t) - 0.5;
    col += grain * uGrainStrength;
    
    // Scanlines
    float scanline = sin(uv.y * uScanlineCount * 3.14159) * 0.5 + 0.5;
    col *= 1.0 - uScanlineStrength * (1.0 - scanline);
    
    // Vignette
    vec2 center = uv - 0.5;
    float dist = length(center);
    float vignette = smoothstep(0.7, 0.1, dist);
    col *= mix(1.0, vignette, uVignetteStrength);
    
    outputColor = vec4(col, inputColor.a);
  }
`;

class OldFilmEffect extends Effect {
  constructor({
    grainStrength = 0.08,
    scanlineStrength = 0.2,
    scanlineCount = 800,
    vignetteStrength = 0.7,
    sepiaStrength = 0.6,
    flicker = 0.5,
    blendFunction = BlendFunction.NORMAL
  } = {}) {
    super('OldFilmEffect', oldFilmShader, {
      blendFunction,
      uniforms: new Map([
        ['uTime', new THREE.Uniform(0)],
        ['uGrainStrength', new THREE.Uniform(grainStrength)],
        ['uScanlineStrength', new THREE.Uniform(scanlineStrength)],
        ['uScanlineCount', new THREE.Uniform(scanlineCount)],
        ['uVignetteStrength', new THREE.Uniform(vignetteStrength)],
        ['uSepiaStrength', new THREE.Uniform(sepiaStrength)],
        ['uFlicker', new THREE.Uniform(flicker)]
      ])
    });
  }
  
  update(renderer, inputBuffer, deltaTime) {
    this.uniforms.get('uTime').value += deltaTime;
  }
}
```

---

## 5. Complete Post-Processing Stack

```javascript
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import {
  EffectComposer,
  EffectPass,
  RenderPass,
  BloomEffect,
  SMAAEffect,
  ToneMappingEffect,
  VignetteEffect,
  ChromaticAberrationEffect,
  GlitchEffect,
  NoiseEffect,
  HueSaturationEffect,
  BrightnessContrastEffect,
  DepthOfFieldEffect,
  SSAOEffect
} from 'postprocessing';
import { ToneMappingMode, BlendFunction } from 'postprocessing';

class AdvancedRenderer {
  constructor(canvas, options = {}) {
    this.canvas = canvas;
    this.options = {
      antialias: true,
      bloom: true,
      ssao: false,
      dof: false,
      colorGrading: true,
      filmGrain: true,
      ...options
    };
    
    this.setupRenderer();
    this.setupComposer();
    this.effects = {};
  }
  
  setupRenderer() {
    this.renderer = new THREE.WebGLRenderer({
      canvas: this.canvas,
      antialias: false,  // ใช้ SMAA แทน
      powerPreference: 'high-performance',
      stencil: false,
      depth: false  // จะสร้าง depth buffer เอง
    });
    
    this.renderer.setSize(window.innerWidth, window.innerHeight);
    this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    this.renderer.shadowMap.enabled = true;
    this.renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    this.renderer.outputColorSpace = THREE.SRGBColorSpace;
    this.renderer.toneMapping = THREE.NoToneMapping;  // tone mapping ทำใน post-processing
    
    return this.renderer;
  }
  
  setupComposer() {
    this.composer = new EffectComposer(this.renderer, {
      frameBufferType: THREE.HalfFloatType
    });
    
    return this.composer;
  }
  
  buildPipeline(scene, camera) {
    this.composer.removeAllPasses();
    
    // 1. Render Pass
    const renderPass = new RenderPass(scene, camera);
    this.composer.addPass(renderPass);
    
    // 2. SSAO (optional)
    if (this.options.ssao) {
      const ssao = new SSAOEffect(camera, scene, {
        samples: 16,
        rings: 4,
        distanceThreshold: 0.02,
        distanceFalloff: 0.0025,
        rangeThreshold: 0.005,
        rangeFalloff: 0.01,
        luminanceInfluence: 0.7,
        radius: 18.25,
        bias: 0.05,
        intensity: 1.0,
        blendFunction: BlendFunction.MULTIPLY
      });
      this.effects.ssao = ssao;
    }
    
    // 3. Depth of Field (optional)
    if (this.options.dof) {
      const dof = new DepthOfFieldEffect(camera, {
        focalLength: 0.025,
        focusDistance: 0.0,
        bokehScale: 3.0,
        height: 480
      });
      this.effects.dof = dof;
    }
    
    // 4. Bloom
    if (this.options.bloom) {
      const bloom = new BloomEffect({
        luminanceThreshold: 0.6,
        luminanceSmoothing: 0.4,
        intensity: 1.5,
        radius: 0.7,
        resolutionScale: 0.5
      });
      this.effects.bloom = bloom;
    }
    
    // 5. Tone Mapping + Color Grading
    const toneMapping = new ToneMappingEffect({
      mode: ToneMappingMode.ACES_FILMIC,
      whitePoint: 4.0,
      middleGrey: 0.6
    });
    this.effects.toneMapping = toneMapping;
    
    // 6. Chromatic Aberration
    const chromaticAberration = new ChromaticAberrationEffect({
      offset: new THREE.Vector2(0.001, 0.001)
    });
    this.effects.chromaticAberration = chromaticAberration;
    
    // 7. Film Grain
    if (this.options.filmGrain) {
      const filmGrain = new FilmGrainEffect({ strength: 0.04 });
      this.effects.filmGrain = filmGrain;
    }
    
    // 8. Vignette
    const vignette = new VignetteEffect({
      offset: 0.4,
      darkness: 0.4
    });
    this.effects.vignette = vignette;
    
    // 9. SMAA (Anti-aliasing - ต้องอยู่ท้ายสุดก่อน gamma)
    const smaa = new SMAAEffect();
    this.effects.smaa = smaa;
    
    // สร้าง effect pass
    const effectList = Object.values(this.effects);
    if (effectList.length > 0) {
      const effectPass = new EffectPass(camera, ...effectList);
      effectPass.renderToScreen = true;
      this.composer.addPass(effectPass);
    }
    
    return this;
  }
  
  render(deltaTime) {
    this.composer.render(deltaTime);
  }
  
  resize(width, height) {
    this.renderer.setSize(width, height);
    this.composer.setSize(width, height);
  }
  
  // Preset: Cinematic
  setCinematicMode() {
    if (this.effects.bloom) this.effects.bloom.intensity = 1.8;
    if (this.effects.chromaticAberration) {
      this.effects.chromaticAberration.offset.set(0.002, 0.002);
    }
    if (this.effects.vignette) this.effects.vignette.darkness = 0.7;
    return this;
  }
  
  // Preset: Retro
  setRetroMode() {
    if (this.effects.bloom) this.effects.bloom.intensity = 0.5;
    // เพิ่ม sepia, scanlines
    return this;
  }
  
  // Preset: Glitch
  setGlitchMode(duration = 2000) {
    const glitch = new GlitchEffect({ mode: GlitchMode.CONSTANT_WILD });
    this.effects.glitch = glitch;
    
    setTimeout(() => {
      delete this.effects.glitch;
      this.rebuildPipeline();
    }, duration);
    
    return this;
  }
}
```

---

## 6. Stylized Renderer ตัวอย่างเต็ม

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title>Stylized Post-Processing Demo</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { overflow: hidden; background: #000; font-family: sans-serif; }
    canvas { display: block; }
    #controls {
      position: fixed;
      top: 10px;
      left: 10px;
      background: rgba(0,0,0,0.8);
      border: 1px solid rgba(255,255,255,0.1);
      border-radius: 8px;
      padding: 15px;
      color: white;
      width: 250px;
    }
    .effect-group {
      margin: 10px 0;
      border-bottom: 1px solid rgba(255,255,255,0.1);
      padding-bottom: 10px;
    }
    .effect-title {
      font-size: 12px;
      font-weight: bold;
      color: #88aaff;
      margin-bottom: 5px;
    }
    .slider-row {
      display: flex;
      align-items: center;
      gap: 8px;
      margin: 3px 0;
    }
    .slider-row label { font-size: 11px; width: 100px; }
    .slider-row input { flex: 1; }
    .slider-row span { font-size: 11px; width: 30px; text-align: right; }
    .preset-btn {
      display: inline-block;
      background: #334;
      border: 1px solid #556;
      color: white;
      padding: 4px 8px;
      border-radius: 4px;
      font-size: 11px;
      cursor: pointer;
      margin: 2px;
    }
    .preset-btn:hover { background: #445; }
    .preset-btn.active { background: #446; border-color: #88aaff; }
  </style>
</head>
<body>
  <canvas id="canvas"></canvas>
  <div id="controls">
    <div style="font-size: 13px; font-weight: bold; margin-bottom: 10px; color: #ffaa44;">
      Post-Processing Controls
    </div>
    
    <div style="margin-bottom: 10px;">
      <div style="font-size: 11px; margin-bottom: 5px;">Presets:</div>
      <button class="preset-btn active" id="presetNormal">Normal</button>
      <button class="preset-btn" id="presetCinematic">Cinematic</button>
      <button class="preset-btn" id="presetRetro">Retro</button>
      <button class="preset-btn" id="presetDream">Dream</button>
    </div>
    
    <div class="effect-group">
      <div class="effect-title">Bloom</div>
      <div class="slider-row">
        <label>Threshold</label>
        <input type="range" id="bloomThreshold" min="0" max="1" step="0.01" value="0.6">
        <span id="bloomThresholdVal">0.6</span>
      </div>
      <div class="slider-row">
        <label>Intensity</label>
        <input type="range" id="bloomIntensity" min="0" max="5" step="0.1" value="1.5">
        <span id="bloomIntensityVal">1.5</span>
      </div>
    </div>
    
    <div class="effect-group">
      <div class="effect-title">Chromatic Aberration</div>
      <div class="slider-row">
        <label>Offset</label>
        <input type="range" id="chromaticOffset" min="0" max="0.01" step="0.0001" value="0.001">
        <span id="chromaticOffsetVal">0.001</span>
      </div>
    </div>
    
    <div class="effect-group">
      <div class="effect-title">Vignette</div>
      <div class="slider-row">
        <label>Darkness</label>
        <input type="range" id="vignetteDarkness" min="0" max="1" step="0.01" value="0.4">
        <span id="vignetteDarknessVal">0.4</span>
      </div>
    </div>
    
    <div class="effect-group">
      <div class="effect-title">Film Grain</div>
      <div class="slider-row">
        <label>Strength</label>
        <input type="range" id="grainStrength" min="0" max="0.3" step="0.01" value="0.04">
        <span id="grainStrengthVal">0.04</span>
      </div>
    </div>
  </div>

  <script type="module">
    // NOTE: ตัวอย่างนี้ใช้ Three.js built-in post-processing
    // ในการใช้งานจริงให้ install postprocessing package
    import * as THREE from 'https://cdn.skypack.dev/three@0.160.0';
    import { EffectComposer } from 'https://cdn.skypack.dev/three@0.160.0/examples/jsm/postprocessing/EffectComposer.js';
    import { RenderPass } from 'https://cdn.skypack.dev/three@0.160.0/examples/jsm/postprocessing/RenderPass.js';
    import { UnrealBloomPass } from 'https://cdn.skypack.dev/three@0.160.0/examples/jsm/postprocessing/UnrealBloomPass.js';
    import { ShaderPass } from 'https://cdn.skypack.dev/three@0.160.0/examples/jsm/postprocessing/ShaderPass.js';
    import { OrbitControls } from 'https://cdn.skypack.dev/three@0.160.0/examples/jsm/controls/OrbitControls.js';
    
    const canvas = document.getElementById('canvas');
    
    // Renderer
    const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.shadowMap.enabled = true;
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.toneMappingExposure = 1.0;
    renderer.outputColorSpace = THREE.SRGBColorSpace;
    
    // Scene
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x050510);
    scene.fog = new THREE.FogExp2(0x050510, 0.03);
    
    // Camera
    const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
    camera.position.set(0, 2, 10);
    
    const controls = new OrbitControls(camera, canvas);
    controls.enableDamping = true;
    controls.target.set(0, 0, 0);
    
    // === Build Scene ===
    // Floor
    const floorGeo = new THREE.PlaneGeometry(40, 40);
    const floorMat = new THREE.MeshStandardMaterial({
      color: 0x111122,
      metalness: 0.9,
      roughness: 0.1
    });
    const floor = new THREE.Mesh(floorGeo, floorMat);
    floor.rotation.x = -Math.PI / 2;
    floor.receiveShadow = true;
    scene.add(floor);
    
    // Central object
    const torusGeo = new THREE.TorusKnotGeometry(1.5, 0.5, 200, 30);
    const torusMat = new THREE.MeshStandardMaterial({
      color: 0x4466ff,
      metalness: 1.0,
      roughness: 0.1,
      emissive: 0x002244,
      emissiveIntensity: 0.5
    });
    const torus = new THREE.Mesh(torusGeo, torusMat);
    torus.position.y = 2;
    torus.castShadow = true;
    scene.add(torus);
    
    // Glowing orbs
    const orbColors = [0xff4400, 0x00ff88, 0x4400ff, 0xff0088, 0x00aaff];
    const orbs = [];
    
    for (let i = 0; i < 20; i++) {
      const orbGeo = new THREE.SphereGeometry(0.2 + Math.random() * 0.3, 16, 16);
      const col = orbColors[Math.floor(Math.random() * orbColors.length)];
      const orbMat = new THREE.MeshStandardMaterial({
        color: col,
        emissive: col,
        emissiveIntensity: 3.0
      });
      const orb = new THREE.Mesh(orbGeo, orbMat);
      
      const angle = (i / 20) * Math.PI * 2;
      const radius = 4 + Math.random() * 3;
      orb.position.set(
        Math.cos(angle) * radius,
        1 + Math.random() * 3,
        Math.sin(angle) * radius
      );
      
      orbs.push({ mesh: orb, angle, radius, speed: 0.3 + Math.random() * 0.5, y: orb.position.y });
      scene.add(orb);
      
      // Point light for each orb
      const light = new THREE.PointLight(col, 0.5, 5);
      orb.add(light);
    }
    
    // Ambient + directional
    scene.add(new THREE.AmbientLight(0x112233, 0.5));
    
    const dirLight = new THREE.DirectionalLight(0xffffff, 0.5);
    dirLight.position.set(5, 10, 5);
    dirLight.castShadow = true;
    scene.add(dirLight);
    
    // === Post-Processing (Three.js built-in) ===
    const composer = new EffectComposer(renderer);
    composer.addPass(new RenderPass(scene, camera));
    
    // Bloom
    const bloomPass = new UnrealBloomPass(
      new THREE.Vector2(window.innerWidth, window.innerHeight),
      1.5,  // strength
      0.4,  // radius
      0.6   // threshold
    );
    composer.addPass(bloomPass);
    
    // Custom Film Grain Pass
    const filmGrainShader = {
      uniforms: {
        tDiffuse: { value: null },
        uTime: { value: 0 },
        uStrength: { value: 0.04 },
        uChromaticOffset: { value: 0.001 },
        uVignetteDarkness: { value: 0.4 }
      },
      vertexShader: `
        varying vec2 vUv;
        void main() { vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }
      `,
      fragmentShader: `
        uniform sampler2D tDiffuse;
        uniform float uTime;
        uniform float uStrength;
        uniform float uChromaticOffset;
        uniform float uVignetteDarkness;
        varying vec2 vUv;
        
        float random(vec2 p) { return fract(sin(dot(p, vec2(12.9898, 78.233))) * 43758.5453); }
        
        void main() {
          // Chromatic aberration
          vec2 offset = vec2(uChromaticOffset);
          float r = texture2D(tDiffuse, vUv + offset).r;
          float g = texture2D(tDiffuse, vUv).g;
          float b = texture2D(tDiffuse, vUv - offset).b;
          vec4 col = vec4(r, g, b, 1.0);
          
          // Film grain
          float grain = random(vUv * 1000.0 + floor(uTime * 24.0)) - 0.5;
          col.rgb += grain * uStrength;
          
          // Vignette
          vec2 center = vUv - 0.5;
          float dist = length(center * vec2(1.2, 1.0));
          float vignette = smoothstep(0.8, 0.2, dist);
          col.rgb *= mix(1.0 - uVignetteDarkness, 1.0, vignette);
          
          gl_FragColor = col;
        }
      `
    };
    
    const filmPass = new ShaderPass(filmGrainShader);
    composer.addPass(filmPass);
    
    // Controls
    function setupSlider(id, valId, onChange) {
      const slider = document.getElementById(id);
      const valEl = document.getElementById(valId);
      slider.addEventListener('input', (e) => {
        const v = parseFloat(e.target.value);
        valEl.textContent = v.toFixed(slider.step.includes('0001') ? 4 : 2);
        onChange(v);
      });
    }
    
    setupSlider('bloomThreshold', 'bloomThresholdVal', v => { bloomPass.threshold = v; });
    setupSlider('bloomIntensity', 'bloomIntensityVal', v => { bloomPass.strength = v; });
    setupSlider('chromaticOffset', 'chromaticOffsetVal', v => { filmPass.uniforms.uChromaticOffset.value = v; });
    setupSlider('vignetteDarkness', 'vignetteDarknessVal', v => { filmPass.uniforms.uVignetteDarkness.value = v; });
    setupSlider('grainStrength', 'grainStrengthVal', v => { filmPass.uniforms.uStrength.value = v; });
    
    // Presets
    const presets = {
      normal: { bloom: [0.6, 1.5], chromatic: 0.001, vignette: 0.4, grain: 0.04 },
      cinematic: { bloom: [0.5, 2.0], chromatic: 0.003, vignette: 0.7, grain: 0.06 },
      retro: { bloom: [0.8, 0.5], chromatic: 0.005, vignette: 0.8, grain: 0.15 },
      dream: { bloom: [0.3, 3.0], chromatic: 0.002, vignette: 0.3, grain: 0.02 }
    };
    
    function applyPreset(name) {
      const p = presets[name];
      bloomPass.threshold = p.bloom[0];
      bloomPass.strength = p.bloom[1];
      filmPass.uniforms.uChromaticOffset.value = p.chromatic;
      filmPass.uniforms.uVignetteDarkness.value = p.vignette;
      filmPass.uniforms.uStrength.value = p.grain;
      
      // Update UI
      document.querySelectorAll('.preset-btn').forEach(b => b.classList.remove('active'));
      document.getElementById(`preset${name.charAt(0).toUpperCase() + name.slice(1)}`).classList.add('active');
    }
    
    document.getElementById('presetNormal').onclick = () => applyPreset('normal');
    document.getElementById('presetCinematic').onclick = () => applyPreset('cinematic');
    document.getElementById('presetRetro').onclick = () => applyPreset('retro');
    document.getElementById('presetDream').onclick = () => applyPreset('dream');
    
    // Resize
    window.addEventListener('resize', () => {
      const w = window.innerWidth, h = window.innerHeight;
      renderer.setSize(w, h);
      composer.setSize(w, h);
      camera.aspect = w / h;
      camera.updateProjectionMatrix();
    });
    
    // Animation
    const clock = new THREE.Clock();
    function animate() {
      requestAnimationFrame(animate);
      
      const time = clock.getElapsedTime();
      const delta = clock.getDelta();
      
      filmPass.uniforms.uTime.value = time;
      
      torus.rotation.x = time * 0.3;
      torus.rotation.y = time * 0.5;
      
      orbs.forEach(({ mesh, angle, radius, speed, y }, i) => {
        const a = angle + time * speed;
        mesh.position.x = Math.cos(a) * radius;
        mesh.position.z = Math.sin(a) * radius;
        mesh.position.y = y + Math.sin(time * 2 + i) * 0.3;
      });
      
      controls.update();
      composer.render();
    }
    animate();
  </script>
</body>
</html>
```

---

## 7. สรุป Post-Processing

### Performance Tips
- รวม effects ใน EffectPass เดียวให้มากที่สุด (pmndrs/postprocessing ทำให้เป็น single pass)
- ลด resolution ของ bloom (`resolutionScale: 0.5`)
- ใช้ `HalfFloatType` สำหรับ framebuffer
- ปิด effects ที่ไม่ได้ใช้แทนการ delete pass

### Effect Priority (ลำดับที่ดีที่สุด)
```
1. RenderPass (render scene)
2. SSAOEffect (ambient occlusion)
3. DepthOfFieldEffect (blur by depth)
4. BloomEffect (glow)
5. ChromaticAberrationEffect (color fringe)
6. FilmGrainEffect (noise)
7. VignetteEffect (dark edges)
8. ToneMappingEffect (exposure/gamma)
9. SMAAEffect (anti-aliasing - ต้องอยู่ท้ายสุด)
```

ในบทถัดไป (Part 84) เราจะเรียนรู้เกี่ยวกับ Deferred Rendering และ G-Buffer
