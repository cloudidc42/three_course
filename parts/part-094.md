# Part 94: 3D UI Components

## บทนำ

การสร้าง User Interface ใน 3D space เป็นงานที่ท้าทายแต่ให้ผลลัพธ์ที่น่าทึ่ง ในบทนี้เราจะสร้าง reusable 3D UI component library ตั้งแต่ปุ่มธรรมดาไปถึง panel, modal, และ layout manager

## พื้นฐาน: Raycaster สำหรับ 3D Interaction

ทุก 3D UI component ต้องพึ่งพา raycaster ในการตรวจจับ mouse events:

```javascript
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();

// อัพเดต mouse position
window.addEventListener('mousemove', (e) => {
    mouse.x = (e.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(e.clientY / window.innerHeight) * 2 + 1;
});

// ใน animation loop
function checkIntersections() {
    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(interactableObjects, true);
    
    if (intersects.length > 0) {
        const obj = intersects[0].object;
        // handle hover...
    }
}
```

## Base UIComponent Class

```javascript
class UIComponent extends THREE.Group {
    constructor(options = {}) {
        super();
        this.options = {
            width: 200,
            height: 50,
            color: 0x2d2d4e,
            hoverColor: 0x3d3d6e,
            activeColor: 0x1a1a3e,
            textColor: '#ffffff',
            ...options,
        };
        this.isHovered = false;
        this.isActive = false;
        this._callbacks = {};
        this.build();
    }
    
    build() { /* override ใน subclass */ }
    
    on(event, callback) {
        this._callbacks[event] = this._callbacks[event] || [];
        this._callbacks[event].push(callback);
        return this;
    }
    
    emit(event, data) {
        (this._callbacks[event] || []).forEach(cb => cb(data));
    }
    
    onHoverEnter() {
        this.isHovered = true;
        this.emit('hoverEnter');
    }
    
    onHoverLeave() {
        this.isHovered = false;
        this.emit('hoverLeave');
    }
    
    onClick() {
        this.emit('click');
    }
    
    update(dt) { /* override สำหรับ animation */ }
}
```

## 3D Button Component

```javascript
class Button3D extends UIComponent {
    constructor(text, options = {}) {
        super({ width: 160, height: 44, ...options });
        this.text = text;
        this.build();
    }
    
    build() {
        const { width, height, color } = this.options;
        
        // Background mesh
        const geo = new THREE.RoundedRectGeometry ?
            new THREE.RoundedRectGeometry(width, height, 8) :
            new THREE.PlaneGeometry(width, height);
        
        this.bgMat = new THREE.MeshStandardMaterial({
            color,
            roughness: 0.3,
            metalness: 0.1,
        });
        this.bgMesh = new THREE.Mesh(geo, this.bgMat);
        this.add(this.bgMesh);
        
        // Border
        const borderGeo = new THREE.EdgesGeometry(geo);
        const borderMat = new THREE.LineBasicMaterial({ color: 0x5555aa });
        this.border = new THREE.LineSegments(borderGeo, borderMat);
        this.border.position.z = 0.5;
        this.add(this.border);
        
        // Label (ใช้ Canvas Texture)
        this.labelTexture = this.createTextTexture(this.text);
        const labelGeo = new THREE.PlaneGeometry(width * 0.9, height * 0.7);
        const labelMat = new THREE.MeshBasicMaterial({
            map: this.labelTexture,
            transparent: true,
        });
        this.labelMesh = new THREE.Mesh(labelGeo, labelMat);
        this.labelMesh.position.z = 1;
        this.add(this.labelMesh);
        
        // Shadow plane
        const shadowGeo = new THREE.PlaneGeometry(width + 10, height + 10);
        const shadowMat = new THREE.MeshBasicMaterial({
            color: 0x000000,
            transparent: true,
            opacity: 0.3,
        });
        this.shadowMesh = new THREE.Mesh(shadowGeo, shadowMat);
        this.shadowMesh.position.set(3, -3, -0.5);
        this.add(this.shadowMesh);
    }
    
    createTextTexture(text) {
        const canvas = document.createElement('canvas');
        canvas.width = 512;
        canvas.height = 128;
        const ctx = canvas.getContext('2d');
        
        ctx.clearRect(0, 0, 512, 128);
        ctx.font = 'bold 42px Arial';
        ctx.fillStyle = this.options.textColor;
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';
        ctx.fillText(text, 256, 64);
        
        const texture = new THREE.CanvasTexture(canvas);
        return texture;
    }
    
    onHoverEnter() {
        super.onHoverEnter();
        this.bgMat.color.setHex(this.options.hoverColor);
        // animate up
        this.targetY = 3;
    }
    
    onHoverLeave() {
        super.onHoverLeave();
        this.bgMat.color.setHex(this.options.color);
        this.targetY = 0;
    }
    
    onClick() {
        super.onClick();
        // press animation
        this.bgMat.color.setHex(this.options.activeColor);
        setTimeout(() => {
            if (this.isHovered) {
                this.bgMat.color.setHex(this.options.hoverColor);
            } else {
                this.bgMat.color.setHex(this.options.color);
            }
        }, 150);
    }
    
    update(dt) {
        if (this.targetY !== undefined) {
            this.position.y += (this.targetY - this.position.y) * 0.2;
        }
    }
}
```

## 3D Slider Component

