# Part 104: Texture Atlasing

## บทนำ: ปัญหาของ Multiple Textures

ในการพัฒนา 3D Scene ที่ซับซ้อน เรามักต้องใช้ textures หลายชิ้น เช่น อาคาร 100 หลังที่มี texture ผนังต่างกัน ปัญหาคือแต่ละ texture ต้องการ:
1. **Texture unit binding** - GPU ต้องเปลี่ยน texture ที่ผูกอยู่
2. **Separate draw call** - ถ้า material ต่างกัน = draw call ต่างกัน
3. **GPU memory** - แต่ละ texture ใช้ VRAM แยกกัน (overhead จาก mipmap headers)

**Texture Atlas** แก้ปัญหาด้วยการรวม textures ทั้งหมดไว้ในรูปเดียวกัน แล้วใช้ UV coordinates ที่ต่างกันเพื่อเลือกส่วนที่ต้องการ

---

## 1. แนวคิดพื้นฐานของ Texture Atlas

```
ก่อน Atlas:                    หลัง Atlas:
┌─────┐ ┌─────┐ ┌─────┐       ┌──────────────┐
│ T1  │ │ T2  │ │ T3  │       │  T1  │  T2  │
│     │ │     │ │     │   →   │──────┼──────│
└─────┘ └─────┘ └─────┘       │  T3  │  T4  │
                               └──────────────┘
3 textures = 3 bindings         1 texture = 1 binding
```

UV Remapping:
```
Original UV (0-1):        Atlas UV:
┌────────────┐            ┌────────────┐
│            │            │  ┌──────┐  │
│  UV(0,0)   │     →      │  │UV    │  │ (offset + scale)
│  to UV(1,1)│            │  │region│  │
└────────────┘            └──┴──────┴──┘
```

---

## 2. การสร้าง Texture Atlas ด้วย Canvas API

```javascript
class TextureAtlasBuilder {
    constructor(options = {}) {
        this.size = options.size || 2048;
        this.padding = options.padding || 2; // pixels ระหว่าง tiles
        this.canvas = document.createElement('canvas');
        this.canvas.width = this.size;
        this.canvas.height = this.size;
        this.ctx = this.canvas.getContext('2d');
        
        this.tiles = new Map(); // name -> { x, y, w, h }
        this.packer = new ShelfPacker(this.size, this.size);
        
        this.texture = null;
    }
    
    // เพิ่ม texture จาก URL
    async addFromURL(name, url, width, height) {
        return new Promise((resolve, reject) => {
            const img = new Image();
            img.crossOrigin = 'anonymous';
            img.onload = () => {
                const rect = this.packer.pack(
                    (width || img.width) + this.padding * 2,
                    (height || img.height) + this.padding * 2
                );
                
                if (!rect) {
                    reject(new Error(`Atlas full! Cannot fit "${name}"`));
                    return;
                }
                
                const tx = rect.x + this.padding;
                const ty = rect.y + this.padding;
                const tw = rect.w - this.padding * 2;
                const th = rect.h - this.padding * 2;
                
                this.ctx.drawImage(img, tx, ty, tw, th);
                this.tiles.set(name, { x: tx, y: ty, w: tw, h: th });
                resolve(this.getUVs(name));
            };
            img.onerror = reject;
            img.src = url;
        });
    }
    
    // เพิ่ม solid color tile (สำหรับ testing)
    addColor(name, color, width = 64, height = 64) {
        const rect = this.packer.pack(
            width + this.padding * 2,
            height + this.padding * 2
        );
        
        if (!rect) throw new Error('Atlas full!');
        
        const tx = rect.x + this.padding;
        const ty = rect.y + this.padding;
        const tw = rect.w - this.padding * 2;
        const th = rect.h - this.padding * 2;
        
        this.ctx.fillStyle = color;
        this.ctx.fillRect(tx, ty, tw, th);
        
        this.tiles.set(name, { x: tx, y: ty, w: tw, h: th });
        return this.getUVs(name);
    }
    
    // เพิ่ม procedural pattern
    addPattern(name, drawFn, width = 128, height = 128) {
        const rect = this.packer.pack(
            width + this.padding * 2,
            height + this.padding * 2
        );
        
        if (!rect) throw new Error('Atlas full!');
        
        const tx = rect.x + this.padding;
        const ty = rect.y + this.padding;
        const tw = rect.w - this.padding * 2;
        const th = rect.h - this.padding * 2;
        
        // สร้าง offscreen canvas สำหรับ pattern
        const offscreen = document.createElement('canvas');
        offscreen.width = tw;
        offscreen.height = th;
        drawFn(offscreen.getContext('2d'), tw, th);
        
        this.ctx.drawImage(offscreen, tx, ty);
        this.tiles.set(name, { x: tx, y: ty, w: tw, h: th });
        return this.getUVs(name);
    }
    
    // ได้ UV coordinates (0-1) ของ tile
    getUVs(name) {
        const tile = this.tiles.get(name);
        if (!tile) return null;
        
        return {
            u: tile.x / this.size,
            v: tile.y / this.size,
            uSize: tile.w / this.size,
            vSize: tile.h / this.size,
            // Helper สำหรับ remap UV
            remap: (origU, origV) => ({
                u: origU * (tile.w / this.size) + (tile.x / this.size),
                v: origV * (tile.h / this.size) + (tile.y / this.size)
            })
        };
    }
    
    // สร้าง Three.js Texture
    buildTexture() {
        if (this.texture) this.texture.dispose();
        this.texture = new THREE.CanvasTexture(this.canvas);
        this.texture.generateMipmaps = true;
        this.texture.minFilter = THREE.LinearMipmapLinearFilter;
        this.texture.magFilter = THREE.LinearFilter;
        this.texture.needsUpdate = true;
        return this.texture;
    }
    
    // Debug: แสดง atlas ใน DOM
    debug(container) {
        const canvas = this.canvas.cloneNode();
        const ctx = canvas.getContext('2d');
        ctx.drawImage(this.canvas, 0, 0);
        
        // วาด grid ของ tiles
        ctx.strokeStyle = 'rgba(255, 0, 0, 0.5)';
        ctx.lineWidth = 1;
        this.tiles.forEach((tile, name) => {
            ctx.strokeRect(tile.x, tile.y, tile.w, tile.h);
            ctx.fillStyle = 'rgba(255, 0, 0, 0.8)';
            ctx.font = '12px monospace';
            ctx.fillText(name, tile.x + 2, tile.y + 14);
        });
        
        canvas.style.maxWidth = '100%';
        canvas.style.border = '1px solid red';
        (container || document.body).appendChild(canvas);
    }
}
```

