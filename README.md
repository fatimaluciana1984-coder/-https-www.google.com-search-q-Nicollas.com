<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Minecraft Ursina Style - Nicollas</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <style>
        body { margin: 0; overflow: hidden; background-color: #000; touch-action: none; }
        canvas { display: block; }
        
        /* Interface Crosshair */
        #crosshair {
            position: absolute;
            top: 50%; left: 50%;
            width: 20px; height: 20px;
            border: 2px solid rgba(255,255,255,0.8);
            transform: translate(-50%, -50%);
            pointer-events: none;
        }

        /* Controles UI */
        #ui-container {
            position: absolute;
            top: 0; left: 0;
            width: 100%; height: 100%;
            pointer-events: none;
            font-family: Arial, sans-serif;
        }

        .joystick-base {
            position: absolute;
            bottom: 50px; left: 50px;
            width: 120px; height: 120px;
            background: rgba(0,0,0,0.33);
            border-radius: 50%;
            pointer-events: auto;
        }
        .joystick-dot {
            position: absolute;
            top: 50%; left: 50%;
            width: 40px; height: 40px;
            background: white;
            border-radius: 50%;
            transform: translate(-50%, -50%);
        }

        .jump-btn {
            position: absolute;
            bottom: 60px; right: 60px;
            width: 90px; height: 90px;
            background: rgba(0,0,0,0.66);
            border: 2px solid white;
            color: white;
            border-radius: 15px;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            pointer-events: auto;
            cursor: pointer;
        }

        .hotbar {
            position: absolute;
            bottom: 20px; left: 50%;
            transform: translateX(-50%);
            width: 300px; height: 50px;
            background: #333;
            border: 3px solid #111;
            pointer-events: auto;
        }
    </style>
</head>
<body>

    <div id="crosshair"></div>
    <div id="ui-container">
        <div class="joystick-base"><div class="joystick-dot"></div></div>
        <div class="jump-btn" id="jumpBtn">
            <span style="font-size: 30px; font-weight: bold;">^</span>
            <span style="font-size: 10px;">PULAR</span>
        </div>
        <div class="hotbar"></div>
    </div>

    <script>
        // --- SETUP ---
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x87CEEB);
        
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.body.appendChild(renderer.domElement);

        // Luzes
        const light = new THREE.AmbientLight(0xffffff, 0.8);
        scene.add(light);
        const dirLight = new THREE.DirectionalLight(0xffffff, 0.5);
        dirLight.position.set(10, 20, 10);
        scene.add(dirLight);

        // --- MATERIAIS ---
        const grassMat = new THREE.MeshLambertMaterial({ color: 0x7cfc00 });
        const woodMat = new THREE.MeshLambertMaterial({ color: 0x8b4513 });
        const boxGeo = new THREE.BoxGeometry(1, 1, 1);

        // --- JOGADOR E MÃO ---
        const player = new THREE.Group();
        player.position.set(7, 2, 7);
        scene.add(player);
        player.add(camera);

        const hand = new THREE.Mesh(
            new THREE.BoxGeometry(0.2, 0.3, 0.5),
            new THREE.MeshLambertMaterial({ color: 0xffa500 }) // Cor Laranja do seu código
        );
        hand.position.set(0.5, -0.4, -0.6);
        camera.add(hand);

        // --- MUNDO ---
        const voxels = [];
        function addVoxel(x, y, z, mat) {
            const mesh = new THREE.Mesh(boxGeo, mat);
            mesh.position.set(x, y, z);
            scene.add(mesh);
            voxels.push(mesh);
        }

        // Chão 15x15
        for(let x = 0; x < 15; x++) {
            for(let z = 0; z < 15; z++) {
                addVoxel(x, 0, z, grassMat);
            }
        }

        // Árvores
        for(let i = 0; i < 4; i++) {
            let tx = Math.floor(Math.random() * 10) + 2;
            let tz = Math.floor(Math.random() * 10) + 2;
            for(let h = 1; h <= 3; h++) {
                addVoxel(tx, h, tz, woodMat);
            }
        }

        // --- LÓGICA DE JOGO ---
        let velocity = new THREE.Vector3();
        let moveForward = false, moveBackward = false, moveLeft = false, moveRight = false;
        let isJumping = false;
        let clock = new THREE.Clock();

        // Clique e Interação
        const raycaster = new THREE.Raycaster();
        const pointer = new THREE.Vector2(0, 0);

        function handleInteraction(event) {
            if (document.pointerLockElement !== renderer.domElement) {
                renderer.domElement.requestPointerLock();
                return;
            }

            // Movimento de Soco (igual ao seu update)
            hand.position.set(0.4, -0.3, -0.4);
            setTimeout(() => hand.position.set(0.5, -0.4, -0.6), 100);

            raycaster.setFromCamera(pointer, camera);
            const intersects = raycaster.intersectObjects(voxels);

            if (intersects.length > 0) {
                const intersect = intersects[0];
                if (event.button === 0 && !event.shiftKey) { // Botão Esquerdo: Quebrar
                    scene.remove(intersect.object);
                    voxels.splice(voxels.indexOf(intersect.object), 1);
                } else { // Botão Direito (ou Shift+Clique): Colocar
                    const pos = intersect.object.position.clone().add(intersect.face.normal);
                    addVoxel(pos.x, pos.y, pos.z, grassMat);
                }
            }
        }

        window.addEventListener('mousedown', handleInteraction);

        // Controles Teclado
        window.addEventListener('keydown', (e) => {
            if(e.code === 'KeyW') moveForward = true;
            if(e.code === 'KeyS') moveBackward = true;
            if(e.code === 'KeyA') moveLeft = true;
            if(e.code === 'KeyD') moveRight = true;
            if(e.code === 'Space' && !isJumping) { velocity.y = 5; isJumping = true; }
        });
        window.addEventListener('keyup', (e) => {
            if(e.code === 'KeyW') moveForward = false;
            if(e.code === 'KeyS') moveBackward = false;
            if(e.code === 'KeyA') moveLeft = false;
            if(e.code === 'KeyD') moveRight = false;
        });

        // Mouse Look
        window.addEventListener('mousemove', (e) => {
            if (document.pointerLockElement === renderer.domElement) {
                player.rotation.y -= e.movementX * 0.002;
                camera.rotation.x -= e.movementY * 0.002;
                camera.rotation.x = Math.max(-Math.PI/2, Math.PI/2, camera.rotation.x);
            }
        });

        // Botão Pular Mobile
        document.getElementById('jumpBtn').addEventListener('pointerdown', () => {
            if(!isJumping) { velocity.y = 5; isJumping = true; }
        });

        // --- LOOP PRINCIPAL ---
        function animate() {
            requestAnimationFrame(animate);
            const delta = clock.getDelta();
            const time = clock.getElapsedTime();

            // Gravidade e Pulo
            velocity.y -= 9.8 * delta;
            player.position.y += velocity.y * delta;

            if (player.position.y < 2) {
                player.position.y = 2;
                velocity.y = 0;
                isJumping = false;
            }

            // Movimento simples
            const speed = 5;
            if (moveForward) player.translateZ(-speed * delta);
            if (moveBackward) player.translateZ(speed * delta);
            if (moveLeft) player.translateX(-speed * delta);
            if (moveRight) player.translateX(speed * delta);

            // Balanço da Mão (Breathe effect igual ao seu código Ursina)
            // hand.y = -0.4 + (sin(time * 5) * 0.02)
            hand.position.y = -0.4 + (Math.sin(time * 5) * 0.02);

            renderer.render(scene, camera);
        }

        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });

        animate();
    </script>
</body>
</html>