```javascript
class Slider3D extends UIComponent {
    constructor(options = {}) {
        super({
            width: 200,
            height: 20,
            min: 0,
            max: 1,
            value: 0.5,
            trackColor: 0x222244,
            fillColor: 0x4466ff,
            thumbColor: 0xffffff,
            ...options,
        });
        this.isDragging = false;
    }
    
    build() {
        const { width, height, trackColor, fillColor, thumbColor, value, min, max } = this.options;
        
        // Track
        const trackGeo = new THREE.BoxGeometry(width, height * 0.3, 4);
        const trackMat = new THREE.MeshStandardMaterial({ color: trackColor, roughness: 0.5 });
        this.trackMesh = new THREE.Mesh(trackGeo, trackMat);
        this.add(this.trackMesh);
        
        // Fill
        const normalValue = (value - min) / (max - min);
        this.fillGeo = new THREE.BoxGeometry(width * normalValue, height * 0.3, 5);
        this.fillMat = new THREE.MeshStandardMaterial({ color: fillColor, roughness: 0.3 });
        this.fillMesh = new THREE.Mesh(this.fillGeo, this.fillMat);
        this.fillMesh.position.x = -width / 2 + (width * normalValue) / 2;
        this.add(this.fillMesh);
        
        // Thumb
        const thumbGeo = new THREE.SphereGeometry(height * 0.7, 16, 16);
        const thumbMat = new THREE.MeshStandardMaterial({ color: thumbColor, roughness: 0.1, metalness: 0.8 });
        this.thumbMesh = new THREE.Mesh(thumbGeo, thumbMat);
        this.thumbMesh.position.x = -width / 2 + width * normalValue;
        this.thumbMesh.position.z = 3;
        this.add(this.thumbMesh);
        
        this._value = value;
        this._normalValue = normalValue;
    }
    
    setValue(value) {
        const { width, min, max } = this.options;
        this._value = Math.max(min, Math.min(max, value));
        this._normalValue = (this._value - min) / (max - min);
        
        // Update fill
        this.fillMesh.scale.x = this._normalValue;
        this.fillMesh.position.x = -width / 2 + (width * this._normalValue) / 2;
        
        // Update thumb
        this.thumbMesh.position.x = -width / 2 + width * this._normalValue;
        
        this.emit('change', this._value);
    }
    
    getValue() { return this._value; }
    
    // เรียกจาก interaction manager เมื่อ drag
    handleDrag(normalizedX) {
        this.setValue(this.options.min + normalizedX * (this.options.max - this.options.min));
    }
}
```

## 3D Toggle/Checkbox

```javascript
class Toggle3D extends UIComponent {
    constructor(options = {}) {
        super({
            width: 60,
            height: 30,
            onColor: 0x4466ff,
            offColor: 0x333355,
            checked: false,
            ...options,
        });
        this._checked = options.checked || false;
        this.animProgress = this._checked ? 1 : 0;
    }
    
    build() {
        const { width, height, onColor, offColor } = this.options;
        
        // Track
        const trackGeo = new THREE.BoxGeometry(width, height, height * 0.5);
        // สร้าง rounded look ด้วย capsule
        this.trackMat = new THREE.MeshStandardMaterial({
            color: this._checked ? onColor : offColor,
            roughness: 0.3,
        });
        this.trackMesh = new THREE.Mesh(trackGeo, this.trackMat);
        this.add(this.trackMesh);
        
        // Thumb
        const thumbRadius = height * 0.38;
        const thumbGeo = new THREE.SphereGeometry(thumbRadius, 20, 20);
        const thumbMat = new THREE.MeshStandardMaterial({
            color: 0xffffff,
            roughness: 0.1,
            metalness: 0.5,
        });
        this.thumbMesh = new THREE.Mesh(thumbGeo, thumbMat);
        this.thumbMesh.position.set(
            this._checked ? width * 0.3 : -width * 0.3,
            0,
            height * 0.35
        );
        this.add(this.thumbMesh);
    }
    
    setChecked(checked) {
        this._checked = checked;
        this.emit('change', checked);
    }
    
    toggle() {
        this.setChecked(!this._checked);
    }
    
    onClick() {
        this.toggle();
        super.onClick();
    }
    
    update(dt) {
        const target = this._checked ? 1 : 0;
        this.animProgress += (target - this.animProgress) * 0.15;
        
        const { width, height, onColor, offColor } = this.options;
        this.thumbMesh.position.x = (-width * 0.3) + (width * 0.6) * this.animProgress;
        
        // interpolate color
        const col = new THREE.Color(offColor).lerp(new THREE.Color(onColor), this.animProgress);
        this.trackMat.color.copy(col);
    }
    
    isChecked() { return this._checked; }
}
```

## 3D Panel/Window