---

## 3. Shelf-Next-Fit Packing Algorithm

```javascript
// Shelf packing: จัด tiles เป็น "ชั้น" เพื่อลด wasted space
class ShelfPacker {
    constructor(width, height) {
        this.width = width;
        this.height = height;
        this.shelves = [{ x: 0, y: 0, height: 0, remainingWidth: width }];
        this.usedArea = 0;
    }
    
    // คำนวณ efficiency ของ atlas
    get efficiency() {
        return this.usedArea / (this.width * this.height);
    }
    
    pack(w, h) {
        // หา shelf ที่พอดีที่สุด
        for (let i = 0; i < this.shelves.length; i++) {
            const shelf = this.shelves[i];
            
            if (shelf.remainingWidth >= w && shelf.height >= h) {
                // Fit ใน shelf นี้
                const x = this.width - shelf.remainingWidth;
                const y = shelf.y;
                shelf.remainingWidth -= w;
                this.usedArea += w * h;
                return { x, y, w, h };
            }
        }
        
        // ต้องสร้าง shelf ใหม่
        const lastShelf = this.shelves[this.shelves.length - 1];
        const newShelfY = lastShelf.y + lastShelf.height;
        
        if (newShelfY + h > this.height) {
            return null; // Atlas เต็ม!
        }
        
        const newShelf = {
            x: 0,
            y: newShelfY,
            height: h,
            remainingWidth: this.width - w
        };
        this.shelves.push(newShelf);
        this.usedArea += w * h;
        
        return { x: 0, y: newShelfY, w, h };
    }
    
    reset() {
        this.shelves = [{ x: 0, y: 0, height: 0, remainingWidth: this.width }];
        this.usedArea = 0;
        // ต้อง clear canvas ด้วย
    }
}
```

---

## 4. UV Remapping สำหรับ Geometry

```javascript
// Remap UV ของ geometry ให้ตรงกับ atlas tile
function remapGeometryUV(geometry, uvInfo) {
    const uv = geometry.attributes.uv;
    
    if (!uv) {
        console.warn('Geometry has no UV attribute');
        return geometry;
    }
    
    const cloned = geometry.clone();
    const clonedUV = cloned.attributes.uv;
    
    for (let i = 0; i < clonedUV.count; i++) {
        const origU = clonedUV.getX(i);
        const origV = clonedUV.getY(i);
        
        // Linear remap: origUV * tileSize + tileOffset
        clonedUV.setX(i, origU * uvInfo.uSize + uvInfo.u);
        clonedUV.setY(i, origV * uvInfo.vSize + uvInfo.v);
    }
    
    clonedUV.needsUpdate = true;
    return cloned;
}

// ตัวอย่างใช้งาน
const atlas = new TextureAtlasBuilder({ size: 1024 });

// เพิ่ม textures ลงใน atlas
const brickUV = atlas.addColor('brick', '#aa6644', 128, 128);
const concreteUV = atlas.addColor('concrete', '#888899', 128, 128);
const glassUV = atlas.addColor('glass', '#aaccff', 128, 128);

const atlasTexture = atlas.buildTexture();
const atlasMaterial = new THREE.MeshStandardMaterial({ map: atlasTexture });

// สร้าง building ด้วย atlas
function createBuilding(x, z, textureType) {
    const uvInfo = atlas.getUVs(textureType);
    const baseGeo = new THREE.BoxGeometry(4, 10, 4);
    const remappedGeo = remapGeometryUV(baseGeo, uvInfo);
    
    remappedGeo.applyMatrix4(new THREE.Matrix4().setPosition(x, 5, z));
    return remappedGeo;
}

// สร้าง buildings ด้วย textures ต่างกัน แต่ใช้ material เดียว!
const textureTypes = ['brick', 'concrete', 'glass'];
const buildingGeos = [];

for (let i = 0; i < 30; i++) {
    const type = textureTypes[i % 3];
    const geo = createBuilding(
        (Math.random() - 0.5) * 100,
        (Math.random() - 0.5) * 100,
        type
    );
    buildingGeos.push(geo);
}

import { mergeGeometries } from 'three/addons/utils/BufferGeometryUtils.js';
const merged = mergeGeometries(buildingGeos);
const mesh = new THREE.Mesh(merged, atlasMaterial);
scene.add(mesh);
// ผลลัพธ์: 1 draw call สำหรับ 30 buildings ที่มี texture ต่างกัน!
```

