# Part 100: 3D Progress Indicators & Status

## บทนำ

ยินดีต้อนรับสู่ Part 100! ในบทนี้เราจะสร้าง progress indicators และ status systems ที่ใช้ใน games และ applications แบบ 3D ตั้งแต่ health bar ง่ายๆ ไปถึง achievement system ที่สมบูรณ์แบบ

## 3D Loading Ring/Spinner

```javascript
class LoadingRing3D {
    constructor(options = {}) {
        this.options = {
            radius: 3,
            tubeRadius: 0.3,
            color: new THREE.Color(0x4488ff),
            speed: 2,
            segments: 3,
            ...options,
        };
        
        this.group = new THREE.Group();
        this.rings = [];
        this.build();
    }
    
    build() {
        const { radius, tubeRadius, color, segments } = this.options;
        
        for (let i = 0; i < segments; i++) {
            // Arc torus (ครึ่งวง)
            const arcGeo = new THREE.TorusGeometry(
                radius - i * (tubeRadius * 2.5),
                tubeRadius,
                8,
                32,
                Math.PI * 1.2 // 216 degrees arc
            );
            
            const col = color.clone().offsetHSL(i * 0.05, 0, 0);
            const mat = new THREE.MeshStandardMaterial({
                color: col,
                emissive: col.clone().multiplyScalar(0.5),
                roughness: 0.2,
                metalness: 0.8,
            });
            
            const ring = new THREE.Mesh(arcGeo, mat);
            ring.userData = {
                rotationSpeed: this.options.speed * (1 + i * 0.3),
                rotationAxis: i % 3, // 0=X, 1=Y, 2=Z
            };
            
            this.group.add(ring);
            this.rings.push(ring);
        }
        
        // Center glow sphere
        const coreGeo = new THREE.SphereGeometry(radius * 0.3, 16, 16);
        const coreMat = new THREE.MeshStandardMaterial({
            color,
            emissive: color.clone().multiplyScalar(1),
            roughness: 0,
        });
        this.coreMesh = new THREE.Mesh(coreGeo, coreMat);
        this.group.add(this.coreMesh);
    }
    
    update(dt) {
        this.rings.forEach(ring => {
            switch (ring.userData.rotationAxis) {
                case 0: ring.rotation.x += ring.userData.rotationSpeed * dt; break;
                case 1: ring.rotation.y += ring.userData.rotationSpeed * dt; break;
                case 2: ring.rotation.z += ring.userData.rotationSpeed * dt; break;
            }
        });
        
        // Pulse core
        const t = performance.now() * 0.003;
        this.coreMesh.scale.setScalar(1 + Math.sin(t * 2) * 0.1);
    }
    
    addToScene(scene) {
        scene.add(this.group);
        return this;
    }
}
```

## 3D Health Bar ใน World Space

```javascript
class HealthBar3D {
    constructor(options = {}) {
        this.options = {
            width: 4,
            height: 0.3,
            maxValue: 100,
            value: 100,
            position: new THREE.Vector3(0, 3, 0),
            trackColor: 0x220000,
            fillColor: 0xff2222,
            borderColor: 0x444444,
            showText: false,
            ...options,
        };
        
        this.group = new THREE.Group();
        this._value = this.options.value;
        this._displayValue = this._value;
        this.build();
    }
    
    build() {
        const { width, height, trackColor, fillColor, borderColor } = this.options;
        
        // Track (background)
        const trackGeo = new THREE.BoxGeometry(width, height, 0.1);
        const trackMat = new THREE.MeshStandardMaterial({ color: trackColor, roughness: 0.5 });
        this.group.add(new THREE.Mesh(trackGeo, trackMat));
        
        // Fill (health)
        this.fillMat = new THREE.MeshStandardMaterial({
            color: fillColor,
            roughness: 0.3,
            emissive: new THREE.Color(fillColor).multiplyScalar(0.2),
        });
        const fillGeo = new THREE.BoxGeometry(width, height - 0.04, 0.12);
        this.fillMesh = new THREE.Mesh(fillGeo, this.fillMat);
        this.fillMesh.position.x = 0;
        this.group.add(this.fillMesh);
        this.fillMesh.userData.width = width;
        
        // Border
        const borderGeo = new THREE.EdgesGeometry(trackGeo);
        const borderMat = new THREE.LineBasicMaterial({ color: borderColor, transparent: true, opacity: 0.7 });
        this.group.add(new THREE.LineSegments(borderGeo, borderMat));
        
        this.group.position.copy(this.options.position);
        this.updateDisplay(this._value);
    }
    
    setValue(value) {
        this._value = Math.max(0, Math.min(this.options.maxValue, value));
        this.updateColorByPercent(this._value / this.options.maxValue);
    }
    
    updateColorByPercent(percent) {
        // High HP: green, medium: yellow, low: red
        let color;
        if (percent > 0.6) {
            color = new THREE.Color(0x22ff44);
        } else if (percent > 0.3) {
            color = new THREE.Color().lerpColors(
                new THREE.Color(0xff8800),
                new THREE.Color(0x22ff44),
                (percent - 0.3) / 0.3
            );
        } else {
            color = new THREE.Color().lerpColors(
                new THREE.Color(0xff2222),
                new THREE.Color(0xff8800),
                percent / 0.3
            );
        }
        this.fillMat.color.copy(color);
        this.fillMat.emissive.copy(color).multiplyScalar(0.1);
    }
    
    updateDisplay(value) {
        const percent = value / this.options.maxValue;
        this.fillMesh.scale.x = Math.max(0.001, percent);
        const w = this.fillMesh.userData.width;
        this.fillMesh.position.x = -w / 2 + (w * percent) / 2;
    }
    
    update(dt) {
        // Smooth animation
        this._displayValue += (this._value - this._displayValue) * 0.1;
        this.updateDisplay(this._displayValue);
        
        // Flicker when low
        if (this._value / this.options.maxValue < 0.2) {
            this.fillMat.emissiveIntensity = 0.3 + Math.sin(performance.now() * 0.01) * 0.3;
        }
    }
}
```

