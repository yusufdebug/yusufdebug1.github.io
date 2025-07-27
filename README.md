<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Night Driver - Tek Dosya</title>
    
    <style>
        /* --- public/style.css --- */
        body {
            margin: 0;
            overflow: hidden;
            background-color: #000;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            color: #fff;
        }

        #gameCanvas {
            display: block;
            width: 100vw;
            height: 100vh;
        }

        #loadingScreen {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background-color: rgba(0, 0, 0, 0.9);
            display: flex;
            justify-content: center;
            align-items: center;
            font-size: 3em;
            font-weight: bold;
            z-index: 1000;
            transition: opacity 1s ease-out;
        }

        #loadingScreen.hidden {
            opacity: 0;
            pointer-events: none;
        }

        #uiOverlay {
            position: absolute;
            top: 10px;
            left: 10px;
            display: flex;
            flex-direction: column;
            gap: 5px;
            background-color: rgba(0, 0, 0, 0.5);
            padding: 10px 15px;
            border-radius: 8px;
            font-size: 1.2em;
            z-index: 500;
        }

        #speedometer, #distanceCounter {
            text-shadow: 0 0 5px rgba(255, 255, 255, 0.7);
        }

        #speedValue, #distanceValue {
            font-weight: bold;
            color: #00ff00;
        }
    </style>