---

## 5. Sprite Sheet Animation บน 3D Plane

```javascript
class SpriteAnimator {
    constructor(options) {
        this.texture = options.texture;       // THREE.Texture
        this.tilesX = options.tilesX || 8;   // จำนวน tiles แนวนอน
        this.tilesY = options.tilesY || 8;   // จำนวน tiles แนวตั้ง
        this.frameCount = options.frameCount || this.tilesX * this.tilesY;
        this.fps = options.fps || 12;
        
        this.currentFrame = 0;
        this.lastFrameTime = 0;
        
        // Setup texture repeat for one tile
        this.texture.repeat.set(1 / this.tilesX, 1 / this.tilesY);
        this.texture.needsUpdate = true;
    }
    
    // Update animation (ต้องเรียกทุก frame)
    update(time) {
        const frameDuration = 1000 / this.fps;
        
        if (time - this.lastFrameTime >= frameDuration) {
            this.currentFrame = (this.currentFrame + 1) % this.frameCount;
            this.lastFrameTime = time;
            this.applyFrame();
        }
    }
    
    // Apply current frame to texture
    applyFrame() {
        const col = this.currentFrame % this.tilesX;
        const row = Math.floor(this.currentFrame / this.tilesX);
        
        this.texture.offset.set(
            col / this.tilesX,
            1 - (row + 1) / this.tilesY  // Y flip เพราะ texture Y กลับหัว
        );
    }
    
    // Jump to specific frame
    setFrame(frame) {
        this.currentFrame = Math.max(0, Math.min(frame, this.frameCount - 1));
        this.applyFrame();
    }
    
    // Play animation from frame A to B
    playRange(startFrame, endFrame, loop = true) {
        this.currentFrame = startFrame;
        this.endFrame = endFrame;
        this.loop = loop;
        this.isPlaying = true;
    }
}

// ใช้งาน SpriteAnimator
const loader = new THREE.TextureLoader();
const spriteTexture = loader.load('sprite-sheet.png'); // 8x8 = 64 frames
spriteTexture.magFilter = THREE.NearestFilter; // ไม่ blur pixel art

const animator = new SpriteAnimator({
    texture: spriteTexture,
    tilesX: 8,
    tilesY: 8,
    frameCount: 32, // ใช้แค่ 32 frames แรก
    fps: 12
});

const spriteMat = new THREE.MeshBasicMaterial({
    map: spriteTexture,
    transparent: true,
    alphaTest: 0.1,
    side: THREE.DoubleSide
});

const spritePlane = new THREE.Mesh(
    new THREE.PlaneGeometry(2, 2),
    spriteMat
);
scene.add(spritePlane);

// Update ใน animation loop
function animate(time) {
    requestAnimationFrame(animate);
    animator.update(time);
    renderer.render(scene, camera);
}
animate(0);
```

---

## 6. Runtime Atlas Builder Class (Complete)