## Radial Progress (Arc Segment)

```javascript
class RadialProgress3D {
    constructor(options = {}) {
        this.options = {
            radius: 2,
            tubeRadius: 0.25,
            color: new THREE.Color(0x4488ff),
            bgColor: new THREE.Color(0x111122),
            value: 0,
            ...options,
        };
        
        this.group = new THREE.Group();
        this._value = 0;
        this.targetValue = options.value || 0;
        this.build();
    }
    
    build() {
        const { radius, tubeRadius, bgColor } = this.options;
        
        // Background ring
        const bgGeo = new THREE.TorusGeometry(radius, tubeRadius, 8, 64);
        const bgMat = new THREE.MeshStandardMaterial({ color: bgColor, roughness: 0.5 });
        this.group.add(new THREE.Mesh(bgGeo, bgMat));
        
        // Progress arc (will be updated)
        this.createProgressArc(0.01);
    }
    
    createProgressArc(progress) {
        if (this.arcMesh) {
            this.group.remove(this.arcMesh);
        }
        
        const { radius, tubeRadius, color } = this.options;
        const arcAngle = progress * Math.PI * 2;
        
        if (arcAngle < 0.01) return;
        
        const arcGeo = new THREE.TorusGeometry(radius, tubeRadius + 0.02, 8, Math.max(4, Math.floor(64 * progress)), arcAngle);
        const mat = new THREE.MeshStandardMaterial({
            color,
            emissive: color.clone().multiplyScalar(0.4),
            roughness: 0.2,
            metalness: 0.6,
        });
        
        this.arcMesh = new THREE.Mesh(arcGeo, mat);
        this.arcMesh.rotation.z = -Math.PI / 2; // Start from top
        this.group.add(this.arcMesh);
    }
    
    setValue(value) {
        this.targetValue = Math.max(0, Math.min(1, value));
    }
    
    update(dt) {
        const prevValue = this._value;
        this._value += (this.targetValue - this._value) * 0.08;
        
        // Recreate arc only when value changes significantly
        if (Math.abs(this._value - prevValue) > 0.001) {
            this.createProgressArc(this._value);
        }
    }
}
```

## 3D Timer Display

```javascript
class Timer3D {
    constructor(scene, position, options = {}) {
        this.options = {
            totalTime: 60,
            countDown: true,
            color: '#ffffff',
            warningColor: '#ff8800',
            criticalColor: '#ff2222',
            ...options,
        };
        
        this.elapsed = 0;
        this.isRunning = false;
        this.group = new THREE.Group();
        this.group.position.copy(position);
        
        this.createDisplay();
        scene.add(this.group);
    }
    
    createDisplay() {
        // Create canvas texture for digits
        this.canvas = document.createElement('canvas');
        this.canvas.width = 256;
        this.canvas.height = 128;
        this.texture = new THREE.CanvasTexture(this.canvas);
        
        const geo = new THREE.PlaneGeometry(3, 1.5);
        const mat = new THREE.MeshBasicMaterial({
            map: this.texture,
            transparent: true,
        });
        this.displayMesh = new THREE.Mesh(geo, mat);
        this.group.add(this.displayMesh);
        
        this.renderDigits();
    }
    
    renderDigits() {
        const ctx = this.canvas.getContext('2d');
        ctx.clearRect(0, 0, 256, 128);
        
        const remaining = this.options.countDown
            ? this.options.totalTime - this.elapsed
            : this.elapsed;
        
        const isWarning = remaining < this.options.totalTime * 0.3;
        const isCritical = remaining < this.options.totalTime * 0.1;
        
        const color = isCritical ? this.options.criticalColor
            : isWarning ? this.options.warningColor
            : this.options.color;
        
        const minutes = Math.floor(Math.abs(remaining) / 60);
        const seconds = Math.floor(Math.abs(remaining) % 60);
        const ms = Math.floor((Math.abs(remaining) % 1) * 100);
        
        ctx.font = 'bold 64px "Courier New"';
        ctx.fillStyle = color;
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';
        ctx.shadowColor = color;
        ctx.shadowBlur = isCritical ? 20 : 8;
        
        const timeStr = `${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
        ctx.fillText(timeStr, 128, 64);
        
        this.texture.needsUpdate = true;
    }
    
    start() { this.isRunning = true; }
    pause() { this.isRunning = false; }
    reset() { this.elapsed = 0; this.isRunning = false; this.renderDigits(); }
    
    update(dt) {
        if (!this.isRunning) return;
        this.elapsed += dt;
        if (this.elapsed > this.options.totalTime) this.elapsed = this.options.totalTime;
        this.renderDigits();
    }
}
```

## 3D Notification Popup

```javascript
class Notification3D {
    constructor(scene, camera) {
        this.scene = scene;
        this.camera = camera;
        this.notifications = [];
        this.maxNotifications = 5;
    }
    
