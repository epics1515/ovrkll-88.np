<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>FPS Arena: Overkill</title>
    <style>
        body { margin: 0; background-color: #000; overflow: hidden; font-family: 'Courier New', Courier, monospace; }
        canvas { display: block; }
        #crosshair {
            position: absolute; top: 50%; left: 50%;
            width: 12px; height: 12px;
            border: 2px solid rgba(0, 255, 0, 0.7);
            border-radius: 50%; transform: translate(-50%, -50%);
            pointer-events: none; z-index: 10;
        }
        #pauseMenu {
            position: absolute; top: 0; left: 0;
            width: 100%; height: 100%;
            background: rgba(0, 0, 0, 0.9);
            display: flex; flex-direction: column;
            justify-content: center; align-items: center;
            color: white; z-index: 20; visibility: visible;
        }
        #pauseMenu h1 { font-size: 4rem; letter-spacing: 10px; margin-bottom: 20px; color: #ff0000; text-shadow: 2px 2px #550000; }
        .setting-box { background: #222; padding: 25px; border: 2px solid #444; border-radius: 8px; text-align: center; margin-bottom: 20px; }
        input[type=range] { cursor: pointer; width: 200px; margin: 10px 0; }
        #resumeBtn {
            padding: 15px 40px; font-size: 1.5rem; background: #ff0000; color: white;
            border: none; cursor: pointer; font-family: inherit; font-weight: bold;
            transition: transform 0.1s, background 0.2s; box-shadow: 0 0 20px rgba(255,0,0,0.4);
        }
        #weaponHUD {
            position: absolute; top: 50%; left: 50%;
            transform: translate(-50%, -50%);
            width: 250px; height: 250px;
            background: rgba(255, 255, 255, 0.1);
            border: 4px solid rgba(255, 255, 255, 0.2);
            border-radius: 50%;
            display: flex; flex-direction: column;
            justify-content: center; align-items: center;
            color: white; opacity: 0; transition: opacity 0.3s;
            pointer-events: none; z-index: 15;
        }
        #weaponHUD.active { opacity: 1; }
        #weaponName { font-size: 1.5rem; font-weight: bold; text-transform: uppercase; color: #00ff00; }
        #cooldown { margin-top: 5px; font-size: 0.7rem; color: #ff4444; height: 1em; }
    </style>
</head>
<body>

<div id="crosshair"></div>
<div id="weaponHUD">
    <div style="font-size: 0.8rem; color: #aaa;">WEAPON</div>
    <div id="weaponName">GUN</div>
    <div id="cooldown"></div>
</div>

<div id="pauseMenu">
    <h1 id="menuTitle">FPS ARENA</h1>
    <div class="setting-box">
        <label>MOUSE SENSITIVITY</label><br>
        <input type="range" id="sensSlider" min="0.1" max="5.0" step="0.1" value="1.0">
        <div id="sensValue">1.0</div>
    </div>
    <button id="resumeBtn">START GAME</button>
</div>

<script type="importmap">
    { "imports": { "three": "https://unpkg.com/three@0.160.0/build/three.module.js" } }
</script>