```javascript
class Panel3D extends UIComponent {
    constructor(title, options = {}) {
        super({
            width: 300,
            height: 200,
            color: 0x1a1a2e,
            borderColor: 0x3333aa,
            titleBarColor: 0x2a2a5e,
            titleBarHeight: 36,
            padding: 16,
            ...options,
        });
        this.title = title;
        this.children3D = []; // UI components ข้างใน
    }
    
    build() {
        const { width, height, color, borderColor, titleBarColor, titleBarHeight } = this.options;
        
        // Background
        const bgGeo = new THREE.PlaneGeometry(width, height);
        const bgMat = new THREE.MeshStandardMaterial({
            color,
            roughness: 0.5,
            transparent: true,
            opacity: 0.95,
        });
        this.bgMesh = new THREE.Mesh(bgGeo, bgMat);
        this.add(this.bgMesh);
        
        // Border
        const borderGeo = new THREE.EdgesGeometry(bgGeo);
        const borderMat = new THREE.LineBasicMaterial({ color: borderColor });
        this.add(new THREE.LineSegments(borderGeo, borderMat));
        
        // Title bar
        const titleGeo = new THREE.PlaneGeometry(width, titleBarHeight);
        const titleMat = new THREE.MeshStandardMaterial({ color: titleBarColor });
        this.titleBar = new THREE.Mesh(titleGeo, titleMat);
        this.titleBar.position.set(0, (height - titleBarHeight) / 2, 0.5);
        this.add(this.titleBar);
        
        // Title text
        const titleTexture = this.createTitleTexture(this.title);
        const titleLabelGeo = new THREE.PlaneGeometry(width * 0.9, titleBarHeight * 0.8);
        this.titleLabel = new THREE.Mesh(titleLabelGeo, new THREE.MeshBasicMaterial({
            map: titleTexture,
            transparent: true,
        }));
        this.titleLabel.position.set(-20, (height - titleBarHeight) / 2, 1);
        this.add(this.titleLabel);
        
        // Close button
        this.closeBtn = this.createCloseButton();
        this.closeBtn.position.set(width / 2 - 20, (height - titleBarHeight) / 2, 1);
        this.add(this.closeBtn);
    }
    
    createTitleTexture(text) {
        const canvas = document.createElement('canvas');
        canvas.width = 512; canvas.height = 64;
        const ctx = canvas.getContext('2d');
        ctx.font = 'bold 28px Arial';
        ctx.fillStyle = '#aabbff';
        ctx.textBaseline = 'middle';
        ctx.fillText(text, 10, 32);
        return new THREE.CanvasTexture(canvas);
    }
    
    createCloseButton() {
        const geo = new THREE.CircleGeometry(8, 16);
        const mat = new THREE.MeshBasicMaterial({ color: 0xff4444 });
        const mesh = new THREE.Mesh(geo, mat);
        return mesh;
    }
    
    addChild(component, x, y) {
        component.position.set(x, y, 1);
        this.add(component);
        this.children3D.push(component);
        return this;
    }
    
    show() {
        this.visible = true;
        this.targetScale = 1;
    }
    
    hide() {
        this.targetScale = 0;
        setTimeout(() => { this.visible = false; }, 300);
    }
    
    update(dt) {
        if (this.targetScale !== undefined) {
            this.scale.setScalar(this.scale.x + (this.targetScale - this.scale.x) * 0.15);
        }
        this.children3D.forEach(c => c.update && c.update(dt));
    }
}
```

## 3D Progress Bar

```javascript
class ProgressBar3D extends UIComponent {
    constructor(options = {}) {
        super({
            width: 200,
            height: 16,
            value: 0,
            color: 0x4466ff,
            bgColor: 0x222244,
            showText: true,
            label: '',
            ...options,
        });
        this._value = options.value || 0;
    }
    
    build() {
        const { width, height, color, bgColor } = this.options;
        
        // Background
        const bgGeo = new THREE.BoxGeometry(width, height, 4);
        const bgMat = new THREE.MeshStandardMaterial({ color: bgColor, roughness: 0.5 });
        this.add(new THREE.Mesh(bgGeo, bgMat));
        
        // Fill
        this.fillMat = new THREE.MeshStandardMaterial({
            color,
            roughness: 0.3,
            emissive: new THREE.Color(color).multiplyScalar(0.3),
        });
        this.fillGeo = new THREE.BoxGeometry(width * this._value, height - 2, 5);
        this.fillMesh = new THREE.Mesh(this.fillGeo, this.fillMat);
        this.fillMesh.position.x = -width / 2 + (width * this._value) / 2;
        this.add(this.fillMesh);
    }
    
    setValue(value) {
        this._value = Math.max(0, Math.min(1, value));
        this.targetValue = this._value;
        this.emit('change', this._value);
    }
    
    update(dt) {
        if (this.targetValue === undefined) return;
        const { width, height } = this.options;
        
        if (this.currentDisplayValue === undefined) {
            this.currentDisplayValue = this._value;
        }
        
        this.currentDisplayValue += (this.targetValue - this.currentDisplayValue) * 0.1;
        
        const v = this.currentDisplayValue;
        this.fillMesh.scale.x = v;
        this.fillMesh.position.x = -width / 2 + (width * v) / 2;
        
        // glow effect near full
        if (v > 0.8) {
            this.fillMat.emissiveIntensity = (v - 0.8) * 5;
        }
    }
}
```

## 3D Tooltip

```javascript
class Tooltip3D {
    constructor(scene) {
        this.scene = scene;
        this.mesh = null;
        this.visible = false;
        this.targetOpacity = 0;
        this.currentOpacity = 0;
    }
    
    createMesh(text) {
        if (this.mesh) {
            this.scene.remove(this.mesh);
            this.mesh.geometry.dispose();
        }
        
        const canvas = document.createElement('canvas');
        const lines = text.split('\n');
        const maxLen = Math.max(...lines.map(l => l.length));
        canvas.width = Math.max(200, maxLen * 14 + 40);
        canvas.height = lines.length * 28 + 20;
        
        const ctx = canvas.getContext('2d');
        
        // Background
        ctx.fillStyle = 'rgba(20, 20, 50, 0.95)';
        ctx.roundRect(0, 0, canvas.width, canvas.height, 8);
        ctx.fill();
        
        // Border
        ctx.strokeStyle = 'rgba(100, 120, 255, 0.6)';
        ctx.lineWidth = 2;
        ctx.roundRect(1, 1, canvas.width - 2, canvas.height - 2, 8);
        ctx.stroke();
        
        // Text
        ctx.font = '16px Arial';
        ctx.fillStyle = '#ccddff';
        lines.forEach((line, i) => {
            ctx.fillText(line, 16, 28 + i * 28);
        });
        
        const texture = new THREE.CanvasTexture(canvas);
        const geo = new THREE.PlaneGeometry(canvas.width * 0.5, canvas.height * 0.5);
        const mat = new THREE.MeshBasicMaterial({
            map: texture,
            transparent: true,
            opacity: 0,
            depthTest: false,
        });
        
        this.mesh = new THREE.Mesh(geo, mat);
        this.scene.add(this.mesh);
        return this.mesh;
    }
    
    show(text, position) {
        this.createMesh(text);
        this.mesh.position.copy(position);
        this.mesh.position.y += 40;
        this.targetOpacity = 1;
    }
    
    hide() {
        this.targetOpacity = 0;
    }
    
    update(camera) {
        if (!this.mesh) return;
        
        this.currentOpacity += (this.targetOpacity - this.currentOpacity) * 0.15;
        this.mesh.material.opacity = this.currentOpacity;
        
        // Billboard
        this.mesh.quaternion.copy(camera.quaternion);
    }
}
```