    show(message, type = 'info', duration = 3000) {
        const colors = {
            info: { bg: 0x112244, border: 0x4488ff, text: '#88aaff', icon: 'ℹ️' },
            success: { bg: 0x112211, border: 0x44ff44, text: '#88ff88', icon: '✅' },
            warning: { bg: 0x221100, border: 0xff8800, text: '#ffaa44', icon: '⚠️' },
            error: { bg: 0x220011, border: 0xff2244, text: '#ff8899', icon: '❌' },
        };
        
        const style = colors[type] || colors.info;
        
        // Create canvas texture
        const canvas = document.createElement('canvas');
        canvas.width = 400;
        canvas.height = 80;
        const ctx = canvas.getContext('2d');
        
        // Background
        ctx.fillStyle = `rgba(${style.bg >> 16 & 0xff}, ${style.bg >> 8 & 0xff}, ${style.bg & 0xff}, 0.9)`;
        ctx.roundRect(0, 0, 400, 80, 10);
        ctx.fill();
        
        // Border
        ctx.strokeStyle = style.text;
        ctx.lineWidth = 2;
        ctx.roundRect(1, 1, 398, 78, 10);
        ctx.stroke();
        
        // Accent bar (left side)
        ctx.fillStyle = style.text;
        ctx.fillRect(0, 10, 4, 60);
        
        // Text
        ctx.font = 'bold 26px Arial';
        ctx.fillStyle = style.text;
        ctx.textBaseline = 'middle';
        ctx.fillText(`${style.icon} ${message}`, 20, 40);
        
        const texture = new THREE.CanvasTexture(canvas);
        const geo = new THREE.PlaneGeometry(5, 1);
        const mat = new THREE.MeshBasicMaterial({
            map: texture,
            transparent: true,
            depthTest: false,
        });
        
        const notif = new THREE.Mesh(geo, mat);
        notif.userData = {
            duration,
            elapsed: 0,
            startPosition: null,
            targetY: null,
            state: 'entering',
        };
        
        this.scene.add(notif);
        this.notifications.push(notif);
        
        // Remove oldest if too many
        if (this.notifications.length > this.maxNotifications) {
            const oldest = this.notifications.shift();
            this.scene.remove(oldest);
        }
        
        this.repositionAll();
        return notif;
    }
    
    repositionAll() {
        const startY = -3;
        this.notifications.forEach((notif, i) => {
            notif.userData.targetY = startY - i * 1.3;
        });
    }
    
    update(dt, camera) {
        this.notifications = this.notifications.filter(notif => {
            notif.userData.elapsed += dt;
            
            // Set initial position relative to camera
            const camDir = new THREE.Vector3();
            camera.getWorldDirection(camDir);
            const forward = camDir.clone().multiplyScalar(6);
            
            const camRight = new THREE.Vector3();
            camRight.crossVectors(camera.up, camDir).normalize();
            
            const basePos = camera.position.clone().add(forward);
            basePos.addScaledVector(camRight, -3);
            
            if (!notif.userData.startPosition) {
                notif.userData.startPosition = basePos.clone();
                notif.userData.startPosition.y -= 10; // start below
            }
            
            // Target position
            const target = basePos.clone();
            target.y += notif.userData.targetY || 0;
            
            // Smooth movement
            notif.position.lerp(target, 0.1);
            
            // Billboard
            notif.quaternion.copy(camera.quaternion);
            
            // Fade
            const progress = notif.userData.elapsed / notif.userData.duration;
            
            if (progress < 0.1) {
                notif.material.opacity = progress / 0.1;
            } else if (progress > 0.8) {
                notif.material.opacity = 1 - (progress - 0.8) / 0.2;
            } else {
                notif.material.opacity = 1;
            }
            
            if (progress >= 1) {
                this.scene.remove(notif);
                notif.geometry.dispose();
                notif.material.dispose();
                notif.material.map?.dispose();
                return false;
            }
            return true;
        });
        
        this.repositionAll();
    }
}
```

## Achievement Badge 3D Pop

```javascript
class AchievementSystem {
    constructor(scene, camera) {
        this.scene = scene;
        this.camera = camera;
        this.queue = [];
        this.isShowing = false;
    }
    