</head>
<body>
    <canvas id="gameCanvas"></canvas>
    <div id="loadingScreen">Night Driver Yükleniyor...</div>
    <div id="uiOverlay">
        <div id="speedometer">Hız: <span id="speedValue">0</span> km/s</div>
        <div id="distanceCounter">Mesafe: <span id="distanceValue">0</span> m</div>
    </div>
    
    <script async src="https://unpkg.com/three@0.165.0/build/three.min.js"></script>

    <script>
        // Tüm JavaScript kodu burada olacak
        // Global THREE objesine erişim için, Three.js yüklendikten sonra kodun çalışmasını beklemeliyiz.

        // --- 1. Constants ---
        const Constants = {
            // Genel
            GRAVITY: -9.81,
            FPS: 60,
            M_S_TO_KM_H: 3.6,

            // Kamera
            FOV: 75,
            NEAR_PLANE: 0.1,
            FAR_PLANE: 1000,
            CAMERA_OFFSET_X: 0,
            CAMERA_OFFSET_Y: 8,
            CAMERA_OFFSET_Z: -18,
            CAMERA_LOOKAT_OFFSET_Y: 2,
            CAMERA_LOOKAT_OFFSET_Z: 5,
            CAMERA_SMOOTHING: 0.08,

            // Sahne
            FOG_COLOR: 0x000000,
            FOG_NEAR: 100,
            FOG_FAR: 400,
            CLEAR_COLOR: 0x000000,

            // Araba Fizik
            CAR_ACCELERATION: 40, // m/s^2
            CAR_BRAKE_DECELERATION: 60,
            CAR_DRAG_COEFFICIENT: 0.005, // Hava direnci katsayısı
            CAR_MAX_SPEED: 70, // m/s (252 km/h)
            CAR_REVERSE_SPEED: 10,
            CAR_TURN_SPEED: 0.02, // Radyan/s
            CAR_TURN_DECAY: 0.9, // Dönüş sonrası toparlanma
            CAR_MASS: 1200, // kg

            // Araba Modeli
            CAR_BODY_COLOR: 0x1111aa,
            CAR_WHEEL_COLOR: 0x333333,
            CAR_BODY_WIDTH: 2.0,
            CAR_BODY_HEIGHT: 1.2,
            CAR_BODY_LENGTH: 4.5,
            CAR_WHEEL_RADIUS: 0.5,
            CAR_WHEEL_THICKNESS: 0.3,
            CAR_WHEEL_FRONT_OFFSET: 1.5,
            CAR_WHEEL_SIDE_OFFSET: 1.1,

            // Farlar
            HEADLIGHT_INTENSITY: 50,
            HEADLIGHT_DISTANCE: 150,
            HEADLIGHT_ANGLE: Math.PI * 0.15,
            HEADLIGHT_PENUMBRA: 0.6,
            HEADLIGHT_DECAY: 2,
            HEADLIGHT_OFFSET_X: 0.7,
            HEADLIGHT_OFFSET_Y: 0.6,
            HEADLIGHT_OFFSET_Z: 2.2,
            HEADLIGHT_TARGET_OFFSET_Z: 50,

            // Yol
            ROAD_WIDTH: 15,
            ROAD_SEGMENT_LENGTH: 50,
            ROAD_VISIBLE_SEGMENTS: 25, // Önde kaç segment görünsün
            ROAD_CLEANUP_THRESHOLD: 5, // Arkada kaç segment silinsin
            ROAD_COLOR: 0x222222,
            ROAD_LINE_COLOR: 0xffff00,
            ROAD_LINE_WIDTH: 0.1,
            ROAD_DASH_LENGTH: 3,
            ROAD_DASH_GAP: 5,
            ROAD_SIDEWALK_WIDTH: 3,
            ROAD_SIDEWALK_HEIGHT: 0.5,
            ROAD_SIDEWALK_COLOR: 0x555555,

            // Yol Oluşturma (Prosedürel)
            CURVE_INTENSITY: 0.0005, // Virajların sıklığı ve keskinliği
            CURVE_CHANGE_INTERVAL: 5, // Kaç segmentte bir viraj yönü değişsin (random)
            MAX_CURVE_ANGLE: Math.PI / 8, // Maksimum viraj açısı

            // Ortam
            AMBIENT_LIGHT_COLOR: 0x404040,
            AMBIENT_LIGHT_INTENSITY: 0.8,
            DIRECTIONAL_LIGHT_COLOR: 0xaaaaaa,
            DIRECTIONAL_LIGHT_INTENSITY: 0.1, // Düşük, gece olduğu için
            DIRECTIONAL_LIGHT_POSITION_Y: 50,
            DIRECTIONAL_LIGHT_POSITION_Z: -100, // Aracın arkasında olsun
            DIRECTIONAL_LIGHT_SHADOW_MAP_SIZE: 2048,
            DIRECTIONAL_LIGHT_SHADOW_CAMERA_SIZE: 100,

            // Skybox
            SKYBOX_SIZE: 1000,
            SKYBOX_COLOR_TOP: 0x010101,
            SKYBOX_COLOR_BOTTOM: 0x000000,

            // Parçacık Sistemi (Basit)
            SPARK_COUNT: 50,
            SPARK_SIZE: 0.1,
            SPARK_LIFESPAN: 0.5,
            SPARK_COLOR: 0xffa500,
            SPARK_VELOCITY_SCALE: 5,
        };

        // --- 2. Utils ---
        class Utils {
            static lerp(a, b, t) {
                return a + (b - a) * t;
            }

            static smoothStep(edge0, edge1, x) {
                x = Math.max(0, Math.min(1, (x - edge0) / (edge1 - edge0)));
                return x * x * (3 - 2 * x);
            }

            static randomRange(min, max) {
                return Math.random() * (max - min) + min;
            }

            static randomInt(min, max) {
                return Math.floor(Math.random() * (max - min + 1)) + min;
            }

            static createLine(points, material) {
                const geometry = new THREE.BufferGeometry().setFromPoints(points);
                return new THREE.Line(geometry, material);
            }

            static createDashedLine(points, material, dashSize, gapSize) {
                const geometry = new THREE.BufferGeometry().setFromPoints(points);
                const dashedMaterial = new THREE.LineDashedMaterial({
                    color: material.color,
                    linewidth: material.linewidth,
                    scale: 1,
                    dashSize: dashSize,
                    gapSize: gapSize,
                });
                const line = new THREE.LineSegments(geometry, dashedMaterial);
                line.computeLineDistances();
                return line;
            }

            static degreesToRadians(degrees) {
                return degrees * Math.PI / 180;
            }

            static radiansToDegrees(radians) {
                return radians * 180 / Math.PI;
            }

            static checkCollision(obj1, obj2) {
                obj1.geometry.computeBoundingBox();
                obj2.geometry.computeBoundingBox();

                const box1 = new THREE.Box3().setFromObject(obj1);
                const box2 = new THREE.Box3().setFromObject(obj2);

                return box1.intersectsBox(box2);
            }
        }

        // --- 3. InputHandler ---
        class InputHandler {
            constructor() {
                this.keys = {};
                this.addEventListeners();
            }

            addEventListeners() {
                document.addEventListener('keydown', (event) => {
                    this.keys[event.code] = true;
                });
                document.addEventListener('keyup', (event) => {
                    this.keys[event.code] = false;
                });
            }

            isKeyDown(keyCode) {
                return this.keys[keyCode] || false;
            }

            areKeysDown(keyCodes) {
                return keyCodes.every(keyCode => this.keys[keyCode]);
            }
        }

        // --- 4. Car ---
        class Car {
            constructor(scene) {
                this.scene = scene;
                this.mesh = new THREE.Group();

                this.speed = 0; // m/s
                this.distanceDriven = 0; // Metre
                this.velocity = new THREE.Vector3(0, 0, 0);
                this.angularVelocityY = 0;

                this.createCarModel();
                this.addHeadlights();

                this.mesh.position.set(0, 0, 0);
                this.scene.add(this.mesh);

                this.lastPositionZ = this.mesh.position.z;
            }

            createCarModel() {
                const bodyGeometry = new THREE.BoxGeometry(Constants.CAR_BODY_WIDTH, Constants.CAR_BODY_HEIGHT, Constants.CAR_BODY_LENGTH);
                const bodyMaterial = new THREE.MeshPhongMaterial({ color: Constants.CAR_BODY_COLOR });
                const bodyMesh = new THREE.Mesh(bodyGeometry, bodyMaterial);
                bodyMesh.position.y = Constants.CAR_BODY_HEIGHT / 2;
                bodyMesh.castShadow = true;
                this.mesh.add(bodyMesh);

                const wheelGeometry = new THREE.CylinderGeometry(Constants.CAR_WHEEL_RADIUS, Constants.CAR_WHEEL_RADIUS, Constants.CAR_WHEEL_THICKNESS, 16);
                const wheelMaterial = new THREE.MeshPhongMaterial({ color: Constants.CAR_WHEEL_COLOR });

                const wheelPositions = [
                    new THREE.Vector3(Constants.CAR_WHEEL_SIDE_OFFSET, Constants.CAR_WHEEL_RADIUS, Constants.CAR_WHEEL_FRONT_OFFSET),
                    new THREE.Vector3(-Constants.CAR_WHEEL_SIDE_OFFSET, Constants.CAR_WHEEL_RADIUS, Constants.CAR_WHEEL_FRONT_OFFSET),
                    new THREE.Vector3(Constants.CAR_WHEEL_SIDE_OFFSET, Constants.CAR_WHEEL_RADIUS, -Constants.CAR_WHEEL_FRONT_OFFSET),
                    new THREE.Vector3(-Constants.CAR_WHEEL_SIDE_OFFSET, Constants.CAR_WHEEL_RADIUS, -Constants.CAR_WHEEL_FRONT_OFFSET)
                ];

                wheelPositions.forEach(pos => {
                    const wheel = new THREE.Mesh(wheelGeometry, wheelMaterial);
                    wheel.position.copy(pos);
                    wheel.rotation.x = Math.PI / 2;
                    wheel.castShadow = true;
                    this.mesh.add(wheel);
                });

                const cockpitGeometry = new THREE.BoxGeometry(Constants.CAR_BODY_WIDTH * 0.8, Constants.CAR_BODY_HEIGHT * 0.6, Constants.CAR_BODY_LENGTH * 0.4);
                const cockpitMaterial = new THREE.MeshPhongMaterial({ color: 0x333333, specular: 0x555555, shininess: 30 });
                const cockpitMesh = new THREE.Mesh(cockpitGeometry, cockpitMaterial);
                cockpitMesh.position.set(0, Constants.CAR_BODY_HEIGHT * 0.8, Constants.CAR_BODY_LENGTH * 0.1);
                cockpitMesh.castShadow = true;
                this.mesh.add(cockpitMesh);
            }

            addHeadlights() {
                this.headlight1 = new THREE.SpotLight(0xffffff, Constants.HEADLIGHT_INTENSITY, Constants.HEADLIGHT_DISTANCE, Constants.HEADLIGHT_ANGLE, Constants.HEADLIGHT_PENUMBRA, Constants.HEADLIGHT_DECAY);
                this.headlight1.position.set(Constants.HEADLIGHT_OFFSET_X, Constants.HEADLIGHT_OFFSET_Y, Constants.HEADLIGHT_OFFSET_Z);
                this.headlight1.castShadow = true;
                this.headlight1.shadow.mapSize.width = 1024;
                this.headlight1.shadow.mapSize.height = 1024;
                this.headlight1.shadow.camera.near = 1;
                this.headlight1.shadow.camera.far = Constants.HEADLIGHT_DISTANCE;
                this.headlight1.shadow.bias = -0.0001;
                this.mesh.add(this.headlight1);
                this.headlightTarget1 = new THREE.Object3D();
                this.headlightTarget1.position.set(Constants.HEADLIGHT_OFFSET_X, Constants.HEADLIGHT_OFFSET_Y, Constants.HEADLIGHT_OFFSET_Z + Constants.HEADLIGHT_TARGET_OFFSET_Z);
                this.mesh.add(this.headlightTarget1);
                this.headlight1.target = this.headlightTarget1;

                this.headlight2 = new THREE.SpotLight(0xffffff, Constants.HEADLIGHT_INTENSITY, Constants.HEADLIGHT_DISTANCE, Constants.HEADLIGHT_ANGLE, Constants.HEADLIGHT_PENUMBRA, Constants.HEADLIGHT_DECAY);
                this.headlight2.position.set(-Constants.HEADLIGHT_OFFSET_X, Constants.HEADLIGHT_OFFSET_Y, Constants.HEADLIGHT_OFFSET_Z);
                this.headlight2.castShadow = true;
                this.headlight2.shadow.mapSize.width = 1024;
                this.headlight2.shadow.mapSize.height = 1024;
                this.headlight2.shadow.camera.near = 1;
                this.headlight2.shadow.camera.far = Constants.HEADLIGHT_DISTANCE;
                this.headlight2.shadow.bias = -0.0001;
                this.mesh.add(this.headlight2);
                this.headlightTarget2 = new THREE.Object3D();
                this.headlightTarget2.position.set(-Constants.HEADLIGHT_OFFSET_X, Constants.HEADLIGHT_OFFSET_Y, Constants.HEADLIGHT_OFFSET_Z + Constants.HEADLIGHT_TARGET_OFFSET_Z);
                this.mesh.add(this.headlightTarget2);
                this.headlight2.target = this.headlightTarget2;
            }

            update(deltaTime, inputHandler) {
                let accelerationForce = 0;
                if (inputHandler.isKeyDown('ArrowUp')) {
                    accelerationForce = Constants.CAR_ACCELERATION;
                } else if (inputHandler.isKeyDown('ArrowDown')) {
                    accelerationForce = -Constants.CAR_BRAKE_DECELERATION;
                    if (this.speed > 0) {
                        this.speed = Math.max(0, this.speed - Constants.CAR_BRAKE_DECELERATION * deltaTime);
                    } else {
                        this.speed = Math.max(-Constants.CAR_REVERSE_SPEED, this.speed + Constants.CAR_BRAKE_DECELERATION * deltaTime * 0.5);
                    }
                } else {
                    if (this.speed > 0) {
                        this.speed -= Constants.CAR_DRAG_COEFFICIENT * this.speed * this.speed * deltaTime;
                        this.speed = Math.max(0, this.speed);
                    } else if (this.speed < 0) {
                        this.speed += Constants.CAR_DRAG_COEFFICIENT * this.speed * this.speed * deltaTime;
                        this.speed = Math.min(0, this.speed);
                    }
                }

                this.speed += accelerationForce * deltaTime;
                this.speed = Math.max(-Constants.CAR_REVERSE_SPEED, Math.min(this.speed, Constants.CAR_MAX_SPEED));

                let turnAmount = 0;
                if (inputHandler.isKeyDown('ArrowLeft')) {
                    turnAmount = Constants.CAR_TURN_SPEED;
                } else if (inputHandler.isKeyDown('ArrowRight')) {
                    turnAmount = -Constants.CAR_TURN_SPEED;
                }

                if (this.speed !== 0) {
                    const turnFactor = Math.min(1, Math.abs(this.speed) / (Constants.CAR_MAX_SPEED / 2));
                    this.mesh.rotation.y += turnAmount * deltaTime * turnFactor;
                }

                const forwardVector = new THREE.Vector3(0, 0, 1);
                forwardVector.applyQuaternion(this.mesh.quaternion);

                this.velocity.copy(forwardVector).multiplyScalar(this.speed);

                this.mesh.position.add(this.velocity.clone().multiplyScalar(deltaTime));

                if (this.speed > 0) {
                    this.distanceDriven += this.speed * deltaTime;
                }

                this.headlight1.target.updateMatrixWorld();
                this.headlight2.target.updateMatrixWorld();

                this.updateWheelRotation(deltaTime);
            }

            updateWheelRotation(deltaTime) {
                const wheelRotationSpeed = (this.speed / (2 * Math.PI * Constants.CAR_WHEEL_RADIUS)) * 2 * Math.PI;
                this.mesh.children.forEach(child => {
                    if (child instanceof THREE.Mesh && child.geometry.type === 'CylinderGeometry') {
                        child.rotation.z += wheelRotationSpeed * deltaTime;
                    }
                });
            }

            getPosition() {
                return this.mesh.position;
            }
        }

        // --- 5. RoadGenerator ---
        class RoadGenerator {
            constructor(scene) {
                this.scene = scene;
                this.roadSegments = [];
                this.lastZPosition = -Constants.ROAD_SEGMENT_LENGTH;
                this.currentCurveAngle = 0;
                this.targetCurveAngle = 0;
                this.curveSegmentCounter = 0;

                this.roadMaterial = new THREE.MeshPhongMaterial({ color: Constants.ROAD_COLOR, side: THREE.DoubleSide, flatShading: false });
                this.lineMaterial = new THREE.LineBasicMaterial({ color: Constants.ROAD_LINE_COLOR, linewidth: Constants.ROAD_LINE_WIDTH });
                this.dashedLineMaterial = new THREE.LineDashedMaterial({
                    color: Constants.ROAD_LINE_COLOR,
                    linewidth: Constants.ROAD_LINE_WIDTH,
                    dashSize: Constants.ROAD_DASH_LENGTH,
                    gapSize: Constants.ROAD_DASH_GAP,
                });
                this.sidewalkMaterial = new THREE.MeshPhongMaterial({ color: Constants.ROAD_SIDEWALK_COLOR, flatShading: true });

                for (let i = 0; i < Constants.ROAD_VISIBLE_SEGMENTS + Constants.ROAD_CLEANUP_THRESHOLD; i++) {
                    this.generateNextSegment();
                }
            }

            generateNextSegment() {
                this.curveSegmentCounter++;
                if (this.curveSegmentCounter >= Constants.CURVE_CHANGE_INTERVAL) {
                    this.targetCurveAngle = Utils.randomRange(-Constants.MAX_CURVE_ANGLE, Constants.MAX_CURVE_ANGLE);
                    this.curveSegmentCounter = 0;
                }
                this.currentCurveAngle = Utils.lerp(this.currentCurveAngle, this.targetCurveAngle, 0.1);

                const segmentGroup = new THREE.Group();
                const segmentCenterZ = this.lastZPosition + Constants.ROAD_SEGMENT_LENGTH / 2;

                const roadGeometry = new THREE.PlaneGeometry(Constants.ROAD_WIDTH, Constants.ROAD_SEGMENT_LENGTH);
                roadGeometry.rotateX(-Math.PI / 2);
                roadGeometry.translate(0, 0, segmentCenterZ);

                const roadMesh = new THREE.Mesh(roadGeometry, this.roadMaterial);
                roadMesh.receiveShadow = true;
                roadMesh.position.y = -0.1;

                segmentGroup.add(roadMesh);

                const lineOffset = Constants.ROAD_WIDTH / 2 - Constants.ROAD_LINE_WIDTH / 2;
                const linePointsLeft = [
                    new THREE.Vector3(-lineOffset, 0, segmentCenterZ + Constants.ROAD_SEGMENT_LENGTH / 2),
                    new THREE.Vector3(-lineOffset, 0, segmentCenterZ - Constants.ROAD_SEGMENT_LENGTH / 2)
                ];
                const linePointsRight = [
                    new THREE.Vector3(lineOffset, 0, segmentCenterZ + Constants.ROAD_SEGMENT_LENGTH / 2),
                    new THREE.Vector3(lineOffset, 0, segmentCenterZ - Constants.ROAD_SEGMENT_LENGTH / 2)
                ];
                segmentGroup.add(Utils.createLine(linePointsLeft, this.lineMaterial));
                segmentGroup.add(Utils.createLine(linePointsRight, this.lineMaterial));

                const centerLinePoints = [
                    new THREE.Vector3(0, 0.01, segmentCenterZ + Constants.ROAD_SEGMENT_LENGTH / 2),
                    new THREE.Vector3(0, 0.01, segmentCenterZ - Constants.ROAD_SEGMENT_LENGTH / 2)
                ];
                segmentGroup.add(Utils.createDashedLine(centerLinePoints, this.dashedLineMaterial, Constants.ROAD_DASH_LENGTH, Constants.ROAD_DASH_GAP));

                const sidewalkGeometry = new THREE.BoxGeometry(Constants.ROAD_SIDEWALK_WIDTH, Constants.ROAD_SIDEWALK_HEIGHT, Constants.ROAD_SEGMENT_LENGTH);
                const sidewalkLeft = new THREE.Mesh(sidewalkGeometry, this.sidewalkMaterial);
                sidewalkLeft.position.set(-(Constants.ROAD_WIDTH / 2 + Constants.ROAD_SIDEWALK_WIDTH / 2), Constants.ROAD_SIDEWALK_HEIGHT / 2, segmentCenterZ);
                sidewalkLeft.castShadow = true;
                sidewalkLeft.receiveShadow = true;
                segmentGroup.add(sidewalkLeft);

                const sidewalkRight = new THREE.Mesh(sidewalkGeometry, this.sidewalkMaterial);
                sidewalkRight.position.set((Constants.ROAD_WIDTH / 2 + Constants.ROAD_SIDEWALK_WIDTH / 2), Constants.ROAD_SIDEWALK_HEIGHT / 2, segmentCenterZ);
                sidewalkRight.castShadow = true;
                sidewalkRight.receiveShadow = true;
                segmentGroup.add(sidewalkRight);

                segmentGroup.rotation.y = this.currentCurveAngle;
                segmentGroup.position.z = this.lastZPosition;
                segmentGroup.position.x = this.currentCurveAngle * (Constants.ROAD_SEGMENT_LENGTH / 2);

                this.scene.add(segmentGroup);
                this.roadSegments.push(segmentGroup);

                this.lastZPosition += Constants.ROAD_SEGMENT_LENGTH;
            }

            update(carZPosition) {
                if (carZPosition + Constants.ROAD_SEGMENT_LENGTH * (Constants.ROAD_VISIBLE_SEGMENTS / 2) > this.lastZPosition - Constants.ROAD_SEGMENT_LENGTH) {
                    this.generateNextSegment();
                }

                if (this.roadSegments.length > Constants.ROAD_VISIBLE_SEGMENTS + Constants.ROAD_CLEANUP_THRESHOLD) {
                    const oldSegment = this.roadSegments.shift();
                    this.scene.remove(oldSegment);
                    oldSegment.traverse((object) => {
                        if (object.isMesh) {
                            object.geometry.dispose();
                            object.material.dispose();
                        }
                    });
                }
            }
        }

        // --- 6. CameraController ---
        class CameraController {
            constructor(camera, targetMesh) {
                this.camera = camera;
                this.targetMesh = targetMesh;
                this.offset = new THREE.Vector3(Constants.CAMERA_OFFSET_X, Constants.CAMERA_OFFSET_Y, Constants.CAMERA_OFFSET_Z);
                this.lookAtOffset = new THREE.Vector3(Constants.CAMERA_OFFSET_X, Constants.CAMERA_LOOKAT_OFFSET_Y, Constants.CAMERA_LOOKAT_OFFSET_Z);
                this.smoothFactor = Constants.CAMERA_SMOOTHING;
            }

            update() {
                const desiredPosition = new THREE.Vector3();
                this.targetMesh.localToWorld(desiredPosition.copy(this.offset));

                this.camera.position.lerp(desiredPosition, this.smoothFactor);

                const lookAtPoint = new THREE.Vector3();
                this.targetMesh.localToWorld(lookAtPoint.copy(this.lookAtOffset));
                this.camera.lookAt(lookAtPoint);
            }
        }

        // --- 7. Skybox ---
        class Skybox {
            constructor(scene) {
                this.scene = scene;
                this.createSkybox();
            }

            createSkybox() {
                const geometry = new THREE.SphereGeometry(Constants.SKYBOX_SIZE, 32, 32);
                geometry.scale(-1, 1, 1);

                const material = new THREE.ShaderMaterial({
                    uniforms: {
                        topColor: { value: new THREE.Color(Constants.SKYBOX_COLOR_TOP) },
                        bottomColor: { value: new THREE.Color(Constants.SKYBOX_COLOR_BOTTOM) },
                        offset: { value: 0 },
                        exponent: { value: 0.6 }
                    },
                    vertexShader: `
                        varying vec3 vWorldPosition;
                        void main() {
                            vec4 worldPosition = modelMatrix * vec4( position, 1.0 );
                            vWorldPosition = worldPosition.xyz;
                            gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
                        }
                    `,
                    fragmentShader: `
                        uniform vec3 topColor;
                        uniform vec3 bottomColor;
                        uniform float offset;
                        uniform float exponent;
                        varying vec3 vWorldPosition;
                        void main() {
                            float h = normalize( vWorldPosition ).y;
                            gl_FragColor = vec4( mix( bottomColor, topColor, max( 0.0, pow( max(0.0, h + offset), exponent ) ) ), 1.0 );
                        }
                    `,
                    side: THREE.BackSide
                });

                const skyboxMesh = new THREE.Mesh(geometry, material);
                this.scene.add(skyboxMesh);
            }
        }

        // --- 8. Lights ---
        class SceneLights {
            constructor(scene) {
                this.scene = scene;
                this.addAmbientLight();
                this.addDirectionalLight();
            }

            addAmbientLight() {
                const ambientLight = new THREE.AmbientLight(Constants.AMBIENT_LIGHT_COLOR, Constants.AMBIENT_LIGHT_INTENSITY);
                this.scene.add(ambientLight);
            }

            addDirectionalLight() {
                const directionalLight = new THREE.DirectionalLight(Constants.DIRECTIONAL_LIGHT_COLOR, Constants.DIRECTIONAL_LIGHT_INTENSITY);
                directionalLight.position.set(0, Constants.DIRECTIONAL_LIGHT_POSITION_Y, Constants.DIRECTIONAL_LIGHT_POSITION_Z);
                directionalLight.castShadow = true;
                directionalLight.shadow.mapSize.width = Constants.DIRECTIONAL_LIGHT_SHADOW_MAP_SIZE;
                directionalLight.shadow.mapSize.height = Constants.DIRECTIONAL_LIGHT_SHADOW_MAP_SIZE;
                directionalLight.shadow.camera.near = 0.5;
                directionalLight.shadow.camera.far = 200;
                directionalLight.shadow.camera.left = -Constants.DIRECTIONAL_LIGHT_SHADOW_CAMERA_SIZE;
                directionalLight.shadow.camera.right = Constants.DIRECTIONAL_LIGHT_SHADOW_CAMERA_SIZE;
                directionalLight.shadow.camera.top = Constants.DIRECTIONAL_LIGHT_SHADOW_CAMERA_SIZE;
                directionalLight.shadow.camera.bottom = -Constants.DIRECTIONAL_LIGHT_SHADOW_CAMERA_SIZE;
                directionalLight.shadow.bias = -0.0005;

                this.scene.add(directionalLight);
            }
        }

        // --- 9. Game (Main Game Class) ---
        class Game {
            constructor(scene, camera, renderer) {
                this.scene = scene;
                this.camera = camera;
                this.renderer = renderer;

                this.inputHandler = new InputHandler();
                this.car = new Car(this.scene);
                this.roadGenerator = new RoadGenerator(this.scene);
                this.cameraController = new CameraController(this.camera, this.car.mesh);
                this.skybox = new Skybox(this.scene);
                this.lights = new SceneLights(this.scene);

                this.setupInitialScene();
            }

            setupInitialScene() {
                this.cameraController.update();
            }

            update(deltaTime) {
                this.car.update(deltaTime, this.inputHandler);
                this.cameraController.update();
                this.roadGenerator.update(this.car.getPosition().z);
            }
        }

        // --- Ana Uygulama Mantığı ---
        let scene, camera, renderer;
        let game;
        let loadingScreen = document.getElementById('loadingScreen');
        let speedometerValue = document.getElementById('speedValue');
        let distanceValue = document.getElementById('distanceValue');

        let lastFrameTime = performance.now();

        function init() {
            // Three.js'in yüklenip yüklenmediğini kontrol et
            if (typeof THREE === 'undefined') {
                console.error('THREE.js kütüphanesi yüklenemedi. Lütfen CDN bağlantısını kontrol edin.');
                setTimeout(init, 100); // Tekrar dene
                return;
            }

            scene = new THREE.Scene();
            scene.fog = new THREE.Fog(Constants.FOG_COLOR, Constants.FOG_NEAR, Constants.FOG_FAR);

            camera = new THREE.PerspectiveCamera(Constants.FOV, window.innerWidth / window.innerHeight, Constants.NEAR_PLANE, Constants.FAR_PLANE);

            const canvas = document.getElementById('gameCanvas');
            renderer = new THREE.WebGLRenderer({ canvas: canvas, antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.setClearColor(Constants.CLEAR_COLOR);
            renderer.shadowMap.enabled = true;
            renderer.shadowMap.type = THREE.PCFSoftShadowMap;

            game = new Game(scene, camera, renderer);

            window.addEventListener('resize', onWindowResize, false);

            loadingScreen.classList.add('hidden');
            animate();
        }

        function onWindowResize() {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        }

        function animate() {
            requestAnimationFrame(animate);

            const currentTime = performance.now();
            const deltaTime = (currentTime - lastFrameTime) / 1000;
            lastFrameTime = currentTime;

            game.update(deltaTime);

            speedometerValue.textContent = Math.round(game.car.speed * Constants.M_S_TO_KM_H).toString();
            distanceValue.textContent = Math.round(game.car.distanceDriven).toString();

            renderer.render(scene, camera);
        }

        // Three.js yüklendikten sonra oyunu başlatmak için
        // window.onload yerine, Three.js scriptinin yüklenmesini bekleyelim
        // Bu daha güvenli bir yöntemdir.
        window.addEventListener('DOMContentLoaded', () => {
            const threeScript = document.querySelector('script[src*="three.min.js"]');
            if (threeScript) {
                threeScript.onload = init;
                threeScript.onerror = () => {
                    console.error("Three.js kütüphanesi yüklenirken bir hata oluştu.");
                };
            } else {
                console.error("Three.js script etiketi bulunamadı.");
                init(); // Yine de deneme amaçlı başlat
            }
        });
    </script>
</body>
</html>