## 3D Modal Dialog

```javascript
class Modal3D extends THREE.Group {
    constructor(title, content, options = {}) {
        super();
        this.options = {
            width: 350,
            height: 250,
            ...options,
        };
        this.title = title;
        this.content = content;
        this.isOpen = false;
        this.scale.setScalar(0);
        this.build();
    }
    
    build() {
        const { width, height } = this.options;
        
        // Overlay (semi-transparent backdrop)
        const overlayGeo = new THREE.PlaneGeometry(2000, 2000);
        const overlayMat = new THREE.MeshBasicMaterial({
            color: 0x000000,
            transparent: true,
            opacity: 0.5,
            depthTest: false,
        });
        this.overlay = new THREE.Mesh(overlayGeo, overlayMat);
        this.overlay.position.z = -1;
        this.add(this.overlay);
        
        // Main panel
        const panelGeo = new THREE.PlaneGeometry(width, height);
        const panelMat = new THREE.MeshStandardMaterial({
            color: 0x12122a,
            roughness: 0.4,
        });
        this.panel = new THREE.Mesh(panelGeo, panelMat);
        this.panel.position.z = 1;
        this.add(this.panel);
        
        // Title bar
        const titleBarGeo = new THREE.PlaneGeometry(width, 44);
        const titleBarMat = new THREE.MeshStandardMaterial({ color: 0x2222aa });
        this.titleBar = new THREE.Mesh(titleBarGeo, titleBarMat);
        this.titleBar.position.set(0, height / 2 - 22, 2);
        this.add(this.titleBar);
        
        // Text content
        this.updateContent(title, content);
        
        // Buttons
        this.confirmBtn = new Button3D('ยืนยัน', {
            width: 120, height: 36,
            color: 0x2244aa, hoverColor: 0x3355cc,
        });
        this.confirmBtn.position.set(70, -height / 2 + 40, 3);
        this.add(this.confirmBtn);
        
        this.cancelBtn = new Button3D('ยกเลิก', {
            width: 120, height: 36,
            color: 0x333344, hoverColor: 0x444466,
        });
        this.cancelBtn.position.set(-70, -height / 2 + 40, 3);
        this.add(this.cancelBtn);
        
        // Close button
        this.confirmBtn.on('click', () => this.close());
        this.cancelBtn.on('click', () => this.close());
    }
    
    updateContent(title, text) {
        if (this.titleTextureMesh) this.remove(this.titleTextureMesh);
        if (this.bodyTextureMesh) this.remove(this.bodyTextureMesh);
        
        const { width, height } = this.options;
        
        // Title texture
        const titleCanvas = document.createElement('canvas');
        titleCanvas.width = 512; titleCanvas.height = 64;
        const tc = titleCanvas.getContext('2d');
        tc.font = 'bold 28px Arial';
        tc.fillStyle = '#ffffff';
        tc.textBaseline = 'middle';
        tc.fillText(title, 16, 32);
        
        this.titleTextureMesh = new THREE.Mesh(
            new THREE.PlaneGeometry(width * 0.8, 32),
            new THREE.MeshBasicMaterial({ map: new THREE.CanvasTexture(titleCanvas), transparent: true })
        );
        this.titleTextureMesh.position.set(-width * 0.05, height / 2 - 22, 3);
        this.add(this.titleTextureMesh);
        
        // Body texture
        const bodyCanvas = document.createElement('canvas');
        bodyCanvas.width = 512; bodyCanvas.height = 256;
        const bc = bodyCanvas.getContext('2d');
        bc.font = '18px Arial';
        bc.fillStyle = '#aabbcc';
        bc.textBaseline = 'top';
        
        const words = text.split(' ');
        let line = '', y = 10;
        words.forEach(word => {
            const test = line + word + ' ';
            if (bc.measureText(test).width > 490) {
                bc.fillText(line, 10, y);
                line = word + ' ';
                y += 26;
            } else {
                line = test;
            }
        });
        bc.fillText(line, 10, y);
        
        this.bodyTextureMesh = new THREE.Mesh(
            new THREE.PlaneGeometry(width - 32, height * 0.5),
            new THREE.MeshBasicMaterial({ map: new THREE.CanvasTexture(bodyCanvas), transparent: true })
        );
        this.bodyTextureMesh.position.set(0, 0, 2);
        this.add(this.bodyTextureMesh);
    }
    
    open() {
        this.isOpen = true;
        this.visible = true;
        this.targetScale = 1;
    }
    
    close() {
        this.isOpen = false;
        this.targetScale = 0;
        setTimeout(() => { this.visible = false; }, 400);
        this.emit && this.emit('close');
    }
    
    update(dt) {
        if (this.targetScale !== undefined) {
            const curr = this.scale.x;
            const newScale = curr + (this.targetScale - curr) * 0.12;
            this.scale.setScalar(newScale);
        }
        this.confirmBtn.update(dt);
        this.cancelBtn.update(dt);
    }
}
```

## UI Layout Manager