    unlock(data) {
        this.queue.push(data);
        if (!this.isShowing) {
            this.showNext();
        }
    }
    
    showNext() {
        if (this.queue.length === 0) {
            this.isShowing = false;
            return;
        }
        
        this.isShowing = true;
        const data = this.queue.shift();
        this.createBadge(data);
    }
    
    createBadge(data) {
        const group = new THREE.Group();
        
        // Badge background
        const badgeGeo = new THREE.CylinderGeometry(2.5, 2.5, 0.5, 6); // hexagon
        const badgeMat = new THREE.MeshStandardMaterial({
            color: data.color || 0xffaa00,
            roughness: 0.2,
            metalness: 0.8,
            emissive: new THREE.Color(data.color || 0xffaa00).multiplyScalar(0.3),
        });
        group.add(new THREE.Mesh(badgeGeo, badgeMat));
        
        // Star decoration
        for (let i = 0; i < 6; i++) {
            const starGeo = new THREE.OctahedronGeometry(0.3, 0);
            const starMesh = new THREE.Mesh(starGeo, new THREE.MeshStandardMaterial({ color: 0xffffff, emissive: 0xffaa00, emissiveIntensity: 0.5 }));
            const angle = (i / 6) * Math.PI * 2;
            starMesh.position.set(Math.cos(angle) * 2, 0.3, Math.sin(angle) * 2);
            starMesh.rotation.y = angle;
            group.add(starMesh);
        }
        
        // Icon text
        const iconCanvas = document.createElement('canvas');
        iconCanvas.width = 128; iconCanvas.height = 128;
        const ictx = iconCanvas.getContext('2d');
        ictx.font = '72px Arial';
        ictx.textAlign = 'center';
        ictx.textBaseline = 'middle';
        ictx.fillText(data.icon || '🏆', 64, 64);
        
        const iconMesh = new THREE.Mesh(
            new THREE.PlaneGeometry(2.5, 2.5),
            new THREE.MeshBasicMaterial({ map: new THREE.CanvasTexture(iconCanvas), transparent: true })
        );
        iconMesh.position.y = 0.3;
        group.add(iconMesh);
        
        // Title banner
        const titleCanvas = document.createElement('canvas');
        titleCanvas.width = 512; titleCanvas.height = 80;
        const tctx = titleCanvas.getContext('2d');
        tctx.fillStyle = 'rgba(0,0,0,0.7)';
        tctx.roundRect(0, 0, 512, 80, 8);
        tctx.fill();
        tctx.font = 'bold 22px Arial';
        tctx.fillStyle = '#ffdd44';
        tctx.textAlign = 'center';
        tctx.textBaseline = 'middle';
        tctx.fillText('🏆 Achievement Unlocked!', 256, 22);
        tctx.font = '18px Arial';
        tctx.fillStyle = 'white';
        tctx.fillText(data.name, 256, 56);
        
        const titleMesh = new THREE.Mesh(
            new THREE.PlaneGeometry(5, 0.8),
            new THREE.MeshBasicMaterial({ map: new THREE.CanvasTexture(titleCanvas), transparent: true, depthTest: false })
        );
        titleMesh.position.set(0, -2.5, 0);
        group.add(titleMesh);
        
        // Position near camera
        const camDir = new THREE.Vector3();
        this.camera.getWorldDirection(camDir);
        group.position.copy(this.camera.position).addScaledVector(camDir, 8);
        
        group.scale.setScalar(0);
        this.scene.add(group);
        
        // Animation sequence
        const startTime = performance.now();
        const duration = 3000;
        
        const animBadge = () => {
            const elapsed = performance.now() - startTime;
            const progress = elapsed / duration;
            
            // Phase 1: Scale in (0-0.15)
            if (progress < 0.15) {
                const s = progress / 0.15;
                const bounce = 1 + Math.sin(s * Math.PI) * 0.3;
                group.scale.setScalar(s * bounce);
            }
            // Phase 2: Hold (0.15-0.75)
            else if (progress < 0.75) {
                group.scale.setScalar(1);
                group.rotation.y += 0.02;
            }
            // Phase 3: Scale out (0.75-1)
            else {
                const s = 1 - (progress - 0.75) / 0.25;
                group.scale.setScalar(s);
                group.position.y += 0.02;
            }
            
            // Billboard
            group.quaternion.copy(this.camera.quaternion);
            
            if (progress < 1) {
                requestAnimationFrame(animBadge);
            } else {
                this.scene.remove(group);
                this.showNext();
            }
        };
        
        requestAnimationFrame(animBadge);
    }
}
```

## Level-Up Particle Burst

```javascript
class LevelUpEffect {
    constructor(scene) {
        this.scene = scene;
        this.particles = [];
    }
    