```javascript
// Atlas builder ที่ใช้งานได้จริงในโปรเจกต์
class RuntimeTextureAtlas {
    constructor(size = 2048) {
        this.size = size;
        this.canvas = document.createElement('canvas');
        this.canvas.width = size;
        this.canvas.height = size;
        this.ctx = this.canvas.getContext('2d');
        
        this.tiles = new Map();
        this.packer = new ShelfPacker(size, size);
        this._texture = null;
        this._dirty = false;
    }
    
    get texture() {
        if (this._dirty || !this._texture) {
            if (this._texture) this._texture.dispose();
            this._texture = new THREE.CanvasTexture(this.canvas);
            this._texture.generateMipmaps = false;
            this._texture.minFilter = THREE.LinearFilter;
            this._dirty = false;
        }
        return this._texture;
    }
    
    addSolidColor(name, hex, w = 64, h = 64) {
        const uv = this._alloc(name, w, h);
        if (!uv) return null;
        const { px, py } = uv;
        this.ctx.fillStyle = hex;
        this.ctx.fillRect(px, py, w, h);
        this._dirty = true;
        return uv;
    }
    
    addGradient(name, colors, w = 128, h = 128) {
        const uv = this._alloc(name, w, h);
        if (!uv) return null;
        const { px, py } = uv;
        const grad = this.ctx.createLinearGradient(px, py, px + w, py + h);
        colors.forEach(([stop, color]) => grad.addColorStop(stop, color));
        this.ctx.fillStyle = grad;
        this.ctx.fillRect(px, py, w, h);
        this._dirty = true;
        return uv;
    }
    
    addBrickPattern(name, w = 128, h = 128, opts = {}) {
        const uv = this._alloc(name, w, h);
        if (!uv) return null;
        const { px, py } = uv;
        const ctx = this.ctx;
        const brickW = opts.brickW || 32;
        const brickH = opts.brickH || 16;
        const mortar = opts.mortar || 2;
        const brickColor = opts.brickColor || '#aa6644';
        const mortarColor = opts.mortarColor || '#ccbbaa';
        
        // Fill mortar
        ctx.fillStyle = mortarColor;
        ctx.fillRect(px, py, w, h);
        
        // Draw bricks
        ctx.fillStyle = brickColor;
        for (let row = 0; row < Math.ceil(h / brickH); row++) {
            const offset = row % 2 === 0 ? 0 : brickW / 2;
            for (let col = -1; col < Math.ceil(w / brickW) + 1; col++) {
                const bx = px + col * brickW + offset + mortar / 2;
                const by = py + row * brickH + mortar / 2;
                ctx.fillRect(bx, by, brickW - mortar, brickH - mortar);
            }
        }
        
        this._dirty = true;
        return uv;
    }
    
    addWindowPattern(name, w = 64, h = 64) {
        const uv = this._alloc(name, w, h);
        if (!uv) return null;
        const { px, py } = uv;
        const ctx = this.ctx;
        
        // Wall
        ctx.fillStyle = '#334455';
        ctx.fillRect(px, py, w, h);
        
        // Window
        const ww = w * 0.6, wh = h * 0.6;
        const wx = px + (w - ww) / 2, wy = py + (h - wh) / 2;
        
        ctx.fillStyle = '#aaccff';
        ctx.fillRect(wx, wy, ww, wh);
        
        // Frame
        ctx.strokeStyle = '#888';
        ctx.lineWidth = 2;
        ctx.strokeRect(wx, wy, ww, wh);
        ctx.beginPath();
        ctx.moveTo(wx + ww / 2, wy);
        ctx.lineTo(wx + ww / 2, wy + wh);
        ctx.moveTo(wx, wy + wh / 2);
        ctx.lineTo(wx + ww, wy + wh / 2);
        ctx.stroke();
        
        this._dirty = true;
        return uv;
    }
    
    remapGeometryUV(geometry, tileName) {
        const uv = this.tiles.get(tileName);
        if (!uv) {
            console.warn(`Tile "${tileName}" not found in atlas`);
            return geometry;
        }
        
        const geo = geometry.clone();
        const uvAttr = geo.attributes.uv;
        
        for (let i = 0; i < uvAttr.count; i++) {
            uvAttr.setX(i, uvAttr.getX(i) * uv.uSize + uv.u);
            uvAttr.setY(i, uvAttr.getY(i) * uv.vSize + uv.v);
        }
        uvAttr.needsUpdate = true;
        return geo;
    }
    
    _alloc(name, w, h) {
        const rect = this.packer.pack(w, h);
        if (!rect) {
            console.error(`Atlas full! Cannot fit "${name}" (${w}x${h})`);
            return null;
        }
        
        const uvInfo = {
            u: rect.x / this.size,
            v: rect.y / this.size,
            uSize: w / this.size,
            vSize: h / this.size,
            px: rect.x,
            py: rect.y,
            pw: w,
            ph: h
        };
        
        this.tiles.set(name, uvInfo);
        return uvInfo;
    }
    
    getStats() {
        return {
            tileCount: this.tiles.size,
            efficiency: (this.packer.usedArea / (this.size * this.size) * 100).toFixed(1) + '%',
            size: `${this.size}x${this.size}`,
            tiles: Array.from(this.tiles.entries()).map(([name, uv]) => ({
                name,
                size: `${uv.pw}x${uv.ph}`,
                uv: `(${uv.u.toFixed(3)}, ${uv.v.toFixed(3)}) + (${uv.uSize.toFixed(3)}, ${uv.vSize.toFixed(3)})`
            }))
        };
    }
}
```

---

