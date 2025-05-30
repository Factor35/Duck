# Duck

This is a test for hosting model files on Github.


<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>3D Model Viewer with Movable Parts</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        body {
            margin: 0;
            overflow: hidden;
            font-family: 'Inter', sans-serif;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            min-height: 100vh;
            background-color: #f0f2f5;
        }
        canvas {
            display: block;
            width: 100%;
            height: calc(100vh - 100px); /* Adjust height to accommodate controls */
            border-radius: 12px;
            box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
            background-color: #ffffff;
        }
        .controls {
            padding: 16px;
            background-color: #ffffff;
            border-radius: 12px;
            box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
            margin-top: 20px;
            display: flex;
            gap: 16px;
            flex-wrap: wrap;
            justify-content: center;
        }
        button {
            padding: 10px 20px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            font-size: 1rem;
            transition: background-color 0.3s ease, transform 0.2s ease;
            box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
        }
        button:hover {
            background-color: #45a049;
            transform: translateY(-2px);
        }
        button:active {
            background-color: #3e8e41;
            transform: translateY(0);
        }
        .message-box {
            position: fixed;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background-color: #333;
            color: white;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
            z-index: 1000;
            display: none; /* Hidden by default */
        }
        .message-box button {
            margin-top: 10px;
            background-color: #007bff;
            padding: 8px 15px;
            border-radius: 5px;
            cursor: pointer;
        }
    </style>
</head>
<body class="bg-gray-100 p-4">

    <div id="container" class="w-full max-w-4xl h-[calc(100vh-120px)] flex flex-col items-center justify-center">
        <canvas id="renderCanvas" class="w-full h-full"></canvas>
    </div>

    <div class="controls">
        <button id="movePartButton" class="bg-blue-500 hover:bg-blue-600">Move Example Part</button>
        <button id="resetViewButton" class="bg-gray-500 hover:bg-gray-600">Reset View</button>
    </div>

    <div id="messageBox" class="message-box">
        <p id="messageText"></p>
        <button onclick="document.getElementById('messageBox').style.display = 'none';">OK</button>
    </div>

    <script type="module">
        import * as THREE from 'https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.module.js';
        import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.128.0/examples/jsm/controls/OrbitControls.js';
        import { GLTFLoader } from 'https://cdn.jsdelivr.net/npm/three@0.128.0/examples/jsm/loaders/GLTFLoader.js';

        let scene, camera, renderer, controls;
        let model; // To store the loaded GLTF model

        function showMessage(message) {
            const messageBox = document.getElementById('messageBox');
            const messageText = document.getElementById('messageText');
            messageText.textContent = message;
            messageBox.style.display = 'block';
        }

        function init() {
            // Scene
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0xf0f2f5); // Light background

            // Camera
            camera = new THREE.PerspectiveCamera(75, window.innerWidth / (window.innerHeight - 100), 0.1, 1000);
            camera.position.set(5, 5, 5); // Initial camera position

            // Renderer
            const canvas = document.getElementById('renderCanvas');
            renderer = new THREE.WebGLRenderer({ canvas: canvas, antialias: true });
            renderer.setSize(canvas.clientWidth, canvas.clientHeight);
            renderer.setPixelRatio(window.devicePixelRatio);

            // Orbit Controls
            controls = new OrbitControls(camera, renderer.domElement);
            controls.enableDamping = true; // For smoother camera movement
            controls.dampingFactor = 0.25;
            controls.screenSpacePanning = false;
            controls.maxPolarAngle = Math.PI / 2; // Limit vertical rotation

            // Lighting
            const ambientLight = new THREE.AmbientLight(0xffffff, 0.7); // Soft white light
            scene.add(ambientLight);

            const directionalLight = new THREE.DirectionalLight(0xffffff, 0.5);
            directionalLight.position.set(5, 10, 7.5).normalize();
            scene.add(directionalLight);

            // Load GLB Model
            const gltfLoader = new GLTFLoader();
            gltfLoader.load(
                'output.glb', // Path to your generated GLB file
                (gltf) => {
                    model = gltf.scene;
                    scene.add(model);

                    // Optional: Fit camera to model
                    const box = new THREE.Box3().setFromObject(model);
                    const center = box.getCenter(new THREE.Vector3());
                    const size = box.getSize(new THREE.Vector3());
                    const maxDim = Math.max(size.x, size.y, size.z);
                    const fov = camera.fov * (Math.PI / 180);
                    let cameraZ = Math.abs(maxDim / 2 / Math.tan(fov / 2));
                    cameraZ *= 1.5; // Add some padding

                    camera.position.set(center.x, center.y, center.z + cameraZ);
                    controls.target.copy(center);
                    controls.update();

                    showMessage('Model loaded successfully! You can now interact with it.');

                    // Log all named objects to the console for debugging
                    console.log("Named objects in the loaded model:");
                    model.traverse((child) => {
                        if (child.isMesh && child.name) {
                            console.log(`- ${child.name}`);
                        }
                    });
                },
                (xhr) => {
                    console.log((xhr.loaded / xhr.total * 100) + '% loaded');
                },
                (error) => {
                    console.error('An error occurred while loading the GLB model:', error);
                    showMessage('Error loading 3D model. Check console for details.');
                }
            );

            // Event Listeners
            window.addEventListener('resize', onWindowResize, false);
            document.getElementById('movePartButton').addEventListener('click', movePart);
            document.getElementById('resetViewButton').addEventListener('click', resetView);

            animate();
        }

        function onWindowResize() {
            const canvas = document.getElementById('renderCanvas');
            camera.aspect = canvas.clientWidth / canvas.clientHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(canvas.clientWidth, canvas.clientHeight);
        }

        function animate() {
            requestAnimationFrame(animate);
            controls.update(); // Only required if controls.enableDamping or controls.autoRotate are set to true
            renderer.render(scene, camera);
        }

        function movePart() {
            if (!model) {
                showMessage('Model not loaded yet.');
                return;
            }

            // IMPORTANT: Replace 'YOUR_PART_NAME_HERE' with an actual name from your DXF/GLB.
            // You can find names by inspecting the GLB file or checking the console output
            // after the model loads (look for "Named objects in the loaded model:").
            const partToMoveName = 'YOUR_PART_NAME_HERE'; 
            // Example: If you have a block named 'ARM', and it contains a solid, its name might be 'ARM_Solid_...'
            // Or if a solid is directly in modelspace, it might be 'ModelspaceSolid_...'

            const part = model.getObjectByName(partToMoveName);

            if (part) {
                // Example: Move the part along the X-axis
                part.position.x += 1;
                showMessage(`Moved part: ${partToMoveName}`);
            } else {
                showMessage(`Part with name "${partToMoveName}" not found. Check console for available names.`);
                console.warn(`Part "${partToMoveName}" not found in the model.`);
            }
        }

        function resetView() {
            if (model) {
                const box = new THREE.Box3().setFromObject(model);
                const center = box.getCenter(new THREE.Vector3());
                const size = box.getSize(new THREE.Vector3());
                const maxDim = Math.max(size.x, size.y, size.z);
                const fov = camera.fov * (Math.PI / 180);
                let cameraZ = Math.abs(maxDim / 2 / Math.tan(fov / 2));
                cameraZ *= 1.5;

                camera.position.set(center.x, center.y, center.z + cameraZ);
                controls.target.copy(center);
                controls.update();
                showMessage('View reset.');
            }
        }

        window.onload = init; // Initialize Three.js scene when the window loads
    </script>
</body>
</html>