    trigger(position) {
        // Text burst
        this.createTextParticle(position, '⬆️ LEVEL UP!');
        
        // Particle ring burst
        this.createParticleBurst(position);
        
        // Screen flash (CSS)
        const flash = document.createElement('div');
        flash.style.cssText = `
            position: fixed; top: 0; left: 0;
            width: 100%; height: 100%;
            background: rgba(255, 220, 0, 0.4);
            pointer-events: none;
            z-index: 200;
            animation: levelFlash 0.5s ease-out forwards;
        `;
        
        if (!document.getElementById('level-flash-style')) {
            const style = document.createElement('style');
            style.id = 'level-flash-style';
            style.textContent = `
                @keyframes levelFlash {
                    0% { opacity: 1; }
                    100% { opacity: 0; }
                }
            `;
            document.head.appendChild(style);
        }
        
        document.body.appendChild(flash);
        setTimeout(() => flash.remove(), 600);
    }
    
    createTextParticle(position, text) {
        const canvas = document.createElement('canvas');
        canvas.width = 384; canvas.height = 80;
        const ctx = canvas.getContext('2d');
        
        ctx.font = 'bold 40px Arial';
        ctx.fillStyle = '#ffdd44';
        ctx.strokeStyle = '#000000';
        ctx.lineWidth = 4;
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';
        ctx.strokeText(text, 192, 40);
        ctx.fillText(text, 192, 40);
        
        const mesh = new THREE.Mesh(
            new THREE.PlaneGeometry(4, 0.8),
            new THREE.MeshBasicMaterial({ map: new THREE.CanvasTexture(canvas), transparent: true, depthTest: false })
        );
        mesh.position.copy(position);
        mesh.userData = { vy: 2, elapsed: 0, duration: 2, type: 'text' };
        this.scene.add(mesh);
        this.particles.push(mesh);
    }
    
    createParticleBurst(position) {
        const count = 80;
        const geo = new THREE.BufferGeometry();
        const positions = new Float32Array(count * 3);
        const velocities = [];
        
        for (let i = 0; i < count; i++) {
            positions[i * 3] = position.x;
            positions[i * 3 + 1] = position.y;
            positions[i * 3 + 2] = position.z;
            
            const theta = Math.random() * Math.PI * 2;
            const phi = Math.random() * Math.PI;
            const speed = 3 + Math.random() * 8;
            velocities.push(
                Math.sin(phi) * Math.cos(theta) * speed,
                Math.cos(phi) * speed * 1.5,
                Math.sin(phi) * Math.sin(theta) * speed
            );
        }
        
        geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
        
        const points = new THREE.Points(geo, new THREE.PointsMaterial({
            color: 0xffdd44,
            size: 0.3,
            transparent: true,
            sizeAttenuation: true,
        }));
        
        points.userData = { velocities, elapsed: 0, duration: 1.5, type: 'burst', originalPositions: positions.slice() };
        this.scene.add(points);
        this.particles.push(points);
    }
    