## 7. Full Interactive Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Texture Atlas Demo</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #0a0a1a; color: #eee; font-family: monospace; overflow: hidden; }
        #canvas-container { position: absolute; inset: 0; }
        #ui {
            position: absolute; top: 20px; left: 20px;
            background: rgba(0,0,20,0.92); padding: 20px;
            border-radius: 8px; border: 1px solid #0af;
            width: 320px; z-index: 10;
        }
        h2 { color: #0af; margin-bottom: 12px; font-size: 14px; }
        .metric { display: flex; justify-content: space-between; margin: 5px 0; font-size: 12px; }
        .good { color: #4f4; font-weight: bold; }
        .bad { color: #f44; font-weight: bold; }
        .normal { color: #fff; }
        .btn-group { display: flex; gap: 6px; margin: 12px 0; flex-wrap: wrap; }
        button {
            padding: 6px 12px; background: #1a1a3a;
            color: #eee; border: 1px solid #446;
            border-radius: 4px; cursor: pointer; font-size: 11px;
        }
        button.active { background: #0af; color: #000; border-color: #0af; }
        #atlas-preview {
            width: 100%; margin-top: 12px;
            border: 1px solid #446; border-radius: 4px;
        }
        .section-title { color: #fa0; font-size: 11px; margin: 10px 0 5px; }
    </style>
</head>
<body>
<div id="canvas-container"><canvas id="c"></canvas></div>
<div id="ui">
    <h2>🖼️ Texture Atlas Demo</h2>
    
    <div class="section-title">โหมด</div>
    <div class="btn-group">
        <button id="btn-separate" class="active">แยก Texture</button>
        <button id="btn-atlas">Atlas Texture</button>
        <button id="btn-sprite">Sprite Anim</button>
    </div>
    
    <div class="section-title">ประสิทธิภาพ</div>
    <div class="metric"><span>Draw Calls:</span><span id="m-calls" class="normal">-</span></div>
    <div class="metric"><span>Textures ใน GPU:</span><span id="m-textures" class="normal">-</span></div>
    <div class="metric"><span>FPS:</span><span id="m-fps" class="normal">-</span></div>
    <div class="metric"><span>Texture Count:</span><span id="m-texcount" class="normal">-</span></div>
    
    <div class="section-title">Atlas Preview</div>
    <canvas id="atlas-preview"></canvas>
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
    import { mergeGeometries } from 'three/addons/utils/BufferGeometryUtils.js';
    
    // ============== Atlas Utilities ==============
    class ShelfPacker {
        constructor(w, h) {
            this.w = w; this.h = h;
            this.shelves = [{ x: 0, y: 0, h: 0, rw: w }];
            this.usedArea = 0;
        }
        pack(w, h) {
            for (const s of this.shelves) {
                if (s.rw >= w && s.h >= h) {
                    const x = this.w - s.rw; const y = s.y;
                    s.rw -= w; this.usedArea += w * h;
                    return { x, y, w, h };
                }
            }
            const last = this.shelves[this.shelves.length - 1];
            const ny = last.y + last.h;
            if (ny + h > this.h) return null;
            this.shelves.push({ x: 0, y: ny, h, rw: this.w - w });
            this.usedArea += w * h;
            return { x: 0, y: ny, w, h };
        }
    }
    
    class Atlas {
        constructor(size = 1024) {
            this.size = size;
            this.canvas = document.createElement('canvas');
            this.canvas.width = size; this.canvas.height = size;
            this.ctx = this.canvas.getContext('2d');
            this.tiles = new Map();
            this.packer = new ShelfPacker(size, size);
        }
        
        addFn(name, fn, w = 128, h = 128) {
            const r = this.packer.pack(w, h);
            if (!r) return null;
            fn(this.ctx, r.x, r.y, w, h);
            const uv = { u: r.x/this.size, v: r.y/this.size, us: w/this.size, vs: h/this.size };
            this.tiles.set(name, uv);
            return uv;
        }
        
        remap(geo, name) {
            const uv = this.tiles.get(name);
            if (!uv) return geo;
            const g = geo.clone();
            const a = g.attributes.uv;
            for (let i = 0; i < a.count; i++) {
                a.setXY(i, a.getX(i) * uv.us + uv.u, a.getY(i) * uv.vs + uv.v);
            }
            a.needsUpdate = true;
            return g;
        }
        
        makeTexture() {
            const t = new THREE.CanvasTexture(this.canvas);
            t.minFilter = THREE.LinearFilter;
            return t;
        }
        
        updatePreview(previewCanvas) {
            const pCtx = previewCanvas.getContext('2d');
            const scale = previewCanvas.width / this.size;
            previewCanvas.height = previewCanvas.width;
            pCtx.drawImage(this.canvas, 0, 0, previewCanvas.width, previewCanvas.height);
            pCtx.strokeStyle = 'rgba(255,100,0,0.5)';
            pCtx.lineWidth = 1;
            this.tiles.forEach((uv, name) => {
                const x = uv.u * previewCanvas.width;
                const y = uv.v * previewCanvas.height;
                const w = uv.us * previewCanvas.width;
                const h = uv.vs * previewCanvas.height;
                pCtx.strokeRect(x, y, w, h);
            });
        }
    }
    
    // ============== Procedural Texture Functions ==============
    function drawBrick(ctx, x, y, w, h) {
        ctx.fillStyle = '#cc8866'; ctx.fillRect(x, y, w, h);
        ctx.fillStyle = '#554433';
        for (let row = 0; row * 12 < h; row++) {
            const off = row % 2 ? 16 : 0;
            for (let col = -1; col * 32 < w + 32; col++) {
                ctx.fillRect(x + col*32 + off + 1, y + row*12 + 1, 30, 10);
            }
        }
    }
    
    function drawConcrete(ctx, x, y, w, h) {
        ctx.fillStyle = '#8899aa'; ctx.fillRect(x, y, w, h);
        for (let i = 0; i < 100; i++) {
            const px = x + Math.random() * w, py = y + Math.random() * h;
            const gray = 100 + Math.random() * 80;
            ctx.fillStyle = `rgb(${gray},${gray},${gray})`;
            ctx.fillRect(px, py, 2, 2);
        }
    }
    
    function drawGlass(ctx, x, y, w, h) {
        const grad = ctx.createLinearGradient(x, y, x + w, y + h);
        grad.addColorStop(0, '#99bbff');
        grad.addColorStop(0.5, '#ccddff');
        grad.addColorStop(1, '#8899ff');
        ctx.fillStyle = grad;
        ctx.fillRect(x, y, w, h);
        ctx.strokeStyle = 'rgba(255,255,255,0.5)';
        ctx.lineWidth = 2;
        ctx.strokeRect(x+2, y+2, w-4, h-4);
    }
    
    function drawGrass(ctx, x, y, w, h) {
        ctx.fillStyle = '#336622'; ctx.fillRect(x, y, w, h);
        for (let i = 0; i < 200; i++) {
            const px = x + Math.random()*w, py = y + Math.random()*h;
            const g = 40 + Math.random()*60;
            ctx.fillStyle = `rgb(${g/2},${g},${g/4})`;
            ctx.fillRect(px, py, 1, 1 + Math.random()*3);
        }
    }
    
    function drawRoad(ctx, x, y, w, h) {
        ctx.fillStyle = '#2a2a2a'; ctx.fillRect(x, y, w, h);
        ctx.fillStyle = '#ffff00';
        for (let i = 0; i * 20 < h; i++) {
            if (i % 2 === 0) ctx.fillRect(x + w/2 - 2, y + i*20, 4, 14);
        }
    }
    
    // ============== Setup Three.js ==============
    const renderer = new THREE.WebGLRenderer({ canvas: document.getElementById('c'), antialias: true });
    renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
    
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x05050f);
    
    const camera = new THREE.PerspectiveCamera(60, 1, 0.1, 500);
    camera.position.set(0, 30, 80);
    
    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.autoRotate = true;
    controls.autoRotateSpeed = 0.4;
    
    scene.add(new THREE.AmbientLight(0xffffff, 0.6));
    const sun = new THREE.DirectionalLight(0xffeedd, 2);
    sun.position.set(50, 100, 50);
    scene.add(sun);
    
    let currentMode = 'separate';
    let separateTextureCount = 0;
    let atlasTextureCount = 0;
    
    function clearScene() {
        const toRemove = [];
        scene.traverse(obj => {
            if (obj.isMesh || obj.isInstancedMesh) toRemove.push(obj);
        });
        toRemove.forEach(obj => {
            if (obj.geometry) obj.geometry.dispose();
            if (obj.material) {
                if (Array.isArray(obj.material)) {
                    obj.material.forEach(m => { if (m.map) m.map.dispose(); m.dispose(); });
                } else {
                    if (obj.material.map) obj.material.map.dispose();
                    obj.material.dispose();
                }
            }
            scene.remove(obj);
        });
    }
    
    // ============== Mode 1: Separate Textures ==============
    function buildSeparate() {
        clearScene();
        
        const textureTypes = ['brick', 'concrete', 'glass', 'grass', 'road'];
        separateTextureCount = textureTypes.length;
        
        const drawFns = { brick: drawBrick, concrete: drawConcrete, glass: drawGlass, grass: drawGrass, road: drawRoad };
        
        // สร้าง separate canvas texture สำหรับแต่ละ type
        const materials = {};
        textureTypes.forEach(type => {
            const canvas = document.createElement('canvas');
            canvas.width = canvas.height = 128;
            drawFns[type](canvas.getContext('2d'), 0, 0, 128, 128);
            const tex = new THREE.CanvasTexture(canvas);
            materials[type] = new THREE.MeshStandardMaterial({ map: tex });
        });
        
        // สร้าง buildings
        for (let i = 0; i < 60; i++) {
            const type = textureTypes[i % textureTypes.length];
            const w = 2 + Math.random() * 4, h = 4 + Math.random() * 15, d = 2 + Math.random() * 4;
            const geo = new THREE.BoxGeometry(w, h, d);
            const mesh = new THREE.Mesh(geo, materials[type]);
            mesh.position.set((Math.random()-0.5)*120, h/2, (Math.random()-0.5)*120);
            scene.add(mesh);
        }
        
        // Ground
        const groundCanvas = document.createElement('canvas');
        groundCanvas.width = groundCanvas.height = 128;
        drawGrass(groundCanvas.getContext('2d'), 0, 0, 128, 128);
        const groundTex = new THREE.CanvasTexture(groundCanvas);
        groundTex.wrapS = groundTex.wrapT = THREE.RepeatWrapping;
        groundTex.repeat.set(20, 20);
        const ground = new THREE.Mesh(
            new THREE.PlaneGeometry(200, 200),
            new THREE.MeshStandardMaterial({ map: groundTex })
        );
        ground.rotation.x = -Math.PI/2;
        scene.add(ground);
        
        document.getElementById('m-texcount').textContent = `${separateTextureCount} แยก`;
        document.getElementById('atlas-preview').style.display = 'none';
    }
    
    // ============== Mode 2: Atlas Texture ==============
    function buildAtlas() {
        clearScene();
        
        const atlas = new Atlas(512);
        atlas.addFn('brick', drawBrick, 128, 128);
        atlas.addFn('concrete', drawConcrete, 128, 128);
        atlas.addFn('glass', drawGlass, 128, 128);
        atlas.addFn('grass', drawGrass, 128, 128);
        atlas.addFn('road', drawRoad, 128, 128);
        
        const atlasTexture = atlas.makeTexture();
        const atlasMat = new THREE.MeshStandardMaterial({ map: atlasTexture });
        atlasTextureCount = 1;
        
        const types = ['brick', 'concrete', 'glass', 'grass'];
        const geos = [];
        
        for (let i = 0; i < 60; i++) {
            const type = types[i % types.length];
            const w = 2 + Math.random() * 4, h = 4 + Math.random() * 15, d = 2 + Math.random() * 4;
            
            const baseGeo = new THREE.BoxGeometry(w, h, d);
            const remapped = atlas.remap(baseGeo, type);
            remapped.applyMatrix4(new THREE.Matrix4().setPosition(
                (Math.random()-0.5)*120, h/2, (Math.random()-0.5)*120
            ));
            geos.push(remapped);
            baseGeo.dispose();
        }
        
        // Ground
        const groundBase = new THREE.PlaneGeometry(200, 200);
        groundBase.rotateX(-Math.PI/2);
        const groundRemapped = atlas.remap(groundBase, 'grass');
        geos.push(groundRemapped);
        groundBase.dispose();
        
        const merged = mergeGeometries(geos);
        geos.forEach(g => g.dispose());
        scene.add(new THREE.Mesh(merged, atlasMat));
        
        // Update atlas preview
        const preview = document.getElementById('atlas-preview');
        preview.style.display = 'block';
        atlas.updatePreview(preview);
        
        document.getElementById('m-texcount').textContent = `1 atlas (${atlas.tiles.size} tiles)`;
    }
    
    // ============== Mode 3: Sprite Animation ==============
    function buildSprite() {
        clearScene();
        
        // สร้าง sprite sheet 4x4 = 16 frames ของ explosion
        const TILES = 4;
        const TILE_SIZE = 64;
        const SHEET_SIZE = TILES * TILE_SIZE;
        
        const sheetCanvas = document.createElement('canvas');
        sheetCanvas.width = sheetCanvas.height = SHEET_SIZE;
        const sCtx = sheetCanvas.getContext('2d');
        
        // วาด frames ของ explosion
        for (let row = 0; row < TILES; row++) {
            for (let col = 0; col < TILES; col++) {
                const frame = row * TILES + col;
                const progress = frame / (TILES * TILES - 1);
                const x = col * TILE_SIZE;
                const y = row * TILE_SIZE;
                
                // Background
                sCtx.fillStyle = '#000';
                sCtx.fillRect(x, y, TILE_SIZE, TILE_SIZE);
                
                // Explosion rings
                const maxR = TILE_SIZE / 2 * progress;
                for (let r = 0; r <= 3; r++) {
                    const radius = maxR * (1 - r * 0.2);
                    const alpha = 1 - r * 0.25;
                    const grad = sCtx.createRadialGradient(
                        x + TILE_SIZE/2, y + TILE_SIZE/2, 0,
                        x + TILE_SIZE/2, y + TILE_SIZE/2, radius
                    );
                    grad.addColorStop(0, `rgba(255,255,200,${alpha})`);
                    grad.addColorStop(0.3, `rgba(255,150,0,${alpha * 0.8})`);
                    grad.addColorStop(0.7, `rgba(200,50,0,${alpha * 0.5})`);
                    grad.addColorStop(1, 'rgba(0,0,0,0)');
                    sCtx.fillStyle = grad;
                    sCtx.beginPath();
                    sCtx.arc(x + TILE_SIZE/2, y + TILE_SIZE/2, radius || 1, 0, Math.PI*2);
                    sCtx.fill();
                }
            }
        }
        
        const sheetTex = new THREE.CanvasTexture(sheetCanvas);
        sheetTex.minFilter = THREE.NearestFilter;
        sheetTex.magFilter = THREE.NearestFilter;
        sheetTex.repeat.set(1/TILES, 1/TILES);
        
        const spriteMat = new THREE.MeshBasicMaterial({
            map: sheetTex,
            transparent: true,
            depthWrite: false,
            side: THREE.DoubleSide
        });
        
        const sprites = [];
        for (let i = 0; i < 20; i++) {
            const plane = new THREE.Mesh(new THREE.PlaneGeometry(5, 5), spriteMat.clone());
            plane.position.set((Math.random()-0.5)*80, 2 + Math.random()*20, (Math.random()-0.5)*80);
            plane.userData.frameOffset = Math.floor(Math.random() * TILES * TILES);
            scene.add(plane);
            sprites.push(plane);
        }
        
        // Ground
        scene.add(new THREE.Mesh(
            new THREE.PlaneGeometry(200, 200),
            new THREE.MeshStandardMaterial({ color: 0x1a1a2a })
        ));
        scene.children[scene.children.length - 1].rotation.x = -Math.PI/2;
        
        // Animate sprites
        let currentFrame = 0;
        let lastSpriteUpdate = 0;
        window._spriteAnimate = (time) => {
            if (time - lastSpriteUpdate > 80) {
                currentFrame = (currentFrame + 1) % (TILES * TILES);
                lastSpriteUpdate = time;
                
                sprites.forEach(sprite => {
                    const frame = (currentFrame + sprite.userData.frameOffset) % (TILES * TILES);
                    const col = frame % TILES;
                    const row = Math.floor(frame / TILES);
                    sprite.material.map.offset.set(col / TILES, 1 - (row + 1) / TILES);
                    // Billboard: face camera
                    sprite.lookAt(camera.position);
                });
            }
        };
        
        // Update atlas preview with sprite sheet
        const preview = document.getElementById('atlas-preview');
        preview.style.display = 'block';
        const pCtx = preview.getContext('2d');
        preview.height = preview.width;
        pCtx.drawImage(sheetCanvas, 0, 0, preview.width, preview.height);
        
        document.getElementById('m-texcount').textContent = `1 sprite sheet (${TILES*TILES} frames)`;
    }
    
    buildSeparate();
    
    function setMode(mode) {
        currentMode = mode;
        document.querySelectorAll('button').forEach(b => b.classList.remove('active'));
        document.getElementById(`btn-${mode}`).classList.add('active');
        window._spriteAnimate = null;
        
        if (mode === 'separate') buildSeparate();
        else if (mode === 'atlas') buildAtlas();
        else if (mode === 'sprite') buildSprite();
    }
    
    document.getElementById('btn-separate').addEventListener('click', () => setMode('separate'));
    document.getElementById('btn-atlas').addEventListener('click', () => setMode('atlas'));
    document.getElementById('btn-sprite').addEventListener('click', () => setMode('sprite'));
    
    let lastTime = performance.now(), frames = 0, fps = 0;
    
    function resize() {
        renderer.setSize(innerWidth, innerHeight);
        camera.aspect = innerWidth / innerHeight;
        camera.updateProjectionMatrix();
    }
    window.addEventListener('resize', resize);
    resize();
    
    function animate(time) {
        requestAnimationFrame(animate);
        if (window._spriteAnimate) window._spriteAnimate(time);
        controls.update();
        renderer.render(scene, camera);
        
        frames++;
        const now = performance.now();
        if (now - lastTime >= 500) {
            fps = Math.round(frames / ((now - lastTime) / 1000));
            frames = 0; lastTime = now;
        }
        
        const calls = renderer.info.render.calls;
        document.getElementById('m-calls').textContent = calls;
        document.getElementById('m-calls').className = calls > 50 ? 'bad' : 'good';
        document.getElementById('m-textures').textContent = renderer.info.memory.textures;
        document.getElementById('m-fps').textContent = fps;
    }
    
    animate(0);
</script>
</body>
</html>
```

---

## 8. Atlas Limitations และ Solutions

```javascript
// ปัญหา 1: Texture Bleeding (สีจาก tile ข้างๆ ซึมเข้ามา)
// สาเหตุ: Bilinear filtering อ่าน pixels ข้างเคียง
// วิธีแก้:
atlas.padding = 4; // เพิ่ม padding รอบ tile
// หรือใช้ GL_NEAREST filter:
texture.minFilter = THREE.NearestFilter;
texture.magFilter = THREE.NearestFilter;

// ปัญหา 2: Mipmap บน Atlas ทำให้ border ซึม
// วิธีแก้: ปิด mipmaps หรือใช้ separate atlas per mip level
texture.generateMipmaps = false;
texture.minFilter = THREE.LinearFilter; // ไม่ใช้ mip

// ปัญหา 3: UV Clamping ไม่ทำงานกับ Atlas
// สาเหตุ: UV wrap ของ atlas จะ wrap ทั้ง atlas ไม่ใช่แค่ tile
// วิธีแก้: ใช้ custom UV clamping ใน shader
const material = new THREE.MeshStandardMaterial();
material.onBeforeCompile = (shader) => {
    shader.fragmentShader = shader.fragmentShader.replace(
        '#include <map_fragment>',
        `
        // Custom UV clamping ภายใน tile bounds
        vec2 tileOffset = vec2(${uvInfo.u.toFixed(4)}, ${uvInfo.v.toFixed(4)});
        vec2 tileSize = vec2(${uvInfo.uSize.toFixed(4)}, ${uvInfo.vSize.toFixed(4)});
        vec2 clampedUV = tileOffset + clamp(vMapUv - tileOffset, vec2(0.001), tileSize - vec2(0.001));
        vec4 texelColor = texture2D(map, clampedUV);
        diffuseColor *= texelColor;
        `
    );
};
```

---

## แบบฝึกหัด

1. **ง่าย**: สร้าง texture atlas 2x2 (4 tiles) และ apply กับ 4 buildings แต่ละหลังมี texture ต่างกัน โดยใช้ material เดียว
2. **กลาง**: สร้าง sprite animation ของตัวละครที่มี 4 ทิศทาง (up, down, left, right) แต่ละทิศ 4 frames
3. **ยาก**: สร้าง `AutoAtlasBuilder` ที่รับ array ของ canvas draw functions และจัด pack อัตโนมัติ พร้อม UV output สำหรับแต่ละ tile
4. **Challenge**: Implement atlas สำหรับ normal map และ roughness map ด้วย PBR material

---

## อ้างอิง

- [Texture Atlas Technique](https://en.wikipedia.org/wiki/Texture_atlas)
- [Shelf-Next-Fit Algorithm](https://pds25.egloos.com/pds/201504/21/98/RectangleBinPack.pdf)
- [Three.js CanvasTexture](https://threejs.org/docs/#api/en/textures/CanvasTexture)
- [Sprite Animation in Three.js](https://threejs.org/examples/#webgl_materials_texture_anisotropy)
