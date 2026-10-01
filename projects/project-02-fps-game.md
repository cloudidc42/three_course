# Project 02: First-Person Shooter Game

## ภาพรวม

สร้าง FPS game ใน browser พร้อม:
- Pointer lock camera (FPS view)
- WASD movement + กระโดด
- Raycaster shooting
- Enemy AI state machine
- Health system + HUD
- Level geometry
- Collectibles

---

## โค้ดสมบูรณ์

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>FPS Game</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { overflow: hidden; background: #000; font-family: monospace; }
    canvas { display: block; }
    
    /* Crosshair */
    #crosshair {
      position: fixed; top: 50%; left: 50%;
      transform: translate(-50%, -50%);
      pointer-events: none; z-index: 10;
    }
    #crosshair div {
      position: absolute;
      background: rgba(255,255,255,0.8);
    }
    #ch-h { width: 16px; height: 2px; top: 50%; left: 50%; transform: translate(-50%,-50%); }
    #ch-v { width: 2px; height: 16px; top: 50%; left: 50%; transform: translate(-50%,-50%); }
    #ch-dot { width: 3px; height: 3px; border-radius: 50%; top: 50%; left: 50%; transform: translate(-50%,-50%); background: white; }
    
    /* HUD */
    #hud {
      position: fixed; inset: 0;
      pointer-events: none;
      z-index: 20;
    }
    
    /* Health */
    #health-bar-wrap {
      position: absolute; bottom: 20px; left: 20px;
    }
    #health-label { color: #aaa; font-size: 11px; margin-bottom: 4px; }
    #health-bar-bg {
      width: 150px; height: 10px;
      background: rgba(255,255,255,0.1);
      border-radius: 5px; overflow: hidden;
      border: 1px solid rgba(255,255,255,0.2);
    }
    #health-bar {
      height: 100%; width: 100%;
      background: linear-gradient(to right, #ff4444, #ff8844);
      border-radius: 5px;
      transition: width 0.3s;
    }
    #health-num { color: #ff6666; font-size: 12px; margin-top: 3px; }
    
    /* Ammo */
    #ammo {
      position: absolute; bottom: 20px; right: 20px;
      text-align: right; color: white;
    }
    #ammo-num { font-size: 32px; color: #88aaff; }
    #ammo-total { font-size: 14px; color: #666; }
    
    /* Score */
    #score {
      position: absolute; top: 20px; right: 20px;
      color: white; text-align: right;
    }
    #score-num { font-size: 24px; color: #ffdd44; }
    #score-label { font-size: 11px; color: #666; }
    
    /* Wave */
    #wave-info {
      position: absolute; top: 20px; left: 20px;
      color: #88aaff; font-size: 14px;
    }
    
    /* Damage flash */
    #damage-flash {
      position: absolute; inset: 0;
      background: rgba(255,0,0,0.3);
      opacity: 0;
      pointer-events: none;
    }
    
    /* Kill feed */
    #kill-feed {
      position: absolute; top: 60px; right: 20px;
      text-align: right;
    }
    .kill-message {
      color: #ff4444; font-size: 12px;
      animation: fadeOut 2s forwards;
    }
    @keyframes fadeOut {
      0% { opacity: 1; }
      70% { opacity: 1; }
      100% { opacity: 0; }
    }
    
    /* Start/Gameover overlays */
    .overlay {
      position: fixed; inset: 0;
      background: rgba(0,0,0,0.85);
      display: flex; flex-direction: column;
      align-items: center; justify-content: center;
      z-index: 100; color: white;
    }
    .overlay h1 { font-size: 36px; margin-bottom: 10px; }
    .overlay p { color: #aaa; margin-bottom: 20px; font-size: 14px; }
    .overlay button {
      padding: 12px 30px;
      background: #4488ff;
      border: none; color: white;
      border-radius: 5px; cursor: pointer;
      font-size: 16px; font-family: monospace;
    }
    .overlay button:hover { background: #5599ff; }
    
    #hit-marker {
      position: fixed; top: 50%; left: 50%;
      transform: translate(-50%,-50%);
      color: #ff4444; font-size: 20px;
      pointer-events: none; z-index: 15;
      opacity: 0;
    }
  </style>
</head>
<body>
  <canvas id="canvas"></canvas>
  
  <div id="crosshair">
    <div id="ch-h"></div>
    <div id="ch-v"></div>
    <div id="ch-dot"></div>
  </div>
  
  <div id="hud">
    <div id="health-bar-wrap">
      <div id="health-label">HP</div>
      <div id="health-bar-bg"><div id="health-bar"></div></div>
      <div id="health-num">100</div>
    </div>
    
    <div id="ammo">
      <div id="ammo-num">30</div>
      <div id="ammo-total">/ 90</div>
    </div>
    
    <div id="score">
      <div id="score-num">0</div>
      <div id="score-label">คะแนน</div>
    </div>
    
    <div id="wave-info">Wave 1</div>
    
    <div id="damage-flash"></div>
    <div id="kill-feed"></div>
  </div>
  
  <div id="hit-marker">✕</div>
  
  <div class="overlay" id="start-screen">
    <h1>🎮 FPS Game</h1>
    <p>WASD: เดิน | Mouse: มอง | คลิก: ยิง | Space: กระโดด | R: เติมกระสุน</p>
    <button id="start-btn">เริ่มเกม</button>
  </div>
  
  <div class="overlay" id="gameover-screen" style="display:none">
    <h1>💀 เกมจบ</h1>
    <p id="final-score"></p>
    <button id="restart-btn">เล่นใหม่</button>
  </div>
  
  <script type="module">
    import * as THREE from 'https://cdn.skypack.dev/three@0.160.0';
    
    // ===== State =====
    let gameState = 'start'; // start | playing | gameover
    let playerHP = 100;
    let score = 0;
    let ammo = 30;
    let totalAmmo = 90;
    let wave = 1;
    let enemiesLeft = 0;
    
    // ===== Renderer =====
    const canvas = document.getElementById('canvas');
    const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.outputColorSpace = THREE.SRGBColorSpace;
    renderer.shadowMap.enabled = true;
    
    // ===== Scene =====
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x111118);
    scene.fog = new THREE.Fog(0x111118, 20, 50);
    
    // ===== Camera (FPS) =====
    const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
    camera.position.set(0, 1.7, 0); // Eye height
    
    // Camera yaw pivot
    const cameraYaw = new THREE.Object3D();
    cameraYaw.add(camera);
    scene.add(cameraYaw);
    
    // ===== Pointer Lock =====
    let pointerLocked = false;
    let yaw = 0, pitch = 0;
    
    document.addEventListener('mousemove', e => {
      if (!pointerLocked || gameState !== 'playing') return;
      const sensitivity = 0.002;
      yaw -= e.movementX * sensitivity;
      pitch -= e.movementY * sensitivity;
      pitch = Math.max(-Math.PI / 3, Math.min(Math.PI / 3, pitch));
      
      cameraYaw.rotation.y = yaw;
      camera.rotation.x = pitch;
    });
    
    document.addEventListener('pointerlockchange', () => {
      pointerLocked = document.pointerLockElement === canvas;
    });
    
    // ===== Input =====
    const keys = {};
    document.addEventListener('keydown', e => {
      keys[e.code] = true;
      
      if (e.code === 'KeyR' && gameState === 'playing') reload();
    });
    document.addEventListener('keyup', e => keys[e.code] = false);
    
    // ===== Level =====
    function createLevel() {
      const group = new THREE.Group();
      
      // Floor
      const floorMat = new THREE.MeshStandardMaterial({ color: 0x333344, roughness: 0.9 });
      const floor = new THREE.Mesh(new THREE.PlaneGeometry(50, 50), floorMat);
      floor.rotation.x = -Math.PI / 2;
      floor.receiveShadow = true;
      group.add(floor);
      
      // Ceiling
      const ceilMat = new THREE.MeshStandardMaterial({ color: 0x222233, roughness: 0.9 });
      const ceil = new THREE.Mesh(new THREE.PlaneGeometry(50, 50), ceilMat);
      ceil.rotation.x = Math.PI / 2;
      ceil.position.y = 4;
      group.add(ceil);
      
      // Walls
      const wallMat = new THREE.MeshStandardMaterial({ color: 0x445566, roughness: 0.8 });
      const wallDefs = [
        { x: 0, z: -25, rx: 0, ry: 0 },
        { x: 0, z: 25, rx: 0, ry: Math.PI },
        { x: -25, z: 0, rx: 0, ry: Math.PI / 2 },
        { x: 25, z: 0, rx: 0, ry: -Math.PI / 2 }
      ];
      wallDefs.forEach(w => {
        const wall = new THREE.Mesh(new THREE.PlaneGeometry(50, 4), wallMat);
        wall.position.set(w.x, 2, w.z);
        wall.rotation.y = w.ry;
        wall.receiveShadow = true;
        group.add(wall);
      });
      
      // Crates (cover)
      const crateMat = new THREE.MeshStandardMaterial({ color: 0x887755 });
      const cratePositions = [
        [-5, 0, -5], [5, 0, -5], [-8, 0, 3], [8, 0, 3],
        [-3, 0, 8], [3, 0, 8], [0, 0, -12], [-12, 0, -2], [12, 0, -2]
      ];
      
      const colliders = [];
      
      cratePositions.forEach(pos => {
        const size = 1.5 + Math.random() * 1;
        const crate = new THREE.Mesh(
          new THREE.BoxGeometry(size, size, size),
          crateMat.clone()
        );
        crate.position.set(pos[0], size / 2, pos[2]);
        crate.castShadow = true;
        crate.receiveShadow = true;
        group.add(crate);
        
        // Store collider bounds
        const box = new THREE.Box3().setFromObject(crate);
        colliders.push(box);
      });
      
      return { group, colliders };
    }
    
    const { group: levelGroup, colliders } = createLevel();
    scene.add(levelGroup);
    
    // ===== Lighting =====
    scene.add(new THREE.AmbientLight(0x334455, 0.5));
    
    // Point lights around the arena
    const lightColors = [0x4466ff, 0xff4444, 0x44ff66, 0xffaa00];
    const lightPositions = [[-15, 3, -15], [15, 3, -15], [-15, 3, 15], [15, 3, 15]];
    lightPositions.forEach((pos, i) => {
      const light = new THREE.PointLight(lightColors[i], 1, 25);
      light.position.set(...pos);
      light.castShadow = true;
      scene.add(light);
    });
    
    // ===== Enemies =====
    const enemies = [];
    
    function createEnemy(position) {
      const group = new THREE.Group();
      
      // Body
      const body = new THREE.Mesh(
        new THREE.CapsuleGeometry(0.35, 1.2, 4, 8),
        new THREE.MeshStandardMaterial({ color: 0xff2222 })
      );
      body.position.y = 1.2;
      group.add(body);
      
      // Head
      const head = new THREE.Mesh(
        new THREE.SphereGeometry(0.3, 12, 12),
        new THREE.MeshStandardMaterial({ color: 0xff4444 })
      );
      head.position.y = 2.3;
      group.add(head);
      
      // Eyes
      for (let s = -1; s <= 1; s += 2) {
        const eye = new THREE.Mesh(
          new THREE.SphereGeometry(0.08, 6, 6),
          new THREE.MeshBasicMaterial({ color: 0xffff00 })
        );
        eye.position.set(s * 0.12, 2.35, 0.27);
        group.add(eye);
      }
      
      group.position.copy(position);
      scene.add(group);
      
      const enemy = {
        group,
        hp: 50,
        maxHp: 50,
        speed: 2.5 + Math.random(),
        state: 'patrol',  // patrol | chase | attack | dead
        direction: new THREE.Vector3(1, 0, 0),
        patrolTarget: new THREE.Vector3(),
        lastAttackTime: 0,
        attackCooldown: 1.5
      };
      
      // Health bar above enemy
      const hpBarGeo = new THREE.PlaneGeometry(0.8, 0.08);
      const hpBarMat = new THREE.MeshBasicMaterial({ color: 0x00ff00 });
      const hpBar = new THREE.Mesh(hpBarGeo, hpBarMat);
      hpBar.position.set(0, 3.2, 0);
      group.add(hpBar);
      enemy.hpBar = hpBar;
      enemy.hpBarMat = hpBarMat;
      
      setNewPatrolTarget(enemy);
      enemies.push(enemy);
      enemiesLeft++;
      
      return enemy;
    }
    
    function setNewPatrolTarget(enemy) {
      enemy.patrolTarget.set(
        (Math.random() - 0.5) * 40,
        0,
        (Math.random() - 0.5) * 40
      );
    }
    
    function updateEnemy(enemy, delta, time) {
      if (enemy.state === 'dead') return;
      
      const playerPos = cameraYaw.position.clone();
      const enemyPos = enemy.group.position.clone();
      const distToPlayer = enemyPos.distanceTo(playerPos);
      
      // Update HP bar color
      const hpRatio = enemy.hp / enemy.maxHp;
      enemy.hpBarMat.color.setHSL(hpRatio * 0.33, 1, 0.5);
      enemy.hpBar.scale.x = hpRatio;
      enemy.hpBar.lookAt(camera.getWorldPosition(new THREE.Vector3()));
      
      // State machine
      if (distToPlayer < 20) {
        enemy.state = 'chase';
      } else {
        enemy.state = 'patrol';
      }
      
      if (distToPlayer < 2.5) {
        enemy.state = 'attack';
      }
      
      if (enemy.state === 'patrol') {
        // Move to patrol target
        const dir = enemy.patrolTarget.clone().sub(enemy.group.position).normalize();
        dir.y = 0;
        enemy.group.position.addScaledVector(dir, enemy.speed * 0.5 * delta);
        enemy.group.lookAt(enemy.patrolTarget.x, enemy.group.position.y, enemy.patrolTarget.z);
        
        if (enemy.group.position.distanceTo(enemy.patrolTarget) < 1) {
          setNewPatrolTarget(enemy);
        }
      }
      
      if (enemy.state === 'chase') {
        const dir = playerPos.clone().sub(enemy.group.position).normalize();
        dir.y = 0;
        enemy.group.position.addScaledVector(dir, enemy.speed * delta);
        enemy.group.lookAt(playerPos.x, enemy.group.position.y, playerPos.z);
      }
      
      if (enemy.state === 'attack') {
        enemy.group.lookAt(playerPos.x, enemy.group.position.y, playerPos.z);
        
        if (time - enemy.lastAttackTime > enemy.attackCooldown) {
          enemy.lastAttackTime = time;
          damagePlayer(10);
        }
      }
    }
    
    // ===== Player Movement =====
    const playerVelocity = new THREE.Vector3();
    let onGround = true;
    const gravity = -15;
    const jumpForce = 6;
    const moveSpeed = 5;
    
    function updatePlayer(delta) {
      if (gameState !== 'playing') return;
      
      // Move direction
      const dir = new THREE.Vector3();
      const forward = new THREE.Vector3(-Math.sin(yaw), 0, -Math.cos(yaw));
      const right = new THREE.Vector3(Math.cos(yaw), 0, -Math.sin(yaw));
      
      if (keys['KeyW'] || keys['ArrowUp']) dir.addScaledVector(forward, 1);
      if (keys['KeyS'] || keys['ArrowDown']) dir.addScaledVector(forward, -1);
      if (keys['KeyA'] || keys['ArrowLeft']) dir.addScaledVector(right, -1);
      if (keys['KeyD'] || keys['ArrowRight']) dir.addScaledVector(right, 1);
      
      if (dir.length() > 0) dir.normalize();
      
      // Apply movement
      playerVelocity.x = dir.x * moveSpeed;
      playerVelocity.z = dir.z * moveSpeed;
      
      // Jump
      if (keys['Space'] && onGround) {
        playerVelocity.y = jumpForce;
        onGround = false;
      }
      
      // Gravity
      playerVelocity.y += gravity * delta;
      
      // Move
      const newPos = cameraYaw.position.clone().addScaledVector(playerVelocity, delta);
      
      // Floor clamp
      if (newPos.y < 0) {
        newPos.y = 0;
        playerVelocity.y = 0;
        onGround = true;
      }
      
      // Wall/crate collision (simple AABB)
      const playerBox = new THREE.Box3(
        new THREE.Vector3(newPos.x - 0.4, newPos.y, newPos.z - 0.4),
        new THREE.Vector3(newPos.x + 0.4, newPos.y + 2, newPos.z + 0.4)
      );
      
      let blocked = false;
      for (const col of colliders) {
        if (playerBox.intersectsBox(col)) {
          blocked = true;
          break;
        }
      }
      
      if (!blocked) {
        cameraYaw.position.copy(newPos);
      } else {
        // Try sliding
        const slideX = new THREE.Vector3(newPos.x, cameraYaw.position.y, cameraYaw.position.z);
        const slideXBox = new THREE.Box3(
          new THREE.Vector3(slideX.x - 0.4, slideX.y, slideX.z - 0.4),
          new THREE.Vector3(slideX.x + 0.4, slideX.y + 2, slideX.z + 0.4)
        );
        let blockedX = false;
        for (const col of colliders) {
          if (slideXBox.intersectsBox(col)) { blockedX = true; break; }
        }
        if (!blockedX) cameraYaw.position.x = newPos.x;
        
        const slideZ = new THREE.Vector3(cameraYaw.position.x, newPos.y, newPos.z);
        const slideZBox = new THREE.Box3(
          new THREE.Vector3(slideZ.x - 0.4, slideZ.y, slideZ.z - 0.4),
          new THREE.Vector3(slideZ.x + 0.4, slideZ.y + 2, slideZ.z + 0.4)
        );
        let blockedZ = false;
        for (const col of colliders) {
          if (slideZBox.intersectsBox(col)) { blockedZ = true; break; }
        }
        if (!blockedZ) cameraYaw.position.z = newPos.z;
        
        // Y always apply
        cameraYaw.position.y = Math.max(0, newPos.y);
      }
      
      // Clamp to arena
      cameraYaw.position.x = Math.max(-24, Math.min(24, cameraYaw.position.x));
      cameraYaw.position.z = Math.max(-24, Math.min(24, cameraYaw.position.z));
    }
    
    // ===== Shooting =====
    const raycaster = new THREE.Raycaster();
    const bulletHoles = [];
    
    function shoot() {
      if (gameState !== 'playing') return;
      if (ammo <= 0) { showMessage('กระสุนหมด! กด R เพื่อเติม'); return; }
      
      ammo--;
      updateHUD();
      
      // Raycast from center screen
      raycaster.setFromCamera(new THREE.Vector2(0, 0), camera);
      
      const enemyMeshes = enemies
        .filter(e => e.state !== 'dead')
        .map(e => e.group);
      
      const hits = raycaster.intersectObjects(enemyMeshes, true);
      
      if (hits.length > 0) {
        const hitGroup = hits[0].object.parent;
        const enemy = enemies.find(e => e.group === hitGroup || e.group.children.includes(hits[0].object));
        
        if (enemy && enemy.state !== 'dead') {
          const damage = 20 + Math.floor(Math.random() * 10);
          enemy.hp -= damage;
          
          showHitMarker();
          
          if (enemy.hp <= 0) {
            killEnemy(enemy);
          }
        }
      } else {
        // Bullet impact on wall/floor (visual only)
        const allObjects = [...scene.children];
        const wallHits = raycaster.intersectObjects(allObjects, true);
        if (wallHits.length > 0) {
          createBulletHole(wallHits[0].point, wallHits[0].face.normal);
        }
      }
    }
    
    function createBulletHole(position, normal) {
      const geo = new THREE.CircleGeometry(0.05, 6);
      const mat = new THREE.MeshBasicMaterial({ color: 0x111111, transparent: true, opacity: 0.8 });
      const hole = new THREE.Mesh(geo, mat);
      hole.position.copy(position).addScaledVector(normal, 0.01);
      hole.lookAt(position.clone().add(normal));
      scene.add(hole);
      
      bulletHoles.push({ mesh: hole, time: Date.now() });
      
      // Limit
      if (bulletHoles.length > 50) {
        const old = bulletHoles.shift();
        scene.remove(old.mesh);
      }
    }
    
    function killEnemy(enemy) {
      score += 100;
      enemiesLeft--;
      enemy.state = 'dead';
      
      // Death animation - fall down
      enemy.group.rotation.x = Math.PI / 2;
      enemy.group.position.y = 0.3;
      
      // Remove after delay
      setTimeout(() => {
        scene.remove(enemy.group);
      }, 2000);
      
      // Kill feed
      addKillFeed();
      
      updateHUD();
      
      // Check wave complete
      if (enemiesLeft <= 0) {
        setTimeout(() => startNextWave(), 3000);
      }
    }
    
    // ===== Player Damage =====
    function damagePlayer(amount) {
      if (gameState !== 'playing') return;
      playerHP = Math.max(0, playerHP - amount);
      
      // Flash red
      const flash = document.getElementById('damage-flash');
      flash.style.opacity = '1';
      setTimeout(() => flash.style.opacity = '0', 200);
      
      updateHUD();
      
      if (playerHP <= 0) {
        endGame();
      }
    }
    
    // ===== Reload =====
    function reload() {
      if (totalAmmo <= 0) return;
      const needed = 30 - ammo;
      const give = Math.min(needed, totalAmmo);
      ammo += give;
      totalAmmo -= give;
      updateHUD();
    }
    
    // ===== Hit Marker =====
    function showHitMarker() {
      const hm = document.getElementById('hit-marker');
      hm.style.opacity = '1';
      hm.style.transition = 'none';
      setTimeout(() => {
        hm.style.transition = 'opacity 0.3s';
        hm.style.opacity = '0';
      }, 50);
    }
    
    // ===== HUD Update =====
    function updateHUD() {
      document.getElementById('health-bar').style.width = `${playerHP}%`;
      document.getElementById('health-num').textContent = playerHP;
      document.getElementById('ammo-num').textContent = ammo;
      document.getElementById('ammo-total').textContent = `/ ${totalAmmo}`;
      document.getElementById('score-num').textContent = score;
      document.getElementById('wave-info').textContent = `Wave ${wave} | เหลือ: ${Math.max(0, enemiesLeft)}`;
    }
    
    function addKillFeed() {
      const feed = document.getElementById('kill-feed');
      const msg = document.createElement('div');
      msg.className = 'kill-message';
      msg.textContent = '💀 +100';
      feed.appendChild(msg);
      setTimeout(() => feed.removeChild(msg), 2000);
    }
    
    function showMessage(text) {
      // Show temporary message
      const existing = document.getElementById('temp-msg');
      if (existing) existing.remove();
      
      const msg = document.createElement('div');
      msg.id = 'temp-msg';
      msg.style.cssText = `
        position: fixed; top: 50%; left: 50%;
        transform: translate(-50%, -120px);
        background: rgba(0,0,0,0.7);
        color: white; padding: 8px 16px;
        border-radius: 5px; font-size: 13px;
        pointer-events: none; z-index: 50;
      `;
      msg.textContent = text;
      document.body.appendChild(msg);
      setTimeout(() => msg.remove(), 2000);
    }
    
    // ===== Wave System =====
    function startWave(waveNum) {
      wave = waveNum;
      const count = 3 + waveNum * 2;
      enemiesLeft = 0;
      
      for (let i = 0; i < count; i++) {
        const angle = (i / count) * Math.PI * 2;
        const dist = 15 + Math.random() * 8;
        const pos = new THREE.Vector3(
          Math.cos(angle) * dist,
          0,
          Math.sin(angle) * dist
        );
        createEnemy(pos);
      }
      
      updateHUD();
    }
    
    function startNextWave() {
      // Clear dead enemies from array
      enemies.length = 0;
      startWave(wave + 1);
      totalAmmo += 30;
      ammo = 30;
      updateHUD();
    }
    
    // ===== Game Flow =====
    function startGame() {
      gameState = 'playing';
      playerHP = 100;
      score = 0;
      ammo = 30;
      totalAmmo = 90;
      wave = 0;
      enemies.length = 0;
      
      cameraYaw.position.set(0, 0, 0);
      yaw = 0;
      pitch = 0;
      
      document.getElementById('start-screen').style.display = 'none';
      document.getElementById('gameover-screen').style.display = 'none';
      
      canvas.requestPointerLock();
      
      startWave(1);
      updateHUD();
    }
    
    function endGame() {
      gameState = 'gameover';
      document.exitPointerLock();
      
      document.getElementById('gameover-screen').style.display = 'flex';
      document.getElementById('final-score').textContent = `คะแนนสุดท้าย: ${score} | Wave: ${wave}`;
    }
    
    // Button handlers
    document.getElementById('start-btn').addEventListener('click', startGame);
    document.getElementById('restart-btn').addEventListener('click', startGame);
    
    canvas.addEventListener('click', () => {
      if (gameState === 'playing') {
        if (!pointerLocked) {
          canvas.requestPointerLock();
        } else {
          shoot();
        }
      }
    });
    
    // ===== Resize =====
    window.addEventListener('resize', () => {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    });
    
    // ===== Animation =====
    const clock = new THREE.Clock();
    
    function animate() {
      requestAnimationFrame(animate);
      
      const delta = Math.min(clock.getDelta(), 0.05);
      const time = clock.getElapsedTime();
      
      if (gameState === 'playing') {
        updatePlayer(delta);
        
        enemies.forEach(e => updateEnemy(e, delta, time));
      }
      
      renderer.render(scene, camera);
    }
    
    animate();
    updateHUD();
  </script>
</body>
</html>
```

---

## Features

- **Pointer Lock**: mouse control FPS style
- **WASD + Jump + Gravity**
- **Collision Detection**: กับ crates
- **Enemy AI**: Patrol → Chase → Attack state machine
- **Raycaster Shooting**: ยิงถูก enemy ลด HP
- **Wave System**: enemies เพิ่มขึ้นทุก wave
- **HUD**: HP bar, ammo counter, score, kill feed
- **Damage flash**: สีแดงเมื่อโดนโจมตี

## การปรับปรุงเพิ่มเติม

- เพิ่ม Cannon.js physics
- เพิ่ม weapon models (GLTF)
- เพิ่ม sound effects (Web Audio API)
- เพิ่ม particle effects สำหรับการยิง
- เพิ่ม minimap