    update(dt, camera) {
        this.particles = this.particles.filter(p => {
            p.userData.elapsed += dt;
            const t = p.userData.elapsed / p.userData.duration;
            
            if (p.userData.type === 'text') {
                p.position.y += p.userData.vy * dt;
                p.material.opacity = t < 0.7 ? 1 : 1 - (t - 0.7) / 0.3;
                p.quaternion.copy(camera.quaternion);
            } else if (p.userData.type === 'burst') {
                const pos = p.geometry.attributes.position.array;
                const vel = p.userData.velocities;
                const orig = p.userData.originalPositions;
                
                for (let i = 0; i < pos.length / 3; i++) {
                    pos[i * 3] = orig[i * 3] + vel[i * 3] * t;
                    pos[i * 3 + 1] = orig[i * 3 + 1] + vel[i * 3 + 1] * t - 5 * t * t; // gravity
                    pos[i * 3 + 2] = orig[i * 3 + 2] + vel[i * 3 + 2] * t;
                }
                p.geometry.attributes.position.needsUpdate = true;
                p.material.opacity = 1 - t;
            }
            
            if (t >= 1) {
                this.scene.remove(p);
                if (p.geometry) p.geometry.dispose();
                if (p.material) {
                    p.material.map?.dispose();
                    p.material.dispose();
                }
                return false;
            }
            return true;
        });
    }
}
```

## Complete Game HUD Demo

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Complete Game HUD</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #0a1a0a; overflow: hidden; font-family: Arial, sans-serif; }
        canvas { display: block; }
        
        #hud {
            position: fixed; top: 0; left: 0;
            width: 100%; height: 100%;
            pointer-events: none;
            z-index: 10;
        }
        
        /* Health/Mana bars */
        .bar-group {
            position: absolute;
            bottom: 24px; left: 20px;
            display: flex;
            flex-direction: column;
            gap: 6px;
        }
        .bar-row {
            display: flex;
            align-items: center;
            gap: 8px;
        }
        .bar-icon { font-size: 16px; }
        .bar-track {
            width: 180px; height: 14px;
            background: rgba(0,0,0,0.5);
            border-radius: 7px;
            overflow: hidden;
            border: 1px solid rgba(255,255,255,0.15);
        }
        .bar-fill {
            height: 100%;
            border-radius: 7px;
            transition: width 0.3s, background-color 0.5s;
        }
        #hp-fill { background: linear-gradient(to right, #cc2222, #ff4444); width: 75%; }
        #mp-fill { background: linear-gradient(to right, #2244cc, #4488ff); width: 60%; }
        #xp-fill { background: linear-gradient(to right, #aa6600, #ffcc00); width: 45%; }
        .bar-text {
            font-size: 10px;
            color: rgba(255,255,255,0.5);
            width: 60px;
        }
        
        /* Skills */
        .skills {
            position: absolute;
            bottom: 24px;
            left: 50%; transform: translateX(-50%);
            display: flex; gap: 6px;
        }
        .skill {
            width: 52px; height: 52px;
            background: rgba(20,20,40,0.8);
            border-radius: 8px;
            border: 1px solid rgba(100,100,200,0.4);
            display: flex; flex-direction: column;
            align-items: center; justify-content: center;
            font-size: 22px;
            position: relative;
        }
        .skill .key {
            position: absolute;
            bottom: 2px; right: 4px;
            font-size: 9px;
            color: rgba(255,255,255,0.4);
        }
        .skill .cooldown-overlay {
            position: absolute; inset: 0;
            background: rgba(0,0,0,0.6);
            border-radius: 8px;
            display: flex; align-items: center; justify-content: center;
            font-size: 16px; font-weight: bold;
            color: white;
        }
        
        /* Score/Level */
        .player-info {
            position: absolute;
            top: 16px; left: 16px;
            color: white;
        }
        .level-badge {
            display: inline-block;
            background: rgba(255,180,0,0.2);
            border: 1px solid #ffaa00;
            border-radius: 6px;
            padding: 3px 10px;
            font-size: 12px;
            color: #ffaa00;
            margin-bottom: 6px;
        }
        .score {
            font-size: 20px;
            font-weight: bold;
            color: #ffdd44;
        }
        
        /* Ammo */
        .ammo-display {
            position: absolute;
            bottom: 30px; right: 20px;
            text-align: right;
        }
        .ammo-main { font-size: 32px; font-weight: bold; color: white; }
        .ammo-reserve { font-size: 14px; color: rgba(255,255,255,0.4); }
        
        /* Kill feed */
        .kill-feed {
            position: absolute;
            top: 16px; right: 16px;
            display: flex;
            flex-direction: column;
            gap: 4px;
        }
        .kill-item {
            background: rgba(0,0,0,0.5);
            border-left: 2px solid #ff4444;
            padding: 3px 8px;
            font-size: 11px;
            color: rgba(255,255,255,0.7);
            animation: killFadeIn 0.3s ease;
        }
        @keyframes killFadeIn {
            from { opacity: 0; transform: translateX(20px); }
            to { opacity: 1; transform: translateX(0); }
        }
    </style>
</head>
<body>
    <canvas id="canvas"></canvas>
    <div id="hud">
        <!-- Stats -->
        <div class="player-info">
            <div class="level-badge">LVL 15</div>
            <div class="score" id="score">12,450</div>
        </div>
        
        <!-- Bars -->
        <div class="bar-group">
            <div class="bar-row">
                <span class="bar-icon">❤️</span>
                <div class="bar-track"><div class="bar-fill" id="hp-fill"></div></div>
                <span class="bar-text" id="hp-text">75/100</span>
            </div>
            <div class="bar-row">
                <span class="bar-icon">💧</span>
                <div class="bar-track"><div class="bar-fill" id="mp-fill"></div></div>
                <span class="bar-text" id="mp-text">60/100</span>
            </div>
            <div class="bar-row">
                <span class="bar-icon">⚡</span>
                <div class="bar-track"><div class="bar-fill" id="xp-fill" style="background: linear-gradient(to right, #aa6600, #ffcc00)"></div></div>
                <span class="bar-text" id="xp-text">450/1000</span>
            </div>
        </div>
        
        <!-- Skills -->
        <div class="skills" id="skills">
            <div class="skill" id="skill-q">🔥<span class="key">Q</span></div>
            <div class="skill" id="skill-w">⚡<span class="key">W</span></div>
            <div class="skill" id="skill-e">🛡️<span class="key">E</span></div>
            <div class="skill" id="skill-r">🌟<span class="key">R</span></div>
        </div>
        
        <!-- Ammo -->
        <div class="ammo-display">
            <div class="ammo-main" id="ammo-main">23</div>
            <div class="ammo-reserve" id="ammo-reserve">/ 90</div>
        </div>
        
        <!-- Kill feed -->
        <div class="kill-feed" id="kill-feed"></div>
    </div>

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
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x0a1a0a);
        scene.fog = new THREE.FogExp2(0x0a1a0a, 0.01);
        
        const camera = new THREE.PerspectiveCamera(75, W / H, 0.1, 500);
        camera.position.set(0, 5, 20);
        
        const renderer = new THREE.WebGLRenderer({ canvas: document.getElementById('canvas'), antialias: true });
        renderer.setSize(W, H);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        renderer.shadowMap.enabled = true;
        
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        
        // Lighting
        scene.add(new THREE.AmbientLight(0x224422, 2));
        const dirLight = new THREE.DirectionalLight(0xffffff, 1.5);
        dirLight.position.set(10, 20, 10);
        dirLight.castShadow = true;
        scene.add(dirLight);
        
        // Ground
        scene.add(Object.assign(new THREE.Mesh(
            new THREE.PlaneGeometry(200, 200),
            new THREE.MeshStandardMaterial({ color: 0x1a3a1a, roughness: 0.9 })
        ), { rotation: { x: -Math.PI / 2 }, receiveShadow: true }));
        
        // Trees and obstacles
        for (let i = 0; i < 30; i++) {
            const trunk = new THREE.Mesh(
                new THREE.CylinderGeometry(0.3, 0.4, 2.5, 8),
                new THREE.MeshStandardMaterial({ color: 0x664422 })
            );
            trunk.position.set((Math.random() - 0.5) * 80, 1.25, (Math.random() - 0.5) * 80);
            trunk.castShadow = true;
            scene.add(trunk);
            
            const leaves = new THREE.Mesh(
                new THREE.ConeGeometry(2.5, 6, 8),
                new THREE.MeshStandardMaterial({ color: 0x225522 })
            );
            leaves.position.copy(trunk.position);
            leaves.position.y += 5.5;
            leaves.castShadow = true;
            scene.add(leaves);
        }
        
        // Player indicator
        const player = new THREE.Mesh(
            new THREE.CapsuleGeometry(0.4, 1.2, 8, 12),
            new THREE.MeshStandardMaterial({ color: 0x4488ff })
        );
        player.position.y = 1;
        player.castShadow = true;
        scene.add(player);
        
        // Game state simulation
        let hp = 75, mp = 60, xp = 450, score = 12450;
        let ammo = 23;
        const cooldowns = { q: 0, w: 0, e: 0, r: 0 };
        const maxCooldowns = { q: 5, w: 8, e: 12, r: 30 };
        const killFeed = document.getElementById('kill-feed');
        
        const enemies = ['Goblin', 'Orc', 'Wolf', 'Bandit', 'Troll'];
        const skills = ['🔥 Fireball', '⚡ Lightning', '🛡️ Shield', '🌟 Ultimate'];
        
        function addKillFeed(killer, victim, skill) {
            const item = document.createElement('div');
            item.className = 'kill-item';
            item.textContent = `${killer} killed ${victim} with ${skill}`;
            killFeed.insertBefore(item, killFeed.firstChild);
            
            setTimeout(() => {
                item.style.opacity = '0';
                item.style.transition = 'opacity 0.5s';
                setTimeout(() => item.remove(), 500);
            }, 3000);
            
            while (killFeed.children.length > 5) {
                killFeed.removeChild(killFeed.lastChild);
            }
        }
        
        // Random events
        setInterval(() => {
            const event = Math.floor(Math.random() * 4);
            if (event === 0) {
                hp = Math.max(10, hp - Math.floor(Math.random() * 20));
            } else if (event === 1) {
                hp = Math.min(100, hp + 15);
                mp = Math.min(100, mp + 10);
            } else if (event === 2) {
                score += Math.floor(Math.random() * 500 + 100);
                xp = Math.min(1000, xp + Math.floor(Math.random() * 50));
                addKillFeed('Player', enemies[Math.floor(Math.random() * enemies.length)],
                    skills[Math.floor(Math.random() * skills.length)]);
            } else {
                ammo = Math.max(0, ammo - Math.floor(Math.random() * 8));
            }
        }, 2000);
        
        function updateHUD() {
            document.getElementById('hp-fill').style.width = hp + '%';
            document.getElementById('hp-fill').style.background =
                hp < 30 ? 'linear-gradient(to right, #aa0000, #ff2222)' :
                hp < 60 ? 'linear-gradient(to right, #aa4400, #ff8800)' :
                'linear-gradient(to right, #cc2222, #ff4444)';
            document.getElementById('hp-text').textContent = `${hp}/100`;
            
            document.getElementById('mp-fill').style.width = mp + '%';
            document.getElementById('mp-text').textContent = `${mp}/100`;
            
            document.getElementById('xp-fill').style.width = (xp / 1000 * 100) + '%';
            document.getElementById('xp-text').textContent = `${xp}/1000`;
            
            document.getElementById('score').textContent = score.toLocaleString();
            document.getElementById('ammo-main').textContent = ammo;
            document.getElementById('ammo-main').style.color = ammo < 5 ? '#ff4444' : 'white';
        }
        
        let t = 0;
        function animate() {
            requestAnimationFrame(animate);
            t = performance.now() * 0.001;
            
            player.position.x = Math.sin(t * 0.5) * 5;
            player.position.z = Math.cos(t * 0.3) * 5;
            player.rotation.y = t * 0.5;
            
            updateHUD();
            
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

## 3D Score Counter (Number Flip)

```javascript
class ScoreCounter3D {
    constructor(scene, position) {
        this.scene = scene;
        this.position = position;
        this.score = 0;
        this.displayScore = 0;
        this.particles = [];
        
        this.createDisplay();
    }
    
