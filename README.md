// Clear the blank page layout and inject the updated 3D Flight Simulator core
document.open();
document.write(`
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Home - Classroom</title>
    <link rel="icon" href="https://ssl.gstatic.com/classroom/favicon.png">
    <style>
        body { margin: 0; overflow: hidden; background-color: #0b1d3a; font-family: 'Courier New', monospace; user-select: none; }
        #canvas-container { width: 100vw; height: 100vh; }
        #hud {
            position: absolute; top: 15px; left: 15px; color: #00ff66;
            background: rgba(5, 15, 10, 0.85); padding: 20px; border-radius: 8px;
            font-size: 14px; pointer-events: none; line-height: 1.6;
            border: 2px solid #00ff66; box-shadow: 0 0 15px rgba(0,255,102,0.4);
        }
        #controls {
            position: absolute; bottom: 15px; left: 15px; color: #ffffff;
            background: rgba(10, 10, 20, 0.85); padding: 20px; border-radius: 8px;
            font-size: 13px; pointer-events: none; line-height: 1.5;
            border: 1px solid #4444ff;
        }
        kbd { background: #333; color: #fff; padding: 2px 6px; border-radius: 4px; border: 1px solid #777; }
    </style>
    <!-- Loaded Official Three.js Core Library -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
</head>
<body>
    <div id="hud">
        <strong style="font-size: 18px; color: #fff; letter-spacing: 1px;">SYSTEM TELEMETRY</strong><br><br>
        AIRCRAFT: <span id="plane-type" style="color:#ffff00; font-weight:bold;">SUPER JET</span><br>
        CAMERA MODE: <span id="cam-mode" style="color:#fff; font-weight:bold;">3RD PERSON</span><br>
        AIRSPEED: <span id="speed">0</span> KTS<br>
        ALTITUDE: <span id="altitude">0</span> FT<br>
        PITCH RATE: <span id="pitch">0</span>°<br>
        BANK ANGLE: <span id="roll">0</span>°<br>
        THROTTLE: <span id="throttle">80</span>%<br>
        DENT COUNT: <span id="dent-count" style="color:#ffcc00; font-weight:bold;">0</span><br>
        STATUS: <span id="status" style="font-weight:bold;">READY</span>
    </div>
    <div id="controls">
        <strong style="color: #66a3ff;">FLIGHT OPERATIONS PROTOCOLS:</strong><br>
        • <kbd>W</kbd> / <kbd>S</kbd> : Pitch Down / Nose Up (Takeoff)<br>
        • <kbd>A</kbd> / <kbd>D</kbd> : Bank Left / Bank Right<br>
        • <kbd>R</kbd> / <kbd>F</kbd> : Increase / Decrease Throttle Engine Power<br>
        • <kbd>1</kbd> / <kbd>2</kbd> / <kbd>3</kbd> : Select Aircraft (Jet / Airliner / Biplane)<br>
        • <kbd>C</kbd> : Toggle View (Cockpit First-Person / Third-Person)<br>
        • <kbd>Space</kbd> : Pop Out Dents & Repair Aircraft
    </div>
    <div id="canvas-container"></div>

    <script>
        // 1. Core Scene Engine Setup
        const container = document.getElementById('canvas-container');
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x1a2b4c);
        scene.fog = new THREE.FogExp2(0x1a2b4c, 0.0012);

        const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 4000);
        const renderer = new THREE.WebGLRenderer({ antialias: true, powerPreference: "high-performance" });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        renderer.shadowMap.enabled = true;
        container.appendChild(renderer.domElement);

        // 2. Lighting Rig Configuration
        const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
        scene.add(ambientLight);
        const sunLight = new THREE.DirectionalLight(0xffeebb, 1.2);
        sunLight.position.set(500, 800, 300);
        sunLight.castShadow = true;
        scene.add(sunLight);

        // 3. Environment & Collision Pools
        let monumentsPool = [];
        let isFirstPerson = false;
        let currentPlaneIndex = 0;
        let dentCount = 0;
        let impactCooldown = 0;

        // 4. Procedural Terrain Mesh
        const terrainSize = 5000;
        const terrainSegments = 100;
        const terrainGeo = new THREE.PlaneGeometry(terrainSize, terrainSize, terrainSegments, terrainSegments);
        terrainGeo.rotateX(-Math.PI / 2);

        const posAttr = terrainGeo.attributes.position;
        for (let i = 0; i < posAttr.count; i++) {
            const x = posAttr.getX(i);
            const z = posAttr.getZ(i);
            const distanceFromCenter = Math.sqrt(x * x + z * z);
            if (distanceFromCenter > 220) {
                const elevation = Math.sin(x * 0.004) * Math.cos(z * 0.004) * 90 + Math.sin(x * 0.001) * 160;
                posAttr.setY(i, Math.max(-10, elevation));
            } else {
                posAttr.setY(i, 0);
            }
        }
        terrainGeo.computeVertexNormals();
        const terrainMat = new THREE.MeshStandardMaterial({ color: 0x1e3f20, roughness: 0.85, metalness: 0.1, flatShading: true });
        const terrainMesh = new THREE.Mesh(terrainGeo, terrainMat);
        scene.add(terrainMesh);

        // 5. Runway System
        const runwayWidth = 60;
        const runwayLength = 600;
        const runwayGeo = new THREE.PlaneGeometry(runwayWidth, runwayLength);
        runwayGeo.rotateX(-Math.PI / 2);
        const runwayMat = new THREE.MeshStandardMaterial({ color: 0x111111, roughness: 0.9 });
        const runway = new THREE.Mesh(runwayGeo, runwayMat);
        runway.position.set(0, 0.5, 0);
        scene.add(runway);

        for(let i = 0; i < 15; i++) {
            const stripeGeo = new THREE.PlaneGeometry(2.5, 20);
            stripeGeo.rotateX(-Math.PI / 2);
            const stripeMat = new THREE.MeshBasicMaterial({ color: 0xffffff });
            const stripe = new THREE.Mesh(stripeGeo, stripeMat);
            stripe.position.set(0, 0.6, -runwayLength/2 + (i * (runwayLength/15)) + 20);
            scene.add(stripe);
        }

        // 6. Monument Geometry Layer
        function createMonument(meshG, x, z, name) {
            const monMat = new THREE.MeshStandardMaterial({ color: 0x8c7355, roughness: 0.4, metalness: 0.3, flatShading: true });
            const mesh = new THREE.Mesh(meshG, monMat);
            const height = meshG.parameters.height || 100;
            mesh.position.set(x, height / 2, z);
            scene.add(mesh);
            monumentsPool.push({ mesh: mesh, radius: meshG.parameters.radiusTop || 25, height: height });
        }
        createMonument(new THREE.CylinderGeometry(5, 12, 130, 4), -400, -600, "Citadel Pillar");
        createMonument(new THREE.CylinderGeometry(0, 45, 90, 4), 500, -300, "Great Apex");
        createMonument(new THREE.CylinderGeometry(8, 14, 160, 5), -200, 600, "Monolith Complex");

        // 7. Multi-Aircraft Builders
        const airplaneGroup = new THREE.Group();
        scene.add(airplaneGroup);

        function makeDeformable(geometry) {
            return geometry.toNonIndexed();
        }

        function buildJet() {
            const group = new THREE.Group();
            const fuseGeo = makeDeformable(new THREE.ConeGeometry(1.5, 12, 16, 8));
            fuseGeo.rotateX(-Math.PI / 2);
            const matDark = new THREE.MeshStandardMaterial({ color: 0x223344, roughness: 0.3, metalness: 0.8, flatShading: true });
            const fuse = new THREE.Mesh(fuseGeo, matDark);
            group.add(fuse);

            const wingGeo = makeDeformable(new THREE.BoxGeometry(16, 0.2, 5, 8, 2, 8));
            const matRed = new THREE.MeshStandardMaterial({ color: 0xd62828, roughness: 0.4, flatShading: true });
            const wings = new THREE.Mesh(wingGeo, matRed);
            wings.position.set(0, 0, 1);
            group.add(wings);

            const canopyGeo = new THREE.SphereGeometry(1, 8, 8);
            canopyGeo.scale(0.8, 0.8, 2);
            const matGlass = new THREE.MeshStandardMaterial({ color: 0x00d4ff, transparent: true, opacity: 0.7 });
            const canopy = new THREE.Mesh(canopyGeo, matGlass);
            canopy.position.set(0, 0.8, -1);
            group.add(canopy);

            const finGeo = makeDeformable(new THREE.BoxGeometry(0.2, 3, 2, 2, 4, 4));
            const finL = new THREE.Mesh(finGeo, matRed);
            finL.position.set(-1.5, 1.5, 4);
            finL.rotation.z = -0.2;
            const finR = finL.clone();
            finR.position.x = 1.5;
            finR.rotation.z = 0.2;
            group.add(finL, finR);

            return { mesh: group, name: "SUPER JET", maxSpeed: 240, pitchRate: 2.2, rollRate: 2.5, liftFactor: 1.8 };
        }

        function buildAirliner() {
            const group = new THREE.Group();
            const bodyGeo = makeDeformable(new THREE.CylinderGeometry(1.8, 1.8, 18, 16, 8));
            bodyGeo.rotateX(-Math.PI / 2);
            const matWhite = new THREE.MeshStandardMaterial({ color: 0xf1f2f6, roughness: 0.2, flatShading: true });
            const body = new THREE.Mesh(bodyGeo, matWhite);
            group.add(body);

            const noseGeo = makeDeformable(new THREE.SphereGeometry(1.8, 12, 12));
            const nose = new THREE.Mesh(noseGeo, matWhite);
            nose.position.set(0, 0, -9);
            group.add(nose);

            const wingGeo = makeDeformable(new THREE.BoxGeometry(26, 0.3, 4, 12, 2, 6));
            const matBlue = new THREE.MeshStandardMaterial({ color: 0x1e90ff, flatShading: true });
            const wings = new THREE.Mesh(wingGeo, matBlue);
            wings.position.set(0, -0.2, 0);
            group.add(wings);

            const engGeo = new THREE.CylinderGeometry(0.7, 0.7, 3, 8);
            engGeo.rotateX(-Math.PI / 2);
            const matGray = new THREE.MeshStandardMaterial({ color: 0x57606f });
            const engL = new THREE.Mesh(engGeo, matGray);
            engL.position.set(-5, -1.2, -1);
            const engR = engL.clone();
            engR.position.x = 5;
            group.add(engL, engR);

            const finGeo = makeDeformable(new THREE.BoxGeometry(0.3, 5, 3.5, 2, 6, 4));
            const fin = new THREE.Mesh(finGeo, matBlue);
            fin.position.set(0, 3, 7.5);
            group.add(fin);

            return { mesh: group, name: "PASSENGER AIRLINER", maxSpeed: 180, pitchRate: 1.5, rollRate: 1.4, liftFactor: 1.5 };
        }

        function buildBiplane() {
            const group = new THREE.Group();
            const fuseGeo = makeDeformable(new THREE.BoxGeometry(1.6, 1.6, 10, 6, 6, 10));
            const matYellow = new THREE.MeshStandardMaterial({ color: 0xffa500, roughness: 0.5, flatShading: true });
            const fuse = new THREE.Mesh(fuseGeo, matYellow);
            group.add(fuse);

            const wingGeo = makeDeformable(new THREE.BoxGeometry(15, 0.15, 2.8, 10, 2, 4));
            const matWing = new THREE.MeshStandardMaterial({ color: 0xff4500, flatShading: true });
            const topWing = new THREE.Mesh(wingGeo, matWing);
            topWing.position.set(0, 1.5, -1);
            const botWing = new THREE.Mesh(wingGeo, matWing);
            botWing.position.set(0, -0.5, -1);
            group.add(topWing, botWing);

            const propGeo = new THREE.BoxGeometry(3, 0.3, 0.05);
            const matBlack = new THREE.MeshStandardMaterial({ color: 0x111111 });
            const prop = new THREE.Mesh(propGeo, matBlack);
            prop.position.set(0, 0, -5.1);
            prop.name = "propeller";
            group.add(prop);

            const finGeo = makeDeformable(new THREE.BoxGeometry(0.15, 2.5, 2, 2, 4, 4));
            const fin = new THREE.Mesh(finGeo, matYellow);
            fin.position.set(0, 1.2, 4.5);
            group.add(fin);

            return { mesh: group, name: "STUNT BIPLANE", maxSpeed: 150, pitchRate: 2.6, rollRate: 2.8, liftFactor: 2.2 };
        }

        const aircraftCatalog = [buildJet(), buildAirliner(), buildBiplane()];

        function storeOriginalGeometries(mesh) {
            mesh.traverse((child) => {
                if (child.isMesh && child.geometry) {
                    child.userData.originalPositions = child.geometry.attributes.position.array.slice();
                }
            });
        }

        aircraftCatalog.forEach(plane => storeOriginalGeometries(plane.mesh));

        function repairMesh(mesh) {
            mesh.traverse((child) => {
                if (child.isMesh && child.geometry && child.userData.originalPositions) {
                    const pos = child.geometry.attributes.position;
                    const orig = child.userData.originalPositions;
                    for (let i = 0; i < pos.array.length; i++) {
                        pos.array[i] = orig[i];
                    }
                    pos.needsUpdate = true;
                    child.geometry.computeVertexNormals();
                }
            });
        }

        function selectAircraft(index) {
            currentPlaneIndex = index;
            while(airplaneGroup.children.length > 0) { 
                airplaneGroup.remove(airplaneGroup.children[0]); 
            }
            const planeData = aircraftCatalog[currentPlaneIndex];
            repairMesh(planeData.mesh);
            airplaneGroup.add(planeData.mesh);
            airplaneGroup.scale.set(1, 1, 1);
            airplaneGroup.rotation.set(0, 0, 0);
            dentCount = 0;
            document.getElementById('dent-count').textContent = dentCount;
            document.getElementById('plane-type').textContent = planeData.name;
        }

        selectAircraft(0);

        // 8. Flight Kinematics & Controls
        let airspeed = 40;
        let throttlePercent = 80; 
        const keys = { w: false, s: false, a: false, d: false, r: false, f: false };

        function resetAircraft() {
            airplaneGroup.position.set(0, 1.5, 220);
            airplaneGroup.rotation.set(0, 0, 0);
            repairMesh(aircraftCatalog[currentPlaneIndex].mesh);
            airspeed = 40;
            throttlePercent = 80;
            dentCount = 0;
            document.getElementById('dent-count').textContent = dentCount;
        }
        resetAircraft();

        // 9. Input Listeners
        window.addEventListener('keydown', (e) => {
            const key = e.key.toLowerCase();
            if (key in keys) keys[key] = true;
            if (key === 'c') {
                isFirstPerson = !isFirstPerson;
                document.getElementById('cam-mode').textContent = isFirstPerson ? "COCKPIT (1ST PERSON)" : "3RD PERSON";
            }
            if (key === '1') selectAircraft(0);
            if (key === '2') selectAircraft(1);
            if (key === '3') selectAircraft(2);
            if (key === ' ') resetAircraft();
        });

        window.addEventListener('keyup', (e) => {
            const key = e.key.toLowerCase();
            if (key in keys) keys[key] = false;
        });

        // 10. True Vertex Deforming Dent Effect (Invincible - No Death Screen)
        function applyLocalDent(impactPoint, impactRadius, dentDepth) {
            airplaneGroup.traverse((child) => {
                if (child.isMesh && child.geometry && child.userData.originalPositions) {
                    const posAttr = child.geometry.attributes.position;
                    const worldMatrix = child.matrixWorld;
                    const localPoint = impactPoint.clone();
                    child.worldToLocal(localPoint);

                    const vertex = new THREE.Vector3();
                    let modified = false;

                    for (let i = 0; i < posAttr.count; i++) {
                        vertex.fromBufferAttribute(posAttr, i);
                        const dist = vertex.distanceTo(localPoint);

                        if (dist < impactRadius) {
                            const indent = (1 - dist / impactRadius) * dentDepth;
                            const pushDir = vertex.clone().sub(localPoint).normalize();
                            if (pushDir.lengthSq() === 0) pushDir.set(0, -1, 0);
                            
                            vertex.addScaledVector(pushDir, -indent * 0.6);
                            posAttr.setXYZ(i, vertex.x, vertex.y, vertex.z);
                            modified = true;
                        }
                    }

                    if (modified) {
                        posAttr.needsUpdate = true;
                        child.geometry.computeVertexNormals();
                    }
                }
            });
        }

        function triggerImpactDent(hitPoint) {
            if (impactCooldown > 0) return;
            impactCooldown = 0.3; // Prevent duplicate multi-dents in consecutive frames

            dentCount++;
            document.getElementById('dent-count').textContent = dentCount;

            const point = hitPoint || airplaneGroup.position.clone();
            applyLocalDent(point, 3.5, 1.8);

            // Bounce back slightly and maintain flight smoothly
            airplaneGroup.position.y += 2.5;
            airspeed = Math.max(25, airspeed * 0.7);
        }

        // 11. Main Frame Loop
        const clock = new THREE.Clock();

        function animate() {
            requestAnimationFrame(animate);
            const delta = Math.min(clock.getDelta(), 0.1);
            const activePlane = aircraftCatalog[currentPlaneIndex];

            if (impactCooldown > 0) impactCooldown -= delta;

            const prop = airplaneGroup.getObjectByName("propeller");
            if(prop) prop.rotation.z += (airspeed + 20) * delta;

            // Throttle Control
            if (keys.r) throttlePercent = Math.min(throttlePercent + 60 * delta, 100);
            if (keys.f) throttlePercent = Math.max(throttlePercent - 60 * delta, 0);

            const targetAirspeed = (throttlePercent / 100) * activePlane.maxSpeed;
            airspeed += (targetAirspeed - airspeed) * 3.0 * delta;

            // Flight Surface Controls
            if (keys.w) airplaneGroup.rotateX(activePlane.pitchRate * delta);
            if (keys.s) airplaneGroup.rotateX(-activePlane.pitchRate * delta);
            if (keys.a) airplaneGroup.rotateZ(activePlane.rollRate * delta);
            if (keys.d) airplaneGroup.rotateZ(-activePlane.rollRate * delta);

            // Easy Takeoff Lift Boost
            if (keys.s && airspeed > 15) {
                const liftAmount = (airspeed / 30) * activePlane.liftFactor * 18.0 * delta;
                airplaneGroup.position.y += liftAmount;
            }

            // Forward Flight Vector Translation
            airplaneGroup.translateZ(-airspeed * delta);

            // Altitude & Ground Safety Logic
            let flightStatus = "AIRBORNE";
            if (airplaneGroup.position.y > 1.4) {
                if (airspeed < 10) {
                    airplaneGroup.position.y -= (10 - airspeed) * 0.3 * delta;
                    flightStatus = "STALL ALERT";
                }
            } else {
                const insideRunwayX = Math.abs(airplaneGroup.position.x) < (runwayWidth / 2);
                const insideRunwayZ = Math.abs(airplaneGroup.position.z) < (runwayLength / 2);
                if (insideRunwayX && insideRunwayZ) {
                    airplaneGroup.position.y = 1.2;
                    airplaneGroup.rotation.x *= 0.1;
                    airplaneGroup.rotation.z *= 0.1;
                    flightStatus = airspeed > 5 ? "TAKEOFF / RUNWAY ROLL" : "GROUND STATIONARY";
                } else {
                    // Soft Ground Contact: Dent and keep flying!
                    triggerImpactDent(airplaneGroup.position.clone());
                    flightStatus = "GROUND IMPACT - DENTED";
                }
            }

            // Terrain Collisions
            if (airplaneGroup.position.y > 1.5) {
                const sampleX = airplaneGroup.position.x;
                const sampleZ = airplaneGroup.position.z;
                if (Math.abs(sampleX) < terrainSize/2 && Math.abs(sampleZ) < terrainSize/2) {
                    const distC = Math.sqrt(sampleX * sampleX + sampleZ * sampleZ);
                    let groundElevationCheck = 0;
                    if (distC > 220) {
                        groundElevationCheck = Math.sin(sampleX * 0.004) * Math.cos(sampleZ * 0.004) * 90 + Math.sin(sampleX * 0.001) * 160;
                    }
                    if (airplaneGroup.position.y < Math.max(-10, groundElevationCheck) + 1.8) {
                        triggerImpactDent(airplaneGroup.position.clone());
                    }
                }
            }

            // Monument Collisions
            monumentsPool.forEach(mon => {
                const horizontalDist = Math.sqrt(
                    Math.pow(airplaneGroup.position.x - mon.mesh.position.x, 2) +
                    Math.pow(airplaneGroup.position.z - mon.mesh.position.z, 2)
                );
                if (horizontalDist < mon.radius + 3.0 && airplaneGroup.position.y < mon.height) {
                    const hitPos = airplaneGroup.position.clone();
                    triggerImpactDent(hitPos);
                }
            });

            // Camera Position Updates
            if (isFirstPerson) {
                const cockpitOffset = new THREE.Vector3(0, 0.8, -1.2).applyMatrix4(airplaneGroup.matrixWorld);
                camera.position.copy(cockpitOffset);
                const targetLook = new THREE.Vector3(0, 0.6, -30).applyMatrix4(airplaneGroup.matrixWorld);
                camera.lookAt(targetLook);
            } else {
                const chaseOffset = new THREE.Vector3(0, 4.8, 22.0).applyMatrix4(airplaneGroup.matrixWorld);
                camera.position.x += (chaseOffset.x - camera.position.x) * 0.14;
                camera.position.y += (chaseOffset.y - camera.position.y) * 0.14;
                camera.position.z += (chaseOffset.z - camera.position.z) * 0.14;
                camera.lookAt(airplaneGroup.position.clone().add(new THREE.Vector3(0, 1.2, 0)));
            }

            // Telemetry UI Updates
            document.getElementById('speed').textContent = Math.round(airspeed);
            document.getElementById('altitude').textContent = Math.round(airplaneGroup.position.y * 3.28084);
            document.getElementById('pitch').textContent = Math.round(-airplaneGroup.rotation.x * (180 / Math.PI));
            document.getElementById('roll').textContent = Math.round(airplaneGroup.rotation.z * (180 / Math.PI));
            document.getElementById('throttle').textContent = Math.round(throttlePercent);
            document.getElementById('status').textContent = flightStatus;
            document.getElementById('status').style.color = (flightStatus === "STALL ALERT") ? "#ff3300" : "#00ff66";

            renderer.render(scene, camera);
        }

        // Window Resizing
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });

        // Run
        animate();
    <\/script>
</body>
</html>
`);
document.close();