```javascript
class UILayout3D {
    constructor(origin, options = {}) {
        this.origin = origin; // THREE.Vector3
        this.options = {
            direction: 'vertical', // 'vertical' | 'horizontal' | 'grid'
            spacing: 10,
            columns: 3, // สำหรับ grid
            ...options,
        };
        this.components = [];
    }
    
    add(component) {
        this.components.push(component);
        this.layout();
        return this;
    }
    
    layout() {
        const { direction, spacing, columns } = this.options;
        let x = this.origin.x;
        let y = this.origin.y;
        
        this.components.forEach((comp, i) => {
            if (direction === 'vertical') {
                comp.position.set(x, y, this.origin.z);
                y -= (comp.options?.height || 50) + spacing;
            } else if (direction === 'horizontal') {
                comp.position.set(x, y, this.origin.z);
                x += (comp.options?.width || 200) + spacing;
            } else if (direction === 'grid') {
                const col = i % columns;
                const row = Math.floor(i / columns);
                comp.position.set(
                    x + col * ((comp.options?.width || 200) + spacing),
                    y - row * ((comp.options?.height || 50) + spacing),
                    this.origin.z
                );
            }
        });
    }
    
    update(dt) {
        this.components.forEach(c => c.update && c.update(dt));
    }
}
```

## Billboarding UI vs World-Space UI

```javascript
// Billboarding: UI หันหน้าหา camera เสมอ
function updateBillboard(uiGroup, camera) {
    uiGroup.quaternion.copy(camera.quaternion);
}

// World-space: UI อยู่ใน world และหมุนตาม scene
// ไม่ต้องทำอะไรพิเศษ

// Partial billboard: หมุนแค่แกน Y (เหมาะกับ name labels)
function updateYBillboard(uiGroup, camera) {
    const target = new THREE.Vector3();
    target.copy(camera.position);
    target.y = uiGroup.position.y; // lock Y
    uiGroup.lookAt(target);
}

// ใน animate loop:
function animate() {
    requestAnimationFrame(animate);
    
    // อัพเดต billboarding UI
    billboardComponents.forEach(c => updateBillboard(c, camera));
    
    // อัพเดต world-space UI
    worldSpaceComponents.forEach(c => c.update(delta));
}
```

## Demo หลัก: Complete 3D UI Component Library

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>3D UI Component Library</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #0a0a18; overflow: hidden; font-family: Arial, sans-serif; }
        #info {
            position: fixed;
            top: 16px; left: 16px;
            color: rgba(255,255,255,0.6);
            font-size: 13px;
            z-index: 10;
            pointer-events: none;
        }
    </style>