    createDisplay() {
        this.canvas = document.createElement('canvas');
        this.canvas.width = 256;
        this.canvas.height = 80;
        this.texture = new THREE.CanvasTexture(this.canvas);
        
        this.mesh = new THREE.Mesh(
            new THREE.PlaneGeometry(4, 1.2),
            new THREE.MeshBasicMaterial({ map: this.texture, transparent: true, depthTest: false })
        );
        this.mesh.position.copy(this.position);
        this.scene.add(this.mesh);
        
        this.render();
    }
    
    render() {
        const ctx = this.canvas.getContext('2d');
        ctx.clearRect(0, 0, 256, 80);
        
        ctx.font = 'bold 52px "Courier New"';
        ctx.fillStyle = '#ffdd44';
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';
        ctx.shadowColor = '#ffaa00';
        ctx.shadowBlur = 10;
        ctx.fillText(Math.floor(this.displayScore).toLocaleString(), 128, 40);
        
        this.texture.needsUpdate = true;
    }
    
    addScore(amount) {
        this.score += amount;
        
        // Floating +score text
        this.spawnFloatingText('+' + amount);
    }
    
    spawnFloatingText(text) {
        const canvas = document.createElement('canvas');
        canvas.width = 192; canvas.height = 48;
        const ctx = canvas.getContext('2d');
        ctx.font = 'bold 32px Arial';
        ctx.fillStyle = '#ffff44';
        ctx.strokeStyle = '#000';
        ctx.lineWidth = 3;
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';
        ctx.strokeText(text, 96, 24);
        ctx.fillText(text, 96, 24);
        
        const mesh = new THREE.Mesh(
            new THREE.PlaneGeometry(1.5, 0.4),
            new THREE.MeshBasicMaterial({ map: new THREE.CanvasTexture(canvas), transparent: true, depthTest: false })
        );
        
        mesh.position.copy(this.position);
        mesh.position.x += (Math.random() - 0.5) * 2;
        mesh.userData = { elapsed: 0, vy: 1.5 };
        this.scene.add(mesh);
        this.particles.push(mesh);
    }
    