<script type="module">
    import * as THREE from 'three';

    // --- Texture Generator ---
    function createGrittyTexture(color1, color2, repeat = 1) {
        const size = 512;
        const canvas = document.createElement('canvas');
        canvas.width = canvas.height = size;
        const ctx = canvas.getContext('2d');
        ctx.fillStyle = color1;
        ctx.fillRect(0, 0, size, size);
        for (let i = 0; i < 12000; i++) {
            ctx.fillStyle = color2;
            ctx.globalAlpha = Math.random() * 0.4;
            ctx.fillRect(Math.random() * size, Math.random() * size, 3, 3);
        }
        const tex = new THREE.CanvasTexture(canvas);
        tex.wrapS = tex.wrapT = THREE.RepeatWrapping;
        tex.repeat.set(repeat, repeat);
        return tex;
    }

    const wallTex = createGrittyTexture('#333333', '#000000', 4);
    const floorTex = createGrittyTexture('#111111', '#222222', 10);
    const ceilingTex = createGrittyTexture('#666666', '#333333', 6);
    const handTex = createGrittyTexture('#ffffff', '#888888');

    // --- Audio ---
    const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    function playSound(freq, type, decay, vol) {
        if (audioCtx.state === 'suspended') audioCtx.resume();
        const osc = audioCtx.createOscillator();
        const gain = audioCtx.createGain();
        osc.type = type;
        osc.frequency.setValueAtTime(freq, audioCtx.currentTime);
        osc.frequency.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + decay);
        gain.gain.setValueAtTime(vol, audioCtx.currentTime);
        gain.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + decay);
        osc.connect(gain); gain.connect(audioCtx.destination);
        osc.start(); osc.stop(audioCtx.currentTime + decay);
    }

    // --- Scene Setup ---
    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.toneMapping = THREE.ReinhardToneMapping;
    document.body.appendChild(renderer.domElement);

    scene.add(new THREE.HemisphereLight(0xffffff, 0x444444, 1.2));
    const dirLight = new THREE.DirectionalLight(0xffffff, 1.0);
    dirLight.position.set(5, 10, 7);
    scene.add(dirLight);

    const floor = new THREE.Mesh(new THREE.PlaneGeometry(100, 100), new THREE.MeshStandardMaterial({ map: floorTex }));
    floor.rotation.x = -Math.PI / 2; floor.position.y = -2;
    scene.add(floor);

    const ceiling = new THREE.Mesh(new THREE.PlaneGeometry(100, 100), new THREE.MeshStandardMaterial({ map: ceilingTex, side: THREE.BackSide }));
    ceiling.rotation.x = -Math.PI / 2; ceiling.position.y = 18;
    scene.add(ceiling);

    const createWall = (w, h, x, z, ry = 0) => {
        const wall = new THREE.Mesh(new THREE.PlaneGeometry(w, h), new THREE.MeshStandardMaterial({ map: wallTex }));
        wall.position.set(x, h/2 - 2, z);
        wall.rotation.y = ry;
        scene.add(wall);
    };
    createWall(100, 20, 0, -25); createWall(100, 20, 0, 25, Math.PI); 
    createWall(50, 20, -25, 0, Math.PI/2); createWall(50, 20, 25, 0, -Math.PI/2);

    // --- Player & Hands ---
    const playerGroup = new THREE.Group();
    scene.add(playerGroup);
    playerGroup.add(camera);
    camera.position.set(0, 1.6, 0);

    const viewModel = new THREE.Group();
    camera.add(viewModel);

    const handMat = new THREE.MeshStandardMaterial({ map: handTex, roughness: 0.7 });
    const leftArm = new THREE.Mesh(new THREE.CylinderGeometry(0.1, 0.1, 0.8), handMat);
    leftArm.rotation.x = Math.PI / 2.2; leftArm.position.set(-0.4, -0.5, -0.6);
    const rightArm = new THREE.Mesh(new THREE.CylinderGeometry(0.1, 0.1, 0.8), handMat);
    rightArm.rotation.x = Math.PI / 2.2; rightArm.position.set(0.4, -0.5, -0.6);
    viewModel.add(leftArm, rightArm);

    // Weapons
    const weapons = [];
    let currentWeaponIndex = 0;
    let weaponYOffset = 0; 
    let switching = false;
    let lastGrenadeTime = 0;
    const GRENADE_COOLDOWN = 1500;

    const gunGroup = new THREE.Group();
    const gunBody = new THREE.Mesh(new THREE.BoxGeometry(0.15, 0.2, 0.8), new THREE.MeshStandardMaterial({ color: 0x111111 }));
    const flash = new THREE.Mesh(new THREE.CylinderGeometry(0.15, 0, 0.4), new THREE.MeshBasicMaterial({ color: 0xffcc00, transparent: true, opacity: 0 }));
    flash.rotation.x = -Math.PI / 2; flash.position.z = -0.7;
    gunGroup.add(gunBody, flash);
    gunGroup.position.set(0.3, -0.3, -0.8);
    viewModel.add(gunGroup);
    weapons.push({ name: "Gun", mesh: gunGroup, type: "auto", hands: 2 });

    const grenadeGroup = new THREE.Group();
    const gBody = new THREE.Mesh(new THREE.SphereGeometry(0.12, 8, 8), new THREE.MeshStandardMaterial({ color: 0x1a3300 }));
    grenadeGroup.add(gBody);
    grenadeGroup.position.set(0.4, -0.3, -0.6);
    grenadeGroup.visible = false;
    viewModel.add(grenadeGroup);
    weapons.push({ name: "Grenade", mesh: grenadeGroup, type: "single", hands: 1 });

    // Dummy
    const dummy = new THREE.Mesh(new THREE.BoxGeometry(1.5, 3, 0.8), new THREE.MeshStandardMaterial({ color: 0x880000, map: wallTex }));
    dummy.position.set(0, -0.5, -15);
    scene.add(dummy);
    let dummyVel = new THREE.Vector3(0, 0, 0);

    // --- State ---
    let isPaused = true;
    let sensitivity = 1.0;
    let pitch = 0, yaw = 0;
    const keys = {};
    const bullets = [];
    const particles = [];
    let velY = 0, isJumping = false, shootTimer = 0, screenShake = 0;
    let hudTimer = 0;

    const switchWeapon = (dir) => {
        if (switching) return;
        switching = true;
        
        const drop = setInterval(() => {
            weaponYOffset -= 0.1;
            if (weaponYOffset <= -1.5) {
                clearInterval(drop);
                weapons[currentWeaponIndex].mesh.visible = false;
                currentWeaponIndex = (currentWeaponIndex + dir + weapons.length) % weapons.length;
                const cur = weapons[currentWeaponIndex];
                cur.mesh.visible = true;
                leftArm.visible = cur.hands === 2;
                
                document.getElementById('weaponName').innerText = cur.name;
                document.getElementById('weaponHUD').classList.add('active');
                hudTimer = 100; // Reset timer for UI visibility
                playSound(300, 'sine', 0.05, 0.1);

                const raise = setInterval(() => {
                    weaponYOffset += 0.1;
                    if (weaponYOffset >= 0) {
                        weaponYOffset = 0;
                        switching = false;
                        clearInterval(raise);
                    }
                }, 16);
            }
        }, 16);
    };

    const spawnExplosion = (pos, color, count, size = 0.2, gravity = 0.01) => {
        for(let i=0; i<count; i++) {
            const p = new THREE.Mesh(new THREE.BoxGeometry(size, size, size), new THREE.MeshBasicMaterial({ color: color }));
            p.position.copy(pos);
            particles.push({ 
                mesh: p, 
                vx: (Math.random()-0.5)*0.6, 
                vy: Math.random()*0.5, 
                vz: (Math.random()-0.5)*0.6, 
                life: 1.0,
                grav: gravity
            });
            scene.add(p);
        }
    }

    const fireAction = () => {
        if (switching) return;
        const now = Date.now();
        const cur = weapons[currentWeaponIndex];
        if(cur.name === "Grenade" && now - lastGrenadeTime < GRENADE_COOLDOWN) return;
        if(cur.name === "Grenade") lastGrenadeTime = now;

        screenShake = cur.name === "Gun" ? 0.04 : 0.1;
        const b = new THREE.Mesh(
            new THREE.SphereGeometry(cur.name === "Gun" ? 0.05 : 0.15),
            new THREE.MeshBasicMaterial({ color: cur.name === "Gun" ? 0xffff00 : 0x1a3300 })
        );
        b.position.copy(cur.mesh.getWorldPosition(new THREE.Vector3()));
        const dir = new THREE.Vector3(0, 0, -1).applyQuaternion(camera.getWorldQuaternion(new THREE.Quaternion()));
        
        bullets.push({ 
            mesh: b, 
            dir: dir.clone().multiplyScalar(cur.name === "Gun" ? 1.8 : 0.6),
            gravity: cur.name === "Grenade" ? 0.018 : 0,
            isGrenade: cur.name === "Grenade",
            groundTime: 0 
        });
        scene.add(b);
        if(cur.name === "Gun") { flash.material.opacity = 1; playSound(160, 'sawtooth', 0.1, 0.2); }
        else playSound(180, 'sine', 0.2, 0.2);
    };

    // --- Loop ---
    document.getElementById('resumeBtn').addEventListener('click', () => renderer.domElement.requestPointerLock());
    document.getElementById('sensSlider').addEventListener('input', (e) => {
        sensitivity = parseFloat(e.target.value);
        document.getElementById('sensValue').innerText = sensitivity.toFixed(1);
    });
    document.addEventListener('pointerlockchange', () => {
        isPaused = document.pointerLockElement !== renderer.domElement;
        document.getElementById('pauseMenu').style.visibility = isPaused ? 'visible' : 'hidden';
    });
    window.addEventListener('mousemove', (e) => {
        if (isPaused) return;
        yaw -= e.movementX * 0.002 * sensitivity;
        pitch -= e.movementY * 0.002 * sensitivity;
        pitch = Math.max(-Math.PI/2.1, Math.min(Math.PI/2.1, pitch));
        playerGroup.rotation.y = yaw;
        camera.rotation.x = pitch;
    });
    window.addEventListener('keydown', (e) => {
        keys[e.code] = true;
        if (e.code === 'KeyE') switchWeapon(1);
        if (e.code === 'KeyQ') switchWeapon(-1);
        if (e.code === 'KeyF' && !isPaused && weapons[currentWeaponIndex].type === 'single') fireAction();
    });
    window.addEventListener('keyup', (e) => keys[e.code] = false);

    function animate() {
        requestAnimationFrame(animate);
        if (!isPaused) {
            // UI Fading
            if (hudTimer > 0) {
                hudTimer--;
                if (hudTimer === 0) document.getElementById('weaponHUD').classList.remove('active');
            }

            const moveSpeed = 0.15;
            const forward = new THREE.Vector3(0, 0, -1).applyQuaternion(playerGroup.quaternion);
            const right = new THREE.Vector3(1, 0, 0).applyQuaternion(playerGroup.quaternion);
            let nX = playerGroup.position.x, nZ = playerGroup.position.z;
            if (keys['KeyW']) { nX += forward.x * moveSpeed; nZ += forward.z * moveSpeed; }
            if (keys['KeyS']) { nX -= forward.x * moveSpeed; nZ -= forward.z * moveSpeed; }
            if (keys['KeyA']) { nX -= right.x * moveSpeed; nZ -= right.z * moveSpeed; }
            if (keys['KeyD']) { nX += right.x * moveSpeed; nZ += right.z * moveSpeed; }
            if (nX > -24 && nX < 24) playerGroup.position.x = nX;
            if (nZ > -24 && nZ < 24) playerGroup.position.z = nZ;

            if (keys['Space'] && !isJumping) { velY = 0.3; isJumping = true; }
            if (isJumping) {
                playerGroup.position.y += velY; velY -= 0.015;
                if (playerGroup.position.y <= 0) { playerGroup.position.y = 0; isJumping = false; }
            }

            if (keys['KeyF'] && weapons[currentWeaponIndex].type === 'auto') {
                shootTimer++; if (shootTimer % 5 === 0) fireAction();
            } else { flash.material.opacity *= 0.5; }

            if(weapons[currentWeaponIndex].name === "Grenade") {
                const rem = Math.max(0, GRENADE_COOLDOWN - (Date.now() - lastGrenadeTime));
                document.getElementById('cooldown').innerText = rem > 0 ? `READY IN: ${(rem/1000).toFixed(1)}s` : "READY";
            } else { document.getElementById('cooldown').innerText = ""; }

            // Projectile Physics
            bullets.forEach((b, i) => {
                b.mesh.position.add(b.dir);
                if (b.gravity) {
                    b.dir.y -= b.gravity;
                    b.mesh.rotation.x += 0.2;
                    if (b.mesh.position.y < -1.8) {
                        b.mesh.position.y = -1.8; 
                        b.dir.y *= -0.3; b.dir.x *= 0.8; b.dir.z *= 0.8;
                        b.groundTime++;
                    }
                }

                const dist = b.mesh.position.distanceTo(dummy.position);
                const isGrenadeExploding = b.isGrenade && b.groundTime > 10;

                if ((!b.isGrenade && dist < 1.5) || isGrenadeExploding) {
                    if (isGrenadeExploding) {
                        spawnExplosion(b.mesh.position, 0xff7700, 40, 0.4);
                        playSound(50, 'sine', 0.5, 0.5);
                        screenShake = 0.6;
                        if (dist < 6) {
                            const force = dummy.position.clone().sub(b.mesh.position).normalize();
                            dummyVel.add(force.multiplyScalar(0.8));
                            dummyVel.y += 0.4; // Launch dummy upward
                            spawnExplosion(dummy.position, 0xaa0000, 20, 0.1);
                        }
                    } else {
                        spawnExplosion(b.mesh.position, 0xaa0000, 10, 0.1);
                        playSound(100, 'square', 0.05, 0.2);
                        dummyVel.z -= 0.05;
                    }
                    scene.remove(b.mesh); bullets.splice(i, 1);
                } else if (b.mesh.position.length() > 60) {
                    scene.remove(b.mesh); bullets.splice(i, 1);
                }
            });

            // Particles & Dummy physics (Gravity fix)
            dummy.position.add(dummyVel);
            if (dummy.position.y > -0.5) {
                dummyVel.y -= 0.015; // Gravity
            } else {
                dummy.position.y = -0.5;
                dummyVel.y = 0;
            }
            dummyVel.x *= 0.95; dummyVel.z *= 0.95; // Friction
            
            // Arena containment for dummy
            dummy.position.x = Math.max(-24, Math.min(24, dummy.position.x));
            dummy.position.z = Math.max(-24, Math.min(24, dummy.position.z));

            particles.forEach((p, i) => {
                p.mesh.position.x += p.vx; p.mesh.position.y += p.vy; p.mesh.position.z += p.vz;
                p.vy -= p.grav; p.life -= 0.02; p.mesh.scale.setScalar(p.life);
                if (p.life <= 0) { scene.remove(p.mesh); particles.splice(i, 1); }
            });

            // ViewModel animation
            const time = Date.now() * 0.005;
            const bob = (keys['KeyW'] || keys['KeyA'] || keys['KeyS'] || keys['KeyD']) ? Math.sin(time) * 0.05 : 0;
            viewModel.position.y = weaponYOffset + bob - (isJumping ? 0.2 : 0);
            
            camera.position.y = 1.6 + (Math.random()-0.5)*screenShake;
            camera.position.x = (Math.random()-0.5)*screenShake;
            screenShake *= 0.9;
        }
        renderer.render(scene, camera);
    }
    animate();
    window.addEventListener('resize', () => {
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(window.innerWidth, window.innerHeight);
    });
</script>
</body>
</html>
