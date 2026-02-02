<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Hand Gesture Particles</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; }
        video { display: none; } /* ক্যামেরা ফিড হাইড রাখা হয়েছে */
        canvas { display: block; }
    </style>
</head>
<body>

<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>

<video id="input_video"></video>

<script>
const videoElement = document.getElementById('input_video');

// --- Three.js Setup ---
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);
camera.position.z = 5;

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// --- Particles ---
const particleCount = 8008;
const geometry = new THREE.BufferGeometry();
const positions = new Float32Array(particleCount * 3);
const colors = new Float32Array(particleCount * 3);

for (let i = 0; i < particleCount * 3; i++) {
    positions[i] = (Math.random() - 0.5) * 10;
    colors[i] = Math.random();
}

geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));

const material = new THREE.PointsMaterial({ size: 0.03, vertexColors: true, transparent: true });
const points = new THREE.Points(geometry, material);
scene.add(points);

// --- Hand Tracking ---
const hands = new Hands({locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`});
hands.setOptions({ maxNumHands: 1, modelComplexity: 1, minDetectionConfidence: 0.7 });

hands.onResults((results) => {
    if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
        const landmarks = results.multiHandLandmarks[0];
        updateParticles(landmarks);
    }
});

const cameraHandler = new Camera(videoElement, {
    onFrame: async () => { await hands.send({image: videoElement}); },
    width: 640, height: 480
});
cameraHandler.start();

// --- logic ---
function updateParticles(landmarks) {
    const pos = geometry.attributes.position.array;
    const thumb = landmarks[4];
    const index = landmarks[8];

    // Pinch distance
    const dist = Math.sqrt(Math.pow(thumb.x - index.x, 2) + Math.pow(thumb.y - index.y, 2));

    for (let i = 0; i < particleCount; i++) {
        const idx = i * 3;
        if (dist < 0.05) { 
            pos[idx] *= 1.1; // Pinch করলে ছড়িয়ে যাবে
            pos[idx+1] *= 1.1;
        } else {
            pos[idx] *= 0.98; // না করলে কেন্দ্রে আসবে
            pos[idx+1] *= 0.98;
        }
    }
    geometry.attributes.position.needsUpdate = true;
}

function animate() {
    requestAnimationFrame(animate);
    points.rotation.y += 0.002;
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