    update(dt, camera) {
        // Smooth score counting
        this.displayScore += (this.score - this.displayScore) * 0.15;
        this.render();
        
        // Billboard
        this.mesh.quaternion.copy(camera.quaternion);
        
        // Float particles
        this.particles = this.particles.filter(p => {
            p.userData.elapsed += dt;
            p.position.y += p.userData.vy * dt;
            p.userData.vy -= 3 * dt; // gravity
            p.material.opacity = Math.max(0, 1 - p.userData.elapsed * 1.5);
            p.quaternion.copy(camera.quaternion);
            
            if (p.userData.elapsed > 1.5) {
                this.scene.remove(p);
                p.geometry.dispose();
                p.material.map?.dispose();
                p.material.dispose();
                return false;
            }
            return true;
        });
    }
}
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Boss Health Bar
สร้าง health bar สำหรับ boss fight ที่มีทั้ง health bar ปกติและ phase indicators แสดงเมื่อ HP ลงถึง threshold

### แบบฝึกหัดที่ 2: Achievement Gallery
สร้าง gallery แสดง achievements ที่ unlock ได้แล้ว 8 รายการ ใน 3D grid แต่ละ badge คลิกได้เพื่อดูรายละเอียด

### แบบฝึกหัดที่ 3: Cooldown System
สร้าง skill bar 4 ช่องที่มีระบบ cooldown ที่สมจริง กด key เพื่อใช้ skill และแสดง countdown

### แบบฝึกหัดที่ 4: Damage Numbers
สร้างระบบ floating damage numbers ที่มี critical hits (สีเหลือง, ใหญ่กว่า), normal hits (สีขาว), และ healing (สีเขียว)

## สรุป

3D Progress Indicators สำคัญสำหรับ:
1. **Feedback ทันที** — ผู้เล่นรู้ว่าเกิดอะไรขึ้น
2. **Status communication** — HP/MP/XP ที่ชัดเจน
3. **Achievement celebration** — reward ที่ satisfying
4. **Juiciness** — particles, flashes ทำให้เกมสนุกขึ้น

Part 100 ครบ! ในบทต่อไปเราจะเรียนรู้ Performance Profiling