</head>
<body>
    <div id="info">
        🖱️ Hover บน buttons | คลิกเพื่อ interact | ลากเพื่อหมุน
    </div>
    <canvas id="canvas"></canvas>

    <script type="importmap">
    {
        "imports": {
            "three": "https://cdn.jsdelivr.net/npm/three@0.165.0/build/three.module.js",
            "three/examples/jsm/": "https://cdn.jsdelivr.net/npm/three@0.165.0/examples/jsm/"
        }
    }
    </script>
    <script type="module">
        import * as THREE from 'three';
        import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

        const W = window.innerWidth, H = window.innerHeight;
        const canvas = document.getElementById('canvas');
        
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x0a0a18);
        
        const camera = new THREE.PerspectiveCamera(60, W / H, 1, 5000);
        camera.position.set(0, 100, 700);
        
        const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
        renderer.setSize(W, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        renderer.shadowMap.enabled = true;
        
        const controls = new OrbitControls(camera, canvas);
        controls.enableDamping = true;
        controls.target.set(0, 50, 0);
        
        // Lighting
        scene.add(new THREE.AmbientLight(0x334466, 1));
        const dirLight = new THREE.DirectionalLight(0x8899ff, 2);
        dirLight.position.set(200, 400, 200);
        scene.add(dirLight);
        const pointLight = new THREE.PointLight(0xff6644, 1.5, 1000);
        pointLight.position.set(-200, 200, 100);
        scene.add(pointLight);
        
        // Raycaster
        const raycaster = new THREE.Raycaster();
        const mouse = new THREE.Vector2();
        const interactables = [];
        let hoveredObj = null;
        
        // ===== Utility: Canvas Text Texture =====
        function makeTextTexture(text, options = {}) {
            const opts = {
                width: 512, height: 64,
                font: 'bold 32px Arial',
                color: '#ffffff',
                bg: null,
                align: 'center',
                ...options,
            };
            const canvas = document.createElement('canvas');
            canvas.width = opts.width;
            canvas.height = opts.height;
            const ctx = canvas.getContext('2d');
            if (opts.bg) {
                ctx.fillStyle = opts.bg;
                ctx.fillRect(0, 0, opts.width, opts.height);
            }
            ctx.font = opts.font;
            ctx.fillStyle = opts.color;
            ctx.textAlign = opts.align;
            ctx.textBaseline = 'middle';
            ctx.fillText(text, opts.align === 'center' ? opts.width / 2 : 16, opts.height / 2);
            return new THREE.CanvasTexture(canvas);
        }
        
        // ===== 3D Button =====
        function createButton(label, x, y, z, color = 0x2233aa) {
            const group = new THREE.Group();
            group.position.set(x, y, z);
            group.userData = { type: 'button', label, baseColor: color, hoverColor: color + 0x112233 };
            
            const geo = new THREE.BoxGeometry(150, 44, 8);
            const mat = new THREE.MeshStandardMaterial({ color, roughness: 0.3, metalness: 0.2 });
            const mesh = new THREE.Mesh(geo, mat);
            group.add(mesh);
            group.userData.mesh = mesh;
            group.userData.mat = mat;
            
            const labelTex = makeTextTexture(label, { color: '#ddeeff' });
            const labelMesh = new THREE.Mesh(
                new THREE.PlaneGeometry(140, 36),
                new THREE.MeshBasicMaterial({ map: labelTex, transparent: true })
            );
            labelMesh.position.z = 5;
            group.add(labelMesh);
            
            // Edge glow
            const edges = new THREE.EdgesGeometry(geo);
            const edgeMat = new THREE.LineBasicMaterial({ color: 0x4455ff, transparent: true, opacity: 0.5 });
            group.add(new THREE.LineSegments(edges, edgeMat));
            
            scene.add(group);
            interactables.push(group);
            return group;
        }
        
        // ===== 3D Toggle =====
        function createToggle(x, y, z, initialOn = false) {
            const group = new THREE.Group();
            group.position.set(x, y, z);
            group.userData = { type: 'toggle', on: initialOn, animT: initialOn ? 1 : 0 };
            
            const trackGeo = new THREE.BoxGeometry(80, 36, 10);
            const trackMat = new THREE.MeshStandardMaterial({
                color: initialOn ? 0x4466ff : 0x333355,
                roughness: 0.4,
            });
            group.add(new THREE.Mesh(trackGeo, trackMat));
            group.userData.trackMat = trackMat;
            
            const thumbGeo = new THREE.SphereGeometry(14, 20, 20);
            const thumbMat = new THREE.MeshStandardMaterial({ color: 0xffffff, roughness: 0.1, metalness: 0.7 });
            const thumb = new THREE.Mesh(thumbGeo, thumbMat);
            thumb.position.set(initialOn ? 22 : -22, 0, 8);
            group.add(thumb);
            group.userData.thumb = thumb;
            
            scene.add(group);
            interactables.push(group);
            return group;
        }
        
        // ===== 3D Progress Bar =====
        function createProgressBar(value, x, y, z, color = 0x4488ff) {
            const group = new THREE.Group();
            group.position.set(x, y, z);
            group.userData = { type: 'progress', value };
            
            const W = 200, H = 20;
            // Track
            const trackMat = new THREE.MeshStandardMaterial({ color: 0x1a1a3a });
            group.add(new THREE.Mesh(new THREE.BoxGeometry(W, H, 6), trackMat));
            
            // Fill
            const fillMat = new THREE.MeshStandardMaterial({
                color,
                emissive: new THREE.Color(color).multiplyScalar(0.4),
            });
            const fill = new THREE.Mesh(new THREE.BoxGeometry(W * value, H - 4, 8), fillMat);
            fill.position.x = -W / 2 + (W * value) / 2;
            group.add(fill);
            group.userData.fill = fill;
            group.userData.fillMat = fillMat;
            
            // Label
            const tex = makeTextTexture(`${Math.round(value * 100)}%`, {
                width: 128, height: 48, font: 'bold 26px Arial', color: '#ffffff'
            });
            const label = new THREE.Mesh(
                new THREE.PlaneGeometry(50, 18),
                new THREE.MeshBasicMaterial({ map: tex, transparent: true })
            );
            label.position.set(W / 2 + 35, 0, 4);
            group.add(label);
            group.userData.label = label;
            
            scene.add(group);
            return group;
        }
        
        // ===== 3D Panel =====
        function createPanel(title, x, y, z) {
            const group = new THREE.Group();
            group.position.set(x, y, z);
            
            const PW = 280, PH = 200;
            
            // Background
            const bgMat = new THREE.MeshStandardMaterial({
                color: 0x111128,
                roughness: 0.5,
                transparent: true,
                opacity: 0.92,
            });
            group.add(new THREE.Mesh(new THREE.PlaneGeometry(PW, PH), bgMat));
            
            // Border
            const borderEdges = new THREE.EdgesGeometry(new THREE.BoxGeometry(PW, PH, 1));
            group.add(new THREE.LineSegments(borderEdges, new THREE.LineBasicMaterial({ color: 0x3344aa })));
            
            // Title bar
            const titleMat = new THREE.MeshStandardMaterial({ color: 0x1a1a5a });
            const titleBar = new THREE.Mesh(new THREE.PlaneGeometry(PW, 38), titleMat);
            titleBar.position.set(0, PH / 2 - 19, 0.5);
            group.add(titleBar);
            
            // Title text
            const titleTex = makeTextTexture(title, {
                width: 512, height: 64, color: '#aabbff', align: 'left', font: 'bold 26px Arial'
            });
            const titleLabel = new THREE.Mesh(
                new THREE.PlaneGeometry(PW - 40, 30),
                new THREE.MeshBasicMaterial({ map: titleTex, transparent: true })
            );
            titleLabel.position.set(-18, PH / 2 - 19, 1);
            group.add(titleLabel);
            
            scene.add(group);
            return group;
        }
        
        // ===== Create UI Scene =====
        
        // Main panel
        const mainPanel = createPanel('⚙️ Settings Panel', 0, 80, 0);
        
        // Buttons in panel
        const btn1 = createButton('▶ Start', -80, 30, 5);
        btn1.userData.onClick = () => console.log('Start clicked!');
        
        const btn2 = createButton('⏸ Pause', 80, 30, 5);
        const btn3 = createButton('🔄 Reset', 0, -20, 5);
        
        // Toggles
        const toggle1 = createToggle(-60, -70, 5, true);
        const toggle2 = createToggle(60, -70, 5, false);
        
        // Progress bars (outside panel)
        const pb1 = createProgressBar(0.72, -350, 120, 0, 0x4488ff);
        const pb2 = createProgressBar(0.45, -350, 80, 0, 0x44ff88);
        const pb3 = createProgressBar(0.88, -350, 40, 0, 0xff8844);
        
        // Label textures for progress bars
        const labelTex1 = makeTextTexture('Health', { width: 256, height: 48, color: '#88aaff', align: 'left' });
        const labelTex2 = makeTextTexture('Mana', { width: 256, height: 48, color: '#88ff88', align: 'left' });
        const labelTex3 = makeTextTexture('XP', { width: 256, height: 48, color: '#ff8844', align: 'left' });
        
        [[pb1, labelTex1], [pb2, labelTex2], [pb3, labelTex3]].forEach(([pb, tex]) => {
            const lm = new THREE.Mesh(
                new THREE.PlaneGeometry(80, 18),
                new THREE.MeshBasicMaterial({ map: tex, transparent: true })
            );
            lm.position.set(-150, 0, 1);
            pb.add(lm);
        });
        
        // Animate progress bars
        let t = 0;
        
        // Mouse events
        window.addEventListener('mousemove', (e) => {
            mouse.x = (e.clientX / W) * 2 - 1;
            mouse.y = -(e.clientY / H) * 2 + 1;
        });
        
        window.addEventListener('click', () => {
            if (hoveredObj) {
                const ud = hoveredObj.userData;
                if (ud.type === 'button') {
                    ud.onClick && ud.onClick();
                    // Flash animation
                    ud.mat.color.setHex(0xffffff);
                    setTimeout(() => ud.mat.color.setHex(ud.baseColor), 150);
                } else if (ud.type === 'toggle') {
                    ud.on = !ud.on;
                }
            }
        });
        
        function animate() {
            requestAnimationFrame(animate);
            t = performance.now() * 0.001;
            
            // Raycast
            raycaster.setFromCamera(mouse, camera);
            const allMeshes = [];
            interactables.forEach(group => {
                group.traverse(child => {
                    if (child.isMesh) allMeshes.push(child);
                });
            });
            
            const hits = raycaster.intersectObjects(allMeshes);
            const newHovered = hits.length > 0 ? hits[0].object.parent : null;
            
            if (newHovered !== hoveredObj) {
                if (hoveredObj && hoveredObj.userData.mat) {
                    hoveredObj.userData.mat.color.setHex(hoveredObj.userData.baseColor);
                    hoveredObj.userData.mat.emissive?.setHex(0x000000);
                }
                hoveredObj = newHovered;
                if (hoveredObj && hoveredObj.userData.type === 'button') {
                    hoveredObj.userData.mat.color.setHex(hoveredObj.userData.hoverColor);
                    document.body.style.cursor = 'pointer';
                } else {
                    document.body.style.cursor = 'default';
                }
            }
            
            // Toggle animation
            interactables.forEach(group => {
                if (group.userData.type === 'toggle') {
                    const ud = group.userData;
                    ud.animT += (ud.on ? 1 : 0 - ud.animT) * 0.15;
                    ud.thumb.position.x = -22 + 44 * ud.animT;
                    const col = new THREE.Color(0x333355).lerp(new THREE.Color(0x4466ff), ud.animT);
                    ud.trackMat.color.copy(col);
                }
            });
            
            // Animate progress bars
            pb1.userData.fill.scale.x = 0.72 + Math.sin(t * 0.5) * 0.1;
            pb2.userData.fill.scale.x = 0.45 + Math.sin(t * 0.7 + 1) * 0.15;
            pb3.userData.fill.scale.x = 0.88 + Math.sin(t * 0.3 + 2) * 0.05;
            
            controls.update();
            renderer.render(scene, camera);
        }
        animate();
        
        window.addEventListener('resize', () => {
            const w = window.innerWidth, h = window.innerHeight;
            camera.aspect = w / h;
            camera.updateProjectionMatrix();
            renderer.setSize(w, h);
        });
    </script>
</body>
</html>
```

## Game HUD ด้วย 3D UI Components

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Game HUD - 3D UI</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; }
        canvas { display: block; }
    </style>
</head>
<body>
    <canvas id="canvas"></canvas>
    <script type="importmap">
    {
        "imports": {
            "three": "https://cdn.jsdelivr.net/npm/three@0.165.0/build/three.module.js",
            "three/examples/jsm/": "https://cdn.jsdelivr.net/npm/three@0.165.0/examples/jsm/"
        }
    }
    </script>
    <script type="module">
        import * as THREE from 'three';
        
        const W = window.innerWidth, H = window.innerHeight;
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x0a1a0a);
        
        // HUD camera (orthographic)
        const hudCamera = new THREE.OrthographicCamera(-W/2, W/2, H/2, -H/2, -100, 100);
        hudCamera.position.z = 50;
        
        // Main camera
        const mainCamera = new THREE.PerspectiveCamera(75, W / H, 0.1, 1000);
        mainCamera.position.set(0, 5, 20);
        
        const renderer = new THREE.WebGLRenderer({ canvas: document.getElementById('canvas'), antialias: true });
        renderer.setSize(W, H);
        renderer.autoClear = false;
        
        // Lighting
        scene.add(new THREE.AmbientLight(0x224422, 1.5));
        scene.add(new THREE.DirectionalLight(0xffffff, 1));
        
        // Game scene
        const terrain = new THREE.Mesh(
            new THREE.PlaneGeometry(200, 200, 20, 20),
            new THREE.MeshStandardMaterial({ color: 0x1a3a1a, roughness: 0.9 })
        );
        terrain.rotation.x = -Math.PI / 2;
        scene.add(terrain);
        
        // Player character
        const player = new THREE.Group();
        const body = new THREE.Mesh(new THREE.CapsuleGeometry(0.5, 1.5, 8, 16), new THREE.MeshStandardMaterial({ color: 0x4488ff }));
        body.position.y = 1.25;
        player.add(body);
        scene.add(player);
        
        // HUD scene
        const hudScene = new THREE.Scene();
        
        function makeTexture(drawFn, w = 512, h = 128) {
            const canvas = document.createElement('canvas');
            canvas.width = w; canvas.height = h;
            drawFn(canvas.getContext('2d'), w, h);
            return new THREE.CanvasTexture(canvas);
        }
        
        function makePlane(tex, pw, ph, x, y) {
            const mesh = new THREE.Mesh(
                new THREE.PlaneGeometry(pw, ph),
                new THREE.MeshBasicMaterial({ map: tex, transparent: true, depthTest: false })
            );
            mesh.position.set(x, y, 0);
            hudScene.add(mesh);
            return mesh;
        }
        
        // Health bar
        let health = 80, maxHealth = 100;
        let mana = 60, maxMana = 100;
        let ammo = 23, maxAmmo = 30;
        let score = 4250;
        
        // Create HUD elements
        const healthBarMesh = makePlane(
            makeTexture((ctx, w, h) => {
                // Background
                ctx.fillStyle = '#1a0000';
                ctx.roundRect(0, 20, w, h - 40, 8);
                ctx.fill();
                // Fill
                ctx.fillStyle = '#ff2244';
                ctx.roundRect(2, 22, (w - 4) * (health / maxHealth), h - 44, 6);
                ctx.fill();
                // Label
                ctx.font = 'bold 24px Arial';
                ctx.fillStyle = '#ffaaaa';
                ctx.textBaseline = 'middle';
                ctx.fillText(`❤️ ${health}/${maxHealth}`, 16, h / 2);
            }),
            300, 60, -W / 2 + 165, H / 2 - 50
        );
        
        const manaBarMesh = makePlane(
            makeTexture((ctx, w, h) => {
                ctx.fillStyle = '#00001a';
                ctx.roundRect(0, 20, w, h - 40, 8);
                ctx.fill();
                ctx.fillStyle = '#2244ff';
                ctx.roundRect(2, 22, (w - 4) * (mana / maxMana), h - 44, 6);
                ctx.fill();
                ctx.font = 'bold 24px Arial';
                ctx.fillStyle = '#aaaaff';
                ctx.textBaseline = 'middle';
                ctx.fillText(`🔷 ${mana}/${maxMana}`, 16, h / 2);
            }),
            300, 60, -W / 2 + 165, H / 2 - 120
        );
        
        const ammoMesh = makePlane(
            makeTexture((ctx, w, h) => {
                ctx.font = 'bold 28px Arial';
                ctx.fillStyle = '#ffdd44';
                ctx.textAlign = 'right';
                ctx.textBaseline = 'middle';
                ctx.fillText(`🔫 ${ammo} / ${maxAmmo}`, w - 10, h / 2);
            }),
            220, 60, W / 2 - 120, H / 2 - 50
        );
        
        const scoreMesh = makePlane(
            makeTexture((ctx, w, h) => {
                ctx.font = 'bold 30px Arial';
                ctx.fillStyle = '#ffcc00';
                ctx.textAlign = 'right';
                ctx.textBaseline = 'middle';
                ctx.fillText(`⭐ ${score.toLocaleString()}`, w - 10, h / 2);
            }),
            240, 60, W / 2 - 130, H / 2 - 110
        );
        
        // Crosshair
        makePlane(
            makeTexture((ctx, w, h) => {
                ctx.strokeStyle = 'rgba(255,255,255,0.8)';
                ctx.lineWidth = 2;
                // Cross
                ctx.beginPath();
                ctx.moveTo(w/2 - 20, h/2); ctx.lineTo(w/2 - 8, h/2);
                ctx.moveTo(w/2 + 8, h/2); ctx.lineTo(w/2 + 20, h/2);
                ctx.moveTo(w/2, h/2 - 20); ctx.lineTo(w/2, h/2 - 8);
                ctx.moveTo(w/2, h/2 + 8); ctx.lineTo(w/2, h/2 + 20);
                ctx.stroke();
                // Center dot
                ctx.fillStyle = 'rgba(255,255,255,0.8)';
                ctx.beginPath();
                ctx.arc(w/2, h/2, 2, 0, Math.PI * 2);
                ctx.fill();
            }, 128, 128),
            64, 64, 0, 0
        );
        
        let t = 0;
        function animate() {
            requestAnimationFrame(animate);
            t = performance.now() * 0.001;
            
            // Animate game
            player.position.x = Math.sin(t * 0.5) * 5;
            mainCamera.lookAt(player.position);
            
            // Simulate changing values
            health = 80 + Math.sin(t * 0.3) * 15;
            mana = 60 + Math.sin(t * 0.5 + 1) * 20;
            
            renderer.clear();
            renderer.render(scene, mainCamera);
            renderer.clearDepth();
            renderer.render(hudScene, hudCamera);
        }
        animate();
        
        window.addEventListener('resize', () => {
            const w = window.innerWidth, h = window.innerHeight;
            mainCamera.aspect = w / h;
            mainCamera.updateProjectionMatrix();
            hudCamera.left = -w/2; hudCamera.right = w/2;
            hudCamera.top = h/2; hudCamera.bottom = -h/2;
            hudCamera.updateProjectionMatrix();
            renderer.setSize(w, h);
        });
    </script>
</body>
</html>
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: 3D Dropdown Menu
สร้าง dropdown menu 3D ที่เมื่อคลิกจะเปิดรายการ options ออกมาด้วย animation สวยงาม

### แบบฝึกหัดที่ 2: Color Picker 3D
สร้าง color picker ที่แสดงใน 3D space ใช้ geometry ของ cylinder หรือ sphere แบ่งเป็น color segments คลิกเพื่อเลือกสี

### แบบฝึกหัดที่ 3: Settings Panel สมบูรณ์
สร้าง settings panel ที่มีทั้ง buttons, toggles, sliders และ progress bars ทำงานร่วมกันได้จริง

### แบบฝึกหัดที่ 4: Inventory Grid
สร้าง inventory system แบบเกม RPG ที่มีช่อง items 4x4 ใน 3D space แต่ละช่องสามารถ hover และ click เพื่อดูรายละเอียด item

## สรุป

การสร้าง 3D UI components ต้องการความเข้าใจใน:
1. **Raycaster** สำหรับการตรวจจับ interaction
2. **Canvas Texture** สำหรับการแสดง text และ graphics
3. **Animation** สำหรับ hover, click, transition states
4. **Layout management** สำหรับจัดเรียง components
5. **Billboarding** สำหรับ UI ที่ต้องหันหน้าหา camera

ในบทถัดไปเราจะเรียนรู้ระบบ Labels, Annotations และ HUD ที่ซับซ้อนขึ้น
